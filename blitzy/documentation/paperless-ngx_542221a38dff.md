# paperless-ngx — ML + OCR Document-Processing Pipeline: Runtime Behavioral Investigation

**Subject:** The paperless-ngx machine-learning (document classifier) and OCR document-processing pipeline, as it behaves during automated test execution.
**Commit under investigation:** `542221a38dff06361e07976452f9aea24d210542` (git `HEAD`, source tree treated as strictly read-only).
**Purpose:** Explain, from **directly observed runtime behavior**, the mechanics behind **non-deterministic failures reported in the document-classification test suite** — by answering four specific questions (Q1–Q4), each grounded in captured, unedited command output with `file:line` citations.

This document leads every answer with the **direct result**, follows with **cause→effect reasoning** naming the exact function/method and `file:line`, and pairs each behavioral claim with the **exact command** and its **complete, unedited output**. Statements derived from reading code rather than observing a run are explicitly labelled **(inferred)**.

---

## Canonical Environment & Methodology

All runtime evidence was captured inside the user-supplied canonical Docker container, which carries the pinned Python runtime, the pinned ML/PDF Python packages, and every OCR/PDF/barcode system binary. The general host sandbox is **not** canonical (it runs Python 3.13 and lacks `django`, `scikit-learn`, `ocrmypdf`, `magic`, `pdf2image`, `pikepdf`, `pyzbar`, and the `tesseract`/`gs`/`unpaper`/`pdftoppm`/`libzbar`/`convert`/`qpdf` binaries), so **no** runtime evidence was taken there.

### Exact container / image provenance

The long-lived container used for every run below is named `paperless-canon`. Its authoritative base image is the full GHCR image; the setup step then installed a few missing OS packages **inside** that running container (`libzbar0` — required for `pyzbar` import, `poppler-utils`, `tesseract-ocr-{deu,fra,ita,spa}`, `pngquant`). The same additions are also baked into a derived image `paperless-ngx-canon:ready`. The exact identifiers:

```console
$ docker ps --filter name=paperless-canon --format '{{.Names}}  image={{.Image}}  container_id={{.ID}}'
paperless-canon  image=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01  container_id=7ed3c09d1a4c

$ docker images --no-trunc --format '{{.Repository}}:{{.Tag}}  {{.ID}}' | grep -E 'swe-atlas|paperless-ngx-canon'
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01  sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f
paperless-ngx-canon:ready  sha256:10595af76b432461efac82fa34f5c3a9fdec6b5a0f62c12f0add0cce75af570f
```

So the running container is the **base GHCR image `…swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` (id `sha256:6e699f225ced…`)** with the setup-time apt additions applied inside it — not plainly the `paperless-ngx-canon:ready` image.

### Commit under investigation, Python, and pristine source tree

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app && git rev-parse HEAD; git status --porcelain | wc -l; python3 --version'
542221a38dff06361e07976452f9aea24d210542
0
Python 3.9.23
```

`git rev-parse HEAD` confirms the exact commit `542221a38dff06361e07976452f9aea24d210542`; the `git status --porcelain` line count of `0` confirms the `/app` source checkout is byte-for-byte pristine at the start of the investigation. Python is **3.9.23** — the version the primary CI job pins (matrix `['3.8', '3.9', '3.10']` at `.github/workflows/reusable-ci-backend.yml:L55`).

### Pinned Python package versions (system-wide, canonical)

```console
$ docker exec --user testuser paperless-canon bash -lc "python3 -m pip show django scikit-learn ocrmypdf python-magic pdf2image pikepdf pyzbar pytest pytest-xdist pytest-django 2>/dev/null | grep -E '^(Name|Version):' | paste - - | sed 's/Name: //; s/\tVersion:/ ==/'"
Django == 4.0.4
scikit-learn == 1.0.2
ocrmypdf == 13.4.3
python-magic == 0.4.25
pdf2image == 1.16.0
pikepdf == 5.1.1
pyzbar == 0.1.9
pytest == 8.4.2
pytest-xdist == 3.8.0
pytest-django == 4.11.1
```

### System binaries

```console
$ docker exec --user testuser paperless-canon bash -lc 'tesseract --version 2>&1 | head -1; echo gs $(gs --version); unpaper --version; pdftoppm -v 2>&1 | head -1; convert --version | head -1; qpdf --version | head -1; dpkg -l libzbar0 | awk "/libzbar0/{print \$2, \$3}"'
tesseract 4.1.1
gs 9.53.3
6.1
pdftoppm version 20.09.0
Version: ImageMagick 6.9.11-60 Q16 x86_64 2021-01-25 https://imagemagick.org
qpdf version 10.1.0
libzbar0:amd64 0.23.90-1+deb11u1
```

### pytest configuration and CI command

```console
$ docker exec --user testuser paperless-canon bash -lc 'sed -n "8,12p" /app/src/setup.cfg'
[tool:pytest]
DJANGO_SETTINGS_MODULE=paperless.settings
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet
env =
  PAPERLESS_DISABLE_DBHANDLER=true

$ docker exec --user testuser paperless-canon bash -lc "grep -nE 'python-version:|pipenv run pytest' /app/.github/workflows/reusable-ci-backend.yml"
55:        python-version: ['3.8', '3.9', '3.10']
70:          python-version: "${{ matrix.python-version }}"
86:          pipenv run pytest
```

`src/setup.cfg [tool:pytest]` pins `DJANGO_SETTINGS_MODULE=paperless.settings` (`src/setup.cfg:L9`), `addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet` (`src/setup.cfg:L10`), and `env: PAPERLESS_DISABLE_DBHANDLER=true` (`src/setup.cfg:L11-L12`). The CI command is `pipenv run pytest` from `src/` (`.github/workflows/reusable-ci-backend.yml:L86`).

### Canonical invocation used for every run below

The pinned dependencies are installed **system-wide** in this container (there is no `pipenv`/virtualenv layer), so the equivalent canonical command is `python3 -m pytest` — it runs the identical test session with the identical `src/setup.cfg` `addopts` (including pytest-xdist `--numprocesses auto`). Every run below is executed as the non-root `testuser` (running as root makes four permission-sensitive tests fail) with `HOME=/tmp/th` created at mode **0700**. The 0700 permission matters: paperless sets `GNUPG_HOME = os.getenv("HOME", "/tmp")` (`src/paperless/settings.py:L544`), so a group/world-readable HOME makes every gpg invocation emit `gpg: WARNING: unsafe permissions on homedir`. Demonstrated contrast:

```console
$ docker exec --user testuser paperless-canon bash -lc 'mkdir -p /tmp/th755 && chmod 0755 /tmp/th755 && gpg --homedir /tmp/th755 --list-keys 2>&1 | grep -i unsafe; rm -rf /tmp/th755'
gpg: WARNING: unsafe permissions on homedir '/tmp/th755'

$ docker exec --user testuser paperless-canon bash -lc 'chmod 0700 /tmp/th && gpg --homedir /tmp/th --list-keys 2>&1 | grep -ic unsafe'
0
```

With `HOME=/tmp/th` at 0700 (used throughout), no `unsafe permissions` warning is emitted, so none appears in the raw output blocks below. The invocation template (concrete targets are substituted per run — e.g. a test node id or a probe path shown verbatim in each section):

```console
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc \
  'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest documents/tests/test_tasks.py::TestTasks::test_train_classifier -o addopts="" -p no:cacheprovider'
```

**Run flags used and why.** For *targeted* observation of a single behaviour I pass `-o addopts=""` (disables `--cov`/`--numprocesses auto` for a fast, single-process run) and `-p no:cacheprovider` (prevents pytest cache writes). When investigating the **non-determinism** I keep the default `--numprocesses auto` in play (adding only `--no-cov` to avoid coverage HTML artifacts) and, to *characterize* the effect, additionally pin `-n0` (serial) — every non-default run is labelled as such.

**Probe discipline.** Every temporary observation script below is shown **in full** (its complete source is embedded next to the command that runs it) so any reviewer can reproduce it independently; each was written under `/tmp/th/work/` (a 0700 scratch directory, outside both git trees) and **deleted** after the investigation (see the closing Read-Only Invariant appendix). No probe used a debug hook, mock, or synthetic stand-in for the behaviour under test except where explicitly labelled non-canonical. For any input that is passed to code that mutates its input in place (the Q3 OCR path), the tracked repository fixture is **copied to a unique scratch path first** and its source checksum is verified unchanged before/after — the tracked fixture is never handed to mutating code.

---

## Q1 — Does the document classifier reuse an existing model or retrain during the same test run, and how does that affect later tests?

**Direct answer.** Within a single run the classifier **reuses** the already-persisted model whenever the training data is unchanged, and **retrains** (rewriting the model file) whenever the training data changes. The decision is made by `DocumentClassifier.train()` comparing a **SHA-1 digest** of the training data against the stored digest: on a match it **short-circuits and returns `False`** (reuse, model file untouched); otherwise it returns `True` and the caller re-persists the pickle. **Effect on later tests:** because the persisted model is a *filesystem* artifact, if a test's `MODEL_FILE` is **not** isolated per-test, one test's trained model **persists into later tests** — under pytest-xdist parallel workers this shared state is the plausible root cause of run-to-run classification failures (fully analyzed in the closing section).

### Cause → effect (mechanism)

- `DocumentClassifier.train()` (`src/documents/classifier.py:L115`) builds a SHA-1 digest — `m = hashlib.sha1()` (`src/documents/classifier.py:L124`) — over the preprocessed content and label ids of every `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:L125-L127`), finalising `new_data_hash = m.digest()` (`src/documents/classifier.py:L161`).
- The **reuse short-circuit** is `if self.data_hash and new_data_hash == self.data_hash: return False` (`src/documents/classifier.py:L163-L164`). When the data differs it proceeds, sets `self.data_hash = new_data_hash` (`src/documents/classifier.py:L247`) and `return True` (`src/documents/classifier.py:L249`) — a **retrain**. Persisted pickles carry `FORMAT_VERSION = 7` (`src/documents/classifier.py:L63`).
- The task wrapper `train_classifier()` (`src/documents/tasks.py:L48-L72`) loads the classifier via `load_classifier()` (`src/documents/tasks.py:L57`; returns `None` when `MODEL_FILE` is absent — `src/documents/classifier.py:L30-L37`) and **saves the pickle only when `train()` returns `True`** (`src/documents/tasks.py:L63`), logging `"Saving updated classifier model to {}..."` (`src/documents/tasks.py:L64`) then `classifier.save()` (`src/documents/tasks.py:L67`); otherwise it logs **`"Training data unchanged."`** (`src/documents/tasks.py:L69`).

### Observed evidence — reuse/retrain log sequence (canonical test `test_train_classifier`)

`test_train_classifier` (`src/documents/tests/test_tasks.py:L75-L94`) asserts the model file is absent, then trains three times: create → `train_classifier()` (retrain, file created), a second identical `train_classifier()` (reuse, `assertEqual(mtime, mtime2)`), then edits `doc.content="test2"` and trains again (retrain, `assertNotEqual(mtime2, mtime3)`). Running it with DEBUG log capture and filtering (transparently, via `grep`) to the two `paperless.tasks` log lines that report the branch taken:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "documents/tests/test_tasks.py::TestTasks::test_train_classifier" -o addopts="" -p no:cacheprovider -o log_cli=true --log-cli-level=DEBUG -s 2>&1 | grep -E "^(INFO|DEBUG) +paperless.tasks:tasks.py:(64|69)"'
INFO     paperless.tasks:tasks.py:64 Saving updated classifier model to /tmp/tmpvsblk2j2/classification_model.pickle...
DEBUG    paperless.tasks:tasks.py:69 Training data unchanged.
INFO     paperless.tasks:tasks.py:64 Saving updated classifier model to /tmp/tmpvsblk2j2/classification_model.pickle...
```

Reading the three verbatim lines in order: the **first** call logs `Saving updated classifier model to …` from `src/documents/tasks.py:L64` (retrain branch — `train()` returned `True`, file created); the **second** call logs `Training data unchanged.` from `src/documents/tasks.py:L69` (reuse branch — `train()` returned `False`, model file untouched); the **third** call, after the content edit, logs `Saving updated classifier model to …` again from `src/documents/tasks.py:L64` (retrain). The log-cli line tag `tasks.py:64` / `tasks.py:69` (shown verbatim in the block above) is emitted by pytest itself and directly names lines in `src/documents/tasks.py`. The full test passes on two identical runs (complete session tail + exit status for each):

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "documents/tests/test_tasks.py::TestTasks::test_train_classifier" -o addopts="" -p no:cacheprovider -q 2>&1 | tail -1'; echo "run1 exit=${PIPESTATUS[0]}"
1 passed, 6 warnings in 2.09s
run1 exit=0
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "documents/tests/test_tasks.py::TestTasks::test_train_classifier" -o addopts="" -p no:cacheprovider -q 2>&1 | tail -1'; echo "run2 exit=${PIPESTATUS[0]}"
1 passed, 6 warnings in 1.99s
run2 exit=0
```

`MODEL_FILE` resolved to a **per-test** temp path (`/tmp/tmpvsblk2j2/classification_model.pickle`) — this is the `DirectoriesMixin` isolation discussed under "cross-test effect" below.

### Observed evidence — direct mtime + `train()` boolean (self-contained probe)

To observe the `MODEL_FILE` state **before / during / after** and the raw boolean returned by `train()`, the following self-contained probe drives the **real** `train_classifier()` task and the **real** `DocumentClassifier.train()`/`load_classifier()` under `DirectoriesMixin` (no mocks or hooks). Its complete source is embedded here so the run is independently reproducible; the file itself was written under the 0700 scratch dir `/tmp/th/work/` and deleted afterward (Read-Only Invariant appendix):

```python
# /tmp/th/work/blitzy_adhoc_test_q1.py
import os

from django.conf import settings
from django.test import TestCase

from documents import tasks
from documents.classifier import DocumentClassifier, load_classifier
from documents.models import Correspondent, Document, Tag
from documents.tests.utils import DirectoriesMixin


class Q1Probe(DirectoriesMixin, TestCase):
    def test_q1_reuse_vs_retrain(self):
        c = Correspondent.objects.create(matching_algorithm=Tag.MATCH_AUTO, name="test")
        doc = Document.objects.create(correspondent=c, content="test", title="test")

        print("Q1_OBS MODEL_FILE =", settings.MODEL_FILE)
        print("Q1_OBS model exists BEFORE any train :", os.path.isfile(settings.MODEL_FILE))

        # call 1: first train -> RETRAIN, model file created
        tasks.train_classifier()
        m1 = os.stat(settings.MODEL_FILE).st_mtime_ns
        print("Q1_OBS call1 train_classifier() model exists :", os.path.isfile(settings.MODEL_FILE), "mtime_ns =", m1)

        # call 2: identical data -> REUSE, mtime unchanged
        tasks.train_classifier()
        m2 = os.stat(settings.MODEL_FILE).st_mtime_ns
        print("Q1_OBS call2 train_classifier() identical  mtime_ns =", m2, "mtime_unchanged_vs_call1 =", m1 == m2)

        # call 3: change content -> RETRAIN, mtime changes
        doc.content = "test2"
        doc.save()
        tasks.train_classifier()
        m3 = os.stat(settings.MODEL_FILE).st_mtime_ns
        print("Q1_OBS call3 train_classifier() edited     mtime_ns =", m3, "mtime_changed_vs_call2 =", m2 != m3)

        # direct DocumentClassifier.train() boolean: True (retrain) then False (reuse)
        clf = DocumentClassifier()
        r1 = clf.train()
        r2 = clf.train()
        print("Q1_OBS direct DocumentClassifier.train() first  returned =", r1)
        print("Q1_OBS direct DocumentClassifier.train() second returned =", r2)

        # persisted reuse: a freshly loaded classifier reuses via the persisted data_hash
        loaded = load_classifier()
        r3 = loaded.train()
        print("Q1_OBS persisted load_classifier().train() returned =", r3)
```

Run #1 — complete, unedited session output including the (6 pre-existing environment) warnings and the exit status:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q1.py -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 1 item

../../tmp/th/work/blitzy_adhoc_test_q1.py Q1_OBS MODEL_FILE = /tmp/tmpit924xp2/classification_model.pickle
Q1_OBS model exists BEFORE any train : False
[2026-07-14 20:53:19,339] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpit924xp2/classification_model.pickle...
Q1_OBS call1 train_classifier() model exists : True mtime_ns = 1784062399338729461
Q1_OBS call2 train_classifier() identical  mtime_ns = 1784062399338729461 mtime_unchanged_vs_call1 = True
[2026-07-14 20:53:19,362] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpit924xp2/classification_model.pickle...
Q1_OBS call3 train_classifier() edited     mtime_ns = 1784062399361729708 mtime_changed_vs_call2 = True
Q1_OBS direct DocumentClassifier.train() first  returned = True
Q1_OBS direct DocumentClassifier.train() second returned = False
Q1_OBS persisted load_classifier().train() returned = False
.

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
======================== 1 passed, 6 warnings in 2.02s =========================
EXIT=0
```

Run #2 — identical inputs, complete unedited output. The relationships reproduce exactly (`call1 == call2` mtime; `call2 != call3`; direct `train()` returns `True` then `False`; persisted reuse `False`); only the absolute mtimes and the per-test temp path differ because `DirectoriesMixin` allocates a fresh temp dir each run:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q1.py -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 1 item

../../tmp/th/work/blitzy_adhoc_test_q1.py Q1_OBS MODEL_FILE = /tmp/tmpoup4atil/classification_model.pickle
Q1_OBS model exists BEFORE any train : False
[2026-07-14 20:53:43,494] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpoup4atil/classification_model.pickle...
Q1_OBS call1 train_classifier() model exists : True mtime_ns = 1784062423493989687
Q1_OBS call2 train_classifier() identical  mtime_ns = 1784062423493989687 mtime_unchanged_vs_call1 = True
[2026-07-14 20:53:43,513] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpoup4atil/classification_model.pickle...
Q1_OBS call3 train_classifier() edited     mtime_ns = 1784062423512989891 mtime_changed_vs_call2 = True
Q1_OBS direct DocumentClassifier.train() first  returned = True
Q1_OBS direct DocumentClassifier.train() second returned = False
Q1_OBS persisted load_classifier().train() returned = False
.

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
======================== 1 passed, 6 warnings in 2.03s =========================
EXIT=0
```

Reading either run: **reuse** = the direct `train()` second call returns `False`, `call2` mtime equals `call1`, and the persisted `load_classifier().train()` returns `False`; **retrain** = the direct `train()` first call returns `True`, `call1` creates the file, and `call3` (after the content edit) advances the mtime. Both branches are exercised on identical inputs across the two runs.

### Observed evidence — hashing / save-reuse canonical tests

`testDatasetHashing` (`src/documents/tests/test_classifier.py:L137-L142`) asserts `assertTrue(train())` then `assertFalse(train())` on identical data (retrain then reuse). `testSaveClassifier` (`src/documents/tests/test_classifier.py:L167-L178`; decorated `@override_settings(DATA_DIR=tempfile.mkdtemp())` at `src/documents/tests/test_classifier.py:L167` — it pins `DATA_DIR`, **not** `MODEL_FILE`) trains, saves, loads a fresh classifier, and `assertFalse(new_classifier.train())` (reuse via the persisted `data_hash`). `test_load_and_classify` (`src/documents/tests/test_classifier.py:L183`) is the test actually decorated `@override_settings(MODEL_FILE=…/data/model.pickle)` (`src/documents/tests/test_classifier.py:L180-L182`). All three pass:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "documents/tests/test_classifier.py::TestClassifier::testDatasetHashing" "documents/tests/test_classifier.py::TestClassifier::testSaveClassifier" "documents/tests/test_classifier.py::TestClassifier::test_load_and_classify" -o addopts="" -p no:cacheprovider -v -p no:sugar'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 3 items

documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED [ 33%]
documents/tests/test_classifier.py::TestClassifier::testSaveClassifier PASSED [ 66%]
documents/tests/test_classifier.py::TestClassifier::test_load_and_classify PASSED [100%]

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
======================== 3 passed, 6 warnings in 3.10s =========================
EXIT=0
```

### Cross-test effect (how reuse/retrain affects later tests)

`DirectoriesMixin` (`src/documents/tests/utils.py`, class at `src/documents/tests/utils.py:L72`; `setUp` → `setup_directories()` at `src/documents/tests/utils.py:L77`) overrides per-test directories and sets `MODEL_FILE=<data_dir>/classification_model.pickle` inside its `override_settings(...)` block (`src/documents/tests/utils.py:L45`). In every mixin-using run above, `MODEL_FILE` resolved to a unique per-test temporary path — the three Q1 runs shown above resolved it to `/tmp/tmpvsblk2j2/`, `/tmp/tmpit924xp2/`, and `/tmp/tmpoup4atil/` respectively (each ending in `classification_model.pickle`) — so **isolation holds only for tests that use the mixin**. Tests that rely on the *default* `MODEL_FILE` (`src/paperless/settings.py:L74`) or pin a fixed path can **share** model state on disk. Because a trained pickle is a filesystem artifact (unlike DB rows, which `TestCase` rolls back), a model written by one test can be **loaded by a later test** — the reuse/retrain outcome of the later test then depends on what earlier tests left behind **(inferred** from the mechanism; the concrete leakage is demonstrated, and the broader parallel-worker hypothesis analyzed, in the closing Determinism section**)**. Under the default `--numprocesses auto` this is the determinism-risk surface examined at the end of this document.

---

## Q2 — In the test exercising automatic correspondent matching: (a) how many training documents are created, (b) when does training occur relative to those inserts, and (c) what confidence threshold accepts/rejects a prediction?

**Direct answer.**
- **(a) 3 documents are created, but only 2 are used for training.** The training query excludes inbox-tagged documents, and exactly one of the three created documents carries an inbox tag, so it is dropped (3 created → **2 trained**). Observed below: `total Document rows CREATED : 3`, `rows carrying an is_inbox_tag tag : 1`, `training docs = exclude(inbox) COUNT : 2`.
- **(b) Training occurs *after* the inserts.** The documents are `Document.objects.create()`-d first; `DocumentClassifier.train()` is invoked afterward, then prediction runs. Observed below: `rows inserted BEFORE any train() call : 2`, then `self.classifier.train() ran AFTER, ret: True`.
- **(c) There is NO probabilistic confidence threshold.** A correspondent prediction is accepted purely when the predicted class id is not `-1` (the null class); otherwise it returns `None`. No numeric probability cutoff exists anywhere in the correspondent-prediction path. The only `>= 90` in the `documents/` package is an unrelated rule-based fuzzy *string*-matching threshold, not an ML confidence value.

### The training query and the acceptance rule (mechanism)

**Cause → effect (a).** The training set is `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:L125-L127`) — every document *except* those bearing a tag whose `is_inbox_tag` is `True`. The correspondent-matching test data is built by `generate_test_data` (`src/documents/tests/test_classifier.py:L25-L82`), which creates `doc1`, `doc2`, and `doc_inbox`; `doc_inbox` receives tag `t2`, whose `is_inbox_tag=True` (`src/documents/tests/test_classifier.py:L44`), so it is excluded from training — leaving 2 of the 3 created documents.

**Cause → effect (c).** `predict_correspondent` (`src/documents/classifier.py:L251-L260`) transforms the content, obtains a predicted class id from the MLP classifier, and applies exactly one rule — `if correspondent_id != -1: return correspondent_id else: return None` (`src/documents/classifier.py:L255-L258`). Acceptance is purely *class id `!= -1`*; the `-1` null class is rejected as `None`. There is no probability comparison in that path.

### Self-contained probe (embedded source)

To observe the training-document **count**, the **ordering** of inserts vs. `train()`, and the accepted-vs-rejected acceptance rule through the real code, the following probe subclasses the real `TestClassifier`, so it reuses the authentic `generate_test_data()`, the real `setUp()` (which constructs a real `DocumentClassifier()`), and the real `DirectoriesMixin` per-test isolation. It calls the real `DocumentClassifier.train()` and `predict_correspondent()` (no mocks or hooks). Its complete source is embedded here so the run is independently reproducible; the file was written under the 0700 scratch dir `/tmp/th/work/` and deleted afterward (Read-Only Invariant appendix):

```python
# /tmp/th/work/blitzy_adhoc_test_q2.py
# Q2 observation probe (temporary; deleted after use).
# Subclasses the REAL TestClassifier so it reuses the authentic
# generate_test_data(), setUp() (real DocumentClassifier()), and
# DirectoriesMixin per-test isolation -- i.e. canonical code paths.
from documents.models import Correspondent, Document
from documents.tests.test_classifier import TestClassifier


class Q2Probe(TestClassifier):
    def test_q2_a_training_document_count(self):
        # REAL data builder used by the correspondent-matching tests.
        self.generate_test_data()
        total_created = Document.objects.count()
        # Exact training query from classifier.py:L125-L127.
        training_qs = Document.objects.order_by("pk").exclude(
            tags__is_inbox_tag=True,
        )
        inbox_excluded = Document.objects.filter(
            tags__is_inbox_tag=True,
        ).count()
        print("===== Q2(a) TRAINING DOCUMENT COUNT =====")
        print(f"(a) total Document rows CREATED           : {total_created}")
        print(f"(a) rows carrying an is_inbox_tag tag     : {inbox_excluded}")
        print(f"(a) training docs = exclude(inbox) COUNT  : {training_qs.count()}")
        print(f"(a) training doc titles                   : {[d.title for d in training_qs]}")

    def test_q2_bc_timing_and_acceptance(self):
        # Faithful replica of test_one_correspondent_predict_manydocs ordering:
        # create() the rows FIRST, then train(), then predict().
        c1 = Correspondent.objects.create(
            name="c1",
            matching_algorithm=Correspondent.MATCH_AUTO,
        )
        doc1 = Document.objects.create(
            title="doc1",
            content="this is a document from c1",
            correspondent=c1,
            checksum="A",
        )
        doc2 = Document.objects.create(
            title="doc2",
            content="this is a document from noone",
            checksum="B",
        )
        rows_before_training = Document.objects.count()
        print("===== Q2(b)/(c) ORDERING + ACCEPTANCE RULE =====")
        print(f"(b) rows inserted BEFORE any train() call : {rows_before_training}")
        # REAL train() AFTER the inserts.
        trained = self.classifier.train()
        print(f"(b) self.classifier.train() ran AFTER, ret: {trained}")
        # REAL predict_correspondent -> id != -1 accept rule.
        p1 = self.classifier.predict_correspondent(doc1.content)
        p2 = self.classifier.predict_correspondent(doc2.content)
        print(f"(c) predict_correspondent(doc1)           : {p1} | c1.pk={c1.pk} | ACCEPTED(id!=-1)={p1 == c1.pk}")
        print(f"(c) predict_correspondent(doc2)           : {p2} | REJECTED(None,i.e. class id==-1)={p2 is None}")
```

### Observed evidence — count, timing, and acceptance rule (probe, two identical runs)

Selecting only the two probe methods by node id (so only the observation methods run, not the parent class's inherited tests). Run #1:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "/tmp/th/work/blitzy_adhoc_test_q2.py::Q2Probe::test_q2_a_training_document_count" "/tmp/th/work/blitzy_adhoc_test_q2.py::Q2Probe::test_q2_bc_timing_and_acceptance" -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 2 items

../../tmp/th/work/blitzy_adhoc_test_q2.py ===== Q2(a) TRAINING DOCUMENT COUNT =====
(a) total Document rows CREATED           : 3
(a) rows carrying an is_inbox_tag tag     : 1
(a) training docs = exclude(inbox) COUNT  : 2
(a) training doc titles                   : ['doc1', 'doc1']
.===== Q2(b)/(c) ORDERING + ACCEPTANCE RULE =====
(b) rows inserted BEFORE any train() call : 2
(b) self.classifier.train() ran AFTER, ret: True
(c) predict_correspondent(doc1)           : [1] | c1.pk=1 | ACCEPTED(id!=-1)=[ True]
(c) predict_correspondent(doc2)           : None | REJECTED(None,i.e. class id==-1)=True
.

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
======================== 2 passed, 6 warnings in 1.92s =========================
EXIT=0
```

Run #2 (same command, same unchanged input):

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "/tmp/th/work/blitzy_adhoc_test_q2.py::Q2Probe::test_q2_a_training_document_count" "/tmp/th/work/blitzy_adhoc_test_q2.py::Q2Probe::test_q2_bc_timing_and_acceptance" -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 2 items

../../tmp/th/work/blitzy_adhoc_test_q2.py ===== Q2(a) TRAINING DOCUMENT COUNT =====
(a) total Document rows CREATED           : 3
(a) rows carrying an is_inbox_tag tag     : 1
(a) training docs = exclude(inbox) COUNT  : 2
(a) training doc titles                   : ['doc1', 'doc1']
.===== Q2(b)/(c) ORDERING + ACCEPTANCE RULE =====
(b) rows inserted BEFORE any train() call : 2
(b) self.classifier.train() ran AFTER, ret: True
(c) predict_correspondent(doc1)           : [1] | c1.pk=1 | ACCEPTED(id!=-1)=[ True]
(c) predict_correspondent(doc2)           : None | REJECTED(None,i.e. class id==-1)=True
.

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
======================== 2 passed, 6 warnings in 1.86s =========================
EXIT=0
```

Both runs exit `0` and their observation lines are byte-identical, so the reported values are stable: **(a)** `total Document rows CREATED : 3`, `rows carrying an is_inbox_tag tag : 1`, `training docs = exclude(inbox) COUNT : 2` (3 created → 2 trained; the training titles `['doc1', 'doc1']` confirm `doc_inbox`, titled `doc235`, was excluded); **(b)** `rows inserted BEFORE any train() call : 2` followed by `self.classifier.train() ran AFTER, ret: True` (inserts precede training, and this first train on new data returns `True` = retrain); **(c)** `predict_correspondent(doc1) : [1]` with `c1.pk=1` is **accepted** because the class id is `!= -1`, while `predict_correspondent(doc2) : None` is **rejected** because the classifier returns the `-1` null class, which the rule maps to `None`. (The predicted id is a NumPy array `[1]`, so the elementwise accept check prints `[ True]`.) This exercises both sibling outcomes of the acceptance rule on identical input: class id present (accept) and class id `-1` (reject).

### (c) corroboration — the acceptance rule body and the only `>= 90`

The full body of `predict_correspondent` contains no numeric probability threshold — only the `!= -1` rule:

```console
$ docker exec --user testuser paperless-canon bash -lc 'cd /app/src && sed -n "251,260p" documents/classifier.py'; echo "EXIT=$?"
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
EXIT=0
```

As corroboration (not an exhaustive proof), a repository grep shows the sole `>= 90` in the `documents/` package lives in the rule-based fuzzy *string* matcher, unrelated to ML classifier confidence:

```console
$ docker exec --user testuser paperless-canon bash -lc 'cd /app/src && grep -rn ">= 90\|>=90" documents/*.py'; echo "EXIT=$?"
documents/matching.py:135:        if fuzz.partial_ratio(match, text) >= 90:
EXIT=0
```

That `fuzz.partial_ratio(match, text) >= 90` sits inside `matches()` (`src/documents/matching.py:L60-L152`) and governs the `MATCH_FUZZY` string algorithm — it compares a rule's keyword against document text and is unrelated to the MLP classifier's output.

### Canonical automatic-correspondent-matching tests (single 4-target run)

The canonical automatic-correspondent-matching-via-signal tests are `test_correspondent_applied` (`src/documents/tests/test_matchables.py:L425`) and `test_correspondent_not_applied` (`src/documents/tests/test_matchables.py:L437`), in class `TestDocumentConsumptionFinishedSignal` (`src/documents/tests/test_matchables.py:L380`, which extends plain `TestCase` — **not** `DirectoriesMixin`). Their document content is `"I contain the keyword."` (`src/documents/tests/test_matchables.py:L390`). Both send the `document_consumption_finished` signal **without a classifier** (so `classifier=None` inside `match_correspondents`), exercising the rule engine rather than the ML path:

- `test_correspondent_applied` (`src/documents/tests/test_matchables.py:L425-L435`) creates `Correspondent.objects.create(name="test", match="keyword", matching_algorithm=Correspondent.MATCH_ANY)`; the content matches `"keyword"`, so the correspondent is applied — a rule-based **positive**.
- `test_correspondent_not_applied` (`src/documents/tests/test_matchables.py:L437-L447`) creates a **`Tag`** — `Tag.objects.create(name="test", match="no-match", matching_algorithm=Correspondent.MATCH_ANY)` — **not** a `Correspondent`. Because no `Correspondent` row exists in the database, `match_correspondents` has nothing to return and the document's correspondent stays `None`. This is a **trivial** negative (there is no correspondent to match at all); it does **not** exercise a present-but-non-matching correspondent. The genuine ML **rejection** signal (a classifier predicting the `-1` null class → `None`) is the `predict_correspondent(doc2) : None` result from the probe above.

All four canonical tests pass in a single invocation (4 collected, 4 passed):

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict" "documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs" "documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_applied" "documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_not_applied" -o addopts="" -p no:cacheprovider -v'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 4 items

documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED [ 25%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED [ 50%]
documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_applied PASSED [ 75%]
documents/tests/test_matchables.py::TestDocumentConsumptionFinishedSignal::test_correspondent_not_applied PASSED [100%]

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
======================== 4 passed, 6 warnings in 2.17s =========================
EXIT=0
```

The dispatch chain is `set_correspondent` (`src/documents/signals/handlers.py:L35-L50`) → `matching.match_correspondents(document, classifier)` (`src/documents/signals/handlers.py:L50`); `match_correspondents` (`src/documents/matching.py:L21-L31`) returns every correspondent for which `matches(o, document) or o.pk == pred_id` (`src/documents/matching.py:L30`) — i.e. **rule match OR ML prediction**.

---

## Q3 — For a document with no extractable text: (a) which OCR subprocess is invoked, and (b) what MIME type is assigned to the output?

**Direct answer.**
- **(a) The OCR subprocess is OCRmyPDF, invoked via `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L261`), which drives Tesseract.** On a no-text document the *primary* call (with `skip_text=True`) produces no text and raises `NoTextFoundException`; that triggers a **force-OCR fallback** — a **second** `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L298`) built with `force_ocr=True` — after which empty text is accepted (`self.text = ""`). The ground-truth executables observed spawning during `parse()` were **`tesseract`, `unpaper`, and `gs`** (Ghostscript). During full consumption the thumbnail step additionally spawns ImageMagick **`convert`** (which fails on this container's PDF security policy) and **`gs`** (fallback), plus **`optipng`** (thumbnail optimisation).
- **(b) The MIME type assigned to the output is `image/png`** — the *original file's* MIME as detected by libmagic (`magic.from_file(self.path, mime=True)`, `src/documents/consumer.py:L219`) and stored on the `Document` (`src/documents/consumer.py:L401`). It is **not** the produced archive PDF's `application/pdf`. Observed: `Document.mime_type STORED : image/png`.

### The no-text fixture and read-only handling

The Q3 fixture is `src/paperless_tesseract/tests/samples/no-text-alpha.png` — a `PNG image data, 297 x 225, 8-bit/color RGBA, non-interlaced` file. **Critical read-only hazard:** during alpha-layer removal the parser executes `background.save(input_file, format=im.format)` (`src/paperless_tesseract/parsers.py:L201`, inside the `if self.has_alpha(input_file):` block at `src/paperless_tesseract/parsers.py:L190-L201`), which **overwrites the input file in place**. Parsing the tracked fixture directly would therefore mutate it (RGBA→RGB). To keep the source tree byte-for-byte unchanged, every probe below **copies the fixture to a private scratch path and parses only the copy**, then checksums the tracked source before and after to prove it is untouched. The fixture is also **not referenced by any Python code** (so it can only be exercised through a probe):

```console
$ docker exec --user testuser paperless-canon bash -lc 'cd /app/src && grep -rn "no-text-alpha" --include="*.py" . || echo "NO .py REFERENCES FOUND"'; echo "EXIT=$?"
NO .py REFERENCES FOUND
EXIT=0
```

### Self-contained probes (embedded source)

The parser-level probe drives the real `RasterisedDocumentParser.parse()` on a scratch copy with a `sys.addaudithook` capturing every real `subprocess.Popen` (ground truth of which binaries fire) and libmagic on input and produced archive:

```python
# /tmp/th/work/blitzy_adhoc_test_q3.py
# Q3 parser probe (temporary; deleted after use).
# Exercises the REAL RasterisedDocumentParser.parse() on the no-text RGBA
# fixture. CRITICAL: parsers.py:L201 `background.save(input_file, ...)`
# overwrites the input in place during alpha removal, so we NEVER parse the
# tracked fixture -- we parse a private scratch COPY and prove (by checksum)
# the tracked source is byte-identical before and after.
import hashlib
import os
import shutil
import subprocess
import sys
import tempfile

import magic
from django.test import TestCase

from documents.tests.utils import DirectoriesMixin
from paperless_tesseract.parsers import RasterisedDocumentParser

SRC = "/app/src/paperless_tesseract/tests/samples/no-text-alpha.png"

_spawned = []


def _audit(event, args):
    # Ground-truth capture of every real subprocess spawn.
    if event == "subprocess.Popen":
        exe = args[0]
        if exe:
            _spawned.append(os.path.basename(exe if isinstance(exe, str) else exe[0]))


def _sha256(path):
    with open(path, "rb") as f:
        return hashlib.sha256(f.read()).hexdigest()


class Q3ParserProbe(DirectoriesMixin, TestCase):
    def test_q3_notext_ocr_chain_on_scratch_copy(self):
        sys.addaudithook(_audit)

        src_before = _sha256(SRC)
        # Independent fresh copy in a private temp dir (never the tracked file).
        workdir = tempfile.mkdtemp(prefix="q3probe_")
        copy_path = os.path.join(workdir, "no-text-alpha.png")
        shutil.copy2(SRC, copy_path)
        copy_before = _sha256(copy_path)

        print("===== Q3 FIXTURE INTEGRITY (before parse) =====")
        print(f"SOURCE sha256 BEFORE : {src_before}")
        print(f"COPY   sha256 BEFORE : {copy_before}")
        print(f"COPY   path          : {copy_path}")
        print(f"input libmagic MIME  : {magic.from_file(copy_path, mime=True)}")

        del _spawned[:]
        parser = RasterisedDocumentParser(None)
        parser.parse(copy_path, "image/png")

        src_after = _sha256(SRC)
        copy_after = _sha256(copy_path)

        print("===== Q3(a) OCR CHAIN RESULT =====")
        print(f"subprocesses spawned during parse(): {sorted(set(_spawned))}")
        print(f"parser.text repr                    : {parser.text!r}")
        print(f"parser.text is empty string         : {parser.text == ''}")
        print(f"archive_path                        : {parser.archive_path}")
        print(f"archive exists                      : {os.path.isfile(parser.archive_path)}")
        if parser.archive_path and os.path.isfile(parser.archive_path):
            print(f"produced archive libmagic MIME      : {magic.from_file(parser.archive_path, mime=True)}")

        print("===== Q3 FIXTURE INTEGRITY (after parse) =====")
        print(f"SOURCE sha256 AFTER  : {src_after}")
        print(f"SOURCE unchanged     : {src_before == src_after}")
        print(f"COPY   sha256 AFTER  : {copy_after}")
        print(f"COPY   mutated by parse (alpha removal): {copy_before != copy_after}")
```

The consumer-level probe drives the real `Consumer.try_consume_file()` end-to-end with **no mocks at all** — `_send_progress` runs for real against an in-memory channel layer (the canonical `test_consumer.py` instead mocks `_send_progress` at `src/documents/tests/test_consumer.py:L290-L291` to avoid its Redis-backed production layer, and also fakes `magic.from_file` at `src/documents/tests/test_consumer.py:L236`; this probe fakes neither, since MIME detection is exactly what Q3(b) measures):

```python
# /tmp/th/work/blitzy_adhoc_test_q3consume.py
# Q3 consumer probe (temporary; deleted after use).
# Drives the REAL Consumer.try_consume_file() end-to-end to observe the MIME
# type STORED on the created Document. Fully canonical: NO mocks at all --
# _send_progress runs for real against an in-memory channel layer (the
# production backend channels_redis needs Redis, which the canonical
# test_consumer.py avoids by mocking _send_progress at
# src/documents/tests/test_consumer.py:L290-L291; here we instead run it for
# real via an in-memory layer). REAL libmagic is used (the canonical test
# fakes magic.from_file at test_consumer.py:L236 -- we do NOT, since MIME
# detection is exactly what Q3(b) measures).
# The consumer parses self.path in place, so we consume a private scratch
# COPY (faithful to real usage, where the consumer operates on a throwaway
# file in the consumption dir) and prove the tracked fixture is unchanged.
import hashlib
import os
import shutil
import sys
import tempfile

import magic
from django.test import override_settings, TestCase

from documents.consumer import Consumer
from documents.models import Document
from documents.tests.utils import DirectoriesMixin

SRC = "/app/src/paperless_tesseract/tests/samples/no-text-alpha.png"

_spawned = []


def _audit(event, args):
    if event == "subprocess.Popen":
        exe = args[0]
        if exe:
            _spawned.append(os.path.basename(exe if isinstance(exe, str) else exe[0]))


def _sha256(path):
    with open(path, "rb") as f:
        return hashlib.sha256(f.read()).hexdigest()


@override_settings(
    CHANNEL_LAYERS={"default": {"BACKEND": "channels.layers.InMemoryChannelLayer"}},
)
class Q3ConsumerProbe(DirectoriesMixin, TestCase):
    def test_q3_stored_mime_type_real_consumer(self):
        sys.addaudithook(_audit)
        src_before = _sha256(SRC)

        workdir = tempfile.mkdtemp(prefix="q3consume_")
        copy_path = os.path.join(workdir, "no-text-alpha.png")
        shutil.copy2(SRC, copy_path)

        print("===== Q3(b) CONSUMER-STORED Document.mime_type (REAL Consumer, REAL libmagic, NO mocks) =====")
        print(f"input libmagic MIME (magic.from_file) : {magic.from_file(copy_path, mime=True)}")

        del _spawned[:]
        consumer = Consumer()
        document = consumer.try_consume_file(copy_path)

        print(f"Document.mime_type STORED             : {document.mime_type}")
        print(f"Document.content repr                 : {document.content!r}")
        print(f"Document has archive checksum         : {document.archive_checksum is not None}")
        print(f"subprocesses spawned during consume   : {sorted(set(_spawned))}")
        print(f"SOURCE sha256 BEFORE                  : {src_before}")
        print(f"SOURCE sha256 AFTER                   : {_sha256(SRC)}")
        print(f"SOURCE unchanged                      : {src_before == _sha256(SRC)}")
```

### (a) Which OCR subprocess is invoked — the full ordered chain

**Cause → effect.** Inside `RasterisedDocumentParser.parse()` (`src/paperless_tesseract/parsers.py:L230`) the parameters are built by `construct_ocrmypdf_parameters(...)` (`src/paperless_tesseract/parsers.py:L135`). With the default `OCR_MODE="skip"` (`src/paperless/settings.py:L522`) this sets `skip_text=True` (`src/paperless_tesseract/parsers.py:L158`) and, because the fixture is RGBA, the alpha layer is removed first (`src/paperless_tesseract/parsers.py:L190-L201`). The **primary** OCR call is `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L261`); if no text results, `NoTextFoundException` (defined at `src/paperless_tesseract/parsers.py:L14`) is raised (`src/paperless_tesseract/parsers.py:L266-L267`). That triggers the **force-OCR fallback**: `construct_ocrmypdf_parameters(..., safe_fallback=True)` (`src/paperless_tesseract/parsers.py:L293`) sets `force_ocr=True` (`src/paperless_tesseract/parsers.py:L155-L156`), followed by a **second** `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L298`). If text is still empty, empty text is accepted at `self.text = ""` (`src/paperless_tesseract/parsers.py:L327`, the "the content will be empty" branch at `src/paperless_tesseract/parsers.py:L324`).

The complete probe run (fixture-integrity checksums, the naturally-logged INFO/WARNING chain, the spawned subprocesses, and the empty result). The decisive values were byte-identical across repeated identical runs:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q3.py -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 1 item

../../tmp/th/work/blitzy_adhoc_test_q3.py ===== Q3 FIXTURE INTEGRITY (before parse) =====
SOURCE sha256 BEFORE : a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b
COPY   sha256 BEFORE : a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b
COPY   path          : /tmp/q3probe_y_pndbh6/no-text-alpha.png
input libmagic MIME  : image/png
[2026-07-14 21:15:52,122] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3probe_y_pndbh6/no-text-alpha.png: 'dpi'
[2026-07-14 21:15:52,123] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/q3probe_y_pndbh6/no-text-alpha.png for compatibility with img2pdf
[2026-07-14 21:15:52,452] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:52,452] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:52,452] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-14 21:15:52,867] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:53,070] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-14 21:15:53,070] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3probe_y_pndbh6/no-text-alpha.png: 'dpi'
[2026-07-14 21:15:53,321] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:53,321] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:53,321] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-14 21:15:53,735] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:15:53,928] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/q3probe_y_pndbh6/no-text-alpha.png, the content will be empty.
===== Q3(a) OCR CHAIN RESULT =====
subprocesses spawned during parse(): ['gs', 'tesseract', 'unpaper']
parser.text repr                    : ''
parser.text is empty string         : True
archive_path                        : /tmp/tmp77srfe_j/paperless-y0txiz26/archive.pdf
archive exists                      : True
produced archive libmagic MIME      : application/pdf
===== Q3 FIXTURE INTEGRITY (after parse) =====
SOURCE sha256 AFTER  : a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b
SOURCE unchanged     : True
COPY   sha256 AFTER  : 2e85ef3e8115e41da9a10c6feedd587846cae462c0e19d3a150a341b5ce69fcf
COPY   mutated by parse (alpha removal): True
.

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
======================== 1 passed, 6 warnings in 3.41s =========================
EXIT=0
```

The two DEBUG-level `Calling OCRmyPDF with args: {...}` lines confirm the parameter difference between the primary attempt (`'skip_text': True`) and the fallback (`'force_ocr': True`). They are shown here through a transparent `grep` filter over the same probe run (the filter command is included so the stream is complete and un-redacted):

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q3.py -o addopts="" -p no:cacheprovider -o log_cli=true --log-cli-level=DEBUG -s 2>&1 | grep -E "paperless.parsing.tesseract:loggers.py:21 (Removing alpha|Calling OCRmyPDF|Encountered an error|Fallback: Calling OCRmyPDF|No text was found)"'; echo "EXIT=$?"
INFO     paperless.parsing.tesseract:loggers.py:21 Removing alpha layer from /tmp/q3probe_ylma6y7f/no-text-alpha.png for compatibility with img2pdf
DEBUG    paperless.parsing.tesseract:loggers.py:21 Calling OCRmyPDF with args: {'input_file': '/tmp/q3probe_ylma6y7f/no-text-alpha.png', 'output_file': '/tmp/tmpr3xch9mf/paperless-zo9zs4n2/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpr3xch9mf/paperless-zo9zs4n2/sidecar.txt', 'image_dpi': 35}
WARNING  paperless.parsing.tesseract:loggers.py:21 Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
DEBUG    paperless.parsing.tesseract:loggers.py:21 Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/q3probe_ylma6y7f/no-text-alpha.png', 'output_file': '/tmp/tmpr3xch9mf/paperless-zo9zs4n2/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpr3xch9mf/paperless-zo9zs4n2/sidecar-fallback.txt', 'image_dpi': 35}
WARNING  paperless.parsing.tesseract:loggers.py:21 No text was found in /tmp/q3probe_ylma6y7f/no-text-alpha.png, the content will be empty.
EXIT=0
```

Reading the two blocks together, the ordered chain — each branch exercised on the real fixture — is:
1. **Alpha-layer removal** — `Removing alpha layer …` (`src/paperless_tesseract/parsers.py:L190-L201`).
2. **Primary OCR** — `Calling OCRmyPDF with args: {... 'skip_text': True ...}` → `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L261`).
3. **NoTextFoundException** — `… No text was found in the original document. Attempting force OCR …` (raised at `src/paperless_tesseract/parsers.py:L266-L267`).
4. **Force-OCR fallback** — `Fallback: Calling OCRmyPDF with args: {... 'force_ocr': True ...}` → second `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L298`).
5. **Empty-text acceptance** — `… the content will be empty.` → `self.text = ""` (`src/paperless_tesseract/parsers.py:L327`); observed `parser.text == ''`.

The **ground-truth subprocesses** spawned during `parse()` were `['gs', 'tesseract', 'unpaper']`: OCRmyPDF drives **Tesseract** (the OCR engine), **`unpaper`** (image cleanup, from `clean=True` / `OCR_CLEAN="clean"` at `src/paperless/settings.py:L526`), and **`gs`** (Ghostscript, for the `output_type='pdfa'` PDF/A archive). The sidecar-based check that distinguishes a genuinely OCR'd page from a text-layer-skipped page is `if "[OCR skipped on page" not in text:` in `extract_text` (`src/paperless_tesseract/parsers.py:L104`).

**Fixture integrity (Q3 read-only proof).** In the block above, `SOURCE sha256` is identical before and after (`SOURCE unchanged : True`) while the scratch `COPY` is mutated by alpha removal (`COPY mutated by parse (alpha removal): True`) — i.e. the in-place `background.save` really does rewrite its input, but only the throwaway copy is affected. The tracked fixture's git blob hash was `e78b22bfbe53be5a9046bbfb32fef7e98abb9be9` both before and after all Q3 runs, and `git status --porcelain` for it was empty (verified in the Read-Only Invariant appendix).

### (b) What MIME type is assigned to the output

**Cause → effect.** MIME detection uses libmagic via `magic.from_file(self.path, mime=True)` in the consumer (`src/documents/consumer.py:L219`; the base-parser variant uses `magic.from_file(path, mime=True)` at `src/documents/parsers.py:L106`). The detected `mime_type` flows to `_store` (`src/documents/consumer.py:L301`) and is written on the row at `Document.objects.create(..., mime_type=mime_type, ...)` (`src/documents/consumer.py:L398`, `src/documents/consumer.py:L401`). For an input image this is the **original file's** MIME — `image/png` — not the archive PDF that OCRmyPDF produces.

Observed by consuming a scratch copy of `no-text-alpha.png` through the real `Consumer.try_consume_file(...)` (no mocks). Note the thumbnail evidence is present **in the same run, adjacent** to the MIME result: ImageMagick `convert` is attempted, fails on this container's PDF security policy (`convert-im6.q16: attempt to perform an operation not allowed by the security policy 'PDF'`), the parser logs `Thumbnail generation with ImageMagick failed, falling back to ghostscript` (`src/documents/parsers.py:L158`), and the `gs` fallback (`src/documents/parsers.py:L164`) completes it:

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q3consume.py -o addopts="" -p no:cacheprovider -s'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 1 item

../../tmp/th/work/blitzy_adhoc_test_q3consume.py ===== Q3(b) CONSUMER-STORED Document.mime_type (REAL Consumer, REAL libmagic, NO mocks) =====
input libmagic MIME (magic.from_file) : image/png
[2026-07-14 21:16:22,120] [INFO] [paperless.consumer] Consuming no-text-alpha.png
[2026-07-14 21:16:22,293] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3consume_jiksakmd/no-text-alpha.png: 'dpi'
[2026-07-14 21:16:22,293] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/q3consume_jiksakmd/no-text-alpha.png for compatibility with img2pdf
[2026-07-14 21:16:22,621] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:22,622] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:22,622] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-14 21:16:23,044] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:23,249] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-14 21:16:23,250] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/q3consume_jiksakmd/no-text-alpha.png: 'dpi'
[2026-07-14 21:16:23,513] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:23,513] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:23,513] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-14 21:16:23,945] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-14 21:16:24,143] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/q3consume_jiksakmd/no-text-alpha.png, the content will be empty.
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmp3q5jfis7/paperless-osl3ufpn/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-14 21:16:24,158] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-14 21:16:27,149] [INFO] [paperless.consumer] Document 2026-07-14 no-text-alpha consumption finished
Document.mime_type STORED             : image/png
Document.content repr                 : ''
Document has archive checksum         : True
subprocesses spawned during consume   : ['convert', 'gs', 'optipng', 'tesseract', 'unpaper']
SOURCE sha256 BEFORE                  : a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b
SOURCE sha256 AFTER                   : a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b
SOURCE unchanged                      : True
.

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
======================== 1 passed, 6 warnings in 6.51s =========================
EXIT=0
```

The stored **`Document.mime_type` is `image/png`** (byte-identical across the repeated runs), matching the libmagic result on the original input and confirming `src/documents/consumer.py:L219` → `src/documents/consumer.py:L401`. The extracted `content` is `''` (consistent with Q3(a)). The pipeline *does* produce a PDF/A archive (`Document has archive checksum : True`) whose own MIME is `application/pdf` (shown in the parser block above), but that belongs to the `archive_path`, **not** the value assigned as the `Document`'s MIME type.

**Thumbnail subprocess chain (observed, Q3(a) supplement).** `get_optimised_thumbnail` → the tesseract parser's `get_thumbnail` (`src/paperless_tesseract/parsers.py:L57-L61`) → `make_thumbnail_from_pdf` (`src/documents/parsers.py:L187`) first calls `run_convert` (ImageMagick `convert`, `src/documents/parsers.py:L111`, args assembled from `settings.CONVERT_BINARY` at `src/documents/parsers.py:L132`, spawned at `src/documents/parsers.py:L145`); on failure it falls back to `make_thumbnail_from_pdf_gs_fallback` (`src/documents/parsers.py:L153`, invoked at `src/documents/parsers.py:L207`) which runs Ghostscript with `-sDEVICE=pngalpha` (`src/documents/parsers.py:L164`). The consumption run's spawned set `['convert', 'gs', 'optipng', 'tesseract', 'unpaper']` reflects both the OCR subprocesses and this thumbnail fallback.

### Canonical no-text / empty-text parser tests

The canonical parser tests that exercise the surrounding empty-/no-text behaviour pass: `test_encrypted` (`src/paperless_tesseract/tests/test_parser.py:L178`, asserts `get_text() == ""`), `test_with_form_error_notext` (`src/paperless_tesseract/tests/test_parser.py:L190`, `@override_settings(OCR_MODE="redo")`), and `test_skip_noarchive_notext` (`src/paperless_tesseract/tests/test_parser.py:L370`, `@override_settings(OCR_MODE="skip_noarchive")`):

```console
$ docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest "paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted" "paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext" "paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext" -o addopts="" -p no:cacheprovider -v'; echo "EXIT=$?"
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 3 items

paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted PASSED [ 33%]
paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED [ 66%]
paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED [100%]

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
======================== 3 passed, 6 warnings in 9.97s =========================
EXIT=0
```

---

## Q4 — For barcode splitting: (a) how many document records are created from a single input, (b) which barcode values trigger a split, (c) where is the decision made, and (d) does this change the effective training data during the run?

**Direct answer.**
- **(a) A single input file splits into `separators + 1` output *segments*, and those segments become `Document` *records* only when subsequently consumed — `consume_file` itself creates zero records.** The split is performed by `separate_pages` (`src/documents/tasks.py:L113-L161`): it writes `{fname}_document_0.pdf` for the pages before the first separator, then one file per subsequent separator, **dropping each separator page** via `for page in range(page_number + 1, next_page)` (`src/documents/tasks.py:L148-L149`). Observed directly: a single-separator input `[1]` on a 3-page PDF → **2 segments** with page counts `[1, 1]` (1 page dropped); a two-separator input `[2, 5]` on a 7-page PDF → **3 segments** with page counts `[2, 2, 1]` (2 pages dropped). `consume_file` (`src/documents/tasks.py:L184-L233`) writes those segments to the consumption directory and returns the literal string `"File successfully split"` (`src/documents/tasks.py:L233`) while creating **zero `Document` rows** (observed `0` before and `0` after). With the gate **off** (the default), no split occurs and the single input becomes exactly **one** `Document` (observed `"Success. New document id 1 created"`, rows `0 → 1`). When the segments *are* consumed, each distinct segment becomes one `Document` record (observed 2 segments → 1 record and 3 segments → 2 records, because paperless rejects the byte-identical **segment files** — the repeated no-barcode body pages, since the `PATCHT` separator pages are dropped rather than consumed — as duplicates at `pre_check_duplicate`, `src/documents/consumer.py:L110`). Observed directly: for `patch-code-t-middle.pdf` the two emitted segments both decode to no barcodes and are byte-identical (same `md5`), and the dropped page is exactly the `PATCHT`-bearing page.
- **(b) The trigger is the decoded barcode *value* `"PATCHT"`** — the default of `settings.CONSUMER_BARCODE_STRING` (`src/paperless/settings.py:L506`). A page becomes a separator when a decoded barcode's value **equals** that string; the match is **symbology-agnostic**, observed triggering across **Code 39** (`barcode-39-PATCHT.png`), **Code 128** (`barcode-128-PATCHT.png`), and **QR** (`qr-code-PATCHT.png`), all decoding to `PATCHT`. Distorted Code 39 variants (`barcode-39-PATCHT-distorsion.png`, `barcode-39-PATCHT-distorsion2.png`) still decode to `PATCHT`; an **unreadable** barcode (`barcode-39-PATCHT-unreadable.png`) and a page with **no barcode** (`simple.png`) both decode to `[]` and do **not** trigger a split. The value is configurable: with `CONSUMER_BARCODE_STRING="CUSTOM BARCODE"`, the three `*-custom` fixtures (Code 39 / QR / Code 128) trigger, whereas the default `"PATCHT"` does **not** match them. The whole feature is gated by `CONSUMER_ENABLE_BARCODES` (`src/paperless/settings.py:L502`, default `False`).
- **(c) The split decision is made in `scan_file_for_separating_barcodes`** — specifically `if separator_barcode in current_barcodes:` → `separator_page_numbers.append(current_page_number)` at **`src/documents/tasks.py:L108-L109`**. That function's return value is the list of separator page numbers observed below (`[0]`, `[1]`, `[2, 5]`, `[]`).
- **(d) Yes — indirectly.** Splitting does not itself create records, but consuming the resulting segments adds `Document` rows, which changes the effective training data. Observed the full real chain: trained-eligible rows (`Document.objects.exclude(tags__is_inbox_tag=True)`, inbox rows excluded per `src/documents/classifier.py:L125-L127`) grew `0 → 1 → 3` as split segments were consumed; `DocumentClassifier.train()` returned `True` (retrain) on first fit, `False` (reuse) when re-run on unchanged data, then `True` again (retrain) once new consumed segments changed the SHA-1 `data_hash` — the exact reuse/retrain mechanism established in Q1 (`src/documents/classifier.py:L163-L164`).

### Read-only handling and canonical entry points

All barcode fixtures live under `src/documents/tests/samples/barcodes/` (with `simple.png` / `simple.pdf` under `src/documents/tests/samples/`) and are read-only inputs. `consume_file` unlinks its input on a successful split (`src/documents/tasks.py:L214`), and `Consumer.try_consume_file` consumes (and removes) the file it is given, so the probe copies any consume-path input to a private scratch path first and never feeds a tracked fixture directly; `separate_pages` writes its segments to a fresh `tempfile.mkdtemp` under `SCRATCH_DIR`. The consume methods run against an in-memory channel layer (`channels.layers.InMemoryChannelLayer`) so the real `Consumer._send_progress` executes without a Redis broker; every other function (`barcode_reader`, `scan_file_for_separating_barcodes`, `separate_pages`, `consume_file`, `Consumer.try_consume_file`, `DocumentClassifier.train`) is the genuine, unmocked code path.

### Self-contained probe (embedded source)

```python
# Q4 barcode probe (temporary; deleted after use).
# Exercises the REAL documents.tasks barcode functions and the REAL Consumer
# across EVERY distinct condition. Uses a private scratch copy for any
# consume_file input (consume_file unlinks its input on success).
import os
import shutil
import tempfile

import pikepdf
from PIL import Image
from pyzbar import pyzbar
from django.test import override_settings, TestCase

from documents import tasks
from documents.consumer import Consumer, ConsumerError
from documents.classifier import DocumentClassifier
from documents.models import Document
from documents.tests.utils import DirectoriesMixin

BC = "/app/src/documents/tests/samples/barcodes"


def _syms_and_values(img_path):
    with Image.open(img_path) as im:
        decoded = pyzbar.decode(im)
    return [d.type for d in decoded], [d.data.decode("utf-8") for d in decoded]


def _reader(img_path):
    with Image.open(img_path) as im:
        return tasks.barcode_reader(im)


class Q4Probe(DirectoriesMixin, TestCase):
    # ---------- (b) barcode_reader: values + symbologies, every image ----------
    def test_q4b_barcode_reader_all(self):
        print("===== Q4(b) barcode_reader VALUES + SYMBOLOGIES (real tasks.barcode_reader) =====")
        cases = [
            ("barcode-39-PATCHT.png", "Code 39 PATCHT"),
            ("barcode-128-PATCHT.png", "Code 128 PATCHT"),
            ("qr-code-PATCHT.png", "QR PATCHT"),
            ("barcode-39-PATCHT-distorsion.png", "Code 39 distorted"),
            ("barcode-39-PATCHT-distorsion2.png", "Code 39 distorted2"),
            ("barcode-39-PATCHT-unreadable.png", "Code 39 UNREADABLE"),
            ("patch-code-t.pbm", "PBM PATCHT"),
            ("simple.png", "NO barcode"),
            ("barcode-39-custom.png", "Code 39 custom"),
            ("barcode-qr-custom.png", "QR custom"),
            ("barcode-128-custom.png", "Code 128 custom"),
        ]
        for fn, label in cases:
            p = os.path.join(BC, fn)
            if fn == "simple.png":
                p = "/app/src/documents/tests/samples/simple.png"
            syms, vals = _syms_and_values(p)
            print(f"  {label:22s} {fn:34s} symbology={syms} value={vals} barcode_reader={_reader(p)}")

    # ---------- (b/c) scan default PATCHT: PDFs -> separator page numbers ----------
    def test_q4bc_scan_default_patcht(self):
        print("===== Q4(b)/(c) scan_file_for_separating_barcodes, DEFAULT 'PATCHT' =====")
        cases = [
            ("patch-code-t.pdf", "page-0 separator"),
            ("simple.pdf", "no barcode"),
            ("patch-code-t-middle.pdf", "single separator"),
            ("several-patcht-codes.pdf", "multi separator"),
            ("patch-code-t-middle_reverse.pdf", "upside-down separator"),
            ("patch-code-t-qr.pdf", "QR separator"),
            ("barcode-39-custom.pdf", "MISMATCH (custom fixture vs default PATCHT)"),
        ]
        p = "/app/src/documents/tests/samples/simple.pdf"
        for fn, label in cases:
            path = p if fn == "simple.pdf" else os.path.join(BC, fn)
            print(f"  {label:44s} {fn:34s} -> {tasks.scan_file_for_separating_barcodes(path)}")

    # ---------- (b) scan with CUSTOM separator string ----------
    @override_settings(CONSUMER_BARCODE_STRING="CUSTOM BARCODE")
    def test_q4b_scan_custom_string(self):
        print("===== Q4(b) scan with CONSUMER_BARCODE_STRING='CUSTOM BARCODE' =====")
        for fn in ["barcode-39-custom.pdf", "barcode-qr-custom.pdf", "barcode-128-custom.pdf"]:
            path = os.path.join(BC, fn)
            print(f"  custom-string scan {fn:26s} -> {tasks.scan_file_for_separating_barcodes(path)}")

    # ---------- (a) separate_pages segment count + separator-page DROP ----------
    def test_q4a_separate_pages_and_drop(self):
        print("===== Q4(a) separate_pages SEGMENT COUNT + separator-page DROP =====")
        for fn, seps in [("patch-code-t-middle.pdf", [1]), ("several-patcht-codes.pdf", [2, 5])]:
            path = os.path.join(BC, fn)
            in_pages = len(pikepdf.open(path).pages)
            segs = tasks.separate_pages(path, seps)
            seg_pages = [len(pikepdf.open(s).pages) for s in segs]
            print(f"  {fn:26s} input_pages={in_pages} separators={seps} "
                  f"-> {len(segs)} segments (separators+1={len(seps)+1}); "
                  f"segment_page_counts={seg_pages}; total_out={sum(seg_pages)}; "
                  f"pages_dropped={in_pages - sum(seg_pages)}")
            print(f"     segment filenames={[os.path.basename(s) for s in segs]}")
        empty = tasks.separate_pages(os.path.join(BC, "patch-code-t-middle.pdf"), [])
        print(f"  empty separator list -> segments={empty} (see 'No pages to split on!' warning)")

    # ---------- (a) gate ON: consume_file splits, ZERO Document rows ----------
    @override_settings(
        CONSUMER_ENABLE_BARCODES=True,
        CHANNEL_LAYERS={"default": {"BACKEND": "channels.layers.InMemoryChannelLayer"}},
    )
    def test_q4a_consume_gate_on(self):
        print("===== Q4(a) consume_file, GATE ON: 'File successfully split' + ZERO Document rows =====")
        src = os.path.join(BC, "patch-code-t-middle.pdf")
        dst = os.path.join(tempfile.mkdtemp(prefix="q4_"), "patch-code-t-middle.pdf")
        shutil.copy(src, dst)
        before = Document.objects.count()
        ret = tasks.consume_file(dst)
        after = Document.objects.count()
        print(f"  Document rows BEFORE = {before}")
        print(f"  consume_file(...) returned = {ret!r}")
        print(f"  Document rows AFTER  = {after}   (consume_file itself creates NO Document rows)")

    # ---------- (a) gate OFF (default): consume_file -> single Document ----------
    @override_settings(
        CHANNEL_LAYERS={"default": {"BACKEND": "channels.layers.InMemoryChannelLayer"}},
    )
    def test_q4a_consume_gate_off_default(self):
        print("===== Q4(a) consume_file, GATE OFF (default CONSUMER_ENABLE_BARCODES): single document, NO split =====")
        print(f"  settings.CONSUMER_ENABLE_BARCODES default = {tasks.settings.CONSUMER_ENABLE_BARCODES}")
        src = os.path.join(BC, "patch-code-t-middle.pdf")
        dst = os.path.join(tempfile.mkdtemp(prefix="q4off_"), "patch-code-t-middle.pdf")
        shutil.copy(src, dst)
        before = Document.objects.count()
        ret = tasks.consume_file(dst)
        after = Document.objects.count()
        print(f"  Document rows BEFORE = {before}")
        print(f"  consume_file(...) returned = {ret!r}")
        print(f"  Document rows AFTER  = {after}   (gate off -> normal single-document consumption)")

    # ---------- (d) REAL chain: split -> consume segments -> rows -> hash -> retrain ----------
    @override_settings(
        CONSUMER_ENABLE_BARCODES=True,
        CHANNEL_LAYERS={"default": {"BACKEND": "channels.layers.InMemoryChannelLayer"}},
    )
    def test_q4d_real_split_consume_train_chain(self):
        print("===== Q4(d) REAL chain: barcode split -> consume segments -> Document rows -> SHA-1 hash -> reuse/retrain =====")
        # The downstream of a split is: each segment produced by the REAL
        # separate_pages() is consumed as an independent file. We drive that
        # exact path: scan_file_for_separating_barcodes -> separate_pages ->
        # Consumer().try_consume_file(segment) for each segment.

        def split_and_consume(fixture):
            src = os.path.join(BC, fixture)
            seps = tasks.scan_file_for_separating_barcodes(src)
            segs = tasks.separate_pages(src, seps)  # real split segments (SCRATCH_DIR)
            consumed = 0
            duplicates = 0
            for s in segs:
                try:
                    Consumer().try_consume_file(s)  # each segment -> one Document row
                    consumed += 1
                except ConsumerError as e:
                    # repeated no-barcode body-page segments share a checksum -> pre_check_duplicate
                    # (src/documents/consumer.py:L110) rejects the repeat (PATCHT pages were dropped)
                    duplicates += 1
                    print(f"    segment {os.path.basename(s)}: ConsumerError -> {e}")
            return seps, len(segs), consumed, duplicates

        def trained_eligible():
            # inbox-tagged docs are excluded from training (src/documents/classifier.py:L125-L127)
            return Document.objects.exclude(tags__is_inbox_tag=True).count()

        clf = DocumentClassifier()
        print(f"  trained-eligible rows (exclude inbox) at start = {trained_eligible()}")

        seps1, nseg1, n1, dup1 = split_and_consume("patch-code-t-middle.pdf")
        print(f"  round1 fixture=patch-code-t-middle.pdf separators={seps1} segments={nseg1} consumed={n1} duplicates={dup1}")
        r_after1 = trained_eligible()
        print(f"  trained-eligible rows after round1 = {r_after1}")
        t1 = clf.train()
        t1b = clf.train()
        print(f"  clf.train() first on {r_after1} rows = {t1} (True=retrain); clf.train() again unchanged = {t1b} (False=reuse)")

        seps2, nseg2, n2, dup2 = split_and_consume("several-patcht-codes.pdf")
        print(f"  round2 fixture=several-patcht-codes.pdf separators={seps2} segments={nseg2} consumed={n2} duplicates={dup2}")
        r_after2 = trained_eligible()
        print(f"  trained-eligible rows after round2 = {r_after2}  (grew from {r_after1})")
        t2 = clf.train()
        print(f"  clf.train() after new consumed segments = {t2} (True=RETRAIN forced by changed SHA-1 data_hash)")
```

### (b) Which barcode values trigger a split — decoded values, symbologies, and scan results

**Cause → effect.** `barcode_reader` decodes an image with `pyzbar.decode(image)` (`src/documents/tasks.py:L82`) and returns the list of decoded string values. `scan_file_for_separating_barcodes` rasterizes each PDF page with `convert_from_path(...)` (poppler; `src/documents/tasks.py:L105`), reads the separator string `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` (`src/documents/tasks.py:L102`), and records a separator page when the value matches (decision site, below). The block below shows the real `tasks.barcode_reader` result for every fixture (with the raw pyzbar `.type` symbology beside it), then the real `scan_file_for_separating_barcodes` result for every PDF under the default `"PATCHT"` and under a custom string:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q4.py \
 -p no:cacheprovider -o addopts="" -n0 -s -v -k "barcode_reader_all or scan_default_patcht or scan_custom_string" 2>&1; echo "PYTEST_EXIT=$?"'
```

```
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 7 items / 4 deselected / 3 selected

../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4b_barcode_reader_all Creating test database for alias 'default'...
===== Q4(b) barcode_reader VALUES + SYMBOLOGIES (real tasks.barcode_reader) =====
  Code 39 PATCHT         barcode-39-PATCHT.png              symbology=['CODE39'] value=['PATCHT'] barcode_reader=['PATCHT']
  Code 128 PATCHT        barcode-128-PATCHT.png             symbology=['CODE128'] value=['PATCHT'] barcode_reader=['PATCHT']
  QR PATCHT              qr-code-PATCHT.png                 symbology=['QRCODE'] value=['PATCHT'] barcode_reader=['PATCHT']
  Code 39 distorted      barcode-39-PATCHT-distorsion.png   symbology=['CODE39'] value=['PATCHT'] barcode_reader=['PATCHT']
  Code 39 distorted2     barcode-39-PATCHT-distorsion2.png  symbology=['CODE39'] value=['PATCHT'] barcode_reader=['PATCHT']
  Code 39 UNREADABLE     barcode-39-PATCHT-unreadable.png   symbology=[] value=[] barcode_reader=[]
  PBM PATCHT             patch-code-t.pbm                   symbology=['CODE39'] value=['PATCHT'] barcode_reader=['PATCHT']
  NO barcode             simple.png                         symbology=[] value=[] barcode_reader=[]
  Code 39 custom         barcode-39-custom.png              symbology=['CODE39'] value=['CUSTOM BARCODE'] barcode_reader=['CUSTOM BARCODE']
  QR custom              barcode-qr-custom.png              symbology=['QRCODE'] value=['CUSTOM BARCODE'] barcode_reader=['CUSTOM BARCODE']
  Code 128 custom        barcode-128-custom.png             symbology=['CODE128'] value=['CUSTOM BARCODE'] barcode_reader=['CUSTOM BARCODE']
PASSED
../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4b_scan_custom_string ===== Q4(b) scan with CONSUMER_BARCODE_STRING='CUSTOM BARCODE' =====
  custom-string scan barcode-39-custom.pdf      -> [0]
  custom-string scan barcode-qr-custom.pdf      -> [0]
  custom-string scan barcode-128-custom.pdf     -> [0]
PASSED
../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4bc_scan_default_patcht ===== Q4(b)/(c) scan_file_for_separating_barcodes, DEFAULT 'PATCHT' =====
  page-0 separator                             patch-code-t.pdf                   -> [0]
  no barcode                                   simple.pdf                         -> []
  single separator                             patch-code-t-middle.pdf            -> [1]
  multi separator                              several-patcht-codes.pdf           -> [2, 5]
  upside-down separator                        patch-code-t-middle_reverse.pdf    -> [1]
  QR separator                                 patch-code-t-qr.pdf                -> [0]
  MISMATCH (custom fixture vs default PATCHT)  barcode-39-custom.pdf              -> []
PASSEDDestroying test database for alias 'default'...


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
================= 3 passed, 4 deselected, 6 warnings in 4.33s ==================
PYTEST_EXIT=0
```

The decoded **value** `PATCHT` triggers the split independent of symbology (Code 39, Code 128, QR all observed), including the two distorted Code 39 variants; the unreadable fixture and the no-barcode image both yield `[]` (no trigger). Under the default `"PATCHT"`, `barcode-39-custom.pdf` scans to `[]` (its value `CUSTOM BARCODE` does not match), whereas under `CONSUMER_BARCODE_STRING="CUSTOM BARCODE"` the three custom fixtures each scan to `[0]`. The upside-down separator (`patch-code-t-middle_reverse.pdf → [1]`) confirms orientation-independent decoding.

### (a) How many segments result from a single input — and when records appear

**Cause → effect.** `separate_pages` emits segment 0 (pages before the first separator), then one segment per separator, skipping the separator page itself (`for page in range(page_number + 1, next_page)`, `src/documents/tasks.py:L148-L149`); hence **segments = separators + 1** and each separator page is dropped. `consume_file` with the gate **on** writes those segments to the consumption directory (`save_to_dir`, `src/documents/tasks.py:L210`), unlinks the original (`src/documents/tasks.py:L214`), and returns `"File successfully split"` (`src/documents/tasks.py:L233`) — creating **no** `Document` rows. With the gate **off** (default) it falls through to `Consumer().try_consume_file` (`src/documents/tasks.py:L236`) and produces exactly one document. The canonical `test_separate_pages` (`src/documents/tests/test_tasks.py:L305-L313`) asserts `len(separate_pages(patch-code-t-middle.pdf, [1])) == 2`; observed directly for single- and multi-separator inputs, plus both gate states:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q4.py \
 -p no:cacheprovider -o addopts="" -n0 -s -v -k "separate_pages_and_drop or consume_gate_on or consume_gate_off_default" 2>&1; echo "PYTEST_EXIT=$?"'
```

```
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 7 items / 4 deselected / 3 selected

../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4a_consume_gate_off_default Creating test database for alias 'default'...
===== Q4(a) consume_file, GATE OFF (default CONSUMER_ENABLE_BARCODES): single document, NO split =====
  settings.CONSUMER_ENABLE_BARCODES default = False
[2026-07-14 21:36:01,511] [INFO] [paperless.consumer] Consuming patch-code-t-middle.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmppusxeng4/paperless-lk1itvui/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-14 21:36:07,759] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-14 21:36:08,972] [INFO] [paperless.consumer] Document 2010-12-01 patch-code-t-middle consumption finished
  Document rows BEFORE = 0
  consume_file(...) returned = 'Success. New document id 1 created'
  Document rows AFTER  = 1   (gate off -> normal single-document consumption)
PASSED
../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4a_consume_gate_on ===== Q4(a) consume_file, GATE ON: 'File successfully split' + ZERO Document rows =====
  Document rows BEFORE = 0
  consume_file(...) returned = 'File successfully split'
  Document rows AFTER  = 0   (consume_file itself creates NO Document rows)
PASSED
../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4a_separate_pages_and_drop ===== Q4(a) separate_pages SEGMENT COUNT + separator-page DROP =====
  patch-code-t-middle.pdf    input_pages=3 separators=[1] -> 2 segments (separators+1=2); segment_page_counts=[1, 1]; total_out=2; pages_dropped=1
     segment filenames=['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
  several-patcht-codes.pdf   input_pages=7 separators=[2, 5] -> 3 segments (separators+1=3); segment_page_counts=[2, 2, 1]; total_out=5; pages_dropped=2
     segment filenames=['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[2026-07-14 21:36:09,399] [WARNING] [paperless.tasks] No pages to split on!
  empty separator list -> segments=[] (see 'No pages to split on!' warning)
PASSEDDestroying test database for alias 'default'...


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
================= 3 passed, 4 deselected, 6 warnings in 9.38s ==================
PYTEST_EXIT=0
```

The single-separator PDF (3 pages, `[1]`) yields 2 segments of 1 page each (1 page — the separator — dropped); the multi-separator PDF (7 pages, `[2, 5]`) yields 3 segments of `[2, 2, 1]` pages (2 separator pages dropped). An empty separator list logs `No pages to split on!` and returns `[]`. Gate-on: `"File successfully split"`, rows `0 → 0`. Gate-off (default `CONSUMER_ENABLE_BARCODES = False`): `"Success. New document id 1 created"`, rows `0 → 1`. (The `convert-im6.q16 … not allowed by the security policy 'PDF'` line is the ImageMagick thumbnail step being denied and falling back to Ghostscript — the same benign fallback documented in Q3, `src/documents/parsers.py:L158`; consumption still succeeds.)

### (c) Where the split decision is made

**Cause → effect.** Inside `scan_file_for_separating_barcodes` (`src/documents/tasks.py:L96-L110`), after rasterizing (`src/documents/tasks.py:L105`) and decoding (`barcode_reader`, `src/documents/tasks.py:L82`) each page, the split point is recorded by:

`if separator_barcode in current_barcodes:` → `separator_page_numbers.append(current_page_number)` at **`src/documents/tasks.py:L108-L109`**.

The separator-page lists returned by this function — `[0]` (`patch-code-t.pdf`), `[1]` (`patch-code-t-middle.pdf`), `[2, 5]` (`several-patcht-codes.pdf`), `[]` (`simple.pdf` / mismatched `barcode-39-custom.pdf`) — are exactly the values shown in the (b) block above, confirming this is the single decision site.

### (d) Does this change the effective training data during the run?

**Direct answer: yes, indirectly.** `consume_file` creates no rows itself; but the segments it produces, **once consumed**, become additional `Document` rows, and additional rows change the SHA-1 `data_hash` computed over `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` in `train()` (`src/documents/classifier.py:L125-L161`), flipping the reuse short-circuit (`src/documents/classifier.py:L163-L164`) to force a retrain. The block below drives the real chain — `scan_file_for_separating_barcodes` → `separate_pages` → `Consumer().try_consume_file(segment)` for each segment — counting trained-eligible rows and calling `train()` between rounds:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q4.py \
 -p no:cacheprovider -o addopts="" -n0 -s -v -k "real_split_consume_train_chain" 2>&1; echo "PYTEST_EXIT=$?"'
```

```
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 7 items / 6 deselected / 1 selected

../../tmp/th/work/blitzy_adhoc_test_q4.py::Q4Probe::test_q4d_real_split_consume_train_chain Creating test database for alias 'default'...
===== Q4(d) REAL chain: barcode split -> consume segments -> Document rows -> SHA-1 hash -> reuse/retrain =====
  trained-eligible rows (exclude inbox) at start = 0
[2026-07-14 21:36:13,198] [INFO] [paperless.consumer] Consuming patch-code-t-middle_document_0.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpo6eg1jsb/paperless-xl2z3euf/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-14 21:36:13,625] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-14 21:36:14,350] [INFO] [paperless.consumer] Document 2026-07-14 patch-code-t-middle_document_0 consumption finished
[2026-07-14 21:36:14,353] [ERROR] [paperless.consumer] Not consuming patch-code-t-middle_document_1.pdf: It is a duplicate.
    segment patch-code-t-middle_document_1.pdf: ConsumerError -> patch-code-t-middle_document_1.pdf: Not consuming patch-code-t-middle_document_1.pdf: It is a duplicate.
  round1 fixture=patch-code-t-middle.pdf separators=[1] segments=2 consumed=1 duplicates=1
  trained-eligible rows after round1 = 1
  clf.train() first on 1 rows = True (True=retrain); clf.train() again unchanged = False (False=reuse)
[2026-07-14 21:36:15,548] [INFO] [paperless.consumer] Consuming several-patcht-codes_document_0.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpo6eg1jsb/paperless-jlm82c7q/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-14 21:36:15,803] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-14 21:36:16,568] [INFO] [paperless.consumer] Document 2026-07-14 several-patcht-codes_document_0 consumption finished
[2026-07-14 21:36:16,571] [ERROR] [paperless.consumer] Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
    segment several-patcht-codes_document_1.pdf: ConsumerError -> several-patcht-codes_document_1.pdf: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
[2026-07-14 21:36:16,572] [INFO] [paperless.consumer] Consuming several-patcht-codes_document_2.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpo6eg1jsb/paperless-rlkhjhrc/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-14 21:36:16,804] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-14 21:36:17,534] [INFO] [paperless.consumer] Document 2026-07-14 several-patcht-codes_document_2 consumption finished
  round2 fixture=several-patcht-codes.pdf separators=[2, 5] segments=3 consumed=2 duplicates=1
  trained-eligible rows after round2 = 3  (grew from 1)
  clf.train() after new consumed segments = True (True=RETRAIN forced by changed SHA-1 data_hash)
PASSEDDestroying test database for alias 'default'...


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
================= 1 passed, 6 deselected, 6 warnings in 6.24s ==================
PYTEST_EXIT=0
```

Round 1 (`patch-code-t-middle.pdf`, separators `[1]`, 2 segments): 1 segment consumed, 1 rejected as a duplicate → trained-eligible rows `0 → 1`; `train()` returns `True` (first fit = retrain) then `False` (re-run on unchanged data = reuse). Round 2 (`several-patcht-codes.pdf`, separators `[2, 5]`, 3 segments): 2 consumed, 1 duplicate → trained-eligible rows `1 → 3`; `train()` returns `True` again — the changed SHA-1 `data_hash` **forces a retrain**. This is the same reuse/retrain mechanism from Q1, now driven end-to-end by barcode splitting: split → consume segments → more `Document` rows → changed training hash → retrain. Inbox-tagged rows are excluded from this count (`src/documents/classifier.py:L125-L127`), exactly as in Q2.

### Canonical barcode test family (all 25 pass)

The complete on-disk barcode/separation/split test family in `src/documents/tests/test_tasks.py`, run canonically, all pass:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest documents/tests/test_tasks.py \
 -p no:cacheprovider -o addopts="" -n0 -v -k "barcode or separating or separate_pages or split" 2>&1; echo "PYTEST_EXIT=$?"'
```

```
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 40 items / 15 deselected / 25 selected

documents/tests/test_tasks.py::TestTasks::test_barcode_reader PASSED     [  4%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader2 PASSED    [  8%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 PASSED [ 12%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_128_separator PASSED [ 16%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_qr_separator PASSED [ 20%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_separator PASSED [ 24%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion PASSED [ 28%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion2 PASSED [ 32%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_no_barcode PASSED [ 36%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr PASSED  [ 40%]
documents/tests/test_tasks.py::TestTasks::test_barcode_reader_unreadable PASSED [ 44%]
documents/tests/test_tasks.py::TestTasks::test_barcode_splitter PASSED   [ 48%]
documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file PASSED [ 52%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED [ 56%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes2 PASSED [ 60%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3 PASSED [ 64%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED [ 68%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes_upsidedown PASSED [ 72%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_128_barcodes PASSED [ 76%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_barcodes PASSED [ 80%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_qr_barcodes PASSED [ 84%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_qr_barcodes PASSED [ 88%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_wrong_qr_barcodes PASSED [ 92%]
documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 96%]
documents/tests/test_tasks.py::TestTasks::test_separate_pages_no_list PASSED [100%]

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
================ 25 passed, 15 deselected, 6 warnings in 5.09s =================
PYTEST_EXIT=0
```

## Closing — The pytest-xdist + shared `MODEL_FILE` determinism hypothesis

**Direct answer (honest result first).** Across **5 identical parallel runs + 3 identical serial runs** of the three classification modules, the reported non-determinism **did not reproduce** here — every run was `77 passed, 1 skipped`. What *is* directly observed is the machinery that makes such non-determinism possible: a **128-worker** xdist configuration collecting **78 test items**, and a **filesystem-backed classifier model** that leaks across tests when `MODEL_FILE` is not isolated per test. The further claim that this shared-state interleaving is the **root cause** of the reported run-to-run classification failures in the *broader* suite is an **(inferred)** hypothesis — it is labelled as such throughout and was **not** reproduced in the modules exercised here.

**Hypothesis (inferred).** The reported non-deterministic failures are most plausibly caused by the combination of **massively parallel pytest-xdist execution** (`--numprocesses auto`) and a **shared/non-isolated `MODEL_FILE`** across tests that do not use `DirectoriesMixin`. The classifier model is a **filesystem** artifact whose reuse-vs-retrain outcome depends on what already exists on disk (Q1); when two tests share one `MODEL_FILE` path and at least one of them writes it, a cross-worker read/write interleaving could make a later test's classification assertion pass or fail run-to-run. This causal step is **(inferred)** — no actual failing race was observed here.

### What was directly observed

**1. xdist parallelism — 128 workers, 78 collected items (observed).** `--numprocesses auto` resolves to the host CPU count (`nproc = 128`), and pytest-xdist creates that many workers. The verbose header shows the worker count and the **collected item count is 78** (which equals the `77 passed + 1 skipped` below) — the `78` is the number of test *items*, distinct from the `128` *workers*:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 mkdir -p /tmp/th/work && cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest documents/tests/test_classifier.py documents/tests/test_tasks.py documents/tests/test_matchables.py \
 --no-cov -p no:cacheprovider -o addopts="" -n auto -v > /tmp/th/work/q6_xdist_v.txt 2>&1'
```

```
$ # the saved -v log /tmp/th/work/q6_xdist_v.txt is 195 lines; showing the collection + xdist worker header (head -8) and the final summary line (tail -1)
$ head -8 /tmp/th/work/q6_xdist_v.txt
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
created: 128/128 workers
128 workers [78 items]
$ tail -1 /tmp/th/work/q6_xdist_v.txt
================= 77 passed, 1 skipped, 774 warnings in 27.09s =================
```

**2. Repeated identical runs — STABLE; non-determinism did NOT reproduce (observed).** Running the three classification modules together five times under the default `--numprocesses auto`, then three times pinned serial (`-n0`, labelled non-default), produced an identical `77 passed, 1 skipped` every time (reproduce-by-repetition, not stabilize-by-variant). Each line below is the verbatim final summary line of an independent full run:

```
# 5x identical PARALLEL (default -n auto) then 3x identical SERIAL (-n0), same three modules each time:
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest documents/tests/test_classifier.py documents/tests/test_tasks.py documents/tests/test_matchables.py \
 --no-cov -p no:cacheprovider -o addopts="" -n auto'   # (-n0 for the serial baseline)
```

```
$ # each line below is the verbatim final summary line (tail -1) of an independent full run;
$ # the full logs (q6_par_run*.txt ~ a few hundred lines each incl the 774-warning summary) were captured to disk.
$ for i in 1 2 3 4 5; do printf "PARALLEL(-n auto) run %s: " "$i"; tail -2 q6_par_run$i.txt | head -1; done
PARALLEL(-n auto) run 1: ================= 77 passed, 1 skipped, 774 warnings in 27.05s =================
PARALLEL(-n auto) run 2: ================= 77 passed, 1 skipped, 774 warnings in 27.13s =================
PARALLEL(-n auto) run 3: ================= 77 passed, 1 skipped, 774 warnings in 27.03s =================
PARALLEL(-n auto) run 4: ================= 77 passed, 1 skipped, 774 warnings in 27.06s =================
PARALLEL(-n auto) run 5: ================= 77 passed, 1 skipped, 774 warnings in 27.07s =================
$ for i in 1 2 3; do printf "SERIAL(-n0)      run %s: " "$i"; tail -2 q6_ser_run$i.txt | head -1; done
SERIAL(-n0)      run 1: ================== 77 passed, 1 skipped, 6 warnings in 7.35s ===================
SERIAL(-n0)      run 2: ================== 77 passed, 1 skipped, 6 warnings in 7.54s ===================
SERIAL(-n0)      run 3: ================== 77 passed, 1 skipped, 6 warnings in 7.45s ===================
```

**Honest result:** across 5 identical parallel runs + 3 identical serial runs, the reported non-determinism **did not reproduce** in these modules. (The ~27 s parallel vs ~7 s serial difference is 128-worker startup overhead, observed.)

**3. The non-isolated matching tests never *write* the shared model (observed).** Two structural facts (from source) frame why these modules stayed stable:
- `TestClassifier` (`src/documents/tests/test_classifier.py:L20`) and `TestTasks` (`src/documents/tests/test_tasks.py:L21`) both inherit **`DirectoriesMixin`**, which overrides `MODEL_FILE` to a per-test temp path `os.path.join(dirs.data_dir, "classification_model.pickle")` (`src/documents/tests/utils.py:L45`) — so each test's model is isolated.
- The plain-`TestCase` matching tests — `_TestMatchingBase` (`src/documents/tests/test_matchables.py:L19`), `TestCaseSensitiveMatching` (`src/documents/tests/test_matchables.py:L216`), and `TestDocumentConsumptionFinishedSignal` (`src/documents/tests/test_matchables.py:L380`) — use the **non-isolated default** `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` (`src/paperless/settings.py:L74`), but they dispatch matching **without a classifier**, so they never *write* that shared model. Observed directly by watching the shared path across a full matchables run:

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src; F=/app/data/classification_model.pickle; rm -f "$F"; \
 echo "before: exists=$([ -f "$F" ] && echo yes || echo no)"; \
 DJANGO_SETTINGS_MODULE=paperless.settings python3 -m pytest documents/tests/test_matchables.py \
   --no-cov -p no:cacheprovider -o addopts="" -n0 -q; \
 echo "after : exists=$([ -f "$F" ] && echo yes || echo no)"'
```

```
F=/app/data/classification_model.pickle
before matchables run: exists=no
matchables result: 15 passed, 6 warnings in 2.14s
after  matchables run: exists=no
```

The shared model file is absent both before and after the 15 matchables tests — confirming these tests read/dispatch but never persist a model (so, in isolation, they cannot drive the leakage).

**4. The cross-test model-file leakage mechanism, directly demonstrated (labelled characterization).** To exhibit the mechanism the hypothesis depends on, a temporary plain-`TestCase` (no `DirectoriesMixin`) pinned to a single fixed shared `MODEL_FILE` had an earlier test train+save the classifier and a *later* test (whose DB rows are rolled back by `TestCase`) attempt to load it. The probe source and its complete output:

```python
# Q6 determinism characterization probe (temporary; deleted after use).
# Demonstrates the leakage MECHANISM behind the pytest-xdist + shared MODEL_FILE
# hypothesis: a plain TestCase (NO DirectoriesMixin) pinned to one fixed shared
# MODEL_FILE. An earlier test trains + saves the classifier pickle; a LATER test
# (whose DB rows are rolled back by TestCase) still sees that pickle, because the
# MODEL_FILE is a FILESYSTEM artifact that is NOT rolled back with the DB.
import os

from django.conf import settings
from django.test import override_settings, TestCase

from documents import tasks
from documents.classifier import load_classifier
from documents.models import Correspondent, Document

SHARED = "/tmp/th/work/q6_shared_model.pickle"


@override_settings(MODEL_FILE=SHARED)
class Q6SharedModelLeakage(TestCase):
    """Plain TestCase => NO per-test MODEL_FILE isolation (unlike DirectoriesMixin)."""

    def _make_training_docs(self):
        c = Correspondent.objects.create(
            name="c1",
            matching_algorithm=Correspondent.MATCH_AUTO,
        )
        for i in range(3):
            Document.objects.create(
                title=f"d{i}",
                content=f"unique training content number {i} keyword",
                correspondent=c,
                checksum=f"q6ck{i}",
                mime_type="text/plain",
            )

    def test_a_train_writes_shared_model(self):
        # start from a clean shared path so the write is unambiguous
        if os.path.exists(SHARED):
            os.remove(SHARED)
        print(f"[char] settings.MODEL_FILE (pinned shared) = {settings.MODEL_FILE}")
        print(f"[char] test_a: shared MODEL_FILE exists BEFORE train = {os.path.exists(SHARED)}")
        self._make_training_docs()
        tasks.train_classifier()  # real task: trains on this test's docs + saves pickle
        print(f"[char] test_a: DB Document rows (this test) = {Document.objects.count()}")
        print(f"[char] test_a: shared MODEL_FILE exists AFTER train = {os.path.exists(SHARED)}")

    def test_z_later_test_sees_leftover(self):
        # NO training here. TestCase already rolled back test_a's DB rows.
        print(f"[char] test_z: DB Document rows (test_a's rows rolled back) = {Document.objects.count()}")
        clf = load_classifier()  # reads settings.MODEL_FILE
        print(
            f"[char] test_z: leftover model file on disk from PRIOR test = {os.path.exists(SHARED)} "
            f"| load_classifier() returned a model (not None) = {clf is not None}"
        )
```

```
docker exec --user testuser -e HOME=/tmp/th paperless-canon bash -lc '
 cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
 python3 -m pytest /tmp/th/work/blitzy_adhoc_test_q6.py \
 --no-cov -p no:cacheprovider -o addopts="" -n0 -s -v'
```

```
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /tmp/th/work
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 2 items

../../tmp/th/work/blitzy_adhoc_test_q6.py::Q6SharedModelLeakage::test_a_train_writes_shared_model Creating test database for alias 'default'...
[char] settings.MODEL_FILE (pinned shared) = /tmp/th/work/q6_shared_model.pickle
[char] test_a: shared MODEL_FILE exists BEFORE train = False
[2026-07-14 21:44:40,676] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/th/work/q6_shared_model.pickle...
[char] test_a: DB Document rows (this test) = 3
[char] test_a: shared MODEL_FILE exists AFTER train = True
PASSED
../../tmp/th/work/blitzy_adhoc_test_q6.py::Q6SharedModelLeakage::test_z_later_test_sees_leftover [char] test_z: DB Document rows (test_a's rows rolled back) = 0
[char] test_z: leftover model file on disk from PRIOR test = True | load_classifier() returned a model (not None) = True
PASSEDDestroying test database for alias 'default'...


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
======================== 2 passed, 6 warnings in 1.94s =========================
PYTEST_EXIT=0
```

`test_a` trained on 3 documents and wrote the shared pickle (`exists BEFORE = False` → `AFTER = True`). `test_z`, which trains nothing and whose predecessor's DB rows were rolled back (`DB Document rows = 0`), nonetheless finds the **leftover model still on disk** and `load_classifier()` returns a non-`None` model. This is the cross-test leakage: the pickle is a **filesystem** artifact that — unlike the DB transaction — is **not** rolled back between tests.

### Conclusion — observed vs inferred

- **Directly observed:** the reuse/retrain SHA-1 mechanism (Q1, `src/documents/classifier.py:L163-L164`); the 128-worker / 78-item xdist configuration; the stable `77 passed, 1 skipped` distribution across 8 identical runs (non-reproduction); that the matchables tests never write the shared model; and the cross-test model-file leakage mechanism (a leftover pickle visible to a later test).
- **Inferred (labelled, NOT reproduced here):** that in the *broader* suite — where non-isolated tests that *do* write the default `MODEL_FILE` can interleave with classifier assertions across 128 workers — this shared-state race is the root cause of the reported run-to-run classification failures. The isolated classification modules exercised above did not trigger it, because they either isolate `MODEL_FILE` (`DirectoriesMixin`) or never write the shared file (`classifier=None`). The leap from "leakage mechanism exists" to "this is the root cause of the reported failures" is **(inferred)**; likewise the claim that the 128-worker interleaving manifests as a genuine failing race is **(inferred)** — neither was observed to fail here.

**Recommended framing for maintainers (documentation only, no code change per scope, per AAP §0.3).** Ensuring every classification-touching test uses `DirectoriesMixin` (or otherwise isolates `MODEL_FILE` per test/worker) would remove the shared on-disk model state that this investigation identifies as the determinism-risk surface. This is a framing suggestion only; per the read-only scope no source, test, or configuration file was changed.

---

## Appendix — Read-Only Invariant (source tree unchanged)

This investigation is **read-only** against the paperless-ngx source tree (AAP §0.3.1: "Read-only source"). Every runtime observation was produced inside the canonical container `paperless-canon`; the six temporary observation scripts —

- `/tmp/th/work/blitzy_adhoc_test_q1.py` (Q1 reuse/retrain)
- `/tmp/th/work/blitzy_adhoc_test_q2.py` (Q2 count/timing/threshold)
- `/tmp/th/work/blitzy_adhoc_test_q3.py` (Q3 no-text parser)
- `/tmp/th/work/blitzy_adhoc_test_q3consume.py` (Q3 consumer/MIME + thumbnail)
- `/tmp/th/work/blitzy_adhoc_test_q4.py` (Q4 barcode split/consume chain)
- `/tmp/th/work/blitzy_adhoc_test_q6.py` (determinism / model-file leakage)

— lived in the container scratch directory `/tmp/th/work` (mode `0700`, outside both git trees) and were **deleted after use**, together with the one artifact they wrote outside the runtime dirs (`/tmp/th/work/q6_shared_model.pickle`). The only change introduced to the destination repository is this single answer document.

### A.1 Canonical source checkout (`/app` in the container) — pristine after cleanup

The following proof was captured **after** all temporary scripts and all gitignored runtime artifacts (`consume/` split PDFs, `data/db.sqlite3`, `data/log/`, `media/media.lock`, `src/.pytest_cache/`) were removed. It shows the tracked working tree is empty of changes, the ignored listing is empty of residuals, no probe scripts remain, and — critically — the one binary fixture the Q3 no-text path touches is byte-for-byte identical to `HEAD`:

```console
$ git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542

$ git -C /app status --porcelain          # TRACKED working tree
(empty == every tracked source file byte-for-byte identical to HEAD)

$ git -C /app status --porcelain --ignored # incl. gitignored runtime dirs
(empty == no residual investigation artifacts remain)

$ ls -A /tmp/th/work                       # temporary probe workspace
(empty == all temporary observation scripts + q6_shared_model.pickle removed)

$ find /tmp /app -name blitzy_adhoc_test_* # residual probe scan
(no output == none remain)

$ git -C /app hash-object src/paperless_tesseract/tests/samples/no-text-alpha.png
e78b22bfbe53be5a9046bbfb32fef7e98abb9be9
$ git -C /app ls-tree HEAD src/paperless_tesseract/tests/samples/no-text-alpha.png
100644 blob e78b22bfbe53be5a9046bbfb32fef7e98abb9be9	src/paperless_tesseract/tests/samples/no-text-alpha.png
(identical hashes == the no-text OCR fixture was NOT mutated by Q3)
```

The Q3 fixture integrity result deserves emphasis: `git hash-object` of the working-tree `no-text-alpha.png` equals the `HEAD` blob hash (`e78b22bfbe53be5a9046bbfb32fef7e98abb9be9`) with **no restoration step required**. This is because the corrected Q3 methodology parsed an **out-of-tree copy** of the fixture (checksummed before/after in the Q3 section), so `ocrmypdf`/PIL rewrote only the disposable copy — the tracked fixture was never opened for writing. A checksum sweep of all eight Q1–Q4 source modules likewise reports `MATCH` against `HEAD`, confirming no source, test, or configuration file under investigation was modified.

### A.2 Destination repository — only the deliverable changed

In the destination checkout on branch `blitzy-be1effe9-f4e7-4a8e-a3a8-33c507232b12`, captured after authoring and cleanup and immediately before committing this deliverable:

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-be1effe9-f4e7-4a8e-a3a8-33c507232b12

$ git status --porcelain --untracked-files=all   # all changes incl. untracked
 M blitzy/documentation/paperless-ngx_542221a38dff.md

$ git status --porcelain --untracked-files=no | wc -l   # tracked-file modifications
1
```

The single porcelain entry is the answer document `blitzy/documentation/paperless-ngx_542221a38dff.md` itself, and the count of tracked-file modifications is `1` — that one file being this deliverable. No source, test, configuration, or fixture file was added, modified, or deleted in the destination repository.

Commit under investigation (unchanged): `542221a38dff06361e07976452f9aea24d210542`.
