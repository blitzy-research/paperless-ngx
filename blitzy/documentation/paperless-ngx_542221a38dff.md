# paperless-ngx — ML + OCR Document-Processing Pipeline: Runtime Behavioral Investigation

**Subject:** The paperless-ngx machine-learning (document classifier) and OCR document-processing pipeline, as it behaves during automated test execution.
**Commit under investigation:** `542221a38dff06361e07976452f9aea24d210542` (git `HEAD`, source tree treated as strictly read-only).
**Purpose:** Explain, from **directly observed runtime behavior**, the mechanics behind **non-deterministic failures reported in the document-classification test suite** — by answering four specific questions (Q1–Q4), each grounded in captured, unedited command output with `file:line` citations.

This document leads every answer with the **direct result**, follows with **cause→effect reasoning** naming the exact function/method and `file:line`, and pairs each behavioral claim with the **exact command** and its **complete, unedited output**. Statements derived from reading code rather than observing a run are explicitly labelled **(inferred)**.

---

## Canonical Environment & Methodology

All runtime evidence was captured inside the canonical Docker container (image `paperless-ngx-canon:ready`, derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_…_qna_1.01`), which carries the pinned runtime and every OCR/PDF/barcode binary. The general host sandbox is **not** canonical (it lacks `django`, `scikit-learn`, `ocrmypdf`, `magic`, `pdf2image`, `pikepdf`, `pyzbar`, and the `tesseract`/`gs`/`unpaper`/`pdftoppm`/`libzbar`/`convert`/`qpdf` binaries), so no evidence was taken there.

Verified environment context:

```
Python 3.9.23   (CI matrix ['3.8','3.9','3.10'] at .github/workflows/reusable-ci-backend.yml:L55; primary job pins 3.9)
django 4.0.4 · scikit-learn 1.0.2 · ocrmypdf 13.4.3 · python-magic 0.4.25 · pdf2image 1.16.0 · pikepdf 5.1.1 · pyzbar 0.1.9
pytest 8.4.2 · pytest-xdist 3.8.0 · pytest-django 4.11.1
Binaries present: tesseract 4.1.1, gs 9.53.3, unpaper, pdftoppm (poppler), convert (ImageMagick), qpdf; libzbar0 backing pyzbar
Container source: /app  (git HEAD 542221a38dff, pristine)
```

**Canonical invocation.** The project's `src/setup.cfg [tool:pytest]` pins the test configuration (`DJANGO_SETTINGS_MODULE=paperless.settings` at L9; `addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet` at L10; `env: PAPERLESS_DISABLE_DBHANDLER=true` at L11–L12). The CI command is `pipenv run pytest` from `src/` (`.github/workflows/reusable-ci-backend.yml:L86`). **In this container the equivalent canonical command is `python3 -m pytest`** — the pinned dependencies are installed **system-wide** (there is no `pipenv`/virtualenv layer), so `python3 -m pytest` runs the identical test session with the identical `setup.cfg` `addopts` (including pytest-xdist `--numprocesses auto`). All runs below were executed as the non-root `testuser` via:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc \
  'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest <targets>'
```

**Run flags used and why.** For *targeted* observation of a single behaviour I pass `-o addopts=""` (disables `--cov`/`--numprocesses auto` for a fast, single-process run) and `-p no:cacheprovider` (prevents pytest cache writes). When investigating the **non-determinism** I keep the default `--numprocesses auto` in play (adding only `--no-cov` to avoid coverage HTML artifacts) and, to *characterize* the effect, additionally pin `-n0` (serial) — every non-default run is labelled as such. Temporary observation scripts referenced below (`/tmp/thq/blitzy_adhoc_test_*.py`) live outside both source trees and were **deleted** after the investigation (see the closing Read-Only Invariant appendix).

---

## Q1 — Does the document classifier reuse an existing model or retrain during the same test run, and how does that affect later tests?

**Direct answer.** Within a single run the classifier **reuses** the already-persisted model whenever the training data is unchanged, and **retrains** (rewriting the model file) whenever the training data changes. The decision is made by `DocumentClassifier.train()` comparing a **SHA-1 digest** of the training data against the stored digest: on a match it **short-circuits and returns `False`** (reuse, model file untouched); otherwise it returns `True` and the caller re-persists the pickle. **Effect on later tests:** because the persisted model is a *filesystem* artifact, if a test's `MODEL_FILE` is **not** isolated per-test, one test's trained model **persists into later tests** — under pytest-xdist parallel workers this shared state is the plausible root cause of run-to-run classification failures (fully analyzed in the closing section).

### Cause → effect (mechanism)

- `DocumentClassifier.train()` (`src/documents/classifier.py:L115`) builds a SHA-1 digest — `m = hashlib.sha1()` (`classifier.py:L124`) — over the preprocessed content and label ids of every `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`classifier.py:L125-L127`), finalising `new_data_hash = m.digest()` (`classifier.py:L161`).
- The **reuse short-circuit** is `if self.data_hash and new_data_hash == self.data_hash: return False` (`src/documents/classifier.py:L163-L164`). When the data differs it proceeds, sets `self.data_hash = new_data_hash` (`classifier.py:L247`) and `return True` (`classifier.py:L249`) — a **retrain**. The feature pipeline is `CountVectorizer(...)` (`classifier.py:L194-L199`) feeding `MLPClassifier(tol=0.01)` (`classifier.py:L219`, `L227`, `L238`); persisted pickles carry `FORMAT_VERSION = 7` (`classifier.py:L63`).
- The task wrapper `train_classifier()` (`src/documents/tasks.py:L48-L72`) loads the classifier via `load_classifier()` (`tasks.py:L57`; returns `None` when `MODEL_FILE` is absent — `classifier.py:L30-L36`) and **saves the pickle only when `train()` returns `True`**, logging `"Saving updated classifier model to {}..."` then `classifier.save()` (`tasks.py:L63-L67`); otherwise it logs **`"Training data unchanged."`** (`tasks.py:L69`).

### Observed evidence — canonical test (`test_train_classifier`)

`test_train_classifier` (`src/documents/tests/test_tasks.py:L75-L94`) asserts the model file is absent, then trains three times: create → `train_classifier()` (retrain, file created), a second identical `train_classifier()` (reuse, `assertEqual(mtime, mtime2)`), then edits `doc.content="test2"` and trains again (retrain, `assertNotEqual(mtime2, mtime3)`). Running it with DEBUG logging shows the exact reuse/retrain log sequence:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 "documents/tests/test_tasks.py::TestTasks::test_train_classifier" \
 -o addopts="" -p no:cacheprovider -o log_cli=true --log-cli-level=DEBUG -s'
```

```
[INFO]  paperless.tasks:tasks.py:64 Saving updated classifier model to /tmp/tmpbgwtsyc6/classification_model.pickle...   <- call 1: RETRAIN (file created)
DEBUG   paperless.tasks:tasks.py:69 Training data unchanged.                                                             <- call 2: REUSE (train() returned False)
[INFO]  paperless.tasks:tasks.py:64 Saving updated classifier model to /tmp/tmpbgwtsyc6/classification_model.pickle...   <- call 3: RETRAIN (after doc.content edit)
```

The middle call logs **`Training data unchanged.`** (`tasks.py:L69`) — the reuse branch — flanked by two `Saving updated classifier model` lines (`tasks.py:L64`) — the retrain branch. The test passes on two identical runs:

```
Run #1:  1 passed, 6 warnings in 2.11s
Run #2:  1 passed, 6 warnings in 1.95s
```

Note the `MODEL_FILE` resolved to a **per-test** temp path `/tmp/tmpbgwtsyc6/classification_model.pickle` — this is the `DirectoriesMixin` isolation discussed under "cross-test effect" below.

### Observed evidence — direct mtime + `train()` boolean (temporary observation script)

To observe the `MODEL_FILE` state **before / during / after** and the raw boolean from `train()`, a temporary script drove the **real** `train_classifier()` task and the **real** `DocumentClassifier.train()` under `DirectoriesMixin`:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q1.py -o addopts="" -p no:cacheprovider -s'
```

Run #1 (complete, unedited observation lines):

```
===== Q1 OBSERVATION (reuse vs. retrain) =====
MODEL_FILE = /tmp/tmpqizdxkjs/classification_model.pickle
model exists BEFORE any train : False
[call1 train_classifier()] model exists : True | mtime_ns = 1784057810368301139
[call2 train_classifier() identical] mtime_ns = 1784057810368301139 | mtime UNCHANGED vs call1 : True
[call3 train_classifier() after content change] mtime_ns = 1784057810427301774 | mtime CHANGED vs call2 : True
[direct DocumentClassifier.train()] first call returned : True (True => RETRAIN)
[direct DocumentClassifier.train()] second call returned: False (False => REUSE, SHA-1 data_hash unchanged)
[persisted] load_classifier().train() returned : False (False => REUSE via persisted data_hash)
```

Run #2 (identical inputs) reproduced the same relationships (`call1 == call2` mtime; `call2 != call3`; direct `train()` returns `True` then `False`); only the absolute mtimes differ because each test gets a fresh temp dir:

```
model exists BEFORE any train : False
[call1 train_classifier()] model exists : True | mtime_ns = 1784057825384462903
[call2 train_classifier() identical] mtime_ns = 1784057825384462903 | mtime UNCHANGED vs call1 : True
[call3 train_classifier() after content change] mtime_ns = 1784057825441463517 | mtime CHANGED vs call2 : True
[direct DocumentClassifier.train()] first call returned : True (True => RETRAIN)
[direct DocumentClassifier.train()] second call returned: False (False => REUSE, SHA-1 data_hash unchanged)
[persisted] load_classifier().train() returned : False (False => REUSE via persisted data_hash)
```

This directly demonstrates both branches: **reuse** = `train()` → `False`, mtime unchanged, `"Training data unchanged."` logged; **retrain** = `train()` → `True`, pickle rewritten, mtime advances.

### Observed evidence — hashing / save-reuse canonical tests

`testDatasetHashing` (`test_classifier.py:L137-L142`) asserts `assertTrue(train())` then `assertFalse(train())` on identical data (retrain then reuse). `testSaveClassifier` (`test_classifier.py:L167-L178`; decorated `@override_settings(DATA_DIR=tempfile.mkdtemp())` at L167 — it pins `DATA_DIR`, **not** `MODEL_FILE`) trains, saves, loads a fresh classifier, and `assertFalse(new_classifier.train())` (reuse via the persisted `data_hash`). `test_load_and_classify` (`test_classifier.py:L183`) is the test actually decorated `@override_settings(MODEL_FILE=…/data/model.pickle)` (`test_classifier.py:L180-L182`). All three pass:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 "documents/tests/test_classifier.py::TestClassifier::testDatasetHashing" \
 "documents/tests/test_classifier.py::TestClassifier::testSaveClassifier" \
 "documents/tests/test_classifier.py::TestClassifier::test_load_and_classify" \
 -o addopts="" -p no:cacheprovider -v'
```

```
documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED     [ 33%]
documents/tests/test_classifier.py::TestClassifier::testSaveClassifier PASSED     [ 66%]
documents/tests/test_classifier.py::TestClassifier::test_load_and_classify PASSED [100%]
======================== 3 passed, 6 warnings in 2.07s =========================
```

### Cross-test effect (how reuse/retrain affects later tests)

`DirectoriesMixin` (`src/documents/tests/utils.py`, class at L72; `setUp` → `setup_directories()` at L77-L78) overrides per-test directories and sets `MODEL_FILE=<data_dir>/classification_model.pickle` inside its `override_settings(...)` block (`utils.py:L45`). In every mixin-using run above, `MODEL_FILE` resolved to a unique `/tmp/tmpXXXX/classification_model.pickle` — so **isolation holds only for tests that use the mixin**. Tests that rely on the *default* `MODEL_FILE` (`src/paperless/settings.py:L74`) or pin a fixed path can **share** model state on disk. Because a trained pickle is a filesystem artifact (unlike DB rows, which `TestCase` rolls back), a model written by one test can be **loaded by a later test** — the reuse/retrain outcome of the later test then depends on what earlier tests left behind. Under the default `--numprocesses auto` this becomes a cross-worker race; the direct reproduction attempt and the mechanism demonstration are in the **Determinism** section at the end of this document.

---

## Q2 — In the test exercising automatic correspondent matching: (a) how many training documents are created, (b) when does training occur relative to those inserts, and (c) what confidence threshold accepts/rejects a prediction?

**Direct answer.**
- **(a) Training documents created: 3 documents are created, but only 2 are used for training.** The training query **excludes inbox-tagged documents**, and one of the three created documents carries an inbox tag, so it is dropped (3 created → **2 trained**).
- **(b) Training occurs *after* the inserts.** The documents are `Document.objects.create()`-d first; `DocumentClassifier.train()` is called **afterward**, then prediction runs — observed as 2 rows present in the DB before `train()` is invoked.
- **(c) There is NO probabilistic confidence threshold.** A correspondent prediction is accepted purely when the **predicted class id is not `-1`** (the null class); otherwise it is rejected (returns `None`). No numeric probability cutoff exists anywhere in the correspondent-prediction path. (The only `>= 90` in the code is an *unrelated* rule-based fuzzy **string**-matching threshold, not an ML confidence value.)

### (a) How many training documents are created

**Cause → effect.** The training set is `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:L125-L127`) — every document **except** those bearing a tag with `is_inbox_tag=True`. The correspondent-matching test data is built by `generate_test_data` (`src/documents/tests/test_classifier.py:L25-L82`), which creates `doc1`, `doc2`, and `doc_inbox`; `doc_inbox` receives tag `t2` whose `is_inbox_tag=True` (`test_classifier.py:L44`), so it is excluded — leaving 2 training documents.

Observed via a temporary script that faithfully replicates `generate_test_data` and counts the training query directly (`len` of `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)`):

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q2.py -o addopts="" -p no:cacheprovider -s'
```

```
===== Q2(a) TRAINING DOCUMENT COUNT =====
(a) total Document rows CREATED           : 3
(a) doc_inbox has an is_inbox_tag tag     : True
(a) training docs = exclude(inbox) COUNT  : 2
(a) training doc titles                   : ['doc1', 'doc1']
```

This count was identical across two identical runs (stable): **3 created → 2 trained**.

### (b) When training occurs relative to the inserts

**Cause → effect.** In `test_one_correspondent_predict` (`test_classifier.py:L191-L204`) and `test_one_correspondent_predict_manydocs` (`test_classifier.py:L206-L225`), documents are `create()`-d first, **then** `self.classifier.train()` is called, **then** `predict_correspondent(...)` runs. The same ordering was observed directly (2 rows already present in the DB when `train()` is called, which then returns `True` — a first-time train/retrain):

```
===== Q2(b)/(c) ORDERING + ACCEPTANCE RULE =====
(b) rows inserted BEFORE training         : 2
(b) clf.train() called AFTER inserts, ret : True
```

### (c) What confidence threshold accepts / rejects a prediction

**Direct answer (restated): there is no probabilistic confidence threshold.** `predict_correspondent` (`src/documents/classifier.py:L251-L260`) transforms the content, obtains a predicted class id, and applies exactly one rule — `if correspondent_id != -1: return correspondent_id else: return None` (`src/documents/classifier.py:L255-L258`). Acceptance is purely **"class id `!= -1`"**; the `-1` null class is rejected as `None`. Observed accepted vs. rejected predictions:

```
(c) predict_correspondent(doc1) = [1] | c1.pk = 1 | ACCEPTED (id != -1): [ True]
(c) predict_correspondent(doc2) = None | REJECTED (None, i.e. class id == -1): True
```

`doc1` (content "this is a document from c1", correspondent `c1` which is `MATCH_AUTO`) predicts `c1.pk` and is **accepted**; `doc2` ("this is a document from noone", no correspondent) predicts the null class and is **rejected** as `None`. (The predicted id is returned as a NumPy array, e.g. `[1]`, so the elementwise comparison prints `[ True]`.)

**No ML probability cutoff exists — proof by exhaustive grep.** The only `>= 90` in the `documents/` package is the **rule-based** fuzzy string matcher, unrelated to ML classifier confidence:

```
docker exec --user testuser paperless-canon bash -lc 'cd /app/src && grep -rn ">= 90\|>=90" documents/*.py'
```

```
documents/matching.py:135:        if fuzz.partial_ratio(match, text) >= 90:
```

That `fuzz.partial_ratio(match, text) >= 90` lives inside `matches()` (`src/documents/matching.py:L60-L152`) and governs the **`MATCH_FUZZY`** *string* algorithm — it compares a rule's keyword against document text, and has nothing to do with the MLP classifier's confidence. `predict_correspondent` contains no numeric threshold at all (its full body, re-displayed from the run, shows only the `!= -1` rule).

### Reproduction targets and dispatch nuance

The canonical automatic-correspondent-matching-via-signal tests are `test_correspondent_applied` (`src/documents/tests/test_matchables.py:L425`) and `test_correspondent_not_applied` (`test_matchables.py:L437`), in class `TestDocumentConsumptionFinishedSignal` (`test_matchables.py:L380`, which extends plain `TestCase` — **not** `DirectoriesMixin`). **Nuance:** these two tests exercise the auto-matching **dispatch** via the `document_consumption_finished` signal using a **rule-based** `Correspondent(match="keyword", matching_algorithm=MATCH_ANY)` and send the signal **without a classifier** (so `classifier=None` inside `match_correspondents`); they demonstrate the signal→matching **wiring**, *not* the ML "class id != -1" rule. The ML acceptance rule is the `test_classifier.py` predict evidence shown above. All four canonical tests pass:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 "documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict" \
 "documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs" \
 "documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_applied" \
 "documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_not_applied" \
 -o addopts="" -p no:cacheprovider -v'
```

```
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED
documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_applied PASSED
documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_not_applied PASSED
(2 passed + 2 passed across the two invocations)
```

The dispatch chain is `set_correspondent` (`src/documents/signals/handlers.py:L35-L50`) → `matching.match_correspondents(document, classifier)` (`handlers.py:L50`); `match_correspondents` (`src/documents/matching.py:L21-L31`) returns every correspondent for which `matches(o, document) or o.pk == pred_id` (`matching.py:L30`) — i.e. **rule match OR ML prediction**.

---

## Q3 — For a document with no extractable text: (a) which OCR subprocess is invoked, and (b) what MIME type is assigned to the output?

**Direct answer.**
- **(a) OCR subprocess:** **OCRmyPDF is invoked via `ocrmypdf.ocr(**args)`**, which drives **Tesseract**. On a no-text document the *primary* OCRmyPDF call (with `skip_text=True`) yields no text and raises `NoTextFoundException`; that triggers a **force-OCR fallback** — a **second** `ocrmypdf.ocr(**args)` with `force_ocr=True` — after which empty text is accepted (`self.text = ""`). Ground-truth subprocesses observed spawning during parsing were **`tesseract`, `unpaper`, and `gs` (Ghostscript)**; thumbnail generation additionally attempts ImageMagick **`convert`** and falls back to Ghostscript **`gs`**. (For the RGBA image fixture, an alpha-layer-removal step runs before OCR.)
- **(b) MIME type assigned to the output:** **`image/png`** — the **original file's** MIME as detected by **libmagic** (`magic.from_file(..., mime=True)`) and stored on the `Document`. It is **not** the produced archive PDF's `application/pdf`.

### The no-text fixture

The exact Q3 fixture is `src/paperless_tesseract/tests/samples/no-text-alpha.png` — a `PNG image data, 297 x 225, 8-bit/color RGBA, non-interlaced` file. It is **not referenced by any canonical test** (verified by repo-wide grep), so it was exercised through the **real** `RasterisedDocumentParser.parse(...)` and the **real** `Consumer.try_consume_file(...)` via temporary scripts:

```
docker exec --user testuser paperless-canon bash -lc 'cd /app/src && grep -rn "no-text-alpha" . | grep -v "\.png:"'
# (no matches — the fixture is not used by any .py code)
```

The canonical no-text/empty-text parser tests do pass, establishing the surrounding behaviour: `test_encrypted` (`src/paperless_tesseract/tests/test_parser.py:L178`, asserts `get_text() == ""`), `test_with_form_error_notext` (`test_parser.py:L190`, `@override_settings(OCR_MODE="redo")`), and `test_skip_noarchive_notext` (`test_parser.py:L370`, `@override_settings(OCR_MODE="skip_noarchive")`):

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 "paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted" \
 "paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext" \
 "paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext" \
 -o addopts="" -p no:cacheprovider -v'
```

```
paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted PASSED             [ 33%]
paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED [ 66%]
paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED [100%]
======================== 3 passed, 6 warnings in 9.83s =========================
```

### (a) Which OCR subprocess is invoked — the full ordered chain

**Cause → effect.** Inside `RasterisedDocumentParser.parse()` (`src/paperless_tesseract/parsers.py:L230`) the parameters are built by `construct_ocrmypdf_parameters(...)` (`parsers.py:L135`). With the default `OCR_MODE="skip"` (`src/paperless/settings.py:L522`) this sets `skip_text=True` (`parsers.py:L157-L158`) and, because the fixture is RGBA, the alpha layer is removed first (`parsers.py:L191-L201`). The **primary** OCR call is `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L261`). If no text results, `NoTextFoundException` (defined `parsers.py:L14`) is raised (`parsers.py:L266-L267`), which triggers the **force-OCR fallback**: `construct_ocrmypdf_parameters(..., safe_fallback=True)` (sets `force_ocr=True`, `parsers.py:L155-L156`) followed by a **second** `ocrmypdf.ocr(**args)` (`parsers.py:L298`). If text is still empty, empty text is accepted at `self.text = ""` (`parsers.py:L318-L327`, specifically `L327`).

Observed by parsing `no-text-alpha.png` through the real parser with DEBUG logging **and** a `sys.addaudithook` capturing every `subprocess.Popen` (ground truth of which binaries fired):

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q3.py -o addopts="" -p no:cacheprovider \
 -o log_cli=true --log-cli-level=DEBUG -s'
```

```
Q3b_MIME libmagic magic.from_file(no-text-alpha.png, mime=True) = image/png
[INFO]    paperless.parsing.tesseract  Removing alpha layer from .../no-text-alpha.png for compatibility with img2pdf
[DEBUG]   paperless.parsing.tesseract  Calling OCRmyPDF with args: {'input_file': '.../no-text-alpha.png', 'output_file': '.../archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '.../sidecar.txt', 'image_dpi': 35}
[WARNING] paperless.parsing.tesseract  Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[DEBUG]   paperless.parsing.tesseract  Fallback: Calling OCRmyPDF with args: {'input_file': '.../no-text-alpha.png', 'output_file': '.../archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '.../sidecar-fallback.txt', 'image_dpi': 35}
[WARNING] paperless.parsing.tesseract  No text was found in .../no-text-alpha.png, the content will be empty.
Q3a_SUBPROC executables spawned during parse(): ['tesseract', 'unpaper', 'gs']
Q3a_TEXT parser.text (repr)  = ''
Q3a_TEXT parser.text is empty string: True
Q3a_ARCHIVE archive_path     = /tmp/tmpppubm_90/paperless-_o2z8hdj/archive.pdf
Q3a_ARCHIVE archive exists   : True
Q3b_ARCHIVE_MIME libmagic of produced archive = application/pdf
```

This is the complete ordered chain, each branch exercised on the real fixture:
1. **Alpha-layer removal** — `Removing alpha layer …` (`parsers.py:L191-L201`).
2. **Primary OCR** — `Calling OCRmyPDF with args: {... 'skip_text': True ...}` → `ocrmypdf.ocr(**args)` (`parsers.py:L261`).
3. **NoTextFoundException** — `… No text was found in the original document. Attempting force OCR …` (raised at `parsers.py:L266-L267`).
4. **Force-OCR fallback** — `Fallback: Calling OCRmyPDF with args: {... 'force_ocr': True ...}` → second `ocrmypdf.ocr(**args)` (`parsers.py:L298`).
5. **Empty-text acceptance** — `… the content will be empty.` → `self.text = ""` (`parsers.py:L327`); observed `parser.text == ''`.

**Ground-truth subprocesses** spawned during `parse()`: `['tesseract', 'unpaper', 'gs']` — OCRmyPDF drives **Tesseract** (the OCR engine), **`unpaper`** (image cleanup, from `clean=True` / `OCR_CLEAN="clean"` at `src/paperless/settings.py:L526`), and **`gs`** (Ghostscript, for the `output_type='pdfa'` PDF/A archive). The sidecar-based check that distinguishes a genuinely OCR'd page from a text-layer-skipped page is `if "[OCR skipped on page" not in text:` in `extract_text` (`src/paperless_tesseract/parsers.py:L104`).

**Supporting thumbnail subprocess (observed):** during full consumption, thumbnail generation attempted ImageMagick `convert` and then fell back to Ghostscript, logged as `Thumbnail generation with ImageMagick failed, falling back to ghostscript.` — this is `make_thumbnail_from_pdf_gs_fallback` (`src/documents/parsers.py:L153`, `gs` invoked with `-sDEVICE=pngalpha` at `L164`), the fallback for `run_convert` (`documents/parsers.py:L111`, `CONVERT_BINARY` at `L132`). (In this container the primary `convert` is blocked by ImageMagick's PDF security policy — an environment detail — so the `gs` fallback path is what completes; **(inferred)** on a host without that policy restriction, ImageMagick `convert` would satisfy the thumbnail directly.)

### (b) What MIME type is assigned to the output

**Cause → effect.** MIME detection uses **libmagic** via `magic.from_file(self.path, mime=True)` in the consumer (`src/documents/consumer.py:L219`; the base-parser variant `get_parser_class` uses `magic.from_file(path, mime=True)` at `src/documents/parsers.py:L106`). The detected `mime_type` is stored on the `Document` at `Document.objects.create(..., mime_type=mime_type, ...)` in `_store` (`src/documents/consumer.py:L401`). For an input image this is the **original file's** MIME — `image/png` — not the archive PDF that OCRmyPDF produces.

Observed by consuming `no-text-alpha.png` through the **real** `Consumer.try_consume_file(...)` (real parser; only the websocket `_send_progress` mocked) and reading the stored value off the created `Document`:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q3consume.py -o addopts="" -p no:cacheprovider -s'
```

```
===== Q3(b) CONSUMER-STORED Document.mime_type (REAL Consumer, REAL parser) =====
Q3b_INPUT libmagic of input file          = image/png
Q3b_DOC  Document.mime_type STORED         = image/png
Q3b_DOC  Document.content (repr)           = ''
Q3b_DOC  Document has archive_path         : True
Q3b_DOC  libmagic of the produced ARCHIVE  = application/pdf
```

The **`Document.mime_type` stored is `image/png`** (identical across two runs), matching the libmagic result on the original input and confirming `consumer.py:L219` → `consumer.py:L401`. The document's extracted `content` is `''` (consistent with Q3(a)). The pipeline *does* produce a PDF/A archive whose own MIME is `application/pdf`, but that is the `archive_path`, **not** the value assigned as the `Document`'s MIME type.

---

## Q4 — For barcode splitting: (a) how many document records are created from a single input, (b) which barcode values trigger a split, (c) where is the decision made, and (d) does this change the effective training data during the run?

**Direct answer.**
- **(a) Records from a single input = (number of separators + 1).** `separate_pages` emits one segment before the first separator plus one per subsequent separator, **dropping the separator page itself**. Observed: a single-separator input `[1]` → **2** segments; a two-separator input `[2, 5]` → **3** segments. Importantly, **`consume_file` itself creates *zero* `Document` rows** — it writes the split segments to the consumption directory and returns the string `"File successfully split"`; the actual `Document` records appear only when those segments are subsequently consumed.
- **(b) Trigger value: `"PATCHT"`** — the default of `settings.CONSUMER_BARCODE_STRING`. A page splits when a decoded barcode's value **equals** this string. It is symbology-agnostic: observed triggering across **Code 39, Code 128, and QR** barcodes all carrying the value `PATCHT`. An **unreadable** barcode decodes to nothing and does not trigger a split. The trigger string is configurable (`CONSUMER_BARCODE_STRING`), and barcode splitting overall is gated by `CONSUMER_ENABLE_BARCODES`.
- **(c) The split decision is made in `scan_file_for_separating_barcodes`** — specifically `if separator_barcode in current_barcodes: separator_page_numbers.append(current_page_number)` at `src/documents/tasks.py:L108-L109`.
- **(d) Yes — indirectly.** Splitting a file does not itself create `Document` rows, but once the resulting segments are consumed they become **additional** `Document` rows. More rows change the SHA-1 `data_hash` computed by `train()`, which **forces a retrain** on the next `train_classifier()` — the exact mechanism from Q1.

The full canonical barcode family passes (25 tests):

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest documents/tests/test_tasks.py \
 -o addopts="" -p no:cacheprovider -k "barcode or separating or separate_pages" -v'
```

```
... (test_barcode_reader*, test_scan_file_for_separating_barcodes*, test_separate_pages,
     test_barcode_splitter, test_consume_barcode_file all PASSED) ...
================ 25 passed, 15 deselected, 6 warnings in 5.27s =================
```

### (a) How many document records result from a single input file

**Cause → effect.** `separate_pages` (`src/documents/tasks.py:L113-L161`) writes `{fname}_document_0.pdf` for the pages before the first separator (`tasks.py:L129-L138`), then one file per subsequent separator, skipping each separator page via `for page in range(page_number + 1, next_page)` (`tasks.py:L148-L149`). The canonical `test_separate_pages` (`src/documents/tests/test_tasks.py:L305-L313`) asserts `len(separate_pages(patch-code-t-middle.pdf, [1])) == 2`. Observed directly for both single- and multi-separator inputs:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q4.py -o addopts="" -p no:cacheprovider -s'
```

```
===== Q4(a)/(c) DECISION SITE + RECORD COUNT =====
  scan_file_for_separating_barcodes(patch-code-t-middle.pdf)  = [1]     (single separator)
  scan_file_for_separating_barcodes(several-patcht-codes.pdf) = [2, 5]  (multi separator)
  separate_pages(single,[1]) -> 2 records (separators+1=2): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
  separate_pages(multi ,[2, 5]) -> 3 records (separators+1=3): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
```

**`consume_file` creates no `Document` rows.** `consume_file` (`src/documents/tasks.py:L184-L233`) scans for separators, calls `separate_pages`, writes each segment to the consumption dir (`save_to_dir`), unlinks the original, and returns `"File successfully split"` (`tasks.py:L233`) — it does **not** create any `Document`. Observed (with `CONSUMER_ENABLE_BARCODES=True`), counting `Document` rows before and after:

```
===== Q4(a) consume_file: 'File successfully split' + NO Document rows =====
  Document rows BEFORE consume_file = 0
  consume_file(...) returned        = 'File successfully split'
  Document rows AFTER consume_file  = 0  <- consume_file itself creates NO rows
```

(A harmless `Connect call failed … 6379` Redis-broker warning appears because the progress notification tries to reach the broker; the `OSError` is caught at `tasks.py:L230-L232` and the function still returns `"File successfully split"`.)

### (b) Which barcode values trigger a split

**Cause → effect.** The separator value is `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` (read in `scan_file_for_separating_barcodes`, `tasks.py:L102`), whose default is `"PATCHT"` (`src/paperless/settings.py:L506`); the whole feature is gated by `CONSUMER_ENABLE_BARCODES` (`src/paperless/settings.py:L502`, default off). Barcodes are decoded by `barcode_reader` via `pyzbar.decode(image)` (`src/documents/tasks.py:L82`). Observed trigger value and symbologies (the decoded `.type` from pyzbar alongside the real `tasks.barcode_reader`), including the unreadable edge case:

```
===== Q4(b) TRIGGER VALUES + SYMBOLOGIES =====
  barcode-39-PATCHT.png      symbology=['CODE39']  value=['PATCHT'] barcode_reader=['PATCHT']
  barcode-128-PATCHT.png     symbology=['CODE128'] value=['PATCHT'] barcode_reader=['PATCHT']
  qr-code-PATCHT.png         symbology=['QRCODE']  value=['PATCHT'] barcode_reader=['PATCHT']
  barcode-39-PATCHT-unreadable.png barcode_reader=[]   <- UNREADABLE => no split
```

So the **value** `PATCHT` triggers the split regardless of symbology — verified across **Code 39** (`barcode-39-PATCHT.png`), **Code 128** (`barcode-128-PATCHT.png`), and **QR** (`qr-code-PATCHT.png`); an **unreadable** barcode (`barcode-39-PATCHT-unreadable.png`) decodes to `[]` and yields no split. The trigger is configurable — observed a custom value matching a custom fixture while the default `PATCHT` does **not** match it:

```
===== Q4(b) CUSTOM string 'CUSTOM BARCODE' matches custom fixture =====
  CONSUMER_BARCODE_STRING='CUSTOM BARCODE' scan(barcode-39-custom.pdf) = [0]
===== Q4(b) DEFAULT 'PATCHT' does NOT match custom fixture =====
  default 'PATCHT' scan(barcode-39-custom.pdf) = []  <- [] no match
```

Single- vs. multi-separator behaviour was also observed above: `patch-code-t-middle.pdf` → `[1]` (single) and `several-patcht-codes.pdf` → `[2, 5]` (multi).

### (c) Where the split decision is made

**Cause → effect.** The decision lives in `scan_file_for_separating_barcodes` (`src/documents/tasks.py:L96-L110`): pages are rasterized with `convert_from_path(...)` (poppler, `tasks.py:L105`), decoded by `barcode_reader` (`tasks.py:L82`), and the split point is recorded by `if separator_barcode in current_barcodes: separator_page_numbers.append(current_page_number)` at **`src/documents/tasks.py:L108-L109`**. The list of separator page numbers observed above (`[1]`, `[2, 5]`, `[0]`, `[]`) is exactly this function's return value.

### (d) Does this change the effective training data during the run?

**Direct answer: yes, indirectly.** `consume_file` creates no rows itself (shown in (a)), but the written segments, **once consumed**, become additional `Document` rows. Additional rows change the SHA-1 `data_hash` computed over `Document.objects...exclude(tags__is_inbox_tag=True)` in `train()` (`src/documents/classifier.py:L125-L161`), which flips the reuse short-circuit (`classifier.py:L163-L164`) and **forces a retrain** on the next `train_classifier()`. Observed the training-data → retrain chain directly (add a row, watch `train()` switch from reuse back to retrain):

```
===== Q4(d) more Document rows => SHA-1 changes => RETRAIN (links to Q1) =====
  train() with 1 doc  -> True (True=retrain)
  train() again, unchanged data       -> False (False=reuse)
  after adding a row, now 2 docs; train() -> True (True=RETRAIN forced by changed training data)
```

This is the same SHA-1 reuse/retrain mechanism established in Q1: barcode splitting → more consumed segments → more `Document` rows → changed training hash → retrain. All Q4 observations above were identical across two identical runs (stable).

---

## Closing — The pytest-xdist + shared `MODEL_FILE` determinism hypothesis

**Hypothesis.** The reported non-deterministic failures in the document-classification test suite are most plausibly caused by the combination of **massively parallel pytest-xdist execution** (`--numprocesses auto`) and a **shared/non-isolated `MODEL_FILE`** across tests that do not use `DirectoriesMixin`. The classifier's model is a **filesystem** artifact whose reuse-vs-retrain outcome depends on what already exists on disk (Q1); when two tests share one `MODEL_FILE` path and at least one of them writes it, a cross-worker read/write race can make a later test's classification assertion pass or fail run-to-run.

### What was directly observed

**1. xdist parallelism is real and large.** `--numprocesses auto` resolves to the host CPU count (`nproc = 128`), and pytest-xdist spins up that many workers:

```
docker exec --user testuser paperless-canon bash -lc 'nproc'          # -> 128
# with -v: "created: 128/128 workers" ; "128 workers [15 items]"
```

**2. Repeated identical runs — the classification suite was STABLE here.** Running the classification modules together five times under the default `--numprocesses auto` produced an identical result every time (reproduce-by-repetition, not stabilize-by-variant):

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 documents/tests/test_classifier.py documents/tests/test_tasks.py documents/tests/test_matchables.py \
 --no-cov -p no:cacheprovider'
```

```
RUN 1 (xdist -n auto): 77 passed, 1 skipped, 774 warnings in 26.60s
RUN 2 (xdist -n auto): 77 passed, 1 skipped, 774 warnings in 26.67s
RUN 3 (xdist -n auto): 77 passed, 1 skipped, 774 warnings in 27.19s
RUN 4 (xdist -n auto): 77 passed, 1 skipped, 774 warnings in 27.26s
RUN 5 (xdist -n auto): 77 passed, 1 skipped, 774 warnings in 26.57s
```

The pinned serial baseline (`-n0`, labelled non-default) was likewise stable and much faster, confirming the ~26 s under `-n auto` is 128-worker startup overhead:

```
RUN 1 (serial -n0): 77 passed, 1 skipped, 6 warnings in 7.11s
RUN 2 (serial -n0): 77 passed, 1 skipped, 6 warnings in 7.22s
RUN 3 (serial -n0): 77 passed, 1 skipped, 6 warnings in 7.20s
```

**Honest result:** across 5 identical parallel runs + 3 identical serial runs, the reported non-determinism **did not reproduce** in these modules.

**3. Why it did not reproduce here (observed).** Two structural facts explain the stability:
- `TestClassifier` (`src/documents/tests/test_classifier.py:L20`) and `TestTasks` (`src/documents/tests/test_tasks.py:L21`) use **`DirectoriesMixin`**, so each test gets an **isolated** `MODEL_FILE=/tmp/tmpXXXX/classification_model.pickle` (`src/documents/tests/utils.py:L45`) — no sharing possible.
- The plain-`TestCase` matching tests — `TestDocumentConsumptionFinishedSignal` (`test_matchables.py:L380`), `_TestMatchingBase` (`test_matchables.py:L19`), `TestCaseSensitiveMatching` (`test_matchables.py:L216`) — do use the **non-isolated default** `MODEL_FILE = <DATA_DIR>/classification_model.pickle` (`src/paperless/settings.py:L74`), but they send the `document_consumption_finished` signal **without a classifier**, so they never *write* the shared model. Confirmed by watching the shared file across a full matchables run:

```
F=/app/data/classification_model.pickle
before: exists=no
after : exists=no      # matchables tests pass classifier=None -> never write the shared model
```

**4. The leakage mechanism, directly demonstrated (labelled characterization).** To show the mechanism that *would* drive non-determinism, a temporary plain-`TestCase` (no `DirectoriesMixin`) using a single fixed shared `MODEL_FILE` had one test train+save and a *later* test (which trains nothing, and whose DB rows are rolled back by `TestCase`) attempt to load a classifier:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest \
 /tmp/thq/blitzy_adhoc_test_q6.py --no-cov -p no:cacheprovider -o addopts="-n0" -s'
```

```
[char] test_a: after train_classifier(), shared MODEL_FILE exists = True
[char] test_z: (no training here) load_classifier() is None = False | leftover model from PRIOR test visible = True
```

A model written by `test_a` is still visible to `test_z` — because the pickle is a **filesystem** artifact that (unlike the DB transaction) is **not** rolled back between tests. This is the cross-test model leakage; under 128 parallel workers sharing one on-disk path, the read/write interleaving is a genuine race.

### Conclusion

- **Directly observed:** the reuse/retrain SHA-1 mechanism (Q1); the 128-worker xdist configuration; the split between mixin-isolated and non-isolated `MODEL_FILE` tests; and the cross-test model-file leakage mechanism.
- **Inferred (not reproduced here):** that in the *broader* suite — where non-isolated tests that *do* write the default `MODEL_FILE` can interleave with classifier assertions across 128 workers — this shared-state race is the root cause of the reported run-to-run classification failures. The isolated classification modules tested above did not trigger it because they either isolate `MODEL_FILE` (`DirectoriesMixin`) or never write the shared file (`classifier=None`). **(inferred)**

**Recommended framing for maintainers (documentation only, no code change per scope):** ensuring every classification-touching test uses `DirectoriesMixin` (or otherwise isolates `MODEL_FILE` per test/worker) would remove the shared on-disk model state that this investigation identifies as the determinism risk surface.

---

## Appendix — Read-Only Invariant (source tree unchanged)

This investigation is read-only against the paperless-ngx source tree. All runtime observation used the canonical container; every temporary observation script (`/tmp/thq/blitzy_adhoc_test_q1.py`, `…_q2.py`, `…_q3.py`, `…_q3consume.py`, `…_q4.py`, `…_q6.py`) lived outside both source trees and was deleted after use. The only change introduced to the destination repository is this single answer document.

Final `git status` of the destination repository (captured after authoring and cleanup, before committing the deliverable):

```console
$ git status
On branch blitzy-be1effe9-f4e7-4a8e-a3a8-33c507232b12
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	blitzy/

nothing added to commit but untracked files present (use "git add" to track)

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/paperless-ngx_542221a38dff.md

$ git status --porcelain --untracked-files=no | wc -l   # tracked-file modifications
0
```

The only entry is the untracked deliverable `blitzy/documentation/paperless-ngx_542221a38dff.md`; the count of modifications to tracked files is `0`, so no source, test, configuration, or fixture file was added, modified, or deleted. The canonical test-runner checkout at `/app` inside the container was independently confirmed pristine (`git status --porcelain` → 0 lines) after restoring the one binary fixture (`src/paperless_tesseract/tests/samples/no-text-alpha.png`) that OCRmyPDF/PIL rewrote in place during the Q3 parse; that restoration returns the source tree to byte-for-byte identical with `HEAD`.

Commit under investigation (unchanged): `542221a38dff06361e07976452f9aea24d210542`.
