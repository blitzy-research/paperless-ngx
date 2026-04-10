# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new investigative documentation** that comprehensively answers a series of questions about how paperless-ngx processes documents at runtime. The user wants to observe and document the end-to-end ingestion pipeline behavior by submitting test PDFs, monitoring service interactions, examining log outputs, understanding ML classifier retraining conditions, and mapping the filesystem and database storage patterns that result from document consumption.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical investigation / Q&A analysis document
- **Output Artifact:** A markdown file named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation/` directory of the destination repository

The user's investigation spans five distinct areas of inquiry:

- **Ingestion Pipeline Behavior:** Which services participate when a PDF is submitted, and what is the sequence of events visible in the logs from start to finish?
- **Multi-Document Observation:** What happens when additional documents are uploaded sequentially — does the system react differently after each one?
- **ML Classifier Retraining Logic:** Does the machine learning classifier retrain automatically on every upload, or only under specific conditions? What log messages differentiate active training from the classifier staying idle?
- **Filesystem Storage Layout:** After processing completes, where does the document physically reside on disk? What is the default directory structure and filename pattern?
- **Database Persistence:** Which database tables receive new rows as part of the ingestion process?

### 0.1.2 Special Instructions and Constraints

- **No Modification of Existing Files:** The user explicitly states: "You should not modify or edit any existing files in the repository." This is a read-only investigation.
- **Temporary Artifacts Allowed:** Temporary scripts or test documents may be created to aid investigation, provided they are cleaned up afterward.
- **Implementation Rules (SWE-AtlasQnA-Repo):**
  - Create a new markdown document named `paperless-ngx_542221a38dff.md`
  - Provide thinking and rationale behind all answers
  - Base all answers on the code as the source of truth — no assumptions
  - Do not modify any existing files in the source repository
  - Place the generated document in the `blitzy/documentation` directory
- **Evidence-Based Answers:** All conclusions must be traceable to specific source files and line numbers in the codebase. No assumptions or speculation.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the ingestion pipeline**, we will trace the complete code path from all three entry points (directory watcher in `src/documents/management/commands/document_consumer.py`, REST API upload in `src/documents/views.py:PostDocumentView`, and email ingestion in `src/paperless_mail/mail.py`) through the Django-Q task queue into `src/documents/tasks.py:consume_file()` and the orchestrator in `src/documents/consumer.py:Consumer.try_consume_file()`.
- To **document service involvement**, we will analyze the three Supervisord-managed processes defined in `docker/supervisord.conf` (Gunicorn, Document Consumer, Django-Q Cluster), plus Redis for message brokering and WebSocket channel layer.
- To **document classifier retraining behavior**, we will analyze `src/documents/classifier.py:DocumentClassifier.train()`, `src/documents/tasks.py:train_classifier()`, and the hourly Django-Q schedule established in `src/documents/migrations/1001_auto_20201109_1636.py`.
- To **document filesystem storage patterns**, we will analyze `src/documents/file_handling.py:generate_filename()`, `src/documents/models.py:Document.source_path`, and the directory constants in `src/paperless/settings.py`.
- To **document database table writes**, we will trace all ORM `create()`, `save()`, and `add()` calls within the consumer pipeline and its signal handlers in `src/documents/signals/handlers.py` and `src/documents/apps.py`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- **Signal Handler Chain Documentation:** The `document_consumption_finished` signal fires six handlers registered in `src/documents/apps.py` — the ordering and behavior of each handler (inbox tagging, correspondent matching, document type matching, tag matching, admin log entry, search index update) must be documented to fully explain what happens after a document is consumed.
- **Classifier Scheduling vs. Consumption Loading:** A critical distinction exists between the classifier being *loaded* during each consumption (to predict labels) versus being *retrained* on an hourly schedule. This dual-mode behavior must be documented clearly.
- **WebSocket Progress Notifications:** The consumer broadcasts status updates at multiple milestones (STARTING 0%, WORKING 20%, WORKING 70%, WORKING 90–95%, SUCCESS 100%) via the Redis-backed channel layer. This is integral to understanding the log/event sequence.
- **Concurrency Control Mechanisms:** The `FileLock`, `transaction.atomic()`, and `AsyncWriter` patterns that protect the persistence phase are essential context for understanding multi-document behavior.
- **Default vs. Custom Filename Formats:** The `PAPERLESS_FILENAME_FORMAT` setting introduces a secondary filename pattern that diverges significantly from the default `{pk:07}{ext}` scheme. Both must be documented.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** hosted on Read the Docs with comprehensive narrative coverage of deployment, configuration, usage, and contribution workflows, but **no existing document that directly addresses the runtime ingestion pipeline behavior from an investigative or observational perspective**.

- **Documentation framework:** Sphinx (pinned at `~=4.5.0` in `Pipfile` dev dependencies)
- **Documentation theme:** `sphinx_rtd_theme` (Read the Docs theme)
- **Documentation generator configuration:** `docs/conf.py` — defines project metadata, extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), and static assets
- **Documentation hosting:** Read the Docs via `.readthedocs.yml` — configured with Python 3.8 and `docs/requirements.txt`
- **Documentation format:** reStructuredText (`.rst`) files
- **Build command:** `make html` (via `docs/Makefile`)
- **Documentation dependency manifest:** `docs/requirements.txt` (currently empty — placeholder)
- **Diagram tools:** None explicitly configured in the documentation system (no Mermaid, no PlantUML in the Sphinx pipeline)

Existing documentation pages in `docs/`:

| File | Coverage Area | Relevance to This Task |
|------|---------------|------------------------|
| `docs/index.rst` | Landing page and navigation hub | Low — structural only |
| `docs/setup.rst` | Installation, migration, reverse proxy, deployment | Medium — describes how services are started |
| `docs/configuration.rst` | All environment variables and runtime settings | High — documents `PAPERLESS_FILENAME_FORMAT`, consumer settings, OCR modes |
| `docs/usage_overview.rst` | Product model, ingestion methods, search, workflows | High — describes ingestion at a user level but not at runtime code level |
| `docs/advanced_usage.rst` | Advanced matching, hooks, filename handling | Medium — covers filename format variables and pre/post-consume scripts |
| `docs/administration.rst` | Backups, updates, utilities, indexing, archiving | Medium — documents management commands including classifier training |
| `docs/api.rst` | REST endpoints, authentication, uploads, search | High — documents the upload API endpoint |
| `docs/extending.rst` | Developer workflows, dev setup, localization, parsers | Medium — describes parser extension points |
| `docs/troubleshooting.rst` | Common operational failures and fixes | Low — operational troubleshooting |
| `docs/faq.rst` | Common support and deployment questions | Low |
| `docs/changelog.rst` | Release history | Low |

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to identify all code modules relevant to answering the user's questions:

- **Ingestion pipeline code path:** `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/management/commands/document_consumer.py`
- **Parser dispatch and registration:** `src/documents/parsers.py`, `src/paperless_tesseract/`, `src/paperless_text/`, `src/paperless_tika/`
- **Signal handlers (post-consumption):** `src/documents/signals/__init__.py`, `src/documents/signals/handlers.py`, `src/documents/apps.py`
- **ML classifier logic:** `src/documents/classifier.py`, `src/documents/tasks.py:train_classifier()`, `src/documents/migrations/1001_auto_20201109_1636.py`
- **Filesystem storage logic:** `src/documents/file_handling.py`, `src/documents/models.py:Document` (source_path, archive_path, thumbnail_path properties)
- **Database models and tables:** `src/documents/models.py` (Document, Tag, Correspondent, DocumentType, Log, SavedView, SavedViewFilterRule)
- **Settings and directory paths:** `src/paperless/settings.py` (lines 61–84 for directories, lines 297–318 for DB config, lines 373–412 for logging, lines 449–457 for Q_CLUSTER)
- **Service configuration:** `docker/supervisord.conf`, `docker/docker-entrypoint.sh`, `docker/docker-prepare.sh`
- **Logging configuration:** `src/paperless/settings.py` lines 373–412 (logging handlers, formatters, file paths)
- **API upload endpoint:** `src/documents/views.py:PostDocumentView` (lines 491–535)
- **Search index schema:** `src/documents/index.py` (lines 31–49)
- **Dependencies:** `Pipfile`, `requirements.txt`

Key directories examined:
- `src/documents/` — Core document engine (consumer, classifier, models, signals, tasks, views, file handling, index)
- `src/paperless/` — Project settings, ASGI bootstrap, URL routing
- `src/paperless_tesseract/` — OCR parser
- `src/paperless_text/` — Plain text parser
- `src/paperless_tika/` — Tika-based parser
- `src/documents/management/commands/` — Management commands including directory consumer and classifier training
- `src/documents/signals/` — Signal definitions and handler implementations
- `docker/` — Container entrypoint, preparation, and supervision configuration
- `docs/` — Existing Sphinx documentation workspace

### 0.2.3 Web Search Research Conducted

No web search research was required for this task. All answers are derived directly from the source code, which is the authoritative ground truth per the user's explicit instruction to "base your answers on the code." The codebase provides complete visibility into every aspect of the ingestion pipeline, classifier training, storage layout, and database persistence.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules contain the code that must be analyzed and documented to answer the user's five questions:

**Question 1 & 2 — Ingestion Pipeline Behavior and Service Involvement:**

- Module: `src/documents/consumer.py`
  - Public APIs: `Consumer.try_consume_file()`, `Consumer.pre_check_file_exists()`, `Consumer.pre_check_duplicate()`, `Consumer.pre_check_directories()`, `Consumer._store()`, `Consumer._send_progress()`
  - Current documentation: Partial coverage in `docs/usage_overview.rst` (user-level), no runtime trace documentation
  - Documentation needed: Step-by-step runtime log sequence, service interaction diagram, WebSocket event progression

- Module: `src/documents/tasks.py`
  - Public APIs: `consume_file()`, `train_classifier()`, `index_optimize()`, `index_reindex()`, `barcode_reader()`, `scan_file_for_separating_barcodes()`
  - Current documentation: None at code-trace level
  - Documentation needed: Task queue dispatch behavior, barcode check pre-processing

- Module: `src/documents/management/commands/document_consumer.py`
  - Public APIs: `Command.handle()`, `_consume()`, `_consume_wait_unmodified()`, `_tags_from_path()`
  - Current documentation: Minimal in `docs/configuration.rst`
  - Documentation needed: File detection mechanism (inotify vs polling), enqueue behavior

- Module: `src/documents/views.py` (PostDocumentView, lines 491–535)
  - Public APIs: `PostDocumentView.post()`
  - Current documentation: REST API documented in `docs/api.rst`
  - Documentation needed: Temp file creation, async_task dispatch, task_id generation

- Module: `src/documents/signals/handlers.py`
  - Public APIs: `add_inbox_tags()`, `set_correspondent()`, `set_document_type()`, `set_tags()`, `set_log_entry()`, `add_to_index()`, `update_filename_and_move_files()`, `cleanup_document_deletion()`
  - Current documentation: No specific documentation of the signal chain
  - Documentation needed: Full signal handler sequence with log messages

- Module: `docker/supervisord.conf`
  - Services defined: `gunicorn`, `consumer`, `scheduler` (qcluster)
  - Current documentation: Mentioned in `docs/setup.rst`
  - Documentation needed: Which process does what in the ingestion flow

**Question 3 — ML Classifier Retraining:**

- Module: `src/documents/classifier.py`
  - Public APIs: `DocumentClassifier.train()`, `DocumentClassifier.load()`, `DocumentClassifier.save()`, `DocumentClassifier.predict_correspondent()`, `DocumentClassifier.predict_document_type()`, `DocumentClassifier.predict_tags()`, `load_classifier()`, `preprocess_content()`
  - Current documentation: Brief mention in `docs/advanced_usage.rst`
  - Documentation needed: Training triggers, data_hash skip logic, log messages, FORMAT_VERSION=7

- Module: `src/documents/tasks.py:train_classifier()` (lines 48–72)
  - Current documentation: None
  - Documentation needed: MATCH_AUTO precondition check, hourly schedule, data_hash comparison

- Module: `src/documents/migrations/1001_auto_20201109_1636.py`
  - Defines: Django-Q hourly schedule for `train_classifier`, daily schedule for `index_optimize`
  - Current documentation: None
  - Documentation needed: Scheduling mechanism proof

**Question 4 — Filesystem Storage Layout:**

- Module: `src/documents/file_handling.py`
  - Public APIs: `generate_filename()`, `generate_unique_filename()`, `create_source_path_directory()`, `delete_empty_directories()`
  - Current documentation: Partial in `docs/advanced_usage.rst` (filename format variables)
  - Documentation needed: Default pattern `{pk:07}{ext}`, custom format variables, archive filename logic

- Module: `src/documents/models.py` (Document class properties, lines 222–278)
  - Properties: `source_path`, `archive_path`, `thumbnail_path`
  - Current documentation: None at code level
  - Documentation needed: Path computation logic for originals, archives, thumbnails

- Module: `src/paperless/settings.py` (lines 61–84)
  - Settings: `ORIGINALS_DIR`, `ARCHIVE_DIR`, `THUMBNAIL_DIR`, `SCRATCH_DIR`, `CONSUMPTION_DIR`, `DATA_DIR`, `MODEL_FILE`, `INDEX_DIR`
  - Current documentation: Covered in `docs/configuration.rst`
  - Documentation needed: Default values and directory structure visualization

**Question 5 — Database Tables:**

- Module: `src/documents/models.py`
  - Models: `Document`, `Correspondent`, `Tag`, `DocumentType`, `Log`, `SavedView`, `SavedViewFilterRule`
  - Current documentation: None at table-write level
  - Documentation needed: Which tables receive INSERTs during ingestion, with specific field mappings

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime trace documentation exists:** While `docs/usage_overview.rst` describes ingestion conceptually, no document traces the actual code path and log output sequence from file detection through final persistence.
- **Classifier retraining schedule undocumented at code level:** The hourly schedule established via Django-Q migration is not documented in any existing documentation file as a code-level finding.
- **Signal handler chain not documented:** The six handlers fired by `document_consumption_finished` and their ordering are defined in `src/documents/apps.py` but not documented anywhere.
- **Default filename pattern not explicitly documented from code perspective:** While `docs/advanced_usage.rst` documents the `PAPERLESS_FILENAME_FORMAT` custom format, the default fallback pattern `{pk:07}{ext}` is only apparent from reading `src/documents/file_handling.py`.
- **Database table write sequence not documented:** No existing documentation maps the specific ORM operations that create rows during ingestion.
- **Multi-document behavior not addressed:** No documentation discusses what changes (or doesn't change) when successive documents are uploaded.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document answering all five investigation areas. The document will be structured as follows:

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Introduction (investigation purpose and methodology)
    ├── Services and Process Architecture
    │   ├── Supervisord-managed processes
    │   ├── Redis role (broker + channel layer)
    │   └── Database backend
    ├── Ingestion Pipeline: Step-by-Step Runtime Trace
    │   ├── Entry points (directory watcher, REST API, email)
    │   ├── Task queue dispatch
    │   ├── Consumer pipeline phases (validation, extraction, persistence)
    │   ├── Signal handler chain
    │   └── Expected log message sequence
    ├── Multi-Document Upload Behavior
    │   ├── Parallel task execution
    │   ├── Duplicate detection
    │   └── Independent processing
    ├── ML Classifier: Training vs. Prediction
    │   ├── Hourly schedule mechanism
    │   ├── Preconditions for training
    │   ├── Data hash skip logic
    │   ├── Prediction during consumption (load, not retrain)
    │   └── Log messages for training vs. idle
    ├── Filesystem Storage Layout
    │   ├── Default directory structure
    │   ├── Default filename pattern
    │   ├── Custom PAPERLESS_FILENAME_FORMAT
    │   └── Archive and thumbnail paths
    ├── Database Tables Affected by Ingestion
    │   ├── documents_document (main record)
    │   ├── documents_document_tags (M2M assignments)
    │   ├── documents_log (application logs)
    │   ├── django_admin_log (audit trail)
    │   ├── Whoosh search index (file-based)
    │   └── django_q task records
    └── Summary and Key Findings
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- "Extract the complete ingestion code path from `src/documents/consumer.py:Consumer.try_consume_file()` by tracing every method call, signal emission, and log statement"
- "Map the classifier training flow from `src/documents/tasks.py:train_classifier()` through `src/documents/classifier.py:DocumentClassifier.train()`, including the data_hash comparison at line 163"
- "Derive the default filename pattern from `src/documents/file_handling.py:generate_filename()` lines 128–199, specifically the fallback at line 193: `f'{doc.pk:07}{counter_str}{filetype_str}'`"
- "Identify all database writes by tracing `Document.objects.create()` in `consumer.py` line 398, `document.tags.add()` in `handlers.py` line 230, `LogEntry.objects.create()` in `handlers.py` line 418, and `document.save()` calls throughout the pipeline"
- "Document the hourly schedule from `src/documents/migrations/1001_auto_20201109_1636.py` lines 10–13, which registers `train_classifier` as `Schedule.HOURLY`"

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for the ingestion pipeline flow and service architecture
- Code citations using `Source: /path/to/file.py:LineNumber` format
- Tables for structured data (database tables, directory paths, log messages)
- Consistent use of exact log message strings from the source code

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Service Architecture Diagram:** Flowchart showing Supervisord → three processes → shared resources (Redis, DB, filesystem)
- **Ingestion Pipeline Sequence:** Flowchart showing the 10 stages from file detection through WebSocket notification, with log messages annotated at each stage
- **Classifier Training vs. Prediction:** Flowchart distinguishing the hourly scheduled training path from the per-consumption prediction path
- **Filesystem Layout Diagram:** Tree diagram showing the default directory structure under `MEDIA_ROOT` and `DATA_DIR`
- **Database Write Sequence:** Sequence diagram showing which tables receive rows in which order during ingestion

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/classifier.py`, `src/documents/models.py`, `src/documents/signals/handlers.py`, `src/documents/file_handling.py`, `src/documents/index.py`, `src/documents/views.py`, `src/documents/apps.py`, `src/documents/loggers.py`, `src/documents/management/commands/document_consumer.py`, `src/paperless/settings.py`, `docker/supervisord.conf`, `docker/docker-entrypoint.sh`, `docker/docker-prepare.sh`, `src/documents/migrations/1001_auto_20201109_1636.py` | Comprehensive Q&A investigation document answering all five user questions about runtime ingestion behavior, service involvement, classifier retraining conditions, filesystem storage layout, and database table writes. Includes Mermaid diagrams, log message traces, code citations, and rationale for every answer. |

This is the only file to be created. No existing files are modified, updated, or deleted per the user's explicit constraint and the implementation rules.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Q&A Analysis
Source Code: 16 source files across src/documents/, src/paperless/, docker/
Sections:
    - Introduction (investigation scope and methodology)
    - Services and Process Architecture
        Source: docker/supervisord.conf, src/paperless/settings.py
    - Ingestion Pipeline Runtime Trace
        Source: src/documents/consumer.py, src/documents/tasks.py,
                src/documents/management/commands/document_consumer.py,
                src/documents/views.py:PostDocumentView
    - Signal Handler Chain
        Source: src/documents/apps.py, src/documents/signals/handlers.py
    - Multi-Document Upload Behavior
        Source: src/documents/consumer.py (duplicate detection, task independence)
    - ML Classifier: Training vs. Prediction
        Source: src/documents/classifier.py, src/documents/tasks.py,
                src/documents/migrations/1001_auto_20201109_1636.py
    - Filesystem Storage Layout
        Source: src/documents/file_handling.py, src/documents/models.py,
                src/paperless/settings.py
    - Database Tables Affected by Ingestion
        Source: src/documents/models.py, src/documents/signals/handlers.py,
                src/documents/index.py
    - Summary and Key Findings
Diagrams:
    - Service architecture (Mermaid flowchart)
    - Ingestion pipeline with log annotations (Mermaid flowchart)
    - Classifier training vs. prediction (Mermaid flowchart)
    - Filesystem directory tree (Mermaid/text tree)
    - Database write sequence (Mermaid sequence diagram)
Key Citations:
    src/documents/consumer.py (lines 180-377)
    src/documents/classifier.py (lines 115-249)
    src/documents/tasks.py (lines 48-72, 184-252)
    src/documents/models.py (lines 88-283)
    src/documents/signals/handlers.py (lines 30-431)
    src/documents/file_handling.py (lines 128-199)
    src/documents/apps.py (lines 11-27)
    src/documents/management/commands/document_consumer.py (lines 46-97)
    src/documents/views.py (lines 491-535)
    src/paperless/settings.py (lines 61-84, 297-318, 373-412, 449-457)
    src/documents/migrations/1001_auto_20201109_1636.py (lines 9-14)
    docker/supervisord.conf (full file)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone markdown document placed in `blitzy/documentation/` and does not integrate with the existing Sphinx documentation system in `docs/`. No changes to `docs/conf.py`, `.readthedocs.yml`, `docs/Makefile`, or any navigation/toctree configuration are needed.

### 0.5.4 Cross-Documentation Dependencies

- **No shared includes or templates:** The output document is self-contained.
- **No navigation link updates:** The document is not part of the Sphinx site.
- **No table of contents updates:** Not applicable.
- **No glossary/index updates:** Not applicable.
- **Source code references only:** The document references source files by path and line number for traceability, but does not create hyperlinks into the existing documentation system.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This task produces a standalone markdown file and does not require any documentation generation tools, build systems, or rendering frameworks. The output document uses standard GitHub-Flavored Markdown with embedded Mermaid diagram syntax, which is natively rendered by GitHub, GitLab, and most modern documentation platforms.

No documentation tool packages need to be installed or configured. For reference, the existing repository documentation toolchain is:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | ~=4.5.0 | Existing Sphinx documentation builder (not used for this task) |
| pip | sphinx_rtd_theme | * | Read the Docs theme for existing docs (not used for this task) |

The following runtime dependencies of the paperless-ngx project are **relevant to understanding the answers** documented in the output, though they are not dependencies of the documentation process itself:

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| pip | django | ~=4.0 | ORM, signals, management commands that drive the ingestion pipeline |
| pip | django-q | ~=1.3 | Task queue scheduling — hourly `train_classifier` schedule |
| pip | scikit-learn | ==1.0.2 | ML classifier (MLPClassifier, CountVectorizer) — pinned for model compatibility |
| pip | whoosh | ~=2.7.4 | Full-text search index updated during ingestion |
| pip | redis | * | Task broker and WebSocket channel layer |
| pip | channels | ~=3.0 | WebSocket status updates during ingestion |
| pip | channels-redis | * | Redis-backed channel layer for WebSocket groups |
| pip | python-magic | * | MIME type detection during ingestion validation |
| pip | ocrmypdf | ~=13.4 | OCR processing via Tesseract for PDF/image documents |
| pip | filelock | * | FileLock concurrency protection for filesystem mutations |
| pip | watchdog | ~=2.1.0 | Polling-based file system observer for the directory consumer |
| pip | inotifyrecursive | ~=0.3 | inotify-based file system observer (Linux, preferred) |
| pip | pyzbar | * | Barcode detection for PDF splitting |
| pip | pikepdf | ~=5.1 | PDF manipulation for barcode-based page separation |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates as part of this task. The output document is a new standalone artifact that does not alter any existing documentation references.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt contains five distinct questions. Coverage is measured by complete, evidence-based answers to each:

| Question Area | Coverage Target | Source Files Required | Status |
|---------------|----------------|----------------------|--------|
| Ingestion pipeline behavior and log sequence | 100% — full runtime trace with exact log messages | `consumer.py`, `tasks.py`, `document_consumer.py`, `views.py` | Addressed |
| Services involved and their roles | 100% — all three Supervisord processes + Redis + DB | `supervisord.conf`, `settings.py`, `docker-prepare.sh` | Addressed |
| Multi-document upload behavior | 100% — parallel execution, duplicate detection, independence | `consumer.py`, `tasks.py`, `settings.py` (Q_CLUSTER) | Addressed |
| ML classifier retraining conditions and log messages | 100% — schedule, preconditions, data_hash, exact log strings | `classifier.py`, `tasks.py`, migration `1001` | Addressed |
| Filesystem storage layout (directories, filename patterns) | 100% — default and custom patterns, all directories | `file_handling.py`, `models.py`, `settings.py` | Addressed |
| Database tables receiving new rows | 100% — every table with INSERT operations traced | `models.py`, `handlers.py`, `consumer.py`, `index.py` | Addressed |

Target coverage: **100%** — every question must be answered completely with code-level evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must cite specific source files with line numbers
- All log messages documented must be exact string literals from the source code (not paraphrased)
- The classifier training vs. prediction distinction must be unambiguous — reader should be able to determine from log output alone whether training is happening
- The filesystem layout must include both default and custom filename patterns with concrete examples
- Database table documentation must map to Django model names AND actual SQL table names

**Accuracy validation:**
- All code citations verified against actual file contents during the analysis phase
- Log message strings extracted verbatim from `logger.info()`, `logger.debug()`, `logger.warning()` calls in source
- Directory paths derived from `settings.py` constants with their default values
- Classifier schedule confirmed from Django-Q migration, not assumed from documentation

**Clarity standards:**
- Each answer section starts with a direct, one-sentence answer before providing supporting detail
- Mermaid diagrams accompany all complex flows (ingestion pipeline, classifier logic, service architecture)
- Tables used for structured reference data (directories, database tables, log messages)
- Rationale provided for each answer as required by the implementation rules

**Maintainability:**
- Source citations in `Source: /path/to/file.py:LineNumber` format for traceability
- Code line references allow future verification if source changes

### 0.7.3 Example and Diagram Requirements

- Minimum diagrams: 4 (service architecture, ingestion pipeline, classifier behavior, database write sequence)
- Log message examples: Every distinct log statement in the ingestion pipeline must be documented with its logger name, level, and message template
- Filename examples: At least two concrete examples showing default pattern (e.g., `0000001.pdf`) and custom format pattern (e.g., `2024-01-15 - Acme Corp - Invoice.pdf`)
- Database table examples: Each table must include the Django model name, SQL table name, and which fields are populated during ingestion

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole output artifact

**Source code analysis (read-only) covering:**
- `src/documents/consumer.py` — ingestion pipeline orchestrator
- `src/documents/tasks.py` — background task definitions including `consume_file` and `train_classifier`
- `src/documents/classifier.py` — ML classifier implementation (train, load, save, predict)
- `src/documents/models.py` — Django ORM models (Document, Tag, Correspondent, DocumentType, Log)
- `src/documents/signals/__init__.py` — signal definitions (consumption_started, consumption_finished, consumer_declaration)
- `src/documents/signals/handlers.py` — six consumption-finished handlers + post_save/post_delete/m2m_changed handlers
- `src/documents/apps.py` — signal handler registration
- `src/documents/file_handling.py` — filename generation and directory management
- `src/documents/index.py` — Whoosh search index schema and update logic
- `src/documents/views.py` — PostDocumentView REST API upload endpoint
- `src/documents/loggers.py` — LoggingMixin with correlation group support
- `src/documents/management/commands/document_consumer.py` — directory watcher command
- `src/documents/management/commands/document_create_classifier.py` — CLI classifier training wrapper
- `src/documents/migrations/1001_auto_20201109_1636.py` — Django-Q hourly/daily schedule registration
- `src/paperless/settings.py` — all directory paths, database config, logging config, Q_CLUSTER config
- `docker/supervisord.conf` — three supervised processes
- `docker/docker-entrypoint.sh` — container initialization
- `docker/docker-prepare.sh` — database migration, Redis probe, search index check
- `Pipfile` — Python dependency versions
- `docs/conf.py` — existing documentation framework identification
- `.readthedocs.yml` — documentation hosting configuration

**Topics explicitly covered:**
- Runtime ingestion pipeline behavior from file detection to final persistence
- All three entry points (directory watcher, REST API, email)
- Django-Q task queue dispatch and worker execution
- WebSocket status update progression
- Parser dispatch and OCR processing
- Signal handler chain (all six handlers)
- Pre/post consume script hooks
- ML classifier prediction (per-consumption) vs. training (hourly)
- Data hash comparison for training skip optimization
- Exact log messages from all logger instances
- Default and custom filesystem directory structures and filename patterns
- Database table writes (Document, Document_Tags M2M, Log, LogEntry, Whoosh index, Django-Q task)
- Multi-document behavior and task independence

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing file in the repository will be modified, per user instruction and implementation rules
- **Test file modifications:** No test files will be altered
- **Feature additions or code refactoring:** No functional changes to the codebase
- **Deployment configuration changes:** No changes to Docker, Compose, or infrastructure files
- **Frontend (Angular) code analysis:** The SPA in `src-ui/` is not analyzed beyond acknowledging it as the WebSocket consumer client
- **Email ingestion deep-dive:** `src/paperless_mail/` is mentioned as an entry point but not traced in full detail since the user specifically asks about submitting a PDF and observing the pipeline
- **Existing documentation updates:** No changes to `docs/*.rst` files or Sphinx configuration
- **Tika parser deep-dive:** The Tika parser in `src/paperless_tika/` is feature-flagged and not the default path for PDF ingestion; it is mentioned but not traced in detail
- **Database migration history:** Individual migration files are not documented except for migration `1001` which establishes the classifier training schedule
- **Security analysis:** Authentication, authorization, and security patterns are out of scope
- **Performance optimization:** Query optimization, caching, and scaling considerations are out of scope
- **Barcode splitting:** While the barcode check occurs in `tasks.py:consume_file()`, it is a conditional branch not relevant to standard PDF ingestion observation

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone markdown file, not part of the Sphinx build.
- **Documentation preview command:** Any markdown renderer (e.g., VS Code preview, GitHub web UI, or `grip` CLI tool).
- **Diagram generation command:** Mermaid diagrams are embedded inline in the markdown and rendered natively by GitHub/GitLab. No separate generation step is required.
- **Documentation deployment command:** Not applicable — the file is committed directly to the `blitzy/documentation/` directory.
- **Default format:** GitHub-Flavored Markdown (`.md`) with embedded Mermaid diagram blocks.
- **Citation requirement:** Every technical claim must reference the source file path and line number(s) from which it was derived.
- **Style guide:** Follow the implementation rule "SWE-AtlasQnA-Repo" — provide thinking/rationale behind answers, base all answers on code as truth, no assumptions.
- **Documentation validation:** Manual review for completeness against the five user questions. No automated linting or link checking is required for a standalone markdown file.

### 0.9.2 Output File Specifications

- **File path:** `blitzy/documentation/paperless-ngx_542221a38dff.md`
- **File naming derivation:** Branch name `paperless-ngx_542221a38dff` + `.md` extension, per implementation rule
- **Directory creation:** The `blitzy/documentation/` directory must be created if it does not already exist
- **Encoding:** UTF-8
- **Line endings:** LF (Unix-style)
- **Maximum line width:** No hard limit — markdown content flows naturally
- **Diagram format:** Mermaid fenced code blocks (` ```mermaid ... ``` `)
- **Code snippet format:** Fenced code blocks with language identifiers (` ```python ... ``` `, ` ```bash ... ``` `)
- **Table format:** GitHub-Flavored Markdown pipe tables

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** This is the user's primary constraint. The only file operation allowed is creating the new output document and the `blitzy/documentation/` directory.
- **Base all answers on the code as the source of truth.** Do not make assumptions. Every claim must be traceable to a specific file and line number in the repository.
- **Provide thinking and rationale behind all answers.** Each answer section must explain not just what happens, but why, and how the conclusion was reached from the code evidence.
- **Temporary artifacts must be cleaned up.** If any temporary scripts or test documents are created during investigation, they must be removed before completion. The final state of the repository must contain only the new `blitzy/documentation/paperless-ngx_542221a38dff.md` file as the sole addition.
- **Use exact log message strings from source code.** When documenting what log messages to expect, reproduce the exact format strings from the Python `logger` calls, not paraphrased versions.
- **Include Mermaid diagrams for all complex flows.** The ingestion pipeline, classifier behavior, service architecture, and database write sequence must each have a visual diagram.
- **Cite source files with path and line numbers.** Every technical claim must include a reference in the format `Source: path/to/file.py:LineNumber` or `Source: path/to/file.py (lines X-Y)`.
- **Document both default and custom behaviors.** Where the system has configurable behavior (e.g., filename format, OCR mode, polling vs. inotify), document the default behavior first, then note how configuration changes the behavior.
- **Answer all five user questions completely.** No question may be left partially answered or deferred. Every question must have a definitive, code-backed answer.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were systematically analyzed to derive the conclusions and answers documented in this Agent Action Plan:

**Core Ingestion Pipeline:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `src/documents/consumer.py` | 1–433 (full file) | 10-stage ingestion orchestrator; `try_consume_file()` method; WebSocket progress at 0%, 20%, 70%, 90%, 95%, 100%; MD5 duplicate detection; MIME type detection via python-magic; parser dispatch; `transaction.atomic()` persistence; `FileLock` for file copies; source file deletion |
| `src/documents/tasks.py` | 1–281 (full file) | `consume_file()` entry point from Django-Q; barcode splitting pre-check; `train_classifier()` with MATCH_AUTO precondition and data_hash comparison; `index_optimize()` and `index_reindex()` maintenance tasks |
| `src/documents/management/commands/document_consumer.py` | 1–241 (full file) | Directory watcher command; inotify (CLOSE_WRITE, MOVED_TO) vs. polling observer; 0.5s debounce; file readiness check (50 retries × 10ms); `async_task("documents.tasks.consume_file")` enqueue; ignore patterns; recursive scanning |
| `src/documents/views.py` | 491–535 | `PostDocumentView.post()` — writes temp file to SCRATCH_DIR, generates task_id, dispatches `async_task("documents.tasks.consume_file")` |

**Signal Handlers and Registration:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `src/documents/apps.py` | 1–29 (full file) | `DocumentsConfig.ready()` connects six handlers to `document_consumption_finished`: `add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index` |
| `src/documents/signals/__init__.py` | (via folder summary) | Defines three signals: `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` |
| `src/documents/signals/handlers.py` | 1–432 (full file) | Six consumption handlers; `update_filename_and_move_files` on `post_save` and `m2m_changed`; `cleanup_document_deletion` on `post_delete`; `FileLock` for file moves; `LogEntry.objects.create()` for admin audit |

**ML Classifier:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `src/documents/classifier.py` | 1–293 (full file) | `DocumentClassifier` with FORMAT_VERSION=7; `train()` uses SHA1 data_hash to skip unchanged data; scikit-learn MLPClassifier + CountVectorizer (unigrams/bigrams, min_df=0.01); excludes inbox-tagged documents from training; `load()` validates version; `save()` uses atomic temp file pattern |
| `src/documents/migrations/1001_auto_20201109_1636.py` | 1–34 (full file) | `schedule("documents.tasks.train_classifier", schedule_type=Schedule.HOURLY)` and `schedule("documents.tasks.index_optimize", schedule_type=Schedule.DAILY)` |
| `src/documents/management/commands/document_create_classifier.py` | (via folder summary) | CLI wrapper for manual `train_classifier()` invocation |

**Filesystem Storage:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `src/documents/file_handling.py` | 1–199 (full file) | `generate_filename()` — default pattern `{pk:07}{ext}`, custom via `PAPERLESS_FILENAME_FORMAT` with variables: title, correspondent, document_type, created, added, asn, tags, tag_list; `generate_unique_filename()` appends `_01`, `_02` to avoid collisions |
| `src/documents/models.py` | 1–467 (full file) | `Document.source_path` → `ORIGINALS_DIR/{filename}`; `Document.archive_path` → `ARCHIVE_DIR/{archive_filename}`; `Document.thumbnail_path` → `THUMBNAIL_DIR/{pk:07}.png`; seven ORM models: Document, Correspondent, Tag, DocumentType, Log, SavedView, SavedViewFilterRule |
| `src/paperless/settings.py` | 61–84, 297–318, 373–412, 449–457, 478–506, 584 | `ORIGINALS_DIR = MEDIA_ROOT/documents/originals`; `ARCHIVE_DIR = MEDIA_ROOT/documents/archive`; `THUMBNAIL_DIR = MEDIA_ROOT/documents/thumbnails`; `MODEL_FILE = DATA_DIR/classification_model.pickle`; `INDEX_DIR = DATA_DIR/index`; `SCRATCH_DIR = /tmp/paperless`; `DATABASES` config; `LOGGING` config; `Q_CLUSTER` config; consumer settings |

**Infrastructure and Services:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `docker/supervisord.conf` | Full file | Three processes: `gunicorn` (ASGI server), `consumer` (directory watcher), `scheduler` (qcluster Django-Q workers) |
| `docker/docker-entrypoint.sh` | 1–93 (full file) | Container initialization: UID/GID remapping, directory creation, permission fixing, optional OCR language installation |
| `docker/docker-prepare.sh` | 1–82 (full file) | PostgreSQL readiness probe, Redis readiness probe, flock-protected migrations, search index version check, superuser creation |

**Logging and Dependencies:**

| File Path | Lines Examined | Key Findings |
|-----------|---------------|--------------|
| `src/documents/loggers.py` | 1–22 (full file) | `LoggingMixin` with UUID correlation group and configurable logger name |
| `src/documents/index.py` | 1–50 | Whoosh schema: 16 fields including id, title, content, asn, correspondent, tag, type, created, modified, added |
| `Pipfile` | 1–72 (full file) | Key dependencies: django ~=4.0, django-q ~=1.3, scikit-learn ==1.0.2, whoosh ~=2.7.4, channels ~=3.0 |
| `docs/conf.py` | 1–332 (full file) | Sphinx documentation config: RTD theme, autodoc/intersphinx/viewcode extensions |
| `.readthedocs.yml` | 1–17 (full file) | Read the Docs config: Python 3.8, docs/conf.py, docs/requirements.txt |

**Folders Explored:**

| Folder Path | Depth | Key Contents Found |
|-------------|-------|-------------------|
| `` (root) | 0 | Repository structure: 7 top-level directories, 20 files |
| `src/` | 1 | 6 Django apps: documents, paperless, paperless_mail, paperless_tesseract, paperless_text, paperless_tika |
| `src/documents/` | 2 | 20 Python modules + 6 subfolders (tests, management, migrations, signals, static, templates) |
| `src/documents/signals/` | 3 | `__init__.py` (3 signal definitions), `handlers.py` (6 consumption handlers + model signal handlers) |
| `src/documents/management/` | 3 | `commands/` subfolder |
| `src/documents/management/commands/` | 4 | 14 management commands including `document_consumer.py`, `document_create_classifier.py` |
| `docs/` | 1 | 13 RST files, conf.py, Makefile, Dockerfile, _static/, _templates/ |
| `docker/` | 1 | entrypoint, prepare, supervisord.conf, compose configs, imagemagick policy |

### 0.11.2 Tech Spec Sections Referenced

| Section | Content Retrieved | Relevance |
|---------|-------------------|-----------|
| 1.1 Executive Summary | Project overview, version 1.7.0, stakeholders | General project context |
| 4.1 High-Level System Workflow | Container startup, process supervision, data flow overview | Service architecture and initialization sequence |
| 4.2 Core Business Processes | 10-stage ingestion pipeline, directory watcher, OCR dispatch, classification workflow | Core pipeline behavior documentation |
| 5.2 Component Details | Documents engine internals, parser subsystem, Docker infrastructure | Component interaction patterns |
| 6.2 Database Design | Schema design, ER model, indexing strategy, data management, filesystem layout | Database tables and storage architecture |
| 9.5 Document Ingestion Pipeline Quick Reference | 10-stage pipeline diagram, entry points table | Pipeline stage overview |

### 0.11.3 Attachments and External References

- **No attachments were provided** by the user for this task.
- **No Figma URLs** were provided.
- **No external URLs** were referenced in the user's prompt.
- **All answers are derived exclusively from the source code repository** — no external documentation, blog posts, or third-party resources were consulted.

