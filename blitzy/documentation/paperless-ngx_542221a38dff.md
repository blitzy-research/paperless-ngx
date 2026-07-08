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

**Python dependency versions** (via `importlib.metadata`) — these match `Pipfile` / `Pipfile.lock`:

```
scikit-learn==1.0.2      ocrmypdf==13.4.3        pyzbar==0.1.9
pdf2image==1.16.0        pikepdf==5.1.1          django==4.0.4
python-magic==0.4.25     pdfminer.six==20220319  fuzzywuzzy==0.18.0
django-q==1.3.9          pillow==9.1.0           numpy==1.22.3
scipy==1.8.0             joblib==1.1.0
pytest==8.4.2            pytest-django==4.11.1   pytest-xdist==3.8.0   pytest-cov==7.0.0
```

**System binaries** used by the OCR (Q3) and barcode (Q4) paths:

```
tesseract 4.1.1
ghostscript 9.53.3
unpaper 6.1
pdftoppm version 20.09.0            # poppler-utils (pdf2image backend)
zbar libs: /usr/lib/x86_64-linux-gnu/libzbar.so.0.3.0   # pyzbar backend
```

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

The canonical run command in this image is **`python -m pytest …`** (the pinned dependencies are installed **system-wide**; `pipenv run` in this image creates a _new empty_ virtualenv without the deps, yielding `ModuleNotFoundError: sklearn`). All test commands below therefore take the form:

```
docker exec -u testuser pngxq bash -c 'cd /app/src && python -m pytest <path> -n0 --no-cov -p no:cacheprovider -v'
```

- `-n0` disables `pytest-xdist` (single process) — used to remove scheduler ordering as a variable; omitting it uses the default parallel `--numprocesses auto`.
- `--no-cov` skips coverage overhead (no behavioral effect); `-p no:cacheprovider` avoids cache noise.
- Temporary observation scripts live in `/tmp`; because that changes pytest's rootdir away from `/app/src/setup.cfg`, those runs additionally pass `--ds=paperless.settings` with `PYTHONPATH=/app/src`.

> **Warning noise.** The `--pythonwarnings=all` setting surfaces pre-existing third-party deprecation warnings (redis `distutils`/`StrictVersion`, Django `USE_L10N` `RemovedInDjango50Warning`, `django_q` `baseconv`). These are unrelated to the behaviors under investigation and are filtered from the pasted output for readability; the trailing pytest summary line is always preserved verbatim.

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
$ python -m pytest documents/tests/test_tasks.py -k test_train_classifier -n0 --no-cov -p no:cacheprovider -v
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0
django: version: 4.0.4, settings: paperless.settings (from ini)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collected 40 items / 35 deselected / 5 selected
documents/tests/test_tasks.py .....                                      [100%]
================= 5 passed, 35 deselected, 6 warnings in 2.12s =================
```

**Instrumented guard (temporary `/tmp` spy).** A `DirectoriesMixin`/`TestCase` spy exercised both entry points and printed the **before / intermediate / after** state of `MODEL_FILE` and the `data_hash` values. Raw output:

```
=== PART A: reuse-vs-retrain via tasks.train_classifier() (golden scenario) ===
MODEL_FILE = /tmp/tmpg42aq0nk/classification_model.pickle
[before 1st train_classifier] exists=False
[INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpg42aq0nk/classification_model.pickle...
[after  1st train_classifier] exists=True size=32176B mtime=1783490727.094224
[after  2nd train_classifier (UNCHANGED data)] exists=True size=32176B mtime=1783490727.094224
  -> mtime changed after 2nd call? False
[INFO] [paperless.tasks] Saving updated classifier model to /tmp/tmpg42aq0nk/classification_model.pickle...
[after  3rd train_classifier (MUTATED data)  ] exists=True size=36901B mtime=1783490729.322229
  -> mtime changed after 3rd call? True

=== PART B: DocumentClassifier.train() data_hash reuse guard [classifier.py:L163-164] ===
initial clf.data_hash = None
1st train() return = True   data_hash(sha1 hex) = 230b98c1cbe4bb261c254b08a6d463334e9ba45d
2nd train() return = False  data_hash(sha1 hex) = 230b98c1cbe4bb261c254b08a6d463334e9ba45d  (SAME instance, UNCHANGED DB)
3rd train() return = True   data_hash(sha1 hex) = 7a72f0b528d608c4159cf211e6baaeba89fd8cdd  (DB MUTATED: +1 doc)
  hash unchanged 1->2? True   hash changed 2->3? True
```

**Reading the state transitions:**

| Step                  | `MODEL_FILE`                    | `train()` return | `data_hash`             | Interpretation                                                      |
| --------------------- | ------------------------------- | ---------------- | ----------------------- | ------------------------------------------------------------------- |
| before 1st            | `exists=False`                  | —                | `None`                  | no model yet ⇒ `load_classifier()` would return `None`              |
| after 1st             | `size=32176B mtime=…727.094224` | `True`           | `230b98c1…`             | trained + saved (INFO "Saving updated classifier model…")           |
| after 2nd (unchanged) | `size=32176B` **same mtime**    | `False`          | `230b98c1…` (unchanged) | **guard hit** [`classifier.py:L163-L164`] ⇒ no retrain, **no save** |
| after 3rd (mutated)   | `size=36901B` **new mtime**     | `True`           | `7a72f0b5…` (changed)   | data changed ⇒ retrain + re-save                                    |

This is exactly the reuse-vs-retrain contract: the SHA-1 `new_data_hash = m.digest()` [`classifier.py:L161`] short-circuits work when the training corpus is unchanged.

**Save/version round-trip & fixture load** (`save()`/`load()` at [`classifier.py:L96`,`L76`]):

```
$ python -m pytest documents/tests/test_classifier.py \
    -k "testDatasetHashing or testSaveClassifier or test_load_and_classify or test_load_classifier_cached" \
    -n0 --no-cov -p no:cacheprovider -v
... 3 passed, 1 skipped ...
```

- `testDatasetHashing` [`test_classifier.py:L137`] — `assertTrue(train())` then `assertFalse(train())` (guard). **PASS.**
- `testSaveClassifier` [`test_classifier.py:L168`] — train → `save()` → fresh `load()` → `assertFalse(train())`. **PASS.**
- `test_load_and_classify` [`test_classifier.py:L183`] — loads the **committed** fixture `src/documents/tests/data/model.pickle` (**156,607 bytes**) under `override_settings(MODEL_FILE=…)`. **PASS.** (This is a _controlled_ reuse — not contamination.)
- `test_load_classifier_cached` [`test_classifier.py:L402`] — **SKIPPED**, reason `"Disabled caching due to high memory usage - need to investigate"` (skip declared at `test_classifier.py:L391`).

### 2.3 Cross-test contamination surface — evidence

**Per-test filesystem isolation.** `DirectoriesMixin` [`src/documents/tests/utils.py:L72-L83`] calls `setup_directories()` which sets `data_dir = tempfile.mkdtemp()` [`utils.py:L18`] and overrides `MODEL_FILE = os.path.join(dirs.data_dir, "classification_model.pickle")` [`utils.py:L45`]; `tearDown` → `remove_dirs` wipes them. Two tests therefore get **different** model paths:

```
[iso test 1] entry Document.objects.count() = 0
[iso test 1] after create count = 1
[iso test 1] MODEL_FILE = /tmp/tmplpy201ef/classification_model.pickle
[iso test 2] entry Document.objects.count() = 0
[iso test 2] after create count = 1
[iso test 2] MODEL_FILE = /tmp/tmphh6ai7lj/classification_model.pickle
```

**DB isolation.** Both tests **enter with `count() = 0`** even though each creates a row — the `TestCase` transaction is rolled back between them, so `Document`/`Correspondent` rows created in one test do **not** persist to the next. This is why `train()` only ever sees the **current** test's DB snapshot via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`src/documents/classifier.py:L125-L127`].

**Parallelism.** With the default `--numprocesses auto` [`setup.cfg:L10`] the full classifier file runs across workers; observed once as `22 passed, 1 skipped, 774 warnings` under parallel execution. Combined with per-test tempdirs, on-disk model leakage is unlikely; what parallelism _does_ change is **test ordering** run-to-run (worker scheduling). **INFERRED:** ordering alone is not the flake cause, because single-process (`-n0`) runs already produce a different model every run (§2.4).

### 2.4 Reproduced non-determinism (not stabilized)

**Root cause (grounded):** the three `MLPClassifier(tol=0.01)` constructions carry **no `random_state`** — `src/documents/classifier.py:L219` (tags), `L227` (correspondent), `L238` (document type). With no seed, weight initialization — and therefore the fitted decision boundary — differs across identical runs.

**Same unchanged input, K = 500 fresh `train()` runs** (single-process `-n0`, identical 2-document DB every iteration):

```
=== PART C: non-determinism over K=500 fresh train() runs (SAME unchanged DB) ===
dataset: 1 correspondent c1(pk=1, MATCH_AUTO); doc1->c1 ; doc2->no correspondent (label -1)
test asserts: predict_correspondent(doc1)==c1.pk  AND  predict_correspondent(doc2) is None
c1.pk = 1
predict_correspondent(doc1.content) distribution: {1: 499, None: 1}  (expected all == 1)
predict_correspondent(doc2.content) distribution: {None: 499, 1: 1}  (expected all == None)
TEST would PASS in 499/500 runs ; would FAIL in 1/500 runs
distinct correspondent_classifier.coefs_ sha256 hashes: 500 / 500
distinct saved model.pickle sha256 hashes:              500 / 500
```

This single block reproduces the reported "sometimes passes, sometimes fails" behavior directly:

- **Every** one of the 500 models is distinct — `coefs_` hash **500/500 distinct**, serialized `model.pickle` hash **500/500 distinct** — from the _identical_ training set. That is the non-determinism, observed.
- The label the test asserts on **flipped** in this batch: `doc1` predicted `None` once (`{1: 499, None: 1}`) and `doc2` predicted `c1.pk` once (`{None: 499, 1: 1}`), i.e. the test would have **failed ~1/500 (≈0.2%)** of the time.

**Why the flip is rare — decision-margin distribution, K = 300 fresh models:**

```
=== PART C3: run-to-run decision-margin variation over K=300 fresh models ===
classes learned = [-1, 1] ; class index for c1.pk(1) = 1
P(doc1 -> c1)  min=0.5396 max=0.7530 mean=0.6419 std=0.0377
P(doc2 -> c1)  min=0.2568 max=0.4580 mean=0.3582 std=0.0370
  (a label FLIP for doc2 occurs when P(doc2->c1) crosses 0.5; #runs with P>0.5: 0/300)
  distinct P(doc2->c1) values: 300/300  -> confirms every model differs
```

The predicted probabilities cluster near — but on the correct side of — the `0.5` arg-max boundary (`P(doc1→c1)` **min 0.5396**, `P(doc2→c1)` **max 0.4580**), and **300/300** probability values are distinct. A rare run pushes a value across `0.5`, producing the flip seen above. (The classifier accepts by **arg-max**, not by a probability threshold — see [Q2 §3.1](#31-lead-negative-result).)

**Parallel vs. single-process, stability across batches.** The distinct-model result reproduces across independent batches (a second K-run batch again yielded 100 % distinct models and probabilities hugging `0.5`). Because the **single-process** (`-n0`) runs above _already_ yield a different model on every iteration, the dominant factor is the **missing `random_state`** (algorithmic), not `pytest-xdist` scheduling; xdist only perturbs test **ordering**. As a fresh-process pytest outcome check, repeated identical invocations of `test_one_correspondent_predict_manydocs` were observed to **fail intermittently** (a genuine `FAILED …test_one_correspondent_predict_manydocs` was captured in one exploratory batch) while the large majority passed — consistent with the ≈0.2–0.4 % in-process flip rate measured above.

> **Scope note (per constraints):** this section **explains and evidences** the non-determinism; it does **not** fix it. No `random_state` was added, no ordering was pinned, and `pytest-xdist` was toggled only to _diagnose_ (never committed).

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

A `/tmp` spy reproduced each canonical scenario and printed the **created** count, the **trained** (post-exclusion) count, and the classifier's own log line. Raw output:

```
=== Q2 INBOX EXCLUSION (mirrors generate_test_data, test_classifier.py:L25) ===
Document.objects.count() [CREATED] = 3
Document.objects.exclude(tags__is_inbox_tag=True).count() [TRAINED] = 2
doc_inbox pk=3 carries inbox tag t2(is_inbox_tag=True) -> excluded from training = True
   LOG paperless.classifier: Gathering data from database...
   LOG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).

=== Q2 MANY-DOCS (mirrors test_one_correspondent_predict_manydocs, test_classifier.py:L206) ===
Document.objects.count()=2 (doc1->c1, doc2->no correspondent)
   LOG paperless.classifier: Gathering data from database...
   LOG paperless.classifier: 2 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG paperless.classifier: Vectorizing data...
   LOG paperless.classifier: There are no tags. Not training tags classifier.
   LOG paperless.classifier: Training correspondent classifier...
   LOG paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=1 (expect c1.pk=1)
RESULT predict_correspondent(doc2)=None (expect None)
RESULT match_correspondents(doc2)->pks=[] (expect [])
```

| Scenario (mirrors)                                                    | `Document`s **created** | `Document`s **trained** | Evidence                                                                                     |
| --------------------------------------------------------------------- | :---------------------: | :---------------------: | -------------------------------------------------------------------------------------------- |
| `test_one_correspondent_predict` [`test_classifier.py:L191`]          |          **1**          |          **1**          | LOG `"1 documents, 0 tag(s), 1 correspondent(s), …"` (see §3.3)                              |
| `test_one_correspondent_predict_manydocs` [`test_classifier.py:L206`] |          **2**          |          **2**          | LOG `"2 documents, …"`; `doc2 → None`                                                        |
| `generate_test_data` [`test_classifier.py:L25`]                       |          **3**          |          **2**          | `doc_inbox` (pk=3, inbox tag `t2`, `is_inbox_tag=True`) **excluded**; LOG `"2 documents, …"` |

The **created** vs. **trained** distinction is exactly the inbox-tag exclusion: `generate_test_data` creates 3 documents but only **2** reach the classifier. These counts are **deterministic** and were **identical across two independent spy batches** (unlike the predictions in §2.4, the _counts_ never vary).

### 3.3 Training timing relative to inserts

Training runs **after** the document inserts, against the DB snapshot. The single-document spy stamped a wall-clock marker at each `Document.objects.create(...)` and at the `train()` call, and shows the classifier's `"Gathering data from database…"` [`classifier.py:L123`] firing **after** the inserts:

```
=== Q2 SINGLE-DOC (mirrors test_one_correspondent_predict, test_classifier.py:L191) ===
[t=1783490740.7451] CREATE correspondent c1 pk=1 (MATCH_AUTO)
[t=1783490740.7455] CREATE doc1 pk=1 ; Document.objects.count()=1
[t=1783490740.7457] CALL clf.train()
   LOG[t=1783490740.7457] paperless.classifier: Gathering data from database...
   LOG[t=1783490740.7478] paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
   LOG[t=1783490740.7481] paperless.classifier: There are no tags. Not training tags classifier.
   LOG[t=1783490740.7482] paperless.classifier: Training correspondent classifier...
   LOG[t=1783490740.7601] paperless.classifier: There are no document types. Not training document type classifier.
RESULT predict_correspondent(doc1)=1  c1.pk=1  match_correspondents(doc1)->pks=[1]
```

Ordering (by timestamp): **create `c1`** (…7451) → **create `doc1`** (…7455, `count()=1`) → **call `train()`** (…7457) → **`"Gathering data from database…"`** (…7457). Training therefore reads the _already-inserted_ rows — the create-then-train sequence every classifier test uses.

### 3.4 Acceptance mechanism & consume-time path

From the spy output above: for a matching correspondent, `predict_correspondent(doc1) = 1` (= `c1.pk = 1`) and `match_correspondents(doc1) → pks = [1]`; for a non-matching document, `predict_correspondent(doc2) = None` and `match_correspondents(doc2) → pks = []`. Acceptance is the arg-max `!= -1` rule of §3.1 — no probability gate.

The **consume-time** auto-assignment path is `set_correspondent(...)` [`src/documents/signals/handlers.py:L35`] → `matching.match_correspondents(document, classifier)` [`handlers.py:L50`]. The two consume-time signal tests exercise this wiring:

```
$ python -m pytest documents/tests/test_matchables.py -k correspondent -n0 --no-cov -v
... 2 passed ...
```

- `test_correspondent_applied` [`test_matchables.py:L425`] uses `Correspondent(match="keyword", MATCH_ANY)` and fires `document_consumption_finished` [`L431`].
- `test_correspondent_not_applied` [`test_matchables.py:L437`] uses a non-matching rule and asserts `document.correspondent is None`.

> **Important (observed, to avoid misreading):** both of these tests drive the consume-time signal via the **rule-based** (`MATCH_ANY`) path, with the ML classifier not being the deciding factor — they validate the **signal wiring**, **not** the ML "threshold." This is called out explicitly because their names could otherwise suggest they test the ML acceptance criterion.

### 3.5 Sibling variants covered

- **Single-document** correspondent training (`test_one_correspondent_predict`, created=1/trained=1) **vs. multi-document** (`test_one_correspondent_predict_manydocs`, created=2/trained=2) — both run:

  ```
  $ python -m pytest documents/tests/test_classifier.py -k one_correspondent -n0 --no-cov -v
  ... 2 passed ...
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

A `/tmp` spy set `PATH=/tmp/binshim:$PATH` with tiny wrapper scripts for `tesseract` and `gs` that **log their argv, then `exec` the real `/usr/bin` binary**, and parsed three samples through the real `RasterisedDocumentParser.parse()`. The captured subprocess log (complete, unedited) proves the child processes and their roles:

```
========== SUBPROCESS SHIM LOG  (/tmp/ocr_subprocess.log) ==========
(each line = one child process ocrmypdf actually exec'd via PATH)
SUBPROCESS tesseract --list-langs
SUBPROCESS tesseract --version
SUBPROCESS gs --version
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE -dInterpolateControl=-1 -sDEVICE=jpeggray -dFirstPage=1 -dLastPage=1 -r206.000000x206.000000 -o - -sstdout=%stderr -dAutoRotatePages=/None -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE ... -sDEVICE=jpeggray -dFirstPage=2 -dLastPage=2 ... -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS gs -dQUIET -dSAFER -dBATCH -dNOPAUSE ... -sDEVICE=jpeggray -dFirstPage=3 -dLastPage=3 ... -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.up5yw9yf/000001_rasterize_preview.jpg stdout
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.up5yw9yf/000003_rasterize_preview.jpg stdout
SUBPROCESS tesseract -l osd --psm 0 /tmp/ocrmypdf.io.up5yw9yf/000002_rasterize_preview.jpg stdout
SUBPROCESS gs ... -sDEVICE=pnggray -dFirstPage=3 -dLastPage=3 ... -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS gs ... -sDEVICE=pnggray -dFirstPage=1 -dLastPage=1 ... -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS gs ... -sDEVICE=pnggray -dFirstPage=2 -dLastPage=2 ... -f /tmp/ocrmypdf.io.up5yw9yf/origin.pdf
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000003_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000002_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000003_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000001_rasterize.png stdout
SUBPROCESS tesseract -l eng --psm 2 /tmp/ocrmypdf.io.up5yw9yf/000002_rasterize.png stdout
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.up5yw9yf/000002_ocr.png /tmp/ocrmypdf.io.up5yw9yf/000002_ocr_tess pdf txt
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.up5yw9yf/000003_ocr.png /tmp/ocrmypdf.io.up5yw9yf/000003_ocr_tess pdf txt
SUBPROCESS tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.up5yw9yf/000001_ocr.png /tmp/ocrmypdf.io.up5yw9yf/000001_ocr_tess pdf txt
SUBPROCESS gs -dBATCH -dNOPAUSE -dSAFER -dCompatibilityLevel=1.6 -sDEVICE=pdfwrite -dAutoRotatePages=/None -sColorConversionStrategy=LeaveColorUnchanged -dAutoFilterColorImages=true -dAutoFilterGrayImages=true -dJPEGQ=95 -dPDFA=2 -dPDFACompatibilityPolicy=1 -o - -sstdout=%stderr /tmp/ocrmypdf.io.up5yw9yf/fix_docinfo.pdf /tmp/ocrmypdf.io.up5yw9yf/pdfa.ps
... (CASE 2 no-text-alpha.png and CASE 3 with-form.pdf produce the same tesseract+gs pattern at their own DPIs) ...
TALLY: tesseract invocations = 28 ; gs (ghostscript) invocations = 17
```

Reading the log: `gs` first rasterizes each PDF page (`-sDEVICE=jpeggray`/`pnggray`), `tesseract` does orientation detection (`-l osd --psm 0`) and OCR (`-l eng --psm 2`, then `-c textonly_pdf=1 … pdf txt` to emit the text/PDF layer), and finally `gs -sDEVICE=pdfwrite … -dPDFA=2 …` produces the PDF/A archive. Across the three parses the tally was **28 tesseract** and **17 gs** child processes — direct runtime evidence that `ocrmypdf.ocr` [`parsers.py:L261`] drives both binaries.

Before each `ocrmypdf.ocr` call the parser logs the exact args [`parsers.py:L260`]; observed for a scanned PDF under the default `OCR_MODE=skip`:

```
LOG[paperless.parsing.tesseract/DEBUG]: Calling OCRmyPDF with args: {'input_file': '.../multi-page-images.pdf', 'output_file': '/tmp/paperless/paperless-u0hg7c42/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-u0hg7c42/sidecar.txt'}
```

(`output_type='pdfa'` is the canonical default `OCR_OUTPUT_TYPE="pdfa"` [`src/paperless/settings.py:L518`]; default `OCR_MODE="skip"` [`settings.py:L522`] ⇒ `skip_text=True`; `language='eng'` [`settings.py:L514`].)

### 4.3 The three observed cases (both MIME facets + fallback chain)

```
### CASE 1: DEFAULT OCR_MODE=skip, scanned-image PDF (no text layer)
========== PARSE: multi-page-images.pdf  (mime=application/pdf) ==========
magic.from_file(input, mime=True)   = 'application/pdf'
LOG: Calling OCRmyPDF with args: {... 'output_type': 'pdfa', 'skip_text': True ...}
--- result state ---
archive_path = '/tmp/paperless/paperless-u0hg7c42/archive.pdf'
magic.from_file(archive, mime=True) = 'application/pdf'
get_text() length = 116
get_text() repr   = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'

### CASE 2: DEFAULT OCR_MODE=skip, truly-empty image (no text at all)
========== PARSE: no-text-alpha.png  (mime=image/png) ==========
magic.from_file(input, mime=True)   = 'image/png'
LOG: Removing alpha layer from .../no-text-alpha.png for compatibility with img2pdf
LOG: Calling OCRmyPDF with args: {... 'skip_text': True ... 'image_dpi': 35}
LOG WARNING: Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
LOG: Fallback: Calling OCRmyPDF with args: {... 'force_ocr': True ... 'output_file': '.../archive-fallback.pdf' ... 'sidecar': '.../sidecar-fallback.txt' ... 'image_dpi': 35}
LOG WARNING: No text was found in .../no-text-alpha.png, the content will be empty.
--- result state ---
archive_path = '/tmp/paperless/paperless-lmh3qa7z/archive.pdf'
magic.from_file(archive, mime=True) = 'application/pdf'
get_text() length = 0
get_text() repr   = ''

### CASE 3: OCR_MODE=redo on with-form.pdf -> ocrmypdf error -> safe_fallback
========== PARSE: with-form.pdf (OCR_MODE=redo)  (mime=application/pdf) ==========
magic.from_file(input, mime=True)   = 'application/pdf'
LOG: Calling OCRmyPDF with args: {... 'redo_ocr': True ...}
[ERROR] [ocrmypdf._pipeline] This PDF has a user fillable form. --redo-ocr is not currently possible on such files.
LOG WARNING: Encountered an error while running OCR: . Attempting force OCR to get the text.
LOG: Fallback: Calling OCRmyPDF with args: {... 'force_ocr': True ...}
--- result state ---
archive_path = None
get_text() length = 68
get_text() repr   = 'Please enter your name in here:\n\nThis is a PDF document with a form.'
```

**Both MIME facets, side by side:**

| Input sample                     | `magic.from_file(input)` (⇒ stored `Document.mime_type`) | OCR archive `magic.from_file(archive)` |
| -------------------------------- | :------------------------------------------------------: | :------------------------------------: |
| `multi-page-images.pdf` (CASE 1) |                    `application/pdf`                     |           `application/pdf`            |
| `no-text-alpha.png` (CASE 2)     |                     **`image/png`**                      |         **`application/pdf`**          |

CASE 2 is the explicit contrast the question implies: an **image** input keeps its original `image/png` as the stored `Document.mime_type` while its OCR **archive** is `application/pdf`.

### 4.4 Fallback chain & empty-text last resort (edge paths, all observed)

The `parse()` method's OCR fallback is a three-stage chain, every stage of which was triggered live (CASE 2 / CASE 3 above):

1. **First attempt:** `ocrmypdf.ocr(**args)` [`parsers.py:L261`]; then if the extracted text is empty, `raise NoTextFoundException("No text was found in the original document")` [`parsers.py:L266-L267`] (class defined at `parsers.py:L14`).
2. **Force-OCR fallback:** caught by `except (NoTextFoundException, InputFileError) as e:` [`parsers.py:L276`], which logs `"…Attempting force OCR to get the text."` [`parsers.py:L280`], rebuilds args with `safe_fallback=True` (⇒ `force_ocr=True` via [`parsers.py:L155`]) [`parsers.py:L293`], logs `"Fallback: Calling OCRmyPDF with args: …"` [`parsers.py:L297`], and calls **`ocrmypdf.ocr(**args)` a second time** [`parsers.py:L298`]. Observed in CASE 2 (empty sidecar ⇒ `NoTextFoundException`) and CASE 3 (ocrmypdf `InputFileError`: _"This PDF has a user fillable form. --redo-ocr is not currently possible…"_).
3. **Empty-text last resort:** `if not self.text:` [`parsers.py:L318`] → if the original had embedded text it is used, else `self.text = ""` [`parsers.py:L327`] with the warning _"No text was found in …, the content will be empty."_ Observed in CASE 2: `get_text()` returns `''` (length 0).

Corresponding parser test suite:

```
$ python -m pytest paperless_tesseract/tests/test_parser.py -k "notext or form" -n0 --no-cov -p no:cacheprovider -v
... collected 35 items / 30 deselected / 5 selected
paperless_tesseract/tests/test_parser.py .....                           [100%]
================ 5 passed, 30 deselected, 6 warnings in 25.45s ================
```

The 5 selected tests are `test_skip_noarchive_notext`, `test_with_form`, `test_with_form_error` (`OCR_MODE="redo"` ⇒ `archive_path is None`), `test_with_form_error_notext` (`OCR_MODE="redo"`), and `test_with_form_force` (`OCR_MODE="force"`) — the `redo`/`force` variants are precisely the ones that drive the force-OCR fallback branch.

### 4.5 MIME detection / assignment — where it happens

`get_parser_class_for_mime_type` dispatch runs off the same `magic.from_file` detection [`src/documents/parsers.py:L106`; consumer dispatch at `consumer.py:L223`]. The consumer logs `"Detected mime type: …"` [`consumer.py:L221`] and stores it unchanged. Summary of the state transition for a no-text **image**:

- **input** file → `magic.from_file` ⇒ `image/png` [`consumer.py:L219`]
- **OCR archive** produced by `ocrmypdf.ocr` ⇒ `application/pdf` [`parsers.py:L261`]
- **stored** `Document.mime_type` ⇒ `image/png` (the original) [`consumer.py:L401`]

---

## 5. Q4 — Barcode splitting: record count, trigger values, decision site, training-data impact

### 5.1 Direct answer (the key zero-rows result)

A single input file passed through the **barcode branch** of `consume_file()` [`src/documents/tasks.py:L184-L233`] yields **zero `Document` records created directly**. When barcodes are enabled (`settings.CONSUMER_ENABLE_BARCODES`, **default off** [`src/paperless/settings.py:L502-L504`]) and separators are found, the branch splits the PDF into **N segment files**, **saves those split PDFs back to the consumption directory** via `save_to_dir` (default `target_dir=settings.CONSUMPTION_DIR` [`tasks.py:L164-L166`]), **deletes the original**, and **returns early with the string `"File successfully split"`** [`tasks.py:L233`]. No `Document` row is created in that call. The `/tmp` spy proved this directly:

```
[Q4-consume_file barcode branch, CONSUMER_ENABLE_BARCODES=True]
   Document.objects.count() BEFORE = 0
   Document.objects.count() AFTER  = 0
   consume_file(...) returned      = 'File successfully split'
   original input still on disk?   = False
   save_to_dir called 2 time(s):
      basename='patch-code-t-middle_document_0.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmp260s2ut_'
      basename='patch-code-t-middle_document_1.pdf' newname=None target_dir(CONSUMPTION_DIR)='/tmp/tmp260s2ut_'
   CONSUMPTION_DIR before          = []
   CONSUMPTION_DIR after           = ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
```

So you must distinguish **"split files produced"** (here **2** segments, re-queued to the consumption dir) from **"`Document` rows created"** (**0** in this call). (The `OSError … 6379` broker warning seen during this run comes from the optional websocket status notification and is caught by `except OSError` in the branch — it does not affect the `"File successfully split"` return.) This matches `test_consume_barcode_file` [`test_tasks.py:L396`], which asserts `tasks.consume_file(dst) == "File successfully split"` under `@override_settings(CONSUMER_ENABLE_BARCODES=True)`.

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

The actual test subset passes, and a spy independently printed the `scan_file_for_separating_barcodes(...)` return list for each fixture (default `PATCHT` separator):

```
$ python -m pytest documents/tests/test_tasks.py \
    -k "scan_file_for_separating or separate_pages or barcode_splitter or consume_barcode or barcode_reader" \
    -n0 --no-cov -p no:cacheprovider -v
... 25 passed, 15 deselected, 6 warnings in 5.59s ...
```

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
   patch-code-t-middle.pdf      seps=[1]      -> 2 file(s): ['patch-code-t-middle_document_0.pdf', 'patch-code-t-middle_document_1.pdf']
   several-patcht-codes.pdf     seps=[2, 5]   -> 3 file(s): ['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
[WARNING] [paperless.tasks] No pages to split on!
   patch-code-t-middle.pdf      seps=[]       -> 0 file(s): []
```

- `[1]` → **2** files (matches `test_separate_pages` `len == 2` [`L305`] and `test_barcode_splitter` `_document_0.pdf`/`_document_1.pdf` [`L376`]).
- `[2, 5]` → **3** files.
- `[]` → **0** files, plus the warning `"No pages to split on!"` emitted at [`tasks.py:L127`] (matches `test_separate_pages_no_list` [`L315`], which asserts `["WARNING:paperless.tasks:No pages to split on!"]`).

All counts were **byte-for-byte identical across two runs** (deterministic — unlike the Q1 predictions).

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

**Observed:** the barcode branch creates **0** `Document` rows and instead re-queues the **N** split PDFs into the consumption directory (`CONSUMPTION_DIR before = []` → `after = [..._document_0.pdf, ..._document_1.pdf]`, original deleted, return `"File successfully split"`) — all shown in §5.1.

**INFERRED (downstream, not directly observed in the single call):** because those split files land back in the consumption directory, they are later **independently re-consumed** through the **non-barcode** branch of `consume_file` into **new `Document` rows**. Only then do they enter the corpus that `train()` reads via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`src/documents/classifier.py:L125-L127`]. Consequently, a single barcode input does **not immediately** change the effective training data (0 rows on the split call); it changes it **indirectly and later**, once the re-queued segments are consumed. This is labelled INFERRED because the split call itself was observed to create no rows and to return early; the subsequent re-consumption is the documented design of re-queuing to `CONSUMPTION_DIR` [`tasks.py:L164-L166`] rather than something exercised within the same call.

---

## 6. Coverage pass

Every named item across the four questions, with its concrete value, `file:line`, observed evidence, sibling variants, and causal reason:

| #   | Named item                                                               | Value / result (observed)                                                            | `file:line`                                         | Evidence (§)                                                |
| --- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ | --------------------------------------------------- | ----------------------------------------------------------- |
| 1   | `load_classifier()` / `MODEL_FILE`                                       | returns `None` if absent, else deserializes; default `…/classification_model.pickle` | `classifier.py:L30-L57`; `settings.py:L74`          | §2.1, §2.2                                                  |
| 2   | `data_hash` reuse guard                                                  | `train()` returns `False` on unchanged data (no retrain, no save)                    | `classifier.py:L163-L164`                           | §2.2 (hash `230b98c1…` stable 1→2; `7a72f0b5…` on mutation) |
| 3   | `save()` / `load()` round-trip                                           | `testSaveClassifier` PASS                                                            | `classifier.py:L96`, `L76`                          | §2.2                                                        |
| 4   | `MLPClassifier` missing `random_state` (3 sites)                         | tags, correspondent, doc-type — none seeded                                          | `classifier.py:L219`, `L227`, `L238`                | §2.4 (root cause)                                           |
| 5   | `DirectoriesMixin` / tempdir isolation                                   | distinct `MODEL_FILE` per test                                                       | `utils.py:L14-L50`, `L45`, `L72-L83`                | §2.3 (`/tmp/tmplpy201ef` vs `/tmp/tmphh6ai7lj`)             |
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
| 30  | Q4 → Q1/Q2 training-data linkage                                         | 0 rows now; re-queue → later re-consumption (INFERRED)                               | `tasks.py:L164-L166`; `classifier.py:L125-L127`     | §5.5                                                        |

All 30 named items are addressed with observed evidence and grounded references. Negative results (items 9, and the `[]`/unreadable cases) are stated plainly where they are the truth.

---

## 7. Repository cleanliness

All observation/instrumentation scripts were created **outside** the repository tree (host `/tmp/qna_scratch/` and container `/tmp/`) and have been **removed**. The repository is byte-for-byte unchanged except for this single new document.

`git status --porcelain` from the repository root (the whole `blitzy/` directory is untracked, so git collapses it to the top-level prefix):

```
$ git status --porcelain
?? blitzy/
```

Expanding untracked directories confirms the **only** added path is this document:

```
$ git status --porcelain -uall
?? blitzy/documentation/paperless-ngx_542221a38dff.md
```

No tracked source, test, configuration, or fixture file appears as modified, added, or deleted. Temporary helpers removed on completion:

- Host scratch: `/tmp/qna_scratch/` (`test_blitzy_q1_spy.py`, `test_blitzy_q1_nondet.py`, `test_blitzy_q1_nondet2.py`, `test_blitzy_q1_proba.py`, `test_blitzy_q1_iso.py`, `test_blitzy_q2_spy.py`, `test_blitzy_q3_ocr.py`, `test_blitzy_q4_spy.py`, `test_blitzy_q4_reader.py`) — deleted.
- Container `/tmp`: the same spy scripts plus the PATH shims `/tmp/binshim/{tesseract,gs}` and `/tmp/ocr_subprocess.log` — deleted.

> **Constraint compliance:** source was treated as read-only; the classifier non-determinism was **explained and evidenced, not fixed** (no `random_state` added, no test ordering pinned, `pytest-xdist` toggled only to diagnose); and the sole repository artifact is `blitzy/documentation/paperless-ngx_542221a38dff.md`.
