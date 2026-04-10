# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive investigative document** that traces and explains the full lifecycle of a document as it flows through the Paperless-NGX ingestion pipeline — from the moment a file lands in the consumption directory through task queuing, parsing, classification, indexing, and final persistence. The user wants to learn by doing: setting up the environment, running the system, and observing its behavior first-hand before working with the codebase.

**Request Category:** Create new documentation

**Documentation Type:** Technical deep-dive / Architecture walkthrough / Internals reference

The user's requirements decompose into the following precise documentation goals:

- **Environment Setup & Runtime Observation** — Stand up the full Paperless-NGX stack (web server, file consumer, Django-Q worker cluster, Redis broker, database) so that all services are operational and ready to process documents.
- **Ingestion Pipeline Tracing** — Drop a test file into the consumption directory, then document every observable step: log messages at file detection, the specific task that gets created, and subsequent tasks that fire as the document moves through parsing, classification, and indexing.
- **Message Broker Inspection** — While a task is queued in Redis via Django-Q, inspect the broker to capture the payload format and data structure that represents a queued task.
- **Database Record Inspection** — After processing completes, query the database (SQLite or PostgreSQL) to identify the table and fields where the document record lives, the metadata columns populated by the pipeline, and any task execution or processing history records.
- **Codepath Tracing** — Using observed runtime behavior as a guide, trace the high-level code path from file detection through task submission, identifying the components responsible for creating the task object and the queuing framework Paperless-NGX relies upon.

### 0.1.2 Special Instructions and Constraints

The user has specified critical operational constraints that must be respected throughout:

- **No source code modifications** — The actual Paperless-NGX source code must not be altered. Temporary helper scripts or test files are permitted but must be cleaned up afterward.
- **Hands-on observational approach** — The documentation is to be derived from real runtime behavior, not solely from static code reading. The user expects log output, broker state, and database query results to be captured.
- **Cleanup requirement** — Any temporary artifacts (test files, helper scripts) must be removed after the investigation is complete.
- **Implementation rule (SWE-AtlasQnA-Repo)** — The output must be a new markdown document named `paperless-ngx_542221a38dff.md`, placed in the `blitzy/documentation` directory. It must provide thinking/rationale behind the answers, base conclusions on the code as truth, and not modify any existing source repository files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the environment setup**, we will install all dependencies per `Pipfile` / `requirements.txt`, configure Redis, set up the SQLite database via `manage.py migrate`, and start the three Supervisord-managed processes: Gunicorn (ASGI server), `document_consumer` (directory watcher), and `qcluster` (Django-Q worker pool).
- To **trace the ingestion pipeline**, we will create a small `.txt` test file in `CONSUMPTION_DIR`, then monitor logs under the `paperless.management.consumer`, `paperless.consumer`, `paperless.tasks`, `paperless.parsing`, `paperless.matching`, `paperless.handlers`, and `paperless.index` logging namespaces.
- To **inspect the message broker**, we will query Redis directly (via `redis-cli` or Python `redis` library) to examine the Django-Q queue keys, capturing the pickled/serialized task payload structure.
- To **query the database**, we will use `sqlite3` or Django shell to examine the `documents_document` table, the `documents_log` table, and the Django-Q ORM tables (`django_q_ormq`, `django_q_task`, `django_q_schedule`) for task execution history.
- To **trace the codepath**, we will follow the chain: `document_consumer.py` (watchdog/inotify detection) → `_consume()` → `django_q.tasks.async_task("documents.tasks.consume_file", ...)` → `tasks.py:consume_file()` → `consumer.py:Consumer.try_consume_file()` → signal handlers in `apps.py` → `handlers.py` (classification, tagging, indexing).

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Django-Q Task Lifecycle** — The user asks about queued task payloads and task execution history. The documentation must explain how Django-Q serializes tasks into Redis, what the `django_q.models.Task` / `OrmQ` models look like after execution, and how success/failure is tracked.
- **Signal Handler Chain** — The `document_consumption_finished` signal triggers six handlers registered in `src/documents/apps.py`. The documentation must enumerate these handlers (inbox tags, correspondent assignment, document type assignment, tag assignment, admin log entry, search index update) and explain their execution order.
- **Parser Discovery Mechanism** — The `document_consumer_declaration` signal dispatches parser selection across `paperless_tesseract`, `paperless_text`, and `paperless_tika`. This mechanism needs to be documented.
- **Concurrency and Locking** — The pipeline uses `transaction.atomic()`, `FileLock`, and `AsyncWriter` for data integrity. These need to be captured to explain why the system behaves as observed.
- **WebSocket Status Broadcasting** — Progress messages are broadcast to the `status_updates` channel group via Redis Channel Layer throughout the pipeline. The documentation should note these as observable side effects.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature **Sphinx-based documentation workspace** located at `docs/`, with moderate coverage of user-facing topics but no dedicated internal-architecture or ingestion-pipeline deep-dive documentation.

**Documentation Framework:**

- **Generator:** Sphinx (version `~=4.5.0` as pinned in `Pipfile` dev dependencies)
- **Theme:** `sphinx_rtd_theme` (Read the Docs theme), configured in `docs/conf.py`
- **Configuration file:** `docs/conf.py` — sets project metadata, extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), static asset paths, and theme assets
- **Build entry point:** `docs/Makefile` (standard Sphinx build targets: `html`, `epub`, `latexpdf`, `linkcheck`, `doctest`)
- **Docker preview:** `docs/Dockerfile` (builds docs and serves on port 8000 via Python HTTP server)
- **Hosted at:** ReadTheDocs (`.readthedocs.yml` at repo root)
- **Source format:** reStructuredText (`.rst`)
- **Dependency manifest:** `docs/requirements.txt` (currently empty placeholder)

**Existing Documentation Pages (from `docs/index.rst` toctree):**

| Document | Path | Coverage Area |
|----------|------|---------------|
| Setup & Installation | `docs/setup.rst` | Installation, migration, reverse proxy, deployment |
| Usage Overview | `docs/usage_overview.rst` | Product model, ingestion methods, search, workflows |
| Advanced Usage | `docs/advanced_usage.rst` | Advanced matching, hooks, filename handling |
| Administration | `docs/administration.rst` | Backups, updates, utilities, indexing, encryption |
| Configuration | `docs/configuration.rst` | Environment variables, runtime settings |
| REST API | `docs/api.rst` | Endpoints, authentication, uploads, versioning |
| FAQ | `docs/faq.rst` | Common support questions |
| Troubleshooting | `docs/troubleshooting.rst` | Operational failures and fixes |
| Extending | `docs/extending.rst` | Contributor workflows, dev setup, parser extension |
| Scanners | `docs/scanners.rst` | Compatible scanner hardware and mobile apps |
| Screenshots | `docs/screenshots.rst` | Visual gallery of UI |
| Changelog | `docs/changelog.rst` | Release history across Paperless, Paperless-ng, Paperless-ngx |

**Documentation Gap Identified:** None of these existing pages provides a step-by-step walkthrough of the internal document processing pipeline from the perspective of runtime observation — covering log messages, Redis broker payloads, database records, and code-path tracing. The user's request maps to a **net-new document** that fills this gap.

### 0.2.2 Repository Code Analysis for Documentation

The following source files and directories were examined to extract the information needed for the new documentation:

**Ingestion Pipeline Core:**
- `src/documents/management/commands/document_consumer.py` — File detection via watchdog/inotify, task enqueueing via `django_q.tasks.async_task`
- `src/documents/tasks.py` — `consume_file()` entry point, barcode splitting, index/classifier tasks
- `src/documents/consumer.py` — `Consumer.try_consume_file()` orchestrator: validation, parsing, classification, atomic persistence
- `src/documents/signals/__init__.py` — Three domain signals: `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration`
- `src/documents/signals/handlers.py` — Six `consumption_finished` handlers + `post_save`/`post_delete` handlers
- `src/documents/apps.py` — Signal handler registration in `DocumentsConfig.ready()`

**Parser Selection & Execution:**
- `src/documents/parsers.py` — Parser discovery via `document_consumer_declaration` signal, MIME detection, date extraction
- `src/paperless_tesseract/signals.py` — Tesseract parser declaration (PDF, JPEG, PNG, TIFF, GIF, BMP)
- `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser` (OCRmyPDF-based)
- `src/paperless_text/` — Plain text parser for `.txt` files
- `src/paperless_tika/` — Tika-based parser (feature-flagged)

**Classification & Indexing:**
- `src/documents/classifier.py` — `DocumentClassifier` (scikit-learn MLPClassifier, FORMAT_VERSION 7)
- `src/documents/matching.py` — Rule-based matching (6 algorithms)
- `src/documents/index.py` — Whoosh 2.7.4 full-text search index

**Data Models & Persistence:**
- `src/documents/models.py` — `Document`, `Correspondent`, `Tag`, `DocumentType`, `Log`, `SavedView`, `FileInfo`
- `src/documents/file_handling.py` — Filename generation, directory management

**Configuration & Infrastructure:**
- `src/paperless/settings.py` — Django-Q `Q_CLUSTER` config, directory paths, consumer settings, logging
- `docker/supervisord.conf` — Three supervised processes: gunicorn, consumer, scheduler (qcluster)
- `docker/docker-prepare.sh` — Startup sequence: Redis/DB readiness, migrations, index rebuild

### 0.2.3 Web Search Research Conducted

No external web search is required for this documentation task. The codebase provides all the information needed to trace the ingestion pipeline. The key frameworks involved — Django-Q (v1.3.9), Whoosh (v2.7.4), watchdog (v2.1.7), Redis — are well-understood from the code and their pinned versions in `requirements.txt`.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The new documentation document must cover the following modules and components, each contributing a piece of the ingestion-to-indexing narrative:

**Module: File Detection Layer**
- Source: `src/documents/management/commands/document_consumer.py`
- Public APIs: `Command.handle()`, `_consume()`, `_consume_wait_unmodified()`, `Handler.on_created()`, `Handler.on_moved()`, `_tags_from_path()`, `_is_ignored()`
- Current documentation: User-facing configuration in `docs/configuration.rst`; no internals walkthrough
- Documentation needed: Runtime behavior description, log messages at file detection, watchdog vs. inotify paths, file stabilization wait logic

**Module: Task Queuing Layer**
- Source: `src/documents/management/commands/document_consumer.py` (lines 84–91), `src/documents/views.py` (lines 521–533), `src/paperless_mail/mail.py` (lines 336–337)
- Framework: Django-Q v1.3.9 (`django_q.tasks.async_task`)
- Configuration: `src/paperless/settings.py` `Q_CLUSTER` (lines 449–457)
- Current documentation: Not documented at internals level
- Documentation needed: `async_task()` call signature and parameters, Redis key structure, serialized task payload format, Django-Q ORM task model

**Module: Consumer Pipeline**
- Source: `src/documents/consumer.py` — `Consumer` class
- Public APIs: `try_consume_file()`, `pre_check_file_exists()`, `pre_check_duplicate()`, `pre_check_directories()`, `run_pre_consume_script()`, `_store()`, `apply_overrides()`, `_write()`
- Current documentation: High-level overview in `docs/usage_overview.rst`; no step-by-step internals
- Documentation needed: Full pipeline stage documentation with log messages, MIME detection, parser dispatch, progress WebSocket messages

**Module: Parser Dispatch**
- Source: `src/documents/parsers.py`, `src/paperless_tesseract/signals.py`, `src/paperless_text/signals.py`, `src/paperless_tika/signals.py`
- Key mechanism: `document_consumer_declaration` signal for parser registration, `get_parser_class_for_mime_type()` for selection
- Documentation needed: How parsers register themselves, weight-based selection, supported MIME types per parser

**Module: Classification & Matching**
- Source: `src/documents/classifier.py`, `src/documents/matching.py`
- Key APIs: `load_classifier()`, `DocumentClassifier.predict_*()`, `match_correspondents()`, `match_document_types()`, `match_tags()`
- Documentation needed: How ML and rule-based matching are combined during post-consumption signal handling

**Module: Signal Handlers (Post-Consumption)**
- Source: `src/documents/apps.py` (registration), `src/documents/signals/handlers.py` (implementation)
- Six handlers: `add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index`
- Documentation needed: Execution order, what each handler does, expected log output

**Module: Search Indexing**
- Source: `src/documents/index.py`
- Key APIs: `add_or_update_document()`, `update_document()`, `open_index_writer()`
- Schema: 16 fields in Whoosh schema (id, title, content, asn, correspondent, tags, type, dates, etc.)
- Documentation needed: What gets indexed, Whoosh AsyncWriter behavior

**Module: Database Persistence**
- Source: `src/documents/models.py`
- Key models: `Document` (15 fields), `Log` (4 fields), plus Django-Q `Task` model
- Documentation needed: Schema of `documents_document` table, `documents_log` table, Django-Q task history tables

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist that the new document will fill:

- **No existing internals-level pipeline walkthrough** — The `docs/usage_overview.rst` covers the user's perspective of ingestion but does not trace log messages, task payloads, or database records
- **No documentation of Django-Q task structure** — The task queuing framework is mentioned in `docs/configuration.rst` via `PAPERLESS_TASK_WORKERS` but the actual task payload format, Redis key structure, and task history model are undocumented
- **No log-message reference for the ingestion flow** — Log namespaces (`paperless.management.consumer`, `paperless.consumer`, `paperless.handlers`, `paperless.index`) and their specific messages during file processing are not documented
- **No database schema walkthrough** — While the tech spec (Section 6.2) provides a comprehensive schema reference, there is no document that walks through the schema from an observational perspective (i.e., "after processing, these fields are populated")
- **No broker inspection guide** — Redis key patterns used by Django-Q and Channels are not documented in any existing material

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `paperless-ngx_542221a38dff.md` will be placed in the `blitzy/documentation` directory and organized as a single comprehensive Markdown file with the following structure:

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Introduction & Objective
    ├── Environment Setup
    │   ├── Prerequisites & Dependencies
    │   ├── Service Startup (Redis, DB, Consumer, QCluster, Web)
    │   └── Verification of Running Services
    ├── File Detection & Task Queuing
    │   ├── Dropping a Test File
    │   ├── Log Messages at Detection
    │   ├── Task Creation via async_task()
    │   └── Code Path: consumer command → _consume() → async_task()
    ├── Message Broker Inspection
    │   ├── Redis Key Structure for Django-Q
    │   ├── Queued Task Payload Format
    │   └── Channel Layer Keys (status_updates)
    ├── Document Processing Pipeline
    │   ├── Consumer.try_consume_file() Stage-by-Stage
    │   ├── Validation & Deduplication
    │   ├── MIME Detection & Parser Dispatch
    │   ├── Text Extraction & Thumbnail Generation
    │   ├── Date Parsing
    │   ├── Classification (ML + Rules)
    │   └── Log Messages Through Processing
    ├── Post-Consumption Signal Handlers
    │   ├── Handler Registration (apps.py)
    │   ├── add_inbox_tags
    │   ├── set_correspondent
    │   ├── set_document_type
    │   ├── set_tags
    │   ├── set_log_entry
    │   ├── add_to_index
    │   └── Atomic Persistence & File Copy
    ├── Database Inspection After Processing
    │   ├── documents_document Table & Fields
    │   ├── documents_log Table (Processing History)
    │   ├── Django-Q Task History (django_q_task)
    │   └── Whoosh Search Index Contents
    ├── High-Level Code Path Summary
    │   ├── Component Chain Diagram
    │   └── Framework Identification (Django-Q, Watchdog, Whoosh)
    └── Cleanup & Conclusion
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract runtime behavior from the three Supervisord-managed processes by starting them and observing log output under `LOGGING_DIR` and stdout"
- "Capture task queuing behavior by monitoring Redis keys during the window between file detection and task execution"
- "Query the SQLite database directly via `sqlite3 DATA_DIR/db.sqlite3` after document consumption completes"
- "Generate Mermaid diagrams by mapping the component chain discovered in `src/documents/management/commands/document_consumer.py` → `src/documents/tasks.py` → `src/documents/consumer.py` → `src/documents/signals/handlers.py`"
- "Extract code path information from static analysis of imports and function calls across the pipeline modules"

**Documentation Standards:**

- Markdown formatting with proper headers (# ## ### ####)
- Mermaid diagram integration for the ingestion pipeline flow and component chain
- Code examples showing actual log output, Redis payloads, and database query results
- Source citations as inline references (e.g., `Source: src/documents/consumer.py:215`)
- Tables for database schema descriptions and signal handler summaries

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **Ingestion Pipeline Flowchart** — End-to-end flow from file drop to database persistence, showing all decision points (barcode check, MIME detection, parser selection)
- **Component Chain Sequence Diagram** — `document_consumer.py` → `async_task()` → `tasks.consume_file()` → `Consumer.try_consume_file()` → signal handlers
- **Signal Handler Execution Diagram** — The six `document_consumption_finished` handlers and their effects
- **Database Entity Relationship** — `documents_document` with relationships to `documents_correspondent`, `documents_documenttype`, `documents_tag`, `documents_log`, and Django-Q task tables

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

Per the implementation rule **SWE-AtlasQnA-Repo**, the output is a single new Markdown document named after the source branch, placed in `blitzy/documentation`. No existing files in the source repository are modified.

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/management/commands/document_consumer.py`, `src/documents/tasks.py`, `src/documents/consumer.py`, `src/documents/signals/handlers.py`, `src/documents/apps.py`, `src/documents/models.py`, `src/documents/index.py`, `src/documents/classifier.py`, `src/documents/matching.py`, `src/documents/parsers.py`, `src/paperless/settings.py`, `docker/supervisord.conf`, `docker/docker-prepare.sh` | Comprehensive investigative document tracing the Paperless-NGX document ingestion pipeline: environment setup, file detection logs, task queuing via Django-Q, Redis broker payload inspection, consumer pipeline stages, parser dispatch, classification, signal-driven post-processing, database record inspection, Whoosh index verification, and full code-path trace with Mermaid diagrams |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Deep-Dive / Architecture Walkthrough
Source Code:
  - src/documents/management/commands/document_consumer.py
  - src/documents/tasks.py
  - src/documents/consumer.py
  - src/documents/signals/__init__.py
  - src/documents/signals/handlers.py
  - src/documents/apps.py
  - src/documents/models.py
  - src/documents/index.py
  - src/documents/classifier.py
  - src/documents/matching.py
  - src/documents/parsers.py
  - src/paperless_tesseract/signals.py
  - src/paperless_tesseract/parsers.py
  - src/paperless_text/signals.py
  - src/paperless/settings.py
  - src/paperless/version.py
  - src/documents/loggers.py
  - src/documents/file_handling.py
  - src/documents/views.py (PostDocumentView)
  - src/paperless_mail/mail.py (MailAccountHandler)
  - docker/supervisord.conf
  - docker/docker-prepare.sh
  - docker/docker-entrypoint.sh
  - Pipfile, requirements.txt

Sections:
  - Introduction: Purpose, scope, version (1.7.0)
  - Environment Setup: Dependencies, service startup, verification
  - File Detection: Watchdog/inotify, _consume(), log messages
  - Task Queuing: async_task() call, Django-Q Redis structure, task payload
  - Consumer Pipeline: try_consume_file() 10-stage walkthrough
  - Parser Dispatch: document_consumer_declaration signal, MIME-based selection
  - Classification: ML classifier + rule-based matching
  - Signal Handlers: Six registered handlers, execution flow
  - Database Inspection: documents_document schema, documents_log, django_q_task
  - Whoosh Index: Schema fields, index update mechanism
  - Code Path Summary: Component chain with Mermaid diagrams
  - Cleanup: Temporary artifact removal

Diagrams:
  - Mermaid flowchart: Full ingestion pipeline from file drop to persistence
  - Mermaid sequence diagram: Component chain (consumer → async_task → tasks → Consumer → signals)
  - Mermaid ER diagram: Document model relationships

Key Citations:
  - src/documents/management/commands/document_consumer.py:84-91 (task enqueueing)
  - src/documents/consumer.py:180-377 (try_consume_file pipeline)
  - src/documents/apps.py:11-27 (signal handler registration)
  - src/documents/signals/handlers.py:30-431 (handler implementations)
  - src/documents/models.py:88-283 (Document model)
  - src/paperless/settings.py:449-457 (Q_CLUSTER configuration)
  - src/documents/index.py:31-49 (Whoosh schema)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new file is a standalone Markdown document placed in the `blitzy/documentation` directory per the implementation rule. It does not integrate with the existing Sphinx documentation system at `docs/`.

### 0.5.4 Cross-Documentation Dependencies

- **No cross-doc dependencies** — The output document is self-contained and does not require changes to any other documentation file.
- **Source code citations** — The document will reference specific files and line numbers from the codebase but does not modify them.
- **No navigation or TOC updates** — The `blitzy/documentation` directory is separate from the `docs/` Sphinx tree.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to the documentation exercise — both for running the Paperless-NGX environment (required to observe the pipeline) and for understanding the systems being documented. Versions are taken from `requirements.txt` (pinned) and `Pipfile` (range-specified).

**Runtime Dependencies (needed to execute the pipeline being documented):**

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | 4.0.4 | Core web framework; ORM, signals, management commands |
| pip | django-q | 1.3.9 | Task queue framework; `async_task()`, `qcluster` worker, Redis-brokered task dispatch |
| pip | redis | 3.5.3 | Python Redis client; broker backend for Django-Q and Channels |
| pip | channels | 3.0.4 | Django Channels; WebSocket status broadcasting via `status_updates` group |
| pip | channels-redis | 3.4.0 | Redis channel layer backend for Django Channels |
| pip | watchdog | 2.1.7 | Filesystem event monitoring; `PollingObserver` for file detection in consumption directory |
| pip | inotifyrecursive | 0.3.5 | Linux inotify-based file watching (preferred over watchdog polling) |
| pip | python-magic | 0.4.25 | MIME type detection via libmagic; drives parser dispatch |
| pip | whoosh | 2.7.4 | Full-text search index; stores and queries document content/metadata |
| pip | scikit-learn | 1.0.2 | ML classification; `MLPClassifier` for correspondent/type/tag prediction |
| pip | ocrmypdf | 13.4.3 | OCR wrapper for Tesseract; produces PDF/A archive copies |
| pip | pikepdf | 5.1.1 | PDF manipulation; metadata extraction, barcode-based page splitting |
| pip | pillow | 9.1.0 | Image processing; thumbnail generation |
| pip | djangorestframework | 3.13.1 | REST API framework; `PostDocumentView` for API-based uploads |
| pip | filelock | 3.6.0 | File-based locking; protects concurrent filesystem mutations on `MEDIA_LOCK` |
| pip | fuzzywuzzy | 0.18.0 | Fuzzy string matching; `MATCH_FUZZY` algorithm for classification rules |
| pip | dateparser | 1.1.1 | Date extraction from document text and filenames |
| pip | gunicorn | 20.1.0 | ASGI/WSGI server; serves web application and WebSocket endpoints |
| pip | uvicorn | 0.17.6 | ASGI server worker; used within Gunicorn for async request handling |
| pip | psycopg2 | 2.9.3 | PostgreSQL adapter (when `PAPERLESS_DBHOST` is set) |
| pip | pyzbar | 0.1.9 | Barcode detection; reads separator barcodes from PDF pages |
| pip | pdf2image | 1.16.0 | PDF-to-image conversion for barcode scanning |
| pip | tika | 1.24 | Apache Tika client; office document text extraction (feature-flagged) |
| pip | concurrent-log-handler | 0.9.20 | Concurrent-safe rotating log file handler |

**Development/Documentation Dependencies:**

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | ~4.5.0 | Documentation site generator (existing `docs/` build) |
| pip | sphinx_rtd_theme | latest | Read the Docs theme for Sphinx |
| pip | pytest | latest | Test runner (dev dependency) |
| pip | pytest-django | latest | Django test integration |

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are applicable. The new document is a standalone Markdown file in `blitzy/documentation/` and does not contain cross-links to the existing `docs/` Sphinx tree. All internal references within the new document will use relative source-code path citations (e.g., `src/documents/consumer.py:215`).

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the user's question areas:**

| Topic Area | Currently Documented? | Where? | Gap |
|------------|-----------------------|--------|-----|
| File detection mechanism (watchdog/inotify) | Partial — user-facing config only | `docs/configuration.rst` (`PAPERLESS_CONSUMER_POLLING`) | No internals: log messages, code path, stabilization wait |
| Task queuing via Django-Q | Not documented | — | No documentation of `async_task()` call, Redis payload, `Q_CLUSTER` internals |
| Consumer pipeline stages | High-level overview only | `docs/usage_overview.rst` | No stage-by-stage walkthrough with log messages |
| Parser dispatch mechanism | Not documented | — | `document_consumer_declaration` signal and weight-based selection undocumented |
| Classification (ML + rules) | Partial — user-facing | `docs/advanced_usage.rst` (matching algorithms) | No explanation of internal classifier loading, prediction, and signal handler invocation |
| Post-consumption signal handlers | Not documented | — | Six handlers registered in `apps.py` are not enumerated or explained |
| Database schema (Document model) | Partial — API field descriptions | `docs/api.rst` | No walkthrough of `documents_document` table fields as populated by pipeline |
| Django-Q task history tables | Not documented | — | `django_q_task`, `django_q_ormq` tables and their fields undocumented |
| Redis broker payload structure | Not documented | — | No documentation of serialized task format in Redis |
| Whoosh search index schema | Not documented | — | 16-field schema in `index.py` not documented externally |
| WebSocket progress messages | Partial — API docs | `docs/api.rst` | Payload structure and status codes not detailed |

**Target coverage:** 100% of the user's stated question areas, derived from observed runtime behavior and code analysis.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every question posed by the user must have a direct, evidence-based answer in the output document
- All pipeline stages must include: the responsible source file, the function/method involved, expected log messages, and data transformations performed
- Database inspection must include actual table names, column names, and example values
- Redis inspection must include key patterns and a decoded payload example
- Code-path trace must include a complete component chain from file detection to index update

**Accuracy validation:**

- All source citations must reference the correct file and line numbers from the current codebase (Paperless-NGX v1.7.0)
- Log message strings must match the actual format strings in the source code (e.g., `f"Consuming {self.filename}"` from `consumer.py:215`)
- Database column names must match the Django ORM field definitions in `models.py`
- Redis key patterns must reflect the actual Django-Q v1.3.9 implementation

**Clarity standards:**

- Technical accuracy with accessible language — explain *why* each component exists, not just *what* it does
- Progressive disclosure: start with observable behavior (logs, database records), then trace into code
- Consistent terminology: "consumption" (not "ingestion" interchangeably), "task" (Django-Q term), "consumer" (the `Consumer` class), "document_consumer" (the management command)

**Maintainability:**

- Source citations for every technical claim enable future verification against code changes
- Mermaid diagrams are text-based and version-controllable

### 0.7.3 Example and Diagram Requirements

- **Minimum examples:** At least one concrete log-output example for each pipeline stage (file detection, parsing, classification, persistence)
- **Diagram types required:** Mermaid flowchart (pipeline), Mermaid sequence diagram (component chain), Mermaid ER diagram (database)
- **Code example testing:** All example commands (Redis queries, SQLite queries) must be verified to work against the running environment
- **Database query examples:** At least one `SELECT` query per table examined (`documents_document`, `documents_log`, `django_q_task`)

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — The sole deliverable document

**Source modules to analyze and document (read-only, no modifications):**
- `src/documents/management/commands/document_consumer.py` — File detection and task enqueueing
- `src/documents/tasks.py` — `consume_file()` entry point, barcode splitting, index tasks
- `src/documents/consumer.py` — `Consumer` class: full pipeline orchestration
- `src/documents/signals/__init__.py` — Domain signal definitions
- `src/documents/signals/handlers.py` — Six post-consumption handlers, file cleanup, rename logic
- `src/documents/apps.py` — Signal handler registration
- `src/documents/models.py` — `Document`, `Log`, `Correspondent`, `Tag`, `DocumentType`, `FileInfo`
- `src/documents/index.py` — Whoosh search index schema, write, query operations
- `src/documents/classifier.py` — `DocumentClassifier`, `load_classifier()`
- `src/documents/matching.py` — Rule-based matching functions
- `src/documents/parsers.py` — Parser discovery, MIME detection, date extraction
- `src/documents/file_handling.py` — Filename generation, directory creation
- `src/documents/loggers.py` — `LoggingMixin` for correlated log groups
- `src/documents/views.py` — `PostDocumentView` (API upload path)
- `src/paperless_tesseract/signals.py` — Tesseract parser declaration
- `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser`
- `src/paperless_text/signals.py` — Text parser declaration
- `src/paperless_tika/signals.py` — Tika parser declaration
- `src/paperless/settings.py` — `Q_CLUSTER`, directory paths, consumer settings, logging config
- `src/paperless/version.py` — Version metadata (1.7.0)
- `src/paperless_mail/mail.py` — Email ingestion path (for completeness in entry-point documentation)
- `docker/supervisord.conf` — Process supervision configuration
- `docker/docker-prepare.sh` — Startup sequence
- `docker/docker-entrypoint.sh` — Container entrypoint
- `Pipfile`, `requirements.txt` — Dependency manifests

**Runtime observations to capture:**
- Log messages from all `paperless.*` logging namespaces during a test file consumption
- Redis key inspection while a task is queued
- SQLite database queries after consumption completes
- Django-Q task model inspection

**Temporary artifacts (to be created and cleaned up):**
- Small `.txt` test file dropped into `CONSUMPTION_DIR`
- Any helper scripts used for Redis or database inspection

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly states "don't modify the actual source code." No changes to any `.py`, `.js`, `.html`, `.rst`, or configuration files in the repository.
- **Existing documentation updates** — The `docs/` Sphinx tree (`*.rst` files, `conf.py`, `Makefile`) must not be modified.
- **Frontend (Angular) analysis** — The `src-ui/` directory is not relevant to the backend pipeline being documented.
- **Feature additions or refactoring** — No functional changes to the codebase.
- **Deployment configuration changes** — Docker Compose files, Dockerfile, and supervisor config are for reading/reference only.
- **Test file modifications** — The `src/documents/tests/` directory must not be altered.
- **Email ingestion deep-dive** — While the email entry point will be mentioned for completeness, the user's focus is on the directory-watcher path; detailed IMAP processing is out of scope.
- **Tika/Gotenberg integration** — These are feature-flagged optional services. The user's test will use a plain `.txt` file, so Tika parsing is not exercised.
- **Multi-database PostgreSQL setup** — The documentation will focus on the default SQLite backend since no `PAPERLESS_DBHOST` is specified.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

**Environment setup commands (to run the system being documented):**

- Install Python dependencies: `pip install -r requirements.txt`
- Apply database migrations: `cd src && python manage.py migrate`
- Create superuser (if needed): `PAPERLESS_ADMIN_USER=admin PAPERLESS_ADMIN_PASSWORD=admin python manage.py manage_superuser`
- Start Redis: `redis-server --daemonize yes` or use existing Redis at `redis://localhost:6379`
- Start Django-Q cluster: `cd src && python manage.py qcluster &`
- Start document consumer: `cd src && python manage.py document_consumer &`
- Start web server (optional for observation): `cd src && gunicorn -c ../gunicorn.conf.py paperless.asgi:application &`

**Test file creation:**

- Create consumption directory: `mkdir -p ../consume`
- Drop test file: `echo "Test document for pipeline tracing" > ../consume/test_pipeline.txt`

**Observation commands:**

- Monitor logs: `tail -f ../data/log/paperless.log`
- Inspect Redis broker: `redis-cli keys "django_q:*"` and `redis-cli lrange django_q:q 0 -1`
- Query database after processing: `sqlite3 ../data/db.sqlite3 "SELECT id, title, mime_type, checksum, created, added, filename FROM documents_document ORDER BY id DESC LIMIT 1;"`
- Check processing log: `sqlite3 ../data/db.sqlite3 "SELECT * FROM documents_log ORDER BY created DESC LIMIT 10;"`
- Check Django-Q task history: `sqlite3 ../data/db.sqlite3 "SELECT id, name, func, started, stopped, success FROM django_q_task ORDER BY id DESC LIMIT 5;"`

**Cleanup commands:**

- Remove test document via Django shell or API
- Delete any temporary helper scripts

**Documentation build command:** N/A — the output is a standalone Markdown file, not a Sphinx build artifact

**Default format:** Markdown with Mermaid diagrams

**Citation requirement:** Every section must reference specific source files and line numbers

**Style guide:** Follow the implementation rule: provide thinking/rationale behind answers, base conclusions on code as truth, do not make assumptions

**Documentation validation:** Manual review against the user's original questions to ensure every question is answered with evidence

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and the implementation rule set, and must be strictly adhered to:

- **Do not modify any existing files in the source repository** — This is both the user's instruction ("don't modify the actual source code") and the SWE-AtlasQnA-Repo rule. All analysis is read-only; only the new `blitzy/documentation/paperless-ngx_542221a38dff.md` file is created.
- **Create the output as `<source_branch_name>.md`** — The branch name is `paperless-ngx_542221a38dff`, so the output file must be `paperless-ngx_542221a38dff.md` in the `blitzy/documentation` directory.
- **Provide thinking and rationale behind answers** — The document must not just state facts but explain *why* the system behaves as observed, tracing behavior to specific code constructs.
- **Base all answers on the code as truth** — Do not make assumptions about behavior. Every claim must be traceable to a specific file and line in the codebase.
- **Clean up temporary artifacts** — Any test files, helper scripts, or temporary data created during the investigation must be removed after the exercise is complete.
- **Temporary helper scripts are permitted** — The user explicitly allows "temporary helper scripts or test files if needed" for observation purposes.
- **Follow the hands-on observational approach** — The documentation should reflect what is actually observed when running the system, not just theoretical code reading. Log output, database queries, and Redis inspection should be captured as evidence.
- **Document the code path from observed behavior** — "Trace the high-level codepath behind this workflow based on the behavior you observe" — the code-path section should be grounded in what the logs and system state reveal, then traced into the source code for explanation.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and directories were comprehensively examined to derive the conclusions in this Agent Action Plan:

**Root-Level Files:**
- `Pipfile` — Python dependency specification with version ranges
- `requirements.txt` — Pinned Python dependency manifest (113 packages)
- `Dockerfile` — Production container image recipe
- `gunicorn.conf.py` — ASGI web server configuration
- `paperless.conf.example` — Example runtime configuration variables
- `.readthedocs.yml` — Documentation hosting configuration
- `README.md` — Project overview and feature summary
- `CONTRIBUTING.md` — Contributor guidelines and team structure

**Backend Source (`src/`):**
- `src/manage.py` — Django management entrypoint
- `src/setup.cfg` — Test, linting, and coverage configuration
- `src/documents/consumer.py` — Ingestion pipeline orchestrator (433 lines)
- `src/documents/tasks.py` — Background task definitions (281 lines)
- `src/documents/models.py` — ORM models: Document, Log, Correspondent, Tag, DocumentType, SavedView, FileInfo (467 lines)
- `src/documents/signals/__init__.py` — Domain signal definitions (6 lines)
- `src/documents/signals/handlers.py` — Signal handler implementations (431 lines)
- `src/documents/apps.py` — Signal registration in `DocumentsConfig.ready()` (29 lines)
- `src/documents/index.py` — Whoosh search index operations (288 lines)
- `src/documents/classifier.py` — ML document classifier (293 lines)
- `src/documents/matching.py` — Rule-based matching functions (inspected first 50 lines)
- `src/documents/parsers.py` — Parser discovery and MIME detection (first 80 lines)
- `src/documents/file_handling.py` — Filename generation and directory management (first 50 lines)
- `src/documents/loggers.py` — LoggingMixin for correlated log groups (22 lines)
- `src/documents/views.py` — REST API views including PostDocumentView (lines 491–535)
- `src/documents/management/commands/document_consumer.py` — File watcher management command (241 lines)
- `src/paperless/settings.py` — Django settings: Q_CLUSTER, paths, logging, consumer config (616 lines)
- `src/paperless/version.py` — Version tuple: (1, 7, 0)
- `src/paperless_tesseract/signals.py` — Tesseract parser declaration (19 lines)
- `src/paperless_tesseract/parsers.py` — RasterisedDocumentParser (first 60 lines)

**Docker Infrastructure (`docker/`):**
- `docker/supervisord.conf` — Process supervision: gunicorn, consumer, scheduler
- `docker/docker-prepare.sh` — Startup sequence: DB/Redis readiness, migrations, index rebuild
- `docker/docker-entrypoint.sh` — Container entrypoint (explored via summary)
- `docker/compose/` — Eight Docker Compose topology files (explored via summary)

**Documentation (`docs/`):**
- `docs/conf.py` — Sphinx configuration (332 lines)
- `docs/index.rst` — Documentation toctree (11 content pages)
- `docs/Makefile` — Sphinx build targets (explored via summary)

**Tech Spec Sections Retrieved:**
- Section 1.1 — Executive Summary (project overview, stakeholders, version)
- Section 4.1 — High-Level System Workflow (data flow diagrams, startup sequence, process supervision)
- Section 5.1 — High-Level Architecture (component table, data flow, integration points)
- Section 6.2 — Database Design (schema, ER diagram, data management, indexing, concurrency)

### 0.11.2 Attachments

No attachments were provided by the user.

### 0.11.3 Figma Screens

No Figma URLs or design screens were provided.

### 0.11.4 External URLs

No external URLs were referenced by the user. The Paperless-NGX project maintains its documentation at ReadTheDocs and its source at GitHub, but neither was explicitly cited as a required input for this task.

