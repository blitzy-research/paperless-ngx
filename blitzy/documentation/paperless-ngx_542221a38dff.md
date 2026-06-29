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

1. **Process RSS via `psutil`** — `psutil.Process().memory_info().rss`. This is the **authoritative total** and the *only* lens that captures C/C++‑extension memory (qpdf/`pikepdf`, and `numpy` buffers). It answers *"how much."*
2. **Python allocations via `tracemalloc`** — `tracemalloc.start()`, paired `take_snapshot()`, `compare_to('lineno')` for `file:line` attribution, and `get_traced_memory()` for the current/peak Python heap. It answers *"where, in Python."*
3. **Object accounting via `gc`** — `len(gc.get_objects())` for object‑count growth and `gc.collect()`'s return value for collectable reference **cycles**. It answers *"is anything leaking through cycles."*

We also used `resource.getrusage(RUSAGE_SELF).ru_maxrss` as a peak cross‑check and `ctypes` `malloc_trim(0)` to demonstrate return‑to‑OS behavior.

> **Critical methodological caveat — the observer effect.** `tracemalloc`, especially at a deep frame count, *itself consumes a large amount of memory* because it stores a traceback for **every** live allocation. In our environment, the identical `simple.txt` consume ended at **RSS ≈ 372 MB with `tracemalloc.start(25)` active** but at **RSS ≈ 132 MB with `tracemalloc` off** — roughly **240 MB of the "spike" was the profiler's own bookkeeping** for ~140 k allocations. We therefore treat **psutil‑only runs as authoritative for magnitude**, and use `tracemalloc` runs **only for attribution** (which `file:line` allocates, and how many objects). Every table below states which mode produced it.

### 1.5 The experiment matrix (one factor at a time)

* **Document types:** plain text (`simple.txt`, 21 B → `paperless_text`); PDF with embedded text (`simple.pdf`, 22,926 B → Tesseract parser, OCR skipped); scanned image needing OCR (`simple.png` 7,913 B, `examples/no-text.png` 32,595 B → Tesseract/`ocrmypdf`); Office via Tika — *reasoned from code only* (no Tika server was reachable, Section 6.5).
* **Batch sizes:** single document; sequential 10 and 50 documents in **one** process (worker recycling disabled) to expose cross‑document growth; contrasted with **fresh‑process‑per‑task** (`recycle=1` semantics).
* **Differential toggles:** classifier model **present vs. absent**; `DEBUG` **False vs. True**; barcode flag **off vs. on**; **import path vs. REST `metadata` endpoint**.

All inputs were the repository's own samples under `src/documents/tests/{samples,examples}`, copied to a scratch directory before consumption (so nothing in the repo was mutated).

---

## 2. TL;DR / Thesis

1. **The single largest import‑time spike is the machine‑learning classifier load**, not anything the user would call "metadata." `load_classifier()` builds a **fresh** `DocumentClassifier` and unpickles the model on **every call**, with **no cache** (`src/documents/classifier.py:L30‑L57`, `L76‑L94`). Measured cost: **≈ 90–93 MB of RSS at peak**, essentially **independent of the document** — a 21‑byte text file and a 22,926‑byte PDF both pay ~90 MB. That is the literal definition of "disproportionate to document size."

2. **The reason it is disproportionate** is that the classifier's size tracks the **training‑corpus vocabulary**, not the imported file. The unpickled model in our test was ~30 MB on disk; its in‑memory footprint is a `CountVectorizer` **vocabulary dictionary of ~107,000 small objects** (`classifier.py:L87`) plus three scikit‑learn estimators backed by **NumPy weight matrices** (`classifier.py:L90‑L92`).

3. **The worst‑case simultaneous live set** occurs **inside the `transaction.atomic()` block** (`src/documents/consumer.py:L298‑L306`), where the full extracted `text` (≡ `Document.content`), the loaded `classifier`, and the new `Document` instance are **all alive at once** while six post‑consume signal handlers run.

4. **There is no application cache accumulating data.** A repository‑wide search found **zero** cache primitives (`lru_cache`, `functools.cache`, `cache.set/get`, `LocMemCache`, `django.core.cache`) in the consumption path — the only match is a test override. Across a 50‑document in‑process batch, RSS reached a **steady‑state high‑water mark after the first document and grew only ~8 MB over the next 49** — a working set, not a leak.

5. **"Not released promptly" is by‑design and lives entirely in `Q_CLUSTER recycle=1`** (`settings.py:L452`). Within a task, freed memory is held in CPython/glibc allocator free‑lists (normal); **across** tasks, the worker is **fully restarted after every single task**, which is the *only* mechanism that returns memory to the OS. Release therefore happens **at task end**, not mid‑task.

6. **The retention is overwhelmingly NORMAL CPython allocator behavior, not a leak.** `gc.collect()` freed **0 reference cycles** in almost every run. Freed memory that "sticks" is **many‑small‑object pymalloc‑arena residue** (demonstrated in Section 8); large NumPy matrices are `mmap`'d and returned to the OS **promptly** on free. The one genuine *code smell* — `pikepdf.open()` without `close()`/context‑manager (`src/paperless_tesseract/parsers.py:L34`) — is on the **REST metadata path, not the import pipeline**, and even there it did **not** leak across repeated calls in this version.

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

The classifier is deliberately loaded **once per document** (the comment at `consumer.py:L288‑L290` explains it is to avoid reloading across the multiple post‑consume hooks). The **peak simultaneous live set** is inside the atomic block (stage 5–6): `text`, `classifier`, and the `Document` row coexist while the six handlers run.

---

## 5. Per‑question answers

### Q1 — What is causing the spikes? (root‑cause with module/method attribution)

**Answer: the classifier unpickle dominates, with a secondary fixed cost from parser/`dateparser` first‑use imports; transient file reads are negligible for normal file sizes.**

Authoritative **psutil‑only** per‑stage RSS deltas (MP_TRACE off), `simple.txt`, model present:

| Stage | ΔRSS (MB) | Notes |
|---|---:|---|
| `pre_check_duplicate` (MD5 full read) `consumer.py:L104` | +1.3 | first‑touch of `hashlib` + the duplicate DB query; the 21‑byte read is nil |
| `parse` (TextDocumentParser) `paperless_text/parsers.py:L40‑L42` | +0.4 | `f.read()` of 21 bytes |
| `get_text` `parsers.py:L342` | +0.0 | returns a reference |
| `parse_date` (dateparser) `parsers.py:L221‑L223` | +0.0 | already imported in this process |
| **`load_classifier` (unpickle)** `consumer.py:L292`, `classifier.py:L76‑L94` | **+93.0** | **the spike** |
| `_store` (DB + MD5 + copy) `consumer.py:L301/L402` | +0.1 | |
| `finished.send` (6 handlers + Whoosh) `consumer.py:L306` | +4.5 | |
| **Net consume** (baseline 60.3 → end 132.4) | **+72.1** | |

Reproduced three times: net `+72.0 / +72.0 / +72.2 MB`; the `load_classifier` stage `+93.1 / +93.0 / +93.3 MB` (variance < 0.5 MB).

**Why ≈ 90 MB peak but ≈ 57 MB retained?** The unpickle transiently allocates the ~30 MB pickle buffer and unpickling scratch (released before the function returns), while it permanently brings in the scikit‑learn/SciPy/NumPy import graph (held in `sys.modules`) and constructs the model object. The *retained* difference between model‑present and model‑absent runs is ≈ 57 MB (132.4 − 75.2; see Q4).

**Attribution via `tracemalloc`** (separate traced run, by `lineno`, baseline → peak) pinpoints the exact lines:

| Allocator (`file:line`) | Size | Object count | What it is |
|---|---:|---:|---|
| `documents/classifier.py:L87` | 10,968 KB | **107,087** | `data_vectorizer` = `CountVectorizer` **vocabulary dict** (many small `str`/`int`) |
| `documents/classifier.py:L90` | 9,410 KB | 190 | `tags_classifier` (NumPy weight matrix) |
| `documents/classifier.py:L91` | 9,404 KB | 149 | `correspondent_classifier` (NumPy weight matrix) |
| `documents/classifier.py:L92` | 9,402 KB | 191 | `document_type_classifier` (NumPy weight matrix) |
| `<frozen importlib._bootstrap_external>:647` | 16,800–20,100 KB | 100k–140k | module bytecode for the sklearn/scipy import graph |

This single table answers "which methods are responsible": **`DocumentClassifier.load()` at `classifier.py:L87‑L92`.** Note the shape difference — the vocabulary is **~107 k tiny objects** (~11 MB) while each estimator is **a handful of objects holding megabytes** of contiguous NumPy data. That distinction drives the normal‑vs‑problematic verdict in Q6 and Section 8.

**The disproportion, demonstrated.** The classifier cost is flat across document size and type (psutil‑only ΔRSS at the `load_classifier` stage):

| Input | Size | `load_classifier` ΔRSS |
|---|---:|---:|
| `simple.txt` | 21 B | +93.0 MB |
| `simple.pdf` (embedded text) | 22,926 B | +89.9 MB |
| `simple.png` (OCR) | 7,913 B | +90.5 MB |
| `examples/no-text.png` (OCR) | 32,595 B | +90.7 MB |

A 1,500×‑larger input changes the classifier cost by ~1 MB. The spike is governed by the **trained model**, not the document.

### Q2 — Is "metadata handling" creating unnecessary copies or holding references longer than needed?

**Answer: the import path makes several *transient* whole‑file copies (cheap for normal files) and holds three large objects across the atomic block; none of it is held *longer than needed* in a way that leaks. The "held references" are released the moment `try_consume_file()` returns.**

We traced the lifetime of each large object, forcing `gc.collect()` at known points to separate "still referenced" from "freed."

**(a) The extracted text — held once, mirrored once.** `text` is produced at `consumer.py:L271`, passed to `_store(text=...)` at `L301`, and persisted into `Document.content`, a `TextField` (`src/documents/models.py:L117`). So at peak there are effectively two references to the text: the local `text` and `document.content`. This is **one logical copy held deliberately** for the duration of the store + indexing, not an accidental duplication. For these samples the text is tiny; for a large OCR'd PDF it scales linearly with extracted text and is the legitimate cost of storing + indexing content.

**(b) The classifier — the big reference, but correctly scoped.** `classifier` is bound at `consumer.py:L292` and passed by reference into the signal (`classifier=classifier`, `L306` → `L310`); the handlers **reuse the one instance** (`handlers.py:L35/L101/L168`) rather than reloading — a deliberate, *good* choice (the comment at `consumer.py:L288‑L290`). Its scope is the `try_consume_file` frame; when the function returns, the only references vanish.

**(c) Whole‑file `f.read()` buffers — transient by construction.** The pipeline reads the entire file into memory at four sites: duplicate MD5 (`consumer.py:L104`), document MD5 in `_store` (`L402`), archive MD5 (`L340‑L341`), and the byte‑for‑byte copy in `_write` (`L429‑L432`, `write_file.write(read_file.read())`). Each is a short‑lived bytes object proportional to file size. Our CPython model demo (Section 8d) shows a ~22 MB `f.read()` is fully reclaimed on free. **Rationale for "unnecessary?":** these are *redundant* in the sense that the file is read multiple times, so for a very large file you can momentarily hold a few multiples of its size — but they are **not retained**; they do not accumulate.

**Is anything held *longer than needed*?** We measured RSS at four lifetime checkpoints for the `simple.txt` (model‑present) run:

| Checkpoint | RSS (MB) | Interpretation |
|---|---:|---|
| End of consume (function returned) | 132.4 | `text`/`classifier` locals already dropped with the frame |
| After `gc.collect()` | 132.4 | **0 cycles freed** → nothing was trapped in reference cycles |
| After `malloc_trim(0)` | 131.0 | only ~1.4 MB returned to OS |

The fact that RSS does **not** fall after `gc.collect()` even though the classifier is no longer referenced is the crux of the user's "held longer than needed" intuition — and the answer is: **the objects are *freed* (no longer referenced), but the freed memory remains in the process's allocator free‑lists**, not handed back to the OS. That is **normal CPython/glibc behavior** (Section 8), not a reference‑lifetime bug. The genuine cross‑task release is the worker restart (Section 7).

**"metadata" mapping:** the classification work (the real import‑time "metadata") creates the classifier reference described in (b); it does **not** create extra copies of the document. The REST `extract_metadata` copies are a separate path (Q6 / Section 6.6).

### Q3 — Is there caching behavior that accumulates data unexpectedly?

**Answer: No. There is no application‑level cache anywhere in the consumption path, and the per‑document classifier reload behaves like a *transient allocation*, not an accumulating cache.**

**Static evidence (grep‑confirmed absence).** A repository‑wide search of `src/` for cache primitives — `lru_cache`, `functools.cache`, `cache.set`, `cache.get`, `LocMemCache`, `from django.core.cache`, `caches[` — found **no matches in the consumption path**. The single hit anywhere is a test override (`src/documents/tests/test_classifier.py:393`). Likewise, there was **no** pre‑existing profiling/memoization harness (`tracemalloc`/`memory_profiler`/`psutil`/`objgraph`) anywhere in `src/`. So there is structurally no place for data to "accumulate in a cache."

**The classifier is the opposite of a cache.** `load_classifier()` (`classifier.py:L30`) constructs a **fresh** `DocumentClassifier()` (`L38`) and unpickles it (`L40`) on **every** call — there is no module‑level global, no `@lru_cache`, no memoization (`L30‑L57`). A cache would *retain* the model to save work; this code *rebuilds* it each time and lets it die at function end. Our model‑demo (Section 8b) confirms each `pickle.loads` produces an **independent fresh copy** (+30.4 MB, then +30.5 MB) — so repeated loads do not share storage, but neither do they accumulate, because each is released when the consume returns.

**Dynamic evidence (no cross‑document accumulation).** We consumed the **same document 50 times in one long‑lived process** (worker recycling disabled), deleting the created `Document` and forcing `gc.collect()` each iteration, and sampled RSS after each:

| Batch (one process) | Base RSS | After doc #1 | After doc #50 | Growth over 49 docs |
|---|---:|---:|---:|---:|
| `simple.txt`, model present | 60.2 | 132.1 | 140.4 | **+8.4 MB** |
| `simple.txt`, model absent | 60.2 | 75.3 | 78.0 | **+2.7 MB** |
| `simple.pdf`, model present (n=10) | 60.2 | 145.2 | 148.5 (#10) | +3.4 MB |

The object count (`len(gc.get_objects())`) held essentially flat (~125 k) across the 50 iterations. **Interpretation:** RSS jumps to a **steady‑state high‑water mark on the very first document** (paying the one‑time sklearn/scipy import + arena working set) and then is **flat** — the small residual creep (~0.17 MB/doc) is ordinary heap fragmentation, not unbounded growth. If a cache were accumulating, RSS would climb roughly linearly with document count; it does not. **There is no accumulating cache.**

### Q4 — What is different between the cases that spike and the cases that don't?

**Answer: the dominant differentiator by far is whether a trained classifier model file exists. `DEBUG` raises the baseline modestly; the barcode flag adds a transient; the REST `metadata` endpoint is a different path entirely.**

We varied one factor at a time (psutil‑only, fresh process each), measuring net consume RSS (`end − baseline`):

| Differential | Case | Baseline (MB) | End (MB) | Net (MB) | Effect |
|---|---|---:|---:|---:|---|
| **Classifier model** | `simple.txt`, **present** | 60.3 | 132.4 | **+72.1** | **dominant** |
| | `simple.txt`, **absent** | 60.2 | 75.2 | **+15.0** | model adds ≈ **+57 MB** |
| | `simple.pdf`, **present** | 60.2 | 145.3 | +85.1 | |
| | `simple.pdf`, **absent** | 60.3 | 91.1 | +30.8 | model adds ≈ **+54 MB** |
| **`DEBUG`** | `simple.txt`, `DEBUG=False` | 60.3 | 132.4 | +72.1 | baseline 60 MB |
| | `simple.txt`, `DEBUG=True` | **79.3** | 150.5 | +71.2 | baseline **+19 MB**; retains ~35 k more objects (160,795 vs 125,263) |
| **Barcode** (Section 6.5) | scan off | — | — | — | no extra work |
| | scan on (`tasks.py:L195/L105`) | 68.6 | 70.0 | +1.3 net | transient bitmap (tracemalloc **peak 7.8 MB**, current 0.66 MB) |
| **Path** (Q6) | import vs REST `extract_metadata` | — | — | — | different code; pikepdf only on REST |

**Reading the differential:**

* **Model present vs. absent is the whole story for "disproportionate."** When `settings.MODEL_FILE` is absent, `load_classifier()` returns `None` immediately (`classifier.py:L31/L36`) — the `load_classifier` stage costs **0 MB** and the consume of a 21‑byte file finishes at **75 MB** (mostly the one‑time parser/Django import baseline). When the model exists, the same consume hits **132 MB**. *The spike literally appears and disappears with the model file.* This is also why a freshly installed instance "doesn't spike" until enough documents have been added for the periodic `train_classifier()` task (`tasks.py:L48`) to produce a model.
* **`DEBUG=True` is a secondary, real factor.** It raises the resident baseline ~19 MB and causes more objects to be retained (Django records every SQL query on `connection.queries` while `DEBUG=True`). For a single document the end‑to‑end effect is small (~18 MB), but in a long‑running process it grows with query volume — a known Django gotcha, and a reason production runs with `DEBUG=False` (`settings.py:L50`).
* **The barcode flag** only matters if `PAPERLESS_CONSUMER_ENABLE_BARCODES` is set, in which case the *task* `consume_file` rasterizes PDF pages (`tasks.py:L195` → `scan_file_for_separating_barcodes` `L96` → `convert_from_path` `L105`) **before** `try_consume_file`. The rasterized bitmaps are a **transient** proportional to page‑count × DPI (peak 7.8 MB for a 1‑page PDF here; far larger for big, high‑resolution PDFs), reclaimed immediately after the scan.

### Q5 — How does behavior change with document type and batch size?

**Document type.** The classifier cost is constant (~90 MB, Q1); what changes with type is the **parser** cost:

| Type | Sample | Parser | `parse` ΔRSS | Net consume (model present) |
|---|---|---|---:|---:|
| Plain text | `simple.txt` (21 B) | `TextDocumentParser` (`paperless_text/parsers.py:L40`) | +0.4 MB | +72.1 MB |
| PDF, embedded text | `simple.pdf` (22.9 KB) | `RasterisedDocumentParser` (OCR skipped) | +23.0 MB | +85.1 MB |
| Image (OCR) | `simple.png` (7.9 KB) | `RasterisedDocumentParser` + Tesseract/`ocrmypdf` | +23.4 MB | +85.3 MB |
| Image (OCR) | `examples/no-text.png` (32.6 KB) | same | +26.2 MB | +87.8 MB |
| Office | (Tika) | `TikaDocumentParser` | *not run (Section 6.5)* | reasoned from code |

The text parser is essentially free (a 21‑byte `f.read()`). The PDF/image parser carries a **~23–26 MB fixed cost** — dominated by **importing** the OCR toolchain (`pikepdf`, `pdfminer`, `PIL`, `ocrmypdf`) on first use, plus modest per‑page work; embedded‑text PDFs and OCR'd images cost about the same here because the *import* dominates the tiny actual OCR. For large multi‑page scans, the per‑page OCR work would add on top of this fixed cost.

**Batch size.** This is where the user's "not released in a timely manner" observation is explained directly. Two regimes:

1. **Many documents in one process (recycling disabled)** — the realistic shape *if* `recycle` were large. RSS climbs to a **plateau on document #1** and stays there (Q3 table): `simple.txt` model‑present sat at **132–140 MB across 50 documents** (+8.4 MB total), `simple.pdf` at **145–149 MB across 10**. The plateau is the worker's **steady‑state working set** (one‑time imports + the high‑water classifier arena), *not* linear growth. So a single worker that processed many documents would settle at a high‑but‑bounded RSS — which *looks* like "memory never comes back" to an observer watching the process.

2. **Fresh process per task (`recycle=1`, the actual configuration)** — every separate‑process consume started at the **same ~60 MB baseline** and the OS reclaimed everything when the process exited. This is why, in production, RSS sawtooths: it spikes to ~130–150 MB during a document and drops back to ~60 MB when the worker is recycled after that single task (Section 7).

**The combined picture:** a *spike per document* (driven by the classifier, ~90 MB) on top of a *baseline that is reset per task by `recycle=1`*. Larger/again‑OCR documents add ~25 MB of parser cost; `DEBUG=True` raises the floor ~19 MB.

### Q6 — Evidence, and: is this normal Python GC behavior or something problematic?

**Answer: it is overwhelmingly NORMAL CPython allocator behavior, not a leak. There is exactly one genuine code smell (`pikepdf.open` without `close()`), and it is on the REST metadata path, not the import pipeline — and even there it did not leak in this version.**

**Evidence 1 — no reference cycles leak.** Across every single‑document run, `gc.collect()` freed **0 collectable cycles** (one PDF‑no‑model run freed 10 — trivial). If the pipeline were leaking through reference cycles, `gc.collect()` would report large collections and `len(gc.get_objects())` would climb run‑over‑run. Neither happened: in the 50‑document batch the object count stayed ~125 k.

**Evidence 2 — "freed but not returned to the OS" is the allocator, not a leak.** After a consume, RSS does not drop on `gc.collect()` (132.4 → 132.4 MB) because the freed memory sits in CPython's pymalloc arenas and glibc free‑lists. Section 8 reproduces this exactly with a controlled workload: a 2,000,000‑entry dict of small objects grows RSS +304 MB; after `del` + `gc.collect()` only 265 MB comes back and **~39 MB remains resident** until `malloc_trim(0)` (or process exit) returns it. The classifier's **~107 k‑object vocabulary** (`classifier.py:L87`) is exactly this many‑small‑object shape, so its freed memory is "sticky" within the worker's lifetime — and is reclaimed wholesale when `recycle=1` restarts the worker.

**Evidence 3 — the lenses agree on attribution, and NumPy is visible.** A common myth is "`tracemalloc` can't see NumPy." In this environment that is **false**: `numpy==1.22.3` registers its allocations with `tracemalloc` via the NumPy domain (`np.lib.tracemalloc_domain == 389047`). A controlled 48 MB array showed **+45.8 MB in BOTH** RSS and `tracemalloc` (Section 8a), and the classifier's NumPy weight matrices appear in `tracemalloc` attribution at `classifier.py:L90‑L92`. The genuinely `tracemalloc`‑**invisible** memory is **qpdf/`pikepdf` (C++)** — see Evidence 4 — for which RSS is essential.

**Evidence 4 — the one real smell, and why it still doesn't leak here.** `src/paperless_tesseract/parsers.py:L26‑L55` opens a PDF with `pdf = pikepdf.open(document_path)` (`L34`), reads metadata, and `return result` (`L55`) with **no `pdf.close()` and no `with` context‑manager**. qpdf holds C++ buffers that are released only when the `pdf` object is collected. We exercised this REST path directly, calling `extract_metadata()` ten times in a loop:

| | RSS (MB) | `tracemalloc` current (MB) |
|---|---:|---:|
| before | 51.1 | 0.0 |
| after call #1 | 64.2 (**+13.1**) | 1.6 (**+1.6 only**) |
| after call #5 | 64.3 | 1.6 |
| after call #10 | 64.3 | 1.6 |

Two findings: (1) **~11 MB of the +13 MB is invisible to `tracemalloc`** — that is the qpdf C++ allocation, confirming RSS is mandatory for this path; (2) **RSS is flat from call #1 to #10**, so the `pdf` local is in fact released by reference‑counting at each function return *despite* the missing `close()`. So in `pikepdf 5.1.1` this is a **latent code smell** (release depends on GC/refcount timing rather than being explicit) but **not an active leak**, and it is **not on the import pipeline** — only the REST details endpoint reaches it (`views.py:L269`). The cleaner form would be `with pikepdf.open(...) as pdf:`.

**Verdict.** The import‑time memory profile is *expected for a Django worker that unpickles a scikit‑learn model per task*: a large, document‑independent classifier load, a fixed parser/import baseline, transient file reads, and allocator free‑list retention that the per‑task worker restart cleans up. It is **normal**, not pathological. The only thing one might *change* (out of scope here, since this is diagnosis only) would be to cache/reuse the classifier across tasks or raise `recycle` — but both trade memory‑release timeliness for CPU, and the current `recycle=1` is a deliberate "release memory aggressively" choice.

---

## 6. Measured tables (per experiment)

All RSS figures are **psutil‑only (authoritative)** unless a row is explicitly labeled *traced*. `tracemalloc` current/peak and top allocators come from separate **traced** runs (and are inflated for RSS by the observer effect, Section 1.4).

### 6.1 Document‑type matrix (single document, classifier present, psutil‑only)

| Type / sample | Size | Baseline RSS | End RSS | Net | `parse` Δ | `load_classifier` Δ | `finished.send` Δ | after `gc` (cycles) | after `malloc_trim` |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| text `simple.txt` | 21 B | 60.3 | 132.4 | +72.1 | +0.4 | +93.0 | +4.5 | 132.4 (0) | 131.0 |
| pdf `simple.pdf` | 22.9 KB | 60.2 | 145.3 | +85.1 | +23.0 | +89.9 | +4.4 | 145.3 (0) | 143.7 |
| pdf `test_with_bom.pdf` | 16.1 KB | 60.2 | ~145 | ~+85 | +23 | +90 | +4.4 | — | — |
| png OCR `simple.png` | 7.9 KB | 60.4 | 145.6 | +85.3 | +23.4 | +90.5 | +4.5 | 145.6 (0) | 144.0 |
| jpg OCR `simple.jpg` | 17.7 KB | 60.1 | ~146 | ~+86 | +23 | +90 | +4.4 | — | — |
| png OCR `examples/no-text.png` | 32.6 KB | 60.2 | 148.0 | +87.8 | +26.2 | +90.7 | +4.2 | 148.0 (0) | 144.7 |

### 6.2 Classifier present vs. absent (single document, psutil‑only)

| Sample | Model | Baseline | End | Net | `load_classifier` Δ |
|---|---|---:|---:|---:|---:|
| `simple.txt` | present | 60.3 | 132.4 | +72.1 | +93.0 |
| `simple.txt` | absent | 60.2 | 75.2 | +15.0 | +0.0 |
| `simple.pdf` | present | 60.2 | 145.3 | +85.1 | +89.9 |
| `simple.pdf` | absent | 60.3 | 91.1 | +30.8 | +0.0 |

Reproducibility (3 reps each, psutil‑only): text present net `+72.0/+72.0/+72.2`; text absent `+15.1/+15.4/+15.1`. Variance < 0.5 MB.

### 6.3 `DEBUG` False vs. True (`simple.txt`, model present)

| `DEBUG` | Baseline | End | Net | `gc.get_objects()` at end |
|---|---:|---:|---:|---:|
| False | 60.3 | 132.4 | +72.1 | 125,263 |
| True | 79.3 | 150.5 | +71.2 | 160,795 |

### 6.4 Batch sweep

| Batch (ONE process, recycle disabled) | Base | doc #1 RSS | last‑doc RSS | growth | object count |
|---|---:|---:|---:|---:|---:|
| `simple.txt`, model present, n=50 | 60.2 | 132.1 | 140.4 | +8.4 over 49 | ~125 k flat |
| `simple.txt`, model absent, n=50 | 60.2 | 75.3 | 78.0 | +2.7 over 49 | ~88 k flat |
| `simple.pdf`, model present, n=10 | 60.2 | 145.2 | 148.5 | +3.4 over 9 | flat |

Fresh‑process‑per‑task (`recycle=1`): every separate process starts at ~60 MB and frees everything on exit.

### 6.5 Barcode rasterization transient (`simple.pdf`)

| | RSS (MB) | `tracemalloc` (cur, peak) |
|---|---:|---|
| before scan | 68.6 | (0.0, 0.0) |
| after `convert_from_path` + scan (`tasks.py:L105`) | 70.0 (**+1.3** net) | (0.66, **7.8**) |
| after `gc.collect()` | 70.0 | (0.66, 7.8) |

The high `tracemalloc` **peak (7.8 MB)** vs low **current (0.66 MB)** is the signature of a transient bitmap that has already been reclaimed. Office/Tika was **not exercised at runtime** because no Tika server was reachable (`localhost:9998` → `ConnectionRefusedError`); see Section 6.6 for the code‑based reasoning.

### 6.6 REST `metadata` endpoint (NOT the import path) — `extract_metadata` loop

| | RSS (MB) | `tracemalloc` current (MB) |
|---|---:|---:|
| before | 51.1 | 0.0 |
| after call #1 | 64.2 (+13.1) | 1.6 |
| after call #5 | 64.3 | 1.6 |
| after call #10 | 64.3 | 1.6 |

**qpdf is `tracemalloc`‑invisible (+13 MB RSS vs +1.6 MB traced) and does not accumulate across calls** (Q6, Evidence 4). For the Tika branch (`paperless_tika/parsers.py`), reasoning from code: `extract_metadata` calls `parser.from_file()` (`L32`) which loads the **entire** parsed content+metadata dict just to read metadata keys (`L45`), and `parse()` assigns `self.text = parsed["content"].strip()` (`L62`) — i.e., the whole document content is materialized in memory; this scales with document size and is the Tika analog of the OCR parser's content cost.

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

* `recycle` — <cite index="1-11,1-12">the number of tasks a worker will process before recycling, useful to release memory resources on a regular basis</cite>, and it <cite index="4-10">defaults to 500</cite>.
* `max_rss` — <cite index="5-16">the maximum resident set size in k[B]</cite> above which a worker is recycled; it is unset here (default `None`), so RSS‑threshold recycling is disabled.

Paperless overrides `recycle` from the default of 500 down to **1**. The cluster's internal sentinel <cite index="3-7">checks the health of all the processes and recycles or reincarnates according to your settings</cite>. With `recycle: 1`, **the worker process is fully torn down and a fresh one spawned after every single task** — i.e., after every single document.

**What that means for memory.** Process teardown is the one event that unconditionally returns *all* of a process's memory to the OS — pymalloc arenas, glibc free‑lists, loaded C‑extension data, everything. So:

* **Within a task**, freed Python objects are *not* promptly returned to the OS; they sit in allocator free‑lists (Section 8). RSS for a single document therefore *stays* near its high‑water mark from the `load_classifier` peak (~130–150 MB) until the task ends — this is exactly the "memory isn't released in a timely manner" the user sees if they watch the worker mid‑task or just after.
* **At task end**, `recycle: 1` restarts the worker, and RSS drops back to the ~60 MB fresh baseline we measured for every new process. The production memory graph is therefore a **sawtooth**: ~60 MB → spike to ~130–150 MB during a document → back to ~60 MB on recycle.

**Why this design exists.** Recycling after every task is the project's deliberate, blunt defense against exactly the kind of allocator residue this report documents: rather than trying to make the per‑document classifier load cheap or cached, Paperless simply throws the worker away and starts clean. The trade‑off is CPU/startup cost (re‑importing Django + sklearn on every task) in exchange for a guaranteed memory reset. (A real‑world Django‑Q memory‑growth report, GitHub issue #621, shows what happens *without* effective recycling — RSS <cite index="8-3">keeps going higher gradually and ends up eating all the memory</cite> — which is the failure mode `recycle: 1` is designed to prevent.)

**Corollary.** If someone raised `recycle` to a large number (or the worker processed many tasks before recycling), our batch measurements (Section 6.4) predict RSS would climb to a **bounded plateau** (~130–150 MB for the classifier working set) and stay there — *not* an unbounded leak, but a sustained high‑water mark that "never comes back" until the next recycle. That is the precise behavioral knob behind "spikes that don't come down."

---

## 8. CPython memory model — the interpretive lens (controlled demos)

To judge "normal vs. problematic," these controlled demonstrations were run in the **same** Python 3.9.23 / `numpy==1.22.3` container (reproduced across two runs; numbers stable). They establish four facts the pipeline analysis relies on.

| Demo | Setup | RSS | `tracemalloc` | Takeaway |
|---|---|---|---|---|
| **(a)** NumPy visibility | allocate 48 MB `np.ones` | +45.8 MB | **+45.8 MB** | NumPy **is** tracemalloc‑visible (`np.lib.tracemalloc_domain == 389047`) |
| **(c‑large)** big array free | `del` + `gc.collect()` | **−45.8 MB** (returned fully) | — | large (>128 KB) allocations are `mmap`'d and `munmap`'d **promptly** on free |
| **(b)** pickle copies | `pickle.loads` of a 32 MB array, twice | +30.4, then +30.5 MB | — | each `pickle.loads` is an **independent fresh copy** (no sharing/cache) |
| **(c‑small)** many small objects | build a 2,000,000‑entry dict of small `str`/`int` | +304.4 MB | — | (see below) |
| ↳ free it | `del` + `gc.collect()` | −265.5 MB (≈ **39 MB stays resident**) | — | small‑object memory is **sticky** in pymalloc arenas after free |
| ↳ `malloc_trim(0)` | | −41.8 MB (back to baseline) | — | glibc returns the residue only on explicit trim / process exit |
| **(d)** whole‑file read | `f.read()` ~22 MB | +21.9 MB | — | transient |
| ↳ free it | `del` + `gc.collect()` | 0 MB returned | — | went to glibc main‑arena free‑list (dynamic mmap threshold) |
| ↳ `malloc_trim(0)` | | −21.9 MB (returned fully) | — | freed at the Python level; returned to OS on trim/reuse |

**How this maps onto the pipeline:**

* **(a)+(c‑large)** explain the **NumPy weight matrices** (`classifier.py:L90‑L92`, ~9.4 MB each): they are visible to `tracemalloc`, and because each is a single large contiguous buffer, it is returned to the OS *promptly* when the classifier is freed. **These are not the sticky residue.** This also corrects the common myth — for `numpy 1.22.3`, "tracemalloc can't see NumPy" is **false**; the genuinely invisible memory is qpdf/`pikepdf` C++ (Section 6.6).
* **(b)** is the **classifier‑per‑document** behavior in miniature: `load_classifier()` produces a fresh independent copy every call (`classifier.py:L38/L40`), exactly like repeated `pickle.loads`. No cache, no sharing — and no accumulation, because each copy dies at task end.
* **(c‑small)** is the **`CountVectorizer` vocabulary** (`classifier.py:L87`, ~107 k tiny objects) in miniature: after the model is freed, much of its memory **stays resident in pymalloc arenas** until `malloc_trim(0)` or — in production — the `recycle: 1` worker restart. **This is the "freed‑but‑not‑returned‑to‑OS" residue, and it is normal CPython behavior, not a leak.**
* **(d)** is the **whole‑file `f.read()`** buffers (`consumer.py:L104/L340/L402/L432`): transient, freed at the Python level, returned to the OS by trim or reuse. No accumulation.

---

## 9. Component‑by‑component memory attribution

Every memory holder and transient site in the import path (plus the REST‑only holders), mapped to its exact `file:line` and its measured contribution. "Measured contribution" is the RSS or `tracemalloc` figure from Section 6; "lifetime" is when the memory is released.

| Component | `file:line` | Kind | Measured contribution | Lifetime / release |
|---|---|---|---|---|
| **Classifier unpickle (whole)** | `classifier.py:L76‑L94` (`load_classifier` `L30/L38/L40`) | Python objects + NumPy | **+90–93 MB RSS** at the `load_classifier` stage (dominant spike); ~74–83 MB `tracemalloc` peak | Held to end of `try_consume_file`; freed when `classifier` goes out of scope; OS‑returned on `recycle: 1` |
| ↳ `CountVectorizer` vocabulary dict | `classifier.py:L87` | many small `str`/`int` | ~10.97 MB across **107,087 objects** (top `tracemalloc` lineno) | Sticky pymalloc‑arena residue after free (Section 8 c‑small) |
| ↳ 3 MLP/estimator weight matrices | `classifier.py:L90`, `L91`, `L92` | NumPy arrays | ~9.41 / 9.40 / 9.40 MB; ~150–191 objects each | `mmap`'d → returned to OS promptly on free (Section 8 c‑large) |
| ↳ first‑time sklearn/scipy import | `<frozen importlib._bootstrap_external>:647` (triggered by `classifier.py` imports) | bytecode + module objects | ~16.8–20.1 MB across 100k–140k objects | One‑time per worker; reset on recycle |
| **Extracted `text` (≡ `Document.content`)** | `consumer.py:L271` (`get_text`); held to `L306`; mirrored at `models.py:L117` | Python `str` | Scales with document text length (small for test corpus; negligible vs. classifier) | Single copy; alive across `transaction.atomic()` `L298‑L306`; freed at task end |
| **Atomic‑block co‑retention (peak live set)** | `consumer.py:L298‑L306` | composite | **The worst‑case instant:** `text` + classifier + `Document` row + transient checksum/copy buffers all live while 6 handlers run | Peak; collapses after the `with` block |
| **Parser `self.text`** | `parsers.py:L296` (init), `L342` (`get_text`); text parser read at `paperless_text/parsers.py:L42` | Python `str` | Same order as `text`; one buffer per parser instance | Freed when parser is GC'd after `try_consume_file` |
| **Whole‑file `f.read()` — duplicate pre‑check** | `consumer.py:L104` | transient `bytes` | Proportional to file size (≤ ~32 KB for samples); reclaimed immediately | Transient (Section 8 d) |
| **Whole‑file `f.read()` — archive checksum** | `consumer.py:L340‑L341` | transient `bytes` | Proportional to archive file size | Transient |
| **Whole‑file `f.read()` — `_store` checksum** | `consumer.py:L402` | transient `bytes` | Proportional to file size | Transient |
| **Whole‑file copy — `_write`** | `consumer.py:L429‑L432` (`write_file.write(read_file.read())`) | transient `bytes` | Proportional to file size | Transient |
| **Whoosh `AsyncWriter` index buffer** | `index.py:L66` (writer), `L87‑L93` (`update_document`, `content=doc.content`) | Python/Whoosh | Buffers the full content field during the `add_to_index` handler | Flushed/committed by the writer; small for test corpus |
| **`dateparser` first‑use language data** | `parsers.py:L221‑L223` (`import dateparser`; `dateparser.parse(...)`) | Python module data | Lazy heavy load on first call → raises resident **baseline** (paid once per worker, i.e., per task under `recycle: 1`) | Part of baseline; reset on recycle |
| **Barcode rasterization (feature‑flagged)** | `tasks.py:L105` (`convert_from_path`), gated by `L195` (`CONSUMER_ENABLE_BARCODES`) | `pdf2image` bitmaps | **+1.3 MB net RSS; ~7.8 MB `tracemalloc` peak** (transient bitmap), simple.pdf | Transient; reclaimed after scan |
| **Django‑Q pickled task payload** | overrides passed as scalar kwargs to `consume_file` (`tasks.py:L184`) | pickled args | Small (no `data_models.py` overrides object in 1.3.9) — **not a suspect** | Held by broker/ORM, not the worker working set |
| **REST‑only: `pikepdf.open` without `close()`** | `paperless_tesseract/parsers.py:L34` (open), `L55` (return, no close) | qpdf **C++** | **+13 MB RSS but only +1.6 MB `tracemalloc`** (C++ invisible); **stable across 10 calls — no leak** | Released by refcount on return; **release‑timing smell, REST path only — not import** |
| **REST‑only: Tika full content+metadata load** | `paperless_tika/parsers.py:L32` (`from_file` for metadata), `L55/L62` (`parse`) | Python dict/`str` | Loads entire parsed dict to read metadata; scales with document size | Per‑request; REST path (and Office `parse`), not the import classification work |

**Reading the table:** the single dominant import‑time contributor is the classifier unpickle (90–93 MB), which is **independent of the imported document's size** — the root of "disproportionate." Everything else in the import path is either small‑and‑persistent (`text`/`content`), transient (file reads, barcode bitmaps), or baseline (`dateparser`, first‑time imports). The two REST‑only holders are *not* on the import path and must not be conflated with it.

---

## 10. Version‑specificity of these conclusions

Every number and conclusion in this report is tied to the dependency pins of this commit. The following version facts materially shape the findings:

* **`django-q==1.3.9` (Koed00 Django‑Q, *not* Celery, *not* django‑q2).** This is verified by `requirements.txt` and by the presence of `Q_CLUSTER` in `settings.py:L449`. The release behavior (Section 7) is governed by this version's `recycle`/`max_rss` semantics. Critically, **this version has no `data_models.py` overrides object** — consumption overrides are passed to `consume_file` (`tasks.py:L184`) as **scalar keyword arguments**, so the pickled task payload (via `django-picklefield==3.0.1`) is small and is *not* a memory suspect. A newer Paperless version that introduced a structured overrides object would change that calculus.
* **`scikit-learn==1.0.2` + `numpy==1.22.3`.** The classifier's `FORMAT_VERSION = 7` (`classifier.py:L63`) and the six‑object pickle layout (`classifier.py:L86‑L92`) are specific to this serialization. The **`tracemalloc`‑visibility of NumPy** (Section 8 a) is a property of `numpy 1.22.3`, which registers allocations in `np.lib.tracemalloc_domain` (observed value `389047`). On a NumPy build without that hook, the weight matrices would be RSS‑only — so the "report what each lens shows" caveat is version‑bound. The absolute classifier size (~30.9 MB on disk; ~90 MB load spike) scales with **training‑corpus vocabulary** (here vocab=4000 over 400 docs); a larger production model produces a proportionally larger spike.
* **`pikepdf==5.1.1` (qpdf C++).** The `extract_metadata` "no `close()`/no context manager" pattern (`paperless_tesseract/parsers.py:L34/L55`) holds qpdf buffers until refcount‑drop on return. In this version the object *is* released promptly by refcount (measured stable across 10 calls, Section 6.6), so the smell does not manifest as a leak — but that depends on `pikepdf`/qpdf not introducing additional retained references, which is version‑dependent.
* **`tika==1.24`.** The Office‑document matrix cell was **not exercised at runtime** (no Tika/Gotenberg server was reachable — connection refused on `:9998`); its memory behavior is reasoned from code (`paperless_tika/parsers.py:L32/L55/L62`), not measured. This is the one matrix cell without direct RSS evidence, and it is flagged as such.
* **Python 3.9.23.** The pymalloc‑arena and glibc `malloc_trim` behavior (Section 8) is characteristic of CPython 3.9 on glibc Linux. The observer effect of `tracemalloc.start(25)` (Section 1) — ~240 MB of traceback storage — is also interpreter‑ and frame‑depth‑specific.

---

## 11. Closing — where the memory actually goes

Putting the three lenses and all experiments together, the per‑document memory story of the Paperless‑ngx import pipeline is:

1. **A fresh worker starts at ~60 MB** (Django + first‑time imports + `dateparser` language data baseline).
2. **The parser reads/extracts the document** — a transient file read (`f.read()`, Section 8 d) plus the persistent `text`/`Document.content` string. For PDFs/images this adds ~23–26 MB of parser/OCR working set (`consumer.py:L261`); for plain text it is negligible.
3. **`load_classifier()` fires the dominant spike** — **+90–93 MB RSS** (`classifier.py:L76‑L94`), *independent of the document size*. Of that, the NumPy weight matrices (~28 MB, `L90‑L92`) are large `mmap`'d buffers returned promptly on free, while the `CountVectorizer` vocabulary (~11 MB across 107 k tiny objects, `L87`) becomes **sticky pymalloc‑arena residue** after free. The rest is one‑time sklearn/scipy import cost.
4. **The `transaction.atomic()` block (`consumer.py:L298‑L306`) is the peak live set** — `text` + classifier + `Document` row + transient checksum/copy buffers, all alive while six signal handlers run.
5. **Nothing accumulates across documents within a worker** — the batch sweep (Section 6.4) shows RSS reaching a **bounded plateau** after the first document (~130–150 MB) and *not* climbing thereafter; `gc.collect()` frees **0 reference cycles** in nearly every run. There is **no application cache** in the import path (grep‑confirmed, Q3).
6. **All of it is returned to the OS at task end via `Q_CLUSTER recycle: 1`** (`settings.py:L452`) — a full worker restart after every document. This is the *only* cross‑task release mechanism, and it is why production memory is a **sawtooth** (~60 MB → ~130–150 MB → ~60 MB), not a monotonic climb.

**Verdict:** the spikes are **disproportionate to document size** because they are dominated by the **training‑corpus‑sized classifier**, reloaded fresh per document with **no cache**; the memory that "isn't released in a timely manner" is **normal CPython allocator residue** (pymalloc arenas / glibc free‑lists holding the many‑small‑object vocabulary), distinguished from a genuine leak by forced `gc.collect()` (0 cycles) + `malloc_trim(0)` (returns the residue). The release is deferred only until the **per‑task worker recycle**, by design. The one genuine code smell — `pikepdf.open` without `close()` (`paperless_tesseract/parsers.py:L34`) — lives on the **REST `metadata` path**, not the import pipeline, and does not leak in these versions. **No source change is recommended by this read‑only engagement;** the behavior is well‑characterized, bounded, and normal for the stack.

---

## Appendix A — Measurement methodology (throwaway harness)

All runtime numbers were produced by a **temporary** triple‑lens harness written outside the repository tree (under `/tmp/`), driving the **real** `Consumer().try_consume_file(...)` against temp copies of the repository's own sample inputs (`src/documents/tests/samples/`, `examples/`). The harness was deleted on completion; the only committed artifact is this document. It sampled, around each pipeline stage:

* **Process RSS** via `psutil.Process().memory_info().rss` — the authoritative total, and the only lens that captures qpdf C++ memory.
* **Python allocations** via `tracemalloc` (`start(25)`, paired `take_snapshot()`, `compare_to('lineno'|'traceback')`, `get_traced_memory()`) — for `file:line` attribution.
* **Object accounting** via `gc` (`get_count()`, `len(get_objects())`, `collect()`), `resource.getrusage(...).ru_maxrss` as a peak cross‑check, and `ctypes` `malloc_trim(0)` to demonstrate return‑to‑OS.

To avoid the `tracemalloc` observer effect (Section 1), the authoritative RSS tables (Sections 6.1–6.5) were captured with `tracemalloc` **off** (`psutil`‑only); the attribution tables (Section 5, Q1) were captured with `tracemalloc` **on** in separate runs. A real classifier model was trained (400 docs, vocab=4000, ~30.9 MB pickle, ~9.2 MB MLP weights) so the classifier‑present differential exercised genuine vocabulary + NumPy matrices. Key experiments were repeated three times; variance was < 0.5 MB.
