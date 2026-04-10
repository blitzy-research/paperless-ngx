# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a runtime-behavioral investigation of Paperless-NGX at commit `542221a38dff`. The user seeks an empirical, source-verified characterization of the system's idle-state behavior — specifically what processes, scheduled tasks, and log entries constitute a healthy, ready Paperless-NGX instance before any code modifications are made.

- **Documentation Category**: Create new documentation
- **Documentation Type**: Technical investigation / runtime-behavior reference document
- **Target Artifact**: A new markdown file named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation/` directory of the destination repository

The user's documentation requirements decompose into the following explicit questions:

- **Q1 — Background Processes at Idle**: What processes or tasks continue executing automatically once Paperless-NGX is up, stable, and idle (no documents being processed)?
- **Q2 — Periodic Health Log Entries**: What are the actual, specific log messages that appear periodically showing the system is healthy and ready? What is their frequency, and what does each indicate?
- **Q3 — Restart/Reconnection Logs**: If part of the system is briefly interrupted and restarted, what specific log messages confirm everything has reconnected and is operational again?
- **Q4 — Continuously Running Components**: What components or processes keep running continuously to maintain Paperless-NGX in a ready state, even when no documents are being processed?

### 0.1.2 Special Instructions and Constraints

- **No source file modifications**: The user explicitly stated: "don't modify any source files and clean up any temporary artifacts when you're done." This means the output is purely observational documentation, not code changes.
- **Empirical evidence required**: The user wants "actual log entries" and "specific log messages," not theoretical descriptions. This necessitates running the system and capturing real output.
- **Specified commit**: The investigation must be performed against commit `542221a38dff06361e07976452f9aea24d210542` (Paperless-NGX v1.7.0).
- **Temporary tools permitted**: "You may use temporary helper commands or inspection tools if needed" — temporary runtime environments for observation are acceptable.
- **Clean-up mandate**: All temporary artifacts must be removed after the investigation.
- **Implementation rule (SWE-AtlasQnA-Repo)**: The output document must be placed in `blitzy/documentation/` and named `paperless-ngx_542221a38dff.md`. It must provide thinking/rationale behind answers, base conclusions on code as truth, and not modify existing source files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document background processes at idle (Q1)**, we will run the three Supervisord-managed processes (Gunicorn, document_consumer, qcluster) and observe their steady-state behavior after initial startup tasks complete, cross-referencing with `docker/supervisord.conf` and `src/paperless/settings.py`.
- To **document periodic health log entries (Q2)**, we will capture log output from the Django-Q cluster's scheduled task execution, cross-referencing with the four Django-Q `Schedule` objects created by database migrations `src/documents/migrations/1001_auto_20201109_1636.py`, `src/documents/migrations/1004_sanity_check_schedule.py`, and `src/paperless_mail/migrations/0002_auto_20201117_1334.py`.
- To **document restart/reconnection logs (Q3)**, we will individually stop and restart each of the three supervised processes and the Redis broker, capturing the exact log messages that confirm successful reconnection.
- To **document continuously running components (Q4)**, we will enumerate all persistent processes (Gunicorn master/workers, document_consumer inotify watcher, Django-Q sentinel/guard/pusher/monitor/workers) and document their roles from source code and live process observation.

### 0.1.4 Inferred Documentation Needs

Based on code analysis and live runtime observation, the following implicit documentation needs have been identified:

- **Django-Q `catch_up: False` behavior**: The `Q_CLUSTER` configuration in `src/paperless/settings.py` sets `catch_up: False`, meaning scheduled tasks that were missed during downtime are NOT retroactively executed on restart. This is critical for understanding post-restart behavior and must be documented.
- **Worker recycling semantics**: The `Q_CLUSTER` sets `recycle: 1`, meaning each Django-Q worker process is terminated and replaced after processing just one task. This creates distinctive "stopped doing work / recycled worker / ready for work" log patterns that need explanation.
- **Redis as single point of dependency**: Both the Django-Q task queue and the Channels WebSocket layer depend on Redis. Redis unavailability triggers cascading error logs from the pusher process, followed by automatic recovery via the sentinel's reincarnation mechanism. This failure/recovery pattern must be documented.
- **Inotify vs. polling duality**: The document_consumer selects between inotify (default, `CONSUMER_POLLING=0`) and polling-based file watching depending on configuration and platform support. The startup log message differs for each mode.
- **Gunicorn lifecycle hooks**: The `gunicorn.conf.py` defines `when_ready`, `pre_fork`, `pre_exec`, `worker_int`, and `worker_abort` hooks that produce specific log messages during startup and graceful restart (SIGHUP) scenarios.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a Sphinx-based documentation system with `.rst` format files, hosted on Read the Docs. The documentation covers setup, configuration, administration, API reference, and troubleshooting, but does not contain a dedicated runtime-behavior or idle-state reference.

- **Documentation framework**: Sphinx ~4.5.0 (from `Pipfile` dev-packages)
- **Documentation generator configuration**: `docs/conf.py`
- **Hosting configuration**: `.readthedocs.yml` (points to `docs/conf.py` and `docs/requirements.txt`)
- **Documentation format**: reStructuredText (`.rst`)
- **Diagram tools detected**: None in existing docs infrastructure (Mermaid will be used in the new markdown document)
- **API documentation**: `docs/api.rst` covers REST API endpoints
- **Configuration reference**: `docs/configuration.rst` documents environment variables
- **Administration guide**: `docs/administration.rst` covers operational topics

Existing documentation files inventoried:

| File | Content Area | Relevance to This Task |
|------|-------------|----------------------|
| `docs/administration.rst` | System management tasks | Partial — covers admin commands but not idle-state behavior |
| `docs/configuration.rst` | Environment variables | High — documents `PAPERLESS_REDIS`, `CONSUMER_POLLING`, task worker settings |
| `docs/setup.rst` | Installation and initial setup | Low — covers one-time setup, not steady-state runtime |
| `docs/troubleshooting.rst` | Common issues and fixes | Moderate — may touch on process failures |
| `docs/advanced_usage.rst` | Advanced features | Low — focuses on user-facing features |
| `README.md` | Project overview | Low — entry-point documentation only |

**Key finding**: No existing document in the repository describes the runtime idle-state behavior, scheduled background tasks, or process supervision architecture in the detail required by the user's questions.

### 0.2.2 Repository Code Analysis for Documentation

The following source code areas were examined to derive runtime behavior documentation:

**Supervisor Process Definitions** (`docker/supervisord.conf`):
- Three managed processes: `gunicorn`, `consumer`, `scheduler` (qcluster)
- All run as user `paperless` with stdout/stderr forwarded to container stdio

**Django-Q Scheduled Tasks** (from database migrations):
- `src/documents/migrations/1001_auto_20201109_1636.py` — Creates `Train the classifier` (HOURLY) and `Optimize the index` (DAILY) schedules
- `src/documents/migrations/1004_sanity_check_schedule.py` — Creates `Perform sanity check` (WEEKLY) schedule
- `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — Creates `Check all e-mail accounts` (every 10 MINUTES) schedule

**Runtime Configuration** (`src/paperless/settings.py`):
- `Q_CLUSTER` configuration at lines 449–457: name, catch_up, recycle, retry, timeout, workers, redis
- `LOGGING` configuration at lines 373–414: console handler (INFO level), file handlers for `paperless.log` and `mail.log`
- `CHANNEL_LAYERS` at lines 178–187: Redis-backed channel layer for WebSocket

**Startup and Initialization** (`docker/docker-prepare.sh`, `docker/docker-entrypoint.sh`):
- PostgreSQL readiness wait (conditional), Redis readiness wait (always), migrations, search index rebuild, superuser creation

**Consumer Process** (`src/documents/management/commands/document_consumer.py`):
- inotify-based or polling-based directory watching
- File validation pipeline before task enqueue

**Gunicorn Configuration** (`gunicorn.conf.py`):
- Lifecycle hook logging: `when_ready`, `pre_exec`, `worker_int`, `worker_abort`
- Worker class: `paperless.workers.ConfigurableWorker` (Uvicorn-based ASGI)

### 0.2.3 Web Search Research Conducted

No external web searches were required for this task. All runtime behavior documentation was derived from direct source code analysis and live system observation at the specified commit. The codebase itself serves as the authoritative source of truth, per the user's implementation rules.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and files contain the runtime behavior that must be documented:

- **Module: `docker/supervisord.conf`**
  - Public interface: Three `[program:*]` sections defining the supervised process model
  - Current documentation: Described briefly in existing setup docs but not from a runtime-behavior perspective
  - Documentation needed: Detailed process inventory with roles, relationships, and idle-state behavior

- **Module: `src/paperless/settings.py` (lines 449–475)**
  - Public interface: `Q_CLUSTER` configuration dictionary, `TASK_WORKERS`, `PAPERLESS_WORKER_TIMEOUT`, `PAPERLESS_WORKER_RETRY`
  - Current documentation: `docs/configuration.rst` lists env vars but does not explain their effect on runtime log output
  - Documentation needed: Explanation of how each setting drives observable behavior (recycle=1, catch_up=False, guard cycle timing)

- **Module: `src/documents/management/commands/document_consumer.py`**
  - Public interface: `Command.handle()`, `handle_inotify()`, `handle_polling()`
  - Current documentation: Brief mention in admin docs
  - Documentation needed: Startup log messages, idle-state behavior (blocking on inotify), and what triggers log output

- **Module: `gunicorn.conf.py`**
  - Public interface: Lifecycle hooks (`when_ready`, `pre_exec`, `worker_int`, `worker_abort`), bind/workers/timeout config
  - Current documentation: None specific to log output
  - Documentation needed: Exact log messages produced at startup and during graceful restart

- **Module: Django-Q cluster internals** (`django_q/cluster.py` in site-packages, version 1.3.9)
  - Public interface: `guard()`, `pusher()`, `monitor()`, `scheduler()` functions
  - Current documentation: Django-Q upstream docs (not project-specific)
  - Documentation needed: Mapping of log messages to internal processes (sentinel guard cycle, task scheduling, worker recycling)

- **Module: `src/documents/tasks.py`**
  - Public interface: `train_classifier()`, `index_optimize()`, `sanity_check()`, `consume_file()`
  - Current documentation: None for scheduled task log output
  - Documentation needed: What each scheduled task logs on completion at idle (no documents)

- **Module: `src/paperless_mail/tasks.py`**
  - Public interface: `process_mail_accounts()`
  - Current documentation: None for idle-state behavior
  - Documentation needed: Log output when no mail accounts are configured

- **Module: `src/documents/sanity_checker.py`**
  - Public interface: `check_sanity()`, `SanityCheckMessages.log_messages()`
  - Current documentation: None for periodic execution output
  - Documentation needed: The specific "Sanity checker detected no issues." log message

- **Module: `docker/wait-for-redis.py`**
  - Public interface: Redis readiness probe with retry logic
  - Current documentation: None
  - Documentation needed: Startup log messages for Redis connectivity

- **Module: `docker/docker-prepare.sh`**
  - Public interface: PostgreSQL wait, Redis wait, migrations, index rebuild, superuser setup
  - Current documentation: Partially in setup docs
  - Documentation needed: Initialization sequence log messages

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gap exists:

- **No runtime idle-state behavior reference**: The entire topic of "what does the system do when it's running but idle" is undocumented
- **No scheduled task log reference**: The four Django-Q scheduled tasks produce specific log patterns that are not documented anywhere
- **No process supervision architecture reference**: The three-process supervisord model and its internal sub-processes (sentinel, pusher, monitor, workers) are not described from a runtime perspective
- **No restart/recovery behavior reference**: The system's self-healing mechanisms (Django-Q sentinel reincarnation, Redis reconnection) are not documented
- **No log message catalog**: Specific log messages, their formats, sources, and meanings are not cataloged anywhere in the project

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output will be a single comprehensive markdown document with the following structure:

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Introduction and Context
    │   ├── Commit and version identification
    │   └── Investigation methodology
    ├── Supervised Process Architecture
    │   ├── Process inventory (3 top-level + sub-processes)
    │   ├── Mermaid diagram of process relationships
    │   └── Role of each process at idle
    ├── Startup Sequence and Initialization Logs
    │   ├── Docker entrypoint / prepare phase
    │   ├── Gunicorn startup messages
    │   ├── Document consumer startup messages
    │   └── Django-Q cluster startup messages
    ├── Idle-State Background Activity
    │   ├── Scheduled task inventory (4 tasks with frequencies)
    │   ├── Initial burst of scheduled tasks
    │   ├── Periodic task execution logs
    │   └── Django-Q sentinel guard cycle
    ├── Specific Periodic Log Messages Reference
    │   ├── Table of log messages with frequency and meaning
    │   └── Annotated real log output samples
    ├── Restart and Reconnection Behavior
    │   ├── Gunicorn restart logs (SIGHUP and full restart)
    │   ├── Django-Q cluster stop/start logs
    │   ├── Document consumer stop/start logs
    │   └── Redis interruption and recovery logs
    ├── Continuously Running Components
    │   ├── Gunicorn master + Uvicorn workers
    │   ├── Document consumer inotify watcher
    │   ├── Django-Q sentinel (guard, pusher, monitor, workers)
    │   └── Redis broker
    └── Rationale and Source Citations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach**:
- Extract process model from `docker/supervisord.conf` and `src/paperless/settings.py`
- Extract scheduled task definitions from Django-Q `Schedule` migration files
- Extract log format from `LOGGING` configuration in `src/paperless/settings.py` (lines 373–414)
- Generate real log samples by running the system at commit `542221a38dff` with SQLite/Redis and capturing stdout/stderr
- Extract Gunicorn lifecycle messages from `gunicorn.conf.py` hook definitions
- Extract Django-Q internal process names from `django_q/cluster.py` (version 1.3.9)

**Template Application**: Not applicable — no user-provided template. The document follows the implementation rule to provide "thinking / rationale behind the answers" and "base answers on the code as truth."

**Documentation Standards**:
- Markdown with proper heading hierarchy (`#`, `##`, `###`)
- Mermaid diagrams for process architecture visualization
- Code blocks with log output samples using `text` syntax highlighting
- Tables for structured reference data (log messages, frequencies, processes)
- Source citations as inline references: `Source: path/to/file.py:LineNumber`
- Consistent use of exact log message text from live observation

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Process Architecture Diagram**: A `graph TD` showing Supervisord managing Gunicorn, Consumer, and QCluster, with QCluster's internal sub-processes (sentinel, pusher, monitor, workers)
- **Scheduled Task Timeline**: A visual representation of the four scheduled tasks and their frequencies (10 min, 1 hr, daily, weekly)
- **Startup Sequence Diagram**: A `sequenceDiagram` showing the initialization flow from docker-entrypoint through docker-prepare to the three supervised processes
- **Redis Recovery Flow**: A `sequenceDiagram` showing the pusher crash, sentinel detection, reincarnation, and successful reconnection

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `docker/supervisord.conf`, `src/paperless/settings.py`, `src/documents/management/commands/document_consumer.py`, `gunicorn.conf.py`, `src/documents/tasks.py`, `src/paperless_mail/tasks.py`, `src/documents/sanity_checker.py`, `docker/docker-prepare.sh`, `docker/docker-entrypoint.sh`, `docker/wait-for-redis.py`, `src/documents/migrations/1001_auto_20201109_1636.py`, `src/documents/migrations/1004_sanity_check_schedule.py`, `src/paperless_mail/migrations/0002_auto_20201117_1334.py`, `src/paperless/asgi.py`, `src/paperless/consumers.py`, `src/paperless/workers.py` | Complete runtime-behavior reference documenting idle-state processes, scheduled task log entries, restart/reconnection messages, and continuously running components with rationale and source citations |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Runtime Behavior Reference
Source Code: Multiple (see table above)
Sections:
    - Introduction (commit, version, methodology)
    - Supervised Process Architecture (from docker/supervisord.conf, src/paperless/settings.py:449-457)
    - Startup Sequence Logs (from gunicorn.conf.py, docker/docker-prepare.sh, document_consumer.py)
    - Idle-State Background Activity (from migrations 1001, 1004, mail 0002, src/documents/tasks.py)
    - Periodic Log Message Reference Table (from live observation, cross-referenced with source)
    - Restart and Reconnection Behavior (from live testing of stop/start cycles)
    - Continuously Running Components (from supervisord.conf, django_q/cluster.py internals)
    - Rationale and Source Citations
Diagrams:
    - Process architecture diagram (Mermaid graph)
    - Scheduled task timeline (Mermaid)
    - Startup sequence diagram (Mermaid)
    - Redis recovery flow (Mermaid)
Key Citations:
    - docker/supervisord.conf (process definitions)
    - src/paperless/settings.py:449-457 (Q_CLUSTER config)
    - src/paperless/settings.py:373-414 (LOGGING config)
    - gunicorn.conf.py (lifecycle hooks)
    - src/documents/management/commands/document_consumer.py (consumer command)
    - src/documents/tasks.py (scheduled task implementations)
    - src/paperless_mail/tasks.py (mail check task)
    - src/documents/sanity_checker.py (sanity check logic)
    - src/documents/migrations/1001_auto_20201109_1636.py (classifier + index schedules)
    - src/documents/migrations/1004_sanity_check_schedule.py (sanity check schedule)
    - src/paperless_mail/migrations/0002_auto_20201117_1334.py (mail check schedule)
    - docker/docker-prepare.sh (initialization sequence)
    - docker/wait-for-redis.py (Redis readiness probe)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output document is a standalone markdown file placed in `blitzy/documentation/` and does not integrate with the project's existing Sphinx documentation system.

### 0.5.4 Cross-Documentation Dependencies

- The new document references source file paths that are stable at the specified commit `542221a38dff`
- No navigation links or table-of-contents updates are required in existing documentation
- No shared content or includes are affected

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following runtime dependencies are relevant to the documentation exercise, as they define the processes, task queues, and logging behavior being documented:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | 4.0.4 | Core application framework hosting the management commands and settings |
| pip | django-q | 1.3.9 | Distributed task queue providing scheduled tasks, worker cluster, and sentinel |
| pip | redis | 3.5.3 | Python Redis client used by Django-Q broker and Channels layer |
| pip | channels | 3.0.4 | ASGI/WebSocket support for real-time status updates |
| pip | channels-redis | 3.4.0 | Redis-backed channel layer for WebSocket broadcasting |
| pip | gunicorn | 20.1.0 | ASGI/WSGI server managing the web application process |
| pip | uvicorn | 0.17.6 | ASGI worker class used inside Gunicorn via `ConfigurableWorker` |
| pip | watchdog | 2.1.7 | Polling-based file system observer (fallback for document consumer) |
| pip | inotifyrecursive | 0.3.5 | inotify-based directory watcher (preferred for document consumer) |
| pip | scikit-learn | 1.0.2 | ML classifier for automatic document tagging (trained hourly) |
| pip | whoosh | 2.7.4 | Full-text search index (optimized daily) |
| pip | concurrent-log-handler | 0.9.20 | Concurrent rotating log file handler |
| pip | ocrmypdf | 13.4.3 | OCR processing wrapper for Tesseract |
| system | redis-server | 6.0+ | Message broker for Django-Q and Channels |
| system | tesseract-ocr | 4.x+ | OCR engine (system binary) |
| system | supervisord | (bundled in Docker) | Multi-process manager for container deployment |

### 0.6.2 Documentation Reference Updates

Not applicable. The new document is a standalone markdown file and does not require link updates in existing documentation.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions addressed**: 4/4 (100%) — All four explicit user questions (Q1–Q4) are fully mapped to source code evidence and live observation data
- **Supervised processes documented**: 3/3 (100%) — Gunicorn, document_consumer, and qcluster
- **Django-Q sub-processes documented**: 4/4 (100%) — Sentinel/guard, pusher, monitor, workers
- **Scheduled tasks documented**: 4/4 (100%) — Train classifier (hourly), Optimize index (daily), Sanity check (weekly), Check email (10 minutes)
- **Restart scenarios documented**: 4/4 (100%) — Gunicorn (SIGHUP + full), qcluster, consumer, Redis interruption/recovery
- **Log message categories captured**: Startup, scheduled task execution, task completion, worker recycling, sentinel reincarnation, error/recovery

### 0.7.2 Documentation Quality Criteria

**Completeness requirements**:
- Every log message cited must appear verbatim from live observation or be directly traceable to a `logger.*()` call in the source code
- Every process described must have its source file cited with line numbers where applicable
- Every scheduled task must reference the migration that creates it and the function that implements it
- All four user questions must receive dedicated sections with clear, direct answers

**Accuracy validation**:
- Log messages were captured from a live running instance at the exact specified commit
- Process architecture confirmed by running `ps aux` during live observation
- Schedule frequencies confirmed by querying the Django-Q `Schedule` model at runtime
- `Q_CLUSTER` configuration verified by printing `settings.Q_CLUSTER` from the running Django environment

**Clarity standards**:
- Each log message is presented in a code block with surrounding context
- Tables summarize structured data (processes, schedules, log messages)
- Mermaid diagrams visualize architecture and flows
- Rationale sections explain "why" behind observed behavior, not just "what"

**Maintainability**:
- Source citations use relative file paths stable at the documented commit
- The document is self-contained with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum log examples per scenario**: At least one complete, real log excerpt for each of the four user questions
- **Diagram types required**: Process architecture (graph), startup sequence (sequenceDiagram), scheduled task timeline, Redis recovery flow
- **Log example verification**: All log samples were captured from a live instance running at commit `542221a38dff` with Python 3.9, Redis, and SQLite

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/paperless-ngx_542221a38dff.md` — The sole deliverable document

- **Runtime behavior topics covered**:
  - Three Supervisord-managed processes and their roles at idle
  - Django-Q cluster internal sub-processes (sentinel, pusher, monitor, workers)
  - Four scheduled background tasks and their frequencies/log output
  - Gunicorn/Uvicorn web server lifecycle and startup messages
  - Document consumer directory watcher (inotify mode) idle behavior
  - Redis broker role and connectivity requirements
  - Startup initialization sequence (docker-prepare.sh flow)
  - Restart and reconnection log messages for each component
  - Redis interruption, error cascading, and automatic recovery
  - Worker recycling behavior (`recycle: 1` setting)
  - `catch_up: False` implications for missed schedules

- **Source files analyzed** (read-only):
  - `docker/supervisord.conf`
  - `docker/docker-entrypoint.sh`
  - `docker/docker-prepare.sh`
  - `docker/wait-for-redis.py`
  - `gunicorn.conf.py`
  - `src/paperless/settings.py`
  - `src/paperless/asgi.py`
  - `src/paperless/consumers.py`
  - `src/paperless/workers.py`
  - `src/paperless/checks.py`
  - `src/documents/management/commands/document_consumer.py`
  - `src/documents/tasks.py`
  - `src/documents/sanity_checker.py`
  - `src/documents/apps.py`
  - `src/paperless_mail/tasks.py`
  - `src/documents/migrations/1001_auto_20201109_1636.py`
  - `src/documents/migrations/1004_sanity_check_schedule.py`
  - `src/paperless_mail/migrations/0002_auto_20201117_1334.py`

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No existing files in the repository will be modified (per user directive and implementation rule SWE-AtlasQnA-Repo)
- **Test file modifications**: No test files will be created or modified
- **Feature additions or refactoring**: The task is purely observational/documentary
- **Document processing pipeline behavior**: The user explicitly asked about idle state, not about what happens when documents are consumed
- **Frontend (Angular) runtime behavior**: The user's questions focus on backend/server processes
- **Deployment configuration changes**: No Docker Compose, CI/CD, or infrastructure changes
- **Existing Sphinx documentation updates**: The deliverable is a standalone markdown file in `blitzy/documentation/`, not an update to the existing `docs/` tree
- **PostgreSQL-specific behavior**: The investigation was performed with SQLite (default); PostgreSQL-specific initialization logs (pg_isready) are documented from source code but not from live observation
- **Tika/Gotenberg integration**: These optional services are feature-flagged off by default and not relevant to idle-state behavior
- **Performance benchmarking**: Resource utilization metrics are not in scope

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Runtime environment used for observation**:
  - Python 3.9.25 (highest version compatible with all pinned dependencies, notably `scikit-learn==1.0.2`)
  - Redis server (daemonized, localhost:6379)
  - SQLite3 (default database, no PostgreSQL required)
  - All Python dependencies installed from `requirements.txt` (with `psycopg2-binary` substituted for `psycopg2`)

- **Observation methodology**:
  - Start the three supervised processes individually (Gunicorn, document_consumer, qcluster)
  - Capture stdout/stderr to log files
  - Wait for initial scheduled task burst (~30 seconds after startup)
  - Wait for periodic scheduled task execution (10-minute mail check cycle)
  - Perform stop/restart of each process individually, capturing logs
  - Perform Redis shutdown/restart, capturing Django-Q error and recovery logs
  - Query Django-Q `Schedule` table to confirm task frequencies and next-run times
  - Verify `Q_CLUSTER` configuration from running `settings.Q_CLUSTER`

- **Default format**: Markdown with Mermaid diagrams
- **Citation requirement**: Every claim in the output document must reference the specific source file and line number (where applicable) or the live log capture
- **Style guide**: Follows the implementation rule: provide thinking/rationale, base answers on code as truth, do not make assumptions
- **Documentation validation**: Manual review of all log samples against source code `logger.*()` calls to confirm accuracy

## 0.10 Rules for Documentation

The following rules govern the creation of this documentation, derived from user directives and the implementation rule `SWE-AtlasQnA-Repo`:

- **Do not modify any existing source files in the repository.** The output is a new file only.
- **Clean up all temporary artifacts when done.** Any temporary runtime environments, log files, or virtual environments used during observation must be removed.
- **Base all answers on the code as the truth.** Every statement in the output document must be traceable to a specific source file, configuration, or live log capture. No assumptions.
- **Provide thinking and rationale behind the answers.** The document must explain not just what happens, but why — citing configuration settings, code paths, and architectural decisions that cause the observed behavior.
- **Place the generated document in `blitzy/documentation/` directory** with the filename `paperless-ngx_542221a38dff.md`.
- **Use the specific commit `542221a38dff`** as the investigation baseline. All observations and citations are pinned to this commit (Paperless-NGX v1.7.0).
- **Temporary helper commands and inspection tools are permitted** for observation, but must not persist in the repository.
- **Include actual, specific log messages** — not paraphrased or hypothetical. Real captured output from running the system at the specified commit.
- **Document frequencies precisely** — cite the Django-Q Schedule model and migration source that defines each interval.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were inspected to derive the conclusions documented in this Agent Action Plan:

**Root-Level Files**:
- `Pipfile` — Python dependency definitions including Django, Django-Q, Redis, Channels, Gunicorn, and all runtime packages
- `requirements.txt` — Pinned Python dependency manifest with exact versions
- `gunicorn.conf.py` — Gunicorn server configuration with lifecycle hooks (`when_ready`, `pre_exec`, `worker_int`, `worker_abort`), bind address, worker count, and worker class
- `README.md` — Project overview and feature summary
- `.readthedocs.yml` — Documentation hosting configuration

**Docker Infrastructure** (`docker/`):
- `docker/supervisord.conf` — Defines the three supervised processes: `gunicorn`, `consumer` (document_consumer), and `scheduler` (qcluster)
- `docker/docker-entrypoint.sh` — Container entrypoint with UID/GID mapping, directory creation, and language package installation
- `docker/docker-prepare.sh` — Startup coordination: PostgreSQL wait, Redis wait, migrations, search index rebuild, superuser creation
- `docker/wait-for-redis.py` — Redis readiness probe with 5 retries at 5-second intervals
- `docker/install_management_commands.sh` — Management command wrapper generator
- `docker/management_script.sh` — Shared management command template

**Backend Source** (`src/`):
- `src/paperless/settings.py` — Django settings including `Q_CLUSTER` (lines 449–457), `LOGGING` (lines 373–414), `CHANNEL_LAYERS` (lines 178–187), `CONSUMER_POLLING` (line 478), `TASK_WORKERS` (line 438)
- `src/paperless/asgi.py` — ASGI application bootstrap with `ProtocolTypeRouter` for HTTP and WebSocket
- `src/paperless/consumers.py` — `StatusConsumer` WebSocket consumer for real-time status broadcasting
- `src/paperless/workers.py` — `ConfigurableWorker` Uvicorn worker subclass for Gunicorn
- `src/paperless/checks.py` — Django system checks for paths, binaries, and debug mode
- `src/paperless/version.py` — Version tuple `(1, 7, 0)`
- `src/paperless/urls.py` — URL routing including WebSocket patterns
- `src/documents/management/commands/document_consumer.py` — Document consumer management command with inotify and polling modes
- `src/documents/tasks.py` — Scheduled task implementations: `train_classifier()`, `index_optimize()`, `sanity_check()`, `consume_file()`
- `src/documents/sanity_checker.py` — Sanity check logic and message logging
- `src/documents/apps.py` — DocumentsConfig app with signal handler registration
- `src/documents/signals/__init__.py` — Document lifecycle signals (referenced via app summary)
- `src/paperless_mail/tasks.py` — `process_mail_accounts()` scheduled task implementation
- `src/paperless_mail/mail.py` — IMAP mail handling logic (first 100 lines inspected)

**Database Migrations** (schedule definitions):
- `src/documents/migrations/1001_auto_20201109_1636.py` — Creates "Train the classifier" (HOURLY) and "Optimize the index" (DAILY) schedules
- `src/documents/migrations/1004_sanity_check_schedule.py` — Creates "Perform sanity check" (WEEKLY) schedule
- `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — Creates "Check all e-mail accounts" (every 10 MINUTES) schedule

**Documentation** (`docs/`):
- `docs/administration.rst` — Existing administration documentation (checked for coverage overlap)
- `docs/configuration.rst` — Existing configuration reference (checked for coverage overlap)

**External Library Source** (read-only, for understanding log output):
- `django_q/cluster.py` (version 1.3.9) — Guard loop, pusher, monitor, scheduler functions
- `django_q/conf.py` (version 1.3.9) — `GUARD_CYCLE` (0.5s default), `SCHEDULER` (True), `RECYCLE` (500 default, overridden to 1)

### 0.11.2 Attachments and External Resources

- **Attachments provided**: None
- **Figma screens provided**: None
- **External URLs referenced**: None — all analysis is based on the repository source code and live runtime observation at commit `542221a38dff`

### 0.11.3 Live Observation Data Summary

The following empirical data was captured during live runtime at commit `542221a38dff`:

- **Gunicorn startup log**: 4 INFO messages confirming server start, listen address, worker class, and readiness
- **Consumer startup log**: 1 INFO message confirming inotify directory watch
- **Django-Q cluster startup log**: 7 INFO messages covering cluster start, worker readiness (×2), monitor readiness, guard activation, pusher activation, and cluster running confirmation
- **Initial scheduled task burst**: 4 tasks enqueued within ~30 seconds of startup (all four schedules fire on first boot)
- **Task completion logs**: Each task produces processing/processed messages with human-readable task names
- **10-minute mail check**: Confirmed firing at the expected interval with enqueue/process/completion logs
- **Worker recycling**: After every task, the worker stops and is replaced (due to `recycle: 1`)
- **Redis interruption**: Produces rapid `ERROR Error 111 connecting to localhost:6379. Connection refused.` messages (~1/second from pusher)
- **Redis recovery**: Sentinel detects pusher death, logs `reincarnated pusher` and starts a new pusher process that successfully reconnects
- **Gunicorn SIGHUP**: Produces `Handling signal: hup`, `Hang up: Master`, and worker termination messages before spawning new workers
- **Gunicorn full restart**: Produces the same 4-message startup sequence
- **Django-Q cluster stop**: Produces 7 messages covering stop signal, process termination, monitor stop, and final cluster stopped confirmation
- **Django-Q cluster restart**: Produces a fresh 7-message startup sequence with a new cluster name

