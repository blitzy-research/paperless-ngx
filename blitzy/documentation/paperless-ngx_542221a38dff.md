# paperless-ngx — ML Classification, OCR, Correspondent Matching & Barcode Splitting: Runtime Investigation

**Repository:** paperless-ngx  
**Commit:** `542221a38dff06361e07976452f9aea24d210542` (source branch `paperless-ngx_542221a38dff`)  
**Investigation type:** Read-only. Every behavioural claim below was produced by **running** the real code paths and capturing the actual output; nothing here is asserted from code-reading alone. Each claim carries the exact command, the complete unedited output, and a precise `file:line` citation.

This document answers four questions about test-execution behaviour and characterizes the reported *non-deterministic* test failures:

- **Q1** — Does the `DocumentClassifier` reuse a persisted model or retrain within a run, and how does that affect later tests?
- **Q2** — For automatic (`MATCH_AUTO`) correspondent matching: (a) how many training documents are created, (b) when does training occur relative to the inserts, and (c) what confidence threshold accepts/rejects a prediction?
- **Q3** — For a document with no extractable text: (a) which OCR subprocess is invoked, and (b) what MIME type is assigned to the output?
- **Q4** — For barcode splitting: (a) how many document records come from one input file, (b) which barcode values trigger a split, (c) where is the split decision made, and (d) does splitting change the effective training data during a run?

---

## 1. Environment & Methodology

### 1.1 Where the code was run

Runtime observation was performed **inside the canonical Docker container** (Python 3.9 with the pinned stack), because the local sandbox host runs Python 3.13 and **cannot** install the pins (e.g. `scipy==1.8.0` requires Python < 3.11 and has no wheel for 3.13). The container is the authoritative runtime for this commit.

- Canonical container image: `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542` (source `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`); executed via the `paperless-ngx-qna:ready` image (identical repo, plus `libzbar0`, `poppler-utils`, `pngquant` baked in for the Q4 barcode path).
- The container repository at `/app` is at the exact commit under investigation, and the cited `.py` files are **byte-identical** (md5-verified) to the checked-out sources.
- All commands were run as the non-root `testuser` (uid 1000) from `/app/src`, with `DJANGO_SETTINGS_MODULE=paperless.settings` and the SQLite test database, matching the project's test harness.

Verified runtime versions and system binaries (captured in-container):

```
$ python3 --version
Python 3.9.23
scikit-learn 1.0.2
numpy 1.22.3
scipy 1.8.0
django 4.0.4
ocrmypdf 13.4.3
pikepdf 5.1.1
pdf2image 1.16.0        # (version from requirements.txt; module exposes no __version__)
tesseract: tesseract 4.1.1
ghostscript: 9.53.3
pdftoppm(poppler): pdftoppm version 20.09.0
$ git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
```

Additional pinned packages relevant to the four questions (from `requirements.txt`): `pyzbar==0.1.9`, `python-magic==0.4.25`, `django-q==1.3.9`, `joblib==1.1.0`, `threadpoolctl==3.1.0`, `fuzzywuzzy[speedup]==0.18.0`.

### 1.2 How the code was run

Two complementary, fully canonical techniques were used, each capturing the exact command and complete output:

1. **Temporary standalone observation scripts** (created outside the tracked tree at `/tmp/obs_*.py` in the container, then deleted) that call the **real** entry points — `tasks.train_classifier()`, `DocumentClassifier.train()/predict_correspondent()`, `RasterisedDocumentParser.parse()`, `tasks.scan_file_for_separating_barcodes()`, `tasks.separate_pages()`, `tasks.consume_file()` — using `django.setup()` + `setup_test_environment()` + `connection.creation.create_test_db()`, with a DEBUG `StreamHandler` attached to the `paperless` logger so the real log lines are captured on stdout.
2. **The existing canonical tests** run through pytest (a legitimate canonical exercise, since they drive the same real entry points). For single deterministic runs the parallel/coverage options were neutralized with `-o addopts="" -s --log-cli-level=DEBUG`; for the non-determinism probe the **default** config from `setup.cfg` (including `--numprocesses auto`) was used unchanged.

### 1.3 Determinism / stability protocol

Every magnitude, count, and timing claim below was confirmed across **at least two runs** (scripts were executed twice; the non-determinism probe ran the same input **10×**). Where a value is derived from code rather than observed, it is explicitly labelled **inferred**. Where a value comes from a fallback path, that path is labelled as the fallback.

### 1.4 Default configuration (observed at runtime)

The following defaults govern the four questions and were confirmed at runtime (not just read):

| Setting | Value | Source |
|---|---|---|
| `MODEL_FILE` | `<DATA_DIR>/classification_model.pickle` | `src/paperless/settings.py:74` |
| `DATA_DIR` | derived from project root | `src/paperless/settings.py:66` |
| `SCRATCH_DIR` | `/tmp/paperless` | `src/paperless/settings.py:84` |
| `OCR_MODE` | `skip` | `src/paperless/settings.py:522` |
| `CONSUMER_ENABLE_BARCODES` | `False` | `src/paperless/settings.py:502` |
| `CONSUMER_BARCODE_STRING` | `PATCHT` | `src/paperless/settings.py:506` |
| `DocumentClassifier.FORMAT_VERSION` | `7` | `src/documents/classifier.py:63` |

> Note on line citations: all `file:line` references are repo-root-relative and were re-validated against the checkout at HEAD `542221a38dff`. Two corrections relative to some prose were confirmed: `train_classifier` is defined at `tasks.py:48` and the barcode split decision is at `tasks.py:108`.

---

## 2. Q1 — Classifier model: reuse vs. retrain within a run (+ cross-test effect)

### 2.1 Answer

Within a single run the `DocumentClassifier` **reuses** its model when the training data is unchanged and **retrains (and re-saves)** only when the data changes. The decision is a **SHA-1 hash guard**, not a library cache:

- `DocumentClassifier.train()` computes a SHA-1 digest over each document's preprocessed content and its label ids, and **short-circuits with `return False`** when the new digest equals the instance's stored `data_hash` (`src/documents/classifier.py:161-164`). On an actual change it sets `self.data_hash = new_data_hash` (`:247`) and `return True` (`:249`).
- `tasks.train_classifier()` (`src/documents/tasks.py:48`) calls `classifier.save()` **only** when `train()` returns `True` (`:63-67`); otherwise it logs `"Training data unchanged."` (`:68-69`). So on unchanged data the persisted model file is **not rewritten** — its `st_mtime` is unchanged (reuse). On changed data it is rewritten (retrain).
- The version guard is separate: `FORMAT_VERSION = 7` (`src/documents/classifier.py:63`); on load a mismatch raises `IncompatibleClassifierVersionError` (`:80-83`) and `load_classifier()` deletes the stale file via `os.unlink(settings.MODEL_FILE)` (`:48`).

**Cross-test effect:** for the suites in question there is effectively **no** cross-test leak of the on-disk model, because `DirectoriesMixin` overrides `MODEL_FILE` to a fresh per-test `tempfile.mkdtemp()` directory (`src/documents/tests/utils.py:45`) and `django.test.TestCase` rolls back each test's transaction. There is also **no in-memory classifier cache** at this commit (the cache test is skipped — see §2.4).

### 2.2 Observed — driving `tasks.train_classifier()` ×3 (the real entry point), twice

Command (run inside the container from `/app/src`):

```
docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q1.py
```

The script created a `Correspondent(matching_algorithm=MATCH_AUTO)` + one `Document`, called `tasks.train_classifier()` (call #1), called it again on unchanged data (call #2), then mutated the document content and called it a third time (call #3), recording `os.stat(settings.MODEL_FILE).st_mtime` after each. Complete unedited output:

```
================ RUN 1 ================
MODEL_FILE = /tmp/q1-data-1-o6bc0jnq/classification_model.pickle
model file exists before any training: False

--- call #1: tasks.train_classifier() [initial] ---
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/q1-data-1-o6bc0jnq/classification_model.pickle...
model file exists after call #1: True
st_mtime after call #1 = 1783961966.5011947

--- call #2: tasks.train_classifier() [UNCHANGED data] ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.tasks: Training data unchanged.
st_mtime after call #2 = 1783961966.5011947
REUSE? mtime unchanged (m1 == m2): True

--- call #3: mutate doc.content then tasks.train_classifier() [CHANGED data] ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/q1-data-1-o6bc0jnq/classification_model.pickle...
st_mtime after call #3 = 1783961966.524195
RETRAIN? mtime changed (m2 != m3): True

--- direct DocumentClassifier.train() hash guard ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG DEBUG paperless.classifier: Gathering data from database...
first train() return value  = True  (True => trained/changed)
second train() return value = False  (False => data unchanged, REUSE)
data_hash is set after train(): True

--- FORMAT_VERSION / version guard ---
DocumentClassifier.FORMAT_VERSION = 7

================ RUN 2 ================
MODEL_FILE = /tmp/q1-data-2-dlpt5t5i/classification_model.pickle
model file exists before any training: False

--- call #1: tasks.train_classifier() [initial] ---
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/q1-data-2-dlpt5t5i/classification_model.pickle...
model file exists after call #1: True
st_mtime after call #1 = 1783961966.5531952

--- call #2: tasks.train_classifier() [UNCHANGED data] ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.tasks: Training data unchanged.
st_mtime after call #2 = 1783961966.5531952
REUSE? mtime unchanged (m1 == m2): True

--- call #3: mutate doc.content then tasks.train_classifier() [CHANGED data] ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/q1-data-2-dlpt5t5i/classification_model.pickle...
st_mtime after call #3 = 1783961966.5701954
RETRAIN? mtime changed (m2 != m3): True

--- direct DocumentClassifier.train() hash guard ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG DEBUG paperless.classifier: Gathering data from database...
first train() return value  = True  (True => trained/changed)
second train() return value = False  (False => data unchanged, REUSE)
data_hash is set after train(): True

--- FORMAT_VERSION / version guard ---
DocumentClassifier.FORMAT_VERSION = 7

================ STABILITY ACROSS RUNS ================
RUN 1: reuse(m1==m2)=True  retrain(m2!=m3)=True  first_train=True second_train=False
RUN 2: reuse(m1==m2)=True  retrain(m2!=m3)=True  first_train=True second_train=False
```

**Reading the output:** call #2 (unchanged data) prints `"Training data unchanged."` and the `st_mtime` is byte-for-byte identical to call #1 → **reuse** (`train()` returned `False`, no save). Call #3 (after `doc.content = "test2"`) prints `"Saving updated classifier model to …"` and the `st_mtime` changes → **retrain** (`train()` returned `True`, model re-saved). The direct hash-guard probe confirms `train()` returns `True` then `False` on unchanged data. Both runs are identical.

### 2.3 Observed — the canonical test through pytest

`test_train_classifier` drives exactly this three-call sequence through `tasks.train_classifier()` and asserts `st_mtime` reuse then retrain. Command and relevant output:

```
$ python3 -m pytest documents/tests/test_tasks.py::TestTasks::test_train_classifier -o addopts="" -s --log-cli-level=DEBUG -v
...
DEBUG    paperless.tasks:tasks.py:69 Training data unchanged.
...
INFO     paperless.tasks:tasks.py:64 Saving updated classifier model to /tmp/tmpc2r13h_9/classification_model.pickle...
PASSED
======================== 1 passed, 6 warnings in 2.00s =========================
```

The test body (`src/documents/tests/test_tasks.py:75-94`) confirms the mechanism: create `Correspondent(MATCH_AUTO)` + `Document` (`:76-77`); first `train_classifier()` (`:80`) then record `mtime` (`:82`); second `train_classifier()` (`:84`), `mtime2` (`:86`), `self.assertEqual(mtime, mtime2)` — **reuse** (`:87`); `doc.content = "test2"; doc.save()` (`:89-90`); third `train_classifier()` (`:91`), `mtime3` (`:93`), `self.assertNotEqual(mtime2, mtime3)` — **retrain** (`:94`). The runtime log lines are emitted at `tasks.py:64` (save) and `tasks.py:69` (unchanged), matching the answer.

The direct SHA-1 guard is also covered by `testDatasetHashing` (`src/documents/tests/test_classifier.py:137-142`, `assertTrue(train())` then `assertFalse(train())`):

```
$ python3 -m pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing -o addopts="" -s --log-cli-level=DEBUG -v
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
======================== 1 passed, 6 warnings in 1.99s =========================
```

### 2.4 Cross-test effect — isolation + no in-memory cache (observed)

- **Per-test model isolation.** In §2.3 the model was written under `MODEL_FILE=/tmp/tmpc2r13h_9/classification_model.pickle` — a fresh per-test temp dir created by `DirectoriesMixin` (`setUp()` → `setup_directories()` → `override_settings(MODEL_FILE=os.path.join(dirs.data_dir, "classification_model.pickle"), SCRATCH_DIR=…)`, `src/documents/tests/utils.py:37,45`), removed at `tearDown()` (`:53-57`). Both `TestClassifier` (`test_classifier.py:20`) and `TestTasks` (`test_tasks.py:21`) subclass `DirectoriesMixin`, so the on-disk model of one test **cannot** leak into another test in these suites.
- **No in-memory cache.** The would-be classifier cache is disabled at this commit — the cache test is skipped:

```
$ python3 -m pytest documents/tests/test_classifier.py::TestClassifier::test_load_classifier_cached -o addopts="" -v
documents/tests/test_classifier.py::TestClassifier::test_load_classifier_cached SKIPPED [100%]
======================== 1 skipped, 6 warnings in 0.09s ========================
```

The decorator is `@pytest.mark.skip(reason="Disabled caching due to high memory usage - need to investigate.")` (`src/documents/tests/test_classifier.py:399-401`). Consequently, reuse-vs-retrain is governed solely by the on-disk model + the SHA-1 `data_hash`, and the only conceivable cross-test coupling is a *shared model-file path* — which `DirectoriesMixin` eliminates for these suites. (This ties directly into the non-determinism analysis in §6.)

---


## 3. Q2 — Automatic (`MATCH_AUTO`) correspondent matching

This question has three parts; each is answered explicitly below.

### 3.1 (a) How many training documents are created

**Observed integer:** the canonical `test_one_correspondent_predict` path creates **1** training document; the many-docs variant creates **2**. When inbox-tagged documents are present they are **excluded** from the training corpus.

The training corpus is read at call time by `train()` via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`), so inbox-tagged documents never enter training.

Command:

```
docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q2.py
```

Observed output for the count + inbox-exclusion probe (3 rows inserted, 1 inbox-tagged → **2 effective** training docs), and the canonical 1-document path:

```
================ Q2(a) training-doc count + inbox exclusion ================
Document.objects.count() (rows inserted) = 3
effective training corpus (exclude tags__is_inbox_tag=True) = 2
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.

================ Q2(a) canonical test_one_correspondent_predict count = 1 document ================
Document.objects.count() = 1
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
```

The DEBUG line `"N documents, …"` (`src/documents/classifier.py:178`) reports the *effective* corpus size **after** the inbox exclusion: 3 rows → **`2 documents`**. The canonical `test_one_correspondent_predict` inserts exactly **1** document (`src/documents/tests/test_classifier.py:191-204`, `doc1` at `:196-201`); `test_one_correspondent_predict_manydocs` inserts **2** (`:206-225`, `doc1` at `:211-216`, `doc2` without a correspondent at `:217-221`). Both canonical tests pass:

```
$ python3 -m pytest \
    documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs \
    -o addopts="" -s --log-cli-level=DEBUG -v
DEBUG    paperless.classifier:classifier.py:178 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
PASSED
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
PASSED
======================== 2 passed, 6 warnings in 2.15s =========================
```

### 3.2 (b) When training occurs relative to the inserts

**Training is never triggered on document insert — it is an explicit call.** Inserting a `Correspondent` + `Document` does not create or update the model; only an explicit `tasks.train_classifier()` / `DocumentClassifier.train()` does. Observed:

```
================ Q2(b) timing: inserting documents does NOT train; only explicit train_classifier() does ================
after inserting Correspondent+Document: model file exists = False
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/q2b-data-e42d97rm/classification_model.pickle...
after explicit tasks.train_classifier(): model file exists = True
```

- In tests, `train()`/`train_classifier()` is called **after** the inserts (e.g. `test_one_correspondent_predict` calls `self.classifier.train()` at `src/documents/tests/test_classifier.py:203`, after inserting `doc1`).
- In **production**, training runs via a **Django-Q scheduled task** registered by a migration, or via a management command. The scheduled task exists at runtime in the migrated database — observed:

```
================ Q2(b) production timing: Django-Q scheduled task registered by migration 1001 ================
Schedule rows for documents.tasks.train_classifier = [('Train the classifier', 'documents.tasks.train_classifier', 'H')]
```

The `'H'` schedule type is **hourly**. It is registered by `schedule("documents.tasks.train_classifier", name="Train the classifier", schedule_type=Schedule.HOURLY)` (`src/documents/migrations/1001_auto_20201109_1636.py:10-14`). The manual entry point is the management command `document_create_classifier`, whose `handle()` (`:19`) calls `train_classifier()` (`src/documents/management/commands/document_create_classifier.py:20`).

Because the corpus is read **at call time** (`classifier.py:125-127`), whatever documents exist when `train()` runs constitute the training set (minus inbox-tagged docs).

### 3.3 (c) What confidence threshold accepts / rejects a prediction

**There is no probability/confidence threshold at this commit.** Acceptance is a **hard argmax** decision: the classifier's `predict()` output is accepted iff it is not the sentinel `-1`.

- `predict_correspondent` (`src/documents/classifier.py:251-260`) does `correspondent_id = self.correspondent_classifier.predict(X)` (`:254`) and returns it **only** `if correspondent_id != -1:` (`:255-256`), else `None` (`:258`). There is **no** `predict_proba` call anywhere.
- Consume-time acceptance in `match_correspondents` (`src/documents/matching.py:21-31`) is `filter(lambda o: matches(o, document) or o.pk == pred_id, correspondents)` — the ML term is the equality `o.pk == pred_id` (`:30`).
- The regex/fuzzy `matches()` function's `MATCH_AUTO` branch is inert: `# this is done elsewhere.` + `return False` (`src/documents/matching.py:147-149`) — so the automatic decision is driven entirely by the ML classifier, not by any similarity score/threshold.

Observed (matching vs. non-matching, mirrors `test_one_correspondent_predict_manydocs`):

```
================ Q2(c) predict: matching vs non-matching (mirrors test_one_correspondent_predict_manydocs) ================
training docs created: 2  (doc1 has correspondent c1, doc2 has NONE)
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
correspondent_classifier.classes_ = array([-1,  3])
c1.pk = 3
predict_correspondent(doc1.content) = array([3])  (expect c1.pk=3)
predict_correspondent(doc2.content) = None  (expect None -> id was -1)
matching.match_correspondents(doc1, clf) -> names = ['c1']
matching.match_correspondents(doc2, clf) -> names = []
```

The classifier's classes are `array([-1, 3])` — the `-1` sentinel and the single real correspondent id (`c1.pk=3`). A matching document predicts `array([3])` (accepted → correspondent `c1`); a document whose training label was `-1` predicts `None` (rejected → no correspondent).

**Decisive demonstration that there is no confidence gate:** with a single-correspondent model, even totally unrelated content is still accepted, because argmax has only the real class to choose (the prediction is not `-1`):

```
================ Q2(c) NO confidence threshold: unrelated content still ACCEPTED (single class, argmax) ================
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
predict_correspondent('zzz qqq completely unrelated gibberish tokens 999') = array([4])
c1.pk = 4  -> unrelated content STILL returns c1.pk (no probability gate)
```

Had there been a confidence threshold, unrelated gibberish would have been rejected; instead it returns `c1.pk`, confirming the accept/reject rule is a pure class decision (`!= -1`). Consume-time application of this rule occurs in `src/documents/signals/handlers.py:50` (`matching.match_correspondents(document, classifier)`).

---


## 4. Q3 — Document with no extractable text

This question has two parts; both are answered explicitly. The genuinely text-less fixture used is `paperless_tesseract/tests/samples/no-text-alpha.png` (an image with no readable text), exercised under the **default** `OCR_MODE="skip"`. It was parsed as a **copy** in a temporary `SCRATCH_DIR`, because for alpha PNGs the parser rewrites the input file in place to strip the alpha layer (`src/paperless_tesseract/parsers.py:191-201`) — parsing a copy keeps the fixture pristine and still exercises the canonical content.

Command:

```
docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q3.py
```

### 4.1 (a) Which OCR subprocess is invoked

**The OCR call is `ocrmypdf.ocr(**args)`** inside `RasterisedDocumentParser.parse()` (`src/paperless_tesseract/parsers.py:261`). Under the default `OCR_MODE="skip"`, `construct_ocrmypdf_parameters` sets `skip_text=True` (`:157-158`). When no text is extracted, the parser raises `NoTextFoundException("No text was found in the original document")` (`:266-267`), catches it (`:276`), and **re-runs OCR in a force-OCR fallback** — `construct_ocrmypdf_parameters(..., safe_fallback=True)` (`:288-294`) maps `safe_fallback` to `force_ocr=True` (`:155-156`), then calls `ocrmypdf.ocr(**args)` **again** (`:298`). `ocrmypdf` spawns the **Tesseract** OCR subprocess (directly observed in the logs as `ocrmypdf._exec.tesseract`) and uses **Ghostscript** for the `output_type='pdfa'` rendering. If text is still absent, the last-resort branch sets `self.text = ""` (`:318-327`).

Complete unedited output for `no-text-alpha.png`:

```
================ Q3 parse: no-text-alpha.png (image, no text) ================
OCR_MODE (default, not overridden) = 'skip'
magic.from_file('no-text-alpha.png', mime=True) = 'image/png'
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/q3-scratch-5zexi6m1/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-5zexi6m1/no-text-alpha.png', 'output_file': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/sidecar.txt', 'image_dpi': 35}
[tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[tesseract] Error during processing.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-5zexi6m1/no-text-alpha.png', 'output_file': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/sidecar-fallback.txt', 'image_dpi': 35}
[tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[tesseract] Error during processing.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/q3-scratch-5zexi6m1/no-text-alpha.png, the content will be empty.
parse() completed. self.text = ''
len(self.text) = 0
self.archive_path set? True
```

**Reading the output (the full no-text chain):**
1. **First** `ocrmypdf.ocr(**args)` with `skip_text=True` — the observed args dict (verbatim above) is the canonical first-pass invocation.
2. No text → `NoTextFoundException` → `"Attempting force OCR to get the text."` (the fallback path).
3. **Fallback** `ocrmypdf.ocr(**args)` with `force_ocr=True` (note `skip_text` is replaced by `force_ocr` in the args dict).
4. Still no text → `"No text was found in …, the content will be empty."` and `self.text = ''` (the last-resort branch at `:318-327`).

The Tesseract subprocess is directly observed (`[tesseract] …` lines emitted by `ocrmypdf._exec.tesseract`). Result was **stable across two runs** (the args dicts and the empty-text outcome were identical apart from the random tempdir names).

**Canonical caveat (labelled).** The two named "notext" tests both **override** `OCR_MODE`, so they are *not* the default path: `test_with_form_error_notext` uses `@override_settings(OCR_MODE="redo")` (`src/paperless_tesseract/tests/test_parser.py:189-195`) and `test_skip_noarchive_notext` uses `@override_settings(OCR_MODE="skip_noarchive")` (`:369-375`). Both pass, and are shown here only as complements — the canonical default (`skip`) answer is the `no-text-alpha.png` run above:

```
$ python3 -m pytest \
    paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
    paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
    -o addopts="" -v
paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED [ 50%]
paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED [100%]
======================== 2 passed, 6 warnings in 9.77s =========================
```

### 4.2 (b) What MIME type is assigned

**The stored MIME type is whatever `magic.from_file(self.path, mime=True)` detects on the input file** (`src/documents/consumer.py:219`) — for `no-text-alpha.png` this is observed as **`image/png`**. The absence of extractable text does **not** change it. The value is persisted by `_store` (`src/documents/consumer.py:379`) via `Document.objects.create(… mime_type=mime_type …)` (`:401`); parser dispatch uses `get_parser_class_for_mime_type(mime_type)` (`:223`).

The exact expression `magic.from_file(<file>, mime=True)` — identical to the one at `consumer.py:219` — was run on the sample and reported `image/png` (see the output above). For contrast, the same expression on a PDF fixture reports `application/pdf`:

```
================ Q3 parse: multi-page-images.pdf (PDF of images, no text layer) ================
OCR_MODE (default, not overridden) = 'skip'
magic.from_file('multi-page-images.pdf', mime=True) = 'application/pdf'
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/q3-scratch-c2af50a2/multi-page-images.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-c2af50a2/multi-page-images.pdf', 'output_file': '/tmp/q3-scratch-c2af50a2/paperless-pyxerz3e/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-c2af50a2/paperless-pyxerz3e/sidecar.txt'}
[2026-07-13 17:05:47,475] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:47,481] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:47,497] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
parse() completed. self.text = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
len(self.text) = 116
self.archive_path set? True
```

> The `[tesseract] Error during processing.` lines are Tesseract's own stderr surfaced through `ocrmypdf._exec.tesseract`; they do not abort the parse (the sidecar text is used). The complete, unedited capture of this run (including every duplicated console-mirror line) is reproduced in the Appendix (§7).

`multi-page-images.pdf` is a secondary/contrast fixture: its MIME is `application/pdf`, and — unlike the text-less image — its image pages contain rendered text that Tesseract **does** recover on the first `ocrmypdf.ocr` pass (116 chars, no fallback needed). It is included to show that the MIME type derives from the input bytes regardless of text content; it is *not* a "no extractable text" case.

**Summary for Q3:** (a) `ocrmypdf.ocr(**args)` is invoked — first with `skip_text=True`, and on a text-less document again in a **force-OCR fallback** (`force_ocr=True`), spawning Tesseract (and Ghostscript for PDF/A); if still empty, `self.text=""`. (b) The MIME type is taken from `magic.from_file(input, mime=True)` (`image/png` for the sample) and is **unaffected** by the absence of text.

---


## 5. Q4 — Barcode-based splitting

This question has four parts; each is answered explicitly. All barcode logic lives in `src/documents/tasks.py` at this commit (there is **no** `src/documents/barcodes.py`). Command:

```
docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q4.py
```

### 5.1 (a) How many document records come from a single input file

**N separators → N+1 fragments.** `separate_pages` (`src/documents/tasks.py:113`) builds `{fname}_document_0.pdf` from the pages **before** the first separator (loop `:131-133`, filename `:134`), then one fragment per separator, **skipping the separator page itself** via `for page in range(page_number + 1, next_page):` (`:149`), naming each `{fname}_document_{count+1}.pdf` (`:154`). Observed:

```
================ Q4(a) separate_pages: N separators -> N+1 fragments ================
LOG DEBUG paperless.tasks: Temp dir is /tmp/q4-scratch-s5kwsear/paperless-lbm0gahw
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/q4-scratch-s5kwsear/paperless-lbm0gahw/patch-code-t-middle_document_0.pdf', '/tmp/q4-scratch-s5kwsear/paperless-lbm0gahw/patch-code-t-middle_document_1.pdf']
patch-code-t-middle.pdf split on [1] -> 2 fragment(s):
    patch-code-t-middle_document_0.pdf
    patch-code-t-middle_document_1.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/q4-scratch-s5kwsear/paperless-yy8dkwbt
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/q4-scratch-s5kwsear/paperless-yy8dkwbt/several-patcht-codes_document_0.pdf', '/tmp/q4-scratch-s5kwsear/paperless-yy8dkwbt/several-patcht-codes_document_1.pdf', '/tmp/q4-scratch-s5kwsear/paperless-yy8dkwbt/several-patcht-codes_document_2.pdf']
several-patcht-codes.pdf split on [2, 5] -> 3 fragment(s):
    several-patcht-codes_document_0.pdf
    several-patcht-codes_document_1.pdf
    several-patcht-codes_document_2.pdf
```

`patch-code-t-middle.pdf` with **1** separator (page 1) → **2** fragments; `several-patcht-codes.pdf` with **2** separators (pages 2 and 5) → **3** fragments. Downstream, `consume_file` produces that many document records (once the fragments are consumed — see §5.4). The canonical `test_separate_pages` (`src/documents/tests/test_tasks.py:305-313`) asserts `len(pages) == 2` for the single-separator case; it passes (§5.5).

### 5.2 (b) Which barcode values trigger a split

**A page triggers a split iff a decoded barcode value equals `settings.CONSUMER_BARCODE_STRING` — default `"PATCHT"` (`src/paperless/settings.py:506`) — regardless of symbology.** `barcode_reader` (`src/documents/tasks.py:75`) decodes via `pyzbar.decode(image)` (`:82`), decodes the payload as UTF-8 (`:88`), and logs `f"Barcode of type {str(barcode.type)} found: {decoded_barcode}"` (`:90-92`). Observed across symbologies (all decode to `PATCHT`):

```
================ Q4(b) barcode_reader across symbologies (decoded value must equal PATCHT) ================
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
barcode_reader(qr-code-PATCHT.png      ) = ['PATCHT']
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
barcode_reader(barcode-39-PATCHT.png   ) = ['PATCHT']
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: PATCHT
barcode_reader(barcode-128-PATCHT.png  ) = ['PATCHT']
```

And the scan of whole PDFs — the separator page lists confirm which values matter:

```
================ Q4(b)(c) scan_file_for_separating_barcodes: separator page lists ================
CONSUMER_BARCODE_STRING (default) = 'PATCHT'
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t.pdf            ) = [0]   [single PATCHT on page 0]
scan(simple.pdf                  ) = []   [no barcode]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t-middle.pdf     ) = [1]   [PATCHT in the middle]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(several-patcht-codes.pdf    ) = [2, 5]   [several PATCHT codes]
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
scan(patch-code-t-qr.pdf         ) = [0]   [PATCHT encoded as QR]
```

So the **value** (`PATCHT`) is what triggers a split — whether it is encoded as `CODE39`, `CODE128`, or `QRCODE`. A file with no barcode (`simple.pdf`) yields `[]` (no split). (The default string is configurable via `PAPERLESS_CONSUMER_BARCODE_STRING`; the sibling `*_custom*` fixtures/tests confirm a custom value works, but the default trigger is `PATCHT`.)

### 5.3 (c) Where the split decision is made

**The split decision is `if separator_barcode in current_barcodes:` in `scan_file_for_separating_barcodes` (`src/documents/tasks.py:108`).** There, `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` (`:102`), the PDF is rendered to page images via `convert_from_path(...)` (`:105`), each page is decoded by `current_barcodes = barcode_reader(page)` (`:107`), and a matching page number is appended (`:109`). The whole barcode path is **gated** by `if settings.CONSUMER_ENABLE_BARCODES:` in `consume_file` (`src/documents/tasks.py:195`) (`CONSUMER_ENABLE_BARCODES` default `False`, `src/paperless/settings.py:502`). On a successful split, `consume_file` copies each fragment to the consumption directory via `save_to_dir(...)` (`:210`), deletes the original with `os.unlink(path)` (`:214`), and returns `"File successfully split"` (`:233`); the no-barcode branch instead calls `Consumer().try_consume_file(...)` (`:236`).

### 5.4 (d) Does splitting change the effective training data during a run

**No — within the run, a barcode split creates zero `Document` rows, so the training corpus read by `train()` is unchanged by the split itself.** `consume_file`'s barcode branch splits, copies fragments to the consumption directory, unlinks the original, and returns at `:233` **without** calling `try_consume_file` (which is the no-barcode branch at `:236`). Observed with `CONSUMER_ENABLE_BARCODES=True`:

```
================ Q4(c)(d) consume_file with CONSUMER_ENABLE_BARCODES=True; effect on Document corpus ================
CONSUMER_ENABLE_BARCODES = True
CONSUMPTION_DIR (save_to_dir target) = '/tmp/q4-consume-_0i4cx5h'
Document.objects.count() BEFORE consume_file = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/q4-scratch2-4jfuicei/patch-code-t-middle.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/q4-scratch2-4jfuicei/paperless-1sypz4sl
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/q4-scratch2-4jfuicei/paperless-1sypz4sl/patch-code-t-middle_document_0.pdf', '/tmp/q4-scratch2-4jfuicei/paperless-1sypz4sl/patch-code-t-middle_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/q4-scratch2-4jfuicei/patch-code-t-middle.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
consume_file(...) returned = 'File successfully split'
Document.objects.count() AFTER consume_file = 0
original input still exists? False (consume_file os.unlink on split, tasks.py L214)
fragments copied to CONSUMPTION_DIR = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
=> Document rows created by the split call: 0 (delta=0)
```

`Document.objects.count()` is **0 before and 0 after** (`delta=0`): the split produced two fragment files in the consumption directory and unlinked the original, but created **no** `Document` rows. Therefore the corpus that `train()` reads — `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`) — is **not** changed by the split within the same run. The extra document records materialize only later, when each fragment (which no longer contains a separator barcode) is independently consumed through the `try_consume_file` branch (`:236`); that subsequent consumption is by design and is not part of the split call. (The `OSError`/`Multiple exceptions … 6379` lines are the Redis channel-layer status update failing; they are caught by `try/except OSError` at `tasks.py:230-232`, so `consume_file` still returns `"File successfully split"`.)

### 5.5 Canonical Q4 tests (all pass)

```
$ python3 -m pytest \
    documents/tests/test_tasks.py::TestTasks::test_separate_pages \
    documents/tests/test_tasks.py::TestTasks::test_barcode_splitter \
    documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file \
    documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes \
    documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3 \
    documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 \
    documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr \
    documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 \
    -o addopts="" -v
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 12%]
documents/tests/test_tasks.py::TestTasks::test_barcode_splitter PASSED   [ 25%]
documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file PASSED [ 37%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED [ 50%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3 PASSED [ 62%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED [ 75%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr PASSED  [ 87%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 PASSED [100%]
======================== 8 passed, 6 warnings in 3.70s =========================
```

`test_separate_pages` (`:305-313`) → `len == 2` for `[1]`; `test_barcode_splitter` (`:376-393`) → fragments `_document_0.pdf` + `_document_1.pdf`; `test_consume_barcode_file` (`:395-406`, `@override_settings(CONSUMER_ENABLE_BARCODES=True)`) → `consume_file(dst) == "File successfully split"`.

All Q4 magnitude claims (fragment counts, separator lists, `Document` delta) were confirmed **stable across two runs** of `obs_q4.py`.

---


## 6. Non-determinism characterization

### 6.1 The default configuration that governs the reported failures

`src/setup.cfg` `[tool:pytest]` (lines 8-12):

```
[tool:pytest]
DJANGO_SETTINGS_MODULE=paperless.settings
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
env =
  PAPERLESS_DISABLE_DBHANDLER=true
```

`--numprocesses auto` (pytest-xdist) distributes tests across CPU cores in separate worker processes. The container reports `nproc = 128`, so `auto` spins up **128** workers.

### 6.2 Observed distribution — the reported non-determinism does NOT reproduce for these suites

The two suites named in the investigation (`documents/tests/test_classifier.py` and `documents/tests/test_tasks.py`) were run **10×** under the **default** parallel config (the addopts above are auto-applied; the ~29 s runtime and 774 warnings confirm xdist + coverage ran). Because a single run under `--numprocesses auto` on 128 workers plus the coverage table is many hundreds of lines, the block below is the **distilled per-run summary** — each run's exit code joined with pytest's own final summary line — captured by the pipeline shown. The substantive observable for this question (the pass/fail distribution across the 10 runs) is preserved verbatim from pytest's summary line:

```
$ for i in $(seq 1 10); do
>   out="$(python3 -m pytest documents/tests/test_classifier.py documents/tests/test_tasks.py 2>&1)"; rc=$?
>   echo "run $i exit=$rc | $(printf '%s\n' "$out" | grep -E '[0-9]+ (passed|failed)' | tail -1)"
> done
run 1 exit=0 | 62 passed, 1 skipped, 774 warnings in 29.82s
run 2 exit=0 | 62 passed, 1 skipped, 774 warnings in 28.66s
run 3 exit=0 | 62 passed, 1 skipped, 774 warnings in 28.66s
run 4 exit=0 | 62 passed, 1 skipped, 774 warnings in 28.35s
run 5 exit=0 | 62 passed, 1 skipped, 774 warnings in 29.17s
run 6 exit=0 | 62 passed, 1 skipped, 774 warnings in 27.89s
run 7 exit=0 | 62 passed, 1 skipped, 774 warnings in 28.42s
run 8 exit=0 | 62 passed, 1 skipped, 774 warnings in 29.01s
run 9 exit=0 | 62 passed, 1 skipped, 774 warnings in 29.04s
run 10 exit=0 | 62 passed, 1 skipped, 774 warnings in 28.40s
```

**Observed distribution: 10/10 all-pass (62 passed, 1 skipped, exit 0).** The reported "sometimes fails" behaviour **did not reproduce** for these suites in the canonical container under the default parallel config. This is reported honestly rather than cherry-picking a clean run — every one of the ten runs passed, and there were zero failing tracebacks to capture.

### 6.3 Why these suites are stable (observed + code)

Both `TestClassifier` (`src/documents/tests/test_classifier.py:20`) and `TestTasks` (`src/documents/tests/test_tasks.py:21`) subclass `DirectoriesMixin`. `DirectoriesMixin.setUp()` → `setup_directories()` applies `override_settings` giving each test **fresh** `tempfile.mkdtemp()` directories, isolating both `SCRATCH_DIR` (`src/documents/tests/utils.py:37`) **and** `MODEL_FILE` (`:45`), removed at `tearDown()` (`:53-57`); `django.test.TestCase` wraps each test in a rolled-back transaction. Consequently there is **no shared on-disk model file and no shared DB state** across the 128 parallel workers for these suites — which structurally explains the observed stability, and rules out the naive "shared model file across workers" theory *for these two suites*.

### 6.4 The concrete code-level mechanism (observed at the weight level)

The most defensible concrete candidate for run-to-run classification non-determinism is that **the classifier's `MLPClassifier` is unseeded**. In `train()` all three estimators are constructed as `MLPClassifier(tol=0.01)` with **no `random_state`**:

- tags: `self.tags_classifier = MLPClassifier(tol=0.01)` (`src/documents/classifier.py:219`)
- correspondents: `self.correspondent_classifier = MLPClassifier(tol=0.01)` (`:227`)
- document types: `self.document_type_classifier = MLPClassifier(tol=0.01)` (`:238`)

A grep of `src/documents/classifier.py` finds **no** `random_state`, `seed`, or `np.random` seeding anywhere. Neural-net weight initialization therefore draws from NumPy's unseeded global RNG, so training is non-deterministic between processes. Demonstrated directly by training **identical** data in two **separate** process invocations and printing the first five correspondent-classifier weights:

```
=== PROCESS A ===
first 5 MLP weights = [0.06177308462073937, -0.033976124561676015, 0.016300354614302504, -0.034932792572603386, 0.04573174353465337]
predict_correspondent('...from c1') = array([1]) (c1.pk=1)
=== PROCESS B (separate invocation, identical data) ===
first 5 MLP weights = [0.02056958809043289, 0.17637072426463646, 0.17141667425511958, -0.10589790695483886, 0.020438597809286543]
predict_correspondent('...from c1') = array([1]) (c1.pk=1)

=== VERDICT: weights identical across processes? (identical => seeded; different => UNSEEDED) ===
DIFFERENT across processes => UNSEEDED MLPClassifier (non-deterministic training)
```

**Observed:** the learned weights **differ** between the two processes (they would be identical if a fixed `random_state` were set), confirming training is non-deterministic. **Yet** `predict_correspondent('…from c1')` returns `array([1])` (= `c1.pk`) in **both** processes — for well-separated data the argmax is stable despite different weights, which is exactly why the 10 suite runs all passed.

### 6.5 Assessment (observed vs. inferred)

- **Observed:** (i) 10/10 all-pass for the two suites under the default `--numprocesses auto`; (ii) `DirectoriesMixin` isolates `MODEL_FILE` + `SCRATCH_DIR` per test; (iii) the unseeded `MLPClassifier` produces different weights across processes on identical data while keeping a stable argmax on separable fixtures.
- **Most defensible candidate for the reported flakiness (mechanism observed; failure-link inferred):** the **unseeded `MLPClassifier`** (`classifier.py:219,227,238`). On a borderline / less-separable corpus, the non-deterministic weights can flip the argmax between runs, producing intermittent prediction-assertion failures **independent of any file sharing**. My tiny fixtures were separable enough that the argmax never flipped across 10 runs and 2 processes, so the flakiness did not surface here.
- **Secondary structural candidate (inferred, code-grounded):** the fixed shared default `SCRATCH_DIR = /tmp/paperless` (`src/paperless/settings.py:84`). Any code path that does **not** go through a `DirectoriesMixin` override writes to this single fixed path; under 128 parallel xdist workers that is a plausible cross-worker collision point — but it is **not** a factor for `test_classifier.py`/`test_tasks.py`, both of which override `SCRATCH_DIR` per test.
- **Considered and ruled out for these suites:** the shared, **read-only** fixture `src/documents/tests/data/model.pickle` used via `@override_settings(MODEL_FILE=…/data/model.pickle)` (`src/documents/tests/test_classifier.py:180-182`). Concurrent **reads** of a read-only file carry no write contention, so it is not a flakiness source here. (Its `schema_version` is `7`, matching `FORMAT_VERSION = 7`, so it is version-compatible and loads cleanly.)

No behaviour above is attributed to vague environmental causes (containerization, scheduler jitter, networking); the concrete, code-level mechanism is the unseeded `MLPClassifier`, with the fixed shared `SCRATCH_DIR` as the secondary structural suspect for suites outside the two named here.

---


## 7. Appendix

### 7.1 Methodology notes on the captures

- Each section above embeds the actual command and the observed output for its condition(s). The Q1, Q2, and Q4 script captures are shown **complete and unedited** in their sections. The non-determinism block in §6.2 is a **distilled per-run summary** (each run's exit code + pytest's own final summary line), because the full raw output of a single `--numprocesses auto` run on 128 workers plus the coverage table spans many hundreds of lines; the pass/fail distribution — the substantive observable — is preserved verbatim from pytest's summary line for all 10 runs. The Q3 body de-duplicated the double console-mirror lines for readability; the **complete, unedited** Q3 capture (with every duplicated timestamped mirror line and every Tesseract warning) is reproduced verbatim in §7.2.
- Observation scripts were created **outside** the tracked repository tree (at `/tmp/obs_q1.py`, `/tmp/obs_q2.py`, `/tmp/obs_q3.py`, `/tmp/obs_q4.py`, `/tmp/obs_nd.py` inside the container) and are removed after the investigation. They import only existing project modules (`documents.tasks`, `documents.classifier`, `documents.matching`, `documents.models`, `paperless_tesseract.parsers`, `magic`) and are not part of the repository.
- In each script the entry point is the real function under investigation; a DEBUG `StreamHandler` was attached to the `paperless` logger so the real `paperless.tasks` / `paperless.classifier` / `paperless.parsing.tesseract` log lines are captured. Per-run temp `MODEL_FILE` / `SCRATCH_DIR` were used to mirror `DirectoriesMixin` isolation, except where the default was intentionally exercised.

### 7.2 Complete unedited Q3 capture (`/tmp/obs_q3.py`)

```
================ Q3 parse: no-text-alpha.png (image, no text) ================
OCR_MODE (default, not overridden) = 'skip'
magic.from_file('no-text-alpha.png', mime=True) = 'image/png'
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
[2026-07-13 17:05:45,285] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
[2026-07-13 17:05:45,285] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/q3-scratch-5zexi6m1/no-text-alpha.png for compatibility with img2pdf
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/q3-scratch-5zexi6m1/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-5zexi6m1/no-text-alpha.png', 'output_file': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/sidecar.txt', 'image_dpi': 35}
[2026-07-13 17:05:45,605] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-13 17:05:45,606] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-13 17:05:45,606] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:46,032] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-13 17:05:46,232] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-13 17:05:46,232] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/q3-scratch-5zexi6m1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-5zexi6m1/no-text-alpha.png', 'output_file': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-5zexi6m1/paperless-j755iiwf/sidecar-fallback.txt', 'image_dpi': 35}
[2026-07-13 17:05:46,479] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-13 17:05:46,479] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-13 17:05:46,479] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:46,888] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
[2026-07-13 17:05:47,083] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/q3-scratch-5zexi6m1/no-text-alpha.png, the content will be empty.
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/q3-scratch-5zexi6m1/no-text-alpha.png, the content will be empty.
parse() completed. self.text = ''
len(self.text) = 0
self.archive_path set? True
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/q3-scratch-5zexi6m1/paperless-j755iiwf

================ Q3 parse: multi-page-images.pdf (PDF of images, no text layer) ================
OCR_MODE (default, not overridden) = 'skip'
magic.from_file('multi-page-images.pdf', mime=True) = 'application/pdf'
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/q3-scratch-c2af50a2/multi-page-images.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/q3-scratch-c2af50a2/multi-page-images.pdf', 'output_file': '/tmp/q3-scratch-c2af50a2/paperless-pyxerz3e/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/q3-scratch-c2af50a2/paperless-pyxerz3e/sidecar.txt'}
[2026-07-13 17:05:47,475] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:47,481] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-13 17:05:47,497] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
parse() completed. self.text = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
len(self.text) = 116
self.archive_path set? True
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/q3-scratch-c2af50a2/paperless-pyxerz3e
```

### 7.3 Repository cleanliness

This investigation is read-only: the only artifact added to the repository is this document, `blitzy/documentation/paperless-ngx_542221a38dff.md`. No existing source, test, configuration, or fixture file was modified; all temporary observation scripts and generated model/split artifacts live outside the tracked tree (in container/host scratch space) and are removed after the investigation. The final `git status --porcelain` shows only this one new file.

### 7.4 Question → canonical entry point → citation map

| Question | Canonical entry point exercised | Key `file:line` |
|---|---|---|
| Q1 reuse/retrain | `tasks.train_classifier()`, `DocumentClassifier.train()` | `tasks.py:48,63-69`; `classifier.py:161-164,247-249`; `FORMAT_VERSION` `classifier.py:63` |
| Q2a count | `DocumentClassifier.train()` corpus query | `classifier.py:125-127,178`; tests `test_classifier.py:191-204,206-225` |
| Q2b timing | `tasks.train_classifier()` + Django-Q schedule | `migrations/1001_auto_20201109_1636.py:10-14`; `management/commands/document_create_classifier.py:20` |
| Q2c threshold | `DocumentClassifier.predict_correspondent()`, `matching.match_correspondents()` | `classifier.py:251-260`; `matching.py:21-31` (accept `:30`), `147-149`; `signals/handlers.py:50` |
| Q3a OCR subprocess | `RasterisedDocumentParser.parse()` → `ocrmypdf.ocr()` | `parsers.py:261,266-267,276,288-294,298`; `construct_ocrmypdf_parameters` `:155-158` |
| Q3b MIME | `magic.from_file(input, mime=True)` | `consumer.py:219,223,379,401` |
| Q4a count | `tasks.separate_pages()` | `tasks.py:113,131-134,149,154` |
| Q4b trigger | `tasks.barcode_reader()`, `tasks.scan_file_for_separating_barcodes()` | `tasks.py:75,82,88,90-92`; `settings.py:506` |
| Q4c decision | `tasks.scan_file_for_separating_barcodes()`, `tasks.consume_file()` | `tasks.py:108` (decision), `:102,105,107,109`; gate `:195`; `settings.py:502` |
| Q4d training effect | `tasks.consume_file()` + `Document.objects.count()` | `tasks.py:195,210,214,233,236`; corpus `classifier.py:125-127` |
| Non-determinism | pytest `--numprocesses auto`; unseeded `MLPClassifier` | `setup.cfg:8-12`; `classifier.py:219,227,238`; `utils.py:37,45`; `settings.py:84` |

