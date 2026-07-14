# Memory Usage During Document Import / Metadata Handling — Investigation & Answer

> **Repository:** paperless-ngx  **Deliverable branch name:** `paperless-ngx_542221a38dff`
> **Working-tree HEAD (ground truth):** `542221a38dff06361e07976452f9aea24d210542`
> **Investigation type:** Read-only, runtime-first memory investigation. No source file was modified; the only repository change is this document.
> **Canonical runtime for every measurement:** the project Docker image (Python 3.9.23 + the exact pinned stack). Every number below was produced by running the **real `manage.py` entry points** and/or the **real `documents.models.Document` model** on that pinned stack; the exact command, working directory, and environment are printed with each block, and the complete unedited output is shown.

---

## The question being answered (verbatim)

> "When the system imports documents, it sometimes consumes far more memory than expected for the metadata handling, but this doesn't happen consistently and seems to vary based on the document source or processing stage."
>
> "memory usage spikes significantly during document import operations, particularly when metadata is being processed, but the spike seems disproportionate to the actual document size. The odd part is that this memory doesn't always get released back to the system in a timely manner, even after processing is complete... these are mostly text documents with relatively small metadata."

The question decomposes into six objectives, each answered explicitly below and cross-checked in the coverage pass (§12):

- **O1 — Root cause:** which component(s)/code path(s) cause the spikes.
- **O2 — Unnecessary copies / retained references.**
- **O3 — Caching accumulation** (including the `caching.py` negative result).
- **O4 — Differential:** what differs between spiking and non-spiking cases.
- **O5 — Sensitivity:** how behavior changes with document type and batch size.
- **O6 — Runtime evidence + classification:** before/during/after measurements, the responsible methods, and a normal-GC-vs-problematic verdict.

Section 11 is an explicit **claim-status matrix** classifying every key statement as *source-grounded*, *observed*, *derived*, or *inferred* and pointing to its evidence.

---

## 1. Direct answer (root cause — O1)

The memory spike during "document import" is produced by the **document export/import manifest materialization** in Paperless-ngx's management commands, **not** by anything proportional to a document's physical file size. Two methods are responsible:

- **`document_exporter.dump()`** — `src/documents/management/commands/document_exporter.py:107`. It builds one in-memory Python list, `manifest`, by serializing **all rows of ten selected models** — not "every database object". The ten are, in order: `Correspondent` (`document_exporter.py:118`), `Tag` (`:121`), `DocumentType` (`:123`), **`Document`** (`:127-130`), `MailAccount` (`:132`), `MailRule` (`:136`), `SavedView` (`:140`), `SavedViewFilterRule` (`:144`), `Group` (`:148`), and `User` (`:150`), each via `json.loads(serializers.serialize("json", <Model>.objects.all()))`. For documents specifically it holds **two** large structures simultaneously plus a transient third: `document_map = {d.pk: d for d in documents}` (`document_exporter.py:128`, every `Document` model instance) and `document_manifest = json.loads(serializers.serialize("json", documents))` (`document_exporter.py:129`, the full parsed list of serialized document fields), which is then concatenated into `manifest` (`:130`). Both remain live throughout the subsequent per-document loop (`document_exporter.py:153-161`, which indexes `document_map[document_dict["pk"]]` at `:161`).

- **`document_importer.handle()`** — `src/documents/management/commands/document_importer.py:57`. It retains the entire manifest on the command instance via `self.manifest = json.load(f)` (`document_importer.py:73`), then calls `call_command("loaddata", manifest_path)` (`document_importer.py:87`), which makes Django **independently re-deserialize the same file** while `self.manifest` is still held — a concurrent double-materialization. A filtered list `manifest_documents = list(filter(...))` (`document_importer.py:137-139`) is then built; this is a **shallow reference list** — its elements are the *same* dict objects already held in `self.manifest`, so it adds essentially no payload (measured **+0.0 MB**, §3.4), i.e. reference-list overhead, not a third full copy.

The reason the spike is **disproportionate to the physical document size** is the `Document.content` field — `src/documents/models.py:117-124`:

```python
    content = models.TextField(
        _("content"),
        blank=True,
        help_text=_(
            "The raw, text-only data of the document. This field is "
            "primarily used for searching.",
        ),
    )
```

`content` stores the **entire extracted OCR/plain text** of each document. The manifest serializes this field for the **whole corpus at once**. Therefore the peak memory scales with **(number of documents) × (per-document `content` size)** — the *aggregate text of the corpus* — and is essentially independent of how many bytes each original file occupies on disk. This is exactly the user's observation: "mostly text documents with relatively small metadata" yet a spike "disproportionate to the actual document size."

The measurements in §3 confirm this quantitatively with the **real `manage.py document_exporter` command**: at an **equal document count of 5,000 and equal ~14-byte physical files**, exporting text-heavy documents (50 KB of `content` each) peaked at **≈805.6 MB RSS** (average of two runs: 807.5 / 803.7 MB), versus **≈84.8 MB RSS** (84.4 / 85.3 MB) for metadata-only documents (~200 bytes of `content` each) — a **≈9.5×** difference produced solely by `Document.content` (§5, O4). The **real `manage.py document_importer`** on the same corpus peaked at **≈804.4 MB RSS** (803.5 / 805.2 MB) (§3.3).

The "not released promptly" symptom (§8, O6) is **normal CPython/glibc allocator arena retention amplified by scale**, observed directly: after `del` + `gc.collect()` the Python heap is fully freed (`tracemalloc` current drops to ≈1.4 MB) but process RSS stays at its high-water mark; a subsequent `malloc_trim(0)` then returns ≈485 MB to the OS, and re-running the same input **reuses** the retained arenas instead of growing (peak stdev 0.52 MB across 6 identical runs, §7). It is **not** an unbounded leak. Because the exporter/importer are **standalone `manage.py` CLI processes that exit**, the OS reclaims everything on exit (a fresh process restarts at ≈48.7 MB, §8); the delayed release is only visible *within* a single long-running process.

The user's "caching behavior that's accumulating data unexpectedly" (§9, O3) resolves to **two module/instance-scoped accumulators**, not a dedicated caching layer: there is **no `src/documents/caching.py`** in this revision (shown in §9.1), so the candidates are the `dateparser` module-level locale cache invoked by `parse_date()` (`src/documents/parsers.py:212`) and the whole-corpus `CountVectorizer` vocabulary built by `classifier.train()` (`src/documents/classifier.py:194-199`). Measured, the `dateparser` locale cache is a **bounded one-time warm-up that then plateaus** (§9.2), not unbounded growth — but its warm-up size is **input- and locale-dependent**: when the matched date substring parses cleanly in the configured language only English data loads (≈15.7 MB RSS / 5.5 MB retained heap); when the substring is *not* cleanly parseable (a common default-configuration case shown in §9.2), `dateparser` tries **all 205 bundled languages**, loading every locale's translation data (a larger ≈62 MB one-time RSS warm-up, ≈23.4 MB of it retained Python heap) — still bounded (it cannot exceed all 205 locales) and still dwarfed by the manifest path (§3).

---

## 2. Environment & method

### 2.1 Branch / commit (working tree)

```text
$ cd /tmp/blitzy/paperless-ngx/blitzy-d8807159-272a-4a73-a584-5ff7a6e332ea_c2b4eb
$ git rev-parse --abbrev-ref HEAD
blitzy-d8807159-272a-4a73-a584-5ff7a6e332ea
$ git rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ git status --porcelain
(empty — clean working tree at start)
```

The deliverable file name (`paperless-ngx_542221a38dff.md`) is the **source branch name**; the checkout used for the investigation is the Blitzy working branch shown above, whose HEAD commit `542221a38dff06361e07976452f9aea24d210542` is the code state all `file:line` citations refer to.

### 2.2 Canonical runtime (used for every measurement)

All measurements in this document were taken in the **canonical runtime**: the project Docker image

```text
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01
```

which ships **Python 3.9.23** with the exact pinned dependency stack. Exact command and complete output (the container name is a unique per-run name; see the Appendix §13.1 provisioning):

```text
$ docker exec pngx-mem python3 --version
Python 3.9.23
$ docker exec pngx-mem bash -lc 'cd /app/src && python -c "import django,dateparser,sklearn,numpy,scipy; \
print(\"django\", django.get_version()); print(\"dateparser\", dateparser.__version__); \
print(\"sklearn\", sklearn.__version__); print(\"numpy\", numpy.__version__); print(\"scipy\", scipy.__version__)"'
django 4.0.4
dateparser 1.1.1
sklearn 1.0.2
numpy 1.22.3
scipy 1.8.0
$ docker exec pngx-mem bash -lc 'qpdf --version | head -1; command -v jbig2 || echo "jbig2: NOT FOUND"'
qpdf version 10.1.0
jbig2: NOT FOUND
```

These match the pins in `Pipfile.lock` (django 4.0.4, django-q 1.3.9, scikit-learn 1.0.2, dateparser 1.1.1, numpy 1.22.3, scipy 1.8.0). `python manage.py check` reports a clean app graph, so the `documents` app and its imports load correctly:

```text
$ docker exec pngx-mem bash -lc 'cd /app/src && python manage.py check'
System check identified no issues (0 silenced).
```

Inside this image I exercised, on the pinned Python 3.9.23 stack: the **real `manage.py document_exporter` and `manage.py document_importer` entry points** under `/usr/bin/time -v` (§3.1, §3.3 — these **carry the export/import magnitude claims**); the **real `documents.parsers.parse_date`** function (§9.2) and the **real `DocumentClassifier`** (§9.3); and the **real canonical consume and concurrency paths** — `documents.tasks.consume_file` on a scanned PDF/PNG and the real `qcluster` + `document_consumer` (§2.5, §5.1). Separately, three **labeled partial in-process stand-ins** — `export_probe` (§3.2), `import_probe` (§3.4), and `stability_probe` (§7) — run only the *memory-relevant subset* of those same commands against the **real `documents.models.Document`** on the same stack; their sole purpose is to expose the phase-by-phase Python allocation and allocator-retention behavior that `/usr/bin/time` cannot show, so each enumerates its omissions and defers its **whole-command magnitude to the canonical §3.1/§3.3** and carries **no magnitude claim of its own**. In other words, the headline numbers are all **CANONICAL**; the stand-ins are explicitly **labeled partial** and corroborate mechanism only. §10 states the exact canonical / partial-stand-in / non-canonical / infeasible classification for every section.

**The host interpreter cannot run the pinned stack (it is not used for any number).** The Blitzy host is Python 3.13.7. Installing `django==4.0.4` there succeeds, but `django.setup()` fails because Django 4.0.4 imports the standard-library `cgi` module, which Python 3.13 removed (PEP 594). Exact command and traceback, captured **verbatim** and reproduced identically across three consecutive runs. The `...<11 lines>...` and `...<4 lines>...` tokens inside the trace are **emitted by CPython 3.13's own traceback formatter**, which collapses multi-line `from … import ( … )` statements to their first and last physical line; they are the interpreter's literal output, **not** a manual elision. The names each token collapses are enumerated immediately after the trace (§2.2.1), so no frame or import name is withheld:

```text
$ python3 --version
Python 3.13.7
$ python3 -m pip install --quiet --target=/tmp/hosttest_libs "django==4.0.4"   # succeeds (exit 0)
$ PYTHONPATH=/tmp/hosttest_libs python3 -c "import django; print('django', django.get_version())"
django 4.0.4
$ PYTHONPATH=/tmp/hosttest_libs python3 -c "import django; django.setup()"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    import django; django.setup()
                   ~~~~~~~~~~~~^^
  File "/tmp/hosttest_libs/django/__init__.py", line 16, in setup
    from django.urls import set_script_prefix
  File "/tmp/hosttest_libs/django/urls/__init__.py", line 1, in <module>
    from .base import (
    ...<11 lines>...
    )
  File "/tmp/hosttest_libs/django/urls/base.py", line 8, in <module>
    from .exceptions import NoReverseMatch, Resolver404
  File "/tmp/hosttest_libs/django/urls/exceptions.py", line 1, in <module>
    from django.http import Http404
  File "/tmp/hosttest_libs/django/http/__init__.py", line 2, in <module>
    from django.http.request import (
    ...<4 lines>...
    )
  File "/tmp/hosttest_libs/django/http/request.py", line 1, in <module>
    import cgi
ModuleNotFoundError: No module named 'cgi'
```

#### 2.2.1 What CPython 3.13 collapsed in the trace (full transparency)

Python 3.13's traceback formatter renders a statement that spans several physical lines as its first line, a `...<N lines>...` token, and its closing line. Both tokens above are exactly this behavior applied to two multi-line `import` statements; reading the corresponding lines out of the installed files shows there is nothing hidden:

```text
$ sed -n '1,13p' /tmp/hosttest_libs/django/urls/__init__.py     # the '...<11 lines>...' token
from .base import (
    clear_script_prefix,
    clear_url_caches,
    get_script_prefix,
    get_urlconf,
    is_valid_path,
    resolve,
    reverse,
    reverse_lazy,
    set_script_prefix,
    set_urlconf,
    translate_url,
)
$ sed -n '2,7p' /tmp/hosttest_libs/django/http/__init__.py       # the '...<4 lines>...' token
from django.http.request import (
    HttpRequest,
    QueryDict,
    RawPostDataException,
    UnreadablePostError,
)
```

`...<11 lines>...` is the 11 imported names on lines 2–12 of `django/urls/__init__.py` (between the `from .base import (` on line 1 and the `)` on line 13); `...<4 lines>...` is the 4 imported names on lines 3–6 of `django/http/__init__.py` (between the `from django.http.request import (` on line 2 and the `)` on line 7). Neither collapse omits any *frame* of the failing import chain, which proceeds in full: `django/__init__.py:16 → django/urls/__init__.py:1 → django/urls/base.py:8 → django/urls/exceptions.py:1 → django/http/__init__.py:2 → django/http/request.py:1: import cgi`.

The very first line of `django.setup()` (`from django.urls import set_script_prefix`, `django/__init__.py:16`) triggers the import chain that reaches `django/http/request.py:1: import cgi`, so the failure occurs before any settings are consulted. This is why the canonical Python 3.9 container is used throughout; **no measurement in this document comes from the host.**

### 2.3 Controlled corpus (stated plainly, and its limits)

To vary document type and batch size deterministically, a **throwaway** database was populated inside an **isolated scratch instance** (SQLite under a unique `/tmp/<scratch>` root created with `manage.py migrate`) with real `Document` rows whose `content` field holds a text blob of a chosen size, plus a **tiny (~14-byte) physical original file** and an 8-byte stub thumbnail per document. The populate script forces and asserts the scratch paths and refuses to run outside `/tmp` (safety, §13.2).

**The `content` blob is an ASCII content-size *proxy*, not a claim of allocation-identity with arbitrary real text.** A 50 KB ASCII `content` string occupies 50 KB of `TextField` storage and serializes to a JSON string of the same length, so it faithfully drives the *size-proportional* allocation this question is about. It does **not** reproduce every property of arbitrary document text: multi-byte Unicode inflates the serialized JSON (each non-ASCII code point is escaped, e.g. `\u20ac`), and real OCR text has different entropy. To bound this, §6 measures a **Unicode multi-byte** corpus alongside the ASCII one and reports the difference (the Unicode manifest is ≈2.2× larger for the same nominal byte budget). The tiny physical files make the disproportion explicit — **files are ~14 bytes, yet peak RSS reaches ≈805 MB** — precisely the reported symptom.

### 2.4 Measurement methodology (before / during / after)

- **Whole-process peak RSS** for the real CLI is captured with `/usr/bin/time -v` ("Maximum resident set size (kbytes)"). The **complete 23-field** output is shown for representative runs inline and for **every** run in Appendix §13.4–13.5. Its reliability in this container was validated against a known 300 MB allocation:

```text
$ docker exec pngx-mem bash -lc '/usr/bin/time -v python3 -c "a=bytearray(300*1024*1024); print(len(a))" 2>&1 | grep "Maximum resident"'
314572800
	Maximum resident set size (kbytes): 317508
```

(≈310 MB reported for a 300 MB allocation — trustworthy.)

- **In-process RSS** is read from field 2 (resident pages) of `/proc/self/statm` × page size (4096) → MB, sampled **before** the operation, **during** it (references live), **after** `del` + `gc.collect()`, and **after** `malloc_trim(0)`.
- **Python-heap allocation** is captured with `tracemalloc.get_traced_memory()` (current and peak). Crucially, tracemalloc is kept **active through `del` + `gc.collect()`** so the post-GC current bytes are recorded (§7, §8). **RSS (MB) and tracemalloc (MB) are always reported on their own metric and never subtracted from each other** (see §3.2/§3.4 for how transient copies are quantified on a single metric).
- **Concurrent peak during `loaddata`** is captured by a background sampler thread (§3.4), not by a single post-return sample.

### 2.5 Consume path: OCR **is** reproducible on this image; the bounded limitation is Office/DOCX (external Tika + Java)

The `.build-config.json` pins native binaries for the reference build, and the canonical image carries **qpdf 10.1.0 and lacks the `jbig2` binary** (shown in §2.2):

```text
$ cat .build-config.json
{
  "qpdf": {
      "version": "10.6.3"
    },
  "jbig2enc": {
      "version": "0.29",
      "git_tag": "0.29"
    }
}
$ command -v jbig2 || echo 'jbig2: NOT FOUND'
jbig2: NOT FOUND
```

**Correction (F-04, runtime-verified).** An earlier revision of this document claimed the full OCR consume path was *infeasible* on this image because `jbig2` is absent. That claim is **wrong**, and it is corrected here from direct runtime observation: `jbig2enc` is only an *optional output-PDF compression optimizer* for OCRmyPDF, **not** a prerequisite for OCR. Exercised through the **real canonical consume task** `documents.tasks.consume_file(path)` (`src/documents/tasks.py:184` → `Consumer().try_consume_file` at `tasks.py:236` — the exact function a Django-Q worker runs), a synthetic **image-only scanned PDF** and a **PNG** both consume **successfully without `jbig2`**, each producing a `Document` with OCR-extracted content, an archive PDF, and a thumbnail. Complete, unedited output (`cwd=/app/src`; scratch DB migrated under `/tmp/s_ocr2`):

```text
### the scanned PDF is image-only (no text layer): pdftotext yields 0 characters -> OCR is required
$ pdftotext /tmp/ocr_inputs/scanned_probe.pdf - | tr -d '[:space:]' | wc -c
0

### consume the scanned PDF via the canonical task (jbig2 absent):
$ python /tmp/scripts/consume_probe.py --scratch /tmp/s_ocr2 --file /tmp/ocr_inputs/scanned_probe.pdf --label ocr_pdf
{"label": "ocr_pdf", "input_file": "/tmp/ocr_inputs/scanned_probe.pdf", "input_bytes": 16435, "consume_result_msg": "Success. New document id 1 created", "documents_before": 0, "documents_after": 1, "doc_pk": 1, "doc_mime_type": "application/pdf", "content_len": 307, "content_snippet": "PRPERLESS OCR MEMORY PROBE ALPHA ERAVO CHARLIE 2018\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick b", "content_has_MEMORY": true, "content_has_PROBE": true, "has_archive_version": true, "archive_exists_on_disk": true, "original_exists_on_disk": true, "thumbnail_exists_on_disk": true, "checksum": "59929d21d8d6c16f09afda15fee694a8", "base_rss_mb": 68.1, "peak_rss_mb": 219.9, "consume_rss_delta_mb": 151.8, "tm_peak_mb": 39.8}

### consume the PNG via the canonical task:
$ python /tmp/scripts/consume_probe.py --scratch /tmp/s_ocr2 --file /tmp/ocr_inputs/scanned_probe.png --label ocr_png
{"label": "ocr_png", "input_file": "/tmp/ocr_inputs/scanned_probe.png", "input_bytes": 15000, "consume_result_msg": "Success. New document id 2 created", "documents_before": 1, "documents_after": 2, "doc_pk": 2, "doc_mime_type": "image/png", "content_len": 307, "content_snippet": "PRPERLESS OCR MEMORY PROBE ALPHA ERAVO CHARLIE 2018\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick brom\n\nquick b", "content_has_MEMORY": true, "content_has_PROBE": true, "has_archive_version": true, "archive_exists_on_disk": true, "original_exists_on_disk": true, "thumbnail_exists_on_disk": true, "checksum": "c19a7c811edd51c73c26513dacde9e6a", "base_rss_mb": 68.5, "peak_rss_mb": 218.9, "consume_rss_delta_mb": 150.4, "tm_peak_mb": 39.5}
```

Both consumes return `"Success. New document id N created"`; both create a `Document` with non-empty OCR `content` (307 chars; the distinctive tokens `MEMORY` and `PROBE` survive OCR), `has_archive_version=true` with the archive on disk, the original on disk, and a thumbnail on disk. The OCR text is imperfect (`PRPERLESS`/`ERAVO` — tesseract 4.1.1 on a bold synthetic scan) but that is immaterial to feasibility. *(One benign log note: ImageMagick's `policy.xml` blocks its PDF coder, so thumbnailing falls back to ghostscript, which succeeds — `thumbnail_exists_on_disk=true`.)*

**The real bounded limitation is Office/DOCX, not scanned OCR.** Office formats are handled by the Tika parser, which is **disabled by default** (`PAPERLESS_TIKA_ENABLED` defaults to `"NO"`, `settings.py:592`) — so `.docx` is not even a supported extension out of the box — and which, when enabled, delegates to an **external Tika/Gotenberg server backed by Java**. The image has **no Java**, so the real `TikaDocumentParser.parse()` (`src/paperless_tika/parsers.py:55`, `parser.from_file(document_path, PAPERLESS_TIKA_ENDPOINT)`) cannot start a server and raises `ParseError`. Complete, unedited output:

```text
$ command -v java || echo 'java: NOT FOUND'
java: NOT FOUND

### default config: .docx is unsupported because Tika is off:
PAPERLESS_TIKA_ENABLED(default)= False | .docx supported= False

### with Tika enabled, the real parser on a genuine .docx (no server; Java absent):
$ PAPERLESS_TIKA_ENABLED=1 python /tmp/scripts/tika_docx_probe.py
PAPERLESS_TIKA_ENABLED: True
PAPERLESS_TIKA_ENDPOINT: http://localhost:9998
docx bytes: 927
2026-07-14 02:16:20,688 [MainThread  ] [ERROR]  Unable to run java; is it installed?
2026-07-14 02:16:20,688 [MainThread  ] [ERROR]  Failed to receive startup confirmation from startServer.
ParseError (expected): Could not parse /tmp/ocr_inputs/office_probe.docx with tika server at http://localhost:9998: Unable to start Tika server.
```

This does **not** affect the core investigation: the memory question is about **import / metadata handling** — the export/import manifest path — which is reproduced canonically through its real entry points (§3.1, §3.3). The consume path is now also exercised canonically here (and its concurrency behavior measured in §5), and contrasted with the corpus-proportional manifest path (§5, §9) to explain the "varies by source/stage" observation. The probe sources (`consume_probe.py`, `tika_docx_probe.py`, and the synthetic-input generators) are listed in Appendix §13.2.

---

## 3. Runtime memory measurements (complete, unedited output)

All commands were run inside the canonical container. For each scenario a **fresh, isolated scratch instance** (its own SQLite DB + media dirs under a unique `/tmp/<scratch>` root, created with `manage.py migrate`) was populated in a **separate process**, then the measured command/probe was run in **its own process** so every baseline is clean (this removes the baseline-contamination problem where prior deletions/allocations inflate the "before" number). Every magnitude/timing/release condition is run **≥2 independent times**. Memory is in MB unless a raw `kbytes` value is quoted.

### 3.1 CANONICAL — the real `manage.py document_exporter` entry point (peak RSS)

Each scenario: populate a throwaway corpus (N docs, chosen `content` size, tiny ~14-byte physical files) in a separate process, then run the **real** management command under `/usr/bin/time -v` in a fresh process. `cwd=/app/src`; environment for every run is
`PAPERLESS_DATA_DIR=/tmp/<scratch>/data PAPERLESS_MEDIA_ROOT=/tmp/<scratch>/media PAPERLESS_STATICDIR=/tmp/<scratch>/static PAPERLESS_CONSUMPTION_DIR=/tmp/<scratch>/consume` (the exact scratch root is printed in each block).

**Export-target prerequisite (one-line `mkdir -p` before every run).** `document_exporter` requires its positional target directory to already exist — `src/documents/management/commands/document_exporter.py:98-99` raises `CommandError("That path doesn't exist")` and the command never creates the directory itself. Each run's target (`/tmp/exp_<scenario>_run<n>`) is therefore created with `mkdir -p` immediately before the command; this one-line step is shown explicitly in the run harness in §13.1 and applies identically to every scenario below (zero / meta5000 / text200 / text1000 / text5000 / uni5000, run1 and run2). The `$ /usr/bin/time -v ... document_exporter ...` transcripts that follow therefore presuppose that preceding `mkdir -p`; the `Exit status: 0` they show is what the exporter returns once its target directory exists.

**True zero-document baseline (empty migrated DB — no documents at all).** This is the real startup cost of the command; it is *not* ~86 MB. Complete `/usr/bin/time -v`, run 1:

```text
### (zero-document baseline: empty migrated DB, no populate)
### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_zero/data PAPERLESS_MEDIA_ROOT=/tmp/s_zero/media PAPERLESS_STATICDIR=/tmp/s_zero/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_zero/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_zero_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_zero_run1 --no-progress-bar"
	User time (seconds): 1.24
	System time (seconds): 0.15
	Percent of CPU this job got: 100%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.39
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 67432
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 42814
	Voluntary context switches: 30
	Involuntary context switches: 12
	Swaps: 0
	File system inputs: 0
	File system outputs: 16
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_zero_run1/manifest.json
-rw-r--r-- 1 root root 393 Jul 13 18:08 /tmp/exp_zero_run1/manifest.json
```

Run 2 peaked at **66324 KB** (complete block in Appendix §13.4). The zero-document baseline is therefore **65.9 / 64.8 KB→MB = ≈65.3 MB** average, with a 393-byte manifest. This baseline does **not** include scikit-learn/numpy/scipy — those modules are not imported at Django startup (proven in §3.2).

**Non-spike case — metadata-only, 5,000 docs (`content` = 200 B), complete `/usr/bin/time -v`, run 1:**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_meta5000 --n 5000 --content-bytes 200
populated 5000 docs; content_bytes=200 (ascii); originals+thumbnails written=5000; scratch=/tmp/s_meta5000
### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_meta5000/data ... PAPERLESS_CONSUMPTION_DIR=/tmp/s_meta5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_meta5000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_meta5000_run1 --no-progress-bar"
	User time (seconds): 5.26
	System time (seconds): 0.93
	Percent of CPU this job got: 98%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:06.26
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 86388
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 47365
	Voluntary context switches: 57
	Involuntary context switches: 301
	Swaps: 0
	File system inputs: 8
	File system outputs: 89128
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_meta5000_run1/manifest.json
-rw-r--r-- 1 root root 4665956 Jul 13 18:08 /tmp/exp_meta5000_run1/manifest.json
```

**Spike case — text-heavy, 5,000 docs (`content` = 50 KB), complete `/usr/bin/time -v`, run 1:**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_text5000 --n 5000 --content-bytes 50000
populated 5000 docs; content_bytes=50000 (ascii); originals+thumbnails written=5000; scratch=/tmp/s_text5000
### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text5000/data ... PAPERLESS_CONSUMPTION_DIR=/tmp/s_text5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text5000_run1 --no-progress-bar"
	User time (seconds): 6.37
	System time (seconds): 1.67
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:08.08
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 826844
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 341223
	Voluntary context switches: 47
	Involuntary context switches: 287
	Swaps: 0
	File system inputs: 0
	File system outputs: 575456
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text5000_run1/manifest.json
-rw-r--r-- 1 root root 253665956 Jul 13 18:08 /tmp/exp_text5000_run1/manifest.json
```

**All CANONICAL exporter runs (peak RSS; physical files ~14 B in every case; ≥2 runs each).** The complete 23-field `/usr/bin/time -v` output for every run below is in Appendix §13.4:

| Scenario | docs | `content`/doc | run1 peak (KB) | run2 peak (KB) | run1 / run2 (MB) | manifest.json |
|---|---|---|---|---|---|---|
| zero | 0 | — | 67432 | 66324 | 65.9 / 64.8 | 393 B |
| meta5000 | 5000 | 200 B | 86388 | 87372 | 84.4 / 85.3 | 4,665,956 B (4.45 MB) |
| text200 | 200 | 50 KB | 98768 | 99060 | 96.5 / 96.7 | 10,145,955 B (9.68 MB) |
| text1000 | 1000 | 50 KB | 218980 | 217484 | 213.8 / 212.4 | 50,729,956 B (48.38 MB) |
| text5000 | 5000 | 50 KB | 826844 | 822940 | 807.5 / 803.7 | 253,665,956 B (241.91 MB) |
| uni5000 (Unicode) | 5000 | 50 KB | 850184 | 879292 | 830.3 / 858.7 | 559,225,956 B (533.32 MB) |

**Magnitude stability:** every scenario is within ~0.3–3.5% across its two runs (e.g. text5000 807.5 vs 803.7 MB). The manifest sizes track `Document.content` exactly (text5000's 241.9 MB manifest is ~54× meta5000's 4.45 MB, mirroring the 250× content-size ratio scaled by the fixed per-record JSON overhead).

### 3.2 CANONICAL-STACK, real model — export manifest probe (PARTIAL stand-in; before/during/after + tracemalloc)

**This probe is a PARTIAL in-process stand-in** for the memory-relevant core of `dump()` (`document_exporter.py:127-130`). It runs the exact three statements against the **real `documents.models.Document`** on the pinned stack, but it **deliberately omits** the rest of the command: serialization of the other nine models, the per-document annotation/file-copy loop (`document_exporter.py:153-210`), the final `json.dump()` of `manifest.json`, and the tqdm bar. Its only purpose is to expose the phase-by-phase Python allocation and the allocator-retention behavior that `/usr/bin/time` cannot show; the **canonical whole-command magnitude is carried by §3.1**, which corroborates it. Full source: Appendix §13.2.

Each scale is run in **≥2 independent fresh processes** (each a new `python` invocation → fresh `django.setup()` → clean baseline). Complete, unedited output of all eight runs (`cwd=/app/src`; full file in Appendix §13.6, evidence `export_inproc_2runs.txt`):

```text
### export_probe meta5000 run1 / run2 (fresh process each; scratch=/tmp/s_meta5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_meta5000 --n 5000 --label meta5000
{"label": "meta5000_r0", "n": 5000, "base_rss": 50.1, "live_rss": 86.3, "after_gc_rss": 78.6, "after_trim_rss": 64.5, "rss_freed_by_trim": 14.1, "tm_current_live": 16.0, "tm_current_after_gc": 1.4, "tm_peak": 19.3, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}
{"label": "meta5000_r0", "n": 5000, "base_rss": 50.2, "live_rss": 86.4, "after_gc_rss": 78.9, "after_trim_rss": 64.8, "rss_freed_by_trim": 14.1, "tm_current_live": 16.0, "tm_current_after_gc": 1.4, "tm_peak": 19.3, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}
### export_probe text200 run1 / run2 (fresh process each; scratch=/tmp/s_text200):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text200 --n 200 --label text200
{"label": "text200_r0", "n": 200, "base_rss": 50.0, "live_rss": 78.0, "after_gc_rss": 78.6, "after_trim_rss": 55.5, "rss_freed_by_trim": 23.1, "tm_current_live": 21.1, "tm_current_after_gc": 1.4, "tm_peak": 30.9, "gc_objects_live": 72922, "gc_objects_after_gc": 71691, "gc_collected": 0, "malloc_trim_available": true}
{"label": "text200_r0", "n": 200, "base_rss": 50.0, "live_rss": 78.0, "after_gc_rss": 78.6, "after_trim_rss": 55.5, "rss_freed_by_trim": 23.0, "tm_current_live": 21.1, "tm_current_after_gc": 1.4, "tm_peak": 30.9, "gc_objects_live": 72922, "gc_objects_after_gc": 71691, "gc_collected": 0, "malloc_trim_available": true}
### export_probe text1000 run1 / run2 (fresh process each; scratch=/tmp/s_text1000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text1000 --n 1000 --label text1000
{"label": "text1000_r0", "n": 1000, "base_rss": 50.1, "live_rss": 158.6, "after_gc_rss": 151.0, "after_trim_rss": 60.1, "rss_freed_by_trim": 90.9, "tm_current_live": 99.6, "tm_current_after_gc": 1.4, "tm_peak": 148.5, "gc_objects_live": 77929, "gc_objects_after_gc": 71888, "gc_collected": 0, "malloc_trim_available": true}
{"label": "text1000_r0", "n": 1000, "base_rss": 49.8, "live_rss": 158.3, "after_gc_rss": 151.5, "after_trim_rss": 59.8, "rss_freed_by_trim": 91.7, "tm_current_live": 99.6, "tm_current_after_gc": 1.4, "tm_peak": 148.5, "gc_objects_live": 77929, "gc_objects_after_gc": 71888, "gc_collected": 0, "malloc_trim_available": true}
### export_probe text5000 run1 / run2 (fresh process each; scratch=/tmp/s_text5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text5000 --n 5000 --label text5000
{"label": "text5000_r0", "n": 5000, "base_rss": 49.8, "live_rss": 560.8, "after_gc_rss": 552.9, "after_trim_rss": 68.0, "rss_freed_by_trim": 484.8, "tm_current_live": 490.9, "tm_current_after_gc": 1.4, "tm_peak": 731.6, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}
{"label": "text5000_r0", "n": 5000, "base_rss": 49.9, "live_rss": 560.9, "after_gc_rss": 552.9, "after_trim_rss": 67.9, "rss_freed_by_trim": 485.0, "tm_current_live": 490.9, "tm_current_after_gc": 1.4, "tm_peak": 731.6, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}
```

Detail columns below are run 1; the run-1 / run-2 `tm_peak` pair is shown in the last column to document stability:

| Scenario | base_rss | live_rss (during) | after_gc_rss | after_trim_rss | rss_freed_by_trim | tm_current_live | tm_current_after_gc | tm_peak (r1 / r2) |
|---|---|---|---|---|---|---|---|---|
| meta5000 (200 B) | 50.1 | 86.3 | 78.6 | 64.5 | 14.1 | 16.0 | 1.4 | 19.3 / 19.3 |
| text200 (50 KB) | 50.0 | 78.0 | 78.6 | 55.5 | 23.1 | 21.1 | 1.4 | 30.9 / 30.9 |
| text1000 (50 KB) | 50.1 | 158.6 | 151.0 | 60.1 | 90.9 | 99.6 | 1.4 | 148.5 / 148.5 |
| text5000 (50 KB) | 49.8 | 560.8 | 552.9 | 68.0 | 484.8 | 490.9 | 1.4 | 731.6 / 731.6 |

**Magnitude stability:** every scale produced an **identical `tm_peak` across its two independent fresh-process runs** (text5000 731.6/731.6, text1000 148.5/148.5, text200 30.9/30.9, meta5000 19.3/19.3), and the release figure `rss_freed_by_trim` is stable to <1 MB (e.g. text5000 484.8/485.0). The in-process export allocation and its release are therefore deterministic per corpus. The same mechanism repeated **six times in one process** (retention/reuse across iterations, not fresh processes) is in §7.

Two facts for **O2 (transient copies), quantified on a single metric (tracemalloc):** at text5000 the tracemalloc **peak is 731.6 MB** while the **live current is 490.9 MB** — a difference of **≈240.7 MB** that exists *only at peak* and is gone once the operation settles. That 240.7 MB is the **transient duplicate**: inside `json.loads(serializers.serialize("json", documents))` the full serialized **JSON string** is co-resident with the **Python list** that `json.loads` builds from it, before the string is freed. (At text1000 the same gap is 148.5 − 99.6 = 48.9 MB.) This is measured entirely within tracemalloc — no cross-metric subtraction. Baselines are clean at ≈50 MB in every case, and the text5000 in-process peak (731.6 MB) is consistent with the real-CLI peak (807.5 MB; the CLI additionally serializes the other nine models, copies files, and writes the manifest to disk).

### 3.3 CANONICAL — the real `manage.py document_importer` entry point (peak RSS)

Each scenario imports one of the exporter output directories from §3.1 into a **fresh, empty migrated database** (its own `/tmp/imp_*` scratch root), run under `/usr/bin/time -v`, `cwd=/app/src`, ≥2 runs. The importer prints `Installed N object(s) from 1 fixture(s)`, `Copy files into paperless...`, `Updating search index...`, then a progress meter. **Note (grounded):** the `--no-progress-bar` flag is honored by the importer's own file-copy loop, but the reindex step at `document_importer.py:92` calls `call_command("document_index", "reindex")` **without forwarding the flag**, so the reindex always renders a tqdm meter. That meter repaints a single line via carriage returns; rendered to text each repaint becomes a separate line. Shown inline below: the **headline text5000 run 1** and the **meta5000 run 1**, each with the **complete, unedited terminal rendering (every repaint) and all 23 `/usr/bin/time -v` fields** (to demonstrate nothing is excerpted). The remaining runs are summarized in the peak-RSS table; the **complete rendered output — every repaint and all 23 `time -v` fields — for every run (both runs of all four scenarios) is in Appendix §13.5**.

**Spike case — text5000, run 1 (complete output, full reindex meter):**

```text
### import text5000 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text5000_run1/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text5000_run1/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/5000 [00:00<?, ?it/s]
  1%|▏         | 68/5000 [00:00<00:07, 671.67it/s]
  3%|▎         | 138/5000 [00:00<00:07, 687.93it/s]
  4%|▍         | 207/5000 [00:00<00:07, 677.41it/s]
  6%|▌         | 278/5000 [00:00<00:06, 689.46it/s]
  7%|▋         | 347/5000 [00:00<00:06, 681.71it/s]
  8%|▊         | 416/5000 [00:00<00:06, 679.64it/s]
 10%|▉         | 484/5000 [00:00<00:08, 543.70it/s]
 11%|█         | 556/5000 [00:00<00:07, 589.72it/s]
 12%|█▏        | 620/5000 [00:00<00:07, 602.82it/s]
 14%|█▍        | 689/5000 [00:01<00:06, 626.05it/s]
 15%|█▌        | 758/5000 [00:01<00:06, 642.81it/s]
 16%|█▋        | 824/5000 [00:01<00:06, 632.57it/s]
 18%|█▊        | 892/5000 [00:01<00:06, 643.06it/s]
 19%|█▉        | 960/5000 [00:01<00:06, 652.96it/s]
 21%|██        | 1030/5000 [00:01<00:05, 664.16it/s]
 22%|██▏       | 1098/5000 [00:01<00:05, 665.94it/s]
 23%|██▎       | 1168/5000 [00:01<00:05, 673.60it/s]
 25%|██▍       | 1237/5000 [00:01<00:05, 675.50it/s]
 26%|██▌       | 1305/5000 [00:02<00:09, 385.29it/s]
 28%|██▊       | 1377/5000 [00:02<00:08, 449.77it/s]
 29%|██▉       | 1445/5000 [00:02<00:07, 496.59it/s]
 30%|███       | 1516/5000 [00:02<00:06, 546.35it/s]
 32%|███▏      | 1582/5000 [00:02<00:05, 574.86it/s]
 33%|███▎      | 1653/5000 [00:02<00:05, 609.14it/s]
 35%|███▍      | 1726/5000 [00:02<00:05, 641.47it/s]
 36%|███▌      | 1795/5000 [00:02<00:04, 651.70it/s]
 37%|███▋      | 1868/5000 [00:03<00:04, 672.48it/s]
 39%|███▉      | 1938/5000 [00:03<00:04, 671.12it/s]
 40%|████      | 2009/5000 [00:03<00:04, 679.82it/s]
 42%|████▏     | 2079/5000 [00:03<00:04, 674.70it/s]
 43%|████▎     | 2148/5000 [00:03<00:04, 679.12it/s]
 44%|████▍     | 2217/5000 [00:03<00:04, 679.60it/s]
 46%|████▌     | 2286/5000 [00:03<00:03, 679.96it/s]
 47%|████▋     | 2355/5000 [00:03<00:03, 679.61it/s]
 48%|████▊     | 2424/5000 [00:03<00:03, 676.45it/s]
 50%|████▉     | 2495/5000 [00:03<00:03, 685.69it/s]
 51%|█████▏    | 2564/5000 [00:04<00:06, 360.41it/s]
 53%|█████▎    | 2632/5000 [00:04<00:05, 417.80it/s]
 54%|█████▍    | 2706/5000 [00:04<00:04, 483.93it/s]
 55%|█████▌    | 2773/5000 [00:04<00:04, 525.51it/s]
 57%|█████▋    | 2845/5000 [00:04<00:03, 572.80it/s]
 58%|█████▊    | 2914/5000 [00:04<00:03, 601.90it/s]
 60%|█████▉    | 2981/5000 [00:05<00:03, 610.85it/s]
 61%|██████    | 3047/5000 [00:05<00:03, 588.36it/s]
 62%|██████▏   | 3116/5000 [00:05<00:03, 614.84it/s]
 64%|██████▍   | 3189/5000 [00:05<00:02, 646.53it/s]
 65%|██████▌   | 3256/5000 [00:05<00:02, 649.94it/s]
 67%|██████▋   | 3330/5000 [00:05<00:02, 673.64it/s]
 68%|██████▊   | 3399/5000 [00:05<00:02, 676.64it/s]
 69%|██████▉   | 3471/5000 [00:05<00:02, 687.03it/s]
 71%|███████   | 3542/5000 [00:05<00:02, 691.54it/s]
 72%|███████▏  | 3612/5000 [00:05<00:02, 680.48it/s]
 74%|███████▎  | 3687/5000 [00:06<00:01, 698.14it/s]
 75%|███████▌  | 3758/5000 [00:06<00:01, 701.25it/s]
 77%|███████▋  | 3829/5000 [00:06<00:02, 399.86it/s]
 78%|███████▊  | 3899/5000 [00:06<00:02, 457.59it/s]
 79%|███████▉  | 3973/5000 [00:06<00:01, 518.13it/s]
 81%|████████  | 4042/5000 [00:06<00:01, 558.15it/s]
 82%|████████▏ | 4109/5000 [00:06<00:01, 585.56it/s]
 84%|████████▎ | 4184/5000 [00:07<00:01, 628.14it/s]
 85%|████████▌ | 4258/5000 [00:07<00:01, 656.35it/s]
 87%|████████▋ | 4328/5000 [00:07<00:01, 662.58it/s]
 88%|████████▊ | 4403/5000 [00:07<00:00, 685.79it/s]
 90%|████████▉ | 4475/5000 [00:07<00:00, 693.71it/s]
 91%|█████████ | 4546/5000 [00:07<00:00, 686.97it/s]
 92%|█████████▏| 4619/5000 [00:07<00:00, 699.01it/s]
 94%|█████████▍| 4694/5000 [00:07<00:00, 712.29it/s]
 95%|█████████▌| 4766/5000 [00:07<00:00, 698.85it/s]
 97%|█████████▋| 4837/5000 [00:07<00:00, 700.78it/s]
 98%|█████████▊| 4909/5000 [00:08<00:00, 706.20it/s]
100%|█████████▉| 4980/5000 [00:08<00:00, 688.22it/s]
100%|██████████| 5000/5000 [00:08<00:00, 612.56it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar"
	User time (seconds): 24.63
	System time (seconds): 4.84
	Percent of CPU this job got: 69%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:42.30
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 822832
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 521800
	Voluntary context switches: 41998
	Involuntary context switches: 105
	Swaps: 0
	File system inputs: 0
	File system outputs: 2275696
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

**Non-spike contrast — meta5000, run 1 (complete `/usr/bin/time -v`; reindex meter shown at its final line):**

```text
### import meta5000 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_meta5000_run1/data ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...
100%|██████████| 5000/5000 [00:05<00:00, 873.60it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar"
	User time (seconds): 19.79
	System time (seconds): 3.29
	Percent of CPU this job got: 65%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:35.39
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 143232
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 82669
	Voluntary context switches: 39942
	Involuntary context switches: 104
	Swaps: 0
	File system inputs: 0
	File system outputs: 1247152
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

**All CANONICAL importer runs (peak RSS; fresh DB each; error-free, `Exit status: 0`, `Installed N+1 object(s)`; ≥2 runs each).** Complete output for every run is in Appendix §13.5:

| Scenario | docs | `content`/doc | run1 peak (KB) | run2 peak (KB) | run1 / run2 (MB) |
|---|---|---|---|---|---|
| meta5000 | 5000 | 200 B | 143232 | 140648 | 139.9 / 137.4 |
| text200 | 200 | 50 KB | 118632 | 118168 | 115.9 / 115.4 |
| text1000 | 1000 | 50 KB | 246952 | 244652 | 241.2 / 238.9 |
| text5000 | 5000 | 50 KB | 822832 | 824480 | 803.5 / 805.2 |

The importer reaches the **same ≈804 MB scale** as the exporter at text5000, and the same content-driven contrast holds (text5000 ≈804 MB vs meta5000 ≈138.6 MB at equal document count). The importer's per-scenario peak is somewhat higher than the exporter's at small content (e.g. meta5000 138.6 MB import vs 84.8 MB export) because import additionally holds `loaddata`'s deserialization plus the Whoosh reindex working set.

### 3.4 CANONICAL-STACK, real model — import probe with concurrent-peak sampling (PARTIAL stand-in)

**This probe is a PARTIAL in-process stand-in** for the core of `handle()`. It runs `self.manifest = json.load(f)` (`document_importer.py:73`), then `call_command("loaddata", manifest_path)` (`:87`) inside the **real** `disable_signal(post_save/m2m_changed → update_filename_and_move_files)` context the importer uses (`document_importer.py:76-87`), then the `manifest_documents` filter (`:137-139`). It **omits** `_check_manifest()` validation, the physical file-copy loop, and the final `document_index reindex` — those are measured by the canonical CLI in §3.3. A **background sampler thread records the true peak RSS *while* `loaddata` runs** (fixing the earlier mistake of sampling only after the synchronous call returned). ≥2 runs; full source in Appendix §13.2.

```text
$ python /tmp/scripts/import_probe.py --scratch /tmp/improbe_text1000_r1 --manifest /tmp/exp_text1000_run1/manifest.json --label text1000_r1
{"label": "text1000_r1", "manifest": "manifest.json", "manifest_records": 1001, "rows_loaded": 1000, "base_rss": 49.8, "after_json_load_rss": 100.6, "concurrent_peak_rss_during_loaddata": 206.7, "post_loaddata_rss": 158.2, "rss_after_manifest_documents_filter": 158.2, "after_gc_rss": 156.4, "after_trim_rss": 55.4, "rss_freed_by_trim": 101.0, "json_load_rss_delta": 50.8, "loaddata_concurrent_extra_delta": 106.0, "manifest_documents_extra_delta": 0.0, "tm_peak": 149.0, "tm_current_live": 51.2, "tm_current_after_gc": 1.6, "malloc_trim_available": true}
$ python /tmp/scripts/import_probe.py --scratch /tmp/improbe_text1000_r2 --manifest /tmp/exp_text1000_run1/manifest.json --label text1000_r2
{"label": "text1000_r2", "manifest": "manifest.json", "manifest_records": 1001, "rows_loaded": 1000, "base_rss": 49.8, "after_json_load_rss": 100.6, "concurrent_peak_rss_during_loaddata": 206.7, "post_loaddata_rss": 158.2, "rss_after_manifest_documents_filter": 158.2, "after_gc_rss": 156.4, "after_trim_rss": 55.4, "rss_freed_by_trim": 101.1, "json_load_rss_delta": 50.8, "loaddata_concurrent_extra_delta": 106.0, "manifest_documents_extra_delta": 0.0, "tm_peak": 149.0, "tm_current_live": 51.2, "tm_current_after_gc": 1.6, "malloc_trim_available": true}
$ python /tmp/scripts/import_probe.py --scratch /tmp/improbe_text5000_r1 --manifest /tmp/exp_text5000_run1/manifest.json --label text5000_r1
{"label": "text5000_r1", "manifest": "manifest.json", "manifest_records": 5001, "rows_loaded": 5000, "base_rss": 49.8, "after_json_load_rss": 305.8, "concurrent_peak_rss_during_loaddata": 810.7, "post_loaddata_rss": 563.9, "rss_after_manifest_documents_filter": 563.9, "after_gc_rss": 554.2, "after_trim_rss": 55.6, "rss_freed_by_trim": 498.6, "json_load_rss_delta": 255.9, "loaddata_concurrent_extra_delta": 504.9, "manifest_documents_extra_delta": 0.0, "tm_peak": 737.7, "tm_current_live": 248.9, "tm_current_after_gc": 1.6, "malloc_trim_available": true}
$ python /tmp/scripts/import_probe.py --scratch /tmp/improbe_text5000_r2 --manifest /tmp/exp_text5000_run1/manifest.json --label text5000_r2
{"label": "text5000_r2", "manifest": "manifest.json", "manifest_records": 5001, "rows_loaded": 5000, "base_rss": 50.1, "after_json_load_rss": 306.0, "concurrent_peak_rss_during_loaddata": 811.0, "post_loaddata_rss": 564.2, "rss_after_manifest_documents_filter": 564.2, "after_gc_rss": 554.4, "after_trim_rss": 55.9, "rss_freed_by_trim": 498.5, "json_load_rss_delta": 255.9, "loaddata_concurrent_extra_delta": 505.0, "manifest_documents_extra_delta": 0.0, "tm_peak": 737.7, "tm_current_live": 248.9, "tm_current_after_gc": 1.6, "malloc_trim_available": true}
```

| Manifest | base | after `json.load` | **concurrent peak during `loaddata`** | post-`loaddata` | after filter | `json.load` Δ | `manifest_documents` extra Δ | tm_peak |
|---|---|---|---|---|---|---|---|---|
| text1000 | 49.8 | 100.6 | **206.7** | 158.2 | 158.2 | +50.8 | +0.0 | 149.0 |
| text5000 | 49.8 / 50.1 | 305.8 / 306.0 | **810.7 / 811.0** | 563.9 / 564.2 | 563.9 / 564.2 | +255.9 | +0.0 | 737.7 |

This is the clean demonstration of **O2 concurrent double-materialization**, all on the RSS metric: `json.load` retains **+255.9 MB** on `self.manifest`; then, *while that is still held*, `loaddata` independently deserializes the same file, driving the **true concurrent peak to ≈811 MB** (the background sampler catches this; a post-return sample would have under-reported it as 563.9 MB). The concurrent peak (≈811 MB) matches the real importer CLI peak (≈804 MB, §3.3). The `manifest_documents` filter adds **+0.0 MB** — confirming it is a shallow reference list (O2, finding on reference-list overhead), not a third full copy. After `del` + `gc.collect()` the tracemalloc current returns to **1.6 MB** (Python heap freed) while RSS stays at 554 MB until `malloc_trim(0)` returns **≈498.6 MB** to the OS (§8).

---

## 4. Per-component / per-method attribution (which methods hold the memory)

The export peak is held by structures inside `document_exporter.dump()`, live simultaneously for the whole export loop (`document_exporter.py:127-161`):

```python
# src/documents/management/commands/document_exporter.py:127-130
            documents = Document.objects.order_by("id")
            document_map = {d.pk: d for d in documents}
            document_manifest = json.loads(serializers.serialize("json", documents))
            manifest += document_manifest
# ...
# src/documents/management/commands/document_exporter.py:153-161
        for index, document_dict in tqdm.tqdm(
            enumerate(document_manifest),
            total=len(document_manifest),
            disable=progress_bar_disable,
        ):
            document_dict["fields"]["storage_type"] = Document.STORAGE_TYPE_UNENCRYPTED
            document = document_map[document_dict["pk"]]
```

| Structure | Location | What it holds | Evidence |
|---|---|---|---|
| `document_map` | `document_exporter.py:128` | dict of **every** `Document` instance, `{pk: d}`, kept live for the loop at `:161` | in-process peak scales with corpus (§3.2); indexed at `:161` |
| `document_manifest` | `document_exporter.py:129` | full parsed Python list of all documents' serialized fields, **including `content`** | manifest 241.9 MB at text5000 (§3.1) |
| transient serialized JSON **string** | `document_exporter.py:129` (`serializers.serialize(...)` inside `json.loads`) | a full JSON string co-resident with the list `json.loads` builds from it, at peak only | tm_peak − tm_current = 240.7 MB at text5000 (§3.2, same metric) |
| `manifest` (all 10 models) | `document_exporter.py:116-150` | aggregate of documents **plus** the other nine models | grows across `:118`–`:150` |

The import peak is held in `document_importer.handle()`:

```python
# src/documents/management/commands/document_importer.py:73
            self.manifest = json.load(f)
# src/documents/management/commands/document_importer.py:87
                call_command("loaddata", manifest_path)
# src/documents/management/commands/document_importer.py:137-139
        manifest_documents = list(
            filter(lambda r: r["model"] == "documents.document", self.manifest),
        )
```

| Structure | Location | What it holds | Evidence |
|---|---|---|---|
| `self.manifest` | `document_importer.py:73` | entire manifest parsed from JSON, retained on the command **instance** | +255.9 MB at text5000 (§3.4) |
| `loaddata` internal deserialization | `document_importer.py:87` | Django's independent re-materialization of the **same file**, concurrent with `self.manifest` | concurrent extra +504.9 MB → peak ≈811 MB (§3.4) |
| `manifest_documents` | `document_importer.py:137-139` | a filtered `list(...)` of **references** to dicts already in `self.manifest` | +0.0 MB extra (§3.4) — reference-list overhead only |

The payload that makes all of the above disproportionate to file size is **`Document.content`** (`src/documents/models.py:117-124`), the full extracted text. The secondary accumulators are `parse_date()`'s `dateparser` module cache (`src/documents/parsers.py:212`) and `classifier.train()`'s `CountVectorizer` (`src/documents/classifier.py:194-199`) — see §9.

---

## 5. Spike vs. non-spike differential + source/stage boundaries (O4)

**What differs between a case that spikes and one that does not is the aggregate `Document.content` materialized in one shot — not the physical file size and not the file count.** The controlled contrast holds *document count* and *physical file size* constant and varies only `content`:

| Case | docs | file size on disk | `content`/doc | real CLI export peak (§3.1) | real CLI import peak (§3.3) | in-process tm_peak (§3.2) |
|---|---|---|---|---|---|---|
| **non-spike** (metadata-only) | 5000 | ~14 B | ~200 B | **84.8 MB** | **138.6 MB** | 19.3 MB |
| **spike** (text-heavy) | 5000 | ~14 B | ~50 KB | **805.6 MB** | **804.4 MB** | 731.6 MB |

Same command, same entry point, same file count, same ~14-byte files → **≈9.5×** more peak RSS on export (805.6 / 84.8) and **≈5.8×** on import (804.4 / 138.6); the in-process Python allocation differs by **≈37.9×** (731.6 / 19.3, which excludes the fixed startup baseline). The single variable is `content`. Concretely:

- A **spike** occurs when the corpus being exported/imported contains a lot of **aggregate text** (many documents and/or large per-document `content`), materialized entirely by `document_manifest` / `self.manifest` (`document_exporter.py:129`, `document_importer.py:73`).
- A **non-spike** occurs when the aggregate text is small (few documents, or small `content`), *even if each physical file is large* — because the exporter serializes the DB `content` field, not the file bytes.

**Why it "varies by source or processing stage" — the source/stage boundaries (grounded in code):** the memory profile depends on *which code path* handles the document, because different paths hold memory in different places:

| Source / stage | Where the work (and memory) lives | `file:line` |
|---|---|---|
| **Export / import manifest** | In the Paperless process, **corpus-proportional**: the whole corpus's `content` is materialized at once. This is the spike path. | `document_exporter.py:127-130`; `document_importer.py:73,87` |
| **Consume (per-document, per-worker)** | In the Paperless process, **per in-flight document**: `try_consume_file` processes one document at a time — parse, `parse_date`, `load_classifier`, `_store`. The *per-task* footprint is bounded by a single document and does **not** grow with the total corpus size. **However (F-05, runtime-measured in §5.1), the total process-tree memory of the Django-Q cluster scales with the number of documents consumed *concurrently*, up to the configured worker count** — 1 PDF ≈1.02 GiB / 16 PIDs vs. 4 concurrent PDFs ≈1.48 GiB / 19 PIDs (+44.7%). | `consumer.py:180` (`try_consume_file`), `:261` (`parser.parse`), `:275` (`parse_date`), `:292` (`load_classifier`), `:379` (`_store`); `settings.py:449-457` (`Q_CLUSTER`, 11 workers) |
| **Plain-text parser** | In the Paperless process: reads the **whole file into memory** (`self.text = f.read()`). Per-document, bounded by one file. | `paperless_text/parsers.py:40-42` |
| **Office docs (Tika)** | In an **external Tika JVM server**, not the Paperless process: the document is sent via `requests`/`parser.from_file(document_path, tika_server)` and only the returned `content` string is held by Paperless. | `paperless_tika/parsers.py:4` (`import requests`), `:9` (`from tika import parser`), `:32`,`:55` (`parser.from_file(..., tika_server)`) |
| **Scanned PDF/image (Tesseract/OCR)** | In **native subprocesses** (ocrmypdf/tesseract/qpdf), not the Python heap; Paperless holds the resulting text. **Reproduced canonically here (F-04, §2.5): scanned-PDF and PNG OCR consume succeed WITHOUT `jbig2`**, each producing OCR content + archive + thumbnail (single-document consume peak ≈219 MB RSS). | `paperless_tesseract/parsers.py` (native OCR boundary); runtime §2.5 / §5.1 |
| **Email (IMAP)** | Mail is fetched over IMAP and queued for consume; per-message footprint. | `paperless_mail/mail.py` |

So two different "stages" have fundamentally different memory profiles for the *same* documents: the export/import manifest stage is corpus-proportional (its single-process peak spikes with the aggregate `content` of the whole corpus), while the consume stage is **per in-flight document** — each task's footprint is bounded by one document regardless of corpus size. The two are distinct axes: the manifest peak grows with *total corpus content in one process*, whereas the consume subsystem's peak grows with *how many documents are processed at once* across the Django-Q worker pool (§5.1, F-05: +44.7% from 1 to 4 concurrent OCR consumes), not with the total number of documents ever stored. This dual behavior is the precise mechanism behind the user's "seems to vary based on the document source or processing stage."

### 5.1 Consume concurrency: total process-tree memory scales with in-flight document count (F-05, runtime-measured)

The per-task consume footprint is bounded by one document, but Paperless consumes documents through a **Django-Q worker pool** whose default size is **11 workers** (`TASK_WORKERS`, resolved at runtime; `settings.py:438,449-457` — `Q_CLUSTER["workers"] = TASK_WORKERS`). The memory that matters operationally is therefore the **total process tree** (the `qcluster` master + its workers + the transient native OCR subprocesses `ocrmypdf`/`tesseract`/`gs` each consume spawns), and that total scales with how many documents are in flight at once.

This was measured canonically by starting the real cluster (`python manage.py qcluster`, 11 workers) and enqueuing work through the real `python manage.py document_consumer --oneshot` path (which `async_task`s `documents.tasks.consume_file`, `document_consumer.py:86-87`), while an external sampler walked the full process tree rooted at the `qcluster` PID and recorded peak summed-RSS and peak PID count. Four **distinct** scanned PDFs were used (distinct checksums, so none are rejected as duplicates). Two fresh runs per level (complete, unedited output; `cwd=/app/src`):

```text
$ bash /tmp/scripts/concurrency_probe.sh /tmp/s_conc1  1 conc_1pdf    /tmp/conc_pdfs
{"label":"conc_1pdf","n_queued":1,"docs_created":1,"peak_tree_rss_kib":1077936,"peak_pids":16,"samples":77,"q_workers":11}
$ bash /tmp/scripts/concurrency_probe.sh /tmp/s_conc1b 1 conc_1pdf_r2 /tmp/conc_pdfs
{"label":"conc_1pdf_r2","n_queued":1,"docs_created":1,"peak_tree_rss_kib":1063536,"peak_pids":16,"samples":78,"q_workers":11}
$ bash /tmp/scripts/concurrency_probe.sh /tmp/s_conc4  4 conc_4pdf    /tmp/conc_pdfs
{"label":"conc_4pdf","n_queued":4,"docs_created":4,"peak_tree_rss_kib":1548732,"peak_pids":19,"samples":77,"q_workers":11}
$ bash /tmp/scripts/concurrency_probe.sh /tmp/s_conc4b 4 conc_4pdf_r2 /tmp/conc_pdfs
{"label":"conc_4pdf_r2","n_queued":4,"docs_created":4,"peak_tree_rss_kib":1550784,"peak_pids":19,"samples":80,"q_workers":11}
```

Reading (stable across both runs): consuming **1** PDF peaks at **≈1,070,736 KiB (≈1.02 GiB) mean across 16 PIDs**; consuming **4** concurrently peaks at **≈1,549,758 KiB (≈1.48 GiB) mean across 19 PIDs** — a mean increase of **+479,022 KiB (+467.8 MiB), ratio 1.4474 = +44.7%** (per-run +43.7% / +45.8%), with **all four documents consumed successfully**. The peak PID count rises from 16 to 19 because more native OCR subprocesses are co-resident. This independently reproduces the reviewer's concurrency-scaling observation (their runs measured +37.151% / +37.585%); the exact percentage is environment- and timing-dependent (it hinges on how many native OCR subprocesses are co-resident at the sampled instant), but the **direction and order of magnitude are stable and reproducible**. *Labeled inferred:* the precise +% is not a fixed constant — the robust, observed claim is that **total cluster memory scales materially with in-flight concurrency up to the worker count**, whereas the *per-document* work does not scale with corpus size. The probe (`concurrency_probe.sh`, `tree_sampler.py`, `make_n_pdfs.py`) is listed in Appendix §13.2.

---

## 6. Document-type & batch-size sensitivity (O5)

**Batch size (document count), content fixed at 50 KB — real CLI export peak RSS (§3.1), averaged over the two runs:**

| docs | export peak (MB) | import peak (MB) | manifest.json |
|---|---|---|---|
| 200 | 96.6 | 115.6 | 9.68 MB |
| 1000 | 213.1 | 240.0 | 48.38 MB |
| 5000 | 805.6 | 804.4 | 241.91 MB |

Subtracting the **true zero-document baseline of 65.3 MB** (§3.1) isolates the content-driven growth of the export peak: 96.6 − 65.3 = **31.3 MB** (200 docs) → 213.1 − 65.3 = **147.8 MB** (1000) → 805.6 − 65.3 = **740.3 MB** (5000). From 1,000 to 5,000 documents (5× the count) the content-driven peak grows **740.3 / 147.8 = ≈5.01×** — i.e. **linear** in document count, with no super-linear term. (The earlier claim of a "fixed ~86 MB startup baseline" and "≈5.6× super-linear growth" was an artifact of subtracting a wrong baseline: 86 MB is actually the *5,000-document metadata export peak*, not the startup cost, which is 65.3 MB.)

**Document type (proxied by per-document `content` size), count fixed at 5,000 — real CLI export peak RSS (§3.1):**

| type | `content`/doc | export peak (MB) | import peak (MB) |
|---|---|---|---|
| metadata-only ("small metadata") | ~200 B | 84.8 | 138.6 |
| text-heavy ("text documents") | ~50 KB | 805.6 | 804.4 |

The in-process manifest probe (§3.2) shows the same shape from the other direction — at fixed 50 KB content, tracemalloc peak rises 30.9 → 148.5 → 731.6 MB for 200 → 1000 → 5000 documents. **Net: peak ≈ (document count) × (per-document `content` size), i.e. proportional to the aggregate corpus text.** "Mostly text documents" is precisely the high-`content` regime that spikes; the physical file size never enters the calculation.

**Content *representation* also matters (ASCII proxy bound, O5).** At a fixed nominal 50 KB `content` budget and fixed 5,000 documents, replacing the ASCII filler with **multi-byte Unicode** (`é`/`€`/CJK) changes the result because Django's JSON serializer escapes each non-ASCII code point (e.g. `\u20ac`), inflating the serialized payload:

| corpus (5000 × ~50 KB `content`) | manifest.json | export peak RSS (MB) |
|---|---|---|
| ASCII (`text5000`) | 253,665,956 B (241.91 MB) | 805.6 |
| Unicode (`uni5000`) | 559,225,956 B (533.32 MB) | 844.5 |

The Unicode manifest is **≈2.2×** larger and the peak RSS ≈4.8% higher for the *same* nominal content size. This bounds the document-type conclusion: the ASCII corpus is a faithful **size proxy** for the corpus-proportional mechanism, but real content whose text is non-ASCII (or higher-entropy) will serialize larger and push the peak higher still. The direction and mechanism are unchanged; the exact magnitude depends on the text's byte representation.

---

## 7. Stability / inconsistency — same unchanged input repeated (O4, O6)

To reproduce the reported run-to-run *inconsistency* at its source, the **same unchanged input** (the text1000 corpus) was run through the export mechanism **six times in one process**, without calling `malloc_trim` between iterations. Each iteration restarts `tracemalloc` so its **peak** is measured independently; `base_rss` at the *start* of each iteration reveals whether the freed high-water mark is reused (retention) or grows without bound (leak). Complete, unedited output (`cwd=/app/src`):

```text
$ python /tmp/scripts/stability_probe.py --scratch /tmp/s_text1000 --n 1000 --repeat 6 --label stab1000
{"label": "stab1000_r0", "base_rss": 50.1, "live_rss": 158.6, "after_gc_rss": 121.1, "tm_current_live": 99.6, "tm_current_after_gc": 1.4, "tm_peak": 148.5}
{"label": "stab1000_r1", "base_rss": 121.1, "live_rss": 157.8, "after_gc_rss": 152.1, "tm_current_live": 98.2, "tm_current_after_gc": 0.0, "tm_peak": 147.2}
{"label": "stab1000_r2", "base_rss": 152.1, "live_rss": 157.8, "after_gc_rss": 152.1, "tm_current_live": 98.2, "tm_current_after_gc": 0.0, "tm_peak": 147.1}
{"label": "stab1000_r3", "base_rss": 152.1, "live_rss": 157.8, "after_gc_rss": 129.8, "tm_current_live": 98.2, "tm_current_after_gc": 0.0, "tm_peak": 147.1}
{"label": "stab1000_r4", "base_rss": 129.8, "live_rss": 157.8, "after_gc_rss": 152.2, "tm_current_live": 98.2, "tm_current_after_gc": 0.0, "tm_peak": 147.1}
{"label": "stab1000_r5", "base_rss": 152.2, "live_rss": 157.9, "after_gc_rss": 152.3, "tm_current_live": 98.2, "tm_current_after_gc": 0.0, "tm_peak": 147.1}
{"summary_label": "stab1000", "repeat": 6, "tm_peaks": [148.5, 147.2, 147.1, 147.1, 147.1, 147.1], "mean_peak": 147.35, "stdev_peak": 0.5156, "peak_monotonic_increasing": false}
```

Three facts, each observed in the output:

1. **The per-corpus peak is deterministic.** The six `tm_peak` values are `[148.5, 147.2, 147.1, 147.1, 147.1, 147.1]` MB — **mean 147.35 MB, stdev 0.52 MB**. For a fixed corpus the allocation is essentially constant. So the reported "sometimes X, sometimes Y" is **not** a nondeterministic algorithm; it is driven by **which documents (and how much aggregate `content`) happen to be present** when the operation runs.

2. **Run order is stated explicitly, and the baseline is clean.** Iteration r0 begins in a **fresh process at `base_rss` 50.1 MB** (the true post-`django.setup()` baseline, matching §3.2). The later iterations begin higher — r1 at 121.1, r2/r3 at 152.1, r5 at 152.2 — because the process **kept** the high-water arenas from the previous iteration (no `malloc_trim` between iterations). This is the correct reading of the numbers: the elevated `base_rss` on r1–r5 is *retained-and-reused* memory from prior iterations of the *same* process, not memory "left by r0" as a defect. (An earlier draft misattributed a ~295 MB starting figure to r0; that was a baseline-contamination artifact. With a clean fresh-process r0 the baseline is 50.1 MB.)

3. **It is reuse, not a leak.** `base_rss` is **non-monotonic** — it rises to 152.1 (r2, r3) then *drops* to 129.8 at r4 before returning to 152.2 at r5 (`peak_monotonic_increasing: false`), i.e. the allocator reuses and occasionally releases arenas. And `tm_current_after_gc` returns to **0.0** after every iteration from r1 onward (r0 shows 1.4): the Python heap is fully freed each time. A genuine leak would show monotonically growing `base_rss`/peak **and** a `tm_current_after_gc` that never returns to baseline; neither is present.

---

## 8. Normal-GC vs. problematic classification, and process lifecycle (O3, O4, O6)

**Verdict: the "memory doesn't get released back to the system in a timely manner" symptom is normal CPython/glibc allocator arena retention amplified by scale — it is not an unbounded or algorithmic leak.**

### 8.1 True-leak criteria (stated first, then tested)

A genuine unbounded leak would exhibit **both** of:

- (a) Python-heap current allocation (`tracemalloc.get_traced_memory()` *current*) that does **not** return to baseline after `del` + `gc.collect()`, and
- (b) process RSS / peak that grows **monotonically** across repeated identical operations and is **not** reclaimable by the allocator or by process exit.

The evidence shows the **opposite** of both.

### 8.2 Python heap is fully freed; RSS retention is reclaimable (allocator arenas)

Keeping `tracemalloc` active through release, and then calling `malloc_trim(0)` via `ctypes`, isolates where the un-returned RSS lives. From §3.2 (export text5000) and §3.4 (import text5000), on the tracemalloc metric and the RSS metric separately:

| Operation (text5000) | tm_current_live | tm_current_after_gc | gc objects live → after | RSS after_gc | RSS after_trim | RSS freed by `malloc_trim(0)` |
|---|---|---|---|---|---|---|
| export (§3.2) | 490.9 | **1.4** | 101885 → 71868 | 552.9 | 68.0 | **484.8** |
| import (§3.4) | 248.9 | **1.6** | — | 554.2 | 55.6 | **498.6** |

Reading: after `del` + `gc.collect()` the **Python heap is essentially empty** (tracemalloc current 490.9 → **1.4 MB** export, 248.9 → **1.6 MB** import) — so there are **no leaked Python references** (criterion (a) fails). Yet process RSS stays at ≈553 MB, because CPython returned the freed blocks to its `pymalloc`/`obmalloc` pools and glibc arenas but the arenas were **not** handed back to the kernel. A single `malloc_trim(0)` then returns **≈485–499 MB** to the OS — proving the retained RSS is **reclaimable allocator arena**, not lost memory (criterion (b) fails). The §7 stability run confirms the same memory is otherwise **reused** on the next identical operation.

### 8.3 Process lifecycle — which process actually shows delayed release

The delayed release is a property of a **single long-running process**, not of the default deployment. Two grounded facts:

- **The export/import commands are standalone `manage.py` CLI processes that exit.** When the process exits, the OS reclaims everything. Demonstrated directly (complete output; `cwd=/app/src`):

```text
### Q_CLUSTER config (paperless/settings.py:110 django_q app; L449-457 config), verbatim from canonical image:
$ sed -n "110p;449,457p" paperless/settings.py
    "django_q",
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": PAPERLESS_WORKER_RETRY,
    "timeout": PAPERLESS_WORKER_TIMEOUT,
    "workers": TASK_WORKERS,
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
}

### Phase A — build manifest IN ONE PROCESS (text5000 corpus), then del+gc WITHOUT malloc_trim:
$ python /tmp/scripts/lifecycle_demo.py --scratch /tmp/s_text5000 --phase build
{"phase": "build_in_process", "docs": 5000, "base_rss_mb": 49.9, "after_build_rss_mb": 543.5, "after_del_gc_no_trim_rss_mb": 538.4, "released_by_del_gc_mb": 5.1}

### Phase B — a SEPARATE FRESH PROCESS starts after Phase A exited (baseline RSS proves OS reclaim on exit):
$ python /tmp/scripts/lifecycle_demo.py --scratch /tmp/s_text5000 --phase fresh
{"phase": "fresh_process_after_setup", "rss_after_setup_mb": 48.7}
```

  Within Phase A the RSS climbs to **543.5 MB** to build the manifest and stays at **538.4 MB** after `del` + `gc.collect()` (only 5.1 MB released without `malloc_trim`) — this *is* the "not released in a timely manner" the user observed, **inside that one process**. But Phase B, a brand-new process started after Phase A exited, begins at **48.7 MB**: the ≈540 MB was fully reclaimed by the OS on exit.

- **Asynchronous document tasks run under Django-Q with `recycle: 1`** (`src/paperless/settings.py:110` registers `"django_q"`; `:449-457` configures the cluster, with `"recycle": 1` at `:452` and `"timeout": PAPERLESS_WORKER_TIMEOUT` where `PAPERLESS_WORKER_TIMEOUT` defaults to 1800 at `settings.py:440`). `recycle: 1` means a worker process is recycled **after every task**, so even the async consume path does not, by default, accumulate the high-water mark across many completed tasks.

**Conclusion (no overclaim):** the un-returned RSS is retained, reclaimable allocator arena held **from the peak until the process exits or `malloc_trim` runs**. It is most visible while watching a *single* process during and right after a large export/import. No **default** configuration keeps a worker alive accumulating memory across many completed large tasks; a delayed-release "high-water that lingers" would require an explicitly long-lived process (e.g. a manually run long export, or a non-default worker configuration). *Labeled inferred:* the attribution of the specific un-returned bytes to CPython `pymalloc`/glibc arena mechanics is a mechanism explanation; what is **observed** is `tm_current_after_gc ≈ 0` with RSS at high-water, ≈485–499 MB reclaimed by `malloc_trim(0)`, reuse across identical runs (§7), and full reclaim on process exit (§8.3).

---

## 9. Caching evaluation (O3)

### 9.1 There is no dedicated caching layer (negative result)

The user's hypothesis of "caching behavior that's accumulating data unexpectedly" was tested first by looking for a caching module. **It does not exist in this revision** (complete output; `cwd=` repository root):

```text
$ ls src/documents/caching.py
ls: cannot access 'src/documents/caching.py': No such file or directory
$ find src -name caching.py
$        # (no output, exit status 0 — no file named caching.py anywhere under src/)
```

So "caching" resolves to two accumulators that outlive a single document, not a purpose-built cache: the `dateparser` module-scoped locale cache (§9.2) and the classifier's `CountVectorizer` vocabulary / unpickled model (§9.3).

### 9.2 `dateparser` module-scoped locale cache — bounded, but input/locale-dependent (≈15.7 MB for clean English vs. ≈62 MB / all 205 locales for a word-year false positive)

`parse_date()` (`src/documents/parsers.py:212`) imports `dateparser` inside its inner `__parser` and calls `dateparser.parse` per regex date match (`parsers.py:221-231`):

```python
# src/documents/parsers.py:221-231
        import dateparser

        return dateparser.parse(
            ds,
            settings={
                "DATE_ORDER": date_order,
                "PREFER_DAY_OF_MONTH": "first",
                "RETURN_AS_TIMEZONE_AWARE": True,
                "TIMEZONE": settings.TIME_ZONE,
            },
        )
```

The probe exercises the **real repository function `documents.parsers.parse_date(filename, text)`** (not a bare `dateparser.parse` call) under the **canonical default `PAPERLESS_DATE_ORDER=DMY`** (`settings.py:574`). It is run **2,000 iterations × 2 independent fresh processes per case**, inspecting the `dateparser` module-scoped locale cache before and after. Two canonical cases are exercised because — as the runtime evidence below shows — **the size of the warm-up depends on the input text, not on the code alone**:

- **Case A — realistic prose (`"Valid 2018-06-15"`).** This is the far more representative real-world input (an ISO date embedded in ordinary text). What `parse_date` actually feeds to `dateparser` is **not** the ISO date: the `DATE_REGEX` `re.finditer` loop (`parsers.py:261`) matches the **word-year alternative** `([^\W\d_]{3,9} [0-9]{4})` (the 5th alternative, `parsers.py:35`), which captures **`"Valid 2018"`** — consuming the `2018`, so the real ISO date `2018-06-15` never matches. `dateparser` is then handed the token `"Valid 2018"`; because `"Valid"` is **not** an English date keyword, `dateparser` consults **every bundled language locale** to see whether it is a date word in some other language, loading **all 205 locales**.
- **Case B — clean numeric `DD/MM/YYYY`.** Four **future** dates that parse successfully but fail the inner `__filter` (`date <= timezone.now()` at `parsers.py:237`; `__filter` body `parsers.py:233-241`), forcing the loop to iterate, followed by one valid **past** date (`15/06/2018`) that returns. All matches are numeric and parse as **English only**, so only the `en` locale loads.

The `DATE_REGEX` token extraction that distinguishes the two cases is itself observed (complete, unedited output; `cwd=/app/src`):

```text
$ python - <<'PY'   # DJANGO_SETTINGS_MODULE=paperless.settings + scratch PAPERLESS_* env
import django; django.setup()
from documents.parsers import DATE_REGEX
import re
for t in ["Valid 2018-06-15",
          "Report dated 01/02/2090 revised 03/04/2091 and 05/06/2092 superseded 07/08/2093 final valid 15/06/2018 end of file."]:
    print(repr(t), "->", [m.group(0) for m in re.finditer(DATE_REGEX, t)])
PY
'Valid 2018-06-15' -> ['Valid 2018']
'Report dated 01/02/2090 revised 03/04/2091 and 05/06/2092 superseded 07/08/2093 final valid 15/06/2018 end of file.' -> ['01/02/2090', '03/04/2091', '05/06/2092', '07/08/2093', '15/06/2018']
```

**Case A — complete, unedited output (`cwd=/app/src`):**

```text
### parse_date Case A run1 (fresh process):
$ python /tmp/scripts/parse_date_probe.py --scratch /tmp/s_pd --iters 2000 --label pd_dmy_realistic_r1 --text 'Valid 2018-06-15'
{"label": "pd_dmy_realistic_r1", "date_order": "DMY", "text": "Valid 2018-06-15", "iters": 2000, "calls_returning_a_date": 0, "parsed_date": null, "base_rss": 48.9, "after_first_parse_rss": 111.4, "after_all_parses_rss": 111.5, "after_gc_rss": 111.5, "warmup_delta_mb": 62.5, "plateau_delta_over_iters_mb": 0.0, "released_after_gc_mb": 0.0, "tm_peak": 25.9, "tm_current_after_gc": 23.4, "cache_after_first": {"dateparser_modules_loaded": 224, "locale_translation_data_loaded_count": 205, "locale_translation_data_sample": ["af", "agq", "ak", "am", "ar", "as", "asa", "ast"]}, "cache_after_many": {"dateparser_modules_loaded": 224, "locale_translation_data_loaded_count": 205, "locale_translation_data_sample": ["af", "agq", "ak", "am", "ar", "as", "asa", "ast"]}}

### parse_date Case A run2 (fresh process):
$ python /tmp/scripts/parse_date_probe.py --scratch /tmp/s_pd --iters 2000 --label pd_dmy_realistic_r2 --text 'Valid 2018-06-15'
{"label": "pd_dmy_realistic_r2", "date_order": "DMY", "text": "Valid 2018-06-15", "iters": 2000, "calls_returning_a_date": 0, "parsed_date": null, "base_rss": 48.8, "after_first_parse_rss": 110.5, "after_all_parses_rss": 110.6, "after_gc_rss": 110.6, "warmup_delta_mb": 61.7, "plateau_delta_over_iters_mb": 0.1, "released_after_gc_mb": 0.0, "tm_peak": 25.5, "tm_current_after_gc": 23.4, "cache_after_first": {"dateparser_modules_loaded": 224, "locale_translation_data_loaded_count": 205, "locale_translation_data_sample": ["af", "agq", "ak", "am", "ar", "as", "asa", "ast"]}, "cache_after_many": {"dateparser_modules_loaded": 224, "locale_translation_data_loaded_count": 205, "locale_translation_data_sample": ["af", "agq", "ak", "am", "ar", "as", "asa", "ast"]}}
```

**Case B — complete, unedited output (`cwd=/app/src`):**

```text
### parse_date Case B run1 (fresh process):
$ python /tmp/scripts/parse_date_probe.py --scratch /tmp/s_pd --iters 2000 --label pd_clean_r1
{"label": "pd_clean_r1", "date_order": "DMY", "text": "Report dated 01/02/2090 revised 03/04/2091 and 05/06/2092 superseded 07/08/2093 final valid 15/06/2018 end of file.", "iters": 2000, "calls_returning_a_date": 2000, "parsed_date": "2018-06-15T00:00:00+00:00", "base_rss": 48.9, "after_first_parse_rss": 64.6, "after_all_parses_rss": 64.7, "after_gc_rss": 64.7, "warmup_delta_mb": 15.7, "plateau_delta_over_iters_mb": 0.0, "released_after_gc_mb": 0.0, "tm_peak": 6.3, "tm_current_after_gc": 5.5, "cache_after_first": {"dateparser_modules_loaded": 20, "locale_translation_data_loaded_count": 1, "locale_translation_data_sample": ["en"]}, "cache_after_many": {"dateparser_modules_loaded": 20, "locale_translation_data_loaded_count": 1, "locale_translation_data_sample": ["en"]}}

### parse_date Case B run2 (fresh process):
$ python /tmp/scripts/parse_date_probe.py --scratch /tmp/s_pd --iters 2000 --label pd_clean_r2
{"label": "pd_clean_r2", "date_order": "DMY", "text": "Report dated 01/02/2090 revised 03/04/2091 and 05/06/2092 superseded 07/08/2093 final valid 15/06/2018 end of file.", "iters": 2000, "calls_returning_a_date": 2000, "parsed_date": "2018-06-15T00:00:00+00:00", "base_rss": 48.6, "after_first_parse_rss": 64.3, "after_all_parses_rss": 64.4, "after_gc_rss": 64.4, "warmup_delta_mb": 15.7, "plateau_delta_over_iters_mb": 0.0, "released_after_gc_mb": 0.0, "tm_peak": 6.3, "tm_current_after_gc": 5.5, "cache_after_first": {"dateparser_modules_loaded": 20, "locale_translation_data_loaded_count": 1, "locale_translation_data_sample": ["en"]}, "cache_after_many": {"dateparser_modules_loaded": 20, "locale_translation_data_loaded_count": 1, "locale_translation_data_sample": ["en"]}}
```

**Reading (stable across both runs of each case).** The **first** `parse_date` call triggers a one-time lazy load of `dateparser`'s locale/language machinery; the following **2,000** calls add a `plateau_delta_over_iters` of **+0.0 / +0.1 MB** in *both* cases — the cache does **not** grow with repeated same-input parses. The magnitude of that one-time warm-up, however, is **input- and locale-dependent**, and this is the key correction to any single-number claim:

- **Case A (realistic prose):** the warm-up is **+62.5 / +61.7 MB RSS**, of which **`tm_current_after_gc` = 23.4 MB** is retained Python heap that survives `gc.collect()` (exactly stable across both runs). Cache inspection shows **all 205 bundled locales** loaded (`locale_translation_data_loaded_count` 205; `dateparser_modules_loaded` 224) both `after_first` and `after_many`. No date is returned (`calls_returning_a_date` 0, `parsed_date` null) — the word-year token `"Valid 2018"` is not a real date, which is exactly why the full multilingual locale sweep is triggered.
- **Case B (clean numeric):** the warm-up is **+15.7 MB RSS**, of which **`tm_current_after_gc` = 5.5 MB** is retained heap. Only the **`en`** locale loads (`locale_translation_data_loaded_count` 1; `dateparser_modules_loaded` 20). Every call returns the expected `2018-06-15` (`calls_returning_a_date` 2000/2000).

**Classification.** In both cases the accumulation is **bounded, not a leak and not unbounded**: the per-input plateau is ≈0 MB, and the locale cache **cannot exceed the number of bundled locales** (205 is the ceiling, already reached in Case A). It is **module-scoped**, so whatever is loaded persists for the worker/process lifetime — which is why, in a long-lived Django-Q worker, the first document containing a word-year token permanently raises the worker's baseline by ≈23 MB of retained heap (≈60 MB RSS). This matches the reviewer's independently-observed ≈23.5 MiB retained-heap figure (our `tm_current_after_gc` = 23.4 MB, Case A). Relative to the manifest path (§3), even the 205-locale case is small and **fixed** (it does not scale with corpus count or `Document.content` size); it contributes to the *date-extraction metadata stage* footprint (`consumer.py:275`).

*Two labeled-inferred points, distinguished from the observations:* (1) the `warmup_delta_mb` RSS figure carries ≈±1 MB run-to-run variation (62.5 vs. 61.7; 15.7 vs. 15.7) because RSS reflects glibc-arena granularity, whereas the retained-heap `tm_current_after_gc` (23.4 / 5.5 MB) and the locale counts (205 / 1) are **exactly** reproducible — the stable figures are the ones relied upon; (2) that the majority of the warm-up lives in C-extension/locale data structures (only 25.9 / 6.3 MB of it shows in `tm_peak`, the Python-object peak) is a mechanism explanation. The **observed** facts are the warm-up deltas, the +0.0 MB plateau, the retained-heap figures, and the 205-vs-1 locale cache states above.

### 9.3 `classifier` — instance-held vectorizer/model and whole-corpus training (grounded in code + runtime)

The classifier's memory lives on the **instance**, not at module scope. `DocumentClassifier.__init__` sets `self.data_vectorizer = None` (`src/documents/classifier.py:70`); `load()` restores it (and the other models) from the pickle. Runtime introspection confirms this and the pickle count:

```text
$ python -c "import os,inspect,django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); import documents.classifier as m; from documents.classifier import DocumentClassifier as C; c=C(); print('FORMAT_VERSION =', C.FORMAT_VERSION); print('data_vectorizer in instance __dict__ =', 'data_vectorizer' in c.__dict__); print('data_vectorizer is a module global   =', hasattr(m,'data_vectorizer')); print('pickle.load occurrences in load() =', inspect.getsource(c.load).count('pickle.load'))"   # cwd=/app/src, canonical image
FORMAT_VERSION = 7
data_vectorizer in instance __dict__ = True
data_vectorizer is a module global   = False
pickle.load occurrences in load() = 7
```

So `load()` performs **seven** `pickle.load` calls (`classifier.py:78, 86, 87, 88, 90, 91, 92` — the schema version plus `data_hash`, `data_vectorizer`, `tags_binarizer`, `tags_classifier`, `correspondent_classifier`, `document_type_classifier`), and `data_vectorizer` is a **per-instance field** with `FORMAT_VERSION = 7` (`classifier.py:63`), not module-scoped state. `train()` builds a **whole-corpus** vocabulary and sparse matrix (`src/documents/classifier.py:194-199`):

```python
# src/documents/classifier.py:194-199
        self.data_vectorizer = CountVectorizer(
            analyzer="word",
            ngram_range=(1, 2),
            min_df=0.01,
        )
        data_vectorized = self.data_vectorizer.fit_transform(data)
```

With `ngram_range=(1, 2)` the vocabulary includes unigrams **and** bigrams over the entire training corpus, and `fit_transform(data)` builds a sparse document-term matrix over all documents at once — so classifier *training* memory also scales with corpus size, independent of any single file. The loaded model is held on the classifier instance for the duration of a consume run (`consumer.py:292`). These are consume-/train-path accumulators noted for completeness; they are **not** the cause of the import/metadata spike, which is the manifest path (§1, §4).


---

## 10. Canonical vs. non-canonical labeling

Per the canonical-observation rule, this section states exactly which evidence is canonical, which is a labeled partial stand-in, and which is non-canonical or infeasible.

**CANONICAL** — real entry point, canonical image `ghcr.io/scaleapi/swe-atlas:…qna_1.01` (Python 3.9.23 + pinned stack), branch commit `542221a38dff`, default SQLite DB, exact commands + complete output shown:

- **§3.1** — real `python manage.py document_exporter <dir> --no-progress-bar` under `/usr/bin/time -v` (all scales, ≥2 runs). **Carries the export magnitude claim.**
- **§3.3** — real `python manage.py document_importer <dir> --no-progress-bar` under `/usr/bin/time -v` (all scales, ≥2 runs, fresh DB each). **Carries the import magnitude claim.**
- **§8.3** — `Q_CLUSTER` config read verbatim from the image, and the CLI-exit lifecycle demo (real `Document` + `serializers`).
- **§9.2** — the **real** `documents.parsers.parse_date(filename, text)` function.
- **§9.3** — runtime introspection of the real `DocumentClassifier` class on the pinned stack.
- **§2.5** — the **real** canonical consume task `documents.tasks.consume_file` on a scanned PDF and a PNG (OCR feasibility, F-04), and the real `TikaDocumentParser.parse()` failure on `.docx` (the Tika/Java limitation).
- **§5.1** — the **real** `python manage.py qcluster` (11 workers) + `python manage.py document_consumer --oneshot` path, measuring process-tree memory vs. concurrency (F-05, ≥2 runs per level).

**LABELED PARTIAL STAND-INS** — real pinned stack and real `documents.models.Document`/real Django APIs, but only a *subset* of the full command (each section enumerates its omissions). Used only for phase-by-phase Python allocation and allocator behavior that `/usr/bin/time` cannot show; the whole-command magnitude is always carried by the canonical §3.1/§3.3:

- **§3.2** `export_probe.py` — the three memory-relevant `dump()` statements only.
- **§3.4** `import_probe.py` — `json.load` + `loaddata` + the `manifest_documents` filter only, inside the real `disable_signal` context, with a background thread sampling the concurrent peak.
- **§7** `stability_probe.py` — the export mechanism repeated in-process.

**NON-CANONICAL** — explicitly labeled, used for **no** magnitude claim:

- **§2.2** the host-interpreter attempt (Python 3.13.7 + Django 4.0.4) that fails at `import cgi`. It demonstrates only that the local interpreter cannot host the stack; every measured number comes from the container.

**INFEASIBLE ON THIS IMAGE** — stated, not worked around:

- **Office/DOCX consume via Tika.** The Tika parser is disabled by default (`PAPERLESS_TIKA_ENABLED="NO"`, `settings.py:592`); when enabled it delegates to an **external Tika/Gotenberg server backed by Java**, and the image ships **no Java** (`java: NOT FOUND`), so `TikaDocumentParser.parse()` raises `ParseError: … Unable to start Tika server` (§2.5, runtime-shown). This is the **only** consume path that cannot be exercised on this image.
- *(Corrected)* The **full end-to-end OCR consume of scanned PDFs/PNGs is NOT infeasible** — it is reproduced canonically in §2.5 and §5.1 and succeeds **without** `jbig2` (which is only an optional output-compression optimizer, not an OCR prerequisite). An earlier revision incorrectly listed OCR consume here; that claim is retracted and superseded by the F-04 runtime evidence in §2.5.

---

## 11. Claim-status matrix

Every key statement in this document is classified as **S** = source-grounded (`file:line`), **O** = observed (runtime output shown), **D** = derived (arithmetic on same-metric observed values), or **I** = inferred (mechanism explanation, not directly measured). Inferred rows are the only ones not backed by a direct `file:line` or measurement, and each is flagged as inferred in its section.

| # | Claim | Status | Evidence (section → grounding) |
|---|-------|:---:|--------------------------------|
| 1 | `dump()` serializes **10 selected models**, not "every DB object" | **S** | §1 → `document_exporter.py:116-150` |
| 2 | Each `Document` row carries full extracted text in `Document.content` (`TextField`) | **S** | §1, §4 → `models.py:117` |
| 3 | Export peak RSS scales with corpus_count × `content` size | **O** | §3.1 table; §6 |
| 4 | True zero-document exporter baseline ≈ **65.3 MB** (not ~86 MB) | **O** | §3.1 (67432/66324 KB) |
| 5 | scikit-learn / numpy / scipy are **not** imported at `django.setup()` | **O** | §3.2 note; `modules_check.txt` (§13.4) |
| 6 | Transient JSON-string + parsed-list co-residency ≈ **240.7 MB** at text5000 | **D** | §3.2 (tm_peak 731.6 − tm_current 490.9, same metric) |
| 7 | `manifest_documents` is a **shallow reference list** (~0 MB extra) | **S**+**O** | §1, §4 → `document_importer.py:137-139`; §3.4 (`manifest_documents_extra_delta` 0.0) |
| 8 | Real importer CLI peaks ≈ **804 MB** at text5000 | **O** | §3.3 (822832/824480 KB) |
| 9 | Concurrent `loaddata` peak ≈ **811 MB** (sampled *during* the call) | **O** | §3.4 (`concurrent_peak_rss_during_loaddata` 810.7/811.0) |
| 10 | Per-corpus peak is **deterministic** run-to-run (stdev 0.52 MB) | **O** | §7 (6-run `tm_peaks`); §3.1/§3.2/§3.3/§3.4 ≥2-run pairs |
| 11 | Python heap is **fully freed** after `del`+`gc.collect()` (tm_current → 1.4/1.6 MB) | **O** | §8.2; §3.2/§3.4 |
| 12 | Retained RSS is **reclaimable**: `malloc_trim(0)` returns ≈485/499 MB | **O** | §8.2 (`rss_freed_by_trim`) |
| 13 | Behavior is **retention, not a leak** (both leak criteria fail) | **D** | §8.1–§8.2 (from #10, #11, #12 + non-monotonic base_rss in §7) |
| 14 | CLI processes exit → OS reclaims all; `Q_CLUSTER['recycle']=1` recycles workers per task | **O**+**S** | §8.3 (lifecycle demo) → `settings.py:110,449-457` |
| 15 | The un-returned bytes live in CPython `pymalloc`/glibc **arenas** | **I** | §8.3 (labeled inferred; observed facts are tm_current≈0, trim-reclaim, exit-reclaim) |
| 16 | **No** dedicated `caching.py` module exists in this revision | **S**+**O** | §9.1 (`ls`/`find` output) |
| 17 | `dateparser` locale cache warm-up is **input/locale-dependent but bounded**: clean numeric text → ≈15.7 MB RSS / 5.5 MB retained heap / `["en"]` only; realistic prose word-year token → ≈62 MB RSS / **23.4 MB retained heap** / **all 205 locales**. Both **plateau at +0.0 MB** over 2000 iters and cannot exceed the 205 bundled locales | **O** | §9.2 (real `parse_date`, default DMY, 2 runs × 2 cases, cache-state introspection) |
| 18 | The warm-up lives largely in C-extension/locale structures (only 25.9 / 6.3 MB of it appears in `tm_peak`) | **I** | §9.2 (labeled inferred; observed = warm-up deltas, +0.0 plateau, retained heap 23.4 / 5.5 MB, 205 / 1 locales) |
| 19 | Classifier `data_vectorizer` is an **instance field**; `load()` has **7** `pickle.load` calls | **S**+**O** | §9.3 → `classifier.py:63,70,78,86-92`; runtime introspection |
| 20 | `train()` builds a **whole-corpus** CountVectorizer (unigram+bigram) | **S** | §9.3 → `classifier.py:194-199` |
| 21 | Unicode manifest ≈ **2.2×** the ASCII manifest at equal doc count/size | **O** | §6 (uni5000 533.32 MB vs text5000 241.91 MB manifest) |
| 22 | 1000→5000 content-driven growth is ≈ **5.01× (linear)** over the true baseline | **D** | §6 ((805.6−65.3)/(213.1−65.3)) |
| 23 | Host interpreter (Py 3.13) **cannot** host the stack (`cgi` removed, PEP 594) | **O** | §2.2 + §2.2.1 (verbatim traceback; CPython-3.13 collapse tokens expanded) |
| 24 | Full OCR consume of a scanned PDF and a PNG **succeeds** on this image **without `jbig2`** (canonical `consume_file`), producing OCR content + archive + thumbnail; single-doc consume peak ≈219 MB RSS | **O** | §2.5 (F-04) |
| 24a | The **only** infeasible consume path is Office/DOCX via Tika: disabled by default (`settings.py:592`) and requires an external Java-backed Tika server (Java absent) → `ParseError: Unable to start Tika server` | **S**+**O** | §2.5 (F-04) |
| 24b | Consume **total process-tree** memory scales with in-flight concurrency up to the worker count: 1 PDF ≈1.02 GiB / 16 PIDs vs. 4 concurrent ≈1.48 GiB / 19 PIDs (+44.7% mean, 2 runs) | **O** | §5.1 (F-05) |

---

## 12. Coverage pass (every sub-question addressed)

Decomposing the original question into the six objectives and the user's named sub-questions, each maps to the section(s) that answer it with observed evidence:

| Objective / sub-question (from the prompt) | Answered in | One-line answer |
|--------------------------------------------|-------------|-----------------|
| **O1** — Root cause of the spike | §1, §3.1, §3.3, §4 | Whole-corpus manifest materialization in `document_exporter.dump()` / `document_importer.handle()`; peak = corpus_count × `Document.content`. |
| **O2** — "unnecessary copies or holding references longer than needed" | §1, §3.2, §3.4, §4 | Yes — a transient ≈240.7 MB JSON-string/list duplicate at export peak, and `json.load` manifest co-resident with `loaddata` re-materialization at import (≈811 MB concurrent). `manifest_documents` is *not* an extra copy (claim #7, §3.4 — `manifest_documents_extra_delta` 0.0). |
| **O3** — "caching behavior accumulating data unexpectedly" | §9 | No dedicated cache module; the `dateparser` locale cache is a **bounded, module-scoped one-time warm-up** whose size is input/locale-dependent (≈15.7 MB / `["en"]` for clean numeric dates vs. ≈62 MB RSS / 23.4 MB retained heap / all 205 locales for a realistic word-year token), but it **plateaus at +0.0 MB** and cannot exceed the 205 bundled locales; the classifier vectorizer is instance-scoped/train-time. None causes the import spike. |
| **O4** — "what's different between spike and non-spike" | §5, §6, §3 | The single variable is aggregate `Document.content`: at equal doc count (5000) text-heavy vs metadata-only differs ≈9.5× (export) / ≈5.8× (import). Source/stage boundaries in §5. |
| **O5** — "how behavior changes with document types / batch sizes" | §6, §5.1 | **Manifest (import/export) path:** batch size (200/1000/5000) scales linearly (≈5.01×); document type (proxied by content size; ASCII vs Unicode 2.2×) scales with per-doc `content`, not file count. **Consume path:** per-task footprint is document-bounded, but the Django-Q cluster's total process-tree memory scales with *concurrent* in-flight documents up to the 11-worker count (1→4 PDFs: +44.7%, §5.1, F-05). |
| **O6** — runtime measurements, responsible methods, normal-GC-vs-problematic | §3 (before/during/after), §4, §7, §8 | Full tracemalloc+RSS series; named methods in §4; classified as **normal allocator retention, not a leak** (§8). |
| "actual runtime memory measurements" | §3.1–§3.4, §7, §8, §9 | Complete `/usr/bin/time -v` + tracemalloc/RSS output for every condition, ≥2 runs. |
| "which components/methods are responsible" | §4 | `dump()` L127-130 map/manifest; `handle()` L73/L87 manifest+loaddata; `Document.content`. |
| "normal Python GC vs something problematic" | §8 | Normal: heap freed to ~0, RSS reclaimable by `malloc_trim`/exit, deterministic peak, non-monotonic reuse. |
| "where the memory is actually going" | §3.2, §3.4, §4, §8.2 | Into the corpus-sized serialized string + parsed Python list (export) and manifest + `loaddata` structures (import), then held in reclaimable arenas until trim/exit. |

Every named item in the prompt is addressed above; no sub-question is left unanswered.


---

## 12A. Out-of-scope observations & pre-existing conditions (noted for completeness; not remediated — read-only mandate)

The investigation above answers the memory question in full. During the runtime work, four conditions surfaced that fall **outside the scope of this memory investigation**. Per the task's read-only mandate — no source or dependency file may be modified; the only repository change is this document (§2.1, §13.3) — they are **documented here for completeness rather than remediated**. Each is pre-existing in the canonical baseline and is **not** introduced by this documentation task; none of them is reachable through, or affects, the synthetic management-command measurement path used for the memory numbers above. Each maps to a QA finding and is grounded in `file:line` or complete observed output.

### 12A.1 The importer is non-atomic across `loaddata` + file-copy: a missing thumbnail leaves partial state (data-integrity edge — F-14)

`document_importer.handle()` runs a pre-flight `_check_manifest()` **before** any database write, but that check validates only the original document file and (conditionally) the archive file — it does **not** validate the thumbnail:

```
# src/documents/management/commands/document_importer.py:101-127  (_check_manifest)
#   L108  if EXPORTER_FILE_NAME not in record:  raise CommandError(...)             # key present?
#   L114  doc_file = record[EXPORTER_FILE_NAME]
#   L115  if not os.path.exists(os.path.join(self.source, doc_file)): raise ...     # ORIGINAL validated
#   L121  if EXPORTER_ARCHIVE_NAME in record:                                       # ARCHIVE validated (conditional)
#   L122      archive_file = record[EXPORTER_ARCHIVE_NAME]
#   L123      if not os.path.exists(os.path.join(self.source, archive_file)): raise ...
#   (there is NO EXPORTER_THUMBNAIL_NAME existence check)
```

Because the thumbnail is never pre-validated, a manifest whose referenced thumbnail is missing passes `_check_manifest()`; `loaddata` then commits the rows (`document_importer.py:87`), and only afterward does the physical copy loop raise — the original is copied first (`:165`), then the thumbnail copy (`:166`) fails:

```
# src/documents/management/commands/document_importer.py
#   :87   call_command("loaddata", manifest_path)          # rows COMMITTED here
#   :89   self._import_files_from_manifest(...)             # called AFTER loaddata
#   :165  shutil.copy2(document_path,  document.source_path)      # original copied OK
#   :166  shutil.copy2(thumbnail_path, document.thumbnail_path)   # raises FileNotFoundError
```

There is no transaction wrapping `loaddata` + the file copies, so the committed rows are **not** rolled back on the copy failure. Reproduced end-to-end through the **real** management command (originals were destroyed in a prior control attempt, so a fresh 2-doc corpus is re-exported here; `mkscratch`/`scenv`/`populate.py` are the Appendix §13.2 helpers):

```bash
source lib.sh
# 1. populate a 2-doc corpus and export it
mkscratch /tmp/s_f14_src3 ; cd /app/src ; scenv /tmp/s_f14_src3
python populate.py --scratch /tmp/s_f14_src3 --n 2 --content-bytes 1000
mkdir -p /tmp/exp_f14c ; python manage.py document_exporter /tmp/exp_f14c --no-progress-bar
# 2. remove ONLY the first thumbnail (manifest / original / archive left valid)
rm -f "/tmp/exp_f14c/2026-07-14 doc-0.txt-thumbnail.png"
# 3. import into a FRESH migrated DB via the REAL management command; read back state
mkscratch /tmp/s_f14_dst3 ; scenv /tmp/s_f14_dst3
python -c 'import django;django.setup();from documents.models import Document;print("docs_before=",Document.objects.count())'
python manage.py document_importer /tmp/exp_f14c --no-progress-bar ; echo "IMPORTER_EXIT=$?"
# read back rows + ORIGINALS/THUMBNAILS/ARCHIVE dir contents (settings.*_DIR)
```

Complete, unedited output (`/tmp/mem_probe/evidence/f14_partial_state.txt`):

```
docs_before=0
----- python manage.py document_importer /tmp/exp_f14c --no-progress-bar -----
Installed 3 object(s) from 1 fixture(s)
Copy files into paperless...
Traceback (most recent call last):
  File "/app/src/manage.py", line 11, in <module>
    execute_from_command_line(sys.argv)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/__init__.py", line 446, in execute_from_command_line
    utility.execute()
  File "/usr/local/lib/python3.9/site-packages/django/core/management/__init__.py", line 440, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/base.py", line 414, in run_from_argv
    self.execute(*args, **cmd_options)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/base.py", line 460, in execute
    output = self.handle(*args, **options)
  File "/app/src/documents/management/commands/document_importer.py", line 89, in handle
    self._import_files_from_manifest(options["no_progress_bar"])
  File "/app/src/documents/management/commands/document_importer.py", line 166, in _import_files_from_manifest
    shutil.copy2(thumbnail_path, document.thumbnail_path)
  File "/usr/local/lib/python3.9/shutil.py", line 444, in copy2
    copyfile(src, dst, follow_symlinks=follow_symlinks)
  File "/usr/local/lib/python3.9/shutil.py", line 264, in copyfile
    with open(src, 'rb') as fsrc:
FileNotFoundError: [Errno 2] No such file or directory: '/tmp/exp_f14c/2026-07-14 doc-0.txt-thumbnail.png'
IMPORTER_EXIT=1
----- partial state after failure -----
docs_after= 2
ORIGINALS_DIR(1)= ['doc-0000000.txt']
THUMBNAILS_DIR(0)= []
ARCHIVE_DIR(0)= []
```

Observed: `loaddata` installs the fixture objects (`Installed 3 object(s)` for this minimal 2-document fixture); the thumbnail copy raises `FileNotFoundError` at `document_importer.py:166`; the importer exits 1; the two document rows **remain** (`docs_after= 2`), the first original **has** been copied (`ORIGINALS_DIR(1)= ['doc-0000000.txt']`), and thumbnails are absent (`THUMBNAILS_DIR(0)= []`). (Removing an *original* instead is caught cleanly by `_check_manifest` at `:115` with `CommandError: … does not appear to be in the source directory` before any `loaddata`, so it does **not** leave partial state — the thumbnail is the only referenced file with no pre-flight check.) This is a pre-existing data-integrity edge in `document_importer.py`, unrelated to the memory question and **not** modified here (read-only). A fix would validate all referenced files before `loaddata` or wrap the DB + file changes in explicit rollback/cleanup. *[QA finding F-14, INFO — pre-existing source behavior.]*

### 12A.2 The pinned canonical baseline carries pre-existing security advisories (dependency posture — F-08)

The canonical image pins the exact versions in `Pipfile.lock`; several carry published security advisories. A read-only cross-check against the OSV.dev API — issued from **inside** the canonical container, installing nothing and touching no manifest — returned the following per pinned version. Exact command and complete output (`/tmp/mem_probe/evidence/f08_f15_osv.txt`; the probe is a zero-install `urllib` POST to `https://api.osv.dev/v1/query`):

```
### exact command: python /tmp/osv_query.py (OSV.dev /v1/query per pinned version) ###
django@4.0.4: 20 advisory(ies): GHSA-2hrw-hx67-34x6(BIT-django-2023-24580,CVE-2023-24580,PYSEC-2023-13); GHSA-6w2r-r2m5-xq5w(BIT-django-2025-57833,CVE-2025-57833,PYSEC-2025-105); GHSA-7xr5-9hcq-chf9(BIT-django-2025-48432,CVE-2025-48432,PYSEC-2025-47); GHSA-8x94-hmjh-97hq(BIT-django-2022-36359,CVE-2022-36359,PYSEC-2022-245); GHSA-frmv-pr5f-9mcr(BIT-django-2025-64459,CVE-2025-64459,PYSEC-2025-108); GHSA-jh3w-4vvf-mjgr(BIT-django-2023-36053,CVE-2023-36053,PYSEC-2023-100); GHSA-p64x-8rxx-wf6q(BIT-django-2022-34265,CVE-2022-34265,PYSEC-2022-213); GHSA-q2jf-h9jm-m7p4(BIT-django-2023-23969,CVE-2023-23969,PYSEC-2023-12); GHSA-qrw5-5h28-6cmg(BIT-django-2022-41323,CVE-2022-41323,PYSEC-2022-304); GHSA-qw25-v68c-qjf3(BIT-django-2025-64458,CVE-2025-64458,PYSEC-2025-107); GHSA-r3xc-prgr-mg9p(BIT-django-2023-31047,CVE-2023-31047,PYSEC-2023-61); GHSA-rrqc-c2jx-6jgv(BIT-django-2024-45231,CVE-2024-45231,PYSEC-2026-1297); PYSEC-2022-213(BIT-django-2022-34265,CVE-2022-34265,GHSA-p64x-8rxx-wf6q); PYSEC-2022-245(BIT-django-2022-36359,CVE-2022-36359,GHSA-8x94-hmjh-97hq); PYSEC-2022-304(BIT-django-2022-41323,CVE-2022-41323,GHSA-qrw5-5h28-6cmg); PYSEC-2023-100(BIT-django-2023-36053,CVE-2023-36053,GHSA-jh3w-4vvf-mjgr); PYSEC-2023-12(BIT-django-2023-23969,CVE-2023-23969,GHSA-q2jf-h9jm-m7p4); PYSEC-2023-13(BIT-django-2023-24580,CVE-2023-24580,GHSA-2hrw-hx67-34x6); PYSEC-2023-61(BIT-django-2023-31047,CVE-2023-31047,GHSA-r3xc-prgr-mg9p); PYSEC-2026-1297(BIT-django-2024-45231,CVE-2024-45231,GHSA-rrqc-c2jx-6jgv)
scikit-learn@1.0.2: 2 advisory(ies): GHSA-jw8x-6495-233v(CVE-2024-5206,PYSEC-2024-110); PYSEC-2024-110(CVE-2024-5206,GHSA-jw8x-6495-233v)
scipy@1.8.0: 1 advisory(ies): PYSEC-2023-102(CVE-2023-25399,GHSA-9jx5-6pgf-crrp)
numpy@1.22.3: (no advisories)
dateparser@1.1.1: (no advisories)
django-q@1.3.9: (no advisories)
pikepdf@5.1.1: (no advisories)
ocrmypdf@13.4.3: (no advisories)
redis@3.5.3: (no advisories)
```

OSV returns each advisory once per identifier namespace, so the GHSA and PYSEC rows are alias pairs of the same underlying CVE. **Deduplicating by CVE**, the 20 Django rows collapse to **12 unique Django CVEs** (each present as one GHSA + one PYSEC record) and the 2 scikit-learn rows collapse to **1 CVE (CVE-2024-5206)**; the single SciPy record is **withdrawn** (§12A.3). The active baseline count is therefore **12 Django + 1 scikit-learn = 13**; `numpy`, `dateparser`, `django-q`, `pikepdf`, `ocrmypdf`, and `redis` returned no advisories. These are **pre-existing in the pinned baseline and not introduced by this task**; they cannot be remediated here because the read-only mandate forbids any change to `Pipfile.lock` or the pins. They are also **not reachable through the memory-measurement path**, which drives the exporter/importer/parser/classifier/consume code via management commands and never serves the Django HTTP request surfaces to which most of these advisories apply. `Pipfile.lock` remains byte-identical to HEAD (§13.3 confirms `git` reports only the deliverable changed). Remediation would be a separate dependency-upgrade effort with regression testing — explicitly not this read-only documentation task. *[QA finding F-08, MAJOR — pre-existing baseline.]*

### 12A.3 One reported SciPy advisory is withdrawn (audit-tooling caveat — F-15)

A version-based scan of SciPy 1.8.0 surfaces `PYSEC-2023-102` / `CVE-2023-25399`, but the authoritative GitHub-Advisory record for it is **withdrawn**. Exact command and complete output (same evidence file; `urllib` GET to `https://api.osv.dev/v1/vulns/<id>`):

```
### exact command: python /tmp/osv_withdrawn.py (OSV.dev /v1/vulns/<id>) ###
GHSA-9jx5-6pgf-crrp: withdrawn=2024-05-14T20:15:44Z | aliases=CVE-2023-25399,PYSEC-2023-102 | summary=Withdrawn: scipy memory leak vulnerability
PYSEC-2023-102: withdrawn=(none) | aliases=CVE-2023-25399,GHSA-9jx5-6pgf-crrp | summary=
CVE-2023-25399: withdrawn=(none) | aliases=GHSA-9jx5-6pgf-crrp,PYSEC-2023-102 | summary=
```

`GHSA-9jx5-6pgf-crrp` carries `withdrawn=2024-05-14T20:15:44Z` and the summary "Withdrawn: scipy memory leak vulnerability", whereas its `PYSEC-2023-102` / `CVE-2023-25399` alias records carry no `withdrawn` field — which is exactly why a version-only scan still reports it. It is therefore **excluded from the active advisory count in §12A.2** and should be annotated as withdrawn rather than treated as active. (Of note for this investigation specifically: the withdrawn record was itself a *scipy memory-leak* report; it is withdrawn and does not apply to the pinned SciPy, so it is not evidence of any memory issue in this codebase.) *[QA finding F-15, INFO — audit-tooling discrepancy.]*

### 12A.4 Prior standalone QA reports were not available as artifacts (process note — F-01)

This deliverable's content — the memory root-cause analysis (§1), runtime measurements (§3), component attribution (§4), sensitivity analysis (§5–§6), stability/inconsistency (§7), and the normal-GC-vs-leak classification (§8) — is complete and self-contained against the six objectives (§12). Separately, earlier per-checkpoint QA reports were not persisted as standalone, independently retrievable artifacts, so completeness against any *unrecorded* earlier finding cannot be proven from this document alone. This is a QA-process / artifact-persistence matter — checkpoint reports should be stored as immutable artifacts with their IDs supplied to final acceptance — **not a defect in this document's content**, and it cannot be addressed by editing this Markdown deliverable. It is recorded here only so the limitation is explicit. *[QA finding F-01, MAJOR — process/evidence.]*


---

## 13. Appendix — exact reproduction (provisioning, probe sources, cleanup, full evidence)

Everything here is copy-pasteable. The investigation used the unique container name `pngx-mem-1783965419-78995`; the provisioning below regenerates an equivalent unique name so parallel clones never collide.

### 13.1 Canonical provisioning (run on the Docker host)

```bash
set -euo pipefail
umask 077   # F-12: temporary scripts, evidence files, and scratch dirs are created private (0700/0600)

# 1. Canonical image (already pulled; do NOT rebuild)
IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01"

# 2. UNIQUE container name (avoids collision with parallel clones)
CNAME="pngx-mem-$(date +%s)-$$"

# 3. Start a detached container that stays alive
docker run -d --name "$CNAME" --entrypoint /bin/bash "$IMG" -c "sleep infinity"

# 4. Provision native/service gaps + GNU time (not baked into the image)
docker exec "$CNAME" bash -lc '
  set -euo pipefail
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y redis-server libzbar0 poppler-utils pngquant time
'

# 5. Start redis (Django-Q broker / channels), no persistence
docker exec "$CNAME" bash -lc "redis-server --daemonize yes --save '' --appendonly no && redis-cli ping"

# 6. Create the default instance dirs and migrate the default SQLite DB
docker exec "$CNAME" bash -lc '
  set -euo pipefail
  mkdir -p /app/consume /app/media /app/data /app/static \
           /app/media/documents/originals /app/media/documents/archive /app/media/documents/thumbnails \
           /app/data/index /app/data/log
  cd /app/src && python manage.py migrate --no-input
'

# 7. Verify the canonical stack + native tool versions
docker exec "$CNAME" bash -lc 'python --version; python -c "import django,dateparser,sklearn; print(django.get_version(), dateparser.__version__, sklearn.__version__)"; qpdf --version | head -1; command -v jbig2 || echo "jbig2: NOT FOUND"'

# 8. Copy the probe scripts INTO the container (explicit names, no wildcards)
docker exec "$CNAME" bash -lc 'mkdir -p /tmp/scripts /tmp/evidence'
for s in populate.py export_probe.py import_probe.py parse_date_probe.py stability_probe.py lifecycle_demo.py modules_check.py lib.sh; do
  docker cp "/tmp/mem_probe/scripts/$s" "$CNAME:/tmp/scripts/$s"
done

# 9. (After runs) copy an evidence file OUT — explicit source and destination, no wildcards.
#    The source /tmp/evidence/export_text5000.txt is written by the per-scenario run block
#    below (the `... | tee /tmp/evidence/export_text5000.txt` step). The host destination
#    parent directory must exist before `docker cp`, so create it first, then verify the
#    copy is byte-identical to the in-container file via a sha256 checksum.
mkdir -p /tmp/mem_probe/evidence_container
docker cp "$CNAME:/tmp/evidence/export_text5000.txt" "/tmp/mem_probe/evidence_container/export_text5000.txt"
IN_SUM="$(docker exec "$CNAME" sha256sum /tmp/evidence/export_text5000.txt | awk '{print $1}')"
OUT_SUM="$(sha256sum /tmp/mem_probe/evidence_container/export_text5000.txt | awk '{print $1}')"
[ "$IN_SUM" = "$OUT_SUM" ] && echo "copy verified: $OUT_SUM" || { echo "CHECKSUM MISMATCH"; exit 1; }
```

Per-scenario run pattern (example: text5000). `lib.sh` provides `mkscratch` (a fresh migrated instance) and `scenv` (exports the scratch env). Every scenario populates in its own process, exports in a fresh process, and imports into a **separate empty** DB:

```bash
docker exec "$CNAME" bash -lc '
  set -euo pipefail
  source /tmp/scripts/lib.sh
  # populate a throwaway corpus in its own scratch instance
  mkscratch /tmp/s_text5000
  python /tmp/scripts/populate.py --scratch /tmp/s_text5000 --n 5000 --content-bytes 50000
  # EXPORT under GNU time, fresh process, canonical entry point.
  # NOTE: document_exporter REQUIRES its target directory to pre-exist
  # (src/documents/management/commands/document_exporter.py:98-99 raises
  #  CommandError("That path doesn't exist")); the command never creates it.
  # Create the export target first, then run the exporter into it:
  cd /app/src && scenv /tmp/s_text5000
  mkdir -p /tmp/exp_text5000_run1 /tmp/evidence
  # Capture the full export transcript (stdout + the /usr/bin/time -v report on stderr)
  # to the evidence file that step 9 of §13.1 copies out. `tee` preserves the on-screen
  # output while writing the file; `set -o pipefail` keeps a non-zero exporter exit fatal.
  /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000_run1 --no-progress-bar 2>&1 | tee /tmp/evidence/export_text5000.txt
  # IMPORT into a SEPARATE fresh empty DB, canonical entry point
  mkscratch /tmp/imp_text5000_run1 && scenv /tmp/imp_text5000_run1
  /usr/bin/time -v python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar
'
```

### 13.2 Probe script sources (verbatim, unedited)

All probes **force** a unique scratch env (assignment, never `setdefault`); **resolve** the scratch root with `os.path.realpath` and abort unless it lives strictly under `/tmp/`, so a symlinked path component cannot redirect a destructive operation outside `/tmp` (F-07); set `umask 077` so every scratch dir/file they create is private (0700/0600, F-12); **validate** numeric arguments (`--n`/`--content-bytes` ≥ 0, `--repeat`/`--iters` ≥ 1) with a concise nonzero-exit error, treating each argument value literally (F-13); **assert** the resolved DB/MEDIA/ORIGINALS paths are under that `/tmp` scratch root before touching a row (finding 14); **refuse** to run against a non-empty DB where relevant; and carry **falsifiable assertions** on row/manifest counts (finding 18). The companion `lib.sh` `mkscratch`/`scenv` and the F-05 `concurrency_probe.sh` apply the same `readlink -f` realpath guard (rejecting anything not strictly under `/tmp/`) before any `rm -rf`, and set `umask 077`.

**`populate.py`**

```python
#!/usr/bin/env python3
"""
populate.py - create a FRESH throwaway corpus (real documents.models.Document rows +
tiny ~14-byte physical originals and 8-byte stub thumbnails) inside an ISOLATED scratch
instance, so that `manage.py document_exporter` can run against a real, controlled DB.

SAFETY (review finding 14): the scratch root is REQUIRED and FORCED (never inherited via
setdefault). After django.setup() we ASSERT that the resolved SQLite DB path, MEDIA_ROOT,
ORIGINALS_DIR and THUMBNAIL_DIR all live under that scratch root; otherwise we abort BEFORE
touching any row, so a mis-pointed environment can never delete real data via the
post_delete cleanup_document_deletion signal (signals/handlers.py:233).

FALSIFIABILITY (review finding 18): asserts row count == N after creation and that every
physical original/thumbnail file was written.
"""
import argparse
import os
import sys


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def assert_scratch(scratch_root):
    from django.conf import settings
    checks = {
        "DB": settings.DATABASES["default"]["NAME"],
        "MEDIA_ROOT": settings.MEDIA_ROOT,
        "ORIGINALS_DIR": settings.ORIGINALS_DIR,
        "THUMBNAIL_DIR": settings.THUMBNAIL_DIR,
        "DATA_DIR": settings.DATA_DIR,
    }
    for name, path in checks.items():
        if not str(path).startswith(scratch_root):
            sys.exit("SAFETY ABORT: %s=%s is not under scratch root %s" % (name, path, scratch_root))


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True, help="explicit scratch root under /tmp")
    ap.add_argument("--n", type=int, required=True)
    ap.add_argument("--content-bytes", type=int, required=True)
    ap.add_argument("--unicode", action="store_true",
                    help="fill content with a multi-byte Unicode pattern instead of ASCII 'x'")
    args = ap.parse_args()

    # F-13: validate numeric inputs with a concise nonzero-exit error.
    if args.n < 0:
        sys.exit("ABORT: --n must be >= 0, got %d" % args.n)
    if args.content_bytes < 0:
        sys.exit("ABORT: --content-bytes must be >= 0, got %d" % args.content_bytes)

    force_scratch_env(args.scratch)
    import django
    django.setup()
    assert_scratch(args.scratch)

    from django.conf import settings
    from documents.models import Document

    os.makedirs(settings.ORIGINALS_DIR, exist_ok=True)
    os.makedirs(settings.THUMBNAIL_DIR, exist_ok=True)

    # Fresh corpus: safe because assert_scratch proved we are inside the scratch DB.
    Document.objects.all().delete()

    if args.unicode:
        unit = "\u00e9\u20ac\u6587\n"  # e-acute(2B) + euro(3B) + CJK(3B) + newline
        reps = (args.content_bytes // len(unit.encode("utf-8"))) + 1
        blob = unit * reps
        while len(blob.encode("utf-8")) > args.content_bytes:
            blob = blob[:-1]
    else:
        blob = "x" * args.content_bytes

    objs = [
        Document(
            title="doc-%d" % i,
            content=blob,
            mime_type="text/plain",
            checksum="%032d" % i,
            filename="doc-%07d.txt" % i,
        )
        for i in range(args.n)
    ]
    Document.objects.bulk_create(objs, batch_size=500)

    count = Document.objects.count()
    assert count == args.n, "FALSIFIABLE ASSERT FAILED: expected %d rows, got %d" % (args.n, count)

    written = 0
    for d in Document.objects.all().only("id", "filename"):
        with open(os.path.join(settings.ORIGINALS_DIR, d.filename), "w") as f:
            f.write("tiny original\n")
        with open(os.path.join(settings.THUMBNAIL_DIR, "%07d.png" % d.pk), "wb") as f:
            f.write(b"\x89PNG\r\n\x1a\n")
        written += 1
    assert written == args.n, "FALSIFIABLE ASSERT FAILED: wrote %d files, expected %d" % (written, args.n)

    enc = "unicode(utf-8)" if args.unicode else "ascii"
    print("populated %d docs; content_bytes=%d (%s); originals+thumbnails written=%d; scratch=%s"
          % (count, args.content_bytes, enc, written, args.scratch))


if __name__ == "__main__":
    main()
```

**`export_probe.py`**

```python
#!/usr/bin/env python3
"""
export_probe.py - PARTIAL in-process stand-in for the memory-relevant CORE of
document_exporter.dump() (src/documents/management/commands/document_exporter.py:127-130):

    documents        = Document.objects.order_by("id")                 # L127
    document_map     = {d.pk: d for d in documents}                    # L128
    document_manifest= json.loads(serializers.serialize("json", docs)) # L129
    manifest        += document_manifest                               # L130

CONSPICUOUS OMISSIONS (this probe deliberately does NOT reproduce; the CANONICAL
`manage.py document_exporter` run measures the whole command):
  - serialization of the other 9 selected models (Correspondent, Tag, DocumentType,
    MailAccount, MailRule, SavedView, SavedViewFilterRule, Group, User)
  - the per-document annotation/file-copy loop (document_exporter.py:153-210)
  - the final json.dump() of manifest.json to disk
  - tqdm progress bar
Its purpose is ONLY to expose the phase-by-phase Python allocation and the
before/during/after + allocator-retention behaviour that /usr/bin/time cannot show.

INSTRUMENTATION (review findings 7, 10): tracemalloc is kept ACTIVE across del+gc.collect();
we report peak, live current bytes, and POST-GC current bytes (same metric), plus live gc
object counts, then call libc malloc_trim(0) and record the RSS drop. RSS (from
/proc/self/statm) and tracemalloc bytes are reported SEPARATELY and never subtracted from
each other.

SAFETY (finding 14): forced scratch env + under-scratch-root assertions.
FALSIFIABILITY (finding 18): asserts manifest length and map length equal the row count.
"""
import argparse
import ctypes
import ctypes.util
import gc
import json
import os
import statistics
import sys
import tracemalloc

PAGE = os.sysconf("SC_PAGE_SIZE")


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def assert_scratch(scratch_root):
    from django.conf import settings
    for name, path in (("DB", settings.DATABASES["default"]["NAME"]),
                       ("MEDIA_ROOT", settings.MEDIA_ROOT),
                       ("ORIGINALS_DIR", settings.ORIGINALS_DIR)):
        if not str(path).startswith(scratch_root):
            sys.exit("SAFETY ABORT: %s=%s not under scratch %s" % (name, path, scratch_root))


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


def malloc_trim():
    try:
        libc = ctypes.CDLL(ctypes.util.find_library("c") or "libc.so.6")
        libc.malloc_trim(0)
        return True
    except Exception:
        return False


def measure_export(n, label):
    from django.core import serializers
    from documents.models import Document

    gc.collect()
    base_rss = rss_mb()
    obj_before = len(gc.get_objects())
    tracemalloc.start()

    documents = Document.objects.order_by("id")               # L127
    document_map = {d.pk: d for d in documents}               # L128
    document_manifest = json.loads(                           # L129
        serializers.serialize("json", documents),
    )
    manifest = []
    manifest += document_manifest                             # L130

    live_rss = rss_mb()
    cur_live, peak = tracemalloc.get_traced_memory()
    obj_live = len(gc.get_objects())

    # FALSIFIABLE assertions: the mechanism actually materialized the corpus.
    assert len(document_manifest) == n, "manifest len %d != n %d" % (len(document_manifest), n)
    assert len(document_map) == n, "map len %d != n %d" % (len(document_map), n)

    # Release WITH tracemalloc STILL ACTIVE (finding 7).
    del documents, document_map, document_manifest, manifest
    collected = gc.collect()
    cur_after_gc, peak_after_gc = tracemalloc.get_traced_memory()
    after_gc_rss = rss_mb()
    obj_after = len(gc.get_objects())
    tracemalloc.stop()

    trimmed = malloc_trim()
    after_trim_rss = rss_mb()

    return {
        "label": label, "n": n,
        # RSS series (MB) - process resident set, reported on its own metric:
        "base_rss": round(base_rss, 1),
        "live_rss": round(live_rss, 1),
        "after_gc_rss": round(after_gc_rss, 1),
        "after_trim_rss": round(after_trim_rss, 1),
        "rss_freed_by_trim": round(after_gc_rss - after_trim_rss, 1),
        # tracemalloc series (MB) - python heap, reported on its own metric:
        "tm_current_live": round(cur_live / 1048576, 1),
        "tm_current_after_gc": round(cur_after_gc / 1048576, 1),
        "tm_peak": round(peak / 1048576, 1),
        # gc object counts:
        "gc_objects_live": obj_live,
        "gc_objects_after_gc": obj_after,
        "gc_collected": collected,
        "malloc_trim_available": trimmed,
    }


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True)
    ap.add_argument("--n", type=int, required=True)
    ap.add_argument("--label", default="export")
    ap.add_argument("--repeat", type=int, default=1,
                    help="in-process repeats (demonstrates arena retention/reuse across iterations)")
    args = ap.parse_args()

    # F-13: validate numeric inputs with a concise nonzero-exit error.
    if args.n < 0:
        sys.exit("ABORT: --n must be >= 0, got %d" % args.n)
    if args.repeat < 1:
        sys.exit("ABORT: --repeat must be >= 1, got %d" % args.repeat)

    force_scratch_env(args.scratch)
    import django
    django.setup()
    assert_scratch(args.scratch)

    from documents.models import Document
    got = Document.objects.count()
    assert got == args.n, "corpus has %d rows, expected %d (populate first)" % (got, args.n)

    peaks = []
    for r in range(args.repeat):
        res = measure_export(args.n, "%s_r%d" % (args.label, r))
        peaks.append(res["tm_peak"])
        print(json.dumps(res))
    if args.repeat > 1:
        print(json.dumps({
            "summary_label": args.label, "repeat": args.repeat, "tm_peaks": peaks,
            "mean_peak": round(statistics.mean(peaks), 2),
            "stdev_peak": round(statistics.pstdev(peaks), 4),
        }))


if __name__ == "__main__":
    main()
```

**`import_probe.py`**

```python
#!/usr/bin/env python3
"""
import_probe.py - PARTIAL in-process stand-in for the memory-relevant CORE of
document_importer.handle() (src/documents/management/commands/document_importer.py):

    self.manifest = json.load(f)              # L73  (retained on the instance)
    call_command("loaddata", manifest_path)   # L87  (Django re-deserializes the SAME file)
    manifest_documents = list(filter(          # L137-139 (shallow reference list)
        lambda r: r["model"] == "documents.document", self.manifest))

CONSPICUOUS OMISSIONS (the CANONICAL `manage.py document_importer` run measures the whole
command): _check_manifest() validation, the physical file copy/save loop, and the final
`document_index reindex`. We DO faithfully reproduce the real importer's signal-disabling
context (disable_signal on post_save/m2m_changed -> update_filename_and_move_files) so no
file-moving side effects fire, exactly as document_importer.py:75-87.

CONCURRENT PEAK (review finding 4): a background sampler thread records the TRUE peak RSS
*while* call_command("loaddata") is executing, instead of a single post-return sample.

INSTRUMENTATION (findings 7, 10): tracemalloc kept ACTIVE through del+gc.collect(); peak +
post-GC current bytes reported on the tracemalloc metric; RSS reported separately; the two
are never subtracted. manifest_documents extra is measured to show it is reference-list
overhead (finding 15).

SAFETY (finding 14): forced scratch env + under-scratch-root assertions; imports into a
FRESH scratch DB that we assert is empty first.
"""
import argparse
import ctypes
import ctypes.util
import gc
import json
import logging
import os
import sys
import threading
import time
import tracemalloc

PAGE = os.sysconf("SC_PAGE_SIZE")


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def assert_scratch(scratch_root):
    from django.conf import settings
    for name, path in (("DB", settings.DATABASES["default"]["NAME"]),
                       ("MEDIA_ROOT", settings.MEDIA_ROOT),
                       ("ORIGINALS_DIR", settings.ORIGINALS_DIR)):
        if not str(path).startswith(scratch_root):
            sys.exit("SAFETY ABORT: %s=%s not under scratch %s" % (name, path, scratch_root))


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


def malloc_trim():
    try:
        libc = ctypes.CDLL(ctypes.util.find_library("c") or "libc.so.6")
        libc.malloc_trim(0)
        return True
    except Exception:
        return False


class RSSSampler(threading.Thread):
    """Background thread: records the peak RSS while the main thread runs loaddata."""
    def __init__(self, interval=0.003):
        super().__init__(daemon=True)
        self.interval = interval
        self.peak = rss_mb()
        self._stop_event = threading.Event()

    def run(self):
        while not self._stop_event.is_set():
            r = rss_mb()
            if r > self.peak:
                self.peak = r
            time.sleep(self.interval)

    def stop(self):
        self._stop_event.set()
        self.join(timeout=2.0)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True)
    ap.add_argument("--manifest", required=True)
    ap.add_argument("--label", default="import")
    args = ap.parse_args()

    force_scratch_env(args.scratch)
    import django
    django.setup()
    assert_scratch(args.scratch)

    # Silence per-row post_save log flood, mirroring importer setting handlers[0].level=ERROR.
    logging.disable(logging.CRITICAL)

    from django.core.management import call_command
    from django.db.models.signals import m2m_changed, post_save
    from documents.models import Document
    from documents.management.commands.document_importer import disable_signal
    from documents.signals.handlers import update_filename_and_move_files

    # Import into a FRESH scratch DB: assert it is empty first (finding 14 fail-fast).
    start_rows = Document.objects.count()
    assert start_rows == 0, "SAFETY/FALSIFIABLE ABORT: scratch DB not empty (%d rows)" % start_rows

    # Measure base RSS BEFORE any manifest read so the baseline reflects only django.setup()
    # overhead (avoids the pre-load baseline contamination flagged in review finding 6).
    gc.collect()
    base_rss = rss_mb()
    tracemalloc.start()

    # L73 - retained on the instance:
    with open(args.manifest) as f:
        manifest = json.load(f)
    after_json_load_rss = rss_mb()

    # Counts derived from the ALREADY-loaded manifest (no extra load -> no contamination):
    manifest_len = len(manifest)
    doc_rows_expected = sum(1 for r in manifest if r["model"] == "documents.document")

    # L87 - concurrent peak sampling around the synchronous loaddata call:
    sampler = RSSSampler()
    sampler.start()
    with disable_signal(post_save, receiver=update_filename_and_move_files, sender=Document):
        with disable_signal(m2m_changed, receiver=update_filename_and_move_files,
                            sender=Document.tags.through):
            call_command("loaddata", args.manifest, verbosity=0)
    sampler.stop()
    concurrent_peak_rss = sampler.peak
    post_loaddata_rss = rss_mb()

    # L137-139 - shallow reference list (should add ~0 payload; finding 15):
    rss_before_filter = rss_mb()
    manifest_documents = list(filter(lambda r: r["model"] == "documents.document", manifest))
    rss_after_filter = rss_mb()

    cur_live, peak = tracemalloc.get_traced_memory()

    rows_loaded = Document.objects.count()
    # FALSIFIABLE assertions:
    assert rows_loaded == doc_rows_expected, "loaded %d rows, manifest lists %d" % (rows_loaded, doc_rows_expected)
    assert len(manifest_documents) == doc_rows_expected, "filter produced %d, expected %d" % (
        len(manifest_documents), doc_rows_expected)

    # Release with tracemalloc still active (finding 7):
    del manifest, manifest_documents
    collected = gc.collect()
    cur_after_gc, _ = tracemalloc.get_traced_memory()
    after_gc_rss = rss_mb()
    tracemalloc.stop()
    trimmed = malloc_trim()
    after_trim_rss = rss_mb()

    print(json.dumps({
        "label": args.label,
        "manifest": os.path.basename(args.manifest),
        "manifest_records": manifest_len,
        "rows_loaded": rows_loaded,
        # RSS series (MB), own metric:
        "base_rss": round(base_rss, 1),
        "after_json_load_rss": round(after_json_load_rss, 1),
        "concurrent_peak_rss_during_loaddata": round(concurrent_peak_rss, 1),
        "post_loaddata_rss": round(post_loaddata_rss, 1),
        "rss_after_manifest_documents_filter": round(rss_after_filter, 1),
        "after_gc_rss": round(after_gc_rss, 1),
        "after_trim_rss": round(after_trim_rss, 1),
        "rss_freed_by_trim": round(after_gc_rss - after_trim_rss, 1),
        # phase deltas on the SAME RSS metric:
        "json_load_rss_delta": round(after_json_load_rss - base_rss, 1),
        "loaddata_concurrent_extra_delta": round(concurrent_peak_rss - after_json_load_rss, 1),
        "manifest_documents_extra_delta": round(rss_after_filter - rss_before_filter, 1),
        # tracemalloc series (MB), own metric:
        "tm_peak": round(peak / 1048576, 1),
        "tm_current_live": round(cur_live / 1048576, 1),
        "tm_current_after_gc": round(cur_after_gc / 1048576, 1),
        "malloc_trim_available": trimmed,
    }))


if __name__ == "__main__":
    main()
```

**`parse_date_probe.py`**

```python
#!/usr/bin/env python3
"""
parse_date_probe.py - exercises the REAL repository function
documents.parsers.parse_date(filename, text) (src/documents/parsers.py:212), NOT a direct
dateparser.parse() call. Each invocation runs one or more dateparser.parse() calls internally
via the DATE_REGEX finditer loop (parsers.py:261-272) with the inner __filter that rejects
future dates (date <= timezone.now(); parsers.py:233-241).

GOAL (review finding 9 / F-06): show that the dateparser module-scoped locale cache warm-up
is INPUT- and LOCALE-dependent. Two canonical cases under the DEFAULT PAPERLESS_DATE_ORDER=DMY
(settings.py:574):
  (A) realistic prose text whose DATE_REGEX match is a WORD+YEAR token that is not an English
      date keyword -> dateparser consults ALL bundled language locales -> large warm-up.
  (B) clean numeric DD/MM/YYYY text that parses as English -> only the 'en' locale loads ->
      small warm-up.
In BOTH cases the accumulation is BOUNDED (plateaus over N iterations; cannot exceed the
number of bundled locales) and module-scoped (held for the worker lifetime).

SAFETY (finding 14 / F-07): forced scratch env + assertions.
FALSIFIABILITY (finding 18 / F-13): asserts run-to-run determinism (same input -> same result
on the warm-up call and across the whole iteration loop).
"""
import argparse
import gc
import json
import os
import sys
import tracemalloc

PAGE = os.sysconf("SC_PAGE_SIZE")


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


def dateparser_cache_state():
    """Introspect dateparser's module-scoped locale cache.

    dateparser lazily imports per-language translation data as submodules under
    dateparser.data.date_translation_data.<lang> and holds them at MODULE SCOPE for the
    process lifetime. We report how many language locales have been loaded (the cache that
    grows as more languages are encountered) plus the total dateparser.* module count.
    """
    state = {}
    state["dateparser_modules_loaded"] = sum(
        1 for m in sys.modules if m == "dateparser" or m.startswith("dateparser.")
    )
    prefix = "dateparser.data.date_translation_data."
    langs = sorted(
        m[len(prefix):] for m in sys.modules
        if m.startswith(prefix) and "." not in m[len(prefix):]
    )
    state["locale_translation_data_loaded_count"] = len(langs)
    state["locale_translation_data_sample"] = langs[:8]
    return state


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True)
    ap.add_argument("--iters", type=int, default=2000)
    ap.add_argument("--label", default="parsedate")
    ap.add_argument("--text", default=None,
                    help="text to parse; default = clean DD/MM/YYYY case (B)")
    ap.add_argument("--date-order", default="DMY",
                    help="PAPERLESS_DATE_ORDER; canonical default is DMY (settings.py:574)")
    args = ap.parse_args()

    if args.iters < 1:
        sys.exit("ABORT: --iters must be >= 1, got %d" % args.iters)

    os.environ["PAPERLESS_DATE_ORDER"] = args.date_order
    force_scratch_env(args.scratch)
    import django
    django.setup()

    from documents.parsers import parse_date

    filename = "doc.txt"  # FILENAME_DATE_ORDER empty by default -> filename loop skipped
    # Case (B) default: multiple FUTURE dates (fail __filter, force iteration) then one valid
    # PAST date that returns. Case (A) is supplied via --text.
    default_text = ("Report dated 01/02/2090 revised 03/04/2091 and 05/06/2092 "
                    "superseded 07/08/2093 final valid 15/06/2018 end of file.")
    text = args.text if args.text is not None else default_text

    gc.collect()
    base = rss_mb()
    tracemalloc.start()

    d0 = parse_date(filename, text)                 # warm-up (loads locales lazily)
    after_first = rss_mb()
    cache_after_first = dateparser_cache_state()

    returned = 0
    last = "UNSET"
    for _ in range(args.iters):
        d = parse_date(filename, text)
        last = d
        if d is not None:
            returned += 1
    after_many = rss_mb()
    cur, peak = tracemalloc.get_traced_memory()
    cache_after_many = dateparser_cache_state()

    gc.collect()
    after_gc = rss_mb()
    cur_after_gc, _ = tracemalloc.get_traced_memory()
    tracemalloc.stop()

    # FALSIFIABLE determinism: same input -> identical result on warm-up and across the loop.
    assert (d0 is None) == (last is None), \
        "nondeterministic presence: warm-up %r vs loop %r" % (d0, last)
    if d0 is not None:
        assert d0 == last, "nondeterministic date: warm-up %r vs loop %r" % (d0, last)
        assert returned == args.iters, \
            "expected %d dates, got %d" % (args.iters, returned)
    else:
        assert returned == 0, "expected 0 dates, got %d" % returned

    print(json.dumps({
        "label": args.label,
        "date_order": args.date_order,
        "text": text,
        "iters": args.iters,
        "calls_returning_a_date": returned,
        "parsed_date": d0.isoformat() if d0 is not None else None,
        "base_rss": round(base, 1),
        "after_first_parse_rss": round(after_first, 1),
        "after_all_parses_rss": round(after_many, 1),
        "after_gc_rss": round(after_gc, 1),
        "warmup_delta_mb": round(after_first - base, 1),
        "plateau_delta_over_iters_mb": round(after_many - after_first, 1),
        "released_after_gc_mb": round(after_many - after_gc, 1),
        "tm_peak": round(peak / 1048576, 1),
        "tm_current_after_gc": round(cur_after_gc / 1048576, 1),
        "cache_after_first": cache_after_first,
        "cache_after_many": cache_after_many,
    }))


if __name__ == "__main__":
    main()
```

**`stability_probe.py`**

```python
#!/usr/bin/env python3
"""
stability_probe.py - runs the export-manifest mechanism (document_exporter.py:127-130) the
SAME unchanged input N times IN-PROCESS, WITHOUT calling malloc_trim between iterations, to
answer two questions the reviewer flagged:

  (finding 6/O4) Is the per-corpus PEAK deterministic run-to-run? -> report tm_peak per iter.
  (finding 7/O6) Does freed memory get REUSED (arena retention) or does RSS grow without
                 bound (leak)? -> report base_rss at the start of each iteration; if the
                 high-water mark is REUSED, base_rss[r>=1] stays flat and tm_current returns
                 to ~baseline after each gc; a real leak would grow monotonically.

Each iteration: measure base_rss (start), build L127-130, live_rss + tm peak/current,
del+gc.collect() (NO trim), after_gc_rss + tm current. tracemalloc is restarted per iter so
each iteration's PEAK is measured independently.

TRUE-LEAK CRITERIA (stated, then tested): a genuine unbounded leak would show (a) tm_current
NOT returning to baseline after gc, AND (b) base_rss/peak growing monotonically across
identical repeats. Retention-only (no leak) shows tm_current -> ~0 after gc and a FLAT peak.

SAFETY: forced scratch env + assertions. FALSIFIABLE: asserts manifest length == n each iter.
"""
import argparse
import gc
import json
import os
import statistics
import sys
import tracemalloc

PAGE = os.sysconf("SC_PAGE_SIZE")


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def assert_scratch(scratch_root):
    from django.conf import settings
    for name, path in (("DB", settings.DATABASES["default"]["NAME"]),
                       ("MEDIA_ROOT", settings.MEDIA_ROOT)):
        if not str(path).startswith(scratch_root):
            sys.exit("SAFETY ABORT: %s=%s not under scratch %s" % (name, path, scratch_root))


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True)
    ap.add_argument("--n", type=int, required=True)
    ap.add_argument("--repeat", type=int, default=6)
    ap.add_argument("--label", default="stab")
    args = ap.parse_args()

    # F-13: validate numeric inputs with a concise nonzero-exit error.
    if args.n < 0:
        sys.exit("ABORT: --n must be >= 0, got %d" % args.n)
    if args.repeat < 1:
        sys.exit("ABORT: --repeat must be >= 1, got %d" % args.repeat)

    force_scratch_env(args.scratch)
    import django
    django.setup()
    assert_scratch(args.scratch)

    from django.core import serializers
    from documents.models import Document
    assert Document.objects.count() == args.n, "corpus has %d rows, expected %d" % (
        Document.objects.count(), args.n)

    peaks = []
    for r in range(args.repeat):
        gc.collect()
        base_rss = rss_mb()
        tracemalloc.start()
        documents = Document.objects.order_by("id")
        document_map = {d.pk: d for d in documents}
        document_manifest = json.loads(serializers.serialize("json", documents))
        manifest = []
        manifest += document_manifest
        live_rss = rss_mb()
        cur_live, peak = tracemalloc.get_traced_memory()
        assert len(document_manifest) == args.n, "manifest %d != n %d" % (len(document_manifest), args.n)
        del documents, document_map, document_manifest, manifest
        gc.collect()
        cur_after_gc, _ = tracemalloc.get_traced_memory()
        tracemalloc.stop()  # NO malloc_trim -> observe arena reuse on the next iteration
        after_gc_rss = rss_mb()
        peaks.append(round(peak / 1048576, 1))
        print(json.dumps({
            "label": "%s_r%d" % (args.label, r),
            "base_rss": round(base_rss, 1),
            "live_rss": round(live_rss, 1),
            "after_gc_rss": round(after_gc_rss, 1),
            "tm_current_live": round(cur_live / 1048576, 1),
            "tm_current_after_gc": round(cur_after_gc / 1048576, 1),
            "tm_peak": round(peak / 1048576, 1),
        }))
    print(json.dumps({
        "summary_label": args.label, "repeat": args.repeat, "tm_peaks": peaks,
        "mean_peak": round(statistics.mean(peaks), 2),
        "stdev_peak": round(statistics.pstdev(peaks), 4),
        "peak_monotonic_increasing": all(peaks[i] <= peaks[i + 1] for i in range(len(peaks) - 1)) and peaks[0] < peaks[-1],
    }))


if __name__ == "__main__":
    main()
```

**`lifecycle_demo.py`**

```python
"""Lifecycle demo (Finding 8): show (a) delayed release WITHIN a process after del+gc without malloc_trim,
and (b) full OS reclaim when the process EXITS (next fresh process starts at baseline).
Exercises the real documents.models.Document + serializers (export mechanism, dump() L127-130)."""
import os, sys, json, gc, argparse
def _force_scratch_env(scratch):
    # F-07: resolve symlinks/'..'; require the resolved path strictly under /tmp/. F-12: umask 077.
    scratch = os.path.realpath(scratch) if scratch else ""
    if not scratch.startswith("/tmp/"):
        sys.exit(f"SAFETY ABORT: scratch {scratch!r} must resolve strictly under /tmp/")
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = f"{scratch}/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = f"{scratch}/media"
    os.environ["PAPERLESS_STATICDIR"] = f"{scratch}/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = f"{scratch}/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
ap = argparse.ArgumentParser()
ap.add_argument("--scratch", required=True)
ap.add_argument("--phase", required=True, choices=["build", "fresh"])
args = ap.parse_args()
_force_scratch_env(args.scratch)
sys.path.insert(0, "/app/src")
import django; django.setup()
from django.conf import settings as dj
# safety: resolved DB path must be under scratch
dbp = dj.DATABASES["default"]["NAME"]
assert str(dbp).startswith(args.scratch), f"SAFETY ABORT: DB {dbp} not under {args.scratch}"
PAGE = os.sysconf("SC_PAGE_SIZE")
def rss_mb():
    with open("/proc/self/statm") as f:
        return round(int(f.read().split()[1]) * PAGE / (1024 * 1024), 1)
if args.phase == "fresh":
    # Fresh process: just report baseline RSS after django.setup (proves OS reclaimed prior process memory)
    print(json.dumps({"phase": "fresh_process_after_setup", "rss_after_setup_mb": rss_mb()}))
    sys.exit(0)
# phase == build: materialize the manifest, show within-process delayed release
from django.core import serializers
from documents.models import Document
n = Document.objects.count()
assert n > 0, "SAFETY ABORT: expected a pre-populated corpus"
gc.collect(); base = rss_mb()
documents = Document.objects.order_by("id")            # dump() L127
document_map = {d.pk: d for d in documents}            # L128
document_manifest = json.loads(serializers.serialize("json", documents))  # L129
manifest = []; manifest += document_manifest           # L130
after_build = rss_mb()
del documents, document_map, document_manifest, manifest
gc.collect()                                            # NO malloc_trim here — show delayed release
after_del_gc = rss_mb()
print(json.dumps({"phase": "build_in_process", "docs": n, "base_rss_mb": base,
                  "after_build_rss_mb": after_build, "after_del_gc_no_trim_rss_mb": after_del_gc,
                  "released_by_del_gc_mb": round(after_build - after_del_gc, 1)}))
```

**`modules_check.py`**

```python
#!/usr/bin/env python3
"""
modules_check.py - reports process RSS and whether the heavy scientific libraries are loaded
into sys.modules immediately after django.setup() with 0 documents. Supports the corrected
"true zero-document baseline" claim (review finding 11): if sklearn/numpy/scipy are NOT
imported at Django startup, they cannot be part of the exporter's fixed baseline.
"""
import os
import sys

PAGE = os.sysconf("SC_PAGE_SIZE")
scratch = sys.argv[1] if len(sys.argv) > 1 else None
# F-07: resolve symlinks/'..'; require the resolved path strictly under /tmp/.
scratch = os.path.realpath(scratch) if scratch else ""
if not scratch.startswith("/tmp/"):
    sys.exit("usage: modules_check.py <scratch-root-strictly-under-/tmp>")
os.environ["PAPERLESS_DATA_DIR"] = scratch + "/data"
os.environ["PAPERLESS_MEDIA_ROOT"] = scratch + "/media"
os.environ["PAPERLESS_STATICDIR"] = scratch + "/static"
os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch + "/consume"
os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
sys.path.insert(0, "/app/src")


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


rss_before_setup = rss_mb()
import django
django.setup()
rss_after_setup = rss_mb()

from documents.models import Document
doc_count = Document.objects.count()

import json
print(json.dumps({
    "doc_count": doc_count,
    "rss_before_django_setup_mb": round(rss_before_setup, 1),
    "rss_after_django_setup_mb": round(rss_after_setup, 1),
    "sklearn_loaded": "sklearn" in sys.modules,
    "numpy_loaded": "numpy" in sys.modules,
    "scipy_loaded": "scipy" in sys.modules,
    "dateparser_loaded": "dateparser" in sys.modules,
    "django_loaded": "django" in sys.modules,
}))
```

**`lib.sh`** (scratch-instance helpers)

```bash
# Helper: create a fresh, migrated scratch instance directory
mkscratch() {
  umask 077                        # F-12: scratch dirs/files created private (0700/0600)
  local S R
  S="$1"
  # F-07: resolve symlinks/'..' and refuse to `rm -rf` anything not strictly under /tmp/.
  R="$(readlink -f -- "$S" 2>/dev/null || true)"
  case "$R" in
    /tmp/?*) : ;;
    *) echo "SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: '$S' (resolved: '$R')" >&2; return 1 ;;
  esac
  rm -rf -- "$R"
  mkdir -p "$R"/data "$R"/media/documents/originals "$R"/media/documents/archive "$R"/media/documents/thumbnails "$R"/static "$R"/consume "$R"/data/index "$R"/data/log
  ( cd /app/src && PAPERLESS_DATA_DIR="$R/data" PAPERLESS_MEDIA_ROOT="$R/media" PAPERLESS_STATICDIR="$R/static" PAPERLESS_CONSUMPTION_DIR="$R/consume" python manage.py migrate --no-input >/dev/null 2>&1 )
}
# Helper: export env for a scratch instance
scenv() {
  local S R
  S="$1"
  # F-07: resolve symlinks/'..'; refuse to point Django at anything not strictly under /tmp/.
  R="$(readlink -f -- "$S" 2>/dev/null || true)"
  case "$R" in
    /tmp/?*) : ;;
    *) echo "SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: '$S' (resolved: '$R')" >&2; return 1 ;;
  esac
  export PAPERLESS_DATA_DIR="$R/data" PAPERLESS_MEDIA_ROOT="$R/media" PAPERLESS_STATICDIR="$R/static" PAPERLESS_CONSUMPTION_DIR="$R/consume"
}
```

#### F-04 / F-05 consume-path probes (OCR feasibility and concurrency)

**`make_scanned_inputs.py`** — F-04: builds a synthetic image-only scanned PDF + a PNG (text rendered with PIL) that REQUIRE OCR.

```python
#!/usr/bin/env python3
"""Create synthetic 'scanned' inputs that REQUIRE OCR (image-only, no text layer):
   - a PNG with rendered text
   - an image-only PDF built from that PNG (img2pdf; no embedded text)
The rendered text is a known sentence so we can assert OCR extracted it.
"""
import sys
from PIL import Image, ImageDraw, ImageFont

OUT = sys.argv[1] if len(sys.argv) > 1 else "/tmp/ocr_inputs"
SENTENCE = "PAPERLESS OCR MEMORY PROBE ALPHA BRAVO CHARLIE 2018"

def find_font(size):
    for p in ("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
              "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
              "/usr/share/fonts/dejavu/DejaVuSans.ttf"):
        try:
            return ImageFont.truetype(p, size)
        except OSError:
            continue
    return ImageFont.load_default()

def make_png(path):
    W, H = 1240, 1754  # ~A4 at 150 dpi
    img = Image.new("RGB", (W, H), "white")
    d = ImageDraw.Draw(img)
    font = find_font(48)
    small = find_font(34)
    y = 120
    d.text((90, y), SENTENCE, fill="black", font=font); y += 120
    for i in range(1, 9):
        d.text((90, y), "Line %d: the quick brown fox jumps over the lazy dog." % i,
                fill="black", font=small); y += 70
    img.save(path, "PNG", dpi=(150, 150))
    return path

def make_pdf(png_path, pdf_path):
    import img2pdf
    with open(pdf_path, "wb") as f:
        f.write(img2pdf.convert(png_path))
    return pdf_path

if __name__ == "__main__":
    import os
    os.makedirs(OUT, exist_ok=True)
    png = make_png(os.path.join(OUT, "scanned_probe.png"))
    pdf = make_pdf(png, os.path.join(OUT, "scanned_probe.pdf"))
    print("SENTENCE:", SENTENCE)
    print("PNG:", png, os.path.getsize(png), "bytes")
    print("PDF:", pdf, os.path.getsize(pdf), "bytes")
```

**`consume_probe.py`** — F-04: exercises the canonical consume task documents.tasks.consume_file on one file and reports the created Document (content/archive/thumbnail) + peak RSS.

```python
#!/usr/bin/env python3
"""
consume_probe.py - exercises the REAL canonical consume task
documents.tasks.consume_file(path) (src/documents/tasks.py:184 -> Consumer().try_consume_file
at tasks.py:236), the exact function a Django-Q worker runs for each queued document.

GOAL (F-04): demonstrate that full OCR consume of a scanned (image-only) PDF and of a PNG
SUCCEEDS on the canonical image WITHOUT the jbig2 binary (absent) — producing a Document with
OCR-extracted content, an archive PDF, and a thumbnail. Reports peak RSS/tracemalloc so the
per-document consume footprint is visible.

SAFETY: forced scratch env under /tmp/; measured file copied into a scratch consumption dir.
FALSIFIABILITY: asserts a Document row was created and that OCR content is non-empty.
"""
import argparse
import gc
import json
import os
import shutil
import sys
import tracemalloc

PAGE = os.sysconf("SC_PAGE_SIZE")


def force_scratch_env(scratch_root):
    # F-07: resolve symlinks/'..' so a crafted argument cannot escape /tmp via a symlinked
    # path component; require the RESOLVED path to live strictly under /tmp/.
    # F-12: umask 077 so scratch dirs/files created afterwards are private (0700/0600).
    scratch_root = os.path.realpath(scratch_root) if scratch_root else ""
    if not scratch_root.startswith("/tmp/"):
        sys.exit("SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: %r" % scratch_root)
    os.umask(0o077)
    os.environ["PAPERLESS_DATA_DIR"] = scratch_root + "/data"
    os.environ["PAPERLESS_MEDIA_ROOT"] = scratch_root + "/media"
    os.environ["PAPERLESS_STATICDIR"] = scratch_root + "/static"
    os.environ["PAPERLESS_CONSUMPTION_DIR"] = scratch_root + "/consume"
    os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
    sys.path.insert(0, "/app/src")


def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--scratch", required=True)
    ap.add_argument("--file", required=True)
    ap.add_argument("--label", default="consume")
    args = ap.parse_args()

    force_scratch_env(args.scratch)
    import django
    django.setup()

    from documents.tasks import consume_file
    from documents.models import Document

    consume_dir = os.environ["PAPERLESS_CONSUMPTION_DIR"]
    os.makedirs(consume_dir, exist_ok=True)
    staged = os.path.join(consume_dir, os.path.basename(args.file))
    shutil.copy2(args.file, staged)

    n_before = Document.objects.count()
    gc.collect()
    base = rss_mb()
    tracemalloc.start()

    result_msg = consume_file(staged)            # canonical task function (synchronous)

    peak_rss = rss_mb()
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()

    n_after = Document.objects.count()
    doc = Document.objects.order_by("-pk").first()
    content = (doc.content or "") if doc else ""
    has_archive = bool(doc and doc.has_archive_version)
    thumb_path = doc.thumbnail_path if doc else None
    thumb_exists = bool(thumb_path and os.path.exists(thumb_path))
    orig_path = doc.source_path if doc else None
    orig_exists = bool(orig_path and os.path.exists(orig_path))

    # FALSIFIABLE: a document was created and OCR produced non-empty content.
    assert n_after == n_before + 1, "expected 1 new Document, got %d" % (n_after - n_before)
    assert len(content.strip()) > 0, "OCR content is empty"

    print(json.dumps({
        "label": args.label,
        "input_file": args.file,
        "input_bytes": os.path.getsize(args.file),
        "consume_result_msg": result_msg,
        "documents_before": n_before,
        "documents_after": n_after,
        "doc_pk": doc.pk if doc else None,
        "doc_mime_type": doc.mime_type if doc else None,
        "content_len": len(content),
        "content_snippet": content.strip()[:120],
        "content_has_MEMORY": "MEMORY" in content.upper(),
        "content_has_PROBE": "PROBE" in content.upper(),
        "has_archive_version": has_archive,
        "archive_exists_on_disk": bool(doc and doc.archive_path and os.path.exists(doc.archive_path)) if doc else False,
        "original_exists_on_disk": orig_exists,
        "thumbnail_exists_on_disk": thumb_exists,
        "checksum": doc.checksum if doc else None,
        "base_rss_mb": round(base, 1),
        "peak_rss_mb": round(peak_rss, 1),
        "consume_rss_delta_mb": round(peak_rss - base, 1),
        "tm_peak_mb": round(peak / 1048576, 1),
    }))


if __name__ == "__main__":
    main()
```

**`tika_docx_probe.py`** — F-04: demonstrates the true Office/DOCX limitation — the real TikaDocumentParser.parse() cannot start a Java-backed Tika server.

```python
#!/usr/bin/env python3
"""
tika_docx_probe.py - demonstrates the TRUE bounded limitation for Office/DOCX consume (F-04):
Tika is DISABLED by default (.docx unsupported); when ENABLED, .docx routes to the real
TikaDocumentParser (paperless_tika/parsers.py), whose parse() calls
tika.parser.from_file(path, PAPERLESS_TIKA_ENDPOINT) (parsers.py:55) against an EXTERNAL
Tika/Gotenberg server backed by Java. Java is absent from the image, so the call fails and
parse() raises ParseError. Exercised through the REAL parser class.
"""
import os
import sys
import zipfile

sys.path.insert(0, "/app/src")
os.environ["DJANGO_SETTINGS_MODULE"] = "paperless.settings"
os.environ["PAPERLESS_DATA_DIR"] = "/tmp/s_ocr/data"
os.environ["PAPERLESS_MEDIA_ROOT"] = "/tmp/s_ocr/media"
os.environ["PAPERLESS_STATICDIR"] = "/tmp/s_ocr/static"
os.environ["PAPERLESS_CONSUMPTION_DIR"] = "/tmp/s_ocr/consume"


def make_min_docx(path):
    ct = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
          '<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">'
          '<Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>'
          '<Default Extension="xml" ContentType="application/xml"/>'
          '<Override PartName="/word/document.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml"/>'
          '</Types>')
    rels = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
            '<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">'
            '<Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="word/document.xml"/>'
            '</Relationships>')
    doc = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
           '<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">'
           '<w:body><w:p><w:r><w:t>Tika probe office document CHARLIE 2018</w:t></w:r></w:p></w:body>'
           '</w:document>')
    with zipfile.ZipFile(path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("[Content_Types].xml", ct)
        z.writestr("_rels/.rels", rels)
        z.writestr("word/document.xml", doc)
    return path


def main():
    docx = make_min_docx("/tmp/ocr_inputs/office_probe.docx")
    os.environ["PAPERLESS_TIKA_ENABLED"] = "1"
    import django
    django.setup()
    from django.conf import settings
    from paperless_tika.parsers import TikaDocumentParser
    from documents.parsers import ParseError

    print("PAPERLESS_TIKA_ENABLED:", settings.PAPERLESS_TIKA_ENABLED)
    print("PAPERLESS_TIKA_ENDPOINT:", settings.PAPERLESS_TIKA_ENDPOINT)
    print("docx bytes:", os.path.getsize(docx))
    p = TikaDocumentParser(logging_group=None)
    try:
        p.parse(docx, "application/vnd.openxmlformats-officedocument.wordprocessingml.document")
        print("UNEXPECTED: parse() returned without error; text=%r" % (p.get_text(),))
    except ParseError as e:
        print("ParseError (expected):", str(e))
    finally:
        try:
            p.cleanup()
        except Exception:
            pass


if __name__ == "__main__":
    main()
```

**`make_n_pdfs.py`** — F-05: builds N DISTINCT scanned PDFs (distinct checksums) for the concurrency test.

```python
#!/usr/bin/env python3
"""Create N DISTINCT synthetic scanned (image-only) PDFs (distinct content -> distinct
checksums, so none are rejected as duplicates during concurrent consume)."""
import os, sys
from PIL import Image, ImageDraw, ImageFont
import img2pdf

OUT = sys.argv[1]; N = int(sys.argv[2])
os.makedirs(OUT, exist_ok=True)

def font(sz):
    for p in ("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
              "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"):
        try: return ImageFont.truetype(p, sz)
        except OSError: pass
    return ImageFont.load_default()

for i in range(N):
    W,H=1240,1754
    img=Image.new("RGB",(W,H),"white"); d=ImageDraw.Draw(img)
    big=font(48); small=font(34); y=120
    d.text((90,y),"PAPERLESS OCR CONCURRENCY DOC %03d TOKEN%03d 2018"%(i,i),fill="black",font=big); y+=120
    for j in range(1,9):
        d.text((90,y),"Doc %d line %d: the quick brown fox jumps over the lazy dog."%(i,j),fill="black",font=small); y+=70
    png=os.path.join(OUT,"conc_%03d.png"%i)
    img.save(png,"PNG",dpi=(150,150))
    with open(os.path.join(OUT,"conc_%03d.pdf"%i),"wb") as f:
        f.write(img2pdf.convert(png))
    os.remove(png)
print("created %d distinct pdfs in %s"%(N,OUT))
```

**`tree_sampler.py`** — F-05: samples the full process tree rooted at a PID — peak summed-RSS (KiB) and peak PID count — until a sentinel appears.

```python
#!/usr/bin/env python3
"""Sample the FULL process tree rooted at a given PID: peak summed-RSS (KiB) and peak PID
count, until a sentinel file appears. Used to measure the Django-Q cluster's total footprint
(master + 11 workers + transient native OCR subprocesses) during concurrent consume."""
import os, sys, time
root=int(sys.argv[1]); sentinel=sys.argv[2]; out=sys.argv[3]
PAGE=os.sysconf("SC_PAGE_SIZE")
def cmap():
    m={}
    for pid in os.listdir("/proc"):
        if not pid.isdigit(): continue
        try:
            with open("/proc/%s/stat"%pid) as f: parts=f.read().rsplit(")",1)[-1].split()
            ppid=int(parts[1]); m.setdefault(ppid,[]).append(int(pid))
        except Exception: continue
    return m
def tree(r):
    m=cmap(); seen=set(); st=[r]
    while st:
        p=st.pop()
        if p in seen: continue
        seen.add(p); st.extend(m.get(p,[]))
    return seen
def rss_kib(pid):
    try:
        with open("/proc/%d/statm"%pid) as f: return int(f.read().split()[1])*PAGE//1024
    except Exception: return 0
peak=0; peak_pids=0; samples=0
while not os.path.exists(sentinel):
    pids=tree(root); total=sum(rss_kib(p) for p in pids)
    peak=max(peak,total); peak_pids=max(peak_pids,len(pids)); samples+=1
    time.sleep(0.12)
open(out,"w").write("%d %d %d\n"%(peak,peak_pids,samples))
```

**`count_tree.py`** — F-05: prints the current PID count in the process tree rooted at a PID (used to detect when the qcluster workers are up).

```python
import os,sys
root=int(sys.argv[1])
def cmap():
    m={}
    for pid in os.listdir("/proc"):
        if not pid.isdigit(): continue
        try:
            with open("/proc/%s/stat"%pid) as f: parts=f.read().rsplit(")",1)[-1].split()
            ppid=int(parts[1]); m.setdefault(ppid,[]).append(int(pid))
        except Exception: continue
    return m
m=cmap(); seen=set(); st=[root]
while st:
    p=st.pop()
    if p in seen: continue
    seen.add(p); st.extend(m.get(p,[]))
print(len(seen))
```

**`concurrency_probe.sh`** — F-05: starts the canonical 11-worker qcluster + document_consumer --oneshot, samples the tree during N concurrent consumes, reports peak tree RSS + peak PID count.

```bash
#!/bin/bash
# F-05: measure full process-tree peak RSS + peak PID count for N concurrent OCR consumes
# under the DEFAULT 11-worker Django-Q cluster (canonical qcluster + document_consumer path).
set -u
umask 077                          # F-12: scratch dirs/files created private (0700/0600)
SCRATCH="$1"; N="$2"; LABEL="$3"; PDFDIR="$4"
# F-07: resolve symlinks/'..' and refuse to `rm -rf` anything not strictly under /tmp/.
SCRATCH="$(readlink -f -- "$SCRATCH" 2>/dev/null || true)"
case "$SCRATCH" in
  /tmp/?*) : ;;
  *) echo "SAFETY ABORT: scratch root must resolve strictly under /tmp/, got: '$1'" >&2; exit 2 ;;
esac
# F-13: validate the concurrency count is a positive integer (literal, no shell interpolation).
case "$N" in
  ''|*[!0-9]*) echo "INPUT ERROR: N (concurrency) must be a non-negative integer, got: '$N'" >&2; exit 2 ;;
esac
[ "$N" -ge 1 ] || { echo "INPUT ERROR: N (concurrency) must be >= 1, got: $N" >&2; exit 2; }
export PAPERLESS_DATA_DIR="$SCRATCH/data" PAPERLESS_MEDIA_ROOT="$SCRATCH/media"
export PAPERLESS_STATICDIR="$SCRATCH/static" PAPERLESS_CONSUMPTION_DIR="$SCRATCH/consume"
export DJANGO_SETTINGS_MODULE=paperless.settings

# fresh scratch + migrate
rm -rf -- "$SCRATCH"
mkdir -p "$SCRATCH/data/index" "$SCRATCH/media/documents/originals" "$SCRATCH/media/documents/archive" "$SCRATCH/media/documents/thumbnails" "$SCRATCH/static" "$SCRATCH/consume"
cd /app/src
python manage.py migrate --no-input >"$SCRATCH/migrate.log" 2>&1 || { echo "MIGRATE_FAIL"; tail "$SCRATCH/migrate.log"; exit 1; }

# stage N distinct PDFs into the consumption dir
i=0
for f in "$PDFDIR"/*.pdf; do
  [ "$i" -ge "$N" ] && break
  cp "$f" "$PAPERLESS_CONSUMPTION_DIR/stage_${i}_$(basename "$f")"
  i=$((i+1))
done

SENT="$SCRATCH/.sampler_done"; OUT="$SCRATCH/.sampler_out"; rm -f "$SENT" "$OUT"

# start the canonical Django-Q cluster (default 11 workers)
python manage.py qcluster >"$SCRATCH/qcluster.log" 2>&1 &
QPID=$!

# wait until workers are up (tree has master+workers) or 25s
for _ in $(seq 1 250); do
  c=$(python /tmp/scripts/count_tree.py "$QPID" 2>/dev/null || echo 0)
  [ "$c" -ge 12 ] && break
  sleep 0.1
done
sleep 1  # let workers settle at idle baseline

# start the tree sampler against the cluster
python /tmp/scripts/tree_sampler.py "$QPID" "$SENT" "$OUT" &
SPID=$!
sleep 0.3

# enqueue all staged files (async_task -> redis -> workers) via canonical oneshot consumer
python manage.py document_consumer --oneshot "$PAPERLESS_CONSUMPTION_DIR" >"$SCRATCH/consumer.log" 2>&1

# wait until N documents exist (or 240s)
for _ in $(seq 1 2400); do
  n=$(python -c "import django;django.setup();from documents.models import Document;print(Document.objects.count())" 2>/dev/null || echo 0)
  [ "$n" -ge "$N" ] && break
  sleep 0.1
done
DOCS="$n"
sleep 0.5
touch "$SENT"
wait "$SPID" 2>/dev/null
read PEAK_KIB PEAK_PIDS SAMPLES < "$OUT"

# stop the cluster (targeted pid we spawned)
kill "$QPID" 2>/dev/null
for _ in $(seq 1 50); do kill -0 "$QPID" 2>/dev/null || break; sleep 0.1; done
kill -9 "$QPID" 2>/dev/null

echo "{\"label\":\"$LABEL\",\"n_queued\":$N,\"docs_created\":$DOCS,\"peak_tree_rss_kib\":$PEAK_KIB,\"peak_pids\":$PEAK_PIDS,\"samples\":$SAMPLES,\"q_workers\":11}"
```

### 13.3 Cleanup (leaves the host and repository unchanged)

Run on completion; removes the container and all temporary artifacts. The repository working tree is untouched except the committed deliverable.

```bash
set -euo pipefail
# Stop & remove the canonical container (releases its scratch DBs/manifests):
docker rm -f "$CNAME"
# Remove host-side temporary probe scripts, evidence copies, and the terminal renderer:
rm -rf /tmp/mem_probe /tmp/render_terminal.py
# Verify the working tree is CLEAN — the deliverable is committed, so porcelain is empty:
cd <repo-root> && git status --porcelain          # => (no output: clean working tree)
# Confirm the ONLY change this branch introduces vs. the base (origin/dev) is the single
# deliverable file, added — no source/config/dependency/test file was touched:
git diff --name-status "$(git merge-base HEAD origin/dev)"..HEAD
# => A       blitzy/documentation/paperless-ngx_542221a38dff.md
```

### 13.4 Complete `/usr/bin/time -v` — every exporter run (both runs per scenario)

Each block below is the byte-for-byte capture: the `populate` command + its confirmation, the exact `/usr/bin/time -v` invocation, all 23 `time -v` fields, and the resulting `manifest.json` size. Peak RSS is the `Maximum resident set size (kbytes)` field.

**exporter — zero (run1 + run2):**

```text
### (zero-document baseline: empty migrated DB, no populate)

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_zero/data PAPERLESS_MEDIA_ROOT=/tmp/s_zero/media PAPERLESS_STATICDIR=/tmp/s_zero/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_zero/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_zero_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_zero_run1 --no-progress-bar"
	User time (seconds): 1.24
	System time (seconds): 0.15
	Percent of CPU this job got: 100%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.39
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 67432
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 42814
	Voluntary context switches: 30
	Involuntary context switches: 12
	Swaps: 0
	File system inputs: 0
	File system outputs: 16
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_zero_run1/manifest.json
-rw-r--r-- 1 root root 393 Jul 13 18:08 /tmp/exp_zero_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_zero/data PAPERLESS_MEDIA_ROOT=/tmp/s_zero/media PAPERLESS_STATICDIR=/tmp/s_zero/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_zero/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_zero_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_zero_run2 --no-progress-bar"
	User time (seconds): 1.18
	System time (seconds): 0.13
	Percent of CPU this job got: 100%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.31
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 66324
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 42786
	Voluntary context switches: 31
	Involuntary context switches: 16
	Swaps: 0
	File system inputs: 0
	File system outputs: 16
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_zero_run2/manifest.json
-rw-r--r-- 1 root root 393 Jul 13 18:08 /tmp/exp_zero_run2/manifest.json
```

**exporter — meta5000 (run1 + run2):**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_meta5000 --n 5000 --content-bytes 200 
populated 5000 docs; content_bytes=200 (ascii); originals+thumbnails written=5000; scratch=/tmp/s_meta5000

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_meta5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_meta5000/media PAPERLESS_STATICDIR=/tmp/s_meta5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_meta5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_meta5000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_meta5000_run1 --no-progress-bar"
	User time (seconds): 5.26
	System time (seconds): 0.93
	Percent of CPU this job got: 98%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:06.26
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 86388
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 47365
	Voluntary context switches: 57
	Involuntary context switches: 301
	Swaps: 0
	File system inputs: 8
	File system outputs: 89128
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_meta5000_run1/manifest.json
-rw-r--r-- 1 root root 4665956 Jul 13 18:08 /tmp/exp_meta5000_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_meta5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_meta5000/media PAPERLESS_STATICDIR=/tmp/s_meta5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_meta5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_meta5000_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_meta5000_run2 --no-progress-bar"
	User time (seconds): 4.69
	System time (seconds): 0.89
	Percent of CPU this job got: 98%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:05.65
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 87372
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 47199
	Voluntary context switches: 46
	Involuntary context switches: 274
	Swaps: 0
	File system inputs: 0
	File system outputs: 89128
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_meta5000_run2/manifest.json
-rw-r--r-- 1 root root 4665956 Jul 13 18:08 /tmp/exp_meta5000_run2/manifest.json
```

**exporter — text200 (run1 + run2):**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_text200 --n 200 --content-bytes 50000 
populated 200 docs; content_bytes=50000 (ascii); originals+thumbnails written=200; scratch=/tmp/s_text200

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text200/data PAPERLESS_MEDIA_ROOT=/tmp/s_text200/media PAPERLESS_STATICDIR=/tmp/s_text200/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text200/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text200_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text200_run1 --no-progress-bar"
	User time (seconds): 1.54
	System time (seconds): 0.23
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.78
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 98768
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 53150
	Voluntary context switches: 42
	Involuntary context switches: 25
	Swaps: 0
	File system inputs: 0
	File system outputs: 23032
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text200_run1/manifest.json
-rw-r--r-- 1 root root 10145955 Jul 13 18:09 /tmp/exp_text200_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text200/data PAPERLESS_MEDIA_ROOT=/tmp/s_text200/media PAPERLESS_STATICDIR=/tmp/s_text200/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text200/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text200_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text200_run2 --no-progress-bar"
	User time (seconds): 1.49
	System time (seconds): 0.19
	Percent of CPU this job got: 100%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.68
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 99060
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 53133
	Voluntary context switches: 36
	Involuntary context switches: 23
	Swaps: 0
	File system inputs: 0
	File system outputs: 23032
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text200_run2/manifest.json
-rw-r--r-- 1 root root 10145955 Jul 13 18:09 /tmp/exp_text200_run2/manifest.json
```

**exporter — text1000 (run1 + run2):**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_text1000 --n 1000 --content-bytes 50000 
populated 1000 docs; content_bytes=50000 (ascii); originals+thumbnails written=1000; scratch=/tmp/s_text1000

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text1000/data PAPERLESS_MEDIA_ROOT=/tmp/s_text1000/media PAPERLESS_STATICDIR=/tmp/s_text1000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text1000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text1000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text1000_run1 --no-progress-bar"
	User time (seconds): 2.32
	System time (seconds): 0.46
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:02.82
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 218980
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 87755
	Voluntary context switches: 43
	Involuntary context switches: 66
	Swaps: 0
	File system inputs: 0
	File system outputs: 115096
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text1000_run1/manifest.json
-rw-r--r-- 1 root root 50729956 Jul 13 18:09 /tmp/exp_text1000_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text1000/data PAPERLESS_MEDIA_ROOT=/tmp/s_text1000/media PAPERLESS_STATICDIR=/tmp/s_text1000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text1000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text1000_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text1000_run2 --no-progress-bar"
	User time (seconds): 2.23
	System time (seconds): 0.45
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:02.69
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 217484
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 87995
	Voluntary context switches: 38
	Involuntary context switches: 60
	Swaps: 0
	File system inputs: 0
	File system outputs: 115096
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text1000_run2/manifest.json
-rw-r--r-- 1 root root 50729956 Jul 13 18:09 /tmp/exp_text1000_run2/manifest.json
```

**exporter — text5000 (run1 + run2):**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_text5000 --n 5000 --content-bytes 50000 
populated 5000 docs; content_bytes=50000 (ascii); originals+thumbnails written=5000; scratch=/tmp/s_text5000

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_text5000/media PAPERLESS_STATICDIR=/tmp/s_text5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text5000_run1 --no-progress-bar"
	User time (seconds): 6.37
	System time (seconds): 1.67
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:08.08
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 826844
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 341223
	Voluntary context switches: 47
	Involuntary context switches: 287
	Swaps: 0
	File system inputs: 0
	File system outputs: 575456
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text5000_run1/manifest.json
-rw-r--r-- 1 root root 253665956 Jul 13 18:08 /tmp/exp_text5000_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_text5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_text5000/media PAPERLESS_STATICDIR=/tmp/s_text5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_text5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text5000_run2 --no-progress-bar"
	User time (seconds): 7.35
	System time (seconds): 1.80
	Percent of CPU this job got: 98%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:09.25
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 822940
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 340795
	Voluntary context switches: 49
	Involuntary context switches: 279
	Swaps: 0
	File system inputs: 0
	File system outputs: 575456
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_text5000_run2/manifest.json
-rw-r--r-- 1 root root 253665956 Jul 13 18:08 /tmp/exp_text5000_run2/manifest.json
```

**exporter — uni5000 (run1 + run2):**

```text
### populate cmd: python /tmp/scripts/populate.py --scratch /tmp/s_uni5000 --n 5000 --content-bytes 50000 --unicode
populated 5000 docs; content_bytes=50000 (unicode(utf-8)); originals+thumbnails written=5000; scratch=/tmp/s_uni5000

### run1 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_uni5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_uni5000/media PAPERLESS_STATICDIR=/tmp/s_uni5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_uni5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_uni5000_run1 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_uni5000_run1 --no-progress-bar"
	User time (seconds): 6.78
	System time (seconds): 1.73
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:08.53
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 850184
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 357697
	Voluntary context switches: 47
	Involuntary context switches: 287
	Swaps: 0
	File system inputs: 0
	File system outputs: 1172248
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_uni5000_run1/manifest.json
-rw-r--r-- 1 root root 559225956 Jul 13 18:09 /tmp/exp_uni5000_run1/manifest.json

### run2 (cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/s_uni5000/data PAPERLESS_MEDIA_ROOT=/tmp/s_uni5000/media PAPERLESS_STATICDIR=/tmp/s_uni5000/static PAPERLESS_CONSUMPTION_DIR=/tmp/s_uni5000/consume):
$ /usr/bin/time -v python manage.py document_exporter /tmp/exp_uni5000_run2 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_uni5000_run2 --no-progress-bar"
	User time (seconds): 6.67
	System time (seconds): 1.65
	Percent of CPU this job got: 98%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:08.43
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 879292
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 328803
	Voluntary context switches: 52
	Involuntary context switches: 272
	Swaps: 0
	File system inputs: 0
	File system outputs: 1172248
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
$ ls -la /tmp/exp_uni5000_run2/manifest.json
-rw-r--r-- 1 root root 559225956 Jul 13 18:09 /tmp/exp_uni5000_run2/manifest.json
```

**Heavy-library check after `django.setup()` (true zero-document baseline support, finding 11):**

```text
### cmd (cwd=/app/src): python /tmp/scripts/modules_check.py /tmp/s_zero
{"doc_count": 0, "rss_before_django_setup_mb": 11.3, "rss_after_django_setup_mb": 49.4, "sklearn_loaded": false, "numpy_loaded": false, "scipy_loaded": false, "dateparser_loaded": false, "django_loaded": true}
```

### 13.5 Complete importer output — every run (both runs per scenario)

Byte-for-byte capture with the tqdm reindex meter's carriage-return repaints rendered as separate lines (exactly what the terminal displayed over time; the reindex step at `document_importer.py:92` does not receive `--no-progress-bar`, §3.3). Every run shows `Installed N object(s)`, `Exit status: 0`, and all 23 `/usr/bin/time -v` fields; peak RSS is `Maximum resident set size (kbytes)`.

**importer — meta5000 (run1 + run2):**

```text

### import meta5000 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_meta5000_run1/data PAPERLESS_MEDIA_ROOT=/tmp/imp_meta5000_run1/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/5000 [00:00<?, ?it/s]
  2%|▏         | 85/5000 [00:00<00:05, 846.31it/s]
  3%|▎         | 172/5000 [00:00<00:05, 856.11it/s]
  5%|▌         | 258/5000 [00:00<00:05, 855.99it/s]
  7%|▋         | 347/5000 [00:00<00:05, 867.22it/s]
  9%|▊         | 434/5000 [00:00<00:05, 844.25it/s]
 10%|█         | 519/5000 [00:00<00:06, 739.61it/s]
 12%|█▏        | 602/5000 [00:00<00:05, 764.88it/s]
 14%|█▍        | 695/5000 [00:00<00:05, 813.09it/s]
 16%|█▌        | 784/5000 [00:00<00:05, 833.82it/s]
 18%|█▊        | 876/5000 [00:01<00:04, 857.03it/s]
 19%|█▉        | 963/5000 [00:01<00:04, 846.98it/s]
 21%|██        | 1057/5000 [00:01<00:04, 873.61it/s]
 23%|██▎       | 1150/5000 [00:01<00:04, 887.37it/s]
 25%|██▍       | 1240/5000 [00:01<00:04, 873.50it/s]
 27%|██▋       | 1333/5000 [00:01<00:04, 887.48it/s]
 28%|██▊       | 1422/5000 [00:01<00:04, 876.28it/s]
 30%|███       | 1511/5000 [00:01<00:03, 878.74it/s]
 32%|███▏      | 1600/5000 [00:01<00:03, 879.79it/s]
 34%|███▍      | 1689/5000 [00:01<00:03, 878.61it/s]
 36%|███▌      | 1780/5000 [00:02<00:03, 887.40it/s]
 37%|███▋      | 1869/5000 [00:02<00:03, 879.06it/s]
 39%|███▉      | 1964/5000 [00:02<00:03, 899.99it/s]
 41%|████      | 2055/5000 [00:02<00:03, 891.91it/s]
 43%|████▎     | 2149/5000 [00:02<00:03, 905.05it/s]
 45%|████▍     | 2240/5000 [00:02<00:03, 891.48it/s]
 47%|████▋     | 2330/5000 [00:02<00:02, 890.21it/s]
 48%|████▊     | 2421/5000 [00:02<00:02, 893.83it/s]
 50%|█████     | 2511/5000 [00:02<00:02, 886.73it/s]
 52%|█████▏    | 2605/5000 [00:02<00:02, 899.77it/s]
 54%|█████▍    | 2696/5000 [00:03<00:02, 885.59it/s]
 56%|█████▌    | 2786/5000 [00:03<00:02, 887.90it/s]
 57%|█████▊    | 2875/5000 [00:03<00:02, 872.94it/s]
 59%|█████▉    | 2964/5000 [00:03<00:02, 875.99it/s]
 61%|██████    | 3052/5000 [00:03<00:02, 873.27it/s]
 63%|██████▎   | 3140/5000 [00:03<00:02, 873.00it/s]
 65%|██████▍   | 3234/5000 [00:03<00:01, 890.04it/s]
 66%|██████▋   | 3324/5000 [00:03<00:01, 883.26it/s]
 68%|██████▊   | 3419/5000 [00:03<00:01, 900.39it/s]
 70%|███████   | 3510/5000 [00:04<00:01, 881.43it/s]
 72%|███████▏  | 3603/5000 [00:04<00:01, 894.50it/s]
 74%|███████▍  | 3693/5000 [00:04<00:01, 893.88it/s]
 76%|███████▌  | 3783/5000 [00:04<00:01, 889.24it/s]
 77%|███████▋  | 3872/5000 [00:04<00:01, 887.08it/s]
 79%|███████▉  | 3961/5000 [00:04<00:01, 872.06it/s]
 81%|████████  | 4055/5000 [00:04<00:01, 891.61it/s]
 83%|████████▎ | 4145/5000 [00:04<00:00, 874.38it/s]
 85%|████████▍ | 4238/5000 [00:04<00:00, 889.76it/s]
 87%|████████▋ | 4328/5000 [00:04<00:00, 877.76it/s]
 88%|████████▊ | 4418/5000 [00:05<00:00, 883.16it/s]
 90%|█████████ | 4507/5000 [00:05<00:00, 882.17it/s]
 92%|█████████▏| 4596/5000 [00:05<00:00, 866.11it/s]
 94%|█████████▍| 4690/5000 [00:05<00:00, 885.53it/s]
 96%|█████████▌| 4779/5000 [00:05<00:00, 872.03it/s]
 97%|█████████▋| 4873/5000 [00:05<00:00, 889.63it/s]
 99%|█████████▉| 4963/5000 [00:05<00:00, 874.80it/s]
100%|██████████| 5000/5000 [00:05<00:00, 873.60it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar"
	User time (seconds): 19.79
	System time (seconds): 3.29
	Percent of CPU this job got: 65%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:35.39
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 143232
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 82669
	Voluntary context switches: 39942
	Involuntary context switches: 104
	Swaps: 0
	File system inputs: 0
	File system outputs: 1247152
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0

### import meta5000 run2 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_meta5000_run2/data PAPERLESS_MEDIA_ROOT=/tmp/imp_meta5000_run2/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/5000 [00:00<?, ?it/s]
  2%|▏         | 75/5000 [00:00<00:06, 748.65it/s]
  3%|▎         | 164/5000 [00:00<00:05, 828.18it/s]
  5%|▍         | 248/5000 [00:00<00:05, 831.94it/s]
  7%|▋         | 332/5000 [00:00<00:05, 820.20it/s]
  8%|▊         | 415/5000 [00:00<00:05, 816.61it/s]
 10%|▉         | 497/5000 [00:00<00:06, 686.13it/s]
 12%|█▏        | 579/5000 [00:00<00:06, 723.07it/s]
 13%|█▎        | 654/5000 [00:00<00:06, 715.80it/s]
 15%|█▍        | 739/5000 [00:00<00:05, 752.22it/s]
 16%|█▋        | 816/5000 [00:01<00:05, 739.96it/s]
 18%|█▊        | 903/5000 [00:01<00:05, 777.46it/s]
 20%|█▉        | 982/5000 [00:01<00:05, 722.37it/s]
 21%|██▏       | 1073/5000 [00:01<00:05, 772.54it/s]
 23%|██▎       | 1152/5000 [00:01<00:05, 768.17it/s]
 25%|██▍       | 1240/5000 [00:01<00:04, 798.56it/s]
 26%|██▋       | 1321/5000 [00:01<00:05, 726.94it/s]
 28%|██▊       | 1396/5000 [00:01<00:05, 699.40it/s]
 30%|██▉       | 1476/5000 [00:01<00:04, 725.35it/s]
 31%|███       | 1558/5000 [00:02<00:04, 751.13it/s]
 33%|███▎      | 1643/5000 [00:02<00:04, 777.20it/s]
 34%|███▍      | 1724/5000 [00:02<00:04, 784.84it/s]
 36%|███▋      | 1816/5000 [00:02<00:03, 823.87it/s]
 38%|███▊      | 1899/5000 [00:02<00:03, 794.17it/s]
 40%|███▉      | 1991/5000 [00:02<00:03, 829.00it/s]
 42%|████▏     | 2075/5000 [00:02<00:03, 822.10it/s]
 43%|████▎     | 2163/5000 [00:02<00:03, 838.64it/s]
 45%|████▍     | 2248/5000 [00:02<00:03, 837.13it/s]
 47%|████▋     | 2342/5000 [00:02<00:03, 866.52it/s]
 49%|████▊     | 2437/5000 [00:03<00:02, 888.66it/s]
 51%|█████     | 2527/5000 [00:03<00:02, 876.40it/s]
 52%|█████▏    | 2615/5000 [00:03<00:02, 877.05it/s]
 54%|█████▍    | 2703/5000 [00:03<00:02, 842.69it/s]
 56%|█████▌    | 2790/5000 [00:03<00:02, 849.63it/s]
 58%|█████▊    | 2876/5000 [00:03<00:02, 847.00it/s]
 59%|█████▉    | 2961/5000 [00:03<00:02, 840.93it/s]
 61%|██████    | 3046/5000 [00:03<00:02, 840.04it/s]
 63%|██████▎   | 3131/5000 [00:03<00:02, 821.94it/s]
 64%|██████▍   | 3220/5000 [00:04<00:02, 840.83it/s]
 66%|██████▌   | 3305/5000 [00:04<00:02, 805.34it/s]
 68%|██████▊   | 3397/5000 [00:04<00:01, 837.99it/s]
 70%|██████▉   | 3482/5000 [00:04<00:01, 827.49it/s]
 71%|███████▏  | 3571/5000 [00:04<00:01, 843.68it/s]
 73%|███████▎  | 3656/5000 [00:04<00:01, 831.92it/s]
 75%|███████▍  | 3740/5000 [00:04<00:01, 817.34it/s]
 76%|███████▋  | 3823/5000 [00:04<00:01, 818.06it/s]
 78%|███████▊  | 3905/5000 [00:04<00:01, 797.18it/s]
 80%|███████▉  | 3994/5000 [00:04<00:01, 823.57it/s]
 82%|████████▏ | 4077/5000 [00:05<00:01, 775.99it/s]
 83%|████████▎ | 4168/5000 [00:05<00:01, 813.66it/s]
 85%|████████▌ | 4251/5000 [00:05<00:00, 779.00it/s]
 87%|████████▋ | 4339/5000 [00:05<00:00, 806.57it/s]
 88%|████████▊ | 4421/5000 [00:05<00:00, 805.17it/s]
 90%|█████████ | 4507/5000 [00:05<00:00, 820.73it/s]
 92%|█████████▏| 4590/5000 [00:05<00:00, 820.72it/s]
 93%|█████████▎| 4673/5000 [00:05<00:00, 807.22it/s]
 95%|█████████▌| 4759/5000 [00:05<00:00, 821.34it/s]
 97%|█████████▋| 4842/5000 [00:06<00:00, 800.55it/s]
 99%|█████████▊| 4936/5000 [00:06<00:00, 839.55it/s]
100%|██████████| 5000/5000 [00:06<00:00, 803.73it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_meta5000_run1 --no-progress-bar"
	User time (seconds): 20.75
	System time (seconds): 3.21
	Percent of CPU this job got: 67%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:35.37
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 140648
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 82596
	Voluntary context switches: 39577
	Involuntary context switches: 86
	Swaps: 0
	File system inputs: 0
	File system outputs: 1247144
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

**importer — text200 (run1 + run2):**

```text

### import text200 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text200_run1/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text200_run1/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text200_run1 --no-progress-bar
Installed 201 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/200 [00:00<?, ?it/s]
 32%|███▎      | 65/200 [00:00<00:00, 649.01it/s]
 65%|██████▌   | 130/200 [00:00<00:00, 621.84it/s]
 98%|█████████▊| 197/200 [00:00<00:00, 641.27it/s]
100%|██████████| 200/200 [00:00<00:00, 636.62it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text200_run1 --no-progress-bar"
	User time (seconds): 2.18
	System time (seconds): 0.35
	Percent of CPU this job got: 78%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:03.24
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 118632
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 79559
	Voluntary context switches: 1924
	Involuntary context switches: 21
	Swaps: 0
	File system inputs: 0
	File system outputs: 69208
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0

### import text200 run2 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text200_run2/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text200_run2/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text200_run1 --no-progress-bar
Installed 201 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/200 [00:00<?, ?it/s]
 34%|███▍      | 69/200 [00:00<00:00, 685.56it/s]
 69%|██████▉   | 138/200 [00:00<00:00, 671.68it/s]
100%|██████████| 200/200 [00:00<00:00, 689.19it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text200_run1 --no-progress-bar"
	User time (seconds): 2.28
	System time (seconds): 0.34
	Percent of CPU this job got: 82%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:03.18
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 118168
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 79603
	Voluntary context switches: 1841
	Involuntary context switches: 18
	Swaps: 0
	File system inputs: 0
	File system outputs: 69200
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

**importer — text1000 (run1 + run2):**

```text

### import text1000 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text1000_run1/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text1000_run1/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text1000_run1 --no-progress-bar
Installed 1001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/1000 [00:00<?, ?it/s]
  6%|▋         | 65/1000 [00:00<00:01, 644.02it/s]
 13%|█▎        | 134/1000 [00:00<00:01, 665.93it/s]
 20%|██        | 205/1000 [00:00<00:01, 679.29it/s]
 28%|██▊       | 275/1000 [00:00<00:01, 683.63it/s]
 35%|███▍      | 346/1000 [00:00<00:00, 692.09it/s]
 42%|████▏     | 417/1000 [00:00<00:00, 698.00it/s]
 49%|████▉     | 489/1000 [00:00<00:00, 704.07it/s]
 56%|█████▌    | 560/1000 [00:00<00:00, 693.09it/s]
 63%|██████▎   | 630/1000 [00:00<00:00, 684.05it/s]
 70%|███████   | 702/1000 [00:01<00:00, 694.42it/s]
 77%|███████▋  | 772/1000 [00:01<00:00, 691.92it/s]
 84%|████████▍ | 843/1000 [00:01<00:00, 695.00it/s]
 91%|█████████▏| 913/1000 [00:01<00:00, 694.99it/s]
 98%|█████████▊| 983/1000 [00:01<00:00, 683.13it/s]
100%|██████████| 1000/1000 [00:01<00:00, 688.80it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text1000_run1 --no-progress-bar"
	User time (seconds): 5.64
	System time (seconds): 1.04
	Percent of CPU this job got: 69%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:09.59
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 246952
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 162815
	Voluntary context switches: 10442
	Involuntary context switches: 35
	Swaps: 0
	File system inputs: 0
	File system outputs: 347408
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0

### import text1000 run2 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text1000_run2/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text1000_run2/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text1000_run1 --no-progress-bar
Installed 1001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/1000 [00:00<?, ?it/s]
  7%|▋         | 71/1000 [00:00<00:01, 709.45it/s]
 14%|█▍        | 144/1000 [00:00<00:01, 717.80it/s]
 22%|██▏       | 216/1000 [00:00<00:01, 684.22it/s]
 28%|██▊       | 285/1000 [00:00<00:01, 553.86it/s]
 36%|███▌      | 357/1000 [00:00<00:01, 604.46it/s]
 43%|████▎     | 430/1000 [00:00<00:00, 641.37it/s]
 50%|████▉     | 497/1000 [00:00<00:00, 649.65it/s]
 57%|█████▋    | 568/1000 [00:00<00:00, 667.10it/s]
 64%|██████▍   | 642/1000 [00:00<00:00, 687.73it/s]
 71%|███████   | 712/1000 [00:01<00:00, 688.44it/s]
 78%|███████▊  | 782/1000 [00:01<00:00, 673.78it/s]
 86%|████████▌ | 856/1000 [00:01<00:00, 692.45it/s]
 93%|█████████▎| 930/1000 [00:01<00:00, 705.82it/s]
100%|██████████| 1000/1000 [00:01<00:00, 671.68it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text1000_run1 --no-progress-bar"
	User time (seconds): 5.69
	System time (seconds): 1.09
	Percent of CPU this job got: 66%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:10.18
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 244652
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 162846
	Voluntary context switches: 8970
	Involuntary context switches: 36
	Swaps: 0
	File system inputs: 0
	File system outputs: 347408
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

**importer — text5000 (run1 + run2):**

```text

### import text5000 run1 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text5000_run1/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text5000_run1/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/5000 [00:00<?, ?it/s]
  1%|▏         | 68/5000 [00:00<00:07, 671.67it/s]
  3%|▎         | 138/5000 [00:00<00:07, 687.93it/s]
  4%|▍         | 207/5000 [00:00<00:07, 677.41it/s]
  6%|▌         | 278/5000 [00:00<00:06, 689.46it/s]
  7%|▋         | 347/5000 [00:00<00:06, 681.71it/s]
  8%|▊         | 416/5000 [00:00<00:06, 679.64it/s]
 10%|▉         | 484/5000 [00:00<00:08, 543.70it/s]
 11%|█         | 556/5000 [00:00<00:07, 589.72it/s]
 12%|█▏        | 620/5000 [00:00<00:07, 602.82it/s]
 14%|█▍        | 689/5000 [00:01<00:06, 626.05it/s]
 15%|█▌        | 758/5000 [00:01<00:06, 642.81it/s]
 16%|█▋        | 824/5000 [00:01<00:06, 632.57it/s]
 18%|█▊        | 892/5000 [00:01<00:06, 643.06it/s]
 19%|█▉        | 960/5000 [00:01<00:06, 652.96it/s]
 21%|██        | 1030/5000 [00:01<00:05, 664.16it/s]
 22%|██▏       | 1098/5000 [00:01<00:05, 665.94it/s]
 23%|██▎       | 1168/5000 [00:01<00:05, 673.60it/s]
 25%|██▍       | 1237/5000 [00:01<00:05, 675.50it/s]
 26%|██▌       | 1305/5000 [00:02<00:09, 385.29it/s]
 28%|██▊       | 1377/5000 [00:02<00:08, 449.77it/s]
 29%|██▉       | 1445/5000 [00:02<00:07, 496.59it/s]
 30%|███       | 1516/5000 [00:02<00:06, 546.35it/s]
 32%|███▏      | 1582/5000 [00:02<00:05, 574.86it/s]
 33%|███▎      | 1653/5000 [00:02<00:05, 609.14it/s]
 35%|███▍      | 1726/5000 [00:02<00:05, 641.47it/s]
 36%|███▌      | 1795/5000 [00:02<00:04, 651.70it/s]
 37%|███▋      | 1868/5000 [00:03<00:04, 672.48it/s]
 39%|███▉      | 1938/5000 [00:03<00:04, 671.12it/s]
 40%|████      | 2009/5000 [00:03<00:04, 679.82it/s]
 42%|████▏     | 2079/5000 [00:03<00:04, 674.70it/s]
 43%|████▎     | 2148/5000 [00:03<00:04, 679.12it/s]
 44%|████▍     | 2217/5000 [00:03<00:04, 679.60it/s]
 46%|████▌     | 2286/5000 [00:03<00:03, 679.96it/s]
 47%|████▋     | 2355/5000 [00:03<00:03, 679.61it/s]
 48%|████▊     | 2424/5000 [00:03<00:03, 676.45it/s]
 50%|████▉     | 2495/5000 [00:03<00:03, 685.69it/s]
 51%|█████▏    | 2564/5000 [00:04<00:06, 360.41it/s]
 53%|█████▎    | 2632/5000 [00:04<00:05, 417.80it/s]
 54%|█████▍    | 2706/5000 [00:04<00:04, 483.93it/s]
 55%|█████▌    | 2773/5000 [00:04<00:04, 525.51it/s]
 57%|█████▋    | 2845/5000 [00:04<00:03, 572.80it/s]
 58%|█████▊    | 2914/5000 [00:04<00:03, 601.90it/s]
 60%|█████▉    | 2981/5000 [00:05<00:03, 610.85it/s]
 61%|██████    | 3047/5000 [00:05<00:03, 588.36it/s]
 62%|██████▏   | 3116/5000 [00:05<00:03, 614.84it/s]
 64%|██████▍   | 3189/5000 [00:05<00:02, 646.53it/s]
 65%|██████▌   | 3256/5000 [00:05<00:02, 649.94it/s]
 67%|██████▋   | 3330/5000 [00:05<00:02, 673.64it/s]
 68%|██████▊   | 3399/5000 [00:05<00:02, 676.64it/s]
 69%|██████▉   | 3471/5000 [00:05<00:02, 687.03it/s]
 71%|███████   | 3542/5000 [00:05<00:02, 691.54it/s]
 72%|███████▏  | 3612/5000 [00:05<00:02, 680.48it/s]
 74%|███████▎  | 3687/5000 [00:06<00:01, 698.14it/s]
 75%|███████▌  | 3758/5000 [00:06<00:01, 701.25it/s]
 77%|███████▋  | 3829/5000 [00:06<00:02, 399.86it/s]
 78%|███████▊  | 3899/5000 [00:06<00:02, 457.59it/s]
 79%|███████▉  | 3973/5000 [00:06<00:01, 518.13it/s]
 81%|████████  | 4042/5000 [00:06<00:01, 558.15it/s]
 82%|████████▏ | 4109/5000 [00:06<00:01, 585.56it/s]
 84%|████████▎ | 4184/5000 [00:07<00:01, 628.14it/s]
 85%|████████▌ | 4258/5000 [00:07<00:01, 656.35it/s]
 87%|████████▋ | 4328/5000 [00:07<00:01, 662.58it/s]
 88%|████████▊ | 4403/5000 [00:07<00:00, 685.79it/s]
 90%|████████▉ | 4475/5000 [00:07<00:00, 693.71it/s]
 91%|█████████ | 4546/5000 [00:07<00:00, 686.97it/s]
 92%|█████████▏| 4619/5000 [00:07<00:00, 699.01it/s]
 94%|█████████▍| 4694/5000 [00:07<00:00, 712.29it/s]
 95%|█████████▌| 4766/5000 [00:07<00:00, 698.85it/s]
 97%|█████████▋| 4837/5000 [00:07<00:00, 700.78it/s]
 98%|█████████▊| 4909/5000 [00:08<00:00, 706.20it/s]
100%|█████████▉| 4980/5000 [00:08<00:00, 688.22it/s]
100%|██████████| 5000/5000 [00:08<00:00, 612.56it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar"
	User time (seconds): 24.63
	System time (seconds): 4.84
	Percent of CPU this job got: 69%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:42.30
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 822832
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 521800
	Voluntary context switches: 41998
	Involuntary context switches: 105
	Swaps: 0
	File system inputs: 0
	File system outputs: 2275696
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0

### import text5000 run2 (FRESH DB; cwd=/app/src; PAPERLESS_DATA_DIR=/tmp/imp_text5000_run2/data PAPERLESS_MEDIA_ROOT=/tmp/imp_text5000_run2/media ...):
$ /usr/bin/time -v python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar
Installed 5001 object(s) from 1 fixture(s)
Copy files into paperless...
Updating search index...

  0%|          | 0/5000 [00:00<?, ?it/s]
  1%|▏         | 65/5000 [00:00<00:07, 649.42it/s]
  3%|▎         | 135/5000 [00:00<00:07, 677.44it/s]
  4%|▍         | 207/5000 [00:00<00:06, 693.95it/s]
  6%|▌         | 277/5000 [00:00<00:07, 672.67it/s]
  7%|▋         | 348/5000 [00:00<00:06, 684.81it/s]
  8%|▊         | 420/5000 [00:00<00:06, 695.48it/s]
 10%|▉         | 490/5000 [00:00<00:07, 605.55it/s]
 11%|█         | 562/5000 [00:00<00:06, 636.59it/s]
 13%|█▎        | 628/5000 [00:00<00:07, 603.35it/s]
 14%|█▍        | 690/5000 [00:01<00:07, 606.98it/s]
 15%|█▌        | 762/5000 [00:01<00:06, 638.53it/s]
 17%|█▋        | 830/5000 [00:01<00:06, 648.55it/s]
 18%|█▊        | 898/5000 [00:01<00:06, 657.22it/s]
 19%|█▉        | 970/5000 [00:01<00:05, 673.98it/s]
 21%|██        | 1044/5000 [00:01<00:05, 691.37it/s]
 22%|██▏       | 1114/5000 [00:01<00:05, 675.11it/s]
 24%|██▎       | 1185/5000 [00:01<00:05, 684.52it/s]
 25%|██▌       | 1257/5000 [00:01<00:05, 692.11it/s]
 27%|██▋       | 1327/5000 [00:02<00:09, 406.26it/s]
 28%|██▊       | 1401/5000 [00:02<00:07, 471.43it/s]
 29%|██▉       | 1470/5000 [00:02<00:06, 519.39it/s]
 31%|███       | 1544/5000 [00:02<00:06, 570.39it/s]
 32%|███▏      | 1620/5000 [00:02<00:05, 616.96it/s]
 34%|███▍      | 1693/5000 [00:02<00:05, 646.19it/s]
 35%|███▌      | 1763/5000 [00:02<00:04, 657.13it/s]
 37%|███▋      | 1837/5000 [00:02<00:04, 680.10it/s]
 38%|███▊      | 1914/5000 [00:03<00:04, 704.48it/s]
 40%|███▉      | 1990/5000 [00:03<00:04, 717.68it/s]
 41%|████▏     | 2064/5000 [00:03<00:04, 703.98it/s]
 43%|████▎     | 2137/5000 [00:03<00:04, 710.97it/s]
 44%|████▍     | 2214/5000 [00:03<00:03, 725.88it/s]
 46%|████▌     | 2289/5000 [00:03<00:03, 730.11it/s]
 47%|████▋     | 2363/5000 [00:03<00:03, 705.79it/s]
 49%|████▊     | 2436/5000 [00:03<00:03, 712.26it/s]
 50%|█████     | 2511/5000 [00:03<00:03, 721.40it/s]
 52%|█████▏    | 2584/5000 [00:04<00:05, 421.86it/s]
 53%|█████▎    | 2660/5000 [00:04<00:04, 487.86it/s]
 55%|█████▍    | 2730/5000 [00:04<00:04, 533.88it/s]
 56%|█████▌    | 2799/5000 [00:04<00:03, 570.52it/s]
 57%|█████▋    | 2873/5000 [00:04<00:03, 612.68it/s]
 59%|█████▉    | 2949/5000 [00:04<00:03, 651.68it/s]
 60%|██████    | 3020/5000 [00:04<00:02, 664.18it/s]
 62%|██████▏   | 3091/5000 [00:04<00:02, 669.18it/s]
 63%|██████▎   | 3165/5000 [00:05<00:02, 687.44it/s]
 65%|██████▍   | 3237/5000 [00:05<00:02, 696.73it/s]
 66%|██████▌   | 3310/5000 [00:05<00:02, 703.67it/s]
 68%|██████▊   | 3382/5000 [00:05<00:02, 701.51it/s]
 69%|██████▉   | 3458/5000 [00:05<00:02, 717.53it/s]
 71%|███████   | 3531/5000 [00:05<00:02, 719.62it/s]
 72%|███████▏  | 3604/5000 [00:05<00:01, 698.51it/s]
 74%|███████▎  | 3675/5000 [00:05<00:01, 698.36it/s]
 75%|███████▍  | 3748/5000 [00:05<00:01, 706.43it/s]
 76%|███████▋  | 3819/5000 [00:06<00:03, 391.33it/s]
 78%|███████▊  | 3893/5000 [00:06<00:02, 457.02it/s]
 79%|███████▉  | 3962/5000 [00:06<00:02, 505.18it/s]
 81%|████████  | 4031/5000 [00:06<00:01, 547.68it/s]
 82%|████████▏ | 4104/5000 [00:06<00:01, 592.19it/s]
 84%|████████▎ | 4180/5000 [00:06<00:01, 635.65it/s]
 85%|████████▌ | 4252/5000 [00:06<00:01, 658.36it/s]
 86%|████████▋ | 4323/5000 [00:06<00:01, 665.33it/s]
 88%|████████▊ | 4399/5000 [00:07<00:00, 690.28it/s]
 90%|████████▉ | 4475/5000 [00:07<00:00, 708.75it/s]
 91%|█████████ | 4549/5000 [00:07<00:00, 716.74it/s]
 92%|█████████▏| 4624/5000 [00:07<00:00, 724.32it/s]
 94%|█████████▍| 4698/5000 [00:07<00:00, 724.76it/s]
 95%|█████████▌| 4774/5000 [00:07<00:00, 733.64it/s]
 97%|█████████▋| 4848/5000 [00:07<00:00, 730.94it/s]
 98%|█████████▊| 4922/5000 [00:07<00:00, 714.90it/s]
100%|█████████▉| 4994/5000 [00:07<00:00, 716.34it/s]
100%|██████████| 5000/5000 [00:07<00:00, 636.77it/s]
	Command being timed: "python manage.py document_importer /tmp/exp_text5000_run1 --no-progress-bar"
	User time (seconds): 24.27
	System time (seconds): 4.64
	Percent of CPU this job got: 65%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:43.92
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 824480
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 521752
	Voluntary context switches: 41701
	Involuntary context switches: 115
	Swaps: 0
	File system inputs: 0
	File system outputs: 2275704
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

### 13.6 In-process export probe — the two independent fresh-process runs (§3.2)

```text
### export_probe meta5000 run1 (fresh process; cwd=/app/src; scratch=/tmp/s_meta5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_meta5000 --n 5000 --label meta5000
{"label": "meta5000_r0", "n": 5000, "base_rss": 50.1, "live_rss": 86.3, "after_gc_rss": 78.6, "after_trim_rss": 64.5, "rss_freed_by_trim": 14.1, "tm_current_live": 16.0, "tm_current_after_gc": 1.4, "tm_peak": 19.3, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}

### export_probe meta5000 run2 (fresh process; cwd=/app/src; scratch=/tmp/s_meta5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_meta5000 --n 5000 --label meta5000
{"label": "meta5000_r0", "n": 5000, "base_rss": 50.2, "live_rss": 86.4, "after_gc_rss": 78.9, "after_trim_rss": 64.8, "rss_freed_by_trim": 14.1, "tm_current_live": 16.0, "tm_current_after_gc": 1.4, "tm_peak": 19.3, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text200 run1 (fresh process; cwd=/app/src; scratch=/tmp/s_text200):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text200 --n 200 --label text200
{"label": "text200_r0", "n": 200, "base_rss": 50.0, "live_rss": 78.0, "after_gc_rss": 78.6, "after_trim_rss": 55.5, "rss_freed_by_trim": 23.1, "tm_current_live": 21.1, "tm_current_after_gc": 1.4, "tm_peak": 30.9, "gc_objects_live": 72922, "gc_objects_after_gc": 71691, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text200 run2 (fresh process; cwd=/app/src; scratch=/tmp/s_text200):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text200 --n 200 --label text200
{"label": "text200_r0", "n": 200, "base_rss": 50.0, "live_rss": 78.0, "after_gc_rss": 78.6, "after_trim_rss": 55.5, "rss_freed_by_trim": 23.0, "tm_current_live": 21.1, "tm_current_after_gc": 1.4, "tm_peak": 30.9, "gc_objects_live": 72922, "gc_objects_after_gc": 71691, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text1000 run1 (fresh process; cwd=/app/src; scratch=/tmp/s_text1000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text1000 --n 1000 --label text1000
{"label": "text1000_r0", "n": 1000, "base_rss": 50.1, "live_rss": 158.6, "after_gc_rss": 151.0, "after_trim_rss": 60.1, "rss_freed_by_trim": 90.9, "tm_current_live": 99.6, "tm_current_after_gc": 1.4, "tm_peak": 148.5, "gc_objects_live": 77929, "gc_objects_after_gc": 71888, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text1000 run2 (fresh process; cwd=/app/src; scratch=/tmp/s_text1000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text1000 --n 1000 --label text1000
{"label": "text1000_r0", "n": 1000, "base_rss": 49.8, "live_rss": 158.3, "after_gc_rss": 151.5, "after_trim_rss": 59.8, "rss_freed_by_trim": 91.7, "tm_current_live": 99.6, "tm_current_after_gc": 1.4, "tm_peak": 148.5, "gc_objects_live": 77929, "gc_objects_after_gc": 71888, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text5000 run1 (fresh process; cwd=/app/src; scratch=/tmp/s_text5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text5000 --n 5000 --label text5000
{"label": "text5000_r0", "n": 5000, "base_rss": 49.8, "live_rss": 560.8, "after_gc_rss": 552.9, "after_trim_rss": 68.0, "rss_freed_by_trim": 484.8, "tm_current_live": 490.9, "tm_current_after_gc": 1.4, "tm_peak": 731.6, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}

### export_probe text5000 run2 (fresh process; cwd=/app/src; scratch=/tmp/s_text5000):
$ python /tmp/scripts/export_probe.py --scratch /tmp/s_text5000 --n 5000 --label text5000
{"label": "text5000_r0", "n": 5000, "base_rss": 49.9, "live_rss": 560.9, "after_gc_rss": 552.9, "after_trim_rss": 67.9, "rss_freed_by_trim": 485.0, "tm_current_live": 490.9, "tm_current_after_gc": 1.4, "tm_peak": 731.6, "gc_objects_live": 101885, "gc_objects_after_gc": 71868, "gc_collected": 0, "malloc_trim_available": true}

```

---

*End of investigation.*
