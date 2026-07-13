# paperless-ngx — Runtime Behavior of the Document-Ingestion / OCR Pipeline

> **Motivating problem (verbatim intent):** a user testing paperless-ngx locally reported **"inconsistent OCR results"** when processing multi-page PDFs. This document captures the *actual runtime behavior* of the ingestion/OCR pipeline so the four questions below can be answered from **observed output**, not from reading code alone.

## Methodology (read-first, run-first)

This is a **read-only, run-first investigation**. Every answer below was produced by **actually running** paperless-ngx in its **default/canonical configuration** inside the project's own Docker container and driving the **canonical upload entry point** `POST /api/documents/post_document/` with a real superuser and a real multipart `document` field. For every claim you will find the **exact command** and its **complete, unedited output** in a fenced block, plus a `[path:Lx]` citation into the source at HEAD `542221a38`. Statements that could only be obtained by reading the code (never triggered at runtime) are explicitly labelled **inferred (code-read)**. Values obtained outside the canonical entry point are labelled **non-canonical**.

- **Runtime host:** the user-provided container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...qna_1.01` (Debian 11.11, Python 3.9.23), which bundles the native OCR toolchain (tesseract, ghostscript, qpdf, unpaper, pngquant). The repository is bind-mounted at `/app`; the Django project lives under `/app/src`. All commands below were executed inside this container.
- **Task queue is django-q, not Celery.** Ingestion runs **asynchronously in a `qcluster` worker process** (`consume_file`), *not* in the web request [src/documents/tasks.py:L184]. The web request returns before processing begins, so the web server **and** the `qcluster` worker must both run or the Q2 stage logs and the OCRmyPDF invocation never appear.
- **Default store is SQLite** at `DATA_DIR/db.sqlite3` [src/paperless/settings.py:L299-L300]; PostgreSQL is used only when `PAPERLESS_DBHOST` is set [src/paperless/settings.py:L304-L311], which it is not.

### Question map

| # | Question | Answered in |
|---|----------|-------------|
| Q1 | What HTTP status/body does the client receive **immediately** after upload? | [Q1](#q1--immediate-synchronous-http-response) |
| Q2 | What are the processing-stage log patterns, and **what exact parameters** are passed to OCRmyPDF? | [Q2](#q2--processing-stage-logs--ocrmypdf-parameters) |
| Q3 | What do the generated **archive PDF** and **thumbnail** filenames look like in media? | [Q3](#q3--generated-media-filenames) |
| Q4 | Which `Document` fields are **actually stored in the database** vs. computed at runtime? | [Q4](#q4--database-stored-fields-vs-computed-properties) |

---

## Environment setup

### 1. Container & native OCR toolchain

The native binaries OCRmyPDF orchestrates are provided by the image (base `FROM python:3.9-slim-bullseye as main-app` [Dockerfile:L18]; ghostscript [Dockerfile:L41], pngquant [Dockerfile:L61], tesseract-ocr + language packs [Dockerfile:L63-L68], unpaper [Dockerfile:L70], jbig2enc [Dockerfile:L12], qpdf [Dockerfile:L13]).

```console
$ docker exec paperless-setup bash -lc 'whoami; cat /etc/debian_version;
    tesseract --version | head -5; gs --version; qpdf --version | head -2;
    unpaper --version | head -2; pngquant --version | head -2'
root
11.11
tesseract 4.1.1
 leptonica-1.79.0
  libgif 5.1.9 : libjpeg 6b (libjpeg-turbo 2.0.6) : libpng 1.6.37 : libtiff 4.2.0 : zlib 1.2.11 : libwebp 0.6.1 : libopenjp2 2.4.0
 Found AVX512BW
 Found AVX512F
9.53.3
qpdf version 10.1.0
Run qpdf --copyright to see copyright and license information.
6.1
2.12.2 (July 2019)
```

### 2. Python runtime & pinned dependencies

The pins match `requirements.txt`: ocrmypdf 13.4.3 [requirements.txt:L60], Django 4.0.4 [L38], django-q 1.3.9 [L37], djangorestframework 3.13.1 [L39], channels 3.0.4 [L23], channels-redis 3.4.0 [L22], redis 3.5.3 [L84], pikepdf 5.1.1 [L65].

```console
$ docker exec paperless-setup bash -lc 'python3 --version;
    pip show ocrmypdf django django-q djangorestframework channels channels-redis redis pikepdf \
      | grep -E "^Name|^Version|^---"'
Python 3.9.23
Name: ocrmypdf
Version: 13.4.3
---
Name: Django
Version: 4.0.4
---
Name: django-q
Version: 1.3.9
---
Name: djangorestframework
Version: 3.13.1
---
Name: channels
Version: 3.0.4
---
Name: channels-redis
Version: 3.4.0
---
Name: redis
Version: 3.5.3
---
Name: pikepdf
Version: 5.1.1
```

### 3. Default-configuration purity

The three settings that would change the answers are **all unset**, so the code takes its default branches: `PAPERLESS_OCR_MODE` (→ `skip`), `PAPERLESS_FILENAME_FORMAT` (→ `None`), `PAPERLESS_DBHOST` (→ SQLite). The only `PAPERLESS_*` variable present is `PAPERLESS_DISABLE_DBHANDLER=true`, which is **not referenced** by the `LOGGING` config (the `file_paperless` handler is unconditional [src/paperless/settings.py:L392-L395] and the `paperless` logger is DEBUG [src/paperless/settings.py:L409]), so file logging — the basis for Q2 — is unaffected.

```console
$ docker exec paperless-setup bash -lc 'env | grep -i "PAPERLESS_\|DJANGO_SETTINGS" | sort'
PAPERLESS_DISABLE_DBHANDLER=true

$ docker exec paperless-setup bash -lc 'for v in PAPERLESS_OCR_MODE PAPERLESS_FILENAME_FORMAT PAPERLESS_DBHOST; do
    printf "%s=[%s]\n" "$v" "${!v-<UNSET>}"; done'
PAPERLESS_OCR_MODE=[<UNSET>]
PAPERLESS_FILENAME_FORMAT=[<UNSET>]
PAPERLESS_DBHOST=[<UNSET>]
```

### 4. Bring up the stack (Redis → migrate → superuser → web server → worker)

Default broker/Channels backend is `PAPERLESS_REDIS=redis://localhost:6379` [paperless.conf.example:L10], satisfied by a local `redis-server`. Migrations were already applied (the DB pre-exists), and the `admin` superuser already exists from environment setup — the ORM confirms it is a superuser and that `admin:admin` authenticates (HTTP Basic is a default DRF auth class [src/paperless/settings.py:L117-L120], so `-u admin:admin` is a canonical authentication method).

```console
$ docker exec paperless-setup bash -lc 'redis-cli ping; cd /app/src && python3 manage.py migrate | tail -6'
PONG
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.

$ docker exec paperless-setup bash -lc 'cd /app/src &&
    DJANGO_SUPERUSER_PASSWORD=admin python3 manage.py createsuperuser --noinput \
      --username admin --email admin@example.com; echo "exit=$?";
    python3 manage.py shell -c "from django.contrib.auth.models import User;
      print([(u.username,u.is_superuser) for u in User.objects.all()])";
    python3 manage.py shell -c "from django.contrib.auth import authenticate;
      u=authenticate(username=\"admin\", password=\"admin\"); print(\"AUTH_OK\" if u else \"AUTH_FAIL\")"'
CommandError: Error: That username is already taken.
exit=1
[('consumer', False), ('admin', True)]
AUTH_OK
```

The Django web server and the **mandatory** `django-q` worker (`qcluster`) were started fresh, each detached with output teed to a log file. `runserver` is invoked with `--noreload` (canonical flag; identical request behavior) so the autoreloader does not restart the process when this `.md` is written into the watched `/app` tree. Startup banners:

```console
$ docker exec -d paperless-setup bash -lc 'cd /app/src && exec python3 manage.py runserver 0.0.0.0:8000 --noreload > /tmp/runserver.log 2>&1'
$ docker exec paperless-setup bash -lc 'cat /tmp/runserver.log'
Performing system checks...

System check identified no issues (0 silenced).
July 13, 2026 - 17:08:38
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.

$ docker exec -d paperless-setup bash -lc 'cd /app/src && exec python3 manage.py qcluster > /tmp/qcluster.log 2>&1'
$ docker exec paperless-setup bash -lc 'cat /tmp/qcluster.log'
17:08:38 [Q] INFO Q Cluster colorado-colorado-papa-potato starting.
17:08:38 [Q] INFO Process-1:1 ready for work at 22403
17:08:38 [Q] INFO Process-1:2 ready for work at 22404
17:08:38 [Q] INFO Process-1:3 ready for work at 22405
17:08:38 [Q] INFO Process-1:4 ready for work at 22406
17:08:38 [Q] INFO Process-1:5 ready for work at 22407
17:08:38 [Q] INFO Process-1:6 ready for work at 22408
17:08:38 [Q] INFO Process-1:7 ready for work at 22409
17:08:38 [Q] INFO Process-1:8 ready for work at 22410
17:08:38 [Q] INFO Process-1:9 ready for work at 22411
17:08:38 [Q] INFO Process-1:10 ready for work at 22412
17:08:38 [Q] INFO Process-1:11 ready for work at 22413
17:08:38 [Q] INFO Process-1:12 monitoring at 22414
17:08:38 [Q] INFO Process-1 guarding cluster colorado-colorado-papa-potato
17:08:38 [Q] INFO Process-1:13 pushing tasks at 22415
17:08:38 [Q] INFO Q Cluster colorado-colorado-papa-potato running.
```

`qcluster` came up with **11 worker processes** (Process-1:1..11) — this is the value that later appears as OCRmyPDF's `jobs` parameter (see Q2).

### 5. Runtime settings the pipeline will use (printed, not assumed)

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.conf import settings
print(\"MEDIA_ROOT       =\", settings.MEDIA_ROOT)
print(\"ORIGINALS_DIR    =\", settings.ORIGINALS_DIR)
print(\"ARCHIVE_DIR      =\", settings.ARCHIVE_DIR)
print(\"THUMBNAIL_DIR    =\", settings.THUMBNAIL_DIR)
print(\"LOGGING_DIR      =\", settings.LOGGING_DIR)
print(\"SCRATCH_DIR      =\", settings.SCRATCH_DIR)
print(\"OCR_MODE         =\", repr(settings.OCR_MODE))
print(\"OCR_LANGUAGE     =\", repr(settings.OCR_LANGUAGE))
print(\"OCR_OUTPUT_TYPE  =\", repr(settings.OCR_OUTPUT_TYPE))
print(\"OCR_CLEAN        =\", repr(settings.OCR_CLEAN))
print(\"OCR_DESKEW       =\", repr(settings.OCR_DESKEW))
print(\"OCR_ROTATE_PAGES =\", repr(settings.OCR_ROTATE_PAGES))
print(\"OCR_ROTATE_PAGES_THRESHOLD =\", repr(settings.OCR_ROTATE_PAGES_THRESHOLD))
print(\"OCR_PAGES        =\", repr(settings.OCR_PAGES))
print(\"OCR_USER_ARGS    =\", repr(settings.OCR_USER_ARGS))
print(\"OCR_IMAGE_DPI    =\", repr(settings.OCR_IMAGE_DPI))
print(\"THREADS_PER_WORKER =\", repr(settings.THREADS_PER_WORKER))
print(\"PAPERLESS_FILENAME_FORMAT =\", repr(settings.PAPERLESS_FILENAME_FORMAT))
print(\"DB ENGINE        =\", settings.DATABASES[\"default\"][\"ENGINE\"])
print(\"DB NAME          =\", settings.DATABASES[\"default\"][\"NAME\"])"'
MEDIA_ROOT       = /app/src/../media
ORIGINALS_DIR    = /app/src/../media/documents/originals
ARCHIVE_DIR      = /app/src/../media/documents/archive
THUMBNAIL_DIR    = /app/src/../media/documents/thumbnails
LOGGING_DIR      = /app/src/../data/log
SCRATCH_DIR      = /tmp/paperless
OCR_MODE         = 'skip'
OCR_LANGUAGE     = 'eng'
OCR_OUTPUT_TYPE  = 'pdfa'
OCR_CLEAN        = 'clean'
OCR_DESKEW       = True
OCR_ROTATE_PAGES = True
OCR_ROTATE_PAGES_THRESHOLD = 12.0
OCR_PAGES        = 0
OCR_USER_ARGS    = '{}'
OCR_IMAGE_DPI    = None
THREADS_PER_WORKER = 11
PAPERLESS_FILENAME_FORMAT = None
DB ENGINE        = django.db.backends.sqlite3
DB NAME          = /app/src/../data/db.sqlite3
```

These confirm the defaults cited throughout: `OCR_MODE='skip'` [src/paperless/settings.py:L522], `OCR_LANGUAGE='eng'` [L514], `OCR_OUTPUT_TYPE='pdfa'` [L518], `OCR_CLEAN='clean'` [L526], `OCR_DESKEW=True` [L528], `OCR_ROTATE_PAGES=True` [L530], `OCR_ROTATE_PAGES_THRESHOLD=12.0` [L532-L533], `OCR_PAGES=0` [L510], `OCR_USER_ARGS='{}'` [L541], `THREADS_PER_WORKER=11` [L469], `PAPERLESS_FILENAME_FORMAT=None` [L584], SQLite backend [L299-L300].


### 6. Fixture that forces OCR (multi-page, image-only PDF)

Under the default `OCR_MODE='skip'`, OCRmyPDF is invoked with `skip_text=True`, which **copies text-bearing pages through unchanged, OCRing only pages with no text layer** [src/paperless_tesseract/parsers.py:L157-L158]. To guarantee the OCR path is exercised on **every** page, the fixture is a **3-page, image-only PDF** (rasterized text, no text layer). It was generated with a temporary Pillow script (`/tmp/make_fixture.py`, deleted during cleanup):

```python
# /tmp/make_fixture.py  (temporary; removed in cleanup)
from PIL import Image, ImageDraw, ImageFont
FONT = "/usr/share/fonts/truetype/liberation/LiberationSans-Regular.ttf"
font_big = ImageFont.truetype(FONT, 56)
font_med = ImageFont.truetype(FONT, 40)
pages = []
for i in range(3):
    im = Image.new("RGB", (1240, 1754), "white")   # ~150 DPI A4
    d = ImageDraw.Draw(im)
    d.text((90, 120), "Canonical OCR test document", font=font_big, fill="black")
    d.text((90, 320), f"Page {i + 1} of 3", font=font_med, fill="black")
    d.text((90, 460), "The quick brown fox jumps over", font=font_med, fill="black")
    d.text((90, 560), "the lazy dog. 1234567890", font=font_med, fill="black")
    d.text((90, 720), f"Unique marker line page {i + 1}: PAPERLESSOCR", font=font_med, fill="black")
    pages.append(im)
pages[0].save("/tmp/ocr_test.pdf", save_all=True, append_images=pages[1:], resolution=150.0)
print("WROTE /tmp/ocr_test.pdf pages=", len(pages))
```

Proof it is **image-only** (no extractable text layer, no embedded fonts) and multi-page. `pdftotext` returns only 3 bytes (page-break characters), which is far below the 50-character threshold at `parse()` [src/paperless_tesseract/parsers.py:L234-L236], so `original_has_text=False` and OCR runs on every page:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 /tmp/make_fixture.py; ls -l /tmp/ocr_test.pdf'
WROTE /tmp/ocr_test.pdf pages= 3
-rw-r--r-- 1 root root 213603 Jul 13 17:10 /tmp/ocr_test.pdf

$ docker exec paperless-setup bash -lc '
    echo -n "chars = "; pdftotext /tmp/ocr_test.pdf - 2>/dev/null | wc -c
    pdffonts /tmp/ocr_test.pdf
    pdfinfo /tmp/ocr_test.pdf | grep -E "Pages|Page size"
    cd /app/src && python3 -c "import magic; print(magic.from_file(\"/tmp/ocr_test.pdf\", mime=True))"'
chars = 3
name                                 type              encoding         emb sub uni object ID
------------------------------------ ----------------- ---------------- --- --- --- ---------
Pages:          3
Page size:      595.2 x 841.92 pts (A4)
application/pdf
```

(The `pdffonts` table is empty — no embedded fonts — confirming a pure image PDF; `python-magic` detects `application/pdf`, the mime the consumer sees.)

---

## Q1 — Immediate (synchronous) HTTP response

> **Question:** What HTTP response status and body does the client receive **immediately** after submitting the upload (i.e., the synchronous API response, before any asynchronous processing completes)?

**Answer: HTTP `200 OK` with the JSON body `"OK"` (4 bytes).** This is the **asynchronous enqueue acknowledgement** returned by `PostDocumentView.post()`; it is emitted **before** any worker stage log appears.

**Code path.** The route `^documents/post_document/` under the `^api/` prefix binds to `PostDocumentView.as_view()` [src/paperless/urls.py:L57-L59; prefix L40; import L16]. `class PostDocumentView(GenericAPIView)` [src/documents/views.py:L491] uses `permission_classes=(IsAuthenticated,)` [L493], `serializer_class=PostDocumentSerializer` [L494], `parser_classes=(MultiPartParser,)` [L495]. `post()` [L497] validates the serializer [L499-L500], writes the upload to a `paperless-upload-*` temp file in `SCRATCH_DIR` [L512-L516], enqueues `async_task("documents.tasks.consume_file", ...)` [L523-L534], and finally **`return Response("OK")`** [src/documents/views.py:L535]. The multipart contract is `PostDocumentSerializer(serializers.Serializer)` [src/documents/serialisers.py:L413] with a required `document = serializers.FileField(...)` [L415] and optional `title`/`correspondent`/`document_type`/`tags` [L420,L426,L434,L442]; the endpoint is documented in [docs/api.rst:L233-L235].

**Command & complete output** (note the log-line delta and the surrounding client clock):

```console
$ docker exec paperless-setup bash -lc '
    LOG=/app/data/log/paperless.log
    before=$(wc -l < "$LOG"); echo "lines_before=$before"
    date "+%Y-%m-%d %H:%M:%S.%3N"
    curl -sS -u admin:admin -F "document=@/tmp/ocr_test.pdf" \
      -w "\n---\nHTTP_STATUS:%{http_code}\nTIME_TOTAL:%{time_total}s\nSIZE_DOWNLOAD:%{size_download}bytes\n" \
      http://localhost:8000/api/documents/post_document/
    date "+%Y-%m-%d %H:%M:%S.%3N"
    after=$(wc -l < "$LOG"); echo "lines_after_immediate=$after (delta=$((after-before)))"'
lines_before=2660
2026-07-13 17:11:28.205
"OK"
---
HTTP_STATUS:200
TIME_TOTAL:0.125369s
SIZE_DOWNLOAD:4bytes
2026-07-13 17:11:28.340
lines_after_immediate=2660 (delta=0)
```

The body is literally the four bytes `"OK"` (a JSON-encoded string). Inspected raw (on an equivalent canonical upload):

```console
$ docker exec paperless-setup bash -lc '
    curl -sS -u admin:admin -F "document=@/tmp/ocr_test.pdf" -o /tmp/ok_body.bin \
      -w "HTTP_STATUS:%{http_code}\n" http://localhost:8000/api/documents/post_document/
    od -c /tmp/ok_body.bin; wc -c < /tmp/ok_body.bin'
HTTP_STATUS:200
0000000   "   O   K   "
0000004
4
```

**Why this is the async ack, not the result.** At the instant `curl` returned (client clock `17:11:28.340`), the paperless log line count was **unchanged** (`2660 → 2660`, delta `0`): **no worker stage log had yet been written**. The worker's first stage line (`Consuming ocr_test.pdf`) is timestamped `17:11:28,489` — *after* the response was received. The `200/"OK"` therefore corresponds to `async_task(...)` + `return Response("OK")` [src/documents/views.py:L523-L535]; the processing (Q2) happens later in the `qcluster` worker (`consume_file` [src/documents/tasks.py:L184]). It must **not** be conflated with processing completion.


---

## Q2 — Processing-stage logs & OCRmyPDF parameters

> **Question:** As the document is processed, what are the key log-line patterns that show it moving through the pipeline's stages, and specifically **what exact parameters are passed to the OCRmyPDF invocation** (every parameter enumerated)?

**Logging setup.** The verbose formatter is `"[{asctime}] [{levelname}] [{name}] {message}"` [src/paperless/settings.py:L378]; the `paperless` logger writes at DEBUG to `LOGGING_DIR/paperless.log` via a `ConcurrentRotatingFileHandler` [src/paperless/settings.py:L392-L395,L409], while the root logger writes to the console [src/paperless/settings.py:L407]. Pipeline loggers are `paperless.consumer`, `paperless.parsing` [src/documents/parsers.py:L40,L287], and `paperless.parsing.tesseract` [src/paperless_tesseract/parsers.py:L24]. Every record additionally carries a per-document `group` correlation uuid via `LoggingMixin.log()` (`extra={"group": self.logging_group}`, uuid4 from `renew_logging_group`) [src/documents/loggers.py:L11-L12,L14,L21] — but the default verbose format string does **not** include `{group}`, so it is attached to the `LogRecord` yet **not rendered** in `paperless.log`.

### Q2a — Ordered stage log lines

Tailing the new lines the worker wrote for the Q1 upload (from line 2661):

```console
$ docker exec paperless-setup bash -lc 'tail -n +2661 /app/data/log/paperless.log'
[2026-07-13 17:11:28,489] [INFO] [paperless.consumer] Consuming ocr_test.pdf
[2026-07-13 17:11:28,490] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 17:11:28,490] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 17:11:28,493] [DEBUG] [paperless.consumer] Parsing ocr_test.pdf...
[2026-07-13 17:11:28,515] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-liwhy831
[2026-07-13 17:11:28,591] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-liwhy831', 'output_file': '/tmp/paperless/paperless-aca568bs/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-aca568bs/sidecar.txt'}
[2026-07-13 17:11:32,349] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 17:11:32,350] [DEBUG] [paperless.consumer] Generating thumbnail for ocr_test.pdf...
[2026-07-13 17:11:32,354] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-aca568bs/archive.pdf[0] /tmp/paperless/paperless-aca568bs/convert.png
[2026-07-13 17:11:33,182] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-aca568bs/convert.png -out /tmp/paperless/paperless-aca568bs/thumb_optipng.png
[2026-07-13 17:11:34,838] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 17:11:34,841] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 17:11:34,869] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-liwhy831
[2026-07-13 17:11:34,895] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-aca568bs
[2026-07-13 17:11:34,896] [INFO] [paperless.consumer] Document 2026-07-13 ocr_test consumption finished
```

Mapping each observed line to the code, in order (all inside `Consumer.try_consume_file()` [src/documents/consumer.py:L180] unless noted):

| Observed log line | Emitted by | Citation |
|-------------------|-----------|----------|
| `Consuming ocr_test.pdf` | `self.log("info", f"Consuming {self.filename}")` | [src/documents/consumer.py:L215] |
| `Detected mime type: application/pdf` | `self.log("debug", f"Detected mime type: {mime_type}")` | [src/documents/consumer.py:L221] |
| `Parser: RasterisedDocumentParser` | `self.log("debug", f"Parser: {type(document_parser).__name__}")` | [src/documents/consumer.py:L246] |
| `Parsing ocr_test.pdf...` | `self.log("debug", "Parsing {}...".format(self.filename))` | [src/documents/consumer.py:L260] |
| `Extracted text from PDF file ...` | pre-extraction in `parse()` → `extract_text` logs it | [src/paperless_tesseract/parsers.py:L122; parse L234] |
| `Calling OCRmyPDF with args: {...}` | `self.log("debug", f"Calling OCRmyPDF with args: {args}")` | [src/paperless_tesseract/parsers.py:L260] |
| `Using text from sidecar file` | `extract_text()` (sidecar had no `"[OCR skipped on page"`) | [src/paperless_tesseract/parsers.py:L107] |
| `Generating thumbnail for ocr_test.pdf...` | `self.log("debug", f"Generating thumbnail for {self.filename}...")` | [src/documents/consumer.py:L263] |
| `Execute: convert ...` / `Execute: optipng ...` | thumbnail render/optimize | [src/documents/parsers.py:L143] |
| `Saving record to database` | `self.log("debug", "Saving record to database")` in `_store()` | [src/documents/consumer.py:L387] |
| `Document ... consumption finished` | `self.log("info", "Document {} consumption finished".format(document))` | [src/documents/consumer.py:L373] |

**Progress milestones are not in this log.** The `STARTING`/`WORKING`/`SUCCESS` milestones are published by `_send_progress()` to the Channels/WebSocket layer (e.g. `_send_progress(0,100,"STARTING",MESSAGE_NEW_FILE)` [src/documents/consumer.py:L202], `WORKING`+`MESSAGE_PARSING_DOCUMENT` [L259], `WORKING`+`MESSAGE_GENERATING_THUMBNAIL` [L264], `MESSAGE_SAVE_DOCUMENT` [L294], `SUCCESS`+`MESSAGE_FINISHED` [L375]) — they are **not** written to `paperless.log`. The message constants are defined at [src/documents/consumer.py:L43-L49].

**Worker task return value.** `consume_file` [src/documents/tasks.py:L184] calls `Consumer().try_consume_file(...)` [L236] and returns `"Success. New document id {} created"` [src/documents/tasks.py:L247]. django-q stores this as the task result:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Task
for t in Task.objects.filter(func=\"documents.tasks.consume_file\").order_by(\"-started\")[:3]:
    print(repr(t.result), \"| success=\", t.success)"'
'Success. New document id 1 created' | success= True
'Success. New document id 2 created' | success= True
'Success. New document id 1 created' | success= True

$ docker exec paperless-setup bash -lc 'tail -n 6 /tmp/qcluster.log'
17:11:28 [Q] INFO Process-1:1 processing [ocr_test.pdf]
[2026-07-13 17:11:28,489] [INFO] [paperless.consumer] Consuming ocr_test.pdf
[2026-07-13 17:11:34,896] [INFO] [paperless.consumer] Document 2026-07-13 ocr_test consumption finished
17:11:34 [Q] INFO Process-1:1 stopped doing work
17:11:34 [Q] INFO Processed [ocr_test.pdf]
17:11:35 [Q] INFO recycled worker Process-1:1
```

(The first shown result — `New document id 1 created` — is the most recent, i.e. this investigation's run; the `id 2`/`id 1` entries below it are earlier environment-setup runs.)


### Q2b — The OCRmyPDF invocation, every parameter enumerated

The args dict is built by `RasterisedDocumentParser.construct_ocrmypdf_parameters()` [src/paperless_tesseract/parsers.py:L135], logged verbatim by `parse()` [src/paperless_tesseract/parsers.py:L230] at the line `Calling OCRmyPDF with args: {...}` [src/paperless_tesseract/parsers.py:L260], immediately before `ocrmypdf.ocr(**args)` [src/paperless_tesseract/parsers.py:L261]. `parse()` also sets `os.environ["OMP_THREAD_LIMIT"]="1"` [src/paperless_tesseract/parsers.py:L232].

For a default-config, multi-page, **image-only PDF** (mime `application/pdf`), the observed dict has **exactly 13 keys**:

```python
{'input_file': '/tmp/paperless/paperless-upload-liwhy831',
 'output_file': '/tmp/paperless/paperless-aca568bs/archive.pdf',
 'use_threads': True,
 'jobs': 11,
 'language': 'eng',
 'output_type': 'pdfa',
 'progress_bar': False,
 'skip_text': True,
 'clean': True,
 'deskew': True,
 'rotate_pages': True,
 'rotate_pages_threshold': 12.0,
 'sidecar': '/tmp/paperless/paperless-aca568bs/sidecar.txt'}
```

| # | Parameter | Observed value | Meaning / why present | Citation |
|---|-----------|----------------|-----------------------|----------|
| 1 | `input_file` | `/tmp/paperless/paperless-upload-liwhy831` | the scratch temp file written by the upload view | [src/paperless_tesseract/parsers.py:L144] |
| 2 | `output_file` | `/tmp/paperless/paperless-aca568bs/archive.pdf` | OCRmyPDF's archive output (`<tempdir>/archive.pdf`) | [src/paperless_tesseract/parsers.py:L145; L249] |
| 3 | `use_threads` | `True` | use threads (workers are daemonized by django-q) | [src/paperless_tesseract/parsers.py:L148] |
| 4 | `jobs` | `11` | `settings.THREADS_PER_WORKER` (observed 11) | [src/paperless_tesseract/parsers.py:L149; src/paperless/settings.py:L469] |
| 5 | `language` | `'eng'` | `settings.OCR_LANGUAGE` | [src/paperless_tesseract/parsers.py:L150; src/paperless/settings.py:L514] |
| 6 | `output_type` | `'pdfa'` | `settings.OCR_OUTPUT_TYPE` → Ghostscript PDF/A archive | [src/paperless_tesseract/parsers.py:L151; src/paperless/settings.py:L518] |
| 7 | `progress_bar` | `False` | disable the console progress bar | [src/paperless_tesseract/parsers.py:L152] |
| 8 | `skip_text` | `True` | from `OCR_MODE=='skip'`: **OCR only pages without text; copy text pages through** | [src/paperless_tesseract/parsers.py:L157-L158; src/paperless/settings.py:L522] |
| 9 | `clean` | `True` | from `OCR_CLEAN=='clean'`: unpaper cleans the **image before OCR only** (not the final image) | [src/paperless_tesseract/parsers.py:L164-L165; src/paperless/settings.py:L526] |
| 10 | `deskew` | `True` | from `OCR_DESKEW` (and mode≠redo): straighten skewed pages before OCR | [src/paperless_tesseract/parsers.py:L172-L173; src/paperless/settings.py:L528] |
| 11 | `rotate_pages` | `True` | from `OCR_ROTATE_PAGES`: auto-correct page orientation | [src/paperless_tesseract/parsers.py:L175-L176; src/paperless/settings.py:L530] |
| 12 | `rotate_pages_threshold` | `12.0` | `settings.OCR_ROTATE_PAGES_THRESHOLD` confidence threshold for rotation | [src/paperless_tesseract/parsers.py:L177-L179; src/paperless/settings.py:L532-L533] |
| 13 | `sidecar` | `/tmp/paperless/paperless-aca568bs/sidecar.txt` | plain-text OCR output; added because `OCR_PAGES==0` takes the `else` branch (`sidecar` is incompatible with `pages`) | [src/paperless_tesseract/parsers.py:L181-L185; src/paperless/settings.py:L510] |

**Two edge facts, explicitly:**

- **`image_dpi` is omitted for a PDF.** It is added **only for image mime types** [src/paperless_tesseract/parsers.py:L187-L215], where `is_image()` is true only for `image/png|jpeg|tiff|bmp|gif` [src/paperless_tesseract/parsers.py:L64-L70]. A PDF is not an image mime, so **`image_dpi` is absent** — confirmed by its absence from the captured dict above.
- **`OCR_USER_ARGS` adds nothing.** Its default is the string `"{}"` [src/paperless/settings.py:L541]; `construct_ocrmypdf_parameters()` merges `json.loads(...)` → an empty dict, contributing no keys [src/paperless_tesseract/parsers.py:L217-L220].

Also absent (mode-dependent): `force_ocr`/`redo_ocr` are only set for `OCR_MODE in {force, redo}` or the safe fallback [src/paperless_tesseract/parsers.py:L155-L160]; under the default `skip` only `skip_text` is set.

### Q2c — Fallback invocation (force_ocr): **inferred (code-read)** — did not fire

If the primary `ocrmypdf.ocr()` raises `NoTextFoundException` or `InputFileError` [src/paperless_tesseract/parsers.py:L276], the consumer retries with `safe_fallback=True`, which sets **`force_ocr=True`** (`OCR_MODE=="force" or safe_fallback`) [src/paperless_tesseract/parsers.py:L155-L156], logs **`Fallback: Calling OCRmyPDF with args: {...}`** [src/paperless_tesseract/parsers.py:L297] and calls `ocrmypdf.ocr(**args)` again [L298] (using `archive-fallback.pdf`/`sidecar-fallback.txt` [L283-L284]; `force_ocr` replaces `skip_text`). **In this run the fallback did not fire** — the primary invocation succeeded (the sidecar yielded text, no `NoTextFoundException`), so there is **no** `Fallback: Calling OCRmyPDF` line:

```console
$ docker exec paperless-setup bash -lc 'grep -c "Fallback: Calling OCRmyPDF with args" /app/data/log/paperless.log'
0
```

The `force_ocr` fallback path above is therefore labelled **inferred (code-read)**, cited to the lines shown; it was not exercised by the image-only fixture because that fixture succeeds on the primary `skip_text` call.

### Q2d — Run-to-run behavior on the SAME input (reproducing the "inconsistency" faithfully)

Per the rules, the reported run-to-run inconsistency is probed by uploading the **same, unchanged** fixture repeatedly (4× total) rather than a variant. Every upload returns the same synchronous `200/"OK"`, but the **worker deduplicates by MD5 checksum**: the first creates the document, and every subsequent identical upload is rejected. Duplicate detection is `pre_check_duplicate()` [src/documents/consumer.py:L102-L112] (md5 [L104]; `Q(checksum=...) | Q(archive_checksum=...)` [L106]; `_fail(..., "Not consuming {filename}: It is a duplicate.")` [L111-L112]), called from `try_consume_file` [L213].

```console
$ docker exec paperless-setup bash -lc '
    curl -sS -u admin:admin -F "document=@/tmp/ocr_test.pdf" -w " HTTP:%{http_code}\n" http://localhost:8000/api/documents/post_document/
    sleep 1
    curl -sS -u admin:admin -F "document=@/tmp/ocr_test.pdf" -w " HTTP:%{http_code}\n" http://localhost:8000/api/documents/post_document/
    sleep 6
    tail -n +2676 /app/data/log/paperless.log'
"OK" HTTP:200
"OK" HTTP:200
[2026-07-13 17:14:54,644] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
[2026-07-13 17:15:11,826] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
[2026-07-13 17:15:12,948] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
```

**Observed distribution for the identical file:** 1 success + 3 duplicate rejections — i.e. **deterministic**, no run-to-run OCR variance. To show the OCR *parameters* themselves are stable across *different* documents, a second, distinct image-only PDF (different checksum → new document, pk=2) was uploaded; its `Calling OCRmyPDF with args` line is **identical except for the necessarily-unique temp paths** (same `jobs=11`, `skip_text=True`, etc.):

```console
$ docker exec paperless-setup bash -lc 'grep "Calling OCRmyPDF with args" /app/data/log/paperless.log | tail -1'
[2026-07-13 17:15:51,921] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-hr0qwmh1', 'output_file': '/tmp/paperless/paperless-xnzpxkkd/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-xnzpxkkd/sidecar.txt'}
```


---

## Q3 — Generated media filenames

> **Question:** After processing finishes, what do the generated **archive PDF** and **thumbnail** filenames look like inside the media storage area?

**Answer (observed, canonical document pk=1): archive PDF = `archive/0000001.pdf`, thumbnail = `thumbnails/0000001.png`** (original = `originals/0000001.pdf`). The default naming is `{pk:07}` (the primary key zero-padded to 7 digits), `.pdf` for original/archive and `.png` for the thumbnail.

```console
$ docker exec paperless-setup bash -lc '
    echo "--- originals ---"; ls -l /app/media/documents/originals
    echo "--- archive ---";   ls -l /app/media/documents/archive
    echo "--- thumbnails ---";ls -l /app/media/documents/thumbnails'
--- originals ---
total 420
-rw-r--r-- 1 root root 213603 Jul 13 17:11 0000001.pdf
-rw-r--r-- 1 root root 209077 Jul 13 17:15 0000002.pdf
--- archive ---
total 228
-rw-r--r-- 1 root root 116354 Jul 13 17:11 0000001.pdf
-rw-r--r-- 1 root root 114128 Jul 13 17:15 0000002.pdf
--- thumbnails ---
total 76
-rw-r--r-- 1 root root 37493 Jul 13 17:11 0000001.png
-rw-r--r-- 1 root root 33618 Jul 13 17:15 0000002.png
```

(`0000001.*` is the canonical Q1 upload; `0000002.*` is the distinct second fixture from Q2d — both removed in cleanup.)

**Code path.** Because `PAPERLESS_FILENAME_FORMAT` is `None` [src/paperless/settings.py:L584], `generate_filename()` skips the custom-format branch [src/documents/file_handling.py:L131] and takes its default branch `filename = f"{doc.pk:07}{counter_str}{filetype_str}"` [src/documents/file_handling.py:L193], where `counter_str=""` (counter is 0) [L186] and `filetype_str=".pdf" if archive_filename else doc.file_type` [L188] (and `doc.file_type` is `".pdf"` for a PDF). The consumer assigns these inside `transaction.atomic()`: `document.filename = generate_unique_filename(document)` [src/documents/consumer.py:L315] and `document.archive_filename = generate_unique_filename(document, archive_filename=True)` [src/documents/consumer.py:L327-L330]. The thumbnail name comes from the computed property `Document.thumbnail_path`, `file_name = "{:07}.png".format(self.pk)` joined to `THUMBNAIL_DIR` [src/documents/models.py:L274,L278]. The post-save `update_filename_and_move_files` handler [src/documents/signals/handlers.py:L311-L312] is a **no-op** under default config (regenerated name equals current name → no move).

The stored column values line up exactly with the on-disk names, and demonstrate the relative-vs-absolute split (see Q4):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
d=Document.objects.get(pk=1)
print(\"filename(stored)        =\", repr(d.filename))
print(\"archive_filename(stored)=\", repr(d.archive_filename))
print(\"source_path(computed)   =\", d.source_path)
print(\"archive_path(computed)  =\", d.archive_path)
print(\"thumbnail_path(computed)=\", d.thumbnail_path)"'
filename(stored)        = '0000001.pdf'
archive_filename(stored)= '0000001.pdf'
source_path(computed)   = /app/src/../media/documents/originals/0000001.pdf
archive_path(computed)  = /app/src/../media/documents/archive/0000001.pdf
thumbnail_path(computed)= /app/src/../media/documents/thumbnails/0000001.png
```

---

## Q4 — Database-stored fields vs computed properties

> **Question:** In the document's database record, which fields are **actually stored in the database** (as opposed to filesystem-only computed metadata), and what values appear for the processed document?

**Answer.** The `documents_document` table has **15 physical columns**; several `Document` attributes commonly mistaken for stored data (`source_path`, `archive_path`, `thumbnail_path`, `file_type`, `has_archive_version`, and the `*_file` openers) are **`@property` methods computed at runtime and are not columns**. The many-to-many `tags` field is stored in a **separate through-table**, not as a column.

**Stored columns (from the ORM, with observed values for pk=1):**

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
d=Document.objects.get(pk=1)
for f in d._meta.get_fields():
    if hasattr(f,\"attname\") and not f.many_to_many and not f.one_to_many:
        print(f\"  {f.attname:24s} = {getattr(d,f.attname)!r}\")
print(\"  tags (M2M, separate table) =\", list(d.tags.values_list(\"name\", flat=True)))"'
  id                       = 1
  correspondent_id         = None
  title                    = 'ocr_test'
  document_type_id         = None
  content                  = 'Canonical OCR test document\n\nPage 1 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 1: PAPERLESSOCR\nCanonical OCR test document\n\nPage 2 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 2: PAPERLESSOCR\nCanonical OCR test document\n\nPage 3 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 3: PAPERLESSOCR'
  mime_type                = 'application/pdf'
  checksum                 = '71995c96af58bcf2b742754bb344026e'
  archive_checksum         = 'b2aed16f13256cdd3c15ea6dcb434e55'
  created                  = datetime.datetime(2026, 7, 13, 17, 11, 28, tzinfo=datetime.timezone.utc)
  modified                 = datetime.datetime(2026, 7, 13, 17, 11, 34, 868929, tzinfo=datetime.timezone.utc)
  storage_type             = 'unencrypted'
  added                    = datetime.datetime(2026, 7, 13, 17, 11, 34, 842991, tzinfo=datetime.timezone.utc)
  filename                 = '0000001.pdf'
  archive_filename         = '0000001.pdf'
  archive_serial_number    = None
  tags (M2M, separate table) = []
```

**Concrete DB schema (SQLite `PRAGMA table_info`):**

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import sqlite3
c=sqlite3.connect(\"/app/src/../data/db.sqlite3\")
print(c.execute(\"SELECT sql FROM sqlite_master WHERE name=\x27documents_document\x27\").fetchone()[0][:120])
print(\"--- columns ---\")
for r in c.execute(\"PRAGMA table_info(documents_document)\"): print(r[1], \"|\", r[2], \"| notnull=\", r[3])"'
CREATE TABLE "documents_document" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "title" varchar(128) NOT NULL, "content
--- columns ---
id | integer | notnull= 1
title | varchar(128) | notnull= 1
content | text | notnull= 1
created | datetime | notnull= 1
modified | datetime | notnull= 1
correspondent_id | integer | notnull= 0
checksum | varchar(32) | notnull= 1
added | datetime | notnull= 1
storage_type | varchar(11) | notnull= 1
archive_serial_number | integer | notnull= 0
document_type_id | integer | notnull= 0
mime_type | varchar(256) | notnull= 1
archive_checksum | varchar(32) | notnull= 0
archive_filename | varchar(1024) | notnull= 0
filename | varchar(1024) | notnull= 0
```

Mapping each column to its model field [src/documents/models.py] (FK fields carry the `_id` suffix at the DB layer):

| DB column | Model field | Citation |
|-----------|-------------|----------|
| `id` | implicit `AutoField` primary key | — |
| `correspondent_id` | `correspondent` (FK) | [src/documents/models.py:L97] |
| `title` | `title` | [src/documents/models.py:L106] |
| `document_type_id` | `document_type` (FK) | [src/documents/models.py:L108] |
| `content` | `content` (OCR text) | [src/documents/models.py:L117] |
| `mime_type` | `mime_type` | [src/documents/models.py:L126] |
| `checksum` | `checksum` (unique) | [src/documents/models.py:L135] |
| `archive_checksum` | `archive_checksum` (null) | [src/documents/models.py:L143] |
| `created` | `created` | [src/documents/models.py:L152] |
| `modified` | `modified` (`auto_now`) | [src/documents/models.py:L154] |
| `storage_type` | `storage_type` | [src/documents/models.py:L161] |
| `added` | `added` | [src/documents/models.py:L169] |
| `filename` | `filename` (`FilePathField`, null) | [src/documents/models.py:L176] |
| `archive_filename` | `archive_filename` (`FilePathField`, null) | [src/documents/models.py:L186] |
| `archive_serial_number` | `archive_serial_number` (null) | [src/documents/models.py:L196] |
| *(separate table)* | `tags` (ManyToMany) | [src/documents/models.py:L128] |

**Computed `@property` attributes — NOT stored** (they compose filesystem paths / derive values at runtime):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
d=Document.objects.get(pk=1)
print(\"file_type      =\", repr(d.file_type))
print(\"source_path    =\", d.source_path)
print(\"archive_path   =\", d.archive_path)
print(\"thumbnail_path =\", d.thumbnail_path)
print(\"has_archive_version =\", d.has_archive_version)"'
file_type      = '.pdf'
source_path    = /app/src/../media/documents/originals/0000001.pdf
archive_path   = /app/src/../media/documents/archive/0000001.pdf
thumbnail_path = /app/src/../media/documents/thumbnails/0000001.png
has_archive_version = True
```

| Computed property | Derivation | Citation |
|-------------------|-----------|----------|
| `source_path` | join `ORIGINALS_DIR` + `filename` | [src/documents/models.py:L222-L231] |
| `source_file` | `open(self.source_path, "rb")` | [src/documents/models.py:L233-L235] |
| `has_archive_version` | `archive_filename is not None` | [src/documents/models.py:L237-L239] |
| `archive_path` | join `ARCHIVE_DIR` + `archive_filename` | [src/documents/models.py:L241-L246] |
| `archive_file` | `open(self.archive_path, "rb")` | [src/documents/models.py:L248-L250] |
| `file_type` | `get_default_file_extension(mime_type)` | [src/documents/models.py:L268-L270] |
| `thumbnail_path` | join `THUMBNAIL_DIR` + `"{:07}.png".format(pk)` | [src/documents/models.py:L272-L278] |
| `thumbnail_file` | `open(self.thumbnail_path, "rb")` | [src/documents/models.py:L280-L282] |

**Key distinction (demonstrated above for the same pk=1 document):** `filename` and `archive_filename` are stored as **relative path strings** (`'0000001.pdf'`), whereas `source_path`/`archive_path`/`thumbnail_path` build **absolute** paths at runtime by joining `ORIGINALS_DIR`/`ARCHIVE_DIR`/`THUMBNAIL_DIR` — they are never persisted. The `content` column holds the OCR text (visibly containing the page-1/2/3 markers, confirming every page was OCR'd), and both `checksum` (of the original) and `archive_checksum` (of the OCR'd archive) are stored MD5 hashes.


---

## Synthesis — why OCR looks "inconsistent"

**Observed facts (from the runs above):**

- The pipeline invokes OCRmyPDF with **`skip_text=True`** under the default `OCR_MODE='skip'` [src/paperless_tesseract/parsers.py:L157-L158; src/paperless/settings.py:L522]. This is the single most consequential parameter for the user's report.
- On the fully **image-only** fixture, OCR ran on **every** page: the log shows `Using text from sidecar file` (not `Incomplete sidecar file: discarding.`) [src/paperless_tesseract/parsers.py:L107,L110], and the stored `content` contains the markers for **pages 1, 2 and 3** (Q4). So when *no* page has a text layer, coverage is complete and consistent.
- The OCRmyPDF **parameters themselves are deterministic** across documents (Q2d): the args dict is byte-for-byte identical except for the unavoidably-unique temp paths. So "inconsistency" does **not** come from varying parameters run to run.
- For an **identical** input file, behavior is deterministic in the other direction too: exactly one document is created and every re-upload is deduplicated by checksum [src/documents/consumer.py:L102-L112]. There is no run-to-run OCR variance for the same bytes.

**Inferred explanation (grounded in the observed `skip_text=True`):** `skip_text` instructs OCRmyPDF to **skip pages that already contain text and copy them through unchanged, OCRing only pages without a text layer**. The sidecar (from which paperless reads the text) therefore **only contains text for the pages that were actually OCR'd** — this is stated directly in the code comment at [src/paperless_tesseract/parsers.py:L104-L106] ("The sidecar file will only contain text for OCR'ed pages."). Consequently, on a **mixed / partially-text multi-page PDF** — the realistic case behind the user's report — pages that already carry a (possibly poor or partial) text layer are **not** re-OCR'd, while text-free pages are. The result is **uneven OCR coverage across pages of the same document**, which reads to a user as "inconsistent OCR results." This is a property of the default `skip` mode, not of randomness in the engine.

This is an **inferred** mechanism (the mixed-PDF case was not the fixture used here, precisely because the rules require reproducing the *reported* condition with a faithful input rather than constructing a variant); it is grounded in the **observed** `skip_text=True` parameter and the code's own sidecar semantics. The alternative modes that would change this are non-default and out of scope to enable: `redo` (`redo_ocr=True`, strips & rewrites the text layer) and `force` (`force_ocr=True`, rasterizes & OCRs every page) [src/paperless_tesseract/parsers.py:L155-L160]; the latter is exactly what the safe **fallback** uses [src/paperless_tesseract/parsers.py:L155-L156,L297].

---

## Coverage-pass checklist

| Item | Answered | Evidence |
|------|----------|----------|
| **Q1** — immediate HTTP **status** | ✅ `200` | curl `-w HTTP_STATUS:%{http_code}` → `200` |
| **Q1** — immediate HTTP **body** | ✅ `"OK"` (4 bytes) | curl body + `od -c` → `" O K "` |
| **Q1** — it is the **async ack** (precedes stage logs) | ✅ | log line count delta `0` at response time; first stage log timestamped later |
| **Q2** — ordered stage log lines | ✅ | full `tail` excerpt + per-line citation table |
| **Q2** — `Calling OCRmyPDF with args: {...}` captured verbatim | ✅ | log line 2666 quoted in full |
| **Q2** — all **13** primary params enumerated | ✅ | `input_file, output_file, use_threads, jobs=11, language, output_type, progress_bar, skip_text, clean, deskew, rotate_pages, rotate_pages_threshold, sidecar` |
| **Q2** — `image_dpi` omitted for PDF (noted) | ✅ | absent from dict; [src/paperless_tesseract/parsers.py:L64-L70,L187-L215] |
| **Q2** — `OCR_USER_ARGS={}` adds nothing (noted) | ✅ | [src/paperless/settings.py:L541; src/paperless_tesseract/parsers.py:L217-L220] |
| **Q2** — fallback (`force_ocr`) captured or inferred | ✅ inferred | `grep -c` → `0`; labelled inferred with [src/paperless_tesseract/parsers.py:L155-L156,L276,L297-L298] |
| **Q2** — worker return string | ✅ | `Success. New document id 1 created` (django-q `Task.result`) |
| **Q2** — run-to-run distribution (same input) | ✅ | 1 success + 3 duplicate rejections; deterministic |
| **Q3** — archive filename | ✅ `archive/0000001.pdf` | `ls -l` + generate_filename [src/documents/file_handling.py:L193] |
| **Q3** — thumbnail filename | ✅ `thumbnails/0000001.png` | `ls -l` + `thumbnail_path` [src/documents/models.py:L274,L278] |
| **Q4** — every stored DB column + values | ✅ | ORM dump + `PRAGMA table_info` (15 columns) |
| **Q4** — every computed `@property` listed | ✅ | property table [src/documents/models.py:L222-L282] |
| **Q4** — relative-stored vs absolute-computed distinction | ✅ | side-by-side `filename` vs `source_path` for pk=1 |
| **Synthesis** — `skip_text` → uneven coverage | ✅ | observed `skip_text=True` + sidecar comment [src/paperless_tesseract/parsers.py:L104-L106] |
| **Environment setup** — exact build/run/invocation commands | ✅ | container/versions/redis/migrate/superuser/runserver/qcluster/settings/fixture, all with output |

**Labelling discipline:** every factual claim about the system carries a `[path:Lx]`/`[path:Lx-Ly]` locator valid at HEAD `542221a38`; every observed value is shown as command + complete unedited output; the only **inferred (code-read)** items are the `force_ocr` fallback path (not triggered) and the mixed-PDF "inconsistency" mechanism (grounded in the observed `skip_text=True`). No value here was obtained from a bypassing interface, so nothing is labelled non-canonical.


---

## Cleanup confirmation

Per the read-only + cleanup mandate, every runtime artifact created during this investigation was removed and the source tree was verified **byte-for-byte clean**. The exact teardown commands and their complete output follow.

**1. Delete the created `Document` rows.** The ORM `delete()` fires the `post_delete` receiver `cleanup_document_deletion` [src/documents/signals/handlers.py:L233-L234], which unlinks the original / archive / thumbnail files as a side effect.

```
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
print(\"before:\", list(Document.objects.all().values_list(\"pk\",\"title\")))
print(\"delete() ->\", Document.objects.all().delete())
print(\"after:\", list(Document.objects.all().values_list(\"pk\",\"title\")))"'
before: [(2, 'ocr_test2'), (1, 'ocr_test')]
delete() -> (2, {'documents.Document': 2})
after: []
```

**2. Confirm the media directories are empty** (the cascade in step 1 removed `0000001.*` and `0000002.*`).

```
$ docker exec paperless-setup bash -lc 'for d in originals archive thumbnails; do echo "-- $d --"; ls -A /app/media/documents/$d; done'
-- originals --
-- archive --
-- thumbnails --
```

**3. Remove the test fixtures, temporary scripts, scratch upload leftovers, and observation-capture files.** (The two pre-existing `paperless-*` scratch directories from before this session were left untouched.)

```
$ docker exec paperless-setup bash -lc 'rm -f /tmp/paperless/paperless-upload-* \
    /tmp/make_fixture.py /tmp/make_fixture2.py /tmp/ocr_test.pdf /tmp/ocr_test2.pdf \
    /tmp/ok_body.bin /tmp/qcluster.log /tmp/runserver.log'
# host-side observation captures + the Phase-9 backup:
$ rm -rf /tmp/cap /tmp/deliverable.bak.md
```

**4. Stop the `runserver` and `qcluster` processes started for this investigation.** Redis, which was already running from environment setup, was **left as found** (not started by this investigation). The container lacks `ps`/`pkill`, so processes were enumerated via `/proc/<pid>/cmdline` and terminated by exact PID; the post-kill re-check excludes the enumerating shell itself.

```
$ docker exec paperless-setup bash -lc '... enumerate /proc; kill -TERM PIDs whose cmdline matches "manage.py runserver"/"manage.py qcluster" ...'
targeting PIDs: 22368 22376 22402 22412 22413 22414 22415 22696 22744 22766 22785 22796 22869 22916 22917 22979
# after SIGTERM + 4s settle, re-check (current shell excluded), port, and redis:
NONE — runserver and qcluster fully stopped
:8000 CLOSED (via /proc/net/tcp)
PONG        # redis left as found
```

**5. Final `git status` on the source checkout — the tree is byte-for-byte clean; the only new path is this deliverable.**

```
$ git rev-parse --abbrev-ref HEAD
blitzy-87b095e2-5299-40e2-89c5-c0b45c0c4edc
$ git rev-parse --short HEAD
542221a38
$ git diff --stat
                       # (empty output — zero tracked files modified)
$ git status --porcelain=v1 --untracked-files=all
?? blitzy/documentation/paperless-ngx_542221a38dff.md
```

No existing source file was modified, and no artifact other than `blitzy/documentation/paperless-ngx_542221a38dff.md` remains. The investigation was conducted entirely through the canonical upload API in the default configuration, and the environment has been restored to its pre-test state. ✅

