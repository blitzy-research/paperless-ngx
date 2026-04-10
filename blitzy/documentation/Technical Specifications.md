# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new comprehensive investigative analysis document** that answers specific questions about memory usage patterns observed during document import operations in the Paperless-ngx v1.7.0 system.

- **Category**: Create new documentation
- **Documentation Type**: Technical investigation / Memory analysis report
- **Target Output**: A markdown document named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation` directory

The user reports the following observable symptoms requiring investigation and documentation:

- Memory usage spikes disproportionately during document import operations, particularly during metadata processing stages
- The spikes are inconsistent — they vary based on document source (scanner, email, API upload) and processing stage (parsing, OCR, classification, indexing)
- Memory is not always released in a timely manner after processing completes, even for relatively small text documents with minimal metadata
- The behavior is non-deterministic — the same operation sometimes consumes more or less memory depending on contextual factors

The user asks for:

- Identification of the root causes behind the memory spikes during document import
- Analysis of whether metadata handling creates unnecessary copies or holds references longer than needed
- Investigation of caching behavior that may accumulate data unexpectedly
- Comparison of memory behavior between spiking and non-spiking cases
- Analysis of how document types and batch sizes affect the behavior
- Runtime memory measurements during processing
- Identification of specific components and methods responsible for memory retention
- Determination of whether this is normal Python garbage collection behavior or a problematic pattern

### 0.1.2 Special Instructions and Constraints

The following critical directives govern this investigation:

- **No Source Code Modifications**: The user explicitly states: "Don't modify any repository source files." All analysis must be observational and documentary only. No changes to any files in `src/`, `docs/`, or other repository directories.
- **Temporary Scripts Permitted**: The user allows creation of temporary test scripts or helper tools to reproduce and analyze behavior, but these must be cleaned up and the codebase left unchanged.
- **Implementation Rule**: Per the `SWE-AtlasQnA-Repo` rule, the output must be a new markdown document named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation` directory, providing thinking/rationale behind all answers, basing all conclusions on the code as the source of truth.
- **Evidence-Based Analysis**: All findings must be grounded in actual code paths, not assumptions. Every claim must reference specific source files and line numbers.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the memory spike root causes, we will analyze the complete document consumption pipeline in `src/documents/consumer.py` (the `Consumer.try_consume_file()` method) and trace every memory-intensive operation: file reads, parser instantiation, classifier loading, signal dispatch, and file copying
- To document metadata handling concerns, we will analyze how `src/documents/parsers.py`, `src/paperless_tesseract/parsers.py`, `src/paperless_tika/parsers.py`, and `src/paperless_text/parsers.py` handle document metadata extraction and text processing
- To document caching and reference retention, we will analyze the classifier model lifecycle in `src/documents/classifier.py` (pickle-based deserialization of scikit-learn models), the Whoosh index interactions in `src/documents/index.py`, and the Django-Q task worker recycling behavior configured in `src/paperless/settings.py`
- To document behavioral differences across document types, we will compare memory profiles of text documents (lightweight `TextDocumentParser`), PDF/image documents (heavyweight `RasterisedDocumentParser` with OCR), and Tika-processed documents (`TikaDocumentParser` with external HTTP calls)
- To document batch size effects, we will analyze the Django-Q worker pool configuration (`Q_CLUSTER` in settings.py with `recycle: 1`) and how concurrent workers interact with shared resources like the classifier model and search index

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Multiple Full-File Reads**: The consumer pipeline reads the entire file into memory at least three separate times during processing — in `pre_check_duplicate()` (line 103), `_store()` (line 397), and `_write()` (line 430) of `src/documents/consumer.py`. This pattern requires documentation as a primary contributor to memory spikes.
- **Classifier Model Deserialization**: The `load_classifier()` function in `src/documents/classifier.py` deserializes a pickle file containing three scikit-learn MLPClassifier models, a CountVectorizer, and label binarizers. This model grows with the document corpus and is loaded into memory for every consumed document.
- **Barcode Processing Memory Explosion**: When `CONSUMER_ENABLE_BARCODES` is active, `src/documents/tasks.py:scan_file_for_separating_barcodes()` converts every PDF page to an in-memory image via `pdf2image.convert_from_path()`, which can consume enormous memory for multi-page PDFs.
- **Signal Handler Cascade**: The `document_consumption_finished` signal triggers a cascade of handlers in `src/documents/signals/handlers.py` — including `set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`, `set_log_entry`, and `add_to_index` — each performing database queries and holding references to the document and classifier objects.
- **Django-Q Worker Recycling**: The `Q_CLUSTER` configuration sets `recycle: 1` (line 452 of settings.py), meaning each worker process handles exactly one task before being replaced. This prevents long-term memory accumulation but forces repeated classifier model loading, creating per-task memory spikes.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a Sphinx-based documentation system hosted via Read the Docs, with limited coverage of memory behavior or performance profiling topics.

- **Documentation Framework**: Sphinx version `~4.5.0` (pinned in `Pipfile`, dev-packages)
- **Documentation Theme**: `sphinx_rtd_theme` (Read the Docs theme), as configured in `docs/conf.py` line 91
- **Documentation Generator Configuration**: `docs/conf.py` — Sphinx configuration with autodoc, intersphinx, todo, imgmath, and viewcode extensions
- **Hosting Configuration**: `.readthedocs.yml` — Read the Docs v2 configuration pointing to `docs/conf.py` with Python 3.8
- **Build System**: `docs/Makefile` — Standard Sphinx Makefile for HTML, EPUB, LaTeX, man pages, gettext, link checking, and doctests
- **Documentation Dependencies**: `docs/requirements.txt` — currently empty (placeholder)
- **Existing Documentation Files**:
  - `docs/index.rst` — Landing page and navigation hub
  - `docs/setup.rst` — Installation, migration, reverse proxy, deployment
  - `docs/configuration.rst` — All environment variables and runtime settings
  - `docs/usage_overview.rst` — Product model, ingestion methods, search, workflows
  - `docs/advanced_usage.rst` — Advanced matching, hooks, filename handling
  - `docs/administration.rst` — Backups, updates, utilities, indexing, archiving
  - `docs/api.rst` — REST endpoints, authentication, uploads, search, versioning
  - `docs/troubleshooting.rst` — Common operational failures and fixes
  - `docs/extending.rst` — Contributor workflows, development setup, localization
  - `docs/faq.rst` — Common support and deployment questions
  - `docs/changelog.rst` — Release history
- **Diagram Tools**: No Mermaid or PlantUML detected in Sphinx configuration; documentation uses reStructuredText with Sphinx directives
- **No existing memory profiling or performance documentation**: The troubleshooting page (`docs/troubleshooting.rst`) covers consumer file pickup issues, OCR failures, and redirect problems, but contains no memory analysis content

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to identify code modules relevant to the memory investigation:

- **Document ingestion pipeline**: `src/documents/consumer.py` — The `Consumer` class orchestrates the entire import lifecycle from file validation through parsing, metadata extraction, classification, storage, and signal dispatch
- **Parser implementations**: `src/documents/parsers.py` (base), `src/paperless_tesseract/parsers.py` (OCR), `src/paperless_text/parsers.py` (text), `src/paperless_tika/parsers.py` (Tika)
- **Classifier system**: `src/documents/classifier.py` — Pickle-based serialization of scikit-learn models with per-document loading
- **Task queue**: `src/documents/tasks.py` — Background jobs including `consume_file()`, barcode processing, classifier training, and reindexing
- **Signal handlers**: `src/documents/signals/handlers.py` — Post-consumption hooks for matching, file renaming, audit logging, and indexing
- **Search index**: `src/documents/index.py` — Whoosh-based full-text search with document update, removal, and delayed query classes
- **File handling**: `src/documents/file_handling.py` — Filename generation with template formatting and ManyToMany field conversion
- **Matching engine**: `src/documents/matching.py` — Rule-based matching including fuzzy matching with full-content regex operations
- **Django settings**: `src/paperless/settings.py` — Worker pool configuration, memory limits, OCR settings, task queue with `recycle: 1`
- **Mail ingestion**: `src/paperless_mail/mail.py` — Email attachment processing with temporary file creation
- **Management commands**: `src/documents/management/commands/document_consumer.py`, `document_importer.py` — Filesystem watching and bulk import
- **Serializers**: `src/documents/serialisers.py` — Upload validation reading entire file into memory
- **Models**: `src/documents/models.py` — Document, Tag, Correspondent, DocumentType ORM definitions with file path properties

Key directories examined:
- `src/documents/` — Core document management application (19 Python files + subdirectories)
- `src/paperless/` — Django project configuration and runtime
- `src/paperless_tesseract/` — OCR parser integration
- `src/paperless_text/` — Plain text parser
- `src/paperless_tika/` — Tika parser integration
- `src/paperless_mail/` — Email ingestion subsystem
- `docs/` — Existing Sphinx documentation

### 0.2.3 Web Search Research Conducted

No external web search was required for this analysis. All findings are derived directly from codebase inspection, as the investigation focuses on actual code patterns rather than external best practices. The memory behavior analysis is entirely evidence-based, grounded in specific code paths identified in the repository.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require in-depth analysis and documentation in the memory investigation report:

- **Module: `src/documents/consumer.py`**
  - Public APIs: `Consumer.try_consume_file()`, `Consumer.pre_check_duplicate()`, `Consumer._store()`, `Consumer._write()`, `Consumer.run_post_consume_script()`
  - Current documentation: No memory-focused documentation exists
  - Documentation needed: Detailed memory flow analysis of the entire consumption pipeline, identifying every point where file contents are read into memory, where large objects are created, and where references are held

- **Module: `src/documents/classifier.py`**
  - Public APIs: `load_classifier()`, `DocumentClassifier.load()`, `DocumentClassifier.train()`, `DocumentClassifier.predict_correspondent()`, `DocumentClassifier.predict_document_type()`, `DocumentClassifier.predict_tags()`
  - Current documentation: No memory-focused documentation exists
  - Documentation needed: Analysis of pickle deserialization memory impact, model size growth characteristics, and per-task loading overhead due to `Q_CLUSTER recycle: 1`

- **Module: `src/documents/tasks.py`**
  - Public APIs: `consume_file()`, `scan_file_for_separating_barcodes()`, `separate_pages()`, `train_classifier()`, `index_reindex()`, `bulk_update_documents()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Analysis of barcode scanning memory explosion via `convert_from_path()`, PDF manipulation with pikepdf, and task-level memory isolation

- **Module: `src/documents/parsers.py`**
  - Public APIs: `DocumentParser` base class, `parse_date()`, `run_convert()`, `get_parser_class_for_mime_type()`
  - Current documentation: No memory-focused documentation exists
  - Documentation needed: Analysis of temporary directory management, date regex iteration over full text, and subprocess memory patterns

- **Module: `src/paperless_tesseract/parsers.py`**
  - Public APIs: `RasterisedDocumentParser.parse()`, `.extract_metadata()`, `.extract_text()`, `.construct_ocrmypdf_parameters()`
  - Current documentation: No memory analysis exists
  - Documentation needed: PIL image processing memory (alpha layer removal creates two full images in memory), pdfminer text extraction memory, ocrmypdf thread/process spawning

- **Module: `src/paperless_text/parsers.py`**
  - Public APIs: `TextDocumentParser.parse()`, `.get_thumbnail()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Lightweight comparison baseline — reads full file but creates PIL image for thumbnail

- **Module: `src/paperless_tika/parsers.py`**
  - Public APIs: `TikaDocumentParser.parse()`, `.extract_metadata()`, `.convert_to_pdf()`
  - Current documentation: No memory analysis exists
  - Documentation needed: HTTP response content held in memory (`response.content`), double parsing (tika + gotenberg), file write patterns

- **Module: `src/documents/signals/handlers.py`**
  - Public APIs: `set_correspondent()`, `set_document_type()`, `set_tags()`, `add_inbox_tags()`, `set_log_entry()`, `add_to_index()`, `update_filename_and_move_files()`, `cleanup_document_deletion()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Signal handler cascade reference retention, database query patterns during post-consumption processing

- **Module: `src/documents/matching.py`**
  - Public APIs: `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Full-content regex operations for every matching model, fuzzy matching creates stripped copies of entire document content

- **Module: `src/documents/index.py`**
  - Public APIs: `open_index()`, `update_document()`, `add_or_update_document()`, `DelayedQuery`, `autocomplete()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Whoosh index writer memory, `DelayedQuery.saved_results` caching pattern

- **Module: `src/paperless/settings.py`**
  - Configuration: `Q_CLUSTER`, `TASK_WORKERS`, `THREADS_PER_WORKER`, `CONVERT_MEMORY_LIMIT`, `OCR_MAX_IMAGE_PIXELS`
  - Current documentation: `docs/configuration.rst` covers settings but not memory implications
  - Documentation needed: How configuration parameters directly affect memory consumption patterns

- **Module: `src/paperless_mail/mail.py`**
  - Public APIs: `MailAccountHandler.handle_message()`, `.handle_mail_rule()`
  - Current documentation: No memory analysis exists
  - Documentation needed: Attachment payload held in memory (`att.payload`), temporary file creation patterns

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No memory profiling documentation exists anywhere in the repository** — neither in `docs/` nor in inline code comments
- **No performance analysis documentation** — the troubleshooting page covers functional issues only
- **No documentation of the multi-read file pattern** — the consumer reads each file 3-4 times fully into memory with no streaming
- **No documentation of classifier model sizing** — the pickle-serialized model grows with the corpus but there is no guidance on expected memory footprint
- **No documentation of barcode processing memory requirements** — the `convert_from_path` call in `tasks.py` can explode memory for large PDFs
- **No documentation of the interaction between `recycle: 1` and classifier loading** — each task forces a fresh model deserialization
- **No documentation of the signal cascade memory overhead** — post-consumption signals hold references to the classifier and document simultaneously across multiple handler invocations

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will follow this structure within `blitzy/documentation/paperless-ngx_542221a38dff.md`:

```
blitzy/documentation/
└── paperless-ngx_542221a38dff.md
    ├── Executive Summary
    ├── Memory Flow Analysis
    │   ├── Document Consumption Pipeline
    │   ├── Per-Stage Memory Breakdown
    │   └── Memory Lifecycle Diagram
    ├── Root Cause Analysis
    │   ├── Multiple Full-File Reads
    │   ├── Classifier Model Loading
    │   ├── Barcode Processing
    │   ├── Parser-Specific Patterns
    │   ├── Signal Handler Cascade
    │   └── Matching Engine Overhead
    ├── Behavioral Differences
    │   ├── By Document Type
    │   ├── By Processing Stage
    │   ├── By Batch Size
    │   └── Spiking vs Non-Spiking Cases
    ├── Memory Retention Analysis
    │   ├── Reference Holding Patterns
    │   ├── Caching Behavior
    │   ├── Python GC Considerations
    │   └── Worker Recycling Impact
    ├── Component-Level Findings
    │   ├── consumer.py Analysis
    │   ├── classifier.py Analysis
    │   ├── tasks.py Analysis
    │   ├── parsers.py Analysis
    │   ├── signals/handlers.py Analysis
    │   └── matching.py Analysis
    ├── Evidence and Measurements
    │   ├── Code Path Memory Estimates
    │   ├── Configuration Impact
    │   └── Critical Code Excerpts
    └── Conclusions and Recommendations
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach**:
  - Extract memory-critical code paths from `src/documents/consumer.py` by tracing every `open(..., "rb")` and `.read()` call
  - Map classifier memory impact from `src/documents/classifier.py` by analyzing the pickle deserialization chain (6 separate `pickle.load()` calls)
  - Identify barcode processing memory explosion from `src/documents/tasks.py:scan_file_for_separating_barcodes()` by tracing the `convert_from_path()` + `pyzbar.decode()` path
  - Analyze parser memory profiles from each parser module by comparing the base `DocumentParser.__init__()` (tempdir creation) with the parser-specific `parse()` implementations
  - Trace signal handler memory retention from `src/documents/signals/handlers.py` by following the `document_consumption_finished.send()` call and its receivers

- **Template Application**: The investigation document will follow the `SWE-AtlasQnA-Repo` rule format — comprehensive markdown answers with thinking/rationale, grounded in code evidence

- **Documentation Standards**:
  - Markdown formatting with proper headers (`#`, `##`, `###`)
  - Mermaid diagrams for memory flow visualization and pipeline stage analysis
  - Code excerpts from actual source files with file path and line number citations
  - Tables for comparison of memory behavior across document types and processing stages
  - Source citations in format: `Source: /path/to/file.py:LineNumber`

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **Document Consumption Memory Flow**: A flowchart showing every memory allocation point in the `try_consume_file()` pipeline, from file existence check through final storage
- **Classifier Loading Sequence**: A sequence diagram showing the pickle deserialization chain and the per-task reload pattern due to `recycle: 1`
- **Parser Comparison Matrix**: A visual comparing the memory footprint characteristics of `RasterisedDocumentParser`, `TextDocumentParser`, and `TikaDocumentParser`
- **Signal Handler Cascade**: A flowchart showing the chain of signal receivers triggered after document consumption and their aggregate memory holding patterns
- **Memory Timeline**: A conceptual timeline diagram showing when memory is allocated and when it becomes eligible for GC release during a single document consumption cycle

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| blitzy/documentation/paperless-ngx_542221a38dff.md | CREATE | src/documents/consumer.py, src/documents/classifier.py, src/documents/tasks.py, src/documents/parsers.py, src/paperless_tesseract/parsers.py, src/paperless_text/parsers.py, src/paperless_tika/parsers.py, src/documents/signals/handlers.py, src/documents/matching.py, src/documents/index.py, src/paperless/settings.py, src/paperless_mail/mail.py, src/documents/models.py, src/documents/file_handling.py, src/documents/serialisers.py, src/documents/sanity_checker.py, src/documents/loggers.py | Comprehensive memory analysis document answering all user questions about memory spikes during document import, with evidence-based findings, code citations, Mermaid diagrams, and actionable conclusions |

No existing documentation files will be modified. No files will be deleted. The scope is strictly limited to creating a single new analysis document.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Memory Analysis Report
Source Code References:
    - src/documents/consumer.py (primary pipeline)
    - src/documents/classifier.py (ML model loading)
    - src/documents/tasks.py (task orchestration, barcode processing)
    - src/documents/parsers.py (base parser, date parsing)
    - src/paperless_tesseract/parsers.py (OCR parser, PIL usage)
    - src/paperless_text/parsers.py (text parser baseline)
    - src/paperless_tika/parsers.py (Tika HTTP parser)
    - src/documents/signals/handlers.py (post-consumption handlers)
    - src/documents/matching.py (rule-based matching engine)
    - src/documents/index.py (Whoosh search index)
    - src/paperless/settings.py (Q_CLUSTER, worker config, memory limits)
    - src/paperless_mail/mail.py (email attachment processing)
    - src/documents/models.py (Document model, file properties)
    - src/documents/file_handling.py (filename generation)
    - src/documents/serialisers.py (upload validation)
    - src/documents/sanity_checker.py (integrity verification)
Sections:
    - Executive Summary (concise findings overview)
    - Memory Flow Analysis (pipeline walkthrough with diagrams)
    - Root Cause Analysis (each identified cause with code evidence)
    - Behavioral Differences (by type, stage, batch size)
    - Memory Retention Analysis (GC, references, caching)
    - Component-Level Findings (per-module deep dive)
    - Evidence and Measurements (code-based estimates)
    - Conclusions and Recommendations (summary of findings)
Diagrams:
    - Document consumption memory flow (Mermaid flowchart)
    - Classifier loading sequence (Mermaid sequence diagram)
    - Signal handler cascade (Mermaid flowchart)
    - Memory timeline (Mermaid timeline)
Key Citations:
    - src/documents/consumer.py:103-104 (duplicate check full-file read)
    - src/documents/consumer.py:292 (classifier loading)
    - src/documents/consumer.py:397-402 (store full-file read)
    - src/documents/consumer.py:430-432 (write full-file read)
    - src/documents/consumer.py:339-342 (archive checksum full-file read)
    - src/documents/classifier.py:77-93 (pickle deserialization chain)
    - src/documents/classifier.py:117-199 (training data collection)
    - src/documents/tasks.py:105 (convert_from_path for barcodes)
    - src/documents/matching.py:128-135 (fuzzy match full-content copy)
    - src/documents/parsers.py:261 (date regex iteration)
    - src/paperless_tesseract/parsers.py:197-201 (PIL alpha processing)
    - src/paperless/settings.py:452 (recycle: 1)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone markdown document placed in the `blitzy/documentation` directory per the `SWE-AtlasQnA-Repo` implementation rule. It does not integrate into the existing Sphinx documentation system.

### 0.5.4 Cross-Documentation Dependencies

- The analysis document references the existing `docs/configuration.rst` for context on environment variables like `PAPERLESS_CONVERT_MEMORY_LIMIT`, `PAPERLESS_TASK_WORKERS`, and `PAPERLESS_THREADS_PER_WORKER`
- The analysis document references the existing `docs/troubleshooting.rst` for context on known operational issues
- No navigation, table of contents, or index updates are needed since this is a standalone deliverable

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to this documentation exercise — they are the runtime dependencies of the Paperless-ngx system whose memory behavior is under investigation. Versions are taken from `requirements.txt` (the pinned install manifest):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | 4.0.4 | Web framework — ORM, signals, settings, middleware |
| pip | djangorestframework | 3.13.1 | REST API — serializers, views, upload validation |
| pip | django-q | 1.3.9 | Task queue — worker pool, task scheduling, recycling |
| pip | scikit-learn | 1.0.2 | ML classification — MLPClassifier, CountVectorizer, binarizers |
| pip | ocrmypdf | 13.4.3 | OCR processing — PDF text extraction, archive generation |
| pip | pikepdf | 5.1.1 | PDF manipulation — page splitting for barcode separation |
| pip | pillow | 9.1.0 | Image processing — thumbnail generation, alpha layer removal |
| pip | pdfminer.six | 20220319 | PDF text extraction — fallback when sidecar unavailable |
| pip | pdf2image | 1.16.0 | PDF-to-image conversion — barcode scanning page rasterization |
| pip | pyzbar | 0.1.9 | Barcode detection — separator page identification |
| pip | whoosh | 2.7.4 | Full-text search — index creation, querying, autocomplete |
| pip | dateparser | 1.1.1 | Date parsing — regex-matched date string interpretation |
| pip | fuzzywuzzy | 0.18.0 | Fuzzy matching — partial_ratio comparison on full document content |
| pip | python-magic | 0.4.25 | MIME type detection — file and buffer type identification |
| pip | redis | 3.5.3 | Message broker — task queue backend, channel layers |
| pip | channels | 3.0.4 | WebSocket — real-time status update broadcasting |
| pip | channels-redis | 3.4.0 | Channel layer backend — Redis-backed channel communication |
| pip | tika | 1.24 | Document parsing — HTTP-based text/metadata extraction |
| pip | filelock | 3.6.0 | File locking — MEDIA_LOCK for concurrent file operations |
| pip | imap-tools | 0.54.0 | Email — IMAP mailbox access and message processing |
| pip | watchdog | 2.1.7 | Filesystem monitoring — consumption directory watching |
| pip | numpy | 1.22.3 | Numerical computing — scikit-learn dependency |
| pip | scipy | 1.8.0 | Scientific computing — scikit-learn dependency |
| pip | sphinx | 4.5.0 | Documentation — existing docs build system (dev only) |
| pip | sphinx_rtd_theme | * | Documentation theme — Read the Docs theme (dev only) |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The output is a new standalone document that does not modify or relocate existing documentation. All internal references within the analysis document will use repository-relative paths (e.g., `src/documents/consumer.py:103`).

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

Current coverage analysis of memory-related documentation:

- Public APIs with memory behavior documented: 0/15 (0%) — no existing memory documentation
- Document processing stages analyzed: Target 12/12 (100%) — every stage from file validation through post-consumption cleanup
- Parser implementations analyzed: Target 3/3 (100%) — Tesseract, Text, Tika parsers
- Configuration parameters with memory impact documented: Target 8/8 (100%) — `TASK_WORKERS`, `THREADS_PER_WORKER`, `CONVERT_MEMORY_LIMIT`, `OCR_MAX_IMAGE_PIXELS`, `CONSUMER_ENABLE_BARCODES`, `OCR_MODE`, `OCR_PAGES`, `OPTIMIZE_THUMBNAILS`

Target coverage: 100% of all user questions answered with code-evidenced findings

Coverage gaps to address:
- **Consumer pipeline**: Currently 0% documented for memory behavior, target 100%
- **Classifier lifecycle**: Currently 0% documented for memory impact, target 100%
- **Parser comparison**: Currently 0% documented for memory profiles, target 100%
- **Signal handler cascade**: Currently 0% documented for reference retention, target 100%
- **Configuration impact**: Currently 0% documented for memory implications, target 100%

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements**:
  - Every user question must be answered with specific code evidence
  - Every identified memory hotspot must include file path, line number, and explanation of the mechanism
  - Every behavioral difference (document type, batch size, processing stage) must be explained with code-path comparisons
  - The analysis must distinguish between normal Python GC behavior and problematic patterns with specific reasoning

- **Accuracy validation**:
  - All code citations must reference actual lines in the repository (verified during analysis)
  - All memory estimates must be based on observable code patterns, not assumptions
  - Parser behavior must be traced through actual parse() method implementations
  - Configuration effects must be traced through settings.py to their usage sites

- **Clarity standards**:
  - Technical accuracy with accessible language for developers investigating memory issues
  - Progressive disclosure: executive summary first, then detailed per-component analysis
  - Consistent use of Mermaid diagrams for complex flow visualization
  - Clear separation between "what the code does" and "what this means for memory"

- **Maintainability**:
  - Source citations in `Source: path/to/file.py:LineNumber` format for traceability
  - Organized by component for future targeted updates
  - Findings tagged to specific Paperless-ngx v1.7.0 codebase (commit `542221a38dff`)

### 0.7.3 Example and Diagram Requirements

- Minimum Mermaid diagrams: 4 (consumption pipeline, classifier loading, signal cascade, memory timeline)
- Code excerpts: Short (2-3 line) snippets for each identified memory hotspot, cited with file path and line numbers
- Comparison tables: At least 2 (parser memory profiles, spiking vs non-spiking conditions)
- All findings must include rationale/thinking as required by the `SWE-AtlasQnA-Repo` rule

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/paperless-ngx_542221a38dff.md` — Complete memory analysis document

- **Source code analyzed for documentation (read-only)**:
  - `src/documents/consumer.py` — Full consumption pipeline memory flow
  - `src/documents/classifier.py` — Classifier model loading and training memory
  - `src/documents/tasks.py` — Task orchestration, barcode processing, reindexing
  - `src/documents/parsers.py` — Base parser, date parsing, image conversion
  - `src/paperless_tesseract/parsers.py` — OCR pipeline memory (PIL, ocrmypdf, pdfminer)
  - `src/paperless_text/parsers.py` — Text parser baseline memory
  - `src/paperless_tika/parsers.py` — Tika HTTP pipeline memory
  - `src/documents/signals/handlers.py` — Post-consumption handler chain
  - `src/documents/signals/__init__.py` — Signal definitions
  - `src/documents/matching.py` — Rule-based and fuzzy matching memory
  - `src/documents/index.py` — Whoosh search index memory
  - `src/documents/models.py` — Document model properties and file access
  - `src/documents/file_handling.py` — Filename generation with template formatting
  - `src/documents/serialisers.py` — Upload validation file read
  - `src/documents/sanity_checker.py` — Integrity check file reads
  - `src/documents/loggers.py` — Logging mixin
  - `src/documents/views.py` — Metadata extraction endpoint
  - `src/paperless/settings.py` — Worker pool, memory limits, OCR, task queue
  - `src/paperless_mail/mail.py` — Email attachment processing
  - `src/documents/management/commands/document_consumer.py` — Filesystem consumer
  - `src/documents/management/commands/document_importer.py` — Bulk import

- **Configuration and dependency files analyzed**:
  - `Pipfile` — Python dependency declarations
  - `requirements.txt` — Pinned dependency versions
  - `src/setup.cfg` — Test and coverage configuration
  - `.readthedocs.yml` — Documentation build configuration
  - `docs/conf.py` — Sphinx configuration
  - `docs/troubleshooting.rst` — Existing troubleshooting content

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any file in `src/`, `docs/`, or any other repository directory. The user explicitly forbids source file modifications.
- **Test file modifications**: No changes to `src/documents/tests/` or any test directory
- **Existing documentation updates**: No modifications to `docs/*.rst` files, `README.md`, `CONTRIBUTING.md`, or any other existing documentation
- **Feature additions or code refactoring**: No code changes of any kind
- **Deployment configuration changes**: No changes to `Dockerfile`, `docker/`, `docker-compose.yml`, or deployment scripts
- **Frontend analysis**: The Angular SPA (`src-ui/`) is not relevant to the backend memory investigation
- **Database schema analysis**: Migration files are not relevant to runtime memory behavior
- **Security analysis**: Not in scope for this memory investigation
- **Performance optimization implementation**: The document will identify issues and describe patterns but will not implement fixes (per user instruction)
- **Temporary test scripts**: While the user permits temporary scripts for analysis, the final deliverable is documentation only — any temporary artifacts must be cleaned up

## 0.9 Rules for Documentation

The following rules and requirements are explicitly mandated for this documentation task:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `paperless-ngx_542221a38dff.md` that comprehensively answers the questions posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base all answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.

- **No Source Modifications**: "Don't modify any repository source files." — This is a firm user directive. All existing files in the repository must remain unchanged.

- **Temporary Scripts Cleanup**: "You can create temporary test scripts or helper tools to reproduce and analyze the behavior, but clean them up and leave the codebase unchanged when done." — Any temporary artifacts created during analysis must be removed before completion.

- **Evidence-Based Findings**: All conclusions about memory behavior must be grounded in specific code paths, file references, and line numbers. No speculative claims without code evidence.

- **Comprehensive Coverage**: The document must address ALL user questions:
  - What's causing memory spikes during document import?
  - Is metadata handling creating unnecessary copies or holding references?
  - Is there caching behavior accumulating data unexpectedly?
  - What's different between spiking and non-spiking cases?
  - How does behavior change with different document types or batch sizes?
  - Which components/methods are responsible for holding onto memory?
  - Is this normal Python GC behavior or something problematic?

- **Code Citation Format**: Every technical finding must reference the source file and relevant line numbers using the format: `Source: path/to/file.py:LineNumber`

- **Markdown Format**: The output document must be in Markdown format with proper headers, code blocks, tables, and Mermaid diagrams for visual clarity

## 0.10 References

### 0.10.1 Codebase Files and Folders Searched

The following files were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `src/documents/consumer.py` | Primary document consumption pipeline — identified 4 full-file reads, classifier loading, signal dispatch, and file copy operations |
| `src/documents/classifier.py` | ML classifier model — identified 6 pickle.load() calls, training data collection from all documents, scikit-learn model instantiation |
| `src/documents/tasks.py` | Task orchestration — identified barcode scanning via convert_from_path(), PDF splitting with pikepdf, consume_file() entry point |
| `src/documents/parsers.py` | Base parser class — identified tempdir management, date regex iteration over full text, subprocess spawning for ImageMagick/Ghostscript |
| `src/paperless_tesseract/parsers.py` | OCR parser — identified PIL image processing (alpha removal creates 2 images), pdfminer text extraction, ocrmypdf thread spawning |
| `src/paperless_text/parsers.py` | Text parser — identified lightweight file read, PIL image creation for thumbnail |
| `src/paperless_tika/parsers.py` | Tika parser — identified HTTP response content in memory, double file parsing (tika + gotenberg) |
| `src/documents/signals/__init__.py` | Signal definitions — identified 3 signals: document_consumption_started, document_consumption_finished, document_consumer_declaration |
| `src/documents/signals/handlers.py` | Signal receivers — identified 7 handlers triggered on consumption: set_correspondent, set_document_type, set_tags, add_inbox_tags, set_log_entry, add_to_index, update_filename_and_move_files |
| `src/documents/matching.py` | Matching engine — identified full-content regex operations, fuzzy matching with re.sub on entire document content |
| `src/documents/index.py` | Search index — identified Whoosh AsyncWriter, DelayedQuery.saved_results caching, open_index_writer context manager |
| `src/documents/models.py` | Document model — identified source_file/archive_file/thumbnail_file properties returning open file handles, content TextField |
| `src/documents/file_handling.py` | File handling — identified many_to_dictionary() loading all tags, generate_filename() template formatting |
| `src/documents/serialisers.py` | API serializers — identified validate_document() reading entire upload into memory at line 451 |
| `src/documents/sanity_checker.py` | Sanity checker — identified full media directory walk, MD5 checksum reads of all files |
| `src/documents/views.py` | API views — identified get_metadata() instantiating parser for metadata extraction, cache_control decorator |
| `src/documents/loggers.py` | Logging mixin — confirmed lightweight (no memory impact) |
| `src/documents/__init__.py` | Package init — confirmed system check registration only |
| `src/documents/apps.py` | App config — confirmed signal handler registration at startup |
| `src/paperless/settings.py` | Django settings — identified Q_CLUSTER recycle:1, TASK_WORKERS, THREADS_PER_WORKER, CONVERT_MEMORY_LIMIT, OCR_MAX_IMAGE_PIXELS, OMP_THREAD_LIMIT |
| `src/paperless_mail/mail.py` | Mail handler — identified attachment payload in memory, temporary file creation, async_task dispatch |
| `src/documents/management/commands/document_consumer.py` | Filesystem consumer — identified inotify/polling file watch, async_task dispatch pattern |
| `src/documents/management/commands/document_importer.py` | Bulk importer — identified manifest loading, signal disabling, file copy pattern |
| `Pipfile` | Python dependencies — identified all runtime and dev package versions |
| `requirements.txt` | Pinned dependencies — identified exact versions for all 113 packages |
| `src/setup.cfg` | Test config — identified pytest settings, PAPERLESS_DISABLE_DBHANDLER flag |
| `.readthedocs.yml` | Docs config — identified Python 3.8 for docs build, Sphinx configuration path |
| `docs/conf.py` | Sphinx config — identified extensions, theme, version extraction from version.py |
| `docs/troubleshooting.rst` | Existing troubleshooting — confirmed no memory-related documentation exists |
| `src/paperless_tesseract/apps.py` | Tesseract app config (folder listing) |
| `src/paperless_tesseract/signals.py` | Tesseract signal registration (folder listing) |
| `src/paperless_text/signals.py` | Text parser signal registration (folder listing) |
| `src/paperless_tika/signals.py` | Tika signal registration (folder listing) |

### 0.10.2 Technical Specification Sections Consulted

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project overview, version (v1.7.0), stakeholders, value proposition |
| 1.3 Scope | In-scope features, system boundaries, deployment configurations, data domains, Python 3.8/3.9 support |

### 0.10.3 Attachments

No attachments were provided by the user. No Figma designs, screenshots, or external documents were referenced.

### 0.10.4 External References

No external URLs or web searches were required. All analysis is derived from repository code inspection and the existing technical specification.

