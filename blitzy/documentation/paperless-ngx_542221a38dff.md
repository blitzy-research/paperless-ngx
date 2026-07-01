# paperless-ngx — Classification, OCR & Barcode Pipeline Behavior (Runtime-Observed Q&A)

> **Branch (source):** `paperless-ngx_542221a38dff` · **Commit:** `542221a38` · **Runtime:** Python 3.9 Docker container · **scikit-learn:** `1.0.2` (hard pin)
>
> **Purpose:** Explain — from *directly observed runtime behavior* — how the paperless-ngx document‑classification, OCR, and barcode‑splitting pipelines behave while the test suite runs, so that the reported **non‑deterministic (flaky) test failures in document classification** can be diagnosed.
>
> **How this document was produced:** Every behavioral claim below was captured by **running code inside the supplied Docker container** (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`, Python 3.9). The local host shell cannot run this code (Django + scikit‑learn are absent), so the container is authoritative. Each claim is paired with **its own verbatim observed output** and the exact command/code that produced it, plus an exact `file:line` citation. Temporary observation scripts (`blitzy_adhoc_test_*`) were used during investigation and **removed afterward**; the source tree is left unchanged (read‑only mandate).
>
> **Report‑exactly‑what‑is‑observed note:** Where an observed value differs from what one might expect (for example, the non‑determinism section below), the observed value is reported verbatim and unadjusted.

---

## 0. Runtime foundation (evidence)

All commands run from `/app/src` inside the container as user `testuser`, with `DJANGO_SETTINGS_MODULE=paperless.settings` and `PAPERLESS_DISABLE_DBHANDLER=true` (both set in `src/setup.cfg:9,12`). Targeted tests use `-o addopts=""` to strip the repo default `addopts` (coverage + xdist) for clean, deterministic, single‑process output; the default `--numprocesses auto` is retained only when reproducing parallel behavior.

**Claim — the runtime is the pinned container at commit `542221a38`, in a detached‑HEAD checkout.**

```
$ git branch --show-current        # (empty output)
$ git rev-parse --short HEAD
542221a38
$ git status | head -1
HEAD detached at 542221a38
```

`git branch --show-current` prints **nothing** because the container checkout is a **detached HEAD** at `542221a38` (not on a named branch). The deliverable filename still follows the `<source_branch_name>.md` convention supplied by the task (`paperless-ngx_542221a38dff.md`).

**Claim — the pinned dependency versions are installed as required.**

```
$ python -c "import sklearn, django; print(sklearn.__version__, django.VERSION)"
1.0.2 (4, 0, 4, 'final', 0)
```

scikit‑learn `1.0.2` (`requirements.txt:88`) and Django `4.0.4` (`requirements.txt:38`) — matching the pins.

**Claim — the system libraries needed for OCR (Q3) and barcode decoding (Q4) are present.**

```
$ tesseract --version | head -1
tesseract 4.1.1
$ python -c "import pyzbar.pyzbar; print('pyzbar import OK')"
pyzbar import OK
$ pdftoppm -v 2>&1 | head -1
pdftoppm version 20.09.0
```

`tesseract-ocr 4.1.1` (OCR), `pyzbar 0.1.9` bound to `libzbar0` (barcodes), and `poppler-utils` `pdftoppm 20.09.0` (PDF rasterization) are all available.

**Claim — the pre‑trained classifier fixture used by the "load" control is present at the expected size.**

```
$ ls -l documents/tests/data/model.pickle
-rw-r--r-- 1 testuser testuser 156607 Feb 14 20:58 documents/tests/data/model.pickle
```

`156607` bytes — the `FORMAT_VERSION 7` model consumed by `test_load_and_classify`. The Python 3.9 base is declared at `Dockerfile:18` (`FROM python:3.9-slim-bullseye as main-app`).

---

## Q1 — Does the classifier reuse an existing serialized model or retrain within the same test run, and how does that decision affect subsequent tests?

**Answer:** Within a run, `DocumentClassifier.train()` decides *reuse vs. retrain* by comparing a **SHA‑1 `data_hash`** computed over the current, training‑eligible `Document` rows. If the newly computed hash equals the stored `self.data_hash`, `train()` short‑circuits and **returns `False`** (reuse — no retrain). If the eligible data changed, the hash differs, the model is retrained, `self.data_hash` is updated, and `train()` **returns `True`**. There is no separate serialized‑model reuse *across* tests in the normal case, because each test gets its own `MODEL_FILE` path and the DB is rolled back per test.

### Q1 mechanism (exact `file:line`)

- The hash accumulator is created in `train()` (def at `src/documents/classifier.py:115`):
  - `m = hashlib.sha1()` — `src/documents/classifier.py:124`
- The training set explicitly **excludes inbox‑tagged documents**:
  - `for doc in Document.objects.order_by("pk").exclude(` / `tags__is_inbox_tag=True,` / `):` — `src/documents/classifier.py:125-127`
- The digest and the reuse short‑circuit:
  - `new_data_hash = m.digest()` — `src/documents/classifier.py:161`
  - `if self.data_hash and new_data_hash == self.data_hash:` — `src/documents/classifier.py:163`
  - `return False` — `src/documents/classifier.py:164`
- The retrain path stores the new hash and returns `True`:
  - `self.data_hash = new_data_hash` — `src/documents/classifier.py:247`
  - `return True` — `src/documents/classifier.py:249`
- Model format and load behavior:
  - `FORMAT_VERSION = 7` — `src/documents/classifier.py:63`
  - `load_classifier()` returns `None` when the model file is absent: `if not os.path.isfile(settings.MODEL_FILE):` … `return None` — `src/documents/classifier.py:30-36`
  - `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` — `src/paperless/settings.py:74` (`DATA_DIR` at `settings.py:66`)
- Per‑test isolation (why a serialized model does not bleed across tests):
  - `DirectoriesMixin` — `src/documents/tests/utils.py:72`; `setup_directories()` — `utils.py:14`; fresh `tempfile.mkdtemp()` dirs — `utils.py:18-21`; `override_settings(... MODEL_FILE=os.path.join(dirs.data_dir, "classification_model.pickle") ...)` — `utils.py:45`.

### Q1 evidence — the canonical reuse test returns `True` then `False`

`testDatasetHashing` (`src/documents/tests/test_classifier.py:137`) asserts `assertTrue(self.classifier.train())` (`test_classifier.py:141`) then `assertFalse(self.classifier.train())` (`test_classifier.py:142`) — i.e. first call trains (`True`), second call on identical DB state reuses (`False`).

```
$ python3 -m pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    -o addopts="" -p no:cacheprovider -v -p no:warnings
documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED [100%]
============================== 1 passed in 1.96s ===============================
```

The `PASSED` marker confirms `train()` → `True` then `train()` → `False` on identical DB state.

### Q1 evidence — observed SHA‑1 hashes and boolean returns (train‑twice, then mutate)

A temporary probe (subclass of the real `TestClassifier`, reusing its `generate_test_data()` fixture and `DirectoriesMixin` isolation) instantiated `DocumentClassifier`, trained twice on identical DB state, then mutated the data (added one `Document`) and trained again — printing `self.data_hash.hex()` and the boolean each time:

```
Q1PROBE FORMAT_VERSION=7
Q1PROBE MODEL_FILE_exists=False
Q1PROBE load_classifier_returns=None
Q1PROBE total_documents=3
Q1PROBE eligible_documents=2
Q1PROBE TRAIN#1 return=True data_hash=b41ce39793cd61f391afe34e6e4de9b361c74447
Q1PROBE TRAIN#2 return=False data_hash=b41ce39793cd61f391afe34e6e4de9b361c74447
Q1PROBE IDENTICAL_HASH_1_vs_2=True
Q1PROBE total_documents_after_mutate=4
Q1PROBE TRAIN#3(after mutate) return=True data_hash=2f0b4d6e6243eb32723da65c40e0de4ec42e3e79
Q1PROBE HASH_CHANGED_2_vs_3=True
```

Reading these observed lines against the mechanism:

- **`FORMAT_VERSION=7`** matches `classifier.py:63`.
- **`MODEL_FILE_exists=False` → `load_classifier_returns=None`** confirms `load_classifier()` returns `None` when the model file is absent (`classifier.py:30-36`) — the fresh per‑test `MODEL_FILE` (DirectoriesMixin) has no serialized model.
- **`TRAIN#1 return=True`** with a 40‑hex‑character (`b41ce397…c74447`) digest. 40 hex chars = 20 bytes = **SHA‑1** (160 bits), confirming `hashlib.sha1()` (`classifier.py:124`) / `m.digest()` (`classifier.py:161`).
- **`TRAIN#2 return=False`** with the **identical** digest (`IDENTICAL_HASH_1_vs_2=True`) — the reuse short‑circuit at `classifier.py:163-164` fired because the eligible‑Document set was unchanged.
- **`TRAIN#3 return=True`** after adding one document (3 → 4 rows) with a **different** digest (`2f0b4d6e…3e79`, `HASH_CHANGED_2_vs_3=True`) — the changed training set forced a retrain (`classifier.py:247,249`).

### Q1 downstream effect on subsequent tests

Within a single run, once `train()` has populated `self.data_hash`, any *subsequent* `train()` on the **same** eligible‑Document set is a no‑op reuse (`return False`), while any change to that set (a new document, a changed correspondent/type/tag label, a deleted row) changes the SHA‑1 and forces a retrain (`return True`). Across tests, a serialized model does **not** normally bleed: `DirectoriesMixin` gives each test its own `MODEL_FILE` path (`utils.py:45`) and Django `TestCase` rolls back the DB, so reuse‑vs‑retrain is decided by `data_hash` equality over DB content, **not** by test order. (The within‑test weight‑initialization randomness that *is* a genuine flakiness vector is analyzed in the Non‑determinism section below.)

---

## Q2 — Automatic (classifier‑driven) correspondent matching

This question has three sub‑parts, each answered individually with its own evidence, followed by the consumption‑time chain and a required disambiguation from the regex tests.

### Q2(a) — How many training documents are created?

**Answer:** The canonical fixture `generate_test_data()` (`src/documents/tests/test_classifier.py:25`) creates **three** `Document` rows, but only **two** are training‑eligible — the inbox‑tagged document is excluded by `classifier.py:125` (`.exclude(tags__is_inbox_tag=True)`).

The three documents are:
- `doc1` — `content="this is a document from c1"`, correspondent `c1` — `test_classifier.py:60`
- `doc2` — `content="this is another document, but from c2"`, correspondent `c2` — `test_classifier.py:67`
- `doc_inbox` — `content="aa"` — `test_classifier.py:73`

`doc_inbox` carries the inbox tag `t2` (created with `is_inbox_tag=True` at `test_classifier.py:44`, added to `doc_inbox` at `test_classifier.py:82`), so it is excluded from training.

**Observed count** (temporary probe after building the fixture):

```
Q2PROBE total_documents_created=3
Q2PROBE training_eligible=2
```

`Document.objects.count()` → **3**; `Document.objects.exclude(tags__is_inbox_tag=True).count()` → **2**. (The Q1 probe independently observed the same `total_documents=3` / `eligible_documents=2`.)

### Q2(b) — When does training occur relative to those inserts?

**Answer:** Training runs **after** the document inserts. Both `testTrain` (`test_classifier.py:103`) and `testPredict` (`test_classifier.py:115`) call `self.generate_test_data()` (which performs the inserts) and *then* `self.classifier.train()`.

**Observed ordering** (probe printed a marker immediately before/after training, with inserts already done):

```
Q2PROBE about_to_train (inserts already done above)
Q2PROBE trained
```

The fixture inserts complete before `train()` is invoked. (In real consumption, classifier training is a separate task that runs after documents already exist in the database.)

### Q2(c) — What confidence threshold accepts or rejects a prediction?

**Answer:** **There is no numeric probability threshold.** Acceptance is decided solely by the **`class != -1` sentinel rule** inside `predict_correspondent` (def `src/documents/classifier.py:251`):

- `if correspondent_id != -1:` — `src/documents/classifier.py:255`
- `return correspondent_id` — `src/documents/classifier.py:256`
- `return None` — `src/documents/classifier.py:258`

The `-1` label is the "no automatic correspondent" sentinel.

**Observed — the classifier's classes and the accept/reject outcomes:**

```
Q2PROBE c1_pk=1 c2_pk=2
Q2PROBE correspondent_classes_=[-1, 1]
Q2PROBE tags_binarizer_classes_=[12, 45]
Q2PROBE predict_correspondent(doc1)=array([1]) (expect c1.pk=1)
Q2PROBE predict_correspondent(doc2)=None (expect None)
Q2PROBE raw_predict(doc1)=[1] raw_predict(doc2)=[-1]
```

- `correspondent_classes_=[-1, 1]` is exactly `[-1, c1.pk]` (asserted at `test_classifier.py:106-109`). This arises because, among the two training documents, `doc1.correspondent=c1` is `MATCH_AUTO` → label `c1.pk=1`, while `doc2.correspondent=c2` is **not** `MATCH_AUTO` → label `-1` (the label expansion happens in `train()`, defaulting `y = -1` unless the correspondent's `matching_algorithm == MatchingModel.MATCH_AUTO`).
- `predict_correspondent(doc1)=array([1])` — the predicted class is `1` (= `c1.pk`), which is `!= -1`, so it is **returned**. (Reported exactly as observed: the method returns a NumPy `array([1])`, not a scalar `1`; the test's `assertEqual(... , c1.pk)` still passes because `array([1]) == 1`.)
- `predict_correspondent(doc2)=None` — the raw predicted class for `doc2` is `[-1]` (the sentinel), so the `if correspondent_id != -1:` gate is false and the method returns `None`.

**Observed — there is no probability/threshold logic in the source:**

```
Q2PROBE predict_correspondent_has_predict_proba=False
Q2PROBE predict_correspondent_has_threshold_literal=False
Q2PROBE_SRC     def predict_correspondent(self, content):
Q2PROBE_SRC         if self.correspondent_classifier:
Q2PROBE_SRC             X = self.data_vectorizer.transform([preprocess_content(content)])
Q2PROBE_SRC             correspondent_id = self.correspondent_classifier.predict(X)
Q2PROBE_SRC             if correspondent_id != -1:
Q2PROBE_SRC                 return correspondent_id
Q2PROBE_SRC             else:
Q2PROBE_SRC                 return None
Q2PROBE_SRC         else:
Q2PROBE_SRC             return None
```

The full method source contains neither `predict_proba` nor any `threshold` literal — confirming acceptance is the categorical `predict()` output gated only by `!= -1`.

**Run markers** (both single‑run assertions pass):

```
documents/tests/test_classifier.py::TestClassifier::testTrain PASSED     [100%]
documents/tests/test_classifier.py::TestClassifier::testPredict PASSED   [100%]
```

`testPredict` asserts `predict_correspondent(doc1.content) == c1.pk` (`test_classifier.py:118-121`) and `predict_correspondent(doc2.content) == None` (`test_classifier.py:122`). (See the Non‑determinism section for repeated‑run behavior of these exact‑equality assertions.)

### Q2 — Supporting chain: how a prediction becomes an assigned correspondent during consumption

- `match_correspondents(document, classifier)` (`src/documents/matching.py:21`) calls `classifier.predict_correspondent(document.content)` (`matching.py:23`) and returns the correspondents where `matches(o, document) or o.pk == pred_id` (`matching.py:30`).
- During consumption, `set_correspondent` (def `src/documents/signals/handlers.py:35`) calls `matching.match_correspondents(document, classifier)` (`handlers.py:50`).
- The classifier is loaded **once** in the consumer: `classifier = load_classifier()` (`src/documents/consumer.py:292`), the document is stored via `self._store(text=text, date=date, mime_type=mime_type)` (`consumer.py:301`), and the classifier is passed to the post‑consume signal `document_consumption_finished.send(... classifier=classifier)` (`consumer.py:306-311`).

### Q2 — Disambiguation from the regex `MATCH_ANY` correspondent tests

The classifier‑driven auto‑matching above is **distinct** from the regex correspondent tests in `src/documents/tests/test_matchables.py`. Those tests drive the helper `_test_matching` (`test_matchables.py:20`), which calls `matching.matches(instance, doc)` (the regex/keyword path; `matches()` is defined at `matching.py:60` and uses the `re` module) across `(Tag, Correspondent, DocumentType)` (`test_matchables.py:28`) for algorithms such as `MATCH_ANY` (`test_matchables.py:101,117,134`) and `MATCH_REGEX` (`test_matchables.py:199`).

**Observed — `test_matchables.py` never touches the classifier:**

```
$ grep -nE "classifier|predict_correspondent|match_correspondents" documents/tests/test_matchables.py
   (no output)
```

The empty grep confirms that the `MATCH_ANY` correspondent tests exercise **regex matching only** and are **not** evidence for Q2's classifier‑driven path.


---

## Q3 — No‑extractable‑text edge case

### Q3(a) — Which OCR subprocess is invoked?

**Answer:** The OCR call is **`ocrmypdf.ocr(**args)`**, invoked inside `RasterisedDocumentParser.parse()` (def `src/paperless_tesseract/parsers.py:230`):

- Primary call: `ocrmypdf.ocr(**args)` — `src/paperless_tesseract/parsers.py:261` (preceded by the log line `Calling OCRmyPDF with args: {args}` at `parsers.py:260`).

When the primary OCR yields no text, a **force‑OCR fallback** runs:

- `if not self.text:` — `src/paperless_tesseract/parsers.py:266`
- `raise NoTextFoundException("No text was found in the original document")` — `src/paperless_tesseract/parsers.py:267` (class defined at `parsers.py:14`)
- caught by `except (NoTextFoundException, InputFileError) as e:` — `src/paperless_tesseract/parsers.py:276`
- the fallback rebuilds args with `safe_fallback=True`, which sets `ocrmypdf_args["force_ocr"] = True` (`parsers.py:155-156`), and retries `ocrmypdf.ocr(**args)` — `src/paperless_tesseract/parsers.py:298`.

**Run markers — the two no‑text parser tests pass:**

```
$ python3 -m pytest paperless_tesseract/tests/test_parser.py -k "notext" -o addopts="" -v
paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED [ 50%]
paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED [100%]
====================== 2 passed, 33 deselected in 10.02s =======================
```

These are `test_with_form_error_notext` (`test_parser.py:190`, `@override_settings(OCR_MODE="redo")`, sample `with-form.pdf`) and `test_skip_noarchive_notext` (`test_parser.py:370`, `OCR_MODE="skip_noarchive"`, sample `multi-page-images.pdf`).

**Observed — the actual invocation and force‑OCR retry** (temporary probe using `unittest.mock.patch("ocrmypdf.ocr")` to record the args, with `extract_text` patched to return `""` so the `NoTextFoundException` branch fires; input sample `no-text-alpha.png`):

```
Q3PROBE input_sample=no-text-alpha.png
Q3PROBE ocrmypdf_ocr_call_count=2
Q3PROBE primary_call_force_ocr=None
Q3PROBE primary_call_keys=['clean', 'deskew', 'image_dpi', 'input_file', 'jobs', 'language', 'output_file', 'output_type', 'progress_bar', 'rotate_pages', 'rotate_pages_threshold', 'sidecar', 'skip_text', 'use_threads']
Q3PROBE fallback_call_force_ocr=True
Q3PROBE fallback_input_file=no-text-alpha.png
Q3PROBE warning_log=Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
Q3PROBE NoTextFoundException_str='No text was found in the original document'
```

- `ocrmypdf_ocr_call_count=2` — `ocrmypdf.ocr` is called **twice**: the primary call (`parsers.py:261`) and the force‑OCR fallback (`parsers.py:298`).
- `primary_call_force_ocr=None` with `skip_text` present in the primary args — under the test's default `OCR_MODE` (`skip`/`skip_noarchive`), the primary call uses `skip_text=True`, not `force_ocr`.
- `fallback_call_force_ocr=True` — the fallback sets **`force_ocr=True`** (`parsers.py:155-156`), i.e. it forces OCR to run.
- The warning log embeds the exact exception text, and the exception message string is exactly **`No text was found in the original document`** (`parsers.py:267`).

**Sample files present** in `src/paperless_tesseract/tests/samples/` (observed by `ls`): `encrypted.pdf`, `multi-page-digital.pdf`, `multi-page-images.pdf`, `multi-page-mixed.pdf`, `no-text-alpha.png`, `rotated.pdf`, `signed.pdf`, `simple-alpha.png`, `simple-digital.pdf`, `simple-no-dpi.png`, `simple.bmp`, `simple.gif`, `simple.jpg`, `simple.png`, `simple.tif`, `with-form.pdf`. There is **no bare `simple.pdf`** in this directory:

```
$ ls paperless_tesseract/tests/samples/simple.pdf
ls: cannot access 'paperless_tesseract/tests/samples/simple.pdf': No such file or directory
```

The sample fed to the probe was **`no-text-alpha.png`** (reported exactly as used).

### Q3(b) — What MIME type is assigned to the output?

**Answer:** The MIME type is detected via **`magic.from_file(self.path, mime=True)`** (`src/documents/consumer.py:219`) and persisted **as‑is** — e.g. **`application/pdf`** for a PDF.

- Detection: `mime_type = magic.from_file(self.path, mime=True)` — `src/documents/consumer.py:219`
- Parser dispatch keys off it: `get_parser_class_for_mime_type(mime_type)` — `src/documents/consumer.py:223`
- Persistence: value flows to `self._store(text=text, date=date, mime_type=mime_type)` (`consumer.py:301`; `_store` def `consumer.py:379`) and is written to the `Document` as `mime_type=mime_type` (`consumer.py:401`).

**Observed — the detected MIME string for PDF samples** (temporary probe):

```
Q3PROBE magic.from_file(simple-digital.pdf)='application/pdf'
Q3PROBE magic.from_file(with-form.pdf)='application/pdf'
Q3PROBE magic.from_file(multi-page-images.pdf)='application/pdf'
```

For each PDF sample, `magic.from_file(..., mime=True)` returns exactly **`application/pdf`**, which is the value stored on the resulting `Document` (`consumer.py:401`). The "no‑extractable‑text" condition does **not** change the assigned MIME type — it changes only the OCR path (Q3(a)); the MIME type reflects the *container* format detected on the input file.


---

## Q4 — Barcode splitting

### Q4(a) — How many document records result from a single input?

**Answer:** `(number_of_separators) + 1` output PDFs, with each separator (barcode) page **removed**. `separate_pages()` (def `src/documents/tasks.py:113`) writes `{fname}_document_0.pdf` for the pages before the first separator (`tasks.py:134`), then loops writing `{fname}_document_{count+1}.pdf` (`tasks.py:154`), skipping each barcode page via `for page in range(page_number + 1, next_page):` (`tasks.py:149`), and returns `document_paths` (`tasks.py:161`).

**Run marker** — `test_separate_pages` (`test_tasks.py:305`) calls `separate_pages(patch-code-t-middle.pdf, [1])` and asserts `len(pages) == 2` (`test_tasks.py:313`):

```
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 85%]
documents/tests/test_tasks.py::TestTasks::test_separate_pages_no_list PASSED [100%]
```

**Observed count and filenames** (temporary probe calling `separate_pages` directly on `patch-code-t-middle.pdf` with its one separator at page index `1`):

```
Q4PROBE separators_for_middle=[1] (num_separators=1)
Q4PROBE separate_pages_output_count=2
Q4PROBE separate_pages_output_names=['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
Q4PROBE separate_pages_empty_list=[]
```

One separator (`[1]`) → **2** output documents (`_document_0.pdf`, `_document_1.pdf`), with the separator page removed. An **empty** split list returns `[]` and logs a warning — `test_separate_pages_no_list` (`test_tasks.py:315`) asserts `pages == []` and `cm.output == ["WARNING:paperless.tasks:No pages to split on!"]` (`test_tasks.py:328`).

### Q4(b) — Which barcode values trigger a split?

**Answer:** The default trigger string is **`"PATCHT"`**: `CONSUMER_BARCODE_STRING = os.getenv("PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")` — `src/paperless/settings.py:506`. The whole feature is gated by `CONSUMER_ENABLE_BARCODES` — `src/paperless/settings.py:502` (default `False`).

**Observed — default settings:**

```
Q4PROBE default_CONSUMER_BARCODE_STRING='PATCHT'
Q4PROBE default_CONSUMER_ENABLE_BARCODES=False
```

**Observed — `scan_file_for_separating_barcodes(...)` return list per named sample** (default `PATCHT`):

```
Q4PROBE scan[barcodes/patch-code-t.pdf]=[0]
Q4PROBE scan[simple.pdf]=[]
Q4PROBE scan[barcodes/patch-code-t-middle.pdf]=[1]
Q4PROBE scan[barcodes/several-patcht-codes.pdf]=[2, 5]
Q4PROBE scan[barcodes/patch-code-t-middle_reverse.pdf]=[1]
Q4PROBE scan[barcodes/patch-code-t-qr.pdf]=[0]
```

- `patch-code-t.pdf` → `[0]` (asserted `test_tasks.py:207-215`)
- `simple.pdf` → `[]` (no PATCHT barcode; `test_tasks.py:217-220`)
- `patch-code-t-middle.pdf` → `[1]` (`test_tasks.py:222-230`)
- `several-patcht-codes.pdf` → `[2, 5]` (two separators; `test_tasks.py:232-240`)
- `patch-code-t-middle_reverse.pdf` → `[1]` (upside‑down; `test_tasks.py:242-250`)
- `patch-code-t-qr.pdf` → `[0]` (QR‑encoded PATCHT; `test_tasks.py:252-260`)

**Observed — a custom trigger string only splits when configured.** With the default `PATCHT`, a custom‑barcode sample does **not** trigger; overriding `CONSUMER_BARCODE_STRING="CUSTOM BARCODE"` makes the custom samples trigger at page `0`:

```
Q4PROBE scan[barcode-39-custom.pdf, default PATCHT]=[]
Q4PROBE override_CONSUMER_BARCODE_STRING='CUSTOM BARCODE'
Q4PROBE scan[barcode-39-custom.pdf, CUSTOM BARCODE]=[0]
Q4PROBE scan[barcode-qr-custom.pdf, CUSTOM BARCODE]=[0]
Q4PROBE scan[barcode-128-custom.pdf, CUSTOM BARCODE]=[0]
```

- `barcode-39-custom.pdf` → `[]` under default `PATCHT`, but `[0]` under `CUSTOM BARCODE` (`test_tasks.py:263`, and the without‑override case `test_tasks.py:295-303`)
- `barcode-qr-custom.pdf` → `[0]` under `CUSTOM BARCODE` (`test_tasks.py:274`)
- `barcode-128-custom.pdf` → `[0]` under `CUSTOM BARCODE` (`test_tasks.py:285`)

The 18 barcode sample files live in `src/documents/tests/samples/barcodes/` (including `barcode-39-PATCHT.png`, `patch-code-t.pdf`, `patch-code-t-middle.pdf`, `several-patcht-codes.pdf`, `barcode-128-custom.pdf`).

### Q4(c) — Where in the code is the decision made?

**Answer:** The split decision executes at **`if separator_barcode in current_barcodes:`** — `src/documents/tasks.py:108`, inside `scan_file_for_separating_barcodes` (def `src/documents/tasks.py:96`), where:

- `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` — `src/documents/tasks.py:102`
- `current_barcodes = barcode_reader(page)` — `src/documents/tasks.py:107`
- `barcode_reader` (def `src/documents/tasks.py:75`) decodes via `pyzbar.decode(image)` — `src/documents/tasks.py:82`

Orchestration is `consume_file` (def `src/documents/tasks.py:184`): the gate `if settings.CONSUMER_ENABLE_BARCODES:` (`tasks.py:195`) → `scan_file_for_separating_barcodes(path)` (`tasks.py:198`) → `separate_pages(path, separators)` (`tasks.py:201`) → each split saved via `save_to_dir(...)` (`tasks.py:210`).

### Q4(d) — How does it change the effective training data during the run?

**Answer:** Each split file is **re‑consumed as its own `Document`** (via `save_to_dir(...)` at `tasks.py:210`, which drops each split PDF back into consumption). So a single input becomes `N+1` `Document` rows, which changes the training‑set count read by `classifier.py:125` (`Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)`), which changes the SHA‑1 `data_hash` (`classifier.py:161-164`), and therefore forces a **retrain** on the next `train()`.

This links **Q4 → Q1/Q2 directly**, and the link is empirically demonstrated by the Q1 probe: adding a single document (3 → 4 rows) changed the observed `data_hash` from `b41ce397…c74447` to `2f0b4d6e…3e79` and flipped the next `train()` from a reuse (`False`) to a retrain (`True`). Thus barcode splitting's `N+1` documents feed both the reuse‑vs‑retrain decision (Q1) and the correspondent training labels (Q2).


---

## Non‑determinism root cause (two independent vectors, observed at realistic magnitude)

> This is a **diagnosis‑only** section. No source was modified — no `random_state` was added, no xdist configuration was changed. Two candidate vectors were investigated, and **exactly what was observed is reported below**, including the fact that neither vector reproduced an *actual* test failure at the magnitudes run.

### Vector 1 — `MLPClassifier(tol=0.01)` constructed WITHOUT a `random_state`

The three classifiers are all constructed without a `random_state`:

- `self.tags_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:219`
- `self.correspondent_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:227`
- `self.document_type_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:238`

With `random_state=None`, scikit‑learn's `MLPClassifier` uses non‑deterministic weight/bias initialization (and batch sampling for the `adam`/`sgd` solvers), so a retrained model's internal weights differ from run to run.

**Observed — the randomness is REAL: `random_state` is `None` and the trained weights differ run‑to‑run.** A probe trained two fresh `DocumentClassifier` instances on identical fixture data and compared the correspondent network's first‑layer weight matrix:

```
ND3PROBE correspondent_random_state=None
ND3PROBE tags_random_state=None
ND3PROBE document_type_random_state=None
ND3PROBE correspondent_weights_identical_run_to_run=False
ND3PROBE correspondent_weights_max_abs_diff=0.433783
ND3PROBE predict_doc1_modelA=1 modelB=1
```

- `random_state=None` on all three fitted estimators — confirming no seed is set.
- `correspondent_weights_identical_run_to_run=False` with `max_abs_diff=0.433783` — two independent trainings on identical data produce **materially different weight matrices**. This is the genuine non‑determinism vector.
- `predict_doc1_modelA=1 modelB=1` — despite different weights, both models still predict `c1.pk=1`.

**Observed — at magnitude, the varying weights did NOT change predictions on the current fixtures.** A probe built the canonical fixture once and trained a fresh classifier `N=100` times, tallying deviations from the expected labels (`c1.pk` / `dt.pk` / `[t1.pk]`):

```
ND1PROBE N=100
ND1PROBE expected correspondent=c1.pk=1 document_type=dt.pk=1 tags=[t1.pk]=[12]
ND1PROBE predict_correspondent DEVIATIONS=0/100
ND1PROBE predict_correspondent value_distribution={'1': 100}
ND1PROBE predict_document_type DEVIATIONS=0/100
ND1PROBE predict_tags DEVIATIONS=0/100
```

Zero deviations in 100 fresh trainings. Repeating on the *harder* fixture from `test_one_correspondent_predict_manydocs` (`test_classifier.py:206`), where the two documents differ by a single word — `"this is a document from c1"` vs. `"this is a document from noone"` — at `N=200`:

```
ND2PROBE N=200 c1.pk=1
ND2PROBE doc1_expected=c1.pk=1 DEVIATIONS=0/200 distribution={'1': 200}
ND2PROBE doc2_expected=None DEVIATIONS=0/200 distribution={'None': 200}
```

Still zero deviations in 200 fresh trainings.

**Observed — the real exact‑equality tests do not flake at 50 repetitions each (serial):**

```
TESTPREDICT_LOOP pass=50 fail=0 out_of=50
LOOP test_one_correspondent_predict: pass=50 fail=0 out_of=50
LOOP test_one_correspondent_predict_manydocs: pass=50 fail=0 out_of=50
```

**Control — the loaded‑model path is deterministic.** `test_load_and_classify` (`test_classifier.py:183`) *loads* the pre‑trained `model.pickle` (`new_classifier.load()`) instead of retraining, so its weights are fixed:

```
LOAD_AND_CLASSIFY_LOOP passed=20 skipped=0 failed=0 out_of=20
```

20/20 deterministic — confirming that any flakiness would originate from **retraining** (random weight init), not from the prediction step or the loaded‑model path.

### Vector 2 — pytest‑xdist `--numprocesses auto`

The suite default enables xdist parallelism: `addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet` — `src/setup.cfg:10`. Parallel workers can surface isolation issues invisible in serial runs.

**Observed — no failure‑rate difference between serial and parallel** for the full `documents/tests/test_classifier.py` (23 tests collected; 22 run, 1 skipped), 10 runs each:

```
SERIAL_RESULT pass=10 fail=0 out_of=10          # -o addopts=""  (single process)
PARALLEL_RESULT pass=10 fail=0 out_of=10        # -n auto  (pytest-xdist 3.8.0)
# sample parallel summary: "22 passed, 1 skipped in 25.41s"
```

The one skipped test is `test_load_classifier_cached` (`test_classifier.py:391`), skipped with reason *"Disabled caching due to high memory usage - need to investigate."* — unrelated to flakiness.

**Isolation / ordering facts observed:**

- No `conftest.py` exists under `src/` (a `find` for it returned nothing), **pytest‑randomly is not installed** (`pip list` shows only `pytest-cov` and `pytest-xdist 3.8.0` among the relevant plugins), and there is **no `.python-version`** file. Test ordering is therefore stable except for xdist worker distribution.
- `DirectoriesMixin` (`utils.py:72`) overrides `DATA_DIR`/`SCRATCH_DIR`/`MEDIA_ROOT`/`MODEL_FILE` per test (`utils.py:35-47`, `MODEL_FILE` at `utils.py:45`) and Django `TestCase` rolls back the DB, so cross‑test model‑file bleed is unlikely. This isolation does **not** remove the within‑test weight‑init randomness of Vector 1.

### Conclusion (reported exactly as observed)

- The **code‑level non‑determinism vector is Vector 1**: the `MLPClassifier` instances are trained with `random_state=None` (`classifier.py:219,227,238`), and this randomness is empirically real — two trainings on identical data produced different weight matrices (`max_abs_diff=0.433783`). This is the mechanism that *can* make the exact‑equality classifier assertions (`testPredict` at `test_classifier.py:115`, `test_one_correspondent_predict` at `:191`, `test_one_correspondent_predict_manydocs` at `:206`) flaky when the training set is larger or less separable.
- **However, at the magnitudes run** (200 fresh trainings on two fixtures with 0 deviations; 50 repetitions each of three real exact‑equality tests with 0 failures; 10 serial and 10 parallel full‑suite runs with 0 failures), **neither vector reproduced an actual failure**. The current fixtures are tiny and linearly separable, so the decision boundary is robust even though the underlying weights vary.
- **Vector 2 (xdist)** showed no difference in failure frequency between serial and parallel execution and is mitigated by `DirectoriesMixin` isolation + DB rollback.
- Practical implication for whoever fixes the flakiness (out of scope here, stated for the reader): the latent risk lives in the unseeded `MLPClassifier`; the observed stability is a property of these specific small fixtures, not a guarantee. Because the mechanism is confirmed present but did not fire at this scale, the exact real‑world trigger (e.g. a larger/edge‑case training set, or a specific worker interleaving) **could not be reproduced by reading or running the code at the scale attempted**, and is reported as such rather than asserted.


---

## Coverage pass — every named item answered

Each item below was addressed above with its own verbatim evidence and an exact `file:line` citation.

- [x] **Q1 — reuse vs. retrain mechanism.** SHA‑1 `data_hash` (`classifier.py:124`, digest `:161`); reuse short‑circuit `if self.data_hash and new_data_hash == self.data_hash:` `:163` → `return False` `:164`; retrain sets `self.data_hash = new_data_hash` `:247` → `return True` `:249`.
- [x] **Q1 — `train()` returns `True` then `False`.** `testDatasetHashing` `PASSED` (`test_classifier.py:141-142`); probe hashes `b41ce397…` (True) → same (False) → `2f0b4d6e…` (True after mutate).
- [x] **Q1 — `FORMAT_VERSION 7`.** Observed `Q1PROBE FORMAT_VERSION=7` (`classifier.py:63`).
- [x] **Q1 — `load_classifier()` returns `None` when the model file is absent.** Observed `MODEL_FILE_exists=False` → `load_classifier_returns=None` (`classifier.py:30-36`).
- [x] **Q1 — DirectoriesMixin per‑test `MODEL_FILE` isolation.** `utils.py:45,72`; DB rollback per `TestCase`.
- [x] **Q1 — downstream effect.** Unchanged eligible set → reuse (`False`); any change → retrain (`True`).
- [x] **Q2(a) — 3 created / 2 training‑eligible.** Observed `total_documents_created=3`, `training_eligible=2`; inbox exclusion `classifier.py:125`, `is_inbox_tag=True` `test_classifier.py:44`.
- [x] **Q2(b) — training after inserts.** `testTrain`/`testPredict` call `generate_test_data()` then `train()` (`test_classifier.py:103-104,115-116`).
- [x] **Q2(c) — acceptance rule `class != -1`, NO numeric threshold.** `if correspondent_id != -1:` `classifier.py:255`; no `predict_proba`/`threshold` in source; `correspondent_classes_=[-1, 1]` = `[-1, c1.pk]`; `predict_correspondent(doc1)=array([1])`, `(doc2)=None`.
- [x] **Q2 chain.** `match_correspondents` `matching.py:21` (predict call `:23`, filter `:30`); `set_correspondent` `handlers.py:35`, call `:50`; consumer `load_classifier()` `consumer.py:292`, signal `:306-311`.
- [x] **Q2 disambiguation.** `test_matchables.py` uses regex `matches()` (`matching.py:60`) via `_test_matching` (`test_matchables.py:20`); grep for classifier symbols returns empty — not the classifier path.
- [x] **Q3(a) — `ocrmypdf.ocr(**args)`.** Primary `parsers.py:261`; `NoTextFoundException("No text was found in the original document")` `:267` (class `:14`); catch `:276`; force‑OCR retry `:298` with `force_ocr=True` (`:155-156`). Observed `ocrmypdf_ocr_call_count=2`, `fallback_call_force_ocr=True`, exact message string. Sample used: `no-text-alpha.png` (no bare `simple.pdf` present).
- [x] **Q3(b) — MIME `application/pdf`.** `magic.from_file(self.path, mime=True)` `consumer.py:219`; persisted `mime_type=mime_type` `consumer.py:401`. Observed `application/pdf` for `simple-digital.pdf`, `with-form.pdf`, `multi-page-images.pdf`.
- [x] **Q4(a) — `N+1` outputs, separators removed.** `separate_pages` `tasks.py:113-161`; observed `separate_pages_output_count=2` for one separator; names `_document_0.pdf`/`_document_1.pdf`; empty list → `[]` with `No pages to split on!`.
- [x] **Q4(b) — default `"PATCHT"` trigger + per‑sample lists.** `settings.py:506`; observed `patch-code-t.pdf→[0]`, `simple.pdf→[]`, `patch-code-t-middle.pdf→[1]`, `several-patcht-codes.pdf→[2, 5]`, `patch-code-t-middle_reverse.pdf→[1]`, `patch-code-t-qr.pdf→[0]`; custom cases `barcode-39-custom.pdf→[]`(PATCHT)/`[0]`(CUSTOM BARCODE), `barcode-qr-custom.pdf→[0]`, `barcode-128-custom.pdf→[0]`.
- [x] **Q4(c) — decision site.** `if separator_barcode in current_barcodes:` `tasks.py:108`; `scan_file_for_separating_barcodes` `:96`; `barcode_reader` `:75` (`pyzbar.decode` `:82`); `consume_file` `:184` (gate `:195`, scan `:198`, split `:201`, `save_to_dir` `:210`).
- [x] **Q4(d) — training‑data impact.** Re‑consumption → `N+1` `Document` rows → training‑set count (`classifier.py:125`) → SHA‑1 `data_hash` (`:161-164`) → retrain; empirically linked via Q1 probe's `3→4` hash change.
- [x] **Non‑determinism — both vectors named + magnitude + which reproduced.** Vector 1 `MLPClassifier(tol=0.01)` no `random_state` (`classifier.py:219/227/238`) — confirmed real (`random_state=None`, weights differ, `max_abs_diff=0.433783`) but 0 deviations at N=100/N=200 and 0 failures over 50× each real test; Vector 2 xdist (`setup.cfg:10`) — 10/10 serial and 10/10 parallel, no difference. Diagnose‑only (nothing modified).
- [x] **Read‑only mandate + cleanup.** Temporary `blitzy_adhoc_test_*` scripts removed; only this document added (verified via `git status`).

### Notes on values reported exactly as observed (not adjusted)

- The container is a **detached‑HEAD** checkout at `542221a38`; `git branch --show-current` is empty (the deliverable name uses the task‑supplied source branch `paperless-ngx_542221a38dff`).
- `predict_correspondent` returns a NumPy `array([1])` (not a scalar `1`).
- The non‑determinism vectors are **real in the code** yet **did not reproduce a failure** at the magnitudes run; this is reported as observed rather than forced toward an expected "flaky" outcome. The precise real‑world trigger could not be reproduced at the attempted scale and is flagged as such.
- Barcode logic resides in `src/documents/tasks.py` at this commit (not a separate `barcodes.py` module).

