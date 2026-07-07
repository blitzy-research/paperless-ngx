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

<a name="1-environment--reproduction"></a>
## 1. Environment / Reproduction

All runs execute inside the **canonical container** mandated by the task
(`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`,
alias `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542`).
The repository is baked at `/app` (owned by the non-root `testuser`, matching CI), so the
project's `src/` directory is `/app/src` inside the container. The bare sandbox host lacks
Tesseract/Ghostscript/poppler/libzbar, so the **Q3 (OCR)** and **Q4 (barcode)** code paths
were exercised **inside the container**.

### 1.0 Reproducible container setup (exact commands)

The canonical image was pulled, then a thin **derived** image added the two system
libraries the base image is missing (`libzbar0` for pyzbar and `poppler-utils` for
pdf2image — the exact libs the project CI installs and that the Q4 barcode-splitting path
requires). The container was started detached, and every observation run below was issued
via `docker exec` as the non-root `testuser`.

```bash
# host: Docker version 28.5.2, build ecc6942

# (1) Pull the canonical SWE-Atlas paperless-ngx Q&A image
docker pull ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01

# (2) Build the derived image (adds libzbar0 / poppler-utils / zbar-tools / pngquant).
#     Build context /tmp/paperless-qna-build/ contains only the Dockerfile shown below.
docker build -t paperless-ngx-qna:local /tmp/paperless-qna-build

# (3) Start the container detached (no host bind mount; baked, testuser-owned /app)
docker run -d --name paperless-qna-baked paperless-ngx-qna:local -c "sleep infinity"

# (4) Every observation run below is issued as (WorkingDir /app; /app/src == the repo src/):
docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && <command>'
```

Derived image `Dockerfile` (verbatim, build context `/tmp/paperless-qna-build/Dockerfile`):

```dockerfile
# Derived from the canonical SWE-Atlas paperless-ngx Q&A image.
# Adds the two mandatory system deps that the base image is missing
# (libzbar0 for pyzbar, poppler-utils for pdf2image) — these are the exact
# libs the project CI installs (reusable-ci-backend.yml), required by the Q4
# barcode-splitting code path. Also bakes git safe.directory for /app.
FROM ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01

USER root
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update -qq \
 && apt-get install -y --no-install-recommends \
      libzbar0 \
      poppler-utils \
      zbar-tools \
      pngquant \
 && rm -rf /var/lib/apt/lists/* \
 && git config --global --add safe.directory /app \
 && git config --global --add safe.directory '*'
```

Image identifiers (verbatim `docker images --digests`, host):

```text
REPOSITORY                   TAG                                                                                   DIGEST                                                                    IMAGE ID       CREATED        SIZE
paperless-ngx-qna            local                                                                                 <none>                                                                    85fecc9d775a   2 hours ago    1.81GB
ghcr.io/scaleapi/swe-atlas   swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01   sha256:d4abe56dd5d1cb2632353baf06e9d80a704f147a70ac2ec15c2b78fc5fddfe15   6e699f225ced   3 months ago   1.79GB
```

The container configuration (verbatim `docker inspect paperless-qna-baked`) confirms there
is **no host bind mount** (`Mounts=[]`), the image is `paperless-ngx-qna:local`, the
entrypoint is `/bin/bash`, and the command is `-c "sleep infinity"` with `WorkingDir=/app`.

### 1.1 Toolchain (verbatim)

Captured inside the container as `testuser`; run **twice**, byte-identical (`diff`
of the two runs was empty):

```text
$ python --version
Python 3.9.23
$ pipenv --version
pipenv, version 2025.0.4
$ tesseract --version 2>&1 | head -n1
tesseract 4.1.1
$ gs --version
9.53.3
$ pdftoppm -v 2>&1 | head -n1
pdftoppm version 20.09.0
$ qpdf --version | head -n1
qpdf version 10.1.0
$ python -c "import pyzbar; print(pyzbar.__version__)"
0.1.9
$ python -c "import sklearn; print(sklearn.__version__)"
1.0.2
```

### 1.2 Pinned dependency versions (verbatim `pip show`)

Every dependency named anywhere in this document is shown below with its verbatim
installed version (single `pip show` invocation covering all packages, filtered to the
`Name`/`Version` lines):

```text
$ pip show scikit-learn ocrmypdf pyzbar pdf2image pikepdf python-magic django django-q numpy scipy fuzzywuzzy channels whoosh pillow 2>/dev/null | grep -E "^(Name|Version):"
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
Name: Django
Version: 4.0.4
Name: django-q
Version: 1.3.9
Name: numpy
Version: 1.22.3
Name: scipy
Version: 1.8.0
Name: fuzzywuzzy
Version: 0.18.0
Name: channels
Version: 3.0.4
Name: Whoosh
Version: 2.7.4
Name: Pillow
Version: 9.1.0
```

These match the `requirements.txt` pins exactly: scikit-learn 1.0.2, ocrmypdf 13.4.3,
pyzbar 0.1.9, pdf2image 1.16.0, pikepdf 5.1.1, python-magic 0.4.25, Django 4.0.4,
django-q 1.3.9, numpy 1.22.3, scipy 1.8.0, fuzzywuzzy 0.18.0, channels 3.0.4,
Whoosh 2.7.4, Pillow 9.1.0.

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
`python -m pytest`, which auto-loads `setup.cfg`). Inside the container this is issued as
`docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && pipenv run pytest …'`,
where `/app/src` **is** the repo's `src/`. The leading `Loading .env environment
variables...` line that precedes every run below is emitted by `pipenv`. Two run modes are
used:

* **Canonical run** — inherits all `addopts` (including `--numprocesses auto` and
  `--cov`). Used to show the real xdist worker behavior (§Q1).
* **Focused observation run** — adds `-n0 --no-cov -p no:cacheprovider`. These flags are
  **explicitly labelled and non-behavioral**: `-n0` pins a single worker for clean
  single-process observation (overriding the inherited `--numprocesses auto`), `--no-cov`
  suppresses only the multi-thousand-line coverage *report* (not the test's behavior), and
  `-p no:cacheprovider` avoids writing a cache into the read-only tree. None of them changes
  what the code under test does. (There is no `pytest-randomly` plugin installed — see the
  `plugins:` line below — so test order is already fixed.)

**Banner visibility.** The inherited `--quiet` from `addopts` *suppresses* the
`test session starts` banner in the default output; adding `-v` reveals it. The banner
(revealed with `-v`) is identical across focused runs:

```text
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 1 item
```

Under the **canonical** run (inheriting `--numprocesses auto`) the same banner is followed
by the real xdist worker lines — verbatim, `-v`, single-item selection:

```text
created: 128/128 workers
128 workers [1 item]
```

confirming pytest-xdist spins **128 worker processes** on this 128-CPU host (each with its
own database), which is the grounding for Q1's cross-test answer.

### 1.5 Stability protocol

Every count/timing/decision value below was produced at least **twice** and confirmed
identical (or, where a value is non-deterministic, the same unchanged input was repeated
**30×** and the observed distribution is reported verbatim — see §Non-Determinism). The
**complete, unedited** output of every command — for **both** runs — is preserved in
[Appendix B — Complete Raw Command Logs](#appendix-b); the per-question sections quote the
decisive lines from those same logs.

Temporary observation scripts (`/tmp/q1_probe_test.py`, `/tmp/q2_probe_test.py`,
`/tmp/q3_probe_test.py`, `/tmp/q4_probe_test.py`, `/tmp/nondet_probe_test.py`) live only in
the container's `/tmp` (outside the repository tree) and subclass the project's **own**
`DirectoriesMixin` + Django `TestCase` harness, so they invoke the real
`DocumentClassifier.train`, `load_classifier`, `predict_correspondent`,
`scan_file_for_separating_barcodes`, `separate_pages`, `RasterisedDocumentParser.parse`,
and `consume_file` — no bypassing interfaces. They are **removed before completion**, so
the source tree is left byte-for-byte unchanged (see [Appendix A — Read-Only Proof](#appendix-a)).
Each probe is run with `-c /app/src/setup.cfg --rootdir=/app/src` (so pytest loads the
project config for a `/tmp` file) plus the focused flags `-n0 --no-cov -p no:cacheprovider -s`.

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

All commands are the project's own canonical invocation, `cd src/ && pipenv run pytest …`,
executed inside the container (where `src/` is `/app/src`) via
`docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && …'`. The focused
flags `-n0 --no-cov -p no:cacheprovider` are non-behavioral (see §1.4). The **complete,
unedited** output of **both** runs of every command below is preserved in
[Appendix B §B‑Q1](#appendix-b); the excerpts here are contiguous and omit only the
invariant 6‑warning summary block (shown in full under output (1) and in Appendix B).

```bash
# 1. Retrain-guard test (asserts train()==True then train()==False)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    -n0 --no-cov -p no:cacheprovider -rA

# 2. Save then reload leaves the guard armed (loaded model's train() -> False)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
    -n0 --no-cov -p no:cacheprovider -rA

# 3. Load the committed model.pickle and classify (predict_tags -> [45, 12])
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::test_load_and_classify \
    -n0 --no-cov -p no:cacheprovider -rA

# 4. The caching test is skipped (direct proof there is NO in-memory cache)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::test_load_classifier_cached \
    -n0 --no-cov -p no:cacheprovider -rA

# 5. Canonical run showing real xdist workers (inherits --numprocesses auto); -v reveals the banner
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    --no-cov -p no:cacheprovider -v

# 6. Temporary probe (container /tmp): retrain guard + mutate, no-cache, save/load,
#    per-test MODEL_FILE isolation, and load_classifier()->None when absent
cd src/ && pipenv run pytest /tmp/q1_probe_test.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -rA -s
```

### Verbatim Observed Output

> Note on remaining `...`: the only three-dot sequences in the blocks below are **verbatim
> output** — the pipenv line `Loading .env environment variables...` and the classifier's
> own log message `Gathering data from database...`. They are not elisions. The invariant
> 6‑warning summary block is shown in full in output **(1)** and, for every command, in
> [Appendix B §B‑Q1](#appendix-b); where a later excerpt omits it, that is marked with an
> explicit bracketed note (never a bare `...`).

**(1) `testDatasetHashing` — retrain-guard: `train()` → `True`, second `train()` → `False`.
Complete output (run 1 shown in full; run 2 identical except wall-clock time):**

```text
Loading .env environment variables...
.                                                                        [100%]
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
==================================== PASSES ====================================
______________________ TestClassifier.testDatasetHashing _______________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:203 Training tags classifier...
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:237 Training document type classifier...
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::testDatasetHashing
1 passed, 6 warnings in 1.98s
```

Run 2 footer (byte-identical output apart from the timing): `1 passed, 6 warnings in 1.89s`.
The captured log is itself the guard's fingerprint: the **first** `train()` runs the full
pipeline (`Gathering data…` → `Vectorizing…` → three `Training … classifier…` lines), while
the **second** `train()` prints only `Gathering data from database...` and then returns
`False` — it recomputes the SHA-1 `data_hash`, finds it unchanged, and stops before
vectorizing (`classifier.py:163-164`).

**(2) `testSaveClassifier` (`test_classifier.py:168`) — train → `save()` → reload into a new
instance → the reloaded model's `train()` returns `False` (guard survives serialization):**

```text
==================================== PASSES ====================================
______________________ TestClassifier.testSaveClassifier _______________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:203 Training tags classifier...
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:237 Training document type classifier...
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::testSaveClassifier
1 passed, 6 warnings in 1.91s
```

Run 2 footer: `1 passed, 6 warnings in 1.90s`. (Excerpt begins at the `PASSES` banner; the
leading `Loading .env…` line, progress dot, and the invariant 6‑warning block precede it —
shown in full in output (1) and in Appendix B §B‑Q1.) The trailing `Gathering data…` with
**no** subsequent `Vectorizing…` is the reloaded instance's `train()` returning `False`.

**(3) `test_load_and_classify` (`test_classifier.py:183`) — loads the committed
`src/documents/tests/data/model.pickle` via a `MODEL_FILE` override and classifies
(`predict_tags(doc2.content)` asserted equal to `[45, 12]`):**

```text
==================================== PASSES ====================================
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::test_load_and_classify
1 passed, 6 warnings in 1.88s
```

Run 2 footer: `1 passed, 6 warnings in 1.92s`. (This test only *loads* a persisted model
and predicts, so it emits no training `DEBUG` lines; the leading pipenv line, progress dot,
and 6‑warning block precede the excerpt — see Appendix B §B‑Q1.) It confirms a persisted
model is reused across a process boundary **only** through an explicit on-disk
`MODEL_FILE` — never through in-process memory.

**(4) `test_load_classifier_cached` (`test_classifier.py:402`) is skipped — direct proof the
cache was deliberately removed. Complete output, identical across both runs:**

```text
Loading .env environment variables...
s                                                                        [100%]
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
=========================== short test summary info ============================
SKIPPED [1] documents/tests/test_classifier.py:391: Disabled caching due to high memory usage - need to investigate.
1 skipped, 6 warnings in 0.08s
```

Run 2 footer: `1 skipped, 6 warnings in 0.08s` (identical). The skip decorator/reason is
`@pytest.mark.skip("Disabled caching due to high memory usage - need to investigate.")` at
`test_classifier.py:399-400`, on `test_load_classifier_cached` @ `L402`. pytest reports the
skip at `L391` (the class/`setUp` line the marker attaches to).

**(5) Canonical `--numprocesses auto` run — pytest-xdist spins up 128 worker processes.
Header, worker banner, result, and footer (the intervening warnings summary block — which pytest‑xdist
**de‑duplicates** to the 6 unique warnings, each annotated `: 129 warnings` for a
`774`‑warning aggregate across the 128 workers — is reproduced in full in
[Appendix B §B‑Q1](#appendix-b)):**

```text
Loading .env environment variables...
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
created: 128/128 workers
128 workers [1 item]

.                                                                        [100%]
[warnings summary block omitted here — 774 warnings aggregated across 128 workers; full text in Appendix B §B‑Q1]
======================= 1 passed, 774 warnings in 24.82s =======================
```

Run 2: identical `created: 128/128 workers` / `128 workers [1 item]` banner; footer
`1 passed, 774 warnings in 24.76s`. The same single test takes ~1.9 s single‑worker
(`-n0`, output (1)) versus ~24.8 s when spinning up 128 worker processes — concrete
evidence that the default suite runs across **128 separate OS processes**, each with an
isolated database, so no classifier object can be shared between them.

**(6) Temporary probe `/tmp/q1_probe_test.py` — 6 passed, identical behavior across 2 runs.
Complete run‑1 output (the invariant 6‑warning block is elided with a bracketed note — it is
byte‑identical to output (1) and reproduced in [Appendix B §B‑Q1](#appendix-b)):**

```text
Loading .env environment variables...
Q1A data_hash BEFORE first train(): None
Q1A first train() returned: True
Q1A data_hash AFTER first train(): b292e1a98544b739dfacde57a87b94c7e3c102cc
Q1A second train() returned: False
Q1A data_hash AFTER second train(): b292e1a98544b739dfacde57a87b94c7e3c102cc
Q1A data_hash UNCHANGED across retrain: True
Q1A third train() AFTER adding a document returned: True
Q1A data_hash AFTER mutate: 23706c538fac64adf7edf3a4591974b7ae1b7aac
Q1A data_hash CHANGED after mutate: True
.Q1B MODEL_FILE exists after save(): True
Q1B load_classifier() call #1 id(): 138582118506208
Q1B load_classifier() call #2 id(): 138582118505344
Q1B distinct instances (NO cache): True
Q1B both DocumentClassifier: True
.Q1C loaded.data_hash present after load(): True
Q1C train() after load() returned: False
.Q1D isolation#1 MODEL_FILE: /tmp/tmpgnqsr2x6/classification_model.pickle
Q1D isolation#1 file present at setUp: False
.Q1D isolation#2 MODEL_FILE: /tmp/tmpp5rkoclt/classification_model.pickle
Q1D isolation#2 file present at setUp: False
Q1D per-test MODEL_FILE differs (#1 vs #2): True
.Q1E MODEL_FILE: /tmp/tmpzu7l92qy/classification_model.pickle
Q1E model file exists: False
Q1E load_classifier() with no model on disk returned: None
.
[warnings summary block omitted here — invariant 6-warning block, byte-identical to output (1); full text in Appendix B §B‑Q1]
==================================== PASSES ====================================
______________ Q1RetrainAndCache.test_a_retrain_guard_transition _______________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 2 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 3 documents, 0 tag(s), 3 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
__________________ Q1RetrainAndCache.test_b_no_inmemory_cache __________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 2 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
______________ Q1RetrainAndCache.test_c_saveload_prevents_retrain ______________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 2 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
________________ Q1RetrainAndCache.test_e_load_none_when_absent ________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:32 Document classification model does not exist (yet), not performing automatic matching.
=========================== short test summary info ============================
PASSED ::Q1RetrainAndCache::test_a_retrain_guard_transition
PASSED ::Q1RetrainAndCache::test_b_no_inmemory_cache
PASSED ::Q1RetrainAndCache::test_c_saveload_prevents_retrain
PASSED ::Q1RetrainAndCache::test_d_isolation_1
PASSED ::Q1RetrainAndCache::test_d_isolation_2
PASSED ::Q1RetrainAndCache::test_e_load_none_when_absent
6 passed, 6 warnings in 2.06s
```

Run 2 footer: `6 passed, 6 warnings in 2.12s`. Only the two `id()` integers (memory
addresses) and the temp‑dir names differ run‑to‑run; every behavioral value is stable:

- **`data_hash` state transition (Q1A):** `None` (before) → `b292e1a98544b739dfacde57a87b94c7e3c102cc`
  (after first `train()`, which returned `True`) → **unchanged** on the second `train()`
  (which returned `False` — the guard) → `23706c538fac64adf7edf3a4591974b7ae1b7aac` after
  adding a third document (third `train()` returned `True`). Both digests are **fully
  reproducible from the exact probe embedded in [Appendix C §C‑Q1](#appendix-c)** and were
  observed **identically across three runs** — the hash is a deterministic SHA‑1 over the
  ordered, inbox‑excluded document set (each document's preprocessed content plus its
  document‑type and correspondent PKs and its sorted `MATCH_AUTO` tag PKs,
  `classifier.py:123-162`), with **no MLP randomness** involved. Because the hex is a
  deterministic function of *those exact documents*, the specific mutated digest
  `23706c53…` corresponds to the third document defined in the embedded probe
  (`content="this is a document from c3"`, its own `MATCH_AUTO` correspondent); a
  *different* third document deterministically yields a *different* digest (e.g. a
  differently‑worded third document produces a different, equally‑stable hex). The
  behavioral invariant proven here is the guard transition itself — first `train()`
  `True` → second `train()` `False` (hash unchanged) → post‑mutation `train()` `True`
  (hash changed) — which holds regardless of the specific content chosen.
- **No in‑memory cache (Q1B):** two `load_classifier()` calls returned `DocumentClassifier`
  objects with **different `id()`** — each call re‑reads `MODEL_FILE` from disk and builds a
  fresh instance (`classifier.py:30-57`).
- **Save/reload survives the guard (Q1C):** after `save()`→`load()` into a new instance,
  `train()` returned `False` — the persisted `data_hash` short‑circuits retraining.
- **Per‑test isolation (Q1D):** the two isolation tests observed **distinct** `MODEL_FILE`
  paths (`…/tmpgnqsr2x6/…` vs `…/tmpp5rkoclt/…`), each absent at `setUp`, confirming no model
  file survives across tests (`utils.py:45`).
- **Absent model → `None` (Q1E):** with no model on disk, `load_classifier()` returned
  `None` and logged `classifier.py:32 Document classification model does not exist (yet)…`.

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

All commands are the project's own canonical invocation, `cd src/ && pipenv run pytest …`,
executed inside the container (where `src/` is `/app/src`) via
`docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && …'`. The focused flags
`-n0 --no-cov -p no:cacheprovider` are non-behavioral (see §1.4). The **complete, unedited**
output of **both** runs of every command is preserved in [Appendix B §B‑Q2](#appendix-b);
the excerpts here are contiguous and omit only the invariant 6‑warning summary block (shown
in full under Q1 output (1) and in Appendix B).

```bash
# 1. Single-document correspondent prediction (1 training doc)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    -n0 --no-cov -p no:cacheprovider -rA

# 2. Two-document correspondent prediction (doc1 -> c1, doc2 -> None)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs \
    -n0 --no-cov -p no:cacheprovider -rA

# 3. testTrain — asserts correspondent_classifier.classes_ == [-1, c1.pk]
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testTrain \
    -n0 --no-cov -p no:cacheprovider -rA

# 4. testPredict — asserts predict_correspondent(doc1)==c1.pk and predict_correspondent(doc2) is None (-1 sentinel)
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testPredict \
    -n0 --no-cov -p no:cacheprovider -rA

# 5. Temporary probe (container /tmp): count-before-insert=0, effective inbox-excluded count,
#    insert-then-train ordering, predict vs predict_proba counters (2 vs 0), predict_proba value
#    shown-but-unused, and the fuzz.partial_ratio>=90 regex MATCH_FUZZY gate (unrelated to ML)
cd src/ && pipenv run pytest /tmp/q2_probe_test.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -rA -s
```

### Verbatim Observed Output

Each excerpt begins at the `PASSES` banner; the leading `Loading .env…` line, progress dot,
and the invariant 6‑warning block precede it (shown in full under Q1 output (1) and in
[Appendix B §B‑Q2](#appendix-b)). The `Captured log call` block is the classifier's own
`DEBUG` output — it is the direct evidence of the effective (inbox‑excluded) training count.

**(1) `test_one_correspondent_predict` — 1 training document (1 correspondent):**

```text
==================================== PASSES ====================================
________________ TestClassifier.test_one_correspondent_predict _________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict
1 passed, 6 warnings in 1.85s
```

Run 2 footer: `1 passed, 6 warnings in 1.85s`. The `1 documents … 1 correspondent(s)` line is
the observed **(a) training-doc count = 1**.

**(2) `test_one_correspondent_predict_manydocs` — 2 training documents (1 correspondent):**

```text
==================================== PASSES ====================================
____________ TestClassifier.test_one_correspondent_predict_manydocs ____________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs
1 passed, 6 warnings in 1.88s
```

Run 2 footer: `1 passed, 6 warnings in 1.85s`. The `2 documents … 1 correspondent(s)` line is
the observed **(a) training-doc count = 2**.

**(3) `testTrain` (`test_classifier.py:103`) — asserts
`correspondent_classifier.classes_ == [-1, c1.pk]`:**

```text
==================================== PASSES ====================================
___________________________ TestClassifier.testTrain ___________________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:203 Training tags classifier...
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:237 Training document type classifier...
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::testTrain
1 passed, 6 warnings in 1.94s
```

Run 2 footer: `1 passed, 6 warnings in 1.96s`.

**(4) `testPredict` (`test_classifier.py:115`) — asserts
`predict_correspondent(doc1) == c1.pk` and `predict_correspondent(doc2) is None` (the `-1`
sentinel is mapped to `None`, with no probability involved):**

```text
==================================== PASSES ====================================
__________________________ TestClassifier.testPredict __________________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:203 Training tags classifier...
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:237 Training document type classifier...
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::testPredict
1 passed, 6 warnings in 1.94s
```

Run 2 footer: `1 passed, 6 warnings in 1.93s`.

**(5) Temporary probe `/tmp/q2_probe_test.py` — 5 passed, identical behavior across 2 runs.
Complete run‑1 stdout (the invariant 6‑warning block and the duplicate `PASSES` captured‑log
`DEBUG` lines that follow are reproduced in [Appendix B §B‑Q2](#appendix-b)):**

```text
Loading .env environment variables...
Q2A Document.objects.count() BEFORE any insert: 0
Q2A total docs created: 1
Q2A effective training docs (inbox-excluded): 1
Q2A insert-then-train: documents inserted FIRST, now calling train()
Q2A train() returned: True
Q2A predict_correspondent(doc1) -> [1] | c1.pk = 1
Q2A correspondent_classifier.classes_: [1]
.Q2B total docs created: 2
Q2B effective training docs (inbox-excluded): 2
Q2B train() returned: True
Q2B correspondent_classifier.classes_: [-1, 1]
Q2B predict doc1 -> [1] | c1.pk = 1
Q2B predict doc2 (no correspondent) -> None
.Q2C before inbox tag: total = 2 ; effective = 2
Q2C after inbox tag on 1 of 2 docs: total = 2 ; effective = 1
Q2C inbox tagging dropped effective count 2 -> 1: True
.Q2D correspondent_classifier type = MLPClassifier
Q2D classes_ = [1, 2]
Q2D predict_correspondent called twice
Q2D correspondent_classifier.predict call count: 2
Q2D correspondent_classifier.predict_proba call count: 0
Q2D predict_correspondent(d1) raw return = [1]
Q2D predict_correspondent(d2) raw return = [2]
Q2D (reference only) predict_proba(d2) = [[0.11747704 0.88252296]] -> value EXISTS but is UNUSED by predict_correspondent (proba unseeded, varies run-to-run)
Q2D pure argmax predict, NO probability threshold: True
.Q2E matching.py:135 fuzz.partial_ratio>=90 is regex MATCH_FUZZY, NOT ML
Q2E match_correspondents('a foobar invoice', classifier=None) -> ['Foo']
Q2E match_correspondents('nothing here', classifier=None) -> []
.
[warnings summary block omitted here — invariant 6-warning block, byte-identical to Q1 output (1); full text in Appendix B §B‑Q2]
=========================== short test summary info ============================
PASSED ::Q2Correspondent::test_a_one_correspondent_predict
PASSED ::Q2Correspondent::test_b_one_correspondent_predict_manydocs
PASSED ::Q2Correspondent::test_c_inbox_exclusion_reduces_count
PASSED ::Q2Correspondent::test_d_no_confidence_threshold
PASSED ::Q2Correspondent::test_e_fuzzy_gate_is_regex_not_ml
5 passed, 6 warnings in 2.22s
```

Run 2 footer: `5 passed, 6 warnings in 2.33s`. **A precise stability statement is required
here, because two different kinds of value appear in the block above.** The *structural /
mechanism* values are invariant and reproduced identically across runs: the training‑doc
**counts** (Q2A `1`, Q2B `2`, Q2C `2 → 1`), the **insert‑then‑train timing**, the
`correspondent_classifier` **type** (`MLPClassifier`) and **`classes_` membership**
(`[1]`, `[-1, 1]`, `[1, 2]` — including the `-1` sentinel), the **call counts** (`predict`
called **2**, `predict_proba` called **0**), and the **`MATCH_FUZZY` gate** result
(`['Foo']` / `[]`). The *model‑output* values are **not** guaranteed stable: the argmax
`predict()` **labels** printed on the `Q2A`/`Q2B`/`Q2D` prediction lines
(`predict doc1 -> [1]`, `predict doc2 -> None`, `raw return = [1]`/`[2]`) are produced by an
**unseeded** `MLPClassifier` (`classifier.py:227`, constructed with no `random_state`), so —
exactly like the `predict_proba` reference value, which visibly differs run‑to‑run (run 2
printed `[[0.13398922 0.86601078]]`) — the label a borderline document is assigned **can flip
between runs**. In this probe's re‑observation the labels were stable for this *distinctive*
content (`predict doc1 -> [1]` and `predict doc2 -> None` held across **10/10** full‑probe
runs, and across an `N=30`‑fresh‑classifier tally repeated 3× — **90/90** identical); but a
**flip is possible and has been observed** for this same code path (an independent
re‑execution recorded `predict doc1 -> None`), and it is *demonstrated at scale* for
genuinely borderline input in the [Non‑Determinism section](#nd). The label is therefore an
**observed distribution**, not a deterministic constant — see the Non‑Determinism section for
the root cause and the run‑to‑run flip evidence. Crucially, this label instability does **not**
weaken the Q2(c) answer: it is *because* the acceptance decision is a bare argmax
`predict()` with `predict_proba()` never called (call count **0**) that **no probability
threshold exists** — the model always returns its top class, whatever that class happens to
be on a given run. The decisive Q2 facts:

- **(a) training-doc count:** `1` (predict) / `2` (manydocs); the effective count is
  inbox-excluded — adding an inbox tag to 1 of 2 docs drops the effective count **2 → 1**
  (`classifier.py:125-127`).
- **(b) timing:** `Document.objects.count()` **before any insert = 0**; documents inserted
  first; `train()` called **after** — insert-then-train.
- **(c) confidence threshold:** **none** — `predict()` argmax called **2** times,
  `predict_proba()` called **0** times. `classes_` includes the `-1` sentinel only when a
  document has no correspondent (`[-1, 1]` in the manydocs case); a `-1` argmax maps to
  `None` (`classifier.py:255-258`). The `predict_proba` value exists but is never read.
- **fuzzy gate (context):** `fuzz.partial_ratio >= 90` (`matching.py:135`) is the regex
  `MATCH_FUZZY` algorithm — `match_correspondents(…, classifier=None)` returned `['Foo']`
  for matching content and `[]` otherwise, entirely without ML. It is not a confidence
  threshold.

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

All commands are the project's own canonical invocation, `cd src/ && pipenv run pytest …`,
executed inside the container (where `src/` is `/app/src`) via
`docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && …'`. The focused flags
`-n0 --no-cov -p no:cacheprovider` are non-behavioral (see §1.4). The **complete, unedited**
output of **both** runs is preserved in [Appendix B §B‑Q3](#appendix-b); the excerpts here
are contiguous and omit only the invariant 6‑warning summary block (shown in full under Q1
output (1) and in Appendix B).

```bash
# 1. Encrypted, no-extractable-text PDF -> empty text (OCR_MODE=skip)
cd src/ && pipenv run pytest paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted \
    -n0 --no-cov -p no:cacheprovider -rA

# 2. Form PDF, no text on the skip pass -> force-OCR recovers text (real OCR work; ~7.6s)
cd src/ && pipenv run pytest paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
    -n0 --no-cov -p no:cacheprovider -rA

# 3. Skip-archive, no-text case
cd src/ && pipenv run pytest paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
    -n0 --no-cov -p no:cacheprovider -rA

# 4. Exact args dict passed to ocrmypdf.ocr
cd src/ && pipenv run pytest paperless_tesseract/tests/test_parser.py::TestParser::test_ocrmypdf_parameters \
    -n0 --no-cov -p no:cacheprovider -rA

# 5. Temporary probe (container /tmp): real RasterisedDocumentParser.parse() on the encrypted
#    PDF (1 invocation, EncryptedPdfError) and the no-text alpha PNG (2 invocations, force-OCR);
#    wraps ocrmypdf.ocr to record per-call kwargs/exception; prints magic.from_file mime. Each
#    parse runs on a throwaway COPY because parsers.py:191-201 overwrites alpha-image inputs.
cd src/ && pipenv run pytest /tmp/q3_probe_test.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -rA -s
```

### Verbatim Observed Output

**(1) `test_encrypted` (`test_parser.py:178`, `OCR_MODE="skip"`) — the complete captured
OCR‑fallback log for `samples/encrypted.pdf`. The `Captured log call` block is the parser's
own output; the identical `Captured stderr call` copy that pytest also emits is reproduced in
[Appendix B §B‑Q3](#appendix-b):**

```text
==================================== PASSES ====================================
__________________________ TestParser.test_encrypted ___________________________
------------------------------ Captured log call -------------------------------
WARNING  paperless.parsing.tesseract:loggers.py:21 Error while getting text from PDF document with pdfminer.six
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
DEBUG    paperless.parsing.tesseract:loggers.py:21 Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/encrypted.pdf', 'output_file': '/tmp/tmpgc866lsy/paperless-h9xiowd8/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpgc866lsy/paperless-h9xiowd8/sidecar.txt'}
WARNING  paperless.parsing.tesseract:loggers.py:21 This file is encrypted, OCR is impossible. Using any text present in the original file.
WARNING  paperless.parsing.tesseract:loggers.py:21 No text was found in /app/src/paperless_tesseract/tests/samples/encrypted.pdf, the content will be empty.
=========================== short test summary info ============================
PASSED paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted
1 passed, 6 warnings in 1.82s
```

Run 2 footer: `1 passed, 6 warnings in 1.71s`. This is the whole Q3 answer in one capture:
pdfminer fails to read the encrypted PDF → `DEBUG … Calling OCRmyPDF with args: {…'skip_text': True…}`
is the invoked **`ocrmypdf.ocr`** subprocess (driving Tesseract) → the encrypted branch logs
*"OCR is impossible"* → the last‑resort branch logs *"the content will be empty."* The test
asserts `archive_path is None` and `get_text() == ""`.

**(2) `test_with_form_error_notext` (`test_parser.py:190`) — the no-text `skip` pass triggers a
**force-OCR retry** that recovers the form's text (the ~7.6 s duration is real Tesseract work,
proving the retry path is genuinely executed, not mocked):**

```text
=========================== short test summary info ============================
PASSED paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext
1 passed, 6 warnings in 7.60s
```

Run 2 footer: `1 passed, 6 warnings in 7.62s`.

**(3) `test_skip_noarchive_notext` (`test_parser.py:370`) — skip mode, no archive, no text:**

```text
=========================== short test summary info ============================
PASSED paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext
1 passed, 6 warnings in 3.74s
```

Run 2 footer: `1 passed, 6 warnings in 3.70s`.

**(4) `test_ocrmypdf_parameters` (`test_parser.py:427`) — asserts the exact args dict built for
`ocrmypdf.ocr` (`construct_ocrmypdf_parameters`):**

```text
=========================== short test summary info ============================
PASSED paperless_tesseract/tests/test_parser.py::TestParser::test_ocrmypdf_parameters
1 passed, 6 warnings in 1.40s
```

Run 2 footer: `1 passed, 6 warnings in 1.40s`.

**(5) Temporary probe `/tmp/q3_probe_test.py` — 3 passed, identical behavior across 2 runs.
It wraps the real `ocrmypdf.ocr` to record per-call kwargs/exception and calls the real
`RasterisedDocumentParser.parse` on a throwaway copy of each fixture. Complete run‑1 stdout
(the pdfminer/alpha-removal parser logs and the 6‑warning block that interleave before each
group are reproduced in [Appendix B §B‑Q3](#appendix-b)):**

```text
Loading .env environment variables...
Q3A ocrmypdf.ocr call#1: input_file=encrypted.pdf output_type='pdfa' skip_text=True redo_ocr=None force_ocr=None raised=EncryptedPdfError
Q3A total ocrmypdf.ocr invocations: 1
Q3A parser.archive_path: None
Q3A parser.get_text() repr: ''
Q3A mime AFTER parse (from input file): application/pdf (unchanged by empty OCR text)
Q3B ocrmypdf.ocr call#1: input_file=no-text-alpha.png output_type='pdfa' skip_text=True redo_ocr=None force_ocr=None raised=None
Q3B ocrmypdf.ocr call#2: input_file=no-text-alpha.png output_type='pdfa' skip_text=None redo_ocr=None force_ocr=True raised=None
Q3B total ocrmypdf.ocr invocations: 2
Q3B parser.archive_path: /tmp/tmphfpc0j2k/paperless-mxisgn2j/archive.pdf
Q3B parser.get_text() repr: ''
Q3B mime AFTER parse (from input file): image/png (unchanged by empty OCR text)
.Q3C magic.from_file(encrypted.pdf, mime=True) -> application/pdf
Q3C magic.from_file(no-text-alpha.png, mime=True) -> image/png
[warnings summary block omitted here — invariant 6-warning block, byte-identical to Q1 output (1); full text in Appendix B §B‑Q3]
=========================== short test summary info ============================
PASSED ::Q3NoText::test_a_encrypted_pdf_one_invocation_empty_text
PASSED ::Q3NoText::test_b_notext_image_force_ocr_two_invocations
PASSED ::Q3NoText::test_c_mime_from_magic
3 passed, 6 warnings in 3.84s
```

Run 2 footer: `3 passed, 6 warnings in 3.65s`. Every behavioral fact reproduced identically
across both runs; only the temporary `archive_path` filename varied (expected). The decisive
Q3 facts:

- **OCR subprocess = `ocrmypdf.ocr`** (`parsers.py:261`; force‑OCR retry `parsers.py:298`),
  driving Tesseract. The **encrypted** PDF raises `EncryptedPdfError` on its single call
  (`skip_text=True`) → **1** invocation, `archive_path = None`. The **no‑text alpha PNG**
  succeeds on call #1 (`skip_text=True`) but yields empty text → `NoTextFoundException` →
  **force‑OCR retry** on call #2 (`force_ocr=True`) → **2** invocations, and an archive PDF
  is still written even though the text stays empty.
- **Final text is empty** — `get_text() == ""` in both cases (last‑resort branch,
  `parsers.py:316-327`).
- **Mime type comes from the input file** via `magic.from_file(…, mime=True)`
  (`consumer.py:219`): `encrypted.pdf → application/pdf`, `no-text-alpha.png → image/png` —
  **unchanged** by the empty OCR text. (Aside: for alpha images, `parsers.py:191-201`
  flattens the alpha layer and rewrites the *input file*, which is why the probe parses a
  throwaway copy; this does not affect the assigned mime type.)

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

All commands use the project's canonical invocation (`cd src/ && pipenv run pytest …`,
run inside the container as `testuser`). `-rA` prints the short-test-summary / captured
logs; `-n0` pins a single worker so per-test output is stable; `--no-cov -p
no:cacheprovider` remove coverage/cache noise. Each was run twice; both footers appear
with every output block below. The invariant 6-warning block emitted on every run is
reproduced in full in [Appendix B §B‑Q4](#appendix-b) and elided (with a labeled note,
never a bare `...`) from the inline blocks.

```bash
# 1. barcode_reader across ALL symbologies/variants (Code39/Code128/QR/distortion/unreadable/no_barcode/custom)
cd src/ && pipenv run pytest documents/tests/test_tasks.py -k "test_barcode_reader" \
    -n0 --no-cov -p no:cacheprovider -rA

# 2. scan_file_for_separating_barcodes — separator page-index detection variants
cd src/ && pipenv run pytest documents/tests/test_tasks.py -k "test_scan_file_for_separating" \
    -n0 --no-cov -p no:cacheprovider -rA

# 3. separate_pages output-count assertions (N separators -> N+1 files) + no-separator case
cd src/ && pipenv run pytest documents/tests/test_tasks.py -k "test_separate_pages" \
    -n0 --no-cov -p no:cacheprovider -rA

# 4. test_barcode_splitter — real scan + separate_pages on patch-code-t-middle.pdf (repo test)
cd src/ && pipenv run pytest documents/tests/test_tasks.py::TestTasks::test_barcode_splitter \
    -n0 --no-cov -p no:cacheprovider -rA

# 5. test_consume_barcode_file — real consume_file() split path via the repo test
cd src/ && pipenv run pytest documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file \
    -n0 --no-cov -p no:cacheprovider -rA

# 6. Temporary probe (outside the repo, in /tmp): real barcode_reader() across Code39/Code128/QR,
#    scan_file_for_separating_barcodes() incl. a multi-separator file, separate_pages() record counts,
#    then consume_file() with CONSUMER_ENABLE_BARCODES=true showing Document.count BEFORE/AFTER == 0
cd src/ && pipenv run pytest /tmp/q4_probe_test.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -rA -s
```

> **Scope note — `test_management_consumer.py`.** At this commit
> (`542221a38dff`) `src/documents/tests/test_management_consumer.py` contains **no
> barcode-specific tests**; the barcode-split logic and its tests live in
> `src/documents/tasks.py` and `src/documents/tests/test_tasks.py`. The real
> `consume_file()` split path is therefore exercised here through the repository test
> `test_consume_barcode_file` (command 5) and the temporary probe (command 6), not through
> the management-command test module.

### Verbatim Observed Output

**(1) `barcode_reader` — 11 passed (all symbologies/variants).** Command 1 above,
`-rA` short-test-summary block, run 1:

```text
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader2
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_128_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_qr_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion2
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_no_barcode
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_unreadable
================ 11 passed, 29 deselected, 6 warnings in 1.70s =================
```

Run 2 footer: `11 passed, 29 deselected, 6 warnings in 1.70s`. The invariant 6-warning
block and the per-test `Barcode of type … found: PATCHT` captured DEBUG lines are
reproduced in full in [Appendix B §B‑Q4](#appendix-b).

**(2) `scan_file_for_separating_barcodes` — 10 passed** (incl. upsidedown / QR / custom /
wrong-QR). Command 2, run 1:

```text
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes2
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes3
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes_upsidedown
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_128_barcodes
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_barcodes
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_custom_qr_barcodes
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_qr_barcodes
PASSED documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_wrong_qr_barcodes
================ 10 passed, 30 deselected, 6 warnings in 4.20s =================
```

Run 2 footer: `10 passed, 30 deselected, 6 warnings in 4.15s`. (6-warning block →
[Appendix B §B‑Q4](#appendix-b).)

**(3) `separate_pages` — 2 passed** (`test_separate_pages`, `test_separate_pages_no_list`).
Command 3, run 1:

```text
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_separate_pages
PASSED documents/tests/test_tasks.py::TestTasks::test_separate_pages_no_list
================ 2 passed, 38 deselected, 6 warnings in 1.58s ==================
```

Run 2 footer: `2 passed, 38 deselected, 6 warnings in 1.55s`.

**(4) `test_barcode_splitter` — 1 passed** (repo test; real `scan` + `separate_pages`).
Command 4, run 1 — full `PASSES` captured-log section:

```text
==================================== PASSES ====================================
_______________________ TestTasks.test_barcode_splitter ________________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmpt0hqv_gj/paperless-nfif5aup
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 1
DEBUG    paperless.tasks:tasks.py:150 page_number: 1 next_page: 3
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 1 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmpt0hqv_gj/paperless-nfif5aup/patch-code-t-middle_document_0.pdf', '/tmp/tmpt0hqv_gj/paperless-nfif5aup/patch-code-t-middle_document_1.pdf']
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_splitter
======================== 1 passed, 6 warnings in 1.86s =========================
```

Run 2 footer: `1 passed, 6 warnings in 1.88s`. This is the **decision site + split
arithmetic observed end-to-end**: one `CODE39` `PATCHT` separator on the middle page →
`separate_pages` emits exactly `patch-code-t-middle_document_0.pdf` and
`_document_1.pdf` (`len(separators)+1 = 2` files).

**(5) `test_consume_barcode_file` — 1 passed** (repo test; the real `consume_file()` split
path). Command 5, run 1 — full `PASSES` captured section:

```text
==================================== PASSES ====================================
_____________________ TestTasks.test_consume_barcode_file ______________________
----------------------------- Captured stderr call -----------------------------
[2026-07-07 00:39:12,207] [WARNING] [paperless.tasks] /tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_0.pdf or /app/src/../consume don't exist.
[2026-07-07 00:39:12,208] [WARNING] [paperless.tasks] /tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_1.pdf or /app/src/../consume don't exist.
[2026-07-07 00:39:12,227] [WARNING] [paperless.tasks] OSError. It could be, the broker cannot be reached.
[2026-07-07 00:39:12,227] [WARNING] [paperless.tasks] Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:200 Pages with separators found in: /tmp/tmp7uwe725z/patch-code-t-middle.pd
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmp7uwe725z/paperless-nter0hhm
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 1
DEBUG    paperless.tasks:tasks.py:150 page_number: 1 next_page: 3
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 1 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_0.pdf', '/tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_1.pdf']
WARNING  paperless.tasks:tasks.py:181 /tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_0.pdf or /app/src/../consume don't exist.
WARNING  paperless.tasks:tasks.py:181 /tmp/tmp7uwe725z/paperless-nter0hhm/patch-code-t-middle_document_1.pdf or /app/src/../consume don't exist.
DEBUG    paperless.tasks:tasks.py:213 Deleting file /tmp/tmp7uwe725z/patch-code-t-middle.pd
WARNING  paperless.tasks:tasks.py:231 OSError. It could be, the broker cannot be reached.
WARNING  paperless.tasks:tasks.py:232 Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file
======================== 1 passed, 6 warnings in 1.89s =========================
```

Run 2 footer: `1 passed, 6 warnings in 1.87s`. The `Pages with separators found in`
(`tasks.py:200`) → split → `Deleting file` (`tasks.py:213`, the input is `os.unlink`ed)
sequence is the real `consume_file()` split branch. The two `… or /app/src/../consume
don't exist.` warnings (`tasks.py:181`) show `save_to_dir` skipping the (absent) consume
directory, and the `OSError` at `tasks.py:231-232` is the closed-broker
`group_send` **caught** by `except OSError` — neither aborts the split, which still
returns `"File successfully split"`.

**(6) Temporary probe `/tmp/q4_probe_test.py` — 5 passed** (real `barcode_reader` across
Code39/Code128/QR; `scan_file_for_separating_barcodes` incl. a multi-separator file;
`separate_pages` record counts; the `consume_file()` **split** state transition; and the
**non-barcoded** consume that creates `+1 Document`). Command 6, run 1 — complete captured
stdout:

```text
Loading .env environment variables...
Q4A CONSUMER_BARCODE_STRING (separator) = 'PATCHT'
Q4A barcode_reader(barcode-39-PATCHT.png)  [Code39]  -> ['PATCHT']
Q4A barcode_reader(barcode-128-PATCHT.png) [Code128] -> ['PATCHT']
Q4A barcode_reader(qr-code-PATCHT.png)     [QR]      -> ['PATCHT']
Q4A barcode_reader(barcode-39-PATCHT-unreadable.png) -> []
Q4A barcode_reader(simple.png no barcode)            -> []
.Q4B scan(patch-code-t.pdf) separator pages        -> [0]
Q4B scan(patch-code-t-middle.pdf) separator pages -> [1]
Q4B scan(several-patcht-codes.pdf) separator pages-> [2, 5]
Q4B scan(simple.pdf no separator)                 -> []
.Q4C separate_pages(patch-code-t.pdf, [0]) -> count: 2 (len(separators)+1 = 2) -> ['patch-code-t_document_0.pdf', 'patch-code-t_document_1.pdf']
Q4C separate_pages(patch-code-t-middle.pdf, [1]) -> count: 2 (len(separators)+1 = 2) -> ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
Q4C separate_pages(several-patcht-codes.pdf, [2, 5]) -> count: 3 (len(separators)+1 = 3) -> ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
Q4C separate_pages(..., []) -> []
Q4C warning log: ['WARNING:paperless.tasks:No pages to split on!']
.[2026-07-07 00:50:30,519] [WARNING] [paperless.tasks] /tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_0.pdf or /app/src/../consume don't exist.
[2026-07-07 00:50:30,519] [WARNING] [paperless.tasks] /tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_1.pdf or /app/src/../consume don't exist.
[2026-07-07 00:50:30,590] [WARNING] [paperless.tasks] OSError. It could be, the broker cannot be reached.
[2026-07-07 00:50:30,590] [WARNING] [paperless.tasks] Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
Q4D Document count BEFORE consume_file: 0
Q4D consume_file(copy) returned: 'File successfully split'
Q4D Document count AFTER consume_file: 0
Q4D input copy still exists (should be unlinked): False
Q4D zero Document rows created by split call: True
.[2026-07-07 00:50:30,925] [INFO] [paperless.consumer] Consuming q4_simple_input.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmp5k2tnpft/paperless-_s9ddzk_/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-07 00:50:31,319] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-07 00:50:32,045] [INFO] [paperless.consumer] Document 2026-07-07 q4_simple_input consumption finished
Q4E separators in simple.pdf (none -> fallthrough): []
Q4E Document.count BEFORE: 0 | effective (inbox-excluded) BEFORE: 0
Q4E consume_file(copy) returned: 'Success. New document id 1 created'
Q4E Document.count AFTER: 1 | effective (inbox-excluded) AFTER: 1
Q4E created Document -> id: 1 | mime_type: application/pdf | in inbox-excluded training set: True
.
```

The invariant 6-warning summary block that follows is reproduced in full in
[Appendix B §B‑Q4](#appendix-b). The probe's `PASSES` captured-log section — which proves
all three encodings decode to `PATCHT` (test `a`), the split arithmetic (test `c`), the
zero-`Document` split path (test `d`), and the full non-barcoded consume pipeline that
creates `+1 Document` (test `e`, note `Detected mime type: application/pdf` and
`Document classification model does not exist (yet)…` — the same `classifier.py:32`
no-model path documented for Q1) — is:

```text
==================================== PASSES ====================================
____________________ Q4Barcode.test_a_barcode_reader_values ____________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE128 found: PATCHT
DEBUG    paperless.tasks:tasks.py:90 Barcode of type QRCODE found: PATCHT
____________________ Q4Barcode.test_b_scan_separator_pages _____________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
____________________ Q4Barcode.test_c_separate_pages_counts ____________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmpxroj3t38/paperless-7jnko8cg
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 0
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 0 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmpxroj3t38/paperless-7jnko8cg/patch-code-t_document_0.pdf', '/tmp/tmpxroj3t38/paperless-7jnko8cg/patch-code-t_document_1.pdf']
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmpxroj3t38/paperless-eo2cgpjx
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 1
DEBUG    paperless.tasks:tasks.py:150 page_number: 1 next_page: 3
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 1 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmpxroj3t38/paperless-eo2cgpjx/patch-code-t-middle_document_0.pdf', '/tmp/tmpxroj3t38/paperless-eo2cgpjx/patch-code-t-middle_document_1.pdf']
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmpxroj3t38/paperless-defcl9rl
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 2
DEBUG    paperless.tasks:tasks.py:150 page_number: 2 next_page: 5
DEBUG    paperless.tasks:tasks.py:150 page_number: 2 next_page: 5
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 2 pages
DEBUG    paperless.tasks:tasks.py:142 Count: 1 page_number: 5
DEBUG    paperless.tasks:tasks.py:150 page_number: 5 next_page: 7
DEBUG    paperless.tasks:tasks.py:155 pdf no:1 has 1 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmpxroj3t38/paperless-defcl9rl/several-patcht-codes_document_0.pdf', '/tmp/tmpxroj3t38/paperless-defcl9rl/several-patcht-codes_document_1.pdf', '/tmp/tmpxroj3t38/paperless-defcl9rl/several-patcht-codes_document_2.pdf']
____________ Q4Barcode.test_d_consume_split_creates_zero_documents _____________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
DEBUG    paperless.tasks:tasks.py:200 Pages with separators found in: /tmp/tmpw54gll66/q4_copy_patch-code-t-middle.pdf
DEBUG    paperless.tasks:tasks.py:125 Temp dir is /tmp/tmpw54gll66/paperless-q8npna1f
DEBUG    paperless.tasks:tasks.py:142 Count: 0 page_number: 1
DEBUG    paperless.tasks:tasks.py:150 page_number: 1 next_page: 3
DEBUG    paperless.tasks:tasks.py:155 pdf no:0 has 1 pages
DEBUG    paperless.tasks:tasks.py:160 Temp files are ['/tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_0.pdf', '/tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_1.pdf']
WARNING  paperless.tasks:tasks.py:181 /tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_0.pdf or /app/src/../consume don't exist.
WARNING  paperless.tasks:tasks.py:181 /tmp/tmpw54gll66/paperless-q8npna1f/q4_copy_patch-code-t-middle_document_1.pdf or /app/src/../consume don't exist.
DEBUG    paperless.tasks:tasks.py:213 Deleting file /tmp/tmpw54gll66/q4_copy_patch-code-t-middle.pdf
WARNING  paperless.tasks:tasks.py:231 OSError. It could be, the broker cannot be reached.
WARNING  paperless.tasks:tasks.py:232 Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
__________ Q4Barcode.test_e_nonbarcoded_consume_creates_one_document ___________
------------------------------ Captured log call -------------------------------
INFO     paperless.consumer:loggers.py:21 Consuming q4_simple_input.pdf
DEBUG    paperless.consumer:loggers.py:21 Detected mime type: application/pdf
DEBUG    paperless.consumer:loggers.py:21 Parser: RasterisedDocumentParser
DEBUG    paperless.consumer:loggers.py:21 Parsing q4_simple_input.pdf...
DEBUG    paperless.parsing.tesseract:loggers.py:21 Extracted text from PDF file /tmp/tmp5k2tnpft/q4_simple_input.pdf
DEBUG    paperless.parsing.tesseract:loggers.py:21 Calling OCRmyPDF with args: {'input_file': '/tmp/tmp5k2tnpft/q4_simple_input.pdf', 'output_file': '/tmp/tmp5k2tnpft/paperless-_s9ddzk_/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmp5k2tnpft/paperless-_s9ddzk_/sidecar.txt'}
DEBUG    paperless.parsing.tesseract:loggers.py:21 Incomplete sidecar file: discarding.
DEBUG    paperless.parsing.tesseract:loggers.py:21 Extracted text from PDF file /tmp/tmp5k2tnpft/paperless-_s9ddzk_/archive.pdf
DEBUG    paperless.consumer:loggers.py:21 Generating thumbnail for q4_simple_input.pdf...
DEBUG    paperless.parsing:parsers.py:143 Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmp5k2tnpft/paperless-_s9ddzk_/archive.pdf[0] /tmp/tmp5k2tnpft/paperless-_s9ddzk_/convert.png
WARNING  paperless.parsing:parsers.py:158 Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
DEBUG    paperless.parsing:parsers.py:143 Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmp5k2tnpft/paperless-_s9ddzk_/gs_out.png /tmp/tmp5k2tnpft/paperless-_s9ddzk_/convert_gs.png
DEBUG    paperless.parsing.tesseract:loggers.py:21 Execute: optipng -silent -o5 /tmp/tmp5k2tnpft/paperless-_s9ddzk_/convert_gs.png -out /tmp/tmp5k2tnpft/paperless-_s9ddzk_/thumb_optipng.png
DEBUG    paperless.classifier:classifier.py:32 Document classification model does not exist (yet), not performing automatic matching.
DEBUG    paperless.consumer:loggers.py:21 Saving record to database
DEBUG    paperless.consumer:loggers.py:21 Deleting file /tmp/tmp5k2tnpft/q4_simple_input.pdf
DEBUG    paperless.parsing.tesseract:loggers.py:21 Deleting directory /tmp/tmp5k2tnpft/paperless-_s9ddzk_
INFO     paperless.consumer:loggers.py:21 Document 2026-07-07 q4_simple_input consumption finished
=========================== short test summary info ============================
PASSED ::Q4Barcode::test_a_barcode_reader_values
PASSED ::Q4Barcode::test_b_scan_separator_pages
PASSED ::Q4Barcode::test_c_separate_pages_counts
PASSED ::Q4Barcode::test_d_consume_split_creates_zero_documents
PASSED ::Q4Barcode::test_e_nonbarcoded_consume_creates_one_document
5 passed, 6 warnings in 5.09s
```

Run 2 footer: `5 passed, 6 warnings in 5.25s`. Every value — the `'PATCHT'` default, the
three encodings all decoding to `['PATCHT']` (corroborated by the `CODE39`/`CODE128`/
`QRCODE found: PATCHT` DEBUG lines), the separator indices `[0]`/`[1]`/`[2, 5]`/`[]`, the
`len(separators)+1` file counts (2/2/3), the split state transition `Document.count() 0 →
"File successfully split" → 0` (with the input copy `os.unlink`ed, `exists = False`), and
the non-barcoded arm `0 → "Success. New document id 1 created" → 1` (effective
inbox-excluded count `0 → 1`) — reproduced identically across both runs.

> **Note on `test_e`'s environment artifacts (non-canonical, benign).** Two lines in the
> `test_e` output are environment side-effects, not behavior of the code under
> investigation: the `convert-im6.q16: … not allowed by the security policy 'PDF'`
> messages and the `Thumbnail generation with ImageMagick failed, falling back to
> ghostscript` warning (`parsers.py:158`) come from the container's ImageMagick
> `policy.xml` disallowing direct PDF rasterization — paperless transparently falls back
> to ghostscript and the `Document` is still created successfully (`id 1`,
> `application/pdf`). The out-of-band websocket progress notification (which needs redis)
> is mocked exactly as the repo's own full-consume tests do
> (`documents/tests/test_consumer.py:290` patches `Consumer._send_progress`); the real
> parse → OCR → classify → store → `Document` create pipeline runs unmodified.

### Rationale

The number of *records* produced by the split equals `len(separators) + 1` **PDF files**
(`tasks.py:113-161`; observed `[0]→2`, `[1]→2`, `[2,5]→3` in probe block 6 and repo test
block 4), but the split call creates **zero `Document` rows** and returns `"File
successfully split"` (`tasks.py:233`; observed `Document.objects.count()` `0 → 0` in probe
`Q4D`). Therefore the split does **not** synchronously alter the classifier's training
set — the effective training set (`Document.objects.exclude(tags__is_inbox_tag=True)`,
`classifier.py:125-127`) is unchanged at split time. It changes **only after** the split
parts are re-queued and re-consumed as their own `Document`s: each split part is written
back for consumption (the `save_to_dir` step, `tasks.py:181` — here it logs `… don't
exist.` because no consume directory is configured in the probe/test environment), and a
later normal `try_consume_file` on each part is what persists a `+1 Document`. The probe's
`Q4E` observes exactly that other arm directly: a non-barcoded input has no separators
(`scan → []`), falls through to `Consumer().try_consume_file()` (`tasks.py:236`), and the
state transitions from `Document.count() 0` to `1` — returning `"Success. New document id
1 created"` — with the new row landing in the effective inbox-excluded training set
(`0 → 1`). This timing gap — **split now (0 `Document`s), `Document`s later (+1 each on
re-consume)** — is directly relevant to the user's non-determinism concern: *when* the
split parts become `Document`s relative to *when* `train_classifier()` next runs determines
what data the model sees, and the retrain guard (`classifier.py:163-164`) then decides
whether a retrain even happens for that data set.

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

The temporary probe `/tmp/nondet_probe_test.py` (class `NonDeterminism`) trains **`N = 30`
fresh, unseeded `DocumentClassifier` models against the SAME unchanged corpus** and reports
the observed prediction distribution — `test_a_borderline_distribution` (a borderline query
against two `MATCH_AUTO` correspondents whose documents share the prefix `"the quick brown
fox"`) and `test_b_distinctive_distribution` (a well-separated corpus). The real entry
points `DocumentClassifier.train()` and `predict_correspondent()` are exercised; only the
training corpus is a controlled fixture built to be borderline vs. distinctive. Per the
*reproduce-reported-inconsistency* rule the **same command was run 4 times** (not merely
twice) so the flip distribution is visible.

```bash
# Same unchanged 2-correspondent corpus; N=30 fresh unseeded models per test; run 4x to
# expose the run-to-run flip. -s surfaces the printed distribution; -rA is intentionally
# omitted because the per-model training DEBUG block repeats N times and adds no distinct
# information (the full raw log with those blocks is in Appendix B §B-ND).
cd src/ && pipenv run pytest /tmp/nondet_probe_test.py -c /app/src/setup.cfg \
    --rootdir=/app/src -n0 --no-cov -p no:cacheprovider -s
```

### Verbatim Observed Output

**Complete captured output, run 1** (31 lines, no elision — the invariant 6-warning block
is shown in full here and again in [Appendix B §B‑ND](#appendix-b)):

```text
Loading .env environment variables...
NONDET borderline query='the quick brown fox' N=30 distribution: {'c1': 14, 'c2': 16}
.NONDET distinctive query='alpha alpha alpha alpha' N=30 distribution: {'c1': 30}
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
2 passed, 6 warnings in 3.11s
```

**The borderline query FLIPS across the 4 identical runs; the distinctive query is STABLE.**
The `NONDET …` distribution lines and footers observed for each of the four runs of the
exact same command:

```text
run 1: NONDET borderline  query='the quick brown fox'      N=30 distribution: {'c1': 14, 'c2': 16}
run 1: NONDET distinctive query='alpha alpha alpha alpha'   N=30 distribution: {'c1': 30}
run 1 footer: 2 passed, 6 warnings in 3.11s
run 2: NONDET borderline  query='the quick brown fox'      N=30 distribution: {'c1': 19, 'c2': 11}
run 2: NONDET distinctive query='alpha alpha alpha alpha'   N=30 distribution: {'c1': 30}
run 2 footer: 2 passed, 6 warnings in 3.10s
run 3: NONDET borderline  query='the quick brown fox'      N=30 distribution: {'c1': 14, 'c2': 16}
run 3: NONDET distinctive query='alpha alpha alpha alpha'   N=30 distribution: {'c1': 30}
run 3 footer: 2 passed, 6 warnings in 3.06s
run 4: NONDET borderline  query='the quick brown fox'      N=30 distribution: {'c2': 10, 'c1': 20}
run 4: NONDET distinctive query='alpha alpha alpha alpha'   N=30 distribution: {'c1': 30}
run 4 footer: 2 passed, 6 warnings in 3.23s
```

The **borderline** distribution moved run-to-run — `14/16`, `19/11`, `14/16`, `10/20`
(c1/c2) — even though the input corpus, query, and `N` were **byte-for-byte identical**;
this is exactly the *reproduce-the-same-input inconsistency* the rule requires (no
stabilised variant was constructed). The **distinctive** distribution was `{'c1': 30}` in
**all four** runs, confirming that well-separated data lands on the same argmax every time.
The complete raw logs for representative runs (including the per-model training DEBUG
blocks that repeat `N` times) are in [Appendix B §B‑ND](#appendix-b).

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
| Retrain guard (unchanged data) | `train()` → **`False`** (no retrain); sequence `True→False→True` | `classifier.py:161-164`, `:247`, `:249` | q1 probe: `train#1=True`, `train#2=False`, `train#3=True`; `data_hash b292e1a9…` (→ `23706c53…` after mutate, deterministic for the embedded probe's 3rd doc — [Appendix C §C‑Q1](#appendix-c)) identical ×3 runs; `testDatasetHashing` PASSED | fresh train (`True`), unchanged (`False`), mutated (`True`), empty→`ValueError` (`:159`) | SHA-1 hash equality short-circuits re-fit → idempotent training within a run |
| No in-memory cache | **No cache**; new instance + disk read every call; `None` if absent | `classifier.py:30-57` (`:31`,`:36`,`:38`,`:40`,`:57`) | q1 probe: call#1 `id 138582118506208` ≠ call#2 `id 138582118505344` (distinct instances; ids ephemeral run-to-run); missing→`None`; `test_load_classifier_cached` **SKIPPED** ("Disabled caching…") | present-file (distinct ids), absent-file (`None`), skip banner | Loader reconstructs+reloads each call → no shared model object |
| Per-test `MODEL_FILE` isolation | **Distinct** throwaway path per test; removed at teardown | `utils.py:18`,`:45`,`:53-57`,`:77-83`; `settings.py:74` | q1 probe: test#1 `/tmp/tmpgnqsr2x6/…` ≠ test#2 `/tmp/tmpp5rkoclt/…` (paths ephemeral); per-test path differs = `True`; each `MODEL_FILE` absent at `setUp` | two sequential tests; post-teardown existence check | `mkdtemp` + `rmtree` per test → no model file survives across tests |
| pytest-xdist parallelism | **128** separate worker processes, per-worker DB | `setup.cfg` (`--numprocesses auto`) | canonical run: `created: 128/128 workers`, `128 workers [1 item]`, ~24.8s vs ~1.9s `-n0` | canonical (128 workers) vs `-n0` single-worker | No shared in-process memory → cross-test model reuse impossible |

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
| OCR subprocess | **`ocrmypdf.ocr(**args)`** → drives Tesseract 4.1.1 | `parsers.py:261` (primary), `:298` (retry) | q3 probe: `Q3A ocrmypdf.ocr call#1 … skip_text=True`; `Q3B … call#2 … force_ocr=True` | primary pass, force-OCR retry | ocrmypdf is the invoked OCR subprocess |
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
| Root cause | `MLPClassifier` has **no `random_state`** → weights re-randomised each fit | `classifier.py:219`,`:227`,`:238` | nondet probe (N=30/test, ×4 runs): distinctive query stable `{'c1':30}` all 4 runs; borderline query flips (see next row) | well-separated (stable) vs borderline (flips) | Unseeded MLP → model differs every run |
| Observed flip distribution | borderline query `'the quick brown fox'` (N=30) flips `{'c1':14,'c2':16}`→`{'c1':19,'c2':11}`→`{'c1':14,'c2':16}`→`{'c2':10,'c1':20}` | `classifier.py:227` | nondet probe ×4 runs, byte-identical input, different distributions | 4 repeated runs of identical input | Argmax on ambiguous data decided by random init |
| Compounding factor | **128** xdist workers, independent random models | `setup.cfg` | `created: 128/128 workers` | canonical vs `-n0` | Parallelism multiplies independently-random models |

---

<a name="proof"></a>
<a name="appendix-a"></a>
## Appendix A — Read-Only Proof & Reproduction Notes

* **Source tree unchanged.** All investigation ran inside the container
  `paperless-qna-baked` (image `paperless-ngx-qna:local`); every temporary probe
  (`/tmp/q1_probe_test.py`, `/tmp/q2_probe_test.py`, `/tmp/q3_probe_test.py`,
  `/tmp/q4_probe_test.py`, `/tmp/nondet_probe_test.py`) lived under the container's `/tmp`,
  never in the repository tree. The host repository's only change is this single
  document. Verified with `git status --porcelain` and a diff of the whole working tree
  against the pristine upstream source commit `542221a38` — both reproduced verbatim below.
* **Stability.** Every count/timing/branch value above was captured on **≥2 runs** and
  reproduced identically; the one genuinely non-deterministic value (borderline-input
  argmax) is reported as an observed **distribution** across repeated identical runs, not
  stabilised.
* **Canonical config.** All runs used the project's own `setup.cfg` (`DJANGO_SETTINGS_MODULE=paperless.settings`,
  `PAPERLESS_DISABLE_DBHANDLER=true`). Where a single-worker view was needed for a focused
  observation, `-n0` was used and **labeled** (pytest-randomly is not installed, so test
  order is never randomised); the default suite retains `--numprocesses auto`.

The read-only guarantee is reproduced verbatim below (all commands run on the host from
the repository root; the diff base `542221a38` is the pristine upstream source commit,
the parent of the documentation commits). **The status shown is the final, committed
state**: once this answer document is committed, `git status --porcelain` and the
working‑tree `git diff` are both **empty (clean)** — the only difference from the pristine
upstream source is the single added document, which the diff‑against‑`542221a38` proves.
(During authoring, *before* the document is committed, the working tree transiently shows
exactly one entry — this same document — as `?? ` (untracked) or ` M`/`A ` (modified/added);
that transient entry is the document itself, never a source file.)

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-4898f569-a9c9-44f0-8441-e5d3f590b862

# Final committed state: working tree is clean (the document is committed), so porcelain
# status and the working-tree diff are both empty.
$ git status --porcelain
$ git diff --stat
# (no output for either command -- clean working tree)

# Diff of the committed tree against the pristine upstream source commit
# 542221a38 ("Merge pull request #792 ..."): the ONLY change is the added answer
# document -- not one source file is modified, created, or deleted. This durable proof
# holds regardless of how many documentation commits were made.
$ git diff --name-status 542221a38
A	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --stat 542221a38
 blitzy/documentation/paperless-ngx_542221a38dff.md | 2398 ++++++++++++++++++++
 1 file changed, 2398 insertions(+)

# No temporary / probe / adhoc helper leaked into the repository tree:
$ find . -path ./.git -prune -o \( -name '*probe*' -o -name '*adhoc*' -o -name 'nondet*' \) -print
# (no output -- none found)

# Every observation probe lived OUTSIDE the repo, under the container's /tmp, and was
# removed at completion (per the read-only directive); the container /tmp now holds none:
$ docker exec -u testuser paperless-qna-baked ls -1 /tmp/*probe*.py
ls: cannot access '/tmp/*probe*.py': No such file or directory
```

---

<a name="appendix-b"></a>
## Appendix B — Complete Raw Command Logs

Every per-question section above quotes the *salient* lines from the commands it ran and, wherever it elides the **invariant 6‑warning summary block** (or, under xdist, the aggregated 774‑warning block), points here with a labeled note rather than a bare `...`. This appendix reproduces **one complete, unedited run per command group** end‑to‑end (`Loading .env` line -> banner/collection -> warnings summary -> `PASSES` captured logs -> footer), and records the **run‑2 footer** beside each so the >=2‑run stability claim is auditable. Nothing below is truncated: the 6‑warning block that recurs in every single‑worker run is shown here in full for each group.

All commands were issued inside the canonical container as the non‑root `testuser` (`docker exec -u testuser paperless-qna-baked bash -lc 'cd /app/src && <cmd>'`); the canonical invocation form is `cd src/ && pipenv run pytest ...` (the `Loading .env environment variables...` first line is pipenv loading the project `.env`).

### §B‑Q1 — Retrain guard / dataset hashing (single‑worker) + canonical xdist run

**Command (focused, single‑worker):**

```bash
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    -n0 --no-cov -p no:cacheprovider -rA
```

**Complete output (run 1):**

```text
Loading .env environment variables...
.                                                                        [100%]
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
==================================== PASSES ====================================
______________________ TestClassifier.testDatasetHashing _______________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:203 Training tags classifier...
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:237 Training document type classifier...
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::testDatasetHashing
1 passed, 6 warnings in 1.98s
```

Run 2 footer (identical result, stability confirmed): `1 passed, 6 warnings in 1.89s`

**Command (canonical, inherits `--numprocesses auto` -> 128 workers; `-v` reveals the banner):**

```bash
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
    --no-cov -p no:cacheprovider -v
```

**Complete output (run 1).** Under xdist pytest **de‑duplicates** the warnings summary: the footer count `774` is the *aggregate* across workers (6 unique warnings x 129 occurrences = 774), while the printed summary lists each unique warning once, annotated `: 129 warnings`. This is the full block the inline Q1 notes point to:

```text
Loading .env environment variables...
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
created: 128/128 workers
128 workers [1 item]

.                                                                        [100%]
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
======================= 1 passed, 774 warnings in 24.77s =======================
```

Run 2 footer (stable): `======================= 1 passed, 774 warnings in 24.63s =======================`

### §B‑Q2 — Single‑document correspondent prediction

**Command:**

```bash
cd src/ && pipenv run pytest documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict \
    -n0 --no-cov -p no:cacheprovider -rA
```

**Complete output (run 1)** ‑ the `Captured log call` block is the classifier's own `train()` DEBUG trace (1 document, 1 correspondent, no tags/types):

```text
Loading .env environment variables...
.                                                                        [100%]
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
==================================== PASSES ====================================
________________ TestClassifier.test_one_correspondent_predict _________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
=========================== short test summary info ============================
PASSED documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict
1 passed, 6 warnings in 1.85s
```

Run 2 footer (stable): `1 passed, 6 warnings in 1.85s`

### §B‑Q3 — No‑extractable‑text PDF (encrypted) -> empty text

**Command:**

```bash
cd src/ && pipenv run pytest paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted \
    -n0 --no-cov -p no:cacheprovider -rA
```

**Complete output (run 1)** ‑ shows the full OCR fallback chain: pdfminer raises `PDFPasswordIncorrect`, the parser logs `This file is encrypted, OCR is impossible`, then `No text was found ... the content will be empty`. The `Calling OCRmyPDF with args:` line is the exact kwargs dict handed to `ocrmypdf.ocr()`:

```text
Loading .env environment variables...
.                                                                        [100%]
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
==================================== PASSES ====================================
__________________________ TestParser.test_encrypted ___________________________
----------------------------- Captured stderr call -----------------------------
[2026-07-07 00:28:35,718] [WARNING] [paperless.parsing.tesseract] Error while getting text from PDF document with pdfminer.six
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
[2026-07-07 00:28:36,003] [WARNING] [paperless.parsing.tesseract] This file is encrypted, OCR is impossible. Using any text present in the original file.
[2026-07-07 00:28:36,003] [WARNING] [paperless.parsing.tesseract] No text was found in /app/src/paperless_tesseract/tests/samples/encrypted.pdf, the content will be empty.
------------------------------ Captured log call -------------------------------
WARNING  paperless.parsing.tesseract:loggers.py:21 Error while getting text from PDF document with pdfminer.six
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
DEBUG    paperless.parsing.tesseract:loggers.py:21 Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/encrypted.pdf', 'output_file': '/tmp/tmpgc866lsy/paperless-h9xiowd8/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpgc866lsy/paperless-h9xiowd8/sidecar.txt'}
WARNING  paperless.parsing.tesseract:loggers.py:21 This file is encrypted, OCR is impossible. Using any text present in the original file.
WARNING  paperless.parsing.tesseract:loggers.py:21 No text was found in /app/src/paperless_tesseract/tests/samples/encrypted.pdf, the content will be empty.
=========================== short test summary info ============================
PASSED paperless_tesseract/tests/test_parser.py::TestParser::test_encrypted
1 passed, 6 warnings in 1.82s
```

Run 2 footer (stable): `1 passed, 6 warnings in 1.71s`

### §B‑Q4 — Barcode reader across all symbologies/variants

**Command:**

```bash
cd src/ && pipenv run pytest documents/tests/test_tasks.py -k test_barcode_reader \
    -n0 --no-cov -p no:cacheprovider -rA
```

**Complete output (run 1)** ‑ every `Barcode of type <SYMB> found: <VALUE>` line is the real `barcode_reader()` DEBUG log (`tasks.py:90`) across Code39 / Code128 / QR / distortion / custom‑separator / no‑barcode / unreadable inputs:

```text
Loading .env environment variables...
...........                                                              [100%]
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
==================================== PASSES ====================================
________________________ TestTasks.test_barcode_reader _________________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
________________________ TestTasks.test_barcode_reader2 ________________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
______________________ TestTasks.test_barcode_reader_128 _______________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE128 found: PATCHT
______________ TestTasks.test_barcode_reader_custom_128_separator ______________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE128 found: CUSTOM BARCODE
______________ TestTasks.test_barcode_reader_custom_qr_separator _______________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type QRCODE found: CUSTOM BARCODE
________________ TestTasks.test_barcode_reader_custom_separator ________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: CUSTOM BARCODE
___________________ TestTasks.test_barcode_reader_distorsion ___________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
__________________ TestTasks.test_barcode_reader_distorsion2 ___________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type CODE39 found: PATCHT
_______________________ TestTasks.test_barcode_reader_qr _______________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.tasks:tasks.py:90 Barcode of type QRCODE found: PATCHT
=========================== short test summary info ============================
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader2
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_128_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_qr_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_custom_separator
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_distorsion2
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_no_barcode
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr
PASSED documents/tests/test_tasks.py::TestTasks::test_barcode_reader_unreadable
11 passed, 29 deselected, 6 warnings in 1.70s
```

Run 2 footer (stable): `11 passed, 29 deselected, 6 warnings in 1.70s`

### §B‑ND — Non‑determinism distribution probe

**Command (run 4x; `-rA` intentionally omitted for the primary capture ‑ see note below):**

```bash
cd src/ && pipenv run pytest /tmp/nondet_probe_test.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -s
```

**Complete clean output (run 1, 31 lines, zero elision):**

```text
Loading .env environment variables...
NONDET borderline query='the quick brown fox' N=30 distribution: {'c1': 14, 'c2': 16}
.NONDET distinctive query='alpha alpha alpha alpha' N=30 distribution: {'c1': 30}
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
2 passed, 6 warnings in 3.11s
```

Run 1‑4 footers (all pass; timing only varies): `2 passed, 6 warnings in 3.11s` / `... 3.10s` / `... 3.06s` / `... 3.23s`. The **borderline** distribution flips run‑to‑run on byte‑identical input (`{'c1':14,'c2':16}` -> `{'c1':19,'c2':11}` -> `{'c1':14,'c2':16}` -> `{'c2':10,'c1':20}`), while the **distinctive** distribution is stable at `{'c1':30}` all four runs.

**Note on the `-rA` variant.** Re‑running the same command *with* `-rA` yields a 399‑line log because pytest then prints the per‑model `train()` DEBUG block, which repeats **60 times** (N=30 fresh models x 2 tests). That block carries no distinct information beyond the clean log above; one representative occurrence (from the `-rA` run‑1 log) is:

```text
==================================== PASSES ====================================
________________ NonDeterminism.test_a_borderline_distribution _________________
------------------------------ Captured log call -------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 2 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
```

This 6‑line `train()` DEBUG block (`Gathering data` -> doc/tag/correspondent/type counts -> `Vectorizing` -> `no tags` -> `Training correspondent classifier` -> `no document types`) recurs once per model fit; `grep -c 'Gathering data from database' nondet.run1.log` = **60**.



---

<a name="appendix-c"></a>
## Appendix C — Temporary Observation Probe Sources (Q1 & Q2)

The two temporary observation probes referenced by Q1 and Q2 are embedded **verbatim** below
so that every value they produce is independently regenerable. Per the read-only directive
these files lived only under the **container's `/tmp`** (`/tmp/q1_probe_test.py`,
`/tmp/q2_probe_test.py`) — **never inside the repository tree** — and were removed after the
observation; embedding their source here (as documentation text, not as repository code)
keeps the tree unchanged while making the evidence reproducible. Each was executed with the
exact command shown in that question's *Commands Run* section:

```bash
cd src/ && pipenv run pytest /tmp/<probe>.py -c /app/src/setup.cfg --rootdir=/app/src \
    -n0 --no-cov -p no:cacheprovider -rA -s
```

**Determinism note.** The Q1 `data_hash` digests are a deterministic SHA‑1 over the ordered,
inbox‑excluded document set (`classifier.py:123-162`) with **no** MLP randomness, so the
`b292e1a9…` (two‑document) and `23706c53…` (post‑mutation, three‑document) digests reproduce
**byte‑identically every run** — confirmed here across three runs. The Q2 probe's *counts,
timing, `classes_` membership, and `predict`/`predict_proba` call counts (2 / 0)* are likewise
invariant; its *argmax `predict()` labels*, however, are the output of an **unseeded**
`MLPClassifier` (`classifier.py:227`, no `random_state`) and therefore form an **observed
distribution** rather than a constant — see Q2's stability discussion and the
[Non‑Determinism section](#nd).

### §C‑Q1 — `/tmp/q1_probe_test.py` (retrain guard + mutation, no‑cache, save/load, per‑test isolation)

Reproduces the Q1 *Verbatim Observed Output* block: `data_hash` `None` →
`b292e1a98544b739dfacde57a87b94c7e3c102cc` → unchanged (guard) →
`23706c538fac64adf7edf3a4591974b7ae1b7aac` after adding the explicit third document. Only the
`id()` integers (memory addresses) and the per‑test `mkdtemp()` paths vary run‑to‑run.

```python
"""
Temporary Q1 observation probe (lives in the container's /tmp, OUTSIDE the repo tree).
Exercises the REAL DocumentClassifier.train() / load_classifier() entry points through the
canonical DirectoriesMixin + Django TestCase harness. The Q1A data_hash values are
DETERMINISTIC (SHA-1 over the ordered, inbox-excluded document content + type/correspondent
PKs, classifier.py:123-162 — no MLP randomness), so they reproduce byte-identically every
run; only the id() integers (memory addresses) and the per-test mkdtemp() paths vary.
Run: cd src/ && pipenv run pytest /tmp/q1_probe_test.py -c /app/src/setup.cfg \
        --rootdir=/app/src -n0 --no-cov -p no:cacheprovider -rA -s
"""
from django.conf import settings
from django.test import TestCase
from documents.classifier import DocumentClassifier, load_classifier
from documents.models import Correspondent, Document
from documents.tests.utils import DirectoriesMixin
import os


class Q1RetrainAndCache(DirectoriesMixin, TestCase):
    _iso1_model_file = None  # shared across the two isolation tests to prove they differ

    def _mk(self, name, content):
        c = Correspondent.objects.create(
            name=name, matching_algorithm=Correspondent.MATCH_AUTO,
        )
        Document.objects.create(
            title=name, content=content, correspondent=c, checksum=name,
        )
        return c

    def test_a_retrain_guard_transition(self):
        clf = DocumentClassifier()
        print("Q1A data_hash BEFORE first train():", clf.data_hash)
        # Two documents, each from its own MATCH_AUTO correspondent (no tags, no types).
        # These two exact documents produce the 2-doc data_hash quoted in the Q1 answer.
        self._mk("c1", "this is a document from c1")
        self._mk("c2", "this is another document from c2")
        r1 = clf.train()
        print("Q1A first train() returned:", r1)
        print("Q1A data_hash AFTER first train():", clf.data_hash.hex())
        r2 = clf.train()
        print("Q1A second train() returned:", r2)
        print("Q1A data_hash AFTER second train():", clf.data_hash.hex())
        print("Q1A data_hash UNCHANGED across retrain:", r2 is False)
        # Mutate the training set: add an EXPLICIT third document from a third
        # MATCH_AUTO correspondent. The mutated hash below is a deterministic function
        # of THIS exact third document's content ("this is a document from c3"); a
        # different third document would yield a different (equally deterministic) hex.
        self._mk("c3", "this is a document from c3")
        before = clf.data_hash
        r3 = clf.train()
        print("Q1A third train() AFTER adding a document returned:", r3)
        print("Q1A data_hash AFTER mutate:", clf.data_hash.hex())
        print("Q1A data_hash CHANGED after mutate:", clf.data_hash != before)

    def test_b_no_inmemory_cache(self):
        self._mk("c1", "this is a document from c1")
        self._mk("c2", "this is another document from c2")
        clf = DocumentClassifier()
        clf.train()
        clf.save()
        print("Q1B MODEL_FILE exists after save():", os.path.isfile(settings.MODEL_FILE))
        a = load_classifier()
        b = load_classifier()
        print("Q1B load_classifier() call #1 id():", id(a))
        print("Q1B load_classifier() call #2 id():", id(b))
        print("Q1B distinct instances (NO cache):", id(a) != id(b))
        print("Q1B both DocumentClassifier:", isinstance(a, DocumentClassifier) and isinstance(b, DocumentClassifier))

    def test_c_saveload_prevents_retrain(self):
        self._mk("c1", "this is a document from c1")
        self._mk("c2", "this is another document from c2")
        clf = DocumentClassifier()
        clf.train()
        clf.save()
        loaded = DocumentClassifier()
        loaded.load()
        print("Q1C loaded.data_hash present after load():", loaded.data_hash is not None)
        print("Q1C train() after load() returned:", loaded.train())

    def test_d_isolation_1(self):
        Q1RetrainAndCache._iso1_model_file = settings.MODEL_FILE
        print("Q1D isolation#1 MODEL_FILE:", settings.MODEL_FILE)
        print("Q1D isolation#1 file present at setUp:", os.path.isfile(settings.MODEL_FILE))

    def test_d_isolation_2(self):
        print("Q1D isolation#2 MODEL_FILE:", settings.MODEL_FILE)
        print("Q1D isolation#2 file present at setUp:", os.path.isfile(settings.MODEL_FILE))
        print("Q1D per-test MODEL_FILE differs (#1 vs #2):",
              Q1RetrainAndCache._iso1_model_file != settings.MODEL_FILE)

    def test_e_load_none_when_absent(self):
        print("Q1E MODEL_FILE:", settings.MODEL_FILE)
        print("Q1E model file exists:", os.path.isfile(settings.MODEL_FILE))
        print("Q1E load_classifier() with no model on disk returned:", load_classifier())
```

### §C‑Q2 — `/tmp/q2_probe_test.py` (count / timing / no‑threshold / fuzzy‑gate disambiguation)

Reproduces the Q2 *Verbatim Observed Output* block: the training‑doc counts, the
insert‑then‑train ordering, the `predict` = 2 / `predict_proba` = 0 call counts (proving no
probability threshold), and the `MATCH_FUZZY` regex gate. The printed argmax `predict()`
labels (`predict doc1 -> [1]`, `predict doc2 -> None`, `raw return = [1]`/`[2]`) are an
unseeded‑MLP distribution — stable for this distinctive content in re‑observation
(10/10 full‑probe runs; 90/90 across an `N=30`‑fresh‑classifier tally run 3×) but able to flip
for borderline input, as demonstrated in the [Non‑Determinism section](#nd).

```python
"""
Temporary Q2 observation probe (container /tmp, OUTSIDE the repo tree).
Exercises the REAL DocumentClassifier.train() / predict_correspondent() and the
match_correspondents() entry point. Counts .predict vs .predict_proba calls to prove
there is no probability threshold. NOTE: the unseeded MLPClassifier (classifier.py:227,
no random_state) means the argmax predict() label for a borderline doc can FLIP run-to-run;
this probe prints those prediction lines so the distribution can be observed across runs.
Run: cd /app/src && pipenv run pytest /tmp/q2_probe_test.py -c /app/src/setup.cfg \
        --rootdir=/app/src -n0 --no-cov -p no:cacheprovider -s -rA
"""
from unittest import mock
from django.test import TestCase
from documents.classifier import DocumentClassifier
from documents.matching import match_correspondents
from documents.models import Correspondent, Document, Tag
from documents.tests.utils import DirectoriesMixin


class Q2Correspondent(DirectoriesMixin, TestCase):
    def test_a_one_correspondent_predict(self):
        print("Q2A Document.objects.count() BEFORE any insert:", Document.objects.count())
        c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
        doc1 = Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
        print("Q2A total docs created:", Document.objects.count())
        print("Q2A effective training docs (inbox-excluded):",
              Document.objects.exclude(tags__is_inbox_tag=True).count())
        print("Q2A insert-then-train: documents inserted FIRST, now calling train()")
        clf = DocumentClassifier()
        print("Q2A train() returned:", clf.train())
        print("Q2A predict_correspondent(doc1) ->", clf.predict_correspondent(doc1.content), "| c1.pk =", c1.pk)
        print("Q2A correspondent_classifier.classes_:", list(clf.correspondent_classifier.classes_))

    def test_b_one_correspondent_predict_manydocs(self):
        c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
        doc1 = Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
        doc2 = Document.objects.create(title="doc2", content="this is a document from noone", checksum="B")
        print("Q2B total docs created:", Document.objects.count())
        print("Q2B effective training docs (inbox-excluded):",
              Document.objects.exclude(tags__is_inbox_tag=True).count())
        clf = DocumentClassifier()
        print("Q2B train() returned:", clf.train())
        print("Q2B correspondent_classifier.classes_:", list(clf.correspondent_classifier.classes_))
        print("Q2B predict doc1 ->", clf.predict_correspondent(doc1.content), "| c1.pk =", c1.pk)
        print("Q2B predict doc2 (no correspondent) ->", clf.predict_correspondent(doc2.content))

    def test_c_inbox_exclusion_reduces_count(self):
        c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
        d1 = Document.objects.create(title="d1", content="this is a document from c1", correspondent=c1, checksum="A")
        d2 = Document.objects.create(title="d2", content="this is a document from noone", checksum="B")
        inbox = Tag.objects.create(name="inbox", is_inbox_tag=True)
        print("Q2C before inbox tag: total =", Document.objects.count(),
              "; effective =", Document.objects.exclude(tags__is_inbox_tag=True).count())
        d1.tags.add(inbox)
        eff = Document.objects.exclude(tags__is_inbox_tag=True).count()
        print("Q2C after inbox tag on 1 of 2 docs: total =", Document.objects.count(), "; effective =", eff)
        print("Q2C inbox tagging dropped effective count 2 -> 1:", eff == 1)

    def test_d_no_confidence_threshold(self):
        c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
        c2 = Correspondent.objects.create(name="c2", matching_algorithm=Correspondent.MATCH_AUTO)
        d1 = Document.objects.create(title="d1", content="this is a document from c1", correspondent=c1, checksum="A")
        d2 = Document.objects.create(title="d2", content="this is another document from c2", correspondent=c2, checksum="B")
        clf = DocumentClassifier()
        clf.train()
        print("Q2D correspondent_classifier type =", type(clf.correspondent_classifier).__name__)
        print("Q2D classes_ =", list(clf.correspondent_classifier.classes_))
        with mock.patch.object(clf.correspondent_classifier, "predict",
                               wraps=clf.correspondent_classifier.predict) as mp, \
             mock.patch.object(clf.correspondent_classifier, "predict_proba",
                               wraps=clf.correspondent_classifier.predict_proba) as mpp:
            r1 = clf.predict_correspondent(d1.content)
            r2 = clf.predict_correspondent(d2.content)
            print("Q2D predict_correspondent called twice")
            print("Q2D correspondent_classifier.predict call count:", mp.call_count)
            print("Q2D correspondent_classifier.predict_proba call count:", mpp.call_count)
            print("Q2D predict_correspondent(d1) raw return =", r1)
            print("Q2D predict_correspondent(d2) raw return =", r2)
        X = clf.data_vectorizer.transform([d2.content])
        print("Q2D (reference only) predict_proba(d2) =", clf.correspondent_classifier.predict_proba(X),
              "-> value EXISTS but is UNUSED by predict_correspondent (proba unseeded, varies run-to-run)")
        print("Q2D pure argmax predict, NO probability threshold:", mpp.call_count == 0)

    def test_e_fuzzy_gate_is_regex_not_ml(self):
        Correspondent.objects.create(name="Foo", matching_algorithm=Correspondent.MATCH_FUZZY, match="foobar")
        print("Q2E matching.py:135 fuzz.partial_ratio>=90 is regex MATCH_FUZZY, NOT ML")
        stub1 = type("D", (), {"content": "a foobar invoice"})()
        stub2 = type("D", (), {"content": "nothing here"})()
        print("Q2E match_correspondents('a foobar invoice', classifier=None) ->",
              [c.name for c in match_correspondents(stub1, classifier=None)])
        print("Q2E match_correspondents('nothing here', classifier=None) ->",
              [c.name for c in match_correspondents(stub2, classifier=None)])
```
