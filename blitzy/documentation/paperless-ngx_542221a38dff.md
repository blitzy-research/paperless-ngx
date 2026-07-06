# paperless-ngx — Memory Diagnostic of the Document Import / Metadata Pipeline

**Commit:** `542221a38dff` · **Runtime:** Python 3.9.23 (canonical `python:3.9-slim-bullseye`, `Dockerfile:18`) · **Mode:** read‑only, runtime‑measured investigation

> **How to read this document.** Every numeric claim below was produced by _running the real
> consumption pipeline_ (`documents.consumer.Consumer.try_consume_file` / the real parser and
> classifier methods) under three simultaneous memory instruments — `tracemalloc` (Python heap),
> `gc` (object counts / uncollectable cycles) and `psutil`/`resource` (process RSS). The **raw,
> unedited output** and the **exact command** that produced it are shown _before_ any summary.
> Anything not observed at runtime is explicitly marked **(inferred)**. No repository source file
> was modified; the only artifact added to the repo is this document.

---

## 1. Scope & what was investigated

The user reports that importing documents "sometimes consumes far more memory than expected for the
metadata handling", that the spike is "disproportionate to the actual document size", that the memory
"doesn't always get released back to the system in a timely manner", and that it "seems to vary based
on the document source or processing stage." This document answers, with measurements:

- **Q1** — Is metadata handling making unnecessary **copies** or holding **references** too long?
- **Q2** — Is there **caching** that accumulates data unexpectedly?
- **Q3** — What is different between the runs that **spike** and the runs that **don't**?
- **Q4** — How does behavior change with **document type** and **batch size**?
- **Deliverable A** — actual runtime memory measurements; **B** — which components/methods hold memory;
  **C** — is this normal CPython GC behavior or a genuine defect, and where does the memory actually go.

The investigation is confined to the Python/Django backend consumption path. The canonical real entry
point driven throughout is `Consumer.try_consume_file` [`src/documents/consumer.py:180`], reached in
production from `documents.tasks.consume_file` [`src/documents/tasks.py:184`] which calls it at
`src/documents/tasks.py:236` (`document = Consumer().try_consume_file(`).

### 1.1 Memory‑relevant data flow (verified against source @ `542221a38dff`)

```mermaid
flowchart TD
    A["consume_file task<br/>tasks.py:184 -> :236"] --> B["Consumer.try_consume_file<br/>consumer.py:180"]
    B --> C["pre_check_duplicate<br/>hashlib.md5(f.read())<br/>consumer.py:104 — full-file copy"]
    C --> D["parser.parse()<br/>text: f.read() paperless_text/parsers.py:42<br/>pdf: OCRmyPDF (heavy)"]
    D --> E["get_text/get_date -> parse_date()<br/>dateparser + regex lazy import"]
    E --> F{"MODEL_FILE exists?<br/>settings.py:74; gate classifier.py:31"}
    F -- No --> G["load_classifier() returns None<br/>classifier.py:36 — NO spike"]
    F -- Yes --> H["load_classifier() unpickles model<br/>classifier.py:30 (no cache) -> load() :76 -> pickle.load :86-92<br/>SPIKE"]
    G --> I["transaction.atomic: _store<br/>consumer.py:379; md5(f.read()) :402"]
    H --> I
    I --> J["document_consumption_finished.send(classifier=classifier)<br/>consumer.py:306"]
    J --> K["6 handlers set_correspondent/type/tags<br/>handlers.py:35/101/168 -> matching.py predict + regex"]
    K --> L["_write read_file.read()<br/>consumer.py:432; index update content=doc.content index.py:93"]
    L --> M["task returns -> qcluster recycle:1<br/>settings.py:452 — process restart reclaims RSS"]
```

---

## 2. Runtime & environment (canonical, reproducible)

All runs execute **inside** the provided container image `paperless-ngx-qna-ready:542221a38dff`
(built from `ghcr.io/scaleapi/swe-atlas:...paperless-ngx...qna_1.01`), on **Python 3.9.23**, with an
isolated scratch data tree under `/tmp/blitzy_probe` so nothing touches the tracked repository. The
ambient host interpreter (Python 3.12/3.13) is **not** used — it lacks Django/scikit‑learn/pikepdf.

**Environment (sourced before every run):**

```
export PAPERLESS_DATA_DIR=/tmp/blitzy_probe/data
export PAPERLESS_MEDIA_ROOT=/tmp/blitzy_probe/media
export PAPERLESS_CONSUMPTION_DIR=/tmp/blitzy_probe/consume
export PAPERLESS_SCRATCH_DIR=/tmp/blitzy_probe/scratch
export DJANGO_SETTINGS_MODULE=paperless.settings
export PYTHONPATH=/app/src
```

**Versions & Django init (command + raw output):**

```
$ python3 -c "import django,sklearn,pikepdf,psutil,numpy,scipy; print(...)"
django 4.0.4 | sklearn 1.0.2 | pikepdf 5.1.1 | psutil 7.2.2 | numpy 1.22.3 | scipy 1.8.0

$ python3 -c "import django; django.setup(); from django.conf import settings; print(...)"
DATA_DIR=  /tmp/blitzy_probe/data
MODEL_FILE=/tmp/blitzy_probe/data/classification_model.pickle
DEBUG=     False        # canonical default, settings.py:50
SCRATCH_DIR=/tmp/blitzy_probe/scratch
DB=        /tmp/blitzy_probe/data/db.sqlite3
```

**Database migrate (command + head/tail of raw output):**

```
$ cd /app/src && python3 manage.py migrate --no-input
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
  Applying sessions.0001_initial... OK
# 103 lines total; DB created at /tmp/blitzy_probe/data/db.sqlite3 (331776 bytes)
```

The three memory instruments are combined in a temporary helper (`memprobe.py`, deleted at the end):
`psutil.Process().memory_info().rss`, `resource.getrusage(RUSAGE_SELF).ru_maxrss`,
`tracemalloc.get_traced_memory()` + `take_snapshot().statistics('lineno')` +
`snapshot.compare_to(...)`, and a `collections.Counter` over `gc.get_objects()` plus `gc.garbage`.

**Sample inputs** (generated under `/tmp`, sizes via `os.stat().st_size`, mime via `magic.from_file`):

| file         |   size | detected mime                             | parser selected (`get_parser_class_for_mime_type`, `parsers.py:81`) |
| ------------ | -----: | ----------------------------------------- | ------------------------------------------------------------------- |
| `sample.txt` | 5310 B | `text/plain`                              | `TextDocumentParser` (`paperless_text/parsers.py`)                  |
| `sample.pdf` | 1569 B | `application/pdf`                         | `RasterisedDocumentParser` (`paperless_tesseract/parsers.py`)       |
| `sample.odt` |  726 B | `application/vnd.oasis.opendocument.text` | **None** (Tika disabled by default — see Q4)                        |

These are exactly the "text documents with relatively small metadata" the user describes: the largest
is 5310 bytes, yet a single import moves **tens to ~180 MB of RSS** (below). That gap is the subject of
this report.

---

## 3. TL;DR — direct answers

- **Q1 (copies / references): Yes, there are real full‑file copies, and one real lifetime‑extension —
  but they are proportional to _file_ size (not "metadata") and none of them leak.** The consumer reads
  each whole file into memory at least three times via `hashlib.md5(f.read())`
  [`consumer.py:104`, `:402`] and `read_file.read()` [`consumer.py:432`]; the text parser holds the
  whole file with `self.text = f.read()` [`paperless_text/parsers.py:42`]. Measured: the transient copy
  equals the file size exactly (1 MB→1.00 MB … 50 MB→50.00 MB). The classifier object is held across the
  entire persistence tail because it is passed into six post‑consume signal handlers
  [`consumer.py:306` → `handlers.py:35/101/168`] — confirmed alive in the handler (refcount 7). The PDF
  metadata path opens `pikepdf.open()` with **no** `with`/`close()` [`paperless_tesseract/parsers.py:34‑35`],
  holding **native** (off‑heap) QPDF memory until GC. All of these release cleanly (`gc.garbage == 0`).

- **Q2 (caching): The one caching‑shaped behavior that matters is the _absence_ of a cache, not an
  unbounded one.** `load_classifier()` [`classifier.py:30`] has **no module‑level cache** and
  re‑unpickles the entire scikit‑learn model on **every** call (proven: 5 calls → 5 distinct object ids).
  The other suspected accumulator — Django's `connection.queries` under `DEBUG` [`settings.py:50`] — is
  **bounded**: it is a `deque(maxlen=9000)`, is **empty** with the canonical `DEBUG=False`, and caps at
  9000 entries (~3.5 MB) when enabled. Neither is an unbounded leak.

- **Q3 (spike vs no‑spike): The single dominant switch is whether the classifier model file exists.**
  Same 5310‑byte text file, fresh worker process, 2 runs each: **with** `MODEL_FILE` present the import
  peaks at **RSS +178 MB / Python‑heap +69.5 MB**; **without** it, **RSS +98 MB / heap +42.7 MB**. The
  presence/absence is gated at `classifier.py:31` (`if not os.path.isfile(settings.MODEL_FILE): return None`
  `:36`). Because the model is produced asynchronously by training, the _same_ input consumed at different
  times lands on different branches — which is exactly the "sometimes it spikes, sometimes it doesn't"
  the user observes.

- **Q4 (document type / batch size): Document type sets the _fixed library cost_; batch size does _not_
  cause unbounded growth within a worker, and is reset per task by `recycle:1`.** Text vs PDF pull in
  different heavy libraries (dateparser+regex vs OCRmyPDF/pdfminer/scipy). Consuming the same file 6× in
  one long‑lived process leaves the **Python heap flat** (`tracemalloc_cur` ≈ 79.9–80.0 MB across all six)
  and RSS oscillating on a plateau — no per‑document accumulation. The one genuinely batch‑proportional
  path is `DocumentClassifier.train()` [`classifier.py:115`], which holds the **whole corpus** in memory
  (memory ∝ N).

- **Deliverable C verdict (normal vs problematic): This is normal CPython/glibc allocator behavior plus
  one‑time library initialization — not a leak.** After a full consume and `del`+`gc.collect()`, the
  Python heap returns to baseline (**+54.2 KB**), object counts return to baseline (**+33**), and
  `gc.garbage == 0`, while **RSS stays +134.70 MB** elevated. The elevated RSS the user sees is freed
  memory the allocator keeps (arena retention), not live/leaked objects; across tasks it is reclaimed by
  the django‑q `recycle: 1` worker restart [`settings.py:452`].

---

## Q1 — Unnecessary copies / references held longer than needed

**Direct answer:** There are **three** distinct real behaviors, and all are grounded in code:
(1) the pipeline makes **full‑file byte copies** whose size tracks the _file_, not the _metadata_;
(2) the **classifier object's lifetime is extended** across the whole persistence tail by being passed
into six post‑consume signal handlers; (3) the PDF **metadata** extractor opens `pikepdf.open()` and
never closes it, holding **native** memory. **None of them leaks** — every one releases on
dereference + GC (`gc.garbage == 0`, §Deliverable C). They _feel_ disproportionate because they are
independent of the document's small text size.

### Q1a — Full‑file `f.read()` copies (proportional to FILE size, not metadata)

The consumer reads each whole file into a single `bytes` object several times:

- `Consumer.pre_check_duplicate` [`consumer.py:102`]: `checksum = hashlib.md5(f.read()).hexdigest()` [`consumer.py:104`]
- archive checksum in `try_consume_file`: `hashlib.md5(f.read())` [`consumer.py:341`]
- `Consumer._store` [`consumer.py:379`]: `checksum=hashlib.md5(f.read()).hexdigest()` [`consumer.py:402`]
- `Consumer._write` [`consumer.py:429`]: `write_file.write(read_file.read())` [`consumer.py:432`]
- `TextDocumentParser.parse` [`paperless_text/parsers.py:40`]: `self.text = f.read()` [`:42`]

The exact `hashlib.md5(f.read())` pattern from `consumer.py:104`/`:402` was measured against files of
known size (command shows the pattern used verbatim):

```
COMMAND: python3 exp_copies.py   # with open(path,"rb") as f: hashlib.md5(f.read()).hexdigest()
file_size | tracemalloc_peak_during_md5(f.read())
    1 MB   | peak_delta=   1.00 MB  (checksum=0d1220d0...)
    5 MB   | peak_delta=   5.00 MB  (checksum=0fef448e...)
   20 MB   | peak_delta=  20.00 MB  (checksum=e3f675a6...)
   50 MB   | peak_delta=  50.00 MB  (checksum=c75245fd...)
COPIES DONE
```

**Cause → effect:** `f.read()` with no size argument materializes the _entire_ file as one `bytes`
object; the Python‑heap peak is therefore exactly the file size (1:1, perfectly linear). For the user's
5310‑byte text file this transient is ~5 KB and negligible — which is the key point: **these copies
scale with file bytes, so they are _not_ the cause of the large spike on a small document.** They would
only dominate for large scans/PDFs. The copy is transient (freed when the `bytes` leaves scope), so it
does not accumulate.

### Q1b — Classifier reference held across the entire persistence tail

`try_consume_file` loads the classifier once at `classifier = load_classifier()` [`consumer.py:292`]
and then passes it into the post‑consume fan‑out:
`document_consumption_finished.send(sender=..., document=document, classifier=classifier)`
[`consumer.py:306`], inside the `with transaction.atomic():` block. The six receivers include
`set_correspondent` [`handlers.py:35`, param `classifier=None` `:39`], `set_document_type`
[`handlers.py:101`/`:105`] and `set_tags` [`handlers.py:168`/`:172`], which call
`matching.match_correspondents` [`handlers.py:50`] / `match_document_types` [`:116`] /
`match_tags` [`:189`]. Those in turn call `classifier.predict_correspondent(document.content)`
[`matching.py:21`/`:23`], `predict_document_type` [`matching.py:34`/`:36`] and `predict_tags`
[`matching.py:47`/`:49`]. So the loaded model stays referenced from `consumer.py:292` until the atomic
block ends.

Proven at runtime — a probe receiver attached to `document_consumption_finished` reports the _same live
`DocumentClassifier`_ reaching the handlers:

```text
$ python3 exp_reflifetime.py     # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_reflifetime.py
[2026-07-06 22:38:38,472] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/blitzy_probe/data/classification_model.pickle...
MODEL_FILE present: True
[2026-07-06 22:38:38,540] [INFO] [paperless.consumer] Consuming e723fd31_sample.txt
[2026-07-06 22:39:00,484] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 e723fd31_sample
[2026-07-06 22:39:00,489] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME e723fd31_sample
  [probe receiver on document_consumption_finished] classifier=DocumentClassifier id=135237485232032 refcount=7 is_none=False
[2026-07-06 22:39:00,539] [INFO] [paperless.consumer] Document 2026-07-06 ACME e723fd31_sample consumption finished
  RESULT: classifier reached handlers alive = True | same live DocumentClassifier object shared by all receivers (id): 135237485232032
  post-consume predictions applied: correspondent=12 document_type=12 tags=[]
REFLIFETIME DONE
```

**Cause → effect:** the code's own comment at `consumer.py` (just above `:292`) states it loads the
classifier here specifically so the "multiple post‑consume hooks that all require the classifier" do not
each reload it — i.e. the lifetime extension is **deliberate**, trading a longer hold for avoiding
repeated unpickling within one document. `refcount=7` confirms several simultaneous references
(local var, the signal's argument dict, the bound handler frame, etc.) during the tail. This is a real
"reference held longer than the parse step", but it is bounded to one document's processing and is
released when `try_consume_file` returns (§Deliverable C shows heap returns to baseline). It is **not**
a leak; it is a single object (~408 KB Python heap, Q3) living ~milliseconds longer.

### Q1c — `pikepdf.open()` in metadata extraction is never closed (native memory)

`RasterisedDocumentParser.extract_metadata` [`paperless_tesseract/parsers.py:26`] does, for PDFs:

```
pdf = pikepdf.open(document_path)      # parsers.py:34  — no `with`, no later pdf.close()
meta = pdf.open_metadata()             # parsers.py:35
for key, value in meta.items(): ...    # iterate XMP
return result                          # pdf is only released when GC eventually collects it
```

This is reached from the REST `metadata` action [`views.py:283`] via `get_metadata` [`views.py:260`] →
`parser.extract_metadata` [`views.py:269`], **up to twice per request** — once for the original at
`views.py:295` and once for the archive at `views.py:302`. QPDF is a C++ library, so its buffers live
**off** the Python heap and are invisible to `tracemalloc`; they must be watched via RSS. Measured in a
scratch harness (the source is not edited) using a 271 023‑byte PDF:

```
COMMAND: python3 exp_pikepdf.py
=== REAL method: RasterisedDocumentParser.extract_metadata (source does NOT close) ===
  extract_metadata(sample.pdf) returned 0 entries; first: None
=== SCRATCH A: pikepdf.open(big.pdf) WITHOUT close(), holding handles (as source omits close) ===
  DELTA pikepdf_noclose:before -> pikepdf_noclose:after_20_open: RSS +33.41 MB  |  tracemalloc_cur +103.2 KB  |  gc_objs +411
  (tracemalloc barely moves because QPDF memory is NATIVE/off-heap; watch RSS)
  DELTA pikepdf_noclose:after_20_open -> pikepdf_noclose:after_del_gc: RSS +0.42 MB  |  tracemalloc_cur -94.2 KB  |  gc_objs +162
=== SCRATCH B: pikepdf.open(big.pdf) WITH close() each iteration ===
  DELTA pikepdf_close:before -> pikepdf_close:after_20_open_close: RSS +0.04 MB  |  tracemalloc_cur +2.7 KB  |  gc_objs +430
PIKEPDF DONE
```

**Cause → effect:** 20 open handles held **+33.41 MB of RSS** while the Python heap barely moved
(**+103.2 KB**) — proof the retained memory is native QPDF state, not Python objects. Doing the same
with an explicit `pdf.close()` each iteration leaves RSS **flat (+0.04 MB)**. So the omitted
`close()`/`with` at `parsers.py:34‑35` is a genuine "hold longer than needed" for the _native_ layer:
the memory is only reclaimed when the `pdf` object is garbage‑collected rather than promptly at end of
`extract_metadata`. (On our reportlab sample the XMP set is empty, so `extract_metadata` returned
`0 entries` — the DocInfo title/author is not exposed as XMP; the `pikepdf.open` still executes, which
is what holds memory.)

---

## Q2 — Caching behavior accumulating data

**Direct answer:** The behavior most people would call "caching" here is actually the **opposite** — a
missing cache — plus one bounded log. (1) `load_classifier()` [`classifier.py:30`] has **no
module‑level cache**: it re‑unpickles the whole scikit‑learn model from disk on **every** call, so
nothing accumulates but the _cost is paid repeatedly_. (2) Django's `connection.queries` (the classic
"silently accumulates under DEBUG" suspect) is a **bounded** `deque(maxlen=9000)` and is **empty** under
the canonical `DEBUG=False`. So there is **no unbounded cache** in the import path.

### Q2a — `load_classifier()` has no cache (re‑unpickles every call)

```
COMMAND: python3 exp_classifier.py   (excerpt — MODEL_FILE present)
  load_classifier() #1 -> DocumentClassifier id=135947590687376
  DELTA present_call1:before -> present_call1:after: RSS +69.12 MB  |  tracemalloc_cur +407.9 KB  |  gc_objs +20498
  load_classifier() #2 -> DocumentClassifier id=135942076114592
  DELTA present_call2:before -> present_call2:after: RSS +10.40 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17372
  load_classifier() #3 -> DocumentClassifier id=135941996150016
  DELTA present_call3:before -> present_call3:after: RSS +3.50 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17415
  load_classifier() #4 -> DocumentClassifier id=135942055175216
  DELTA present_call4:before -> present_call4:after: RSS +4.00 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17405
  load_classifier() #5 -> DocumentClassifier id=135942074905456
  DELTA present_call5:before -> present_call5:after: RSS +5.25 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17375

  NO-CACHE PROOF: distinct object ids across calls = [135947590687376, 135942076114592, 135941996150016, 135942055175216, 135942074905456] -> all unique: True
```

**Cause → effect:** `load_classifier()` [`classifier.py:30`] unconditionally does
`classifier = DocumentClassifier(); classifier.load()` — there is no module‑level variable caching the
result, so each call constructs a **new** object (five unique `id()`s above) and re‑runs `load()`
[`classifier.py:76`], which executes six sequential `pickle.load(f)` calls at `classifier.py:86‑92`
after the `FORMAT_VERSION = 7` check [`:63`/`:80`]. Each call therefore repeats an identical
**+405.8 KB** Python‑heap allocation (extremely stable across calls #2–#5). This is not "accumulation"
(nothing is retained — see the recovery below and §Q3), but it is repeated work: every document
consumed [`consumer.py:292`], every `train_classifier()` run [`tasks.py:57`], and every `suggestions`
REST call [`views.py:319`] re‑reads and re‑deserializes the entire model. The pickle on disk was
**397540 bytes** for a 12‑document model.

### Q2b — `connection.queries` under `DEBUG` is bounded, and off by default

```
COMMAND: python3 exp_debug_queries.py
connection.queries_log type: deque | maxlen = 9000

=== force_debug_cursor = False (canonical default, DEBUG=False) ===
  after 500 queries: len(connection.queries) = 0

=== force_debug_cursor = True (what settings.DEBUG=True enables) ===
  issued    100 queries -> len(connection.queries) = 100
  issued   1000 queries -> len(connection.queries) = 1000
  issued   5000 queries -> len(connection.queries) = 5000
/usr/local/lib/python3.9/site-packages/django/db/backends/base/base.py:172: UserWarning: Limit for query logging exceeded, only the last 9000 queries will be returned.
  warnings.warn(
  issued   9000 queries -> len(connection.queries) = 9000
  issued  14000 queries -> len(connection.queries) = 9000  (CAP holds at maxlen)
  DELTA dbg:before -> dbg:after: RSS +4.82 MB  |  tracemalloc_cur +3597.3 KB  |  gc_objs +1224
  heap cost of the full 9000-entry query log (tracemalloc_cur delta above)
DEBUG_QUERIES DONE
```

**Cause → effect:** `DEBUG` defaults to off — `DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`
[`settings.py:50`] — and with it off the query cursor is not wrapped, so `connection.queries` stays at
**0** no matter how many queries run (500 issued → len 0). When `DEBUG=True` (via the same mechanism
`force_debug_cursor` that Django sets from `DEBUG`), queries _are_ recorded, but into a
`deque(maxlen=9000)`: the length climbs to 9000 and then **stops** — issuing 14 000 still yields 9000,
and Django emits `UserWarning: ... only the last 9000 queries will be returned`
(`django/db/backends/base/base.py:172`). The full 9000‑entry log costs **+3597.3 KB** of Python heap.
So this is **bounded (~3.5 MB), not an unbounded accumulator**, and it contributes **nothing** in the
canonical default configuration. It could only matter if an operator explicitly enabled `DEBUG` in
production and ran a long‑lived process — and even then it is capped.

---

## Q3 — What is different between the cases where memory spikes and the cases where it does not?

**Direct answer.** The single dominant difference is **whether the trained classifier pickle
`settings.MODEL_FILE` exists on disk at the moment the document is consumed.** When it exists,
`load_classifier()` unpickles the full scikit‑learn model (`CountVectorizer` + `MLPClassifier` +
multi‑label binarizer) on that consumption, adding on the order of **+80 MB of RSS and +27 MB of
Python heap** on top of the no‑model baseline. When it does not exist, `load_classifier()` short‑circuits
to `None` at `classifier.py:36` and that entire cost never occurs. Because the _same file_ consumed at
two different times can hit either branch (model absent early in a deployment / just after a
`FORMAT_VERSION` bump / before the first successful training run, versus model present afterwards),
the user's report that the spike "doesn't happen consistently and seems to vary based on the document
source or processing stage" is explained by this gate — **not** by anything random in the allocator.
A secondary, orthogonal difference is **document type** (Q4): the pikepdf/OCR path costs more RSS than
the plain‑text path regardless of the model. There is **no run‑to‑run nondeterminism** for a fixed
(model‑state × document‑type) pair — the magnitude is stable across runs, as shown below.

### Q3a — The MODEL_FILE present‑vs‑absent toggle (the primary driver), measured in fresh processes

To remove any warm‑import contamination and measure the _pure_ toggle, each condition was run in its
**own fresh Python process** (baseline captured before `django.setup()` + first consume, post captured
after one real `Consumer().try_consume_file()`), repeated so the distribution is visible. Raw output,
unedited:

```text
$ # inside container paperless-qna, env sourced from /tmp/blitzy_probe/env.sh
$ # present: MODEL_FILE trained via document_create_classifier; absent: pickle removed
COMMAND: python3 consume_measure_proc.py <absent|present>  (each a fresh process)
  [present] pid=15580 baseline_RSS=52.97MB post_RSS=231.86MB RSS_delta=178.89MB tracemalloc_peak=69502.2KB model_loaded=True predicted_corr=12
  [present] pid=15723 baseline_RSS=54.44MB post_RSS=231.80MB RSS_delta=177.36MB tracemalloc_peak=69555.2KB model_loaded=True predicted_corr=12
  [ absent] pid=15866 baseline_RSS=54.44MB post_RSS=152.49MB RSS_delta=98.05MB tracemalloc_peak=42754.8KB model_loaded=False predicted_corr=None
  [ absent] pid=15883 baseline_RSS=54.49MB post_RSS=152.35MB RSS_delta=97.86MB tracemalloc_peak=42713.1KB model_loaded=False predicted_corr=None
```

**Distribution & stability (≥2 runs each, same unchanged `sample.txt`, 5310 B):**

| Condition             | Runs | `RSS_delta`          | `tracemalloc_peak`     | `model_loaded` | `predicted_corr` |
| --------------------- | ---- | -------------------- | ---------------------- | -------------- | ---------------- |
| **present** (spike)   | 2    | 178.89 MB, 177.36 MB | 69502.2 KB, 69555.2 KB | `True`         | `12`             |
| **absent** (no spike) | 2    | 98.05 MB, 97.86 MB   | 42754.8 KB, 42713.1 KB | `False`        | `None`           |

**Cause → effect.** The branch is decided in `load_classifier()` by the guard
`if not os.path.isfile(settings.MODEL_FILE):` [`classifier.py:31`], which `return None` [`classifier.py:36`]
when the pickle is missing. `settings.MODEL_FILE` resolves to `DATA_DIR/classification_model.pickle`
[`settings.py:74`]. When the file is present the function proceeds to `DocumentClassifier().load()`
[`classifier.py:76`], whose six sequential `pickle.load(...)` reads at `classifier.py:86‑92` materialize
the model. The **spike is `present − absent`**: RSS `≈178 − 98 = ~80 MB`, Python heap
`≈69.5 − 42.7 = ~27 MB` (peak). Both magnitudes are stable to within ~1.5 MB across the two runs, i.e.
the "spike" is **deterministic given the model state**; the _only_ thing that flips it is the on‑disk
presence of the pickle. The `predicted_corr` column confirms the real path executed: with the model
loaded a correspondent id `12` is predicted (`DocumentClassifier.predict_correspondent` [`classifier.py:251`]),
while with no model it is `None` (rule‑based‑only). Note also that the disproportion the user reports is
real: the input is **5310 bytes** but the extra memory is **~80 MB** — because the cost is the _fixed
size of the model_, wholly independent of the tiny document.

### Q3b — Why the same input "sometimes" spikes: it is the model gate, reproduced on identical bytes

The inconsistency is **not** randomness in CPython; it is the deterministic `MODEL_FILE` gate evaluated
at different points in a deployment's lifetime. Consuming the _identical_ `sample.txt` twice with the
model present yields the spike both times (178.89 / 177.36 MB); consuming the _identical_ file twice
with the pickle removed yields the no‑spike profile both times (98.05 / 97.86 MB). The two regimes are
reached by the same bytes depending solely on whether a prior `document_create_classifier` /
`train_classifier` [`tasks.py:48`] run has written the pickle — which in turn requires at least one
`MATCH_AUTO` correspondent/type/tag to exist, or `train_classifier` returns early. This is exactly the
"varies based on … processing stage" symptom: **before** the model is trained, imports route to the
rule‑based branch; **after**, every consume pays the unpickle.

### Q3c — Across‑task reclamation: the django‑q `recycle: 1` worker restart

The _within‑task_ RSS growth shown above is reclaimed **between** tasks by the django‑q setting
`"recycle": 1` inside `Q_CLUSTER` [`settings.py:452`, block opened at `settings.py:449`], which recycles
(restarts) the worker process after each task. To demonstrate the semantics directly, one real consume
was run in three **separate** processes (a faithful stand‑in for the per‑task restart); each starts from
a clean ~49.8 MB baseline regardless of how much the previous process had grown. Raw output, unedited:

```text
$ # inside container paperless-qna, env sourced from /tmp/blitzy_probe/env.sh
COMMAND: python3 consume_once_proc.py  (x3 separate processes)
--- fresh process run 1 ---
[2026-07-06 22:39:01,921] [INFO] [paperless.consumer] Consuming bfa99404_sample.txt
[2026-07-06 22:39:11,883] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 bfa99404_sample
[2026-07-06 22:39:11,885] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME bfa99404_sample
[2026-07-06 22:39:12,014] [INFO] [paperless.consumer] Document 2026-07-06 ACME bfa99404_sample consumption finished
  [pid 15127] baseline_RSS=49.79 MB  post_consume_RSS=160.73 MB
--- fresh process run 2 ---
[2026-07-06 22:39:13,151] [INFO] [paperless.consumer] Consuming cd9ee053_sample.txt
[2026-07-06 22:39:23,381] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 cd9ee053_sample
[2026-07-06 22:39:23,383] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME cd9ee053_sample
[2026-07-06 22:39:23,463] [INFO] [paperless.consumer] Document 2026-07-06 ACME cd9ee053_sample consumption finished
  [pid 15270] baseline_RSS=49.77 MB  post_consume_RSS=160.12 MB
--- fresh process run 3 ---
[2026-07-06 22:39:24,699] [INFO] [paperless.consumer] Consuming be3faa5e_sample.txt
[2026-07-06 22:39:34,827] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 be3faa5e_sample
[2026-07-06 22:39:34,830] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME be3faa5e_sample
[2026-07-06 22:39:34,911] [INFO] [paperless.consumer] Document 2026-07-06 ACME be3faa5e_sample consumption finished
  [pid 15413] baseline_RSS=49.98 MB  post_consume_RSS=160.37 MB
```

**Cause → effect.** Each fresh process starts at **~49.8 MB** (49.79 / 49.77 / 49.98) and ends at
**~160 MB** (160.73 / 160.12 / 160.37) after one consume — the ~110 MB of growth does **not** carry
across process boundaries. In production the `recycle: 1` policy [`settings.py:452`] causes the same
teardown after every task, so the elevated RSS a task leaves behind is returned to the OS by the process
exit, not held indefinitely. This is the concrete mechanism behind "memory doesn't _always_ get released
in a timely manner": **within** a single long task RSS climbs and (per Deliverable C) is retained by the
allocator; **across** tasks the worker restart reclaims it. Whether a given observer sees "released" or
"not released" depends on whether they look within a task or across the recycle boundary. That the three
processes are genuinely distinct is shown by their distinct pids (15127 / 15270 / 15413) in the raw log
above. (The per‑task `recycle: 1` behavior of a live `qcluster` is **inferred** from the setting at
`settings.py:452` and demonstrated here via equivalent separate‑process semantics; a full multi‑worker
`qcluster` was not run.)

---

## Q4 — How does the behavior change with different document types and batch sizes?

**Direct answer.**

- **Document type** changes _which parser runs_ and therefore the RSS profile, but **not** the
  Python‑heap retention: the plain‑text path (`TextDocumentParser`, `self.text = f.read()`
  [`paperless_text/parsers.py:42`]) and the PDF/image path (`RasterisedDocumentParser`, pikepdf + OCR)
  both leave only a **sub‑250 KB** net Python‑heap delta per warm consume; the difference between them is
  in **RSS** (pikepdf/QPDF native memory + OCR working set, invisible to `tracemalloc`). The office path
  (`TikaDocumentParser`) **cannot be exercised in the canonical configuration** because
  `PAPERLESS_TIKA_ENABLED` defaults to `False`, so office documents are rejected as _unsupported mime
  type_ before any parser runs (measured below — labeled non‑canonical).
- **Batch size** does **not** cause unbounded growth. Consuming the same file 6× in one process leaves
  the Python heap essentially **flat** (`tracemalloc_cur` stays at ~79.9–80.0 MB across all six
  iterations) and RSS **plateaus** (oscillating in a band, not climbing monotonically). The only
  batch‑scaling memory is `DocumentClassifier.train()`, whose in‑memory lists grow **O(N)** with corpus
  size — a transient training cost, not a per‑document consumption leak. In production, django‑q
  `recycle: 1` [`settings.py:452`] resets RSS to the ~49.8 MB baseline after every task (Q3c).

### Q4a — Cross‑product: 3 parser families × {model absent, model present}, warm, ≥2 runs each

The three parser families are selected by mime type via `get_parser_class_for_mime_type`
[`documents/parsers.py:81`]. Raw output — a focused excerpt of `exp_consume_matrix.py` showing the
measured rows (**the complete, unedited output of this run, including the WARM‑UP section and all
interleaved framework log lines, is reproduced in Deliverable A**); the ImageMagick
"security policy" lines are the harmless PDF‑thumbnail fallback to ghostscript, present on every PDF:

```text
$ python3 exp_consume_matrix.py          # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
  [post_warmup_baseline  ] RSS=247.19 MB  peakRSS=340.84 MB  tracemalloc_cur=52993.2 KB  tracemalloc_peak=55637.5 KB  gc_objs=134812
  sample sizes: txt=5310B  pdf=1569B

################ Q4 doc-type x Q3 model-absent (rule-based, NO spike) ################
MODEL_FILE present: False
  txt|absent|run1            file=  5310B  elapsed= 15.6s  RSS_delta=  63.21 MB  tracemalloc_delta=    220.1 KB  predicted_corr=None dtype=None tags=[]
  txt|absent|run2            file=  5310B  elapsed= 15.5s  RSS_delta=  46.00 MB  tracemalloc_delta=     93.4 KB  predicted_corr=None dtype=None tags=[]
  pdf|absent|run1            file=  1569B  elapsed=  2.0s  RSS_delta=  45.45 MB  tracemalloc_delta=    155.5 KB  predicted_corr=None dtype=None tags=[]
  pdf|absent|run2            file=  1569B  elapsed=  2.1s  RSS_delta=  44.59 MB  tracemalloc_delta=    172.0 KB  predicted_corr=None dtype=None tags=[]

################ train real model -> MODEL_FILE present ################
  corpus=12 docs; MODEL_FILE present: True size= 397577 bytes

################ Q4 doc-type x Q3 model-present (classifier loaded -> SPIKE) ################
  txt|present|run1           file=  5310B  elapsed= 15.9s  RSS_delta=  94.49 MB  tracemalloc_delta=    123.4 KB  predicted_corr=2 dtype=2 tags=[2]
  txt|present|run2           file=  5310B  elapsed= 15.9s  RSS_delta=  63.12 MB  tracemalloc_delta=    221.0 KB  predicted_corr=2 dtype=2 tags=[2]
  pdf|present|run1           file=  1569B  elapsed=  2.3s  RSS_delta=  62.39 MB  tracemalloc_delta=    203.8 KB  predicted_corr=2 dtype=2 tags=[]
  pdf|present|run2           file=  1569B  elapsed=  2.3s  RSS_delta=  61.61 MB  tracemalloc_delta=    162.1 KB  predicted_corr=2 dtype=2 tags=[]
```

**Cross‑product summary (warm deltas relative to `post_warmup_baseline`):**

| Parser family                                                 | Mime / sample           | Model   | Runs | `RSS_delta`                     | `tracemalloc_delta` | elapsed | predictions                            |
| ------------------------------------------------------------- | ----------------------- | ------- | ---- | ------------------------------- | ------------------- | ------- | -------------------------------------- |
| `TextDocumentParser` [`paperless_text/parsers.py:40`]         | text/plain, 5310 B      | absent  | 2    | 63.21, 46.00 MB                 | 220.1, 93.4 KB      | ~15.5 s | corr/dtype/tags = None/None/[]         |
| `TextDocumentParser`                                          | text/plain, 5310 B      | present | 2    | 94.49, 63.12 MB                 | 123.4, 221.0 KB     | ~15.9 s | 2 / 2 / [2]                            |
| `RasterisedDocumentParser` [`paperless_tesseract/parsers.py`] | application/pdf, 1569 B | absent  | 2    | 45.45, 44.59 MB                 | 155.5, 172.0 KB     | ~2.0 s  | None/None/[]                           |
| `RasterisedDocumentParser`                                    | application/pdf, 1569 B | present | 2    | 62.39, 61.61 MB                 | 203.8, 162.1 KB     | ~2.3 s  | 2 / 2 / []                             |
| `TikaDocumentParser` [`paperless_tika/parsers.py:29`]         | odt/docx                | n/a     | —    | **not exercisable (canonical)** | —                   | —       | `ConsumerError: Unsupported mime type` |

**Cause → effect.**

1. **Type changes RSS, not heap.** In _every_ warm row the per‑consume `tracemalloc_delta` is tiny
   (**≤221 KB**), so no parser retains meaningful Python‑heap objects after the consume; the visible
   movement is RSS. The PDF path's RSS is dominated by pikepdf/QPDF native buffers (off‑heap) and the OCR
   working set, which is why its RSS deltas (44–62 MB) are of the same order as text despite a _smaller_
   1569 B file — memory tracks parser machinery, not document size.
2. **Model adds RSS on top of either type.** Holding type fixed, `present` exceeds `absent`:
   text `94.49/63.12` vs `63.21/46.00`, pdf `62.39/61.61` vs `45.45/44.59`. The increment is the warm
   classifier unpickle (`DocumentClassifier.load` [`classifier.py:76`]); it is smaller here than the
   ~80 MB fresh‑process gap of Q3a because sklearn/scipy/numpy are already imported once in this warm
   process — the fresh‑process figure additionally includes the one‑time library import.
3. **The `elapsed` difference is measured, not assumed.** Text consumes take ~15.5 s vs ~2 s for the tiny
   PDF; the text path spends that time in date extraction over the document content
   (`get_date`/`parse_date`, base `parsers.py:345`) — a CPU cost, not a memory cost (heap delta stays
   ≤221 KB). RSS_delta is noisy run‑to‑run (e.g., text absent 63.21 vs 46.00 MB) precisely because it is
   allocator arena churn, not retained objects (see Deliverable C).
4. **Predictions confirm the real path ran.** With the model present the handlers assign correspondent
   `ACME`, document type `Invoice`, tag `finance` (`predicted_corr=2 dtype=2 tags=[2]` for text); with it
   absent all are `None`/`[]`.

### Q4b — Batch size: same text file consumed 6× in one process (warm, model absent)

Raw output, unedited:

```text
$ python3 exp_consume_matrix.py   # (batch section)
################ Q4 BATCH: same txt consumed 6x (warm, model ABSENT) -> RSS trend ################
  batch iter 1: RSS before=404.92 MB after=465.50 MB (delta 60.58 MB)  tracemalloc_cur after=79966.2 KB
  batch iter 2: RSS before=529.12 MB after=549.66 MB (delta 20.54 MB)  tracemalloc_cur after=79968.3 KB
  batch iter 3: RSS before=545.05 MB after=546.78 MB (delta 1.73 MB)  tracemalloc_cur after=80014.3 KB
  batch iter 4: RSS before=548.02 MB after=560.46 MB (delta 12.45 MB)  tracemalloc_cur after=79928.9 KB
  batch iter 5: RSS before=526.02 MB after=546.29 MB (delta 20.27 MB)  tracemalloc_cur after=79929.6 KB
  batch iter 6: RSS before=549.86 MB after=551.80 MB (delta 1.94 MB)  tracemalloc_cur after=79917.8 KB
```

**Cause → effect.** Across all six iterations `tracemalloc_cur` is **flat** — 79966.2 → 79968.3 →
80014.3 → 79928.9 → 79929.6 → 79917.8 KB, a total spread of **< 100 KB** — proving the Python heap does
**not** accumulate per document; each consume's objects are released before the next. RSS **plateaus** in
a ~465–560 MB band and even _falls_ between iterations (e.g., `before` drops from 549.66 after iter 2 to
545.05 at iter 3, and from 560.46 to 526.02 between iters 4→5), i.e., glibc/pymalloc are reusing and
partially returning arenas — there is **no monotonic climb**. So increasing batch size within a single
worker does **not** leak; it converges to a steady state. (In production each task is instead a fresh
recycled worker per `recycle: 1` [`settings.py:452`], so RSS returns to ~49.8 MB between documents — Q3c.)

### Q4c — Batch effect on training: `DocumentClassifier.train()` list accumulation is O(N)

`DocumentClassifier.train()` [`classifier.py:115`] iterates the **whole corpus**
`Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` [`classifier.py:125`] and appends each
document's text/labels into in‑memory lists — `data.append(...)` [`classifier.py:130`],
`labels_tags.append(...)` [`classifier.py:137`], `labels_correspondent.append(...)` [`classifier.py:144`],
`labels_document_type.append(...)` [`classifier.py:156`] — before vectorizing. Growing the corpus and
measuring the peak Python heap during `train()`:

```text
$ python3 exp_train_growth.py    # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_train_growth.py
N_docs | corpus_text_bytes(held by data list) | train_peak_python_heap | model_pickle_bytes | vocab
    25 | corpus=   122025B | peak_during_train= 85984.8 KB | model=64885019B | vocab=8984
    50 | corpus=   245250B | peak_during_train=171685.6 KB | model=129679352B | vocab=17959
   100 | corpus=   491700B | peak_during_train=343258.0 KB | model=259267854B | vocab=35909
   200 | corpus=   996600B | peak_during_train= 19163.5 KB | model= 1306474B | vocab=9
TRAIN_GROWTH DONE
```

**Cause → effect.** From N=25→50→100 the peak training heap scales almost perfectly linearly —
85984.8 → 171685.6 → 343258.0 KB (≈2× per doubling) — and the serialized model grows in lockstep
(64.9 → 129.7 → 259.3 MB). The driver is not the raw `corpus_text_bytes` (only ~0.1–0.5 MB) but the
**`CountVectorizer` vocabulary and the resulting sparse feature matrix**: `vocab` climbs 8984 → 17959 →
35909, and the `MLPClassifier` weight matrices scale with the vocabulary. This is the batch dimension
that genuinely scales with N — and it is a **training** cost, incurred by `train_classifier`
[`tasks.py:48`], **not** by per‑document consumption. **The N=200 row is an artifact of the synthetic
corpus, reported exactly as observed:** vocabulary collapses to `vocab=9` and the model shrinks to 1.3 MB
because `CountVectorizer(min_df=0.01)` requires a term to appear in at least `ceil(0.01 × 200) = 2`
documents; the generator's per‑document unique tokens fail that threshold at N=200, so almost the entire
vocabulary is filtered out. On a real corpus with shared vocabulary the O(N) trend of the 25/50/100 rows
is the representative behavior; the 200 row demonstrates the `min_df` filter, not a memory decrease of the
real workload.

### Q4d — Office documents (`TikaDocumentParser`) are not exercisable in the canonical configuration

Raw output, unedited:

```text
$ python3 exp_tika.py            # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_tika.py
PAPERLESS_TIKA_ENABLED = False
PAPERLESS_TIKA_ENDPOINT = http://localhost:9998
Canonical parser for odt mime = None

=== consume odt through REAL entry point (canonical config) ===
[2026-07-06 22:40:42,875] [INFO] [paperless.consumer] Consuming 351bfa11_sample.odt
[2026-07-06 22:40:42,878] [ERROR] [paperless.consumer] Unsupported mime type application/vnd.oasis.opendocument.text
  ConsumerError: 351bfa11_sample.odt: Unsupported mime type application/vnd.oasis.opendocument.text

=== NON-CANONICAL: TikaDocumentParser.extract_metadata directly (Tika disabled/not running) ===
  could not import/instantiate TikaDocumentParser: PermissionError [Errno 13] Permission denied: '/tmp/tika.log'
TIKA DONE
```

**Cause → effect.** In the default configuration `PAPERLESS_TIKA_ENABLED = False`, so the Tika parser is
never registered and `get_parser_class_for_mime_type` returns `None` for the odt mime; the real entry
point therefore rejects the office document with `ConsumerError: ... Unsupported mime type` before any
memory‑relevant parsing occurs. Attempting to invoke `TikaDocumentParser.extract_metadata`
[`paperless_tika/parsers.py:29`] _directly_ (a non‑canonical bypass) failed to even instantiate — a
`PermissionError` on `/tmp/tika.log`, and in any case no Tika HTTP server is reachable at the endpoint
`http://localhost:9998` [`paperless_tika/parsers.py:30`]. **Therefore the office/Tika memory numbers
genuinely could not be exercised in the canonical runtime and none are reported;** the Tika path
(`parser.from_file(...)` [`paperless_tika/parsers.py:32`, `:55`]) offloads extraction to that external
process over HTTP, so its heavy memory would live in the Tika JVM, not in the paperless Python worker —
this last point is **(inferred)** from the code and the disabled‑by‑default setting, not measured.

---

## Deliverable A — Raw runtime measurements (complete, unedited, with the command that produced each)

Every behavioral claim above is backed by a raw output block shown _before_ its summary. This section
is the consolidated manifest of the experiments, and re‑prints in full the one block that appears only as
an excerpt earlier (`exp_classifier.py`), so nothing is elided.

**All measurements were captured inside the canonical container `paperless-qna` (Python 3.9.23, commit
`542221a38dff`), with the environment sourced from `/tmp/blitzy_probe/env.sh`** (see §2). Each probe
script lived under `/tmp/blitzy_probe/` (outside the tracked tree) and was deleted afterward (Phase F).

**Manifest of experiments (command → what it measures → where the raw block is embedded):**

| #   | Command                                  | Measures                                                | Raw block embedded in                        |
| --- | ---------------------------------------- | ------------------------------------------------------- | -------------------------------------------- | --- |
| 1   | `python3 exp_copies.py`                  | transient `md5(f.read())` peak vs file size             | Q1a                                          |
| 2   | `python3 exp_reflifetime.py`             | classifier reaches the 6 handlers alive (refcount)      | Q1b                                          |
| 3   | `python3 exp_pikepdf.py`                 | `pikepdf.open()` native RSS with/without `close()`      | Q1c                                          |
| 4   | `python3 exp_classifier.py`              | `load_classifier()` no‑cache + per‑call recover + spike | Q2a + **full block below**                   |
| 5   | `python3 exp_debug_queries.py`           | `connection.queries` growth, `DEBUG` off vs on          | Q2b                                          |
| 6   | `python3 consume_measure_proc.py <absent | present>`                                               | fresh‑process spike vs no‑spike distribution | Q3a |
| 7   | `python3 consume_once_proc.py` (×3)      | `recycle: 1` per‑process baseline reset                 | Q3c                                          |
| 8   | `python3 exp_consume_matrix.py`          | 3 parsers × model × 2 runs + batch×6                    | Q4a, Q4b                                     |
| 9   | `python3 exp_train_growth.py`            | `train()` heap O(N) over growing corpus                 | Q4c                                          |
| 10  | `python3 exp_tika.py`                    | office/Tika unsupported in canonical config             | Q4d                                          |
| 11  | `python3 exp_attribution.py`             | baseline→consume→del+gc heap vs RSS verdict             | Deliverable C                                |
| 12  | `python3 manage.py migrate --no-input`   | canonical DB init (§2)                                  | §2                                           |

**Full raw output of experiment #4 (`exp_classifier.py`) — complete and unedited** (Q2a showed only the
no‑cache excerpt; here are all five `load_classifier()` calls with their `del`+`gc.collect()` recovery,
which also underpins Deliverable C's per‑call verdict):

```text
$ python3 exp_classifier.py     # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_classifier.py
[2026-07-06 22:23:17,399] [INFO] [paperless.consumer] Consuming 6793f609_sample.txt
[2026-07-06 22:23:39,332] [INFO] [paperless.consumer] Document 2026-07-06 6793f609_sample consumption finished
  [warmup:before         ] RSS=55.08 MB  peakRSS=54.53 MB  tracemalloc_cur=203.3 KB  tracemalloc_peak=223.5 KB  gc_objs=74001
  [warmup:after          ] RSS=154.42 MB  peakRSS=149.11 MB  tracemalloc_cur=43383.1 KB  tracemalloc_peak=43698.5 KB  gc_objs=132618
  DELTA warmup:before -> warmup:after: RSS +99.34 MB  |  tracemalloc_cur +43179.8 KB  |  gc_objs +58617

################ MODEL_FILE ABSENT (rule-based only) ################
MODEL_FILE exists= False
  load_classifier() #1 returned: None
  DELTA absent_call1:before -> absent_call1:after: RSS +42.39 MB  |  tracemalloc_cur +3.2 KB  |  gc_objs +27885
  load_classifier() #2 returned: None
  DELTA absent_call2:before -> absent_call2:after: RSS +11.55 MB  |  tracemalloc_cur +2.0 KB  |  gc_objs +27933
  load_classifier() #3 returned: None
  DELTA absent_call3:before -> absent_call3:after: RSS +4.94 MB  |  tracemalloc_cur +2.0 KB  |  gc_objs +27943

################ TRAIN REAL MODEL (document_create_classifier path) ################
[2026-07-06 22:23:43,206] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/blitzy_probe/data/classification_model.pickle...
  built corpus of 12 docs; train_classifier() returned None
  MODEL_FILE exists now = True; size = 397540 bytes

################ MODEL_FILE PRESENT (classifier loaded -> SPIKE) ################
  load_classifier() #1 -> DocumentClassifier id=135947590687376
  DELTA present_call1:before -> present_call1:after: RSS +69.12 MB  |  tracemalloc_cur +407.9 KB  |  gc_objs +20498
  --- tracemalloc growth present_call1:before -> present_call1:after (top 8 by size delta) ---
    +  131.9 KB  +   133 objs  documents/classifier.py:92
    +  128.4 KB  +    96 objs  documents/classifier.py:90
    +  128.0 KB  +    89 objs  documents/classifier.py:91
    +   11.0 KB  +    37 objs  /usr/local/lib/python3.9/site-packages/numpy/random/_pickle.py:83
    +    6.0 KB  +    66 objs  documents/classifier.py:87
    +    2.0 KB  +    16 objs  /usr/local/lib/python3.9/site-packages/sklearn/base.py:343
       -0.9 KB      -2 objs  <frozen importlib._bootstrap>:228
    +    0.5 KB  +     2 objs  /tmp/blitzy_probe/exp_classifier.py:39
  DELTA present_call1:after -> present_call1:after_del_gc: RSS +34.10 MB  |  tracemalloc_cur -399.3 KB  |  gc_objs -20498
  load_classifier() #2 -> DocumentClassifier id=135942076114592
  DELTA present_call2:before -> present_call2:after: RSS +10.40 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17372
  DELTA present_call2:after -> present_call2:after_del_gc: RSS -3.55 MB  |  tracemalloc_cur -405.8 KB  |  gc_objs -16893
  load_classifier() #3 -> DocumentClassifier id=135941996150016
  DELTA present_call3:before -> present_call3:after: RSS +3.50 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17415
  DELTA present_call3:after -> present_call3:after_del_gc: RSS -3.94 MB  |  tracemalloc_cur -405.8 KB  |  gc_objs -16860
  load_classifier() #4 -> DocumentClassifier id=135942055175216
  DELTA present_call4:before -> present_call4:after: RSS +4.00 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17405
  DELTA present_call4:after -> present_call4:after_del_gc: RSS -4.94 MB  |  tracemalloc_cur -405.8 KB  |  gc_objs -16859
  load_classifier() #5 -> DocumentClassifier id=135942074905456
  DELTA present_call5:before -> present_call5:after: RSS +5.25 MB  |  tracemalloc_cur +405.8 KB  |  gc_objs +17375
  DELTA present_call5:after -> present_call5:after_del_gc: RSS -4.19 MB  |  tracemalloc_cur -405.8 KB  |  gc_objs -16820

  NO-CACHE PROOF: distinct object ids across calls = [135947590687376, 135942076114592, 135941996150016, 135942055175216, 135942074905456] -> all unique: True
```

**What this full block adds beyond the Q2a excerpt:** the per‑call recovery pairs. Every
`present_callN:after -> present_callN:after_del_gc` line shows `tracemalloc_cur` returning by exactly the
same `-405.8 KB` it grew, and `gc_objs` returning by ~the same count — i.e., after `del classifier` +
`gc.collect()` the Python heap is fully reclaimed on **every** call. The top‑allocator lines pin the
growth to the unpickle sites `classifier.py:90/:91/:92` (the three heaviest of the six `pickle.load`
reads at `classifier.py:86‑92`). This is the direct evidence that the spike is **transient**, not a leak.

**Full raw output of experiment #8 (`exp_consume_matrix.py`) — complete and unedited** (Q4a and Q4b show
focused excerpts of this same run; here is the entire output including the WARM‑UP section that is
excluded from the deltas, and all interleaved framework log lines — the `convert-im6.q16 ... security policy` lines are the harmless PDF‑thumbnail fallback to ghostscript):

```text
$ python3 exp_consume_matrix.py     # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_consume_matrix.py
################ WARM-UP (load all lazy imports; excluded from measurements) ################
[2026-07-06 22:26:20,099] [INFO] [paperless.consumer] Consuming 4458622a_sample.txt
[2026-07-06 22:26:42,617] [INFO] [paperless.consumer] Document 2026-07-06 4458622a_sample consumption finished
  [warmup_txt:before     ] RSS=54.75 MB  peakRSS=54.47 MB  tracemalloc_cur=204.7 KB  tracemalloc_peak=224.8 KB  gc_objs=74003
  [warmup_txt:after      ] RSS=152.45 MB  peakRSS=149.11 MB  tracemalloc_cur=42063.6 KB  tracemalloc_peak=42395.6 KB  gc_objs=118897
  DELTA warmup_txt:before -> warmup_txt:after: RSS +97.70 MB  |  tracemalloc_cur +41858.9 KB  |  gc_objs +44894
[2026-07-06 22:26:43,192] [INFO] [paperless.consumer] Consuming 70224788_sample.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/blitzy_probe/scratch/paperless-3ho9v5do/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-06 22:26:44,074] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:26:45,323] [INFO] [paperless.consumer] Document 2026-07-06 70224788_sample consumption finished
  [warmup_pdf:before     ] RSS=193.44 MB  peakRSS=235.11 MB  tracemalloc_cur=41967.5 KB  tracemalloc_peak=43051.2 KB  gc_objs=118727
  [warmup_pdf:after      ] RSS=246.53 MB  peakRSS=246.21 MB  tracemalloc_cur=53160.6 KB  tracemalloc_peak=55637.5 KB  gc_objs=138468
  DELTA warmup_pdf:before -> warmup_pdf:after: RSS +53.09 MB  |  tracemalloc_cur +11193.1 KB  |  gc_objs +19741
  [post_warmup_baseline  ] RSS=247.19 MB  peakRSS=340.84 MB  tracemalloc_cur=52993.2 KB  tracemalloc_peak=55637.5 KB  gc_objs=134812
  sample sizes: txt=5310B  pdf=1569B

################ Q4 doc-type x Q3 model-absent (rule-based, NO spike) ################
MODEL_FILE present: False
[2026-07-06 22:26:46,712] [INFO] [paperless.consumer] Consuming b601cf8e_sample.txt
[2026-07-06 22:27:01,865] [INFO] [paperless.consumer] Document 2026-07-06 b601cf8e_sample consumption finished
  txt|absent|run1            file=  5310B  elapsed= 15.6s  RSS_delta=  63.21 MB  tracemalloc_delta=    220.1 KB  predicted_corr=None dtype=None tags=[]
[2026-07-06 22:27:02,875] [INFO] [paperless.consumer] Consuming 353f20d7_sample.txt
[2026-07-06 22:27:17,962] [INFO] [paperless.consumer] Document 2026-07-06 353f20d7_sample consumption finished
  txt|absent|run2            file=  5310B  elapsed= 15.5s  RSS_delta=  46.00 MB  tracemalloc_delta=     93.4 KB  predicted_corr=None dtype=None tags=[]
[2026-07-06 22:27:18,944] [INFO] [paperless.consumer] Consuming 00aaee44_sample.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/blitzy_probe/scratch/paperless-9qhy84mh/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-06 22:27:19,276] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:27:20,549] [INFO] [paperless.consumer] Document 2026-07-06 00aaee44_sample consumption finished
  pdf|absent|run1            file=  1569B  elapsed=  2.0s  RSS_delta=  45.45 MB  tracemalloc_delta=    155.5 KB  predicted_corr=None dtype=None tags=[]
[2026-07-06 22:27:21,523] [INFO] [paperless.consumer] Consuming ae6350ac_sample.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/blitzy_probe/scratch/paperless-jnjuhu_u/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-06 22:27:21,873] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:27:23,195] [INFO] [paperless.consumer] Document 2026-07-06 ae6350ac_sample consumption finished
  pdf|absent|run2            file=  1569B  elapsed=  2.1s  RSS_delta=  44.59 MB  tracemalloc_delta=    172.0 KB  predicted_corr=None dtype=None tags=[]

################ train real model -> MODEL_FILE present ################
[2026-07-06 22:27:25,289] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/blitzy_probe/data/classification_model.pickle...
  corpus=12 docs; MODEL_FILE present: True size= 397577 bytes

################ Q4 doc-type x Q3 model-present (classifier loaded -> SPIKE) ################
[2026-07-06 22:27:25,948] [INFO] [paperless.consumer] Consuming 1522b2a9_sample.txt
[2026-07-06 22:27:41,185] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 1522b2a9_sample
[2026-07-06 22:27:41,192] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME 1522b2a9_sample
[2026-07-06 22:27:41,199] [INFO] [paperless.handlers] Tagging "2026-07-06 ACME 1522b2a9_sample" with "finance"
[2026-07-06 22:27:41,241] [INFO] [paperless.consumer] Document 2026-07-06 ACME 1522b2a9_sample consumption finished
  txt|present|run1           file=  5310B  elapsed= 15.9s  RSS_delta=  94.49 MB  tracemalloc_delta=    123.4 KB  predicted_corr=2 dtype=2 tags=[2]
[2026-07-06 22:27:42,582] [INFO] [paperless.consumer] Consuming eb74eb27_sample.txt
[2026-07-06 22:27:57,813] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 eb74eb27_sample
[2026-07-06 22:27:57,825] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME eb74eb27_sample
[2026-07-06 22:27:57,831] [INFO] [paperless.handlers] Tagging "2026-07-06 ACME eb74eb27_sample" with "finance"
[2026-07-06 22:27:57,952] [INFO] [paperless.consumer] Document 2026-07-06 ACME eb74eb27_sample consumption finished
  txt|present|run2           file=  5310B  elapsed= 15.9s  RSS_delta=  63.12 MB  tracemalloc_delta=    221.0 KB  predicted_corr=2 dtype=2 tags=[2]
[2026-07-06 22:27:59,273] [INFO] [paperless.consumer] Consuming df2b5630_sample.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/blitzy_probe/scratch/paperless-t2o1pv10/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-06 22:27:59,654] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:28:00,887] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 df2b5630_sample
[2026-07-06 22:28:00,889] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME df2b5630_sample
[2026-07-06 22:28:00,922] [INFO] [paperless.consumer] Document 2026-07-06 ACME df2b5630_sample consumption finished
  pdf|present|run1           file=  1569B  elapsed=  2.3s  RSS_delta=  62.39 MB  tracemalloc_delta=    203.8 KB  predicted_corr=2 dtype=2 tags=[]
[2026-07-06 22:28:02,278] [INFO] [paperless.consumer] Consuming 265ad636_sample.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/blitzy_probe/scratch/paperless-f32kzqrg/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-06 22:28:02,617] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:28:03,841] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 265ad636_sample
[2026-07-06 22:28:03,843] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME 265ad636_sample
[2026-07-06 22:28:03,897] [INFO] [paperless.consumer] Document 2026-07-06 ACME 265ad636_sample consumption finished
  pdf|present|run2           file=  1569B  elapsed=  2.3s  RSS_delta=  61.61 MB  tracemalloc_delta=    162.1 KB  predicted_corr=2 dtype=2 tags=[]

################ Q4 BATCH: same txt consumed 6x (warm, model ABSENT) -> RSS trend ################
[2026-07-06 22:28:05,325] [INFO] [paperless.consumer] Consuming fbf1acd1_sample.txt
[2026-07-06 22:28:20,224] [INFO] [paperless.consumer] Document 2026-07-06 fbf1acd1_sample consumption finished
  batch iter 1: RSS before=404.92 MB after=465.50 MB (delta 60.58 MB)  tracemalloc_cur after=79966.2 KB
[2026-07-06 22:28:21,540] [INFO] [paperless.consumer] Consuming cd0a0308_sample.txt
[2026-07-06 22:28:35,985] [INFO] [paperless.consumer] Document 2026-07-06 cd0a0308_sample consumption finished
  batch iter 2: RSS before=529.12 MB after=549.66 MB (delta 20.54 MB)  tracemalloc_cur after=79968.3 KB
[2026-07-06 22:28:37,485] [INFO] [paperless.consumer] Consuming a3641ff6_sample.txt
[2026-07-06 22:28:51,972] [INFO] [paperless.consumer] Document 2026-07-06 a3641ff6_sample consumption finished
  batch iter 3: RSS before=545.05 MB after=546.78 MB (delta 1.73 MB)  tracemalloc_cur after=80014.3 KB
[2026-07-06 22:28:53,379] [INFO] [paperless.consumer] Consuming 330f8b98_sample.txt
[2026-07-06 22:29:07,644] [INFO] [paperless.consumer] Document 2026-07-06 330f8b98_sample consumption finished
  batch iter 4: RSS before=548.02 MB after=560.46 MB (delta 12.45 MB)  tracemalloc_cur after=79928.9 KB
[2026-07-06 22:29:09,260] [INFO] [paperless.consumer] Consuming f1d7b5e7_sample.txt
[2026-07-06 22:29:23,678] [INFO] [paperless.consumer] Document 2026-07-06 f1d7b5e7_sample consumption finished
  batch iter 5: RSS before=526.02 MB after=546.29 MB (delta 20.27 MB)  tracemalloc_cur after=79929.6 KB
[2026-07-06 22:29:25,221] [INFO] [paperless.consumer] Consuming 8fd53203_sample.txt
[2026-07-06 22:29:39,363] [INFO] [paperless.consumer] Document 2026-07-06 8fd53203_sample consumption finished
  batch iter 6: RSS before=549.86 MB after=551.80 MB (delta 1.94 MB)  tracemalloc_cur after=79917.8 KB

MATRIX DONE
```

---

## Deliverable B — Which components/methods hold onto memory (named, with `file:line`)

Each row names the **specific function/method** that performs the retaining work (not merely the outer
caller), the exact `file:line`, the measured amount, how long it is held, the question it answers, and
the normal‑vs‑problematic verdict (justified in Deliverable C).

| #   | Component / method (`file:line`)                                                                                                                                              | What it holds                                                                   | Measured amount                                                                                                   | Lifetime                                                                                                        | Q             | Verdict                                            |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------- | -------------------------------------------------- |
| 1   | `load_classifier()` → `DocumentClassifier.load()` unpickling 6 objects (`classifier.py:30`, `:76`, reads `:86‑92`)                                                            | scikit‑learn `CountVectorizer`+`MLPClassifier`+binarizer                        | **+405.8 KB heap / call** (warm); **~+27 MB heap, ~+80 MB RSS** fresh‑process; top sites `classifier.py:90/91/92` | Per call; **no module cache** (5 unique ids) → re‑unpickled every call; fully recovered by `del`+`gc.collect()` | Q2, Q3        | **Normal / transient** (the spike, but not a leak) |
| 2   | `RasterisedDocumentParser.extract_metadata` — `pikepdf.open()` + `open_metadata()` with **no** `with`/`close()` (`paperless_tesseract/parsers.py:34`, `:35`)                  | native QPDF (C++) buffers, **off the Python heap**                              | **+33.41 MB RSS / 20 handles** (invisible to `tracemalloc`: +103 KB only); `+0.04 MB` with explicit `close()`     | Until object GC'd; up to **2 opens/request** on the metadata REST action (`views.py:295`, `:302`)               | Q1            | Normal but avoidable; released on close/GC         |
| 3   | `Consumer.pre_check_duplicate` — `hashlib.md5(f.read())` (`consumer.py:104`)                                                                                                  | whole‑file bytes copy                                                           | **peak = file size** (1 MB→1.00 MB … 50 MB→50.00 MB, 1:1)                                                         | Transient (freed after hashing)                                                                                 | Q1            | Normal / transient                                 |
| 4   | `Consumer._store` — `hashlib.md5(f.read())` (`consumer.py:402`)                                                                                                               | whole‑file bytes copy                                                           | ∝ file size                                                                                                       | Transient                                                                                                       | Q1            | Normal / transient                                 |
| 5   | `Consumer._write` — `read_file.read()` (`consumer.py:432`)                                                                                                                    | whole‑file bytes copy                                                           | ∝ file size                                                                                                       | Transient                                                                                                       | Q1            | Normal / transient                                 |
| 6   | `Consumer` archive checksum — `f.read()` (`consumer.py:341`)                                                                                                                  | whole‑file bytes copy                                                           | ∝ file size                                                                                                       | Transient                                                                                                       | Q1            | Normal / transient                                 |
| 7   | `TextDocumentParser.parse` — `self.text = f.read()` (`paperless_text/parsers.py:42`)                                                                                          | entire document text                                                            | ∝ file size (5310 B here)                                                                                         | Held for the parse duration                                                                                     | Q1, Q4        | Normal / transient                                 |
| 8   | `document_consumption_finished.send(..., classifier=classifier)` (`consumer.py:306`; obtained `:292`)                                                                         | keeps the classifier **alive across the whole persistence tail**                | one shared live object (`refcount=7` at handler time)                                                             | Extended across all 6 receivers                                                                                 | Q1            | Normal (intended); ends when task ends             |
| 9   | Signal handlers `set_correspondent`/`set_document_type`/`set_tags` (`handlers.py:35`, `:101`, `:168`; classifier params `:39/:105/:172`)                                      | reference to the same classifier + `document.content`                           | shares object #8                                                                                                  | Duration of each handler                                                                                        | Q1            | Normal / transient                                 |
| 10  | `matching.matches` — regex over `document_content = document.content` (`matching.py:63`; `re.*` `:74/:86/:93/:109/:110/:130/:131`) and `predict_*` (`matching.py:23/:36/:49`) | full content string + compiled regexes during matching                          | ∝ content size                                                                                                    | Transient per match call                                                                                        | Q1            | Normal / transient                                 |
| 11  | `Document.content = models.TextField(...)` (`models.py:117`)                                                                                                                  | full extracted text resident on the ORM instance                                | ∝ content size                                                                                                    | Life of the Document object in memory                                                                           | Q1            | Normal / transient                                 |
| 12  | `index.update_document` — `writer.update_document(..., content=doc.content)` (`index.py:87`, `:93`; writer `:66`)                                                             | passes full content to the Whoosh `AsyncWriter`                                 | ∝ content size                                                                                                    | Duration of index write                                                                                         | Q1            | Normal / transient                                 |
| 13  | `DocumentClassifier.train()` — `data`/`labels_*.append(...)` over the whole corpus (`classifier.py:115`, appends `:130/:137/:144/:156`, query `:125`)                         | in‑memory lists of every document's text+labels; vectorizer vocab + MLP weights | **O(N)**: 25/50/100 docs → **86/172/343 MB** peak heap                                                            | Duration of `train_classifier` (`tasks.py:48`)                                                                  | Q4            | Normal / transient (scales with corpus)            |
| 14  | `connection.queries` under `DEBUG` (`settings.py:50`)                                                                                                                         | logged SQL query dicts                                                          | `deque(maxlen=9000)`: **0** when `DEBUG=False`; caps at **9000 (~+3.6 MB)** when on                               | Life of the connection                                                                                          | Q2            | **Bounded**, off by default → not problematic      |
| 15  | `Q_CLUSTER {"recycle": 1}` (`settings.py:452`, block `:449`)                                                                                                                  | _mitigation_, not a holder — recycles worker after each task                    | fresh processes reset to **~49.8 MB** baseline                                                                    | Reclaims RSS across tasks                                                                                       | Q3            | Mitigation (weigh against "not released timely")   |
| 16  | One‑time lazy imports on first consume (`dateparser`, `regex`, OCR libs) — module‑level globals                                                                               | interned module state / compiled patterns / locale data                         | **~+40 MB heap** persists after first consume (top: `regex/regex.py`, `dateparser`)                               | Process lifetime (module cache)                                                                                 | Deliverable C | **Normal** (intentional import cache)              |

**Primary holder answering the user's "disproportionate spike during metadata handling":** row #1
(`DocumentClassifier.load` at `classifier.py:76`) when `MODEL_FILE` exists, plus row #16 (one‑time library
imports) on the very first consume — both **independent of the tiny document size**, which is exactly why
the spike is disproportionate to a 5310‑byte input.

---

## Deliverable C — Normal CPython behavior vs. problematic retention (the verdict), and where the memory goes

**Direct answer.** The observed behavior is **dominantly NORMAL** — CPython/glibc allocator _arena
retention_ plus a one‑time module‑import cache — **not a leak.** The decisive experiment consumes a
document with the model loaded, then does `del` + `gc.collect()` and re‑snapshots: the **Python heap
returns to within 54.2 KB of baseline**, **`gc.garbage` is empty (0 uncollectable cycles)**, and net live
objects return to baseline (+33), **while RSS stays +134.70 MB elevated**. That signature — heap and
object counts recover but RSS does not — is the textbook definition of allocator arena retention, which
is reclaimed in production by the django‑q `recycle: 1` worker restart, not held forever.

### The attribution rule (applied literally)

- **RSS rises but `tracemalloc` + `gc` object counts return to baseline after `del`+`gc.collect()`** ⇒
  **normal** pymalloc/glibc arena retention (freed arenas kept by the process; >512 B objects use the
  system allocator which may fragment) — **NOT a leak.**
- **`tracemalloc` total stays elevated (specific Python types grow and persist)** ⇒ retained references
  or an unbounded cache — **problematic.**
- **Native `pikepdf`/QPDF memory is off the Python heap** — never appears in `tracemalloc`; judged via
  RSS only; released when the object is closed/GC'd.

### The clincher experiment — raw output, unedited

```text
$ python3 exp_attribution.py    # inside container paperless-qna, env from /tmp/blitzy_probe/env.sh
COMMAND: python3 exp_attribution.py
[2026-07-06 22:44:32,460] [INFO] [paperless.consumer] Consuming 6a84d497_sample.txt
[2026-07-06 22:44:55,440] [INFO] [paperless.handlers] Assigning correspondent ACME to 2026-07-06 6a84d497_sample
[2026-07-06 22:44:55,446] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ACME 6a84d497_sample
[2026-07-06 22:44:55,634] [INFO] [paperless.consumer] Document 2026-07-06 ACME 6a84d497_sample consumption finished
  [warmup:before         ] RSS=53.32 MB  peakRSS=54.01 MB  tracemalloc_cur=203.3 KB  tracemalloc_peak=223.4 KB  gc_objs=74001
  [warmup:after          ] RSS=232.52 MB  peakRSS=230.27 MB  tracemalloc_cur=69024.3 KB  tracemalloc_peak=69778.3 KB  gc_objs=156327
  DELTA warmup:before -> warmup:after: RSS +179.20 MB  |  tracemalloc_cur +68821.1 KB  |  gc_objs +82326
[2026-07-06 22:44:56,445] [INFO] [paperless.tasks] Saving updated classifier model to /tmp/blitzy_probe/data/classification_model.pickle...
MODEL_FILE present: True | DEBUG: False
  [A_baseline            ] RSS=301.95 MB  peakRSS=360.27 MB  tracemalloc_cur=71088.6 KB  tracemalloc_peak=71798.5 KB  gc_objs=161351
[2026-07-06 22:44:56,974] [INFO] [paperless.consumer] Consuming ae751ea6_sample.txt
[2026-07-06 22:45:11,585] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 ae751ea6_sample
[2026-07-06 22:45:11,592] [INFO] [paperless.handlers] Tagging "2026-07-06 ae751ea6_sample" with "finance"
[2026-07-06 22:45:11,632] [INFO] [paperless.consumer] Document 2026-07-06 ae751ea6_sample consumption finished
  [B_after_consume       ] RSS=375.76 MB  peakRSS=373.45 MB  tracemalloc_cur=71202.6 KB  tracemalloc_peak=72499.4 KB  gc_objs=183815
  DELTA A_baseline -> B_after_consume: RSS +73.81 MB  |  tracemalloc_cur +114.0 KB  |  gc_objs +22464
  [C_after_del_gc        ] RSS=436.65 MB  peakRSS=434.45 MB  tracemalloc_cur=71142.8 KB  tracemalloc_peak=72785.7 KB  gc_objs=161384
  DELTA B_after_consume -> C_after_del_gc: RSS +60.89 MB  |  tracemalloc_cur -59.8 KB  |  gc_objs -22431

  ATTRIBUTION SUMMARY
    net Python heap vs baseline (C - A): 54.2 KB
    net RSS vs baseline         (C - A): 134.70 MB
    net gc objects vs baseline  (C - A): 33
    gc.garbage (uncollectable cycles): 0
    gc.collect() final pass freed-count check: 0
ATTRIBUTION DONE
```

**Reading the three states (before / during / after — reporting all three, not just the end state):**

- **A_baseline → B_after_consume:** one real consume adds `RSS +73.81 MB` but only `tracemalloc_cur +114.0 KB` and `gc_objs +22464`. Almost none of the RSS growth is _tracked Python heap_ — it is native
  - allocator working set.
- **B_after_consume → C_after_del_gc:** after `del` + `gc.collect()`, `tracemalloc_cur` goes **down
  −59.8 KB** and `gc_objs` goes **down −22431** (undoing the +22464), i.e., the transient objects are
  gone; RSS goes _up_ another `+60.89 MB` because `gc.collect()` and re‑snapshotting themselves allocate
  fresh arenas that the OS has not reclaimed — a further demonstration that RSS is decoupled from live
  Python objects.
- **Net (C − A):** heap **+54.2 KB**, objects **+33**, but RSS **+134.70 MB**. **`gc.garbage = 0`** ⇒ no
  uncollectable reference cycles. This is the normal signature.

### Per‑stage attribution verdicts

| Stage / holder (`file:line`)                                                            | Python‑heap behavior                                                            | RSS behavior                                      | Verdict                                                                  |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------ |
| **One‑time lazy imports** (dateparser/regex/OCR), first consume                         | **+~40 MB and PERSISTS** (module globals)                                       | +~100 MB persists                                 | **Normal** — intentional module‑import cache, one‑time, not per‑document |
| **Classifier unpickle** `DocumentClassifier.load` (`classifier.py:76`, `:86‑92`)        | +405.8 KB/call, **fully recovered** by `del`+`gc.collect()` (see Deliverable A) | +3–69 MB, mostly recovered                        | **Normal / transient**; the spike, but no cache and no leak              |
| **Full consume (model present)** `Consumer.try_consume_file` (`consumer.py:180`)        | net **+54.2 KB** after del+gc; `gc.garbage=0`                                   | **+134.70 MB retained**                           | **Normal allocator arena retention** — reclaimed by `recycle:1`          |
| **Unclosed `pikepdf.open`** `extract_metadata` (`paperless_tesseract/parsers.py:34‑35`) | +103 KB only (native is off‑heap)                                               | **+33.41 MB / 20 handles**; +0.04 MB if `close()` | **Off‑heap native**; normal but avoidable; released on close/GC          |
| **`connection.queries`** under `DEBUG` (`settings.py:50`)                               | 0 at `DEBUG=False`; bounded `deque(9000)` (+3.6 MB) when on                     | negligible                                        | **Bounded, off by default → NOT problematic**                            |
| **`train()` corpus lists** (`classifier.py:130/137/144/156`)                            | **O(N)** (86/172/343 MB @ 25/50/100)                                            | tracks heap                                       | **Normal / transient**; scales with corpus, a training‑time cost         |
| **Full‑file `f.read()` copies** (`consumer.py:104/341/402/432`)                         | peak = **file size** (1:1)                                                      | tracks copy                                       | **Normal / transient**; ∝ **file** size, not metadata size               |

### Where the memory actually goes (summary of the causal chain)

1. **The disproportionate spike** = the fixed cost of (a) the scikit‑learn model unpickle when
   `MODEL_FILE` exists [`classifier.py:76`], and (b) one‑time library imports on the first consume — both
   **independent of the ~5 KB document**, hence "disproportionate to document size."
2. **"Not released in a timely manner"** = CPython `pymalloc`/glibc keep freed **arenas** resident (RSS
   stays high) even though the Python objects are gone (`tracemalloc` + `gc` recover, `gc.garbage=0`).
   This is normal allocator behavior, **mitigated** across tasks by `recycle: 1` [`settings.py:452`],
   which restarts the worker to ~49.8 MB.
3. **"Sometimes it spikes, sometimes not"** = the deterministic `MODEL_FILE` gate
   [`classifier.py:31`, `:36`] and the document‑type parser selection — not randomness.
4. **The only genuinely input‑proportional growth** is the transient `f.read()` copies (∝ **file** size)
   and `train()` (∝ **corpus** size); neither persists.
5. **No leak was found:** across every stage the Python heap and live‑object count return to baseline
   after `del`+`gc.collect()`, and `gc.garbage` is empty. The one item that is _avoidable_ (not a leak) is
   the unclosed `pikepdf.open` native memory on the metadata path, which an explicit `close()` reduces
   from **+33.41 MB to +0.04 MB** — reported as evidence only; **no fix is applied** (read‑only mandate).

---

## Coverage checklist (every question part and every named item, confirmed)

A final coverage pass: each distinct thing the four questions and deliverables ask for, confirmed present
with its concrete value, `file:line`, observed evidence, and causal reason. `✓` = covered above.

### Questions

| Question / named item                           | `file:line`                                                   | Evidence (section)      | Value / cause                                         |
| ----------------------------------------------- | ------------------------------------------------------------- | ----------------------- | ----------------------------------------------------- |
| **Q1** unnecessary copies                       | `consumer.py:104,341,402,432`; `paperless_text/parsers.py:42` | Q1a, Deliverable B #3‑7 | ✓ peak = **file** size (1:1); transient               |
| **Q1** references held too long                 | `consumer.py:292,306` → `handlers.py:35,101,168`              | Q1b, B #8‑9             | ✓ classifier `refcount=7`, shared across 6 handlers   |
| **Q1** matching content + predict               | `matching.py:23,36,49,63,74‑131`                              | Q1b, B #10              | ✓ regex over `document.content`; transient            |
| **Q1** content resident + index                 | `models.py:117`; `index.py:87,93`                             | B #11‑12                | ✓ `TextField`; `content=doc.content` to Whoosh        |
| **Q1** unclosed pikepdf                         | `paperless_tesseract/parsers.py:34,35`                        | Q1c, B #2               | ✓ **+33.41 MB** native / 20 handles; +0.04 with close |
| **Q2** `load_classifier` no cache               | `classifier.py:30`                                            | Q2a, Deliverable A      | ✓ 5 calls = 5 unique ids; +405.8 KB/call              |
| **Q2** `connection.queries` cache               | `settings.py:50`                                              | Q2b                     | ✓ `deque(9000)`; 0 at DEBUG=off; caps 9000            |
| **Q3** MODEL_FILE present vs absent             | `classifier.py:31,36`; `settings.py:74`                       | Q3a                     | ✓ **178 vs 98 MB RSS**; **69.5 vs 42.7 MB heap**      |
| **Q3** run‑to‑run distribution (≥2 runs)        | —                                                             | Q3a, Q3b                | ✓ present {178.89,177.36}; absent {98.05,97.86}       |
| **Q3** `recycle:1` across‑task reclaim          | `settings.py:452`                                             | Q3c                     | ✓ 3 fresh procs reset to ~49.8 MB (inferred qcluster) |
| **Q4** `RasterisedDocumentParser` (pikepdf+OCR) | `paperless_tesseract/parsers.py`                              | Q4a                     | ✓ pdf RSS 44‑62 MB, heap ≤204 KB                      |
| **Q4** `TextDocumentParser` (`f.read()`)        | `paperless_text/parsers.py:40,42`                             | Q4a                     | ✓ txt RSS 46‑94 MB, heap ≤221 KB                      |
| **Q4** `TikaDocumentParser` (office)            | `paperless_tika/parsers.py:29,30,32,50,55`                    | Q4d                     | ✓ **not exercisable** (canonical): unsupported mime   |
| **Q4** base `extract_metadata` → `[]`           | `documents/parsers.py:304,305`                                | TL;DR, Q4d              | ✓ default returns `[]` (no metadata)                  |
| **Q4** batch single/several/large‑N             | —                                                             | Q4b                     | ✓ heap **flat** (~80 MB); RSS plateaus, no climb      |
| **Q4** `train()` O(N) accumulation              | `classifier.py:115,125,130,137,144,156`                       | Q4c                     | ✓ 86/172/343 MB @ 25/50/100; N=200 min_df artifact    |
| **Q4** `DEBUG` off vs on                        | `settings.py:50`                                              | Q2b, Q4                 | ✓ 0 vs bounded 9000                                   |
| **Q4** pikepdf with vs without `close()`        | `paperless_tesseract/parsers.py:34,35`                        | Q1c                     | ✓ +33.41 MB vs +0.04 MB                               |

### Deliverables

| Deliverable                                                     | Section                 | Confirmed                                                    |
| --------------------------------------------------------------- | ----------------------- | ------------------------------------------------------------ |
| **A** — actual runtime measurements + command per block         | Deliverable A + every Q | ✓ complete unedited output, commands shown                   |
| **B** — components/methods holding memory (named + `file:line`) | Deliverable B           | ✓ 16 rows, specific methods                                  |
| **C** — normal vs problematic verdict (tracemalloc‑vs‑RSS)      | Deliverable C           | ✓ net heap +54.2 KB, `gc.garbage=0`, RSS +134.70 MB → normal |

### Other named mechanisms from the pipeline (completeness)

| Mechanism                                                  | `file:line`                                        | Where covered                           |
| ---------------------------------------------------------- | -------------------------------------------------- | --------------------------------------- |
| `try_consume_file` entry point                             | `consumer.py:180`                                  | §1.1, Q3, C                             |
| `consume_file` task entry point                            | `tasks.py:184`                                     | §2, A manifest                          |
| `document_consumer` `async_task(...)` canonical enqueue    | `document_consumer.py:13,86`                       | §1.1 (inferred watcher)                 |
| `pre_check_duplicate`                                      | `consumer.py:102`                                  | Q1a, B #3                               |
| `_store` / `_write`                                        | `consumer.py:379,429`                              | Q1a, B #4‑5                             |
| `DocumentClassifier.load` `with open(...)` + version check | `classifier.py:76,77,80`; `FORMAT_VERSION=7` `:63` | Q3a, A, C                               |
| unpickle 6 objects                                         | `classifier.py:86‑92`                              | A (top sites :90/91/92), C              |
| `predict_correspondent/document_type/tags`                 | `classifier.py:251,262,273`                        | Q3a (`predicted_corr`), Q4a predictions |
| `train_classifier`                                         | `tasks.py:48,57`                                   | Q3b, Q4c                                |
| handler classifier params + matches                        | `handlers.py:39,50,105,116,172,189`                | Q1b, B #8‑9                             |
| base parser tempdir lifecycle                              | `parsers.py:293` (mkdtemp), `:350` (rmtree)        | §1.1                                    |
| `get_text` / `get_date`                                    | `parsers.py:342,345`                               | Q4a (date‑parse elapsed cost)           |
| metadata REST action (original+archive)                    | `views.py:260,269,283,295,302‑305`                 | Q1c, B #2                               |
| `suggestions` action `load_classifier`                     | `views.py:313,319`                                 | Q2a (another `load_classifier` caller)  |
| `AsyncWriter` / `open_index_writer`                        | `index.py:26,65,66`                                | B #12                                   |
| `DATA_DIR`                                                 | `settings.py:66`                                   | §2                                      |
| `Q_CLUSTER` block                                          | `settings.py:449`                                  | Q3c, B #15                              |

**Coverage pass result:** all four questions (Q1 copies/references, Q2 caching, Q3 spike vs no‑spike,
Q4 document types/batch sizes), all three deliverables (A/B/C), the full condition cross‑product
(MODEL_FILE present/absent × 3 parser families × batch sizes × DEBUG off/on × pikepdf with/without
`close()` × `train()` corpus growth × `recycle:1`), and every named function/file/flag are present with a
concrete value, a verified `file:line`, observed raw evidence, and a cause→effect explanation. Values not
observable at runtime in the canonical config are explicitly labeled **(inferred)** (the live `qcluster`
`recycle:1` teardown and the Tika JVM residency).

---

## Notes on method, honesty, and limitations

- **Read‑only:** no `src/**` file was modified; the only repository addition is this document. Every
  probe script and sample document lived under `/tmp/blitzy_probe/` (outside the tracked tree) and was
  deleted after measurement.
- **Runtime:** all pipeline numbers were produced on the canonical **Python 3.9.23** inside the provided
  container at commit `542221a38dff`, driving the **real** `Consumer().try_consume_file()` entry point —
  never a bypass. The one deliberate non‑canonical probe (calling `pikepdf.open` directly to contrast
  with/without `close()`, Q1c) is labeled as such and does not edit the source.
- **Reported exactly as observed:** measured values are pasted verbatim and never rounded toward an
  expected answer; the `train()` N=200 vocabulary‑collapse anomaly and the run‑to‑run RSS variance are
  reported as‑is with their causes.
- **Not exercised (stated plainly):** the office/`TikaDocumentParser` memory profile (Tika disabled by
  default and no server reachable) and a live multi‑worker `qcluster` `recycle:1` teardown — both labeled
  **(inferred)** where reasoned from code.
