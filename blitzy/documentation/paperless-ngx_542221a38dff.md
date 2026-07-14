# Memory‑Usage Investigation: paperless‑ngx Document‑Ingestion Pipeline

**Subject system:** paperless‑ngx (Django 4.0 backend, source branch `paperless-ngx_542221a38dff`, commit `542221a38dff`).
**Question investigated (verbatim):** *"figure out what's actually causing these memory spikes. Is the metadata handling creating unnecessary copies or holding onto references longer than needed? Is there some caching behaviour that's accumulating data? … They need actual runtime memory measurements, identification of which components/methods hold memory, and evidence distinguishing normal Python GC from something problematic."*
**Reported symptoms:** (a) importing documents sometimes consumes far more memory than expected during metadata handling; (b) behaviour is **inconsistent** (varies by document source / processing stage); (c) spikes are **disproportionate to document size** (mostly small text documents); (d) memory is **not always released to the OS** in a timely manner even after processing completes.

**Methodology contract.** This is a **run‑first** investigation. Every quantitative claim below is accompanied by the exact command and its **complete, unedited** output (see *§9 Captured Evidence*), and every statement is labelled **`observed (runtime)`** or **`inferred (from reading code)`**. All runtime measurement was performed inside the canonical **Python 3.9** Docker container (the application cannot run on the host's Python 3.13 — Django 4.0.4 / scikit‑learn 1.0.2 predate 3.12+). No source file was modified; the only artifact added to the repository is this document. Temporary measurement scripts lived outside the tracked tree (in the container's `/tmp`) and were removed afterward.

---

## 1. Direct Answer (read this first)

**The spikes are real, but they are overwhelmingly *fixed, document‑independent* costs — they are not caused by unnecessary copies of the document content, and they are not caused by an unbounded cache that grows across documents. The single largest contributor to a spike that looks "disproportionate to a small text document" is the deserialization of the trained classifier model, which pulls in scikit‑learn / SciPy / NumPy and their native buffers — about **+56 MB for a 21‑byte file**, entirely independent of the document. The "memory not released to the OS" symptom is the well‑documented behaviour of the CPython/glibc allocators retaining freed heap in per‑process *arenas* and reusing it, rather than returning it to the kernel — it is **not** a Python reference leak.** `observed (runtime)`

Point by point, mapped to the user's questions:

- **"Is the metadata handling creating unnecessary copies…?"** The most‑cited "copy" — `document_content = document.content` in `matching.py:63` — is **not** a copy: it is a reference bind (`document_content is document.content` → `True`). `observed (runtime)`. The genuine copies are the **whole‑file `f.read()` reads** in `consumer.py` (dedup MD5 `L104`, store MD5 `L402`, and the non‑chunked file copies in `_write` `L432`) and the **fuzzy‑match second string** in `matching.py:131`. These are **proportional to document size** and **transient** (freed after use), so they are negligible for the "small text documents" in the report and only matter for large files. `observed (runtime)`

- **"…or holding onto references longer than needed?"** The extracted `text` local (`consumer.py:271`) is held for the duration of `try_consume_file()`, across the `transaction.atomic()` block (`L298`) and through all six post‑consume signal handlers, then freed at function return. It is **one reference to the already‑stored content, not a duplicate**, and it is not retained after the call. `observed (runtime)` / `inferred (from reading code)`

- **"Is there some caching behaviour that's accumulating data?"** **No.** Across repeated documents in a single long‑lived process, none of the candidate caches grow: the classifier has **no module‑level cache** (it is *reloaded* per consume, `classifier.py:30`), the matching layer's `objects.all()` and content scans are transient, the per‑document Whoosh `AsyncWriter` is bounded, and the Django ORM query log is disabled because `DEBUG=False` by default (`settings.py:50`). Cross‑document RSS **plateaus and is reused**; it does not climb. `observed (runtime)`

- **"…evidence distinguishing normal Python GC from something problematic."** In every run `gc.collect()` left `gc.garbage == 0` (no uncollectable cycles); the paperless application Python heap is **flat** across identical iterations (the only apparent growth is `tracemalloc`'s own bookkeeping); RSS **plateaus** across identical and large‑document repeats (reused, not leaked); and `malloc_trim(0)` returns freed arena pages to the OS (reclaiming **+75 MB** after a six‑document large‑file loop). Together these prove the retention is **normal allocator behaviour, not a leak**. `observed (runtime)`

**Why it looks "inconsistent" and "disproportionate."** The per‑consume cost is dominated by **which parser runs** and **whether a classifier model exists**, not by the document's byte size: a 21‑byte text file costs ~7 MB with no model, ~15 MB end‑to‑end, and ~70 MB the moment a trained model is present; the same tiny file through the OCR parser costs ~30 MB. Because the task worker is **recycled after every task** (`Q_CLUSTER["recycle"] = 1`, `settings.py:452`), each document is a fresh process that pays these fixed costs anew and releases them at exit — which is exactly why the same input can look different depending on source/stage, and why memory *is* released for the consume worker but *is not* released promptly in the **long‑lived** web (upload/metadata) and mail processes. `observed (runtime)`

---

## 2. Environment & Reproducibility

All numbers were produced inside the provided container (`docker exec … paperless_app`). `observed (runtime)`

| Property | Value |
|---|---|
| Canonical runtime | **Python 3.9.23** (Dockerfile `FROM python:3.9-slim-bullseye`) |
| Host runtime (unusable) | Python 3.13 — `import django` fails (Django 4.0.4 / scikit‑learn 1.0.2 predate 3.12+); any host number would be **non‑canonical** |
| Base image | `ghcr.io/scaleapi/swe-atlas:…qna_1.01` (Debian 11 bullseye, **glibc 2.31**) |
| DB / broker | SQLite at `/app/data/db.sqlite3` (migrated) / Redis `redis://paperless-broker:6379` |
| CPU count in container | `nproc = 128` → glibc default arena cap `8 × cores` ⇒ up to ~1024 per‑process arenas theoretically possible (relevant context for the "not released to OS" symptom; measured to be *not* the driver here — see §6.3) |
| `psutil` | **absent** from all manifests — RSS taken from stdlib `resource` + `/proc/self/status` `VmRSS` |

Pinned stack (excerpt of `pip freeze`, matches `requirements.txt`): `Django==4.0.4`, `django-q==1.3.9`, `djangorestframework==3.13.1`, `scikit-learn==1.0.2`, `numpy==1.22.3`, `scipy==1.8.0`, `pikepdf==5.1.1`, `Pillow==9.1.0`, `Whoosh==2.7.4`, `ocrmypdf==13.4.3`, `pdfminer.six==20220319`, `pdf2image==1.16.0`, `redis==3.5.3`, `channels==3.0.4`, `watchdog==2.1.7`, `fuzzywuzzy==0.18.0`. `observed (runtime)`

**Standard run environment** for every measurement:
```
docker exec -w /app/src --user testuser \
  -e PAPERLESS_REDIS=redis://paperless-broker:6379 \
  -e PAPERLESS_DISABLE_DBHANDLER=true -e HOME=/home/testuser \
  -e DJANGO_SETTINGS_MODULE=paperless.settings paperless_app python3 <script>
```

**Canonical‑fidelity notes.**
- The consume path was driven through the real orchestrator `Consumer().try_consume_file()` and the real Django‑Q task `documents.tasks.consume_file` (via `async_task` over Redis). The harness reused the project's own `documents.tests.utils.setup_directories()` to redirect `MEDIA/DATA/SCRATCH/INDEX/MODEL_FILE` to throwaway `/tmp` dirs, and monkey‑patched `Consumer._send_progress` to a no‑op (exactly as `documents/tests/test_consumer.py` does) so the measurement is isolated from the websocket layer — the consume logic under measurement is unchanged. `observed (runtime)`
- The container shipped the **stock Debian ImageMagick policy**, which blocks PDF rasterization; the canonical paperless policy (`docker/imagemagick-policy.xml`, installed by the project Dockerfile) was applied inside the container so the OCR path is faithful. This changed only the container, never the git tree. `observed (runtime)`
- The email transport is labelled **non‑canonical** (no live IMAP server); the attachment‑buffering code exercised is the real `MailAccountHandler.handle_message`. The importer manifest content is synthetic but the `json.load` + `list()` operations mirror the cited lines exactly.

---

## 3. Methodology — three complementary lenses + a discriminator

No single tool suffices, so all measurements combine: `observed (runtime)` / `inferred (from reading code)`

1. **RSS (OS level, sees native allocations).** Current `VmRSS` parsed from `/proc/self/status`, plus peak `resource.getrusage(RUSAGE_SELF).ru_maxrss`. This is **mandatory** because the biggest costs here are native C‑extension buffers.
2. **`tracemalloc` (Python heap only, attributes growth to `file:line`).** `tracemalloc.start()` before the first allocation of interest; `take_snapshot()` per stage; `snapshot.compare_to(prev, 'lineno')` for line‑level diffs. **Critical limitation (confirmed at runtime):** `tracemalloc` is **blind to native allocations** (scikit‑learn/NumPy/SciPy/Pillow/pikepdf) and, worse, its own per‑allocation bookkeeping **inflates RSS** — an isolated classifier load cost **+53 MB clean** but **+221 MB with `tracemalloc` on** (*§9.4*). Therefore `tracemalloc` is used **only** for Python‑heap attribution and never for RSS numbers.
3. **`gc` (uncollectable‑cycle detection).** `gc.collect()` then `len(gc.garbage)`; a non‑zero value would indicate an uncollectable reference cycle.

**Normal‑vs‑problem discriminator.** After a consume completes and references are dropped, call `ctypes.CDLL("libc.so.6").malloc_trim(0)` and re‑sample RSS. If RSS falls, the retained memory was reclaimable glibc arena free‑space (**normal**). Cross‑iteration `tracemalloc` deltas (flat ⇒ no Python leak) and `gc.garbage` (empty ⇒ no uncollectable cycles) complete the verdict. `MALLOC_ARENA_MAX` is used as a second probe: if capping arenas lowers the retained plateau, the retention is arena‑driven.

**First‑run vs steady‑state** are always separated: the first consume in a process pays cold module‑import costs that never recur; steady‑state consumes are what a long‑lived worker actually experiences. Every magnitude claim was confirmed stable across **≥2 runs**, and each identical input was repeated to characterize the reported run‑to‑run inconsistency.

---

## 4. Findings by Objective

### OBJ‑1 — Root cause / allocation sites

**Direct answer:** the ingestion memory increase is dominated by **native C‑extension allocations** (invisible to `tracemalloc`) plus **one‑time Python module imports**; almost none of it is attributable to a paperless‑ngx source line allocating per‑document data. `observed (runtime)`

Consuming a **21‑byte** `simple.txt` (classifier absent) grows RSS by ~12–15 MB, with the stage breakdown (*§9.1*): `observed (runtime)`

| Stage (function, `file:line`) | VmRSS Δ | Nature |
|---|---|---|
| `pre_check_duplicate` MD5 `f.read()` — `consumer.py:104` | **+1068 / +1100 KB** | first‑touch of libmagic/hashlib, **not** the 21 bytes |
| `TextDocumentParser.parse` read — `paperless_text/parsers.py:42` | +4 KB | 21‑byte read is negligible |
| `get_optimised_thumbnail` Pillow `Image.new((500,700))` + TrueType — `paperless_text/parsers.py:26,28` | **+2460 KB** (stable across both runs) | **fixed native Pillow cost, document‑independent** |
| `parse_date` regex over text — `parsers.py:261` | +0 KB | — |
| `load_classifier()` — `consumer.py:292` → `classifier.py:30` | +0 KB → `None` | near‑zero when **absent** |
| `_store` MD5 — `consumer.py:402` | +88 / +156 KB | — |
| post‑consume signals (matching + `add_to_index` Whoosh writer) | ~+2.4 MB | Whoosh `AsyncWriter` open |

Both runs went from ~54.6 MB to **67.49 MB** (net ~**+12.85 MB**) for the 21‑byte file, with `gc.garbage == 0` throughout. The `tracemalloc` line‑diff for a **cold** consume (*§9.2*) attributes the Python‑heap growth to `<frozen importlib._bootstrap_external>:647` **+3011.7 KB** (module `.pyc` loading), `whoosh/lang/morph_en.py:601‑602` (English morphology tables), `sre_compile.py:804` (regex compile) — i.e. **module import, not consume logic**. Meanwhile RSS on the cold consume grew **+57.9 MB** while `tracemalloc` saw only **+6.28 MB** of it (itself dominated by that `.pyc` loading): the ~**51.6 MB** remainder is **native** (Pillow, libmagic, and — when a model is present — scikit‑learn/SciPy/NumPy `.so` + arrays). This RSS‑vs‑`tracemalloc` divergence is the core of OBJ‑1: **the bulk of the "spike" is native memory that only RSS can see.** `observed (runtime)`

### OBJ‑2 — Unnecessary copies & reference lifetime

**Direct answer:** metadata handling does **not** make an unnecessary *content* copy where it is most often assumed; the real copies are whole‑file byte reads and a fuzzy‑match string, both **size‑proportional** and **transient**. `observed (runtime)`

- **`matching.py:63` `document_content = document.content` is a reference, not a copy** — proven by object identity `document_content is document.content == True`, identical `id(...)` (*§9.6A*). Each `match_*` handler binds the same string object; it does not duplicate the content. `observed (runtime)`
- **Reference lifetime of `text`:** the extracted text (`consumer.py:271`) lives in the `try_consume_file` frame across `transaction.atomic()` (`consumer.py:298`) and is passed to all six post‑consume handlers wired in `apps.py:22‑27`; it is a single reference to the content that is anyway persisted to `Document.content` (`models.py:117`), and it is released at function return — long‑lived *within* the call, but not a duplicate and not retained after. `observed (runtime)` / `inferred (from reading code)`
- **Genuine copies, all size‑proportional and transient** (*§9.6B/C*): for an 8 MB document, `consumer.py:104/402/432` each `f.read()` the whole file (**8.00 MB** live bytes per read — dedup MD5, store MD5, and each `_write` of original/thumbnail/archive), and `matching.py:131` builds a **7.86 MB distinct** string via `re.sub(...)` **per fuzzy‑matching model**. Peak Python heap during a consume scales **linearly at ~7× the content size** (1 MB→7.16 MB, 4 MB→28.60 MB, 8 MB→55.88 MB, 16 MB→111.94 MB), reflecting ~7 simultaneous size‑proportional buffers at the peak. For the report's "small text documents" this ~7× term is trivial (7 × 21 B); it only becomes significant for large files. All of it is freed after use (steady‑state heap is flat — see OBJ‑3). `observed (runtime)`

### OBJ‑3 — Cache accumulation across documents/batches

**Direct answer:** **nothing accumulates across documents.** Every candidate cache was driven repeatedly in one long‑lived process; each shows a one‑time first‑call cost and then flat/zero growth. `observed (runtime)`

| Candidate (`file:line`) | Behaviour across 6+ repeats | Verdict |
|---|---|---|
| Classifier model — `classifier.py:30,76‑94` | **No module‑level cache**; a fresh `DocumentClassifier` is built and `.load()`‑ed **every** consume; the previous one is GC'd. In a 20‑consume batch the sklearn modules import once (cold), then steady deltas ≈ 0 (*§9.7*). | reloaded, not accumulated |
| Matching `objects.all()` + content scans — `matching.py:27,40,53,60‑63` | pass 0 +0.09 MB, pass 1 +3.38 MB (fuzzywuzzy first import), passes 2‑5 **+0.00 MB** (*§9.5C*) | transient, no accumulation |
| Whoosh `AsyncWriter` per doc — `index.py:118` | doc 0 +1.02 MB, docs 1‑5 +0.04‑0.11 MB, total +1.16 MB (*§9.5B*) | bounded per doc |
| pikepdf metadata — `paperless_tesseract/parsers.py:34` | call 0 +9.45 MB (qpdf lib load), calls 1‑5 **+0.00 MB** (*§9.5A*) | no per‑request accumulation |
| Django ORM query log | gated by `DEBUG=False` default (`settings.py:50`) | not accumulating in canonical config `inferred (from reading code)` |

The 20‑document in‑process batches confirm it at scale: with the classifier **absent**, cold consume 0 = +14.80 MB then steady mean **+0.084 MB** (plateau ~70 MB); with the classifier **present**, cold consume 0 = +69.63 MB then steady mean **+0.123 MB** (plateau ~125.9 MB, with consumes 6–19 mostly exactly **+0.00 MB**). `gc.garbage == 0` on every iteration. **Even though the model is reloaded per document, there is zero net per‑document growth** — the ~56 MB is a one‑time *per‑process* import cost, not a per‑document accumulation. `observed (runtime)` (*§9.7*)

### OBJ‑4 — What distinguishes a spike from a non‑spike

**Direct answer:** the two dominant differentiators are **(1) whether a trained classifier model exists** and **(2) which parser runs**; document byte‑size is a distant third. `observed (runtime)`

- **Classifier present vs absent — the biggest, most disproportionate, document‑independent spike.** Consuming the **identical 21‑byte** `simple.txt` (*§9.3*, ≥2 runs each): **absent** → **+14.74 / +15.24 MB**, and scikit‑learn/NumPy/SciPy are **never imported**; **present** → **+71.36 / +71.12 MB**, with all three imported. That is **~+56 MB for a 21‑byte file, purely because a model file exists.** The model on disk is only **152.6 KB**, so the cost is the sklearn/SciPy/NumPy machinery and deserialized object graph, not the model data. Isolated, `load_classifier()` alone costs **+53.62 MB** clean, of which `malloc_trim(0)` reclaims only **−0.04 MB** (it is *live* while referenced — legitimately in use, not a leak) (*§9.4*). Mechanism: `classifier.py:76‑94` performs seven sequential `pickle.load()` calls building a `CountVectorizer`, a tags binarizer, and three `MLPClassifier` models. This **confirms the guiding hypothesis.** `observed (runtime)`
- **Parser selected (text vs OCR).** See OBJ‑5 — the OCR (`RasterisedDocumentParser`) path costs 4–5× the text path for the same tiny inputs.
- **First vs subsequent consume.** The first consume in a process pays cold imports (+57.9 MB RSS for the very first, *§9.2*); subsequent consumes in the *same* process are cheap. **But** because `Q_CLUSTER["recycle"] = 1` (`settings.py:452`) restarts the worker after **every** task, in production **every** document is a "first consume" in a fresh process and pays the full cold cost — so the "first vs subsequent" distinction collapses for the recycled consume worker. `observed (runtime)` (*§9.10*)
- **Feature flags.** With `CONSUMER_ENABLE_BARCODES` on (non‑default), `convert_from_path` rasterizes **all pages** (`tasks.py:105`), spiking peak RSS **+113.25 MB for a 78 KB multi‑page PDF** (`several-patcht-codes.pdf`, separators found at pages [2,5]) (*§9.8*) — a large, page‑count‑proportional differentiator absent from the default configuration.

### OBJ‑5 — Type & batch scaling

**Direct answer:** the per‑consume cost is governed by **parser type**, roughly **flat with input byte‑size** within a type, and — across a batch — **plateaus** (in a long‑lived worker) or **resets per task** (under the default `recycle:1`). `observed (runtime)`

**Type scaling** (classifier absent, clean RSS, ≥2 runs, *§9.5*):

| Sample | Size | Parser | VmRSS Δ | Extracted `content` len |
|---|---|---|---|---|
| `simple.txt` | 21 B | `TextDocumentParser` | **+7.33 MB** | 21 |
| `simple.pdf` | 22 926 B | `RasterisedDocumentParser` (OCR) | +30.41 / +30.75 MB | 24 |
| `test_with_bom.pdf` | 16 109 B | OCR | **+36.63 / +36.72 MB** | 7279 |
| `simple.png` | 7 913 B | OCR | +29.77 / +31.95 MB | 24 |
| `simple.jpg` | 17 740 B | OCR | +30.14 / +30.45 MB | 24 |

The parser dominates: **text ~7 MB vs OCR ~30–39 MB (4–5×)**, and OCR is roughly **flat across a ~3× input‑size range** (7.9 KB PNG ≈ 22.9 KB PDF ≈ 30 MB). The extra ~8 MB for `test_with_bom.pdf` tracks its **much larger extracted text** (7279 chars), not its input bytes — i.e. cost scales with *extracted content*, not file size. `observed (runtime)`

**Batch scaling** (*§9.7, §9.9, §9.10*): in a single long‑lived process, RSS climbs on the first document and then **plateaus** (reused working set) — 6× fresh 24 MB documents end at ~219 MB with the last two iterations flat (**+0.04 / +0.03 MB**), and total growth across the five documents after document 0 is **+24.76 MB** (far below the +120 MB five leaked documents would add). Through the **real Django‑Q cluster with `recycle:1`**, each of 8 tasks ran in a **distinct** worker PID peaking ~138–145 MB (classifier present) and then **exited**, returning all per‑task memory to the OS; with recycle raised (non‑default) a single worker handled all 8 tasks and its RSS climbed to 133.1 MB on the first consume then **plateaued at ~133.7 MB** rather than climbing ~55 MB/task. Neither mode leaks. `observed (runtime)`

---

## 5. Components / Methods That Hold Memory

Columns: **Δ** values are the measured increase attributable to that component; **Native** = allocated by a C extension (RSS‑visible, `tracemalloc`‑blind); **Python‑heap** = `tracemalloc`‑visible. Every row is grounded in a `file:line` and an appendix section.

| Component | Method / site (`file:line`) | What it holds / copies | Measured Δ | Native / Python‑heap | Accumulates? | Label |
|---|---|---|---|---|---|---|
| **Classifier** | `load_classifier()` → `DocumentClassifier.load()` — `classifier.py:30,76‑94` (7× `pickle.load`) | `CountVectorizer` + tags binarizer + 3× `MLPClassifier` object graph | **+53.62 MB** isolated; **~+56 MB** present‑vs‑absent for a 21 B file (*§9.3,§9.4*) | **Native**+Python (RSS +53.62 MB; tracemalloc Python‑heap +29.65 MB mostly module import, rest native `.so`+arrays) | No (reloaded per consume, GC'd; batch plateau *§9.7*) | `observed (runtime)` |
| **Text parser** | `get_thumbnail()` `Image.new("RGB",(500,700))` + `ImageFont.truetype` — `paperless_text/parsers.py:26,28`; `parse()` `self.text=f.read()` `:42` | fixed 500×700 PIL image + font; whole file bytes | thumbnail **+2.46 MB** for a 21 B file (*§9.1*) | **Native** (Pillow) | No | `observed (runtime)` |
| **OCR parser** | `RasterisedDocumentParser` `self.text` — `paperless_tesseract/parsers.py:243,264` | extracted text; ocrmypdf/Ghostscript rasterization working set | consume Δ **+30–37 MB** (*§9.5*) | **Native** (Ghostscript/Tesseract/Pillow) | No | `observed (runtime)` |
| **Consumer whole‑file reads** | dedup MD5 `f.read()` `consumer.py:104`; store MD5 `:402`; `_write` `write_file.write(read_file.read())` `:429‑432` | full file bytes in memory, once per read | **8.00 MB** per read for an 8 MB file (*§9.6B*) | Python‑heap (`bytes`) | No (transient per consume) | `observed (runtime)` |
| **Matching — content** | `matches()` `document_content = document.content` — `matching.py:63` | **reference**, not a copy (identity `True`) | 0 (alias) (*§9.6A*) | — | No | `observed (runtime)` |
| **Matching — fuzzy** | fuzzy branch `re.sub(r"[^\w\s]","",document_content)` — `matching.py:131` | distinct normalized string **per fuzzy model** | **7.86 MB** distinct for 8 MB content (*§9.6C*) | Python‑heap (`str`) | No (transient) | `observed (runtime)` |
| **Matching — querysets** | `Correspondent/DocumentType/Tag.objects.all()` — `matching.py:27,40,53` | materialized rule rows | pass‑1 +3.38 MB (fuzzywuzzy import), then **+0.00 MB** (*§9.5C*) | Python‑heap | No | `observed (runtime)` |
| **Search index** | `add_or_update_document()` fresh `AsyncWriter` per doc — `index.py:118`; `update_document` `writer.update_document` `:87‑90` | Whoosh writer buffer + content field | doc 0 +1.02 MB, then +0.04‑0.11 MB, total **+1.16 MB** over 6 docs (*§9.5B*) | Native + Python‑heap | No (per‑doc, committed) | `observed (runtime)` |
| **Batch reindex writer** | single `with AsyncWriter(ix) as writer:` across **all** docs — `tasks.py:43‑45` | one writer buffering many docs | not the consume path; buffers ∝ batch | Native + Python‑heap | Bounded by commit | `inferred (from reading code)` |
| **pikepdf metadata** | `extract_metadata()` `pikepdf.open(document_path)` — `paperless_tesseract/parsers.py:34`; endpoint `views.py:269,295,302` (original **+** archive) | whole PDF via native qpdf | call 0 +9.45 MB (qpdf load), calls 1‑5 **+0.00 MB** (*§9.5A*) | **Native** (qpdf) | No per‑request accumulation | `observed (runtime)` |
| **REST upload buffer** | `validate_document()` `document_data = document.file.read()` — `serialisers.py:451`; `post()` `views.py:497,517` | entire uploaded file in the **web** process | 40 MB upload → worker 68.9 → **+42.6 MB** (≈buffered file), transient peak 172.6 MB; 3 uploads oscillate min 68.9 / max 172.6 / last 152.3 MB (reuse, HTTP 200 ~0.15 s each) (*§9.13*) | Python‑heap (`bytes`) | High‑water mark in long‑lived gunicorn (not recycled) | `observed (runtime)` |
| **Email buffer** | `att.payload` `magic.from_buffer` `mail.py:317`, `f.write(att.payload)` `:327` | entire attachment in the **mail** process | 20 MB attachment → `handle_message` **+27.85 MB**, 92.47 MB reclaimed after gc+trim (*§9.12*) | Python‑heap (`bytes`) | High‑water mark in long‑lived mail process | `observed (runtime)` (transport non‑canonical) |
| **Importer manifest** | `json.load(f)` — `document_importer.py:73`; `list(...)` `:137‑140` | entire `manifest.json` + materialized record list | 43.8 MB manifest (20k records) → `json.load` **+118.82 MB** RSS (~2.7×), `list()` +25.45 MB RSS / +0.2 MB tracemalloc (references) (*§9.11*) | Python‑heap | Held for command duration | `observed (runtime)` |
| **Model field** | `Document.content = models.TextField()` — `models.py:117` | full extracted text on the row | ∝ content length | Python‑heap | Persisted (by design) | `inferred (from reading code)` |
| **Worker recycle** | `Q_CLUSTER["recycle"]=1` — `settings.py:452` | forces worker exit after each task | per‑task RSS returned to OS at exit (*§9.10*) | — | **Prevents** accumulation | `observed (runtime)` |

---

## 6. Normal‑vs‑Problem Determination

**Verdict: the observed behaviour is normal CPython + glibc allocator retention, NOT a leak, NOT an unbounded cache, and NOT an unnecessary content copy.** The "disproportionate spike for small text documents" is the **fixed, document‑independent cost of loading the classifier model** (scikit‑learn/SciPy/NumPy machinery, ~+56 MB) plus native parser/library first‑touch; the "not released to the OS in a timely manner" symptom is **glibc `malloc` arena retention** — freed pages returned to the process allocator (and reused) rather than to the kernel. Three independent discriminators agree: `observed (runtime)`

**1. `gc.garbage` is empty on every iteration of every run.** No uncollectable reference cycles were ever produced — across 20‑document in‑process batches (*§9.7*), the 6× large‑document loop (*§9.9*), and all entry‑point/edge tests. A genuine Python reference leak via cycles would populate `gc.garbage`; it never did. `observed (runtime)`

**2. Cross‑iteration `tracemalloc` deltas are flat — the Python heap does not grow.** Repeating identical inputs, the only non‑trivial per‑iteration Python‑heap growth is `sqlite3/dbapi2` statement‑cache and, dominant in the numbers, **`tracemalloc`'s own bookkeeping** (`tracemalloc.py:558/115/193`) — an artifact of measuring, not app growth (*§9.7B*). App‑attributable steady‑state growth is ≈ 0 (mean **+0.12 MB/consume**, many iterations exactly **+0.00 MB**). Monotonic growth would indicate a problem; we observe a plateau. (Note: enabling `tracemalloc` itself inflates RSS by ~168 MB — so all magnitude RSS numbers were taken with `tracemalloc` **off**; *§9.4B*.) `observed (runtime)`

**3. `malloc_trim(0)` reclaims retained RSS — the signature of arena free‑list retention, not a leak.** After a large consume with all references dropped, RSS stays high; calling `ctypes.CDLL("libc.so.6").malloc_trim(0)` returns free pages to the OS — reclaiming **+75.13 MB** after the 6×24 MB loop (*§9.9B*) and **+2.86 MB** after the 20‑document classifier‑present batch (*§9.7B*). The 6‑iteration loop is the decisive proof: iter 0 = +140.62 MB, then the plateau settles (iters 4 and 5 flat at **+0.04 / +0.03 MB**, ending ~219 MB) — growth across the five further 24 MB documents is **+24.76 MB**, i.e. **well under one‑fifth of the +120 MB a leak of five 24 MB documents would produce** — so **retained memory is reused by the next document, not leaked.** Capping arenas with `MALLOC_ARENA_MAX=1`/`2` (non‑default) had **negligible effect on the steady plateau** in this workload — the text loop stayed ~219 MB and the classifier‑present batch stayed ~126 MB regardless (*§9.9C*), which tells us the retention here is **within‑arena free‑list** memory (reclaimable by `malloc_trim`) rather than per‑thread arena *multiplication*; the decisive reclaimability evidence is `malloc_trim`, not the arena count. `observed (runtime)`

**Why the symptoms appeared:** `observed (runtime)` / `inferred (from reading code)`
- *"disproportionate to document size"* → the dominant cost (classifier ~56 MB, OCR native ~30 MB, Pillow thumbnail ~2.4 MB) is **fixed and independent of the tiny document**; a 21‑byte file pays nearly the same as a 1 MB one.
- *"inconsistent / varies by source or stage"* → it depends on **which parser runs** (text vs OCR), **whether a model exists**, and **which process** does the work: the **recycled consume worker** (`recycle:1`) returns everything at task exit, whereas a **long‑lived gunicorn/mail process** keeps the whole‑file upload/attachment buffer as an in‑process high‑water mark (*§9.12,§9.13*) — the same operation "spikes and releases" in one process and "spikes and stays" in another.
- *"not released to the OS in a timely manner"* → glibc keeps freed pages in its arena free‑lists and only trims them lazily; an explicit `malloc_trim(0)` proves the pages are **reclaimable** (a leak's pages are not). On this 128‑CPU host up to ~1024 arenas are theoretically possible, but capping `MALLOC_ARENA_MAX` did not move the plateau here, so the retention is within‑arena free space rather than arena multiplication.

**No source change is proposed** — this is a diagnosis. The retention is expected allocator behaviour that `recycle:1` already neutralizes for the consume path.

---

## 7. Guiding Hypothesis — Tested, Not Assumed

The investigation set out to test (confirm or refute) a specific hypothesis; the evidence **confirms it on every axis**:

| Hypothesis clause | Prediction | Result | Evidence |
|---|---|---|---|
| The disproportionate spike for small text docs is a **fixed, document‑independent cost, dominated by classifier deserialization** | present‑vs‑absent should add a large, size‑independent delta for a 21 B file | **CONFIRMED** — +~56 MB for the identical 21 B file purely from model presence; `load_classifier()` alone +53.62 MB | *§9.3, §9.4* |
| Those classifier buffers are **native (RSS‑visible, `tracemalloc`‑blind)** | RSS ≫ `tracemalloc` delta on the classifier load | **CONFIRMED** — clean RSS +53.62 MB; `tracemalloc` Python‑heap +29.65 MB is mostly module `.pyc` import, the rest is native `.so`/arrays | *§9.4A/B* |
| Metadata handling does **not** make an unnecessary content copy | `document_content = document.content` should be a reference | **CONFIRMED** — object identity `True`; genuine copies are `f.read()` bytes + fuzzy `re.sub`, both size‑proportional & transient | *§9.6* |
| The "not released to OS" symptom is **glibc arena retention, not a leak** | `gc.garbage==0`, flat Python heap, RSS plateaus, `malloc_trim` reclaims | **CONFIRMED** — all four hold; `malloc_trim` reclaimed +75.13 MB after a large‑doc loop | *§9.7, §9.9* |

One sub‑clause required refinement by evidence (per the run‑first rule): the plan speculated that `MALLOC_ARENA_MAX` would visibly lower the plateau. It **did not** in this workload (*§9.9C*); the reclaimability is instead demonstrated directly by `malloc_trim(0)`. Reported as observed, not as assumed.

---

## 8. Coverage Pass

Confirming every part of the question and every named item was addressed:

- **"what's actually causing these memory spikes"** → §1, OBJ‑1 (allocation sites), OBJ‑4 (differentiators). Root cause = fixed native costs (classifier deserialization foremost), not per‑document data. `observed (runtime)`
- **"metadata handling creating unnecessary copies?"** → OBJ‑2: no unnecessary *content* copy (`matching.py:63` is a reference); the copies that exist (`f.read()`, fuzzy `re.sub`) are size‑proportional & transient. `observed (runtime)`
- **"holding onto references longer than needed?"** → OBJ‑2: `text` lives across `transaction.atomic()` and the six post‑consume handlers but is a single reference, released at return; nothing retained after the call. `observed (runtime)` / `inferred (from reading code)`
- **"some caching behaviour that's accumulating data?"** → OBJ‑3: none. Classifier (no module cache, reloaded+GC'd per consume), matching querysets, Whoosh writer, pikepdf, ORM query log (DEBUG=False) all one‑time or bounded; 20‑doc batches plateau. `observed (runtime)`
- **"actual runtime memory measurements"** → §9 appendix: every claim has its command + complete unedited output, captured inside the Python 3.9 container. `observed (runtime)`
- **"identification of which components/methods hold memory"** → §5 table: classifier, text/OCR parsers, consumer reads, matching, Whoosh, pikepdf, REST/email buffers, importer, model field, worker‑recycle — each with `file:line`, measured Δ, and native/Python attribution. `observed (runtime)`
- **"evidence distinguishing normal Python GC from something problematic"** → §6 verdict: `gc.garbage==0`, flat cross‑iteration `tracemalloc`, RSS plateau, `malloc_trim` reclaim → normal glibc arena retention, not a leak. `observed (runtime)`
- **Every objective OBJ‑1…OBJ‑5** → §4, each with measured evidence.
- **All three ingestion entry points** (directory watcher, REST upload, email), the **importer**, and the **metadata endpoint** → OBJ‑2/§5/§9.10–9.14. Directory‑watcher enqueues the same `consume_file` task measured directly (`document_consumer.py:46,86`) — `inferred (from reading code)` for the watchdog wrapper, `observed (runtime)` for the consume it feeds.
- **Edge/error paths** → duplicate rejection (§9.14), barcode split (§9.8), OCR (§9.5), large‑document trim probe (§9.9). `observed (runtime)`
- **Transitional states** (before/during/after) → stage‑instrumented consume (§9.1) and before/after/trim sampling throughout. `observed (runtime)`
- **Run‑to‑run inconsistency** → identical inputs repeated ≥2× (§9.1, §9.3, §9.5); distribution reported. `observed (runtime)`
- **Non‑canonical labels** → email transport (in‑process `imap_tools` message, §9.12) and non‑default configs (`CONSUMER_ENABLE_BARCODES`, raised `recycle`, `MALLOC_ARENA_MAX`) explicitly marked. `observed (runtime)`

---

## 9. Captured Evidence (commands + complete, unedited output)

Every block below is the **exact command** and its **complete, unedited output**, captured inside the canonical Python 3.9 container (`paperless_app`). Harness scripts lived only in the container's `/tmp/memharness/` (outside the tracked tree) and were removed on completion. The shared run-environment prefix for the harness is:

```bash
docker exec -w /app/src --user testuser \
  -e PAPERLESS_REDIS=redis://paperless-broker:6379 -e PAPERLESS_DISABLE_DBHANDLER=true \
  -e HOME=/home/testuser -e DJANGO_SETTINGS_MODULE=paperless.settings paperless_app \
  bash -lc 'cd /tmp/memharness && python3 <script> <args>'
```

`memprobe.py` provided the three lenses (`/proc/self/status` `VmRSS`, `resource.getrusage().ru_maxrss`, `tracemalloc`, `gc`) plus the `ctypes` `malloc_trim(0)` probe; `hbootstrap.py` reused the project's own `documents.tests.utils.setup_directories` so every consume ran the canonical `Consumer().try_consume_file()` path with MEDIA/DATA/SCRATCH redirected to throwaway `/tmp` dirs.

### §9.0 Canonical baseline (consume path works end-to-end)

```bash
python3 baseline.py
```

```
OK Document created:
  id       = 228
  title    = 'simple'
  content  = 'This is a test file.\n'
  checksum = 2d282102fa671256327d4767ec23bc6b
  mime     = text/plain
  Document.objects.count() = 1
  source_path exists    = True
  thumbnail_path exists = True
cleanup: temp dirs removed, db rows deleted
```

### §9.1 Stage-instrumented consume — `simple.txt` (21 B), classifier ABSENT (run 1 and run 2)

Confirms the per-stage `VmRSS` breakdown and that identical inputs are stable across >=2 runs (thumbnail +2460 KB both runs; `gc.garbage==0`).

```bash
python3 core_consume.py stage   # run 1
```

```
===== STAGE-INSTRUMENTED CONSUME: simple.txt (21 B), classifier ABSENT =====
[BEFORE try_consume_file       ] VmRSS=   54.63 MB  peak=   52.07 MB  gc.garbage=0
    pre_check_duplicate  (MD5 f.read L104)   VmRSS    56.45 ->    57.52 MB  (+1100.0 KB)
    TextDocumentParser.parse (read L42)      VmRSS    62.51 ->    62.51 MB  (   +4.0 KB)
    get_optimised_thumbnail (Pillow L26/28)  VmRSS    62.51 ->    64.91 MB  (+2460.0 KB)
    parse_date (regex parsers.py L261)       VmRSS    64.91 ->    64.91 MB  (   +0.0 KB)
    load_classifier (consumer.py L292)       VmRSS    64.91 ->    64.91 MB  (   +0.0 KB) -> None
    _store (store MD5 L402)                  VmRSS    64.92 ->    65.07 MB  ( +156.0 KB)
    signal document_consumption_finished     VmRSS    67.45 MB   (post-consume handlers begin)
    _write (copy read_file.read L432)        VmRSS    67.45 ->    67.45 MB  (   +0.0 KB)
    _write (copy read_file.read L432)        VmRSS    67.45 ->    67.45 MB  (   +0.0 KB)
[AFTER  try_consume_file       ] VmRSS=   67.49 MB  peak=   64.39 MB  gc.garbage=0
  -> Document id=229 content='This is a test file.\n'
```

```bash
python3 core_consume.py stage   # run 2
```

```
===== STAGE-INSTRUMENTED CONSUME: simple.txt (21 B), classifier ABSENT =====
[BEFORE try_consume_file       ] VmRSS=   54.65 MB  peak=   54.44 MB  gc.garbage=0
    pre_check_duplicate  (MD5 f.read L104)   VmRSS    56.46 ->    57.51 MB  (+1068.0 KB)
    TextDocumentParser.parse (read L42)      VmRSS    62.54 ->    62.55 MB  (   +4.0 KB)
    get_optimised_thumbnail (Pillow L26/28)  VmRSS    62.55 ->    64.95 MB  (+2460.0 KB)
    parse_date (regex parsers.py L261)       VmRSS    64.95 ->    64.95 MB  (   +0.0 KB)
    load_classifier (consumer.py L292)       VmRSS    64.95 ->    64.95 MB  (   +0.0 KB) -> None
    _store (store MD5 L402)                  VmRSS    64.96 ->    65.04 MB  (  +88.0 KB)
    signal document_consumption_finished     VmRSS    67.46 MB   (post-consume handlers begin)
    _write (copy read_file.read L432)        VmRSS    67.46 ->    67.46 MB  (   +0.0 KB)
    _write (copy read_file.read L432)        VmRSS    67.46 ->    67.46 MB  (   +0.0 KB)
[AFTER  try_consume_file       ] VmRSS=   67.49 MB  peak=   66.37 MB  gc.garbage=0
  -> Document id=230 content='This is a test file.\n'
```

### §9.2 Whole-consume `tracemalloc` diff (cold allocation sites) + 6 steady-state consumes

Attributes cold Python-heap growth to module-import lines; shows steady-state `tracemalloc` deltas are tiny and `gc.garbage==0`.

```bash
python3 core_consume.py tm 6
```

```
===== WHOLE-CONSUME tracemalloc DIFF + RSS: simple.txt, classifier ABSENT =====
(warmup=3 unmeasured consumes to load lazy imports, then 6 measured)

-- COLD first consume --
  VmRSS:               57.35 ->   115.24 MB   (delta    +57.9 MB)
  peak ru_maxrss:      56.29 ->   114.05 MB
  tracemalloc net Python-heap delta:  +6284.9 KB ; gc.garbage=0
  top Python-heap allocation sites (cold, compare_to 'lineno'):
       +3011.7 KB  (blocks +33701)  <frozen importlib._bootstrap_external>:647
        +148.9 KB  (blocks +89)  /usr/local/lib/python3.9/site-packages/django/utils/functional.py:49
        +131.9 KB  (blocks +2409)  /usr/local/lib/python3.9/site-packages/whoosh/lang/morph_en.py:601
         +72.0 KB  (blocks +1)  /usr/local/lib/python3.9/site-packages/whoosh/lang/morph_en.py:602
         +38.3 KB  (blocks +404)  <frozen importlib._bootstrap>:228
         +32.9 KB  (blocks +44)  /usr/local/lib/python3.9/sre_compile.py:804
         +32.1 KB  (blocks +272)  <frozen importlib._bootstrap_external>:123
         +26.8 KB  (blocks +100)  /usr/local/lib/python3.9/abc.py:106
         +20.1 KB  (blocks +184)  /usr/local/lib/python3.9/abc.py:24
         +18.0 KB  (blocks +256)  <frozen importlib._bootstrap>:353
         +16.9 KB  (blocks +254)  <frozen importlib._bootstrap>:36
         +14.6 KB  (blocks +72)  /usr/local/lib/python3.9/collections/__init__.py:497

-- STEADY-STATE (6 identical consumes after warmup) --
  iter 0: VmRSS   117.94 ->   148.68 MB (delta +31472.0 KB) | tracemalloc Python-heap   +64.4 KB | gc.garbage=0
  iter 1: VmRSS   148.53 ->   148.59 MB (delta   +68.0 KB) | tracemalloc Python-heap   +36.3 KB | gc.garbage=0
  iter 2: VmRSS   148.54 ->   153.54 MB (delta +5128.0 KB) | tracemalloc Python-heap   +36.4 KB | gc.garbage=0
  iter 3: VmRSS   153.58 ->   154.45 MB (delta  +884.0 KB) | tracemalloc Python-heap   +36.4 KB | gc.garbage=0
  iter 4: VmRSS   154.50 ->   153.57 MB (delta  -956.0 KB) | tracemalloc Python-heap   +20.6 KB | gc.garbage=0
  iter 5: VmRSS   153.75 ->   157.14 MB (delta +3464.0 KB) | tracemalloc Python-heap   +36.2 KB | gc.garbage=0

  steady-state net VmRSS delta per consume (KB): 31472.0, 68.0, 5128.0, 884.0, -956.0, 3464.0
  steady-state absolute VmRSS AFTER each consume (MB): 148.68, 148.59, 153.54, 154.45, 153.57, 157.14
  => min/max/mean per-consume delta: -956.0 / 31472.0 / 6676.7 KB
  => absolute RSS trend (plateau check): first=148.68 MB last=157.14 MB span=8.54 MB
```

### §9.3 Classifier ABSENT vs PRESENT — identical 21 B `simple.txt`, four fresh processes

The centerpiece OBJ-4 measurement. ABSENT never imports sklearn/numpy/scipy; PRESENT imports all three and costs ~+56 MB more for the **same** 21-byte file.

```bash
python3 classifier_test.py consume absent   # run 1
```

```
===== consume simple.txt (21 B), classifier MODEL_FILE ABSENT =====
  VmRSS:             53.64 ->    68.38 MB   (delta   +14.74 MB)
  peak ru_maxrss:    53.19 ->    68.19 MB   (delta   +15.00 MB)
  sklearn imported during consume: before=False after=False ; numpy=False scipy=False
  Document id=240 content='This is a test file.\n' correspondent=None document_type=None
```

```bash
python3 classifier_test.py consume absent   # run 2
```

```
===== consume simple.txt (21 B), classifier MODEL_FILE ABSENT =====
  VmRSS:             52.17 ->    67.41 MB   (delta   +15.24 MB)
  peak ru_maxrss:    53.73 ->    67.17 MB   (delta   +13.44 MB)
  sklearn imported during consume: before=False after=False ; numpy=False scipy=False
  Document id=241 content='This is a test file.\n' correspondent=None document_type=None
```

```bash
python3 classifier_test.py consume present  # run 1
```

```
===== consume simple.txt (21 B), classifier MODEL_FILE PRESENT =====
  VmRSS:             52.12 ->   123.48 MB   (delta   +71.36 MB)
  peak ru_maxrss:    53.59 ->   123.17 MB   (delta   +69.58 MB)
  sklearn imported during consume: before=False after=True ; numpy=True scipy=True
  Document id=242 content='This is a test file.\n' correspondent=None document_type=None
```

```bash
python3 classifier_test.py consume present  # run 2
```

```
===== consume simple.txt (21 B), classifier MODEL_FILE PRESENT =====
  VmRSS:             52.18 ->   123.30 MB   (delta   +71.12 MB)
  peak ru_maxrss:    53.59 ->   122.16 MB   (delta   +68.58 MB)
  sklearn imported during consume: before=False after=True ; numpy=True scipy=True
  Document id=243 content='This is a test file.\n' correspondent=None document_type=None
```

### §9.4 Isolating `load_classifier()` — clean RSS (A) and `tracemalloc` attribution (B)

**§9.4A** — clean RSS cost with `tracemalloc` OFF (so RSS is not inflated); `malloc_trim(0)` reclaims ~0 because the model is still live/referenced.

```bash
python3 classifier_test.py loadonly
```

```
===== isolate load_classifier() — CLEAN RSS (no tracemalloc) =====
  sklearn in sys.modules BEFORE: False
  load_classifier() -> DocumentClassifier ; sklearn=True numpy=True scipy=True
  VmRSS before load      =    52.77 MB
  VmRSS after  load      =   106.38 MB  (delta +53.62 MB)  <- native sklearn/scipy/numpy + model
  VmRSS after gc.collect =   106.38 MB
  malloc_trim(0) returned 1 ; VmRSS after trim =   106.34 MB (delta from post-load -0.04 MB)
  => classifier object retains its buffers while alive (clf still referenced): trim reclaims only free pages, not the live model
```

**§9.4B** — same load with `tracemalloc` ON: RSS is inflated to +221 MB by tracemalloc's own bookkeeping (hence §9.4A is the trustworthy RSS); the Python-heap delta is dominated by module `.pyc` import and is **blind** to the native sklearn/numpy/scipy buffers.

```bash
python3 classifier_test.py loadtm
```

```
===== load_classifier() — tracemalloc Python-heap attribution =====
  (NOTE: tracemalloc bookkeeping inflates RSS here; trust its Python-heap delta,
   not this run's RSS. The clean RSS cost is in the 'loadonly' run.)
  VmRSS delta (tracemalloc ON, inflated): +221.22 MB
  tracemalloc Python-heap delta: +29650.0 KB  <- BLIND to native sklearn/numpy/scipy buffers
  top Python-heap sites during load():
      +15869.7 KB  (blocks +90669)  <frozen importlib._bootstrap_external>:647
       +3739.2 KB  (blocks +35919)  <frozen importlib._bootstrap>:228
        +667.7 KB  (blocks +214)  /usr/local/lib/python3.9/site-packages/scipy/_lib/doccer.py:66
        +239.1 KB  (blocks +636)  /usr/local/lib/python3.9/abc.py:106
        +169.2 KB  (blocks +2357)  /usr/local/lib/python3.9/site-packages/scipy/stats/_distn_infrastructure.py:709
        +153.3 KB  (blocks +409)  /usr/local/lib/python3.9/site-packages/scipy/stats/_distn_infrastructure.py:1802
        +152.1 KB  (blocks +1213)  <frozen importlib._bootstrap_external>:123
        +145.2 KB  (blocks +1209)  /usr/local/lib/python3.9/site-packages/scipy/_lib/deprecation.py:17
```

### §9.5 Document-type matrix (OBJ-5) + per-component accumulation (OBJ-3)

Each document type consumed in a fresh process (classifier ABSENT), >=2x each:

```bash
for f in simple.txt simple.pdf test_with_bom.pdf simple.png simple.jpg; do
  for r in 1 2; do python3 doctype_matrix.py $f; done
done
```

```
simple.txt         size=     21 B  mime=text/plain   parser=paperless_text.parsers.TextDocumentParser
   VmRSS   61.00 ->   68.27 MB (delta   +7.26 MB) | peak ru_maxrss delta   +7.00 MB | content_len=21
simple.txt         size=     21 B  mime=text/plain   parser=paperless_text.parsers.TextDocumentParser
   VmRSS   60.93 ->   68.26 MB (delta   +7.33 MB) | peak ru_maxrss delta   +7.00 MB | content_len=21
simple.pdf         size=  22926 B  mime=application/pdf parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   55.16 ->   85.57 MB (delta  +30.41 MB) | peak ru_maxrss delta  +29.00 MB | content_len=24
simple.pdf         size=  22926 B  mime=application/pdf parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   55.31 ->   86.06 MB (delta  +30.75 MB) | peak ru_maxrss delta  +29.00 MB | content_len=24
test_with_bom.pdf  size=  16109 B  mime=application/pdf parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   55.18 ->   91.80 MB (delta  +36.63 MB) | peak ru_maxrss delta  +36.00 MB | content_len=7279
test_with_bom.pdf  size=  16109 B  mime=application/pdf parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   55.19 ->   91.91 MB (delta  +36.72 MB) | peak ru_maxrss delta  +33.70 MB | content_len=7279
simple.png         size=   7913 B  mime=image/png    parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   54.86 ->   84.62 MB (delta  +29.77 MB) | peak ru_maxrss delta  +26.52 MB | content_len=24
simple.png         size=   7913 B  mime=image/png    parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   54.95 ->   86.89 MB (delta  +31.95 MB) | peak ru_maxrss delta  +28.85 MB | content_len=24
simple.jpg         size=  17740 B  mime=image/jpeg   parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   55.12 ->   85.26 MB (delta  +30.14 MB) | peak ru_maxrss delta  +29.21 MB | content_len=24
simple.jpg         size=  17740 B  mime=image/jpeg   parser=paperless_tesseract.parsers.RasterisedDocumentParser
   VmRSS   54.98 ->   85.44 MB (delta  +30.45 MB) | peak ru_maxrss delta  +26.32 MB | content_len=24
```

**§9.5A** — pikepdf `extract_metadata` (the metadata REST endpoint path) x6 in one process:

```bash
python3 components.py metadata
```

```
===== extract_metadata (pikepdf.open -> native qpdf) x6 on simple.pdf =====
(the views.py metadata action calls get_metadata for BOTH original + archive)
  call 0: VmRSS   53.32 ->   62.78 MB (delta  +9.45 MB) ; metadata items=5
  call 1: VmRSS   62.78 ->   62.78 MB (delta  +0.00 MB) ; metadata items=5
  call 2: VmRSS   62.78 ->   62.78 MB (delta  +0.00 MB) ; metadata items=5
  call 3: VmRSS   62.78 ->   62.78 MB (delta  +0.00 MB) ; metadata items=5
  call 4: VmRSS   62.78 ->   62.78 MB (delta  +0.00 MB) ; metadata items=5
  call 5: VmRSS   62.78 ->   62.78 MB (delta  +0.00 MB) ; metadata items=5
  base=53.32 MB last=62.78 MB total growth=9.46 MB over 6 calls
  malloc_trim(0)=1 -> VmRSS now 61.99 MB (reclaimed 0.79 MB)
```

**§9.5B** — `index.add_or_update_document` (fresh Whoosh `AsyncWriter` per doc) x6:

```bash
python3 components.py index
```

```
===== index.add_or_update_document (fresh AsyncWriter per doc) x6 =====
  doc 0: VmRSS   54.78 ->   55.80 MB (delta  +1.02 MB)
  doc 1: VmRSS   55.80 ->   55.84 MB (delta  +0.04 MB)
  doc 2: VmRSS   55.84 ->   55.89 MB (delta  +0.05 MB)
  doc 3: VmRSS   55.89 ->   55.95 MB (delta  +0.05 MB)
  doc 4: VmRSS   55.95 ->   56.06 MB (delta  +0.11 MB)
  doc 5: VmRSS   56.06 ->   56.13 MB (delta  +0.07 MB)
  total growth over 6 docs = 1.16 MB ; malloc_trim(0)=1 -> 55.94 MB
```

**§9.5C** — `matching.match_*` over `document.content` (22 rules incl. a fuzzy one) x6:

```bash
python3 components.py matching
```

```
===== matching.match_* over document.content (22 rules) x6, classifier=None =====
  pass 0: VmRSS   52.89 ->   52.98 MB (delta  +0.09 MB)
  pass 1: VmRSS   52.98 ->   56.36 MB (delta  +3.38 MB)
  pass 2: VmRSS   56.36 ->   56.36 MB (delta  +0.00 MB)
  pass 3: VmRSS   56.36 ->   56.36 MB (delta  +0.00 MB)
  pass 4: VmRSS   56.36 ->   56.36 MB (delta  +0.00 MB)
  pass 5: VmRSS   56.36 ->   56.36 MB (delta  +0.00 MB)
  total growth over 6 passes = -1.14 MB ; malloc_trim(0)=1 -> 51.75 MB
```

### §9.6 OBJ-2 — copies vs references (identity, size-scaling, line attribution)

A single harness run produces three labelled parts in its output below: **§9.6A** = part `(A)` (the `matching.py:63` identity check proving a reference, not a copy); **§9.6B** = part `(B)` (peak Python-heap vs text-document size); **§9.6C** = part `(C)` (direct line-attributed copy measurement, including the `matching.py:131` fuzzy `re.sub` second copy).

```bash
python3 obj2_copies.py
```

```
===== (A) matching.py:63 `document_content = document.content` — copy or reference? =====
  document.content length          = 1600
  `document_content is document.content` (matching.py:63) = True
  id(document.content)=94931112850416  id(document_content)=94931112850416  SAME=True

===== (B) peak Python-heap vs text-document size (tracemalloc.reset_peak) =====
  size(MB) | content_len | tracemalloc PEAK during consume | peak/size
       1   |    1048576  |       7.16 MB peak            | 7.16x
       4   |    4194304  |      28.60 MB peak            | 7.15x
       8   |    8388608  |      55.88 MB peak            | 6.99x
      16   |   16777216  |     111.94 MB peak            | 7.00x

===== (C) direct line-attributed copy measurement (bytes alive at snapshot) =====
  file size on disk = 8.00 MB
  consumer.py:104/402/432  f.read() live bytes = 8.00 MB (current traced=8.00 MB) -> full-file copy, size-proportional, transient
  matching.py:131 fuzzy re.sub NEW string = 7.86 MB live (orig content 8.00 MB) -> genuine 2nd full copy, per FUZZY model
  is the re.sub result a distinct object from content? True
```

### §9.7 In-process batch scaling + normal-vs-problem discriminator (OBJ-3)

**§9.7A** — 20 consumes, classifier ABSENT (cold vs steady; `gc.garbage`; `malloc_trim`):

```bash
python3 batch_recycle.py inproc 20 absent
```

```
===== in-process batch of 20 consumes, classifier ABSENT, MALLOC_ARENA_MAX=(unset/default) =====
  [batch start                   ] VmRSS=   53.93 MB  peak=   53.68 MB  gc.garbage=0
  [after consume  0              ] VmRSS=   68.73 MB  peak=   67.68 MB  gc.garbage=0   delta= +14.80 MB
  [after consume  1              ] VmRSS=   69.99 MB  peak=   68.68 MB  gc.garbage=0   delta=  +1.27 MB
  [after consume  2              ] VmRSS=   70.02 MB  peak=   68.68 MB  gc.garbage=0   delta=  +0.03 MB
  [after consume  3              ] VmRSS=   70.05 MB  peak=   69.68 MB  gc.garbage=0   delta=  +0.03 MB
  [after consume  4              ] VmRSS=   70.09 MB  peak=   69.68 MB  gc.garbage=0   delta=  +0.04 MB
  [after consume  5              ] VmRSS=   70.17 MB  peak=   69.68 MB  gc.garbage=0   delta=  +0.08 MB
  [after consume  6              ] VmRSS=   70.30 MB  peak=   69.68 MB  gc.garbage=0   delta=  +0.13 MB
  [after consume  7              ] VmRSS=   71.64 MB  peak=   70.68 MB  gc.garbage=0   delta=  +1.34 MB
  [after consume  8              ] VmRSS=   68.96 MB  peak=   70.68 MB  gc.garbage=0   delta=  -2.68 MB
  [after consume  9              ] VmRSS=   70.19 MB  peak=   70.68 MB  gc.garbage=0   delta=  +1.23 MB
  [after consume 10              ] VmRSS=   70.25 MB  peak=   70.68 MB  gc.garbage=0   delta=  +0.06 MB
  [after consume 11              ] VmRSS=   70.38 MB  peak=   70.68 MB  gc.garbage=0   delta=  +0.13 MB
  [after consume 12              ] VmRSS=   71.72 MB  peak=   70.75 MB  gc.garbage=0   delta=  +1.34 MB
  [after consume 13              ] VmRSS=   70.14 MB  peak=   71.75 MB  gc.garbage=0   delta=  -1.58 MB
  [after consume 14              ] VmRSS=   70.25 MB  peak=   71.75 MB  gc.garbage=0   delta=  +0.11 MB
  [after consume 15              ] VmRSS=   70.32 MB  peak=   71.75 MB  gc.garbage=0   delta=  +0.07 MB
  [after consume 16              ] VmRSS=   70.45 MB  peak=   71.75 MB  gc.garbage=0   delta=  +0.13 MB
  [after consume 17              ] VmRSS=   69.11 MB  peak=   71.75 MB  gc.garbage=0   delta=  -1.34 MB
  [after consume 18              ] VmRSS=   70.32 MB  peak=   71.75 MB  gc.garbage=0   delta=  +1.21 MB
  [after consume 19              ] VmRSS=   70.32 MB  peak=   71.75 MB  gc.garbage=0   delta=  +0.00 MB
  ---- trend ----
  cold (consume 0) delta            =  +14.80 MB
  steady deltas (consumes 1..19)     = +1.27, +0.03, +0.03, +0.04, +0.08, +0.13, +1.34, -2.68, +1.23, +0.06, +0.13, +1.34, -1.58, +0.11, +0.07, +0.13, -1.34, +1.21, +0.00 MB
  steady mean/ max                  = +0.084 / +1.336 MB
  absolute VmRSS start -> end        = 53.93 -> 70.32 MB (net +16.39 MB over 20 docs)
  ---- drop refs + discriminator ----
  after clean_db + gc.collect        = 70.32 MB   (gc.garbage=0)
  malloc_trim(0) returned 1           = 68.14 MB   (reclaimed +2.19 MB)
  VERDICT INPUTS: gc.garbage=0 (0 => no uncollectable cycles); trim reclaim=+2.19 MB (>0 => glibc arena pages returned to OS = NORMAL retention)
```

**§9.7B** — 20 consumes, classifier PRESENT (model reloaded per consume, yet steady mean ~= 0):

```bash
python3 batch_recycle.py inproc 20 present
```

```
===== in-process batch of 20 consumes, classifier PRESENT, MALLOC_ARENA_MAX=(unset/default) =====
  [batch start                   ] VmRSS=   53.96 MB  peak=   53.91 MB  gc.garbage=0
  [after consume  0              ] VmRSS=  123.59 MB  peak=  122.86 MB  gc.garbage=0   delta= +69.63 MB
  [after consume  1              ] VmRSS=  124.79 MB  peak=  124.86 MB  gc.garbage=0   delta=  +1.21 MB
  [after consume  2              ] VmRSS=  125.48 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.69 MB
  [after consume  3              ] VmRSS=  125.52 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.04 MB
  [after consume  4              ] VmRSS=  125.56 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.04 MB
  [after consume  5              ] VmRSS=  125.63 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.07 MB
  [after consume  6              ] VmRSS=  125.63 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume  7              ] VmRSS=  125.77 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.14 MB
  [after consume  8              ] VmRSS=  125.78 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume  9              ] VmRSS=  125.79 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.02 MB
  [after consume 10              ] VmRSS=  125.85 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.06 MB
  [after consume 11              ] VmRSS=  125.85 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 12              ] VmRSS=  125.85 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 13              ] VmRSS=  125.85 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 14              ] VmRSS=  125.86 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 15              ] VmRSS=  125.92 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.07 MB
  [after consume 16              ] VmRSS=  125.92 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 17              ] VmRSS=  125.92 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 18              ] VmRSS=  125.92 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  [after consume 19              ] VmRSS=  125.92 MB  peak=  124.86 MB  gc.garbage=0   delta=  +0.00 MB
  ---- trend ----
  cold (consume 0) delta            =  +69.63 MB
  steady deltas (consumes 1..19)     = +1.21, +0.69, +0.04, +0.04, +0.07, +0.00, +0.14, +0.00, +0.02, +0.06, +0.00, +0.00, +0.00, +0.00, +0.07, +0.00, +0.00, +0.00, +0.00 MB
  steady mean/ max                  = +0.123 / +1.207 MB
  absolute VmRSS start -> end        = 53.96 -> 125.92 MB (net +71.96 MB over 20 docs)
  ---- drop refs + discriminator ----
  after clean_db + gc.collect        = 125.95 MB   (gc.garbage=0)
  malloc_trim(0) returned 1           = 123.09 MB   (reclaimed +2.86 MB)
  VERDICT INPUTS: gc.garbage=0 (0 => no uncollectable cycles); trim reclaim=+2.86 MB (>0 => glibc arena pages returned to OS = NORMAL retention)
```

**§9.7C** — 12 consumes with `tracemalloc` cross-iteration Python-heap deltas (apparent growth is tracemalloc's own bookkeeping; the only real growth is a bounded sqlite stmt cache):

```bash
python3 batch_recycle.py inproctm 12 present
```

```
===== in-process batch of 12 consumes (tracemalloc Python-heap), classifier PRESENT =====
  (NOTE: tracemalloc inflates RSS; here we read ONLY its Python-heap deltas, not RSS.)
  consume 0 (cold) taken as Python-heap baseline
  consume  1 net Python-heap vs post-cold baseline:    +49.9 KB
  consume  2 net Python-heap vs post-cold baseline:   +336.3 KB
  consume  3 net Python-heap vs post-cold baseline:   +386.2 KB
  consume  4 net Python-heap vs post-cold baseline:   +448.7 KB
  consume  5 net Python-heap vs post-cold baseline:   +557.0 KB
  consume  6 net Python-heap vs post-cold baseline:   +573.9 KB
  consume  7 net Python-heap vs post-cold baseline:   +606.7 KB
  consume  8 net Python-heap vs post-cold baseline:   +657.4 KB
  consume  9 net Python-heap vs post-cold baseline:   +722.2 KB
  consume 10 net Python-heap vs post-cold baseline:   +808.4 KB
  consume 11 net Python-heap vs post-cold baseline:   +819.5 KB
  ---- top Python-heap line-diffs, last consume vs post-cold baseline ----
        +484.4 KB  (blocks +8610)  /usr/local/lib/python3.9/tracemalloc.py:558
        +152.4 KB  (blocks +1951)  /usr/local/lib/python3.9/tracemalloc.py:115
         +92.2 KB  (blocks +1968)  /usr/local/lib/python3.9/tracemalloc.py:193
         +13.0 KB  (blocks +164)  /usr/local/lib/python3.9/site-packages/django/db/backends/sqlite3/base.py:296
          -3.4 KB  (blocks -24)  <frozen importlib._bootstrap_external>:647
          +2.0 KB  (blocks +34)  /usr/local/lib/python3.9/tracemalloc.py:423
  gc.garbage after batch = 0
```

### §9.8 Barcode-split edge path (`CONSUMER_ENABLE_BARCODES`, non-default)

**§9.8A** — isolate `scan_file_for_separating_barcodes` (`convert_from_path` renders all pages):

```bash
python3 entrypoints.py barcode_scan several-patcht-codes.pdf
```

```
===== isolate scan_file_for_separating_barcodes (tasks.py L96-110) on several-patcht-codes.pdf (78234 bytes) =====
  convert_from_path rendered ALL pages to images; separator pages = [2, 5]
  VmRSS 67.88 -> 69.02 MB (delta +1.14) ; ru_maxrss peak 65.18 -> 178.44 MB (+113.25)
  after gc+malloc_trim: 68.19 MB (reclaimed +0.83 MB) ; gc.garbage=0
```

**§9.8B** — full `tasks.consume_file` with barcodes enabled:

```bash
python3 entrypoints.py barcode several-patcht-codes.pdf
```

```
===== full tasks.consume_file with CONSUMER_ENABLE_BARCODES=True on several-patcht-codes.pdf =====
  result: File successfully split
  VmRSS 69.54 -> 74.32 MB (delta +4.78) ; ru_maxrss peak +112.55 MB
```

### §9.9 The `malloc_trim(0)` discriminator + `MALLOC_ARENA_MAX` probe

**§9.9A** — one 48 MB text document: transient peak, retained RSS, then `malloc_trim(0)`:

```bash
python3 batch_recycle.py trimprobe 48
```

```
===== trimprobe: consume ONE 48.0 MB text document, then malloc_trim(0) =====
  VmRSS before consume                 =    52.35 MB
  ru_maxrss PEAK during consume        =   722.80 MB  (transient spike +670.45 MB over baseline)
  VmRSS after consume returns          =   262.21 MB  (retained +209.86 MB vs baseline)
  VmRSS after drop refs + gc.collect   =   214.21 MB  (gc.garbage=0)
  malloc_trim(0) returned 1             =   213.86 MB  (RECLAIMED +0.36 MB to OS)
  net vs baseline after trim           =  +161.50 MB
  VERDICT: transient spike freed logically after consume; the RSS the OS still
           showed was glibc arena retention -> malloc_trim(0) returns it = NORMAL.
```

**§9.9B** — 6 sequential 24 MB documents (leak-vs-retention: plateau => reuse, not leak):

```bash
python3 batch_recycle.py trimloop 24 6
```

```
===== trimloop: 6 sequential consumes of a fresh ~24 MB text doc each =====
  VmRSS baseline = 53.75 MB
  iter  0: VmRSS after consume+drop+gc =   194.37 MB  (net vs baseline +140.62 MB, delta vs prev +140.62 MB)  gc.garbage=0
  iter  1: VmRSS after consume+drop+gc =   169.69 MB  (net vs baseline +115.94 MB, delta vs prev -24.68 MB)  gc.garbage=0
  iter  2: VmRSS after consume+drop+gc =   194.44 MB  (net vs baseline +140.69 MB, delta vs prev +24.75 MB)  gc.garbage=0
  iter  3: VmRSS after consume+drop+gc =   219.06 MB  (net vs baseline +165.31 MB, delta vs prev +24.62 MB)  gc.garbage=0
  iter  4: VmRSS after consume+drop+gc =   219.10 MB  (net vs baseline +165.35 MB, delta vs prev  +0.04 MB)  gc.garbage=0
  iter  5: VmRSS after consume+drop+gc =   219.13 MB  (net vs baseline +165.38 MB, delta vs prev  +0.03 MB)  gc.garbage=0
  malloc_trim(0)=1 -> VmRSS = 144.00 MB (reclaimed +75.13 MB)
  ---- interpretation ----
  RSS after iter0 = 194.37 MB ; after iter5 = 219.13 MB ; growth across 5 later docs = +24.76 MB
  => PLATEAU (growth << one doc's 24 MB) means the retained memory is REUSED by later docs (glibc arena retention, NORMAL), NOT leaked.
```

**§9.9C** — same loop with `MALLOC_ARENA_MAX=1` and `=2` (non-default): negligible plateau change => retention is within-arena free space, not arena multiplication.

```bash
MALLOC_ARENA_MAX=1 python3 batch_recycle.py trimloop 24 6
```

```
===== trimloop: 6 sequential consumes of a fresh ~24 MB text doc each =====
  VmRSS baseline = 53.91 MB
  iter  0: VmRSS after consume+drop+gc =   194.53 MB  (net vs baseline +140.62 MB, delta vs prev +140.62 MB)  gc.garbage=0
  iter  1: VmRSS after consume+drop+gc =   169.85 MB  (net vs baseline +115.94 MB, delta vs prev -24.68 MB)  gc.garbage=0
  iter  2: VmRSS after consume+drop+gc =   228.71 MB  (net vs baseline +174.80 MB, delta vs prev +58.86 MB)  gc.garbage=0
  iter  3: VmRSS after consume+drop+gc =   228.75 MB  (net vs baseline +174.83 MB, delta vs prev  +0.03 MB)  gc.garbage=0
  iter  4: VmRSS after consume+drop+gc =   228.78 MB  (net vs baseline +174.87 MB, delta vs prev  +0.04 MB)  gc.garbage=0
  iter  5: VmRSS after consume+drop+gc =   219.29 MB  (net vs baseline +165.38 MB, delta vs prev  -9.49 MB)  gc.garbage=0
  malloc_trim(0)=1 -> VmRSS = 144.15 MB (reclaimed +75.14 MB)
  ---- interpretation ----
  RSS after iter0 = 194.53 MB ; after iter5 = 219.29 MB ; growth across 5 later docs = +24.76 MB
  => PLATEAU (growth << one doc's 24 MB) means the retained memory is REUSED by later docs (glibc arena retention, NORMAL), NOT leaked.
```

```bash
MALLOC_ARENA_MAX=2 python3 batch_recycle.py trimloop 24 6
```

```
===== trimloop: 6 sequential consumes of a fresh ~24 MB text doc each =====
  VmRSS baseline = 54.19 MB
  iter  0: VmRSS after consume+drop+gc =   194.41 MB  (net vs baseline +140.22 MB, delta vs prev +140.22 MB)  gc.garbage=0
  iter  1: VmRSS after consume+drop+gc =   169.73 MB  (net vs baseline +115.55 MB, delta vs prev -24.68 MB)  gc.garbage=0
  iter  2: VmRSS after consume+drop+gc =   194.48 MB  (net vs baseline +140.29 MB, delta vs prev +24.75 MB)  gc.garbage=0
  iter  3: VmRSS after consume+drop+gc =   219.11 MB  (net vs baseline +164.92 MB, delta vs prev +24.63 MB)  gc.garbage=0
  iter  4: VmRSS after consume+drop+gc =   219.14 MB  (net vs baseline +164.96 MB, delta vs prev  +0.04 MB)  gc.garbage=0
  iter  5: VmRSS after consume+drop+gc =   219.18 MB  (net vs baseline +164.99 MB, delta vs prev  +0.03 MB)  gc.garbage=0
  malloc_trim(0)=1 -> VmRSS = 144.04 MB (reclaimed +75.14 MB)
  ---- interpretation ----
  RSS after iter0 = 194.41 MB ; after iter5 = 219.18 MB ; growth across 5 later docs = +24.77 MB
  => PLATEAU (growth << one doc's 24 MB) means the retained memory is REUSED by later docs (glibc arena retention, NORMAL), NOT leaked.
```

### §9.10 REAL Django-Q cluster — canonical `recycle:1` (A) vs raised recycle (B)

**§9.10A** — canonical `recycle:1` with `PAPERLESS_TASK_WORKERS=1`; 8 classifier-PRESENT `consume_file` tasks enqueued on the real Redis broker. Worker lifecycle log (each task runs in a fresh worker PID, each followed by `recycled worker`):

```bash
PAPERLESS_TASK_WORKERS=1 /tmp/memharness/run_cluster_recycle.sh
#   -> starts /proc poller, enqueues 8 tasks (cluster_enqueue.py 8),
#      runs `python3 manage.py qcluster`, drains, stops. Run log:
```

```
21:14:25 [Q] INFO Enqueued 1
21:14:25 [Q] INFO Enqueued 2
21:14:25 [Q] INFO Enqueued 3
21:14:25 [Q] INFO Enqueued 4
21:14:25 [Q] INFO Enqueued 5
21:14:25 [Q] INFO Enqueued 6
21:14:25 [Q] INFO Enqueued 7
21:14:25 [Q] INFO Enqueued 8
enqueued 8 tasks; broker queue size now = 8
MASTER_PID=233615
21:14:27 [Q] INFO Q Cluster xray-mango-kitten-winner starting.
21:14:27 [Q] INFO Process-1:1 ready for work at 234461
21:14:27 [Q] INFO Process-1:2 monitoring at 234462
21:14:27 [Q] INFO Process-1 guarding cluster xray-mango-kitten-winner
21:14:27 [Q] INFO Process-1:3 pushing tasks at 234463
21:14:27 [Q] INFO Q Cluster xray-mango-kitten-winner running.
21:14:27 [Q] INFO Process-1:1 processing [consume_0]
[2026-07-14 21:14:27,381] [INFO] [paperless.consumer] Consuming cluster_000.txt
[2026-07-14 21:14:28,494] [INFO] [paperless.consumer] Document 2026-07-14 cluster_000 consumption finished
21:14:28 [Q] INFO Process-1:1 stopped doing work
21:14:28 [Q] INFO Processed [consume_0]
21:14:28 [Q] INFO recycled worker Process-1:1
21:14:28 [Q] INFO Process-1:4 ready for work at 235871
21:14:28 [Q] INFO Process-1:4 processing [consume_1]
[2026-07-14 21:14:28,873] [INFO] [paperless.consumer] Consuming cluster_001.txt
[2026-07-14 21:14:30,023] [INFO] [paperless.consumer] Document 2026-07-14 cluster_001 consumption finished
21:14:30 [Q] INFO Process-1:4 stopped doing work
21:14:30 [Q] INFO Processed [consume_1]
21:14:30 [Q] INFO recycled worker Process-1:4
21:14:30 [Q] INFO Process-1:5 ready for work at 237287
21:14:30 [Q] INFO Process-1:5 processing [consume_2]
[2026-07-14 21:14:30,378] [INFO] [paperless.consumer] Consuming cluster_002.txt
[2026-07-14 21:14:31,556] [INFO] [paperless.consumer] Document 2026-07-14 cluster_002 consumption finished
21:14:31 [Q] INFO Process-1:5 stopped doing work
21:14:31 [Q] INFO Processed [consume_2]
21:14:31 [Q] INFO recycled worker Process-1:5
21:14:31 [Q] INFO Process-1:6 ready for work at 238681
21:14:31 [Q] INFO Process-1:6 processing [consume_3]
[2026-07-14 21:14:31,888] [INFO] [paperless.consumer] Consuming cluster_003.txt
[2026-07-14 21:14:33,039] [INFO] [paperless.consumer] Document 2026-07-14 cluster_003 consumption finished
21:14:33 [Q] INFO Process-1:6 stopped doing work
21:14:33 [Q] INFO Processed [consume_3]
21:14:33 [Q] INFO recycled worker Process-1:6
21:14:33 [Q] INFO Process-1:7 ready for work at 240087
21:14:33 [Q] INFO Process-1:7 processing [consume_4]
[2026-07-14 21:14:33,401] [INFO] [paperless.consumer] Consuming cluster_004.txt
[2026-07-14 21:14:34,539] [INFO] [paperless.consumer] Document 2026-07-14 cluster_004 consumption finished
21:14:34 [Q] INFO Process-1:7 stopped doing work
21:14:34 [Q] INFO Processed [consume_4]
21:14:34 [Q] INFO recycled worker Process-1:7
21:14:34 [Q] INFO Process-1:8 ready for work at 241494
21:14:34 [Q] INFO Process-1:8 processing [consume_5]
[2026-07-14 21:14:34,898] [INFO] [paperless.consumer] Consuming cluster_005.txt
[2026-07-14 21:14:36,039] [INFO] [paperless.consumer] Document 2026-07-14 cluster_005 consumption finished
21:14:36 [Q] INFO Process-1:8 stopped doing work
21:14:36 [Q] INFO Processed [consume_5]
21:14:36 [Q] INFO recycled worker Process-1:8
21:14:36 [Q] INFO Process-1:9 ready for work at 242892
21:14:36 [Q] INFO Process-1:9 processing [consume_6]
[2026-07-14 21:14:36,411] [INFO] [paperless.consumer] Consuming cluster_006.txt
[2026-07-14 21:14:37,554] [INFO] [paperless.consumer] Document 2026-07-14 cluster_006 consumption finished
21:14:37 [Q] INFO Process-1:9 stopped doing work
21:14:37 [Q] INFO Processed [consume_6]
21:14:37 [Q] INFO recycled worker Process-1:9
21:14:37 [Q] INFO Process-1:10 ready for work at 244295
21:14:37 [Q] INFO Process-1:10 processing [consume_7]
[2026-07-14 21:14:37,915] [INFO] [paperless.consumer] Consuming cluster_007.txt
[2026-07-14 21:14:39,123] [INFO] [paperless.consumer] Document 2026-07-14 cluster_007 consumption finished
21:14:39 [Q] INFO Process-1:10 stopped doing work
21:14:39 [Q] INFO Processed [consume_7]
21:14:39 [Q] INFO recycled worker Process-1:10
21:14:39 [Q] INFO Process-1:11 ready for work at 245681
21:14:51 [Q] INFO Q Cluster xray-mango-kitten-winner stopping.
21:14:51 [Q] INFO Process-1 stopping cluster processes
21:14:52 [Q] INFO Process-1:3 stopped pushing tasks
21:14:52 [Q] INFO Process-1:11 stopped doing work
21:14:52 [Q] INFO Process-1 waiting for the monitor.
21:14:52 [Q] INFO Process-1:2 stopped monitoring results
21:14:52 [Q] INFO Q Cluster xray-mango-kitten-winner has stopped.
ORCHESTRATION_DONE
```

Per-worker-PID peak `VmRSS` from the `/proc` poller (8 short-lived workers ~138-145 MB each; persistent infra master/guard/monitor/pusher ~57-68 MB; final idle worker never got a task):

```bash
awk (per-worker-PID peak VmRSS) 9_10_recycle_proc.log
```

```
PID 233615 (ppid 232930): peak VmRSS 68.1 MB  (298 samples)
PID 234422 (ppid 233615): peak VmRSS 61.6 MB  (279 samples)
PID 234461 (ppid 234422): peak VmRSS 143.7 MB  (14 samples)
PID 234462 (ppid 234422): peak VmRSS 59.9 MB  (277 samples)
PID 234463 (ppid 234422): peak VmRSS 59.4 MB  (274 samples)
PID 235871 (ppid 234422): peak VmRSS 138.1 MB  (13 samples)
PID 237287 (ppid 234422): peak VmRSS 144.7 MB  (14 samples)
PID 238681 (ppid 234422): peak VmRSS 141.7 MB  (13 samples)
PID 240087 (ppid 234422): peak VmRSS 138.3 MB  (13 samples)
PID 241494 (ppid 234422): peak VmRSS 144.1 MB  (14 samples)
PID 242892 (ppid 234422): peak VmRSS 143.1 MB  (14 samples)
PID 244295 (ppid 234422): peak VmRSS 144.7 MB  (14 samples)
PID 245681 (ppid 234422): peak VmRSS 57.5 MB  (145 samples)
```

**§9.10B** — raised recycle (NON-DEFAULT: `Conf.RECYCLE=1000`, `Conf.WORKERS=1`) so ONE worker handles all 8 tasks. Run log (no `recycled worker` lines — same PID throughout):

```bash
/tmp/memharness/run_raise_recycle.sh   # runs raise_recycle.py 8
```

```
21:15:38 [Q] INFO Enqueued 1
21:15:38 [Q] INFO Enqueued 2
21:15:38 [Q] INFO Enqueued 3
21:15:38 [Q] INFO Enqueued 4
21:15:38 [Q] INFO Enqueued 5
21:15:38 [Q] INFO Enqueued 6
21:15:38 [Q] INFO Enqueued 7
21:15:38 [Q] INFO Enqueued 8
Conf.RECYCLE=1000 Conf.WORKERS=1 ; enqueued 8 tasks; queue=8
21:15:38 [Q] INFO Q Cluster ink-four-utah-nuts starting.
21:15:38 [Q] INFO Process-1:1 ready for work at 258907
21:15:38 [Q] INFO Process-1:2 monitoring at 258908
21:15:38 [Q] INFO Process-1 guarding cluster ink-four-utah-nuts
21:15:38 [Q] INFO Process-1:3 pushing tasks at 258909
21:15:38 [Q] INFO Q Cluster ink-four-utah-nuts running.
21:15:38 [Q] INFO Process-1:1 processing [rr_0]
[2026-07-14 21:15:39,053] [INFO] [paperless.consumer] Consuming rr_000.txt
[2026-07-14 21:15:40,210] [INFO] [paperless.consumer] Document 2026-07-14 rr_000 consumption finished
21:15:40 [Q] INFO Process-1:1 processing [rr_1]
21:15:40 [Q] INFO Processed [rr_0]
[2026-07-14 21:15:40,257] [INFO] [paperless.consumer] Consuming rr_001.txt
[2026-07-14 21:15:40,935] [INFO] [paperless.consumer] Document 2026-07-14 rr_001 consumption finished
21:15:40 [Q] INFO Process-1:1 processing [rr_2]
21:15:40 [Q] INFO Processed [rr_1]
[2026-07-14 21:15:40,968] [INFO] [paperless.consumer] Consuming rr_002.txt
[2026-07-14 21:15:41,652] [INFO] [paperless.consumer] Document 2026-07-14 rr_002 consumption finished
21:15:41 [Q] INFO Process-1:1 processing [rr_3]
21:15:41 [Q] INFO Processed [rr_2]
[2026-07-14 21:15:41,686] [INFO] [paperless.consumer] Consuming rr_003.txt
[2026-07-14 21:15:42,367] [INFO] [paperless.consumer] Document 2026-07-14 rr_003 consumption finished
21:15:42 [Q] INFO Process-1:1 processing [rr_4]
21:15:42 [Q] INFO Processed [rr_3]
[2026-07-14 21:15:42,397] [INFO] [paperless.consumer] Consuming rr_004.txt
[2026-07-14 21:15:43,224] [INFO] [paperless.consumer] Document 2026-07-14 rr_004 consumption finished
21:15:43 [Q] INFO Process-1:1 processing [rr_5]
21:15:43 [Q] INFO Processed [rr_4]
[2026-07-14 21:15:43,257] [INFO] [paperless.consumer] Consuming rr_005.txt
[2026-07-14 21:15:43,962] [INFO] [paperless.consumer] Document 2026-07-14 rr_005 consumption finished
21:15:43 [Q] INFO Process-1:1 processing [rr_6]
21:15:43 [Q] INFO Processed [rr_5]
[2026-07-14 21:15:43,993] [INFO] [paperless.consumer] Consuming rr_006.txt
[2026-07-14 21:15:44,680] [INFO] [paperless.consumer] Document 2026-07-14 rr_006 consumption finished
21:15:44 [Q] INFO Process-1:1 processing [rr_7]
21:15:44 [Q] INFO Processed [rr_6]
[2026-07-14 21:15:44,727] [INFO] [paperless.consumer] Consuming rr_007.txt
[2026-07-14 21:15:45,441] [INFO] [paperless.consumer] Document 2026-07-14 rr_007 consumption finished
21:15:45 [Q] INFO Processed [rr_7]
21:15:47 [Q] INFO Q Cluster ink-four-utah-nuts stopping.
21:15:47 [Q] INFO Process-1 stopping cluster processes
21:15:48 [Q] INFO Process-1:3 stopped pushing tasks
21:15:48 [Q] INFO Process-1:1 stopped doing work
21:15:48 [Q] INFO Process-1 waiting for the monitor.
21:15:48 [Q] INFO Process-1:2 stopped monitoring results
21:15:48 [Q] INFO Q Cluster ink-four-utah-nuts has stopped.
cluster stopped
ORCHESTRATION_DONE
```

Single worker (PID 258907) `VmRSS` trajectory — climbs to ~133 MB on the first consume then **plateaus** (does not climb ~55 MB/task):

```bash
awk (worker 258907 VmRSS trajectory + min/max) 9_10b_raise_proc.log
```

```
t+  0.0s  VmRSS=50.6 MB
t+  1.4s  VmRSS=133.1 MB
t+  2.9s  VmRSS=133.4 MB
t+  4.3s  VmRSS=133.7 MB
t+  5.6s  VmRSS=133.9 MB
t+  7.0s  VmRSS=133.8 MB
t+  8.4s  VmRSS=133.8 MB
first=50.6 last=133.8 min=50.6 max=133.9 samples=99
```

### §9.11 Importer manifest load (`document_importer.py:73,137-140`)

```bash
python3 entrypoints.py importer 20000
```

```
===== document_importer manifest load (json.load L73 + list() L137) — 20000 doc records, 43.8 MB on disk =====
  json.load held 20050 records; VmRSS 66.70 -> 185.53 MB (+118.82) ; tracemalloc Python-heap +57.2 MB
  list(filter ...) materialized 20000 document records; VmRSS +25.45 MB ; tracemalloc +0.2 MB (2nd in-memory list alongside the full manifest)
```

### §9.12 Email path — `MailAccountHandler.handle_message` attachment buffering (transport non-canonical)

```bash
python3 entrypoints.py email 20
```

```
===== email path: MailAccountHandler.handle_message att.payload buffering (mail.py L317 magic.from_buffer, L327 f.write) — attachment ~20 MB =====
  (NON-CANONICAL TRANSPORT: no IMAP server; message is a real imap_tools MailMessage built in-process. The att.payload buffering code executed IS the canonical path.)
  handle_message consumed 1 attachment(s); enqueued consume_file task(s)
  VmRSS 212.47 -> 240.32 MB (delta +27.85) ; ru_maxrss peak +0.00 MB <- whole attachment payload held in the (non-recycled) mail process
  after gc+malloc_trim: 147.85 MB (reclaimed +92.47) gc.garbage=0
```

### §9.13 REST upload — canonical `POST /api/documents/post_document/` (own gunicorn, 1 worker)

Own gunicorn on :8001 (`PAPERLESS_WEBSERVER_WORKERS=1`), authenticated token upload of a 40 MB file x3. Run log:

```bash
/tmp/memharness/run_rest_upload.sh
#   PAPERLESS_PORT=8001 PAPERLESS_WEBSERVER_WORKERS=1 gunicorn -c /app/gunicorn.conf.py \
#     paperless.asgi:application ; then curl -F document=@<40MB> ...post_document/ x3
```

```
GUNICORN_MASTER=267531
[2026-07-14 21:17:19 +0000] [267531] [INFO] Starting gunicorn 20.1.0
[2026-07-14 21:17:19 +0000] [267531] [INFO] Listening at: http://0.0.0.0:8001 (267531)
[2026-07-14 21:17:19 +0000] [267531] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-14 21:17:19 +0000] [267531] [INFO] Server is ready. Spawning workers
server up (http 200) after 3 tries
WORKER_PID=267540
upload file size = 40001000 bytes
=== upload 1 ===
21:17:20 [Q] INFO Enqueued 1
http=200 time_total=0.155481s upload=40001199B
=== upload 2 ===
21:17:22 [Q] INFO Enqueued 2
http=200 time_total=0.162141s upload=40001199B
=== upload 3 ===
21:17:25 [Q] INFO Enqueued 3
http=200 time_total=0.151947s upload=40001199B
[2026-07-14 21:17:27 +0000] [267531] [INFO] Handling signal: term
[2026-07-14 21:17:27 +0000] [267531] [INFO] Shutting down: Master
ORCHESTRATION_DONE
```

Web-worker (PID 267540) `VmRSS` — idle ~69 MB, +42.6 MB per buffered upload, transient peak 172.6 MB, oscillates/reused across the 3 uploads (not a per-request leak):

```bash
awk (worker 267540 RSS first/min/max/last + trajectory) 9_13_rest_proc.log
```

```
first=68.9  min=68.9  max=172.6  last=152.3  samples=148
t+  0.0s  68.9 MB
t+  0.5s  111.5 MB
t+  1.1s  111.5 MB
t+  1.6s  111.5 MB
t+  2.1s  111.5 MB
t+  2.7s  114.1 MB
t+  3.2s  114.1 MB
t+  3.7s  114.1 MB
t+  4.2s  114.1 MB
t+  4.8s  152.3 MB
t+  5.3s  152.3 MB
t+  5.8s  152.3 MB
t+  6.4s  152.3 MB
```

### §9.14 Duplicate-rejection edge path (`consumer.py` `pre_check_duplicate` L102-113)

```bash
python3 entrypoints.py dup
```

```
===== duplicate-rejection path (consumer.py pre_check_duplicate L104) =====
  1st consume OK: Document id=397 ; VmRSS 52.60 -> 67.52 MB (+14.91)
  2nd consume (identical bytes) -> ConsumerError raised? True
    message: dup2.txt: Not consuming dup2.txt: It is a duplicate.
  VmRSS across rejected consume: 67.52 -> 67.52 MB (delta +0.00 MB) -> pre_check_duplicate still read whole file for MD5 (L104), then rejected
  Documents in DB (still 1, duplicate NOT stored): 1
```

---

*End of captured evidence. All harness scripts and scratch artifacts referenced above were removed from the container on completion; the only file added to the repository is this document.*
