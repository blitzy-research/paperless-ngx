# Paperless-ngx Import Memory Investigation — Diagnostic Report

> **Deliverable identity.** This is the single, only file added to the repository for this investigation. The filename is derived from the **source branch** mandated by the AAP — `paperless-ngx_542221a38dff` — so the deliverable is named exactly `paperless-ngx_542221a38dff.md`, placed in `blitzy/documentation/`. This source-branch name is distinct from the current implementation branch: `git rev-parse --abbrev-ref HEAD` returns `blitzy-680a32bd-58ed-4a69-b43f-ef1a0acdff8b` (the working branch), not the source-branch name.
>
> **Scope: READ-ONLY.** Governed by the `SWE-AtlasQnA-Repo` rule set. No existing repository file was modified, created, or deleted; no code remediation was implemented. All temporary observation scripts lived outside the repository (under `/tmp/mem_probe`) and were removed. The git working tree ends unchanged except for this one `.md` file.

---

## Executive Summary (verdict up front)

- **The "disproportionate" spike is the classifier model reconstruction, not the document.** Loading the trained scikit-learn model via six sequential `pickle.load` calls in `DocumentClassifier.load()` (`src/documents/classifier.py:L86-L92`) adds **+123.48 MB RSS** for a **28-byte** document — quote `[A1] RSS after ONE load : 219.92 MB (delta +123.48 MB)` against `[A1] document content size : 28 bytes` (EXACT). The spike is driven by model size, which is **independent of the current document's size**.
- **The "not released in a timely manner" symptom is normal glibc allocator (arena) retention — NOT a Python leak.** After `del` + `gc.collect()`, Python-object memory returns to baseline — `[D] tracemalloc AFTER del+gc : 0.00 MB` — while process RSS stays high — `[D] RSS AFTER del+gc : 221.05 MB` — and only drops when the allocator is trimmed — `[D] RSS AFTER malloc_trim(0) : 180.10 MB (trim rc=1)` (EXACT).
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

**Critical measurement limitation.** `tracemalloc` does **NOT** observe C-extension native allocations — specifically qpdf via `pikepdf` and NumPy's C buffers. Those are **only** visible via RSS. Wherever native memory is discussed, this limitation is restated, and the gap is quantified (e.g., EXACT `[A1] RSS-vs-tracemalloc gap : 42.84 MB`, and the `[P]` block below where RSS moves +32.28 MB while `tracemalloc` sees 0.223 MB).

**Two environments were used and are labeled throughout:**

| Label | Environment | Purpose |
|-------|-------------|---------|
| **EXACT** (gold standard) | Docker `python:3.9-slim-bullseye` — the runtime baseline per `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`) — with the real pinned stack `scikit-learn==1.0.2`, `pikepdf==5.1.1`, `numpy==1.22.3`, `scipy==1.8.0`. Reported env line: `Python 3.9.23 \| scikit-learn 1.0.2 \| numpy 1.22.3 \| scipy 1.8.0 \| pikepdf 5.1.1`. | Full-stack, faithful reproduction with the real model and real qpdf handle. |
| **REPRESENTATIVE** | Sandbox `Python 3.12.3` + `numpy 2.5.0` + `psutil 7.2.2` (sklearn/pikepdf absent). The classifier is simulated with NumPy arrays of the **same shape**, and the `preprocess_content` / `MATCH_FUZZY` transformation lines are copied **verbatim** from the repo. | Reproduces the allocation *patterns* where the pinned native wheels cannot be built. |

Cross-environment agreement is noted where relevant: the **patterns are identical**; only the absolute MB differ by environment. The EXACT 48.22 MB model is consistent with the AAP reference of 48.41 MB, and the REPRESENTATIVE spike of +52.55 MB is consistent with the AAP reference of +53.52 MB.

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

- **Claim:** a real ~48 MB model load produces a large RSS spike that is unrelated to document size.
- **Evidence (EXACT):** `[A1] RSS after ONE load        :   219.92 MB   (delta +123.48 MB)` measured against `[A1] document content size     :       28 bytes` and `[save]  classification_model.pickle size :    48.22 MB`.
- **Conclusion:** the spike is driven by **model reconstruction**, independent of document size — a 28-byte document still triggers ~+123 MB RSS.

**The classification/rule-matching stage is the rest of the "metadata processing" work.** In `src/documents/classifier.py`: `predict_correspondent` (`L251`), `predict_document_type` (`L262`), `predict_tags` (`L273`). Rule matching lives in `src/documents/matching.py`: `matches()` (`src/documents/matching.py:L60`), driven from `match_correspondents` (`src/documents/matching.py:L21`), `match_document_types` (`src/documents/matching.py:L34`), and `match_tags` (`src/documents/matching.py:L47`).

### Question 2 — Unnecessary copies / references held too long

**`preprocess_content` makes two transient copies per call, and runs 3× per document.** `preprocess_content` (`src/documents/classifier.py:L24-L27`) executes `content = content.lower().strip()` (`src/documents/classifier.py:L25`) and `content = re.sub(r"\s+", " ", content)` (`src/documents/classifier.py:L26`). Each of `predict_correspondent`/`predict_document_type`/`predict_tags` calls `self.data_vectorizer.transform([preprocess_content(content)])`, so it runs **three times per document**.

- **Claim:** the three per-document preprocess copies are modest.
- **Evidence (REPRESENTATIVE):** `[E] tracemalloc peak (3 copies): 2.723 MB`.

**The `MATCH_FUZZY` branch builds a full-content copy per matching model.** In `matches()` the fuzzy branch (`src/documents/matching.py:L127-L134`) computes `text = re.sub(r"[^\w\s]", "", document_content)` (`src/documents/matching.py:L131`) and then `text = text.lower()` (`src/documents/matching.py:L134`). Because `matches()` is invoked once **per** `Correspondent`/`DocumentType`/`Tag` object via `filter(lambda o: matches(o, document) ...)` over `.objects.all()` (`match_correspondents` `src/documents/matching.py:L21`, `match_document_types` `src/documents/matching.py:L34`, `match_tags` `src/documents/matching.py:L47`), the copies scale with the number of matching objects.

- **Claim:** fuzzy copies scale roughly linearly with the number of matching models.
- **Evidence (REPRESENTATIVE):** `[C-cumulative] N=10: 2.853 MB (13.2x); N=20: 4.922 MB (22.7x); N=40: 9.062 MB (41.8x)` (of a 0.217 MB content); and in the faithful transient case where each copy is freed per call, `[C-transient] 40 sequential matches(): tracemalloc peak = 1.197 MB (5.5x)`.

**The native PDF handle is never explicitly closed.** In `extract_metadata` (`src/paperless_tesseract/parsers.py:L26`), the handle `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`) is **never explicitly closed** before `return result`; it is released non-deterministically when the local goes out of scope at function return. **This is native qpdf memory NOT visible to `tracemalloc`** and must be assessed via RSS.

- **Claim:** repeatedly opening PDF handles without closing grows native RSS that `tracemalloc` cannot see, and that gc alone does not reclaim.
- **Evidence (EXACT):** `[P] RSS after 50 open (no close):   214.55 MB (delta +32.28 MB)` while `[P] tracemalloc peak (same)    :     0.223 MB`; then `[P] RSS after del+gc handles   :   214.55 MB` (not freed by gc alone) versus `[P] RSS after malloc_trim(0)   :   183.46 MB (trim rc=1)`.

**Two references are held for the document's processing duration** (by design, not a leak): the plain-text parser reads the whole file into memory at `self.text = f.read()` (`src/paperless_text/parsers.py:L42`), and `Document.content` is held on the model instance as a `TextField` (`src/documents/models.py:L117` = `content = models.TextField(`).

### Question 3 — Is caching accumulating data unexpectedly?

**No server-side in-memory cache exists.** Each of the following absences was confirmed empirically by grep across `src/documents`, `src/paperless`, and the four parser apps (`paperless_tesseract`, `paperless_text`, `paperless_tika`, `paperless_mail`), each returning zero:

- **No** `functools.lru_cache` / `functools.cache` / `@cache` — grep result: `ZERO: no lru_cache/functools.cache/@cache`.
- **No** `gc.collect` / `import gc` — grep result: `ZERO: no gc.collect/import gc` (the pipeline relies entirely on automatic reference-counting and cyclic GC).
- **No** module-global memoization dict.
- **No** Django `CACHES` setting in `src/paperless/settings.py` — grep result: `ZERO: no CACHES setting`.

**The only caching-adjacent constructs do not retain document metadata server-side.** They are (a) an HTTP `cache_control` response header for **client-side** caching in `src/documents/views.py` — imported at `src/documents/views.py:L25` (`from django.views.decorators.cache import cache_control`) and applied at `src/documents/views.py:L340` (`@method_decorator(cache_control(public=False, max_age=315360000))`) — and (b) a Redis-backed `CHANNEL_LAYERS` (`src/paperless/settings.py:L178`) used strictly for WebSocket notifications. **Neither retains document metadata server-side.**

**The classifier is reloaded per consume, not retained across documents.** The load happens inside `try_consume_file` at `classifier = load_classifier()` (`src/documents/consumer.py:L292`) for each document.

- **Claim:** RSS does not accumulate across a batch of documents (no cross-document retention).
- **Evidence (EXACT):** `[A2] after  20 loads: RSS=  250.66 MB (no explicit gc)` / `[A2] after  40 loads: RSS=  250.66 MB (no explicit gc)` / `[A2] after  60 loads: RSS=  250.66 MB (no explicit gc)` — flat, no monotonic growth (full series in Section 6 and Section 7).

### Question 4 — What differs between spike and no-spike cases?

**Primary determinant: presence/absence of the trained model artifact.** The artifact path is `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` (`src/paperless/settings.py:L74`). If the file is absent, `load_classifier()` returns `None` at `src/documents/classifier.py:L31` (`if not os.path.isfile(settings.MODEL_FILE): ... return None`), so `DocumentClassifier.load()` never runs and the model-load spike does not occur.

**Secondary determinant: the configured matching-algorithm mix.** `MATCH_FUZZY` (`src/documents/matching.py:L127-L134`) triggers the per-model full-content copies, whereas the literal/auto/regex branches do not build those copies.

- **Claim:** toggling the model artifact toggles the spike on and off.
- **Evidence (REPRESENTATIVE, measured), quoted side by side:**
  - `[NO-SPIKE] MODEL_FILE absent  -> load_classifier() returns None; RSS delta = +0.00 MB`
  - `[SPIKE]    MODEL_FILE present -> load() reconstructs model; RSS delta = +52.55 MB`

### Question 5 — Behavior across document types and batch sizes

**Batch size — no monotonic growth.** Loading/processing across a batch keeps RSS flat, then trims back down.

- **Claim (EXACT):** RSS is flat across 20/40/60 loads and trims down afterward.
- **Evidence (EXACT):** `[A2] after  20 loads: RSS=  250.66 MB (no explicit gc)` … `[A2] peak RSS during batch     :   250.66 MB` … `[A2] RSS after malloc_trim(0)  :   180.04 MB  (trim rc=1)`.
- **Claim (REPRESENTATIVE):** the same flat-then-trim pattern holds.
- **Evidence (REPRESENTATIVE):** `[A2] after  20 docs: RSS=   67.57 MB (no explicit gc)` / `[A2] after  40 docs: RSS=   67.58 MB (no explicit gc)` / `[A2] after  60 docs: RSS=   67.60 MB (no explicit gc)`, trim → `[A2] RSS after malloc_trim(0)  :    40.39 MB  (trim rc=1)`.

**Document type — each arm maps to a distinct parser code path:**

- **Plain text** reads the whole file into memory: `self.text = f.read()` (`src/paperless_text/parsers.py:L42`); footprint is proportional to document size.
- **PDF-with-text** opens a native qpdf handle: `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`). **Measured native +32.28 MB, tracemalloc-invisible** — evidence `[P] RSS after 50 open (no close):   214.55 MB (delta +32.28 MB)` vs `[P] tracemalloc peak (same)    :     0.223 MB`.
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
| `DocumentClassifier.load()` — 6× `pickle.load` | `src/documents/classifier.py:L86-L92` | **DOMINANT** model reconstruction (doc-size independent) | **EXACT +123.48 MB RSS** |
| `preprocess_content` (×3 / doc) | `src/documents/classifier.py:L24-L27` | transient content copies (`.lower().strip()`, `re.sub`) | REPR 2.723 MB peak |
| `predict_correspondent` / `predict_document_type` / `predict_tags` | `src/documents/classifier.py:L251` / `L262` / `L273` | vectorizer transform + predict | — |
| `matches()` `MATCH_FUZZY` copy | `src/documents/matching.py:L131,L134` | full-content copy **per matching model** | REPR 41.8× at N=40 |
| `pikepdf.open` (no close) | `src/paperless_tesseract/parsers.py:L34` | native qpdf handle held until function return | **EXACT +32.28 MB RSS**, tracemalloc 0.223 MB |
| `self.text = f.read()` | `src/paperless_text/parsers.py:L42` | whole-file read into memory | ∝ doc size |
| `Document.content` `TextField` | `src/documents/models.py:L117` | full extracted text held on the model instance | — |
| REST metadata extraction (original always; archive conditional) | `src/documents/views.py:L295` (original, always) and `L300-L305` (archive, only if `doc.has_archive_version`) via `get_metadata` (`L260`) | one `extract_metadata` call without an archive version; a second only when `doc.has_archive_version` is true | — |
| `Q_CLUSTER {"recycle": 1}` | `src/paperless/settings.py:L452` (block `L449-L457`) | worker restart per task — **bounding factor** | — |

---

## 6. Runtime Measurement Evidence (verbatim, with producing commands)

Both evidence sets are reproduced verbatim below in fenced blocks, each labeled with its environment and the command/code that produced it. **No number is rounded or paraphrased.**

### 6.1 EXACT (gold standard)

Environment: `Python 3.9.23 | scikit-learn 1.0.2 | numpy 1.22.3 | scipy 1.8.0 | pikepdf 5.1.1`.

Producing command:

```bash
docker run --rm -v /tmp/mem_probe:/work -w /work python:3.9-slim-bullseye bash -c \
  "pip install --no-cache-dir numpy==1.22.3 scipy==1.8.0 joblib==1.1.0 threadpoolctl==3.1.0 scikit-learn==1.0.2 pikepdf==5.1.1 psutil && python exact_repro.py"
```

The script trains a real `CountVectorizer(analyzer="word", ngram_range=(1,2), min_df=0.01)` + `MultiLabelBinarizer` + 3× `MLPClassifier(tol=0.01)`, pickles them in the exact 7-object order of `save()` — `FORMAT_VERSION`, `data_hash`, `data_vectorizer`, `tags_binarizer`, `tags_classifier`, `correspondent_classifier`, `document_type_classifier` (`src/documents/classifier.py:L101-L109`) — then loads via six sequential `pickle.load` mirroring `load()` (`src/documents/classifier.py:L86-L92`). Verbatim output:

```text
[train] REAL model: n_features(vocab)=6000, docs=600, hidden_layer_sizes(default)=(100,)
[save]  classification_model.pickle size :    48.22 MB
[A1] document content size     :       28 bytes
[A1] RSS before load           :    96.44 MB
[A1] RSS after ONE load        :   219.92 MB   (delta +123.48 MB)
[A1] tracemalloc peak on load  :    80.64 MB
[A1] RSS-vs-tracemalloc gap    :    42.84 MB
[A2] after  20 loads: RSS=  250.66 MB (no explicit gc)
[A2] after  40 loads: RSS=  250.66 MB (no explicit gc)
[A2] after  60 loads: RSS=  250.66 MB (no explicit gc)
[A2] peak RSS during batch     :   250.66 MB
[A2] RSS after malloc_trim(0)  :   180.04 MB  (trim rc=1)
[D] tracemalloc AFTER load     :    80.63 MB peak
[D] RSS AFTER load             :   309.05 MB
[D] tracemalloc AFTER del+gc   :     0.00 MB
[D] RSS AFTER del+gc           :   221.05 MB
[D] RSS AFTER malloc_trim(0)   :   180.10 MB  (trim rc=1)
[P] sample.pdf size            :    0.047 MB
[P] RSS after 50 open (no close):   214.55 MB (delta +32.28 MB)
[P] tracemalloc peak (same)    :     0.223 MB
[P] RSS after del+gc handles   :   214.55 MB
[P] RSS after malloc_trim(0)   :   183.46 MB (trim rc=1)
VmRSS KB (final): 187860 ; ru_maxrss MB (peak): 307.54
```

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

The **patterns are identical** across both environments; only the absolute MB differ (each block is labeled). The EXACT model size `48.22 MB` is consistent with the AAP reference `48.41 MB`; the REPRESENTATIVE spike `+52.55 MB` is consistent with the AAP reference `+53.52 MB`. In both environments: a single model load produces a large RSS spike, the batch series is flat (no monotonic growth), `tracemalloc` returns to `0.00 MB` after `del`+`gc`, and RSS drops after `malloc_trim(0)` with `trim rc=1`.

---

## 7. Matrices — Batch Size, Document Type, Spike vs No-Spike

### 7.1 Batch-size matrix (peak vs steady-state)

Peak RSS during the batch is contrasted with the post-`malloc_trim` steady state. The RSS series is **flat** (no monotonic growth), which is the signature of *no* unbounded leak.

| Batch point | EXACT RSS | REPRESENTATIVE RSS |
|-------------|-----------|--------------------|
| single load (`[A1]` after ONE load) | `219.92 MB` (delta `+123.48 MB`) | `89.45 MB` (delta `+37.16 MB`) |
| after 20 | `250.66 MB` | `67.57 MB` |
| after 40 | `250.66 MB` | `67.58 MB` |
| after 60 | `250.66 MB` | `67.60 MB` |
| peak during batch | `250.66 MB` | `67.60 MB` |
| steady-state after `malloc_trim(0)` | `180.04 MB` (`trim rc=1`) | `40.39 MB` (`trim rc=1`) |

### 7.2 Document-type matrix

| Document type | Parser code path | Memory characteristic | Measured |
|---------------|------------------|-----------------------|----------|
| Plain text | `self.text = f.read()` (`src/paperless_text/parsers.py:L42`) | whole-file read into memory | ∝ doc size |
| PDF-with-text | `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`) | native qpdf handle, not closed until return | **EXACT** `[P]` +32.28 MB RSS; tracemalloc `0.223 MB` |
| Image / scanned OCR | `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:L246-L261`; fallback `L288-L298`) | higher-memory arm | described; not separately measured here |
| Office / Tika | dict comprehension over Tika response (`src/paperless_tika/parsers.py:L29-L49`) | small metadata dict | **REPR** `[B]` 0.014 MB |

### 7.3 Spike vs no-spike (toggled by the model artifact)

| Condition | Mechanism | RSS delta |
|-----------|-----------|-----------|
| `MODEL_FILE` **present** | `load()` reconstructs the model (`src/documents/classifier.py:L86-L92`) | **REPR** `+52.55 MB` / **EXACT** `+123.48 MB` |
| `MODEL_FILE` **absent** | `load_classifier()` returns `None` (`src/documents/classifier.py:L31`); `load()` never runs | `+0.00 MB` |

---

## 8. Determination — Normal GC / Allocator Behavior vs a Genuine Leak

### 8.1 Criteria (stated explicitly)

- Retention is classified **NORMAL** when **(a)** `tracemalloc` current returns to baseline after `del` + `gc.collect()` (Python objects reclaimed) **AND (b)** RSS drops after `ctypes` `malloc_trim(0)` (glibc arena release).
- Retention is classified a **GENUINE LEAK** only when memory grows **monotonically** across a batch and **never resets**.

### 8.2 Verdict

**The observed retention is NORMAL glibc allocator (arena) retention, NOT a Python-level leak.** The paired evidence:

- Python objects are fully reclaimed: `[D] tracemalloc AFTER del+gc   :     0.00 MB` (EXACT).
- Yet process RSS remains high: `[D] RSS AFTER del+gc           :   221.05 MB` (EXACT).
- And it drops only when the allocator is trimmed: `[D] RSS AFTER malloc_trim(0)   :   180.10 MB  (trim rc=1)` (EXACT).
- The batch series shows no monotonic growth: `[A2] after  20 loads: RSS=  250.66 MB` … `[A2] after  60 loads: RSS=  250.66 MB` (EXACT).

This directly explains the user's "not released in a timely manner": it is **allocator arena retention** under the default glibc allocator. **No `MALLOC_*` tuning** is set in the image (the `Dockerfile` and `docker/` assets set no `MALLOC_ARENA_MAX` / `MALLOC_MMAP_THRESHOLD_`), so the process runs with default arena behavior — freed blocks are held in per-thread arenas for reuse rather than returned to the OS immediately.

### 8.3 Bounding factor (why the symptom is intermittent, not fatal)

All findings are framed against `Q_CLUSTER = { ..., "recycle": 1, ... }` (`src/paperless/settings.py:L452`, block `src/paperless/settings.py:L449-L457`), which restarts the django-q worker **after every task**. Process teardown reclaims all memory — including retained glibc arenas — at each document boundary, bounding any leak-like retention to a single task's lifetime. The default worker timeout is 1800 seconds: `PAPERLESS_WORKER_TIMEOUT: Final[int] = __get_int("PAPERLESS_WORKER_TIMEOUT", 1800)` (`src/paperless/settings.py:L440`). Together these explain why the spike is **intermittent and non-fatal**.

### 8.4 Measurement limitation (restated where native memory is involved)

`tracemalloc` observes only Python-object allocations, so qpdf-via-`pikepdf` and NumPy C buffers are **assessed via RSS**, not `tracemalloc`. This is quantified by the EXACT `[A1] RSS-vs-tracemalloc gap    :    42.84 MB` and by the `[P]` block, where RSS moves `+32.28 MB` while `tracemalloc` sees only `0.223 MB`. Any conclusion about native memory in this report rests on RSS, by necessity.

---

## 9. Remediation Ideas — RECOMMENDATIONS ONLY (NOT implemented)

> The following are **optional recommendations**. Consistent with the read-only mandate, **none were applied**, and they are **not part of this deliverable**. The shipped `recycle: 1` policy (`src/paperless/settings.py:L452`) already bounds the practical impact of the retention described above.

1. **Return arenas to the OS sooner.** Call `ctypes` `malloc_trim(0)` at task boundaries, or set `MALLOC_ARENA_MAX` / `MALLOC_MMAP_THRESHOLD_` in the environment, to release retained glibc arenas back to the OS earlier. Evidence that this works: `[D] RSS AFTER malloc_trim(0) : 180.10 MB (trim rc=1)` (EXACT).
2. **Context-manage the PDF handle.** Wrap the qpdf handle as `with pikepdf.open(...) as pdf:` in `extract_metadata` (`src/paperless_tesseract/parsers.py:L34`) so the native handle closes deterministically rather than at function return. Evidence of the current native retention: `[P] RSS after 50 open (no close): 214.55 MB (delta +32.28 MB)`.
3. **Keep the loaded classifier across post-consume hooks.** Optionally cache/keep the loaded classifier across hooks; this is already partially mitigated by the single per-consume load at `src/documents/consumer.py:L292` (rationale comment `L288-L290`).

---

## 10. Coverage Pass (every named item, answered by name)

| User-named item | Where answered | Anchor evidence / citation |
|-----------------|----------------|----------------------------|
| Spikes / root-cause | §4 Q1, §5 | `[A1] RSS after ONE load : 219.92 MB (delta +123.48 MB)` vs `28 bytes`; `src/documents/classifier.py:L86-L92` |
| Unnecessary copies | §4 Q2 | `[E] tracemalloc peak (3 copies): 2.723 MB`; `[C-cumulative] … N=40: 9.062 MB (41.8x)`; `src/documents/classifier.py:L24-L27`, `src/documents/matching.py:L131,L134` |
| References held too long | §4 Q2 | `[P] RSS after del+gc handles : 214.55 MB`; `src/paperless_tesseract/parsers.py:L34`, `src/paperless_text/parsers.py:L42`, `src/documents/models.py:L117` |
| Caching accumulation | §4 Q3 | grep `ZERO: no CACHES setting`, `ZERO: no lru_cache/functools.cache/@cache`, `ZERO: no gc.collect/import gc`; `src/documents/consumer.py:L292` |
| Spike-vs-no-spike difference | §4 Q4, §7.3 | `[NO-SPIKE] … +0.00 MB` vs `[SPIKE] … +52.55 MB`; `src/paperless/settings.py:L74`, `src/documents/classifier.py:L31` |
| Document types | §4 Q5, §7.2 | `src/paperless_text/parsers.py:L42`, `src/paperless_tesseract/parsers.py:L34`, `src/paperless_tika/parsers.py:L29-L49`; `[P]` and `[B]` |
| Batch sizes | §4 Q5, §7.1 | `[A2] after 20/40/60 loads: RSS= 250.66 MB`; `[A2] RSS after malloc_trim(0) : 180.04 MB (trim rc=1)` |
| Normal-GC vs problematic | §8 | `[D] tracemalloc AFTER del+gc : 0.00 MB`; `[D] RSS AFTER malloc_trim(0) : 180.10 MB (trim rc=1)` |
| Actual runtime measurements shown | §6 (EXACT + REPRESENTATIVE), §7 | both verbatim blocks with producing commands |
| Which components/methods hold memory | §5 | attribution table with `file:line` per row |
| Where memory is actually going | §5, §8 | model reconstruction (`classifier.py:L86-L92`) + glibc arenas (RSS-vs-tracemalloc gap `42.84 MB`) |

**Structural note present:** Section 3 documents that `extract_metadata` runs on the REST metadata endpoint (`src/documents/views.py:L260-L305`), not the core consume path, and both interpretations are covered.

**One-claim/one-evidence discipline:** every behavioral claim in Sections 4 and 8 is paired with its specific verbatim line, and every value is cited with a `file:line` locator and quoted without paraphrase.

**Both environments present and labeled:** Section 6 contains the EXACT and REPRESENTATIVE verbatim blocks, each with its producing command.

---

### Environment & scope footer

- **EXACT** runtime: `Python 3.9.23 | scikit-learn 1.0.2 | numpy 1.22.3 | scipy 1.8.0 | pikepdf 5.1.1`, per `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`).
- **REPRESENTATIVE** runtime: `Python 3.12.3 | numpy 2.5.0 | psutil 7.2.2` (sklearn/pikepdf absent; classifier simulated with same-shape NumPy arrays; `preprocess_content` / `MATCH_FUZZY` transformation lines copied verbatim from the repo).
- **Read-only proof:** the only repository change is this file. All temporary observation scripts lived under `/tmp/mem_probe` (outside the repository) and were removed; the git working tree is otherwise unchanged.
