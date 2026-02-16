# Development Audit

Cross-project compliance tracking against [development_principles.md](development_principles.md).

Last updated: 2026-02-16

---

## Compliance Matrix

| # | Convention | fedi-monitor | artbots | boekwinkeltjes | fedi-dashboard |
|---|-----------|-------------|---------|----------------|----------------|
| 1 | Cross-Platform Compatibility | OK | OK | OK | OK |
| 2 | Console Output Encoding | OK | OK | OK | OK |
| 3 | Configuration Externalization | OK | OK | OK | OK |
| 4 | File Naming Convention | OK | GAP | OK | OK |
| 5 | Project Structure Convention | OK | OK | OK | OK |
| 6 | Database Access Pattern | OK | OK | OK | OK |
| 7 | Logging Architecture | GAP | OK | OK | GAP |
| 8 | Error Handling Philosophy | OK | OK | OK | OK |
| 9 | Separation of Concerns | OK | OK | OK | OK |
| 10 | Entry Points and CLI Pattern | OK | OK | OK | OK |
| 11 | Dependency Management | GAP | OK | OK | GAP |
| 12 | Testing Conventions | OK | OK | GAP | GAP |
| 13 | Git Conventions | GAP | OK | OK | OK |
| 14 | Web Application Pattern | N/A | OK | OK | OK |
| 15 | Documentation Standards | OK | OK | OK | OK |
| 16 | Local Development Setup | OK | OK | OK | OK |
| 17 | Code Style and Type Hints | GAP | OK | OK | OK |

---

## Gaps to Fix

### GAP-001: fedi-monitor missing .gitattributes

**Convention:** 13 (Git Conventions)
**Status:** Open
**Action:** Add standard `.gitattributes` template. Currently relies on the legacy `dos2unix` approach in its deploy script.

### GAP-002: fedi-dashboard has no test suite

**Convention:** 12 (Testing Conventions)
**Status:** Open
**Action:** Add tests. CLAUDE.md specifies "mandatory regression: all tests must pass after every system update" but the project has no tests to enforce this.

### GAP-003: boekwinkeltjes has no active test suite

**Convention:** 12 (Testing Conventions)
**Status:** Open
**Action:** Reactivate or rewrite tests. Currently has 1 archived file, no active tests. Same CLAUDE.md requirement as fedi-dashboard.

### GAP-004: artbots post-builder.js uses kebab-case

**Convention:** 4 (File Naming Convention)
**Status:** Open
**Action:** Rename `post-builder.js` to `post_builder.js` when convenient.

### GAP-005: fedi-monitor uses typing module imports

**Convention:** 17 (Code Style and Type Hints)
**Status:** Open
**Action:** Migrate from `from typing import List, Dict, Optional` to built-in generics (`list[str]`, `dict[str, Any]`) when Python 3.8 compatibility is no longer needed.

### GAP-006: fedi-monitor and fedi-dashboard use >= version pinning

**Convention:** 11 (Dependency Management)
**Status:** Open
**Action:** Consider switching to `==` pinning (like boekwinkeltjes) for reproducible builds, with a separate process for dependency updates.

### GAP-007: fedi-dashboard uses basic stdlib logging

**Convention:** 7 (Logging Architecture)
**Status:** Open
**Action:** Implement structured JSON logging with a custom logger class. Currently uses `logging.basicConfig` (console only). Note: fedi-dashboard intentionally reads fedi-monitor's structured JSON logs via `log_reader.py` for pipeline health visibility, which partially mitigates this gap.

### GAP-008: fedi-monitor logging is custom but not the shared pattern

**Convention:** 7 (Logging Architecture)
**Status:** Informational
**Action:** fedi-monitor's `FediLogger` class follows the convention. No action needed -- this entry is for tracking only. The convention was modeled on fedi-monitor's existing implementation.

---

## Detailed Evidence

### Convention 1: Cross-Platform Compatibility

| Project | Develops On | Deploys On | Path Handling | `.gitattributes` |
|---------|------------|------------|---------------|-------------------|
| fedi-monitor | Windows 11 | Ubuntu Linux | `os.path.join()` throughout | Not yet added |
| artbots | Windows 11 | Ubuntu Linux | `path.join()` throughout | Yes (comprehensive) |
| boekwinkeltjes | Windows 11 | Ubuntu Linux | `os.path.join()` + `pathlib.Path` | Yes |
| fedi-dashboard | Windows 11 | Ubuntu Linux | `os.path.join()` throughout | Yes |

**Real example:** fedi-monitor's `aiodns` library provides async DNS resolution that works identically on Windows and Linux -- no platform-specific code paths needed despite different underlying DNS implementations.

### Convention 2: Console Output Encoding

All four `CLAUDE.md` files contain the rule:

> "Windows console compatibility: Never use Unicode characters in console output (print statements). Windows console uses CP1252 encoding which cannot handle Unicode characters. Use ASCII alternatives only."

### Convention 3: Configuration Externalization

| Project | Config Files | Key Config | Secrets in `.env` |
|---------|-------------|------------|-------------------|
| fedi-monitor | `database.json`, `discovery.json`, `filter_rules.json`, `health_checker.json`, `nodeinfo_scraper.json` (5 files) | Environment paths, performance tuning, filter rules | None (public API scraper) |
| artbots | `accounts.json` | 7 bot accounts with templates, API handles, composite settings | Many (Bluesky passwords, Mastodon tokens per account) |
| boekwinkeltjes | `settings.py`, `sites.json`, `logging.json`, `url_sanitization.json`, `filters/*.json` | URL configs, per-URL filter rules, scrape intervals | `SECRET_KEY` (Flask sessions) |
| fedi-dashboard | `dashboard.json` | fedi-monitor path, environment, server settings | None |

**Config validation with range checking** (fedi-monitor `health_checker.json`):

```python
if self.concurrent_requests < 1 or self.concurrent_requests > 200:
    errors.append(f"concurrent_requests must be 1-200, got {self.concurrent_requests}")
```

### Convention 4: File Naming Convention

| Project | Source Files | Config Files | Test Files | Notes |
|---------|-------------|-------------|------------|-------|
| fedi-monitor | `database_manager.py`, `logging_module.py`, `daily_metrics_collector.py` | `database.json`, `health_checker.json` | `test_database_manager.py`, `test_domain_matching.py` | Numbered pipeline scripts: `0_run_pipeline_v3.py` |
| artbots | `artbot.js`, `logger.js`, `social.js`, `image.js` | `accounts.json` | `artbot.test.js`, `database.test.js` | `post-builder.js` uses kebab-case (GAP-004) |
| boekwinkeltjes | `base_scraper.py`, `config_loader.py`, `marktplaats_scraper.py` | `sites.json`, `logging.json`, `url_sanitization.json` | (none active) | All consistent snake_case |
| fedi-dashboard | `db.py`, `queries.py`, `routes.py`, `log_reader.py` | `dashboard.json` | (none yet) | All consistent snake_case |

### Convention 5: Project Structure Convention

**Side-by-side directory comparison:**

```
fedi-monitor/              artbots/                  boekwinkeltjes/           fedi-dashboard/
  config/                    config/                   config/                   config/
    database.json              accounts.json             settings.py               dashboard.json
    health_checker.json                                  sites.json
    discovery.json                                       logging.json
    filter_rules.json                                    filters/
    nodeinfo_scraper.json
  [pipeline scripts]         src/                      src/                      app/
    0_run_pipeline_v3.py       index.js                  database.py               __init__.py
    database_manager.py        artbot.js                 scraper.py                db.py
    logging_module.py          lib/                      web_app.py                queries.py
                                 logger.js               config_loader.py          routes.py
                                 social.js               filters.py                log_reader.py
                                 image.js
                               gui/
  tests/                     tests/                    (archive/)                (none yet)
    test_*.py (34 files)       unit/
                               integration/
  scripts/                   scripts/                  scripts/                  scripts/
  deployment/                deployment/               deployment/               deployment/
  docs/                                                docs/                     docs/
  databases/                 data/                     data/                     (no owned data)
  logs/                        logs/                     database/
  state/                       images/                   logs/
```

> **Note:** fedi-monitor keeps pipeline scripts at the project root with numbered prefixes. This is a project-specific convention for pipeline stage ordering, not a general pattern to follow.

### Convention 6: Database Access Pattern

| Project | Library | Pattern | Row Factory | Retry Logic | WAL Mode |
|---------|---------|---------|-------------|-------------|----------|
| fedi-monitor | `sqlite3` | Connection pooling (3 conns), `DatabaseManager` class | `sqlite3.Row` | Yes (3 retries, exponential backoff) | Yes |
| artbots | `promised-sqlite3` | Per-request (open in `run()`, close in `finally`) | Default | No | No |
| boekwinkeltjes | `sqlite3` | Instance-level connection in `Database` class | `sqlite3.Row` | No | No |
| fedi-dashboard | `sqlite3` | Per-request (`get_connection()` + `finally: conn.close()`) | `sqlite3.Row` | No | Reads WAL via `immutable=1` |

**Performance pragmas** (fedi-monitor `DatabaseManager`):

```python
conn.execute('PRAGMA journal_mode=WAL')
conn.execute('PRAGMA synchronous=NORMAL')
conn.execute('PRAGMA cache_size=10000')      # ~40MB
conn.execute('PRAGMA temp_store=MEMORY')
conn.execute('PRAGMA mmap_size=268435456')   # 256MB
```

These are appropriate for fedi-monitor's write-heavy pipeline workload. Read-only projects like fedi-dashboard don't need them.

### Convention 7: Logging Architecture

| Project | Logger Class | Factory | Log Dir | Transports |
|---------|-------------|---------|---------|------------|
| fedi-monitor | `FediLogger` | `get_logger(module_name)` | `logs/` | JSON file (rotating) |
| artbots | Winston logger | `createLogger(accountId)` | `data/logs/` | Console + text file + JSON file |
| boekwinkeltjes | `BookLogger` | `get_logger(module_name)` | `data/logs/` | JSON file (rotating) |
| fedi-dashboard | stdlib `logging` | `logging.getLogger(__name__)` | N/A (console only) | Console via `basicConfig` |

**File rotation settings:**

| Project | Max File Size | Backup Count | Handler |
|---------|--------------|--------------|---------|
| fedi-monitor | 10 MB | 5 | `SafeRotatingFileHandler` (custom) |
| artbots | 5 MB | 5 | Winston `File` transport |
| boekwinkeltjes | 10 MB | 5 | `SafeRotatingFileHandler` (custom) |
| fedi-dashboard | N/A | N/A | Basic `logging.basicConfig` |

**Performance tracking decorator** (fedi-monitor and boekwinkeltjes):

```python
@PerformanceTracker.performance_monitor(logger, "scrape_url")
def scrape_url(url_config, db):
    # ... automatically logs duration_ms, memory_usage_mb, memory_change_mb
```

### Convention 8: Error Handling Philosophy

| Project | Fail-Fast Examples | Graceful Degradation Examples |
|---------|-------------------|------------------------------|
| fedi-monitor | Config validation, DB existence, schema check, script existence | Per-server failures in health checker, per-item in scraper |
| artbots | Credential validation, account validation | Per-platform publishing, image download retry |
| boekwinkeltjes | Config validation (`sys.exit(1)`) | Per-URL scraping isolation |
| fedi-dashboard | DB resolution at startup | Per-widget error handling (9 independent try/except blocks) |

**Additional code examples:**

Fail fast at startup (fedi-monitor):

```python
if self.environment not in databases:
    available = list(databases.keys())
    print(f"ERROR: Environment '{self.environment}' not found in config")
    print(f"Available environments: {available}")
    sys.exit(1)
```

Fail fast at startup (artbots):

```javascript
if (!bskyPassword && !mastoToken && process.env.PUBLISH !== 'false') {
    errors.push(`No credentials found for account '${accountId}'`);
}
if (errors.length > 0) {
    process.exit(2);
}
```

Graceful degradation -- per-widget (fedi-dashboard `routes.py`):

```python
try:
    hero_stats = queries.get_server_stats()
except Exception as e:
    logger.error("Failed to load server stats: %s", e)
    hero_stats = {"total_servers": 0, "operating_servers": 0}
```

Graceful degradation -- per-URL (boekwinkeltjes `run_scraper.py`):

```python
try:
    books = scraper.scrape_latest_books()
except Exception as e:
    logger.error(f"Failed to scrape URL: {e}")
    return stats  # Empty stats, continue with next URL
```

### Convention 9: Separation of Concerns

**Module responsibility mapping:**

| Responsibility | fedi-monitor | artbots | boekwinkeltjes | fedi-dashboard |
|---------------|-------------|---------|----------------|----------------|
| Database access | `database_manager.py` | Inline in `artbot.js` | `src/database.py` | `app/db.py` |
| Configuration | `config/*.json` + inline loading | `config/accounts.json` + dotenv | `src/config_loader.py` + `config/settings.py` | `app/db.py` (`load_dashboard_config`) |
| Business logic | Pipeline scripts (`1_discovery_v3.py`, etc.) | `src/artbot.js` | `src/scraper.py`, `src/marktplaats_scraper.py` | `app/queries.py` |
| Web routes | N/A | `src/gui/routes/` | `src/web_app.py` | `app/routes.py` |
| Logging | `logging_module.py` | `src/lib/logger.js` | `logging_module.py` | stdlib `logging` |
| Filtering | `2_filter_v3.py` | N/A | `src/filters.py` | N/A |
| Entry point | `0_run_pipeline_v3.py` | `src/index.js` | `run_scraper.py`, `run_web.py` | `run.py` |

### Convention 10: Entry Points and CLI Pattern

| Project | Entry Point | Environment Selection | Dry-Run | Key Flags |
|---------|------------|----------------------|---------|-----------|
| fedi-monitor | `python 0_run_pipeline_v3.py` | `--environment production\|test\|development` (required) | `--dry-run` | `--noscraper`, `--fresh-start`, `--limit N`, `--stats` |
| artbots | `node src/index.js <account>` or `npm run <account>` | `NODE_ENV` env var | `PUBLISH=false` or `--dry-run` | `--help`, `--list` |
| boekwinkeltjes | `python run_scraper.py` (scraper), `python run_web.py` (web) | `BOEKWINKELTJES_ENV` env var | N/A | N/A |
| fedi-dashboard | `python run.py` (dev), `gunicorn wsgi:app` (prod) | `config/dashboard.json` `environment` field | N/A | N/A |

### Convention 11: Dependency Management

| Project | Language | Deps Count | Pinning Style | Key Packages | Build Tools |
|---------|----------|-----------|---------------|--------------|-------------|
| fedi-monitor | Python | 5 | `>=` (minimum version) | `aiohttp`, `aiodns`, `requests`, `psutil`, `urllib3` | None |
| artbots | Node.js | 8 prod + 2 dev | `^` (compatible) | `@atproto/api`, `masto`, `winston`, `sharp`, `promised-sqlite3`, `express` | None |
| boekwinkeltjes | Python | 6 | `==` (exact) | `flask`, `requests`, `beautifulsoup4`, `playwright`, `gunicorn`, `python-dotenv` | None |
| fedi-dashboard | Python | 2 | `>=` (minimum version) | `flask`, `gunicorn` | None |

**Frontend CDN usage:**

| Library | Used By | CDN |
|---------|---------|-----|
| Pico CSS | fedi-dashboard | `cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css` |
| Chart.js | fedi-dashboard | CDN link in `base.html` |

### Convention 12: Testing Conventions

| Project | Framework | Test Files | Active? |
|---------|-----------|-----------|---------|
| fedi-monitor | `unittest` | 34 files in `tests/` | Yes |
| artbots | `node --test` | 8+ files in `tests/unit/` and `tests/integration/` | Yes |
| boekwinkeltjes | pytest (planned) | 1 archived file | No active tests (GAP-003) |
| fedi-dashboard | (none) | None | No (GAP-002) |

**Test commands:**

| Project | Run All Tests | Run Unit Only | Run Integration Only |
|---------|--------------|---------------|---------------------|
| fedi-monitor | `python -m pytest tests/` | `python -m pytest -m unit` | `python -m pytest -m integration` |
| artbots | `npm test` | `npm run test:unit` | `npm run test:integration` |
| boekwinkeltjes | `python -m pytest tests/` (planned) | N/A | N/A |
| fedi-dashboard | (none yet) | N/A | N/A |

### Convention 13: Git Conventions

| Project | `.gitattributes` | Commit Prefixes | `.gitignore` | Branch |
|---------|-----------------|-----------------|-------------|--------|
| fedi-monitor | Not yet added (GAP-001) | `Fixed:`, `New:`, `Updated:` | Comprehensive | `main` |
| artbots | Yes (comprehensive -- includes binary rules) | `Fixed`, `New:`, `Update:` | Comprehensive | `main` |
| boekwinkeltjes | Yes (includes `.timer` and `.env.example` rules) | `Fixed:`, `New:`, `Updated:` | Comprehensive | `main` |
| fedi-dashboard | Yes (minimal but correct) | `Fixed:`, `New:` | Comprehensive | `main` |

**`.gitignore` standard set** (present across all projects):

```
# Language artifacts
__pycache__/
*.py[cod]
node_modules/

# Runtime data
data/
logs/
*.db
*.db-journal

# Environment
venv/
.env

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

### Convention 14: Web Application Pattern

| Project | Web Framework | CSS Framework | Templating | Charts | Port |
|---------|--------------|---------------|-----------|--------|------|
| fedi-dashboard | Flask + gunicorn | Pico CSS (CDN) | Jinja2 (`base.html` -> `dashboard.html`) | Chart.js (CDN) | 8015 |
| boekwinkeltjes | Flask + gunicorn | Custom CSS | Jinja2 (with `config/` subtemplates) | N/A | 8005 |
| artbots | Express | Custom CSS | Express templates (`src/gui/`) | N/A | 8010 |

**Custom Jinja2 filters:**

| Project | Filter | Purpose |
|---------|--------|---------|
| fedi-dashboard | `eu_number` | Format integers with dots as thousands separators (European notation) |
| boekwinkeltjes | `european_date` | Format datetime as DD-MM-YYYY HH:MM |

### Convention 15: Documentation Standards

| Project | `CLAUDE.md` | `README.md` | `docs/` | `.env.example` |
|---------|------------|------------|---------|---------------|
| fedi-monitor | 556 lines | Yes | Yes (operations guides) | Yes |
| artbots | 119 lines | Yes | No (uses `DEPLOYMENT.md` at root) | Yes |
| boekwinkeltjes | 130 lines | Yes | Yes (deployment docs) | Yes |
| fedi-dashboard | 148 lines | Yes | Yes (`deployment.md`, `DEPLOYMENT_QUICK_REFERENCE.md`) | Yes (`.env.example` + `dashboard.json.example`) |

**CLAUDE.md structure** (shared across all four projects):

All four `CLAUDE.md` files share a common "Code of Conduct" section covering:

1. **Claude's Attitude** -- Be brutally honest, explicit about confidence, ask questions
2. **Problem-Solving & Self-Assessment Protocol** -- Stop and assess, confidence ratings 1-10, challenge solutions
3. **Technical Problem-Solving Standards** -- Evidence vs plausibility, production-level scrutiny
4. **Core Principles** -- Perfect documentation, honest uncertainty, collaborative partnership
5. **Professional Standards** -- Expertise through precision, transparency, documentation excellence
6. **Quality Assurance** -- Verify before claiming, document assumptions, flag but don't fix out-of-scope issues

Each CLAUDE.md then adds project-specific sections for tech stack, structure, architecture, and database access.

### Convention 16: Local Development Setup

| Project | Venv/npm | Config Step | Dev Server Command | Dev Port |
|---------|----------|------------|-------------------|----------|
| fedi-monitor | `python -m venv venv` + `pip install -r requirements.txt` | Copy `.env.example` | `python 0_run_pipeline_v3.py --environment development` | N/A |
| artbots | `npm install` | Copy `.env.example` | `npm run gui` (web) or `npm run <account>:dry` | 8010 |
| boekwinkeltjes | `python -m venv venv` + `pip install -r requirements.txt` | Copy `.env.example`, adjust settings | `python run_web.py` | 8005 |
| fedi-dashboard | `python -m venv venv` + `pip install -r requirements.txt` | Copy `config/dashboard.json.example` to `config/dashboard.json`, set local `fedi_monitor_path` | `python run.py` | 5000 |

### Convention 17: Code Style and Type Hints

| Project | Type Hints Style | Module System | Notes |
|---------|-----------------|---------------|-------|
| fedi-monitor | `from typing import List, Dict, Optional` | N/A (Python) | Uses older style for Python 3.8 compatibility (GAP-005) |
| artbots | N/A (JavaScript) | ES6 (`"type": "module"`, `import`/`export`) | Node.js 18+ required |
| boekwinkeltjes | `list[str]`, `dict[str, Any]`, `str \| None` | N/A (Python) | Modern 3.9+/3.10+ style throughout |
| fedi-dashboard | Minimal (no complex type annotations) | N/A (Python) | Simple codebase, types not yet needed |

---

## Quick Reference: Cross-Project Comparison

| Aspect | fedi-monitor | artbots | boekwinkeltjes | fedi-dashboard |
|--------|-------------|---------|----------------|----------------|
| **Language** | Python | Node.js | Python | Python |
| **Framework** | Custom pipeline | Express (GUI) | Flask | Flask |
| **Test runner** | unittest | `node --test` | pytest (planned) | (none yet) |
| **Database module** | `database_manager.py` (pooled, retry) | `promised-sqlite3` (per-request) | `src/database.py` (instance-level) | `app/db.py` (per-request, immutable) |
| **Logger** | `FediLogger` + `get_logger()` | Winston + `createLogger()` | `BookLogger` + `get_logger()` | stdlib `logging` |
| **Config files** | 5 JSON files | `accounts.json` | `settings.py` + 4 JSON files + `filters/*.json` | `dashboard.json` |
| **Web framework** | N/A | Express (port 8010) | Flask + gunicorn (port 8005) | Flask + gunicorn (port 8015) |
| **Entry point** | `0_run_pipeline_v3.py --environment` | `node src/index.js <account>` | `run_scraper.py` / `run_web.py` | `run.py` / `wsgi.py` |
| **Dependencies** | 5 packages | 8 + 2 dev | 6 packages | 2 packages |
| **Type hints** | `typing` module | N/A (JS) | 3.9+ built-ins | Minimal |
| **CSS** | N/A | Custom | Custom | Pico CSS (CDN) |
| **`.gitattributes`** | Not yet | Yes | Yes | Yes |
| **Active tests** | 34 files | 8+ files | None | None |
