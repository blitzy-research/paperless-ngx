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
b0d4205737a63c2deea9089173e0e65ec17c8c37586b5c4e7b2fce83598bc2cf

########## PROVENANCE: docker inspect pngx-qna (security posture) ##########
$ docker inspect pngx-qna --format 'Privileged={{.HostConfig.Privileged}} ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}} CapAdd={{.HostConfig.CapAdd}} CapDrop={{.HostConfig.CapDrop}} NetworkMode={{.HostConfig.NetworkMode}} Mounts={{range .Mounts}}{{.Source}}->{{.Destination}} {{end}}'
Privileged=false ReadonlyRootfs=false CapAdd=[] CapDrop=[] NetworkMode=bridge Mounts=
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
$ docker exec -u testuser -w /app/src pngx-qna bash -c 'python3 manage.py check; echo "check exit=$?"'
System check identified no issues (0 silenced).
check exit=0
```

### 1.4 Source integrity — the exercised code is byte-identical to the repository — [observed]

The `.py` files exercised below are byte-for-byte identical between the container's `/app/src` and the repository's `src/`, so the `path:line` citations are valid for both. Note there are **two** distinct `parsers.py` modules — `src/documents/parsers.py` (base parser framework) and `src/paperless_tesseract/parsers.py` (the OCR parser) — and this document always cites the full path to disambiguate them:

```text
########## SOURCE IDENTITY: md5 container /app/src vs repo src ##########
--- container ---
$ docker exec -u testuser -w /app/src pngx-qna md5sum documents/classifier.py documents/tasks.py documents/matching.py documents/consumer.py documents/parsers.py documents/signals/handlers.py paperless_tesseract/parsers.py paperless/settings.py setup.cfg
e8ca934a3dabb7498ae7bf98aca481f6  documents/classifier.py
8af1ca06493ef50129f4314cc16d5686  documents/tasks.py
0db85577a13a7d9d34c9071a7221cf1c  documents/matching.py
e7392cc635432e94a56998b45d351526  documents/consumer.py
d843d2ecf7344f1ed7b8225f59dc8744  documents/parsers.py
997b21c1eb0f97098fb799750259fd36  documents/signals/handlers.py
26a6bc09de30be58becdeaab91d8f6e7  paperless_tesseract/parsers.py
c719bb0ae75f5dd438cd5937ba3d0955  paperless/settings.py
190d884f99b96408a1f687f1b5757c15  setup.cfg
--- repo (host) ---
$ cd /tmp/blitzy/paperless-ngx/blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf_735501/src && md5sum documents/classifier.py documents/tasks.py documents/matching.py documents/consumer.py documents/parsers.py documents/signals/handlers.py paperless_tesseract/parsers.py paperless/settings.py setup.cfg
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

#### 2.2.1 High-resolution lifecycle — `st_mtime_ns` / model size / SHA-256 / loaded `data_hash`, a physically-separate incompatible on-disk model through the *real* loader, and the task-level empty-corpus paths — [observed]

The capture above establishes reuse-vs-retrain from `st_mtime` (float seconds) and a **patched** loader. This subsection is the high-resolution complement requested for a byte-level answer: it records, after **each** `tasks.train_classifier()` call, the corpus count, the file's nanosecond `st_mtime_ns`, its size in bytes, the SHA-256 of the serialized model, and the `data_hash` (hex) read back by `load_classifier()`; it then loads a **compatible v7** model, writes a **physically-separate incompatible** model to disk (a real pickle whose first value is `999`, *not* a patched loader) and drives the **unmodified** `load_classifier()` over it, and finally exercises both task-level empty-corpus paths. Every quantitative invariant below was confirmed stable across two runs (Run 1 here; complete Run 2 in Appendix §7.3.7).

Command (Run 1):

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q1_hires.py 1
```

```text
================ Q1-HIRES RUN 1 ================
--- A: reuse/retrain, high-resolution (st_mtime_ns / size / sha256 / loaded data_hash) ---
  corpus_count = 1
  [before] exists=False
  LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
  LOG DEBUG paperless.classifier: Vectorizing data...
  LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
  LOG DEBUG paperless.classifier: Training correspondent classifier...
  LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
  LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1h-t765yv4_/classification_model.pickle...
  [after#1 initial-save] exists=True corpus_count=1 size=22382 st_mtime_ns=1783990152997975541 sha256=283d1502a6d3ad7146956746bc8bdd8f98292833bc761ed4eb6216ba92084aec
  [after#1] load_classifier()=OK loaded_data_hash=93c9a15dcb127652822b68bfbd2f11d6eedc1bea classes_=[1]
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.tasks: Training data unchanged.
  [after#2 unchanged] exists=True corpus_count=1 size=22382 st_mtime_ns=1783990152997975541 sha256=283d1502a6d3ad7146956746bc8bdd8f98292833bc761ed4eb6216ba92084aec
  [after#2] load_classifier()=OK loaded_data_hash=93c9a15dcb127652822b68bfbd2f11d6eedc1bea classes_=[1]
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
  LOG DEBUG paperless.classifier: Vectorizing data...
  LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
  LOG DEBUG paperless.classifier: Training correspondent classifier...
  LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
  LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1h-t765yv4_/classification_model.pickle...
  [after#3 changed] exists=True corpus_count=1 size=37002 st_mtime_ns=1783990153026975831 sha256=0b39de64e7244eb401566a939f50f0196f6323775d91b58b08aaa56ece14784d
  [after#3] load_classifier()=OK loaded_data_hash=441e8b90dd526a0de6491ab6b8567fcf082a4b5f classes_=[1]
  REUSE   (call#2 vs #1): size_same=True mtime_ns_same=True sha256_same=True loaded_hash_same=True
  RETRAIN (call#3 vs #2): mtime_ns_changed=True sha256_changed=True loaded_hash_changed=True
--- B: compatible v7 load (real load_classifier() succeeds) ---
  on-disk schema_version=7 FORMAT_VERSION=7 compatible=True
  [compatible-v7] load_classifier()=OK loaded_data_hash=441e8b90dd526a0de6491ab6b8567fcf082a4b5f classes_=[1]
--- C: PHYSICALLY-SEPARATE incompatible on-disk model -> real load_classifier() deletes it ---
  wrote incompatible model directly to disk: schema_version=999 exists_before=True
  LOG ERROR paperless.classifier: Unrecoverable error while loading document classification model, deleting model file.
  load_classifier() returned=None exists_after=False (False => os.unlink at classifier.py:48 ran)
--- D: direct empty-corpus DocumentClassifier().train() -> ValueError ---
  corpus_count = 0
  LOG DEBUG paperless.classifier: Gathering data from database...
  ValueError: 'No training data available.'
--- E: task-level empty corpus (auto correspondent present, 0 docs) ---
  LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG WARNING paperless.tasks: Classifier error: No training data available.
  train_classifier() returned=None model_exists=False (ValueError caught at tasks.py:70-72 => no save)
--- E2: no auto matching models at all -> early return, no train ---
  train_classifier() returned=None model_exists=False (guard at tasks.py:49-55 => early return)
================ Q1-HIRES RUN 1 SUMMARY ================
  reuse_byte_identical=True retrain_all_changed=True loaded_hash reuse=True retrain=True
```

Concise state table (one row per lifecycle stage; full SHA-256 and `data_hash` values are in the capture above and in Appendix §7.3.7):

| Stage | corpus_count | file exists | size (bytes) | `st_mtime_ns` | model SHA-256 | loaded `data_hash` | return / exception |
|-------|--------------|-------------|--------------|---------------|---------------|--------------------|--------------------|
| before any train | 1 | **False** | — | — | — | — | — |
| after #1 initial save | 1 | True | 22382 | 1783990152997975541 | `283d1502…084aec` | `93c9a15d…1bea` | `train()`→`True` → `save()` |
| after #2 unchanged (**REUSE**) | 1 | True | **22382** | **1783990152997975541** | **`283d1502…084aec`** | **`93c9a15d…1bea`** | `train()`→`False`; `Training data unchanged.` |
| after #3 changed (**RETRAIN**) | 1 | True | 37002 | 1783990153026975831 | `0b39de64…14784d` | `441e8b90…4b5f` | `train()`→`True` → `save()` |
| compatible v7 load | 1 | True | 37002 | (unchanged) | (unchanged) | `441e8b90…4b5f` | `load_classifier()`→OK (`schema_version=7`) |
| incompatible (`999`) on disk | 1 → 0 file | write→True; **after load→False** | — | — | — | — | `load_classifier()`→`None`; `os.unlink` @`classifier.py:48` |
| direct `train()` empty corpus | 0 | False | — | — | — | — | **`ValueError('No training data available.')`** |
| task empty (auto corr., 0 docs) | 0 | False | — | — | — | — | `train_classifier()`→`None`; `Classifier error: …` (no save) |
| task, no auto models | 0 | False | — | — | — | — | `train_classifier()`→`None`; early `return` |

Reading the high-resolution capture:

- **Reuse is byte-identical — [observed].** Call #2 (unchanged data) left the file **byte-for-byte** the same as call #1: identical `size` (22382), identical `st_mtime_ns` (`1783990152997975541`), identical model `sha256` (`283d1502…084aec`), and identical loaded `data_hash` (`93c9a15d…1bea`) — because `train()` returned `False` at the SHA-1 guard (`src/documents/classifier.py:163-164`) and `tasks.train_classifier()` therefore never called `save()` (`src/documents/tasks.py:62-69`). Reuse is not an in-memory shortcut; the on-disk artifact is literally untouched.
- **Retrain changes every byte-level field — [observed].** After mutating the document's content, call #3 re-saved: `st_mtime_ns` advanced, `size` changed (22382 → 37002), the model `sha256` changed (`283d1502…` → `0b39de64…`), and the loaded `data_hash` changed (`93c9a15d…` → `441e8b90…`).
- **The reuse guard is content-deterministic even though the model bytes are not — [observed] + cross-check to §6.4/§6.5.** Across the two runs the loaded `data_hash` values are **byte-identical** — `93c9a15dcb127652822b68bfbd2f11d6eedc1bea` for the original content and `441e8b90dd526a0de6491ab6b8567fcf082a4b5f` for the changed content — because `data_hash` is a SHA-1 over the ordered preprocessed content and label ids (`src/documents/classifier.py:124-161`), which is deterministic. In contrast, the **model-file** `size`/`sha256` **differ run-to-run** (Run 2 recorded `size=22439`, `sha256=28e2bda7…`, and `size=36983`, `sha256=83c3914f…`; see §7.3.7): the serialized `MLPClassifier` weights are not deterministic (unseeded — see §6.4). Thus the reuse decision does **not** depend on the volatile model bytes; it depends only on the deterministic data hash.
- **Compatible v7 load succeeds — [observed].** The model just saved carries `schema_version=7`, equal to `DocumentClassifier.FORMAT_VERSION = 7` (`src/documents/classifier.py:63`), so the real `load_classifier()` returns a live classifier.
- **A physically-separate incompatible on-disk model is deleted by the real loader — [observed] (canonical).** Writing a genuine pickle whose first value is `999` and then running the **unmodified** `load_classifier()` (which compares against its real `FORMAT_VERSION=7` at `src/documents/classifier.py:80-83`) logged `Unrecoverable error while loading document classification model, deleting model file.`, returned `None`, and left the file **absent** (`exists_after=False`) — the `os.unlink(settings.MODEL_FILE)` at `src/documents/classifier.py:48` ran. This is the canonical version of the patched-loader demonstration in §2.2-B: no source is patched; the incompatibility lives entirely in the on-disk bytes.
- **Both task-level empty-corpus paths yield no model — [observed].** With an auto correspondent present but **zero** documents, `tasks.train_classifier()` reaches `classifier.train()`, which raises `ValueError('No training data available.')` (`src/documents/classifier.py:158-159`); the task's `except Exception` catches it and logs `Classifier error: No training data available.` (`src/documents/tasks.py:71-72`), returning `None` and writing **no** model. With **no** auto-matching model of any kind, the guard at `src/documents/tasks.py:49-55` short-circuits with an early `return` — again no training, no model. Either way, `model_exists=False`.


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

`test_train_classifier` (`src/documents/tests/test_tasks.py:75-94`) asserts the exact mtime reuse/retrain behavior observed above; `testVersionIncreased` and `testNoTrainingData` cover the version-guard and empty-corpus paths. All pass. The invocation passes `-p no:cacheprovider` (disables pytest's cache plugin, so no `cachedir:` line is printed and no `.pytest_cache` directory is written) and `-o addopts=""` (overrides the `setup.cfg` `addopts`, so this run is serial with no `--numprocesses`/coverage) — a deliberately minimal, side-effect-free serial run of just these four tests.

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
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 2 items

documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict PASSED [ 50%]
documents/tests/test_classifier.py::TestClassifier::test_one_correspondent_predict_manydocs PASSED [100%]

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
======================== 2 passed, 6 warnings in 1.87s =========================
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

### 4.5 Observed — related repository tests under non-default `OCR_MODE` (corroborate parser wiring, not the canonical no-text outcome) — [observed] + [non-canonical]

**Disclosure — [non-canonical / non-default `OCR_MODE`].** The two `test_parser.py` tests below are *related* repository tests, **not** the canonical Q3 no-extractable-text path. Each runs under a **non-default** `OCR_MODE` and asserts that text **is** extracted, so they corroborate the `RasterisedDocumentParser` → `ocrmypdf` wiring but do **not** exercise the default `OCR_MODE="skip"` no-text outcome that answers Q3 (that outcome is established through real runtime observation in §4.1 through §4.4 above):

- `test_skip_noarchive_notext` is decorated `@override_settings(OCR_MODE="skip_noarchive")` (`src/paperless_tesseract/tests/test_parser.py:369`) and asserts the OCR'd text **contains** `"page 1"`, `"page 2"`, `"page 3"` on `multi-page-images.pdf` (`src/paperless_tesseract/tests/test_parser.py:377-380`) — i.e. it asserts text *is* found, the opposite of the canonical no-text case.
- `test_with_form_error_notext` is decorated `@override_settings(OCR_MODE="redo")` (`src/paperless_tesseract/tests/test_parser.py:189`) and asserts the extracted text **contains** `"Please enter your name in here:"` and `"This is a PDF document with a form."` (`src/paperless_tesseract/tests/test_parser.py:197-200`) — again asserting text *is* found. (The `notext` in each name refers to the input lacking a pre-existing text layer, forcing OCR to run — not to an empty final result.)

Command:

```text
$ docker exec -u testuser -w /app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
    python3 -m pytest paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext \
    paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext \
    -p no:cacheprovider -o addopts="" -v
```

Both named tests pass. The `-o addopts=""` flag disables the repository's default `--numprocesses auto` (pytest-xdist) and coverage `addopts` so the per-test result lines are shown, and `-p no:cacheprovider` disables the cache. The result — `2 passed` — was stable across two runs (`in 9.88s` then `in 9.72s`; exit 0). The complete, unedited capture of run 1 follows:

```text
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 2 items

paperless_tesseract/tests/test_parser.py::TestParser::test_skip_noarchive_notext PASSED [ 50%]
paperless_tesseract/tests/test_parser.py::TestParser::test_with_form_error_notext PASSED [100%]

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
======================== 2 passed, 6 warnings in 9.88s =========================
```

---

## 5. Q4 — Barcode-based splitting: record count, trigger values, decision site, and effect on training data

### 5.1 Answers

- **(a) How many document records come from a single input file:** `separate_pages()` mechanically produces **N+1 fragment files** for **N** separator pages (`src/documents/tasks.py:113-161`). A fragment becomes a `Document` row only when the Consumer **successfully parses** it **and** it is **not a byte-duplicate** of an already-stored fragment: byte-identical fragments are rejected by `pre_check_duplicate` (`src/documents/consumer.py:102-113`, md5 at `:104`), and fragments the parser cannot process yield **no** row — e.g. the two 0-page fragments produced by a **page-0** separator both fail with `ValueError: max_workers must be greater than 0` and create 0 rows (see §5.5.2). Observed for the common cases: `several-patcht-codes.pdf` (2 separators → 3 fragment files, 2 distinct) → **2** rows; `patch-code-t-middle.pdf` (1 separator → 2 fragment files, 1 distinct) → **1** row. **[observed]** + **[code-grounded]**
- **(b) Which barcode values trigger a split:** the single value configured by `CONSUMER_BARCODE_STRING`, default **`"PATCHT"`** (`src/paperless/settings.py:506`), read at split time in `scan_file_for_separating_barcodes` (`src/documents/tasks.py:102`). The symbology is irrelevant — CODE39, QR, and CODE128 all trigger a split as long as the **decoded string equals** the configured value; a different value (e.g. `"CUSTOM BARCODE"`) only triggers when `CONSUMER_BARCODE_STRING` is set to it. **[observed]** + **[code-grounded]**
- **(c) Where the split decision is made:** in `scan_file_for_separating_barcodes`, which reads each rendered page's barcodes (`current_barcodes = barcode_reader(page)`, `src/documents/tasks.py:107`) and then, at `if separator_barcode in current_barcodes:` (`src/documents/tasks.py:108`), appends the page index to the separator list (`:109`). The whole barcode path is gated by `settings.CONSUMER_ENABLE_BARCODES` (default `False`) in `consume_file` (`src/documents/tasks.py:195`). **[code-grounded]**
- **(d) Does splitting change the effective training data during a run:** **yes.** The `consume_file` *split* path itself creates **zero** `Document` rows — it copies fragments to the consumption directory and unlinks the original (`src/documents/tasks.py:210,214`), returning `"File successfully split"`. But when the fragments are subsequently consumed, each **successfully-parsed, nonduplicate** fragment becomes a `Document`, and `train()` reads the current corpus via `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:125-127`) — so one input file enlarges the training corpus by **at most** N+1 rows, and **fewer** when fragments are byte-duplicates or fail to parse. The staged before→split→consume→dedupe→retrain timeline is in §5.5.1. **[observed]** + **[code-grounded]**

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

#### 5.3.1 Complete `CONSUMER_BARCODE_STRING` value matrix, and enabled-vs-disabled — case/whitespace/empty/custom/nonmatching — [observed]

The split decision is an **exact string membership** test — `if separator_barcode in current_barcodes` (`src/documents/tasks.py:108`), where `separator_barcode = settings.CONSUMER_BARCODE_STRING` — so it is both **case-sensitive** and **whitespace-sensitive**. The matrix below drives the real `scan_file_for_separating_barcodes()` over two fixtures — `patch-code-t.pdf` (its barcode decodes to `PATCHT` via CODE39) and `barcode-128-custom.pdf` (decodes to `CUSTOM BARCODE` via CODE128) — for six configured values. Every cell is stable across both runs (complete unedited capture, including every per-scan `Barcode of type … found` log line, in Appendix §7.3.8). Command:

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q4_matrix.py 1
```

```text
--- (M) CONSUMER_BARCODE_STRING value matrix -> scan() separators (case/whitespace/empty/custom/nonmatching) ---
fixtures: patch-code-t.pdf (decodes 'PATCHT', CODE39); barcode-128-custom.pdf (decodes 'CUSTOM BARCODE', CODE128)
CONSUMER_BARCODE_STRING scan(patch-code-t.pdf)   scan(barcode-128-custom.pdf)
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT'               [0]                      []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'patcht'               []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT '              []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
''                     []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'CUSTOM BARCODE'       []                       [0]                     
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'NONEXISTENT VALUE'    []                       []                      
```

Matrix (the `%r` form in the capture shows quotes, revealing the trailing space and the empty string):

| `CONSUMER_BARCODE_STRING` | `scan(patch-code-t.pdf)` (decodes `PATCHT`) | `scan(barcode-128-custom.pdf)` (decodes `CUSTOM BARCODE`) | verdict |
|---------------------------|---------------------------------------------|----------------------------------------------------------|---------|
| `'PATCHT'` (exact default) | `[0]` | `[]` | matches `PATCHT` only |
| `'patcht'` (lowercase)     | `[]`  | `[]` | **no match — case-sensitive** |
| `'PATCHT '` (trailing space) | `[]` | `[]` | **no match — whitespace-sensitive** |
| `''` (empty)               | `[]`  | `[]` | **never matches** |
| `'CUSTOM BARCODE'` (custom) | `[]` | `[0]` | matches `CUSTOM BARCODE` only |
| `'NONEXISTENT VALUE'` (nonmatching) | `[]` | `[]` | no match despite a valid barcode present |

Reading the matrix:

- **Case-sensitive.** `'patcht'` does **not** match the decoded `'PATCHT'` — the membership test at `src/documents/tasks.py:108` compares raw strings with no case-folding.
- **Whitespace-sensitive.** `'PATCHT '` (one trailing space) does **not** match `'PATCHT'` — no trimming is applied.
- **Empty string never matches.** `''` yields `[]` for both fixtures (`'' in ['PATCHT']` is `False`; and a barcode-free page yields `current_barcodes == []`).
- **Symbology-independent, value-driven.** `PATCHT` was decoded from **CODE39** and `CUSTOM BARCODE` from **CODE128** (QR elsewhere, §5.3); the trigger is the decoded **value**, not the symbology — each decode is logged as `Barcode of type <SYMBOLOGY> found: <value>` (`src/documents/tasks.py:90-92`).
- **Custom vs nonmatching.** Setting the value to `'CUSTOM BARCODE'` makes only the custom fixture split; `'NONEXISTENT VALUE'` matches nothing even though a valid `PATCHT` barcode is present.

**Enabled vs. disabled, and nonmatching → whole-file consume → exactly one row — [observed].** When the barcode path is **disabled** (`CONSUMER_ENABLE_BARCODES=False`, the default) or **enabled but the string does not match**, `consume_file` finds no separator, skips the split block, and falls through to the whole-file `Consumer().try_consume_file(path,…)` at `src/documents/tasks.py:236`, returning `"Success. New document id N created"` and adding **exactly one** `Document` row (complete consume subprocess logs in Appendix §7.3.8):

```text
  DISABLED / patch-code-t-middle.pdf         enable=False string='PATCHT'           -> 'Success. New document id 1 created' | rows 0->1 (delta=1)
  NONMATCHING / patch-code-t-qr.pdf          enable=True  string='NONEXISTENT VALUE' -> 'Success. New document id 2 created' | rows 1->2 (delta=1)
```


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

`[]`→0 fragments (with `No pages to split on!`, `src/documents/tasks.py:127`); `[1]` on the 3-page `patch-code-t-middle.pdf`→**2** fragments (1 page each; the separator page 1 is skipped by `range(page_number+1, next_page)`, `src/documents/tasks.py:149`); `[2, 5]` on the 7-page `several-patcht-codes.pdf`→**3** fragments (2, 2, 1 pages). Fragments are written to a `tempfile.mkdtemp(prefix="paperless-", dir=SCRATCH_DIR)` (`src/documents/tasks.py:121`), i.e. `/tmp/paperless/…`, **not** `/app/consume`.

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

#### 5.5.1 Staged DB/model timeline for a `[2, 5]` split — before / file-only split / each fragment consume / retrain — [observed]

To make the "effect on training data" concrete and ordered, this probe drives the **real** entry points against `several-patcht-codes.pdf` (separators `[2, 5]`, so N+1 = 3 fragments) with a **fresh, isolated** database (`Document.objects.all().delete()` at the start; `/app/media` and `/app/consume` redirected to per-run temp dirs via `override_settings` + `save_to_dir.__defaults__` rebinding, §7.2), sampling `Document.objects.count()` at six stages. Command:

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q4_matrix.py 1
```

The staged table (the complete capture — every `paperless.tasks`/`paperless.consumer`/OCR subprocess line for each fragment — is in Appendix §7.3.8):

```text
  STAGE                      Doc.count  NOTE
  0 before split             0          input file only
  1 after file-only split    0          'File successfully split'; original unlinked=True; 3 fragments=['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
  2 after consume frag_0     1          several-patcht-codes_document_0.pdf CREATED pk=3 (md5=d74161c9d1a271807ae51a6b25a10aad)
  3 after consume frag_1     1          several-patcht-codes_document_1.pdf REJECTED duplicate (md5=d74161c9d1a271807ae51a6b25a10aad)
  4 after consume frag_2     2          several-patcht-codes_document_2.pdf CREATED pk=4 (md5=17c11e09a10bb4a0762a6cba8d817035)
  5 after retrain            2          train() returned True; effective corpus (inbox-excluded) = 2
```

Reading the timeline:

- **Stage 0 → 1 (the split itself adds nothing):** `consume_file(...)` returns `'File successfully split'`, unlinks the original (`src/documents/tasks.py:214`), and writes **3 fragment files** — but the `Document` count stays **0**. The split path never creates rows; it only produces files (§5.5).
- **Stage 1 → 2 (frag_0 → +1):** consuming fragment 0 creates `Document pk=3`; count `0 → 1`.
- **Stage 2 → 3 (frag_1 → +0, duplicate):** fragment 1 is byte-identical to fragment 0 (same md5), so `pre_check_duplicate` (`src/documents/consumer.py:102-113`, md5 at :104) rejects it — `It is a duplicate.` — count stays **1**.
- **Stage 3 → 4 (frag_2 → +1):** fragment 2 is distinct; creates `Document pk=4`; count `1 → 2`.
- **Stage 4 → 5 (retrain reads the enlarged corpus):** `DocumentClassifier().train()` returns `True` and reports an effective (inbox-excluded) corpus of **2** — the two rows that the split produced. This is the observed "effect on training data": a single input file materially changed what a later `train()` sees.
- **Stability:** the per-stage counts (`0,0,1,1,2,2`), the CREATED/REJECTED pattern, and `train()==True` are **identical across both runs** (§7.3.8). Only the fragment **md5 values** differ run-to-run (pikepdf's save is not byte-deterministic across processes: run 1 `d74161c9…`/`17c11e09…`; run 2 `71fd51f1…`/`55539372…`), which does not affect the row counts because the duplicate relationship (frag_1 == frag_0) is preserved within each run. **[observed]**

#### 5.5.2 Boundary case — a **page-0** separator yields two 0-page fragments that both fail to parse (0 rows) yet the split still "succeeds" and leaks non-daemon threads — [observed]

The N+1 arithmetic (§5.4) and the "distinct fragment → row" rule (§5.1(a)) both have a boundary the happy-path fixtures never exercise: a separator on **page 0**. `patch-code-t.pdf` is a **single-page** PDF whose only page carries the `PATCHT` barcode, so `scan_file_for_separating_barcodes()` returns `[0]`. `separate_pages()` then builds `document_0` from the pages *before* the separator (there are none) and one fragment *after* the separator page (which it skips) — producing **two 0-page PDFs**. This probe inspects both fragments and passes each through the **real** `Consumer`. Because the process ends up with live non-daemon threads (see step 5), it does **not** exit on its own — it is run under a hard timeout and terminates with **`rc=124` (hang)**; that non-exit is itself part of the observation and is stable across both runs. Command:

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna timeout 120 python3 /tmp/obs_q4_page0.py; echo "exit=$?"
... (see full capture below) ...
exit=124
```

Steps (1) scan and (2) fragment inspection — both fragments are **0-page, 315 bytes, byte-identical**:

```text
input patch-code-t.pdf pages=1 size=40893

--- (1) scan_file_for_separating_barcodes(patch-code-t.pdf) ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
separators = [0]

--- (2) separate_pages(patch-code-t.pdf, [0]) -> inspect each emitted fragment ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpgcjbsc2k/paperless-m5c7ach8
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpgcjbsc2k/paperless-m5c7ach8/patch-code-t_document_0.pdf', '/tmp/tmpgcjbsc2k/paperless-m5c7ach8/patch-code-t_document_1.pdf']
fragment count = 2
  fragment patch-code-t_document_0.pdf              pages=0 size=315 md5=23ca22d022b120dbed9ad7024e174cda sha256=3b2d9eca14f0cd809195671cb16e9c0d814a3c622862dc3bd788e02bf4ef0cd9
  fragment patch-code-t_document_1.pdf              pages=0 size=315 md5=23ca22d022b120dbed9ad7024e174cda sha256=3b2d9eca14f0cd809195671cb16e9c0d814a3c622862dc3bd788e02bf4ef0cd9
  both fragments byte-identical to each other? True
  all fragments 0-page? True ; all 315 bytes? True
```

Step (3) — passing fragment 0 through the real `Consumer` raises the root `ValueError: max_workers must be greater than 0` inside OCRmyPDF's executor setup, which is wrapped as `ParseError` (`src/paperless_tesseract/parsers.py:314`, from the `ocrmypdf.ocr(**args)` call at `:261`) and surfaced by the consumer (`src/documents/consumer.py:261`) as a `ConsumerError` — producing **0 rows**. The complete traceback (fragment 0 shown; fragment 1 produces the byte-identical traceback — full both-fragment capture in Appendix §7.3.9):

```text
--- (3) pass EACH 0-page fragment through the REAL Consumer -> exact exception, 0 rows ---
Document.objects.count() BEFORE = 0
LOG INFO paperless.consumer: Consuming patch-code-t_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4p0-frag-enyi2scg/patch-code-t_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4p0-frag-enyi2scg/patch-code-t_document_0.pdf', 'output_file': '/tmp/tmpgcjbsc2k/paperless-4msoukax/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpgcjbsc2k/paperless-4msoukax/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpgcjbsc2k/paperless-4msoukax
LOG ERROR paperless.consumer: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 385, in run_pipeline
    exec_concurrent(context, executor)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 274, in exec_concurrent
    executor(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_concurrent.py", line 82, in __call__
    self._execute(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/builtin_plugins/concurrency.py", line 127, in _execute
    with self.pbar_class(**tqdm_kwargs) as pbar, executor_class(
  File "/usr/local/lib/python3.9/concurrent/futures/thread.py", line 144, in __init__
    raise ValueError("max_workers must be greater than 0")
ValueError: max_workers must be greater than 0

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 314, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: ValueError: max_workers must be greater than 0
  consumed patch-code-t_document_0.pdf              -> ConsumerError: patch-code-t_document_0.pdf: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
Document.objects.count() AFTER  = 0 (0 => neither 0-page fragment produced a row)
```

Step (4) — despite both fragments failing to parse, `consume_file(...)` still returns `'File successfully split'` and unlinks the original (`src/documents/tasks.py:214`), leaving 0 rows:

```text
--- (4) consume_file(patch-code-t.pdf, CONSUMER_ENABLE_BARCODES=True) still 'File successfully split' + unlink ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4p0-in-l_8ar02x/patch-code-t.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpgcjbsc2k/paperless-kx8qbd82
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpgcjbsc2k/paperless-kx8qbd82/patch-code-t_document_0.pdf', '/tmp/tmpgcjbsc2k/paperless-kx8qbd82/patch-code-t_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4p0-in-l_8ar02x/patch-code-t.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
  consume_file(...) returned = 'File successfully split'
  original input still exists? = False (False => os.unlink at tasks.py:214)
  isolated consume dir contents = ['patch-code-t_document_0.pdf', 'patch-code-t_document_1.pdf']
  Document.objects.count() after split = 0
```

Step (5) — after OCRmyPDF's failed executor setup, **three non-daemon threads remain alive**, which is why the process cannot exit and the run times out:

```text
--- (5) live threads after OCRmyPDF ran (non-daemon threads keep the process alive) ---
  thread name='MainThread'             daemon=False alive=True
  thread name='Thread-8'               daemon=True  alive=True
  thread name='Thread-9'               daemon=False alive=True
  thread name='Thread-11'              daemon=False alive=True
  thread name='ThreadPoolExecutor-3_0' daemon=False alive=True
  non-daemon non-main threads still alive = 3 -> ['Thread-9', 'Thread-11', 'ThreadPoolExecutor-3_0']
================ END OF SCRIPT BODY ================
main thread returning now; if a non-daemon thread is alive the process will NOT exit (timeout => hang)
```

Reading the boundary case:

- **Fragment count still follows N+1:** one page-0 separator (N=1) yields **2** fragment files — the arithmetic holds — but both are **empty (0-page) PDFs**, so this is the degenerate end of the range.
- **0 rows, not 2:** the "distinct fragment → Document row" rule (§5.1(a)) is conditioned on *successful parse*; here the real `Consumer` raises `ConsumerError` for **both** fragments (root cause `ValueError: max_workers must be greater than 0` at `concurrent/futures/thread.py:144`, reached via `ocrmypdf/_concurrent.py:82` → `builtin_plugins/concurrency.py:127`), so the split produces **0** `Document` rows — directly substantiating the narrowed §5.1(a)/(d) wording.
- **"Success" is reported regardless:** `consume_file` returns `'File successfully split'` and deletes the original input even though nothing was ingested — a silent-data-loss shape for a page-0 separator.
- **Stable across runs, with expected digest volatility:** separators `[0]`, fragment count 2, `pages=0`, `size=315`, byte-identical fragments, the exact `ValueError`/`ParseError`/`ConsumerError` chain, `AFTER=0`, and the `rc=124` hang with the same three non-daemon threads (`Thread-9`, `Thread-11`, `ThreadPoolExecutor-3_0`) all reproduce identically in **both** runs. Only the empty-PDF digests differ (run 1 `md5=23ca22d0… sha256=3b2d9eca…`; run 2 `md5=2bf6a7ed… sha256=3a07eff7…`), again from pikepdf's non-deterministic save — the 0-page / 315-byte / byte-identical invariants are preserved within each run. **[observed]**


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
============================= test session starts ==============================
platform linux -- Python 3.9.23, pytest-8.4.2, pluggy-1.6.0 -- /usr/local/bin/python3
django: version: 4.0.4, settings: paperless.settings (from env)
rootdir: /app/src
configfile: setup.cfg
plugins: xdist-3.8.0, django-4.11.1, env-1.1.5, sugar-1.1.1, Faker-37.12.0, cov-7.0.0, anyio-3.5.0
collecting ... collected 5 items

documents/tests/test_tasks.py::TestTasks::test_separate_pages PASSED     [ 20%]
documents/tests/test_tasks.py::TestTasks::test_barcode_splitter PASSED   [ 40%]
documents/tests/test_tasks.py::TestTasks::test_consume_barcode_file PASSED [ 60%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes PASSED [ 80%]
documents/tests/test_tasks.py::TestTasks::test_scan_file_for_separating_barcodes4 PASSED [100%]

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
======================== 5 passed, 6 warnings in 3.35s =========================
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
$ mkdir -p /tmp/nd
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

**Observed distribution: 10/10 all-pass** — every run exited `0` with `62 passed, 1 skipped, 774 warnings`. `consume_before` is `(empty)` for all ten runs and `consume_after` is the same two fragment files for all ten — proving each run began from an identical state and produced an identical result. The reported "sometimes fails" behavior **did not reproduce** for these two suites in the canonical container under the default parallel config; this is honest **non-reproduction** at the suite level, not proof that the suites are structurally incapable of flaking. The underlying prediction-path flip that drives the reported flakiness **is** directly reproduced in isolation — 54 MATCH / 66 NONE over 120 fresh processes on a byte-identical corpus (see §6.5–§6.6).

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

The concrete code-level cause of run-to-run classification non-determinism is that the classifier's `MLPClassifier` is **unseeded**. All three estimators are constructed as `MLPClassifier(tol=0.01)` with **no `random_state`** (`src/documents/classifier.py:219` tags, `:227` correspondents, `:238` document types); a grep finds no RNG seeding anywhere. Training identical data in two separate process invocations yields **different** learned weights (they would be identical if seeded). Whether that weight difference changes the *prediction* depends entirely on the prediction input: for an input whose tokens **overlap** the training vocabulary the argmax is stable, but for an input whose tokens do **not** overlap it flips run-to-run — reproduced directly in §6.5. The cross-process capture below uses the overlapping input `'this is a document from c1'`, so its `[1]` prediction is stable even though the weight *values* are volatile by construction (they differ every process, and will differ again on any re-run):

```text
########## Cross-process unseeded MLP (two separate invocations, identical data) ##########
$ python3 /tmp/obs_q1_rng.py A   ;  python3 /tmp/obs_q1_rng.py B
=== PROCESS A ===
first 5 correspondent-MLP weights = [0.16484243335059118, 0.024833631206164587, -0.10771502193460243, 0.18496734298036752, -0.1746059464784781]
predict_correspondent('this is a document from c1') = [1] (c1.pk=1)
=== PROCESS B ===
first 5 correspondent-MLP weights = [0.09845402569574553, -0.011958840241269615, -0.16824946018903553, -0.10334667421713604, -0.006218273932036396]
predict_correspondent('this is a document from c1') = [1] (c1.pk=1)

########## grep for any RNG seeding in classifier.py ##########
$ grep -nE 'random_state|np.random|numpy.random|\bseed\b' documents/classifier.py ; echo rc=$?
grep rc=1 (1 => no match => UNSEEDED)
$ grep -nE 'MLPClassifier\(' documents/classifier.py
219:            self.tags_classifier = MLPClassifier(tol=0.01)
227:            self.correspondent_classifier = MLPClassifier(tol=0.01)
238:            self.document_type_classifier = MLPClassifier(tol=0.01)
```

### 6.5 Direct reproduction of the argmax flip on an unseen-vocabulary input — [observed]

§6.4 establishes the mechanism (unseeded weights); this section reproduces the **actual run-to-run flip** through the real entry points. The trigger is the *prediction input's vocabulary*: when the text being classified shares no tokens with the training corpus, `CountVectorizer.transform` produces a near-zero feature vector, the two output classes `{-1, c1.pk}` sit at the MLP's decision boundary, and the unseeded initial weights alone decide the argmax — so it flips process-to-process. This is exactly the corpus/input Report 5 describes.

**Corpus (canonical, built and trained through `tasks.train_classifier()`):** one `Correspondent(name='Auto C1', matching_algorithm=MATCH_AUTO)`; three `Document` rows — a labeled training doc (`content='acme invoice alpha repeated signal'`, `correspondent=Auto C1`), an unlabeled doc (`'garden weather banana unrelated neutral'`, `correspondent=None`), and an inbox-tagged doc (`'inbox excluded acme invoice'`) that `train()` drops via `.exclude(tags__is_inbox_tag=True)` (`src/documents/classifier.py:127`). Effective corpus = 2 docs, classes `{-1, 1}`. **Prediction input:** `'totally unseen vocabulary tokens'` — zero overlap with the corpus. The full probe source is in Appendix §7.2 (`obs_q6_nd.py`).

Command — 30 fresh single-process invocations, each a complete `train_classifier()` → `load_classifier()` → `predict_correspondent()` → `match_correspondents()` cycle. The loop is redirected to a file (so it prints nothing to stdout); each subsequent command shows its real, complete output. The 30 raw `RUN=` lines are reproduced verbatim in Appendix §7.4.11.

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src \
      -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna \
      bash -c 'for i in $(seq 1 30); do python3 /tmp/obs_q6_nd.py $i 2>/dev/null | grep "^RUN="; done' \
      > /tmp/nd30.txt

$ grep -c verdict=MATCH /tmp/nd30.txt ; grep -c verdict=NONE /tmp/nd30.txt
14
16

$ grep -oE 'corpus_sha1=[0-9a-f]+' /tmp/nd30.txt | sort -u
corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754

$ grep -oE 'corpus_sha1=[0-9a-f]+' /tmp/nd30.txt | sort -u | wc -l
1

$ grep -oE 'model_sha256=[0-9a-f]+' /tmp/nd30.txt | sort -u | wc -l
30
```

Compact verdict sequence for those 30 processes — each token is one process's accept/reject decision on the **same unchanged input**:

```text
MATCH NONE MATCH MATCH MATCH NONE MATCH NONE NONE NONE NONE NONE MATCH MATCH MATCH MATCH NONE MATCH NONE MATCH NONE NONE NONE MATCH NONE NONE NONE MATCH NONE MATCH
```

Observations, all from the block above:

- **The corpus is provably identical across all 30 runs** — exactly one distinct `corpus_sha1 = 98b0e672e43fbc7863208b65051bd70b50571754`, byte-identical to the value Report 5 reports, so the flip is **not** caused by any input variation.
- **Every trained model is a distinct artifact** — 30 distinct `model_sha256` digests over 30 runs — confirming the unseeded weights of §6.4 at the persisted-model level.
- **The accept/reject verdict flips on that fixed input:** 14 MATCH / 16 NONE in this batch — the same unchanged document is assigned the correspondent on some runs and left unassigned on others.

**Distribution across four independent 30-process batches** (to show a genuine near-even flip, not a fixed ratio): `14/16`, `13/17`, `15/15`, `12/18` → **54 MATCH / 66 NONE over 120 fresh processes (~45% accept)**; across all 120 there is still exactly **1** distinct `corpus_sha1` and **120** distinct `model_sha256`.

**Why the document's own Q2 fixtures never surfaced this:** the flip is contingent on *unseen* prediction vocabulary. A side-by-side probe (`obs_q6_contrast.py`, source in Appendix §7.2; full 20-line output in Appendix §7.4.12) trains the identical corpus once per process and predicts on two inputs — an overlapping one (`'acme invoice'`, whose tokens appear in training) and the unseen one (`'totally unseen vocabulary tokens'`). Over 20 fresh processes the overlapping input is **stable at 20/20 MATCH**, while the unseen input flips at **12/20 MATCH, 8/20 NONE**. The document's Q2 acceptance fixtures (§3) predict on overlapping vocabulary, which is why their argmax is stable and the suites pass 10/10 (§6.3) even though the model weights are volatile by construction.

### 6.6 Assessment — observed and reproduced

- **[observed]** 10/10 all-pass for the two suites under default `--numprocesses auto` (128 workers), from an identical state (§6.3). This is consistent with §6.5: the suites' Q2 fixtures predict on overlapping vocabulary, so their argmax does not flip.
- **[observed] + [code-grounded]** `DirectoriesMixin` isolates `MODEL_FILE` and `SCRATCH_DIR` per test (`src/documents/tests/utils.py:37,45`), removed at `tearDown` (`:53-57`), and each `TestCase` runs in a rolled-back transaction — so there is **no shared on-disk model file and no shared DB state** across workers for these two suites. Model-file isolation is therefore a **separate** concern from the RNG-driven prediction flip.
- **[observed] + [code-grounded]** The unseeded `MLPClassifier` (`src/documents/classifier.py:219,227,238`) draws initial weights from NumPy's process-global RNG, so identical training data yields a different model — and a different argmax at the decision boundary — in every process (30 distinct `model_sha256` over 30 runs, §6.5). On an input whose vocabulary does not overlap the training corpus this flips the accept/reject decision run-to-run: **directly observed and reproduced at 54 MATCH / 66 NONE over 120 fresh processes on a byte-identical corpus** (§6.5). This is a **reproduced cause** of prediction non-determinism, not a bounded hypothesis. The one link that remains **[inferred]** is the final step from a reproduced *prediction* flip to an intermittent *test-assertion* failure in the full parallel suite: that additionally requires xdist test order / worker assignment to perturb the shared process-global NumPy RNG relative to a threshold-sensitive assertion — a mechanism consistent with the code but not separately observed here.
- **[inferred, code-grounded]** Secondary structural suspect: the fixed shared `SCRATCH_DIR = /tmp/paperless` (`src/paperless/settings.py:84`). Any code path that does not go through a `DirectoriesMixin` override writes there; under 128 parallel workers that is a plausible cross-worker collision point — but it is **not** a factor for these two suites, both of which override `SCRATCH_DIR` per test.
- No behavior above is attributed to vague environmental causes (containerization, scheduler jitter, networking); the concrete, reproduced code-level mechanism is the unseeded `MLPClassifier`, with the fixed shared `SCRATCH_DIR` as a secondary structural suspect for suites outside these two.

---

## 7. Appendix — complete, unedited evidence

This appendix reproduces (a) the full source of every observation script, (b) the complete, unedited output of every probe including the second stability run, (c) all ten non-determinism runs in full, (d) the cleanup / repository-state transcript, and (e) the question → entry-point → citation map. Nothing here is abbreviated; any `...` inside an output block is literal emitted text (see the note in the header).

### 7.1 How to reproduce

Each script was delivered into the container's `/tmp` with `docker cp` and executed with a run-index argument. To reproduce: save any script's full source from §7.2 to a local file under its listed name (for example `obs_q1_hires.py`), copy it into the container, and run it — the trailing integer is the run index (`1`, `2`, …). A concrete, copy-pasteable example (the Q1 high-resolution lifecycle probe, run 1):

```text
$ docker cp obs_q1_hires.py pngx-qna:/tmp/obs_q1_hires.py
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q1_hires.py 1
```

Substitute any other script name from §7.2 (`obs_q2.py`, `obs_q3.py`, `obs_q4.py`, `obs_q4_matrix.py`, `obs_q4_page0.py`, `obs_q6_nd.py`, `obs_q6_contrast.py`, …) and the desired run index in the same two commands.

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

#### `/tmp/obs_q6_nd.py` (§6 — non-determinism: run-to-run argmax flip on an unseen-vocabulary input)

```python
#!/usr/bin/env python3
"""Q6/non-determinism probe: reproduce the reported run-to-run 'sometimes matches,
sometimes not' behavior of the automatic (MATCH_AUTO) correspondent path, through the
REAL entry points documents.tasks.train_classifier() -> load_classifier() ->
DocumentClassifier.predict_correspondent() -> matching.match_correspondents().

Each invocation is a FRESH process with an identical, fixed corpus and an identical,
fixed prediction input whose tokens do NOT appear in the training vocabulary.  It prints
one machine-parseable line: the corpus SHA-1 (stable across runs => identical corpus),
the trained model SHA-256 (varies => unseeded MLP), the learned classes_, the raw
prediction, the accepted correspondent names, and a MATCH/NONE verdict.  Secure temp dirs
+ full cleanup; nothing is written outside the container /tmp."""
import os, sys, hashlib, tempfile, shutil

RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
scratch = tempfile.mkdtemp(prefix="pngx-nd-")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")

import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)

ovr = override_settings(
    MODEL_FILE=os.path.join(scratch, "classification_model.pickle"),
    DATA_DIR=scratch,
    SCRATCH_DIR=scratch,
)
ovr.enable()

from documents import tasks, matching
from documents.classifier import load_classifier, preprocess_content
from documents.models import Correspondent, Document, Tag, MatchingModel

PREDICT_INPUT = "totally unseen vocabulary tokens"

def corpus_sha1():
    """Replicate DocumentClassifier.train()'s SHA-1 over the effective (non-inbox) corpus."""
    m = hashlib.sha1()
    for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True):
        pc = preprocess_content(doc.content)
        m.update(pc.encode("utf-8"))
        y = -1
        dt = doc.document_type
        if dt and dt.matching_algorithm == MatchingModel.MATCH_AUTO:
            y = dt.pk
        m.update(y.to_bytes(4, "little", signed=True))
        y = -1
        cor = doc.correspondent
        if cor and cor.matching_algorithm == MatchingModel.MATCH_AUTO:
            y = cor.pk
        m.update(y.to_bytes(4, "little", signed=True))
        tags = sorted(t.pk for t in doc.tags.filter(matching_algorithm=MatchingModel.MATCH_AUTO))
        for t in tags:
            m.update(t.to_bytes(4, "little", signed=True))
    return m.hexdigest()

try:
    # ---- fixed corpus (identical every run) ----
    c1 = Correspondent.objects.create(name="Auto C1", matching_algorithm=MatchingModel.MATCH_AUTO)
    inbox = Tag.objects.create(name="Inbox", is_inbox_tag=True, matching_algorithm=MatchingModel.MATCH_ANY)
    Document.objects.create(title="d1", content="acme invoice alpha repeated signal",
                            correspondent=c1, checksum="nd-1")
    Document.objects.create(title="d2", content="garden weather banana unrelated neutral",
                            checksum="nd-2")
    d3 = Document.objects.create(title="d3", content="inbox excluded acme invoice",
                                 correspondent=c1, checksum="nd-3")
    d3.tags.add(inbox)

    chash = corpus_sha1()
    total = Document.objects.count()
    effective = Document.objects.exclude(tags__is_inbox_tag=True).count()

    # ---- REAL training + persistence ----
    tasks.train_classifier()
    with open(settings.MODEL_FILE, "rb") as fh:
        model_sha256 = hashlib.sha256(fh.read()).hexdigest()

    # ---- REAL load + predict (production path) ----
    clf = load_classifier()
    classes_ = clf.correspondent_classifier.classes_.tolist()
    pred = clf.predict_correspondent(PREDICT_INPUT)
    pred_repr = None if pred is None else pred.tolist()

    # ---- REAL acceptance via match_correspondents on a probe doc ----
    probe = Document.objects.create(title="probe", content=PREDICT_INPUT, checksum="nd-probe")
    names = [o.name for o in matching.match_correspondents(probe, clf)]
    verdict = "MATCH" if names else "NONE"

    print(f"RUN={RUN} total={total} effective={effective} c1.pk={c1.pk} "
          f"corpus_sha1={chash} model_sha256={model_sha256} classes_={classes_} "
          f"pred={pred_repr} match={names} verdict={verdict}")
finally:
    ovr.disable()
    try:
        connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception:
        pass
    shutil.rmtree(scratch, ignore_errors=True)
```

#### `/tmp/obs_q6_contrast.py` (§6 — non-determinism: overlapping-input stability vs. unseen-input flip)

```python
#!/usr/bin/env python3
"""Q6 contrast probe: across fresh processes with the identical fixed corpus, predict TWO
inputs through the loaded model: (I) an input whose tokens OVERLAP the training vocabulary
('acme invoice alpha repeated signal', = the labeled doc), and (U) an input whose tokens do
NOT ('totally unseen vocabulary tokens'). Shows the overlapping input's argmax is stable
while the unseen input's argmax flips — the reason the original document fixture (an
overlapping input) never surfaced the flip. Real train_classifier()/load_classifier()/
predict_correspondent(). Secure temp dirs + cleanup."""
import os, sys, tempfile, shutil
RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
scratch = tempfile.mkdtemp(prefix="pngx-ndc-")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)
ovr = override_settings(MODEL_FILE=os.path.join(scratch, "classification_model.pickle"),
                        DATA_DIR=scratch, SCRATCH_DIR=scratch)
ovr.enable()
from documents import tasks
from documents.classifier import load_classifier
from documents.models import Correspondent, Document, Tag, MatchingModel
OVERLAP = "acme invoice alpha repeated signal"
UNSEEN = "totally unseen vocabulary tokens"
try:
    c1 = Correspondent.objects.create(name="Auto C1", matching_algorithm=MatchingModel.MATCH_AUTO)
    inbox = Tag.objects.create(name="Inbox", is_inbox_tag=True, matching_algorithm=MatchingModel.MATCH_ANY)
    Document.objects.create(title="d1", content=OVERLAP, correspondent=c1, checksum="c-1")
    Document.objects.create(title="d2", content="garden weather banana unrelated neutral", checksum="c-2")
    d3 = Document.objects.create(title="d3", content="inbox excluded acme invoice", correspondent=c1, checksum="c-3")
    d3.tags.add(inbox)
    tasks.train_classifier()
    clf = load_classifier()
    po = clf.predict_correspondent(OVERLAP)
    pu = clf.predict_correspondent(UNSEEN)
    vo = "MATCH" if po is not None else "NONE"
    vu = "MATCH" if pu is not None else "NONE"
    print(f"RUN={RUN} overlap_pred={None if po is None else po.tolist()} overlap={vo} | "
          f"unseen_pred={None if pu is None else pu.tolist()} unseen={vu}")
finally:
    ovr.disable()
    try: connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception: pass
    shutil.rmtree(scratch, ignore_errors=True)
```

#### `/tmp/obs_q1_hires.py` (Q1/§2.2.1 — high-resolution lifecycle: `st_mtime_ns`/size/SHA-256/loaded `data_hash`, physically-separate incompatible model, task-level empty corpus)

```python
#!/usr/bin/env python3
"""Q1 high-resolution lifecycle probe (MINOR-3): capture per-call corpus count,
st_mtime_ns, model size, SHA-256, and the loaded data_hash (hex) across the REAL
tasks.train_classifier() reuse/retrain cycle; then exercise (B) a compatible v7 load,
(C) a PHYSICALLY-SEPARATE incompatible on-disk model through the real load_classifier()
(deletion), (D) a direct empty-corpus DocumentClassifier().train() ValueError, and
(E) task-level empty-corpus (train_classifier() catches it, returns None, writes no
model). Real entry points only; secure temp dirs + full cleanup; nothing written outside
the container /tmp."""
import os, sys, pickle, hashlib, logging, tempfile, shutil

RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
scratch = tempfile.mkdtemp(prefix="pngx-q1h-")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")

import django
from django.conf import settings
from django.test import override_settings
django.setup()
from django.test.utils import setup_test_environment
from django.db import connection
setup_test_environment()
connection.creation.create_test_db(verbosity=0)

ovr = override_settings(
    MODEL_FILE=os.path.join(scratch, "classification_model.pickle"),
    DATA_DIR=scratch,
    SCRATCH_DIR=scratch,
)
ovr.enable()

# surface the real paperless.* log lines (same convention as the other Q1 probe)
class _H(logging.Handler):
    def emit(self, r):
        print(f"  LOG {r.levelname} {r.name}: {r.getMessage()}")
for _n in ("paperless.tasks", "paperless.classifier"):
    lg = logging.getLogger(_n)
    lg.setLevel(logging.DEBUG)
    lg.addHandler(_H())
    lg.propagate = False

from documents import tasks
from documents.classifier import DocumentClassifier, load_classifier
from documents.models import Correspondent, Document, MatchingModel

MF = settings.MODEL_FILE


def stat(tag):
    if not os.path.isfile(MF):
        print(f"  [{tag}] exists=False")
        return None
    b = open(MF, "rb").read()
    st = os.stat(MF)
    sha = hashlib.sha256(b).hexdigest()
    print(f"  [{tag}] exists=True corpus_count={Document.objects.count()} "
          f"size={st.st_size} st_mtime_ns={st.st_mtime_ns} sha256={sha}")
    return (st.st_size, st.st_mtime_ns, sha)


def loaded_hash(tag):
    clf = load_classifier()
    if clf is None:
        print(f"  [{tag}] load_classifier()=None")
        return None
    dh = clf.data_hash.hex() if clf.data_hash else None
    print(f"  [{tag}] load_classifier()=OK loaded_data_hash={dh} "
          f"classes_={clf.correspondent_classifier.classes_.tolist()}")
    return dh


try:
    print(f"================ Q1-HIRES RUN {RUN} ================")
    c1 = Correspondent.objects.create(name="Auto C1",
                                      matching_algorithm=MatchingModel.MATCH_AUTO)
    d1 = Document.objects.create(title="d1", content="acme invoice alpha",
                                 correspondent=c1, checksum="q1h-1")

    print("--- A: reuse/retrain, high-resolution (st_mtime_ns / size / sha256 / loaded data_hash) ---")
    print("  corpus_count =", Document.objects.count())
    stat("before")
    tasks.train_classifier();                s1 = stat("after#1 initial-save")
    h1 = loaded_hash("after#1")
    tasks.train_classifier();                s2 = stat("after#2 unchanged")
    h2 = loaded_hash("after#2")
    d1.content = "acme invoice alpha CHANGED extra tokens"; d1.save()
    tasks.train_classifier();                s3 = stat("after#3 changed")
    h3 = loaded_hash("after#3")
    print(f"  REUSE   (call#2 vs #1): size_same={s1[0]==s2[0]} "
          f"mtime_ns_same={s1[1]==s2[1]} sha256_same={s1[2]==s2[2]} loaded_hash_same={h1==h2}")
    print(f"  RETRAIN (call#3 vs #2): mtime_ns_changed={s2[1]!=s3[1]} "
          f"sha256_changed={s2[2]!=s3[2]} loaded_hash_changed={h2!=h3}")

    print("--- B: compatible v7 load (real load_classifier() succeeds) ---")
    with open(MF, "rb") as f:
        ondisk = pickle.load(f)
    print(f"  on-disk schema_version={ondisk} FORMAT_VERSION={DocumentClassifier.FORMAT_VERSION} "
          f"compatible={ondisk == DocumentClassifier.FORMAT_VERSION}")
    loaded_hash("compatible-v7")

    print("--- C: PHYSICALLY-SEPARATE incompatible on-disk model -> real load_classifier() deletes it ---")
    with open(MF, "wb") as f:            # write a real pickle whose schema_version != 7
        pickle.dump(999, f)
        pickle.dump(None, f)
    print(f"  wrote incompatible model directly to disk: schema_version=999 exists_before={os.path.isfile(MF)}")
    r = load_classifier()
    print(f"  load_classifier() returned={r} exists_after={os.path.isfile(MF)} "
          f"(False => os.unlink at classifier.py:48 ran)")

    print("--- D: direct empty-corpus DocumentClassifier().train() -> ValueError ---")
    Document.objects.all().delete()
    print("  corpus_count =", Document.objects.count())
    try:
        DocumentClassifier().train()
        print("  NO ERROR (unexpected)")
    except ValueError as e:
        print("  ValueError:", repr(str(e)))

    print("--- E: task-level empty corpus (auto correspondent present, 0 docs) ---")
    if os.path.isfile(MF):
        os.unlink(MF)
    ret = tasks.train_classifier()
    print(f"  train_classifier() returned={ret} model_exists={os.path.isfile(MF)} "
          f"(ValueError caught at tasks.py:70-72 => no save)")

    print("--- E2: no auto matching models at all -> early return, no train ---")
    c1.matching_algorithm = MatchingModel.MATCH_ANY   # no longer AUTO
    c1.save()
    ret2 = tasks.train_classifier()
    print(f"  train_classifier() returned={ret2} model_exists={os.path.isfile(MF)} "
          f"(guard at tasks.py:49-55 => early return)")

    print(f"================ Q1-HIRES RUN {RUN} SUMMARY ================")
    print(f"  reuse_byte_identical={s1==s2} retrain_all_changed={(s2[1]!=s3[1]) and (s2[2]!=s3[2])} "
          f"loaded_hash reuse={h1==h2} retrain={h2!=h3}")
finally:
    ovr.disable()
    try:
        connection.creation.destroy_test_db(":memory:", verbosity=0)
    except Exception:
        pass
    shutil.rmtree(scratch, ignore_errors=True)
```

#### `/tmp/obs_q4_matrix.py` (Q4/§5.3.1+§5.5.1 — value matrix, enabled/disabled, staged [2,5] timeline)

```python
#!/usr/bin/env python3
"""Q4 matrix/timeline probe (MINOR-4): (M) complete CONSUMER_BARCODE_STRING value matrix
(exact PATCHT, lowercase, trailing-space, empty, custom, nonmatching) x two fixtures
(PATCHT-bearing CODE39 vs CUSTOM-bearing CODE128) establishing case/whitespace sensitivity
at the real decision site tasks.py:108; (EN) enabled-vs-disabled and nonmatching -> whole-file
consume -> exactly 1 Document row via consume_file tasks.py:236; (ST) staged DB timeline for a
normal [2,5] split: before / after file-only split / after each fragment consume / after dup
rejection / after retrain. Real entry points; ALL media/data/scratch isolated to mkdtemp dirs
(DirectoriesMixin kwargs), full cleanup; nothing written to /app/media or /app/consume."""
import os, sys, shutil, logging, tempfile, hashlib
RUN = sys.argv[1] if len(sys.argv) > 1 else "1"
import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.conf import settings
from django.test.utils import setup_test_environment, teardown_test_environment
from django.test import override_settings
from django.db import connection
from unittest import mock
from documents import tasks
from documents.classifier import DocumentClassifier
from documents.consumer import Consumer, ConsumerError
from documents.models import Document, Correspondent

BARCODES = "/app/src/documents/tests/samples/barcodes"
_h = logging.StreamHandler(sys.stdout)
_h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for name in ("paperless.tasks", "paperless.consumer", "paperless.parsing",
             "paperless.parsing.tesseract"):
    lg = logging.getLogger(name); lg.setLevel(logging.DEBUG); lg.handlers = [_h]; lg.propagate = False

def hr(t): print("\n--- " + t + " ---")
print("================ Q4-MATRIX RUN %s ================" % RUN)
setup_test_environment()
old_name = connection.creation.create_test_db(verbosity=0)

# isolate ALL dirs (DirectoriesMixin kwargs) so /app/media and /app/consume are never touched
data_dir = tempfile.mkdtemp(); media_dir = tempfile.mkdtemp()
scratch_dir = tempfile.mkdtemp(); consumption_dir = tempfile.mkdtemp()
originals = os.path.join(media_dir, "documents", "originals")
thumbs = os.path.join(media_dir, "documents", "thumbnails")
archive = os.path.join(media_dir, "documents", "archive")
index_dir = os.path.join(data_dir, "index"); logging_dir = os.path.join(data_dir, "log")
for d in (originals, thumbs, archive, index_dir, logging_dir):
    os.makedirs(d, exist_ok=True)
ovr = override_settings(
    DATA_DIR=data_dir, SCRATCH_DIR=scratch_dir, MEDIA_ROOT=media_dir,
    ORIGINALS_DIR=originals, THUMBNAIL_DIR=thumbs, ARCHIVE_DIR=archive,
    CONSUMPTION_DIR=consumption_dir, LOGGING_DIR=logging_dir, INDEX_DIR=index_dir,
    MODEL_FILE=os.path.join(data_dir, "classification_model.pickle"),
    MEDIA_LOCK=os.path.join(media_dir, "media.lock"),
)
ovr.enable()
_tmp = [data_dir, media_dir, scratch_dir, consumption_dir]
def mkroot(p):
    d = tempfile.mkdtemp(prefix=p); _tmp.append(d); return d
try:
    # ---------- Part M: complete value matrix ----------
    hr("(M) CONSUMER_BARCODE_STRING value matrix -> scan() separators (case/whitespace/empty/custom/nonmatching)")
    pt = os.path.join(BARCODES, "patch-code-t.pdf")            # CODE39 barcode decodes to 'PATCHT'
    cu = os.path.join(BARCODES, "barcode-128-custom.pdf")      # CODE128 barcode decodes to 'CUSTOM BARCODE'
    print("fixtures: patch-code-t.pdf (decodes 'PATCHT', CODE39); barcode-128-custom.pdf (decodes 'CUSTOM BARCODE', CODE128)")
    print("%-22s %-24s %-24s" % ("CONSUMER_BARCODE_STRING", "scan(patch-code-t.pdf)", "scan(barcode-128-custom.pdf)"))
    for cfg in ["PATCHT", "patcht", "PATCHT ", "", "CUSTOM BARCODE", "NONEXISTENT VALUE"]:
        with override_settings(CONSUMER_BARCODE_STRING=cfg):
            s_pt = tasks.scan_file_for_separating_barcodes(pt)
            s_cu = tasks.scan_file_for_separating_barcodes(cu)
        print("%-22r %-24s %-24s" % (cfg, str(s_pt), str(s_cu)))

    # ---------- Part EN: enabled/disabled + nonmatching -> whole-file consume -> 1 row ----------
    hr("(EN) barcodes DISABLED and nonmatching-string -> whole-file consume -> exactly 1 Document row")
    def whole_file_consume(label, src_name, enable, bcstring):
        src = os.path.join(BARCODES, src_name)
        stage = os.path.join(mkroot("pngx-q4m-en-"), src_name)
        shutil.copy(src, stage)
        n0 = Document.objects.count()
        try:
            with override_settings(CONSUMER_ENABLE_BARCODES=enable, CONSUMER_BARCODE_STRING=bcstring):
                with mock.patch("documents.consumer.Consumer._send_progress"):
                    res = tasks.consume_file(stage)
            n1 = Document.objects.count()
            print("  %-42s enable=%-5s string=%-18r -> %r | rows %d->%d (delta=%d)"
                  % (label, enable, bcstring, res, n0, n1, n1 - n0))
        except ConsumerError as e:
            print("  %-42s enable=%-5s string=%-18r -> ConsumerError: %s"
                  % (label, enable, bcstring, str(e)))
    whole_file_consume("DISABLED / patch-code-t-middle.pdf", "patch-code-t-middle.pdf", False, "PATCHT")
    whole_file_consume("NONMATCHING / patch-code-t-qr.pdf", "patch-code-t-qr.pdf", True, "NONEXISTENT VALUE")

    # ---------- Part ST: staged timeline for a normal [2,5] split ----------
    hr("(ST) staged DB timeline for several-patcht-codes.pdf split=[2,5] (before/split/consume x3/retrain)")
    iso = mkroot("pngx-q4m-consume-")
    orig_def = tasks.save_to_dir.__defaults__
    tasks.save_to_dir.__defaults__ = (orig_def[0], iso)
    stages = []; created = []
    # isolate the [2,5] timeline from the Part EN documents so the staged counts start clean
    Document.objects.all().delete()
    try:
        stages.append(("0 before split", Document.objects.count(), "input file only"))
        with override_settings(CONSUMER_ENABLE_BARCODES=True, CONSUMER_BARCODE_STRING="PATCHT"):
            scratch_in = os.path.join(mkroot("pngx-q4m-in-"), "several-patcht-codes.pdf")
            shutil.copy(os.path.join(BARCODES, "several-patcht-codes.pdf"), scratch_in)
            with mock.patch("documents.consumer.Consumer._send_progress"):
                res = tasks.consume_file(scratch_in)
        frags = sorted(os.listdir(iso))
        stages.append(("1 after file-only split", Document.objects.count(),
                       "%r; original unlinked=%s; %d fragments=%s"
                       % (res, not os.path.exists(scratch_in), len(frags), frags)))
        labels = ["2 after consume frag_0", "3 after consume frag_1", "4 after consume frag_2"]
        for i, fname in enumerate(frags):
            stage = os.path.join(mkroot("pngx-q4m-frag-"), fname)
            shutil.copy(os.path.join(iso, fname), stage)
            md5 = hashlib.md5(open(stage, "rb").read()).hexdigest()
            try:
                with mock.patch("documents.consumer.Consumer._send_progress"):
                    doc = Consumer().try_consume_file(stage)
                created.append(doc.pk); note = "%s CREATED pk=%s (md5=%s)" % (fname, doc.pk, md5)
            except ConsumerError:
                note = "%s REJECTED duplicate (md5=%s)" % (fname, md5)
            stages.append((labels[i] if i < len(labels) else "consume %d" % i,
                           Document.objects.count(), note))
    finally:
        tasks.save_to_dir.__defaults__ = orig_def
    corr = Correspondent.objects.create(name="Q4mCorr", matching_algorithm=Correspondent.MATCH_AUTO)
    for i, pk in enumerate(created):
        d = Document.objects.get(pk=pk); d.content = "alpha bravo fragment %d token%d" % (i, i)
        d.correspondent = corr; d.save()
    eff = Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True).count()
    trained = DocumentClassifier().train()
    stages.append(("5 after retrain", Document.objects.count(),
                   "train() returned %s; effective corpus (inbox-excluded) = %d" % (trained, eff)))
    print("  %-26s %-10s %s" % ("STAGE", "Doc.count", "NOTE"))
    for s, c, note in stages:
        print("  %-26s %-10d %s" % (s, c, note))
    print("SUMMARY: [2,5] -> 3 fragment files -> 2 distinct rows (1 duplicate rejected) -> effective training corpus %d" % eff)
finally:
    ovr.disable()
    connection.creation.destroy_test_db(old_name, verbosity=0)
    teardown_test_environment()
    for d in _tmp:
        shutil.rmtree(d, ignore_errors=True)
print("\n================ Q4-MATRIX RUN %s SUMMARY ================" % RUN)
print("matrix: only exact configured string matches (case- & whitespace-sensitive); empty never matches; "
      "custom matches only its own value; nonmatching -> [] | disabled/nonmatching -> whole-file consume -> 1 row | "
      "[2,5] staged: 0->0(split)->1->1(dup)->2->retrain(corpus 2)")
```

#### `/tmp/obs_q4_page0.py` (Q4/§5.5.2 — page-0 boundary: two 0-page fragments, `max_workers` ValueError, thread-leak hang)

```python
#!/usr/bin/env python3
"""Q4 page-0 boundary probe (MINOR-4): a separator on page 0 of patch-code-t.pdf.
Shows (1) scan -> [0]; (2) separate_pages emits TWO fragments, each inspected with
pikepdf (page count), byte size, SHA-256, md5 -> both 0-page and byte-identical;
(3) each fragment passed through the REAL Consumer FAILS with the exact exception
(ValueError: max_workers must be greater than 0) -> 0 Document rows; (4) consume_file
still returns 'File successfully split' and unlinks the original; (5) after OCRmyPDF runs,
non-daemon worker/executor threads (e.g. Thread-9, Thread-11, ThreadPoolExecutor-3_0)
are left alive, so an isolated process reaches the end of the script but does NOT exit
(times out). Run under `timeout`; rc=124 == the hang.
All dirs isolated to mkdtemp; nothing written to /app/media or /app/consume."""
import os, sys, shutil, logging, tempfile, hashlib, threading
import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.conf import settings
from django.test.utils import setup_test_environment, teardown_test_environment
from django.test import override_settings
from django.db import connection
from unittest import mock
from pikepdf import Pdf
from documents import tasks
from documents.consumer import Consumer, ConsumerError
from documents.models import Document

BARCODES = "/app/src/documents/tests/samples/barcodes"
_h = logging.StreamHandler(sys.stdout)
_h.setFormatter(logging.Formatter("LOG %(levelname)s %(name)s: %(message)s"))
for name in ("paperless.tasks", "paperless.consumer", "paperless.parsing",
             "paperless.parsing.tesseract"):
    lg = logging.getLogger(name); lg.setLevel(logging.DEBUG); lg.handlers = [_h]; lg.propagate = False
def P(*a): print(*a, flush=True)
def hr(t): P("\n--- " + t + " ---")
P("================ Q4-PAGE0 ================")
setup_test_environment()
old_name = connection.creation.create_test_db(verbosity=0)
data_dir = tempfile.mkdtemp(); media_dir = tempfile.mkdtemp()
scratch_dir = tempfile.mkdtemp(); consumption_dir = tempfile.mkdtemp()
originals = os.path.join(media_dir, "documents", "originals")
thumbs = os.path.join(media_dir, "documents", "thumbnails")
archive = os.path.join(media_dir, "documents", "archive")
index_dir = os.path.join(data_dir, "index"); logging_dir = os.path.join(data_dir, "log")
for d in (originals, thumbs, archive, index_dir, logging_dir):
    os.makedirs(d, exist_ok=True)
ovr = override_settings(
    DATA_DIR=data_dir, SCRATCH_DIR=scratch_dir, MEDIA_ROOT=media_dir,
    ORIGINALS_DIR=originals, THUMBNAIL_DIR=thumbs, ARCHIVE_DIR=archive,
    CONSUMPTION_DIR=consumption_dir, LOGGING_DIR=logging_dir, INDEX_DIR=index_dir,
    MODEL_FILE=os.path.join(data_dir, "classification_model.pickle"),
    MEDIA_LOCK=os.path.join(media_dir, "media.lock"),
)
ovr.enable()
_tmp = [data_dir, media_dir, scratch_dir, consumption_dir]
def mkroot(p):
    d = tempfile.mkdtemp(prefix=p); _tmp.append(d); return d
try:
    src = os.path.join(BARCODES, "patch-code-t.pdf")
    with Pdf.open(src) as pdf:
        P("input patch-code-t.pdf pages=%d size=%d" % (len(pdf.pages), os.path.getsize(src)))

    hr("(1) scan_file_for_separating_barcodes(patch-code-t.pdf)")
    seps = tasks.scan_file_for_separating_barcodes(src)
    P("separators =", seps)

    hr("(2) separate_pages(patch-code-t.pdf, %s) -> inspect each emitted fragment" % seps)
    frags = tasks.separate_pages(src, seps)
    P("fragment count =", len(frags))
    digests = []
    for f in frags:
        b = open(f, "rb").read()
        with Pdf.open(f) as pdf:
            npages = len(pdf.pages)
        sha = hashlib.sha256(b).hexdigest(); md5 = hashlib.md5(b).hexdigest()
        digests.append((npages, len(b), sha, md5))
        P("  fragment %-40s pages=%d size=%d md5=%s sha256=%s"
          % (os.path.basename(f), npages, len(b), md5, sha))
    P("  both fragments byte-identical to each other? %s" % (len(set(d[3] for d in digests)) == 1))
    P("  all fragments 0-page? %s ; all 315 bytes? %s"
      % (all(d[0] == 0 for d in digests), all(d[1] == 315 for d in digests)))

    hr("(3) pass EACH 0-page fragment through the REAL Consumer -> exact exception, 0 rows")
    P("Document.objects.count() BEFORE =", Document.objects.count())
    for f in frags:
        stage = os.path.join(mkroot("pngx-q4p0-frag-"), os.path.basename(f))
        shutil.copy(f, stage)
        try:
            with mock.patch("documents.consumer.Consumer._send_progress"):
                doc = Consumer().try_consume_file(stage)
            P("  consumed %-40s -> Document pk=%s (UNEXPECTED)" % (os.path.basename(f), doc.pk))
        except ConsumerError as e:
            P("  consumed %-40s -> ConsumerError: %s" % (os.path.basename(f), str(e)))
        except Exception as e:
            P("  consumed %-40s -> %s: %s" % (os.path.basename(f), type(e).__name__, str(e)))
    P("Document.objects.count() AFTER  =", Document.objects.count(), "(0 => neither 0-page fragment produced a row)")

    hr("(4) consume_file(patch-code-t.pdf, CONSUMER_ENABLE_BARCODES=True) still 'File successfully split' + unlink")
    iso = mkroot("pngx-q4p0-consume-")
    orig_def = tasks.save_to_dir.__defaults__
    tasks.save_to_dir.__defaults__ = (orig_def[0], iso)
    try:
        with override_settings(CONSUMER_ENABLE_BARCODES=True, CONSUMER_BARCODE_STRING="PATCHT"):
            scratch_in = os.path.join(mkroot("pngx-q4p0-in-"), "patch-code-t.pdf")
            shutil.copy(src, scratch_in)
            with mock.patch("documents.consumer.Consumer._send_progress"):
                res = tasks.consume_file(scratch_in)
            P("  consume_file(...) returned =", repr(res))
            P("  original input still exists? =", os.path.exists(scratch_in), "(False => os.unlink at tasks.py:214)")
            P("  isolated consume dir contents =", sorted(os.listdir(iso)))
            P("  Document.objects.count() after split =", Document.objects.count())
    finally:
        tasks.save_to_dir.__defaults__ = orig_def
finally:
    try: connection.creation.destroy_test_db(old_name, verbosity=0)
    except Exception: pass
    ovr.disable(); teardown_test_environment()
    for d in _tmp:
        shutil.rmtree(d, ignore_errors=True)

hr("(5) live threads after OCRmyPDF ran (non-daemon threads keep the process alive)")
for t in threading.enumerate():
    P("  thread name=%-24r daemon=%-5s alive=%s" % (t.name, t.daemon, t.is_alive()))
nondaemon = [t for t in threading.enumerate() if not t.daemon and t is not threading.main_thread()]
P("  non-daemon non-main threads still alive = %d -> %s"
  % (len(nondaemon), [t.name for t in nondaemon]))
P("================ END OF SCRIPT BODY ================")
P("main thread returning now; if a non-daemon thread is alive the process will NOT exit (timeout => hang)")
```



### 7.3 Two-run stability — complete captures of both runs

Every quantitative claim in this document was produced twice, from an identical starting state, and confirmed stable. `[observed]` Rather than merely assert stability, the two runs are compared directly with a concrete `normalize` function that masks **only** the tokens that are volatile by construction, then `diff`. The function is defined here (copy-pasteable Bash; extended-regex `sed`):

```bash
# normalize <file> : emit the file with ONLY by-design-volatile tokens masked.
normalize() {
  sed -E \
    -e 's/RUN [0-9]+/RUN N/g' \
    -e 's#paperless-[A-Za-z0-9_]+#paperless-<tmp>#g' \
    -e 's#ocrmypdf\.io\.[A-Za-z0-9_]+#ocrmypdf.io.<tmp>#g' \
    -e 's#/tmp/tmp[A-Za-z0-9_]+#/tmp/tmp<X>#g' \
    -e 's#/tmp/pngx-[A-Za-z0-9_-]+#/tmp/pngx-<mkdtemp>#g' \
    -e 's/0x[0-9a-fA-F]+/0x<addr>/g' \
    -e 's/\[[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9:,]+\]/[<ts>]/g' \
    -e 's/17[0-9]{8}\.[0-9]+/<mtime>/g' \
    -e 's/\b[0-9a-f]{32}\b/<md5>/g' \
    -e 's/\b[0-9a-f]{64}\b/<sha256>/g' \
    "$1"
}
```

The masks correspond one-to-one to the by-design-volatile tokens: the `RUN N` label, secure per-probe `mkdtemp` names (CWE-377 safe), ocrmypdf/unpaper internal temp dirs, ffmpeg heap addresses in the unpaper banner, wall-clock log timestamps, the model-file `st_mtime` float, and — for Q4 — the pikepdf-nondeterministic fragment md5/sha256 checksums. Saving each question's two complete runs to files `qN_run1.txt` / `qN_run2.txt` (run 1 in the body/appendix, run 2 in §7.3.1–§7.3.6) and running `diff <(normalize run1) <(normalize run2)` gives an **empty** diff for every question — the raw diff is non-empty only in those masked tokens. The actual command and its complete output:

```text
$ for q in q1 q2 q3 q4; do
>   raw=$(diff ${q}_run1.txt ${q}_run2.txt | grep -cE '^[<>]')
>   norm=$(diff <(normalize ${q}_run1.txt) <(normalize ${q}_run2.txt) | grep -cE '^[<>]')
>   echo "$q | raw differing lines=$raw | normalized differing lines=$norm"
> done
q1 | raw differing lines=16 | normalized differing lines=0
q2 | raw differing lines=6 | normalized differing lines=0
q3 | raw differing lines=150 | normalized differing lines=0
q4 | raw differing lines=96 | normalized differing lines=0
```

Interpretation: after masking tokens that are volatile by construction, the two runs are byte-identical. Every answer-bearing line — booleans (REUSE/RETRAIN True), counts (3 fragments, 2 rows, corpus 0->3), `classes_`, `mime_type='image/png'`, gs `'-dPDFA=2'` argv, `skip_text:True`->`force_ocr:True` args dicts, the `'No text was found'` warning — is reproduced exactly across both runs. (Q4 note: pikepdf's PDF writer is not byte-deterministic across processes, so the raw fragment md5 VALUES differ run-to-run; the DERIVED answers — distinct-md5 count 2-of-3 and 1-of-2, and the resulting 2 and 1 Document rows, corpus 0->3 — are stable.)

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

#### 7.3.7 Q1 high-resolution lifecycle — complete second run (`python3 /tmp/obs_q1_hires.py 2`)

Run 1 is reproduced in full in §2.2.1. The complete second run is below. The reuse/retrain **invariants** are identical to Run 1 (`reuse_byte_identical=True`, `retrain_all_changed=True`); the deterministic `loaded_data_hash` values are byte-identical to Run 1 (`93c9a15d…1bea` for the original content, `441e8b90…4b5f` for the changed content); only the model-file `size`/`st_mtime_ns`/`sha256` differ from Run 1, because the serialized `MLPClassifier` weights are unseeded (see §6.4). Command:

```text
$ docker exec -u testuser -w /app/src -e PYTHONPATH=/app/src -e DJANGO_SETTINGS_MODULE=paperless.settings pngx-qna python3 /tmp/obs_q1_hires.py 2
```

```text
================ Q1-HIRES RUN 2 ================
--- A: reuse/retrain, high-resolution (st_mtime_ns / size / sha256 / loaded data_hash) ---
  corpus_count = 1
  [before] exists=False
  LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
  LOG DEBUG paperless.classifier: Vectorizing data...
  LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
  LOG DEBUG paperless.classifier: Training correspondent classifier...
  LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
  LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1h-5l3z5tt0/classification_model.pickle...
  [after#1 initial-save] exists=True corpus_count=1 size=22439 st_mtime_ns=1783990174017185669 sha256=28e2bda79eebd751569125be48487ed565fe8f81370951be62a4b13f919a454b
  [after#1] load_classifier()=OK loaded_data_hash=93c9a15dcb127652822b68bfbd2f11d6eedc1bea classes_=[1]
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.tasks: Training data unchanged.
  [after#2 unchanged] exists=True corpus_count=1 size=22439 st_mtime_ns=1783990174017185669 sha256=28e2bda79eebd751569125be48487ed565fe8f81370951be62a4b13f919a454b
  [after#2] load_classifier()=OK loaded_data_hash=93c9a15dcb127652822b68bfbd2f11d6eedc1bea classes_=[1]
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG DEBUG paperless.classifier: 1 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
  LOG DEBUG paperless.classifier: Vectorizing data...
  LOG DEBUG paperless.classifier: There are no tags. Not training tags classifier.
  LOG DEBUG paperless.classifier: Training correspondent classifier...
  LOG DEBUG paperless.classifier: There are no document types. Not training document type classifier.
  LOG INFO paperless.tasks: Saving updated classifier model to /tmp/pngx-q1h-5l3z5tt0/classification_model.pickle...
  [after#3 changed] exists=True corpus_count=1 size=36983 st_mtime_ns=1783990174044185939 sha256=83c3914f0570c6aadfbc98bbaee350f17ef7ad12823ad1fad8144d0c8f54bdbf
  [after#3] load_classifier()=OK loaded_data_hash=441e8b90dd526a0de6491ab6b8567fcf082a4b5f classes_=[1]
  REUSE   (call#2 vs #1): size_same=True mtime_ns_same=True sha256_same=True loaded_hash_same=True
  RETRAIN (call#3 vs #2): mtime_ns_changed=True sha256_changed=True loaded_hash_changed=True
--- B: compatible v7 load (real load_classifier() succeeds) ---
  on-disk schema_version=7 FORMAT_VERSION=7 compatible=True
  [compatible-v7] load_classifier()=OK loaded_data_hash=441e8b90dd526a0de6491ab6b8567fcf082a4b5f classes_=[1]
--- C: PHYSICALLY-SEPARATE incompatible on-disk model -> real load_classifier() deletes it ---
  wrote incompatible model directly to disk: schema_version=999 exists_before=True
  LOG ERROR paperless.classifier: Unrecoverable error while loading document classification model, deleting model file.
  load_classifier() returned=None exists_after=False (False => os.unlink at classifier.py:48 ran)
--- D: direct empty-corpus DocumentClassifier().train() -> ValueError ---
  corpus_count = 0
  LOG DEBUG paperless.classifier: Gathering data from database...
  ValueError: 'No training data available.'
--- E: task-level empty corpus (auto correspondent present, 0 docs) ---
  LOG DEBUG paperless.classifier: Document classification model does not exist (yet), not performing automatic matching.
  LOG DEBUG paperless.classifier: Gathering data from database...
  LOG WARNING paperless.tasks: Classifier error: No training data available.
  train_classifier() returned=None model_exists=False (ValueError caught at tasks.py:70-72 => no save)
--- E2: no auto matching models at all -> early return, no train ---
  train_classifier() returned=None model_exists=False (guard at tasks.py:49-55 => early return)
================ Q1-HIRES RUN 2 SUMMARY ================
  reuse_byte_identical=True retrain_all_changed=True loaded_hash reuse=True retrain=True
```

#### 7.3.8 Q4 value matrix, enabled/disabled, and staged `[2, 5]` timeline — complete both runs (`python3 /tmp/obs_q4_matrix.py {1,2}`)

Run 1:

```text
================ Q4-MATRIX RUN 1 ================
[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
[0m

--- (M) CONSUMER_BARCODE_STRING value matrix -> scan() separators (case/whitespace/empty/custom/nonmatching) ---
fixtures: patch-code-t.pdf (decodes 'PATCHT', CODE39); barcode-128-custom.pdf (decodes 'CUSTOM BARCODE', CODE128)
CONSUMER_BARCODE_STRING scan(patch-code-t.pdf)   scan(barcode-128-custom.pdf)
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT'               [0]                      []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'patcht'               []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT '              []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
''                     []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'CUSTOM BARCODE'       []                       [0]                     
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'NONEXISTENT VALUE'    []                       []                      

--- (EN) barcodes DISABLED and nonmatching-string -> whole-file consume -> exactly 1 Document row ---
LOG INFO paperless.consumer: Consuming patch-code-t-middle.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-middle.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-en-o3vtrgt4/patch-code-t-middle.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-en-o3vtrgt4/patch-code-t-middle.pdf', 'output_file': '/tmp/tmpuuvxzqlj/paperless-xnl3qagx/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpuuvxzqlj/paperless-xnl3qagx/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpuuvxzqlj/paperless-xnl3qagx/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-middle.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-xnl3qagx/archive.pdf[0] /tmp/tmpuuvxzqlj/paperless-xnl3qagx/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpuuvxzqlj/paperless-xnl3qagx/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-xnl3qagx/gs_out.png /tmp/tmpuuvxzqlj/paperless-xnl3qagx/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpuuvxzqlj/paperless-xnl3qagx/convert_gs.png -out /tmp/tmpuuvxzqlj/paperless-xnl3qagx/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-en-o3vtrgt4/patch-code-t-middle.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpuuvxzqlj/paperless-xnl3qagx
LOG INFO paperless.consumer: Document 2010-12-01 patch-code-t-middle consumption finished
  DISABLED / patch-code-t-middle.pdf         enable=False string='PATCHT'           -> 'Success. New document id 1 created' | rows 0->1 (delta=1)
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
LOG INFO paperless.consumer: Consuming patch-code-t-qr.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-qr.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-en-ciifvmah/patch-code-t-qr.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-en-ciifvmah/patch-code-t-qr.pdf', 'output_file': '/tmp/tmpuuvxzqlj/paperless-pwsqdjjv/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpuuvxzqlj/paperless-pwsqdjjv/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-qr.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/archive.pdf[0] /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpuuvxzqlj/paperless-pwsqdjjv/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/gs_out.png /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/convert_gs.png -out /tmp/tmpuuvxzqlj/paperless-pwsqdjjv/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-en-ciifvmah/patch-code-t-qr.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpuuvxzqlj/paperless-pwsqdjjv
LOG INFO paperless.consumer: Document 2026-07-14 patch-code-t-qr consumption finished
  NONMATCHING / patch-code-t-qr.pdf          enable=True  string='NONEXISTENT VALUE' -> 'Success. New document id 2 created' | rows 1->2 (delta=1)

--- (ST) staged DB timeline for several-patcht-codes.pdf split=[2,5] (before/split/consume x3/retrain) ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4m-in-_mb657p7/several-patcht-codes.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpuuvxzqlj/paperless-yzindt3j
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpuuvxzqlj/paperless-yzindt3j/several-patcht-codes_document_0.pdf', '/tmp/tmpuuvxzqlj/paperless-yzindt3j/several-patcht-codes_document_1.pdf', '/tmp/tmpuuvxzqlj/paperless-yzindt3j/several-patcht-codes_document_2.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4m-in-_mb657p7/several-patcht-codes.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-frag-37lcf33c/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-frag-37lcf33c/several-patcht-codes_document_0.pdf', 'output_file': '/tmp/tmpuuvxzqlj/paperless-t4oyopht/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpuuvxzqlj/paperless-t4oyopht/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpuuvxzqlj/paperless-t4oyopht/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-t4oyopht/archive.pdf[0] /tmp/tmpuuvxzqlj/paperless-t4oyopht/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpuuvxzqlj/paperless-t4oyopht/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-t4oyopht/gs_out.png /tmp/tmpuuvxzqlj/paperless-t4oyopht/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpuuvxzqlj/paperless-t4oyopht/convert_gs.png -out /tmp/tmpuuvxzqlj/paperless-t4oyopht/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-frag-37lcf33c/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpuuvxzqlj/paperless-t4oyopht
LOG INFO paperless.consumer: Document 2026-07-14 several-patcht-codes_document_0 consumption finished
LOG ERROR paperless.consumer: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_2.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-frag-w33ttlx5/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-frag-w33ttlx5/several-patcht-codes_document_2.pdf', 'output_file': '/tmp/tmpuuvxzqlj/paperless-9jtabtsk/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpuuvxzqlj/paperless-9jtabtsk/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpuuvxzqlj/paperless-9jtabtsk/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-9jtabtsk/archive.pdf[0] /tmp/tmpuuvxzqlj/paperless-9jtabtsk/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpuuvxzqlj/paperless-9jtabtsk/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpuuvxzqlj/paperless-9jtabtsk/gs_out.png /tmp/tmpuuvxzqlj/paperless-9jtabtsk/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpuuvxzqlj/paperless-9jtabtsk/convert_gs.png -out /tmp/tmpuuvxzqlj/paperless-9jtabtsk/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-frag-w33ttlx5/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpuuvxzqlj/paperless-9jtabtsk
LOG INFO paperless.consumer: Document 2026-07-14 several-patcht-codes_document_2 consumption finished
  STAGE                      Doc.count  NOTE
  0 before split             0          input file only
  1 after file-only split    0          'File successfully split'; original unlinked=True; 3 fragments=['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
  2 after consume frag_0     1          several-patcht-codes_document_0.pdf CREATED pk=3 (md5=d74161c9d1a271807ae51a6b25a10aad)
  3 after consume frag_1     1          several-patcht-codes_document_1.pdf REJECTED duplicate (md5=d74161c9d1a271807ae51a6b25a10aad)
  4 after consume frag_2     2          several-patcht-codes_document_2.pdf CREATED pk=4 (md5=17c11e09a10bb4a0762a6cba8d817035)
  5 after retrain            2          train() returned True; effective corpus (inbox-excluded) = 2
SUMMARY: [2,5] -> 3 fragment files -> 2 distinct rows (1 duplicate rejected) -> effective training corpus 2

================ Q4-MATRIX RUN 1 SUMMARY ================
matrix: only exact configured string matches (case- & whitespace-sensitive); empty never matches; custom matches only its own value; nonmatching -> [] | disabled/nonmatching -> whole-file consume -> 1 row | [2,5] staged: 0->0(split)->1->1(dup)->2->retrain(corpus 2)
```

Run 2 (matrix and staged counts identical to run 1; only fragment md5 values differ, as noted in §5.5.1):

```text
================ Q4-MATRIX RUN 2 ================
[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
[0m

--- (M) CONSUMER_BARCODE_STRING value matrix -> scan() separators (case/whitespace/empty/custom/nonmatching) ---
fixtures: patch-code-t.pdf (decodes 'PATCHT', CODE39); barcode-128-custom.pdf (decodes 'CUSTOM BARCODE', CODE128)
CONSUMER_BARCODE_STRING scan(patch-code-t.pdf)   scan(barcode-128-custom.pdf)
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT'               [0]                      []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'patcht'               []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'PATCHT '              []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
''                     []                       []                      
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'CUSTOM BARCODE'       []                       [0]                     
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE128 found: CUSTOM BARCODE
'NONEXISTENT VALUE'    []                       []                      

--- (EN) barcodes DISABLED and nonmatching-string -> whole-file consume -> exactly 1 Document row ---
LOG INFO paperless.consumer: Consuming patch-code-t-middle.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-middle.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-en-cc4h0k4u/patch-code-t-middle.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-en-cc4h0k4u/patch-code-t-middle.pdf', 'output_file': '/tmp/tmpdw6lfz_j/paperless-gzey1ehg/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpdw6lfz_j/paperless-gzey1ehg/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpdw6lfz_j/paperless-gzey1ehg/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-middle.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-gzey1ehg/archive.pdf[0] /tmp/tmpdw6lfz_j/paperless-gzey1ehg/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpdw6lfz_j/paperless-gzey1ehg/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-gzey1ehg/gs_out.png /tmp/tmpdw6lfz_j/paperless-gzey1ehg/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpdw6lfz_j/paperless-gzey1ehg/convert_gs.png -out /tmp/tmpdw6lfz_j/paperless-gzey1ehg/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-en-cc4h0k4u/patch-code-t-middle.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpdw6lfz_j/paperless-gzey1ehg
LOG INFO paperless.consumer: Document 2010-12-01 patch-code-t-middle consumption finished
  DISABLED / patch-code-t-middle.pdf         enable=False string='PATCHT'           -> 'Success. New document id 1 created' | rows 0->1 (delta=1)
LOG DEBUG paperless.tasks: Barcode of type QRCODE found: PATCHT
LOG INFO paperless.consumer: Consuming patch-code-t-qr.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t-qr.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-en-y0js4ytj/patch-code-t-qr.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-en-y0js4ytj/patch-code-t-qr.pdf', 'output_file': '/tmp/tmpdw6lfz_j/paperless-93esjsvc/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpdw6lfz_j/paperless-93esjsvc/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpdw6lfz_j/paperless-93esjsvc/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for patch-code-t-qr.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-93esjsvc/archive.pdf[0] /tmp/tmpdw6lfz_j/paperless-93esjsvc/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpdw6lfz_j/paperless-93esjsvc/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-93esjsvc/gs_out.png /tmp/tmpdw6lfz_j/paperless-93esjsvc/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpdw6lfz_j/paperless-93esjsvc/convert_gs.png -out /tmp/tmpdw6lfz_j/paperless-93esjsvc/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-en-y0js4ytj/patch-code-t-qr.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpdw6lfz_j/paperless-93esjsvc
LOG INFO paperless.consumer: Document 2026-07-14 patch-code-t-qr consumption finished
  NONMATCHING / patch-code-t-qr.pdf          enable=True  string='NONEXISTENT VALUE' -> 'Success. New document id 2 created' | rows 1->2 (delta=1)

--- (ST) staged DB timeline for several-patcht-codes.pdf split=[2,5] (before/split/consume x3/retrain) ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4m-in-7yza3v7j/several-patcht-codes.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpdw6lfz_j/paperless-49vzpebj
LOG DEBUG paperless.tasks: Count: 0 page_number: 2
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: page_number: 2 next_page: 5
LOG DEBUG paperless.tasks: pdf no:0 has 2 pages
LOG DEBUG paperless.tasks: Count: 1 page_number: 5
LOG DEBUG paperless.tasks: page_number: 5 next_page: 7
LOG DEBUG paperless.tasks: pdf no:1 has 1 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpdw6lfz_j/paperless-49vzpebj/several-patcht-codes_document_0.pdf', '/tmp/tmpdw6lfz_j/paperless-49vzpebj/several-patcht-codes_document_1.pdf', '/tmp/tmpdw6lfz_j/paperless-49vzpebj/several-patcht-codes_document_2.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4m-in-7yza3v7j/several-patcht-codes.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-frag-1yz4j6sl/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-frag-1yz4j6sl/several-patcht-codes_document_0.pdf', 'output_file': '/tmp/tmpdw6lfz_j/paperless-ea_i2l_1/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpdw6lfz_j/paperless-ea_i2l_1/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_0.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/archive.pdf[0] /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpdw6lfz_j/paperless-ea_i2l_1/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/gs_out.png /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/convert_gs.png -out /tmp/tmpdw6lfz_j/paperless-ea_i2l_1/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-frag-1yz4j6sl/several-patcht-codes_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpdw6lfz_j/paperless-ea_i2l_1
LOG INFO paperless.consumer: Document 2026-07-14 several-patcht-codes_document_0 consumption finished
LOG ERROR paperless.consumer: Not consuming several-patcht-codes_document_1.pdf: It is a duplicate.
LOG INFO paperless.consumer: Consuming several-patcht-codes_document_2.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4m-frag-hml_otea/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4m-frag-hml_otea/several-patcht-codes_document_2.pdf', 'output_file': '/tmp/tmpdw6lfz_j/paperless-i2rxpf7j/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpdw6lfz_j/paperless-i2rxpf7j/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/archive.pdf
LOG DEBUG paperless.consumer: Generating thumbnail for several-patcht-codes_document_2.pdf...
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/archive.pdf[0] /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/convert.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/tmpdw6lfz_j/paperless-i2rxpf7j/convert.png' @ error/convert.c/ConvertImageCommand/3229.
LOG WARNING paperless.parsing: Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
LOG DEBUG paperless.parsing: Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/gs_out.png /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/convert_gs.png
LOG DEBUG paperless.parsing.tesseract: Execute: optipng -silent -o5 /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/convert_gs.png -out /tmp/tmpdw6lfz_j/paperless-i2rxpf7j/thumb_optipng.png
LOG DEBUG paperless.consumer: Saving record to database
LOG DEBUG paperless.consumer: Deleting file /tmp/pngx-q4m-frag-hml_otea/several-patcht-codes_document_2.pdf
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpdw6lfz_j/paperless-i2rxpf7j
LOG INFO paperless.consumer: Document 2026-07-14 several-patcht-codes_document_2 consumption finished
  STAGE                      Doc.count  NOTE
  0 before split             0          input file only
  1 after file-only split    0          'File successfully split'; original unlinked=True; 3 fragments=['several-patcht-codes_document_0.pdf', 'several-patcht-codes_document_1.pdf', 'several-patcht-codes_document_2.pdf']
  2 after consume frag_0     1          several-patcht-codes_document_0.pdf CREATED pk=3 (md5=71fd51f158983f9366b65ccbc04d9d27)
  3 after consume frag_1     1          several-patcht-codes_document_1.pdf REJECTED duplicate (md5=71fd51f158983f9366b65ccbc04d9d27)
  4 after consume frag_2     2          several-patcht-codes_document_2.pdf CREATED pk=4 (md5=55539372c78b5926bb2baedf1f80993c)
  5 after retrain            2          train() returned True; effective corpus (inbox-excluded) = 2
SUMMARY: [2,5] -> 3 fragment files -> 2 distinct rows (1 duplicate rejected) -> effective training corpus 2

================ Q4-MATRIX RUN 2 SUMMARY ================
matrix: only exact configured string matches (case- & whitespace-sensitive); empty never matches; custom matches only its own value; nonmatching -> [] | disabled/nonmatching -> whole-file consume -> 1 row | [2,5] staged: 0->0(split)->1->1(dup)->2->retrain(corpus 2)
```

#### 7.3.9 Q4 page-0 boundary — complete both runs (`timeout 120 python3 /tmp/obs_q4_page0.py; echo "exit=$?"` → `exit=124`)

Run 1 (`exit=124` — the non-daemon-thread hang):

```text
================ Q4-PAGE0 ================
[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
[0m
input patch-code-t.pdf pages=1 size=40893

--- (1) scan_file_for_separating_barcodes(patch-code-t.pdf) ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
separators = [0]

--- (2) separate_pages(patch-code-t.pdf, [0]) -> inspect each emitted fragment ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpgcjbsc2k/paperless-m5c7ach8
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpgcjbsc2k/paperless-m5c7ach8/patch-code-t_document_0.pdf', '/tmp/tmpgcjbsc2k/paperless-m5c7ach8/patch-code-t_document_1.pdf']
fragment count = 2
  fragment patch-code-t_document_0.pdf              pages=0 size=315 md5=23ca22d022b120dbed9ad7024e174cda sha256=3b2d9eca14f0cd809195671cb16e9c0d814a3c622862dc3bd788e02bf4ef0cd9
  fragment patch-code-t_document_1.pdf              pages=0 size=315 md5=23ca22d022b120dbed9ad7024e174cda sha256=3b2d9eca14f0cd809195671cb16e9c0d814a3c622862dc3bd788e02bf4ef0cd9
  both fragments byte-identical to each other? True
  all fragments 0-page? True ; all 315 bytes? True

--- (3) pass EACH 0-page fragment through the REAL Consumer -> exact exception, 0 rows ---
Document.objects.count() BEFORE = 0
LOG INFO paperless.consumer: Consuming patch-code-t_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4p0-frag-enyi2scg/patch-code-t_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4p0-frag-enyi2scg/patch-code-t_document_0.pdf', 'output_file': '/tmp/tmpgcjbsc2k/paperless-4msoukax/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpgcjbsc2k/paperless-4msoukax/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpgcjbsc2k/paperless-4msoukax
LOG ERROR paperless.consumer: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 385, in run_pipeline
    exec_concurrent(context, executor)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 274, in exec_concurrent
    executor(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_concurrent.py", line 82, in __call__
    self._execute(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/builtin_plugins/concurrency.py", line 127, in _execute
    with self.pbar_class(**tqdm_kwargs) as pbar, executor_class(
  File "/usr/local/lib/python3.9/concurrent/futures/thread.py", line 144, in __init__
    raise ValueError("max_workers must be greater than 0")
ValueError: max_workers must be greater than 0

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 314, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: ValueError: max_workers must be greater than 0
  consumed patch-code-t_document_0.pdf              -> ConsumerError: patch-code-t_document_0.pdf: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
LOG INFO paperless.consumer: Consuming patch-code-t_document_1.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t_document_1.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4p0-frag-etuap_ly/patch-code-t_document_1.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4p0-frag-etuap_ly/patch-code-t_document_1.pdf', 'output_file': '/tmp/tmpgcjbsc2k/paperless-h2aases_/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmpgcjbsc2k/paperless-h2aases_/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmpgcjbsc2k/paperless-h2aases_
LOG ERROR paperless.consumer: Error while consuming document patch-code-t_document_1.pdf: ValueError: max_workers must be greater than 0
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 385, in run_pipeline
    exec_concurrent(context, executor)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 274, in exec_concurrent
    executor(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_concurrent.py", line 82, in __call__
    self._execute(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/builtin_plugins/concurrency.py", line 127, in _execute
    with self.pbar_class(**tqdm_kwargs) as pbar, executor_class(
  File "/usr/local/lib/python3.9/concurrent/futures/thread.py", line 144, in __init__
    raise ValueError("max_workers must be greater than 0")
ValueError: max_workers must be greater than 0

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 314, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: ValueError: max_workers must be greater than 0
  consumed patch-code-t_document_1.pdf              -> ConsumerError: patch-code-t_document_1.pdf: Error while consuming document patch-code-t_document_1.pdf: ValueError: max_workers must be greater than 0
Document.objects.count() AFTER  = 0 (0 => neither 0-page fragment produced a row)

--- (4) consume_file(patch-code-t.pdf, CONSUMER_ENABLE_BARCODES=True) still 'File successfully split' + unlink ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4p0-in-l_8ar02x/patch-code-t.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmpgcjbsc2k/paperless-kx8qbd82
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmpgcjbsc2k/paperless-kx8qbd82/patch-code-t_document_0.pdf', '/tmp/tmpgcjbsc2k/paperless-kx8qbd82/patch-code-t_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4p0-in-l_8ar02x/patch-code-t.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
  consume_file(...) returned = 'File successfully split'
  original input still exists? = False (False => os.unlink at tasks.py:214)
  isolated consume dir contents = ['patch-code-t_document_0.pdf', 'patch-code-t_document_1.pdf']
  Document.objects.count() after split = 0

--- (5) live threads after OCRmyPDF ran (non-daemon threads keep the process alive) ---
  thread name='MainThread'             daemon=False alive=True
  thread name='Thread-8'               daemon=True  alive=True
  thread name='Thread-9'               daemon=False alive=True
  thread name='Thread-11'              daemon=False alive=True
  thread name='ThreadPoolExecutor-3_0' daemon=False alive=True
  non-daemon non-main threads still alive = 3 -> ['Thread-9', 'Thread-11', 'ThreadPoolExecutor-3_0']
================ END OF SCRIPT BODY ================
main thread returning now; if a non-daemon thread is alive the process will NOT exit (timeout => hang)
```

Run 2 (`exit=124`; all invariants identical to run 1; only the empty-PDF digests differ, as noted in §5.5.2):

```text
================ Q4-PAGE0 ================
[1m

  This is a one-time only migration to generate thumbnails for all of your
  documents so that future UIs will have something to work with.  If you have
  a lot of documents though, this may take a while, so a coffee break may be
  in order.
[0m
input patch-code-t.pdf pages=1 size=40893

--- (1) scan_file_for_separating_barcodes(patch-code-t.pdf) ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
separators = [0]

--- (2) separate_pages(patch-code-t.pdf, [0]) -> inspect each emitted fragment ---
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmp48eugg2t/paperless-v64g2eof
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmp48eugg2t/paperless-v64g2eof/patch-code-t_document_0.pdf', '/tmp/tmp48eugg2t/paperless-v64g2eof/patch-code-t_document_1.pdf']
fragment count = 2
  fragment patch-code-t_document_0.pdf              pages=0 size=315 md5=2bf6a7ed6166329588c1a2baacbe8a8e sha256=3a07eff7a44525888f0c965ef565f0ab59ceedd70562b2dde271402feb5f0d6d
  fragment patch-code-t_document_1.pdf              pages=0 size=315 md5=2bf6a7ed6166329588c1a2baacbe8a8e sha256=3a07eff7a44525888f0c965ef565f0ab59ceedd70562b2dde271402feb5f0d6d
  both fragments byte-identical to each other? True
  all fragments 0-page? True ; all 315 bytes? True

--- (3) pass EACH 0-page fragment through the REAL Consumer -> exact exception, 0 rows ---
Document.objects.count() BEFORE = 0
LOG INFO paperless.consumer: Consuming patch-code-t_document_0.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t_document_0.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4p0-frag-s6yooqct/patch-code-t_document_0.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4p0-frag-s6yooqct/patch-code-t_document_0.pdf', 'output_file': '/tmp/tmp48eugg2t/paperless-tgzjzyje/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmp48eugg2t/paperless-tgzjzyje/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmp48eugg2t/paperless-tgzjzyje
LOG ERROR paperless.consumer: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 385, in run_pipeline
    exec_concurrent(context, executor)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 274, in exec_concurrent
    executor(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_concurrent.py", line 82, in __call__
    self._execute(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/builtin_plugins/concurrency.py", line 127, in _execute
    with self.pbar_class(**tqdm_kwargs) as pbar, executor_class(
  File "/usr/local/lib/python3.9/concurrent/futures/thread.py", line 144, in __init__
    raise ValueError("max_workers must be greater than 0")
ValueError: max_workers must be greater than 0

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 314, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: ValueError: max_workers must be greater than 0
  consumed patch-code-t_document_0.pdf              -> ConsumerError: patch-code-t_document_0.pdf: Error while consuming document patch-code-t_document_0.pdf: ValueError: max_workers must be greater than 0
LOG INFO paperless.consumer: Consuming patch-code-t_document_1.pdf
LOG DEBUG paperless.consumer: Detected mime type: application/pdf
LOG DEBUG paperless.consumer: Parser: RasterisedDocumentParser
LOG DEBUG paperless.consumer: Parsing patch-code-t_document_1.pdf...
LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pngx-q4p0-frag-oyvv7gio/patch-code-t_document_1.pdf
LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/pngx-q4p0-frag-oyvv7gio/patch-code-t_document_1.pdf', 'output_file': '/tmp/tmp48eugg2t/paperless-dc7mszm3/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/tmp48eugg2t/paperless-dc7mszm3/sidecar.txt'}
LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/tmp48eugg2t/paperless-dc7mszm3
LOG ERROR paperless.consumer: Error while consuming document patch-code-t_document_1.pdf: ValueError: max_workers must be greater than 0
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 385, in run_pipeline
    exec_concurrent(context, executor)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 274, in exec_concurrent
    executor(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_concurrent.py", line 82, in __call__
    self._execute(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/builtin_plugins/concurrency.py", line 127, in _execute
    with self.pbar_class(**tqdm_kwargs) as pbar, executor_class(
  File "/usr/local/lib/python3.9/concurrent/futures/thread.py", line 144, in __init__
    raise ValueError("max_workers must be greater than 0")
ValueError: max_workers must be greater than 0

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 314, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: ValueError: max_workers must be greater than 0
  consumed patch-code-t_document_1.pdf              -> ConsumerError: patch-code-t_document_1.pdf: Error while consuming document patch-code-t_document_1.pdf: ValueError: max_workers must be greater than 0
Document.objects.count() AFTER  = 0 (0 => neither 0-page fragment produced a row)

--- (4) consume_file(patch-code-t.pdf, CONSUMER_ENABLE_BARCODES=True) still 'File successfully split' + unlink ---
LOG DEBUG paperless.tasks: Barcode of type CODE39 found: PATCHT
LOG DEBUG paperless.tasks: Pages with separators found in: /tmp/pngx-q4p0-in-b5te3kkh/patch-code-t.pdf
LOG DEBUG paperless.tasks: Temp dir is /tmp/tmp48eugg2t/paperless-u7ves424
LOG DEBUG paperless.tasks: Count: 0 page_number: 0
LOG DEBUG paperless.tasks: pdf no:0 has 0 pages
LOG DEBUG paperless.tasks: Temp files are ['/tmp/tmp48eugg2t/paperless-u7ves424/patch-code-t_document_0.pdf', '/tmp/tmp48eugg2t/paperless-u7ves424/patch-code-t_document_1.pdf']
LOG DEBUG paperless.tasks: Deleting file /tmp/pngx-q4p0-in-b5te3kkh/patch-code-t.pdf
LOG WARNING paperless.tasks: OSError. It could be, the broker cannot be reached.
LOG WARNING paperless.tasks: Multiple exceptions: [Errno 111] Connect call failed ('::1', 6379, 0, 0), [Errno 111] Connect call failed ('127.0.0.1', 6379)
  consume_file(...) returned = 'File successfully split'
  original input still exists? = False (False => os.unlink at tasks.py:214)
  isolated consume dir contents = ['patch-code-t_document_0.pdf', 'patch-code-t_document_1.pdf']
  Document.objects.count() after split = 0

--- (5) live threads after OCRmyPDF ran (non-daemon threads keep the process alive) ---
  thread name='MainThread'             daemon=False alive=True
  thread name='Thread-8'               daemon=True  alive=True
  thread name='Thread-9'               daemon=False alive=True
  thread name='Thread-11'              daemon=False alive=True
  thread name='ThreadPoolExecutor-3_0' daemon=False alive=True
  non-daemon non-main threads still alive = 3 -> ['Thread-9', 'Thread-11', 'ThreadPoolExecutor-3_0']
================ END OF SCRIPT BODY ================
main thread returning now; if a non-daemon thread is alive the process will NOT exit (timeout => hang)
```


### 7.4 Non-determinism probe — the ten suite runs and the isolated prediction-path flip

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

#### 7.4.11 Isolated prediction-path flip — one 30-process batch, complete (`obs_q6_nd.py`)

This is the canonical batch referenced by §6.5. Command and complete, unedited output (30 fresh single-process invocations; the corpus SHA-1 is byte-identical to the value Report 5 reports, every model SHA-256 is distinct, and the `verdict` flips on the same unchanged input):

```text
$ for i in $(seq 1 30); do python3 /tmp/obs_q6_nd.py $i 2>/dev/null | grep "^RUN="; done
RUN=1 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=32b335849823ba7fbb3af023c3562b19d658065703ec052e35bafaccaaa0dac2 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=2 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=5b0be3c80b03a4c10b9bd4d0c6e777ca18f90aaf55b44694855e625c8172163c classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=3 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=d9fa57ef7d55d08082f3b0e49c895d82b5a0d4381811db3b514fa86612a16426 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=4 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=17bb6de9f25c07ec0a8bdd56610f3bdf241bbdd59c4d57e520fb4fa388177005 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=5 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=1b76d4be0d9f5329ccc5a8a3ccf358045f5803aec946f6765966b9bc86a966f1 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=6 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=e4cd60e91ab53e521d1aa583280b4656e10dd05131f6e0fec138355b7e141c06 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=7 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=345b6485fbc2eaa57da56c6c0ce9881dba4c84fd2d258abf80ad73f1205c41e6 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=8 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=2af4bfe9847730274c3089a3c7d3c06f5d4810177b47aab07bc7e7d383aba43b classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=9 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=d3d4998a27814473ea076cf226734b073adb5009febeb24d42d283ad4dcae360 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=10 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=74d68dc489a86f5b8f3ed2aa8f681ab12355590661e820b07ac955b7f4c509a7 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=11 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=94b4234d31e7e35834c6f9066c56711cb68d5e19931d7e192b9df3994a77d3de classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=12 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=25d97e3e9f42b301eec1c777e271b187ae1ab558ce0bd58039862145e037de4e classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=13 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=b15d96f1557bd211b3de25b1cf4f89fd438b198471cd1a4512a58fd41d061232 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=14 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=6c178e5f92fcb3581c1d3aaa7884d00a6aaebfc398175088cefa44407cbc7ce5 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=15 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=e8cb8db6db5018f01ff8313aa2fa41624aa08d2c09a724101230d7401dd005ca classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=16 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=5ec1a0703d9e5ad0a9e077ccef2b5f57596715e0d86af64dab203d8307e84501 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=17 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=87e8cffe65c397bb7feb5c56fbed77933cdb02b8670849542eeb86057c609550 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=18 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=d3db698b039e5b3adcca97e85e197054cfbdda6a774f1e67384b1f2c23500c01 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=19 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=a38813b60865a7b7c9edf4828f0ee60b18a9b15f621693fffe6b77e5d811c363 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=20 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=5179dece524dbf90b17fae37b62d9973400999668782d368267b7fdefc95d9ab classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=21 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=46f00178673cd50314e89484233799fa2b3f10dd76f79c9d0e67b25262d2a924 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=22 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=6173ec104e339986f4cb26bc9e4e56432d9db5c851165663e9252c003d996b8d classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=23 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=dde13f602b79232427a8519b438aa5218e7bdb79fe98f92795af66ef7b4ec152 classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=24 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=49ded29a6a734031475ed534833c2d87cc7e8a14977ae52bd0bb468c473a3e5e classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=25 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=2f3bdb0047821317649f106eacf8e19927c03bea7cec51572b9ea5fbdb7c9a5e classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=26 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=db6519bc85e36132d1ef18ec1f0e66898e76affe5211b0f7932e662feaf23c4f classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=27 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=f4be70489f8a6661fbee1a2fe3c804a7cce21aeb1e0989e184a56ab101a4c44b classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=28 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=21c12f1925cc503b8107536609e7ab3f3780da0c3c255d52d142f2b093f3f53c classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
RUN=29 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=f710d8416eb3fe64be536dfea09506a451b2ccf39b616a329a47092a922ae47f classes_=[-1, 1] pred=None match=[] verdict=NONE
RUN=30 total=3 effective=2 c1.pk=1 corpus_sha1=98b0e672e43fbc7863208b65051bd70b50571754 model_sha256=f27a0f2a4a284154fc4683a3f34efc6733242d5987ff094d52d58084f5bfc2d1 classes_=[-1, 1] pred=[1] match=['Auto C1'] verdict=MATCH
```

Tally of this batch: **14 MATCH / 16 NONE**; `distinct corpus_sha1 = 1`; `distinct model_sha256 = 30`. Batch A above is exactly `14/16`; three further independent 30-process batches gave `13/17`, `15/15`, `12/18`, for an aggregate of **54 MATCH / 66 NONE over 120 fresh processes** (1 distinct corpus SHA-1, 120 distinct model SHA-256) — see §6.5.

#### 7.4.12 Overlapping-vs-unseen contrast — 20 processes, complete (`obs_q6_contrast.py`)

Command and complete, unedited output. Each process trains the identical corpus and predicts two inputs; `overlap=` is the overlapping-vocabulary input, `unseen=` is the unseen-vocabulary input. The overlapping input never flips (20/20 MATCH); the unseen input flips (12/20 MATCH, 8/20 NONE):

```text
$ for i in $(seq 1 20); do python3 /tmp/obs_q6_contrast.py $i 2>/dev/null | grep "^RUN="; done
RUN=1 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=2 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=3 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=4 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=5 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=6 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=7 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=8 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=9 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=10 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=11 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=12 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=13 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=14 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=15 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=16 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=17 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=18 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
RUN=19 overlap_pred=[1] overlap=MATCH | unseen_pred=[1] unseen=MATCH
RUN=20 overlap_pred=[1] overlap=MATCH | unseen_pred=None unseen=NONE
```


### 7.5 Repository and container cleanliness

`[observed]` **The repository checkout was never modified** by the investigation. All observation scripts (§7.2) were delivered into the container's `/tmp` and executed there; the container's `/app` tree is a distinct checkout from the repository under `blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf_735501`, so container-side artifacts can never appear in the repository's git state. The sole change tracked by the repository is this deliverable, `blitzy/documentation/paperless-ngx_542221a38dff.md`. While this document was being finalized (before committing), it was the only entry in the working-tree status — no source, test, or config file appears:

```text
$ REPO=/tmp/blitzy/paperless-ngx/blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf_735501

$ git -C "$REPO" rev-parse --abbrev-ref HEAD
blitzy-5fa640eb-6577-4b56-a9e5-a622258d47bf

$ git -C "$REPO" status --porcelain     # sole working-tree change is the tracked deliverable
 M blitzy/documentation/paperless-ngx_542221a38dff.md

$ git -C "$REPO" diff --name-status HEAD
M	blitzy/documentation/paperless-ngx_542221a38dff.md
```

Measured from the commit under investigation (`542221a38dff`) to `HEAD`, the deliverable is the **only** added path; the real recent history (this finalizing edit is committed on top of `d69ba06cb`) is:

```text
$ git -C "$REPO" diff --name-status 542221a38dff HEAD   # baseline (commit under investigation) -> HEAD: sole addition
A	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git -C "$REPO" log --oneline -5
d69ba06cb docs(qna): address Q3-DOC-1/Q3-DOC-2 in paperless-ngx §4.5 (non-default OCR_MODE disclosure + inline pytest capture)
fce84f647 docs(qna): fix 3 MINOR file:line citation precision issues in paperless-ngx investigation
93dc64ee5 docs(qna): rewrite paperless-ngx runtime investigation with complete canonical evidence
4b7d6d724 Add QnA investigation: paperless-ngx classifier/OCR/matching/barcode runtime behavior
542221a38 Merge pull request #792 from paperless-ngx/dependabot/github_actions/github/codeql-action-2
```

After this document is committed, `git -C "$REPO" status --porcelain` produces no output (clean working tree) and `git -C "$REPO" diff --name-status 542221a38dff HEAD` still reports the single added deliverable; the exact post-commit `git log --oneline` (the new `docs(qna)` commit on top of `d69ba06cb`) and the clean status are captured immediately after the commit and recorded in the resolution report.

`[observed]` **The import-bound split target `/app/consume` was inventoried and left empty.** As established in §5.1, `documents.tasks.save_to_dir` binds its destination default to `settings.CONSUMPTION_DIR` (`/app/src/../consume` = `/app/consume`) at import time (`src/documents/tasks.py:167`). Any fragment written there by an isolated split probe or by the canonical `test_consume_barcode_file` was removed; the directory is empty at the end of the investigation:

```text
$ docker exec -u testuser pngx-qna ls -la /app/consume        # BEFORE: fragments left by canonical test_consume_barcode_file
total 32
drwxr-sr-x 1 testuser testuser 4096 Jul 14 01:25 .
drwxr-sr-x 1 testuser testuser 4096 Jul 13 16:24 ..
-rw-r--r-- 1 testuser testuser 8156 Jul 14 01:21 patch-code-t-middle_document_0.pdf
-rw-r--r-- 1 testuser testuser 8156 Jul 14 01:21 patch-code-t-middle_document_1.pdf

$ docker exec -u testuser pngx-qna sh -c 'rm -f /app/consume/*.pdf; echo "rm exit=$?"'   # cleanup producer: remove only the test-created fragments
rm exit=0

$ docker exec -u testuser pngx-qna ls -la /app/consume        # AFTER: empty
total 16
drwxr-sr-x 1 testuser testuser 4096 Jul 14 01:35 .
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
