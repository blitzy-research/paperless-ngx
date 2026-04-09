# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of architectural investigation questions about paperless-ngx's background maintenance subsystem. The user seeks a thorough, evidence-based analysis of every automatic background task, periodic schedule, health check, and maintenance mechanism in the paperless-ngx codebase.

**Documentation Type:** Technical Q&A / Architecture Analysis Document

**Category:** Create new documentation — an investigative technical document answering specific questions about the system's autonomous maintenance behavior.

**Documentation Requirements with Enhanced Clarity:**

- **Periodic Task Inventory:** Identify every task registered with the background task scheduler, including the function path, human-readable name, schedule interval, and the Django migration that registers it. The user expects a complete enumeration — not a partial list.
- **Schedule Configuration Source:** Pinpoint where schedule configuration is defined in the codebase. The user suspects Celery beat but the actual system uses django-q with schedules stored as database records created via Django data migrations — this correction is a critical finding.
- **Sanity Checker Deep Dive:** Locate the sanity checker module, enumerate every validation it performs, document its log output format and logger name, and confirm its execution frequency.
- **Index Optimization and Database Cleanup:** Determine whether automatic Whoosh index optimization exists (it does — daily) and whether any periodic database cleanup, vacuuming, or log pruning tasks are registered (they are not).
- **Failed Document Retry / Stuck Job Handling:** Investigate whether a dedicated task retries failed document consumption or unsticks stuck processing jobs. Document the retry/timeout semantics provided by django-q's `Q_CLUSTER` configuration.
- **Task Scheduler Configuration:** Extract the full `Q_CLUSTER` settings block and explain each parameter's role (workers, timeout, retry, catch_up, recycle).
- **Task Registration Tracing:** Trace through code to show exactly where and how periodic tasks are defined and registered — via Django data migrations inserting `django_q.models.Schedule` records.
- **Failure Handling and Alerting:** Document what happens when a scheduled task fails — does django-q retry, alert, or silently record the failure? Examine each task's internal exception handling.
- **Startup vs. Scheduled Tasks:** Distinguish tasks that execute at container startup (`docker/docker-prepare.sh`) from strictly periodic tasks managed by the scheduler.
- **Task Execution History:** Identify database tables that track task execution history and scheduled job state — django-q's `Task`, `Failure`/`Success`, `Schedule`, and `OrmQ` models.
- **Enable/Disable Controls:** Document every configuration knob that enables, disables, or tunes these maintenance features.

**Inferred Documentation Needs:**

- Based on code analysis: The `src/documents/tasks.py` module contains all core periodic task functions but has no inline docstrings explaining scheduling behavior — the documentation must bridge this gap.
- Based on structure: Periodic schedule registration is split across three separate Django migration files in two different apps (`documents` and `paperless_mail`) — consolidated documentation is essential for comprehension.
- Based on dependencies: The interaction between the `docker/supervisord.conf` process supervision (which starts `qcluster`), `docker/docker-prepare.sh` (which runs startup tasks), and the django-q scheduler (which manages periodic tasks) forms a three-layer execution model that requires unified documentation.
- Based on user journey: The user explicitly mentioned "Celery beat" indicating a misconception — the documentation must clearly establish that django-q (not Celery) is the task framework and explain the architectural rationale.

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**

- **Read-Only Constraint:** "Don't modify source code; observing system behavior is fine, clean up any test artifacts when done." No source files may be altered. The output is purely observational documentation.
- **Implementation Rule — SWE-AtlasQnA-Repo:** Create a new markdown document named `paperless-ngx_542221a38dff.md` that comprehensively answers all questions posed. Provide thinking and rationale behind answers. Base all answers on the code as truth — no assumptions. Place the document in the `blitzy/documentation` directory.
- **Artifact Cleanup:** Any temporary test artifacts created during investigation must be cleaned up.
- **No Assumptions:** All claims must be directly traceable to source code files with file paths and line references.

**Style Preferences:**

- Evidence-based prose with code citations
- Answers structured by question topic
- Include rationale and reasoning behind each answer
- Use Mermaid diagrams for complex relationships (task scheduling flow, startup sequence)

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document periodic tasks**, we will **create** a comprehensive inventory by tracing `Schedule.objects.create()` calls in `src/documents/migrations/1001_auto_20201109_1636.py`, `src/documents/migrations/1004_sanity_check_schedule.py`, and `src/paperless_mail/migrations/0002_auto_20201117_1334.py`, cross-referencing each with the task function implementations in `src/documents/tasks.py` and `src/paperless_mail/tasks.py`.
- To **document the schedule configuration**, we will **extract and annotate** the `Q_CLUSTER` dictionary from `src/paperless/settings.py`, explaining each parameter and its environment variable override.
- To **document the sanity checker**, we will **analyze** `src/documents/sanity_checker.py` (the `check_sanity()` function and `SanityCheckMessages` class) and its CLI wrapper `src/documents/management/commands/document_sanity_checker.py`.
- To **document startup vs. scheduled tasks**, we will **trace** the `docker/docker-prepare.sh` startup script and `docker/supervisord.conf` process definitions, contrasting them with the django-q periodic scheduler.
- To **document task failure handling**, we will **examine** exception handling patterns in each task function and django-q's built-in retry/timeout configuration.
- To **document task history tracking**, we will **identify** django-q's database models (`Task`, `Schedule`, `OrmQ`) and the application's `documents.Log` model.
- To **document enable/disable controls**, we will **catalog** every `PAPERLESS_*` environment variable and `Q_CLUSTER` setting that governs background task behavior.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with reStructuredText (`.rst`) source files under the `docs/` directory. Coverage of background maintenance topics exists but is fragmented across administration and configuration pages without a unified treatment.

**Documentation Framework:**

- **Generator:** Sphinx (configuration at `docs/conf.py`)
- **Source Format:** reStructuredText (`.rst`)
- **Source Directory:** `docs/`
- **Key Documentation Files Discovered:**

| File | Content | Relevance |
|------|---------|-----------|
| `docs/administration.rst` | Management commands: exporter, importer, retagger, classifier training, index management, sanity checker, mail fetcher | High — covers CLI interfaces to periodic task functions |
| `docs/configuration.rst` | All `PAPERLESS_*` environment variables including `PAPERLESS_TASK_WORKERS`, `PAPERLESS_WORKER_TIMEOUT`, `PAPERLESS_WORKER_RETRY`, `PAPERLESS_REDIS` | High — documents configuration knobs for task scheduling |
| `docs/setup.rst` | Installation and initial setup | Medium — covers initial deployment but not ongoing maintenance |
| `docs/api.rst` | REST API documentation | Low — no direct coverage of background tasks |
| `docs/troubleshooting.rst` | Common issues and solutions | Medium — may reference task failures |
| `docs/changelog.rst` | Version history | Low |

- **No dedicated documentation page exists** for background maintenance architecture, periodic task scheduling, or django-q configuration as a unified topic.
- **No documentation generator for API docstrings** (such as autodoc or Sphinx apidoc) is configured for the Python source code.
- **No diagram tools** are configured in the Sphinx build — Mermaid diagrams exist only in the tech spec context, not in the project's own docs.

### 0.2.2 Repository Code Analysis for Documentation

**Search patterns used to discover background task infrastructure:**

- Searched for `Schedule`, `async_task`, and `django_q` references across all `.py` files in `src/` (excluding migrations, tests, `__pycache__`)
- Examined all Django migration files for `Schedule.objects.create()` calls
- Traced `supervisord.conf` and `docker-prepare.sh` for startup and process management
- Inspected `settings.py` for `Q_CLUSTER` configuration
- Read all files in `src/documents/management/commands/` for CLI wrappers around task functions

**Key Directories Examined:**

| Directory | Purpose | Files Read |
|-----------|---------|------------|
| `src/documents/` | Core document management app | `tasks.py`, `sanity_checker.py`, `consumer.py`, `index.py`, `classifier.py`, `models.py`, `views.py`, `apps.py`, `checks.py`, `signals/handlers.py` |
| `src/documents/migrations/` | Database schema and data migrations | `1001_auto_20201109_1636.py`, `1004_sanity_check_schedule.py` |
| `src/documents/management/commands/` | Django management commands | `document_consumer.py`, `document_index.py`, `document_sanity_checker.py` |
| `src/paperless/` | Django project settings and config | `settings.py`, `checks.py` |
| `src/paperless_mail/` | Email ingestion app | `tasks.py`, `mail.py` |
| `src/paperless_mail/migrations/` | Mail app migrations | `0002_auto_20201117_1334.py` |
| `docker/` | Container orchestration | `supervisord.conf`, `docker-prepare.sh` |
| `docs/` | Sphinx documentation | `administration.rst`, `configuration.rst` |

**Related Documentation Found:**

- `docs/administration.rst` (lines ~180-420) documents management commands that wrap the periodic task functions — `document_index` (optimize, reindex), `document_sanity_checker`, `mail_fetcher`, and `document_retagger`. However, this documentation describes only the manual CLI invocation, not the automatic periodic execution.
- `docs/configuration.rst` documents `PAPERLESS_TASK_WORKERS`, `PAPERLESS_WORKER_TIMEOUT`, and `PAPERLESS_WORKER_RETRY` environment variables but does not explain how they feed into the `Q_CLUSTER` configuration or what django-q does with them.

### 0.2.3 Web Search Research Conducted

- **django-q 1.3 Schedule model, OrmQ, and task history:** Confirmed that django-q stores schedules as Django ORM models, provides `Success` and `Failure` proxy models for task result tracking, uses `OrmQ` for the ORM broker queue, and supports admin integration for schedule management. The `catch_up` configuration option controls whether missed schedules are executed retroactively. The `save_limit` option controls how many successful task results are retained. Failures are always saved. The scheduler can be disabled globally via the `scheduler: False` configuration option.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules Requiring Documentation:**

- **Module: `src/documents/tasks.py`**
  - Public Functions: `index_optimize()`, `index_reindex()`, `train_classifier()`, `consume_file()`, `sanity_check()`, `bulk_update_documents()`
  - Current Documentation: Partial — `docs/administration.rst` covers CLI wrappers but not the automatic scheduling or internal behavior
  - Documentation Needed: Complete function-by-function analysis of what each periodic task does, its error handling, logging output, and scheduling frequency

- **Module: `src/documents/sanity_checker.py`**
  - Public Functions: `check_sanity()` function, `SanityCheckMessages` class, `SanityCheckFailedException` exception
  - Current Documentation: Minimal — `docs/administration.rst` mentions the `document_sanity_checker` management command with a brief description
  - Documentation Needed: Detailed validation inventory (every check performed), log output format with logger name `paperless.sanity_checker`, error/warning/info classification, and the exception raised on failure

- **Module: `src/documents/index.py`**
  - Public Functions: `open_index()`, `update_document()`, `remove_document()`, Whoosh schema definition
  - Current Documentation: `docs/administration.rst` mentions `document_index optimize` and `document_index reindex` commands
  - Documentation Needed: Explanation of what the daily `index_optimize` task does (Whoosh `AsyncWriter` with `optimize=True`) and how it relates to the search index

- **Module: `src/documents/classifier.py`**
  - Public Classes: `DocumentClassifier` with `train()` method, `FORMAT_VERSION`, scikit-learn MLPClassifier usage
  - Current Documentation: `docs/administration.rst` mentions the `document_retagger` command
  - Documentation Needed: How the hourly `train_classifier` task works — data hash comparison to skip unchanged training sets, model serialization to `MODEL_FILE`, exception handling (catches all, logs warning, does not re-raise)

- **Module: `src/paperless_mail/tasks.py`**
  - Public Functions: `process_mail_accounts()`
  - Current Documentation: `docs/administration.rst` mentions `mail_fetcher` management command
  - Documentation Needed: Per-account error isolation, `MailError` handling, 10-minute polling interval

- **Module: `src/paperless/settings.py` (Q_CLUSTER section)**
  - Configuration: `Q_CLUSTER` dictionary with `name`, `workers`, `retry`, `timeout`, `recycle`, `catch_up`, `redis`
  - Current Documentation: `docs/configuration.rst` documents individual env vars but not the assembled `Q_CLUSTER` structure
  - Documentation Needed: Complete annotated Q_CLUSTER configuration with environment variable mappings and behavioral explanations

- **Module: `docker/supervisord.conf`**
  - Processes: `gunicorn`, `consumer`, `scheduler` (qcluster)
  - Current Documentation: None in `docs/` — only exists as the raw config file
  - Documentation Needed: Process supervision architecture, how the three processes interact

- **Module: `docker/docker-prepare.sh`**
  - Startup Steps: PostgreSQL wait, Redis wait, migrations, search index rebuild, superuser creation
  - Current Documentation: None as a unified topic
  - Documentation Needed: Startup-vs-scheduled distinction, conditional execution logic (e.g., search index only rebuilds if version marker is outdated)

- **Module: `src/documents/apps.py`**
  - Startup Behavior: `DocumentsConfig.ready()` registers 6 signal handlers on `document_consumption_finished` and 2 handlers on model signals
  - Current Documentation: None
  - Documentation Needed: Signal-driven behavior that runs on every document consumption (not on schedule)

- **Module: `src/paperless/checks.py` and `src/documents/checks.py`**
  - Startup Checks: `paths_check`, `binaries_check`, `debug_mode_check`, `changed_password_check`, `parser_check`
  - Current Documentation: None
  - Documentation Needed: Django system checks that run at process startup, what they validate, and their failure behavior

**Configuration Options Requiring Documentation:**

| Environment Variable | Setting in `Q_CLUSTER` | Default | Documented in `configuration.rst` |
|---------------------|----------------------|---------|----------------------------------|
| `PAPERLESS_TASK_WORKERS` | `workers` | `max(1, CPU_count / 4)` | Yes |
| `PAPERLESS_WORKER_TIMEOUT` | `timeout` | `1800` (30 min) | Yes |
| `PAPERLESS_WORKER_RETRY` | `retry` | `timeout + 10` (1810s) | Yes |
| `PAPERLESS_REDIS` | `redis` | `redis://localhost:6379` | Yes |
| N/A (hardcoded) | `catch_up` | `False` | No |
| N/A (hardcoded) | `recycle` | `1` | No |
| N/A (hardcoded) | `name` | `"paperless"` | No |

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No unified background maintenance document:** Periodic tasks are mentioned in `administration.rst` only as manual CLI commands, never as automatically scheduled background jobs. There is no single page explaining what happens automatically.
- **No django-q architecture explanation:** The project documentation never mentions django-q by name, never explains the `Q_CLUSTER` configuration as a whole, and never documents the `qcluster` management command or its role.
- **No startup-vs-scheduled task distinction:** The `docker-prepare.sh` startup tasks and the django-q periodic tasks are not documented as distinct execution contexts.
- **No task failure documentation:** What happens when `sanity_check` raises `SanityCheckFailedException` or when `train_classifier` catches and logs an exception is not documented.
- **No task history/monitoring guidance:** The django-q admin pages (`Success`, `Failure`, `Scheduled Tasks`) are not referenced in the project's documentation.
- **No database table documentation for task state:** The `django_q_task`, `django_q_schedule`, `django_q_ormq` tables and the `documents_log` table are undocumented.
- **Hardcoded Q_CLUSTER settings undocumented:** `catch_up=False`, `recycle=1`, and the cluster name `"paperless"` are not exposed as environment variables and are not documented in `configuration.rst`.
- **No enable/disable control documentation:** There is no documented way to disable specific periodic tasks (though they can be deleted or modified via Django admin as django-q Schedule objects).
- **Django system checks undocumented:** The `paths_check`, `binaries_check`, `debug_mode_check`, `changed_password_check`, and `parser_check` system checks that run at startup are not documented as a maintenance or validation feature.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document placed at `blitzy/documentation/paperless-ngx_542221a38dff.md` per the SWE-AtlasQnA-Repo implementation rule. The document structure addresses every question posed by the user, organized by topic:

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Overview & Key Architectural Finding (django-q, not Celery)
    ├── Periodic Task Inventory (all 4 scheduled tasks with details)
    ├── Schedule Configuration Source (migrations + Q_CLUSTER)
    ├── Sanity Checker Deep Dive (validations, logs, frequency)
    ├── Index Optimization (daily Whoosh optimize)
    ├── Database Cleanup (not implemented — documented absence)
    ├── Failed Document Retries / Stuck Jobs (retry semantics)
    ├── Task Scheduler Configuration (Q_CLUSTER annotated)
    ├── Task Registration Code Path (migration tracing)
    ├── Task Failure Handling (per-task exception analysis)
    ├── Startup vs. Scheduled Tasks (docker-prepare.sh vs qcluster)
    ├── Task Execution History (database tables)
    ├── Enable/Disable Controls (configuration knobs)
    └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract periodic task registrations from Django migration files `src/documents/migrations/1001_auto_20201109_1636.py`, `src/documents/migrations/1004_sanity_check_schedule.py`, and `src/paperless_mail/migrations/0002_auto_20201117_1334.py` by reading `Schedule.objects.create()` calls
- Extract task function behavior from `src/documents/tasks.py` and `src/paperless_mail/tasks.py` by reading function implementations, exception handling, and logging calls
- Extract sanity checker validations from `src/documents/sanity_checker.py` by enumerating every conditional check in `check_sanity()`
- Extract Q_CLUSTER configuration from `src/paperless/settings.py` by reading the dictionary literal and mapping each key to its environment variable source
- Extract startup task sequence from `docker/docker-prepare.sh` by tracing the shell functions in execution order
- Extract process supervision from `docker/supervisord.conf` by reading the `[program:*]` sections
- Extract signal-driven behavior from `src/documents/apps.py` and `src/documents/signals/handlers.py`
- Extract Django system checks from `src/paperless/checks.py` and `src/documents/checks.py`

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for task scheduling flow and startup sequence
- Code citations as inline references: `Source: src/path/to/file.py`
- Tables for periodic task inventory, configuration parameters, and database tables
- Consistent terminology: "django-q" (not "Celery"), "periodic task" (not "cron job"), "Schedule model" (not "beat schedule")

### 0.4.3 Diagram and Visual Strategy

**Mermaid Diagrams to Create:**

- **Task Scheduling Architecture Diagram:** Flowchart showing the three supervised processes (gunicorn, consumer, qcluster), how qcluster reads Schedule records from the database, and dispatches tasks to workers via Redis
- **Startup vs. Scheduled Execution Diagram:** Sequence diagram distinguishing docker-prepare.sh one-time startup steps from periodic qcluster-managed tasks
- **Sanity Checker Validation Flow:** Flowchart showing each validation step (thumbnail → original → archive → content → orphan detection) with error/warning/info classification
- **Document Consumption Pipeline Diagram:** Sequence diagram showing file detection → `async_task` dispatch → consumer worker → signal handlers → index update, illustrating where retry logic applies
- **Task Failure Handling Decision Tree:** Flowchart showing per-task exception handling behavior — which tasks catch and suppress exceptions vs. which propagate them to django-q's failure tracking

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/tasks.py`, `src/documents/sanity_checker.py`, `src/documents/index.py`, `src/documents/classifier.py`, `src/paperless_mail/tasks.py`, `src/paperless/settings.py`, `src/documents/migrations/1001_auto_20201109_1636.py`, `src/documents/migrations/1004_sanity_check_schedule.py`, `src/paperless_mail/migrations/0002_auto_20201117_1334.py`, `docker/supervisord.conf`, `docker/docker-prepare.sh`, `src/documents/apps.py`, `src/documents/signals/handlers.py`, `src/paperless/checks.py`, `src/documents/checks.py`, `src/documents/consumer.py`, `src/documents/management/commands/document_consumer.py`, `src/documents/management/commands/document_index.py`, `src/documents/management/commands/document_sanity_checker.py`, `src/documents/models.py`, `src/documents/views.py`, `src/paperless_mail/mail.py`, `docs/administration.rst`, `docs/configuration.rst`, `Pipfile` | Comprehensive Q&A document covering all background maintenance topics: periodic task inventory, schedule configuration, sanity checker analysis, index optimization, database cleanup (absence documented), failed retries, task registration tracing, failure handling, startup vs. scheduled, task history tables, enable/disable controls |

This is the sole deliverable file. No existing documentation files are modified per the implementation rule "Do not modify any existing files in the source repository."

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Q&A / Architecture Analysis
Source Code: 22 source files across 6 directories (see table above)
Sections:
    - Overview & Key Finding (django-q architecture, not Celery)
    - Periodic Task Inventory (4 tasks with full details)
      - train_classifier: src/documents/tasks.py, src/documents/classifier.py
      - index_optimize: src/documents/tasks.py, src/documents/index.py
      - sanity_check: src/documents/tasks.py, src/documents/sanity_checker.py
      - process_mail_accounts: src/paperless_mail/tasks.py, src/paperless_mail/mail.py
    - Schedule Configuration (Q_CLUSTER from src/paperless/settings.py)
    - Schedule Registration (3 migration files)
    - Sanity Checker Deep Dive (src/documents/sanity_checker.py)
    - Index Optimization (src/documents/index.py, daily optimize)
    - Database Cleanup (absence documented with evidence)
    - Failed Document Retries (consumer retry logic, django-q retry param)
    - Task Failure Handling (per-task exception analysis)
    - Startup Tasks (docker/docker-prepare.sh, src/paperless/checks.py, src/documents/checks.py)
    - Task History Database Tables (django-q models, documents.Log)
    - Enable/Disable Controls (env vars, Q_CLUSTER settings, admin interface)
Diagrams:
    - Task scheduling architecture (Mermaid flowchart)
    - Startup vs. scheduled execution (Mermaid sequence)
    - Sanity checker validation flow (Mermaid flowchart)
    - Document consumption pipeline (Mermaid sequence)
    - Task failure handling decision tree (Mermaid flowchart)
Key Citations:
    src/documents/tasks.py, src/documents/sanity_checker.py,
    src/documents/index.py, src/documents/classifier.py,
    src/paperless_mail/tasks.py, src/paperless/settings.py,
    src/documents/migrations/1001_auto_20201109_1636.py,
    src/documents/migrations/1004_sanity_check_schedule.py,
    src/paperless_mail/migrations/0002_auto_20201117_1334.py,
    docker/supervisord.conf, docker/docker-prepare.sh,
    src/documents/apps.py, src/documents/signals/handlers.py,
    src/paperless/checks.py, src/documents/checks.py,
    src/documents/models.py, docs/administration.rst, docs/configuration.rst
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need updating. The output document is a standalone markdown file in `blitzy/documentation/` and is not integrated into the project's Sphinx documentation build. No `docs/conf.py`, `mkdocs.yml`, or sidebar configuration changes are required.

### 0.5.4 Cross-Documentation Dependencies

- **No shared includes:** The output document is self-contained
- **No navigation link updates:** The document lives outside the Sphinx docs tree
- **Internal cross-references:** The document will reference existing `docs/administration.rst` and `docs/configuration.rst` by path for context but does not modify them
- **No index or glossary updates required**

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to this documentation exercise — they are the subject matter being documented, not tools used to generate the documentation. Since the deliverable is a standalone markdown file, no documentation generation tools are required.

**Subject-Matter Packages (from `Pipfile`):**

| Registry | Package Name | Version (from Pipfile) | Purpose in Background Maintenance |
|----------|-------------|----------------------|----------------------------------|
| PyPI | django-q | ~=1.3 (resolved: 1.3.9) | Task queue, scheduler, and worker framework — the core of all periodic task execution |
| PyPI | django | ~=4.0 | Web framework providing ORM, migrations, signals, management commands, and system checks |
| PyPI | redis | ~=4.1 | Python client for Redis — the message broker connecting django-q scheduler to workers |
| PyPI | whoosh | ~=2.7 | Full-text search library — subject of daily `index_optimize` periodic task |
| PyPI | scikit-learn | ~=1.0 | Machine learning — subject of hourly `train_classifier` periodic task (MLPClassifier, CountVectorizer) |
| PyPI | watchdog | ~=2.1 | Filesystem event monitoring — used by `document_consumer` management command (not a periodic task but a long-running process) |
| PyPI | inotifyrecursive | ~=0.3 | Linux inotify wrapper — alternative to watchdog polling for filesystem monitoring |
| PyPI | channels | ~=3.0 | Django Channels for WebSocket — used to broadcast document consumption progress |
| PyPI | channels-redis | ~=3.3 | Redis backend for Django Channels |
| PyPI | ocrmypdf | ~=13.2 | OCR processing — invoked during document consumption (not a periodic task itself) |

**Infrastructure Dependencies (from `docker/supervisord.conf` and `docker/docker-prepare.sh`):**

| Component | Version | Purpose in Background Maintenance |
|-----------|---------|----------------------------------|
| supervisord | (bundled in Docker image) | Process supervisor — starts and monitors `gunicorn`, `consumer`, and `scheduler` (qcluster) processes |
| Redis server | (external service) | Message broker for django-q task queue; required for scheduler operation |
| PostgreSQL | (external service, optional) | Database backend; SQLite is default; stores django-q `Schedule`, `Task`, `OrmQ` tables |

### 0.6.2 Documentation Reference Updates

Not applicable. Per the implementation rule, no existing documentation files are modified. The new document is placed in `blitzy/documentation/` outside the existing `docs/` Sphinx tree. No link transformations are required.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current Coverage Analysis (of background maintenance topics in existing docs):**

| Topic | Currently Documented | Where | Gap |
|-------|---------------------|-------|-----|
| Periodic task inventory | 0/4 tasks documented as periodic | — | Complete gap — no existing doc lists automatically scheduled tasks |
| Schedule configuration (Q_CLUSTER) | 3/7 Q_CLUSTER params exposed as env vars | `docs/configuration.rst` | `catch_up`, `recycle`, `name`, cluster architecture undocumented |
| Sanity checker validations | 1-line description only | `docs/administration.rst` | Detailed validation list, log output, frequency undocumented |
| Index optimization | CLI command documented | `docs/administration.rst` | Automatic daily scheduling undocumented |
| Database cleanup | N/A (feature absent) | — | Absence not documented |
| Failed document retries | Not documented | — | Complete gap |
| Task registration code path | Not documented | — | Complete gap |
| Task failure handling | Not documented | — | Complete gap |
| Startup vs. scheduled tasks | Partially (setup steps exist) | `docs/setup.rst` | No distinction drawn between one-time startup and recurring |
| Task execution history tables | Not documented | — | Complete gap |
| Enable/disable controls | Partially (TASK_WORKERS, TIMEOUT, RETRY) | `docs/configuration.rst` | No unified controls reference |
| Django system checks | Not documented | — | Complete gap |

**Summary:** Of 12 documentation topics requested, 0 are fully covered, 3 are partially covered, and 9 have complete gaps in the existing documentation.

**Target Coverage:** 100% of all user questions answered with evidence-based, code-cited responses in the output document.

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**

- Every periodic task documented with: function path, human-readable name, schedule interval, registration source (migration file), internal behavior, exception handling, and log output
- Every Q_CLUSTER parameter documented with: key name, value, source (env var or hardcoded), and behavioral explanation
- Every sanity checker validation documented with: what is checked, failure classification (error/warning/info), and the log message pattern
- Every startup task documented with: execution order, conditional logic, and what triggers it
- Every database table for task tracking documented with: table name, key columns, and purpose
- Every enable/disable control documented with: environment variable, default value, and effect

**Accuracy Validation:**

- All code citations reference actual file paths and are verified against the repository
- All function signatures match the current codebase (commit `542221a38`)
- All configuration defaults match the values in `src/paperless/settings.py`
- All migration-registered schedules match the actual `Schedule.objects.create()` parameters
- The document explicitly corrects the user's assumption of "Celery beat" to "django-q"

**Clarity Standards:**

- Technical accuracy with accessible language — explain django-q concepts for readers unfamiliar with the framework
- Progressive disclosure: start with the high-level inventory, then deep-dive into each topic
- Consistent terminology: always "django-q" (not "celery"), always "periodic task" (not "cron"), always "Schedule model" (not "beat entry")
- Rationale provided for every answer per the implementation rule

**Maintainability:**

- Source citations (`Source: src/path/to/file.py`) for every technical claim
- Structured tables and diagrams for quick reference
- Clear section headings matching user questions for easy navigation

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per topic:** At least one code snippet or configuration excerpt per major topic (Q_CLUSTER block, migration schedule creation, sanity checker log output pattern, task function signature)
- **Diagram types required:** 5 Mermaid diagrams (task architecture, startup sequence, sanity checker flow, consumption pipeline, failure handling)
- **Code example testing:** Not applicable — examples are excerpts from existing source code, not runnable samples
- **Visual content freshness:** All diagrams derived from current codebase state at commit `542221a38`

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New Documentation Files:**

- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable, a comprehensive Q&A document on background maintenance architecture

**Source Code Files to Analyze (read-only):**

- `src/documents/tasks.py` — all periodic task function implementations
- `src/documents/sanity_checker.py` — sanity checker validation logic and message classes
- `src/documents/index.py` — Whoosh search index operations (optimize, reindex, update, remove)
- `src/documents/classifier.py` — ML classifier training logic (scikit-learn MLPClassifier)
- `src/documents/consumer.py` — document consumption pipeline and progress broadcasting
- `src/documents/models.py` — `Log` model and other data models
- `src/documents/apps.py` — `DocumentsConfig.ready()` signal handler registration
- `src/documents/signals/__init__.py` — signal definitions
- `src/documents/signals/handlers.py` — signal handler implementations
- `src/documents/checks.py` — Django system checks (password, parser)
- `src/documents/views.py` — `PostDocumentView` async_task usage
- `src/documents/management/commands/document_consumer.py` — filesystem watcher with retry logic
- `src/documents/management/commands/document_index.py` — CLI for index operations
- `src/documents/management/commands/document_sanity_checker.py` — CLI wrapper for sanity check
- `src/documents/migrations/1001_auto_20201109_1636.py` — schedule registration (classifier + index)
- `src/documents/migrations/1004_sanity_check_schedule.py` — schedule registration (sanity check)
- `src/paperless/settings.py` — `Q_CLUSTER` configuration and all `PAPERLESS_*` env var parsing
- `src/paperless/checks.py` — Django system checks (paths, binaries, debug mode)
- `src/paperless_mail/tasks.py` — mail processing task function
- `src/paperless_mail/mail.py` — IMAP mail handler implementation
- `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — schedule registration (mail check)
- `docker/supervisord.conf` — process supervision configuration
- `docker/docker-prepare.sh` — container startup script

**Existing Documentation Files to Reference (read-only):**

- `docs/administration.rst` — existing management command documentation
- `docs/configuration.rst` — existing environment variable documentation
- `Pipfile` — dependency manifest

**Documentation Topics In Scope:**

- Complete periodic task inventory with intervals and registration sources
- `Q_CLUSTER` configuration analysis with every parameter explained
- Sanity checker validation inventory with log output format
- Index optimization mechanism and scheduling
- Database cleanup presence/absence analysis
- Failed document retry mechanisms (consumer-level and django-q-level)
- Task registration code path tracing through Django migrations
- Per-task failure handling and exception behavior
- Startup tasks vs. strictly scheduled tasks distinction
- Database tables for task execution history and job state
- Configuration knobs for enabling/disabling maintenance features
- Django system checks that run at startup

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — the implementation rule explicitly prohibits modifying existing files
- **Test file modifications** — no test files are created, modified, or executed
- **Feature additions or code refactoring** — this is purely observational documentation
- **Deployment configuration changes** — no Docker, supervisord, or infrastructure changes
- **Existing documentation updates** — no changes to `docs/*.rst` files per implementation rule
- **Frontend/UI analysis** — the Angular frontend (`src-ui/`) has no relevance to background tasks
- **REST API endpoint documentation** — API behavior is only relevant where `PostDocumentView` dispatches `async_task`
- **Consumer parsers** (`paperless_tesseract/`, `paperless_text/`, `paperless_tika/`) — these are invoked during consumption but are not periodic tasks or maintenance features
- **Data migration schema changes** — migrations are read for schedule registration but schema changes are not documented
- **Performance benchmarking** — no load testing or timing analysis of periodic tasks
- **Unrelated documentation** not specified by the user (e.g., user guides, API references, installation guides)

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone markdown file, not part of a documentation build system.
- **Documentation preview command:** Any markdown viewer or `cat blitzy/documentation/paperless-ngx_542221a38dff.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the markdown using fenced mermaid code blocks. Any Mermaid-compatible renderer (GitHub, VS Code Mermaid extension, mermaid-cli) can render them.
- **Documentation deployment command:** Not applicable — file is committed to the repository.
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim must reference the source file path where the evidence was found
- **Style guide:** SWE-AtlasQnA-Repo rule — provide thinking/rationale behind answers, base answers on code as truth, no assumptions
- **Documentation validation:** Manual review — verify all cited file paths exist and all claimed behaviors match source code at commit `542221a38`

### 0.9.2 Output File Placement

Per the implementation rule, the output document is placed at:

- **Directory:** `blitzy/documentation/`
- **Filename:** `paperless-ngx_542221a38dff.md`

The `blitzy/documentation/` directory must be created if it does not exist. The file is placed in the destination repo as specified by the rule.

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are explicitly stated by the user or derived from implementation rules and must be strictly followed:

- **"Don't modify source code; observing system behavior is fine, clean up any test artifacts when done."** — No existing files in the source repository may be modified. The output is a new document only. Any temporary artifacts created during investigation must be cleaned up.
- **"Do not make assumptions, base your answers on the code as the truth."** — Every claim in the output document must be directly traceable to a specific source file. Speculative or assumed behavior is not permitted.
- **"Provide thinking / rationale behind the answers."** — The document must include reasoning and evidence chains, not just bare facts. Explain *why* the code behaves a certain way, not just *what* it does.
- **"Create a new markdown document named `<source_branch_name>.md`"** — The output file must be named `paperless-ngx_542221a38dff.md` exactly matching the source branch name.
- **"Place the generated document in the `blitzy/documentation` directory in the destination repo."** — The document goes in `blitzy/documentation/`, not in the project's `docs/` directory.
- **"Do not modify any existing files in the source repository."** — Reinforces the read-only constraint. No changes to any file that existed before this session.

### 0.10.2 Derived Documentation Standards

Based on the user's questions and the nature of the investigation:

- **Correct the Celery misconception explicitly:** The user asked about "Celery beat schedule configuration" — the document must prominently state that paperless-ngx uses django-q, not Celery, and explain the architectural difference.
- **Document absences as findings:** When the user asks "Is there automatic database cleanup?" and the answer is no, the document must explicitly state this with evidence (searched files, found no matching tasks) rather than silently omitting the topic.
- **Provide exhaustive inventories:** When listing periodic tasks, every single registered task must be listed — partial lists are unacceptable.
- **Include source file paths with every claim:** Per the evidence-based approach, every statement about code behavior must cite the file path where the evidence is found.
- **Use Mermaid diagrams for complex flows:** The scheduling architecture, startup sequence, and sanity checker flow are complex enough to warrant visual representation.

## 0.11 References

### 0.11.1 Source Files Searched and Analyzed

The following files and folders were searched across the codebase to derive all conclusions in this Agent Action Plan:

**Core Task Infrastructure:**

| File Path | Purpose | Key Findings |
|-----------|---------|-------------|
| `src/documents/tasks.py` | Periodic task function implementations | Contains `index_optimize()`, `train_classifier()`, `sanity_check()`, `consume_file()`, `index_reindex()`, `bulk_update_documents()` |
| `src/documents/sanity_checker.py` | Sanity checker validation logic | `check_sanity()` validates thumbnails, originals, archives (existence, readability, checksums), content presence, orphaned files; uses `SanityCheckMessages` class with error/warning/info levels; logs to `paperless.sanity_checker` |
| `src/documents/index.py` | Whoosh search index operations | Schema definition (id, title, content, asn, correspondent, tag, type, created, modified, added), `open_index()`, `update_document()`, `remove_document()`, `DelayedQuery` classes |
| `src/documents/classifier.py` | ML classifier training | `DocumentClassifier` with `FORMAT_VERSION=7`, scikit-learn `MLPClassifier`/`CountVectorizer`, `train()` hashes data to skip if unchanged, persists model to pickle |
| `src/documents/consumer.py` | Document consumption pipeline | `Consumer` class with Channels WebSocket progress broadcasting, error handling |
| `src/paperless_mail/tasks.py` | Mail processing task | `process_mail_accounts()` iterates `MailAccount` objects, delegates to `MailAccountHandler`, catches `MailError` per-account |
| `src/paperless_mail/mail.py` | IMAP mail handler | `MailAccountHandler` connects IMAP, evaluates rules, fetches/filters messages, extracts attachments, queues via `async_task` |

**Schedule Registration (Django Migrations):**

| File Path | Schedules Registered |
|-----------|---------------------|
| `src/documents/migrations/1001_auto_20201109_1636.py` | `train_classifier` (HOURLY), `index_optimize` (DAILY) |
| `src/documents/migrations/1004_sanity_check_schedule.py` | `sanity_check` (WEEKLY) |
| `src/paperless_mail/migrations/0002_auto_20201117_1334.py` | `process_mail_accounts` (every 10 MINUTES) |

**Configuration and Settings:**

| File Path | Key Findings |
|-----------|-------------|
| `src/paperless/settings.py` | `Q_CLUSTER` config: `name="paperless"`, `catch_up=False`, `recycle=1`, `retry=PAPERLESS_WORKER_RETRY`, `timeout=PAPERLESS_WORKER_TIMEOUT`, `workers=TASK_WORKERS`, `redis` from env |
| `Pipfile` | `django-q = "~=1.3"`, confirmed django-q is the task framework (not Celery) |
| `docs/configuration.rst` | Documents `PAPERLESS_TASK_WORKERS`, `PAPERLESS_WORKER_TIMEOUT`, `PAPERLESS_WORKER_RETRY`, `PAPERLESS_REDIS` env vars |
| `docs/administration.rst` | Documents management commands: `document_index`, `document_sanity_checker`, `mail_fetcher`, `document_retagger` |

**Docker and Process Management:**

| File Path | Key Findings |
|-----------|-------------|
| `docker/supervisord.conf` | Three supervised processes: `gunicorn` (web), `consumer` (document_consumer), `scheduler` (qcluster) |
| `docker/docker-prepare.sh` | Startup tasks: wait_for_postgres, wait_for_redis, migrations (with flock), search_index rebuild (conditional), superuser creation (conditional) |

**Application Startup and Signals:**

| File Path | Key Findings |
|-----------|-------------|
| `src/documents/apps.py` | `DocumentsConfig.ready()` registers 6 signal handlers on `document_consumption_finished` and model signals |
| `src/documents/signals/__init__.py` | Defines `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` signals |
| `src/documents/signals/handlers.py` | Handlers for matching/tagging, file move on save, deletion cleanup, index update |
| `src/paperless/checks.py` | Django system checks: `paths_check`, `binaries_check`, `debug_mode_check` |
| `src/documents/checks.py` | Django system checks: `changed_password_check`, `parser_check` |

**Management Commands:**

| File Path | Key Findings |
|-----------|-------------|
| `src/documents/management/commands/document_consumer.py` | Filesystem watcher using watchdog/inotify, file retry logic (50 × 10ms readability, configurable stability retries) |
| `src/documents/management/commands/document_index.py` | CLI with `reindex` and `optimize` subcommands |
| `src/documents/management/commands/document_sanity_checker.py` | Thin wrapper calling `check_sanity(progress=...)` |

**Data Models:**

| File Path | Key Findings |
|-----------|-------------|
| `src/documents/models.py` | `Log` model (group UUID, message, level, created) for document consumption logging |
| `src/documents/views.py` | `PostDocumentView` dispatches `async_task("documents.tasks.consume_file", ...)` for uploaded documents |

**Folders Explored:**

| Folder Path | Contents |
|-------------|----------|
| (repository root) | `src/`, `src-ui/`, `docker/`, `docs/`, `Pipfile`, `Dockerfile`, etc. |
| `src/` | `documents/`, `paperless/`, `paperless_mail/`, `paperless_tesseract/`, `paperless_text/`, `paperless_tika/`, `manage.py` |
| `src/documents/` | Core app: tasks, models, views, signals, checks, consumer, index, classifier, sanity_checker |
| `src/documents/migrations/` | Data migrations registering django-q schedules |
| `src/documents/management/commands/` | CLI wrappers for background task functions |
| `src/documents/signals/` | Signal definitions and handler implementations |
| `src/paperless/` | Project settings, URL configuration, system checks |
| `src/paperless_mail/` | Email ingestion app: tasks, mail handler, migrations |
| `docker/` | supervisord.conf, docker-prepare.sh, Dockerfile-related |
| `docs/` | Sphinx documentation: administration.rst, configuration.rst |

### 0.11.2 External Research

| Search Query | Source | Key Finding |
|-------------|--------|-------------|
| "django-q 1.3 Schedule model OrmQ task history" | django-q.readthedocs.io | Confirmed Schedule is a regular Django model; admin exposes Success, Failure, and Scheduled Tasks views; `catch_up=False` skips missed schedules; `save_limit` controls successful task retention; failures are always saved; scheduler can be disabled via `scheduler: False` config |

### 0.11.3 Attachments and External Resources

- **No attachments** were provided for this project.
- **No Figma URLs** were provided.
- **No external API documentation** was referenced.

### 0.11.4 Tech Spec Sections Referenced

| Section | Key Information Extracted |
|---------|-------------------------|
| 1.1 Executive Summary | Confirmed paperless-ngx architecture overview, django-q role |
| 3.2 Frameworks & Libraries | Confirmed django-q 1.3.9 version, scikit-learn pinning, overall dependency landscape |

