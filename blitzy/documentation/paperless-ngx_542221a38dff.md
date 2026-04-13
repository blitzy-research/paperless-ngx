# Runtime Memory Profiling and Analysis: Paperless-ngx Document Import Pipeline

**Branch:** `paperless-ngx_542221a38dff`
**Date:** 2024
**Scope:** Comprehensive memory investigation of the document consumption pipeline in Paperless-ngx v1.7.0
**Runtime:** CPython 3.9 on `python:3.9-slim-bullseye`

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Document Consumption Pipeline — Stage-by-Stage Memory Analysis](#2-document-consumption-pipeline--stage-by-stage-memory-analysis)
   - 2.1 [File Existence and Duplicate Check](#21-file-existence-and-duplicate-check)
   - 2.2 [MIME Detection](#22-mime-detection)
   - 2.3 [Parser Dispatch](#23-parser-dispatch)
   - 2.4 [Document Parsing](#24-document-parsing)
   - 2.5 [Thumbnail Generation](#25-thumbnail-generation)
   - 2.6 [Text Extraction and Date Parsing](#26-text-extraction-and-date-parsing)
   - 2.7 [Classifier Loading](#27-classifier-loading)
   - 2.8 [Document Storage](#28-document-storage-_store)
   - 2.9 [Signal Handler Chain](#29-signal-handler-chain-document_consumption_finished)
   - 2.10 [File Write Operations](#210-file-write-operations)
   - 2.11 [Parser Cleanup](#211-parser-cleanup)
3. [Memory Hotspot Summary Table](#3-memory-hotspot-summary-table)
4. [Metadata Handling — Object Copies and Reference Retention](#4-metadata-handling--object-copies-and-reference-retention)
5. [Caching Behavior Investigation](#5-caching-behavior-investigation)
6. [Ingestion Pathway Comparison](#6-ingestion-pathway-comparison)
7. [Document Type Impact on Memory](#7-document-type-impact-on-memory)
8. [Batch Size Impact on Cumulative Memory](#8-batch-size-impact-on-cumulative-memory)
9. [GC Behavior Analysis — Normal vs. Problematic](#9-gc-behavior-analysis--normal-vs-problematic)
10. [Peak Memory Model](#10-peak-memory-model)
11. [Recommendations for Observation](#11-recommendations-for-observation)
12. [Conclusion](#12-conclusion)

---

## 1. Executive Summary

Memory spikes during document imports in Paperless-ngx are primarily driven by **four compounding factors**, not a single root cause. The most significant contributor is the **ML classifier model deserialization**: every document consumption triggers a full pickle load of a `CountVectorizer` plus three `MLPClassifier` neural network models from `src/documents/classifier.py` (lines 76–94). This classifier payload can range from 50 to 200+ MB depending on corpus size, and it is loaded **regardless of the incoming document's size**. A 5 KB text file triggers the exact same classifier load as a 50 MB PDF. This single factor explains why memory consumption appears disproportionate to document size, especially for text-based documents with small metadata.

The second major factor is **repeated full-file reads**: the same file is read entirely into memory up to three times during a single consumption — once for the duplicate check MD5 (`src/documents/consumer.py` line 103), again for the storage checksum (line 402), and a third time for the non-streaming file copy (lines 429–432). Archive files undergo an additional full read for their checksum (lines 339–342). None of these reads use incremental/chunked processing despite `hashlib.md5()` supporting `update()` calls. This means a 50 MB document temporarily allocates up to 150–200 MB in file-buffer memory alone.

The third factor is the **signal handler chain's matching logic**: after each document is stored, six signal handlers fire sequentially (`src/documents/apps.py` lines 22–27). Three of these — `set_correspondent`, `set_document_type`, and `set_tags` — each independently call `Correspondent.objects.all()` / `DocumentType.objects.all()` / `Tag.objects.all()` (loading every matching-model instance from the database into memory), invoke the classifier's `predict_*()` method (which calls `preprocess_content()` and `data_vectorizer.transform()`, creating new string copies and sparse matrices each time), and for fuzzy-matching models, create full-text copies of the document content stripped of punctuation. This results in 3× classifier vectorization, 3× preprocessed content copies, 3× full table scans, and N× fuzzy-match text copies — all for a single document. Worker recycling via `Q_CLUSTER["recycle"] = 1` (`src/paperless/settings.py` line 452) prevents cross-document accumulation by exiting the worker process after each task, but does **nothing** to reduce peak within-task memory.

---

## 2. Document Consumption Pipeline — Stage-by-Stage Memory Analysis

The entire document import pipeline is coordinated by the `try_consume_file()` method in `src/documents/consumer.py` (lines 180–377). This section traces each pipeline stage chronologically, documenting exactly what memory is allocated, when it is released, and whether it constitutes a hotspot.

### 2.1 File Existence and Duplicate Check

**Code Location:** `pre_check_duplicate()` in `src/documents/consumer.py` (lines 102–113)

**What happens:**

```python
# consumer.py lines 103-104
with open(self.path, "rb") as f:
    checksum = hashlib.md5(f.read()).hexdigest()
```

The method opens the target file in binary mode and calls `f.read()` — which reads the **entire file contents** into a single `bytes` object on the Python heap — then passes this bytes object to `hashlib.md5()` to compute a hexadecimal digest.

**Memory allocated:** One `bytes` object equal to the full file size. For a 50 MB PDF, this temporarily places 50 MB on the heap.

**When released:** The `f.read()` return value is a temporary passed directly to `hashlib.md5()`. Once the MD5 computation completes and the `with` block exits, the reference count on the bytes object drops to zero, and CPython's reference-counting GC immediately frees it. The resulting `checksum` string (32 hex characters = 32 bytes) persists until the function returns.

**Proportionate to document size?** Yes — the allocation is exactly 1× the file size.

**Hotspot?** **Yes.** This is the **first** of multiple full-file reads for the same file. The `hashlib` module natively supports incremental hashing via `md5.update(chunk)`, which would allow processing the file in fixed-size chunks (e.g., 8 KB at a time) with negligible memory overhead. The current `f.read()` approach forces a full file-size allocation. Additionally, the computed checksum value is **not preserved** for later reuse — it is used only for the duplicate comparison and then discarded, forcing a second identical read in `_store()` later.

**Rationale:** This is a clear code-structural anti-pattern, not a Python/GC limitation. Chunked hashing would reduce this stage's memory footprint from O(file_size) to O(chunk_size) — typically 8 KB regardless of file size.

---

### 2.2 MIME Detection

**Code Location:** `src/documents/consumer.py` line 219

```python
mime_type = magic.from_file(self.path, mime=True)
```

**What happens:** Calls `python-magic`, which invokes the `libmagic` C library. Libmagic reads only the first few kilobytes of file header bytes to identify the MIME type.

**Memory allocated:** Minimal — a small buffer for file header inspection (typically < 4 KB) plus the returned MIME type string.

**When released:** Immediately after the call returns.

**Proportionate to document size?** No — constant overhead regardless of file size.

**Hotspot?** **No.** This is an efficient, well-bounded operation.

---

### 2.3 Parser Dispatch

**Code Location:** `src/documents/consumer.py` line 223, dispatching to `src/documents/parsers.py` lines 81–98

```python
parser_class = get_parser_class_for_mime_type(mime_type)
```

**What happens:** The function `get_parser_class_for_mime_type()` (parsers.py lines 81–98) fires the `document_consumer_declaration` Django signal by calling `document_consumer_declaration.send(None)` at line 87. This signal causes all registered parser apps (Tesseract, Text, Tika) to respond with dictionaries declaring their supported MIME types and parser class references. The function iterates these responses, filters by matching MIME type, and returns the parser class with the highest weight.

**Memory allocated:** A list of small dictionaries (typically 3 entries, one per parser app). Each dictionary contains MIME type strings, a weight integer, and a class reference. Total: a few hundred bytes.

**When released:** The response list and intermediate variables are released when `get_parser_class_for_mime_type()` returns. The selected `parser_class` reference persists as a local variable in `try_consume_file()`.

**Proportionate to document size?** No — constant overhead.

**Hotspot?** **No.** Negligible memory impact.

---

### 2.4 Document Parsing

**Code Location:** `src/documents/consumer.py` line 261

```python
document_parser.parse(self.path, mime_type, self.filename)
```

This dispatches to one of three parser implementations depending on MIME type. Each has a dramatically different memory profile:

#### 2.4a RasterisedDocumentParser (Tesseract/OCR — PDF and Image Documents)

**Code Location:** `src/paperless_tesseract/parsers.py` lines 230–328

This is the **heaviest parser** and the most common, handling all PDF and image documents.

**Key memory events:**

1. **pikepdf PDF structure load** (line 34 in `extract_metadata`): `pikepdf.open(document_path)` loads the PDF's internal structure (page tree, metadata dictionaries, cross-reference tables) into memory. For a multi-page PDF, this can be 2–5× the file size due to decompressed stream objects.

2. **Triple `Image.open()` calls** (lines 73–97 in `has_alpha`, `get_dpi`, `calculate_a4_dpi`): For image files, `Image.open()` is called three separate times on the same file. While PIL's `Image.open()` is lazy (deferring pixel decode), accessing `.mode`, `.info["dpi"]`, and `.size` may trigger partial header decoding each time. Each call opens a new file handle.

3. **Alpha channel removal** (lines 197–201 in `construct_ocrmypdf_parameters`): For images with alpha channels, the code opens the image a **fourth** time, creates a new blank RGBA background image at full resolution (`Image.new("RGBA", im.size, (255, 255, 255))`), composites the original onto it, converts to RGB, and saves. This **temporarily holds 2 full-resolution uncompressed images in memory simultaneously**. For a 4000×3000 RGBA image, that's approximately 2 × (4000 × 3000 × 4 bytes) ≈ **96 MB**.

4. **OCRmyPDF invocation** (line 261): `ocrmypdf.ocr(**args)` spawns Tesseract as a subprocess. The actual OCR processing runs in child processes with independent memory spaces. However, the `ocrmypdf` Python module itself and its orchestration objects (argument structures, logging handlers) remain in the parent process.

5. **Text extraction** (lines 99–133 in `extract_text`): Either reads a sidecar text file (line 101–102: `f.read()` on the OCR sidecar) or calls `pdfminer_extract_text(pdf_file)` at line 120. The pdfminer extraction processes the entire PDF and produces a single Python string containing all extracted text, stored in `self.text`. For a text-heavy 100-page document, this could be 500 KB–5 MB of text.

6. **Reference retention**: The parser instance holds `self.text` (full document text), `self.archive_path` (file path string), and `self.tempdir` (path string) until `cleanup()` is called at `consumer.py` line 369 in the `finally` block.

**Estimated memory for a 10 MB PDF:** pikepdf structure ~20–50 MB + extracted text ~100 KB–1 MB + subprocess orchestration overhead ~5 MB = **25–56 MB** parser-specific allocation.

#### 2.4b TextDocumentParser (Plain Text Documents)

**Code Location:** `src/paperless_text/parsers.py` lines 40–42

```python
def parse(self, document_path, mime_type, file_name=None):
    with open(document_path, "r") as f:
        self.text = f.read()
```

This is the **lightest parser**. It reads the entire file into `self.text` as a single string.

**Memory allocated:** 1× the file size (as a Python `str` object, which uses more memory than the raw bytes due to Unicode encoding — approximately 1–4 bytes per character depending on content).

**When released:** `self.text` persists on the parser instance until `cleanup()` at `consumer.py` line 369.

**Thumbnail generation** (lines 19–37): Creates a 500×700 PIL `Image` in RGB mode (~1 MB), draws text onto it, saves to PNG. The image is released after `save()`.

**Estimated memory for a 5 KB text file:** ~5 KB text + ~1 MB thumbnail image = **~1 MB** total.

#### 2.4c TikaDocumentParser (Office Documents)

**Code Location:** `src/paperless_tika/parsers.py` lines 50–72

```python
# Line 55
parsed = parser.from_file(document_path, tika_server)
# Line 62
self.text = parsed["content"].strip()
```

**Key memory events:**

1. **Tika response** (line 55): Sends the document to the Tika server via HTTP. The full Tika response — including extracted text, metadata, and status information — is held in the `parsed` dictionary. For a large Office document, the response can be several MB.

2. **PDF conversion** (lines 74–99 in `convert_to_pdf`): Sends the document to the Gotenberg server for LibreOffice conversion. Line 96: `file.write(response.content)` — `response.content` holds the **entire converted PDF** in memory as a `bytes` object before being written to disk. For a large PowerPoint or Excel file, the converted PDF could be 10–50 MB.

**When released:** `parsed` dict is released when `parse()` returns. The `response.content` bytes in `convert_to_pdf()` are released when that function's scope exits. `self.text` persists on the parser instance.

**Estimated memory for a 5 MB Office document:** Tika response ~2–5 MB + Gotenberg PDF response ~5–15 MB + extracted text ~100 KB = **7–20 MB** (sequentially, not simultaneously — the Tika response can be GC'd before the Gotenberg call).

---

### 2.5 Thumbnail Generation

**Code Location:** `src/documents/consumer.py` lines 265–269, dispatching to parser-specific `get_thumbnail()` then `get_optimised_thumbnail()` in `src/documents/parsers.py` lines 319–340

**What happens:** Each parser produces a thumbnail image file. The base class `get_optimised_thumbnail()` optionally runs OptiPNG as a subprocess (`subprocess.Popen(args).wait()` at parsers.py line 335) to compress the PNG.

**Memory allocated:** Minimal in the parent process — OptiPNG runs as a child subprocess with independent memory. The only parent-side allocation is the output file path string.

**Hotspot?** **No.** Subprocess-based, negligible parent memory impact.

---

### 2.6 Text Extraction and Date Parsing

**Code Location:** `src/documents/consumer.py` lines 271–275, dispatching to `src/documents/parsers.py` lines 212–274

```python
# consumer.py line 271
text = document_parser.get_text()
# consumer.py line 275
date = parse_date(self.filename, text)
```

**Text extraction (line 271):** `get_text()` (parsers.py line 342) simply returns `self.text` — a reference to the same string object already stored on the parser. **No new allocation**; the local variable `text` points to the same object as `document_parser.text`.

**Date parsing (line 275):** `parse_date()` in parsers.py lines 212–274:

1. Iterates all regex matches in the document text via `re.finditer(DATE_REGEX, text)` (line 261). The `DATE_REGEX` pattern (lines 30–37) is a compiled regex matching various date formats. `finditer()` is lazy — it yields matches one at a time without loading all matches into memory simultaneously.

2. For each match, calls the inner `__parser()` function (lines 217–231) which:
   - **Lazily imports `dateparser`** (line 221): `import dateparser`. This is a heavy module (~10–20 MB of code and data when fully loaded). Python's `sys.modules` cache means the first import incurs the cost but subsequent imports in the same process are free lookups. Since `Q_CLUSTER["recycle"] = 1` forces worker process recycling after every task, this module is re-imported for every document.
   - Calls `dateparser.parse(ds, settings={...})` (lines 223–231) which creates temporary datetime objects.

3. Returns the first valid date found, or `None`.

**Memory allocated:** The `dateparser` module import adds ~10–20 MB to the process's resident memory on first use (persists for process lifetime). Each regex match creates a small `Match` object (~200 bytes). Each `dateparser.parse()` call creates temporary datetime-related objects (~1 KB each).

**When released:** Match objects and datetime temporaries are released per-iteration. The `dateparser` module stays loaded in `sys.modules` for the process lifetime but is reclaimed on worker recycling.

**Hotspot?** **Moderate.** The `dateparser` module load is a one-time-per-process cost. For documents with many date-like strings (e.g., financial statements), the iteration could create hundreds of temporary objects, but each is small and immediately released. The more significant issue is that `dateparser` is a heavyweight module that adds a fixed ~10–20 MB overhead.

---

### 2.7 Classifier Loading

**Code Location:** `src/documents/consumer.py` line 292, dispatching to `src/documents/classifier.py` lines 30–57 and 76–94

```python
# consumer.py line 292
classifier = load_classifier()
```

**This is the single largest disproportionate memory allocation in the entire pipeline.**

**What happens:** `load_classifier()` (classifier.py lines 30–57) checks if the model file exists, instantiates a `DocumentClassifier`, and calls `classifier.load()`. The `load()` method (lines 76–94) opens the pickle file and calls `pickle.load(f)` **seven times sequentially** to deserialize:

| Pickle Load | Line | Object | Estimated Size |
|-------------|------|--------|----------------|
| 1 | 78 | `schema_version` (int) | ~28 bytes |
| 2 | 86 | `self.data_hash` (SHA1 digest bytes) | ~20 bytes |
| 3 | 87 | `self.data_vectorizer` — `CountVectorizer` | **10–100+ MB** |
| 4 | 88 | `self.tags_binarizer` — `MultiLabelBinarizer`/`LabelBinarizer` | ~1–10 KB |
| 5 | 90 | `self.tags_classifier` — `MLPClassifier` | **10–50+ MB** |
| 6 | 91 | `self.correspondent_classifier` — `MLPClassifier` | **10–50+ MB** |
| 7 | 92 | `self.document_type_classifier` — `MLPClassifier` | **10–50+ MB** |

The `CountVectorizer` at line 87 is the most variable: it contains a vocabulary dictionary mapping every unique word and bigram found across the entire document corpus to integer indices. For a system with thousands of documents, this vocabulary can contain hundreds of thousands of entries, each being a Python string key + integer value. The `MLPClassifier` objects contain NumPy weight matrices whose dimensions depend on the vocabulary size and hidden layer configuration.

**Rationale for size estimates:** The `CountVectorizer` is created in `train()` (classifier.py lines 194–199) with `analyzer="word"` and `ngram_range=(1, 2)`, meaning it captures both unigrams and bigrams. With `min_df=0.01`, terms appearing in fewer than 1% of documents are pruned, but for a corpus of 1,000 documents, this could still retain 50,000–200,000 unique terms. Each Python dict entry costs ~100 bytes (key string + hash + value), giving a vocabulary dictionary of 5–20 MB. The `MLPClassifier` weight matrices (`coefs_` and `intercepts_`) scale with vocabulary size × hidden layers — for 100,000 features and the default hidden layer size of 100, the primary weight matrix alone is 100,000 × 100 × 8 bytes (float64) = ~80 MB.

**When released:** The `classifier` local variable in `try_consume_file()` holds a reference to the `DocumentClassifier` instance from line 292 through to the function return at line 377. This means the **entire classifier model persists in memory through all signal handlers** (lines 306–311). It is only released when `try_consume_file()` returns and the `classifier` reference goes out of scope.

**The comment at lines 288–290 of consumer.py explicitly acknowledges this design tension:**

> "TODO: I don't really like to do this here, but this way we avoid reloading the classifier multiple times, since there are multiple post-consume hooks that all require the classifier."

The developer recognized that loading the classifier once and passing it to all signal handlers is preferable to loading it multiple times. However, this means the model's full memory footprint is held for the entire duration of the signal handler chain.

**Proportionate to document size?** **No — this is the key finding.** The classifier model size is a function of the **total document corpus**, not the individual document being consumed. A 5 KB text file triggers the exact same 50–200 MB classifier load as a 50 MB PDF. This is the primary explanation for why memory appears disproportionate to document size, especially for small or text-based documents.

**Hotspot?** **Critical hotspot.** This is the dominant memory consumer for small-to-medium documents and a major contributor for all document sizes.

---

### 2.8 Document Storage (`_store()`)

**Code Location:** `src/documents/consumer.py` lines 379–412

```python
# consumer.py lines 397-402
with open(self.path, "rb") as f:
    document = Document.objects.create(
        title=(self.override_title or file_info.title)[:127],
        content=text,
        mime_type=mime_type,
        checksum=hashlib.md5(f.read()).hexdigest(),
        created=created,
        modified=created,
        storage_type=storage_type,
    )
```

**What happens:** This is the **second full-file read** for MD5 checksum computation. The exact same file that was read at line 103 in `pre_check_duplicate()` is read again here at line 402. The `f.read()` call loads the entire file into memory, passes it to `hashlib.md5()`, and the resulting hexdigest is used as the `checksum` field for the `Document` model.

**Memory allocated:** 1× file size for the `f.read()` bytes object, plus the `Document` Django model instance (which includes `content=text` — a reference to the full document text string).

**When released:** The `f.read()` temporary is released when the `with` block exits and the MD5 computation completes. The `document` model instance persists as a local variable through the rest of `try_consume_file()`.

**Hotspot?** **Yes — duplicate work.** The checksum computed at line 103–104 in `pre_check_duplicate()` is never saved or returned. If it were, this second read could be eliminated entirely, saving one full file-size memory allocation. This is a clear code-structural inefficiency.

**Rationale:** The `pre_check_duplicate()` method (lines 102–113) computes the checksum, uses it for the duplicate query, and then discards it. The `_store()` method recomputes the identical checksum. The fix would be to return the checksum from `pre_check_duplicate()` and pass it to `_store()`, but this analysis does not modify code.

---

### 2.9 Signal Handler Chain (`document_consumption_finished`)

**Code Location:** `src/documents/consumer.py` lines 306–311, with handlers connected in `src/documents/apps.py` lines 22–27

```python
# consumer.py lines 306-311
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,
)
```

The signal fires **inside** the `transaction.atomic()` block (line 298), meaning all signal handler operations occur within the same database transaction.

**Six handlers fire in this order** (as connected in apps.py lines 22–27):

#### Handler 1: `add_inbox_tags` (handlers.py lines 30–32)

```python
inbox_tags = Tag.objects.filter(is_inbox_tag=True)
document.tags.add(*inbox_tags)
```

**Memory:** Lightweight — a filtered QuerySet typically returning 0–5 Tag instances. ~1 KB.

#### Handler 2: `set_correspondent` (handlers.py lines 35–98)

Calls `matching.match_correspondents(document, classifier)` (matching.py lines 21–31):

1. **Classifier prediction** (matching.py line 23): `classifier.predict_correspondent(document.content)` calls `classifier.py` line 253: `self.data_vectorizer.transform([preprocess_content(content)])`. This:
   - Calls `preprocess_content(content)` (classifier.py lines 24–27): `content.lower().strip()` creates a new lowercase string copy, then `re.sub(r"\s+", " ", content)` creates **another** new string with whitespace normalized. **Two new full-text-size string allocations.**
   - Calls `self.data_vectorizer.transform([...])` which creates a **new sparse CSR matrix** representing the document's term frequencies against the vocabulary. Size depends on vocabulary size — typically 1–10 MB for the matrix structure.
   - The `correspondent_classifier.predict(X)` call processes the sparse matrix through the neural network's forward pass, creating temporary NumPy arrays for each hidden layer.

2. **Load all correspondents** (matching.py line 27): `Correspondent.objects.all()` evaluates a full table scan, loading **every** Correspondent record as a Django model instance into memory. Each instance is ~500 bytes–1 KB including Python object overhead.

3. **Match evaluation** (matching.py lines 29–31): `filter(lambda o: matches(o, document) or o.pk == pred_id, correspondents)` iterates all correspondents, calling `matches()` for each. For correspondents using `MATCH_FUZZY` (matching.py lines 127–145):
   - Line 130: `match = re.sub(r"[^\w\s]", "", matching_model.match)` — strips punctuation from the model's match string (small).
   - Line 131: `text = re.sub(r"[^\w\s]", "", document_content)` — **creates a full copy** of the entire document content with all punctuation removed. This is a new string allocation proportional to document text size, and it is created **for every fuzzy-matching correspondent**.

**Memory for this handler:** preprocessed content (~2× text size) + sparse matrix (~1–10 MB) + all Correspondent instances + N × fuzzy text copies.

#### Handler 3: `set_document_type` (handlers.py lines 101–165)

**Identical pattern** to `set_correspondent`: calls `matching.match_document_types(document, classifier)` (matching.py lines 34–44):

1. `classifier.predict_document_type(document.content)` → another `preprocess_content()` call (**2 more** string copies) + another `data_vectorizer.transform()` call (**another** sparse matrix).
2. `DocumentType.objects.all()` — loads **all** DocumentType records.
3. Fuzzy matching creates additional text copies per fuzzy-matching document type.

#### Handler 4: `set_tags` (handlers.py lines 168–230)

**Same pattern again**: calls `matching.match_tags(document, classifier)` (matching.py lines 47–57):

1. `classifier.predict_tags(document.content)` → yet another `preprocess_content()` + `data_vectorizer.transform()` (**third** sparse matrix + **third** pair of string copies).
2. `Tag.objects.all()` — loads **all** Tag records.
3. Fuzzy matching creates additional text copies per fuzzy-matching tag.

#### Handler 5: `set_log_entry` (handlers.py lines 413–425)

```python
ct = ContentType.objects.get(model="document")
user = User.objects.get(username="consumer")
LogEntry.objects.create(...)
```

**Memory:** Lightweight — two single-object queries + one INSERT. ~2 KB.

#### Handler 6: `add_to_index` (handlers.py lines 428–431)

```python
from documents import index
index.add_or_update_document(document)
```

Calls `index.py` lines 118–120: `open_index_writer()` creates an `AsyncWriter` wrapping a Whoosh index, then `update_document(writer, document)` (lines 87–107) serializes document metadata (title, content, tags, correspondent, type) into the index.

**Memory:** The `AsyncWriter` holds modified index segments in memory until commit. For a single document update, this is typically 1–5 MB depending on content length.

#### Cumulative Signal Handler Memory Impact

For **one document consumption**, the signal handler chain creates:

| Allocation | Count | Size Each | Total |
|------------|-------|-----------|-------|
| `preprocess_content()` string copies | 3 calls × 2 strings each = 6 | ~text size | 6× text size |
| Sparse matrices from `data_vectorizer.transform()` | 3 | 1–10 MB | 3–30 MB |
| `Correspondent.objects.all()` instances | 1 query | All correspondents | N_corr × ~1 KB |
| `DocumentType.objects.all()` instances | 1 query | All types | N_type × ~1 KB |
| `Tag.objects.all()` instances | 1 query | All tags | N_tag × ~1 KB |
| Fuzzy match text copies | N_fuzzy models | ~text size each | N_fuzzy × text size |
| Whoosh index writer | 1 | 1–5 MB | 1–5 MB |

**Rationale:** These allocations are **not all simultaneously live** — each handler's QuerySet and sparse matrix can be GC'd after the handler returns. However, within a handler, all allocations coexist. The preprocessed content strings and sparse matrices from earlier handlers may or may not be collected before later handlers run, depending on CPython's reference counting.

---

### 2.10 File Write Operations

**Code Location:** `src/documents/consumer.py` lines 315–342, using the `_write()` method at lines 429–432

```python
# consumer.py lines 429-432
def _write(self, storage_type, source, target):
    with open(source, "rb") as read_file:
        with open(target, "wb") as write_file:
            write_file.write(read_file.read())
```

**What happens:** The `_write()` method reads the **entire source file** into memory via `read_file.read()` and then writes it to the target in a single call. This is used for:

- **Original document** (line 319): Reads the full original file.
- **Thumbnail** (lines 321–325): Reads the thumbnail PNG (typically 50–200 KB).
- **Archive file** (lines 333–336): Reads the full archive PDF (often equal to or larger than the original).

After writing the archive file, **another full-file read** occurs for archive checksum computation:

```python
# consumer.py lines 339-342
with open(archive_path, "rb") as f:
    document.archive_checksum = hashlib.md5(
        f.read(),
    ).hexdigest()
```

**Memory allocated:** For each `_write()` call, 1× the file size is held in memory during the write. For the archive checksum, another 1× archive file size. These are sequential, not simultaneous (previous read is released before the next begins), but each individually can be very large.

**Hotspot?** **Yes.** The `_write()` method performs non-streaming file copy. Python's standard library provides `shutil.copyfileobj(read_file, write_file)` which copies in chunks (default 16 KB), or `shutil.copy2()` for full metadata preservation. The current approach holds the entire file in memory unnecessarily.

**Rationale:** For a 50 MB original + 50 MB archive:
- `_write()` for original: 50 MB (released after write)
- `_write()` for archive: 50 MB (released after write)
- Archive checksum `f.read()`: 50 MB (released after MD5)

Total peak for file writes: 50 MB (sequential, not cumulative). But each allocation is entirely avoidable with streaming APIs.

---

### 2.11 Parser Cleanup

**Code Location:** `src/documents/consumer.py` line 369, calling `src/documents/parsers.py` lines 348–350

```python
# consumer.py line 369
document_parser.cleanup()

# parsers.py lines 348-350
def cleanup(self):
    self.log("debug", f"Deleting directory {self.tempdir}")
    shutil.rmtree(self.tempdir)
```

**What happens:** Removes the parser's temporary directory and all files within it from disk.

**What does NOT happen:** The `cleanup()` method does **not** explicitly clear `self.text`, `self.archive_path`, or other in-memory attributes. These attributes remain on the parser instance as Python object references.

**When released:** The `document_parser` variable is a local in `try_consume_file()`. After `cleanup()` at line 369, the parser instance still holds its attributes but the function continues with `run_post_consume_script(document)` at line 371, progress notification at line 375, and returns at line 377. The parser instance (and its `self.text`) is finally eligible for GC when the function returns and all local variable references go out of scope.

**Hotspot?** **Moderate.** The parser's `self.text` holds the full document text from line 261 through line 377. Meanwhile, the same text is also referenced by the local `text` variable (line 271) and stored in the `Document` model's `content` field (line 400 in `_store()`). After `_store()`, the text exists in three places simultaneously: `document_parser.text`, the local `text`, and `document.content`. Python strings are immutable and reference-shared, so these are three references to the same object — not three copies. But the shared reference prevents GC until all three references are gone.

---

## 3. Memory Hotspot Summary Table

| # | Component / Method | File (Lines) | Memory Impact | Proportionate to Doc Size? | Retained Until |
|---|-------------------|--------------|---------------|---------------------------|----------------|
| 1 | `pre_check_duplicate()` `f.read()` | `consumer.py` (103–104) | 1× file size | Yes | `with` block exit |
| 2 | `_store()` `f.read()` for checksum | `consumer.py` (397–402) | 1× file size | Yes | `with` block exit |
| 3 | `_write()` `read_file.read()` | `consumer.py` (429–432) | 1× file size per call | Yes | `with` block exit |
| 4 | Archive checksum `f.read()` | `consumer.py` (339–342) | 1× archive file size | Yes | `with` block exit |
| 5 | **`load_classifier()` pickle load** | `classifier.py` (76–94) | **50–200+ MB** (corpus-dependent) | **No** | `try_consume_file()` return (line 377) |
| 6 | `data_vectorizer.transform()` ×3 | `classifier.py` (253, 264, 277) | Sparse matrix per call (~1–10 MB) | No | Signal handler scope exit |
| 7 | `preprocess_content()` ×3 | `classifier.py` (24–27) | Full text copy ×6 (2 per call) | Yes | Signal handler scope exit |
| 8 | Fuzzy match `re.sub()` | `matching.py` (130–131) | Full text copy per fuzzy model | Yes | `matches()` return |
| 9 | `.objects.all()` ×3 | `matching.py` (27, 40, 53) | All model instances in table | No | Signal handler scope exit |
| 10 | Parser `self.text` | Various parsers | Full extracted text | Yes | Parser cleanup / scope exit |
| 11 | `pdfminer_extract_text()` | `tesseract/parsers.py` (120) | Full text extraction | Yes | `extract_text()` return |
| 12 | Tika `response.content` | `tika/parsers.py` (96) | Full converted PDF bytes | Yes | `convert_to_pdf()` scope exit |
| 13 | Image alpha processing | `tesseract/parsers.py` (197–201) | 2× full uncompressed image | Yes | `with` block exit |
| 14 | `dateparser` module load | `parsers.py` (221) | ~10–20 MB (one-time per process) | No | Process lifetime |

---

## 4. Metadata Handling — Object Copies and Reference Retention

**Question:** "Does metadata handling create unnecessary object copies or hold references longer than needed?"

**Answer: Yes, on both counts.** The metadata-handling stages — specifically the classifier prediction and rule-based matching during the signal handler chain — create multiple unnecessary copies of document content and hold references to large objects longer than structurally necessary.

### Finding 1: `preprocess_content()` Creates Redundant String Copies

**Location:** `src/documents/classifier.py` lines 24–27

```python
def preprocess_content(content):
    content = content.lower().strip()
    content = re.sub(r"\s+", " ", content)
    return content
```

This function is called inside **each** of the three `predict_*()` methods:
- `predict_correspondent()` at line 253
- `predict_document_type()` at line 264
- `predict_tags()` at line 277

Each call to `preprocess_content()` creates **two new string objects**: one from `.lower().strip()` and another from `re.sub()`. Since Python strings are immutable, each operation creates a new allocation. Across three prediction calls, this produces **6 new string objects**, each approximately equal to the full document content size.

**The preprocessed result is NOT cached between calls.** Each `predict_*()` method independently calls `preprocess_content(content)` with the same `content` argument. The identical preprocessing work is performed three times. If the preprocessed content were computed once and reused, this would eliminate 4 of the 6 string allocations.

**Rationale:** This is a clear inefficiency in the code structure. The `predict_*()` methods are called with the same `document.content` value (from matching.py lines 23, 36, 49), and the preprocessing is deterministic. Caching the result would be trivial but is not implemented.

### Finding 2: Matching Loads All Model Instances Unnecessarily

**Location:** `src/documents/matching.py` lines 27, 40, 53

```python
# Line 27 (in match_correspondents)
correspondents = Correspondent.objects.all()
# Line 40 (in match_document_types)
document_types = DocumentType.objects.all()
# Line 53 (in match_tags)
tags = Tag.objects.all()
```

These are **three separate and independent** Django QuerySet evaluations. Each one executes a `SELECT * FROM ...` query and instantiates Python model objects for **every row** in the respective table. The results are **not shared** between the three functions — each function loads its own complete set of model instances.

If the system has 500 correspondents, 50 document types, and 200 tags, this loads 750 ORM instances into memory during signal handler execution. Each Django model instance consumes approximately 500 bytes–1 KB of Python heap (object header + field values + descriptor overhead), totaling approximately 375 KB–750 KB for 750 instances.

**Furthermore:** The QuerySets load **all** records regardless of matching algorithm. Records with `matching_algorithm = MATCH_AUTO` are checked against the classifier prediction (via `o.pk == pred_id`), while records with other algorithms go through `matches()`. Records with `match.strip() == ""` are immediately skipped at matching.py line 66–67. A filtered query (e.g., excluding empty-match records) could reduce the loaded set.

### Finding 3: Fuzzy Matching Creates Full Document Text Copies Per Model

**Location:** `src/documents/matching.py` lines 130–131

```python
match = re.sub(r"[^\w\s]", "", matching_model.match)
text = re.sub(r"[^\w\s]", "", document_content)
```

For every `MatchingModel` instance that uses `MATCH_FUZZY` (algorithm value 5), the `matches()` function (lines 60–152) creates a **full copy** of the document content with all punctuation characters removed (line 131). This `text` variable is a new string allocation approximately equal to the document content size.

If 10 correspondents use fuzzy matching, 10 separate copies of the stripped document content are created during the `match_correspondents()` call. Then the pattern repeats for `match_document_types()` and `match_tags()`.

**Rationale:** The `document_content` variable (line 63: `document_content = document.content`) is the original text. The `re.sub()` at line 131 creates a new string each time because Python strings are immutable. This could be optimized by performing the strip once per matching function call and reusing the result, but the current code strips inside the per-model `matches()` function.

### Finding 4: Signal Handlers Hold References Within `transaction.atomic()` Scope

**Location:** `src/documents/consumer.py` lines 298–346

The signal handler chain fires at lines 306–311, **inside** the `transaction.atomic()` block that starts at line 298. All database operations performed by signal handlers (QuerySet evaluations, `document.save()` calls in handlers.py lines 98, 165) are part of this transaction.

While the Django transaction mechanism itself doesn't hold Python objects in memory, the scope structure means that all signal handler local variables (QuerySet results, sparse matrices, string copies) remain reachable from the call stack until each handler returns. Since handlers fire sequentially, objects from earlier handlers may be eligible for GC before later handlers run — but Python's GC is not guaranteed to collect between handler invocations.

### Finding 5: Parser Holds `self.text` Until Well After It's Needed

**Location:** Various parsers → `src/documents/consumer.py` lines 261–377

The document parser stores the full text in `self.text` (e.g., `src/paperless_text/parsers.py` line 42). This reference is maintained through:

1. `text = document_parser.get_text()` at consumer.py line 271 — creates a **second reference** to the same string.
2. `self._store(text=text, ...)` at line 301 — the `text` local is passed to `_store()`, where it becomes `Document.objects.create(content=text, ...)` at line 400 — a **third reference** (now stored in the database model instance).
3. `document_parser.cleanup()` at line 369 — deletes temp directory but does NOT set `self.text = None`.

After `_store()`, the text exists as three live references: `document_parser.text`, the local `text`, and `document.content`. Since they all point to the same immutable string object, this doesn't triple the memory — but it does prevent the string from being collected until all three references are released (when `try_consume_file()` returns at line 377).

Setting `self.text = None` in `cleanup()` would drop one reference earlier, though it would only help GC if the other references were also released before the function exit.

---

## 5. Caching Behavior Investigation

**Question:** "Is caching accumulating data unexpectedly across processing stages?"

**Answer: No significant problematic caching is occurring in the import pipeline.** The investigation identified five potential caching sites and determined that none cause unbounded memory growth during document consumption.

### Investigation 1: `DelayedQuery.saved_results` in index.py

**Location:** `src/documents/index.py` line 196

```python
self.saved_results = dict()
```

The `DelayedQuery` class (lines 128–237) stores search result pages in a dictionary keyed by start offset. There is **no eviction policy**, no maximum size, and no TTL.

**However, this is not relevant to the import pipeline.** `DelayedQuery` objects are created by API view code for search requests — not during document consumption. The `add_to_index` signal handler (handlers.py lines 428–431) calls `index.add_or_update_document(document)`, which uses `open_index_writer()` (index.py lines 64–74) — a completely different code path that creates an `AsyncWriter`, not a `DelayedQuery`.

**Verdict: Not a problem for imports.** `DelayedQuery` caching is request-scoped (instantiated per-request, GC'd when the HTTP response completes) and is never used in the consumption pipeline.

### Investigation 2: `document_consumer_declaration.send(None)` in parsers.py

**Location:** `src/documents/parsers.py` lines 48, 71, 87

This Django signal fires to discover registered parsers. It's called in three functions:
- `get_default_file_extension()` (line 48)
- `get_supported_file_extensions()` (line 71)
- `get_parser_class_for_mime_type()` (line 87)

During document consumption, only `get_parser_class_for_mime_type()` is called (at consumer.py line 223). The signal dispatch itself is stateless — it calls each connected receiver and collects responses. The responses are small dictionaries (MIME types + parser class references). No caching occurs; each call rediscovers parsers fresh.

**Verdict: Negligible memory impact.** The signal dispatch is lightweight and stateless. Multiple calls to `get_supported_file_extensions()` or `get_default_file_extension()` elsewhere (e.g., in API views) would redundantly rediscover parsers, but the overhead is minimal.

### Investigation 3: Classifier Model File

**Location:** `src/documents/consumer.py` line 292, `src/documents/classifier.py` lines 30–57

The classifier pickle is loaded from disk on **every** `consume_file` task via `load_classifier()`. There is **no module-level or class-level cache** of the deserialized classifier object. Each call to `load_classifier()` creates a fresh `DocumentClassifier` instance and deserializes the pickle from scratch.

Combined with `Q_CLUSTER["recycle"] = 1` (`src/paperless/settings.py` line 452), which forces the django-q worker process to exit after every single task, there is zero opportunity for the classifier to persist across document consumptions — even if caching were implemented.

**Verdict: No problematic caching exists.** The classifier is loaded fresh per-task by design (and by infrastructure — worker recycling prevents any cross-task state). This is **not** a caching problem but a **per-task cost** problem: the full pickle deserialization (~50–200 MB) occurs for every document regardless of whether the model has changed.

### Investigation 4: `dateparser` Module

**Location:** `src/documents/parsers.py` line 221

```python
import dateparser
```

Python's `sys.modules` dict caches imported modules for the process lifetime. The first `import dateparser` loads the module (including its extensive locale data), and subsequent imports are no-ops that return the cached reference.

Since `Q_CLUSTER["recycle"] = 1` forces worker process recycling per task, the `dateparser` module is reloaded from scratch for every document consumption. The module stays loaded from the first `import` until the worker process exits.

**Verdict: Acceptable behavior.** Module caching via `sys.modules` is fundamental Python behavior and not problematic. The module load adds ~10–20 MB to the process, but this is a one-time per-process cost that persists for a single task before the worker recycles.

### Investigation 5: Django QuerySet Evaluation

**Location:** `src/documents/matching.py` lines 27, 40, 53

Django QuerySets are lazy — they don't execute SQL until iterated. In matching.py, each `.objects.all()` QuerySet is iterated by the `filter(lambda ...)` call, which forces full evaluation. The evaluated results are **not cached** between the three matching functions — each function evaluates its own independent QuerySet.

Django's built-in QuerySet caching stores results **within a single QuerySet object** (so iterating the same QS twice reuses the cached results), but three separate `.objects.all()` calls create three separate QuerySet objects with independent caches.

**Verdict: No problematic cross-call caching.** Each matching function pays its own full query cost independently. This is redundant (the data could be queried once and shared), but it does not create growing/unbounded caches.

---

## 6. Ingestion Pathway Comparison

Paperless-ngx supports three document ingestion pathways. All three ultimately dispatch to the same `Consumer.try_consume_file()` pipeline analyzed above, but they differ in how the file arrives and how the consumption task is initiated.

### 6.1 Filesystem Watcher

**Entry Point:** `src/documents/management/commands/document_consumer.py`

**Flow:**
1. Watchdog (or inotify) detects a new file in `CONSUMPTION_DIR`
2. `Handler.on_created()` (line 129) spawns a `Thread` calling `_consume_wait_unmodified()` (lines 99–125)
3. After the file stabilizes (no changes for `CONSUMER_POLLING_DELAY` × `CONSUMER_POLLING_RETRY_COUNT`), `_consume()` (lines 46–96) is called
4. `async_task("documents.tasks.consume_file", filepath, ...)` (line 86–91) serializes the file path and optional tag IDs to Redis
5. A django-q worker process picks up the task and executes `consume_file()` from `tasks.py` (lines 184–250)
6. `Consumer().try_consume_file(path, ...)` is called at tasks.py line 236

**Memory characteristics:**
- The filesystem watcher process itself is lightweight — it holds file path strings and watchdog/inotify state.
- Each spawned `Thread` for `_consume_wait_unmodified()` holds minimal state (file path, stat results).
- The actual consumption runs in a **separate django-q worker process** with its own memory space.
- `Q_CLUSTER["recycle"] = 1` ensures the worker process exits after each task, releasing **all** memory.

**Memory impact:** Equivalent to single-document consumption in an isolated process. No cross-document memory accumulation.

### 6.2 REST API Upload

**Entry Point:** `src/documents/views.py` `PostDocumentView` (lines 491–535)

**Flow:**
1. HTTP POST multipart request arrives with the uploaded file
2. DRF `MultiPartParser` parses the request body, placing the file data in `serializer.validated_data`
3. Line 502: `doc_name, doc_data = serializer.validated_data.get("document")` — `doc_data` holds the **entire uploaded file** as a Python `bytes` object in the API request handler's memory
4. Lines 512–519: `f.write(doc_data)` writes the bytes to a `tempfile.NamedTemporaryFile` in `SCRATCH_DIR`
5. Lines 523–533: `async_task("documents.tasks.consume_file", temp_filename, ...)` queues the consumption task via Redis

**Memory characteristics:**
- **The uploaded file exists in memory TWICE** briefly: once in `doc_data` (validated serializer data) and once being written to the temp file. The DRF MultiPartParser may hold additional copies in its parsing buffers.
- After the `with` block exits (line 519), `doc_data` remains live until the `post()` method returns at line 535 (when the HTTP response is sent). Python's GC frees it after the response.
- The actual consumption runs in a separate worker process via `async_task()`.

**Memory impact:** The API handler's memory usage scales with upload file size. For a 50 MB upload, the API process temporarily holds ~50–100 MB. This is **separate from** the worker process's consumption memory. If multiple uploads arrive simultaneously, each API request handler holds its own file data.

### 6.3 Email Ingestion

**Entry Point:** `src/paperless_mail/mail.py`

**Flow:**
1. Periodically checks configured IMAP accounts for new emails
2. Downloads email attachments via the imap-tools library
3. Writes attachments to temporary files
4. Queues consumption via `async_task()`, similar to the filesystem watcher

**Memory characteristics:**
- Similar to the API upload pathway: email attachments are downloaded into memory, then written to temp files.
- The IMAP fetch operation may hold the full email (including all attachments) in memory during download.
- Consumption runs in a separate worker process.

**Memory impact:** Comparable to API upload for per-attachment memory. The email processing itself adds overhead for MIME parsing and attachment extraction.

### Pathway Comparison Summary

| Pathway | File Held in Memory Before Queuing? | Consumption Process | Cross-Document Accumulation? |
|---------|-------------------------------------|--------------------|-----------------------------|
| Filesystem Watcher | No — file is on disk; only path is queued | Separate worker | No (worker recycles) |
| REST API Upload | Yes — entire file in `doc_data` bytes | Separate worker | No (worker recycles; API request completes) |
| Email Ingestion | Yes — attachment downloaded into memory | Separate worker | No (worker recycles) |

**Key insight:** The filesystem watcher pathway has the **lowest** pre-consumption memory overhead because the file already exists on disk — only the file path string is serialized to Redis. The API and email pathways hold the full file in the ingestion process's memory during the upload/download phase, but this memory is released before the actual consumption begins in the worker process.

---

## 7. Document Type Impact on Memory

The parser selected for a document determines the most variable component of memory consumption. The classifier, checksum, and signal handler costs are **identical** regardless of document type — the difference lies entirely in the parsing stage.

### 7.1 PDF Documents — RasterisedDocumentParser

**Parser:** `src/paperless_tesseract/parsers.py`
**Status:** **Highest memory consumer**

Memory profile for a typical 10 MB multi-page PDF:

| Component | Estimated Size | Duration |
|-----------|---------------|----------|
| pikepdf PDF structure | 20–50 MB | `extract_metadata()` scope |
| OCRmyPDF orchestration objects | ~5 MB | `parse()` scope |
| Tesseract subprocess memory | 50–200 MB (in child process — invisible to parent) | Subprocess lifetime |
| pdfminer text extraction | 100 KB–5 MB (text string) | Stored in `self.text` until cleanup |
| Image processing (if image input with alpha) | 2× uncompressed image (~96 MB for 4000×3000 RGBA) | `construct_ocrmypdf_parameters()` scope |
| Archive PDF (output) | ~10 MB on disk; read for checksum = 10 MB in memory | `_write()` scope |

The Tesseract parser stands out because it **opens** the PDF structure in memory (pikepdf), **processes** it through OCR (subprocess), and **extracts** text (pdfminer) — three heavyweight operations on the same document. The archive PDF it produces is then read entirely for checksum computation and non-streaming file copy.

### 7.2 Plain Text Documents — TextDocumentParser

**Parser:** `src/paperless_text/parsers.py`
**Status:** **Lowest memory consumer**

Memory profile for a typical 5 KB text file:

| Component | Estimated Size | Duration |
|-----------|---------------|----------|
| `self.text = f.read()` | ~5 KB | Stored in `self.text` until cleanup |
| Thumbnail PIL Image (500×700 RGB) | ~1 MB | `get_thumbnail()` scope |

**However:** The classifier loading (50–200 MB) is the **same** regardless of document size. For a 5 KB text file:
- Parser memory: ~1 MB
- Classifier memory: 50–200 MB
- File reads (duplicate check + store checksum): ~10 KB
- **Classifier dominates at >98% of total memory**

This is the primary explanation for the user's observation that "memory consumption appears out of proportion to actual document sizes" for text-based documents with small metadata.

### 7.3 Office Documents — TikaDocumentParser

**Parser:** `src/paperless_tika/parsers.py`
**Status:** **Network-dependent, moderate memory**

Memory profile for a typical 5 MB Office document:

| Component | Estimated Size | Duration |
|-----------|---------------|----------|
| Tika HTTP response (`parsed` dict) | 2–5 MB | `parse()` scope |
| Gotenberg PDF response (`response.content`) | 5–15 MB | `convert_to_pdf()` scope |
| Extracted text (`self.text`) | 50–500 KB | Stored until cleanup |

The Tika parser is unique in that the heavy processing (text extraction, PDF conversion) happens on external servers. The parent process only holds the HTTP response bodies. These can be large for big Office documents but are **sequential** (Tika response is processed before Gotenberg is called, so the first can be GC'd before the second arrives).

### Document Type Comparison Matrix

| Factor | PDF (Tesseract) | Text | Office (Tika) |
|--------|-----------------|------|---------------|
| Parser-specific memory | **High** (50–250 MB) | **Low** (~1 MB) | **Moderate** (7–20 MB) |
| Classifier memory | 50–200 MB | 50–200 MB | 50–200 MB |
| File reads (checksum/copy) | High (3× file size) | Low (2× file size, no archive) | Moderate (2× file + archive) |
| Signal handler overhead | Same | Same | Same |
| Subprocess memory (child) | **Very high** (Tesseract OCR) | None | None (external servers) |
| **Total parent process peak** | **150–500+ MB** | **60–250 MB** | **80–300 MB** |

---

## 8. Batch Size Impact on Cumulative Memory

**Question:** "How does batch size affect cumulative memory during import operations?"

### Single-Task Worker Recycling

The most important configuration for understanding batch memory behavior is in `src/paperless/settings.py` line 452:

```python
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,           # <-- Worker recycled after EVERY task
    "retry": PAPERLESS_WORKER_RETRY,
    "timeout": PAPERLESS_WORKER_TIMEOUT,
    "workers": TASK_WORKERS,
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
}
```

The `"recycle": 1` setting means each django-q worker process is **terminated and replaced after every single task**. This is the most aggressive recycling setting possible.

**Consequence:** Batch size does **NOT** cause cumulative memory growth across documents. Each `consume_file` task runs in a fresh worker process. When the task completes (or fails), the worker process exits and the operating system reclaims **all** of its memory — including the classifier model, parser objects, string copies, sparse matrices, QuerySet instances, and everything else. The next task starts in a completely clean process.

### Within-Task Accumulation

However, **within a single document consumption**, memory **does** accumulate through the pipeline stages. Since `try_consume_file()` is a single function invocation, all local variables coexist simultaneously:

1. **At line 261** (parsing begins): Parser instance is live
2. **At line 271** (text extracted): Parser + `text` local variable are live
3. **At line 292** (classifier loaded): Parser + `text` + **classifier model** are live — **this is the first major spike**
4. **At line 301** (`_store` called): Parser + `text` + classifier + `Document` model instance are live
5. **At lines 306–311** (signal handlers): Parser + `text` + classifier + `Document` + handler allocations — **peak memory**
6. **At lines 319–342** (file writes): Parser + `text` + classifier + `Document` + file read buffers
7. **At line 369** (cleanup): Temp directory removed but all Python objects still live
8. **At line 377** (return): All local variables released

The peak memory for a single document occurs during the signal handler chain (steps 4–5), when the classifier model, parser text, document model, and handler-specific allocations all coexist.

### Barcode Processing Exception

When `CONSUMER_ENABLE_BARCODES` is True (`src/paperless/settings.py`), the consumption flow in `src/documents/tasks.py` lines 195–233 adds a **significant pre-processing step**:

```python
# tasks.py line 198
separators = scan_file_for_separating_barcodes(path)
```

`scan_file_for_separating_barcodes()` (tasks.py lines 96–110) calls:

```python
# tasks.py line 105
pages_from_path = convert_from_path(filepath, output_folder=path)
```

`pdf2image.convert_from_path()` renders **all pages** of the PDF as PIL image objects. For a 100-page PDF at 200 DPI, this creates 100 images of approximately 1654×2339 pixels × 3 channels (RGB) ≈ **11.6 MB per page** = **~1.16 GB total** for all page images.

The `output_folder=path` parameter directs `pdf2image` to save rendered images to a temporary directory, which helps with disk I/O. However, the returned `pages_from_path` list contains PIL Image objects or file references that are iterated for barcode detection:

```python
# tasks.py lines 106-109
for current_page_number, page in enumerate(pages_from_path):
    current_barcodes = barcode_reader(page)
```

`barcode_reader(page)` (tasks.py lines 75–93) calls `pyzbar.decode(image)` which processes each page image for barcodes.

**Memory impact:** Barcode processing can be the **single largest memory consumer** in the entire pipeline for multi-page PDFs. A 100-page PDF could require over 1 GB of image memory if all pages are rendered simultaneously. This dwarfs the classifier model and file-read overhead.

**Mitigation:** The `TemporaryDirectory()` context manager (tasks.py line 104) ensures cleanup of the temp directory when the `with` block exits (line 110), and the `pages_from_path` list is released at the same time.

---

## 9. GC Behavior Analysis — Normal vs. Problematic

**Question:** "Is the observed memory behavior normal Python/CPython garbage collection behavior, or does it indicate a genuine problem?"

**Answer: The behavior is a mixture of normal Python scoping patterns and genuine code-structural anti-patterns.** Some memory retention is inherent to Python's execution model, while other allocations are avoidable and represent real inefficiencies.

### Normal Python/CPython Behavior

#### 1. Reference Counting (Immediate Reclamation)

CPython (Python 3.9 as used in this project) uses reference counting as its primary memory management mechanism. When an object's reference count drops to zero, it is **immediately** freed — no GC cycle required.

This applies to the `f.read()` buffers in `pre_check_duplicate()` and `_store()`: when the `with` block exits and no other variable holds a reference to the bytes object, it is freed immediately. There is no delayed GC issue here.

#### 2. Scope-Based Retention (By Design)

The classifier, parser, text, and other objects persist through `try_consume_file()` because they are local variables in that function's scope. They are freed when the function returns at line 377. This is **normal Python scoping behavior** — variables exist for the lifetime of their enclosing scope. There is no GC bug here.

The developer even acknowledges this trade-off at consumer.py lines 288–290: the classifier is loaded once and shared across signal handlers intentionally, because loading it multiple times would be even more expensive.

#### 3. Cyclic GC for Complex Objects

Django model instances may have cyclic references (e.g., ForeignKey back-references, related manager objects). CPython's cyclic garbage collector handles these, but collection is deferred to GC generations (gen0 → gen1 → gen2). During the `transaction.atomic()` block (consumer.py lines 298–367), model instances created by signal handlers may form cycles that aren't collected until a GC sweep.

This is **normal Django behavior**. The cyclic GC runs periodically (default: after every 700 gen0 allocations) and typically has negligible impact. The worker recycling (`recycle=1`) ensures all cycles are collected when the process exits.

#### 4. Worker Process Recycling

`Q_CLUSTER["recycle"] = 1` is the most aggressive recycling setting. It ensures that **no memory persists across document consumptions**. This is a deliberate design choice that trades startup overhead (new process per task) for guaranteed memory isolation.

### Genuine Problems Identified

#### Problem 1: Duplicate File Reads (Avoidable)

**Severity:** Moderate
**Location:** `consumer.py` lines 103–104 and 397–402

The same file is read entirely into memory twice for MD5 checksum computation. The first computation at line 103–104 (in `pre_check_duplicate()`) produces a checksum that is used for the duplicate query and then **discarded**. The second computation at line 402 (in `_store()`) recomputes the identical value.

**Why this is a genuine problem:** The `hashlib.md5()` call is deterministic — reading the same file twice produces the same hash. The fix is trivial: return the checksum from `pre_check_duplicate()` and reuse it in `_store()`. This would eliminate one full file-size memory allocation.

Additionally, neither read uses chunked hashing (`md5.update(chunk)`), which would reduce peak memory from O(file_size) to O(chunk_size).

#### Problem 2: Non-Streaming File Copy (Avoidable)

**Severity:** Moderate
**Location:** `consumer.py` lines 429–432

```python
def _write(self, storage_type, source, target):
    with open(source, "rb") as read_file:
        with open(target, "wb") as write_file:
            write_file.write(read_file.read())
```

This reads the entire source file into memory before writing. Python's `shutil.copyfileobj(read_file, write_file)` performs chunked copying (default 16 KB buffer) with negligible memory overhead. The current approach unnecessarily holds the full file in memory during the copy.

**Why this is a genuine problem:** For a 50 MB file, this creates a 50 MB bytes object that exists only to be written immediately to another file. Streaming would use ~16 KB regardless of file size.

#### Problem 3: Triplicate Classifier Prediction (Avoidable)

**Severity:** Moderate
**Location:** `classifier.py` lines 253, 264, 277; `matching.py` lines 23, 36, 49

The same `preprocess_content(content) → data_vectorizer.transform()` sequence is executed three times with identical input — once for correspondent prediction, once for document type prediction, and once for tags prediction. Each call creates new string copies and a new sparse matrix.

**Why this is a genuine problem:** The preprocessing and vectorization are deterministic. Computing the result once and passing the sparse matrix to all three prediction methods would eliminate 4 string copies and 2 sparse matrix allocations.

#### Problem 4: Triplicate QuerySet Evaluation (Avoidable)

**Severity:** Low
**Location:** `matching.py` lines 27, 40, 53

Three separate `SELECT * FROM` queries load all records from Correspondents, DocumentTypes, and Tags tables. While these are different tables, the pattern of loading entire tables is consistent and potentially optimizable with filtered queries.

#### Problem 5: Per-Document Classifier Loading (Structural)

**Severity:** High (but mitigated by worker recycling)
**Location:** `classifier.py` lines 76–94, `consumer.py` line 292

The classifier model (50–200+ MB) is deserialized from pickle for every single document consumption. With `recycle=1`, there is no way to cache it across tasks. This is the **dominant memory allocation** for most documents and is entirely independent of document size.

**Why this is a genuine problem:** The pickle load is the single most expensive operation in the entire pipeline for small documents. However, fixing it would require either relaxing worker recycling (risking memory leaks) or implementing a shared memory classifier cache (complex architecture change).

### Verdict

| Behavior | Classification | Impact |
|----------|---------------|--------|
| Scope-based variable retention | **Normal** Python behavior | Expected |
| Worker recycling preventing accumulation | **Normal** and intentional | Positive |
| Cyclic GC for Django models | **Normal** Django behavior | Negligible |
| Duplicate file reads | **Genuine code problem** | Moderate — 1× file size per consumption |
| Non-streaming file copy | **Genuine code problem** | Moderate — 1× file size per copy |
| Triplicate prediction work | **Genuine code problem** | Moderate — 3× text preprocessing + 3× vectorization |
| Per-document classifier load | **Structural issue** | High — 50–200 MB per consumption regardless of document size |

---

## 10. Peak Memory Model

This section estimates the peak resident memory for the worker process during document consumption, at the point of maximum simultaneous allocation (during the signal handler chain, when the classifier, parser text, document model, and handler allocations all coexist).

### Baseline: Python Process Overhead

A bare django-q worker process running on CPython 3.9 with Django 4.0 and all Paperless dependencies loaded consumes approximately **50–80 MB** of resident memory before any task execution. This includes the Python interpreter, loaded modules, Django framework, and DRF.

### Scenario 1: Small Text File (5 KB)

| Component | Estimated Size |
|-----------|---------------|
| Python baseline | 50–80 MB |
| File read: duplicate check | ~5 KB |
| Parser (`self.text`) | ~5 KB |
| dateparser module load | ~10–20 MB |
| **Classifier model** | **50–200 MB** |
| File read: store checksum | ~5 KB |
| Signal handler: 3× preprocess strings | ~30 KB |
| Signal handler: 3× sparse matrices | ~3–30 MB |
| Signal handler: QuerySet instances | ~100–750 KB |
| File write (non-streaming) | ~5 KB |
| **Estimated total peak** | **~113–330 MB** |
| **Classifier as % of total** | **~44–61%** |

**Key insight:** For a 5 KB text file, the classifier model accounts for approximately **half** of the total memory usage. The file itself is negligible.

### Scenario 2: Medium PDF (5 MB)

| Component | Estimated Size |
|-----------|---------------|
| Python baseline | 50–80 MB |
| File read: duplicate check | ~5 MB |
| pikepdf PDF structure | ~10–25 MB |
| pdfminer text extraction | ~200 KB |
| **Classifier model** | **50–200 MB** |
| File read: store checksum | ~5 MB |
| Signal handler: 3× preprocess strings | ~1.2 MB |
| Signal handler: 3× sparse matrices | ~3–30 MB |
| Signal handler: QuerySet instances | ~100–750 KB |
| File write: original (non-streaming) | ~5 MB |
| File write: archive (non-streaming) | ~5 MB |
| Archive checksum read | ~5 MB |
| **Estimated total peak** | **~139–356 MB** |

### Scenario 3: Large PDF (50 MB)

| Component | Estimated Size |
|-----------|---------------|
| Python baseline | 50–80 MB |
| File read: duplicate check | ~50 MB |
| pikepdf PDF structure | ~100–250 MB |
| pdfminer text extraction | ~1–5 MB |
| **Classifier model** | **50–200 MB** |
| File read: store checksum | ~50 MB |
| Signal handler: 3× preprocess strings | ~6–30 MB |
| Signal handler: 3× sparse matrices | ~3–30 MB |
| Signal handler: QuerySet instances | ~100–750 KB |
| File write: original (non-streaming) | ~50 MB |
| File write: archive (non-streaming) | ~50 MB |
| Archive checksum read | ~50 MB |
| **Estimated total peak** | **~410–745 MB** |

**Note:** Not all of these allocations are simultaneously live. File reads are sequential and released between stages. The peak is when the classifier + parser text + signal handler allocations coexist (during the signal handler chain), plus any file write in progress.

### Scenario 4: Large PDF with Barcode Processing (50 MB, 100 pages)

| Component | Estimated Size |
|-----------|---------------|
| All of Scenario 3 | ~410–745 MB |
| Barcode: pdf2image page rendering (100 pages) | ~1,000–1,500 MB |
| Barcode: pyzbar processing overhead | ~5 MB |
| **Estimated total peak** | **~1.4–2.2 GB** |

**Note:** Barcode processing occurs **before** the main consumption pipeline (tasks.py lines 195–233). The barcode processing memory is released before `Consumer().try_consume_file()` begins. However, the worker process's peak RSS would include the barcode processing phase.

---

## 11. Recommendations for Observation

Since this analysis does not modify existing code, the following are **monitoring and observation recommendations** for continued investigation:

### 1. Track Classifier Model File Size

The classifier pickle file (`settings.MODEL_FILE`) grows as the document corpus grows. The model's vocabulary expands with new unique terms, and the neural network weights are retrained on the larger dataset. Monitor this file's size over time:

```bash
ls -la data/classification_model.pickle
```

If the file exceeds 100 MB, the per-document memory overhead for classifier loading alone exceeds 100 MB.

### 2. Monitor Worker Process RSS

Verify that worker recycling (`recycle=1`) is functioning correctly by monitoring the resident set size (RSS) of django-q worker processes:

```bash
# During consumption
ps aux | grep "python.*qcluster"
```

After each task, the worker process should exit (PID changes). If the same PID persists across multiple tasks, recycling may be misconfigured.

### 3. Use `tracemalloc` for Per-Stage Snapshots

Python's built-in `tracemalloc` module can capture memory snapshots at specific code points without modifying production code. A temporary profiling wrapper could snapshot before and after each pipeline stage.

### 4. Monitor Django ORM Query Counts

During signal handler execution, 3× `.objects.all()` queries load all matching-model instances. Monitor query counts with Django Debug Toolbar or `django.db.connection.queries`:

```python
from django.db import connection
print(len(connection.queries))
```

### 5. Correlate Peak Memory with Parser Selection

Track peak RSS per document type to quantify the difference between Tesseract, Text, and Tika parsers:

| Document Type | Expected Peak RSS | Parser |
|---------------|-------------------|--------|
| PDF | 200–500+ MB | RasterisedDocumentParser |
| Text | 100–300 MB | TextDocumentParser |
| Office | 150–350 MB | TikaDocumentParser |

---

## 12. Conclusion

This investigation traces memory usage through every stage of the Paperless-ngx document consumption pipeline in `src/documents/consumer.py`, from file detection through duplicate check, parsing, classification, metadata matching, storage, signal handling, file writes, and cleanup. The analysis is based entirely on the source code as the authoritative reference, with specific file paths and line numbers cited for every finding.

### Primary Findings

1. **Memory spikes during import are caused by four compounding factors:**
   - **Classifier model deserialization** (`classifier.py` lines 76–94): 50–200+ MB loaded from pickle per-document, regardless of document size
   - **Multiple full-file reads** (`consumer.py` lines 103, 402, 429–432, 339–342): The same file is read into memory 2–3 times with no chunking
   - **Non-streaming file copies** (`consumer.py` lines 429–432): `read_file.read()` instead of chunked `shutil.copyfileobj()`
   - **Repeated matching evaluations** (`matching.py` lines 21–57 → `classifier.py` lines 251–292): 3× identical preprocessing, vectorization, and full-table queries

2. **The disproportionate memory-to-document-size ratio** is explained by the classifier model: it is a function of the total document corpus, not the individual document. A 5 KB text file and a 50 MB PDF both trigger the same 50–200 MB classifier load.

3. **Worker recycling** (`Q_CLUSTER["recycle"] = 1` in `settings.py` line 452) **prevents cross-document accumulation** by exiting the worker process after every task. This is effective at preventing memory leaks but does **not** reduce per-document peak memory.

4. **The behavior is a combination of:**
   - **Normal Python scoping** (variables live until function returns — by design)
   - **Normal CPython reference counting** (immediate freeing when refcount hits zero — efficient)
   - **Genuine code-structural anti-patterns** (duplicate reads, non-streaming copies, redundant preprocessing — avoidable without architectural changes)
   - **A structural design constraint** (classifier loaded per-document because worker recycling prevents caching — a trade-off between memory isolation and deserialization cost)

5. **What differentiates memory-spiking imports from normal ones:**
   - **Parser type**: PDF/OCR documents (RasterisedDocumentParser) can use 3–10× more parser-specific memory than text documents
   - **Document size**: Larger files amplify the non-streaming read/copy overhead linearly
   - **Barcode processing**: PDFs with barcode scanning enabled can spike to 1+ GB due to full-page rendering
   - **Corpus size**: Larger document corpora produce larger classifier models, increasing the fixed per-document memory floor
   - **Number of matching models**: More correspondents, types, and tags increase signal handler memory from QuerySet evaluations and matching iterations

6. **No problematic caching was identified** in the import pipeline. The `DelayedQuery.saved_results` in `index.py` is request-scoped (not used during imports), the classifier is loaded fresh each time, and Django QuerySets are independently evaluated per matching function.
