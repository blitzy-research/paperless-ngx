# Paperless-ngx ML Classification Pipeline — Investigative Analysis

This document is a comprehensive investigative analysis of the Paperless-ngx machine learning classification pipeline's runtime behavior during test execution. It targets non-deterministic test failures in the document classification subsystem. All findings are grounded exclusively in the source code of the repository and cite specific file paths and line numbers. No assumptions are made; every claim traces an actual execution path in the codebase.

The investigation answers five specific questions:

1. Does the `DocumentClassifier` reuse a persisted model or retrain from scratch during a single test run?
2. How many `Document` records are created during correspondent matching tests, and is there a confidence threshold?
3. What code path is exercised when a document has no extractable text?
4. How many `Document` records result from barcode splitting, and how does this affect training data?
5. What are the root causes of non-deterministic test failures in the ML pipeline?

---

## 1. Classifier Reuse vs. Retraining

### Question

Does the `DocumentClassifier` reuse a persisted model (loaded from `settings.MODEL_FILE`) or retrain from scratch during a single test run? What is the exact mechanism that governs that decision?

### Answer

The classifier's behavior depends on two factors: **(a)** whether a persisted model file exists on disk, and **(b)** whether the SHA-1 hash of the current training data matches the hash stored in the classifier instance. During tests, the model is always freshly trained on its first invocation because each test class receives an isolated temporary directory with no pre-existing model file. Within the same classifier instance, subsequent `train()` calls skip retraining if the training data hasn't changed.

#### 1.1 The `train_classifier()` Task Entry Point

The task entry point is defined at `src/documents/tasks.py`, line 48:

```python
def train_classifier():
```

- **Lines 49–55**: The function first checks whether any `Tag`, `DocumentType`, or `Correspondent` has `matching_algorithm == MATCH_AUTO`. If none exist, it returns early without any training:

  ```python
  if (
      not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
      and not DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
      and not Correspondent.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
  ):
      return
  ```

- **Line 57**: Calls `load_classifier()` to attempt loading a previously persisted model from disk:

  ```python
  classifier = load_classifier()
  ```

- **Lines 59–60**: If no persisted model was found (i.e., `load_classifier()` returned `None`), a fresh `DocumentClassifier()` is created:

  ```python
  if not classifier:
      classifier = DocumentClassifier()
  ```

- **Line 63**: Calls `classifier.train()`, which returns `True` if retraining occurred or `False` if training data was unchanged:

  ```python
  if classifier.train():
  ```

- **Line 67**: If training occurred, the classifier is persisted to disk via `classifier.save()`:

  ```python
  classifier.save()
  ```

#### 1.2 The `load_classifier()` Function

Defined at `src/documents/classifier.py`, line 30:

```python
def load_classifier():
```

- **Line 31**: Checks whether the model file exists on disk. If not, returns `None` immediately:

  ```python
  if not os.path.isfile(settings.MODEL_FILE):
  ```

  The default `MODEL_FILE` path is set at `src/paperless/settings.py`, line 74:

  ```python
  MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")
  ```

- **Line 38**: If the file exists, creates a `DocumentClassifier()` instance and calls `load()`:

  ```python
  classifier = DocumentClassifier()
  try:
      classifier.load()
  ```

- **Lines 42–55**: Handles four exception categories:
  - `ClassifierModelCorruptError` and `IncompatibleClassifierVersionError` (line 42): Deletes the corrupt model file at line 48 (`os.unlink(settings.MODEL_FILE)`) and returns `None`
  - `OSError` (line 50): Logs and returns `None`
  - Generic `Exception` (line 53): Logs and returns `None`

#### 1.3 The `train()` Method — SHA-1 Data Hash Comparison

Defined at `src/documents/classifier.py`, line 115:

```python
def train(self):
```

**Step 1 — Gather and hash training data:**

- **Line 124**: Initializes a SHA-1 hash object:

  ```python
  m = hashlib.sha1()
  ```

- **Lines 125–127**: Queries all non-inbox documents ordered by primary key:

  ```python
  for doc in Document.objects.order_by("pk").exclude(
      tags__is_inbox_tag=True,
  ):
  ```

  This query excludes any document tagged with an inbox tag (`is_inbox_tag=True`), which directly affects the effective training dataset size (see Section 2).

- **Line 129**: Hashes preprocessed document content:

  ```python
  m.update(preprocessed_content.encode("utf-8"))
  ```

- **Lines 132–137**: For each document, extracts the document type label. If the document type has `matching_algorithm == MATCH_AUTO`, uses its primary key; otherwise uses `-1`. Hashes the label at line 136:

  ```python
  m.update(y.to_bytes(4, "little", signed=True))
  ```

- **Lines 139–144**: Same process for the correspondent label. Hashes at line 143.

- **Lines 146–156**: Extracts and hashes all `MATCH_AUTO` tag primary keys for each document, sorted. Each tag pk is hashed at lines 154–155.

**Step 2 — Compare hashes:**

- **Line 161**: Finalizes the hash:

  ```python
  new_data_hash = m.digest()
  ```

- **Line 163**: **KEY MECHANISM** — If the classifier already has a stored `data_hash` and it matches the new hash, training is skipped entirely:

  ```python
  if self.data_hash and new_data_hash == self.data_hash:
      return False
  ```

  This is the sole mechanism governing reuse vs. retraining. When `return False` is reached, no new `MLPClassifier` instances are created and no `fit()` calls are made.

**Step 3 — Train (if data changed):**

- **Lines 188–190**: Lazy imports of scikit-learn components:

  ```python
  from sklearn.feature_extraction.text import CountVectorizer
  from sklearn.neural_network import MLPClassifier
  from sklearn.preprocessing import MultiLabelBinarizer, LabelBinarizer
  ```

- **Lines 194–198**: Creates the `CountVectorizer`:

  ```python
  self.data_vectorizer = CountVectorizer(
      analyzer="word",
      ngram_range=(1, 2),
      min_df=0.01,
  )
  ```

- **Line 219**: Creates the tags classifier — **no `random_state` parameter**:

  ```python
  self.tags_classifier = MLPClassifier(tol=0.01)
  ```

- **Line 227**: Creates the correspondent classifier — **no `random_state` parameter**:

  ```python
  self.correspondent_classifier = MLPClassifier(tol=0.01)
  ```

- **Line 238**: Creates the document type classifier — **no `random_state` parameter**:

  ```python
  self.document_type_classifier = MLPClassifier(tol=0.01)
  ```

- **Line 247**: Stores the new data hash for future comparisons:

  ```python
  self.data_hash = new_data_hash
  ```

- **Line 249**: Returns `True` indicating retraining occurred:

  ```python
  return True
  ```

#### 1.4 Behavior in Tests

- **File**: `src/documents/tests/utils.py`, function `setup_directories()` at line 14.
- **Line 45**: Sets `MODEL_FILE` to an isolated temporary path:

  ```python
  MODEL_FILE=os.path.join(dirs.data_dir, "classification_model.pickle"),
  ```

  Since `dirs.data_dir` is created via `tempfile.mkdtemp()` (line 18), it is a fresh empty directory. Therefore, `os.path.isfile(settings.MODEL_FILE)` returns `False` on the first access, and `load_classifier()` returns `None`.

- **File**: `src/documents/tests/utils.py`, line 72: The `DirectoriesMixin` class calls `setup_directories()` in its `setUp()` method (line 78), ensuring every test class using this mixin gets isolated filesystem state.

- **File**: `src/documents/tests/test_classifier.py`, method `testDatasetHashing()` at line 137:

  ```python
  def testDatasetHashing(self):
      self.generate_test_data()
      self.assertTrue(self.classifier.train())   # line 141: returns True (trained)
      self.assertFalse(self.classifier.train())  # line 142: returns False (hash match)
  ```

  This test explicitly demonstrates the reuse mechanism: the first `train()` call returns `True` (training occurred), and the second call returns `False` (data hash matched, training skipped).

#### 1.5 Conclusion

During a single test run, the classifier is **always freshly trained** on its first invocation because `DirectoriesMixin` creates an isolated temp directory with no pre-existing model file. Within the same classifier instance, subsequent `train()` calls return `False` (no retraining) if the training data hasn't changed, thanks to the SHA-1 `data_hash` comparison at `src/documents/classifier.py`, line 163. The model **is not reused across different test classes** because each test class gets its own isolated `MODEL_FILE` path via `DirectoriesMixin`. The model **is not reused across different classifier instances** because the `data_hash` field is initialized to `None` in the constructor (line 68), meaning a freshly constructed `DocumentClassifier()` will always train on first invocation.

---

## 2. Training Document Creation and Correspondent Matching

### Question

How many `Document` records are created when a test exercises automatic correspondent matching (via `MatchingModel.MATCH_AUTO = 6`)? When is `train()` invoked relative to those database inserts? Is there a confidence threshold in the prediction path?

### Answer

The standard test data creates **exactly 3 `Document` records**, but only **2 are used as effective training data** because one is tagged with an inbox tag and excluded by the training query. Training always occurs **after** all document inserts. There is **no confidence threshold** anywhere in the prediction or matching pipeline.

#### 2.1 Test Data Creation

Defined at `src/documents/tests/test_classifier.py`, method `generate_test_data()` at line 25.

**Correspondents (lines 26–34):**

| Variable | Name | `matching_algorithm` | Line |
|---|---|---|---|
| `self.c1` | "c1" | `Correspondent.MATCH_AUTO` (6) | 26–29 |
| `self.c2` | "c2" | Default `MATCH_ANY` (1) per `src/documents/models.py` line 44 | 30 |
| `self.c3` | "c3" | `Correspondent.MATCH_AUTO` (6) | 31–34 |

**Tags (lines 35–50):**

| Variable | Name | `matching_algorithm` | pk | `is_inbox_tag` | Line |
|---|---|---|---|---|---|
| `self.t1` | "t1" | `Tag.MATCH_AUTO` (6) | 12 | `False` (default) | 35–39 |
| `self.t2` | "t2" | `Tag.MATCH_ANY` (1) | 34 | **`True`** | 40–45 |
| `self.t3` | "t3" | `Tag.MATCH_AUTO` (6) | 45 | `False` (default) | 46–50 |

**Document Types (lines 51–58):**

| Variable | Name | `matching_algorithm` | Line |
|---|---|---|---|
| `self.dt` | "dt" | `DocumentType.MATCH_AUTO` (6) | 51–54 |
| `self.dt2` | "dt2" | `DocumentType.MATCH_AUTO` (6) | 55–58 |

**Documents (lines 60–82):**

| Variable | Title | Content | Correspondent | Document Type | Checksum | Tags | Line |
|---|---|---|---|---|---|---|---|
| `self.doc1` | "doc1" | "this is a document from c1" | `c1` | `dt` | "A" | `t1` | 60–66, 79 |
| `self.doc2` | "doc1" | "this is another document, but from c2" | `c2` | None | "B" | `t1`, `t3` | 67–72, 80–81 |
| `self.doc_inbox` | "doc235" | "aa" | None | None | "C" | `t2` (inbox) | 73–77, 82 |

**Total `Document` records created: 3**

#### 2.2 Effective Training Data

The training query at `src/documents/classifier.py`, lines 125–127 is:

```python
for doc in Document.objects.order_by("pk").exclude(
    tags__is_inbox_tag=True,
):
```

- `doc_inbox` is tagged with `t2`, which has `is_inbox_tag=True` (test_classifier.py line 44). Therefore, `doc_inbox` is **excluded** from the training data.
- **Effective training documents: 2** (`doc1` and `doc2`)

**Training labels derived from the 2 effective documents:**

| Document | `labels_correspondent` | `labels_document_type` | `labels_tags` |
|---|---|---|---|
| `doc1` | `c1.pk` (c1 has MATCH_AUTO) | `dt.pk` (dt has MATCH_AUTO) | `[t1.pk]` (t1 has MATCH_AUTO) |
| `doc2` | `-1` (c2 does NOT have MATCH_AUTO) | `-1` (no document type) | `[t1.pk, t3.pk]` (both have MATCH_AUTO) |

**Timing**: Training always happens **after** all `Document.objects.create()` calls and `tags.add()` calls in `generate_test_data()`. The `train()` method is invoked explicitly in the test method body — for example, at `src/documents/tests/test_classifier.py`, line 105:

```python
self.classifier.train()
```

This call occurs in `testTrain()` (line 103), which first calls `self.generate_test_data()` at line 104, meaning all 3 documents and all tag assignments are committed to the database before `train()` executes.

#### 2.3 Absence of Confidence Threshold — CRITICAL

**There is NO confidence threshold anywhere in the prediction or matching pipeline.** This is traced through the complete chain:

**Step 1 — Classifier Prediction** (`src/documents/classifier.py`, line 251):

```python
def predict_correspondent(self, content):
    if self.correspondent_classifier:
        X = self.data_vectorizer.transform([preprocess_content(content)])  # line 253
        correspondent_id = self.correspondent_classifier.predict(X)        # line 254
        if correspondent_id != -1:                                         # line 255
            return correspondent_id                                        # line 256
        else:
            return None                                                    # line 258
    else:
        return None
```

At line 254, `self.correspondent_classifier.predict(X)` calls scikit-learn's `MLPClassifier.predict()`, which returns the **raw class label** (the argmax of the decision function). It does **not** return a probability. There is no call to `predict_proba()`, no probability threshold, no score filtering, and no minimum confidence check.

The only check is at line 255: `if correspondent_id != -1`, which filters out the "no match" sentinel value. This is a label check, not a confidence check.

**Step 2 — Matching Bridge** (`src/documents/matching.py`, line 21):

```python
def match_correspondents(document, classifier):
    if classifier:
        pred_id = classifier.predict_correspondent(document.content)  # line 23
    else:
        pred_id = None

    correspondents = Correspondent.objects.all()

    return list(
        filter(lambda o: matches(o, document) or o.pk == pred_id, correspondents),  # line 30
    )
```

At line 23, the raw prediction is retrieved. At line 30, the filter includes any correspondent whose `pk` equals `pred_id` — **regardless of prediction confidence**. There is no confidence scoring or threshold applied to `pred_id`.

**Step 3 — Signal Handler** (`src/documents/signals/handlers.py`, line 35):

```python
def set_correspondent(sender, document=None, logging_group=None, classifier=None,
                      replace=False, use_first=True, suggest=False, base_url=None,
                      color=False, **kwargs):
    if document.correspondent and not replace:
        return

    potential_correspondents = matching.match_correspondents(document, classifier)  # line 50

    potential_count = len(potential_correspondents)
    if potential_correspondents:
        selected = potential_correspondents[0]  # line 54
    else:
        selected = None
```

At line 50, the handler retrieves the list of matched correspondents. At line 54, it takes the **first match** without any scoring or ranking. At lines 97–98, it assigns directly:

```python
document.correspondent = selected
document.save(update_fields=("correspondent",))
```

**Conclusion**: The entire chain from `MLPClassifier.predict()` → `match_correspondents()` → `set_correspondent()` operates **without any confidence score, probability threshold, or minimum certainty check**. The scikit-learn `MLPClassifier.predict()` method returns a single class label based on argmax of the decision function, and this label is used as-is to assign a correspondent to the document.

The same pattern applies to `predict_document_type()` (classifier.py line 262) and `predict_tags()` (classifier.py line 273). None of them use `predict_proba()` or apply any confidence filtering.

---

## 3. No-Text Edge Case and OCR Fallback Chain

### Question

What code path is exercised when a document has no extractable text? Which OCR subprocess is invoked? What is the fallback strategy? What MIME type is assigned to the output?

### Answer

When a document has no extractable text, the `RasterisedDocumentParser` follows a three-level fallback chain: (1) standard OCR via `ocrmypdf.ocr()`, (2) force-OCR retry with `safe_fallback=True`, and (3) assignment of an empty string as content. The archive output is always `application/pdf`. The original file's MIME type is detected by `magic.from_file()`.

#### 3.1 Parser Dispatch and MIME Detection

- **File**: `src/documents/consumer.py`, line 219 — MIME type is detected using python-magic:

  ```python
  mime_type = magic.from_file(self.path, mime=True)
  ```

- **Line 223**: The parser class is selected based on MIME type:

  ```python
  parser_class = get_parser_class_for_mime_type(mime_type)
  ```

- **File**: `src/paperless_tesseract/signals.py`, lines 7–19 — The tesseract parser registers for these MIME types at weight 0:

  | MIME Type | Extension |
  |---|---|
  | `application/pdf` | `.pdf` |
  | `image/jpeg` | `.jpg` |
  | `image/png` | `.png` |
  | `image/tiff` | `.tif` |
  | `image/gif` | `.gif` |
  | `image/bmp` | `.bmp` |

#### 3.2 OCR Subprocess Invocation

Defined at `src/paperless_tesseract/parsers.py`, class `RasterisedDocumentParser`, method `parse()` at line 230.

**Pre-extraction attempt (lines 234–239):**

```python
if mime_type == "application/pdf":
    text_original = self.extract_text(None, document_path)
    original_has_text = text_original and len(text_original) > 50
else:
    text_original = None
    original_has_text = False
```

For PDFs, `extract_text()` (line 99) first checks for a sidecar file, then falls back to `pdfminer_extract_text()` (line 117–120). For images, no pre-extraction is attempted.

**Skip-no-archive mode check (lines 241–244):**

```python
if settings.OCR_MODE == "skip_noarchive" and original_has_text:
    self.log("debug", "Document has text, skipping OCRmyPDF entirely.")
    self.text = text_original
    return
```

If the document already has substantial text (>50 characters) and OCR mode is `"skip_noarchive"`, OCR is bypassed entirely.

**Primary OCR invocation (lines 252–261):**

```python
args = self.construct_ocrmypdf_parameters(
    document_path, mime_type, archive_path, sidecar_file,
)
```

- **Line 261**: The primary OCR subprocess call:

  ```python
  ocrmypdf.ocr(**args)
  ```

- **Line 264**: Text is extracted from the sidecar file or the archive PDF:

  ```python
  self.text = self.extract_text(sidecar_file, archive_path)
  ```

#### 3.3 The Fallback Chain When No Text Found

**Level 1 failure — NoTextFoundException (lines 266–267):**

```python
if not self.text:
    raise NoTextFoundException("No text was found in the original document")
```

The `NoTextFoundException` class is defined at `src/paperless_tesseract/parsers.py`, line 14:

```python
class NoTextFoundException(Exception):
    pass
```

**Level 2 — Force-OCR retry (lines 276–306):**

The `NoTextFoundException` and `InputFileError` are caught at line 276:

```python
except (NoTextFoundException, InputFileError) as e:
```

The fallback creates new output paths (lines 283–284):

```python
archive_path_fallback = os.path.join(self.tempdir, "archive-fallback.pdf")
sidecar_file_fallback = os.path.join(self.tempdir, "sidecar-fallback.txt")
```

OCR parameters are reconstructed with `safe_fallback=True` (lines 288–294):

```python
args = self.construct_ocrmypdf_parameters(
    document_path, mime_type, archive_path_fallback,
    sidecar_file_fallback, safe_fallback=True,
)
```

In `construct_ocrmypdf_parameters()` at line 135, the `safe_fallback` flag triggers force-OCR mode at line 155:

```python
if settings.OCR_MODE == "force" or safe_fallback:
    ocrmypdf_args["force_ocr"] = True
```

This means `safe_fallback=True` forces OCR **regardless of the configured `OCR_MODE`**.

The second OCR attempt is made at lines 297–298:

```python
self.log("debug", f"Fallback: Calling OCRmyPDF with args: {args}")
ocrmypdf.ocr(**args)
```

Text is extracted from the fallback files at lines 303–306:

```python
self.text = self.extract_text(
    sidecar_file_fallback, archive_path_fallback,
)
```

**Level 3 — Empty string as last resort (lines 316–327):**

```python
if not self.text:
    if original_has_text:
        self.text = text_original
    else:
        self.log(
            "warning",
            f"No text was found in {document_path}, the content will "
            f"be empty.",
        )
        self.text = ""
```

If `self.text` is still empty after both OCR attempts:
- If the original PDF had text (i.e., `original_has_text is True`), it uses the pre-extracted `text_original` (line 319–320)
- Otherwise, it sets `self.text = ""` — an empty string (line 327)

#### 3.4 MIME Type Assignment

- The **original file's MIME type** is detected by `magic.from_file(self.path, mime=True)` at `src/documents/consumer.py`, line 219. This detected type is used for parser dispatch and stored with the document.
- The **archive output** from `ocrmypdf` is always written to `archive.pdf` (or `archive-fallback.pdf`) in the temp directory. Regardless of the input file type (PDF, JPEG, PNG, TIFF, GIF, BMP), the archive is always a PDF file with MIME type `application/pdf`.
- The archive path is set at `src/paperless_tesseract/parsers.py`, line 249:

  ```python
  archive_path = os.path.join(self.tempdir, "archive.pdf")
  ```

#### 3.5 Complete Fallback Chain Summary

```
Input document
    │
    ▼
[Pre-extraction via pdfminer (PDFs only, line 235)]
    │
    ▼
[OCR_MODE == "skip_noarchive" && has_text? → return early (line 241)]
    │ (no early return)
    ▼
[Primary ocrmypdf.ocr() call (line 261)]
    │
    ├─ Text extracted → success
    │
    ├─ No text → NoTextFoundException (line 267)
    │   │
    │   ▼
    │   [Force-OCR retry with safe_fallback=True (line 298)]
    │   │
    │   ├─ Text extracted → success
    │   │
    │   └─ Exception → ParseError raised (line 310)
    │
    └─ EncryptedPdfError → use original text if available (line 274-275)
        │
        ▼
[Final fallback: use original text or empty string (lines 318-327)]
```

---

## 4. Barcode Splitting and Document Record Creation

### Question

How many `Document` records result from a single input PDF when barcode splitting is active? Which barcode values trigger a split? Where in the code is that decision made? Does the resulting split affect the effective training data for the classifier?

### Answer

A single input PDF with N separator barcodes produces **N+1 document fragments**, each of which is independently consumed through the standard pipeline, resulting in **N+1 separate `Document` records**. The default separator barcode value is `"PATCHT"`. The fragments are deposited into the consumption directory for re-ingestion, meaning each fragment's resulting `Document` record becomes part of the training data for subsequent classifier training.

#### 4.1 Barcode Splitting Trigger

Defined at `src/documents/tasks.py`, function `consume_file()` at line 184.

- **Line 195**: Barcode splitting is gated by a settings flag:

  ```python
  if settings.CONSUMER_ENABLE_BARCODES:
  ```

  This setting defaults to `False`, as configured at `src/paperless/settings.py`, lines 502–504:

  ```python
  CONSUMER_ENABLE_BARCODES = __get_boolean(
      "PAPERLESS_CONSUMER_ENABLE_BARCODES",
  ```

- **Line 198**: If enabled, the file is scanned for separator barcodes:

  ```python
  separators = scan_file_for_separating_barcodes(path)
  ```

- The default barcode string is `"PATCHT"`, configured at `src/paperless/settings.py`, line 506:

  ```python
  CONSUMER_BARCODE_STRING = os.getenv("PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")
  ```

#### 4.2 Barcode Scanning

Defined at `src/documents/tasks.py`, function `scan_file_for_separating_barcodes()` at line 96.

- **Line 102**: Reads the separator barcode string from settings:

  ```python
  separator_barcode = str(settings.CONSUMER_BARCODE_STRING)
  ```

- **Line 105**: Converts all PDF pages to images using `pdf2image`:

  ```python
  pages_from_path = convert_from_path(filepath, output_folder=path)
  ```

- **Lines 106–109**: Iterates over every page, calling `barcode_reader()` for each. If the separator barcode string is found among the decoded barcodes on a page, that page number (0-indexed) is added to the separator list:

  ```python
  for current_page_number, page in enumerate(pages_from_path):
      current_barcodes = barcode_reader(page)
      if separator_barcode in current_barcodes:
          separator_page_numbers.append(current_page_number)
  ```

- Returns a list of 0-indexed page numbers containing the separator barcode.

#### 4.3 Barcode Reader

Defined at `src/documents/tasks.py`, function `barcode_reader()` at line 75.

- **Line 82**: Uses `pyzbar` to detect all barcodes in the image:

  ```python
  detected_barcodes = pyzbar.decode(image)
  ```

- **Line 88**: Decodes each barcode's data as UTF-8:

  ```python
  decoded_barcode = barcode.data.decode("utf-8")
  ```

- Returns a list of decoded barcode strings found on the page.

#### 4.4 Page Separation

Defined at `src/documents/tasks.py`, function `separate_pages()` at line 113.

- **Line 123**: Opens the source PDF via `pikepdf`:

  ```python
  pdf = Pdf.open(filepath)
  ```

- **Lines 130–138**: Creates the first document fragment from all pages **before** the first separator page:

  ```python
  dst = Pdf.new()
  for n, page in enumerate(pdf.pages):
      if n < pages_to_split_on[0]:
          dst.pages.append(page)
  ```

  The result is saved as `{fname}_document_0.pdf`.

- **Lines 141–159**: For each separator page, creates a fragment from the pages **between** the current separator and the next one (or end of document). **Separator pages themselves are excluded** — the inner loop starts at `page_number + 1` (line 149):

  ```python
  for page in range(page_number + 1, next_page):
      dst.pages.append(pdf.pages[page])
  ```

- The function returns a list of temporary file paths, one per fragment. Given N separator pages, this produces **N+1 fragments**.

**Example**: A 7-page PDF with separator barcodes on pages 2 and 5 (0-indexed):
- Fragment 0: pages 0, 1 (before separator at page 2)
- Fragment 1: pages 3, 4 (between separator at page 2 and separator at page 5)
- Fragment 2: page 6 (after separator at page 5)
- Total: **3 fragments from 2 separators**

#### 4.5 Fragment Consumption and Document Record Creation

Back in `consume_file()` at `src/documents/tasks.py`, lines 199–233:

- **Line 201**: Gets the list of fragment file paths:

  ```python
  document_list = separate_pages(path, separators)
  ```

- **Lines 203–210**: Each fragment is saved to the consumption directory:

  ```python
  for n, document in enumerate(document_list):
      if override_filename:
          newname = f"{str(n)}_" + override_filename
      else:
          newname = None
      save_to_dir(document, newname=newname)
  ```

  The `save_to_dir()` function (line 164) copies the fragment to `settings.CONSUMPTION_DIR`.

- **Line 214**: The original file is deleted:

  ```python
  os.unlink(path)
  ```

**Critical observation**: Fragments are **NOT consumed in-line** within the `consume_file()` function. They are deposited into the consumption directory (`CONSUMPTION_DIR`), where the standard directory watcher picks them up. Each fragment goes through the **full** `consume_file()` → `Consumer.try_consume_file()` pipeline independently. This means **each fragment produces its own `Document` record** through the normal consumption flow.

#### 4.6 Training Data Impact

- The new `Document` records created by consuming fragments contribute to the training dataset for subsequent `train_classifier()` calls.
- The training query at `src/documents/classifier.py`, lines 125–127 queries **all** non-inbox documents:

  ```python
  Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)
  ```

- If barcode splitting creates real `Document` records and `train_classifier()` is invoked afterward, the effective training data **grows** during the test run.
- Whether this actually affects test isolation depends on the Django test infrastructure:
  - `TestCase` wraps each test method in a transaction that is rolled back after the method completes, so documents created within one test method do not persist to the next.
  - However, if barcode splitting and classifier training occur within the **same** test method, the fragments' documents are visible to the training query during that method's execution.
  - `DirectoriesMixin` provides **filesystem** isolation (separate `MODEL_FILE`, `CONSUMPTION_DIR`, etc.) but does not provide additional database isolation beyond what `TestCase` already provides.

---

## 5. Root Causes of Non-Deterministic Test Failures

### Question

What are the root causes of non-deterministic behavior in the ML classification pipeline during test execution?

### Answer

Five distinct sources of non-determinism were identified in the codebase.

#### 5.1 Missing `random_state` on `MLPClassifier` — PRIMARY CAUSE

At `src/documents/classifier.py`, three `MLPClassifier` instances are created without a `random_state` parameter:

| Line | Classifier | Code |
|---|---|---|
| 219 | Tags | `self.tags_classifier = MLPClassifier(tol=0.01)` |
| 227 | Correspondent | `self.correspondent_classifier = MLPClassifier(tol=0.01)` |
| 238 | Document Type | `self.document_type_classifier = MLPClassifier(tol=0.01)` |

scikit-learn's `MLPClassifier` uses random weight initialization for the neural network layers. When `random_state` is not set (defaults to `None`), the weight initialization uses a different random seed on each instantiation. This produces different trained models and potentially different predictions for the same input data, even when the training data is identical.

This is the **primary source of non-deterministic predictions**. Given the same training documents and content, two separate `train()` calls on two separate `DocumentClassifier()` instances can produce classifiers that make different predictions for the same input.

#### 5.2 Parallel Test Execution

At `src/setup.cfg`, line 10:

```
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
```

The `--numprocesses auto` flag enables pytest-xdist parallel execution, which creates multiple worker processes. This introduces several sources of non-determinism:

- **Test ordering varies between runs**: pytest-xdist distributes tests across workers in a non-deterministic order.
- **Filesystem race conditions**: If `DirectoriesMixin` is not used consistently, multiple workers could access the same `MODEL_FILE` pickle concurrently. The default `MODEL_FILE` at `src/paperless/settings.py`, line 74 is a fixed path:

  ```python
  MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")
  ```

  Without `DirectoriesMixin` overriding this to a temp path, concurrent workers would read/write the same file.

- **Inconsistent test isolation**: Some test classes do NOT use `DirectoriesMixin`. For example, `TestDocumentConsumptionFinishedSignal` at `src/documents/tests/test_matchables.py`, line 380 extends `TestCase` directly without `DirectoriesMixin`. It only creates a temporary `INDEX_DIR` (line 394) and uses `override_settings(INDEX_DIR=self.index_dir)` (line 396), but does NOT override `MODEL_FILE`, `SCRATCH_DIR`, or `CONSUMPTION_DIR`.

#### 5.3 Small Training Datasets Amplify Non-Determinism

The standard test data created by `generate_test_data()` in `src/documents/tests/test_classifier.py` (line 25) creates only **2 effective training documents** (`doc1` and `doc2`; `doc_inbox` is excluded).

With such a small dataset:
- The MLP decision boundary is highly sensitive to random weight initialization
- Minor numerical differences in training (caused by different random seeds) can flip predictions between classes
- The `CountVectorizer` at line 194 with `min_df=0.01` means virtually all features from only 2 documents will be included, but the MLP's learned weights over these features are unstable

For example, the `testPredict` test at line 115 expects:
- `predict_correspondent(doc1.content)` → `c1.pk` (line 119)
- `predict_correspondent(doc2.content)` → `None` (line 122)

With only 2 training points and random weight initialization, these predictions are not guaranteed to be stable across different classifier instantiations.

#### 5.4 `CountVectorizer` Feature Vocabulary Sensitivity

At `src/documents/classifier.py`, lines 194–198:

```python
self.data_vectorizer = CountVectorizer(
    analyzer="word",
    ngram_range=(1, 2),
    min_df=0.01,
)
```

The `min_df=0.01` parameter means features appearing in fewer than 1% of documents are dropped. With only 2 training documents, this threshold effectively requires a feature to appear in at least 1 document (since `0.01 * 2 = 0.02`, which rounds up to 1). While this is a permissive threshold, the feature vocabulary is still a function of the exact training corpus.

If parallel test workers create different documents or in a different order, the feature vocabulary could theoretically differ, leading to different vectorized representations and different classifier behavior.

#### 5.5 Signal-Driven Post-Consumption Flow

At `src/documents/apps.py`, lines 22–27, six handlers are connected to the `document_consumption_finished` signal in this order:

```python
document_consumption_finished.connect(add_inbox_tags)     # line 22
document_consumption_finished.connect(set_correspondent)  # line 23
document_consumption_finished.connect(set_document_type)  # line 24
document_consumption_finished.connect(set_tags)           # line 25
document_consumption_finished.connect(set_log_entry)      # line 26
document_consumption_finished.connect(add_to_index)       # line 27
```

Django signals execute handlers in the order they were connected. Three of these handlers use the classifier:
1. `set_correspondent` (`src/documents/signals/handlers.py`, line 35) — calls `matching.match_correspondents(document, classifier)` at line 50
2. `set_document_type` (line 101) — calls `matching.match_document_types(document, classifier)` at line 116
3. `set_tags` (line 168) — calls `matching.match_tags(document, classifier)` at line 189

Since the classifier is non-deterministic (Section 5.1), the tags, correspondent, and document type assigned to a document can vary between runs. Furthermore, `add_inbox_tags` (line 22 in apps.py) runs **before** the classifier-dependent handlers and adds inbox tags to the document. This modifies the document's tag set before `set_tags` is called, but since `set_tags` adds to existing tags (via `document.tags.add()` at handlers.py line 230), this is additive and doesn't cause ordering conflicts.

The more significant concern is that `set_correspondent` and `set_document_type` each call `document.save(update_fields=...)` (handlers.py lines 98 and 165 respectively), which persists their non-deterministic assignments to the database within the same transaction.

#### 5.6 Summary Table of Non-Determinism Sources

| Source | Location | Severity | Impact |
|---|---|---|---|
| Missing `random_state` on `MLPClassifier` | `src/documents/classifier.py` lines 219, 227, 238 | **High** | Different weight initialization produces different trained models; predictions vary across classifier instances |
| Parallel test execution (`--numprocesses auto`) | `src/setup.cfg` line 10 | **Medium** | Non-deterministic test ordering, potential filesystem race conditions on `MODEL_FILE` |
| Small training datasets (2 effective docs) | `src/documents/tests/test_classifier.py` line 25 | **Medium** | Amplifies weight initialization sensitivity; decision boundaries are unstable with few data points |
| `CountVectorizer` feature vocabulary | `src/documents/classifier.py` line 194 | **Low** | Feature set depends on exact corpus; theoretically varies if document timing differs across workers |
| Signal-driven classification flow | `src/documents/apps.py` lines 22–27 | **Low** | Non-deterministic classifier propagates through signal chain, potentially causing cascading inconsistencies in assigned metadata |

---

## 6. Signal-Driven Post-Consumption Flow

This section documents the complete signal chain that executes after a document is consumed, which is the mechanism by which the ML classifier's predictions are applied to documents.

### 6.1 Signal Definitions

At `src/documents/signals/__init__.py`, three signals are defined:

```python
document_consumption_started = Signal()    # line 3
document_consumption_finished = Signal()   # line 4
document_consumer_declaration = Signal()   # line 5
```

- `document_consumption_started`: Fired before parsing begins (consumer.py lines 229–233)
- `document_consumption_finished`: Fired after the document is stored in the database (consumer.py lines 306–311)
- `document_consumer_declaration`: Used for parser registration (parsers.py `get_parser_class_for_mime_type()`)

### 6.2 Handler Wiring

At `src/documents/apps.py`, the `DocumentsConfig.ready()` method (line 11) wires six handlers to the `document_consumption_finished` signal:

| Order | Handler | Source File | Line (apps.py) | Uses Classifier? |
|---|---|---|---|---|
| 1 | `add_inbox_tags` | `src/documents/signals/handlers.py` line 30 | 22 | No |
| 2 | `set_correspondent` | `src/documents/signals/handlers.py` line 35 | 23 | **Yes** |
| 3 | `set_document_type` | `src/documents/signals/handlers.py` line 101 | 24 | **Yes** |
| 4 | `set_tags` | `src/documents/signals/handlers.py` line 168 | 25 | **Yes** |
| 5 | `set_log_entry` | `src/documents/signals/handlers.py` | 26 | No |
| 6 | `add_to_index` | `src/documents/signals/handlers.py` | 27 | No |

### 6.3 Signal Emission in the Consumer

At `src/documents/consumer.py`, the signal is emitted within a `transaction.atomic()` block (lines 298–311):

```python
with transaction.atomic():
    document = self._store(text=text, date=date, mime_type=mime_type)  # line 301

    document_consumption_finished.send(
        sender=self.__class__,
        document=document,
        logging_group=self.logging_group,
        classifier=classifier,                                         # line 310
    )
```

The classifier is loaded **before** the transaction begins, at line 292:

```python
classifier = load_classifier()
```

This means all six signal handlers operate within the same database transaction, and the classifier instance is shared across all three classifier-dependent handlers. If the classifier is `None` (no model file exists), all prediction-dependent matching falls back to `pred_id = None`, and only rule-based matching applies.

### 6.4 Handler Behavior Details

**`add_inbox_tags`** (handlers.py line 30): Adds all tags with `is_inbox_tag=True` to the document. Does not use the classifier.

**`set_correspondent`** (handlers.py line 35):
- Skips if document already has a correspondent and `replace=False` (line 47)
- Calls `matching.match_correspondents(document, classifier)` (line 50)
- Takes `potential_correspondents[0]` — the first match (line 54)
- Assigns and saves (lines 97–98)

**`set_document_type`** (handlers.py line 101):
- Same pattern as `set_correspondent`
- Calls `matching.match_document_types(document, classifier)` (line 116)
- Takes first match (line 120)
- Assigns and saves (lines 164–165)

**`set_tags`** (handlers.py line 168):
- Computes `relevant_tags = set(matched_tags) - current_tags` (line 191)
- Only adds tags that are not already present
- Calls `document.tags.add(*relevant_tags)` (line 230)

---

## 7. Test Infrastructure and Isolation

### 7.1 `DirectoriesMixin`

Defined at `src/documents/tests/utils.py`, line 72:

```python
class DirectoriesMixin:
    def setUp(self) -> None:
        self.dirs = setup_directories()
        super(DirectoriesMixin, self).setUp()

    def tearDown(self) -> None:
        super(DirectoriesMixin, self).tearDown()
        remove_dirs(self.dirs)
```

This mixin provides filesystem isolation for test classes by creating fresh temporary directories and overriding Django settings to point to them.

### 7.2 `setup_directories()`

Defined at `src/documents/tests/utils.py`, line 14. Creates the following isolated directories:

| Setting | Created At | Line |
|---|---|---|
| `DATA_DIR` | `tempfile.mkdtemp()` | 18 |
| `SCRATCH_DIR` | `tempfile.mkdtemp()` | 19 |
| `MEDIA_ROOT` | `tempfile.mkdtemp()` | 20 |
| `CONSUMPTION_DIR` | `tempfile.mkdtemp()` | 21 |
| `INDEX_DIR` | `os.path.join(dirs.data_dir, "index")` | 22 |
| `ORIGINALS_DIR` | `os.path.join(dirs.media_dir, "documents", "originals")` | 23 |
| `THUMBNAIL_DIR` | `os.path.join(dirs.media_dir, "documents", "thumbnails")` | 24 |
| `ARCHIVE_DIR` | `os.path.join(dirs.media_dir, "documents", "archive")` | 25 |
| `LOGGING_DIR` | `os.path.join(dirs.data_dir, "log")` | 26 |
| **`MODEL_FILE`** | `os.path.join(dirs.data_dir, "classification_model.pickle")` | **45** |
| `MEDIA_LOCK` | `os.path.join(dirs.media_dir, "media.lock")` | 46 |

The `MODEL_FILE` override at line 45 is critical for classifier isolation — it ensures each test class that uses `DirectoriesMixin` gets its own pickle file path, preventing cross-test contamination.

### 7.3 Parallel Test Configuration

At `src/setup.cfg`, line 10:

```
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
```

- `--numprocesses auto`: Enables pytest-xdist parallel execution with automatic worker count detection
- Each worker gets its own database instance (Django test runner handles this automatically)
- Filesystem isolation depends on test classes using `DirectoriesMixin`

At `src/setup.cfg`, line 12:

```
PAPERLESS_DISABLE_DBHANDLER=true
```

This environment variable disables the custom database log handler during tests, preventing log-related database operations from interfering with test execution.

### 7.4 Test Classes and Their Isolation

| Test Class | File | Uses `DirectoriesMixin`? | Risk Level |
|---|---|---|---|
| `TestClassifier` | `test_classifier.py` line 20 | **Yes** | Low — fully isolated |
| `TestDocumentConsumptionFinishedSignal` | `test_matchables.py` line 380 | **No** | Medium — only `INDEX_DIR` is isolated |

`TestDocumentConsumptionFinishedSignal` (test_matchables.py line 380) does not use `DirectoriesMixin`. It only creates a temporary `INDEX_DIR` at line 394:

```python
self.index_dir = tempfile.mkdtemp()
override_settings(INDEX_DIR=self.index_dir).enable()
```

It sends the `document_consumption_finished` signal **without** a `classifier` kwarg (e.g., line 407–410):

```python
document_consumption_finished.send(
    sender=self.__class__,
    document=self.doc_contains,
)
```

Since no `classifier=` keyword argument is passed, the signal handlers receive `classifier=None` via `**kwargs`, causing all ML predictions to return `None` and only rule-based matching to apply.

---

## 8. Dependency Versions

The following dependency versions are pinned in `requirements.txt` and are directly relevant to the ML classification pipeline, OCR processing, and barcode splitting subsystems analyzed in this investigation:

| Package | Version | Source Line | Role in Pipeline |
|---|---|---|---|
| scikit-learn | 1.0.2 | `requirements.txt` line 88 | ML classifier engine: `MLPClassifier`, `CountVectorizer`, `MultiLabelBinarizer`, `LabelBinarizer` |
| ocrmypdf | 13.4.3 | `requirements.txt` line 60 | OCR subprocess invoked by `RasterisedDocumentParser.parse()` via `ocrmypdf.ocr()` |
| pyzbar | 0.1.9 | `requirements.txt` line 83 | Barcode decoding in `tasks.barcode_reader()` via `pyzbar.decode()` |
| pdf2image | 1.16.0 | `requirements.txt` line 63 | PDF page rendering in `tasks.scan_file_for_separating_barcodes()` via `convert_from_path()` |
| pikepdf | 5.1.1 | `requirements.txt` line 65 | PDF manipulation in `tasks.separate_pages()` via `Pdf.open()` / `Pdf.new()` |
| django | 4.0.4 | `requirements.txt` line 38 | Web framework: ORM, test runner, signals, settings, `TestCase` |
| pillow | 9.1.0 | `requirements.txt` line 66 | Image handling in `RasterisedDocumentParser` for alpha detection, DPI extraction |
| fuzzywuzzy | 0.18.0 | `requirements.txt` line 41 | Fuzzy matching in `matching.py` line 135 via `fuzz.partial_ratio()` with threshold 90 |
| python-magic | 0.4.25 | `requirements.txt` | MIME type detection in `consumer.py` line 219 via `magic.from_file()` |
| filelock | 3.6.0 | `requirements.txt` line 40 | Concurrency control in `consumer.py` and `signals/handlers.py` for media file operations |

**Note on scikit-learn version pinning**: The exact version `1.0.2` is pinned because `DocumentClassifier.FORMAT_VERSION = 7` (classifier.py line 63) is tied to the scikit-learn version. Upgrading scikit-learn would require incrementing `FORMAT_VERSION` to invalidate existing pickled models, as the serialized `MLPClassifier` objects may not be compatible across versions.
