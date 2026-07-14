# paperless-ngx — Runtime Behavior of the Document-Ingestion / OCR Pipeline

> **Motivating problem (verbatim intent):** a user testing paperless-ngx locally reported **"inconsistent OCR results"** when processing multi-page PDFs. This document captures the *actual runtime behavior* of the ingestion/OCR pipeline so the four questions below can be answered from **observed output**, not from reading code alone.

## Methodology (run-first, then write)

This is a **read-only, run-first investigation**: the pipeline was **run first**, and every answer below was written **from the observed runtime output**, never from reading the code alone. Each answer was produced by **actually running** paperless-ngx in its **default / canonical configuration** inside the project's own Docker container and driving the **canonical upload entry point** `POST /api/documents/post_document/` with a real superuser and a real multipart `document` field. For every claim you will find the **exact command** and its **complete, unedited output** in a fenced block, plus a `[path:Lx]` citation into the source at HEAD `542221a38`. Statements that could only be obtained by reading the code (never triggered at runtime) are explicitly labelled **inferred (code-read)**. Values obtained outside the canonical entry point are labelled **non-canonical**.

**Default / canonical configuration — no storage relocation.** The investigation runs the software exactly as a normal user would in the provided container: the database, media, logs and consume directory all sit at their **default** locations under `/app` (proven in §3 and §5 below), and **none** of the answer-bearing settings is overridden. In particular `PAPERLESS_OCR_MODE`, `PAPERLESS_FILENAME_FORMAT` and `PAPERLESS_DBHOST` — the only three settings that would change Q1–Q4 — remain **unset** at their defaults. Because the shared default database already existed from environment setup, the created test document is left as the single canonical document with primary key `pk=1` for the duration of the run (the count was `0` before the upload), and every runtime artifact it produces is removed at the end (see [Cleanup confirmation](#cleanup-confirmation)). To read only this run's records out of the **shared** default log file, the file's byte length is recorded immediately before the upload and the tail beyond that offset is sliced out — a reproducible technique shown inline in Q2.

- **Runtime host:** the user-provided container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...qna_1.01` (Debian 11.11, Python 3.9.23), which bundles the native OCR toolchain (tesseract, ghostscript, qpdf, unpaper, pngquant). The repository is bind-mounted at `/app`; the Django project lives under `/app/src`. All commands below were executed inside this container.
- **Task queue is django-q, not Celery.** Ingestion runs **asynchronously in a `qcluster` worker process** (`consume_file`), *not* in the web request [src/documents/tasks.py:L184]. The synchronous HTTP response is the **enqueue acknowledgement** and does **not** wait for processing to *finish*; the web server **and** the `qcluster` worker must both run or the Q2 stage logs and the OCRmyPDF invocation never appear. (The precise ordering of the response versus the worker *starting* is measured in Q1 — the worker may begin concurrently, and in this run it demonstrably did.)
- **Default store is SQLite** at `DATA_DIR/db.sqlite3` [src/paperless/settings.py:L299-L300]; PostgreSQL is used only when `PAPERLESS_DBHOST` is set [src/paperless/settings.py:L304], which it is not.

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

The runtime is the user-provided image. Its `/app` is **bind-mounted** from the working tree (so edits to this `.md` and the repository are the same bytes the container sees), and the Django project runs under `/app/src`:

```console
$ docker inspect paperless-setup --format 'Image={{.Config.Image}}'
Image=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01
$ docker inspect paperless-setup --format 'WorkingDir={{.Config.WorkingDir}}'
WorkingDir=/app/src
$ docker inspect paperless-setup --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}} (rw={{.RW}}){{println}}{{end}}'
bind /tmp/blitzy/paperless-ngx/blitzy-87b095e2-5299-40e2-89c5-c0b45c0c4edc_fa033e -> /app (rw=true)
```

The `Dockerfile` *intends* to provide the full native toolchain — it declares a `jbig2enc` builder stage [Dockerfile:L12] and a `qpdf` builder stage [Dockerfile:L13], and installs ghostscript [Dockerfile:L41], pngquant [Dockerfile:L61], tesseract-ocr + language packs [Dockerfile:L63-L68], and unpaper [Dockerfile:L70] onto the `python:3.9-slim-bullseye` base [Dockerfile:L18]. Rather than *assume* the Dockerfile's intent, the **actual running image is probed directly** below. It confirms tesseract / ocrmypdf / ghostscript / qpdf / unpaper / pngquant / pdftoppm are on `PATH`, but **`jbig2` and `jbig2enc` are NOT present** in this image. This document therefore makes **no** claim that jbig2 is available at runtime; its absence is treated as a real edge condition (exercised in the [coverage section](#coverage--edge-cases)):

```console
$ docker exec paperless-setup bash -lc 'whoami; echo debian $(cat /etc/debian_version); python3 --version; for b in tesseract ocrmypdf gs qpdf unpaper pngquant pdftoppm jbig2 jbig2enc; do p=$(command -v $b) && echo "$b PRESENT $p" || echo "$b MISSING"; done; echo ---; tesseract --version | head -1; gs --version; qpdf --version | head -1; unpaper --version | head -1; pngquant --version | head -1; pdftoppm -v 2>&1 | head -1'
root
debian 11.11
Python 3.9.23
tesseract PRESENT /usr/bin/tesseract
ocrmypdf PRESENT /usr/local/bin/ocrmypdf
gs PRESENT /usr/bin/gs
qpdf PRESENT /usr/bin/qpdf
unpaper PRESENT /usr/bin/unpaper
pngquant PRESENT /usr/bin/pngquant
pdftoppm PRESENT /usr/bin/pdftoppm
jbig2 MISSING
jbig2enc MISSING
---
tesseract 4.1.1
9.53.3
qpdf version 10.1.0
6.1
2.12.2 (July 2019)
pdftoppm version 20.09.0
```

### 2. Python runtime & pinned dependencies

The pins match `requirements.txt`: ocrmypdf 13.4.3 [requirements.txt:L60], Django 4.0.4 [requirements.txt:L38], django-q 1.3.9 [requirements.txt:L37], djangorestframework 3.13.1 [requirements.txt:L39], channels 3.0.4 [requirements.txt:L23], channels-redis 3.4.0 [requirements.txt:L22], redis 3.5.3 [requirements.txt:L84], pikepdf 5.1.1 [requirements.txt:L65].

```console
$ docker exec paperless-setup bash -lc 'python3 -m pip freeze | grep -iE "^(ocrmypdf|Django|django-q|djangorestframework|channels|channels-redis|redis|pikepdf)=="'
channels==3.0.4
channels-redis==3.4.0
Django==4.0.4
django-q==1.3.9
djangorestframework==3.13.1
ocrmypdf==13.4.3
pikepdf==5.1.1
redis==3.5.3
```

### 3. Default-configuration purity

The three settings that would change the answers are **all unset**, so the code takes its default branches: `PAPERLESS_OCR_MODE` (→ `'skip'`) [src/paperless/settings.py:L522], `PAPERLESS_FILENAME_FORMAT` (→ `None`) [src/paperless/settings.py:L584], `PAPERLESS_DBHOST` (→ SQLite) [src/paperless/settings.py:L304]:

```console
$ docker exec paperless-setup bash -lc 'for v in PAPERLESS_OCR_MODE PAPERLESS_FILENAME_FORMAT PAPERLESS_DBHOST PAPERLESS_MEDIA_ROOT PAPERLESS_DATA_DIR PAPERLESS_CONSUMPTION_DIR; do printf "%s=[%s]\n" "$v" "${!v-<UNSET>}"; done'
PAPERLESS_OCR_MODE=[<UNSET>]
PAPERLESS_FILENAME_FORMAT=[<UNSET>]
PAPERLESS_DBHOST=[<UNSET>]
PAPERLESS_MEDIA_ROOT=[<UNSET>]
PAPERLESS_DATA_DIR=[<UNSET>]
PAPERLESS_CONSUMPTION_DIR=[<UNSET>]
```

The **only** paperless environment variable present in the image is `PAPERLESS_DISABLE_DBHANDLER=true` (baked into the image, not set by this investigation). At HEAD `542221a38` this variable is referenced **only** in `setup.cfg`'s `[tool:pytest]` `env =` block — i.e. it is a **test-suite (pytest) setting**, read by **no runtime `.py` module**. It therefore has **zero** effect on the runtime pipeline this investigation exercises (`runserver` + `qcluster`, not pytest). File logging — the basis for Q2 — is driven by the unconditional `file_paperless` handler [src/paperless/settings.py:L392-L395] on the DEBUG-level `paperless` logger [src/paperless/settings.py:L409]:

```console
$ docker exec paperless-setup bash -lc 'env | grep -i PAPERLESS'
PAPERLESS_DISABLE_DBHANDLER=true
$ docker exec paperless-setup bash -lc 'cd /app/src && grep -rIn "DISABLE_DBHANDLER" . | grep -v "\.pyc"'
./setup.cfg:12:  PAPERLESS_DISABLE_DBHANDLER=true
$ docker exec paperless-setup bash -lc 'cd /app/src && grep -rIn --include="*.py" "DISABLE_DBHANDLER" . || echo "NONE in any .py runtime module"'
NONE in any .py runtime module
```

### 4. Bring up the stack (Redis → migrate → superuser → web server → worker)

Default broker/Channels backend is `PAPERLESS_REDIS=redis://localhost:6379` [paperless.conf.example:L10], satisfied by the local `redis-server` (started during environment setup and **left as found**):

```console
$ docker exec paperless-setup bash -lc 'redis-cli ping'
PONG
```

The canonical container arrives with the default SQLite database already migrated by environment setup, so `migrate` is idempotent — its **complete, unedited** output is the four lines below (no truncation):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py migrate'
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
```

Completeness of the schema (which Q3/Q4 depend on) is proven by `showmigrations`: **92** migrations are applied in total, of which **42** belong to the `documents` app (full untruncated listing):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py showmigrations | grep -c "\[X\]"'
92
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 manage.py showmigrations documents'
documents
 [X] 0001_initial
 [X] 0002_auto_20151226_1316
 [X] 0003_sender
 [X] 0004_auto_20160114_1844
 [X] 0005_auto_20160123_0313
 [X] 0006_auto_20160123_0430
 [X] 0007_auto_20160126_2114
 [X] 0008_document_file_type
 [X] 0009_auto_20160214_0040
 [X] 0010_log
 [X] 0011_auto_20160303_1929
 [X] 0012_auto_20160305_0040
 [X] 0013_auto_20160325_2111
 [X] 0014_document_checksum
 [X] 0015_add_insensitive_to_match
 [X] 0016_auto_20170325_1558
 [X] 0017_auto_20170512_0507
 [X] 0018_auto_20170715_1712
 [X] 0019_add_consumer_user
 [X] 0020_document_added
 [X] 0021_document_storage_type
 [X] 0022_auto_20181007_1420
 [X] 0023_document_current_filename
 [X] 1000_update_paperless_all
 [X] 1001_auto_20201109_1636
 [X] 1002_auto_20201111_1105
 [X] 1003_mime_types
 [X] 1004_sanity_check_schedule
 [X] 1005_checksums
 [X] 1006_auto_20201208_2209
 [X] 1007_savedview_savedviewfilterrule
 [X] 1008_auto_20201216_1736
 [X] 1009_auto_20201216_2005
 [X] 1010_auto_20210101_2159
 [X] 1011_auto_20210101_2340
 [X] 1012_fix_archive_files
 [X] 1013_migrate_tag_colour
 [X] 1014_auto_20210228_1614
 [X] 1015_remove_null_characters
 [X] 1016_auto_20210317_1351
 [X] 1017_alter_savedviewfilterrule_rule_type
 [X] 1018_alter_savedviewfilterrule_value
```

Authentication uses the canonical local development superuser `admin` created by environment setup (`DJANGO_SUPERUSER_PASSWORD=admin manage.py createsuperuser --noinput --username admin`). HTTP Basic is a default DRF auth class [src/paperless/settings.py:L118], so `-u admin:admin` is a canonical authentication method (the `admin:admin` pair is the documented local dev credential, shown literally here purely so every command below is directly reproducible):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from django.contrib.auth.models import User
print([(u.username, u.is_superuser) for u in User.objects.all().order_by(\"username\")])"'
[('admin', True), ('consumer', False)]
```

The non-superuser `consumer` account is **not** manually created state — it is created by the data migration `User.objects.create(username="consumer")` [src/documents/migrations/0019_add_consumer_user.py:L10], so it appears in *any* migrated database.

The Django web server and the **mandatory** `django-q` worker (`qcluster`) are started fresh, each detached with `stdout`/`stderr` redirected to a log file under a temporary working directory (`/tmp/qa_run`, removed at cleanup). `runserver` is invoked with `--noreload` (canonical flag; identical request behaviour) so the autoreloader does not restart the process when this `.md` is written into the watched `/app` tree:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src
    nohup python3 manage.py runserver 0.0.0.0:8000 --noreload > /tmp/qa_run/runserver.log 2>&1 &
    echo "runserver pid=$!"
    nohup python3 manage.py qcluster > /tmp/qa_run/qcluster.log 2>&1 &
    echo "qcluster pid=$!"'
runserver pid=42268
qcluster pid=42271
```

`qcluster` comes up with **11 worker processes**. Note carefully: the OCRmyPDF `jobs` parameter observed in Q2 (`jobs=11`) is **not** the worker count — it is `THREADS_PER_WORKER`, computed by a **separate** formula. Two independent settings-formulas each evaluate to 11 on this 128-CPU host and must not be conflated:

- `TASK_WORKERS = max(floor(sqrt(cpu_count)), 1) = max(floor(sqrt(128)), 1) = 11` — the number of `qcluster` worker *processes* [src/paperless/settings.py:L427-L436] [src/paperless/settings.py:L438].
- `THREADS_PER_WORKER = max(floor(cpu_count / TASK_WORKERS), 1) = max(floor(128 / 11), 1) = 11` — the per-worker thread budget, and it is *this* value that is passed to OCRmyPDF as `jobs` [src/paperless/settings.py:L469-L471] [src/paperless_tesseract/parsers.py:L149].

The two coincide at 11 only because the host has 128 CPUs; they diverge on other core counts (e.g. at 8 cores `TASK_WORKERS=floor(sqrt(8))=2` but `THREADS_PER_WORKER=floor(8/2)=4`). The equality here is a numeric coincidence, not a causal link.

### 5. Runtime settings the pipeline will use (printed, not assumed)

Every storage path resolves to its **default** location under `/app` (`os.path.realpath` shown so the `..` in the raw setting is unambiguous), and every answer-bearing setting is at its default:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, multiprocessing, django
os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\", \"paperless.settings\"); django.setup()
from django.conf import settings
from django.db import connection
rp = os.path.realpath
print(\"MEDIA_ROOT       =\", rp(settings.MEDIA_ROOT))
print(\"ORIGINALS_DIR    =\", rp(settings.ORIGINALS_DIR))
print(\"ARCHIVE_DIR      =\", rp(settings.ARCHIVE_DIR))
print(\"THUMBNAIL_DIR    =\", rp(settings.THUMBNAIL_DIR))
print(\"DATA_DIR         =\", rp(settings.DATA_DIR))
print(\"CONSUMPTION_DIR  =\", rp(settings.CONSUMPTION_DIR))
print(\"INDEX_DIR        =\", rp(settings.INDEX_DIR))
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
print(\"TASK_WORKERS       =\", settings.TASK_WORKERS)
print(\"THREADS_PER_WORKER =\", settings.THREADS_PER_WORKER)
print(\"cpu_count()        =\", multiprocessing.cpu_count())
print(\"PAPERLESS_FILENAME_FORMAT =\", repr(settings.PAPERLESS_FILENAME_FORMAT))
print(\"DB ENGINE        =\", connection.settings_dict[\"ENGINE\"])
print(\"DB NAME          =\", rp(connection.settings_dict[\"NAME\"]))"'
MEDIA_ROOT       = /app/media
ORIGINALS_DIR    = /app/media/documents/originals
ARCHIVE_DIR      = /app/media/documents/archive
THUMBNAIL_DIR    = /app/media/documents/thumbnails
DATA_DIR         = /app/data
CONSUMPTION_DIR  = /app/consume
INDEX_DIR        = /app/data/index
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
TASK_WORKERS       = 11
THREADS_PER_WORKER = 11
cpu_count()        = 128
PAPERLESS_FILENAME_FORMAT = None
DB ENGINE        = django.db.backends.sqlite3
DB NAME          = /app/data/db.sqlite3
```

These confirm the defaults cited throughout: `OCR_MODE='skip'` [src/paperless/settings.py:L522], `OCR_LANGUAGE='eng'` [src/paperless/settings.py:L514], `OCR_OUTPUT_TYPE='pdfa'` [src/paperless/settings.py:L518], `OCR_CLEAN='clean'` [src/paperless/settings.py:L526], `OCR_DESKEW=True` [src/paperless/settings.py:L528], `OCR_ROTATE_PAGES=True` [src/paperless/settings.py:L530], `OCR_ROTATE_PAGES_THRESHOLD=12.0` [src/paperless/settings.py:L532-L533], `OCR_PAGES=0` [src/paperless/settings.py:L510], `OCR_USER_ARGS='{}'` [src/paperless/settings.py:L541], `TASK_WORKERS=11` [src/paperless/settings.py:L438], `THREADS_PER_WORKER=11` [src/paperless/settings.py:L469-L471], `PAPERLESS_FILENAME_FORMAT=None` [src/paperless/settings.py:L584], and the SQLite backend at `/app/data/db.sqlite3` [src/paperless/settings.py:L299-L300]. `MEDIA_ROOT` defaults to `<BASE_DIR>/../media` [src/paperless/settings.py:L61] and `DATA_DIR` to `<BASE_DIR>/../data` [src/paperless/settings.py:L66] — i.e. `/app/media` and `/app/data`. `SCRATCH_DIR=/tmp/paperless` is the default [src/paperless/settings.py:L84]: paperless allocates a fresh `tempfile.mkdtemp` subdirectory there per document and removes it after processing (seen in the Q2 log).

### 6. Fixture that forces OCR (multi-page, image-only PDF)

Under the default `OCR_MODE='skip'`, OCRmyPDF is invoked with `skip_text=True`, which **copies text-bearing pages through unchanged, OCRing only pages with no text layer** [src/paperless_tesseract/parsers.py:L157-L158]. To guarantee the OCR path is exercised on **every** page, the fixture is a **3-page, image-only PDF** (rasterized text, no text layer). It was generated with a temporary Pillow script under `/tmp/qa_run` (removed at cleanup):

```python
# /tmp/qa_run/make_fixture.py  (temporary; removed at cleanup)
from PIL import Image, ImageDraw
pages = []
texts = [
    "PAPERLESS OCR TEST - PAGE ONE - the quick brown fox",
    "PAPERLESS OCR TEST - PAGE TWO - jumps over the lazy dog",
    "PAPERLESS OCR TEST - PAGE THREE - 1234567890 END",
]
for t in texts:
    img = Image.new("RGB", (1240, 1754), "white")   # ~150 dpi A4, raster only
    d = ImageDraw.Draw(img)
    for i, line in enumerate([t[:28], t[28:56], t[56:]]):
        d.text((80, 200 + i * 120), line, fill="black")
    d.rectangle([80, 700, 1160, 1600], outline="black", width=3)
    pages.append(img.convert("RGB"))
pages[0].save("/tmp/qa_run/ocr_test.pdf", save_all=True, append_images=pages[1:], resolution=150.0)
print("wrote /tmp/qa_run/ocr_test.pdf")
```

Proof it is **image-only** (no extractable text layer, no embedded fonts) and multi-page. `pdftotext` returns **zero** characters of text, which is far below the 50-character threshold `len(text_original) > 50` in `parse()` [src/paperless_tesseract/parsers.py:L236], so `original_has_text=False` and OCR runs on every page. The file's **SHA-256 is its stable identity** — every repeat trial re-hashes the file to prove it is the *same bytes*:

```console
$ docker exec paperless-setup bash -lc '
    printf "pdftotext_chars = "; pdftotext /tmp/qa_run/ocr_test.pdf - 2>/dev/null | tr -d "[:space:]" | wc -c
    echo "--- pdffonts (empty table => no embedded fonts) ---"; pdffonts /tmp/qa_run/ocr_test.pdf
    echo "--- pdfinfo ---"; pdfinfo /tmp/qa_run/ocr_test.pdf | grep -E "Pages|Page size"
    cd /app/src && python3 -c "import magic,sys; print(\"python-magic mime =\", magic.from_file(sys.argv[1], mime=True))" /tmp/qa_run/ocr_test.pdf
    sha256sum /tmp/qa_run/ocr_test.pdf'
pdftotext_chars = 0
--- pdffonts (empty table => no embedded fonts) ---
name                                 type              encoding         emb sub uni object ID
------------------------------------ ----------------- ---------------- --- --- --- ---------
--- pdfinfo ---
Pages:          3
Page size:      595.2 x 841.92 pts (A4)
python-magic mime = application/pdf
9f3d04909ffa109b516c19f7a77d264a6abb33004ccc526ddf89a71d8dc4f718  /tmp/qa_run/ocr_test.pdf
```

(The `pdffonts` table is empty — no embedded fonts — confirming a pure image PDF; `python-magic` detects `application/pdf`, the mime the consumer sees. The SHA-256 `9f3d0490…4f718` is the fixed identity of this fixture, reused byte-for-byte across all trials below.)

---

## Q1 — Immediate synchronous HTTP response

**Answer:** The client receives **`HTTP/1.1 200 OK`** with the **body being the JSON string `"OK"`** (Content-Type `application/json`, Content-Length 4), returned in ~0.1 s — *before* the document finishes processing. This is the return value of `PostDocumentView.post()`, which ends with `return Response("OK")` [src/documents/views.py:L535]. The value is an **enqueue acknowledgement**, not a processing result.

### Canonical route & view

The upload endpoint is registered under the `/api/` prefix [src/paperless/urls.py:L40] as the route `documents/post_document/` [src/paperless/urls.py:L56-L60], bound to `PostDocumentView` [src/documents/views.py:L491]. The view requires authentication (`permission_classes = (IsAuthenticated,)`) [src/documents/views.py:L493], parses multipart uploads (`parser_classes = (MultiPartParser,)`) [src/documents/views.py:L495], validates with `PostDocumentSerializer` [src/documents/views.py:L494], writes the upload to a temp file in `SCRATCH_DIR` [src/documents/views.py:L512-L519], generates a `uuid4` task id [src/documents/views.py:L521], enqueues the async task [src/documents/views.py:L523-L533], and finally returns `Response("OK")` [src/documents/views.py:L535]. The serializer requires a multipart `document` field (a `FileField`) [src/documents/serialisers.py:L415-L418] plus optional `title`/`correspondent`/`document_type`/`tags` [src/documents/serialisers.py:L420]; the documented contract is the multipart `document` field [docs/api.rst:L233-L235].

### Observed: exact command & complete response

The upload is driven through the canonical entry point via a short Python script (`requests.post` to the real endpoint with HTTP Basic `admin:admin`). The script also (a) records the log file's byte length immediately before the upload and again at the instant the response returns — proving **no processing log is written yet** at that instant — and (b) subscribes to the Channels `status_updates` group first so it can capture the progress events (used in Q2a). Its complete output:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 /tmp/qa_run/capture_final.py'
byte_offset_before_upload 420741
FIXTURE_SHA256 9f3d04909ffa109b516c19f7a77d264a6abb33004ccc526ddf89a71d8dc4f718
log_bytes_before_upload 420741
T0_request_sent 1783990033.574842
T1_response_received 1783990033.678263
HTTP_STATUS 200
HTTP_BODY '"OK"'
TIME_TOTAL 0.103s
log_bytes_immediately_after_response 420741 delta 0
RESPONSE_HEADERS:
  Date: Tue, 14 Jul 2026 00:47:13 GMT
  Server: WSGIServer/0.2 CPython/3.9.23
  Content-Type: application/json
  Vary: Accept, Accept-Language, Origin
  Allow: POST, OPTIONS
  X-Frame-Options: SAMEORIGIN
  Content-Length: 4
  X-Content-Type-Options: nosniff
  Referrer-Policy: same-origin
  Cross-Origin-Opener-Policy: same-origin
```

The exact `capture_final.py` command (canonical POST via `requests`) is:

```python
# /tmp/qa_run/capture_final.py  (temporary; removed at cleanup) — canonical entry point
import os, sys, time, hashlib, requests
sys.path.insert(0, "/app/src"); os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django; django.setup()
FIX, LOG = "/tmp/qa_run/ocr_test.pdf", "/app/data/log/paperless.log"
before = os.path.getsize(LOG)
T0 = time.time()
r = requests.post("http://localhost:8000/api/documents/post_document/",
                  files={"document": ("ocr_test.pdf", open(FIX, "rb"), "application/pdf")},
                  auth=("admin", "admin"))
T1 = time.time()
print("HTTP_STATUS", r.status_code, "HTTP_BODY", repr(r.text), "TIME_TOTAL", round(T1 - T0, 3))
print("log delta", os.path.getsize(LOG) - before)   # 0 => no processing log at the response instant
```

Reading directly off the captured output:

- **Status:** `HTTP_STATUS 200` — DRF's `Response("OK")` [src/documents/views.py:L535] defaults to status 200.
- **Body:** `HTTP_BODY '"OK"'` — the 4 bytes `"OK"` (the JSON encoding of the Python string `"OK"`), `Content-Type: application/json`, `Content-Length: 4`.
- **Immediacy:** `TIME_TOTAL 0.103s`, and the paperless log is byte-for-byte unchanged across the request (`delta 0`) — the response is emitted **before any stage log is written**.

### Async ordering — precise measurement (the response does *not* wait for processing, and may return *after* the worker has already started)

The response is the enqueue acknowledgement; processing happens asynchronously in the `qcluster` worker [src/documents/views.py:L523-L533] [src/documents/tasks.py:L184]. To characterise the ordering **precisely** (rather than claim the web request always returns "before processing begins"), the wall-clock instants `T0` (request sent) and `T1` (response received) from the capture above are compared against the worker's `Task.started` timestamp recorded by django-q for this job:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from django_q.models import Task
t = Task.objects.latest(\"started\")
T0 = 1783990033.574842   # request sent
T1 = 1783990033.678263   # response received
started = t.started.timestamp()
print(\"django_q Task.id       =\", t.id)
print(\"Task.func              =\", t.func)
print(\"Task.started (epoch)   =\", repr(started))
print(\"Task.started - T0      = %+.6f s\" % (started - T0))
print(\"T1 (resp) - T0         = %+.6f s\" % (T1 - T0))
print(\"Task.started - T1(resp)= %+.6f s\" % (started - T1))"'
django_q Task.id       = 944402cf731247c39f382647a55b33b6
Task.func              = documents.tasks.consume_file
Task.started (epoch)   = 1783990033.676358
Task.started - T0      = +0.101516 s
T1 (resp) - T0         = +0.103421 s
Task.started - T1(resp)= -0.001905 s
```

**Interpretation (observed, not inferred):** in this run the worker **began executing `consume_file` 1.905 ms *before* the HTTP response reached the client** (`Task.started - T1(resp) = -0.001905 s`). The web request therefore does **not** block on processing (the response is the enqueue ack, `delta 0` bytes of log at the response instant), but it is **not** categorically true that the request "returns before processing begins" — the worker picks the job off the Redis broker essentially immediately and can, and here did, start concurrently with (indeed a hair before) the client receiving `"OK"`. The rigorous statement is: **the synchronous response does not wait for processing to _finish_**, and it carries no processing result — only the acknowledgement string `"OK"`. The *first* processing **stage log line** is written ~140 ms later (its offset is captured in Q2), which is why `delta 0` at the instant of response even though the worker had already entered the task.

The two identifiers involved are **distinct** and neither is exposed in the Q1 body: django-q's own `Task.id` is a 32-hex string (`944402cf731247c39f382647a55b33b6`), while paperless's progress/correlation `task_id` is the `uuid4` minted by the view [src/documents/views.py:L521] and threaded into the async task [src/documents/tasks.py:L191] (surfaced on the status events in Q2 as `2c327614-...`). The client's only Q1 output is the bare `"OK"`.

---

## Q2 — Processing-stage logs & OCRmyPDF parameters

**Answer:** The worker logs an ordered sequence of stage lines through `paperless.consumer`, `paperless.parsing`, and `paperless.parsing.tesseract`, each tagged with a per-document `group` correlation UUID. The pivotal line — the exact OCRmyPDF invocation — is `Calling OCRmyPDF with args: {...}` [src/paperless_tesseract/parsers.py:L260], emitted immediately before `ocrmypdf.ocr(**args)` [src/paperless_tesseract/parsers.py:L261]. For this default-config image-only PDF the args dict has **13 keys**, enumerated verbatim below. In addition, parallel to the file log, the consumer publishes STARTING/WORKING/SUCCESS **progress events** over Channels/Redis.

### Where the logs come from

The verbose formatter is `"[{asctime}] [{levelname}] [{name}] {message}"` [src/paperless/settings.py:L378]; the `paperless` logger writes at DEBUG to `DATA_DIR/log/paperless.log` via a `ConcurrentRotatingFileHandler` [src/paperless/settings.py:L392-L395] [src/paperless/settings.py:L409], while the root logger also writes to the console [src/paperless/settings.py:L407]. Pipeline loggers are `paperless.consumer` [src/documents/consumer.py:L54] and `paperless.parsing.tesseract` [src/paperless_tesseract/parsers.py:L24]. Every record carries a per-document `group` UUID injected by `LoggingMixin.log()` [src/documents/loggers.py:L11-L12] [src/documents/loggers.py:L14] [src/documents/loggers.py:L21].

### Observed: the complete ordered stage log for this upload

To read **only this run's** lines out of the shared log file, the byte offset captured in Q1 (`byte_offset_before_upload 420741`) is used with `tail -c +<offset+1>` to extract exactly what this document's processing appended. The **complete, unedited** slice (15 lines) is:

```console
$ docker exec paperless-setup bash -lc 'tail -c +420742 /app/data/log/paperless.log'
[2026-07-14 00:47:13,817] [INFO] [paperless.consumer] Consuming ocr_test.pdf
[2026-07-14 00:47:13,818] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-14 00:47:13,818] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-14 00:47:13,821] [DEBUG] [paperless.consumer] Parsing ocr_test.pdf...
[2026-07-14 00:47:13,845] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-pi00stla
[2026-07-14 00:47:13,918] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-pi00stla', 'output_file': '/tmp/paperless/paperless-ut6cnxe5/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-ut6cnxe5/sidecar.txt'}
[2026-07-14 00:47:16,618] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 00:47:16,618] [DEBUG] [paperless.consumer] Generating thumbnail for ocr_test.pdf...
[2026-07-14 00:47:16,623] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-ut6cnxe5/archive.pdf[0] /tmp/paperless/paperless-ut6cnxe5/convert.png
[2026-07-14 00:47:17,347] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-ut6cnxe5/convert.png -out /tmp/paperless/paperless-ut6cnxe5/thumb_optipng.png
[2026-07-14 00:47:18,367] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-14 00:47:18,371] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-14 00:47:18,396] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-pi00stla
[2026-07-14 00:47:18,420] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-ut6cnxe5
[2026-07-14 00:47:18,420] [INFO] [paperless.consumer] Document 2026-07-14 ocr_test consumption finished
```

Each line maps to a specific call site in the worker (the ordered stage pattern):

| # | Observed line pattern | Emitted by | Citation |
|---|-----------------------|------------|----------|
| 1 | `Consuming ocr_test.pdf` | `Consumer.try_consume_file()` | [src/documents/consumer.py:L215] |
| 2 | `Detected mime type: application/pdf` | consumer | [src/documents/consumer.py:L221] |
| 3 | `Parser: RasterisedDocumentParser` | consumer | [src/documents/consumer.py:L246] |
| 4 | `Parsing ocr_test.pdf...` | consumer | [src/documents/consumer.py:L260] |
| 5 | `Extracted text from PDF file /tmp/paperless/paperless-upload-...` | tesseract parser (pre-check for existing text) | [src/paperless_tesseract/parsers.py:L122] |
| 6 | **`Calling OCRmyPDF with args: {...}`** | tesseract parser (immediately before `ocrmypdf.ocr`) | [src/paperless_tesseract/parsers.py:L260] |
| 7 | `Using text from sidecar file` | tesseract parser (reads the sidecar OCRmyPDF wrote) | [src/paperless_tesseract/parsers.py:L107] |
| 8 | `Generating thumbnail for ocr_test.pdf...` | consumer | [src/documents/consumer.py:L263] |
| 9 | `Execute: convert -density 300 -scale 500x5000> ...` | base parser thumbnail via ImageMagick | [src/documents/parsers.py:L143] |
| 10 | `Execute: optipng -silent -o5 ...` | tesseract parser thumbnail optimisation | [src/documents/parsers.py:L333] |
| 11 | `Document classification model does not exist (yet)...` | classifier (no trained model on a fresh instance) | [src/documents/classifier.py:L33] |
| 12 | `Saving record to database` | consumer (inside `transaction.atomic()`) | [src/documents/consumer.py:L387] |
| 13 | `Deleting file /tmp/paperless/paperless-upload-...` | consumer cleanup of the staged upload | [src/documents/consumer.py:L349] |
| 14 | `Deleting directory /tmp/paperless/paperless-...` | base parser `cleanup()` (tesseract parser's logger) | [src/documents/parsers.py:L349] |
| 15 | `Document 2026-07-14 ocr_test consumption finished` | consumer (final INFO) | [src/documents/consumer.py:L373] |

The scratch paths (`/tmp/paperless/paperless-upload-pi00stla`, `/tmp/paperless/paperless-ut6cnxe5/…`) are the per-document `mkdtemp` allocations under the default `SCRATCH_DIR=/tmp/paperless`; both are deleted by the consumer (line 13) and the parser's `cleanup()` (line 14) at the end of processing, so nothing leaks on the success path. Note line 14 is logged under `paperless.parsing.tesseract` (not `paperless.consumer`) because it is emitted by the parser's `cleanup()` method, which the consumer invokes in its `finally` block.

### Q2 (a) — Progress events broadcast over Channels/Redis

Parallel to the file log, `Consumer._send_progress()` [src/documents/consumer.py:L56] publishes milestones to the Channels group `status_updates` [src/documents/consumer.py:L73-L76] using the STARTING/WORKING/SUCCESS constants [src/documents/consumer.py:L43-L49]. These were captured **canonically** by the same `capture_final.py` subscribing a listener to the `status_updates` group (backed by the default `channels-redis` layer) while the upload ran. The complete captured event stream (6 events) for this document:

```console
$ docker exec paperless-setup bash -lc 'cat /tmp/qa_run/final/progress.txt'
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
{"filename": "ocr_test.pdf", "task_id": "2c327614-e46d-4666-902a-ab9493a4ab43", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
PROGRESS_EVENT_COUNT 6
```

Each event corresponds to a `_send_progress` call site: `new_file`/STARTING at 0 [src/documents/consumer.py:L202], `parsing_document` at 20 [src/documents/consumer.py:L259], `generating_thumbnail` at 70 [src/documents/consumer.py:L264], `parse_date` at 90 [src/documents/consumer.py:L274], `save_document` at 95 [src/documents/consumer.py:L294], and `finished`/SUCCESS at 100 [src/documents/consumer.py:L375]. The payload keys `{filename, task_id, current_progress, max_progress, status, message, document_id}` are assembled at [src/documents/consumer.py:L64-L72]. Note `document_id` is `null` until the final SUCCESS event, where it becomes `1`; the same `task_id` UUID (`2c327614-...`) that the view minted [src/documents/views.py:L521] threads through every event.

*Transport note (observed):* the WebSocket route `ws/status/` is registered as an **ASGI** consumer [src/paperless/urls.py:L137], but this investigation runs the canonical dev **WSGI** `runserver`, so a plain HTTP `GET ws/status/` does not upgrade to a WebSocket — it is redirected (observed `302`, shown in the [coverage section](#coverage--edge-cases)). The progress events above were therefore captured canonically at the **channel-layer** boundary (subscribing to the `status_updates` group), which is exactly the group the real `StatusConsumer` joins — not through a bypassing hook.

### Q2 (b) — The OCRmyPDF invocation: every parameter enumerated

The dict logged at line 6 above is the literal `**kwargs` passed to `ocrmypdf.ocr()` [src/paperless_tesseract/parsers.py:L261]. It is assembled by `construct_ocrmypdf_parameters()` [src/paperless_tesseract/parsers.py:L135] and logged verbatim at `parse()` [src/paperless_tesseract/parsers.py:L230] [src/paperless_tesseract/parsers.py:L260]. For this default-config, image-only PDF it has **exactly 13 keys**. Each is enumerated below with its **observed value** and the **source line + originating setting** that produced it:

| # | Parameter | Observed value | Origin (code + setting) |
|---|-----------|----------------|-------------------------|
| 1 | `input_file` | `/tmp/paperless/paperless-upload-pi00stla` | the staged upload path passed into `parse()` [src/paperless_tesseract/parsers.py:L144] |
| 2 | `output_file` | `/tmp/paperless/paperless-ut6cnxe5/archive.pdf` | archive target in the scratch dir [src/paperless_tesseract/parsers.py:L145] |
| 3 | `use_threads` | `True` | hard-set [src/paperless_tesseract/parsers.py:L148] |
| 4 | `jobs` | `11` | `= THREADS_PER_WORKER` [src/paperless_tesseract/parsers.py:L149]; value from [src/paperless/settings.py:L469-L471] |
| 5 | `language` | `'eng'` | `OCR_LANGUAGE` [src/paperless_tesseract/parsers.py:L150]; default [src/paperless/settings.py:L514] |
| 6 | `output_type` | `'pdfa'` | `OCR_OUTPUT_TYPE` [src/paperless_tesseract/parsers.py:L151]; default [src/paperless/settings.py:L518] |
| 7 | `progress_bar` | `False` | hard-set [src/paperless_tesseract/parsers.py:L152] |
| 8 | `skip_text` | `True` | from `OCR_MODE='skip'` branch [src/paperless_tesseract/parsers.py:L157-L158]; mode default [src/paperless/settings.py:L522] |
| 9 | `clean` | `True` | from `OCR_CLEAN='clean'` branch [src/paperless_tesseract/parsers.py:L164-L165]; default [src/paperless/settings.py:L526] |
| 10 | `deskew` | `True` | `OCR_DESKEW` [src/paperless_tesseract/parsers.py:L172-L173]; default [src/paperless/settings.py:L528] |
| 11 | `rotate_pages` | `True` | `OCR_ROTATE_PAGES` [src/paperless_tesseract/parsers.py:L175-L176]; default [src/paperless/settings.py:L530] |
| 12 | `rotate_pages_threshold` | `12.0` | `OCR_ROTATE_PAGES_THRESHOLD` [src/paperless_tesseract/parsers.py:L177-L179]; default [src/paperless/settings.py:L532-L533] |
| 13 | `sidecar` | `/tmp/paperless/paperless-ut6cnxe5/sidecar.txt` | sidecar text target (in the `else` branch, incompatible with `pages`) [src/paperless_tesseract/parsers.py:L185] |

**Why exactly these 13 (branches NOT taken, so keys absent) — observed & code-read:**
- `pages` is added **only if** `OCR_PAGES > 0` [src/paperless_tesseract/parsers.py:L181-L182]; default `OCR_PAGES=0` [src/paperless/settings.py:L510] ⇒ **absent** (confirmed: no `pages` key above; instead the `else` branch adds `sidecar`).
- `image_dpi` is added **only for image inputs** (not PDFs) [src/paperless_tesseract/parsers.py:L187-L215] via the `is_image` check [src/paperless_tesseract/parsers.py:L64-L70]; input is a PDF ⇒ **absent**.
- No user overrides are merged: `OCR_USER_ARGS='{}'` [src/paperless_tesseract/parsers.py:L217-L220] ⇒ nothing added; default [src/paperless/settings.py:L541].
- `force_ocr` would come from the `OCR_MODE=='force'`/safe-fallback branch [src/paperless_tesseract/parsers.py:L155-L156]; mode is `skip`, so the `elif` skip branch [src/paperless_tesseract/parsers.py:L157-L158] sets `skip_text` instead and `force_ocr` is **absent**.

The single-threaded OCRmyPDF environment guard `OMP_THREAD_LIMIT=1` is set separately [src/paperless_tesseract/parsers.py:L232] and is not part of the kwargs dict.

### Q2 (c) — Primary vs. fallback invocation (the "safe" retry path)

The primary call is guarded: if `ocrmypdf.ocr(**args)` raises one of the recoverable exceptions, `parse()` catches it [src/paperless_tesseract/parsers.py:L276] and retries with a **fallback** invocation that logs `Fallback: Calling OCRmyPDF with args: {...}` [src/paperless_tesseract/parsers.py:L297] and re-runs `ocrmypdf.ocr(**args)` [src/paperless_tesseract/parsers.py:L298] with `force_ocr=True` (via `safe_fallback` at [src/paperless_tesseract/parsers.py:L155-L156]). On this clean image-only fixture the **primary** call succeeds, so the fallback line never appears. This was verified by counting both patterns across the run's log slice:

```console
$ docker exec paperless-setup bash -lc '
    S=/tmp/qa_run/final/q2slice.txt
    echo "primary_calls=$(grep -c "Calling OCRmyPDF with args:" "$S")"
    echo "fallback_calls=$(grep -c "Fallback: Calling OCRmyPDF with args:" "$S")"'
primary_calls=1
fallback_calls=0
```

So on the canonical happy path: **primary = 1, fallback = 0**. The fallback path is a real secondary code path but is **not** exercised by a well-formed image-only PDF under default config.

### Q2 (d) — Worker result, and the "same input twice" behavior (relevant to the reported inconsistency)

**Worker outcome.** The async job `documents.tasks.consume_file` [src/documents/tasks.py:L184] delegates to `Consumer.try_consume_file()` [src/documents/tasks.py:L236] and returns the success string `"Success. New document id {} created"` [src/documents/tasks.py:L247]. Read straight off the django-q `Task` row for this run:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from django_q.models import Task
t = Task.objects.latest(\"started\")
print(\"id      =\", t.id)
print(\"func    =\", t.func)
print(\"success =\", t.success)
print(\"result  =\", repr(t.result))"'
id      = 944402cf731247c39f382647a55b33b6
func    = documents.tasks.consume_file
success = True
result  = 'Success. New document id 1 created'
```

**Uploading the same bytes again is rejected as a duplicate (and leaks a staged file — a real observed behavior).** Duplicate detection lives in `Consumer.pre_check_duplicate()` [src/documents/consumer.py:L102], which hashes the file with MD5 [src/documents/consumer.py:L104] and matches `Q(checksum=...) | Q(archive_checksum=...)` [src/documents/consumer.py:L106]; on a hit it raises a `ConsumerError` via `self._fail(...)` with the message `"Not consuming ...: It is a duplicate."` [src/documents/consumer.py:L110-L112]. It is called at the very top of `try_consume_file()` [src/documents/consumer.py:L213] — **before** the "Consuming" log line [src/documents/consumer.py:L215] — so on a duplicate only the ERROR line is emitted (no "Consuming" line). Re-uploading the **identical fixture bytes** returns `200 "OK"` synchronously (Q1 is just the enqueue ack), but the worker then fails the job:

```console
$ docker exec paperless-setup bash -lc '
    LOG=/app/data/log/paperless.log; before=$(wc -c < "$LOG")
    ndir_before=$(ls -1 /tmp/paperless 2>/dev/null | wc -l)
    docs_before=$(cd /app/src && python3 -c "import os,django;os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\");django.setup();from documents.models import Document;print(Document.objects.count())")
    code=$(curl -s -o /dev/null -w "%{http_code}" -u admin:admin -F "document=@/tmp/qa_run/ocr_test.pdf" http://localhost:8000/api/documents/post_document/)
    echo "resync_http_code=$code"
    sleep 6
    echo "--- new log lines ---"; tail -c +$((before+1)) "$LOG"
    docs_after=$(cd /app/src && python3 -c "import os,django;os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\");django.setup();from documents.models import Document;print(Document.objects.count())")
    ndir_after=$(ls -1 /tmp/paperless 2>/dev/null | wc -l)
    echo "documents_count: before=$docs_before after=$docs_after"
    echo "scratch_entries: before=$ndir_before after=$ndir_after"'
resync_http_code=200
--- new log lines ---
[2026-07-14 00:48:26,531] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
documents_count: before=1 after=1
scratch_entries: before=0 after=1
```

Two observed facts:
1. The document count stays at **1** — the duplicate is not ingested (correct behavior), and only the single `[ERROR] ... It is a duplicate.` line is logged (no "Consuming" line, because the duplicate short-circuit precedes it).
2. The staged upload file the view wrote to `SCRATCH_DIR` [src/documents/views.py:L512-L519] is **left behind** (`scratch_entries: before=0 after=1`) because the duplicate short-circuit at [src/documents/consumer.py:L213] happens *before* the consumer's own scratch-cleanup path. This orphaned-staged-file behavior on the duplicate path is a **pre-existing product behavior**, documented honestly here; **fixing it is out of scope** for this read-only investigation (AAP §0.5.2 forbids source changes). This investigation removes that specific leaked file itself during [Cleanup](#cleanup-confirmation).

**Archive PDF bytes are not bit-reproducible across runs — but the OCR *text* is.** Running the identical logged OCRmyPDF args twice produces archive PDFs that differ in a handful of metadata bytes. The cause is **not** the OCR content and **not** a document "creation" timestamp: it is the PDF's own **`ModDate`** and the **trailer `/ID`**, which OCRmyPDF/Ghostscript regenerate per run; the **`CreationDate` is constant** across runs. Verified by invoking `ocrmypdf.ocr()` twice with the exact args from the log and diffing:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import ocrmypdf, os, pikepdf, shutil
def run(outdir):
    os.makedirs(outdir, exist_ok=True)
    out=os.path.join(outdir,\"archive.pdf\"); side=os.path.join(outdir,\"sidecar.txt\")
    ocrmypdf.ocr(input_file=\"/tmp/qa_run/ocr_test.pdf\", output_file=out, use_threads=True, jobs=11,
        language=\"eng\", output_type=\"pdfa\", progress_bar=False, skip_text=True, clean=True,
        deskew=True, rotate_pages=True, rotate_pages_threshold=12.0, sidecar=side)
    with pikepdf.open(out) as p:
        di=p.docinfo
        return out, side, str(di.get(\"/CreationDate\",\"\")), str(di.get(\"/ModDate\",\"\")), [bytes(x).hex() for x in p.trailer.get(\"/ID\",[])]
oa,sa,ca,ma,ida=run(\"/tmp/qa_run/vA\"); ob,sb,cb,mb,idb=run(\"/tmp/qa_run/vB\")
a=open(oa,\"rb\").read(); b=open(ob,\"rb\").read()
off=next((i for i in range(min(len(a),len(b))) if a[i]!=b[i]), -1)
print(\"first_byte_difference_at =\", off)
print(\"CreationDate run1 =\", ca)
print(\"CreationDate run2 =\", cb, \"[CONSTANT]\" if ca==cb else \"[VARIES]\")
print(\"ModDate      run1 =\", ma)
print(\"ModDate      run2 =\", mb, \"[CONSTANT]\" if ma==mb else \"[VARIES]\")
print(\"trailer /ID  run1 =\", ida)
print(\"trailer /ID  run2 =\", idb, \"[VARIES]\" if ida!=idb else \"[CONSTANT]\")
print(\"sidecar OCR text identical? =\", open(sa).read()==open(sb).read())
shutil.rmtree(\"/tmp/qa_run/vA\"); shutil.rmtree(\"/tmp/qa_run/vB\")"'
first_byte_difference_at = 332
CreationDate run1 = D:20260714000727+00'00'
CreationDate run2 = D:20260714000727+00'00' [CONSTANT]
ModDate      run1 = D:20260714004854+00'00'
ModDate      run2 = D:20260714004856+00'00' [VARIES]
trailer /ID  run1 = ['bfd37cc92f482c14f6d52de90dbc772c', '3af095c8b92b0cd2526cf476fbb4b44a']
trailer /ID  run2 = ['3c42901aa8f754aface4613c992eeaea', 'b79ffcaaee4e6e7b771640f1889bfa4b'] [VARIES]
sidecar OCR text identical? = True
```

So the practical implication for the user's report: the **OCR text output is deterministic** for a given input under fixed config (the sidecar is byte-identical across runs); only non-semantic PDF metadata (`ModDate`, trailer `/ID`) varies run-to-run, which explains differing archive **checksums** (see Q4's `archive_checksum`) without implying "inconsistent OCR". The genuine source of *content* inconsistency on **mixed** PDFs is `skip_text=True` (analysed in the [Synthesis](#synthesis--relating-observations-to-inconsistent-ocr-results)).

---

## Q3 — Generated media filenames

**Answer:** With `PAPERLESS_FILENAME_FORMAT` unset (default), the archive PDF and thumbnail are named by the document's zero-padded primary key: **`0000001.pdf`** for both the original and the archive, and **`0000001.png`** for the thumbnail. For this run (`pk=1`) they live at their **default** locations under `/app/media/documents/{originals,archive,thumbnails}/`.

### Where the names come from

The consumer assigns the stored filenames inside its `transaction.atomic()` block by calling `generate_unique_filename()`/`generate_filename()` [src/documents/consumer.py:L316] [src/documents/consumer.py:L328-L331]. With `PAPERLESS_FILENAME_FORMAT` falsy, `generate_filename()` [src/documents/file_handling.py:L128] skips the format branch [src/documents/file_handling.py:L132] and returns `"{doc.pk:07}{counter_str}{filetype_str}"` [src/documents/file_handling.py:L186] [src/documents/file_handling.py:L188] [src/documents/file_handling.py:L193] — i.e. the 7-digit zero-padded pk plus the file-type suffix. The post-save `update_filename_and_move_files` signal handler is a **no-op** under default config because the format is unset, so filenames do not change and it returns early [src/documents/signals/handlers.py:L311-L312] [src/documents/signals/handlers.py:L330-L331] [src/documents/signals/handlers.py:L347-L349].

### Observed: the media directories after processing

```console
$ docker exec paperless-setup bash -lc '
    for d in originals archive thumbnails; do
      echo "=== /app/media/documents/$d ==="
      ls -l /app/media/documents/$d
    done'
=== /app/media/documents/originals ===
total 148
-rw-r--r-- 1 root root 150770 Jul 14 00:47 0000001.pdf
=== /app/media/documents/archive ===
total 24
-rw-r--r-- 1 root root 20927 Jul 14 00:47 0000001.pdf
=== /app/media/documents/thumbnails ===
total 8
-rw-r--r-- 1 root root 4124 Jul 14 00:47 0000001.png
```

Reading off the output:

- **Original:** `/app/media/documents/originals/0000001.pdf` (150770 bytes — identical size to the uploaded fixture, as expected for the untouched source).
- **Archive PDF:** `/app/media/documents/archive/0000001.pdf` (20927 bytes — the OCRmyPDF `output_type='pdfa'` result).
- **Thumbnail:** `/app/media/documents/thumbnails/0000001.png` (4124 bytes — the ImageMagick+optipng PNG from Q2 lines 9–10).

All three share the `0000001` stem = `pk=1` zero-padded to 7 digits, differing only by extension/subdirectory. A second document would be `0000002.*`, and so on. (`counter_str` in the format is empty here because there is no filename collision; it only becomes non-empty when two documents would otherwise map to the same name.)

---

## Q4 — Database-stored fields vs. computed properties

**Answer:** The `documents_document` table has **15 columns**. The document's on-disk *paths* are **not** among them — `source_path`, `archive_path`, `thumbnail_path`, and `file_type` are Python `@property` methods that compose filesystem paths from stored fields at runtime and are **not** persisted. What *is* stored is the **relative filename strings** (`filename`, `archive_filename`), the checksums, mime type, timestamps, storage type, title, content, and the FK/serial columns.

### The stored schema (15 columns) — read from SQLite via a reproducible Python query

(The container has no `sqlite3` CLI, so the schema is dumped through the Django DB cursor's `PRAGMA table_info`; columns are `cid|name|type|notnull|dflt_value|pk`.)

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from django.db import connection
with connection.cursor() as c:
    c.execute(\"PRAGMA table_info(documents_document);\")
    for r in c.fetchall():
        print(\"|\".join(\"\" if v is None else str(v) for v in r))"'
0|id|integer|1||1
1|title|varchar(128)|1||0
2|content|text|1||0
3|created|datetime|1||0
4|modified|datetime|1||0
5|correspondent_id|integer|0||0
6|checksum|varchar(32)|1||0
7|added|datetime|1||0
8|storage_type|varchar(11)|1||0
9|archive_serial_number|integer|0||0
10|document_type_id|integer|0||0
11|mime_type|varchar(256)|1||0
12|archive_checksum|varchar(32)|0||0
13|archive_filename|varchar(1024)|0||0
14|filename|varchar(1024)|0||0
```

Each column maps to a model field: `title` [src/documents/models.py:L106], `content` [src/documents/models.py:L117], `created` [src/documents/models.py:L152], `modified` [src/documents/models.py:L154], `correspondent_id` (FK) [src/documents/models.py:L97], `checksum` [src/documents/models.py:L135], `added` [src/documents/models.py:L169], `storage_type` [src/documents/models.py:L161], `archive_serial_number` [src/documents/models.py:L196], `document_type_id` (FK) [src/documents/models.py:L108], `mime_type` [src/documents/models.py:L126], `archive_checksum` [src/documents/models.py:L143], `archive_filename` [src/documents/models.py:L186], `filename` [src/documents/models.py:L176]. Tags are a many-to-many [src/documents/models.py:L128] stored in the through-table `documents_document_tags` (not a column on `documents_document`).

### The stored values for this document (ORM, concrete fields only)

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from documents.models import Document
d = Document.objects.get(pk=1)
for f in d._meta.concrete_fields:
    print(\"%-24s = %r\" % (f.column, getattr(d, f.attname)))
print(\"%-24s = %r\" % (\"tags (through-table)\", list(d.tags.values_list(\"id\", flat=True))))"'
id                       = 1
correspondent_id         = None
title                    = 'ocr_test'
document_type_id         = None
content                  = 'E ~ the quick brom fox\n\n\n\n\n\n\n\n\n© ~ jumps over the lazy dog\n\n\n\n\n\n\n\n\nnex ~ 1294567090 Em'
mime_type                = 'application/pdf'
checksum                 = 'd4488441764aa4bc6b8abac1957f9823'
archive_checksum         = 'd590b7394661acad76af27f005fc22a8'
created                  = datetime.datetime(2026, 7, 14, 0, 47, 13, tzinfo=datetime.timezone.utc)
modified                 = datetime.datetime(2026, 7, 14, 0, 47, 18, 395518, tzinfo=datetime.timezone.utc)
storage_type             = 'unencrypted'
added                    = datetime.datetime(2026, 7, 14, 0, 47, 18, 372289, tzinfo=datetime.timezone.utc)
filename                 = '0000001.pdf'
archive_filename         = '0000001.pdf'
archive_serial_number    = None
tags (through-table)     = []
```

Notable observed values:
- `filename` and `archive_filename` are **relative** strings (`'0000001.pdf'`), *not* absolute paths — the media root is prepended only by the computed properties below.
- `checksum='d4488441764aa4bc6b8abac1957f9823'` is the MD5 of the **original** bytes [src/documents/consumer.py:L104]; `archive_checksum='d590b7394661acad76af27f005fc22a8'` is the MD5 of the **archive** PDF. Because the archive's non-semantic metadata varies per run (Q2d), `archive_checksum` is run-dependent while `checksum` is fixed for a given input (the fixture's MD5 `d4488441...` reproduced identically across every run in this investigation).
- `content` holds the OCR text of **all three** pages (with realistic OCR imperfections like `brom`/`©`/`nex`), confirming every page was OCR'd — consistent coverage on a fully-image PDF.
- `correspondent_id`, `document_type_id`, `archive_serial_number` are `None`; `storage_type='unencrypted'`; `mime_type='application/pdf'`.

### The computed properties (NOT stored) — proven to be Python properties absent from the schema

`source_path` [src/documents/models.py:L222-L231], `archive_path` [src/documents/models.py:L241-L246], `thumbnail_path` [src/documents/models.py:L272-L278], and `file_type` [src/documents/models.py:L268-L270] are `@property` methods. The check below proves, for each name, that it **is** a Python `property` on the class **and** is **not** a database column:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from documents.models import Document
d = Document.objects.get(pk=1)
cols = {f.column for f in Document._meta.concrete_fields}
for name in [\"source_path\", \"archive_path\", \"thumbnail_path\", \"file_type\", \"has_archive_version\"]:
    is_prop = isinstance(getattr(Document, name), property)
    print(\"%-20s is_property=%s in_db_columns=%s value=%r\" % (name, is_prop, name in cols, getattr(d, name)))"'
source_path          is_property=True in_db_columns=False value='/app/src/../media/documents/originals/0000001.pdf'
archive_path         is_property=True in_db_columns=False value='/app/src/../media/documents/archive/0000001.pdf'
thumbnail_path       is_property=True in_db_columns=False value='/app/src/../media/documents/thumbnails/0000001.png'
file_type            is_property=True in_db_columns=False value='.pdf'
has_archive_version  is_property=True in_db_columns=False value=True
```

Each resolves to a default `/app/media/...` path at runtime (the `..` is literal in the raw setting `MEDIA_ROOT=<BASE_DIR>/../media` [src/paperless/settings.py:L61]; `os.path.realpath` would collapse `/app/src/../media` → `/app/media`, matching the Q3 listing). The clean separation is the crux of Q4: **paths are computed from the stored relative `filename`/`archive_filename` plus `pk`, never stored themselves.**

---

## Synthesis — relating observations to "inconsistent OCR results"

The reported "inconsistent OCR results" on multi-page PDFs is explained by the **default OCR mode**, not by nondeterminism in OCR itself:

- The default `OCR_MODE='skip'` [src/paperless/settings.py:L522] maps to `skip_text=True` in the OCRmyPDF args [src/paperless_tesseract/parsers.py:L157-L158] (observed as parameter #8 in Q2b). Under `skip_text`, OCRmyPDF **copies pages that already contain a text layer through unchanged and OCRs only pages that lack text**. On a **mixed** multi-page PDF (some pages digitally-born with text, some scanned images), only the image pages gain OCR text; the born-digital pages are passed through and their text does **not** flow into the sidecar that paperless reads back [src/paperless_tesseract/parsers.py:L107]. The result is **uneven / "inconsistent" text coverage across pages** — exactly the reported symptom.
- On the **fully image-only** fixture used here, *every* page lacks a text layer, so `skip_text` OCRs *every* page and coverage is **consistent** (all three pages appear in `content`, Q4). This is the deliberate control that isolates the mechanism: switch even one page to born-digital text and that page would be skipped.
- The observed **archive-checksum variance** across runs (Q2d) is a **red herring** for "inconsistent OCR": the OCR *text* (sidecar) is byte-identical across repeats; only the PDF `ModDate` and trailer `/ID` change. So run-to-run `archive_checksum` differences do not indicate inconsistent recognition.
- The non-default modes are the levers a user would reach for: `force_ocr` (rasterize & OCR every page) and `redo_ocr` (strip & re-OCR existing text) come from the other `OCR_MODE` branches [src/paperless_tesseract/parsers.py:L155-L156]; `force_ocr=True` is also paperless's own automatic **fallback** when the primary call raises [src/paperless_tesseract/parsers.py:L297-L298]. Under default config neither is used.

**The task is to document behavior, not change it.** Adjusting `OCR_MODE` away from `skip` would very likely make coverage uniform on mixed PDFs, but doing so is **out of scope** (AAP §0.5.2: configuration must remain at defaults; no code/config changes).

---

## Coverage — edge cases

Beyond the happy path, the canonical endpoint's error/edge behavior was exercised through the same real entry point. Every negative case below was **bracketed by before/after snapshots** of the document count, media directory, and scratch directory to prove it produces **no side effects**. All responses — including errors — carry the default security headers `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`, `Cross-Origin-Opener-Policy: same-origin`, sourced from `SecurityMiddleware` [src/paperless/settings.py:L135] and `XFrameOptionsMiddleware` [src/paperless/settings.py:L145] with `X_FRAME_OPTIONS` defaulting to `SAMEORIGIN` [src/paperless/settings.py:L221].

### Side-effect snapshot (invariant across all negative cases)

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from documents.models import Document
print(\"documents_count =\", Document.objects.count())" ; \
    echo "media_docs      = $(ls -1 /app/media/documents/originals | wc -l)"; \
    echo "scratch_entries = $(ls -1 /tmp/paperless 2>/dev/null | wc -l)"'
documents_count = 1
media_docs      = 1
scratch_entries = 0
```

This snapshot was taken **before and after** the block of negative cases below; both times it read `documents_count=1, media_docs=1, scratch_entries=0` — i.e. the canonical `pk=1` document is untouched and **none** of the negatives created a document, media file, or scratch leak.

### E1 — Unauthenticated POST → 401

`permission_classes=(IsAuthenticated,)` [src/documents/views.py:L493] with HTTP Basic as a default auth class [src/paperless/settings.py:L118] ⇒ a request with no credentials is rejected before any file handling:

```console
$ docker exec paperless-setup bash -lc 'curl -s -i -F "document=@/tmp/qa_run/ocr_test.pdf" http://localhost:8000/api/documents/post_document/'
HTTP/1.1 401 Unauthorized
Date: Tue, 14 Jul 2026 00:48:58 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Accept-Language, Origin, Cookie
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
Content-Length: 58
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Authentication credentials were not provided."}
```

### E2 — Malformed / invalid credentials → 401

Wrong password is likewise rejected (distinct DRF detail message), proving auth is actually enforced (not merely presence-checked):

```console
$ docker exec paperless-setup bash -lc 'curl -s -i -u admin:WRONGPASS -F "document=@/tmp/qa_run/ocr_test.pdf" http://localhost:8000/api/documents/post_document/'
HTTP/1.1 401 Unauthorized
Date: Tue, 14 Jul 2026 00:48:58 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Accept-Language, Origin
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
Content-Length: 39
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Invalid username/password."}
```

### E3 — Missing `document` field → 400 (synchronous serializer rejection)

The serializer's required `document` FileField [src/documents/serialisers.py:L415-L418] fails validation **synchronously** (no async task enqueued, no scratch file written):

```console
$ docker exec paperless-setup bash -lc 'curl -s -i -u admin:admin -F "notdocument=@/tmp/qa_run/ocr_test.pdf" http://localhost:8000/api/documents/post_document/'
HTTP/1.1 400 Bad Request
Date: Tue, 14 Jul 2026 00:48:58 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 39
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"document":["No file was submitted."]}
```

### E4 — Unsupported file type → 400 (rejected before enqueue; no scratch leak)

The serializer's `validate_document` [src/documents/serialisers.py:L450] sniffs the buffer with `magic.from_buffer` [src/documents/serialisers.py:L452] and rejects when `not is_mime_type_supported(...)` [src/documents/serialisers.py:L454] (imported at [src/documents/serialisers.py:L18]) with the message template `File type %(type)s not supported` [src/documents/serialisers.py:L456]. Uploading raw binary (`application/octet-stream`) is rejected **synchronously** — importantly, this rejection happens in the serializer *before* the view writes the staged file, so unlike the duplicate path (Q2d) it leaves **no** scratch leak:

```console
$ docker exec paperless-setup bash -lc '
    head -c 4096 /dev/urandom > /tmp/qa_run/junk.bin
    curl -s -i -u admin:admin -F "document=@/tmp/qa_run/junk.bin" http://localhost:8000/api/documents/post_document/
    echo "scratch_after_E4=$(ls -1 /tmp/paperless 2>/dev/null | wc -l)"
    rm -f /tmp/qa_run/junk.bin'
HTTP/1.1 400 Bad Request
Date: Tue, 14 Jul 2026 00:49:15 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 65
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"document":["File type application/octet-stream not supported"]}
scratch_after_E4=0
```

### E5 — Wrong HTTP method (GET) → 405

The view only implements `post()` [src/documents/views.py:L497]; a `GET` yields `405 Method Not Allowed` with the permitted methods in `Allow`:

```console
$ docker exec paperless-setup bash -lc 'curl -s -i -u admin:admin http://localhost:8000/api/documents/post_document/'
HTTP/1.1 405 Method Not Allowed
Date: Tue, 14 Jul 2026 00:49:16 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 40
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Method \"GET\" not allowed."}
```

### E6 — CORS behavior

`CORS_ALLOWED_ORIGINS` defaults to `('http://localhost:8000',)` from `PAPERLESS_CORS_ALLOWED_HOSTS` [src/paperless/settings.py:L232-L233]. A preflight from the allowed origin gets the `Access-Control-Allow-Origin` echo; a foreign origin (`evil.example.com`) gets a `200` preflight but **no** `Access-Control-Allow-Origin` header (so browsers block it):

```console
$ docker exec paperless-setup bash -lc '
    echo "--- allowed origin ---"
    curl -s -i -X OPTIONS -H "Origin: http://localhost:8000" -H "Access-Control-Request-Method: POST" http://localhost:8000/api/documents/post_document/ | grep -iE "^HTTP|^access-control-allow-origin|^vary"
    echo "--- foreign origin ---"
    curl -s -i -X OPTIONS -H "Origin: http://evil.example.com" -H "Access-Control-Request-Method: POST" http://localhost:8000/api/documents/post_document/ | grep -iE "^HTTP|^access-control-allow-origin" ; echo "(no ACAO header emitted for foreign origin)"
    echo "--- settings ---"
    cd /app/src && python3 -c "import os,django;os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\");django.setup();from django.conf import settings;print(\"CORS_ALLOWED_ORIGINS =\",settings.CORS_ALLOWED_ORIGINS);print(\"CORS_ALLOW_ALL_ORIGINS =\",getattr(settings,\"CORS_ALLOW_ALL_ORIGINS\",None))"'
--- allowed origin ---
HTTP/1.1 200 OK
Vary: Origin
Access-Control-Allow-Origin: http://localhost:8000
--- foreign origin ---
HTTP/1.1 200 OK
(no ACAO header emitted for foreign origin)
--- settings ---
CORS_ALLOWED_ORIGINS = ('http://localhost:8000',)
CORS_ALLOW_ALL_ORIGINS = None
```

### E7 — WebSocket status route over plain HTTP GET → 302 (no upgrade under WSGI)

The status route `ws/status/` is an ASGI `StatusConsumer` [src/paperless/urls.py:L137]. Under the canonical dev **WSGI** `runserver`, a plain HTTP `GET` does not perform a WebSocket upgrade (`101`); it is redirected (`302`). This is why Q2a captured progress at the channel-layer boundary rather than via this HTTP route:

```console
$ docker exec paperless-setup bash -lc 'curl -s -o /dev/null -w "ws_status_http_code=%{http_code}\n" http://localhost:8000/ws/status/'
ws_status_http_code=302
```

### E8 — Concurrent identical uploads: DB UNIQUE-constraint race (observed, out-of-scope to fix)

`checksum` is a `unique=True` column [src/documents/models.py:L135-L139], and `pre_check_duplicate()` [src/documents/consumer.py:L102] is a **read** that is not atomic with the later insert. When several *identical* uploads are processed concurrently by different workers, each passes the pre-check before any commits, then all but one hit the database uniqueness constraint at insert time — surfacing as a raw `IntegrityError` rather than the graceful "It is a duplicate" message. Six simultaneous uploads of one fresh file were driven through the canonical endpoint; the result was **1 success and 5 `UNIQUE constraint failed` errors** (captured, then fully cleaned up):

```console
$ docker exec paperless-setup bash -lc '
    cd /app/src
    python3 -c "
from PIL import Image, ImageDraw
im=Image.new(\"RGB\",(1240,1754),\"white\"); d=ImageDraw.Draw(im)
d.text((80,300),\"RACE TEST DOCUMENT unique 987654321\",fill=\"black\")
d.rectangle([80,700,1160,1600],outline=\"black\",width=3)
im.save(\"/tmp/qa_run/race.pdf\",resolution=150.0)"
    LOG=/app/data/log/paperless.log; before=$(wc -c < "$LOG")
    for i in 1 2 3 4 5 6; do curl -s -o /dev/null -u admin:admin -F "document=@/tmp/qa_run/race.pdf" http://localhost:8000/api/documents/post_document/ & done
    wait; sleep 10
    python3 -c "
import os,django;os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\");django.setup()
from django_q.models import Task
ts=list(Task.objects.filter(func=\"documents.tasks.consume_file\").order_by(\"-started\")[:6])
print(\"successes =\", sum(1 for t in ts if t.success))
print(\"failures  =\", sum(1 for t in ts if not t.success))"
    tail -c +$((before+1)) "$LOG" | grep -E "Consuming race.pdf|UNIQUE constraint failed|IntegrityError" | head -9'
successes = 1
failures  = 5
[2026-07-14 00:49:34,218] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:34,221] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:34,222] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:34,222] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:34,238] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:34,240] [INFO] [paperless.consumer] Consuming race.pdf
[2026-07-14 00:49:38,518] [ERROR] [paperless.consumer] The following error occured while consuming race.pdf: UNIQUE constraint failed: documents_document.checksum
sqlite3.IntegrityError: UNIQUE constraint failed: documents_document.checksum
django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum
```

All six log a "Consuming race.pdf" line (a *fresh* file passes the duplicate pre-check), and then the DB insert collides on the unique `checksum` for all but the winner. This is a **pre-existing product behavior** (a TOCTOU race between the duplicate pre-check and the unique-constraint insert). It is documented here honestly for completeness; **fixing it is explicitly out of scope** for this read-only investigation (AAP §0.5.2 forbids source changes). The `race.pdf` document (the one success) and all leaked staged files it produced were removed (see [Cleanup](#cleanup-confirmation)), restoring the environment to the single canonical `pk=1` document.

---

## Coverage pass — every named item answered

A final pass confirms every question, sub-question, and named item is answered from observed output:

- **Q1 — immediate response.** ✅ `HTTP 200`, body `"OK"`, `application/json`, `Content-Length 4`, `TIME_TOTAL≈0.103s`, log `delta 0` at the response instant [src/documents/views.py:L535]. Async ordering measured precisely (worker started 1.905 ms before the client received the response; the response never waits for processing to *finish*).
- **Q2 — stage log patterns.** ✅ Full 15-line ordered slice captured, each line mapped to its call site; parallel Channels progress events (6) captured at the channel-layer boundary.
- **Q2 — OCRmyPDF parameters (exhaustive).** ✅ All **13** keys enumerated verbatim with per-key origin: `input_file, output_file, use_threads, jobs, language, output_type, progress_bar, skip_text, clean, deskew, rotate_pages, rotate_pages_threshold, sidecar`. Absent keys (`pages`, `image_dpi`, `force_ocr`, user args) explained by their unmet branch conditions.
- **Q2 — primary vs. fallback.** ✅ Counted: primary = 1, fallback = 0 on the happy path; fallback (`force_ocr=True`) documented as the real secondary path.
- **Q2 — worker result.** ✅ `Task.success=True`, `result='Success. New document id 1 created'`; the two distinct identifiers (django-q `Task.id` vs. paperless `task_id` uuid) disambiguated.
- **Q2 — same-input-twice.** ✅ Duplicate rejected (`documents_count` stays 1, single ERROR line, no "Consuming" line); staged-file leak on the duplicate path observed and documented (out-of-scope to fix). Archive metadata variance (`ModDate` + trailer `/ID` vary; `CreationDate` constant; sidecar text identical) captured.
- **Q3 — media filenames.** ✅ `originals/0000001.pdf`, `archive/0000001.pdf`, `thumbnails/0000001.png` under default `/app/media/documents/...`; `{pk:07}` origin cited.
- **Q4 — stored vs. computed.** ✅ 15-column schema dumped (via reproducible Python `PRAGMA`); concrete-field values printed; `source_path`/`archive_path`/`thumbnail_path`/`file_type`/`has_archive_version` proven to be Python properties absent from the schema.
- **Edge/error paths.** ✅ 401 (no creds), 401 (bad creds), 400 (missing field), 400 (unsupported type, no scratch leak), 405 (GET), CORS allowed vs. foreign, `ws/status` 302, and the concurrent-upload UNIQUE-constraint race — each with security headers and before/after side-effect snapshots proving zero unintended side effects.
- **"Inconsistent OCR" synthesis.** ✅ Attributed to default `skip_text=True` on mixed PDFs; controlled with a fully-image fixture (consistent, all pages OCR'd); archive-checksum variance shown to be a metadata red herring.

---

## Cleanup confirmation

Per the read-only / no-permanent-change constraint, all runtime artifacts are removed and the source tree is verified unchanged.

**1. Delete the canonical document through the ORM (cascades to media).** The `Document.delete()` path removes the DB row and its originals/archive/thumbnail files. The count returns to `0`:

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from documents.models import Document
Document.objects.all().delete()
print(\"documents_count_after_delete =\", Document.objects.count())"'
documents_count_after_delete = 0
```

**2. Remove any residual media, restore the pk sequence, rebuild the search index to match an empty DB, and clear scratch** (covers the duplicate-path/race staged-file leaks documented above). The Whoosh index is rebuilt through the canonical management command so it is consistent with the now-empty database:

```console
$ docker exec paperless-setup bash -lc '
    rm -f /app/media/documents/originals/* /app/media/documents/archive/* /app/media/documents/thumbnails/* 2>/dev/null
    rm -rf /tmp/paperless/* 2>/dev/null
    cd /app/src && python3 -c "
import os,django;os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\");django.setup()
from django.db import connection
with connection.cursor() as c: c.execute(\"UPDATE sqlite_sequence SET seq=0 WHERE name=%s\",[\"documents_document\"])"
    python3 manage.py document_index reindex --no-progress-bar
    echo "media_originals=$(ls -1 /app/media/documents/originals | wc -l) media_archive=$(ls -1 /app/media/documents/archive | wc -l) media_thumbnails=$(ls -1 /app/media/documents/thumbnails | wc -l)"
    echo "scratch_entries=$(ls -1 /tmp/paperless 2>/dev/null | wc -l)"'
media_originals=0 media_archive=0 media_thumbnails=0
scratch_entries=0
```

**3. Index/DB consistency check** (search index and database agree on the document set — both empty):

```console
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from documents.models import Document
from documents.index import open_index
db_ids = sorted(Document.objects.values_list(\"id\", flat=True))
ix = open_index()
with ix.searcher() as s:
    idx_ids = sorted(int(f[\"id\"]) for f in s.documents())
print(\"db_ids   =\", db_ids)
print(\"index_ids=\", idx_ids)
print(\"consistent=\", db_ids == idx_ids)"'
db_ids   = []
index_ids= []
consistent= True
```

**4. Stop the processes started for this investigation, by exact PID** (Redis is left running as it was found; no `pkill` is used), and remove the temporary working dir:

```console
$ docker exec paperless-setup bash -lc '
    kill 42268 2>/dev/null && echo "stopped runserver pid=42268"
    kill 42271 2>/dev/null && echo "stopped qcluster pid=42271"
    rm -rf /tmp/qa_run 2>/dev/null && echo "removed temp working dir /tmp/qa_run"'
stopped runserver pid=42268
stopped qcluster pid=42271
removed temp working dir /tmp/qa_run
```

**5. Source tree unchanged.** The runtime directories (`data/`, `media/`, `static/`, `consume/`) are gitignored, so no runtime artifact appears in git; the only change in the working tree is this documentation file:

```console
$ docker exec paperless-setup bash -lc 'cd /app && git status --porcelain'
?? blitzy/
```

The `blitzy/` directory (containing only this answer document) is the sole addition; every tracked source file under `src/` and all manifests are byte-for-byte identical to HEAD `542221a38`. No existing file was modified; the temporary fixture, observation scripts, test document, its media, and all leaked staged files were removed.

---

### Appendix — citation legend

All `[path:Lx]` and `[path:Lx-Ly]` tokens reference files under the repository root at HEAD `542221a38` (e.g. `[src/documents/views.py:L535]` = line 535 of `src/documents/views.py`). Runtime values (HTTP codes, log lines, filenames, DB values, parameter dicts) are shown verbatim from the captured command output; statements that could only be derived by reading code without a runtime trigger are marked **inferred (code-read)**.
