# Paperless‑ngx Document‑Import Memory Investigation

**A code‑grounded, measurement‑backed diagnosis of memory spikes during the document consumption pipeline**

---

## 1. Scope, Method, and Environment

### 1.1 What was investigated

This document answers a specific set of questions about why memory consumption spikes — sometimes *disproportionately to document size* — while Paperless‑ngx imports (consumes) documents, and why that memory is *not always returned to the operating system promptly*. The user asked, verbatim:

> "Can you figure out what's causing these memory spikes? I'm curious whether there's something in the metadata handling that's creating unnecessary copies or holding references longer than needed, or if there's caching behavior that's accumulating data unexpectedly. What's different between the cases where memory spikes and cases where it doesn't? How does the behavior change with different document types or batch sizes?"
>
> "I need to see actual runtime memory measurements during processing, understand which components or methods are responsible for holding onto memory, and get evidence showing whether this is normal Python garbage collection behavior or something problematic."

This is a **read‑only diagnostic engagement**. No application source file was modified. Every measurement below was produced by *throwaway* profiling scripts that drove the **real** consumption pipeline and were deleted on completion; the only artifact committed to the repository is this Markdown document.

### 1.2 The rule we held ourselves to

**Code is the source of truth.** Every claim that follows is backed either by a `file:line` citation against this commit, or by a measured runtime number. Where we reason, we show the reasoning.

### 1.3 Environment (measurements are specific to these pins)

| Property | Value |
|---|---|
| Commit | `542221a38dff06361e07976452f9aea24d210542` |
| Python | 3.9.23 (production base `python:3.9-slim-bullseye`, `Dockerfile:L18`) |
| Task queue | Django‑Q **1.3.9** (not Celery), Redis broker `redis==3.5.3` |
| ML stack | `scikit-learn==1.0.2`, `numpy==1.22.3`, `scipy==1.8.0`, `joblib==1.1.0` |
| PDF/OCR | `pikepdf==5.1.1`, `ocrmypdf==13.4.3`, `pdf2image==1.16.0` |
| Index / office | `whoosh==2.7.4`, `tika==1.24` |
| DB / process model | SQLite scratch DB; document consumption runs in the **Django‑Q `qcluster` worker**, not the Gunicorn web process |
| Measurement tool | `psutil==7.2.2` + std‑lib `tracemalloc` / `gc` / `resource` / `ctypes` (used transiently; **never** added to `Pipfile`/`requirements.txt`) |

The consumption code executes inside the `qcluster` worker because `consume_file` is a Django‑Q task (`src/documents/tasks.py:L184`); the worker is configured with `"recycle": 1` (`src/paperless/settings.py:L452`), which is central to the "release" question (Section 7).

### 1.4 Measurement methodology — a mandatory triple lens

No single tool tells the whole story, so every experiment sampled **three lenses** around each pipeline stage:

1. **Process RSS via `psutil`** — `psutil.Process().memory_info().rss`. This is the **authoritative total** and is **mandatory for any memory `tracemalloc` cannot see** — above all the **qpdf/`pikepdf` C++** allocations (Section 6.6) and the poppler/`pdf2image` rasterizer (Section 6.5). (NumPy is a deliberate **exception**: in this environment `numpy 1.22.3` registers its allocations with `tracemalloc` via `np.lib.tracemalloc_domain`[^np-tracemalloc], so the classifier's weight matrices are visible in *both* lenses — Section 8a. We therefore never assume "C extension ⇒ invisible to `tracemalloc`"; we report what each lens actually shows.) It answers *"how much."*
2. **Python allocations via `tracemalloc`** — `tracemalloc.start()`, paired `take_snapshot()`, `compare_to('lineno')` for `file:line` attribution, and `get_traced_memory()` for the current/peak Python heap.[^py-tracemalloc] It answers *"where, in Python."*
3. **Object accounting via `gc`** — `len(gc.get_objects())` for object‑count growth and `gc.collect()`'s return value for collectable reference **cycles**. It answers *"is anything leaking through cycles."*

We also used `resource.getrusage(RUSAGE_SELF).ru_maxrss` as a peak cross‑check and `ctypes` `malloc_trim(0)` to demonstrate return‑to‑OS behavior.

> **Critical methodological caveat — the observer effect.** `tracemalloc`, especially at a deep frame count, *itself consumes a large amount of memory* because it stores a traceback for **every** live allocation. In our environment, the identical `simple.txt` consume ended at **RSS ≈ 235 MB with `tracemalloc.start(25)` active** but at **RSS ≈ 128 MB with `tracemalloc` off** — roughly **105 MB of the apparent "spike" was the profiler's own bookkeeping** for the ~120 k tracked allocations (and it climbs further once post‑run snapshots are retained). We therefore treat **psutil‑only runs as authoritative for magnitude**, and use `tracemalloc` runs **only for attribution** (which `file:line` allocates, and how many objects). Every table below states which mode produced it.

### 1.5 The experiment matrix (one factor at a time)

* **Document types:** plain text (`simple.txt`, 21 B → `paperless_text`); PDF with embedded text (`simple.pdf`, 22,926 B → Tesseract parser, OCR skipped); scanned image needing OCR (`simple.png` 7,913 B, `examples/no-text.png` 32,595 B → Tesseract/`ocrmypdf`); Office via Tika — *reasoned from code only* (no Tika server was reachable, Section 6.6).
* **Batch sizes:** single document; sequential 10 and 50 documents in **one** process (worker recycling disabled) to expose cross‑document growth; contrasted with **fresh‑process‑per‑task** (`recycle=1` semantics).
* **Differential toggles:** classifier model **present vs. absent**; `DEBUG` **False vs. True**; barcode flag **off vs. on**; **import path vs. REST `metadata` endpoint**.

All inputs were the repository's own samples under `src/documents/tests/{samples,examples}`, copied to a scratch directory before consumption (so nothing in the repo was mutated).

---

## 2. TL;DR / Thesis

1. **The single largest import‑time spike is the machine‑learning classifier load**, not anything the user would call "metadata." `load_classifier()` builds a **fresh** `DocumentClassifier` and unpickles the model on **every call**, with **no cache** (`src/documents/classifier.py:L30‑L57`, `L76‑L94`). Measured cost: **≈ 104–108 MB of RSS at the load stage**, essentially **independent of the document** — a 21‑byte text file and a 22,926‑byte PDF both pay ~104–108 MB. That is the literal definition of "disproportionate to document size."

2. **The reason it is disproportionate** is that the classifier's size tracks the **training‑corpus vocabulary**, not the imported file. The unpickled model in our test was ~43 MB on disk; its in‑memory footprint is a `CountVectorizer` **vocabulary dictionary of ~111,000 small objects** (`classifier.py:L87`) plus three scikit‑learn estimators backed by **NumPy weight matrices** (~14 MB each, `classifier.py:L90‑L92`).

3. **The worst‑case simultaneous live set** occurs **inside the `transaction.atomic()` block** (`src/documents/consumer.py:L298`), at the `document_consumption_finished.send(...)` call (`L306`): the full extracted `text` (≡ `Document.content`), the loaded `classifier`, the new `Document` instance, and the **handlers' own matching/indexing allocations** are alive together while the six post‑consume handlers run. The transient whole‑file buffers are *not* in this set — `_store`'s MD5 read (`L402`) completes at `L301` **before** the signal, and the `_write` file copies (`L319/L321/L333`) and archive checksum (`L340`) run **after** it (`L315‑L342`) — all within the same transaction, but not concurrently with the handlers.

4. **There is no application cache accumulating data.** A repository‑wide search found **zero** cache primitives (`lru_cache`, `functools.cache`, `cache.set/get`, `LocMemCache`, `django.core.cache`) in the consumption path — the only match is a test override. Across a 50‑document in‑process batch, RSS reached a **steady‑state high‑water mark after the first document and grew only ~9 MB over the next 49** — a working set, not a leak.

5. **"Not released promptly" is by‑design and lives entirely in `Q_CLUSTER recycle=1`** (`settings.py:L452`). Within a task, freed memory is held in CPython/glibc allocator free‑lists (normal); **across** tasks, the worker is **fully restarted after every single task**, which is the *only* mechanism that returns memory to the OS. Release therefore happens **at task end**, not mid‑task.

6. **The retention is overwhelmingly NORMAL CPython behavior, not a leak.** `gc.collect()` freed only a small, constant **~30 objects** per run (no growth run‑over‑run). Freed memory that "sticks" within a worker is **glibc free‑list retention of large transient buffers** plus the **live scikit‑learn import graph in `sys.modules`** (demonstrated in Section 8); large NumPy matrices are `mmap`'d and returned to the OS **promptly** on free. The one genuine *code smell* — `pikepdf.open()` without `close()`/context‑manager (`src/paperless_tesseract/parsers.py:L34`) — is on the **REST metadata path, not the import pipeline**, and even there it did **not** leak across repeated calls in this version.

The rest of this document substantiates each point with numbers and citations.

---

## 3. The "metadata" disambiguation (read this first)

The user asked about "metadata handling." In this codebase the word *metadata* is ambiguous, and disambiguating it against the code is essential to answering the question correctly.

There is a method literally named `extract_metadata()`:
* `src/paperless_tesseract/parsers.py:L26` — opens the PDF with `pikepdf.open()` (`L34`) and reads XMP metadata.
* `src/paperless_tika/parsers.py:L29` — calls `parser.from_file()` (`L32`) and reads metadata keys.

**`extract_metadata()` is *not* part of the import/consumption pipeline.** A repository‑wide search for its callers returns exactly two references:

| Caller | Location | Kind |
|---|---|---|
| REST `metadata` endpoint | `src/documents/views.py:L269` (`get_metadata` at `L260`, `@action metadata` at `L282‑L283`) | **production** |
| Tika unit test | `src/paperless_tika/tests/test_tika_parser.py:50` | test only |

So the **only** production caller of `extract_metadata()` is the REST endpoint a browser hits when you open a document's *details* page (`src/documents/views.py:L260‑L311`). The consumer (`src/documents/consumer.py`) never calls it — confirmed by the absence of any `extract_metadata` reference in `consumer.py`.

**What the user perceives as "metadata processing" during import is actually two different pieces of work:**

1. **Classification** — loading the ML model (`load_classifier()`, `consumer.py:L292`) and assigning a correspondent, document type, and tags via the post‑consume signal handlers (`set_correspondent` `handlers.py:L35`, `set_document_type` `L101`, `set_tags` `L168`).
2. **Date extraction** — `parse_date()` (`consumer.py:L275` → `src/documents/parsers.py:L212`), which lazily imports `dateparser` (`parsers.py:L221`).

Throughout this document we therefore answer the user's "metadata" question about **import‑time classification + date extraction**, and we treat the `pikepdf`/Tika `extract_metadata` behavior as a **separate, REST‑only** concern (covered in Section 6.6 and Q6).

---

## 4. The import pipeline at a glance

`Consumer.try_consume_file()` (`src/documents/consumer.py:L180`) is a ten‑stage orchestrator. The stages relevant to memory, in order:

| # | Stage | Code | Memory behavior |
|---|---|---|---|
| 1 | Pre‑checks incl. duplicate MD5 | `pre_check_duplicate` `L102`, `hashlib.md5(f.read())` `L104` | Reads the **entire file** into a transient bytes buffer |
| 2 | Parse | `document_parser.parse(...)` `L261` | Loads content into parser `self.text` (`parsers.py:L296`); PDF/image parsers import heavy OCR machinery |
| 2b | Get text | `text = document_parser.get_text()` `L271` (`parsers.py:L342`) | Hands the full text to the consumer |
| 3 | Date | `parse_date(self.filename, text)` `L275` | First use lazily loads `dateparser` language data |
| 4 | **Load classifier** | `classifier = load_classifier()` `L292` | **Unpickles the whole model — the dominant spike** |
| 5 | Store (atomic) | `with transaction.atomic():` `L298` → `self._store(...)` `L301` | Saves `Document`; `_store` MD5 `L402`; archive MD5 `L340‑L341`; `_write` whole‑file copy `L429‑L432` |
| 6 | Post‑consume signals | `document_consumption_finished.send(..., classifier=classifier)` `L306` | Fires **six** handlers (`apps.py:L22‑L27`), including Whoosh indexing |

The classifier is deliberately loaded **once per document** (the comment at `consumer.py:L288‑L290` explains it is to avoid reloading across the multiple post‑consume hooks). The **peak simultaneous live set** is at the signal send (`L306`): `text`, `classifier`, the `Document` row, and the handlers' own matching/indexing allocations coexist while the six handlers run. The whole‑file checksum/copy buffers are *not* co‑resident with the handlers — `_store`'s MD5 read precedes the signal (`L301`/`L402`) and the `_write`/archive copies follow it (`L315‑L342`), all inside the same `transaction.atomic()` block (`L298`).

---

## 5. Per‑question answers

### Q1 — What is causing the spikes? (root‑cause with module/method attribution)

**Answer: the classifier unpickle dominates, with a secondary fixed cost from parser/`dateparser` first‑use imports; transient file reads are negligible for normal file sizes.**

Authoritative **psutil‑only** per‑stage RSS deltas (MP_TRACE off), `simple.txt`, model present:

| Stage | ΔRSS (MB) | Notes |
|---|---:|---|
| `pre_check_duplicate` (MD5 full read) `consumer.py:L104` | +0.6 | first‑touch of `hashlib` + the duplicate DB query; the 21‑byte read is nil |
| `parse` (TextDocumentParser) `paperless_text/parsers.py:L40‑L42` | +0.0 | `f.read()` of 21 bytes |
| `get_text` `parsers.py:L342` | +0.0 | returns a reference |
| `parse_date` (dateparser) `parsers.py:L221‑L223` | +0.0 | already imported in this process |
| **`load_classifier` (unpickle)** `consumer.py:L292`, `classifier.py:L76‑L94` | **+107.6** | **the spike** |
| `_store` (DB + MD5 + copy) `consumer.py:L301/L402` | +0.1 | |
| `finished.send` (6 handlers + Whoosh) `consumer.py:L306` | +4.8 | |
| **Net consume** (baseline 57.2 → end 128.5) | **+71.3** | |

Reproduced three times: net `+71.3 / +71.8 / +72.2 MB`; the `load_classifier` stage `+107.6 / +107.6 / +107.6 MB` (variance < 0.5 MB).

**Why ≈ 108 MB peak but ≈ 59 MB retained?** The `load_classifier` stage momentarily pushes RSS up **+107.6 MB**, but the net retained after `try_consume_file()` returns is **+71.3 MB**. The ~36 MB difference is transient: the ~43 MB pickle buffer (`classifier.py:L78`) and the unpickling scratch are freed before the call returns. What *stays* is (a) the scikit‑learn/SciPy/NumPy import graph held in `sys.modules` for the life of the process, and (b) allocator arenas retained after the classifier object itself is collected. The *retained* difference between model‑present and model‑absent runs is ≈ **58.8 MB** (net 71.3 − 12.5; see Q4).

**Attribution via `tracemalloc`** (a separate **warm‑import** traced run that isolates a fresh `load_classifier()`, so the one‑time import graph does not mask the model) pinpoints the exact lines (by `lineno`, snapshot diff):

| Allocator (`file:line`) | Size | Object count | What it is |
|---|---:|---:|---|
| `documents/classifier.py:L87` | 11,490 KB | **111,269** | `data_vectorizer` = `CountVectorizer` **vocabulary dict** (many small `str`/`int`) |
| `documents/classifier.py:L90` | 14,087 KB | 169 | `tags_classifier` (NumPy weight matrix) |
| `documents/classifier.py:L91` | 14,081 KB | 143 | `correspondent_classifier` (NumPy weight matrix) |
| `documents/classifier.py:L92` | 14,084 KB | 228 | `document_type_classifier` (NumPy weight matrix) |
| `<frozen importlib._bootstrap_external>:647` | 18,800–23,600 KB | 122k–191k | module bytecode for the sklearn/scipy import graph (cold full‑consume run) |

The first four rows sum to ≈ 53.7 MB of `tracemalloc`‑tracked memory (current 52.51 MB / peak 52.55 MB in that warm run; RSS Δ +63.7 MB). This single table answers "which methods are responsible": **`DocumentClassifier.load()` at `classifier.py:L87‑L92`.** Note the shape difference — the vocabulary is **~111 k tiny objects** (~11 MB) while each estimator is **a handful of objects holding ~14 MB** of contiguous NumPy data. That distinction drives the normal‑vs‑problematic verdict in Q6 and Section 8. The three estimator lines each appear at ~14 MB **inside `tracemalloc`**, directly confirming the NumPy weight matrices are tracemalloc‑visible (Section 8a).

**The disproportion, demonstrated.** The classifier cost is flat across document size and type (psutil‑only ΔRSS at the `load_classifier` stage):

| Input | Size | `load_classifier` ΔRSS |
|---|---:|---:|
| `simple.txt` | 21 B | +107.6 MB |
| `simple.pdf` (embedded text) | 22,926 B | +104.2 MB |
| `simple.png` (OCR) | 7,913 B | +106.0 MB |
| `examples/no-text.png` (OCR) | 32,595 B | +106.3 MB |

A 1,550×‑larger input changes the classifier‑load cost by only ~1–3 MB. The spike is governed by the **trained model**, not the document.

### Q2 — Is "metadata handling" creating unnecessary copies or holding references longer than needed?

**Answer: the import path makes several *transient* whole‑file copies (cheap for normal files) and holds three large objects across the atomic block; none of it is held *longer than needed* in a way that leaks. The "held references" are released the moment `try_consume_file()` returns.**

We traced the lifetime of each large object, forcing `gc.collect()` at known points to separate "still referenced" from "freed."

**(a) The extracted text — held once, mirrored once.** `text` is produced at `consumer.py:L271`, passed to `_store(text=...)` at `L301`, and persisted into `Document.content`, a `TextField` (`src/documents/models.py:L117`). So at peak there are effectively two references to the text: the local `text` and `document.content`. This is **one logical copy held deliberately** for the duration of the store + indexing, not an accidental duplication. For these samples the text is tiny; for a large OCR'd PDF it scales linearly with extracted text and is the legitimate cost of storing + indexing content.

**(b) The classifier — the big reference, but correctly scoped.** `classifier` is bound at `consumer.py:L292` and passed by reference into the signal (`classifier=classifier`, `L306` → `L310`); the handlers **reuse the one instance** (`handlers.py:L35/L101/L168`) rather than reloading — a deliberate, *good* choice (the comment at `consumer.py:L288‑L290`). Its scope is the `try_consume_file` frame; when the function returns, the only references vanish.

**(c) Whole‑file `f.read()` buffers — transient by construction.** The pipeline reads the entire file into memory at four sites: duplicate MD5 (`consumer.py:L104`), document MD5 in `_store` (`L402`), archive MD5 (`L340‑L341`), and the byte‑for‑byte copy in `_write` (`L429‑L432`, `write_file.write(read_file.read())`). Each is a short‑lived bytes object proportional to file size. Our CPython model demo (Section 8d) shows a ~22 MB `f.read()` is fully reclaimed on free. **Rationale for "unnecessary?":** these are *redundant* in the sense that the file is read multiple times, so for a very large file you can momentarily hold a few multiples of its size — but they are **not retained**; they do not accumulate.

**Is anything held *longer than needed*?** We measured RSS at four lifetime checkpoints for the `simple.txt` (model‑present) run:

| Checkpoint | RSS (MB) | Interpretation |
|---|---:|---|
| End of consume (function returned) | 128.5 | `text`/`classifier` locals already dropped with the frame |
| After `gc.collect()` | 129.5 | only **~30 objects** freed (constant, non‑growing) → nothing trapped in growing reference cycles |
| After `malloc_trim(0)` | 127.7 | only ~1.8 MB returned to OS (vs the post‑gc figure) |

The fact that RSS does **not** fall after `gc.collect()` (it holds ~flat at ~129 MB) even though the classifier is no longer referenced is the crux of the user's "held longer than needed" intuition — and the answer is: **the objects are *freed* (no longer referenced), but the process keeps the memory** — partly in allocator free‑lists, and mostly as the **still‑imported scikit‑learn/SciPy graph in `sys.modules`**. That is **normal CPython/glibc behavior** (Section 8), not a reference‑lifetime bug. The genuine cross‑task release is the worker restart (Section 7).

**"metadata" mapping:** the classification work (the real import‑time "metadata") creates the classifier reference described in (b); it does **not** create extra copies of the document. The REST `extract_metadata` copies are a separate path (Q6 / Section 6.6).

### Q3 — Is there caching behavior that accumulates data unexpectedly?

**Answer: No. There is no application‑level cache anywhere in the consumption path, and the per‑document classifier reload behaves like a *transient allocation*, not an accumulating cache.**

**Static evidence (grep‑confirmed absence).** A repository‑wide search of `src/` for cache primitives — `lru_cache`, `functools.cache`, `cache.set`, `cache.get`, `LocMemCache`, `from django.core.cache`, `caches[` — found **no matches in the consumption path**. The single hit anywhere is a test override (`src/documents/tests/test_classifier.py:393`). Likewise, there was **no** pre‑existing profiling/memoization harness (`tracemalloc`/`memory_profiler`/`psutil`/`objgraph`) anywhere in `src/`. So there is structurally no place for data to "accumulate in a cache."

**The classifier is the opposite of a cache.** `load_classifier()` (`classifier.py:L30`) constructs a **fresh** `DocumentClassifier()` (`L38`) and unpickles it (`L40`) on **every** call — there is no module‑level global, no `@lru_cache`, no memoization (`L30‑L57`). A cache would *retain* the model to save work; this code *rebuilds* it each time and lets it die at function end. Our model‑demo (Section 8b) confirms each `pickle.loads` produces an **independent fresh copy** (+32.0 MB, then +32.0 MB) — so repeated loads do not share storage, but neither do they accumulate, because each is released when the consume returns.

**Dynamic evidence (no cross‑document accumulation).** We consumed the **same document 50 times in one long‑lived process** (worker recycling disabled), deleting the created `Document` and forcing `gc.collect()` each iteration, and sampled RSS after each:

| Batch (one process) | Base RSS | After doc #1 | After doc #50 | Growth over 49 docs |
|---|---:|---:|---:|---:|
| `simple.txt`, model present | 56.5 | 127.2 | 136.2 | **+9.0 MB** |
| `simple.txt`, model absent | 56.3 | 71.2 | 73.5 | **+2.3 MB** |
| `simple.pdf`, model present (n=10) | 56.4 | 141.6 | 143.7 (#10) | +2.1 MB |

The object count (`len(gc.get_objects())`) held essentially flat (~122 k) across the 50 iterations (121,726 → 121,898, +172 total). **Interpretation:** RSS jumps to a **steady‑state high‑water mark on the very first document** (paying the one‑time sklearn/scipy import + arena working set) and then is **flat** — the small residual creep (~0.18 MB/doc) is ordinary heap fragmentation, not unbounded growth. If a cache were accumulating, RSS would climb roughly linearly with document count; it does not. **There is no accumulating cache.**

### Q4 — What is different between the cases that spike and the cases that don't?

**Answer: the dominant differentiator by far is whether a trained classifier model file exists. `DEBUG` raises the baseline modestly; the barcode flag adds a transient; the REST `metadata` endpoint is a different path entirely.**

We varied one factor at a time (psutil‑only, fresh process each). The **Classifier** and **`DEBUG`** rows report net consume RSS (`end − baseline`); the **Barcode** and **Path** rows report the relevant operation's own before→after RSS (their full three‑lens detail is in §6.5 and §6.6, cross‑referenced below) — because the barcode scan happens in the *task* before `try_consume_file`, and the REST path is a different entry point entirely:

| Differential | Case | Baseline / before (MB) | End / after (MB) | Net (MB) | Effect (with reference) |
|---|---|---:|---:|---:|---|
| **Classifier model** | `simple.txt`, **present** | 57.2 | 128.5 | **+71.3** | **dominant**; model adds **+58.8** vs absent (§6.2) |
| | `simple.txt`, **absent** | 59.6 | 72.2 | **+12.5** | `load_classifier()`→`None` (`classifier.py:L31/L36`); 0 MB at load stage |
| | `simple.pdf`, **present** | 57.2 | 143.0 | +85.9 | model adds **+55.5** vs absent (§6.2) |
| | `simple.pdf`, **absent** | 59.4 | 89.9 | +30.4 | parser/OCR import cost only, no classifier |
| **`DEBUG`** | `simple.txt`, `DEBUG=False` | 57.2 | 128.5 | +71.3 | objects at end **121,849** (§6.3) |
| | `simple.txt`, `DEBUG=True` | **79.3** | 148.4 | +69.1 | baseline **+22 MB**; **+35.8 k** more objects (157,629 vs 121,849); `connection.queries` |
| **Barcode** (scan op; full lenses §6.5) | **off** (flag unset) | 50.2 | 50.2 | **+0.0** | `scan_file_for_separating_barcodes` never called |
| | **on** (`tasks.py:L195`→`L105`) | 50.2 | 71.2 | **+21.0** | transient bitmap; `tracemalloc` cur 0.65 / **peak 7.78**; +14,772 obj; 0 cycles |
| **Path** (import vs REST; §6.6) | **import** consume (`simple.pdf`, model present) | 57.2 | 143.0 | **+85.9** | classifier‑dominated; `extract_metadata` **never called** on import |
| | **REST** `extract_metadata` (`simple.pdf`, per call) | 52.8 | 67.1 | **+14.3** | qpdf/`pikepdf` (~12.7 MB `tracemalloc`‑invisible); **no classifier**; flat across 10 calls |

**Reading the differential:**

* **Model present vs. absent is the whole story for "disproportionate."** When `settings.MODEL_FILE` is absent, `load_classifier()` returns `None` immediately (`classifier.py:L31/L36`) — the `load_classifier` stage costs **0 MB** and the consume of a 21‑byte file finishes at **~72 MB** (mostly the one‑time parser/Django import baseline). When the model exists, the same consume hits **~128 MB**. *The spike literally appears and disappears with the model file.* This is also why a freshly installed instance "doesn't spike" until enough documents have been added for the periodic `train_classifier()` task (`tasks.py:L48`) to produce a model.
* **`DEBUG=True` is a secondary, real factor.** It raises the resident baseline **~22 MB** (79.3 vs 57.2) and causes more objects to be retained — **+35.8 k** at end (157,629 vs 121,849), because Django records every SQL query on `connection.queries` while `DEBUG=True`. For a single document the end‑to‑end effect is modest (**~20 MB**: end 148.4 vs 128.5), but in a long‑running process it grows with query volume — a known Django gotcha, and a reason production runs with `DEBUG=False` (`settings.py:L50`).
* **The barcode flag** only matters if `PAPERLESS_CONSUMER_ENABLE_BARCODES` is set, in which case the *task* `consume_file` rasterizes PDF pages (`tasks.py:L195` → `scan_file_for_separating_barcodes` `L96` → `convert_from_path` `L105`) **before** `try_consume_file`. The rasterized bitmaps are a **transient** of **+21.0 MB** for this 1‑page PDF (`tracemalloc` peak only 7.78 MB → overwhelmingly C‑level poppler/`pdf2image`; §6.5), proportional to page‑count × DPI and reclaimed immediately after the scan; far larger for big, high‑resolution PDFs.
* **Import path vs. REST `metadata` path are different memory profiles.** The import pipeline (`try_consume_file`) loads the classifier and is **classifier‑dominated** (+85.9 MB net for `simple.pdf`); it **never** calls `extract_metadata`. The REST `metadata` endpoint (`views.py:L260‑L269`) calls `extract_metadata` (`paperless_tesseract/parsers.py:L26`), which opens the PDF via `pikepdf`/qpdf — **+14.3 MB per call** (of which ~12.7 MB is C++ qpdf, invisible to `tracemalloc`), and **never** loads the classifier; the cost is flat across 10 repeated calls (§6.6). So a spike seen during *import* is the classifier; a smaller bump while viewing a document's *details page* is qpdf.

### Q5 — How does behavior change with document type and batch size?

**Document type.** The classifier cost is constant (~104–108 MB at the load stage, Q1); what changes with type is the **parser** cost:

| Type | Sample | Parser | `parse` ΔRSS | Net consume (model present) |
|---|---|---|---:|---:|
| Plain text | `simple.txt` (21 B) | `TextDocumentParser` (`paperless_text/parsers.py:L40`) | +0.0 MB | +71.3 MB |
| PDF, embedded text | `simple.pdf` (22.9 KB) | `RasterisedDocumentParser` (OCR skipped) | +25.2 MB | +85.9 MB |
| Image (OCR) | `simple.png` (7.9 KB) | `RasterisedDocumentParser` + Tesseract/`ocrmypdf` | +26.4 MB | +89.0 MB |
| Image (OCR) | `examples/no-text.png` (32.6 KB) | same | +29.0 MB | +91.5 MB |
| Office | (Tika) | `TikaDocumentParser` | *not run (Section 6.6, §10)* | reasoned from code |

The text parser is essentially free (a 21‑byte `f.read()`). The PDF/image parser carries a **~25–29 MB fixed cost** — dominated by **importing** the OCR toolchain (`pikepdf`, `pdfminer`, `PIL`, `ocrmypdf`) on first use, plus modest per‑page work; embedded‑text PDFs and OCR'd images cost about the same here because the *import* dominates the tiny actual OCR. For large multi‑page scans, the per‑page OCR work would add on top of this fixed cost.

**Batch size.** This is where the user's "not released in a timely manner" observation is explained directly. Two regimes:

1. **Many documents in one process (recycling disabled)** — the realistic shape *if* `recycle` were large. RSS climbs to a **plateau on document #1** and stays there (Q3 table): `simple.txt` model‑present sat at **127–136 MB across 50 documents** (+9.0 MB total), `simple.pdf` at **142–144 MB across 10**. The plateau is the worker's **steady‑state working set** (one‑time imports + the high‑water classifier arena), *not* linear growth. So a single worker that processed many documents would settle at a high‑but‑bounded RSS — which *looks* like "memory never comes back" to an observer watching the process.

2. **Fresh process per task (`recycle=1`, the actual configuration)** — every separate‑process consume started at the **same ~57 MB baseline** and the OS reclaimed everything when the process exited. This is why, in production, RSS sawtooths: it spikes to ~130–150 MB during a document and drops back to ~57 MB when the worker is recycled after that single task (Section 7).

**The combined picture:** a *spike per document* (driven by the classifier, **~104–108 MB at the load stage**) on top of a *baseline that is reset per task by `recycle=1`*. Larger/again‑OCR documents add ~25–29 MB of parser cost; `DEBUG=True` raises the floor ~22 MB.

### Q6 — Evidence, and: is this normal Python GC behavior or something problematic?

**Answer: it is overwhelmingly NORMAL CPython allocator behavior, not a leak. There is exactly one genuine code smell (`pikepdf.open` without `close()`), and it is on the REST metadata path, not the import pipeline — and even there it did not leak in this version.**

**Evidence 1 — no reference‑cycle leak.** Across every single‑document run, `gc.collect()` freed only a **small, constant number of objects** (~30; one PDF‑no‑model run freed 40) — incidental cyclic structures from Django/imports, not growth. If the pipeline were leaking through reference cycles, `gc.collect()` would report ever‑larger collections and `len(gc.get_objects())` would climb run‑over‑run. Neither happened: in the 50‑document batch, `gc.collect()` freed 30 objects on **every** iteration (1,500 total) and the object count stayed essentially flat (121,726 → 121,898, **+172 over 49 documents**).

**Evidence 2 — "freed but not returned to the OS" is the allocator, not a leak.** After a consume, RSS does not fall on `gc.collect()` (128.5 → 129.5 MB) and `malloc_trim(0)` returns only ~2 MB (→ 127.7 MB) — because most of the retained memory is **legitimately live**: the scikit‑learn/SciPy/NumPy import graph stays in `sys.modules` for the worker's lifetime. Section 8 isolates the *pure* allocator effect with controlled workloads: a 22 MB whole‑file `f.read()` is freed at the Python level but stays resident after `del` + `gc.collect()` and is returned to the OS only by `malloc_trim(0)` (Section 8d) — the glibc large‑buffer free‑list behavior. (A 2,000,000‑entry small‑object dict, by contrast, *is* returned promptly on collection — pymalloc releases its arenas, Section 8c — so the residue is allocator‑ and shape‑dependent, not universal.) Either way, everything is reclaimed wholesale when `recycle=1` restarts the worker.

**Evidence 3 — the lenses agree on attribution, and NumPy is visible.** A common myth is "`tracemalloc` can't see NumPy." In this environment that is **false**: `numpy==1.22.3` registers its allocations with `tracemalloc` via the NumPy domain (`np.lib.tracemalloc_domain == 389047`). A controlled 48 MB array showed **+48.0 MB in BOTH** RSS and `tracemalloc` (Section 8a), and the classifier's NumPy weight matrices appear in `tracemalloc` attribution at ~14 MB each (`classifier.py:L90‑L92`, Q1). The genuinely `tracemalloc`‑**invisible** memory is **qpdf/`pikepdf` (C++)** — see Evidence 4 — for which RSS is essential.

**Evidence 4 — the one real smell, and why it still doesn't leak here.** `src/paperless_tesseract/parsers.py:L26‑L55` opens a PDF with `pdf = pikepdf.open(document_path)` (`L34`), reads metadata, and `return result` (`L55`) with **no `pdf.close()` and no `with` context‑manager**. qpdf holds C++ buffers that are released only when the `pdf` object is collected. We exercised this REST path directly, calling `extract_metadata()` ten times in a loop:

| | RSS (MB) | `tracemalloc` current (MB) |
|---|---:|---:|
| before | 52.8 | 0.0 |
| after call #1 | 67.1 (**+14.3**) | 1.6 (**+1.6 only**) |
| after call #5 | 67.1 | 1.6 |
| after call #10 | 67.1 | 1.6 |

Two findings: (1) **~12.7 MB of the +14.3 MB is invisible to `tracemalloc`** — that is the qpdf C++ allocation, confirming RSS is mandatory for this path; (2) **RSS is flat from call #1 to #10** (and `gc.collect()` freed 0 cycles), so the `pdf` local is in fact released by reference‑counting at each function return *despite* the missing `close()`. So in `pikepdf 5.1.1` this is a **latent code smell** (release depends on GC/refcount timing rather than being explicit) but **not an active leak**, and it is **not on the import pipeline** — only the REST details endpoint reaches it (`views.py:L269`). The cleaner form would be `with pikepdf.open(...) as pdf:`.

**Verdict.** The import‑time memory profile is *expected for a Django worker that unpickles a scikit‑learn model per task*: a large, document‑independent classifier load, a fixed parser/import baseline, transient file reads, and allocator free‑list retention that the per‑task worker restart cleans up. It is **normal**, not pathological. The only thing one might *change* (out of scope here, since this is diagnosis only) would be to cache/reuse the classifier across tasks or raise `recycle` — but both trade memory‑release timeliness for CPU, and the current `recycle=1` is a deliberate "release memory aggressively" choice.

---

## 6. Measured tables (per experiment)

Every experiment was run as a **fresh process** (the `recycle=1` semantics). RSS, object‑count, and `gc.collect()` cycle figures come from the **psutil‑only (authoritative)** run; `tracemalloc` current/peak and the top allocator come from a **paired traced run of the same experiment** (RSS in a traced run is inflated by the observer effect, Section 1.4, so the lenses are captured in separate runs and reported side‑by‑side). Each table below therefore carries all three lenses: **RSS** (psutil), **`tracemalloc`** (traced), and **`gc`** (object‑count delta + collected‑cycle count).

### 6.1 Document‑type matrix (single document, classifier present)

| Type / sample | Size | RSS base → end (Δ) | gc objΔ | gc cycles | `tracemalloc` cur / peak | top allocator (traced) | `parse` Δ | `load_classifier` Δ | after gc / trim |
|---|---:|---|---:|---:|---|---|---:|---:|---|
| text `simple.txt` | 21 B | 57.2 → 128.5 (**+71.3**) | +46,425 | 30 | 34.96 / 87.91 | importlib :647 (18.8 MB) | +0.0 | +107.6 | 129.5 / 127.7 |
| pdf `simple.pdf` | 22.9 KB | 57.2 → 143.0 (**+85.9**) | +61,858 | 30 | 45.25 / 98.13 | importlib :647 (22.9 MB) | +25.2 | +104.2 | 144.2 / 142.3 |
| pdf `test_with_bom.pdf` | 16.1 KB | 58.7 → 149.6 (**+90.9**) | +67,809 | 30 | 50.70 / 103.74 | importlib :647 (23.6 MB) | +26.7 | +104.7 | 150.5 / 148.7 |
| png OCR `simple.png` | 7.9 KB | 57.2 → 146.2 (**+89.0**) | +61,922 | 30 | 45.41 / 98.73 | importlib :647 (23.0 MB) | +26.4 | +106.0 | 147.2 / 145.4 |
| jpg OCR `simple.jpg` | 17.7 KB | 57.3 → 146.5 (**+89.1**) | +61,920 | 30 | 45.43 / 98.30 | importlib :647 (23.0 MB) | +26.9 | +106.1 | 147.4 / 145.7 |
| png OCR `examples/no-text.png` | 32.6 KB | 57.3 → 148.8 (**+91.5**) | +61,923 | 30 | 45.28 / 98.14 | importlib :647 (23.0 MB) | +29.0 | +106.3 | 149.7 / 144.8 |

The top `tracemalloc` allocator in every **cold full‑consume** run is the module‑bytecode line `<frozen importlib._bootstrap_external>:647` — the one‑time scikit‑learn/SciPy import graph triggered by the classifier load. To attribute memory to the *model itself* (rather than the import noise), see the warm‑import run in Q1 (`classifier.py:L87‑L92`). "after gc / trim" is RSS after `del`+`gc.collect()` then after `malloc_trim(0)`; the small post‑run rise then ~2 MB trim‑back confirms the retained bulk is live module state, not freed‑but‑stuck buffers.

### 6.2 Classifier present vs. absent (single document)

| Sample | Model | RSS base → end (Δ) | gc objΔ | gc cycles | `tracemalloc` cur / peak | `load_classifier` Δ |
|---|---|---|---:|---:|---|---:|
| `simple.txt` | **present** | 57.2 → 128.5 (**+71.3**) | +46,425 | 30 | 34.96 / 87.91 | +107.6 |
| `simple.txt` | absent | 59.6 → 72.2 (**+12.5**) | +9,246 | 30 | 6.22 / 7.10 | +0.0 |
| `simple.pdf` | **present** | 57.2 → 143.0 (**+85.9**) | +61,858 | 30 | 45.25 / 98.13 | +104.2 |
| `simple.pdf` | absent | 59.4 → 89.9 (**+30.4**) | +25,039 | 40 | 19.09 / 19.98 | +0.0 |

**The classifier alone** (present − absent): text **+58.8 MB RSS / +37,179 objects / +28.7 MB `tracemalloc`**; pdf **+55.5 MB RSS / +36,819 objects / +26.2 MB `tracemalloc`**. Because the classifier's NumPy matrices are tracemalloc‑visible, it shows in *both* lenses; the one‑time import‑graph residue shows mostly in RSS. Reproducibility (text present, 3 reps): net `+71.3 / +71.8 / +72.2`; variance < 0.5 MB.

### 6.3 `DEBUG` False vs. True (`simple.txt`, model present)

| `DEBUG` | RSS base → end (Δ) | gc objΔ | objects at end | gc cycles | `tracemalloc` cur / peak |
|---|---|---:|---:|---:|---|
| False | 57.2 → 128.5 (**+71.3**) | +46,425 | 121,849 | 30 | 34.96 / 87.91 |
| True | 79.3 → 148.4 (**+69.1**) | +45,710 | 157,629 | 30 | 32.18 / 85.32 |

`DEBUG=True` raises the **startup baseline ~+22 MB** (79.3 vs 57.2) and ends holding ~35.8 k more objects (157,629 vs 121,849) — Django records every SQL statement on `connection.queries`. The per‑document RSS *delta* (+69.1 vs +71.3) and the `tracemalloc` profile (32.2/85.3 vs 35.0/87.9) are essentially unchanged; the extra cost is a higher floor that, in a long‑running process, grows with query volume.

### 6.4 Batch sweep (one process, document deleted each iteration)

RSS / object / cycle columns are from psutil batches; the `tracemalloc` current column is from a **paired traced batch** (its RSS is omitted — observer effect).

| Batch (recycle disabled) | Base | doc #1 RSS | last RSS | RSS growth | objects #1 → last | cycles/iter | `tracemalloc` cur #1 → last |
|---|---:|---:|---:|---:|---|---:|---|
| `simple.txt`, present, n=50 | 56.5 | 127.2 | 136.2 | **+9.0 over 49** | 121,726 → 121,898 (+172) | 30 (1,500 total) | 35.71 → 35.80 (flat) |
| `simple.txt`, absent, n=50 | 56.3 | 71.2 | 73.5 | **+2.3 over 49** | 84,472 → 84,651 (+179) | 30 (1,500 total) | 6.97 → 7.02 (flat) |
| `simple.pdf`, present, n=10 | 56.4 | 141.6 | 143.7 | **+2.1 over 9** | 136,982 → 137,302 (+320) | 30 (last iter 343) | 46.08 → 46.13 (flat) |

All three lenses agree on **plateau, not accumulation**: RSS reaches its high‑water mark on document #1 and barely moves; the object count is essentially flat (+172 over 50 text documents); `tracemalloc` current is flat (≈ 35.8 MB text, ≈ 46.1 MB pdf). The per‑iteration `gc.collect()` returns a constant ~30 (never growing), so there is no cycle accumulation. Fresh‑process‑per‑task (`recycle=1`) starts every process at ~57 MB and frees everything on exit.

### 6.5 Barcode rasterization transient (`simple.pdf`, `tasks.py` path)

| Scan | RSS before → after (Δ) | gc objΔ | gc cycles | `tracemalloc` cur / peak |
|---|---|---:|---:|---|
| **off** (flag unset) | 50.2 → 50.2 (**+0.0**) | 0 | 0 | 0.0 / 0.0 (path not entered) |
| **on** (`tasks.py:L195` → `L105`) | 50.2 → 71.2 (**+21.0**) | +14,772 | 0 | 0.65 / **7.78** |

With the flag **off**, `scan_file_for_separating_barcodes` is never called — zero extra memory. With it **on**, `convert_from_path` rasterizes the page(s): RSS jumps **+21 MB** but `tracemalloc` sees only 0.65 MB current / 7.78 MB peak — i.e., the bitmap is overwhelmingly **C‑level (poppler/`pdf2image`)** and tracemalloc‑near‑invisible, and it is a **transient** (0 cycles; reclaimed after the scan). This 1‑page PDF found no separator; high‑resolution multi‑page PDFs would scale this transient with page‑count × DPI.

### 6.6 REST `metadata` endpoint (NOT the import path) — `extract_metadata` loop (×10)

| Sample | RSS before | RSS after #1 (Δ) | RSS #10 | `tracemalloc` cur | gc objΔ | gc cycles |
|---|---:|---|---:|---:|---:|---:|
| `simple.pdf` | 52.8 | 67.1 (**+14.3**) | 67.1 | 1.61 (flat) | +4,060 | 0 |
| `test_with_bom.pdf` | 52.8 | 67.0 (**+14.2**) | 67.0 | 1.62 (flat) | +4,080 | 0 |

**qpdf is `tracemalloc`‑invisible (+14 MB RSS vs +1.6 MB traced) and does not accumulate across calls** (RSS, `tracemalloc` current, and object count are all flat from call #1 to #10; 0 cycles), despite the missing `pdf.close()` (Q6, Evidence 4). The Office/**Tika** cell was **not exercised at runtime** — no Tika/Gotenberg server was reachable (`localhost:9998` → `ConnectionRefusedError`), so it is reasoned from code: `extract_metadata` calls `parser.from_file()` (`L32`) which loads the **entire** parsed content+metadata dict just to read metadata keys (`L45`), and `parse()` assigns `self.text = parsed["content"].strip()` (`L62`) — i.e., the whole document content is materialized in memory; this scales with document size and is the Tika analog of the OCR parser's content cost (see also Section 10).

---

## 7. The release mechanism — why memory "isn't released in a timely manner"

This is the most important section for the user's "not always released" observation, and the answer is entirely about **how Django‑Q recycles workers**, not about anything inside the consumption code.

**What the configuration says.** Paperless sets, in `src/paperless/settings.py:L449‑L452`:

```python
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": ...,
    "timeout": ...,
    "workers": ...,
    "redis": ...,
}
```

There is **no `max_rss` key**. The two relevant Django‑Q settings behave as follows (Django‑Q 1.3.x documentation):

* `recycle` — "the number of tasks a worker will process before recycling." It is "useful to release memory resources on a regular basis," and defaults to **500**.[^q-recycle]
* `max_rss` — "the maximum resident set size in kilobytes before a worker will recycle and release resources"; it is unset here (default `None`), so RSS‑threshold recycling is disabled.[^q-maxrss]

Paperless overrides `recycle` from the default of 500 down to **1**. The cluster's internal sentinel "checks the health of all the processes and recycles or reincarnates according to your settings."[^q-sentinel] With `recycle: 1`, **the worker process is fully torn down and a fresh one spawned after every single task** — i.e., after every single document.

**What that means for memory.** Process teardown is the one event that unconditionally returns *all* of a process's memory to the OS — pymalloc arenas, glibc free‑lists, loaded C‑extension data, everything. So:

* **Within a task**, freed Python objects are *not* promptly returned to the OS; they sit in allocator free‑lists (Section 8). RSS for a single document therefore *stays* near its high‑water mark from the `load_classifier` peak (~130–150 MB) until the task ends — this is exactly the "memory isn't released in a timely manner" the user sees if they watch the worker mid‑task or just after.
* **At task end**, `recycle: 1` restarts the worker, and RSS drops back to the ~60 MB fresh baseline we measured for every new process. The production memory graph is therefore a **sawtooth**: ~60 MB → spike to ~130–150 MB during a document → back to ~60 MB on recycle.

**Why this design exists.** Recycling after every task is the project's deliberate, blunt defense against exactly the kind of allocator residue this report documents: rather than trying to make the per‑document classifier load cheap or cached, Paperless simply throws the worker away and starts clean. The trade‑off is CPU/startup cost (re‑importing Django + sklearn on every task) in exchange for a guaranteed memory reset. (A real‑world Django‑Q memory‑growth report, GitHub issue #621, shows what happens *without* effective recycling — RSS "keeps going higher gradually and ends up eating all the memory"[^q-issue621] — which is the failure mode `recycle: 1` is designed to prevent.)

**Corollary.** If someone raised `recycle` to a large number (or the worker processed many tasks before recycling), our batch measurements (Section 6.4) predict RSS would climb to a **bounded plateau** (~130–150 MB for the classifier working set) and stay there — *not* an unbounded leak, but a sustained high‑water mark that "never comes back" until the next recycle. That is the precise behavioral knob behind "spikes that don't come down."

---

## 8. CPython memory model — the interpretive lens (controlled demos)

To judge "normal vs. problematic," these controlled demonstrations were run in the **same** Python 3.9.23 / `numpy==1.22.3` container (reproduced across two runs; numbers stable). They establish four facts the pipeline analysis relies on. As in Section 6, the **RSS** column is from a **psutil‑only** run (authoritative for return‑to‑OS magnitude) and the **`tracemalloc`** column is the Python‑allocation delta from a **paired traced run** (whose own RSS is inflated by the observer effect — e.g., tracing the 2 M‑entry dict build pushes RSS to +631 MB of traceback bookkeeping while the genuine traced heap is +241 MB; Section 1.4). "→ 0" marks that the traced Python objects were released back to the tracker; `malloc_trim` is an allocator operation that `tracemalloc` does not observe.

| Demo | Setup | RSS | `tracemalloc` | Takeaway |
|---|---|---|---|---|
| **(a)** NumPy visibility | allocate 48 MB `np.ones` | +48.0 MB | **+48.0 MB** | NumPy **is** tracemalloc‑visible (`np.lib.tracemalloc_domain == 389047`) |
| **(c‑large)** big array free | `del` + `gc.collect()` | **−48.0 MB** (returned fully) | → 0 (freed) | large (>128 KB) allocations are `mmap`'d and `munmap`'d **promptly** on free |
| **(b)** pickle copies | `pickle.loads` of a 32 MB array, twice | +32.0, then +32.0 MB | +32.0, then +32.0 MB | each `pickle.loads` is an **independent fresh copy** (no sharing/cache); visible in **both** lenses |
| **(c‑small)** many small objects | build a 2,000,000‑entry dict of small `str`/`int` | +265.9 MB | +241.1 MB | (see below) |
| ↳ free it | `del` + `gc.collect()` | −265.5 MB (≈ **0.4 MB stays resident**) | → 0 (freed) | pymalloc **releases its arenas on collection** — the small‑object memory comes back here |
| ↳ `malloc_trim(0)` | | ~0 MB (nothing left to return) | n/a (allocator op) | little glibc residue remained to trim in this case |
| **(d)** whole‑file read | `f.read()` ~22 MB | +22.0 MB | +22.0 MB | transient; the `bytes` buffer is a Python allocation, so it is tracemalloc‑visible |
| ↳ free it | `del` + `gc.collect()` | **0 MB returned** (stays resident) | → 0 (freed) | single large buffer parked in the glibc main‑arena free‑list |
| ↳ `malloc_trim(0)` | | **−22.0 MB** (returned fully) | n/a (allocator op) | the genuine "freed‑but‑not‑returned‑until‑trim" case |

**How this maps onto the pipeline:**

* **(a)+(c‑large)** explain the **NumPy weight matrices** (`classifier.py:L90‑L92`, ~14 MB each): they are visible to `tracemalloc`, and because each is a single large contiguous buffer, it is returned to the OS *promptly* when the classifier is freed. **These are not the sticky residue.** This also corrects the common myth — for `numpy 1.22.3`, "tracemalloc can't see NumPy" is **false**; the genuinely invisible memory is qpdf/`pikepdf` C++ (Section 6.6).
* **(b)** is the **classifier‑per‑document** behavior in miniature: `load_classifier()` produces a fresh independent copy every call (`classifier.py:L38/L40`), exactly like repeated `pickle.loads`. No cache, no sharing — and no accumulation, because each copy dies at task end.
* **(c‑small)** is the **`CountVectorizer` vocabulary** (`classifier.py:L87`, ~111 k tiny objects) in miniature: in this Python 3.9 environment, after the model is freed, pymalloc **releases its arenas on collection** and the small‑object memory comes back (≈ 0.4 MB residual). So the vocabulary is *not* the sticky residue either — which is why the post‑consume RSS that "stays" is dominated instead by the **live scikit‑learn/SciPy import graph held in `sys.modules`**, reclaimed only by the `recycle: 1` worker restart.
* **(d)** is the **whole‑file `f.read()`** buffers (`consumer.py:L104/L340/L402/L432`): a single large buffer that is freed at the Python level but **returned to the OS only on `malloc_trim(0)`** (or reuse / worker recycle) — the one clear "freed‑but‑not‑returned‑in‑a‑timely‑manner" shape. It is transient per call and does **not** accumulate.

---

## 9. Component‑by‑component memory attribution

Every memory holder and transient site in the import path (plus the REST‑only holders), mapped to its exact `file:line` and its measured contribution. "Measured contribution" is the RSS or `tracemalloc` figure from Section 6; "lifetime" is when the memory is released.

| Component | `file:line` | Kind | Measured contribution | Lifetime / release |
|---|---|---|---|---|
| **Classifier unpickle (whole)** | `classifier.py:L76‑L94` (`load_classifier` `L30/L38/L40`) | Python objects + NumPy | **+104–108 MB RSS** at the `load_classifier` stage (dominant spike); ~88 MB `tracemalloc` peak in a full traced consume | Held to end of `try_consume_file`; freed when `classifier` goes out of scope; OS‑returned on `recycle: 1` |
| ↳ `CountVectorizer` vocabulary dict | `classifier.py:L87` | many small `str`/`int` | ~11.49 MB across **111,269 objects** (top `tracemalloc` lineno) | Returned to the allocator on collection (pymalloc releases arenas, Section 8 c‑small); reset on recycle |
| ↳ 3 MLP/estimator weight matrices | `classifier.py:L90`, `L91`, `L92` | NumPy arrays | ~14.09 / 14.08 / 14.08 MB; ~143–228 objects each | `mmap`'d → returned to OS promptly on free (Section 8 c‑large) |
| ↳ first‑time sklearn/scipy import | `<frozen importlib._bootstrap_external>:647` (triggered by `classifier.py` imports) | bytecode + module objects | ~18.8–23.6 MB across 122k–191k objects | One‑time per worker; reset on recycle |
| **Extracted `text` (≡ `Document.content`)** | `consumer.py:L271` (`get_text`); passed to `_store` (`L301`) and used by the signal (`L306`); mirrored at `models.py:L117` | Python `str` | Scales with document text length (small for test corpus; negligible vs. classifier) | Single copy; alive across the `transaction.atomic()` block (from `L298`); freed at task end |
| **Atomic‑block co‑retention (peak live set)** | `consumer.py:L298` (signal at `L306`) | composite | **The worst‑case instant (at the signal send):** `text` + `classifier` + `Document` row + the handlers' matching/indexing allocations live together while the 6 handlers run. The `_store` MD5 read (`L402`) is **before** the signal; the `_write`/archive copies (`L315‑L342`) are **after** it — same transaction, not concurrent with the handlers | Peak at the signal; collapses after the `with` block |
| **Parser `self.text`** | `parsers.py:L296` (init), `L342` (`get_text`); text parser read at `paperless_text/parsers.py:L42` | Python `str` | Same order as `text`; one buffer per parser instance | Freed when parser is GC'd after `try_consume_file` |
| **Whole‑file `f.read()` — duplicate pre‑check** | `consumer.py:L104` | transient `bytes` | Proportional to file size (≤ ~32 KB for samples); reclaimed immediately | Transient (Section 8 d) |
| **Whole‑file `f.read()` — archive checksum** | `consumer.py:L340‑L341` | transient `bytes` | Proportional to archive file size | Transient |
| **Whole‑file `f.read()` — `_store` checksum** | `consumer.py:L402` | transient `bytes` | Proportional to file size | Transient |
| **Whole‑file copy — `_write`** | `consumer.py:L429‑L432` (`write_file.write(read_file.read())`) | transient `bytes` | Proportional to file size | Transient |
| **Whoosh `AsyncWriter` index buffer** | `index.py:L66` (writer), `L87‑L93` (`update_document`, `content=doc.content`) | Python/Whoosh | Buffers the full content field during the `add_to_index` handler | Flushed/committed by the writer; small for test corpus |
| **`dateparser` first‑use language data** | `parsers.py:L221‑L223` (`import dateparser`; `dateparser.parse(...)`) | Python module data | Lazy heavy load on first call → raises resident **baseline** (paid once per worker, i.e., per task under `recycle: 1`) | Part of baseline; reset on recycle |
| **Barcode rasterization (feature‑flagged)** | `tasks.py:L105` (`convert_from_path`), gated by `L195` (`CONSUMER_ENABLE_BARCODES`) | `pdf2image` bitmaps | **+21.0 MB net RSS; 0.65 MB `tracemalloc` current / 7.78 MB peak** (mostly C‑level poppler), simple.pdf | Transient; reclaimed after scan |
| **Django‑Q pickled task payload** | overrides passed as scalar kwargs to `consume_file` (`tasks.py:L184`) | pickled args | Small (no `data_models.py` overrides object in 1.3.9) — **not a suspect** | Held by broker/ORM, not the worker working set |
| **REST‑only: `pikepdf.open` without `close()`** | `paperless_tesseract/parsers.py:L34` (open), `L55` (return, no close) | qpdf **C++** | **+14 MB RSS but only +1.6 MB `tracemalloc`** (C++ invisible); **flat across 10 calls — no leak** | Released by refcount on return; **release‑timing smell, REST path only — not import** |
| **REST‑only: Tika full content+metadata load** | `paperless_tika/parsers.py:L32` (`from_file` for metadata), `L55/L62` (`parse`) | Python dict/`str` | Loads entire parsed dict to read metadata; scales with document size | Per‑request; REST path (and Office `parse`), not the import classification work |

**Reading the table:** the single dominant import‑time contributor is the classifier unpickle (**+104–108 MB at the load stage**), which is **independent of the imported document's size** — the root of "disproportionate." Everything else in the import path is either small‑and‑persistent (`text`/`content`), transient (file reads, barcode bitmaps), or baseline (`dateparser`, first‑time imports). The two REST‑only holders are *not* on the import path and must not be conflated with it.

---

## 10. Version‑specificity of these conclusions

Every number and conclusion in this report is tied to the dependency pins of this commit. The following version facts materially shape the findings:

* **`django-q==1.3.9` (Koed00 Django‑Q, *not* Celery, *not* django‑q2).** This is verified by `requirements.txt` and by the presence of `Q_CLUSTER` in `settings.py:L449`. The release behavior (Section 7) is governed by this version's `recycle`/`max_rss` semantics. Critically, **this version has no `data_models.py` overrides object** — consumption overrides are passed to `consume_file` (`tasks.py:L184`) as **scalar keyword arguments**, so the pickled task payload (via `django-picklefield==3.0.1`) is small and is *not* a memory suspect. A newer Paperless version that introduced a structured overrides object would change that calculus.
* **`scikit-learn==1.0.2` + `numpy==1.22.3`.** The classifier's `FORMAT_VERSION = 7` (`classifier.py:L63`) and the six‑object pickle layout (`classifier.py:L86‑L92`) are specific to this serialization. The **`tracemalloc`‑visibility of NumPy** (Section 8 a) is a property of `numpy 1.22.3`, which registers allocations in `np.lib.tracemalloc_domain` (observed value `389047`). On a NumPy build without that hook, the weight matrices would be RSS‑only — so the "report what each lens shows" caveat is version‑bound. The absolute classifier size (~43.4 MB on disk; ~104–108 MB load spike) scales with **training‑corpus vocabulary** (here vocab≈6000 over 400 docs); a larger production model produces a proportionally larger spike.
* **`pikepdf==5.1.1` (qpdf C++).** The `extract_metadata` "no `close()`/no context manager" pattern (`paperless_tesseract/parsers.py:L34/L55`) holds qpdf buffers until refcount‑drop on return. In this version the object *is* released promptly by refcount (measured stable across 10 calls, Section 6.6), so the smell does not manifest as a leak — but that depends on `pikepdf`/qpdf not introducing additional retained references, which is version‑dependent.
* **`tika==1.24`.** The Office‑document matrix cell was **not exercised at runtime** (no Tika/Gotenberg server was reachable — connection refused on `:9998`); its memory behavior is reasoned from code (`paperless_tika/parsers.py:L32/L55/L62`), not measured. This is the one matrix cell without direct RSS evidence, and it is flagged as such.
* **Python 3.9.23.** The pymalloc‑arena and glibc `malloc_trim` behavior (Section 8) is characteristic of CPython 3.9 on glibc Linux. The observer effect of `tracemalloc.start(25)` (Section 1) — ~105 MB of traceback storage for the ~120 k tracked allocations — is also interpreter‑ and frame‑depth‑specific.

---

## 11. Closing — where the memory actually goes

Putting the three lenses and all experiments together, the per‑document memory story of the Paperless‑ngx import pipeline is:

1. **A fresh worker starts at ~60 MB** (Django + first‑time imports + `dateparser` language data baseline).
2. **The parser reads/extracts the document** — a transient file read (`f.read()`, Section 8 d) plus the persistent `text`/`Document.content` string. For PDFs/images this adds ~25–29 MB of parser/OCR working set (`consumer.py:L261`); for plain text it is negligible.
3. **`load_classifier()` fires the dominant spike** — **+104–108 MB RSS at the load stage** (`classifier.py:L76‑L94`), *independent of the document size*. Of that, the NumPy weight matrices (~42 MB total, ~14 MB each, `L90‑L92`) are large `mmap`'d buffers returned promptly on free, and the `CountVectorizer` vocabulary (~11 MB across 111 k tiny objects, `L87`) is returned to the allocator on collection (pymalloc arenas). What actually *stays* resident between the consume and the worker recycle is the one‑time scikit‑learn/SciPy import graph in `sys.modules`.
4. **The `transaction.atomic()` block (`consumer.py:L298`) holds the peak live set at the signal send (`L306`)** — `text` + `classifier` + `Document` row + the handlers' own matching/indexing allocations, alive together while the six signal handlers run. The transient whole‑file buffers are *not* part of that instant: `_store`'s MD5 read (`L402`) completes before the signal, and the `_write`/archive copies (`L315‑L342`) run after it — within the same transaction, but not concurrently with the handlers.
5. **Nothing accumulates across documents within a worker** — the batch sweep (Section 6.4) shows RSS reaching a **bounded plateau** after the first document (~130–150 MB) and *not* climbing thereafter; `gc.collect()` frees only ~30 objects per run (constant, non‑growing), and `tracemalloc` current is flat across the batch. There is **no application cache** in the import path (grep‑confirmed, Q3).
6. **All of it is returned to the OS at task end via `Q_CLUSTER recycle: 1`** (`settings.py:L452`) — a full worker restart after every document. This is the *only* cross‑task release mechanism, and it is why production memory is a **sawtooth** (~57 MB → ~130–150 MB → ~57 MB), not a monotonic climb.

**Verdict:** the spikes are **disproportionate to document size** because they are dominated by the **training‑corpus‑sized classifier**, reloaded fresh per document with **no cache**; the memory that "isn't released in a timely manner" is **normal CPython behavior** — chiefly the scikit‑learn import graph held live in `sys.modules`, plus glibc free‑list retention of large transient buffers — distinguished from a genuine leak by forced `gc.collect()` (~30 objects, no growth) + `malloc_trim(0)` (returns the glibc residue). The release is deferred only until the **per‑task worker recycle**, by design. The one genuine code smell — `pikepdf.open` without `close()` (`paperless_tesseract/parsers.py:L34`) — lives on the **REST `metadata` path**, not the import pipeline, and does not leak in these versions. **No source change is recommended by this read‑only engagement;** the behavior is well‑characterized, bounded, and normal for the stack.

---

## Appendix A — Measurement methodology (throwaway harness)

All runtime numbers were produced by a **temporary** triple‑lens harness written outside the repository tree (under `/tmp/`), driving the **real** `Consumer().try_consume_file(...)` against temp copies of the repository's own sample inputs (`src/documents/tests/samples/`, `examples/`). The harness was deleted on completion; the only committed artifact is this document. It sampled, around each pipeline stage:

* **Process RSS** via `psutil.Process().memory_info().rss` — the authoritative total, and the only lens that captures qpdf C++ memory.
* **Python allocations** via `tracemalloc` (`start(25)`, paired `take_snapshot()`, `compare_to('lineno'|'traceback')`, `get_traced_memory()`) — for `file:line` attribution.
* **Object accounting** via `gc` (`get_count()`, `len(get_objects())`, `collect()`), `resource.getrusage(...).ru_maxrss` as a peak cross‑check, and `ctypes` `malloc_trim(0)` to demonstrate return‑to‑OS.

To avoid the `tracemalloc` observer effect (Section 1), the authoritative RSS/object/cycle figures (Section 6) were captured with `tracemalloc` **off** (`psutil`‑only), and the `tracemalloc` current/peak/top‑allocator figures from **paired traced runs** of the same experiments; the Q1 line attribution used a **warm‑import** traced run that isolates a fresh `load_classifier()`. A real classifier model was trained (400 docs, vocab≈6000, ~43.4 MB pickle, ~14 MB per MLP weight matrix) so the classifier‑present differential exercised genuine vocabulary + NumPy matrices. Key experiments were repeated; variance was < 0.5 MB.

---

## References

Claims grounded in this repository's code are cited inline as `file:Lnn`, and claims grounded in measurement are cited to the tables in Sections 5–8. The external‑behavior claims (Django‑Q worker recycling, and the `tracemalloc` measurement boundary) are grounded in the following authoritative sources, referenced as footnotes above:

[^q-recycle]: Django‑Q 1.3.x — *Configuration*, `recycle` setting: "The number of tasks a worker will process before recycling … Defaults to 500." (documented as useful for releasing memory resources on a regular basis). https://django-q.readthedocs.io/en/latest/configure.html

[^q-maxrss]: Django‑Q 1.3.x — *Configuration*, `max_rss` setting: "The maximum resident set size in kilobytes before a worker will recycle and release resources … Defaults to None." https://django-q.readthedocs.io/en/latest/configure.html

[^q-sentinel]: Django‑Q 1.3.x — *Cluster*: the cluster has "an internal sentinel that checks the health of all the processes and recycles or reincarnates according to your settings" (or in case of unexpected crashes). https://django-q.readthedocs.io/en/latest/cluster.html

[^q-issue621]: r0b1n1sl4m, "Memory usage keeps going higher gradually and ends up crashing the server," Koed00/django‑q issue #621 (18 Oct 2021) — reporter observes RSS "keeps going higher gradually and ends up eating all the memory." Cited only as an illustrative real‑world report of the failure mode `recycle: 1` prevents; not a claim about this codebase. https://github.com/Koed00/django-q/issues/621

[^py-tracemalloc]: Python 3 Standard Library — *`tracemalloc` — Trace memory allocations*: "a debug tool to trace memory blocks allocated by Python." It "uses the domain 0 to trace memory allocations made by Python," while "C extensions can use other domains" — the basis for this report's lens‑boundary reasoning. https://docs.python.org/3/library/tracemalloc.html

[^np-tracemalloc]: NumPy Manual — *Memory management in NumPy*: "NumPy places its CPU memory allocations into the np.lib.tracemalloc_domain domain," so NumPy array allocations are visible to `tracemalloc` (empirically confirmed here: domain id `389047`, Section 8a). https://numpy.org/doc/stable/reference/c-api/data_memory.html
