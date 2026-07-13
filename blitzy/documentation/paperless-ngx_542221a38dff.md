# Memory Usage During Document Import / Metadata Handling — Investigation & Answer

> **Repository:** paperless-ngx  **Deliverable branch name:** `paperless-ngx_542221a38dff`
> **Working-tree HEAD (ground truth):** `542221a38dff06361e07976452f9aea24d210542`
> **Investigation type:** Read-only, runtime-first memory investigation. No source file was modified; the only repository change is this document.

---

## The question being answered (verbatim)

> "When the system imports documents, it sometimes consumes far more memory than expected for the metadata handling, but this doesn't happen consistently and seems to vary based on the document source or processing stage."
>
> "memory usage spikes significantly during document import operations, particularly when metadata is being processed, but the spike seems disproportionate to the actual document size. The odd part is that this memory doesn't always get released back to the system in a timely manner, even after processing is complete... these are mostly text documents with relatively small metadata."

The question decomposes into six objectives, each answered explicitly below and cross-checked in the coverage pass (§10):

- **O1 — Root cause:** which component(s)/code path(s) cause the spikes.
- **O2 — Unnecessary copies / retained references.**
- **O3 — Caching accumulation** (including the `caching.py` negative result).
- **O4 — Differential:** what differs between spiking and non-spiking cases.
- **O5 — Sensitivity:** how behavior changes with document type and batch size.
- **O6 — Runtime evidence + classification:** before/during/after measurements, the responsible methods, and a normal-GC-vs-problematic verdict.

---

## 1. Direct answer (root cause — O1)

The memory spike during "document import" is produced by the **document export/import manifest materialization** in Paperless-ngx's management commands, **not** by anything proportional to a document's physical file size. Two methods are responsible:

- **`document_exporter.dump()`** — `src/documents/management/commands/document_exporter.py:107`. It builds an in-memory `manifest` of **every** database object (all correspondents, tags, document types, **documents**, mail accounts/rules, saved views, groups, users) by repeatedly calling `json.loads(serializers.serialize("json", <Model>.objects.all()))` across `document_exporter.py:116-150`. For documents specifically it holds **three** large structures simultaneously: `documents = Document.objects.order_by("id")` (`document_exporter.py:127`, whose result cache is populated), `document_map = {d.pk: d for d in documents}` (`document_exporter.py:128`), and `document_manifest = json.loads(serializers.serialize("json", documents))` (`document_exporter.py:129`). All three remain live throughout the subsequent export loop (`document_exporter.py:153-161`, which indexes `document_map[document_dict["pk"]]` at `document_exporter.py:161`).

- **`document_importer.handle()`** — `src/documents/management/commands/document_importer.py:57`. It retains the entire manifest on the command instance via `self.manifest = json.load(f)` (`document_importer.py:73`), then calls `call_command("loaddata", manifest_path)` (`document_importer.py:87`), which makes Django **independently re-deserialize the same file** while `self.manifest` is still held — a concurrent double-materialization. A third filtered copy `manifest_documents = list(filter(...))` (`document_importer.py:137-139`) is then held alongside it.

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

The measurements in §3 confirm this quantitatively: at an **equal document count of 5,000 and equal ~19-byte physical files**, exporting text-heavy documents (50 KB of `content` each) peaked at **≈798 MB RSS**, versus **≈86 MB RSS** for metadata-only documents (~200 bytes of `content` each) — a **~9.3×** difference produced solely by `Document.content` (§5, O4).

The "not released promptly" symptom (§7, O6) is **normal CPython/glibc allocator arena retention amplified by scale**, observed directly: after `del` + `gc.collect()` the process released only single-digit MB and kept RSS at its high-water mark, then **reused** that memory on the next iteration. It is **not** an unbounded leak — the per-corpus peak is deterministic (stdev 0.48 MB across 6 identical runs, §3).

The user's "caching behavior that's accumulating data unexpectedly" (§8, O3) resolves to **two module-scoped accumulators**, not a dedicated caching layer: there is **no `src/documents/caching.py`** in this revision (shown in §8), so the candidates are the `dateparser` module-level locale cache invoked by `parse_date()` (`src/documents/parsers.py:212-230`) and the whole-corpus `CountVectorizer` vocabulary built by `classifier.train()` (`src/documents/classifier.py:194-199`).

---

## 2. Environment & method

### 2.1 Branch / commit (working tree)

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-d8807159-272a-4a73-a584-5ff7a6e332ea
$ git rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ git status --porcelain
(empty — clean working tree at start)
```

The deliverable file name (`paperless-ngx_542221a38dff.md`) is the **source branch name**; the checkout used for the investigation is the Blitzy working branch shown above, whose HEAD commit `542221a38dff06361e07976452f9aea24d210542` is the code state all `file:line` citations refer to.

### 2.2 Canonical runtime vs. the host interpreter

All measurements in this document were taken in the **canonical runtime**: the project Docker image

```text
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01
```

which ships **Python 3.9.23** with the exact pinned dependency stack:

```text
$ docker exec pngx python -c "import django,dateparser,sklearn; \
    print('django',django.get_version()); \
    print('dateparser',dateparser.__version__); print('sklearn',sklearn.__version__)"
django 4.0.4
dateparser 1.1.1
sklearn 1.0.2
```

This matches the pins in `Pipfile.lock` (django 4.0.4, django-q 1.3.9, scikit-learn 1.0.2, dateparser 1.1.1, numpy 1.22.3, scipy 1.8.0). Inside this image I exercised the **real `manage.py document_exporter` entry point** and the real `documents.models.Document` model, so the primary evidence is **CANONICAL**, not a synthetic stand-in.

**The host interpreter cannot run the pinned stack** (this is why the container is used, and why no "local NON-CANONICAL" numbers appear):

```text
$ python3 --version
Python 3.13.7
$ python3 -m pip install --quiet --target=/tmp/mem_probe/libs "django==4.0.4"   # succeeds
$ PYTHONPATH=/tmp/mem_probe/libs python3 -c "import django; print(django.get_version())"
4.0.4
$ PYTHONPATH=/tmp/mem_probe/libs python3 -c "import django; django.setup()"
...
  File "/tmp/mem_probe/libs/django/http/request.py", line 1, in <module>
    import cgi
ModuleNotFoundError: No module named 'cgi'
```

Django 4.0.4's `django/http/request.py` imports the `cgi` module, which **Python 3.13 removed** (PEP 594). `django.setup()` therefore fails on the host, so the pinned stack is only runnable in the canonical Python 3.9 container. Every measurement below is labeled accordingly:

- **CANONICAL** — the real `python manage.py document_exporter` command (real entry point, real code, real model, pinned stack).
- **CANONICAL-STACK / real-model, in-process** — small probe scripts that call the **exact** `dump()` / `handle()` APIs (`serializers.serialize`, `json.loads`, `{d.pk: d for d in documents}`, `json.load`, `loaddata`) against the **real `documents.models.Document`** on the pinned stack, invoked in-process so RSS can be sampled *before / during / after* and `tracemalloc` peak captured. The only thing the CLI does that these omit is copying physical files *after* the manifest is built — and the CANONICAL CLI runs (§3.1) confirm the same magnitude, so the in-process numbers are corroborated end-to-end.

### 2.3 Controlled corpus (stated plainly)

To vary document type and batch size deterministically, I populated a **throwaway** database (isolated SQLite under `/tmp/inst`, created with `manage.py migrate`) with real `Document` rows whose `content` field holds a text blob of a chosen size, and — for the full-CLI runs — a **tiny (~19-byte) physical original file** plus an 8-byte stub thumbnail per document. This is a controlled dataset, not a production corpus, but it is faithful for a *memory* question: a 50 KB `content` string occupies 50 KB of `TextField` storage and serializes to the same size whether the characters are real OCR text or repeated filler, so the allocation behavior is identical. The tiny physical files make the point directly — **files are ~19 bytes, yet peak RSS reaches ~798 MB**, precisely the reported disproportion.

### 2.4 Measurement methodology (before / during / after)

- **Process RSS** is read from field 2 (resident pages) of `/proc/self/statm` × page size (4096) → MB, sampled **before** the operation, **during** it (while references are still live), and **after** `del` + `gc.collect()`.
- **Peak allocation** of Python objects is captured with `tracemalloc.get_traced_memory()` (peak component).
- For the **real CLI**, whole-process peak RSS is captured with `/usr/bin/time -v` ("Maximum resident set size"). Its reliability in this container was validated against a known 300 MB allocation:

```text
$ /usr/bin/time -v python -c "a=bytearray(300*1024*1024); print(len(a))" 2>&1 | egrep "Maximum resident"
	Maximum resident set size (kbytes): 317440
```

(310 MB reported for a 300 MB allocation — trustworthy.)

### 2.5 Scope limitation (full OCR consume path)

The full end-to-end **OCR consume** path (actual ingestion of scanned PDFs) depends on native binaries pinned in `.build-config.json`:

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
```

The canonical image carries qpdf 10.1.0 and lacks the `jbig2` binary, so a *complete OCR ingest* is not reproduced here. This does **not** affect the investigation: the memory question concerns **import / metadata handling**, i.e. the export/import manifest path, which is reproduced canonically through its real entry point. The per-document consume path is addressed analytically from code (§4, §5) and contrasted with the corpus-proportional import/export path to explain the "varies by source/stage" observation.

---

## 3. Runtime memory measurements (complete, unedited output)

All commands were run inside the canonical container (started as `docker run -d --name pngx ... sleep infinity`, then provisioned with `apt-get install -y libzbar0 time` and `manage.py migrate`). The full probe-script sources are in the Appendix (§11); each exercises the exact Django APIs cited in §1/§4. Memory is in MB unless a raw `kbytes` value is quoted.

### 3.1 CANONICAL — the real `manage.py document_exporter` entry point (peak RSS)

Each scenario: populate a throwaway corpus (N docs, chosen `content` size, tiny ~19-byte physical files), then run the **real** management command under `/usr/bin/time -v`. Environment for every run:
`PAPERLESS_DATA_DIR=/tmp/inst/data PAPERLESS_MEDIA_ROOT=/tmp/inst/media PAPERLESS_CONSUMPTION_DIR=/tmp/inst/consume PAPERLESS_STATICDIR=/tmp/inst/static`.

**Document-type contrast at equal count (5,000 docs, equal ~19-byte files):**

```text
### meta5000 — 5000 docs, content=200 B
$ python canon_populate_files.py --n 5000 --content-bytes 200
populated 5000 docs; content_bytes=200; originals+thumbnails written
$ mkdir -p /tmp/exp_meta5000 && /usr/bin/time -v python manage.py document_exporter /tmp/exp_meta5000 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_meta5000 --no-progress-bar"
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:06.54
	Maximum resident set size (kbytes): 87936
$ ls -la /tmp/exp_meta5000/manifest.json
-rw-r--r-- 1 root root 4672063 Jul 13 16:58 /tmp/exp_meta5000/manifest.json

### text5000 — 5000 docs, content=50000 B
$ python canon_populate_files.py --n 5000 --content-bytes 50000
populated 5000 docs; content_bytes=50000; originals+thumbnails written
$ mkdir -p /tmp/exp_text5000 && /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000 --no-progress-bar
	Command being timed: "python manage.py document_exporter /tmp/exp_text5000 --no-progress-bar"
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:09.25
	Maximum resident set size (kbytes): 817104
$ ls -la /tmp/exp_text5000/manifest.json
-rw-r--r-- 1 root root 253672063 Jul 13 16:58 /tmp/exp_text5000/manifest.json
```

At an identical document count (5,000) and identical ~19-byte physical files, peak RSS is **817104 KB ≈ 798.0 MB** for text-heavy content vs **87936 KB ≈ 85.9 MB** for metadata-only content. The manifest sizes (253.67 MB vs 4.67 MB) show the payload is dominated by `Document.content`.

**Batch-size sweep (content fixed at 50 KB):**

```text
### text200 — 200 docs, content=50000 B
$ python canon_populate_files.py --n 200 --content-bytes 50000
populated 200 docs; content_bytes=50000; originals+thumbnails written
$ mkdir -p /tmp/exp_text200 && /usr/bin/time -v python manage.py document_exporter /tmp/exp_text200 --no-progress-bar
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.72
	Maximum resident set size (kbytes): 94996
manifest.json size: 9.7M

### text1000 — 1000 docs, content=50000 B
$ python canon_populate_files.py --n 1000 --content-bytes 50000
populated 1000 docs; content_bytes=50000; originals+thumbnails written
$ mkdir -p /tmp/exp_text1000 && /usr/bin/time -v python manage.py document_exporter /tmp/exp_text1000 --no-progress-bar
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:03.47
	Maximum resident set size (kbytes): 217048
manifest.json size: 49M

### text5000 (2nd run — magnitude stability) — 5000 docs, content=50000 B
$ python canon_populate_files.py --n 5000 --content-bytes 50000
populated 5000 docs; content_bytes=50000; originals+thumbnails written
$ mkdir -p /tmp/exp_text5000b && /usr/bin/time -v python manage.py document_exporter /tmp/exp_text5000b --no-progress-bar
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:10.36
	Maximum resident set size (kbytes): 823324
manifest.json size: 242M
```

Summary of the CANONICAL real-command peak RSS (files ~19 bytes in every case):

| Scenario | docs | `content`/doc | peak RSS (KB) | peak RSS (MB) | manifest.json |
|---|---|---|---|---|---|
| meta5000 | 5000 | 200 B | 87936 | 85.9 | 4.67 MB |
| text200 | 200 | 50 KB | 94996 | 92.8 | 9.7 MB |
| text1000 | 1000 | 50 KB | 217048 | 212.0 | 49 MB |
| text5000 (run 1) | 5000 | 50 KB | 817104 | 798.0 | 253 MB |
| text5000 (run 2) | 5000 | 50 KB | 823324 | 804.0 | 242 MB |

**Magnitude stability:** text5000 measured **798.0 MB** then **804.0 MB** on two independent runs (~0.7% apart) — the magnitude is stable. (The real-CLI peak includes a fixed ~86 MB baseline for importing Django + scikit-learn/numpy/scipy at startup; subtracting it isolates the content-driven growth, which is what scales.)

### 3.2 CANONICAL-STACK, real model — export manifest probe (before / during / after + `tracemalloc` peak)

This runs the exact `dump()` sequence (`document_exporter.py:127-130`) against the real `documents.models.Document`, sampling RSS before, during (references live), and after `del`+`gc.collect()`:

```text
$ python canon_manifest_probe.py --n 5000 --content-bytes 200   --label meta5000
{"label": "meta5000_r0", "n": 5000, "content_bytes": 200, "base_rss": 53.8, "live_rss": 86.5, "after_rss": 76.5, "retained_delta": 32.7, "tracemalloc_peak_mb": 19.1, "released_mb": 10.0}

$ python canon_manifest_probe.py --n 200  --content-bytes 50000 --label text200
{"label": "text200_r0", "n": 200, "content_bytes": 50000, "base_rss": 62.2, "live_rss": 86.9, "after_rss": 86.9, "retained_delta": 24.7, "tracemalloc_peak_mb": 30.8, "released_mb": 0.0}

$ python canon_manifest_probe.py --n 1000 --content-bytes 50000 --label text1000 --write-manifest /tmp/inst/manifest_text1000.json
{"label": "text1000_r0", "n": 1000, "content_bytes": 50000, "base_rss": 62.7, "live_rss": 162.6, "after_rss": 155.9, "retained_delta": 99.9, "tracemalloc_peak_mb": 148.4, "released_mb": 6.7}

$ python canon_manifest_probe.py --n 5000 --content-bytes 50000 --label text5000 --write-manifest /tmp/inst/manifest_text5000.json
{"label": "text5000_r0", "n": 5000, "content_bytes": 50000, "base_rss": 101.8, "live_rss": 566.3, "after_rss": 556.8, "retained_delta": 464.5, "tracemalloc_peak_mb": 731.5, "released_mb": 9.5}
```

| Scenario | base_rss | live_rss (during) | after_rss (del+gc) | retained Δ | tracemalloc peak | released |
|---|---|---|---|---|---|---|
| meta5000 (200 B) | 53.8 | 86.5 | 76.5 | +32.7 | 19.1 | 10.0 |
| text200 (50 KB) | 62.2 | 86.9 | 86.9 | +24.7 | 30.8 | 0.0 |
| text1000 (50 KB) | 62.7 | 162.6 | 155.9 | +99.9 | 148.4 | 6.7 |
| text5000 (50 KB) | 101.8 | 566.3 | 556.8 | +464.5 | 731.5 | 9.5 |

The in-process text5000 `tracemalloc` peak (**731.5 MB**) cross-validates the real-CLI text5000 peak (**798 MB**; the CLI additionally holds the ~86 MB import baseline). For **O2**, note `tracemalloc peak > retained Δ` in every text case (e.g. 731.5 vs 464.5 → ≈267 MB of **transient duplicate copies** — the serialized JSON string co-resident with the parsed Python list; 148.4 vs 99.9 at text1000).

### 3.3 CANONICAL-STACK — stability / inconsistency (same input × 6)

```text
$ python canon_manifest_probe.py --n 1000 --content-bytes 50000 --label stab1000 --repeat 6
{"label": "stab1000_r0", "n": 1000, "content_bytes": 50000, "base_rss": 294.6, "live_rss": 295.6, "after_rss": 295.6, "retained_delta": 1.0, "tracemalloc_peak_mb": 148.4, "released_mb": 0.0}
{"label": "stab1000_r1", "n": 1000, "content_bytes": 50000, "base_rss": 295.6, "live_rss": 295.8, "after_rss": 295.8, "retained_delta": 0.2, "tracemalloc_peak_mb": 147.1, "released_mb": 0.0}
{"label": "stab1000_r2", "n": 1000, "content_bytes": 50000, "base_rss": 295.8, "live_rss": 295.8, "after_rss": 295.8, "retained_delta": 0.0, "tracemalloc_peak_mb": 147.1, "released_mb": 0.0}
{"label": "stab1000_r3", "n": 1000, "content_bytes": 50000, "base_rss": 295.8, "live_rss": 295.8, "after_rss": 295.8, "retained_delta": 0.0, "tracemalloc_peak_mb": 147.2, "released_mb": 0.0}
{"label": "stab1000_r4", "n": 1000, "content_bytes": 50000, "base_rss": 295.8, "live_rss": 295.8, "after_rss": 295.8, "retained_delta": 0.0, "tracemalloc_peak_mb": 147.1, "released_mb": 0.0}
{"label": "stab1000_r5", "n": 1000, "content_bytes": 50000, "base_rss": 295.8, "live_rss": 295.8, "after_rss": 295.8, "retained_delta": 0.0, "tracemalloc_peak_mb": 147.1, "released_mb": 0.0}
{"summary_label": "stab1000", "repeat": 6, "peaks": [148.4, 147.1, 147.1, 147.2, 147.1, 147.1], "mean_peak": 147.33, "stdev_peak": 0.4784}
```

Running the **same unchanged input six times** yields a peak of `[148.4, 147.1, 147.1, 147.2, 147.1, 147.1]`, **mean 147.33 MB, stdev 0.48 MB** → the per-corpus allocation is **deterministic**. Two things this proves for **O6**:

1. The reported run-to-run *inconsistency* is **not** a nondeterministic algorithm — for a fixed corpus the peak is essentially constant.
2. Runs r1–r5 start at `base_rss ≈ 295 MB` (the high-water mark left by r0) with `retained_delta ≈ 0` and `released = 0` — a **direct demonstration of allocator arena retention and reuse**: the process kept the freed memory and reused it on the next iteration rather than returning it to the OS.

### 3.4 CANONICAL-STACK, real model — import probe (`json.load` retained + `loaddata` re-materialization)

Exercises `document_importer.handle()`: `self.manifest = json.load(f)` (`document_importer.py:73`), then `call_command("loaddata", ...)` (`document_importer.py:87`) while `self.manifest` is held, then the filtered copy `manifest_documents` (`document_importer.py:137-139`). (Logging is silenced so `loaddata`'s post-save signal handlers do not emit a per-row "File has gone" line for the synthetic rows — mirroring the real importer, which disables those signals across `document_importer.py:75-87`.)

```text
$ python canon_import_probe.py --manifest /tmp/inst/manifest_text1000.json
{"manifest": "manifest_text1000.json", "rows_loaded": 1000, "base_rss": 100.4, "after_json_load_rss": 102.1, "live_rss_during_loaddata": 112.1, "after_manifest_documents_filter_rss": 112.1, "after_release_rss": 111.1, "json_load_delta": 1.7, "loaddata_additional_delta": 10.0, "manifest_documents_extra_delta": 0.0, "tracemalloc_peak_mb": 148.4, "released_mb": 1.0}

$ python canon_import_probe.py --manifest /tmp/inst/manifest_text5000.json
{"manifest": "manifest_text5000.json", "rows_loaded": 5000, "base_rss": 100.4, "after_json_load_rss": 306.0, "live_rss_during_loaddata": 560.8, "after_manifest_documents_filter_rss": 560.8, "after_release_rss": 553.0, "json_load_delta": 205.6, "loaddata_additional_delta": 254.8, "manifest_documents_extra_delta": 0.0, "tracemalloc_peak_mb": 735.0, "released_mb": 7.9}
```

| Manifest | base | after `json.load` | during `loaddata` | after release | `json.load` Δ | `loaddata` extra Δ | tracemalloc peak | released |
|---|---|---|---|---|---|---|---|---|
| text1000 | 100.4 | 102.1 | 112.1 | 111.1 | +1.7 | +10.0 | 148.4 | 1.0 |
| text5000 | 100.4 | 306.0 | 560.8 | 553.0 | +205.6 | +254.8 | 735.0 | 7.9 |

The text5000 row is the clean demonstration of **O2 concurrent double-materialization**: `json.load` adds **+205.6 MB** retained on `self.manifest`, then `loaddata` independently deserializes the same file for **+254.8 MB more** while `self.manifest` is still alive — peak **735 MB**, of which only **7.9 MB** is released after both are cleared. (At text1000 the RSS deltas are muted — `json.load` shows only +1.7 MB — because the ~100 MB startup process already holds freed arenas that absorb the allocation; the `tracemalloc` peak of 148.4 MB shows the true transient allocation. That muting is itself arena reuse, O6.)

### 3.5 CANONICAL — `dateparser` module-scoped cache growth (O3)

Mirrors `parsers.py:221-230` (`import dateparser`; `dateparser.parse(ds, settings={...})`) with the pinned dateparser 1.1.1 and `TIMEZONE = settings.TIME_ZONE`:

```text
$ python canon_dateparser_probe.py
{"timezone_used": "UTC", "base_rss": 56.3, "after_first_parse_rss": 57.2, "after_2000_parses_rss": 119.9, "after_gc_rss": 111.9, "first_parse_cache_load_mb": 0.9, "growth_over_2000_more_parses_mb": 62.7, "released_after_gc_mb": 8.0, "tracemalloc_peak_mb": 28.3}
```

The first `parse()` lazily loads locale data (+0.9 MB); 2,000 further parses grow RSS by **+62.7 MB**, of which `gc.collect()` releases only **8.0 MB**. The `tracemalloc` peak is only **28.3 MB**, so **most of the 62.7 MB growth is in C-extension / locale structures outside Python's tracked heap**. Because this cache lives at **module scope**, it persists for the worker's lifetime (§8).

---

## 4. Per-component / per-method attribution (which methods hold the memory)

The export peak is held by three structures inside `document_exporter.dump()`, all live simultaneously for the whole export loop (`document_exporter.py:127-161`):

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
| queryset result cache of `documents` | `document_exporter.py:127` | every `Document` model instance (populated when `document_map` iterates it) | in-process peak scales with corpus (§3.2) |
| `document_map` | `document_exporter.py:128` | dict of **every** `Document` instance, `{pk: d}`, kept live for the loop at L161 | co-resident for whole loop (L161 indexes it) |
| `document_manifest` | `document_exporter.py:129` | full parsed Python list of all documents' serialized fields, **including `content`** | manifest 253 MB at text5000 (§3.1) |
| transient serialized JSON **string** | `document_exporter.py:129` (`serializers.serialize(...)` inside `json.loads`) | a full JSON string co-resident with the list `json.loads` builds from it | `tracemalloc peak > retained Δ` (§3.2, O2) |
| `manifest` (all models) | `document_exporter.py:116-150` | aggregate of documents **plus** all other models | grows across L116–L150 |

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
| `self.manifest` | `document_importer.py:73` | the entire manifest parsed from JSON, retained on the command **instance** | +205.6 MB at text5000 (§3.4) |
| `loaddata` internal deserialization | `document_importer.py:87` | Django's independent re-materialization of the **same file**, concurrent with `self.manifest` | +254.8 MB additional (§3.4) |
| `manifest_documents` | `document_importer.py:137-139` | a filtered `list(...)` copy held alongside `self.manifest` | extra retained copy |

The payload that makes all of the above disproportionate to file size is **`Document.content`** (`src/documents/models.py:117-124`), the full extracted text. The secondary accumulators are `parse_date()`'s `dateparser` module cache (`src/documents/parsers.py:212-230`) and `classifier.train()`'s `CountVectorizer` (`src/documents/classifier.py:194-199`) — see §8.

---

## 5. Spike vs. non-spike differential (O4)

**What differs between a case that spikes and one that does not is the aggregate `Document.content` materialized in one shot — not the physical file size and not the file count.** The controlled contrast holds *document count* and *physical file size* constant and varies only `content`:

| Case | docs | file size on disk | `content`/doc | real-CLI peak RSS (§3.1) | in-process tracemalloc peak (§3.2) |
|---|---|---|---|---|---|
| **non-spike** (metadata-only) | 5000 | ~19 B | ~200 B | **85.9 MB** | 19.1 MB |
| **spike** (text-heavy) | 5000 | ~19 B | ~50 KB | **798.0 MB** | 731.5 MB |

Same command, same entry point, same file count, same ~19-byte files → **~9.3×** more peak RSS (real CLI) / **~38×** more Python allocation (in-process, which excludes the fixed import baseline). The single variable is `content`. Concretely:

- A **spike** occurs when the corpus being exported/imported contains a lot of **aggregate text** (many documents and/or large per-document `content`). This is materialized entirely by `document_manifest`/`self.manifest` (`document_exporter.py:129`, `document_importer.py:73`).
- A **non-spike** occurs when the aggregate text is small (few documents, or small `content`), even if each physical file is large — because the exporter serializes the DB `content` field, not the file bytes.

This is why the user sees it "vary based on the document source or processing stage": the **corpus-proportional** export/import path (`document_exporter.dump()` / `document_importer.handle()`) materializes the whole corpus's text at once, whereas the **per-document consume** path (`src/documents/consumer.py:180` `try_consume_file` → parse at `consumer.py:261`, `parse_date` at `consumer.py:275`, `load_classifier` at `consumer.py:292`, `_store` at `consumer.py:301/379`) processes one document at a time and has a per-document footprint that does not scale with corpus size. Two different "stages" therefore have fundamentally different memory profiles for the same documents.

---

## 6. Document-type & batch-size sensitivity (O5)

**Batch size (document count), content fixed at 50 KB — real CLI peak RSS (§3.1):**

| docs | peak RSS (MB) | manifest.json |
|---|---|---|
| 200 | 92.8 | 9.7 MB |
| 1000 | 212.0 | 49 MB |
| 5000 | 798.0 / 804.0 | 253 MB / 242 MB |

Subtracting the fixed ~86 MB startup baseline isolates the content-driven growth: ≈7 MB → ≈126 MB → ≈712 MB. From 1,000 to 5,000 documents (5× the count) the content-driven peak grows ≈5.6× — **approximately linear in document count**, slightly super-linear because the co-resident duplicate copies (§4, O2) multiply the per-document cost. At 200 documents the transient copies still fit inside the freed arenas the process already holds, so net RSS growth is suppressed — an arena-reuse effect (O6), not a change in the algorithm.

**Document type (proxied by per-document `content` size), count fixed at 5,000 — real CLI peak RSS (§3.1):**

| type | `content`/doc | peak RSS (MB) |
|---|---|---|
| metadata-only ("small metadata") | ~200 B | 85.9 |
| text-heavy ("text documents") | ~50 KB | 798.0 |

The in-process manifest probe (§3.2) shows the same shape from the other direction — at fixed 50 KB content, `tracemalloc` peak rises 30.8 → 148.4 → 731.5 MB for 200 → 1000 → 5000 documents. **Net: peak ≈ (document count) × (per-document `content` size), i.e. proportional to the aggregate corpus text.** "Mostly text documents" is precisely the high-`content` regime that spikes; the physical file size never enters the calculation.

---

## 7. Normal-GC vs. problematic classification (O6)

**Verdict: the "memory doesn't get released back to the system in a timely manner" symptom is normal CPython/glibc allocator arena retention amplified by scale — it is not an unbounded or algorithmic leak.** The evidence:

1. **Freed but not returned to the OS.** In every export/import scenario, `released` after `del` + `gc.collect()` is single-digit MB while RSS stays at its high-water mark: export text5000 `released 9.5` MB with `after_rss 556.8` vs `live_rss 566.3` (§3.2); import text5000 `released 7.9` MB with `after_release_rss 553.0` (§3.4). The Python objects are freed (returned to CPython's `pymalloc`/`obmalloc` pools and glibc arenas) but the arenas are not handed back to the kernel, so RSS does not drop.

2. **Kept and reused (the decisive test).** In the 6-run stability test (§3.3), runs r1–r5 begin at `base_rss ≈ 295 MB` — the high-water mark left by r0 — with `retained_delta ≈ 0` and `released = 0`. The process **reuses** the retained arenas for the next corpus rather than growing further. That is the textbook signature of arena retention, and it is the opposite of a leak: a leak would grow monotonically across the six runs; here it is flat.

3. **Bounded and deterministic per corpus.** The peak is `[148.4, 147.1, 147.1, 147.2, 147.1, 147.1]` MB, **stdev 0.48 MB** (§3.3). A fixed corpus produces a fixed peak. So the reported "sometimes X, sometimes Y" is **not** nondeterministic code; it is driven by **which documents (and how much `content`) happen to be present** when the operation runs, combined with the arena high-water mark persisting between operations inside a long-lived worker.

**Why "a long-lived worker" matters (inferred mechanism, grounded in config):** Paperless runs asynchronous work under Django-Q, registered at `src/paperless/settings.py:110` (`"django_q"`) and configured at `src/paperless/settings.py:449-456`:

```python
# src/paperless/settings.py:449-456
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": PAPERLESS_WORKER_RETRY,
    "timeout": PAPERLESS_WORKER_TIMEOUT,
    "workers": TASK_WORKERS,
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
```

with `PAPERLESS_WORKER_TIMEOUT` defaulting to 1800 s (`settings.py:440`). A worker process that lives across many tasks will show its RSS pinned at the highest manifest it ever built, because (per items 1–2 above) freed arenas are retained. **Labeled inferred:** the attribution of the un-returned RSS specifically to CPython `pymalloc`/glibc arena behavior is a mechanism explanation; what is *observed* is `released ≈ 0` with RSS at high-water and subsequent reuse (§3.2–3.4), and the worker-lifetime context is grounded in `settings.py:449-456`. (`recycle: 1` means Django-Q recycles a worker after each task in the default config, which *mitigates* the persistence for async tasks; the export/import commands, however, run as standalone `manage.py` CLI processes, where the high-water mark simply lasts until the process exits.)

---

## 8. Caching evaluation (O3)

### 8.1 There is no dedicated caching layer (negative result)

The user's hypothesis of "caching behavior that's accumulating data unexpectedly" was tested first by looking for a caching module. **It does not exist in this revision:**

```text
$ ls src/documents/caching.py
ls: cannot access 'src/documents/caching.py': No such file or directory
$ find src -name caching.py
$        # (no output — no file named caching.py anywhere under src/)
```

So "caching" resolves to two **module-scoped accumulators** that outlive a single document, not a purpose-built cache.

### 8.2 `dateparser` module-scoped locale cache — grows and is largely not released

`parse_date()` imports and calls `dateparser` per regex date match (`src/documents/parsers.py:212-230`):

```python
# src/documents/parsers.py:221-230
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

`dateparser` lazily loads locale/language data and holds it at **module scope**, so it is process-global state, not per-call. The measured behavior (§3.5): +0.9 MB on first parse (lazy load), **+62.7 MB across 2,000 further parses**, of which `gc.collect()` reclaims only **8.0 MB**; the `tracemalloc` peak is only 28.3 MB, so most of the growth is in C-extension/locale structures outside Python's heap. In a long-lived worker (§7) this cache stays resident for the worker's lifetime. It is bounded (it does not grow without limit once locales are loaded), so it is **accumulation, not a leak** — but it does contribute to the "metadata processing" (date extraction is a metadata stage, `consumer.py:275`) footprint the user noticed.

### 8.3 `classifier` — whole-corpus vectorizer and unpickled model (analytical, grounded in code)

`classifier.train()` builds a **whole-corpus** vocabulary and sparse matrix (`src/documents/classifier.py:194-199`):

```python
# src/documents/classifier.py:194-199
        self.data_vectorizer = CountVectorizer(
            analyzer="word",
            ngram_range=(1, 2),
            min_df=0.01,
        )
        data_vectorized = self.data_vectorizer.fit_transform(data)
```

With `ngram_range=(1, 2)` the vocabulary includes unigrams **and** bigrams over the entire training corpus, and `fit_transform(data)` builds a sparse document-term matrix over all documents at once — so classifier *training* memory also scales with corpus size, independent of any single file. Separately, `load_classifier()` (`src/documents/classifier.py:30`) unpickles the trained model via six `pickle.load` calls (`classifier.py:78, 86, 87, 88, 90, 91, 92`) into memory; the loaded model is held for the duration of a consume run (`consumer.py:292`). These are consume-/train-path accumulators; they are noted here for completeness because they are the other place "data accumulates," but they are **not** the cause of the import/metadata spike, which is the manifest path (§1, §4).

---

## 9. Canonical vs. non-canonical labeling & limitations (explicit)

- **CANONICAL — real entry point.** §3.1 runs the actual `python manage.py document_exporter <dir> --no-progress-bar` command in the canonical Docker image (Python 3.9.23; Django 4.0.4, dateparser 1.1.1, scikit-learn 1.0.2 — the exact `Pipfile.lock` pins), against the real `documents.models.Document`. Peak RSS via `/usr/bin/time -v`, validated against a 300 MB control (§2.4).
- **CANONICAL-STACK, real model, in-process.** §3.2–3.5 run small probes that call the **exact** `dump()`/`handle()`/`parse_date` APIs against the real model on the pinned stack, invoked in-process only so RSS can be sampled before/during/after and `tracemalloc` peak captured. The one behavior they omit — copying physical files *after* the manifest is built — adds no material memory and is confirmed by the CANONICAL CLI runs, which reproduce the same magnitude (in-process 731.5 MB vs CLI 798 MB at text5000).
- **Controlled corpus.** The documents are throwaway rows with synthetic `content` and ~19-byte stub files (§2.3). This is appropriate for a memory question because allocation depends on `content` **size**, not its linguistic content, and it makes the disproportion explicit (19-byte files, ~798 MB peak).
- **Host interpreter is INFEASIBLE for the pinned stack (not used for any number).** Python 3.13.7 cannot `django.setup()` Django 4.0.4 (`ModuleNotFoundError: No module named 'cgi'`, PEP 594 removal — §2.2). No measurement in this document comes from the host; this is why the canonical container was used throughout.
- **Full OCR consume path not reproduced.** Native binaries pinned in `.build-config.json` (qpdf 10.6.3, jbig2enc 0.29) are not fully present in the image (qpdf 10.1.0; no `jbig2`), so a complete scanned-PDF ingest is out of scope (§2.5). It is not needed: the memory question is about import/metadata handling, which is the export/import manifest path exercised canonically here. Consume-path behavior is analyzed from code (§4, §5, §8.3) and clearly labeled as such.
- **Inferred vs. observed.** The only *inferred* claim is the attribution of un-returned RSS specifically to CPython `pymalloc`/glibc arena mechanics (§7); it is grounded in the observed `released ≈ 0` + reuse numbers and the worker-lifetime config at `settings.py:449-456`. All magnitudes are observed output.

---

## 10. Coverage pass

| Objective | Sub-question | Where answered | Primary evidence |
|---|---|---|---|
| **O1** | Root cause / responsible components | §1, §3.1, §4 | `document_exporter.dump()` `document_exporter.py:116-161`; `document_importer.handle()` `document_importer.py:73/87`; `Document.content` `models.py:117-124` |
| **O2** | Unnecessary copies / retained references | §3.2, §3.4, §4 | `tracemalloc peak > retained Δ` (export text5000 731.5 vs 464.5 → ≈267 MB transient); import `self.manifest` (+205.6) co-resident with `loaddata` (+254.8); filtered `manifest_documents` `document_importer.py:137-139` |
| **O3** | Caching accumulation | §8 | `caching.py` negative result (`ls`/`find`); `dateparser` module cache +62.7 MB, releases 8.0 MB (§3.5); `CountVectorizer` `classifier.py:194-199` |
| **O4** | Spike vs. non-spike | §5 | equal-count (5000) doc-type contrast: 798.0 MB vs 85.9 MB real CLI (~9.3×); 731.5 vs 19.1 MB in-process |
| **O5** | Document type & batch size | §6 | batch sweep 200/1000/5000 → 92.8/212.0/798.0 MB; content contrast 200 B vs 50 KB; peak ≈ count × content |
| **O6** | Runtime evidence + normal-GC-vs-problematic | §3.2–3.4, §7 | before/during/after tables; `released ≈ 0`; 6-run peak stdev 0.48 MB; arena retention **and reuse** (r1–r5 `retained_delta ≈ 0`) |

Every named item in the question is addressed: responsible components/paths (O1); "unnecessary copies or holding references longer than needed" (O2); "caching behavior that's accumulating data" including the `caching.py` non-existence (O3); "what's different between the cases where memory spikes and cases where it doesn't" (O4); "how does the behavior change with different document types or batch sizes" (O5); and "actual runtime memory measurements … which components or methods are responsible … whether this is normal Python garbage collection behavior or something problematic … where the memory is actually going" (O6, §3/§4/§7).

**Bottom line:** the spike is the corpus-proportional manifest materialization in `document_exporter.dump()` / `document_importer.handle()`, driven by aggregate `Document.content`, with transient duplicate copies at peak; the "not released" behavior is normal, bounded CPython/glibc arena retention (deterministic per corpus, reused across runs), not a leak; and "caching" is the module-scoped `dateparser`/`CountVectorizer` accumulation, with no dedicated `caching.py` in this revision.

---

## 11. Appendix — reproduction (provisioning + probe sources)

These are the exact temporary tools used. They were created outside the repository (under `/tmp`), copied into the canonical container, and removed on completion; **no repository file was modified**.

### 11.1 Provisioning the canonical container

```bash
IMG=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01
docker run -d --name pngx --entrypoint /bin/bash "$IMG" -c "sleep infinity"
docker exec pngx bash -lc "apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -y libzbar0 time"
docker exec pngx bash -lc 'export PAPERLESS_DATA_DIR=/tmp/inst/data PAPERLESS_MEDIA_ROOT=/tmp/inst/media \
    PAPERLESS_STATICDIR=/tmp/inst/static PAPERLESS_CONSUMPTION_DIR=/tmp/inst/consume; \
  mkdir -p /tmp/inst/{data,media,static,consume} \
    /tmp/inst/media/documents/{originals,archive,thumbnails} /tmp/inst/data/index /tmp/inst/data/log; \
  cd /app/src && python manage.py migrate --no-input'
# scripts copied in with: docker cp <host>/canon_*.py pngx:/tmp/scripts/
# probes run with:        docker exec pngx bash -lc 'cd /tmp/scripts && python <probe>.py ...'
# CLI runs run with:      docker exec pngx bash -lc 'cd /app/src && export PAPERLESS_*=... ; \
#                             /usr/bin/time -v python manage.py document_exporter <dir> --no-progress-bar'
```

`libzbar0` is required or the `documents` app fails to import (`pyzbar`); `time` provides `/usr/bin/time -v`; `migrate` creates the throwaway SQLite schema. `rss_mb()` in each probe reads field 2 (resident pages) of `/proc/self/statm` × page size.

### 11.2 `canon_manifest_probe.py` (export mechanism — `dump()` L127–130)

```python
"""CANONICAL export manifest probe: real documents.models.Document + real serializers,
exact document_exporter.dump() L127-130. RSS before/during/after + tracemalloc peak."""
import os, sys, json, gc, tracemalloc, argparse, statistics
os.environ.setdefault("PAPERLESS_DATA_DIR", "/tmp/inst/data")
os.environ.setdefault("PAPERLESS_MEDIA_ROOT", "/tmp/inst/media")
os.environ.setdefault("PAPERLESS_CONSUMPTION_DIR", "/tmp/inst/consume")
os.environ.setdefault("PAPERLESS_STATICDIR", "/tmp/inst/static")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
sys.path.insert(0, "/app/src")
import django; django.setup()
PAGE = os.sysconf("SC_PAGE_SIZE")
def rss_mb():
    with open("/proc/self/statm") as f:
        return int(f.read().split()[1]) * PAGE / (1024 * 1024)
def build_db(n, content_bytes):
    from documents.models import Document
    Document.objects.all().delete()
    blob = "x" * content_bytes
    objs = [Document(title=f"doc-{i}", content=blob, mime_type="text/plain",
                     checksum=f"{i:032d}", filename=f"doc-{i:07}.txt") for i in range(n)]
    Document.objects.bulk_create(objs, batch_size=500)
def measure_export(n, content_bytes, label, write_manifest=None):
    from django.core import serializers
    from documents.models import Document
    gc.collect(); base_rss = rss_mb(); tracemalloc.start()
    documents = Document.objects.order_by("id")                               # L127
    document_map = {d.pk: d for d in documents}                               # L128
    document_manifest = json.loads(serializers.serialize("json", documents))  # L129
    manifest = []; manifest += document_manifest                              # L130
    live_rss = rss_mb(); cur, peak = tracemalloc.get_traced_memory(); tracemalloc.stop()
    if write_manifest:
        with open(write_manifest, "w") as f: json.dump(manifest, f)
    del documents, document_map, document_manifest, manifest
    gc.collect(); after_rss = rss_mb()
    return {"label": label, "n": n, "content_bytes": content_bytes,
            "base_rss": round(base_rss, 1), "live_rss": round(live_rss, 1),
            "after_rss": round(after_rss, 1), "retained_delta": round(live_rss - base_rss, 1),
            "tracemalloc_peak_mb": round(peak / (1024 * 1024), 1),
            "released_mb": round(live_rss - after_rss, 1)}
def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--n", type=int, default=1000); ap.add_argument("--content-bytes", type=int, default=50000)
    ap.add_argument("--label", default="run"); ap.add_argument("--repeat", type=int, default=1)
    ap.add_argument("--write-manifest", default=None); args = ap.parse_args()
    build_db(args.n, args.content_bytes); peaks = []
    for r in range(args.repeat):
        wm = args.write_manifest if r == 0 else None
        res = measure_export(args.n, args.content_bytes, f"{args.label}_r{r}", wm)
        peaks.append(res["tracemalloc_peak_mb"]); print(json.dumps(res))
    if args.repeat > 1:
        print(json.dumps({"summary_label": args.label, "repeat": args.repeat, "peaks": peaks,
                          "mean_peak": round(statistics.mean(peaks), 2),
                          "stdev_peak": round(statistics.pstdev(peaks), 4)}))
if __name__ == "__main__": main()
```

### 11.3 `canon_import_probe.py` (import mechanism — `handle()` L73/L87/L137)

```python
"""CANONICAL import manifest probe: real Document model; self.manifest = json.load (L73) held
concurrently with call_command("loaddata") (L87), plus manifest_documents filter (L137-139)."""
import os, sys, json, gc, tracemalloc, argparse, logging
os.environ.setdefault("PAPERLESS_DATA_DIR", "/tmp/inst/data")
os.environ.setdefault("PAPERLESS_MEDIA_ROOT", "/tmp/inst/media")
os.environ.setdefault("PAPERLESS_CONSUMPTION_DIR", "/tmp/inst/consume")
os.environ.setdefault("PAPERLESS_STATICDIR", "/tmp/inst/static")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
sys.path.insert(0, "/app/src")
import django; django.setup()
logging.disable(logging.CRITICAL)   # silence per-row post_save signal log flood (importer disables signals L75-87)
PAGE = os.sysconf("SC_PAGE_SIZE")
def rss_mb():
    with open("/proc/self/statm") as f: return int(f.read().split()[1]) * PAGE / (1024 * 1024)
class ImporterLike:
    def __init__(self): self.manifest = None
def main():
    ap = argparse.ArgumentParser(); ap.add_argument("--manifest", required=True); args = ap.parse_args()
    from django.core.management import call_command
    from documents.models import Document
    Document.objects.all().delete()
    inst = ImporterLike(); gc.collect(); base_rss = rss_mb(); tracemalloc.start()
    with open(args.manifest) as f:
        inst.manifest = json.load(f)                        # L73 retained on instance
    after_load_rss = rss_mb()
    call_command("loaddata", args.manifest, verbosity=0)    # L87 independent re-deserialization
    live_rss = rss_mb()
    manifest_documents = list(filter(lambda r: r["model"] == "documents.document", inst.manifest))  # L137-139
    after_filter_rss = rss_mb(); cur, peak = tracemalloc.get_traced_memory(); tracemalloc.stop()
    inst.manifest = None; del manifest_documents; gc.collect(); after_rss = rss_mb()
    print(json.dumps({"manifest": os.path.basename(args.manifest), "rows_loaded": Document.objects.count(),
        "base_rss": round(base_rss, 1), "after_json_load_rss": round(after_load_rss, 1),
        "live_rss_during_loaddata": round(live_rss, 1),
        "after_manifest_documents_filter_rss": round(after_filter_rss, 1),
        "after_release_rss": round(after_rss, 1), "json_load_delta": round(after_load_rss - base_rss, 1),
        "loaddata_additional_delta": round(live_rss - after_load_rss, 1),
        "manifest_documents_extra_delta": round(after_filter_rss - live_rss, 1),
        "tracemalloc_peak_mb": round(peak / (1024 * 1024), 1),
        "released_mb": round(after_filter_rss - after_rss, 1)}))
if __name__ == "__main__": main()
```

### 11.4 `canon_dateparser_probe.py` (O3 — `parse_date()` L221–230) and `canon_populate_files.py`

```python
# canon_dateparser_probe.py  — pinned dateparser 1.1.1, mirrors parsers.py:221-230
import os, sys, gc, tracemalloc, json
os.environ.setdefault("PAPERLESS_DATA_DIR", "/tmp/inst/data"); os.environ.setdefault("PAPERLESS_MEDIA_ROOT", "/tmp/inst/media")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings"); sys.path.insert(0, "/app/src")
import django; django.setup()
from django.conf import settings as dj_settings
PAGE = os.sysconf("SC_PAGE_SIZE")
def rss_mb():
    with open("/proc/self/statm") as f: return int(f.read().split()[1]) * PAGE / (1024 * 1024)
import dateparser
SET = {"DATE_ORDER": "DMY", "PREFER_DAY_OF_MONTH": "first",
       "RETURN_AS_TIMEZONE_AWARE": True, "TIMEZONE": dj_settings.TIME_ZONE}
samples = ["2021-01-01", "01/02/2020", "March 5, 2019", "15.06.2018",
           "2017/11/23", "1 Jan 2016", "12-12-2015", "2014-08-09"]
gc.collect(); base = rss_mb(); tracemalloc.start()
_ = dateparser.parse(samples[0], settings=SET); after_first = rss_mb()
for i in range(2000): dateparser.parse(samples[i % len(samples)], settings=SET)
after_many = rss_mb(); cur, peak = tracemalloc.get_traced_memory(); tracemalloc.stop()
gc.collect(); after_gc = rss_mb()
print(json.dumps({"timezone_used": dj_settings.TIME_ZONE, "base_rss": round(base, 1),
    "after_first_parse_rss": round(after_first, 1), "after_2000_parses_rss": round(after_many, 1),
    "after_gc_rss": round(after_gc, 1), "first_parse_cache_load_mb": round(after_first - base, 1),
    "growth_over_2000_more_parses_mb": round(after_many - after_first, 1),
    "released_after_gc_mb": round(after_many - after_gc, 1),
    "tracemalloc_peak_mb": round(peak / (1024 * 1024), 1)}))
```

```python
# canon_populate_files.py — N real Document rows (large content) + tiny ~19-byte files, so the REAL CLI runs
import os, sys, argparse
os.environ.setdefault("PAPERLESS_DATA_DIR", "/tmp/inst/data"); os.environ.setdefault("PAPERLESS_MEDIA_ROOT", "/tmp/inst/media")
os.environ.setdefault("PAPERLESS_CONSUMPTION_DIR", "/tmp/inst/consume"); os.environ.setdefault("PAPERLESS_STATICDIR", "/tmp/inst/static")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings"); sys.path.insert(0, "/app/src")
import django; django.setup()
from django.conf import settings
def main():
    ap = argparse.ArgumentParser(); ap.add_argument("--n", type=int, required=True)
    ap.add_argument("--content-bytes", type=int, required=True); args = ap.parse_args()
    from documents.models import Document
    Document.objects.all().delete()
    os.makedirs(settings.ORIGINALS_DIR, exist_ok=True); os.makedirs(settings.THUMBNAIL_DIR, exist_ok=True)
    blob = "x" * args.content_bytes
    objs = [Document(title=f"doc-{i}", content=blob, mime_type="text/plain",
                     checksum=f"{i:032d}", filename=f"doc-{i:07}.txt") for i in range(args.n)]
    Document.objects.bulk_create(objs, batch_size=500)
    for d in Document.objects.all().only("id", "filename"):
        with open(os.path.join(settings.ORIGINALS_DIR, d.filename), "w") as f: f.write("tiny original file\n")
        with open(os.path.join(settings.THUMBNAIL_DIR, f"{d.pk:07}.png"), "wb") as f: f.write(b"\x89PNG\r\n\x1a\n")
    print(f"populated {Document.objects.count()} docs; content_bytes={args.content_bytes}; originals+thumbnails written")
if __name__ == "__main__": main()
```

*End of investigation.*
