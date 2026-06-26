# Paperless-ngx OCR Document-Processing Pipeline — Runtime Behavior Report

**Repository under investigation:** `paperless-ngx`
**Branch:** `paperless-ngx_542221a38dff`
**Commit (HEAD):** `542221a38dff06361e07976452f9aea24d210542`
**Scope:** Trace a multi-page PDF from upload → asynchronous OCR processing → on-disk media storage → database persistence, answering four discrete questions with **code-as-truth** evidence and **live runtime observation**.

> **Why this report exists.** A user reported *"inconsistent OCR results"* with multi-page PDFs and asked four precise questions about how a document actually flows through the system. This document answers each question with (a) a direct **Answer**, (b) **Evidence** (exact `file:line` citations and short code excerpts), (c) the **Rationale** connecting code to behavior, and (d) the **observed runtime output** captured by genuinely building and running the stack. Per the governing method directive, **the source code at this commit is authoritative**; where public/community examples (drawn from newer paperless-ngx releases) disagree, this commit's code prevails.

---

## 1. Methodology & Environment Bring-Up

### 1.1 How the stack was built and run

The investigation used the user-provided runtime image (Python **3.9** with Tesseract, Ghostscript, qpdf, and Redis available). The optional **jbig2enc** encoder was **not** present on `PATH` in the observed container (`command -v jbig2` → not found); OCRmyPDF treats jbig2 as optional, so it simply skipped jbig2 lossless image compression and **degraded gracefully** — this is non-blocking for OCR and does not affect any answer below. The dev stack was brought up *normally* — Redis broker, database migrations, an admin superuser, the Django-Q worker, and the ASGI web server:

```bash
# Redis broker/channel layer (hard prerequisite for async processing)
redis-server --daemonize yes            # redis-cli ping -> PONG

# From src/: apply migrations and ensure the admin user
python3 manage.py migrate --no-input
python3 manage.py manage_superuser      # PAPERLESS_ADMIN_USER/PASSWORD -> admin/admin

# The Django-Q worker cluster that EXECUTES consume_file (emits the Q2 logs)
python3 manage.py qcluster              # logs -> data/log/qcluster.out + data/log/paperless.log

# The ASGI web server that ACCEPTS the upload and returns the Q1 response
gunicorn -c gunicorn.conf.py paperless.asgi:application   # :8000
```

Verified live at startup: `redis-cli ping` → `PONG`; the Django-Q cluster reported **11 worker processes** ready; `GET /api/` returned **HTTP 200** with `admin:admin`; and the database began **pristine** (`Document.objects.count()` → `0`), so the first processed document receives **`pk = 1`**.

The admin user is created by `src/documents/management/commands/manage_superuser.py`, which reads `PAPERLESS_ADMIN_USER` / `PAPERLESS_ADMIN_PASSWORD` / `PAPERLESS_ADMIN_MAIL` and calls `User.objects.create_superuser(...)` (`src/documents/management/commands/manage_superuser.py:L22–L38`). Authentication is mandatory because the upload endpoint enforces `IsAuthenticated` (see §3).

### 1.2 Asynchronous architecture awareness (the key framing)

Paperless-ngx **decouples upload acceptance from document processing**:

- **Acceptance** is handled *synchronously* by the ASGI web process (gunicorn). It validates the upload, writes it to a temp file, **enqueues** a task, and returns immediately.
- **Processing** (OCR, thumbnailing, file moves, DB write) runs *asynchronously* in a **Django-Q `qcluster` worker**, reached via the **Redis** broker.

Two consequences drive every answer below: **(1)** Q1's HTTP response is produced *before* OCR runs, and **(2)** Q2's processing logs originate in the **worker** process — not the web process. Redis is therefore a hard prerequisite: it carries the task from web to worker.

### 1.3 Test-fixture rationale — why an *image-only* multi-page PDF

A document that genuinely "requires OCR" must be an **image-only (rasterised) multi-page PDF**. Under the default OCR mode `skip`, OCRmyPDF is invoked with `skip_text=True`, which skips OCR only on pages that **already contain a text layer**. A born-digital text PDF would have OCR skipped per page and would not exercise the OCR path the user is asking about. An **image-only** PDF (no embedded text) guarantees OCRmyPDF performs OCR on every page.

This is corroborated by `parse()` in the tesseract parser (`src/paperless_tesseract/parsers.py` L230–L244): for `application/pdf` it computes `original_has_text`, but **only** the `OCR_MODE='skip_noarchive'` branch short-circuits when text exists — plain `skip` (the default) still invokes OCRmyPDF.

For this run, a **3-page image-only PDF** was generated entirely outside the source tree (under `/tmp`) using Pillow. It was verified to contain **zero embedded text** (extracted-text length = 0) before upload, so OCR was forced on all three pages. (This transient file was removed during cleanup — see §8.)

### 1.4 The standard upload interface

The "standard interface" is the documents upload endpoint **`POST /api/documents/post_document/`** — the same endpoint the Angular SPA drag-and-drop uploader uses. It is registered in `src/paperless/urls.py` L56–L60.

### 1.5 Live vs. code-derived evidence

Every answer below was both **grounded in the source** (Phase 1) and **observed live** (Phase 2). Where a detail could not be surfaced live (specifically, OCRmyPDF's *own* internal log lines — see §4), this is stated explicitly and **no output is fabricated**.

---

## 2. Process Topology

The production image runs three long-lived services under Supervisor (`docker/supervisord.conf`); Redis is the broker/channel layer. Which process answers which question matters:

| Service (`supervisord.conf`) | Command | Role | Relevant to |
|---|---|---|---|
| `[program:gunicorn]` | `gunicorn -c gunicorn.conf.py paperless.asgi:application` | **ASGI web server** — accepts the upload, returns `Response("OK")` | **Q1** |
| `[program:scheduler]` | `python3 manage.py qcluster` | **Django-Q worker cluster** — executes `consume_file`, emits the processing logs | **Q2, Q3, Q4** |
| `[program:consumer]` | `python3 manage.py document_consumer` | **Directory watcher** (consume folder) — a *separate* ingestion path (logger `paperless.management.consumer`), **out of scope** | (disambiguation only) |
| Redis | `redis://localhost:6379` | Broker that carries the task web→worker; backs the Channels `status_updates` progress broadcasts | prerequisite |

> The web upload does **not** go through `document_consumer`. The directory watcher is mentioned only to make clear which logs are relevant: the Q2 stage logs come from the **`qcluster`** worker executing `documents.tasks.consume_file`.

---

## 3. Q1 — Immediate HTTP Upload Response

> *"What HTTP response status and message do you receive immediately after upload submission?"*

### 3.1 Answer

**HTTP `200 OK`** with the JSON body **`"OK"`** (`Content-Type: application/json`). The response is **synchronous and returns *before* any OCR processing begins** — acceptance is decoupled from processing.

### 3.2 Evidence

**Route** — `src/paperless/urls.py` L56–L60:

```python
re_path(
    r"^documents/post_document/",
    PostDocumentView.as_view(),
    name="post_document",
),
```

**View** — `src/documents/views.py` L491–L535. The class declares authentication, serializer, and parser, then `post()` validates, writes a temp file, **enqueues** the task, and returns:

```python
class PostDocumentView(GenericAPIView):

    permission_classes = (IsAuthenticated,)          # L493
    serializer_class = PostDocumentSerializer        # L494
    parser_classes = (parsers.MultiPartParser,)      # L495

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)    # L500
        ...
        with tempfile.NamedTemporaryFile(            # L512–L519
            prefix="paperless-upload-",
            dir=settings.SCRATCH_DIR,
            delete=False,
        ) as f:
            f.write(doc_data)
            ...
        async_task(                                  # L523–L533  (enqueue only)
            "documents.tasks.consume_file",
            temp_filename,
            ...
            task_id=task_id,
        )
        return Response("OK")                        # L535
```

### 3.3 Rationale

- **Why `200`:** the DRF `Response` at L535 is constructed with **no explicit `status=` argument**, so DRF applies its default of `status.HTTP_200_OK`. The body is the serialized string `"OK"`, rendered as the JSON value `"OK"`.
- **Why "returns before processing":** `async_task(...)` (L523) only *enqueues* the job onto the Django-Q/Redis broker; `post()` returns on the very next statement (L535). The OCR work happens **later**, in the `qcluster` worker. This is the core synchronous-vs-asynchronous decoupling.

### 3.4 Observed runtime output (live)

```text
$ curl -sS -u admin:admin -F "document=@/tmp/test_multipage_image_only.pdf" \
       http://localhost:8000/api/documents/post_document/ -D -

HTTP/1.1 200 OK
content-type: application/json
allow: POST, OPTIONS
x-content-type-options: nosniff

"OK"
```

The call returned in **~0.10 s**, whereas the asynchronous consume pipeline for the same document took **~8 s** (timestamps `21:39:44` → `21:39:52` in §4.4) — a direct, empirical demonstration that the `200 "OK"` is returned **before** processing.

A further confirmation: uploading the **identical** file a second time *also* returned **`200 "OK"`**, yet the async worker subsequently **rejected it as a duplicate** (Django-Q task `success=False`, result `"...Not consuming ...: It is a duplicate."`; log line `[ERROR] [paperless.consumer] Not consuming ...: It is a duplicate.`). This proves the `200` signals **acceptance only**, never processing success.

### 3.5 Edge cases (code-derived, corroborated by in-repo tests)

- **Unsupported MIME type → HTTP `400`** with message `"File type <mime> not supported"`. Source: `src/documents/serialisers.py` `PostDocumentSerializer.validate_document` L450–L457:

  ```python
  mime_type = magic.from_buffer(document_data, mime=True)
  if not is_mime_type_supported(mime_type):
      raise serializers.ValidationError(
          _("File type %(type)s not supported") % {"type": mime_type},
      )
  ```

- **Missing `document` form field → HTTP `400`** (serializer validation failure on the required `FileField`).
- **Unauthenticated request → rejected** by `IsAuthenticated` (`403`/`401` depending on the auth scheme).

**In-repo runtime corroboration** (`src/documents/tests/test_api.py`): `test_upload` (L762) asserts `status_code == 200` for a valid PDF upload to `/api/documents/post_document/` **with `documents.views.async_task` mocked** — proving the response does not wait for (or even perform) processing; `test_upload_invalid_form` (L808) asserts `400` for a missing `document` field; `test_upload_invalid_file` (L822) asserts `400` for an unsupported type (`simple.zip`).

---

## 4. Q2 — Stage Log Patterns + OCRmyPDF Invocation Parameters

> *"As processing happens, what key log patterns appear that show the document moving through different stages, including the OCRmyPDF invocation parameters?"*

### 4.1 Answer

As the **`qcluster` worker** runs `documents.tasks.consume_file`, it emits an ordered sequence of `paperless.*` log lines marking each stage — `Consuming …` → `Detected mime type …` → `Parser: …` → `Parsing …` → **`Calling OCRmyPDF with args: {…}`** → `Generating thumbnail …` → `Saving record to database` → `Document … consumption finished`. The OCRmyPDF invocation is a single keyword-argument dictionary; under the **default configuration** (PDF input, `OCR_MODE=skip`) it is:

```python
{
  'input_file':  '<SCRATCH_DIR>/paperless-upload-XXXXXXXX',
  'output_file':  '<parser tempdir>/archive.pdf',
  'use_threads':  True,
  'jobs':         11,                # = THREADS_PER_WORKER (host-dependent; see §4.5)
  'language':     'eng',
  'output_type':  'pdfa',
  'progress_bar': False,
  'skip_text':    True,              # OCR_MODE == 'skip'
  'clean':        True,              # OCR_CLEAN == 'clean'
  'deskew':       True,              # OCR_DESKEW and mode != 'redo'
  'rotate_pages': True,              # OCR_ROTATE_PAGES
  'rotate_pages_threshold': 12.0,    # OCR_ROTATE_PAGES_THRESHOLD
  'sidecar':      '<parser tempdir>/sidecar.txt',   # because OCR_PAGES == 0
}
```

### 4.2 Where the logs come from

- The process is the **Django-Q `qcluster` worker** (`docker/supervisord.conf` `[program:scheduler]`), executing `documents.tasks.consume_file`.
- The log **format** is the *verbose* formatter `"[{asctime}] [{levelname}] [{name}] {message}"` (`src/paperless/settings.py` L377–L379).
- Logger **names** are dotted `paperless.*`. `src/documents/loggers.py` `LoggingMixin.log` (L14–L21) resolves `self.logging_name` to a logger and attaches a per-file group UUID via `extra={"group": self.logging_group}`:

  ```python
  def log(self, level, message, **kwargs):
      if self.logging_name:
          logger = logging.getLogger(self.logging_name)
      ...
      getattr(logger, level)(message, extra={"group": self.logging_group}, **kwargs)
  ```

  The consumer logs under **`paperless.consumer`** (`Consumer.logging_name`, `src/documents/consumer.py:L54`); the tesseract parser logs under **`paperless.parsing.tesseract`** (`RasterisedDocumentParser.logging_name`, `src/paperless_tesseract/parsers.py:L24`). The `paperless` logger writes to the `file_paperless` handler → `data/log/paperless.log` (settings L409).

### 4.3 Task entry/exit

`src/documents/tasks.py` `consume_file` (L184) optionally splits on barcodes **only when `CONSUMER_ENABLE_BARCODES`** (default off — verified `False` live), then delegates to `Consumer().try_consume_file(...)` (L236) and on success returns the Django-Q task result string (L247):

```python
if document:
    return "Success. New document id {} created".format(document.pk)
```

### 4.4 Ordered stage messages (runtime order)

Note this is **runtime** order, not source-line order — `_store` is *defined* at `consumer.py` L379 but *invoked* at L301. The stage `self.log(...)` calls, in execution order:

| # | Logger | Message | Source |
|---|---|---|---|
| 1 | `paperless.consumer` | `Consuming <filename>` | `consumer.py` L215 (`info`) |
| 2 | `paperless.consumer` | `Detected mime type: application/pdf` | `consumer.py` L221 (`debug`) |
| 3 | `paperless.consumer` | `Parser: RasterisedDocumentParser` | `consumer.py` L246 (`debug`) |
| 4 | `paperless.consumer` | `Parsing <filename>...` | `consumer.py` L260 (`debug`) |
| 5 | `paperless.parsing.tesseract` | `Calling OCRmyPDF with args: {…}` | `src/paperless_tesseract/parsers.py` L260 (`debug`), immediately before `ocrmypdf.ocr(**args)` at L261 |
| 6 | `paperless.consumer` | `Generating thumbnail for <filename>...` | `consumer.py` L263 (`debug`) |
| 7 | `paperless.consumer` | `Saving record to database` | `consumer.py` L387 (`debug`, inside `_store`) |
| 8 | `paperless.consumer` | `Document <str(document)> consumption finished` | `consumer.py` L373 (`info`) |

> **Subtlety:** `document_consumption_finished.send(...)` fires at `consumer.py` L306 — *immediately after* `_store` (L301) and *before* the file-move block (L316–L346). So the post-consume signal handlers (tags / correspondent / type / search-index) run **before** the files are moved into place and before the "consumption finished" line.

**Observed runtime output (live, `data/log/paperless.log`)** — the actual sequence for the test document, which contains *more* real lines than the predicted core set above (all genuinely emitted; none fabricated):

```text
[2026-06-26 21:39:44,840] [INFO]  [paperless.consumer] Consuming test_multipage_image_only.pdf
[2026-06-26 21:39:44,841] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-06-26 21:39:44,841] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-06-26 21:39:44,844] [DEBUG] [paperless.consumer] Parsing test_multipage_image_only.pdf...
[2026-06-26 21:39:44,866] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-im8vf708
[2026-06-26 21:39:44,935] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-im8vf708', 'output_file': '/tmp/paperless/paperless-slo533h9/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-slo533h9/sidecar.txt'}
[2026-06-26 21:39:47,909] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-06-26 21:39:47,910] [DEBUG] [paperless.consumer] Generating thumbnail for test_multipage_image_only.pdf...
[2026-06-26 21:39:47,913] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-slo533h9/archive.pdf[0] /tmp/paperless/paperless-slo533h9/convert.png
[2026-06-26 21:39:48,626] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-slo533h9/convert.png -out /tmp/paperless/paperless-slo533h9/thumb_optipng.png
[2026-06-26 21:39:52,613] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-06-26 21:39:52,616] [DEBUG] [paperless.consumer] Saving record to database
[2026-06-26 21:39:52,636] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-im8vf708
[2026-06-26 21:39:52,660] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-slo533h9
[2026-06-26 21:39:52,661] [INFO]  [paperless.consumer] Document 2026-06-26 test_multipage_image_only consumption finished
```

The Django-Q worker stdout (`data/log/qcluster.out`) framed the same task:

```text
21:39:44 [Q] INFO Process-1:7 processing [test_multipage_image_only.pdf]
21:39:52 [Q] INFO Processed [test_multipage_image_only.pdf]
21:39:52 [Q] INFO recycled worker Process-1:7
```

and the stored Django-Q task result was exactly **`Success. New document id 1 created`** (`success=True`, `func=documents.tasks.consume_file`) — matching `tasks.py` L247.

**Explanatory notes on the additional real lines** (so the report is complete and honest):
- `[paperless.parsing.tesseract] Extracted text from PDF file …` is emitted by `extract_text` (`src/paperless_tesseract/parsers.py:L122`), which `parse()` calls at `src/paperless_tesseract/parsers.py:L235` to compute `original_has_text` *before* invoking OCRmyPDF.
- `[paperless.parsing.tesseract] Using text from sidecar file` is `extract_text` reading the OCR sidecar produced by OCRmyPDF.
- `[paperless.parsing] Execute: convert … archive.pdf[0] … convert.png` is the PDF→PNG thumbnail render (`make_thumbnail_from_pdf`, `src/documents/parsers.py:L187`), and `[paperless.parsing.tesseract] Execute: optipng …` is the optional thumbnail optimisation (`get_optimised_thumbnail`, `src/documents/parsers.py:L319–L340`).
- `[paperless.classifier] … not performing automatic matching.` is `load_classifier()` (consumer.py L292) reporting there is no trained model yet.
- `Deleting file …` (consumer.py L349) and `Deleting directory …` (parser cleanup) remove the temp original and the parser tempdir.

> **Honest caveat on OCRmyPDF's *own* logs.** With `progress_bar=False` and default verbosity, the **ocrmypdf** library (logger `ocrmypdf.*`) ran **quietly** in this configuration: its internal lines (e.g. a `Start processing N pages concurrently` style message) **did not appear** in either `paperless.log` or `qcluster.out` during the live run. They are therefore **not reproduced here** — the authoritative paperless stage logs are exactly as shown above.

### 4.5 The OCRmyPDF args, line-by-line

The dict is assembled by `construct_ocrmypdf_parameters` (`src/paperless_tesseract/parsers.py` L135–L228) and logged at L260. The **insertion order** (which Python `dict` preserves and which is exactly how it is logged) and the controlling settings:

| Key | Value (default) | Source line | Controlling setting (default) |
|---|---|---|---|
| `input_file` | temp upload under `SCRATCH_DIR` | L144 | the `NamedTemporaryFile` from the view |
| `output_file` | `<tempdir>/archive.pdf` | L145 / L249 | `tempfile.mkdtemp(prefix="paperless-", dir=SCRATCH_DIR)` (`src/documents/parsers.py:L293`) |
| `use_threads` | `True` | L148 | required under django-q daemonized processes |
| `jobs` | `11` (observed) | L149 | `THREADS_PER_WORKER` (settings L469–L472) |
| `language` | `'eng'` | L150 | `OCR_LANGUAGE` (settings L514) |
| `output_type` | `'pdfa'` | L151 | `OCR_OUTPUT_TYPE` (settings L518) |
| `progress_bar` | `False` | L152 | — |
| `skip_text` | `True` | L157–L158 | `OCR_MODE == 'skip'` (settings L522) |
| `clean` | `True` | L164–L165 | `OCR_CLEAN == 'clean'` (settings L526) |
| `deskew` | `True` | L172–L173 | `OCR_DESKEW` true and mode ≠ `redo` (settings L528) |
| `rotate_pages` | `True` | L175–L176 | `OCR_ROTATE_PAGES` (settings L530) |
| `rotate_pages_threshold` | `12.0` | L177–L179 | `OCR_ROTATE_PAGES_THRESHOLD` (settings L532) |
| `sidecar` | `<tempdir>/sidecar.txt` | L181–L185 | `OCR_PAGES == 0` (settings L510) |

**The `jobs` value is environment-dependent.** It is `THREADS_PER_WORKER = max(floor(cpu_count / TASK_WORKERS), 1)` (settings L460–L472), where `TASK_WORKERS = floor(sqrt(cpu_count))` for ≥4 cores (settings L427–L435). On this 128-core host: `TASK_WORKERS = floor(sqrt(128)) = 11` and `jobs = floor(128 / 11) = 11` — confirmed live (`jobs: 11`). On a 4-core laptop it would be `2`; on an 8-core machine, `jobs = floor(8 / floor(sqrt(8))) = floor(8 / 2) = **4**` (whereas `TASK_WORKERS` is `2`). Report it as the **formula**, not a fixed integer.

**Mode-dependent variations** (all in `construct_ocrmypdf_parameters`), for configurations other than the default:
- `OCR_MODE='force'` (or the safe-fallback retry path) → `force_ocr=True` **instead of** `skip_text` (L155–L156).
- `OCR_MODE='redo'` → `redo_ocr=True` (L159–L160).
- `OCR_CLEAN='clean-final'` → `clean_final=True` **unless** `OCR_MODE='redo'`, which uses `clean=True` instead (L166–L170); `OCR_CLEAN='none'` → neither key.
- `OCR_DESKEW` true **and** `OCR_MODE != 'redo'` → `deskew=True` (L172–L173).
- **Image input** (not PDF) → an `image_dpi` key is added (detected DPI, else `OCR_IMAGE_DPI`, else a computed A4 DPI; raises if none) (L187–L215). **Not present for PDF input.**
- `OCR_PAGES > 0` → `pages="1-N"` **replaces** the `sidecar` key (sidecar is incompatible with `pages`) (L181–L185).
- `PAPERLESS_OCR_USER_ARGS` (JSON) is merged **last** and can override any key, unless `safe_fallback` (L217–L220); its default is `'{}'` (settings L541), i.e. no overrides.

**In-repo runtime corroboration** (`src/paperless_tesseract/tests/test_parser.py`): `test_ocrmypdf_parameters` (L427) builds the dict via `construct_ocrmypdf_parameters` and asserts `input_file` / `output_file` / `sidecar` (L439), plus the `OCR_CLEAN` matrix (`none`→neither; `clean`→`clean`; `clean-final`+`skip`→`clean_final`; `clean-final`+`redo`→`clean`) and the `OCR_DESKEW` matrix (`skip`→`deskew`; `redo`→none; `false`→none). `TestParserFileTypes` (L474) confirms image inputs (bmp/jpg/gif/tiff) produce an archive PDF and extracted OCR text.


---

## 5. Q3 — Generated Archive PDF & Thumbnail Filenames

> *"After processing finishes, what do the generated archive PDF and thumbnail filenames look like in the media storage area?"*

### 5.1 Answer

For the first processed document (`pk = 1`) under the default configuration:

| Artifact | Path |
|---|---|
| Original | `MEDIA_ROOT/documents/originals/0000001.pdf` |
| **Archive PDF** | `MEDIA_ROOT/documents/archive/0000001.pdf` |
| **Thumbnail** | `MEDIA_ROOT/documents/thumbnails/0000001.png` ← **PNG** in this commit |

The base name is the **zero-padded 7-digit primary key** (`0000001`), with extension `.pdf` for the original/archive and `.png` for the thumbnail.

### 5.2 Evidence & rationale

**Media layout** — `src/paperless/settings.py` L61–L64:

```python
MEDIA_ROOT = os.getenv("PAPERLESS_MEDIA_ROOT", os.path.join(BASE_DIR, "..", "media"))
ORIGINALS_DIR = os.path.join(MEDIA_ROOT, "documents", "originals")
ARCHIVE_DIR   = os.path.join(MEDIA_ROOT, "documents", "archive")
THUMBNAIL_DIR = os.path.join(MEDIA_ROOT, "documents", "thumbnails")
```

**Default naming** — `PAPERLESS_FILENAME_FORMAT` is **unset** by default (`settings.py` L584 → `None`). In `src/documents/file_handling.py` `generate_filename` (L128–L199), the custom-format branch at L132 (`if settings.PAPERLESS_FILENAME_FORMAT is not None:`) is therefore skipped, and execution falls through to the default at L186–L193:

```python
counter_str   = f"_{counter:02}" if counter else ""          # L186
filetype_str  = ".pdf" if archive_filename else doc.file_type # L188
...
filename = f"{doc.pk:07}{counter_str}{filetype_str}"          # L193 (default)
```

- `{doc.pk:07}` zero-pads the primary key to 7 digits → `0000001` for `pk=1`.
- For the **archive**, `filetype_str` is hard-coded to `".pdf"` (L188). For the **original**, it is `doc.file_type` — the `@property` `get_default_file_extension(self.mime_type)` (`models.py` L268–L270), which returns `".pdf"` for `application/pdf` via the tesseract consumer's `mime_types` map (`src/paperless_tesseract/signals.py` L11–L12: `"application/pdf": ".pdf"`).
- The consumer assigns these during the file-move block: `document.filename = generate_unique_filename(document)` (`consumer.py` L316) and `document.archive_filename = generate_unique_filename(document, archive_filename=True)` (`consumer.py` L328).

**Thumbnail is PNG, with no DB field** — `src/documents/models.py` `thumbnail_path` `@property` (L272–L278):

```python
@property
def thumbnail_path(self):
    file_name = "{:07}.png".format(self.pk)
    if self.storage_type == self.STORAGE_TYPE_GPG:
        file_name += ".gpg"
    return os.path.join(settings.THUMBNAIL_DIR, file_name)
```

The thumbnail name is derived **purely from the primary key** — it does **not** consult `PAPERLESS_FILENAME_FORMAT`, and there is **no thumbnail column** in the database (see Q4). The extension is hard-coded **`.png`** in this commit.

**Conflict suffix** — `generate_unique_filename` (L81–L125) appends `_01`, `_02`, … *only* on a filename collision: it loops, incrementing `counter`, until `os.path.join(root, new_filename)` does not already exist (L110–L125). For the first/only document there is no collision, so **no suffix** appears.

**Thumbnail generation pipeline (supporting)** — `src/documents/parsers.py`: `make_thumbnail_from_pdf` (L187) renders the first PDF page to PNG via ImageMagick `convert … archive.pdf[0] → convert.png` (density 300, scaled to 500px wide), with a Ghostscript fallback; `get_optimised_thumbnail` (L319–L340) optionally runs `optipng` → `thumb_optipng.png` when `OPTIMIZE_THUMBNAILS` is on (default true). The parser temp dir is `tempfile.mkdtemp(prefix="paperless-", dir=settings.SCRATCH_DIR)` (L293).

### 5.3 Observed runtime output (live)

```text
$ ls -la media/documents/{originals,archive,thumbnails}/
originals/0000001.pdf    192378 bytes   PDF document, version 1.4   (== uploaded original)
archive/0000001.pdf       97807 bytes   PDF document, version 1.7   (the OCR'd PDF/A)
thumbnails/0000001.png    32238 bytes   PNG image data, 500 x 707, 16-bit grayscale
```

All three names follow the default `0000001.<ext>` pattern exactly; the thumbnail is a **500-px-wide PNG**; no `_NN` suffix appears (single document, no conflict).

> **Note on `pk`:** the exact number depends on how many documents already exist when the upload is processed. On a freshly-migrated dev DB the first document is `pk=1` → `0000001.*`. A different starting count would yield a different zero-padded number, but the `{pk:07}` rule is unchanged.

---

## 6. Q4 — Persisted Database Columns vs. Derived (Filesystem/Computed) Metadata

> *"Check the document record in the database - what fields are actually stored (not filesystem-only metadata) and what values appear for the processed document?"*

### 6.1 Answer

The `Document` model stores **15 real database columns** on the `documents_document` table (including the auto `id`/primary key), plus a **`tags` many-to-many relationship** that is persisted **not as a column** but through a separate join table (`documents_document_tags`). Crucially, several "path" attributes that *look* like stored metadata are actually **computed Python `@property` values, not columns** — and the **thumbnail has no database field at all**. The database stores the **relative** `filename`/`archive_filename` (e.g. `0000001.pdf`), **never** absolute paths; the OCR text is stored in the `content` column.

### 6.2 Persisted columns (with the live values for `pk=1`)

Source: `src/documents/models.py` `Document` L88–L205. Values observed via the Django shell.

| Column | Field type (`models.py`) | Observed value (`pk=1`) |
|---|---|---|
| `id` | `AutoField` (primary key) | `1` |
| `correspondent` | `ForeignKey(Correspondent, null=True, on_delete=SET_NULL)` (L97) | `None` |
| `title` | `CharField(max_length=128, db_index=True)` (L106) | `'test_multipage_image_only'` |
| `document_type` | `ForeignKey(DocumentType, null=True, on_delete=SET_NULL)` (L108) | `None` |
| `content` | `TextField` (L117) — **the OCR text** | `'PAPERLESS OCR TEST DOCUMENT\n\nPage One of Three\nInvoice Number: INV-2024-0001 … END OF DOCUMENT'` |
| `mime_type` | `CharField(max_length=256, editable=False)` (L126) | `'application/pdf'` |
| `checksum` | `CharField(max_length=32, unique=True, editable=False)` (L135) — MD5 of **original** | `'41f8660b78120102d834671cc9df1e9d'` |
| `archive_checksum` | `CharField(max_length=32, null=True, editable=False)` (L143) — MD5 of **archive** | `'156cb2cb294d00f0de7728400c181552'` |
| `created` | `DateTimeField(default=timezone.now, db_index=True)` (L152) | `2026-06-26 21:39:44+00:00` |
| `modified` | `DateTimeField(auto_now=True, editable=False, db_index=True)` (L154) | `2026-06-26 21:39:52.636411+00:00` |
| `storage_type` | `CharField(max_length=11, choices=…, default='unencrypted', editable=False)` (L161) | `'unencrypted'` |
| `added` | `DateTimeField(default=timezone.now, editable=False, db_index=True)` (L169) | `2026-06-26 21:39:52.617916+00:00` |
| `filename` | `FilePathField(max_length=1024, unique=True, null=True, editable=False)` (L176) — **relative** | `'0000001.pdf'` |
| `archive_filename` | `FilePathField(max_length=1024, unique=True, null=True, editable=False)` (L186) — **relative** | `'0000001.pdf'` |
| `archive_serial_number` | `IntegerField(null=True, unique=True, db_index=True)` (L196) | `None` |

That is **15 columns** on `documents_document` (live SQLite `PRAGMA table_info(documents_document)` returns exactly these 15, with the two foreign keys materialised as `correspondent_id` and `document_type_id`).

> **Persisted relationship — `tags` (NOT a `documents_document` column).** `tags` is a `ManyToManyField(Tag)` (`src/documents/models.py:L128-L133`), so it is **not** stored as a column on the `documents_document` row. Django persists it through the auto-generated **join table `documents_document_tags`** (each row pairs a `document_id` with a `tag_id`). It is therefore counted **separately** from the 15 columns above. Observed value for `pk=1`: `[]` (empty) — no tags were assigned (no upload overrides and no matching automation; see §6.4).

### 6.3 NOT columns — computed `@property` (reconstructed at runtime)

Source: `src/documents/models.py`. These are **not** stored in the database; they are derived on access:

| `@property` | How it is computed | Observed value (`pk=1`) |
|---|---|---|
| `source_path` (L222–L231) | `os.path.join(ORIGINALS_DIR, filename)` | `…/media/documents/originals/0000001.pdf` |
| `archive_path` (L241–L246) | `os.path.join(ARCHIVE_DIR, archive_filename)` | `…/media/documents/archive/0000001.pdf` |
| `thumbnail_path` (L272–L278) | `os.path.join(THUMBNAIL_DIR, "{:07}.png".format(pk))` | `…/media/documents/thumbnails/0000001.png` |
| `file_type` (L268–L270) | `get_default_file_extension(mime_type)` | `.pdf` |

> **The thumbnail has no database field at all** — its path is derived purely from the primary key. The DB stores only the **relative** `filename` / `archive_filename`; the absolute paths are reconstructed at runtime by joining `ORIGINALS_DIR` / `ARCHIVE_DIR` / `THUMBNAIL_DIR` with the relative name (or, for thumbnails, with `{pk:07}.png`).

### 6.4 How the row is populated (rationale)

1. **`_store`** (`consumer.py` L379–L412) creates the row inside an atomic transaction:

   ```python
   document = Document.objects.create(
       title=(self.override_title or file_info.title)[:127],   # L399
       content=text,                                            # L400 (OCR text)
       mime_type=mime_type,                                     # L401
       checksum=hashlib.md5(f.read()).hexdigest(),              # L402 (original MD5)
       created=created,                                         # L403
       modified=created,                                        # L404
       storage_type=storage_type,                               # L405 ('unencrypted')
   )
   ```

   Note `created` and `modified` are both seeded with the same derived `created` timestamp here; `modified` then tracks `auto_now` on subsequent saves.

2. **File-move block** (`consumer.py` L316–L346, inside the transaction): sets `filename` (L316), `archive_filename` (L328), and `archive_checksum = md5(archive)` (L340), then `document.save()` (L346). Because `modified` is `auto_now`, this final save advances `modified` to the move time — exactly what we observe (`modified` 21:39:52.636 > `created` 21:39:44).

3. **`created` vs `added` vs `modified`** (a real, observable nuance): `created` came out as `21:39:44` with **zero microseconds**, because `_store` (L389–L393) derives it from the uploaded temp file's mtime, which the view set via `os.utime(..., (t, t))` with an **integer-second** timestamp (`views.py` L508/L518). `added` (`default=timezone.now`) is evaluated when the row is created (`21:39:52.617`), and `modified` (`auto_now`) is set on the final save (`21:39:52.636`). So `created` (document's logical date) precedes `added`/`modified` (when paperless actually wrote the row).

4. **`correspondent` / `document_type` / `tags`** are set **only** via explicit upload overrides **or** post-consume matching. `src/documents/apps.py` (L22–L27) connects the handlers to `document_consumption_finished`:

   ```python
   document_consumption_finished.connect(add_inbox_tags)      # src/documents/signals/handlers.py:L30
   document_consumption_finished.connect(set_correspondent)   # src/documents/signals/handlers.py:L35
   document_consumption_finished.connect(set_document_type)   # src/documents/signals/handlers.py:L101
   document_consumption_finished.connect(set_tags)            # src/documents/signals/handlers.py:L168
   document_consumption_finished.connect(set_log_entry)       # src/documents/signals/handlers.py:L413
   document_consumption_finished.connect(add_to_index)        # src/documents/signals/handlers.py:L428
   ```

   In this run there were no overrides and no trained classifier (`[paperless.classifier] … not performing automatic matching` — §4.4) and no inbox tags defined, so `correspondent`/`document_type` are `None` and `tags` is empty. `add_to_index` (`src/documents/signals/handlers.py:L428` → `index.add_or_update_document`) writes the document into the **Whoosh** full-text search index (a filesystem artifact under `data/index/`, **not** a DB column — and a cleanup target, see §8).

### 6.5 Observed runtime output (live)

```text
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); ..."
id=1  correspondent=None  title='test_multipage_image_only'  document_type=None
content='PAPERLESS OCR TEST DOCUMENT\n\nPage One of Three\nInvoice Number: INV-2024-0001\n\n
         Total Amount: $1234.56\nSecond Page Content\n\nCustomer: Acme Corporation\n
         Date: 2024-01-15\n\nReference: PO-9876\nThird And Final Page\n\n
         Thank you for your business\nSignature line below\n\nEND OF DOCUMENT'
mime_type='application/pdf'
checksum='41f8660b78120102d834671cc9df1e9d'   archive_checksum='156cb2cb294d00f0de7728400c181552'
created=2026-06-26 21:39:44+00:00   added=2026-06-26 21:39:52.617916+00:00   modified=2026-06-26 21:39:52.636411+00:00
storage_type='unencrypted'   filename='0000001.pdf'   archive_filename='0000001.pdf'
archive_serial_number=None   tags=[]
# computed @property (NOT columns):
source_path=.../media/documents/originals/0000001.pdf
archive_path=.../media/documents/archive/0000001.pdf
thumbnail_path=.../media/documents/thumbnails/0000001.png
file_type=.pdf   has_archive_version=True
```

The fully-populated `content` column — the OCR'd text of all three image-only pages — is direct proof that OCR genuinely ran (the input PDF had **zero** embedded text). The two MD5 checksums differ because the archive PDF/A is a distinct file from the original.


---

## 7. Critical Version Caveat

The most prominent public/community log and behavior examples come from **newer** paperless-ngx releases and **do not match this commit**. Verified against `Pipfile.lock`, `Dockerfile`, and the source, this commit (`542221a38dff…`):

- Uses **Django-Q** (`django-q == 1.3.9`, run via `manage.py qcluster`) — **not Celery**, and has **no plugin pipeline** (no `BarcodePlugin` / `ConsumeTaskPlugin` / `WorkflowTriggerPlugin`). Newer releases are Celery-based with a plugin pipeline.
- Runs on **Python 3.9** (`Dockerfile` L18: `FROM python:3.9-slim-bullseye as main-app`). Newer releases use Python 3.11/3.12.
- Pins **`ocrmypdf == 13.4.3`** and **`Django == 4.0.4`** (also `djangorestframework == 3.13.1`, `redis == 3.5.3`, `channels == 3.0.4`, `pikepdf == 5.1.1`, `pdf2image == 1.16.0`, `scikit-learn == 1.0.2`, `whoosh == 2.7.4`, `python-magic == 0.4.25`). Newer releases use Django 5.
- Emits **PNG** thumbnails (`{pk:07}.png`, `models.py` L274). **Newer releases emit WebP.**

**Therefore the exact ordering/set of stage log lines, the OCRmyPDF args dict, and the thumbnail format in this report are taken from THIS commit's `src/documents/consumer.py`, `src/paperless_tesseract/parsers.py`, `src/documents/models.py`, and `src/paperless/settings.py` — the code is authoritative over any newer public example.** This is also why the report does not assume Celery task logs or a `BarcodePlugin`-style pipeline: they do not exist here.

---

## 8. Cleanup / Non-Persistence Confirmation

This was a **read-only investigation**. The only file written anywhere is **this report**, in the destination repository at `blitzy/documentation/paperless-ngx_542221a38dff.md`. No file in the source repository was created, modified, or deleted.

All transient runtime artifacts produced during the investigation were removed, and the cleanup was **re-verified directly** afterward:

- **Database:** the test `Document` row was deleted with a **targeted** query — `Document.objects.filter(pk=1).delete()` (equivalently `Document.objects.filter(checksum="41f8660b78120102d834671cc9df1e9d").delete()`) via the Django shell. A targeted filter is used deliberately so that **only the investigation's own row** is removed and never any pre-existing data — `Document.objects.all().delete()` would be unsafe in any non-pristine database. The table returned to **0 documents**.
- **Media:** the generated files `documents/originals/0000001.pdf`, `documents/archive/0000001.pdf`, and `documents/thumbnails/0000001.png` were removed (paperless's own `post_delete` handler removes the document's files on row deletion; any residue was swept explicitly), returning all three media directories to **0 files**.
- **Search index:** the **Whoosh** entry written by `add_to_index` was removed; the index reports **0 indexed documents** (`doc_count() == 0`).
- **Scratch (`/tmp/paperless`):** for the successfully-consumed document, paperless auto-deleted its own per-file scratch during the run (`Deleting file …` / `Deleting directory …` in §4.4). However, two classes of artifact remained behind in `SCRATCH_DIR` and were **explicitly removed in a dedicated cleanup pass**: (1) the temporary upload file from the **duplicate** re-upload (used in §3.4 to prove the `200` is acceptance-only) — paperless only unlinks a duplicate's temp file when `PAPERLESS_CONSUMER_DELETE_DUPLICATES` is enabled, and it defaults to **false** (`src/paperless/settings.py:L486`; the `os.unlink(self.path)` at `src/documents/consumer.py:L108-L109` is guarded by that flag), so the duplicate's temp upload was intentionally left in place by paperless; and (2) the test-fixture generator script and its intermediate images/OCR-probe files, which were created under `/tmp/paperless` (outside the source tree). After the cleanup pass, `/tmp/paperless` is **empty (0 entries)**.

**Verified post-cleanup state** (captured directly in the running container *after* the cleanup pass — not assumed):

```text
$ ls -A /tmp/paperless | wc -l                                   # scratch dir
0
$ python3 manage.py shell -c "from documents.models import Document; print(Document.objects.count())"
0                                                                # documents
$ for d in originals archive thumbnails; do echo "$d: $(ls -A media/documents/$d | wc -l)"; done
originals: 0
archive: 0
thumbnails: 0                                                    # media dirs
$ python3 manage.py shell -c "from documents.index import open_index; print(open_index().doc_count())"
0                                                                # Whoosh index
```

**Source-tree integrity:** `git status` in the source tree under investigation was verified to report a **clean** working tree — no source file was added, modified, or deleted by this investigation. The verified result:

```text
$ git rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542

$ git status --porcelain        # (no output below = clean working tree)

$ git status
HEAD detached at 542221a38
nothing to commit, working tree clean
```

> **Plainly:** *No source file was created, modified, or deleted; all runtime test artifacts (the DB row, media files, Whoosh entry, and the `/tmp/paperless` scratch — including the duplicate-upload temp file and the test-fixture helpers) were removed and re-verified; `git status` is clean.* The single new file is `blitzy/documentation/paperless-ngx_542221a38dff.md` in the destination repository.

---

## 9. Summary

| # | Question | Answer (this commit, default config) |
|---|---|---|
| **Q1** | HTTP response immediately after upload | **`200 OK`**, JSON body **`"OK"`** (`Content-Type: application/json`); synchronous, returned **before** processing. Unsupported type → `400`; unauthenticated → rejected. |
| **Q2** | Stage log patterns + OCRmyPDF parameters | Ordered `paperless.consumer` / `paperless.parsing.tesseract` lines (`Consuming` → `Detected mime type` → `Parser` → `Parsing` → **`Calling OCRmyPDF with args: {…}`** → `Generating thumbnail` → `Saving record to database` → `consumption finished`). OCRmyPDF args: `skip_text/clean/deskew/rotate_pages(=12.0)/language='eng'/output_type='pdfa'/sidecar`, `jobs = floor(cpu/TASK_WORKERS)` (observed `11`). |
| **Q3** | Archive PDF & thumbnail filenames | `archive/0000001.pdf` and `thumbnails/0000001.png` (**PNG**); originals `originals/0000001.pdf`. Default `{pk:07}` naming; `_NN` suffix only on conflict. |
| **Q4** | DB columns vs. derived metadata | **15 stored columns** on `documents_document` (incl. `content` = OCR text, **relative** `filename`/`archive_filename`, `checksum`/`archive_checksum`, `created`/`modified`/`added`, `mime_type`, `storage_type='unencrypted'`), **plus** the `tags` many-to-many relationship persisted via the join table `documents_document_tags` (not a column). `source_path`/`archive_path`/`thumbnail_path`/`file_type` are computed `@property` values, **not columns**; the **thumbnail has no DB field at all**. |

**Key insight throughout:** acceptance (synchronous, `200 "OK"`) is decoupled from processing (asynchronous, in the `qcluster` worker); the database stores **relative** filenames and the OCR **content** as columns, while the on-disk paths and `file_type` are **computed properties** — and the **thumbnail's location is derived purely from the primary key with no database column**.

