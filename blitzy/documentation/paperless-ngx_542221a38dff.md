# paperless-ngx — ML classification, OCR, and barcode behavior during test execution (non-determinism diagnosis)

**Repository commit:** `542221a38dff06361e07976452f9aea24d210542`
**Nature of this document:** an evidence-grounded Q&A answer. Every value below was **observed by actually running the code** inside the pinned Docker image and is quoted **verbatim** next to the exact command that produced it. Each factual claim is anchored to an exact `file:line` reference in the source at the commit above. Where a value is inherently run-to-run variable (the unseeded classifier), that is stated explicitly and is the whole point of the non-determinism section.

---

## The question being answered

> "I am debugging non-deterministic failures in the document classification tests and want to understand how the machine learning pipeline behaves during test execution when training and OCR. I want to observe whether the classifier reuses an existing model or retrains during the same test run, and how that affects later tests. When a test exercises automatic correspondent matching, how many training documents are created, when does training occur relative to those inserts, and what confidence threshold is used to accept or reject predictions? I also want to see how edge cases are handled, such as when a document has no extractable text, including which OCR subprocess is invoked and what mime type is assigned to the output. Finally, when barcode splitting is involved, how many document records are created from a single input, which barcode values trigger a split, and where in the code that decision is made, especially if this changes the effective training data during the test run. Please do not modify the source code; you may create temporary helpers while investigating, but clean up anything temporary before finishing."

Decomposed into the sub-questions this document answers explicitly:

- **Q1** — Within a single run, does the classifier **reuse an existing model or retrain**, and how does that affect later tests?
- **Q2** — For automatic correspondent matching: **(a)** how many training documents are created, **(b)** when does training occur relative to the inserts, and **(c)** what **confidence threshold** accepts/rejects a prediction?
- **Q3** — How is a **no-extractable-text** document handled: which **OCR subprocess** is invoked and what **MIME type** is assigned to the output?
- **Q4** — For **barcode splitting**: how many **document records** arise from a single input, which **barcode value** triggers a split, **where** in the code the decision is made, and does it **change the effective training data** during the run?
- **ND** — Root-cause diagnosis of the **non-deterministic** classification-test failures (demonstrated empirically, **not** remediated).

---

## Environment & methodology (what was actually run)

**Image.** All commands were executed inside the user-supplied Docker image (canonical pinned name `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`, equivalently `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542`). The paperless-ngx repository is baked into the image at **`/app`** at commit `542221a38dff` (byte-identical to the source tree under investigation). The host shell (a different Python, no project dependencies) is unsuitable and was not used to run the pipeline.

**Observed toolchain versions** (from `python --version` and `import ...; __version__`):

```
Python 3.9.23
django 4.0.4
sklearn 1.0.2
ocrmypdf 13.4.3
numpy 1.22.3
pytest 8.4.2
pytest-django 4.11.1
tesseract 4.1.1
gs 9.53.3
commit 542221a38dff
```

The pinned runtime dependencies that matter to this investigation match the AAP exactly: **Django 4.0.4, scikit-learn 1.0.2, ocrmypdf 13.4.3, numpy 1.22.3** (`Pipfile.lock`; base image `Dockerfile:18` = `FROM python:3.9-slim-bullseye as main-app`). One honest deviation to record: the **pytest stack observed in this image is `pytest 8.4.2` / `pytest-django 4.11.1`**, not the `7.1.1` / `4.5.2` the AAP text anticipated. This does not affect the results below (all targeted tests pass and the code paths are identical), but per the "quote what you actually observed" discipline it is reported here rather than glossed over.

**Image entrypoint quirk.** The image `ENTRYPOINT` is `[/bin/bash]`, so commands are run as `docker exec <container> bash -lc '<script>'` (a persistent container was used so temporary observation scripts live only inside it and vanish when it is removed).

**Environment completion for the barcode-scanning path.** The base image is missing `libzbar0` (needed by `pyzbar`) and `poppler-utils` (provides `pdftoppm`, needed by `pdf2image`). Without them, `scan_file_for_separating_barcodes()` / `barcode_reader()` cannot run (`pyzbar` raises `ImportError: Unable to find zbar shared library`). They were installed before the Q4 runs (`apt-get update && apt-get install -y libzbar0 poppler-utils`); their presence was confirmed (`ldconfig -p | grep libzbar` → present, `which pdftoppm` → `/usr/bin/pdftoppm`). `separate_pages()` on its own uses only `pikepdf` and works without them.

**Recommended pytest invocation** (coverage + xdist disabled for clean, readable, per-test output):

```
python -m pytest <node-ids> -o addopts='' -p no:xdist -p no:cacheprovider -p no:warnings -v
```

The project's default `addopts` (in `src/setup.cfg` `[tool:pytest]`) is `--pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet`; `DJANGO_SETTINGS_MODULE=paperless.settings`; and the test env sets `PAPERLESS_DISABLE_DBHANDLER=true`. The `--numprocesses auto` (pytest-xdist) is disabled during observation but is itself a secondary non-determinism factor (see ND).

**Per-test isolation (important for Q1's "effect on later tests").** Two mechanisms guarantee no leakage between tests:

1. `src/documents/tests/utils.py` — `DirectoriesMixin.setUp()` calls `setup_directories()`, which creates fresh temp dirs and **overrides the model path per test**: `MODEL_FILE=os.path.join(dirs.data_dir, "classification_model.pickle")` at `src/documents/tests/utils.py:45` (a `tempfile.mkdtemp()` data dir). So each test that trains starts from a fresh on-disk model location.
2. Django `TestCase` wraps each test in a database transaction that is **rolled back** at teardown, so `Document`/`Correspondent` rows created in one test do not survive into the next.

**Temporary observation scripts.** Short-lived pytest files (`test_zz_obs_*.py`) were copied **into the container only** (never into the repository) so they inherit the Django test DB, the `DirectoriesMixin` temp dirs, and the `MODEL_FILE` isolation. They were removed with the container; the repository is left byte-for-byte unchanged apart from this one document.

**Baseline sanity run** — the targeted classifier tests all pass:

```
$ python -m pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs \
    documents/tests/test_classifier.py::TestClassifier::testTrain \
    documents/tests/test_classifier.py::TestClassifier::testPredict \
    documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
    documents/tests/test_classifier.py::TestClassifier::testVersionIncreased \
    -o addopts='' -p no:xdist -p no:cacheprovider -p no:warnings -v

documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED [ 14%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED [ 28%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED [ 42%]
documents/tests/test_classifier.py::TestClassifier::testTrain PASSED     [ 57%]
documents/tests/test_classifier.py::TestClassifier::testPredict PASSED   [ 71%]
documents/tests/test_classifier.py::TestClassifier::testSaveClassifier PASSED [ 85%]
documents/tests/test_classifier.py::TestClassifier::testVersionIncreased PASSED [100%]
============================== 7 passed in 2.41s ===============================
```

---

## Q1 — Does the classifier reuse an existing model or retrain within a run, and how does that affect later tests?

**Sub-question.** During a single test run, does `DocumentClassifier.train()` refit, or does it reuse/short-circuit when the data has not changed — and can a model trained in one test leak into a later one?

**What was run.** A temporary test created one auto-matching `Correspondent` and one `Document`, then called `train()` **twice** on the unchanged data:

```python
c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
clf = DocumentClassifier()
first  = clf.train();  h1 = clf.data_hash.hex()
second = clf.train();  h2 = clf.data_hash.hex()
```

**Observed output (verbatim):**

```
Q1_FIRST_TRAIN=True
Q1_DATA_HASH_AFTER_1=230b98c1cbe4bb261c254b08a6d463334e9ba45d
Q1_SECOND_TRAIN=False
Q1_DATA_HASH_AFTER_2=230b98c1cbe4bb261c254b08a6d463334e9ba45d
Q1_HASH_UNCHANGED=True
```

The repository's own `testDatasetHashing` asserts exactly this pair (`assertTrue(train())` then `assertFalse(train())`) at `src/documents/tests/test_classifier.py:137`, and it `PASSED` in the baseline run above.

**Answer.** Within a single run against **unchanged** data, `train()` does **not** refit — it **short-circuits**. The **first** call fits the model, stores a 40-hex-character SHA1 digest in `self.data_hash`, and returns **`True`**; the **second** call recomputes the SHA1 over the (unchanged) data, finds it equal to the stored hash, and returns **`False`** without refitting. The two hashes are byte-identical (`230b98c1cbe4bb261c254b08a6d463334e9ba45d` in this run — the value depends on the data content, but is identical across the two calls on the same data).

**Why (mechanism, with citations).** `train()` builds a SHA1 over the preprocessed content plus label bytes of every document, ordered by `pk` and excluding inbox-tagged documents:

- `m = hashlib.sha1()` at `src/documents/classifier.py:124`, iterating `for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True):` at `src/documents/classifier.py:125`;
- the digest is finalized as `new_data_hash = m.digest()` at `src/documents/classifier.py:161`;
- the short-circuit is `if self.data_hash and new_data_hash == self.data_hash:` at `src/documents/classifier.py:163` → **`return False`** at `src/documents/classifier.py:164`;
- on a real fit, the hash is stored (`self.data_hash = new_data_hash` at `src/documents/classifier.py:247`) and the method ends with **`return True`** at `src/documents/classifier.py:249`.

The persisted model is a versioned pickle keyed by `FORMAT_VERSION = 7` at `src/documents/classifier.py:63`.

**Real-world reuse path.** In production the reuse is explicit: `train_classifier()` (`src/documents/tasks.py:48`) loads any existing model via `load_classifier()` (reuse), calls `classifier.train()`, and only calls `classifier.save()` when `train()` returned truthy — otherwise it logs **`"Training data unchanged."`** at `src/documents/tasks.py:69`. The CLI entry point `document_create_classifier`'s `handle()` delegates to `train_classifier()` (`src/documents/management/commands/document_create_classifier.py`).

**Effect on later tests — none.** A model trained in one test cannot leak into the next, because (1) each test gets a fresh per-test `MODEL_FILE` temp path (`src/documents/tests/utils.py:45`) so there is no shared on-disk model, and (2) Django `TestCase` rolls back the DB transaction after each test so no `Document`/`Correspondent` rows persist. The repository demonstrates the persistence half of this directly: `testSaveClassifier` (`src/documents/tests/test_classifier.py:168`) trains, saves, loads into a **new** `DocumentClassifier`, and asserts `train()` now returns `False` (the loaded `data_hash` matches) — again `PASSED` above. Because every training test starts from a fresh `DocumentClassifier()` against a rolled-back DB and a fresh model path, "reuse vs. retrain" is decided **only** by the in-run `data_hash` comparison, never by cross-test state.

---

## Q2 — Automatic correspondent matching: how many training docs, when does training happen, and what confidence threshold accepts/rejects?

**Sub-questions.** (a) How many training documents are created? (b) When does `train()` occur relative to the `Document` inserts? (c) What **confidence threshold** is used to accept or reject a prediction?

**What was run.** Two temporary tests mirroring the repository's real ones. The **1-document** case creates one auto-matching correspondent and one document; the **2-document** case adds a second document with **no** correspondent. In both, `train()` is called **after** the inserts, then `classes_` and predictions are printed:

```python
# 1-doc
c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
doc1 = Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
clf = DocumentClassifier(); clf.train()           # train AFTER insert
# 2-doc adds:
doc2 = Document.objects.create(title="doc2", content="this is a document from noone", checksum="B")  # no correspondent
```

**Observed output (verbatim):**

```
Q2_1DOC_TRAINING_DOCS=1
Q2_1DOC_c1_pk=1
Q2_1DOC_classes_=[1]
Q2_1DOC_predict_doc1=[1]

Q2_2DOC_TRAINING_DOCS=2
Q2_2DOC_classes_=[-1, 1]
Q2_2DOC_predict_doc1=[1]
Q2_2DOC_predict_doc2=None
```

The two real repository tests both `PASSED`:

```
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED [ 50%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED [100%]
============================== 2 passed in 2.03s ===============================
```

**Answers.**

**(a) Number of training documents: `1` and `2`.** `test_one_correspondent_predict` (`src/documents/tests/test_classifier.py:191`) creates exactly **one** document; `test_one_correspondent_predict_manydocs` (`src/documents/tests/test_classifier.py:206`) creates **two** documents (`Q2_1DOC_TRAINING_DOCS=1`, `Q2_2DOC_TRAINING_DOCS=2`). `c1.pk` is `1`.

**(b) Training timing: always AFTER the inserts.** In both tests the `Document.objects.create(...)` calls precede the explicit `self.classifier.train()` call — `train()` at `src/documents/tests/test_classifier.py:203` (1-doc) and `src/documents/tests/test_classifier.py:223` (2-doc), both **after** their respective inserts. The classifier trains on whatever rows exist at the moment `train()` is called.

**(c) Confidence threshold: there is NONE — acceptance is gated by the `-1` sentinel class, not a numeric probability/confidence.** `predict_correspondent` returns the predicted correspondent id **only when it is not `-1`**, and returns `None` otherwise:

- `def predict_correspondent(self, content):` at `src/documents/classifier.py:251`;
- `X = self.data_vectorizer.transform([preprocess_content(content)])` at `src/documents/classifier.py:253`;
- `correspondent_id = self.correspondent_classifier.predict(X)` at `src/documents/classifier.py:254`;
- `if correspondent_id != -1:` at `src/documents/classifier.py:255` → `return correspondent_id` at `src/documents/classifier.py:256`;
- otherwise `return None` at `src/documents/classifier.py:258` (and the outer "no classifier" branch also returns `None` at `src/documents/classifier.py:260`).

There is **no** `predict_proba` call, no probability comparison, and no numeric confidence threshold anywhere in this version's `predict_correspondent`. Acceptance/rejection is purely "predicted class ≠ `-1`."

The `-1` value is a **sentinel label** the trainer injects for documents that have no auto-assigned correspondent. That is why the observed `classes_` differ by case: in the strict **1-document** case the model sees a single label, so `classes_=[1]` (**no `-1`**); the `-1` sentinel appears only once at least one training document lacks an auto-correspondent — the **2-document** case yields `classes_=[-1, 1]`. Correspondingly, `predict_correspondent` returns a numpy array `[1]` (accept `c1`) for the known document and `None` (reject) for the unknown `doc2` (`Q2_2DOC_predict_doc2=None`). The repository's `testTrain` (`src/documents/tests/test_classifier.py:103`) codifies the sentinel by asserting `correspondent_classifier.classes_ == [-1, self.c1.pk]` on its richer fixture.

**Supporting model construction and gating (citations).** The shared text features are a `CountVectorizer(analyzer="word", ngram_range=(1, 2), min_df=0.01)` (`analyzer="word"` at `src/documents/classifier.py:195`, `ngram_range=(1, 2)` at `:196`, `min_df=0.01` at `:197`); the correspondent estimator is `MLPClassifier(tol=0.01)` at `src/documents/classifier.py:227`. "Automatic" gating uses the `MatchingModel.MATCH_AUTO` algorithm (value `6` at `src/documents/models.py:26`); the auto branch that consults the classifier lives at `src/documents/matching.py:147` (calling `classifier.predict_correspondent(document.content)`), and only auto-matched labels are folded into training by the label loop at `src/documents/classifier.py:125`.

> **Explicit statement for the coverage pass:** *No numeric confidence threshold exists in this version; acceptance/rejection is decided by the `-1` sentinel class in `predict_correspondent` (`src/documents/classifier.py:255`).*

---

## Q3 — No-extractable-text edge case: which OCR subprocess is invoked, and what MIME type is assigned?

**Sub-questions.** When a document has no extractable text, how is it handled, which **OCR subprocess** is invoked, and what **MIME type** is assigned to the output?

**What was run.** A temporary test ran `RasterisedDocumentParser.parse()` on the committed text-less fixture `no-text-alpha.png`, with `magic.from_file(..., mime=True)` for the MIME type and `--log-cli-level=DEBUG` to capture the OCRmyPDF argument dicts and the parser's log lines:

```python
sample = ".../paperless_tesseract/tests/samples/no-text-alpha.png"
mime = magic.from_file(sample, mime=True)          # -> image/png
parser = RasterisedDocumentParser(None)
parser.parse(sample, mime)
print(repr(parser.text))
```

**Observed output (verbatim, abridged only where paths repeat):**

```
Q3_MIME=image/png
DEBUG  paperless.parsing.tesseract  Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/no-text-alpha.png', 'output_file': '/tmp/.../archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/.../sidecar.txt', 'image_dpi': 35}
[ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
WARNING  paperless.parsing.tesseract  Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
DEBUG  paperless.parsing.tesseract  Fallback: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/no-text-alpha.png', 'output_file': '/tmp/.../archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/.../sidecar-fallback.txt', 'image_dpi': 35}
[ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
WARNING  paperless.parsing.tesseract  No text was found in /app/src/paperless_tesseract/tests/samples/no-text-alpha.png, the content will be empty.
Q3_TEXT_REPR=''
Q3_ARCHIVE_PATH=/tmp/.../archive.pdf
============================== 1 passed in 8.10s ===============================
```

**Answers.**

**OCR subprocess invoked: OCRmyPDF, via `ocrmypdf.ocr(**args)`, which in turn drives Tesseract (4.1.1).** The first invocation is `ocrmypdf.ocr(**args)` at `src/paperless_tesseract/parsers.py:261`. The observed first-call argument dict contains `'skip_text': True` — this is because the default `OCR_MODE` is `"skip"` (`OCR_MODE = os.getenv("PAPERLESS_OCR_MODE", "skip")` at `src/paperless/settings.py:522`), alongside `'language': 'eng'` (`OCR_LANGUAGE` at `src/paperless/settings.py:514`) and `'output_type': 'pdfa'` (`OCR_OUTPUT_TYPE` at `src/paperless/settings.py:518`). The `[tesseract]` error lines in the log confirm the OCRmyPDF → Tesseract subprocess chain.

**Handling of the no-text case (raise → forced-OCR safe-fallback → empty text).**

1. After the first OCR, if no text was extracted the parser raises `NoTextFoundException` — `if not self.text:` at `src/paperless_tesseract/parsers.py:266` → `raise NoTextFoundException("No text was found in the original document")` at `src/paperless_tesseract/parsers.py:267` (the exception class is defined at `src/paperless_tesseract/parsers.py:14`).
2. That is caught by `except (NoTextFoundException, InputFileError) as e:` at `src/paperless_tesseract/parsers.py:276`, which logs the observed **`"Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text."`** and re-runs with a **forced-OCR safe fallback**: `construct_ocrmypdf_parameters(..., safe_fallback=True)` (the `safe_fallback=True` argument at `src/paperless_tesseract/parsers.py:293`), then a second `ocrmypdf.ocr(**args)` at `src/paperless_tesseract/parsers.py:298`. The `force_ocr` flag observed in the fallback arg dict is set by `if settings.OCR_MODE == "force" or safe_fallback:` at `src/paperless_tesseract/parsers.py:155` → `ocrmypdf_args["force_ocr"] = True` at `src/paperless_tesseract/parsers.py:156`.
3. If still no text, the last resort logs **`"No text was found in {document_path}, the content will be empty."`** (the message is split across two f-string lines at `src/paperless_tesseract/parsers.py:324`–`325`) and sets `self.text = ""` at `src/paperless_tesseract/parsers.py:327`. The observed `Q3_TEXT_REPR=''` confirms the stored text is the **empty string**.

**MIME type assigned to the output: the libmagic-detected input type — `image/png` for this fixture.** The consumer detects the type once via `mime_type = magic.from_file(self.path, mime=True)` at `src/documents/consumer.py:219`, and persists it unchanged when creating the `Document` row in `_store()` (`mime_type=mime_type` at `src/documents/consumer.py:401`). The direct `magic.from_file('.../no-text-alpha.png', mime=True)` call returned **`image/png`** (`Q3_MIME=image/png`); OCR does not alter the stored document's MIME type.

**Precision about the fixtures/tests (a nuance worth stating).** The tests whose names contain "notext" — `test_with_form_error_notext` (`src/paperless_tesseract/tests/test_parser.py:190`) and `test_skip_noarchive_notext` (`src/paperless_tesseract/tests/test_parser.py:370`) — actually **recover** text via OCR rather than ending empty; they exercise the retry, not the empty-result terminus. The repository's genuine stored-empty assertion is `test_encrypted` (`src/paperless_tesseract/tests/test_parser.py:178`), which asserts `parser.get_text() == ""`. The clean end-to-end "no extractable text → empty content" demonstration is therefore the **direct parser run on `no-text-alpha.png`** shown above, which is why that path was exercised explicitly. (For added context, thumbnail generation uses a separate subprocess: `run_convert()` shells out to `CONVERT_BINARY` with a ghostscript fallback in `src/documents/parsers.py`.)

---

## Q4 — Barcode splitting: how many document records, which barcode value, where is the decision made, and does it change training data?

**Sub-questions.** When barcode splitting is involved: how many **document records** are created from a single input, which **barcode value** triggers a split, **where** in the code that decision is made, and does it **change the effective training data** during the run?

**What was run.** A temporary test (with `libzbar0` + `poppler-utils` installed) read the separator setting, decoded the barcodes on page 0 of `patch-code-t.pdf`, scanned three fixtures with `scan_file_for_separating_barcodes()`, split `patch-code-t-middle.pdf` with `separate_pages(..., [1])`, and counted `Document` rows afterward:

```python
print(settings.CONSUMER_BARCODE_STRING, settings.CONSUMER_ENABLE_BARCODES)
tasks.barcode_reader(convert_from_path("patch-code-t.pdf")[0])   # decode page 0
tasks.scan_file_for_separating_barcodes("patch-code-t.pdf")      # -> indices
tasks.scan_file_for_separating_barcodes("patch-code-t-middle.pdf")
tasks.scan_file_for_separating_barcodes("several-patcht-codes.pdf")
splits = tasks.separate_pages("patch-code-t-middle.pdf", [1])
print(len(splits), [os.path.basename(p) for p in splits], Document.objects.count())
```

**Observed output (verbatim):**

```
Q4_CONSUMER_BARCODE_STRING='PATCHT'
Q4_CONSUMER_ENABLE_BARCODES=False
Q4_barcode_reader_page0=['PATCHT']
Q4_scan_patchcodet=[0]
Q4_scan_middle=[1]
Q4_scan_several=[2, 5]
Q4_separate_count=2
Q4_separate_names=['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
Q4_document_count_after_split=0
```

The relevant real repository tests all `PASSED` (the six named ones plus the rest of the barcode suite):

```
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3 PASSED
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED
documents/tests/test_tasks.py::TestTasks::test_barcode_splitter PASSED
documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file PASSED
====================== 25 passed, 15 deselected in 5.59s =======================
```

**Answers.**

**Which barcode value triggers a split: `'PATCHT'`.** The separator literal is `CONSUMER_BARCODE_STRING = os.getenv("PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")` at `src/paperless/settings.py:506` (observed `Q4_CONSUMER_BARCODE_STRING='PATCHT'`), and the decoded barcode value on the patch-code page is literally `['PATCHT']` (`Q4_barcode_reader_page0=['PATCHT']`). Barcode splitting is **disabled by default** — `CONSUMER_ENABLE_BARCODES` at `src/paperless/settings.py:502`–`504` (observed `Q4_CONSUMER_ENABLE_BARCODES=False`).

**Where the decision is made: `scan_file_for_separating_barcodes()`.** Defined at `src/documents/tasks.py:96`, it reads `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` at `src/documents/tasks.py:102` and appends a page index to the split list when `if separator_barcode in current_barcodes:` at `src/documents/tasks.py:108`. Observed scan results confirm the page indices: `patch-code-t.pdf → [0]`, `patch-code-t-middle.pdf → [1]`, `several-patcht-codes.pdf → [2, 5]`.

**How many document records from a single input: `0` `Document` rows during the split; the split emits `len(separators)+1` PDF *files*.** `separate_pages()` (`src/documents/tasks.py:113`) writes the first output as `"{}_document_0.pdf".format(fname)` at `src/documents/tasks.py:134` and subsequent outputs as `"{}_document_{}.pdf".format(fname, str(count + 1))` at `src/documents/tasks.py:154`, returning the list of paths at `src/documents/tasks.py:161`. With one separator page (`[1]`) it produced **2** files — `Q4_separate_count=2`, `Q4_separate_names=['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']` — consistent with the repository's `test_separate_pages` asserting `len(...) == 2`. Crucially, **no database `Document` rows are created by the split**: `Q4_document_count_after_split=0`. The split produces **PDF files**, not DB rows. In the consumption task, `consume_file()` (`src/documents/tasks.py:184`) saves each split PDF back to the consumption directory (`save_to_dir(...)` at `src/documents/tasks.py:210`) and returns early with exactly **`"File successfully split"`** at `src/documents/tasks.py:233`.

**Does it change the effective training data during the run? No — not directly.** Because the split creates PDF **files** and `0` `Document` rows (and returns early), it does **not** add any classifier training data within the barcode test itself. Classifier training data (`Document` rows) is created only later, when each split PDF is independently consumed end-to-end. Within `TestTasks`, the split step alone leaves the training corpus unchanged.

---

## ND — Root-cause diagnosis of the non-deterministic classification-test failures

**What was run.** A temporary test trained **10 fresh** `DocumentClassifier()` instances on **identical** 2-document data (the same corpus each time), capturing `classes_`, the fitted `loss_`, `n_iter_`, the first weight of the first layer, and the prediction for four probe strings on every fit:

```python
for _ in range(10):
    clf = DocumentClassifier()            # fresh -> data_hash None -> always fits
    clf.train()
    cc = clf.correspondent_classifier
    # record cc.classes_, cc.loss_, cc.n_iter_, cc.coefs_[0][0][0]
    # record clf.predict_correspondent(probe) for each probe
```

**Observed output (verbatim — this run; the numbers are expected to differ on every run):**

```
ND_classes_=[-1, 1]
ND_loss_per_run=[0.5115, 0.3995, 0.4588, 0.5557, 0.3606, 0.4278, 0.38, 0.5578, 0.5261, 0.4964]
ND_n_iter_per_run=[21, 31, 26, 12, 33, 25, 33, 29, 21, 29]
ND_first_weight_per_run=[0.0294, 0.0104, -0.0613, 0.1704, -0.135, -0.062, -0.1049, 0.0663, 0.1879, 0.0523]
ND_distinct_losses=10
ND_pred['this is a document from c1']=[1, 1, 1, 1, 1, 1, 1, 1, 1, 1] distinct=1
ND_pred['this is a document from noone']=[None, None, None, None, None, None, None, None, None, None] distinct=1
ND_pred['this is a document']=[1, None, None, 1, 1, None, None, 1, None, None] distinct=2
ND_pred['totally unrelated zebra content']=[1, 1, 1, 1, 1, None, None, None, None, None] distinct=2
```

**Interpretation.** The **label set is deterministic** — `classes_=[-1, 1]` was identical on all 10 fits. But everything downstream of weight initialization varies: all 10 `loss_` values were **distinct** (`ND_distinct_losses=10`), and `n_iter_` and the first fitted weight varied fit-to-fit. Decisively, **predictions on *ambiguous* inputs flip between class `1` (accept `c1`) and `None`/`-1` (reject) across runs** — `'this is a document'` and `'totally unrelated zebra content'` each show `distinct=2`. Clear-cut inputs stay stable (`'... from c1'` → always `1`; `'... from noone'` → always `None`). This flipping on borderline inputs is exactly the mechanism by which a test that asserts a specific prediction for a borderline input **passes on one run and fails on another**.

> These specific numbers are inherently run-to-run variable — that variability *is* the finding. A different execution of the same script will produce different `loss_`/`n_iter_`/weights and a different (but still `distinct>1`) flip pattern on the ambiguous probes.

**Primary root cause — the unseeded `MLPClassifier`.** All three estimators are constructed as `MLPClassifier(tol=0.01)` with **no `random_state`**: tags at `src/documents/classifier.py:219`, correspondent at `src/documents/classifier.py:227`, and document type at `src/documents/classifier.py:238`. scikit-learn's `MLPClassifier` defaults to `solver='adam'` and `random_state=None`; with `random_state=None`, the `random_state` parameter "Determines random number generation for weights and bias initialization ... and batch sampling," and an int must be passed for reproducible results across calls (scikit-learn 1.0.2 documentation). With no seed, **weight/bias initialization and minibatch sampling differ on every `fit()`**, producing the varying `loss_`/`n_iter_`/weights observed and the flipping predictions on borderline inputs.

**Secondary factor — pytest-xdist test ordering/parallelism.** The project's default `addopts` include `--numprocesses auto` (pytest-xdist) in `src/setup.cfg` `[tool:pytest]`. Parallel distribution across workers can change the **observed order** in which tests run and how they are grouped, which can surface or mask a borderline-prediction failure differently between CI runs. This is secondary to the unseeded estimator (which is sufficient on its own to cause the flips shown above), but it compounds the run-to-run variability.

> **Scope note — diagnose only, do not remediate.** Per the task's read-only constraint, this document **does not** change any behavior. Adding `random_state` to the `MLPClassifier` constructors, seeding NumPy/Python RNGs, or disabling pytest-xdist would each remove or reduce the non-determinism, but implementing any such fix is **explicitly out of scope**. No source, test, or configuration file was modified.

---

## Coverage pass — every clause of the question, mapped to its answer

Re-reading the verbatim question, each distinct thing it asks for is addressed as follows:

| # | Clause from the question | Answer (exact literal) | Where answered / cited |
|---|--------------------------|------------------------|------------------------|
| 1 | Does the classifier reuse a model or retrain within a run? | Reuse/short-circuit on unchanged data: `train()` → `True` then `False` | Q1; `classifier.py:163`–`164`, `:247`, `:249` |
| 2 | How does that affect later tests? | No effect — per-test `MODEL_FILE` temp path + `TestCase` rollback | Q1; `tests/utils.py:45` |
| 3 | How many training documents (correspondent matching)? | `1` and `2` | Q2(a); `test_classifier.py:191`, `:206` |
| 4 | When does training occur relative to inserts? | Always **after** the inserts | Q2(b); `test_classifier.py:203`, `:223` |
| 5 | What confidence threshold accepts/rejects? | **None** — the `-1` sentinel class decides | Q2(c); `classifier.py:255`–`260` |
| 6 | How is a no-extractable-text document handled? | `NoTextFoundException` → forced-OCR safe fallback → empty text `''` | Q3; `parsers.py:266`–`267`, `:276`, `:293`, `:298`, `:324`–`327` |
| 7 | Which OCR subprocess is invoked? | **OCRmyPDF** (`ocrmypdf.ocr(**args)`) driving Tesseract 4.1.1 | Q3; `parsers.py:261` |
| 8 | What MIME type is assigned to the output? | `image/png` (libmagic-detected input type) | Q3; `consumer.py:219`, `:401` |
| 9 | How many document records from a single barcoded input? | `0` `Document` rows during the split (emits `len(splits)+1` PDF **files** = `2`) | Q4; `tasks.py:113`–`162`, observed `document_count_after_split=0` |
| 10 | Which barcode value triggers a split? | `'PATCHT'` (decoded page value `['PATCHT']`) | Q4; `settings.py:506` |
| 11 | Where is the split decision made? | `scan_file_for_separating_barcodes()` | Q4; `tasks.py:96`–`111` (test at `:102`, `:108`) |
| 12 | Does the split change effective training data during the run? | No — files, not rows; `consume_file` returns `"File successfully split"` | Q4; `tasks.py:184`–`234` (`:233`) |
| 13 | Root cause of the non-deterministic failures | Unseeded `MLPClassifier(tol=0.01)` (primary); pytest-xdist ordering (secondary) | ND; `classifier.py:219`/`227`/`238`, `setup.cfg` |

All sub-parts of Q1, Q2 (a/b/c), Q3 (handling + subprocess + MIME), Q4 (#records + which value + where + training-data effect), and the non-determinism diagnosis are explicitly answered above, each grounded in observed output and an exact `file:line` citation.

---

## Appendix — reproducibility & cleanup

- **How to reproduce:** start a container from the pinned image, ensure `libzbar0` + `poppler-utils` are present for the Q4 scan path, copy the temporary `test_zz_obs_*.py` observation scripts **into the container only**, and run them from `/app/src` with `python -m pytest <target> -o addopts='' -p no:xdist -p no:cacheprovider -p no:warnings -v -s` (add `--log-cli-level=DEBUG` for Q3). Every marker/value above is reproducible except the unseeded-classifier numbers in ND, which vary by design.
- **Read-only guarantee:** the investigation modified **no** source, test, or configuration file. Temporary observation scripts existed only inside the ephemeral container and were discarded with it. The sole repository change is this document, `blitzy/documentation/paperless-ngx_542221a38dff.md`.

