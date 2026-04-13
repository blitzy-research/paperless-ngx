# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **conduct a comprehensive runtime memory profiling and analysis investigation** of the Paperless-ngx document processing system's import pipeline, with the following specific objectives:

- **Diagnose disproportionate memory spikes** during document import operations, particularly during metadata handling stages, where memory consumption appears out of proportion to actual document sizes (especially for text-based documents with small metadata)
- **Identify whether metadata handling creates unnecessary object copies** or holds references longer than needed, causing delayed garbage collection or memory accumulation
- **Investigate caching behavior** that may be accumulating data unexpectedly across processing stages, leading to growing memory footprints
- **Determine what differentiates memory-spiking imports from normal ones** — specifically, how document type, source (filesystem watcher vs. API upload vs. email ingestion), batch size, and processing stage affect memory behavior
- **Produce runtime memory measurements** with evidence-backed findings showing exactly which components and methods are responsible for holding memory, and whether the behavior constitutes a genuine problem or is normal Python/CPython garbage collection behavior
- **Deliver all findings in a comprehensive markdown document** placed in the `blitzy/documentation` directory, without modifying any existing repository source files

Implicit requirements detected:

- The investigation must trace memory usage through every stage of the document consumption pipeline: file detection → duplicate check → MIME detection → parser dispatch → text extraction → metadata extraction → classification → persistence → indexing → signal handlers
- Analysis must cover all three ingestion pathways: directory consumer (`document_consumer.py`), REST API upload (`PostDocumentView`), and email ingestion (`paperless_mail`)
- The analysis must differentiate between transient memory usage (expected during processing) and retained memory that fails to release after processing completes
- Temporary test/profiling scripts may be created for investigation but must be cleaned up afterward, leaving the codebase unchanged

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No modification of existing source files.** The user explicitly states: "Don't modify any repository source files." Only temporary test/analysis scripts are permitted, and these must be cleaned up upon completion.
- **Implementation rule — SWE-AtlasQnA-Repo:** Create a new markdown document named after the source branch that comprehensively answers the posed questions. The document must include thinking/rationale, be based on the code as the truth (no assumptions), and be placed in the `blitzy/documentation` directory.
- **Architectural convention:** All analysis must follow the repository's existing Django/Python patterns and respect the project's settings structure defined in `src/paperless/settings.py`.
- **Evidence-based conclusions required:** Findings must reference specific source file locations, line numbers, and code patterns — not speculative or theoretical assessments.
- **Cleanup requirement:** Any temporary profiling scripts or helper tools created during the investigation must be removed when complete.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **identify memory spike sources**, we will analyze the complete document consumption pipeline in `src/documents/consumer.py`, tracing every method call from `try_consume_file()` through `_store()`, parser invocations, classifier loading, signal handler execution, and file write operations, identifying patterns where full file contents are read into memory unnecessarily (e.g., `f.read()` for MD5 checksums at lines 103 and 402, and buffer-less file copy at lines 430–432)
- To **evaluate classifier memory impact**, we will analyze `src/documents/classifier.py` where `load_classifier()` is called during every consumption (line 292 of consumer.py), deserializing pickle objects containing CountVectorizer, MLPClassifier, and binarizer models into memory
- To **assess metadata processing overhead**, we will examine the signal handler chain registered in `src/documents/apps.py` (lines 22–27): `add_inbox_tags → set_correspondent → set_document_type → set_tags → set_log_entry → add_to_index`, where each handler in `src/documents/signals/handlers.py` queries the database and invokes matching logic from `src/documents/matching.py` that loads all MatchingModel instances
- To **investigate caching behavior**, we will analyze the `DelayedQuery.saved_results` dict in `src/documents/index.py` (line 196) for unbounded result caching, and the `document_consumer_declaration` signal dispatch in `src/documents/parsers.py` that iterates parser registrations on every call
- To **produce runtime evidence**, we will create temporary profiling scripts using Python's `tracemalloc`, `sys.getsizeof`, `gc` module, and `objgraph` (if available) to capture memory snapshots at each pipeline stage, then remove these scripts after analysis
- To **document all findings**, we will create a comprehensive markdown document in `blitzy/documentation/` that answers each user question with code-referenced evidence and measured data


## 0.2 Repository Scope Discovery


### 0.2.1 Comprehensive File Analysis

The following files and folders have been exhaustively analyzed to understand the document import and metadata processing pipeline, identify all memory-relevant code paths, and determine the scope of the investigation.

#### Primary Document Consumption Pipeline Files

| File Path | Relevance | Memory-Critical Patterns Found |
|-----------|-----------|-------------------------------|
| `src/documents/consumer.py` | Core ingestion coordinator — orchestrates the entire import pipeline | Full-file `f.read()` for MD5 in `pre_check_duplicate()` (line 103) and `_store()` (line 402); buffer-less file copy in `_write()` (lines 430–432); classifier loaded per-document (line 292); archive file fully read for checksum (lines 339–342) |
| `src/documents/parsers.py` | Parser base class, MIME-type dispatch, date parsing, thumbnail generation | Regex date parsing iterates all matches in document text (lines 261–272); `dateparser` module lazy-imported per-call (line 221); `document_consumer_declaration.send(None)` called on every parser lookup (lines 87, 48, 71) |
| `src/documents/tasks.py` | Task orchestration: barcode splitting, consumption, indexing, training | `pdf2image.convert_from_path()` renders all PDF pages as PIL images into temp dir (line 105); `Consumer()` instantiated per task (line 236); imports heavy libraries at module level (`pikepdf`, `pyzbar`, `pdf2image`) |
| `src/documents/classifier.py` | ML document classifier — pickle load/save, training, prediction | `load()` unpickles 6 large objects (vectorizer, binarizers, classifiers) into memory (lines 77–93); `train()` loads ALL document content from DB into Python lists (lines 117–157); `preprocess_content()` creates new string copies (line 25) |
| `src/documents/matching.py` | Rule-based matching: correspondent, type, tags | `match_correspondents/document_types/tags` each query ALL records from DB (lines 27, 40, 53); fuzzy matching creates stripped copies of entire document content (lines 130–131); classifier prediction re-invoked per matching function |
| `src/documents/models.py` | Document, Correspondent, Tag, DocumentType ORM definitions | `Document.content` stores full text in a TextField; `source_file`/`archive_file` properties return open file handles (lines 234, 250); `FileInfo.from_filename()` compiles regex per call (lines 460–466) |
| `src/documents/index.py` | Whoosh full-text search indexing | `DelayedQuery.saved_results` dict caches results without bounds (line 196); `AsyncWriter` usage (line 66); `open_index()` creates new index objects (line 52) |
| `src/documents/file_handling.py` | File naming, directory management, tag-to-dictionary conversion | `many_to_dictionary()` queries M2M tags via `field.all()` (line 60); `generate_filename()` queries tags and metadata for each rename |

#### Signal Handler Chain (Post-Consumption)

| File Path | Relevance | Memory-Critical Patterns Found |
|-----------|-----------|-------------------------------|
| `src/documents/signals/__init__.py` | Defines `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` signals | Signal instances live as module-level singletons |
| `src/documents/signals/handlers.py` | Receivers: inbox tags, correspondent, type, tags, log entry, index update | Each handler runs sequentially within same transaction; `set_correspondent/set_document_type/set_tags` each invoke full matching logic loading all DB records; `add_to_index` opens and commits Whoosh index writer |
| `src/documents/apps.py` | Connects 6 signal handlers to `document_consumption_finished` at startup | Handlers fire in order: `add_inbox_tags → set_correspondent → set_document_type → set_tags → set_log_entry → add_to_index` |

#### Parser Implementation Files

| File Path | Relevance | Memory-Critical Patterns Found |
|-----------|-----------|-------------------------------|
| `src/paperless_tesseract/parsers.py` | OCR parser — largest memory consumer for PDF/image docs | `pikepdf.open()` keeps PDF in memory (line 34); `Image.open()` called multiple times for alpha/DPI checks (lines 74, 79, 88); `ocrmypdf.ocr()` spawns subprocesses but intermediate results stay in temp dir; `pdfminer_extract_text()` extracts full text (line 120) |
| `src/paperless_text/parsers.py` | Plain-text parser — lightest parser | Reads entire file text into `self.text` (line 42); creates PIL Image for thumbnail (lines 26–37) |
| `src/paperless_tika/parsers.py` | Tika-based parser for Office documents | `parser.from_file()` sends file to Tika server and receives full parsed response (line 55); `response.content` holds entire PDF conversion in memory (line 96) |

#### Configuration and Settings Files

| File Path | Relevance | Memory-Critical Patterns Found |
|-----------|-----------|-------------------------------|
| `src/paperless/settings.py` | All configuration constants: dirs, threading, task workers, OCR settings | `Q_CLUSTER["recycle"] = 1` means workers are recycled after every task (line 452); `TASK_WORKERS` dynamic sizing (lines 427–438); `THREADS_PER_WORKER` controls OCR parallelism (lines 460–472); `CHANNEL_LAYERS` Redis capacity 2000 (line 183) |
| `src/setup.cfg` | pytest, flake8, coverage config | `PAPERLESS_DISABLE_DBHANDLER=true` for tests (line 11) |
| `Pipfile` | Python dependency declarations | `scikit-learn==1.0.2` pinned (large ML library); `ocrmypdf~=13.4` (spawns Tesseract processes) |
| `requirements.txt` | Pinned dependency manifest (113 packages) | Full package versions for reproducibility |

#### Ingestion Pathway Files

| File Path | Relevance | Memory-Critical Patterns Found |
|-----------|-----------|-------------------------------|
| `src/documents/management/commands/document_consumer.py` | Filesystem watcher — directory consumption entry point | Spawns threads per file event (line 130); `async_task()` queues consumption through Django-Q (line 86) |
| `src/documents/management/commands/document_importer.py` | Manifest-based import from export | Loads entire `manifest.json` into memory (line 73); iterates all document records; `shutil.copy2` for file transfers |
| `src/documents/views.py` | REST API upload entry point (`PostDocumentView`) | `doc_data` holds entire uploaded file in memory (line 502); writes to `tempfile.NamedTemporaryFile` (lines 512–519) |
| `src/paperless_mail/mail.py` | Email ingestion via IMAP | Processes mail attachments through imap-tools; queues via `async_task` |

#### Test Files (for Understanding Expected Behavior)

| File Path | Relevance |
|-----------|-----------|
| `src/documents/tests/test_consumer.py` | Consumer pipeline test coverage — expected behaviors |
| `src/documents/tests/test_classifier.py` | Classifier load/train/predict tests |
| `src/documents/tests/test_tasks.py` | Task orchestration tests including barcode splitting |
| `src/documents/tests/test_parsers.py` | Parser dispatch and behavior tests |
| `src/documents/tests/test_matchables.py` | Matching algorithm and signal handler tests |
| `src/documents/tests/test_file_handling.py` | File handling and rename logic tests |

### 0.2.2 Web Search Research Conducted

No external web searches are required for this investigation. The analysis is based entirely on the repository source code, which the user's rules specify as "the truth." All memory behavior patterns are identifiable through static code analysis and can be validated through runtime profiling of the actual codebase.

Key analysis areas already covered through code inspection:
- Python memory management patterns (CPython reference counting + cyclic GC)
- Django ORM QuerySet evaluation and memory behavior
- scikit-learn model serialization size and in-memory footprint
- Whoosh index writer behavior and file handle management
- `pickle` deserialization memory characteristics
- `hashlib` MD5 computation with `f.read()` vs streaming

### 0.2.3 New File Requirements

Since the user's rules specify that no existing files may be modified and the deliverable is a documentation/analysis file, the new file requirements are:

- **CREATE: `blitzy/documentation/<source_branch_name>.md`** — Comprehensive markdown document answering all memory investigation questions, containing:
  - Analysis of memory spike causes in the import pipeline
  - Identification of specific components/methods holding memory
  - Comparison of memory behavior across document types and batch sizes
  - Evidence of whether behavior is normal GC or problematic
  - Code-referenced findings with file paths and line numbers
  - Rationale and thinking behind each conclusion

No temporary profiling scripts will remain in the repository after the investigation is complete.


## 0.3 Dependency Inventory


### 0.3.1 Private and Public Packages

The following packages are directly relevant to the memory investigation, as they participate in the document import pipeline and influence memory behavior during processing. All versions are taken from the pinned `requirements.txt` manifest.

| Package Registry | Package Name | Pinned Version | Purpose in Memory Investigation |
|------------------|-------------|----------------|-------------------------------|
| PyPI | django | 4.0.4 | Core framework — ORM QuerySet evaluation, signal dispatch, and transaction management all affect memory lifecycle |
| PyPI | djangorestframework | 3.13.1 | REST API upload handling — `PostDocumentView` receives and buffers document data in memory |
| PyPI | django-q | 1.3.9 | Task queue — `consume_file` tasks are dispatched asynchronously; `Q_CLUSTER["recycle"] = 1` forces worker recycling |
| PyPI | scikit-learn | 1.0.2 | ML classifier — `MLPClassifier`, `CountVectorizer`, `MultiLabelBinarizer` are pickle-loaded into memory per consumption |
| PyPI | ocrmypdf | 13.4.3 | OCR pipeline — spawns subprocesses but orchestration objects remain in parent process memory |
| PyPI | pikepdf | 5.1.1 | PDF manipulation — `pikepdf.open()` loads PDF structure into memory for metadata extraction and barcode splitting |
| PyPI | Pillow | 9.1.0 | Image processing — `Image.open()` for DPI checks, alpha detection, thumbnail creation |
| PyPI | pdfminer.six | 20220319 | PDF text extraction — `extract_text()` reads and processes entire PDF content into strings |
| PyPI | whoosh | 2.7.4 | Full-text search — `AsyncWriter` and `Searcher` objects hold index state in memory |
| PyPI | fuzzywuzzy | 0.18.0 | Fuzzy matching — `partial_ratio()` operates on full document content copies |
| PyPI | dateparser | 1.1.1 | Date parsing — lazily imported heavy module; `dateparser.parse()` called per regex match |
| PyPI | python-magic | 0.4.25 | MIME detection — calls into libmagic C library |
| PyPI | filelock | 3.6.0 | File locking — `FileLock(settings.MEDIA_LOCK)` used during file writes |
| PyPI | channels | 3.0.4 | WebSocket — `get_channel_layer()` for progress notifications during consumption |
| PyPI | channels-redis | 3.4.0 | Redis-backed channel layer — capacity 2000 messages, 15-second expiry |
| PyPI | redis | 3.5.3 | Redis client — task broker and channel layer backend |
| PyPI | tika | 1.24 | Tika client — sends documents to external Tika server, receives full parsed responses |
| PyPI | pdf2image | 1.16.0 | PDF page rendering — `convert_from_path()` renders all pages as PIL images for barcode scanning |
| PyPI | pyzbar | 0.1.9 | Barcode detection — `pyzbar.decode()` operates on PIL image objects in memory |
| PyPI | tqdm | 4.64.0 | Progress bars — minimal memory impact |
| PyPI | python-gnupg | 0.4.8 | GPG decryption (legacy) — relevant for encrypted document processing path |

### 0.3.2 Dependency Updates

No dependency updates are required for this investigation. The analysis is conducted against the existing dependency set. The deliverable is a documentation file that does not introduce any new imports or modify existing code.

#### Import Analysis (Read-Only)

The following import chains are relevant to understanding memory allocation flow during document consumption:

- **`consumer.py`** imports: `classifier.load_classifier`, `parsers.get_parser_class_for_mime_type`, `parsers.parse_date`, `models.Document/Correspondent/DocumentType/Tag/FileInfo`, `signals.document_consumption_finished/started`
- **`classifier.py`** imports: `pickle`, `documents.models.Document/MatchingModel` — and lazily imports `sklearn.feature_extraction.text.CountVectorizer`, `sklearn.neural_network.MLPClassifier`, `sklearn.preprocessing.MultiLabelBinarizer/LabelBinarizer`
- **`matching.py`** imports: `documents.models.Correspondent/DocumentType/Tag/MatchingModel` — and lazily imports `fuzzywuzzy.fuzz`
- **`parsers.py`** lazily imports: `dateparser` (inside `parse_date.__parser()`)
- **`tasks.py`** imports at module level: `pdf2image.convert_from_path`, `pikepdf.Pdf`, `pyzbar.pyzbar`, `whoosh.writing.AsyncWriter`

#### External Reference Context

| File | Reference Type | Relevance |
|------|---------------|-----------|
| `Pipfile` | Dependency declaration | Source of version ranges (e.g., `scikit-learn==1.0.2`, `django~=4.0`) |
| `requirements.txt` | Pinned manifest | Exact versions for all 113 packages |
| `Dockerfile` | Runtime base image | `python:3.9-slim-bullseye` — establishes CPython 3.9 GC behavior baseline |
| `.build-config.json` | Build tool versions | qpdf 10.6.3 and jbig2enc 0.29 compiled from source |
| `src/paperless/settings.py` | Runtime configuration | Task worker count, thread settings, directory paths, Redis connection |


## 0.4 Integration Analysis


### 0.4.1 Existing Code Touchpoints

The memory investigation requires deep analysis of the following integration points where memory allocation, accumulation, and release occur during document import operations. No modifications are made — these are read-only analysis targets.

#### Document Consumption Pipeline (Primary Memory Flow)

The consumer pipeline in `src/documents/consumer.py` method `try_consume_file()` (lines 180–377) orchestrates the entire import flow. Each stage represents a memory allocation event:

- **`pre_check_duplicate()`** (lines 102–113): Opens the file and calls `f.read()` to compute MD5 hash. The entire file content is held in memory until the hash is computed and the `with` block exits. For a 50MB PDF, this allocates 50MB of heap.
- **`magic.from_file()`** (line 219): Calls into the libmagic C library. Reads file header bytes — minimal memory impact.
- **`parser_class(self.logging_group, progress_callback)`** (line 244): Parser instantiation creates a temporary directory via `tempfile.mkdtemp()` in `settings.SCRATCH_DIR`.
- **`document_parser.parse()`** (line 261): Dispatches to the specific parser. For `RasterisedDocumentParser`, this invokes OCRmyPDF (subprocess) and pdfminer text extraction. For `TextDocumentParser`, reads entire file into `self.text`. For `TikaDocumentParser`, sends file to Tika and holds response.
- **`document_parser.get_optimised_thumbnail()`** (line 265): Generates thumbnail via ImageMagick subprocess, optionally runs OptiPNG.
- **`document_parser.get_text()`** (line 271): Returns `self.text` — the full document text string held in the parser instance.
- **`parse_date(self.filename, text)`** (line 275): Iterates regex matches in document text, calling `dateparser.parse()` for each match.
- **`load_classifier()`** (line 292): Deserializes the classifier pickle file containing vectorizer, binarizers, and ML models.
- **`self._store(text, date, mime_type)`** (lines 379–412): Reads file AGAIN for checksum (`f.read()` at line 402), creates the Document ORM object with full content text.
- **`document_consumption_finished.send()`** (lines 306–311): Fires 6 signal handlers sequentially, each performing database queries and matching logic.
- **File write operations** (lines 315–337): `_write()` method reads entire file into memory and writes to destination — no streaming.
- **`document_parser.cleanup()`** (line 369): Deletes temp directory via `shutil.rmtree()`.

#### Signal Handler Chain (Post-Consumption Memory Events)

Connected in `src/documents/apps.py` (lines 22–27), these handlers fire within the same database transaction:

- **`add_inbox_tags`** (`handlers.py` line 30): `Tag.objects.filter(is_inbox_tag=True)` — lightweight query.
- **`set_correspondent`** (`handlers.py` lines 35–98): Calls `matching.match_correspondents(document, classifier)` which loads ALL Correspondent objects from DB (`Correspondent.objects.all()`) and evaluates each against the document. The classifier also runs `predict_correspondent()` which transforms content through the vectorizer.
- **`set_document_type`** (`handlers.py` lines 101–165): Same pattern — loads ALL DocumentType objects and evaluates each. Classifier runs `predict_document_type()`.
- **`set_tags`** (`handlers.py` lines 168–230): Loads ALL Tag objects from DB, evaluates each against document, and classifier runs `predict_tags()`.
- **`set_log_entry`** (`handlers.py` lines 413–425): Creates LogEntry — minimal memory.
- **`add_to_index`** (`handlers.py` lines 428–431): Opens Whoosh index writer, updates document entry, commits.

#### Memory Accumulation Pattern During Signal Handlers

The classifier is loaded once (line 292 of `consumer.py`) and passed to each signal handler. However, each handler independently:
1. Queries ALL matching model instances from the database
2. Invokes the classifier's prediction methods, which call `self.data_vectorizer.transform()` creating a new sparse matrix per call
3. For fuzzy matching (`MATCH_FUZZY` in `matching.py` lines 127–145), creates regex-stripped copies of the entire document content

This means for a single document consumption, the classifier's vectorizer is invoked 3 times (once per prediction type), and 3 separate `QuerySet.all()` loads occur.

#### Cross-Cutting Integration Points

| Integration Point | File | Memory Relevance |
|-------------------|------|-----------------|
| Django-Q task dispatch | `src/documents/management/commands/document_consumer.py` line 86 | `async_task()` serializes arguments for Redis queue; `Q_CLUSTER["recycle"] = 1` forces worker process recycling after each task, releasing all per-process memory |
| WebSocket progress updates | `src/documents/consumer.py` lines 56–76 | `async_to_sync(self.channel_layer.group_send)` sends progress payloads through Redis channel layer |
| Post-save signal | `src/documents/signals/handlers.py` lines 310–410 | `update_filename_and_move_files` fires on Document save, regenerating filenames and potentially moving files |
| Database transactions | `src/documents/consumer.py` lines 298–367 | `transaction.atomic()` wraps store + signal handlers + file writes; rollback on failure |
| File locking | `src/documents/consumer.py` line 315 | `FileLock(settings.MEDIA_LOCK)` serializes file system operations |

### 0.4.2 Database and Schema Analysis

No schema updates are required. The investigation examines existing model behavior:

- `Document.content` (TextField) — stores full document text; loaded into memory during matching and classification
- `Document.checksum` / `archive_checksum` — computed via full file reads
- `MatchingModel` (abstract base for Correspondent, Tag, DocumentType) — all instances loaded for rule matching
- `Log` model — new entries created per consumption via signal handler

### 0.4.3 Service and Middleware Interactions

| Service | Integration Method | Memory Impact |
|---------|-------------------|---------------|
| Redis | Task broker + channel layer | Task arguments serialized to Redis; channel messages buffered (capacity 2000, expiry 15s) |
| Whoosh index | `AsyncWriter` in `index.py` | Writer holds modified segments in memory until commit |
| Django ORM | QuerySet evaluation | `QuerySet.all()` in matching loads all model instances; `Document.objects.create()` in store |
| Python `pickle` | Classifier load/save | Entire classifier model graph deserialized into memory |
| `subprocess.Popen` | OCRmyPDF, ImageMagick, Ghostscript, OptiPNG | Child processes have independent memory; parent waits via `.wait()` |


## 0.5 Technical Implementation


### 0.5.1 File-by-File Execution Plan

Since this is a read-only investigation, the execution plan describes analysis activities per file rather than modifications. One new file will be created as the investigation deliverable.

#### Group 1 — Core Analysis Targets (Memory Hotspot Files)

- **ANALYZE: `src/documents/consumer.py`** — Trace memory through `try_consume_file()` lifecycle. Key hotspots:
  - Line 103: `f.read()` in `pre_check_duplicate()` — reads full file for MD5; file reference released on `with` exit but buffer held until GC
  - Line 292: `load_classifier()` — deserializes pickle model; classifier object persists until function scope exit at line 377
  - Lines 397–406: `_store()` reads file AGAIN for checksum — second full-file allocation
  - Lines 430–432: `_write()` reads entire file into memory (`read_file.read()`) then writes — no streaming, peak memory equals file size
  - Lines 339–342: Archive file read for checksum — third potential full-file read
- **ANALYZE: `src/documents/classifier.py`** — Evaluate classifier memory footprint:
  - Lines 77–93: `load()` deserializes 6 pickle objects: format version, data hash, vectorizer, tags binarizer, tags classifier, correspondent classifier, document type classifier
  - Lines 115–249: `train()` loads ALL documents' preprocessed content into a Python `list` (line 117–128), plus label lists
  - Lines 251–292: `predict_*` methods create sparse matrices via `self.data_vectorizer.transform()`
- **ANALYZE: `src/documents/matching.py`** — Evaluate per-document matching cost:
  - Lines 21–31: `match_correspondents()` — `Correspondent.objects.all()` + `filter(lambda)` evaluates all correspondents
  - Lines 34–44: `match_document_types()` — same pattern with DocumentType
  - Lines 47–57: `match_tags()` — same pattern with Tag
  - Lines 127–145: Fuzzy matching creates regex-stripped copies of document content
- **ANALYZE: `src/documents/parsers.py`** — Date parsing and parser dispatch:
  - Lines 212–274: `parse_date()` iterates all regex matches, each calling `dateparser.parse()` which is a heavy operation
  - Lines 81–98: `get_parser_class_for_mime_type()` calls `document_consumer_declaration.send(None)` iterating all registered parsers

#### Group 2 — Parser-Specific Analysis

- **ANALYZE: `src/paperless_tesseract/parsers.py`** — Heaviest parser, OCR path:
  - Lines 26–55: `extract_metadata()` opens PDF with `pikepdf`, iterates metadata
  - Lines 99–133: `extract_text()` reads sidecar file or calls `pdfminer_extract_text()`
  - Lines 186–215: Image-processing path opens images multiple times with `Image.open()` for alpha/DPI checks
  - Lines 230–328: `parse()` orchestrates OCRmyPDF, text extraction, and fallback logic
- **ANALYZE: `src/paperless_text/parsers.py`** — Lightest parser:
  - Line 42: Reads entire file into `self.text`
  - Lines 26–37: Creates PIL Image for thumbnail
- **ANALYZE: `src/paperless_tika/parsers.py`** — External service parser:
  - Line 55: `parser.from_file()` holds full Tika response
  - Line 96: `response.content` holds entire converted PDF in memory

#### Group 3 — Signal Handlers and Index

- **ANALYZE: `src/documents/signals/handlers.py`** — Post-consumption handlers:
  - Lines 30–32: `add_inbox_tags` — lightweight
  - Lines 35–98: `set_correspondent` — full matching evaluation
  - Lines 101–165: `set_document_type` — full matching evaluation
  - Lines 168–230: `set_tags` — full matching evaluation
  - Lines 428–431: `add_to_index` — Whoosh index writer
- **ANALYZE: `src/documents/index.py`** — Search index operations:
  - Lines 64–74: `open_index_writer()` — creates `AsyncWriter`, commits on exit
  - Lines 87–107: `update_document()` — serializes all document metadata for indexing

#### Group 4 — Deliverable

- **CREATE: `blitzy/documentation/<source_branch_name>.md`** — Comprehensive analysis document containing:
  - Executive summary of memory investigation findings
  - Stage-by-stage memory profile of the consumption pipeline
  - Identification of specific hotspot methods with line references
  - Comparison of memory behavior across document types (PDF, text, Office)
  - Analysis of batch size impact on cumulative memory
  - Evidence of whether memory retention is GC-related or code-structural
  - Recommendations for observation (not code changes)

### 0.5.2 Implementation Approach

The investigation follows this analytical methodology:

- **Establish the memory lifecycle** by tracing every allocation point in `consumer.py`'s `try_consume_file()` through the parser, classifier, signal handler, and file-write phases
- **Identify double-read patterns** where the same file content is loaded into memory multiple times (duplicate check + store checksum + archive checksum + file copy)
- **Evaluate classifier overhead** by analyzing the pickle model structure and the per-document cost of vectorizer transforms and classifier predictions
- **Assess QuerySet impact** by identifying where `.all()` loads entire tables versus filtered queries
- **Analyze scope-based retention** by determining when large objects (parser text, classifier, file buffers) go out of scope and become eligible for GC
- **Compare ingestion pathways** to identify why certain document sources or types exhibit different memory profiles (e.g., Tesseract parser with image processing vs. TextDocumentParser with simple file read)
- **Document findings** with precise file paths, line numbers, and code-based rationale

### 0.5.3 Key Memory Behavior Analysis Points

The investigation must answer these specific questions with code-referenced evidence:

| Question | Analysis Location | Expected Finding |
|----------|------------------|-----------------|
| Why do memory spikes occur during metadata handling? | `consumer.py` lines 292–311 (classifier load + signal handlers) | Classifier pickle deserialization + 3× matching evaluations loading all model instances |
| Why is memory disproportionate to document size? | `consumer.py` lines 103, 402, 430–432 | File read into memory 2–3 times; classifier model can be much larger than document |
| Why doesn't memory release promptly? | `consumer.py` lines 180–377 (function scope) | Large objects (classifier, parser, text) persist until `try_consume_file()` returns; CPython reference counting may not reclaim cycles immediately |
| What varies between spiking and non-spiking cases? | Parser selection at lines 223–225 | Tesseract parser (images, PDFs) allocates significantly more than TextDocumentParser; barcode processing adds pdf2image rendering |
| How does batch size affect memory? | `tasks.py` line 236; `settings.py` Q_CLUSTER | `Q_CLUSTER["recycle"] = 1` recycles workers per task, resetting memory; but within one task, all allocations accumulate |
| Is this normal GC behavior? | CPython reference counting analysis | Most objects are reference-counted and freed on scope exit; cyclic references (Django model back-references) may delay collection |


## 0.6 Scope Boundaries


### 0.6.1 Exhaustively In Scope

All files and components subject to analysis in this memory investigation:

**Core Consumption Pipeline:**
- `src/documents/consumer.py` — all methods, particularly `try_consume_file()`, `pre_check_duplicate()`, `_store()`, `_write()`
- `src/documents/tasks.py` — `consume_file()`, `barcode_reader()`, `scan_file_for_separating_barcodes()`, `separate_pages()`
- `src/documents/parsers.py` — `parse_date()`, `get_parser_class_for_mime_type()`, `DocumentParser` base class, `run_convert()`

**Classifier and Matching:**
- `src/documents/classifier.py` — `load_classifier()`, `DocumentClassifier.load()`, `DocumentClassifier.train()`, `predict_*()` methods
- `src/documents/matching.py` — `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()`, `_split_match()`

**Signal Handlers and Lifecycle:**
- `src/documents/signals/__init__.py` — signal definitions
- `src/documents/signals/handlers.py` — all receivers: `add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index`, `update_filename_and_move_files`, `cleanup_document_deletion`
- `src/documents/apps.py` — signal connection configuration

**Parser Implementations:**
- `src/paperless_tesseract/parsers.py` — `RasterisedDocumentParser` (OCR pipeline)
- `src/paperless_text/parsers.py` — `TextDocumentParser` (plain text)
- `src/paperless_tika/parsers.py` — `TikaDocumentParser` (Office documents)
- `src/paperless_tesseract/signals.py` — parser registration metadata

**Search and Indexing:**
- `src/documents/index.py` — `open_index_writer()`, `update_document()`, `add_or_update_document()`, `DelayedQuery`
- `src/documents/file_handling.py` — `generate_unique_filename()`, `generate_filename()`, `many_to_dictionary()`

**Models and Data Layer:**
- `src/documents/models.py` — `Document`, `Correspondent`, `Tag`, `DocumentType`, `MatchingModel`, `FileInfo`
- `src/documents/loggers.py` — `LoggingMixin`

**Ingestion Entry Points:**
- `src/documents/management/commands/document_consumer.py` — filesystem watcher
- `src/documents/management/commands/document_importer.py` — manifest import
- `src/documents/views.py` — `PostDocumentView` (API upload)
- `src/paperless_mail/mail.py` — email ingestion

**Configuration:**
- `src/paperless/settings.py` — task worker settings, directory paths, OCR settings, Q_CLUSTER configuration
- `src/setup.cfg` — test environment settings
- `Pipfile` — dependency version declarations
- `requirements.txt` — pinned dependency versions
- `Dockerfile` — runtime base image and environment

**Test Files (Reference for Expected Behavior):**
- `src/documents/tests/test_consumer.py`
- `src/documents/tests/test_classifier.py`
- `src/documents/tests/test_tasks.py`
- `src/documents/tests/test_parsers.py`
- `src/documents/tests/test_matchables.py`
- `src/documents/tests/test_file_handling.py`

**Deliverable:**
- `blitzy/documentation/<source_branch_name>.md` — investigation results document

### 0.6.2 Explicitly Out of Scope

- **Frontend code** (`src-ui/**/*`) — Angular SPA does not participate in backend memory behavior
- **Docker/deployment infrastructure** (`docker/**/*`, `Dockerfile`, `docker-compose*.yml`) — container-level memory limits are a deployment concern, not an application code analysis target
- **Documentation infrastructure** (`docs/**/*`, `.readthedocs.yml`) — Sphinx documentation build is unrelated
- **CI/CD workflows** (`.github/**/*`) — GitHub Actions pipelines do not affect runtime memory
- **Database migrations** (`src/documents/migrations/**/*`) — schema history is irrelevant to runtime memory behavior
- **Static assets** (`src/documents/static/**/*`, `src/documents/templates/**/*`) — CSS/HTML templates have no memory impact
- **Resource files** (`resources/**/*`) — branding assets are not in the processing path
- **Internationalization** (`src/locale/**/*`) — translation strings are minimal memory consumers
- **Code modification or optimization** — the user explicitly prohibits modifying source files; the scope is analysis and documentation only
- **Performance optimization beyond memory** — CPU profiling, I/O throughput, and network latency are not in scope
- **Frontend memory behavior** — browser-side memory consumption is excluded


## 0.7 Rules for Feature Addition


### 0.7.1 User-Specified Rules

The following rules are explicitly emphasized by the user and must be strictly observed:

- **SWE-AtlasQnA-Repo Rule:** Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the question(s) posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Do not add any other code in the source repository (besides the above requested document). Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **No Source Modification Rule:** "Don't modify any repository source files." This is the user's explicit directive. All existing `.py`, `.cfg`, `.json`, `.yml`, `.md`, and other repository files must remain unchanged.
- **Temporary Script Rule:** "You can create temporary test scripts or helper tools to reproduce and analyze the behavior, but clean them up and leave the codebase unchanged when done." Any profiling or analysis scripts created during the investigation are ephemeral and must be deleted before completion.
- **Evidence-Based Analysis Rule:** "Show me what patterns you can find and where the memory is actually going." All findings must reference specific code locations with file paths and line numbers. Speculation without code evidence is not permitted.

### 0.7.2 Investigation-Specific Conventions

- All analysis conclusions must trace back to specific source files and line numbers in the repository
- The investigation document must answer each user question individually and explicitly
- Memory behavior must be explained in terms of Python/CPython mechanics (reference counting, cyclic GC, scope-based lifetime)
- Comparisons between document types must reference the specific parser implementations and their distinct memory profiles
- The classifier's pickle-based model loading must be analyzed for its actual in-memory footprint impact
- Signal handler chain behavior must be documented in execution order as defined in `apps.py`

### 0.7.3 Documentation Standards

- The deliverable markdown document must be placed in `blitzy/documentation/` directory
- The document must be self-contained and readable without requiring access to the source code
- Code references should use the format: `file_path` (line N) or `file_path` (lines N–M)
- The document must include clear section headers for each investigation question
- Rationale and thinking must accompany every finding


## 0.8 References


### 0.8.1 Repository Files and Folders Analyzed

The following is a comprehensive list of all files and folders searched, retrieved, and analyzed to derive the conclusions in this Agent Action Plan:

**Root-Level Configuration Files:**
- `Pipfile` — Python dependency declarations (runtime and dev packages, version constraints)
- `requirements.txt` — Pinned Python dependency manifest (113 packages with exact versions)
- `Dockerfile` — Production container image recipe (Python 3.9-slim-bullseye base)
- `gunicorn.conf.py` — ASGI server configuration
- `paperless.conf.example` — Configuration variable documentation
- `.build-config.json` — Build tool version pins (qpdf, jbig2enc)
- `.editorconfig` — Code formatting standards
- `.pre-commit-config.yaml` — Pre-commit hook configuration

**Core Document Processing (`src/documents/`):**
- `src/documents/__init__.py` — Package initializer, exports system checks
- `src/documents/apps.py` — Django app configuration, signal handler connections (6 handlers)
- `src/documents/consumer.py` — Document ingestion coordinator (432 lines)
- `src/documents/tasks.py` — Background task definitions (281 lines)
- `src/documents/parsers.py` — Parser base class, MIME dispatch, date parsing (351 lines)
- `src/documents/classifier.py` — ML document classifier with pickle persistence (293 lines)
- `src/documents/matching.py` — Rule-based matching engine (172 lines)
- `src/documents/models.py` — ORM model definitions (467 lines)
- `src/documents/index.py` — Whoosh search index operations (288 lines)
- `src/documents/file_handling.py` — File naming and directory management (200 lines)
- `src/documents/loggers.py` — Logging mixin utility (22 lines)
- `src/documents/sanity_checker.py` — Archive integrity checker (134 lines)
- `src/documents/bulk_download.py` — Bulk ZIP download strategies (59 lines)
- `src/documents/serialisers.py` — DRF serializer definitions (partial read)
- `src/documents/views.py` — API views including PostDocumentView (partial read, lines 1–80, 485–550)
- `src/documents/settings.py` — Export/import manifest constants (5 lines)
- `src/documents/signals/__init__.py` — Signal definitions (5 lines)
- `src/documents/signals/handlers.py` — Signal receivers for document lifecycle (432 lines)

**Parser Implementations:**
- `src/paperless_tesseract/parsers.py` — OCR/Tesseract parser (342 lines)
- `src/paperless_tesseract/signals.py` — Parser registration metadata
- `src/paperless_tesseract/apps.py` — App configuration and signal wiring
- `src/paperless_tesseract/checks.py` — Tesseract language validation
- `src/paperless_text/parsers.py` — Plain text parser (43 lines)
- `src/paperless_tika/parsers.py` — Tika/Gotenberg parser (100 lines)

**Management Commands:**
- `src/documents/management/commands/document_consumer.py` — Filesystem watcher (241 lines)
- `src/documents/management/commands/document_importer.py` — Manifest-based import (175 lines)

**Project Settings:**
- `src/paperless/settings.py` — Django/Paperless configuration (615 lines, fully read)
- `src/setup.cfg` — pytest, flake8, coverage configuration

**Test Files (Reference):**
- `src/documents/tests/` — Full test suite folder structure reviewed (30 test files identified)

**Folder Structures Explored:**
- Root folder (`""`) — Full repository overview
- `src/` — Backend source tree structure
- `src/documents/` — Core documents app structure
- `src/documents/signals/` — Signal handler package
- `src/documents/management/` — Management command namespace
- `src/documents/management/commands/` — All 14 management commands
- `src/documents/tests/` — Test suite structure
- `src/paperless_tesseract/` — OCR parser package
- `src/paperless_text/` — Text parser package (via folder summary)
- `src/paperless_tika/` — Tika parser package (via folder summary)

### 0.8.2 Tech Spec Sections Referenced

- **Section 1.1 — Executive Summary:** Project overview, version (1.7.0), architecture context
- **Section 2.1 — Feature Catalog:** Feature definitions for F-001 (Ingestion), F-002 (OCR), F-003 (Search), F-004 (Classification), F-005 (Metadata), F-010 (File Management), F-012 (Sanity Checker), F-016 (CLI), F-017 (Deployment)
- **Section 3.1 — Programming Languages:** Python 3.9 runtime, CI matrix (3.8, 3.9, 3.10)
- **Section 3.3 — Open Source Dependencies:** Complete package inventory with pinned versions

### 0.8.3 Attachments

No attachments were provided by the user for this project.

### 0.8.4 External URLs

No external URLs were specified by the user. No Figma designs or external resources are referenced.


