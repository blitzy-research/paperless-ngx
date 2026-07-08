# paperless-ngx — ML Classification & OCR Pipeline: Runtime Investigation

**Branch:** `paperless-ngx_542221a38dff` &nbsp;•&nbsp; **HEAD commit:** `542221a38` ("Merge pull request #792 …")
**Deliverable:** evidence-based answers to four runtime-behavior questions, written **from directly observed runtime output** (build-and-run first, then write), in the project's **default / canonical configuration**.

> **Reading guide.** Each question section (Q1–Q4) **leads with the direct answer** (including negative results), then shows the **exact command(s)** run and their **complete, unedited output** in fenced blocks, and grounds every factual claim in a `file:line` reference or in observed output. Anything not directly observed is explicitly labelled **INFERRED**. A final **Coverage pass** re-lists every named item, and a **Repository cleanliness** section proves the repository is byte-for-byte unchanged except for this one document.

---

## Table of contents

1. [Environment & methodology](#1-environment--methodology)
2. [Q1 — Classifier model reuse vs. retraining & cross-test contamination](#2-q1--classifier-model-reuse-vs-retraining--cross-test-contamination)
3. [Q2 — Automatic correspondent matching: training-document count, timing, threshold](#3-q2--automatic-correspondent-matching-training-document-count-timing-threshold)
4. [Q3 — Document with no extractable text: OCR subprocess & output MIME](#4-q3--document-with-no-extractable-text-ocr-subprocess--output-mime)
5. [Q4 — Barcode splitting: record count, trigger values, decision site, training-data impact](#5-q4--barcode-splitting-record-count-trigger-values-decision-site-training-data-impact)
6. [Coverage pass](#6-coverage-pass)
7. [Repository cleanliness](#7-repository-cleanliness)

---

## 1. Environment & methodology

### 1.1 Run-first methodology

Every value in this document was produced by **building and running the real code paths** inside the project's canonical container, capturing the raw output, and only then writing the answer. Temporary observation scripts were written **outside the repository tree** (under `/tmp`), executed, and **removed before finishing** (see [§7](#7-repository-cleanliness)). No source, test, configuration, or fixture file was modified — the sole repository artifact is this Markdown document.

Where a question asks about magnitude / frequency / timing, the path was run **at scale** and the value confirmed **stable across at least two runs**. Where a question reports run-to-run inconsistency (Q1), the **same unchanged input** was run **repeatedly** and the observed distribution is reported — the behavior was **not** stabilized.

### 1.2 Canonical environment (observed)

All observations were captured inside the user-provided Docker image (`paperless-ngx-qna:ready`, derived from `ghcr.io/scaleapi/swe-atlas:…qna_1.01`), on a clean checkout of the repository baked at `/app` (HEAD `542221a38`). Commands were run as the non-root **`testuser`** (running as root causes false failures in permission tests and a root-owned scratch dir).

```
$ python --version
Python 3.9.23
$ (cd /app && git log --oneline -1)
542221a38 Merge pull request #792 from paperless-ngx/dependabot/github_actions/github/codeql-action-2
```

> **Observed vs. plan note.** The task plan referred to "Python 3.10"; the canonical image actually ships **Python 3.9.23** (reported above as observed). All results below are from this interpreter.

**Python dependency versions** — installed **runtime** versions reported by `importlib.metadata`. Exact command:

```
$ docker exec -u testuser pngx python - <<'PY'
import importlib.metadata as m
for p in ["scikit-learn","ocrmypdf","pyzbar","pdf2image","pikepdf","django",
          "python-magic","pdfminer.six","fuzzywuzzy","django-q","pillow","numpy",
          "scipy","joblib","img2pdf"]:
    print(f"{p}=={m.version(p)}")
print("--- test tooling (runtime, prepared image) ---")
for p in ["pytest","pytest-django","pytest-xdist","pytest-cov"]:
    print(f"{p}=={m.version(p)}")
PY
scikit-learn==1.0.2
ocrmypdf==13.4.3
pyzbar==0.1.9
pdf2image==1.16.0
pikepdf==5.1.1
django==4.0.4
python-magic==0.4.25
pdfminer.six==20220319
fuzzywuzzy==0.18.0
django-q==1.3.9
pillow==9.1.0
numpy==1.22.3
scipy==1.8.0
joblib==1.1.0
img2pdf==0.4.4
--- test tooling (runtime, prepared image) ---
pytest==8.4.2
pytest-django==4.11.1
pytest-xdist==3.8.0
pytest-cov==7.0.0
```

> **Dependency accuracy — locked project deps vs. prepared-image test tooling (important distinction).**
> The **runtime project dependencies** above **match `Pipfile.lock` `[default]`** exactly (scikit-learn 1.0.2, ocrmypdf 13.4.3, django 4.0.4, pyzbar 0.1.9, pdf2image 1.16.0, pikepdf 5.1.1, python-magic 0.4.25, pdfminer.six 20220319, fuzzywuzzy 0.18.0, django-q 1.3.9, pillow 9.1.0). These are the versions that govern the behaviors under investigation, so **all Q1–Q4 results are on the locked project stack.**
> The **test tooling** shown (`pytest 8.4.2`, `pytest-django 4.11.1`, `pytest-xdist 3.8.0`, `pytest-cov 7.0.0`) is the **prepared-image runtime** version and does **not** match `Pipfile.lock` `[develop]`, which pins `pytest 7.1.1`, `pytest-django 4.5.2`, `pytest-xdist 2.5.0`, `pytest-cov 3.0.0`. The prepared image ships newer test runners; this is a **test-harness** difference only and does not affect the classifier/OCR/barcode behaviors observed below. The raw lock-file comparison (exact command + output):

```
$ docker exec -u testuser pngx python - <<'PY'
import json
d=json.load(open("/app/Pipfile.lock"))
default, dev = d["default"], d["develop"]
print("[Pipfile.lock DEFAULT / runtime project deps]")
for k in ["scikit-learn","ocrmypdf","pyzbar","pdf2image","pikepdf","django",
          "python-magic","pdfminer.six","fuzzywuzzy","django-q","pillow"]:
    if k in default: print("  " + k + default[k]["version"])
print("[Pipfile.lock DEVELOP / test tooling]")
for k in ["pytest","pytest-django","pytest-xdist","pytest-cov","factory-boy"]:
    if k in dev: print("  " + k + dev[k]["version"])
PY
[Pipfile.lock DEFAULT / runtime project deps]
  scikit-learn==1.0.2
  ocrmypdf==13.4.3
  pyzbar==0.1.9
  pdf2image==1.16.0
  pikepdf==5.1.1
  django==4.0.4
  python-magic==0.4.25
  pdfminer.six==20220319
  fuzzywuzzy==0.18.0
  django-q==1.3.9
  pillow==9.1.0
[Pipfile.lock DEVELOP / test tooling]
  pytest==7.1.1
  pytest-django==4.5.2
  pytest-xdist==2.5.0
  pytest-cov==3.0.0
  factory-boy==3.2.1
```

**System binaries** used by the OCR (Q3) and barcode (Q4) paths. Exact command:

```
$ docker exec -u testuser pngx bash -c 'tesseract --version 2>&1 | head -1; \
    gs --version 2>&1 | head -1 | sed "s/^/ghostscript /"; \
    unpaper --version 2>&1 | head -1 | sed "s/^/unpaper /"; \
    pdftoppm -v 2>&1 | head -1; \
    ls /usr/lib/x86_64-linux-gnu/libzbar.so.*'
tesseract 4.1.1
ghostscript 9.53.3
unpaper 6.1
pdftoppm version 20.09.0
/usr/lib/x86_64-linux-gnu/libzbar.so.0
/usr/lib/x86_64-linux-gnu/libzbar.so.0.3.0
```

(`pdftoppm` is the poppler-utils backend for `pdf2image`; `libzbar.so.0.3.0` is the zbar backend for `pyzbar`.)

### 1.3 Test configuration (`src/setup.cfg`)

```ini
[tool:pytest]
DJANGO_SETTINGS_MODULE = paperless.settings          # setup.cfg:L9
addopts = --pythonwarnings=all --cov --cov-report=html --numprocesses auto --quiet   # setup.cfg:L10
env =
    PAPERLESS_DISABLE_DBHANDLER=true                 # setup.cfg:L11-12
```

Two configuration facts matter for the investigation:

- **`--numprocesses auto`** (`setup.cfg:L10`) means the _default_ test run is **parallel** via `pytest-xdist` (this host reports 128 CPUs). This is one of the two variables probed for Q1 non-determinism.
- The default test database is **in-memory SQLite**, and standard Django `TestCase` tests are **transaction-wrapped and rolled back** per test (relevant to Q1/Q2 cross-test isolation).

### 1.4 Invocation used throughout

All commands in this document are executed inside the **running container named `pngx`** as the non-root **`testuser`**, wrapped as:

```
docker exec -u testuser pngx bash -c '<COMMAND>'
```

Each evidence block below shows its exact `<COMMAND>` (prefixed `$`) immediately before its complete, unedited output, so every observation is independently reproducible.

**Why `python -m pytest` and not `pipenv run` (observed).** The pinned dependencies are installed **system-wide** in the prepared image, so `python -m pytest` is the canonical runner. `pipenv run` instead builds a _new empty_ virtualenv that lacks the test deps — observed directly:

```
$ cd /app/src && pipenv run python -m pytest documents/tests/test_tasks.py -k test_train_classifier -n0 --no-cov -q
Using /usr/local/bin/python3.9.23 to create virtualenv...
created virtual environment CPython3.9.23.final.0-64 in 1331ms
✔ Successfully created virtual environment!
Virtualenv location: /home/testuser/.local/share/virtualenvs/app-4PlAip0Q
/home/testuser/.local/share/virtualenvs/app-4PlAip0Q/bin/python: No module named pytest
```

**Canonical test-suite command form** (in-repo tests):

```
$ cd /app/src && python -m pytest <path> -k "<expr>" -n0 --no-cov -p no:cacheprovider -p no:warnings -v
```

**Canonical `/tmp` observation-spy command form** (temporary scripts live under `/tmp`, outside the repo):

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/<script>.py --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
```

Every non-default flag, and why each is behavior-neutral for the code paths under investigation:

- `-n0` disables `pytest-xdist` (single process) — removes scheduler ordering as a variable. Where a question needs the **default parallel** configuration, `-n0` is replaced by **`-n auto`** (identical to `setup.cfg`'s `--numprocesses auto` [`setup.cfg:L10`]).
- `--no-cov` skips coverage overhead (no behavioral effect); `-p no:cacheprovider` avoids cache noise.
- `--ds=paperless.settings` + `PYTHONPATH=/app/src` are needed for `/tmp` spies because running from `/tmp` moves pytest's rootdir away from `/app/src/setup.cfg`.
- `-p no:warnings` disables the pytest warnings plugin **at the source** so the command emits **no** warnings-summary block. This is _not_ post-hoc editing: each pasted block is the **complete, unedited output of the exact command shown above it** — no `...`, no `{...}`, no hand-deletion.

> **What `-p no:warnings` suppresses (shown once, in full, for transparency).** The only warnings the canonical `--pythonwarnings=all` [`setup.cfg:L10`] surfaces are **pre-existing third-party deprecations**, unrelated to the classifier/OCR/barcode behaviors. Here is one representative run **with** the warnings left in, complete and unedited, so nothing is hidden:
>
> ```
> $ cd /app/src && python -m pytest documents/tests/test_tasks.py -k test_train_classifier -n0 --no-cov -p no:cacheprovider -v
> ============================= test session starts ==============================
> platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
> django: version: 4.0.4, settings: paperless.settings (from ini)
> rootdir: /app/src
> configfile: setup.cfg
> plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
> collected 40 items / 35 deselected / 5 selected
>
> documents/tests/test_tasks.py .....                                      [100%]
>
> =============================== warnings summary ===============================
> ../../usr/local/lib/python3.9/site-packages/redis/connection.py:67
>   /usr/local/lib/python3.9/site-packages/redis/connection.py:67: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
>     hiredis_version = StrictVersion(hiredis.__version__)
>
> ../../usr/local/lib/python3.9/site-packages/redis/connection.py:69
>   /usr/local/lib/python3.9/site-packages/redis/connection.py:69: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
>     hiredis_version >= StrictVersion('0.1.3')
>
> ../../usr/local/lib/python3.9/site-packages/redis/connection.py:71
>   /usr/local/lib/python3.9/site-packages/redis/connection.py:71: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
>     hiredis_version >= StrictVersion('0.1.4')
>
> ../../usr/local/lib/python3.9/site-packages/redis/connection.py:73
>   /usr/local/lib/python3.9/site-packages/redis/connection.py:73: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
>     hiredis_version >= StrictVersion('1.0.0')
>
> ../../usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229
>   /usr/local/lib/python3.9/site-packages/django/conf/__init__.py:229: RemovedInDjango50Warning: The USE_L10N setting is deprecated. Starting with Django 5.0, localized formatting of data will always be enabled. For example Django will display numbers and dates using the format of the current locale.
>     warnings.warn(USE_L10N_DEPRECATED_MSG, RemovedInDjango50Warning)
>
> ../../usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9
>   /usr/local/lib/python3.9/site-packages/django_q/core_signing.py:9: RemovedInDjango50Warning: The django.utils.baseconv module is deprecated.
>     from django.utils import baseconv
>
> -- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
> ================= 5 passed, 35 deselected, 6 warnings in 2.17s =================
> ```
>
> Every other block in this document uses `-p no:warnings`, so this identical, unrelated summary does not repeat — and what is pasted is always the command's complete output.

---

## 2. Q1 — Classifier model reuse vs. retraining & cross-test contamination

### 2.1 Direct answer

Within a single run, whether the classifier **reuses** an already-built model or **retrains** is governed by **two independent mechanisms**:

1. **Persisted-model reuse across the process** — `load_classifier()` reads the pickle at `settings.MODEL_FILE`; if the file is **absent** it returns `None` (no matching happens); if **present** it deserializes the prior model [`src/documents/classifier.py:L30-L57`]. The default is `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` [`src/paperless/settings.py:L74`].
2. **A retrain-skip guard inside `DocumentClassifier.train()`** — a SHA-1 `data_hash` is computed over all preprocessed training content; if it equals the previously stored hash, `train()` returns `False` and **does not retrain and does not re-save**: `if self.data_hash and new_data_hash == self.data_hash: return False` [`src/documents/classifier.py:L163-L164`]. `train_classifier()` only calls `save()` when `train()` is truthy, otherwise it logs `"Training data unchanged."` [`src/documents/tasks.py:L48-L72`, guard/save at `L63-L69`].

**Effect on later tests / cross-test contamination:** in the test suite this reuse is **contained**, for two reasons observed below — (a) `DirectoriesMixin` gives every test a **fresh** `MODEL_FILE` under its own `tempfile.mkdtemp()` [`src/documents/tests/utils.py:L14-L50`, override at `L45`], and (b) each `TestCase` runs in a **rolled-back transaction** so DB rows do not leak. What is **not** contained is the **algorithmic non-determinism** of the classifier itself: the three `MLPClassifier(tol=0.01)` instances are constructed with **no `random_state`** [`src/documents/classifier.py:L219` (tags), `L227` (correspondent), `L238` (document type)], so an _identical_ training set yields a _different_ fitted model on every run — which is the real source of the reported flakiness (§2.4).

### 2.2 Reuse-vs-retrain mechanism — evidence

**Golden test** `test_train_classifier` [`src/documents/tests/test_tasks.py:L75`] creates a `Correspondent(MATCH_AUTO)` + one `Document`, calls `train_classifier()` (asserts `MODEL_FILE` now exists, captures `mtime`), calls it **again** (asserts `mtime` **unchanged** — the `data_hash` guard skipped retraining), then mutates `doc.content` and calls it a third time (asserts `mtime` **changed** — data changed ⇒ retrain + re-save).

```
$ cd /app/src && python -m pytest documents/tests/test_tasks.py -k test_train_classifier -n0 --no-cov -p no:cacheprovider -p no:warnings -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 40 items / 35 deselected / 5 selected

documents/tests/test_tasks.py .....                                      [100%]

======================= 5 passed, 35 deselected in 2.03s =======================
```

**Instrumented guard (temporary `/tmp` spy).** A `DirectoriesMixin`/`TestCase` spy (`/tmp/test_blitzy_q1_reuse.py`, methods `test_partA_reuse_retrain` + `test_partB_datahash`) exercised both entry points and printed the **before / intermediate / after** state of `MODEL_FILE` and the `data_hash` values. The spy routes the `paperless.tasks`/`paperless.classifier` loggers to stdout, so the `[INFO] … Saving updated classifier model` line [`tasks.py:L64`] and the `[DEBUG] … Training data unchanged.` line [`tasks.py:L69`] are visible verbatim. Exact command and its **complete** output:

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q1_reuse.py -k TestQ1PartAB \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 4 items / 2 deselected / 2 selected

../../tmp/test_blitzy_q1_reuse.py::TestQ1PartAB::test_partA_reuse_retrain Creating test database for alias 'default'...

=== PART A: reuse-vs-retrain via tasks.train_classifier() (golden scenario) ===
MODEL_FILE = /tmp/tmpwe_a7hxw/classification_model.pickle
[before 1st train_classifier] exists=False
[DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.classifier] 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
[DEBUG] [paperless.classifier] Vectorizing data...
[DEBUG] [paperless.classifier] There are no tags. Not training tags classifier.
[DEBUG] [paperless.classifier] Training correspondent classifier...
[DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpwe_a7hxw/classification_model.pickle...
[after  1st train_classifier] exists=True size=12421B mtime=1783494901.939924
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.tasks] Training data unchanged.
[after  2nd train_classifier (UNCHANGED data)] exists=True size=12421B mtime=1783494901.939924
  -> mtime changed after 2nd call? False
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.classifier] 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
[DEBUG] [paperless.classifier] Vectorizing data...
[DEBUG] [paperless.classifier] There are no tags. Not training tags classifier.
[DEBUG] [paperless.classifier] Training correspondent classifier...
[DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpwe_a7hxw/classification_model.pickle...
[after  3rd train_classifier (MUTATED data)  ] exists=True size=12422B mtime=1783494901.954924
  -> mtime changed after 3rd call? True
PASSED
../../tmp/test_blitzy_q1_reuse.py::TestQ1PartAB::test_partB_datahash
=== PART B: DocumentClassifier.train() data_hash reuse guard [classifier.py:L163-164] ===
initial clf.data_hash = None
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.classifier] 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
[DEBUG] [paperless.classifier] Vectorizing data...
[DEBUG] [paperless.classifier] There are no tags. Not training tags classifier.
[DEBUG] [paperless.classifier] Training correspondent classifier...
[DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
1st train() return = True   data_hash(sha1 hex) = 230b98c1cbe4bb261c254b08a6d463334e9ba45d
[DEBUG] [paperless.classifier] Gathering data from database...
2nd train() return = False  data_hash(sha1 hex) = 230b98c1cbe4bb261c254b08a6d463334e9ba45d  (SAME instance, UNCHANGED DB)
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.classifier] 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
[DEBUG] [paperless.classifier] Vectorizing data...
[DEBUG] [paperless.classifier] There are no tags. Not training tags classifier.
[DEBUG] [paperless.classifier] Training correspondent classifier...
[DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
3rd train() return = True   data_hash(sha1 hex) = faa493d006d464bdfc90cc8375754ee573e26b13  (DB MUTATED: +1 doc)
  hash unchanged 1->2? True   hash changed 2->3? True
PASSED
Destroying test database for alias 'default'...

======================= 2 passed, 2 deselected in 2.23s ========================
```

**Reading the state transitions:**

| Step                  | `MODEL_FILE`                    | `train()` return | `data_hash`             | Interpretation                                                      |
| --------------------- | ------------------------------- | ---------------- | ----------------------- | ------------------------------------------------------------------- |
| before 1st            | `exists=False`                  | —                | `None`                  | no model yet ⇒ `load_classifier()` would return `None`              |
| after 1st             | `size=12421B mtime=…901.939924` | `True`           | `230b98c1…`             | trained + saved (INFO "Saving updated classifier model…")           |
| after 2nd (unchanged) | `size=12421B` **same mtime**    | `False`          | `230b98c1…` (unchanged) | **guard hit** [`classifier.py:L163-L164`] ⇒ no retrain, **no save** |
| after 3rd (mutated)   | `size=12422B` **new mtime**     | `True`           | `faa493d0…` (changed)   | data changed ⇒ retrain + re-save                                    |

This is exactly the reuse-vs-retrain contract: the SHA-1 `new_data_hash = m.digest()` [`classifier.py:L161`] short-circuits work when the training corpus is unchanged.

**Save/version round-trip & fixture load** (`save()`/`load()` at [`classifier.py:L96`,`L76`]):

```
$ cd /app/src && python -m pytest documents/tests/test_classifier.py \
    -k "testDatasetHashing or testSaveClassifier or test_load_and_classify or test_load_classifier_cached" \
    -n0 --no-cov -p no:cacheprovider -p no:warnings -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 23 items / 19 deselected / 4 selected

documents/tests/test_classifier.py::TestClassifier::testDatasetHashing PASSED [ 25%]
documents/tests/test_classifier.py::TestClassifier::testSaveClassifier PASSED [ 50%]
documents/tests/test_classifier.py::TestClassifier::test_load_and_classify PASSED [ 75%]
documents/tests/test_classifier.py::TestClassifier::test_load_classifier_cached SKIPPED [100%]

================= 3 passed, 1 skipped, 19 deselected in 2.03s ==================
```

- `testDatasetHashing` [`test_classifier.py:L137`] — `assertTrue(train())` then `assertFalse(train())` (guard). **PASS.**
- `testSaveClassifier` [`test_classifier.py:L168`] — train → `save()` → fresh `load()` → `assertFalse(train())`. **PASS.**
- `test_load_and_classify` [`test_classifier.py:L183`] — loads the **committed** fixture `src/documents/tests/data/model.pickle` (**156,607 bytes**) under `override_settings(MODEL_FILE=…)`. **PASS.** (This is a _controlled_ reuse — not contamination.)
- `test_load_classifier_cached` [`test_classifier.py:L402`] — **SKIPPED**, reason `"Disabled caching due to high memory usage - need to investigate."` (the `@pytest.mark.skip` decorator is declared at `test_classifier.py:L399`; pytest's own summary reports the skip location as `test_classifier.py:391` — the first line of the test's stacked decorators, `@override_settings` at `L391`). Verified with `-rs`: `SKIPPED [1] documents/tests/test_classifier.py:391: Disabled caching due to high memory usage - need to investigate.`

### 2.3 Cross-test contamination surface — evidence

**Per-test filesystem isolation.** `DirectoriesMixin` [`src/documents/tests/utils.py:L72-L83`] calls `setup_directories()` which sets `data_dir = tempfile.mkdtemp()` [`utils.py:L18`] and overrides `MODEL_FILE = os.path.join(dirs.data_dir, "classification_model.pickle")` [`utils.py:L45`]; `tearDown` → `remove_dirs` wipes them. Two `DirectoriesMixin` tests in the spy (`/tmp/test_blitzy_q1_reuse.py`, class `TestQ1IsoA`) therefore get **different** model paths and each enters with an empty DB. Exact command and its **complete** output:

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q1_reuse.py -k TestQ1IsoA \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 4 items / 2 deselected / 2 selected

../../tmp/test_blitzy_q1_reuse.py::TestQ1IsoA::test_iso_1 Creating test database for alias 'default'...

[iso test 1] entry Document.objects.count() = 0
[iso test 1] after create count = 1
[iso test 1] MODEL_FILE = /tmp/tmpxiy5xzu3/classification_model.pickle
PASSED
../../tmp/test_blitzy_q1_reuse.py::TestQ1IsoA::test_iso_2 [iso test 2] entry Document.objects.count() = 0
[iso test 2] after create count = 1
[iso test 2] MODEL_FILE = /tmp/tmpjdtrqnbt/classification_model.pickle
PASSED
Destroying test database for alias 'default'...

======================= 2 passed, 2 deselected in 1.62s ========================
```

**DB isolation.** Both tests **enter with `count() = 0`** even though each creates a row — the `TestCase` transaction is rolled back between them, so `Document`/`Correspondent` rows created in one test do **not** persist to the next. This is why `train()` only ever sees the **current** test's DB snapshot via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`src/documents/classifier.py:L125-L127`].

**Parallelism.** With the default `--numprocesses auto` [`setup.cfg:L10`] the full classifier file runs across workers. Exact command and its **complete, unedited** output — the entire `pytest-xdist` progress bar fits on the single row `s......................` (one `s` for the skipped test + 22 `.` for the 22 passers = all 23 collected tests), so **nothing is elided**. The pass/skip counts are stable run-to-run (a second run also reported `22 passed, 1 skipped`; only the wall-clock, ≈ 30 s, varies):

```
$ cd /app/src && python -m pytest documents/tests/test_classifier.py -n auto --no-cov -p no:cacheprovider -p no:warnings
bringing up nodes...
bringing up nodes...

s......................                                                  [100%]
22 passed, 1 skipped in 30.03s
```

Combined with per-test tempdirs, on-disk model leakage is unlikely; what parallelism _does_ change is **test ordering** run-to-run (worker scheduling) and it gives **each xdist worker a fresh, separately-seeded process** (relevant in §2.4). **INFERRED:** test ordering alone is not the flake cause, because single-process (`-n0`) runs already produce a different model every run (§2.4).

### 2.4 Reproduced non-determinism (not stabilized)

**Root cause (grounded):** the three `MLPClassifier(tol=0.01)` constructions carry **no `random_state`** — `src/documents/classifier.py:L219` (tags), `L227` (correspondent), `L238` (document type). With no seed, scikit-learn draws the network's initial weights from NumPy's global RNG, so the fitted decision boundary — and therefore any prediction sitting near it — differs across otherwise-identical runs. The two facets below are reported **separately** because they reproduce very differently: the **model difference is 100% reproducible**, whereas the **label flip that actually fails the test is a rare tail event** (this distinction is the honest core of the "sometimes passes, sometimes fails" report).

#### 2.4.1 The non-determinism itself — 100% reproducible (every model is distinct)

A single unified spy runs the **real** `DocumentClassifier.train()` on the **identical** 2-document DB — `c1` (MATCH_AUTO); `doc1→c1`; `doc2→` no correspondent, i.e. the exact rows of `test_one_correspondent_predict_manydocs` [`src/documents/tests/test_classifier.py:L206-L225`] — **K** times in one `-n0` process, hashing each fitted `correspondent_classifier`, recording every `predict_correspondent` output, and measuring the arg-max margin `P(x→c1)` via `predict_proba`. Core of `/tmp/test_q1_nondet.py` (temporary; removed before finishing):

```python
# /tmp/test_q1_nondet.py  — K from env Q1_K; identical 2-doc DB rebuilt is NOT needed (rows fixed), train() K times
for _ in range(K):
    clf = DocumentClassifier()
    clf.train()                                                     # real training; no random_state
    coefs_hashes.add(sha256(coefs_[0].tobytes() + coefs_[-1].tobytes()))
    pickle_hashes.add(sha256(pickle.dumps(clf.correspondent_classifier)))
    r1 = clf.predict_correspondent(doc1.content)                    # expect c1.pk (== 1)
    r2 = clf.predict_correspondent(doc2.content)                    # expect None
    p_doc1.append(predict_proba(doc1)[c1_index]); p_doc2.append(predict_proba(doc2)[c1_index])
```

**Run 1, K = 500** (single-process `-n0`). Exact command and its **complete, unedited** output:

```
$ cd /app/src && PYTHONPATH=/app/src Q1_K=500 python -m pytest /tmp/test_q1_nondet.py -k test_nondet \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 1 item

../../tmp/test_q1_nondet.py::TestQ1Nondet::test_nondet Creating test database for alias 'default'...

=== Q1 NON-DETERMINISM SPY : K=500 fresh train() on identical 2-doc DB ===
classes learned = [-1, 1] ; class index for c1.pk(1) = 1
c1.pk = 1
predict_correspondent(doc1) distribution: {1: 500}  (expected all == 1 )
predict_correspondent(doc2) distribution: {None: 500}  (expected all == None)
doc2 label FLIP count (predicted a correspondent instead of None): 0 / 500
doc1 label FLIP count (predicted None/other instead of c1):        0 / 500
distinct correspondent coefs_ sha256 hashes: 500 / 500
distinct serialized correspondent-classifier pickle hashes: 500 / 500
P(doc1->c1)  min=0.5397 max=0.7474 mean=0.6378 std=0.0356
P(doc2->c1)  min=0.2558 max=0.4609 mean=0.3646 std=0.0356
  #runs with P(doc2->c1) > 0.5 (would flip doc2 to c1): 0 / 500
  distinct P(doc2->c1) values: 500 / 500
PASSED
Destroying test database for alias 'default'...

============================== 1 passed in 10.83s ==============================
```

The **load-bearing, always-reproducible** observation is here: from the _identical_ training set, **every one of the 500 fitted models is distinct** — `correspondent_classifier.coefs_` hash **500/500 distinct**, serialized `model.pickle` hash **500/500 distinct**, and **500/500 distinct** `P(doc2→c1)` values. That per-run model difference **is** the non-determinism, and it appears on every run at every scale.

**Cross-run + scale confirmation** (the ≥2-run stability the methodology requires). Same command with `Q1_K` changed; shown is the spy's own self-printed report (the two constant header lines `classes learned = [-1, 1]` / `c1.pk = 1` and the identical pytest session header/footer are the only lines omitted, and that omission is disclosed here):

```
# Run 2, K = 500:
predict_correspondent(doc1) distribution: {1: 500}  (expected all == 1 )
predict_correspondent(doc2) distribution: {None: 500}  (expected all == None)
doc2 label FLIP count (predicted a correspondent instead of None): 0 / 500
doc1 label FLIP count (predicted None/other instead of c1):        0 / 500
distinct correspondent coefs_ sha256 hashes: 500 / 500
distinct serialized correspondent-classifier pickle hashes: 500 / 500
P(doc1->c1)  min=0.5248 max=0.7380 mean=0.6370 std=0.0385
P(doc2->c1)  min=0.2634 max=0.4798 mean=0.3625 std=0.0393
  #runs with P(doc2->c1) > 0.5 (would flip doc2 to c1): 0 / 500
  distinct P(doc2->c1) values: 500 / 500
1 passed in 11.95s

# K = 2000:
predict_correspondent(doc1) distribution: {1: 2000}  (expected all == 1 )
predict_correspondent(doc2) distribution: {None: 2000}  (expected all == None)
doc2 label FLIP count (predicted a correspondent instead of None): 0 / 2000
doc1 label FLIP count (predicted None/other instead of c1):        0 / 2000
distinct correspondent coefs_ sha256 hashes: 2000 / 2000
distinct serialized correspondent-classifier pickle hashes: 2000 / 2000
P(doc1->c1)  min=0.5129 max=0.7476 mean=0.6370 std=0.0386
P(doc2->c1)  min=0.2320 max=0.4920 mean=0.3626 std=0.0382
  #runs with P(doc2->c1) > 0.5 (would flip doc2 to c1): 0 / 2000
  distinct P(doc2->c1) values: 2000 / 2000
1 passed in 42.17s
```

`distinct == K` in **all three** runs (500/500, 500/500, 2000/2000): the model-level non-determinism is **stable and total**. The margin distribution is likewise **cross-run stable** — `P(doc2→c1)` mean ≈ 0.36 (std ≈ 0.038) and `P(doc1→c1)` mean ≈ 0.64 in every run.

#### 2.4.2 Why the flip is rare — the arg-max margin

`predict_correspondent()` accepts the classifier's **arg-max** label with **no probability threshold** (see [Q2 §3.1](#31-lead-negative-result)). `doc2` is predicted `None` (arg-max class `-1`) as long as `P(doc2→c1) < 0.5`, and only **flips** to `c1` when a particular run's weights push `P(doc2→c1)` **across `0.5`**. The spy quantifies how far that margin sits from the boundary: across the 3000 identical-input trainings above, `P(doc2→c1)` reached a max of only **0.4609 / 0.4798 / 0.4920** and crossed `0.5` in **0/500, 0/500, 0/2000** runs, while `P(doc1→c1)` stayed **above** `0.5` (min **0.5397 / 0.5248 / 0.5129**). **Derived from the observed mean/std**, the `doc2` boundary is ≈ (0.5 − 0.363)/0.038 ≈ **3.5–3.8 σ** away — so most batches of a few hundred-to-thousand runs show **zero** flips and the test **passes**.

#### 2.4.3 Exhibiting the rare flip at scale — K = 30 000

To actually observe the flip and measure its frequency, a second spy `/tmp/test_q1_flip.py` runs the identical training **30 000** times and counts boundary crossings. Exact command and its **complete, unedited** captured output (`1 passed` after ≈ 9 minutes):

```
$ cd /app/src && PYTHONPATH=/app/src Q1_K=30000 python -m pytest /tmp/test_q1_flip.py -k test_flip \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -q

=== Q1 LARGE-SCALE FLIP EXHIBITION : K=30000 fresh train() on identical 2-doc DB ===
c1.pk = 1 ; assertion under test: predict_correspondent(doc2) is None
doc2 FLIP count (predicted a correspondent instead of None): 1 / 30000  (rate 3.33e-05)
doc1 FLIP count (predicted None/other instead of c1):        2 / 30000
max observed P(doc2->c1) over the batch: 0.5010
first observed doc2 flips (iteration, predicted_label, P(doc2->c1)):
   iter=14317  predict_correspondent(doc2)=array([1])  P=0.5010  -> 'assert array([1]) is None' FAILS
.
1 passed in 544.61s (0:09:04)
```

At K = 30 000 the flip is finally observed: **`doc2` flips 1/30 000 (rate 3.33 × 10⁻⁵)** and **`doc1` flips 2/30 000**. Because the manydocs test asserts **both** `predict_correspondent(doc1) == c1.pk` **and** `predict_correspondent(doc2) is None` [`test_classifier.py:L223-L225`], the test would fail on **≈ 3/30 000 ≈ 1 × 10⁻⁴** of identical-input runs. The single `doc2` flip only barely crossed the boundary (`max P(doc2→c1) = 0.5010`), producing exactly the failing prediction `array([1])` the real test asserts against.

**Cross-run confirmation — a second K = 30 000 run** (identical command; the spy's self-printed report, its pytest header/footer omitted as disclosed). This satisfies the ≥ 2-run magnitude check — and, tellingly, exhibits a *different* per-facet split:

```
=== Q1 LARGE-SCALE FLIP EXHIBITION : K=30000 fresh train() on identical 2-doc DB ===
c1.pk = 1 ; assertion under test: predict_correspondent(doc2) is None
doc2 FLIP count (predicted a correspondent instead of None): 0 / 30000  (rate 0.00e+00)
doc1 FLIP count (predicted None/other instead of c1):        4 / 30000
max observed P(doc2->c1) over the batch: 0.4992
no doc2 flip observed in this batch of K=30000
.
1 passed in 519.25s (0:08:39)
```

Across the two K = 30 000 runs the **aggregate test-failure rate is stable at ≈ 1 × 10⁻⁴** — run 1 had **3** boundary crossings (1 `doc2` + 2 `doc1` = 3/30 000 ≈ 1.0 × 10⁻⁴), run 2 had **4** (0 `doc2` + 4 `doc1` = 4/30 000 ≈ 1.3 × 10⁻⁴). But the **per-facet split is itself stochastic**: `doc2` flipped once in run 1 and **not at all** in run 2 (`max P(doc2→c1) = 0.4992`, never crossing `0.5`), while `doc1` flipped **2 then 4** times. That is direct evidence that *which* assertion fails — and how many times — is **not reproducible**, whereas the order-of-magnitude rate (≈ 1e-4) and the model-distinctness (`K/K`) are the stable, reproducible quantities. (In run 2 the manydocs test would still fail ≈ 4/30 000, but via the `predict_correspondent(doc1) == c1.pk` assertion rather than the `doc2` one.)

#### 2.4.4 Observed distribution & why it is "sometimes passes, sometimes fails"

| Scale (identical input)   | distinct models       | `doc2` flips | `doc1` flips | max `P(doc2→c1)` | test verdict     |
| ------------------------- | --------------------- | ------------ | ------------ | ---------------- | ---------------- |
| K = 500, run 1            | 500 / 500             | 0            | 0            | 0.4609           | would pass       |
| K = 500, run 2            | 500 / 500             | 0            | 0            | 0.4798           | would pass       |
| K = 2000                  | 2000 / 2000           | 0            | 0            | 0.4920           | would pass       |
| K = 30000, run 1          | distinct every iter   | 1            | 2            | 0.5010           | would fail ~3×   |
| K = 30000, run 2          | distinct every iter   | 0            | 4            | 0.4992           | would fail ~4×   |

**Reading — exactly what was observed, including the honest, non-reproducible part:**

- **What reproduces every time:** the fitted model differs on every run (`distinct == K` at all scales) and the margin distribution is stable. This is fully reproducible and is the true, code-grounded non-determinism — root cause the missing `random_state` [`src/documents/classifier.py:L219,L227,L238`].
- **What does _not_ reproduce per batch:** the actual label **flip** that fails the test. It is a **≈ 1 × 10⁻⁴ tail event**, so a batch of K ≤ 2000 (and a single run of the real test) usually shows **0 flips and passes** — which is exactly why re-running the K = 500 / K = 2000 spies, or the real test, most often reports all-`None`/all-pass. One must run **≈ K = 30 000** to reliably observe even one flip. This is the direct, honest explanation of "**sometimes passes, sometimes fails**": the failure probability per run is ≈ 0.01%, not a per-run coin toss.
- **Consequence for reproducibility:** because the flip is stochastic, **no specific failing index or per-batch flip count is reproducible** — a captured failure lands on a different case each run (shown in §2.4.5). Only the aggregate rate (≈ 1e-4) and the model-distinctness (`K/K`) are stable, reproducible quantities.

#### 2.4.5 The same flip at the pytest level

Normally the real test **passes** — six consecutive single-process runs of the actual project test all pass:

```
$ cd /app/src && for r in 1 2 3 4 5 6; do python -m pytest \
    "documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs" \
    -n0 --no-cov -p no:cacheprovider -p no:warnings -v 2>&1 | grep -E "==.*passed"; done
============================== 1 passed in 1.89s ===============================
============================== 1 passed in 2.26s ===============================
============================== 1 passed in 2.46s ===============================
============================== 1 passed in 1.90s ===============================
============================== 1 passed in 1.92s ===============================
============================== 1 passed in 1.90s ===============================
```

To surface the rare failure at the pytest level within a feasible wall-clock, a temporary spy parametrizes the **exact** manydocs body N times so the identical case is scheduled repeatedly across `pytest-xdist` workers (default `--numprocesses auto`). Complete `/tmp/test_q1_pytest_flake.py` body (temporary; removed before finishing):

```python
# /tmp/test_q1_pytest_flake.py — N from env Q1_N; identical 2-doc input every case
@pytest.mark.django_db
@pytest.mark.parametrize("i", range(N))
def test_manydocs_repeat(i):
    c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
    doc1 = Document.objects.create(title="doc1", content="this is a document from c1", correspondent=c1, checksum="A")
    doc2 = Document.objects.create(title="doc2", content="this is a document from noone", checksum="B")
    clf = DocumentClassifier(); clf.train()
    assert clf.predict_correspondent(doc1.content) == c1.pk
    assert clf.predict_correspondent(doc2.content) is None
```

Because the flip is ≈ 1e-4, most parallel batches at N = 1000 pass (`1000 passed`); a failure appears sporadically. The following is one **caught** run — its **complete, unedited** output, all nine `pytest-xdist` progress rows included (nothing elided):

```
$ cd /app/src && PYTHONPATH=/app/src COLUMNS=120 Q1_N=1000 python -m pytest /tmp/test_q1_pytest_flake.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -n auto --no-cov -q
bringing up nodes...
bringing up nodes...

................................................................................................................ [ 11%]
................................................................................................................ [ 22%]
................................................................................................................ [ 33%]
................................................................................................................ [ 44%]
................................................................................................................ [ 56%]
................................................................................................................ [ 67%]
................................................................................................................ [ 78%]
.................................................................................F.............................. [ 89%]
........................................................................................................         [100%]
======================================================= FAILURES =======================================================
______________________________________________ test_manydocs_repeat[705] _______________________________________________
[gw118] linux -- Python 3.9.23 /usr/local/bin/python

i = 705

    @pytest.mark.django_db
    @pytest.mark.parametrize("i", range(N))
    def test_manydocs_repeat(i):
        c1 = Correspondent.objects.create(
            name="c1", matching_algorithm=Correspondent.MATCH_AUTO,
        )
        doc1 = Document.objects.create(
            title="doc1", content="this is a document from c1",
            correspondent=c1, checksum="A",
        )
        doc2 = Document.objects.create(
            title="doc2", content="this is a document from noone", checksum="B",
        )
        clf = DocumentClassifier()
        clf.train()
        assert clf.predict_correspondent(doc1.content) == c1.pk
>       assert clf.predict_correspondent(doc2.content) is None
E       AssertionError: assert array([1]) is None
E        +  where array([1]) = predict_correspondent('this is a document from noone')
E        +    where predict_correspondent = <documents.classifier.DocumentClassifier object at 0x7b0936c71760>.predict_correspondent
E        +    and   'this is a document from noone' = <Document: 2026-07-08 doc2>.content

/tmp/test_q1_pytest_flake.py:29: AssertionError
-------------------------------------------------- Captured log call ---------------------------------------------------
DEBUG    paperless.classifier:classifier.py:123 Gathering data from database...
DEBUG    paperless.classifier:classifier.py:178 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
DEBUG    paperless.classifier:classifier.py:193 Vectorizing data...
DEBUG    paperless.classifier:classifier.py:223 There are no tags. Not training tags classifier.
DEBUG    paperless.classifier:classifier.py:226 Training correspondent classifier...
DEBUG    paperless.classifier:classifier.py:242 There are no document types. Not training document type classifier.
=============================================== short test summary info ================================================
FAILED ../../tmp/test_q1_pytest_flake.py::test_manydocs_repeat[705] - AssertionError: assert array([1]) is None
1 failed, 999 passed in 29.95s
```

The failing case is `test_manydocs_repeat[705]` — a **`doc2` flip** (`predict_correspondent('this is a document from noone')` returned `array([1])` instead of `None`). The **specific index is not reproducible**: it was `[705]` in this run and lands on a different index (or on no case at all) every run — consistent with the ≈ 1e-4 per-case rate measured in §2.4.3, not with any fixed schedule position.

**INFERRED (not code-grounded):** parallel batches surface the flip somewhat more readily than a single contiguous `-n0` stream because each `pytest-xdist` worker is a fresh OS process whose unseeded NumPy RNG is seeded independently; this changes only **how often** the rare crossing is sampled, not **whether** the models differ — the `distinct == K` counts (§2.4.1) prove the non-determinism exists identically in single-process mode.

> **Scope note (per constraints):** this section **explains and evidences** the non-determinism; it does **not** fix it. No `random_state` was added, no ordering was pinned, and `pytest-xdist` was toggled only to _diagnose_ (never committed). All spies live under `/tmp` (outside the repo) and are removed before finishing.

---

## 3. Q2 — Automatic correspondent matching: training-document count, timing, threshold

### 3.1 LEAD (negative result)

**There is no probability confidence threshold for the ML correspondent path.** `predict_correspondent()` calls `MLPClassifier.predict()` — which returns an **arg-max class label**, not a probability — and accepts it **only when it is `!= -1`**:

```python
# src/documents/classifier.py:L251-L260 (predict_correspondent)
X = self.data_vectorizer.transform([preprocess_content(content)])
correspondent_id = self.correspondent_classifier.predict(X)   # L254 (arg-max label)
if correspondent_id != -1:                                    # L255
    return correspondent_id                                   # L256
else:
    return None                                               # L258
```

`match_correspondents()` then accepts a candidate when its pk equals that predicted id — `matches(o, document) or o.pk == pred_id` [`src/documents/matching.py:L21-L31`, accept at `L30`]. The **only** numeric threshold anywhere in matching is the **fuzzy** comparison `fuzz.partial_ratio(match, text) >= 90` [`src/documents/matching.py:L135`], which lives in the **rule-based `MATCH_FUZZY`** path [`matching.py:L127`] and is **unrelated** to ML correspondent prediction. So the answer to "what confidence threshold is used?" is: **none for the ML path; the label `-1` sentinel (meaning 'no correspondent') is the only accept/reject criterion, and the fuzzy `>= 90` belongs to a different, non-ML mechanism.**

### 3.2 Training-document count (created vs. trained; inbox exclusion)

`DocumentClassifier.train()` gathers its corpus from `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`src/documents/classifier.py:L125-L127`] — i.e. it trains on **all** documents in the current DB snapshot **except** those carrying an inbox tag. The subsequent log line `"<N> documents, <t> tag(s), <c> correspondent(s), <d> document type(s)."` [`classifier.py:L178-L185`] reports the **trained** count `len(data)`.

A `/tmp` spy (`/tmp/test_blitzy_q2.py`, a `DirectoriesMixin`/`TestCase`) reproduces each canonical scenario with the **real** `generate_test_data` structure and prints the **created** count, the **trained** (post-exclusion) count, and the classifier's own log line (the `paperless.classifier` logger is routed to stdout at DEBUG, so `"N documents, T tag(s), C correspondent(s), D document type(s)."` [`classifier.py:L178`] appears verbatim). Exact command and its **complete** output — **run 1**:

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q2.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 3 items

../../tmp/test_blitzy_q2.py::TestQ2::test_A_inbox_exclusion Creating test database for alias 'default'...
=== Q2 INBOX EXCLUSION (real generate_test_data, test_classifier.py:L25) ===
Document.objects.count() [CREATED] = 3
Document.objects.exclude(tags__is_inbox_tag=True).count() [TRAINED] = 2
doc_inbox pk=3 carries inbox tag t2(is_inbox_tag=True) -> excluded from training = True
   LOG[t=1783495973.1449] paperless.classifier: Gathering data from database...
   LOG[t=1783495973.1485] paperless.classifier: 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
   LOG[t=1783495973.5908] paperless.classifier: Vectorizing data...
   LOG[t=1783495973.5915] paperless.classifier: Training tags classifier...
   LOG[t=1783495973.6203] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495973.6372] paperless.classifier: Training document type classifier...
PASSED
../../tmp/test_blitzy_q2.py::TestQ2::test_B_manydocs === Q2 MANY-DOCS (test_one_correspondent_predict_manydocs, test_classifier.py:L206) ===
Document.objects.count()=2 (doc1->c1, doc2->no correspondent)
   LOG[t=1783495973.6605] paperless.classifier: Gathering data from database...
   LOG[t=1783495973.6636] paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG[t=1783495973.6637] paperless.classifier: Vectorizing data...
   LOG[t=1783495973.6641] paperless.classifier: There are no tags. Not training tags classifier.
   LOG[t=1783495973.6641] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495973.6795] paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=[1] (expect c1.pk=1)
RESULT predict_correspondent(doc2)=None (expect None)
RESULT match_correspondents(doc2)->pks=[] (expect [])
PASSED
../../tmp/test_blitzy_q2.py::TestQ2::test_C_single_doc_timing === Q2 SINGLE-DOC (test_one_correspondent_predict, test_classifier.py:L191) ===
[t=1783495973.6843] CREATE correspondent c1 pk=1 (MATCH_AUTO)
[t=1783495973.6846] CREATE doc1 pk=1 ; Document.objects.count()=1
[t=1783495973.6849] CALL clf.train()
   LOG[t=1783495973.6849] paperless.classifier: Gathering data from database...
   LOG[t=1783495973.6869] paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG[t=1783495973.6870] paperless.classifier: Vectorizing data...
   LOG[t=1783495973.6873] paperless.classifier: There are no tags. Not training tags classifier.
   LOG[t=1783495973.6873] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495973.7046] paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=[1]  c1.pk=1  match_correspondents(doc1)->pks=[1]
PASSED
Destroying test database for alias 'default'...

============================== 3 passed in 2.18s ===============================
```

**Run 2 (F4 stability — same command, identical counts):** re-running the identical spy produced the **same** created/trained counts and the same `"N documents, …"` log lines (predictions may vary run-to-run per §2.4, but the _counts_ do not):

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q2.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 3 items

../../tmp/test_blitzy_q2.py::TestQ2::test_A_inbox_exclusion Creating test database for alias 'default'...
=== Q2 INBOX EXCLUSION (real generate_test_data, test_classifier.py:L25) ===
Document.objects.count() [CREATED] = 3
Document.objects.exclude(tags__is_inbox_tag=True).count() [TRAINED] = 2
doc_inbox pk=3 carries inbox tag t2(is_inbox_tag=True) -> excluded from training = True
   LOG[t=1783495996.9600] paperless.classifier: Gathering data from database...
   LOG[t=1783495996.9635] paperless.classifier: 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
   LOG[t=1783495997.4076] paperless.classifier: Vectorizing data...
   LOG[t=1783495997.4083] paperless.classifier: Training tags classifier...
   LOG[t=1783495997.4399] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495997.4592] paperless.classifier: Training document type classifier...
PASSED
../../tmp/test_blitzy_q2.py::TestQ2::test_B_manydocs === Q2 MANY-DOCS (test_one_correspondent_predict_manydocs, test_classifier.py:L206) ===
Document.objects.count()=2 (doc1->c1, doc2->no correspondent)
   LOG[t=1783495997.5149] paperless.classifier: Gathering data from database...
   LOG[t=1783495997.5179] paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG[t=1783495997.5180] paperless.classifier: Vectorizing data...
   LOG[t=1783495997.5185] paperless.classifier: There are no tags. Not training tags classifier.
   LOG[t=1783495997.5185] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495997.5402] paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=[1] (expect c1.pk=1)
RESULT predict_correspondent(doc2)=None (expect None)
RESULT match_correspondents(doc2)->pks=[] (expect [])
PASSED
../../tmp/test_blitzy_q2.py::TestQ2::test_C_single_doc_timing === Q2 SINGLE-DOC (test_one_correspondent_predict, test_classifier.py:L191) ===
[t=1783495997.5455] CREATE correspondent c1 pk=1 (MATCH_AUTO)
[t=1783495997.5458] CREATE doc1 pk=1 ; Document.objects.count()=1
[t=1783495997.5461] CALL clf.train()
   LOG[t=1783495997.5461] paperless.classifier: Gathering data from database...
   LOG[t=1783495997.5483] paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG[t=1783495997.5484] paperless.classifier: Vectorizing data...
   LOG[t=1783495997.5487] paperless.classifier: There are no tags. Not training tags classifier.
   LOG[t=1783495997.5488] paperless.classifier: Training correspondent classifier...
   LOG[t=1783495997.5677] paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=[1]  c1.pk=1  match_correspondents(doc1)->pks=[1]
PASSED
Destroying test database for alias 'default'...

============================== 3 passed in 2.19s ===============================
```

| Scenario (mirrors)                                                    | `Document`s **created** | `Document`s **trained** | Evidence                                                                                     |
| --------------------------------------------------------------------- | :---------------------: | :---------------------: | -------------------------------------------------------------------------------------------- |
| `test_one_correspondent_predict` [`test_classifier.py:L191`]          |          **1**          |          **1**          | LOG `"1 documents, 0 tag(s), 1 correspondent(s), …"` (see §3.3)                              |
| `test_one_correspondent_predict_manydocs` [`test_classifier.py:L206`] |          **2**          |          **2**          | LOG `"2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s)."`; `doc2 → None`         |
| `generate_test_data` [`test_classifier.py:L25`]                       |          **3**          |          **2**          | `doc_inbox` (pk=3, inbox tag `t2`, `is_inbox_tag=True`) **excluded**; LOG `"2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s)."` |

The **created** vs. **trained** distinction is exactly the inbox-tag exclusion: `generate_test_data` creates 3 documents but only **2** reach the classifier. These counts are **deterministic**: **run 1** and **run 2** above show byte-identical counts (`CREATED = 3`, `TRAINED = 2`; manydocs `count() = 2`; single-doc `count() = 1`) and identical `"N documents, …"` log lines — unlike the predictions in §2.4, the _counts_ never vary.

### 3.3 Training timing relative to inserts

Training runs **after** the document inserts, against the DB snapshot. The `test_C_single_doc_timing` portion of the **run 1** output in §3.2 (same invocation, shown complete above) stamps a wall-clock marker `[t=…]` at each `Document.objects.create(...)` and at the `train()` call, and shows the classifier's `"Gathering data from database…"` [`classifier.py:L123`] firing **after** the inserts. The relevant lines, quoted verbatim from that run-1 output:

```
[t=1783495973.6843] CREATE correspondent c1 pk=1 (MATCH_AUTO)
[t=1783495973.6846] CREATE doc1 pk=1 ; Document.objects.count()=1
[t=1783495973.6849] CALL clf.train()
   LOG[t=1783495973.6849] paperless.classifier: Gathering data from database...
   LOG[t=1783495973.6869] paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
```

Ordering (by timestamp): **create `c1`** (…973.6843) → **create `doc1`** (…973.6846, `count()=1`) → **call `train()`** (…973.6849) → **`"Gathering data from database…"`** (…973.6849) → **`"1 documents, …"`** (…973.6869). Training therefore reads the _already-inserted_ rows — the create-then-train sequence every classifier test uses.

### 3.4 Acceptance mechanism & consume-time path

From the spy output above: for a matching correspondent, `predict_correspondent(doc1) = [1]` (the arg-max label as a 1-element array; `== c1.pk = 1`) and `match_correspondents(doc1) → pks = [1]`; for a non-matching document, `predict_correspondent(doc2) = None` and `match_correspondents(doc2) → pks = []`. Acceptance is the arg-max `!= -1` rule of §3.1 — no probability gate.

The **consume-time** auto-assignment path is `set_correspondent(...)` [`src/documents/signals/handlers.py:L35`] → `matching.match_correspondents(document, classifier)` [`handlers.py:L50`]. The two consume-time signal tests exercise this wiring. Exact command and its **complete** output:

```
$ cd /app/src && python -m pytest documents/tests/test_matchables.py -k correspondent -n0 --no-cov -p no:cacheprovider -p no:warnings -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 15 items / 13 deselected / 2 selected

documents/tests/test_matchables.py ..                                    [100%]

======================= 2 passed, 13 deselected in 1.69s =======================
```

- `test_correspondent_applied` [`test_matchables.py:L425`] uses `Correspondent(match="keyword", MATCH_ANY)` and fires `document_consumption_finished` [`L431`].
- `test_correspondent_not_applied` [`test_matchables.py:L437`] uses a non-matching rule and asserts `document.correspondent is None`.

> **Important (observed, to avoid misreading):** both of these tests drive the consume-time signal via the **rule-based** (`MATCH_ANY`) path, with the ML classifier not being the deciding factor — they validate the **signal wiring**, **not** the ML "threshold." This is called out explicitly because their names could otherwise suggest they test the ML acceptance criterion.

### 3.5 Sibling variants covered

- **Single-document** correspondent training (`test_one_correspondent_predict`, created=1/trained=1) **vs. multi-document** (`test_one_correspondent_predict_manydocs`, created=2/trained=2) — both run. Exact command and its **complete** output:

  ```
  $ cd /app/src && python -m pytest documents/tests/test_classifier.py -k one_correspondent -n0 --no-cov -p no:cacheprovider -p no:warnings -v
  ============================= test session starts ==============================
  platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
  django: version: 4.0.4, settings: paperless.settings (from ini)
  rootdir: /app/src
  configfile: setup.cfg
  plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
  collected 23 items / 21 deselected / 2 selected

  documents/tests/test_classifier.py ..                                    [100%]

  ======================= 2 passed, 21 deselected in 1.95s =======================
  ```

- **Inbox-tag exclusion** (`generate_test_data`'s `doc_inbox`) — evidenced above: **3 created → 2 trained**, the excluded document being the inbox-tagged one [`classifier.py:L125-L127`].

---

## 4. Q3 — Document with no extractable text: OCR subprocess & output MIME

### 4.1 Direct answer

- **OCR subprocess invoked.** The single OCR entry point is **`ocrmypdf.ocr(**args)`** [`src/paperless_tesseract/parsers.py:L261`] inside `RasterisedDocumentParser.parse()` [`parsers.py:L230-L327`]. `ocrmypdf` in turn drives **Tesseract** (the OCR engine) and **Ghostscript** (`gs`, PDF rasterization / PDF-A production) as its **child subprocesses** — **verified at runtime** below (not asserted from documentation).
- **MIME assigned to the output — both facets:**

  1. The OCR **archive** produced by `ocrmypdf` is always a PDF ⇒ **`application/pdf`** (observed via `magic.from_file(archive)`).
  2. The stored **`Document.mime_type`** is the **original** type detected on the _input_ by `magic.from_file(self.path, mime=True)` [`src/documents/consumer.py:L219`], threaded through `_store(self, text, date, mime_type)` [`consumer.py:L379`] and persisted as `mime_type=mime_type` [`consumer.py:L401`]. The parser only accepts its declared input MIME types [`src/paperless_tesseract/signals.py:L11-L19`]: `application/pdf`→`.pdf`, `image/jpeg`→`.jpg`, `image/png`→`.png`, `image/tiff`→`.tif`, `image/gif`→`.gif`, `image/bmp`→`.bmp`.

  So for a **no-text PDF** input the stored `Document.mime_type` is `application/pdf` (coincidentally equal to the archive), while for a **no-text image** input (e.g. `image/png`) the stored `Document.mime_type` stays **`image/png`** even though the OCR archive is **`application/pdf`**. This contrast is shown directly below.

### 4.2 Runtime proof that `ocrmypdf.ocr` spawns Tesseract + Ghostscript

A `/tmp` spy (`/tmp/test_blitzy_q3_ocr.py`) sets `PATH=/tmp/binshim:$PATH` with tiny wrapper scripts for `tesseract` and `gs` that **log their argv to `/tmp/ocr_subprocess.log`, then `exec` the real `/usr/bin` binary**, and parses three samples through the real `RasterisedDocumentParser.parse()`: CASE 1 scanned-image PDF (`multi-page-images.pdf`, default `OCR_MODE=skip`), CASE 2 truly-empty image (`no-text-alpha.png`, default skip ⇒ empty ⇒ force fallback), CASE 3 fillable-form PDF (`with-form.pdf`, `OCR_MODE=redo` ⇒ ocrmypdf error ⇒ force fallback). Exact command and its **complete** output (the `[ERROR]`/`[WARNING]` lines are `ocrmypdf`/`tesseract`'s own real stderr, shown verbatim):

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q3_ocr.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 1 item

../../tmp/test_blitzy_q3_ocr.py::test_q3_all_cases ### CASE 1: DEFAULT OCR_MODE=skip, scanned-image PDF (no text layer)
LOG[paperless.parsing.tesseract/DEBUG]: Extracted text from PDF file /app/src/paperless_tesseract/tests/samples/multi-page-images.pdf
LOG[paperless.parsing.tesseract/DEBUG]: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf', 'output_file': '/tmp/paperless/paperless-3tjo09ox/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-3tjo09ox/sidecar.txt'}
[2026-07-08 07:40:16,941] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:40:16,949] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:40:16,954] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
LOG[paperless.parsing.tesseract/DEBUG]: Using text from sidecar file
========== PARSE: multi-page-images.pdf  (mime=application/pdf) ==========
magic.from_file(input, mime=True)   = 'application/pdf'
--- result state ---
archive_path = '/tmp/paperless/paperless-3tjo09ox/archive.pdf'
magic.from_file(archive, mime=True) = 'application/pdf'
get_text() length = 116
get_text() repr   = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'

### CASE 2: DEFAULT OCR_MODE=skip, truly-empty image (no text at all)
LOG[paperless.parsing.tesseract/WARNING]: Error while getting DPI from image /app/src/paperless_tesseract/tests/samples/no-text-alpha.png: 'dpi'
LOG[paperless.parsing.tesseract/DEBUG]: Estimated DPI 35 based on image width 297
LOG[paperless.parsing.tesseract/INFO]: Removing alpha layer from /app/src/paperless_tesseract/tests/samples/no-text-alpha.png for compatibility with img2pdf
LOG[paperless.parsing.tesseract/DEBUG]: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/no-text-alpha.png', 'output_file': '/tmp/paperless/paperless-z10i9ik8/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-z10i9ik8/sidecar.txt', 'image_dpi': 35}
[2026-07-08 07:40:18,871] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-08 07:40:18,872] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-08 07:40:18,872] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:40:19,339] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
LOG[paperless.parsing.tesseract/DEBUG]: Using text from sidecar file
LOG[paperless.parsing.tesseract/WARNING]: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG[paperless.parsing.tesseract/WARNING]: Error while getting DPI from image /app/src/paperless_tesseract/tests/samples/no-text-alpha.png: 'dpi'
LOG[paperless.parsing.tesseract/DEBUG]: Estimated DPI 35 based on image width 297
LOG[paperless.parsing.tesseract/DEBUG]: Fallback: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/no-text-alpha.png', 'output_file': '/tmp/paperless/paperless-z10i9ik8/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-z10i9ik8/sidecar-fallback.txt', 'image_dpi': 35}
[2026-07-08 07:40:19,809] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
[2026-07-08 07:40:19,809] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning. Invalid resolution 35 dpi. Using 70 instead.
[2026-07-08 07:40:19,809] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:40:20,254] [WARNING] [ocrmypdf._exec.tesseract] [tesseract] Warning: Invalid resolution 35 dpi. Using 70 instead.
LOG[paperless.parsing.tesseract/DEBUG]: Using text from sidecar file
LOG[paperless.parsing.tesseract/WARNING]: No text was found in /app/src/paperless_tesseract/tests/samples/no-text-alpha.png, the content will be empty.
========== PARSE: no-text-alpha.png  (mime=image/png) ==========
magic.from_file(input, mime=True)   = 'image/png'
--- result state ---
archive_path = '/tmp/paperless/paperless-z10i9ik8/archive.pdf'
magic.from_file(archive, mime=True) = 'application/pdf'
get_text() length = 0
get_text() repr   = ''

### CASE 3: OCR_MODE=redo on with-form.pdf -> ocrmypdf error -> force fallback
LOG[paperless.parsing.tesseract/DEBUG]: Extracted text from PDF file /app/src/paperless_tesseract/tests/samples/with-form.pdf
LOG[paperless.parsing.tesseract/DEBUG]: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/with-form.pdf', 'output_file': '/tmp/paperless/paperless-af3k68e2/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'redo_ocr': True, 'clean': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-af3k68e2/sidecar.txt'}
[2026-07-08 07:40:20,621] [ERROR] [ocrmypdf._pipeline] This PDF has a user fillable form. --redo-ocr is not currently possible on such files.
LOG[paperless.parsing.tesseract/WARNING]: Encountered an error while running OCR: . Attempting force OCR to get the text.
LOG[paperless.parsing.tesseract/DEBUG]: Fallback: Calling OCRmyPDF with args: {'input_file': '/app/src/paperless_tesseract/tests/samples/with-form.pdf', 'output_file': '/tmp/paperless/paperless-af3k68e2/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-af3k68e2/sidecar-fallback.txt'}
[2026-07-08 07:40:20,731] [WARNING] [ocrmypdf._pipeline] This PDF has a fillable form. Chances are it is a pure digital document that does not need OCR.
LOG[paperless.parsing.tesseract/DEBUG]: Using text from sidecar file
========== PARSE: with-form.pdf  (mime=application/pdf) ==========
magic.from_file(input, mime=True)   = 'application/pdf'
--- result state ---
archive_path = None
magic.from_file(archive, mime=True) = None
get_text() length = 68
get_text() repr   = 'Please enter your name in here:\n\nThis is a PDF document with a form.'

========== SUBPROCESS SHIM LOG  (/tmp/ocr_subprocess.log) ==========
(each line = one child process ocrmypdf actually exec'd via PATH)
SUBPROCESS tesseract --list-langs
SUBPROCESS tesseract --version
SUBPROCESS gs --version
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=1 -dLastPage=1 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=2 -dLastPage=2 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=3 -dLastPage=3 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.waawojm7/000001_rasterize_preview.jpg stdout
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.waawojm7/000002_rasterize_preview.jpg stdout
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.waawojm7/000003_rasterize_preview.jpg stdout
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=pnggray -dFirstPage=1 -dLastPage=1 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=pnggray -dFirstPage=2 -dLastPage=2 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=pnggray -dFirstPage=3 -dLastPage=3 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.waawojm7/origin.pdf
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000003_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000002_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000002_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.waawojm7/000003_rasterize.png stdout
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.waawojm7/000002_ocr.png /tmp/ocrmypdf.io.waawojm7/000002_ocr_tess pdf txt
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.waawojm7/000001_ocr.png /tmp/ocrmypdf.io.waawojm7/000001_ocr_tess pdf txt
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.waawojm7/000003_ocr.png /tmp/ocrmypdf.io.waawojm7/000003_ocr_tess pdf txt
SUBPROCESS gs -dBATCH -dNOPAUSE -dSAFER -dCompatibilityLevel=1.6 -sDEVICE=pdfwrite -dAutoRotatePages=/None -sColorConversionStrategy=LeaveColorUnchanged -dAutoFilterColorImages=true -dAutoFilterGrayImages=true -dJPEGQ=95 -dPDFA=2 -dPDFACompatibilityPolicy=1 -o - -sstdout=%stderr /tmp/ocrmypdf.io.waawojm7/fix_docinfo.pdf /tmp/ocrmypdf.io.waawojm7/pdfa.ps
SUBPROCESS tesseract --list-langs
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=1 -dLastPage=1 -r35.000003x35.000003 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.cwwb4ow5/origin.pdf
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.cwwb4ow5/000001_rasterize_preview.jpg stdout
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=png16m -dFirstPage=1 -dLastPage=1 -r35.000003x35.000003 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.cwwb4ow5/origin.pdf
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.cwwb4ow5/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.cwwb4ow5/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.cwwb4ow5/000001_ocr.png /tmp/ocrmypdf.io.cwwb4ow5/000001_ocr_tess pdf txt
SUBPROCESS gs -dBATCH -dNOPAUSE -dSAFER -dCompatibilityLevel=1.6 -sDEVICE=pdfwrite -dAutoRotatePages=/None -sColorConversionStrategy=LeaveColorUnchanged -dAutoFilterColorImages=true -dAutoFilterGrayImages=true -dJPEGQ=95 -dPDFA=2 -dPDFACompatibilityPolicy=1 -o - -sstdout=%stderr /tmp/ocrmypdf.io.cwwb4ow5/fix_docinfo.pdf /tmp/ocrmypdf.io.cwwb4ow5/pdfa.ps
SUBPROCESS tesseract --list-langs
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=1 -dLastPage=1 -r35.000003x35.000003 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.5v_ogub5/origin.pdf
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.5v_ogub5/000001_rasterize_preview.jpg stdout
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=png16m -dFirstPage=1 -dLastPage=1 -r35.000003x35.000003 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.5v_ogub5/origin.pdf
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.5v_ogub5/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.5v_ogub5/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.5v_ogub5/000001_ocr.png /tmp/ocrmypdf.io.5v_ogub5/000001_ocr_tess pdf txt
SUBPROCESS gs -dBATCH -dNOPAUSE -dSAFER -dCompatibilityLevel=1.6 -sDEVICE=pdfwrite -dAutoRotatePages=/None -sColorConversionStrategy=LeaveColorUnchanged -dAutoFilterColorImages=true -dAutoFilterGrayImages=true -dJPEGQ=95 -dPDFA=2 -dPDFACompatibilityPolicy=1 -o - -sstdout=%stderr /tmp/ocrmypdf.io.5v_ogub5/fix_docinfo.pdf /tmp/ocrmypdf.io.5v_ogub5/pdfa.ps
SUBPROCESS tesseract --list-langs
SUBPROCESS tesseract --list-langs
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=1 -dLastPage=1 -r400.000000x400.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.g0yowqi4/origin.pdf
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.g0yowqi4/000001_rasterize_preview.jpg stdout
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=png16m -dFirstPage=1 -dLastPage=1 -r400.000000x400.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.g0yowqi4/origin.pdf
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.g0yowqi4/000001_ocr.png /tmp/ocrmypdf.io.g0yowqi4/000001_ocr_tess pdf txt
SUBPROCESS gs -dBATCH -dNOPAUSE -dSAFER -dCompatibilityLevel=1.6 -sDEVICE=pdfwrite -dAutoRotatePages=/None -sColorConversionStrategy=LeaveColorUnchanged -dAutoFilterColorImages=true -dAutoFilterGrayImages=true -dJPEGQ=95 -dPDFA=2 -dPDFACompatibilityPolicy=1 -o - -sstdout=%stderr /tmp/ocrmypdf.io.g0yowqi4/fix_docinfo.pdf /tmp/ocrmypdf.io.g0yowqi4/pdfa.ps
TALLY: tesseract invocations = 28 ; gs (ghostscript) invocations = 17
PASSED

============================== 1 passed in 10.62s ==============================
```

**Reading the log:** `gs` first rasterizes each PDF page (`-sDEVICE=jpeggray`/`pnggray`/`png16m`), `tesseract` does orientation detection (`-l osd --psm 0`) and OCR (`-l eng --psm 2`, then `-c textonly_pdf=1 … pdf txt` to emit the text/PDF layer), and finally `gs -sDEVICE=pdfwrite … -dPDFA=2 …` produces the PDF/A archive. Across the three parses the tally was **28 tesseract** and **17 gs** child processes — direct runtime evidence that `ocrmypdf.ocr` [`parsers.py:L261`] drives both binaries. Before each `ocrmypdf.ocr` call the parser logs the exact args [`parsers.py:L260`]; for the CASE-1 scanned PDF under the default `OCR_MODE=skip` the args are `{… 'output_type': 'pdfa', … 'skip_text': True, …}`.

(`output_type='pdfa'` is the canonical default `OCR_OUTPUT_TYPE="pdfa"` [`src/paperless/settings.py:L518`]; default `OCR_MODE="skip"` [`settings.py:L522`] ⇒ `skip_text=True`; `language='eng'` [`settings.py:L514`]. The CASE-2 fallback args carry `'force_ocr': True` and CASE-3's first attempt carries `'redo_ocr': True` — all visible verbatim in the complete output above.)

### 4.3 Both MIME facets — parser output vs. **persisted** `Document.mime_type`

The three `parse()` cases in §4.2 already show the **input** MIME (`magic.from_file(input)`) and the **archive** MIME (`magic.from_file(archive)`) for each sample: CASE 1 (`multi-page-images.pdf`) input `application/pdf` / archive `application/pdf`; CASE 2 (`no-text-alpha.png`) input `image/png` / archive `application/pdf`; CASE 3 (`with-form.pdf`) input `application/pdf` / archive `None` (the fallback archive is intentionally not retained). What those parser-level parses do **not** show is the value actually written to the DB. To observe the **persisted** `Document.mime_type`, a second `/tmp` spy (`/tmp/test_blitzy_q3_store.py`, a `DirectoriesMixin`/`TestCase`) drives the **real** `Consumer.try_consume_file()` — with the parser registry **not** mocked, so the real `RasterisedDocumentParser` runs — on a PDF and a PNG, and reads back `document.mime_type` after `_store()`/`save()`. Exact command and its **complete** output:

```
$ cd /app/src && PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q3_store.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 1 item

../../tmp/test_blitzy_q3_store.py::TestQ3StoredMime::test_stored_mime_pdf_and_png Creating test database for alias 'default'...

=== Q3 F5: REAL consumer/_store persisted Document.mime_type ===
[2026-07-08 07:40:49,804] [INFO] [paperless.consumer] Consuming q3_pdf.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpmj795ku9/paperless-k0q4n_7l/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-08 07:40:50,254] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 07:40:51,022] [INFO] [paperless.consumer] Document 2026-07-08 q3_pdf consumption finished
[PDF ] input magic.from_file        = 'application/pdf'
[PDF ] persisted Document.pk        = 1
[PDF ] persisted Document.mime_type = 'application/pdf'   (consumer.py:L401)
[PDF ] OCR archive magic.from_file  = 'application/pdf'
[2026-07-08 07:40:51,023] [INFO] [paperless.consumer] Consuming q3_png.png
[2026-07-08 07:40:51,291] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpmj795ku9/paperless-n6eb5c5a/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-08 07:40:51,964] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 07:40:52,371] [INFO] [paperless.consumer] Document 2026-07-08 q3_png consumption finished
[PNG ] input magic.from_file        = 'image/png'
[PNG ] persisted Document.pk        = 2
[PNG ] persisted Document.mime_type = 'image/png'   (consumer.py:L401)
[PNG ] OCR archive magic.from_file  = 'application/pdf'
=== END Q3 F5 ===
PASSED
Destroying test database for alias 'default'...

============================== 1 passed in 4.11s ===============================
```

(The `convert-im6.q16 … security policy` lines are the incidental ImageMagick PDF-thumbnail policy warning; the parser transparently falls back to Ghostscript for the thumbnail — unrelated to the stored MIME, shown here only because the output is complete and unedited.)

(Fixture-naming note: the consume-time input filenames in the log above — `q3_pdf.pdf` and `q3_png.png` — are copies of the committed fixtures `simple.pdf` and `simple.png`, which the table below labels by their canonical fixture names. The consume-time copy does not change MIME detection: `magic.from_file(…, mime=True)` returns `application/pdf` for `simple.pdf` and `image/png` for `simple.png` — exactly the values in the table.)

**All three MIME facets, side by side (input detection, OCR archive, and the value actually persisted to the DB):**

| Input sample                     | input `magic.from_file` [`consumer.py:L219`] | OCR archive `magic.from_file` | **persisted `Document.mime_type`** [`consumer.py:L401`] |
| -------------------------------- | :------------------------------------------: | :---------------------------: | :-----------------------------------------------------: |
| `simple.pdf` (PDF input)         |               `application/pdf`              |        `application/pdf`      |                    `application/pdf`                    |
| `simple.png` (PNG input)         |                **`image/png`**               |      **`application/pdf`**    |                     **`image/png`**                     |

This is the explicit contrast the question implies, now confirmed at the **stored-row** level: an **image** input keeps its original `image/png` as the persisted `Document.mime_type` (pk=2) even though its OCR **archive** is `application/pdf`; a **PDF** input persists `application/pdf` (pk=1), coincidentally equal to the archive type. The stored value is the **input** detection threaded through `_store(mime_type=mime_type)` [`consumer.py:L401`], never the archive type.

### 4.4 Fallback chain & empty-text last resort (edge paths, all observed)

The `parse()` method's OCR fallback is a three-stage chain, every stage of which was triggered live (CASE 2 / CASE 3 above):

1. **First attempt:** `ocrmypdf.ocr(**args)` [`parsers.py:L261`]; then if the extracted text is empty, `raise NoTextFoundException("No text was found in the original document")` [`parsers.py:L266-L267`] (class defined at `parsers.py:L14`).
2. **Force-OCR fallback:** caught by `except (NoTextFoundException, InputFileError) as e:` [`parsers.py:L276`], which logs `"…Attempting force OCR to get the text."` [`parsers.py:L280`], rebuilds args with `safe_fallback=True` (⇒ `force_ocr=True` via [`parsers.py:L155`]) [`parsers.py:L293`], logs `"Fallback: Calling OCRmyPDF with args: …"` [`parsers.py:L297`], and calls **`ocrmypdf.ocr(**args)` a second time** [`parsers.py:L298`]. Observed in CASE 2 (empty sidecar ⇒ `NoTextFoundException`) and CASE 3 (ocrmypdf `InputFileError`: _"This PDF has a user fillable form. --redo-ocr is not currently possible…"_).
3. **Empty-text last resort:** `if not self.text:` [`parsers.py:L318`] → if the original had embedded text it is used, else `self.text = ""` [`parsers.py:L327`] with the warning _"No text was found in …, the content will be empty."_ Observed in CASE 2: `get_text()` returns `''` (length 0).

Corresponding parser test suite:

```
$ cd /app/src && python -m pytest paperless_tesseract/tests/test_parser.py -k "notext or form" -n0 --no-cov -p no:cacheprovider -p no:warnings -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 35 items / 30 deselected / 5 selected

paperless_tesseract/tests/test_parser.py .....                           [100%]

====================== 5 passed, 30 deselected in 25.19s =======================
```

The 5 selected tests are `test_skip_noarchive_notext`, `test_with_form`, `test_with_form_error` (`OCR_MODE="redo"` ⇒ `archive_path is None`), `test_with_form_error_notext` (`OCR_MODE="redo"`), and `test_with_form_force` (`OCR_MODE="force"`) — the `redo`/`force` variants are precisely the ones that drive the force-OCR fallback branch.

### 4.5 MIME detection / assignment — where it happens

`get_parser_class_for_mime_type` dispatch runs off the same `magic.from_file` detection [`src/documents/parsers.py:L106`; consumer dispatch at `consumer.py:L223`]. The consumer logs `"Detected mime type: …"` [`consumer.py:L221`] and stores it unchanged. Summary of the state transition for a no-text **image**:

- **input** file → `magic.from_file` ⇒ `image/png` [`consumer.py:L219`]
- **OCR archive** produced by `ocrmypdf.ocr` ⇒ `application/pdf` [`parsers.py:L261`]
- **stored** `Document.mime_type` ⇒ `image/png` (the original) [`consumer.py:L401`]

---

## 5. Q4 — Barcode splitting: record count, trigger values, decision site, training-data impact

> **How every `[Q4-*]` block in this section was produced (exact command).** All labelled `[Q4-*]` blocks below are verbatim slices of one single spy run (**run 1**). The spy (`/tmp/test_blitzy_q4.py`, a `DirectoriesMixin, TestCase`) opens each fixture with `PIL.Image.open` and calls the real `documents.tasks` functions — `barcode_reader`, `scan_file_for_separating_barcodes`, `separate_pages`, and `consume_file` — with `@override_settings` used exactly as the project's own tests use it (`CONSUMER_BARCODE_STRING="CUSTOM BARCODE"` for the custom cases, `CONSUMER_ENABLE_BARCODES=True` for the consume case). For the consume case it wraps `documents.tasks.save_to_dir` in a recording side-effect that forwards to the real function, directing output at the test-isolated `CONSUMPTION_DIR`. The command:
>
> ```
> $ docker exec -u testuser pngx bash -c 'cd /app/src && rm -rf /tmp/paperless; \
>     PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q4.py \
>     --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v'
> ```
>
> The **complete, unedited run-1 output** is shown in §5.1 (consume branch) and sliced by topic in §5.2–§5.4; the **complete run-2 output** proving stability is shown in full in §5.3.

### 5.1 Direct answer (the key zero-rows result)

A single input file passed through the **barcode branch** of `consume_file()` [`src/documents/tasks.py:L184-L233`] yields **zero `Document` records created directly**. When barcodes are enabled (`settings.CONSUMER_ENABLE_BARCODES`, **default off** [`src/paperless/settings.py:L502-L504`]) and separators are found, the branch splits the PDF into **N segment files**, **saves those split PDFs back to the consumption directory** via `save_to_dir` (default `target_dir=settings.CONSUMPTION_DIR` [`tasks.py:L164-L168`]), **deletes the original**, and **returns early with the string `"File successfully split"`** [`tasks.py:L233`]. No `Document` row is created in that call. The `/tmp` spy proved this directly:

```
[Q4-consume_file barcode branch, CONSUMER_ENABLE_BARCODES=True]
   Document.objects.count() BEFORE = 0
   Document.objects.count() AFTER  = 0
   consume_file(...) returned      = 'File successfully split'
   original input still on disk?   = False
   save_to_dir called 2 time(s):
      basename='patch-code-t-middle_document_0.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpitvn0igj'
      basename='patch-code-t-middle_document_1.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpitvn0igj'
   CONSUMPTION_DIR before          = []
   CONSUMPTION_DIR after           = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
```

So you must distinguish **"split files produced"** (here **2** segments, re-queued to the consumption dir) from **"`Document` rows created"** (**0** in this call). (No broker/`OSError` warning appeared in this canonical run: under the test channel layer the `status_updates` send at [`tasks.py:L222-L232`] completes without raising, so the `except OSError` at [`tasks.py:L230-L232`] is not exercised here; in a production deployment with an unreachable message broker that branch would log a warning, but it never affects the `"File successfully split"` return.) This matches `test_consume_barcode_file` [`test_tasks.py:L396`], which asserts `tasks.consume_file(dst) == "File successfully split"` under `@override_settings(CONSUMER_ENABLE_BARCODES=True)`.

### 5.2 Decision site & trigger values

- The split decision is a **single line** — `if separator_barcode in current_barcodes:` [`src/documents/tasks.py:L108`] inside `scan_file_for_separating_barcodes()` [`tasks.py:L96-L110`], where `separator_barcode = str(settings.CONSUMER_BARCODE_STRING)` [`tasks.py:L102`].
- **Default trigger value** = **`"PATCHT"`**: `CONSUMER_BARCODE_STRING = os.getenv("PAPERLESS_CONSUMER_BARCODE_STRING", "PATCHT")` [`src/paperless/settings.py:L506`]. A custom value overrides it (tests use `@override_settings(CONSUMER_BARCODE_STRING="CUSTOM BARCODE")`).
- Barcodes are read by `barcode_reader(image)` via `pyzbar.decode(image)` [`tasks.py:L75-L93`, decode at `L82`], which decodes Code 39, Code 128 and QR alike; a page triggers a split only if a decoded value **equals** the separator string.
- Page splitting is done by `separate_pages(filepath, pages_to_split_on)` [`tasks.py:L113-L161`], emitting one PDF per segment (`{fname}_document_0.pdf` [`tasks.py:L133-L135`], then `{fname}_document_{count+1}.pdf` [`tasks.py:L154`]); the separator page itself is skipped (`range(page_number+1, next_page)` [`tasks.py:L149`]).

Observed defaults (spy):

```
[Q4-defaults] CONSUMER_BARCODE_STRING  = 'PATCHT'
[Q4-defaults] CONSUMER_ENABLE_BARCODES = False
```

### 5.3 Exact split counts per fixture (stable across ≥2 runs)

The actual test subset passes (the 25 barcode tests — 11 `barcode_reader*`, 10 `scan_file_for_separating*`, 2 `separate_pages*`, 1 `barcode_splitter`, 1 `consume_barcode`), and the spy independently printed the `scan_file_for_separating_barcodes(...)` return list for each fixture (default `PATCHT` separator). First the real project test subset, complete and unedited:

```
$ docker exec -u testuser pngx bash -c 'cd /app/src && python -m pytest \
    documents/tests/test_tasks.py \
    -k "scan_file_for_separating or separate_pages or barcode_splitter or consume_barcode or barcode_reader" \
    -n0 --no-cov -p no:cacheprovider -p no:warnings -v'
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 40 items / 15 deselected / 25 selected

documents/tests/test_tasks.py .........................                  [100%]

====================== 25 passed, 15 deselected in 5.22s =======================
```

Then the per-fixture `scan_file_for_separating_barcodes(...)` returns (verbatim slice of the run-1 spy output; the exact command that produced it is at the top of §5):

```
[Q4-scan default PATCHT] scan_file_for_separating_barcodes(...) ->
   patch-code-t.pdf                   -> [0]
   simple.pdf                         -> []
   patch-code-t-middle.pdf            -> [1]
   several-patcht-codes.pdf           -> [2, 5]
   patch-code-t-middle_reverse.pdf    -> [1]
   patch-code-t-qr.pdf                -> [0]
```

| Fixture                                         | `scan_…` return | Test anchor                                        |
| ----------------------------------------------- | :-------------: | -------------------------------------------------- |
| `patch-code-t.pdf`                              |      `[0]`      | `test_scan_file_for_separating_barcodes` [`L207`]  |
| `simple.pdf` (parent `samples/`, no barcode)    |      `[]`       | `test_scan_file_for_separating_barcodes2` [`L217`] |
| `patch-code-t-middle.pdf`                       |      `[1]`      | `test_scan_file_for_separating_barcodes3` [`L222`] |
| `several-patcht-codes.pdf` (multi-separator)    |    `[2, 5]`     | `test_scan_file_for_separating_barcodes4` [`L232`] |
| `patch-code-t-middle_reverse.pdf` (upside-down) |      `[1]`      | `…_upsidedown` [`L242`]                            |
| `patch-code-t-qr.pdf`                           |      `[0]`      | `…_qr_barcodes` [`L252`]                           |

`separate_pages(...)` then produces **`1 + len(separators)`** segment files (segment 0 = pages before the first separator, then one segment per separator; the separator pages themselves are dropped):

```
[Q4-separate_pages] split-file counts ->
   patch-code-t-middle.pdf      seps=[1]        -> 2 file(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
   several-patcht-codes.pdf     seps=[2, 5]     -> 3 file(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[2026-07-08 07:53:08,012] [WARNING] [paperless.tasks] No pages to split on!
   patch-code-t-middle.pdf      seps=[]         -> 0 file(s): []
```

- `[1]` → **2** files (matches `test_separate_pages` `len == 2` [`L305`] and `test_barcode_splitter` `_document_0.pdf`/`_document_1.pdf` [`L376`]).
- `[2, 5]` → **3** files.
- `[]` → **0** files, plus the warning `"No pages to split on!"` emitted at [`tasks.py:L127`] (matches `test_separate_pages_no_list` [`L315`], which asserts `["WARNING:paperless.tasks:No pages to split on!"]`).

**Stability across two runs (F4).** The entire Q4 spy was executed **twice** with the identical command shown at the top of §5. **Both complete, unedited session outputs are shown below — run 1 first, then run 2.** Every count — the `barcode_reader` decoded values, the `scan_…` returns (`[0]`/`[]`/`[1]`/`[2, 5]`), the `separate_pages` file counts (2/3/0), and the consume-branch `Document` counts (0 → 0) — is byte-for-byte identical between the two runs; only the test-isolated temp path (`/tmp/tmpitvn0igj` in run 1 vs `/tmp/tmpoa2w45j5` in run 2), the log timestamp, and the wall-clock total (`4.95s` vs `5.17s`) differ. This is **deterministic** — unlike the Q1 predictions.

**Run 1 (complete):**

```
$ docker exec -u testuser pngx bash -c 'cd /app/src && rm -rf /tmp/paperless; \
    PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q4.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v'
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 7 items

../../tmp/test_blitzy_q4.py::Q4::test_A_defaults Creating test database for alias 'default'...

[Q4-defaults] CONSUMER_BARCODE_STRING  = 'PATCHT'
[Q4-defaults] CONSUMER_ENABLE_BARCODES = False
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_B_barcode_reader
barcode_reader(image) decoded values [tasks.py:L75, pyzbar.decode L82]:
   Code 39                  barcode-39-PATCHT.png              -> ['PATCHT']
   Code 39 (.pbm patch-t)   patch-code-t.pbm                   -> ['PATCHT']
   Code 39 distorsion       barcode-39-PATCHT-distorsion.png   -> ['PATCHT']
   Code 39 distorsion2      barcode-39-PATCHT-distorsion2.png  -> ['PATCHT']
   Code 39 UNREADABLE       barcode-39-PATCHT-unreadable.png   -> []
   QR                       qr-code-PATCHT.png                 -> ['PATCHT']
   Code 128                 barcode-128-PATCHT.png             -> ['PATCHT']
   NO barcode (simple.png)  simple.png                         -> []
   Code 39 custom           barcode-39-custom.png              -> ['CUSTOM BARCODE']
   QR custom                barcode-qr-custom.png              -> ['CUSTOM BARCODE']
   Code 128 custom          barcode-128-custom.png             -> ['CUSTOM BARCODE']
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_C_scan_default
[Q4-scan default PATCHT] scan_file_for_separating_barcodes(...) ->
   patch-code-t.pdf                   -> [0]
   simple.pdf                         -> []
   patch-code-t-middle.pdf            -> [1]
   several-patcht-codes.pdf           -> [2, 5]
   patch-code-t-middle_reverse.pdf    -> [1]
   patch-code-t-qr.pdf                -> [0]
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_D_scan_custom
[Q4-scan CUSTOM BARCODE separator] ->
   barcode-39-custom.pdf    -> [0]
   barcode-qr-custom.pdf    -> [0]
   barcode-128-custom.pdf   -> [0]
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_E_scan_negative
[Q4-scan NEGATIVE cross-product] barcode-39-custom.pdf under default 'PATCHT' -> []
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_F_separate_pages
[Q4-separate_pages] split-file counts ->
   patch-code-t-middle.pdf      seps=[1]        -> 2 file(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
   several-patcht-codes.pdf     seps=[2, 5]     -> 3 file(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[2026-07-08 07:53:08,012] [WARNING] [paperless.tasks] No pages to split on!
   patch-code-t-middle.pdf      seps=[]         -> 0 file(s): []
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_G_consume_zero_rows
[Q4-consume_file barcode branch, CONSUMER_ENABLE_BARCODES=True]
   Document.objects.count() BEFORE = 0
   Document.objects.count() AFTER  = 0
   consume_file(...) returned      = 'File successfully split'
   original input still on disk?   = False
   save_to_dir called 2 time(s):
      basename='patch-code-t-middle_document_0.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpitvn0igj'
      basename='patch-code-t-middle_document_1.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpitvn0igj'
   CONSUMPTION_DIR before          = []
   CONSUMPTION_DIR after           = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
PASSEDDestroying test database for alias 'default'...


============================== 7 passed in 4.95s ===============================
```

**Run 2 (complete):**

```
$ docker exec -u testuser pngx bash -c 'cd /app/src && rm -rf /tmp/paperless; \
    PYTHONPATH=/app/src python -m pytest /tmp/test_blitzy_q4.py \
    --ds=paperless.settings -p no:cacheprovider -p no:warnings -s -n0 --no-cov -v'
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python
django: version: 4.0.4, settings: paperless.settings (from option)
rootdir: /tmp
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 7 items

../../tmp/test_blitzy_q4.py::Q4::test_A_defaults Creating test database for alias 'default'...

[Q4-defaults] CONSUMER_BARCODE_STRING  = 'PATCHT'
[Q4-defaults] CONSUMER_ENABLE_BARCODES = False
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_B_barcode_reader
barcode_reader(image) decoded values [tasks.py:L75, pyzbar.decode L82]:
   Code 39                  barcode-39-PATCHT.png              -> ['PATCHT']
   Code 39 (.pbm patch-t)   patch-code-t.pbm                   -> ['PATCHT']
   Code 39 distorsion       barcode-39-PATCHT-distorsion.png   -> ['PATCHT']
   Code 39 distorsion2      barcode-39-PATCHT-distorsion2.png  -> ['PATCHT']
   Code 39 UNREADABLE       barcode-39-PATCHT-unreadable.png   -> []
   QR                       qr-code-PATCHT.png                 -> ['PATCHT']
   Code 128                 barcode-128-PATCHT.png             -> ['PATCHT']
   NO barcode (simple.png)  simple.png                         -> []
   Code 39 custom           barcode-39-custom.png              -> ['CUSTOM BARCODE']
   QR custom                barcode-qr-custom.png              -> ['CUSTOM BARCODE']
   Code 128 custom          barcode-128-custom.png             -> ['CUSTOM BARCODE']
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_C_scan_default
[Q4-scan default PATCHT] scan_file_for_separating_barcodes(...) ->
   patch-code-t.pdf                   -> [0]
   simple.pdf                         -> []
   patch-code-t-middle.pdf            -> [1]
   several-patcht-codes.pdf           -> [2, 5]
   patch-code-t-middle_reverse.pdf    -> [1]
   patch-code-t-qr.pdf                -> [0]
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_D_scan_custom
[Q4-scan CUSTOM BARCODE separator] ->
   barcode-39-custom.pdf    -> [0]
   barcode-qr-custom.pdf    -> [0]
   barcode-128-custom.pdf   -> [0]
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_E_scan_negative
[Q4-scan NEGATIVE cross-product] barcode-39-custom.pdf under default 'PATCHT' -> []
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_F_separate_pages
[Q4-separate_pages] split-file counts ->
   patch-code-t-middle.pdf      seps=[1]        -> 2 file(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
   several-patcht-codes.pdf     seps=[2, 5]     -> 3 file(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[2026-07-08 07:53:31,056] [WARNING] [paperless.tasks] No pages to split on!
   patch-code-t-middle.pdf      seps=[]         -> 0 file(s): []
PASSED
../../tmp/test_blitzy_q4.py::Q4::test_G_consume_zero_rows
[Q4-consume_file barcode branch, CONSUMER_ENABLE_BARCODES=True]
   Document.objects.count() BEFORE = 0
   Document.objects.count() AFTER  = 0
   consume_file(...) returned      = 'File successfully split'
   original input still on disk?   = False
   save_to_dir called 2 time(s):
      basename='patch-code-t-middle_document_0.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpoa2w45j5'
      basename='patch-code-t-middle_document_1.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmpoa2w45j5'
   CONSUMPTION_DIR before          = []
   CONSUMPTION_DIR after           = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
PASSEDDestroying test database for alias 'default'...


============================== 7 passed in 5.17s ===============================
```

### 5.4 Named barcode variants — every one covered

`barcode_reader(image)` [`tasks.py:L75`] decoded values, captured live for each symbology:

```
barcode_reader(image) decoded values [tasks.py:L75, pyzbar.decode L82]:
   Code 39                  barcode-39-PATCHT.png              -> ['PATCHT']
   Code 39 (.pbm patch-t)   patch-code-t.pbm                   -> ['PATCHT']
   Code 39 distorsion       barcode-39-PATCHT-distorsion.png   -> ['PATCHT']
   Code 39 distorsion2      barcode-39-PATCHT-distorsion2.png  -> ['PATCHT']
   Code 39 UNREADABLE       barcode-39-PATCHT-unreadable.png   -> []
   QR                       qr-code-PATCHT.png                 -> ['PATCHT']
   Code 128                 barcode-128-PATCHT.png             -> ['PATCHT']
   NO barcode (simple.png)  simple.png                         -> []
   Code 39 custom           barcode-39-custom.png              -> ['CUSTOM BARCODE']
   QR custom                barcode-qr-custom.png              -> ['CUSTOM BARCODE']
   Code 128 custom          barcode-128-custom.png             -> ['CUSTOM BARCODE']
```

- **QR** — `qr-code-PATCHT.png` → `['PATCHT']`; page-level `patch-code-t-qr.pdf` → `[0]` (`test_scan_file_for_separating_qr_barcodes` [`L252`]).
- **Code 39** — `barcode-39-PATCHT.png` → `['PATCHT']`, and its `-distorsion` / `-distorsion2` variants also decode to `['PATCHT']`; the **`-unreadable`** variant yields **`[]`** (no decoded separator ⇒ no split).
- **Code 128** — `barcode-128-PATCHT.png` → `['PATCHT']`; custom `barcode-128-custom.pdf` → `[0]` under the custom separator (`test_scan_file_for_separating_custom_128_barcodes` [`L285`]).
- **Custom separator `"CUSTOM BARCODE"`** (cross-product with symbology, under `@override_settings(CONSUMER_BARCODE_STRING="CUSTOM BARCODE")`):

  ```
  [Q4-scan CUSTOM BARCODE separator] ->
     barcode-39-custom.pdf    -> [0]
     barcode-qr-custom.pdf    -> [0]
     barcode-128-custom.pdf   -> [0]
  ```

  and the decoded custom value string is exactly `"CUSTOM BARCODE"` (reader table above).

- **Negative cross-product** — a custom-encoded barcode under the **default `PATCHT`** separator does **not** match:

  ```
  [Q4-scan NEGATIVE cross-product] barcode-39-custom.pdf under default 'PATCHT' -> []
  ```

  (matches `test_scan_file_for_separating_wrong_qr_barcodes` [`L295`]).

- **Unreadable / no-barcode** — `barcode-39-PATCHT-unreadable.png` → `[]` and `simple.png` → `[]` (empty barcode list ⇒ no separator ⇒ no split); matches `test_barcode_reader_unreadable` [`L140`] and `test_barcode_reader_no_barcode` [`L172`].

### 5.5 Training-data impact (link Q4 → Q1/Q2)

**Observed:** the barcode branch creates **0** `Document` rows and instead re-queues the **N** split PDFs into the consumption directory (`CONSUMPTION_DIR before = []` → `after = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']`, original deleted, return `"File successfully split"`) — all shown in §5.1.

**INFERRED (downstream, not directly observed in the single call):** because those split files land back in the consumption directory, they are later **independently re-consumed** through the **non-barcode** branch of `consume_file` into **new `Document` rows**. Only then do they enter the corpus that `train()` reads via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`src/documents/classifier.py:L125-L127`]. Consequently, a single barcode input does **not immediately** change the effective training data (0 rows on the split call); it changes it **indirectly and later**, once the re-queued segments are consumed. This is labelled INFERRED because the split call itself was observed to create no rows and to return early; the subsequent re-consumption is the documented design of re-queuing to `CONSUMPTION_DIR` [`tasks.py:L164-L168`] rather than something exercised within the same call.

---

## 6. Coverage pass

Every named item across the four questions, with its concrete value, `file:line`, observed evidence, sibling variants, and causal reason:

| #   | Named item                                                               | Value / result (observed)                                                            | `file:line`                                         | Evidence (§)                                                |
| --- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ | --------------------------------------------------- | ----------------------------------------------------------- |
| 1   | `load_classifier()` / `MODEL_FILE`                                       | returns `None` if absent, else deserializes; default `…/classification_model.pickle` | `classifier.py:L30-L57`; `settings.py:L74`          | §2.1, §2.2                                                  |
| 2   | `data_hash` reuse guard                                                  | `train()` returns `False` on unchanged data (no retrain, no save)                    | `classifier.py:L163-L164`                           | §2.2 (hash `230b98c1…` stable 1→2; `faa493d0…` on mutation) |
| 3   | `save()` / `load()` round-trip                                           | `testSaveClassifier` PASS                                                            | `classifier.py:L96`, `L76`                          | §2.2                                                        |
| 4   | `MLPClassifier` missing `random_state` (3 sites)                         | tags, correspondent, doc-type — none seeded                                          | `classifier.py:L219`, `L227`, `L238`                | §2.4 (root cause)                                           |
| 5   | `DirectoriesMixin` / tempdir isolation                                   | distinct `MODEL_FILE` per test                                                       | `utils.py:L14-L50`, `L45`, `L72-L83`                | §2.3 (`/tmp/tmpxiy5xzu3` vs `/tmp/tmpjdtrqnbt`)             |
| 6   | committed `model.pickle` fixture                                         | 156,607 bytes; `test_load_and_classify` PASS                                         | `test_classifier.py:L183`                           | §2.2                                                        |
| 7   | `pytest-xdist` parallelism                                               | default `--numprocesses auto` (128 CPUs)                                             | `setup.cfg:L10`                                     | §2.3                                                        |
| 8   | `predict_correspondent` arg-max & `!= -1`                                | accept label only when `!= -1`, else `None`                                          | `classifier.py:L251-L260` (`L255-L256`)             | §3.1, §3.4                                                  |
| 9   | **NO probability threshold** (negative)                                  | ML path has no confidence threshold                                                  | `classifier.py:L255-L256`                           | §3.1                                                        |
| 10  | fuzzy `>= 90` (the only numeric threshold)                               | belongs to rule-based `MATCH_FUZZY`, not ML                                          | `matching.py:L135` (`L127`)                         | §3.1                                                        |
| 11  | `match_correspondents`                                                   | accepts when `o.pk == pred_id`                                                       | `matching.py:L21-L31` (`L30`)                       | §3.1, §3.4                                                  |
| 12  | `set_correspondent` consume-time path                                    | → `matching.match_correspondents(document, classifier)`                              | `handlers.py:L35`, `L50`                            | §3.4                                                        |
| 13  | training count: created vs. trained + inbox exclusion                    | 1/1, 2/2, **3 created → 2 trained**                                                  | `classifier.py:L125-L127`, `L178-L185`              | §3.2                                                        |
| 14  | create-then-train timing                                                 | `"Gathering data…"` fires after inserts                                              | `classifier.py:L123`                                | §3.3                                                        |
| 15  | single- vs. multi-document training                                      | both PASS (1 doc; 2 docs, `doc2→None`)                                               | `test_classifier.py:L191`, `L206`                   | §3.2, §3.5                                                  |
| 16  | `ocrmypdf.ocr` (Tesseract + Ghostscript)                                 | verified at runtime: 28 `tesseract`, 17 `gs` child procs                             | `parsers.py:L261`                                   | §4.2                                                        |
| 17  | `NoTextFoundException` → force-OCR fallback                              | 2nd `ocrmypdf.ocr` with `force_ocr=True`                                             | `parsers.py:L266-L267`, `L276`, `L293`, `L297-L298` | §4.3, §4.4                                                  |
| 18  | empty-text last resort                                                   | `self.text = ""`; observed `get_text()=''`                                           | `parsers.py:L318`, `L327`                           | §4.3, §4.4                                                  |
| 19  | `magic.from_file` MIME detection                                         | input type detected & stored unchanged                                               | `consumer.py:L219`, `L401`                          | §4.1, §4.5                                                  |
| 20  | declared parser MIME map                                                 | pdf/jpeg/png/tiff/gif/bmp → extensions                                               | `signals.py:L11-L19`                                | §4.1                                                        |
| 21  | both MIME facets (archive vs. stored)                                    | archive `application/pdf`; stored = original (`image/png` for PNG)                   | `parsers.py:L261`; `consumer.py:L219`,`L401`        | §4.1, §4.3                                                  |
| 22  | `CONSUMER_BARCODE_STRING` default `"PATCHT"` & custom `"CUSTOM BARCODE"` | observed `'PATCHT'`; custom decodes/triggers                                         | `settings.py:L506`                                  | §5.2, §5.4                                                  |
| 23  | `CONSUMER_ENABLE_BARCODES` default off                                   | observed `False`                                                                     | `settings.py:L502-L504`                             | §5.2                                                        |
| 24  | split decision site                                                      | `if separator_barcode in current_barcodes:`                                          | `tasks.py:L108`                                     | §5.2                                                        |
| 25  | `separate_pages`                                                         | `1 + len(separators)` segments; `[1]`→2, `[2,5]`→3, `[]`→0+warning                   | `tasks.py:L113-L161` (`L127`)                       | §5.3                                                        |
| 26  | `barcode_reader` / `pyzbar`                                              | decodes 39/128/QR; unreadable/none → `[]`                                            | `tasks.py:L75-L93` (`L82`)                          | §5.4                                                        |
| 27  | zero `Document` rows + `"File successfully split"`                       | count 0 before **and** after; original deleted                                       | `tasks.py:L184-L233` (`L233`)                       | §5.1                                                        |
| 28  | fixture counts `[0]` / `[1]` / `[2,5]`                                   | observed exactly                                                                     | `test_tasks.py:L207`,`L222`,`L232`                  | §5.3                                                        |
| 29  | QR / Code 39 / Code 128 / custom / unreadable / multi-separator          | all decoded/scanned as tabled                                                        | `tasks.py:L75`, `L96`                               | §5.3, §5.4                                                  |
| 30  | Q4 → Q1/Q2 training-data linkage                                         | 0 rows now; re-queue → later re-consumption (INFERRED)                               | `tasks.py:L164-L168`; `classifier.py:L125-L127`     | §5.5                                                        |

All 30 named items are addressed with observed evidence and grounded references. Negative results (items 9, and the `[]`/unreadable cases) are stated plainly where they are the truth.

---

## 7. Repository cleanliness

All observation/instrumentation scripts were created **outside** the repository tree — on the host under `/tmp/` scratch directories (`/tmp/qna_scratch/`, `/tmp/q1_scratch/`) and, after `docker cp`, in the canonical container under `/tmp/`. Nothing temporary was ever written inside the repository working tree, so no temporary file could ever appear in `git status`. The repository is byte-for-byte unchanged except for this single new document.

**Net change versus the baseline.** The deliverable branch descends from the source baseline commit `542221a38` (the repository `HEAD` at the start of the investigation). The complete set of changes the branch introduces relative to that baseline is exactly **one added file** — this document — as `git diff --name-status` shows (this evidence is stable regardless of how many commits the branch has, because it compares end-states):

```
$ git diff --name-status 542221a38..HEAD
A	blitzy/documentation/paperless-ngx_542221a38dff.md
```

The diff is entirely insertions of this one new file; **no pre-existing line anywhere in the repository is modified or deleted**.

**Working tree after the document is committed.** The answer document is a **committed, tracked** file on the branch — not an untracked one. A clean working tree therefore reports nothing: `git status --porcelain` and its untracked-expanding form `-uall` both print no output:

```
$ git status --porcelain
$ git status --porcelain -uall
$
```

(Both commands returned to the prompt with **zero lines** of output.) This is the corrected, current state. An **earlier pre-commit snapshot** — captured while the file was still untracked — instead showed the untracked markers `?? blitzy/` (collapsed to the top-level prefix) and, with `-uall`, `?? blitzy/documentation/paperless-ngx_542221a38dff.md`. Those untracked forms are **pre-commit only** and no longer apply once the file is committed to the branch.

No tracked source, test, configuration, or fixture file appears as modified, added, or deleted — verified in **both** the deliverable working tree and the canonical container checkout at `/app`, where `git status --porcelain` is likewise empty after the investigation (confirming every fixture a spy touched was restored). Temporary helpers removed on completion:

- Host scratch (`/tmp/qna_scratch/`, `/tmp/q1_scratch/`): the spy scripts `test_blitzy_q1_reuse.py`, `test_q1_nondet.py`, `test_q1_flip.py`, `test_q1_pytest_flake.py`, `test_blitzy_q2.py`, `test_blitzy_q3_ocr.py`, `test_blitzy_q3_store.py`, `test_blitzy_q4.py`, plus their captured-output logs — deleted.
- Container `/tmp`: the same spy scripts (copied in via `docker cp`) plus the OCR PATH shims `/tmp/binshim/{tesseract,gs}` and their argv log `/tmp/ocr_subprocess.log` — deleted.

> **Constraint compliance:** source was treated as read-only; the classifier non-determinism was **explained and evidenced, not fixed** (no `random_state` added, no test ordering pinned, `pytest-xdist` toggled only to diagnose). One fixture that a spy rewrote **in place** — `no-text-alpha.png`, which the parser overwrites while stripping its alpha layer at [`src/paperless_tesseract/parsers.py:L201`] (`background.save(input_file, format=im.format)`) — was **restored with `git checkout`**, so the source tree is byte-for-byte unchanged. The sole repository artifact is `blitzy/documentation/paperless-ngx_542221a38dff.md`.
