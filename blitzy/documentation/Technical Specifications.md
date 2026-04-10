# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a comprehensive, code-grounded explanation of the paperless-ngx document processing lifecycle, metadata model, and organizational taxonomy.

**Documentation Type:** Technical Architecture / System Internals Guide

The user's request decomposes into four distinct documentation requirements:

- **Document Ingestion Flow** — Trace and explain every pathway by which a document enters paperless-ngx, from the initial file detection or API upload through to the final persisted state. Source entry points include directory watching (`src/documents/management/commands/document_consumer.py`), REST API upload (`src/documents/views.py:PostDocumentView`), and IMAP email ingestion (`src/paperless_mail/mail.py`).
- **Processing Pipeline Stages** — Describe each stage a document traverses: validation, MIME detection, parser dispatch, OCR / text extraction, date parsing, thumbnail generation, classification, atomic persistence, search indexing, and post-consume hooks. Key source: `src/documents/consumer.py:Consumer.try_consume_file()`.
- **Background Job Architecture** — Document the role of Django-Q as the task broker backed by Redis, including scheduled tasks (classifier training, index optimization, sanity checks, email fetching), the `consume_file` task entry point in `src/documents/tasks.py`, and how `async_task` dispatches work.
- **Metadata Model & Organizational Taxonomy** — Catalog every field on the `Document` model (`src/documents/models.py`), classify each as required versus optional versus runtime-derived, illustrate with a concrete runtime example, and explain how `Tag`, `Correspondent`, and `DocumentType` are used together for document organization via rule-based and ML-powered matching (`src/documents/matching.py`, `src/documents/classifier.py`).

**Category:** Create new documentation

### 0.1.2 Special Instructions and Constraints

- **No repository modifications:** The user explicitly stated *"Please don't make any changes to the repository itself."* The documentation output is a new file placed in `blitzy/documentation/` per the project's implementation rules.
- **Temporary scripts allowed:** If any scripts are needed for testing, they must be cleaned up afterward.
- **Runtime example requested:** The user specifically asked *"Can you show with a runtime example?"* — the documentation must include a concrete, realistic JSON or Python-dict example of a `Document` record illustrating required vs. optional vs. derived fields.
- **Practical organizational explanation:** The user asked *"how things like tags, correspondents, and document types are used together to organize documents in a practical way"* — the documentation should include a real-world scenario, not just abstract definitions.
- **Output file name:** Per the implementation rules, the file must be named `paperless-ngx_542221a38dff.md` and placed in the `blitzy/documentation/` directory.
- **Answers must be code-grounded:** Per the implementation rule *"Do not make assumptions, base your answers on the code as the truth."*

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **ingestion flow**, we will **create** a section in `blitzy/documentation/paperless-ngx_542221a38dff.md` that traces the three entry points (`document_consumer.py` → directory watcher, `views.py:PostDocumentView` → REST API, `paperless_mail/mail.py` → IMAP fetch) through the Django-Q `async_task` dispatch into `documents.tasks.consume_file`, which instantiates the `Consumer` class.
- To document the **processing stages**, we will **create** a detailed walkthrough of `Consumer.try_consume_file()` in `src/documents/consumer.py`, covering pre-checks, parser dispatch via signals (`document_consumer_declaration`), text/date/thumbnail extraction, classification via `load_classifier()` and signal handlers (`set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`), and atomic DB+file persistence.
- To document the **background job system**, we will **create** a section explaining the Django-Q cluster configuration in `src/paperless/settings.py:Q_CLUSTER`, the scheduled tasks registered via migrations (`1001`, `1004`), and the ad-hoc tasks dispatched by `async_task` throughout the codebase.
- To document the **metadata model**, we will **create** a field-by-field catalog of `Document` in `src/documents/models.py`, including its ForeignKey relationships to `Correspondent`, `DocumentType`, and the ManyToMany relationship to `Tag`, with a runtime JSON example.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Parser Plugin Architecture** — The `document_consumer_declaration` signal pattern (`src/documents/signals/__init__.py`) is a plugin system allowing `paperless_tesseract`, `paperless_text`, and `paperless_tika` to register parsers with MIME type mappings and weights. This mechanism should be briefly documented to explain how parser dispatch works.
- **Matching Algorithm Reference** — The six matching algorithms (Any, All, Literal, Regex, Fuzzy, Auto/ML) defined in `MatchingModel` (`src/documents/models.py`) and implemented in `src/documents/matching.py` are central to how tags, correspondents, and document types are assigned. A brief taxonomy is needed.
- **File Naming and Storage Layout** — The `generate_filename()` function in `src/documents/file_handling.py` uses `PAPERLESS_FILENAME_FORMAT` with template variables (`{title}`, `{correspondent}`, `{document_type}`, `{created}`, `{tags}`, etc.). This explains how metadata directly influences file organization on disk.
- **Search Index Schema** — The Whoosh schema in `src/documents/index.py:get_schema()` indexes `title`, `content`, `correspondent`, `tag`, `type`, `created`, `modified`, `added`, and `asn`, which is directly relevant to understanding what metadata is queryable at runtime.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** hosted on ReadTheDocs, with reStructuredText (`.rst`) source files.

- **Documentation framework:** Sphinx ~4.5.0 (from `Pipfile` dev-packages)
- **Theme:** `sphinx_rtd_theme` (ReadTheDocs theme, as configured in `docs/conf.py`)
- **Configuration file:** `docs/conf.py`
- **Hosting configuration:** `.readthedocs.yml` (Python 3.8, Sphinx builder)
- **Documentation requirements:** `docs/requirements.txt` — empty file (theme installed via Pipfile)
- **API documentation tools:** None detected (no JSDoc, autodoc, or Swagger configs)
- **Diagram tools:** None currently in the documentation (no Mermaid or PlantUML configuration)
- **Project version documented:** 1.7.0 (from `src/paperless/version.py`)

**Existing documentation files discovered:**

| File | Format | Content Summary |
|------|--------|-----------------|
| `docs/index.rst` | RST | Root index with project overview, links to all doc sections |
| `docs/usage_overview.rst` | RST | Terms/definitions, adding documents, consumption directory, email, recommended workflow |
| `docs/advanced_usage.rst` | RST | Matching algorithms, automatic matching, file naming |
| `docs/configuration.rst` | RST | All configuration options for the application |
| `docs/setup.rst` | RST | Installation and setup instructions |
| `docs/api.rst` | RST | REST API documentation |
| `docs/administration.rst` | RST | Administration and management commands |
| `docs/extending.rst` | RST | Parser plugin extension documentation |
| `docs/faq.rst` | RST | Frequently asked questions |
| `docs/troubleshooting.rst` | RST | Common issues and resolutions |
| `docs/changelog.rst` | RST | Version history |
| `docs/scanners.rst` | RST | Scanner hardware recommendations |
| `docs/screenshots.rst` | RST | UI screenshots |
| `README.md` | Markdown | Project overview, quick start, features list |
| `CONTRIBUTING.md` | Markdown | Contribution guidelines, team structure |
| `CODE_OF_CONDUCT.md` | Markdown | Community code of conduct |

### 0.2.2 Repository Code Analysis for Documentation

The following source modules were examined to extract information for the documentation deliverable:

**Document Ingestion Pipeline:**
- `src/documents/management/commands/document_consumer.py` — Directory watcher using inotify/polling, file validation, `async_task` dispatch
- `src/documents/views.py:PostDocumentView` — REST API upload endpoint at `POST /api/documents/post_document/`
- `src/paperless_mail/mail.py` — IMAP email ingestion handler with configurable rules
- `src/paperless_mail/models.py` — MailAccount and MailRule ORM models
- `src/paperless_mail/tasks.py` — Mail processing task (`process_mail_accounts`)
- `src/documents/tasks.py` — Core task functions (`consume_file`, `train_classifier`, `index_reindex`, `index_optimize`, `sanity_check`, `bulk_update_documents`, barcode splitting)
- `src/documents/consumer.py` — The `Consumer` class with `try_consume_file()` orchestration

**Parsing and Text Extraction:**
- `src/documents/parsers.py` — Base `DocumentParser`, `parse_date()`, `get_parser_class_for_mime_type()`, signal-driven parser discovery
- `src/paperless_tesseract/parsers.py` — OCR via OCRmyPDF for PDFs and images
- `src/paperless_text/parsers.py` — Plain text/CSV direct reading
- `src/paperless_tika/parsers.py` — Apache Tika for Office documents (doc, docx, xls, xlsx, ppt, pptx, odt, ods, odp, rtf)
- `src/paperless_tesseract/signals.py`, `src/paperless_text/signals.py`, `src/paperless_tika/signals.py` — Parser registration via `document_consumer_declaration` signal

**Data Models and Metadata:**
- `src/documents/models.py` — `Document`, `Correspondent`, `Tag`, `DocumentType`, `MatchingModel`, `SavedView`, `SavedViewFilterRule`, `FileInfo`, `Log`
- `src/documents/serialisers.py` — REST API serialization layer for all models

**Classification and Matching:**
- `src/documents/classifier.py` — `DocumentClassifier` using scikit-learn MLPClassifier with CountVectorizer
- `src/documents/matching.py` — Rule-based matching functions (`match_correspondents`, `match_document_types`, `match_tags`)
- `src/documents/signals/handlers.py` — Post-consume signal handlers: `set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`, `add_to_index`, `set_log_entry`, `update_filename_and_move_files`

**Search and Indexing:**
- `src/documents/index.py` — Whoosh search index schema, document update/remove, full-text and "more like this" queries

**Configuration and Infrastructure:**
- `src/paperless/settings.py` — All settings including `Q_CLUSTER`, consumer options, OCR options, directory paths
- `src/paperless/urls.py` — API router and URL patterns
- `src/documents/apps.py` — Signal handler registration on app ready
- `src/documents/file_handling.py` — Filename generation and file move logic

### 0.2.3 Web Search Research Conducted

No web search was required for this task. The documentation deliverable is exclusively code-grounded per the user's instruction: *"Do not make assumptions, base your answers on the code as the truth."* All information is derived directly from source code inspection of the repository.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's questions map to the following modules requiring documentation:

**Module: Document Ingestion Entry Points**
- `src/documents/management/commands/document_consumer.py`
  - Public interfaces: `_consume()`, `_consume_wait_unmodified()`, `_tags_from_path()`, `Command.handle()`
  - Current documentation: Partial coverage in `docs/usage_overview.rst` (high-level), no internal architecture docs
  - Documentation needed: Detailed technical walkthrough of the directory watcher mechanism (inotify vs. polling), file validation, and async task dispatch
- `src/documents/views.py:PostDocumentView`
  - Public interface: `POST /api/documents/post_document/` accepting multipart form data
  - Current documentation: API endpoint documented in `docs/api.rst`
  - Documentation needed: Internal flow from upload to `async_task` dispatch with parameter mapping
- `src/paperless_mail/mail.py:MailAccountHandler`
  - Public interfaces: `handle_mail_account()`, `handle_mail_rule()`
  - Current documentation: User-facing docs in `docs/usage_overview.rst` (IMAP section)
  - Documentation needed: Internal mechanics of mail rule evaluation, attachment extraction, and task dispatch

**Module: Consumer Processing Pipeline**
- `src/documents/consumer.py:Consumer`
  - Public interface: `try_consume_file()` — the core pipeline orchestrator
  - Current documentation: No dedicated internal architecture docs
  - Documentation needed: Stage-by-stage walkthrough covering pre-checks, MIME detection, parser dispatch, text/date/thumbnail extraction, classification, atomic persistence, and file operations

**Module: Background Task System**
- `src/documents/tasks.py`
  - Public interfaces: `consume_file()`, `train_classifier()`, `index_reindex()`, `index_optimize()`, `sanity_check()`, `bulk_update_documents()`
  - Current documentation: No dedicated documentation of the task architecture
  - Documentation needed: Task inventory, scheduling configuration, Django-Q cluster settings
- `src/paperless/settings.py:Q_CLUSTER`
  - Configuration: Redis broker, worker count, timeout/retry settings
  - Documentation needed: Explanation of how Django-Q orchestrates background work

**Module: Data Models and Metadata**
- `src/documents/models.py:Document`
  - Fields: 16 direct fields + 3 ForeignKey/M2M relations + 7 computed properties
  - Current documentation: Basic field listing in `docs/api.rst`
  - Documentation needed: Full field catalog with required/optional/derived classification and a runtime example
- `src/documents/models.py:Correspondent`, `Tag`, `DocumentType`
  - Fields: Inherited from `MatchingModel` (name, match, matching_algorithm, is_insensitive) plus type-specific fields
  - Documentation needed: How these three organizational entities work together with matching

**Module: Classification and Matching**
- `src/documents/classifier.py:DocumentClassifier`
  - Public interfaces: `train()`, `predict_correspondent()`, `predict_document_type()`, `predict_tags()`
  - Documentation needed: ML pipeline explanation (CountVectorizer + MLPClassifier)
- `src/documents/matching.py`
  - Public interfaces: `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`
  - Documentation needed: How rule-based and ML-based matching combine

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist:

- **No dedicated document lifecycle guide** — Existing docs in `docs/usage_overview.rst` cover user-facing workflows but do not trace the internal code path end-to-end. No single document answers "what happens inside the code after a file enters the system."
- **No metadata field reference with classification** — `docs/api.rst` lists API fields but does not classify them as required vs. optional vs. runtime-derived, nor does it provide a runtime example.
- **No background job architecture documentation** — Django-Q configuration and the scheduled task inventory are not documented outside of inline comments in `src/paperless/settings.py`.
- **No practical organizational taxonomy guide** — While `docs/usage_overview.rst` defines terms, it does not illustrate with a concrete scenario how tags, correspondents, and document types interact in practice, including how matching algorithms and ML classification assign them automatically.
- **No parser plugin architecture overview** — The signal-based parser registration system is not documented at a conceptual level; `docs/extending.rst` exists but focuses on external plugin creation, not the internal dispatch mechanism.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive Markdown document answering all of the user's questions. It will be placed at:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
```

**Internal structure of the document:**

```
paperless-ngx_542221a38dff.md
├── 1. How Documents Enter Paperless-ngx
│   ├── 1.1 Consumption Directory (Filesystem Watcher)
│   ├── 1.2 REST API Upload
│   └── 1.3 Email (IMAP) Ingestion
├── 2. Processing Pipeline Stages
│   ├── 2.1 Pre-Checks (Existence, Directories, Duplicate)
│   ├── 2.2 MIME Detection and Parser Dispatch
│   ├── 2.3 Text Extraction and OCR
│   ├── 2.4 Date Parsing
│   ├── 2.5 Thumbnail Generation
│   ├── 2.6 Classification (ML + Rule-Based Matching)
│   ├── 2.7 Atomic Persistence (Database + Filesystem)
│   ├── 2.8 Search Index Update
│   └── 2.9 Post-Consume Hooks
├── 3. Background Jobs and Task Execution
│   ├── 3.1 Django-Q Cluster Configuration
│   ├── 3.2 Ad-Hoc Tasks (async_task)
│   └── 3.3 Scheduled Tasks
├── 4. Document Metadata Fields
│   ├── 4.1 Required Fields
│   ├── 4.2 Optional Fields
│   ├── 4.3 Runtime-Derived / Computed Properties
│   └── 4.4 Runtime Example
├── 5. Organizational Taxonomy
│   ├── 5.1 Tags
│   ├── 5.2 Correspondents
│   ├── 5.3 Document Types
│   ├── 5.4 Matching Algorithms
│   ├── 5.5 ML-Based Automatic Classification
│   └── 5.6 Practical Scenario
└── Rationale / Source Citations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the full ingestion flow by tracing code paths in `src/documents/management/commands/document_consumer.py` → `src/documents/tasks.py:consume_file()` → `src/documents/consumer.py:Consumer.try_consume_file()`
- Extract metadata fields by analyzing the `Document` model in `src/documents/models.py` and cross-referencing with `src/documents/serialisers.py`
- Extract the matching logic from `src/documents/matching.py` and `src/documents/classifier.py`
- Generate a runtime example by constructing a realistic `Document` record based on the model field definitions and default values

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration using fenced code blocks for pipeline visualization
- Code examples using Python fenced blocks with syntax highlighting
- Source citations as inline references in the format `Source: path/to/file.py:LineNumber`
- Tables for metadata field catalogs with field name, type, required/optional status, and description
- Consistent terminology aligned with the existing `docs/usage_overview.rst` definitions

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be included in the documentation:

- **Document Ingestion Flow** — A flowchart showing the three entry points converging at the Django-Q task queue and feeding into the Consumer pipeline
- **Processing Pipeline Stages** — A sequential diagram showing the step-by-step stages within `Consumer.try_consume_file()`
- **Classification Decision Flow** — A diagram showing how rule-based matching and ML prediction combine in signal handlers
- **Organizational Taxonomy Relationships** — An entity-relationship diagram showing Document ↔ Tag (M2M), Document → Correspondent (FK), Document → DocumentType (FK)


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/models.py`, `src/documents/matching.py`, `src/documents/classifier.py`, `src/documents/signals/handlers.py`, `src/documents/index.py`, `src/documents/parsers.py`, `src/documents/management/commands/document_consumer.py`, `src/documents/views.py`, `src/paperless_mail/mail.py`, `src/paperless/settings.py`, `src/documents/file_handling.py`, `src/paperless_tesseract/parsers.py`, `src/paperless_tesseract/signals.py`, `src/paperless_text/parsers.py`, `src/paperless_text/signals.py`, `src/paperless_tika/parsers.py`, `src/paperless_tika/signals.py`, `src/documents/apps.py` | Comprehensive Markdown document answering all four user questions: document ingestion flow, processing pipeline stages, background jobs, metadata fields, and organizational taxonomy with runtime example |

**Transformation mode:** CREATE — This is an entirely new documentation file that does not exist in the repository.

**Reference files consulted for style (REFERENCE mode — not modified):**

| Reference File | Purpose |
|----------------|---------|
| `docs/usage_overview.rst` | Existing user-facing documentation for terminology alignment |
| `docs/advanced_usage.rst` | Matching algorithm documentation for accuracy cross-reference |
| `README.md` | Project overview and feature descriptions |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Architecture / System Internals Guide
Source Code: 20+ source files across src/documents/, src/paperless/, src/paperless_mail/, src/paperless_tesseract/, src/paperless_text/, src/paperless_tika/
Sections:
    - Document Entry Points (three ingestion pathways with code traces)
    - Processing Pipeline (10-stage walkthrough of Consumer.try_consume_file)
    - Background Jobs (Django-Q cluster, scheduled tasks, async_task usage)
    - Metadata Fields (complete field catalog with required/optional/derived classification)
    - Runtime Example (concrete JSON representation of a fully processed Document)
    - Organizational Taxonomy (Tags, Correspondents, DocumentTypes with practical scenario)
    - Rationale and Source Citations
Diagrams:
    - Mermaid flowchart: Document ingestion entry points → task queue → consumer pipeline
    - Mermaid flowchart: Processing pipeline stages within Consumer
    - Mermaid ER diagram: Document ↔ Tag, Correspondent, DocumentType relationships
Key Citations:
    src/documents/consumer.py, src/documents/tasks.py, src/documents/models.py,
    src/documents/matching.py, src/documents/classifier.py,
    src/documents/signals/handlers.py, src/documents/index.py,
    src/documents/management/commands/document_consumer.py,
    src/documents/views.py, src/paperless_mail/mail.py,
    src/paperless/settings.py, src/documents/file_handling.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The deliverable is a standalone Markdown file in `blitzy/documentation/` that is independent of the existing Sphinx documentation system. The existing `docs/conf.py`, `.readthedocs.yml`, and `docs/index.rst` will not be modified per the user's instruction to make no repository changes.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tooling is required for this task. The deliverable is a standalone Markdown file that does not depend on any documentation generator or build tool. However, the following packages from the existing project are relevant to the documentation content (i.e., they are the systems being documented):

| Registry | Package Name | Version | Purpose (as documented in output) |
|----------|--------------|---------|-----------------------------------|
| pip | django | 4.0.4 | Core web framework; ORM for Document model |
| pip | django-q | 1.3.9 | Background task queue with Redis broker |
| pip | redis | 3.5.3 | Message broker for Django-Q and Channels |
| pip | channels | 3.0.4 | WebSocket support for real-time status updates |
| pip | channels-redis | 3.4.0 | Redis channel layer backend for Django Channels |
| pip | ocrmypdf | 13.4.3 | PDF OCR processing engine |
| pip | scikit-learn | 1.0.2 | ML classifier (MLPClassifier) for auto-matching |
| pip | whoosh | 2.7.4 | Full-text search index engine |
| pip | python-magic | 0.4.25 | MIME type detection for parser dispatch |
| pip | pyzbar | 0.1.9 | Barcode detection for document separation |
| pip | pikepdf | 5.1.1 | PDF page splitting for barcode separation |
| pip | dateparser | 1.1.1 | Date extraction from document text |
| pip | fuzzywuzzy | 0.18.0 | Fuzzy string matching algorithm |
| pip | watchdog | 2.1.7 | Polling-based filesystem observer (fallback) |
| pip | inotifyrecursive | 0.3.5 | Linux inotify filesystem watcher (primary) |
| pip | tika | 1.24 | Apache Tika client for Office document parsing |
| pip | pdfminer.six | 20220319 | PDF text extraction fallback |
| pip | imap-tools | 0.54.0 | IMAP email fetching library |
| pip | djangorestframework | 3.13.1 | REST API framework for document upload endpoint |
| pip | gunicorn | 20.1.0 | ASGI/WSGI server with Uvicorn workers |
| pip | sphinx | ~4.5.0 | Documentation generator (existing docs system, dev dependency) |
| pip | sphinx_rtd_theme | * | ReadTheDocs theme for existing Sphinx docs |

### 0.6.2 Documentation Reference Updates

Not applicable. Since the deliverable is a standalone file in `blitzy/documentation/` and no existing documentation files are being modified, no link updates are required.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Target coverage by user question:**

| User Question | Source Modules to Cover | Target Coverage |
|---------------|------------------------|-----------------|
| How does a document enter the system? | 3 entry points (directory watcher, API upload, email) | 100% of entry points documented |
| What are the main processing stages? | 10 stages in `Consumer.try_consume_file()` | 100% of stages documented |
| Background jobs and execution engine? | Django-Q config, 6 task functions, 3 scheduled tasks | 100% of tasks and schedules documented |
| What metadata fields are stored? | 16 direct fields + 3 relationships + 7 computed properties on `Document` | 100% of fields cataloged |
| Required vs. optional vs. derived fields? | Classification of all 26 fields/properties | 100% classified with runtime example |
| How are tags, correspondents, types used together? | 3 organizational models + 6 matching algorithms + ML classifier | 100% with practical scenario |

**Current coverage (before this documentation):** The existing `docs/usage_overview.rst` and `docs/advanced_usage.rst` cover approximately 40% of the user's questions at a user-facing level, with 0% coverage of internal code architecture, metadata field classification, or runtime examples.

**Target coverage (after this documentation):** 100% of the user's questions fully answered with code-grounded explanations, source citations, diagrams, and a runtime example.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must be answered with a dedicated section
- All public entry points for document ingestion are traced with source file citations
- Every `Document` model field is listed with its Django field type, constraints, and classification
- The runtime example includes realistic values for all field categories
- The organizational taxonomy explanation includes a concrete real-world scenario

**Accuracy validation:**
- All code paths described must reference actual method names and file paths verified via `read_file`
- Field types and constraints must exactly match `src/documents/models.py` definitions
- Django-Q configuration values must match `src/paperless/settings.py:Q_CLUSTER`
- Scheduled task functions and frequencies must match migration files `1001` and `1004`

**Clarity standards:**
- Technical accuracy with accessible language suitable for a developer exploring the codebase
- Progressive disclosure: high-level overview first, then detailed walkthrough
- Mermaid diagrams for visual pipeline comprehension
- Tables for structured field catalogs
- Inline source citations for traceability

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 3 Mermaid diagrams (ingestion flow, processing pipeline, entity relationships)
- **Runtime example:** 1 complete JSON/dict representation of a `Document` with all field categories populated
- **Practical scenario:** 1 narrative example showing tags, correspondents, and document types working together for a realistic use case (e.g., household utility bill management)
- **Code snippets:** Short excerpts from source files where they illuminate key mechanisms (e.g., signal handler registration in `apps.py`)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — The sole deliverable answering all user questions

**Topics covered in the documentation:**
- Document ingestion entry points: consumption directory watcher, REST API upload, IMAP email fetch
- Processing pipeline stages: validation, MIME detection, parser dispatch, OCR/text extraction, date parsing, thumbnail generation, classification, atomic persistence, search indexing, post-consume hooks
- Background job architecture: Django-Q cluster configuration, `async_task` dispatch pattern, scheduled tasks (classifier training, index optimization, sanity check, email fetch)
- Document metadata model: complete field catalog from `src/documents/models.py:Document`
- Required vs. optional vs. runtime-derived field classification
- Runtime example with realistic field values
- Organizational taxonomy: Tags, Correspondents, Document Types
- Matching algorithms: Any, All, Literal, Regex, Fuzzy, Auto (ML)
- ML classification pipeline: CountVectorizer + MLPClassifier from scikit-learn
- Practical organizational scenario illustrating how all taxonomy elements work together
- Parser plugin architecture: signal-based registration of Tesseract, Text, and Tika parsers
- File naming and storage layout driven by metadata via `PAPERLESS_FILENAME_FORMAT`
- Whoosh search index schema and its relationship to document metadata

**Source files analyzed (read during context gathering):**
- `src/documents/consumer.py`, `src/documents/tasks.py`, `src/documents/models.py`
- `src/documents/matching.py`, `src/documents/classifier.py`, `src/documents/parsers.py`
- `src/documents/signals/__init__.py`, `src/documents/signals/handlers.py`
- `src/documents/index.py`, `src/documents/views.py`, `src/documents/apps.py`
- `src/documents/file_handling.py`, `src/documents/bulk_edit.py`
- `src/documents/serialisers.py`, `src/documents/management/commands/document_consumer.py`
- `src/paperless/settings.py`, `src/paperless/urls.py`, `src/paperless/workers.py`
- `src/paperless_mail/mail.py`, `src/paperless_mail/models.py`, `src/paperless_mail/tasks.py`
- `src/paperless_tesseract/parsers.py`, `src/paperless_tesseract/signals.py`, `src/paperless_tesseract/apps.py`
- `src/paperless_text/parsers.py`, `src/paperless_text/signals.py`, `src/paperless_text/apps.py`
- `src/paperless_tika/parsers.py`, `src/paperless_tika/signals.py`, `src/paperless_tika/apps.py`
- `docs/conf.py`, `docs/index.rst`, `docs/usage_overview.rst`, `docs/advanced_usage.rst`
- `Pipfile`, `requirements.txt`, `.readthedocs.yml`, `README.md`

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly stated no changes to the repository
- **Existing documentation updates** — No changes to `docs/*.rst`, `README.md`, or any existing file
- **Frontend (Angular) internals** — The `src-ui/` directory is not analyzed; this is a backend-focused documentation task
- **Docker / deployment configuration** — While `docker/` files exist, the user's questions focus on runtime behavior, not deployment
- **Test files** — `src/documents/tests/` are not documented (not part of the user's questions)
- **Migration files** — Only referenced for scheduled task registration; migration internals are out of scope
- **Locale / translation files** — `src/locale/` is not relevant to the user's questions
- **UI screenshots or visual guides** — The user asked for code-level understanding, not UI documentation
- **Performance tuning or optimization** — Not part of the user's questions
- **Security architecture** — Not part of the user's questions
- **API endpoint reference** — Only the `PostDocumentView` upload endpoint is documented as an ingestion pathway; the full REST API is out of scope


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not part of the Sphinx documentation build
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/paperless-ngx_542221a38dff.md`
- **Diagram generation:** Mermaid diagrams are embedded inline in the Markdown using fenced code blocks (`\`\`\`mermaid`); they render in any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim references the source file and, where relevant, the specific class or function name
- **Style guide:** Markdown with ATX-style headers, fenced code blocks, GFM tables
- **Documentation validation:** Manual review for completeness against user questions; verify all cited file paths exist in the repository

### 0.9.2 Deliverable Output

The output file must be created at:
```
blitzy/documentation/paperless-ngx_542221a38dff.md
```

This file must:
- Comprehensively answer all four user questions with code-grounded explanations
- Include thinking and rationale behind the answers (per the implementation rule)
- Not modify any existing files in the source repository (per user instruction and implementation rule)
- Be self-contained with all diagrams and examples embedded inline


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The only file created is `blitzy/documentation/paperless-ngx_542221a38dff.md`.
- **Temporary scripts are allowed for testing but must be cleaned up.** If any test scripts are used to verify behavior, they must be removed before completion.
- **Do not make assumptions — base all answers on the code as truth.** Every claim in the documentation must be traceable to a specific source file, class, method, or configuration value.
- **Provide thinking and rationale behind the answers.** The documentation should not just state facts but explain *why* the system works the way it does, based on the code patterns observed.
- **Include a runtime example for metadata fields.** The user specifically requested this; it must be a concrete, realistic illustration of a `Document` record.
- **Explain practical usage of organizational taxonomy.** The documentation must go beyond definitions to show how tags, correspondents, and document types work together in a real-world scenario.
- **Place the output in `blitzy/documentation/` directory.** The file must be named `paperless-ngx_542221a38dff.md` per the naming convention `<source_branch_name>.md`.


## 0.11 References

### 0.11.1 Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive all conclusions in this Agent Action Plan:

**Core Document Pipeline:**
| File Path | Purpose |
|-----------|---------|
| `src/documents/consumer.py` | Core `Consumer` class with `try_consume_file()` orchestrating the full processing pipeline |
| `src/documents/tasks.py` | Background task functions: `consume_file`, `train_classifier`, `index_reindex`, `index_optimize`, `sanity_check`, `bulk_update_documents`, barcode splitting utilities |
| `src/documents/models.py` | ORM models: `Document`, `Correspondent`, `Tag`, `DocumentType`, `MatchingModel`, `SavedView`, `SavedViewFilterRule`, `FileInfo`, `Log` |
| `src/documents/parsers.py` | Base `DocumentParser` class, `parse_date()`, `get_parser_class_for_mime_type()`, parser discovery via signals |
| `src/documents/matching.py` | Rule-based matching: `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()` with 6 algorithms |
| `src/documents/classifier.py` | `DocumentClassifier` using scikit-learn MLPClassifier + CountVectorizer for auto-matching |
| `src/documents/signals/__init__.py` | Django signal definitions: `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` |
| `src/documents/signals/handlers.py` | Post-consume handlers: `set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`, `add_to_index`, `set_log_entry`, `update_filename_and_move_files`, `cleanup_document_deletion` |
| `src/documents/index.py` | Whoosh search index: schema definition, `update_document()`, `add_or_update_document()`, `DelayedFullTextQuery`, `DelayedMoreLikeThisQuery` |
| `src/documents/views.py` | REST API views including `PostDocumentView` (document upload endpoint) |
| `src/documents/apps.py` | Signal handler registration on Django app ready |
| `src/documents/file_handling.py` | `generate_filename()`, `generate_unique_filename()`, `create_source_path_directory()`, `delete_empty_directories()` |
| `src/documents/bulk_edit.py` | Bulk edit operations dispatching `async_task` for index updates |
| `src/documents/serialisers.py` | DRF serializers for Document, Correspondent, Tag, DocumentType, SavedView |
| `src/documents/management/commands/document_consumer.py` | Directory watcher management command with inotify and polling modes |
| `src/documents/management/commands/document_retagger.py` | Re-tagging management command using classifier and signal handlers |

**Parser Plugins:**
| File Path | Purpose |
|-----------|---------|
| `src/paperless_tesseract/parsers.py` | `RasterisedDocumentParser` — OCR via OCRmyPDF for PDFs and images |
| `src/paperless_tesseract/signals.py` | Parser registration for PDF, JPEG, PNG, TIFF, GIF, BMP (weight 0) |
| `src/paperless_tesseract/apps.py` | Signal connection for Tesseract parser |
| `src/paperless_text/parsers.py` | `TextDocumentParser` — Direct text reading for .txt and .csv |
| `src/paperless_text/signals.py` | Parser registration for text/plain and text/csv (weight 10) |
| `src/paperless_text/apps.py` | Signal connection for Text parser |
| `src/paperless_tika/parsers.py` | `TikaDocumentParser` — Apache Tika for Office documents |
| `src/paperless_tika/signals.py` | Parser registration for doc, docx, xls, xlsx, ppt, pptx, odp, ods, odt, rtf (weight 10) |
| `src/paperless_tika/apps.py` | Conditional signal connection (only when `PAPERLESS_TIKA_ENABLED`) |

**Email Ingestion:**
| File Path | Purpose |
|-----------|---------|
| `src/paperless_mail/mail.py` | `MailAccountHandler` — IMAP mail fetching, rule evaluation, attachment extraction |
| `src/paperless_mail/models.py` | `MailAccount` and `MailRule` models with action/filter definitions |
| `src/paperless_mail/tasks.py` | `process_mail_accounts()` — iterates all accounts and processes via handler |

**Configuration and Infrastructure:**
| File Path | Purpose |
|-----------|---------|
| `src/paperless/settings.py` | All application settings: `Q_CLUSTER`, consumer options, OCR options, directory paths, filename format |
| `src/paperless/urls.py` | API router, URL patterns, WebSocket routes |
| `src/paperless/workers.py` | Gunicorn Uvicorn worker configuration |
| `src/paperless/version.py` | Version number (1.7.0) |

**Existing Documentation:**
| File Path | Purpose |
|-----------|---------|
| `docs/conf.py` | Sphinx configuration — theme, extensions, version extraction |
| `docs/index.rst` | Documentation root index |
| `docs/usage_overview.rst` | User-facing overview: terms, adding documents, consumption directory, email |
| `docs/advanced_usage.rst` | Matching algorithms, automatic matching, file naming |
| `.readthedocs.yml` | ReadTheDocs hosting configuration |
| `README.md` | Project overview, feature list, community links |
| `CONTRIBUTING.md` | Contribution guidelines, team structure |

**Dependency Manifests:**
| File Path | Purpose |
|-----------|---------|
| `Pipfile` | Python dependency declarations with version constraints |
| `requirements.txt` | Pinned dependency versions (auto-generated from Pipfile.lock) |
| `src/setup.cfg` | Pytest/flake8 configuration |

**Migration Files (referenced for scheduled tasks only):**
| File Path | Purpose |
|-----------|---------|
| `src/documents/migrations/1001_auto_20201109_1636.py` | Registers hourly `train_classifier` and daily `index_optimize` schedules |
| `src/documents/migrations/1004_sanity_check_schedule.py` | Registers weekly `sanity_check` schedule |

### 0.11.2 Attachments Provided

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens Provided

No Figma screens were provided for this project.


