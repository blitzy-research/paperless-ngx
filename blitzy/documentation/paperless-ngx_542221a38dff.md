# paperless-ngx — ML Classification, OCR & Barcode Behavior During Test Execution

**Branch:** `paperless-ngx_542221a38dff`  **Commit:** `542221a38dff06361e07976452f9aea24d210542`

This document answers four questions about how the paperless-ngx machine-learning
classification pipeline behaves while its test suite runs, in service of debugging
**non-deterministic** failures in the document-classification tests. Every behavioral
claim below is authored **from actually observed runtime output** (the *run-first*
methodology), captured inside the canonical container. Every factual claim carries a
`file:line` reference or the verbatim command output that produced it. Anything that
could only be inferred from reading (not executed) is explicitly labelled **inferred**.

> **Scope note.** This was a read-only investigation. No source file was modified,
> created, or deleted; the only artifact written is this document. All observation
> scripts were temporary, lived outside the repository (`/tmp` inside the container),
> and were removed afterward. See the final "Repository-unchanged proof" section.

---

## Table of Contents

1. [Environment / Reproduction](#1-environment--reproduction)
2. [Q1 — Classifier reuse vs. retrain within a run, and cross-test effect](#q1)
3. [Q2 — Automatic correspondent matching: count, timing, confidence threshold](#q2)
4. [Q3 — Documents with no extractable text: OCR subprocess & mime type](#q3)
5. [Q4 — Barcode splitting: record count, trigger values, decision site, training-data effect](#q4)
6. [Non-Determinism — the root cause of the flaky failures](#nd)
7. [Final Coverage Checklist](#checklist)
8. [Repository-unchanged proof & temp cleanup](#proof)

---

## 1. Environment / Reproduction

All runs execute inside the **canonical container** mandated by the task
(`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`,
alias `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff…`).
The repository is baked at `/app` (owned by the non-root `testuser`, matching CI). The
bare sandbox host lacks Tesseract/Ghostscript/poppler/libzbar, so the **Q3 (OCR)** and
**Q4 (barcode)** code paths were exercised **inside the container**.

The running container used for every run below is `paperless-qna-baked`. Commands were
issued as:

```bash
docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && <command>'
```

### 1.1 Toolchain (verbatim)

```text
$ python --version           ->  Python 3.9.23
$ tesseract --version | head -1  ->  tesseract 4.1.1
$ gs --version              ->  9.53.3
$ pdftoppm -v  (poppler)    ->  pdftoppm version 20.09.0
$ python -c "import pyzbar; print(pyzbar.__version__)"       ->  0.1.9
$ python -c "import sklearn; print(sklearn.__version__)"     ->  1.0.2
```

### 1.2 Pinned dependency versions (verbatim `pip show`)

```text
$ pip show scikit-learn ocrmypdf pyzbar pdf2image pikepdf python-magic | grep -E "^(Name|Version):"
Name: scikit-learn
Version: 1.0.2
Name: ocrmypdf
Version: 13.4.3
Name: pyzbar
Version: 0.1.9
Name: pdf2image
Version: 1.16.0
Name: pikepdf
Version: 5.1.1
Name: python-magic
Version: 0.4.25
```

These match the `requirements.txt` pins exactly (scikit-learn 1.0.2, ocrmypdf 13.4.3,
pyzbar 0.1.9, pdf2image 1.16.0, pikepdf 5.1.1, python-magic 0.4.25, Django 4.0.4,
django-q 1.3.9, numpy 1.22.3, scipy 1.8.0, fuzzywuzzy 0.18.0).

### 1.3 pytest configuration — `src/setup.cfg` (verbatim)

```ini
[tool:pytest]
DJANGO_SETTINGS_MODULE=paperless.settings
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
env =
  PAPERLESS_DISABLE_DBHANDLER=true
```

The `--numprocesses auto` flag is **pytest-xdist** parallelism (per-worker processes,
each with its own database). This is essential grounding for Q1's cross-test answer.

### 1.4 Canonical invocation and observation flags

The project's own command is `cd src/ && pipenv run pytest <path>::<test>` (equivalently
`python -m pytest`, which auto-loads `setup.cfg`). Two run modes are used below:

* **Canonical run** — inherits all `addopts` (including `--numprocesses auto` and
  `--cov`). Used to show the real xdist worker behavior (§Q1).
* **Focused observation run** — adds `-n0 --no-cov -p no:randomly -p no:cacheprovider`.
  These flags are **explicitly labelled and non-behavioral**: `-n0` pins a single worker
  for a clean single-process observation, `--no-cov` suppresses only the multi-thousand-line
  coverage *report* (not the test's behavior), `-p no:randomly` fixes order, and
  `-p no:cacheprovider` avoids writing a cache into the tree. None of them changes what
  the code under test does.

The canonical pytest banner (identical across all focused runs) is:

```text
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
```

### 1.5 Stability protocol

Every count/timing/decision value below was produced at least **twice** and confirmed
identical (or, where a value is non-deterministic, the same unchanged input was repeated
**25×** and the observed distribution is reported verbatim — see §Non-Determinism).
Temporary observation scripts (`/tmp/q1_probe_test.py`, `/tmp/q2_probe_test.py`,
`/tmp/q3_probe_test.py`, `/tmp/q4_probe_test.py`, `/tmp/nondet_probe_test.py`) subclass
the project's **own** `DirectoriesMixin` + Django `TestCase` harness, so they invoke the
real `DocumentClassifier.train`, `load_classifier`, `predict_correspondent`,
`scan_file_for_separating_barcodes`, `separate_pages`, `RasterisedDocumentParser.parse`,
and `consume_file` — no bypassing interfaces.

---

<a name="q1"></a>
## Q1 — Classifier reuse vs. retrain within a run, and cross-test effect

### Direct Answer

* **Within a run:** an **unchanged dataset short-circuits retraining.** The first
  `DocumentClassifier.train()` returns `True` and stores a SHA-1 `data_hash`; a second
  `train()` on the same data returns `False` (no retrain). Mutating the dataset changes
  the hash and forces a retrain (`True`). Observed sequence: **`True → False → True`.**
* **No in-memory cache:** `load_classifier()` constructs a **new** `DocumentClassifier`
  and re-reads `settings.MODEL_FILE` **from disk on every call** (distinct object `id()`
  each time; `a is b` → `False`). It returns **`None`** when the model file is absent.
* **Cross-test:** a later test **cannot** reuse an earlier test's trained model. Each
  test gets a **fresh per-test `MODEL_FILE`** inside a `tempfile.mkdtemp()` directory
  that is `rmtree`-d at teardown, and the suite runs under **pytest-xdist with 128
  separate worker processes** (`--numprocesses auto`), so there is no shared in-process
  memory and no surviving model file across tests.

### Mechanism (cause → effect, with `file:line`)

* **Retrain guard** — `src/documents/classifier.py:161-164`:
  ```python
  new_data_hash = m.digest()                              # L161 (SHA-1, m = hashlib.sha1() @ L124)
  if self.data_hash and new_data_hash == self.data_hash:  # L163
      return False                                        # L164  -> no retrain
  ```
  On success the training set is rebuilt from the DB and `self.data_hash = new_data_hash`
  (`L247`), `return True` (`L249`). An empty set raises `ValueError("No training data
  available.")` (`L159`).
* **Cacheless loader** — `src/documents/classifier.py:30-57`: `load_classifier()` returns
  `None` if `not os.path.isfile(settings.MODEL_FILE)` (`L31`, `return None` @ `L36`),
  else constructs `classifier = DocumentClassifier()` (`L38`, a **new** instance) and
  `classifier.load()` reads the file from disk (`L40`). There is no module-level
  memoization anywhere.
* **Inbox-excluded training set** — `src/documents/classifier.py:125-127`:
  `for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True,):`.
* **Per-test isolation** — `src/documents/tests/utils.py`: `setup_directories()` calls
  `tempfile.mkdtemp()` for the data dir (`L18`) and enables an `override_settings(...,
  MODEL_FILE=os.path.join(dirs.data_dir, "classification_model.pickle"), ...)` (`L45`,
  `.enable()` @ `L48`); `DirectoriesMixin.setUp` runs this per test (`L77-78`);
  `tearDown` (`L81-83`) calls `remove_dirs` → `shutil.rmtree(...)` (`L53-57`).
* **Default model path** — `src/paperless/settings.py:74`:
  `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")`.
* **Parallelism** — `src/setup.cfg`: `--numprocesses auto` (xdist) + env
  `PAPERLESS_DISABLE_DBHANDLER=true`.

### Commands Run

```bash
# 1. Retrain-guard test (asserts train()==True then train()==False)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 2. Save/reload leaves the guard armed (loaded model's train() -> False)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 3. The caching test is skipped (proves NO cache)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::test_load_classifier_cached \
    -p no:cacheprovider -n0 --no-cov -rs -v

# 4. Canonical run showing real xdist workers (inherits --numprocesses auto)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    -p no:cacheprovider -v

# 5. Temporary probe: no-cache, retrain guard, per-test isolation (real functions)
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/q1_probe_test.py -c /app/src/setup.cfg --rootdir /app/src \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v
```

### Verbatim Observed Output

**(1) `testDatasetHashing` — identical across 2 runs:**

```text
documents/tests/test_classifier.py .                                     [100%]
======================== 1 passed, 6 warnings in 1.90s =========================
```

**(3) The caching test is skipped — direct proof there is no in-memory cache:**

```text
documents/tests/test_classifier.py s                                     [100%]
...
SKIPPED [1] documents/tests/test_classifier.py:391: Disabled caching due to high memory usage - need to investigate.
======================== 1 skipped, 6 warnings in 0.08s ========================
```

(The skip decorator/reason is `@pytest.mark.skip(reason="Disabled caching due to high
memory usage - need to investigate.")` at `test_classifier.py:399-400`, on
`test_load_classifier_cached` @ `L402`.)

**(4) Canonical `--numprocesses auto` run — pytest-xdist spins up 128 worker processes:**

```text
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
...
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
created: 128/128 workers
128 workers [1 item]
.                                                                        [100%]
======================= 1 passed, 774 warnings in 28.16s =======================
```

The same single test takes ~1.9 s single-worker (`-n0`) versus ~28 s spinning up 128
worker processes — concrete evidence that the default suite runs across **128 separate
OS processes**, each with an isolated database. No classifier object can be shared
between them.

**(5) Temporary probe `/tmp/q1_probe_test.py` — 5 passed, identical behavior across 2 runs:**

```text
[Q1-MISSING] --- load_classifier() returns None if model absent ---
[Q1-MISSING] MODEL_FILE: /tmp/tmpx8s6dapx/classification_model.pickle
[Q1-MISSING] model file exists: False
[Q1-MISSING] load_classifier() returned: None
[Q1-NOCACHE] --- load_classifier() has NO in-memory cache ---
[Q1-NOCACHE] MODEL_FILE: /tmp/tmpv392z_25/classification_model.pickle
[Q1-NOCACHE] model file exists: True
[Q1-NOCACHE] call#1 -> type: DocumentClassifier id: 134508057215040
[Q1-NOCACHE] call#2 -> type: DocumentClassifier id: 134513617478992
[Q1-NOCACHE] a is b (same cached object?): False
[Q1-NOCACHE] distinct instances (id differ): True
[Q1-RETRAIN] --- data_hash retrain guard (before/during/after) ---
[Q1-RETRAIN] BEFORE first train -> data_hash: None
[Q1-RETRAIN] train() call#1 (fresh data) returned: True
[Q1-RETRAIN] AFTER first train -> data_hash: 230b98c1cbe4bb261c254b08a6d463334e9ba45d
[Q1-RETRAIN] train() call#2 (UNCHANGED data) returned: False
[Q1-RETRAIN] data_hash identical to call#1: True
[Q1-RETRAIN] train() call#3 (AFTER adding a doc) returned: True
[Q1-RETRAIN] data_hash changed after mutate: True
[Q1-ISO] --- per-test MODEL_FILE isolation (test #1) ---
[Q1-ISO] test#1 MODEL_FILE: /tmp/tmpzw6caqju/classification_model.pickle
[Q1-ISO] --- per-test MODEL_FILE isolation (test #2) ---
[Q1-ISO] test#2 MODEL_FILE: /tmp/tmpen8yw4qy/classification_model.pickle
[Q1-ISO] test#1 MODEL_FILE (from prior test): /tmp/tmpzw6caqju/classification_model.pickle
[Q1-ISO] test#1 temp data_dir still exists after its teardown: False
[Q1-ISO] test#2 path DIFFERS from test#1 path: True
======================== 5 passed, 6 warnings in 1.94s =========================
```

The `data_hash` value `230b98c1cbe4bb261c254b08a6d463334e9ba45d` reproduced **identically**
across both runs (SHA-1 over the fixed content is deterministic), while the two
`load_classifier()` calls returned objects with **different `id()`** and the two tests'
`MODEL_FILE` paths were **distinct**, with the first test's temp dir already **removed**
by the time the second test ran.

### Rationale

The `data_hash` guard (`classifier.py:163-164`) makes repeated training within a single
classifier instance idempotent, so re-invoking `train()` on unchanged data is a no-op —
the model is *reused*, not retrained. Across tests there is nothing to reuse: the loader
holds no cache (`classifier.py:30-57`), every test writes to its own throwaway
`MODEL_FILE` (`utils.py:45`) that is deleted at teardown, and the 128 xdist worker
processes share no memory. Therefore a later test never inherits an earlier test's
trained model.

---


<a name="q2"></a>
## Q2 — Automatic correspondent matching: training-doc count, train timing, confidence threshold

### Direct Answer

* **(c) There is NO probability/confidence threshold.** `predict_correspondent()` calls
  `MLPClassifier.predict()` (which returns the **argmax class label**) and accepts that
  label unless it equals the sentinel **`-1`** (in which case it returns `None`). It
  **never** calls `predict_proba()`. Observed instrumentation: over two predictions,
  `mlp.predict()` was called **2** times and `mlp.predict_proba()` was called **0** times.
  The model always yields its top class; acceptance is a sentinel check, not a cutoff.
* **(a) Training-document count** (effective count fed to `train()`, inbox-excluded):
  `test_one_correspondent_predict` → **1** document (1 total, 1 effective);
  `test_one_correspondent_predict_manydocs` → **2** documents (2 total, 2 effective).
  Tagging a document with an inbox tag drops the effective count (observed **2 → 1**),
  because `train()` excludes inbox-tagged documents.
* **(b) Training timing:** documents are inserted **first**, then `train()` is called
  **after** — an **insert-then-train** ordering. Observed `Document.objects.count()`
  before any insert = `0`; documents inserted; `train()` invoked afterward.

### Mechanism (cause → effect, with `file:line`)

* **No threshold — argmax + sentinel only** — `src/documents/classifier.py:251-260`:
  ```python
  def predict_correspondent(self, content):                              # L251
      if self.correspondent_classifier:                                  # L252
          X = self.data_vectorizer.transform([preprocess_content(content)])  # L253
          correspondent_id = self.correspondent_classifier.predict(X)    # L254  (argmax label)
          if correspondent_id != -1:                                     # L255
              return correspondent_id                                    # L256
          else:
              return None                                                # L258
      else:
          return None                                                    # L260
  ```
  Per the scikit-learn 1.0.2 API, `MLPClassifier.predict()` returns the argmax class
  label; `predict_proba()` is a **separate** method that returns probabilities and is
  never called here — so no probability cutoff is applied. **(inferred from the
  scikit-learn API doc; confirmed observationally below by the `predict_proba` call
  count of 0.)**
* **Effective (inbox-excluded) training set** — `src/documents/classifier.py:125-127`:
  `for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True,):`.
* **Production matching entry point (union of regex OR ML id)** —
  `src/documents/matching.py:21-31`: `pred_id = classifier.predict_correspondent(
  document.content)` (`L23`), then
  `filter(lambda o: matches(o, document) or o.pk == pred_id, correspondents)` (`L30`) —
  a correspondent is selected if it matches by regex rule **or** its pk equals the ML
  prediction. `set_correspondent()` in `src/documents/signals/handlers.py` is the
  production caller during consumption.
* **The only ratio comparison is unrelated to ML** — `src/documents/matching.py:135`:
  `if fuzz.partial_ratio(match, text) >= 90:`. This `>= 90` gate belongs to the regex
  **`MATCH_FUZZY`** algorithm, **not** to the ML prediction — it must not be mistaken
  for a "confidence threshold."

### Commands Run

```bash
# 1. Single-document correspondent prediction
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 2. Two-document correspondent prediction (doc1 -> c1, doc2 -> none)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 3. Train + Predict assertions (classes_ == [-1, c1.pk]; -1 -> None)
cd /app/src && python -m pytest documents/tests/test_classifier.py::TestClassifier::testTrain \
    documents/tests/test_classifier.py::TestClassifier::testPredict \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 4. Temporary probe: effective count, insert-then-train, no predict_proba, inbox edge, fuzzy clarification
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/q2_probe_test.py -c /app/src/setup.cfg --rootdir /app/src \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v
```

### Verbatim Observed Output

**(1)+(2)+(3) pytest — passing, identical across 2 runs:**

```text
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED
======================== 1 passed, 6 warnings in 1.88s =========================

documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED
======================== 1 passed, 6 warnings in 1.91s =========================

documents/tests/test_classifier.py::TestClassifier::testTrain PASSED
documents/tests/test_classifier.py::TestClassifier::testPredict PASSED
======================== 2 passed, 6 warnings in 1.93s =========================
```

`testTrain` asserts `self.classifier.correspondent_classifier.classes_ == [-1, c1.pk]`
and `testPredict` asserts `predict_correspondent(doc1_content) == c1.pk` and
`predict_correspondent(doc2_content) is None` — i.e. the `-1` sentinel is mapped to
`None`, with no probability involved.

**(4) Temporary probe `/tmp/q2_probe_test.py` — 4 passed, identical across 2 runs:**

```text
[Q2a-COUNT] --- effective (inbox-excluded) training count before train() ---
[Q2a-COUNT] one_correspondent_predict fixture: Document.objects total = 1
[Q2a-COUNT] one_correspondent_predict fixture: EFFECTIVE (exclude inbox) = 1
[Q2a-COUNT] manydocs fixture: Document.objects total = 2
[Q2a-COUNT] manydocs fixture: EFFECTIVE (exclude inbox) = 2
[Q2a-EDGE] add inbox tag to one of 2 docs -> total = 2 ; EFFECTIVE = 1
[Q2b-TIMING] Document.objects.count() BEFORE any insert = 0
[Q2b-TIMING] inserted 2 documents; now count = 2
[Q2b-TIMING] calling classifier.train() AFTER inserts -> insert-then-train confirmed
[Q2c-THRESHOLD] correspondent_classifier type = MLPClassifier
[Q2c-THRESHOLD] classes_ (manydocs) = [-1  1]
[Q2c-THRESHOLD] mlp.predict()      call count during 2 predictions = 2
[Q2c-THRESHOLD] mlp.predict_proba() call count during 2 predictions = 0
[Q2c-THRESHOLD] predict_correspondent(doc1_content) raw return = [1]  (== c1.pk=1, label != -1 -> accepted)
[Q2c-THRESHOLD] predict_correspondent(doc2_content) raw return = None (raw argmax label = [-1] sentinel -> rejected)
[Q2c-THRESHOLD] (for reference only) predict_proba(doc2) = [0.6203 0.3797] max=0.6203 EXISTS but is UNUSED by code
[Q2-FUZZY] matching.py:135 fuzz.partial_ratio>=90 belongs to regex MATCH_FUZZY, NOT ML
[Q2-FUZZY] match_correspondents(doc,None) FUZZY correspondent: 'a foobar invoice' -> ['Foo'] ; 'nothing here' -> []
======================== 4 passed, 6 warnings in 2.12s =========================
```

The behavioral facts are stable across runs: the **effective count** (1 / 2, dropping to
1 when an inbox tag is added), the **insert-then-train** ordering, and — decisively —
`predict_proba()` **call count = 0**. The probability values printed *for reference*
(`0.6203`) vary run-to-run because the MLP is unseeded (see Non-Determinism), but the
code never reads them, so they cannot form a threshold.

### Rationale

The training set is small and inbox-excluded (`classifier.py:125-127`); training strictly
follows the document inserts; and because acceptance is `argmax(predict()) != -1`
(`classifier.py:255-256`) with `predict_proba()` never called, the acceptance decision is
**not thresholded on any probability** — the model always returns its top class. The only
`>= 90` ratio in the matching layer (`matching.py:135`) is the regex FUZZY gate, wholly
separate from the ML path.

---


<a name="q3"></a>
## Q3 — Documents with no extractable text: OCR subprocess and assigned mime type

### Direct Answer

* **OCR subprocess:** the invoked subprocess is **`ocrmypdf.ocr(**args)`**
  (`parsers.py:261` primary; `parsers.py:298` force-OCR retry), which drives
  **Tesseract** (container `tesseract 4.1.1`). When the first pass finds no text, a
  `NoTextFoundException` triggers a **force-OCR retry** (`safe_fallback=True`,
  `force_ocr=True`); if text is still absent the **last-resort branch sets
  `self.text = ""`**. (For an *encrypted* no-text PDF, `ocrmypdf.ocr` raises
  `EncryptedPdfError` on the first call, the encrypted branch logs that OCR is
  impossible, and the last-resort branch again yields empty text.)
* **Mime type:** derived from the **input file** via
  **`magic.from_file(self.path, mime=True)`** (`consumer.py:219`). A PDF stays
  `application/pdf` and a PNG stays `image/png` **regardless** of whether any text was
  extracted — the empty OCR text does **not** change the mime type.

### Mechanism (cause → effect, with `file:line`)

* **Primary OCR call** — `src/paperless_tesseract/parsers.py:261`: `ocrmypdf.ocr(**args)`
  (drives Tesseract). Text is read at `L264`; if empty, `raise NoTextFoundException(...)`
  (`L266-267`).
* **Fallback / force-OCR retry** — `src/paperless_tesseract/parsers.py:276-310`:
  `except (NoTextFoundException, InputFileError) as e:` (`L276`) builds fallback params
  via `construct_ocrmypdf_parameters(..., safe_fallback=True)` (`L288-293`) and calls
  `ocrmypdf.ocr(**args)` again (`L298`).
* **Last-resort empty text** — `src/paperless_tesseract/parsers.py:316-327`:
  `if not self.text:` (`L318`) → if the original had text use it, else warn
  *"No text was found in {document_path}, the content will be empty."* (`L324`) and set
  `self.text = ""` (`L327`).
* **Mime detection from input file** — `src/documents/consumer.py:219`:
  `mime_type = magic.from_file(self.path, mime=True)`; the classifier is loaded once per
  consume at `L292` (`classifier = load_classifier()`).

### Commands Run

```bash
# 1. Encrypted, no-extractable-text PDF -> empty text (OCR_MODE=skip)
cd /app/src && python -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 2. Form PDF, no text on skip -> force-OCR recovers text (OCR_MODE=redo)
cd /app/src && python -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 3. Skip-archive, no-text case
cd /app/src && python -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 4. Exact args dict passed to ocrmypdf.ocr
cd /app/src && python -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_ocrmypdf_parameters \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 5. Temporary probe: real RasterisedDocumentParser.parse() on no-text samples;
#    wraps ocrmypdf.ocr to count invocations & spies the parser log; prints magic mime
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/q3_probe_test.py -c /app/src/setup.cfg --rootdir /app/src \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v
```

### Verbatim Observed Output

**(1)-(4) pytest — passing, identical across 2 runs (durations show real OCR work):**

```text
paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted PASSED
======================== 1 passed, 6 warnings in 1.74s =========================

paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED
======================== 1 passed, 6 warnings in 8.11s =========================

paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED
======================== 1 passed, 6 warnings in 3.88s =========================

paperless_tesseract/tests/test_parser.py::TestParser::test_ocrmypdf_parameters PASSED
======================== 1 passed, 6 warnings in 1.48s =========================
```

`test_encrypted` (`test_parser.py:178`) asserts `archive_path is None` and
`get_text() == ""` for `samples/encrypted.pdf`; `test_with_form_error_notext`
(`test_parser.py:190`, OCR_MODE=`redo`) proves the `ocrmypdf → Tesseract` pipeline
*does* extract text on force-OCR (it recovers the form's text), confirming the retry
path is real.

**(5) Temporary probe `/tmp/q3_probe_test.py` — 2 passed, identical across 2 runs
(the probe wraps `ocrmypdf.ocr` to count invocations and log the exact kwargs, and
spies on the `paperless.parsing.tesseract` logger; `parse()` is the real method):**

```text
[Q3] ==== parsing encrypted.pdf (hint mime application/pdf) ====
[Q3-MIME] magic.from_file(path, mime=True) -> application/pdf
[Q3-LOG] parser.log(warning): Error while getting text from PDF document with pdfminer.six
[Q3-OCR] ocrmypdf.ocr INVOKED call#1: input_file=encrypted.pdf output_type='pdfa' skip_text=True redo_ocr=None force_ocr=None
[Q3-OCR] ocrmypdf.ocr call#1 RAISED EncryptedPdfError: Input PDF is encrypted. The encryption must be removed to
[Q3-LOG] parser.log(warning): This file is encrypted, OCR is impossible. Using any text present in the original file.
[Q3-LOG] parser.log(warning): No text was found in /app/src/paperless_tesseract/tests/samples/encrypted.pdf, the content will be empty.
[Q3-OCR] total ocrmypdf.ocr invocations: 1
[Q3-TEXT] parser.get_text() repr: ''
[Q3-TEXT] parser.get_text() == '' : True
[Q3-TEXT] parser.archive_path: None
[Q3-MIME] mime AFTER parse (still from input file): application/pdf (unchanged by empty OCR text)
[Q3] ==== parsing no-text-alpha.png (hint mime image/png) ====
[Q3-MIME] magic.from_file(path, mime=True) -> image/png
[Q3-LOG] parser.log(warning): Error while getting DPI from image /app/src/paperless_tesseract/tests/samples/no-text-alpha.png: 'dpi'
[Q3-OCR] ocrmypdf.ocr INVOKED call#1: input_file=no-text-alpha.png output_type='pdfa' skip_text=True redo_ocr=None force_ocr=None
[Q3-LOG] parser.log(warning): Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[Q3-LOG] parser.log(warning): Error while getting DPI from image /app/src/paperless_tesseract/tests/samples/no-text-alpha.png: 'dpi'
[Q3-OCR] ocrmypdf.ocr INVOKED call#2: input_file=no-text-alpha.png output_type='pdfa' skip_text=None redo_ocr=None force_ocr=True
[Q3-LOG] parser.log(warning): No text was found in /app/src/paperless_tesseract/tests/samples/no-text-alpha.png, the content will be empty.
[Q3-OCR] total ocrmypdf.ocr invocations: 2
[Q3-TEXT] parser.get_text() repr: ''
[Q3-TEXT] parser.get_text() == '' : True
[Q3-TEXT] parser.archive_path: /tmp/tmpt1vt340o/paperless-tjvy00ql/archive.pdf
[Q3-MIME] mime AFTER parse (still from input file): image/png (unchanged by empty OCR text)
======================== 2 passed, 6 warnings in 3.61s =========================
```

(Second run footer: `2 passed, 6 warnings in 3.93s`.) All behavioral facts — the mime
types (`application/pdf`, `image/png`), the invocation counts (**1** for the encrypted
branch that raised on the first call, **2** for the force-OCR retry branch), the empty
`get_text()`, and the branch log lines — reproduced identically across both runs; only
the temporary `archive_path` filename varied (expected). Note the encrypted PDF's first
`ocrmypdf.ocr` call raises `EncryptedPdfError` **before** producing an archive
(`archive_path = None`), whereas the PNG's force-OCR retry still writes an archive PDF
even though the extracted text is empty.

### Rationale

The OCR engine and the mime source are **independent** facts. OCR text can end up empty
through the documented fallback chain (primary `ocrmypdf.ocr` → `NoTextFoundException`
force-OCR retry / `EncryptedPdfError` branch → last-resort `self.text = ""`,
`parsers.py:261/276-310/316-327`), but the mime type is computed by `magic.from_file`
over the **input file bytes** (`consumer.py:219`) and is therefore unaffected by whether
any text was extracted.

---


<a name="q4"></a>
## Q4 — Barcode splitting: record count, trigger values, decision site, effect on training data

### Direct Answer

* **Effect on training data (lead result):** a barcode-split call creates **ZERO
  `Document` rows synchronously.** `consume_file()` returns the string
  **`"File successfully split"`** and creates no `Document`. Observed state transition on
  a barcoded input: `Document.objects.count()` **BEFORE = 0 → returns "File successfully
  split" → AFTER = 0.** The effective training set (`Document.objects.exclude(inbox)`)
  therefore does **not** change at split time; it changes **only after** the split parts
  are re-consumed as their own `Document`s.
* **Record count:** a single barcoded input with `N` separator pages yields
  **`len(separators) + 1` output PDF files** from `separate_pages()`. Observed:
  `[0] → 2 files`, `[1] → 2 files`, `[2, 5] → 3 files` (named
  `<name>_document_0.pdf`, `_document_1.pdf`, ...). These are **files**, not `Document`
  rows.
* **Trigger value:** the separator string `settings.CONSUMER_BARCODE_STRING`, default
  **`"PATCHT"`**. PATCHT is detected identically across **Code 39, Code 128, and QR** —
  `barcode_reader()` returned `['PATCHT']` for all three encodings.
* **Decision site:** `scan_file_for_separating_barcodes()`
  (`tasks.py:96-110`), specifically `if separator_barcode in current_barcodes:
  separator_page_numbers.append(current_page_number)` (`tasks.py:108-109`).

### Mechanism (cause → effect, with `file:line`)

* **Barcode read** — `src/documents/tasks.py:82`: `detected_barcodes =
  pyzbar.decode(image)` (inside `barcode_reader`, `L75-93`), after `convert_from_path`
  rasterization (poppler).
* **Trigger string + decision site** — `src/documents/tasks.py:96-110`:
  `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` (`L102`), then the decision
  `if separator_barcode in current_barcodes: separator_page_numbers.append(
  current_page_number)` (`L108-109`).
* **Split arithmetic** — `src/documents/tasks.py:113-161`: `separate_pages()` writes
  `"{}_document_0.pdf"` (`L134`); the loop `range(page_number + 1, next_page)` (`L149`)
  **skips the separator page itself**; subsequent parts are `"{}_document_{}.pdf"`
  with `count + 1` (`L154`); the function returns `len(separators) + 1` paths.
* **Zero-Document split return** — `src/documents/tasks.py:233`: `return "File
  successfully split"` (no `Document` created). The no-barcode fallthrough at `L236`,
  `Consumer().try_consume_file(...)`, is what creates **1** `Document`.
* **Config defaults** — `src/paperless/settings.py:502-506`: `CONSUMER_ENABLE_BARCODES`
  (default `false`, `L502-503`); `CONSUMER_BARCODE_STRING = os.getenv(
  "PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")` (`L506`).
* **Training re-read (inbox-excluded)** — `src/documents/classifier.py:125-127`; and
  `train_classifier()` (`tasks.py:48-55`) short-circuits unless a
  `Tag`/`DocumentType`/`Correspondent` uses `MATCH_AUTO`.

### Commands Run

```bash
# 1. barcode_reader across ALL symbologies/variants (Code39/Code128/QR/distortion/unreadable/no_barcode/custom)
cd /app/src && python -m pytest documents/tests/test_tasks.py -k "barcode_reader" \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -p no:sugar -vv

# 2. scan_file_for_separating_barcodes — separator page-index detection variants
cd /app/src && python -m pytest documents/tests/test_tasks.py -k "scan_file_for_separating" \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -p no:sugar -vv

# 3. separate_pages output-count assertions (N separators -> N+1 files) + no-separator case
cd /app/src && python -m pytest documents/tests/test_tasks.py::TestTasks::test_separate_pages \
    documents/tests/test_tasks.py::TestTasks::test_separate_pages_no_list \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -v

# 4. Temporary probe: real scan_file_for_separating_barcodes() + separate_pages() on the PATCHT corpus,
#    then consume_file() with CONSUMER_ENABLE_BARCODES=true showing Document.count before/after == 0,
#    and a non-barcoded consume creating exactly 1 Document that enters the inbox-excluded training set
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/q4_probe_test.py -c /app/src/setup.cfg --rootdir /app/src \
    -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v
```

### Verbatim Observed Output

**(1) `barcode_reader` — 11 passed (all symbologies/variants), identical across 2 runs:**

```text
documents/tests/test_tasks.py::TestTasks::test_barcode_reader PASSED     [  9%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader2 PASSED    [ 18%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 PASSED [ 27%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_128_separator PASSED [ 36%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_qr_separator PASSED [ 45%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_separator PASSED [ 54%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion PASSED [ 63%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion2 PASSED [ 72%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_no_barcode PASSED [ 81%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr PASSED  [ 90%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_unreadable PASSED [100%]
================ 11 passed, 29 deselected, 6 warnings in 2.45s =================
```

(Run 2 footer: `11 passed, 29 deselected, 6 warnings in 2.33s`.)

**(2) `scan_file_for_separating_barcodes` — 10 passed (incl. upsidedown/QR/custom/wrong-QR):**

```text
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED [ 10%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes2 PASSED [ 20%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3 PASSED [ 30%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED [ 40%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes_upsidedown PASSED [ 50%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_128_barcodes PASSED [ 60%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_barcodes PASSED [ 70%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_qr_barcodes PASSED [ 80%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_qr_barcodes PASSED [ 90%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_wrong_qr_barcodes PASSED [100%]
================ 10 passed, 30 deselected, 6 warnings in 4.78s =================
```

(Run 2 footer: `10 passed, 30 deselected, 6 warnings in 4.19s`.)

**(3) `separate_pages` + `separate_pages_no_list` — 2 passed:**

```text
======================== 2 passed, 6 warnings in 1.55s =========================
```

(Run 2 footer: `2 passed, 6 warnings in 1.60s`.)

**(4) Temporary probe `/tmp/q4_probe_test.py` — 4 passed, identical across 2 runs:**

```text
[Q4-READ] === which barcode VALUES trigger a split ===
[Q4-READ] settings.CONSUMER_BARCODE_STRING (default): 'PATCHT'
[Q4-READ] barcode_reader(barcode-39-PATCHT.png) -> ['PATCHT']
[Q4-READ] barcode_reader(barcode-128-PATCHT.png) -> ['PATCHT']
[Q4-READ] barcode_reader(qr-code-PATCHT.png) -> ['PATCHT']
[Q4-SCAN] === decision site: separator page indices ===
[Q4-SCAN] scan_file_for_separating_barcodes(patch-code-t.pdf) -> [0] (single separator on page 0)
[Q4-SCAN] scan_file_for_separating_barcodes(patch-code-t-middle.pdf) -> [1] (separator in the middle)
[Q4-SCAN] scan_file_for_separating_barcodes(several-patcht-codes.pdf) -> [2, 5] (two separators)
[Q4-SCAN] scan_file_for_separating_barcodes(simple.pdf) -> [] (NO barcode)
[Q4-SEP] === record count = len(separators) + 1 (PDF files) ===
[Q4-SEP] patch-code-t.pdf: separators=[0] -> 2 output files (len(separators)+1 = 2) -> basenames ['patch-code-t_document_0.pdf', 'patch-code-t_document_1.pdf']
[Q4-SEP] patch-code-t-middle.pdf: separators=[1] -> 2 output files (len(separators)+1 = 2) -> basenames ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
[Q4-SEP] several-patcht-codes.pdf: separators=[2, 5] -> 3 output files (len(separators)+1 = 3) -> basenames ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[Q4-CONSUME] === consume_file() with barcodes: STATE TRANSITIONS ===
[Q4-CONSUME] Document.objects.count() BEFORE: 0 | effective inbox-excluded BEFORE: 0
[Q4-CONSUME] consume_file() RETURNED: 'File successfully split'
[Q4-CONSUME] Document.objects.count() AFTER: 0 | effective inbox-excluded AFTER: 0
[Q4-CONSUME] original input deleted by split path (os.unlink): True
[Q4-RECONSUME] === non-barcoded consume -> 1 Document -> enters inbox-excluded training set ===
[Q4-RECONSUME] separators in simple.pdf: [] (none -> fallthrough to try_consume_file)
[Q4-RECONSUME] Document.count BEFORE: 0 | effective training count BEFORE: 0
[2026-07-06 22:39:41,523] [INFO] [paperless.consumer] Document 2026-07-06 simple_input consumption finished
[Q4-RECONSUME] consume_file() returned: 'Success. New document id 1 created'
[Q4-RECONSUME] Document.count AFTER: 1 | effective training count AFTER: 1
[Q4-RECONSUME] created Document -> id: 1 | mime_type: application/pdf | in inbox-excluded training set: True
======================== 4 passed, 6 warnings in 6.57s =========================
```

(Run 2 footer: `4 passed, 6 warnings in 6.62s`.) Every value — the `'PATCHT'` default,
the three encodings all decoding to `['PATCHT']`, the separator indices `[0]`/`[1]`/`[2,
5]`/`[]`, the `len(separators)+1` file counts, and the crucial split state transition
`0 → "File successfully split" → 0` — reproduced identically across both runs.

### Rationale

The number of *records* produced by the split equals `len(separators) + 1` **PDF files**
(`tasks.py:113-161`), but the split call creates **zero `Document` rows** and returns
`"File successfully split"` (`tasks.py:233`). Therefore the split does **not**
synchronously alter the classifier's training data — the effective training set
(`Document.objects.exclude(tags__is_inbox_tag=True)`, `classifier.py:125-127`) only grows
once the split parts are re-consumed and persisted as their own `Document`s (each a
normal `try_consume_file` → `+1 Document`, as the re-consumption probe shows). This
timing gap — split now, `Document`s later — is directly relevant to the user's
non-determinism concern: *when* the split parts become `Document`s relative to *when*
`train_classifier()` next runs determines what data the model sees.

---


<a name="nd"></a>
## Non-Determinism — root cause of the flaky classification tests

> Per the governing rule *"explain, do not fix"*: the analysis below **diagnoses** the
> non-determinism from observed distributions. **No source file was modified.**

### Direct Answer

The classification non-determinism originates from the **`MLPClassifier` being
constructed with no `random_state`** (`classifier.py:219`, `:227`, `:238`). With
`random_state=None` (scikit-learn 1.0.2), the network's weights are **randomly
initialised on every `.fit()` call**, so the trained model differs from run to run. For
**well-separated** training data the argmax is stable (the model still lands on the same
top class), but for **borderline / ambiguous / tiny-degenerate** training data the argmax
— and therefore `predict_correspondent()`'s accept/reject decision — **flips run to
run**. `--numprocesses auto` (128 xdist workers, each with its own random model and its
own DB) is a compounding second layer, but the root cause is the missing seed.

### Mechanism (cause → effect, with `file:line`)

* `src/documents/classifier.py:219` — `self.tags_classifier = MLPClassifier(tol=0.01)`
* `src/documents/classifier.py:227` — `self.correspondent_classifier =
  MLPClassifier(tol=0.01)`
* `src/documents/classifier.py:238` — `self.document_type_classifier =
  MLPClassifier(tol=0.01)`

  None pass `random_state`. Per the scikit-learn 1.0.2 API, `random_state=None` means the
  weight/bias initialisation and the solver's stochastic behavior draw from the global RNG
  each fit → different weights → potentially different argmax on ambiguous inputs.
* `src/documents/classifier.py:194` — the vectorizer is `CountVectorizer(...)`
  (deterministic given fixed vocabulary); the variance is entirely in the MLP fit.
* `src/setup.cfg` — `--numprocesses auto` spawns independent worker processes (observed:
  **128 workers**), each training its own unseeded MLP against its own per-worker DB.

### Commands Run

```bash
# A. SAME well-separated manydocs fixture, repeated x25 -> is the argmax stable?
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/nondet_probe_test.py::NonDeterminismDistribution -c /app/src/setup.cfg \
    --rootdir /app/src -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v

# B. CONSTRUCTED borderline input (two MATCH_AUTO correspondents on IDENTICAL content),
#    SAME data repeated x25 -> report the flip distribution (labeled non-canonical)
cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true \
    python -m pytest /tmp/nondet_probe_test.py::NonDeterminismFlip -c /app/src/setup.cfg \
    --rootdir /app/src -p no:cacheprovider -p no:randomly -n0 --no-cov -s -v
```

### Verbatim Observed Output

**(A) Real `manydocs` fixture x25 — argmax is STABLE for well-separated data, but the
underlying model differs every run:**

```text
[ND] === repeat SAME manydocs fixture x25, report distribution ===
[ND] predict_correspondent(doc1) distribution over 25 runs: {'1': 25} (expected c1.pk = 1 )
[ND] predict_correspondent(doc2) distribution over 25 runs: {'None': 25} (expected None via -1 sentinel)
[ND] correspondent_classifier.coefs_[0][0][0] distinct values: 25 of 25 -> weights RANDOMLY initialised each fit (no random_state)
[ND] doc2 max predict_proba distinct values: 25 of 25 sample(first 5): [0.6345, 0.6853, 0.6486, 0.6521, 0.6078]
======================== 1 passed, 6 warnings in 2.49s =========================
```

This is the key nuance: `predict_correspondent(doc1)` was `1` in **all 25** runs and
`doc2` was `None` in **all 25** runs — so `test_one_correspondent_predict[_manydocs]`
themselves are **not** the flaky tests. Yet the model's first weight
(`coefs_[0][0][0]`) took **25 distinct values across 25 runs**, and `doc2`'s max
probability likewise took **25 distinct values** (`0.6345, 0.6853, ...`) — proving the
model is re-randomised every fit even when the argmax happens to be stable.

**(B) Constructed borderline case x25 (labeled NON-CANONICAL — built to expose the
mechanism) — the argmax FLIPS run to run:**

```text
[NDFLIP] === constructed borderline input, SAME data x25 ===
[NDFLIP] c1.pk=1 c2.pk=2; both trained on identical content 'payment invoice total due'
[NDFLIP] predict_correspondent(ambiguous) distribution over 25 runs: {'1': 12, '2': 13} -> argmax FLIPS run-to-run purely from random init
======================== 1 passed, 6 warnings in 2.19s =========================
```

Repeating the **same unchanged input** a second time produced a **different**
distribution:

```text
[NDFLIP] predict_correspondent(ambiguous) distribution over 25 runs: {'1': 11, '2': 14} -> argmax FLIPS run-to-run purely from random init
======================== 1 passed, 6 warnings in 2.42s =========================
```

When two `MATCH_AUTO` correspondents are trained on **identical** content, the predicted
class is decided purely by the random weight initialisation, so it flips
(`{'1': 12, '2': 13}` then `{'1': 11, '2': 14}`) across otherwise-identical runs. This
directly reproduces the class of failure the user is chasing.

### Rationale

Because the MLP has no fixed `random_state` (`classifier.py:219/227/238`), each `train()`
yields different weights. For well-separated fixtures the argmax is stable (so those
specific tests pass deterministically), but any test whose assertion depends on the
argmax of **ambiguous or degenerate** training data can pass or fail depending on the
run — a genuinely non-deterministic outcome. Under `--numprocesses auto` this is
amplified across 128 independently-randomised worker processes. The `data_hash` (Q1) and
`magic` mime (Q3) paths are deterministic; the **unseeded MLP argmax on borderline
inputs** is the non-deterministic surface. (Fixing it — e.g. by seeding — is explicitly
out of scope per the read-only / "explain, do not fix" directive.)

---


<a name="checklist"></a>
## Final Coverage Checklist

Every distinct sub-item named by the four questions, with its **value**, **`file:line`**,
**observed evidence** (command + output shown above), **sibling variants exercised**, and
**rationale**.

### Q1 — Classifier reuse vs. retrain, cross-test effect

| Sub-item | Value (direct answer) | `file:line` | Observed evidence | Sibling variants exercised | Rationale |
|---|---|---|---|---|---|
| Retrain guard (unchanged data) | `train()` → **`False`** (no retrain); sequence `True→False→True` | `classifier.py:161-164`, `:247`, `:249` | q1 probe: `train#1=True`, `train#2=False`, `train#3=True`; `data_hash 230b98c1…` identical ×2 runs; `testDatasetHashing` PASSED | fresh train (`True`), unchanged (`False`), mutated (`True`), empty→`ValueError` (`:159`) | SHA-1 hash equality short-circuits re-fit → idempotent training within a run |
| No in-memory cache | **No cache**; new instance + disk read every call; `None` if absent | `classifier.py:30-57` (`:31`,`:36`,`:38`,`:40`,`:57`) | q1 probe: call#1 `id 134508057215040` ≠ call#2 `id 134513617478992`, `a is b: False`; missing→`None`; `test_load_classifier_cached` **SKIPPED** ("Disabled caching…") | present-file (distinct ids), absent-file (`None`), skip banner | Loader reconstructs+reloads each call → no shared model object |
| Per-test `MODEL_FILE` isolation | **Distinct** throwaway path per test; removed at teardown | `utils.py:18`,`:45`,`:53-57`,`:77-83`; `settings.py:74` | q1 probe: test#1 `/tmp/tmpzw6caqju/…` ≠ test#2 `/tmp/tmpen8yw4qy/…`; test#1 dir exists after teardown = `False` | two sequential tests; post-teardown existence check | `mkdtemp` + `rmtree` per test → no model file survives across tests |
| pytest-xdist parallelism | **128** separate worker processes, per-worker DB | `setup.cfg` (`--numprocesses auto`) | canonical run: `created: 128/128 workers`, `128 workers [1 item]`, 28.16s vs 1.9s `-n0` | canonical (128 workers) vs `-n0` single-worker | No shared in-process memory → cross-test model reuse impossible |

### Q2 — Automatic correspondent matching

| Sub-item | Value (direct answer) | `file:line` | Observed evidence | Sibling variants exercised | Rationale |
|---|---|---|---|---|---|
| (a) training-doc count | **1** (predict), **2** (manydocs); effective = inbox-excluded | `classifier.py:125-127` | q2 probe: total 1→eff 1; total 2→eff 2; **inbox edge** total 2→eff **1** | 1-doc, 2-doc, inbox-tagged edge | Inbox-tagged docs excluded from training set |
| (b) train timing | **Insert-then-train** (docs first, `train()` after) | test source `test_classifier.py:191-225`; `classifier.py:125` | q2 probe: count BEFORE insert=0 → insert 2 → `train()` after | single-doc & many-doc fixtures | Training reads the DB *after* inserts complete |
| (c) confidence threshold | **NONE** — argmax `predict()`, reject only sentinel `-1` | `classifier.py:251-260` (`:254`,`:255-256`) | q2 probe: `predict()` calls=**2**, `predict_proba()` calls=**0**; `doc1→[1]`, `doc2→None`; `predict_proba` value exists but UNUSED | accepted (label 1), rejected (`-1`→None) | Only `.predict()` argmax used; no probability cutoff |
| Fuzzy-gate clarification | `fuzz.partial_ratio ≥ 90` is regex FUZZY, **unrelated** to ML | `matching.py:135` | q2 probe: FUZZY correspondent `'a foobar invoice'→['Foo']`, `'nothing here'→[]` | positive & negative fuzzy match | Distinct algorithm; not an ML confidence threshold |
| Production entry point (union) | select if regex-match **OR** `pk == predicted id` | `matching.py:21-31` (`:23`,`:30`); `signals/handlers.py` `set_correspondent()` | referenced; matching behavior observed via probe | — | ML prediction is unioned with regex rules |

### Q3 — No-extractable-text OCR subprocess and mime type

| Sub-item | Value (direct answer) | `file:line` | Observed evidence | Sibling variants exercised | Rationale |
|---|---|---|---|---|---|
| OCR subprocess | **`ocrmypdf.ocr(**args)`** → drives Tesseract 4.1.1 | `parsers.py:261` (primary), `:298` (retry) | q3 probe: `ocrmypdf.ocr INVOKED call#1 … skip_text=True`; `call#2 … force_ocr=True` | primary pass, force-OCR retry | ocrmypdf is the invoked OCR subprocess |
| Fallback chain | primary → `NoTextFound`/`EncryptedPdf` branch → force-OCR retry → last-resort `text=""` | `parsers.py:264-267`,`:276-310`,`:316-327` (`:318`,`:324`,`:327`) | q3 probe: encrypted→`EncryptedPdfError` (1 call); png→NoTextFound→retry (2 calls); both `get_text()==''` | encrypted (1 call), non-encrypted no-text (2 calls) | Exhausting OCR yields empty text via documented fallbacks |
| Assigned mime type | from **input file** via `magic.from_file(self.path, mime=True)`; PDF stays `application/pdf` | `consumer.py:219` | q3 probe: `application/pdf` (encrypted.pdf), `image/png` (png), **unchanged** by empty OCR | pdf & png inputs | Mime derived from input bytes, independent of OCR text |
| Invocation counts | **1** (encrypted, raised) / **2** (force-OCR retry) | `parsers.py:261`,`:298` | q3 probe: `total ocrmypdf.ocr invocations: 1` / `: 2` | both branches | Retry path calls OCR twice; encrypted raises on first |

### Q4 — Barcode splitting

| Sub-item | Value (direct answer) | `file:line` | Observed evidence | Sibling variants exercised | Rationale |
|---|---|---|---|---|---|
| Record count from one input | **`len(separators)+1` PDF files**; **ZERO `Document` rows** in the split call | `tasks.py:113-161`,`:233` | q4 probe: `[0]→2`, `[1]→2`, `[2,5]→3` files; consume `0 → "File successfully split" → 0` | 1-sep, mid-sep, 2-sep, no-sep | Split emits files, not `Document`s |
| Trigger values | **`"PATCHT"`**; detected as **Code 39, Code 128, QR** | `settings.py:506`; `tasks.py:82` | q4 probe: all three encodings → `['PATCHT']`; `barcode_reader` 11 passed | Code39, Code128, QR, distortion, unreadable, no-barcode, custom | Configured separator string matched across symbologies |
| Decision site | `scan_file_for_separating_barcodes()`; append page if separator present | `tasks.py:96-110` (`:102`,`:108-109`) | q4 probe: indices `[0]`/`[1]`/`[2,5]`/`[]`; `scan_file_for_separating` 10 passed | normal, upsidedown, QR, custom, wrong-QR | Page index recorded where separator barcode is found |
| Effect on training data | **No synchronous change** (0 Documents); training set grows only after re-consumption | `tasks.py:233`,`:236`; `classifier.py:125-127` | q4 probe: split AFTER count=0; non-barcoded re-consume → `New document id 1`, count=1, in training set=True | split path (0 docs) vs fallthrough consume (+1 doc) | Effective training set changes only when split parts become `Document`s |

### Non-Determinism

| Sub-item | Value (direct answer) | `file:line` | Observed evidence | Sibling variants exercised | Rationale |
|---|---|---|---|---|---|
| Root cause | `MLPClassifier` has **no `random_state`** → weights re-randomised each fit | `classifier.py:219`,`:227`,`:238` | nondet probe: `coefs_[0][0][0]` 25 distinct/25; well-separated argmax stable (`{'1':25}`/`{'None':25}`) | well-separated (stable) vs borderline (flips) | Unseeded MLP → model differs every run |
| Observed flip distribution | borderline input flips: `{'1':12,'2':13}` then `{'1':11,'2':14}` | `classifier.py:227` | ndflip probe ×2 runs, SAME input, different distributions | 2 repeated runs of identical input | Argmax on ambiguous data decided by random init |
| Compounding factor | **128** xdist workers, independent random models | `setup.cfg` | `created: 128/128 workers` | canonical vs `-n0` | Parallelism multiplies independently-random models |

---

<a name="proof"></a>
## Appendix — Read-Only Proof & Reproduction Notes

* **Source tree unchanged.** All investigation ran inside the container
  `paperless-qna-baked` (image `paperless-ngx-qna:local`); every temporary probe
  (`/tmp/q1_probe_test.py`, `/tmp/q2_probe_test.py`, `/tmp/q3_probe_test.py`,
  `/tmp/q4_probe_test.py`, `/tmp/nondet_probe_test.py`) lived under the container's `/tmp`,
  never in the repository tree. The host repository's only change is this single
  document. Verified with `git status --porcelain` (see below).
* **Stability.** Every count/timing/branch value above was captured on **≥2 runs** and
  reproduced identically; the one genuinely non-deterministic value (borderline-input
  argmax) is reported as an observed **distribution** across repeated identical runs, not
  stabilised.
* **Canonical config.** All runs used the project's own `setup.cfg` (`DJANGO_SETTINGS_MODULE=paperless.settings`,
  `PAPERLESS_DISABLE_DBHANDLER=true`). Where a single-worker or no-random-order view was
  needed for a focused observation, `-n0 -p no:randomly` was used and **labeled**; the
  default suite retains `--numprocesses auto`.
