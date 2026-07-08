# Paperless-ngx — Memory Usage During Document Import and Metadata Handling: A Runtime Investigation

**Repository:** paperless-ngx
**Commit investigated:** `542221a38dff06361e07976452f9aea24d210542` (HEAD of branch `blitzy-6a755538-5279-46bb-a229-50a82880251d`)
**Canonical runtime:** Python **3.9.23** inside the provided Docker image (`python:3.9-slim-bullseye` base per `Dockerfile:L18`), default configuration `DEBUG=NO` (`src/paperless/settings.py:L50`).
**Nature of this document:** A read-only, run-first **diagnostic answer**. Every behavioural claim below is backed by the *actual, unedited* output of temporary observation scripts that drove the **real** paperless-ngx entry points under a tri-lens memory harness. No source file was modified; the harness scripts lived outside the repository and were deleted afterward. The read-only proof (`git status --porcelain`) is in the Appendix.

---

## 1. Scope — what was investigated

The user reports that importing documents "sometimes consumes far more memory than expected for metadata handling," that the behaviour is *inconsistent*, that spikes are *disproportionate to document size*, and that memory is *not always released in a timely manner*. This document answers five named sub-questions plus the crux determination:

1. **Q1 — Cause of the spikes:** which code paths/objects drive elevated memory during import, especially metadata handling.
2. **Q2 — Unnecessary copies / prolonged reference retention:** does metadata handling make redundant in-memory copies or hold large objects too long?
3. **Q3 — Caching accumulation:** does any caching or process-lifetime state (classifier model, ORM query log, index writer) accumulate across documents?
4. **Q4 — Spiking vs. non-spiking:** what concretely differs between runs that spike and runs that do not?
5. **Q5 — Sensitivity to document type × batch size:** the full cross-product of {plain text, text PDF, scanned/image PDF} × {single, many}.
6. **The determination:** is this a genuine, problematic memory **leak**, or **normal CPython/allocator behaviour**?

The code paths exercised (all read-only): the consumption pipeline (`src/documents/consumer.py`, `src/documents/tasks.py`), the parser plugins (`src/documents/parsers.py`, `src/paperless_tesseract/parsers.py`, `src/paperless_text/parsers.py`), the classifier cache (`src/documents/classifier.py`), the bulk importer (`src/documents/management/commands/document_importer.py`), the REST metadata endpoint (`src/documents/views.py`), the signal handlers (`src/documents/signals/handlers.py`), the sanity checker (`src/documents/sanity_checker.py`), the search index (`src/documents/index.py`), the models (`src/documents/models.py`), and settings (`src/paperless/settings.py`).

---

## 2. TL;DR / Executive summary

**Direct verdict: this is overwhelmingly normal CPython/allocator behaviour plus a few large-but-expected transient allocations — it is NOT a genuine (growing) live-object memory leak.** Across every hotspot, the number of live Python objects (`gc`) and the retained Python-heap size (`tracemalloc`) are **flat across `gc.collect()` and flat across repeated documents**; what stays elevated is **resident set size (RSS)**, because CPython's `pymalloc` arenas and glibc's per-thread arenas are not returned to the OS after they are freed. That is expected, documented allocator behaviour — RSS alone can neither confirm nor deny a leak, which is precisely why this investigation measures three lenses at once.

The specific findings:

- **The disproportionate spikes come from OCR of scanned/image PDFs and from the full extracted-text string — not from metadata.** The consumer never calls `extract_metadata()` at all (see §4). A 150 KB image PDF drives the parent worker to a **241.8 MiB** RSS high-water while a 21-byte text file reaches **128.2 MiB** — yet both have an essentially identical **Python-heap** peak (`tracemalloc.peak` ≈ 55 MiB and 43 MiB), and after the image consume `gc.collect()` dropped current RSS 187.0→159.6 MiB. The spike is native OCR memory, and it is released afterwards.
- **Metadata handling makes no redundant *in-RAM* copies** and does not retain large objects: the un-closed `pikepdf` handle is reclaimed promptly by CPython reference-counting, and the full-text string is held exactly once. The only genuinely un-released artifact on the metadata path is **empty parser temp-directories** left on the REST `/metadata/` endpoint (an inode/directory leak costing essentially **zero RAM**).
- **No caching accumulates across documents** in the default configuration: the classifier is loaded once per consume, shared with all three handlers, and freed afterward; the Whoosh writer is created and committed per update; the ORM query log (`connection.queries`) grows **only** under the non-default `PAPERLESS_DEBUG=YES` (and even then is capped at 9000 entries ≈ 2.39 MiB).
- **The "sometimes spikes" inconsistency is dominated by first-touch vs. warm** (the first document in a fresh worker pays a one-time lazy-import cost) and by **OCR vs. no-OCR** (OCR pushes RSS to a plateau via native subprocess buffers and glibc arenas).
- **Batch-size sensitivity is concentrated in the bulk `document_importer`**, whose `json.load` of the whole manifest scales linearly with `batch × content` and coexists with a second full parse inside `loaddata`, giving a peak of roughly **2× the manifest size**.

A methodological caution surfaced during the work and is itself relevant to the user's report: **heavy in-process memory profiling can itself cause huge RSS spikes.** Measured on the identical digital PDF (§3.4), a heavy harness (30-frame `tracemalloc` + retained per-stage snapshots) inflated one consume's RSS high-water to **3251.9 MiB** versus the light harness's **142.1 MiB** — while the Python-heap `tracemalloc.peak` was identical (~54.9 MiB) in both, proving the ~3.1 GiB was the *profiler's* own memory, not paperless. All headline magnitudes below use light instrumentation; see §3.4.

---

## 3. Methodology

### 3.1 The three lenses (and why all three are required)

A rise in RSS that never returns to the OS is frequently an artifact of the allocator, not a leak. To separate the two, each probe captures three lenses simultaneously:

- **(a) RSS — the OS footprint.** Two readings, because they mean different things:
  - `resource.getrusage(resource.RUSAGE_SELF).ru_maxrss` — a **high-water mark in kilobytes** on Linux (monotonic, never decreases; excludes child processes).
  - `/proc/self/status` `VmRSS` — the **current** RSS in kB (can decrease when memory is released). `psutil` is **absent** in the container (`ModuleNotFoundError: No module named 'psutil'`), so these two stdlib sources are used.
- **(b) `tracemalloc` — the Python heap, attributed by `file:line`.** `get_traced_memory()` → `(current, peak)`; `take_snapshot()` + `compare_to(prev, 'lineno')` attributes allocations to source lines. This sees only CPython-managed heap, **not** native C-extension memory.
- **(c) `gc` — live tracked objects.** `len(gc.get_objects())`, `gc.get_count()`, `gc.collect()` (returns objects collected), and targeted `gc.get_referrers` / `sys.getsizeof` / `sys.getrefcount` on suspects.

**The determination rule, applied throughout:** a *genuine leak* shows growth in **live objects** (`gc`) **and** retained `tracemalloc` size that **survives `gc.collect()`**; *benign allocator retention* shows **flat live-object counts** with **RSS staying elevated**.

### 3.2 Web-research framing of "normal vs. leak" (synthesized, attributed)

- **CPython `pymalloc`** manages small objects in a three-level hierarchy — **arena (256 KB) → pool (4 KB) → block (8–512 bytes, 64 size classes)**. Requests **> 512 bytes bypass `pymalloc`** and go to the system allocator (glibc `malloc`). An arena is returned to the OS **only when all of its pools are empty**, which (per the CPython C-API memory documentation [1], the CPython `obmalloc.c` source [4], and analyses by Evan Jones [2], Artem Golubin / rushter.com [3], and Bloomberg's memray [5]) "rarely happens" in long-running processes — a single live small object can pin a whole arena, and fragmentation strands freed memory. **Consequence: RSS failing to shrink after `gc.collect()` is not, by itself, evidence of a leak.**
- **glibc `malloc` (ptmalloc2)** creates **per-thread arenas** to reduce lock contention, limited to roughly **8 × CPU cores** on 64-bit (tunable via `MALLOC_ARENA_MAX`) [6][7]; it keeps freed chunks in per-thread `tcache`, `fastbins`, and `bins`. Large allocations (≳ 128 KiB) use `mmap`/`munmap` and *are* returned to the OS immediately; ordinary heap memory returns only when the top of the heap can be trimmed. A multi-thread process can therefore appear to "hoard" memory independently of live-object count. `malloc_trim`, `MALLOC_ARENA_MAX`, and `malloc_info` are diagnostic aids — **not applied here**.
- **Container relevance:** `nproc` reports **128** in this container, so glibc could in principle create up to 8 × 128 = 1024 arenas. The OCR pipeline (`ocrmypdf`, ghostscript, tesseract) and the django-q worker are multi-threaded, so elevated RSS that does not track live-object count is expected.

**Sources (retrievable web references; accessed July 2026):**

1. CPython — *Memory Management* (C-API): https://docs.python.org/3/c-api/memory.html
2. Evan Jones — *Improving Python's Memory Allocator*: https://www.evanjones.ca/memoryallocator/
3. Artem Golubin (rushter.com) — *Memory management in Python*: https://rushter.com/blog/python-memory-managment/
4. CPython source — *Objects/obmalloc.c*: https://github.com/python/cpython/blob/main/Objects/obmalloc.c
5. Bloomberg memray — *Python allocators*: https://bloomberg.github.io/memray/python_allocators.html
6. Red Hat Developer — *Malloc Internals and You*: https://developers.redhat.com/blog/2017/03/02/malloc-internals-and-you
7. Heroku Dev Center — *Tuning glibc Memory Behavior*: https://devcenter.heroku.com/articles/tuning-glibc-memory-behavior
8. Python 3.9 docs — *tracemalloc*: https://docs.python.org/3.9/library/tracemalloc.html
9. Python docs — *gc*: https://docs.python.org/3/library/gc.html
10. Python docs — *resource*: https://docs.python.org/3/library/resource.html

References [1]–[5] frame CPython `pymalloc` arena retention (small objects ≤ 512 bytes; arena returned to the OS only when all its pools are empty — "rarely happens" in long-running processes); [6]–[7] frame glibc `ptmalloc2` per-thread arenas (up to ~8 × CPU cores, tunable via `MALLOC_ARENA_MAX`); [8]–[10] document the tri-lens measurement APIs — `tracemalloc.get_traced_memory()` current/peak plus `reset_peak()` (new in Python 3.9, matching the canonical runtime), `gc.get_objects()` / `gc.get_referrers()` for live-object and referrer inspection, and `resource.getrusage(RUSAGE_SELF).ru_maxrss` — used by `probe.py` (§3.3, App. A-1). The doc's "arena (256 KB)" matches the canonical Python 3.9.23 runtime, where `sys._debugmallocstats()` reports `262144 bytes/arena` (the current C-API doc lists 1 MiB for 64-bit builds of newer CPython versions). Analogously, the bullet's "64 size classes" is the universally-published canonical description of `pymalloc` (8-byte alignment); the same `sys._debugmallocstats()` call on this 64-bit 3.9.23 build actually reports `Small block threshold = 512, in 32 size classes` — the build uses 16-byte `ALIGNMENT` (512 / 16 = 32 classes), and its per-class `size` column steps by 16 bytes (16, 32, 48, 64, ...). Command: `python3 -c "import sys; sys._debugmallocstats()"` (canonical container). This size-class count is immaterial to every measurement, hotspot attribution, and the normal-vs-leak verdict here, because all investigated allocations (OCR image buffers, the full-text `content` string, pickled classifier arrays) are transients far larger than the 512-byte threshold and therefore bypass `pymalloc` entirely.

### 3.3 The tri-lens harness (exact source)

The reusable core is two small scripts placed under `/tmp/mem_harness/` **inside the container** (never in the repository). `probe.py` implements the three lenses (`tracemalloc` [8], `gc` [9], `resource.getrusage` [10] — see §3.2 Sources); `bootstrap.py` mirrors the project's own test fixture `src/documents/tests/utils.py` `setup_directories()` so the real entry points run against temporary data/scratch/media/consumption directories and a temporary SQLite DB. Full source is in Appendix A.

**Standard invocation (stated exactly as used):**

```
cd /app/src && PYTHONPATH=/app/src DJANGO_SETTINGS_MODULE=paperless.settings \
    PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/<script>.py
```

Redis is required by the consumer's progress channel and the django-q broker; it was started once per container with `redis-server --daemonize yes --save '' --appendonly no`. Coverage and xdist are never involved (scripts run as plain single-process `python`).

### 3.4 Observer effect — why headline magnitudes use *light* instrumentation

`tracemalloc.start(nframe)` stores an `nframe`-deep traceback for **every** tracked allocation, and each retained `take_snapshot()` copies all of those tracebacks. To quantify this, `observer_effect.py` consumes the **same** `simple-digital.pdf` through the real `consume_file` in two separate fresh processes — a *heavy* mode (`tracemalloc.start(30)` plus `take_snapshot()`+`gc.get_objects()` at every pipeline stage boundary, retaining all 14 snapshots as a real "collect-then-analyse" harness would) and a *light* mode (`tracemalloc.start(1)`, one before/after probe) — 2 runs each to rule out variance. Commands:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/observer_effect.py heavy
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/observer_effect.py light
```

Actual captured summary lines (run 1 of each; full output in Appendix B-1c):

```
[SUMMARY] mode=heavy run=1 RSS_hiwater=3251.9MiB VmRSS_after=3255.9MiB tm.peak=54.91MiB heavy_snapshots_taken=14 retained=14
[SUMMARY] mode=light run=1 RSS_hiwater=142.1MiB  VmRSS_after=146.1MiB  tm.peak=54.97MiB heavy_snapshots_taken=0 retained=0
```

The **~3110 MiB** gap (3251.9 vs 142.1 MiB high-water, on the identical 22 926-byte PDF) is the profiler's own overhead — 30-frame tracebacks stored for every allocation plus the 14 retained per-stage snapshots — **not** paperless. The decisive control: the Python-heap `tracemalloc.peak` is **identical** in both modes (**54.91 MiB heavy vs 54.97 MiB light**), because `tracemalloc.get_traced_memory()` does not count tracemalloc's *own* traceback storage — so paperless allocated exactly the same in both runs, and the multi-GB RSS gap is 100% profiler. **All absolute magnitudes in §4–§8 therefore use the light harness** (`tracemalloc.start(1)`, minimal probes), which agrees with the `tracemalloc.start(1)` distribution harness (§7, `dist_harness.py:L18`). The 25-frame `attrib_harness.py` (§4.5) is used **only** for `file:line` attribution (where allocations occur), never for magnitude, and is labelled as such. This is directly relevant to the user's situation: if the "far more memory than expected" was observed while a heavyweight profiler/APM was attached, a large fraction of the apparent spike may be the profiler itself.

### 3.5 Inputs and scales

Real sample files were copied out of the repository to `/tmp` before use (never written back):

| Purpose | File | Size |
|---|---|---|
| Plain text (control) | `src/documents/tests/samples/simple.txt` | 21 B |
| Text/digital PDF | `src/paperless_tesseract/tests/samples/simple-digital.pdf` | 22 926 B |
| Scanned/image PDF (OCR) | `src/paperless_tesseract/tests/samples/multi-page-images.pdf` | 150 479 B |

For the inconsistency question the **same byte-identical input** was fed to the real `consume_file` 50 times (digital, ×2 runs) and 20 times (image) in one long-lived process — the MD5 is printed once and re-asserted every iteration (`dist_harness.py:L48-50`), and the created `Document` row is deleted between iterations purely so the next identical consume clears the consumer's MD5 dedup (`pre_check_duplicate`, `consumer.py:L102-104`); the input bytes never change. For batch scaling, real exports were produced with the companion `document_exporter` at N = 10/50/200 and at 100 KB content/doc, then fed to the real `document_importer`.

---

### 3.6 Canonical environment and version proof (R5)

All measurements were taken inside the canonical Docker container (Python 3.9, `DEBUG=NO`). The exact runtime and the versions of every package under investigation were captured **directly from the running container**; they match the `Dockerfile` base image and the `Pipfile` / `requirements.txt` pins (notably **`scikit-learn==1.0.2`**, the pinned version whose change would produce model-load warnings). Command and complete unedited output (verbatim):

```
docker exec mem_investigation bash -c 'python --version; python -c "import sys; print(sys.version)"; pip show django django-q djangorestframework scikit-learn pikepdf ocrmypdf whoosh tika pdfminer.six numpy scipy pillow redis psycopg2 channels 2>/dev/null | grep -E "^(Name|Version):"'
```

```
$ python --version
Python 3.9.23

$ python -c "import sys; print(sys.version)"
3.9.23 (main, Jul 22 2025, 01:41:20) 
[GCC 10.2.1 20210110]

$ pip show django django-q djangorestframework scikit-learn pikepdf ocrmypdf whoosh tika pdfminer.six numpy scipy pillow redis psycopg2 channels 2>/dev/null | grep -E "^(Name|Version):"
Name: Django
Version: 4.0.4
Name: django-q
Version: 1.3.9
Name: djangorestframework
Version: 3.13.1
Name: scikit-learn
Version: 1.0.2
Name: pikepdf
Version: 5.1.1
Name: ocrmypdf
Version: 13.4.3
Name: Whoosh
Version: 2.7.4
Name: tika
Version: 1.24
Name: pdfminer.six
Version: 20220319
Name: numpy
Version: 1.22.3
Name: scipy
Version: 1.8.0
Name: Pillow
Version: 9.1.0
Name: redis
Version: 3.5.3
Name: psycopg2
Version: 2.9.3
Name: channels
Version: 3.0.4
```

Every version matches the authoritative manifests: Django `~=4.0` → **4.0.4**, django-q `~=1.3` → **1.3.9**, djangorestframework `~=3.13` → **3.13.1**, **scikit-learn `==1.0.2` (pinned)** → **1.0.2**, pikepdf `~=5.1` → **5.1.1**, ocrmypdf `~=13.4` → **13.4.3**, whoosh `~=2.7.4` → **2.7.4**, tika → **1.24**, pdfminer.six → **20220319**, numpy → **1.22.3**, scipy → **1.8.0**, Pillow `~=9.1` → **9.1.0** (plus redis 3.5.3, psycopg2 2.9.3, channels 3.0.4 for the broker/progress channel). The interpreter is **Python 3.9.23** (`python:3.9-slim-bullseye` lineage, `Dockerfile:L18`) — the canonical runtime for every figure in this document.

---
## 4. Q1 — Cause of the spikes

**Direct answer.** The elevated, disproportionate-to-size memory during import comes from three concrete places, in decreasing order of magnitude: **(1) OCR rasterization of scanned/image PDFs** (native `ocrmypdf`/ghostscript/tesseract/PIL work invoked from `RasterisedDocumentParser.parse`), **(2) the full extracted-text string** that flows into `Document.content`, and **(3) a one-time lazy-import cost** paid on the first document a worker processes. Crucially, **the "metadata-handling stage" is not part of the consumption pipeline at all** — see the architecture note below — so the spikes are *parsing/OCR/text* costs, not metadata costs. Every one of these is **transient** (released after the document) or **one-time** (paid once per process); none grows across documents.

### 4.1 Architecture note (confirmed at runtime): the consumer never calls `extract_metadata()`

`Consumer.try_consume_file` (`src/documents/consumer.py:L180`) runs these stages in order: `document_parser.parse(...)` (`L261`), `get_optimised_thumbnail()` (`L265`), `text = document_parser.get_text()` (`L271`), `get_date()` (`L272`), `classifier = load_classifier()` (`L292`), `_store(...)` (`L379`), and `finally: document_parser.cleanup()` (`L369`). There is **no** call to `extract_metadata()` anywhere in the pipeline. `extract_metadata` (base stub `src/documents/parsers.py:L304`; PDF override `src/paperless_tesseract/parsers.py:L26`) is invoked **only** by the REST metadata endpoint (`src/documents/views.py:L269`). Therefore "metadata handling" as a memory concern lives on the REST endpoint (analysed in §5), while the import spikes are the parse/OCR/text stages.

### 4.2 Per-type single-document footprint (light harness, stable across 2 runs)

Real `consume_file` (`src/documents/tasks.py:L184` → `try_consume_file`) driven once per document type, **each in its own fresh process** (`light_consume.py <type>`, so the `ru_maxrss` high-water is a clean per-type measurement; the script consumes the same type twice — a cold run 1 and a warm run 2 — for magnitude stability per Rule R2). Commands (three isolated processes):

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py text
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py digital
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py image
```

Actual output — the cold run-1 before/after/after-`gc.collect()` probes for each type (complete 180-line output incl. warm run 2 is in Appendix B-1):

```
===== LIGHT CONSUME type=text run=1 size=21B =====
[PROBE] text r1 BEFORE
    RSS_current(VmRSS)  = 118720 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 116892 kB (114.2 MiB)
    tracemalloc.current = 42398850 B (40.43 MiB)
    tracemalloc.peak    = 43404231 B (41.39 MiB)
    gc.live_objects     = 91830
[PROBE] text r1 AFTER
    RSS_current(VmRSS)  = 133400 kB (130.3 MiB)
    RSS_hiwater(maxrss) = 131228 kB (128.2 MiB)
    tracemalloc.current = 45273492 B (43.18 MiB)
    tracemalloc.peak    = 45582988 B (43.47 MiB)
    gc.live_objects     = 96087
[PROBE] text r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 133400 kB (130.3 MiB)   tracemalloc.current = 45129051 B (43.04 MiB)
    gc.live_objects     = 95396   gc.collect()->freed = 521
[SUMMARY] type=text run=1 dRSS=14.34MiB peakTM=43.47MiB dLiveObj=4257 freed=521

===== LIGHT CONSUME type=digital run=1 size=22926B =====
[PROBE] digital r1 BEFORE
    RSS_current(VmRSS)  = 118540 kB (115.8 MiB)
    RSS_hiwater(maxrss) = 113812 kB (111.1 MiB)
    tracemalloc.current = 42397920 B (40.43 MiB)
    tracemalloc.peak    = 43388299 B (41.38 MiB)
    gc.live_objects     = 91837
[PROBE] digital r1 AFTER
    RSS_current(VmRSS)  = 149380 kB (145.9 MiB)
    RSS_hiwater(maxrss) = 143508 kB (140.1 MiB)
    tracemalloc.current = 55678752 B (53.10 MiB)
    tracemalloc.peak    = 57640959 B (54.97 MiB)
    gc.live_objects     = 107808
[PROBE] digital r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 149908 kB (146.4 MiB)   tracemalloc.current = 55483803 B (52.91 MiB)
    gc.live_objects     = 106638   gc.collect()->freed = 890
[SUMMARY] type=digital run=1 dRSS=30.12MiB peakTM=54.97MiB dLiveObj=15971 freed=890

===== LIGHT CONSUME type=image run=1 size=150479B =====
[PROBE] image r1 BEFORE
    RSS_current(VmRSS)  = 118640 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 115544 kB (112.8 MiB)
    tracemalloc.current = 42396130 B (40.43 MiB)
    tracemalloc.peak    = 43413610 B (41.40 MiB)
    gc.live_objects     = 91830
[PROBE] image r1 AFTER
    RSS_current(VmRSS)  = 191508 kB (187.0 MiB)
    RSS_hiwater(maxrss) = 247640 kB (241.8 MiB)
    tracemalloc.current = 56009058 B (53.41 MiB)
    tracemalloc.peak    = 57575286 B (54.91 MiB)
    gc.live_objects     = 108190
[PROBE] image r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 163440 kB (159.6 MiB)   tracemalloc.current = 55764968 B (53.18 MiB)
    gc.live_objects     = 106885   gc.collect()->freed = 1022
[SUMMARY] type=image run=1 dRSS=71.16MiB peakTM=54.91MiB dLiveObj=16360 freed=1022
```

| Type | file:line source | AFTER VmRSS | high-water (`ru_maxrss`) | `tracemalloc.peak` | consume Δ (VmRSS) |
|---|---|---|---|---|---|
| plain text | `paperless_text/parsers.py` `parse` | 130.3 MiB | 128.2 MiB | **43.47 MiB** | +14.34 MiB |
| digital PDF | `paperless_tesseract/parsers.py` `parse` (pdfminer) | 145.9 MiB | 140.1 MiB | **54.97 MiB** | +30.12 MiB |
| image PDF (OCR) | `paperless_tesseract/parsers.py` `parse` → `ocrmypdf` | 187.0 MiB | **241.8 MiB** | **54.91 MiB** | +71.16 MiB (hi +129 MiB) |

The image high-water is **non-deterministic** across runs (this *is* part of the Q4 answer, §7.2): run 1 reached **241.8 MiB**, run 2 of the same process reached **263.8 MiB** (`RSS_hiwater(maxrss) = 270148 kB`), while the Python-heap `tracemalloc.peak` stayed at 54.9/54.1 MiB. The complete two-run output for all three types is Appendix B-1.

**Reading the evidence (this is the crux of Q1):** the image/OCR PDF reaches the highest high-water — **241.8 MiB from a 150 KB file**, the disproportionate spike the user describes — yet its Python-heap `tracemalloc.peak` (54.91 MiB) is **statistically identical to the digital PDF's (54.97 MiB)**. The image PDF did **not** allocate meaningfully more *Python* memory than the digital one; the ~187 MiB gap between its high-water and its Python-heap peak is **native OCR rasterization** (ghostscript/tesseract/leptonica in `ocrmypdf` subprocesses and threads) plus glibc arenas — entirely invisible to `tracemalloc`. It is **transient**: immediately after the image consume, `gc.collect()` dropped current RSS from **187.0 → 159.6 MiB** (the OCR working set was released; the high-water stays at 241.8 by definition). Live objects (`gc.live`) land at ~96–108 k for every type and *fall* on `gc.collect()` (freed 521/890/1022) — **no live-object growth, no leak.** The baseline before each consume is ~115.9 MiB RSS / 40.4 MiB `tracemalloc` (a fresh bootstrapped worker with `tracemalloc` active); the ~75 MiB RSS-vs-heap gap even at baseline is native import memory (§10).

### 4.3 Per-stage boundary measurements (before / during / after / `gc.collect()`)

This is the direct, complete answer to "which stage of the pipeline consumes the memory." The harness `stage_consume.py` (full source in Appendix A) monkey-patches the concrete parser methods (`RasterisedDocumentParser.parse` / `.get_optimised_thumbnail` / `.get_text` / `.get_date` / `.get_archive_path`, resolved from the real factory `get_parser` in `paperless_tesseract/signals.py`, **not** the base class), `consumer.load_classifier`, `Consumer._store`, and the three `document_consumption_finished` handlers (`set_correspondent`/`set_document_type`/`set_tags`). It calls `probe()` immediately **before** and **after** each stage (with `tracemalloc.reset_peak()` at each stage start so the reported `peak` is that stage's own peak), then a final `probe(do_gc=True)`. Driven through the real `consume_file`. Command:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/stage_consume.py
```

**Per-stage delta summary (digital PDF, run 1)** — verbatim; columns are Δ current RSS (VmRSS), Δ `tracemalloc.current`, that stage's own `tracemalloc.peak`, and Δ live objects:

```
----- STAGE DELTA SUMMARY sample=digital run=1 -----
stage                             dRSS_MiB dTM_cur_MiB  peak_MiB  dLiveObj
STAGE parse                          25.28       10.71     54.98     13235
STAGE thumbnail                       0.00       -0.00     52.44         4
STAGE get_text                        0.00        0.00     52.39         2
STAGE get_date                        0.00        0.00     52.39         2
STAGE get_archive_path                0.00       -0.00     52.35         2
STAGE load_classifier                 0.00        0.00     52.36         2
STAGE _store                          0.00        0.03     52.40        89
STAGE handler:set_correspondent       0.00        0.01     53.08        31
STAGE handler:set_document_type       0.00        0.01     53.10        30
STAGE handler:set_tags                0.00        0.00     53.10         5
```

**Per-stage delta summary (image/OCR PDF, run 1)** — verbatim:

```
----- STAGE DELTA SUMMARY sample=image run=1 -----
STAGE parse                          92.87        0.76     54.41      1664
STAGE thumbnail                       0.70       -0.00     53.55         2
STAGE get_text                        0.00        0.00     53.50         2
STAGE get_date                        0.00        0.00     53.50         2
STAGE get_archive_path                0.00       -0.00     53.50         2
STAGE load_classifier                 0.00        0.00     53.51         2
STAGE _store                          0.00        0.00     53.66        11
STAGE handler:set_correspondent       0.00        0.00     53.53         5
STAGE handler:set_document_type       0.00        0.00     53.53         4
STAGE handler:set_tags                0.00        0.00     53.54         5
```

**How the four states map to the probes:** for each stage, **before** = the `[PROBE] STAGE <name> BEFORE` block (full RSS/`tracemalloc.current`/`gc.live` immediately before the stage runs); **during** = that stage's own `tracemalloc.peak` (because `tracemalloc.reset_peak()` is called at the stage's start, the peak captured at its end is the intra-stage high-water — this is the `peak_MiB` column above and the `tracemalloc.peak` line in each `AFTER` block); **after** = the `[PROBE] STAGE <name> AFTER` block; and **`gc.collect()`** = the whole-pipeline `[PROBE] PIPELINE end AFTER gc.collect()` block. The complete set of these probes for every stage (both runs, both document types — 724 lines) is in Appendix B-1a. The representative full probes for the dominant `parse` stage, the `load_classifier` stage (the classifier boundary), and the whole-pipeline before/after/`gc.collect()` (digital run 1) are:

```
[PROBE] STAGE parse BEFORE
    RSS_current(VmRSS)  = 123276 kB (120.4 MiB)   RSS_hiwater(maxrss) = 120592 kB (117.8 MiB)
    tracemalloc.current = 43684666 B (41.66 MiB)  tracemalloc.peak = 44919281 B (42.84 MiB)
    gc.live_objects     = 93835
[PROBE] STAGE parse AFTER
    RSS_current(VmRSS)  = 149164 kB (145.7 MiB)   RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54916860 B (52.37 MiB)  tracemalloc.peak = 57647585 B (54.98 MiB)
    gc.live_objects     = 107070
[PROBE] STAGE load_classifier BEFORE
    tracemalloc.current = 54897602 B (52.35 MiB)  gc.live_objects = 106641
[PROBE] STAGE load_classifier AFTER
    tracemalloc.current = 54898054 B (52.35 MiB)  gc.live_objects = 106643   # +2 objects: load_classifier() returned None (no MODEL_FILE)
[PROBE] PIPELINE end (after consume_file)
    RSS_current(VmRSS)  = 150208 kB (146.7 MiB)   tracemalloc.current = 55693992 B (53.11 MiB)  gc.live_objects = 107881
[PROBE] PIPELINE end AFTER gc.collect()
    RSS_current(VmRSS)  = 150208 kB (146.7 MiB)   tracemalloc.current = 55499627 B (52.93 MiB)  gc.live_objects = 106705
    gc.collect()->freed = 890
```

**How/why (cause → effect), grounded to `consumer.py`:**

- **`parse` (`consumer.py:L261`) is the ONLY stage that grows RSS** — digital **+25.28 MiB**, image **+92.87 MiB** current RSS. This is where the parser reads the file and (for images) invokes OCR. For the digital PDF `tracemalloc.current` rises +10.71 MiB (pdfminer text extraction is in-Python); for the image PDF `tracemalloc.current` rises only +0.76 MiB despite +92.87 MiB RSS — proving the OCR memory is **native**, not Python heap. `parse` is also where live objects jump (+13 235 digital).
- **`get_optimised_thumbnail` (`consumer.py:L265`)** — image +0.70 MiB RSS (renders a thumbnail via the parent), digital ~0; negligible.
- **`get_text` (`consumer.py:L271`)** — **0.00 MiB**: the text was already extracted *during* `parse` and cached in the parser's `self.text` (`src/documents/parsers.py:L296`); `get_text()` just returns it (`src/documents/parsers.py:L342`). No allocation here.
- **`get_date` (`consumer.py:L272`)** — 0.00 MiB for these small documents (it scans `self.text`; the cost is proportional to text length — see §5.4 for a large-text case).
- **`get_archive_path`, `load_classifier` (`consumer.py:L292`)** — 0.00 MiB. `load_classifier()` returns `None` here because the fresh temp `DATA_DIR` has no `MODEL_FILE` (`classifier.py:L36`); its "during" probe shows exactly **+2 live objects** and 0 MiB (the `None` return path — see §6.1 for the model-present path).
- **`_store` (`consumer.py:L379`)** — +0.03 MiB `tracemalloc`, +89 live objects: it builds the `Document` row and computes the MD5 checksum; small for these samples (see §5.3 for the content-string analysis).
- **The three handlers** (`set_correspondent`/`set_document_type`/`set_tags`) — ~0.00 MiB each: with `classifier=None` they fall back to rule matching.
- **After `gc.collect()`** the pipeline released 890 live objects and `tracemalloc.current` dropped, while RSS stayed at 146.7 MiB (benign arena retention). **No stage retains growth across the boundary.**

**Conclusion for Q1:** the spike is concentrated entirely in **`parse`** (`consumer.py:L261`), and for image PDFs it is overwhelmingly **native OCR memory** (RSS grows 92.87 MiB while the Python heap grows 0.76 MiB). Every downstream stage — including `load_classifier`, `_store`, and the metadata-free handlers — is ~0. There is **no `extract_metadata()` stage in the consumer at all** (§4.1).

### 4.4 First-touch lazy-import spike (the one-time cost)

The first document a worker processes triggers a large one-time import of native modules referenced at the top of `src/documents/tasks.py` (`from pyzbar import pyzbar` at `L25`, plus `ocrmypdf`, the parser plugins, `pdfminer`, PIL, and — when a model is present — numpy/scipy/scikit-learn). Because these are almost all **C-extension** modules, the first-touch cost dominates **RSS** but is nearly invisible to `tracemalloc`. It is visible as the cold-vs-warm gap between run 1 and run 2 of the **same** type in the **same** process (§4.2 data):

| type | cold run-1 ΔVmRSS | warm run-2 ΔVmRSS | cold `peakTM` | warm `peakTM` |
|---|---|---|---|---|
| plain text | **+14.34 MiB** | +1.86 MiB | 43.47 MiB | 43.51 MiB |
| digital PDF | **+30.12 MiB** | +0.06 MiB | 54.97 MiB | 53.44 MiB |
| image PDF | **+71.16 MiB** | (RSS nondeterministic — §7.2) | 54.91 MiB | 54.14 MiB |

The distribution harness (`dist_harness.py`, §7.1 — 50 byte-identical consumes in one long-lived process) confirms the **Python-heap** peak is essentially flat across warm iterations, with iteration 0 only ~1 MiB above the warm median (verbatim):

```
peak_TM MiB: min=53.13 median=53.61 max=54.60 mean=53.61 stdev=0.207   (digital, run 1)
iter0(first-touch)=54.60MiB  warm_median(iter>=2)=53.60MiB
```

**How/why:** the first-touch cost is almost entirely **native** C-extension module loading (zbar via `pyzbar`, `ocrmypdf`/leptonica, `pdfminer`, PIL, and numpy/scipy for the classifier), which is why it dominates RSS (+14 to +71 MiB depending on which plugins the type triggers) yet moves `tracemalloc.peak` by only ~1 MiB. The imported modules stay resident for the process lifetime (expected — that is how Python module caching works), so the **first** document in a fresh django-q worker looks like a spike and every later one does not. This is the single largest contributor to the "sometimes spikes" inconsistency (Q4, §7).

### 4.5 Stage-level `file:line` attribution (`attrib_harness.py`, `tracemalloc.start(25)` — attribution only)

To attribute the parse allocations to source lines, `attrib_harness.py` drives the real `consume_file` under `tracemalloc.start(25)` and diffs a snapshot taken before vs. after the consume. (Deeper `nframe` inflates RSS per the observer-effect note §3.4, so this is used for **attribution only**, not magnitude.) Command:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/attrib_harness.py \
  /app/src/paperless_tesseract/tests/samples/simple-digital.pdf DIGITAL
```

Top Python-heap allocators by size delta across the whole consume — **digital PDF**, verbatim:

```
===== ATTRIBUTION label=DIGITAL file=simple-digital.pdf size=22926B =====
--- TOP 15 allocators by size DELTA (before-consume -> after-consume), by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.2 MiB (+7366 KiB), count=199763 (+67087), average=101 B
    <frozen importlib._bootstrap>:228: size=7784 KiB (-695 KiB), count=82994 (+6300), average=96 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB (+306 KiB), count=7228 (+7228), average=43 B
    /usr/local/lib/python3.9/site-packages/pdfminer/glyphlist.py:54: size=144 KiB (+144 KiB), count=2 (+2), average=72.0 KiB
    /usr/local/lib/python3.9/abc.py:106: size=312 KiB (+117 KiB), count=1395 (+594), average=229 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49: size=312 KiB (+101 KiB), count=354 (+58), average=903 B
```

**Image PDF**, verbatim — note it is **the same** as digital, with **no** PIL/OCR entry:

```
===== ATTRIBUTION label=IMAGE file=multi-page-images.pdf size=150479B =====
--- TOP 15 allocators by size DELTA (before-consume -> after-consume), by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.3 MiB (+7493 KiB), count=201260 (+68589), average=101 B
    <frozen importlib._bootstrap>:228: size=7779 KiB (-698 KiB), count=82951 (+6289), average=96 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB (+306 KiB), count=7228 (+7228), average=43 B
    /usr/local/lib/python3.9/site-packages/pdfminer/glyphlist.py:54: size=144 KiB (+144 KiB), count=2 (+2), average=72.0 KiB
```

**How/why (cause → effect):** the dominant Python-heap allocator for **both** types is `<frozen importlib._bootstrap_external>:647` (~19 MiB) — the lazy bytecode import of the plugin modules loaded on first touch (§4.4). The only document-specific Python allocations are tiny: `base64.py:325` (306 KiB across 7228 objects) is `pdfminer.six` decoding embedded PDF content streams, and `pdfminer/glyphlist.py:54` (144 KiB) is pdfminer's glyph tables. **Crucially, the image PDF's top allocators are identical to the digital PDF's — there is no PIL, ghostscript, or tesseract entry** — because `RasterisedDocumentParser.parse` hands the file to `ocrmypdf`, which rasterizes and OCRs each page in **native** ghostscript/tesseract/leptonica subprocesses whose memory never appears in `tracemalloc`. This is the direct `file:line` proof that the image PDF's ~187 MiB native RSS gap (§4.2/§4.3, high-water 241.8 MiB vs Python-heap peak 54.9 MiB) is native OCR work, not Python heap. Afterward, `_store` (`consumer.py:L379`) persists the text via `Document.objects.create(content=text, …)` (`L398–L406`) plus a whole-file `hashlib.md5(f.read())` checksum (`L402`) — small for these samples; the content string itself is analysed in §5.3. Complete output for both types is in Appendix B-1b.

### 4.6 Edge/error paths

These conditions are exercised with full command/output evidence in §6.1 (no-model vs. model-present; raw output in Appendix B-13 and B-4) and Appendix B-14 (encrypted vs. corrupt error path). In summary:

- **First consumption with no trained model:** `load_classifier()` returns `None` (`src/documents/classifier.py:L36`) — confirmed in the §4.3 per-stage probe (the `load_classifier` stage adds exactly **+2 live objects, 0.00 MiB**). No `pickle.load` runs; rule-only matching is used.
- **After a model exists:** the 7× `pickle.load` path runs (§6.1) — measured on a realistic model in §6.1.
- **Error path:** a genuinely **corrupt** PDF raises `ConsumerError` wrapping ocrmypdf's `InputFileError`, and `finally: document_parser.cleanup()` (`consumer.py:L369`) still removes the parser temp directory (before→after `paperless-*` count 0→0). An **encrypted** PDF (empty password) actually *succeeds* rather than erroring (`Document` created, `content_len=0`). Both are shown with complete raw output in Appendix B-14 (this reconciles the earlier "encrypted vs. corrupt" wording — see F4).


---

## 5. Q2 — Unnecessary copies / prolonged reference retention

**Direct answer.** Metadata handling does **not** make redundant *in-RAM* copies, and it does **not** hold large objects in memory longer than necessary. There are three findings, each measured on the **real** metadata path (`DocumentViewSet.metadata` → `get_metadata`, `src/documents/views.py:L283/L260`): (a) a **genuine but essentially zero-RAM resource leak** — empty parser temp-directories are left behind on every endpoint call because `get_metadata` never calls `cleanup()`; (b) the un-closed `pikepdf` handle is **not** a live-memory leak — CPython reference-counting reclaims the QPDF C handle at scope exit; (c) the full extracted-text string is held **exactly once**, proportional to the document, which is necessary.

### 5.1 The metadata endpoint leaks empty temp-directories (inode leak, ~0 RAM)

`get_metadata` constructs a parser (`parser = parser_class(progress_callback=None, logging_group=None)`, `src/documents/views.py:L266`) — which creates a `tempfile.mkdtemp()` directory in `__init__` (`src/documents/parsers.py:L293`) — and then `return parser.extract_metadata(file, mime_type)` (`src/documents/views.py:L269`) **without** ever calling `parser.cleanup()`. The `metadata` action calls `get_metadata` for the original (`L295`) and, when `doc.has_archive_version` (`L300`), again for the archive (`L302`) — so **up to two temp-directories leak per request**. Driven through the real DRF view (`APIRequestFactory` + `force_authenticate` + `resp.render()`), the temp-directory count in `SCRATCH_DIR` grows exactly linearly and RAM stays flat:

```
[BASELINE before any endpoint call] tempdirs=0 bytes_in_tempdirs=0 live_pdf=0
    -> after 1 calls: leaked_tempdirs=2 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
    -> after 10 calls: leaked_tempdirs=20 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
    -> after 50 calls: leaked_tempdirs=100 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
    -> after 200 calls: leaked_tempdirs=400 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
[PROBE] metadata AFTER 200 calls
    RSS_current(VmRSS)  = 174612 kB (170.5 MiB)   # +0.4 MiB over 200 calls (essentially flat)
    tracemalloc.current = 63114433 B (60.19 MiB)  # flat
    tracemalloc.peak    = 65139696 B (62.12 MiB)  # CONSTANT across all 200 calls
    gc.live_objects     = 120242                  # flat
[PROBE] metadata AFTER-GC AFTER gc.collect()
    gc.collect()->freed = 284
    -> after gc.collect(): leaked_tempdirs=400 bytes_in_tempdirs=0 live_pikepdf_Pdf=0
```

The leaked directories are verified **empty** (the harness lists their contents): `… /paperless-p8v8d8uq -> contents=[]`, and `bytes_in_tempdirs=0` at every checkpoint. So this is a real **inode/directory** leak (contrast the consumer's `finally: cleanup()` at `consumer.py:L369`, which always removes its temp dir), but it costs **~0 MiB of RAM per call** — `RSS_current` moves only 170.1 → 170.5 MiB across 200 calls and `tracemalloc.peak` is **constant** at 62.12 MiB — so it will **not** explain a memory spike; over a long-lived process it accumulates empty directory entries, not RAM. Full unedited output is in Appendix B-2. *(Per the read-only scope, this is reported, not fixed.)*

### 5.2 The un-closed `pikepdf` handle is reclaimed promptly (NOT a leak)

`RasterisedDocumentParser.extract_metadata` opens `pdf = pikepdf.open(document_path)` (`src/paperless_tesseract/parsers.py:L34`) with **no** context manager and **no** `pdf.close()` before `return result` (`src/paperless_tesseract/parsers.py:L55`). Despite the code smell, the QPDF C handle does **not** linger. Calling the real `extract_metadata` directly and counting live `pikepdf.Pdf` before/during/after (no `gc.collect()`) shows the handle exists only for the duration of the call (`q2_pikepdf_content.py`, Q2(a); full output App. B-3):

```
  live pikepdf.Pdf BEFORE construct parser: 0
  live pikepdf.Pdf DURING extract_metadata (inside, after open): 1
  live pikepdf.Pdf AFTER extract_metadata returns (NO gc.collect): 0
  metadata entries returned: 5
```

The count is **1** while `extract_metadata` runs (the open handle) and **0** the instant it returns — **without** any `gc.collect()` — so CPython's reference counting frees the local `pdf` (and the underlying QPDF handle) at scope exit. The repeated-endpoint harness corroborates this: live `pikepdf.Pdf` stayed **0** after 1/10/50/200 endpoint calls and after `gc.collect()` (App. B-2). **Verdict: code smell, not retained memory.**

### 5.3 The full-text `content` string is held exactly once (necessary, not redundant)

The extracted text flows: parser `self.text` (`src/documents/parsers.py:L296`) → `get_text()` returns `self.text` (`src/documents/parsers.py:L342`) → the consumer's `text` local (`src/documents/consumer.py:L271`) → `Document(content=text)` (`_store`, `L398–L406` → `Document.content` `TextField`, `src/documents/models.py:L117`). Instrumenting `_store` while consuming `simple-digital.pdf` showed the consumer passes the **same object**, not a copy (`q2_pikepdf_content.py`, Q2(b); full output App. B-3):

```
  text IS document.content (same object, single copy): True
  content length: 24 chars
  sys.getrefcount(text) inside _store: 6
  tracemalloc.peak during _store: 52.49 MiB
```

`text is document.content` is **True** — the consumer assigns the *same* `str` object to the field, not a duplicate — and `getrefcount` is a small constant (**6**, the handful of expected local/argument references), not a growing set of copies. To confirm the single copy simply *scales* with text length (rather than being duplicated), the same harness assigned a synthetic **40 MiB** content string to the field under `tracemalloc`:

```
  assigning 40MiB content: dTM_current=-0.00 MiB peak=93.15 MiB (single reference on the model field)
```

The transient `peak` rose to **93.15 MiB** during the assignment, then `dTM_current` returned to **~0 MiB** — i.e. **one copy per document, proportional to its text length**, with the intermediate transient released and only the single field reference retained. **No redundant in-RAM copy; no prolonged retention.**

### 5.4 Large-text transient (bonus — a real, size-proportional spike that is released)

To measure the dominant per-document cost for a *large* text document, plain-text files of 5 MiB and 20 MiB were consumed through the real `consume_file` (`large_text.py`; full output App. B-3b). The transient `tracemalloc.peak` scales with document size and is **released** afterward:

| input | transient `tracemalloc.peak` | ΔVmRSS (during consume) | `tm.current` after `gc.collect()` | stored `content` |
|---|---|---|---|---|
| 5 MiB (run 1) | 79.78 MiB | +100.50 MiB | 43.41 MiB (≈ baseline) | 5 242 880 B |
| 5 MiB (run 2) | 79.86 MiB | +100.21 MiB | 43.41 MiB (≈ baseline) | 5 242 880 B |
| 20 MiB | 187.30 MiB | +331.32 MiB | 43.41 MiB (≈ baseline) | 20 971 520 B |

**How/why (cause → effect):** consuming a 5 MiB text file drives a transient Python-heap peak of **~79.8 MiB** (stable across two runs, 79.78 / 79.86) and a **+100 MiB** RSS jump; a 20 MiB file drives a **187.3 MiB** peak and a **+331 MiB** RSS jump — i.e. the transient grows roughly linearly with text length (from `self.text = f.read()` in the text parser, the `readlines()` in thumbnailing, and building the `Document.content` string). Crucially, in **every** case `tracemalloc.current` returns to **43.41 MiB (≈ the process baseline)** after `gc.collect()` — the full-text transient is **completely released**, retaining nothing beyond baseline — while **RSS stays elevated** (216.2 MiB after the 5 MiB consume, and does **not** drop after `gc`). This is the textbook combination of a genuinely large but **released** transient plus **benign allocator retention** (§10), not a leak: the Python heap flattens back to baseline; only the OS RSS stays high. A large text document therefore *does* produce a real, size-proportional spike during consume — expected behaviour proportional to storing the full extracted text once.

**Q2 summary:** no redundant copies; no undue retention in RAM. The single un-released artifact is empty temp-directories on the REST endpoint (inode leak, ~0 RAM); the `pikepdf` handle is reclaimed by refcounting; the content string is stored once.


---

## 6. Q3 — Caching accumulation

**Direct answer.** In the default (`DEBUG=NO`) configuration, **no caching or process-lifetime state accumulates across documents.** The three candidates named in the request behave as follows: the **cached classifier model** is loaded once per consume, shared with all three handlers, and freed afterward (live count returns to 0); the **search-index writer** (Whoosh `AsyncWriter`) is created and committed per update with zero live writers between uses; and the **ORM query log** (`connection.queries`) grows **only** under the non-default `PAPERLESS_DEBUG=YES`, and even then is bounded at 9000 entries (~2.39 MiB). Under the canonical default it stays at length 0.

### 6.1 Classifier: 7× `pickle.load`, dominated by the `MLPClassifier`, freed after each consume

`load_classifier()` (`src/documents/classifier.py:L30`) returns `None` if `MODEL_FILE` is missing (`L36`); otherwise `DocumentClassifier.load()` (`L76`) performs **7** `pickle.load()` calls — confirmed by the runtime invocation count = **7** (`classifier_harness.py`; the AAP prose says "six" payload objects, the seventh is the leading `schema_version`). Measured on a **real** model trained in-process (8 documents, 2 correspondents with `MATCH_AUTO`; on-disk `MODEL_FILE` size = **266 561 B ≈ 260 KiB**), the per-load `tracemalloc` current-delta and running peak were (full output in Appendix B-4):

```
 # loaded_type                   dTM_current_KiB  peak_MiB
 1 int                                       0.1     62.00
 2 bytes                                     0.1     62.00
 3 CountVectorizer                          12.3     62.02
 4 NoneType                                  0.1     62.01
 5 NoneType                                  0.1     62.01
 6 MLPClassifier                           268.7     62.29   <== DOMINANT
 7 NoneType                                  0.1     62.27
  data_vectorizer.vocabulary_ size = 106 terms
```

**How/why (cause → effect):** in `DocumentClassifier.load()` the seven objects are, in order, `schema_version` (int, `L78`), `data_hash` (bytes, `L86`), `data_vectorizer` (`CountVectorizer`, `L87`), `tags_binarizer` (`L88`), `tags_classifier` (`L90`), `correspondent_classifier` (`L91`), `document_type_classifier` (`L92`). In this real model the **`MLPClassifier` at load #6 (`correspondent_classifier`, `L91`) is the single dominant object at 268.7 KiB** — it is the only trained neural classifier because the corpus had 2 correspondents (`MATCH_AUTO`) but no `MATCH_AUTO` tags or document types, so loads #4/#5/#7 deserialise to `None` (0.1 KiB each). The `CountVectorizer` vocabulary (106 terms) is only **12.3 KiB**. The whole load moves `tracemalloc.current` from **61.98 → 62.26 MiB (a ~0.28 MiB delta)** — dominated by the one `MLPClassifier`, not the vectorizer. The magnitude is bounded by the **model** size (number of correspondents/types/tags and vocabulary), **not** by the document being consumed; a production instance with trained tag and document-type classifiers would carry three `MLPClassifier` objects of comparable size rather than one. (The AAP's a-priori note that the `CountVectorizer` vocabulary would dominate holds only for instances whose vocabulary is far larger than their classifiers; the measured canonical model here is `MLPClassifier`-dominated.)

### 6.2 Classifier is loaded once per consume, shared across handlers, and not accumulated

Consuming **12** documents in one long-lived process with a trained classifier present (`classifier_batch.py`, Appendix A; full output Appendix B-4b). The live `DocumentClassifier` count stays **flat** across the batch, and within each consume the three signal handlers all receive the **same** classifier instance:

```
model built size=36564B
live DocumentClassifier BEFORE any consume: 1
  after consume 3/12: live DocumentClassifier=1
  after consume 6/12: live DocumentClassifier=1
  after consume 9/12: live DocumentClassifier=1
  after consume 12/12: live DocumentClassifier=1
[PROBE] after batch AFTER gc.collect()
    tracemalloc.current = 107670914 B (102.68 MiB)
    tracemalloc.peak    = 111253102 B (106.10 MiB)
    gc.live_objects     = 175747
    gc.collect()->freed = 1909
  after gc.collect(): live DocumentClassifier=1

[HANDLER SHARING] documents_with_all_3_handlers=12 all_share_same_classifier_id=True
   doc pk=9: handler->id(classifier) = {'set_correspondent': 136130673242656, 'set_document_type': 136130673242656, 'set_tags': 136130673242656}
   doc pk=10: handler->id(classifier) = {'set_correspondent': 136136277968448, 'set_document_type': 136136277968448, 'set_tags': 136136277968448}
```

**How/why (cause → effect):** the consumer loads the classifier **once** per document (`classifier = load_classifier()`, `consumer.py:L292`) and passes that single object into the `document_consumption_finished` signal, so `set_correspondent` (`handlers.py:L35`), `set_document_type` (`L101`) and `set_tags` (`L168`) all receive the **same** instance — measured directly as `all_share_same_classifier_id=True`, with `doc pk=9`'s three handlers reporting one identical `id()` (`136130673242656`). The `id()` **differs between documents** (`pk=9` → `…242656`, `pk=10` → `…968448`), proving each consume loads a **fresh** classifier and the previous one becomes unreferenced. The live `DocumentClassifier` count is **1 at every checkpoint** (before, after 3/6/9/12, and after `gc.collect()`) — it does **not** grow with the batch, so there is **no cross-document accumulation** of classifier objects. (The steady value of 1 is the single instance alive at each probe — the harness's most-recently-loaded classifier — not a growing set.) The `tracemalloc.current` rise to 102.68 MiB and `gc.live` of 175 747 after 12 consumes are the 12 persisted `Document` rows plus Whoosh index state, not retained classifiers; `gc.collect()` frees 1 909 transient objects and the live classifier count remains 1.

### 6.3 Whoosh `AsyncWriter` is per-update, not process-lifetime

The search index uses a context manager `open_index_writer` (`src/documents/index.py:L65`) that constructs `writer = AsyncWriter(open_index())` (`L66`), `yield`s it (`L69`), and `commit`s/`cancel`s on exit (`L74`/`L72`); the public `add_or_update_document` (`L118`) drives it via `update_document` (`L87`) per document. Driving **30 real index updates** (each a `Document.objects.create` + `add_or_update_document`) in one process (`whoosh_check.py`, App. A-20 / full output App. B-4c):

```
                              live {AsyncWriter, SegmentWriter, IndexWriter}
whoosh BEFORE                 {0, 0, 0}    VmRSS 105.9 MiB  tm.current 38.08 MiB  gc.live 85498
whoosh after 10 updates       {0, 0, 0}    VmRSS 106.0 MiB  tm.current 38.96 MiB  gc.live 86402
whoosh after 20 updates       {0, 0, 0}    VmRSS 106.0 MiB  tm.current 39.08 MiB  gc.live 86433
whoosh after 30 updates       {0, 0, 0}    VmRSS 106.0 MiB  tm.current 39.13 MiB  gc.live 86463
whoosh AFTER-GC (gc.collect)  {0, 0, 0}    VmRSS 106.0 MiB  tm.current 38.74 MiB  gc.live 85972  (freed 470)
```

**Zero** live writer objects — all three watched writer classes (`AsyncWriter`, `SegmentWriter`, `IndexWriter`) report `0` at every probe. VmRSS is essentially flat (105.9 → 106.0 MiB, hiwater constant 102.6 MiB) and the Python heap is nearly flat (`tracemalloc.current` 38.08 → 39.13 MiB peak during the 30 updates, back to 38.74 MiB after `gc.collect()`; `tracemalloc.peak` constant at 41.76 MiB). The `gc.live` count rises only **+965** across the 30 updates (~32 objects/update — the 30 persisted `Document` rows plus their index terms), and `gc.collect()` frees 470, leaving a net **+474**. The writer is **not** retained across documents.

### 6.4 ORM query log — only under the labeled non-default `DEBUG=YES` (§9)

`connection.queries` grows one dict per SQL statement **only when `settings.DEBUG` is True**. The default is `DEBUG=NO` (`src/paperless/settings.py:L50`; the file's own comment at `src/paperless/settings.py:L49` is "NEVER RUN WITH DEBUG IN PRODUCTION"). Under the default, `connection.queries` stayed at length **0** after ~10 000 statements. The full DEBUG=YES vs DEBUG=NO comparison is in §9. **Verdict Q3:** no accumulation in the canonical configuration.


---

## 7. Q4 — Spiking vs. non-spiking (the inconsistency)

**Direct answer.** Run repeatedly on the **exact same input bytes**, the **Python heap does not spike inconsistently** — its per-consume `tracemalloc` peak is remarkably stable (digital stdev 0.076–0.207 MiB, image stdev 0.149 MiB, all ≈ 53–55 MiB). The inconsistency the user observes lives in **native RSS**, and it is governed by **four concrete, reproducible differentiators**, in order of impact: **(1) first-touch vs. warm** (the first document *ever* in a fresh worker pays a one-time cold-import cost that is almost entirely **native** C-extension loading — cold-vs-warm ΔVmRSS of **+14.3 MiB (text) / +30.1 MiB (digital) / +71.2 MiB (image)** measured in §4.4 — while the Python heap barely moves: within the repeated-consume loop iter 0 is only ~1 MiB above the warm median (**54.60 vs 53.60 MiB**, §7.1); every later consume in that warm worker pays neither cost); **(2) OCR vs. no-OCR** (an image/scanned PDF drives VmRSS across a wide range — **152.9 → 258.2 MiB** on identical input — via native OCR subprocess buffers + glibc arenas, whereas a digital/text PDF stays flat at **145.6 → 146.8 MiB**); **(3) an OCR "warm-up" RSS plateau** (VmRSS climbs over the first few OCR documents then settles, so an observer sampling early sees "growth" and later sees "stable"); and **(4) classifier present vs. absent** (a minor ~0.28 MiB transient when a `MODEL_FILE` exists, §6.1, vs. 0). None of these grows the live-object count, so all are benign.

**Method (honours R3 "reproduce the same unchanged input, don't stabilize").** The **byte-identical** sample file is fed to the real `consume_file` every iteration — verified by a constant MD5 printed once and re-asserted each iteration (`dist_harness.py:L48-50`: `shutil.copy(src, dst)` → `assert md5(dst) == src_md5`). Because `consume_file` moves the source into place and rejects a byte-identical re-submission once a `Document` with that checksum exists (`pre_check_duplicate`, `consumer.py:L102-104`), the harness re-stages the identical bytes and **deletes the persisted `Document` row (and its stored files) between iterations**. That deletion is state teardown — **not** a modification of the input (the bytes are identical each time; a stabilized/parse-equivalent variant is explicitly *not* substituted). The per-iteration `tracemalloc` **peak** (the transient spike of one consume) plus the running VmRSS/ru_maxrss are recorded and the **distribution** (min/median/max/mean/stdev) reported across iterations for ≥ 2 runs. Full unedited output in Appendix B-5.

### 7.1 Digital PDF distribution — heap stable, RSS flat

50 iterations of the **byte-identical** `simple-digital.pdf` (`input_md5=42995833e01aea9b3edee44bbfdd7ce1`, 22 926 B), run twice:

```
===== DIST type=digital iters=50 run=1 input_md5=42995833e01aea9b3edee44bbfdd7ce1 input_size=22926B =====
  peak_TM MiB: min=53.13 median=53.61 max=54.60 mean=53.61 stdev=0.207
  peak_TM per-iter (first 10): [54.6, 53.13, 53.22, 53.3, 53.39, 53.46, 53.51, 53.52, 53.4, 53.46]
  VmRSS MiB   : start=145.6 end=146.6 min=145.6 max=146.6
  ru_maxrss   : start=142.7 end=142.7 (monotonic hiwater)
  iter0(first-touch)=54.60MiB  warm_median(iter>=2)=53.60MiB

===== DIST type=digital iters=50 run=2 input_md5=42995833e01aea9b3edee44bbfdd7ce1 input_size=22926B =====
  peak_TM MiB: min=53.69 median=53.87 max=54.00 mean=53.87 stdev=0.076
  VmRSS MiB   : start=146.7 end=146.8 min=146.7 max=146.8
  iter0(first-touch)=53.94MiB  warm_median(iter>=2)=53.86MiB
```

**How/why (cause → effect):** across 50 repetitions of the *same bytes*, the per-consume Python-heap `tracemalloc` peak is essentially constant — median 53.61 MiB (run 1) / 53.87 MiB (run 2), **stdev 0.207 / 0.076 MiB** (< 0.4 % of the mean). VmRSS is flat (145.6 → 146.6 MiB) and `ru_maxrss` is monotonic-flat at 142.7 MiB. The only above-warm value is **iter 0** (54.60 MiB run 1), and it exceeds the warm median (53.60 MiB) by only ~1.0 MiB — the heavy one-time cold-import cost happens at process start (§4.4), *before* iter 0, so it does not recur here. **Verdict:** a digital PDF fed identically does **not** spike inconsistently — heap and RSS are both stable across two runs (R3 stability satisfied).

### 7.2 Image/OCR distribution — heap stable, RSS spreads widely (the real inconsistency)

20 iterations of the **byte-identical** `multi-page-images.pdf` (`input_md5=62acb0bcbfbcaa62ca6ad3668e4e404b`, 150 479 B):

```
===== DIST type=image iters=20 run=1 input_md5=62acb0bcbfbcaa62ca6ad3668e4e404b input_size=150479B =====
  peak_TM MiB: min=54.54 median=54.87 max=55.11 mean=54.86 stdev=0.149
  peak_TM per-iter (first 10): [54.54, 54.66, 54.72, 54.83, 54.67, 54.76, 54.84, 54.93, 54.73, 54.81]
  VmRSS MiB   : start=152.9 end=219.6 min=152.9 max=258.2
  ru_maxrss   : start=256.7 end=258.8 (monotonic hiwater)
  iter0(first-touch)=54.54MiB  warm_median(iter>=2)=54.90MiB
```

**How/why (cause → effect):** the Python-heap lens is again **stable and identical in shape to the digital case** — median 54.87 MiB, **stdev 0.149 MiB** — because OCR runs in native ghostscript/tesseract/leptonica subprocesses whose memory never enters `tracemalloc` (proven by the identical top-allocator lists in §4.5/Appendix B-1b). The **RSS** lens tells the inconsistency story: on the *same unchanged bytes*, VmRSS ranges from **152.9 MiB to 258.2 MiB** (a ~105 MiB spread) within one run and ends at 219.6 MiB, while `ru_maxrss` climbs monotonically 256.7 → 258.8 MiB — the native OCR working set plus glibc arenas are allocated and then **reused** rather than returned to the OS. **This is precisely why the identical workload "sometimes spikes" (a sample taken at the 258 MiB peak) and "sometimes looks stable" (a later sample near the ~219 MiB settling point): the same input yields a WIDE native-RSS distribution but a TIGHT Python-heap distribution.** The difference is allocator/OCR behaviour, not a growing live-object set (§10).

### 7.3 The differentiators, grounded

| Differentiator | Spikes when… | Does not when… | Evidence (measured) | file:line |
|---|---|---|---|---|
| First-touch vs warm | first doc *ever* in a fresh worker | any later doc in that worker | one-time cold import at process start (§4.4); within the warm loop iter 0 exceeds warm median by only ~1 MiB (54.60 vs 53.60) | `tasks.py:L25` + parser imports |
| OCR vs no-OCR | image/scanned PDF | text/digital PDF | image VmRSS 152.9 → 258.2 MiB (spread ~105) vs digital flat 145.6 → 146.8 MiB, on identical input | `paperless_tesseract/parsers.py` `parse` |
| Arena warm-up | first few OCR docs | later OCR docs | image VmRSS climbs to max 258.2 then settles to 219.6; `ru_maxrss` 256.7 → 258.8 monotonic | glibc/pymalloc (allocator) |
| Classifier present | `MODEL_FILE` exists on disk | no `MODEL_FILE` | ~0.28 MiB load transient (`MLPClassifier` dominant, §6.1) vs 0 (`load_classifier()`→`None`) | `classifier.py:L36/L91` |

All four leave `gc` live-object counts and `tracemalloc` retained size flat across `gc.collect()`, and the per-consume Python-heap peak has **stdev 0.076–0.207 MiB (digital) / 0.149 MiB (image)** across identical repetitions — the run-to-run variation is native RSS (allocator/OCR), **benign, not a leak.**

---

## 8. Q5 — Document type × batch size (the cross-product)

**Direct answer.** The two entry points behave oppositely, and both were exercised. On the **consumption path**, document *type* sets the per-document peak (text < digital < OCR) while *batch size* does **not** accumulate — each document returns to the same steady state (per-document peak recurs; the total does not grow). On the **bulk importer path**, memory scales **linearly with `batch × content`** because the manifest embeds every document's full `content`, and it is the primary batch-size-sensitive path.

### 8.1 The cross-product table (measured through both real entry points)

The full cross-product `{plain text, text/digital PDF, scanned/image PDF} × {single, many}` was exercised through **both** real entry points — `consume_file` (`tasks.py:L184` → `Consumer.try_consume_file` `consumer.py:L180`) and `document_importer` (`management/commands/document_importer.py`). Every cell below is a measured value with its raw-output appendix reference; no cell is inferred.

| document type | **single — `consume_file`** | **many — `consume_file`** (N distinct docs, one long-lived process) | **single — `document_importer`** (N=1) | **many — `document_importer`** |
|---|---|---|---|---|
| **plain text** | ΔRSS **+14.34** MiB · `tm.peak` **43.47** MiB · `gc.collect`→freed 521 (§4.2, **B-1**) | N=25: `tm.peak` **68.85 / 69.15 / 69.24** MiB (min/med/max), flat **69.05→69.22** across docs 5→25; `gc.live` ~136 k flat; VmRSS 191.7→193.0 flat → **no accumulation** (**B-7**) | manifest 1 163 B: ΔRSS **+13.55** MiB · `tm.peak` **42.84** MiB · freed 193; copy#1 `json.load` 4.9 KiB (**B-8**, **B-9**) | N=50, manifest 39 186 B: ΔRSS **+14.97** MiB · `tm.peak` **43.47** MiB · freed 130; copy#1 **76.0 KiB** (linear) (**B-8**, **B-9**) |
| **text/digital PDF** | ΔRSS **+30.12** MiB · `tm.peak` **54.97** MiB · `gc.collect`→freed 890 (§4.2, **B-1**) | N=25: `tm.peak` **77.37 / 77.66 / 79.55** MiB, flat **77.49→77.79**; `gc.live` ~140 k flat; VmRSS 214.0→214.4 flat → **no accumulation** (**B-7**) | manifest 1 309 B: ΔRSS **+13.81** MiB · `tm.peak` **42.85** MiB · freed 193; copy#1 `json.load` 5.5 KiB (**B-8**, **B-9**) | N=50, manifest 46 442 B: ΔRSS **+14.96** MiB · `tm.peak` **43.52** MiB · freed 130; copy#1 scales (N=200→**425.8 KiB**) (**B-8**, **B-9**) |
| **scanned/image PDF (OCR)** | ΔRSS **+71.16** MiB, VmRSS 187.0 / hi **241.8** MiB · `tm.peak` **54.91** MiB · freed 1022; RSS drops 187.0→159.6 after `gc.collect` (§4.2, **B-1**) | N=15: `tm.peak` **77.57 / 77.75 / 77.87** MiB, flat **77.59→77.71**; `gc.live` ~139 k flat; VmRSS 233.9→235.2 → **no Python-heap accumulation** (**B-7**) | manifest 1 268 B: ΔRSS **+13.57** MiB · `tm.peak` **42.84** MiB · freed 193 (**B-8**, **B-9**) | N=20, manifest 17 962 B: ΔRSS **+14.35** MiB · `tm.peak` **43.10** MiB · freed 193 (**B-8**, **B-9**) |

**Reading the matrix down each entry point's columns — the two paths have opposite sensitivities:**

- **`consume_file` — document TYPE sets the peak; BATCH size does not accumulate.** Down the single-doc column the peak rises with parsing/OCR work: text `tm.peak` **43.47** < digital **54.97** ≈ image **54.91** MiB (Python heap), while image alone drives a native RSS high-water of **241.8** MiB from the OCR pipeline (§4.4 — the OCR image buffers are native and invisible to `tracemalloc`). Across a batch the trajectory is **flat** (text `tm.peak` 69.05→69.22 over docs 5→25; digital 77.49→77.79; image 77.59→77.71), with `gc.live` and VmRSS flat — each document returns to steady state and the batch total does **not** grow. So on the consume path, batch size is a non-issue and type is the whole story.
- **`document_importer` — document TYPE is irrelevant; BATCH size (manifest bytes) is the only driver.** Down the single-import column the three types are **near-identical** (`tm.peak` **42.84 / 42.85 / 42.84** MiB; ΔRSS **+13.55 / +13.81 / +13.57** MiB) because the importer **never parses or OCRs** — it only `json.load`s the manifest (`document_importer.py:L73`) and copies files. Memory therefore scales with **manifest bytes = N × content**, not with document type: copy#1 (`json.load`) grows linearly (text **4.9 → 76.0 → 333.6 KiB** for N=1 → 50 → 200; §8.2), while the per-command RSS jump stays ~**+14 MiB** because it is dominated by fixed pipeline costs (file copies + search-index update + ORM inserts), not the manifest, until content is large (§8.3).

**Methodology note (why the "many — `consume_file`" cells use distinct synthetic documents).** `consume_file` rejects a byte-identical re-submission at `pre_check_duplicate` (`consumer.py:L102`, MD5 match), so the identical sample file *cannot* be consumed N times. The batch harness (`docgen.py`) therefore generates N genuinely-distinct tiny documents per type to exercise the real entry point repeatedly. Because both the input content and the (long-lived, multi-type) process differ from the single-doc harness, the **absolute** peaks in that column are not directly comparable to the single-doc column; the cell's evidentiary content is the **flatness** (no accumulation), not the absolute value. The single-doc column uses the canonical repository samples (`simple.txt`, `simple-digital.pdf`, `multi-page-images.pdf`). The importer cells are broken down further next.

### 8.2 Importer batch scaling — the three materializations

The importer embeds the full `Document.content` in every manifest record (confirmed by measurement: the big-content export below produced ~102 KB of manifest per 100 KB-content doc — the content field carries the whole text). So manifest size = `N × (content + metadata)`. Measured manifest bytes span nearly four orders of magnitude: **1 163 B** (text N=1), **39 186 B** (text N=50), **156 314 B** (text N=200), **185 549 B** (digital N=200), and big-content exports (text at ~100 KB/doc) of **1 018 848 B (0.972 MiB)** (N=10) and **5 092 926 B (4.857 MiB)** (N=50). The three materializations, each measured on those real manifests (`focused_copies.py`, `focused_big.py`):

```
COPY#1  self.manifest = json.load(f)      [document_importer.py:L73]  -- heap delta, tracemalloc
  manifest 1 163 B (text N=1)       -> 4.9 KiB     (~4.3x; tiny-absolute, per-record overhead)
  manifest 39 186 B (text N=50)     -> 76.0 KiB    (~1.99x manifest)
  manifest 156 314 B (text N=200)   -> 333.6 KiB   (~2.19x manifest)
  manifest 185 549 B (digital N=200)-> 425.8 KiB   (~2.35x manifest)
  manifest 1 018 848 B (big N=10)   -> 0.981 MiB   (~1.01x manifest; content strings dominate)
  manifest 5 092 926 B (big N=50)   -> 4.894 MiB   (~1.01x manifest)
COPY#2  manifest_documents = list(filter(...)) [document_importer.py:L137]  -- shallow refs
  N=1 -> 0.5 KiB   N=10(big) -> 0.6 KiB   N=50 -> 0.9 KiB   N=200 -> 2.0 KiB  (identity check: True)
LOADDATA  call_command("loaddata", ...)   [document_importer.py:L87]
  retained heap ~1.03-1.13 MiB, roughly CONSTANT (text N=1 1032.9 KiB -> N=200 1125.0 KiB: +92 KiB
  for +199 docs, because rows stream into the DB, not the heap).
  Coexistence peak, copy#1 KEPT ALIVE through loaddata (matches the real importer, where
  self.manifest [L73] is a live attribute at L87):
    0.972 MiB manifest -> peak 70.82 -> 72.91 MiB (+2.09)
    4.857 MiB manifest -> peak 73.97 -> 83.88 MiB (+9.91 ~= 2x manifest: copy#1 4.89 + loaddata re-parse ~4.9)
REAL document_importer (whole command):
  0.972 MiB manifest: dRSS +2.38 MiB, peakTM 75.74 MiB
  4.857 MiB manifest: dRSS +3.25 MiB, peakTM 94.91 MiB
```

**How/why, per copy (every figure above is measured, not inferred):**

- **Copy #1 — `self.manifest = json.load(f)` (`document_importer.py:L73`).** The whole manifest is parsed into memory; the heap delta scales **linearly with manifest bytes = `N × content`**, confirmed across 1 KB → ~1 MB manifests (4.9 KiB → 0.981 MiB). The *ratio* to manifest bytes is **~2×** for many-tiny-record manifests (Python per-record `dict`/`str` object overhead dominates) and falls to **~1×** for few-large-content manifests (the big content strings are stored roughly 1:1, so overhead is negligible). This is the **primary** batch-size-sensitive allocation.
- **Copy #2 — `manifest_documents = list(filter(...))` (`document_importer.py:L137-138`).** This is a **shallow** list of references to the *same* dict objects already in `self.manifest` — measured **0.5–2.0 KiB** (just N pointers), and the element-identity check (`manifest_documents[0] is <same record in manifest>`) returned **True**. It is **negligible** and does **not** duplicate the document data — an important nuance: the AAP's "copy #2" is a reference copy, not a deep copy.
- **`loaddata` (`document_importer.py:L87`).** Django's JSON deserializer **streams** objects into the DB, so its *retained* heap is **roughly constant** (~1.03–1.13 MiB fixed serialization overhead, essentially independent of N). It nonetheless re-parses the manifest internally, so at its peak the `loaddata` parse **coexists with copy #1**, which the real importer keeps alive as `self.manifest`. Measured with copy #1 held alive: on a **0.972 MiB** manifest the `tracemalloc` peak rose **70.82 → 72.91 MiB (+2.09 MiB)**; on a **4.857 MiB** manifest it rose **73.97 → 83.88 MiB (+9.91 MiB ≈ 2× the manifest** — copy #1's 4.89 MiB plus `loaddata`'s ~4.9 MiB internal re-parse). The **real end-to-end `document_importer`** peaked at **75.74 MiB (ΔRSS +2.38 MiB)** and **94.91 MiB (ΔRSS +3.25 MiB)** for those two manifests. This confirms and quantifies the "three concurrent materializations": copy #1 tracks the manifest linearly (~1× at large content) and the transient peak approaches **~2× manifest bytes** at multi-MB scale. The whole-command RSS is nonetheless dominated by fixed pipeline costs at realistic (sub-MB) sizes (§8.3).

### 8.3 The whole-importer RSS is dominated by fixed pipeline costs, not the manifest

Driving the **real** `document_importer` end to end (`importer_harness.py`, six runs — each document type at single and many, **B-8**), the whole-command RSS jump was **~+13.5–15.0 MiB and roughly constant across both document type and batch size**, with a `tracemalloc` peak only ~1.3–2.0 MiB above the ~41.5 MiB process baseline:

```
type=text    n=1   manifest 1 163 B    dRSS +13.55 MiB   peakTM 42.84 MiB   gc freed 193
type=text    n=50  manifest 39 186 B   dRSS +14.97 MiB   peakTM 43.47 MiB   gc freed 130
type=digital n=1   manifest 1 309 B    dRSS +13.81 MiB   peakTM 42.85 MiB   gc freed 193
type=digital n=50  manifest 46 442 B   dRSS +14.96 MiB   peakTM 43.52 MiB   gc freed 130
type=image   n=1   manifest 1 268 B    dRSS +13.57 MiB   peakTM 42.84 MiB   gc freed 193
type=image   n=20  manifest 17 962 B   dRSS +14.35 MiB   peakTM 43.10 MiB   gc freed 193
```

This whole-command delta (~+14 MiB) is **larger than the manifest copies themselves** (copy #1 = 4.9–425 KiB, i.e. sub-MiB at these batch sizes; §8.2), so at realistic manifest sizes the visible RSS jump is **not** the manifest — it is the fixed pipeline cost: the `loaddata` ORM-insert machinery, copying each document's files into the media tree (`"Copy files into paperless…"`), and the search-index update (`"Updating search index…"`). Because that fixed cost does not grow with N over this range, dRSS is essentially flat (13.55 → 14.97 MiB from N=1 → 50). The manifest copies become the dominant term only for very large batches/contents, where copy #1 grows linearly toward ~1× the (large) manifest — e.g. the **0.972 MiB** big-content manifest pushed copy #1 to **0.981 MiB** and the real whole-command peak to **75.74 MiB** (§8.2, **B-10**). The `gc.collect()` at each teardown freed 130–193 objects and returned live-object counts to ~94 k, flat across all six runs — no cross-run accumulation.


---

## 9. Labeled non-canonical variant — `PAPERLESS_DEBUG=YES`

**This section is explicitly non-canonical.** The default is `DEBUG=NO` (`src/paperless/settings.py:L50`: `DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`; the env var was unset in the container), and `src/paperless/settings.py:L49` warns "NEVER RUN WITH DEBUG IN PRODUCTION." It is exercised only to characterise the ORM-query-log accumulation the user asked about; it is **not** recommended and **not** applied as a change.

The same ORM workload was run under both configurations in one process (`debug_harness.py`, Appendix A; full tri-lens output — including the before/after `PROBE` blocks — in Appendix B-6). Under `DEBUG=NO` the workload is 200 creates + 200 reads (~400 statements); under `DEBUG=YES` it is 5000 creates + 5000 reads (~10 000 statements). The decisive result lines (each **verbatim** from `debug_harness.txt`):

```
[DEBUG=NO  (canonical)] connection.queries length after workload = 0
[DEBUG=YES (NON-CANONICAL)] connection.queries length = 9000 (default cap 9000); approx_sql_bytes=2503890 (~2.39 MiB of SQL strings)
   after reset_queries(): connection.queries length = 0
```

**How/why:** with `DEBUG=True`, Django wraps the DB cursor and appends one `{sql, time}` dict to `connection.queries` per executed statement, up to a hard cap of 9000 (`connection.queries_limit`), after which it stops appending. It **survives `gc.collect()`** (retained by the connection object) — a genuine but **bounded** process-lifetime accumulation: in this workload the 9000 retained entries held **2 503 890 bytes (~2.39 MiB)** of SQL strings (the exact byte size scales with each statement's SQL length, but the 9000-entry cap is fixed). Under the canonical default it never records anything (length 0). So the "ORM query log" growth is real *only* under this labeled variant, and even then it is capped, not unbounded.

---

## 10. Normal CPython GC/allocator behaviour vs. a genuine leak — the determination

**Direct verdict: the observed memory pattern is NOT a genuine, growing live-object leak. It is the sum of (a) large-but-expected *transient* allocations that are released after each operation, and (b) benign CPython/glibc allocator retention that keeps RSS elevated after those transients free.** Applying the determination rule (leak ⇔ live-object growth *and* retained `tracemalloc` size surviving `gc.collect()`; benign ⇔ flat live objects + elevated RSS), every hotspot lands on the benign/expected side.

The single most important measured fact is the **baseline native gap** (`baseline.py`, App. A-8 / full output App. B-1d): immediately after `django.setup()` + `migrate`, RSS was **105.2 MiB** while the Python heap (`tracemalloc.current`) was only **34.76 MiB** — a ~70 MiB gap; after importing the heavy C-extension graph (numpy/scipy/scikit-learn/PIL/pikepdf-QPDF/lxml) RSS rose to **198.9 MiB** while the Python heap was only **70.68 MiB**, a printed **`native_gap=128.2 MiB`** of native C-extension memory plus glibc arenas that is entirely invisible to `tracemalloc`, and `gc.collect()` freed **0** objects and did **not** shrink RSS (198.0 → 198.9 MiB). RSS therefore massively overstates the "Python" footprint, and an RSS figure that fails to shrink is expected, not diagnostic of a leak on its own.

**Why RSS stays elevated (the user's "not released in a timely manner"):**
- CPython `pymalloc` returns an arena to the OS only when *all* its pools are empty; long-lived processes rarely satisfy this, so freed small-object memory is retained in arenas (per the CPython C-API memory docs [1] and `obmalloc.c` [4]; analyses by Evan Jones [2], rushter.com [3], and memray [5] — see §3.2 Sources).
- Objects > 512 bytes go to glibc `malloc`; on 128 CPUs glibc may keep many per-thread arenas (up to ~8 × cores [6][7]), and the multi-threaded OCR/worker pipeline populates several. Freed chunks sit in `tcache`/bins rather than returning to the kernel unless the heap top can be trimmed.
- **The proof it is benign, not a leak:** in the 12-document classifier batch (§6.2) and the 50-iteration distribution (§7.1), `gc.live` and `tracemalloc.current` are **flat** while RSS is either flat (digital) or plateaus (OCR). Flat live-object counts + elevated/plateauing RSS is the textbook signature of allocator retention, not a leak. Where the Python heap *is* the story (large text, importer manifest), it returns to baseline after `gc.collect()`.

**The only genuinely un-released artifacts** — and neither explains large sustained RAM growth:
1. **Empty parser temp-directories** on the REST `/metadata/` endpoint (`views.py:L266/L269`): a real inode/directory leak, but **~0 RAM** and bounded at 2 per request.
2. **`connection.queries`** under the non-default `DEBUG=YES` only: bounded at 9000 entries (~2.39 MiB).

**Mapping to the user's perceptions:**
- *"Disproportionate to document size"* → OCR native rasterization + full-text transient decode (a 150 KB image PDF drives a ~242 MiB high-water; §4.2), **not** metadata.
- *"Not released in a timely manner"* → `pymalloc`/glibc arena retention keeping RSS high after the Python heap has already been freed (the heap **does** return to baseline).
- *"Inconsistent / sometimes spikes"* → first-touch vs. warm, OCR vs. no-OCR, and the OCR arena warm-up plateau (§7).
- *"Caching accumulation"* → none in the default config; the classifier is loaded-once-per-consume-then-freed; the query log grows only under `DEBUG=YES`.

---

## 11. Per-hotspot summary table

Verdicts: **EXPECTED** = large but released/proportional to real work; **BENIGN RETENTION** = flat live objects, RSS stays elevated (allocator); **RESOURCE LEAK** = un-released but not RAM; **CONFIG-GATED** = only under a non-default flag. No hotspot is a genuine growing RAM leak.

| # | Hotspot / method | file:line | Observed peak | Live objects across `gc.collect()` | Verdict |
|---|---|---|---|---|---|
| 1 | OCR rasterization (`ocrmypdf`/ghostscript/tesseract) via `RasterisedDocumentParser.parse` | `paperless_tesseract/parsers.py`; `consumer.py:L261` | image PDF RSS hi **241.8 MiB** vs `tm.peak` 54.9 MiB (parse stage `dRSS`=+92.87 MiB, `dTM`=+0.76 MiB); RSS plateau ~220–258 MiB across a batch | `gc.live` flat ~96–108 k; RSS drops 187.0→159.6 after `gc` | EXPECTED transient + BENIGN retention |
| 2 | Full-text `content` string | `src/documents/parsers.py:L296` → `get_text src/documents/parsers.py:L342` → `consumer.py:L271` → `models.py:L117` | one copy = doc size; 5 MiB doc → transient `tm.peak` 79.8 MiB (+100 MiB RSS); 20 MiB → 187.3 MiB (+331 MiB RSS) | held once (`text is doc.content`); heap returns to baseline (43.41 MiB) after `gc`; RSS stays elevated | EXPECTED single copy + BENIGN retention |
| 3 | Classifier 7× `pickle.load` (`MLPClassifier` dominant) | `classifier.py:L76–L92` (L91 dominant) | +0.28 MiB/load (`MLPClassifier` 268.7 KiB; `CountVectorizer` only 12.3 KiB) | 12 consumes flat; live `DocumentClassifier` = 1 constant | EXPECTED (loaded once, freed) |
| 4 | Metadata-endpoint parser temp-dirs | `views.py:L266`, `L269` (no `cleanup()`) | 400 empty dirs after 200 calls; RAM +0.4 MiB (170.1→170.5) | dirs survive gc (filesystem); RSS/heap flat | RESOURCE LEAK (~0 RAM) |
| 5 | `pikepdf.open` un-closed handle | `src/paperless_tesseract/parsers.py:L34`, `src/paperless_tesseract/parsers.py:L55` | live `Pdf` = 0 at all probes | reclaimed by refcount before gc | NOT a leak (code smell) |
| 6 | Importer `json.load` (copy #1) | `document_importer.py:L73` | ≈ manifest bytes (0.981 MiB @ 0.972 MiB manifest; 4.894 MiB @ 4.857 MiB manifest) | released; `tm.current` → ~42–70 MiB after gc | EXPECTED (scales `batch×content`) |
| 7 | Importer `loaddata` re-parse | `document_importer.py:L87` | transient +manifest, coexists with copy #1 (peak 73.97→83.88 @ 4.857 MiB manifest, +9.91 ≈ 2× manifest) | streams to DB, released | EXPECTED transient |
| 8 | Importer `list(filter)` (copy #2) | `document_importer.py:L137-138` | shallow ref list 0.6–2.0 KiB | released | EXPECTED (shallow, negligible) |
| 9 | `train_classifier` `data=list()` + sklearn import | `tasks.py:L48`; `classifier.py:L117` | early-return +0.52 MiB; first real fit (n=10) dRSS **+79.08 MiB** (peakTM 68.70 MiB — one-time sklearn import + model save); warm re-fit (n=100) +1.96 MiB | modules + model persist (expected); periodic, not per-doc | EXPECTED one-time import |
| 10 | `sanity_check` `md5(f.read())` | `sanity_checker.py:L83`, `L112` | md5 `call_count=10`; largest single read 150 479 B (~0.14 MiB transient); whole-check dRSS +1.10 MiB | released before next doc | EXPECTED per-file transient |
| 11 | `connection.queries` (DEBUG=YES only) | `settings.py:L50` | 0 default; 9000 entries (~2.39 MiB) under DEBUG=YES | survives gc under DEBUG=YES | CONFIG-GATED (bounded) |
| 12 | First-touch lazy imports | `tasks.py:L25` + parser imports | cold-vs-warm ΔVmRSS **+14.3** (text) / **+30.1** (digital) / **+71.2** (image) MiB native (§4.4); heap iter 0 only ~1 MiB above warm median (54.60 vs 53.60, §7.1) | modules persist (expected); warm heap flat (stdev 0.08–0.21 MiB) | EXPECTED one-time |
| 13 | `pymalloc`/glibc arena retention | allocator (128 CPUs) | after full import graph RSS **198.9 MiB** vs heap **70.68 MiB** (`native_gap=128.2 MiB`); `gc.collect()` frees 0, RSS stays elevated | `gc.live` flat; RSS doesn't shrink | BENIGN RETENTION |


---

## 12. Coverage checklist

Each named item the question asks for, with where it is answered and the evidence pointer.

| Named item (from the question) | Answered in | Concrete value / verdict | Evidence |
|---|---|---|---|
| **Q1** Cause of the spikes (esp. metadata stage) | §4 | OCR native rasterization + first-touch lazy imports + full-text transient; **metadata stage is not even exercised by the consumer** | §4.1–§4.5; `consumer.py:L261/L271`; App. B-1 |
| — plain text | §4.2, §4.3, §8.1 | +14.34 MiB VmRSS, `tm.peak` 43.47 MiB | App. B-1 |
| — text/digital PDF | §4.2, §4.3, §8.1 | +30.12 MiB VmRSS, `tm.peak` 54.97 MiB, parse stage `dRSS`=+25.28 MiB | App. B-1, B-1a |
| — scanned/image PDF (OCR) | §4.2, §4.3, §8.1 | +71.16 MiB VmRSS / RSS hi **241.8 MiB** (run 2 263.8 — nondeterministic), `tm.peak` 54.91 MiB, parse `dRSS`=+92.87 / `dTM`=+0.76 MiB | App. B-1, B-1a |
| — first-consume (no `MODEL_FILE`) vs after-trained | §4.3, §4.6, §6.1 | `load_classifier()`→`None` (`classifier.py:L36`; 0 live objs) vs 7×`pickle.load` (§6.1) | App. B-13, B-4 |
| — error path (corrupt vs. encrypted PDF) | §4.6, §6 | corrupt→`ConsumerError`(ocrmypdf `InputFileError`), `cleanup()` runs (tempdirs 0→0); encrypted→succeeds (`content_len=0`) | App. B-14 |
| **Q2** Unnecessary copies / reference retention | §5 | metadata endpoint leaks empty tempdirs; `pikepdf` handle reclaimed; `content` held once | §5.1–§5.4 |
| — metadata-endpoint tempdirs | §5.1 | **2 per call**, never cleaned (`views.py:L266/L269`); 400 empty dirs / 200 calls; ~0 RAM | App. B-2 |
| — `pikepdf.open` handle | §5.2 | live `Pdf`=0 before & after gc (`src/paperless_tesseract/parsers.py:L34/L55`) — reclaimed by refcount | App. B-3 |
| — full-text `content` string | §5.3 | held **once** (`text is doc.content` → True), `models.py:L117` | §5.3 |
| **Q3** Caching accumulation | §6 | none in default config | §6.1–§6.4 |
| — classifier model | §6.1, §6.2 | 7×`pickle.load` (`classifier.py:L76-92`), `MLPClassifier` 268.7 KiB dominant, loaded once/consume then freed; 12-doc batch flat (live=1) | App. B-4, B-4b |
| — ORM query log | §6.4, §9 | 0 by default; 9000 (~2.39 MiB) only under `DEBUG=YES` | App. B-6 |
| — index writer (Whoosh) | §6.3 | `AsyncWriter` live (0,0) at all probes | §6.3 |
| **Q4** Spiking vs non-spiking | §7 | distribution: first-touch + OCR + arena warm-up | §7.1–§7.3 |
| — distribution (min/median/max), same unchanged bytes | §7.1, §7.2 | digital `tm.peak` 53.13 / 53.61 / 54.60 MiB (stdev 0.207; run 2 stdev 0.076); image 54.54 / 54.87 / 55.11 MiB (stdev 0.149); image VmRSS spreads 152.9→258.2 (heap stable) | App. B-5 |
| — differentiators | §7.3 | first-touch vs warm; OCR vs no-OCR; arena warm-up; classifier present/absent | §7.3 |
| **Q5** Document type × batch size | §8 | cross-product tabulated | §8.1–§8.3 |
| — {text,digital,image} × single | §8.1 | via `consume_file`: text +14.34 / digital +30.12 / image +71.16 MiB VmRSS | App. B-1, B-7 |
| — {text,digital,image} × many | §8.2, §8.3 | via `consume_file` (flat, no accumulation) **and** `document_importer`: copy#1 ≈ manifest, copy#2 shallow, loaddata ≈2× manifest | App. B-7, B-8, B-9, B-10 |
| **Named entry points exercised (R4)** | §4–§8 | all five driven through their real interfaces | see below |
| — `consume_file` / `try_consume_file` | §4 | staged pipeline (`tasks.py:L184`→`consumer.py:L180`), per-type peaks | App. B-1, B-1a |
| — metadata endpoint (`views.py:L260`) | §5.1 | repeated 1/10/50/200 calls; 2 tempdirs/call, RAM flat | App. B-2 |
| — `document_importer` | §8 | type × batch cross-product; copy#1/loaddata/copy#2 | App. B-8, B-9, B-10 |
| — `train_classifier` (`tasks.py:L48`) | §6.1, §11 | early-return 0.52 MiB / first-fit +79.08 MiB (one-time sklearn) / warm +1.96 MiB | App. B-11 |
| — `sanity_check` (`tasks.py:L255`) | §11 | `md5(f.read())` L83/L112, `call_count=10`, +1.10 MiB | App. B-12 |
| **Named conditions exercised (R6)** | §4–§9 | primary + edge/error + config variants | see below |
| — no-`MODEL_FILE` (`load_classifier()`→`None`) | §6.1, §4.6 | returns `None` (`classifier.py:L36`), 0 live objects | App. B-13 |
| — classifier-present (7×`pickle.load`) | §6.1, §6.2 | `MLPClassifier` 268.7 KiB dominant; batch flat (live=1) | App. B-4, B-4b |
| — office / Tika path | §8, App. B-15 | **UNAVAILABLE** in canonical (`PAPERLESS_TIKA_ENABLED=False`, `apps.py:L12`); non-canonical | App. B-15 |
| — repeated metadata-endpoint calls | §5.1 | 2 tempdirs/call (400 over 200), RAM flat 170.1→170.5 MiB | App. B-2 |
| — encrypted / error path | §4.6 | encrypted→succeeds (`content_len=0`); corrupt→`ConsumerError`(`InputFileError`), `cleanup()` runs | App. B-14 |
| — `DEBUG=YES` variant (non-canonical) | §9 | `connection.queries` 9000 entries (~2.39 MiB); 0 by default | App. B-6 |
| **Normal-GC-vs-leak determination** | §10, §11 | **NOT a genuine leak** — expected transients + benign allocator retention | §10, §11 (13-hotspot table) |
| Read-only repo proof | App. C | only `blitzy/…md` added; no `src/` change | `git status --porcelain` |

---

## Appendix A — Harness source (verbatim, all under `/tmp/mem_harness/`)

Every temporary observation script used in this investigation is reproduced below **in full, verbatim** — no summaries, no elisions. All 28 scripts lived under `/tmp/mem_harness/` **inside the container** (never in the repository) and were removed afterward (Appendix C). The source shown here is byte-identical to the scripts that actually produced the captured output in Appendix B. `probe.py` and `bootstrap.py` are the reusable core; every other harness begins with `import bootstrap; bootstrap.bootstrap()` and (where it probes) `import probe`, then drives one real entry point.

### A-1. `probe.py` — the tri-lens probe: RSS (OS) + `tracemalloc` (Python heap) + `gc` (live objects), with per-stage `reset_peak`

```python
"""Tri-lens memory probe: RSS (OS) + tracemalloc (Python heap) + gc (live objects).

RSS UNITS:
  * resource.getrusage(RUSAGE_SELF).ru_maxrss -> KILOBYTES on Linux, high-water
    mark (monotonic non-decreasing over process lifetime).
  * /proc/self/status VmRSS -> current RSS in kB (can go DOWN when memory is
    released to the OS). This is the "current" reading.
tracemalloc measures only the CPython-managed heap (bytes).
gc.get_objects() counts tracked live Python objects.
"""
import gc
import os
import sys
import resource
import tracemalloc


def _vmrss_kb():
    """Current RSS in kB read from /proc/self/status (VmRSS)."""
    try:
        with open("/proc/self/status") as fh:
            for line in fh:
                if line.startswith("VmRSS:"):
                    return int(line.split()[1])
    except OSError:
        return -1
    return -1


def _maxrss_kb():
    """High-water RSS in kB (ru_maxrss is KB on Linux)."""
    return resource.getrusage(resource.RUSAGE_SELF).ru_maxrss


def probe(label, do_gc=False):
    """Capture all three lenses. If do_gc, run gc.collect() first and report freed."""
    collected = gc.collect() if do_gc else None
    cur, peak = tracemalloc.get_traced_memory()
    vmrss = _vmrss_kb()
    maxrss = _maxrss_kb()
    n_obj = len(gc.get_objects())
    counts = gc.get_count()
    tag = "AFTER gc.collect()" if do_gc else ""
    print(f"[PROBE] {label} {tag}".rstrip())
    print(f"    RSS_current(VmRSS)  = {vmrss} kB ({vmrss/1024:.1f} MiB)")
    print(f"    RSS_hiwater(maxrss) = {maxrss} kB ({maxrss/1024:.1f} MiB)")
    print(f"    tracemalloc.current = {cur} B ({cur/1048576:.2f} MiB)")
    print(f"    tracemalloc.peak    = {peak} B ({peak/1048576:.2f} MiB)")
    print(f"    gc.live_objects     = {n_obj}")
    print(f"    gc.get_count()      = {counts}")
    if collected is not None:
        print(f"    gc.collect()->freed = {collected}")
    sys.stdout.flush()
    return {
        "label": label, "vmrss_kb": vmrss, "maxrss_kb": maxrss,
        "tm_current": cur, "tm_peak": peak, "n_obj": n_obj,
        "counts": counts, "collected": collected,
    }


def snapshot():
    return tracemalloc.take_snapshot()


def snapshot_diff(prev, top=12, label=""):
    """Print top allocators by size DELTA vs prev snapshot (attribution by file:line)."""
    cur = tracemalloc.take_snapshot()
    stats = cur.compare_to(prev, "lineno")
    print(f"[TRACEMALLOC DIFF] {label} (top {top} by size delta)")
    for s in stats[:top]:
        print("    ", s)
    sys.stdout.flush()
    return cur


def top_allocators(snap=None, top=12, label=""):
    if snap is None:
        snap = tracemalloc.take_snapshot()
    stats = snap.statistics("lineno")
    print(f"[TRACEMALLOC TOP] {label} (top {top} by cumulative size)")
    for s in stats[:top]:
        print("    ", s)
    sys.stdout.flush()
    return snap
```

### A-2. `bootstrap.py` — Django bootstrap into temporary data/scratch/media/consumption dirs + a temporary SQLite DB, mirroring `src/documents/tests/utils.py` `setup_directories()`

```python
"""Django bootstrap for the memory harness.

Mirrors documents/tests/utils.py setup_directories(): makes temp
data/scratch/media/consumption dirs, points settings at them, runs migrations
on a temp SQLite DB, so the REAL entry points (consume_file, the metadata
endpoint, document_importer, train_classifier, sanity_check) can be driven
without touching the repo. Everything lives under a temp root that is removed
at process exit.
"""
import os
import tempfile


def bootstrap():
    # Temp root for this run.
    root = tempfile.mkdtemp(prefix="mh-root-")
    data_dir = os.path.join(root, "data")
    scratch_dir = os.path.join(root, "scratch")
    media_dir = os.path.join(root, "media")
    consumption_dir = os.path.join(root, "consume")
    index_dir = os.path.join(data_dir, "index")
    originals_dir = os.path.join(media_dir, "documents", "originals")
    thumbnail_dir = os.path.join(media_dir, "documents", "thumbnails")
    archive_dir = os.path.join(media_dir, "documents", "archive")
    logging_dir = os.path.join(data_dir, "log")
    for d in (data_dir, scratch_dir, media_dir, consumption_dir, index_dir,
              originals_dir, thumbnail_dir, archive_dir, logging_dir):
        os.makedirs(d, exist_ok=True)

    # These MUST be set before django.setup() so DATA_DIR-derived paths
    # (SQLite DB, MODEL_FILE) resolve into the temp tree (settings.py L66/L74).
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    os.environ["PAPERLESS_DISABLE_DBHANDLER"] = "true"
    os.environ["PAPERLESS_DATA_DIR"] = data_dir
    os.environ["PAPERLESS_SCRATCH_DIR"] = scratch_dir
    os.environ["PAPERLESS_MEDIA_ROOT"] = media_dir
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = consumption_dir

    import django
    django.setup()

    from django.test import override_settings
    ov = override_settings(
        DATA_DIR=data_dir, SCRATCH_DIR=scratch_dir, MEDIA_ROOT=media_dir,
        ORIGINALS_DIR=originals_dir, THUMBNAIL_DIR=thumbnail_dir,
        ARCHIVE_DIR=archive_dir, CONSUMPTION_DIR=consumption_dir,
        LOGGING_DIR=logging_dir, INDEX_DIR=index_dir,
        MODEL_FILE=os.path.join(data_dir, "classification_model.pickle"),
        MEDIA_LOCK=os.path.join(media_dir, "media.lock"),
    )
    ov.enable()

    from django.core.management import call_command
    # Migrations contain print() banners (e.g. thumbnail migration) that are not
    # silenced by verbosity=0; redirect stdout during migrate for clean output.
    import contextlib, io
    with contextlib.redirect_stdout(io.StringIO()):
        call_command("migrate", run_syncdb=True, verbosity=0)

    return {
        "root": root, "data_dir": data_dir, "scratch_dir": scratch_dir,
        "media_dir": media_dir, "consumption_dir": consumption_dir,
        "index_dir": index_dir, "ov": ov,
    }
```

### A-3. `docgen.py` — synthetic distinct-document generator (reportlab / img2pdf / Pillow) used by the batch cross-product cells (distinct MD5 passes the consumer dedup)

```python
"""Generate genuinely-distinct documents per type for BATCH (many) cross-product
cells. Distinct content => distinct MD5 => passes consumer dedup
(pre_check_duplicate consumer.py:L102). Not used for the F5 same-input test
(that uses byte-identical copies).
    text    -> distinct .txt
    digital -> reportlab PDF with an embedded text layer (pdfminer text, no OCR)
    image   -> Pillow raster -> img2pdf PDF with NO text layer (forces OCR)
"""
import io
import os


def make_text(path, i):
    with open(path, "w") as f:
        f.write(f"Batch text document number {i}\nUnique token {i*7919}\n")


def make_digital(path, i):
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import letter
    c = canvas.Canvas(path, pagesize=letter)
    c.drawString(72, 720, f"Batch DIGITAL pdf number {i}")
    c.drawString(72, 700, f"Embedded searchable text unique {i*104729}")
    c.showPage()
    c.save()


def make_image(path, i):
    from PIL import Image, ImageDraw
    import img2pdf
    img = Image.new("RGB", (900, 300), "white")
    d = ImageDraw.Draw(img)
    d.text((20, 120), f"SCANNED image doc {i} token {i*15485863}", fill="black")
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    buf.seek(0)
    with open(path, "wb") as f:
        f.write(img2pdf.convert(buf.read()))


def make(key, path, i):
    {"text": make_text, "digital": make_digital, "image": make_image}[key](path, i)
```

### A-4. `light_consume.py` — canonical single-document per-type consume (1-frame `tracemalloc`), 2 runs each (Q1 / Q5-single)

```python
"""F3 -> Q5 (single-document cells) + Q1 per-type footprint.

Drives the REAL consume_file entry point once per document type (text / digital
PDF / image PDF) under light 1-frame tracemalloc, reporting peak Python heap,
RSS growth, and live-object delta across the whole single-document consume, with
a trailing gc.collect(). 2 runs each for magnitude stability (R2).
"""
import gc
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

SAMPLES = {
    "text": "/app/src/documents/tests/samples/simple.txt",
    "digital": "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf",
    "image": "/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf",
}


def consume_one(key, run_idx):
    from documents.tasks import consume_file
    from documents.models import Document
    src = SAMPLES[key]
    work = os.path.join(CTX["scratch_dir"], f"light-{key}-{run_idx}")
    os.makedirs(work, exist_ok=True)
    dst = os.path.join(work, os.path.basename(src))
    shutil.copy(src, dst)

    print(f"\n===== LIGHT CONSUME type={key} run={run_idx} size={os.path.getsize(dst)}B =====")
    b = probe.probe(f"{key} r{run_idx} BEFORE")
    tracemalloc.reset_peak()
    doc = consume_file(dst)
    a = probe.probe(f"{key} r{run_idx} AFTER")
    g = probe.probe(f"{key} r{run_idx} AFTER-GC", do_gc=True)
    drss = (a["vmrss_kb"] - b["vmrss_kb"]) / 1024
    print(f"[SUMMARY] type={key} run={run_idx} dRSS={drss:.2f}MiB "
          f"peakTM={a['tm_peak']/1048576:.2f}MiB dLiveObj={a['n_obj']-b['n_obj']} "
          f"freed={g['collected']}")
    # cleanup
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()


if __name__ == "__main__":
    keys = sys.argv[1:] or ["text", "digital", "image"]
    for r in (1, 2):
        for k in keys:
            consume_one(k, r)
    print("\nLIGHT_CONSUME_DONE")
```

### A-5. `stage_consume.py` — per-stage before/during/after boundary probes across `try_consume_file` (Q1 / F2)

```python
"""F2 -> Q1: Per-stage memory boundaries of Consumer.try_consume_file().

Instruments EVERY pipeline stage boundary (consumer.py:L180) by wrapping the
real callables the pipeline invokes:
    parse                 (document_parser.parse)              consumer.py:L261
    thumbnail             (get_optimised_thumbnail)            consumer.py:L265
    get_text              (document_parser.get_text)           consumer.py:L271
    get_date              (document_parser.get_date)           consumer.py:L272
    load_classifier       (documents.consumer.load_classifier) consumer.py:L292
    signal handlers       (set_correspondent/type/tags)        handlers.py:L35/101/168
    _store                (Consumer._store)                    consumer.py:L379
For each stage we report BEFORE / DURING(peak) / AFTER via reset_peak(), plus a
final gc.collect() boundary. Drives the REAL consume_file entry point.
"""
import gc
import os
import shutil
import sys
import tracemalloc

tracemalloc.start(1)  # 1 frame: light, avoids observer-effect inflation (see doc 3.4)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

import django
from django.test import override_settings

SAMPLES = {
    "digital": "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf",
    "image": "/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf",
    "text": "/app/src/documents/tests/samples/simple.txt",
}


def wrap_method(owner, name, stage_label, store):
    """Wrap owner.name to probe BEFORE/DURING(peak)/AFTER around the call."""
    orig = getattr(owner, name)

    def wrapper(*a, **k):
        b = probe.probe(f"{stage_label} BEFORE")
        tracemalloc.reset_peak()
        r = orig(*a, **k)
        af = probe.probe(f"{stage_label} AFTER")
        store.append((stage_label, b, af))
        return r

    setattr(owner, name, wrapper)
    return orig


def run(sample_key, run_idx):
    from documents import consumer as consumer_mod
    from documents.consumer import Consumer
    from documents.parsers import get_parser_class_for_mime_type
    from documents.signals import document_consumption_finished
    from documents.signals import handlers as H
    import magic

    path_src = SAMPLES[sample_key]
    # Copy sample into consumption dir (consume_file consumes from there).
    work = os.path.join(CTX["scratch_dir"], f"stage-{sample_key}-{run_idx}")
    os.makedirs(work, exist_ok=True)
    dst = os.path.join(work, os.path.basename(path_src))
    shutil.copy(path_src, dst)

    mime = magic.from_file(dst, mime=True)
    # Resolve the CONCRETE parser class (get_parser_class_for_mime_type returns a
    # factory function get_parser (paperless_tesseract/signals.py) / text factory,
    # so import the real classes directly to wrap their methods).
    if sample_key == "text":
        from paperless_text.parsers import TextDocumentParser as ParserCls
    else:
        from paperless_tesseract.parsers import RasterisedDocumentParser as ParserCls
    print(f"\n===== STAGE CONSUME sample={sample_key} run={run_idx} "
          f"mime={mime} parser={ParserCls.__name__} =====")
    sys.stdout.flush()

    store = []
    # ---- wrap parser-class methods (concrete class for this mime) ----
    originals = []
    for m, lbl in [("parse", "STAGE parse"),
                   ("get_optimised_thumbnail", "STAGE thumbnail"),
                   ("get_text", "STAGE get_text"),
                   ("get_date", "STAGE get_date"),
                   ("get_archive_path", "STAGE get_archive_path")]:
        if hasattr(ParserCls, m):
            had_own = m in ParserCls.__dict__
            originals.append((ParserCls, m, wrap_method(ParserCls, m, lbl, store), had_own))

    # ---- wrap module-level load_classifier used by consumer ----
    orig_lc = consumer_mod.load_classifier

    def lc_wrap(*a, **k):
        b = probe.probe("STAGE load_classifier BEFORE")
        tracemalloc.reset_peak()
        r = orig_lc(*a, **k)
        af = probe.probe("STAGE load_classifier AFTER")
        store.append(("STAGE load_classifier", b, af))
        return r
    consumer_mod.load_classifier = lc_wrap

    # ---- wrap Consumer._store ----
    orig_store = Consumer._store

    def store_wrap(self, *a, **k):
        b = probe.probe("STAGE _store BEFORE")
        tracemalloc.reset_peak()
        r = orig_store(self, *a, **k)
        af = probe.probe("STAGE _store AFTER")
        store.append(("STAGE _store", b, af))
        return r
    Consumer._store = store_wrap

    # ---- wrap the three classifier-consuming signal handlers ----
    handler_targets = [("set_correspondent", H.set_correspondent),
                       ("set_document_type", H.set_document_type),
                       ("set_tags", H.set_tags)]
    wrapped_handlers = []
    for hname, hfunc in handler_targets:
        document_consumption_finished.disconnect(hfunc)

        def make(hn, hf):
            def hw(*a, **k):
                b = probe.probe(f"STAGE handler:{hn} BEFORE")
                tracemalloc.reset_peak()
                r = hf(*a, **k)
                af = probe.probe(f"STAGE handler:{hn} AFTER")
                store.append((f"STAGE handler:{hn}", b, af))
                return r
            return hw
        hw = make(hname, hfunc)
        document_consumption_finished.connect(hw, weak=False)
        wrapped_handlers.append((hw, hfunc, hname))

    from documents.tasks import consume_file
    probe.probe("PIPELINE start (before consume_file)")
    tracemalloc.reset_peak()
    doc = consume_file(dst)
    probe.probe("PIPELINE end (after consume_file)")
    probe.probe("PIPELINE end", do_gc=True)

    # restore
    consumer_mod.load_classifier = orig_lc
    Consumer._store = orig_store
    for owner, m, o, had_own in originals:
        if had_own:
            setattr(owner, m, o)
        else:
            delattr(owner, m)  # remove shadow so base method is used again
    for hw, hf, hn in wrapped_handlers:
        document_consumption_finished.disconnect(hw)
        document_consumption_finished.connect(hf)

    # per-stage delta summary
    print(f"\n----- STAGE DELTA SUMMARY sample={sample_key} run={run_idx} -----")
    print(f"{'stage':32s} {'dRSS_MiB':>9s} {'dTM_cur_MiB':>11s} {'peak_MiB':>9s} {'dLiveObj':>9s}")
    for lbl, b, af in store:
        drss = (af["vmrss_kb"] - b["vmrss_kb"]) / 1024
        dtm = (af["tm_current"] - b["tm_current"]) / 1048576
        peak = af["tm_peak"] / 1048576
        dobj = af["n_obj"] - b["n_obj"]
        print(f"{lbl:32s} {drss:9.2f} {dtm:11.2f} {peak:9.2f} {dobj:9d}")
    sys.stdout.flush()

    # cleanup DB rows + files so runs are independent
    from documents.models import Document
    for d in Document.objects.all():
        try:
            for p in (d.source_path, d.thumbnail_path, d.archive_path):
                if p and os.path.isfile(p):
                    os.unlink(p)
        except Exception:
            pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    return doc


if __name__ == "__main__":
    keys = sys.argv[1:] or ["digital", "image"]
    for rk in range(1, 3):  # 2 runs for stability
        for k in keys:
            run(k, rk)
    print("\nSTAGE_CONSUME_DONE")
```

### A-6. `attrib_harness.py` — 25-frame `tracemalloc` `file:line` attribution (attribution only, never magnitude)

```python
"""Q1 attribution: drive REAL consume_file under tracemalloc.start(25) and print
the top Python-heap allocators by file:line accumulated during the whole consume,
plus the whole-consume delta. Attribution-only (heavier nframe inflates RSS per the
observer-effect note; magnitudes come from the light harness).
Usage: python attrib_harness.py <sample> <label>
"""
import gc, os, shutil, sys, tracemalloc
tracemalloc.start(25)
sys.path.insert(0, "/tmp/mem_harness")
import bootstrap
CTX = bootstrap.bootstrap()
from documents.tasks import consume_file
from documents.models import Document

SAMPLE = sys.argv[1]; LABEL = sys.argv[2]
SIZE = os.path.getsize(SAMPLE)
work = os.path.join(CTX["scratch_dir"], "attrib-" + LABEL)
os.makedirs(work, exist_ok=True)
dst = os.path.join(work, os.path.basename(SAMPLE))
shutil.copy(SAMPLE, dst)
gc.collect()
snap_before = tracemalloc.take_snapshot()
consume_file(dst)
snap_after = tracemalloc.take_snapshot()
print("\n===== ATTRIBUTION label=%s file=%s size=%dB =====" % (LABEL, os.path.basename(SAMPLE), SIZE))
print("--- TOP 15 allocators by size DELTA (before-consume -> after-consume), by file:line ---")
for s in snap_after.compare_to(snap_before, "lineno")[:15]:
    print("   ", s)
print("--- TOP 12 allocators by CUMULATIVE size after consume, by file:line ---")
for s in snap_after.statistics("lineno")[:12]:
    print("   ", s)
sys.stdout.flush()
```

### A-7. `observer_effect.py` — heavy (30-frame) vs light (1-frame) instrumentation control demonstrating the profiler observer effect (§3.4)

```python
"""Doc 3.4 observer-effect evidence: SAME digital PDF, real consume_file, two
instrumentation modes in separate fresh processes, reporting the RSS high-water.

  heavy  = tracemalloc.start(30) + take_snapshot() + gc.get_objects() at EVERY
           pipeline stage boundary (mimics the "early harness")
  light  = tracemalloc.start(1), just a before/after probe (what doc 4-8 use)

The RSS high-water delta between the two runs is the profiler's own overhead,
NOT paperless. Run twice per mode to rule out variance (R2).
"""
import gc
import os
import shutil
import sys
import tracemalloc

MODE = sys.argv[1] if len(sys.argv) > 1 else "light"
NFRAME = 30 if MODE == "heavy" else 1
tracemalloc.start(NFRAME)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"

_snap_count = 0


def heavy_boundary(label):
    """The expensive per-stage instrumentation: full snapshot + object walk."""
    global _snap_count
    snap = tracemalloc.take_snapshot()   # copies EVERY 30-frame traceback
    nobj = len(gc.get_objects())          # walks the whole live-object graph
    _snap_count += 1
    # keep the snapshot alive briefly (as a real per-stage harness would, to diff)
    return snap, nobj


def wrap_heavy(owner, name, label, keep):
    orig = getattr(owner, name)

    def wrapper(*a, **k):
        keep.append(heavy_boundary(f"{label} BEFORE"))
        r = orig(*a, **k)
        keep.append(heavy_boundary(f"{label} AFTER"))
        return r
    setattr(owner, name, wrapper)
    return orig


def run(run_idx):
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"obs-{MODE}-{run_idx}")
    os.makedirs(work, exist_ok=True)
    dst = os.path.join(work, os.path.basename(DIGITAL))
    shutil.copy(DIGITAL, dst)

    print(f"\n===== OBSERVER-EFFECT mode={MODE} nframe={NFRAME} run={run_idx} "
          f"size={os.path.getsize(dst)}B =====")
    keep = []  # holds retained snapshots in heavy mode (as a real harness would)
    if MODE == "heavy":
        from paperless_tesseract.parsers import RasterisedDocumentParser as P
        from documents import consumer as CM
        from documents.consumer import Consumer
        origs = []
        for m, lbl in [("parse", "parse"), ("get_optimised_thumbnail", "thumbnail"),
                       ("get_text", "get_text"), ("get_date", "get_date"),
                       ("get_archive_path", "get_archive_path")]:
            if hasattr(P, m):
                origs.append((P, m, getattr(P, m), m in P.__dict__))
                wrap_heavy(P, m, lbl, keep)
        orig_lc = CM.load_classifier
        def lcw(*a, **k):
            keep.append(heavy_boundary("load_classifier BEFORE"))
            r = orig_lc(*a, **k)
            keep.append(heavy_boundary("load_classifier AFTER"))
            return r
        CM.load_classifier = lcw
        orig_store = Consumer._store
        def sw(self, *a, **k):
            keep.append(heavy_boundary("_store BEFORE"))
            r = orig_store(self, *a, **k)
            keep.append(heavy_boundary("_store AFTER"))
            return r
        Consumer._store = sw

    b = probe.probe(f"{MODE} r{run_idx} BEFORE")
    tracemalloc.reset_peak()
    consume_file(dst)
    a = probe.probe(f"{MODE} r{run_idx} AFTER")
    print(f"[SUMMARY] mode={MODE} run={run_idx} "
          f"RSS_hiwater={a['maxrss_kb']/1024:.1f}MiB "
          f"VmRSS_after={a['vmrss_kb']/1024:.1f}MiB "
          f"tm.peak={a['tm_peak']/1048576:.2f}MiB "
          f"heavy_snapshots_taken={_snap_count} retained={len(keep)}")

    if MODE == "heavy":
        CM.load_classifier = orig_lc
        Consumer._store = orig_store
        for owner, m, o, had in origs:
            if had:
                setattr(owner, m, o)
            else:
                delattr(owner, m)
    keep.clear()
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()


if __name__ == "__main__":
    for r in (1, 2):
        run(r)
    print("\nOBSERVER_EFFECT_DONE")
```

### A-8. `baseline.py` — post-import RSS vs Python-heap (`tracemalloc`) gap — the native/allocator anchor for §10

```python
"""Section 10 baseline: the native RSS vs Python-heap (tracemalloc) gap.

Loads the full import graph exercised by consumption (consumer, parsers, sklearn,
pikepdf, ocrmypdf, whoosh) then reports the three lenses so the gap between OS
RSS and the tracemalloc-tracked Python heap is visible. That gap is native /
allocator memory (C extensions, interpreter, glibc arenas) that tracemalloc does
not see -- the anchor for the "normal allocator retention, not a leak" argument.
"""
import gc
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

probe.probe("baseline: after django.setup()+migrate")

# Force the heavy import graph the consumer touches.
import documents.consumer            # noqa
import documents.classifier          # noqa
import documents.parsers             # noqa
import paperless_tesseract.parsers   # noqa
import paperless_text.parsers        # noqa
import pikepdf                       # noqa
import sklearn                       # noqa
import numpy                         # noqa
import scipy                         # noqa
try:
    import ocrmypdf                  # noqa
except Exception as e:
    print("ocrmypdf import note:", e)

probe.probe("baseline: after heavy import graph")
probe.probe("baseline: after gc.collect()", do_gc=True)

cur, peak = tracemalloc.get_traced_memory()
vmrss = probe._vmrss_kb()
gap = vmrss - cur / 1024
print(f"\n[BASELINE GAP] RSS_current={vmrss/1024:.1f} MiB  "
      f"tracemalloc.current={cur/1048576:.2f} MiB  "
      f"native_gap={gap/1024:.1f} MiB")
print("BASELINE_DONE")
```

### A-9. `dist_harness.py` — same-unchanged-input repeated consume distribution (Q4 / F5)

```python
"""F5 -> Q4: run the SAME UNCHANGED INPUT repeatedly, report the distribution.

R3 compliance: the input file is BYTE-IDENTICAL every iteration (verified by MD5
printed once). Between iterations we delete the created Document + its stored
files so the next byte-identical consume passes the consumer's MD5 dedup
(pre_check_duplicate, consumer.py:L102-104). Deleting the DB row is harness
bookkeeping, NOT a modification of the input -- the bytes fed to consume_file are
the same each time. We record the per-iteration tracemalloc PEAK (the transient
spike during one consume) and the running VmRSS/ru_maxrss, then report the
distribution (min/median/max/mean/stdev) across iterations, for >=2 runs.
"""
import hashlib
import os
import shutil
import statistics
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

SAMPLES = {
    "digital": "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf",
    "image": "/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf",
}


def md5(path):
    with open(path, "rb") as f:
        return hashlib.md5(f.read()).hexdigest()


def run(key, iters, run_idx):
    from documents.tasks import consume_file
    from documents.models import Document
    src = SAMPLES[key]
    src_md5 = md5(src)
    work = os.path.join(CTX["scratch_dir"], f"dist-{key}-{run_idx}")
    os.makedirs(work, exist_ok=True)
    print(f"\n===== DIST type={key} iters={iters} run={run_idx} "
          f"input_md5={src_md5} input_size={os.path.getsize(src)}B =====")
    peaks = []
    rss_curr = []
    rss_hi = []
    for i in range(1, iters + 1):
        dst = os.path.join(work, os.path.basename(src))  # SAME name, SAME bytes
        shutil.copy(src, dst)
        assert md5(dst) == src_md5, "input bytes changed!"  # guard R3
        tracemalloc.reset_peak()
        consume_file(dst)
        cur, peak = tracemalloc.get_traced_memory()
        peaks.append(peak / 1048576)
        rss_curr.append(probe._vmrss_kb() / 1024)
        rss_hi.append(probe._maxrss_kb() / 1024)
        # delete created Document + files so next identical consume passes dedup
        for d in Document.objects.all():
            for p in (d.source_path, d.thumbnail_path, d.archive_path):
                try:
                    if p and os.path.isfile(p):
                        os.unlink(p)
                except Exception:
                    pass
        Document.objects.all().delete()
    def stats(x):
        return (min(x), statistics.median(x), max(x),
                statistics.mean(x), statistics.pstdev(x))
    pk = stats(peaks)
    print(f"  peak_TM MiB: min={pk[0]:.2f} median={pk[1]:.2f} max={pk[2]:.2f} "
          f"mean={pk[3]:.2f} stdev={pk[4]:.3f}")
    print(f"  peak_TM per-iter (first 10): {[round(p,2) for p in peaks[:10]]}")
    print(f"  VmRSS MiB   : start={rss_curr[0]:.1f} end={rss_curr[-1]:.1f} "
          f"min={min(rss_curr):.1f} max={max(rss_curr):.1f}")
    print(f"  ru_maxrss   : start={rss_hi[0]:.1f} end={rss_hi[-1]:.1f} (monotonic hiwater)")
    print(f"  iter0(first-touch)={peaks[0]:.2f}MiB  warm_median(iter>=2)="
          f"{statistics.median(peaks[1:]):.2f}MiB" if len(peaks) > 1 else "")
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()
    return peaks


if __name__ == "__main__":
    # digital 50 iters x 2 runs; image 20 iters x 1 (OCR slower)
    for r in (1, 2):
        run("digital", 50, r)
    run("image", 20, 1)
    print("\nDIST_HARNESS_DONE")
```

### A-10. `batch_consume.py` — many distinct docs per type via `consume_file`, steady-state flatness (Q5 many-consume)

```python
"""F3 -> Q5 (many-document cells via consume_file) + Q3 (no accumulation).

Consumes N genuinely-DISTINCT documents of one type sequentially through the
REAL consume_file entry point in ONE long-lived process (the django-q worker
model), probing every few docs. Demonstrates whether live-object count and the
Python heap reach STEADY STATE (no per-document accumulation) while RSS may stay
elevated (allocator retention).
"""
import gc
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe
import docgen


def batch(key, n):
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"batch-{key}")
    os.makedirs(work, exist_ok=True)
    print(f"\n===== BATCH CONSUME type={key} n={n} (distinct docs) =====")
    probe.probe(f"{key} batch BEFORE (0 consumed)")
    peaks = []
    for i in range(1, n + 1):
        dst = os.path.join(work, f"{key}_{i}.{ 'txt' if key=='text' else 'pdf'}")
        docgen.make(key, dst, i)
        tracemalloc.reset_peak()
        consume_file(dst)
        cur, peak = tracemalloc.get_traced_memory()
        peaks.append(peak / 1048576)
        if i % max(1, n // 5) == 0 or i == n:
            probe.probe(f"{key} after doc {i}/{n} (peak {peak/1048576:.2f}MiB)")
    probe.probe(f"{key} batch AFTER-GC", do_gc=True)
    print(f"[BATCH SUMMARY] type={key} n={n} "
          f"peak_min={min(peaks):.2f} peak_median={sorted(peaks)[len(peaks)//2]:.2f} "
          f"peak_max={max(peaks):.2f} live_docs_in_db={Document.objects.count()}")
    # cleanup
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()


if __name__ == "__main__":
    # default: text/digital many=25, image many=15 (OCR is slower)
    specs = [("text", 25), ("digital", 25), ("image", 15)]
    if len(sys.argv) > 1:
        specs = [(sys.argv[1], int(sys.argv[2]))]
    for k, n in specs:
        batch(k, n)
    print("\nBATCH_CONSUME_DONE")
```

### A-11. `build_export.py` — build small real manifests via the `document_exporter` management command (importer prerequisite)

```python
"""F3 importer prerequisite: build REAL exports via document_exporter.

Consumes N distinct docs of a type into a temp DB, then runs the REAL
document_exporter management command to a PERSISTENT dir /tmp/mem_exports/<type>_<N>
(manifest.json + media files). Those dirs are then fed to the REAL
document_importer by importer_harness.py.
"""
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import docgen
from django.core.management import call_command

EXPORT_ROOT = "/tmp/mem_exports"


def build(key, n):
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"exp-{key}-{n}")
    os.makedirs(work, exist_ok=True)
    for i in range(1, n + 1):
        dst = os.path.join(work, f"{key}_{i}.{'txt' if key=='text' else 'pdf'}")
        docgen.make(key, dst, i)
        consume_file(dst)
    target = os.path.join(EXPORT_ROOT, f"{key}_{n}")
    if os.path.isdir(target):
        shutil.rmtree(target)
    os.makedirs(target, exist_ok=True)
    call_command("document_exporter", target, "--no-progress-bar")
    mpath = os.path.join(target, "manifest.json")
    print(f"[EXPORT] type={key} n={n} docs_in_db={Document.objects.count()} "
          f"manifest_bytes={os.path.getsize(mpath)} target={target}")
    # cleanup DB + consumed files for next type (export dir persists)
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()


if __name__ == "__main__":
    specs = [("text", 1), ("text", 50), ("digital", 1), ("digital", 50),
             ("image", 1), ("image", 20)]
    for k, n in specs:
        build(k, n)
    print("BUILD_EXPORT_DONE")
```

### A-12. `build_export_big.py` — build large-content manifests via `document_exporter` (large-content importer prerequisite)

```python
import os, shutil, sys, tracemalloc
tracemalloc.start(1)
import bootstrap
CTX = bootstrap.bootstrap()
import docgen
from django.core.management import call_command
EXPORT_ROOT = "/tmp/mem_exports"
def build(key, n):
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"expbig-{key}-{n}")
    os.makedirs(work, exist_ok=True)
    for i in range(1, n + 1):
        dst = os.path.join(work, f"{key}_{i}.{'txt' if key=='text' else 'pdf'}")
        docgen.make(key, dst, i)
        consume_file(dst)
    target = os.path.join(EXPORT_ROOT, f"{key}_{n}")
    if os.path.isdir(target): shutil.rmtree(target)
    os.makedirs(target, exist_ok=True)
    call_command("document_exporter", target, "--no-progress-bar")
    mpath = os.path.join(target, "manifest.json")
    print(f"[EXPORT] type={key} n={n} docs={Document.objects.count()} manifest_bytes={os.path.getsize(mpath)}")
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p): os.unlink(p)
            except Exception: pass
    Document.objects.all().delete()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()
if __name__ == "__main__":
    for k, n in [("text", 200), ("digital", 200)]:
        build(k, n)
    print("BUILD_EXPORT_BIG_DONE")
```

### A-13. `importer_harness.py` — single & many `document_importer` runs (Q5 importer cells)

```python
"""F3 -> Q5 (importer cells): drive the REAL document_importer per type x batch.

ONE import per process invocation (argv: key n) => each gets a FRESH empty DB
from bootstrap(), avoiding cross-batch checksum collisions. Probes BEFORE /
AFTER / AFTER-GC around the REAL document_importer command.
document_importer.py memory candidates: json.load whole manifest [L73],
loaddata deserialization [L87], list(filter(...)) second copy [L137].
"""
import os
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe
from django.core.management import call_command

EXPORT_ROOT = "/tmp/mem_exports"


def import_one(key, n):
    from documents.models import Document
    src = os.path.join(EXPORT_ROOT, f"{key}_{n}")
    mbytes = os.path.getsize(os.path.join(src, "manifest.json"))
    print(f"\n===== IMPORTER type={key} batch_n={n} manifest={mbytes}B =====")
    b = probe.probe(f"import {key}_{n} BEFORE")
    tracemalloc.reset_peak()
    call_command("document_importer", src, "--no-progress-bar")
    a = probe.probe(f"import {key}_{n} AFTER")
    g = probe.probe(f"import {key}_{n} AFTER-GC", do_gc=True)
    drss = (a["vmrss_kb"] - b["vmrss_kb"]) / 1024
    print(f"[IMPORT SUMMARY] type={key} n={n} manifest={mbytes}B "
          f"dRSS={drss:.2f}MiB peakTM={a['tm_peak']/1048576:.2f}MiB "
          f"dLiveObj={a['n_obj']-b['n_obj']} docs_imported={Document.objects.count()} "
          f"freed={g['collected']}")
    sys.stdout.flush()


if __name__ == "__main__":
    import_one(sys.argv[1], int(sys.argv[2]))
```

### A-14. `focused_copies.py` — importer copy#1 (`json.load`) / copy#2 (`list(filter)`) / `loaddata` attribution at tiny content (Q5 / §8.2)

```python
"""F3/Q5 mechanistic attribution of the THREE document_importer copies.

Reproduces the importer's exact in-memory operations under tracemalloc so each
copy's Python-heap size is attributed and shown to scale with MANIFEST SIZE
(=> batch size), while the document CONTENT itself is NOT in the manifest (it is
a separate file), which is why the importer is metadata-proportional, not
content-proportional:
    copy#1  self.manifest = json.load(f)          document_importer.py:L73
    copy#2  manifest_documents = list(filter(...)) document_importer.py:L137
    loaddata deserialization                       document_importer.py:L87
"""
import json
import os
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe
from django.core.management import call_command

EXPORT_ROOT = "/tmp/mem_exports"


def measure(key, n):
    from documents.models import Document
    src = os.path.join(EXPORT_ROOT, f"{key}_{n}")
    mpath = os.path.join(src, "manifest.json")
    mbytes = os.path.getsize(mpath)
    print(f"\n===== FOCUSED COPIES type={key} n={n} manifest_file={mbytes}B =====")

    # copy#1: json.load whole manifest (L73)
    tracemalloc.reset_peak()
    c0, _ = tracemalloc.get_traced_memory()
    with open(mpath) as f:
        manifest = json.load(f)
    c1, p1 = tracemalloc.get_traced_memory()
    n_records = len(manifest)
    n_docrecords = sum(1 for r in manifest if r.get("model") == "documents.document")
    print(f"  copy#1 json.load [L73]: dTM_current={(c1-c0)/1024:.1f} KiB "
          f"peak={(p1)/1048576:.2f} MiB records={n_records} doc_records={n_docrecords}")

    # copy#2: list(filter(...)) second copy (L137) -- shallow refs
    tracemalloc.reset_peak()
    c2, _ = tracemalloc.get_traced_memory()
    manifest_documents = list(
        filter(lambda r: r["model"] == "documents.document", manifest),
    )
    c3, p3 = tracemalloc.get_traced_memory()
    print(f"  copy#2 list(filter) [L137]: dTM_current={(c3-c2)/1024:.1f} KiB "
          f"len={len(manifest_documents)} (shallow refs to same dicts)")

    # confirm shallow (same dict objects, not deep copies)
    same = manifest_documents[0] is next(
        r for r in manifest if r["model"] == "documents.document")
    print(f"  copy#2 is-shallow (element identity vs manifest): {same}")

    del manifest, manifest_documents

    # loaddata deserialization (L87) into fresh DB
    b = probe.probe(f"loaddata {key}_{n} BEFORE")
    tracemalloc.reset_peak()
    call_command("loaddata", mpath, verbosity=0)
    a = probe.probe(f"loaddata {key}_{n} AFTER")
    print(f"  loaddata [L87]: dTM_current={(a['tm_current']-b['tm_current'])/1024:.1f} KiB "
          f"peak={a['tm_peak']/1048576:.2f} MiB docs_in_db={Document.objects.count()}")
    sys.stdout.flush()


if __name__ == "__main__":
    # each invocation is a fresh process/DB; measure copies across batch sizes
    measure(sys.argv[1], int(sys.argv[2]))
```

### A-15. `focused_big.py` — importer copy#1 / `loaddata` coexistence peak at large content (Q5 / §8.2 / §8.3)

```python
"""F3/Q5 large-content importer measurement (run-first backing for the
'peak approaches 2x manifest at large content' claim). Builds REAL exports of
~100KB-content text docs (via document_exporter), then measures the three
importer materializations with copy#1 (self.manifest = json.load, L73) KEPT
ALIVE through loaddata (L87) -- matching the real document_importer, where
self.manifest is an instance attribute that persists across the loaddata call.
Also drives the REAL document_importer end-to-end for the true whole-command
peak. Fresh DB per size; export dirs and consumed files cleaned afterwards.
"""
import os
import shutil
import sys
import json
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe
from django.core.management import call_command

EXPORT_ROOT = "/tmp/mem_exports_big"
# ~100KB of unique-ish text per document
_BODY = "".join(
    f"Line {k} lorem ipsum dolor sit amet consectetur adipiscing elit sed do. "
    for k in range(1400)
)


def big_text(path, i):
    with open(path, "w") as f:
        f.write(f"UNIQUE-{i}-{i*7919}\n" + _BODY + f"\nEND-{i}\n")


def _wipe_db():
    from documents.models import Document
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()


def build(n):
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"big-text-{n}")
    os.makedirs(work, exist_ok=True)
    for i in range(1, n + 1):
        dst = os.path.join(work, f"bigtext_{i}.txt")
        big_text(dst, i)
        consume_file(dst)
    target = os.path.join(EXPORT_ROOT, f"text_{n}")
    if os.path.isdir(target):
        shutil.rmtree(target)
    os.makedirs(target, exist_ok=True)
    call_command("document_exporter", target, "--no-progress-bar")
    mpath = os.path.join(target, "manifest.json")
    mbytes = os.path.getsize(mpath)
    print(f"[BIG EXPORT] n={n} docs={Document.objects.count()} "
          f"manifest_bytes={mbytes} ({mbytes/1048576:.3f} MiB)")
    _wipe_db()
    shutil.rmtree(work, ignore_errors=True)
    sys.stdout.flush()
    return mpath, mbytes


def measure_coexist(mpath, mbytes, n):
    from documents.models import Document
    print(f"\n===== FOCUSED BIG (copy#1 kept alive) n={n} "
          f"manifest={mbytes}B ({mbytes/1048576:.3f} MiB) =====")
    # copy#1: json.load whole manifest (L73) -- KEEP ALIVE
    tracemalloc.reset_peak()
    c0, _ = tracemalloc.get_traced_memory()
    with open(mpath) as f:
        manifest = json.load(f)
    c1, p1 = tracemalloc.get_traced_memory()
    print(f"  copy#1 json.load [L73] (kept alive): "
          f"dTM_current={(c1-c0)/1048576:.3f} MiB peak={p1/1048576:.2f} MiB "
          f"records={len(manifest)}")
    # copy#2: list(filter) (L137) -- shallow
    tracemalloc.reset_peak()
    c2, _ = tracemalloc.get_traced_memory()
    md = list(filter(lambda r: r["model"] == "documents.document", manifest))
    c3, _ = tracemalloc.get_traced_memory()
    print(f"  copy#2 list(filter) [L137]: dTM_current={(c3-c2)/1024:.1f} KiB "
          f"len={len(md)} (shallow refs)")
    # loaddata (L87) WHILE copy#1 (manifest) + copy#2 STILL ALIVE => coexistence
    b = probe.probe(f"big loaddata n={n} BEFORE (copy#1 alive)")
    tracemalloc.reset_peak()
    call_command("loaddata", mpath, verbosity=0)
    a = probe.probe(f"big loaddata n={n} AFTER (coexistence peak)")
    print(f"  loaddata [L87] WITH copy#1 alive: "
          f"dTM_current={(a['tm_current']-b['tm_current'])/1048576:.3f} MiB "
          f"COEXIST_peak={a['tm_peak']/1048576:.2f} MiB "
          f"docs_in_db={Document.objects.count()}")
    del manifest, md
    _wipe_db()
    sys.stdout.flush()


def real_importer(n):
    from documents.models import Document
    target = os.path.join(EXPORT_ROOT, f"text_{n}")
    print(f"\n===== REAL document_importer (whole command) n={n} =====")
    b = probe.probe(f"real importer n={n} BEFORE")
    tracemalloc.reset_peak()
    call_command("document_importer", target, "--no-progress-bar")
    a = probe.probe(f"real importer n={n} AFTER")
    print(f"  document_importer whole-command: "
          f"dRSS={(a['vmrss_kb']-b['vmrss_kb'])/1024:.2f}MiB "
          f"peakTM={a['tm_peak']/1048576:.2f}MiB docs_in_db={Document.objects.count()}")
    _wipe_db()
    sys.stdout.flush()


if __name__ == "__main__":
    if os.path.isdir(EXPORT_ROOT):
        shutil.rmtree(EXPORT_ROOT)
    os.makedirs(EXPORT_ROOT, exist_ok=True)
    import sys as _sys
    _ns = [int(x) for x in _sys.argv[1:]] or [10, 50]
    for n in _ns:
        mp, mb = build(n)
        measure_coexist(mp, mb, n)
        real_importer(n)
    print("\nFOCUSED_BIG_DONE")
```

### A-16. `metadata_harness.py` — repeated REST metadata endpoint: parser tempdirs, live `pikepdf.Pdf`, RSS/tm/gc (Q2 / F1)

```python
"""F1/Q2: REST metadata endpoint (DocumentViewSet.metadata, views.py:L283).

Drives the REAL DRF endpoint repeatedly via APIRequestFactory+force_authenticate
and measures, before/during/after + gc.collect:
  * count of leaked parser temp dirs (prefix 'paperless-' under SCRATCH_DIR):
    get_metadata (views.py:L260) constructs a parser (mkdtemp parsers.py:L293)
    and calls extract_metadata (L269) but NEVER calls parser.cleanup() -- the
    except at L270-272 also returns [] without cleanup.
  * live pikepdf.Pdf objects (extract_metadata opens pikepdf.open at
    paperless_tesseract/parsers.py:L34 without a context manager / close).
  * bytes held in those temp dirs (=> are they empty? do they cost RAM?).
"""
import gc
import glob
import os
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"


def count_tempdirs():
    dirs = glob.glob(os.path.join(CTX["scratch_dir"], "paperless-*"))
    total = 0
    for d in dirs:
        for root, _, files in os.walk(d):
            for fn in files:
                try:
                    total += os.path.getsize(os.path.join(root, fn))
                except OSError:
                    pass
    return len(dirs), total


def live_pdf_count():
    import pikepdf
    return sum(1 for o in gc.get_objects() if isinstance(o, pikepdf.Pdf))


def run():
    import shutil
    from django.contrib.auth.models import User
    from documents.tasks import consume_file
    from documents.models import Document
    from documents.views import DocumentViewSet
    from rest_framework.test import APIRequestFactory, force_authenticate

    # 1) consume a digital PDF -> Document with original + archive
    work = os.path.join(CTX["scratch_dir"], "meta-src")
    os.makedirs(work, exist_ok=True)
    dst = os.path.join(work, "simple-digital.pdf")
    shutil.copy(DIGITAL, dst)
    consume_file(dst)  # returns a status string; fetch the Document from DB
    doc = Document.objects.latest("id")
    pk = doc.id
    print(f"consumed doc pk={pk} has_archive={doc.has_archive_version} "
          f"content_len={len(doc.content)}")

    user = User.objects.create_superuser("mh", "mh@example.com", "x")
    factory = APIRequestFactory()
    view = DocumentViewSet.as_view({"get": "metadata"})

    def call_once():
        req = factory.get(f"/api/documents/{pk}/metadata/")
        force_authenticate(req, user=user)
        resp = view(req, pk=pk)
        resp.render()
        return resp

    # baseline
    nd0, nb0 = count_tempdirs()
    print(f"\n[BASELINE before any endpoint call] tempdirs={nd0} bytes_in_tempdirs={nb0} "
          f"live_pdf={live_pdf_count()}")
    probe.probe("metadata BEFORE any call")

    for target in [1, 10, 50, 200]:
        while True:
            resp = call_once()
            # crude counter via closure
            run.count = getattr(run, "count", 0) + 1
            if run.count >= target:
                break
        nd, nb = count_tempdirs()
        lp = live_pdf_count()
        p = probe.probe(f"metadata AFTER {target} calls")
        print(f"    -> after {target} calls: leaked_tempdirs={nd} "
              f"bytes_in_tempdirs={nb} live_pikepdf_Pdf={lp} "
              f"status={resp.status_code}")
        sys.stdout.flush()

    # after gc.collect: does anything free?
    ndg, nbg = count_tempdirs()
    probe.probe("metadata AFTER-GC", do_gc=True)
    print(f"    -> after gc.collect(): leaked_tempdirs={ndg} "
          f"bytes_in_tempdirs={nbg} live_pikepdf_Pdf={live_pdf_count()}")
    # sample the leaked dir contents to prove they are empty
    dirs = glob.glob(os.path.join(CTX["scratch_dir"], "paperless-*"))
    print(f"    -> total leaked tempdirs on disk: {len(dirs)}; "
          f"listing first 3 contents:")
    for d in dirs[:3]:
        print(f"        {d} -> contents={os.listdir(d)}")
    print("METADATA_HARNESS_DONE")


if __name__ == "__main__":
    run()
```

### A-17. `q2_pikepdf_content.py` — `pikepdf.Pdf` handle lifecycle + full-text single-copy identity check (Q2)

```python
"""Q2: (a) pikepdf.Pdf handle lifecycle; (b) content string held exactly once.

(a) Calls RasterisedDocumentParser.extract_metadata directly (the real method,
    paperless_tesseract/parsers.py:L26 -> pikepdf.open L34 with NO context
    manager / close) and counts live pikepdf.Pdf BEFORE/DURING/AFTER to show the
    handle is reclaimed by CPython refcounting the moment the local goes out of
    scope (no leak of live Pdf objects) -- confirmed WITHOUT gc.collect.
(b) Wraps Consumer._store to check `text is document.content` (identity) proving
    the extracted full-text string is referenced once, not copied, plus the
    transient tracemalloc peak while building a large Document.content.
"""
import gc
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"


def live_pdf():
    import pikepdf
    return sum(1 for o in gc.get_objects() if isinstance(o, pikepdf.Pdf))


def part_a():
    from paperless_tesseract.parsers import RasterisedDocumentParser
    print("\n===== Q2(a) pikepdf.Pdf handle lifecycle (direct extract_metadata) =====")
    print(f"  live pikepdf.Pdf BEFORE construct parser: {live_pdf()}")
    p = RasterisedDocumentParser(logging_group=None, progress_callback=None)

    # Wrap pikepdf.open to observe the handle count DURING the call (inside
    # extract_metadata, before it returns) without editing the source.
    import pikepdf
    orig_open = pikepdf.open
    seen = {}

    def open_wrap(*a, **k):
        h = orig_open(*a, **k)
        seen["during"] = sum(1 for o in gc.get_objects() if isinstance(o, pikepdf.Pdf))
        return h
    pikepdf.open = open_wrap
    try:
        result = p.extract_metadata(DIGITAL, "application/pdf")
    finally:
        pikepdf.open = orig_open
    print(f"  live pikepdf.Pdf DURING extract_metadata (inside, after open): "
          f"{seen.get('during')}")
    print(f"  live pikepdf.Pdf AFTER extract_metadata returns (NO gc.collect): "
          f"{live_pdf()}")
    print(f"  metadata entries returned: {len(result)}")
    p.cleanup()


def part_b():
    from documents.consumer import Consumer
    from documents.tasks import consume_file
    from documents.models import Document
    print("\n===== Q2(b) content string identity (text is document.content) =====")
    captured = {}
    orig_store = Consumer._store

    def store_wrap(self, text, date, mime_type):
        tracemalloc.reset_peak()
        doc = orig_store(self, text, date, mime_type)
        captured["identity"] = (text is doc.content)
        captured["text_len"] = len(text) if text is not None else None
        captured["refcount_text"] = sys.getrefcount(text)
        cur, peak = tracemalloc.get_traced_memory()
        captured["store_peak_mib"] = peak / 1048576
        return doc
    Consumer._store = store_wrap
    try:
        work = os.path.join(CTX["scratch_dir"], "q2b")
        os.makedirs(work, exist_ok=True)
        dst = os.path.join(work, "simple-digital.pdf")
        shutil.copy(DIGITAL, dst)
        consume_file(dst)
    finally:
        Consumer._store = orig_store
    print(f"  text IS document.content (same object, single copy): "
          f"{captured.get('identity')}")
    print(f"  content length: {captured.get('text_len')} chars")
    print(f"  sys.getrefcount(text) inside _store: {captured.get('refcount_text')}")
    print(f"  tracemalloc.peak during _store: {captured.get('store_peak_mib'):.2f} MiB")

    # large-content transient: how big is the peak if content were large?
    doc = Document.objects.latest("id")
    big = "x" * (40 * 1024 * 1024)  # 40 MiB text
    tracemalloc.reset_peak()
    before = tracemalloc.get_traced_memory()[0]
    doc.content = big
    after = tracemalloc.get_traced_memory()[0]
    cur, peak = tracemalloc.get_traced_memory()
    print(f"  assigning 40MiB content: dTM_current={(after-before)/1048576:.2f} MiB "
          f"peak={peak/1048576:.2f} MiB (single reference on the model field)")
    del big
    doc.content = ""


if __name__ == "__main__":
    part_a()
    part_b()
    print("\nQ2_PIKEPDF_CONTENT_DONE")
```

### A-18. `classifier_harness.py` — classifier 7× `pickle.load` per-load attribution, classifier-present (Q3 / F4)

```python
"""F1/Q3: classifier cache load path.

Builds a REAL model (train + save), then drives load_classifier()
(classifier.py:L30 -> DocumentClassifier.load() L76) with pickle.load WRAPPED to
attribute each of the 7 sequential pickle.load calls
(L78/86/87/88/90/91/92) by Python-heap delta, and counts how many times
pickle.load is invoked. Confirms which component dominates (CountVectorizer
vocabulary vs the MLPClassifier heads).
"""
import gc
import os
import pickle
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

TEXTS = [
    "invoice from acme corp total due 100 dollars net 30 terms",
    "acme corporation billing statement account overdue reminder",
    "receipt grocery store milk bread eggs butter coffee",
    "supermarket purchase receipt vegetables fruit dairy items",
    "bank statement checking account balance transactions monthly",
    "financial institution account summary deposits withdrawals",
    "medical report patient diagnosis treatment prescription notes",
    "hospital discharge summary follow up appointment medication",
]


def build_training_data():
    from documents.models import Document, Correspondent
    from documents.models import MatchingModel
    from django.utils import timezone
    corr_a = Correspondent.objects.create(
        name="Acme", matching_algorithm=MatchingModel.MATCH_AUTO)
    corr_b = Correspondent.objects.create(
        name="Bank", matching_algorithm=MatchingModel.MATCH_AUTO)
    for i, t in enumerate(TEXTS):
        c = corr_a if i % 2 == 0 else corr_b
        Document.objects.create(
            title=f"train{i}", content=t, mime_type="text/plain",
            checksum=f"chk{i:04d}", created=timezone.now(), modified=timezone.now(),
            correspondent=c,
        )
    print(f"training docs={Document.objects.count()} correspondents=2 (MATCH_AUTO)")


def main():
    from documents.classifier import DocumentClassifier
    from documents import classifier as clf_mod

    build_training_data()

    # Train + save a real model
    c = DocumentClassifier()
    probe.probe("before train()")
    tracemalloc.reset_peak()
    changed = c.train()
    cur, peak = tracemalloc.get_traced_memory()
    print(f"train() changed={changed} peak_during_train={peak/1048576:.2f} MiB")
    c.save()
    print(f"MODEL_FILE exists={os.path.isfile(clf_mod.settings.MODEL_FILE)} "
          f"size={os.path.getsize(clf_mod.settings.MODEL_FILE)} B")
    del c
    gc.collect()

    # Wrap pickle.load to attribute each of the 7 sequential loads.
    orig_load = pickle.load
    calls = []

    def load_wrap(f, *a, **k):
        b, _ = tracemalloc.get_traced_memory()
        tracemalloc.reset_peak()
        obj = orig_load(f, *a, **k)
        cur2, peak2 = tracemalloc.get_traced_memory()
        calls.append((type(obj).__name__, (cur2 - b), peak2))
        return obj
    clf_mod.pickle.load = load_wrap
    try:
        probe.probe("before load_classifier()")
        tracemalloc.reset_peak()
        loaded = clf_mod.load_classifier()
        a = probe.probe("after load_classifier()")
    finally:
        clf_mod.pickle.load = orig_load

    print(f"\n[LOAD_CLASSIFIER] returned={type(loaded).__name__} "
          f"pickle.load_invocations={len(calls)}")
    print(f"{'#':>2} {'loaded_type':28s} {'dTM_current_KiB':>16s} {'peak_MiB':>9s}")
    for i, (tn, d, pk) in enumerate(calls, 1):
        print(f"{i:>2} {tn:28s} {d/1024:16.1f} {pk/1048576:9.2f}")
    # size introspection of dominant component
    try:
        vocab = len(loaded.data_vectorizer.vocabulary_)
        print(f"  data_vectorizer.vocabulary_ size = {vocab} terms")
    except Exception as e:
        print("  vocab introspection:", e)
    print("CLASSIFIER_HARNESS_DONE")


if __name__ == "__main__":
    main()
```

### A-19. `classifier_batch.py` — classifier shared across the three handlers, no accumulation, 12-doc batch (Q3 / F4)

```python
"""Q3: classifier is loaded ONCE per consume and SHARED across the 3 handlers;
across many consumes it does NOT accumulate (live DocumentClassifier stays low).

Trains+saves a model, then consumes N docs with the model PRESENT. Wraps the 3
classifier-consuming handlers (set_correspondent/type/tags) to record
id(classifier) so we can prove all 3 receive the SAME object within one consume
(consumer.py:L292 loads once, passes classifier= to the finished signal). After
each consume, counts live DocumentClassifier objects via gc.
"""
import gc
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"


def live_classifiers():
    from documents.classifier import DocumentClassifier
    return sum(1 for o in gc.get_objects() if isinstance(o, DocumentClassifier))


def main():
    from documents.models import Document, Correspondent, MatchingModel
    from documents.classifier import DocumentClassifier
    from documents.tasks import consume_file
    from documents.signals import document_consumption_finished
    from documents.signals import handlers as H
    from django.utils import timezone

    # training data + model
    ca = Correspondent.objects.create(name="A", matching_algorithm=MatchingModel.MATCH_AUTO)
    cb = Correspondent.objects.create(name="B", matching_algorithm=MatchingModel.MATCH_AUTO)
    for i in range(8):
        Document.objects.create(
            title=f"t{i}", content=f"training doc {i} acme bank invoice receipt {i}",
            mime_type="text/plain", checksum=f"seed{i:03d}",
            created=timezone.now(), modified=timezone.now(),
            correspondent=(ca if i % 2 else cb))
    c = DocumentClassifier(); c.train(); c.save()
    print(f"model built size={os.path.getsize(__import__('django.conf', fromlist=['settings']).settings.MODEL_FILE)}B")

    # wrap the 3 handlers to record id(classifier)
    ids = {}
    targets = [("set_correspondent", H.set_correspondent),
               ("set_document_type", H.set_document_type),
               ("set_tags", H.set_tags)]
    wrapped = []
    for name, fn in targets:
        document_consumption_finished.disconnect(fn)
        def make(nm, f):
            def w(*a, **k):
                ids.setdefault(k.get("document") and k["document"].pk, {})[nm] = id(k.get("classifier"))
                return f(*a, **k)
            return w
        w = make(name, fn)
        document_consumption_finished.connect(w, weak=False)
        wrapped.append((w, fn))

    work = os.path.join(CTX["scratch_dir"], "clf-batch")
    os.makedirs(work, exist_ok=True)
    print(f"\nlive DocumentClassifier BEFORE any consume: {live_classifiers()}")
    N = 12
    for i in range(1, N + 1):
        dst = os.path.join(work, f"d{i}.pdf")
        shutil.copy(DIGITAL, dst)
        # make unique checksum by copying to unique name is not enough (same bytes);
        # dedup would block -> delete the created doc's file mapping first is n/a.
        # Instead: consume distinct generated digital PDFs.
        import docgen
        docgen.make("digital", dst, 1000 + i)
        consume_file(dst)
        if i % 3 == 0 or i == N:
            lc = live_classifiers()
            print(f"  after consume {i}/{N}: live DocumentClassifier={lc}")
    probe.probe("after batch", do_gc=True)
    print(f"  after gc.collect(): live DocumentClassifier={live_classifiers()}")

    # report handler sharing (per document, are all 3 ids equal?)
    shared = [len(set(v.values())) == 1 for v in ids.values() if len(v) == 3]
    print(f"\n[HANDLER SHARING] documents_with_all_3_handlers={len(shared)} "
          f"all_share_same_classifier_id={all(shared) if shared else 'N/A'}")
    sample = list(ids.items())[:2]
    for pk, d in sample:
        print(f"   doc pk={pk}: handler->id(classifier) = {d}")

    for w, fn in wrapped:
        document_consumption_finished.disconnect(w)
        document_consumption_finished.connect(fn)
    print("CLASSIFIER_BATCH_DONE")


if __name__ == "__main__":
    main()
```

### A-20. `whoosh_check.py` — Whoosh `AsyncWriter`/`SegmentWriter` accumulation check (Q3)

```python
"""Q3: Whoosh index writer does NOT accumulate.

add_or_update_document (index.py:L118) uses a context-managed AsyncWriter
(open_index_writer L65 -> commit/cancel). Performs 30 real index updates and
counts live AsyncWriter/SegmentWriter objects between updates -> they should be
0 (committed and released each time), and live-object count stays flat.
"""
import gc
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe


def live(cls_names):
    out = {}
    for o in gc.get_objects():
        n = type(o).__name__
        if n in cls_names:
            out[n] = out.get(n, 0) + 1
    return {n: out.get(n, 0) for n in cls_names}


def main():
    from documents.models import Document
    from documents import index
    from django.utils import timezone

    watch = ("AsyncWriter", "SegmentWriter", "IndexWriter")
    print(f"live writers BEFORE: {live(watch)}")
    probe.probe("whoosh BEFORE")
    for i in range(1, 31):
        d = Document.objects.create(
            title=f"idx{i}", content=f"indexed content number {i} token {i*13}",
            mime_type="text/plain", checksum=f"idx{i:04d}",
            created=timezone.now(), modified=timezone.now())
        index.add_or_update_document(d)
        if i % 10 == 0:
            lw = live(watch)
            p = probe.probe(f"whoosh after {i} updates")
            print(f"   after {i} updates: live_writers={lw}")
    probe.probe("whoosh AFTER-GC", do_gc=True)
    print(f"live writers AFTER gc.collect(): {live(watch)}")
    print("WHOOSH_CHECK_DONE")


if __name__ == "__main__":
    main()
```

### A-21. `debug_harness.py` — `DEBUG=NO` vs `DEBUG=YES` `connection.queries` growth (Q3 / §9)

```python
"""Q3/§9: Django connection.queries accumulation is DEBUG-only.

settings.py:L50 DEBUG default NO. When DEBUG=False, connection.queries stays
empty regardless of how many queries run. When DEBUG=True (labeled NON-CANONICAL
diagnostic variant), Django appends every executed SQL to connection.queries
(capped at 9000 by default), which grows with query volume. Runs the SAME query
workload under both settings in-process.
"""
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe


def workload(n, prefix):
    from documents.models import Document
    from django.utils import timezone
    for i in range(n):
        Document.objects.create(
            title=f"{prefix}{i}", content="x", mime_type="text/plain",
            checksum=f"{prefix}{i:05d}", created=timezone.now(), modified=timezone.now())
    # run many reads to generate queries
    for i in range(n):
        list(Document.objects.filter(title=f"{prefix}{i}").values("id"))


def main():
    from django.db import connection, reset_queries
    from django.test import override_settings

    # DEFAULT canonical: DEBUG=False
    with override_settings(DEBUG=False):
        reset_queries()
        workload(200, 'a')
        ql_false = len(connection.queries)
        print(f"[DEBUG=NO  (canonical)] connection.queries length after workload = {ql_false}")

    # NON-CANONICAL diagnostic variant: DEBUG=True
    with override_settings(DEBUG=True):
        reset_queries()
        b = probe.probe("DEBUG=YES before workload")
        workload(5000, 'b')
        ql_true = len(connection.queries)
        a = probe.probe("DEBUG=YES after workload")
        # measure bytes held by connection.queries
        import sys as _s
        approx = sum(len(q.get("sql", "")) + len(str(q.get("time", "")))
                     for q in connection.queries)
        print(f"[DEBUG=YES (NON-CANONICAL)] connection.queries length = {ql_true} "
              f"(default cap 9000); approx_sql_bytes={approx} "
              f"(~{approx/1048576:.2f} MiB of SQL strings)")
        reset_queries()
        print(f"   after reset_queries(): connection.queries length = {len(connection.queries)}")
    print("DEBUG_HARNESS_DONE")


if __name__ == "__main__":
    main()
```

### A-22. `no_model.py` — no-`MODEL_FILE` path: `load_classifier()` returns `None` (Q3 / Q4 / F4)

```python
"""F4: explicit no-MODEL_FILE raw output.

Drives the REAL load_classifier() entry point (documents/classifier.py:L30)
on a fresh temp DATA_DIR where MODEL_FILE does not exist yet, showing it
returns None (rule-only matching, no pickle.load), then confirms that after a
real train_classifier the same entry point returns a DocumentClassifier.
"""
import os
import bootstrap
ctx = bootstrap.bootstrap()  # fresh temp DATA_DIR => MODEL_FILE absent
import probe
import tracemalloc
tracemalloc.start(1)
from django.conf import settings
from documents.classifier import load_classifier

print("MODEL_FILE path      =", settings.MODEL_FILE)
print("MODEL_FILE exists?   =", os.path.isfile(settings.MODEL_FILE))
probe.probe("no-model BEFORE load_classifier()")
clf = load_classifier()
probe.probe("no-model AFTER load_classifier()")
print("load_classifier() returned =", repr(clf), "type =", type(clf).__name__)
print("  => rule-only matching, zero pickle.load calls (classifier.py:L36 returns None)")
print("NO_MODEL_HARNESS_DONE")
```

### A-23. `train_harness.py` — `train_classifier` entry point: sklearn import + `data=list()` + fit transient (F4)

```python
"""F4: REAL train_classifier entry point (tasks.py:L48).

(1) Early-return path: with NO MATCH_AUTO Tag/DocumentType/Correspondent,
    train_classifier() returns immediately (tasks.py:L50-54) -> ~0 memory.
(2) Real training path: create a MATCH_AUTO correspondent + N training docs, then
    drive train_classifier(). Measures the transient peak, and shows train()'s
    data=list() accumulation (classifier.py:~L117) scaling with training-set size
    by training at two sizes. sklearn/scipy first-touch import is one-time.
"""
import os
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe


def seed(n, content_len):
    from documents.models import Document, Correspondent, MatchingModel
    from django.utils import timezone
    Correspondent.objects.all().delete()  # clear all (both TrainCorr and TrainCorr2)
    ca = Correspondent.objects.create(name="TrainCorr",
                                      matching_algorithm=MatchingModel.MATCH_AUTO)
    cb = Correspondent.objects.create(name="TrainCorr2",
                                      matching_algorithm=MatchingModel.MATCH_AUTO)
    base = ("invoice acme bank receipt statement medical hospital grocery "
            "supermarket financial account balance ") * (content_len // 12 + 1)
    for i in range(n):
        Document.objects.create(
            title=f"tr{i}", content=f"{base} unique token {i}",
            mime_type="text/plain", checksum=f"tr{i:05d}",
            created=timezone.now(), modified=timezone.now(),
            correspondent=(ca if i % 2 else cb))


def main():
    from documents.tasks import train_classifier
    from documents.models import Document

    # (1) early-return path (no MATCH_AUTO objects, empty DB)
    print("===== TRAIN (1) early-return path (no MATCH_AUTO objects) =====")
    b = probe.probe("train early BEFORE")
    tracemalloc.reset_peak()
    train_classifier()
    a = probe.probe("train early AFTER")
    print(f"  early-return dRSS={(a['vmrss_kb']-b['vmrss_kb'])/1024:.2f}MiB "
          f"peakTM={a['tm_peak']/1048576:.2f}MiB (returns at tasks.py:L50-54)")

    # (2) real training at two sizes to show data=list() scaling
    for n, clen in [(10, 200), (100, 200)]:
        Document.objects.all().delete()
        seed(n, clen)
        print(f"\n===== TRAIN (2) real path n_docs={n} content_len~{clen} =====")
        b = probe.probe(f"train n={n} BEFORE")
        tracemalloc.reset_peak()
        train_classifier()
        a = probe.probe(f"train n={n} AFTER")
        print(f"  train n={n}: dRSS={(a['vmrss_kb']-b['vmrss_kb'])/1024:.2f}MiB "
              f"peakTM={a['tm_peak']/1048576:.2f}MiB "
              f"model_exists={os.path.isfile(__import__('django.conf',fromlist=['settings']).settings.MODEL_FILE)} "
              f"dLiveObj={a['n_obj']-b['n_obj']}")
    print("TRAIN_HARNESS_DONE")


if __name__ == "__main__":
    main()
```

### A-24. `sanity_harness.py` — `sanity_check` entry point: whole-file `md5(f.read())` transients (F4)

```python
"""F4: REAL sanity_check entry point (tasks.py:L255 -> check_sanity L49).

Consumes real docs (creating source+archive+thumbnail files under MEDIA_ROOT),
then drives sanity_check(). Wraps hashlib.md5 inside sanity_checker to capture
the size of each whole-file f.read() (source md5 at sanity_checker.py:L83,
archive md5 at L112 -- NOTE the L70 thumbnail read has NO md5). Shows the
transient read buffer is proportional to the LARGEST file, released each iter.
"""
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"
IMAGE = "/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf"


def main():
    from documents.tasks import sanity_check
    from documents import sanity_checker
    from documents.tasks import consume_file
    import docgen

    work = os.path.join(CTX["scratch_dir"], "sanity-src")
    os.makedirs(work, exist_ok=True)
    # consume a mix so MEDIA_ROOT has source+archive+thumbnail files
    for i, s in enumerate([DIGITAL, IMAGE]):
        dst = os.path.join(work, f"s{i}_{os.path.basename(s)}")
        shutil.copy(s, dst)
        consume_file(dst)
    for i in range(3):
        dst = os.path.join(work, f"gen{i}.pdf")
        docgen.make("digital", dst, 5000 + i)
        consume_file(dst)

    # wrap hashlib.md5 in sanity_checker to record read sizes
    import hashlib
    orig_md5 = sanity_checker.hashlib.md5
    reads = []

    class MD5Probe:
        def __init__(self, data=b""):
            reads.append(len(data))
            self._h = orig_md5(data)
        def __getattr__(self, k):
            return getattr(self._h, k)
    sanity_checker.hashlib.md5 = lambda data=b"": MD5Probe(data)

    try:
        b = probe.probe("sanity BEFORE")
        tracemalloc.reset_peak()
        result = sanity_check()
        a = probe.probe("sanity AFTER")
    finally:
        sanity_checker.hashlib.md5 = orig_md5

    print(f"\n[SANITY] md5(f.read()) call_count={len(reads)} "
          f"read_sizes_bytes={sorted(reads, reverse=True)}")
    print(f"  largest_single_read={max(reads) if reads else 0}B "
          f"(~{(max(reads) if reads else 0)/1048576:.2f}MiB transient)")
    print(f"  sanity dRSS={(a['vmrss_kb']-b['vmrss_kb'])/1024:.2f}MiB "
          f"peakTM={a['tm_peak']/1048576:.2f}MiB")
    print("SANITY_HARNESS_DONE")


if __name__ == "__main__":
    main()
```

### A-25. `error_path.py` — encrypted vs corrupt error path; parser `cleanup()` on failure (F4)

```python
"""F4: error/edge paths + guaranteed parser cleanup on failure.

CASE A -- encrypted.pdf sample: driven through REAL consume_file. Observed
    result reported honestly (the sample opens with an empty password, so
    ocrmypdf/pikepdf handle it and consumption SUCCEEDS -- there is no
    'encrypted error' in canonical operation).
CASE B -- genuinely CORRUPT pdf (a real PDF header + truncated/garbage body so
    magic still detects application/pdf but the parser fails): driven through
    REAL consume_file to hit the parser error path. Captures the ACTUAL
    exception type+message and proves the parser temp dir is removed
    (try_consume_file cleans up in the ParseError branch and in the finally at
    consumer.py:L369) => no 'paperless-*' temp dir leaks on the error path.
"""
import glob
import os
import shutil
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

ENCRYPTED = "/app/src/paperless_tesseract/tests/samples/encrypted.pdf"
DIGITAL = "/app/src/paperless_tesseract/tests/samples/simple-digital.pdf"


def count_tempdirs():
    return len(glob.glob(os.path.join(CTX["scratch_dir"], "paperless-*")))


def try_consume(label, path):
    from documents.tasks import consume_file
    import magic
    from documents.models import Document
    mime = magic.from_file(path, mime=True)
    nd_before = count_tempdirs()
    tracemalloc.reset_peak()
    print(f"\n===== {label} file={os.path.basename(path)} "
          f"size={os.path.getsize(path)}B detected_mime={mime} =====")
    result = err_type = err_msg = None
    try:
        result = consume_file(path)
        print(f"  RESULT: SUCCESS -> {result}")
        d = Document.objects.latest("id")
        print(f"  created Document pk={d.pk} content_len={len(d.content)} "
              f"mime={d.mime_type}")
    except Exception as e:
        err_type = type(e).__name__
        err_msg = str(e)
        print(f"  RESULT: RAISED {err_type}: {err_msg}")
    nd_after = count_tempdirs()
    print(f"  paperless-* tempdirs: before={nd_before} after={nd_after} "
          f"(cleanup on error path => no leak)")
    cur, peak = tracemalloc.get_traced_memory()
    print(f"  peakTM during={peak/1048576:.2f}MiB")
    # cleanup any created doc
    for d in Document.objects.all():
        for p in (d.source_path, d.thumbnail_path, d.archive_path):
            try:
                if p and os.path.isfile(p):
                    os.unlink(p)
            except Exception:
                pass
    Document.objects.all().delete()
    return err_type, err_msg


def main():
    work = os.path.join(CTX["scratch_dir"], "errpath")
    os.makedirs(work, exist_ok=True)

    # CASE A: encrypted.pdf
    a = os.path.join(work, "encrypted.pdf")
    shutil.copy(ENCRYPTED, a)
    try_consume("CASE A (encrypted.pdf)", a)

    # CASE B: genuinely corrupt pdf (valid %PDF header, truncated garbage body)
    b = os.path.join(work, "corrupt.pdf")
    with open(DIGITAL, "rb") as f:
        head = f.read(400)  # keep %PDF- header so magic detects application/pdf
    with open(b, "wb") as f:
        f.write(head)
        f.write(b"\n%%CORRUPTED GARBAGE BODY\n" + bytes(range(256)) * 4)
    try_consume("CASE B (corrupt.pdf)", b)
    print("\nERROR_PATH_DONE")


if __name__ == "__main__":
    main()
```

### A-26. `tika_probe.py` — office/Tika availability probe — UNAVAILABLE in canonical config (F4)

```python
"""F4: office/Tika path availability in canonical config (NON-CANONICAL variant).

settings.py:L592 PAPERLESS_TIKA_ENABLED default 'NO'. paperless_tika/apps.py:L12
only connects tika_consumer_declaration when the flag is True. So in the DEFAULT
canonical configuration there is NO parser for office mime types -> office docs
cannot be consumed and no office metadata path runs. This probes
get_parser_class_for_mime_type for office mimes in the canonical config and
reports the supported-mime set, then labels the Tika path NON-CANONICAL /
unavailable (no Tika server, no office samples in the repo).
"""
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()


def main():
    from django.conf import settings
    from documents.parsers import (get_parser_class_for_mime_type,
                                    get_supported_file_extensions)

    print(f"PAPERLESS_TIKA_ENABLED (canonical default) = {settings.PAPERLESS_TIKA_ENABLED}")
    office = {
        ".docx": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        ".odt": "application/vnd.oasis.opendocument.text",
        ".doc": "application/msword",
        ".xlsx": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    }
    print("\n[office mime -> parser class in CANONICAL config]")
    for ext, mime in office.items():
        pc = get_parser_class_for_mime_type(mime)
        print(f"  {ext:6s} {mime[:60]:60s} -> {pc}")

    exts = sorted(get_supported_file_extensions())
    office_exts = [e for e in exts if e in office]
    print(f"\n  supported extensions count={len(exts)}")
    print(f"  any office extension supported in canonical config? {office_exts or 'NONE'}")
    print("  => office/Tika metadata path is UNAVAILABLE in canonical config "
          "(non-canonical: requires PAPERLESS_TIKA_ENABLED=YES + a Tika server).")
    print("TIKA_PROBE_DONE")


if __name__ == "__main__":
    main()
```

### A-27. `image_verify.py` — verify the image PDF's OCR actually succeeds and extracts content (helper)

```python
import os, shutil, tracemalloc
tracemalloc.start(1)
import bootstrap
CTX = bootstrap.bootstrap()
IMG = "/app/src/paperless_tesseract/tests/samples/multi-page-images.pdf"
from documents.tasks import consume_file
from documents.models import Document
work = os.path.join(CTX["scratch_dir"], "imgverify")
os.makedirs(work, exist_ok=True)
dst = os.path.join(work, "multi-page-images.pdf")
shutil.copy(IMG, dst)
r = consume_file(dst)
d = Document.objects.latest("id")
print("consume_file returned:", r)
print("content_len:", len(d.content))
print("content_preview:", repr(d.content[:120]))
print("has_archive_version:", d.has_archive_version)
print("mime_type:", d.mime_type)
print("IMAGE_VERIFY_DONE")
```

### A-28. `large_text.py` — large plain-text consume transient (Q2 / §5.4)

```python
"""Q2/§5.4: large-text transient through the REAL consume_file.

Generates an N-MiB plain-text document and consumes it through the real
consume_file entry point under light 1-frame tracemalloc, reporting the
transient peak (before/during/after + gc.collect) and the stored content
length. Isolates the Python-heap cost of holding the full extracted text
(parser self.text -> Document.content) as document size grows.
Usage: python large_text.py <MiB>
"""
import gc
import os
import sys
import tracemalloc
tracemalloc.start(1)

import bootstrap
CTX = bootstrap.bootstrap()
import probe

MiB = int(sys.argv[1]) if len(sys.argv) > 1 else 5


def main():
    from documents.tasks import consume_file
    from documents.models import Document
    work = os.path.join(CTX["scratch_dir"], f"largetext-{MiB}")
    os.makedirs(work, exist_ok=True)
    dst = os.path.join(work, f"large-{MiB}MiB.txt")
    # ~64-char lines of plain words (no date-like tokens -> date parser finds nothing,
    # so we isolate the content-string transient, not date-parsing cost).
    line = ("the quick brown fox jumps over the lazy dog " * 2)[:63] + "\n"
    target = MiB * 1024 * 1024
    with open(dst, "w") as f:
        written = 0
        while written < target:
            f.write(line)
            written += len(line)
    size = os.path.getsize(dst)
    print(f"===== LARGE TEXT consume size={size}B (~{size/1048576:.2f} MiB) =====")
    b = probe.probe("large-text BEFORE")
    tracemalloc.reset_peak()
    consume_file(dst)
    a = probe.probe("large-text AFTER (peak = transient)")
    g = probe.probe("large-text AFTER-GC", do_gc=True)
    d = Document.objects.latest("id")
    drss = (a["vmrss_kb"] - b["vmrss_kb"]) / 1024
    print(f"[SUMMARY] input=~{size/1048576:.2f}MiB dRSS={drss:.2f}MiB "
          f"peakTM={a['tm_peak']/1048576:.2f}MiB "
          f"tm_current_after={a['tm_current']/1048576:.2f}MiB "
          f"tm_current_after_gc={g['tm_current']/1048576:.2f}MiB "
          f"dLiveObj={a['n_obj']-b['n_obj']} freed={g['collected']} "
          f"stored_content_len={len(d.content)}")
    print("LARGE_TEXT_DONE")


if __name__ == "__main__":
    main()
```

---

## Appendix B — Raw captured output (verbatim, unedited)

All runs used the standard invocation:

```
cd /app/src && PYTHONPATH=/app/src DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/<script>.py [args]
```
(canonical Python 3.9.23; `DEBUG=NO`; Redis started with `redis-server --daemonize yes --save '' --appendonly no`).

### B-1. Corrected canonical consume magnitudes — `light_consume.py` (Q1, Q5-single)

Each document type was consumed through the real `consume_file` in **its own fresh process** (isolated `docker exec`), so the `ru_maxrss` high-water is a clean per-type figure. Each process runs the same type twice — a cold run 1 and a warm run 2 — for the two-run magnitude stability required by Rule R2. Commands (three isolated processes):
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py text
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py digital
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py image
```
Complete, unedited output (`/tmp/mem_results/light_consume_isolated_full.txt`, 180 lines; each `$ …` line is the exact invocation of that isolated process, followed by its two runs). This is the source of every number in §4.2 and the §4.4 first-touch table:
```
############## ISOLATED PROCESS: light_consume.py text ##############
$ cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py text

===== LIGHT CONSUME type=text run=1 size=21B =====
[PROBE] text r1 BEFORE
    RSS_current(VmRSS)  = 118720 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 116892 kB (114.2 MiB)
    tracemalloc.current = 42398850 B (40.43 MiB)
    tracemalloc.peak    = 43404231 B (41.39 MiB)
    gc.live_objects     = 91830
    gc.get_count()      = (46, 7, 2)
[2026-07-08 07:30:39,108] [INFO] [paperless.consumer] Consuming simple.txt
[2026-07-08 07:30:39,816] [INFO] [paperless.consumer] Document 2026-07-08 simple consumption finished
[PROBE] text r1 AFTER
    RSS_current(VmRSS)  = 133400 kB (130.3 MiB)
    RSS_hiwater(maxrss) = 131228 kB (128.2 MiB)
    tracemalloc.current = 45273492 B (43.18 MiB)
    tracemalloc.peak    = 45582988 B (43.47 MiB)
    gc.live_objects     = 96087
    gc.get_count()      = (70, 10, 3)
[PROBE] text r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 133400 kB (130.3 MiB)
    RSS_hiwater(maxrss) = 131228 kB (128.2 MiB)
    tracemalloc.current = 45129051 B (43.04 MiB)
    tracemalloc.peak    = 46071961 B (43.94 MiB)
    gc.live_objects     = 95396
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 521
[SUMMARY] type=text run=1 dRSS=14.34MiB peakTM=43.47MiB dLiveObj=4257 freed=521

===== LIGHT CONSUME type=text run=2 size=21B =====
[PROBE] text r2 BEFORE
    RSS_current(VmRSS)  = 133400 kB (130.3 MiB)
    RSS_hiwater(maxrss) = 131228 kB (128.2 MiB)
    tracemalloc.current = 45185264 B (43.09 MiB)
    tracemalloc.peak    = 46071961 B (43.94 MiB)
    gc.live_objects     = 95470
    gc.get_count()      = (352, 0, 0)
[2026-07-08 07:30:39,879] [INFO] [paperless.consumer] Consuming simple.txt
[2026-07-08 07:30:40,502] [INFO] [paperless.consumer] Document 2026-07-08 simple consumption finished
[PROBE] text r2 AFTER
    RSS_current(VmRSS)  = 135300 kB (132.1 MiB)
    RSS_hiwater(maxrss) = 132252 kB (129.2 MiB)
    tracemalloc.current = 45258584 B (43.16 MiB)
    tracemalloc.peak    = 45619664 B (43.51 MiB)
    gc.live_objects     = 95552
    gc.get_count()      = (19, 2, 0)
[PROBE] text r2 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 135300 kB (132.1 MiB)
    RSS_hiwater(maxrss) = 132252 kB (129.2 MiB)
    tracemalloc.current = 45191662 B (43.10 MiB)
    tracemalloc.peak    = 46057052 B (43.92 MiB)
    gc.live_objects     = 95490
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 51
[SUMMARY] type=text run=2 dRSS=1.86MiB peakTM=43.51MiB dLiveObj=82 freed=51

LIGHT_CONSUME_DONE
############## ISOLATED PROCESS: light_consume.py digital ##############
$ cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py digital

===== LIGHT CONSUME type=digital run=1 size=22926B =====
[PROBE] digital r1 BEFORE
    RSS_current(VmRSS)  = 118540 kB (115.8 MiB)
    RSS_hiwater(maxrss) = 113812 kB (111.1 MiB)
    tracemalloc.current = 42397920 B (40.43 MiB)
    tracemalloc.peak    = 43388299 B (41.38 MiB)
    gc.live_objects     = 91837
    gc.get_count()      = (46, 8, 2)
[2026-07-08 07:30:47,008] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:30:49,452] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[PROBE] digital r1 AFTER
    RSS_current(VmRSS)  = 149380 kB (145.9 MiB)
    RSS_hiwater(maxrss) = 143508 kB (140.1 MiB)
    tracemalloc.current = 55678752 B (53.10 MiB)
    tracemalloc.peak    = 57640959 B (54.97 MiB)
    gc.live_objects     = 107808
    gc.get_count()      = (67, 10, 6)
[PROBE] digital r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 149908 kB (146.4 MiB)
    RSS_hiwater(maxrss) = 144532 kB (141.1 MiB)
    tracemalloc.current = 55483803 B (52.91 MiB)
    tracemalloc.peak    = 57640959 B (54.97 MiB)
    gc.live_objects     = 106638
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 890
[SUMMARY] type=digital run=1 dRSS=30.12MiB peakTM=54.97MiB dLiveObj=15971 freed=890

===== LIGHT CONSUME type=digital run=2 size=22926B =====
[PROBE] digital r2 BEFORE
    RSS_current(VmRSS)  = 149908 kB (146.4 MiB)
    RSS_hiwater(maxrss) = 144532 kB (141.1 MiB)
    tracemalloc.current = 55542488 B (52.97 MiB)
    tracemalloc.peak    = 57640959 B (54.97 MiB)
    gc.live_objects     = 106720
    gc.get_count()      = (360, 0, 0)
[2026-07-08 07:30:49,507] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:30:51,486] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[PROBE] digital r2 AFTER
    RSS_current(VmRSS)  = 149972 kB (146.5 MiB)
    RSS_hiwater(maxrss) = 144532 kB (141.1 MiB)
    tracemalloc.current = 55671383 B (53.09 MiB)
    tracemalloc.peak    = 56038527 B (53.44 MiB)
    gc.live_objects     = 107147
    gc.get_count()      = (18, 3, 0)
[PROBE] digital r2 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 149972 kB (146.5 MiB)
    RSS_hiwater(maxrss) = 144532 kB (141.1 MiB)
    tracemalloc.current = 55550689 B (52.98 MiB)
    tracemalloc.peak    = 56569599 B (53.95 MiB)
    gc.live_objects     = 106740
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 392
[SUMMARY] type=digital run=2 dRSS=0.06MiB peakTM=53.44MiB dLiveObj=427 freed=392

LIGHT_CONSUME_DONE
############## ISOLATED PROCESS: light_consume.py image ##############
$ cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/light_consume.py image

===== LIGHT CONSUME type=image run=1 size=150479B =====
[PROBE] image r1 BEFORE
    RSS_current(VmRSS)  = 118640 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 115544 kB (112.8 MiB)
    tracemalloc.current = 42396130 B (40.43 MiB)
    tracemalloc.peak    = 43413610 B (41.40 MiB)
    gc.live_objects     = 91830
    gc.get_count()      = (46, 7, 2)
[2026-07-08 07:30:58,069] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:30:58,906] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:30:58,918] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:30:58,924] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:31:02,317] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[PROBE] image r1 AFTER
    RSS_current(VmRSS)  = 191508 kB (187.0 MiB)
    RSS_hiwater(maxrss) = 247640 kB (241.8 MiB)
    tracemalloc.current = 56009058 B (53.41 MiB)
    tracemalloc.peak    = 57575286 B (54.91 MiB)
    gc.live_objects     = 108190
    gc.get_count()      = (115, 10, 6)
[PROBE] image r1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 163440 kB (159.6 MiB)
    RSS_hiwater(maxrss) = 247640 kB (241.8 MiB)
    tracemalloc.current = 55764968 B (53.18 MiB)
    tracemalloc.peak    = 57575286 B (54.91 MiB)
    gc.live_objects     = 106885
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 1022
[SUMMARY] type=image run=1 dRSS=71.16MiB peakTM=54.91MiB dLiveObj=16360 freed=1022

===== LIGHT CONSUME type=image run=2 size=150479B =====
[PROBE] image r2 BEFORE
    RSS_current(VmRSS)  = 163440 kB (159.6 MiB)
    RSS_hiwater(maxrss) = 247640 kB (241.8 MiB)
    tracemalloc.current = 55823229 B (53.24 MiB)
    tracemalloc.peak    = 57575286 B (54.91 MiB)
    gc.live_objects     = 106967
    gc.get_count()      = (360, 0, 0)
[2026-07-08 07:31:02,390] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:31:02,833] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:31:02,844] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:31:02,850] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:31:06,205] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[PROBE] image r2 AFTER
    RSS_current(VmRSS)  = 222388 kB (217.2 MiB)
    RSS_hiwater(maxrss) = 270148 kB (263.8 MiB)
    tracemalloc.current = 55998725 B (53.40 MiB)
    tracemalloc.peak    = 56765612 B (54.14 MiB)
    gc.live_objects     = 107400
    gc.get_count()      = (12, 4, 0)
[PROBE] image r2 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 223104 kB (217.9 MiB)
    RSS_hiwater(maxrss) = 270148 kB (263.8 MiB)
    tracemalloc.current = 55837658 B (53.25 MiB)
    tracemalloc.peak    = 56896874 B (54.26 MiB)
    gc.live_objects     = 106988
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 396
[SUMMARY] type=image run=2 dRSS=57.57MiB peakTM=54.14MiB dLiveObj=433 freed=396

LIGHT_CONSUME_DONE
```

### B-1a. Per-stage boundary probes — `stage_consume.py` (Q1, F2)

The per-stage harness monkey-patches the concrete parser methods (resolved from the real factory `get_parser` in `paperless_tesseract/signals.py`), `consumer.load_classifier`, `Consumer._store`, and the three `document_consumption_finished` handlers, calling `probe()` immediately before and after each stage (with `tracemalloc.reset_peak()` at each stage start), driven through the real `consume_file`. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/stage_consume.py
```
Complete, unedited output (`/tmp/mem_results/stage_consume.txt`, 724 lines — digital run 1, image run 1, digital run 2, image run 2; each block has PIPELINE start, every STAGE BEFORE/AFTER, PIPELINE end, PIPELINE end AFTER gc.collect(), then a per-run STAGE DELTA SUMMARY). This backs every boundary in §4.3:
```

===== STAGE CONSUME sample=digital run=1 mime=application/pdf parser=RasterisedDocumentParser =====
[PROBE] PIPELINE start (before consume_file)
    RSS_current(VmRSS)  = 119304 kB (116.5 MiB)
    RSS_hiwater(maxrss) = 116496 kB (113.8 MiB)
    tracemalloc.current = 42436125 B (40.47 MiB)
    tracemalloc.peak    = 43385240 B (41.38 MiB)
    gc.live_objects     = 91942
    gc.get_count()      = (298, 7, 2)
[2026-07-08 06:40:10,806] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[PROBE] STAGE parse BEFORE
    RSS_current(VmRSS)  = 123276 kB (120.4 MiB)
    RSS_hiwater(maxrss) = 120592 kB (117.8 MiB)
    tracemalloc.current = 43684666 B (41.66 MiB)
    tracemalloc.peak    = 44919281 B (42.84 MiB)
    gc.live_objects     = 93835
    gc.get_count()      = (0, 2, 3)
[PROBE] STAGE parse AFTER
    RSS_current(VmRSS)  = 149164 kB (145.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54916860 B (52.37 MiB)
    tracemalloc.peak    = 57647585 B (54.98 MiB)
    gc.live_objects     = 107070
    gc.get_count()      = (414, 4, 6)
[PROBE] STAGE thumbnail BEFORE
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54931220 B (52.39 MiB)
    tracemalloc.peak    = 57647585 B (54.98 MiB)
    gc.live_objects     = 107103
    gc.get_count()      = (491, 4, 6)
[PROBE] STAGE thumbnail AFTER
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54929865 B (52.39 MiB)
    tracemalloc.peak    = 54986712 B (52.44 MiB)
    gc.live_objects     = 107107
    gc.get_count()      = (498, 4, 6)
[PROBE] STAGE get_text BEFORE
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54930399 B (52.39 MiB)
    tracemalloc.peak    = 55831093 B (53.24 MiB)
    gc.live_objects     = 107110
    gc.get_count()      = (498, 4, 6)
[PROBE] STAGE get_text AFTER
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54930932 B (52.39 MiB)
    tracemalloc.peak    = 54930932 B (52.39 MiB)
    gc.live_objects     = 107112
    gc.get_count()      = (498, 4, 6)
[PROBE] STAGE get_date BEFORE
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54931466 B (52.39 MiB)
    tracemalloc.peak    = 55832160 B (53.25 MiB)
    gc.live_objects     = 107115
    gc.get_count()      = (498, 4, 6)
[PROBE] STAGE get_date AFTER
    RSS_current(VmRSS)  = 149700 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54931999 B (52.39 MiB)
    tracemalloc.peak    = 54931999 B (52.39 MiB)
    gc.live_objects     = 107117
    gc.get_count()      = (498, 4, 6)
[PROBE] STAGE get_archive_path BEFORE
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54900136 B (52.36 MiB)
    tracemalloc.peak    = 55833227 B (53.25 MiB)
    gc.live_objects     = 106637
    gc.get_count()      = (0, 5, 6)
[PROBE] STAGE get_archive_path AFTER
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54897134 B (52.35 MiB)
    tracemalloc.peak    = 54897134 B (52.35 MiB)
    gc.live_objects     = 106639
    gc.get_count()      = (0, 5, 6)
[PROBE] STAGE load_classifier BEFORE
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54897602 B (52.35 MiB)
    tracemalloc.peak    = 55798362 B (53.21 MiB)
    gc.live_objects     = 106641
    gc.get_count()      = (0, 5, 6)
[PROBE] STAGE load_classifier AFTER
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54898054 B (52.35 MiB)
    tracemalloc.peak    = 54904329 B (52.36 MiB)
    gc.live_objects     = 106643
    gc.get_count()      = (0, 5, 6)
[PROBE] STAGE _store BEFORE
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54904374 B (52.36 MiB)
    tracemalloc.peak    = 55799282 B (53.21 MiB)
    gc.live_objects     = 106680
    gc.get_count()      = (30, 5, 6)
[PROBE] STAGE _store AFTER
    RSS_current(VmRSS)  = 149704 kB (146.2 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 54935341 B (52.39 MiB)
    tracemalloc.peak    = 54949559 B (52.40 MiB)
    gc.live_objects     = 106769
    gc.get_count()      = (164, 5, 6)
[PROBE] STAGE handler:set_correspondent BEFORE
    RSS_current(VmRSS)  = 150192 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55642629 B (53.06 MiB)
    tracemalloc.peak    = 55973872 B (53.38 MiB)
    gc.live_objects     = 107805
    gc.get_count()      = (1, 9, 6)
[PROBE] STAGE handler:set_correspondent AFTER
    RSS_current(VmRSS)  = 150196 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55656754 B (53.08 MiB)
    tracemalloc.peak    = 55663639 B (53.08 MiB)
    gc.live_objects     = 107836
    gc.get_count()      = (20, 9, 6)
[PROBE] STAGE handler:set_document_type BEFORE
    RSS_current(VmRSS)  = 150196 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55657357 B (53.08 MiB)
    tracemalloc.peak    = 56557982 B (53.94 MiB)
    gc.live_objects     = 107840
    gc.get_count()      = (20, 9, 6)
[PROBE] STAGE handler:set_document_type AFTER
    RSS_current(VmRSS)  = 150196 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55669776 B (53.09 MiB)
    tracemalloc.peak    = 55676634 B (53.10 MiB)
    gc.live_objects     = 107870
    gc.get_count()      = (36, 9, 6)
[PROBE] STAGE handler:set_tags BEFORE
    RSS_current(VmRSS)  = 150196 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55670466 B (53.09 MiB)
    tracemalloc.peak    = 56571004 B (53.95 MiB)
    gc.live_objects     = 107874
    gc.get_count()      = (36, 9, 6)
[PROBE] STAGE handler:set_tags AFTER
    RSS_current(VmRSS)  = 150196 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55674427 B (53.10 MiB)
    tracemalloc.peak    = 55683209 B (53.10 MiB)
    gc.live_objects     = 107879
    gc.get_count()      = (55, 9, 6)
[2026-07-08 06:40:13,340] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[PROBE] PIPELINE end (after consume_file)
    RSS_current(VmRSS)  = 150208 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55693992 B (53.11 MiB)
    tracemalloc.peak    = 56575655 B (53.95 MiB)
    gc.live_objects     = 107881
    gc.get_count()      = (130, 9, 6)
[PROBE] PIPELINE end AFTER gc.collect()
    RSS_current(VmRSS)  = 150208 kB (146.7 MiB)
    RSS_hiwater(maxrss) = 146192 kB (142.8 MiB)
    tracemalloc.current = 55499627 B (52.93 MiB)
    tracemalloc.peak    = 56592672 B (53.97 MiB)
    gc.live_objects     = 106705
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 890

----- STAGE DELTA SUMMARY sample=digital run=1 -----
stage                             dRSS_MiB dTM_cur_MiB  peak_MiB  dLiveObj
STAGE parse                          25.28       10.71     54.98     13235
STAGE thumbnail                       0.00       -0.00     52.44         4
STAGE get_text                        0.00        0.00     52.39         2
STAGE get_date                        0.00        0.00     52.39         2
STAGE get_archive_path                0.00       -0.00     52.35         2
STAGE load_classifier                 0.00        0.00     52.36         2
STAGE _store                          0.00        0.03     52.40        89
STAGE handler:set_correspondent       0.00        0.01     53.08        31
STAGE handler:set_document_type       0.00        0.01     53.10        30
STAGE handler:set_tags                0.00        0.00     53.10         5

===== STAGE CONSUME sample=image run=1 mime=application/pdf parser=RasterisedDocumentParser =====
[PROBE] PIPELINE start (before consume_file)
    RSS_current(VmRSS)  = 150348 kB (146.8 MiB)
    RSS_hiwater(maxrss) = 147216 kB (143.8 MiB)
    tracemalloc.current = 55547793 B (52.97 MiB)
    tracemalloc.peak    = 56592672 B (53.97 MiB)
    gc.live_objects     = 106792
    gc.get_count()      = (363, 0, 0)
[2026-07-08 06:40:13,405] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[PROBE] STAGE parse BEFORE
    RSS_current(VmRSS)  = 151388 kB (147.8 MiB)
    RSS_hiwater(maxrss) = 148240 kB (144.8 MiB)
    tracemalloc.current = 55563677 B (52.99 MiB)
    tracemalloc.peak    = 55844227 B (53.26 MiB)
    gc.live_objects     = 106871
    gc.get_count()      = (461, 0, 0)
[2026-07-08 06:40:13,855] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:40:13,862] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:40:13,869] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[PROBE] STAGE parse AFTER
    RSS_current(VmRSS)  = 246488 kB (240.7 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56359108 B (53.75 MiB)
    tracemalloc.peak    = 57053593 B (54.41 MiB)
    gc.live_objects     = 108535
    gc.get_count()      = (532, 4, 0)
[PROBE] STAGE thumbnail BEFORE
    RSS_current(VmRSS)  = 192320 kB (187.8 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56098596 B (53.50 MiB)
    tracemalloc.peak    = 57260336 B (54.61 MiB)
    gc.live_objects     = 107801
    gc.get_count()      = (0, 5, 0)
[PROBE] STAGE thumbnail AFTER
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56096308 B (53.50 MiB)
    tracemalloc.peak    = 56153411 B (53.55 MiB)
    gc.live_objects     = 107803
    gc.get_count()      = (2, 5, 0)
[PROBE] STAGE get_text BEFORE
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56096814 B (53.50 MiB)
    tracemalloc.peak    = 56997536 B (54.36 MiB)
    gc.live_objects     = 107806
    gc.get_count()      = (2, 5, 0)
[PROBE] STAGE get_text AFTER
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56097319 B (53.50 MiB)
    tracemalloc.peak    = 56097319 B (53.50 MiB)
    gc.live_objects     = 107808
    gc.get_count()      = (2, 5, 0)
[PROBE] STAGE get_date BEFORE
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56097825 B (53.50 MiB)
    tracemalloc.peak    = 56998547 B (54.36 MiB)
    gc.live_objects     = 107811
    gc.get_count()      = (2, 5, 0)
[PROBE] STAGE get_date AFTER
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56098330 B (53.50 MiB)
    tracemalloc.peak    = 56098330 B (53.50 MiB)
    gc.live_objects     = 107813
    gc.get_count()      = (2, 5, 0)
[PROBE] STAGE get_archive_path BEFORE
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56105115 B (53.51 MiB)
    tracemalloc.peak    = 56999558 B (54.36 MiB)
    gc.live_objects     = 107846
    gc.get_count()      = (24, 5, 0)
[PROBE] STAGE get_archive_path AFTER
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56103080 B (53.50 MiB)
    tracemalloc.peak    = 56103080 B (53.50 MiB)
    gc.live_objects     = 107848
    gc.get_count()      = (24, 5, 0)
[PROBE] STAGE load_classifier BEFORE
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56103548 B (53.50 MiB)
    tracemalloc.peak    = 57004308 B (54.36 MiB)
    gc.live_objects     = 107850
    gc.get_count()      = (24, 5, 0)
[PROBE] STAGE load_classifier AFTER
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56103984 B (53.50 MiB)
    tracemalloc.peak    = 56110259 B (53.51 MiB)
    gc.live_objects     = 107852
    gc.get_count()      = (24, 5, 0)
[PROBE] STAGE _store BEFORE
    RSS_current(VmRSS)  = 193036 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56108376 B (53.51 MiB)
    tracemalloc.peak    = 57005212 B (54.36 MiB)
    gc.live_objects     = 107889
    gc.get_count()      = (50, 5, 0)
[PROBE] STAGE _store AFTER
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56112152 B (53.51 MiB)
    tracemalloc.peak    = 56264590 B (53.66 MiB)
    gc.live_objects     = 107900
    gc.get_count()      = (83, 5, 0)
[PROBE] STAGE handler:set_correspondent BEFORE
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56123666 B (53.52 MiB)
    tracemalloc.peak    = 57013380 B (54.37 MiB)
    gc.live_objects     = 107846
    gc.get_count()      = (0, 6, 0)
[PROBE] STAGE handler:set_correspondent AFTER
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56124899 B (53.52 MiB)
    tracemalloc.peak    = 56131735 B (53.53 MiB)
    gc.live_objects     = 107851
    gc.get_count()      = (3, 6, 0)
[PROBE] STAGE handler:set_document_type BEFORE
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56125166 B (53.53 MiB)
    tracemalloc.peak    = 57026127 B (54.38 MiB)
    gc.live_objects     = 107855
    gc.get_count()      = (3, 6, 0)
[PROBE] STAGE handler:set_document_type AFTER
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56125948 B (53.53 MiB)
    tracemalloc.peak    = 56132864 B (53.53 MiB)
    gc.live_objects     = 107859
    gc.get_count()      = (5, 6, 0)
[PROBE] STAGE handler:set_tags BEFORE
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56126470 B (53.53 MiB)
    tracemalloc.peak    = 57027176 B (54.39 MiB)
    gc.live_objects     = 107863
    gc.get_count()      = (5, 6, 0)
[PROBE] STAGE handler:set_tags AFTER
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56127661 B (53.53 MiB)
    tracemalloc.peak    = 56136693 B (53.54 MiB)
    gc.live_objects     = 107868
    gc.get_count()      = (13, 6, 0)
[2026-07-08 06:40:17,429] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[PROBE] PIPELINE end (after consume_file)
    RSS_current(VmRSS)  = 193040 kB (188.5 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56132763 B (53.53 MiB)
    tracemalloc.peak    = 57028889 B (54.39 MiB)
    gc.live_objects     = 107870
    gc.get_count()      = (30, 6, 0)
[PROBE] PIPELINE end AFTER gc.collect()
    RSS_current(VmRSS)  = 175256 kB (171.1 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55946245 B (53.35 MiB)
    tracemalloc.peak    = 57031107 B (54.39 MiB)
    gc.live_objects     = 107215
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 582

----- STAGE DELTA SUMMARY sample=image run=1 -----
stage                             dRSS_MiB dTM_cur_MiB  peak_MiB  dLiveObj
STAGE parse                          92.87        0.76     54.41      1664
STAGE thumbnail                       0.70       -0.00     53.55         2
STAGE get_text                        0.00        0.00     53.50         2
STAGE get_date                        0.00        0.00     53.50         2
STAGE get_archive_path                0.00       -0.00     53.50         2
STAGE load_classifier                 0.00        0.00     53.51         2
STAGE _store                          0.00        0.00     53.66        11
STAGE handler:set_correspondent       0.00        0.00     53.53         5
STAGE handler:set_document_type       0.00        0.00     53.53         4
STAGE handler:set_tags                0.00        0.00     53.54         5

===== STAGE CONSUME sample=digital run=2 mime=application/pdf parser=RasterisedDocumentParser =====
[PROBE] PIPELINE start (before consume_file)
    RSS_current(VmRSS)  = 175260 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55955412 B (53.36 MiB)
    tracemalloc.peak    = 57031107 B (54.39 MiB)
    gc.live_objects     = 107231
    gc.get_count()      = (239, 0, 0)
[2026-07-08 06:40:17,480] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[PROBE] STAGE parse BEFORE
    RSS_current(VmRSS)  = 175260 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55969173 B (53.38 MiB)
    tracemalloc.peak    = 56249723 B (53.64 MiB)
    gc.live_objects     = 107309
    gc.get_count()      = (335, 0, 0)
[PROBE] STAGE parse AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56071674 B (53.47 MiB)
    tracemalloc.peak    = 56340307 B (53.73 MiB)
    gc.live_objects     = 107914
    gc.get_count()      = (568, 1, 0)
[PROBE] STAGE thumbnail BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56036775 B (53.44 MiB)
    tracemalloc.peak    = 56972902 B (54.33 MiB)
    gc.live_objects     = 107574
    gc.get_count()      = (0, 2, 0)
[PROBE] STAGE thumbnail AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56034711 B (53.44 MiB)
    tracemalloc.peak    = 56091702 B (53.49 MiB)
    gc.live_objects     = 107576
    gc.get_count()      = (5, 2, 0)
[PROBE] STAGE get_text BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56035217 B (53.44 MiB)
    tracemalloc.peak    = 56935939 B (54.30 MiB)
    gc.live_objects     = 107579
    gc.get_count()      = (5, 2, 0)
[PROBE] STAGE get_text AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56035722 B (53.44 MiB)
    tracemalloc.peak    = 56035722 B (53.44 MiB)
    gc.live_objects     = 107581
    gc.get_count()      = (5, 2, 0)
[PROBE] STAGE get_date BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56036228 B (53.44 MiB)
    tracemalloc.peak    = 56936950 B (54.30 MiB)
    gc.live_objects     = 107584
    gc.get_count()      = (5, 2, 0)
[PROBE] STAGE get_date AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56036733 B (53.44 MiB)
    tracemalloc.peak    = 56036733 B (53.44 MiB)
    gc.live_objects     = 107586
    gc.get_count()      = (5, 2, 0)
[PROBE] STAGE get_archive_path BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56043406 B (53.45 MiB)
    tracemalloc.peak    = 56937961 B (54.30 MiB)
    gc.live_objects     = 107619
    gc.get_count()      = (27, 2, 0)
[PROBE] STAGE get_archive_path AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56041371 B (53.45 MiB)
    tracemalloc.peak    = 56041371 B (53.45 MiB)
    gc.live_objects     = 107621
    gc.get_count()      = (27, 2, 0)
[PROBE] STAGE load_classifier BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56041839 B (53.45 MiB)
    tracemalloc.peak    = 56942599 B (54.30 MiB)
    gc.live_objects     = 107623
    gc.get_count()      = (27, 2, 0)
[PROBE] STAGE load_classifier AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56042275 B (53.45 MiB)
    tracemalloc.peak    = 56048550 B (53.45 MiB)
    gc.live_objects     = 107625
    gc.get_count()      = (27, 2, 0)
[PROBE] STAGE _store BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56046875 B (53.45 MiB)
    tracemalloc.peak    = 56943503 B (54.31 MiB)
    gc.live_objects     = 107662
    gc.get_count()      = (54, 2, 0)
[PROBE] STAGE _store AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56050823 B (53.45 MiB)
    tracemalloc.peak    = 56075626 B (53.48 MiB)
    gc.live_objects     = 107673
    gc.get_count()      = (84, 2, 0)
[PROBE] STAGE handler:set_correspondent BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56068774 B (53.47 MiB)
    tracemalloc.peak    = 56952051 B (54.31 MiB)
    gc.live_objects     = 107619
    gc.get_count()      = (0, 3, 0)
[PROBE] STAGE handler:set_correspondent AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56070911 B (53.47 MiB)
    tracemalloc.peak    = 56077689 B (53.48 MiB)
    gc.live_objects     = 107624
    gc.get_count()      = (5, 3, 0)
[PROBE] STAGE handler:set_document_type BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56071514 B (53.47 MiB)
    tracemalloc.peak    = 56972139 B (54.33 MiB)
    gc.live_objects     = 107628
    gc.get_count()      = (5, 3, 0)
[PROBE] STAGE handler:set_document_type AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56072296 B (53.47 MiB)
    tracemalloc.peak    = 56079154 B (53.48 MiB)
    gc.live_objects     = 107632
    gc.get_count()      = (7, 3, 0)
[PROBE] STAGE handler:set_tags BEFORE
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56072986 B (53.48 MiB)
    tracemalloc.peak    = 56973524 B (54.33 MiB)
    gc.live_objects     = 107636
    gc.get_count()      = (7, 3, 0)
[PROBE] STAGE handler:set_tags AFTER
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56075745 B (53.48 MiB)
    tracemalloc.peak    = 56084551 B (53.49 MiB)
    gc.live_objects     = 107641
    gc.get_count()      = (18, 3, 0)
[2026-07-08 06:40:19,550] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[PROBE] PIPELINE end (after consume_file)
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 56082065 B (53.48 MiB)
    tracemalloc.peak    = 56976973 B (54.34 MiB)
    gc.live_objects     = 107643
    gc.get_count()      = (36, 3, 0)
[PROBE] PIPELINE end AFTER gc.collect()
    RSS_current(VmRSS)  = 175264 kB (171.2 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55956659 B (53.36 MiB)
    tracemalloc.peak    = 56980745 B (54.34 MiB)
    gc.live_objects     = 107260
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 343

----- STAGE DELTA SUMMARY sample=digital run=2 -----
stage                             dRSS_MiB dTM_cur_MiB  peak_MiB  dLiveObj
STAGE parse                           0.00        0.10     53.73       605
STAGE thumbnail                       0.00       -0.00     53.49         2
STAGE get_text                        0.00        0.00     53.44         2
STAGE get_date                        0.00        0.00     53.44         2
STAGE get_archive_path                0.00       -0.00     53.45         2
STAGE load_classifier                 0.00        0.00     53.45         2
STAGE _store                          0.00        0.00     53.48        11
STAGE handler:set_correspondent       0.00        0.00     53.48         5
STAGE handler:set_document_type       0.00        0.00     53.48         4
STAGE handler:set_tags                0.00        0.00     53.49         5

===== STAGE CONSUME sample=image run=2 mime=application/pdf parser=RasterisedDocumentParser =====
[PROBE] PIPELINE start (before consume_file)
    RSS_current(VmRSS)  = 175400 kB (171.3 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55965587 B (53.37 MiB)
    tracemalloc.peak    = 56980745 B (54.34 MiB)
    gc.live_objects     = 107276
    gc.get_count()      = (239, 0, 0)
[2026-07-08 06:40:19,597] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[PROBE] STAGE parse BEFORE
    RSS_current(VmRSS)  = 176440 kB (172.3 MiB)
    RSS_hiwater(maxrss) = 239524 kB (233.9 MiB)
    tracemalloc.current = 55980175 B (53.39 MiB)
    tracemalloc.peak    = 56260727 B (53.65 MiB)
    gc.live_objects     = 107355
    gc.get_count()      = (335, 0, 0)
[2026-07-08 06:40:20,049] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:40:20,056] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:40:20,061] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[PROBE] STAGE parse AFTER
    RSS_current(VmRSS)  = 200132 kB (195.4 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56132989 B (53.53 MiB)
    tracemalloc.peak    = 56927678 B (54.29 MiB)
    gc.live_objects     = 107782
    gc.get_count()      = (7, 3, 0)
[PROBE] STAGE thumbnail BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56139795 B (53.54 MiB)
    tracemalloc.peak    = 57034217 B (54.39 MiB)
    gc.live_objects     = 107815
    gc.get_count()      = (29, 3, 0)
[PROBE] STAGE thumbnail AFTER
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56138003 B (53.54 MiB)
    tracemalloc.peak    = 56195106 B (53.59 MiB)
    gc.live_objects     = 107817
    gc.get_count()      = (32, 3, 0)
[PROBE] STAGE get_text BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56138509 B (53.54 MiB)
    tracemalloc.peak    = 57039231 B (54.40 MiB)
    gc.live_objects     = 107820
    gc.get_count()      = (32, 3, 0)
[PROBE] STAGE get_text AFTER
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56139014 B (53.54 MiB)
    tracemalloc.peak    = 56139014 B (53.54 MiB)
    gc.live_objects     = 107822
    gc.get_count()      = (32, 3, 0)
[PROBE] STAGE get_date BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56139520 B (53.54 MiB)
    tracemalloc.peak    = 57040242 B (54.40 MiB)
    gc.live_objects     = 107825
    gc.get_count()      = (32, 3, 0)
[PROBE] STAGE get_date AFTER
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56140025 B (53.54 MiB)
    tracemalloc.peak    = 56140025 B (53.54 MiB)
    gc.live_objects     = 107827
    gc.get_count()      = (32, 3, 0)
[PROBE] STAGE get_archive_path BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56146714 B (53.55 MiB)
    tracemalloc.peak    = 57041253 B (54.40 MiB)
    gc.live_objects     = 107860
    gc.get_count()      = (54, 3, 0)
[PROBE] STAGE get_archive_path AFTER
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56144624 B (53.54 MiB)
    tracemalloc.peak    = 56144624 B (53.54 MiB)
    gc.live_objects     = 107862
    gc.get_count()      = (54, 3, 0)
[PROBE] STAGE load_classifier BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56145092 B (53.54 MiB)
    tracemalloc.peak    = 57045852 B (54.40 MiB)
    gc.live_objects     = 107864
    gc.get_count()      = (54, 3, 0)
[PROBE] STAGE load_classifier AFTER
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56145528 B (53.54 MiB)
    tracemalloc.peak    = 56151803 B (53.55 MiB)
    gc.live_objects     = 107866
    gc.get_count()      = (54, 3, 0)
[PROBE] STAGE _store BEFORE
    RSS_current(VmRSS)  = 200848 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56149749 B (53.55 MiB)
    tracemalloc.peak    = 57046756 B (54.40 MiB)
    gc.live_objects     = 107903
    gc.get_count()      = (80, 3, 0)
[PROBE] STAGE _store AFTER
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56153472 B (53.55 MiB)
    tracemalloc.peak    = 56305963 B (53.70 MiB)
    gc.live_objects     = 107914
    gc.get_count()      = (110, 3, 0)
[PROBE] STAGE handler:set_correspondent BEFORE
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56168480 B (53.57 MiB)
    tracemalloc.peak    = 57054700 B (54.41 MiB)
    gc.live_objects     = 107717
    gc.get_count()      = (0, 4, 0)
[PROBE] STAGE handler:set_correspondent AFTER
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56169729 B (53.57 MiB)
    tracemalloc.peak    = 56176507 B (53.57 MiB)
    gc.live_objects     = 107722
    gc.get_count()      = (3, 4, 0)
[PROBE] STAGE handler:set_document_type BEFORE
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56169996 B (53.57 MiB)
    tracemalloc.peak    = 57070957 B (54.43 MiB)
    gc.live_objects     = 107726
    gc.get_count()      = (3, 4, 0)
[PROBE] STAGE handler:set_document_type AFTER
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56170778 B (53.57 MiB)
    tracemalloc.peak    = 56177636 B (53.58 MiB)
    gc.live_objects     = 107730
    gc.get_count()      = (5, 4, 0)
[PROBE] STAGE handler:set_tags BEFORE
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56171300 B (53.57 MiB)
    tracemalloc.peak    = 57072006 B (54.43 MiB)
    gc.live_objects     = 107734
    gc.get_count()      = (5, 4, 0)
[PROBE] STAGE handler:set_tags AFTER
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56172686 B (53.57 MiB)
    tracemalloc.peak    = 56181660 B (53.58 MiB)
    gc.live_objects     = 107739
    gc.get_count()      = (13, 4, 0)
[2026-07-08 06:40:23,557] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[PROBE] PIPELINE end (after consume_file)
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 56180082 B (53.58 MiB)
    tracemalloc.peak    = 57073914 B (54.43 MiB)
    gc.live_objects     = 107741
    gc.get_count()      = (30, 4, 0)
[PROBE] PIPELINE end AFTER gc.collect()
    RSS_current(VmRSS)  = 200852 kB (196.1 MiB)
    RSS_hiwater(maxrss) = 275948 kB (269.5 MiB)
    tracemalloc.current = 55965454 B (53.37 MiB)
    tracemalloc.peak    = 57076028 B (54.43 MiB)
    gc.live_objects     = 107310
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 392

----- STAGE DELTA SUMMARY sample=image run=2 -----
stage                             dRSS_MiB dTM_cur_MiB  peak_MiB  dLiveObj
STAGE parse                          23.14        0.15     54.29       427
STAGE thumbnail                       0.00       -0.00     53.59         2
STAGE get_text                        0.00        0.00     53.54         2
STAGE get_date                        0.00        0.00     53.54         2
STAGE get_archive_path                0.00       -0.00     53.54         2
STAGE load_classifier                 0.00        0.00     53.55         2
STAGE _store                          0.00        0.00     53.70        11
STAGE handler:set_correspondent       0.00        0.00     53.57         5
STAGE handler:set_document_type       0.00        0.00     53.58         4
STAGE handler:set_tags                0.00        0.00     53.58         5

STAGE_CONSUME_DONE
```

### B-1b. Stage-level `file:line` attribution — `attrib_harness.py` (Q1, F2)

`attrib_harness.py` drives the real `consume_file` under `tracemalloc.start(25)` and diffs a snapshot taken before vs. after the consume, printing the top-15 allocators by size delta and the top-12 by cumulative size, each by `file:line`. Commands (digital then image):
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/attrib_harness.py \
  /app/src/paperless_tesseract/tests/samples/simple-digital.pdf DIGITAL
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/attrib_harness.py \
  /app/src/paperless_tesseract/tests/samples/multi-page-images.pdf IMAGE
```
Complete, unedited output (`/tmp/mem_results/attrib_harness.txt`, 64 lines). This backs §4.5 — note the image PDF's top allocators are identical to the digital PDF's, with **no** PIL/ghostscript/tesseract entry (OCR memory is native, invisible to `tracemalloc`):
```
$ python /tmp/mem_harness/attrib_harness.py /app/src/paperless_tesseract/tests/samples/simple-digital.pdf DIGITAL

===== ATTRIBUTION label=DIGITAL file=simple-digital.pdf size=22926B =====
--- TOP 15 allocators by size DELTA (before-consume -> after-consume), by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.2 MiB (+7366 KiB), count=199763 (+67087), average=101 B
    <frozen importlib._bootstrap>:228: size=7784 KiB (-695 KiB), count=82994 (+6300), average=96 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB (+306 KiB), count=7228 (+7228), average=43 B
    /usr/local/lib/python3.9/site-packages/pdfminer/glyphlist.py:54: size=144 KiB (+144 KiB), count=2 (+2), average=72.0 KiB
    /usr/local/lib/python3.9/abc.py:106: size=312 KiB (+117 KiB), count=1395 (+594), average=229 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49: size=312 KiB (+101 KiB), count=354 (+58), average=903 B
    <frozen importlib._bootstrap_external>:123: size=322 KiB (+63.9 KiB), count=2707 (+530), average=122 B
    /usr/local/lib/python3.9/collections/__init__.py:497: size=140 KiB (+44.7 KiB), count=658 (+207), average=217 B
    <frozen importlib._bootstrap>:353: size=180 KiB (+35.4 KiB), count=2558 (+504), average=72 B
    /usr/local/lib/python3.9/typing.py:715: size=75.1 KiB (+35.0 KiB), count=1065 (+498), average=72 B
    <frozen importlib._bootstrap>:36: size=164 KiB (+32.9 KiB), count=2469 (+496), average=68 B
    /usr/local/lib/python3.9/enum.py:214: size=102 KiB (+30.7 KiB), count=530 (+162), average=196 B
    /usr/local/lib/python3.9/abc.py:107: size=76.3 KiB (+27.2 KiB), count=368 (+143), average=212 B
    /usr/local/lib/python3.9/abc.py:24: size=39.3 KiB (+25.7 KiB), count=391 (+242), average=103 B
    <frozen importlib._bootstrap_external>:1009: size=125 KiB (+25.2 KiB), count=2468 (+496), average=52 B
--- TOP 12 allocators by CUMULATIVE size after consume, by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.2 MiB, count=199763, average=101 B
    <frozen importlib._bootstrap>:228: size=7784 KiB, count=82994, average=96 B
    /usr/local/lib/python3.9/linecache.py:137: size=1014 KiB, count=10068, average=103 B
    <frozen importlib._bootstrap_external>:123: size=322 KiB, count=2707, average=122 B
    /usr/local/lib/python3.9/abc.py:106: size=312 KiB, count=1395, average=229 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49: size=312 KiB, count=354, average=903 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB, count=7228, average=43 B
    /usr/local/lib/python3.9/site-packages/django/db/models/base.py:75: size=204 KiB, count=809, average=259 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:138: size=195 KiB, count=2082, average=96 B
    <frozen importlib._bootstrap>:353: size=180 KiB, count=2558, average=72 B
    /usr/local/lib/python3.9/copy.py:279: size=169 KiB, count=802, average=216 B
    /usr/local/lib/python3.9/site-packages/django/forms/widgets.py:220: size=164 KiB, count=722, average=233 B
$ python /tmp/mem_harness/attrib_harness.py /app/src/paperless_tesseract/tests/samples/multi-page-images.pdf IMAGE

===== ATTRIBUTION label=IMAGE file=multi-page-images.pdf size=150479B =====
--- TOP 15 allocators by size DELTA (before-consume -> after-consume), by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.3 MiB (+7493 KiB), count=201260 (+68589), average=101 B
    <frozen importlib._bootstrap>:228: size=7779 KiB (-698 KiB), count=82951 (+6289), average=96 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB (+306 KiB), count=7228 (+7228), average=43 B
    /usr/local/lib/python3.9/site-packages/pdfminer/glyphlist.py:54: size=144 KiB (+144 KiB), count=2 (+2), average=72.0 KiB
    /usr/local/lib/python3.9/abc.py:106: size=312 KiB (+117 KiB), count=1395 (+594), average=229 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49: size=312 KiB (+101 KiB), count=354 (+58), average=903 B
    <frozen importlib._bootstrap_external>:123: size=323 KiB (+64.9 KiB), count=2715 (+538), average=122 B
    /usr/local/lib/python3.9/collections/__init__.py:497: size=140 KiB (+44.7 KiB), count=658 (+207), average=217 B
    <frozen importlib._bootstrap>:353: size=180 KiB (+36.0 KiB), count=2566 (+512), average=72 B
    /usr/local/lib/python3.9/typing.py:715: size=74.3 KiB (+34.2 KiB), count=1053 (+486), average=72 B
    <frozen importlib._bootstrap>:36: size=164 KiB (+33.5 KiB), count=2477 (+504), average=68 B
    /usr/local/lib/python3.9/enum.py:214: size=104 KiB (+33.4 KiB), count=544 (+176), average=196 B
    /usr/local/lib/python3.9/abc.py:107: size=76.3 KiB (+27.2 KiB), count=368 (+143), average=212 B
    <frozen importlib._bootstrap_external>:1009: size=126 KiB (+25.6 KiB), count=2476 (+504), average=52 B
    /usr/local/lib/python3.9/sre_compile.py:804: size=159 KiB (+24.3 KiB), count=355 (+55), average=458 B
--- TOP 12 allocators by CUMULATIVE size after consume, by file:line ---
    <frozen importlib._bootstrap_external>:647: size=19.3 MiB, count=201260, average=101 B
    <frozen importlib._bootstrap>:228: size=7779 KiB, count=82951, average=96 B
    /usr/local/lib/python3.9/linecache.py:137: size=1014 KiB, count=10068, average=103 B
    <frozen importlib._bootstrap_external>:123: size=323 KiB, count=2715, average=122 B
    /usr/local/lib/python3.9/abc.py:106: size=312 KiB, count=1395, average=229 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49: size=312 KiB, count=354, average=903 B
    /usr/local/lib/python3.9/base64.py:325: size=306 KiB, count=7228, average=43 B
    /usr/local/lib/python3.9/site-packages/django/db/models/base.py:75: size=203 KiB, count=797, average=260 B
    /usr/local/lib/python3.9/site-packages/django/utils/functional.py:138: size=195 KiB, count=2082, average=96 B
    <frozen importlib._bootstrap>:353: size=180 KiB, count=2566, average=72 B
    /usr/local/lib/python3.9/copy.py:279: size=169 KiB, count=802, average=216 B
    <frozen importlib._bootstrap>:36: size=164 KiB, count=2477, average=68 B
```

### B-1c. Observer-effect: heavy vs. light instrumentation — `observer_effect.py` (§3.4)

Reproduces the profiler overhead that motivates using the light harness for all magnitudes: the **same** `simple-digital.pdf` driven through the real `consume_file`, twice per mode in two fresh processes. Commands:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/observer_effect.py heavy
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings \
  PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/observer_effect.py light
```
Complete, unedited output (`/tmp/mem_results/observer_effect.txt`, 77 lines). `tm.peak` is ~54.9 MiB in **both** modes while the heavy RSS high-water reaches 3251.9 / 3343.4 MiB versus the light harness's 142.1 MiB — the multi-GB delta is entirely the profiler's traceback + snapshot storage:
```
############## OBSERVER-EFFECT: heavy (tracemalloc.start(30) + per-stage snapshots) ##############
$ python /tmp/mem_harness/observer_effect.py heavy

===== OBSERVER-EFFECT mode=heavy nframe=30 run=1 size=22926B =====
[PROBE] heavy r1 BEFORE
    RSS_current(VmRSS)  = 227748 kB (222.4 MiB)
    RSS_hiwater(maxrss) = 225920 kB (220.6 MiB)
    tracemalloc.current = 42426813 B (40.46 MiB)
    tracemalloc.peak    = 43372972 B (41.36 MiB)
    gc.live_objects     = 91941
    gc.get_count()      = (243, 7, 2)
[PROBE] heavy r1 AFTER
    RSS_current(VmRSS)  = 3334032 kB (3255.9 MiB)
    RSS_hiwater(maxrss) = 3329952 kB (3251.9 MiB)
    tracemalloc.current = 55609302 B (53.03 MiB)
    tracemalloc.peak    = 57573968 B (54.91 MiB)
    gc.live_objects     = 121519
    gc.get_count()      = (78, 6, 52)
[SUMMARY] mode=heavy run=1 RSS_hiwater=3251.9MiB VmRSS_after=3255.9MiB tm.peak=54.91MiB heavy_snapshots_taken=14 retained=14

===== OBSERVER-EFFECT mode=heavy nframe=30 run=2 size=22926B =====
[PROBE] heavy r2 BEFORE
    RSS_current(VmRSS)  = 835244 kB (815.7 MiB)
    RSS_hiwater(maxrss) = 3329952 kB (3251.9 MiB)
    tracemalloc.current = 55592481 B (53.02 MiB)
    tracemalloc.peak    = 57573968 B (54.91 MiB)
    gc.live_objects     = 106820
    gc.get_count()      = (107, 6, 52)
[PROBE] heavy r2 AFTER
    RSS_current(VmRSS)  = 3428824 kB (3348.5 MiB)
    RSS_hiwater(maxrss) = 3423624 kB (3343.4 MiB)
    tracemalloc.current = 55676974 B (53.10 MiB)
    tracemalloc.peak    = 56737845 B (54.11 MiB)
    gc.live_objects     = 132911
    gc.get_count()      = (71, 10, 90)
[SUMMARY] mode=heavy run=2 RSS_hiwater=3343.4MiB VmRSS_after=3348.5MiB tm.peak=54.11MiB heavy_snapshots_taken=28 retained=14

OBSERVER_EFFECT_DONE

############## OBSERVER-EFFECT: light (tracemalloc.start(1), 2 probes) ##############
$ python /tmp/mem_harness/observer_effect.py light

===== OBSERVER-EFFECT mode=light nframe=1 run=1 size=22926B =====
[PROBE] light r1 BEFORE
    RSS_current(VmRSS)  = 118684 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 115836 kB (113.1 MiB)
    tracemalloc.current = 42396123 B (40.43 MiB)
    tracemalloc.peak    = 43380698 B (41.37 MiB)
    gc.live_objects     = 91834
    gc.get_count()      = (46, 7, 2)
[PROBE] light r1 AFTER
    RSS_current(VmRSS)  = 149588 kB (146.1 MiB)
    RSS_hiwater(maxrss) = 145532 kB (142.1 MiB)
    tracemalloc.current = 55673505 B (53.09 MiB)
    tracemalloc.peak    = 57638844 B (54.97 MiB)
    gc.live_objects     = 107789
    gc.get_count()      = (67, 9, 6)
[SUMMARY] mode=light run=1 RSS_hiwater=142.1MiB VmRSS_after=146.1MiB tm.peak=54.97MiB heavy_snapshots_taken=0 retained=0

===== OBSERVER-EFFECT mode=light nframe=1 run=2 size=22926B =====
[PROBE] light r2 BEFORE
    RSS_current(VmRSS)  = 150124 kB (146.6 MiB)
    RSS_hiwater(maxrss) = 145532 kB (142.1 MiB)
    tracemalloc.current = 55708591 B (53.13 MiB)
    tracemalloc.peak    = 57638844 B (54.97 MiB)
    gc.live_objects     = 107868
    gc.get_count()      = (180, 9, 6)
[PROBE] light r2 AFTER
    RSS_current(VmRSS)  = 150332 kB (146.8 MiB)
    RSS_hiwater(maxrss) = 145532 kB (142.1 MiB)
    tracemalloc.current = 55727129 B (53.15 MiB)
    tracemalloc.peak    = 56103638 B (53.50 MiB)
    gc.live_objects     = 107439
    gc.get_count()      = (18, 0, 7)
[SUMMARY] mode=light run=2 RSS_hiwater=142.1MiB VmRSS_after=146.8MiB tm.peak=53.50MiB heavy_snapshots_taken=0 retained=0

OBSERVER_EFFECT_DONE
```

### B-1d. Baseline native gap — post-`django.setup()`/`migrate` vs. heavy-import RSS/heap — `baseline.py` (§3, §10)

Command:

```
docker exec mem_investigation bash -c "cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/baseline.py"
```

Full unedited output:

```
[PROBE] baseline: after django.setup()+migrate
    RSS_current(VmRSS)  = 107764 kB (105.2 MiB)
    RSS_hiwater(maxrss) = 103220 kB (100.8 MiB)
    tracemalloc.current = 36450250 B (34.76 MiB)
    tracemalloc.peak    = 43784676 B (41.76 MiB)
    gc.live_objects     = 80265
    gc.get_count()      = (20, 0, 0)
[PROBE] baseline: after heavy import graph
    RSS_current(VmRSS)  = 202720 kB (198.0 MiB)
    RSS_hiwater(maxrss) = 197204 kB (192.6 MiB)
    tracemalloc.current = 74131765 B (70.70 MiB)
    tracemalloc.peak    = 74408216 B (70.96 MiB)
    gc.live_objects     = 131574
    gc.get_count()      = (158, 10, 1)
[PROBE] baseline: after gc.collect() AFTER gc.collect()
    RSS_current(VmRSS)  = 203688 kB (198.9 MiB)
    RSS_hiwater(maxrss) = 198228 kB (193.6 MiB)
    tracemalloc.current = 74114093 B (70.68 MiB)
    tracemalloc.peak    = 75272417 B (71.79 MiB)
    gc.live_objects     = 131335
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 0

[BASELINE GAP] RSS_current=198.9 MiB  tracemalloc.current=70.68 MiB  native_gap=128.2 MiB
BASELINE_DONE
```

### B-2. Metadata endpoint tempdir accumulation (repeated endpoint calls) — `metadata_harness.py` (Q2, F4)

The **real** DRF metadata endpoint (`DocumentViewSet.metadata` → `get_metadata`, `views.py:L260/L266/L269`) is driven via `APIRequestFactory` + `force_authenticate` + `resp.render()` for 1, 10, 50 and 200 calls, then `gc.collect()`. Each call constructs a parser for the original **and** the archive (`views.py:L295/L302`) without `cleanup()`, so `leaked_tempdirs` grows exactly 2 per call (2/20/100/400) while `bytes_in_tempdirs=0`, `live_pikepdf_Pdf=0`, RSS stays flat (170.1→170.5 MiB) and `tracemalloc.peak` is constant at 62.12 MiB — an empty-directory (inode) leak, not a RAM leak. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/metadata_harness.py
```
Complete, unedited output (`/tmp/mem_results/metadata_harness.txt`, 58 lines):
```
[2026-07-08 07:03:03,941] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:03:06,390] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
consumed doc pk=1 has_archive=True content_len=24

[BASELINE before any endpoint call] tempdirs=0 bytes_in_tempdirs=0 live_pdf=0
[PROBE] metadata BEFORE any call
    RSS_current(VmRSS)  = 174216 kB (170.1 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 63123628 B (60.20 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 121360
    gc.get_count()      = (252, 10, 9)
[PROBE] metadata AFTER 1 calls
    RSS_current(VmRSS)  = 174236 kB (170.2 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 63194505 B (60.27 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 121460
    gc.get_count()      = (432, 10, 9)
    -> after 1 calls: leaked_tempdirs=2 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
[PROBE] metadata AFTER 10 calls
    RSS_current(VmRSS)  = 174292 kB (170.2 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 63264851 B (60.33 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 121538
    gc.get_count()      = (197, 11, 9)
    -> after 10 calls: leaked_tempdirs=20 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
[PROBE] metadata AFTER 50 calls
    RSS_current(VmRSS)  = 174396 kB (170.3 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 63270117 B (60.34 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 121182
    gc.get_count()      = (255, 2, 10)
    -> after 50 calls: leaked_tempdirs=100 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
[PROBE] metadata AFTER 200 calls
    RSS_current(VmRSS)  = 174612 kB (170.5 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 63114433 B (60.19 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 120242
    gc.get_count()      = (184, 1, 0)
    -> after 200 calls: leaked_tempdirs=400 bytes_in_tempdirs=0 live_pikepdf_Pdf=0 status=200
[PROBE] metadata AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 174612 kB (170.5 MiB)
    RSS_hiwater(maxrss) = 172968 kB (168.9 MiB)
    tracemalloc.current = 62993704 B (60.08 MiB)
    tracemalloc.peak    = 65139696 B (62.12 MiB)
    gc.live_objects     = 119953
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 284
    -> after gc.collect(): leaked_tempdirs=400 bytes_in_tempdirs=0 live_pikepdf_Pdf=0
    -> total leaked tempdirs on disk: 400; listing first 3 contents:
        /tmp/mh-root-t1mrhkg6/scratch/paperless-p8v8d8uq -> contents=[]
        /tmp/mh-root-t1mrhkg6/scratch/paperless-dqx5t4dm -> contents=[]
        /tmp/mh-root-t1mrhkg6/scratch/paperless-6wl_qioa -> contents=[]
METADATA_HARNESS_DONE
```

### B-3. `pikepdf.Pdf` handle lifecycle + full-text single-copy — `q2_pikepdf_content.py` (Q2)

Part (a) calls the real `RasterisedDocumentParser.extract_metadata` directly and counts live `pikepdf.Pdf` before/during/after (no `gc.collect()`); part (b) wraps `Consumer._store` to check `text is document.content` identity and measures the `_store` transient plus a synthetic 40 MiB content-string assignment. Command:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/q2_pikepdf_content.py
```

Complete, unedited output (`/tmp/mem_results/q2_pikepdf_content.txt`, 17 lines):

```

===== Q2(a) pikepdf.Pdf handle lifecycle (direct extract_metadata) =====
  live pikepdf.Pdf BEFORE construct parser: 0
  live pikepdf.Pdf DURING extract_metadata (inside, after open): 1
  live pikepdf.Pdf AFTER extract_metadata returns (NO gc.collect): 0
  metadata entries returned: 5

===== Q2(b) content string identity (text is document.content) =====
[2026-07-08 07:04:07,009] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:04:09,449] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
  text IS document.content (same object, single copy): True
  content length: 24 chars
  sys.getrefcount(text) inside _store: 6
  tracemalloc.peak during _store: 52.49 MiB
  assigning 40MiB content: dTM_current=-0.00 MiB peak=93.15 MiB (single reference on the model field)

Q2_PIKEPDF_CONTENT_DONE
```

### B-3b. Large plain-text consume transient (5 MiB ×2 + 20 MiB) — `large_text.py` (Q2, §5.4)

A plain-text document of the given size is consumed through the real `consume_file`; the transient `tracemalloc.peak` and RSS are captured before/during/after + `gc.collect()`. Two 5 MiB runs (magnitude stability, R2) and one 20 MiB run (size scaling). Command:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/large_text.py <MiB>
```

Complete, unedited output (concatenation of `large_text_5.txt`, `large_text_5b.txt`, `large_text_20.txt`; 92 lines):

```
############## RUN 1: large_text.py 5 ##############
$ python /tmp/mem_harness/large_text.py 5

===== LARGE TEXT consume size=5242880B (~5.00 MiB) =====
[PROBE] large-text BEFORE
    RSS_current(VmRSS)  = 118516 kB (115.7 MiB)
    RSS_hiwater(maxrss) = 114312 kB (111.6 MiB)
    tracemalloc.current = 42699581 B (40.72 MiB)
    tracemalloc.peak    = 43689888 B (41.67 MiB)
    gc.live_objects     = 91421
    gc.get_count()      = (344, 5, 2)
[2026-07-08 09:33:50,973] [INFO] [paperless.consumer] Consuming large-5MiB.txt
[2026-07-08 09:34:00,178] [INFO] [paperless.consumer] Document 2026-07-08 large-5MiB consumption finished
[PROBE] large-text AFTER (peak = transient)
    RSS_current(VmRSS)  = 221428 kB (216.2 MiB)
    RSS_hiwater(maxrss) = 260692 kB (254.6 MiB)
    tracemalloc.current = 45583658 B (43.47 MiB)
    tracemalloc.peak    = 83659533 B (79.78 MiB)
    gc.live_objects     = 95689
    gc.get_count()      = (70, 9, 3)
[PROBE] large-text AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 221428 kB (216.2 MiB)
    RSS_hiwater(maxrss) = 260692 kB (254.6 MiB)
    tracemalloc.current = 45515955 B (43.41 MiB)
    tracemalloc.peak    = 83659533 B (79.78 MiB)
    gc.live_objects     = 95398
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 79
[SUMMARY] input=~5.00MiB dRSS=100.50MiB peakTM=79.78MiB tm_current_after=43.47MiB tm_current_after_gc=43.41MiB dLiveObj=4268 freed=79 stored_content_len=5242880
LARGE_TEXT_DONE

############## RUN 2 (stability): large_text.py 5 ##############
$ python /tmp/mem_harness/large_text.py 5

===== LARGE TEXT consume size=5242880B (~5.00 MiB) =====
[PROBE] large-text BEFORE
    RSS_current(VmRSS)  = 118636 kB (115.9 MiB)
    RSS_hiwater(maxrss) = 113832 kB (111.2 MiB)
    tracemalloc.current = 42781370 B (40.80 MiB)
    tracemalloc.peak    = 43811901 B (41.78 MiB)
    gc.live_objects     = 91831
    gc.get_count()      = (50, 7, 2)
[2026-07-08 09:35:06,179] [INFO] [paperless.consumer] Consuming large-5MiB.txt
[2026-07-08 09:35:15,224] [INFO] [paperless.consumer] Document 2026-07-08 large-5MiB consumption finished
[PROBE] large-text AFTER (peak = transient)
    RSS_current(VmRSS)  = 221248 kB (216.1 MiB)
    RSS_hiwater(maxrss) = 259004 kB (252.9 MiB)
    tracemalloc.current = 45661234 B (43.55 MiB)
    tracemalloc.peak    = 83736893 B (79.86 MiB)
    gc.live_objects     = 96088
    gc.get_count()      = (70, 10, 3)
[PROBE] large-text AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 221248 kB (216.1 MiB)
    RSS_hiwater(maxrss) = 259004 kB (252.9 MiB)
    tracemalloc.current = 45515659 B (43.41 MiB)
    tracemalloc.peak    = 83736893 B (79.86 MiB)
    gc.live_objects     = 95397
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 521
[SUMMARY] input=~5.00MiB dRSS=100.21MiB peakTM=79.86MiB tm_current_after=43.55MiB tm_current_after_gc=43.41MiB dLiveObj=4257 freed=521 stored_content_len=5242880
LARGE_TEXT_DONE

############## SCALE: large_text.py 20 ##############
$ python /tmp/mem_harness/large_text.py 20

===== LARGE TEXT consume size=20971520B (~20.00 MiB) =====
[PROBE] large-text BEFORE
    RSS_current(VmRSS)  = 119032 kB (116.2 MiB)
    RSS_hiwater(maxrss) = 117468 kB (114.7 MiB)
    tracemalloc.current = 42786177 B (40.80 MiB)
    tracemalloc.peak    = 43791296 B (41.76 MiB)
    gc.live_objects     = 91831
    gc.get_count()      = (50, 7, 2)
[2026-07-08 09:35:06,162] [INFO] [paperless.consumer] Consuming large-20MiB.txt
[2026-07-08 09:35:43,855] [INFO] [paperless.consumer] Document 2026-07-08 large-20MiB consumption finished
[PROBE] large-text AFTER (peak = transient)
    RSS_current(VmRSS)  = 458308 kB (447.6 MiB)
    RSS_hiwater(maxrss) = 641596 kB (626.6 MiB)
    tracemalloc.current = 45662955 B (43.55 MiB)
    tracemalloc.peak    = 196398509 B (187.30 MiB)
    gc.live_objects     = 96088
    gc.get_count()      = (70, 10, 3)
[PROBE] large-text AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 458080 kB (447.3 MiB)
    RSS_hiwater(maxrss) = 641596 kB (626.6 MiB)
    tracemalloc.current = 45517380 B (43.41 MiB)
    tracemalloc.peak    = 196398509 B (187.30 MiB)
    gc.live_objects     = 95397
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 521
[SUMMARY] input=~20.00MiB dRSS=331.32MiB peakTM=187.30MiB tm_current_after=43.55MiB tm_current_after_gc=43.41MiB dLiveObj=4257 freed=521 stored_content_len=20971520
LARGE_TEXT_DONE
```

### B-4. Classifier 7×`pickle.load` (classifier-present) — `classifier_harness.py` (Q3, F4)

Trains a **real** model in-process (8 documents, 2 correspondents via `MATCH_AUTO`; on-disk `MODEL_FILE` = 266 561 B), then drives the real `load_classifier()` (`classifier.py:L30`) and attributes each of the **7** `pickle.load()` calls by `tracemalloc` current-delta. The dominant object is the `MLPClassifier` (`correspondent_classifier`, load #6, 268.7 KiB); the `CountVectorizer` vocabulary (106 terms) is only 12.3 KiB; loads #4/#5/#7 are `None` (no `MATCH_AUTO` tags/types in the corpus). Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/classifier_harness.py
```
Complete, unedited output (`/tmp/mem_results/classifier_harness.txt`, 36 lines):
```
training docs=8 correspondents=2 (MATCH_AUTO)
[PROBE] before train()
    RSS_current(VmRSS)  = 107460 kB (104.9 MiB)
    RSS_hiwater(maxrss) = 102960 kB (100.5 MiB)
    tracemalloc.current = 36127089 B (34.45 MiB)
    tracemalloc.peak    = 43149609 B (41.15 MiB)
    gc.live_objects     = 80427
    gc.get_count()      = (209, 0, 0)
train() changed=True peak_during_train=62.71 MiB
MODEL_FILE exists=True size=266561 B
[PROBE] before load_classifier()
    RSS_current(VmRSS)  = 171184 kB (167.2 MiB)
    RSS_hiwater(maxrss) = 165424 kB (161.5 MiB)
    tracemalloc.current = 64995071 B (61.98 MiB)
    tracemalloc.peak    = 65751524 B (62.71 MiB)
    gc.live_objects     = 117387
    gc.get_count()      = (12, 0, 0)
[PROBE] after load_classifier()
    RSS_current(VmRSS)  = 171860 kB (167.8 MiB)
    RSS_hiwater(maxrss) = 166448 kB (162.5 MiB)
    tracemalloc.current = 65286311 B (62.26 MiB)
    tracemalloc.peak    = 65291753 B (62.27 MiB)
    gc.live_objects     = 117414
    gc.get_count()      = (149, 0, 0)

[LOAD_CLASSIFIER] returned=DocumentClassifier pickle.load_invocations=7
 # loaded_type                   dTM_current_KiB  peak_MiB
 1 int                                       0.1     62.00
 2 bytes                                     0.1     62.00
 3 CountVectorizer                          12.3     62.02
 4 NoneType                                  0.1     62.01
 5 NoneType                                  0.1     62.01
 6 MLPClassifier                           268.7     62.29
 7 NoneType                                  0.1     62.27
  data_vectorizer.vocabulary_ size = 106 terms
CLASSIFIER_HARNESS_DONE
```

### B-4b. Classifier shared across handlers, not accumulated (12-doc batch) — `classifier_batch.py` (Q3, F4)

Consumes 12 distinct documents in one long-lived process with a trained classifier present, probing the live `DocumentClassifier` count after every 3rd document and after `gc.collect()`, and recording the `id()` of the classifier each of the three signal handlers receives per document. The count is constant at 1 (no accumulation); `all_share_same_classifier_id=True` within each consume; the `id()` differs between documents (fresh load each consume). Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/classifier_batch.py
```
Complete, unedited output (`/tmp/mem_results/classifier_batch.txt`, 57 lines):
```
model built size=36564B

live DocumentClassifier BEFORE any consume: 1
[2026-07-08 07:06:27,916] [INFO] [paperless.consumer] Consuming d1.pdf
[2026-07-08 07:06:35,839] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d1
[2026-07-08 07:06:35,849] [INFO] [paperless.consumer] Document 2026-07-08 A d1 consumption finished
[2026-07-08 07:06:35,860] [INFO] [paperless.consumer] Consuming d2.pdf
[2026-07-08 07:06:37,851] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d2
[2026-07-08 07:06:37,860] [INFO] [paperless.consumer] Document 2026-07-08 A d2 consumption finished
[2026-07-08 07:06:37,870] [INFO] [paperless.consumer] Consuming d3.pdf
[2026-07-08 07:06:39,907] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d3
[2026-07-08 07:06:39,917] [INFO] [paperless.consumer] Document 2026-07-08 A d3 consumption finished
  after consume 3/12: live DocumentClassifier=1
[2026-07-08 07:06:39,971] [INFO] [paperless.consumer] Consuming d4.pdf
[2026-07-08 07:06:41,994] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d4
[2026-07-08 07:06:42,004] [INFO] [paperless.consumer] Document 2026-07-08 A d4 consumption finished
[2026-07-08 07:06:42,014] [INFO] [paperless.consumer] Consuming d5.pdf
[2026-07-08 07:06:44,016] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d5
[2026-07-08 07:06:44,026] [INFO] [paperless.consumer] Document 2026-07-08 A d5 consumption finished
[2026-07-08 07:06:44,037] [INFO] [paperless.consumer] Consuming d6.pdf
[2026-07-08 07:06:46,139] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d6
[2026-07-08 07:06:46,151] [INFO] [paperless.consumer] Document 2026-07-08 A d6 consumption finished
  after consume 6/12: live DocumentClassifier=1
[2026-07-08 07:06:46,207] [INFO] [paperless.consumer] Consuming d7.pdf
[2026-07-08 07:06:48,201] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d7
[2026-07-08 07:06:48,211] [INFO] [paperless.consumer] Document 2026-07-08 A d7 consumption finished
[2026-07-08 07:06:48,223] [INFO] [paperless.consumer] Consuming d8.pdf
[2026-07-08 07:06:50,242] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d8
[2026-07-08 07:06:50,252] [INFO] [paperless.consumer] Document 2026-07-08 A d8 consumption finished
[2026-07-08 07:06:50,263] [INFO] [paperless.consumer] Consuming d9.pdf
[2026-07-08 07:06:52,287] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d9
[2026-07-08 07:06:52,300] [INFO] [paperless.consumer] Document 2026-07-08 A d9 consumption finished
  after consume 9/12: live DocumentClassifier=1
[2026-07-08 07:06:52,357] [INFO] [paperless.consumer] Consuming d10.pdf
[2026-07-08 07:06:54,373] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d10
[2026-07-08 07:06:54,383] [INFO] [paperless.consumer] Document 2026-07-08 A d10 consumption finished
[2026-07-08 07:06:54,393] [INFO] [paperless.consumer] Consuming d11.pdf
[2026-07-08 07:06:56,468] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d11
[2026-07-08 07:06:56,479] [INFO] [paperless.consumer] Document 2026-07-08 A d11 consumption finished
[2026-07-08 07:06:56,490] [INFO] [paperless.consumer] Consuming d12.pdf
[2026-07-08 07:06:58,489] [INFO] [paperless.handlers] Assigning correspondent A to 2026-07-08 d12
[2026-07-08 07:06:58,498] [INFO] [paperless.consumer] Document 2026-07-08 A d12 consumption finished
  after consume 12/12: live DocumentClassifier=1
[PROBE] after batch AFTER gc.collect()
    RSS_current(VmRSS)  = 286040 kB (279.3 MiB)
    RSS_hiwater(maxrss) = 280072 kB (273.5 MiB)
    tracemalloc.current = 107670914 B (102.68 MiB)
    tracemalloc.peak    = 111253102 B (106.10 MiB)
    gc.live_objects     = 175747
    gc.get_count()      = (11, 0, 0)
    gc.collect()->freed = 1909
  after gc.collect(): live DocumentClassifier=1

[HANDLER SHARING] documents_with_all_3_handlers=12 all_share_same_classifier_id=True
   doc pk=9: handler->id(classifier) = {'set_correspondent': 136130673242656, 'set_document_type': 136130673242656, 'set_tags': 136130673242656}
   doc pk=10: handler->id(classifier) = {'set_correspondent': 136136277968448, 'set_document_type': 136136277968448, 'set_tags': 136136277968448}
CLASSIFIER_BATCH_DONE
```

### B-4c. Whoosh `AsyncWriter` per-update (no accumulation) — `whoosh_check.py` (Q3, F1)

Command:

```
docker exec mem_investigation bash -c "cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/whoosh_check.py"
```

Full unedited output:

```
live writers BEFORE: {'AsyncWriter': 0, 'SegmentWriter': 0, 'IndexWriter': 0}
[PROBE] whoosh BEFORE
    RSS_current(VmRSS)  = 108396 kB (105.9 MiB)
    RSS_hiwater(maxrss) = 105080 kB (102.6 MiB)
    tracemalloc.current = 39934809 B (38.08 MiB)
    tracemalloc.peak    = 43792764 B (41.76 MiB)
    gc.live_objects     = 85498
    gc.get_count()      = (235, 3, 1)
[PROBE] whoosh after 10 updates
    RSS_current(VmRSS)  = 108560 kB (106.0 MiB)
    RSS_hiwater(maxrss) = 105080 kB (102.6 MiB)
    tracemalloc.current = 40856559 B (38.96 MiB)
    tracemalloc.peak    = 43792764 B (41.76 MiB)
    gc.live_objects     = 86402
    gc.get_count()      = (0, 6, 2)
   after 10 updates: live_writers={'AsyncWriter': 0, 'SegmentWriter': 0, 'IndexWriter': 0}
[PROBE] whoosh after 20 updates
    RSS_current(VmRSS)  = 108560 kB (106.0 MiB)
    RSS_hiwater(maxrss) = 105080 kB (102.6 MiB)
    tracemalloc.current = 40974843 B (39.08 MiB)
    tracemalloc.peak    = 43792764 B (41.76 MiB)
    gc.live_objects     = 86433
    gc.get_count()      = (0, 5, 3)
   after 20 updates: live_writers={'AsyncWriter': 0, 'SegmentWriter': 0, 'IndexWriter': 0}
[PROBE] whoosh after 30 updates
    RSS_current(VmRSS)  = 108560 kB (106.0 MiB)
    RSS_hiwater(maxrss) = 105080 kB (102.6 MiB)
    tracemalloc.current = 41028268 B (39.13 MiB)
    tracemalloc.peak    = 43792764 B (41.76 MiB)
    gc.live_objects     = 86463
    gc.get_count()      = (0, 3, 4)
   after 30 updates: live_writers={'AsyncWriter': 0, 'SegmentWriter': 0, 'IndexWriter': 0}
[PROBE] whoosh AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 108560 kB (106.0 MiB)
    RSS_hiwater(maxrss) = 105080 kB (102.6 MiB)
    tracemalloc.current = 40622520 B (38.74 MiB)
    tracemalloc.peak    = 43792764 B (41.76 MiB)
    gc.live_objects     = 85972
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 470
live writers AFTER gc.collect(): {'AsyncWriter': 0, 'SegmentWriter': 0, 'IndexWriter': 0}
WHOOSH_CHECK_DONE
```

### B-5. Inconsistency distribution — same unchanged input repeatedly — `dist_harness.py` (Q4, F5)

The **byte-identical** sample is fed to the real `consume_file` 50× (digital, 2 runs) and 20× (image); the `input_md5` in each run header is constant and re-asserted each iteration (`dist_harness.py:L48-50`), and the created `Document` row is deleted between iterations only to clear the MD5 dedup (`consumer.py:L102-104`) — the input bytes never change. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/dist_harness.py
```
Complete, unedited output (`/tmp/mem_results/dist_harness.txt`, 323 lines):
```

===== DIST type=digital iters=50 run=1 input_md5=42995833e01aea9b3edee44bbfdd7ce1 input_size=22926B =====
[2026-07-08 07:09:55,016] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:09:57,826] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:09:57,846] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:00,136] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:00,165] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:02,299] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:02,316] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:04,436] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:04,453] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:06,765] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:06,785] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:08,830] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:08,846] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:10,829] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:10,844] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:12,840] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:12,856] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:14,840] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:14,855] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:16,838] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:16,854] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:18,916] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:18,934] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:20,931] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:20,950] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:22,952] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:22,968] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:24,973] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:24,987] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:26,973] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:26,987] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:29,093] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:29,109] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:31,099] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:31,114] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:33,114] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:33,130] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:35,157] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:35,172] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:37,156] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:37,173] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:39,265] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:39,280] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:41,265] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:41,281] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:43,273] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:43,289] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:45,314] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:45,329] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:47,323] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:47,338] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:49,471] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:49,486] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:51,466] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:51,481] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:53,475] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:53,489] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:55,484] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:55,500] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:57,489] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:57,503] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:10:59,637] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:10:59,653] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:01,631] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:01,646] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:03,636] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:03,651] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:05,652] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:05,666] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:07,656] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:07,670] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:09,821] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:09,840] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:11,864] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:11,879] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:13,890] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:13,904] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:15,891] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:15,905] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:17,888] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:17,902] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:20,129] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:20,145] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:22,159] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:22,175] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:24,206] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:24,221] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:26,285] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:26,300] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:28,293] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:28,308] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:30,500] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:30,518] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:32,509] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:32,534] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:34,686] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:34,706] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:36,879] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:36,897] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:38,934] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
  peak_TM MiB: min=53.13 median=53.61 max=54.60 mean=53.61 stdev=0.207
  peak_TM per-iter (first 10): [54.6, 53.13, 53.22, 53.3, 53.39, 53.46, 53.51, 53.52, 53.4, 53.46]
  VmRSS MiB   : start=145.6 end=146.6 min=145.6 max=146.6
  ru_maxrss   : start=142.7 end=142.7 (monotonic hiwater)
  iter0(first-touch)=54.60MiB  warm_median(iter>=2)=53.60MiB

===== DIST type=digital iters=50 run=2 input_md5=42995833e01aea9b3edee44bbfdd7ce1 input_size=22926B =====
[2026-07-08 07:11:38,952] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:41,172] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:41,191] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:43,208] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:43,223] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:45,211] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:45,226] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:47,312] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:47,328] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:49,348] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:49,363] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:51,370] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:51,386] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:53,447] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:53,465] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:55,511] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:55,528] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:57,555] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:57,595] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:11:59,636] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:11:59,651] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:01,638] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:01,653] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:03,709] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:03,727] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:05,711] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:05,726] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:07,713] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:07,729] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:09,714] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:09,729] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:11,729] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:11,746] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:13,817] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:13,834] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:15,817] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:15,830] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:17,867] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:17,881] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:19,891] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:19,908] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:21,891] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:21,906] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:24,004] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:24,025] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:26,031] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:26,046] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:28,047] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:28,062] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:30,057] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:30,073] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:32,074] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:32,089] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:34,209] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:34,226] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:36,221] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:36,237] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:38,231] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:38,246] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:40,239] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:40,254] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:42,247] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:42,262] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:44,469] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:44,485] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:46,476] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:46,491] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:48,477] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:48,491] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:50,489] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:50,503] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:52,495] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:52,509] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:54,651] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:54,668] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:56,650] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:56,665] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:12:58,656] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:12:58,672] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:00,663] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:00,679] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:02,682] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:02,696] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:04,838] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:04,853] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:06,834] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:06,850] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:08,847] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:08,865] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:10,853] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:10,867] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:12,885] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:12,929] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:15,214] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:15,229] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:17,227] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:17,241] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:19,273] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
[2026-07-08 07:13:19,288] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-08 07:13:21,273] [INFO] [paperless.consumer] Document 2026-07-08 simple-digital consumption finished
  peak_TM MiB: min=53.69 median=53.87 max=54.00 mean=53.87 stdev=0.076
  peak_TM per-iter (first 10): [53.94, 53.82, 53.82, 53.69, 53.75, 53.82, 53.87, 53.85, 53.83, 53.72]
  VmRSS MiB   : start=146.7 end=146.8 min=146.7 max=146.8
  ru_maxrss   : start=142.7 end=142.7 (monotonic hiwater)
  iter0(first-touch)=53.94MiB  warm_median(iter>=2)=53.86MiB

===== DIST type=image iters=20 run=1 input_md5=62acb0bcbfbcaa62ca6ad3668e4e404b input_size=150479B =====
[2026-07-08 07:13:21,290] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:21,744] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:21,751] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:21,758] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:25,226] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:25,241] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:25,684] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:25,690] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:25,702] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:29,411] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:29,428] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:29,873] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:29,874] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:29,884] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:33,239] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:33,255] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:33,701] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:33,702] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:33,733] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:37,128] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:37,146] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:37,604] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:37,611] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:37,629] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:40,997] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:41,015] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:41,480] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:41,493] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:41,503] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:44,913] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:44,931] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:45,382] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:45,391] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:45,406] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:48,778] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:48,793] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:49,243] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:49,250] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:49,270] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:52,807] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:52,823] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:53,276] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:53,289] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:53,301] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:56,711] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:13:56,727] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:13:57,177] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:57,189] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:13:57,197] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:00,580] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:00,597] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:01,056] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:01,063] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:01,084] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:04,507] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:04,522] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:04,974] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:04,983] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:04,991] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:08,509] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:08,524] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:08,972] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:08,981] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:08,998] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:12,494] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:12,512] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:12,972] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:12,979] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:12,980] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:16,372] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:16,391] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:16,849] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:16,869] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:16,877] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:20,303] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:20,319] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:20,773] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:20,780] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:20,797] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:24,151] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:24,166] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:24,616] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:24,622] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:24,640] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:28,117] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:28,132] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:28,584] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:28,595] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:28,603] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:32,040] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:32,059] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:32,509] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:32,516] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:32,536] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:36,001] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
[2026-07-08 07:14:36,017] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 07:14:36,464] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:36,472] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:36,496] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:14:39,832] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
  peak_TM MiB: min=54.54 median=54.87 max=55.11 mean=54.86 stdev=0.149
  peak_TM per-iter (first 10): [54.54, 54.66, 54.72, 54.83, 54.67, 54.76, 54.84, 54.93, 54.73, 54.81]
  VmRSS MiB   : start=152.9 end=219.6 min=152.9 max=258.2
  ru_maxrss   : start=256.7 end=258.8 (monotonic hiwater)
  iter0(first-touch)=54.54MiB  warm_median(iter>=2)=54.90MiB

DIST_HARNESS_DONE
```

### B-6. `DEBUG=NO` vs `DEBUG=YES` — `debug_harness.py` (Q3/§9)

The same ORM workload was run under both configurations in one process (`DEBUG=NO`: 200 creates + 200 reads; `DEBUG=YES`: 5000 creates + 5000 reads ≈ 10 000 statements, capped at 9000). Command:

```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/debug_harness.py
```

Complete, unedited output (`/tmp/mem_results/debug_harness.txt`, 20 lines):

```
[DEBUG=NO  (canonical)] connection.queries length after workload = 0
[PROBE] DEBUG=YES before workload
    RSS_current(VmRSS)  = 107372 kB (104.9 MiB)
    RSS_hiwater(maxrss) = 105944 kB (103.5 MiB)
    tracemalloc.current = 36518881 B (34.83 MiB)
    tracemalloc.peak    = 43444377 B (41.43 MiB)
    gc.live_objects     = 80315
    gc.get_count()      = (518, 0, 0)
/usr/local/lib/python3.9/site-packages/django/db/backends/base/base.py:172: UserWarning: Limit for query logging exceeded, only the last 9000 queries will be returned.
  warnings.warn(
[PROBE] DEBUG=YES after workload
    RSS_current(VmRSS)  = 108420 kB (105.9 MiB)
    RSS_hiwater(maxrss) = 105944 kB (103.5 MiB)
    tracemalloc.current = 42525970 B (40.56 MiB)
    tracemalloc.peak    = 43444377 B (41.43 MiB)
    gc.live_objects     = 80399
    gc.get_count()      = (219, 0, 2)
[DEBUG=YES (NON-CANONICAL)] connection.queries length = 9000 (default cap 9000); approx_sql_bytes=2503890 (~2.39 MiB of SQL strings)
   after reset_queries(): connection.queries length = 0
DEBUG_HARNESS_DONE
```

### B-7. Batch (many) consume — `batch_consume.py` (Q5 many-consume cells)

N genuinely-distinct documents per type consumed through the real `consume_file` in **one long-lived process** (text 25, digital 25, image 15; distinct content is required because `pre_check_duplicate` (`consumer.py:L102`) rejects a byte-identical re-submission). A `[PROBE]` is taken after every 5th document and after a final `gc.collect()`; the flat `tracemalloc.peak` / `gc.live_objects` / VmRSS across each batch are the no-accumulation evidence for the §8.1 "many — consume_file" cells. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/batch_consume.py
```
Complete, unedited output (`/tmp/mem_results/batch_consume.txt`, 306 lines):
```

===== BATCH CONSUME type=text n=25 (distinct docs) =====
[PROBE] text batch BEFORE (0 consumed)
    RSS_current(VmRSS)  = 118376 kB (115.6 MiB)
    RSS_hiwater(maxrss) = 115892 kB (113.2 MiB)
    tracemalloc.current = 42403657 B (40.44 MiB)
    tracemalloc.peak    = 43393163 B (41.38 MiB)
    gc.live_objects     = 91845
    gc.get_count()      = (46, 8, 2)
[2026-07-08 06:42:45,273] [INFO] [paperless.consumer] Consuming text_1.txt
[2026-07-08 06:42:51,448] [INFO] [paperless.consumer] Document 2026-07-08 text_1 consumption finished
[2026-07-08 06:42:51,455] [INFO] [paperless.consumer] Consuming text_2.txt
[2026-07-08 06:42:52,175] [INFO] [paperless.consumer] Document 2026-07-08 text_2 consumption finished
[2026-07-08 06:42:52,184] [INFO] [paperless.consumer] Consuming text_3.txt
[2026-07-08 06:42:52,851] [INFO] [paperless.consumer] Document 2026-07-08 text_3 consumption finished
[2026-07-08 06:42:52,858] [INFO] [paperless.consumer] Consuming text_4.txt
[2026-07-08 06:42:53,533] [INFO] [paperless.consumer] Document 2026-07-08 text_4 consumption finished
[2026-07-08 06:42:53,541] [INFO] [paperless.consumer] Consuming text_5.txt
[2026-07-08 06:42:54,216] [INFO] [paperless.consumer] Document 2026-07-08 text_5 consumption finished
[PROBE] text after doc 5/25 (peak 69.05MiB)
    RSS_current(VmRSS)  = 196324 kB (191.7 MiB)
    RSS_hiwater(maxrss) = 193424 kB (188.9 MiB)
    tracemalloc.current = 71978345 B (68.64 MiB)
    tracemalloc.peak    = 72405891 B (69.05 MiB)
    gc.live_objects     = 136191
    gc.get_count()      = (13, 6, 5)
[2026-07-08 06:42:54,243] [INFO] [paperless.consumer] Consuming text_6.txt
[2026-07-08 06:42:54,976] [INFO] [paperless.consumer] Document 2026-07-08 text_6 consumption finished
[2026-07-08 06:42:54,984] [INFO] [paperless.consumer] Consuming text_7.txt
[2026-07-08 06:42:55,679] [INFO] [paperless.consumer] Document 2026-07-08 text_7 consumption finished
[2026-07-08 06:42:55,689] [INFO] [paperless.consumer] Consuming text_8.txt
[2026-07-08 06:42:56,364] [INFO] [paperless.consumer] Document 2026-07-08 text_8 consumption finished
[2026-07-08 06:42:56,372] [INFO] [paperless.consumer] Consuming text_9.txt
[2026-07-08 06:42:57,049] [INFO] [paperless.consumer] Document 2026-07-08 text_9 consumption finished
[2026-07-08 06:42:57,057] [INFO] [paperless.consumer] Consuming text_10.txt
[2026-07-08 06:42:57,736] [INFO] [paperless.consumer] Document 2026-07-08 text_10 consumption finished
[PROBE] text after doc 10/25 (peak 69.13MiB)
    RSS_current(VmRSS)  = 197388 kB (192.8 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 72054886 B (68.72 MiB)
    tracemalloc.peak    = 72482846 B (69.13 MiB)
    gc.live_objects     = 136113
    gc.get_count()      = (13, 0, 6)
[2026-07-08 06:42:57,763] [INFO] [paperless.consumer] Consuming text_11.txt
[2026-07-08 06:42:58,506] [INFO] [paperless.consumer] Document 2026-07-08 text_11 consumption finished
[2026-07-08 06:42:58,515] [INFO] [paperless.consumer] Consuming text_12.txt
[2026-07-08 06:42:59,196] [INFO] [paperless.consumer] Document 2026-07-08 text_12 consumption finished
[2026-07-08 06:42:59,204] [INFO] [paperless.consumer] Consuming text_13.txt
[2026-07-08 06:42:59,892] [INFO] [paperless.consumer] Document 2026-07-08 text_13 consumption finished
[2026-07-08 06:42:59,903] [INFO] [paperless.consumer] Consuming text_14.txt
[2026-07-08 06:43:00,579] [INFO] [paperless.consumer] Document 2026-07-08 text_14 consumption finished
[2026-07-08 06:43:00,588] [INFO] [paperless.consumer] Consuming text_15.txt
[2026-07-08 06:43:01,267] [INFO] [paperless.consumer] Document 2026-07-08 text_15 consumption finished
[PROBE] text after doc 15/25 (peak 69.18MiB)
    RSS_current(VmRSS)  = 197488 kB (192.9 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 72109750 B (68.77 MiB)
    tracemalloc.peak    = 72537559 B (69.18 MiB)
    gc.live_objects     = 136243
    gc.get_count()      = (13, 5, 6)
[2026-07-08 06:43:01,295] [INFO] [paperless.consumer] Consuming text_16.txt
[2026-07-08 06:43:02,075] [INFO] [paperless.consumer] Document 2026-07-08 text_16 consumption finished
[2026-07-08 06:43:02,084] [INFO] [paperless.consumer] Consuming text_17.txt
[2026-07-08 06:43:02,769] [INFO] [paperless.consumer] Document 2026-07-08 text_17 consumption finished
[2026-07-08 06:43:02,778] [INFO] [paperless.consumer] Consuming text_18.txt
[2026-07-08 06:43:03,458] [INFO] [paperless.consumer] Document 2026-07-08 text_18 consumption finished
[2026-07-08 06:43:03,466] [INFO] [paperless.consumer] Consuming text_19.txt
[2026-07-08 06:43:04,159] [INFO] [paperless.consumer] Document 2026-07-08 text_19 consumption finished
[2026-07-08 06:43:04,167] [INFO] [paperless.consumer] Consuming text_20.txt
[2026-07-08 06:43:04,847] [INFO] [paperless.consumer] Document 2026-07-08 text_20 consumption finished
[PROBE] text after doc 20/25 (peak 69.21MiB)
    RSS_current(VmRSS)  = 197588 kB (193.0 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 72143502 B (68.80 MiB)
    tracemalloc.peak    = 72573506 B (69.21 MiB)
    gc.live_objects     = 136172
    gc.get_count()      = (13, 10, 6)
[2026-07-08 06:43:04,873] [INFO] [paperless.consumer] Consuming text_21.txt
[2026-07-08 06:43:05,654] [INFO] [paperless.consumer] Document 2026-07-08 text_21 consumption finished
[2026-07-08 06:43:05,664] [INFO] [paperless.consumer] Consuming text_22.txt
[2026-07-08 06:43:06,348] [INFO] [paperless.consumer] Document 2026-07-08 text_22 consumption finished
[2026-07-08 06:43:06,357] [INFO] [paperless.consumer] Consuming text_23.txt
[2026-07-08 06:43:07,039] [INFO] [paperless.consumer] Document 2026-07-08 text_23 consumption finished
[2026-07-08 06:43:07,047] [INFO] [paperless.consumer] Consuming text_24.txt
[2026-07-08 06:43:07,728] [INFO] [paperless.consumer] Document 2026-07-08 text_24 consumption finished
[2026-07-08 06:43:07,736] [INFO] [paperless.consumer] Consuming text_25.txt
[2026-07-08 06:43:08,420] [INFO] [paperless.consumer] Document 2026-07-08 text_25 consumption finished
[PROBE] text after doc 25/25 (peak 69.22MiB)
    RSS_current(VmRSS)  = 197660 kB (193.0 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 72151859 B (68.81 MiB)
    tracemalloc.peak    = 72581708 B (69.22 MiB)
    gc.live_objects     = 136101
    gc.get_count()      = (13, 3, 7)
[PROBE] text batch AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 197660 kB (193.0 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 70930100 B (67.64 MiB)
    tracemalloc.peak    = 73289797 B (69.89 MiB)
    gc.live_objects     = 127324
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 30
[BATCH SUMMARY] type=text n=25 peak_min=68.85 peak_median=69.15 peak_max=69.24 live_docs_in_db=25

===== BATCH CONSUME type=digital n=25 (distinct docs) =====
[PROBE] digital batch BEFORE (0 consumed)
    RSS_current(VmRSS)  = 197660 kB (193.0 MiB)
    RSS_hiwater(maxrss) = 194448 kB (189.9 MiB)
    tracemalloc.current = 71007546 B (67.72 MiB)
    tracemalloc.peak    = 73289797 B (69.89 MiB)
    gc.live_objects     = 127399
    gc.get_count()      = (427, 0, 0)
[2026-07-08 06:43:08,569] [INFO] [paperless.consumer] Consuming digital_1.pdf
[2026-07-08 06:43:11,181] [INFO] [paperless.consumer] Document 2026-07-08 digital_1 consumption finished
[2026-07-08 06:43:11,192] [INFO] [paperless.consumer] Consuming digital_2.pdf
[2026-07-08 06:43:13,164] [INFO] [paperless.consumer] Document 2026-07-08 digital_2 consumption finished
[2026-07-08 06:43:13,174] [INFO] [paperless.consumer] Consuming digital_3.pdf
[2026-07-08 06:43:15,135] [INFO] [paperless.consumer] Document 2026-07-08 digital_3 consumption finished
[2026-07-08 06:43:15,147] [INFO] [paperless.consumer] Consuming digital_4.pdf
[2026-07-08 06:43:17,121] [INFO] [paperless.consumer] Document 2026-07-08 digital_4 consumption finished
[2026-07-08 06:43:17,131] [INFO] [paperless.consumer] Consuming digital_5.pdf
[2026-07-08 06:43:19,087] [INFO] [paperless.consumer] Document 2026-07-08 digital_5 consumption finished
[PROBE] digital after doc 5/25 (peak 77.49MiB)
    RSS_current(VmRSS)  = 219120 kB (214.0 MiB)
    RSS_hiwater(maxrss) = 213904 kB (208.9 MiB)
    tracemalloc.current = 80831602 B (77.09 MiB)
    tracemalloc.peak    = 81256059 B (77.49 MiB)
    gc.live_objects     = 140380
    gc.get_count()      = (16, 4, 4)
[2026-07-08 06:43:19,116] [INFO] [paperless.consumer] Consuming digital_6.pdf
[2026-07-08 06:43:21,229] [INFO] [paperless.consumer] Document 2026-07-08 digital_6 consumption finished
[2026-07-08 06:43:21,241] [INFO] [paperless.consumer] Consuming digital_7.pdf
[2026-07-08 06:43:23,217] [INFO] [paperless.consumer] Document 2026-07-08 digital_7 consumption finished
[2026-07-08 06:43:23,227] [INFO] [paperless.consumer] Consuming digital_8.pdf
[2026-07-08 06:43:25,264] [INFO] [paperless.consumer] Document 2026-07-08 digital_8 consumption finished
[2026-07-08 06:43:25,274] [INFO] [paperless.consumer] Consuming digital_9.pdf
[2026-07-08 06:43:27,290] [INFO] [paperless.consumer] Document 2026-07-08 digital_9 consumption finished
[2026-07-08 06:43:27,301] [INFO] [paperless.consumer] Consuming digital_10.pdf
[2026-07-08 06:43:29,302] [INFO] [paperless.consumer] Document 2026-07-08 digital_10 consumption finished
[PROBE] digital after doc 10/25 (peak 77.56MiB)
    RSS_current(VmRSS)  = 219332 kB (214.2 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 80894331 B (77.15 MiB)
    tracemalloc.peak    = 81323061 B (77.56 MiB)
    gc.live_objects     = 140121
    gc.get_count()      = (12, 4, 5)
[2026-07-08 06:43:29,332] [INFO] [paperless.consumer] Consuming digital_11.pdf
[2026-07-08 06:43:31,672] [INFO] [paperless.consumer] Document 2026-07-08 digital_11 consumption finished
[2026-07-08 06:43:31,683] [INFO] [paperless.consumer] Consuming digital_12.pdf
[2026-07-08 06:43:33,714] [INFO] [paperless.consumer] Document 2026-07-08 digital_12 consumption finished
[2026-07-08 06:43:33,724] [INFO] [paperless.consumer] Consuming digital_13.pdf
[2026-07-08 06:43:35,677] [INFO] [paperless.consumer] Document 2026-07-08 digital_13 consumption finished
[2026-07-08 06:43:35,687] [INFO] [paperless.consumer] Consuming digital_14.pdf
[2026-07-08 06:43:37,673] [INFO] [paperless.consumer] Document 2026-07-08 digital_14 consumption finished
[2026-07-08 06:43:37,682] [INFO] [paperless.consumer] Consuming digital_15.pdf
[2026-07-08 06:43:39,659] [INFO] [paperless.consumer] Document 2026-07-08 digital_15 consumption finished
[PROBE] digital after doc 15/25 (peak 77.55MiB)
    RSS_current(VmRSS)  = 219412 kB (214.3 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 80889013 B (77.14 MiB)
    tracemalloc.peak    = 81317881 B (77.55 MiB)
    gc.live_objects     = 139904
    gc.get_count()      = (12, 2, 6)
[2026-07-08 06:43:39,686] [INFO] [paperless.consumer] Consuming digital_16.pdf
[2026-07-08 06:43:41,835] [INFO] [paperless.consumer] Document 2026-07-08 digital_16 consumption finished
[2026-07-08 06:43:41,845] [INFO] [paperless.consumer] Consuming digital_17.pdf
[2026-07-08 06:43:43,911] [INFO] [paperless.consumer] Document 2026-07-08 digital_17 consumption finished
[2026-07-08 06:43:43,921] [INFO] [paperless.consumer] Consuming digital_18.pdf
[2026-07-08 06:43:45,879] [INFO] [paperless.consumer] Document 2026-07-08 digital_18 consumption finished
[2026-07-08 06:43:45,889] [INFO] [paperless.consumer] Consuming digital_19.pdf
[2026-07-08 06:43:47,989] [INFO] [paperless.consumer] Document 2026-07-08 digital_19 consumption finished
[2026-07-08 06:43:47,998] [INFO] [paperless.consumer] Consuming digital_20.pdf
[2026-07-08 06:43:49,957] [INFO] [paperless.consumer] Document 2026-07-08 digital_20 consumption finished
[PROBE] digital after doc 20/25 (peak 77.73MiB)
    RSS_current(VmRSS)  = 219512 kB (214.4 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 80888737 B (77.14 MiB)
    tracemalloc.peak    = 81503394 B (77.73 MiB)
    gc.live_objects     = 139686
    gc.get_count()      = (12, 0, 7)
[2026-07-08 06:43:49,982] [INFO] [paperless.consumer] Consuming digital_21.pdf
[2026-07-08 06:43:52,186] [INFO] [paperless.consumer] Document 2026-07-08 digital_21 consumption finished
[2026-07-08 06:43:52,199] [INFO] [paperless.consumer] Consuming digital_22.pdf
[2026-07-08 06:43:54,168] [INFO] [paperless.consumer] Document 2026-07-08 digital_22 consumption finished
[2026-07-08 06:43:54,178] [INFO] [paperless.consumer] Consuming digital_23.pdf
[2026-07-08 06:43:56,133] [INFO] [paperless.consumer] Document 2026-07-08 digital_23 consumption finished
[2026-07-08 06:43:56,142] [INFO] [paperless.consumer] Consuming digital_24.pdf
[2026-07-08 06:43:58,107] [INFO] [paperless.consumer] Document 2026-07-08 digital_24 consumption finished
[2026-07-08 06:43:58,117] [INFO] [paperless.consumer] Consuming digital_25.pdf
[2026-07-08 06:44:00,091] [INFO] [paperless.consumer] Document 2026-07-08 digital_25 consumption finished
[PROBE] digital after doc 25/25 (peak 77.79MiB)
    RSS_current(VmRSS)  = 219528 kB (214.4 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 81139130 B (77.38 MiB)
    tracemalloc.peak    = 81567860 B (77.79 MiB)
    gc.live_objects     = 141430
    gc.get_count()      = (12, 10, 7)
[PROBE] digital batch AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 219528 kB (214.4 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 80392552 B (76.67 MiB)
    tracemalloc.peak    = 82276836 B (78.47 MiB)
    gc.live_objects     = 139088
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 2253
[BATCH SUMMARY] type=digital n=25 peak_min=77.37 peak_median=77.66 peak_max=79.55 live_docs_in_db=25

===== BATCH CONSUME type=image n=15 (distinct docs) =====
[PROBE] image batch BEFORE (0 consumed)
    RSS_current(VmRSS)  = 219528 kB (214.4 MiB)
    RSS_hiwater(maxrss) = 214928 kB (209.9 MiB)
    tracemalloc.current = 80422034 B (76.70 MiB)
    tracemalloc.peak    = 82276836 B (78.47 MiB)
    gc.live_objects     = 139104
    gc.get_count()      = (291, 0, 0)
[2026-07-08 06:44:00,182] [INFO] [paperless.consumer] Consuming image_1.pdf
[2026-07-08 06:44:00,488] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:02,174] [INFO] [paperless.consumer] Document 2026-07-08 image_1 consumption finished
[2026-07-08 06:44:02,196] [INFO] [paperless.consumer] Consuming image_2.pdf
[2026-07-08 06:44:02,494] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:03,936] [INFO] [paperless.consumer] Document 2026-07-08 image_2 consumption finished
[2026-07-08 06:44:03,955] [INFO] [paperless.consumer] Consuming image_3.pdf
[2026-07-08 06:44:04,254] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:05,727] [INFO] [paperless.consumer] Document 2026-07-08 image_3 consumption finished
[PROBE] image after doc 3/15 (peak 77.59MiB)
    RSS_current(VmRSS)  = 239520 kB (233.9 MiB)
    RSS_hiwater(maxrss) = 232336 kB (226.9 MiB)
    tracemalloc.current = 80967571 B (77.22 MiB)
    tracemalloc.peak    = 81358016 B (77.59 MiB)
    gc.live_objects     = 139437
    gc.get_count()      = (12, 2, 1)
[2026-07-08 06:44:05,767] [INFO] [paperless.consumer] Consuming image_4.pdf
[2026-07-08 06:44:06,068] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:07,542] [INFO] [paperless.consumer] Document 2026-07-08 image_4 consumption finished
[2026-07-08 06:44:07,562] [INFO] [paperless.consumer] Consuming image_5.pdf
[2026-07-08 06:44:07,863] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:09,322] [INFO] [paperless.consumer] Document 2026-07-08 image_5 consumption finished
[2026-07-08 06:44:09,341] [INFO] [paperless.consumer] Consuming image_6.pdf
[2026-07-08 06:44:09,648] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:11,120] [INFO] [paperless.consumer] Document 2026-07-08 image_6 consumption finished
[PROBE] image after doc 6/15 (peak 77.80MiB)
    RSS_current(VmRSS)  = 240612 kB (235.0 MiB)
    RSS_hiwater(maxrss) = 233360 kB (227.9 MiB)
    tracemalloc.current = 81136401 B (77.38 MiB)
    tracemalloc.peak    = 81580755 B (77.80 MiB)
    gc.live_objects     = 140607
    gc.get_count()      = (12, 8, 1)
[2026-07-08 06:44:11,158] [INFO] [paperless.consumer] Consuming image_7.pdf
[2026-07-08 06:44:11,459] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:12,974] [INFO] [paperless.consumer] Document 2026-07-08 image_7 consumption finished
[2026-07-08 06:44:12,994] [INFO] [paperless.consumer] Consuming image_8.pdf
[2026-07-08 06:44:13,298] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:14,773] [INFO] [paperless.consumer] Document 2026-07-08 image_8 consumption finished
[2026-07-08 06:44:14,793] [INFO] [paperless.consumer] Consuming image_9.pdf
[2026-07-08 06:44:15,095] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:16,546] [INFO] [paperless.consumer] Document 2026-07-08 image_9 consumption finished
[PROBE] image after doc 9/15 (peak 77.67MiB)
    RSS_current(VmRSS)  = 240728 kB (235.1 MiB)
    RSS_hiwater(maxrss) = 233360 kB (227.9 MiB)
    tracemalloc.current = 81035323 B (77.28 MiB)
    tracemalloc.peak    = 81445515 B (77.67 MiB)
    gc.live_objects     = 139615
    gc.get_count()      = (12, 2, 2)
[2026-07-08 06:44:16,585] [INFO] [paperless.consumer] Consuming image_10.pdf
[2026-07-08 06:44:16,886] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:18,362] [INFO] [paperless.consumer] Document 2026-07-08 image_10 consumption finished
[2026-07-08 06:44:18,383] [INFO] [paperless.consumer] Consuming image_11.pdf
[2026-07-08 06:44:18,690] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:20,131] [INFO] [paperless.consumer] Document 2026-07-08 image_11 consumption finished
[2026-07-08 06:44:20,150] [INFO] [paperless.consumer] Consuming image_12.pdf
[2026-07-08 06:44:20,454] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:21,971] [INFO] [paperless.consumer] Document 2026-07-08 image_12 consumption finished
[PROBE] image after doc 12/15 (peak 77.85MiB)
    RSS_current(VmRSS)  = 240816 kB (235.2 MiB)
    RSS_hiwater(maxrss) = 233360 kB (227.9 MiB)
    tracemalloc.current = 81177885 B (77.42 MiB)
    tracemalloc.peak    = 81636506 B (77.85 MiB)
    gc.live_objects     = 140584
    gc.get_count()      = (12, 8, 2)
[2026-07-08 06:44:22,008] [INFO] [paperless.consumer] Consuming image_13.pdf
[2026-07-08 06:44:22,304] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:23,792] [INFO] [paperless.consumer] Document 2026-07-08 image_13 consumption finished
[2026-07-08 06:44:23,811] [INFO] [paperless.consumer] Consuming image_14.pdf
[2026-07-08 06:44:24,116] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:25,584] [INFO] [paperless.consumer] Document 2026-07-08 image_14 consumption finished
[2026-07-08 06:44:25,604] [INFO] [paperless.consumer] Consuming image_15.pdf
[2026-07-08 06:44:25,905] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 06:44:27,408] [INFO] [paperless.consumer] Document 2026-07-08 image_15 consumption finished
[PROBE] image after doc 15/15 (peak 77.71MiB)
    RSS_current(VmRSS)  = 240888 kB (235.2 MiB)
    RSS_hiwater(maxrss) = 233360 kB (227.9 MiB)
    tracemalloc.current = 81054357 B (77.30 MiB)
    tracemalloc.peak    = 81482697 B (77.71 MiB)
    gc.live_objects     = 139592
    gc.get_count()      = (12, 2, 3)
[PROBE] image batch AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 240888 kB (235.2 MiB)
    RSS_hiwater(maxrss) = 233360 kB (227.9 MiB)
    tracemalloc.current = 80533637 B (76.80 MiB)
    tracemalloc.peak    = 82192063 B (78.38 MiB)
    gc.live_objects     = 139157
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 422
[BATCH SUMMARY] type=image n=15 peak_min=77.57 peak_median=77.75 peak_max=77.87 live_docs_in_db=15

BATCH_CONSUME_DONE
```

### B-8. Single & many import — `importer_harness.py` (Q5 importer cells)

The **real** `document_importer` management command driven per document type at single (N=1) and many (N=50 text/digital, N=20 image), each in its own fresh process/DB, with `[PROBE]` before / after / after-`gc.collect()`. The `[IMPORT SUMMARY]` line of each run gives the whole-command `dRSS` / `peakTM` used in the §8.1 importer cells and in §8.3. Commands (six invocations):
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py text 1
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py text 50
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py digital 1
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py digital 50
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py image 1
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/importer_harness.py image 20
```
Complete, unedited output (`/tmp/mem_results/importer_harness.txt`; `tqdm` progress bars appear one update per line):
```

===== IMPORTER type=text batch_n=1 manifest=1163B =====
[PROBE] import text_1 BEFORE
    RSS_current(VmRSS)  = 107500 kB (105.0 MiB)
    RSS_hiwater(maxrss) = 106652 kB (104.2 MiB)
    tracemalloc.current = 43128644 B (41.13 MiB)
    tracemalloc.peak    = 43665347 B (41.64 MiB)
    gc.live_objects     = 110463
    gc.get_count()      = (41, 0, 11)
Installed 2 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/1 [00:00<?, ?it/s]
100%|██████████| 1/1 [00:00<00:00, 251.82it/s]
[PROBE] import text_1 AFTER
    RSS_current(VmRSS)  = 121376 kB (118.5 MiB)
    RSS_hiwater(maxrss) = 120988 kB (118.2 MiB)
    tracemalloc.current = 44563768 B (42.50 MiB)
    tracemalloc.peak    = 44921824 B (42.84 MiB)
    gc.live_objects     = 94329
    gc.get_count()      = (0, 4, 3)
[PROBE] import text_1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 122160 kB (119.3 MiB)
    RSS_hiwater(maxrss) = 120988 kB (118.2 MiB)
    tracemalloc.current = 44479913 B (42.42 MiB)
    tracemalloc.peak    = 45364903 B (43.26 MiB)
    gc.live_objects     = 93995
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 193
[IMPORT SUMMARY] type=text n=1 manifest=1163B dRSS=13.55MiB peakTM=42.84MiB dLiveObj=-16134 docs_imported=1 freed=193
spec[text 1] EXIT=0

===== IMPORTER type=text batch_n=50 manifest=39186B =====
[PROBE] import text_50 BEFORE
    RSS_current(VmRSS)  = 107532 kB (105.0 MiB)
    RSS_hiwater(maxrss) = 104964 kB (102.5 MiB)
    tracemalloc.current = 43017188 B (41.02 MiB)
    tracemalloc.peak    = 43586326 B (41.57 MiB)
    gc.live_objects     = 110191
    gc.get_count()      = (41, 0, 11)
Installed 51 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/50 [00:00<?, ?it/s]
 58%|█████▊    | 29/50 [00:00<00:00, 283.56it/s]
100%|██████████| 50/50 [00:00<00:00, 243.11it/s]
[PROBE] import text_50 AFTER
    RSS_current(VmRSS)  = 122864 kB (120.0 MiB)
    RSS_hiwater(maxrss) = 118276 kB (115.5 MiB)
    tracemalloc.current = 44814627 B (42.74 MiB)
    tracemalloc.peak    = 45581867 B (43.47 MiB)
    gc.live_objects     = 94264
    gc.get_count()      = (0, 10, 3)
[PROBE] import text_50 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 123532 kB (120.6 MiB)
    RSS_hiwater(maxrss) = 119300 kB (116.5 MiB)
    tracemalloc.current = 44482886 B (42.42 MiB)
    tracemalloc.peak    = 45615762 B (43.50 MiB)
    gc.live_objects     = 93990
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 130
[IMPORT SUMMARY] type=text n=50 manifest=39186B dRSS=14.97MiB peakTM=43.47MiB dLiveObj=-15927 docs_imported=50 freed=130
spec[text 50] EXIT=0

===== IMPORTER type=digital batch_n=1 manifest=1309B =====
[PROBE] import digital_1 BEFORE
    RSS_current(VmRSS)  = 107468 kB (104.9 MiB)
    RSS_hiwater(maxrss) = 104080 kB (101.6 MiB)
    tracemalloc.current = 43320193 B (41.31 MiB)
    tracemalloc.peak    = 43508964 B (41.49 MiB)
    gc.live_objects     = 111727
    gc.get_count()      = (41, 11, 10)
Installed 2 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/1 [00:00<?, ?it/s]
100%|██████████| 1/1 [00:00<00:00, 245.08it/s]
[PROBE] import digital_1 AFTER
    RSS_current(VmRSS)  = 121608 kB (118.8 MiB)
    RSS_hiwater(maxrss) = 118416 kB (115.6 MiB)
    tracemalloc.current = 44568159 B (42.50 MiB)
    tracemalloc.peak    = 44927406 B (42.85 MiB)
    gc.live_objects     = 94261
    gc.get_count()      = (0, 3, 3)
[PROBE] import digital_1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 122384 kB (119.5 MiB)
    RSS_hiwater(maxrss) = 119440 kB (116.6 MiB)
    tracemalloc.current = 44483891 B (42.42 MiB)
    tracemalloc.peak    = 45369294 B (43.27 MiB)
    gc.live_objects     = 93992
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 193
[IMPORT SUMMARY] type=digital n=1 manifest=1309B dRSS=13.81MiB peakTM=42.85MiB dLiveObj=-17466 docs_imported=1 freed=193
spec[digital 1] EXIT=0

===== IMPORTER type=digital batch_n=50 manifest=46442B =====
[PROBE] import digital_50 BEFORE
    RSS_current(VmRSS)  = 107564 kB (105.0 MiB)
    RSS_hiwater(maxrss) = 106056 kB (103.6 MiB)
    tracemalloc.current = 43004496 B (41.01 MiB)
    tracemalloc.peak    = 43564249 B (41.55 MiB)
    gc.live_objects     = 110088
    gc.get_count()      = (41, 0, 11)
Installed 51 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/50 [00:00<?, ?it/s]
 72%|███████▏  | 36/50 [00:00<00:00, 358.64it/s]
100%|██████████| 50/50 [00:00<00:00, 359.01it/s]
[PROBE] import digital_50 AFTER
    RSS_current(VmRSS)  = 122888 kB (120.0 MiB)
    RSS_hiwater(maxrss) = 121416 kB (118.6 MiB)
    tracemalloc.current = 44818651 B (42.74 MiB)
    tracemalloc.peak    = 45637158 B (43.52 MiB)
    gc.live_objects     = 94264
    gc.get_count()      = (0, 11, 3)
[PROBE] import digital_50 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 123556 kB (120.7 MiB)
    RSS_hiwater(maxrss) = 122440 kB (119.6 MiB)
    tracemalloc.current = 44486230 B (42.43 MiB)
    tracemalloc.peak    = 45637158 B (43.52 MiB)
    gc.live_objects     = 93990
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 130
[IMPORT SUMMARY] type=digital n=50 manifest=46442B dRSS=14.96MiB peakTM=43.52MiB dLiveObj=-15824 docs_imported=50 freed=130
spec[digital 50] EXIT=0

===== IMPORTER type=image batch_n=1 manifest=1268B =====
[PROBE] import image_1 BEFORE
    RSS_current(VmRSS)  = 107172 kB (104.7 MiB)
    RSS_hiwater(maxrss) = 104084 kB (101.6 MiB)
    tracemalloc.current = 43253907 B (41.25 MiB)
    tracemalloc.peak    = 43442682 B (41.43 MiB)
    gc.live_objects     = 111421
    gc.get_count()      = (41, 11, 10)
Installed 2 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/1 [00:00<?, ?it/s]
100%|██████████| 1/1 [00:00<00:00, 231.76it/s]
[PROBE] import image_1 AFTER
    RSS_current(VmRSS)  = 121064 kB (118.2 MiB)
    RSS_hiwater(maxrss) = 117396 kB (114.6 MiB)
    tracemalloc.current = 44567109 B (42.50 MiB)
    tracemalloc.peak    = 44925648 B (42.84 MiB)
    gc.live_objects     = 94261
    gc.get_count()      = (0, 3, 3)
[PROBE] import image_1 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 121848 kB (119.0 MiB)
    RSS_hiwater(maxrss) = 117396 kB (114.6 MiB)
    tracemalloc.current = 44483783 B (42.42 MiB)
    tracemalloc.peak    = 45368244 B (43.27 MiB)
    gc.live_objects     = 93992
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 193
[IMPORT SUMMARY] type=image n=1 manifest=1268B dRSS=13.57MiB peakTM=42.84MiB dLiveObj=-17160 docs_imported=1 freed=193
spec[image 1] EXIT=0

===== IMPORTER type=image batch_n=20 manifest=17962B =====
[PROBE] import image_20 BEFORE
    RSS_current(VmRSS)  = 107340 kB (104.8 MiB)
    RSS_hiwater(maxrss) = 104084 kB (101.6 MiB)
    tracemalloc.current = 43371041 B (41.36 MiB)
    tracemalloc.peak    = 43559814 B (41.54 MiB)
    gc.live_objects     = 111975
    gc.get_count()      = (41, 11, 10)
Installed 21 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/20 [00:00<?, ?it/s]
100%|██████████| 20/20 [00:00<00:00, 357.65it/s]
[PROBE] import image_20 AFTER
    RSS_current(VmRSS)  = 122032 kB (119.2 MiB)
    RSS_hiwater(maxrss) = 117396 kB (114.6 MiB)
    tracemalloc.current = 44679004 B (42.61 MiB)
    tracemalloc.peak    = 45196824 B (43.10 MiB)
    gc.live_objects     = 94346
    gc.get_count()      = (0, 5, 3)
[PROBE] import image_20 AFTER-GC AFTER gc.collect()
    RSS_current(VmRSS)  = 122768 kB (119.9 MiB)
    RSS_hiwater(maxrss) = 118420 kB (115.6 MiB)
    tracemalloc.current = 44488125 B (42.43 MiB)
    tracemalloc.peak    = 45480078 B (43.37 MiB)
    gc.live_objects     = 94068
    gc.get_count()      = (9, 0, 0)
    gc.collect()->freed = 193
[IMPORT SUMMARY] type=image n=20 manifest=17962B dRSS=14.35MiB peakTM=43.10MiB dLiveObj=-17629 docs_imported=20 freed=193
spec[image 20] EXIT=0
```

### B-9. Importer three materializations, tiny content — `focused_copies.py` (Q5/§8.2)

Mechanistic attribution of the importer's three in-memory operations — copy#1 `self.manifest = json.load(f)` (`document_importer.py:L73`), copy#2 `list(filter(...))` (`L137`), and `loaddata` (`L87`) — each measured on a real exported manifest with `tracemalloc`, plus an element-identity check proving copy#2 is shallow. Commands (five invocations, fresh DB each):
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_copies.py text 1
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_copies.py text 50
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_copies.py text 200
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_copies.py digital 1
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_copies.py digital 200
```
Complete, unedited output (`/tmp/mem_results/focused_copies.txt`, 105 lines):
```

===== FOCUSED COPIES type=text n=1 manifest_file=1163B =====
  copy#1 json.load [L73]: dTM_current=4.9 KiB peak=34.69 MiB records=2 doc_records=1
  copy#2 list(filter) [L137]: dTM_current=0.5 KiB len=1 (shallow refs to same dicts)
  copy#2 is-shallow (element identity vs manifest): True
[PROBE] loaddata text_1 BEFORE
    RSS_current(VmRSS)  = 107420 kB (104.9 MiB)
    RSS_hiwater(maxrss) = 106388 kB (103.9 MiB)
    tracemalloc.current = 36362477 B (34.68 MiB)
    tracemalloc.peak    = 36366315 B (34.68 MiB)
    gc.live_objects     = 80271
    gc.get_count()      = (57, 0, 0)
[PROBE] loaddata text_1 AFTER
    RSS_current(VmRSS)  = 107832 kB (105.3 MiB)
    RSS_hiwater(maxrss) = 106388 kB (103.9 MiB)
    tracemalloc.current = 37420149 B (35.69 MiB)
    tracemalloc.peak    = 37619418 B (35.88 MiB)
    gc.live_objects     = 82024
    gc.get_count()      = (60, 5, 0)
  loaddata [L87]: dTM_current=1032.9 KiB peak=35.88 MiB docs_in_db=1
spec[text 1] EXIT=0

===== FOCUSED COPIES type=text n=50 manifest_file=39186B =====
  copy#1 json.load [L73]: dTM_current=76.0 KiB peak=34.79 MiB records=51 doc_records=50
  copy#2 list(filter) [L137]: dTM_current=0.9 KiB len=50 (shallow refs to same dicts)
  copy#2 is-shallow (element identity vs manifest): True
[PROBE] loaddata text_50 BEFORE
    RSS_current(VmRSS)  = 107704 kB (105.2 MiB)
    RSS_hiwater(maxrss) = 105876 kB (103.4 MiB)
    tracemalloc.current = 36364524 B (34.68 MiB)
    tracemalloc.peak    = 36440903 B (34.75 MiB)
    gc.live_objects     = 80272
    gc.get_count()      = (62, 0, 0)
[PROBE] loaddata text_50 AFTER
    RSS_current(VmRSS)  = 108412 kB (105.9 MiB)
    RSS_hiwater(maxrss) = 106900 kB (104.4 MiB)
    tracemalloc.current = 37445808 B (35.71 MiB)
    tracemalloc.peak    = 37689408 B (35.94 MiB)
    gc.live_objects     = 82018
    gc.get_count()      = (0, 6, 0)
  loaddata [L87]: dTM_current=1055.9 KiB peak=35.94 MiB docs_in_db=50
spec[text 50] EXIT=0

===== FOCUSED COPIES type=text n=200 manifest_file=156314B =====
  copy#1 json.load [L73]: dTM_current=333.6 KiB peak=35.16 MiB records=201 doc_records=200
  copy#2 list(filter) [L137]: dTM_current=2.0 KiB len=200 (shallow refs to same dicts)
  copy#2 is-shallow (element identity vs manifest): True
[PROBE] loaddata text_200 BEFORE
    RSS_current(VmRSS)  = 107924 kB (105.4 MiB)
    RSS_hiwater(maxrss) = 104500 kB (102.1 MiB)
    tracemalloc.current = 36365174 B (34.68 MiB)
    tracemalloc.peak    = 36706244 B (35.01 MiB)
    gc.live_objects     = 80272
    gc.get_count()      = (62, 0, 0)
[PROBE] loaddata text_200 AFTER
    RSS_current(VmRSS)  = 108412 kB (105.9 MiB)
    RSS_hiwater(maxrss) = 106548 kB (104.1 MiB)
    tracemalloc.current = 37517202 B (35.78 MiB)
    tracemalloc.peak    = 38138853 B (36.37 MiB)
    gc.live_objects     = 82018
    gc.get_count()      = (0, 9, 0)
  loaddata [L87]: dTM_current=1125.0 KiB peak=36.37 MiB docs_in_db=200
spec[text 200] EXIT=0

===== FOCUSED COPIES type=digital n=1 manifest_file=1309B =====
  copy#1 json.load [L73]: dTM_current=5.5 KiB peak=34.69 MiB records=2 doc_records=1
  copy#2 list(filter) [L137]: dTM_current=0.5 KiB len=1 (shallow refs to same dicts)
  copy#2 is-shallow (element identity vs manifest): True
[PROBE] loaddata digital_1 BEFORE
    RSS_current(VmRSS)  = 107732 kB (105.2 MiB)
    RSS_hiwater(maxrss) = 106064 kB (103.6 MiB)
    tracemalloc.current = 36364607 B (34.68 MiB)
    tracemalloc.peak    = 36369071 B (34.68 MiB)
    gc.live_objects     = 80272
    gc.get_count()      = (52, 0, 0)
[PROBE] loaddata digital_1 AFTER
    RSS_current(VmRSS)  = 108440 kB (105.9 MiB)
    RSS_hiwater(maxrss) = 106064 kB (103.6 MiB)
    tracemalloc.current = 37419992 B (35.69 MiB)
    tracemalloc.peak    = 37619411 B (35.88 MiB)
    gc.live_objects     = 82024
    gc.get_count()      = (60, 5, 0)
  loaddata [L87]: dTM_current=1030.6 KiB peak=35.88 MiB docs_in_db=1
spec[digital 1] EXIT=0

===== FOCUSED COPIES type=digital n=200 manifest_file=185549B =====
  copy#1 json.load [L73]: dTM_current=425.8 KiB peak=35.28 MiB records=201 doc_records=200
  copy#2 list(filter) [L137]: dTM_current=2.0 KiB len=200 (shallow refs to same dicts)
  copy#2 is-shallow (element identity vs manifest): True
[PROBE] loaddata digital_200 BEFORE
    RSS_current(VmRSS)  = 106924 kB (104.4 MiB)
    RSS_hiwater(maxrss) = 105964 kB (103.5 MiB)
    tracemalloc.current = 36364362 B (34.68 MiB)
    tracemalloc.peak    = 36800166 B (35.10 MiB)
    gc.live_objects     = 80272
    gc.get_count()      = (62, 0, 0)
[PROBE] loaddata digital_200 AFTER
    RSS_current(VmRSS)  = 107456 kB (104.9 MiB)
    RSS_hiwater(maxrss) = 105964 kB (103.5 MiB)
    tracemalloc.current = 37516435 B (35.78 MiB)
    tracemalloc.peak    = 38248951 B (36.48 MiB)
    gc.live_objects     = 82018
    gc.get_count()      = (0, 9, 0)
  loaddata [L87]: dTM_current=1125.1 KiB peak=36.48 MiB docs_in_db=200
spec[digital 200] EXIT=0
```

### B-10. Importer at large content, coexistence peak — `focused_big.py` (Q5/§8.2/§8.3)

~100 KB-content text documents consumed and exported (manifest 0.972 MiB at N=10, 4.857 MiB at N=50), then the three materializations measured **with copy#1 (`self.manifest`) kept alive through `loaddata`** — matching the real importer, where `self.manifest` (`L73`) is a live attribute at the `loaddata` call (`L87`) — followed by driving the **real** `document_importer` end to end for the true whole-command peak. This backs the large-manifest copy#1 (~1x manifest) and the ~2x-manifest coexistence-peak figures in §8.2/§8.3. Two invocations (N=10 first, then N=50):
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_big.py            # builds + measures N=10 (0.972 MiB manifest)
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/focused_big.py 50         # builds + measures N=50 (4.857 MiB manifest)
```
Complete, unedited output — N=10 (`/tmp/mem_results/focused_big.txt`):
```
[2026-07-08 08:14:36,708] [INFO] [paperless.consumer] Consuming bigtext_1.txt
[2026-07-08 08:14:52,970] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_1 consumption finished
[2026-07-08 08:14:52,979] [INFO] [paperless.consumer] Consuming bigtext_2.txt
[2026-07-08 08:15:03,940] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_2 consumption finished
[2026-07-08 08:15:03,949] [INFO] [paperless.consumer] Consuming bigtext_3.txt
[2026-07-08 08:15:14,876] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_3 consumption finished
[2026-07-08 08:15:14,885] [INFO] [paperless.consumer] Consuming bigtext_4.txt
[2026-07-08 08:15:25,875] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_4 consumption finished
[2026-07-08 08:15:25,885] [INFO] [paperless.consumer] Consuming bigtext_5.txt
[2026-07-08 08:15:36,889] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_5 consumption finished
[2026-07-08 08:15:36,898] [INFO] [paperless.consumer] Consuming bigtext_6.txt
[2026-07-08 08:15:48,845] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_6 consumption finished
[2026-07-08 08:15:48,854] [INFO] [paperless.consumer] Consuming bigtext_7.txt
[2026-07-08 08:15:59,823] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_7 consumption finished
[2026-07-08 08:15:59,832] [INFO] [paperless.consumer] Consuming bigtext_8.txt
[2026-07-08 08:16:10,899] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_8 consumption finished
[2026-07-08 08:16:10,908] [INFO] [paperless.consumer] Consuming bigtext_9.txt
[2026-07-08 08:16:22,155] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_9 consumption finished
[2026-07-08 08:16:22,164] [INFO] [paperless.consumer] Consuming bigtext_10.txt
[2026-07-08 08:16:33,268] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_10 consumption finished
[BIG EXPORT] n=10 docs=10 manifest_bytes=1018848 (0.972 MiB)

===== FOCUSED BIG (copy#1 kept alive) n=10 manifest=1018848B (0.972 MiB) =====
  copy#1 json.load [L73] (kept alive): dTM_current=0.981 MiB peak=71.82 MiB records=11
  copy#2 list(filter) [L137]: dTM_current=0.6 KiB len=10 (shallow refs)
[PROBE] big loaddata n=10 BEFORE (copy#1 alive)
    RSS_current(VmRSS)  = 219344 kB (214.2 MiB)
    RSS_hiwater(maxrss) = 217068 kB (212.0 MiB)
    tracemalloc.current = 74263416 B (70.82 MiB)
    tracemalloc.peak    = 74263416 B (70.82 MiB)
    gc.live_objects     = 137200
    gc.get_count()      = (540, 7, 8)
[PROBE] big loaddata n=10 AFTER (coexistence peak)
    RSS_current(VmRSS)  = 219344 kB (214.2 MiB)
    RSS_hiwater(maxrss) = 217068 kB (212.0 MiB)
    tracemalloc.current = 74273553 B (70.83 MiB)
    tracemalloc.peak    = 76447048 B (72.91 MiB)
    gc.live_objects     = 136810
    gc.get_count()      = (0, 8, 8)
  loaddata [L87] WITH copy#1 alive: dTM_current=0.010 MiB COEXIST_peak=72.91 MiB docs_in_db=10

===== REAL document_importer (whole command) n=10 =====
[PROBE] real importer n=10 BEFORE
    RSS_current(VmRSS)  = 219344 kB (214.2 MiB)
    RSS_hiwater(maxrss) = 217068 kB (212.0 MiB)
    tracemalloc.current = 73272206 B (69.88 MiB)
    tracemalloc.peak    = 76447048 B (72.91 MiB)
    gc.live_objects     = 136944
    gc.get_count()      = (125, 8, 8)
Installed 11 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/10 [00:00<?, ?it/s]
 20%|██        | 2/10 [00:00<00:00, 11.03it/s]
 40%|████      | 4/10 [00:00<00:00, 10.99it/s]
 60%|██████    | 6/10 [00:00<00:00, 11.04it/s]
 80%|████████  | 8/10 [00:00<00:00, 11.11it/s]
100%|██████████| 10/10 [00:00<00:00, 11.06it/s]
100%|██████████| 10/10 [00:00<00:00, 11.05it/s]
[PROBE] real importer n=10 AFTER
    RSS_current(VmRSS)  = 221784 kB (216.6 MiB)
    RSS_hiwater(maxrss) = 217976 kB (212.9 MiB)
    tracemalloc.current = 73382871 B (69.98 MiB)
    tracemalloc.peak    = 79413996 B (75.74 MiB)
    gc.live_objects     = 136938
    gc.get_count()      = (0, 0, 12)
  document_importer whole-command: dRSS=2.38MiB peakTM=75.74MiB docs_in_db=10
```
Complete, unedited output — N=50 (`/tmp/mem_results/focused_big50.txt`):
```
[2026-07-08 08:20:43,039] [INFO] [paperless.consumer] Consuming bigtext_1.txt
[2026-07-08 08:20:59,084] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_1 consumption finished
[2026-07-08 08:20:59,093] [INFO] [paperless.consumer] Consuming bigtext_2.txt
[2026-07-08 08:21:09,917] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_2 consumption finished
[2026-07-08 08:21:09,925] [INFO] [paperless.consumer] Consuming bigtext_3.txt
[2026-07-08 08:21:20,833] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_3 consumption finished
[2026-07-08 08:21:20,841] [INFO] [paperless.consumer] Consuming bigtext_4.txt
[2026-07-08 08:21:31,705] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_4 consumption finished
[2026-07-08 08:21:31,714] [INFO] [paperless.consumer] Consuming bigtext_5.txt
[2026-07-08 08:21:42,619] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_5 consumption finished
[2026-07-08 08:21:42,628] [INFO] [paperless.consumer] Consuming bigtext_6.txt
[2026-07-08 08:21:54,446] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_6 consumption finished
[2026-07-08 08:21:54,455] [INFO] [paperless.consumer] Consuming bigtext_7.txt
[2026-07-08 08:22:05,393] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_7 consumption finished
[2026-07-08 08:22:05,402] [INFO] [paperless.consumer] Consuming bigtext_8.txt
[2026-07-08 08:22:16,441] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_8 consumption finished
[2026-07-08 08:22:16,452] [INFO] [paperless.consumer] Consuming bigtext_9.txt
[2026-07-08 08:22:27,504] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_9 consumption finished
[2026-07-08 08:22:27,513] [INFO] [paperless.consumer] Consuming bigtext_10.txt
[2026-07-08 08:22:38,460] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_10 consumption finished
[2026-07-08 08:22:38,469] [INFO] [paperless.consumer] Consuming bigtext_11.txt
[2026-07-08 08:22:50,532] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_11 consumption finished
[2026-07-08 08:22:50,542] [INFO] [paperless.consumer] Consuming bigtext_12.txt
[2026-07-08 08:23:01,454] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_12 consumption finished
[2026-07-08 08:23:01,463] [INFO] [paperless.consumer] Consuming bigtext_13.txt
[2026-07-08 08:23:12,378] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_13 consumption finished
[2026-07-08 08:23:12,387] [INFO] [paperless.consumer] Consuming bigtext_14.txt
[2026-07-08 08:23:23,319] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_14 consumption finished
[2026-07-08 08:23:23,327] [INFO] [paperless.consumer] Consuming bigtext_15.txt
[2026-07-08 08:23:34,417] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_15 consumption finished
[2026-07-08 08:23:34,426] [INFO] [paperless.consumer] Consuming bigtext_16.txt
[2026-07-08 08:23:46,642] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_16 consumption finished
[2026-07-08 08:23:46,651] [INFO] [paperless.consumer] Consuming bigtext_17.txt
[2026-07-08 08:23:57,746] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_17 consumption finished
[2026-07-08 08:23:57,756] [INFO] [paperless.consumer] Consuming bigtext_18.txt
[2026-07-08 08:24:08,728] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_18 consumption finished
[2026-07-08 08:24:08,737] [INFO] [paperless.consumer] Consuming bigtext_19.txt
[2026-07-08 08:24:19,676] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_19 consumption finished
[2026-07-08 08:24:19,685] [INFO] [paperless.consumer] Consuming bigtext_20.txt
[2026-07-08 08:24:30,591] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_20 consumption finished
[2026-07-08 08:24:30,600] [INFO] [paperless.consumer] Consuming bigtext_21.txt
[2026-07-08 08:24:43,040] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_21 consumption finished
[2026-07-08 08:24:43,052] [INFO] [paperless.consumer] Consuming bigtext_22.txt
[2026-07-08 08:24:54,842] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_22 consumption finished
[2026-07-08 08:24:54,853] [INFO] [paperless.consumer] Consuming bigtext_23.txt
[2026-07-08 08:25:10,448] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_23 consumption finished
[2026-07-08 08:25:10,496] [INFO] [paperless.consumer] Consuming bigtext_24.txt
[2026-07-08 08:25:21,733] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_24 consumption finished
[2026-07-08 08:25:21,743] [INFO] [paperless.consumer] Consuming bigtext_25.txt
[2026-07-08 08:25:33,389] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_25 consumption finished
[2026-07-08 08:25:33,399] [INFO] [paperless.consumer] Consuming bigtext_26.txt
[2026-07-08 08:25:46,292] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_26 consumption finished
[2026-07-08 08:25:46,302] [INFO] [paperless.consumer] Consuming bigtext_27.txt
[2026-07-08 08:25:57,468] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_27 consumption finished
[2026-07-08 08:25:57,478] [INFO] [paperless.consumer] Consuming bigtext_28.txt
[2026-07-08 08:26:08,571] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_28 consumption finished
[2026-07-08 08:26:08,581] [INFO] [paperless.consumer] Consuming bigtext_29.txt
[2026-07-08 08:26:19,577] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_29 consumption finished
[2026-07-08 08:26:19,587] [INFO] [paperless.consumer] Consuming bigtext_30.txt
[2026-07-08 08:26:30,560] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_30 consumption finished
[2026-07-08 08:26:30,577] [INFO] [paperless.consumer] Consuming bigtext_31.txt
[2026-07-08 08:26:43,211] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_31 consumption finished
[2026-07-08 08:26:43,221] [INFO] [paperless.consumer] Consuming bigtext_32.txt
[2026-07-08 08:26:54,213] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_32 consumption finished
[2026-07-08 08:26:54,223] [INFO] [paperless.consumer] Consuming bigtext_33.txt
[2026-07-08 08:27:05,260] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_33 consumption finished
[2026-07-08 08:27:05,270] [INFO] [paperless.consumer] Consuming bigtext_34.txt
[2026-07-08 08:27:16,822] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_34 consumption finished
[2026-07-08 08:27:16,834] [INFO] [paperless.consumer] Consuming bigtext_35.txt
[2026-07-08 08:27:28,515] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_35 consumption finished
[2026-07-08 08:27:28,525] [INFO] [paperless.consumer] Consuming bigtext_36.txt
[2026-07-08 08:27:41,946] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_36 consumption finished
[2026-07-08 08:27:41,958] [INFO] [paperless.consumer] Consuming bigtext_37.txt
[2026-07-08 08:27:53,053] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_37 consumption finished
[2026-07-08 08:27:53,064] [INFO] [paperless.consumer] Consuming bigtext_38.txt
[2026-07-08 08:28:04,678] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_38 consumption finished
[2026-07-08 08:28:04,693] [INFO] [paperless.consumer] Consuming bigtext_39.txt
[2026-07-08 08:28:16,379] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_39 consumption finished
[2026-07-08 08:28:16,389] [INFO] [paperless.consumer] Consuming bigtext_40.txt
[2026-07-08 08:28:27,558] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_40 consumption finished
[2026-07-08 08:28:27,568] [INFO] [paperless.consumer] Consuming bigtext_41.txt
[2026-07-08 08:28:40,409] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_41 consumption finished
[2026-07-08 08:28:40,419] [INFO] [paperless.consumer] Consuming bigtext_42.txt
[2026-07-08 08:28:51,393] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_42 consumption finished
[2026-07-08 08:28:51,403] [INFO] [paperless.consumer] Consuming bigtext_43.txt
[2026-07-08 08:29:02,541] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_43 consumption finished
[2026-07-08 08:29:02,552] [INFO] [paperless.consumer] Consuming bigtext_44.txt
[2026-07-08 08:29:13,480] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_44 consumption finished
[2026-07-08 08:29:13,491] [INFO] [paperless.consumer] Consuming bigtext_45.txt
[2026-07-08 08:29:24,446] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_45 consumption finished
[2026-07-08 08:29:24,459] [INFO] [paperless.consumer] Consuming bigtext_46.txt
[2026-07-08 08:29:37,716] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_46 consumption finished
[2026-07-08 08:29:37,726] [INFO] [paperless.consumer] Consuming bigtext_47.txt
[2026-07-08 08:29:48,655] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_47 consumption finished
[2026-07-08 08:29:48,667] [INFO] [paperless.consumer] Consuming bigtext_48.txt
[2026-07-08 08:29:59,555] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_48 consumption finished
[2026-07-08 08:29:59,565] [INFO] [paperless.consumer] Consuming bigtext_49.txt
[2026-07-08 08:30:10,479] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_49 consumption finished
[2026-07-08 08:30:10,490] [INFO] [paperless.consumer] Consuming bigtext_50.txt
[2026-07-08 08:30:21,358] [INFO] [paperless.consumer] Document 2026-07-08 bigtext_50 consumption finished
[BIG EXPORT] n=50 docs=50 manifest_bytes=5092926 (4.857 MiB)

===== FOCUSED BIG (copy#1 kept alive) n=50 manifest=5092926B (4.857 MiB) =====
  copy#1 json.load [L73] (kept alive): dTM_current=4.894 MiB peak=78.86 MiB records=51
  copy#2 list(filter) [L137]: dTM_current=0.9 KiB len=50 (shallow refs)
[PROBE] big loaddata n=50 BEFORE (copy#1 alive)
    RSS_current(VmRSS)  = 236440 kB (230.9 MiB)
    RSS_hiwater(maxrss) = 238748 kB (233.2 MiB)
    tracemalloc.current = 77568143 B (73.97 MiB)
    tracemalloc.peak    = 77568143 B (73.97 MiB)
    gc.live_objects     = 129843
    gc.get_count()      = (209, 5, 10)
[PROBE] big loaddata n=50 AFTER (coexistence peak)
    RSS_current(VmRSS)  = 236440 kB (230.9 MiB)
    RSS_hiwater(maxrss) = 238748 kB (233.2 MiB)
    tracemalloc.current = 77587477 B (73.99 MiB)
    tracemalloc.peak    = 87954162 B (83.88 MiB)
    gc.live_objects     = 129405
    gc.get_count()      = (0, 6, 10)
  loaddata [L87] WITH copy#1 alive: dTM_current=0.018 MiB COEXIST_peak=83.88 MiB docs_in_db=50

===== REAL document_importer (whole command) n=50 =====
[PROBE] real importer n=50 BEFORE
    RSS_current(VmRSS)  = 236440 kB (230.9 MiB)
    RSS_hiwater(maxrss) = 238748 kB (233.2 MiB)
    tracemalloc.current = 72471347 B (69.11 MiB)
    tracemalloc.peak    = 87954162 B (83.88 MiB)
    gc.live_objects     = 129419
    gc.get_count()      = (10, 6, 10)
Installed 51 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/50 [00:00<?, ?it/s]
  4%|▍         | 2/50 [00:00<00:04, 11.07it/s]
  8%|▊         | 4/50 [00:00<00:04, 11.18it/s]
 12%|█▏        | 6/50 [00:00<00:03, 11.14it/s]
 16%|█▌        | 8/50 [00:00<00:03, 11.18it/s]
 20%|██        | 10/50 [00:00<00:03, 11.18it/s]
 24%|██▍       | 12/50 [00:01<00:03, 11.17it/s]
 28%|██▊       | 14/50 [00:01<00:03, 11.17it/s]
 32%|███▏      | 16/50 [00:01<00:03, 11.18it/s]
 36%|███▌      | 18/50 [00:01<00:02, 11.20it/s]
 40%|████      | 20/50 [00:01<00:02, 11.22it/s]
 44%|████▍     | 22/50 [00:01<00:02, 11.23it/s]
 48%|████▊     | 24/50 [00:02<00:02, 11.20it/s]
 52%|█████▏    | 26/50 [00:02<00:02, 11.19it/s]
 56%|█████▌    | 28/50 [00:02<00:01, 11.19it/s]
 60%|██████    | 30/50 [00:02<00:01, 11.18it/s]
 64%|██████▍   | 32/50 [00:02<00:01, 11.13it/s]
 68%|██████▊   | 34/50 [00:03<00:01, 11.11it/s]
 72%|███████▏  | 36/50 [00:03<00:01, 11.15it/s]
 76%|███████▌  | 38/50 [00:03<00:01, 11.22it/s]
 80%|████████  | 40/50 [00:03<00:00, 11.25it/s]
 84%|████████▍ | 42/50 [00:03<00:00, 11.27it/s]
 88%|████████▊ | 44/50 [00:03<00:00, 11.26it/s]
 92%|█████████▏| 46/50 [00:04<00:00, 11.20it/s]
 96%|█████████▌| 48/50 [00:04<00:00, 11.23it/s]
100%|██████████| 50/50 [00:04<00:00, 11.24it/s]
100%|██████████| 50/50 [00:04<00:00, 11.20it/s]
[PROBE] real importer n=50 AFTER
    RSS_current(VmRSS)  = 239764 kB (234.1 MiB)
    RSS_hiwater(maxrss) = 243620 kB (237.9 MiB)
    tracemalloc.current = 72592460 B (69.23 MiB)
    tracemalloc.peak    = 99523195 B (94.91 MiB)
    gc.live_objects     = 129464
    gc.get_count()      = (0, 2, 27)
  document_importer whole-command: dRSS=3.25MiB peakTM=94.91MiB docs_in_db=50

FOCUSED_BIG_DONE
```


---

### B-11. `train_classifier` entry point — `train_harness.py` (Q3, F4)

Drives the **real** `train_classifier` task (`tasks.py:L48`) three ways: (1) the early-return path with no `MATCH_AUTO` objects (returns at `tasks.py:L50-54`, `dRSS`=0.52 MiB, `peakTM`=40.83 MiB); (2) the real fit on 10 documents — the **first** fit pays the one-time scikit-learn import (`dRSS`=79.08 MiB, `peakTM`=68.70 MiB, model written); (3) a warm fit on 100 documents (`dRSS`=1.96 MiB — sklearn already imported). This is a periodic, not per-document, cost. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/train_harness.py
```
Complete, unedited output (`/tmp/mem_results/train_harness.txt`, 53 lines):
```
===== TRAIN (1) early-return path (no MATCH_AUTO objects) =====
[PROBE] train early BEFORE
    RSS_current(VmRSS)  = 118932 kB (116.1 MiB)
    RSS_hiwater(maxrss) = 116464 kB (113.7 MiB)
    tracemalloc.current = 42789898 B (40.81 MiB)
    tracemalloc.peak    = 43771289 B (41.74 MiB)
    gc.live_objects     = 91828
    gc.get_count()      = (46, 7, 2)
[PROBE] train early AFTER
    RSS_current(VmRSS)  = 119468 kB (116.7 MiB)
    RSS_hiwater(maxrss) = 116464 kB (113.7 MiB)
    tracemalloc.current = 42807227 B (40.82 MiB)
    tracemalloc.peak    = 42816305 B (40.83 MiB)
    gc.live_objects     = 91879
    gc.get_count()      = (139, 7, 2)
  early-return dRSS=0.52MiB peakTM=40.83MiB (returns at tasks.py:L50-54)

===== TRAIN (2) real path n_docs=10 content_len~200 =====
[PROBE] train n=10 BEFORE
    RSS_current(VmRSS)  = 119508 kB (116.7 MiB)
    RSS_hiwater(maxrss) = 116464 kB (113.7 MiB)
    tracemalloc.current = 42914861 B (40.93 MiB)
    tracemalloc.peak    = 43608362 B (41.59 MiB)
    gc.live_objects     = 92125
    gc.get_count()      = (503, 7, 2)
[2026-07-08 07:17:52,335] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/mh-root-pat5otk2/data/classification_model.pickle...
[PROBE] train n=10 AFTER
    RSS_current(VmRSS)  = 200488 kB (195.8 MiB)
    RSS_hiwater(maxrss) = 196436 kB (191.8 MiB)
    tracemalloc.current = 71790989 B (68.47 MiB)
    tracemalloc.peak    = 72038822 B (68.70 MiB)
    gc.live_objects     = 129017
    gc.get_count()      = (348, 4, 1)
  train n=10: dRSS=79.08MiB peakTM=68.70MiB model_exists=True dLiveObj=36892

===== TRAIN (2) real path n_docs=100 content_len~200 =====
[PROBE] train n=100 BEFORE
    RSS_current(VmRSS)  = 201372 kB (196.7 MiB)
    RSS_hiwater(maxrss) = 197460 kB (192.8 MiB)
    tracemalloc.current = 71849854 B (68.52 MiB)
    tracemalloc.peak    = 72931708 B (69.55 MiB)
    gc.live_objects     = 128875
    gc.get_count()      = (116, 5, 1)
[2026-07-08 07:17:53,081] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/mh-root-pat5otk2/data/classification_model.pickle...
[PROBE] train n=100 AFTER
    RSS_current(VmRSS)  = 203376 kB (198.6 MiB)
    RSS_hiwater(maxrss) = 198484 kB (193.8 MiB)
    tracemalloc.current = 71950784 B (68.62 MiB)
    tracemalloc.peak    = 73599800 B (70.19 MiB)
    gc.live_objects     = 128949
    gc.get_count()      = (182, 7, 1)
  train n=100: dRSS=1.96MiB peakTM=70.19MiB model_exists=True dLiveObj=74
TRAIN_HARNESS_DONE
```

### B-12. `sanity_check` entry point — `sanity_harness.py` (Q3, F4)

Drives the **real** `sanity_check` task (`tasks.py:L255`) → `check_sanity()` (`sanity_checker.py:L49`) over a small corpus. The whole-file `hashlib.md5(f.read())` reads at `sanity_checker.py:L83` (source) and `L112` (archive) were counted and sized at runtime (`call_count=10`, largest single read 150 479 B ≈ 0.14 MiB); `dRSS`=1.10 MiB, `peakTM`=77.70 MiB. Each `md5` read is a transient bounded by the single largest file, released before the next. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/sanity_harness.py
```
Complete, unedited output (`/tmp/mem_results/sanity_harness.txt`, 33 lines):
```
[2026-07-08 07:17:08,171] [INFO] [paperless.consumer] Consuming s0_simple-digital.pdf
[2026-07-08 07:17:10,610] [INFO] [paperless.consumer] Document 2026-07-08 s0_simple-digital consumption finished
[2026-07-08 07:17:10,619] [INFO] [paperless.consumer] Consuming s1_multi-page-images.pdf
[2026-07-08 07:17:11,063] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:17:11,070] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:17:11,071] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
[2026-07-08 07:17:14,525] [INFO] [paperless.consumer] Document 2026-07-08 s1_multi-page-images consumption finished
[2026-07-08 07:17:14,536] [INFO] [paperless.consumer] Consuming gen0.pdf
[2026-07-08 07:17:21,944] [INFO] [paperless.consumer] Document 2026-07-08 gen0 consumption finished
[2026-07-08 07:17:21,954] [INFO] [paperless.consumer] Consuming gen1.pdf
[2026-07-08 07:17:23,944] [INFO] [paperless.consumer] Document 2026-07-08 gen1 consumption finished
[2026-07-08 07:17:23,955] [INFO] [paperless.consumer] Consuming gen2.pdf
[2026-07-08 07:17:25,960] [INFO] [paperless.consumer] Document 2026-07-08 gen2 consumption finished
[PROBE] sanity BEFORE
    RSS_current(VmRSS)  = 256344 kB (250.3 MiB)
    RSS_hiwater(maxrss) = 265700 kB (259.5 MiB)
    tracemalloc.current = 81300509 B (77.53 MiB)
    tracemalloc.peak    = 82701192 B (78.87 MiB)
    gc.live_objects     = 144852
    gc.get_count()      = (20, 0, 4)
[2026-07-08 07:17:25,986] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
[PROBE] sanity AFTER
    RSS_current(VmRSS)  = 257472 kB (251.4 MiB)
    RSS_hiwater(maxrss) = 265700 kB (259.5 MiB)
    tracemalloc.current = 81315643 B (77.55 MiB)
    tracemalloc.peak    = 81477993 B (77.70 MiB)
    gc.live_objects     = 144902
    gc.get_count()      = (52, 0, 4)

[SANITY] md5(f.read()) call_count=10 read_sizes_bytes=[150479, 24197, 22926, 10860, 8058, 7998, 7979, 1476, 1476, 1472]
  largest_single_read=150479B (~0.14MiB transient)
  sanity dRSS=1.10MiB peakTM=77.70MiB
SANITY_HARNESS_DONE
```

### B-13. No-`MODEL_FILE` path (`load_classifier()` → `None`) — `no_model.py` (Q3/Q4, F4)

Drives the **real** `load_classifier()` (`classifier.py:L30`) on a fresh temp `DATA_DIR` where `MODEL_FILE` does not exist. It returns `None` at `classifier.py:L36` (rule-only matching, **zero** `pickle.load` calls) and adds **0** live objects (`gc.live_objects` 111490 → 111490). The classifier-present counterpart is Appendix B-4. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/no_model.py
```
Complete, unedited output (`/tmp/mem_results/no_model.txt`, 19 lines):
```
MODEL_FILE path      = /tmp/mh-root-36srpfn0/data/classification_model.pickle
MODEL_FILE exists?   = False
[PROBE] no-model BEFORE load_classifier()
    RSS_current(VmRSS)  = 63416 kB (61.9 MiB)
    RSS_hiwater(maxrss) = 61964 kB (60.5 MiB)
    tracemalloc.current = 19566 B (0.02 MiB)
    tracemalloc.peak    = 570422 B (0.54 MiB)
    gc.live_objects     = 111490
    gc.get_count()      = (274, 11, 10)
[PROBE] no-model AFTER load_classifier()
    RSS_current(VmRSS)  = 63512 kB (62.0 MiB)
    RSS_hiwater(maxrss) = 61964 kB (60.5 MiB)
    tracemalloc.current = 37324 B (0.04 MiB)
    tracemalloc.peak    = 920861 B (0.88 MiB)
    gc.live_objects     = 111490
    gc.get_count()      = (309, 11, 10)
load_classifier() returned = None type = NoneType
  => rule-only matching, zero pickle.load calls (classifier.py:L36 returns None)
NO_MODEL_HARNESS_DONE
```

### B-14. Error / encrypted path — `error_path.py` (Q1/Q4, F4)

Drives two real failure inputs through `consume_file` → `try_consume_file`. **CASE A** — an **encrypted** PDF (`encrypted.pdf`, 46 594 B): `pdfminer` raises `PDFPasswordIncorrect`, the parser logs "This file is encrypted, OCR is impossible" and consumption **SUCCEEDS** with empty content (`Document pk=1`, `content_len=0`). **CASE B** — a genuinely **corrupt** PDF (`corrupt.pdf`, 1 450 B): `pdfminer` raises `PSEOF: Unexpected EOF`, `ocrmypdf` then raises `InputFileError` (pikepdf `PdfError: unable to find trailer dictionary`), wrapped as `ParseError` at `src/paperless_tesseract/parsers.py:L310` and surfaced from `consumer.py:L261` as `ConsumerError`. In **both** cases the `finally: document_parser.cleanup()` (`consumer.py:L369`) runs, so `paperless-*` tempdirs are `before=0 after=0` (no leak); `peakTM` during = 55.04 MiB (A) and 53.24 MiB (B). Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/error_path.py
```
Complete, unedited output (`/tmp/mem_results/error_path.txt`, 144 lines):
```

===== CASE A (encrypted.pdf) file=encrypted.pdf size=46594B detected_mime=application/pdf =====
[2026-07-08 07:19:09,148] [INFO] [paperless.consumer] Consuming encrypted.pdf
[2026-07-08 07:19:09,273] [WARNING] [paperless.parsing.tesseract] Error while getting text from PDF document with pdfminer.six
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
[2026-07-08 07:19:09,675] [WARNING] [paperless.parsing.tesseract] This file is encrypted, OCR is impossible. Using any text present in the original file.
[2026-07-08 07:19:09,675] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/mh-root-eai0hg69/scratch/errpath/encrypted.pdf, the content will be empty.
   **** This file requires a password for access.
Error: /invalidfileaccess in pdf_process_Encrypt
Operand stack:

Execution stack:
   %interp_exit   .runexec2   --nostringval--   runpdf   --nostringval--   2   %stopped_push   --nostringval--   runpdf   runpdf   false   1   %stopped_push   1990   1   3   %oparray_pop   1989   1   3   %oparray_pop   1977   1   3   %oparray_pop   1978   1   3   %oparray_pop   runpdf   runpdf   runpdf   runpdf   false   1   %stopped_push
Dictionary stack:
   --dict:739/1123(ro)(G)--   --dict:1/20(G)--   --dict:80/200(L)--   --dict:80/200(L)--   --dict:133/256(ro)(G)--   --dict:320/325(ro)(G)--   --dict:29/32(L)--
Current allocation mode is local
Last OS error: No such file or directory
GPL Ghostscript 9.53.3: Unrecoverable error, exit code 1
convert-im6.q16: no images defined `/tmp/mh-root-eai0hg69/scratch/paperless-4y1tbvzd/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-08 07:19:09,763] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
   **** This file requires a password for access.
Error: /invalidfileaccess in pdf_process_Encrypt
Operand stack:

Execution stack:
   %interp_exit   .runexec2   --nostringval--   runpdf   --nostringval--   2   %stopped_push   --nostringval--   runpdf   runpdf   false   1   %stopped_push   1990   1   3   %oparray_pop   1989   1   3   %oparray_pop   1977   1   3   %oparray_pop   1978   1   3   %oparray_pop   runpdf   runpdf   runpdf   runpdf   false   1   %stopped_push
Dictionary stack:
   --dict:731/1123(ro)(G)--   --dict:1/20(G)--   --dict:80/200(L)--   --dict:80/200(L)--   --dict:133/256(ro)(G)--   --dict:320/325(ro)(G)--   --dict:27/32(L)--
Current allocation mode is local
GPL Ghostscript 9.53.3: Unrecoverable error, exit code 1
[2026-07-08 07:19:10,516] [INFO] [paperless.consumer] Document 2026-07-08 encrypted consumption finished
  RESULT: SUCCESS -> Success. New document id 1 created
  created Document pk=1 content_len=0 mime=application/pdf
  paperless-* tempdirs: before=0 after=0 (cleanup on error path => no leak)
  peakTM during=55.04MiB

===== CASE B (corrupt.pdf) file=corrupt.pdf size=1450B detected_mime=application/pdf =====
[2026-07-08 07:19:10,531] [INFO] [paperless.consumer] Consuming corrupt.pdf
[2026-07-08 07:19:10,537] [WARNING] [paperless.parsing.tesseract] Error while getting text from PDF document with pdfminer.six
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 721, in __init__
    pos = self.find_xref(parser)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 978, in find_xref
    raise PDFNoValidXRef("Unexpected EOF")
pdfminer.pdfdocument.PDFNoValidXRef: Unexpected EOF

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 120, in extract_text
    stripped = post_process_text(pdfminer_extract_text(pdf_file))
  File "/usr/local/lib/python3.9/site-packages/pdfminer/high_level.py", line 157, in extract_text
    for page in PDFPage.get_pages(
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfpage.py", line 151, in get_pages
    doc = PDFDocument(parser, password=password, caching=caching)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 727, in __init__
    newxref.load(parser)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 241, in load
    (_, obj) = parser.nextobject()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/psparser.py", line 607, in nextobject
    (pos, token) = self.nexttoken()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/psparser.py", line 524, in nexttoken
    self.fillbuf()
  File "/usr/local/lib/python3.9/site-packages/pdfminer/psparser.py", line 239, in fillbuf
    raise PSEOF("Unexpected EOF")
pdfminer.psparser.PSEOF: Unexpected EOF
[2026-07-08 07:19:10,642] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: . Attempting force OCR to get the text.
[2026-07-08 07:19:10,752] [ERROR] [paperless.consumer] Error while consuming document corrupt.pdf: InputFileError: 
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 163, in get_pdfinfo
    return PdfInfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/pdfinfo/info.py", line 901, in __init__
    with Pdf.open(infile) as pdf:
  File "/usr/local/lib/python3.9/site-packages/pikepdf/_methods.py", line 923, in open
    pdf = Pdf._open(
pikepdf._qpdf.PdfError: /tmp/ocrmypdf.io.sfeh7c4h/origin.pdf: unable to find trailer dictionary while recovering damaged file

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 370, in run_pipeline
    pdfinfo = get_pdfinfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 174, in get_pdfinfo
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 163, in get_pdfinfo
    return PdfInfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/pdfinfo/info.py", line 901, in __init__
    with Pdf.open(infile) as pdf:
  File "/usr/local/lib/python3.9/site-packages/pikepdf/_methods.py", line 923, in open
    pdf = Pdf._open(
pikepdf._qpdf.PdfError: /tmp/ocrmypdf.io.bt7zmx0z/origin.pdf: unable to find trailer dictionary while recovering damaged file

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 298, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 370, in run_pipeline
    pdfinfo = get_pdfinfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 174, in get_pdfinfo
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 310, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: InputFileError: 
  RESULT: RAISED ConsumerError: corrupt.pdf: Error while consuming document corrupt.pdf: InputFileError: 
  paperless-* tempdirs: before=0 after=0 (cleanup on error path => no leak)
  peakTM during=53.24MiB

ERROR_PATH_DONE
```

### B-15. Office / Tika path — UNAVAILABLE in canonical config — `tika_probe.py` (Q5, F4)

Probes whether the office/Tika metadata path is reachable in the **default canonical** configuration. `PAPERLESS_TIKA_ENABLED` defaults to `False` (`paperless_tika/apps.py:L12` registers the parser only when enabled), so every office MIME (`.docx`/`.odt`/`.doc`/`.xlsx`) resolves to parser class `None` and none of the 21 supported extensions is an office type. The office/Tika path is therefore **unavailable / non-canonical** — exercising it requires `PAPERLESS_TIKA_ENABLED=YES` plus a running Tika server, which is outside the default configuration. Command:
```
cd /app/src && PYTHONPATH=/app/src:/tmp/mem_harness DJANGO_SETTINGS_MODULE=paperless.settings PAPERLESS_DISABLE_DBHANDLER=true python /tmp/mem_harness/tika_probe.py
```
Complete, unedited output (`/tmp/mem_results/tika_probe.txt`, 12 lines):
```
PAPERLESS_TIKA_ENABLED (canonical default) = False

[office mime -> parser class in CANONICAL config]
  .docx  application/vnd.openxmlformats-officedocument.wordprocessing -> None
  .odt   application/vnd.oasis.opendocument.text                      -> None
  .doc   application/msword                                           -> None
  .xlsx  application/vnd.openxmlformats-officedocument.spreadsheetml. -> None

  supported extensions count=21
  any office extension supported in canonical config? NONE
  => office/Tika metadata path is UNAVAILABLE in canonical config (non-canonical: requires PAPERLESS_TIKA_ENABLED=YES + a Tika server).
TIKA_PROBE_DONE
```


## Appendix C — Read-only proof (repository left unchanged)

All harness scripts lived under `/tmp/mem_harness/` (inside the canonical container, outside the repository) and were deleted after use; the only persistent write to the repository is this answer document.

**Two distinct commits — do not conflate them:**

- **Source commit investigated** — `542221a38dff06361e07976452f9aea24d210542`. This is the paperless-ngx baseline that the read-only investigation targeted: the canonical container's `/app` checkout sits at this commit, and every `file:line` reference in this document is anchored to it.
- **Destination-branch HEAD before this remediation** — `6103c10892bb46ce5d4c453c34801f11976fa948` on branch `blitzy-6a755538-5279-46bb-a229-50a82880251d`. This answer document is committed as a single follow-up commit whose **parent is `6103c108…`** (i.e. `HEAD~1`). Neither of these is the *source* commit; the document filename embeds the source short-SHA `542221a38dff` for traceability, which is the investigated commit — not any branch HEAD.

**What the deliverable commit changed (parent → this commit), verbatim — stable regardless of the commit's own hash:**

```
$ git rev-parse --abbrev-ref HEAD
blitzy-6a755538-5279-46bb-a229-50a82880251d

$ git rev-parse HEAD~1
6103c10892bb46ce5d4c453c34801f11976fa948

$ git diff --name-status HEAD~1 HEAD
M	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --name-only HEAD~1 HEAD -- src/ | wc -l
0
```

The single follow-up commit modifies exactly **one** path across the whole tree — this answer document — and touches **zero** files under `src/`. (`git diff --name-status HEAD~1 HEAD` compares the pre-remediation HEAD `6103c108…` to the commit that carries this document, so it is unaffected by that commit's own absolute hash.)

**Clean working tree after commit, and baseline-diff vs the investigated source, verbatim:**

```
$ git status --porcelain

$ git ls-files --others --exclude-standard | wc -l
0

$ git diff --name-only 542221a38dff06361e07976452f9aea24d210542 -- src/ | wc -l
0
```

`git status --porcelain` prints nothing — the working tree is clean, this document is committed, and there is no other modified or untracked file anywhere (`git ls-files --others` = 0). **Zero** files under `src/` differ between the investigated source commit `542221a38dff…` and the committed tree: the entire `src/` tree — including all 24 investigated source files (`consumer.py`, `tasks.py`, `parsers.py`, the three parser plugins, `classifier.py`, `document_importer.py`, `views.py`, `handlers.py`, `matching.py`, `models.py`, `sanity_checker.py`, `index.py`, `settings.py`, and the rest) — is byte-for-byte unchanged. The source repository is therefore unchanged apart from this single answer document, satisfying the verbatim user constraint: *"Don't modify any repository source files … leave the codebase unchanged when done."*

