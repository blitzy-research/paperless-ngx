# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive investigative analysis document** that explains the runtime behavior of the Paperless-ngx machine learning classification pipeline during test execution. The investigation targets non-deterministic test failures in the document classification subsystem and must answer specific behavioral questions by tracing the actual codebase — not by modifying it. The deliverable is a Markdown document placed in `blitzy/documentation/` that serves as a definitive QnA reference.

The specific investigation objectives are:

- **Classifier reuse vs. retraining** — Determine whether the `DocumentClassifier` in `src/documents/classifier.py` reuses a persisted model (loaded from `settings.MODEL_FILE`) or retrains from scratch during a single test run, and identify the exact mechanism (`data_hash` SHA-1 comparison at line 163) that governs that decision.
- **Training document creation during correspondent matching tests** — Quantify how many `Document` records are created when a test exercises automatic correspondent matching (via `MatchingModel.MATCH_AUTO = 6`), identify when `train()` is invoked relative to those database inserts, and document that no confidence threshold exists in the prediction path (`predict_correspondent` at line 251 simply returns the sklearn `predict()` output without any score filtering).
- **No-text edge case handling** — Trace the code path exercised when a document has no extractable text, identifying which OCR subprocess (`ocrmypdf.ocr()` in `src/paperless_tesseract/parsers.py` line 261) is invoked, the fallback strategy (force OCR via `safe_fallback=True`), and what MIME type is assigned to the output (the archive is always `application/pdf`; the original MIME type is detected by `magic.from_file()` in `src/documents/consumer.py` line 219).
- **Barcode splitting document record creation** — Determine how many `Document` records result from a single input PDF when barcode splitting is active (`settings.CONSUMER_ENABLE_BARCODES`), which barcode values trigger a split (default `"PATCHT"` configured via `settings.CONSUMER_BARCODE_STRING`), and where in the code that decision is made (`scan_file_for_separating_barcodes` at `src/documents/tasks.py` line 96 combined with `barcode_reader` at line 75), including whether the resulting split fragments change the effective training data for the classifier during that test run.

Implicit requirements detected:

- The investigation must identify root causes of **non-determinism** in the ML pipeline, which includes the absence of a `random_state` parameter in the `MLPClassifier(tol=0.01)` instantiation at `src/documents/classifier.py` lines 219, 228, and 238.
- The analysis must account for **parallel test execution** configured via `--numprocesses auto` in `src/setup.cfg` line 10, which can cause race conditions on shared state such as the `MODEL_FILE` pickle.
- The investigation must explain the **signal-driven post-consumption flow** defined in `src/documents/apps.py` where `document_consumption_finished` connects to `set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`, `set_log_entry`, and `add_to_index`.

### 0.1.2 Special Instructions and Constraints

- **No source code modification**: The user explicitly states "Please do not modify the source code." The implementation rule `SWE-AtlasQnA-Repo` reinforces this: "Do not modify any existing files in the source repository."
- **Temporary helpers allowed with cleanup**: The user permits creating temporary investigation helpers, but they must be cleaned up before finishing. Per the SWE-AtlasQnA-Repo rule, the only permanent artifact is a Markdown document in `blitzy/documentation/`.
- **Output artifact**: A Markdown document named `<source_branch_name>.md` must be created in `blitzy/documentation/` containing comprehensive answers with rationale grounded in the source code.
- **Evidence-based answers**: The rule mandates: "Do not make assumptions, base your answers on the code as the truth."
- **Architectural convention**: Follow repository conventions. The investigation must reference actual file paths, line numbers, and function signatures.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **determine classifier reuse vs. retraining**, we will trace the `train_classifier()` function in `src/documents/tasks.py` (line 48) and the `DocumentClassifier.train()` method in `src/documents/classifier.py` (line 115), documenting the SHA-1 data hash comparison at line 163 that causes early return when training data is unchanged, and the `load_classifier()` function at line 30 that checks for `settings.MODEL_FILE` existence.
- To **quantify training documents for correspondent matching**, we will analyze the `generate_test_data()` method in `src/documents/tests/test_classifier.py` (line 25) which creates exactly 3 `Document` records and 3 `Correspondent` records (2 with `MATCH_AUTO`), and trace how `matching.match_correspondents()` in `src/documents/matching.py` (line 21) combines classifier prediction with rule-based matching.
- To **trace the no-text edge case**, we will follow the `RasterisedDocumentParser.parse()` method in `src/paperless_tesseract/parsers.py` (line 230) through its `NoTextFoundException` handling at line 267, force-OCR fallback at line 288, and final empty-string assignment at line 327.
- To **analyze barcode splitting**, we will trace `consume_file()` in `src/documents/tasks.py` (line 184) through `scan_file_for_separating_barcodes()` (line 96) and `separate_pages()` (line 113), documenting how split fragments are saved to the consumption directory for independent re-ingestion, which means each fragment produces its own `Document` record through the standard consumption pipeline.
- To **create the deliverable**, we will generate a comprehensive Markdown document in `blitzy/documentation/` that answers each question with code evidence, file paths, and line numbers.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation spans the entire ML classification pipeline, document consumption workflow, OCR integration, barcode splitting subsystem, and the test infrastructure. Every file listed below was retrieved and analyzed to derive conclusions.

**Core ML Pipeline Files (read and analyzed):**

| File Path | Relevance to Investigation |
|---|---|
| `src/documents/classifier.py` | Central ML pipeline: `DocumentClassifier` class, `train()`, `predict_*()`, `load_classifier()`, SHA-1 data hashing, pickle persistence, `FORMAT_VERSION = 7`, `MLPClassifier(tol=0.01)` without `random_state` |
| `src/documents/matching.py` | Bridges classifier predictions and rule-based matching: `match_correspondents()`, `match_document_types()`, `match_tags()`, six matching algorithms, fuzzy threshold 90 |
| `src/documents/consumer.py` | Ingestion pipeline: MIME detection via `magic.from_file()`, parser dispatch, classifier loading post-parse, `document_consumption_finished` signal emission with classifier |
| `src/documents/tasks.py` | Task orchestration: `train_classifier()`, `consume_file()`, barcode splitting via `scan_file_for_separating_barcodes()`, `separate_pages()`, `barcode_reader()`, `save_to_dir()` |
| `src/documents/models.py` | Domain models: `Document`, `Correspondent`, `Tag`, `DocumentType`, `MatchingModel` with `MATCH_AUTO = 6`, `FileInfo` |
| `src/documents/parsers.py` | Base parser infrastructure: `DocumentParser`, `get_parser_class_for_mime_type()`, MIME-to-parser routing via `document_consumer_declaration` signal, date parsing |

**Signal and Wiring Files (read and analyzed):**

| File Path | Relevance to Investigation |
|---|---|
| `src/documents/signals/__init__.py` | Defines `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` signals |
| `src/documents/signals/handlers.py` | Post-consumption handlers: `set_correspondent()`, `set_document_type()`, `set_tags()`, `add_inbox_tags()`, `set_log_entry()`, `add_to_index()` — all fire after classifier is loaded |
| `src/documents/apps.py` | Wires signal handlers at Django `ready()` time: connects all six handlers to `document_consumption_finished` |

**OCR and Parser Files (read and analyzed):**

| File Path | Relevance to Investigation |
|---|---|
| `src/paperless_tesseract/parsers.py` | `RasterisedDocumentParser`: OCR via `ocrmypdf.ocr()`, `NoTextFoundException`, force-OCR fallback, sidecar/pdfminer text extraction, DPI handling, alpha channel removal |
| `src/paperless_tesseract/signals.py` | Parser registration: supported MIME types `application/pdf`, `image/jpeg`, `image/png`, `image/tiff`, `image/gif`, `image/bmp` with weight 0 |
| `src/paperless_tesseract/checks.py` | Tesseract language validation |
| `src/paperless_text/parsers.py` | `TextDocumentParser` for `text/plain` and `text/csv` |
| `src/paperless_text/signals.py` | Text parser registration with weight 10 |

**Test Files (read and analyzed):**

| File Path | Relevance to Investigation |
|---|---|
| `src/documents/tests/test_classifier.py` | `TestClassifier`: `generate_test_data()` creates 3 docs / 3 correspondents / 2 doc types / 3 tags, tests training, prediction, persistence, versioning, data hashing |
| `src/documents/tests/test_tasks.py` | `TestTasks`: barcode reader tests, separator scanning, page separation, `consume_barcode_file` integration, classifier training tests (mocked and real), sanity check tests |
| `src/documents/tests/test_consumer.py` | `TestConsumer`: consumer pipeline tests with mocked parser/magic, `testClassifyDocument` mocks the classifier, duplicate handling, filename handling |
| `src/documents/tests/test_matchables.py` | `TestMatching` and `TestDocumentConsumptionFinishedSignal`: matching algorithm tests, signal-driven tag/correspondent assignment |
| `src/documents/tests/utils.py` | `DirectoriesMixin` and `setup_directories()`: isolated temp dirs, `MODEL_FILE` override, `MEDIA_LOCK` setup |
| `src/documents/tests/factories.py` | `CorrespondentFactory` and `DocumentFactory` for test data creation |
| `src/setup.cfg` | Pytest configuration: `DJANGO_SETTINGS_MODULE=paperless.settings`, `--numprocesses auto` (parallel), `--cov`, `PAPERLESS_DISABLE_DBHANDLER=true` |

**Configuration and Dependency Files (read and analyzed):**

| File Path | Relevance to Investigation |
|---|---|
| `src/paperless/settings.py` | Settings: `MODEL_FILE`, `CONSUMER_ENABLE_BARCODES`, `CONSUMER_BARCODE_STRING` ("PATCHT"), `OCR_LANGUAGE`, `OCR_MODE`, `SCRATCH_DIR`, `CONSUMPTION_DIR` |
| `Pipfile` | Dependencies: `scikit-learn==1.0.2`, `ocrmypdf~=13.4`, `pyzbar`, `pdf2image`, `pikepdf~=5.1`, `fuzzywuzzy`, `python-magic`, `django~=4.0` |
| `requirements.txt` | Pinned dependencies: `scikit-learn==1.0.2`, `ocrmypdf==13.4.3`, `pyzbar==0.1.9`, `pdf2image==1.16.0`, `pikepdf==5.1.1`, `pillow==9.1.0` |

**Test Sample Files (enumerated):**

| Directory | Contents |
|---|---|
| `src/documents/tests/samples/barcodes/` | 18 barcode test fixtures: Code 39, Code 128, QR, PATCHT, custom, distortion, unreadable, multi-separator PDFs |
| `src/documents/tests/samples/documents/` | Original and archive PDFs, thumbnails, GPG-encrypted samples |
| `src/documents/tests/samples/` | `simple.pdf`, `simple.png`, `simple.jpg`, `simple.txt`, `simple-noalpha.png`, `test_with_bom.pdf` |

### 0.2.2 Integration Point Discovery

- **Classifier → Consumer integration**: `src/documents/consumer.py` line 292 calls `load_classifier()` after parsing completes, passes the classifier to the `document_consumption_finished` signal at line 310.
- **Signal → Matching integration**: `src/documents/signals/handlers.py` functions `set_correspondent()` (line 35), `set_document_type()` (line 101), and `set_tags()` (line 168) receive the classifier via signal kwargs and pass it to `matching.match_*()` functions.
- **Matching → Classifier integration**: `src/documents/matching.py` functions `match_correspondents()` (line 21), `match_document_types()` (line 34), and `match_tags()` (line 47) call `classifier.predict_*()` methods and combine results with rule-based matching.
- **Barcode → Consumer integration**: `src/documents/tasks.py` `consume_file()` (line 184) checks `settings.CONSUMER_ENABLE_BARCODES`, scans for barcodes, splits, then saves fragments to `CONSUMPTION_DIR` for standard re-ingestion.
- **OCR → Consumer integration**: `src/documents/parsers.py` `get_parser_class_for_mime_type()` (line 81) routes MIME types to parsers via `document_consumer_declaration` signal; the tesseract parser handles `application/pdf` and image types.
- **Training → Classifier integration**: `src/documents/tasks.py` `train_classifier()` (line 48) checks for `MATCH_AUTO` entities, loads or creates a `DocumentClassifier`, calls `train()`, and saves if data changed.

### 0.2.3 Web Search Research Conducted

No external web search was required for this investigation. All questions are answerable directly from the source code. The codebase provides complete evidence for:

- scikit-learn `MLPClassifier` non-determinism characteristics (inherent to the algorithm when `random_state` is not set)
- OCRmyPDF invocation patterns and fallback strategies
- pyzbar barcode decoding behavior
- Django test isolation patterns with `TestCase` vs `TransactionTestCase`

### 0.2.4 New File Requirements

**New documentation file to create:**

- `blitzy/documentation/<source_branch_name>.md` — Comprehensive investigative analysis document answering all questions about the ML pipeline behavior during test execution, classifier reuse/retraining, correspondent matching, OCR edge cases, and barcode splitting

No new source files, test files, or configuration files are required. The investigation is strictly read-only with respect to the existing codebase.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are directly relevant to the ML classification pipeline, OCR processing, barcode splitting, and test execution that this investigation analyzes. All versions are sourced from `requirements.txt` (pinned install manifest) and `Pipfile` (dependency specification).

| Package Registry | Package Name | Version | Purpose in Investigation |
|---|---|---|---|
| PyPI | scikit-learn | 1.0.2 | ML classifier engine — `MLPClassifier`, `CountVectorizer`, `MultiLabelBinarizer`, `LabelBinarizer`; pinned exactly because version updates cause model pickle incompatibility |
| PyPI | ocrmypdf | 13.4.3 | OCR subprocess invoked by `RasterisedDocumentParser.parse()` via `ocrmypdf.ocr()`; controls text extraction, archive PDF generation, and fallback OCR |
| PyPI | pyzbar | 0.1.9 | Barcode decoding in `tasks.barcode_reader()` via `pyzbar.decode()`; detects separator barcodes on rendered PDF pages |
| PyPI | pdf2image | 1.16.0 | PDF page rendering in `tasks.scan_file_for_separating_barcodes()` via `convert_from_path()`; converts PDF pages to images for barcode scanning |
| PyPI | pikepdf | 5.1.1 | PDF manipulation in `tasks.separate_pages()` via `Pdf.open()` / `Pdf.new()`; splits PDFs at separator page boundaries |
| PyPI | python-magic | 0.4.25 | MIME type detection in `consumer.py` line 219 via `magic.from_file(path, mime=True)`; determines parser dispatch |
| PyPI | django | 4.0.4 | Web framework; provides ORM, test runner, signals, settings, and `TestCase` / `override_settings` used throughout the test suite |
| PyPI | djangorestframework | 3.13.1 | REST API framework; serializers, views, and token auth |
| PyPI | pdfminer.six | 20220319 | Text extraction fallback in `RasterisedDocumentParser.extract_text()` via `pdfminer_extract_text()`; used when sidecar file is incomplete |
| PyPI | pillow | 9.1.0 | Image handling in `RasterisedDocumentParser` for alpha detection, DPI extraction, A4 DPI calculation; also used by `TextDocumentParser` for thumbnails |
| PyPI | fuzzywuzzy | 0.18.0 | Fuzzy matching in `matching.py` line 135 via `fuzz.partial_ratio()` with threshold 90; the `[speedup]` extra includes `python-levenshtein` |
| PyPI | numpy | 1.22.3 | Numerical foundation for scikit-learn classifiers |
| PyPI | scipy | 1.8.0 | Scientific computing foundation for scikit-learn `MLPClassifier` optimization |
| PyPI | filelock | 3.6.0 | Concurrency control via `FileLock` in `consumer.py` and `signals/handlers.py` for media file operations |
| PyPI | factory-boy | (dev) | Test data generation via `CorrespondentFactory` and `DocumentFactory` in `src/documents/tests/factories.py` |
| PyPI | pytest | (dev) | Test runner configured in `setup.cfg` with `--numprocesses auto` parallel execution |
| PyPI | pytest-xdist | (dev) | Parallel test execution plugin; `--numprocesses auto` creates worker processes that can cause shared-state race conditions |
| PyPI | pytest-django | (dev) | Django test integration; provides `DJANGO_SETTINGS_MODULE` configuration |
| PyPI | pytest-env | (dev) | Test environment variables; sets `PAPERLESS_DISABLE_DBHANDLER=true` |

### 0.3.2 Dependency Updates

No dependency updates are required. This investigation is read-only and produces only a documentation artifact. The existing dependency versions in `requirements.txt` and `Pipfile` are documented above for reference in the investigative analysis.

**Import context relevant to the investigation:**

- `src/documents/classifier.py` imports `sklearn.feature_extraction.text.CountVectorizer`, `sklearn.neural_network.MLPClassifier`, `sklearn.preprocessing.MultiLabelBinarizer`, `sklearn.preprocessing.LabelBinarizer` lazily inside `train()` (lines 188–190)
- `src/documents/classifier.py` imports `sklearn.utils.multiclass.type_of_target` lazily inside `predict_tags()` (line 274)
- `src/documents/tasks.py` imports `pyzbar.pyzbar`, `pdf2image.convert_from_path`, and `pikepdf.Pdf` at module level (lines 23–25)
- `src/paperless_tesseract/parsers.py` imports `ocrmypdf` lazily inside `parse()` (line 246)
- `src/documents/consumer.py` imports `magic` at module level (line 7)

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation traces five interconnected subsystems. The following documents every integration point, the signal chain, and the data flow relevant to the user's questions about non-deterministic test failures.

**Classifier Training Flow (Question: reuse vs. retrain)**

- `src/documents/tasks.py` line 48 — `train_classifier()` entry point
  - Line 50–55: Checks if any `Tag`, `DocumentType`, or `Correspondent` with `matching_algorithm == MATCH_AUTO` exists; returns early if none
  - Line 57: Calls `load_classifier()` to attempt loading a persisted model
  - Line 59–60: If no persisted model, creates a fresh `DocumentClassifier()`
  - Line 63: Calls `classifier.train()` which may return `False` (no change) or `True` (retrained)
  - Line 67: If trained, calls `classifier.save()` to persist to `settings.MODEL_FILE`
- `src/documents/classifier.py` line 30 — `load_classifier()` function
  - Line 31: Checks `os.path.isfile(settings.MODEL_FILE)` — if file does not exist, returns `None`
  - Line 38–57: If file exists, creates `DocumentClassifier()`, calls `load()`, handles corruption/version errors by deleting the model file
- `src/documents/classifier.py` line 115 — `train()` method
  - Lines 125–127: Queries `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` to gather training data
  - Lines 124, 129, 136, 142, 155: Computes SHA-1 hash of all document content + labels
  - Line 163: Compares `new_data_hash` with `self.data_hash` — if identical, returns `False` (skips retraining)
  - Lines 188–190: Lazy imports of `CountVectorizer`, `MLPClassifier`, `MultiLabelBinarizer`, `LabelBinarizer`
  - Lines 219, 228, 238: Creates `MLPClassifier(tol=0.01)` **without** `random_state` — this is the primary source of non-deterministic predictions

**Consumer → Classifier → Signal Chain (Question: when does classification happen)**

- `src/documents/consumer.py` line 180 — `try_consume_file()` entry point
  - Line 219: MIME type detected via `magic.from_file(self.path, mime=True)`
  - Line 223: Parser selected via `get_parser_class_for_mime_type(mime_type)`
  - Line 261: Parser runs `parse(self.path, mime_type, self.filename)` — text extraction and OCR happen here
  - Line 292: **After parsing completes**, `load_classifier()` loads the ML model
  - Line 298–311: Within `transaction.atomic()`, document is stored via `_store()`, then `document_consumption_finished` signal fires with the classifier
- `src/documents/apps.py` line 11–27 — Signal handler registration at app `ready()`:
  - `document_consumption_finished` → `add_inbox_tags` (adds inbox-tagged tags)
  - `document_consumption_finished` → `set_correspondent` (ML + rule matching)
  - `document_consumption_finished` → `set_document_type` (ML + rule matching)
  - `document_consumption_finished` → `set_tags` (ML + rule matching)
  - `document_consumption_finished` → `set_log_entry` (admin log)
  - `document_consumption_finished` → `add_to_index` (search index update)
- `src/documents/signals/handlers.py` line 35 — `set_correspondent()`:
  - Line 50: Calls `matching.match_correspondents(document, classifier)`
  - Line 54: Takes `potential_correspondents[0]` (first match) if any exist
  - Line 97–98: Assigns and saves correspondent — **no confidence threshold, no score filtering**
- `src/documents/matching.py` line 21 — `match_correspondents()`:
  - Line 23: If classifier exists, calls `classifier.predict_correspondent(document.content)`
  - Line 30: Returns all correspondents where either rule-based `matches()` returns True OR `o.pk == pred_id`
- `src/documents/classifier.py` line 251 — `predict_correspondent()`:
  - Line 253: Vectorizes content via `self.data_vectorizer.transform()`
  - Line 254: Calls `self.correspondent_classifier.predict(X)` — **returns raw class label, no probability threshold**
  - Line 255–258: Returns the predicted ID if not -1, else `None`

**OCR Edge Case Flow (Question: no extractable text)**

- `src/paperless_tesseract/parsers.py` line 230 — `parse()` method
  - Line 234–236: For PDFs, attempts pre-extraction via `extract_text(None, document_path)` using pdfminer
  - Line 241–244: If `OCR_MODE == "skip_noarchive"` and text exists, skips OCR entirely
  - Line 252–257: Constructs `ocrmypdf` parameters via `construct_ocrmypdf_parameters()`
  - Line 261: Invokes `ocrmypdf.ocr(**args)` — this is the OCR subprocess
  - Line 264: Extracts text from sidecar or archive PDF
  - Line 267: If no text found, raises `NoTextFoundException`
  - Lines 276–310: Catches `NoTextFoundException` and `InputFileError`, retries with `safe_fallback=True` which sets `force_ocr=True`
  - Lines 318–327: As last resort, uses original text if available, otherwise sets `self.text = ""` (empty string)
- `src/paperless_tesseract/parsers.py` line 135 — `construct_ocrmypdf_parameters()`:
  - Returns a dictionary consumed by `ocrmypdf.ocr()`, including `language`, `output_type`, `use_threads`, `jobs`, mode flags (`force_ocr`, `skip_text`, `redo_ocr`), and image DPI settings

**Barcode Splitting Flow (Question: document record creation and training data impact)**

- `src/documents/tasks.py` line 184 — `consume_file()` entry point
  - Line 195: Checks `settings.CONSUMER_ENABLE_BARCODES` (default `False`)
  - Line 198: Calls `scan_file_for_separating_barcodes(path)` to find separator pages
- `src/documents/tasks.py` line 96 — `scan_file_for_separating_barcodes()`:
  - Line 102: Reads `settings.CONSUMER_BARCODE_STRING` (default `"PATCHT"`)
  - Line 105: Converts PDF pages to images via `convert_from_path(filepath, output_folder=path)`
  - Line 106–109: Iterates pages, calls `barcode_reader(page)`, checks if `separator_barcode in current_barcodes`
  - Returns list of 0-indexed page numbers containing the separator barcode
- `src/documents/tasks.py` line 75 — `barcode_reader()`:
  - Line 82: Calls `pyzbar.decode(image)` to detect all barcodes in the image
  - Line 88: Decodes barcode data as UTF-8
  - Returns list of decoded barcode strings
- `src/documents/tasks.py` line 113 — `separate_pages()`:
  - Line 123: Opens source PDF via `Pdf.open(filepath)`
  - Lines 130–138: Creates first document fragment from pages before the first separator
  - Lines 141–159: Creates subsequent fragments from pages between separators; separator pages are excluded
  - Returns list of temporary file paths, one per fragment
- `src/documents/tasks.py` lines 199–233 — Fragment consumption:
  - Line 201: Calls `separate_pages(path, separators)` to get fragment paths
  - Lines 203–210: Each fragment is saved to `CONSUMPTION_DIR` via `save_to_dir()`
  - Line 214: Original file is deleted
  - **Fragments are NOT consumed in-line** — they are deposited into the consumption directory for the standard directory watcher to pick up, meaning each fragment goes through the full `consume_file()` → `Consumer.try_consume_file()` pipeline independently
  - Each fragment produces its own `Document` record, increasing the effective training data for subsequent classifier training

### 0.4.2 Test Infrastructure Integration Points

- `src/documents/tests/utils.py` line 14 — `setup_directories()`:
  - Creates isolated temp directories for `DATA_DIR`, `SCRATCH_DIR`, `MEDIA_ROOT`, `CONSUMPTION_DIR`, `INDEX_DIR`
  - Line 46: Sets `MODEL_FILE` to `os.path.join(dirs.data_dir, "classification_model.pickle")` — isolated per test class via `DirectoriesMixin`
  - Line 47: Sets `MEDIA_LOCK` to a temp path
- `src/setup.cfg` line 10 — `addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet`:
  - `--numprocesses auto` enables pytest-xdist parallel execution, creating multiple worker processes
  - Each worker gets its own database (Django test runner handles this), but filesystem paths could overlap if `DirectoriesMixin` is not used consistently
- `src/documents/tests/test_classifier.py` line 20 — `TestClassifier(DirectoriesMixin, TestCase)`:
  - Uses `DirectoriesMixin` for isolated MODEL_FILE
  - `generate_test_data()` creates: 3 Correspondents (c1 with MATCH_AUTO, c2 without, c3 with MATCH_AUTO), 3 Tags (t1 with MATCH_AUTO pk=12, t2 with MATCH_ANY and is_inbox_tag pk=34, t3 with MATCH_AUTO pk=45), 2 DocumentTypes (dt with MATCH_AUTO, dt2 with MATCH_AUTO), 3 Documents (doc1, doc2, doc_inbox)
  - doc_inbox is tagged with t2 (inbox tag) so it is **excluded** from training data by the `exclude(tags__is_inbox_tag=True)` filter at classifier.py line 127
- `src/documents/tests/test_matchables.py` line 380 — `TestDocumentConsumptionFinishedSignal`:
  - Does NOT use `DirectoriesMixin`
  - Manually creates a temp `INDEX_DIR` and uses `override_settings`
  - Sends `document_consumption_finished` signal directly without a classifier (no `classifier=` kwarg), so ML predictions are `None`

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this is a read-only investigation per the `SWE-AtlasQnA-Repo` rule, the only file to create is the investigative Markdown document. All other interactions are read-only analysis of existing files.

**Group 1 — Investigation Deliverable:**

- CREATE: `blitzy/documentation/<source_branch_name>.md` — Comprehensive QnA document answering all five investigation questions with rationale derived from the source code

**Group 2 — Source Files to Analyze (read-only, no modifications):**

- READ: `src/documents/classifier.py` — Full ML pipeline: training, prediction, persistence, hashing, versioning
- READ: `src/documents/matching.py` — Matching bridge between classifier and rule-based algorithms
- READ: `src/documents/consumer.py` — Ingestion pipeline, MIME detection, parser dispatch, classifier loading
- READ: `src/documents/tasks.py` — Barcode splitting, classifier training task, consume_file orchestration
- READ: `src/documents/parsers.py` — Parser base class, parser routing, date parsing
- READ: `src/documents/models.py` — MatchingModel, Document, Correspondent, Tag, DocumentType
- READ: `src/documents/signals/__init__.py` — Signal definitions
- READ: `src/documents/signals/handlers.py` — Post-consumption handlers for classification assignment
- READ: `src/documents/apps.py` — Signal handler wiring
- READ: `src/paperless_tesseract/parsers.py` — OCR parser, text extraction, fallback strategies
- READ: `src/paperless_tesseract/signals.py` — MIME type registration for tesseract parser

**Group 3 — Test Files to Analyze (read-only, no modifications):**

- READ: `src/documents/tests/test_classifier.py` — Classifier training/prediction test scenarios
- READ: `src/documents/tests/test_tasks.py` — Barcode and classifier task tests
- READ: `src/documents/tests/test_consumer.py` — Consumer pipeline tests with mocked classifier
- READ: `src/documents/tests/test_matchables.py` — Matching algorithm and signal tests
- READ: `src/documents/tests/utils.py` — Test infrastructure: DirectoriesMixin, setup_directories
- READ: `src/documents/tests/factories.py` — Factory-boy test fixtures
- READ: `src/setup.cfg` — Pytest configuration including parallel execution

**Group 4 — Configuration Files to Analyze (read-only):**

- READ: `src/paperless/settings.py` — MODEL_FILE, CONSUMER_ENABLE_BARCODES, CONSUMER_BARCODE_STRING, OCR settings
- READ: `Pipfile` — Python dependency specifications with version constraints
- READ: `requirements.txt` — Pinned dependency versions

### 0.5.2 Implementation Approach

The investigation document must be structured to answer each of the user's five questions with direct code evidence:

**Question 1: Classifier Reuse vs. Retraining**

Establish the answer by tracing these modules:

- `src/documents/tasks.py` `train_classifier()` (line 48): loads existing model via `load_classifier()`, which checks `os.path.isfile(settings.MODEL_FILE)` in `src/documents/classifier.py` line 31
- `src/documents/classifier.py` `train()` (line 115): computes SHA-1 of training data, compares with `self.data_hash` at line 163; if unchanged, returns `False` without retraining
- During tests, `DirectoriesMixin` sets `MODEL_FILE` to a temp path (utils.py line 46), so each test class starts with no model file — the classifier is always freshly trained on first call within a test class
- Within a single test method, if `train()` is called twice with the same data, the second call returns `False` (data hash match), demonstrating reuse behavior

**Question 2: Training Document Creation and Confidence Thresholds**

Document the exact counts and timing:

- `src/documents/tests/test_classifier.py` `generate_test_data()` (line 25): creates exactly 3 `Document` records (doc1, doc2, doc_inbox), but doc_inbox is excluded from training by `exclude(tags__is_inbox_tag=True)`, leaving **2 effective training documents**
- Training occurs when `classifier.train()` is called explicitly in the test method — always AFTER all Document inserts
- There is **no confidence threshold** anywhere in the prediction path: `predict_correspondent()` at classifier.py line 254 calls `self.correspondent_classifier.predict(X)` which returns the class label directly; `match_correspondents()` at matching.py line 23 uses this raw prediction; `set_correspondent()` at handlers.py line 54 takes the first match without scoring

**Question 3: No-Text Edge Case and OCR Subprocess**

Trace the complete fallback chain:

- OCR subprocess: `ocrmypdf.ocr(**args)` invoked at `src/paperless_tesseract/parsers.py` line 261
- If no text found: `NoTextFoundException` raised at line 267
- Fallback: `construct_ocrmypdf_parameters()` called with `safe_fallback=True` (line 288), which sets `force_ocr=True` (line 155)
- Retry: `ocrmypdf.ocr(**args)` invoked again with force-OCR at line 298
- Final fallback: if still no text, `self.text = ""` at line 327
- MIME type: detected by `magic.from_file(path, mime=True)` in consumer.py line 219; the archive output is always `application/pdf` regardless of input type
- For image inputs: the tesseract parser handles `image/jpeg`, `image/png`, `image/tiff`, `image/gif`, `image/bmp` as registered in `src/paperless_tesseract/signals.py` lines 11–18

**Question 4: Barcode Splitting Document Records**

Document the splitting mechanics and training data impact:

- Trigger: `settings.CONSUMER_BARCODE_STRING` defaults to `"PATCHT"` (settings.py line 506)
- Decision point: `src/documents/tasks.py` `scan_file_for_separating_barcodes()` line 96 scans each page via `barcode_reader()` line 75, which uses `pyzbar.decode(image)` line 82
- Split operation: `separate_pages()` line 113 creates N+1 fragments from N separator pages (pages before first separator become document 0, pages between separators become subsequent documents, separator pages excluded)
- Document creation: each fragment is saved to `CONSUMPTION_DIR` (line 210) and will be consumed independently by the directory watcher, each producing its own `Document` record
- Training data impact: the new `Document` records created by consuming fragments contribute to the training dataset for subsequent `train_classifier()` calls; this means the effective training data grows during the test run if barcode splitting tests create real documents and training is invoked afterward

**Question 5: Non-Determinism Root Causes**

Synthesize the findings into a root-cause analysis:

- `MLPClassifier(tol=0.01)` at classifier.py lines 219, 228, 238 — no `random_state` parameter, meaning the neural network weight initialization uses different random seeds on each instantiation
- `--numprocesses auto` in setup.cfg line 10 — parallel test execution via pytest-xdist can cause test ordering and timing variations
- `CountVectorizer(min_df=0.01)` at classifier.py line 194 — feature vocabulary depends on training corpus, which may vary if document creation timing differs across parallel workers
- Small training datasets in tests (2–4 documents) amplify the effect of neural network non-determinism, as the decision boundary is less stable with few examples

### 0.5.3 User Interface Design

Not applicable. This investigation produces a documentation artifact only and has no UI component.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Investigation target files (read-only analysis):**

- ML classifier pipeline: `src/documents/classifier.py`
- Matching bridge: `src/documents/matching.py`
- Consumer pipeline: `src/documents/consumer.py`
- Task orchestration and barcode splitting: `src/documents/tasks.py`
- Parser infrastructure: `src/documents/parsers.py`
- Domain models: `src/documents/models.py`
- Signal definitions: `src/documents/signals/__init__.py`
- Signal handlers: `src/documents/signals/handlers.py`
- App wiring: `src/documents/apps.py`
- OCR parser: `src/paperless_tesseract/parsers.py`
- OCR registration: `src/paperless_tesseract/signals.py`
- Text parser: `src/paperless_text/parsers.py`
- Text registration: `src/paperless_text/signals.py`
- Settings: `src/paperless/settings.py`

**Test files (read-only analysis):**

- Classifier tests: `src/documents/tests/test_classifier.py`
- Task tests: `src/documents/tests/test_tasks.py`
- Consumer tests: `src/documents/tests/test_consumer.py`
- Matching signal tests: `src/documents/tests/test_matchables.py`
- Test utilities: `src/documents/tests/utils.py`
- Test factories: `src/documents/tests/factories.py`
- Test samples: `src/documents/tests/samples/**/*`

**Configuration files (read-only analysis):**

- Test runner config: `src/setup.cfg`
- Python dependencies: `Pipfile`, `requirements.txt`

**Deliverable (to be created):**

- `blitzy/documentation/<source_branch_name>.md` — The sole output artifact

### 0.6.2 Explicitly Out of Scope

- **Source code modifications** — Expressly prohibited by the user: "Please do not modify the source code." Reinforced by `SWE-AtlasQnA-Repo` rule: "Do not modify any existing files in the source repository."
- **New code in the source repository** — The SWE-AtlasQnA-Repo rule states: "Do not add any other code in the source repository (besides the above requested document)."
- **Fixing the non-deterministic test failures** — The user wants to understand the behavior, not fix it. The deliverable is an analysis document, not a patch.
- **Performance optimizations to the classifier** — Not requested; the investigation is diagnostic only.
- **Frontend (Angular) code** — The `src-ui/` directory is entirely outside the investigation scope; the ML pipeline operates entirely in the Python backend.
- **Email ingestion subsystem** — `src/paperless_mail/` is unrelated to the ML classification test behavior.
- **Tika parser** — `src/paperless_tika/` is not part of the classification or OCR pipeline under investigation.
- **Migration files** — `src/documents/migrations/` are not relevant to runtime ML behavior.
- **Docker/infrastructure** — `Dockerfile`, `docker/`, `.github/` are unrelated to test execution behavior.
- **Documentation build** — `docs/` (Sphinx documentation) is not part of the investigation.
- **Temporary helpers** — The user permits creating temporary helpers during investigation but requires cleanup: "you may create temporary helpers while investigating, but clean up anything temporary before finishing." Any temporary scripts or logging must be removed before the task is complete.

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The user and the project-level implementation rules impose the following constraints:

**SWE-AtlasQnA-Repo Rule (Project-Level):**

- Create a new Markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed in the prompt
- Provide thinking and rationale behind the answers
- Do not make assumptions; base answers on the code as the truth
- Do not modify any existing files in the source repository
- Do not add any other code in the source repository besides the requested document
- Place the generated document in the `blitzy/documentation` directory in the destination repo

**User-Specified Constraints:**

- "Please do not modify the source code" — No changes to any `.py`, `.cfg`, `.txt`, or other source files
- "You may create temporary helpers while investigating" — Transient scripts or logging hooks are permitted during the investigation phase
- "Clean up anything temporary before finishing" — All temporary artifacts must be removed before the task completes; the only remaining artifact is the documentation file

### 0.7.2 Investigation Quality Standards

- Every claim in the Markdown document must cite a specific file path and line number
- Answers must trace actual execution paths, not hypothetical behavior
- Where the code has no explicit threshold or configuration (e.g., no confidence threshold in the classifier prediction), the document must explicitly state that absence with evidence
- The analysis must distinguish between behavior observed in tests (with mocks and overrides) versus production runtime behavior
- The document must identify and explain all sources of non-determinism discovered during the investigation

### 0.7.3 Repository Convention Compliance

- The output document must be placed in `blitzy/documentation/` as specified by the SWE-AtlasQnA-Repo rule
- The document must be named `<source_branch_name>.md` where the branch name is determined from the current Git branch
- The document must be Markdown format with clear headings, code references, and structured answers
- No changes to `.gitignore`, `setup.cfg`, `Pipfile`, or any other configuration files

## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were retrieved, read, and analyzed to derive all conclusions in this Agent Action Plan:

**Root-level files:**

| File Path | Purpose |
|---|---|
| `Pipfile` | Python dependency specifications with version constraints for scikit-learn, ocrmypdf, pyzbar, pdf2image, pikepdf, django, and dev packages |
| `requirements.txt` | Pinned dependency versions: scikit-learn==1.0.2, ocrmypdf==13.4.3, pyzbar==0.1.9, pdf2image==1.16.0, pikepdf==5.1.1, django==4.0.4 |

**Core ML pipeline and document processing:**

| File Path | Purpose |
|---|---|
| `src/documents/classifier.py` | DocumentClassifier class — training, prediction, persistence, FORMAT_VERSION=7, MLPClassifier(tol=0.01), SHA-1 data hashing, load_classifier() |
| `src/documents/matching.py` | Matching bridge — match_correspondents(), match_document_types(), match_tags(), six algorithms (ANY, ALL, LITERAL, REGEX, FUZZY, AUTO), fuzzy threshold 90 |
| `src/documents/consumer.py` | Consumer pipeline — MIME detection via magic.from_file(), parser dispatch, classifier loading, document_consumption_finished signal, _store(), apply_overrides() |
| `src/documents/tasks.py` | Task orchestration — train_classifier(), consume_file(), barcode_reader(), scan_file_for_separating_barcodes(), separate_pages(), save_to_dir(), bulk_update_documents() |
| `src/documents/parsers.py` | Parser infrastructure — DocumentParser base class, get_parser_class_for_mime_type(), run_convert(), parse_date(), make_thumbnail_from_pdf() |
| `src/documents/models.py` | Domain models — Document, Correspondent, Tag, DocumentType, MatchingModel (MATCH_AUTO=6), SavedView, SavedViewFilterRule, FileInfo, Log |

**Signal and application wiring:**

| File Path | Purpose |
|---|---|
| `src/documents/signals/__init__.py` | Signal definitions — document_consumption_started, document_consumption_finished, document_consumer_declaration |
| `src/documents/signals/handlers.py` | Post-consumption handlers — set_correspondent(), set_document_type(), set_tags(), add_inbox_tags(), set_log_entry(), add_to_index(), cleanup_document_deletion(), update_filename_and_move_files() |
| `src/documents/apps.py` | App configuration — DocumentsConfig.ready() wires all six signal handlers to document_consumption_finished |

**OCR and parser implementations:**

| File Path | Purpose |
|---|---|
| `src/paperless_tesseract/parsers.py` | RasterisedDocumentParser — OCR via ocrmypdf.ocr(), NoTextFoundException, force-OCR fallback, sidecar/pdfminer text extraction, DPI handling |
| `src/paperless_tesseract/signals.py` | Parser registration — MIME types application/pdf, image/jpeg, image/png, image/tiff, image/gif, image/bmp at weight 0 |
| `src/paperless_text/parsers.py` | TextDocumentParser — plain text file parsing for text/plain and text/csv |
| `src/paperless_text/signals.py` | Text parser registration — MIME types text/plain and text/csv at weight 10 |

**Test suite:**

| File Path | Purpose |
|---|---|
| `src/documents/tests/test_classifier.py` | TestClassifier — generate_test_data() (3 docs, 3 correspondents, 3 tags, 2 doc types), training, prediction, persistence, versioning, data hashing tests |
| `src/documents/tests/test_tasks.py` | TestTasks — barcode reader tests (9 variants), separator scanning (8 tests), page separation, consume_barcode_file, classifier training (mocked and real), sanity check |
| `src/documents/tests/test_consumer.py` | TestConsumer — consumer pipeline with mocked parser/magic, testClassifyDocument with mocked classifier, duplicate handling, filename handling, pre/post consume scripts |
| `src/documents/tests/test_matchables.py` | TestMatching and TestDocumentConsumptionFinishedSignal — all six matching algorithms, case sensitivity, signal-driven tag/correspondent/document-type assignment |
| `src/documents/tests/utils.py` | DirectoriesMixin — isolated temp directories, MODEL_FILE override, setup_directories(), remove_dirs(), paperless_environment() context manager |
| `src/documents/tests/factories.py` | CorrespondentFactory and DocumentFactory — factory-boy test fixtures |
| `src/setup.cfg` | Pytest config — DJANGO_SETTINGS_MODULE, --numprocesses auto, --cov, PAPERLESS_DISABLE_DBHANDLER=true |

**Configuration:**

| File Path | Purpose |
|---|---|
| `src/paperless/settings.py` | Settings searched for: MODEL_FILE, CONSUMER_ENABLE_BARCODES, CONSUMER_BARCODE_STRING, OCR_LANGUAGE, OCR_MODE, SCRATCH_DIR, CONSUMPTION_DIR |

**Folders explored:**

| Folder Path | Purpose |
|---|---|
| (root) | Repository root — identified Paperless-ngx project structure |
| `src/` | Python source tree — Django apps, test runner config |
| `src/documents/` | Primary domain app — models, consumer, classifier, matching, tasks, signals |
| `src/documents/tests/` | Test suite — 29 test files and samples directory |
| `src/documents/tests/samples/` | Test fixtures — barcode images/PDFs, document originals/archives/thumbnails, simple test files |
| `src/documents/signals/` | Signal definitions and handlers |
| `src/paperless_tesseract/` | OCR integration package — parser, signals, checks, tests |
| `src/paperless_text/` | Plain text parser package |

### 0.8.2 Attachments

No attachments were provided by the user for this project.

### 0.8.3 External References

No Figma URLs, external design documents, or third-party API documentation was referenced. All analysis is derived exclusively from the repository source code.

### 0.8.4 Tech Spec Sections Consulted

| Section | Relevant Content |
|---|---|
| 1.1 Executive Summary | Project overview — Paperless-ngx v1.7.0, GPL-3.0, document management system |
| 2.1 Feature Catalog | Feature definitions — F-001 (Ingestion), F-002 (OCR), F-004 (Classification & Matching), F-011 (Barcode Splitting), confirming the scope of the investigation |

