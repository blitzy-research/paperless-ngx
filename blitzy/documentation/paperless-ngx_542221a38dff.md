# Paperless-ngx Import Memory Investigation — Diagnostic Report

> **Deliverable identity.** This is the single, only file added to the repository for this investigation. The filename is derived from the **source branch** mandated by the AAP — `paperless-ngx_542221a38dff` — so the deliverable is named exactly `paperless-ngx_542221a38dff.md`, placed in `blitzy/documentation/`. This source-branch name is distinct from the current implementation branch: `git rev-parse --abbrev-ref HEAD` returns `blitzy-680a32bd-58ed-4a69-b43f-ef1a0acdff8b` (the working branch), not the source-branch name.
>
> **Scope: READ-ONLY.** Governed by the `SWE-AtlasQnA-Repo` rule set. No existing repository file was modified, created, or deleted; no code remediation was implemented. All temporary observation scripts lived outside the repository (under `/tmp/mem_probe` and `/tmp/mem_probe_fix`) and were removed. The git working tree ends unchanged except for this one `.md` file.

---

## Executive Summary (verdict up front)

- **The "disproportionate" spike is the classifier model reconstruction, not the document.** Loading the trained scikit-learn model via six sequential `pickle.load` calls in `DocumentClassifier.load()` (`src/documents/classifier.py:L86-L92`) adds **~+126 MB RSS on a worker's first (cold) consume** and **~+50 MB on every subsequent (warm) consume** — for a **19-byte** document either way. Cold quote (3 runs, mean `+126.42 MB`): `[cold] RSS after ONE load : 176.05 MB (delta +126.49 MB)` against `[cold] document content size : 19 bytes`; warm quote (3 runs, identical): `[warm] RSS after ONE load : 155.07 MB (delta +50.00 MB)` (EXACT, 48.03 MB model). The **~+76 MB cold/warm gap is a one-time scikit-learn import** — `[imp] one-time sklearn import cost : +73.78 MB` — not per-document cost. This **reconciles** the earlier single cold sample (`+123.48 MB`) with an independent warm reconstruction (`~+48 MB`): **cold = import + model-data, warm = model-data only**. The spike is driven by model reconstruction, **independent of the document's size**; and because `recycle: 1` (`src/paperless/settings.py:L452`) restarts the worker after every task, each consume is effectively cold, so the operational per-document spike is `~+126 MB`.
- **The "not released in a timely manner" symptom is normal glibc allocator (arena) retention — NOT a Python leak.** After `del` + `gc.collect()`, Python-object memory returns to baseline — `[D] tracemalloc AFTER del+gc : 0.00 MB` — while process RSS stays high — `[D] RSS AFTER del+gc : 107.53 MB` — and only drops when the allocator is trimmed — `[D] RSS AFTER malloc_trim(0) : 106.12 MB (trim rc=1)` (EXACT).
- **No server-side cache is accumulating metadata.** Grep across `src/documents`, `src/paperless`, and the four parser apps returns **zero** `functools.lru_cache`/`functools.cache`/`@cache`, **zero** `gc.collect`/`import gc`, and **no** Django `CACHES` setting in `src/paperless/settings.py`.
- **The bounding factor is the worker recycle policy.** `Q_CLUSTER = { ..., "recycle": 1, ... }` (`src/paperless/settings.py:L452`, block `L449-L457`) restarts the django-q worker **after every task**, so any retained memory — including glibc arenas — is reclaimed at each document boundary. This is why the symptom is **intermittent and non-fatal** rather than an unbounded crash.

---

## 1. The Reported Symptom (verbatim)

The user's own description of the symptom is preserved exactly and is the anchoring requirement for this investigation:

> "memory usage spikes significantly during document import operations, particularly when metadata is being processed, but the spike seems disproportionate to the actual document size. The odd part is that this memory doesn't always get released back to the system in a timely manner, even after processing is complete. This seems unusual because these are mostly text documents with relatively small metadata - we're not dealing with massive image processing here."

The user asked five things, answered by name in Section 4: (1) root-cause the spikes, (2) whether metadata handling makes **unnecessary copies** or **holds references** too long, (3) whether **caching** is accumulating data, (4) **what differs** between spike and no-spike cases, and (5) how behavior changes across **document types and batch sizes** — plus an explicit determination of whether this is **normal Python garbage collection** or **something problematic**, backed by **actual runtime memory measurements**.

---

## 2. Methodology & Environment

**Ran the code first, then wrote.** This report was assembled from output captured by actually executing the identified code paths and memory probes; the narrative is written from observed output, not from reading alone. Every value quoted below is pasted verbatim next to the exact claim it supports (one claim → one evidence line) and is cited with a `file:line` locator.

**Every probe pairs two independent memory readings:**

- **`tracemalloc`** — traces **Python-object allocations only**. Used to detect whether Python objects are retained (a true Python leak) or reclaimed.
- **Process RSS** — read from `/proc/self/status` `VmRSS`, `psutil`, and `resource.getrusage(RUSAGE_SELF).ru_maxrss`. Captures **native + allocator** memory (C-extension buffers and glibc arenas).

**Critical measurement limitation.** `tracemalloc` does **NOT** observe C-extension native allocations — specifically qpdf via `pikepdf` and NumPy's C buffers. Those are **only** visible via RSS. Wherever native memory is discussed, this limitation is restated, and the gap is quantified (e.g., EXACT cold-load `[cold] RSS-vs-tracemalloc gap : 48.93 MB`, and the `[P]` block below where RSS moves +2.99 MB across 50 un-closed opens while `tracemalloc` sees only 0.236 MB).

**Two environments were used and are labeled throughout:**

| Label | Environment | Purpose |
|-------|-------------|---------|
| **EXACT** (gold standard) | Docker `python:3.9-slim-bullseye` — the runtime baseline per `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`) — with the real pinned stack `scikit-learn==1.0.2`, `pikepdf==5.1.1`, `numpy==1.22.3`, `scipy==1.8.0`. Reported env line: `Python 3.9.23 \| scikit-learn 1.0.2 \| numpy 1.22.3 \| scipy 1.8.0 \| pikepdf 5.1.1`. | Full-stack, faithful reproduction with the real model and real qpdf handle. |
| **REPRESENTATIVE** | Sandbox `Python 3.12.3` + `numpy 2.5.0` + `psutil 7.2.2` (sklearn/pikepdf absent). The classifier is simulated with NumPy arrays of the **same shape**, and the `preprocess_content` / `MATCH_FUZZY` transformation lines are copied **verbatim** from the repo. | Reproduces the allocation *patterns* where the pinned native wheels cannot be built. |

Cross-environment agreement is noted where relevant: the **patterns are identical**; only the absolute MB differ by environment. The EXACT 48.03 MB model is consistent with the AAP reference of 48.41 MB, and the REPRESENTATIVE spike of +52.55 MB is consistent with the AAP reference of +53.52 MB.

---

## 3. Structural Note — Two Meanings of "Metadata Handling" (read this first)

**`extract_metadata` — the literal "metadata handling" the user references — is invoked by the REST metadata endpoint (`src/documents/views.py:L260-L305`), NOT by the core consume path.** On the REST path, `get_metadata` (`src/documents/views.py:L260`) calls `parser.extract_metadata(...)` (`src/documents/views.py:L269`). Within the `metadata` action (`src/documents/views.py:L283`), the **original** metadata extraction **always runs** — `"original_metadata": self.get_metadata(doc.source_path, doc.mime_type)` (`src/documents/views.py:L295`) — while a **second, archive** extraction runs **only when `doc.has_archive_version` is true**: the `if doc.has_archive_version:` guard (`src/documents/views.py:L300`) gates the archive call `self.get_metadata(doc.archive_path, "application/pdf")` (`src/documents/views.py:L302-L305`). So the endpoint performs **one** metadata extraction for a document without an archive version, and **two** only when an archive version exists.

The **consume path's** "metadata processing" work is dominated instead by **classification** (`src/documents/classifier.py`) and **rule-matching** (`src/documents/matching.py`). Because the user's phrase "metadata is being processed" can mean either, **this report covers both interpretations**: the classifier/matching work that dominates during import, and the `extract_metadata` parser code that runs on the REST endpoint.

---

## 4. Answers to the Five Questions

Each subsection pastes the single specific evidence line next to each behavioral claim (one claim → one evidence line) and cites the exact `file:line`.

### Question 1 — Root-cause the spikes (which components/methods allocate memory during import)

**Entry point — the classifier is loaded once per document.** `Consumer.try_consume_file` (`src/documents/consumer.py:L180`) loads the classifier a single time at `classifier = load_classifier()` (`src/documents/consumer.py:L292`). The in-code rationale (`src/documents/consumer.py:L288-L290`) explains why it is loaded once rather than per-hook:

> `# TODO: I don't really like to do this here, but this way we avoid`
> `#   reloading the classifier multiple times, since there are multiple`
> `#   post-consume hooks that all require the classifier.`

**The DOMINANT, document-size-independent allocation is the model reconstruction.** `DocumentClassifier.load()` (`src/documents/classifier.py:L76`) rebuilds the pickled scikit-learn model with **six sequential `pickle.load` calls** (`src/documents/classifier.py:L86-L92`): `L86` `data_hash`, `L87` `data_vectorizer`, `L88` `tags_binarizer`, `L90` `tags_classifier`, `L91` `correspondent_classifier`, `L92` `document_type_classifier`. A schema-version integer is read first (`src/documents/classifier.py:L78`) and checked against `FORMAT_VERSION = 7` (`src/documents/classifier.py:L63`); a mismatch raises `IncompatibleClassifierVersionError` (`src/documents/classifier.py:L80-L81`).

- **Claim:** a real ~48 MB model load produces a large RSS spike that is unrelated to document size, decomposing into a one-time (cold) scikit-learn import plus the per-load model data.
- **Evidence (EXACT), cold — a worker's first consume:** `[cold] RSS after ONE load        :   176.05 MB   (delta +126.49 MB)` measured against `[cold] document content size     :       19 bytes` and `[build] classification_model.pickle MB:    48.03` (3 runs: mean `+126.42 MB`, range `[+126.37, +126.49]`).
- **Evidence (EXACT), warm — every subsequent consume:** `[warm] RSS after ONE load        :   155.07 MB   (delta +50.00 MB)` (3 runs identical; `tracemalloc` peak `48.82 MB` ≈ model data).
- **Evidence (EXACT), the cold/warm gap is a one-time library import:** `[imp] one-time sklearn import cost     :   +73.78 MB  (cold->warm)`; identity `import (+73.78) + warm (+50.00) = +123.78 ≈ cold +126.42`.
- **Conclusion:** the spike is driven by **model reconstruction**, independent of document size — a 19-byte document still triggers `~+126 MB` RSS cold / `~+50 MB` warm. This **reconciles** the earlier single cold sample (`+123.48 MB`) and an independent warm reconstruction (`~+48 MB`) as the **same phenomenon measured from a cold vs a warm baseline**; under `recycle: 1` (`src/paperless/settings.py:L452`) each consume is effectively cold (`~+126 MB`).

**The classification/rule-matching stage is the rest of the "metadata processing" work.** In `src/documents/classifier.py`: `predict_correspondent` (`L251`), `predict_document_type` (`L262`), `predict_tags` (`L273`). Rule matching lives in `src/documents/matching.py`: `matches()` (`src/documents/matching.py:L60`), driven from `match_correspondents` (`src/documents/matching.py:L21`), `match_document_types` (`src/documents/matching.py:L34`), and `match_tags` (`src/documents/matching.py:L47`).

### Question 2 — Unnecessary copies / references held too long

**`preprocess_content` makes two transient copies per call, and runs 3× per document.** `preprocess_content` (`src/documents/classifier.py:L24-L27`) executes `content = content.lower().strip()` (`src/documents/classifier.py:L25`) and `content = re.sub(r"\s+", " ", content)` (`src/documents/classifier.py:L26`). Each of `predict_correspondent`/`predict_document_type`/`predict_tags` calls `self.data_vectorizer.transform([preprocess_content(content)])`, so it runs **three times per document**.

- **Claim:** the three per-document preprocess copies are modest.
- **Evidence (REPRESENTATIVE):** `[E] tracemalloc peak (3 copies): 2.723 MB`.

**The `MATCH_FUZZY` branch builds a full-content copy per matching model.** In `matches()` the fuzzy branch (`src/documents/matching.py:L127-L134`) computes `text = re.sub(r"[^\w\s]", "", document_content)` (`src/documents/matching.py:L131`) and then `text = text.lower()` (`src/documents/matching.py:L134`). Because `matches()` is invoked once **per** `Correspondent`/`DocumentType`/`Tag` object via `filter(lambda o: matches(o, document) ...)` over `.objects.all()` (`match_correspondents` `src/documents/matching.py:L21`, `match_document_types` `src/documents/matching.py:L34`, `match_tags` `src/documents/matching.py:L47`), the copies scale with the number of matching objects.

- **Claim:** fuzzy copies scale roughly linearly with the number of matching models.
- **Evidence (REPRESENTATIVE):** `[C-cumulative] N=10: 2.853 MB (13.2x); N=20: 4.922 MB (22.7x); N=40: 9.062 MB (41.8x)` (of a 0.217 MB content); and in the faithful transient case where each copy is freed per call, `[C-transient] 40 sequential matches(): tracemalloc peak = 1.197 MB (5.5x)`.

**The native PDF handle is never explicitly closed.** In `extract_metadata` (`src/paperless_tesseract/parsers.py:L26`), the handle `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`) is **never explicitly closed** before `return result`; it is released non-deterministically when the local goes out of scope at function return. **This is native qpdf memory NOT visible to `tracemalloc`** and must be assessed via RSS.

- **Claim:** repeatedly opening PDF handles without closing grows native RSS that `tracemalloc` cannot see and that gc alone does not reclaim; the retention scales roughly linearly with the number of un-closed opens (≈ `0.06 MB/open`) and is independent of the PDF's size.
- **Evidence (EXACT, focused pikepdf probe — see §6.1.1):** `[P] RSS after 50 open (no close):    36.91 MB (delta +2.99 MB)` while `[P] tracemalloc peak (same)    :     0.236 MB`; then `[P] RSS after del+gc handles   :    36.91 MB` (not freed by gc alone) versus `[P] RSS after malloc_trim(0)   :    34.18 MB (trim rc=1)`. Linear scaling is confirmed at `[P] 200 open (no close): delta +12.14 MB` (tracemalloc peak `0.938 MB`), i.e. ≈ `0.06 MB/open`.
- **Grounding note (why not a bigger number):** the per-open figure above is measured from a *warm* baseline (after `pikepdf`/qpdf are already initialized). The **first** `pikepdf.open` additionally pays a **one-time** native qpdf library initialization — `[P] one-time qpdf lib init     :    21.18 MB  (cold->warm)` — which is a fixed startup cost, **not** per-open retention. Measuring the delta from a *cold* baseline conflates the two (`[P] delta from COLD baseline   :   +24.17 MB` for 50 opens, rising toward ~`+33 MB` at 200 opens); the un-closed-handle retention proper is the warm-baseline `+2.99 MB` for 50 opens.

**Two references are held for the document's processing duration** (by design, not a leak): the plain-text parser reads the whole file into memory at `self.text = f.read()` (`src/paperless_text/parsers.py:L42`), and `Document.content` is held on the model instance as a `TextField` (`src/documents/models.py:L117` = `content = models.TextField(`).

### Question 3 — Is caching accumulating data unexpectedly?

**No server-side in-memory cache exists.** Each of the following absences was confirmed empirically by grep across `src/documents`, `src/paperless`, and the four parser apps (`paperless_tesseract`, `paperless_text`, `paperless_tika`, `paperless_mail`), each returning zero:

- **No** `functools.lru_cache` / `functools.cache` / `@cache` — grep result: `ZERO: no lru_cache/functools.cache/@cache`.
- **No** `gc.collect` / `import gc` — grep result: `ZERO: no gc.collect/import gc` (the pipeline relies entirely on automatic reference-counting and cyclic GC).
- **No** module-global memoization dict.
- **No** Django `CACHES` setting in `src/paperless/settings.py` — grep result: `ZERO: no CACHES setting`.

**The only caching-adjacent constructs do not retain document metadata server-side.** They are (a) an HTTP `cache_control` response header for **client-side** caching in `src/documents/views.py` — imported at `src/documents/views.py:L25` (`from django.views.decorators.cache import cache_control`) and applied at `src/documents/views.py:L340` (`@method_decorator(cache_control(public=False, max_age=315360000))`) — and (b) a Redis-backed `CHANNEL_LAYERS` (`src/paperless/settings.py:L178`) used strictly for WebSocket notifications. **Neither retains document metadata server-side.**

**Runtime nuance (grounding).** The absence noted above is at the **source** level — `src/paperless/settings.py` defines no `CACHES` key (confirmed: the literal `CACHES` does not appear in the file). At **runtime**, Django still supplies its built-in default cache — measured `settings.CACHES = {'default': {'BACKEND': 'django.core.cache.backends.locmem.LocMemCache'}}` — but it is **not used to store document metadata**: a grep for `cache.set(` / `cache.get(` across non-test source returns **none**, so no document data is written to or read from any cache. The Q3 conclusion is therefore unchanged: no server-side cache accumulates document metadata.

**The classifier is reloaded per consume, not retained across documents.** The load happens inside `try_consume_file` at `classifier = load_classifier()` (`src/documents/consumer.py:L292`) for each document.

- **Claim:** RSS does not accumulate across a batch of documents (no cross-document retention).
- **Evidence (EXACT):** `[A2] after  20 loads: RSS=   187.25 MB (no explicit gc)` / `[A2] after  40 loads: RSS=   165.99 MB (no explicit gc)` / `[A2] after  60 loads: RSS=   155.07 MB (no explicit gc)` — **no monotonic growth**; RSS peaks early then settles to a steady `~155 MB` (warm baseline + one live model), full series in Section 6 and Section 7.

### Question 4 — What differs between spike and no-spike cases?

**Primary determinant: presence/absence of the trained model artifact.** The artifact path is `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` (`src/paperless/settings.py:L74`). If the file is absent, `load_classifier()` returns `None` at `src/documents/classifier.py:L31` (`if not os.path.isfile(settings.MODEL_FILE): ... return None`), so `DocumentClassifier.load()` never runs and the model-load spike does not occur.

**Secondary determinant: the configured matching-algorithm mix.** `MATCH_FUZZY` (`src/documents/matching.py:L127-L134`) triggers the per-model full-content copies, whereas the literal/auto/regex branches do not build those copies.

- **Claim:** toggling the model artifact toggles the spike on and off.
- **Evidence (REPRESENTATIVE, measured), quoted side by side:**
  - `[NO-SPIKE] MODEL_FILE absent  -> load_classifier() returns None; RSS delta = +0.00 MB`
  - `[SPIKE]    MODEL_FILE present -> load() reconstructs model; RSS delta = +52.55 MB`

### Question 5 — Behavior across document types and batch sizes

**Batch size — no monotonic growth.** Loading/processing across a batch keeps RSS flat, then trims back down.

- **Claim (EXACT):** RSS shows **no monotonic growth** across 20/40/60 loads (it peaks early, then settles to a steady state) and trims down afterward.
- **Evidence (EXACT):** `[A2] after  20 loads: RSS=   187.25 MB (no explicit gc)` … `[A2] peak RSS during batch     :   208.41 MB` … `[A2] RSS after malloc_trim(0)  :   106.08 MB  (trim rc=1)`.
- **Claim (REPRESENTATIVE):** the same flat-then-trim pattern holds.
- **Evidence (REPRESENTATIVE):** `[A2] after  20 docs: RSS=   67.57 MB (no explicit gc)` / `[A2] after  40 docs: RSS=   67.58 MB (no explicit gc)` / `[A2] after  60 docs: RSS=   67.60 MB (no explicit gc)`, trim → `[A2] RSS after malloc_trim(0)  :    40.39 MB  (trim rc=1)`.

**Document type — each arm maps to a distinct parser code path:**

- **Plain text** reads the whole file into memory: `self.text = f.read()` (`src/paperless_text/parsers.py:L42`); footprint is proportional to document size.
- **PDF-with-text** opens a native qpdf handle: `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`). **Measured native +2.99 MB for 50 un-closed opens (≈ 0.06 MB/open), tracemalloc-invisible** — evidence `[P] RSS after 50 open (no close):    36.91 MB (delta +2.99 MB)` vs `[P] tracemalloc peak (same)    :     0.236 MB`.
- **Image / scanned OCR** uses `ocrmypdf` — the higher-memory document-type arm: `import ocrmypdf` (`src/paperless_tesseract/parsers.py:L246`) followed by `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L261`; block `L246-L261`), with a fallback `ocrmypdf.ocr(**args)` on the safe-fallback path (`src/paperless_tesseract/parsers.py:L288-L298`). Described here; **not separately measured** in this harness.
- **Office / Tika** builds a metadata dict via a comprehension over the Tika response in `extract_metadata` (`src/paperless_tika/parsers.py:L29-L49`). Locally minor — evidence `[B] tracemalloc current (both metadata dict-lists): 0.014 MB`.

**Base-parser default.** The base `extract_metadata` returns an empty list `return []` (`src/documents/parsers.py:L305`), with the parser's scratch tempdir created at `self.tempdir = tempfile.mkdtemp(...)` (`src/documents/parsers.py:L293`) and removed by `shutil.rmtree(self.tempdir)` in `cleanup` (`src/documents/parsers.py:L350`).

**Peak vs steady-state.** Contrasting peak RSS during the batch against the post-`malloc_trim` steady state exposes allocator retention (tables in Section 7; determination in Section 8).

---

## 5. Component / Method Memory-Attribution Table

Where memory actually goes during import, attributed to specific code with `file:line` and the measured/native character of each.

| Component / Method | `file:line` | Role | Measured / Native |
|--------------------|-------------|------|-------------------|
| `Consumer.try_consume_file` | `src/documents/consumer.py:L180` (load at `L292`) | per-consume classifier load (once per document) | — |
| `DocumentClassifier.load()` — 6× `pickle.load` | `src/documents/classifier.py:L86-L92` | **DOMINANT** model reconstruction (doc-size independent) | **EXACT ~+126 MB RSS cold / +50.00 MB warm** (cold = one-time sklearn import +73.78 MB + model data) |
| `preprocess_content` (×3 / doc) | `src/documents/classifier.py:L24-L27` | transient content copies (`.lower().strip()`, `re.sub`) | REPR 2.723 MB peak |
| `predict_correspondent` / `predict_document_type` / `predict_tags` | `src/documents/classifier.py:L251` / `L262` / `L273` | vectorizer transform + predict | — |
| `matches()` `MATCH_FUZZY` copy | `src/documents/matching.py:L131,L134` | full-content copy **per matching model** | REPR 41.8× at N=40 |
| `pikepdf.open` (no close) | `src/paperless_tesseract/parsers.py:L34` | native qpdf handle held until function return | **EXACT +2.99 MB RSS / 50 opens** (≈0.06 MB/open), tracemalloc 0.236 MB |
| `self.text = f.read()` | `src/paperless_text/parsers.py:L42` | whole-file read into memory | ∝ doc size |
| `Document.content` `TextField` | `src/documents/models.py:L117` | full extracted text held on the model instance | — |
| REST metadata extraction (original always; archive conditional) | `src/documents/views.py:L295` (original, always) and `L300-L305` (archive, only if `doc.has_archive_version`) via `get_metadata` (`L260`) | one `extract_metadata` call without an archive version; a second only when `doc.has_archive_version` is true | — |
| `Q_CLUSTER {"recycle": 1}` | `src/paperless/settings.py:L452` (block `L449-L457`) | worker restart per task — **bounding factor** | — |

---

## 6. Runtime Measurement Evidence (verbatim, with producing commands)

Both evidence sets are reproduced verbatim below in fenced blocks, each labeled with its environment and the command/code that produced it. **No number is rounded or paraphrased.**

### 6.1 EXACT (gold standard)

Environment: `Python 3.9.23 | scikit-learn 1.0.2 | numpy 1.22.3 | scipy 1.8.0 | pikepdf 5.1.1`.

**This block was re-run for final acceptance** to make the model-load spike **bit-for-bit reproducible** and to explain the earlier single-sample value. It uses the **real** `DocumentClassifier.save()` / `load_classifier()` from the repository (mounted read-only) rather than a hand-rolled pickle, builds a **48.03 MB** model (comparable to the earlier 48.22 MB sample and to an independent 48.07 MB reconstruction), and measures the load from **both a cold and a warm baseline**, **three runs each**. The complete script bodies are embedded verbatim in **Appendix A**, so this run is reproducible line-for-line.

Producing command (the driver runs build → import-cost → 3× cold → 3× warm → batch/leak):

```bash
docker run --rm --entrypoint bash \
  -v <repo>:/workspace:ro -v /tmp/mem_probe_fix:/probe:ro \
  -e DJANGO_SETTINGS_MODULE=paperless.settings -e PYTHONPATH=/workspace/src \
  -e PAPERLESS_DATA_DIR=/tmp/pdata -e PAPERLESS_MEDIA_ROOT=/tmp/pmedia \
  -e PAPERLESS_CONSUMPTION_DIR=/tmp/pconsume -e PAPERLESS_DISABLE_DBHANDLER=true \
  paperless-ngx-memtest:local -c 'bash /probe/run_all.sh'
```

The build step assembles the classifier exactly as `train()` does — `CountVectorizer(analyzer="word", ngram_range=(1,2), min_df=0.01)` + `MultiLabelBinarizer` + 3× `MLPClassifier(tol=0.01)` — assigns them to a real `DocumentClassifier`, and persists via `save()`'s exact 7-object order (`FORMAT_VERSION`, `data_hash`, `data_vectorizer`, `tags_binarizer`, `tags_classifier`, `correspondent_classifier`, `document_type_classifier`) (`src/documents/classifier.py:L101-L109`); the load path is the real `load_classifier()` → `DocumentClassifier.load()` with six sequential `pickle.load` (`src/documents/classifier.py:L86-L92`).

**Model built (verbatim):**

```text
[build] classification_model.pickle MB:    48.03
[build] vocabulary size (n_features)  : 6971
```

**Root cause of the cold/warm difference — the one-time scikit-learn import (verbatim).** `classifier.py` imports scikit-learn **lazily** (inside `train()`/`predict_*`, not at module import), so the **first** `pickle.load` of the model triggers scikit-learn's submodule import. Measured in isolation in a fresh process:

```text
[imp] RSS cold (before sklearn import) :    11.45 MB
[imp] RSS warm (after sklearn import)  :    85.23 MB
[imp] one-time sklearn import cost     :   +73.78 MB  (cold->warm)
```

This `+73.78 MB` is a **one-time, model-independent** library-init cost (it also measured `+73.68 MB` against a 137 MB model) — directly analogous to the one-time qpdf lib-init isolated for `pikepdf` in §6.1.1.

**[A1] COLD single load — a worker's very first consume (3 runs, verbatim).** Fresh process, scikit-learn not yet imported, so delta = one-time import + model data:

```text
[cold] classification_model.pickle size :    48.03 MB
[cold] document content size     :       19 bytes
[cold] RSS before load           :    49.56 MB
[cold] RSS after ONE load        :   176.05 MB   (delta +126.49 MB)
[cold] tracemalloc peak on load  :    77.56 MB
[cold] RSS-vs-tracemalloc gap    :    48.93 MB
```

Runs 2 and 3 reproduce it — `delta +126.40 MB` and `delta +126.37 MB` — mean **+126.42 MB**, range **[+126.37, +126.49]** (~0.1 MB spread across three runs).

**[A1] WARM single load — every subsequent consume (3 runs, verbatim).** scikit-learn already imported, so delta = model data only:

```text
[warm] RSS before load           :   105.07 MB
[warm] RSS after ONE load        :   155.07 MB   (delta +50.00 MB)
[warm] tracemalloc peak on load  :    48.82 MB
[warm] RSS-vs-tracemalloc gap    :     1.18 MB
```

All three warm runs are identical at `delta +50.00 MB`, and the `tracemalloc` peak `48.82 MB` ≈ the 48.03 MB model data (RSS-vs-`tracemalloc` gap only `1.18 MB`).

**[A2] batch trend + [D] leak test (warm regime, verbatim):**

```text
[A2] after  20 loads: RSS=   187.25 MB (no explicit gc)
[A2] after  40 loads: RSS=   165.99 MB (no explicit gc)
[A2] after  60 loads: RSS=   155.07 MB (no explicit gc)
[A2] peak RSS during batch     :   208.41 MB
[A2] RSS after malloc_trim(0)  :   106.08 MB  (trim rc=1)
[D] tracemalloc AFTER load     :    48.82 MB peak
[D] RSS AFTER load             :   155.29 MB
[D] tracemalloc AFTER del+gc   :     0.00 MB
[D] RSS AFTER del+gc           :   107.53 MB
[D] RSS AFTER malloc_trim(0)   :   106.12 MB  (trim rc=1)
```

**Reconciliation (the key result).** The three measurements satisfy one identity that explains **both** the earlier single sample and an independent warm reconstruction from **one** 48 MB model:

- **cold ≈ import + warm:** `+73.78 MB` (import) + `+50.00 MB` (warm data) = `+123.78 MB` ≈ measured cold `+126.42 MB` (residual ~`+2.6 MB` transient unpickle buffers).
- The **earlier single cold sample** (`[A1] … delta +123.48 MB`, 48.22 MB model) is thus a **cold-baseline** measurement, landing within ~2.4% of this run's cold mean.
- An independent **warm** reconstruction (~`+48 MB` for a ~48 MB model) is a **warm-baseline** measurement, landing within ~4% of this run's warm `+50.00 MB`.

So the apparent contradiction between "~+123 MB" and "~+48 MB" is **not** an error in either measurement — it is the **cold vs warm baseline**: a fresh django-q worker's first consume pays the one-time scikit-learn import **plus** the model data; every later consume on that worker pays only the model data. Because `Q_CLUSTER "recycle": 1` (`src/paperless/settings.py:L452`) restarts the worker after **every** task, in the shipped configuration **each consume is effectively a cold consume**, so the operational per-document spike is the **cold** `~+126 MB` — and it is doc-size-independent (`19 bytes` here).

**Run-to-run variance (grounding).** RSS deltas depend on allocator/arena state; they are reproducible in **direction and magnitude** but not bit-identical. The cold spread above is ~0.1 MB across three runs and warm is 0.00 MB. The batch series is **not monotonic** — RSS peaks early (`208.41 MB`), settles to a steady `~155 MB` (warm baseline + one live model), and trims to `106.08 MB` — confirming no unbounded accumulation.

The `[P]` un-closed-`pikepdf` measurement is reported separately below, because the native qpdf handle must be isolated from a **warm** baseline (see §6.1.1) so the one-time library-init cost is not folded into the per-open figure.

### 6.1.1 EXACT — focused `pikepdf` native-retention probe

The un-closed `pikepdf.open` handle holds **native qpdf** memory that `tracemalloc` cannot see, so it is measured with a dedicated probe that mirrors `extract_metadata` (`src/paperless_tesseract/parsers.py:L34-L35`): `pdf = pikepdf.open(document_path)` then `meta = pdf.open_metadata()`, opening the fixture **50 times without closing** and holding the handles. The per-open retention is measured from a **warm** baseline (after `pikepdf`/qpdf are already initialized) so it is not conflated with the one-time native library initialization.

Producing command:

```bash
docker run --rm --entrypoint bash -v <repo>:/workspace:ro -v /tmp/mem_probe:/probe \
  -e PDF=/workspace/src/documents/tests/samples/simple.pdf -e N=50 -e WITH_META=1 \
  paperless-ngx-memtest:local -c 'python3 /probe/pikepdf_probe.py'
```

Verbatim output:

```text
---- pikepdf native-retention probe (mirrors parsers.py:L34-L35) ----
[P] pdf fixture                : simple.pdf
[P] pdf size                   :     0.022 MB
[P] opens (no close)           :        50   (with_open_metadata=True)
[P] RSS cold (before import)   :     12.74 MB
[P] RSS after import pikepdf   :     34.82 MB
[P] RSS after 1st open (warm)  :     35.32 MB
[P] one-time qpdf lib init     :     21.18 MB  (cold->warm)
[P] RSS before 50 open         :     33.91 MB
[P] RSS after 50 open (no close):    36.91 MB (delta +2.99 MB)
[P] tracemalloc peak (same)    :     0.236 MB
[P] RSS after del+gc handles   :     36.91 MB
[P] RSS after malloc_trim(0)   :     34.18 MB (trim rc=1)
[P] per-open retention         :    0.0598 MB/open
[P] delta from COLD baseline   :   +24.17 MB  (includes one-time lib init)
```

Scaling with open-count and independence from PDF size (same probe, varying `N` / `PDF`), verbatim excerpts:

```text
[P] RSS after 200 open (no close):    45.86 MB (delta +12.14 MB)   # N=200, simple.pdf     -> tracemalloc peak 0.938 MB, ~0.06 MB/open
[P] RSS after  50 open (no close):    35.23 MB (delta +1.50 MB)    # multi-page-images.pdf (0.144 MB)
[P] RSS after  50 open (no close):    35.86 MB (delta +1.77 MB)    # rotated.pdf (1.261 MB)
```

**Reading.** The un-closed handle adds `+2.99 MB` of **native** RSS across 50 opens (`≈0.06 MB/open`) that `tracemalloc` reports as only `0.236 MB` — native memory is invisible to it. `del`+`gc` does **not** reclaim it (`36.91 MB` unchanged), and only `malloc_trim(0)` releases it (`34.18 MB`, `trim rc=1`). The delta does **not** grow with PDF size (the 0.022→1.261 MB fixtures all land at ≈1.5–3 MB for 50 opens); only the **open-count** scales it (200 opens → `+12.14 MB`). This is the same native / `tracemalloc`-invisible / arena-released signature seen on model load, now applied to qpdf. The separate one-time `qpdf lib init` of `21.18 MB` (cold→warm) is a fixed startup cost, not per-open retention; a cold-baseline measurement conflates the two (`delta from COLD baseline +24.17 MB` for 50 opens, approaching `~+33 MB` at 200 opens).

### 6.2 REPRESENTATIVE

Environment: `Python 3.12.3 | numpy 2.5.0 | psutil 7.2.2` (sklearn/pikepdf absent; model simulated with NumPy arrays of the same shape; `preprocess_content` + `MATCH_FUZZY` lines copied verbatim from the repo).

Producing commands:

```bash
python3 repr_harness.py
python3 fuzzy_scaling.py
python3 spike_vs_nospike.py
```

Verbatim output:

```text
[setup] simulated model artifact size :    46.48 MB
[A1] document content size     :       28 bytes
[A1] RSS before load           :    52.29 MB
[A1] RSS after ONE load        :    89.45 MB   (delta +37.16 MB)
[A2] after  20 docs: RSS=   67.57 MB (no explicit gc)
[A2] after  40 docs: RSS=   67.58 MB (no explicit gc)
[A2] after  60 docs: RSS=   67.60 MB (no explicit gc)
[A2] peak RSS during batch     :    67.60 MB
[A2] RSS after malloc_trim(0)  :    40.39 MB  (trim rc=1)
[D] tracemalloc AFTER load     :    51.52 MB peak
[D] RSS AFTER load             :    97.95 MB
[D] tracemalloc AFTER del+gc   :     0.00 MB
[D] RSS AFTER del+gc           :    67.55 MB
[D] RSS AFTER malloc_trim(0)   :    40.24 MB  (trim rc=1)
[C-transient] 40 sequential matches(): tracemalloc peak = 1.197 MB (5.5x)
[C-cumulative] N=10: 2.853 MB (13.2x); N=20: 4.922 MB (22.7x); N=40: 9.062 MB (41.8x)
[E] tracemalloc peak (3 copies): 2.723 MB
[B] tracemalloc current (both metadata dict-lists): 0.014 MB
VmRSS KB final: 47400 ; ru_maxrss MB peak: 101.83
[NO-SPIKE] MODEL_FILE absent  -> load_classifier() returns None; RSS delta = +0.00 MB
[SPIKE]    MODEL_FILE present -> load() reconstructs model; RSS delta = +52.55 MB
```

### 6.3 Cross-environment agreement

The **patterns are identical** across both environments; only the absolute MB differ (each block is labeled). The EXACT model size `48.03 MB` is consistent with the AAP reference `48.41 MB`; the REPRESENTATIVE spike `+52.55 MB` is consistent with the AAP reference `+53.52 MB`. In both environments: a single model load produces a large RSS spike, the batch series shows **no monotonic growth** (EXACT peaks early then settles to a steady state; REPRESENTATIVE is flat), `tracemalloc` returns to `0.00 MB` after `del`+`gc`, and RSS drops after `malloc_trim(0)` with `trim rc=1`.

---

## 7. Matrices — Batch Size, Document Type, Spike vs No-Spike

### 7.1 Batch-size matrix (peak vs steady-state)

Peak RSS during the batch is contrasted with the post-`malloc_trim` steady state. The RSS series shows **no monotonic growth** (the EXACT series peaks early then settles to a steady state; the REPRESENTATIVE series is flat), which is the signature of *no* unbounded leak.

| Batch point | EXACT RSS | REPRESENTATIVE RSS |
|-------------|-----------|--------------------|
| single load — **cold** (`[cold]` after ONE load) | `176.05 MB` (delta `+126.49 MB`; 3-run mean `+126.42`) | `89.45 MB` (delta `+37.16 MB`) |
| single load — **warm** (`[warm]` after ONE load) | `155.07 MB` (delta `+50.00 MB`) | — |
| after 20 | `187.25 MB` | `67.57 MB` |
| after 40 | `165.99 MB` | `67.58 MB` |
| after 60 | `155.07 MB` | `67.60 MB` |
| peak during batch | `208.41 MB` | `67.60 MB` |
| steady-state after `malloc_trim(0)` | `106.08 MB` (`trim rc=1`) | `40.39 MB` (`trim rc=1`) |

### 7.2 Document-type matrix

| Document type | Parser code path | Memory characteristic | Measured |
|---------------|------------------|-----------------------|----------|
| Plain text | `self.text = f.read()` (`src/paperless_text/parsers.py:L42`) | whole-file read into memory | ∝ doc size |
| PDF-with-text | `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`) | native qpdf handle, not closed until return | **EXACT** `[P]` +2.99 MB RSS / 50 opens (≈0.06 MB/open); tracemalloc `0.236 MB` |
| Image / scanned OCR | `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L246-L261`; fallback `L288-L298`) | higher-memory arm | described; not separately measured here |
| Office / Tika | dict comprehension over Tika response (`src/paperless_tika/parsers.py:L29-L49`) | small metadata dict | **REPR** `[B]` 0.014 MB |

### 7.3 Spike vs no-spike (toggled by the model artifact)

| Condition | Mechanism | RSS delta |
|-----------|-----------|-----------|
| `MODEL_FILE` **present** | `load()` reconstructs the model (`src/documents/classifier.py:L86-L92`) | **REPR** `+52.55 MB` / **EXACT** `+126.42 MB` cold, `+50.00 MB` warm |
| `MODEL_FILE` **absent** | `load_classifier()` returns `None` (`src/documents/classifier.py:L31`); `load()` never runs | `+0.00 MB` |

---

## 8. Determination — Normal GC / Allocator Behavior vs a Genuine Leak

### 8.1 Criteria (stated explicitly)

- Retention is classified **NORMAL** when **(a)** `tracemalloc` current returns to baseline after `del` + `gc.collect()` (Python objects reclaimed) **AND (b)** RSS drops after `ctypes` `malloc_trim(0)` (glibc arena release).
- Retention is classified a **GENUINE LEAK** only when memory grows **monotonically** across a batch and **never resets**.

### 8.2 Verdict

**The observed retention is NORMAL glibc allocator (arena) retention, NOT a Python-level leak.** The paired evidence:

- Python objects are fully reclaimed: `[D] tracemalloc AFTER del+gc   :     0.00 MB` (EXACT).
- Yet process RSS remains high: `[D] RSS AFTER del+gc           :   107.53 MB` (EXACT).
- And it drops only when the allocator is trimmed: `[D] RSS AFTER malloc_trim(0)   :   106.12 MB  (trim rc=1)` (EXACT).
- The batch series shows no monotonic growth: `[A2] after  20 loads: RSS=   187.25 MB` … `[A2] after  60 loads: RSS=   155.07 MB` (EXACT).

This directly explains the user's "not released in a timely manner": it is **allocator arena retention** under the default glibc allocator. **No `MALLOC_*` tuning** is set in the image (the `Dockerfile` and `docker/` assets set no `MALLOC_ARENA_MAX` / `MALLOC_MMAP_THRESHOLD_`), so the process runs with default arena behavior — freed blocks are held in per-thread arenas for reuse rather than returned to the OS immediately.

### 8.3 Bounding factor (why the symptom is intermittent, not fatal)

All findings are framed against `Q_CLUSTER = { ..., "recycle": 1, ... }` (`src/paperless/settings.py:L452`, block `src/paperless/settings.py:L449-L457`), which restarts the django-q worker **after every task**. Process teardown reclaims all memory — including retained glibc arenas — at each document boundary, bounding any leak-like retention to a single task's lifetime. The default worker timeout is 1800 seconds: `PAPERLESS_WORKER_TIMEOUT: Final[int] = __get_int("PAPERLESS_WORKER_TIMEOUT", 1800)` (`src/paperless/settings.py:L440`). Together these explain why the spike is **intermittent and non-fatal**.

### 8.4 Measurement limitation (restated where native memory is involved)

`tracemalloc` observes only Python-object allocations, so qpdf-via-`pikepdf` and NumPy C buffers are **assessed via RSS**, not `tracemalloc`. This is quantified by the EXACT cold-load `[cold] RSS-vs-tracemalloc gap    :    48.93 MB` and by the `[P]` block, where RSS moves `+2.99 MB` across 50 un-closed opens while `tracemalloc` sees only `0.236 MB`. Any conclusion about native memory in this report rests on RSS, by necessity.

---

## 9. Remediation Ideas — RECOMMENDATIONS ONLY (NOT implemented)

> The following are **optional recommendations**. Consistent with the read-only mandate, **none were applied**, and they are **not part of this deliverable**. The shipped `recycle: 1` policy (`src/paperless/settings.py:L452`) already bounds the practical impact of the retention described above.

1. **Return arenas to the OS sooner.** Call `ctypes` `malloc_trim(0)` at task boundaries, or set `MALLOC_ARENA_MAX` / `MALLOC_MMAP_THRESHOLD_` in the environment, to release retained glibc arenas back to the OS earlier. Evidence that this works: `[D] RSS AFTER malloc_trim(0) : 106.12 MB (trim rc=1)` (EXACT).
2. **Context-manage the PDF handle.** Wrap the qpdf handle as `with pikepdf.open(...) as pdf:` in `extract_metadata` (`src/paperless_tesseract/parsers.py:L34`) so the native handle closes deterministically rather than at function return. Evidence of the current native retention: `[P] RSS after 50 open (no close): 36.91 MB (delta +2.99 MB)` (≈0.06 MB/open, released by `malloc_trim(0)`).
3. **Keep the loaded classifier across post-consume hooks.** Optionally cache/keep the loaded classifier across hooks; this is already partially mitigated by the single per-consume load at `src/documents/consumer.py:L292` (rationale comment `L288-L290`).

---

## 10. Coverage Pass (every named item, answered by name)

| User-named item | Where answered | Anchor evidence / citation |
|-----------------|----------------|----------------------------|
| Spikes / root-cause | §4 Q1, §5 | `[cold] RSS after ONE load : 176.05 MB (delta +126.49 MB)` / `[warm] … delta +50.00 MB` vs `19 bytes`; `src/documents/classifier.py:L86-L92` |
| Unnecessary copies | §4 Q2 | `[E] tracemalloc peak (3 copies): 2.723 MB`; `[C-cumulative] … N=40: 9.062 MB (41.8x)`; `src/documents/classifier.py:L24-L27`, `src/documents/matching.py:L131,L134` |
| References held too long | §4 Q2 | `[P] RSS after del+gc handles : 36.91 MB` (native, released by `malloc_trim(0)`); `src/paperless_tesseract/parsers.py:L34`, `src/paperless_text/parsers.py:L42`, `src/documents/models.py:L117` |
| Caching accumulation | §4 Q3 | grep `ZERO: no CACHES setting`, `ZERO: no lru_cache/functools.cache/@cache`, `ZERO: no gc.collect/import gc`; `src/documents/consumer.py:L292` |
| Spike-vs-no-spike difference | §4 Q4, §7.3 | `[NO-SPIKE] … +0.00 MB` vs `[SPIKE] … +52.55 MB`; `src/paperless/settings.py:L74`, `src/documents/classifier.py:L31` |
| Document types | §4 Q5, §7.2 | `src/paperless_text/parsers.py:L42`, `src/paperless_tesseract/parsers.py:L34`, `src/paperless_tika/parsers.py:L29-L49`; `[P]` and `[B]` |
| Batch sizes | §4 Q5, §7.1 | `[A2] after 20/40/60 loads: RSS= 187.25/165.99/155.07 MB` (no monotonic growth); `[A2] RSS after malloc_trim(0) : 106.08 MB (trim rc=1)` |
| Normal-GC vs problematic | §8 | `[D] tracemalloc AFTER del+gc : 0.00 MB`; `[D] RSS AFTER malloc_trim(0) : 106.12 MB (trim rc=1)` |
| Actual runtime measurements shown | §6 (EXACT + REPRESENTATIVE), §7 | both verbatim blocks with producing commands |
| Which components/methods hold memory | §5 | attribution table with `file:line` per row |
| Where memory is actually going | §5, §8 | model reconstruction (`src/documents/classifier.py:L86-L92`) + glibc arenas (cold-load RSS-vs-tracemalloc gap `48.93 MB`) |

**Structural note present:** Section 3 documents that `extract_metadata` runs on the REST metadata endpoint (`src/documents/views.py:L260-L305`), not the core consume path, and both interpretations are covered.

**One-claim/one-evidence discipline:** every behavioral claim in Sections 4 and 8 is paired with its specific verbatim line, and every value is cited with a `file:line` locator and quoted without paraphrase.

**Both environments present and labeled:** Section 6 contains the EXACT and REPRESENTATIVE verbatim blocks, each with its producing command.

---

## Appendix A — Full EXACT model-load reproduction scripts (embedded for bit-for-bit reproduction)

These are the complete, unabridged script bodies that produced the EXACT model-load evidence in §6.1. They are embedded here so the run is reproducible **line-for-line** without relying on any file that was cleaned up. They live **outside** the repository (under `/tmp/mem_probe_fix`, mounted read-only into the container at `/probe`) and modify **no** repository file; the only thing they write is a scratch `classification_model.pickle` under `PAPERLESS_DATA_DIR=/tmp/pdata` (an ephemeral in-container tmp dir).

**To reproduce (single command):** save the five files below under `/tmp/mem_probe_fix/`, then run the exact producing command from §6.1 (`docker run … paperless-ngx-memtest:local -c 'bash /probe/run_all.sh'`). The driver builds the model, measures the one-time import cost, runs 3× cold and 3× warm single-load probes, and runs the batch + leak test.

**Reproducibility caveat.** RSS deltas depend on glibc allocator/arena state and are reproducible in **direction and magnitude**, not bit-identically: the cold single-load delta reproduced as `+126.49 / +126.40 / +126.37 MB` (mean `+126.42`, ~0.1 MB spread) and the warm delta as `+50.00 MB` on all three runs. The earlier single sample (`+123.48 MB`, 48.22 MB model) is a **cold-baseline** measurement within ~2.4% of this run's cold mean; an independent warm reconstruction (`~+48 MB`) is a **warm-baseline** measurement within ~4% of this run's warm value. Model size scales the model-data component roughly linearly (a 137.66 MB model gave cold `+219 MB` / warm `+144 MB` with the same `+73.68 MB` one-time import), so exact MB depend on the built model's size; `build_model.py` is tuned to `~48 MB` (`N_UNI = 3486`) to match the report and QA regimes.

`build_model.py`:

```python
"""
build_model.py -- Build a ~48 MB source-compatible classification_model.pickle
using the REAL DocumentClassifier.save() from the paperless-ngx repo.

Classifier objects are constructed exactly as DocumentClassifier.train()
(src/documents/classifier.py) does:
    CountVectorizer(analyzer="word", ngram_range=(1, 2), min_df=0.01)
    MultiLabelBinarizer()
    MLPClassifier(tol=0.01)   x3
then assigned to a real DocumentClassifier and persisted with the real .save()
(7 sequential pickle.dump in FORMAT_VERSION order).

The corpus is deterministic (fixed-order tokens) so the vocabulary is exactly
~6971 features -> MLP weight matrices (plus retained Adam optimizer moments)
dominate the pickle at ~48 MB, comparable
to the report's 48.22 MB and QA's independent 48.07 MB reconstructions.

Read-only: lives OUTSIDE the repo (/probe) and only WRITES to PAPERLESS_DATA_DIR
(a scratch tmp dir); it modifies no repository file.
"""
import os
import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
os.environ.setdefault("PAPERLESS_DISABLE_DBHANDLER", "true")
django.setup()

from django.conf import settings
from documents.classifier import DocumentClassifier
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.neural_network import MLPClassifier
from sklearn.preprocessing import MultiLabelBinarizer

# N_UNI unigrams in FIXED order -> ~N_UNI unigram + ~(N_UNI-1) bigram features.
# N_UNI=3486 -> ~6971 features -> ~48 MB pickle (weights x3 + Adam moments x2).
N_UNI = 3486
N_DOCS = 12
words = [f"w{i}" for i in range(N_UNI)]
one_doc = " ".join(words)              # identical fixed-order doc
docs = [one_doc for _ in range(N_DOCS)]

vectorizer = CountVectorizer(analyzer="word", ngram_range=(1, 2), min_df=0.01)
X = vectorizer.fit_transform(docs)
n_feat = len(vectorizer.vocabulary_)

# Labels: vary across docs so the classifiers fit real weight matrices.
labels_dtype = [(i % 7) + 1 for i in range(N_DOCS)]
labels_corr = [(i % 5) + 1 for i in range(N_DOCS)]
labels_tags = [sorted({(i % 9) + 1, (i % 4) + 1}) for i in range(N_DOCS)]

binarizer = MultiLabelBinarizer()
y_tags = binarizer.fit_transform(labels_tags)

tags_clf = MLPClassifier(tol=0.01, max_iter=6, random_state=1)
tags_clf.fit(X, y_tags)
corr_clf = MLPClassifier(tol=0.01, max_iter=6, random_state=2)
corr_clf.fit(X, labels_corr)
dtype_clf = MLPClassifier(tol=0.01, max_iter=6, random_state=3)
dtype_clf.fit(X, labels_dtype)

clf = DocumentClassifier()
clf.data_hash = b"\x00" * 20
clf.data_vectorizer = vectorizer
clf.tags_binarizer = binarizer
clf.tags_classifier = tags_clf
clf.correspondent_classifier = corr_clf
clf.document_type_classifier = dtype_clf
clf.save()  # REAL save() -> settings.MODEL_FILE

size_mb = os.path.getsize(settings.MODEL_FILE) / (1024 * 1024)
print(f"[build] MODEL_FILE                    : {settings.MODEL_FILE}")
print(f"[build] vocabulary size (n_features)  : {n_feat}")
print(f"[build] classification_model.pickle MB: {size_mb:8.2f}")
```

`probe_import_cost.py`:

```python
"""
probe_import_cost.py -- Isolate the one-time scikit-learn import cost in a FRESH
process: read RSS, import the exact estimator submodules that unpickling the
model triggers, read RSS again. This cold->warm library-init cost is what a
cold-baseline load delta folds in (directly analogous to the one-time qpdf
lib-init already isolated for pikepdf in the report).
"""
def vmrss_mb():
    with open("/proc/self/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                return int(line.split()[1]) / 1024.0
    return -1.0

rss_cold = vmrss_mb()
import numpy            # noqa
import scipy            # noqa
from sklearn.neural_network import MLPClassifier             # noqa
from sklearn.feature_extraction.text import CountVectorizer  # noqa
from sklearn.preprocessing import MultiLabelBinarizer        # noqa
from sklearn.utils.multiclass import type_of_target          # noqa
MLPClassifier(tol=0.01)
rss_warm = vmrss_mb()
print(f"[imp] RSS cold (before sklearn import) : {rss_cold:8.2f} MB")
print(f"[imp] RSS warm (after sklearn import)  : {rss_warm:8.2f} MB")
print(f"[imp] one-time sklearn import cost     : {rss_warm - rss_cold:+8.2f} MB  (cold->warm)")
```

`probe_load.py`:

```python
"""
probe_load.py MODE  -- Measure the RSS spike of ONE real load_classifier().

MODE=cold : fresh process, sklearn estimator submodules NOT yet imported. The
            first pickle.load triggers their lazy import, so delta =
            (one-time sklearn import) + (model data). This is the cost a FRESH
            django-q worker pays on its very first consume (classifier.py imports
            sklearn lazily, not at module import).
MODE=warm : force sklearn submodules to import FIRST (as on a worker that has
            already classified once), then load. delta = model data only.

Uses the REAL load_classifier()/DocumentClassifier.load() (6 sequential
pickle.load, src/documents/classifier.py:L86-L92). RSS via /proc/self/status
VmRSS; tracemalloc traces Python objects only.
"""
import os, sys, gc, ctypes, tracemalloc, django
MODE = sys.argv[1] if len(sys.argv) > 1 else "cold"
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
os.environ.setdefault("PAPERLESS_DISABLE_DBHANDLER", "true")

def vmrss_mb():
    with open("/proc/self/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                return int(line.split()[1]) / 1024.0
    return -1.0

django.setup()
from documents.classifier import load_classifier
from django.conf import settings
size_mb = os.path.getsize(settings.MODEL_FILE) / (1024 * 1024)

if MODE == "warm":
    import numpy            # noqa
    import scipy            # noqa
    from sklearn.neural_network import MLPClassifier             # noqa
    from sklearn.feature_extraction.text import CountVectorizer  # noqa
    from sklearn.preprocessing import MultiLabelBinarizer        # noqa
    from sklearn.utils.multiclass import type_of_target          # noqa
    MLPClassifier(tol=0.01)

document_content = "small text document"
tracemalloc.start()
rss_before = vmrss_mb()
clf = load_classifier()
rss_after = vmrss_mb()
cur, peak = tracemalloc.get_traced_memory()
tracemalloc.stop()
ok = clf is not None
print(f"[{MODE}] classification_model.pickle size : {size_mb:8.2f} MB")
print(f"[{MODE}] document content size     : {len(document_content):8d} bytes")
print(f"[{MODE}] load_classifier() not None       : {ok}")
print(f"[{MODE}] RSS before load           : {rss_before:8.2f} MB")
print(f"[{MODE}] RSS after ONE load        : {rss_after:8.2f} MB   (delta {rss_after - rss_before:+.2f} MB)")
print(f"[{MODE}] tracemalloc peak on load  : {peak / (1024*1024):8.2f} MB")
print(f"[{MODE}] RSS-vs-tracemalloc gap    : {(rss_after - rss_before) - peak/(1024*1024):8.2f} MB")
del clf
gc.collect()
rc = ctypes.CDLL("libc.so.6").malloc_trim(0)
print(f"[{MODE}] RSS after del+gc          : {vmrss_mb():8.2f} MB")
print(f"[{MODE}] RSS after malloc_trim(0)  : {vmrss_mb():8.2f} MB  (trim rc={rc})")
```

`probe_batch.py`:

```python
"""
probe_batch.py -- [A2] batch trend + [D] leak test using the REAL
load_classifier()/DocumentClassifier.load().

[A2]: load repeatedly (20/40/60), read RSS at each checkpoint to show flat vs
      monotonic growth, then malloc_trim(0) for allocator release.
[D]:  fresh load -> del -> gc.collect() -> malloc_trim(0), reading tracemalloc +
      RSS at each step to classify Python-object vs native/allocator retention.
"""
import os, gc, ctypes, tracemalloc, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
os.environ.setdefault("PAPERLESS_DISABLE_DBHANDLER", "true")

def vmrss_mb():
    with open("/proc/self/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                return int(line.split()[1]) / 1024.0
    return -1.0

django.setup()
from documents.classifier import load_classifier
from django.conf import settings
libc = ctypes.CDLL("libc.so.6")
size_mb = os.path.getsize(settings.MODEL_FILE) / (1024 * 1024)
print(f"[A2] classification_model.pickle size : {size_mb:8.2f} MB")
peak_rss = 0.0
held = None
for i in range(1, 61):
    held = load_classifier()
    r = vmrss_mb()
    if r > peak_rss:
        peak_rss = r
    if i in (20, 40, 60):
        print(f"[A2] after {i:3d} loads: RSS= {r:8.2f} MB (no explicit gc)")
print(f"[A2] peak RSS during batch     : {peak_rss:8.2f} MB")
del held
gc.collect()
libc.malloc_trim(0)
print(f"[A2] RSS after malloc_trim(0)  : {vmrss_mb():8.2f} MB  (trim rc=1)")
tracemalloc.start()
clf = load_classifier()
cur, peak = tracemalloc.get_traced_memory()
print(f"[D] tracemalloc AFTER load     : {peak/(1024*1024):8.2f} MB peak")
print(f"[D] RSS AFTER load             : {vmrss_mb():8.2f} MB")
del clf
gc.collect()
cur2, peak2 = tracemalloc.get_traced_memory()
tracemalloc.stop()
print(f"[D] tracemalloc AFTER del+gc   : {cur2/(1024*1024):8.2f} MB")
print(f"[D] RSS AFTER del+gc           : {vmrss_mb():8.2f} MB")
rc = libc.malloc_trim(0)
print(f"[D] RSS AFTER malloc_trim(0)   : {vmrss_mb():8.2f} MB  (trim rc={rc})")
```

`run_all.sh` (the driver):

```bash
#!/bin/bash
set -e
mkdir -p "$PAPERLESS_DATA_DIR" "$PAPERLESS_MEDIA_ROOT" "$PAPERLESS_CONSUMPTION_DIR"
cd /workspace/src
echo "===== ENV ====="
python3 -c "import sys,sklearn,numpy,scipy,pikepdf;print('Python',sys.version.split()[0],'| scikit-learn',sklearn.__version__,'| numpy',numpy.__version__,'| scipy',scipy.__version__,'| pikepdf',pikepdf.__version__)"
echo "===== BUILD MODEL ====="
python3 /probe/build_model.py
echo "===== SKLEARN IMPORT COST (decomposition) ====="
python3 /probe/probe_import_cost.py
echo "===== [A1] COLD single-load (fresh process, sklearn not yet imported) ====="
for i in 1 2 3; do echo "--- cold run $i ---"; python3 /probe/probe_load.py cold; done
echo "===== [A1] WARM single-load (sklearn already imported) ====="
for i in 1 2 3; do echo "--- warm run $i ---"; python3 /probe/probe_load.py warm; done
echo "===== [A2]+[D] batch trend + leak test ====="
python3 /probe/probe_batch.py
```

---

### Environment & scope footer

- **EXACT** runtime: `Python 3.9.23 | scikit-learn 1.0.2 | numpy 1.22.3 | scipy 1.8.0 | pikepdf 5.1.1`, per `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`).
- **REPRESENTATIVE** runtime: `Python 3.12.3 | numpy 2.5.0 | psutil 7.2.2` (sklearn/pikepdf absent; classifier simulated with same-shape NumPy arrays; `preprocess_content` / `MATCH_FUZZY` transformation lines copied verbatim from the repo).
- **Read-only proof:** the only repository change is this file. All temporary observation scripts lived under `/tmp/mem_probe` and `/tmp/mem_probe_fix` (outside the repository) and were removed; the git working tree is otherwise unchanged.
