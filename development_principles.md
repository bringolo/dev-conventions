# Development Principles

Cross-project conventions shared by **artbots**, **fedi-monitor**, **boekwinkeltjes-scraper**, and **fedi-dashboard**. This document defines how code is written, structured, configured, and tested. For how code reaches the server, see [deployment_principles.md](deployment_principles.md).

For per-project compliance status, see [development_audit.md](development_audit.md).

---

## Table of Contents

1. [Cross-Platform Compatibility](#1-cross-platform-compatibility)
2. [Console Output Encoding](#2-console-output-encoding)
3. [Configuration Externalization](#3-configuration-externalization)
4. [File Naming Convention](#4-file-naming-convention)
5. [Project Structure Convention](#5-project-structure-convention)
6. [Database Access Pattern](#6-database-access-pattern)
7. [Logging Architecture](#7-logging-architecture)
8. [Error Handling Philosophy](#8-error-handling-philosophy)
9. [Separation of Concerns](#9-separation-of-concerns)
10. [Entry Points and CLI Pattern](#10-entry-points-and-cli-pattern)
11. [Dependency Management](#11-dependency-management)
12. [Testing Conventions](#12-testing-conventions)
13. [Git Conventions](#13-git-conventions)
14. [Web Application Pattern](#14-web-application-pattern)
15. [Documentation Standards](#15-documentation-standards)
16. [Local Development Setup](#16-local-development-setup)
17. [Code Style and Type Hints](#17-code-style-and-type-hints)

---

## 1. Cross-Platform Compatibility

**Rule:** All application code must run on Windows, Linux, and macOS without modification. Only deployment scripts (bash, systemd) are platform-specific.

**Rationale:** All projects are developed on Windows 11 and deployed on Ubuntu Linux. Code that assumes one platform's conventions will break silently on the other. Cross-platform correctness must be the default.

**Path handling:**

| Language | Preferred | Also acceptable |
|----------|-----------|-----------------|
| Python | `os.path.join("config", "database.json")` | `pathlib.Path("config") / "database.json"` |
| Node.js | `path.join("config", "accounts.json")` | -- |

Forward-slash paths like `"config/database.json"` work in Python on all platforms, but `os.path.join` (or `pathlib.Path`) is preferred practice because it makes cross-platform intent explicit.

**Line endings:** Use `.gitattributes` to normalize line endings at the Git level. See [Git Conventions](#13-git-conventions) for the standard template.

**No platform-specific APIs** without cross-platform fallbacks. When a library only works on one platform, either find a cross-platform alternative or isolate the platform-specific code behind a conditional.

---

## 2. Console Output Encoding

**Rule:** Never use Unicode characters in print/console output. ASCII only.

**Rationale:** Windows console uses CP1252 encoding by default. Printing Unicode characters causes `UnicodeEncodeError` crashes. This was discovered through real crashes across multiple projects.

**Substitution table:**

| Intended | Unicode (crashes) | ASCII (safe) |
|----------|-------------------|--------------|
| Right arrow | `\u2192` | `->` |
| Checkmark | `\u2713` | `+` or `[OK]` |
| Euro sign | `\u20AC` | `EUR` |
| Em-dash | `\u2014` | `--` |
| Bullet | `\u2022` | `-` or `*` |

**Applies to:** Python `print()`, Node.js `console.log()`/`console.error()`, logger output that may reach console transports, and error messages.

---

## 3. Configuration Externalization

**Rule:** All configuration lives in dedicated config files. Secrets live in `.env` (never committed). No hardcoded paths, URLs, intervals, or credentials anywhere in application code.

**Rationale:** Hardcoded values create invisible coupling between code and environment. External configuration creates a single source of truth that can be adjusted without touching code.

**Config file formats:**

| Format | When to use |
|--------|-------------|
| JSON (`config/*.json`) | Default for all projects. Simple, universally parseable, no implicit type coercion. |
| Python module (`config/settings.py`) | Acceptable when config requires computed values or conditional logic. |

**Secrets:** `.env.example` committed as a template showing required variables with placeholder values. The real `.env` is listed in `.gitignore` and never committed.

**Config chain pattern:** Configuration files can reference other configuration files, forming a resolution chain:

```
dashboard.json -> fedi-monitor/config/database.json -> databases/prod_fedi_2.db
```

This indirection allows projects to share configuration without duplicating values.

**Example -- environment-aware config:**

```python
ENVIRONMENT = os.getenv('BOEKWINKELTJES_ENV', 'development').lower()
IS_PRODUCTION = ENVIRONMENT == 'production'

if IS_PRODUCTION:
    DATA_DIR = Path(os.getenv('BOEKWINKELTJES_DATA_DIR', '/srv/boekwinkeltjes/data'))
else:
    DATA_DIR = Path(os.getenv('BOEKWINKELTJES_DATA_DIR', PROJECT_DIR / 'data'))
```

---

## 4. File Naming Convention

**Rule:** All filenames use `lowercase_snake_case`. No ALL_CAPS, no hyphens in source files, no camelCase.

**Rationale:** Consistent naming eliminates cognitive load when navigating any project. Snake_case is the Python convention and transfers cleanly to JSON config files, documentation, and test files. The informal motto: "We don't yell at each other in this project."

**Applies to:** Python files, JavaScript files, config files, test files, documentation, and task files.

**Example:** `database_manager.py`, `config_loader.py`, `health_checker.json`, `test_domain_matching.py`

**Naming patterns:**

- Script prefixes for execution order: `0_run_pipeline_v3.py`, `1_discovery_v3.py`
- Descriptive compounds: `database_manager`, `config_loader`, `marktplaats_scraper`
- Test files mirror source: `test_database_manager.py` (Python), `artbot.test.js` (Node.js)

---

## 5. Project Structure Convention

**Rule:** All projects follow a consistent directory layout with clear separation between source code, configuration, runtime data, deployment, and documentation.

**Rationale:** A uniform layout means anyone who has worked on one project can navigate any other without learning a new structure. It also allows shared tooling (deploy scripts, CI checks) to make assumptions about where things live.

**Standard directories:**

| Directory | Purpose | Gitignored? |
|-----------|---------|-------------|
| `config/` | Configuration files (single source of truth) | No |
| `src/` or app modules | Application source code | No |
| `templates/` | Jinja2 HTML templates (web projects) | No |
| `static/` | CSS, JS, images (web projects) | No |
| `tests/` | Test suite | No |
| `scripts/` | Utility scripts, `deploy.sh` | No |
| `deployment/` | Systemd unit files | No |
| `docs/` | Documentation | No |
| `data/` | Runtime data, databases, logs | **Yes** |
| `venv/` or `node_modules/` | Dependencies | **Yes** |

---

## 6. Database Access Pattern

*Applies to: projects with a database.*

**Rule:** Direct SQLite via `sqlite3` (Python) or `promised-sqlite3` (Node.js). No ORM, no query builder. Per-request connections for read-only access. Connection pooling with retry logic for write-heavy workloads.

**Rationale:** An ORM adds complexity, obscures the actual queries being run, and provides no benefit for projects that use a single SQLite database with straightforward queries. Direct SQL keeps queries visible, debuggable, and performant.

**Per-request pattern** (read-only or light workloads): Open a connection, execute the query, close in a `finally` block. Simple and safe. Use `sqlite3.Row` for dict-like column access.

```python
def get_connection():
    db_path = resolve_db_path()
    conn = sqlite3.connect(f"file:{db_path}?immutable=1", uri=True)
    conn.row_factory = sqlite3.Row
    return conn

def get_server_stats():
    conn = get_connection()
    try:
        row = conn.execute("SELECT ... FROM daily_metrics ...").fetchone()
        return {"total_servers": row["total_servers"]}
    finally:
        conn.close()
```

**Pooled pattern with retry** (write-heavy workloads): Use a connection pool with exponential backoff for transient `OperationalError`s. Appropriate for pipeline workloads with concurrent writes.

```python
class DatabaseManager:
    def execute_query(self, query, params=None, fetch=False):
        for attempt in range(self.retry_count):
            try:
                conn = self._get_connection()
                cursor = conn.execute(query, params or ())
                conn.commit()
                if fetch:
                    return cursor.fetchall()
            except sqlite3.OperationalError:
                time.sleep(self.backoff_factor * (2 ** attempt))
```

**Cross-project read-only access:** When reading another project's WAL-mode database, use `immutable=1` in the connection URI, **not** `mode=ro`. The `mode=ro` parameter still requires write access to the `-shm` (shared memory) file. `immutable=1` skips WAL/SHM entirely -- safe when using per-request connections because each request gets a fresh view of the data.

**Row factory:** All Python projects use `sqlite3.Row` for dict-like column access: `row["column_name"]`.

---

## 7. Logging Architecture

**Rule:** Structured JSON logging with file rotation and module-specific logger instances. Every log entry is a valid JSON object with timestamp, level, module, message, and context fields.

**Rationale:** Structured logs are machine-parseable, which enables monitoring, alerting, and cross-module correlation. File rotation prevents disk exhaustion. Module-specific loggers make it easy to filter output by component during debugging.

**Python pattern:**

```python
class FediLogger:  # or BookLogger
    def __init__(self, module_name: str, log_dir: str = "logs"):
        self.module_name = module_name
        self.logger = self._setup_logger()

    def info(self, message: str, context: dict = None):
        self._log_with_context("INFO", message, context)

def get_logger(module_name: str) -> FediLogger:
    return FediLogger(module_name)
```

**Node.js pattern:**

```javascript
export function createLogger(accountId) {
    return winston.createLogger({
        level: process.env.LOG_LEVEL || 'info',
        defaultMeta: { account: accountId },
        transports: [
            new winston.transports.Console({ format: logFormat }),
            new winston.transports.File({ format: jsonFormat, ... }),
        ]
    });
}
```

**JSON log entry format:**

```json
{
    "timestamp": "2025-02-15T16:30:45.123456Z",
    "level": "INFO",
    "module": "pipeline_runner",
    "message": "Pipeline initialized for environment: development",
    "context": {}
}
```

**File rotation:** Use rotating file handlers (10 MB max, 5 backups). Both fedi-monitor and boekwinkeltjes use a custom `SafeRotatingFileHandler` that catches `OSError`/`PermissionError` during rotation when multiple processes log concurrently.

---

## 8. Error Handling Philosophy

**Rule:** Fail fast at system boundaries, degrade gracefully within components.

**Rationale:** Startup failures (missing config, unreachable database, invalid credentials) should crash immediately with a clear error message. Once the system is running, individual component failures should be isolated -- one widget failing should not take down the entire dashboard, and one URL timing out should not stop the scraper from processing the remaining URLs.

**Fail fast at startup:**

```python
try:
    db_path = resolve_db_path()
    logger.info("Database resolved: %s", db_path)
except (FileNotFoundError, ValueError) as e:
    logger.error("Database resolution failed: %s", e)
    sys.exit(1)
```

**Graceful degradation within components:**

```javascript
try {
    results.bluesky = await this.publishToBluesky(statusText, imagePath, altText);
} catch (error) {
    this.logger.error('Bluesky publish failed', { error: error.message });
}
try {
    results.mastodon = await this.publishToMastodon(statusText, imagePath, altText);
} catch (error) {
    this.logger.error('Mastodon publish failed', { error: error.message });
}
if (!results.bluesky && !results.mastodon) {
    throw new Error('Failed to publish to any platform');
}
```

**Retry transient errors** with exponential backoff for operations that can reasonably fail transiently (database locks, network requests, file system contention).

---

## 9. Separation of Concerns

**Rule:** Single responsibility per module/file. Database logic, configuration loading, route handling, logging, and filtering each live in their own module.

**Rationale:** When every module has one job, bugs are easier to locate, changes are less likely to have unintended side effects, and new team members can understand the system by reading one module at a time.

**Standard module responsibilities:**

| Role | Description |
|------|-------------|
| Database access | Connection management, query execution, row factory setup |
| Configuration | Loading and validating config files, resolving config chains |
| Business logic | Core application functionality (pipeline stages, scraping, data processing) |
| Web routes | HTTP request handling, template rendering, response formatting |
| Logging | Logger setup, format configuration, rotation settings |
| Entry point | CLI argument parsing, startup validation, application bootstrap |

**Key separations to maintain:**

- Routes should call query functions, not write SQL directly
- Query functions should use the database module, not open connections themselves
- Configuration loading should be separate from business logic
- Logging setup should be in a dedicated module, not scattered across files

---

## 10. Entry Points and CLI Pattern

**Rule:** Explicit parameters, environment-aware, dry-run capability where relevant. No silent fallbacks to default values unless explicitly designed and agreed upon by the developer.

**Rationale:** Every entry point should make its requirements explicit. Every configuration value should be either explicitly provided or explicitly designed as a default. Magic defaults that silently pick an environment or database are dangerous. The `--environment` flag is one way to achieve this; the principle is that the developer always knows what values are in effect and why.

**Python CLI example:**

```python
parser = argparse.ArgumentParser(description="Fediverse server health checker")
parser.add_argument('--environment', required=True,
                    choices=['production', 'test', 'development'],
                    help='Database environment to use')
parser.add_argument('--dry-run', action='store_true',
                    help='Show what would happen without making changes')
```

**Node.js example:**

```json
{
    "scripts": {
        "monet": "node src/index.js monet",
        "monet:dry": "cross-env PUBLISH=false node src/index.js monet",
        "test": "node --test tests/**/*.test.js"
    }
}
```

**Exit code convention:**

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Operational failure (runtime error) |
| `2` | Configuration or argument error |

---

## 11. Dependency Management

**Rule:** Minimal dependencies, explicitly pinned, no unnecessary build tools. Frontend dependencies via CDN only -- no npm/webpack/bundler for client-side code.

**Rationale:** Every dependency is a liability -- it can break, introduce vulnerabilities, or require maintenance. Pin versions for reproducibility. Use CDN for frontend libraries because they require zero build infrastructure and are cached across sites.

**Python:** `requirements.txt` with pinned versions. Always `pip install --upgrade pip` before installing. Prefer `==` pinning for reproducible builds.

**Node.js:** `package.json` + `package-lock.json`. ES6 modules enabled with `"type": "module"`.

**Frontend CDN:** No `node_modules` for client-side code, no transpilers, no build steps. Use SRI (Subresource Integrity) hashes when available for CDN resources.

```html
<link rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css"
      integrity="sha384-..."
      crossorigin="anonymous">
```

---

## 12. Testing Conventions

**Rule:** Framework-appropriate testing, organized by type, with mandatory regression. All tests must pass after every change.

**Rationale:** Tests catch regressions before they reach production. Organizing by type (unit, integration) allows running fast tests frequently and slow tests less often.

**Frameworks:**

| Language | Framework | Test naming |
|----------|-----------|-------------|
| Python | `pytest` (or `unittest`) | `test_*.py` |
| Node.js | Built-in `node --test` | `*.test.js` |

**Test organization:**

```
tests/
  unit/           # Fast, no external I/O
  integration/    # Requires database or network
  fixtures/       # Test data
```

**Python patterns:** Use `tempfile.NamedTemporaryFile()` for isolated test databases. Explicit `setUp()`/`tearDown()` or pytest fixtures for cleanup. Descriptive test method names: `test_basic_insert_and_select`.

**Node.js patterns:** Use `node:test` built-in module (`import { describe, it } from 'node:test'`). Use `node:assert` for assertions. No external test framework dependency.

---

## 13. Git Conventions

**Rule:** `main` branch as default. `.gitattributes` for line ending normalization. Conventional commit prefixes. SSH remotes only.

**Rationale:** Consistent Git practices prevent the "it works on my machine" class of bugs. `.gitattributes` solves line endings at the source. Commit prefixes make the git log scannable.

**`.gitattributes` standard template:**

```
# Auto-detect text files and normalize line endings (LF in repo)
* text=auto

# Force LF for files that must work on Linux
*.sh text eol=lf
*.py text eol=lf
*.js text eol=lf
*.service text eol=lf
*.timer text eol=lf
```

Note: `* text=auto` handles `.json`, `.md`, and other text files automatically. Explicit `eol=lf` is only needed for files that must be LF on checkout (scripts, service files, source code executed on Linux).

**After adding `.gitattributes` to an existing repo:**

```bash
git add --renormalize .
git commit -m "Normalize line endings via .gitattributes"
```

**Commit prefixes:**

| Prefix | Meaning | Example |
|--------|---------|---------|
| `New:` | New feature or functionality | `New: logging info added to dashboard` |
| `Fixed:` | Bug fix | `Fixed: line endings with git attributes` |
| `Updated:` | Enhancement to existing feature | `Updated: retry logic for image downloads` |

**SSH remotes:** All repos use `git@github.com:bringolo/<repo>.git`.

---

## 14. Web Application Pattern

*Applies to: projects with a web interface.*

**Rule:** Server-side rendering with minimal client-side JavaScript. No SPA, no build step. Python/Flask + Jinja2 templates for Python projects; Express for Node.js projects. Pico CSS via CDN as the default classless CSS framework.

**Rationale:** Server-side rendering is simpler, faster for initial page load, and requires no build infrastructure. Classless CSS (Pico) provides good defaults without writing CSS classes on every element.

**Flask app factory pattern:**

```python
def create_app():
    app = Flask(__name__,
        template_folder=os.path.join(os.path.dirname(os.path.dirname(__file__)), "templates"),
        static_folder=os.path.join(os.path.dirname(os.path.dirname(__file__)), "static"),
    )
    from app.routes import dashboard_bp
    app.register_blueprint(dashboard_bp)
    return app
```

**Development vs production:**

| Mode | Server | Debug |
|------|--------|-------|
| Development | Flask built-in (`python run.py`) | Yes (auto-reload) |
| Production | gunicorn (`gunicorn --bind 0.0.0.0:<port> --workers 2 --timeout 120 wsgi:app`) | No |

**WSGI entry point** (`wsgi.py`):

```python
from app import create_app
app = create_app()
```

---

## 15. Documentation Standards

**Rule:** Documentation lives with the code. This document and `deployment_principles.md` are the authoritative sources for cross-project conventions. Each project's `CLAUDE.md` provides project-specific guidance and references these shared documents.

**Rationale:** Documentation that lives separately from code becomes stale. Keeping docs in the repo means they are versioned, reviewed, and deployed alongside the code.

**Required files per project:**

| File | Purpose |
|------|---------|
| `CLAUDE.md` | AI assistant guidance -- project-specific rules + reference to shared conventions |
| `README.md` | Human-readable project overview, setup instructions |
| `docs/` directory | Detailed documentation (deployment, architecture) |
| `.env.example` | Environment variable template with comments |

A template `CLAUDE.md` for new projects is maintained at [claude_template.md](claude_template.md).

**Code documentation:** Self-documenting code preferred over comments. Google-style docstrings for Python functions and classes. JSDoc-style comments for Node.js functions where needed. Only add comments where the logic is not self-evident.

---

## 16. Local Development Setup

**Rule:** Every project can be set up for local development with documented steps. Python projects use `venv`, Node.js projects use `npm install`. Configuration uses `.env.example` as a template.

**Rationale:** A new developer (or the same developer on a new machine) should be able to clone and run any project without guessing.

**Python:**

```bash
git clone git@github.com:bringolo/<project>.git
cd <project>
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # Linux/macOS
pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env           # Edit with local values
python run.py                  # Flask dev server with debug mode
```

**Node.js:**

```bash
git clone git@github.com:bringolo/artbots.git
cd artbots
npm install
cp .env.example .env           # Edit with API credentials
npm run gui                    # Web dashboard
npm run monet:dry              # Test bot (no actual posting)
```

---

## 17. Code Style and Type Hints

**Rule:** Language-appropriate modern style. Python uses built-in generic type hints. Node.js uses ES6 modules (`import`/`export`).

**Rationale:** Modern type hints improve IDE support and catch bugs early. ES6 modules are the JavaScript standard, replacing the legacy CommonJS `require`.

**Python type hints:**

Python 3.9+ introduced built-in generics (`list`, `dict`, `set`, `tuple`), removing the need for `from typing import List, Dict`. Python 3.10+ added the union syntax `X | Y`, replacing `Optional[X]`.

```python
# Python 3.9+ (built-in generics)
def get_servers(status: str) -> list[dict[str, Any]]:
    pass

# Python 3.10+ (union syntax)
def find_server(name: str) -> dict[str, Any] | None:
    pass

# Avoid (only needed for Python 3.8 compatibility)
from typing import List, Dict, Optional
def get_servers(status: str) -> List[Dict[str, Any]]:
    pass
```

**Node.js modules:**

```javascript
// Correct (ES6 modules)
import { AsyncDatabase } from 'promised-sqlite3';
export function createLogger(accountId) { ... }

// Avoid (CommonJS)
const sqlite3 = require('promised-sqlite3');
module.exports = { createLogger };
```

Enabled in `package.json` with `"type": "module"` and `"engines": { "node": ">=18.0.0" }`.

**Self-documenting names:** Variables and functions should describe their purpose without needing comments. `resolve_db_path()` not `get_path()`, `operating_servers` not `os`.
