# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that performs a deep investigative analysis of the Paperless-ngx machine learning pipeline's behavior during test execution. The user is debugging non-deterministic test failures in document classification tests and needs a comprehensive, code-grounded reference document that answers specific behavioral questions about the classifier, OCR subprocess, correspondent matching, and barcode-splitting subsystems as they operate inside the test harness.

**Documentation Type:** Technical investigation / Q&A reference document

**Category:** Create new documentation

**Requirements with Enhanced Clarity:**

- **R-1 — Classifier Reuse vs. Retrain Behavior:** Determine whether `DocumentClassifier` reuses an existing persisted model file or retrains from scratch during a single test run, and trace the code path that makes that decision. The critical decision point is `classifier.py` line 163 (`if self.data_hash and new_data_hash == self.data_hash: return False`) and the `load_classifier()` function (lines 30–57) which checks `settings.MODEL_FILE` existence.
- **R-2 — Automatic Correspondent Matching — Training Data Volume and Timing:** Document how many `Document` records are created (the training corpus), when `train_classifier()` is invoked relative to those inserts, and what confidence threshold governs accept/reject of predictions. The classifier in `classifier.py` uses `MLPClassifier.predict()` (no probability threshold — returns class directly; lines 253–256) and `-1` as the null class.
- **R-3 — OCR Edge Case — No Extractable Text:** Trace the code path for a document with no extractable text: which OCR subprocess is invoked (OCRmyPDF Python API at `parsers.py` line 261, not a shell subprocess), what MIME type is assigned (via `magic.from_file()` in `consumer.py` line 219), and what fallback strategies are used (force OCR, then pdfminer, then empty string).
- **R-4 — Barcode Splitting — Document Record Creation:** Document how many document records result from a single input when barcode splitting is active (`settings.CONSUMER_ENABLE_BARCODES`), which barcode values trigger a split (`settings.CONSUMER_BARCODE_STRING`, default `"PATCHT"`), and where in the code that decision is made (`tasks.py` lines 96–110, 184–233).
- **R-5 — Training Data Contamination During Tests:** Analyze whether barcode splitting during a test run changes the effective training data by creating additional document records that alter subsequent classifier retraining results.

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**
- **Read-only investigation:** "Please do not modify the source code." The user explicitly prohibits changes to the existing repository files.
- **Temporary helpers allowed with cleanup:** "You may create temporary helpers while investigating, but clean up anything temporary before finishing."
- **Implementation rule — SWE-AtlasQnA-Repo:** The output must be a new markdown document named `paperless-ngx_542221a38dff.md` placed in `blitzy/documentation/`. No existing files may be modified.
- **Evidence-based answers:** "Do not make assumptions, base your answers on the code as the truth." Every claim must cite specific file paths and line numbers.
- **Provide rationale:** "Provide thinking / rationale behind the answers."

**Style Preferences:**
- Analytical/investigative tone
- Code-grounded with file:line citations
- Structured as Q&A sections addressing each investigation area
- Mermaid diagrams for flow visualization where helpful

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer R-1** (classifier reuse/retrain), we will **create** a section in `blitzy/documentation/paperless-ngx_542221a38dff.md` that traces the execution flow through `src/documents/classifier.py` (`load_classifier()`, `DocumentClassifier.train()`, data-hash comparison), `src/documents/tasks.py` (`train_classifier()`), and `src/documents/consumer.py` (line 292 where `load_classifier()` is called during consumption), relating these to the test fixtures in `src/documents/tests/utils.py` (`DirectoriesMixin`, `setup_directories()`) which create a fresh `MODEL_FILE` path per test.
- To **answer R-2** (correspondent matching), we will **create** a section documenting the signal-driven matching chain: `src/documents/apps.py` (signal connections), `src/documents/signals/handlers.py` (`set_correspondent`, `set_document_type`, `set_tags`), `src/documents/matching.py` (`match_correspondents`, `match_document_types`, `match_tags`), and `src/documents/classifier.py` (`predict_correspondent`, `predict_document_type`, `predict_tags`), noting the absence of a probability threshold.
- To **answer R-3** (OCR/no-text edge case), we will **create** a section tracing `src/paperless_tesseract/parsers.py` (`RasterisedDocumentParser.parse()`) through its OCR, fallback-OCR, and empty-text paths, including MIME type detection via `python-magic` in `src/documents/consumer.py`.
- To **answer R-4** (barcode splitting), we will **create** a section documenting `src/documents/tasks.py` (`consume_file`, `scan_file_for_separating_barcodes`, `separate_pages`, `barcode_reader`) and the `pyzbar` decode path, including how split fragments re-enter consumption.
- To **answer R-5** (training data contamination), we will **create** a section analyzing whether barcode-split documents, once consumed, become part of the training corpus for subsequent classifier operations within the same test run.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Test isolation mechanism:** `DirectoriesMixin` in `src/documents/tests/utils.py` creates temporary directories for each test, including a unique `MODEL_FILE` path. This means each test class starts without a persisted classifier model — a critical detail for understanding non-determinism.
- **Signal handler registration:** `src/documents/apps.py` connects `document_consumption_finished` to six handlers (inbox tags, correspondent, document type, tags, log entry, index). The order of these handlers can affect test outcomes.
- **Training data filtering:** `classifier.py` line 125–127 excludes documents tagged with `is_inbox_tag=True` from training data. If a test creates inbox-tagged documents, they will not participate in training.
- **Hash-based retrain skipping:** `classifier.py` lines 162–164 compare SHA-1 hashes of training data to skip retraining when data is unchanged. This is a source of determinism (same data → same model) but also a source of confusion if data changes between test steps.
- **MLPClassifier non-determinism:** scikit-learn's `MLPClassifier` uses random weight initialization by default. The `tol=0.01` parameter affects convergence but does not eliminate randomness across runs — a likely root cause of non-deterministic test failures.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with reStructuredText (`.rst`) source files and Read the Docs hosting.

**Documentation Framework:** Sphinx ~4.5.0 (from `Pipfile` dev-packages)
**Documentation Generator Configuration:** `docs/conf.py`
**Documentation Build System:** `docs/Makefile` (standard Sphinx targets: html, epub, latexpdf, linkcheck, doctest)
**Documentation Container Preview:** `docs/Dockerfile` (builds and serves HTML via Python HTTP server on port 8000)
**Hosting:** Read the Docs (configured via `.readthedocs.yml` pointing to `docs/conf.py`)
**Theme:** `sphinx_rtd_theme` (from `Pipfile` dev-packages)
**Diagram Tools Detected:** None natively configured in docs (Mermaid diagrams will be used in the new markdown document)

**Existing Documentation Pages (`.rst` files in `docs/`):**

| File | Topic | Relevance to This Task |
|------|-------|------------------------|
| `docs/index.rst` | Landing page / navigation hub | Low — general entry point |
| `docs/configuration.rst` | Environment variables and runtime settings | Medium — documents `CONSUMER_BARCODE_STRING`, `CONSUMER_ENABLE_BARCODES`, OCR settings |
| `docs/advanced_usage.rst` | Matching, hooks, filename handling | High — documents matching algorithms and auto-classification |
| `docs/extending.rst` | Contributor workflows, dev setup, parser extension | Medium — documents parser architecture |
| `docs/api.rst` | REST API endpoints | Low — not directly relevant to test behavior |
| `docs/troubleshooting.rst` | Operational failures and fixes | Low — runtime troubleshooting, not test debugging |
| `docs/administration.rst` | Backups, updates, indexing, archiving | Low — operational admin tasks |
| `docs/usage_overview.rst` | Product model, ingestion, search | Medium — high-level pipeline description |

**Key Finding:** No existing documentation covers the internal behavior of the ML pipeline during test execution, the classifier's retrain-vs-reuse decision logic, or the interaction between barcode splitting and classification training data. The new document fills a previously undocumented area.

### 0.2.2 Repository Code Analysis for Documentation

**Search patterns used for code to document:**

- Classifier pipeline: `src/documents/classifier.py` — `DocumentClassifier` class, `load_classifier()`, `preprocess_content()`
- Matching pipeline: `src/documents/matching.py` — `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`
- Consumer pipeline: `src/documents/consumer.py` — `Consumer.try_consume_file()`, `_store()`, classifier loading at line 292
- Task orchestration: `src/documents/tasks.py` — `train_classifier()`, `consume_file()`, `barcode_reader()`, `scan_file_for_separating_barcodes()`, `separate_pages()`
- Signal handlers: `src/documents/signals/handlers.py` — `set_correspondent()`, `set_document_type()`, `set_tags()`, `add_inbox_tags()`
- Signal definitions: `src/documents/signals/__init__.py` — `document_consumption_finished`, `document_consumer_declaration`
- App wiring: `src/documents/apps.py` — `DocumentsConfig.ready()` signal connections
- OCR parser: `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser.parse()`, `extract_text()`, `construct_ocrmypdf_parameters()`
- OCR checks: `src/paperless_tesseract/checks.py` — `get_tesseract_langs()` (uses `subprocess.Popen(["tesseract", "--list-langs"])`)
- Parser registration: `src/paperless_tesseract/signals.py` — `tesseract_consumer_declaration()` (MIME type map)
- Models: `src/documents/models.py` — `MatchingModel`, `Document`, `Correspondent`, `Tag`, `DocumentType`
- Test infrastructure: `src/documents/tests/utils.py` — `DirectoriesMixin`, `setup_directories()`
- Test suites: `src/documents/tests/test_classifier.py`, `src/documents/tests/test_tasks.py`, `src/documents/tests/test_consumer.py`, `src/documents/tests/test_matchables.py`
- Base parser: `src/documents/parsers.py` — `DocumentParser`, `get_parser_class_for_mime_type()`
- Project config: `src/setup.cfg` — pytest settings (`PAPERLESS_DISABLE_DBHANDLER=true`)
- Dependencies: `Pipfile` — `scikit-learn==1.0.2`, `ocrmypdf~=13.4`, `pyzbar`, `pdf2image`

**Key directories examined:**
- `src/documents/` — Core domain app (classifier, consumer, matching, tasks, models, signals)
- `src/documents/tests/` — Test suite (test_classifier, test_tasks, test_consumer, test_matchables)
- `src/documents/tests/samples/barcodes/` — Barcode test fixtures (18 files: PNG, PBM, PDF)
- `src/paperless_tesseract/` — OCR parser integration
- `src/paperless_text/` — Plain text parser (weight 10, MIME: text/plain, text/csv)
- `docs/` — Sphinx documentation workspace

### 0.2.3 Web Search Research Conducted

No web search was necessary for this task. All answers are derived from the codebase itself, which is the authoritative source per the user's instruction: "Do not make assumptions, base your answers on the code as the truth." The codebase provides complete visibility into classifier behavior, OCR flow, barcode splitting logic, and test infrastructure.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `src/documents/classifier.py` — ML Classifier**
- Public APIs: `DocumentClassifier` (class), `load_classifier()`, `preprocess_content()`, `DocumentClassifier.train()`, `.predict_correspondent()`, `.predict_document_type()`, `.predict_tags()`, `.load()`, `.save()`
- Current documentation: The Sphinx docs in `docs/advanced_usage.rst` describe auto-matching at a user level but do not cover internal classifier mechanics, retrain decision logic, data-hash comparison, or test-time behavior.
- Documentation needed: Detailed trace of `train()` flow (data gathering → hashing → vectorization → MLPClassifier fitting), retrain-skip logic, and `FORMAT_VERSION` gating.

**Module: `src/documents/matching.py` — Rule-Based + ML Matching**
- Public APIs: `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`
- Current documentation: `docs/advanced_usage.rst` lists matching algorithms (Any, All, Literal, Regex, Fuzzy, Auto) at a user level.
- Documentation needed: How `MATCH_AUTO` delegates to the classifier, how rule-based and ML results are combined via `filter(lambda o: matches(o, document) or o.pk == pred_id, ...)`, and the absence of a confidence threshold.

**Module: `src/documents/consumer.py` — Document Consumption Pipeline**
- Public APIs: `Consumer.try_consume_file()`, `Consumer._store()`, `Consumer.apply_overrides()`
- Current documentation: `docs/usage_overview.rst` describes ingestion conceptually.
- Documentation needed: The precise point where `load_classifier()` is called (line 292, after parsing, before `_store()`), and how the classifier is passed to `document_consumption_finished` signal.

**Module: `src/documents/tasks.py` — Task Orchestration**
- Public APIs: `train_classifier()`, `consume_file()`, `barcode_reader()`, `scan_file_for_separating_barcodes()`, `separate_pages()`, `save_to_dir()`
- Current documentation: `docs/configuration.rst` documents barcode settings at user level.
- Documentation needed: Detailed trace of barcode scan → split → re-ingest flow, training-data-presence gate in `train_classifier()`, and how split fragments create new Document records.

**Module: `src/paperless_tesseract/parsers.py` — OCR Parser**
- Public APIs: `RasterisedDocumentParser.parse()`, `.extract_text()`, `.construct_ocrmypdf_parameters()`
- Current documentation: `docs/configuration.rst` documents OCR settings.
- Documentation needed: The exact fallback chain for no-text documents (OCRmyPDF → force OCR → pdfminer → empty string), and that OCRmyPDF is invoked as a Python library call (`ocrmypdf.ocr(**args)`) not a subprocess.

**Module: `src/documents/tests/utils.py` — Test Infrastructure**
- Public APIs: `DirectoriesMixin`, `setup_directories()`, `remove_dirs()`
- Current documentation: None.
- Documentation needed: How temporary directories and `MODEL_FILE` path isolation prevent cross-test classifier state leakage.

**Module: `src/documents/signals/handlers.py` — Signal Handlers**
- Public APIs: `set_correspondent()`, `set_document_type()`, `set_tags()`, `add_inbox_tags()`, `set_log_entry()`, `add_to_index()`
- Current documentation: None at the internal level.
- Documentation needed: Handler execution order, how classifier is passed through signal kwargs, and the side effects each handler applies to the Document instance.

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internal behavior:** No existing document explains the classifier's retrain-vs-reuse decision path, the SHA-1 data-hash mechanism, or the `FORMAT_VERSION` gating for model compatibility.
- **Missing test infrastructure documentation:** The `DirectoriesMixin` pattern that creates isolated temp directories (including a unique `MODEL_FILE`) per test class is undocumented.
- **Absent training-data lifecycle documentation:** No document traces which `Document` records participate in training (excludes `is_inbox_tag=True`, requires `matching_algorithm == MATCH_AUTO`), when training occurs relative to document creation, and whether barcode splitting alters the training corpus.
- **No OCR fallback chain documentation:** The three-tier fallback (OCRmyPDF → force-OCR retry → pdfminer extraction → empty string) is not documented as a unified flow.
- **No confidence threshold documentation:** The fact that the classifier uses hard class prediction (`predict()`) rather than probability-based acceptance (`predict_proba()`) is not documented anywhere, and is critical to understanding matching behavior.
- **Missing signal handler chain documentation:** The order in which `document_consumption_finished` handlers fire and the cascading effects on Document state are undocumented.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/paperless-ngx_542221a38dff.md` will be structured as follows:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── # ML Pipeline Behavior During Test Execution
        ├── ## 1. Classifier Reuse vs. Retrain Behavior
        │   ├── ### 1.1 Test Isolation: DirectoriesMixin and MODEL_FILE
        │   ├── ### 1.2 load_classifier() Decision Path
        │   ├── ### 1.3 train() Data-Hash Comparison
        │   ├── ### 1.4 MLPClassifier Non-Determinism
        │   └── (Mermaid: classifier decision flowchart)
        ├── ## 2. Automatic Correspondent Matching
        │   ├── ### 2.1 Training Data: Which Documents, How Many
        │   ├── ### 2.2 Training Timing Relative to Inserts
        │   ├── ### 2.3 Confidence Threshold (Absence Thereof)
        │   ├── ### 2.4 Signal Handler Chain
        │   └── (Mermaid: consumption-to-matching sequence diagram)
        ├── ## 3. OCR Edge Case: No Extractable Text
        │   ├── ### 3.1 OCR Invocation Path (Python API, Not Subprocess)
        │   ├── ### 3.2 Fallback Chain
        │   ├── ### 3.3 MIME Type Assignment
        │   └── (Mermaid: OCR fallback flowchart)
        ├── ## 4. Barcode Splitting
        │   ├── ### 4.1 Splitting Decision Point
        │   ├── ### 4.2 Barcode Values That Trigger Splits
        │   ├── ### 4.3 Document Record Count From Single Input
        │   ├── ### 4.4 Impact on Training Data
        │   └── (Mermaid: barcode split pipeline diagram)
        └── ## 5. Non-Determinism Root Causes
            ├── ### 5.1 MLPClassifier Random Initialization
            ├── ### 5.2 Test Ordering and State Leakage
            └── ### 5.3 Recommendations for Deterministic Testing
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract classifier decision logic from `src/documents/classifier.py` lines 115–249 using direct code analysis
- Extract matching pipeline behavior from `src/documents/matching.py` lines 21–57 and `src/documents/signals/handlers.py` lines 35–230
- Extract OCR fallback chain from `src/paperless_tesseract/parsers.py` lines 230–328
- Extract barcode splitting logic from `src/documents/tasks.py` lines 75–233
- Generate examples by analyzing test fixtures in `src/documents/tests/test_classifier.py` (especially `generate_test_data()` at lines 25–82) and `src/documents/tests/test_tasks.py`
- Create flowcharts by mapping decision branches discovered in the source code

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for classifier decision flow, consumption-matching sequence, OCR fallback chain, and barcode split pipeline
- Code path citations as inline references: `Source: src/documents/classifier.py:163`
- Tables for parameter inventories and training data composition
- Thinking/rationale paragraphs before each conclusion

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Classifier decision flowchart:** Shows `load_classifier()` → file exists? → load → version check → `train()` → hash compare → retrain or skip. Covers the full decision tree from `classifier.py`.
- **Consumption-to-matching sequence diagram:** Shows `Consumer.try_consume_file()` → parse → `load_classifier()` → `_store()` → `document_consumption_finished` signal → handler chain (`add_inbox_tags` → `set_correspondent` → `set_document_type` → `set_tags` → `set_log_entry` → `add_to_index`).
- **OCR fallback flowchart:** Shows `parse()` → `ocrmypdf.ocr()` → success path / `NoTextFoundException` → force-OCR retry → pdfminer fallback → empty string.
- **Barcode split pipeline diagram:** Shows `consume_file()` → `CONSUMER_ENABLE_BARCODES` check → `scan_file_for_separating_barcodes()` → `separate_pages()` → `save_to_dir()` → re-consumption loop.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/classifier.py`, `src/documents/matching.py`, `src/documents/consumer.py`, `src/documents/tasks.py`, `src/paperless_tesseract/parsers.py`, `src/documents/signals/handlers.py`, `src/documents/tests/test_classifier.py`, `src/documents/tests/test_tasks.py`, `src/documents/tests/test_consumer.py`, `src/documents/tests/test_matchables.py`, `src/documents/tests/utils.py`, `src/documents/models.py`, `src/documents/apps.py`, `src/documents/signals/__init__.py`, `src/paperless_tesseract/signals.py`, `src/paperless_tesseract/checks.py` | Comprehensive investigative document answering all five research questions about ML pipeline test behavior: classifier reuse/retrain, correspondent matching mechanics, OCR no-text handling, barcode splitting document creation, and training data contamination analysis. Includes Mermaid diagrams, code-path citations, and rationale. |

**No other files are created, updated, or deleted.** The implementation rule states: "Do not modify any existing files in the source repository." The single output is the new markdown investigation document.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Q&A Reference
Source Code:
  - src/documents/classifier.py (primary — classifier logic)
  - src/documents/matching.py (primary — matching pipeline)
  - src/documents/consumer.py (primary — consumption flow)
  - src/documents/tasks.py (primary — training task, barcode splitting)
  - src/paperless_tesseract/parsers.py (primary — OCR parsing)
  - src/documents/signals/handlers.py (primary — post-consumption handlers)
  - src/documents/apps.py (supporting — signal wiring)
  - src/documents/signals/__init__.py (supporting — signal definitions)
  - src/documents/models.py (supporting — MatchingModel, Document)
  - src/documents/tests/utils.py (supporting — test isolation)
  - src/documents/tests/test_classifier.py (supporting — test patterns)
  - src/documents/tests/test_tasks.py (supporting — barcode/training tests)
  - src/documents/tests/test_consumer.py (supporting — consumer test patterns)
  - src/documents/tests/test_matchables.py (supporting — matching/signal tests)
  - src/paperless_tesseract/signals.py (supporting — MIME type registration)
  - src/paperless_tesseract/checks.py (supporting — Tesseract subprocess)
  - Pipfile (supporting — dependency versions)
  - src/setup.cfg (supporting — pytest configuration)
Sections:
  - Title and Overview (purpose and scope of investigation)
  - Section 1: Classifier Reuse vs. Retrain Behavior
    - 1.1 Test Isolation: DirectoriesMixin and MODEL_FILE (from tests/utils.py:14-50)
    - 1.2 load_classifier() Decision Path (from classifier.py:30-57)
    - 1.3 train() Data-Hash Comparison (from classifier.py:115-249)
    - 1.4 MLPClassifier Non-Determinism (from classifier.py:219,228,238 — tol=0.01)
  - Section 2: Automatic Correspondent Matching
    - 2.1 Training Data: Which Documents, How Many (from classifier.py:125-157)
    - 2.2 Training Timing Relative to Inserts (from tasks.py:48-72)
    - 2.3 Confidence Threshold — Absence (from classifier.py:251-260)
    - 2.4 Signal Handler Chain (from apps.py:11-28, handlers.py:35-98)
  - Section 3: OCR Edge Case — No Extractable Text
    - 3.1 OCR Invocation Path (from parsers.py:230-261)
    - 3.2 Fallback Chain (from parsers.py:266-327)
    - 3.3 MIME Type Assignment (from consumer.py:219, tesseract/signals.py:7-19)
  - Section 4: Barcode Splitting
    - 4.1 Splitting Decision Point (from tasks.py:194-233)
    - 4.2 Barcode Values That Trigger Splits (from tasks.py:96-110)
    - 4.3 Document Record Count From Single Input (from tasks.py:113-161)
    - 4.4 Impact on Training Data (analysis of re-consumption path)
  - Section 5: Non-Determinism Root Causes and Recommendations
Diagrams:
  - Mermaid flowchart: Classifier retrain-vs-reuse decision tree
  - Mermaid sequence diagram: Consumption-to-matching signal chain
  - Mermaid flowchart: OCR three-tier fallback chain
  - Mermaid flowchart: Barcode split pipeline and re-ingestion
Key Citations:
  src/documents/classifier.py, src/documents/matching.py,
  src/documents/consumer.py, src/documents/tasks.py,
  src/paperless_tesseract/parsers.py, src/documents/signals/handlers.py,
  src/documents/apps.py, src/documents/models.py,
  src/documents/tests/utils.py, src/documents/tests/test_classifier.py,
  src/documents/tests/test_tasks.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new file is a standalone markdown document placed in `blitzy/documentation/` as specified by the implementation rule. It does not integrate with the existing Sphinx documentation system at `docs/`.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The new document is self-contained.
- **No navigation links:** It is not part of the Sphinx `docs/` tree.
- **No table of contents updates:** No `index.rst` or `mkdocs.yml` changes needed.
- **No glossary updates:** All terms are defined inline in the document.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to the documentation exercise because the investigation traces code paths through these libraries. Versions are extracted from `Pipfile` (the project's dependency manifest).

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | scikit-learn | ==1.0.2 | ML classifier (MLPClassifier, CountVectorizer, LabelBinarizer, MultiLabelBinarizer) — pinned to prevent model compatibility issues |
| pip | ocrmypdf | ~=13.4 | OCR processing library invoked via Python API (`ocrmypdf.ocr()`) in `paperless_tesseract/parsers.py` |
| pip | pyzbar | * (latest) | Barcode detection library used by `tasks.barcode_reader()` |
| pip | pdf2image | * (latest) | PDF-to-image conversion for barcode scanning (`convert_from_path()`) |
| pip | pikepdf | ~=5.1 | PDF splitting via `pikepdf.Pdf.open()` and `.pages` manipulation |
| pip | python-magic | * (latest) | MIME type detection via `magic.from_file(path, mime=True)` in consumer |
| pip | django | ~=4.0 | Web framework, ORM, signal dispatch, test infrastructure |
| pip | djangorestframework | ~=3.13 | REST API framework |
| pip | fuzzywuzzy | * (latest, with speedup extras) | Fuzzy matching algorithm (`MATCH_FUZZY`, threshold 90) |
| pip | pdfminer.six | * (latest) | PDF text extraction fallback when sidecar file is incomplete |
| pip | pillow | ~=9.1 | Image handling for barcode processing and OCR DPI detection |
| pip | filelock | * (latest) | Concurrency safety for media file operations |
| pip | channels | ~=3.0 | WebSocket layer for status updates during consumption |
| pip | pytest | * (latest, dev) | Test runner configured in `setup.cfg` |
| pip | pytest-django | * (latest, dev) | Django test integration |
| pip | pytest-xdist | * (latest, dev) | Parallel test execution (`--numprocesses auto`) |
| pip | factory-boy | * (latest, dev) | Test data factories (`CorrespondentFactory`, `DocumentFactory`) |

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The new document is a standalone markdown file that does not modify or cross-reference existing documentation links. All references within the document point to source code file paths, not to other documentation files.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the investigation topics:**

| Topic | Source Files Analyzed | Coverage Before | Coverage Target |
|-------|----------------------|-----------------|-----------------|
| Classifier retrain/reuse logic | `classifier.py`, `tasks.py`, `tests/test_classifier.py`, `tests/utils.py` | 0% (undocumented internal behavior) | 100% |
| Correspondent matching pipeline | `matching.py`, `signals/handlers.py`, `apps.py`, `consumer.py` | ~20% (user-level docs in `advanced_usage.rst`) | 100% |
| OCR no-text fallback chain | `paperless_tesseract/parsers.py`, `consumer.py` | ~15% (configuration docs in `configuration.rst`) | 100% |
| Barcode splitting mechanics | `tasks.py`, `tests/test_tasks.py` | ~10% (configuration docs in `configuration.rst`) | 100% |
| Training data contamination analysis | `classifier.py`, `tasks.py`, `models.py` | 0% (completely undocumented) | 100% |
| Test isolation infrastructure | `tests/utils.py`, `setup.cfg` | 0% (undocumented) | 100% |
| MLPClassifier non-determinism | `classifier.py` (lines 219, 228, 238) | 0% (undocumented) | 100% |

**Overall target:** 100% coverage of all five investigation questions with code-grounded evidence and rationale.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every investigation question posed by the user has a dedicated section with a definitive answer
- Every answer cites specific file paths and line numbers
- Every decision branch in the classifier, matching, OCR, and barcode code paths is traced and documented
- All Mermaid diagrams accurately reflect the code paths they depict

**Accuracy validation:**
- All code citations must reference actual file paths and line numbers verified against the repository
- All claims about behavior (e.g., "no confidence threshold exists") must be grounded in specific code evidence
- Mermaid diagrams must match the actual branching logic in the source code

**Clarity standards:**
- Each section opens with a thinking/rationale paragraph before presenting the conclusion (per user rule: "Provide thinking / rationale behind the answers")
- Technical accuracy with accessible language — assumes the reader has Python/Django knowledge but may not know Paperless-ngx internals
- Progressive disclosure: high-level answer first, then detailed code trace
- Consistent terminology: "classifier" (not "model"), "correspondent" (not "sender"), "training corpus" (not "dataset")

**Maintainability:**
- Source citations as inline `Source: path/to/file.py:LineNumber` references
- Mermaid diagrams are text-based and version-controllable
- Document is self-contained with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 Mermaid diagrams (classifier decision, matching sequence, OCR fallback, barcode pipeline)
- **Code path traces:** Each section includes a step-by-step code trace with file:line citations
- **Test fixture examples:** Reference concrete test data from `test_classifier.py::generate_test_data()` (3 documents, 3 correspondents, 3 tags, 2 document types) to illustrate training data composition
- **Barcode test examples:** Reference `test_tasks.py` test cases with specific fixture files (e.g., `several-patcht-codes.pdf` with separators at pages [2, 5] producing 3 output documents)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — The sole deliverable: a comprehensive investigative document

**Source code files analyzed (read-only, no modifications):**
- `src/documents/classifier.py` — Classifier class, load, train, predict methods
- `src/documents/matching.py` — Rule-based and ML-assisted matching functions
- `src/documents/consumer.py` — Document consumption pipeline, classifier loading point
- `src/documents/tasks.py` — Training task, barcode reader, page separator, consume_file
- `src/documents/models.py` — MatchingModel (MATCH_AUTO=6), Document, Correspondent, Tag, DocumentType
- `src/documents/parsers.py` — Base DocumentParser, get_parser_class_for_mime_type, parse_date
- `src/documents/apps.py` — Signal handler registration in ready()
- `src/documents/signals/__init__.py` — Signal definitions (document_consumption_finished, etc.)
- `src/documents/signals/handlers.py` — set_correspondent, set_document_type, set_tags, add_inbox_tags
- `src/paperless_tesseract/parsers.py` — RasterisedDocumentParser, OCR fallback chain
- `src/paperless_tesseract/signals.py` — MIME type registration map
- `src/paperless_tesseract/checks.py` — Tesseract language validation subprocess
- `src/paperless_text/signals.py` — Text parser MIME type registration
- `src/documents/tests/utils.py` — DirectoriesMixin, setup_directories, MODEL_FILE isolation
- `src/documents/tests/test_classifier.py` — Classifier test patterns and fixtures
- `src/documents/tests/test_tasks.py` — Barcode and training task tests
- `src/documents/tests/test_consumer.py` — Consumer pipeline tests, DummyParser mock
- `src/documents/tests/test_matchables.py` — Matching algorithm tests and signal tests
- `src/documents/tests/factories.py` — Factory definitions
- `src/documents/tests/samples/barcodes/` — Barcode fixture files (18 files)
- `src/setup.cfg` — Pytest configuration (PAPERLESS_DISABLE_DBHANDLER, parallel execution)
- `Pipfile` — Dependency versions (scikit-learn==1.0.2, ocrmypdf~=13.4, etc.)

**Investigation topics in scope:**
- Classifier retrain-vs-reuse decision during test execution
- Training data composition, volume, and filtering rules
- Training timing relative to document creation
- Absence of confidence/probability threshold in prediction
- OCR invocation mechanism (Python API vs. subprocess)
- OCR fallback chain for no-text documents
- MIME type detection and assignment during consumption
- Barcode splitting decision point and trigger values
- Document record count from barcode-split inputs
- Training data contamination from barcode-split re-ingestion
- MLPClassifier non-determinism as root cause of flaky tests
- Test isolation via DirectoriesMixin temp directories

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** Explicitly prohibited by user instruction — "Please do not modify the source code"
- **Test file modifications:** No changes to any test files
- **Feature additions or refactoring:** No new functionality
- **Temporary helpers:** The user allows creation of temporary helpers during investigation, but all must be cleaned up. The documentation agent will not create temporary helpers; the investigation is purely analytical.
- **Existing documentation updates:** No changes to `docs/*.rst` files or Sphinx configuration
- **Frontend analysis:** The Angular SPA (`src-ui/`) is not relevant to the ML pipeline investigation
- **Email ingestion analysis:** `src/paperless_mail/` is not relevant to the classifier/OCR/barcode investigation
- **Tika parser analysis:** `src/paperless_tika/` is not relevant to this investigation
- **Deployment/infrastructure analysis:** Docker, CI/CD, and deployment configuration are not in scope
- **Performance optimization:** The investigation documents behavior, not performance
- **Runtime configuration changes:** No `paperless.conf` or environment variable modifications


## 0.9 Rules for Documentation

### 0.9.1 User-Specified Rules

The following rules are explicitly emphasized by the user and the implementation rule configuration:

- **Do not modify any existing files in the source repository.** The investigation is read-only. All output goes to the new markdown document.
- **Base answers on the code as the truth.** Every claim in the document must cite specific source files and line numbers. No assumptions or generalizations from external documentation.
- **Provide thinking / rationale behind the answers.** Each section must lead with reasoning before presenting conclusions.
- **Create a new markdown document named `paperless-ngx_542221a38dff.md`.** The filename matches the source branch name as required by the `SWE-AtlasQnA-Repo` implementation rule.
- **Place the generated document in the `blitzy/documentation` directory.** The directory must be created if it does not exist.
- **Temporary helpers are permitted but must be cleaned up.** If any scratch files or helper scripts are created during investigation, they must be removed before the task is complete. The final repository state must contain only the new markdown document and nothing else that was not there before.

### 0.9.2 Derived Documentation Standards

Based on the investigation nature of this task, the following additional standards apply:

- **Citation format:** All code references use `Source: relative/path/to/file.py:LineNumber` or `Source: relative/path/to/file.py:StartLine-EndLine` format.
- **Diagram format:** All diagrams use Mermaid syntax within fenced code blocks (` ```mermaid ... ``` `).
- **Answer structure:** Each investigation question is answered with: (1) Rationale/thinking paragraph, (2) Code-traced answer with citations, (3) Summary conclusion.
- **No speculative content:** If a question cannot be definitively answered from the code, state what the code shows and identify what is ambiguous.
- **Test fixture references:** When discussing test behavior, reference specific test methods (e.g., `TestClassifier.testPredict`) and fixture data (e.g., `generate_test_data()` creates `doc1`, `doc2`, `doc_inbox`).


## 0.10 References

### 0.10.1 Source Code Files Searched and Analyzed

The following files and folders were searched across the codebase to derive all conclusions in this Agent Action Plan:

**Primary Source Files (full content read and analyzed):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `src/documents/classifier.py` | DocumentClassifier class: train(), predict_*(), load(), save(), data-hash logic, FORMAT_VERSION, load_classifier() |
| `src/documents/matching.py` | match_correspondents(), match_document_types(), match_tags(), matches() with all 6 algorithms |
| `src/documents/consumer.py` | Consumer.try_consume_file() pipeline, MIME detection, classifier loading at line 292, signal dispatch |
| `src/documents/tasks.py` | train_classifier(), consume_file(), barcode_reader(), scan_file_for_separating_barcodes(), separate_pages(), save_to_dir() |
| `src/documents/models.py` | MatchingModel (MATCH_AUTO=6), Document, Correspondent, Tag, DocumentType, FileInfo |
| `src/documents/parsers.py` | DocumentParser base class, get_parser_class_for_mime_type(), parse_date(), run_convert() |
| `src/documents/apps.py` | DocumentsConfig.ready() — signal handler registration order |
| `src/documents/signals/__init__.py` | Signal definitions: document_consumption_started, document_consumption_finished, document_consumer_declaration |
| `src/documents/signals/handlers.py` | set_correspondent(), set_document_type(), set_tags(), add_inbox_tags(), set_log_entry(), add_to_index(), cleanup handlers |
| `src/paperless_tesseract/parsers.py` | RasterisedDocumentParser: parse(), extract_text(), construct_ocrmypdf_parameters(), OCR fallback chain |
| `src/paperless_tesseract/signals.py` | tesseract_consumer_declaration() — MIME type map (PDF, JPEG, PNG, TIFF, GIF, BMP) |
| `src/paperless_tesseract/checks.py` | get_tesseract_langs() — subprocess.Popen(["tesseract", "--list-langs"]) |
| `src/paperless_text/signals.py` | text_consumer_declaration() — MIME type map (text/plain, text/csv), weight=10 |
| `src/documents/tests/utils.py` | DirectoriesMixin, setup_directories(), remove_dirs() — test isolation with temp MODEL_FILE |
| `src/documents/tests/test_classifier.py` | TestClassifier: generate_test_data(), testTrain, testPredict, testDatasetHashing, testSaveClassifier, load tests |
| `src/documents/tests/test_tasks.py` | TestTasks: train_classifier tests, barcode_reader tests, scan_file_for_separating_barcodes tests, separate_pages tests, consume_barcode_file test |
| `src/documents/tests/test_consumer.py` | TestConsumer: DummyParser, CopyParser, FaultyParser, testClassifyDocument, testNormalOperation, duplicate tests |
| `src/documents/tests/test_matchables.py` | TestMatching, TestDocumentConsumptionFinishedSignal: tag/correspondent matching via signals |
| `src/documents/tests/factories.py` | CorrespondentFactory, DocumentFactory — factory_boy test fixtures |
| `Pipfile` | Python dependency versions (scikit-learn==1.0.2, ocrmypdf~=13.4, pyzbar, pdf2image, pikepdf, etc.) |
| `src/setup.cfg` | Pytest configuration (DJANGO_SETTINGS_MODULE, PAPERLESS_DISABLE_DBHANDLER, numprocesses auto) |

**Folders Explored:**

| Folder Path | Purpose in Analysis |
|-------------|---------------------|
| `` (root) | Repository structure, build files, dependency manifests |
| `src/` | Python source tree overview |
| `src/documents/` | Core domain app structure |
| `src/documents/tests/` | Test suite inventory |
| `src/documents/tests/samples/barcodes/` | Barcode fixture files (18 files: PNG, PBM, PDF) |
| `src/documents/signals/` | Signal definitions and handlers |
| `src/paperless_tesseract/` | OCR parser package |
| `docs/` | Existing Sphinx documentation |

**Tech Spec Sections Retrieved:**

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project context, version (1.7.0), core problem domain |
| 1.3 Scope | In-scope features, technical requirements (Python 3.8/3.9), deployment variants |
| 2.1 Feature Catalog | Feature details for F-001 (Ingestion), F-002 (OCR), F-004 (Classification), F-011 (Barcode Splitting) |

### 0.10.2 Attachments

No attachments were provided by the user for this project.

### 0.10.3 Figma Screens

No Figma screens were provided or referenced for this project.

### 0.10.4 External URLs

No external URLs were referenced in the user's requirements. All investigation is code-based.


