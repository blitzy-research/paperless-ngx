# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively explains how paperless-ngx handles background processing at runtime during document ingestion. The user seeks to go beyond configuration reference material and instead build a deep, code-grounded narrative that traces the lifecycle of an asynchronous job from the moment it is created to the point where its outcome can be inspected after completion.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical deep-dive / Architecture explanation document
- **Target Output:** A standalone Markdown file named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation/` directory of the destination repository

The user's requirements decompose into the following distinct documentation objectives:

| # | Requirement (User Intent) | Clarified Technical Scope |
|---|---------------------------|---------------------------|
| R1 | How async work behaves in a live setup | Document the runtime service topology: Supervisor managing Gunicorn, the directory watcher consumer, and the Django-Q `qcluster` worker pool, all communicating through a shared Redis broker |
| R2 | What services are involved behind the scenes | Enumerate every process, infrastructure dependency, and communication pathway: Supervisord, Gunicorn/Uvicorn ASGI, `document_consumer`, `qcluster`, Redis, the database, and the channel layer |
| R3 | How background jobs appear once created | Trace the exact code path of `django_q.tasks.async_task()` calls and explain how tasks materialize in the Redis broker queue and ultimately in the Django-Q ORM tables |
| R4 | Difference between waiting and actively processing work | Explain Django-Q cluster's dequeueing model: tasks queued in Redis vs. tasks actively picked up by a sentinel/worker, referencing the `Q_CLUSTER` configuration including `timeout`, `retry`, and `workers` settings |
| R5 | Where the task state ends up being stored | Document that Django-Q persists task outcomes to the relational database (`django_q_ormq` for queued tasks, `django_q_task` for completed results, `django_q_schedule` for recurring schedules) |
| R6 | How to tell what happened to a given job after the fact | Show how to query the `Task` model from `django_q.models` and how WebSocket `status_updates` broadcasts provide real-time progress during execution |
| R7 | Where background jobs are triggered in the code | Map every call site of `async_task()` across the codebase: `PostDocumentView`, `document_consumer` management command, `MailAccountHandler`, and all five `bulk_edit.py` functions |
| R8 | What code is responsible for sending jobs into the queue | Identify the entry-point modules and functions that serve as producers for the task queue |

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — No Source Code Modifications:** The user explicitly stated: "Don't modify any repository source files." The existing codebase must remain untouched. If temporary scripts are created to observe behavior, they must be cleaned up afterward.
- **New File Only:** Per the project implementation rule (`SWE-AtlasQnA-Repo`), the deliverable is a single new Markdown document placed at `blitzy/documentation/paperless-ngx_542221a38dff.md`.
- **Code-as-Truth Principle:** Answers must be grounded in actual code analysis, not assumptions. Every claim about runtime behavior must reference specific source files, line numbers, and configuration values observed in the repository.
- **Provide Rationale:** The documentation must include the thinking and rationale behind each answer, not just bare facts.
- **Style:** Technical deep-dive; the tone should be explanatory and investigative — the user explicitly said "I want to see what happens at runtime" and "help me make sense of how async work behaves."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the runtime service topology** (R1, R2), we will create a new section in the output document that maps the three Supervisord-managed processes (`gunicorn`, `consumer`, `scheduler`) from `docker/supervisord.conf` and explains how Redis serves a dual role as both the Django-Q task broker (`Q_CLUSTER` in `src/paperless/settings.py` lines 449–457) and the Channels WebSocket layer (`CHANNEL_LAYERS` in `src/paperless/settings.py` lines 178–187).
- To **explain task creation and queuing** (R3, R7, R8), we will trace every `async_task()` call site found via codebase grep in `src/documents/views.py:523`, `src/documents/management/commands/document_consumer.py:86`, `src/paperless_mail/mail.py:336`, and `src/documents/bulk_edit.py:18,31,47,63,87`, documenting the function signature, arguments, and context of each invocation.
- To **explain task lifecycle states** (R4), we will describe Django-Q's internal sentinel/worker model where the `qcluster` process spawns worker processes that poll Redis for queued tasks, referencing the `Q_CLUSTER` configuration including `workers`, `timeout`, `retry`, and `recycle` settings.
- To **document task state persistence** (R5, R6), we will explain the Django-Q ORM models: `OrmQ` (queued tasks in Redis plus overflow), `Task` (completed task results), and `Schedule` (recurring tasks like mail fetching registered in migration `src/paperless_mail/migrations/0002_auto_20201117_1334.py`).

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **WebSocket progress reporting:** The `Consumer._send_progress()` method in `src/documents/consumer.py` (lines 56–76) broadcasts real-time status payloads through the Redis channel layer to the `status_updates` group. This is a key mechanism for understanding "what happened to a given job" in real-time and should be documented alongside post-hoc database inspection.
- **Signal-driven post-processing:** The six signal handlers registered in `src/documents/apps.py` (lines 22–27) — inbox tag assignment, correspondent matching, document type matching, tag matching, admin log entry, and search index update — all fire as part of the `document_consumption_finished` signal within the atomic transaction. These represent "hidden" background work that occurs inside the task but is not itself a separate queued job.
- **Scheduled recurring tasks:** The mail fetching schedule (every 10 minutes, registered via Django migration) is a background processing behavior that the user may not be aware of but is relevant to understanding "how the system tells the difference between work that is waiting and work that is actively being processed."
- **Barcode-based document splitting:** The `consume_file()` function in `src/documents/tasks.py` (lines 184–253) includes a barcode detection and page splitting code path that, when triggered, saves split documents back to the consumption directory for re-ingestion rather than consuming them inline. This creates a recursive task submission pattern that should be documented.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with moderate coverage of configuration and usage concepts, but no dedicated deep-dive documentation on background processing internals.

- **Documentation Framework:** Sphinx (pinned at `~=4.5.0` in `Pipfile` dev-packages)
- **Theme:** `sphinx_rtd_theme` (Read the Docs theme)
- **Configuration File:** `docs/conf.py` — sets project metadata, enables `autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode` extensions
- **Build Driver:** `docs/Makefile` — standard Sphinx `make html` target
- **Hosting:** Read the Docs (configured in `.readthedocs.yml` at repository root)
- **Container Preview:** `docs/Dockerfile` — builds docs and serves via Python HTTP server on port 8000
- **Dependency Manifest:** `docs/requirements.txt` — currently empty (placeholder)

**Existing documentation files examined:**

| File | Content Summary | Relevance to This Task |
|------|-----------------|------------------------|
| `docs/configuration.rst` | All `PAPERLESS_*` environment variables including `PAPERLESS_TASK_WORKERS`, `PAPERLESS_THREADS_PER_WORKER`, `PAPERLESS_WORKER_TIMEOUT`, `PAPERLESS_WORKER_RETRY`, `PAPERLESS_REDIS` | High — documents config knobs but not runtime behavior |
| `docs/usage_overview.rst` | Defines "consumer" and "web server" as the two conceptual parts; explains ingestion from consumption directory and email | Medium — provides user-facing concepts but not internals |
| `docs/api.rst` | REST endpoint documentation including upload | Low — covers the API surface, not the background queue |
| `docs/extending.rst` | Development setup, parser extension, localization | Low — for contributors, not for understanding runtime |
| `docs/setup.rst` | Installation, migration, reverse proxy, deployment | Low — deployment procedures, not architecture |
| `docs/administration.rst` | Backups, updates, indexing, sanity checking | Low — operational tasks |

**Key Finding:** No existing documentation file explains the Django-Q task queue lifecycle, the Supervisord process model, or the Redis-mediated communication patterns at runtime. The `docs/configuration.rst` file references `PAPERLESS_TASK_WORKERS` and `PAPERLESS_REDIS` but only as configuration knobs — it does not explain what these settings control at a system level. This confirms the need for a new, standalone documentation artifact.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to locate all code relevant to background processing:

- **Task dispatcher calls:** `grep -rn "async_task\|from django_q" src/ --include="*.py"` — identified 4 producer modules with 11 total `async_task()` call sites
- **Task function definitions:** Direct read of `src/documents/tasks.py` — identified 8 task-eligible functions
- **Process management:** Read of `docker/supervisord.conf` — identified 3 supervised processes
- **Queue configuration:** Read of `src/paperless/settings.py` lines 414–472 — identified `Q_CLUSTER` dict and worker scaling logic
- **WebSocket status channel:** Read of `src/paperless/consumers.py` and `src/paperless/asgi.py` — identified the `status_updates` group and `StatusConsumer`
- **Signal registration:** Read of `src/documents/apps.py` — identified 6 `document_consumption_finished` handlers
- **Scheduled tasks:** `grep -rn "schedule\|Schedule" src/paperless_mail/migrations/` — identified recurring mail fetch schedule
- **Startup readiness:** Read of `docker/docker-prepare.sh` and `docker/wait-for-redis.py` — identified Redis/Postgres wait loops

**Key directories examined:**
- `src/documents/` — Core ingestion engine, tasks, consumer, views, bulk_edit, signals
- `src/paperless/` — Settings, ASGI routing, WebSocket consumer, URLs
- `src/paperless_mail/` — Mail ingestion including `async_task()` dispatch and scheduled task migration
- `docker/` — Supervisord config, entrypoint, preparation scripts, Redis wait script
- `docs/` — All existing `.rst` documentation files

### 0.2.3 Web Search Research Conducted

No external web searches were required for this task. All answers are derived directly from the repository source code per the user's instruction to base answers on the code as truth. The Django-Q library version (`1.3.9` per `requirements.txt`) and its documented behavior are well understood from the pinned source and the codebase's usage patterns.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage in the output artifact. Each entry identifies the specific code elements that must be explained to answer the user's questions.

**Module: `src/paperless/settings.py` — Task Queue Configuration**
- Public APIs / Settings: `Q_CLUSTER` dict (line 449), `TASK_WORKERS` (line 438), `PAPERLESS_WORKER_TIMEOUT` (line 440), `PAPERLESS_WORKER_RETRY` (line 444), `THREADS_PER_WORKER` (line 469), `CHANNEL_LAYERS` (line 178)
- Current documentation: Configuration knobs listed in `docs/configuration.rst` but behavior not explained
- Documentation needed: Explanation of what each setting controls at runtime, how `Q_CLUSTER` configures the Django-Q sentinel/worker model, and the dual role of Redis

**Module: `src/documents/tasks.py` — Task Function Definitions**
- Public APIs: `consume_file()` (line 184), `index_optimize()` (line 32), `index_reindex()` (line 38), `train_classifier()` (line 48), `sanity_check()` (line 255), `bulk_update_documents()` (line 270), `barcode_reader()` (line 75), `scan_file_for_separating_barcodes()` (line 96), `separate_pages()` (line 113)
- Current documentation: None
- Documentation needed: What each function does, when it is invoked as a background job, and how `consume_file()` orchestrates the full ingestion pipeline

**Module: `src/documents/views.py` — API-Triggered Task Dispatch**
- Key element: `PostDocumentView.post()` (line 497) — calls `async_task("documents.tasks.consume_file", ...)`
- Current documentation: `docs/api.rst` covers the upload endpoint but not the queuing mechanism
- Documentation needed: How an API upload creates a background job, including task_id generation and the immediate return pattern

**Module: `src/documents/management/commands/document_consumer.py` — Directory Watcher Task Dispatch**
- Key element: `_consume()` function (line 46) — calls `async_task("documents.tasks.consume_file", ...)`
- Current documentation: `docs/usage_overview.rst` mentions the consumer conceptually
- Documentation needed: How the directory watcher detects files and enqueues them as async tasks

**Module: `src/paperless_mail/mail.py` — Email-Triggered Task Dispatch**
- Key element: `MailAccountHandler.handle_message()` (line 272) — calls `async_task("documents.tasks.consume_file", ...)`
- Current documentation: `docs/usage_overview.rst` covers mail ingestion at a user level
- Documentation needed: How email attachment processing leads to task creation

**Module: `src/documents/bulk_edit.py` — Bulk Operation Task Dispatch**
- Key elements: `set_correspondent()`, `set_document_type()`, `add_tag()`, `remove_tag()`, `modify_tags()` — each calls `async_task("documents.tasks.bulk_update_documents", ...)`
- Current documentation: None
- Documentation needed: How bulk edits trigger follow-up background re-indexing

**Module: `src/documents/consumer.py` — Ingestion Pipeline & Status Broadcasting**
- Key element: `Consumer._send_progress()` (line 56), `Consumer.try_consume_file()` (line 180)
- Current documentation: None at this level of detail
- Documentation needed: The 10-stage pipeline and how WebSocket progress messages are emitted at each stage

**Module: `src/paperless/consumers.py` — WebSocket Status Consumer**
- Key element: `StatusConsumer` class — joins `status_updates` group, serializes events to JSON
- Current documentation: None
- Documentation needed: How clients receive real-time task progress

**Module: `docker/supervisord.conf` — Process Topology**
- Key elements: Three `[program:*]` sections defining `gunicorn`, `consumer`, `scheduler`
- Current documentation: Not documented beyond deployment setup guides
- Documentation needed: What each process does and how they interact at runtime

**Module: `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — Scheduled Task Registration**
- Key element: `schedule("paperless_mail.tasks.process_mail_accounts", ...)` with `Schedule.MINUTES` at 10-minute intervals
- Current documentation: Mentioned indirectly in `docs/usage_overview.rst`
- Documentation needed: How recurring tasks are registered and executed by Django-Q

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented runtime architecture:** No existing document explains the Supervisord three-process model, how the `qcluster` command spawns workers, or how tasks flow from producer → Redis → worker → database
- **Undocumented task lifecycle:** The transition from "queued in Redis" → "picked up by sentinel" → "dispatched to worker" → "result stored in DB" is entirely absent from existing docs
- **Undocumented task state storage:** Django-Q's ORM models (`Task`, `OrmQ`, `Schedule`) and their role in persisting task results are not documented anywhere
- **Undocumented WebSocket progress mechanism:** The `_send_progress()` → Redis channel layer → `StatusConsumer` → frontend pathway is not explained
- **Undocumented signal-driven post-processing:** The six handlers fired by `document_consumption_finished` represent background work that happens within a task but is invisible unless documented
- **Undocumented task dispatch call sites:** The four distinct modules that call `async_task()` are not mapped in any existing documentation

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single, self-contained Markdown document that comprehensively answers the user's questions about background processing in paperless-ngx. The document structure is designed to mirror the user's inquiry flow — from high-level runtime overview down to code-level specifics.

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Overview (purpose and context)
    ├── Runtime Service Topology
    │   ├── Supervisord process model
    │   ├── The three managed processes
    │   └── Redis as central communication hub
    ├── How Background Jobs Are Created
    │   ├── API upload path (PostDocumentView)
    │   ├── Directory watcher path (document_consumer)
    │   ├── Email ingestion path (MailAccountHandler)
    │   ├── Bulk edit path (bulk_edit.py)
    │   └── Scheduled tasks (mail fetching)
    ├── Task Lifecycle: From Queue to Completion
    │   ├── Django-Q cluster architecture
    │   ├── Queued vs. actively processing
    │   ├── The consume_file pipeline
    │   └── Signal-driven post-processing
    ├── Where Task State Is Stored
    │   ├── Django-Q ORM models
    │   ├── Redis transient state
    │   └── Database persistent state
    ├── Observing What Happened to a Job
    │   ├── Real-time: WebSocket status_updates
    │   ├── Post-hoc: Task model queries
    │   └── Logs: paperless.log and mail.log
    └── Source Code Reference Map
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the `Q_CLUSTER` configuration block from `src/paperless/settings.py` (lines 449–457) to explain worker pool settings
- Extract `async_task()` call signatures from all four producer modules to map task creation patterns
- Extract the `Consumer.try_consume_file()` method flow from `src/documents/consumer.py` (lines 180–377) to document the ingestion pipeline stages
- Extract the `StatusConsumer` class from `src/paperless/consumers.py` to explain WebSocket broadcasting
- Extract the six signal handler registrations from `src/documents/apps.py` (lines 22–27) to document post-processing hooks
- Extract the scheduled task registration from `src/paperless_mail/migrations/0002_auto_20201117_1334.py` to explain recurring jobs

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for: the runtime process topology, the task flow from creation to completion, and the WebSocket broadcast pathway
- Code references using the format `Source: /path/to/file.py:LineNumber`
- Tables for parameter descriptions, task function inventories, and call site mappings
- Consistent terminology: "task" (Django-Q unit of work), "worker" (Django-Q subprocess), "consumer" (directory watcher process), "pipeline" (ingestion workflow within consume_file)

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created for the output document:

- **Runtime Process Topology Diagram:** A graph showing Supervisord managing three processes (Gunicorn, Consumer, Scheduler/QCluster) with Redis as the shared communication hub and the database/filesystem as shared storage
- **Task Lifecycle Sequence Diagram:** A sequence diagram showing the flow: Producer → `async_task()` → Redis queue → QCluster sentinel → Worker process → `consume_file()` → Database result → WebSocket broadcast
- **Task Dispatch Call Site Map:** A flowchart showing the four entry points (API, directory watcher, email, bulk edit) converging at the Redis task queue
- **Ingestion Pipeline Stage Diagram:** A flowchart of the 10 stages within `Consumer.try_consume_file()` showing status broadcast points

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/tasks.py`, `src/documents/consumer.py`, `src/documents/views.py`, `src/documents/bulk_edit.py`, `src/documents/management/commands/document_consumer.py`, `src/paperless_mail/mail.py`, `src/paperless/settings.py`, `src/paperless/consumers.py`, `src/paperless/asgi.py`, `src/documents/signals/__init__.py`, `src/documents/apps.py`, `src/paperless_mail/migrations/0002_auto_20201117_1334.py`, `docker/supervisord.conf`, `docker/docker-prepare.sh`, `docker/wait-for-redis.py`, `gunicorn.conf.py` | Complete technical deep-dive document answering all user questions about background processing behavior at runtime, with Mermaid diagrams, code citations, and rationale |

This is the **only** file to be created. No existing files are modified, updated, or deleted per the user's explicit instruction.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Deep-Dive / Architecture Q&A Document
Source Code: 16 source files across src/documents/, src/paperless/, src/paperless_mail/, and docker/
Sections:
    - Overview and Context
    - Runtime Service Topology (from docker/supervisord.conf, src/paperless/settings.py)
    - How Background Jobs Are Created (from 4 async_task producer modules)
    - Task Lifecycle: Queued vs. Active (from Q_CLUSTER config and Django-Q model)
    - The consume_file Pipeline (from src/documents/tasks.py, src/documents/consumer.py)
    - Signal-Driven Post-Processing (from src/documents/apps.py, src/documents/signals/handlers.py)
    - Where Task State Is Stored (from Django-Q ORM models: Task, OrmQ, Schedule)
    - Observing Job Outcomes (WebSocket status_updates, Task model queries, log files)
    - Complete Source Code Reference Map (all async_task call sites with line numbers)
Diagrams:
    - Mermaid graph: Runtime process topology
    - Mermaid sequence: Task lifecycle from creation to completion
    - Mermaid flowchart: Task dispatch convergence from 4 entry points
    - Mermaid flowchart: Ingestion pipeline stages with status broadcast points
Key Citations:
    - src/paperless/settings.py (Q_CLUSTER, CHANNEL_LAYERS, TASK_WORKERS)
    - src/documents/tasks.py (consume_file, bulk_update_documents, train_classifier)
    - src/documents/consumer.py (Consumer._send_progress, Consumer.try_consume_file)
    - src/documents/views.py (PostDocumentView.post)
    - src/documents/management/commands/document_consumer.py (_consume)
    - src/paperless_mail/mail.py (MailAccountHandler.handle_message)
    - src/documents/bulk_edit.py (set_correspondent, set_document_type, add_tag, remove_tag, modify_tags)
    - src/paperless/consumers.py (StatusConsumer)
    - src/documents/apps.py (DocumentsConfig.ready - 6 signal handlers)
    - src/documents/signals/__init__.py (3 domain signals)
    - docker/supervisord.conf (3 supervised processes)
    - docker/docker-prepare.sh (startup readiness probes)
    - docker/wait-for-redis.py (Redis health check)
    - src/paperless_mail/migrations/0002_auto_20201117_1334.py (scheduled mail task)
    - gunicorn.conf.py (ASGI server config)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone Markdown document placed in the `blitzy/documentation/` directory. It does not integrate into the Sphinx documentation build system (which uses `.rst` files in `docs/`) and does not require changes to `docs/conf.py`, `docs/Makefile`, or `.readthedocs.yml`.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The output document is self-contained with no dependencies on existing documentation infrastructure
- **No navigation links required:** The document is not part of the Sphinx toctree
- **No index/glossary updates:** The document stands alone in the `blitzy/documentation/` directory
- **Internal cross-references:** The document will use relative file paths (e.g., `src/documents/tasks.py`) as source citations, maintaining traceability back to the codebase

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to this documentation exercise because they implement the background processing features being documented. Versions are taken directly from the pinned `requirements.txt` manifest in the repository root.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | 4.0.4 | Core web framework providing ORM, signals, management commands |
| pip | django-q | 1.3.9 | Task queue framework: async_task dispatch, qcluster workers, ORM-backed result storage, scheduled tasks |
| pip | redis | 3.5.3 | Python Redis client used by Django-Q as broker and by Channels as WebSocket layer |
| pip | channels | 3.0.4 | Django ASGI extension providing WebSocket support and the channel layer abstraction |
| pip | channels-redis | 3.4.0 | Redis-backed channel layer for broadcasting status_updates group messages |
| pip | gunicorn | 20.1.0 | ASGI/WSGI server that hosts the Django application in production |
| pip | uvicorn | 0.17.6 | ASGI server used as the Gunicorn worker class via `paperless.workers.ConfigurableWorker` |
| pip | watchdog | 2.1.7 | Filesystem monitoring library used by document_consumer for polling-based file detection |
| pip | inotifyrecursive | 0.3.5 | Linux inotify bindings for native filesystem event detection (preferred over polling) |
| pip | asgiref | 3.5.0 | ASGI utilities including `async_to_sync` used to bridge sync code to async channel layer |
| pip | python-magic | 0.4.25 | MIME type detection for incoming documents |
| pip | filelock | 3.6.0 | Advisory file locking for `MEDIA_LOCK` protecting concurrent filesystem mutations |
| pip | concurrent-log-handler | 0.9.20 | Multi-process safe rotating log handler for `paperless.log` and `mail.log` |
| system | supervisord | (system) | Process manager running gunicorn, consumer, and scheduler within the Docker container |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation links require updates since the output is a new standalone document. No link transformation rules are needed.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (before this documentation task):**
- Background processing runtime behavior documented: 0/8 user questions (0%)
- Task dispatch call sites documented: 0/11 `async_task()` calls (0%)
- Task lifecycle states explained: 0/4 states (queued, active, success, failure) (0%)
- WebSocket status mechanism documented: 0/1 (0%)

**Target coverage (after this documentation task):**
- Background processing runtime behavior documented: 8/8 user questions (100%)
- Task dispatch call sites documented: 11/11 `async_task()` calls (100%)
- Task lifecycle states explained: 4/4 states (100%)
- WebSocket status mechanism documented: 1/1 (100%)

**Coverage gaps to address:**

| Area | Current | Target | Focus |
|------|---------|--------|-------|
| Runtime service topology | 0% | 100% | Supervisord processes, Redis roles, communication paths |
| Task creation entry points | 0% | 100% | All 4 producer modules, all 11 async_task calls |
| Task lifecycle states | 0% | 100% | Queued → Active → Success/Failure with Django-Q internals |
| Task state storage | 0% | 100% | Django-Q ORM models, Redis transient state, DB persistence |
| Post-hoc job inspection | 0% | 100% | Task model queries, WebSocket history, log file locations |
| Signal-driven post-processing | 0% | 100% | 6 handlers in document_consumption_finished |
| Scheduled recurring tasks | 0% | 100% | Mail fetching every 10 minutes via Django-Q Schedule |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question (R1–R8 from Section 0.1.1) must have a dedicated section with a clear, grounded answer
- Every `async_task()` call site must be documented with module path, line number, function name, and the task string it dispatches
- The task lifecycle must be explained end-to-end: from producer call to Redis queue to worker pickup to result storage
- All Mermaid diagrams must accurately reflect the code — no hypothetical or assumed relationships

**Accuracy validation:**
- All code references must cite actual file paths and line numbers verified through `read_file` tool calls during this analysis
- All configuration values (e.g., `Q_CLUSTER` parameters) must quote the actual defaults from `src/paperless/settings.py`
- No claims about Django-Q internals that are not evidenced by the codebase's usage patterns or the pinned library version

**Clarity standards:**
- Explanations must be accessible to a developer who is unfamiliar with Django-Q but understands Django basics
- Progressive disclosure: start with the high-level process topology, then drill into task creation, then lifecycle, then state storage
- Consistent terminology table at the start of the document to prevent confusion between "consumer" (directory watcher), "Consumer" (Python class), and "consumer" (in the pub/sub sense)

**Maintainability:**
- All source citations include file paths for traceability
- Document is standalone and does not depend on external documentation infrastructure

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 Mermaid diagrams (process topology, task lifecycle sequence, dispatch convergence, pipeline stages)
- **Code example testing:** Not applicable — the document explains existing code, not new code. All code references are read-only citations.
- **Visual content freshness:** Diagrams are generated from the current codebase analysis and are accurate as of the branch `paperless-ngx_542221a38dff`

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable

**Source code analyzed (read-only — no modifications):**
- `src/documents/tasks.py` — all task function definitions
- `src/documents/consumer.py` — ingestion pipeline and WebSocket progress broadcasting
- `src/documents/views.py` — `PostDocumentView.post()` task dispatch via API upload
- `src/documents/bulk_edit.py` — all five bulk edit functions dispatching `bulk_update_documents`
- `src/documents/management/commands/document_consumer.py` — directory watcher `_consume()` function
- `src/documents/apps.py` — `DocumentsConfig.ready()` signal handler registration
- `src/documents/signals/__init__.py` — three domain signal definitions
- `src/documents/signals/handlers.py` — all post-consumption handlers
- `src/documents/models.py` — Document model referenced in tasks
- `src/paperless/settings.py` — `Q_CLUSTER`, `CHANNEL_LAYERS`, `TASK_WORKERS`, Redis URL, logging config
- `src/paperless/consumers.py` — `StatusConsumer` WebSocket handler
- `src/paperless/asgi.py` — `ProtocolTypeRouter` for HTTP/WebSocket multiplexing
- `src/paperless/urls.py` — WebSocket URL patterns (`ws/status/$`)
- `src/paperless_mail/mail.py` — `MailAccountHandler.handle_message()` task dispatch
- `src/paperless_mail/tasks.py` — `process_mail_accounts()` and `process_mail_account()`
- `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — scheduled task registration
- `docker/supervisord.conf` — three supervised processes
- `docker/docker-prepare.sh` — startup readiness sequence
- `docker/wait-for-redis.py` — Redis health probe
- `gunicorn.conf.py` — ASGI server configuration
- `Pipfile` — dependency declarations
- `requirements.txt` — pinned dependency versions

**Documentation assets:**
- Mermaid diagrams embedded inline in the output Markdown file

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in `src/`, `docker/`, `docs/`, or any other existing directory will be modified, per the user's explicit instruction
- **Test file modifications:** No test files will be created, modified, or referenced as deliverables
- **Existing documentation updates:** No changes to `docs/*.rst` files, `docs/conf.py`, `docs/Makefile`, or `.readthedocs.yml`
- **Feature additions or code refactoring:** This is strictly a documentation exercise
- **Frontend code analysis:** The Angular frontend in `src-ui/` is not in scope; the document focuses on backend processing
- **Deployment configuration changes:** No changes to Docker Compose files, `Dockerfile`, or environment files
- **Tika/Tesseract/OCR parser internals:** The document explains how tasks are created and managed, not the internal workings of individual parsers
- **Database schema documentation:** The document references Django-Q's ORM models conceptually but does not document database schema details beyond what is needed to answer "where is task state stored"
- **Temporary scripts:** The user allowed creation of temporary observation scripts, but the codebase analysis via tooling was sufficient — no temporary scripts were needed

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not integrated into the Sphinx build system
- **Documentation preview command:** Standard Markdown rendering (e.g., `cat blitzy/documentation/paperless-ngx_542221a38dff.md` or any Markdown viewer)
- **Diagram generation command:** Mermaid diagrams are embedded as fenced code blocks in the Markdown and render natively in GitHub, GitLab, and compatible Markdown renderers
- **Documentation deployment command:** Not applicable — the file is committed to the repository
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim must reference specific source files with path notation (e.g., `Source: src/documents/tasks.py:184`)
- **Style guide:** Technical investigative narrative — explanatory, grounded in code, with rationale for each conclusion. Matches the user's request to "make sense of" and "understand" runtime behavior
- **Documentation validation:** The Markdown structure can be validated with any standard Markdown linter. Mermaid syntax can be validated with the mermaid-cli (`mmdc`) if needed, though this is not a build requirement

## 0.10 Rules for Documentation

The following rules govern the creation of the output documentation, derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The output is exclusively a new file at `blitzy/documentation/paperless-ngx_542221a38dff.md`. No other files may be created, modified, or deleted.
- **Do not make assumptions; base all answers on the code as the truth.** Every claim about runtime behavior must be traceable to specific source files, line numbers, and configuration values observed in the repository during analysis.
- **Provide thinking and rationale behind the answers.** The document should not just state facts but explain *why* the system behaves a certain way, connecting configuration values to runtime outcomes.
- **Create the output document in the `blitzy/documentation` directory** with the filename matching the source branch name: `paperless-ngx_542221a38dff.md`.
- **If temporary scripts or artifacts are created to observe behavior, clean them up afterward and leave the codebase unchanged.** In this case, no temporary scripts were needed — all analysis was performed through tooling-based file reads and searches.
- **Use Mermaid diagrams for all workflow and architecture visualizations** to maintain consistency with the technical specification's visual standards.
- **Include source code citations for all technical details** using the format `Source: path/to/file.py:LineNumber` or `Source: path/to/file.py` for general file references.
- **Maintain consistent terminology** to avoid confusion between overloaded terms (e.g., "consumer" as the directory watcher process vs. `Consumer` as the Python class vs. "consumer" in the pub/sub pattern).

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

The following is a comprehensive list of every file and folder retrieved or searched during the analysis phase to derive the conclusions in this Agent Action Plan.

**Folder-Level Exploration (via `get_source_folder_contents`):**
- `` (repository root) — identified top-level structure and all major directories
- `src/` — identified all Django application packages
- `src/documents/` — identified core ingestion engine modules
- `src/paperless/` — identified core project configuration and runtime modules
- `src/paperless_mail/` — identified mail ingestion subsystem
- `src/documents/signals/` — identified signal definitions and handlers
- `src/documents/management/` — identified management command namespace
- `src/documents/management/commands/` — identified all CLI commands including `document_consumer.py`
- `docs/` — identified existing Sphinx documentation workspace
- `docker/` — identified container runtime and process management assets

**File-Level Reads (via `read_file`):**
- `src/documents/tasks.py` (full) — task function definitions
- `src/documents/consumer.py` (full) — ingestion pipeline and status broadcasting
- `src/documents/views.py` (lines 1–100, 100–250, 250–500, 491–620, 620–710) — API views and PostDocumentView
- `src/documents/bulk_edit.py` (full) — bulk operation task dispatch
- `src/documents/management/commands/document_consumer.py` (full) — directory watcher implementation
- `src/documents/apps.py` (full) — signal handler registration
- `src/documents/signals/__init__.py` (full) — domain signal definitions
- `src/documents/signals/handlers.py` (full) — post-consumption handlers
- `src/documents/models.py` (lines 1–50) — model declarations
- `src/paperless/settings.py` (full) — all Django settings including Q_CLUSTER, CHANNEL_LAYERS
- `src/paperless/consumers.py` (full) — WebSocket StatusConsumer
- `src/paperless/asgi.py` (full) — ASGI protocol routing
- `src/paperless/urls.py` (full) — URL patterns including WebSocket routes
- `src/paperless_mail/mail.py` (full) — IMAP processing and task dispatch
- `src/paperless_mail/tasks.py` (full) — mail account processing functions
- `src/paperless_mail/migrations/0002_auto_20201117_1334.py` (full) — scheduled task registration
- `docker/supervisord.conf` (full) — process management configuration
- `docker/docker-prepare.sh` (full) — container startup readiness probes
- `docker/wait-for-redis.py` (full) — Redis health check script
- `gunicorn.conf.py` (full) — ASGI server configuration
- `Pipfile` (full) — dependency declarations
- `requirements.txt` (full) — pinned dependency versions
- `docs/conf.py` (full) — Sphinx documentation configuration

**Bash Searches Performed:**
- `find / -name ".blitzyignore"` — checked for ignore patterns (none found)
- `grep -rn "async_task\|from django_q" src/` — mapped all task dispatch call sites (11 found across 4 modules)
- `grep -rn "schedule\|Schedule" src/paperless_mail/migrations/` — identified scheduled task registration
- `grep -n "task\|worker\|queue\|background" docs/configuration.rst` — checked existing docs coverage
- `grep -n "task\|worker\|queue\|background\|consume" docs/usage_overview.rst` — checked existing docs coverage
- `git rev-parse --abbrev-ref HEAD` — identified branch name: `paperless-ngx_542221a38dff`

**Tech Spec Sections Retrieved (via `get_tech_spec_section`):**
- `1.1 Executive Summary` — project overview and stakeholder context
- `5.1 High-Level Architecture` — system architecture, data flow, integration points
- `6.1 Core Services Architecture` — multi-process container architecture, inter-process communication, resilience patterns

### 0.11.2 Attachments Provided

No attachments were provided by the user for this task. No Figma screens, templates, or supplementary files were included.

