# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a comprehensive, runtime-observation-grounded explanation of how data moves through the Paperless-NGX document ingestion pipeline at commit `542221a38dff`.

**Category:** Create new documentation
**Documentation Type:** Technical walkthrough / Architecture runtime-behavior guide

The user's requirements translate to the following documentation deliverables:

- A detailed narrative explaining the observable runtime behavior of the document ingestion pipeline — from the moment a file is detected to its final indexed and persisted state
- Identification and explanation of specific log messages, task names, progress status codes, and state transitions that are emitted at each stage of processing
- Documentation of where document data ends up after processing (database tables, filesystem locations, search index)
- An explanation of the duplicate-detection mechanism and how the system avoids re-processing already-consumed documents
- All claims grounded in actual code analysis at the specified commit, not in assumptions

**Specific User Requirements:**

- When a new document appears in the system, what observable runtime behavior shows how the document is detected and handed off for processing?
- What log messages, task names, or state changes indicate the transition from initial detection into parsing, classification, and indexing?
- How does the system reflect progress or completion of each stage while the document is being processed?
- After processing finishes, what observable evidence shows where the document's data ends up and how its final state is recorded?
- How does Paperless-NGX track whether a document has already been processed or needs further work?
- What behavior or logs indicate how the system avoids duplicate processing?
- Explanation must be grounded in runtime observations rather than assumptions from reading the code alone

### 0.1.2 Special Instructions and Constraints

- **No source file modifications**: The user explicitly states "don't modify any source files." Only temporary test files may be created for exploration purposes, and these must be cleaned up afterward.
- **Runtime-observation focus**: The documentation must be grounded in observable behavior — log output, task names, WebSocket messages, database records, filesystem artifacts — rather than speculative descriptions from reading code alone.
- **Commit-pinned**: All analysis must be performed against the repository at commit `542221a38dff` (Paperless-NGX version 1.7.0).
- **Output format**: Per project rules, the deliverable is a markdown file named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation` directory.
- **Thinking / Rationale requirement**: The document must provide rationale behind all answers, citing the code as the source of truth.
- **No assumptions**: Answers must be evidence-based from the codebase.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document file detection behavior, we will trace the `document_consumer` management command (`src/documents/management/commands/document_consumer.py`) which uses either inotify or watchdog polling to detect new files in `CONSUMPTION_DIR`, and the REST API upload endpoint `PostDocumentView` in `src/documents/views.py`, and the email ingestion path in `src/paperless_mail/mail.py`
- To document the handoff to processing, we will trace how detected files are enqueued as Django-Q async tasks via `async_task("documents.tasks.consume_file", ...)` with logged messages like `"Adding {filepath} to the task queue."`
- To document parsing and OCR, we will trace the `Consumer.try_consume_file()` method in `src/documents/consumer.py` and the parser dispatch through `src/documents/parsers.py` to the Tesseract/Text/Tika parsers
- To document classification, we will trace the `document_consumption_finished` signal handlers registered in `src/documents/apps.py` that trigger `set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`, `set_log_entry`, and `add_to_index`
- To document progress/status, we will trace the `_send_progress()` calls that emit WebSocket messages on the `status_updates` channel group with `STARTING`, `WORKING`, `SUCCESS`, and `FAILED` statuses
- To document final state, we will trace the `Document` model fields in `src/documents/models.py`, the filesystem paths under `ORIGINALS_DIR`, `ARCHIVE_DIR`, `THUMBNAIL_DIR`, and the Whoosh search index in `INDEX_DIR`
- To document duplicate prevention, we will trace the MD5 checksum comparison in `Consumer.pre_check_duplicate()` which queries existing `Document.checksum` and `Document.archive_checksum` fields

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Entry point inventory**: The system has three distinct document entry points (directory watcher, REST API upload, email ingestion) that all converge on the same `documents.tasks.consume_file` task — this convergence pattern requires consolidated documentation
- **Signal-driven post-processing chain**: The `document_consumption_finished` signal fires six connected handlers in `src/documents/apps.py` — the order and behavior of these handlers requires explicit documentation
- **Django-Q task lifecycle**: Documents are processed by Django-Q workers managed via `qcluster`, and the Q_CLUSTER settings in `src/paperless/settings.py` govern retry/timeout behavior that affects observable runtime behavior
- **Barcode splitting path**: When `CONSUMER_ENABLE_BARCODES` is active, documents may be split before consumption — this alternate path produces different observable behavior
- **Classifier model lifecycle**: The ML classifier (`src/documents/classifier.py`) is loaded during consumption and may or may not exist, producing different log output depending on whether a trained model is available

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with comprehensive narrative guides, hosted on ReadTheDocs.

- **Documentation framework**: Sphinx ~4.5.0 (from `Pipfile` dev-packages)
- **Theme**: `sphinx_rtd_theme` (Read the Docs theme, configured in `docs/conf.py`)
- **Configuration file**: `docs/conf.py` — sets project metadata, enables extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), and wires in static assets
- **Build system**: `docs/Makefile` provides standard Sphinx targets (HTML, EPUB, PDF, linkcheck, doctest)
- **Container preview**: `docs/Dockerfile` builds and serves docs via Python's HTTP server on port 8000
- **ReadTheDocs configuration**: `.readthedocs.yml` pins Python 3.8 for doc builds and references `docs/requirements.txt` (currently empty)
- **Documentation dependencies manifest**: `docs/requirements.txt` is present but empty — Sphinx and theme are pulled from `Pipfile` dev-packages
- **Diagram tools detected**: No Mermaid or PlantUML tooling present in the existing docs infrastructure; diagrams will be created in Mermaid-compatible markdown for the new deliverable
- **API documentation tools**: No dedicated API doc generators (JSDoc, Swagger) detected; the `docs/api.rst` file is manually maintained

**Existing documentation pages** (all `.rst` files in `docs/`):

| File | Purpose |
|------|---------|
| `docs/index.rst` | Landing page and navigation hub with toctree |
| `docs/setup.rst` | Installation, migration, reverse proxy, deployment modes |
| `docs/configuration.rst` | All environment variables and runtime settings |
| `docs/usage_overview.rst` | Product model, ingestion methods, search, recommended workflows |
| `docs/advanced_usage.rst` | Advanced matching, hooks, filename handling |
| `docs/administration.rst` | Backups, updates, utilities, indexing, archiving, encryption |
| `docs/api.rst` | REST endpoints, authentication, uploads, search, versioning |
| `docs/troubleshooting.rst` | Common operational failures and fixes |
| `docs/extending.rst` | Contributor workflows, dev setup, localization, parser extension |
| `docs/faq.rst` | Common support and deployment questions |
| `docs/scanners.rst` | Compatible scanners and mobile scanning apps |
| `docs/screenshots.rst` | Visual gallery of the UI and key workflows |
| `docs/changelog.rst` | Release history across Paperless, Paperless-ng, and Paperless-ngx |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code related to the document ingestion pipeline:

- **Consumer pipeline**: `src/documents/consumer.py` — the central `Consumer` class with `try_consume_file()` method orchestrating the entire ingestion flow
- **Task queue**: `src/documents/tasks.py` — `consume_file()` function wrapping Consumer invocation plus barcode splitting logic
- **File detection**: `src/documents/management/commands/document_consumer.py` — filesystem watcher using `watchdog.observers.polling.PollingObserver` or `inotifyrecursive.INotify`
- **REST API upload**: `src/documents/views.py` — `PostDocumentView` handling `POST /api/documents/post_document/`
- **Email ingestion**: `src/paperless_mail/mail.py` and `src/paperless_mail/tasks.py` — IMAP fetch and attachment extraction
- **Signal handlers**: `src/documents/signals/__init__.py` (signal definitions) and `src/documents/signals/handlers.py` (receiver implementations for classification, indexing, logging, file management)
- **App wiring**: `src/documents/apps.py` — `DocumentsConfig.ready()` connecting six handlers to `document_consumption_finished`
- **Parser registry**: `src/documents/parsers.py` — parser discovery via `document_consumer_declaration` signal
- **Parser implementations**: `src/paperless_tesseract/parsers.py`, `src/paperless_text/parsers.py`, `src/paperless_tika/parsers.py`
- **Parser signal registrations**: `src/paperless_tesseract/signals.py`, `src/paperless_text/signals.py`, `src/paperless_tika/signals.py` and their respective `apps.py` files
- **Models**: `src/documents/models.py` — `Document`, `Tag`, `Correspondent`, `DocumentType`, `Log`, `FileInfo`
- **Classification**: `src/documents/classifier.py` — `DocumentClassifier` with scikit-learn MLP and `src/documents/matching.py` — rule-based matching
- **Search index**: `src/documents/index.py` — Whoosh-backed full-text search with schema definition
- **File handling**: `src/documents/file_handling.py` — filename generation and directory management
- **WebSocket**: `src/paperless/consumers.py` — `StatusConsumer` broadcasting progress via Channels
- **Settings**: `src/paperless/settings.py` — all consumer, OCR, classifier, and index configuration
- **Sanity checker**: `src/documents/sanity_checker.py` — audit tool comparing database records to filesystem
- **Logging mixin**: `src/documents/loggers.py` — `LoggingMixin` with UUID-based group correlation
- **Supervisor config**: `docker/supervisord.conf` — three supervised processes: `gunicorn`, `consumer`, `scheduler`
- **Docker preparation**: `docker/docker-prepare.sh` — migrations, search index rebuild, superuser creation

### 0.2.3 Web Search Research Conducted

No web search was necessary for this task. The documentation deliverable is a technical analysis document grounded entirely in codebase evidence at a specific commit. The Paperless-NGX codebase at commit `542221a38dff` (version 1.7.0) is the sole source of truth, as mandated by the user's instructions.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation as part of the ingestion-pipeline runtime-behavior analysis:

**Stage 1 — File Detection and Queueing:**

- Module: `src/documents/management/commands/document_consumer.py`
  - Key functions: `_consume()`, `_consume_wait_unmodified()`, `_is_ignored()`, `_tags_from_path()`, `Handler.on_created()`, `Handler.on_moved()`, `Command.handle()`, `Command.handle_polling()`, `Command.handle_inotify()`
  - Observable behavior: inotify/polling log messages, file-stability polling, async task enqueue logs
  - Logger: `paperless.management.consumer`

- Module: `src/documents/views.py` (`PostDocumentView`)
  - Key functions: `PostDocumentView.post()`
  - Observable behavior: temp file creation in `SCRATCH_DIR`, `async_task()` enqueue with `task_name`

- Module: `src/paperless_mail/mail.py` (`MailAccountHandler`)
  - Key functions: `MailAccountHandler.handle_mail_account()`, attachment extraction
  - Observable behavior: per-attachment log messages, `async_task()` enqueue
  - Logger: inherits from `documents.loggers.LoggingMixin`

**Stage 2 — Task Execution and Consumer Entry:**

- Module: `src/documents/tasks.py`
  - Key functions: `consume_file()`, barcode scanning and splitting logic
  - Observable behavior: barcode detection logs, file splitting logs, `Consumer().try_consume_file()` invocation
  - Logger: `paperless.tasks`

**Stage 3 — Consumer Pipeline Execution:**

- Module: `src/documents/consumer.py`
  - Key class: `Consumer` (extends `LoggingMixin`)
  - Key functions: `try_consume_file()`, `pre_check_file_exists()`, `pre_check_duplicate()`, `pre_check_directories()`, `run_pre_consume_script()`, `_store()`, `apply_overrides()`, `_write()`, `run_post_consume_script()`
  - Observable behavior: progress WebSocket messages (`STARTING`, `WORKING`, `SUCCESS`, `FAILED`), message constants (`MESSAGE_NEW_FILE`, `MESSAGE_PARSING_DOCUMENT`, `MESSAGE_GENERATING_THUMBNAIL`, `MESSAGE_PARSE_DATE`, `MESSAGE_SAVE_DOCUMENT`, `MESSAGE_FINISHED`), info/debug log messages
  - Logger: `paperless.consumer`

**Stage 4 — Parser Dispatch and Execution:**

- Module: `src/documents/parsers.py`
  - Key functions: `get_parser_class_for_mime_type()`, `parse_date()`, `DocumentParser` base class
  - Observable behavior: parser selection logs, `document_consumer_declaration` signal dispatch

- Module: `src/paperless_tesseract/parsers.py`
  - Key class: `RasterisedDocumentParser`
  - Observable behavior: OCRmyPDF invocation logs, sidecar text extraction, DPI detection
  - Logger: `paperless.parsing.tesseract`

- Module: `src/paperless_text/parsers.py`
  - Key class: `TextDocumentParser`
  - Observable behavior: plain text file reading

- Module: `src/paperless_tika/parsers.py`
  - Key class: `TikaDocumentParser`
  - Observable behavior: Tika/Gotenberg HTTP calls (gated by `PAPERLESS_TIKA_ENABLED`)

**Stage 5 — Post-Consumption Signal Chain:**

- Module: `src/documents/signals/handlers.py`
  - Key functions: `add_inbox_tags()`, `set_correspondent()`, `set_document_type()`, `set_tags()`, `set_log_entry()`, `add_to_index()`, `update_filename_and_move_files()`, `cleanup_document_deletion()`
  - Observable behavior: "Assigning correspondent X to Y", "Assigning document type X to Y", "Tagging X with Y" log messages, Django admin LogEntry creation, Whoosh index updates
  - Logger: `paperless.handlers`

- Module: `src/documents/matching.py`
  - Key functions: `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`
  - Observable behavior: debug-level match reason logs
  - Logger: `paperless.matching`

- Module: `src/documents/classifier.py`
  - Key class: `DocumentClassifier`
  - Key functions: `load_classifier()`, `predict_correspondent()`, `predict_document_type()`, `predict_tags()`
  - Observable behavior: "Document classification model does not exist (yet)" or successful load
  - Logger: `paperless.classifier`

**Stage 6 — Indexing and Notification:**

- Module: `src/documents/index.py`
  - Key functions: `add_or_update_document()`, `update_document()`, `open_index()`
  - Observable behavior: Whoosh index write operations
  - Logger: `paperless.index`

- Module: `src/paperless/consumers.py`
  - Key class: `StatusConsumer` (WebSocket)
  - Observable behavior: JSON payloads sent to `ws/status/` with `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, `document_id`

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the existing Paperless-NGX documentation covers the ingestion pipeline at a high level in `docs/usage_overview.rst` (describing the three operations: OCR, archive PDF creation, and automatic matching), but there is **no existing documentation** that:

- Maps specific log messages to pipeline stages
- Documents the WebSocket progress protocol (`status_updates` channel group payloads)
- Explains the Django-Q task lifecycle for document consumption
- Details the exact sequence of signal handler invocations after `document_consumption_finished` fires
- Describes the MD5-based duplicate detection mechanism with its dual-checksum query logic
- Traces the complete data flow from file detection through to final indexed state in a runtime-observation-grounded manner

This analysis deliverable will address all of these gaps as a standalone markdown document.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive markdown file named `paperless-ngx_542221a38dff.md` to be placed in the `blitzy/documentation` directory. The document structure follows the user's question sequence:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── Introduction & Context
        │   ├── Commit and Version Reference
        │   └── Three Entry Points Overview
        ├── Stage 1: Document Detection
        │   ├── Directory Watcher (inotify/polling)
        │   ├── REST API Upload
        │   └── Email Ingestion
        ├── Stage 2: Task Queueing & Handoff
        │   ├── Django-Q async_task Dispatch
        │   └── Barcode Splitting Pre-check
        ├── Stage 3: Consumer Pipeline — Pre-checks
        │   ├── File Existence Verification
        │   ├── Duplicate Detection (MD5 Checksum)
        │   └── Directory Preparation
        ├── Stage 4: Parsing & Text Extraction
        │   ├── MIME Type Detection
        │   ├── Parser Selection
        │   └── OCR / Text / Tika Processing
        ├── Stage 5: Metadata & Date Extraction
        ├── Stage 6: Classification (ML + Rules)
        │   ├── Correspondent Assignment
        │   ├── Document Type Assignment
        │   ├── Tag Assignment
        │   └── Inbox Tag Application
        ├── Stage 7: Atomic Persistence
        │   ├── Database Record Creation
        │   ├── File Copy to Managed Storage
        │   └── Archive PDF Storage
        ├── Stage 8: Post-Consumption Hooks
        │   ├── Signal Handler Chain
        │   ├── Search Index Update
        │   ├── Admin Log Entry
        │   └── Filename Generation & File Move
        ├── Stage 9: Progress & Completion Reporting
        │   ├── WebSocket Status Protocol
        │   └── Post-Consume Script Execution
        ├── Duplicate Prevention Mechanism
        ├── Final State Summary
        │   ├── Database Fields
        │   ├── Filesystem Layout
        │   └── Search Index Schema
        └── Pipeline Diagram (Mermaid)
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the exact sequence of method calls, log statements, and WebSocket payloads from `src/documents/consumer.py:try_consume_file()` to document the runtime behavior chain
- Extract all `MESSAGE_*` constants from `src/documents/consumer.py` (lines 37–49) to catalog observable status messages
- Extract signal handler registration order from `src/documents/apps.py:DocumentsConfig.ready()` to document the post-consumption processing chain
- Extract the `_send_progress()` payload structure from `src/documents/consumer.py` (lines 56–76) to document the WebSocket protocol
- Extract the `Document` model field definitions from `src/documents/models.py` to document final data locations
- Extract the Whoosh index schema from `src/documents/index.py:get_schema()` to document the search index structure
- Extract the duplicate-check query from `src/documents/consumer.py:pre_check_duplicate()` to document the dedup mechanism
- Extract the parser registration pattern from `src/paperless_tesseract/signals.py`, `src/paperless_text/signals.py`, and `src/paperless_tika/signals.py`

**Documentation Standards:**

- Markdown formatting with `#` / `##` / `###` heading hierarchy
- Mermaid diagrams for the pipeline flow and signal handler chain
- Code-path citations in the format `Source: src/path/to/file.py:LineNumber`
- Log message examples formatted as code blocks
- Tables for configuration options, status codes, and model fields
- Consistent terminology aligned with the codebase (e.g., "consumer" not "ingestor", "correspondent" not "sender")

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams:

- **End-to-end pipeline flowchart**: Showing the ten stages from file detection through WebSocket notification, with entry points for directory watcher, REST API, and email
- **Signal handler chain sequence diagram**: Showing the order in which `document_consumption_finished` handlers fire (add_inbox_tags → set_correspondent → set_document_type → set_tags → set_log_entry → add_to_index)
- **WebSocket progress state diagram**: Showing the status transitions from `STARTING` → `WORKING` → `SUCCESS` or `FAILED`
- **File storage layout diagram**: Showing where originals, archives, and thumbnails end up on disk

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/management/commands/document_consumer.py`, `src/documents/signals/handlers.py`, `src/documents/signals/__init__.py`, `src/documents/apps.py`, `src/documents/models.py`, `src/documents/parsers.py`, `src/documents/index.py`, `src/documents/classifier.py`, `src/documents/matching.py`, `src/documents/loggers.py`, `src/documents/file_handling.py`, `src/documents/sanity_checker.py`, `src/documents/views.py`, `src/paperless/consumers.py`, `src/paperless/settings.py`, `src/paperless_tesseract/parsers.py`, `src/paperless_tesseract/signals.py`, `src/paperless_tesseract/apps.py`, `src/paperless_text/parsers.py`, `src/paperless_text/signals.py`, `src/paperless_text/apps.py`, `src/paperless_tika/parsers.py`, `src/paperless_tika/signals.py`, `src/paperless_tika/apps.py`, `src/paperless_mail/mail.py`, `src/paperless_mail/tasks.py`, `docker/supervisord.conf`, `docker/docker-prepare.sh`, `docker/docker-entrypoint.sh` | Comprehensive runtime-behavior analysis of the document ingestion pipeline covering file detection, task queueing, parsing, classification, persistence, indexing, WebSocket notifications, and duplicate prevention, with Mermaid diagrams, log-message catalogs, and data-destination mapping |

This task produces exactly **one** new documentation file. No existing documentation files are updated, deleted, or used as reference templates — the deliverable is a standalone markdown analysis document as required by the project rules.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Runtime Behavior Analysis
Source Code: 30+ source files across src/documents/, src/paperless/, 
    src/paperless_tesseract/, src/paperless_text/, src/paperless_tika/,
    src/paperless_mail/, and docker/
Sections:
    - Introduction (commit, version, repository context)
    - Document Detection (three entry points: directory watcher, REST API, email)
    - Task Queueing and Handoff (Django-Q async_task dispatch)
    - Consumer Pre-checks (file existence, duplicate MD5 check, directories)
    - Parsing and Text Extraction (MIME detection, parser dispatch, OCR/text/Tika)
    - Metadata and Date Extraction (filename and content date parsing)
    - Classification (ML classifier + rule-based matching for correspondent/type/tags)
    - Atomic Persistence (database record creation, file copy to managed storage)
    - Post-Consumption Signal Chain (inbox tags, classification, admin log, indexing)
    - Progress and Completion Reporting (WebSocket status_updates protocol)
    - Duplicate Prevention Mechanism (MD5 checksum dual-query)
    - Final State Summary (database fields, filesystem layout, index schema)
    - Pipeline Diagram (end-to-end Mermaid flowchart)
Diagrams:
    - End-to-end ingestion pipeline flowchart (Mermaid)
    - Signal handler execution order sequence diagram (Mermaid)
    - WebSocket progress state diagram (Mermaid)
    - Filesystem storage layout (text diagram)
Key Citations:
    - src/documents/consumer.py (central pipeline orchestrator)
    - src/documents/tasks.py (task wrapper with barcode logic)
    - src/documents/management/commands/document_consumer.py (file watcher)
    - src/documents/signals/handlers.py (post-consumption handlers)
    - src/documents/apps.py (signal connection order)
    - src/documents/models.py (Document model — final data schema)
    - src/documents/index.py (Whoosh search index schema)
    - src/documents/classifier.py (ML classifier lifecycle)
    - src/paperless/consumers.py (WebSocket StatusConsumer)
    - src/paperless/settings.py (all configuration variables)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new file is a standalone markdown document placed in the `blitzy/documentation` directory and is not integrated into the existing Sphinx documentation system (`docs/`).

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes**: The new document is self-contained
- **No navigation link updates**: Not integrated into existing docs toctree
- **No table of contents updates**: Standalone deliverable
- **Internal cross-references**: The document will reference specific source files with line numbers for traceability, but has no dependency on other documentation files

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The deliverable is a Markdown file and does not require building with any documentation tooling. However, the following packages from the repository are relevant to the document's content — they are the runtime dependencies that power the ingestion pipeline being documented:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | ~=4.0 | Web framework powering the application and ORM |
| pip | django-q | ~=1.3 | Task queue for asynchronous document consumption |
| pip | channels | ~=3.0 | WebSocket support for real-time status updates |
| pip | channels-redis | * | Redis channel layer backend for WebSocket |
| pip | redis | * | Broker for Django-Q and Channels |
| pip | ocrmypdf | ~=13.4 | OCR processing for PDF and image documents |
| pip | scikit-learn | ==1.0.2 | ML classifier for automatic tag/correspondent/type assignment |
| pip | whoosh | ~=2.7.4 | Full-text search index engine |
| pip | python-magic | * | MIME type detection via libmagic |
| pip | watchdog | ~=2.1.0 | Filesystem polling observer for consumption directory |
| pip | inotifyrecursive | ~=0.3 | Linux inotify watcher (alternative to polling) |
| pip | pillow | ~=9.1 | Image processing for thumbnails |
| pip | pikepdf | ~=5.1 | PDF manipulation for barcode splitting |
| pip | pyzbar | * | Barcode detection in scanned pages |
| pip | pdf2image | * | PDF-to-image conversion for barcode scanning |
| pip | pdfminer.six | * | Text extraction from PDF files |
| pip | dateparser | ~=1.1 | Date parsing from document text and filenames |
| pip | filelock | * | Filesystem locking for media directory operations |
| pip | fuzzywuzzy | * (with speedup) | Fuzzy string matching for document classification |
| pip | tika | * | Apache Tika client for Office document parsing |
| pip | gunicorn | * | WSGI/ASGI server for production |
| pip | uvicorn | * (with standard) | ASGI server with WebSocket support |
| pip | sphinx | ~=4.5.0 | Documentation build system (dev dependency) |
| pip | sphinx_rtd_theme | * | ReadTheDocs theme for Sphinx (dev dependency) |

### 0.6.2 Documentation Reference Updates

Not applicable — no existing documentation links need to be updated. The deliverable is a standalone new file that does not alter any existing documentation structure or cross-references.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Target coverage for the ingestion pipeline runtime-behavior analysis:**

| Pipeline Stage | Key Source Files | Coverage Target |
|----------------|-----------------|-----------------|
| File Detection (3 entry points) | `document_consumer.py`, `views.py`, `mail.py` | 100% — all three entry points documented with log messages |
| Task Queueing | `document_consumer.py:_consume()`, `views.py:PostDocumentView.post()`, `mail.py` | 100% — async_task dispatch documented for each path |
| Consumer Pre-checks | `consumer.py:pre_check_file_exists()`, `pre_check_duplicate()`, `pre_check_directories()` | 100% — all three pre-checks documented |
| MIME Detection & Parser Dispatch | `consumer.py:219-226`, `parsers.py:get_parser_class_for_mime_type()` | 100% — parser selection mechanism and all three parser types |
| Parsing & Text Extraction | `parsers.py`, `paperless_tesseract/parsers.py`, `paperless_text/parsers.py`, `paperless_tika/parsers.py` | 100% — OCR, text, and Tika paths |
| Thumbnail Generation | `consumer.py:264-269`, `parsers.py:make_thumbnail_from_pdf()` | 100% — thumbnail creation and optimization |
| Date Extraction | `parsers.py:parse_date()`, `consumer.py:273-275` | 100% — filename and content date parsing |
| Classification | `classifier.py`, `matching.py`, `signals/handlers.py` | 100% — ML prediction and rule-based matching |
| Signal Handler Chain | `apps.py:ready()`, all six connected handlers | 100% — all six handlers documented in execution order |
| Database Persistence | `consumer.py:_store()`, `models.py:Document` | 100% — all Document model fields mapped |
| Filesystem Storage | `consumer.py:315-337`, `file_handling.py` | 100% — originals, archive, thumbnail paths |
| Search Indexing | `index.py:add_or_update_document()`, `get_schema()` | 100% — full Whoosh schema documented |
| WebSocket Notifications | `consumer.py:_send_progress()`, `paperless/consumers.py` | 100% — payload structure and status codes |
| Duplicate Prevention | `consumer.py:pre_check_duplicate()` | 100% — MD5 dual-checksum logic |
| Barcode Splitting | `tasks.py:consume_file()`, `scan_file_for_separating_barcodes()` | 100% — alternate path documented |

**Overall target: 100% coverage** of the user's questions about observable runtime behavior across all pipeline stages.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every pipeline stage has descriptions of observable log messages, with logger names and log levels
- Every status transition is documented with the exact WebSocket payload fields
- All database fields on the `Document` model are listed with their data types and purposes
- All filesystem locations are specified with their settings variable names
- The duplicate-detection mechanism is explained with the exact query filter logic

**Accuracy validation:**
- Every claim in the document cites a specific source file and line number
- Log message examples match the exact format strings found in the source code
- WebSocket payload field names match the exact dictionary keys in `_send_progress()`
- Configuration variable names match their definitions in `src/paperless/settings.py`

**Clarity standards:**
- Technical accuracy with progressive disclosure (overview first, then details)
- Consistent use of codebase terminology (consumer, correspondent, document type, tag, archive serial number)
- Each stage is self-contained but connected to adjacent stages via clear transitions

**Maintainability:**
- Source file citations enable future developers to locate the relevant code
- Mermaid diagrams can be updated independently
- The document is commit-pinned, so readers know exactly which version it describes

### 0.7.3 Example and Diagram Requirements

- **Minimum log message examples per stage**: At least one representative log message per pipeline stage, formatted as code blocks
- **Diagram types**: Mermaid flowcharts (pipeline overview), Mermaid sequence diagrams (signal chain), Mermaid state diagrams (WebSocket status)
- **Code example format**: Brief inline snippets showing key function signatures and log format strings, never exceeding 2-3 lines
- **Rationale sections**: Each major answer includes a "Thinking / Rationale" explanation grounding it in code evidence

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable

**Source files analyzed for content (read-only, not modified):**
- `src/documents/consumer.py` — ingestion pipeline orchestrator
- `src/documents/tasks.py` — async task wrapper and barcode logic
- `src/documents/management/commands/document_consumer.py` — filesystem watcher
- `src/documents/signals/__init__.py` — signal definitions
- `src/documents/signals/handlers.py` — signal receivers for classification, indexing, file management
- `src/documents/apps.py` — signal handler registration order
- `src/documents/models.py` — Document, Tag, Correspondent, DocumentType, Log models
- `src/documents/parsers.py` — parser registry and base class
- `src/documents/index.py` — Whoosh search index
- `src/documents/classifier.py` — ML document classifier
- `src/documents/matching.py` — rule-based matching
- `src/documents/loggers.py` — logging mixin
- `src/documents/file_handling.py` — filename generation and directory management
- `src/documents/sanity_checker.py` — integrity audit tool
- `src/documents/views.py` — REST API upload endpoint
- `src/documents/serialisers.py` — DRF serializers for upload
- `src/paperless/consumers.py` — WebSocket StatusConsumer
- `src/paperless/settings.py` — all configuration variables
- `src/paperless/asgi.py` — ASGI application with WebSocket routing
- `src/paperless/urls.py` — URL patterns including WebSocket
- `src/paperless_tesseract/parsers.py` — OCR parser
- `src/paperless_tesseract/signals.py` — Tesseract parser registration
- `src/paperless_tesseract/apps.py` — Tesseract app config
- `src/paperless_text/parsers.py` — plain text parser
- `src/paperless_text/signals.py` — text parser registration
- `src/paperless_text/apps.py` — text app config
- `src/paperless_tika/parsers.py` — Tika parser
- `src/paperless_tika/signals.py` — Tika parser registration
- `src/paperless_tika/apps.py` — Tika app config
- `src/paperless_mail/mail.py` — email ingestion handler
- `src/paperless_mail/tasks.py` — email processing tasks
- `docker/supervisord.conf` — supervised process definitions
- `docker/docker-prepare.sh` — startup preparation script
- `docker/docker-entrypoint.sh` — container entry point

**Topics covered in the documentation:**
- File detection mechanisms (inotify, watchdog polling, REST upload, email fetch)
- Task queueing via Django-Q `async_task()`
- Consumer pre-checks (file existence, duplicate detection, directory creation)
- MIME type detection and parser selection
- OCR, text extraction, and Tika-based parsing
- Thumbnail generation and optimization
- Date extraction from filename and content
- ML classification (correspondent, document type, tags)
- Rule-based matching (any, all, literal, regex, fuzzy, auto)
- Inbox tag application
- Atomic database persistence within transaction
- File copy to managed storage (originals, archive, thumbnails)
- Search index update via Whoosh
- Django admin log entry creation
- Filename generation and file relocation
- WebSocket progress reporting protocol
- Pre/post-consume script execution
- Duplicate prevention via MD5 checksums
- Barcode-based PDF splitting (alternate path)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any source files in the repository
- **Frontend Angular code**: The `src-ui/` directory is not analyzed; WebSocket consumption on the client side is not documented
- **Test file modifications**: No test files are created or modified
- **Database schema migrations**: Not creating or running migrations
- **Deployment configuration changes**: Not modifying Docker, supervisor, or CI/CD configurations
- **Existing documentation updates**: Not modifying any files in the `docs/` directory
- **Feature additions or code refactoring**: No code changes of any kind
- **Mail ingestion IMAP protocol details**: Only the handoff to `consume_file` is documented, not IMAP connection management
- **User interface behavior**: UI components consuming WebSocket updates are not documented
- **Performance benchmarking**: No performance measurements or load testing
- **Security audit**: Not analyzing authentication, authorization, or encryption mechanisms beyond their role in the pipeline
- **Multi-user scenarios**: Not documenting permission models or user-specific behavior

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the deliverable is a standalone Markdown file, not part of the Sphinx build system
- **Documentation preview command**: Standard markdown preview via any markdown renderer
- **Diagram generation**: Mermaid diagrams embedded directly in the markdown file; renderable by GitHub, GitLab, or any Mermaid-compatible viewer
- **Documentation deployment**: Not applicable — the file is committed to the `blitzy/documentation` directory in the repository
- **Default format**: Markdown (`.md`) with embedded Mermaid diagram blocks
- **Citation requirement**: Every technical claim must reference a specific source file and line number from the repository at commit `542221a38dff`
- **Style guide**: Follow the project rule requiring thinking/rationale behind answers, with code as the source of truth and no assumptions
- **Documentation validation**: Verify all cited file paths exist in the repository; verify all referenced function names exist in the cited files

### 0.9.2 Key Configuration Variables Referenced in Documentation

The following settings from `src/paperless/settings.py` are referenced throughout the pipeline and should be documented in the deliverable:

| Setting Variable | Environment Variable | Default | Pipeline Role |
|-----------------|---------------------|---------|---------------|
| `CONSUMPTION_DIR` | `PAPERLESS_CONSUMPTION_DIR` | `../consume` | Directory watched for new files |
| `CONSUMER_POLLING` | `PAPERLESS_CONSUMER_POLLING` | `0` (inotify) | `0` = inotify, `>0` = polling interval in seconds |
| `CONSUMER_POLLING_DELAY` | `PAPERLESS_CONSUMER_POLLING_DELAY` | `5` | Seconds between file-stability checks |
| `CONSUMER_POLLING_RETRY_COUNT` | `PAPERLESS_CONSUMER_POLLING_RETRY_COUNT` | `5` | Max file-stability retries |
| `CONSUMER_RECURSIVE` | `PAPERLESS_CONSUMER_RECURSIVE` | `false` | Enable recursive directory watching |
| `CONSUMER_DELETE_DUPLICATES` | `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | `false` | Delete duplicate files from consumption dir |
| `CONSUMER_SUBDIRS_AS_TAGS` | `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | `false` | Create tags from subdirectory names |
| `CONSUMER_ENABLE_BARCODES` | `PAPERLESS_CONSUMER_ENABLE_BARCODES` | `false` | Enable barcode-based PDF splitting |
| `CONSUMER_IGNORE_PATTERNS` | `PAPERLESS_CONSUMER_IGNORE_PATTERNS` | `[".DS_STORE/*", "._*"]` | Glob patterns to ignore |
| `ORIGINALS_DIR` | — | `<MEDIA_ROOT>/documents/originals` | Storage for original document files |
| `ARCHIVE_DIR` | — | `<MEDIA_ROOT>/documents/archive` | Storage for archive PDF files |
| `THUMBNAIL_DIR` | — | `<MEDIA_ROOT>/documents/thumbnails` | Storage for thumbnail images |
| `INDEX_DIR` | — | `<DATA_DIR>/index` | Whoosh search index directory |
| `MODEL_FILE` | — | `<DATA_DIR>/classification_model.pickle` | Trained ML classifier model |
| `SCRATCH_DIR` | `PAPERLESS_SCRATCH_DIR` | `/tmp/paperless` | Temporary working directory |
| `MEDIA_LOCK` | — | `<MEDIA_ROOT>/media.lock` | Filesystem lock for media operations |
| `TRASH_DIR` | `PAPERLESS_TRASH_DIR` | `None` | Optional trash directory for deleted docs |
| `TASK_WORKERS` | `PAPERLESS_TASK_WORKERS` | CPU-dependent | Number of Django-Q worker processes |
| `OCR_LANGUAGE` | `PAPERLESS_OCR_LANGUAGE` | `eng` | Tesseract OCR language |
| `OCR_MODE` | `PAPERLESS_OCR_MODE` | `skip` | OCR processing mode |

## 0.10 Rules for Documentation

The following rules are explicitly mandated by the user's instructions and project configuration:

- **Do not modify any existing files in the source repository.** The deliverable is a new file only. No source code, configuration, test, or documentation files may be altered.
- **Base all answers on the code as the truth.** Every claim in the document must be traceable to specific source files at commit `542221a38dff`. No speculation or assumptions about behavior not evidenced in the code.
- **Provide thinking / rationale behind the answers.** Each section of the deliverable must include a rationale explaining how the conclusion was derived from the codebase evidence.
- **Do not make assumptions.** If the code is ambiguous about a behavior, state what is known and note the ambiguity rather than guessing.
- **Keep the explanation grounded in runtime observations.** Focus on what a developer or operator would actually see — log messages, task names, state changes, database records, filesystem artifacts — rather than theoretical code-reading descriptions.
- **Temporary test files allowed but must be cleaned up.** If temporary files are created during analysis, they must be removed before completion.
- **Output file naming**: The deliverable must be named `paperless-ngx_542221a38dff.md` (derived from the source branch name `paperless-ngx_542221a38dff`).
- **Output file location**: The deliverable must be placed in the `blitzy/documentation` directory in the destination repository.

## 0.11 References

### 0.11.1 Files and Folders Searched

The following source files were directly retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Core Document Ingestion Pipeline:**
- `src/documents/consumer.py` — Central `Consumer` class with `try_consume_file()`, pre-checks, progress reporting, and file storage logic
- `src/documents/tasks.py` — `consume_file()` task wrapper, barcode splitting, `train_classifier()`, `index_reindex()`, `bulk_update_documents()`
- `src/documents/management/commands/document_consumer.py` — Filesystem watcher using inotify or polling, file stability checks, async task dispatch

**Signal System:**
- `src/documents/signals/__init__.py` — Defines `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` signals
- `src/documents/signals/handlers.py` — Signal receivers: `add_inbox_tags()`, `set_correspondent()`, `set_document_type()`, `set_tags()`, `set_log_entry()`, `add_to_index()`, `update_filename_and_move_files()`, `cleanup_document_deletion()`
- `src/documents/apps.py` — `DocumentsConfig.ready()` connecting six handlers to `document_consumption_finished`

**Data Models and Storage:**
- `src/documents/models.py` — `Document`, `Correspondent`, `Tag`, `DocumentType`, `Log`, `SavedView`, `FileInfo` models
- `src/documents/file_handling.py` — `generate_unique_filename()`, `generate_filename()`, `create_source_path_directory()`, `delete_empty_directories()`
- `src/documents/index.py` — Whoosh search index: `get_schema()`, `open_index()`, `update_document()`, `add_or_update_document()`

**Classification and Matching:**
- `src/documents/classifier.py` — `DocumentClassifier` with scikit-learn MLP, `load_classifier()`, `train()`, `predict_*()` methods
- `src/documents/matching.py` — `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()` with six algorithm types

**Parsers:**
- `src/documents/parsers.py` — `DocumentParser` base class, `get_parser_class_for_mime_type()`, `parse_date()`, `run_convert()`
- `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser` with OCRmyPDF integration
- `src/paperless_tesseract/signals.py` — Tesseract parser declaration (PDF, JPEG, PNG, TIFF, GIF, BMP)
- `src/paperless_tesseract/apps.py` — Tesseract app config connecting signal
- `src/paperless_text/parsers.py` — `TextDocumentParser` for plain text and CSV
- `src/paperless_text/signals.py` — Text parser declaration
- `src/paperless_text/apps.py` — Text app config connecting signal
- `src/paperless_tika/parsers.py` — `TikaDocumentParser` for Office documents
- `src/paperless_tika/signals.py` — Tika parser declaration (DOC, DOCX, XLS, XLSX, PPT, PPTX, ODP, ODS, ODT, RTF)
- `src/paperless_tika/apps.py` — Tika app config with `PAPERLESS_TIKA_ENABLED` gate

**REST API and WebSocket:**
- `src/documents/views.py` — `PostDocumentView` handling document uploads
- `src/paperless/consumers.py` — `StatusConsumer` WebSocket broadcasting status_updates
- `src/paperless/asgi.py` — ASGI application with WebSocket routing
- `src/paperless/urls.py` — URL patterns including `ws/status/`

**Email Ingestion:**
- `src/paperless_mail/mail.py` — `MailAccountHandler`, attachment extraction, async_task dispatch
- `src/paperless_mail/tasks.py` — `process_mail_accounts()`, `process_mail_account()`

**Configuration and Infrastructure:**
- `src/paperless/settings.py` — All consumer, OCR, classifier, index, and task worker settings
- `src/documents/loggers.py` — `LoggingMixin` with UUID-based group correlation
- `src/documents/sanity_checker.py` — Integrity audit comparing database to filesystem
- `docker/supervisord.conf` — Three supervised processes: gunicorn, consumer, scheduler
- `docker/docker-prepare.sh` — Startup: migrations, search index rebuild, superuser creation
- `docker/docker-entrypoint.sh` — Container initialization and language installation

**Project Configuration:**
- `Pipfile` — Python dependency definitions with version constraints
- `.readthedocs.yml` — ReadTheDocs build configuration
- `docs/conf.py` — Sphinx configuration
- `src/paperless/version.py` — Version 1.7.0

**Existing Documentation Reviewed:**
- `docs/usage_overview.rst` — High-level ingestion overview (insufficient for user's detailed questions)
- `docs/configuration.rst` — Environment variable documentation
- `docs/index.rst` — Documentation landing page

### 0.11.2 Attachments

No attachments were provided by the user.

### 0.11.3 External Resources

No Figma screens or external URLs were provided. No web searches were performed — all analysis is derived entirely from the source code at commit `542221a38dff`.

