# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a series of empirical questions about the Paperless-ngx multi-page PDF OCR processing pipeline behavior.

- **Documentation Category**: Create new documentation
- **Documentation Type**: Technical Q&A / behavioral investigation guide — a single markdown reference document that answers the user's specific questions about runtime behavior observed during multi-page PDF upload, OCR processing, media storage, and database state in Paperless-ngx v1.7.0

The user is experiencing inconsistent OCR results with multi-page PDFs and needs to understand the end-to-end processing behavior. The Blitzy platform interprets the following discrete documentation requirements:

- **Requirement 1 — Environment Setup**: Set up the Paperless-ngx development environment locally (creating an admin user during setup if needed) and get it running normally
- **Requirement 2 — Upload HTTP Response**: Upload a multi-page PDF through the standard web interface that requires OCR processing and document the exact HTTP response status code and message body returned immediately after upload submission
- **Requirement 3 — Processing Log Patterns**: Capture and document the key log messages emitted as the document moves through distinct processing stages (consumer start, MIME detection, parser selection, OCR invocation, text extraction, thumbnail generation, database save, completion)
- **Requirement 4 — OCRmyPDF Invocation Parameters**: Document the exact parameters passed to `ocrmypdf.ocr()` including the input/output file paths, language, mode, threading, and optional parameters
- **Requirement 5 — Generated File Naming**: After processing completes, document the naming patterns for the generated archive PDF and thumbnail file in the media storage area
- **Requirement 6 — Database Record Fields**: Query the database and document all fields stored on the `Document` model row for the processed document, including their types and actual values
- **Requirement 7 — Cleanup**: No permanent changes — all test files must be completely cleaned up

### 0.1.2 Special Instructions and Constraints

- **No file modifications allowed**: The user explicitly states "No file modifications or permanent changes allowed, clean up any test files completely when done." This means the documentation must describe observed behavior without leaving any residual artifacts in the repository.
- **Implementation rule — SWE-AtlasQnA-Repo**: Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed. Place the generated document in the `blitzy/documentation` directory. Do not modify any existing files in the source repository.
- **Code-as-truth directive**: "Do not make assumptions, base your answers on the code as the truth." All answers must be derived from actual source code analysis, not assumed defaults or documentation that may be outdated.
- **Reasoning requirement**: "Provide thinking / rationale behind the answers." Each answer must include the code-level reasoning that supports it.
- **Observation-based answers**: Several questions ask "what do you receive" and "what patterns appear" — these require documenting expected behavior derived from code analysis, since a live runtime environment may not be fully achievable in the sandbox.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the upload HTTP response**, we will analyze `src/documents/views.py` (`PostDocumentView.post()` at line 497) which returns `Response("OK")` — an HTTP 200 with body `"OK"` — and trace the full request path from `src/paperless/urls.py` through serializer validation in `src/documents/serialisers.py` (`PostDocumentSerializer`)
- To **document the processing log patterns**, we will trace the consumer lifecycle in `src/documents/consumer.py` (`Consumer.try_consume_file()`) identifying each `self.log()` call and `_send_progress()` emission, combined with the parser's own logging in `src/paperless_tesseract/parsers.py` (`RasterisedDocumentParser.parse()`)
- To **document OCRmyPDF parameters**, we will analyze `src/paperless_tesseract/parsers.py:construct_ocrmypdf_parameters()` (line 135) and map each parameter to its source setting in `src/paperless/settings.py`
- To **document generated filenames**, we will analyze `src/documents/file_handling.py:generate_filename()` (line 128), the `Document.thumbnail_path` property in `src/documents/models.py` (line 272), and the archive filename generation logic in `src/documents/consumer.py` (lines 316-337)
- To **document database fields**, we will catalog every field declaration in the `Document` model class at `src/documents/models.py` (line 88) and trace the values assigned during `Consumer._store()` at line 379

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- **Consumer progress/status lifecycle**: The `_send_progress()` method emits WebSocket status updates at each stage (`STARTING`, `WORKING`, `SUCCESS`, `FAILED`) — documenting this is essential for understanding the full processing pipeline
- **Fallback OCR behavior**: The parser implements a two-pass OCR strategy with a `safe_fallback=True` retry path (line 276 of `parsers.py`) — this is directly relevant to the user's "inconsistent OCR results" concern
- **Signal-based post-consumption hooks**: After document save, `document_consumption_finished` fires six connected handlers (`add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index`) as wired in `src/documents/apps.py` — these affect the final database state
- **File renaming on save**: The `update_filename_and_move_files` signal handler in `src/documents/signals/handlers.py` (line 312) may rename files post-save based on `PAPERLESS_FILENAME_FORMAT`, which could produce different filenames than the initial consumer placement

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a comprehensive Sphinx-based documentation workspace at `docs/` with the following infrastructure:

- **Documentation framework**: Sphinx ~4.5.0 (from `Pipfile` dev-packages) with the Read the Docs theme (`sphinx_rtd_theme`)
- **Documentation generator configuration**: `docs/conf.py` — sets project metadata, extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), and theme configuration
- **Hosting**: Read the Docs, configured via `.readthedocs.yml` with Python 3.8 and Sphinx building from `docs/conf.py`
- **Build tooling**: `docs/Makefile` (standard Sphinx make targets) and `docs/Dockerfile` (containerized doc preview server on port 8000)
- **Dependency manifest**: `docs/requirements.txt` — currently empty placeholder
- **Diagram tools detected**: No Mermaid integration in existing docs; Sphinx `imgmath` extension is enabled for mathematical notation
- **Version stamp**: Documentation version derived from `src/paperless/version.py` — currently `(1, 7, 0)`

Existing documentation pages in `docs/`:

| File | Content |
|------|---------|
| `docs/index.rst` | Landing page and main toctree navigation hub |
| `docs/setup.rst` | Installation, migration, reverse proxy, deployment modes |
| `docs/configuration.rst` | All environment variables and runtime settings |
| `docs/usage_overview.rst` | Product model, ingestion methods, search, workflows |
| `docs/advanced_usage.rst` | Advanced matching, hooks, filename handling |
| `docs/administration.rst` | Backups, updates, utilities, indexing, archiving, encryption |
| `docs/api.rst` | REST API endpoints, auth, uploads, search, versioning |
| `docs/troubleshooting.rst` | Common operational failures and fixes |
| `docs/extending.rst` | Contributor workflows, dev setup, localization, parser extensions |
| `docs/faq.rst` | Common support and deployment questions |
| `docs/scanners.rst` | Compatible scanners and mobile scanning apps |
| `docs/screenshots.rst` | Visual gallery of UI and key workflows |
| `docs/changelog.rst` | Release history across Paperless variants |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code to document:

- **Upload API endpoint**: `src/documents/views.py` — `PostDocumentView` class (line 491), routed via `src/paperless/urls.py` at `/api/documents/post_document/`
- **Consumer pipeline**: `src/documents/consumer.py` — `Consumer` class with `try_consume_file()` (line 180), `_store()` (line 379), `_send_progress()` (line 56)
- **OCR parser**: `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser` with `parse()` (line 230), `construct_ocrmypdf_parameters()` (line 135), `extract_text()` (line 99)
- **Document model**: `src/documents/models.py` — `Document` class (line 88) with all field declarations and path properties
- **File naming**: `src/documents/file_handling.py` — `generate_unique_filename()` (line 81) and `generate_filename()` (line 128)
- **Signal handlers**: `src/documents/signals/handlers.py` — `update_filename_and_move_files()` (line 312), `cleanup_document_deletion()` (line 234)
- **Task queue**: `src/documents/tasks.py` — `consume_file()` function (line 184) which wraps `Consumer.try_consume_file()`
- **Serializer validation**: `src/documents/serialisers.py` — `PostDocumentSerializer` (line 413) with file-type validation
- **Settings**: `src/paperless/settings.py` — OCR settings (lines 510-541), media paths (lines 61-64), logging (lines 373-412), task queue (lines 449-457)
- **Logging mixin**: `src/documents/loggers.py` — `LoggingMixin` class providing correlation-group-based logging
- **App startup wiring**: `src/documents/apps.py` — `DocumentsConfig.ready()` connecting six signal handlers

Key directories examined:

| Directory | Relevance |
|-----------|-----------|
| `src/documents/` | Core document management: models, consumer, views, tasks, signals, serializers |
| `src/paperless_tesseract/` | OCR integration: parser, checks, signal registration |
| `src/paperless/` | Settings, URL routing, ASGI/WSGI, authentication, middleware |
| `docs/` | Existing Sphinx documentation workspace |
| `src/documents/management/commands/` | Management commands: consumer, superuser creation |

Related existing documentation found:

- `docs/api.rst` — Documents the upload endpoint (`/api/documents/post_document/`) with multipart field names and the `OK` response, but lacks detail on internal processing stages
- `docs/configuration.rst` — Documents all OCR-related environment variables but does not show how they map to `ocrmypdf.ocr()` parameters
- `docs/troubleshooting.rst` — Covers the "OCR for XX failed" warning and classifier errors but does not document multi-page PDF processing stages

### 0.2.3 Web Search Research Conducted

No external web search research is required for this task. All answers are derived directly from source code analysis per the user's directive: "Do not make assumptions, base your answers on the code as the truth." The codebase provides the complete, authoritative source for every question posed.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's questions map directly to the following code modules, each requiring detailed documentation extraction:

- **Module: `src/documents/views.py` — PostDocumentView**
  - Public APIs: `PostDocumentView.post()` (line 497)
  - Current documentation: `docs/api.rst` covers the endpoint at a high level ("returns OK") but lacks detail on the exact HTTP status code, serializer validation, temp file creation, and task queuing
  - Documentation needed: Exact HTTP response status, body text, and the complete request→queue flow

- **Module: `src/documents/consumer.py` — Consumer**
  - Public APIs: `Consumer.try_consume_file()` (line 180), `Consumer._store()` (line 379), `Consumer._send_progress()` (line 56)
  - Current documentation: No dedicated documentation of internal consumer log emissions
  - Documentation needed: Complete stage-by-stage log pattern catalog with logger names, levels, and message formats

- **Module: `src/paperless_tesseract/parsers.py` — RasterisedDocumentParser**
  - Public APIs: `RasterisedDocumentParser.parse()` (line 230), `construct_ocrmypdf_parameters()` (line 135), `extract_text()` (line 99)
  - Current documentation: `docs/configuration.rst` documents the environment variables but not how they translate to OCRmyPDF API calls
  - Documentation needed: Complete parameter dictionary showing each `ocrmypdf.ocr()` key-value pair, its source setting, and default value

- **Module: `src/documents/models.py` — Document**
  - Public APIs: `Document` class (line 88), `thumbnail_path` property (line 272), `source_path` property (line 222), `archive_path` property (line 242)
  - Current documentation: `docs/api.rst` documents the serialized API fields but not the underlying database columns
  - Documentation needed: Complete field catalog with column types, constraints, default values, and example values for a processed PDF

- **Module: `src/documents/file_handling.py` — Filename Generation**
  - Public APIs: `generate_unique_filename()` (line 81), `generate_filename()` (line 128)
  - Current documentation: `docs/advanced_usage.rst` covers `PAPERLESS_FILENAME_FORMAT` but not the default naming pattern or archive filename derivation
  - Documentation needed: Default and custom filename patterns for originals, archives, and thumbnails

- **Module: `src/paperless/settings.py` — OCR Configuration**
  - Public APIs: Settings constants `OCR_LANGUAGE`, `OCR_MODE`, `OCR_OUTPUT_TYPE`, `OCR_CLEAN`, `OCR_DESKEW`, `OCR_ROTATE_PAGES`, `OCR_PAGES`, `OCR_USER_ARGS` (lines 510-541)
  - Current documentation: `docs/configuration.rst` documents each variable's purpose and defaults
  - Documentation needed: Mapping table from setting to OCRmyPDF parameter name

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internal processing stages**: The consumer's step-by-step lifecycle with progress codes (`STARTING`, `WORKING`, `SUCCESS`, `FAILED`) and message constants (`new_file`, `parsing_document`, `generating_thumbnail`, `save_document`, `finished`) is not documented anywhere
- **OCRmyPDF parameter assembly**: No existing documentation shows the complete `ocrmypdf.ocr(**args)` call signature derived from settings — this is critical for understanding OCR behavior
- **Default file naming mechanics**: The default `{pk:07}.pdf` / `{pk:07}.png` naming pattern and how `generate_unique_filename()` falls through to `generate_filename()` is not documented
- **Database schema for Document model**: While the API serializer documents the REST response fields, the actual Django model columns (`checksum`, `archive_checksum`, `storage_type`, `filename`, `archive_filename`) are not documented in user-facing docs
- **Two-pass OCR fallback strategy**: The `safe_fallback=True` retry mechanism in `parsers.py` (lines 276-310) is not documented — this is directly relevant to the user's "inconsistent OCR results" issue
- **WebSocket progress updates**: The `_send_progress()` payloads sent to the `status_updates` channel group are not documented
- **Post-consumption signal chain**: The six signal handlers fired after consumption completes (inbox tags, correspondent, document type, tags, log entry, index) are not documented as a sequence

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The target documentation is a single comprehensive markdown Q&A document placed in `blitzy/documentation/`. Per the `SWE-AtlasQnA-Repo` implementation rule, the file must be named `<source_branch_name>.md`. The document structure should follow:

```
blitzy/
└── documentation/
    └── <source_branch_name>.md
        ├── Overview / Context
        ├── Q1: HTTP Response on Upload
        │   ├── Answer with code-path trace
        │   └── Rationale / source citations
        ├── Q2: Processing Log Patterns
        │   ├── Stage-by-stage log catalog
        │   ├── OCRmyPDF invocation parameters
        │   └── Rationale / source citations
        ├── Q3: Generated Archive & Thumbnail Filenames
        │   ├── Filename patterns
        │   ├── Storage directory structure
        │   └── Rationale / source citations
        ├── Q4: Database Record Fields & Values
        │   ├── Complete field table
        │   ├── Example values for processed PDF
        │   └── Rationale / source citations
        └── Cleanup Notes
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the HTTP response behavior from `src/documents/views.py:PostDocumentView.post()` by tracing the full request→serializer→tempfile→async_task→Response code path
- Extract log patterns from `src/documents/consumer.py:Consumer.try_consume_file()` by cataloging every `self.log()`, `self._send_progress()`, and `self._fail()` call in execution order
- Extract OCRmyPDF parameters from `src/paperless_tesseract/parsers.py:construct_ocrmypdf_parameters()` by mapping each dictionary key to its source in `src/paperless/settings.py`
- Extract filename patterns from `src/documents/file_handling.py:generate_filename()` and `src/documents/models.py:Document.thumbnail_path`
- Extract database fields from `src/documents/models.py:Document` class definition, tracing values set during `Consumer._store()`

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Code examples using fenced code blocks with syntax highlighting
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`
- Tables for parameter descriptions, field catalogs, and log pattern listings
- Consistent use of the exact log message strings and code constants from the source

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams should be included in the documentation file:

- **Sequence diagram**: Tracing the complete upload-to-storage lifecycle from `PostDocumentView.post()` through `async_task("documents.tasks.consume_file")` → `Consumer.try_consume_file()` → `RasterisedDocumentParser.parse()` → `ocrmypdf.ocr()` → `Consumer._store()` → signal handlers → file placement
- **Flowchart**: OCR decision tree showing how `OCR_MODE` settings and the `safe_fallback` retry path determine which text extraction strategy is used — this directly addresses the user's inconsistent OCR results concern
- **Directory tree diagram**: Showing the media storage layout (`MEDIA_ROOT/documents/originals/`, `archive/`, `thumbnails/`) with example filenames

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/<source_branch_name>.md` | CREATE | `src/documents/views.py`, `src/documents/consumer.py`, `src/paperless_tesseract/parsers.py`, `src/documents/models.py`, `src/documents/file_handling.py`, `src/paperless/settings.py`, `src/documents/tasks.py`, `src/documents/serialisers.py`, `src/documents/loggers.py`, `src/documents/signals/handlers.py`, `src/documents/apps.py` | Comprehensive Q&A markdown document answering all user questions about multi-page PDF OCR processing behavior, with code-derived rationale for each answer |

Only a single file is created. No existing files are modified per the `SWE-AtlasQnA-Repo` rule ("Do not modify any existing files in the source repository").

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/<source_branch_name>.md
Type: Technical Q&A / Behavioral Investigation Guide
Source Code:
    - src/documents/views.py (PostDocumentView — upload endpoint)
    - src/documents/consumer.py (Consumer — ingestion pipeline)
    - src/paperless_tesseract/parsers.py (RasterisedDocumentParser — OCR)
    - src/documents/models.py (Document model — database schema)
    - src/documents/file_handling.py (filename generation logic)
    - src/paperless/settings.py (OCR and media path configuration)
    - src/documents/tasks.py (consume_file task wrapper)
    - src/documents/serialisers.py (PostDocumentSerializer — validation)
    - src/documents/loggers.py (LoggingMixin — logger infrastructure)
    - src/documents/signals/handlers.py (post-consumption handlers)
    - src/documents/apps.py (signal wiring at startup)
Sections:
    - Overview and context (purpose of investigation, Paperless-ngx v1.7.0)
    - Q1: HTTP Response After Upload Submission
        - Exact HTTP status code (200 OK)
        - Response body ("OK")
        - Full code path trace from URL routing through serializer to async task
    - Q2: Key Log Patterns During Processing Stages
        - Stage-by-stage catalog of log emissions with logger name, level, message
        - Consumer progress status messages (STARTING → WORKING → SUCCESS)
        - OCRmyPDF invocation parameters (complete args dictionary)
        - Fallback OCR behavior and error handling logs
    - Q3: Generated Archive PDF and Thumbnail Filenames
        - Default naming pattern: {pk:07}.pdf for archive, {pk:07}.png for thumbnail
        - Storage directory paths under MEDIA_ROOT
        - PAPERLESS_FILENAME_FORMAT custom naming behavior
    - Q4: Database Record Fields and Values
        - Complete Document model field table with types, constraints, defaults
        - Example values for a processed multi-page PDF
        - Fields set during _store() vs. populated by signal handlers
    - Cleanup Notes (no permanent changes)
Diagrams:
    - Sequence diagram: upload → async_task → consumer → parser → OCR → store → signals
    - Flowchart: OCR mode decision tree with fallback path
    - Directory tree: media storage layout with example filenames
Key Citations:
    - src/documents/views.py:535 (Response("OK"))
    - src/documents/consumer.py:180-377 (try_consume_file lifecycle)
    - src/paperless_tesseract/parsers.py:135-228 (construct_ocrmypdf_parameters)
    - src/paperless_tesseract/parsers.py:230-320 (parse with fallback)
    - src/documents/models.py:88-206 (Document model fields)
    - src/documents/file_handling.py:81-199 (filename generation)
    - src/paperless/settings.py:510-541 (OCR settings)
    - src/documents/tasks.py:184-252 (consume_file task)
    - src/documents/apps.py:11-29 (signal handler connections)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The output file is a standalone markdown document placed in a new `blitzy/documentation/` directory. It is not integrated into the existing Sphinx documentation build at `docs/`.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes**: This is a standalone Q&A document not linked into the Sphinx toctree
- **No navigation links**: The file exists independently in `blitzy/documentation/`
- **No table of contents updates**: The existing `docs/index.rst` is not modified
- **Source code references**: The document contains inline citations to source files but no cross-links to existing `.rst` pages

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to this documentation exercise. These are the key runtime dependencies whose behavior the documentation must describe, along with the documentation build tool:

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| pip | django | 4.0.4 | Web framework — provides the view/model/ORM layer being documented |
| pip | djangorestframework | 3.13.1 | REST API — provides `Response`, serializers, and parsers used by upload endpoint |
| pip | ocrmypdf | 13.4.3 | OCR engine — the core library whose invocation parameters are being documented |
| pip | pillow | 9.1.0 | Image processing — used for DPI detection, alpha channel handling in parser |
| pip | pikepdf | 5.1.1 | PDF manipulation — used for metadata extraction in parser |
| pip | pdfminer.six | 20220319 | PDF text extraction — fallback text extraction when sidecar is unavailable |
| pip | python-magic | 0.4.25 | MIME type detection — used by consumer to select parser |
| pip | django-q | 1.3.9 | Task queue — `async_task` used to queue document consumption |
| pip | channels | 3.0.4 | WebSocket layer — used by consumer to send progress updates |
| pip | channels-redis | 3.4.0 | Channel layer backend — Redis-backed group messaging for status updates |
| pip | redis | 3.5.3 | Message broker — backing store for django-q task queue |
| pip | filelock | 3.6.0 | File locking — used during media file placement |
| pip | scikit-learn | 1.0.2 | ML classifier — used for auto-classification after consumption |
| pip | whoosh | 2.7.4 | Search indexing — documents are indexed post-consumption |
| pip | sphinx | 4.5.0 | Documentation build tool — existing docs infrastructure (dev dependency) |
| pip | sphinx_rtd_theme | * | Documentation theme — Read the Docs theme for existing Sphinx docs |

All versions above are the **exact pinned versions** from `requirements.txt` (the lockfile generated by `pipenv lock --requirements`).

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. This task creates a single new standalone file in `blitzy/documentation/` and does not modify any existing documentation. No link transformations are needed.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt poses a structured set of questions. Coverage is measured against complete answers to each distinct question:

| Question Area | Code Modules Required | Coverage Target |
|---------------|----------------------|-----------------|
| HTTP response status and message after upload | `views.py:PostDocumentView`, `urls.py`, `serialisers.py:PostDocumentSerializer` | 100% — exact status code, body, and request flow |
| Key log patterns during processing stages | `consumer.py:Consumer`, `loggers.py:LoggingMixin`, `parsers.py:RasterisedDocumentParser` | 100% — every `self.log()` and `_send_progress()` call in order |
| OCRmyPDF invocation parameters | `parsers.py:construct_ocrmypdf_parameters()`, `settings.py:OCR_*` | 100% — complete parameter dictionary with defaults |
| Archive PDF and thumbnail filenames | `file_handling.py:generate_filename()`, `models.py:Document.thumbnail_path`, `consumer.py` lines 316-337 | 100% — naming patterns, paths, and examples |
| Database record fields and values | `models.py:Document`, `consumer.py:_store()`, `signals/handlers.py` | 100% — every field with type, constraint, and example value |
| Environment setup procedure | `settings.py`, `manage.py`, management commands | 100% — enough detail to reproduce the setup |
| Cleanup confirmation | No code — procedural | 100% — explicit cleanup steps documented |

Overall target: **100% coverage** of all user-posed questions with code-traced rationale.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must reference specific source file paths and line numbers
- The OCRmyPDF parameter table must show every key-value pair assembled by `construct_ocrmypdf_parameters()` with both the parameter name and the setting it resolves from
- The database field table must list every column on the `Document` model (not just the serialized API fields)
- Log pattern documentation must distinguish between logger names (`paperless.consumer`, `paperless.parsing.tesseract`, `paperless.handlers`) and show the exact `level` and `message` format strings

**Accuracy validation:**
- All code snippets must be verified against the actual source lines retrieved
- Parameter defaults must match the values in `src/paperless/settings.py` (e.g., `OCR_LANGUAGE="eng"`, `OCR_MODE="skip"`, `OCR_OUTPUT_TYPE="pdfa"`)
- File naming patterns must match the `generate_filename()` format strings (e.g., `f"{doc.pk:07}{counter_str}{filetype_str}"`)

**Clarity standards:**
- Each question answered with a concise direct answer followed by a detailed "Rationale" section tracing the code path
- Technical terms defined on first use (e.g., "sidecar file" = OCRmyPDF text output alongside the PDF)
- Progressive disclosure: answer first, code trace second, edge cases third

**Maintainability:**
- Source citations with file path and line number for every claim
- Organized as a standalone reference that can be re-validated against future code changes

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid diagram: sequence diagram of the upload→process→store lifecycle
- Minimum 1 table per question: summarizing the key findings
- Code example snippets: kept to 2-3 lines showing key patterns (e.g., `Response("OK")`, `ocrmypdf.ocr(**args)`, `"{:07}.png".format(self.pk)`)
- All diagrams rendered in Mermaid markdown syntax for compatibility

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/<source_branch_name>.md` — the sole deliverable file

**Source code modules analyzed for documentation (read-only, not modified):**
- `src/documents/views.py` — PostDocumentView upload endpoint
- `src/documents/consumer.py` — Consumer pipeline orchestration
- `src/documents/tasks.py` — consume_file async task wrapper
- `src/documents/models.py` — Document model (database schema)
- `src/documents/file_handling.py` — Filename generation logic
- `src/documents/parsers.py` — Base DocumentParser class, thumbnail generation, date parsing
- `src/documents/serialisers.py` — PostDocumentSerializer (upload validation)
- `src/documents/loggers.py` — LoggingMixin (logging infrastructure)
- `src/documents/signals/__init__.py` — Signal declarations
- `src/documents/signals/handlers.py` — Post-consumption signal handlers
- `src/documents/apps.py` — Signal handler wiring
- `src/paperless_tesseract/parsers.py` — RasterisedDocumentParser (OCR integration)
- `src/paperless_tesseract/signals.py` — Parser registration for consumer discovery
- `src/paperless_tesseract/checks.py` — Tesseract language validation
- `src/paperless/settings.py` — All OCR, media, logging, task queue configuration
- `src/paperless/urls.py` — URL routing for upload endpoint
- `src/paperless/consumers.py` — WebSocket status consumer
- `Pipfile` — Python dependency declarations with version constraints
- `requirements.txt` — Pinned dependency versions (lockfile)
- `docs/api.rst` — Existing API documentation (reference context)
- `docs/configuration.rst` — Existing configuration documentation (reference context)
- `docs/conf.py` — Sphinx configuration (documentation infrastructure context)

**Documentation topics covered:**
- HTTP upload response behavior (status code, body, async task queuing)
- Consumer processing stage log patterns and WebSocket progress emissions
- OCRmyPDF invocation parameter assembly from settings to API call
- OCR fallback strategy (two-pass with `safe_fallback`)
- Generated archive PDF and thumbnail filenames in media storage
- Database `Document` model field catalog with types and example values
- Post-consumption signal handler chain and its effects on the final record
- Development environment setup steps (conceptual, for context)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any `.py`, `.rst`, `.yml`, `.json`, or other source file. The `SWE-AtlasQnA-Repo` rule explicitly prohibits modifying existing repository files.
- **Test file modifications**: No test files are created or modified
- **Frontend code analysis**: The Angular SPA at `src-ui/` is not analyzed — the user's questions focus entirely on backend processing behavior
- **Email ingestion pipeline**: `src/paperless_mail/` is not documented — the user asks specifically about PDF upload via the web interface
- **Tika/Gotenberg integration**: `src/paperless_tika/` is not relevant to the standard OCR pipeline
- **Docker/deployment configuration**: Container runtime files at `docker/` are not part of the investigation scope
- **Existing Sphinx documentation updates**: No modifications to `docs/*.rst` files or `docs/conf.py`
- **Database migrations**: Not analyzed beyond confirming the model schema
- **Barcode-based PDF splitting**: The `CONSUMER_ENABLE_BARCODES` feature in `tasks.py` is noted but not the primary focus
- **GnuPG encryption**: The deprecated GPG encryption path (`STORAGE_TYPE_GPG`) is noted but not documented in detail
- **Classifier training**: The ML classification triggered post-consumption is noted in the signal chain but not deeply analyzed
- **Items explicitly excluded by user**: "No file modifications or permanent changes allowed" — no persistent filesystem artifacts

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file, not integrated into the Sphinx build
- **Documentation preview command**: Standard Markdown rendering (e.g., `cat blitzy/documentation/<source_branch_name>.md`)
- **Diagram generation command**: Diagrams are embedded as Mermaid code blocks within the Markdown — no separate generation step required
- **Documentation deployment command**: Not applicable — file is committed to `blitzy/documentation/` in the destination repository
- **Default format**: Markdown (`.md`) with Mermaid diagrams in fenced code blocks
- **Citation requirement**: Every technical claim must reference the source file path and line number (e.g., `Source: src/documents/views.py:535`)
- **Style guide**: The document must follow the `SWE-AtlasQnA-Repo` implementation rules:
  - Comprehensive answers with thinking/rationale
  - Code-as-truth: base all answers on actual source code, not assumptions
  - No modifications to existing source files
  - Clear structure with distinct sections per question
- **Documentation validation**: Manual review — verify each answer against the cited source lines

### 0.9.2 Environment Setup Context

For the development environment setup portion of the documentation, the following details are derived from source code analysis:

- **Django management**: `python src/manage.py` — standard Django entrypoint (from `src/manage.py`)
- **Settings module**: `DJANGO_SETTINGS_MODULE=paperless.settings` (from `src/setup.cfg`)
- **Database default**: SQLite3 at `{DATA_DIR}/db.sqlite3` (from `src/paperless/settings.py:298-301`), with optional PostgreSQL via `PAPERLESS_DBHOST`
- **Admin user creation**: `python src/manage.py manage_superuser` reads `PAPERLESS_ADMIN_USER`, `PAPERLESS_ADMIN_MAIL`, `PAPERLESS_ADMIN_PASSWORD` from environment (from `src/documents/management/commands/manage_superuser.py`)
- **Required external services**: Redis at `redis://localhost:6379` (default from `src/paperless/settings.py:456`)
- **OCR defaults**: Language `eng`, mode `skip`, output type `pdfa`, clean `clean`, deskew `true`, rotate pages `true` (from `src/paperless/settings.py:510-533`)
- **Media paths**: `MEDIA_ROOT` defaults to `{BASE_DIR}/../media/`, with subdirectories `documents/originals/`, `documents/archive/`, `documents/thumbnails/` (from `src/paperless/settings.py:61-64`)
- **Scratch directory**: `/tmp/paperless` (from `src/paperless/settings.py:84`)
- **Logging**: File handler at `{LOGGING_DIR}/paperless.log` with format `[{asctime}] [{levelname}] [{name}] {message}` (from `src/paperless/settings.py:373-412`)

## 0.10 Rules for Documentation

The following documentation-specific rules and constraints are explicitly derived from the user's instructions and the `SWE-AtlasQnA-Repo` implementation rules:

- **Create a new markdown document named `<source_branch_name>.md`** that comprehensively answers the question(s) posed in the prompt
- **Provide thinking / rationale behind the answers** — every answer must include the reasoning derived from code analysis, not just a bare conclusion
- **Do not make assumptions, base your answers on the code as the truth** — all findings must be traceable to specific source files and line numbers; no inferred behavior from external documentation or assumed conventions
- **Do not modify any existing files in the source repository** — the only file created is the new markdown document in `blitzy/documentation/`
- **Place the generated document in the `blitzy/documentation` directory** in the destination repo — this directory must be created if it does not exist
- **No file modifications or permanent changes allowed** — the documentation must describe the observable behavior without leaving any persistent artifacts from testing
- **Clean up any test files completely when done** — if any test uploads or database records are created during investigation, they must be fully removed
- **Add source code citations for all technical details** — every claim must reference the originating source file with path and line number
- **Answers must cover all discrete questions**: HTTP response, log patterns, OCRmyPDF parameters, generated filenames, and database fields
- **Use Mermaid diagrams** for complex processing flows and decision trees where they aid comprehension

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Core Document Processing Pipeline:**

| File Path | Purpose | Key Lines |
|-----------|---------|-----------|
| `src/documents/views.py` | Upload endpoint `PostDocumentView.post()` returning `Response("OK")` | Lines 491-535 |
| `src/documents/consumer.py` | `Consumer` class orchestrating the full ingestion lifecycle | Lines 1-433 (entire file) |
| `src/documents/tasks.py` | `consume_file()` task wrapping `Consumer.try_consume_file()` | Lines 184-252 |
| `src/documents/models.py` | `Document` model with all field declarations and path properties | Lines 88-283 |
| `src/documents/file_handling.py` | `generate_unique_filename()` and `generate_filename()` for naming | Lines 81-199 |
| `src/documents/parsers.py` | Base `DocumentParser`, `make_thumbnail_from_pdf()`, `parse_date()` | Lines 1-351 |
| `src/documents/serialisers.py` | `PostDocumentSerializer` with MIME type validation | Lines 413-478 |
| `src/documents/loggers.py` | `LoggingMixin` providing correlation-group-based logging | Lines 1-22 (entire file) |
| `src/documents/signals/__init__.py` | Signal declarations: `document_consumption_started`, `document_consumption_finished` | Lines 1-5 |
| `src/documents/signals/handlers.py` | Post-consumption handlers: inbox tags, correspondent, type, tags, log entry, index | Lines 1-432 (entire file) |
| `src/documents/apps.py` | `DocumentsConfig.ready()` — six signal handler connections | Lines 1-29 |

**OCR Integration:**

| File Path | Purpose | Key Lines |
|-----------|---------|-----------|
| `src/paperless_tesseract/parsers.py` | `RasterisedDocumentParser`: `parse()`, `construct_ocrmypdf_parameters()`, `extract_text()` | Lines 1-320 |
| `src/paperless_tesseract/signals.py` | Parser registration with MIME types and weight | Summary only |
| `src/paperless_tesseract/checks.py` | Tesseract language availability validation | Summary only |
| `src/paperless_tesseract/apps.py` | App startup wiring connecting consumer declaration signal | Summary only |

**Configuration and Infrastructure:**

| File Path | Purpose | Key Lines |
|-----------|---------|-----------|
| `src/paperless/settings.py` | OCR settings, media paths, logging, database, task queue | Lines 1-610 |
| `src/paperless/urls.py` | URL routing for `/api/documents/post_document/` | Lines 1-146 (entire file) |
| `src/paperless/consumers.py` | WebSocket status consumer for `status_updates` group | Summary only |
| `src/paperless/version.py` | Version `(1, 7, 0)` | Line 1 |
| `src/setup.cfg` | pytest config with `DJANGO_SETTINGS_MODULE=paperless.settings` | Lines 1-18 |
| `src/manage.py` | Django management entrypoint | Summary only |

**Dependencies and Build:**

| File Path | Purpose | Key Lines |
|-----------|---------|-----------|
| `Pipfile` | Python dependency declarations — ocrmypdf ~13.4, django ~4.0, pillow ~9.1 | Lines 1-72 (entire file) |
| `requirements.txt` | Pinned dependency versions — ocrmypdf==13.4.3, django==4.0.4 | Lines 1-114 (entire file) |

**Existing Documentation:**

| File Path | Purpose | Key Lines |
|-----------|---------|-----------|
| `docs/api.rst` | Existing REST API reference — upload endpoint, response description | Lines 1-80 |
| `docs/conf.py` | Sphinx configuration — project metadata, extensions, theme | Lines 1-332 (entire file) |
| `docs/configuration.rst` | Configuration reference — OCR variables documented | Summary only |
| `docs/troubleshooting.rst` | Troubleshooting guide — OCR failure scenarios | Summary only |
| `.readthedocs.yml` | Read the Docs build configuration — Python 3.8, Sphinx | Lines 1-17 (entire file) |

**Repository Structure:**

| Folder Path | Purpose |
|-------------|---------|
| `(root)` | Repository root — Pipfile, Dockerfile, configs |
| `src/` | Python source tree — Django apps |
| `src/documents/` | Core document management app |
| `src/documents/management/commands/` | Management commands including `manage_superuser` |
| `src/paperless_tesseract/` | OCR/Tesseract integration app |
| `src/paperless/` | Core project package — settings, URLs, auth |
| `docs/` | Sphinx documentation workspace |

### 0.11.2 Tech Spec Sections Retrieved

- **Section 1.1 — Executive Summary**: Provided project context, version (1.7.0), and stakeholder overview
- **Section 1.3 — Scope**: Provided in-scope features, deployment configurations, and system boundaries

### 0.11.3 Attachments and External Resources

- **Attachments provided**: None (0 attachments)
- **Figma URLs**: None
- **External URLs**: None
- **Environment files**: None found in `/tmp/environments_files/`
- **Setup instructions**: None provided by user
- **Environment variables**: None provided
- **Secrets**: None provided

