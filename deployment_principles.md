# Deployment Principles

Cross-project reference documenting the deployment conventions shared by **artbots**, **fedi-monitor**, and **boekwinkeltjes-scraper** on the same Ubuntu server. This is not a deployment guide for fedi-dashboard -- it is a living document that captures the "how and why" behind every deployment decision so that future projects follow the same patterns.

---

## Table of Contents

1. [Directory Structure Convention](#1-directory-structure-convention)
2. [Service User Model](#2-service-user-model)
3. [GitHub Deploy User (SSH Authentication)](#3-github-deploy-user-ssh-authentication)
4. [Git Operations](#4-git-operations)
5. [File Permissions Model](#5-file-permissions-model)
6. [Environment Configuration](#6-environment-configuration)
7. [Python Virtual Environment](#7-python-virtual-environment)
8. [Node.js Dependencies](#8-nodejs-dependencies)
9. [Systemd Service Configuration](#9-systemd-service-configuration)
10. [Systemd Security Hardening](#10-systemd-security-hardening)
11. [Deploy Script Architecture](#11-deploy-script-architecture)
12. [Database Backup Strategy](#12-database-backup-strategy)
13. [Cross-Platform Considerations](#13-cross-platform-considerations)
14. [Firewall Configuration](#14-firewall-configuration)
15. [Logging and Monitoring](#15-logging-and-monitoring)
16. [Web Server Pattern](#16-web-server-pattern)
17. [Deployment Documentation Convention](#17-deployment-documentation-convention)
18. [Update Workflow](#18-update-workflow)
19. [Rollback Strategy](#19-rollback-strategy)
20. [Service Naming Convention](#20-service-naming-convention)
21. [Port Allocation](#21-port-allocation)
22. [Cross-Project Database Access](#22-cross-project-database-access)
23. [Tracked Files with Intentional Local Overrides](#23-tracked-files-with-intentional-local-overrides)
24. [Large-Scale Scraping: Ubuntu Network Tuning](#24-large-scale-scraping-ubuntu-network-tuning) *(includes IPv6 routing check)*

---

## 1. Directory Structure Convention

**Rule:** All projects live under `/srv/<project-name>/` with a consistent subdirectory layout.

**Rationale:** `/srv/` is the FHS-standard location for site-specific data served by the system. A uniform layout means any team member can navigate any project without learning a new structure, and deploy scripts can share logic.

**Layout:**

```
/srv/<project>/
    .env                  # Environment secrets (mode 600)
    config/               # JSON configuration files
    data/                 # Runtime data
        database/         # SQLite database files (or databases/)
        backups/          # Timestamped database backups
    logs/                 # Application log files
    deployment/           # Systemd unit files (source of truth)
    scripts/              # deploy.sh and utilities
    venv/                 # Python virtual environment (Python projects)
```

**Evidence:**

| Project | Path | Notes |
|---------|------|-------|
| artbots | `/srv/artbots/` | `data/` contains `artbots.db` and `backups/` |
| fedi-monitor | `/srv/fedi-monitor/` | Uses `databases/` (plural) and separate `backups/`, `state/`, `logs/` |
| boekwinkeltjes | `/srv/boekwinkeltjes/` | `data/database/` for SQLite, `data/backups/` for backups |

---

## 2. Service User Model

**Rule:** Every project gets a dedicated system user with no home directory and no login shell. The service user owns all application files. A human user (jahof) performs Git and administrative operations via sudo.

**Rationale:** Least-privilege isolation. If one service is compromised, the attacker gets only that user's permissions. No login shell means the account cannot be used interactively. No SSH keys on the service user means it cannot reach external systems.

**Creation pattern:**

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin <username>
```

> **Note:** On Ubuntu, `--system` implies `--no-create-home`, but being explicit is safer for portability across distributions.

**Evidence:**

| Project | Service User | Notes |
|---------|-------------|-------|
| artbots | `artbots` | Owns `/srv/artbots/`, no SSH keys |
| fedi-monitor | `fedimonitor` | Owns `/srv/fedi-monitor/`, no SSH keys |
| boekwinkeltjes | `boekwinkeltjes` | Owns `/srv/boekwinkeltjes/`, no SSH keys |

**Key constraints:**

- Service user has NO SSH keys (security isolation)
- Service user NEVER runs Git operations
- Human user (jahof) does Git operations, then chowns files back to service user
- Root performs initial clone and system-level operations

---

## 3. GitHub Deploy User (SSH Authentication)

**Rule:** All GitHub authentication on the server uses a dedicated GitHub machine user (`hofnet-deploy-bot`), never a personal account SSH key or per-repo deploy keys. The machine user is added as a **Read** collaborator on each repo the server needs to access. Its SSH keypair lives at `/root/.ssh/hofnet_machine_key`.

**Rationale:** A personal SSH key on the server grants access to every repo the personal account can reach — a large blast radius if the server is compromised. A dedicated machine user limits access to explicitly listed repos, is independent of personal account credential changes, and requires only one SSH keypair for all repos rather than one per repo.

**First-time server setup:** See `deploy_user_setup.md`.

### Adding a new repo

1. Add `hofnet-deploy-bot` as a **Read** collaborator on the GitHub repo (don not give this user write privilege!!).
2. Clone with `git@github.com:Owner/repo.git` — no further key setup needed.

**Evidence:** All server repos are accessed via `hofnet-deploy-bot`. Key stored at `/root/.ssh/hofnet_machine_key`.

---

## 4. Git Operations

**Rule:** Always use SSH remotes (never HTTPS); authentication is handled by the deploy user SSH key (Section 3). Git operations are performed as root (or via sudo), never as the service user. After every pull, ownership is fixed.

**Rationale:** SSH avoids credential prompts and token expiry. The service user has no SSH keys (by design), so Git operations must be done by a user that does. Fixing ownership after pull is mandatory because git pull creates files owned by whoever ran it.

**Remote format:**

```
git@github.com:bringolo/<repo>.git
```

**Safe directory configuration** (required because the repo is owned by the service user, not root):

```bash
# As root (for sudo git operations)
sudo git config --global --add safe.directory /srv/<project>

# As the human user jahof (for non-sudo git operations)
git config --global --add safe.directory /srv/<project>
```

This must be set for both root and the human user (jahof).

**Initial clone workflow:**

```bash
sudo git clone git@github.com:bringolo/<repo>.git /srv/<project>
cd /srv/<project>
sudo git checkout main    # Verify correct branch -- clone may default to develop
sudo chown -R <user>:<user> /srv/<project>
```

> **Note:** All projects use `main` as the default branch. Always verify the branch name after initial clone, especially if the repo was created before GitHub's default branch rename from `master` to `main`.

**Standard pull workflow:**

```bash
cd /srv/<project>
sudo git fetch origin main
sudo git pull origin main
sudo chown -R <user>:<user> /srv/<project>
```

> **Common pitfall:** Never attempt git operations as the service user, even with a fallback to root. The service user has no SSH keys by design and git will fail silently or with confusing errors. Always run git as root (via `sudo`) or as the human user. This was learned the hard way when boekwinkeltjes' deploy script originally tried `sudo -u $SERVICE_USER git fetch` with a root fallback -- the correct fix was to remove the service-user attempt entirely.

> **Troubleshooting:** If `git pull` fails with "local changes would be overwritten by merge", use `sudo git reset --hard origin/main` on the production server. Production servers should always match the remote exactly. Always check `sudo git status` and `sudo git diff` before resetting to verify no important uncommitted changes exist.

**Evidence:** All three projects use SSH remotes and follow this ownership-fix pattern in their `deploy.sh` scripts. The safe.directory configuration is set during initial server setup and persists globally.

---

## 5. File Permissions Model

**Rule:** Permissions follow a strict hierarchy. Secrets are locked down to owner-only. No world-writable files exist anywhere.

**Rationale:** Defense in depth. Even if another service user is compromised, it cannot read secrets from a sibling project. Group-readable databases allow monitoring tools to inspect without write access.

| Resource | Mode | Meaning |
|----------|------|---------|
| `.env` | `600` | Owner read/write only -- CRITICAL for secrets |
| Database files (`.db`) | `640` | Owner read/write, group read |
| Data directories | `750` | Owner full, group read/execute |
| Project root | `755` | World-readable (code is public anyway) |
| Scripts (`.sh`) | `755` | Executable by all (guarded by sudo in practice) |

**Evidence:** All three deploy scripts enforce these permissions in their "Permissions" phase. The `.env` file permission (600) is treated as a security-critical check -- deploy scripts warn if it is more permissive.

---

## 6. Environment Configuration

**Rule:** Runtime configuration and secrets live in a `.env` file at the project root, loaded by systemd's `EnvironmentFile=` directive. A `.env.example` is committed to the repo as a template; the real `.env` is never committed.

**Rationale:** Secrets must never enter version control. The `.env` file is the single source of truth for environment-specific values (production vs development). systemd's `EnvironmentFile=` makes these variables available to the service without custom loading code.

**Pattern:**

```ini
# .env.example (committed to repo)
FEDI_MONITOR_ENV=production
# SECRET_KEY=generate-a-real-key-on-server
```

```ini
# .env (on server, mode 600, never committed)
FEDI_MONITOR_ENV=production
SECRET_KEY=actual-secret-value-here
```

**Secrets are generated on the server:**

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

**Evidence:**

| Project | Key Variables | Secrets |
|---------|--------------|---------|
| artbots | Bot API credentials, Bluesky/Mastodon tokens | Many (API keys, passwords) |
| fedi-monitor | `FEDI_MONITOR_ENV`, optional DB tuning | None (public API scraper) |
| boekwinkeltjes | `BOEKWINKELTJES_ENV`, `SECRET_KEY` | `SECRET_KEY` (Flask sessions) |

---

## 7. Python Virtual Environment

*Applies to: Python projects (fedi-monitor, boekwinkeltjes-scraper, fedi-dashboard)*

**Rule:** Create the venv as the service user. Install dependencies as the service user. All systemd `ExecStart` directives use the full venv path. Always upgrade pip before installing requirements.

**Rationale:** Running venv creation and pip install as the service user ensures all files are owned by the correct user without needing a chown pass. The full path in systemd avoids relying on PATH or activation scripts. Upgrading pip first prevents version-mismatch warnings and ensures the latest resolver is used.

**Creation:**

```bash
sudo -u <user> python3 -m venv /srv/<project>/venv
```

**Dependency installation:**

```bash
sudo -u <user> /srv/<project>/venv/bin/pip install --upgrade pip
sudo -u <user> /srv/<project>/venv/bin/pip install -r /srv/<project>/requirements.txt
```

**systemd ExecStart:**

```ini
ExecStart=/srv/<project>/venv/bin/python /srv/<project>/run.py
# or for gunicorn:
ExecStart=/srv/<project>/venv/bin/gunicorn --bind 0.0.0.0:8005 wsgi:app
```

**Evidence:** Both fedi-monitor and boekwinkeltjes-scraper follow this pattern exactly. Their deploy scripts create the venv if missing and always run `pip install --upgrade pip` before `pip install -r requirements.txt`.

> **Gap to fix:** fedi-monitor's `docs/deployment.md` skips the `pip install --upgrade pip` step in the manual installation instructions, contradicting this principle. The deploy script does it correctly.

---

## 8. Node.js Dependencies

*Applies to: Node.js projects (artbots)*

**Rule:** Override npm's cache and HOME directories for the system user (which has no home directory). Install with `--omit=dev` for production.

**Rationale:** System users have no home directory, so npm's default cache location (`~/.npm`) fails. Overriding `HOME` and using a temp-based cache avoids this. Omitting dev dependencies reduces the attack surface and installation time.

**Pattern:**

```bash
sudo -u artbots env HOME=/srv/artbots npm ci --omit=dev --cache /tmp/npm-cache-artbots
```

**Evidence:** artbots' deploy script sets `HOME=/srv/artbots` and `--cache /tmp/npm-cache-artbots` for all npm operations. Uses `npm ci` (clean install) rather than `npm install` for reproducible builds.

> **Note:** The `--production` flag is deprecated since npm v7. Always use `--omit=dev` instead. This was a real deployment fix in artbots (commit `a280299`).

---

## 9. Systemd Service Configuration

**Rule:** Service files are version-controlled in the `deployment/` directory and copied to `/etc/systemd/system/` during deployment. After copying, run `systemctl daemon-reload`.

**Rationale:** Keeping unit files in the repo means they are versioned, reviewed, and deployed alongside the code. Copying (rather than symlinking) avoids issues if the repo directory permissions change.

**Service types:**

| Type | Use Case | Example |
|------|----------|---------|
| `simple` | Long-running processes (web servers) | `artbots-gui.service`, `boekwinkeltjes-web.service` |
| `oneshot` | Batch jobs, scrapers, pipelines | `fedi-monitor-pipeline.service`, `artbots@.service` |

**Common directives for all services:**

```ini
[Service]
User=<service-user>
Group=<service-user>
WorkingDirectory=/srv/<project>
EnvironmentFile=/srv/<project>/.env
SyslogIdentifier=<project-name>
StandardOutput=journal
StandardError=journal
```

> **Gap to fix:** `artbots-gui.service` is currently missing `SyslogIdentifier`, `StandardOutput`, and `StandardError`. These should be added for consistency with all other services.

**Restart policy for long-running services:**

```ini
[Unit]
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Restart=on-failure
RestartSec=10
```

> **Note:** `StartLimitBurst` and `StartLimitIntervalSec` belong in the `[Unit]` section, not `[Service]`. `Restart=on-failure` is the recommended default. artbots-gui currently uses `Restart=always` which also restarts on clean exit -- use this only when the service should never stop.

**Timer directives:**

```ini
[Timer]
Persistent=true              # Run missed executions after downtime
RandomizedDelaySec=600       # Prevent thundering herd (max random delay in seconds)
AccuracySec=60               # Timer precision (default 1min is fine for most jobs)

[Install]
WantedBy=timers.target       # Timers use timers.target, NOT multi-user.target
```

- `OnCalendar=` for fixed schedules (daily, weekly)
- `OnUnitActiveSec=` for interval-based schedules (every 5 minutes)

> **Common pitfall:** When referencing timer files in deploy scripts, be careful not to double-append the `.timer` extension. Store the full filename in variables (e.g., `SCRAPER_TIMER="boekwinkeltjes-scraper.timer"`) and reference it directly without adding another extension. This was a real bug fixed in boekwinkeltjes (commit `8dd1219`).

**Evidence:**

| Project | Services | Timers |
|---------|----------|--------|
| artbots | `artbots-gui.service` (simple), `artbots@.service` (oneshot template) | 8 timers (daily bot posting + daily stats) |
| fedi-monitor | `fedi-monitor-pipeline.service`, `fedi-monitor-db-update.service`, `fedi-monitor-status.service` (all oneshot) | 2 timers (daily pipeline + weekly DB update) |
| boekwinkeltjes | `boekwinkeltjes-web.service` (simple), `boekwinkeltjes-scraper.service` (oneshot) | 1 timer (5-minute interval scraping) |

---

## 10. Systemd Security Hardening

**Rule:** Every service file includes a mandatory security hardening block. No exceptions.

**Rationale:** These directives implement the principle of least privilege at the kernel level. Even if an attacker gains code execution within the service, they cannot escalate privileges, access other users' files, or modify the system.

**Mandatory block (all services):**

```ini
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
ReadWritePaths=/srv/<project>/data    # Only what the service needs
```

| Directive | Protection |
|-----------|-----------|
| `NoNewPrivileges` | Prevents SUID/SGID escalation |
| `ProtectSystem=strict` | Makes `/usr`, `/boot`, `/etc` read-only |
| `ProtectHome=true` | Makes `/home`, `/root`, `/run/user` inaccessible |
| `PrivateTmp=true` | Gives the service its own `/tmp` |
| `ProtectKernelTunables` | Prevents writing to `/proc/sys` and `/sys` |
| `ProtectKernelModules` | Prevents loading kernel modules |
| `ProtectControlGroups` | Prevents cgroup modifications |
| `ReadWritePaths` | Explicit allowlist for writable directories |

**Additional hardening (used by fedi-monitor):**

```ini
PrivateDevices=true
RestrictSUIDSGID=true
RestrictNamespaces=true
RestrictRealtime=true
LockPersonality=true
```

> **Note:** systemd accepts both `true`/`false` and `yes`/`no` for boolean directives. Standardize on `true`/`false` for consistency across all projects.

> **Gap to close:** boekwinkeltjes has adopted `RestrictSUIDSGID` and `RestrictRealtime` but is still missing `PrivateDevices`, `RestrictNamespaces`, and `LockPersonality`. These should be added to both `boekwinkeltjes-web.service` and `boekwinkeltjes-scraper.service`.

**Resource limits (always set):**

```ini
MemoryMax=<appropriate-limit>
CPUQuota=<appropriate-limit>
```

| Project | Component | MemoryMax | CPUQuota | TimeoutStartSec |
|---------|-----------|-----------|----------|-----------------|
| artbots | GUI | 256M | 50% | default |
| artbots | Bots | 512M | 80% | 300 |
| fedi-monitor | Pipeline | 1536M | 80% | 7200 |
| boekwinkeltjes | Web | 256M | 25% | default |
| boekwinkeltjes | Scraper | 1G | 75% | default |

**Evidence:** All three projects include the mandatory hardening block. fedi-monitor uses the most comprehensive set. Resource limits are tuned per-service based on observed usage.

> **Debugging tip:** If a service fails with permission errors under `ProtectSystem=strict`, check that all required writable paths are listed in `ReadWritePaths`. Use `journalctl -u <service>` and look for EACCES/EPERM errors. This was a real issue with `fedi-monitor-status.service` which initially lacked `ReadWritePaths=/srv/fedi-monitor/databases` (fixed in commit `4c11743`).

---

## 11. Deploy Script Architecture

**Rule:** Every project ships a `scripts/deploy.sh` that follows an 8-phase pattern with consistent CLI flags and error codes.

**Rationale:** A uniform deploy script means anyone who has deployed one project can deploy any other. The phases are ordered to fail fast (pre-flight first) and to protect data (backup before changes). Consistent error codes allow monitoring scripts to parse failures programmatically.

**The 8 phases:**

| Phase | Name | Purpose |
|-------|------|---------|
| 1 | **Initialization** | Constants, color codes, CLI argument parsing, cleanup traps |
| 2 | **Pre-flight checks** | Validate user (must be root), required tools, directories, `.env` exists, disk space, available memory |
| 3 | **Database backup** | Integrity check on live DB, timestamped copy, verify backup integrity, rotate old backups |
| 4 | **Git operations** | `git fetch origin main`, `git pull origin main`, fix ownership with `chown` |
| 5 | **Dependencies** | `pip install` or `npm ci`, upgrading package managers first |
| 6 | **Permissions** | `chown -R` to service user, `chmod` for `.env` (600), databases (640), scripts (755) |
| 7 | **Services** | Stop services, copy unit files from `deployment/` to `/etc/systemd/system/`, `daemon-reload`, start, enable |
| 8 | **Verification** | Health checks (curl for web services), service status, database integrity, deployment summary |

**Phase 8 — Deployment summary:**

**Rule:** On successful completion, the deploy script must print a clearly visible summary block containing all information needed to verify and operate the deployment.

**Rationale:** After a deployment, the operator needs to know at a glance: did it work, what was deployed, what data is safe, and what to do next. A consistent summary block makes this a predictable, scannable artifact rather than scattered log lines.

**Required fields:**

| Field | Purpose |
|-------|---------|
| Deployment timestamp | Matches the backup filename; ties backup to this exact run |
| Commit (before → after) | Confirms what code was deployed |
| Backup filename | Lets the operator verify the backup exists and use it in `--rollback` |
| Web interface URL | Immediately confirms the service is reachable |
| View logs command | Removes friction when the first thing to do is check logs |
| Rollback command | Immediately available if something looks wrong |

**Format:**

```bash
echo ""
echo "============================================================================"
echo " DEPLOYMENT COMPLETED SUCCESSFULLY"
echo "============================================================================"
echo "[INFO] Deployment timestamp: ${TIMESTAMP}"
echo "[INFO] Commit: ${PREVIOUS_COMMIT} -> ${CURRENT_COMMIT}"
echo "[INFO] Backup: ${BACKUP_FILENAME}"
echo "[INFO] Web interface: http://${SERVER_IP}:${PORT}"
echo "[INFO] View logs: journalctl -u ${WEB_SERVICE} -f"
echo "[INFO] Rollback: sudo bash ./scripts/deploy.sh --rollback"
echo ""
```

- The `===` border must span the full terminal width (80 characters) so the block is visually distinct from phase output
- Omit `Web interface` for projects without a web service (e.g. fedi-monitor)
- Include a log command per service if the project has multiple services
- `PREVIOUS_COMMIT` and `CURRENT_COMMIT` are captured before and after the git pull in Phase 4

**Evidence:** boekwinkeltjes implements this pattern exactly.

---

**Script conventions:**

```bash
#!/bin/bash
set -euo pipefail    # Exit on error, undefined vars, pipe failures
```

- Run with `sudo bash scripts/deploy.sh` (not direct execution)
- Colored output: `RED`, `GREEN`, `YELLOW`, `BLUE`, `NC` (No Color)
- Functions for each phase, called sequentially from `main()`

**Phase 7 — Services step detail:**

**Rule:** The services phase must follow this exact sequence: copy unit files, daemon-reload, restart, enable. Skipping any step causes a different class of failure.

**Rationale:** Each step exists for a specific reason:

1. **Copy unit files** from `deployment/` to `/etc/systemd/system/` — ensures the running configuration matches the repository. Without this, a deploy that changes a unit file (e.g. adding an environment variable) has no effect until someone manually copies the file.
2. **`systemctl daemon-reload`** — tells systemd to re-read unit files from disk. Without this, systemd continues using its cached version even after files are copied.
3. **`systemctl restart`** — applies the new configuration to running services immediately.
4. **`systemctl enable`** — creates the symlinks that make services start on boot. Without this, everything works fine until the server reboots, at which point nothing starts.

**What to enable vs. not enable:**

- **Enable:** long-running services (e.g. `artbots-gui.service`) and timers (e.g. `ubuntu-monitor-collector.timer`)
- **Do not enable:** oneshot services triggered by timers (e.g. `ubuntu-monitor-collector.service`) — they are activated by their timer, not by boot

**Code example:**

```bash
phase_services() {
    print_phase 7 "Services"

    # Copy unit files from repo to systemd
    cp deployment/*.service /etc/systemd/system/
    cp deployment/*.timer   /etc/systemd/system/

    # Reload systemd so it picks up any changes
    systemctl daemon-reload

    # Restart services to apply new code/config
    systemctl restart my-project-web.service
    systemctl restart my-project-collector.timer

    # Enable for boot — without this, a reboot kills everything
    systemctl enable my-project-web.service
    systemctl enable my-project-collector.timer
}
```

> **Gap to fix:** artbots' deploy script was missing `systemctl enable` calls for all three of its services, meaning a server reboot would leave artbots completely down. The enable calls have now been fixed. The script is also missing the unit file copy step (copying from `deployment/` to `/etc/systemd/system/`), tracked as TD-001.

**CLI flags (consistent across all three projects):**

| Flag | Purpose |
|------|---------|
| `--dry-run` | Show what would happen without making changes |
| `--check` | Pre-flight validation only |
| `--status` | Show current service and timer status |
| `--verify` | Run verification checks only |
| `--rollback` | Restore latest DB backup and reset Git |
| `--skip-backup` | Skip database backup phase |
| `--restart-only` | Only restart services (skip git, deps, permissions) |
| `--setup-firewall` | Configure UFW rules for the project |
| `--help` | Show usage information |

**Error codes:**

| Code | Category | Meaning |
|------|----------|---------|
| E001 | Git | Git fetch/pull failure |
| E002 | Dependencies | pip/npm install failure |
| E003 | Service | systemd start/enable failure |
| E004 | Database | Backup or integrity check failure |
| E005 | Permissions | chown/chmod failure |

**Evidence:** All three projects have deploy scripts between 900-975 lines following this exact pattern. The CLI flags are identical. Error codes use the same numbering scheme (artbots and fedi-monitor use string codes like `E001`, boekwinkeltjes maps them to exit codes: `E001_GIT=11`, `E002_DEPS=12`, etc.).

---

## 12. Database Backup Strategy

**Rule:** Database backups are timestamped, integrity-verified, and automatically rotated. A backup always runs before any deployment.

**Rationale:** SQLite databases are the most valuable asset in each project. Code can always be re-pulled from Git, but data is irreplaceable. Integrity checks before AND after backup catch corruption early. Keeping the last 10 strikes a balance between safety and disk usage.

**Backup procedure:**

1. Run `PRAGMA integrity_check;` on the live database
2. Create timestamped copy: `<db-name>_YYYYMMDD_HHMMSS.db`
3. Run `PRAGMA integrity_check;` on the backup copy
4. Remove backups beyond the last 10

**Backup locations:**

| Project | Backup Directory | Naming Pattern |
|---------|-----------------|----------------|
| artbots | `/srv/artbots/data/backups/` | `backup-YYYYMMDD_HHMMSS.db` |
| fedi-monitor | `/srv/fedi-monitor/backups/` | `prod_fedi_2_YYYYMMDD_HHMMSS.db` |
| boekwinkeltjes | `/srv/boekwinkeltjes/data/backups/` | `books_production_YYYYMMDD_HHMMSS.db` |

**Backup methods:**

- artbots and boekwinkeltjes: `cp` (simple copy, safe when service is stopped)
- fedi-monitor: SQLite `.backup` command (safer for databases that might be in use)

**Retention:** Last 10 backups kept, older ones deleted automatically. All three projects use the same retention count.

---

## 13. Cross-Platform Considerations

**Rule:** All development happens on Windows; all deployment happens on Linux. Handle the differences explicitly.

**Rationale:** Windows and Linux disagree on line endings and file permissions. Git on Windows doesn't track the execute bit. Ignoring these differences leads to broken scripts on the server.

**Line endings (preferred -- `.gitattributes`):**

The correct solution is to normalize line endings at the Git level using a `.gitattributes` file in the repository root:

```
# Auto-detect text files and normalize line endings (LF in repo)
* text=auto

# Force LF for files that must work on Linux (even when checked out on Windows)
*.sh text eol=lf
*.py text eol=lf
*.service text eol=lf
```

**How it works:**

| Setting | Effect |
|---------|--------|
| `* text=auto` | Git auto-detects text files and normalizes to LF in the repository |
| `eol=lf` | Forces LF on checkout, even on Windows (safe -- modern editors handle LF) |

This eliminates the CRLF/LF mismatch at the source. Files are always LF in the repo and always LF on checkout for file types that must be LF on Linux. No post-pull conversion is needed.

**After adding `.gitattributes` to an existing repo**, Git needs to re-normalize tracked files:

```bash
# On the development machine (one-time)
git add --renormalize .
git commit -m "Normalize line endings via .gitattributes"
```

**Why this replaces `dos2unix`:**

The previous approach used `dos2unix` in the deploy script to convert line endings after every `git pull`. This worked but had a critical side effect: converting a tracked file's line endings creates a local modification that blocks the next `git pull` with "local changes would be overwritten by merge". The `.gitattributes` approach prevents the problem at the source rather than fixing it after the fact.

**Line endings (legacy -- `dos2unix`):**

Projects that have not yet adopted `.gitattributes` still use `dos2unix` as a fallback:

```bash
# Install dos2unix on the server
sudo apt install dos2unix

# Convert all shell scripts after git pull
find /srv/<project> -name "*.sh" -exec dos2unix {} +
```

Deploy scripts that still use this approach handle it automatically in the Git phase.

**Execute bit:**

Windows does not track the execute permission. After every git pull, deploy scripts explicitly set:

```bash
chmod +x /srv/<project>/scripts/*.sh
```

This is why scripts are invoked with `sudo bash scripts/deploy.sh` rather than `./scripts/deploy.sh` -- it sidesteps the execute-bit issue entirely.

**Pitfall:** If a shell script is committed from Windows without explicitly setting the executable bit in git, it is stored as mode `100644`. On Linux, the deploy script's `chmod 755` then flips it to `100755` on every run, leaving `git status` showing the file as modified after every clean deployment. Fix by committing the correct mode once:

```bash
git add --chmod=+x scripts/deploy.sh
git commit -m "Fixed: mark deploy.sh as executable in git (mode 100755)"
```

After that, `chmod 755` in the deploy script is a no-op from git's perspective.

**Database files:**

SQLite databases are binary-compatible between Windows and Linux. No conversion needed. Transfer via SCP:

```bash
scp database.db user@server:/tmp/
```

**Tools that assume a home directory:**

System users have no home directory by design. Tools that default to `~/` paths will fail. Override paths explicitly via `Environment=` directives in the service file:

| Tool | Default Path | Override |
|------|-------------|----------|
| npm | `~/.npm` | `env HOME=/srv/<project> npm ci --cache /tmp/npm-cache-<project>` |
| Playwright | `~/.cache/ms-playwright` | `Environment="PLAYWRIGHT_BROWSERS_PATH=/srv/<project>/data/playwright"` |

This was a real deployment fix in boekwinkeltjes (commit `d2e240b`) where the scraper failed because Playwright tried to create `~/.cache` for a user with no home directory.

**PATH for venv-based services:**

For services that invoke tools installed in the venv (beyond just the Python interpreter), set the `PATH` environment variable explicitly in the service file:

```ini
Environment="PATH=/srv/<project>/venv/bin:/usr/local/bin:/usr/bin:/bin"
```

Both boekwinkeltjes service files use this pattern because the scraper invokes Playwright from the venv.

**Evidence:** fedi-dashboard uses `.gitattributes` for line ending normalization (no `dos2unix` needed). The other three projects (artbots, fedi-monitor, boekwinkeltjes) still use `dos2unix` in their deploy scripts and should migrate to `.gitattributes` when convenient. All deploy scripts include `chmod +x` for shell scripts.

### Async HTTP and DNS Resolution

**Rule:** fedi-monitor uses `aiodns` (`aiohttp.AsyncResolver()`) for DNS resolution and limits `concurrent_requests` to 40. Both W11 and Ubuntu run identical code and configuration.

**Rationale:** On Linux, Python's `asyncio.getaddrinfo()` delegates DNS lookups to a thread pool (default ~8 workers). Under high concurrency (e.g., 80 simultaneous requests each resolving a unique domain), DNS lookups queue up and consume seconds of the request timeout budget before the HTTP request even starts. This caused ~9,900 connection/timeout errors on Ubuntu while Windows (which uses its native async DNS Client service) had near-zero errors checking the same servers.

`aiodns` uses the c-ares library for fully asynchronous DNS resolution, eliminating the thread pool bottleneck on Linux. On Windows, it works identically — no platform-specific code paths needed.

**Configuration (`config/health_checker.json`):**

```json
"concurrent_requests": 40
```

**Code (`3_health_v3.py`, TCPConnector):**

```python
connector = aiohttp.TCPConnector(
    resolver=aiohttp.AsyncResolver(),  # c-ares async DNS
    ...
)
```

**Dependency (`requirements.txt`):**

```
aiodns>=3.0.0
```

**Evidence:** Feb 2026 investigation. With 80 concurrent requests, Ubuntu's health checker recovered only 17,437/27,510 servers (63%) at 10.38 srv/s with 9,892 timeout errors. Windows recovered 26,068/27,510 (95%) at 23.58 srv/s. The difference (8,631 servers) matched the timeout error count almost exactly. Reducing concurrency to 40 and adding `aiodns` eliminates the DNS thread pool as a bottleneck on both platforms.

> **See also:** Section 23 — Large-Scale Scraping: Ubuntu Network Tuning for additional system-level fixes (file descriptor limits, TCP port range, aiohttp timeout split) that further close the Ubuntu/Windows performance gap.

---

## 14. Firewall Configuration

**Rule:** Use UFW with default-deny incoming. Always allow SSH first. Only open ports that the project actually needs.

**Rationale:** Default-deny means only explicitly allowed traffic gets through. Allowing SSH first prevents lockout. Projects that only make outbound requests (like fedi-monitor) need no inbound rules at all, reducing attack surface.

**Base configuration:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh           # ALWAYS first -- prevents lockout
```

**Per-project rules:**

| Project | Inbound Port | Rule |
|---------|-------------|------|
| artbots | 8010 | `sudo ufw allow 8010/tcp` |
| boekwinkeltjes | 8005 | `sudo ufw allow 8005/tcp` |
| fedi-dashboard | 8015 | `sudo ufw allow 8015/tcp` |
| fedi-monitor | none | Outbound-only (no rules needed) |

**Deploy script integration:** All three deploy scripts have a `--setup-firewall` flag that configures UFW rules specific to the project. This is typically run once during initial setup.

---

## 15. Logging and Monitoring

**Rule:** Use systemd journal as the primary log aggregator. Application logs go to the project's `logs/` directory as a secondary source.

**Rationale:** systemd journal captures stdout/stderr automatically without application-level configuration. It provides structured querying by unit, time range, and priority. Application-level log files provide more detailed, structured output for debugging.

**systemd journal (primary):**

```bash
# Follow live logs
journalctl -f -u <service-name>

# Today's logs
journalctl -u <service-name> --since today

# Last hour
journalctl -u <service-name> --since "1 hour ago"

# All logs for a project (wildcard)
journalctl -u "<project>-*"
```

**Service configuration for clean journal entries:**

```ini
SyslogIdentifier=<project-name>
StandardOutput=journal
StandardError=journal
```

**Journal maintenance:**

```bash
journalctl --vacuum-size=500M    # Keep journal under 500MB
journalctl --vacuum-time=30d     # Remove entries older than 30 days
```

**Disk monitoring:**

```bash
df -h /srv/<project>
du -sh /srv/<project>/data/backups/
```

**Evidence:** All three projects set `SyslogIdentifier`, `StandardOutput=journal`, and `StandardError=journal` in their service files. fedi-monitor additionally writes structured JSON logs to its `logs/` directory.

---

## 16. Web Server Pattern

*Applies to: Projects with web interfaces (artbots, boekwinkeltjes-scraper, fedi-dashboard)*

**Rule:** Development uses the framework's built-in server (localhost only, debug mode). Production uses a proper WSGI/application server with explicit worker count, timeout, and bind address.

**Rationale:** Framework dev servers are single-threaded, insecure, and not designed for production traffic. gunicorn (Python) provides proper process management, graceful restarts, and concurrent request handling.

**Python/Flask production pattern (gunicorn):**

```
/srv/<project>/wsgi.py          # Entry point: imports Flask app
```

```python
# wsgi.py
from app import create_app

app = create_app()
```

```ini
# systemd ExecStart
ExecStart=/srv/<project>/venv/bin/gunicorn \
    --bind 0.0.0.0:<port> \
    --workers 2 \
    --timeout 120 \
    wsgi:app
```

**Node.js production pattern:**

```ini
# systemd ExecStart (path is relative to WorkingDirectory)
ExecStart=/usr/bin/node src/gui/server.js
```

**Key settings:**

| Setting | Value | Rationale |
|---------|-------|-----------|
| `--bind 0.0.0.0:<port>` | Bind to all interfaces | Required for network accessibility |
| `--workers 2` | 2 worker processes | Appropriate for low-traffic internal tools |
| `--timeout 120` | 120 second timeout | Prevents hung workers from blocking the server |

**Evidence:**

| Project | Server | Port | Workers |
|---------|--------|------|---------|
| artbots | Node.js built-in http | 8010 | 1 (single-threaded) |
| boekwinkeltjes | gunicorn | 8005 | 2 |
| fedi-monitor | N/A (no web interface) | -- | -- |

---

## 17. Deployment Documentation Convention

**Rule:** Every project ships with a complete set of deployment documentation and automation.

**Rationale:** Documentation is the safety net when automation fails. A new team member should be able to deploy any project from scratch using only the docs in the repo.

**Required files:**

| File | Purpose |
|------|---------|
| `docs/deployment.md` (or `DEPLOYMENT.md`) | Comprehensive step-by-step guide covering initial setup through verification |
| `docs/deployment_quick_reference.md` | One-page cheat sheet for routine operations (daily ops, common commands) |
| `deployment/` directory | All systemd unit files (`.service` and `.timer` files) |
| `scripts/deploy.sh` | Automated deployment script following the 8-phase pattern |
| `.env.example` | Environment variable template with comments explaining each variable |
| `scripts/backup_database.py` | Standalone database backup utility (Python projects) |

**Deployment guide structure:**

Every `docs/deployment.md` must be organized into two clearly separated main sections:

**A. Deployment Updates** (comes first -- this is what you need 95% of the time)

1. Standard deployment workflow (push-then-deploy via `deploy.sh`)
2. Manual update procedure (without the deploy script)
3. Special procedure for when `deploy.sh` itself has changed
4. Rollback procedure

**B. First-Time Installation** (full setup from scratch)

1. Prerequisites and system dependencies
2. Service user creation
3. Repository clone and branch setup
4. Virtual environment / dependency installation
5. Configuration (`.env`, database setup, data migration)
6. Systemd service and timer installation
7. Firewall configuration
8. Verification and health checks

**C. Troubleshooting** (standalone section, not a sub-section of installation)

Every `docs/deployment.md` must include a dedicated Troubleshooting section covering at minimum:

1. **Service fails to start** — `systemctl status <service>` and `journalctl -u <service> -n 50`, plus the most common causes for that project
2. **Permission errors** — `chown -R <user>:<user> /srv/<project>`, `chmod 600 .env`, and a verification command
3. **Web interface not responding** — port check (`ss -tlnp | grep <port>`), service restart, `ufw status`
4. **Line ending issues** — `dos2unix` fix for scripts and service files (applies to all Windows-developed projects)
5. **Git pull blocked by local changes** — all three resolution options (reset, checkout, stash) with ✅/❌ safety labels on when to use `git reset --hard origin/main`
6. **Project-specific issues** — e.g. database locked/access denied, timer not firing, cross-service permission issues

**Rationale:** Routine updates are the most common operation. Putting them first means the reader finds what they need immediately. First-time installation is a one-time event and can be longer and more detailed. These six troubleshooting categories cover the failure modes that actually occur across the project fleet — standardising them ensures any operator can diagnose a problem on an unfamiliar project using the same mental model.

**Evidence:** artbots, fedi-monitor, and boekwinkeltjes-scraper all include these files. boekwinkeltjes has the most comprehensive documentation with 5 separate deployment-related markdown files.

> **Gap to fix:** boekwinkeltjes' `docs/deployment.md` still references `git push origin master` and `git pull origin master` in the "Deploying Code Fixes" section (Method B). These should be changed to `main`.

**Verification checklist when rewriting deployment docs:**

When updating a project's `docs/deployment.md` to match these conventions, verify the following after the rewrite:

1. **1:1 step mapping** — every old installation step that contained concrete commands must map to a new step with concrete commands, even if the commands changed completely. No step should silently disappear during restructuring.
2. **Cross-reference shared setup; inline project-specific setup** — when a step involves shared server setup documented elsewhere (e.g., `deploy_user_setup.md`), cross-reference the shared guide and include only the project-specific actions plus a verification command. Do NOT duplicate the shared guide inline (bloat), but also do NOT omit the step entirely (the previous failure mode). The step must still exist with a clear cross-reference and at least a verification command.
3. **Post-rewrite step count audit** — count the installation steps before and after the rewrite. If a step disappeared, confirm it was intentionally merged into another step (not dropped). Pay special attention to steps that changed topic (e.g., an SSH setup step becoming a directory setup step means the SSH setup was lost).

---

## 18. Update Workflow

**Rule:** Routine updates follow a push-then-deploy pattern. When the deploy script itself has changed, pull manually first.

**Rationale:** The deploy script pulls code, but it cannot update itself mid-execution. When `deploy.sh` has changes, a manual pull ensures the latest version of the script runs.

**Routine deployment:**

```
Developer (Windows) -> git push -> SSH to server -> sudo bash scripts/deploy.sh
```

**When deploy.sh itself changed:**

```bash
cd /srv/<project>
sudo git pull origin main
sudo chown -R <user>:<user> /srv/<project>
sudo bash scripts/deploy.sh
```

**Evidence:** This workflow is documented in all three projects' deployment guides and quick reference sheets. The "pull first if deploy.sh changed" caveat is called out explicitly.

---

## 19. Rollback Strategy

**Rule:** Rollback restores the latest database backup and resets Git to the previous state. Always test with `--dry-run` or `--check` before a full deployment.

**Rationale:** Database backups are the critical safety net. Code can always be re-fetched from Git, but data loss is permanent. The rollback procedure prioritizes data integrity over speed.

**Rollback procedure:**

```bash
sudo bash scripts/deploy.sh --rollback
```

This performs:

1. Stop all project services
2. Restore the most recent database backup
3. `git reset --hard` to the previous commit (or `git stash` if there were local changes)
4. Restart services
5. Verify integrity

**Pre-deployment safety:**

```bash
# Check what would happen without making changes
sudo bash scripts/deploy.sh --dry-run

# Run only pre-flight validation
sudo bash scripts/deploy.sh --check
```

**Evidence:** All three deploy scripts implement `--rollback`, `--dry-run`, and `--check` flags.

---

## 20. Service Naming Convention

**Rule:** Service and timer files follow the pattern `<project>-<component>.service` (or `.timer`). Template services use `<project>@.service` with `%i` for instance names.

**Rationale:** Consistent naming makes it possible to find all services for a project with `systemctl list-units '<project>-*'` and allows journal queries with wildcards (`journalctl -u '<project>-*'`).

**Patterns:**

| Pattern | Use Case | Example |
|---------|----------|---------|
| `<project>-<component>.service` | Standard service | `boekwinkeltjes-web.service` |
| `<project>-<component>.timer` | Timer unit | `fedi-monitor-pipeline.timer` |
| `<project>@.service` | Template (multi-instance) | `artbots@.service` (instantiated as `artbots@monet`, `artbots@vangogh`, etc.) |

**Evidence:**

| Project | Services | Timers |
|---------|----------|--------|
| artbots | `artbots-gui.service`, `artbots@.service`, `artbots-social-stats.service` | `artbots-curtis.timer`, `artbots-monet.timer`, etc. (8 total) |
| fedi-monitor | `fedi-monitor-pipeline.service`, `fedi-monitor-db-update.service`, `fedi-monitor-status.service` | `fedi-monitor-pipeline.timer`, `fedi-monitor-db-update.timer` |
| boekwinkeltjes | `boekwinkeltjes-web.service`, `boekwinkeltjes-scraper.service` | `boekwinkeltjes-scraper.timer` |

---

## 21. Port Allocation

**Rule:** Each project with a web interface gets a unique port in the 8000+ range. Ports are tracked in this document to prevent collisions.

**Rationale:** All projects run on the same server. Unique ports prevent binding conflicts and make firewall rules unambiguous.

**Current allocation:**

| Port | Project | Component |
|------|---------|-----------|
| 8005 | boekwinkeltjes-scraper | Web interface (gunicorn) |
| 8010 | artbots | GUI dashboard (Node.js) |
| -- | fedi-monitor | No web interface (outbound-only) |
| 8015 | fedi-dashboard | Web dashboard (gunicorn) |

---

## 22. Cross-Project Database Access

*Applies to: fedi-dashboard (reads fedi-monitor's database)*

**Problem:** fedi-dashboard is a read-only dashboard that visualizes data from fedi-monitor's SQLite database. Under `ProtectSystem=strict`, the fedi-dashboard service needs explicit filesystem access to another project's data directory.

**Proposed approaches:**

| Approach | Directive | Pros | Cons |
|----------|-----------|------|------|
| **A. `ReadOnlyPaths`** | `ReadOnlyPaths=/srv/fedi-monitor/databases` | Enforces read-only at the systemd level; defense in depth on top of SQLite's `PRAGMA query_only` | Couples the service file to fedi-monitor's directory layout; breaks if fedi-monitor moves its database |
| **B. Shared group** | Add `fedimonitor` as supplementary group to fedi-dashboard's service user | No cross-project path in the service file; access controlled via standard Unix permissions (640 on `.db`) | Weaker isolation; if fedi-dashboard is compromised, the attacker gets group-level read access to fedi-monitor's data directory |
| **C. Symlink** | Symlink `/srv/fedi-dashboard/data/fedi-monitor.db` -> `/srv/fedi-monitor/databases/prod_fedi_2.db` | Decouples the service file from the source path; fedi-dashboard only references its own `/srv/fedi-dashboard/data/` | Symlink must be maintained manually; `ProtectSystem=strict` may block following symlinks outside `ReadWritePaths` |

**Recommendation:** Approach A (`ReadOnlyPaths`) is the most explicit and secure. It makes the cross-project dependency visible in the service file and enforces read-only access at the kernel level:

```ini
# In fedi-dashboard's service file
ReadWritePaths=/srv/fedi-dashboard/data
ReadOnlyPaths=/srv/fedi-monitor/databases
```

**Chosen approach:** Approach A (`ReadOnlyPaths`) was implemented in `fedi-dashboard-web.service`. No `ReadWritePaths` is needed since fedi-dashboard has no owned data directories.

### SQLite WAL Mode and Read-Only Access

**Problem discovered during deployment:** SQLite databases using WAL (Write-Ahead Logging) mode require write access to the `-shm` (shared memory) file, even for read-only connections. This means:

- `mode=ro` in the connection URI **fails** with `attempt to write a readonly database` when the reader has no write permission to the database directory
- `PRAGMA query_only = ON` also fails as a "write" on a truly read-only filesystem
- `ReadOnlyPaths` in systemd and Unix file permissions both block `-shm` writes

**Solution:** Use `immutable=1` in the SQLite connection URI instead of `mode=ro`:

```python
# FAILS on WAL-mode databases when user lacks write access to directory
conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
conn.execute("PRAGMA query_only = ON")  # Also fails

# WORKS -- skips WAL/SHM entirely
conn = sqlite3.connect(f"file:{db_path}?immutable=1", uri=True)
```

**Why `immutable=1` is safe for fedi-dashboard:**

- `immutable=1` tells SQLite the database file will not change during the connection
- fedi-dashboard uses per-request connections (open, query, close), so each request gets a fresh view of the data
- The staleness window is effectively zero for practical purposes
- Write protection is enforced by `immutable=1` (SQLite level) + `ReadOnlyPaths` (kernel level)

> **Note for future cross-project readers:** Any project that reads another project's WAL-mode SQLite database without write access to the database directory must use `immutable=1`. This applies to both application code and diagnostic scripts. The `mode=ro` URI parameter is insufficient.

---

## 23. Tracked Files with Intentional Local Overrides

**Rule:** When a tracked file must have different content on the server than in the repository, use `git update-index --assume-unchanged <file>` to suppress it from `git status`. Apply this in the deploy script after every `git pull`.

**Rationale:** Some files need to be tracked in version control (so new clones get a working default) but always differ on the server (e.g., configuration pointing to server-specific paths). Without `--assume-unchanged`, `git status` permanently shows these files as modified, which obscures real uncommitted changes and creates noise during deployments.

**The problem:**

```
$ git status
  modified:   config/dashboard.json    # <-- not a real change, just server config
```

This is distracting and can cause the deploy script's local-change detection (stash logic) to trigger unnecessarily.

**The solution:**

```bash
# After git pull, mark files with intentional local overrides
git update-index --assume-unchanged config/dashboard.json
```

**How it works:**

| Aspect | Behavior |
|--------|----------|
| `git status` | File is hidden from output (assumed unchanged) |
| `git diff` | File is hidden from output |
| `git pull` | If the upstream version changes, git will warn and refuse to overwrite -- the override is not silently lost |
| `git stash` | Does NOT stash assume-unchanged files |
| `git reset --hard` | DOES overwrite the file (the flag is advisory, not protective) |

**To reverse the flag** (e.g., when debugging or updating the server config intentionally):

```bash
git update-index --no-assume-unchanged config/dashboard.json
```

**To list all assume-unchanged files:**

```bash
git ls-files -v | grep '^h'
```

(Lowercase `h` means assume-unchanged; uppercase `H` means normally tracked.)

**Deploy script integration:** The `--assume-unchanged` flag is applied in Phase 4 (Git Operations) after `git pull` and before line-ending conversion. It is re-applied on every deployment to ensure the flag persists even if git operations reset it.

**Evidence:**

| Project | File | Reason |
|---------|------|--------|
| fedi-dashboard | `config/dashboard.json` | Server-specific database path differs from the development default |

**When NOT to use this approach:**

- If the file contains secrets -> use `.env` + `.gitignore` instead (principle 5)
- If the file is only needed on the server -> don't track it at all, use a `.example` template
- If multiple files need overrides -> consider whether the config model needs restructuring (e.g., a `config/local.json` overlay pattern)

> **Alternative considered:** `.git/info/exclude` (a local-only gitignore). This does not work for files already tracked by git -- it only prevents untracked files from appearing in `git status`. For tracked files with local overrides, `--assume-unchanged` is the correct tool.

---

## 24. Large-Scale Scraping: Ubuntu Network Tuning

**When to apply:** Any project that makes more than ~1,000 outbound HTTP connections per pipeline run on Ubuntu.

**Context:** Ubuntu's default kernel and system limits are tuned for general-purpose use, not for processes making tens of thousands of short-lived outbound TCP connections. On Windows, the defaults are more generous for outbound scenarios. Without tuning, Ubuntu scraping jobs produce significantly more timeouts than identical Windows runs — not due to network conditions, but due to resource exhaustion.

---

### 23.1 File descriptor limits (primary fix)

**Rule:** Raise the open-files limit to 65535 before deploying any high-volume scraper on Ubuntu.

**Rationale:** Ubuntu defaults to 1024 open files per process. Each TCP socket uses one file descriptor. At 40+ concurrent connections plus database handles and log files, this limit is hit almost immediately. New connections then stall and surface as timeout errors. Raising the limit to 65535 eliminates this class of failure entirely.

**System-level fix (`/etc/security/limits.conf`):**

```
*    soft    nofile    65535
*    hard    nofile    65535
```

Also raise the kernel-wide limit in `/etc/sysctl.conf`:

```ini
fs.file-max = 200000
```

> **Critical:** `limits.conf` changes only take effect after log out and back in (or reboot). They do NOT apply to the current shell session.

**systemd override (mandatory):**

systemd ignores `limits.conf` for services it launches. Always add `LimitNOFILE=65535` to the `[Service]` section of any service unit that runs a scraper:

```ini
[Service]
LimitNOFILE=65535
```

Without this, the service user gets the default 1024 limit regardless of `limits.conf`. This is the most commonly missed step.

---

### 23.2 aiohttp timeout split (code-level fix)

**Rule:** Never use `aiohttp.ClientTimeout(total=N)` alone for large-scale scraping. Always split into component timeouts.

**Rationale:** The `total` timeout starts when `session.get()` is called. If DNS or connection setup consumes time, the remaining budget for the HTTP response shrinks accordingly. With many queued requests, the `total` window can expire before the TCP handshake completes — producing "false" timeouts on servers that would have responded correctly.

**Anti-pattern:**

```python
timeout = aiohttp.ClientTimeout(total=10)
```

**Correct pattern:**

```python
timeout = aiohttp.ClientTimeout(
    total=10,
    connect=8,        # cap time to establish connection
    sock_connect=8,   # cap time for socket connect phase
    sock_read=8,      # cap time for response data
)
```

Setting `connect`, `sock_connect`, and `sock_read` to `total - 2` gives explicit budgets for each phase while keeping `total` as the hard ceiling.

> **Warning:** Explicit `sock_connect` makes broken IPv6 routing catastrophic. If the server has no IPv6 routing, `sock_connect=8` will be consumed entirely by a hanging IPv6 attempt with no time left for IPv4 fallback. Always verify IPv6 routing (§23.5) before applying this pattern.

---

### 23.3 aiodns (mandatory for Linux async DNS at scale)

**Rule:** Include `aiodns` in `requirements.txt` for any Python project using `aiohttp` on Linux.

**Rationale:** Without `aiodns`, Python's `asyncio.getaddrinfo()` delegates DNS to a blocking thread pool (~8 workers by default). Under high concurrency, DNS lookups queue behind each other and consume the request timeout budget before the HTTP connection starts. `aiodns` uses the c-ares library for fully asynchronous DNS, eliminating the thread pool bottleneck.

**`requirements.txt`:**

```
aiodns>=3.0.0
```

**Code (`aiohttp.TCPConnector`):**

```python
connector = aiohttp.TCPConnector(
    resolver=aiohttp.AsyncResolver(),  # c-ares async DNS
)
```

aiohttp automatically detects and uses aiodns when `AsyncResolver()` is specified. Confirm installation after `pip install -r requirements.txt`:

```bash
python -c "import aiodns; print('aiodns OK')"
```

---

### 23.4 TCP port range and TIME_WAIT (secondary fix)

**Rule:** Expand the ephemeral port range and enable TIME_WAIT socket reuse on servers making >10K outbound connections per run.

**Rationale:** Ubuntu's default ephemeral port range is 32768-60999 (~28K ports). Each closed TCP connection holds its port in TIME_WAIT for 60 seconds by default. At high volume, this can exhaust available ports, causing `connect()` to fail.

**`/etc/sysctl.d/99-scraper-tuning.conf`:**

```ini
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_tw_buckets = 1048576
```

---

### 23.5 IPv6 routing: verify before deploying any outbound scraper

**Rule:** On every new server, verify IPv6 routing before deploying a scraper. If IPv6 is not routable, force IPv4-only in the `TCPConnector`.

**Rationale:** Ubuntu servers on consumer or hosting ISPs frequently have IPv6 addresses configured at the OS level but no actual IPv6 routing upstream. `socket.getaddrinfo()` (and c-ares) will return both AAAA and A records for most fediverse servers. aiohttp then attempts IPv6 first. If the IPv6 connection silently black-holes (no RST, no ICMP unreachable — just no response), it hangs until `sock_connect` expires. This consumes the entire connection budget and the IPv4 fallback never runs. The result is thousands of timeout failures on servers that are perfectly reachable via IPv4.

**This is distinct from the FD limit problem** — the server can have a 65535 FD limit and aiodns installed and still produce mass timeouts if IPv6 routing is broken and `sock_connect` is set explicitly.

**Verification (run on every new server before first scraping run):**

```bash
curl -6 --max-time 5 https://mastodon.social
```

| Result | Meaning |
|--------|---------|
| HTTP 200 response | IPv6 routing works — no action needed |
| `Failed to connect ... after <10ms: Couldn't connect to server` | No IPv6 routing — apply fix below |
| Hangs for 5 seconds then times out | IPv6 black-hole routing — apply fix below (most dangerous case) |

**Fix — force IPv4 in `TCPConnector`:**

```python
import socket

connector = aiohttp.TCPConnector(
    ...
    family=socket.AF_INET,  # IPv4 only — server has no IPv6 routing
)
```

`family=socket.AF_INET` has two effects:
1. Tells aiohttp to only request A records from the resolver (no AAAA queries)
2. Prevents any IPv6 connection attempts regardless of what DNS returns

This is safe on servers with working IPv6 too — if you later gain IPv6 routing, remove the parameter rather than having the code silently ignore available addresses.

**Interaction with `sock_connect` timeout (§23.2):**

The `sock_connect=N` timeout makes this bug particularly destructive. Without an explicit `sock_connect`, a black-holed IPv6 attempt eventually yields to IPv4 within the `total` budget. With `sock_connect=8`, the 8-second budget is fully consumed by the hanging IPv6 attempt and the IPv4 fallback never executes.

**Evidence:** Feb 2026, fedi-monitor on hofnetserver. After adding `sock_connect=8` (§23.2 fix), Linux timeouts jumped from 7,173 to 25,423 on identical server sets. `curl -6` confirmed no IPv6 routing. Adding `family=socket.AF_INET` resolved the regression.

---

### 23.6 What NOT to do

| Setting | Why to skip |
|---------|-------------|
| `net.core.somaxconn` | Controls the incoming connection backlog on a *server accepting* connections. Zero effect for outbound client connections. |
| `net.ipv4.tcp_max_syn_backlog` | Same — server-side incoming queue. Irrelevant for outbound clients. |
| TCP buffer sizes (`rmem_max`, `wmem_max`, etc.) | Primarily benefit high-throughput long-lived connections (e.g., file transfers). Marginal improvement for thousands of short HTTP health checks. |
| BBR congestion control | Optimizes bandwidth estimation for long-lived streams. Negligible impact on short health-check requests. |
| `chattr +i /etc/resolv.conf` | Permanently immutably locks the file. Breaks system updates, NetworkManager, and any future DNS reconfiguration. Never do this. |
| Increasing concurrency to 300+ | Causes fediverse servers to rate-limit or block the scraper. Conservative concurrency (40) is intentional. |

---

### 23.7 Startup FD limit check (code-level defense)

**Rule:** Add a startup check in any long-running scraper that warns when the FD limit is below the safe threshold.

**Rationale:** Silent FD limit failures are hard to diagnose. A startup warning in the application log makes the problem immediately visible without requiring sysadmin access.

**Python pattern:**

```python
import platform

if platform.system() == "Linux":
    try:
        import resource
        soft_limit, _ = resource.getrlimit(resource.RLIMIT_NOFILE)
        if soft_limit < 4096:
            logger.warning(
                f"File descriptor soft limit is {soft_limit} — run 'ulimit -n 65535' "
                f"or apply limits.conf and add LimitNOFILE=65535 to the systemd unit file."
            )
    except Exception:
        pass
```

This is safe to include on Windows (the `platform.system()` guard prevents the Linux-only `resource` module from being imported).

---

**Evidence:** Feb 2026 comparison of fedi-monitor health scraper across 32,230 servers.

| Date | Ubuntu timeouts | Windows timeouts | Ubuntu run time | Root cause |
|------|----------------|-----------------|-----------------|-----------|
| Feb 23 | 7,173 (22.2%) | 288 (0.9%) | ~58 min | FD limits + no `sock_connect` split |
| Feb 24 (after §23.1–23.3 fixes) | 25,423 (79.2%) | 359 (1.1%) | 22 min | IPv6 black-hole + `sock_connect=8` interaction (§23.5) |
| Feb 24 (after `family=AF_INET`) | ~expected <2% | ~1% | ~15 min | — |

Feb 24 regression confirmed that `sock_connect=8` combined with broken IPv6 routing is more destructive than the original FD limit problem. `curl -6 --max-time 5 https://mastodon.social` on hofnetserver returned an immediate failure confirming no IPv6 routing. Fix: `family=socket.AF_INET` on `TCPConnector`.

---

## Quick Reference: Differences Between Projects

While the principles are shared, each project adapts them to its specific needs:

| Aspect | artbots | fedi-monitor | boekwinkeltjes |
|--------|---------|--------------|----------------|
| **Runtime** | Node.js | Python + venv | Python + venv |
| **Web server** | Node.js http module | N/A | gunicorn |
| **Port** | 8010 | None | 8005 |
| **Service type** | 1 simple + template oneshots | 3 oneshot | 1 simple + 1 oneshot |
| **Timers** | 8 (daily bot posting + daily stats) | 2 (daily + weekly) | 1 (5-min interval) |
| **Template services** | Yes (`artbots@.service`) | No | No |
| **Database** | SQLite in `data/` | SQLite in `databases/` | SQLite in `data/database/` |
| **Secrets** | Many (API tokens) | None (public APIs) | 1 (`SECRET_KEY`) |
| **Network** | Outbound + inbound 8010 | Outbound only | Outbound + inbound 8005 |
| **Special needs** | npm cache override, Sharp native module | Long-running pipeline (2h timeout), state persistence | Playwright browser, custom browser path |
