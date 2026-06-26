# Where the Memory Actually Goes: A Diagnostic Investigation into Paperless‑ngx Document Import & Metadata Handling

> **Scope.** This is a **read‑only diagnostic**. It explains *where* and *why* memory grows during document import and metadata processing in Paperless‑ngx, grounded in (a) line‑level code citations and (b) runtime measurements captured under the target runtime. **No repository source file was modified**; all profiling tooling was external and has been deleted. The only artifact added to the repository is this document.
>
> **Target runtime for all headline numbers.** Python **3.9.23**, Django **4.0.4**, `DEBUG=False`, SQLite backend, inside the pinned image `paperless-ngx-mem:py39-ready` (derived from the project's `python:3.9-slim-bullseye` base — `Dockerfile:L18`). Every measured figure in this report was captured under that interpreter unless a number is *explicitly labeled otherwise*. Dependency versions in effect: scikit‑learn 1.0.2, numpy 1.22.3, scipy 1.8.0, whoosh 2.7.4, joblib 1.1.0, django‑q 1.3.9, dateparser 1.1.1 (all matching `requirements.txt` / `Pipfile.lock`).
>
> **Why fidelity matters.** `pymalloc` arena behavior and garbage‑collector thresholds differ across interpreter versions, so the "memory is not released" symptom must be measured on 3.9, not on a host 3.12 shell.

---

## Table of contents

1. [Executive summary & headline verdict](#1-executive-summary--headline-verdict)
2. [Per‑question answers (Q1–Q5)](#2-perquestion-answers)
   - [Q1 — Root cause of the spikes](#q1--root-cause-of-the-spikes)
   - [Q2 — Copies vs. retained references](#q2--copies-vs-retained-references)
   - [Q3 — Unexpected caching](#q3--unexpected-caching)
   - [Q4 — Spike vs. no‑spike differential](#q4--spike-vs-nospike-differential)
   - [Q5 — Sensitivity to document type & batch size](#q5--sensitivity-to-document-type--batch-size)
3. [Normal vs. problematic: CPython memory behavior](#3-normal-vs-problematic-cpython-memory-behavior)
4. [Measurement methodology](#4-measurement-methodology)
5. [Memory‑hotspot table (verified citations)](#5-memoryhotspot-table-verified-citations)
6. [Recommendations (suggestions only — NOT applied)](#6-recommendations-suggestions-only--not-applied)
7. [Appendix: harness, raw results, environment facts](#7-appendix-harness-raw-results-environment-facts)

---

## 1. Executive summary & headline verdict

The user observes three things: (1) memory **spikes disproportionately** during import — especially while metadata is processed — even for collections of *small* text documents; (2) the behavior is **inconsistent across runs**; and (3) memory is **not returned to the OS in a timely manner**. All three are explained below, and each maps to concrete code and concrete numbers.

**Headline verdict.**

- **The "spike" is not caused by any single document being large.** It is dominated by code paths whose cost scales with the **batch / source size** or with the **corpus**, not with the size of an individual file:
  - **Manifest‑driven import** parses the *entire* export into one dict held for the whole command (`document_importer.py:L72‑L73`) **and** re‑deserializes the same file a second time via `loaddata` (`document_importer.py:L87`). Measured: a 300‑document export produced a **23.59 MiB** `manifest.json`; `json.load` retained **+23.9 MiB** (proven by `gc.get_referrers` to be held by `self.manifest`), and `loaddata` re‑deserialized at a traced peak of **47.9 MiB** — two independent in‑memory copies of the same data.
  - **QuerySet result‑cache materialization of the `content` `TextField`.** Loops such as `index_reindex` iterate `Document.objects.all()` without `.iterator()`/`.only()`/`.defer()` (`tasks.py:L38‑L45`), so every row — *including its full text* — is cached for the loop's lifetime. Measured: materializing 300 rows held **+25.3 MiB** RSS / **23.5 MiB** of content; the same loop with `.defer("content")` cost **0.5 MiB** and `.iterator()` retained **0.0 MiB**.
  - **Native (C‑extension) allocations that profilers under‑report.** A full `index_reindex` grew RSS by **+331 MiB** while `tracemalloc` saw only **166.7 MiB** — the Whoosh `AsyncWriter` postings buffer lives in native memory. The scikit‑learn classifier behaves similarly.

- **"Especially during metadata processing"** is explained by three metadata‑adjacent costs: the **classifier** is **unpickled on every consume** and is **never cached** (`consumer.py:L292`, `classifier.py:L76‑L94`) — measured at **+263 MiB** on first load and **+330 MiB again** on the very next load (identical, because nothing is cached); **date extraction** runs a regex over the *entire* document text (`parsers.py:L261`) — **+151.8 MiB** for a 2 MiB document; and **rule matching** builds a punctuation‑stripped, lower‑cased **full‑content copy per fuzzy rule** (`matching.py:L131,L134`).

- **"Inconsistent across runs"** is the signature of **first‑vs‑subsequent execution inside a long‑lived `django‑q` worker** plus **allocator fragmentation**. One‑time lazy loads (dateparser locale/regex init, the first classifier unpickle) inflate the *first* task handled by a fresh worker; subsequent tasks differ. Measured: dateparser's first parse cost a one‑time init and its second parse cost **0.0 MiB**; classifier‑absent runs cost **0.0 MiB** versus classifier‑present runs costing hundreds of MiB.

- **"Memory not released in a timely manner"** is **mostly normal allocator behavior, not a leak** — but layered on top of a few **genuine over‑retentions**. CPython's `pymalloc` only returns a 256 KiB arena to the OS when *every* pool in it is empty, and glibc's `malloc` keeps freed large blocks in per‑arena free lists. Measured: after allocating 300 000 distinct small objects and then **deleting all of them**, RSS fell only from 164.7 MiB to 122.4 MiB; keeping just **120 scattered survivors (~0.007 MiB live)** pinned **170.4 MiB** resident. That is expected fragmentation. The *genuine* problems — `self.manifest` and the QuerySet result cache held across long loops — are distinct live references the program holds longer than necessary.

**One‑sentence answer to "where is the memory actually going":** into (i) **two whole‑export copies** during manifest import, (ii) the **`content` text of every row** cached by un‑streamed QuerySet loops, (iii) **repeated, un‑cached classifier unpickling** and per‑document text copies during metadata processing, and (iv) **native Whoosh/scikit‑learn buffers** — all parked in **long‑lived worker arenas** that the allocator legitimately does not hand back to the OS right away.

The amplifier behind almost every item is the data model itself: <code>content = models.TextField(...)</code> (`models.py:L117‑L124`) means *every* loaded `Document` row inherently carries the full raw text.

---

## 2. Per‑question answers

> Every number below is a Python 3.9.23 / `DEBUG=False` measurement from the harness described in §4 (raw output in §7). Each scenario is labeled `S0…S6`.

### Q1 — Root cause of the spikes

**Which code paths / methods / objects drive the disproportionate memory growth during import and metadata processing?**

There is no single cause; there are **four dominant terms**, ranked here by measured impact. Critically, the three largest scale with **batch/source/corpus size**, which is exactly why a batch of *small* documents can still spike.

**1. The manifest‑import path holds two whole‑export copies (the strongest match for "spikes during import").**
`document_importer.py` reads the *entire* `manifest.json` into a dict and **keeps it on the instance for the whole command**:

```text
document_importer.py:L72-L73   with open(manifest_path) as f:
                                   self.manifest = json.load(f)
document_importer.py:L87        call_command("loaddata", manifest_path)   # re-reads & re-deserializes
document_importer.py:L137-L139  manifest_documents = list(filter(
                                   lambda r: r["model"] == "documents.document", self.manifest))
```

The manifest's *origin* explains why it scales with the whole library rather than one document: the exporter serializes **all** documents *with every field, including `content`* (`document_exporter.py:L127‑L130`).

- **Measured (S2, 300 docs):** `manifest.json` on disk = **23.59 MiB**. `self.manifest = json.load(f)` retained **+23.9 MiB** (traced peak 47.5 MiB), and `gc.get_referrers` proved the dict is *"held by → instance `__dict__` (`self.manifest`)"*. The subsequent `loaddata` re‑deserialized the same file at a traced peak of **47.9 MiB** — a **second** in‑memory copy that coexists with the still‑retained `self.manifest`. Peak import memory therefore tracks **total export size**, not per‑document size.

**2. QuerySet result‑cache materialization of `content`.**
`index_reindex` (and several batch commands) iterate a QuerySet without `.iterator()`:

```text
tasks.py:L39   documents = Document.objects.all()
tasks.py:L44   for document in tqdm.tqdm(documents, ...):   # tqdm calls len() -> full evaluation
```

`tqdm` calls `len()` on the QuerySet, which forces full evaluation; Django then **caches every row in the QuerySet's `_result_cache`** for the loop's entire duration, and each row drags its `content` `TextField` (`models.py:L117‑L124`) along.

- **Measured (S3, 300 docs):** `list(Document.objects.all())` held **+25.3 MiB** RSS, of which **23.5 MiB** is document text. Replacing it with `.defer("content")` dropped the traced peak from **23.9 MiB → 0.5 MiB**, and `.iterator()` retained **0.0 MiB**. The growth is therefore *caused by caching the `content` column across the loop*, confirmed by the line attribution (`django/db/utils.py:98`, the row‑fetch site).

**3. Repeated, un‑cached classifier unpickling during metadata processing.**
Every consume calls `load_classifier()` (`consumer.py:L292‑L293`), which `pickle.load`s the vectorizer vocabulary and the neural‑net weight matrices each time (`classifier.py:L76‑L94`):

```text
classifier.py:L87   self.data_vectorizer        = pickle.load(f)   # CountVectorizer vocabulary dict
classifier.py:L91   self.correspondent_classifier = pickle.load(f) # MLPClassifier (NumPy float64 weights)
```

- **Measured (S6):** first `load_classifier()` = **+263 MiB** (attributed to `classifier.py:87` +305 MiB vectorizer and `classifier.py:91` +114 MiB MLP weights); the **very next** `load_classifier()` cost **+330 MiB again** — *identical, because the model is never cached in worker memory*. (Absolute magnitude is inflated by the synthetic corpus — see the honesty note in §2/Q3 and §4 — but the **recurring, un‑cached shape is faithful**.)

**4. Date extraction scans the entire document text.**
`parse_date` runs a regex `finditer` over the whole content (`parsers.py:L261`), and is invoked from the consume path (`consumer.py:L275`).

- **Measured (S1):** `parse_date` over a 2 MiB document grew RSS by **+151.8 MiB** (traced peak 25.7 MiB; the remainder is the native `regex` engine + first‑use locale init). This is per‑call work that scales with text length.

**Rationale.** A run that imports a manifest pays terms 1 (×2 copies), and any reindex/bulk pass pays term 2; metadata processing during consume pays terms 3 and 4. None of these are bounded by a single small file — they are bounded by *how many* documents and *how much total text* are in play, which is precisely why "mostly small text documents" still spike.

### Q2 — Copies vs. retained references

**Does metadata handling make unnecessary in‑memory copies of document content, or hold references to large objects longer than necessary?**

**Both happen, and they are different mechanisms** — the investigation deliberately separated them with `gc.get_referrers`.

**Unnecessary copies (transient, O(document size)):**

- **OCR/text post‑processing** chains three `re.sub` calls plus a `strip().replace(...)` (`paperless_tesseract/parsers.py:L330‑L341`), each producing a **new** full‑text string while the previous one is still alive:
  - **Measured (S4):** for a 4 MiB input, `post_process_text` peaked at **36.50 MiB** of traced memory (≈ 9× the input — several simultaneous copies). For 1 MiB → 9.30 MiB; for 16 KiB → 0.15 MiB. Clean `O(document size)`.
- **Fuzzy rule matching** builds a punctuation‑stripped *and* a lower‑cased copy of the **entire** `document.content` **per rule** (`matching.py:L131,L134`):
  ```text
  matching.py:L63    document_content = document.content
  matching.py:L131   text = re.sub(r"[^\w\s]", "", document_content)  # full-content copy #1 (per rule)
  matching.py:L134   text = text.lower()                              # full-content copy #2 (per rule)
  ```
  *(Note: the punctuation‑strip/lower at `L130`/`L133` operate on the short `match` string; the full‑content copies are at `L131`/`L134`.)* With *N* fuzzy correspondent/type/tag rules, that is up to `2N` transient full‑content copies. **Measured (S4):** 10 fuzzy rules over an 80 KiB document showed each copy ≈ **0.078 MiB**; the cost is `O(document size × number of fuzzy rules)`.
- **File I/O copies** read whole files into RAM: md5 dedup `hashlib.md5(f.read())` (`consumer.py:L102‑L104`) and the non‑streaming `_write` `write_file.write(read_file.read())` (`consumer.py:L429‑L432`). **Measured (S1):** a 2 MiB file held **2.0 MiB** at once for `f.read()`, versus **0.1 MiB** for a chunked `hashlib` update — a 20× transient reduction available for free.

**Retained references (long‑lived — the genuine over‑retention):**

- `self.manifest` is held for the **command's entire lifetime** (`document_importer.py:L72‑L73`). **Measured (S2):** `gc.get_referrers(cmd.manifest)` reported the holder as the **instance `__dict__`** — i.e. it is a deliberate, long‑lived reference, not allocator residue. It is still alive while `loaddata` builds its own copy.
- The QuerySet `_result_cache` is held across the *whole* reindex loop (`tasks.py:L39‑L45`) and across **two** loops in `bulk_update_documents` (`tasks.py:L275` and `L279` iterate the *same* queryset). **Measured (S3):** 23.5 MiB of content stays resident for the loop's duration; `.iterator()` reduces that retention to ~0.

**Rationale.** "Copies" are transient and freed quickly (their cost is a *peak*, governed by document size and rule count). "Retained references" stay resident for as long as the owning object lives (their cost is *sustained*, governed by batch/source size). The user's "metadata processing" spike is a blend: transient copies (post‑process, fuzzy) raise the *peak*, while retained references (`self.manifest`, result cache) raise the *baseline* that the peak sits on.

### Q3 — Unexpected caching

**Do model caches, Django QuerySet result caches, Whoosh index buffers, or locale data accumulate across documents/tasks?**

| Cache | Accumulates? | Evidence | Measured |
|---|---|---|---|
| **Django QuerySet result cache** | **Yes — per loop** | `Document.objects.all()` iterated without `.iterator()` (`tasks.py:L39‑L45`, `sanity_checker.py:L61`, `document_archiver.py:L129`, `document_retagger.py:L74`) caches all rows incl. `content`. | S3a: +25.3 MiB / 23.5 MiB content for 300 rows; `.iterator()` → 0.0 MiB. |
| **Whoosh `AsyncWriter` RAM buffer** | **Yes — until commit** | `AsyncWriter` buffers postings in RAM; `update_document` indexes the full `content` (`index.py:L64‑L66`, `L87‑L93`). Constructed with **no** `limitmb`/`procs` override anywhere. | S3e: `index_reindex` over 300 docs grew RSS **+331 MiB** while `tracemalloc` saw only **166.7 MiB** — the buffer is largely **native** memory. |
| **scikit‑learn model** | **No caching — re‑loaded every time** (a *different* problem) | `load_classifier()` is called per consume (`consumer.py:L292`) and per suggestions request (`views.py:L319`); it always `pickle.load`s from disk (`classifier.py:L76‑L94`). There is **no** in‑memory model cache. | S6c first load +263 MiB; **S6d second load +330 MiB — identical**, confirming nothing is cached. |
| **dateparser locale / `regex`** | **One‑time lazy load, then stable** | First date parse imports `regex` core modules and locale data; subsequent parses reuse them. | S6e first parse: one‑time init (traced 1.1 MiB of `regex` core); **S6f second parse: 0.0 MiB**. |
| **Django SQL query cache (`connection.queries`)** | **Only if `DEBUG=True`** | When `DEBUG` is on, Django stores every executed query, growing without bound. `settings.py:L50` defaults it off (`PAPERLESS_DEBUG=NO`). | Confirmed `DEBUG=False` at measurement time so this is excluded; flagged because enabling it would dwarf everything else. |

**Honesty note on the classifier absolute numbers.** The harness trained the model on a synthetic corpus of *random* tokens, which produces a pathologically large bigram vocabulary; with `min_df=0.01` pruning over natural‑language documents the real vocabulary (and thus the +263/+330 MiB figures) would be **substantially smaller**. What is **not** corpus‑dependent and **is** faithfully demonstrated: (a) the model is re‑unpickled on *every* call and never cached (S6c == S6d), and (b) a meaningful fraction of its weight memory is **native NumPy** that RSS sees but `tracemalloc` under‑reports.

**Rationale.** Two of these are *accumulation* caches (QuerySet result cache, Whoosh buffer) that grow with the work in front of them; one (classifier) is the *opposite* pathology — a large object that is **never** cached and is therefore **rebuilt repeatedly**, churning the heap; and one (dateparser) is a benign one‑time load that nonetheless contributes to "first run looks worse."

### Q4 — Spike vs. no‑spike differential

**What concretely differs between runs that spike and those that do not?**

Three concrete differentials, each measured:

1. **First vs. subsequent task in a long‑lived `django‑q` worker.** The first task a fresh worker handles pays **one‑time lazy loads** that later tasks do not: the `regex`/dateparser locale init and the first classifier unpickle. **Measured:** dateparser first parse incurs init, second parse = **0.0 MiB** (S6e→S6f). Because `django‑q==1.3.9` workers are **long‑lived processes**, this one‑time cost is amortized — but the memory it parks in arenas stays resident, so the worker's RSS *steps up* on its first heavy task and then plateaus, which reads as "inconsistent."
2. **Classifier‑present vs. classifier‑absent.** `load_classifier()` returns `None` immediately when no model file exists (`classifier.py:L31‑L36`). **Measured:** the no‑model path cost **0.0 MiB** (S6a) versus **+263 MiB** when a model is present (S6c). A library that has never trained a classifier simply never pays this term — a large, binary spike/no‑spike switch.
3. **Lightweight text parser vs. OCR parser.** A plain‑text document skips OCR entirely; a scanned PDF runs the tesseract path and the `post_process_text` copy chain (`paperless_tesseract/parsers.py:L330‑L341`) plus native OCR/pikepdf buffers. **Measured (S4):** the post‑process copy chain alone scales to **36.5 MiB** for a 4 MiB text layer, and OCR adds native allocations on top that `tracemalloc` cannot see.

A fourth, **non‑deterministic** differential is **allocator fragmentation**: identical work can leave different RSS depending on allocation order. **Measured (S0):** keeping 120 *scattered* survivors (0.007 MiB live) pinned **170.4 MiB**; had the survivors been contiguous, far more arenas could have been released. Run‑to‑run ordering differences therefore produce run‑to‑run RSS differences with no change in live data.

**Rationale.** "Spike vs. no‑spike" is rarely about the document — it is about **process history** (first vs. Nth task), **configuration** (is a classifier present? is OCR triggered?), and **allocator state** (fragmentation). This fully accounts for the user's "inconsistent across runs."

### Q5 — Sensitivity to document type & batch size

**How does the memory profile change for plain text vs. OCR/PDF vs. office docs, and for single upload vs. bulk vs. manifest‑driven import?**

Each growth term classified as **O(document size)**, **O(batch/source size)**, or **constant**:

| Term | Code | Class | Evidence |
|---|---|---|---|
| md5 `f.read()` | `consumer.py:L102‑L104` | **O(document size)** | S1a: 2 MiB file → 2.0 MiB held; chunked → 0.1 MiB |
| `_write` full read | `consumer.py:L429‑L432` | **O(document size)** | S1b: 2.0 MiB held |
| `parse_date` regex scan | `parsers.py:L261` | **O(document size)** | S1c: 2 MiB → +151.8 MiB RSS |
| `post_process_text` chain | `paperless_tesseract/parsers.py:L330‑L341` | **O(document size)** | S4: 16 KiB→0.15, 1 MiB→9.30, 4 MiB→36.50 MiB |
| Fuzzy match copies | `matching.py:L131,L134` | **O(document size × #fuzzy rules)** | S4: 0.078 MiB per copy per rule |
| Whoosh `update_document(content)` | `index.py:L87‑L93` | **O(document size)**, buffered | part of S3e |
| `self.manifest` (`json.load`) | `document_importer.py:L72‑L73` | **O(batch/source size)** | S2b: +23.9 MiB for 300 docs |
| `loaddata` re‑deserialize | `document_importer.py:L87` | **O(batch/source size)** | S2d: traced peak 47.9 MiB |
| `list(filter(...))` extra list | `document_importer.py:L137‑L139` | **O(batch/source size)** | S2c: 300‑element list of references |
| QuerySet result cache | `tasks.py:L39‑L45` | **O(batch/source size)** | S3a: +25.3 MiB / 23.5 MiB for 300 rows |
| `AsyncWriter` buffer (reindex) | `tasks.py:L43` + `index.py:L66` | **O(batch/source size)**, native | S3e: +331 MiB RSS for 300 docs |
| Classifier training accumulation | `classifier.py:L115‑L130` | **O(corpus size)** | S6b: +1661 MiB (synthetic upper bound) |
| Classifier unpickle | `classifier.py:L76‑L94` | **constant per call, but recurring** | S6c == S6d (~+263/+330 MiB each) |
| dateparser locale/regex init | (dateparser) | **constant, one‑time** | S6e→S6f (init, then 0.0 MiB) |

**By document type:**
- **Plain text:** pays only the `O(document size)` per‑file terms (read, date scan, indexing). Cheapest path; for small text docs the *per‑file* peak is small (S1a/b ≈ 2 MiB for a 2 MiB file).
- **OCR/PDF:** adds the `post_process_text` copy chain (S4, up to ~9× the text‑layer size) **and** native tesseract/pikepdf/Ghostscript allocations that RSS sees but `tracemalloc` does not.
- **Office:** behaves like the large‑text case (its extracted text drives the same `O(document size)` terms); modeled by the 4 MiB "office‑like" input in S4 (36.5 MiB post‑process peak).

**By batch shape:**
- **Single upload / consume:** dominated by per‑document `O(document size)` terms + the (recurring) classifier load. Bounded per file.
- **Bulk (`index_reindex`, `bulk_update_documents`, archiver, retagger, sanity check):** dominated by the **`O(batch/source size)`** QuerySet result cache + native index buffer. This is where "many small docs" become a large number.
- **Manifest‑driven import:** the worst case — **two** `O(batch/source size)` whole‑export copies (`self.manifest` + `loaddata`) that coexist, so peak ≈ 2× the export size regardless of individual document sizes.

**Rationale.** The user's report that *small* documents still spike is fully consistent with this classification: the dominant spike terms are **batch/source/corpus**‑sized, so the spike grows with **count and total text**, not with any single file. Document *type* mainly changes the per‑file constant (OCR adds copies + native memory); batch *shape* selects whether the `O(batch)` terms fire at all (manifest import fires the largest ones).

---

## 3. Normal vs. problematic: CPython memory behavior

A central requirement of this investigation is to **separate normal interpreter/allocator behavior from a genuine problem**. The user's "memory is not released in a timely manner" is, in large part, **expected** — but it sits on top of a few real over‑retentions. Here is the full mental model, validated with measurements.

### 3.1 Reference counting and the cyclic GC

Most objects are freed **immediately** when their reference count hits zero; CPython's cyclic garbage collector exists only to reclaim **reference cycles**. The crucial point: **freeing a Python object returns its memory to the *allocator*, not necessarily to the *operating system*.** So `del big_obj; gc.collect()` can leave process RSS unchanged and still be perfectly healthy.

### 3.2 `pymalloc` arenas — for small objects (≤ 512 bytes)

CPython's small‑object allocator, `pymalloc`, requests memory from the OS in **256 KiB arenas**, subdivides each arena into 4 KiB **pools**, and pools into fixed‑size **blocks**. An arena is returned to the OS **only when *every* pool inside it is empty**. Therefore:

- A **single surviving small object** can keep an entire **256 KiB arena** resident.
- **Fragmentation** — survivors scattered across many arenas — keeps many arenas resident even when very little is actually live.

**Measured (S0, Python 3.9):**

| Step | Live data | RSS |
|---|---|---|
| Allocate 300 000 distinct 64‑byte objects | ~30 MiB traced | 164.7 MiB |
| `del` **all** + `gc.collect()` | ~0 | 122.4 MiB (fell only ~42 MiB) |
| Re‑allocate, then keep **120 scattered survivors** | **0.007 MiB** | **170.4 MiB** |

The last row is the textbook result: **7 KiB of live data pins 170 MiB of RSS** purely through arena fragmentation. **This is normal and is not a leak.**

### 3.3 System (glibc) `malloc` — for large objects (> 512 bytes)

Objects larger than 512 bytes — and Paperless‑ngx's `content` strings are typically KiB–MiB — bypass `pymalloc` and go to the **system allocator (glibc `malloc`** on this Debian‑based image). glibc *also* retains freed memory: it keeps freed chunks in **per‑arena free lists** and does not promptly `munmap` them back to the OS. The number of glibc arenas defaults to roughly `8 × CPU‑cores` and is tunable via `MALLOC_ARENA_MAX`. A long‑lived allocation made *between* transient ones can **pin an arena**, producing the same "RSS won't drop" effect for large objects that `pymalloc` produces for small ones.

**Measured (S0c, Python 3.9):** allocating 6 000 distinct 4 KiB strings tracked cleanly in `tracemalloc` (peak 47.2 MiB ≈ the live payload), and after freeing them all, RSS remained ~100 MiB resident — glibc holding the freed heap. Because document `content` is large, **both** the `pymalloc` (small‑object) story **and** the glibc (large‑object) story apply to Paperless‑ngx; the dominant objects here (full text) are large, so glibc retention is the more relevant of the two for the `content` strings, while `pymalloc` fragmentation dominates for the millions of small objects (dict entries, tokens) created during serialization/vectorization.

### 3.4 The `django‑q` amplifier

`django‑q==1.3.9` workers are **long‑lived processes**. Every term above that parks memory in an arena — a retained reference *or* mere allocator residue — **persists for the life of the worker** and accumulates across the successive documents/tasks that the same worker handles. This is the mechanism that makes the per‑task costs *look like* a cross‑document leak even when each individual task is well‑behaved.

### 3.5 Genuine over‑retention (the real problems, distinct from arena residue)

Two things in this codebase are **not** allocator artifacts — they are live references held longer than necessary, and `gc.get_referrers` proves it:

- `self.manifest` kept alive for the whole import command (`document_importer.py:L72‑L73`) — referrer is the **instance `__dict__`** (S2b).
- The QuerySet `_result_cache` held across an entire long loop (`tasks.py:L39‑L45`) — eliminated by `.iterator()` (S3a vs. S3b).

These would not be reclaimed by any allocator tuning, because the program is **still pointing at them**.

### 3.6 Why `sys.getsizeof` is the wrong tool here

`sys.getsizeof(obj)` returns only the **shallow** size of an object — the container itself, not the objects it references. A `list` of 300 documents reports a few KiB via `getsizeof` even though it transitively retains tens of MiB of strings. Deep retention must therefore be measured with **`tracemalloc` snapshot diffs** (which attribute the *allocations* themselves) and **`gc.get_referrers`** (which proves *who* holds an object) — both used throughout this report.

**Bottom line for the user's "not released in a timely manner":** the *majority* of the lingering RSS is **normal** `pymalloc`/glibc arena retention in long‑lived workers (§3.2–§3.4) and should not be chased as a leak; the *actionable* part is the small set of **genuine over‑retentions** in §3.5 (and the recurring un‑cached classifier in Q3), which are live references the code can choose to drop sooner.

---

## 4. Measurement methodology

### 4.1 Where and how

All headline numbers were captured **inside the pinned Docker image** `paperless-ngx-mem:py39-ready` (Python **3.9.23**, the project's `python:3.9-slim-bullseye` base — `Dockerfile:L18`) with the **full dependency set and system binaries** (tesseract, qpdf, Ghostscript, poppler‑utils, pikepdf, libatlas for NumPy). Those native libraries are *exactly* why a Python‑only profiler is insufficient (see §4.3). Django booted with **`DEBUG=False`** (verified at runtime; `settings.py:L50` defaults `PAPERLESS_DEBUG=NO`) on the SQLite backend, and the harness asserted `settings.DEBUG is False` before measuring so the `connection.queries` cache could never inflate results.

The harness lived entirely **outside** the repository (`/tmp/profiling/`, mounted/copied into the container) and called the **real** Paperless‑ngx functions (`parse_date`, `post_process_text`, `matching.matches`, `serializers.serialize`, `load_classifier`, `index_reindex`, `DocumentClassifier.train`) so that `tracemalloc` line attribution resolves to the actual `/app/src/...` source lines cited in this report.

### 4.2 Two instruments, always paired

| Instrument | Sees | Blind to | Used for |
|---|---|---|---|
| **`tracemalloc`** (stdlib) | Python‑heap allocations, per file:line | C‑extension / native allocations (NumPy, pikepdf, Whoosh native buffers) | Attributing growth to a specific `path:line` via `snapshot2.compare_to(snapshot1, "lineno")`; traced peak via `get_traced_memory()` after `reset_peak()` |
| **Process RSS** (`psutil`, else `/proc/self/status` `VmRSS`, else `resource.getrusage().ru_maxrss`) | **All** memory incl. native | (nothing — but it is process‑global, not per‑line) | Capturing the *true* footprint, including native allocations that `tracemalloc` cannot see |

Pairing them is mandatory: where the two diverge, the gap **is** the native allocation. The clearest example is **S3e** (`index_reindex`): RSS **+331 MiB** vs. `tracemalloc` **166.7 MiB** — the ~164 MiB gap is the Whoosh `AsyncWriter`'s native postings buffer. The classifier (S6c) shows the same effect for scikit‑learn/NumPy weight matrices.

> **`tracemalloc` limitation, stated explicitly.** `tracemalloc` records only allocations made through CPython's allocators. C libraries that allocate with their own `malloc` (Whoosh's C parts, pikepdf, parts of NumPy) are invisible to it. Any term that is "RSS‑heavy but `tracemalloc`‑light" in this report is therefore a **native** allocation, and RSS is the authoritative figure for it.

### 4.3 Distinguishing retention from allocator residue

After each stage the harness ran `gc.collect()` and then sampled RSS, and used **`gc.get_referrers(obj)`** to prove *which* live reference keeps an object alive. This is how `self.manifest` was shown to be held by the importer instance's `__dict__` (S2b) rather than merely parked in an arena — the distinction at the heart of §3.5.

### 4.4 The `Measure` context manager (per‑window protocol)

Each measured window: `gc.collect()` → `tracemalloc.clear_traces()` → `tracemalloc.reset_peak()` → snapshot **before** → *run stage* → snapshot **after** → record traced current/peak and RSS, then `gc.collect()` and re‑sample RSS as `after_gc`. The before/after snapshots are diffed with `compare_to(..., "lineno")` to rank the responsible source lines.

### 4.5 Scenarios

`S0` allocator illustration (pymalloc + glibc); `S1` single‑consume primitives on a 2 MiB plain‑text doc; `S2` manifest export→import (the `self.manifest`/`loaddata`/`list(filter)` terms); `S3` bulk reindex + QuerySet caching variants (`all()` vs `.iterator()` vs `.defer` vs `.only`) + real `index_reindex`; `S4` type sensitivity (16 KiB / 1 MiB / 4 MiB) + fuzzy matching copies; `S6` differential (classifier absent/present, first vs. second load, dateparser first vs. second parse). Raw output for all of them is in §7.

### 4.6 Honest caveats

- **Synthetic corpus inflates the classifier numbers.** The seeded documents use *random* tokens, which makes the bigram vocabulary far larger than natural language would after `min_df=0.01` pruning. The classifier **absolute** figures (train +1661 MiB; load +263/+330 MiB) are therefore an **upper bound**; the **qualitative** findings (accumulate‑then‑fit; un‑cached repeated unpickle; native weights) are faithful.
- **Negative window deltas** appear for a few stages (e.g., S1d, S0c, S6e) because memory freed by a *previous* window's `gc.collect()` was handed back during the current window. For those stages the **traced peak** is the meaningful figure (e.g., S1d post‑process peak 18.6 MiB on a 2 MiB input), not the RSS delta.
- **SQLite, not PostgreSQL.** The DB backend does not change the Python‑side memory behavior under study (QuerySet caching, serialization, content retention); it only changes the driver. Findings about row/`content` materialization are backend‑independent.

---

## 5. Memory‑hotspot table (verified citations)

Every line number below was confirmed line‑by‑line against the source at HEAD `542221a38dff`. All files are under `src/` and were analyzed **read‑only**.

| # | Component | File | Lines | Verified behavior | Measured (Py 3.9) |
|---|---|---|---|---|---|
| 1 | Consume: md5 dedup | `documents/consumer.py` | **L102–L104** | `with open(self.path,"rb") as f: checksum = hashlib.md5(f.read()).hexdigest()` — whole file read into RAM. | S1a: 2.0 MiB held (chunked: 0.1 MiB) |
| 2 | Consume: text held & passed | `documents/consumer.py` | **L271, L275** | `text = document_parser.get_text()`; `date = parse_date(self.filename, text)`. | feeds S1c |
| 3 | Consume: classifier per‑doc | `documents/consumer.py` | **L292–L293** | `classifier = load_classifier()` invoked **per consumed document** (not cached). | S6c/S6d ~+263/+330 MiB each |
| 4 | Consume: non‑streaming write | `documents/consumer.py` | **L429–L432** | `_write` → `write_file.write(read_file.read())` — entire source in RAM, no chunking. | S1b: 2.0 MiB held |
| 5 | Import: retained dict | `documents/management/commands/document_importer.py` | **L52–L55, L72–L73** | `self.manifest = json.load(f)` — entire manifest parsed and **retained on the instance for the command lifetime**. | S2b: +23.9 MiB retained; referrer = instance `__dict__` |
| 6 | Import: re‑deserialization | `…/document_importer.py` | **L87** | `call_command("loaddata", manifest_path)` — second in‑memory copy of all records. | S2d: traced peak 47.9 MiB |
| 7 | Import: extra list | `…/document_importer.py` | **L137–L139** | `manifest_documents = list(filter(lambda r: r["model"]=="documents.document", self.manifest))`. | S2c: 300‑element list of refs |
| 8 | Export: manifest origin | `…/document_exporter.py` | **L116–L130** | Serializes correspondents/tags/types and **all documents incl. `content`** (`L127` `Document.objects.order_by("id")`, `L128` full `document_map`, `L129` serialize all). | S2a: +64.5 MiB; manifest 23.59 MiB on disk |
| 9 | Reindex: QuerySet materialization | `documents/tasks.py` | **L38–L45** | `Document.objects.all()` iterated under `tqdm` (which calls `len()` → full eval); `_result_cache` holds every row incl. `content`. | S3a: +25.3 MiB / 23.5 MiB content |
| 10 | Bulk update: double iteration | `documents/tasks.py` | **L270–L280** | Same queryset iterated twice (`L275`, `L279`); result cache held across both loops. | O(batch) (same mechanism as #9) |
| 11 | Classifier unpickle | `documents/classifier.py` | **L30–L57** (`load_classifier`), **L76–L94** (`load`) | Sequential `pickle.load` of schema, hash, `data_vectorizer` (`L87`), binarizer, and three `MLPClassifier`s (`L90`‑`L92`) whose `coefs_`/`intercepts_` are **native NumPy float64** matrices. | S6c: `classifier.py:87` +305 MiB, `:91` +114 MiB |
| 12 | Classifier train accumulation | `documents/classifier.py` | **L115–L130** | `data = list()` then `data.append(preprocess_content(doc.content))` for every doc before `fit_transform` — O(corpus). | S6b: +1661 MiB (synthetic upper bound) |
| 13 | Matching: per‑rule scans | `documents/matching.py` | **L21–L57** | `match_correspondents`/`_document_types`/`_tags` each load `*.objects.all()` and scan `document.content` per object. | part of S4 |
| 14 | Matching: fuzzy copies | `documents/matching.py` | **L60–L135** (copies at **L131, L134**) | Fuzzy branch builds a punctuation‑stripped (`L131`) and lower‑cased (`L134`) **full‑content copy per rule**. *(L130/L133 act on the short match string.)* | S4: 0.078 MiB per copy per rule |
| 15 | Whoosh writer buffer | `documents/index.py` | **L64–L66** (`AsyncWriter`), **L87–L93** (`update_document`) | `AsyncWriter` buffers postings in RAM until commit; indexes full `content=doc.content` (`L93`). No `limitmb`/`procs` set anywhere. | S3e: +331 MiB RSS vs 166.7 MiB traced |
| 16 | Data model amplifier | `documents/models.py` | **L117–L124** | `content = models.TextField(...)` — every loaded row carries full text. | underlies #5,#8,#9,#15 |
| 17 | Date parse over full text | `documents/parsers.py` | **L212–L274** (`finditer` at **L261**, filename at **L247**) | `for m in re.finditer(DATE_REGEX, text)` scans the **entire** content. | S1c: +151.8 MiB on 2 MiB doc |
| 18 | OCR post‑process copy chain | `paperless_tesseract/parsers.py` | **L330–L341** | Three chained `re.sub` + `strip().replace("\0"," ")` — multiple transient full‑text copies alive at once. | S4: 4 MiB→36.5 MiB peak |
| 19 | OCR sidecar read | `paperless_tesseract/parsers.py` | **L99–L108** | `with open(sidecar_file,"r") as f: text = f.read()` — whole sidecar into RAM. | O(document size) |
| 20 | Suggestions endpoint | `documents/views.py` | **L312–L320** | `suggestions()` calls `load_classifier()` (`L319`) **per HTTP request**. | same cost as S6c per request |
| 21 | Metadata serializer | `documents/serialisers.py` | `DocumentSerializer` **L201**, `content` in `fields` **L227** | REST list/detail responses serialize the full `content` field. | corroborates Q2/Q3 |
| 22 | Corroborating materialization | `documents/sanity_checker.py` | **L61** | `for doc in tqdm(Document.objects.all(), ...)` — same non‑`iterator()` pattern. | same as #9 |
| 23 | Corroborating materialization | `…/document_archiver.py` | **L129** | `documents = Document.objects.all()` materialized. | same as #9 |
| 24 | Corroborating materialization | `…/document_retagger.py` | **L74** | `queryset = Document.objects.all()` materialized. | same as #9 |

**Design observation (confirmed by grep):** there is **no** use of `.iterator()`, `.only()`, or `.defer()` anywhere in `src/documents/` or `src/paperless_tesseract/`. Every full‑QuerySet loop therefore caches every row's `content`.

**Amplifiers to keep in mind:** (a) `django‑q==1.3.9` long‑lived workers accumulate arena residue across tasks; (b) Django 4.0.4 QuerySet result caching holds all rows incl. `content` for a loop; (c) the `content` `TextField` (#16) is the multiplier behind nearly every other row.

---

## 6. Recommendations (suggestions only — NOT applied)

> **None of the following were implemented.** This is a read‑only investigation; the items below are described so the team can decide. Each is tied to the hotspot it addresses.

1. **Stream the full‑QuerySet loops.** Use `.iterator()` (optionally `chunk_size=...`) and/or `.only(...)`/`.defer("content")` in `tasks.py` (`L39`, `L271`), `sanity_checker.py:L61`, `document_archiver.py:L129`, `document_retagger.py:L74`. Measured headroom: S3a → S3b/S3c cut retained content from **23.5 MiB → ~0 / 0.5 MiB** for 300 docs (scales with library size). *Caveat:* `tqdm(...)` currently relies on `len()`; pass `total=` explicitly when switching to `.iterator()`.
2. **Stream file reads.** Replace `hashlib.md5(f.read())` (`consumer.py:L102‑L104`) with a chunked `update()` loop, and `_write`'s `read_file.read()` (`L429‑L432`) with `shutil.copyfileobj`. Measured: 2.0 MiB → 0.1 MiB transient for the 2 MiB fixture (S1a).
3. **Cache the classifier in worker memory.** `load_classifier()` is invoked per consume (`consumer.py:L292`) and per request (`views.py:L319`) and re‑unpickles every time (S6c == S6d). Loading once per worker (with mtime/hash invalidation) would remove a recurring multi‑hundred‑MiB churn term.
4. **Process the import manifest incrementally / drop it after use.** Avoid holding `self.manifest` for the command lifetime (`document_importer.py:L72‑L73`); stream records, and avoid the separate `loaddata` pass re‑reading the same file (`L87`) so the two whole‑export copies don't coexist.
5. **Bound the Whoosh writer.** Pass `limitmb`/`procs` to `AsyncWriter` (`index.py:L66`, `tasks.py:L43`/`L278`) to cap the native postings buffer that drove S3e's +331 MiB.
6. **Reduce per‑rule fuzzy copies.** Compute the punctuation‑stripped/lower‑cased content **once per document** rather than once per rule (`matching.py:L131,L134`).
7. **OS‑level allocator tuning for long‑lived workers.** Set `MALLOC_ARENA_MAX` (e.g., `2`) or switch to `jemalloc`/`tcmalloc` to curb glibc arena retention/fragmentation (§3.3–§3.4). This addresses the "RSS won't drop" symptom directly, without code changes.
8. **Keep `DEBUG=False` in production** (already the default, `settings.py:L50`) so `connection.queries` never accumulates.

---

## 7. Appendix: harness, raw results, environment facts

### 7.1 Environment (verified at runtime)

```
Python            3.9.23            (Dockerfile:L18 -> python:3.9-slim-bullseye)
Django            4.0.4   DEBUG=False   DB engine=django.db.backends.sqlite3
scikit-learn      1.0.2   numpy 1.22.3   scipy 1.8.0   joblib 1.1.0
whoosh            2.7.4   django-q 1.3.9  djangorestframework 3.13.1
dateparser        1.1.1   psutil 7.2.2 (profiling only; never added to repo manifests)
Image             paperless-ngx-mem:py39-ready  (HEAD 542221a38dff)
```

### 7.2 Raw per‑scenario results (Python 3.9.23, DEBUG=False)

```
SCENARIO 0 — CPython allocator (pymalloc + glibc)
  S0a 300k DISTINCT 64B (pymalloc):  RSS +114.7 MiB | tracemalloc peak 30.2 MiB
      after del ALL + gc.collect:    RSS 164.7 -> 122.4 MiB (most retained)
  fragmentation: 120 scattered survivors (~0.007 MiB live) -> RSS still 170.4 MiB
  S0c 6000 DISTINCT 4096B str (glibc): tracemalloc peak 47.2 MiB; RSS ~100 MiB after free

SCENARIO 1 — single consume primitives (2.00 MiB plain-text fixture)
  S1a  md5(f.read()) whole-file         tracemalloc peak  2.0 MiB   [consumer.py:L102-L104]
  S1a' chunked md5 (recommendation)     tracemalloc peak  0.1 MiB
  S1b  _write read_file.read() copy     tracemalloc peak  2.0 MiB   [consumer.py:L429-L432]
  S1c  parse_date() over full content   RSS +151.8 MiB | peak 25.7  [parsers.py:L261]
  S1d  post_process_text() copy chain   tracemalloc peak 18.6 MiB   [tesseract/parsers.py:L330-L341]

SCENARIO 2 — manifest export/import (300 docs)
  S2a  build manifest (serialize)       RSS +64.5 | peak 73.1 MiB   [exporter.py:L117-L130]
       manifest.json on disk            23.59 MiB (scales with TOTAL export)
  S2b  self.manifest = json.load(f)     +23.9 MiB RETAINED | peak 47.5  [importer.py:L72-L73]
       gc.get_referrers -> held by instance __dict__ (self.manifest)
  S2c  manifest_documents=list(filter)  extra list of 300 refs         [importer.py:L137-L139]
  S2d  loaddata (re-deserialize)        tracemalloc peak 47.9 MiB       [importer.py:L87]

SCENARIO 3 — QuerySet result cache + Whoosh (300 docs)
  S3a  list(Document.objects.all())     RSS +25.3 | peak 23.9 MiB | retained content 23.5 MiB [tasks.py:L39,L44]
  S3b  .iterator()                      RSS +0.0 retained | peak 15.9 MiB transient
  S3c  .defer('content')                tracemalloc peak  0.5 MiB
  S3d  .only('id','title')              tracemalloc peak  0.2 MiB
  S3e  tasks.index_reindex()            RSS +331.0 MiB | tracemalloc peak 166.7 | proc peak 595.3 MiB [tasks.py:L38-L45 + AsyncWriter]

SCENARIO 4 — type sensitivity + fuzzy matching
  post_process_text  16KB -> peak 0.15 MiB | 1MB -> 9.30 MiB | 4MB -> 36.50 MiB  [tesseract/parsers.py:L330-L341]
  fuzzy matches() 10 rules / 80KB doc   per-rule full-content copy 0.078 MiB each  [matching.py:L131,L134]

SCENARIO 6 — first-vs-subsequent / caching (300-doc corpus)
  S6a  load_classifier() NO model       returns None | RSS +0.0 MiB              [classifier.py:L31-L36]
  S6b  classifier.train()               RSS +1661.8 | peak 845.9 MiB (SYNTHETIC UPPER BOUND) [classifier.py:L115-L130]
  S6c  FIRST load_classifier() unpickle  RSS +263.4 | peak 420.0 MiB             [classifier.py:87 +305, :91 +114]
  S6d  SECOND load_classifier()          RSS +330.5 | peak 420.0 MiB (IDENTICAL -> NOT cached)
  S6e  dateparser FIRST parse            one-time init (regex core, ~1.1 MiB traced)
  S6f  dateparser SECOND parse           +0.0 MiB (locale already loaded)
```

### 7.3 Harness design (reference only — these scripts were external and have been deleted)

The harness was **not** committed to the repository. It consisted of two throwaway files under `/tmp/profiling/`:

- `memlib.py` — a `Measure` context manager that, per window, ran `gc.collect()`, `tracemalloc.clear_traces()`, `tracemalloc.reset_peak()`, took a *before* snapshot, ran the stage, took an *after* snapshot, and recorded `tracemalloc.get_traced_memory()` (current + peak), process RSS (`psutil` → `/proc/self/status` → `resource.getrusage`), and a post‑`gc.collect()` RSS. `snapshot_after.compare_to(snapshot_before, "lineno")` produced the per‑`file:line` deltas quoted above.
- `driver.py` — Django bootstrap (`DJANGO_SETTINGS_MODULE=paperless.settings`, asserting `DEBUG is False`), a seeder creating 300 distinct‑content `Document` rows, and scenario functions `s0…s6` that call the **real** Paperless‑ngx functions so attribution lands on `/app/src/...` lines.

Both files, plus all fixtures and the SQLite scratch DB, were deleted after measurement. Per the project rule and the user's verbatim instruction, the repository's working tree contains **only** this one new document as a change.

### 7.4 Verification of the read‑only guarantee

After authoring, `git status --porcelain` shows exactly one entry — this file — and no source file was modified or deleted. No profiling script remains inside the repository tree.

---

*End of report.*
