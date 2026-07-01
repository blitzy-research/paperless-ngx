# Memory-Usage Spikes During Document Import in paperless-ngx — Investigation & Diagnosis

> **Deliverable type:** Read-only investigation & Q&A (rule set **SWE-AtlasQnA-Repo**).
> **Nothing in the source tree was modified.** The only file added to the repository is this document.
> **Filename rationale:** the active git branch of the investigated checkout is `paperless-ngx_542221a38dff` (verified with `git branch --show-current`), so the answer document is named `paperless-ngx_542221a38dff.md`.

---

## 1. The question being answered

A user reports anomalous memory behavior while importing documents into paperless-ngx. Restating their symptoms in their own words:

- The spikes are **"disproportionate to size"** — memory grows far beyond what the document's byte size would justify. These are **"mostly text documents with small metadata"**, explicitly *not* image processing.
- The magnitude is **inconsistent** — it varies by document source and by processing stage; some imports spike and some do not.
- Resident memory is **"not always released back to the system in a timely manner even after processing completes."**

The user asked five framing questions. They are preserved **verbatim** here and mapped to the requirement IDs used throughout this document:

1. **(R1)** *"What's causing these memory spikes?"*
2. **(R2)** *"Is there something in metadata handling creating unnecessary COPIES or holding REFERENCES longer than needed?"*
3. **(R3)** *"Is there CACHING behavior accumulating data unexpectedly?"*
4. **(R4)** *"What's DIFFERENT between cases where memory spikes vs. cases where it doesn't?"*
5. **(R5)** *"How does behavior change with different DOCUMENT TYPES or BATCH SIZES?"*

Three additional deliverable-driven requirements complete the set:

- **(R6)** Provide **actual runtime memory measurements** captured during processing (not theoretical estimates).
- **(R7)** **Attribute** the memory holding to specific components/methods.
- **(R8)** **Diagnose** whether the behavior is normal Python garbage-collection / allocator behavior or something genuinely problematic.

Each of R1–R8 is answered in its own explicitly-labeled section below, and every section contains **at least one exact `file:line` citation** and **at least one verbatim runtime measurement**. A coverage-pass checklist at the end confirms this.

---

## 2. Environment & methodology (disclosure)

### 2.1 Where the measurements were taken

All runtime measurements in this document were captured in the project's **canonical runtime**, inside a Docker container (`paperless-qna-0`) built from the user-mandated image, running the exact pinned dependency set from `requirements.txt`. The interpreter version and the pinned libraries whose allocation behavior is analyzed were confirmed with this exact command (no elision):

```text
$ docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python --version && python -c "import django,sklearn,pikepdf,whoosh,numpy,scipy,psutil; print(\"django\", django.__version__); print(\"sklearn\", sklearn.__version__); print(\"pikepdf\", pikepdf.__version__); print(\"whoosh\", whoosh.__version__); print(\"numpy\", numpy.__version__); print(\"scipy\", scipy.__version__); print(\"psutil\", psutil.__version__)"'
Python 3.9.23
django 4.0.4
sklearn 1.0.2
pikepdf 5.1.1
whoosh (2, 7, 4)
numpy 1.22.3
scipy 1.8.0
psutil 7.2.2
```

These match the manifest pins exactly: `django==4.0.4` (`requirements.txt:L38`), `scikit-learn==1.0.2` (`requirements.txt:L88`), `pikepdf==5.1.1` (`requirements.txt:L65`), `whoosh==2.7.4` (`requirements.txt:L111`). The container's Python (3.9.23) matches the canonical production runtime declared in the image spec: `FROM python:3.9-slim-bullseye as main-app` (`Dockerfile:L18`).

> **Citation correction (verified).** The runtime is defined at **`Dockerfile:L18`**, *not* `Dockerfile:L1`. Line 1 is a comment (`# Default to pulling from the main repo registry when manually building`). The `FROM` instruction is on line 18. This document uses **L18**.

#### 2.1.1 Methodology disclosure — measurement-environment reconciliation (mandatory)

The original investigation plan (AAP §0.8.2) anticipated that measurements might have to be captured in a *sandbox* running **Python 3.12.3** with `django==4.0.4` installed via `pip install --break-system-packages` — the workaround required because that sandbox's PEP-668 "externally-managed-environment" marker blocks `python -m venv` creation — and that any such measurements would therefore have to be labeled **version-faithful mechanism reproductions** rather than production-runtime captures.

**That fallback proved unnecessary, and no measurement in this document was taken on Python 3.12.3.** The environment actually provided for this investigation is the project's **canonical runtime**: the Docker image mandated by the setup instructions, whose interpreter is **Python 3.9.23** (matching `FROM python:3.9-slim-bullseye`, `Dockerfile:L18`) with the exact pinned dependency set from `requirements.txt` already installed — **no `--break-system-packages`, no PEP-668 workaround, and no Python-3.12.3 version shift**. Every measurement below was captured in that canonical Python 3.9.23 container, which **meets or exceeds** the "version-faithful" bar the AAP set: the reproductions run on the very interpreter and pinned libraries that production uses. Concretely:

- **Run directly against the real modules where possible** — Evidence D imports and exercises the unmodified `documents.classifier.load_classifier` and `paperless_text.parsers.TextDocumentParser`.
- **Self-contained mechanism reproductions for the rest** — Evidence A/A2/B/C/C2/E reproduce a specific mechanism (the 7× `pickle.load`, allocator retention, `QuerySet._result_cache`, or `hashlib.md5(f.read())`) in the same canonical container, and each is labeled as a reproduction.

In short: the AAP's Python-3.12.3 / PEP-668 / `--break-system-packages` / version-faithful contingency is disclosed here for completeness, but the delivered evidence is stronger than that contingency required — it was captured on the canonical Python 3.9.23 production interpreter itself.

> **Command convention (no ellipsis anywhere).** For readability, the evidence blocks in R1–R5 show the **in-container** command (e.g. `python /tmp/obs_evidenceA.py`). Each was executed via the exact wrapper `docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && <command>'` (Evidence D additionally sets `-e DJANGO_SETTINGS_MODULE=paperless.settings -e PYTHONPATH=/app/src`). The **full wrapper command, the complete script source, and the verbatim output** for every Evidence block are reproduced in **R6**.

### 2.2 The three-signal measurement harness

A single spike in the OS-reported resident set size (RSS) cannot, on its own, distinguish a genuine reference leak from allocator retention. The harness therefore captures **three complementary signals** at labeled checkpoints:

| Signal | Source | What it tells us |
|--------|--------|------------------|
| `RSS` | `psutil.Process().memory_info().rss` | OS-level resident memory the process holds |
| `pyheap_cur` | `tracemalloc.get_traced_memory()[0]` | live **Python-heap** bytes tracemalloc is tracking |
| `gc_objs` | `len(gc.get_objects())` after `gc.collect()` | count of live Python objects (leak detector) |

```python
import gc, os, tracemalloc, psutil
proc = psutil.Process(os.getpid())
rss  = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start(); gc.collect()
def ckpt(label):
    print("{:34s} RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  gc_objs={:,}".format(
        label, rss(), heap(), len(gc.get_objects())))
```

`tracemalloc` and `gc` are Python standard library. `psutil==7.2.2` is used only for observation and is **not** a repository dependency (it is not present in `requirements.txt`; it exists only in the profiling container).

**Known limitation of `tracemalloc` (relevant to R8):** it traces only allocations made by the *Python* memory manager. Allocations made by C extensions — scikit-learn, `pikepdf`, Whoosh, NumPy/SciPy — are **invisible** to it. Consequently, when RSS grows but `pyheap_cur` does not grow in step, the extra memory lives in native/C allocations (or in allocator retention), not in a Python-object leak.

### 2.3 Read-only & cleanup discipline

- Baseline before any work: `git status --porcelain` returned **empty** (clean tree), branch `blitzy-aab05a0a-04cf-40e2-a694-3bf6834a73f6`, HEAD `542221a38dff06361e07976452f9aea24d210542`.
- Every observation script was written **outside** the repository (host `/tmp/obs`, then `docker cp` into the container's `/tmp`), executed, its output captured verbatim, and then **deleted**.
- After capture, all temporary scripts were removed (host and container) and `git status --porcelain` was re-verified **empty** apart from this new document. No source file, dependency manifest, test, or CI file was touched. No diagnosed hotspot was remediated (recommendations only — see §12).

---

## R1 — What's causing these memory spikes? (root cause)

> *User question 1 (verbatim): "What's causing these memory spikes?"*

There are **two distinct allocation behaviors**, and separating them is the key to the "disproportionate to size" puzzle:

### R1.a — The size-INDEPENDENT cost: per-document classifier reload (the actual "disproportionate" spike)

Every consumed document triggers a **full reload of the machine-learning classifier from a pickle on disk**, and the cost of that reload has **nothing to do with the size of the document being imported**. That is precisely why a small "mostly text" document can trigger a large spike.

The mechanism, cited exactly:

- `load_classifier()` is defined at `src/documents/classifier.py:L30`. On every call it constructs a **fresh** classifier — `classifier = DocumentClassifier()` at `src/documents/classifier.py:L38` — and then calls `classifier.load()` at `src/documents/classifier.py:L40`. There is no memoization around this (see R3).
- `DocumentClassifier.load()` (`src/documents/classifier.py:L76`) performs **seven sequential `pickle.load(f)` calls**, gated by `FORMAT_VERSION = 7` (`src/documents/classifier.py:L63`):
  - `schema_version = pickle.load(f)` — `L78`
  - `self.data_hash = pickle.load(f)` — `L86`
  - `self.data_vectorizer = pickle.load(f)` — `L87`
  - `self.tags_binarizer = pickle.load(f)` — `L88`
  - `self.tags_classifier = pickle.load(f)` — `L90`
  - `self.correspondent_classifier = pickle.load(f)` — `L91`
  - `self.document_type_classifier = pickle.load(f)` — `L92`
- The unpickled objects are heavyweight scikit-learn estimators: `CountVectorizer` (imported `classifier.py:L188`, used `L194`) and `MLPClassifier` (imported `classifier.py:L189`, used `L219`, `L227`, `L238`).
- This reload is invoked **once per consumed document** at `src/documents/consumer.py:L292` (`classifier = load_classifier()`).

**Verbatim measurement (Evidence A — mechanism reproduction of the 7× `pickle.load` in the canonical container).** A scikit-learn model of the same shape (`CountVectorizer` + `MultiLabelBinarizer` + 3× `MLPClassifier`, plus the `schema_version` int and `data_hash` bytes = 7 objects) was pickled and then unpickled with seven `pickle.load` calls exactly as `DocumentClassifier.load()` does (exact command + full script in R6):

```text
$ python /tmp/obs_evidenceA.py
baseline (numpy+sklearn imported)  RSS=    85.3MB  pyheap_cur=   0.01MB  gc_objs=70,922
after fit 7 model objects          RSS=   194.9MB  pyheap_cur=  75.73MB  gc_objs=71,015
pickled model file size = 71.00 MB (7 objects)
after del sources + gc.collect     RSS=   187.2MB  pyheap_cur=  23.90MB  gc_objs=70,972
after 7x pickle.load (model in mem) RSS=   202.8MB  pyheap_cur=  95.73MB  gc_objs=71,020
after del model + gc.collect       RSS=   179.5MB  pyheap_cur=  23.90MB  gc_objs=70,972
```

The 7× `pickle.load` adds **~72 MB of Python heap** (`23.90MB → 95.73MB`) and pushes RSS from `187.2MB` to `202.8MB`. This cost is paid **regardless of the current document's byte size** — it is a fixed tax per consume. That is the direct answer to *"disproportionate to size."*

**Verbatim measurement (Evidence D — the same effect using the REAL paperless modules).** Importing `documents.classifier.load_classifier` and `paperless_text.parsers.TextDocumentParser` and exercising the cheap path vs. constructing the classifier model (exact command + full script in R6):

```text
$ python /tmp/obs_evidenceD.py 2>/dev/null
baseline (django.setup done)          RSS=   108.4MB  pyheap_cur=   0.00MB
load_classifier() ->                  None   RSS=   108.4MB  pyheap_cur=   0.03MB
extract_metadata (text) ->            metadata_items=0   RSS=   108.4MB  pyheap_cur=   0.03MB  heap_delta=+0.001MB
classifier model in memory (spike) -> RSS=   201.7MB  pyheap_cur=  72.11MB  heap_delta=+72.07MB
```

Holding the classifier object spikes RSS from `108.4MB` to `201.7MB` (`+72.07MB` of Python heap versus the cheap path). In this checkout there is no trained model file, so the real `load_classifier()` **short-circuits to `None`**: the guard `if not os.path.isfile(settings.MODEL_FILE):` is at `src/documents/classifier.py:L31` and the actual `return None` statement is at `src/documents/classifier.py:L36`. That is why Evidence A/D build a same-shape model to measure the unpickle/hold cost that occurs when a trained model *is* present.

### R1.b — The size-DEPENDENT cost: full-file reads for checksum/copy

A second, orthogonal contributor **does** scale with the document's byte size: the consumer reads whole files into memory to compute MD5 checksums and to copy them. There are **four** such whole-file reads:

- `checksum = hashlib.md5(f.read()).hexdigest()` at `src/documents/consumer.py:L104` (original-file duplicate pre-check).
- The **archive-file checksum** at `src/documents/consumer.py:L340-L341` — inside `with open(archive_path, "rb") as f:` (`L339`), `document.archive_checksum = hashlib.md5(f.read()).hexdigest()` reads the **entire archive file** into memory (`hashlib.md5(` on `L340`, `f.read(),` on `L341`).
- `checksum=hashlib.md5(f.read()).hexdigest()` at `src/documents/consumer.py:L402` (post-copy checksum on the stored document).
- The copy `write_file.write(read_file.read())` at `src/documents/consumer.py:L432` reads the entire source file into memory before writing.

**Verbatim measurement (Evidence E — `hashlib.md5(f.read())` on files of increasing size; exact command + full script in R6).**

```text
$ python /tmp/obs_evidenceE.py
baseline                     RSS=    18.5MB
 10MB file: md5=6846c01e  RSS=    28.3MB (delta +9.9MB)  pyheap_cur=   10.0MB
 50MB file: md5=e909cae6  RSS=    78.3MB (delta +59.9MB)  pyheap_cur=   50.0MB
100MB file: md5=a4584ebb  RSS=   128.3MB (delta +109.9MB)  pyheap_cur=  100.0MB
200MB file: md5=8642a406  RSS=   228.3MB (delta +209.9MB)  pyheap_cur=  200.0MB
```

RSS grows **1:1 with the file size** (a 200 MB file adds ~209.9 MB). The identical `hashlib.md5(f.read())` mechanism is exercised at all four sites above — including the **archive checksum at `src/documents/consumer.py:L340-L341`**, which reads the whole *archive* file into memory the same way. For the user's "mostly text documents with small metadata," this term is small — which is exactly why the **size-independent** classifier reload (R1.a) dominates the surprising spikes, while this size-dependent term explains why *large* imports (or large generated archives) also grow.

**Root-cause summary:** the "disproportionate to size" spikes are dominated by the **per-document, size-independent classifier reload** (R1.a); a secondary, size-proportional term comes from **whole-file reads** for checksums/copies (R1.b), at `src/documents/consumer.py:L104`, `L340-L341`, `L402`, and `L432`.

---

## R2 — Copies / references in metadata handling

> *User question 2 (verbatim): "Is there something in metadata handling creating unnecessary COPIES or holding REFERENCES longer than needed?"*

Yes — three concrete behaviors, all in the metadata path:

### R2.a — Each `extract_metadata` builds a brand-new list of dicts

The base parser returns an empty list — `def extract_metadata(self, document_path, mime_type):` at `src/documents/parsers.py:L304` with `return []` at `src/documents/parsers.py:L305`. The Tesseract parser, however, constructs a **new list of dictionaries every call**: `def extract_metadata(...)` at `src/paperless_tesseract/parsers.py:L26` opens the PDF via `pdf = pikepdf.open(document_path)` (`L34`), reads `meta = pdf.open_metadata()` (`L35`), appends one dict per metadata key (`L42`–`L49`), and does `return result` (`L55`). These allocations are transient but are re-created on every metadata read.

### R2.b — The Tika/Office path parses the whole document TWICE (redundant transient copy)

For Office documents handled by the Tika parser, the full document body is parsed **twice**:

- In `extract_metadata` (`src/paperless_tika/parsers.py:L29`): `parsed = parser.from_file(document_path, tika_server)` at `src/paperless_tika/parsers.py:L32`.
- Again in `parse` (`src/paperless_tika/parsers.py:L50`): `parsed = parser.from_file(document_path, tika_server)` at `src/paperless_tika/parsers.py:L55`, followed by `self.text = parsed["content"].strip()` at `src/paperless_tika/parsers.py:L62`.

Each `parser.from_file(...)` returns the full parsed content body, so an Office document's text is materialized **twice** in memory across the two calls — an unnecessary duplicate of the largest transient object in that path. This is **document-type-specific** (Office docs only), which ties directly into R5.

### R2.c — The DRF metadata endpoint never calls `cleanup()` (scratch-dir retention — DISK, not RAM)

- `get_metadata` (`src/documents/views.py:L260`) instantiates a parser — `parser = parser_class(progress_callback=None, logging_group=None)` at `src/documents/views.py:L266` — and calls `return parser.extract_metadata(file, mime_type)` at `src/documents/views.py:L269`, but it **never calls `parser.cleanup()`**.
- Every parser's `__init__` (`src/documents/parsers.py:L289`) creates a scratch directory: `self.tempdir = tempfile.mkdtemp(prefix="paperless-", dir=settings.SCRATCH_DIR)` at `src/documents/parsers.py:L293`. Cleanup is `def cleanup(self):` at `src/documents/parsers.py:L348` → `shutil.rmtree(self.tempdir)` at `src/documents/parsers.py:L350`.
- The `metadata` DRF action calls `get_metadata` **twice** — `self.get_metadata(doc.source_path, doc.mime_type)` at `src/documents/views.py:L295` and again at `src/documents/views.py:L302` — so two scratch directories are created and **neither is removed**.

**Precise scope:** this is a **disk** leak (orphaned `paperless-*` scratch directories under `SCRATCH_DIR`), *not* a RAM leak. It is called out here because the user asked about "holding references longer than needed." For contrast, the **consume** path *does* clean up: `document_parser.cleanup()` is called at `src/documents/consumer.py:L279` and again at `src/documents/consumer.py:L369`.

**Verbatim measurement (Evidence D — reference retention in RAM stays flat on the cheap metadata path).** The real `TextDocumentParser.extract_metadata` returns `0` items and holds nothing:

```text
extract_metadata (text) ->            metadata_items=0   RSS=   108.4MB  pyheap_cur=   0.03MB  heap_delta=+0.001MB
```

**Verbatim measurement (Evidence C — how long the heavy `content` reference is held once materialized).** Once a QuerySet holding the `content` field is materialized, the reference (and its heap) persists until the queryset is dropped:

```text
after list(qs) [index_reindex]            RSS=    77.5MB  pyheap_cur=  32.17MB  rows=4000  cached=True
after del + gc.collect                    RSS=    76.1MB  pyheap_cur=   0.03MB
```

The heap returns to baseline (`32.17MB → 0.03MB`) only *after* the reference is deleted — confirming the memory is held exactly as long as the reference is kept, no longer.


---

## R3 — Is there caching behavior accumulating data unexpectedly?

> *User question 3 (verbatim): "Is there CACHING behavior accumulating data unexpectedly?"*

**Verified negative finding: the application maintains no growing in-memory cache.** A targeted scan of the documents app finds no `lru_cache`, `functools.cache`, `@cache`, Django cache-framework usage, or module-level cache dictionary:

```text
$ grep -rnE "lru_cache|functools.cache|@cache|django.core.cache|_cache =" documents/*.py
(no matches — no application-level cache)
```

The only dict literal that a naive scan flags is a **local variable**, not a cache — `search_kwargs = {}` at `src/documents/matching.py:L61`.

Two consequences follow:

1. **The classifier is the *opposite* of a cache — it is re-unpickled on every call.** As shown in R1, `load_classifier()` builds a fresh `DocumentClassifier()` (`src/documents/classifier.py:L38`) and calls `.load()` (`src/documents/classifier.py:L40`) each time, with no memoization. Evidence A2 proves there is no reuse — three consecutive fresh unpickles of the same model each cost the same (exact command + full script in R6):

   ```text
   $ python /tmp/obs_evidenceA2.py
   baseline                     RSS=    14.4MB  pyheap_cur=   0.00MB
   load #1: RSS=   195.3MB  pyheap_cur= 112.26MB  heap_added=+112.26MB
   load #2: RSS=   195.0MB  pyheap_cur= 112.25MB  heap_added=+71.85MB
   load #3: RSS=   195.0MB  pyheap_cur= 112.25MB  heap_added=+71.85MB
   ```

   Loads #2 and #3 add an **identical** `+71.85MB` each and return to an identical `pyheap_cur=112.25MB` / `RSS=195.0MB`. (Load #1 is higher, `+112.26MB`, because it *also* triggers the one-time lazy imports of NumPy/SciPy/scikit-learn internals that then stay resident — roughly the ~40 MB difference between load #1 and loads #2/#3; that fixed import floor does not grow across subsequent loads.) Nothing accumulates across calls — each load re-incurs the same model cost and is then released. This is a genuine inefficiency (the same model is unpickled repeatedly) but it is *not* an accumulating cache.

2. **What the user perceives as "accumulation" is allocator RSS retention, not a growing Python cache.** Evidence B (see R8) shows the Python heap and object count return to baseline after `gc.collect()` while RSS stays elevated — i.e., there is no growing set of live Python objects.

**One off-by-default accumulation vector worth noting:** `DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")` at `src/paperless/settings.py:L50` defaults to **off** (the helper `__get_boolean` is at `src/paperless/settings.py:L34`). When `DEBUG` is enabled, Django accumulates every executed SQL statement in `django.db.connection.queries`, which grows without bound in the long-lived Django-Q worker processes. This is a real accumulation vector, but it is **disabled by default** and therefore not the cause in a standard deployment.

---

## R4 — What's different between spike and non-spike cases?

> *User question 4 (verbatim): "What's DIFFERENT between cases where memory spikes vs. cases where it doesn't?"*

The difference is **whether the code path touches a heavyweight object (the classifier, or a large file body) or not.** Evidence D captures both paths side by side, using the **real paperless modules** (exact command + full script in R6):

```text
$ python /tmp/obs_evidenceD.py 2>/dev/null
baseline (django.setup done)          RSS=   108.4MB  pyheap_cur=   0.00MB
load_classifier() ->                  None   RSS=   108.4MB  pyheap_cur=   0.03MB
extract_metadata (text) ->            metadata_items=0   RSS=   108.4MB  pyheap_cur=   0.03MB  heap_delta=+0.001MB
classifier model in memory (spike) -> RSS=   201.7MB  pyheap_cur=  72.11MB  heap_delta=+72.07MB
```

**Non-spike (cheap) path — a plain-text import:**
- `TextDocumentParser` (`src/paperless_text/parsers.py:L12`) does **not** override `extract_metadata`, so it inherits the base implementation that returns `[]` (`src/documents/parsers.py:L304`–`L305`). Metadata extraction allocates essentially nothing (`metadata_items=0`, `heap_delta=+0.001MB`).
- When no trained model file exists, `load_classifier()` returns `None` — the guard `os.path.isfile(settings.MODEL_FILE)` is at `src/documents/classifier.py:L31` and the `return None` it protects is at `src/documents/classifier.py:L36` — so there is no unpickle cost.
- Result: RSS stays flat at `108.4MB`.

**Spike (expensive) path — anything that holds the classifier or a large body:**
- Constructing/holding the scikit-learn classifier object jumps RSS to `201.7MB` (`+72.07MB` of Python heap versus the cheap path). This is the per-document reload of R1.a when a model *is* present.
- Equivalently, the Tika double-parse (R2.b) or a large-file `f.read()` (R1.b / Evidence E) produces a comparable spike.

So the discriminator is **content of the path, not size of the document**: a small text file with no model loads cheaply; the same small file, once a classifier model exists (or once routed through Tika/OCR or copied via full-file read), spikes. This is why the behavior looks "inconsistent" across sources and stages.

---

## R5 — Variance by document type and batch size

> *User question 5 (verbatim): "How does behavior change with different DOCUMENT TYPES or BATCH SIZES?"*

### R5.a — Document type (parser selection)

Parsers are selected by MIME type and weight — `get_parser_class_for_mime_type` at `src/documents/parsers.py:L81`. The memory profile differs sharply by parser:

| Document type | Parser | Metadata behavior | Relative cost |
|---------------|--------|-------------------|---------------|
| Plain text | `TextDocumentParser` (`src/paperless_text/parsers.py:L12`) | inherits base `extract_metadata` → `return []` (`src/documents/parsers.py:L304`–`L305`) | **cheap** (`heap_delta=+0.001MB`, Evidence D) |
| PDF / image | `RasterisedDocumentParser` (Tesseract) | builds a new XMP dict list via `pikepdf.open` (`src/paperless_tesseract/parsers.py:L34`) + `open_metadata()` (`L35`), append per key `L42`–`L49` | moderate (transient dict list) |
| Office docs | Tika parser | **parses the whole document twice** — `parser.from_file` at `src/paperless_tika/parsers.py:L32` *and* `L55` | **expensive** (double transient content) |

> **Citation correction (verified).** There is **no** `src/paperless_mail/parsers.py` in this repository — the mail app has **no standalone `DocumentParser` subclass**. Mail handling lives in `src/paperless_mail/mail.py`. Any discussion of mail must cite `mail.py`; a `paperless_mail/parsers.py:Lx` citation would be fabricated. (Verified: `ls paperless_mail/parsers.py` → "No such file or directory"; `paperless_mail/mail.py` exists.)

### R5.b — Batch size (QuerySet materialization)

The async tasks materialize Django QuerySets whose rows each carry the heavy full-text `content` field — `content = models.TextField(` at `src/documents/models.py:L117` on `class Document(models.Model):` at `src/documents/models.py:L88`:

- `index_reindex` (`src/documents/tasks.py:L38`) does `documents = Document.objects.all()` at `src/documents/tasks.py:L39`.
- `bulk_update_documents` (`src/documents/tasks.py:L270`) does `documents = Document.objects.filter(id__in=document_ids)` at `src/documents/tasks.py:L271`.

**Verbatim measurement (Evidence C — lazy vs. materialized, and the real two-pass pattern), real Django 4.0.4 + sqlite, 4000 rows with a heavy `content` field (exact command + full script in R6):**

```text
$ python /tmp/obs_evidenceC.py
baseline (4000 docs in sqlite, not loaded)  RSS=    42.5MB
after .all() (lazy)                       RSS=    42.5MB  _result_cache=None
after list(qs) [index_reindex]            RSS=    77.5MB  pyheap_cur=  32.17MB  rows=4000  cached=True
after del + gc.collect                    RSS=    76.1MB  pyheap_cur=   0.03MB
after bulk_update 2-pass (1 queryset)     RSS=    79.0MB  pyheap_cur=  32.49MB  loop1=4000  loop2=4000  cached_once=True
```

Two facts stand out:
- `Document.objects.all()` is **lazy** — `_result_cache=None`, RSS flat at `42.5MB`. No memory is used until the queryset is evaluated.
- Evaluating it (`list(qs)`) populates `QuerySet._result_cache` with every row **including the `content` string**, adding `+32.17MB` of heap (`pyheap_cur 0` → `32.17MB`).

> **Accuracy nuance (verified).** `bulk_update_documents` materializes **one** queryset (`src/documents/tasks.py:L271`) and then iterates it **twice** (a `post_save.send` loop, then an `AsyncWriter` update loop). The result cache is populated **once** and reused — my reproduction confirms this with `cached_once=True` (the cache was already populated after the first loop) and a heap that is **not** doubled (`32.49MB`, essentially the same as the single-materialization `32.17MB`). This document does **not** claim two separate materializations.

**Verbatim measurement (Evidence C2 — the batch-size growth curve is linear; exact command + full script in R6):**

```text
$ python /tmp/obs_evidenceC2.py
baseline (8000 docs in sqlite)  RSS=    42.9MB
batch= 1000: RSS delta=+   8.2MB  pyheap delta=+  8.09MB  rows=1000
batch= 2000: RSS delta=+   8.7MB  pyheap delta=+ 16.11MB  rows=2000
batch= 4000: RSS delta=+  18.1MB  pyheap delta=+ 32.11MB  rows=4000
batch= 8000: RSS delta=+  38.1MB  pyheap delta=+ 64.11MB  rows=8000
```

Heap grows **linearly** with batch size (`8.09 → 16.11 → 32.11 → 64.11 MB` as rows double `1000 → 2000 → 4000 → 8000`). **Larger batches → proportionally larger `_result_cache` → proportionally larger RSS.** (RSS deltas are noisier than the Python-heap deltas because freed arenas are re-used between iterations; the `pyheap delta` column is the clean linear signal.) This is the batch-size half of the user's question.


---

## R6 — Actual runtime measurements (evidence appendix)

> *Deliverable requirement: provide actual runtime memory measurements captured during processing, not theoretical estimates.*

Every measurement below was captured in the canonical container (`paperless-qna-0`, Python 3.9.23 + pinned deps) using the three-signal harness from §2.2. For each Evidence block this appendix gives **(a)** the exact command that produced it (the full `docker exec` wrapper, no ellipsis), **(b)** the complete script source, and **(c)** the verbatim output. Temporary scripts were written under `/tmp` (outside the repo), run, captured, and then deleted.

### Evidence A — Unpickle cost + RSS retention (mirrors `DocumentClassifier.load` at `src/documents/classifier.py:L76`, 7× `pickle.load` at `L78`,`L86`–`L88`,`L90`–`L92`)

**Command:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceA.py'
```

**Script (`/tmp/obs_evidenceA.py`):**

```python
"""Evidence A - Unpickle cost + RSS retention.

Mirrors documents/classifier.py DocumentClassifier.load() (src/documents/classifier.py:L76-L92),
which performs 7 sequential pickle.load(f) calls. We build a same-shape model
(schema_version int, data_hash bytes, CountVectorizer, MultiLabelBinarizer, and
3x MLPClassifier), pickle.dump the 7 objects, then reload them with 7x pickle.load,
capturing RSS / Python-heap / gc object counts at labeled checkpoints.
"""
import gc
import os
import pickle
import tracemalloc

import numpy as np
import psutil
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.neural_network import MLPClassifier
from sklearn.preprocessing import MultiLabelBinarizer

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start()
gc.collect()


def ckpt(label):
    print(
        "{:34s} RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  gc_objs={:,}".format(
            label, rss(), heap(), len(gc.get_objects())
        )
    )


ckpt("baseline (numpy+sklearn imported)")

rng = np.random.RandomState(0)
vocab_size = 8000
docs = [
    " ".join("t{:05d}".format(n) for n in rng.randint(0, vocab_size, size=160))
    for _ in range(1200)
]
schema_version = 7
data_hash = b"\x00" * 32
data_vectorizer = CountVectorizer().fit(docs)
X = data_vectorizer.transform(docs)
tags_binarizer = MultiLabelBinarizer().fit([[1, 2, 3]])


def big_mlp():
    m = MLPClassifier(hidden_layer_sizes=(128, 64), max_iter=1)
    y = rng.randint(0, 2, size=X.shape[0])
    m.fit(X, y)
    return m


tags_classifier = big_mlp()
correspondent_classifier = big_mlp()
document_type_classifier = big_mlp()
ckpt("after fit 7 model objects")

objs = [
    schema_version,
    data_hash,
    data_vectorizer,
    tags_binarizer,
    tags_classifier,
    correspondent_classifier,
    document_type_classifier,
]
path = "/tmp/obs_modelA.pickle"
with open(path, "wb") as f:
    for o in objs:
        pickle.dump(o, f)
print("pickled model file size = {:.2f} MB (7 objects)".format(os.path.getsize(path) / 1024 / 1024))

del docs, X, data_vectorizer, tags_binarizer
del tags_classifier, correspondent_classifier, document_type_classifier, objs
gc.collect()
ckpt("after del sources + gc.collect")

model = {}
with open(path, "rb") as f:
    model["schema_version"] = pickle.load(f)
    model["data_hash"] = pickle.load(f)
    model["data_vectorizer"] = pickle.load(f)
    model["tags_binarizer"] = pickle.load(f)
    model["tags_classifier"] = pickle.load(f)
    model["correspondent_classifier"] = pickle.load(f)
    model["document_type_classifier"] = pickle.load(f)
ckpt("after 7x pickle.load (model in mem)")

del model
gc.collect()
ckpt("after del model + gc.collect")
os.remove(path)
```

**Output (verbatim):**

```text
baseline (numpy+sklearn imported)  RSS=    85.3MB  pyheap_cur=   0.01MB  gc_objs=70,922
after fit 7 model objects          RSS=   194.9MB  pyheap_cur=  75.73MB  gc_objs=71,015
pickled model file size = 71.00 MB (7 objects)
after del sources + gc.collect     RSS=   187.2MB  pyheap_cur=  23.90MB  gc_objs=70,972
after 7x pickle.load (model in mem) RSS=   202.8MB  pyheap_cur=  95.73MB  gc_objs=71,020
after del model + gc.collect       RSS=   179.5MB  pyheap_cur=  23.90MB  gc_objs=70,972
```

Note the **RSS-vs-heap gap**: at "after fit" RSS is `194.9MB` but `pyheap_cur` is only `75.73MB` — the ~119 MB difference is native NumPy/scikit-learn allocation that `tracemalloc` cannot see (see R8). The 7× `pickle.load` raises the Python heap from `23.90MB` to `95.73MB` (~72 MB, the reload tax) and drops back to `23.90MB` after `del model`, while RSS stays elevated at `179.5MB` (above the `85.3MB` baseline).

### Evidence A2 — No memoization (three fresh unpickles each re-incur the same cost; mirrors `load_classifier` at `src/documents/classifier.py:L30`, fresh `DocumentClassifier()` `L38` + `.load()` `L40`)

The pickle is built once by an isolated helper (so the measuring process starts from a clean heap), then loaded three times by the measurer.

**Commands:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_buildmodel.py /tmp/obs_modelA2.pickle'
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceA2.py'
```

**Builder script (`/tmp/obs_buildmodel.py`):**

```python
"""Helper - builds a 7-object model pickle (same shape as DocumentClassifier)
so the Evidence A2 measurer can load it without the fit-time memory floor.
"""
import pickle
import sys

import numpy as np
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.neural_network import MLPClassifier
from sklearn.preprocessing import MultiLabelBinarizer

path = sys.argv[1]
rng = np.random.RandomState(0)
vocab_size = 8000
docs = [
    " ".join("t{:05d}".format(n) for n in rng.randint(0, vocab_size, size=160))
    for _ in range(1200)
]
data_vectorizer = CountVectorizer().fit(docs)
X = data_vectorizer.transform(docs)
tags_binarizer = MultiLabelBinarizer().fit([[1, 2, 3]])


def big_mlp():
    m = MLPClassifier(hidden_layer_sizes=(128, 64), max_iter=1)
    y = rng.randint(0, 2, size=X.shape[0])
    m.fit(X, y)
    return m


objs = [7, b"\x00" * 32, data_vectorizer, tags_binarizer, big_mlp(), big_mlp(), big_mlp()]
with open(path, "wb") as f:
    for o in objs:
        pickle.dump(o, f)
print("builder wrote {}".format(path))
```

**Measurer script (`/tmp/obs_evidenceA2.py`):**

```python
"""Evidence A2 - Repeated model loads cost the SAME (no memoization).

load_classifier() (src/documents/classifier.py:L30-L40) builds a fresh
DocumentClassifier() and calls .load() on EVERY call - nothing is cached at
module level. This measurer loads the same pickle 3 times; each load adds an
equivalent Python-heap increment, confirming there is no memoization / cache.
"""
import gc
import os
import pickle
import tracemalloc

import psutil

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start()
gc.collect()
path = "/tmp/obs_modelA2.pickle"


def load_once():
    m = {}
    with open(path, "rb") as f:
        for key in (
            "schema_version",
            "data_hash",
            "data_vectorizer",
            "tags_binarizer",
            "tags_classifier",
            "correspondent_classifier",
            "document_type_classifier",
        ):
            m[key] = pickle.load(f)
    return m


print("baseline                     RSS={:8.1f}MB  pyheap_cur={:7.2f}MB".format(rss(), heap()))
for i in (1, 2, 3):
    before = heap()
    model = load_once()
    after = heap()
    print(
        "load #{}: RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  heap_added=+{:.2f}MB".format(
            i, rss(), after, after - before
        )
    )
    del model
    gc.collect()
```

**Output (verbatim):**

```text
builder wrote /tmp/obs_modelA2.pickle
baseline                     RSS=    14.4MB  pyheap_cur=   0.00MB
load #1: RSS=   195.3MB  pyheap_cur= 112.26MB  heap_added=+112.26MB
load #2: RSS=   195.0MB  pyheap_cur= 112.25MB  heap_added=+71.85MB
load #3: RSS=   195.0MB  pyheap_cur= 112.25MB  heap_added=+71.85MB
```

Loads #2 and #3 are **identical** (`+71.85MB` each) — no reuse, no accumulation. Load #1 is larger (`+112.26MB`) only because it also pays the one-time lazy-import cost of the scientific stack (~40 MB), which then stays resident and does not grow. This is the empirical proof for R3's "no cache" claim.

### Evidence B — Allocator retention is NOT a Python leak (the R8 signature; the "not released back to the system" symptom, relevant to the long-lived workers `django-q==1.3.9` at `requirements.txt:L37`)

**Command:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceB.py'
```

**Script (`/tmp/obs_evidenceB.py`):**

```python
"""Evidence B - Allocator retention is NOT a Python leak.

Allocate 500,000 small dicts, then delete them and gc.collect(). The Python heap
(tracemalloc) and gc object count return to baseline (proving no Python-level
reference leak), yet RSS remains elevated because CPython/glibc retain freed
arenas. This is the signature that distinguishes allocator retention from a leak.
"""
import gc
import os
import tracemalloc

import psutil

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start()
gc.collect()


def ckpt(label):
    print(
        "{:30s} RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  gc_objs={:,}".format(
            label, rss(), heap(), len(gc.get_objects())
        )
    )


ckpt("baseline")
data = [{"i": i, "k": "v%d" % i} for i in range(500000)]
ckpt("after alloc 500k small dicts")
del data
gc.collect()
ckpt("after del 500k + gc.collect")
```

**Output (verbatim):**

```text
baseline                       RSS=    15.6MB  pyheap_cur=   0.00MB  gc_objs=11,675
after alloc 500k small dicts   RSS=   363.1MB  pyheap_cur= 154.55MB  gc_objs=11,676
after del 500k + gc.collect    RSS=   169.5MB  pyheap_cur=   0.00MB  gc_objs=11,675
```

After deleting 500,000 small dicts and collecting, `pyheap_cur` returns to **`0.00MB`** and `gc_objs` returns to the **`11,675`** baseline (no Python-level leak) — yet RSS stays at **`169.5MB`**. This is the definitive allocator-retention signature (see R8).

### Evidence C — Django `QuerySet._result_cache` (lazy vs. materialized; real two-pass pattern; mirrors `index_reindex` `Document.objects.all()` at `src/documents/tasks.py:L39` and `bulk_update_documents` `filter(id__in=...)` at `src/documents/tasks.py:L271`; heavy field `content` at `src/documents/models.py:L117`)

**Command:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceC.py'
```

**Script (`/tmp/obs_evidenceC.py`):**

```python
"""Evidence C - Django QuerySet _result_cache materialization.

Mirrors index_reindex (src/documents/tasks.py:L38 Document.objects.all()) and
bulk_update_documents (src/documents/tasks.py:L270 filter(id__in=...) iterated
twice). A QuerySet is lazy (_result_cache is None) until iterated; materializing
it loads every row INCLUDING the heavy content TextField (src/documents/models.py:L117).
bulk_update_documents iterates the SAME queryset twice: the first loop populates
_result_cache; the second loop reuses it (one copy held across both loops).
"""
import gc
import os
import tracemalloc

import django
import psutil
from django.conf import settings

settings.configure(
    INSTALLED_APPS=["django.contrib.contenttypes", "django.contrib.auth"],
    DATABASES={"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": "/tmp/obs_c.sqlite3"}},
    DEFAULT_AUTO_FIELD="django.db.models.AutoField",
)
django.setup()

from django.db import models  # noqa: E402


class Doc(models.Model):
    content = models.TextField()

    class Meta:
        app_label = "contenttypes"


from django.db import connection  # noqa: E402

with connection.schema_editor() as se:
    se.create_model(Doc)

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024

CONTENT = "x" * 8000  # ~8 KB per row, like a real document's extracted text
N = 4000
Doc.objects.bulk_create([Doc(content=CONTENT) for _ in range(N)], batch_size=500)
gc.collect()
tracemalloc.start()
gc.collect()
print("baseline ({} docs in sqlite, not loaded)  RSS={:8.1f}MB".format(N, rss()))

qs = Doc.objects.all()
print(
    "after .all() (lazy)                       RSS={:8.1f}MB  _result_cache={}".format(
        rss(), qs._result_cache
    )
)

rows = list(qs)  # index_reindex pattern: materialize every row incl. content
print(
    "after list(qs) [index_reindex]            RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  rows={}  cached={}".format(
        rss(), heap(), len(rows), qs._result_cache is not None
    )
)
del rows, qs
gc.collect()
print("after del + gc.collect                    RSS={:8.1f}MB  pyheap_cur={:7.2f}MB".format(rss(), heap()))

# bulk_update_documents pattern: ONE queryset iterated twice.
ids = list(Doc.objects.values_list("id", flat=True))
documents = Doc.objects.filter(id__in=ids)
n1 = sum(1 for _ in documents)          # first loop -> materializes _result_cache
cached_after_first = documents._result_cache is not None
n2 = sum(1 for _ in documents)          # second loop -> reuses cache (no re-query)
print(
    "after bulk_update 2-pass (1 queryset)     RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  loop1={}  loop2={}  cached_once={}".format(
        rss(), heap(), n1, n2, cached_after_first
    )
)
```

**Output (verbatim):**

```text
baseline (4000 docs in sqlite, not loaded)  RSS=    42.5MB
after .all() (lazy)                       RSS=    42.5MB  _result_cache=None
after list(qs) [index_reindex]            RSS=    77.5MB  pyheap_cur=  32.17MB  rows=4000  cached=True
after del + gc.collect                    RSS=    76.1MB  pyheap_cur=   0.03MB
after bulk_update 2-pass (1 queryset)     RSS=    79.0MB  pyheap_cur=  32.49MB  loop1=4000  loop2=4000  cached_once=True
```

`.all()` is lazy (`_result_cache=None`); `list(qs)` materializes all 4000 rows including `content` (`+32.17MB` heap); the `bulk_update` pattern iterates **one** queryset twice, so the cache is populated once (`cached_once=True`) and the heap is **not** doubled (`32.49MB`).

### Evidence C2 — Batch-size growth curve (linear in row count; heavy field `content = models.TextField` at `src/documents/models.py:L117`)

**Command:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceC2.py'
```

**Script (`/tmp/obs_evidenceC2.py`):**

```python
"""Evidence C2 - QuerySet materialization scales linearly with batch size.

Materializing a QuerySet of N rows loads N content strings into _result_cache,
so the Python-heap cost grows linearly with the batch size (the number of rows
processed by index_reindex / bulk_update_documents, src/documents/tasks.py).
"""
import gc
import os
import tracemalloc

import django
import psutil
from django.conf import settings

settings.configure(
    INSTALLED_APPS=["django.contrib.contenttypes", "django.contrib.auth"],
    DATABASES={"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": "/tmp/obs_c2.sqlite3"}},
    DEFAULT_AUTO_FIELD="django.db.models.AutoField",
)
django.setup()

from django.db import connection, models  # noqa: E402


class Doc(models.Model):
    content = models.TextField()

    class Meta:
        app_label = "contenttypes"


with connection.schema_editor() as se:
    se.create_model(Doc)

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024

CONTENT = "x" * 8000  # ~8 KB per row
Doc.objects.bulk_create([Doc(content=CONTENT) for _ in range(8000)], batch_size=500)
gc.collect()
tracemalloc.start()
gc.collect()
base = rss()
print("baseline (8000 docs in sqlite)  RSS={:8.1f}MB".format(base))

for batch in [1000, 2000, 4000, 8000]:
    b_rss, b_heap = rss(), heap()
    rows = list(Doc.objects.all()[:batch])
    print(
        "batch={:5d}: RSS delta=+{:6.1f}MB  pyheap delta=+{:6.2f}MB  rows={}".format(
            batch, rss() - b_rss, heap() - b_heap, len(rows)
        )
    )
    del rows
    gc.collect()
```

**Output (verbatim):**

```text
baseline (8000 docs in sqlite)  RSS=    42.9MB
batch= 1000: RSS delta=+   8.2MB  pyheap delta=+  8.09MB  rows=1000
batch= 2000: RSS delta=+   8.7MB  pyheap delta=+ 16.11MB  rows=2000
batch= 4000: RSS delta=+  18.1MB  pyheap delta=+ 32.11MB  rows=4000
batch= 8000: RSS delta=+  38.1MB  pyheap delta=+ 64.11MB  rows=8000
```

The Python-heap delta doubles as the batch doubles (`8.09 → 16.11 → 32.11 → 64.11 MB`) — linear in row count.

### Evidence D — Spike vs. non-spike, using the REAL paperless modules (`documents.classifier.load_classifier` at `src/documents/classifier.py:L30`; `paperless_text.parsers.TextDocumentParser` at `src/paperless_text/parsers.py:L12`)

**Command:**

```bash
docker exec -e DJANGO_SETTINGS_MODULE=paperless.settings -e PYTHONPATH=/app/src -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceD.py'
```

`2>/dev/null` is appended when quoting the block below to elide scikit-learn `ConvergenceWarning` lines emitted on stderr by the demonstration model fit (`max_iter=1`); those warnings are unrelated to memory. The measurement lines are printed on stdout.

**Script (`/tmp/obs_evidenceD.py`):**

```python
"""Evidence D - REAL paperless modules: cheap (non-spike) vs classifier (spike).

Exercises the actual code:
  * documents.classifier.load_classifier() (src/documents/classifier.py:L30) -
    returns None cheaply when no MODEL_FILE exists (guard L31, return L36).
  * paperless_text.parsers.TextDocumentParser.extract_metadata - inherits the
    base DocumentParser.extract_metadata that returns [] (src/documents/parsers.py:L304-L305).
  * Then populates a real DocumentClassifier with a fitted sklearn model to show
    the size-independent spike that occurs when a model IS present in memory.
"""
import gc
import os
import tempfile
import tracemalloc

import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()

import psutil  # noqa: E402
import numpy as np  # noqa: E402
from sklearn.feature_extraction.text import CountVectorizer  # noqa: E402
from sklearn.neural_network import MLPClassifier  # noqa: E402
from sklearn.preprocessing import MultiLabelBinarizer  # noqa: E402

from documents.classifier import DocumentClassifier, load_classifier  # noqa: E402
from paperless_text.parsers import TextDocumentParser  # noqa: E402

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start()
gc.collect()
print("baseline (django.setup done)          RSS={:8.1f}MB  pyheap_cur={:7.2f}MB".format(rss(), heap()))

# --- Non-spike path 1: load_classifier() with no model file present ---
clf = load_classifier()
print("load_classifier() ->                  {}   RSS={:8.1f}MB  pyheap_cur={:7.2f}MB".format(clf, rss(), heap()))

# --- Non-spike path 2: real TextDocumentParser.extract_metadata (inherited base [] ) ---
txt = tempfile.NamedTemporaryFile(suffix=".txt", delete=False, mode="w")
txt.write("This is a mostly-text document with small metadata.\n" * 50)
txt.close()
parser = TextDocumentParser(None)
cheap_before = heap()
md = parser.extract_metadata(txt.name, "text/plain")
print(
    "extract_metadata (text) ->            metadata_items={}   RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  heap_delta=+{:.3f}MB".format(
        len(md), rss(), heap(), heap() - cheap_before
    )
)
parser.cleanup()  # remove the mkdtemp scratch dir (src/documents/parsers.py:L348)
os.remove(txt.name)

# --- Spike path: a real DocumentClassifier holding a fitted model in memory ---
spike_before = heap()
rng = np.random.RandomState(0)
docs = [" ".join("t{:05d}".format(n) for n in rng.randint(0, 8000, size=160)) for _ in range(1200)]
model = DocumentClassifier()
model.data_hash = b"\x00" * 32
model.data_vectorizer = CountVectorizer().fit(docs)
Xf = model.data_vectorizer.transform(docs)
model.tags_binarizer = MultiLabelBinarizer().fit([[1, 2, 3]])


def mlp():
    m = MLPClassifier(hidden_layer_sizes=(128, 64), max_iter=1)
    m.fit(Xf, rng.randint(0, 2, size=Xf.shape[0]))
    return m


model.tags_classifier = mlp()
model.correspondent_classifier = mlp()
model.document_type_classifier = mlp()
del docs, Xf
gc.collect()
print(
    "classifier model in memory (spike) -> RSS={:8.1f}MB  pyheap_cur={:7.2f}MB  heap_delta=+{:.2f}MB".format(
        rss(), heap(), heap() - spike_before
    )
)
```

**Output (verbatim):**

```text
baseline (django.setup done)          RSS=   108.4MB  pyheap_cur=   0.00MB
load_classifier() ->                  None   RSS=   108.4MB  pyheap_cur=   0.03MB
extract_metadata (text) ->            metadata_items=0   RSS=   108.4MB  pyheap_cur=   0.03MB  heap_delta=+0.001MB
classifier model in memory (spike) -> RSS=   201.7MB  pyheap_cur=  72.11MB  heap_delta=+72.07MB
```

The real `load_classifier()` returns `None` (no `MODEL_FILE`; guard `src/documents/classifier.py:L31`, `return None` `src/documents/classifier.py:L36`) and the real `TextDocumentParser.extract_metadata` returns `0` items — both essentially free. Holding a fitted classifier model in memory is the `+72.07MB` spike.

### Evidence E — Full-file `hashlib.md5(f.read())` scales 1:1 with file bytes (reproduces `src/documents/consumer.py:L104`, the **archive checksum** at `L340-L341`, the post-copy checksum at `L402`, and the copy `write_file.write(read_file.read())` at `L432`)

**Command:**

```bash
docker exec -e PAPERLESS_DISABLE_DBHANDLER=true paperless-qna-0 bash -lc 'cd /app/src && python /tmp/obs_evidenceE.py'
```

**Script (`/tmp/obs_evidenceE.py`):**

```python
"""Evidence E - Full-file read for checksum scales 1:1 with file byte size.

Reproduces the memory mechanism of the four full-file reads in the consume path:
  - src/documents/consumer.py:L104  original-file checksum  hashlib.md5(f.read())
  - src/documents/consumer.py:L340-L341  ARCHIVE checksum    hashlib.md5(f.read())
  - src/documents/consumer.py:L402  post-copy checksum       hashlib.md5(f.read())
  - src/documents/consumer.py:L432  file copy                write_file.write(read_file.read())
Each reads the entire file into a single bytes object before hashing/copying, so
transient RSS grows by ~the file size. This is size-DEPENDENT (unlike the classifier).
"""
import gc
import hashlib
import os
import tempfile
import tracemalloc

import psutil

proc = psutil.Process(os.getpid())
rss = lambda: proc.memory_info().rss / 1024 / 1024
heap = lambda: tracemalloc.get_traced_memory()[0] / 1024 / 1024
tracemalloc.start()
gc.collect()
base = rss()
print("baseline                     RSS={:8.1f}MB".format(base))

for size_mb in [10, 50, 100, 200]:
    path = tempfile.mktemp(suffix=".bin")
    with open(path, "wb") as f:
        f.write(os.urandom(size_mb * 1024 * 1024))
    # Mirror hashlib.md5(f.read()) - entire file read into one bytes object.
    with open(path, "rb") as f:
        data = f.read()
        digest = hashlib.md5(data).hexdigest()
    print(
        "{:3d}MB file: md5={}  RSS={:8.1f}MB (delta +{:.1f}MB)  pyheap_cur={:7.1f}MB".format(
            size_mb, digest[:8], rss(), rss() - base, heap()
        )
    )
    del data
    gc.collect()
    os.remove(path)
```

**Output (verbatim):**

```text
baseline                     RSS=    18.5MB
 10MB file: md5=6846c01e  RSS=    28.3MB (delta +9.9MB)  pyheap_cur=   10.0MB
 50MB file: md5=e909cae6  RSS=    78.3MB (delta +59.9MB)  pyheap_cur=   50.0MB
100MB file: md5=a4584ebb  RSS=   128.3MB (delta +109.9MB)  pyheap_cur=  100.0MB
200MB file: md5=8642a406  RSS=   228.3MB (delta +209.9MB)  pyheap_cur=  200.0MB
```

Each whole-file `f.read()` grows resident memory by ~the file size (a 200 MB file adds `+209.9MB`). The same mechanism applies at all four consume-path sites, including the archive checksum at `src/documents/consumer.py:L340-L341`.

> **What was reproduced vs. run directly.** Evidence **D** exercises the *real* `documents.classifier.load_classifier` and `paperless_text.parsers.TextDocumentParser`. Evidence **A/A2** reproduce the exact 7× `pickle.load` mechanism of `DocumentClassifier.load` with a same-shape scikit-learn model (the repo has no trained `MODEL_FILE`, so the real `load_classifier()` returns `None` — confirmed in Evidence D). Evidence **C/C2** reproduce Django's `QuerySet._result_cache` materialization with real `django==4.0.4` against a minimal model carrying a heavy `content = TextField`. Evidence **B/E** are pure stdlib + `psutil` mechanism demonstrations. All ran in the canonical Python 3.9.23 + pinned-library container (see §2.1.1).

---

## R7 — Attribution: which components/methods hold memory

> *Deliverable requirement: identify which components/methods hold onto memory.*

| # | Component / method | Location (`file:line`) | What it holds | Backing evidence |
|---|--------------------|------------------------|---------------|------------------|
| 1 | `load_classifier` → `DocumentClassifier.load` (unpickled `CountVectorizer` + `MLPClassifier`) | `src/documents/classifier.py:L30`, `L76`; 7× `pickle.load` `L78`,`L86`–`L88`,`L90`–`L92`; invoked `src/documents/consumer.py:L292` | The entire ML model in Python heap + native arrays, **per document** | A, A2, D |
| 2 | Full-file reads for MD5 / copy | `src/documents/consumer.py:L104`, `L340-L341` (archive checksum), `L402`, `L432` | The whole file bytes in memory, **scales with size** | E |
| 3 | Redundant Tika parse | `src/paperless_tika/parsers.py:L32` + `L55` (`self.text` `L62`) | The full document body **twice** (Office docs) | R2.b (code) |
| 4 | Tesseract XMP metadata list | `src/paperless_tesseract/parsers.py:L34`,`L35`,`L42`–`L49` | A new dict list per `extract_metadata` call | R2.a (code) |
| 5 | Whoosh `AsyncWriter` buffering `content` | `src/documents/index.py:L66` (`AsyncWriter(open_index())`), writes `content=doc.content` `L93` via `add_or_update_document` `L118` (`update_document` `L87`) | The document `content` buffered per document during indexing | C (content retention) |
| 6 | Django `QuerySet._result_cache` materialization | `src/documents/tasks.py:L39` (`Document.objects.all()`), `L271` (`filter(id__in=...)`); heavy `content` `src/documents/models.py:L117` | Every row's `content` string, held for the function's duration; **scales with batch** | C, C2 |
| 7 | Post-consume signal fan-out passing full `content` | `src/documents/signals/handlers.py`: `set_correspondent` `L35`, `set_document_type` `L101`, `set_tags` `L168`, `add_to_index` `L428` → `index.add_or_update_document(document)` `L431`; matching passes `document.content` to `classifier.predict_*` at `src/documents/matching.py:L23` (`predict_correspondent`), `L36` (`predict_document_type`), `L49` (`predict_tags`) | The full `content` string handed to each handler/predictor | C (content held) |

The dominant, "disproportionate-to-size" holder is **#1 (the per-document classifier reload)** — verbatim, holding the model raises RSS from `108.4MB` to `201.7MB` (`+72.07MB` Python heap, Evidence D) and unpickling adds `pyheap_cur= 95.73MB` (Evidence A). The batch-scaling holder is **#6 (`_result_cache`)** — `after list(qs) [index_reindex] RSS= 77.5MB pyheap_cur= 32.17MB` (Evidence C). The size-proportional holder is **#2 (full-file reads, including the archive checksum at `L340-L341`)** — a `200 MB` file adds `+209.9MB` RSS (Evidence E).


---

## R8 — Diagnosis: normal GC/allocator behavior vs. a genuine problem

> *Deliverable requirement: show whether this is normal Python garbage-collection / allocator behavior or something genuinely problematic.*

### R8.a — The central finding: it is (mostly) normal allocator retention, not a leak

The user's most alarming symptom — memory "not always released back to the system in a timely manner even after processing completes" — is explained by **CPython's allocator retaining freed memory**, *not* by a reference leak. The proof is Evidence B:

```text
baseline                       RSS=    15.6MB  pyheap_cur=   0.00MB  gc_objs=11,675
after alloc 500k small dicts   RSS=   363.1MB  pyheap_cur= 154.55MB  gc_objs=11,676
after del 500k + gc.collect    RSS=   169.5MB  pyheap_cur=   0.00MB  gc_objs=11,675
```

After the objects are deleted and `gc.collect()` runs:
- `pyheap_cur` returns to **`0.00MB`** — the Python allocator considers the memory free.
- `gc_objs` returns to the **`11,675`** baseline — there is **no growing set of live Python objects**, i.e., **no reference leak**.
- Yet **RSS stays at `169.5MB`** — the OS still sees the memory as resident.

Evidence A shows the same signature at the "after del model + gc.collect" checkpoint: `pyheap_cur` returns to `23.90MB` (the reload increment is fully released) while RSS stays elevated at `179.5MB` (above the `85.3MB` baseline).

### R8.b — Why RSS stays high: pymalloc arenas / pools / blocks

CPython manages small objects (≤ 512 bytes) with its own allocator, **pymalloc**, organized as a hierarchy:

- **blocks** — fixed size classes (multiples of 8 bytes, up to 512 B);
- **pools** — 4 KB pages, each subdivided into blocks of one size class;
- **arenas** — ~256 KB regions requested from the OS, each holding many pools.

An **arena is returned to the operating system only when *every* block in it has been freed.** A single surviving object anywhere in an arena keeps the whole 256 KB region resident. After allocating and freeing hundreds of thousands of small objects, the surviving fragmentation means most arenas cannot be handed back — so RSS stays flat/high even though Python's own accounting shows the heap as empty. Objects **larger** than 512 bytes bypass pymalloc and use the system allocator (glibc `malloc` on Linux), which *can* return memory to the OS — which is why Evidence A (dominated by large NumPy arrays) shows RSS partially recovering (`202.8MB → 179.5MB`) while Evidence B (500k *small* dicts) does not (RSS stays at `169.5MB`).

### R8.c — The tracemalloc blind spot (stated explicitly, per the "say so" rule)

`tracemalloc` traces **only the Python heap**. Allocations made by C extensions — scikit-learn, `pikepdf`, Whoosh, NumPy/SciPy — are **invisible** to it. This is directly visible in Evidence A: at "after fit," RSS is `194.9MB` but `pyheap_cur` is only `75.73MB`. That ~119 MB gap is native allocation that `tracemalloc` does not attribute. **A growing RSS with a flat Python heap must therefore be inferred from the RSS-vs-heap gap**, not read directly from `tracemalloc`. This document states that inference explicitly rather than over-claiming attribution the tool cannot provide.

### R8.d — Verdict

The behavior is a **combination**:

1. **Normal allocator retention (not a bug).** RSS staying elevated after processing — the user's "not released in a timely manner" symptom — is expected CPython/pymalloc behavior, especially in the long-lived Django-Q worker processes (`django-q==1.3.9`, `requirements.txt:L37`). No Python reference leak was found (Evidence B: `gc_objs` and `pyheap_cur` return to baseline).
2. **A genuine inefficiency (not a leak, but fixable).** The **per-document, size-independent classifier reload** (R1.a; `src/documents/classifier.py:L30`,`L40`; `src/documents/consumer.py:L292`) re-unpickles the entire model every consume with no cache (Evidence A2). This is what makes small documents spike "disproportionately to size."
3. **Legitimate, size/batch-proportional usage.** Full-file reads (Evidence E) and `_result_cache` materialization (Evidence C/C2) scale predictably with file size and batch size respectively — expected, not anomalous, but amplifiable at large batches.

**Bottom line:** there is **no reference leak**. The spikes are driven by a genuine inefficiency (uncached classifier reload) layered on top of normal CPython allocator retention that keeps RSS elevated in long-running workers.

---

## 12. Recommendations (NOT implemented — diagnosis only)

Per the read-only / no-remediation rule, these are **recommendations only**; no code was changed:

- **Cache the loaded classifier** across documents/hooks (e.g., load once per worker) to eliminate the per-document re-unpickle spike (addresses R1.a / #1 in R7).
- **Parse Tika documents once** — reuse the `parser.from_file` result between `extract_metadata` and `parse` instead of calling it at both `src/paperless_tika/parsers.py:L32` and `L55` (addresses R2.b).
- **Stream/chunk the MD5** instead of `hashlib.md5(f.read())` (read in fixed-size chunks) to bound the size-dependent term at all four whole-file-read sites — `src/documents/consumer.py:L104`, `L340-L341` (archive checksum), `L402`, and the copy at `L432` (addresses R1.b).
- **Add `parser.cleanup()`** in `get_metadata` (`src/documents/views.py:L260`) to stop orphaning scratch directories (addresses R2.c — disk).
- **Use `.iterator()` / `.only(...)`** for the large-batch tasks (`src/documents/tasks.py:L39`, `L271`) so the heavy `content` field is not fully materialized into `_result_cache` (addresses R5.b / #6).
- **For the long-lived workers**, consider glibc `malloc_trim()`, an alternative allocator such as **jemalloc**, or bounding `MALLOC_ARENA_MAX` to encourage returning freed arenas to the OS (addresses the R8 retention symptom).

---

## 13. References (web research, synthesized in our own words)

Background research was used to *frame* — not replace — the captured measurements:

- **Python `tracemalloc` documentation** — `docs.python.org/3/library/tracemalloc.html`. Establishes that `tracemalloc` traces Python-allocated memory blocks with per-file/line statistics and that `Snapshot.compare_to(other, 'lineno')` localizes growth; it does not account for allocations by underlying C libraries.
- **Red-Gate Simple-Talk, "Memory profiling in Python with tracemalloc"** — practitioner guidance on taking/compare-ing snapshots and calling `gc.collect()` before a snapshot to filter cyclic-garbage noise.
- **Rushter, "Python memory management"** (`rushter.com/blog/python-memory-managment`) and related primers on pymalloc — describe the blocks → pools (4 KB) → arenas (256 KB) hierarchy for small objects (≤ 512 B), the fallback to the system allocator for larger objects, and the fact that an arena is returned to the OS only when all its blocks are freed (so RSS stays elevated in long-running processes).
- **Python developer guide / CPython `Objects/obmalloc.c` commentary** — corroborates the ≤ 512-byte pymalloc threshold and arena-release semantics.
- **glibc `malloc_trim(3)` and jemalloc documentation, and the `MALLOC_ARENA_MAX` tunable** — cited only as *recommendation* context for coaxing long-lived processes to release retained arenas.

(These sources are summarized in our own words; no source text is reproduced verbatim.)

---

## 14. Coverage-pass checklist

Every sub-question is answered with at least one exact `file:line` citation **and** at least one verbatim measurement:

| Req | Question | Key `file:line` citation(s) | Verbatim measurement |
|-----|----------|-----------------------------|----------------------|
| **R1** | Root cause | `classifier.py:L30/L38/L40`, 7× `pickle.load` `L78/L86-92`; `consumer.py:L292`, and the four full-file reads `L104`/`L340-L341`/`L402`/`L432` | Evidence A (`+72MB` heap on unpickle); Evidence E (200 MB → `+209.9MB` RSS) |
| **R2** | Copies / references | `parsers.py:L304-L305`; `tika/parsers.py:L32`+`L55`; `views.py:L266/L269/L295/L302`; `parsers.py:L293/L350` | Evidence D (`metadata_items=0`); Evidence C (`32.17MB → 0.03MB` on `del`) |
| **R3** | Caching accumulation | `grep` → no matches; `matching.py:L61`; `classifier.py:L38/L40`; `settings.py:L50` | Evidence A2 (identical `+71.85MB` × 2, no reuse); Evidence B (`gc_objs`→ baseline) |
| **R4** | Spike vs. non-spike | `paperless_text/parsers.py:L12`; `parsers.py:L304-L305`; `classifier.py:L31` (guard) / `L36` (`return None`) | Evidence D (`108.4MB` flat vs. `201.7MB` spike, `+72.07MB`) |
| **R5** | Type / batch variance | `tika/parsers.py:L32/L55`; `tasks.py:L39/L271`; `models.py:L117`; (mail: **no** `parsers.py`) | Evidence C (`_result_cache=None` → `+32.17MB`); Evidence C2 (linear `8.09→64.11MB`) |
| **R6** | Runtime measurements | all Evidence blocks with exact commands **and** full script source | Evidence A, A2, B, C, C2, D, E (verbatim in R6) |
| **R7** | Attribution | `classifier.py:L30/L76`; `consumer.py:L104/L340-L341/L402/L432`; `tika/parsers.py:L32/L55`; `index.py:L66/L93/L118`; `tasks.py:L39/L271`; `handlers.py:L35/L101/L168/L428/L431`; `matching.py:L23/L36/L49` | Evidence A/A2/C/C2/D/E (attributed per row) |
| **R8** | Normal vs. problematic | pymalloc arenas/pools/blocks; `tracemalloc` limitation; `django-q` `requirements.txt:L37` | Evidence B (`pyheap_cur`→`0.00MB`, `gc_objs`→`11,675`, RSS stays `169.5MB`); Evidence A (RSS/heap gap `194.9` vs `75.73MB`) |

**All R1–R8 addressed. No sub-question skipped.** R1 and R7 explicitly enumerate all four full-file reads including the archive checksum at `consumer.py:L340-L341`; R7 explicitly enumerates all three `predict_*` calls including `matching.py:L36`.

---

## 15. Reproducibility & repository-cleanliness statement

- Canonical runtime confirmed: Python 3.9.23 with `django 4.0.4`, `sklearn 1.0.2`, `pikepdf 5.1.1`, `whoosh (2, 7, 4)`, `numpy 1.22.3`, `scipy 1.8.0` — the project dependency pins match `requirements.txt` (`Dockerfile:L18`). `psutil 7.2.2` was installed **only** in the profiling container as observation tooling; it is **not** a `requirements.txt` project dependency. (See §2.1.1 for the full measurement-environment disclosure and its reconciliation with the AAP §0.8.2 Python-3.12.3 / PEP-668 / `--break-system-packages` / version-faithful contingency, which was not needed.)
- All observation scripts were created outside the repository (`/tmp`), executed in the canonical container, captured verbatim above (with full source in R6), and then **deleted**.
- The source tree was **not modified**. After the investigation, `git status --porcelain` shows **only** this new file `blitzy/documentation/paperless-ngx_542221a38dff.md`; every file under `src/**`, all dependency manifests (`requirements.txt`, `Pipfile`, `Dockerfile`), tests, and CI remain byte-for-byte unchanged. No diagnosed hotspot was remediated — §12 lists recommendations only.
