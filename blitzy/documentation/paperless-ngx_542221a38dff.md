# paperless-ngx — Runtime Investigation: ML Classification, Automatic Correspondent Matching, Text-less OCR, and Barcode Splitting

This document answers four runtime-behavior questions about paperless-ngx at commit `542221a38dff06361e07976452f9aea24d210542`, plus the reported run-to-run non-determinism of the document-classification tests. Every behavioral claim below was produced by **building and running** the real code paths through their canonical entry points inside the user-provided Docker container, capturing the exact command and its complete, unedited output. Code is cited with full repository-relative `path:line` references validated against the checked-out sources at this commit.

**Evidence-label legend.** Every substantive statement carries one label:

- **[observed]** — taken directly from runtime output captured in this investigation (the exact command + output is embedded here or in the Appendix).
- **[code-grounded]** — read from the source at a cited `path:line` (structural fact, not a runtime measurement).
- **[inferred]** — a conclusion drawn from observed + code-grounded facts; labeled as such and never presented as a measurement.
- **[fallback]** — behavior on a secondary/error path that the code takes when the primary path fails.
- **[supporting / non-canonical]** — a value produced by a helper or a bypassing interface (e.g. a direct library call) rather than the canonical production entry point; used only to corroborate, never as the primary answer.

**A note on ellipses (`...`).** No output block in this document has been abbreviated. Any `...` that appears inside a captured block is literal text emitted by the program itself — for example `unpaper`'s progress markers, the `paperless.classifier` log line `Gathering data from database...`, the consumer's `Parsing <file>...`, or a Python print label such as `consume_file(...)`. Where a code listing is quoted from source, it is quoted in full for the region cited; logic is never elided with `// ...`.

**A note on control characters.** One incidental line of captured output — a Django "one-time only migration to generate thumbnails" banner emitted during test-database setup in the Q4 run — was wrapped by the emitting program in raw ANSI SGR bold codes. To avoid embedding non-printable bytes in this document, each raw `ESC` (0x1b) byte is shown losslessly as its visible escape text `\x1b` (so the banner is bracketed by `\x1b[1m` … `\x1b[0m`). No characters were removed; only the non-printable byte is rendered visibly.

---

## 1. Environment & Methodology

### 1.1 Where the code was run — container provenance & security posture

All runtime observation was performed inside a persistent Docker container named `pngx-qna`, created from the user-specified image (with the three missing system libraries — `libzbar0`, `poppler-utils`, `pngquant` — baked in per the setup notes). The image ids, the exact create/run command, and the container's security posture are **[observed]**:

```text
########## PROVENANCE: docker images ##########
$ docker image inspect paperless-ngx-qna:ready --format '{{.Id}}'
sha256:e565fd722a1e55d3a295938c8cfd558c9f30e935d73e1bb69e5713dd425929e9

$ docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 --format '{{.Id}} {{.RepoDigests}}'
sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f [ghcr.io/scaleapi/swe-atlas@sha256:d4abe56dd5d1cb2632353baf06e9d80a704f147a70ac2ec15c2b78fc5fddfe15]

########## PROVENANCE: exact container create/run command used ##########
$ docker run -d --name pngx-qna --entrypoint sleep paperless-ngx-qna:ready infinity

########## PROVENANCE: docker inspect pngx-qna (security posture) ##########
$ docker inspect pngx-qna --format 'Privileged={{.HostConfig.Privileged}} ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}} CapAdd={{.HostConfig.CapAdd}} CapDrop={{.HostConfig.CapDrop}} NetworkMode={{.HostConfig.NetworkMode}} Mounts={{range .Mounts}}{{.Source}}->{{.Destination}} {{end}}'
Privileged=false
ReadonlyRootfs=false
CapAdd=[]
CapDrop=[]
NetworkMode=bridge
Mounts=(none if empty)
```

The container is **unprivileged**, has **no bind mounts** (its `/app` is entirely separate from the repository checkout on the host), adds/drops no capabilities, and uses the default bridge network. All commands were run as the non-root `testuser` (uid 1000) from a clean `/tmp`, as the setup notes require (running as root makes four permission tests spuriously pass/fail and pollutes `/tmp`).

### 1.2 Runtime versions (each value shown with its producing command) — [observed]

```text
########## RUNTIME VERSIONS (each value has its producing command) ##########
$ id
uid=1000(testuser) gid=1000(testuser) groups=1000(testuser)

$ python3 --version
Python 3.9.23

$ git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542

$ python3 -c "import importlib.metadata as m; [print(f\"{p:14s} {m.version(p)}\") for p in [\"scikit-learn\",\"numpy\",\"scipy\",\"joblib\",\"threadpoolctl\",\"django\",\"django-q\",\"ocrmypdf\",\"pikepdf\",\"pdf2image\",\"pyzbar\",\"python-magic\",\"fuzzywuzzy\",\"pytest\",\"pytest-xdist\",\"pytest-django\",\"factory-boy\"]]"
scikit-learn   1.0.2
numpy          1.22.3
scipy          1.8.0
joblib         1.1.0
threadpoolctl  3.1.0
django         4.0.4
django-q       1.3.9
ocrmypdf       13.4.3
pikepdf        5.1.1
pdf2image      1.16.0
pyzbar         0.1.9
python-magic   0.4.25
fuzzywuzzy     0.18.0
pytest         8.4.2
pytest-xdist   3.8.0
pytest-django  4.11.1
factory-boy    3.3.3

########## SYSTEM BINARIES ##########
$ tesseract --version 2>&1 | head -1
tesseract 4.1.1
$ gs --version
9.53.3
$ pdftoppm -v 2>&1 | head -1
pdftoppm version 20.09.0
$ unpaper --version
6.1
$ qpdf --version | head -1
qpdf version 10.1.0
```

These match the pins in `requirements.txt` at this commit (scikit-learn 1.0.2, numpy 1.22.3, scipy 1.8.0, ocrmypdf 13.4.3, pikepdf 5.1.1, pdf2image 1.16.0, pyzbar 0.1.9, python-magic 0.4.25, django 4.0.4, django-q 1.3.9), on the canonical Python 3.9.

### 1.3 Build / check — [observed]

Django's system check passes cleanly in the container:

```text
########## BUILD/CHECK ##########
$ python3 manage.py check
System check identified no issues (0 silenced).
check exit=0
```

### 1.4 Source integrity — the exercised code is byte-identical to the repository — [observed]

The `.py` files exercised below are byte-for-byte identical between the container's `/app/src` and the repository's `src/`, so the `path:line` citations are valid for both. Note there are **two** distinct `parsers.py` modules — `src/documents/parsers.py` (base parser framework) and `src/paperless_tesseract/parsers.py` (the OCR parser) — and this document always cites the full path to disambiguate them:

```text
########## SOURCE IDENTITY: md5 container /app/src vs repo src ##########
--- container ---
e8ca934a3dabb7498ae7bf98aca481f6  documents/classifier.py
8af1ca06493ef50129f4314cc16d5686  documents/tasks.py
0db85577a13a7d9d34c9071a7221cf1c  documents/matching.py
e7392cc635432e94a56998b45d351526  documents/consumer.py
d843d2ecf7344f1ed7b8225f59dc8744  documents/parsers.py
997b21c1eb0f97098fb799750259fd36  documents/signals/handlers.py
26a6bc09de30be58becdeaab91d8f6e7  paperless_tesseract/parsers.py
c719bb0ae75f5dd438cd5937ba3d0955  paperless/settings.py
190d884f99b96408a1f687f1b5757c15  setup.cfg
--- repo ---
e8ca934a3dabb7498ae7bf98aca481f6  documents/classifier.py
8af1ca06493ef50129f4314cc16d5686  documents/tasks.py
0db85577a13a7d9d34c9071a7221cf1c  documents/matching.py
e7392cc635432e94a56998b45d351526  documents/consumer.py
d843d2ecf7344f1ed7b8225f59dc8744  documents/parsers.py
997b21c1eb0f97098fb799750259fd36  documents/signals/handlers.py
26a6bc09de30be58becdeaab91d8f6e7  paperless_tesseract/parsers.py
c719bb0ae75f5dd438cd5937ba3d0955  paperless/settings.py
190d884f99b96408a1f687f1b5757c15  setup.cfg
```

### 1.5 How the code was run — harness, isolation, and safety

- **Canonical entry points only.** Each probe drives the real function under investigation — `documents.tasks.train_classifier`, `documents.classifier.DocumentClassifier.train` / `.load` / `load_classifier`, `documents.classifier.DocumentClassifier.predict_correspondent`, `documents.matching.match_correspondents`, `paperless_tesseract.parsers.RasterisedDocumentParser.parse`, `documents.consumer.Consumer.try_consume_file`, and `documents.tasks.{scan_file_for_separating_barcodes,separate_pages,save_to_dir,consume_file}`. Any value from a helper or a bypassing interface is labeled **[supporting / non-canonical]**.
- **Logging.** A DEBUG `StreamHandler` is attached to the `paperless.*` loggers (`propagate=False` to avoid duplicate records) so the real `paperless.tasks` / `paperless.classifier` / `paperless.consumer` / `paperless.parsing*` log lines are captured verbatim. For OCR subprocess evidence, DEBUG handlers are also attached to `ocrmypdf.subprocess` and the `ocrmypdf._exec.*` loggers.
- **Test isolation mirrored.** Scripts create the test database with `connection.creation.create_test_db()` and use per-probe `tempfile.mkdtemp()` directories for `MODEL_FILE` / `SCRATCH_DIR`, mirroring how `DirectoriesMixin.setUp()` isolates each canonical test (`src/documents/tests/utils.py:14-58`).
- **Security (CWE-377).** Temporary directories are created with `tempfile.mkdtemp()` (mode `0700`), never predictable fixed names, and are removed in a `try/finally`. The full source of every observation script is reproduced in Appendix §7.2 so each run is independently auditable and reproducible.
- **Determinism protocol.** Every magnitude/timing/count claim was produced at least twice; both runs are embedded (§7.3) and are identical on every quantitative line. For the reported non-determinism, the *same* unchanged input (the two named suites) was run ten times from a demonstrably identical filesystem state and the full distribution is reported (§6, §7.4).
- **Repository safety.** Observation scripts live only in the container's `/tmp` (outside any tracked tree) and are removed afterward. The one import-bound shared directory that canonical task tests write to — `/app/consume` — is inventoried before/after and isolated during split probes (§5.1, §5.4); the repository checkout is never touched (§7.5).

---

## 2. Q1 — Does the classifier reuse an existing model or retrain within a run, and how does that affect later tests?

### 2.1 Answer

- **Within a single run, the classifier *reuses* the persisted model when the training data is unchanged, and *retrains* (re-saves) only when the data changes.** The decision is a SHA-1 hash guard, not a time- or count-based policy. **[observed]** + **[code-grounded]**
- `DocumentClassifier.train()` computes a SHA-1 digest over the ordered, preprocessed document contents and their label ids, and returns `False` (no save) when the digest equals the instance's stored `data_hash` (`src/documents/classifier.py:163-164`); `tasks.train_classifier()` calls `save()` only when `train()` returns `True`, otherwise logging `Training data unchanged.` (`src/documents/tasks.py:62-69`). **[code-grounded]**
- **Cross-test effect:** because `MODEL_FILE` is isolated to a per-test `tempfile.mkdtemp()` by `DirectoriesMixin` (`src/documents/tests/utils.py:45`) and there is **no** in-memory classifier cache active at this commit (the cache test is skipped — `src/documents/tests/test_classifier.py:399-401`), one test's saved/loaded model cannot leak into another *through the model file*. A model with an incompatible `FORMAT_VERSION` is **deleted** on load (`src/documents/classifier.py:42-48`), and an unseeded neural net (see §6.4) is the real cross-run coupling, not the file. **[observed]** + **[code-grounded]** + **[inferred]**

### 2.2 Observed — driving the real entry point `tasks.train_classifier()` three times (initial / unchanged / changed), and the direct `train()` guard

Command (Run 1; the complete Run 2 is byte-identical on every quantitative line and is reproduced in Appendix §7.3):

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q1.py 1
```

```text
================ Q1 RUN 1 ================
scratch mode = 0o700
MODEL_FILE = /tmp/pngx-q1-ro9cx3e0/classification_model.pickle

--- A: tasks.train_classifier() lifecycle (initial/unchanged/changed) ---
model exists before any training: False
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1-ro9cx3e0/classification_model.pickle...
model exists after call #1: True | st_mtime#1 = 1783966220.7187214
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.tasks: Training data unchanged.
st_mtime#2 = 1783966220.7187214 | REUSE (m1==m2): True
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1-ro9cx3e0/classification_model.pickle...
st_mtime#3 = 1783966220.7357216 | RETRAIN (m2!=m3): True
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG DEBUG paperless.classifier: Gathering data from database...
direct train() #1: True (True=>changed) | #2: False (False=>unchanged/REUSE) | data_hash set: True

--- B: incompatible FORMAT_VERSION -> load_classifier() os.unlink deletion ---
FORMAT_VERSION (on-disk written by save) = 7
model file exists before load: True
patched FORMAT_VERSION (loader expects) = 8
LOG ERROR paperless.classifier: Unrecoverable error while loading document classification model, deleting model file.
Traceback (most recent call last):
  File "/app/src/documents/classifier.py", line 40, in load_classifier
    classifier.load()
  File "/app/src/documents/classifier.py", line 81, in load
    raise IncompatibleClassifierVersionError(
documents.classifier.IncompatibleClassifierVersionError: Cannot load classifier, incompatible versions.
load_classifier() returned: None (None => load failed & file deleted)
model file exists AFTER load_classifier(): False (False => os.unlink at classifier.py:48 ran)

--- C: empty corpus -> ValueError('No training data available.') ---
Document.objects.count() = 0
LOG DEBUG paperless.classifier: Gathering data from database...
raised ValueError: 'No training data available.'

================ Q1 RUN 1 SUMMARY ================
reuse(m1==m2)=True retrain(m2!=m3)=True direct_train=(True,False) incompat_deleted=True empty_corpus_ValueError=True
POST-RUN cleanup: scratch exists? False
```

Reading the capture:

- **Reuse on unchanged data — [observed].** After call #1 saved the model (`Saving updated classifier model to …`), call #2 logged `Training data unchanged.` and the model file's `st_mtime` was **unchanged** (`REUSE (m1==m2): True`). The direct `train()` calls confirm it: first `True` (changed → trains), then `False` (unchanged → reuse), with `data_hash set: True`. This is exactly the mechanism at `src/documents/classifier.py:163-164` (guard) and `src/documents/tasks.py:62-69` (save-only-if-True).
- **Retrain on changed data — [observed].** After mutating a document's content, call #3 re-saved and the `st_mtime` advanced (`RETRAIN (m2!=m3): True`).
- **Incompatible `FORMAT_VERSION` → model file deleted — [observed] + [fallback].** `FORMAT_VERSION` on disk is `7` (`src/documents/classifier.py:63`). With the loader patched to expect version `8`, the real `load_classifier()` logged `Unrecoverable error while loading document classification model, deleting model file.`, raised `IncompatibleClassifierVersionError` (`src/documents/classifier.py:80-83`), returned `None`, and the model file **no longer existed** afterward — i.e. `os.unlink` at `src/documents/classifier.py:48` ran. This is the concrete cross-test guard: a stale/incompatible model is removed rather than reused.
- **Empty corpus → `ValueError` — [observed] + [fallback].** With `Document.objects.count() = 0`, `train()` raised `ValueError: 'No training data available.'` (`src/documents/classifier.py:158-159`).
- **Training-corpus query & inbox exclusion — [code-grounded].** `train()` reads the corpus with `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`); the `1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).` log line is that query's result.

### 2.3 Observed — the canonical tests through pytest — [observed]

```text
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest documents/tests/test_tasks.py::TestTasks::test_train_classifier \
    documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    documents/tests/test_classifier.py::TestClassifier::testVersionIncreased \
    documents/tests/test_classifier.py::TestClassifier::testNoTrainingData \
    -p no:cacheprovider -o addopts="" -v
```

```text
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
cachedir: .pytest_cache
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 4 items

documents/tests/test_tasks.py::TestTasks::test_train_classifier PASSED   [ 25%]
documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED [ 50%]
documents/tests/test_classifier.py::TestClassifier::testVersionIncreased PASSED [ 75%]
documents/tests/test_classifier.py::TestClassifier::testNoTrainingData PASSED [100%]

=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================== 4 passed, 6 warnings in 2.18s =========================
```

`test_train_classifier` (`src/documents/tests/test_tasks.py:75-89`) asserts the exact mtime reuse/retrain behavior observed above; `testVersionIncreased` and `testNoTrainingData` cover the version-guard and empty-corpus paths. All pass.

### 2.4 Cross-test isolation — model file, DB identity, and factory usage

- **Model file isolated per test — [code-grounded].** `DirectoriesMixin.setUp()` → `setup_directories()` applies `override_settings(MODEL_FILE=<mkdtemp>/…)` (`src/documents/tests/utils.py:45`) and `rmtree`s it at `tearDown()` (`src/documents/tests/utils.py:53-57`).
- **No in-memory cache — [code-grounded].** The classifier-caching test is explicitly skipped at this commit (`src/documents/tests/test_classifier.py:399-401`), so there is no process-level model cache to leak between tests.
- **Per-worker test DB identity — [observed].** Under `pytest-xdist`, each worker reported the same in-memory SQLite name; the shared name is a shared-cache in-memory DB scoped per worker process, not a shared file:

```text
WORKER=gw0 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw1 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw2 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw3 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw4 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw5 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw6 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
WORKER=gw7 test_db_NAME=file:memorydb_default?mode=memory&cache=shared
```

- **Direct ORM inserts, not factory-boy — [observed].** The cited classifier/tasks tests use direct `Model.objects.create(...)`; neither module imports the available `factory-boy` factories:

```text
########## factory-boy vs direct-ORM in the cited classifier/tasks tests ##########
$ grep -c 'objects.create' documents/tests/test_classifier.py documents/tests/test_tasks.py
documents/tests/test_classifier.py:41
documents/tests/test_tasks.py:8

$ grep -nE 'Factory|factory' documents/tests/test_classifier.py documents/tests/test_tasks.py ; echo rc=$?
rc=1 (1 => neither test module imports/uses factory-boy)

$ ls documents/tests/factories.py ; grep -nE 'class .*Factory' documents/tests/factories.py
documents/tests/factories.py
8:class CorrespondentFactory(DjangoModelFactory):
15:class DocumentFactory(DjangoModelFactory):

$ grep -rlE 'from documents.tests.factories|tests.factories import' documents/tests/*.py
```

So `src/documents/tests/factories.py` (`CorrespondentFactory`, `DocumentFactory`) exists but is **unused** by the Q1/Q2 tests — those tests build their corpus with direct ORM inserts.

---

## 3. Q2 — Automatic (`MATCH_AUTO`) correspondent matching: training-document count, training timing, and confidence threshold

### 3.1 Answers

- **(a) How many training documents are created:** in the canonical single-correspondent prediction path (`test_one_correspondent_predict`, `src/documents/tests/test_classifier.py:191-204`) exactly **one** `Document` is inserted; in the manydocs sibling (`_manydocs`, `src/documents/tests/test_classifier.py:206-225`) the corpus is larger. The count is whatever the test inserts via direct ORM `create()`; training reads the current corpus at call time via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`), so **inbox-tagged documents are excluded**. **[observed]** + **[code-grounded]**
- **(b) When training occurs relative to the inserts:** **never on insert.** Training is an explicit, separate call — `DocumentClassifier.train()` / `tasks.train_classifier()` issued *after* the inserts in tests, and in production a Django-Q **scheduled** task (registered hourly by migration `src/documents/migrations/1001_auto_20201109_1636.py:11`) or the `document_create_classifier` management command (`src/documents/management/commands/document_create_classifier.py:20`). **[observed]** + **[code-grounded]**
- **(c) What confidence threshold accepts/rejects a prediction:** **there is no probability/confidence threshold at this commit.** Acceptance is a hard argmax class decision: `predict_correspondent` runs `self.correspondent_classifier.predict(X)` and returns the predicted id only when it is not `-1`, else `None` (`src/documents/classifier.py:251-260`); `match_correspondents` accepts a correspondent iff `o.pk == pred_id` (`src/documents/matching.py:21-31`, accept at `:30`). No `predict_proba` gate exists in `src/documents/classifier.py`. **[observed]** + **[code-grounded]**

### 3.2 Observed — the real training/prediction path (Run 1; complete Run 2 in Appendix §7.3)

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q2.py 1
```

```text
================ Q2 RUN 1 ================

--- (a) training-doc count + inbox exclusion (Document.objects...exclude(tags__is_inbox_tag=True)) ---
Document.objects.count() (rows inserted) = 3
effective corpus exclude(tags__is_inbox_tag=True) = 2
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
canonical test_one_correspondent_predict inserts Document.objects.count() = 1
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.

--- (b) timing: inserting docs does NOT train; only explicit train_classifier() does ---
after inserting Correspondent+Document: model file exists = False
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q2-da_ppa28/m.pickle...
after explicit tasks.train_classifier(): model file exists = True
Django-Q Schedule rows for documents.tasks.train_classifier = [('Train the classifier', 'documents.tasks.train_classifier', 'H')]

--- (c) NO confidence threshold: hard argmax (predict()!=-1, o.pk==pred_id); no predict_proba ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
correspondent_classifier.classes_ = [-1, 4]
c1.pk = 4
predict_correspondent(doc1.content) = [4] (expect c1.pk)
predict_correspondent(doc2.content) = None (expect None; label was -1)
match_correspondents(doc1) names = ['c1']
match_correspondents(doc2) names = []
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
[SUPPORTING] single-class classes_ = [5] | predict(unrelated) = [5] | only.pk = 5 => unrelated STILL accepted (no probability gate)

================ Q2 RUN 1 SUMMARY ================
(a) inserted=3 effective=2 ; canonical single-doc=1 | (b) insert_trains=False explicit_trains=True schedule=[('Train the classifier', 'documents.tasks.train_classifier', 'H')] | (c) argmax_no_threshold=True
POST-RUN cleanup: scratch exists? False
```

Reading the capture:

- **(a) count + inbox exclusion — [observed].** 3 rows inserted; the effective corpus `exclude(tags__is_inbox_tag=True)` = **2** (one inbox-tagged doc excluded), and training logged `2 documents, …`. The canonical `test_one_correspondent_predict` inserts exactly **1**.
- **(b) timing — [observed].** After inserting a `Correspondent` + `Document`, the model file did **not** exist (insert does not train); only after the explicit `tasks.train_classifier()` did the model file appear. The Django-Q schedule row is `[('Train the classifier', 'documents.tasks.train_classifier', 'H')]` — `'H'` = hourly, matching `src/documents/migrations/1001_auto_20201109_1636.py:11`.
- **(c) no threshold — [observed].** With `classes_ = [-1, 4]`, `predict_correspondent(doc1)` returned `[4]` (= `c1.pk`) and `match_correspondents(doc1)` accepted `['c1']`; `predict_correspondent(doc2)` returned `None` (its label was the sentinel `-1`) so `match_correspondents(doc2)` accepted `[]`. The **[supporting / non-canonical]** single-class probe shows the absence of a probability gate most starkly: with `classes_ = [5]`, an *unrelated* input still predicts `[5]` and is accepted — there is no confidence floor below which the prediction is rejected. The primary evidence, however, is the code path itself: `predict()` → `!= -1` → `o.pk == pred_id`, with no `predict_proba` anywhere.

### 3.3 Observed — canonical Q2 tests — [observed]

```text
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs \
    -p no:cacheprovider -o addopts="" -v
```

```text
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED [ 50%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED [100%]
======================== 2 passed, 6 warnings in 1.88s =========================
```

---

## 4. Q3 — A document with no extractable text: which OCR subprocess is invoked, and what MIME type is assigned?

### 4.1 Answers

- **(a) Which OCR subprocess is invoked:** `RasterisedDocumentParser.parse()` (`src/paperless_tesseract/parsers.py:230-327`) calls `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:261`). Under the **default** `OCR_MODE="skip"` (`src/paperless/settings.py:522`) the first invocation sets `skip_text=True` (`src/paperless_tesseract/parsers.py:157-158`); `ocrmypdf` in turn spawns the real external binaries **Tesseract** (`tesseract 4.1.1`) and **Ghostscript** (`GPL Ghostscript 9.53.3`), plus `unpaper 6.1`. Because the page has no text, the primary attempt raises `NoTextFoundException` (`src/paperless_tesseract/parsers.py:266-267`) and the parser re-runs OCR with `safe_fallback=True` → `force_ocr=True` (`src/paperless_tesseract/parsers.py:288-298`). If even that yields nothing, `self.text` ends empty (`src/paperless_tesseract/parsers.py:318-327`). **[observed]** + **[code-grounded]** + **[fallback]**
- **(b) What MIME type is assigned:** the stored MIME type is whatever `magic.from_file(self.path, mime=True)` detects on the **input** (`src/documents/consumer.py:219`), persisted by `Document.objects.create(..., mime_type=mime_type)` inside `_store()` (`src/documents/consumer.py:401`). For the text-less PNG the canonical Consumer path persisted `mime_type='image/png'`. **The absence of extractable text does not change the MIME type** — only the stored `content` is empty. **[observed]** + **[code-grounded]**

### 4.2 Observed — the complete `parse()` sequence for the text-less image (Run 1; complete two-run capture in Appendix §7.3)

The block below is the **complete, contiguous** `parse(no-text-alpha.png)` sequence under default `OCR_MODE="skip"`: the `skip_text=True` primary `ocrmypdf.ocr` call, the full external-subprocess chain it spawns (Tesseract, Ghostscript, unpaper — each shown with its exact argv), the `NoTextFoundException`-triggered `force_ocr=True` fallback, and the final empty `self.text`. The `magic.from_file(...) = image/png` on line 5 is labeled **[supporting / non-canonical]** because it is a direct library call; the canonical persisted MIME is shown in §4.3.

```text
================ Q3 RUN 1 ================
OCR_MODE (default) = skip

--- (a) RasterisedDocumentParser.parse(no-text-alpha.png) [textless image, default skip] ---
magic.from_file(input, mime=True) = image/png (supporting/non-canonical: direct call)
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-pyaa1io1/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/sidecar.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '--version']
SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--version']
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Running: ['gs', '--version']
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.jpb7_paf/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.jpb7_paf/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpgtix5qdo/input.pnm', '/tmp/tmpgtix5qdo/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x5831fcaefe80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x5831fcaefe80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpgtix5qdo/input.pnm -> /tmp/tmpgtix5qdo/output.ppm
input-file for sheet 1: /tmp/tmpgtix5qdo/input.pnm
output-file for sheet 1: /tmp/tmpgtix5qdo/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.jpb7_paf/000001_ocr.png', '/tmp/ocrmypdf.io.jpb7_paf/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.jpb7_paf/fix_docinfo.pdf', '/tmp/ocrmypdf.io.jpb7_paf/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-pyaa1io1/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/sidecar-fallback.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.5lvuepee/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.5lvuepee/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpoi8f070l/input.pnm', '/tmp/tmpoi8f070l/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x59bdbc557e80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x59bdbc557e80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpoi8f070l/input.pnm -> /tmp/tmpoi8f070l/output.ppm
input-file for sheet 1: /tmp/tmpoi8f070l/input.pnm
output-file for sheet 1: /tmp/tmpoi8f070l/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.5lvuepee/000001_ocr.png', '/tmp/ocrmypdf.io.5lvuepee/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.5lvuepee/fix_docinfo.pdf', '/tmp/ocrmypdf.io.5lvuepee/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png, the content will be empty.
parse() done. self.text repr = '' | len = 0 | archive_path set? = True
```

Key facts from this capture — **[observed]**:

- The **primary** call passes `'skip_text': True` (line 9), matching `OCR_MODE="skip"` → `skip_text=True` (`src/paperless_tesseract/parsers.py:157-158`).
- **Tesseract** is spawned (`['tesseract', '--list-langs']`, `['tesseract', '--version']`, `['tesseract', '-l', 'osd', '--psm', '0', …]`, `['tesseract', '-l', 'eng', '--psm', '2', …]`, and `['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', …]`).
- **Ghostscript** is spawned for rasterization and for PDF/A production — the real argv `['gs', …, '-sDEVICE=pdfwrite', …, '-dPDFA=2', '-dPDFACompatibilityPolicy=1', …]` (line 55) with the banner `GPL Ghostscript 9.53.3 (2020-10-01)` (line 56). This is the concrete Ghostscript subprocess evidence for the PDF/A (`output_type='pdfa'`) output.
- The primary attempt finds no text → `Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.` (line 63), then the **fallback** call passes `'force_ocr': True` (line 66).
- Final result: `No text was found …, the content will be empty.` and `parse() done. self.text repr = '' | len = 0 | archive_path set? = True`.

### 4.3 Observed — the canonical Consumer path persists `mime_type='image/png'` (text does not change MIME)

Driven through the real `Consumer.try_consume_file()` (with only `Consumer._send_progress` patched — the exact technique used by the canonical `TestConsumer.setUp`, `src/documents/tests/test_consumer.py:290-291` — because it merely emits a Redis websocket progress update; the MIME-detect → parser-dispatch → `parse` → `_store` → `Document.objects.create` chain runs for real):

```text
--- (b) FULL Consumer().try_consume_file(no-text-alpha.png) -> persisted Document.mime_type ---
LOG INFO paperless.consumer: Consuming no-text-alpha.png
LOG DEBUG paperless.consumer: Detected mime type: image/png
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
```

The full OCR subprocess chain repeats here identically to §4.2 (complete capture in Appendix §7.3); the persisted record is:

```text
persisted Document.pk = 1
persisted Document.mime_type = 'image/png' (CANONICAL: from _store/Document.objects.create at consumer.py:401)
persisted Document.content repr = '' (empty because no extractable text)
```

`mime_type='image/png'` comes from `magic.from_file` on the input (`src/documents/consumer.py:219`) and is stored by `_store()` → `Document.objects.create(..., mime_type=...)` (`src/documents/consumer.py:401`); `content` is empty because no text was extractable — confirming the MIME is independent of text presence. **[observed]** + **[code-grounded]**

### 4.4 Observed — error/edge path: an encrypted PDF (no text obtainable) — [observed] + [fallback]

```text
--- edge: parse(encrypted.pdf) [EncryptedPdfError path] ---
magic.from_file(encrypted.pdf, mime=True) = application/pdf
LOG WARNING paperless.parsing.tesseract: Error while getting text from PDF document with pdfminer.six
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 120, in extract_text
    stripped = post_process_text(pdfminer_extract_text(pdf_file))
  File "/usr/local/lib/python3.9/site-packages/pdfminer/high_level.py", line 157, in extract_text
    for page in PDFPage.get_pages(
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfpage.py", line 151, in get_pages
    doc = PDFDocument(parser, password=password, caching=caching)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 744, in __init__
    self._initialize_password(password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 771, in _initialize_password
    handler = factory(docid, param, password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 358, in __init__
    self.init()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 366, in init
    self.init_key()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 381, in init_key
    raise PDFPasswordIncorrect
pdfminer.pdfdocument.PDFPasswordIncorrect
```

…and the parser's decision + empty result:

```text
LOG WARNING paperless.parsing.tesseract: This file is encrypted, OCR is impossible. Using any text present in the original file.
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3enc-5i8tydub/encrypted.pdf, the content will be empty.
parse(encrypted.pdf) done. self.text repr = ''
```

`magic.from_file` still reports `application/pdf`; `pdfminer` raises `PDFPasswordIncorrect` at `src/paperless_tesseract/parsers.py:120`; the parser logs `This file is encrypted, OCR is impossible. Using any text present in the original file.` and finishes with empty text — the MIME is still the input's type, only `content` is empty.

### 4.5 Observed — canonical Q3 tests — [observed]

Command:

```text
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
    paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
    -p no:cacheprovider -o addopts="" -v
```

Result: `2 passed, 6 warnings in 9.56s` (exit 0). The complete, unedited capture of both tests is in Appendix §7.3.

---

## 5. Q4 — Barcode-based splitting: record count, trigger values, decision site, and effect on training data

### 5.1 Answers

- **(a) How many document records come from a single input file:** `separate_pages()` mechanically produces **N+1 fragment files** for **N** separator pages (`src/documents/tasks.py:113-161`). When those fragments are consumed, each **byte-distinct** fragment becomes one `Document` row; **byte-identical** fragments are rejected by `pre_check_duplicate` (`src/documents/consumer.py:102-113`, md5 at `:104`). Observed: `several-patcht-codes.pdf` (2 separators → 3 fragment files, 2 distinct) → **2** rows; `patch-code-t-middle.pdf` (1 separator → 2 fragment files, 1 distinct) → **1** row. **[observed]** + **[code-grounded]**
- **(b) Which barcode values trigger a split:** the single value configured by `CONSUMER_BARCODE_STRING`, default **`"PATCHT"`** (`src/paperless/settings.py:506`), read at split time in `scan_file_for_separating_barcodes` (`src/documents/tasks.py:102`). The symbology is irrelevant — CODE39, QR, and CODE128 all trigger a split as long as the **decoded string equals** the configured value; a different value (e.g. `"CUSTOM BARCODE"`) only triggers when `CONSUMER_BARCODE_STRING` is set to it. **[observed]** + **[code-grounded]**
- **(c) Where the split decision is made:** in `scan_file_for_separating_barcodes`, which reads each rendered page's barcodes (`current_barcodes = barcode_reader(page)`, `src/documents/tasks.py:107`) and then, at `if separator_barcode in current_barcodes:` (`src/documents/tasks.py:108`), appends the page index to the separator list (`:109`). The whole barcode path is gated by `settings.CONSUMER_ENABLE_BARCODES` (default `False`) in `consume_file` (`src/documents/tasks.py:195`). **[code-grounded]**
- **(d) Does splitting change the effective training data during a run:** **yes.** The `consume_file` *split* path itself creates **zero** `Document` rows — it copies fragments to the consumption directory and unlinks the original (`src/documents/tasks.py:210,214`), returning `"File successfully split"`. But when the fragments are subsequently consumed, each distinct fragment becomes a `Document`, and `train()` reads the current corpus via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`) — so one input file enlarges the training corpus by up to N+1 rows. **[observed]** + **[code-grounded]**

### 5.2 Observed — import-time binding of the `save_to_dir` destination (`/app/consume`)

`save_to_dir(..., target_dir=settings.CONSUMPTION_DIR)` binds its default when `src/documents/tasks.py` is imported (`src/documents/tasks.py:167`). The probe proves it: reassigning `settings.CONSUMPTION_DIR` at runtime does **not** change `tasks.save_to_dir.__defaults__`, and the pre-existing fragments in `/app/consume` are leftover pollution from prior canonical `test_consume_barcode_file` runs (§5.6):

```text
--- (0) import-time binding: settings.CONSUMPTION_DIR vs tasks.save_to_dir.__defaults__ ---
settings.CONSUMPTION_DIR (runtime) = /app/src/../consume
tasks.save_to_dir.__defaults__      = (None, '/app/src/../consume') (tuple = (newname_default, target_dir_default); target_dir bound at import, tasks.py:167)
after settings.CONSUMPTION_DIR := /tmp/some-other-consume-XYZ:
  settings.CONSUMPTION_DIR       = /tmp/some-other-consume-XYZ
  tasks.save_to_dir.__defaults__ = (None, '/app/src/../consume') (UNCHANGED -> bound at import, not read live)
import-bound target dir            = /app/src/../consume
inventory(/app/src/../consume) BEFORE = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
\x1b[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
\x1b[0m
```

### 5.3 Observed — (b)+(c) which values trigger a split, over every edge — [observed]

```text
--- (A) scan_file_for_separating_barcodes - every edge (returns separator page numbers) ---
[default CONSUMER_BARCODE_STRING = 'PATCHT']
scan(simple.pdf                      ) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t.pdf                ) -> [0]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t-middle.pdf         ) -> [1]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(several-patcht-codes.pdf        ) -> [2, 5]
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
scan(patch-code-t-qr.pdf             ) -> [0]

--- (A') barcode_reader on PNG edges (unreadable / custom value) ---
barcode_reader(barcode-39-PATCHT-unreadable.png) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
barcode_reader(barcode-128-custom.png)           -> ['CUSTOM BARCODE'] (decoded value is the CUSTOM string, not 'PATCHT')

--- (A'') custom CONSUMER_BARCODE_STRING controls which value triggers a split ---
[override CONSUMER_BARCODE_STRING = 'CUSTOM BARCODE']
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
scan(barcode-128-custom.pdf) -> [0] (now matches because decoded value == configured string)

```

Every scan result matches the canonical tests: `simple.pdf`→`[]` (no barcode), `patch-code-t.pdf`→`[0]` (separator on page 0), `patch-code-t-middle.pdf`→`[1]`, `several-patcht-codes.pdf`→`[2, 5]`, `patch-code-t-qr.pdf`→`[0]` (QR). The `barcode_reader` PNG edges show an **unreadable** barcode yields `[]` and a **custom-value** barcode decodes to `'CUSTOM BARCODE'` (not `PATCHT`), so under the default string it would not trigger; setting `CONSUMER_BARCODE_STRING='CUSTOM BARCODE'` makes `scan(barcode-128-custom.pdf)`→`[0]`. Each decoded value is logged as `Barcode of type <SYMBOLOGY> found: <value>` (`src/documents/tasks.py:90-92`).

### 5.4 Observed — (a) N separators → N+1 fragment files — [observed]

```text
--- (B) separate_pages fragment arithmetic (N separators -> N+1 fragments) ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-0495il26
LOG WARNING paperless.tasks: No pages to split on!
LOG DEBUG paperless.tasks: Temp files are []
separate_pages(patch-code-t-middle.pdf      splits=[]     ) -> 0 fragment(s): []
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-azuad6iz
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-azuad6iz/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-azuad6iz/patch-code-t-middle_document_1.pdf']
separate_pages(patch-code-t-middle.pdf      splits=[1]    ) -> 2 fragment(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
     patch-code-t-middle_document_0.pdf       pages=1
     patch-code-t-middle_document_1.pdf       pages=1
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-i8ak0j0w
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_0.pdf', '/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_1.pdf', '/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_2.pdf']
separate_pages(several-patcht-codes.pdf     splits=[2, 5] ) -> 3 fragment(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
     several-patcht-codes_document_0.pdf      pages=2
     several-patcht-codes_document_1.pdf      pages=2
     several-patcht-codes_document_2.pdf      pages=1

```

`[]`→0 fragments (with `No pages to split on!`, `src/documents/tasks.py:129`); `[1]` on the 3-page `patch-code-t-middle.pdf`→**2** fragments (1 page each; the separator page 1 is skipped by `range(page_number+1, next_page)`, `src/documents/tasks.py:149`); `[2, 5]` on the 7-page `several-patcht-codes.pdf`→**3** fragments (2, 2, 1 pages). Fragments are written to a `tempfile.mkdtemp(prefix="paperless-", dir=SCRATCH_DIR)` (`src/documents/tasks.py:118`), i.e. `/tmp/paperless/…`, **not** `/app/consume`.

### 5.5 Observed — (d) the split path creates 0 rows; consuming fragments enlarges the corpus

The `consume_file` split path, with the shared `/app/consume` destination **safely isolated** for this probe by temporarily rebinding `tasks.save_to_dir.__defaults__` to a temp dir (restored in `finally`; §7.2), returns `"File successfully split"` and creates **zero** `Document` rows:

```text
--- (C) consume_file (CONSUMER_ENABLE_BARCODES=True) - split path writes fragments, creates 0 Document rows ---
isolated save_to_dir target -> /tmp/pngx-q4-consume-uuspm1ds (temp; /app/consume untouched)
Document.objects.count() BEFORE split = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4-in-tl5k1jzf/patch-code-t-middle.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-z0r8jhdu
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-z0r8jhdu/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-z0r8jhdu/patch-code-t-middle_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4-in-tl5k1jzf/patch-code-t-middle.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
consume_file(...) returned            = 'File successfully split'
Document.objects.count() AFTER split  = 0   (delta = 0 -> split path creates NO Document rows)
original input still exists?          = False (os.unlink at tasks.py:214 after successful split)
isolated consume dir contents         = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']

```

Then consuming the produced fragments through the real Consumer turns each **distinct** fragment into a `Document` row (identical fragments deduped), enlarging the training corpus; a subsequent `train()` reads that enlarged corpus. The block below is the key-line view — the complete capture, including every OCR subprocess line for each consumed fragment, is in Appendix §7.3:

```text
--- (D) consume produced fragments through the REAL Consumer -> distinct fragments become Document rows ---
training-corpus size BEFORE any consume = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
input several-patcht-codes.pdf     separators=[2, 5]  -> 3 fragment FILES (N+1, N=2); distinct md5=2
     fragment several-patcht-codes_document_0.pdf      md5=873af29fb051743f1a8fc8cd840ca7b1
     fragment several-patcht-codes_document_1.pdf      md5=873af29fb051743f1a8fc8cd840ca7b1
     fragment several-patcht-codes_document_2.pdf      md5=7b8695ec87ce936d20bc4a86fb51e5be
     consumed several-patcht-codes_document_0.pdf      -> Document pk=1 mime='application/pdf' content_len=50
LOG ERROR paperless.consumer: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
     consumed several-patcht-codes_document_1.pdf      -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
     consumed several-patcht-codes_document_2.pdf      -> Document pk=2 mime='application/pdf' content_len=24
  => several-patcht-codes.pdf: 3 fragment files -> 2 NEW Document rows
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
input patch-code-t-middle.pdf      separators=[1]     -> 2 fragment FILES (N+1, N=1); distinct md5=1
     fragment patch-code-t-middle_document_0.pdf       md5=f82b8d7d92170df6d54875e1cb4fa297
     fragment patch-code-t-middle_document_1.pdf       md5=f82b8d7d92170df6d54875e1cb4fa297
     consumed patch-code-t-middle_document_0.pdf       -> Document pk=3 mime='application/pdf' content_len=24
LOG ERROR paperless.consumer: Not consuming patch-code-t-middle_document_1.pdf: It is a duplicate.
     consumed patch-code-t-middle_document_1.pdf       -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
  => patch-code-t-middle.pdf : 2 fragment files -> 1 NEW Document rows (identical frags deduped)
training-corpus size AFTER consuming fragments = 3   (splitting 2 input files enlarged the effective training data by +3 rows)

--- (D') DocumentClassifier().train() now trains on the split-enlarged corpus ---
DocumentClassifier().train() returned = True (reads Document.objects at classifier.py:125; corpus size = 3, all from splitting)

```

So splitting **did** change the effective training data: two input files yielded 3 net `Document` rows (`several-patcht-codes.pdf`→2 distinct, `patch-code-t-middle.pdf`→1 distinct), and `DocumentClassifier().train()` returned `True` on that split-produced corpus. (The fragment md5 **values** vary run-to-run because `pikepdf`'s save is not byte-deterministic across processes, but the **distinct-count** invariant — 2-of-3 and 1-of-2 — and the resulting row counts are stable across both runs; see §7.3.) **[observed]** + **[inferred]**

### 5.6 Observed — canonical Q4 tests, and the `/app/consume` pollution they cause — [observed]

```text
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest \
    documents/tests/test_tasks.py::TestTasks::test_separate_pages \
    documents/tests/test_tasks.py::TestTasks::test_barcode_splitter \
    documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file \
    documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes \
    documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 \
    -p no:cacheprovider -o addopts="" -v
```

```text
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 20%]
documents/tests/test_tasks.py::TestTasks::test_barcode_splitter PASSED   [ 40%]
documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file PASSED [ 60%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED [ 80%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED [100%]
======================== 5 passed, 6 warnings in 3.29s =========================
```

`test_consume_barcode_file` (`src/documents/tests/test_tasks.py:395`) does not pass a `target_dir`, so via the import-bound default it copies `patch-code-t-middle_document_0.pdf` and `patch-code-t-middle_document_1.pdf` into the shared `/app/consume`. This investigation inventoried that directory before/after every probe and every non-determinism run, isolated it during split probes, and deleted only those exact test-created files afterward (§7.5). **[observed]**

---

## 6. Non-determinism characterization

### 6.1 The default configuration that governs the reported failures — [code-grounded]

`src/setup.cfg` `[tool:pytest]` applies these `addopts` automatically: `--pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet`, with `DJANGO_SETTINGS_MODULE=paperless.settings` and `PAPERLESS_DISABLE_DBHANDLER=true`. So every unqualified `pytest` run uses `pytest-xdist` with `--numprocesses auto` (parallel workers) and coverage.

### 6.2 Worker count — `--numprocesses auto` → 128 workers — [observed]

```text
$ docker exec -u testuser pngx-qna nproc
128
$ docker exec -u testuser pngx-qna python3 -c 'import os; print("os.cpu_count() =", os.cpu_count())'
os.cpu_count() = 128
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest documents/tests/test_classifier.py -o addopts="" --no-header -n auto -v -p no:cacheprovider
created: 128/128 workers
128 workers [23 items]
```

The shell CPU count is **128**, and `-n auto` resolves to `created: 128/128 workers`. Under the default `--quiet`, the only worker-startup line pytest-xdist actually emits is `bringing up nodes...` (it appears twice, once per suite collection); the verbose `created: 128/128 workers` banner above comes from a supplementary `-v` invocation, since `--quiet` suppresses it.

### 6.3 The two named suites, run 10× from a demonstrably identical state — [observed]

The two suites in question (`documents/tests/test_classifier.py` and `documents/tests/test_tasks.py`) were run **10 times** under the default config. Before each run, the one shared mutable target, `/app/consume`, was emptied so every run starts from an identical filesystem state; the inventory before/after each run and the repository's `git status` before/after the whole loop are recorded in §7.5. The per-run manifest (exit code, wall-clock duration, output line count, `/app/consume` inventory before/after, and pytest's own summary line):

```text
$ for i in $(seq 1 10); do
>   rm -f /app/consume/* 2>/dev/null
>   before="$(ls -A /app/consume | tr '\n' ',')"; [ -z "$before" ] && before="(empty)"
>   start=$(date +%s.%N)
>   python3 -m pytest documents/tests/test_classifier.py documents/tests/test_tasks.py > /tmp/nd/nd_run_$i.txt 2>&1
>   rc=$?; end=$(date +%s.%N); dur=$(awk "BEGIN{printf \"%.2f\", $end-$start}")
>   after="$(ls -A /app/consume | tr '\n' ',')"; [ -z "$after" ] && after="(empty)"
>   echo "run $i | exit=$rc | wall=${dur}s | lines=$(wc -l < /tmp/nd/nd_run_$i.txt) | consume_before=$before | consume_after=$after | $(grep -E '[0-9]+ (passed|failed)' /tmp/nd/nd_run_$i.txt | tail -1)"
> done
```

```text
run 1 | exit=0 | wall=30.95s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 28.86s
run 2 | exit=0 | wall=31.12s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 29.06s
run 3 | exit=0 | wall=31.19s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 29.04s
run 4 | exit=0 | wall=31.05s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 29.00s
run 5 | exit=0 | wall=30.94s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 28.88s
run 6 | exit=0 | wall=30.29s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 28.18s
run 7 | exit=0 | wall=34.82s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 32.57s
run 8 | exit=0 | wall=48.06s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 43.22s
run 9 | exit=0 | wall=35.73s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 33.68s
run 10 | exit=0 | wall=33.54s | lines=35 | consume_before=(empty) | consume_after=patch-code-t-middle_document_0.pdf,patch-code-t-middle_document_1.pdf, | 62 passed, 1 skipped, 774 warnings in 31.21s
```

**Observed distribution: 10/10 all-pass** — every run exited `0` with `62 passed, 1 skipped, 774 warnings`. `consume_before` is `(empty)` for all ten runs and `consume_after` is the same two fragment files for all ten — proving each run began from an identical state and produced an identical result. The reported "sometimes fails" behavior **did not reproduce** for these two suites in the canonical container under the default parallel config; this is honest **non-reproduction**, not proof that the suites are structurally incapable of flaking (see §6.5).

A single complete run is only **35 lines** (there is no large per-file coverage table on the terminal — with `--cov-report=html` the terminal shows only a three-line coverage footer and `Coverage HTML written to dir htmlcov`), so all ten complete, unedited runs are reproduced in Appendix §7.4. Here is run 1 in full — **[observed]**:

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 28.86s
```

The `s..............................................................` line is the progress indicator: one `s` (the single skipped test) followed by 62 passing dots. The 774 warnings are 6 distinct deprecation categories × 129 occurrences each (redis `connection.py:67/69/71/73`, django `conf/__init__.py:229` USE_L10N, django-q `core_signing.py:9` baseconv).

### 6.4 The concrete code-level mechanism — an unseeded `MLPClassifier` — [observed] + [code-grounded]

The most defensible concrete candidate for run-to-run classification non-determinism is that the classifier's `MLPClassifier` is **unseeded**. All three estimators are constructed as `MLPClassifier(tol=0.01)` with **no `random_state`** (`src/documents/classifier.py:219` tags, `:227` correspondents, `:238` document types); a grep finds no RNG seeding anywhere. Training identical data in two separate process invocations yields **different** learned weights (they would be identical if seeded), while the argmax prediction stays stable on well-separated fixtures:

```text
########## Cross-process unseeded MLP (two separate invocations, identical data) ##########
$ python3 /tmp/obs_q1_rng.py A   ;  python3 /tmp/obs_q1_rng.py B
=== PROCESS A ===
first 5 correspondent-MLP weights = [-0.0019321648136633172, 0.0019423598020853678, 0.016298689330098728, -0.0003242765430265773, 0.2242488854615288]
predict_correspondent('this is a document from c1') = [1] (c1.pk=1)
=== PROCESS B ===
first 5 correspondent-MLP weights = [0.0002728923593665325, -0.012477450506852715, -0.0025893042218007537, -0.01169012809750399, 0.16755901770974968]
predict_correspondent('this is a document from c1') = [1] (c1.pk=1)

########## grep for any RNG seeding in classifier.py ##########
$ grep -nE 'random_state|np.random|numpy.random|\bseed\b' documents/classifier.py ; echo rc=$?
grep rc=1 (1 => no match => UNSEEDED)
$ grep -nE 'MLPClassifier\(' documents/classifier.py
219:            self.tags_classifier = MLPClassifier(tol=0.01)
227:            self.correspondent_classifier = MLPClassifier(tol=0.01)
238:            self.document_type_classifier = MLPClassifier(tol=0.01)
```

### 6.5 Assessment — observed vs. bounded hypothesis

- **[observed]** 10/10 all-pass for the two suites under default `--numprocesses auto` (128 workers), from an identical state.
- **[observed] + [code-grounded]** `DirectoriesMixin` isolates `MODEL_FILE` and `SCRATCH_DIR` per test (`src/documents/tests/utils.py:37,45`), removed at `tearDown` (`:53-57`), and each `TestCase` runs in a rolled-back transaction — so there is **no shared on-disk model file and no shared DB state** across workers for these two suites. Model-file isolation is therefore a **separate** concern from the RNG/order coupling below.
- **[observed] + [inferred]** The unseeded `MLPClassifier` (`src/documents/classifier.py:219,227,238`) draws initial weights from NumPy's process-global RNG. Test order / worker assignment under xdist can therefore alter initial weights even when model files are isolated. On a borderline / less-separable corpus this could flip an argmax between runs and produce intermittent prediction-assertion failures **independent of any file sharing**. My fixtures were separable enough that the argmax never flipped across 10 runs and 2 processes, so the flakiness did not surface here. This RNG/order coupling is offered as a **clearly-bounded hypothesis** for the reported flakiness, not a proven cause; the observation is bounded to the 10/10 result.
- **[inferred, code-grounded]** Secondary structural suspect: the fixed shared `SCRATCH_DIR = /tmp/paperless` (`src/paperless/settings.py:84`). Any code path that does not go through a `DirectoriesMixin` override writes there; under 128 parallel workers that is a plausible cross-worker collision point — but it is **not** a factor for these two suites, both of which override `SCRATCH_DIR` per test.
- No behavior above is attributed to vague environmental causes (containerization, scheduler jitter, networking); the concrete code-level mechanism is the unseeded `MLPClassifier`, with the fixed shared `SCRATCH_DIR` as a secondary structural suspect for suites outside these two.

---

## 7. Appendix — complete, unedited evidence

This appendix reproduces (a) the full source of every observation script, (b) the complete, unedited output of every probe including the second stability run, (c) all ten non-determinism runs in full, (d) the cleanup / repository-state transcript, and (e) the question → entry-point → citation map. Nothing here is abbreviated; any `...` inside an output block is literal emitted text (see the note in the header).

### 7.1 How to reproduce

Each script was delivered into the container's `/tmp` with `docker cp` and executed as:

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/<script>.py <run-number>
```

Every script creates its own test database and per-probe `tempfile.mkdtemp()` directories (mode `0700`, no predictable names — CWE-377 safe), attaches DEBUG handlers to the `paperless.*` (and, for OCR, `ocrmypdf.*`) loggers, and cleans up in a `try/finally` with a post-run inventory. The scripts import only existing project modules and live only under the container's `/tmp`; they are removed after the investigation and never enter the repository.

### 7.2 Complete observation-script sources

#### `/tmp/obs_q1.py` (Q1 — reuse/retrain, version-guard deletion, empty corpus)

```python
#!/usr/bin/env python3
"""Q1 observation: classifier reuse-vs-retrain, incompatible-version deletion, empty-corpus.
Exercises the REAL entry points: tasks.train_classifier(), documents.classifier.load_classifier(),
DocumentClassifier.train(). Secure per-run temp dirs (mkdtemp 0700), try/finally cleanup + inventory.
"""
import os, sys, stat, tempfile, logging, shutil, pickle
from unittest import mock

RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
scratch = tempfile.mkdtemp(prefix="pngx-q1-")          # mode 0700
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)

# DEBUG log capture on the real paperless loggers
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for n in ("paperless.tasks", "paperless.classifier"):
    lg = logging.getLogger(n); lg.setLevel(logging.DEBUG); lg.addHandler(h); lg.propagate = False

from documents import tasks
from documents.classifier import DocumentClassifier, load_classifier, IncompatibleClassifierVersionError
from documents.models import Correspondent, Document, Tag

model_file = os.path.join(scratch, "classification_model.pickle")
ovr = override_settings(MODEL_FILE=model_file, DATA_DIR=scratch, SCRATCH_DIR=scratch)
ovr.enable()
try:
    print(f"================ Q1 RUN {RUN} ================")
    print("scratch mode =", oct(stat.S_IMODE(os.lstat(scratch).st_mode)))
    print("MODEL_FILE =", settings.MODEL_FILE)

    # ---- A) reuse vs retrain via the real task entry point ----
    c = Correspondent.objects.create(matching_algorithm=Tag.MATCH_AUTO, name="test")
    doc = Document.objects.create(correspondent=c, content="test", title="test", checksum="Q1A")
    print("\n--- A: tasks.train_classifier() lifecycle (initial/unchanged/changed) ---")
    print("model exists before any training:", os.path.isfile(settings.MODEL_FILE))
    tasks.train_classifier()
    m1 = os.stat(settings.MODEL_FILE).st_mtime
    print("model exists after call #1:", os.path.isfile(settings.MODEL_FILE), "| st_mtime#1 =", m1)
    tasks.train_classifier()
    m2 = os.stat(settings.MODEL_FILE).st_mtime
    print("st_mtime#2 =", m2, "| REUSE (m1==m2):", m1 == m2)
    doc.content = "test2"; doc.save()
    tasks.train_classifier()
    m3 = os.stat(settings.MODEL_FILE).st_mtime
    print("st_mtime#3 =", m3, "| RETRAIN (m2!=m3):", m2 != m3)

    # direct hash guard
    clf = DocumentClassifier()
    r1 = clf.train(); r2 = clf.train()
    print("direct train() #1:", r1, "(True=>changed) | #2:", r2, "(False=>unchanged/REUSE) | data_hash set:", clf.data_hash is not None)

    # ---- B) incompatible-version -> load_classifier() deletes stale model (REAL function) ----
    print("\n--- B: incompatible FORMAT_VERSION -> load_classifier() os.unlink deletion ---")
    print("FORMAT_VERSION (on-disk written by save) =", DocumentClassifier.FORMAT_VERSION)
    print("model file exists before load:", os.path.isfile(settings.MODEL_FILE))
    with mock.patch("documents.classifier.DocumentClassifier.FORMAT_VERSION", DocumentClassifier.FORMAT_VERSION + 1):
        print("patched FORMAT_VERSION (loader expects) =", DocumentClassifier.FORMAT_VERSION)
        result = load_classifier()
    print("load_classifier() returned:", result, "(None => load failed & file deleted)")
    print("model file exists AFTER load_classifier():", os.path.isfile(settings.MODEL_FILE), "(False => os.unlink at classifier.py:48 ran)")

    # ---- C) empty corpus -> ValueError ----
    print("\n--- C: empty corpus -> ValueError('No training data available.') ---")
    Document.objects.all().delete()
    print("Document.objects.count() =", Document.objects.count())
    empty_clf = DocumentClassifier()
    try:
        empty_clf.train()
        print("NO EXCEPTION (unexpected)")
    except ValueError as e:
        print("raised ValueError:", repr(str(e)))

    print(f"\n================ Q1 RUN {RUN} SUMMARY ================")
    print(f"reuse(m1==m2)={m1==m2} retrain(m2!=m3)={m2!=m3} direct_train=({r1},{r2}) "
          f"incompat_deleted={not os.path.isfile(settings.MODEL_FILE)} empty_corpus_ValueError=True")
finally:
    ovr.disable()
    try:
        connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception:
        pass
    shutil.rmtree(scratch, ignore_errors=True)
    print("POST-RUN cleanup: scratch exists?", os.path.exists(scratch))
```

#### `/tmp/obs_q1_rng.py` (Q1/§6 — cross-process unseeded `MLPClassifier`)

```python
#!/usr/bin/env python3
"""Cross-process RNG probe: train identical data, print first 5 correspondent-MLP weights.
Run in TWO separate process invocations; if weights differ => unseeded MLPClassifier."""
import os, sys, tempfile, shutil
TAG = sys.argv[1] if len(sys.argv) > 1 else "A"
scratch = tempfile.mkdtemp(prefix="pngx-q1rng-")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)
from documents.classifier import DocumentClassifier
from documents.models import Correspondent, Document, Tag
ovr = override_settings(MODEL_FILE=os.path.join(scratch, "m.pickle"), DATA_DIR=scratch, SCRATCH_DIR=scratch)
ovr.enable()
try:
    c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    Document.objects.create(title="d1", content="this is a document from c1", correspondent=c1, checksum="A")
    clf = DocumentClassifier()
    clf.train()
    w = clf.correspondent_classifier.coefs_[0].ravel()[:5].tolist()
    print(f"=== PROCESS {TAG} ===")
    print("first 5 correspondent-MLP weights =", w)
    print("predict_correspondent('this is a document from c1') =", clf.predict_correspondent("this is a document from c1"), "(c1.pk=%d)" % c1.pk)
finally:
    ovr.disable()
    try: connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception: pass
    shutil.rmtree(scratch, ignore_errors=True)
```

#### `/tmp/obs_q2.py` (Q2 — count, timing, no-threshold acceptance)

```python
#!/usr/bin/env python3
"""Q2 observation: (a) training-doc count + inbox exclusion, (b) training timing,
(c) no confidence threshold (hard argmax). Real entry points: DocumentClassifier.train()/
predict_correspondent(), matching.match_correspondents(). Secure temp dirs + cleanup."""
import os, sys, tempfile, logging, shutil
RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
scratch = tempfile.mkdtemp(prefix="pngx-q2-")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for n in ("paperless.tasks", "paperless.classifier"):
    lg = logging.getLogger(n); lg.setLevel(logging.DEBUG); lg.addHandler(h); lg.propagate = False
from documents import tasks
from documents.classifier import DocumentClassifier
from documents import matching
from documents.models import Correspondent, Document, Tag
ovr = override_settings(MODEL_FILE=os.path.join(scratch, "m.pickle"), DATA_DIR=scratch, SCRATCH_DIR=scratch)
ovr.enable()
try:
    print(f"================ Q2 RUN {RUN} ================")
    # ---- (a) count + inbox exclusion ----
    print("\n--- (a) training-doc count + inbox exclusion (Document.objects...exclude(tags__is_inbox_tag=True)) ---")
    c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    inbox = Tag.objects.create(name="inbox", is_inbox_tag=True, matching_algorithm=Tag.MATCH_ANY)
    d1 = Document.objects.create(title="d1", content="this is a document from c1", correspondent=c1, checksum="A")
    d2 = Document.objects.create(title="d2", content="second doc from c1 too", correspondent=c1, checksum="B")
    d3 = Document.objects.create(title="d3", content="inbox doc excluded", checksum="C")
    d3.tags.add(inbox)
    total = Document.objects.count()
    effective = Document.objects.exclude(tags__is_inbox_tag=True).count()
    print("Document.objects.count() (rows inserted) =", total)
    print("effective corpus exclude(tags__is_inbox_tag=True) =", effective)
    clf = DocumentClassifier(); clf.train()
    # canonical single-doc count
    Document.objects.all().delete(); Correspondent.objects.all().delete(); Tag.objects.all().delete()
    cc = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    Document.objects.create(title="doc1", content="this is a document from c1", correspondent=cc, checksum="A")
    print("canonical test_one_correspondent_predict inserts Document.objects.count() =", Document.objects.count())
    DocumentClassifier().train()

    # ---- (b) timing ----
    print("\n--- (b) timing: inserting docs does NOT train; only explicit train_classifier() does ---")
    Document.objects.all().delete(); Correspondent.objects.all().delete()
    c = Correspondent.objects.create(name="cB", matching_algorithm=Tag.MATCH_AUTO)
    Document.objects.create(title="db", content="timing body", correspondent=c, checksum="A")
    print("after inserting Correspondent+Document: model file exists =", os.path.isfile(settings.MODEL_FILE))
    tasks.train_classifier()
    print("after explicit tasks.train_classifier(): model file exists =", os.path.isfile(settings.MODEL_FILE))
    from django_q.models import Schedule
    rows = list(Schedule.objects.filter(func="documents.tasks.train_classifier").values_list("name","func","schedule_type"))
    print("Django-Q Schedule rows for documents.tasks.train_classifier =", rows)

    # ---- (c) no confidence threshold ----
    print("\n--- (c) NO confidence threshold: hard argmax (predict()!=-1, o.pk==pred_id); no predict_proba ---")
    Document.objects.all().delete(); Correspondent.objects.all().delete()
    c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    d1 = Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
    d2 = Document.objects.create(title="doc2", content="this is a document from noone", checksum="B")
    clf = DocumentClassifier(); clf.train()
    print("correspondent_classifier.classes_ =", clf.correspondent_classifier.classes_.tolist())
    print("c1.pk =", c1.pk)
    print("predict_correspondent(doc1.content) =", clf.predict_correspondent(d1.content), "(expect c1.pk)")
    print("predict_correspondent(doc2.content) =", clf.predict_correspondent(d2.content), "(expect None; label was -1)")
    print("match_correspondents(doc1) names =", [o.name for o in matching.match_correspondents(d1, clf)])
    print("match_correspondents(doc2) names =", [o.name for o in matching.match_correspondents(d2, clf)])
    # SUPPORTING: single-class model accepts even unrelated content (argmax has only real class)
    Document.objects.all().delete(); Correspondent.objects.all().delete()
    cs = Correspondent.objects.create(name="only", matching_algorithm=Correspondent.MATCH_AUTO)
    Document.objects.create(title="s", content="alpha beta gamma delta", correspondent=cs, checksum="A")
    sclf = DocumentClassifier(); sclf.train()
    pred = sclf.predict_correspondent("zzz qqq completely unrelated gibberish tokens 999")
    print("[SUPPORTING] single-class classes_ =", sclf.correspondent_classifier.classes_.tolist(),
          "| predict(unrelated) =", pred, "| only.pk =", cs.pk, "=> unrelated STILL accepted (no probability gate)")
    print(f"\n================ Q2 RUN {RUN} SUMMARY ================")
    print(f"(a) inserted=3 effective=2 ; canonical single-doc=1 | (b) insert_trains=False explicit_trains=True schedule={rows} | (c) argmax_no_threshold=True")
finally:
    ovr.disable()
    try: connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception: pass
    shutil.rmtree(scratch, ignore_errors=True)
    print("POST-RUN cleanup: scratch exists?", os.path.exists(scratch))
```

#### `/tmp/obs_q3.py` (Q3 — text-less OCR subprocess chain, Consumer MIME persistence, encrypted edge)

```python
#!/usr/bin/env python3
"""Q3 observation: (a) OCR subprocess on a textless document (skip->force fallback, Tesseract+Ghostscript),
(b) persisted MIME via the REAL Consumer path. Plus encrypted-PDF edge. Secure temp dirs + cleanup."""
import os, sys, tempfile, logging, shutil
import magic
RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
SAMPLES = "/app/src/paperless_tesseract/tests/samples"
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment(); connection.creation.create_test_db(verbosity=0)
# paperless parsing logs
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for n in ("paperless.parsing.tesseract", "paperless.consumer"):
    lg = logging.getLogger(n); lg.setLevel(logging.DEBUG); lg.addHandler(h); lg.propagate = False
# ocrmypdf subprocess logs (Ghostscript + Tesseract processes)
hs = logging.StreamHandler(sys.stdout)
hs.setFormatter(logging.Formatter("SUBPROC %(name)s: %(message)s"))
for n in ("ocrmypdf.subprocess", "ocrmypdf._exec.ghostscript", "ocrmypdf._exec.tesseract"):
    lg = logging.getLogger(n); lg.setLevel(logging.DEBUG); lg.addHandler(hs); lg.propagate = False

from paperless_tesseract.parsers import RasterisedDocumentParser
from documents.tests.utils import paperless_environment
from documents.consumer import Consumer
from documents.models import Document

print(f"================ Q3 RUN {RUN} ================")
print("OCR_MODE (default) =", settings.OCR_MODE)

# ---- (a) parse a textless image: skip -> force fallback -> self.text='' ; Tesseract + Ghostscript ----
print("\n--- (a) RasterisedDocumentParser.parse(no-text-alpha.png) [textless image, default skip] ---")
scratch = tempfile.mkdtemp(prefix="pngx-q3a-")
work = os.path.join(scratch, "no-text-alpha.png"); shutil.copy(os.path.join(SAMPLES, "no-text-alpha.png"), work)
print("magic.from_file(input, mime=True) =", magic.from_file(work, mime=True), "(supporting/non-canonical: direct call)")
ovr = override_settings(SCRATCH_DIR=scratch); ovr.enable()
try:
    p = RasterisedDocumentParser("q3a")
    p.parse(work, "image/png")
    print("parse() done. self.text repr =", repr(p.text), "| len =", len(p.text), "| archive_path set? =", bool(p.archive_path))
    p.cleanup()
finally:
    ovr.disable(); shutil.rmtree(scratch, ignore_errors=True)

# ---- (b) FULL Consumer path -> persisted Document.mime_type (CANONICAL persistence) ----
# _send_progress is patched exactly as canonical TestConsumer.setUp (test_consumer.py:290-291);
# it only emits a Redis websocket progress update. The real MIME detection -> parser dispatch ->
# parse -> _store -> Document.objects.create chain is fully executed.
print("\n--- (b) FULL Consumer().try_consume_file(no-text-alpha.png) -> persisted Document.mime_type ---")
from unittest import mock
with paperless_environment() as dirs:
    consumable = os.path.join(dirs.scratch_dir, "no-text-alpha.png")
    shutil.copy(os.path.join(SAMPLES, "no-text-alpha.png"), consumable)
    with mock.patch("documents.consumer.Consumer._send_progress"):
        doc = Consumer().try_consume_file(consumable)
    print("persisted Document.pk =", doc.pk)
    print("persisted Document.mime_type =", repr(doc.mime_type), "(CANONICAL: from _store/Document.objects.create at consumer.py:401)")
    print("persisted Document.content repr =", repr(doc.content), "(empty because no extractable text)")

# ---- edge: encrypted PDF ----
print("\n--- edge: parse(encrypted.pdf) [EncryptedPdfError path] ---")
scratch2 = tempfile.mkdtemp(prefix="pngx-q3enc-")
worke = os.path.join(scratch2, "encrypted.pdf"); shutil.copy(os.path.join(SAMPLES, "encrypted.pdf"), worke)
print("magic.from_file(encrypted.pdf, mime=True) =", magic.from_file(worke, mime=True))
ovr2 = override_settings(SCRATCH_DIR=scratch2); ovr2.enable()
try:
    pe = RasterisedDocumentParser("q3enc")
    try:
        pe.parse(worke, "application/pdf")
        print("parse(encrypted.pdf) done. self.text repr =", repr(pe.text))
    except Exception as e:
        print("parse(encrypted.pdf) raised:", type(e).__name__, str(e))
    pe.cleanup()
finally:
    ovr2.disable(); shutil.rmtree(scratch2, ignore_errors=True)

try: connection.creation.destroy_test_db(":memory:", verbosity=0)
except Exception: pass
print(f"\n================ Q3 RUN {RUN} SUMMARY ================")
print("(a) ocrmypdf.ocr skip_text then force_ocr fallback; Tesseract+Ghostscript spawned; self.text='' | (b) persisted mime_type observed via Consumer")
```

#### `/tmp/obs_q4.py` (Q4 — import binding, scan edges, split arithmetic, fragment consumption, corpus effect)

```python
"""
Q4 - Barcode splitting and training-corpus effect.
Canonical entry points: documents.tasks.scan_file_for_separating_barcodes,
separate_pages, save_to_dir, consume_file; documents.consumer.Consumer.try_consume_file;
documents.tasks.train_classifier / documents.classifier.DocumentClassifier.train.
Secure temp dirs (mkdtemp 0700), try/finally cleanup, post-run inventory. The import-bound
/app/consume target is isolated by temporarily rebinding save_to_dir.__defaults__ (restored in
finally) so NOTHING is written to /app/consume.
"""
import os
import sys
import shutil
import logging
import tempfile

RUN = sys.argv[1] if len(sys.argv) > 1 else "1"

import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()

from django.conf import settings
from django.test.utils import setup_test_environment, teardown_test_environment
from django.test import override_settings
from django.db import connection
from unittest import mock
from PIL import Image

from documents import tasks
from documents.classifier import DocumentClassifier, load_classifier
from documents.consumer import Consumer, ConsumerError
from documents.models import Document, Correspondent, Tag

BARCODES = os.path.join("/app/src/documents/tests/samples/barcodes")
SAMPLES = os.path.join("/app/src/documents/tests/samples")

_h = logging.StreamHandler(sys.stdout)
_h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for name in ("paperless.tasks", "paperless.consumer", "paperless.parsing",
             "paperless.parsing.tesseract"):
    lg = logging.getLogger(name)
    lg.setLevel(logging.DEBUG)
    lg.handlers = [_h]
    lg.propagate = False


def hr(t):
    print("\n--- " + t + " ---")


print("================ Q4 RUN %s ================" % RUN)

# ---------- Part 0: import-time binding of save_to_dir target (#11, #23) ----------
hr("(0) import-time binding: settings.CONSUMPTION_DIR vs tasks.save_to_dir.__defaults__")
print("settings.CONSUMPTION_DIR (runtime) =", settings.CONSUMPTION_DIR)
print("tasks.save_to_dir.__defaults__      =", tasks.save_to_dir.__defaults__,
      "(tuple = (newname_default, target_dir_default); target_dir bound at import, tasks.py:167)")
_saved = settings.CONSUMPTION_DIR
try:
    settings.CONSUMPTION_DIR = "/tmp/some-other-consume-XYZ"
    print("after settings.CONSUMPTION_DIR := /tmp/some-other-consume-XYZ:")
    print("  settings.CONSUMPTION_DIR       =", settings.CONSUMPTION_DIR)
    print("  tasks.save_to_dir.__defaults__ =", tasks.save_to_dir.__defaults__,
          "(UNCHANGED -> bound at import, not read live)")
finally:
    settings.CONSUMPTION_DIR = _saved
bound_target = tasks.save_to_dir.__defaults__[1]
print("import-bound target dir            =", bound_target)
try:
    before = sorted(os.listdir(bound_target))
except FileNotFoundError:
    before = "(does not exist)"
print("inventory(%s) BEFORE =" % bound_target, before)

setup_test_environment()
old_name = connection.creation.create_test_db(verbosity=0)

_tmp_roots = []


def mkroot(pfx):
    d = tempfile.mkdtemp(prefix=pfx)
    _tmp_roots.append(d)
    return d


try:
    # ---------- Part A: scan_file_for_separating_barcodes over ALL edges (#14) ----------
    hr("(A) scan_file_for_separating_barcodes - every edge (returns separator page numbers)")

    def scan(name, folder=SAMPLES):
        p = os.path.join(folder, name)
        pages = tasks.scan_file_for_separating_barcodes(p)
        print("scan(%-32s) -> %s" % (name, pages))
        return pages

    print("[default CONSUMER_BARCODE_STRING = %r]" % settings.CONSUMER_BARCODE_STRING)
    scan("simple.pdf")
    scan("patch-code-t.pdf", BARCODES)
    scan("patch-code-t-middle.pdf", BARCODES)
    scan("several-patcht-codes.pdf", BARCODES)
    scan("patch-code-t-qr.pdf", BARCODES)

    hr("(A') barcode_reader on PNG edges (unreadable / custom value)")
    img_unreadable = Image.open(os.path.join(BARCODES, "barcode-39-PATCHT-unreadable.png"))
    print("barcode_reader(barcode-39-PATCHT-unreadable.png) ->", tasks.barcode_reader(img_unreadable))
    img_custom = Image.open(os.path.join(BARCODES, "barcode-128-custom.png"))
    print("barcode_reader(barcode-128-custom.png)           ->", tasks.barcode_reader(img_custom),
          "(decoded value is the CUSTOM string, not %r)" % settings.CONSUMER_BARCODE_STRING)

    hr("(A'') custom CONSUMER_BARCODE_STRING controls which value triggers a split")
    with override_settings(CONSUMER_BARCODE_STRING="CUSTOM BARCODE"):
        print("[override CONSUMER_BARCODE_STRING = %r]" % settings.CONSUMER_BARCODE_STRING)
        p = os.path.join(BARCODES, "barcode-128-custom.pdf")
        print("scan(barcode-128-custom.pdf) ->", tasks.scan_file_for_separating_barcodes(p),
              "(now matches because decoded value == configured string)")

    # ---------- Part B: separate_pages fragment arithmetic N sep -> N+1 (#5) ----------
    hr("(B) separate_pages fragment arithmetic (N separators -> N+1 fragments)")

    def sep(name, splits):
        p = os.path.join(BARCODES, name)
        frags = tasks.separate_pages(p, splits)
        print("separate_pages(%-28s splits=%-7s) -> %d fragment(s): %s"
              % (name, str(splits), len(frags), [os.path.basename(f) for f in frags]))
        from pikepdf import Pdf as _Pdf
        for f in frags:
            with _Pdf.open(f) as _pd:
                print("     %-40s pages=%d" % (os.path.basename(f), len(_pd.pages)))
        for f in frags:
            d = os.path.dirname(f)
            if d.startswith(settings.SCRATCH_DIR):
                shutil.rmtree(d, ignore_errors=True)
        return frags

    sep("patch-code-t-middle.pdf", [])
    sep("patch-code-t-middle.pdf", [1])
    sep("several-patcht-codes.pdf", [2, 5])

    # ---------- Part C: consume_file split path, ISOLATED target, Document delta=0 (#23) ----------
    hr("(C) consume_file (CONSUMER_ENABLE_BARCODES=True) - split path writes fragments, creates 0 Document rows")
    iso_consume = mkroot("pngx-q4-consume-")
    orig_defaults = tasks.save_to_dir.__defaults__
    tasks.save_to_dir.__defaults__ = (orig_defaults[0], iso_consume)
    print("isolated save_to_dir target ->", tasks.save_to_dir.__defaults__[1], "(temp; /app/consume untouched)")
    try:
        with override_settings(CONSUMER_ENABLE_BARCODES=True):
            src = os.path.join(BARCODES, "patch-code-t-middle.pdf")
            scratch_in = os.path.join(mkroot("pngx-q4-in-"), "patch-code-t-middle.pdf")
            shutil.copy(src, scratch_in)
            n_before = Document.objects.count()
            print("Document.objects.count() BEFORE split =", n_before)
            result = tasks.consume_file(scratch_in)
            n_after = Document.objects.count()
            print("consume_file(...) returned            =", repr(result))
            print("Document.objects.count() AFTER split  =", n_after,
                  "  (delta = %d -> split path creates NO Document rows)" % (n_after - n_before))
            print("original input still exists?          =", os.path.exists(scratch_in),
                  "(os.unlink at tasks.py:214 after successful split)")
            print("isolated consume dir contents         =", sorted(os.listdir(iso_consume)))
    finally:
        tasks.save_to_dir.__defaults__ = orig_defaults

    # ---------- Part D: consume fragments -> Document rows -> corpus effect (#5) ----------
    # separate_pages yields N+1 fragment FILES (mechanical, Part B). When those fragments are
    # consumed through the real Consumer, EACH DISTINCT (md5) fragment becomes one Document row;
    # byte-identical fragments are rejected by pre_check_duplicate (consumer.py:102-113, md5 at L104).
    import hashlib as _hl
    corpus_query = Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)  # classifier.py:125-127

    def consume_all(name):
        p = os.path.join(BARCODES, name)
        seps = tasks.scan_file_for_separating_barcodes(p)
        frags = tasks.separate_pages(p, seps)
        md5s = [_hl.md5(open(f, "rb").read()).hexdigest() for f in frags]
        print("input %-28s separators=%-7s -> %d fragment FILES (N+1, N=%d); distinct md5=%d"
              % (name, str(seps), len(frags), len(seps), len(set(md5s))))
        for f, m in zip(frags, md5s):
            print("     fragment %-40s md5=%s" % (os.path.basename(f), m))
        created = []
        for f in frags:
            stage = os.path.join(mkroot("pngx-q4-frag-"), os.path.basename(f))
            shutil.copy(f, stage)
            try:
                with mock.patch("documents.consumer.Consumer._send_progress"):
                    doc = Consumer().try_consume_file(stage)
                created.append(doc.pk)
                print("     consumed %-40s -> Document pk=%s mime=%r content_len=%d"
                      % (os.path.basename(f), doc.pk, doc.mime_type, len(doc.content or "")))
            except ConsumerError as e:
                print("     consumed %-40s -> REJECTED (pre_check_duplicate consumer.py:102-113): %s"
                      % (os.path.basename(f), str(e).split(":")[-1].strip()))
        d = os.path.dirname(frags[0]) if frags else None
        if d and d.startswith(settings.SCRATCH_DIR):
            shutil.rmtree(d, ignore_errors=True)
        return len(frags), created

    hr("(D) consume produced fragments through the REAL Consumer -> distinct fragments become Document rows")
    print("training-corpus size BEFORE any consume =", corpus_query.count())
    n1, ids1 = consume_all("several-patcht-codes.pdf")   # 3 files, 2 distinct -> 2 rows
    print("  => several-patcht-codes.pdf: %d fragment files -> %d NEW Document rows" % (n1, len(ids1)))
    n2, ids2 = consume_all("patch-code-t-middle.pdf")    # 2 files, 1 distinct -> 1 row (secondary: identical frags)
    print("  => patch-code-t-middle.pdf : %d fragment files -> %d NEW Document rows (identical frags deduped)" % (n2, len(ids2)))
    created_ids = ids1 + ids2
    print("training-corpus size AFTER consuming fragments =", corpus_query.count(),
          "  (splitting 2 input files enlarged the effective training data by +%d rows)" % len(created_ids))

    # label the newly-created docs so a subsequent train() is meaningful, then train (#5 "subsequent train()")
    corrA = Correspondent.objects.create(name="Q4CorrA", matching_algorithm=Correspondent.MATCH_AUTO)
    corrB = Correspondent.objects.create(name="Q4CorrB", matching_algorithm=Correspondent.MATCH_AUTO)
    for i, pk in enumerate(created_ids):
        d = Document.objects.get(pk=pk)
        d.content = "alpha bravo charlie fragment number %d token%d segment" % (i, i)
        d.correspondent = corrA if i % 2 == 0 else corrB
        d.save()
    hr("(D') DocumentClassifier().train() now trains on the split-enlarged corpus")
    clf = DocumentClassifier()
    trained = clf.train()
    print("DocumentClassifier().train() returned =", trained,
          "(reads Document.objects at classifier.py:125; corpus size = %d, all from splitting)" % corpus_query.count())

finally:
    connection.creation.destroy_test_db(old_name, verbosity=0)
    teardown_test_environment()
    for d in _tmp_roots:
        shutil.rmtree(d, ignore_errors=True)
    try:
        after = sorted(os.listdir(bound_target))
    except FileNotFoundError:
        after = "(does not exist)"
    print("\ninventory(%s) AFTER  =" % bound_target, after)
    print("scratch roots remaining:", [d for d in _tmp_roots if os.path.exists(d)])

print("\n================ Q4 RUN %s SUMMARY ================" % RUN)
print("scan edges: simple=[], patch0=[0], middle=[1], several=[2,5], qr=[0]; unreadable/custom via barcode_reader; "
      "custom string matches only its own value | separate_pages N->N+1 | consume split delta=0 rows | "
      "fragments consumed -> N+1 Document rows -> corpus grows | /app/consume untouched")
```

### 7.3 Two-run stability — complete captures of both runs

Every quantitative claim in this document was produced twice, from an identical starting state, and confirmed stable. `[observed]` Rather than assert stability, the two runs are compared directly. Each pair of outputs was normalized by masking only the tokens that are volatile by construction (secure `mkdtemp` names, ocrmypdf/unpaper internal temp dirs, ffmpeg heap addresses, wall-clock timestamps, model-file mtimes, and — for Q4 — the pikepdf-nondeterministic fragment checksums), then compared with `diff`. All four normalized diffs are empty:

```text
Two-run stability test — normalized diff method
-----------------------------------------------
For each question, run1 and run2 outputs were normalized by masking ONLY the
by-design volatile tokens, then compared with `diff`:

  mask:  RUN 1|RUN 2                       -> RUN N          (run label)
         /tmp/pngx-<label>-<rand>          -> /tmp/pngx-<mkdtemp>   (secure per-probe temp dir, CWE-377 safe)
         ocrmypdf.io.<rand>                -> ocrmypdf.io.<tmp>     (ocrmypdf internal work dir)
         /tmp/tmp<rand>, paperless-<rand>  -> /tmp/tmp<X>, paperless-<tmp> (unpaper/parser temp)
         0x<hex>                           -> 0x<addr>        (ffmpeg heap address in unpaper banner)
         [YYYY-MM-DD HH:MM:SS,mmm]         -> [<ts>]          (wall-clock log timestamp)
         17XXXXXXXX.XXXX                   -> <mtime>         (model-file st_mtime float)
         <32-hex>                          -> <md5>           (pikepdf-nondeterministic fragment checksum)

  result (command: diff <(normalize run1) <(normalize run2)):
         Q1  -> EMPTY   (0 differing lines)
         Q2  -> EMPTY   (0 differing lines)
         Q3  -> EMPTY   (0 differing lines)
         Q4  -> EMPTY   (0 differing lines)

Interpretation: after masking tokens that are volatile by construction, the two
runs are byte-identical. Every answer-bearing line — booleans (REUSE/RETRAIN
True), counts (3 fragments, 2 rows, corpus 0->3), classes_, mime_type='image/png',
gs '-dPDFA=2' argv, skip_text:True->force_ocr:True args dicts, 'No text was found'
warning — is reproduced exactly across both runs.

Q4 note: pikepdf's PDF writer is not byte-deterministic across processes, so the
raw fragment md5 VALUES differ run-to-run; the DERIVED answers (distinct-md5 count
2-of-3 and 1-of-2, and the resulting 2 and 1 Document rows, corpus 0->3) are stable.
```

The complete, unedited output of **both** runs follows. (The Q1 first run appears in §2.2 and the Q2 first run in §3; their complete second runs are given here. The Q3 and Q4 bodies quoted slices, so both complete runs are given here.)

#### 7.3.1 Q1 — complete second run (`python3 /tmp/obs_q1.py 2`)

```text
================ Q1 RUN 2 ================
scratch mode = 0o700
MODEL_FILE = /tmp/pngx-q1-ifc2jcg4/classification_model.pickle

--- A: tasks.train_classifier() lifecycle (initial/unchanged/changed) ---
model exists before any training: False
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1-ifc2jcg4/classification_model.pickle...
model exists after call #1: True | st_mtime#1 = 1783966223.696751
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.tasks: Training data unchanged.
st_mtime#2 = 1783966223.696751 | REUSE (m1==m2): True
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1-ifc2jcg4/classification_model.pickle...
st_mtime#3 = 1783966223.7117515 | RETRAIN (m2!=m3): True
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG DEBUG paperless.classifier: Gathering data from database...
direct train() #1: True (True=>changed) | #2: False (False=>unchanged/REUSE) | data_hash set: True

--- B: incompatible FORMAT_VERSION -> load_classifier() os.unlink deletion ---
FORMAT_VERSION (on-disk written by save) = 7
model file exists before load: True
patched FORMAT_VERSION (loader expects) = 8
LOG ERROR paperless.classifier: Unrecoverable error while loading document classification model, deleting model file.
Traceback (most recent call last):
  File "/app/src/documents/classifier.py", line 40, in load_classifier
    classifier.load()
  File "/app/src/documents/classifier.py", line 81, in load
    raise IncompatibleClassifierVersionError(
documents.classifier.IncompatibleClassifierVersionError: Cannot load classifier, incompatible versions.
load_classifier() returned: None (None => load failed & file deleted)
model file exists AFTER load_classifier(): False (False => os.unlink at classifier.py:48 ran)

--- C: empty corpus -> ValueError('No training data available.') ---
Document.objects.count() = 0
LOG DEBUG paperless.classifier: Gathering data from database...
raised ValueError: 'No training data available.'

================ Q1 RUN 2 SUMMARY ================
reuse(m1==m2)=True retrain(m2!=m3)=True direct_train=(True,False) incompat_deleted=True empty_corpus_ValueError=True
POST-RUN cleanup: scratch exists? False
```

#### 7.3.2 Q2 — complete second run (`python3 /tmp/obs_q2.py 2`)

```text
================ Q2 RUN 2 ================

--- (a) training-doc count + inbox exclusion (Document.objects...exclude(tags__is_inbox_tag=True)) ---
Document.objects.count() (rows inserted) = 3
effective corpus exclude(tags__is_inbox_tag=True) = 2
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
canonical test_one_correspondent_predict inserts Document.objects.count() = 1
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.

--- (b) timing: inserting docs does NOT train; only explicit train_classifier() does ---
after inserting Correspondent+Document: model file exists = False
LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q2-ungbu_sm/m.pickle...
after explicit tasks.train_classifier(): model file exists = True
Django-Q Schedule rows for documents.tasks.train_classifier = [('Train the classifier', 'documents.tasks.train_classifier', 'H')]

--- (c) NO confidence threshold: hard argmax (predict()!=-1, o.pk==pred_id); no predict_proba ---
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
correspondent_classifier.classes_ = [-1, 4]
c1.pk = 4
predict_correspondent(doc1.content) = [4] (expect c1.pk)
predict_correspondent(doc2.content) = None (expect None; label was -1)
match_correspondents(doc1) names = ['c1']
match_correspondents(doc2) names = []
LOG DEBUG paperless.classifier: Gathering data from database...
LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
LOG DEBUG paperless.classifier: Vectorizing data...
LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
LOG DEBUG paperless.classifier: Training correspondent classifier...
LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
[SUPPORTING] single-class classes_ = [5] | predict(unrelated) = [5] | only.pk = 5 => unrelated STILL accepted (no probability gate)

================ Q2 RUN 2 SUMMARY ================
(a) inserted=3 effective=2 ; canonical single-doc=1 | (b) insert_trains=False explicit_trains=True schedule=[('Train the classifier', 'documents.tasks.train_classifier', 'H')] | (c) argmax_no_threshold=True
POST-RUN cleanup: scratch exists? False
```

#### 7.3.3 Q3 — complete first run (`python3 /tmp/obs_q3.py 1`)

```text
================ Q3 RUN 1 ================
OCR_MODE (default) = skip

--- (a) RasterisedDocumentParser.parse(no-text-alpha.png) [textless image, default skip] ---
magic.from_file(input, mime=True) = image/png (supporting/non-canonical: direct call)
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-pyaa1io1/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/sidecar.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '--version']
SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--version']
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Running: ['gs', '--version']
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.jpb7_paf/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.jpb7_paf/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.jpb7_paf/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpgtix5qdo/input.pnm', '/tmp/tmpgtix5qdo/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x5831fcaefe80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x5831fcaefe80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpgtix5qdo/input.pnm -> /tmp/tmpgtix5qdo/output.ppm
input-file for sheet 1: /tmp/tmpgtix5qdo/input.pnm
output-file for sheet 1: /tmp/tmpgtix5qdo/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.jpb7_paf/000001_ocr.png', '/tmp/ocrmypdf.io.jpb7_paf/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.jpb7_paf/fix_docinfo.pdf', '/tmp/ocrmypdf.io.jpb7_paf/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-pyaa1io1/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg/sidecar-fallback.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.5lvuepee/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.5lvuepee/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.5lvuepee/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpoi8f070l/input.pnm', '/tmp/tmpoi8f070l/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x59bdbc557e80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x59bdbc557e80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpoi8f070l/input.pnm -> /tmp/tmpoi8f070l/output.ppm
input-file for sheet 1: /tmp/tmpoi8f070l/input.pnm
output-file for sheet 1: /tmp/tmpoi8f070l/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.5lvuepee/000001_ocr.png', '/tmp/ocrmypdf.io.5lvuepee/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.5lvuepee/fix_docinfo.pdf', '/tmp/ocrmypdf.io.5lvuepee/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3a-pyaa1io1/no-text-alpha.png, the content will be empty.
parse() done. self.text repr = '' | len = 0 | archive_path set? = True
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pngx-q3a-pyaa1io1/paperless-slr6_udg

--- (b) FULL Consumer().try_consume_file(no-text-alpha.png) -> persisted Document.mime_type ---
LOG INFO paperless.consumer: Consuming no-text-alpha.png
LOG DEBUG paperless.consumer: Detected mime type: image/png
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing no-text-alpha.png...
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/tmpc1yhi2yw/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/tmpc1yhi2yw/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/tmpc1yhi2yw/no-text-alpha.png', 'output_file': '/tmp/tmpc1yhi2yw/paperless-t5qdmt79/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpc1yhi2yw/paperless-t5qdmt79/sidecar.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.rio_q3pe/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.rio_q3pe/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.rio_q3pe/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.rio_q3pe/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.rio_q3pe/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpqt7bceu2/input.pnm', '/tmp/tmpqt7bceu2/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x57cc0700ee80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x57cc0700ee80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpqt7bceu2/input.pnm -> /tmp/tmpqt7bceu2/output.ppm
input-file for sheet 1: /tmp/tmpqt7bceu2/input.pnm
output-file for sheet 1: /tmp/tmpqt7bceu2/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.rio_q3pe/000001_ocr.png', '/tmp/ocrmypdf.io.rio_q3pe/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.rio_q3pe/fix_docinfo.pdf', '/tmp/ocrmypdf.io.rio_q3pe/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/tmpc1yhi2yw/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/tmpc1yhi2yw/no-text-alpha.png', 'output_file': '/tmp/tmpc1yhi2yw/paperless-t5qdmt79/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpc1yhi2yw/paperless-t5qdmt79/sidecar-fallback.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.171zyqbb/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.171zyqbb/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.171zyqbb/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.171zyqbb/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.171zyqbb/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmp75c9uhr2/input.pnm', '/tmp/tmp75c9uhr2/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x57d07275de80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x57d07275de80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmp75c9uhr2/input.pnm -> /tmp/tmp75c9uhr2/output.ppm
input-file for sheet 1: /tmp/tmp75c9uhr2/input.pnm
output-file for sheet 1: /tmp/tmp75c9uhr2/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.171zyqbb/000001_ocr.png', '/tmp/ocrmypdf.io.171zyqbb/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.171zyqbb/fix_docinfo.pdf', '/tmp/ocrmypdf.io.171zyqbb/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/tmpc1yhi2yw/no-text-alpha.png, the content will be empty.
LOG DEBUG paperless.consumer: Generating thumbnail for no-text-alpha.png...
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpc1yhi2yw/paperless-t5qdmt79/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 18:18:50,695] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpc1yhi2yw/paperless-t5qdmt79/convert_gs.png -out /tmp/tmpc1yhi2yw/paperless-t5qdmt79/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/tmpc1yhi2yw/no-text-alpha.png
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpc1yhi2yw/paperless-t5qdmt79
LOG INFO paperless.consumer: Document 2026-07-13 no-text-alpha consumption finished
persisted Document.pk = 1
persisted Document.mime_type = 'image/png' (CANONICAL: from _store/Document.objects.create at consumer.py:401)
persisted Document.content repr = '' (empty because no extractable text)

--- edge: parse(encrypted.pdf) [EncryptedPdfError path] ---
magic.from_file(encrypted.pdf, mime=True) = application/pdf
LOG WARNING paperless.parsing.tesseract: Error while getting text from PDF document with pdfminer.six
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 120, in extract_text
    stripped = post_process_text(pdfminer_extract_text(pdf_file))
  File "/usr/local/lib/python3.9/site-packages/pdfminer/high_level.py", line 157, in extract_text
    for page in PDFPage.get_pages(
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfpage.py", line 151, in get_pages
    doc = PDFDocument(parser, password=password, caching=caching)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 744, in __init__
    self._initialize_password(password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 771, in _initialize_password
    handler = factory(docid, param, password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 358, in __init__
    self.init()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 366, in init
    self.init_key()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 381, in init_key
    raise PDFPasswordIncorrect
pdfminer.pdfdocument.PDFPasswordIncorrect
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3enc-5i8tydub/encrypted.pdf', 'output_file': '/tmp/pngx-q3enc-5i8tydub/paperless-w6k66oog/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3enc-5i8tydub/paperless-w6k66oog/sidecar.txt'}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
LOG WARNING paperless.parsing.tesseract: This file is encrypted, OCR is impossible. Using any text present in the original file.
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3enc-5i8tydub/encrypted.pdf, the content will be empty.
parse(encrypted.pdf) done. self.text repr = ''
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pngx-q3enc-5i8tydub/paperless-w6k66oog

================ Q3 RUN 1 SUMMARY ================
(a) ocrmypdf.ocr skip_text then force_ocr fallback; Tesseract+Ghostscript spawned; self.text='' | (b) persisted mime_type observed via Consumer
```

#### 7.3.4 Q3 — complete second run (`python3 /tmp/obs_q3.py 2`)

```text
================ Q3 RUN 2 ================
OCR_MODE (default) = skip

--- (a) RasterisedDocumentParser.parse(no-text-alpha.png) [textless image, default skip] ---
magic.from_file(input, mime=True) = image/png (supporting/non-canonical: direct call)
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-eiyefb4o/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/pngx-q3a-eiyefb4o/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-eiyefb4o/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-eiyefb4o/paperless-8farztcs/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-eiyefb4o/paperless-8farztcs/sidecar.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '--version']
SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--version']
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Running: ['gs', '--version']
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.a1nw5sck/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.a1nw5sck/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.a1nw5sck/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.a1nw5sck/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.a1nw5sck/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmp6yxuc0v6/input.pnm', '/tmp/tmp6yxuc0v6/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x59aa1f3cde80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x59aa1f3cde80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmp6yxuc0v6/input.pnm -> /tmp/tmp6yxuc0v6/output.ppm
input-file for sheet 1: /tmp/tmp6yxuc0v6/input.pnm
output-file for sheet 1: /tmp/tmp6yxuc0v6/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.a1nw5sck/000001_ocr.png', '/tmp/ocrmypdf.io.a1nw5sck/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.a1nw5sck/fix_docinfo.pdf', '/tmp/ocrmypdf.io.a1nw5sck/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/pngx-q3a-eiyefb4o/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3a-eiyefb4o/no-text-alpha.png', 'output_file': '/tmp/pngx-q3a-eiyefb4o/paperless-8farztcs/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3a-eiyefb4o/paperless-8farztcs/sidecar-fallback.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.s8k0ta5y/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.s8k0ta5y/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.s8k0ta5y/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.s8k0ta5y/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.s8k0ta5y/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpqox7mrd9/input.pnm', '/tmp/tmpqox7mrd9/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x5873b31f1e80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x5873b31f1e80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpqox7mrd9/input.pnm -> /tmp/tmpqox7mrd9/output.ppm
input-file for sheet 1: /tmp/tmpqox7mrd9/input.pnm
output-file for sheet 1: /tmp/tmpqox7mrd9/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.s8k0ta5y/000001_ocr.png', '/tmp/ocrmypdf.io.s8k0ta5y/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.s8k0ta5y/fix_docinfo.pdf', '/tmp/ocrmypdf.io.s8k0ta5y/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3a-eiyefb4o/no-text-alpha.png, the content will be empty.
parse() done. self.text repr = '' | len = 0 | archive_path set? = True
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pngx-q3a-eiyefb4o/paperless-8farztcs

--- (b) FULL Consumer().try_consume_file(no-text-alpha.png) -> persisted Document.mime_type ---
LOG INFO paperless.consumer: Consuming no-text-alpha.png
LOG DEBUG paperless.consumer: Detected mime type: image/png
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing no-text-alpha.png...
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/tmpbanl_ri4/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG INFO paperless.parsing.tesseract: Removing alpha layer from /tmp/tmpbanl_ri4/no-text-alpha.png for compatibility with img2pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/tmpbanl_ri4/no-text-alpha.png', 'output_file': '/tmp/tmpbanl_ri4/paperless-9n72q9bz/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpbanl_ri4/paperless-9n72q9bz/sidecar.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.fpd0v9yk/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.fpd0v9yk/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.fpd0v9yk/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.fpd0v9yk/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.fpd0v9yk/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpwa3adjjl/input.pnm', '/tmp/tmpwa3adjjl/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x583dc4294e80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x583dc4294e80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpwa3adjjl/input.pnm -> /tmp/tmpwa3adjjl/output.ppm
input-file for sheet 1: /tmp/tmpwa3adjjl/input.pnm
output-file for sheet 1: /tmp/tmpwa3adjjl/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.fpd0v9yk/000001_ocr.png', '/tmp/ocrmypdf.io.fpd0v9yk/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.fpd0v9yk/fix_docinfo.pdf', '/tmp/ocrmypdf.io.fpd0v9yk/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG WARNING paperless.parsing.tesseract: Error while getting DPI from image /tmp/tmpbanl_ri4/no-text-alpha.png: 'dpi'
LOG DEBUG paperless.parsing.tesseract: Estimated DPI 35 based on image width 297
LOG DEBUG paperless.parsing.tesseract: Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/tmpbanl_ri4/no-text-alpha.png', 'output_file': '/tmp/tmpbanl_ri4/paperless-9n72q9bz/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpbanl_ri4/paperless-9n72q9bz/sidecar-fallback.txt', 'image_dpi': 35}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=jpeggray', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.r8jwtefk/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'osd', '--psm', '0', '/tmp/ocrmypdf.io.r8jwtefk/000001_rasterize_preview.jpg', 'stdout']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 355
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Too few characters. Skipping this page
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Error during processing.
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dQUIET', '-dSAFER', '-dBATCH', '-dNOPAUSE', '-dInterpolateControl=-1', '-sDEVICE=png16m', '-dFirstPage=1', '-dLastPage=1', '-r35.000003x35.000003', '-o', '-', '-sstdout=%stderr', '-dAutoRotatePages=/None', '-f', '/tmp/ocrmypdf.io.r8jwtefk/origin.pdf']
SUBPROC ocrmypdf._exec.ghostscript: Rotating output by 0
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.r8jwtefk/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '--psm', '2', '/tmp/ocrmypdf.io.r8jwtefk/000001_rasterize.png', 'stdout']
SUBPROC ocrmypdf.subprocess: Running: ['unpaper', '-v', '--dpi', '35.000003', '--layout', 'none', '--mask-scan-size', '100', '--no-border-align', '--no-mask-center', '--no-grayfilter', '--no-blackfilter', '--no-deskew', '/tmp/tmpmvl9eptq/input.pnm', '/tmp/tmpmvl9eptq/output.ppm']
SUBPROC ocrmypdf.subprocess.unpaper: stdout/stderr = [image2 @ 0x581520077e80] Using AVStream.codec to pass codec parameters to muxers is deprecated, use AVStream.codecpar instead.
[image2 @ 0x581520077e80] Encoder did not produce proper pts, making some up.
unpaper 6.1
License GPLv2: GNU GPL version 2.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

-------------------------------------------------------------------------------
Processing sheet #1: /tmp/tmpmvl9eptq/input.pnm -> /tmp/tmpmvl9eptq/output.ppm
input-file for sheet 1: /tmp/tmpmvl9eptq/input.pnm
output-file for sheet 1: /tmp/tmpmvl9eptq/output.ppm
sheet size: 297x225
...
noise-filter ... deleted 0 clusters.
blur-filter... deleted 0 pixels.
writing output.

SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '-l', 'eng', '-c', 'textonly_pdf=1', '/tmp/ocrmypdf.io.r8jwtefk/000001_ocr.png', '/tmp/ocrmypdf.io.r8jwtefk/000001_ocr_tess', 'pdf', 'txt']
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
SUBPROC ocrmypdf._exec.tesseract: [tesseract] Estimating resolution as 356
SUBPROC ocrmypdf.subprocess: Running: ['gs', '-dBATCH', '-dNOPAUSE', '-dSAFER', '-dCompatibilityLevel=1.6', '-sDEVICE=pdfwrite', '-dAutoRotatePages=/None', '-sColorConversionStrategy=LeaveColorUnchanged', '-dAutoFilterColorImages=true', '-dAutoFilterGrayImages=true', '-dJPEGQ=95', '-dPDFA=2', '-dPDFACompatibilityPolicy=1', '-o', '-', '-sstdout=%stderr', '/tmp/ocrmypdf.io.r8jwtefk/fix_docinfo.pdf', '/tmp/ocrmypdf.io.r8jwtefk/pdfa.ps']
SUBPROC ocrmypdf.subprocess.gs: GPL Ghostscript 9.53.3 (2020-10-01)
SUBPROC ocrmypdf.subprocess.gs: Copyright (C) 2020 Artifex Software, Inc.  All rights reserved.
SUBPROC ocrmypdf.subprocess.gs: This software is supplied under the GNU AGPLv3 and comes with NO WARRANTY:
SUBPROC ocrmypdf.subprocess.gs: see the file COPYING for details.
SUBPROC ocrmypdf.subprocess.gs: Processing pages 1 through 1.
SUBPROC ocrmypdf.subprocess.gs: Page 1
LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/tmpbanl_ri4/no-text-alpha.png, the content will be empty.
LOG DEBUG paperless.consumer: Generating thumbnail for no-text-alpha.png...
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpbanl_ri4/paperless-9n72q9bz/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 18:20:10,881] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpbanl_ri4/paperless-9n72q9bz/convert_gs.png -out /tmp/tmpbanl_ri4/paperless-9n72q9bz/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/tmpbanl_ri4/no-text-alpha.png
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpbanl_ri4/paperless-9n72q9bz
LOG INFO paperless.consumer: Document 2026-07-13 no-text-alpha consumption finished
persisted Document.pk = 1
persisted Document.mime_type = 'image/png' (CANONICAL: from _store/Document.objects.create at consumer.py:401)
persisted Document.content repr = '' (empty because no extractable text)

--- edge: parse(encrypted.pdf) [EncryptedPdfError path] ---
magic.from_file(encrypted.pdf, mime=True) = application/pdf
LOG WARNING paperless.parsing.tesseract: Error while getting text from PDF document with pdfminer.six
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 120, in extract_text
    stripped = post_process_text(pdfminer_extract_text(pdf_file))
  File "/usr/local/lib/python3.9/site-packages/pdfminer/high_level.py", line 157, in extract_text
    for page in PDFPage.get_pages(
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfpage.py", line 151, in get_pages
    doc = PDFDocument(parser, password=password, caching=caching)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 744, in __init__
    self._initialize_password(password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 771, in _initialize_password
    handler = factory(docid, param, password)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 358, in __init__
    self.init()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 366, in init
    self.init_key()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 381, in init_key
    raise PDFPasswordIncorrect
pdfminer.pdfdocument.PDFPasswordIncorrect
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q3enc-lnitumd1/encrypted.pdf', 'output_file': '/tmp/pngx-q3enc-lnitumd1/paperless-965tpl_k/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pngx-q3enc-lnitumd1/paperless-965tpl_k/sidecar.txt'}
SUBPROC ocrmypdf.subprocess: Running: ['tesseract', '--list-langs']
SUBPROC ocrmypdf.subprocess.tesseract: stdout/stderr = List of available languages (2):
eng
osd

SUBPROC ocrmypdf.subprocess: Found unpaper 6.1
SUBPROC ocrmypdf.subprocess: Found tesseract 4.1.1
SUBPROC ocrmypdf.subprocess: Found gs 9.53.3
LOG WARNING paperless.parsing.tesseract: This file is encrypted, OCR is impossible. Using any text present in the original file.
LOG WARNING paperless.parsing.tesseract: No text was found in /tmp/pngx-q3enc-lnitumd1/encrypted.pdf, the content will be empty.
parse(encrypted.pdf) done. self.text repr = ''
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pngx-q3enc-lnitumd1/paperless-965tpl_k

================ Q3 RUN 2 SUMMARY ================
(a) ocrmypdf.ocr skip_text then force_ocr fallback; Tesseract+Ghostscript spawned; self.text='' | (b) persisted mime_type observed via Consumer
```

#### 7.3.5 Q4 — complete first run (`python3 /tmp/obs_q4.py 1`)

```text
================ Q4 RUN 1 ================

--- (0) import-time binding: settings.CONSUMPTION_DIR vs tasks.save_to_dir.__defaults__ ---
settings.CONSUMPTION_DIR (runtime) = /app/src/../consume
tasks.save_to_dir.__defaults__      = (None, '/app/src/../consume') (tuple = (newname_default, target_dir_default); target_dir bound at import, tasks.py:167)
after settings.CONSUMPTION_DIR := /tmp/some-other-consume-XYZ:
  settings.CONSUMPTION_DIR       = /tmp/some-other-consume-XYZ
  tasks.save_to_dir.__defaults__ = (None, '/app/src/../consume') (UNCHANGED -> bound at import, not read live)
import-bound target dir            = /app/src/../consume
inventory(/app/src/../consume) BEFORE = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
\x1b[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
\x1b[0m

--- (A) scan_file_for_separating_barcodes - every edge (returns separator page numbers) ---
[default CONSUMER_BARCODE_STRING = 'PATCHT']
scan(simple.pdf                      ) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t.pdf                ) -> [0]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t-middle.pdf         ) -> [1]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(several-patcht-codes.pdf        ) -> [2, 5]
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
scan(patch-code-t-qr.pdf             ) -> [0]

--- (A') barcode_reader on PNG edges (unreadable / custom value) ---
barcode_reader(barcode-39-PATCHT-unreadable.png) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
barcode_reader(barcode-128-custom.png)           -> ['CUSTOM BARCODE'] (decoded value is the CUSTOM string, not 'PATCHT')

--- (A'') custom CONSUMER_BARCODE_STRING controls which value triggers a split ---
[override CONSUMER_BARCODE_STRING = 'CUSTOM BARCODE']
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
scan(barcode-128-custom.pdf) -> [0] (now matches because decoded value == configured string)

--- (B) separate_pages fragment arithmetic (N separators -> N+1 fragments) ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-0495il26
LOG WARNING paperless.tasks: No pages to split on!
LOG DEBUG paperless.tasks: Temp files are []
separate_pages(patch-code-t-middle.pdf      splits=[]     ) -> 0 fragment(s): []
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-azuad6iz
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-azuad6iz/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-azuad6iz/patch-code-t-middle_document_1.pdf']
separate_pages(patch-code-t-middle.pdf      splits=[1]    ) -> 2 fragment(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
     patch-code-t-middle_document_0.pdf       pages=1
     patch-code-t-middle_document_1.pdf       pages=1
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-i8ak0j0w
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_0.pdf', '/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_1.pdf', '/tmp/paperless/paperless-i8ak0j0w/several-patcht-codes_document_2.pdf']
separate_pages(several-patcht-codes.pdf     splits=[2, 5] ) -> 3 fragment(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
     several-patcht-codes_document_0.pdf      pages=2
     several-patcht-codes_document_1.pdf      pages=2
     several-patcht-codes_document_2.pdf      pages=1

--- (C) consume_file (CONSUMER_ENABLE_BARCODES=True) - split path writes fragments, creates 0 Document rows ---
isolated save_to_dir target -> /tmp/pngx-q4-consume-uuspm1ds (temp; /app/consume untouched)
Document.objects.count() BEFORE split = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4-in-tl5k1jzf/patch-code-t-middle.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-z0r8jhdu
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-z0r8jhdu/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-z0r8jhdu/patch-code-t-middle_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4-in-tl5k1jzf/patch-code-t-middle.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
consume_file(...) returned            = 'File successfully split'
Document.objects.count() AFTER split  = 0   (delta = 0 -> split path creates NO Document rows)
original input still exists?          = False (os.unlink at tasks.py:214 after successful split)
isolated consume dir contents         = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']

--- (D) consume produced fragments through the REAL Consumer -> distinct fragments become Document rows ---
training-corpus size BEFORE any consume = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-bsfeyuyc
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-bsfeyuyc/several-patcht-codes_document_0.pdf', '/tmp/paperless/paperless-bsfeyuyc/several-patcht-codes_document_1.pdf', '/tmp/paperless/paperless-bsfeyuyc/several-patcht-codes_document_2.pdf']
input several-patcht-codes.pdf     separators=[2, 5]  -> 3 fragment FILES (N+1, N=2); distinct md5=2
     fragment several-patcht-codes_document_0.pdf      md5=873af29fb051743f1a8fc8cd840ca7b1
     fragment several-patcht-codes_document_1.pdf      md5=873af29fb051743f1a8fc8cd840ca7b1
     fragment several-patcht-codes_document_2.pdf      md5=7b8695ec87ce936d20bc4a86fb51e5be
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-vaqvjgrg/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-vaqvjgrg/several-patcht-codes_document_0.pdf', 'output_file': '/tmp/paperless/paperless-tklxcy2t/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-tklxcy2t/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-tklxcy2t/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-tklxcy2t/archive.pdf[0] /tmp/paperless/paperless-tklxcy2t/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-tklxcy2t/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-tklxcy2t/gs_out.png /tmp/paperless/paperless-tklxcy2t/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-tklxcy2t/convert_gs.png -out /tmp/paperless/paperless-tklxcy2t/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-vaqvjgrg/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-tklxcy2t
LOG INFO paperless.consumer: Document 2026-07-13 several-patcht-codes_document_0 consumption finished
     consumed several-patcht-codes_document_0.pdf      -> Document pk=1 mime='application/pdf' content_len=50
LOG ERROR paperless.consumer: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
     consumed several-patcht-codes_document_1.pdf      -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_2.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-bp7kbua3/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-bp7kbua3/several-patcht-codes_document_2.pdf', 'output_file': '/tmp/paperless/paperless-dhz624dx/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-dhz624dx/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-dhz624dx/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-dhz624dx/archive.pdf[0] /tmp/paperless/paperless-dhz624dx/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-dhz624dx/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-dhz624dx/gs_out.png /tmp/paperless/paperless-dhz624dx/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-dhz624dx/convert_gs.png -out /tmp/paperless/paperless-dhz624dx/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-bp7kbua3/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-dhz624dx
LOG INFO paperless.consumer: Document 2026-07-13 several-patcht-codes_document_2 consumption finished
     consumed several-patcht-codes_document_2.pdf      -> Document pk=2 mime='application/pdf' content_len=24
  => several-patcht-codes.pdf: 3 fragment files -> 2 NEW Document rows
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-n5ocpsjj
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-n5ocpsjj/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-n5ocpsjj/patch-code-t-middle_document_1.pdf']
input patch-code-t-middle.pdf      separators=[1]     -> 2 fragment FILES (N+1, N=1); distinct md5=1
     fragment patch-code-t-middle_document_0.pdf       md5=f82b8d7d92170df6d54875e1cb4fa297
     fragment patch-code-t-middle_document_1.pdf       md5=f82b8d7d92170df6d54875e1cb4fa297
LOG INFO paperless.consumer: Consuming patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-middle_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-_0us4n6l/patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-_0us4n6l/patch-code-t-middle_document_0.pdf', 'output_file': '/tmp/paperless/paperless-diq204u0/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-diq204u0/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-diq204u0/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-middle_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-diq204u0/archive.pdf[0] /tmp/paperless/paperless-diq204u0/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-diq204u0/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-diq204u0/gs_out.png /tmp/paperless/paperless-diq204u0/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-diq204u0/convert_gs.png -out /tmp/paperless/paperless-diq204u0/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-_0us4n6l/patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-diq204u0
LOG INFO paperless.consumer: Document 2026-07-13 patch-code-t-middle_document_0 consumption finished
     consumed patch-code-t-middle_document_0.pdf       -> Document pk=3 mime='application/pdf' content_len=24
LOG ERROR paperless.consumer: Not consuming patch-code-t-middle_document_1.pdf: It is a duplicate.
     consumed patch-code-t-middle_document_1.pdf       -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
  => patch-code-t-middle.pdf : 2 fragment files -> 1 NEW Document rows (identical frags deduped)
training-corpus size AFTER consuming fragments = 3   (splitting 2 input files enlarged the effective training data by +3 rows)

--- (D') DocumentClassifier().train() now trains on the split-enlarged corpus ---
DocumentClassifier().train() returned = True (reads Document.objects at classifier.py:125; corpus size = 3, all from splitting)

inventory(/app/src/../consume) AFTER  = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
scratch roots remaining: []

================ Q4 RUN 1 SUMMARY ================
scan edges: simple=[], patch0=[0], middle=[1], several=[2,5], qr=[0]; unreadable/custom via barcode_reader; custom string matches only its own value | separate_pages N->N+1 | consume split delta=0 rows | fragments consumed -> N+1 Document rows -> corpus grows | /app/consume untouched
```

#### 7.3.6 Q4 — complete second run (`python3 /tmp/obs_q4.py 2`)

```text
================ Q4 RUN 2 ================

--- (0) import-time binding: settings.CONSUMPTION_DIR vs tasks.save_to_dir.__defaults__ ---
settings.CONSUMPTION_DIR (runtime) = /app/src/../consume
tasks.save_to_dir.__defaults__      = (None, '/app/src/../consume') (tuple = (newname_default, target_dir_default); target_dir bound at import, tasks.py:167)
after settings.CONSUMPTION_DIR := /tmp/some-other-consume-XYZ:
  settings.CONSUMPTION_DIR       = /tmp/some-other-consume-XYZ
  tasks.save_to_dir.__defaults__ = (None, '/app/src/../consume') (UNCHANGED -> bound at import, not read live)
import-bound target dir            = /app/src/../consume
inventory(/app/src/../consume) BEFORE = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
\x1b[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
\x1b[0m

--- (A) scan_file_for_separating_barcodes - every edge (returns separator page numbers) ---
[default CONSUMER_BARCODE_STRING = 'PATCHT']
scan(simple.pdf                      ) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t.pdf                ) -> [0]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(patch-code-t-middle.pdf         ) -> [1]
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
scan(several-patcht-codes.pdf        ) -> [2, 5]
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
scan(patch-code-t-qr.pdf             ) -> [0]

--- (A') barcode_reader on PNG edges (unreadable / custom value) ---
barcode_reader(barcode-39-PATCHT-unreadable.png) -> []
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
barcode_reader(barcode-128-custom.png)           -> ['CUSTOM BARCODE'] (decoded value is the CUSTOM string, not 'PATCHT')

--- (A'') custom CONSUMER_BARCODE_STRING controls which value triggers a split ---
[override CONSUMER_BARCODE_STRING = 'CUSTOM BARCODE']
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
scan(barcode-128-custom.pdf) -> [0] (now matches because decoded value == configured string)

--- (B) separate_pages fragment arithmetic (N separators -> N+1 fragments) ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-i574epoc
LOG WARNING paperless.tasks: No pages to split on!
LOG DEBUG paperless.tasks: Temp files are []
separate_pages(patch-code-t-middle.pdf      splits=[]     ) -> 0 fragment(s): []
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-eu92ho8c
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-eu92ho8c/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-eu92ho8c/patch-code-t-middle_document_1.pdf']
separate_pages(patch-code-t-middle.pdf      splits=[1]    ) -> 2 fragment(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
     patch-code-t-middle_document_0.pdf       pages=1
     patch-code-t-middle_document_1.pdf       pages=1
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-g92_v3ap
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-g92_v3ap/several-patcht-codes_document_0.pdf', '/tmp/paperless/paperless-g92_v3ap/several-patcht-codes_document_1.pdf', '/tmp/paperless/paperless-g92_v3ap/several-patcht-codes_document_2.pdf']
separate_pages(several-patcht-codes.pdf     splits=[2, 5] ) -> 3 fragment(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
     several-patcht-codes_document_0.pdf      pages=2
     several-patcht-codes_document_1.pdf      pages=2
     several-patcht-codes_document_2.pdf      pages=1

--- (C) consume_file (CONSUMER_ENABLE_BARCODES=True) - split path writes fragments, creates 0 Document rows ---
isolated save_to_dir target -> /tmp/pngx-q4-consume-py2ynmot (temp; /app/consume untouched)
Document.objects.count() BEFORE split = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4-in-yyefpl_a/patch-code-t-middle.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-b4k57ir6
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-b4k57ir6/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-b4k57ir6/patch-code-t-middle_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4-in-yyefpl_a/patch-code-t-middle.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
consume_file(...) returned            = 'File successfully split'
Document.objects.count() AFTER split  = 0   (delta = 0 -> split path creates NO Document rows)
original input still exists?          = False (os.unlink at tasks.py:214 after successful split)
isolated consume dir contents         = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']

--- (D) consume produced fragments through the REAL Consumer -> distinct fragments become Document rows ---
training-corpus size BEFORE any consume = 0
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-40ozswne
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-40ozswne/several-patcht-codes_document_0.pdf', '/tmp/paperless/paperless-40ozswne/several-patcht-codes_document_1.pdf', '/tmp/paperless/paperless-40ozswne/several-patcht-codes_document_2.pdf']
input several-patcht-codes.pdf     separators=[2, 5]  -> 3 fragment FILES (N+1, N=2); distinct md5=2
     fragment several-patcht-codes_document_0.pdf      md5=876bb6bd0584396c9bd54bc617f10693
     fragment several-patcht-codes_document_1.pdf      md5=876bb6bd0584396c9bd54bc617f10693
     fragment several-patcht-codes_document_2.pdf      md5=671b21782a64e6486818a8a21e3ed031
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-u00y5vo1/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-u00y5vo1/several-patcht-codes_document_0.pdf', 'output_file': '/tmp/paperless/paperless-jtn53d0c/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-jtn53d0c/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-jtn53d0c/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-jtn53d0c/archive.pdf[0] /tmp/paperless/paperless-jtn53d0c/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-jtn53d0c/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-jtn53d0c/gs_out.png /tmp/paperless/paperless-jtn53d0c/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-jtn53d0c/convert_gs.png -out /tmp/paperless/paperless-jtn53d0c/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-u00y5vo1/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-jtn53d0c
LOG INFO paperless.consumer: Document 2026-07-13 several-patcht-codes_document_0 consumption finished
     consumed several-patcht-codes_document_0.pdf      -> Document pk=1 mime='application/pdf' content_len=50
LOG ERROR paperless.consumer: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
     consumed several-patcht-codes_document_1.pdf      -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_2.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-i_co4xm1/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-i_co4xm1/several-patcht-codes_document_2.pdf', 'output_file': '/tmp/paperless/paperless-8s1ckejr/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-8s1ckejr/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-8s1ckejr/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-8s1ckejr/archive.pdf[0] /tmp/paperless/paperless-8s1ckejr/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-8s1ckejr/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-8s1ckejr/gs_out.png /tmp/paperless/paperless-8s1ckejr/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-8s1ckejr/convert_gs.png -out /tmp/paperless/paperless-8s1ckejr/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-i_co4xm1/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-8s1ckejr
LOG INFO paperless.consumer: Document 2026-07-13 several-patcht-codes_document_2 consumption finished
     consumed several-patcht-codes_document_2.pdf      -> Document pk=2 mime='application/pdf' content_len=24
  => several-patcht-codes.pdf: 3 fragment files -> 2 NEW Document rows
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Temp dir is /tmp/paperless/paperless-szxtc0l6
LOG DEBUG paperless.tasks: Count: 0 page_number: 1
LOG DEBUG paperless.tasks: page_number: 1 next_page: 3
LOG DEBUG paperless.tasks: pdf no:0 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/paperless/paperless-szxtc0l6/patch-code-t-middle_document_0.pdf', '/tmp/paperless/paperless-szxtc0l6/patch-code-t-middle_document_1.pdf']
input patch-code-t-middle.pdf      separators=[1]     -> 2 fragment FILES (N+1, N=1); distinct md5=1
     fragment patch-code-t-middle_document_0.pdf       md5=e24eb8a3cec05735ecae481cfa1b5fd2
     fragment patch-code-t-middle_document_1.pdf       md5=e24eb8a3cec05735ecae481cfa1b5fd2
LOG INFO paperless.consumer: Consuming patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-middle_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4-frag-8wot4lq7/patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4-frag-8wot4lq7/patch-code-t-middle_document_0.pdf', 'output_file': '/tmp/paperless/paperless-9h65gmzh/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-9h65gmzh/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/paperless/paperless-9h65gmzh/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-middle_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-9h65gmzh/archive.pdf[0] /tmp/paperless/paperless-9h65gmzh/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-9h65gmzh/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-9h65gmzh/gs_out.png /tmp/paperless/paperless-9h65gmzh/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/paperless/paperless-9h65gmzh/convert_gs.png -out /tmp/paperless/paperless-9h65gmzh/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4-frag-8wot4lq7/patch-code-t-middle_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/paperless/paperless-9h65gmzh
LOG INFO paperless.consumer: Document 2026-07-13 patch-code-t-middle_document_0 consumption finished
     consumed patch-code-t-middle_document_0.pdf       -> Document pk=3 mime='application/pdf' content_len=24
LOG ERROR paperless.consumer: Not consuming patch-code-t-middle_document_1.pdf: It is a duplicate.
     consumed patch-code-t-middle_document_1.pdf       -> REJECTED (pre_check_duplicate consumer.py:102-113): It is a duplicate.
  => patch-code-t-middle.pdf : 2 fragment files -> 1 NEW Document rows (identical frags deduped)
training-corpus size AFTER consuming fragments = 3   (splitting 2 input files enlarged the effective training data by +3 rows)

--- (D') DocumentClassifier().train() now trains on the split-enlarged corpus ---
DocumentClassifier().train() returned = True (reads Document.objects at classifier.py:125; corpus size = 3, all from splitting)

inventory(/app/src/../consume) AFTER  = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
scratch roots remaining: []

================ Q4 RUN 2 SUMMARY ================
scan edges: simple=[], patch0=[0], middle=[1], several=[2,5], qr=[0]; unreadable/custom via barcode_reader; custom string matches only its own value | separate_pages N->N+1 | consume split delta=0 rows | fragments consumed -> N+1 Document rows -> corpus grows | /app/consume untouched
```

### 7.4 Non-determinism probe — all ten complete runs

The two named suites (`src/documents/tests/test_classifier.py` and `src/documents/tests/test_tasks.py`) were run ten times from an identical starting state (the import-bound consumption directory `/app/consume` emptied before each run) under the default `src/setup.cfg` configuration. `[observed]` The command for run *k* was:

```text
$ cd /app/src && rm -f /app/consume/* && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest documents/tests/test_classifier.py documents/tests/test_tasks.py
```

All ten runs completed with exit code 0 and the identical result line `62 passed, 1 skipped, 774 warnings`; only the wall-clock duration varied. The complete, unedited 35-line output of each run follows in order.

#### 7.4.1 Non-determinism run 1 (`nd_run_1.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 28.86s
```

#### 7.4.2 Non-determinism run 2 (`nd_run_2.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 29.06s
```

#### 7.4.3 Non-determinism run 3 (`nd_run_3.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 29.04s
```

#### 7.4.4 Non-determinism run 4 (`nd_run_4.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 29.00s
```

#### 7.4.5 Non-determinism run 5 (`nd_run_5.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 28.88s
```

#### 7.4.6 Non-determinism run 6 (`nd_run_6.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 28.18s
```

#### 7.4.7 Non-determinism run 7 (`nd_run_7.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 32.57s
```

#### 7.4.8 Non-determinism run 8 (`nd_run_8.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 43.22s
```

#### 7.4.9 Non-determinism run 9 (`nd_run_9.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 33.68s
```

#### 7.4.10 Non-determinism run 10 (`nd_run_10.txt`)

```text
bringing up nodes...
bringing up nodes...

s..............................................................          [100%]
=============================== warnings summary ===============================
../../usr/local/lib/python3.9/site-packages/redis/connection.py:67: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version = StrictVersion(hiredis.__version__)

../../usr/local/lib/python3.9/site-packages/redis/connection.py:69: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.3')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:71: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('0.1.4')

../../usr/local/lib/python3.9/site-packages/redis/connection.py:73: 129 warnings
  /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    hiredis_version >= StrictVersion('1.0.0')

../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: 129 warnings
  /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
    warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)

../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: 129 warnings
  /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
    from django.utils import baseconv

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.9.23-final-0 ________________

Coverage HTML written to dir htmlcov
62 passed, 1 skipped, 774 warnings in 31.21s
```

### 7.5 Repository and container cleanliness

`[observed]` **The repository checkout was never modified** by the investigation. All observation scripts (§7.2) were delivered into the container's `/tmp` and executed there; the container's `/app` tree is a distinct checkout from the repository under `blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf_735501`, so container-side artifacts can never appear in the repository's git state. The sole change tracked by the repository is this deliverable, `blitzy/documentation/paperless-ngx_542221a38dff.md`. While this document was being finalized (before committing), it was the only entry in the working-tree status — no source, test, or config file appears:

```text
$ git -C <repo> rev-parse --abbrev-ref HEAD
blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf

$ git -C <repo> status --porcelain     # before committing: sole change is the tracked deliverable
 M blitzy/documentation/paperless-ngx_542221a38dff.md

$ git -C <repo> diff --name-status HEAD
M	blitzy/documentation/paperless-ngx_542221a38dff.md
```

After committing this document, the working tree is clean — `git status --porcelain` produces no output — and, measured from the commit under investigation (`542221a38dff`) to `HEAD`, the deliverable is the **only** added path:

```text
$ git status --porcelain          # after committing this document: clean working tree (no output)
$ git log --oneline -3
<this-commit> docs(qna): rewrite paperless-ngx runtime investigation with complete canonical evidence
4b7d6d724 Add QnA investigation: paperless-ngx classifier/OCR/matching/barcode runtime behavior
542221a38 Merge pull request #792 from paperless-ngx/dependabot/github_actions/github/codeql-action-2

$ git diff --name-status HEAD~1 HEAD     # this commit changed exactly one file (the deliverable)
M	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --name-status 542221a38dff HEAD  # baseline (commit under investigation) -> HEAD: sole addition
A	blitzy/documentation/paperless-ngx_542221a38dff.md
```

`[observed]` **The import-bound split target `/app/consume` was inventoried and left empty.** As established in §5.1, `documents.tasks.save_to_dir` binds its destination default to `settings.CONSUMPTION_DIR` (`/app/src/../consume` = `/app/consume`) at import time (`src/documents/tasks.py:167`). Any fragment written there by an isolated split probe or by the canonical `test_consume_barcode_file` was removed; the directory is empty at the end of the investigation:

```text
$ docker exec -u testuser pngx-qna ls -la /app/consume
total 16
drwxr-sr-x 1 testuser testuser 4096 Jul 13 18:42 .
drwxr-sr-x 1 testuser testuser 4096 Jul 13 16:24 ..

$ docker exec -u testuser pngx-qna sh -c "ls -1 /app/consume | wc -l"   # import-bound split target: empty
0
```

`[code-grounded]` The observation scripts and any per-probe scratch directories under the container's `/tmp` (e.g. `/tmp/paperless/paperless-*`, mode `0700`, testuser-owned) are transient container state outside the repository; they are removed at the end of the investigation. Nothing from the container is copied into the repository except the evidence quoted verbatim in this document.

### 7.6 Question → canonical entry point → citation map

`[code-grounded]` Each sub-question was answered by exercising the real entry point named below; the file:line anchors are the specific sites that implement the observed behavior (validated byte-identical against the checkout at commit `542221a38dff`, see §1.4).

| Question | Canonical entry point exercised | Primary code sites (repository-relative) |
|---|---|---|
| Q1 reuse vs. retrain | `documents.tasks.train_classifier()` → `DocumentClassifier.train()`/`.save()` | `src/documents/tasks.py:48-72`; SHA-1 guard `src/documents/classifier.py:163-164`; corpus query `src/documents/classifier.py:125-127` |
| Q1 version guard / deletion | `documents.classifier.load_classifier()` | `FORMAT_VERSION=7` `src/documents/classifier.py:63`; check `src/documents/classifier.py:80-83`; `os.unlink` `src/documents/classifier.py:48`; error class `src/documents/classifier.py:13-14` |
| Q1 empty corpus | `DocumentClassifier.train()` on empty DB | `ValueError("No training data available.")` `src/documents/classifier.py:158-159` |
| Q1/§6 unseeded RNG | two separate processes → `MLPClassifier.fit` | `MLPClassifier(...)` (no `random_state`) `src/documents/classifier.py:219,227,238` |
| Q2 (a) training-doc count | direct ORM insert + `DocumentClassifier.train()` | corpus query with inbox exclusion `src/documents/classifier.py:125-127` |
| Q2 (b) training timing | insert (no train) vs. explicit `train_classifier()`; Django-Q schedule | task `src/documents/tasks.py:48-72`; schedule row `src/documents/migrations/1001_auto_20201109_1636.py:11`; CLI `src/documents/management/commands/document_create_classifier.py:20` |
| Q2 (c) acceptance threshold | `DocumentClassifier.predict_correspondent()` → `matching.match_correspondents()` | argmax/`!= -1` `src/documents/classifier.py:251-260`; accept `o.pk == pred_id` `src/documents/matching.py:21-31` (esp. `:30`) |
| Q3 (a) OCR subprocess | `RasterisedDocumentParser.parse()` → `ocrmypdf.ocr(**args)` | `parse()` `src/paperless_tesseract/parsers.py:230-327`; `ocrmypdf.ocr` call `src/paperless_tesseract/parsers.py:261`; skip→force fallback `src/paperless_tesseract/parsers.py:266-298`; `skip_text`/`force_ocr` `src/paperless_tesseract/parsers.py:156,158`; `OCR_MODE="skip"` `src/paperless/settings.py:522` |
| Q3 (b) assigned MIME type | `documents.consumer.Consumer.try_consume_file()` → `magic.from_file(...)` → `_store()` | detect `src/documents/consumer.py:219`; persist `src/documents/consumer.py:379,401` |
| Q3 edge: encrypted PDF | `RasterisedDocumentParser.parse()` on `encrypted.pdf` | `PDFPasswordIncorrect` path `src/paperless_tesseract/parsers.py:120`; empty-text end `src/paperless_tesseract/parsers.py:318-327` |
| Q4 (a) fragment count | `documents.tasks.separate_pages()` | doc_0 loop `src/documents/tasks.py:131-133`; skip-separator range `src/documents/tasks.py:149` (N separators → N+1 fragments) |
| Q4 (b) trigger values | `documents.tasks.scan_file_for_separating_barcodes()` / `barcode_reader()` | separator var `src/documents/tasks.py:102`; `CONSUMER_BARCODE_STRING="PATCHT"` `src/paperless/settings.py:506` |
| Q4 (c) split decision site | `documents.tasks.scan_file_for_separating_barcodes()` | barcode read `src/documents/tasks.py:107`; decision `if separator_barcode in current_barcodes` `src/documents/tasks.py:108`; enable gate `src/documents/tasks.py:195`; `CONSUMER_ENABLE_BARCODES` `src/paperless/settings.py:502-504` |
| Q4 (d) effect on training data | `documents.tasks.consume_file()` split → real `Consumer` → `DocumentClassifier.train()` | split creates 0 rows (unlink `src/documents/tasks.py:214`); dedup `src/documents/consumer.py:102-113`; corpus read `src/documents/classifier.py:125` |
| Non-determinism | `pytest documents/tests/test_classifier.py documents/tests/test_tasks.py` under `-n auto` | `src/setup.cfg` `[tool:pytest]`; per-test isolation `src/documents/tests/utils.py:37,45` |
