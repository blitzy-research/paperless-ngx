# paperless-ngx v1.7.0 — How a multi-page, image-based PDF flows through the OCR ingestion pipeline

**What this document is.** A read-only, evidence-driven Q&A investigation of **paperless-ngx v1.7.0** that explains — **from actual runtime observation** — exactly what happens when a **multi-page, image-based PDF that requires OCR** is uploaded through the standard REST interface. It answers four questions: (Q1) the immediate HTTP response; (Q2) the processing-stage log lines plus the OCRmyPDF invocation parameters; (Q3) the generated archive/thumbnail filenames; and (Q4) the fields actually persisted to the database.

**Version.** `src/paperless/version.py:1` declares `__version__ = (1, 7, 0)`. The live server independently confirmed this: the Q1 upload response carried the header `x-version: 1.7.0` (shown below).

**Test input (mandatory, not substituted).** `src/paperless_tesseract/tests/samples/multi-page-images.pdf` — an image-based, 3-page PDF (150 479 bytes). `file(1)` on the stored original reported `PDF document, version 1.7, 3 page(s)`. Because its pages carry no text layer, OCR is genuinely exercised under the default `OCR_MODE=skip`. (Its born-digital sibling `multi-page-digital.pdf` would have its text layer copied through and would not demonstrate OCR.)

**Methodology (run-first, read-only).** Every behavioral claim below is backed by concrete, unedited output captured from a live run, next to a `file:line` citation for the code that produced it. The application was built/run in its **default, canonical configuration** (SQLite database, `OCR_MODE=skip`, `OCR_OUTPUT_TYPE=pdfa`, `OCR_LANGUAGE=eng`, `clean`/`deskew`/`rotate_pages` on) using the pinned dependency set (`django==4.0.4`, `djangorestframework==3.13.1`, `django-q==1.3.9`, `ocrmypdf==13.4.3`, …) under the canonical Python 3.9 interpreter. No source file was modified; the only file added is this document. The test document and all runtime artifacts were deleted afterward (see **Cleanup confirmation**). Anything not directly observed at runtime is explicitly labelled **(inferred from code, not observed)**.

**Environment note.** Values that vary per run are the process-local temp directories under `SCRATCH_DIR=/tmp/paperless` (created via `mkdtemp`) and the document primary key. The observed primary key in this run was **`2`** (the SQLite autoincrement sequence had already advanced past `1` during environment setup, then that row was removed — so `Document.objects.count()` read `0` immediately before the upload, yet the next assigned `id` was `2`). Per the run-first mandate, all filenames/values below report the **observed** `pk=2`, not the textbook `1`.

---

## Environment bring-up

**Direct answer.** The app was brought up in its default configuration with four moving parts: **Redis** (django-q broker + Channels layer), the **SQLite** database (migrated), an **admin** superuser for API auth, the **`qcluster`** django-q worker (which executes the consume task and emits all consumer/OCR log lines), and the **gunicorn** ASGI webserver (which handles the upload and returns the immediate response). The exact commands and their key output follow.

Redis was already running; confirmed with a ping:

```bash
$ redis-cli ping
PONG
```

Database migrations were already applied (the canonical SQLite DB lives at `DATA_DIR/db.sqlite3`, `src/paperless/settings.py:299-300`); the `documents` app is migrated through the latest migration, and the pre-upload document count was zero:

```bash
$ cd src && ../venv/bin/python manage.py shell -c \
    "from documents.models import Document; from django.contrib.auth.models import User; \
     print('DOC_COUNT=', Document.objects.count()); \
     print('USERS=', list(User.objects.values_list('username','is_superuser')))"
DOC_COUNT= 0
USERS= [('consumer', False), ('admin', True)]
```

An **admin** superuser (`admin`, `is_superuser=True`) is present and its password authenticates. This is the same non-interactive path the container uses — `docker/docker-prepare.sh:60-63` runs `manage.py manage_superuser` when `PAPERLESS_ADMIN_USER` is set:

```bash
$ cd src && ../venv/bin/python manage.py shell -c \
    "from django.contrib.auth import authenticate; \
     print('AUTH =', 'OK' if authenticate(username='admin', password='admin') else 'FAILED')"
AUTH = OK
```

The **django-q worker** (`qcluster`) was started in the background with its stdout captured. This is the process that runs `documents.tasks.consume_file` and therefore emits every consumer/OCR log line (`docker/supervisord.conf:28-30` runs the same command as `[program:scheduler]`):

```bash
$ cd src && nohup ../venv/bin/python manage.py qcluster > /tmp/qcluster.stdout 2>&1 &
$ head -3 /tmp/qcluster.stdout
04:42:59 [Q] INFO Q Cluster robin-vermont-equal-jersey starting.
04:42:59 [Q] INFO Process-1:1 ready for work at 54562
04:42:59 [Q] INFO Process-1:2 ready for work at 54563
```

The **gunicorn webserver** (ASGI) was started next — the same command `docker/supervisord.conf:10-11` uses for `[program:gunicorn]`. It binds `0.0.0.0:8000` with the `paperless.workers.ConfigurableWorker` (an uvicorn worker), per `gunicorn.conf.py`:

```bash
$ cd src && nohup ../venv/bin/gunicorn -c ../gunicorn.conf.py paperless.asgi:application > /tmp/gunicorn.stdout 2>&1 &
$ head -4 /tmp/gunicorn.stdout
[2026-07-08 04:42:59 +0000] [54540] [INFO] Starting gunicorn 20.1.0
[2026-07-08 04:42:59 +0000] [54540] [INFO] Listening at: http://0.0.0.0:8000 (54540)
[2026-07-08 04:42:59 +0000] [54540] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 04:42:59 +0000] [54540] [INFO] Server is ready. Spawning workers
```

The API was then reachable:

```bash
$ curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/api/
HTTP 200
```

**Why both processes are needed.** The gunicorn webserver alone demonstrates only Q1 (it enqueues the task and returns immediately). The consumer/OCR log lines for Q2, the filesystem artifacts for Q3, and the database row for Q4 are all produced by the `qcluster` worker executing `documents.tasks.consume_file:184` → `Consumer.try_consume_file` (`src/documents/consumer.py:180`). The `[program:consumer]` folder-watcher (`document_consumer`, `docker/supervisord.conf:19-20`) is **not** on the REST upload path and was not relied upon.

---

## Q1 — Immediate HTTP response

**Direct answer.** The upload returns **`HTTP/1.1 200 OK`** with the JSON body **`"OK"`** (the four bytes `"OK"`, quotes included). The response is returned **immediately**, before the document is processed, because the upload is asynchronous.

First an auth token was obtained (DRF token auth); the real token value is redacted here as `<TOKEN>`:

```bash
$ curl -sS -X POST -F "username=admin" -F "password=admin" http://localhost:8000/api/token/
{"token":"<TOKEN>"}
```

Then the mandatory image-based multi-page PDF was POSTed to the real endpoint `api/documents/post_document/` (routed at `src/paperless/urls.py:57-59`, name `post_document`). The full, unedited response — status line, headers, and body:

```bash
$ curl -sS -i -H "Authorization: Token <TOKEN>" \
    -F "document=@src/paperless_tesseract/tests/samples/multi-page-images.pdf" \
    http://localhost:8000/api/documents/post_document/
HTTP/1.1 200 OK
date: Wed, 08 Jul 2026 04:43:41 GMT
server: uvicorn
content-type: application/json
vary: Accept, Accept-Language, Origin, Cookie
allow: POST, OPTIONS
x-frame-options: SAMEORIGIN
x-api-version: 2
x-version: 1.7.0
content-length: 4
content-language: en-us
x-content-type-options: nosniff
referrer-policy: same-origin
cross-origin-opener-policy: same-origin

"OK"
```

**Cause → effect.** The endpoint is served by `PostDocumentView.post` (`src/documents/views.py:497`, class declared at `:491`, `parser_classes = (parsers.MultiPartParser,)` at `:495`). The method:

1. validates the multipart payload with `PostDocumentSerializer` (`src/documents/serialisers.py:413`) — which accepts the fields `document` (FileField, `:415`), `title` (`:420`), `correspondent` (`:426`), `document_type` (`:434`) and `tags` (`:442`), and runs `validate_document` (`:450`) that sniffs the MIME type with `magic` and checks it is supported;
2. writes the bytes to a scratch temp file `NamedTemporaryFile(prefix="paperless-upload-", dir=settings.SCRATCH_DIR, delete=False)` (`src/documents/views.py:512-520`);
3. generates `task_id = str(uuid.uuid4())` (`:521`) and enqueues the work with `async_task("documents.tasks.consume_file", temp_filename, …)` (`:523-534`) onto the django-q/Redis broker;
4. returns **`return Response("OK")`** (`:535`).

Because step 4 runs synchronously right after the *enqueue* (not after processing), the client receives `200 OK` while the worker has not yet touched the document. DRF renders the Python string `"OK"` as a JSON string, so the wire body is the quoted token `"OK"` — exactly the 4 bytes reported by `content-length: 4`. This matches the read-only corroboration test `test_upload` in `src/documents/tests/test_api.py`, which POSTs a sample PDF to the same route and asserts `status_code == 200` with `async_task` called once.

**Stability.** A second, identical upload returned the same result (`HTTP 200`, body `"OK"`), confirming the immediate response is deterministic and independent of whether the document will ultimately be accepted (see the duplicate-rejection note in Q2):

```bash
$ curl -sS -o /dev/null -w "HTTP %{http_code} | content-length %{size_download}\n" \
    -H "Authorization: Token <TOKEN>" \
    -F "document=@src/paperless_tesseract/tests/samples/multi-page-images.pdf" \
    http://localhost:8000/api/documents/post_document/
HTTP 200 | content-length 4
```

---

## Q2 — Processing stage logs & OCRmyPDF parameters

**Direct answer.** As the `qcluster` worker consumes the document, it emits an ordered set of stage log lines under the `paperless.consumer`, `paperless.parsing`, `paperless.parsing.tesseract` and `paperless.classifier` loggers, ending with `Document … consumption finished`. Immediately before the OCR call, `RasterisedDocumentParser` logs the complete OCRmyPDF keyword-argument dict. The full, unedited block captured from `data/log/paperless.log` for this run:

```text
[2026-07-08 04:43:42,840] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 04:43:42,841] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 04:43:42,842] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 04:43:42,844] [DEBUG] [paperless.consumer] Parsing multi-page-images.pdf...
[2026-07-08 04:43:42,864] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-jgyld_lv
[2026-07-08 04:43:42,903] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-jgyld_lv', 'output_file': '/tmp/paperless/paperless-sueqi99v/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-sueqi99v/sidecar.txt'}
[2026-07-08 04:43:44,783] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-08 04:43:44,784] [DEBUG] [paperless.consumer] Generating thumbnail for multi-page-images.pdf...
[2026-07-08 04:43:44,787] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-sueqi99v/archive.pdf[0] /tmp/paperless/paperless-sueqi99v/convert.png
[2026-07-08 04:43:44,806] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 04:43:44,942] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-sueqi99v/gs_out.png /tmp/paperless/paperless-sueqi99v/convert_gs.png
[2026-07-08 04:43:44,998] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-sueqi99v/convert_gs.png -out /tmp/paperless/paperless-sueqi99v/thumb_optipng.png
[2026-07-08 04:43:45,329] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 04:43:45,332] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 04:43:45,351] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-jgyld_lv
[2026-07-08 04:43:45,379] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-sueqi99v
[2026-07-08 04:43:45,380] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
```

The log line format `[{asctime}] [{levelname}] [{name}] {message}` is the verbose formatter defined at `src/paperless/settings.py:378`; these lines land in `data/log/paperless.log` via the file handler at `:395`.

### The ordered stage lines and where each is emitted

Each consumer line is produced by `self.log(level, message)` (`LoggingMixin.log`, `src/documents/loggers.py:14`, which calls `getLogger(self.logging_name).<level>(message, extra={"group": …})`). The named stages, in order:

| # | Observed line (logger) | Emitted by |
|---|------------------------|------------|
| 1 | `[paperless.consumer] Consuming multi-page-images.pdf` (INFO) | `Consumer.try_consume_file` → `src/documents/consumer.py:215` |
| 2 | `[paperless.consumer] Detected mime type: application/pdf` (DEBUG) | `src/documents/consumer.py:221` |
| 3 | `[paperless.consumer] Parser: RasterisedDocumentParser` (DEBUG) | `src/documents/consumer.py:246` — the selected parser class name |
| 4 | `[paperless.consumer] Parsing multi-page-images.pdf...` (DEBUG) | `src/documents/consumer.py:260` (immediately before `document_parser.parse(...)`) |
| 5 | `[paperless.parsing.tesseract] Extracted text from PDF file …` (DEBUG) | `RasterisedDocumentParser.parse` → `extract_text(None, document_path)` at `src/paperless_tesseract/parsers.py:236` (checks whether the original already has a text layer) |
| 6 | `[paperless.parsing.tesseract] Calling OCRmyPDF with args: {…}` (DEBUG) | `src/paperless_tesseract/parsers.py:260`, immediately before `ocrmypdf.ocr(**args)` at `:261` |
| 7 | `[paperless.parsing.tesseract] Using text from sidecar file` (DEBUG) | after OCR, `parse` reads the sidecar text (`src/paperless_tesseract/parsers.py:264` area) |
| 8 | `[paperless.consumer] Generating thumbnail for multi-page-images.pdf...` (DEBUG) | `src/documents/consumer.py:263` |
| 9 | `[paperless.parsing] Execute: convert … archive.pdf[0] … convert.png` (DEBUG) | base `DocumentParser` thumbnail helper (logger `paperless.parsing`, `src/documents/parsers.py:287`) — ImageMagick attempt |
| 10 | `[paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript…` (WARNING) | base parser thumbnail fallback (see note below) |
| 11 | `[paperless.parsing] Execute: convert … gs_out.png … convert_gs.png` (DEBUG) | ghostscript-fallback thumbnail rasterization |
| 12 | `[paperless.parsing.tesseract] Execute: optipng … thumb_optipng.png` (DEBUG) | thumbnail PNG optimization |
| 13 | `[paperless.classifier] Document classification model does not exist (yet)…` (DEBUG) | the classifier — no trained model, so no auto tag/type matching |
| 14 | `[paperless.consumer] Saving record to database` (DEBUG) | `Consumer._store` → `src/documents/consumer.py:387` |
| 15 | `[paperless.consumer] Deleting file /tmp/paperless/paperless-upload-…` (DEBUG) | cleanup of the scratch upload temp file |
| 16 | `[paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-…` (DEBUG) | cleanup of the parser tempdir |
| 17 | `[paperless.consumer] Document 2026-07-08 multi-page-images consumption finished` (INFO) | `src/documents/consumer.py:373` |

The trailing text `2026-07-08 multi-page-images` in line 17 is the model's `str(document)` (`"{created:%Y-%m-%d} {title}"`); it confirms the parsed `title` (`multi-page-images`) and `created` date (`2026-07-08`) that Q4 shows as stored columns.

Lines 9–12 show that thumbnail generation first tried ImageMagick and **fell back to Ghostscript** after ImageMagick's PDF policy blocked it (the `WARNING … Check your /etc/ImageMagick-x/policy.xml!`). This is an observed, environment-specific fallback within the thumbnailer; the thumbnail was still produced (see Q3). *(The specific ImageMagick-policy cause is inferred from the warning text, not separately verified.)*

### The OCRmyPDF invocation parameters

The complete, unedited dict logged at `src/paperless_tesseract/parsers.py:260` (the values below are the **real captured** ones — `jobs` and the tempdir paths are runtime values, not placeholders):

```json
{
  "input_file": "/tmp/paperless/paperless-upload-jgyld_lv",
  "output_file": "/tmp/paperless/paperless-sueqi99v/archive.pdf",
  "use_threads": true,
  "jobs": 11,
  "language": "eng",
  "output_type": "pdfa",
  "progress_bar": false,
  "skip_text": true,
  "clean": true,
  "deskew": true,
  "rotate_pages": true,
  "rotate_pages_threshold": 12.0,
  "sidecar": "/tmp/paperless/paperless-sueqi99v/sidecar.txt"
}
```

The dict is assembled by `RasterisedDocumentParser.construct_ocrmypdf_parameters` (`src/paperless_tesseract/parsers.py:135`). Each key, its source setting, the parser line that adds it, and its meaning:

| Key = observed value | Source setting (`src/paperless/settings.py`) | Added at (`src/paperless_tesseract/parsers.py`) | Meaning (OCRmyPDF 13.4.3) |
|----------------------|----------------------------------------------|--------------------------------------------------|----------------------------|
| `input_file` = `/tmp/paperless/paperless-upload-jgyld_lv` | — (the scratch upload temp from the view) | base dict `:143` | the PDF handed to OCRmyPDF |
| `output_file` = `/tmp/paperless/paperless-sueqi99v/archive.pdf` | — (parser tempdir) | base dict | where the OCR'd archive PDF is written |
| `use_threads` = `True` | — (constant) | `:148` | run OCRmyPDF's stages threaded rather than multi-process |
| `jobs` = `11` | `THREADS_PER_WORKER` (computed, `:469`) | `:149` | cap concurrency at N workers; observed **11** on this machine (do not assume — it is CPU-derived) |
| `language` = `eng` | `OCR_LANGUAGE` (`:514`, default `eng`) | `:150` | Tesseract language pack(s) to use; OCRmyPDF assumes English unless told otherwise |
| `output_type` = `pdfa` | `OCR_OUTPUT_TYPE` (`:518`, default `pdfa`) | `:151` | produce a PDF/A file for long-term archiving. On 13.4.3 the library default was also `pdfa`; the default only changed to `auto` in OCRmyPDF **v17.0.0**, so `pdfa` here is both the paperless default and the pinned-library default |
| `progress_bar` = `False` | — (constant) | `:152` | disable the tqdm progress bar (correct for a daemonized worker) |
| `skip_text` = `True` | `OCR_MODE` (`:522`, default `skip`) | `:157-158` | with skip-text, pages that already carry text are copied through untouched while text-less (scanned/image) pages are OCR'd — normalizing everything to PDF/A regardless of contents |
| `clean` = `True` | `OCR_CLEAN` (`:526`, default `clean`) | `:165` | run **unpaper** to clean scanning artifacts from pages before OCR; it does not alter the final visible output |
| `deskew` = `True` | `OCR_DESKEW` (`:528`, default true) | `:173` | rotate pages scanned at a small skew back to horizontal |
| `rotate_pages` = `True` | `OCR_ROTATE_PAGES` (`:530`, default true) | `:176` | detect the correct cardinal orientation of each page and rotate if wrong |
| `rotate_pages_threshold` = `12.0` | `OCR_ROTATE_PAGES_THRESHOLD` (`:532`, default `12.0`) | `:178` | only rotate when Tesseract's orientation confidence exceeds this value; paperless lowers OCRmyPDF's conservative default to 12.0 |
| `sidecar` = `/tmp/paperless/paperless-sueqi99v/sidecar.txt` | added because `OCR_PAGES == 0` (`:510`) | `:185` (`sidecar_file` computed at `parse` `:250`) | write a companion text file with the recognized OCR text; paperless then reads it back (`Using text from sidecar file`, line 7). Pages that already had text are excluded from the sidecar |

Two keys are notably **absent**, and both absences are correct:

- **No `image_dpi`.** That key is only added for image inputs (`is_image` true). The input MIME is `application/pdf` (line 2), so `is_image` is false and no `image_dpi` is set.
- **No `pages`.** The `pages` key is only added when `OCR_PAGES > 0` (`src/paperless_tesseract/parsers.py:181` area); the default `OCR_PAGES=0` (`src/paperless/settings.py:510`) means the whole document is processed and a `sidecar` entry is added instead.

The read-only corroboration test `test_ocrmypdf_parameters` (`src/paperless_tesseract/tests/test_parser.py:427`) asserts exactly these `input_file`/`output_file`/`sidecar` keys and the conditional `clean`/`clean_final`/`deskew` flags; `test_multi_page` (`:269`) asserts the parser produces an archive and extracts the "page 1/2/3" text — which Q4's `content` column confirms at runtime.

### The `skip` vs `skip_noarchive` boundary (must be explained)

`RasterisedDocumentParser.parse` contains a shortcut that skips OCRmyPDF **entirely**, but it fires only under a non-default mode. At `src/paperless_tesseract/parsers.py:241` the guard is:

```python
if settings.OCR_MODE == "skip_noarchive" and original_has_text:
    # ... skip calling ocrmypdf, no archive produced
```

`original_has_text` is set at `:236` from `len(text_original) > 50`. Under the **default** `OCR_MODE == "skip"` this branch is **not** taken, so OCRmyPDF is **always** invoked (with `skip_text=True`) and a PDF/A archive is always produced. That is why the `Calling OCRmyPDF with args` line (line 6) reliably appears for this image-based input — and it did.

### Fallback path — did it fire?

If the first `ocrmypdf.ocr(**args)` raises, `parse` retries and logs `Fallback: Calling OCRmyPDF with args: {…}` (`src/paperless_tesseract/parsers.py:297`) before a second `ocrmypdf.ocr(**args)` (`:298`). **In this run the fallback did NOT fire** — the captured log contains no `Fallback:` line; the single `Calling OCRmyPDF with args` line (line 6) was followed directly by successful sidecar text extraction (line 7).

### Duplicate-guard edge case (observed)

A second, identical upload (same bytes → same checksum) was **rejected** by the consumer before any OCR. The only new log line produced was:

```text
[2026-07-08 04:46:07,427] [ERROR] [paperless.consumer] Not consuming multi-page-images.pdf: It is a duplicate.
```

No second `Calling OCRmyPDF with args` line was emitted, and the document count stayed at 1. This confirms two things: (a) the immediate `200 OK` (Q1) is returned regardless — the webserver enqueues and responds before the worker detects the duplicate; and (b) the OCRmyPDF argument dict is **deterministic** (it is derived entirely from the `OCR_*` settings and `THREADS_PER_WORKER` in `construct_ocrmypdf_parameters`), so the only parts that vary run-to-run are the random `mkdtemp` tempdir paths in `input_file`/`output_file`/`sidecar`.

---

## Q3 — Generated archive & thumbnail filenames

**Direct answer.** With the default (empty) `PAPERLESS_FILENAME_FORMAT`, every stored file is named from the zero-padded 7-digit primary key. For the observed `pk=2` the media area contains:

- original: `media/documents/originals/0000002.pdf`
- **archive PDF: `media/documents/archive/0000002.pdf`**
- **thumbnail: `media/documents/thumbnails/0000002.png`**

**Before** the upload, all three media directories were empty:

```bash
$ ls -la media/documents/originals media/documents/archive media/documents/thumbnails
=== BEFORE: media/documents/originals ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:25 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
=== BEFORE: media/documents/archive ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:25 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
=== BEFORE: media/documents/thumbnails ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:25 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
```

**After** `consumption finished`, each directory holds exactly one file:

```bash
$ ls -la media/documents/originals media/documents/archive media/documents/thumbnails
=== AFTER: media/documents/originals ===
total 156
drwxr-xr-x 2 root root   4096 Jul  8 04:43 .
drwxr-xr-x 5 root root   4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 150479 Jul  8 04:43 0000002.pdf
=== AFTER: media/documents/archive ===
total 32
drwxr-xr-x 2 root root  4096 Jul  8 04:43 .
drwxr-xr-x 5 root root  4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 24207 Jul  8 04:43 0000002.pdf
=== AFTER: media/documents/thumbnails ===
total 12
drwxr-xr-x 2 root root 4096 Jul  8 04:43 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 2421 Jul  8 04:43 0000002.png
```

`file(1)` on the three artifacts, confirming what they are (the original is the 3-page input; the archive is a PDF/A produced by OCRmyPDF; the thumbnail is a 500-px-wide grayscale PNG):

```bash
$ file media/documents/originals/0000002.pdf media/documents/archive/0000002.pdf media/documents/thumbnails/0000002.png
media/documents/originals/0000002.pdf:  PDF document, version 1.7, 3 page(s)
media/documents/archive/0000002.pdf:    PDF document, version 1.7
media/documents/thumbnails/0000002.png: PNG image data, 500 x 647, 8-bit grayscale, non-interlaced
```

The archive carries a PDF/A-2b XMP identity block (`pdfaid:part="2" pdfaid:conformance="B"`), confirming `output_type='pdfa'` produced a **PDF/A-2b** file.

**Cause → effect.**

- **The directories** come from `src/paperless/settings.py`: `MEDIA_ROOT` (`:61`), `ORIGINALS_DIR` = `media/documents/originals` (`:62`), `ARCHIVE_DIR` = `media/documents/archive` (`:63`), `THUMBNAIL_DIR` = `media/documents/thumbnails` (`:64`).
- **The `.pdf` names** come from `generate_filename` (`src/documents/file_handling.py:128`). Because `PAPERLESS_FILENAME_FORMAT` is `None` by default (`src/paperless/settings.py:584`; guard at `file_handling.py:132`), the custom-format branch is skipped and the function falls through to `counter_str = f"_{counter:02}" if counter else ""` (`:186`), `filetype_str = ".pdf" if archive_filename else doc.file_type` (`:188`), and finally `filename = f"{doc.pk:07}{counter_str}{filetype_str}"` (`:193`). With `pk=2` and no name collision, `counter_str` is empty, so both the original and the archive resolve to `0000002.pdf`. The consumer assigns these via `generate_unique_filename` at `src/documents/consumer.py:316` (original) and `:328` (archive, `archive_filename=True`).
- **The thumbnail name** comes from the `thumbnail_path` property (`src/documents/models.py:273`), which joins `THUMBNAIL_DIR` with `"{:07}.png".format(self.pk)` → `0000002.png`.

`counter_str` is empty here because there was no filename collision; it would become `_01`, `_02`, … only if a same-named file already existed. *(That collision branch is not exercised in this run — inferred from `file_handling.py:186`, not observed.)*

---

## Q4 — Persisted database fields

**Direct answer.** The `documents_document` row (SQLite, `src/paperless/settings.py:299-300`) stores **15 real columns**. For the processed document their observed values are:

| Column (ORM field) | DB column | Observed value |
|--------------------|-----------|----------------|
| `id` (pk) | `id` INTEGER | `2` |
| `title` | `title` varchar(128) | `'multi-page-images'` |
| `content` | `content` TEXT | `'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'` |
| `mime_type` | `mime_type` varchar(256) | `'application/pdf'` |
| `checksum` | `checksum` varchar(32) | `'62acb0bcbfbcaa62ca6ad3668e4e404b'` (MD5 of the original) |
| `archive_checksum` | `archive_checksum` varchar(32) | `'2a21478186c02e38dc187ebc9b84cc1c'` (MD5 of the archive) |
| `created` | `created` datetime | `2026-07-08 04:43:42+00:00` |
| `modified` | `modified` datetime | `2026-07-08 04:43:45.350637+00:00` |
| `added` | `added` datetime | `2026-07-08 04:43:45.333248+00:00` |
| `storage_type` | `storage_type` varchar(11) | `'unencrypted'` |
| `filename` | `filename` varchar(1024) | `'0000002.pdf'` |
| `archive_filename` | `archive_filename` varchar(1024) | `'0000002.pdf'` |
| `correspondent` | `correspondent_id` INTEGER (FK) | `None` |
| `document_type` | `document_type_id` INTEGER (FK) | `None` |
| `archive_serial_number` | `archive_serial_number` INTEGER | `None` |

The ORM dump (every DB field via `d._meta.fields`, plus the three computed path properties for contrast):

```bash
$ cd src && ../venv/bin/python manage.py shell -c \
    "from documents.models import Document; d=Document.objects.latest('added'); \
     [print(f.name,'=',repr(getattr(d,f.name))) for f in d._meta.fields]; \
     print('source_path =', d.source_path); print('archive_path =', d.archive_path); \
     print('thumbnail_path =', d.thumbnail_path)"
--- DB COLUMNS (d._meta.fields) ---
id = 2
correspondent = None
title = 'multi-page-images'
document_type = None
content = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
mime_type = 'application/pdf'
checksum = '62acb0bcbfbcaa62ca6ad3668e4e404b'
archive_checksum = '2a21478186c02e38dc187ebc9b84cc1c'
created = datetime.datetime(2026, 7, 8, 4, 43, 42, tzinfo=datetime.timezone.utc)
modified = datetime.datetime(2026, 7, 8, 4, 43, 45, 350637, tzinfo=datetime.timezone.utc)
storage_type = 'unencrypted'
added = datetime.datetime(2026, 7, 8, 4, 43, 45, 333248, tzinfo=datetime.timezone.utc)
filename = '0000002.pdf'
archive_filename = '0000002.pdf'
archive_serial_number = None
--- COMPUTED PROPERTIES (NOT columns) ---
source_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/originals/0000002.pdf
archive_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/archive/0000002.pdf
thumbnail_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/thumbnails/0000002.png
```

To prove these are genuine table columns (and to read the byte-sensitive checksums straight from the stored row rather than recomputing them), the raw row was dumped with Python's `sqlite3` module (the `sqlite3` CLI is not installed in this image):

```bash
$ ./venv/bin/python -c "import sqlite3; c=sqlite3.connect('data/db.sqlite3').cursor(); \
    c.execute('SELECT * FROM documents_document ORDER BY id DESC LIMIT 1'); \
    cols=[d[0] for d in c.description]; \
    [print(k,'=',repr(v)) for k,v in zip(cols,c.fetchone())]"
id = 2
title = 'multi-page-images'
content = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
created = '2026-07-08 04:43:42'
modified = '2026-07-08 04:43:45.350637'
correspondent_id = None
checksum = '62acb0bcbfbcaa62ca6ad3668e4e404b'
added = '2026-07-08 04:43:45.333248'
storage_type = 'unencrypted'
archive_serial_number = None
document_type_id = None
mime_type = 'application/pdf'
archive_checksum = '2a21478186c02e38dc187ebc9b84cc1c'
archive_filename = '0000002.pdf'
filename = '0000002.pdf'
```

`PRAGMA table_info(documents_document)` lists the real columns (note the FK columns are physically named `correspondent_id` and `document_type_id`):

```text
cid|name|type|notnull|dflt_value|pk
0|id|INTEGER|1|None|1
1|title|varchar(128)|1|None|0
2|content|TEXT|1|None|0
3|created|datetime|1|None|0
4|modified|datetime|1|None|0
5|correspondent_id|INTEGER|0|None|0
6|checksum|varchar(32)|1|None|0
7|added|datetime|1|None|0
8|storage_type|varchar(11)|1|None|0
9|archive_serial_number|INTEGER|0|None|0
10|document_type_id|INTEGER|0|None|0
11|mime_type|varchar(256)|1|None|0
12|archive_checksum|varchar(32)|0|None|0
13|archive_filename|varchar(1024)|0|None|0
14|filename|varchar(1024)|0|None|0
```

The two checksums are the MD5 of the actual stored files (cross-checked, not recomputed from a re-serialization) — `md5sum` of the media files matches the DB columns exactly:

```bash
$ md5sum media/documents/originals/0000002.pdf media/documents/archive/0000002.pdf
62acb0bcbfbcaa62ca6ad3668e4e404b  media/documents/originals/0000002.pdf
2a21478186c02e38dc187ebc9b84cc1c  media/documents/archive/0000002.pdf
```

### Stored columns vs. computed properties (the crux)

`source_path`, `archive_path` and `thumbnail_path` are **not** database columns — they are Python `@property` accessors on the `Document` model that compose a filesystem path at access time from `MEDIA_ROOT` and the stored `filename`/`archive_filename`/`pk`:

- `source_path` — `src/documents/models.py:223`
- `archive_path` — `src/documents/models.py:242` (returns `None` unless `has_archive_version`, `:238`)
- `thumbnail_path` — `src/documents/models.py:273`

The proof that they are not columns: they do **not** appear in `PRAGMA table_info` above, yet they resolve on the ORM instance (shown in the ORM dump). Programmatic confirmation:

```bash
$ ./venv/bin/python -c "import sqlite3; c=sqlite3.connect('data/db.sqlite3').cursor(); \
    c.execute('PRAGMA table_info(documents_document)'); n=[r[1] for r in c.fetchall()]; \
    [print(p,'in columns?',p in n) for p in ('source_path','archive_path','thumbnail_path')]"
source_path in columns? False
archive_path in columns? False
thumbnail_path in columns? False
```

### Where each stored column is written

`Consumer._store` (`src/documents/consumer.py:379`) creates the row with `Document.objects.create(...)` (`:398`), setting `title=(self.override_title or file_info.title)[:127]`, `content=text` (the OCR sidecar text), `mime_type`, `checksum=hashlib.md5(...).hexdigest()`, `created`, `modified=created`, and `storage_type=STORAGE_TYPE_UNENCRYPTED` (`'unencrypted'`). `added` is auto-populated (`default=timezone.now`, `src/documents/models.py:169`). Afterwards, back in `try_consume_file`, three more columns are assigned and saved: `filename` (`:316`), `archive_filename` (`:328`), and `archive_checksum = hashlib.md5(f.read()).hexdigest()` (`:340`). `correspondent` and `document_type` remain `None` because the upload supplied no overrides and no classifier model exists (Q2 line 13); `archive_serial_number` is `None` (assigned only when a document is moved to the archive-serial workflow — not exercised here). The model column declarations backing these values are `title` (`:106`), `content` (`:117`), `mime_type` (`:126`), `checksum` (`:135`), `archive_checksum` (`:143`), `created` (`:152`), `modified` (`:154`), `storage_type` (`:161`), `added` (`:169`), `filename` (`:176`), `archive_filename` (`:186`), `archive_serial_number` (`:196`), `correspondent` (`:97`), `document_type` (`:108`).

The `content` value (`This is a multi page document. Page 1/2/3.`) is the text OCRmyPDF wrote to the sidecar and paperless read back — direct runtime proof that OCR ran on all three image pages, matching the `test_multi_page` expectation in `src/paperless_tesseract/tests/test_parser.py:269`.

---

## Cleanup confirmation

**Direct answer.** The test document and every runtime artifact it produced were removed, the temporary observation scripts/processes were torn down, and the repository was left pristine — `git status` shows only this one new documentation file as untracked, and no tracked source file was modified.

The document was deleted through the ORM; the model's delete handling also removes its media files (original, archive, thumbnail) and its full-text-search index entry:

```bash
$ cd src && ../venv/bin/python manage.py shell -c \
    "from documents.models import Document; \
     d=Document.objects.latest('added'); pk=d.pk; d.delete(); \
     print('deleted pk', pk, '-> remaining DOC_COUNT =', Document.objects.count())"
deleted pk 2 -> remaining DOC_COUNT = 0
```

The media directories were empty again afterward:

```bash
$ ls -la media/documents/originals media/documents/archive media/documents/thumbnails
media/documents/archive:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..

media/documents/originals:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..

media/documents/thumbnails:
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
```

The scratch upload temp files, the captured worker stdout, and the helper evidence files were removed, and the `qcluster`/`gunicorn` processes started for this investigation were stopped (by the exact PIDs captured at start — no broad `pkill`):

```bash
$ rm -f /tmp/paperless/paperless-upload-* /tmp/qcluster.stdout /tmp/gunicorn.stdout
$ kill "$(cat /tmp/blitzy_evidence/qcluster.pid)" "$(cat /tmp/blitzy_evidence/gunicorn.pid)"
$ rm -rf /tmp/blitzy_evidence
```

Finally, `git status` confirms the working tree is pristine — the only change is the new, untracked answer document (runtime data under `media/`, `data/`, and `*.log` is already gitignored, so it never dirties tracked files):

```bash
$ git status
On branch blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	blitzy/

nothing added to commit but untracked files present (use "git add" to track)
```

No tracked file was modified and no file other than `blitzy/documentation/paperless-ngx_542221a38dff.md` was added, satisfying the read-only mandate.

