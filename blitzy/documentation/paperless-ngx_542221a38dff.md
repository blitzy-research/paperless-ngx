# Paperless-ngx Document-Classification Test Flakiness — Root-Cause Analysis

> This is a **read-only, code-grounded investigation** into why the document-classification
> tests in paperless-ngx behave non-deterministically ("flaky"). Every factual claim below is
> anchored to a `[path:line]` citation in the actual source on this branch, and the key behaviors
> are corroborated by **actually running the relevant tests** in the provided environment. No
> source file was modified; the only artifact produced is this document. The diagnosed fixes are
> presented as **written recommendations only** — they were intentionally **not** implemented,
> per the scope of this task.

---

## 1. Introduction & Methodology

### 1.1 Purpose

The document-classification subsystem of paperless-ngx trains a small neural network
(`MLPClassifier`) over the stored documents and uses it to auto-assign correspondents, document
types, and tags. Its unit tests are reported to fail intermittently. This analysis answers five
precise questions about the pipeline's behavior during test execution and then synthesizes the
**root causes of the non-determinism**.

The five questions:

| # | Question |
|---|----------|
| **Q1** | Does `DocumentClassifier` load a previously serialized model from disk or retrain a fresh model within the same test run, and how does the persisted/shared model artifact affect later tests? |
| **Q2** | For a correspondent-matching test: (a) how many training documents are created, (b) when does training occur relative to the inserts, and (c) what confidence threshold accepts/rejects a prediction? |
| **Q3** | For a document with no extractable text, which OCR subprocess is invoked and what MIME type is assigned to the produced output? |
| **Q4** | How many document records result from a single multi-page input, which barcode values trigger a split, and where exactly is the split decision made? |
| **Q5** | Does barcode splitting change the effective number of documents that feed the classifier / auto-matching during a test run? |

### 1.2 Code-as-truth principle

Every conclusion is derived from the source code, which is treated as the single source of truth.
No behavior is assumed. Where a behavior could be misread from the code alone, it is **corroborated
by running the relevant test(s)** and, where useful, by an isolated probe that reproduces the exact
library call the code makes.

### 1.3 Citation format

Citations use the form **`[path:line]`** for a single line and **`[path:Lstart-Lend]`** for a span,
where `path` is repository-relative (e.g. `src/documents/classifier.py`). All line numbers were
re-confirmed against the current file contents on this branch at the time of writing; if the code
shifts, the code — not the citation — is authoritative.

### 1.4 Environment & pinned dependency versions

The runtime is pinned by the base image to **Python 3.9** — `FROM python:3.9-slim-bullseye as main-app`
`[Dockerfile:L18]`. The dependencies that govern the behaviors under investigation are pinned in
`requirements.txt` (verified in the running environment as Python 3.9.23):

| Package | Version | Relevance |
|---------|---------|-----------|
| scikit-learn | `1.0.2` | `MLPClassifier` used by `DocumentClassifier` `[requirements.txt]` |
| joblib | `1.1.0` | serialization backend for scikit-learn `[requirements.txt]` |
| numpy | `1.22.3` | numerical backend; global RNG used by the estimator `[requirements.txt]` |
| scipy | `1.8.0` | scientific backend `[requirements.txt]` |
| ocrmypdf | `13.4.3` | the OCR subprocess invoked by the parser `[requirements.txt]` |
| pdf2image | `1.16.0` | renders PDF pages for barcode scanning `[requirements.txt]` |
| pyzbar | `0.1.9` | decodes barcodes during the split decision `[requirements.txt]` |
| django | `4.0.4` | web/ORM framework hosting the `documents` app `[requirements.txt]` |
| python-magic | (per `Pipfile`) | MIME-type detection of the input document `[Pipfile]` |
| pytest-xdist | (dev) | parallel test execution (`--numprocesses auto`) — central to the flakiness `[Pipfile]` |

The dev/test harness (`pytest-django`, `pytest-env`, `pytest-xdist`, `pytest-sugar`, `pytest-cov`,
`factory-boy`) is declared in `[Pipfile]`.

### 1.5 How the tests are run

The **only** pytest configuration lives in `src/setup.cfg` under `[tool:pytest]` (there is no
`tox.ini`, `pytest.ini`, or `pyproject.toml`):

```ini
[tool:pytest]
DJANGO_SETTINGS_MODULE=paperless.settings
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
env =
  PAPERLESS_DISABLE_DBHANDLER=true
```

The `--numprocesses auto` flag `[src/setup.cfg:L10]` runs the suite **in parallel** across worker
processes (pytest-xdist), and `PAPERLESS_DISABLE_DBHANDLER=true` `[src/setup.cfg:L12]` disables the
DB log handler. The parallel flag is central to the non-determinism analysis (Section 7).

For this investigation, the targeted tests were run **serially with coverage disabled**
(`-n0 --no-cov`) to obtain clean, reproducible signals without leaving coverage/cache artifacts in
the repository. The results are embedded inline in each section under **Observed behavior**.

### 1.6 Document structure

Sections 2–6 answer Q1–Q5. Each uses a three-part pattern:

- **Evidence** — exact code citations.
- **Observed behavior** — what the corresponding test(s) actually do when run.
- **Rationale / Conclusion** — the reasoned answer.

Section 7 consolidates the determinism-relevant facts into a root-cause synthesis with
documentation-only recommendations. Section 8 lists references.

### 1.7 Two factual corrections discovered during investigation

So the reader trusts code over any prior specification, two naming facts were verified directly
against disk:

1. **Barcode logic lives in `src/documents/tasks.py`**, not a standalone `barcodes.py`. The
   functions `barcode_reader` `[src/documents/tasks.py:L75-L93]`,
   `scan_file_for_separating_barcodes` `[src/documents/tasks.py:L96-L110]`, and `separate_pages`
   `[src/documents/tasks.py:L113-L161]` are all defined there.
2. **The rule-based matching tests live in `src/documents/tests/test_matchables.py`**, not
   `test_matching.py`.

---

## 2. Q1 — Does the classifier reuse a saved model or retrain?

**Short answer:** It **reuses** an existing model whenever the training-data hash is unchanged, and
only **retrains + saves** when the underlying documents actually change. A leftover or shared
on-disk model file whose hash matches therefore **suppresses retraining** in later tests, leaking
model state across the suite.

### 2.1 Evidence

**Loading an existing model.** `load_classifier()` returns `None` when the model file does not exist;
otherwise it constructs a `DocumentClassifier` and calls `.load()`
`[src/documents/classifier.py:L30-L57]`:

```python
def load_classifier():
    if not os.path.isfile(settings.MODEL_FILE):
        ...
        return None
    classifier = DocumentClassifier()
    try:
        classifier.load()
    ...
    return classifier
```

`DocumentClassifier.load()` reads the pickle, checks `FORMAT_VERSION`, and restores `self.data_hash`
together with the vectorizer, binarizer, and the three classifiers
`[src/documents/classifier.py:L76-L94]`. Crucially, the persisted **`data_hash`** is restored
`[src/documents/classifier.py:L86]`, so a loaded model "remembers" the fingerprint of the data it
was last trained on.

**The retrain short-circuit.** `DocumentClassifier.train()` computes a SHA-1 hash over **all
non-inbox documents ordered by primary key** `[src/documents/classifier.py:L124-L127]`:

```python
m = hashlib.sha1()
for doc in Document.objects.order_by("pk").exclude(
    tags__is_inbox_tag=True,
):
    preprocessed_content = preprocess_content(doc.content)
    m.update(preprocessed_content.encode("utf-8"))
    ...
```

It then compares the freshly computed hash against the stored one
`[src/documents/classifier.py:L161-L164]`:

```python
new_data_hash = m.digest()

if self.data_hash and new_data_hash == self.data_hash:
    return False
```

If the hash is unchanged it returns `False` **without retraining**. Only on a genuine (re)train does
the model fit, set `self.data_hash = new_data_hash` `[src/documents/classifier.py:L247]`, and
`return True` `[src/documents/classifier.py:L249]`.

**Persistence is gated on `train()` returning `True`.** The `train_classifier` task first tries
`load_classifier()` and falls back to a fresh `DocumentClassifier()`
`[src/documents/tasks.py:L57-L60]`, then **saves only when `train()` returns `True`**, otherwise it
logs that the data is unchanged `[src/documents/tasks.py:L63-L69]`:

```python
if classifier.train():
    logger.info("Saving updated classifier model to {}...".format(settings.MODEL_FILE))
    classifier.save()
else:
    logger.debug("Training data unchanged.")
```

**Where the model lives.** `save()` writes the model atomically via a `.part` temp file then
`shutil.move` `[src/documents/classifier.py:L96-L113]`, and `MODEL_FILE` defaults to
`<DATA_DIR>/classification_model.pickle` `[src/paperless/settings.py:L74]`.

### 2.2 Observed behavior

Two tests prove the reuse logic. Run serially with coverage disabled:

```
$ pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
         documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
         documents/tests/test_classifier.py::TestClassifier::testTrain \
         -n0 --no-cov
3 passed in 2.22s
```

- `testDatasetHashing` `[src/documents/tests/test_classifier.py:L137-L142]` asserts `train()`
  returns `True` on the first call and `False` on the second call over the **same** data:

  ```python
  self.assertTrue(self.classifier.train())
  self.assertFalse(self.classifier.train())
  ```

  This directly exercises the hash short-circuit at `[src/documents/classifier.py:L163-L164]`.

- `testSaveClassifier` `[src/documents/tests/test_classifier.py:L167-L178]` trains, saves, then
  loads a **fresh** classifier from disk and asserts its `train()` returns `False`
  `[src/documents/tests/test_classifier.py:L178]`:

  ```python
  new_classifier = DocumentClassifier()
  new_classifier.load()
  self.assertFalse(new_classifier.train())
  ```

  Because `load()` restored the persisted `data_hash`, the freshly loaded model recognizes the data
  as unchanged and **does not retrain** — proving that a persisted model suppresses retraining.

### 2.3 Rationale / Conclusion

The system is **reuse-first**: it loads the on-disk model if present and retrains only when the
SHA-1 fingerprint of the (non-inbox) document set changes. Consequently, **a stale or shared
`MODEL_FILE` whose hash matches the current data will cause `train()` to return early and reuse the
previously fitted weights**, rather than producing a fresh model for the current test. This is a
direct cross-test state-leakage vector and is revisited in Section 7 as **root cause #2**.


---

## 3. Q2 — Correspondent-matching test: document count, training timing, confidence threshold

**Short answer:** (a) **3 documents are created but only 2 are effective** training data (one is
inbox-tagged and excluded); (b) **training runs after the document inserts** in the test, while in
production matching is a post-consume signal-handler step decoupled from the separate
`train_classifier` task; (c) **there is no confidence/probability threshold at all** — acceptance is
the estimator's hard predicted class id, rejected only when that id is `-1`.

### 3.1 (a) Document count — Evidence

`generate_test_data()` `[src/documents/tests/test_classifier.py:L25-L82]` creates exactly **three**
`Document` records:

| Document | Definition | Correspondent | Tags |
|----------|-----------|---------------|------|
| `doc1` | `[L60-L66]` | `c1` (`MATCH_AUTO`) | `t1` |
| `doc2` | `[L67-L72]` | `c2` (not `MATCH_AUTO`) | `t1`, `t3` |
| `doc_inbox` | `[L73-L77]` | none | `t2` |

The third document, `doc_inbox`, is tagged with `t2` `[src/documents/tests/test_classifier.py:L82]`,
and `t2` is created with `is_inbox_tag=True` `[src/documents/tests/test_classifier.py:L40-L45]`.
Because `train()` **excludes inbox-tagged documents** via `.exclude(tags__is_inbox_tag=True)`
`[src/documents/classifier.py:L125-L127]`, `doc_inbox` never enters the training set.

> **Therefore: 3 documents are created, but only 2 (`doc1`, `doc2`) are effective training data.**
> This is a frequently-missed nuance. A further subtlety relevant to the prediction outcome: `doc1`'s
> correspondent `c1` is `MATCH_AUTO` `[src/documents/tests/test_classifier.py:L26-L29]` so its label
> is `c1.pk`, whereas `doc2`'s correspondent `c2` is **not** auto-matching
> `[src/documents/tests/test_classifier.py:L30]`, so its correspondent label is `-1`. This is why
> the trained correspondent classifier's classes are `[-1, c1.pk]` (see Observed behavior).

### 3.2 (b) Training timing — Evidence

**In tests**, training runs **after** the inserts. Both `testTrain`
`[src/documents/tests/test_classifier.py:L103-L113]` and `testPredict`
`[src/documents/tests/test_classifier.py:L115-L135]` call `generate_test_data()` first and then
`self.classifier.train()`.

**In production**, matching and (re)training are **decoupled**:

- Matching runs **post-consumption via signal handlers**, using a classifier obtained at consume
  time. `set_correspondent` calls `match_correspondents(document, classifier)`
  `[src/documents/signals/handlers.py:L50]`; `set_document_type` calls `match_document_types`
  `[src/documents/signals/handlers.py:L116]`; `set_tags` calls `match_tags`
  `[src/documents/signals/handlers.py:L189]`.
- `match_correspondents` itself calls `classifier.predict_correspondent(document.content)`
  `[src/documents/matching.py:L21-L31]` (the prediction call is at
  `[src/documents/matching.py:L23]`).
- (Re)training is a **separate task**, `train_classifier` `[src/documents/tasks.py:L48-L72]`, which
  is invoked on its own schedule rather than inline with consumption.

### 3.3 (c) Confidence threshold — Evidence (the key finding: there is none)

`predict_correspondent` returns the estimator's **hard predicted class id**; it returns `None`
**only** when that id is `-1` `[src/documents/classifier.py:L251-L260]`:

```python
def predict_correspondent(self, content):
    if self.correspondent_classifier:
        X = self.data_vectorizer.transform([preprocess_content(content)])
        correspondent_id = self.correspondent_classifier.predict(X)
        if correspondent_id != -1:
            return correspondent_id
        else:
            return None
    else:
        return None
```

There is **no probability comparison**: the code calls `predict(X)`
`[src/documents/classifier.py:L254]`, not `predict_proba`. A repository-wide search confirms this:

```
$ grep -rn "predict_proba" src/
(no matches)
```

The only numeric thresholds anywhere in the matching/classification code are **unrelated** to
prediction acceptance:

- The **fuzzy rule-based matcher** accepts when `fuzz.partial_ratio(match, text) >= 90`
  `[src/documents/matching.py:L135]`. This is one of six rule-based algorithms —
  `MATCH_ALL` `[src/documents/matching.py:L72]`, `MATCH_ANY` `[src/documents/matching.py:L84]`,
  `MATCH_LITERAL` `[src/documents/matching.py:L91]`, `MATCH_REGEX` `[src/documents/matching.py:L107]`,
  `MATCH_FUZZY` `[src/documents/matching.py:L127]`, `MATCH_AUTO` `[src/documents/matching.py:L147]` —
  and is the *keyword* matcher, not the classifier's acceptance rule.
- The **MLP convergence tolerance** `tol=0.01` `[src/documents/classifier.py:L219]` is an optimizer
  stopping criterion, not a prediction-acceptance probability.

Neither is a confidence threshold for accepting a classifier prediction.

### 3.4 Observed behavior

`testPredict` `[src/documents/tests/test_classifier.py:L115-L135]` asserts a **hard** prediction:

```python
self.assertEqual(self.classifier.predict_correspondent(self.doc1.content), self.c1.pk)   # L118-L121
self.assertEqual(self.classifier.predict_correspondent(self.doc2.content), None)         # L122
```

`testTrain` `[src/documents/tests/test_classifier.py:L103-L113]` asserts the learned classes are
`[-1, c1.pk]` `[src/documents/tests/test_classifier.py:L108]`, consistent with the label analysis in
§3.1.

Run in isolation, these pass reliably for this tiny, well-separated 2-document dataset:

```
$ for i in $(seq 1 20); do pytest .../testPredict -n0 --no-cov -q; done
testPredict over 20 independent runs: PASS=20 FAIL=0
```

However, the **underlying estimator is provably non-deterministic**. An isolated probe that
reproduces the exact training pipeline (same `CountVectorizer` config + `MLPClassifier(tol=0.01)`
with **no** `random_state`) on the same two effective documents shows the fitted weights differ on
every run:

```
distinct fitted-weight signatures across 8 identical fits: 8
=> weights differ run-to-run: True
predicted label for doc2 across 8 runs: [-1, -1, -1, -1, -1, -1, -1, -1]
```

For this particular dataset the *hard prediction* for `doc2` is stable (always `-1` → `None`), so
`testPredict`'s assertion at `[src/documents/tests/test_classifier.py:L122]` does not flip in
isolation — but the weights themselves are random run-to-run, so the assertion carries a **latent
flip risk** that grows as the training set becomes larger or less separable (see Q5 and Section 7).

### 3.5 Rationale / Conclusion

- **(a)** Three documents are created; **two are effective** (the inbox-tagged `doc_inbox` is
  excluded by `[src/documents/classifier.py:L125-L127]`).
- **(b)** In the test, `train()` runs **after** the inserts; in production, matching is a
  post-consume signal-handler step (`[src/documents/signals/handlers.py:L50,L116,L189]`) that is
  **decoupled** from the separate `train_classifier` task `[src/documents/tasks.py:L48-L72]`.
- **(c)** There is **no confidence/probability threshold**. Acceptance is the **hard predicted class
  id**, rejected only when it equals `-1` `[src/documents/classifier.py:L254-L258]`. The `>= 90`
  fuzzy ratio `[src/documents/matching.py:L135]` and `tol=0.01` `[src/documents/classifier.py:L219]`
  are unrelated to prediction acceptance.

A hard class id with no probability cushion, produced by an estimator with no fixed seed, is exactly
the combination that makes assertions like `testPredict` flaky — tied to Section 7, **root cause #1**.


---

## 4. Q3 — No-extractable-text OCR: which subprocess runs and what MIME type is assigned

**Short answer:** The invoked subprocess is **`ocrmypdf.ocr(...)`** — a primary call and, on a
"no text found" condition, a **force-OCR fallback** call. The OCR archive is produced as **PDF/A**
(`OCR_OUTPUT_TYPE="pdfa"`). The **document's MIME type is taken from the input file** via
`magic.from_file(self.path, mime=True)` (commonly `application/pdf`) — it is **not** redefined by the
OCR output. For a genuinely text-free document, the pipeline ultimately yields empty text
(`self.text = ""`) while still producing the PDF/A archive.

### 4.1 Evidence

`RasterisedDocumentParser.parse()` begins at `[src/paperless_tesseract/parsers.py:L230]`. For a PDF
input it extracts the original text and considers text "present" only when it exceeds 50 characters
`[src/paperless_tesseract/parsers.py:L236]`:

```python
if mime_type == "application/pdf":
    text_original = self.extract_text(None, document_path)
    original_has_text = text_original and len(text_original) > 50
```

**Default mode runs OCR.** The early "skip OCR entirely" return is taken *only* if
`settings.OCR_MODE == "skip_noarchive" and original_has_text`
`[src/paperless_tesseract/parsers.py:L241]`. The default `OCR_MODE` is **`"skip"`**, not
`"skip_noarchive"` `[src/paperless/settings.py:L522]`, so **by default this branch is not taken** and
OCR proceeds. (This distinction matters: a no-text document would run OCR under the default config
regardless.)

**Primary OCR subprocess** — `ocrmypdf.ocr(**args)` `[src/paperless_tesseract/parsers.py:L261]`:

```python
ocrmypdf.ocr(**args)
self.archive_path = archive_path
self.text = self.extract_text(sidecar_file, archive_path)
if not self.text:
    raise NoTextFoundException("No text was found in the original document")
```

If no text results, it raises `NoTextFoundException` `[src/paperless_tesseract/parsers.py:L266-L267]`.

**Force-OCR fallback** — on `except (NoTextFoundException, InputFileError)`
`[src/paperless_tesseract/parsers.py:L276]`, it rebuilds the arguments with `safe_fallback=True`
`[src/paperless_tesseract/parsers.py:L293]` and calls `ocrmypdf.ocr(**args)` **again**
`[src/paperless_tesseract/parsers.py:L298]`.

**Last resort** — if there is still no text, it sets `self.text = ""`
`[src/paperless_tesseract/parsers.py:L318-L327]` (specifically `self.text = ""`
`[src/paperless_tesseract/parsers.py:L327]`).

**OCR output type** — `OCR_OUTPUT_TYPE` defaults to `"pdfa"` `[src/paperless/settings.py:L518]`, so
OCR produces a separate PDF/A archive.

**Document MIME-type assignment** — taken from the **input** file, before parsing, in the consumer
`[src/documents/consumer.py:L219]`:

```python
mime_type = magic.from_file(self.path, mime=True)
```

This MIME type is what is stored for the document; OCR produces a separate PDF/A *archive* file and
does not redefine the document's MIME type.

### 4.2 Observed behavior

The OCR parser tests in `src/paperless_tesseract/tests/test_parser.py` exercise `parse()` with the
MIME type `application/pdf`, including the no-text and force-OCR paths. Representative methods run
serially:

```
$ pytest paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
         paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_force \
         paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
         -n0 --no-cov
3 passed in 17.20s
```

These pass and exercise the `ocrmypdf.ocr` primary/fallback paths and the no-text handling described
above.

### 4.3 Rationale / Conclusion

- The invoked subprocess is **`ocrmypdf.ocr(...)`** — primary at
  `[src/paperless_tesseract/parsers.py:L261]`, force-OCR fallback at
  `[src/paperless_tesseract/parsers.py:L298]`.
- The OCR archive is **PDF/A** (`OCR_OUTPUT_TYPE="pdfa"` `[src/paperless/settings.py:L518]`).
- The **document's MIME type is assigned from the input** via `magic.from_file(self.path, mime=True)`
  `[src/documents/consumer.py:L219]` — commonly `application/pdf` — and is **not** redefined by OCR.
- For a truly text-free document, the pipeline yields empty text (`self.text = ""`
  `[src/paperless_tesseract/parsers.py:L327]`) while still producing the PDF/A archive.


---

## 5. Q4 — Barcode splitting: document count, trigger values, decision site

**Short answer:** A single multi-page input yields **K+1 documents for K separator pages** (one
separator → **2** documents). The default trigger barcode value is **`"PATCHT"`**. The split
**decision** is made in `scan_file_for_separating_barcodes()` (the page is flagged when the
configured barcode string is found on it); the actual page partitioning — which **skips the
separator page itself** — happens in `separate_pages()`.

### 5.1 Evidence

**Decision site.** `scan_file_for_separating_barcodes(filepath)`
`[src/documents/tasks.py:L96-L110]` renders each page (`convert_from_path`), reads its barcodes with
`barcode_reader` `[src/documents/tasks.py:L75-L93]`, and flags a page when the configured separator
string appears among the decoded barcodes `[src/documents/tasks.py:L108]`:

```python
separator_barcode = str(settings.CONSUMER_BARCODE_STRING)          # L102
...
    current_barcodes = barcode_reader(page)
    if separator_barcode in current_barcodes:                       # L108  <-- decision
        separator_page_numbers.append(current_page_number)
```

**Trigger value.** `CONSUMER_BARCODE_STRING` defaults to **`"PATCHT"`**
`[src/paperless/settings.py:L506]`, and the entire feature is gated by `CONSUMER_ENABLE_BARCODES`
`[src/paperless/settings.py:L502]`.

**Split mechanics.** `separate_pages(filepath, pages_to_split_on)`
`[src/documents/tasks.py:L113-L161]` builds the first document from the pages **before** the first
separator `[src/documents/tasks.py:L130-L138]`, then one document per subsequent separator, and
**skips the separator (barcode) page itself** `[src/documents/tasks.py:L148-L149]`:

```python
# skip the first page_number. This contains the barcode page
for page in range(page_number + 1, next_page):
    dst.pages.append(pdf.pages[page])
```

**Readable symbologies (from tests).** The barcode reader handles multiple symbologies, all encoding
the `PATCHT` value: **Code 39** (`barcode-39-PATCHT.png` `[src/documents/tests/test_tasks.py:L101]`),
**QR** (`qr-code-PATCHT.png` `[src/documents/tests/test_tasks.py:L155]`), and **Code 128**
(`barcode-128-PATCHT.png` `[src/documents/tests/test_tasks.py:L166]`).

### 5.2 Observed behavior

`test_separate_pages` `[src/documents/tests/test_tasks.py:L305-L313]` runs one separator and asserts
two resulting documents:

```python
pages = tasks.separate_pages(test_file, [1])   # L312  (one separator at page index 1)
self.assertEqual(len(pages), 2)                # L313
```

Run serially together with several barcode-reader tests:

```
$ pytest documents/tests/test_tasks.py::TestTasks::test_separate_pages \
         documents/tests/test_tasks.py::TestTasks::test_barcode_reader \
         documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr \
         documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 \
         -n0 --no-cov
4 passed in 1.52s
```

This confirms one separator → two documents, and that Code 39 / QR / Code 128 barcodes are all
decoded to the `PATCHT` separator value.

### 5.3 Rationale / Conclusion

| Separator pages (K) | Resulting documents |
|---------------------|---------------------|
| 0 | 1 (no split) |
| 1 | 2 |
| 2 | 3 |
| K | **K + 1** |

- A single multi-page input yields **K+1 documents for K separator pages** (one separator → **2**).
- The **trigger barcode value is `"PATCHT"`** by default `[src/paperless/settings.py:L506]`.
- The **split decision is made in `scan_file_for_separating_barcodes()`** at
  `[src/documents/tasks.py:L108]`; the partitioning that skips the barcode page is in
  `separate_pages()` at `[src/documents/tasks.py:L148-L149]`.

---

## 6. Q5 — Effect of barcode splitting on training-set size

**Short answer:** **Yes.** Splitting one input into N parts increases the effective (non-inbox)
training-set size by **N — not by 1**. The original input produces **no** `Document` in the splitting
call; instead its N split files are written back to the consumption directory and re-consumed
independently, each becoming a `Document` that the classifier/auto-matching then sees.

### 6.1 Evidence

`consume_file()` `[src/documents/tasks.py:L184-L252]` enters the barcode branch when
`settings.CONSUMER_ENABLE_BARCODES` is true `[src/documents/tasks.py:L195]`. It scans the file
`[src/documents/tasks.py:L198]`, and if separators are found it calls `separate_pages`
`[src/documents/tasks.py:L201]`, writes **each** split PDF to the consumption directory via
`save_to_dir` `[src/documents/tasks.py:L203-L210]`, deletes the original with `os.unlink(path)`
`[src/documents/tasks.py:L213-L214]`, and **returns `"File successfully split"`**
`[src/documents/tasks.py:L233]`:

```python
if document_list:
    for n, document in enumerate(document_list):
        ...
        save_to_dir(document, newname=newname)        # L210
    os.unlink(path)                                    # L214  (delete original)
    ...
    return "File successfully split"                   # L233  -> NO Document created here
```

> The key point: in this branch **no `Document` record is created**. The function returns early
> after saving the split files to the consumption directory. The N split files are then **picked up
> and consumed independently**, each producing **one** `Document`.

The classifier trains over the full document set minus inbox-tagged ones
`[src/documents/classifier.py:L125-L127]`:

```python
for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True):
    ...
```

### 6.2 Rationale / Conclusion

Because the original input yields no `Document` in the splitting call but its **N split files each
become a `Document`** upon re-consumption, splitting one input into N parts increases the effective
(non-inbox) training-set size by **N**, not by 1. This changes the data the classifier/auto-matching
sees during a run. It also connects back to **Q1**: more (and different) documents change the SHA-1
`data_hash` `[src/documents/classifier.py:L124-L127]`, which is precisely what causes `train()` to
**stop short-circuiting and retrain** `[src/documents/classifier.py:L161-L164]`. A larger,
freshly-retrained set then amplifies the seedless-estimator non-determinism discussed in Section 7.


---

## 7. Non-Determinism Root Causes (Synthesis) & Recommendations

Pulling the threads from Q1–Q5 together, the flakiness of the document-classification tests is the
product of **four interacting facts**. Cause #1 is the prime suspect; the others are amplifiers and
exposure mechanisms.

### 7.1 Cause #1 (prime suspect) — the estimators have no `random_state`

All three `MLPClassifier` instances are created with **only `tol=0.01` and no `random_state`**:

- tags classifier — `MLPClassifier(tol=0.01)` `[src/documents/classifier.py:L219]`
- correspondent classifier — `MLPClassifier(tol=0.01)` `[src/documents/classifier.py:L227]`
- document-type classifier — `MLPClassifier(tol=0.01)` `[src/documents/classifier.py:L238]`

scikit-learn's `MLPClassifier` uses `random_state` to control weight/bias initialization and the
stochastic optimizer; the documentation explains that `random_state` determines random number
generation for weight and bias initialization and that an int should be passed for reproducible
results across calls. Without it, each `fit` starts from different random weights and follows a
different stochastic optimization path, so the **fitted model differs run-to-run**.

This was confirmed directly with an isolated probe reproducing the exact pipeline
(`CountVectorizer(analyzer="word", ngram_range=(1,2), min_df=0.01)` + `MLPClassifier(tol=0.01)`) on
the two effective documents from `generate_test_data()`:

```
distinct fitted-weight signatures across 8 identical fits: 8   -> weights differ on every fit
```

Critically, acceptance uses a **hard predicted class id with no probability threshold**
`[src/documents/classifier.py:L254-L258]` (Q2c). With no probability cushion to absorb small
variations, any run whose stochastic training nudges a borderline document across the decision
boundary flips a hard-equality assertion such as `predict_correspondent(doc2) == None`
`[src/documents/tests/test_classifier.py:L122]`. For the tiny, well-separated 2-document fixture the
hard prediction happens to stay stable in isolation, but the latent risk is intrinsic and grows with
training-set size and class overlap.

### 7.2 Cause #2 — persisted/shared on-disk model + data-hash reuse

The model is persisted to `MODEL_FILE` `[src/paperless/settings.py:L74]`, and `train()` short-circuits
when the SHA-1 `data_hash` is unchanged `[src/documents/classifier.py:L163-L164]` (Q1). A stale or
shared model file whose hash matches the current data therefore causes `train()` to **skip retraining
and reuse previously fitted weights**, leaking model state into later tests. Whether a given test
sees a fresh model or an inherited one becomes order- and environment-dependent.

### 7.3 Cause #3 — uneven test isolation

The base mixin provides good isolation, but two specific overrides undermine it:

- **Good (default):** `DirectoriesMixin` `[src/documents/tests/utils.py:L72-L83]` calls
  `setup_directories()` in `setUp` `[src/documents/tests/utils.py:L77-L79]`, which points `MODEL_FILE`
  at a **fresh per-test temp directory** `[src/documents/tests/utils.py:L45]` (inside the
  `override_settings` block `[src/documents/tests/utils.py:L35-L47]`).
- **Gap A — shared checked-in fixture:** `test_load_and_classify` overrides `MODEL_FILE` to a
  **checked-in shared fixture** `documents/tests/data/model.pickle`
  `[src/documents/tests/test_classifier.py:L180-L181]`, so that test deliberately reads a static
  model on disk.
- **Gap B — import-time temp dir:** `testSaveClassifier` uses
  `@override_settings(DATA_DIR=tempfile.mkdtemp())` `[src/documents/tests/test_classifier.py:L167]`.
  The `tempfile.mkdtemp()` is evaluated **once, at import/decoration time**, so every invocation of
  that test shares the **same** directory rather than getting a fresh one.

These inconsistencies mean the on-disk model location is not uniformly per-test isolated.

### 7.4 Cause #4 — parallel execution exposes shared-file coupling

The suite runs under `--numprocesses auto` (pytest-xdist) `[src/setup.cfg:L10]`. Parallelism does not
*introduce* randomness; it **exposes** hidden coupling — most commonly a shared file on disk such as
the persisted model — by letting differently-scheduled workers interleave reads/writes. This aligns
with widely-documented pytest-xdist best practice: isolate per-test/per-worker state (e.g.,
worker-scoped temp paths) so that shared on-disk resources cannot couple otherwise-independent tests.

### 7.5 How the causes combine

```
no random_state (C1)  ->  fitted weights differ every run
        +
hard class id, no probability threshold (Q2c)  ->  small variations flip assertions
        +
persisted MODEL_FILE + data-hash reuse (C2)  ->  some tests inherit a stale model
        +
uneven isolation (C3)  ->  which tests inherit it is inconsistent
        +
--numprocesses auto (C4)  ->  parallel scheduling exposes the coupling
        =
intermittent, order-/schedule-dependent failures (flaky tests)
```

Barcode splitting (Q5) is an aggravating input: by turning one input into N documents it enlarges and
changes the training set, which both alters the `data_hash` (forcing retrains, C2) and increases the
chance that the seedless estimator (C1) produces a class assignment that flips a hard assertion.

### 7.6 Recommendations (DOCUMENTATION-ONLY — intentionally NOT implemented)

> Per the scope of this task, the following are **recommendations only**. No source code was changed;
> these are **not** applied in the repository.

1. **Seed the estimators.** Pass an explicit integer `random_state` to each `MLPClassifier(...)`
   `[src/documents/classifier.py:L219, L227, L238]`. This is the single highest-leverage fix: it makes
   training reproducible and directly removes the prime suspect (C1).
2. **Make per-test model isolation uniform.** Ensure every test uses a fresh, per-test `MODEL_FILE`.
   Avoid the import-time-evaluated `tempfile.mkdtemp()` decorator pattern
   `[src/documents/tests/test_classifier.py:L167]` (evaluate the temp dir per test, e.g. in `setUp`),
   and treat the shared checked-in fixture `[src/documents/tests/test_classifier.py:L180-L181]` as
   read-only so it cannot be mutated or inherited (addresses C2/C3).
3. **Optionally seed numpy** at test-session start for defense-in-depth against any other
   unseeded stochastic paths.
4. **Worker-scoped isolation under xdist.** If any on-disk artifact must be shared, scope it per
   worker so parallel scheduling cannot couple tests (addresses C4).

Applying #1 alone would most likely eliminate the observed flakiness; #2–#4 harden the suite against
recurrence.


---

## 8. References

### 8.1 Code citations (this branch)

**`src/documents/classifier.py`**
- `load_classifier()` — `[L30-L57]`
- `DocumentClassifier.load()` (restores `data_hash`) — `[L76-L94]` (hash restore at `[L86]`)
- `save()` — `[L96-L113]`
- training query (non-inbox, ordered by pk) — `[L124-L127]`
- data-hash short-circuit (`return False` when unchanged) — `[L161-L164]`
- `MLPClassifier(tol=0.01)` instances (no `random_state`) — tags `[L219]`, correspondent `[L227]`,
  document type `[L238]`
- `self.data_hash = new_data_hash` / `return True` — `[L247]`, `[L249]`
- `predict_correspondent` (hard class id, reject only on `-1`) — `[L251-L260]` (predict at `[L254]`,
  accept/reject at `[L255-L258]`)

**`src/documents/tasks.py`**
- `train_classifier` (save only when `train()` is `True`) — `[L48-L72]` (load `[L57]`, save branch
  `[L63-L67]`, "Training data unchanged." `[L68-L69]`)
- `barcode_reader` — `[L75-L93]`
- `scan_file_for_separating_barcodes` (split **decision**) — `[L96-L110]` (decision at `[L108]`)
- `separate_pages` (skips the barcode page) — `[L113-L161]` (skip at `[L148-L149]`)
- `consume_file` barcode branch (returns "File successfully split", no `Document` created) —
  `[L184-L252]` (enable check `[L195]`, return `[L233]`)

**`src/documents/consumer.py`** — input MIME via `magic.from_file(self.path, mime=True)` — `[L219]`

**`src/documents/matching.py`** — `match_correspondents` `[L21-L31]`; six algorithms `MATCH_ALL`
`[L72]`, `MATCH_ANY` `[L84]`, `MATCH_LITERAL` `[L91]`, `MATCH_REGEX` `[L107]`, `MATCH_FUZZY` `[L127]`,
`MATCH_AUTO` `[L147]`; fuzzy threshold `>= 90` — `[L135]`

**`src/documents/signals/handlers.py`** — `match_correspondents` `[L50]`, `match_document_types`
`[L116]`, `match_tags` `[L189]`

**`src/paperless_tesseract/parsers.py`** — `parse()` `[L230]`; `len > 50` text check `[L236]`;
`skip_noarchive` early-return `[L241]`; primary `ocrmypdf.ocr` `[L261]`; `NoTextFoundException` raise
`[L266-L267]`; fallback `except` `[L276]`; `safe_fallback=True` `[L293]`; fallback `ocrmypdf.ocr`
`[L298]`; last-resort `self.text = ""` `[L318-L327]` (`[L327]`)

**`src/paperless/settings.py`** — `MODEL_FILE` `[L74]`; `CONSUMER_ENABLE_BARCODES` `[L502]`;
`CONSUMER_BARCODE_STRING` default `"PATCHT"` `[L506]`; `OCR_OUTPUT_TYPE` default `"pdfa"` `[L518]`;
`OCR_MODE` default `"skip"` `[L522]`

**`src/documents/tests/test_classifier.py`** — `generate_test_data` `[L25-L82]`; `t2` inbox tag
`[L40-L45]`/`[L82]`; `testTrain` `[L103-L113]` (classes at `[L108]`); `testPredict` `[L115-L135]`
(asserts at `[L118-L122]`); `testDatasetHashing` `[L137-L142]`; `testSaveClassifier` `[L167-L178]`;
`test_load_and_classify` shared fixture `[L180-L181]`

**`src/documents/tests/test_tasks.py`** — barcode symbology fixtures Code 39 `[L101]`, QR `[L155]`,
Code 128 `[L166]`; `test_separate_pages` `[L305-L313]`

**`src/documents/tests/utils.py`** — `DirectoriesMixin` `[L72-L83]`; `override_settings` block
`[L35-L47]`; per-test `MODEL_FILE` `[L45]`; `setUp` `[L77-L79]`

**`src/setup.cfg`** — `addopts` (incl. `--numprocesses auto`) `[L10]`; `PAPERLESS_DISABLE_DBHANDLER`
`[L12]`

**`Dockerfile`** — `FROM python:3.9-slim-bullseye` `[L18]`

**`requirements.txt` / `Pipfile`** — pinned dependency versions (see §1.4)

### 8.2 External best-practice sources (corroboration only)

These external references corroborate — but do not replace — the code-grounded findings above. They
are paraphrased, not quoted.

1. **scikit-learn `MLPClassifier` documentation** — describes `random_state` as controlling the
   random number generation used for weight and bias initialization (and the stochastic optimizer),
   and recommends passing an integer to obtain reproducible results across runs. This corroborates
   Cause #1: instantiating the estimator without `random_state` makes training non-deterministic by
   design. <https://scikit-learn.org/stable/modules/generated/sklearn.neural_network.MLPClassifier.html>
2. **pytest-xdist test-isolation guidance** — parallel execution does not add randomness; it exposes
   pre-existing coupling such as shared on-disk state, and the recommended remedy is per-test /
   per-worker isolation of such resources. This corroborates Cause #4 (the persisted/shared model
   under `--numprocesses auto`).

### 8.3 Behavioral verification commands (for reproducibility)

All tests were run serially with coverage disabled to avoid leaving artifacts:

```
# Q1 — model reuse / retrain
pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
       documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
       documents/tests/test_classifier.py::TestClassifier::testTrain -n0 --no-cov   # 3 passed

# Q2 — correspondent matching (hard prediction)
pytest documents/tests/test_classifier.py::TestClassifier::testPredict -n0 --no-cov # passes in isolation

# Q3 — no-text / force OCR
pytest paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
       paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_force \
       paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext -n0 --no-cov  # 3 passed

# Q4 — barcode splitting
pytest documents/tests/test_tasks.py::TestTasks::test_separate_pages \
       documents/tests/test_tasks.py::TestTasks::test_barcode_reader \
       documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr \
       documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 -n0 --no-cov  # 4 passed
```

The estimator non-determinism (Cause #1) was demonstrated with an isolated probe (run outside the
repository tree) that fits `MLPClassifier(tol=0.01)` repeatedly on the two effective training
documents and observes a distinct fitted-weight signature on every run.

---

*End of analysis. This document is the sole artifact of the investigation; no source file was
modified.*

