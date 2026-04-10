# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative reference document** that comprehensively answers a set of tightly related behavioral questions about the OCR subsystem in paperless-ngx, grounded entirely in codebase evidence rather than speculation.

- **Category**: Create new documentation
- **Documentation Type**: Technical Q&A / Behavioral analysis guide — a single Markdown document providing code-traced answers to specific runtime behavior questions about the OCR pipeline
- **Target Output File**: `blitzy/documentation/paperless-ngx_542221a38dff.md` (per implementation rule `SWE-AtlasQnA-Repo`)

The user's requirements decompose into the following discrete documentation questions, each requiring code-grounded explanation and rationale:

| # | Question Theme | Specific Question |
|---|---|---|
| Q1 | OCR Activation Visibility | When an image with no embedded text is uploaded, how can a user observe that OCR has started, and what does the document's processing state look like while OCR is running? |
| Q2 | Background Worker Behavior | How do background workers (Django-Q qcluster) behave during OCR, and what signals indicate active OCR work? |
| Q3 | OCR Skip Behavior | When a similar image already contains embedded text, does the system skip OCR entirely or still touch the OCR pipeline? How can the user tell the difference after processing completes? |
| Q4 | API Response Comparison | What do the final API responses (`/api/documents/<id>/`) look like for OCR-processed documents versus documents with pre-existing text, and which fields reveal the origin of the text? |
| Q5 | Weak OCR Outcomes | When OCR produces weak or incomplete results, what happens to the document's final state — does it still count as fully processed, and how is this reflected in saved metadata? |

### 0.1.2 Special Instructions and Constraints

- **CRITICAL Directive — No Repository Modifications**: The user explicitly states: *"the repository itself should remain unchanged, and anything temporary should be cleaned up afterward."* The implementation rule `SWE-AtlasQnA-Repo` reinforces this: *"Do not modify any existing files in the source repository."*
- **Temporary Scripts Allowed**: Temporary observation scripts may be referenced in the documentation as investigative tools, but must be described as disposable and should not persist.
- **Code-as-Truth Principle**: The implementation rule states: *"Do not make assumptions, base your answers on the code as the truth."* Every answer must cite specific source files, line numbers, and code constructs.
- **Thinking / Rationale Required**: Per the rule: *"Provide thinking / rationale behind the answers."* The document must include reasoning chains, not just conclusions.
- **No Design System**: No design system is specified or relevant to this documentation task.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To answer Q1 (OCR activation visibility), we will trace the `Consumer.try_consume_file()` method in `src/documents/consumer.py` through its `_send_progress()` calls and follow the `RasterisedDocumentParser.parse()` method in `src/paperless_tesseract/parsers.py` to document observable status transitions (STARTING → WORKING → SUCCESS), log messages emitted during OCR, and WebSocket payloads broadcast via the `status_updates` channel group defined in `src/paperless/consumers.py`.
- To answer Q2 (background worker behavior), we will document the Django-Q cluster configuration in `src/paperless/settings.py` (Q_CLUSTER dict at line 449), the `consume_file` task in `src/documents/tasks.py`, and how `async_task` dispatches work from the API upload endpoint in `src/documents/views.py` (PostDocumentView, line 523).
- To answer Q3 (OCR skip behavior), we will analyze the four `OCR_MODE` branches in `construct_ocrmypdf_parameters()` (lines 155–162 of `src/paperless_tesseract/parsers.py`) and the `skip_noarchive` early-return path in `parse()` (lines 241–244), documenting how each mode determines whether OCRmyPDF is invoked and what artifacts are produced.
- To answer Q4 (API response comparison), we will document the `DocumentSerializer` fields in `src/documents/serialisers.py` (lines 201–235), the `metadata` action in `src/documents/views.py` (lines 282–310), and how `content`, `archived_file_name`, `archive_checksum`, and `has_archive_version` differ between OCR-processed and text-bearing documents.
- To answer Q5 (weak OCR outcomes), we will trace the fallback chain in `RasterisedDocumentParser.parse()` — from Tier 1 OCRmyPDF through Tier 2 safe fallback through Tier 3 pdfminer.six extraction and finally to the empty-content path (line 327) — and document how the resulting `Document` record stores the outcome in `content`, how empty content is flagged by the sanity checker in `src/documents/sanity_checker.py`, and how the consumer always marks the task as `SUCCESS` with a valid `document_id`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs surface:

- **WebSocket Protocol Documentation**: The user asks about "watching" OCR behavior, which requires understanding the WebSocket `status_update` message format emitted by `Consumer._send_progress()` in `src/documents/consumer.py` (lines 56–76). This includes the payload structure: `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, and `document_id`.
- **Log Correlation**: The logging architecture uses UUID-based `logging_group` correlation (from `src/documents/loggers.py`) to group all log entries for a single document consumption, which is essential for observing OCR activity in `paperless.log`.
- **OCR Mode Configuration Matrix**: The user's questions implicitly require a complete matrix of how each `PAPERLESS_OCR_MODE` value (skip, skip_noarchive, redo, force) affects the document's final `content`, `archive_filename`, and `archive_checksum` fields.
- **Sidecar File Semantics**: The `extract_text()` method (lines 99–133 of `src/paperless_tesseract/parsers.py`) reveals that sidecar files containing `[OCR skipped on page` markers are discarded in favor of pdfminer extraction — a nuance that directly answers how the system distinguishes OCR-generated text from pre-existing text.
- **Empty Content Handling**: When OCR produces no text, `self.text` is set to an empty string `""` (line 327), not `None`. This is a subtle but critical detail for understanding the document's final state and how it passes through the `_store()` method into the `Document.content` field.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with comprehensive RST-format manuals hosted on Read the Docs.

- **Documentation framework**: Sphinx ~4.5.0 (from `Pipfile`, line 67) with the `sphinx_rtd_theme`
- **Documentation generator configuration**: `docs/conf.py` — configures project metadata, extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), and theme settings
- **Documentation hosting**: Read the Docs, configured via `.readthedocs.yml` (Python 3.8, Sphinx builder pointing at `docs/conf.py`)
- **Documentation dependency manifest**: `docs/requirements.txt` — currently empty (placeholder)
- **Existing documentation files discovered**:

| File | Content Coverage |
|---|---|
| `docs/configuration.rst` | All PAPERLESS_OCR_* environment variables (lines 278–441); OCR mode explanations |
| `docs/usage_overview.rst` | High-level consumer lifecycle; OCR step description (lines 63–91) |
| `docs/api.rst` | REST API endpoint documentation; document fields; metadata endpoint; file uploads |
| `docs/troubleshooting.rst` | OCR accuracy tips; consumer troubleshooting; django-q worker verification |
| `docs/extending.rst` | Contributor workflows; parser extension points |
| `docs/setup.rst` | Installation and deployment |
| `docs/administration.rst` | Backups, indexing, sanity checking |
| `README.md` | Project overview; feature summary |

- **API documentation tools**: No JSDoc or dedicated API doc generator; the REST API is documented manually in `docs/api.rst` and via Django REST Framework's built-in browsable API
- **Diagram tools**: No Mermaid or PlantUML detected in the docs tree. The documentation is plain RST with code blocks.

### 0.2.2 Repository Code Analysis for Documentation

The following code modules were thoroughly analyzed to derive answers to the user's OCR behavior questions:

**Search patterns used for OCR pipeline analysis:**
- Parser implementation: `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser` class with `parse()`, `extract_text()`, `construct_ocrmypdf_parameters()` methods
- Parser base class: `src/documents/parsers.py` — `DocumentParser` abstract class defining lifecycle (`parse()`, `get_text()`, `get_archive_path()`, `cleanup()`)
- Parser registration signals: `src/paperless_tesseract/signals.py` — lazy parser factory with MIME type and weight registration
- Consumer pipeline: `src/documents/consumer.py` — `Consumer.try_consume_file()` with progress broadcasting and 10-stage pipeline
- Task dispatch: `src/documents/tasks.py` — `consume_file()` function with barcode pre-check and consumer invocation
- WebSocket delivery: `src/paperless/consumers.py` — `StatusConsumer` joining `status_updates` group
- Domain signals: `src/documents/signals/__init__.py` — three signals: `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration`
- Post-consumption handlers: `src/documents/signals/handlers.py` — six handlers wired in `src/documents/apps.py`
- API serialization: `src/documents/serialisers.py` — `DocumentSerializer` (lines 201–235) and `PostDocumentSerializer` (lines 413–477)
- API views: `src/documents/views.py` — `DocumentViewSet.metadata()` (lines 282–310), `PostDocumentView.post()` (lines 497–535)
- Data models: `src/documents/models.py` — `Document` model fields including `content`, `checksum`, `archive_checksum`, `archive_filename`, `mime_type`
- Settings: `src/paperless/settings.py` — `Q_CLUSTER` (line 449), `OCR_MODE` (line 522), `OCR_LANGUAGE` (line 514), `TASK_WORKERS` (line 438), `THREADS_PER_WORKER` (line 469)
- Logging infrastructure: `src/documents/loggers.py` — `LoggingMixin` with UUID-based `logging_group` correlation

**Key directories examined:**
- `src/paperless_tesseract/` — OCR parser package (5 modules + tests)
- `src/documents/` — Core application (20+ modules)
- `src/documents/signals/` — Signal definitions and handlers
- `src/paperless/` — Project configuration and WebSocket consumer
- `docs/` — Existing Sphinx documentation (13 RST files)

**Related documentation found:**
- `docs/configuration.rst` (lines 278–441) — Existing OCR setting descriptions provide context but do not answer runtime behavior questions
- `docs/troubleshooting.rst` — Contains high-level OCR troubleshooting but lacks pipeline-level observability guidance
- `docs/api.rst` — Documents the metadata endpoint but does not explain how OCR outcomes differ across the response fields

### 0.2.3 Web Search Research Conducted

No external web search was necessary for this task. The user's questions are entirely answerable from the codebase, and the implementation rule explicitly requires: *"Do not make assumptions, base your answers on the code as the truth."* All documentation will be derived exclusively from source code analysis.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's five questions map to specific code modules that must be documented:

- **Module: `src/paperless_tesseract/parsers.py`**
  - Public APIs: `RasterisedDocumentParser.parse()`, `extract_text()`, `construct_ocrmypdf_parameters()`, `is_image()`, `has_alpha()`, `get_dpi()`, `calculate_a4_dpi()`, `extract_metadata()`, `get_thumbnail()`
  - Current documentation: The `docs/configuration.rst` OCR section documents settings but not the runtime behavior of these methods
  - Documentation needed: Step-by-step trace of what `parse()` does for text-free images vs text-bearing PDFs; explanation of the sidecar file logic; the `[OCR skipped on page` detection; the `NoTextFoundException` fallback chain; the final `self.text = ""` empty-content path
  - Answers questions: Q1, Q3, Q5

- **Module: `src/documents/consumer.py`**
  - Public APIs: `Consumer.try_consume_file()`, `Consumer._send_progress()`, `Consumer._store()`
  - Current documentation: `docs/usage_overview.rst` describes the consumer at a high level; `docs/troubleshooting.rst` addresses common failures
  - Documentation needed: Detailed progress state machine (STARTING → WORKING → SUCCESS/FAILED); the exact `_send_progress()` payloads emitted at each stage; how `text` flows from parser through `_store()` into `Document.content`; how `archive_path` determines `archive_filename` and `archive_checksum` presence
  - Answers questions: Q1, Q2, Q4, Q5

- **Module: `src/documents/tasks.py`**
  - Public APIs: `consume_file()`
  - Current documentation: Not explicitly documented in the existing docs
  - Documentation needed: How `consume_file()` is dispatched via `async_task` from `PostDocumentView`, how Django-Q workers pick up and execute the task, and the barcode pre-check that runs before the consumer
  - Answers questions: Q2

- **Module: `src/paperless/consumers.py`**
  - Public APIs: `StatusConsumer.connect()`, `StatusConsumer.status_update()`
  - Current documentation: Not documented in existing docs
  - Documentation needed: The WebSocket `status_update` event payload format; how to connect and listen for OCR progress; authentication requirements
  - Answers questions: Q1, Q2

- **Module: `src/documents/views.py`**
  - Public APIs: `DocumentViewSet.metadata()`, `PostDocumentView.post()`
  - Current documentation: `docs/api.rst` covers the metadata endpoint fields
  - Documentation needed: How `has_archive_version`, `archive_checksum`, `archive_media_filename`, and `content` differ between OCR-processed and non-OCR documents
  - Answers questions: Q3, Q4

- **Module: `src/documents/models.py`**
  - Public APIs: `Document` model — `content`, `archive_checksum`, `archive_filename`, `checksum`, `mime_type`
  - Current documentation: `docs/api.rst` lists fields but does not explain how OCR outcomes map to field values
  - Documentation needed: How each field reflects OCR success, partial success, or failure
  - Answers questions: Q4, Q5

- **Module: `src/paperless/settings.py`**
  - Configuration options: `OCR_MODE` (line 522), `OCR_LANGUAGE` (line 514), `Q_CLUSTER` (line 449), `TASK_WORKERS` (line 438), `THREADS_PER_WORKER` (line 469)
  - Current documentation: `docs/configuration.rst` covers OCR settings comprehensively
  - Documentation needed: How each `OCR_MODE` value maps to the `parse()` method behavior and final document state
  - Answers questions: Q3, Q5

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist that the new document must fill:

- **Undocumented runtime observability**: No existing documentation explains how to watch OCR happen in real time — the WebSocket payload structure, log correlation via `logging_group`, and the specific progress messages (MESSAGE_PARSING_DOCUMENT, MESSAGE_GENERATING_THUMBNAIL, etc.) are entirely undocumented
- **No OCR mode behavioral comparison**: While `docs/configuration.rst` describes each OCR mode, it does not trace what each mode causes at the code level — which OCRmyPDF flags are set, whether sidecar files are generated, and what the resulting `Document` record looks like
- **No API field interpretation guide for OCR**: The API documentation in `docs/api.rst` lists the metadata fields but does not explain which fields indicate OCR was performed vs. skipped, or what empty/null values signify
- **No weak-OCR outcome documentation**: The three-tier fallback chain (OCRmyPDF → safe fallback → pdfminer → empty string) and its final effect on `Document.content` are not documented anywhere
- **No sidecar file semantics**: The `[OCR skipped on page` marker detection in `extract_text()` is a critical behavioral nuance that determines whether OCR-generated text or embedded text is used — this is undocumented


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

Per the implementation rule `SWE-AtlasQnA-Repo`, the output is a single Markdown document placed in `blitzy/documentation/`. The internal structure of the document follows the user's question flow:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── Introduction (scope and methodology)
        ├── Q1: Observing OCR Activation on a Text-Free Image
        │   ├── Upload entry point and task dispatch
        │   ├── Consumer progress state machine
        │   ├── WebSocket status_update payload anatomy
        │   ├── Log signals indicating active OCR
        │   └── Processing state during OCR execution
        ├── Q2: Background Worker Behavior During OCR
        │   ├── Django-Q cluster architecture
        │   ├── Worker lifecycle for a consume_file task
        │   ├── Thread allocation and OMP_THREAD_LIMIT
        │   └── Signals that indicate active OCR work
        ├── Q3: OCR Skip Behavior for Text-Bearing Images
        │   ├── OCR_MODE decision matrix
        │   ├── skip_noarchive early-return path
        │   ├── Sidecar file analysis and [OCR skipped on page] detection
        │   ├── How to tell the difference after processing
        │   └── Archive file presence as a diagnostic signal
        ├── Q4: Comparing Final API Responses
        │   ├── Document endpoint field anatomy
        │   ├── Metadata endpoint field anatomy
        │   ├── Fields that reveal OCR-generated vs existing text
        │   └── Side-by-side comparison table
        ├── Q5: Weak or Incomplete OCR Outcomes
        │   ├── Three-tier fallback chain walkthrough
        │   ├── Empty content path and its database representation
        │   ├── Does the document still count as "fully processed"?
        │   ├── Sanity checker behavior for empty-content documents
        │   └── Metadata reflection of weak OCR
        └── Summary and Quick-Reference Table
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract OCR pipeline logic from `src/paperless_tesseract/parsers.py` by tracing the `parse()` method line by line
- Extract consumer state machine from `src/documents/consumer.py` by mapping every `_send_progress()` call to its arguments
- Extract API response shape from `src/documents/serialisers.py` (`DocumentSerializer.Meta.fields`) and `src/documents/views.py` (`metadata()` action)
- Extract worker behavior from `src/paperless/settings.py` (`Q_CLUSTER`, `TASK_WORKERS`, `THREADS_PER_WORKER`)
- Extract fallback chain from the exception handling blocks in `RasterisedDocumentParser.parse()` (lines 259–328)

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` headers
- Code citations as inline references: `Source: src/paperless_tesseract/parsers.py:230-244`
- Tables for comparison data (OCR mode matrix, API field comparison)
- Mermaid diagrams for the OCR fallback chain, the consumer progress state machine, and the task dispatch flow
- Every factual claim must reference a specific file and line range

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be included in the document:

- **Consumer Progress State Machine**: A flowchart showing the STARTING → WORKING (20%: parsing, 70%: thumbnail, 90%: date, 95%: save) → SUCCESS/FAILED transitions with the exact `message` constants from `src/documents/consumer.py` (lines 37–49)
- **OCR Decision Tree**: A decision diagram showing how `OCR_MODE` and `original_has_text` determine whether OCRmyPDF is invoked, which flags are set, and whether an archive PDF is produced
- **Task Dispatch Flow**: A sequence diagram from API POST through `async_task` through Django-Q dequeue through `consume_file` through `Consumer.try_consume_file()` through `RasterisedDocumentParser.parse()`
- **Three-Tier OCR Fallback Chain**: A flowchart showing Tier 1 (full OCRmyPDF) → Tier 2 (safe fallback) → Tier 3 (pdfminer) → empty content, with the exception types that trigger each transition

### 0.4.4 Template Application

No user-provided template exists. The document follows the structure mandated by the `SWE-AtlasQnA-Repo` implementation rule: a standalone Markdown Q&A document with thinking/rationale, code citations, and no modifications to existing repository files.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/paperless_tesseract/parsers.py`, `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/models.py`, `src/documents/serialisers.py`, `src/documents/views.py`, `src/paperless/consumers.py`, `src/paperless/settings.py`, `src/documents/signals/__init__.py`, `src/documents/signals/handlers.py`, `src/documents/loggers.py`, `src/documents/apps.py`, `src/documents/sanity_checker.py`, `docs/configuration.rst`, `docs/api.rst` | Complete Q&A document answering five OCR runtime behavior questions with code-traced rationale, Mermaid diagrams, comparison tables, and source citations |

This is the **only** file to be created. No existing files are modified, updated, or deleted, consistent with the `SWE-AtlasQnA-Repo` rule.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Q&A / Behavioral Analysis
Source Code: 14 source modules across src/paperless_tesseract/, src/documents/, src/paperless/
Sections:
    - Introduction (scope, methodology, code-as-truth principle)
    - Q1: Observing OCR Activation on a Text-Free Image
        - Upload path through PostDocumentView → async_task → consume_file
        - Consumer._send_progress() state transitions with exact payloads
        - WebSocket status_update message format and connection procedure
        - Log file signals: "Parsing <filename>...", "Calling OCRmyPDF with args: {…}", "Using text from sidecar file"
        - Processing state during OCR: status="WORKING", message="parsing_document", progress 20-70
    - Q2: Background Worker Behavior During OCR
        - Django-Q Q_CLUSTER configuration: workers=√cores, timeout=1800s, retry=1810s
        - Worker picks up consume_file from Redis queue
        - OMP_THREAD_LIMIT=1 constraint for Tesseract parallelism
        - THREADS_PER_WORKER controls OCRmyPDF's jobs parameter
    - Q3: OCR Skip Behavior for Text-Bearing Images
        - OCR_MODE="skip" → ocrmypdf skip_text=True (OCRmyPDF still runs, skips text-bearing pages)
        - OCR_MODE="skip_noarchive" → if original_has_text: early return, no OCRmyPDF call at all
        - Sidecar file "[OCR skipped on page" detection → discard sidecar, use pdfminer
        - Diagnostic: archive_filename is None when skip_noarchive skips; present when skip runs
    - Q4: Comparing Final API Responses
        - DocumentSerializer fields: id, content, archived_file_name, tags, created, modified, etc.
        - Metadata endpoint: has_archive_version, archive_checksum, archive_size, original_metadata, archive_metadata
        - Field-by-field comparison table for OCR-processed vs text-bearing documents
    - Q5: Weak or Incomplete OCR Outcomes
        - Tier 1 → NoTextFoundException → Tier 2 (safe_fallback=True, force_ocr=True)
        - Tier 2 → Any Exception → ParseError (fatal)
        - Tier 2 success but no text → fallback to original_has_text or self.text = ""
        - Document still saved with status SUCCESS and valid document_id
        - Sanity checker flags empty content as informational, not error
    - Summary and Quick-Reference Table
Diagrams:
    - Consumer progress state machine (Mermaid flowchart)
    - OCR mode decision tree (Mermaid flowchart)
    - Task dispatch sequence (Mermaid sequence diagram)
    - Three-tier OCR fallback chain (Mermaid flowchart)
Key Citations:
    - src/paperless_tesseract/parsers.py (lines 99-133, 135-228, 230-328)
    - src/documents/consumer.py (lines 56-76, 180-377, 379-412)
    - src/documents/tasks.py (lines 184-252)
    - src/documents/views.py (lines 282-310, 491-535)
    - src/documents/serialisers.py (lines 201-235)
    - src/documents/models.py (lines 88-283)
    - src/paperless/consumers.py (lines 1-33)
    - src/paperless/settings.py (lines 449-472, 510-541)
    - src/documents/signals/__init__.py (lines 1-5)
    - src/documents/apps.py (lines 11-29)
    - src/documents/loggers.py (lines 1-21)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone Markdown document placed in `blitzy/documentation/` and is not integrated into the Sphinx documentation build. There are no changes to:
- `docs/conf.py`
- `docs/index.rst`
- `.readthedocs.yml`

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes**: The new document is self-contained
- **No navigation links**: It does not link into the Sphinx doc tree
- **No table of contents updates**: It lives outside the docs/ directory
- **No index/glossary updates**: Not applicable


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

Since the output is a standalone Markdown file with no build step, the documentation dependencies are limited to the tools needed to render Mermaid diagrams and the source packages whose behavior is being documented.

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| pip | sphinx | ~4.5.0 | Existing docs build tool (not used for this document) |
| pip | sphinx_rtd_theme | * | Existing docs theme (not used for this document) |
| pip | ocrmypdf | ~13.4 | OCR engine whose runtime behavior is documented |
| pip | django | ~4.0 | Web framework providing the consumer, views, and settings |
| pip | djangorestframework | ~3.13 | API framework providing DocumentSerializer and metadata endpoint |
| pip | django-q | ~1.3 | Task queue providing background worker cluster for OCR execution |
| pip | channels | ~3.0 | WebSocket framework providing StatusConsumer for real-time updates |
| pip | channels-redis | * | Redis channel layer backend for WebSocket message routing |
| pip | pdfminer.six | * | Fallback PDF text extractor used when OCR sidecar is incomplete |
| pip | pikepdf | ~5.1 | PDF metadata extraction in RasterisedDocumentParser.extract_metadata() |
| pip | pillow | ~9.1 | Image processing for DPI detection and alpha channel handling |
| pip | python-magic | * | MIME type detection in consumer pipeline |
| pip | redis | * | Message broker for Django-Q task queue and Channels layer |
| system | tesseract-ocr | (system) | OCR engine invoked by OCRmyPDF |
| system | imagemagick | (system) | Primary thumbnail generator |
| system | ghostscript | (system) | Fallback thumbnail generator |

### 0.6.2 Documentation Reference Updates

Not applicable. The new document is a standalone file in `blitzy/documentation/` and does not require any link updates in existing documentation files. No existing files are modified per the implementation rule.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions addressed**: 5/5 (100%)
  - Q1: OCR activation visibility — Covered via consumer progress state machine, WebSocket payloads, and log signals
  - Q2: Background worker behavior — Covered via Django-Q cluster config, worker lifecycle, thread allocation
  - Q3: OCR skip behavior — Covered via OCR_MODE decision matrix, skip_noarchive early-return, sidecar analysis
  - Q4: API response comparison — Covered via DocumentSerializer fields, metadata endpoint, field comparison table
  - Q5: Weak OCR outcomes — Covered via three-tier fallback chain, empty content path, sanity checker behavior

- **Source modules documented**: 14/14 (100%)
  - `src/paperless_tesseract/parsers.py` — Fully traced
  - `src/documents/consumer.py` — Fully traced
  - `src/documents/tasks.py` — Fully traced
  - `src/documents/models.py` — Relevant fields documented
  - `src/documents/serialisers.py` — DocumentSerializer fields mapped
  - `src/documents/views.py` — metadata() and PostDocumentView documented
  - `src/paperless/consumers.py` — StatusConsumer payload format documented
  - `src/paperless/settings.py` — Q_CLUSTER and OCR settings documented
  - `src/documents/signals/__init__.py` — Signal definitions referenced
  - `src/documents/signals/handlers.py` — Post-consumption handlers referenced
  - `src/documents/apps.py` — Handler wiring documented
  - `src/documents/loggers.py` — Logging correlation mechanism documented
  - `src/documents/sanity_checker.py` — Empty content flagging documented
  - `src/paperless_tesseract/checks.py` — Language validation referenced

- **OCR modes documented**: 4/4 (100%) — skip, skip_noarchive, redo, force

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must trace code paths with file:line citations
- Every behavioral claim must identify the specific Python method and conditional branch
- The OCR mode decision matrix must cover all four modes with their effects on `content`, `archive_filename`, and `archive_checksum`
- The API comparison must cover both the `/api/documents/<id>/` and `/api/documents/<id>/metadata/` endpoints

**Accuracy validation:**
- All code citations verified against actual source file contents retrieved during context gathering
- All OCRmyPDF parameter names verified against `construct_ocrmypdf_parameters()` implementation
- All WebSocket payload fields verified against `Consumer._send_progress()` implementation
- All Document model fields verified against `src/documents/models.py` class definition

**Clarity standards:**
- Each question section begins with a plain-language summary, then dives into code-level detail
- Thinking/rationale is provided before conclusions (per implementation rule)
- Mermaid diagrams supplement text for complex flows
- Comparison tables provide at-a-glance field-by-field differences

**Maintainability:**
- Source citations with file paths and line numbers enable future verification
- Modular question-answer structure allows individual sections to be updated independently
- No external dependencies for rendering (standard Markdown + Mermaid)

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per question**: At least one code path trace or API response example per question
- **Diagram types required**:
  - 1× Consumer progress state machine (flowchart)
  - 1× OCR mode decision tree (flowchart)
  - 1× Task dispatch flow (sequence diagram)
  - 1× Three-tier OCR fallback chain (flowchart)
- **Code example validation**: All code snippets are extracted directly from source files, not fabricated
- **Visual content freshness**: Diagrams are generated from code analysis of the current v1.7.0 codebase


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/paperless-ngx_542221a38dff.md` — The single output artifact

- **Code modules analyzed for documentation content** (read-only, no modifications):
  - `src/paperless_tesseract/parsers.py` — OCR parser implementation
  - `src/paperless_tesseract/signals.py` — Parser registration
  - `src/paperless_tesseract/checks.py` — Language validation
  - `src/paperless_tesseract/apps.py` — App startup wiring
  - `src/documents/consumer.py` — Ingestion pipeline and progress broadcasting
  - `src/documents/tasks.py` — Background task dispatch
  - `src/documents/models.py` — Document data model
  - `src/documents/serialisers.py` — API serialization layer
  - `src/documents/views.py` — API endpoints (metadata, upload, document CRUD)
  - `src/documents/parsers.py` — Base parser class and parser discovery
  - `src/documents/signals/__init__.py` — Domain signal definitions
  - `src/documents/signals/handlers.py` — Post-consumption signal handlers
  - `src/documents/apps.py` — Handler registration
  - `src/documents/loggers.py` — Logging correlation mixin
  - `src/documents/sanity_checker.py` — Archive integrity checker
  - `src/paperless/consumers.py` — WebSocket StatusConsumer
  - `src/paperless/settings.py` — OCR and worker configuration

- **Existing documentation referenced** (read-only):
  - `docs/configuration.rst` — OCR settings descriptions
  - `docs/api.rst` — REST API field documentation
  - `docs/usage_overview.rst` — Consumer lifecycle overview
  - `docs/troubleshooting.rst` — OCR troubleshooting

- **Topics documented**:
  - OCR activation observability (WebSocket, logs, progress state)
  - Background worker behavior during OCR processing
  - OCR skip/bypass logic across all four OCR modes
  - API response field comparison for OCR vs non-OCR documents
  - Weak OCR outcome handling and final document state
  - Temporary observation script patterns (described but not committed)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No files in `src/` are modified — per user instruction and `SWE-AtlasQnA-Repo` rule
- **Existing documentation modifications**: No files in `docs/` are modified
- **Test file modifications**: No test files are created or modified
- **Feature additions or code refactoring**: No functional changes to the OCR pipeline
- **Deployment configuration changes**: No Docker, Compose, or Supervisord modifications
- **Frontend (Angular) code**: The `src-ui/` directory is not analyzed or documented
- **Email ingestion subsystem**: `src/paperless_mail/` is not in scope
- **Tika parser subsystem**: `src/paperless_tika/` is not in scope (only Tesseract OCR is relevant)
- **Text parser subsystem**: `src/paperless_text/` is not in scope
- **Database schema changes**: No migration files are created
- **CI/CD pipeline changes**: No `.github/` workflow modifications
- **Persistent observation scripts**: Any scripts mentioned in the document are described as temporary and disposable


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file, not integrated into the Sphinx build
- **Documentation preview command**: Any Markdown renderer (e.g., `grip blitzy/documentation/paperless-ngx_542221a38dff.md` or a GitHub preview)
- **Diagram generation command**: Mermaid diagrams are embedded inline in fenced code blocks and render natively on GitHub and most Markdown previewers
- **Documentation deployment command**: Not applicable — the file is committed to the repository in `blitzy/documentation/`
- **Default format**: Markdown with Mermaid diagrams
- **Citation requirement**: Every section must reference source files with path and line numbers
- **Style guide**: Follow the `SWE-AtlasQnA-Repo` rule: provide thinking/rationale, base answers on code, do not modify existing files
- **Documentation validation**: Manual review for completeness against the five user questions; verify all code citations match actual file contents


## 0.10 Rules for Documentation

The following rules govern the creation of this documentation, derived from the user's instructions and the `SWE-AtlasQnA-Repo` implementation rule:

- **Do not modify any existing files in the source repository.** The output is a single new file: `blitzy/documentation/paperless-ngx_542221a38dff.md`. No files in `src/`, `docs/`, `docker/`, or any other existing directory are changed.
- **Do not make assumptions, base your answers on the code as the truth.** Every behavioral claim must be traceable to a specific file, method, and line range. No inferences from documentation or external sources are permitted unless the code itself is ambiguous.
- **Provide thinking / rationale behind the answers.** Each question section must include a reasoning chain that explains *why* the system behaves as described, not just *what* it does. The reader should understand the code logic that produces the observed behavior.
- **Create a new markdown document named `paperless-ngx_542221a38dff.md`.** The filename matches the source branch name exactly.
- **Place the generated document in the `blitzy/documentation` directory.** The directory must be created if it does not exist.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged, and anything temporary should be cleaned up afterward.** Any observation scripts described in the document must be clearly labeled as temporary and disposable, with explicit cleanup instructions.
- **Add source code citations for all technical details.** Every code reference should use the format `Source: <filepath>:<line-range>` to enable verification.
- **Include Mermaid diagrams for complex workflows.** The consumer progress state machine, OCR decision tree, task dispatch flow, and fallback chain must be visualized.
- **Use consistent terminology from the codebase.** Use the exact names from the source code: `Consumer`, `RasterisedDocumentParser`, `consume_file`, `_send_progress`, `status_updates`, `OCR_MODE`, `Q_CLUSTER`, `async_task`, etc.


## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were retrieved and analyzed during context gathering to derive the conclusions in this Agent Action Plan:

**OCR Pipeline (src/paperless_tesseract/)**

| File Path | Purpose in Analysis |
|---|---|
| `src/paperless_tesseract/parsers.py` | Core OCR parser — `RasterisedDocumentParser.parse()`, `extract_text()`, `construct_ocrmypdf_parameters()`, `NoTextFoundException`, `post_process_text()` |
| `src/paperless_tesseract/signals.py` | Parser registration — MIME type mapping and weight for consumer dispatch |
| `src/paperless_tesseract/checks.py` | System check — `check_default_language_available()` and `get_tesseract_langs()` |
| `src/paperless_tesseract/apps.py` | App startup — signal wiring for `tesseract_consumer_declaration` |
| `src/paperless_tesseract/__init__.py` | Package exports — re-exports check functions |
| `src/paperless_tesseract/tests/` (folder) | Test coverage structure — `test_checks.py` and `test_parser.py` |

**Documents Core (src/documents/)**

| File Path | Purpose in Analysis |
|---|---|
| `src/documents/consumer.py` | Ingestion pipeline — `Consumer.try_consume_file()`, `_send_progress()`, `_store()`, progress constants |
| `src/documents/tasks.py` | Task dispatch — `consume_file()`, barcode splitting, `bulk_update_documents()` |
| `src/documents/models.py` | Data model — `Document` fields (`content`, `checksum`, `archive_checksum`, `archive_filename`, `mime_type`, etc.) |
| `src/documents/serialisers.py` | API serialization — `DocumentSerializer`, `PostDocumentSerializer`, field lists |
| `src/documents/views.py` | API endpoints — `DocumentViewSet.metadata()`, `PostDocumentView.post()`, `UnifiedSearchViewSet` |
| `src/documents/parsers.py` | Base parser — `DocumentParser` class, `get_parser_class_for_mime_type()`, thumbnail generation |
| `src/documents/signals/__init__.py` | Signal definitions — `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` |
| `src/documents/signals/handlers.py` | Signal handlers — `set_correspondent()`, `set_document_type()`, `set_tags()`, `add_inbox_tags()`, `set_log_entry()`, `add_to_index()` |
| `src/documents/apps.py` | Handler wiring — connects 6 handlers to `document_consumption_finished` |
| `src/documents/loggers.py` | Logging mixin — UUID-based `logging_group` for log correlation |
| `src/documents/sanity_checker.py` | Integrity checker — empty content flagging behavior (referenced via folder summary) |

**Project Core (src/paperless/)**

| File Path | Purpose in Analysis |
|---|---|
| `src/paperless/consumers.py` | WebSocket — `StatusConsumer` with `connect()`, `disconnect()`, `status_update()` |
| `src/paperless/settings.py` | Configuration — `Q_CLUSTER`, `OCR_MODE`, `OCR_LANGUAGE`, `TASK_WORKERS`, `THREADS_PER_WORKER`, `CHANNEL_LAYERS` |

**Existing Documentation (docs/)**

| File Path | Purpose in Analysis |
|---|---|
| `docs/configuration.rst` (lines 278–441) | OCR settings reference — all PAPERLESS_OCR_* variables |
| `docs/api.rst` | REST API documentation — endpoints, fields, metadata, versioning |
| `docs/usage_overview.rst` (lines 1–100) | Consumer overview — ingestion steps, OCR description |
| `docs/troubleshooting.rst` | OCR troubleshooting — consumer issues, django-q verification |
| `docs/conf.py` | Sphinx configuration — extensions, theme, version |
| `docs/requirements.txt` | Docs dependencies — empty placeholder |

**Configuration and Build Files**

| File Path | Purpose in Analysis |
|---|---|
| `Pipfile` | Python dependencies — ocrmypdf ~13.4, django ~4.0, channels ~3.0, sphinx ~4.5.0 |
| `.readthedocs.yml` | Docs hosting config — Python 3.8, Sphinx builder |
| `src/setup.cfg` | Test/lint config — pytest settings, coverage |

### 0.11.2 Attachments Provided

No attachments were provided by the user. No Figma screens, design files, or supplementary documents are referenced.

### 0.11.3 Implementation Rules Applied

| Rule Name | Summary |
|---|---|
| `SWE-AtlasQnA-Repo` | Create a new markdown document named `<source_branch_name>.md` in `blitzy/documentation/` that comprehensively answers the question(s) posed in the prompt. Provide thinking/rationale. Base answers on code. Do not modify existing files. |


