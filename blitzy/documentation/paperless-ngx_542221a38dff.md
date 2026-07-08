# paperless-ngx v1.7.0 — How a multi-page, image-based PDF flows through the OCR ingestion pipeline

**What this document is.** A read-only, evidence-driven Q&A investigation of **paperless-ngx v1.7.0** that explains — **from actual runtime observation** — exactly what happens when a **multi-page, image-based PDF that requires OCR** is uploaded through the standard REST interface. It answers four questions: (Q1) the immediate HTTP response; (Q2) the processing-stage log lines plus the OCRmyPDF invocation parameters; (Q3) the generated archive/thumbnail filenames; and (Q4) the fields actually persisted to the database.

**Version.** `src/paperless/version.py:1` declares `__version__ = (1, 7, 0)`. The live server independently confirmed this: the Q1 upload response carried the header `x-version: 1.7.0` (shown below).

**Test input (mandatory, not substituted).** `src/paperless_tesseract/tests/samples/multi-page-images.pdf` — an image-based, 3-page PDF (150 479 bytes). `file(1)` on the stored original reported `PDF document, version 1.7, 3 page(s)`. Its pages carry **no text layer**, which was verified at runtime *before* OCR (not merely assumed): `pdftotext` extracts only 3 bytes (three form-feed page separators, no words) and the application's own extractor (`pdfminer.six`, the exact call at `src/paperless_tesseract/parsers.py:122`) returns an empty string, so `original_has_text` is `False`. The full observed evidence is in **Environment bring-up → Pre-OCR text-layer inspection**. Because there is no text layer, OCR is genuinely exercised under the default `OCR_MODE=skip`. (Its born-digital sibling `multi-page-digital.pdf` would have its text layer copied through and would not demonstrate OCR.)

**Methodology (run-first, read-only).** Every behavioral claim below is backed by concrete, unedited output captured from a live run, next to a `file:line` citation for the code that produced it. The application was built/run in its **default, canonical configuration** (SQLite database, `OCR_MODE=skip`, `OCR_OUTPUT_TYPE=pdfa`, `OCR_LANGUAGE=eng`, `clean`/`deskew`/`rotate_pages` on) inside the user-specified canonical runtime image `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542` (from `ghcr.io/scaleapi/swe-atlas`), using the pinned dependency set (`django==4.0.4`, `djangorestframework==3.13.1`, `django-q==1.3.9`, `ocrmypdf==13.4.3`, …) under the canonical Python **3.9** interpreter (observed `Python 3.9.25`; see below). No source file was modified; the only file added is this document. The test documents and all runtime artifacts were deleted afterward (see **Cleanup confirmation**). Anything not directly observed at runtime is explicitly labelled **(inferred from code, not observed)**.

**Security note.** The local admin credential (`admin`) was used **only** to authenticate the API calls in this local investigation. Its password is **redacted** as `<REDACTED>` in every embedded command and output below, and the DRF auth token is redacted as `<TOKEN>`; only non-secret proof (e.g. `AUTH = OK`) is shown.

**Environment note.** Values that vary per run are the process-local temp directories under `SCRATCH_DIR=/tmp/paperless` (created via `mkdtemp`) and the document primary key. The observed primary key of the primary run was **`3`** (the SQLite autoincrement sequence had already advanced past `1`/`2` during earlier environment setup, and those rows were removed — so `Document.objects.count()` read `0` immediately before the upload, yet the next assigned `id` was `3`). Per the run-first mandate, all filenames/values below report the **observed** `pk=3` (and `pk=4` for the second stability run), not the textbook `1`.

---

## Environment bring-up

**Direct answer.** The app was brought up in its default configuration inside the canonical runtime image with these moving parts: **Redis** (django-q broker + Channels layer), the **SQLite** database (migrated), an **admin** superuser for API auth, the **`qcluster`** django-q worker (which executes the consume task and emits all consumer/OCR log lines), and the **gunicorn** ASGI webserver (which handles the upload and returns the immediate response). The exact commands and their key output follow.

### Canonical runtime & interpreter

The runtime is the user-specified container image `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542` (per the task's setup instructions / AAP §0.8.1; the image tag is not queryable from inside the container, so it is cited rather than observed). The **observed** interpreter and key system tools were:

```bash
$ ../venv/bin/python --version
Python 3.9.25

$ cat /etc/os-release | head -3
PRETTY_NAME="Ubuntu 25.10"
NAME="Ubuntu"
VERSION_ID="25.10"

$ gs --version
9.53.3

$ tesseract --version 2>&1 | head -1
tesseract 5.5.0
```

The application runs under the venv's **Python 3.9.25**, which is the canonical **3.9** series pinned by the project (`Dockerfile`/CI). *(The AAP references a `python:3.9-slim-bullseye` base; the provided image instead uses an Ubuntu 25.10 base with a uv-managed 3.9.25 venv — the Python 3.9 series matches, which is what the OCR/library pins require. Reported honestly from the observed `python --version` / `os-release` above.)* Ghostscript **9.53.3** is used (OCRmyPDF 13.4.3 is incompatible with gs 10.x), resolved via `/usr/local/bin` ahead of `/usr/bin` on `PATH`.

### Redis

Redis was already running; confirmed with a ping:

```bash
$ redis-cli ping
PONG
```

### Database migrations

The canonical SQLite DB lives at `DATA_DIR/db.sqlite3` (`src/paperless/settings.py:300`). Applying migrations shows the schema is already fully migrated (idempotent — the expected steady state), and `showmigrations documents` confirms the `documents` app is migrated through its latest migration:

```bash
$ cd src && ../venv/bin/python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.

$ ../venv/bin/python manage.py showmigrations documents | tail -6
 [X] 1013_migrate_tag_colour
 [X] 1014_auto_20210228_1614
 [X] 1015_remove_null_characters
 [X] 1016_auto_20210317_1351
 [X] 1017_alter_savedviewfilterrule_rule_type
 [X] 1018_alter_savedviewfilterrule_value
```

### Admin superuser provisioning

An **admin** superuser (`admin`, `is_superuser=True`) is present. It is provisioned by the same non-interactive path the container uses — `docker/docker-prepare.sh:60-63` runs `manage.py manage_superuser` when `PAPERLESS_ADMIN_USER` is set, and `manage_superuser` (`src/documents/management/commands/manage_superuser.py`) creates the user or, if it already exists, resets its password. Running that exact command confirms provisioning (its output never prints the password), and a Django `authenticate()` check confirms the credential works — password **redacted**:

```bash
$ ../venv/bin/python manage.py shell -c \
    "from django.contrib.auth.models import User; \
     print('SUPERUSERS =', list(User.objects.filter(is_superuser=True).values_list('username','is_superuser')))"
SUPERUSERS = [('admin', True)]

$ PAPERLESS_ADMIN_USER=admin PAPERLESS_ADMIN_PASSWORD=<REDACTED> ../venv/bin/python manage.py manage_superuser
Changed password of user admin.

$ PW=<REDACTED> ../venv/bin/python manage.py shell -c \
    "from django.contrib.auth import authenticate; import os; \
     print('AUTH =', 'OK' if authenticate(username='admin', password=os.environ['PW']) else 'FAILED')"
AUTH = OK
```

### django-q worker (`qcluster`)

The **django-q worker** was started in the background with its stdout captured. This is the process that runs `documents.tasks.consume_file` and therefore emits every consumer/OCR log line (`docker/supervisord.conf` runs the same command as `[program:scheduler]`):

```bash
$ cd src && nohup ../venv/bin/python manage.py qcluster > /tmp/blitzy_evidence/qcluster.stdout 2>&1 &
$ head -3 /tmp/blitzy_evidence/qcluster.stdout
05:28:13 [Q] INFO Q Cluster utah-florida-three-seven starting.
05:28:13 [Q] INFO Process-1:1 ready for work at 81637
05:28:13 [Q] INFO Process-1:2 ready for work at 81638
```

### gunicorn webserver (ASGI)

The **gunicorn webserver** was started next — the same command `docker/supervisord.conf` uses for `[program:gunicorn]`. It binds `0.0.0.0:8000` with the `paperless.workers.ConfigurableWorker` (an uvicorn worker), per `gunicorn.conf.py`:

```bash
$ cd src && nohup ../venv/bin/gunicorn -c ../gunicorn.conf.py paperless.asgi:application > /tmp/blitzy_evidence/gunicorn.stdout 2>&1 &
$ head -4 /tmp/blitzy_evidence/gunicorn.stdout
[2026-07-08 05:28:13 +0000] [81612] [INFO] Starting gunicorn 20.1.0
[2026-07-08 05:28:13 +0000] [81612] [INFO] Listening at: http://0.0.0.0:8000 (81612)
[2026-07-08 05:28:13 +0000] [81612] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 05:28:13 +0000] [81612] [INFO] Server is ready. Spawning workers
```

The API was then reachable:

```bash
$ curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/api/
HTTP 200
```

### Pre-state (empty before) and pre-OCR text-layer inspection

Immediately before the upload the document count was zero and the three media directories were empty:

```bash
$ cd src && ../venv/bin/python manage.py shell -c \
    "from documents.models import Document; from django.contrib.auth.models import User; \
     print('DOC_COUNT=', Document.objects.count()); \
     print('USERS=', list(User.objects.values_list('username','is_superuser')))"
DOC_COUNT= 0
USERS= [('consumer', False), ('admin', True)]
```

To prove OCR is genuinely required (not skipped for a born-digital PDF), the fixture's text layer was inspected **before** OCR — both with `pdftotext` and with the application's own extractor (`pdfminer.six`, the exact function paperless calls at `src/paperless_tesseract/parsers.py:122`):

```bash
$ pdftotext paperless_tesseract/tests/samples/multi-page-images.pdf - | wc -c   # bytes of extractable text layer
3
$ pdftotext paperless_tesseract/tests/samples/multi-page-images.pdf - | cat -A | head   # the (word-free) content
^L^L^L
$ ../venv/bin/python -c "from pdfminer.high_level import extract_text; \
    t=(extract_text('paperless_tesseract/tests/samples/multi-page-images.pdf') or '').strip(); \
    print('pdfminer stripped length =', len(t)); \
    print('original_has_text (len>50 per parsers.py:236) =', bool(t and len(t) > 50)); \
    print('repr(first 80) =', repr(t[:80]))"
pdfminer stripped length = 0
original_has_text (len>50 per parsers.py:236) = False
repr(first 80) = ''
```

The only bytes `pdftotext` produces are the three form-feed (`^L`) page separators — there are **no words** — and pdfminer returns an empty string, so `original_has_text` computes to `False` at `src/paperless_tesseract/parsers.py:236`. That is exactly the condition under which OCRmyPDF must run (Q2).

### Why both processes are needed

The gunicorn webserver alone demonstrates only Q1 (it enqueues the task and returns immediately). The consumer/OCR log lines for Q2, the filesystem artifacts for Q3, and the database row for Q4 are all produced by the `qcluster` worker executing `documents.tasks.consume_file:184` → `Consumer.try_consume_file` (`src/documents/consumer.py:180`). The `[program:consumer]` folder-watcher (`document_consumer`) is **not** on the REST upload path and was not relied upon.

---

## Q1 — Immediate HTTP response

**Direct answer.** The upload returns **`HTTP/1.1 200 OK`** with the JSON body **`"OK"`** (the four bytes `"OK"`, quotes included). The response is returned **immediately**, before the document is processed, because the upload is asynchronous.

First an auth token was obtained (DRF token auth); both the password and the real token value are redacted here (`<REDACTED>` / `<TOKEN>`):

```bash
$ curl -sS -X POST -F "username=admin" -F "password=<REDACTED>" http://localhost:8000/api/token/
{"token":"<TOKEN>"}
```

Then the mandatory image-based multi-page PDF was POSTed to the real endpoint `api/documents/post_document/` (routed at `src/paperless/urls.py:57-59`, name `post_document`). The full, unedited response — status line, headers, and body:

```bash
$ curl -sS -i -H "Authorization: Token <TOKEN>" \
    -F "document=@src/paperless_tesseract/tests/samples/multi-page-images.pdf" \
    http://localhost:8000/api/documents/post_document/
HTTP/1.1 200 OK
date: Wed, 08 Jul 2026 05:29:10 GMT
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
2. writes the bytes to a scratch temp file `NamedTemporaryFile(prefix="paperless-upload-", dir=settings.SCRATCH_DIR, delete=False)` (`src/documents/views.py:512-519`);
3. generates `task_id = str(uuid.uuid4())` (`:521`) and enqueues the work with `async_task("documents.tasks.consume_file", temp_filename, …)` (`:523`) onto the django-q/Redis broker;
4. returns **`return Response("OK")`** (`:535`).

Because step 4 runs synchronously right after the *enqueue* (not after processing), the client receives `200 OK` while the worker has not yet touched the document. DRF renders the Python string `"OK"` as a JSON string, so the wire body is the quoted token `"OK"` — exactly the 4 bytes reported by `content-length: 4`. This matches the read-only corroboration test `test_upload` in `src/documents/tests/test_api.py`, which POSTs a sample PDF to the same route and asserts `status_code == 200` with `async_task` called once.

**Stability.** The immediate response is deterministic and independent of whether the document will ultimately be accepted. Both the duplicate re-upload (Q2, F6) and the second successful run (Q2, "Repeatability") returned the identical `HTTP 200` with `content-length 4`:

```bash
$ curl -sS -o /dev/null -w "HTTP %{http_code} | content-length %{size_download}\n" \
    -H "Authorization: Token <TOKEN>" \
    -F "document=@src/paperless_tesseract/tests/samples/multi-page-images.pdf" \
    http://localhost:8000/api/documents/post_document/
HTTP 200 | content-length 4
```

---

## Q2 — Processing stage logs & OCRmyPDF parameters

**Direct answer.** As the `qcluster` worker consumes the document, it emits an ordered set of stage log lines under the `paperless.consumer`, `paperless.parsing`, `paperless.parsing.tesseract` and `paperless.classifier` loggers, ending with `Document … consumption finished`. Immediately before the OCR call, `RasterisedDocumentParser` logs the complete OCRmyPDF keyword-argument dict. The full, unedited block captured from `data/log/paperless.log` for the primary run (pk=3):

```text
[2026-07-08 05:29:11,830] [INFO] [paperless.consumer] Consuming multi-page-images.pdf
[2026-07-08 05:29:11,831] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 05:29:11,834] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 05:29:11,836] [DEBUG] [paperless.consumer] Parsing multi-page-images.pdf...
[2026-07-08 05:29:11,857] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-guh5v_d6
[2026-07-08 05:29:11,903] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-guh5v_d6', 'output_file': '/tmp/paperless/paperless-xd151v1g/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-xd151v1g/sidecar.txt'}
[2026-07-08 05:29:14,114] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-08 05:29:14,114] [DEBUG] [paperless.consumer] Generating thumbnail for multi-page-images.pdf...
[2026-07-08 05:29:14,118] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-xd151v1g/archive.pdf[0] /tmp/paperless/paperless-xd151v1g/convert.png
[2026-07-08 05:29:14,192] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 05:29:14,332] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-xd151v1g/gs_out.png /tmp/paperless/paperless-xd151v1g/convert_gs.png
[2026-07-08 05:29:14,396] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-xd151v1g/convert_gs.png -out /tmp/paperless/paperless-xd151v1g/thumb_optipng.png
[2026-07-08 05:29:14,737] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 05:29:14,740] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 05:29:14,761] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-guh5v_d6
[2026-07-08 05:29:14,789] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-xd151v1g
[2026-07-08 05:29:14,790] [INFO] [paperless.consumer] Document 2026-07-08 multi-page-images consumption finished
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
| 5 | `[paperless.parsing.tesseract] Extracted text from PDF file …` (DEBUG) | emitted inside `RasterisedDocumentParser.extract_text` at `src/paperless_tesseract/parsers.py:122`; called from `parse` at `:235` (`text_original = self.extract_text(None, document_path)`), whose result feeds the `original_has_text` check at `:236` |
| 6 | `[paperless.parsing.tesseract] Calling OCRmyPDF with args: {…}` (DEBUG) | `src/paperless_tesseract/parsers.py:260`, immediately before `ocrmypdf.ocr(**args)` at `:261` |
| 7 | `[paperless.parsing.tesseract] Using text from sidecar file` (DEBUG) | emitted inside `extract_text` at `src/paperless_tesseract/parsers.py:107`; called after OCR from `parse` at `:264` |
| 8 | `[paperless.consumer] Generating thumbnail for multi-page-images.pdf...` (DEBUG) | `src/documents/consumer.py:263` |
| 9 | `[paperless.parsing] Execute: convert … archive.pdf[0] … convert.png` (DEBUG) | base `DocumentParser` thumbnail helper (logger `paperless.parsing`, `src/documents/parsers.py:287`) — ImageMagick attempt |
| 10 | `[paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript…` (WARNING) | base parser thumbnail fallback (see note below) |
| 11 | `[paperless.parsing] Execute: convert … gs_out.png … convert_gs.png` (DEBUG) | ghostscript-fallback thumbnail rasterization |
| 12 | `[paperless.parsing.tesseract] Execute: optipng … thumb_optipng.png` (DEBUG) | thumbnail PNG optimization |
| 13 | `[paperless.classifier] Document classification model does not exist (yet)…` (DEBUG) | the classifier — no trained model, so no auto tag/type matching |
| 14 | `[paperless.consumer] Saving record to database` (DEBUG) | `Consumer._store` → `src/documents/consumer.py:387` |
| 15 | `[paperless.consumer] Deleting file /tmp/paperless/paperless-upload-…` (DEBUG) | cleanup of the scratch upload temp file (`src/documents/consumer.py:349`) |
| 16 | `[paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-…` (DEBUG) | cleanup of the parser tempdir |
| 17 | `[paperless.consumer] Document 2026-07-08 multi-page-images consumption finished` (INFO) | `src/documents/consumer.py:373` |

The trailing text `2026-07-08 multi-page-images` in line 17 is the model's `str(document)` (`"{created:%Y-%m-%d} {title}"`); it confirms the parsed `title` (`multi-page-images`) and `created` date (`2026-07-08`) that Q4 shows as stored columns.

Lines 9–12 show that thumbnail generation first tried ImageMagick and **fell back to Ghostscript** after ImageMagick's PDF policy blocked it (the `WARNING … Check your /etc/ImageMagick-x/policy.xml!`). This is an observed, environment-specific fallback within the thumbnailer; the thumbnail was still produced (see Q3). *(The specific ImageMagick-policy cause is inferred from the warning text, not separately verified.)*

### The OCRmyPDF invocation parameters

The complete, unedited dict logged at `src/paperless_tesseract/parsers.py:260` (the values below are the **real captured** ones — `jobs` and the tempdir paths are runtime values, not placeholders):

```json
{
  "input_file": "/tmp/paperless/paperless-upload-guh5v_d6",
  "output_file": "/tmp/paperless/paperless-xd151v1g/archive.pdf",
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
  "sidecar": "/tmp/paperless/paperless-xd151v1g/sidecar.txt"
}
```

The dict is assembled by `RasterisedDocumentParser.construct_ocrmypdf_parameters` (`src/paperless_tesseract/parsers.py:135`). Each key, its source setting, the parser line that adds it, and its meaning:

| Key = observed value | Source setting (`src/paperless/settings.py`) | Added at (`src/paperless_tesseract/parsers.py`) | Meaning (OCRmyPDF 13.4.3) |
|----------------------|----------------------------------------------|--------------------------------------------------|----------------------------|
| `input_file` = `/tmp/paperless/paperless-upload-guh5v_d6` | — (the scratch upload temp from the view) | base dict `:143` | the PDF handed to OCRmyPDF |
| `output_file` = `/tmp/paperless/paperless-xd151v1g/archive.pdf` | — (parser tempdir) | base dict `:143` | where the OCR'd archive PDF is written |
| `use_threads` = `True` | — (constant) | `:148` | run OCRmyPDF's stages threaded rather than multi-process |
| `jobs` = `11` | `THREADS_PER_WORKER` (computed, `:469`) | `:149` | cap concurrency at N workers; observed **11** on this machine (do not assume — it is CPU-derived) |
| `language` = `eng` | `OCR_LANGUAGE` (`:514`, default `eng`) | `:150` | Tesseract language pack(s) to use; OCRmyPDF assumes English unless told otherwise |
| `output_type` = `pdfa` | `OCR_OUTPUT_TYPE` (`:518`, default `pdfa`) | `:151` | produce a PDF/A file for long-term archiving. On 13.4.3 the library default was also `pdfa`; the default only changed to `auto` in OCRmyPDF **v17.0.0**, so `pdfa` here is both the paperless default and the pinned-library default |
| `progress_bar` = `False` | — (constant) | `:152` | disable the tqdm progress bar (correct for a daemonized worker) |
| `skip_text` = `True` | `OCR_MODE` (`:522`, default `skip`) | `:158` | with skip-text, pages that already carry text are copied through untouched while text-less (scanned/image) pages are OCR'd — normalizing everything to PDF/A regardless of contents |
| `clean` = `True` | `OCR_CLEAN` (`:526`, default `clean`) | `:165` | run **unpaper** to clean scanning artifacts from pages before OCR; it does not alter the final visible output |
| `deskew` = `True` | `OCR_DESKEW` (`:528`, default true) | `:173` | rotate pages scanned at a small skew back to horizontal |
| `rotate_pages` = `True` | `OCR_ROTATE_PAGES` (`:530`, default true) | `:176` | detect the correct cardinal orientation of each page and rotate if wrong |
| `rotate_pages_threshold` = `12.0` | `OCR_ROTATE_PAGES_THRESHOLD` (`:532`, default `12.0`) | `:178` | only rotate when Tesseract's orientation confidence exceeds this value; paperless lowers OCRmyPDF's conservative default to 12.0 |
| `sidecar` = `/tmp/paperless/paperless-xd151v1g/sidecar.txt` | added because `OCR_PAGES == 0` (`:510`) | `:185` (`sidecar_file` computed at `parse` `:250`) | write a companion text file with the recognized OCR text; paperless then reads it back (`Using text from sidecar file`, line 7). Pages that already had text are excluded from the sidecar |

Two keys are notably **absent**, and both absences are correct:

- **No `image_dpi`.** That key is only added for image inputs (`is_image` true, `src/paperless_tesseract/parsers.py:205-209`). The input MIME is `application/pdf` (line 2), so `is_image` is false and no `image_dpi` is set.
- **No `pages`.** The `pages` key is only added when `OCR_PAGES > 0` (`src/paperless_tesseract/parsers.py:181`); the default `OCR_PAGES=0` (`src/paperless/settings.py:510`) means the whole document is processed and a `sidecar` entry is added instead.

The read-only corroboration test `test_ocrmypdf_parameters` (`src/paperless_tesseract/tests/test_parser.py:427`) asserts exactly these `input_file`/`output_file`/`sidecar` keys and the conditional `clean`/`clean_final`/`deskew` flags; `test_multi_page` (`:269`) asserts the parser produces an archive and extracts the "page 1/2/3" text — which Q4's `content` column confirms at runtime.

### The `skip` vs `skip_noarchive` boundary (must be explained)

`RasterisedDocumentParser.parse` contains a shortcut that skips OCRmyPDF **entirely**, but it fires only under a non-default mode. At `src/paperless_tesseract/parsers.py:241` the guard is:

```python
if settings.OCR_MODE == "skip_noarchive" and original_has_text:
    # ... skip calling ocrmypdf, no archive produced
```

`original_has_text` is set at `:236` from `len(text_original) > 50`. Under the **default** `OCR_MODE == "skip"` this branch is **not** taken, so OCRmyPDF is **always** invoked (with `skip_text=True`) and a PDF/A archive is always produced. Combined with the observed `original_has_text = False` (Environment bring-up → pre-OCR inspection), that is why the `Calling OCRmyPDF with args` line (line 6) reliably appears for this image-based input — and it did.

### Fallback path — did it fire?

If the first `ocrmypdf.ocr(**args)` raises, `parse` retries and logs `Fallback: Calling OCRmyPDF with args: {…}` (`src/paperless_tesseract/parsers.py:297`) before a second `ocrmypdf.ocr(**args)` (`:298`). **In this run the fallback did NOT fire** — the captured log contains no `Fallback:` line; the single `Calling OCRmyPDF with args` line (line 6) was followed directly by successful sidecar text extraction (line 7).

### Duplicate-guard edge case (observed)

Immediately after the primary run, a second, identical upload (same bytes → same checksum) was submitted. The webserver still returned the immediate `HTTP 200 | content-length 4` (see Q1 stability), but the worker **rejected** it before any OCR. The only new consumer log line produced was:

```text
[2026-07-08 05:30:48,079] [ERROR] [paperless.consumer] Not consuming multi-page-images.pdf: It is a duplicate.
```

This is emitted by `Consumer.pre_check_duplicate` (`src/documents/consumer.py:102`) via `_fail` (message at `:112`, logged at `:80`), which matches on `Q(checksum=...) | Q(archive_checksum=...)`. The two subclaims — that the document count did **not** change and that **no** second OCR call was made — were verified directly (not assumed). First, the count immediately after the duplicate attempt was still `1`:

```bash
$ ../venv/bin/python manage.py shell -c "from documents.models import Document; print('DOC_COUNT =', Document.objects.count())"
DOC_COUNT = 1
```

Second, a timestamped grep of this session's OCR-invocation and duplicate lines shows the duplicate (`05:30:48`) produced **no** `Calling OCRmyPDF` line — it sits between the run-1 OCR call (`05:29:11`) and the run-2 OCR call (`05:31:45`), with no OCR invocation of its own (the args dicts are abbreviated with `{…}` here only to keep the chronology readable; the full dicts appear above and in Repeatability below):

```bash
$ grep -E "Calling OCRmyPDF with args|It is a duplicate" data/log/paperless.log | grep -E "2026-07-08 05:(29|30|31)"
[2026-07-08 05:29:11,903] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {…}   # run 1 (pk=3)
[2026-07-08 05:30:48,079] [ERROR] [paperless.consumer] Not consuming multi-page-images.pdf: It is a duplicate.   # duplicate -> NO OCR
[2026-07-08 05:31:45,271] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {…}   # run 2 (pk=4)

$ grep "Calling OCRmyPDF with args" data/log/paperless.log | grep -cE "2026-07-08 05:(29|30|31)"
2
```

So the immediate `200 OK` (Q1) is returned regardless — the webserver enqueues and responds before the worker detects the duplicate — and the duplicate produced exactly **zero** OCRmyPDF invocations while `DOC_COUNT` stayed at `1`. The two OCR invocations in the session are the primary run and the stability run (next), not the duplicate.

### Repeatability — two clean successful OCR runs (R3)

To confirm the scale/config-derived values are stable, a **second successful** OCR run was performed. Because the same file would otherwise be rejected as a duplicate, the primary document (pk=3) was first deleted via the real API destroy path (see **Cleanup confirmation**, which also frees its checksum), then the identical file was uploaded again, producing a new document (pk=4). Its `Calling OCRmyPDF with args` line:

```text
[2026-07-08 05:31:45,271] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-l7irefmw', 'output_file': '/tmp/paperless/paperless-tgju0h4j/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-tgju0h4j/sidecar.txt'}
```

A direct dict comparison of the two runs shows every functional argument is **identical**, and the only differences are the per-run `mkdtemp` temp paths and the primary key:

```text
STABLE across both runs (identical value):
  use_threads            = True
  jobs                   = 11
  language               = 'eng'
  output_type            = 'pdfa'
  progress_bar           = False
  skip_text              = True
  clean                  = True
  deskew                 = True
  rotate_pages           = True
  rotate_pages_threshold = 12.0
VARYING (per-run mkdtemp temp paths only):
  input_file  : run1='/tmp/paperless/paperless-upload-guh5v_d6'      run2='/tmp/paperless/paperless-upload-l7irefmw'
  output_file : run1='/tmp/paperless/paperless-xd151v1g/archive.pdf' run2='/tmp/paperless/paperless-tgju0h4j/archive.pdf'
  sidecar     : run1='/tmp/paperless/paperless-xd151v1g/sidecar.txt' run2='/tmp/paperless/paperless-tgju0h4j/sidecar.txt'

pk run1 = 3 | pk run2 = 4  (pk varies)
```

**What is stable vs. what varies.** The `jobs` value (`11`, from `THREADS_PER_WORKER`, CPU-derived on this machine) and every OCR flag are stable across the two identical runs, because they are derived entirely from the `OCR_*` settings in `construct_ocrmypdf_parameters`. The values expected to vary are the random `mkdtemp` tempdir paths in `input_file`/`output_file`/`sidecar` and the autoincrement `pk` (3 → 4); the archive **byte content/checksum** also varies run-to-run (Q4 note) because PDF/A output embeds timestamps, while the **original** checksum is invariant (same input bytes).

---

## Q3 — Generated archive & thumbnail filenames

**Direct answer.** With the default (empty) `PAPERLESS_FILENAME_FORMAT`, every stored file is named from the zero-padded 7-digit primary key. For the observed `pk=3` the media area contains:

- original: `media/documents/originals/0000003.pdf`
- **archive PDF: `media/documents/archive/0000003.pdf`**
- **thumbnail: `media/documents/thumbnails/0000003.png`**

**Before** the upload, all three media directories were empty:

```bash
$ ls -la media/documents/originals media/documents/archive media/documents/thumbnails
=== BEFORE: media/documents/originals ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
=== BEFORE: media/documents/archive ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
=== BEFORE: media/documents/thumbnails ===
total 8
drwxr-xr-x 2 root root 4096 Jul  8 04:57 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
```

**After** `consumption finished`, each directory holds exactly one file:

```bash
$ ls -la media/documents/originals media/documents/archive media/documents/thumbnails
=== AFTER: media/documents/originals ===
total 156
drwxr-xr-x 2 root root   4096 Jul  8 05:29 .
drwxr-xr-x 5 root root   4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 150479 Jul  8 05:29 0000003.pdf
=== AFTER: media/documents/archive ===
total 32
drwxr-xr-x 2 root root  4096 Jul  8 05:29 .
drwxr-xr-x 5 root root  4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 24207 Jul  8 05:29 0000003.pdf
=== AFTER: media/documents/thumbnails ===
total 12
drwxr-xr-x 2 root root 4096 Jul  8 05:29 .
drwxr-xr-x 5 root root 4096 Jul  8 04:04 ..
-rw-r--r-- 1 root root 2421 Jul  8 05:29 0000003.png
```

`file(1)` on the three artifacts, confirming what they are (the original is the 3-page input; the archive is a PDF produced by OCRmyPDF; the thumbnail is a 500-px-wide grayscale PNG):

```bash
$ file media/documents/originals/0000003.pdf media/documents/archive/0000003.pdf media/documents/thumbnails/0000003.png
media/documents/originals/0000003.pdf:  PDF document, version 1.7, 3 page(s)
media/documents/archive/0000003.pdf:    PDF document, version 1.7
media/documents/thumbnails/0000003.png: PNG image data, 500 x 647, 8-bit grayscale, non-interlaced
```

The archive is a **PDF/A-2b** file (from `output_type='pdfa'`), verified by reading its XMP identity block both with `pikepdf` and with a raw grep of the archive:

```bash
$ ../venv/bin/python -c "import pikepdf; pdf=pikepdf.open('media/documents/archive/0000003.pdf'); \
    m=pdf.open_metadata(); \
    print('pdfaid:part        =', m.get('pdfaid:part')); \
    print('pdfaid:conformance =', m.get('pdfaid:conformance'))"
pdfaid:part        = 2
pdfaid:conformance = B

$ grep -ao 'pdfaid:part="[0-9]*"\|pdfaid:conformance="[A-Z]*"' media/documents/archive/0000003.pdf | sort -u
pdfaid:conformance="B"
pdfaid:part="2"
```

So `pdfaid:part="2"` and `pdfaid:conformance="B"` confirm a **PDF/A-2b** archive.

**Cause → effect.**

- **The directories** come from `src/paperless/settings.py`: `MEDIA_ROOT` (`:61`), `ORIGINALS_DIR` = `media/documents/originals` (`:62`), `ARCHIVE_DIR` = `media/documents/archive` (`:63`), `THUMBNAIL_DIR` = `media/documents/thumbnails` (`:64`).
- **The `.pdf` names** come from `generate_filename` (`src/documents/file_handling.py:128`). Because `PAPERLESS_FILENAME_FORMAT` is `None` by default (`src/paperless/settings.py:584`; guard at `file_handling.py:132`), the custom-format branch is skipped and the function falls through to `counter_str = f"_{counter:02}" if counter else ""` (`:186`), `filetype_str = ".pdf" if archive_filename else doc.file_type` (`:188`), and finally `filename = f"{doc.pk:07}{counter_str}{filetype_str}"` (`:193`). With `pk=3` and no name collision, `counter_str` is empty, so both the original and the archive resolve to `0000003.pdf`. The consumer assigns these via `generate_unique_filename` at `src/documents/consumer.py:316` (original) and `:328` (archive, `archive_filename=True`).
- **The thumbnail name** comes from the `thumbnail_path` property (`src/documents/models.py:273`), which joins `THUMBNAIL_DIR` with `"{:07}.png".format(self.pk)` → `0000003.png`.

`counter_str` is empty here because there was no filename collision; it would become `_01`, `_02`, … only if a same-named file already existed. *(That collision branch is not exercised in this run — inferred from `file_handling.py:186`, not observed.)*

---

## Q4 — Persisted database fields

**Direct answer.** The `documents_document` row (SQLite, `src/paperless/settings.py:300`) stores **15 real columns**. For the processed document (pk=3) their observed values are:

| Column (ORM field) | DB column | Observed value |
|--------------------|-----------|----------------|
| `id` (pk) | `id` INTEGER | `3` |
| `title` | `title` varchar(128) | `'multi-page-images'` |
| `content` | `content` TEXT | `'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'` |
| `mime_type` | `mime_type` varchar(256) | `'application/pdf'` |
| `checksum` | `checksum` varchar(32) | `'62acb0bcbfbcaa62ca6ad3668e4e404b'` (MD5 of the original) |
| `archive_checksum` | `archive_checksum` varchar(32) | `'68b9098d4fcc6fc45147cb8dfbc3bd5e'` (MD5 of the archive) |
| `created` | `created` datetime | `2026-07-08 05:29:11+00:00` |
| `modified` | `modified` datetime | `2026-07-08 05:29:14.761453+00:00` |
| `added` | `added` datetime | `2026-07-08 05:29:14.741450+00:00` |
| `storage_type` | `storage_type` varchar(11) | `'unencrypted'` |
| `filename` | `filename` varchar(1024) | `'0000003.pdf'` |
| `archive_filename` | `archive_filename` varchar(1024) | `'0000003.pdf'` |
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
id = 3
correspondent = None
title = 'multi-page-images'
document_type = None
content = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
mime_type = 'application/pdf'
checksum = '62acb0bcbfbcaa62ca6ad3668e4e404b'
archive_checksum = '68b9098d4fcc6fc45147cb8dfbc3bd5e'
created = datetime.datetime(2026, 7, 8, 5, 29, 11, tzinfo=datetime.timezone.utc)
modified = datetime.datetime(2026, 7, 8, 5, 29, 14, 761453, tzinfo=datetime.timezone.utc)
storage_type = 'unencrypted'
added = datetime.datetime(2026, 7, 8, 5, 29, 14, 741450, tzinfo=datetime.timezone.utc)
filename = '0000003.pdf'
archive_filename = '0000003.pdf'
archive_serial_number = None
--- COMPUTED PROPERTIES (NOT columns) ---
source_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/originals/0000003.pdf
archive_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/archive/0000003.pdf
thumbnail_path = /tmp/blitzy/paperless-ngx/blitzy-f925031a-cd29-41f8-8c33-c6b05064f8a2_6cc5ba/src/../media/documents/thumbnails/0000003.png
```

To prove these are genuine table columns (and to read the byte-sensitive checksums straight from the stored row rather than recomputing them), the raw row was dumped with Python's `sqlite3` module (the `sqlite3` CLI is not installed in this image):

```bash
$ ./venv/bin/python -c "import sqlite3; c=sqlite3.connect('data/db.sqlite3').cursor(); \
    c.execute('SELECT * FROM documents_document ORDER BY id DESC LIMIT 1'); \
    cols=[d[0] for d in c.description]; \
    [print(k,'=',repr(v)) for k,v in zip(cols,c.fetchone())]"
id = 3
title = 'multi-page-images'
content = 'This is a multi page document. Page 1.\nThis is a multi page document. Page 2.\nThis is a multi page document. Page 3.'
created = '2026-07-08 05:29:11'
modified = '2026-07-08 05:29:14.761453'
correspondent_id = None
checksum = '62acb0bcbfbcaa62ca6ad3668e4e404b'
added = '2026-07-08 05:29:14.741450'
storage_type = 'unencrypted'
archive_serial_number = None
document_type_id = None
mime_type = 'application/pdf'
archive_checksum = '68b9098d4fcc6fc45147cb8dfbc3bd5e'
archive_filename = '0000003.pdf'
filename = '0000003.pdf'
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
$ md5sum media/documents/originals/0000003.pdf media/documents/archive/0000003.pdf
62acb0bcbfbcaa62ca6ad3668e4e404b  media/documents/originals/0000003.pdf
68b9098d4fcc6fc45147cb8dfbc3bd5e  media/documents/archive/0000003.pdf
```

*(The original checksum `62acb0…` is invariant across runs because the input bytes never change; the archive checksum differs from the second run because PDF/A output embeds timestamps — see Q2 "Repeatability".)*

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

**Direct answer.** Both test documents (pk=3 from the primary run, pk=4 from the second/stability run) and every runtime artifact they produced were removed, the temporary observation scripts/processes were torn down, and the repository was left pristine — `git status` reports no modified tracked files and no added file other than this single documentation file.

**Deletion via the real API destroy path (removes DB row + media files + search-index entry).** Deletion was performed through the canonical `DELETE /api/documents/{id}/` endpoint, i.e. `DocumentViewSet.destroy` (`src/documents/views.py:219`), which first calls `index.remove_document_from_index(self.get_object())` (`src/documents/views.py:222` → `src/documents/index.py:123`) to remove the full-text-search entry, then `super().destroy()` (`src/documents/views.py:223`) deletes the DB row, whose `post_delete` signal handler `cleanup_document_deletion` (`src/documents/signals/handlers.py:234`, registered via `@receiver(post_delete, sender=Document)` at `:233`) unlinks the original/archive/thumbnail files. *(Note: a plain ORM `Document.delete()` would trigger only the `post_delete` media cleanup — it does **not** touch the Whoosh index; index removal is specific to the API/admin destroy paths. That is why the deletion below uses the real API endpoint.)*

The search-index entry was verified present and then absent (Whoosh index at `INDEX_DIR = DATA_DIR/index`, `src/paperless/settings.py:73`). For the primary document (pk=3), deleting it via the API also freed its checksum so the second stability run could proceed:

```bash
# index entry present for pk=3 before delete:
index hits for id=3 BEFORE delete: 1

$ curl -sS -i -X DELETE -H "Authorization: Token <TOKEN>" http://localhost:8000/api/documents/3/ | head -1
HTTP/1.1 204 No Content

index hits for id=3 AFTER delete: 0
$ ../venv/bin/python manage.py shell -c "from documents.models import Document; print('DOC_COUNT =', Document.objects.count())"
DOC_COUNT = 0
```

The second document (pk=4) was removed the same way as the final cleanup step, again proving the index entry is gone and the media files are unlinked:

```bash
index hits for id=4 BEFORE delete: 1
$ curl -sS -i -X DELETE -H "Authorization: Token <TOKEN>" http://localhost:8000/api/documents/4/ | head -1
HTTP/1.1 204 No Content
index hits for id=4 AFTER delete: 0
$ ../venv/bin/python manage.py shell -c "from documents.models import Document; print('DOC_COUNT =', Document.objects.count())"
DOC_COUNT = 0
```

The media directories were empty again afterward, and the Whoosh index held zero documents:

```bash
$ ls media/documents/originals media/documents/archive media/documents/thumbnails   # each empty
$ ../venv/bin/python -c "from django.conf import settings; import django,os; \
    os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); \
    from whoosh.index import open_dir; ix=open_dir(settings.INDEX_DIR); \
    s=ix.searcher(); print('index total docs =', s.doc_count())"
index total docs = 0
```

The scratch upload temp files, the captured worker stdout, and the helper evidence files were removed, and the `qcluster`/`gunicorn` processes started for this investigation were stopped **by the exact PIDs captured at start** (no broad `pkill`):

```bash
$ rm -f /tmp/paperless/paperless-upload-*
$ kill "$(cat /tmp/blitzy_evidence/qcluster.pid)" "$(cat /tmp/blitzy_evidence/gunicorn.pid)"
$ rm -rf /tmp/blitzy_evidence
```

Finally, `git status` confirms the working tree is pristine — the only change is this answer document (runtime data under `media/`, `data/`, and `*.log` is gitignored, so it never dirties tracked files):

```bash
$ git status --porcelain
 M blitzy/documentation/paperless-ngx_542221a38dff.md
```

No tracked source file was modified and no file other than `blitzy/documentation/paperless-ngx_542221a38dff.md` was added, satisfying the read-only mandate.
