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

**Claim — the scikit‑learn pin is the exact equality‑form literal `scikit-learn==1.0.2` at `requirements.txt:88`.**

```bash
docker exec -u testuser -w /app/src paperless bash -c "sed -n '88p' /app/requirements.txt"
```

```text
scikit-learn==1.0.2
```

The pin is a hard equality (`==`), not a range — required because the `FORMAT_VERSION 7` model pickle (`src/documents/classifier.py:63`) is not cross‑version compatible. This is the exact literal the questions concern, cited to `requirements.txt:88`.

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
============================== 1 passed in 1.92s ===============================
```

The `PASSED` marker confirms `train()` → `True` then `train()` → `False` on identical DB state.

### Q1 evidence — observed SHA‑1 hashes and boolean returns (train‑twice, then mutate)

**Producing code** — temporary probe `documents/tests/blitzy_adhoc_test_probe.py`, a subclass of the real `TestClassifier` that reuses its `generate_test_data()` fixture and `DirectoriesMixin` isolation (fresh per‑test `MODEL_FILE` + DB rollback). Invocation:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_q1_probe \
    -o addopts="" -p no:cacheprovider -s -q
```

```python
def test_q1_probe(self):
    print("Q1PROBE FORMAT_VERSION=%d" % DocumentClassifier.FORMAT_VERSION)
    print("Q1PROBE MODEL_FILE_exists=%s" % os.path.isfile(settings.MODEL_FILE))
    print("Q1PROBE load_classifier_returns=%s" % load_classifier())
    self.generate_test_data()
    print("Q1PROBE total_documents=%d" % Document.objects.count())
    print("Q1PROBE eligible_documents=%d"
          % Document.objects.exclude(tags__is_inbox_tag=True).count())
    r1 = self.classifier.train(); h1 = self.classifier.data_hash.hex()
    print("Q1PROBE TRAIN#1 return=%s data_hash=%s" % (r1, h1))
    r2 = self.classifier.train(); h2 = self.classifier.data_hash.hex()
    print("Q1PROBE TRAIN#2 return=%s data_hash=%s" % (r2, h2))
    print("Q1PROBE IDENTICAL_HASH_1_vs_2=%s" % (h1 == h2))
    Document.objects.create(title="doc_extra",
        content="this is an extra document from c1",
        correspondent=self.c1, checksum="Z")
    print("Q1PROBE total_documents_after_mutate=%d" % Document.objects.count())
    r3 = self.classifier.train(); h3 = self.classifier.data_hash.hex()
    print("Q1PROBE TRAIN#3(after mutate) return=%s data_hash=%s" % (r3, h3))
    print("Q1PROBE HASH_CHANGED_2_vs_3=%s" % (h2 != h3))
```

Each claim below pairs the single producing `print(...)` line with its own verbatim observed output line.

**Claim — `FORMAT_VERSION` is `7`** (`classifier.py:63`).
```python
print("Q1PROBE FORMAT_VERSION=%d" % DocumentClassifier.FORMAT_VERSION)
```
```
Q1PROBE FORMAT_VERSION=7
```

**Claim — `load_classifier()` returns `None` when the model file is absent** (`classifier.py:30-36`; the fresh per‑test `MODEL_FILE` from `DirectoriesMixin` has no serialized model).
```python
print("Q1PROBE MODEL_FILE_exists=%s" % os.path.isfile(settings.MODEL_FILE))
print("Q1PROBE load_classifier_returns=%s" % load_classifier())
```
```
Q1PROBE MODEL_FILE_exists=False
Q1PROBE load_classifier_returns=None
```

**Claim — the fixture creates 3 documents but only 2 are training‑eligible** (inbox exclusion at `classifier.py:125`).
```python
print("Q1PROBE total_documents=%d" % Document.objects.count())
print("Q1PROBE eligible_documents=%d"
      % Document.objects.exclude(tags__is_inbox_tag=True).count())
```
```
Q1PROBE total_documents=3
Q1PROBE eligible_documents=2
```

**Claim — the first `train()` returns `True` and computes a 40‑hex‑char (SHA‑1, 160‑bit) `data_hash`** (`hashlib.sha1()` `classifier.py:124`, `m.digest()` `classifier.py:161`).
```python
r1 = self.classifier.train(); h1 = self.classifier.data_hash.hex()
print("Q1PROBE TRAIN#1 return=%s data_hash=%s" % (r1, h1))
```
```
Q1PROBE TRAIN#1 return=True data_hash=b41ce39793cd61f391afe34e6e4de9b361c74447
```

**Claim — the second `train()` on identical DB state returns `False` with the identical hash (reuse short‑circuit)** (`if self.data_hash and new_data_hash == self.data_hash:` → `return False` — `classifier.py:163-164`).
```python
r2 = self.classifier.train(); h2 = self.classifier.data_hash.hex()
print("Q1PROBE TRAIN#2 return=%s data_hash=%s" % (r2, h2))
print("Q1PROBE IDENTICAL_HASH_1_vs_2=%s" % (h1 == h2))
```
```
Q1PROBE TRAIN#2 return=False data_hash=b41ce39793cd61f391afe34e6e4de9b361c74447
Q1PROBE IDENTICAL_HASH_1_vs_2=True
```

**Claim — mutating the eligible set (adding one non‑inbox `Document`, 3 → 4 rows) changes the hash and forces a retrain, so the next `train()` returns `True`** (`self.data_hash = new_data_hash` `classifier.py:247`; `return True` `classifier.py:249`).
```python
Document.objects.create(title="doc_extra",
    content="this is an extra document from c1",
    correspondent=self.c1, checksum="Z")
print("Q1PROBE total_documents_after_mutate=%d" % Document.objects.count())
r3 = self.classifier.train(); h3 = self.classifier.data_hash.hex()
print("Q1PROBE TRAIN#3(after mutate) return=%s data_hash=%s" % (r3, h3))
print("Q1PROBE HASH_CHANGED_2_vs_3=%s" % (h2 != h3))
```
```
Q1PROBE total_documents_after_mutate=4
Q1PROBE TRAIN#3(after mutate) return=True data_hash=1c7c836d999c3931b1b85c6bc673431a5733f7b0
Q1PROBE HASH_CHANGED_2_vs_3=True
```

> Reported exactly as observed: the `TRAIN#3` digest `1c7c836d…f7b0` differs from the `TRAIN#1`/`TRAIN#2` digest `b41ce397…c74447` because the added document changed the preprocessed‑content/label byte‑stream fed into `hashlib.sha1()`. The exact `TRAIN#3` value depends on the specific mutating document (here content `"this is an extra document from c1"`, correspondent `c1`); the invariant demonstrated is `HASH_CHANGED_2_vs_3=True` → retrain (`return True`).

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

**Producing code** — temporary probe `documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_q2_probe` (same `TestClassifier` subclass), run with:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_q2_probe \
    -o addopts="" -p no:cacheprovider -s -q
```

The full method is shown once here (it produces every Q2 marker below); each subsequent claim repeats only its own producing `print(...)` line:

```python
def test_q2_probe(self):
    self.generate_test_data()
    print("Q2PROBE total_documents_created=%d" % Document.objects.count())
    print("Q2PROBE training_eligible=%d"
          % Document.objects.exclude(tags__is_inbox_tag=True).count())
    print("Q2PROBE about_to_train (inserts already done above)")
    self.classifier.train()
    print("Q2PROBE trained")
    print("Q2PROBE c1_pk=%d c2_pk=%d" % (self.c1.pk, self.c2.pk))
    print("Q2PROBE correspondent_classes_=%s"
          % list(self.classifier.correspondent_classifier.classes_))
    print("Q2PROBE tags_binarizer_classes_=%s"
          % list(self.classifier.tags_binarizer.classes_))
    print("Q2PROBE predict_correspondent(doc1)=%r (expect c1.pk=%d)"
          % (self.classifier.predict_correspondent(self.doc1.content), self.c1.pk))
    print("Q2PROBE predict_correspondent(doc2)=%r (expect None)"
          % (self.classifier.predict_correspondent(self.doc2.content),))
    vec = self.classifier.data_vectorizer
    raw1 = self.classifier.correspondent_classifier.predict(
        vec.transform(["this is a document from c1"]))
    raw2 = self.classifier.correspondent_classifier.predict(
        vec.transform(["this is another document, but from c2"]))
    print("Q2PROBE raw_predict(doc1)=%s raw_predict(doc2)=%s" % (list(raw1), list(raw2)))
    src = inspect.getsource(DocumentClassifier.predict_correspondent)
    print("Q2PROBE predict_correspondent_has_predict_proba=%s" % ("predict_proba" in src))
    print("Q2PROBE predict_correspondent_has_threshold_literal=%s" % ("threshold" in src))
    for line in src.rstrip().splitlines():
        print("Q2PROBE_SRC %s" % line)
```

**Claim — the fixture creates 3 `Document` rows.**
```python
print("Q2PROBE total_documents_created=%d" % Document.objects.count())
```
```
Q2PROBE total_documents_created=3
```

**Claim — only 2 of them are training‑eligible** (`Document.objects.exclude(tags__is_inbox_tag=True)` mirrors the inbox exclusion at `classifier.py:125`).
```python
print("Q2PROBE training_eligible=%d"
      % Document.objects.exclude(tags__is_inbox_tag=True).count())
```
```
Q2PROBE training_eligible=2
```

### Q2(b) — When does training occur relative to those inserts?

**Answer:** Training runs **after** the document inserts. Both `testTrain` (`test_classifier.py:103`) and `testPredict` (`test_classifier.py:115`) call `self.generate_test_data()` (which performs the inserts) and *then* `self.classifier.train()`.

**Observed ordering** — the probe prints a marker immediately before and after `train()`, with the fixture inserts already complete (`generate_test_data()` runs first in the method above).
```python
print("Q2PROBE about_to_train (inserts already done above)")
self.classifier.train()
print("Q2PROBE trained")
```
```
Q2PROBE about_to_train (inserts already done above)
Q2PROBE trained
```

The `about_to_train` marker prints only after `generate_test_data()` has inserted the rows, so `train()` is invoked strictly after the inserts. (In real consumption, classifier training is a separate task that runs after documents already exist in the database.)

### Q2(c) — What confidence threshold accepts or rejects a prediction?

**Answer:** **There is no numeric probability threshold.** Acceptance is decided solely by the **`class != -1` sentinel rule** inside `predict_correspondent` (def `src/documents/classifier.py:251`):

- `if correspondent_id != -1:` — `src/documents/classifier.py:255`
- `return correspondent_id` — `src/documents/classifier.py:256`
- `return None` — `src/documents/classifier.py:258`

The `-1` label is the "no automatic correspondent" sentinel.

**Observed — the classifier's classes and the accept/reject outcomes** (one claim per block):

**Claim — `c1.pk=1` and `c2.pk=2`.**
```python
print("Q2PROBE c1_pk=%d c2_pk=%d" % (self.c1.pk, self.c2.pk))
```
```
Q2PROBE c1_pk=1 c2_pk=2
```

**Claim — `correspondent_classifier.classes_` is `[-1, c1.pk]` = `[-1, 1]`** (asserted at `test_classifier.py:106-109`). Among the two training docs, `doc1.correspondent=c1` is `MATCH_AUTO` → label `c1.pk=1`, while `doc2.correspondent=c2` is not auto → label `-1` (the default `y = -1` in `train()`).
```python
print("Q2PROBE correspondent_classes_=%s"
      % list(self.classifier.correspondent_classifier.classes_))
```
```
Q2PROBE correspondent_classes_=[-1, 1]
```

**Claim — `tags_binarizer.classes_` is `[t1.pk, t3.pk]` = `[12, 45]`** (asserted at `test_classifier.py:110-113`).
```python
print("Q2PROBE tags_binarizer_classes_=%s"
      % list(self.classifier.tags_binarizer.classes_))
```
```
Q2PROBE tags_binarizer_classes_=[12, 45]
```

**Claim — `predict_correspondent(doc1)` is accepted** — predicted class `1` (= `c1.pk`) is `!= -1`, so it is returned. Reported exactly as observed: the method returns a NumPy `array([1])`, not a scalar `1`; `assertEqual(..., c1.pk)` still passes because `array([1]) == 1`.
```python
print("Q2PROBE predict_correspondent(doc1)=%r (expect c1.pk=%d)"
      % (self.classifier.predict_correspondent(self.doc1.content), self.c1.pk))
```
```
Q2PROBE predict_correspondent(doc1)=array([1]) (expect c1.pk=1)
```

**Claim — `predict_correspondent(doc2)` is rejected → `None`** — the raw predicted class for `doc2` is the `-1` sentinel, so the `if correspondent_id != -1:` gate is false.
```python
print("Q2PROBE predict_correspondent(doc2)=%r (expect None)"
      % (self.classifier.predict_correspondent(self.doc2.content),))
```
```
Q2PROBE predict_correspondent(doc2)=None (expect None)
```

**Claim — the raw categorical `predict()` output (before the `!= -1` gate) is `[1]` for doc1 and `[-1]` for doc2.**
```python
raw1 = self.classifier.correspondent_classifier.predict(vec.transform(["this is a document from c1"]))
raw2 = self.classifier.correspondent_classifier.predict(vec.transform(["this is another document, but from c2"]))
print("Q2PROBE raw_predict(doc1)=%s raw_predict(doc2)=%s" % (list(raw1), list(raw2)))
```
```
Q2PROBE raw_predict(doc1)=[1] raw_predict(doc2)=[-1]
```

**Claim — there is NO numeric probability threshold: the method contains neither `predict_proba` nor any `threshold` literal.** The probe checks the method source and dumps it verbatim:
```python
src = inspect.getsource(DocumentClassifier.predict_correspondent)
print("Q2PROBE predict_correspondent_has_predict_proba=%s" % ("predict_proba" in src))
print("Q2PROBE predict_correspondent_has_threshold_literal=%s" % ("threshold" in src))
for line in src.rstrip().splitlines():
    print("Q2PROBE_SRC %s" % line)
```
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
Both flags are `False` and the dumped body shows acceptance is the categorical `predict()` output gated only by `if correspondent_id != -1:` (`classifier.py:255`).

**Run markers** — produced by:
```
$ python3 -m pytest documents/tests/test_classifier.py::TestClassifier::testTrain \
    documents/tests/test_classifier.py::TestClassifier::testPredict \
    -o addopts="" -p no:cacheprovider -v -p no:warnings
```
```
documents/tests/test_classifier.py::TestClassifier::testTrain PASSED     [ 50%]
documents/tests/test_classifier.py::TestClassifier::testPredict PASSED   [100%]
============================== 2 passed in 1.99s ===============================
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
================= 2 passed, 33 deselected, 6 warnings in 9.90s =================
```

These are `test_with_form_error_notext` (`test_parser.py:190`, `@override_settings(OCR_MODE="redo")`, sample `with-form.pdf`) and `test_skip_noarchive_notext` (`test_parser.py:370`, `OCR_MODE="skip_noarchive"`, sample `multi-page-images.pdf`).

**Observed — the actual invocation and force‑OCR retry.** A temporary probe mocks `ocrmypdf.ocr` to record its call arguments and patches `RasterisedDocumentParser.extract_text` to return `""` so the `NoTextFoundException` branch fires. The sample `no-text-alpha.png` is **copied to a temp path first** because `construct_ocrmypdf_parameters()` rewrites alpha images in place, and the source sample must remain unmodified (read‑only mandate). Producing command:

```
$ python3 -m pytest paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py::BlitzyOcrProbe::test_q3_ocr_fallback_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code — temporary file `paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py`, module preamble plus the `test_q3_ocr_fallback_probe` method (verbatim as executed):

```python
import os
import shutil
import tempfile
from unittest import mock

import magic
from django.test import TestCase

from documents.tests.utils import DirectoriesMixin
from paperless_tesseract.parsers import NoTextFoundException
from paperless_tesseract.parsers import RasterisedDocumentParser

SAMPLES = os.path.join(os.path.dirname(__file__), "samples")


class BlitzyOcrProbe(DirectoriesMixin, TestCase):

    def test_q3_ocr_fallback_probe(self):
        # copy the sample so the in-place alpha rewrite never touches the source
        workdir = tempfile.mkdtemp()
        work_png = os.path.join(workdir, "no-text-alpha.png")
        shutil.copyfile(os.path.join(SAMPLES, "no-text-alpha.png"), work_png)
        print("Q3PROBE input_sample=%s" % os.path.basename(work_png))

        parser = RasterisedDocumentParser(None)
        with mock.patch("ocrmypdf.ocr") as mock_ocr, mock.patch.object(
            RasterisedDocumentParser, "extract_text", return_value="",
        ):
            with self.assertLogs("paperless.parsing", level="WARNING") as cm:
                parser.parse(work_png, "image/png")

        print("Q3PROBE ocrmypdf_ocr_call_count=%d" % mock_ocr.call_count)
        primary = mock_ocr.call_args_list[0].kwargs
        fallback = mock_ocr.call_args_list[1].kwargs
        print("Q3PROBE primary_call_force_ocr=%s" % primary.get("force_ocr"))
        print("Q3PROBE primary_call_keys=%s" % sorted(primary.keys()))
        print("Q3PROBE fallback_call_force_ocr=%s" % fallback.get("force_ocr"))
        print("Q3PROBE fallback_input_file=%s" % os.path.basename(fallback.get("input_file")))
        warn = [m for m in cm.output if "Attempting force OCR" in m][0]
        # strip the "WARNING:paperless.parsing:" prefix for the raw message
        print("Q3PROBE warning_log=%s" % warn.split(":", 2)[-1])
        print("Q3PROBE NoTextFoundException_str=%r" % str(NoTextFoundException("No text was found in the original document")))
```

**Claim — the input sample fed to the probe is `no-text-alpha.png`.**

```python
print("Q3PROBE input_sample=%s" % os.path.basename(work_png))
```
```
Q3PROBE input_sample=no-text-alpha.png
```

**Claim — `ocrmypdf.ocr` is invoked exactly TWICE** — the primary call (`parsers.py:261`) plus the force‑OCR fallback (`parsers.py:298`).

```python
print("Q3PROBE ocrmypdf_ocr_call_count=%d" % mock_ocr.call_count)
```
```
Q3PROBE ocrmypdf_ocr_call_count=2
```

**Claim — the primary call does NOT set `force_ocr`; it carries `skip_text` instead** (under the default `OCR_MODE="skip"`, `skip_text = True` is set at `parsers.py:157-158`).

```python
print("Q3PROBE primary_call_force_ocr=%s" % primary.get("force_ocr"))
print("Q3PROBE primary_call_keys=%s" % sorted(primary.keys()))
```
```
Q3PROBE primary_call_force_ocr=None
Q3PROBE primary_call_keys=['clean', 'deskew', 'image_dpi', 'input_file', 'jobs', 'language', 'output_file', 'output_type', 'progress_bar', 'rotate_pages', 'rotate_pages_threshold', 'sidecar', 'skip_text', 'use_threads']
```

`force_ocr` is absent (reported as `None`) while `skip_text` is present in the key list — confirming the primary call uses `skip_text`, not `force_ocr`.

**Claim — the fallback call sets `force_ocr=True`** (`parsers.py:155-156`), re‑using the same input file `no-text-alpha.png`.

```python
print("Q3PROBE fallback_call_force_ocr=%s" % fallback.get("force_ocr"))
print("Q3PROBE fallback_input_file=%s" % os.path.basename(fallback.get("input_file")))
```
```
Q3PROBE fallback_call_force_ocr=True
Q3PROBE fallback_input_file=no-text-alpha.png
```

**Claim — the fallback is triggered by the exact warning, whose text embeds the exception message `No text was found in the original document`** (exception raised at `parsers.py:267`; caught at `parsers.py:276`; warning "Attempting force OCR to get the text." emitted at `parsers.py:280`).

```python
warn = [m for m in cm.output if "Attempting force OCR" in m][0]
print("Q3PROBE warning_log=%s" % warn.split(":", 2)[-1])
print("Q3PROBE NoTextFoundException_str=%r"
      % str(NoTextFoundException("No text was found in the original document")))
```
```
Q3PROBE warning_log=Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
Q3PROBE NoTextFoundException_str='No text was found in the original document'
```

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

**Observed — the detected MIME string for PDF samples.** A temporary probe calls `magic.from_file(..., mime=True)` — the exact function used by the consumer at `consumer.py:219` — on three sample PDFs. Producing command:

```
$ python3 -m pytest paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py::BlitzyOcrProbe::test_q3_mime_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code — the `test_q3_mime_probe` method of the same `blitzy_adhoc_test_ocr_probe.py` file (`magic` is imported at module top; `SAMPLES` defined as shown in Q3(a)), verbatim as executed:

```python
def test_q3_mime_probe(self):
    for name in ["simple-digital.pdf", "with-form.pdf", "multi-page-images.pdf"]:
        mt = magic.from_file(os.path.join(SAMPLES, name), mime=True)
        print("Q3PROBE magic.from_file(%s)=%r" % (name, mt))
```

**Claim — `magic.from_file("simple-digital.pdf", mime=True)` returns `'application/pdf'`.**

```
Q3PROBE magic.from_file(simple-digital.pdf)='application/pdf'
```

**Claim — `magic.from_file("with-form.pdf", mime=True)` returns `'application/pdf'`.**

```
Q3PROBE magic.from_file(with-form.pdf)='application/pdf'
```

**Claim — `magic.from_file("multi-page-images.pdf", mime=True)` returns `'application/pdf'`.**

```
Q3PROBE magic.from_file(multi-page-images.pdf)='application/pdf'
```

For each PDF sample, `magic.from_file(..., mime=True)` returns exactly **`application/pdf`**, which is the value stored on the resulting `Document` (`consumer.py:401`). The "no‑extractable‑text" condition does **not** change the assigned MIME type — it changes only the OCR path (Q3(a)); the MIME type reflects the *container* format detected on the input file.


---

## Q4 — Barcode splitting

### Q4(a) — How many document records result from a single input?

**Answer:** `(number_of_separators) + 1` output PDFs, with each separator (barcode) page **removed**. `separate_pages()` (def `src/documents/tasks.py:113`) writes `{fname}_document_0.pdf` for the pages before the first separator (`tasks.py:134`), then loops writing `{fname}_document_{count+1}.pdf` (`tasks.py:154`), skipping each barcode page via `for page in range(page_number + 1, next_page):` (`tasks.py:149`), and returns `document_paths` (`tasks.py:161`).

**Run marker** — `test_separate_pages` (`test_tasks.py:305`) calls `separate_pages(patch-code-t-middle.pdf, [1])` and asserts `len(pages) == 2` (`test_tasks.py:313`). Producing command:

```
$ python3 -m pytest documents/tests/test_tasks.py -k "separate_pages" -o addopts="" -v
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 50%]
documents/tests/test_tasks.py::TestTasks::test_separate_pages_no_list PASSED [100%]
================= 2 passed, 38 deselected, 6 warnings in 1.58s =================
```

**Observed count and filenames.** A temporary probe calls `separate_pages` directly on `patch-code-t-middle.pdf` (one separator at page index `1`). Producing command:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_barcode_probe.py::BlitzyBarcodeProbe::test_q4_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code — the `separate_pages` portion of `test_q4_probe` (temporary file `documents/tests/blitzy_adhoc_test_barcode_probe.py`; `BARCODES = os.path.join(SAMPLES, "barcodes")`, and `_scan(relpath)` wraps `tasks.scan_file_for_separating_barcodes(os.path.join(SAMPLES, relpath))`), verbatim as executed:

```python
middle = os.path.join(BARCODES, "patch-code-t-middle.pdf")
seps = self._scan("barcodes/patch-code-t-middle.pdf")
print("Q4PROBE separators_for_middle=%s (num_separators=%d)" % (seps, len(seps)))
out = tasks.separate_pages(middle, [1])
print("Q4PROBE separate_pages_output_count=%d" % len(out))
print("Q4PROBE separate_pages_output_names=%s" % [os.path.basename(p) for p in out])
empty = tasks.separate_pages(middle, [])
print("Q4PROBE separate_pages_empty_list=%s" % empty)
```

**Claim — `patch-code-t-middle.pdf` has exactly one separator, at page index `1`.**

```python
print("Q4PROBE separators_for_middle=%s (num_separators=%d)" % (seps, len(seps)))
```
```
Q4PROBE separators_for_middle=[1] (num_separators=1)
```

**Claim — one separator produces `N+1 = 2` output documents.**

```python
print("Q4PROBE separate_pages_output_count=%d" % len(out))
```
```
Q4PROBE separate_pages_output_count=2
```

**Claim — the two outputs are named `_document_0.pdf` and `_document_1.pdf`** (the separator page itself is removed).

```python
print("Q4PROBE separate_pages_output_names=%s" % [os.path.basename(p) for p in out])
```
```
Q4PROBE separate_pages_output_names=['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
```

**Claim — an EMPTY separator list returns `[]`** (no split performed).

```python
print("Q4PROBE separate_pages_empty_list=%s" % empty)
```
```
Q4PROBE separate_pages_empty_list=[]
```

The empty-list case also logs a warning — `test_separate_pages_no_list` (`test_tasks.py:315`) asserts `pages == []` and `cm.output == ["WARNING:paperless.tasks:No pages to split on!"]` (`test_tasks.py:328`).

### Q4(b) — Which barcode values trigger a split?

**Answer:** The default trigger string is **`"PATCHT"`**: `CONSUMER_BARCODE_STRING = os.getenv("PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")` — `src/paperless/settings.py:506`. The whole feature is gated by `CONSUMER_ENABLE_BARCODES` — `src/paperless/settings.py:502` (default `False`).

**Observed — default settings.** The same `test_q4_probe` (producing command as in Q4(a)) reads the two settings directly. Producing code:

```python
print("Q4PROBE default_CONSUMER_BARCODE_STRING=%r" % settings.CONSUMER_BARCODE_STRING)
print("Q4PROBE default_CONSUMER_ENABLE_BARCODES=%s" % settings.CONSUMER_ENABLE_BARCODES)
```

**Claim — the default trigger string is `'PATCHT'`** (`settings.py:506`).

```
Q4PROBE default_CONSUMER_BARCODE_STRING='PATCHT'
```

**Claim — the barcode feature is gated off by default (`CONSUMER_ENABLE_BARCODES=False`)** (`settings.py:502`).

```
Q4PROBE default_CONSUMER_ENABLE_BARCODES=False
```

**Observed — `scan_file_for_separating_barcodes(...)` return list per named sample (default `PATCHT`).** Producing code — the per-sample scan loop in `test_q4_probe` (each line calls the `_scan` helper shown above):

```python
print("Q4PROBE scan[barcodes/patch-code-t.pdf]=%s" % self._scan("barcodes/patch-code-t.pdf"))
print("Q4PROBE scan[simple.pdf]=%s" % self._scan("simple.pdf"))
print("Q4PROBE scan[barcodes/patch-code-t-middle.pdf]=%s" % self._scan("barcodes/patch-code-t-middle.pdf"))
print("Q4PROBE scan[barcodes/several-patcht-codes.pdf]=%s" % self._scan("barcodes/several-patcht-codes.pdf"))
print("Q4PROBE scan[barcodes/patch-code-t-middle_reverse.pdf]=%s" % self._scan("barcodes/patch-code-t-middle_reverse.pdf"))
print("Q4PROBE scan[barcodes/patch-code-t-qr.pdf]=%s" % self._scan("barcodes/patch-code-t-qr.pdf"))
```

**Claim — `patch-code-t.pdf` → `[0]`** (PATCHT on page 0; asserted `test_tasks.py:207-215`).

```
Q4PROBE scan[barcodes/patch-code-t.pdf]=[0]
```

**Claim — `simple.pdf` → `[]`** (no PATCHT barcode; `test_tasks.py:217-220`).

```
Q4PROBE scan[simple.pdf]=[]
```

**Claim — `patch-code-t-middle.pdf` → `[1]`** (PATCHT on page 1; `test_tasks.py:222-230`).

```
Q4PROBE scan[barcodes/patch-code-t-middle.pdf]=[1]
```

**Claim — `several-patcht-codes.pdf` → `[2, 5]`** (two separators; `test_tasks.py:232-240`).

```
Q4PROBE scan[barcodes/several-patcht-codes.pdf]=[2, 5]
```

**Claim — `patch-code-t-middle_reverse.pdf` → `[1]`** (upside‑down PATCHT; `test_tasks.py:242-250`).

```
Q4PROBE scan[barcodes/patch-code-t-middle_reverse.pdf]=[1]
```

**Claim — `patch-code-t-qr.pdf` → `[0]`** (QR‑encoded PATCHT; `test_tasks.py:252-260`).

```
Q4PROBE scan[barcodes/patch-code-t-qr.pdf]=[0]
```

**Observed — a custom trigger string only splits when configured.** Under the default `PATCHT`, a custom‑barcode sample does **not** trigger; overriding `CONSUMER_BARCODE_STRING="CUSTOM BARCODE"` (via `@override_settings`) makes the custom samples trigger at page `0`. Two producing commands:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_barcode_probe.py::BlitzyBarcodeProbe::test_q4_custom_default_probe \
    -o addopts='' -p no:cacheprovider -s -q
$ python3 -m pytest documents/tests/blitzy_adhoc_test_barcode_probe.py::BlitzyBarcodeProbe::test_q4_custom_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code (two methods):

```python
def test_q4_custom_default_probe(self):
    # same custom sample under the DEFAULT PATCHT string -> no split
    print("Q4PROBE scan[barcode-39-custom.pdf, default PATCHT]=%s" % self._scan("barcodes/barcode-39-custom.pdf"))

@override_settings(CONSUMER_BARCODE_STRING="CUSTOM BARCODE")
def test_q4_custom_probe(self):
    print("Q4PROBE override_CONSUMER_BARCODE_STRING=%r" % settings.CONSUMER_BARCODE_STRING)
    print("Q4PROBE scan[barcode-39-custom.pdf, CUSTOM BARCODE]=%s" % self._scan("barcodes/barcode-39-custom.pdf"))
    print("Q4PROBE scan[barcode-qr-custom.pdf, CUSTOM BARCODE]=%s" % self._scan("barcodes/barcode-qr-custom.pdf"))
    print("Q4PROBE scan[barcode-128-custom.pdf, CUSTOM BARCODE]=%s" % self._scan("barcodes/barcode-128-custom.pdf"))
```

**Claim — under the default `PATCHT`, `barcode-39-custom.pdf` → `[]`** (its Code‑39 "CUSTOM BARCODE" is not `PATCHT`; `test_tasks.py:295-303`).

```
Q4PROBE scan[barcode-39-custom.pdf, default PATCHT]=[]
```

**Claim — the override sets the trigger string to `'CUSTOM BARCODE'`.**

```
Q4PROBE override_CONSUMER_BARCODE_STRING='CUSTOM BARCODE'
```

**Claim — under `CUSTOM BARCODE`, `barcode-39-custom.pdf` → `[0]`** (`test_tasks.py:263`).

```
Q4PROBE scan[barcode-39-custom.pdf, CUSTOM BARCODE]=[0]
```

**Claim — under `CUSTOM BARCODE`, `barcode-qr-custom.pdf` → `[0]`** (`test_tasks.py:274`).

```
Q4PROBE scan[barcode-qr-custom.pdf, CUSTOM BARCODE]=[0]
```

**Claim — under `CUSTOM BARCODE`, `barcode-128-custom.pdf` → `[0]`** (`test_tasks.py:285`).

```
Q4PROBE scan[barcode-128-custom.pdf, CUSTOM BARCODE]=[0]
```

The 18 barcode sample files live in `src/documents/tests/samples/barcodes/` (including `barcode-39-PATCHT.png`, `patch-code-t.pdf`, `patch-code-t-middle.pdf`, `several-patcht-codes.pdf`, `barcode-128-custom.pdf`).

### Q4(c) — Where in the code is the decision made?

**Answer:** The split decision executes at **`if separator_barcode in current_barcodes:`** — `src/documents/tasks.py:108`, inside `scan_file_for_separating_barcodes` (def `src/documents/tasks.py:96`), where:

- `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` — `src/documents/tasks.py:102`
- `current_barcodes = barcode_reader(page)` — `src/documents/tasks.py:107`
- `barcode_reader` (def `src/documents/tasks.py:75`) decodes via `pyzbar.decode(image)` — `src/documents/tasks.py:82`

Orchestration is `consume_file` (def `src/documents/tasks.py:184`): the gate `if settings.CONSUMER_ENABLE_BARCODES:` (`tasks.py:195`) → `scan_file_for_separating_barcodes(path)` (`tasks.py:198`) → `separate_pages(path, separators)` (`tasks.py:201`) → each split saved via `save_to_dir(...)` (`tasks.py:210`).

### Q4(d) — How does it change the effective training data during the run?

**Answer:** Each split file is **re‑consumed as its own `Document`** (via `save_to_dir(...)` at `tasks.py:210`, which drops each split PDF back into consumption). So a single input becomes `N+1` `Document` rows, which changes the training‑set count read by `classifier.py:125` (`Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)`), which changes the SHA‑1 `data_hash` (`classifier.py:161-164`), and therefore forces a **retrain** on the next `train()`.

This links **Q4 → Q1/Q2 directly**, and the link is empirically demonstrated by the Q1 probe: adding a single document changed the observed `data_hash` from `b41ce397…c74447` (2 eligible docs) to `1c7c836d…f7b0` (3 eligible docs) and flipped the next `train()` from a reuse (`False`) to a retrain (`True`). Thus barcode splitting's `N+1` documents feed both the reuse‑vs‑retrain decision (Q1) and the correspondent training labels (Q2).


---

## Non‑determinism root cause (two independent vectors, observed at realistic magnitude)

> This is a **diagnosis‑only** section. No source was modified — no `random_state` was added, no xdist configuration was changed. Two candidate vectors were investigated, and **exactly what was observed is reported below**, including the fact that neither vector reproduced an *actual* test failure at the magnitudes run.

### Vector 1 — `MLPClassifier(tol=0.01)` constructed WITHOUT a `random_state`

The three classifiers are all constructed without a `random_state`:

- `self.tags_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:219`
- `self.correspondent_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:227`
- `self.document_type_classifier = MLPClassifier(tol=0.01)` — `src/documents/classifier.py:238`

With `random_state=None`, scikit‑learn's `MLPClassifier` uses non‑deterministic weight/bias initialization (and batch sampling for the `adam`/`sgd` solvers), so a retrained model's internal weights differ from run to run.

**Observed — the randomness is REAL: `random_state` is `None` and the trained weights differ run‑to‑run.** A probe trained two fresh `DocumentClassifier` instances on identical fixture data and compared the correspondent network's first‑layer weight matrix. Producing command:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_nd3_weights_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code (temporary probe method, subclass of the real `TestClassifier` so it reuses `generate_test_data()`):

```python
def test_nd3_weights_probe(self):
    self.generate_test_data()
    a = DocumentClassifier(); a.train()
    b = DocumentClassifier(); b.train()
    print("ND3PROBE correspondent_random_state=%s" % a.correspondent_classifier.random_state)
    print("ND3PROBE tags_random_state=%s" % a.tags_classifier.random_state)
    print("ND3PROBE document_type_random_state=%s" % a.document_type_classifier.random_state)
    wa = a.correspondent_classifier.coefs_[0]
    wb = b.correspondent_classifier.coefs_[0]
    print("ND3PROBE correspondent_weights_identical_run_to_run=%s" % bool(np.array_equal(wa, wb)))
    print("ND3PROBE correspondent_weights_max_abs_diff=%.6f" % float(np.max(np.abs(wa - wb))))
    pa = a.predict_correspondent(self.doc1.content)
    pb = b.predict_correspondent(self.doc1.content)
    print("ND3PROBE predict_doc1_modelA=%s modelB=%s" % (int(pa), int(pb)))
```

**Claim — no seed is set: `random_state` is `None` on all three fitted estimators** (`classifier.py:219,227,238`).

```python
print("ND3PROBE correspondent_random_state=%s" % a.correspondent_classifier.random_state)
print("ND3PROBE tags_random_state=%s" % a.tags_classifier.random_state)
print("ND3PROBE document_type_random_state=%s" % a.document_type_classifier.random_state)
```
```
ND3PROBE correspondent_random_state=None
ND3PROBE tags_random_state=None
ND3PROBE document_type_random_state=None
```

**Claim — two independent trainings on identical data produce different weight matrices (`identical_run_to_run=False`).**

```python
print("ND3PROBE correspondent_weights_identical_run_to_run=%s" % bool(np.array_equal(wa, wb)))
```
```
ND3PROBE correspondent_weights_identical_run_to_run=False
```

**Claim — the first‑layer weight matrices differ materially; a representative observed max absolute difference is `0.484841`.**

```python
print("ND3PROBE correspondent_weights_max_abs_diff=%.6f" % float(np.max(np.abs(wa - wb))))
```
```
ND3PROBE correspondent_weights_max_abs_diff=0.484841
```

> Reported exactly as observed (R7): this `max_abs_diff` figure **changes on every run** (earlier runs in this investigation produced `0.508544` and `0.433783`) — that run‑to‑run variability *is* the non‑determinism being demonstrated. The reproducible invariants are `random_state=None` and `identical_run_to_run=False`; the specific magnitude is inherently non‑reproducible.

**Claim — despite the differing weights, both models still predict `c1.pk=1` on `doc1`.**

```python
print("ND3PROBE predict_doc1_modelA=%s modelB=%s" % (int(pa), int(pb)))
```
```
ND3PROBE predict_doc1_modelA=1 modelB=1
```

**Observed — at magnitude, the varying weights did NOT change predictions on the current fixtures.** A probe built the canonical fixture once and trained a fresh classifier `N=100` times, tallying deviations from the expected labels. Producing command:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_nd1_magnitude_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code (temporary probe method):

```python
def test_nd1_magnitude_probe(self):
    self.generate_test_data()
    N = 100
    exp_corr, exp_dt, exp_tags = self.c1.pk, self.dt.pk, [self.t1.pk]
    print("ND1PROBE N=%d" % N)
    print("ND1PROBE expected correspondent=c1.pk=%d document_type=dt.pk=%d tags=[t1.pk]=%s"
          % (exp_corr, exp_dt, exp_tags))
    corr_dev = dt_dev = tag_dev = 0
    dist = {}
    for _ in range(N):
        c = DocumentClassifier(); c.train()
        pc = c.predict_correspondent(self.doc1.content)
        pcv = None if pc is None else int(pc)
        dist[str(pcv)] = dist.get(str(pcv), 0) + 1
        if pcv != exp_corr: corr_dev += 1
        if c.predict_document_type(self.doc1.content) != exp_dt: dt_dev += 1
        if c.predict_tags(self.doc1.content) != exp_tags: tag_dev += 1
    print("ND1PROBE predict_correspondent DEVIATIONS=%d/%d" % (corr_dev, N))
    print("ND1PROBE predict_correspondent value_distribution=%s" % dist)
    print("ND1PROBE predict_document_type DEVIATIONS=%d/%d" % (dt_dev, N))
    print("ND1PROBE predict_tags DEVIATIONS=%d/%d" % (tag_dev, N))
```

**Claim — the expected labels are `correspondent=c1.pk=1`, `document_type=dt.pk=1`, `tags=[t1.pk]=[12]`.**

```
ND1PROBE N=100
ND1PROBE expected correspondent=c1.pk=1 document_type=dt.pk=1 tags=[t1.pk]=[12]
```

**Claim — over 100 fresh trainings, `predict_correspondent` had `0/100` deviations (always `1`).**

```
ND1PROBE predict_correspondent DEVIATIONS=0/100
ND1PROBE predict_correspondent value_distribution={'1': 100}
```

**Claim — over 100 fresh trainings, `predict_document_type` had `0/100` deviations.**

```
ND1PROBE predict_document_type DEVIATIONS=0/100
```

**Claim — over 100 fresh trainings, `predict_tags` had `0/100` deviations.**

```
ND1PROBE predict_tags DEVIATIONS=0/100
```

Repeating on the *harder* fixture from `test_one_correspondent_predict_manydocs` (`test_classifier.py:206`), where the two documents differ by a single word — `"this is a document from c1"` vs. `"this is a document from noone"` — at `N=200`. Producing command:

```
$ python3 -m pytest documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_nd2_manydocs_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code (temporary probe method — builds its own single‑word‑diff fixture):

```python
def test_nd2_manydocs_probe(self):
    c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    doc1 = Document.objects.create(title="doc1", content="this is a document from c1",
                                   correspondent=c1, checksum="A")
    doc2 = Document.objects.create(title="doc2", content="this is a document from noone", checksum="B")
    N = 200
    print("ND2PROBE N=%d c1.pk=%d" % (N, c1.pk))
    d1_dev = d2_dev = 0; dist1 = {}; dist2 = {}
    for _ in range(N):
        c = DocumentClassifier(); c.train()
        p1 = c.predict_correspondent(doc1.content); p1v = None if p1 is None else int(p1)
        dist1[str(p1v)] = dist1.get(str(p1v), 0) + 1
        if p1v != c1.pk: d1_dev += 1
        p2 = c.predict_correspondent(doc2.content); p2v = None if p2 is None else int(p2)
        dist2[str(p2v)] = dist2.get(str(p2v), 0) + 1
        if p2v is not None: d2_dev += 1
    print("ND2PROBE doc1_expected=c1.pk=%d DEVIATIONS=%d/%d distribution=%s" % (c1.pk, d1_dev, N, dist1))
    print("ND2PROBE doc2_expected=None DEVIATIONS=%d/%d distribution=%s" % (d2_dev, N, dist2))
```

**Claim — `doc1` (expected `c1.pk=1`) had `0/200` deviations (always `1`).**

```
ND2PROBE N=200 c1.pk=1
ND2PROBE doc1_expected=c1.pk=1 DEVIATIONS=0/200 distribution={'1': 200}
```

**Claim — `doc2` (expected `None`) had `0/200` deviations (always `None`).**

```
ND2PROBE doc2_expected=None DEVIATIONS=0/200 distribution={'None': 200}
```

**Observed — the real exact‑equality tests do not flake at 50 repetitions each.** Three probe methods replicate the exact‑equality assertions of `testPredict`, `test_one_correspondent_predict`, and `test_one_correspondent_predict_manydocs`, each against a freshly retrained classifier, 50 times. Producing command:

```
$ python3 -m pytest \
    documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_loop_testpredict_probe \
    documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_loop_one_correspondent_probe \
    documents/tests/blitzy_adhoc_test_probe.py::BlitzyClassifierProbe::test_loop_one_correspondent_manydocs_probe \
    -o addopts='' -p no:cacheprovider -s -q
```

Producing code (the core of each loop method; `N = 50`, fresh `DocumentClassifier().train()` per iteration):

```python
# test_loop_testpredict_probe — replicates testPredict's correspondent asserts
ok = (c.predict_correspondent(self.doc1.content) == self.c1.pk
      and c.predict_correspondent(self.doc2.content) is None)
# test_loop_one_correspondent_probe — replicates test_one_correspondent_predict
if c.predict_correspondent(doc1.content) == c1.pk: passed += 1
# test_loop_one_correspondent_manydocs_probe — replicates the manydocs variant
if c.predict_correspondent(doc1.content) == c1.pk and c.predict_correspondent(doc2.content) is None:
    passed += 1
```

**Claim — the `testPredict` assertions passed `50/50`.**

```
TESTPREDICT_LOOP pass=50 fail=0 out_of=50
```

**Claim — the `test_one_correspondent_predict` assertion passed `50/50`.**

```
LOOP test_one_correspondent_predict: pass=50 fail=0 out_of=50
```

**Claim — the `test_one_correspondent_predict_manydocs` assertions passed `50/50`.**

```
LOOP test_one_correspondent_predict_manydocs: pass=50 fail=0 out_of=50
```

**Control — the loaded‑model path is deterministic.** `test_load_and_classify` (`test_classifier.py:183`) *loads* the pre‑trained `model.pickle` (`new_classifier.load()`) instead of retraining, so its weights are fixed. This ran the real test 20 times via a shell loop. Producing command:

```
$ for i in $(seq 1 20); do \
    python3 -m pytest documents/tests/test_classifier.py::TestClassifier::test_load_and_classify \
      -o addopts="" -p no:cacheprovider -q 2>&1 | tail -1; \
  done   # tallied into passed/skipped/failed
```

**Claim — the loaded‑model test passed all `20/20` runs (0 skipped, 0 failed).**

```
LOAD_AND_CLASSIFY_LOOP passed=20 skipped=0 failed=0 out_of=20
```

20/20 deterministic — confirming that any flakiness would originate from **retraining** (random weight init), not from the prediction step or the loaded‑model path.

### Vector 2 — pytest‑xdist `--numprocesses auto`

The suite default enables xdist parallelism: `addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet` — `src/setup.cfg:10`. Parallel workers can surface isolation issues invisible in serial runs.

**Observed — no failure‑rate difference between serial and parallel** for the full `documents/tests/test_classifier.py` (23 tests collected; 22 run, 1 skipped), 10 runs each. Producing commands (two shell loops — one single‑process, one xdist `-n auto`):

```
$ for i in $(seq 1 10); do \
    python3 -m pytest documents/tests/test_classifier.py -o addopts="" -p no:cacheprovider -q 2>&1 | tail -1; \
  done   # SERIAL (single process)
$ for i in $(seq 1 10); do \
    python3 -m pytest documents/tests/test_classifier.py -o addopts="" -n auto -p no:cacheprovider -q 2>&1 | tail -1; \
  done   # PARALLEL (pytest-xdist)
```

**Claim — 10 serial runs all passed (`10/10`), sample summary `22 passed, 1 skipped, 6 warnings in 2.53s`.**

```
SERIAL_RESULT pass=10 fail=0 out_of=10
sample: 22 passed, 1 skipped, 6 warnings in 2.53s
```

**Claim — 10 parallel runs (pytest‑xdist `3.8.0`) all passed (`10/10`), sample summary `22 passed, 1 skipped, 774 warnings in 25.56s`.**

```
xdist_version=3.8.0
PARALLEL_RESULT pass=10 fail=0 out_of=10
sample: 22 passed, 1 skipped, 774 warnings in 25.56s
```

**Claim — the one skipped test is `test_load_classifier_cached`, skipped via `@pytest.mark.skip(...)` at `test_classifier.py:399-401` (def at `:402`), reason `"Disabled caching due to high memory usage - need to investigate."`** — unrelated to flakiness.

```
$ sed -n '399,402p' documents/tests/test_classifier.py
    @pytest.mark.skip(
        reason="Disabled caching due to high memory usage - need to investigate.",
    )
    def test_load_classifier_cached(self):
```

**Isolation / ordering facts observed:**

- No `conftest.py` exists under `src/` (a `find` for it returned nothing). Producing command + output:

```
$ find . -name conftest.py | wc -l
0
```

- **pytest‑randomly is not installed**; the observed plugin set is `pytest-cov 7.0.0`, `pytest-django 4.11.1`, `pytest-env 1.1.5`, `pytest-sugar 1.1.1`, `pytest-xdist 3.8.0` (on `pytest 8.4.2`). Producing command + output:

```
$ pip list 2>/dev/null | grep -Ei 'pytest'
pytest                 8.4.2
pytest-cov             7.0.0
pytest-django          4.11.1
pytest-env             1.1.5
pytest-sugar           1.1.1
pytest-xdist           3.8.0
```

(No `pytest-randomly` line appears.) There is also **no `.python-version`** file. Test ordering is therefore stable except for xdist worker distribution.

- `DirectoriesMixin` (`utils.py:72`) overrides `DATA_DIR`/`SCRATCH_DIR`/`MEDIA_ROOT`/`MODEL_FILE` per test (`utils.py:35-47`, `MODEL_FILE` at `utils.py:45`) and Django `TestCase` rolls back the DB, so cross‑test model‑file bleed is unlikely. This isolation does **not** remove the within‑test weight‑init randomness of Vector 1.

### Conclusion (reported exactly as observed)

- The **code‑level non‑determinism vector is Vector 1**: the `MLPClassifier` instances are trained with `random_state=None` (`classifier.py:219,227,238`), and this randomness is empirically real — two trainings on identical data produced different weight matrices (`max_abs_diff=0.484841` in the representative run reported above; the figure varies each run, which is the point). This is the mechanism that *can* make the exact‑equality classifier assertions (`testPredict` at `test_classifier.py:115`, `test_one_correspondent_predict` at `:191`, `test_one_correspondent_predict_manydocs` at `:206`) flaky when the training set is larger or less separable.
- **However, at the magnitudes run** (200 fresh trainings on two fixtures with 0 deviations; 50 repetitions each of three real exact‑equality tests with 0 failures; 10 serial and 10 parallel full‑suite runs with 0 failures), **neither vector reproduced an actual failure**. The current fixtures are tiny and linearly separable, so the decision boundary is robust even though the underlying weights vary.
- **Vector 2 (xdist)** showed no difference in failure frequency between serial and parallel execution and is mitigated by `DirectoriesMixin` isolation + DB rollback.
- Practical implication for whoever fixes the flakiness (out of scope here, stated for the reader): the latent risk lives in the unseeded `MLPClassifier`; the observed stability is a property of these specific small fixtures, not a guarantee. Because the mechanism is confirmed present but did not fire at this scale, the exact real‑world trigger (e.g. a larger/edge‑case training set, or a specific worker interleaving) **could not be reproduced by reading or running the code at the scale attempted**, and is reported as such rather than asserted.

### External references checked

The two non‑determinism vectors depend on documented behavior of external packages. The **primary evidence for this repository's behavior is the runtime/source observation above**; the official documentation below is cited only to substantiate the general package semantics that the observations rely on.

- **scikit‑learn `MLPClassifier.random_state`** — official API reference: <https://scikit-learn.org/stable/modules/generated/sklearn.neural_network.MLPClassifier.html>. The documented constructor default is `random_state=None`. Per the official parameter description, `random_state` governs the random number generation used for the network's weight and bias initialization (and for batch sampling under the `sgd`/`adam` solvers); the documentation states to "Pass an int for reproducible results across multiple function calls." paperless‑ngx constructs `MLPClassifier(tol=0.01)` with no `random_state` (`src/documents/classifier.py:219,227,238`), so per the official semantics the weight/bias initialization is seeded from an unspecified source and is not reproducible across runs — exactly the Vector‑1 behavior observed above (`correspondent_weights_identical_run_to_run=False`, `max_abs_diff=0.484841`). The pinned runtime is scikit‑learn `1.0.2`, whose `MLPClassifier` carries the same `random_state=None` default (verified in‑container below).

- **pytest‑xdist worker/parallel behavior** — official documentation: <https://pytest-xdist.readthedocs.io/en/stable/distribution.html>. Per the official docs, running with `-n auto` spawns one worker process per available CPU, each worker being a **separate process** with its own Python interpreter, and the default `--dist load` scheduler distributes pending tests to whichever worker is free with no guaranteed ordering. This is the documented basis for why parallelism can surface isolation bugs (shared files/DB/ports) that a serial run hides. The pinned runtime is pytest‑xdist `3.8.0`; `src/setup.cfg:10` sets `--numprocesses auto`. As observed above, at the attempted magnitude the paperless‑ngx suite showed no serial‑vs‑parallel failure difference because `DirectoriesMixin` (`src/documents/tests/utils.py:35-72`) namespaces `MODEL_FILE`/`DATA_DIR` per test and Django's `TestCase` rolls back the DB, removing the shared‑state that xdist would otherwise expose.

Confirmation that the pinned scikit‑learn build carries the `random_state=None` default (run inside the container, not read from the web):

```bash
docker exec -u testuser -w /app/src paperless python3 -c "import sklearn, inspect; from sklearn.neural_network import MLPClassifier; print('sklearn', sklearn.__version__); print('random_state default =', inspect.signature(MLPClassifier).parameters['random_state'].default)"
```

```text
sklearn 1.0.2
random_state default = None
```

The observed default `None` at the pinned version `1.0.2` matches the official documentation, closing the loop between the external reference and the in‑repo behavior.

---

## Coverage pass — every named item answered

Each item below was addressed above with its own verbatim evidence and an exact `file:line` citation.

- [x] **Q1 — reuse vs. retrain mechanism.** SHA‑1 `data_hash` (`classifier.py:124`, digest `:161`); reuse short‑circuit `if self.data_hash and new_data_hash == self.data_hash:` `:163` → `return False` `:164`; retrain sets `self.data_hash = new_data_hash` `:247` → `return True` `:249`.
- [x] **Q1 — `train()` returns `True` then `False`.** `testDatasetHashing` `PASSED` (`test_classifier.py:141-142`); probe hashes `b41ce397…` (True) → same (False) → `1c7c836d…` (True after mutate).
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
- [x] **Non‑determinism — both vectors named + magnitude + which reproduced.** Vector 1 `MLPClassifier(tol=0.01)` no `random_state` (`classifier.py:219/227/238`) — confirmed real (`random_state=None`, weights differ, `max_abs_diff` varies run‑to‑run, e.g. `0.484841`) but 0 deviations at N=100/N=200 and 0 failures over 50× each real test; Vector 2 xdist (`setup.cfg:10`) — 10/10 serial and 10/10 parallel, no difference. Diagnose‑only (nothing modified).
- [x] **Read‑only mandate + cleanup.** All three temporary `blitzy_adhoc_test_*` probe scripts removed from the container; source tree left unchanged. Proven by the pasted `git status --porcelain` / `git diff --stat` / artifact‑search output in **§ Cleanup & read‑only verification** below (both git commands return empty; no probe/log/cache artifacts remain in the source tree).

### Notes on values reported exactly as observed (not adjusted)

- The container is a **detached‑HEAD** checkout at `542221a38`; `git branch --show-current` is empty (the deliverable name uses the task‑supplied source branch `paperless-ngx_542221a38dff`).
- `predict_correspondent` returns a NumPy `array([1])` (not a scalar `1`).
- The non‑determinism vectors are **real in the code** yet **did not reproduce a failure** at the magnitudes run; this is reported as observed rather than forced toward an expected "flaky" outcome. The precise real‑world trigger could not be reproduced at the attempted scale and is flagged as such.
- Barcode logic resides in `src/documents/tasks.py` at this commit (not a separate `barcodes.py` module).

---

## Cleanup & read‑only verification (source tree unchanged)

The investigation used three temporary probe scripts inside the container (`documents/tests/blitzy_adhoc_test_probe.py`, `documents/tests/blitzy_adhoc_test_barcode_probe.py`, `paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py`). Per the read‑only mandate they were **deleted after evidence capture**, and no existing source file was modified. This is proven below with verbatim command output run inside the container against the source repo at `/app`.

**Claim — the three temporary probe scripts were the only untracked additions, and they were removed.** Before removal, `git status --porcelain` listed exactly the three probes; the removal command deleted all three:

```bash
docker exec -u testuser -w /app/src paperless bash -c '
  rm -v documents/tests/blitzy_adhoc_test_barcode_probe.py \
        documents/tests/blitzy_adhoc_test_probe.py \
        paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py'
```

```text
removed 'documents/tests/blitzy_adhoc_test_barcode_probe.py'
removed 'documents/tests/blitzy_adhoc_test_probe.py'
removed 'paperless_tesseract/tests/blitzy_adhoc_test_ocr_probe.py'
```

**Claim — after cleanup, `git status --porcelain` is empty (working tree clean, nothing added or modified).** Producing command and its (empty) output:

```bash
docker exec -u testuser -w /app paperless git status --porcelain
```

```text
```

(No lines printed — the working tree is clean.)

**Claim — `git diff --stat` is empty (no tracked source file was modified).** Producing command and its (empty) output:

```bash
docker exec -u testuser -w /app paperless git diff --stat
```

```text
```

(No lines printed — zero files changed.)

**Claim — no temporary probe scripts, logs, or coverage/cache artifacts remain in the source tree.** Producing command and its (empty) output:

```bash
docker exec -u testuser -w /app paperless bash -c \
  'find /app/src \( -name "blitzy_adhoc_test_*" -o -name "*_probe.py" -o -name ".pytest_cache" -o -name "htmlcov" -o -name ".coverage" \) 2>/dev/null'
```

```text
```

(No lines printed — the git‑ignored `src/.pytest_cache` produced by test runs was also removed; the only git‑ignored runtime file, `/app/data/log/paperless.log`, lives under `/data/` — excluded by `.gitignore:83` and outside the source tree, and is a runtime log, not a repository file.)

**Claim — the source repo remains at the pristine pinned commit `542221a38`.** Producing command and output:

```bash
docker exec -u testuser -w /app paperless git rev-parse --short HEAD
```

```text
542221a38
```

Both git commands return empty and the artifact search finds nothing, confirming the paperless‑ngx source repository is left **exactly** as found (read‑only mandate satisfied). The sole artifact produced by this task is this documentation file, `blitzy/documentation/paperless-ngx_542221a38dff.md`, which is added in the **destination** repository — not in the source tree verified above.
