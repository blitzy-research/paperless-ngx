# paperless-ngx — Runtime Behavior of the Document-Ingestion / OCR Pipeline

> **Motivating problem (verbatim intent):** a user testing paperless-ngx locally reported **"inconsistent OCR results"** when processing multi-page PDFs. This document captures the *actual runtime behavior* of the ingestion/OCR pipeline so the four questions below can be answered from **observed output**, not from reading code alone.

## Methodology (run-first, then write)

This is a **read-only, run-first investigation**: the pipeline was **run first**, and every answer below was written **from the observed runtime output**, never from reading the code alone. Each answer was produced by **actually running** paperless-ngx in its **default/canonical configuration** inside the project's own Docker container and driving the **canonical upload entry point** `POST /api/documents/post_document/` with a real superuser and a real multipart `document` field. For every claim you will find the **exact command** and its **complete, unedited output** in a fenced block, plus a `[path:Lx]` citation into the source at HEAD `542221a38`. Statements that could only be obtained by reading the code (never triggered at runtime) are explicitly labelled **inferred (code-read)**. Values obtained outside the canonical entry point are labelled **non-canonical**.

**Isolated, disposable data namespace.** So that the investigation neither reads nor mutates any pre-existing state and leaves the repository byte-for-byte unchanged, the run relocates only the three *storage-location* settings — `PAPERLESS_DATA_DIR` [src/paperless/settings.py:L66], `PAPERLESS_MEDIA_ROOT` [src/paperless/settings.py:L61], and `PAPERLESS_CONSUMPTION_DIR` [src/paperless/settings.py:L78] — into a fresh `mktemp -d` directory created with mode `0700`. This is **not** a change to any answer-bearing setting: `PAPERLESS_OCR_MODE`, `PAPERLESS_FILENAME_FORMAT`, and `PAPERLESS_DBHOST` (the only three settings that would change Q1–Q4) remain **unset** at their defaults (proven in §3 below). Relocating storage yields a genuinely first-run SQLite database (so the created document is truly `pk=1`), a run-scoped log file (so every `grep` counts only this run's records), and a single-directory teardown (`rm -rf` of one `mktemp` dir — no global `delete()`, no wildcards, no disclosure of unrelated state).

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

The runtime is the user-provided image, pinned here by its immutable **RepoDigest** and **ImageID** via `docker inspect` — the exact provenance of every observation in this document. Its `/app` is **bind-mounted** from the working tree (so edits to this `.md` and the repository are the same bytes the container sees), and the Django project runs under `/app/src`:

```console
$ docker inspect paperless-setup --format 'Image={{.Config.Image}}'
Image=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01
$ docker inspect paperless-setup --format 'ImageID={{.Image}}  WorkingDir={{.Config.WorkingDir}}'
ImageID=sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f  WorkingDir=/app/src
$ docker inspect paperless-setup --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}} (rw={{.RW}}){{println}}{{end}}'
bind /tmp/blitzy/paperless-ngx/blitzy-87b095e2-5299-40e2-89c5-c0b45c0c4edc_fa033e -> /app (rw=true)
$ IMG=$(docker inspect paperless-setup --format '{{.Config.Image}}')
$ docker image inspect "$IMG" --format 'RepoDigests={{.RepoDigests}}
Id={{.Id}}'
RepoDigests=[ghcr.io/scaleapi/swe-atlas@sha256:d4abe56dd5d1cb2632353baf06e9d80a704f147a70ac2ec15c2b78fc5fddfe15]
Id=sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f
```

The `Dockerfile` *intends* to provide the full native toolchain — it declares a `jbig2enc` builder stage [Dockerfile:L12] and a `qpdf` builder stage [Dockerfile:L13], and installs ghostscript [Dockerfile:L41], pngquant [Dockerfile:L61], tesseract-ocr + language packs [Dockerfile:L63-L68], and unpaper [Dockerfile:L70] onto the `python:3.9-slim-bullseye` base [Dockerfile:L18]. Rather than *assume* the Dockerfile's intent, the **actual running image is probed directly** below. It confirms tesseract / ocrmypdf / ghostscript / qpdf / unpaper / pngquant / pdftoppm are on `PATH`, but **`jbig2` and `jbig2enc` are NOT present** in this image. This document therefore makes **no** claim that jbig2 is available at runtime; its absence is treated as a real edge condition (exercised in the [coverage section](#coverage--edge-cases)):

```console
$ docker exec paperless-setup bash -lc 'whoami; echo debian $(cat /etc/debian_version); python3 --version; for b in tesseract ocrmypdf gs qpdf unpaper pngquant pdftoppm jbig2 jbig2enc; do p=$(command -v $b) && echo "$b PRESENT $p" || echo "$b MISSING"; done; echo ---; tesseract --version | head -1; gs --version; qpdf --version | head -1; unpaper --version | head -1; pngquant --version | head -1'
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
```

### 2. Python runtime & pinned dependencies

The pins match `requirements.txt`: ocrmypdf 13.4.3 [requirements.txt:L60], Django 4.0.4 [requirements.txt:L38], django-q 1.3.9 [requirements.txt:L37], djangorestframework 3.13.1 [requirements.txt:L39], channels 3.0.4 [requirements.txt:L23], channels-redis 3.4.0 [requirements.txt:L22], redis 3.5.3 [requirements.txt:L84], pikepdf 5.1.1 [requirements.txt:L65].

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

The three settings that would change the answers are **all unset**, so the code takes its default branches: `PAPERLESS_OCR_MODE` (→ `'skip'`) [src/paperless/settings.py:L522], `PAPERLESS_FILENAME_FORMAT` (→ `None`) [src/paperless/settings.py:L584], `PAPERLESS_DBHOST` (→ SQLite) [src/paperless/settings.py:L304]:

```console
$ source /tmp/.inv_env; for v in PAPERLESS_OCR_MODE PAPERLESS_FILENAME_FORMAT PAPERLESS_DBHOST; do
    printf "%s=[%s]\n" "$v" "${!v-<UNSET>}"; done
PAPERLESS_OCR_MODE=[<UNSET>]
PAPERLESS_FILENAME_FORMAT=[<UNSET>]
PAPERLESS_DBHOST=[<UNSET>]
```

The only environment variables set for this run are (a) the pre-existing `PAPERLESS_DISABLE_DBHANDLER=true` — **not referenced** by the `LOGGING` config (the `file_paperless` handler is unconditional [src/paperless/settings.py:L392-L395] and the `paperless` logger is DEBUG [src/paperless/settings.py:L409]), so file logging (the basis for Q2) is unaffected — and (b) the three **storage-location** overrides that form the disposable namespace. These relocate *where* data is written but change **no** answer-bearing behaviour, and are disclosed here in full:

```console
$ source /tmp/.inv_env; for v in PAPERLESS_DATA_DIR PAPERLESS_MEDIA_ROOT PAPERLESS_CONSUMPTION_DIR PAPERLESS_DISABLE_DBHANDLER; do
    printf "%s=[%s]\n" "$v" "${!v-<UNSET>}"; done
PAPERLESS_DATA_DIR=[/tmp/inv_LVzCw9Qz/data]
PAPERLESS_MEDIA_ROOT=[/tmp/inv_LVzCw9Qz/media]
PAPERLESS_CONSUMPTION_DIR=[/tmp/inv_LVzCw9Qz/consume]
PAPERLESS_DISABLE_DBHANDLER=[true]
```

### 4. Bring up the stack (Redis → migrate → superuser → web server → worker)

Default broker/Channels backend is `PAPERLESS_REDIS=redis://localhost:6379` [paperless.conf.example:L10], satisfied by the local `redis-server`. Against the **fresh isolated database** the full migration set is applied from scratch — proving a clean, first-run DB with nothing pre-existing in this namespace:

```console
$ redis-cli ping
PONG
$ source /tmp/.inv_env && cd /app/src && python3 manage.py migrate 2>&1 | (head -8; echo ...; tail -8)
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
...
  Applying paperless_mail.0012_alter_mailrule_assign_tags... OK
  Applying paperless_mail.0009_alter_mailrule_action_alter_mailrule_folder... OK
  Applying paperless_mail.0013_merge_20220412_1051... OK
  Applying paperless_mail.0014_alter_mailrule_action... OK
  Applying sessions.0001_initial... OK
```

Authentication uses a **uniquely-named temporary superuser created with a randomly-generated secret** that is held only in the disposable namespace and **never published**. HTTP Basic is a default DRF auth class [src/paperless/settings.py:L118], so `-u "$INV_USER:$INV_PASS"` is a canonical authentication method; the credential is destroyed with the namespace at cleanup:

```console
$ INV_USER="inv_$(openssl rand -hex 4)"
$ INV_PASS="$(python3 -c 'import secrets; print(secrets.token_urlsafe(24))')"
$ source /tmp/.inv_env && cd /app/src \
    && DJANGO_SUPERUSER_PASSWORD="$INV_PASS" python3 manage.py createsuperuser --noinput \
         --username "$INV_USER" --email inv@example.invalid \
    && python3 manage.py shell -c "from django.contrib.auth.models import User; print([(u.username,u.is_superuser) for u in User.objects.all()])"
Superuser created successfully.
[('consumer', False), ('inv_df856218', True)]
```

The generated username for this run was `inv_df856218`; its 24-byte URL-safe token password lives only in the `mktemp` namespace and is removed at teardown. The non-superuser `consumer` account is **not** imported pre-existing state — it is created by the data migration `User.objects.create(username="consumer")` [src/documents/migrations/0019_add_consumer_user.py:L10], so it appears in *any* freshly-migrated database (its presence here, without the shared setup's manually-created `admin`, is further proof the DB is genuinely fresh and isolated).

The Django web server and the **mandatory** `django-q` worker (`qcluster`) were started fresh, each detached with `stdout`/`stderr` **redirected** to a log file in the disposable namespace. `runserver` is invoked with `--noreload` (canonical flag; identical request behaviour) so the autoreloader does not restart the process when this `.md` is written into the watched `/app` tree. Because the image ships **no `ps` or `pgrep`**, the exact PID of each owned master process is resolved from `/proc/<pid>/cmdline` and recorded — these are the **only** two PIDs signalled at teardown:

```console
$ docker exec -d paperless-setup bash -lc 'source /tmp/.inv_env && cd /app/src && exec python3 manage.py runserver 0.0.0.0:8000 --noreload > "$INV/work/runserver.log" 2>&1'
$ docker exec -d paperless-setup bash -lc 'source /tmp/.inv_env && cd /app/src && exec python3 manage.py qcluster > "$INV/work/qcluster.log" 2>&1'
$ docker exec paperless-setup bash -lc 'for d in /proc/[0-9]*; do cl=$(tr "\0" " " < "$d/cmdline" 2>/dev/null); case "$cl" in *"manage.py runserver 0.0.0.0:8000 --noreload"*) echo "OWNED runserver PID=${d#/proc/}";; esac; done'
OWNED runserver PID=24375
$ docker exec paperless-setup bash -lc 'grep -l "manage.py qcluster" /proc/[0-9]*/cmdline 2>/dev/null | head -1 | sed -E "s#/proc/([0-9]+)/cmdline#OWNED qcluster PID=\1#"'
OWNED qcluster PID=24376
```

Startup banners (from the redirected log files in the namespace):

```console
$ cat $INV/work/runserver.log
Performing system checks...

System check identified no issues (0 silenced).
July 13, 2026 - 18:13:03
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
[13/Jul/2026 18:13:08] "GET / HTTP/1.1" 302 0

$ cat $INV/work/qcluster.log
18:13:03 [Q] INFO Q Cluster island-xray-pluto-august starting.
18:13:03 [Q] INFO Process-1:1 ready for work at 24391
18:13:03 [Q] INFO Process-1:2 ready for work at 24392
18:13:03 [Q] INFO Process-1:3 ready for work at 24393
18:13:03 [Q] INFO Process-1:4 ready for work at 24394
18:13:03 [Q] INFO Process-1:5 ready for work at 24395
18:13:03 [Q] INFO Process-1:6 ready for work at 24396
18:13:03 [Q] INFO Process-1:7 ready for work at 24397
18:13:03 [Q] INFO Process-1:8 ready for work at 24398
18:13:03 [Q] INFO Process-1:9 ready for work at 24399
18:13:03 [Q] INFO Process-1:10 ready for work at 24400
18:13:03 [Q] INFO Process-1:11 ready for work at 24401
18:13:03 [Q] INFO Process-1:12 monitoring at 24402
18:13:03 [Q] INFO Process-1 guarding cluster island-xray-pluto-august
18:13:03 [Q] INFO Process-1:13 pushing tasks at 24403
18:13:03 [Q] INFO Q Cluster island-xray-pluto-august running.
```

`qcluster` came up with **11 worker processes** (`Process-1:1..11`). Note carefully: the OCRmyPDF `jobs` parameter observed in Q2 (`jobs=11`) is **not** the worker count — it is `THREADS_PER_WORKER`, computed by a **separate** formula. Two independent settings-formulas each evaluate to 11 on this 128-CPU host and must not be conflated:

- `TASK_WORKERS = max(floor(sqrt(cpu_count)), 1) = max(floor(sqrt(128)), 1) = 11` — the number of `qcluster` worker *processes* [src/paperless/settings.py:L427-L436, src/paperless/settings.py:L438].
- `THREADS_PER_WORKER = max(floor(cpu_count / TASK_WORKERS), 1) = max(floor(128 / 11), 1) = 11` — the per-worker thread budget, and it is *this* value that is passed to OCRmyPDF as `jobs` [src/paperless/settings.py:L460-L466, src/paperless/settings.py:L469-L471; src/paperless_tesseract/parsers.py:L149].

The two coincide at 11 only because the host has 128 CPUs; they diverge on other core counts (e.g. at 8 cores `TASK_WORKERS=floor(sqrt(8))=2` but `THREADS_PER_WORKER=floor(8/2)=4`). The equality here is a numeric coincidence, not a causal link.

### 5. Runtime settings the pipeline will use (printed, not assumed)

```console
$ source /tmp/.inv_env && cd /app/src && python3 manage.py shell -c "
import multiprocessing
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
print(\"TASK_WORKERS       =\", settings.TASK_WORKERS)
print(\"THREADS_PER_WORKER =\", settings.THREADS_PER_WORKER)
print(\"cpu_count()        =\", multiprocessing.cpu_count())
print(\"PAPERLESS_FILENAME_FORMAT =\", repr(settings.PAPERLESS_FILENAME_FORMAT))
print(\"DB ENGINE        =\", settings.DATABASES[\"default\"][\"ENGINE\"])
print(\"DB NAME          =\", settings.DATABASES[\"default\"][\"NAME\"])"
MEDIA_ROOT       = /tmp/inv_LVzCw9Qz/media
ORIGINALS_DIR    = /tmp/inv_LVzCw9Qz/media/documents/originals
ARCHIVE_DIR      = /tmp/inv_LVzCw9Qz/media/documents/archive
THUMBNAIL_DIR    = /tmp/inv_LVzCw9Qz/media/documents/thumbnails
LOGGING_DIR      = /tmp/inv_LVzCw9Qz/data/log
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
TASK_WORKERS       = 11
THREADS_PER_WORKER = 11
cpu_count()        = 128
PAPERLESS_FILENAME_FORMAT = None
DB ENGINE        = django.db.backends.sqlite3
DB NAME          = /tmp/inv_LVzCw9Qz/data/db.sqlite3
```

These confirm the defaults cited throughout, and that the isolated namespace relocated only storage (`MEDIA_ROOT`/`LOGGING_DIR`/DB path point into `/tmp/inv_LVzCw9Qz`): `OCR_MODE='skip'` [src/paperless/settings.py:L522], `OCR_LANGUAGE='eng'` [src/paperless/settings.py:L514], `OCR_OUTPUT_TYPE='pdfa'` [src/paperless/settings.py:L518], `OCR_CLEAN='clean'` [src/paperless/settings.py:L526], `OCR_DESKEW=True` [src/paperless/settings.py:L528], `OCR_ROTATE_PAGES=True` [src/paperless/settings.py:L530], `OCR_ROTATE_PAGES_THRESHOLD=12.0` [src/paperless/settings.py:L532-L533], `OCR_PAGES=0` [src/paperless/settings.py:L510], `OCR_USER_ARGS='{}'` [src/paperless/settings.py:L541], `TASK_WORKERS=11` [src/paperless/settings.py:L438], `THREADS_PER_WORKER=11` [src/paperless/settings.py:L469-L471], `PAPERLESS_FILENAME_FORMAT=None` [src/paperless/settings.py:L584], SQLite backend [src/paperless/settings.py:L299-L300]. `SCRATCH_DIR=/tmp/paperless` is the default [src/paperless/settings.py:L84] and is deliberately left un-isolated: paperless allocates a fresh `tempfile.mkdtemp` subdirectory there per document and removes it after processing (seen in the Q2 log), so it needs no relocation.


### 6. Fixture that forces OCR (multi-page, image-only PDF)

Under the default `OCR_MODE='skip'`, OCRmyPDF is invoked with `skip_text=True`, which **copies text-bearing pages through unchanged, OCRing only pages with no text layer** [src/paperless_tesseract/parsers.py:L157-L158]. To guarantee the OCR path is exercised on **every** page, the fixture is a **3-page, image-only PDF** (rasterized text, no text layer). It was generated with a temporary Pillow script inside the disposable namespace (`$INV/work/make_fixture.py`, removed at cleanup):

```python
# $INV/work/make_fixture.py  (temporary; removed with the namespace at cleanup)
import sys
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
out = sys.argv[1]
pages[0].save(out, save_all=True, append_images=pages[1:], resolution=150.0)
print("WROTE", out, "pages=", len(pages))
```

The built file's **SHA-256 is recorded as its stable identity** — every repeat trial in Q2 re-hashes the file and confirms it is the *same bytes*, so the "same unchanged input" requirement is provable rather than assumed:

```console
$ docker exec paperless-setup bash -lc 'source /tmp/.inv_env && python3 "$INV/work/make_fixture.py" "$INV/work/ocr_test.pdf" && sha256sum "$INV/work/ocr_test.pdf" && ls -l "$INV/work/ocr_test.pdf"'
WROTE /tmp/inv_LVzCw9Qz/work/ocr_test.pdf pages= 3
d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e  /tmp/inv_LVzCw9Qz/work/ocr_test.pdf
-rw-r--r-- 1 root root 213603 Jul 13 18:13 /tmp/inv_LVzCw9Qz/work/ocr_test.pdf
```

Proof it is **image-only** (no extractable text layer, no embedded fonts) and multi-page. `pdftotext` returns only 3 bytes (page-break characters), which is far below the 50-character threshold `len(text_original) > 50` at `parse()` [src/paperless_tesseract/parsers.py:L234-L236], so `original_has_text=False` and OCR runs on every page:

```console
$ docker exec paperless-setup bash -lc 'source /tmp/.inv_env
    printf "pdftotext_chars = "; pdftotext "$INV/work/ocr_test.pdf" - 2>/dev/null | wc -c
    echo "--- pdffonts (empty table => no embedded fonts) ---"; pdffonts "$INV/work/ocr_test.pdf"
    echo "--- pdfinfo ---"; pdfinfo "$INV/work/ocr_test.pdf" | grep -E "Pages|Page size"
    cd /app/src && python3 -c "import magic,sys; print(\"python-magic mime =\", magic.from_file(sys.argv[1], mime=True))" "$INV/work/ocr_test.pdf"'
pdftotext_chars = 3
--- pdffonts (empty table => no embedded fonts) ---
name                                 type              encoding         emb sub uni object ID
------------------------------------ ----------------- ---------------- --- --- --- ---------
--- pdfinfo ---
Pages:          3
Page size:      595.2 x 841.92 pts (A4)
python-magic mime = application/pdf
```

(The `pdffonts` table is empty — no embedded fonts — confirming a pure image PDF; `python-magic` detects `application/pdf`, the mime the consumer sees. The SHA-256 `d3ec86b9…4f1e4e` is **not** claimed to be reproducible across independent rebuilds — Pillow embeds a creation timestamp — but it is the fixed identity of *this* fixture, reused byte-for-byte across all trials below.)

---

## Q1 — Immediate (synchronous) HTTP response

> **Question:** What HTTP response status and body does the client receive **immediately** after submitting the upload (i.e., the synchronous API response, before any asynchronous processing completes)?

**Answer: HTTP `200 OK` with the JSON body `"OK"` (4 bytes).** This is the **asynchronous enqueue acknowledgement** returned by `PostDocumentView.post()` via `return Response("OK")` [src/documents/views.py:L535]. It does **not** wait for processing to finish; in this run it was received before the first worker stage log line was written (log-delta `0`, below). This exact ordering is not guaranteed by the code — the `qcluster` worker may dequeue and begin concurrently — but the response is definitively the *enqueue acknowledgement*, never the processing result.

**Code path.** The route `^documents/post_document/` sits under the `^api/` prefix [src/paperless/urls.py:L40] and binds to `PostDocumentView.as_view()` [src/paperless/urls.py:L56-L60] (imported at [src/paperless/urls.py:L16]). `class PostDocumentView(GenericAPIView)` [src/documents/views.py:L491] sets `permission_classes=(IsAuthenticated,)` [src/documents/views.py:L493], `serializer_class=PostDocumentSerializer` [src/documents/views.py:L494], `parser_classes=(MultiPartParser,)` [src/documents/views.py:L495]. `post()` [src/documents/views.py:L497] validates the serializer [src/documents/views.py:L499-L500], writes the upload to a `paperless-upload-*` temp file in `SCRATCH_DIR` [src/documents/views.py:L512-L519], generates a correlation id `task_id = str(uuid.uuid4())` [src/documents/views.py:L521], enqueues `async_task("documents.tasks.consume_file", …, task_id=task_id, …)` [src/documents/views.py:L523-L533], and finally **`return Response("OK")`** [src/documents/views.py:L535]. The multipart contract is `PostDocumentSerializer(serializers.Serializer)` [src/documents/serialisers.py:L413] with a required `document = serializers.FileField(...)` [src/documents/serialisers.py:L415-L418] plus optional `title` [src/documents/serialisers.py:L420], `correspondent` [src/documents/serialisers.py:L426], `document_type` [src/documents/serialisers.py:L434], and `tags` [src/documents/serialisers.py:L442]; the endpoint is documented at [docs/api.rst:L233-L235].

**Command & complete output.** A small observation script (`$INV/work/capture_progress.py`, removed at cleanup) drives the **canonical** endpoint with `requests` under the temporary superuser, records the immediate HTTP status/body/time and the log-line delta at the instant the response returns, and — so the `STARTING/WORKING/SUCCESS` milestones can also be captured — subscribes to the `"status_updates"` Channels group over the configured `channels-redis` layer (the exact group `StatusConsumer` joins [src/paperless/consumers.py:L17-L20] receiving the exact payload `_send_progress()` emits [src/documents/consumer.py:L64-L76]). The channel-layer subscription is used because `manage.py runserver` serves **WSGI only** and cannot perform the WebSocket upgrade (demonstrated in the [coverage section](#coverage--edge-cases)); it observes the identical `group_send` events a browser WebSocket would, and is labelled as captured at the channel layer rather than through a browser socket.

```python
# $INV/work/capture_progress.py  (temporary; removed with the namespace at cleanup)
import os, sys, time, json, hashlib
sys.path.insert(0, "/app/src")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django; django.setup()
import requests
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer

USER, PW, INV = os.environ["INV_USER"], os.environ["INV_PASS"], os.environ["INV"]
FIX, LOG = f"{INV}/work/ocr_test.pdf", f"{INV}/data/log/paperless.log"
cl = get_channel_layer()
ch = async_to_sync(cl.new_channel)()
async_to_sync(cl.group_add)("status_updates", ch)          # same group StatusConsumer joins

def logcount():
    try:
        return sum(1 for _ in open(LOG, "rb"))
    except FileNotFoundError:
        return 0

print("progress observed via configured channels-redis layer (same group_send transport as StatusConsumer)")
print("FIXTURE_SHA256", hashlib.sha256(open(FIX, "rb").read()).hexdigest())
before = logcount(); print("log_lines_before_upload", before)
t0 = time.time()
r = requests.post("http://localhost:8000/api/documents/post_document/",
                  files={"document": ("ocr_test.pdf", open(FIX, "rb"), "application/pdf")}, auth=(USER, PW))
dt = time.time() - t0
print("HTTP_STATUS", r.status_code)
print("HTTP_BODY", repr(r.text))
print(f"TIME_TOTAL {dt:.3f}s")
after = logcount(); print("log_lines_immediately_after_response", after, "delta", after - before)
n = 0
deadline = time.time() + 40
while time.time() < deadline:
    try:
        ev = async_to_sync(cl.receive)(ch)
    except Exception:
        break
    data = ev.get("data", {})
    print("PROGRESS_EVENT", json.dumps(data)); n += 1
    if data.get("status") in ("SUCCESS", "FAILED"):
        break
print("PROGRESS_EVENT_COUNT", n)
```

```console
$ docker exec paperless-setup bash -lc 'source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 "$INV/work/capture_progress.py"'
progress observed via configured channels-redis layer (same group_send transport as StatusConsumer)
FIXTURE_SHA256 d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e
log_lines_before_upload 1
HTTP_STATUS 200
HTTP_BODY '"OK"'
TIME_TOTAL 0.106s
log_lines_immediately_after_response 1 delta 0
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
PROGRESS_EVENT {"filename": "ocr_test.pdf", "task_id": "6c8c075e-151a-42c4-beb2-06dd7e982fef", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
PROGRESS_EVENT_COUNT 6
```

**Why this is the async ack, not the result.** The `HTTP_STATUS 200` / `HTTP_BODY '"OK"'` came back in `TIME_TOTAL 0.106s`, and the paperless log line count was **unchanged at that instant** (`1 → 1`, `delta 0`): no worker stage log had yet been written. The `200/"OK"` therefore corresponds to `async_task(...)` + `return Response("OK")` [src/documents/views.py:L523-L535]; the processing (Q2) runs afterwards in the `qcluster` worker (`consume_file` [src/documents/tasks.py:L184]) and must **not** be conflated with processing completion. The `task_id` shown in every progress event (`6c8c075e-151a-42c4-beb2-06dd7e982fef`) is exactly the `str(uuid.uuid4())` generated at [src/documents/views.py:L521] and threaded through to `_send_progress()` — the correlation handle between this synchronous response and the asynchronous milestones in Q2, whose terminal `SUCCESS` event carries `document_id: 1`.


---

## Q2 — Processing-stage logs & OCRmyPDF parameters

> **Question:** As the document is processed, what are the key log-line patterns that show it moving through the pipeline's stages, and specifically **what exact parameters are passed to the OCRmyPDF invocation** (every parameter enumerated)?

**Logging setup.** The verbose formatter is `"[{asctime}] [{levelname}] [{name}] {message}"` [src/paperless/settings.py:L378]; the `paperless` logger writes at DEBUG to `LOGGING_DIR/paperless.log` via a `ConcurrentRotatingFileHandler` [src/paperless/settings.py:L392-L395,L409], while the root logger writes to the console [src/paperless/settings.py:L407]. Pipeline loggers are `paperless.consumer`, `paperless.parsing` [src/documents/parsers.py:L40,L287], and `paperless.parsing.tesseract` [src/paperless_tesseract/parsers.py:L24]. Every record additionally carries a per-document `group` correlation uuid via `LoggingMixin.log()` (`extra={"group": self.logging_group}`, uuid4 from `renew_logging_group`) [src/documents/loggers.py:L11-L12,L14,L21] — but the default verbose format string does **not** include `{group}`, so it is attached to the `LogRecord` yet **not rendered** in `paperless.log`.

### Q2a — Ordered stage log lines

Because the run uses an isolated, first-run log file, `cat` shows the **entire run-scoped log** after the single canonical upload of Q1 (no unrelated records; the leading line is the startup scheduled sanity check, the rest are this document's stages, in order):

```console
$ docker exec paperless-setup bash -lc 'cat "$INV/data/log/paperless.log"'
[2026-07-13 18:13:32,969] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
[2026-07-13 18:17:03,899] [INFO] [paperless.consumer] Consuming ocr_test.pdf
[2026-07-13 18:17:03,900] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 18:17:03,901] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 18:17:03,903] [DEBUG] [paperless.consumer] Parsing ocr_test.pdf...
[2026-07-13 18:17:03,927] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-j835nkgm
[2026-07-13 18:17:04,002] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-j835nkgm', 'output_file': '/tmp/paperless/paperless-t56o3ht6/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-t56o3ht6/sidecar.txt'}
[2026-07-13 18:17:07,530] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 18:17:07,531] [DEBUG] [paperless.consumer] Generating thumbnail for ocr_test.pdf...
[2026-07-13 18:17:07,535] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-t56o3ht6/archive.pdf[0] /tmp/paperless/paperless-t56o3ht6/convert.png
[2026-07-13 18:17:08,368] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-t56o3ht6/convert.png -out /tmp/paperless/paperless-t56o3ht6/thumb_optipng.png
[2026-07-13 18:17:09,874] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:17:09,877] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:17:09,900] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-j835nkgm
[2026-07-13 18:17:09,925] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-t56o3ht6
[2026-07-13 18:17:09,925] [INFO] [paperless.consumer] Document 2026-07-13 ocr_test consumption finished
```

Mapping each observed line to the code, in order (all inside `Consumer.try_consume_file()` [src/documents/consumer.py:L180] unless noted):

| Observed log line | Emitted by | Citation |
|-------------------|-----------|----------|
| `Consuming ocr_test.pdf` | `self.log("info", f"Consuming {self.filename}")` | [src/documents/consumer.py:L215] |
| `Detected mime type: application/pdf` | `self.log("debug", f"Detected mime type: {mime_type}")` | [src/documents/consumer.py:L221] |
| `Parser: RasterisedDocumentParser` | `self.log("debug", f"Parser: {type(document_parser).__name__}")` | [src/documents/consumer.py:L246] |
| `Parsing ocr_test.pdf...` | `self.log("debug", "Parsing {}...".format(self.filename))` | [src/documents/consumer.py:L260] |
| `Extracted text from PDF file ...` | pre-OCR text check `extract_text(None, document_path)` called from `parse()` [src/paperless_tesseract/parsers.py:L235]; the line is logged inside `extract_text` | [src/paperless_tesseract/parsers.py:L122] |
| `Calling OCRmyPDF with args: {...}` | `self.log("debug", f"Calling OCRmyPDF with args: {args}")` | [src/paperless_tesseract/parsers.py:L260] |
| `Using text from sidecar file` | `extract_text()` (sidecar had no `"[OCR skipped on page"`) | [src/paperless_tesseract/parsers.py:L107] |
| `Generating thumbnail for ocr_test.pdf...` | `self.log("debug", f"Generating thumbnail for {self.filename}...")` | [src/documents/consumer.py:L263] |
| `[paperless.parsing] Execute: convert ...` | thumbnail render — module-level `run_convert()` (logger `paperless.parsing`) | [src/documents/parsers.py:L143] |
| `[paperless.parsing.tesseract] Execute: optipng ...` | thumbnail optimize — `get_optimised_thumbnail()` via `self.log` (logger `paperless.parsing.tesseract`) | [src/documents/parsers.py:L333] |
| `Saving record to database` | `self.log("debug", "Saving record to database")` in `_store()` | [src/documents/consumer.py:L387] |
| `Document ... consumption finished` | `self.log("info", "Document {} consumption finished".format(document))` | [src/documents/consumer.py:L373] |

**Progress milestones are published to the Channels layer, not to `paperless.log`.** The `STARTING`/`WORKING`/`SUCCESS` milestones are emitted by `Consumer._send_progress()` [src/documents/consumer.py:L56] to the Channels group `"status_updates"` via `async_to_sync(...)(group_send(...))` [src/documents/consumer.py:L73-L76]; they are **not** written to the log file, which is why they do not appear in the `cat` output above. These are exactly the six events captured in Q1 (all carrying the same paperless `task_id` `6c8c075e-151a-42c4-beb2-06dd7e982fef`, with the terminal `SUCCESS` event carrying `document_id: 1`). Each observed event maps one-to-one to a `_send_progress()` call site:

| Observed event (from Q1 capture) | Emitting call site |
|---|---|
| `STARTING` `new_file` `0/100` `document_id: null` | `self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)` [src/documents/consumer.py:L202] |
| `WORKING` `parsing_document` `20/100` `document_id: null` | `self._send_progress(20, 100, "WORKING", MESSAGE_PARSING_DOCUMENT)` [src/documents/consumer.py:L259] |
| `WORKING` `generating_thumbnail` `70/100` `document_id: null` | `self._send_progress(70, 100, "WORKING", MESSAGE_GENERATING_THUMBNAIL)` [src/documents/consumer.py:L264] |
| `WORKING` `parse_date` `90/100` `document_id: null` | `self._send_progress(90, 100, "WORKING", MESSAGE_PARSE_DATE)` [src/documents/consumer.py:L274] |
| `WORKING` `save_document` `95/100` `document_id: null` | `self._send_progress(95, 100, "WORKING", MESSAGE_SAVE_DOCUMENT)` [src/documents/consumer.py:L294] |
| `SUCCESS` `finished` `100/100` `document_id: 1` | `self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)` [src/documents/consumer.py:L375] |

The `MESSAGE_*` string constants used above are defined at [src/documents/consumer.py:L43-L49].

**Worker task identity and return value.** `consume_file` [src/documents/tasks.py:L184] calls `Consumer().try_consume_file(...)` [src/documents/tasks.py:L236] and returns `"Success. New document id {} created".format(document.pk)` [src/documents/tasks.py:L247]. django-q persists this string as the task result. Querying the isolated namespace's `django_q.models.Task` rows after the run:

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 manage.py shell   # read-only query of the consume_file Task row (isolated namespace)
django_q Task.id     = f696dfb31b534c79b5487f29d5aa1364
name (task_name)     = ocr_test.pdf
func                 = documents.tasks.consume_file
started              = 2026-07-13 18:17:03.761380+00:00
stopped              = 2026-07-13 18:17:09.928976+00:00
success              = True
result               = 'Success. New document id 1 created'

--- all tasks recorded by the qcluster in this isolated run ---
  documents.tasks.train_classifier           success=True result=None
  documents.tasks.index_optimize             success=True result=None
  documents.tasks.sanity_check               success=True result='No issues detected.'
  paperless_mail.tasks.process_mail_accounts success=True result='No new documents were added.'
  documents.tasks.consume_file               success=True result='Success. New document id 1 created'
```

The `consume_file` task succeeded and its `result` string names **document id 1** — the same id carried by the terminal `SUCCESS` progress event above. Two **distinct** correlation identifiers exist for this single ingestion; they are not the same value:

- The **paperless `task_id`** — a `str(uuid.uuid4())` generated in `PostDocumentView.post()` [src/documents/views.py:L521] and passed as a task keyword argument via `async_task(..., task_id=task_id, ...)` [src/documents/views.py:L523-L533]. It flows into `consume_file(..., task_id=None)` [src/documents/tasks.py:L191] and onward into `Consumer().try_consume_file(..., task_id=task_id)` [src/documents/tasks.py:L243], and appears (dash-formatted) in every progress payload as `6c8c075e-151a-42c4-beb2-06dd7e982fef`. This is the id the frontend `StatusConsumer` correlates against.
- The **django-q internal `Task.id`** — a 32-character hex string (`f696dfb31b534c79b5487f29d5aa1364`) that django-q assigns to the persisted task row. It is unrelated to the paperless `task_id` above.

The four other successful rows (`train_classifier`, `index_optimize`, `sanity_check`, `process_mail_accounts`) are the scheduled maintenance jobs the `qcluster` executed during the isolated session; only `consume_file` performed the ingestion under investigation.


### Q2b — The OCRmyPDF invocation, every parameter enumerated

The args dict is built by `RasterisedDocumentParser.construct_ocrmypdf_parameters()` [src/paperless_tesseract/parsers.py:L135], logged verbatim by `parse()` [src/paperless_tesseract/parsers.py:L230] at the line `Calling OCRmyPDF with args: {...}` [src/paperless_tesseract/parsers.py:L260], immediately before `ocrmypdf.ocr(**args)` [src/paperless_tesseract/parsers.py:L261]. `parse()` also sets `os.environ["OMP_THREAD_LIMIT"]="1"` [src/paperless_tesseract/parsers.py:L232].

For a default-config, multi-page, **image-only PDF** (mime `application/pdf`), the observed dict has **exactly 13 keys**:

```python
{'input_file': '/tmp/paperless/paperless-upload-j835nkgm',
 'output_file': '/tmp/paperless/paperless-t56o3ht6/archive.pdf',
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
 'sidecar': '/tmp/paperless/paperless-t56o3ht6/sidecar.txt'}
```

This dict is the parsed form of the exact `Calling OCRmyPDF with args: {...}` log line shown verbatim in Q2a above (same isolated run, same temp paths `paperless-upload-j835nkgm` / `paperless-t56o3ht6`).

| # | Parameter | Observed value | Meaning / why present | Citation |
|---|-----------|----------------|-----------------------|----------|
| 1 | `input_file` | `/tmp/paperless/paperless-upload-j835nkgm` | the scratch temp file written by the upload view in `SCRATCH_DIR` (default `/tmp/paperless` [src/paperless/settings.py:L84]) | [src/paperless_tesseract/parsers.py:L144] |
| 2 | `output_file` | `/tmp/paperless/paperless-t56o3ht6/archive.pdf` | OCRmyPDF's archive output (`<tempdir>/archive.pdf`) | [src/paperless_tesseract/parsers.py:L145]; [src/paperless_tesseract/parsers.py:L249] |
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
| 13 | `sidecar` | `/tmp/paperless/paperless-t56o3ht6/sidecar.txt` | plain-text OCR output; added because `OCR_PAGES==0` takes the `else` branch (`sidecar` is incompatible with `pages`) | [src/paperless_tesseract/parsers.py:L181-L185]; [src/paperless/settings.py:L510] |

**Two edge facts, explicitly:**

- **`image_dpi` is omitted for a PDF.** It is added **only for image mime types** [src/paperless_tesseract/parsers.py:L187-L215], where `is_image()` is true only for `image/png|jpeg|tiff|bmp|gif` [src/paperless_tesseract/parsers.py:L64-L70]. A PDF is not an image mime, so **`image_dpi` is absent** — confirmed by its absence from the captured dict above.
- **`OCR_USER_ARGS` adds nothing.** Its default is the string `"{}"` [src/paperless/settings.py:L541]; `construct_ocrmypdf_parameters()` merges `json.loads(...)` → an empty dict, contributing no keys [src/paperless_tesseract/parsers.py:L217-L220].

Also absent (mode-dependent): `force_ocr`/`redo_ocr` are only set for `OCR_MODE in {force, redo}` or the safe fallback [src/paperless_tesseract/parsers.py:L155-L160]; under the default `skip` only `skip_text` is set.

### Q2c — Fallback invocation (force_ocr): **inferred (code-read)** — did not fire

If the primary `ocrmypdf.ocr()` raises `NoTextFoundException` or `InputFileError` [src/paperless_tesseract/parsers.py:L276], the consumer retries with `safe_fallback=True`, which sets **`force_ocr=True`** (`OCR_MODE=="force" or safe_fallback`) [src/paperless_tesseract/parsers.py:L155-L156], logs **`Fallback: Calling OCRmyPDF with args: {...}`** [src/paperless_tesseract/parsers.py:L297] and calls `ocrmypdf.ocr(**args)` again [src/paperless_tesseract/parsers.py:L298] (using `archive-fallback.pdf`/`sidecar-fallback.txt` [src/paperless_tesseract/parsers.py:L283-L284]; `force_ocr` replaces `skip_text`). **In this run the fallback did not fire** — the primary invocation succeeded (the sidecar yielded text, no `NoTextFoundException`), so there is **no** `Fallback: Calling OCRmyPDF` line:

```console
$ source /tmp/.inv_env && grep -c "Fallback: Calling OCRmyPDF with args" "$INV/data/log/paperless.log"
0
$ source /tmp/.inv_env && grep -c "Calling OCRmyPDF with args" "$INV/data/log/paperless.log"
1
```

Scoped to **this run's isolated log** (`$INV/data/log/paperless.log`, not the shared `/app/data/log`), the primary `Calling OCRmyPDF with args` line appears **exactly once** and the `Fallback: Calling OCRmyPDF with args` line appears **zero** times. The `force_ocr` fallback path is therefore labelled **inferred (code-read)**, cited to the lines shown; it was not exercised by the image-only fixture because that fixture succeeds on the primary `skip_text` call (the sidecar yields text, so no `NoTextFoundException`/`InputFileError` is raised at [src/paperless_tesseract/parsers.py:L276]).

### Q2d — Run-to-run behavior on the SAME input (reproducing the "inconsistency" faithfully)

Per the rules, the reported run-to-run inconsistency is probed by feeding the **same, unchanged** fixture (SHA-256 `d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e`, verified before every trial) into the canonical pipeline repeatedly — never a variant. The experiment has two parts because the pipeline **deduplicates by checksum**, which would otherwise mask any re-OCR behavior:

- **Part A — identical bytes, no between-run cleanup (deduplication path).** Every upload still returns the same synchronous `200`/`"OK"`, but the worker rejects each re-upload of an already-stored file. Duplicate detection is `Consumer.pre_check_duplicate()` [src/documents/consumer.py:L102-L112] (md5 at [src/documents/consumer.py:L104]; `Q(checksum=...) | Q(archive_checksum=...)` at [src/documents/consumer.py:L106]; `self._fail(..., "Not consuming {filename}: It is a duplicate.")` at [src/documents/consumer.py:L111-L112]), called from `try_consume_file` [src/documents/consumer.py:L213].
- **Part B — identical bytes, with exact between-run cleanup (forces re-OCR).** Deleting the created document and its media between runs makes the checksum novel again, so the OCR path executes afresh on the identical bytes each time. This is what actually exercises "run-to-run OCR" behavior on the same input.

Both parts were driven by a single observation script through the canonical upload endpoint (authenticated with the run-scoped generated-secret superuser, never reusable credentials):

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 $INV/work/capture_repeats.py
=== PART A: identical bytes re-uploaded WITHOUT between-run cleanup (deduplication) ===
pre-existing document pk=1 md5_checksum=bc7d1187ff19ca3d7058f89b9826c78c
[dedup trial 1] input SHA256 = d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e
  immediate HTTP=200 body='"OK"'
[dedup trial 2] input SHA256 = d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e
  immediate HTTP=200 body='"OK"'
  LOG: [2026-07-13 18:23:04,870] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
  LOG: [2026-07-13 18:23:04,944] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
documents after Part A (unchanged by dedup): [1]

=== PART B: same identical bytes, WITH exact between-run cleanup -> OCR re-runs each time ===
[repeat trial 1] input SHA256 = d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e  (docs now: 0)
  immediate HTTP=200 body='"OK"'
  -> pk=2 ocr_invocations=1 md5_checksum=bc7d1187ff19ca3d7058f89b9826c78c archive_checksum=e1bf9bc834733ac59eb45f671a8eb75f content==first_run:True
[repeat trial 2] input SHA256 = d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e  (docs now: 0)
  immediate HTTP=200 body='"OK"'
  -> pk=3 ocr_invocations=1 md5_checksum=bc7d1187ff19ca3d7058f89b9826c78c archive_checksum=7045d0c2defa5cbe0c387fe2a475abd4 content==first_run:True
[repeat trial 3] input SHA256 = d3ec86b9ac739e920cf91c3b6a4e11fd4bc6c68658513925981858cf4d4f1e4e  (docs now: 0)
  immediate HTTP=200 body='"OK"'
  -> pk=4 ocr_invocations=1 md5_checksum=bc7d1187ff19ca3d7058f89b9826c78c archive_checksum=34d1a7b5ce4ab596c98d83e857ad5595 content==first_run:True
```

**Part A** confirms the deduplication path: two identical-byte re-uploads each return the synchronous `200`/`'"OK"'`, but the worker rejects both (`It is a duplicate.`), and the document set is **unchanged** (`[1]`).

**Part B** is the actual same-input OCR repeat, and it separates two very different notions of "consistency":

| Trial (identical SHA-256 `d3ec86b9ac73…`) | pk | OCR invocations | source `md5_checksum` | OCR text content | PDF/A `archive_checksum` |
|---|---|---|---|---|---|
| init (Q1 run) | 1 | 1 | `bc7d1187…` | baseline | — |
| 1 | 2 | 1 | `bc7d1187…` (same) | `== first run: True` | `e1bf9bc834733ac59eb45f671a8eb75f` |
| 2 | 3 | 1 | `bc7d1187…` (same) | `== first run: True` | `7045d0c2defa5cbe0c387fe2a475abd4` |
| 3 | 4 | 1 | `bc7d1187…` (same) | `== first run: True` | `34d1a7b5ce4ab596c98d83e857ad5595` |

**What is deterministic, and what is not (observed):**

- **The OCR *result* is deterministic.** Every re-OCR of the identical bytes performed **exactly one** primary OCRmyPDF invocation (`ocr_invocations=1`), produced a **byte-identical original-file checksum** (`md5_checksum = bc7d1187…` every time), and yielded **identical extracted OCR text** (`content == first run: True` for all three repeats). There is **no** run-to-run OCR-text variance for this fully-image fixture.
- **The PDF/A archive artifact is *not* byte-reproducible.** The stored `archive_checksum` [src/documents/models.py:L143] **differs on every run** (`e1bf9bc8…`, `7045d0c2…`, `34d1a7b5…`). This is expected: `output_type='pdfa'` routes the output through Ghostscript PDF/A conversion [src/paperless_tesseract/parsers.py:L151], which embeds run-specific metadata (creation timestamp / document IDs), so the *container bytes* change while the *rendered/extracted content* does not.
- **The invocation parameters do not vary.** Each repeat logged a single `Calling OCRmyPDF with args` line whose values are read from Django settings by `construct_ocrmypdf_parameters()` [src/paperless_tesseract/parsers.py:L135]; because settings did not change, the parameters are identical to the 13-key dict enumerated in Q2b except for the necessarily-unique scratch temp paths (inferred from the config-derived construction, and consistent with the observed `ocr_invocations=1` per run).

Neither behavior reproduces the "inconsistent OCR results" the user reported for this **single, fully-image** PDF: the text output is stable across runs. That points away from run-to-run nondeterminism and toward the **`skip_text` per-page mechanism** on *mixed* (partially text-bearing) multi-page PDFs — developed in the Synthesis section below.


---

## Q3 — Generated media filenames

> **Question:** After processing finishes, what do the generated **archive PDF** and **thumbnail** filenames look like inside the media storage area?

**Answer (observed, canonical document pk=1): archive PDF = `archive/0000001.pdf`, thumbnail = `thumbnails/0000001.png`** (original = `originals/0000001.pdf`). The default naming is `{pk:07}` (the primary key zero-padded to 7 digits), `.pdf` for original/archive and `.png` for the thumbnail.

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 manage.py shell   # resolve the pk from the ORM, then `ls -l` the isolated MEDIA_ROOT/documents/{originals,archive,thumbnails}
resolved document PK = 1
--- originals ---
total 212
-rw-r--r-- 1 root root 213603 Jul 13 18:17 0000001.pdf
--- archive ---
total 116
-rw-r--r-- 1 root root 116353 Jul 13 18:17 0000001.pdf
--- thumbnails ---
total 40
-rw-r--r-- 1 root root 37493 Jul 13 18:17 0000001.png
```

The listing is scoped to the **isolated** `MEDIA_ROOT` (under `$INV/media`, not the shared `/app/media`), and the pk is resolved from the ORM rather than hard-coded. Only the single Q1 document exists (`pk=1`), so exactly one file appears in each subdirectory — the original (`213603` bytes), the OCRmyPDF PDF/A archive (`116353` bytes), and the PNG thumbnail (`37493` bytes) — all named `0000001` with the `{pk:07}` zero-padding.

**Code path.** Because `PAPERLESS_FILENAME_FORMAT` is `None` [src/paperless/settings.py:L584], `generate_filename()` [src/documents/file_handling.py:L128] does not enter the custom-format branch guarded by `if settings.PAPERLESS_FILENAME_FORMAT is not None` [src/documents/file_handling.py:L132] and instead takes its default branch `filename = f"{doc.pk:07}{counter_str}{filetype_str}"` [src/documents/file_handling.py:L193], where `counter_str=""` (counter is 0) [src/documents/file_handling.py:L186] and `filetype_str=".pdf" if archive_filename else doc.file_type` [src/documents/file_handling.py:L188] (and `doc.file_type` is `".pdf"` for a PDF). The consumer assigns these inside the `FileLock`/`transaction.atomic()` block: `document.filename = generate_unique_filename(document)` [src/documents/consumer.py:L316] and `document.archive_filename = generate_unique_filename(document, archive_filename=True)` [src/documents/consumer.py:L328-L331]. The thumbnail name comes from the computed property `Document.thumbnail_path` — `file_name = "{:07}.png".format(self.pk)` [src/documents/models.py:L274] joined to `THUMBNAIL_DIR` [src/documents/models.py:L278]. The post-save `update_filename_and_move_files` handler [src/documents/signals/handlers.py:L311-L312] is a **no-op** under default config (inferred from the code): it regenerates the identical `{pk:07}` name, so both `move_original` and `move_archive` evaluate false [src/documents/signals/handlers.py:L330-L331] and the handler returns without moving anything [src/documents/signals/handlers.py:L347-L349].

The stored column values line up exactly with the on-disk names, and demonstrate the relative-vs-absolute split (see Q4):

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 manage.py shell   # stored filename columns vs computed path @property, for the resolved pk
pk                        = 1
filename (stored)         = '0000001.pdf'
archive_filename (stored) = '0000001.pdf'
source_path (computed)    = /tmp/inv_LVzCw9Qz/media/documents/originals/0000001.pdf
archive_path (computed)   = /tmp/inv_LVzCw9Qz/media/documents/archive/0000001.pdf
thumbnail_path (computed) = /tmp/inv_LVzCw9Qz/media/documents/thumbnails/0000001.png
```

The **stored** columns hold only the relative basename `'0000001.pdf'` (`filename`, `archive_filename`), while `source_path`/`archive_path`/`thumbnail_path` are `@property` methods that prepend the (isolated) `MEDIA_ROOT` at runtime — the relative-vs-absolute split examined in Q4. Here the absolute paths resolve under the disposable namespace `/tmp/inv_LVzCw9Qz/media/documents/...`; under the shared default they would resolve under `/app/media/documents/...`.

---

## Q4 — Database-stored fields vs computed properties

> **Question:** In the document's database record, which fields are **actually stored in the database** (as opposed to filesystem-only computed metadata), and what values appear for the processed document?

**Answer.** The `documents_document` table has **15 physical columns**; several `Document` attributes commonly mistaken for stored data (`source_path`, `archive_path`, `thumbnail_path`, `file_type`, `has_archive_version`, and the `*_file` openers) are **`@property` methods computed at runtime and are not columns**. The many-to-many `tags` field is stored in a **separate through-table**, not as a column.

**Stored columns (from the ORM, with observed values for pk=1):**

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 manage.py shell   # iterate concrete (non-relational) fields + resolve the tags M2M, for the resolved pk
  id                       = 1
  correspondent_id         = None
  title                    = 'ocr_test'
  document_type_id         = None
  content                  = 'Canonical OCR test document\n\nPage 1 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 1: PAPERLESSOCR\nCanonical OCR test document\n\nPage 2 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 2: PAPERLESSOCR\nCanonical OCR test document\n\nPage 3 of 3\n\nThe quick brown fox jumps over\n\nthe lazy dog. 1234567890\n\nUnique marker line page 3: PAPERLESSOCR'
  mime_type                = 'application/pdf'
  checksum                 = 'bc7d1187ff19ca3d7058f89b9826c78c'
  archive_checksum         = 'ab63cfbf243405d8641b96d3cb69e26e'
  created                  = datetime.datetime(2026, 7, 13, 18, 17, 3, tzinfo=datetime.timezone.utc)
  modified                 = datetime.datetime(2026, 7, 13, 18, 17, 9, 899914, tzinfo=datetime.timezone.utc)
  storage_type             = 'unencrypted'
  added                    = datetime.datetime(2026, 7, 13, 18, 17, 9, 878458, tzinfo=datetime.timezone.utc)
  filename                 = '0000001.pdf'
  archive_filename         = '0000001.pdf'
  archive_serial_number    = None
  tags (M2M -> through table) = []
```

The `content` column holds the full extracted OCR text — visibly containing the `Unique marker line page 1/2/3: PAPERLESSOCR` markers from **all three** pages, confirming every page was OCR'd. `checksum` (`bc7d1187…`) is the MD5 of the original file (matching Q2d); `archive_checksum` (`ab63cfbf…`) is the MD5 of this run's PDF/A archive (a per-run value, per Q2d).

**Concrete DB schema (SQLite `PRAGMA table_info`):**

```console
$ source /tmp/.inv_env && cd /app/src && python3 -c "
import os, sqlite3
c = sqlite3.connect(os.path.join(os.environ['INV'], 'data', 'db.sqlite3'))
print('documents_document columns (name | type | notnull):')
for r in c.execute('PRAGMA table_info(documents_document)'):
    print(f'  {r[1]:24s} | {r[2]:14s} | notnull={r[3]}')
cols = [r[1] for r in c.execute('PRAGMA table_info(documents_document)')]
print()
print('is there a physical tags column on documents_document? ->', 'tags' in cols)"
documents_document columns (name | type | notnull):
  id                       | integer        | notnull=1
  title                    | varchar(128)   | notnull=1
  content                  | text           | notnull=1
  created                  | datetime       | notnull=1
  modified                 | datetime       | notnull=1
  correspondent_id         | integer        | notnull=0
  checksum                 | varchar(32)    | notnull=1
  added                    | datetime       | notnull=1
  storage_type             | varchar(11)    | notnull=1
  archive_serial_number    | integer        | notnull=0
  document_type_id         | integer        | notnull=0
  mime_type                | varchar(256)   | notnull=1
  archive_checksum         | varchar(32)    | notnull=0
  archive_filename         | varchar(1024)  | notnull=0
  filename                 | varchar(1024)  | notnull=0

is there a physical tags column on documents_document? -> False
```

The physical table has **exactly 15 columns**, and there is **no** `tags` column. The many-to-many `tags` field is instead stored in a **separate auto-generated junction table** `documents_document_tags`, physically distinct from `documents_document`:

```console
$ source /tmp/.inv_env && cd /app/src && python3 -c "
import os, sqlite3
c = sqlite3.connect(os.path.join(os.environ['INV'], 'data', 'db.sqlite3'))
print('CREATE TABLE sql:')
print(' ', c.execute(\"SELECT sql FROM sqlite_master WHERE name='documents_document_tags'\").fetchone()[0])
print('columns:')
for r in c.execute('PRAGMA table_info(documents_document_tags)'):
    print(f'  {r[1]:14s} | {r[2]}')
print('rows for document_id=1:', c.execute('SELECT * FROM documents_document_tags WHERE document_id=1').fetchall())
print('total rows in through-table:', c.execute('SELECT COUNT(*) FROM documents_document_tags').fetchone()[0])"
CREATE TABLE sql:
  CREATE TABLE "documents_document_tags" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "document_id" integer NOT NULL REFERENCES "documents_document" ("id") DEFERRABLE INITIALLY DEFERRED, "tag_id" integer NOT NULL REFERENCES "documents_tag" ("id") DEFERRABLE INITIALLY DEFERRED)
columns:
  id             | integer
  document_id    | integer
  tag_id         | integer
rows for document_id=1: []
total rows in through-table: 0
```

So `Document.tags` (`models.ManyToManyField` [src/documents/models.py:L128]) is realized as the junction table `documents_document_tags` with its own `id` PK plus two foreign keys — `document_id` → `documents_document(id)` and `tag_id` → `documents_tag(id)`. For this untagged document the through-table holds **0 rows**, matching the empty `tags` list in the ORM dump above. This is why `tags` never appears as a column on `documents_document` and cannot be read as a scalar field.

Mapping each `documents_document` column to its model field [src/documents/models.py] (FK fields carry the `_id` suffix at the DB layer):

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
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 manage.py shell   # evaluate the computed @property attributes at runtime, for the resolved pk
file_type           = '.pdf'
source_path         = /tmp/inv_LVzCw9Qz/media/documents/originals/0000001.pdf
archive_path        = /tmp/inv_LVzCw9Qz/media/documents/archive/0000001.pdf
thumbnail_path      = /tmp/inv_LVzCw9Qz/media/documents/thumbnails/0000001.png
has_archive_version = True
source_file  (opener returns) = BufferedReader
archive_file (opener returns) = BufferedReader
thumbnail_file (opener returns) = BufferedReader
```

The three `*_path` values are absolute paths composed at runtime under the (isolated) `MEDIA_ROOT`; none is a stored column. The three `*_file` attributes are opener properties — evaluated here, each **returns an open `BufferedReader`** (i.e. `open(<path>, "rb")`), observed at runtime rather than merely read from source.

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
- For an **identical** input file, the **OCR result is deterministic** (Q2d, observed): every re-OCR of the same bytes performed exactly one OCRmyPDF invocation (`ocr_invocations=1`), produced the same original-file checksum, and yielded **identical extracted text** (`content == first run` on all repeats). There is no run-to-run OCR-text variance for the same bytes.
- The invocation **parameters do not vary run-to-run**: they are read from Django settings by `construct_ocrmypdf_parameters()` [src/paperless_tesseract/parsers.py:L135] and settings did not change, so the args dict matches the 13-key dict of Q2b except for the unavoidably-unique scratch temp paths (inferred from the config-derived construction, consistent with the observed single invocation per run). So "inconsistency" does **not** come from varying parameters.
- The **PDF/A archive bytes are *not* reproducible**, however (Q2d, observed): `archive_checksum` [src/documents/models.py:L143] differed on every identical-input run, because `output_type='pdfa'` [src/paperless_tesseract/parsers.py:L151] routes through Ghostscript, which embeds run-specific metadata. This is a property of the archive *container*, not of the OCR *text*, and is not the "inconsistency" a user would notice in recognized content.
- Re-uploading the same file does **not** create divergent documents: it is deduplicated by checksum [src/documents/consumer.py:L102-L112], so exactly one document exists for a given input.

**Inferred explanation (grounded in the observed `skip_text=True`):** `skip_text` instructs OCRmyPDF to **skip pages that already contain text and copy them through unchanged, OCRing only pages without a text layer**. The sidecar (from which paperless prefers to read the text) therefore **only contains text for the pages that were actually OCR'd** — stated directly in the code comment at [src/paperless_tesseract/parsers.py:L104-L106] ("The sidecar file will only contain text for OCR'ed pages."). On the fully-image fixture used here, **no** page had text, so every page was OCR'd, the sidecar carried no skip marker, and paperless used it (`Using text from sidecar file` [src/paperless_tesseract/parsers.py:L107]) — observed in Q2a. On a **mixed / partially-text multi-page PDF** — the realistic case behind the user's report — the sidecar would instead contain the `[OCR skipped on page` marker for the skipped pages, so paperless discards it (`Incomplete sidecar file: discarding.` [src/paperless_tesseract/parsers.py:L110]) and falls back to extracting text from the archive PDF via `pdfminer` [src/paperless_tesseract/parsers.py:L117]. Either way, the pages that already carry a (possibly poor or partial) text layer are **not** re-OCR'd while text-free pages are, so the freshly-recognized OCR layer is applied **unevenly across pages of the same document** — which reads to a user as "inconsistent OCR results." This is a property of the default `skip` mode, not of randomness in the engine.

This is an **inferred** mechanism (the mixed-PDF case was not the fixture used here, precisely because the rules require reproducing the *reported* condition with a faithful input rather than constructing a variant); it is grounded in the **observed** `skip_text=True` parameter and the code's own sidecar semantics. The alternative modes that would change this are non-default and out of scope to enable: `redo` (`redo_ocr=True`, strips & rewrites the text layer) and `force` (`force_ocr=True`, rasterizes & OCRs every page) [src/paperless_tesseract/parsers.py:L155-L160]; the latter is exactly what the safe **fallback** uses [src/paperless_tesseract/parsers.py:L155-L156,L297].

---

## Coverage — edge cases

Beyond the happy path, the canonical endpoint and pipeline were exercised against negative/error inputs and a missing-dependency condition — all through the real interface, no bypass. Every result below is captured output.

**Negative HTTP cases at `POST /api/documents/post_document/`** (driven by `$INV/work/edges.py`, removed at cleanup, authenticated with the temporary generated-secret superuser where applicable):

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 $INV/work/edges.py
=== EDGE 1: unauthenticated POST to canonical endpoint (no credentials) ===
HTTP=401
body='{"detail":"Authentication credentials were not provided."}'
WWW-Authenticate='Basic realm="api"'

=== EDGE 2: authenticated POST but WRONG field name (no 'document' part) ===
HTTP=400
body='{"document":["No file was submitted."]}'

=== EDGE 3: authenticated GET on the POST-only endpoint ===
HTTP=405
body='{"detail":"Method \\"GET\\" not allowed."}'

=== EDGE 4: authenticated POST with empty multipart (no files at all) ===
HTTP=400
body='{"document":["No file was submitted."]}'
```

- **Unauthenticated → `401`** (a *run-first correction* of the natural `403` assumption): `permission_classes = (IsAuthenticated,)` [src/documents/views.py:L493] combined with the default `BasicAuthentication` [src/paperless/settings.py:L118] makes DRF return `401` with a `WWW-Authenticate: Basic realm="api"` challenge rather than `403`.
- **Renamed / missing `document` field → `400`**: `document` is a required `FileField` on `PostDocumentSerializer` [src/documents/serialisers.py:L415-L418], so its absence fails validation with `{"document":["No file was submitted."]}`.
- **`GET` on the POST-only view → `405`**: `PostDocumentView` implements only `post()` [src/documents/views.py:L497], so any other method yields `405 Method Not Allowed`.

**WebSocket upgrade is unavailable under `runserver` (WSGI).** This is exactly why Q1 captured the progress milestones at the Channels layer rather than through a browser socket — a plain HTTP `GET` to the `ws/status/` route [src/paperless/urls.py:L137] does not negotiate a `101 Switching Protocols`:

```console
$ source /tmp/.inv_env && cd /app/src && PYTHONPATH=/app/src python3 -c 'import os,requests; r=requests.get("http://localhost:8000/ws/status/"); print("ws/status/ over HTTP GET -> HTTP", r.status_code, "(dev runserver serves WSGI; no 101 Upgrade)")'
ws/status/ over HTTP GET -> HTTP 200 (dev runserver serves WSGI; no 101 Upgrade)
```

**A missing native binary (`jbig2enc`) does not break the pipeline.** The running image lacks `jbig2`/`jbig2enc` (Container section), yet OCRmyPDF completes and produces both the archive and the thumbnail. `jbig2enc` is an *optional* output optimisation (JBIG2 image compression); when absent, OCRmyPDF silently omits it:

```console
# (a) both jbig2 names absent on PATH
$ command -v jbig2 jbig2enc ; echo exit=$?
exit=1

# (b) any jbig2 / optimisation / 'not installed' / fallback messages in the isolated run log?
$ grep -inE "jbig2|not installed|could not|falling back|optimiz" "$INV/data/log/paperless.log" ; echo matches_exit=$?
matches_exit=1

# (c) the ONLY [ERROR] lines in the isolated log are the same-input duplicate rejections (Q2d) — no OCR/jbig2 error
$ grep -nE "\[ERROR\]" "$INV/data/log/paperless.log"
17:[2026-07-13 18:21:02,066] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
18:[2026-07-13 18:21:02,200] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
19:[2026-07-13 18:23:04,870] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.
20:[2026-07-13 18:23:04,944] [ERROR] [paperless.consumer] Not consuming ocr_test.pdf: It is a duplicate.

# (d) primary OCRmyPDF invocations recorded (initial run + 3 Q2d repeats = 4), all succeeded
$ grep -c "Calling OCRmyPDF with args" "$INV/data/log/paperless.log"
4

# (e) archive + thumbnail produced despite the missing binary (last surviving pk=4 from the Q2d repeats)
$ ls -l "$INV/media/documents/archive/"
total 116
-rw-r--r-- 1 root root 116354 Jul 13 18:25 0000004.pdf
$ ls -l "$INV/media/documents/thumbnails/"
total 40
-rw-r--r-- 1 root root 37493 Jul 13 18:25 0000004.png
```

`grep` exits non-zero (`matches_exit=1`) because there are **no** jbig2/optimisation/fallback messages at all; the only `[ERROR]` lines are the expected duplicate rejections from the Q2d same-input experiment. The missing binary is therefore a real, exercised edge condition the pipeline tolerates — it does **not** contribute to the reported "inconsistent OCR". (The archive here is pk=4's `116354`-byte file, differing from pk=1's `116353` — consistent with the per-run archive non-reproducibility from Q2d.)

---

## Coverage-pass checklist

| Item | Answered | Evidence |
|------|----------|----------|
| **Q1** — immediate HTTP **status** | ✅ `200` | `capture_progress.py` → `HTTP_STATUS 200` |
| **Q1** — immediate HTTP **body** | ✅ `"OK"` | `HTTP_BODY '"OK"'` |
| **Q1** — it is the **async ack** (does not wait for processing) | ✅ | log-line delta `0` at response time (`TIME_TOTAL 0.106s`); ordering not code-guaranteed, but the response is definitively the enqueue ack |
| **Q1** — fixture identity fixed by hash | ✅ | `FIXTURE_SHA256 d3ec86b9ac73…` printed before upload |
| **Q1/Q2** — progress milestones (6 events) + `task_id` correlation | ✅ | 6 `PROGRESS_EVENT`s over the Channels layer, all `task_id 6c8c075e-…`; terminal `SUCCESS` carries `document_id: 1` |
| **Q2** — ordered stage log lines | ✅ | full run-scoped `cat "$INV/data/log/paperless.log"` + per-line citation table |
| **Q2** — `Calling OCRmyPDF with args: {...}` captured verbatim | ✅ | quoted in full from the run-scoped log (Q2a) |
| **Q2** — all **13** primary params enumerated | ✅ | `input_file, output_file, use_threads, jobs=11, language, output_type, progress_bar, skip_text, clean, deskew, rotate_pages, rotate_pages_threshold, sidecar` |
| **Q2** — `image_dpi` omitted for PDF (noted) | ✅ | absent from dict; [src/paperless_tesseract/parsers.py:L64-L70]; [src/paperless_tesseract/parsers.py:L187-L215] |
| **Q2** — `OCR_USER_ARGS={}` adds nothing (noted) | ✅ | [src/paperless/settings.py:L541]; [src/paperless_tesseract/parsers.py:L217-L220] |
| **Q2** — fallback (`force_ocr`) captured or inferred | ✅ inferred | isolated-log `grep` → `Fallback`=`0`, primary=`1`; labelled inferred with [src/paperless_tesseract/parsers.py:L155-L156]; [src/paperless_tesseract/parsers.py:L297-L298] |
| **Q2** — worker return string + Task identity | ✅ | `Success. New document id 1 created`; django-q `Task.id f696…`; two distinct correlation ids explained |
| **Q2d** — same-input repeats (reproduce inconsistency faithfully) | ✅ | Part A dedup (2 rejections, doc set unchanged) + Part B 3 identical-byte OCR runs: **OCR text deterministic**, **archive_checksum non-reproducible** |
| **Q3** — archive filename | ✅ `archive/0000001.pdf` | isolated `ls -l` (pk from ORM) + `generate_filename` [src/documents/file_handling.py:L193] |
| **Q3** — thumbnail filename | ✅ `thumbnails/0000001.png` | isolated `ls -l` + `thumbnail_path` [src/documents/models.py:L274]; [src/documents/models.py:L278] |
| **Q4** — every stored DB column + values | ✅ | ORM dump + `PRAGMA table_info` → **15 columns** |
| **Q4** — `tags` is a through-table, not a column | ✅ | `documents_document_tags` `CREATE TABLE` + `PRAGMA` + `0` rows; `'tags' in cols → False` |
| **Q4** — every computed `@property` listed | ✅ | property table [src/documents/models.py:L222-L282]; `*_file` openers observed returning `BufferedReader` |
| **Q4** — relative-stored vs absolute-computed distinction | ✅ | side-by-side `filename` vs `source_path` for the resolved pk |
| **Synthesis** — `skip_text` → uneven coverage | ✅ | observed `skip_text=True` + sidecar comment [src/paperless_tesseract/parsers.py:L104-L106] |
| **Edge cases** — 401 / 400 / 405, WSGI-no-upgrade, missing `jbig2` tolerated | ✅ | see [Coverage — edge cases](#coverage--edge-cases): unauth→`401`, bad field→`400`, GET→`405`, `ws/status/`→`200` (no `101`), archive+thumbnail produced without `jbig2enc` |
| **Environment setup** — exact build/run/invocation commands | ✅ | container/versions/redis/migrate/temp-superuser/runserver/qcluster/settings/fixture, all with output |
| **Cleanup** — no residue; source unchanged | ✅ | single `rm -rf "$INV"`; shared `/app` byte-identical pre/post; temp superuser absent from shared DB |

**Labelling discipline:** every factual claim about the system carries a `[path:Lx]`/`[path:Lx-Ly]` locator (the source files match HEAD `542221a38`); every observed value is shown as command + complete unedited output. The only **inferred (code-read)** items are (a) the `force_ocr` fallback path (not triggered by the image-only fixture), (b) the run-to-run stability of the OCRmyPDF *parameters* (config-derived; the single dict + one invocation per run were observed), and (c) the mixed-PDF "inconsistency" mechanism (grounded in the observed `skip_text=True` and the code's sidecar semantics). No value here was obtained from a bypassing interface, so nothing is labelled non-canonical.


---

## Cleanup confirmation

Per the read-only + cleanup mandate, every runtime artifact created during this investigation was removed and the shared environment was verified unchanged. Because the whole investigation ran inside a **single disposable, mode-`0700` namespace** — `$INV=/tmp/inv_LVzCw9Qz`, with `PAPERLESS_DATA_DIR` [src/paperless/settings.py:L66], `MEDIA_ROOT` [src/paperless/settings.py:L61] and `CONSUMPTION_DIR` [src/paperless/settings.py:L78] relocated there — teardown is a **single recursive removal of that one directory**. There is no per-record `Document.objects.all().delete()` and no wildcard glob against shared paths; the database, media, and log for this run all live under `$INV` and vanish together. The exact commands and their complete output follow.

**1. Pre-teardown fingerprint.** Capture the owned process ids and the shared `/app` runtime-state baseline so the post-teardown state can be compared byte-for-byte. (`ps`/`pgrep`/`pkill` are **not** in this image, so owned PIDs are verified via `/proc/<pid>/cmdline`.)

```
########## PRE-TEARDOWN STATE FINGERPRINT ##########
# owned masters (verified via /proc): runserver=24375, qcluster=24376
$ ps -o pid,cmd -p 24375,24376
bash: line 1: ps: command not found

# SHARED /app state that must be unchanged by the whole investigation:
$ git -C /app rev-parse HEAD
4a296ad1d77d188516c612e3e83e7fdc5b417172
shared_documents= 0
shared_qtasks= 17
shared_log_lines=2699
shared_media_files=1

# isolated namespace currently EXISTS (its own db/log/media) and default scratch has my 4 uploads + 2 PRE-EXISTING dirs
$ find "$INV" -maxdepth 3 -type f | wc -l
isolated_file_count=24
$ ls -la /tmp/paperless
drwx------ 2 root root   4096 Jul 13 16:42 paperless-d4f0db24    # pre-existing (NOT mine)
drwx------ 2 root root   4096 Jul 13 16:42 paperless-dyspt444    # pre-existing (NOT mine)
-rw------- 1 root root 213603 Jul 13 18:21 paperless-upload-2ury4j6o
-rw------- 1 root root 213603 Jul 13 18:21 paperless-upload-gjbxwt_4
-rw------- 1 root root 213603 Jul 13 18:23 paperless-upload-_keeb7sp
-rw------- 1 root root 213603 Jul 13 18:23 paperless-upload-asugjwod
```

**2. Terminate ONLY the two owned processes**, by exact PID (never a broad `pkill`). Redis was already running from environment setup and is **left as found** (this investigation did not start it).

```
# (A) SIGTERM the two owned masters ONLY (cmdlines verified via /proc beforehand)
$ kill -TERM 24375 24376
term_sent_exit=0

# (B) confirm they are gone — the container's PID 1 is not a reaping init, so terminated
#     children remain as harmless zombies (State: Z) that hold no sockets/memory
$ grep State /proc/24375/status /proc/24376/status
PID 24375: State:	Z (zombie) cmdline=[]
PID 24376: State:	Z (zombie) cmdline=[]

# (C) nothing listens on :8000, and an authenticated request now fails to connect
$ (check /proc/net/tcp for 0x1F40)
port 8000: NOT listening (no 0x1F40 in /proc/net/tcp*)
$ curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://localhost:8000/api/ ; echo " exit=$?"
000 exit=7          # connection refused
```

**3. Remove my scratch upload temps (by exact name, not a glob of the shared scratch dir), then the whole namespace + env file.** Only the four `paperless-upload-…` files I created are removed — each listed explicitly below; the two pre-existing `16:42` scratch directories are left untouched.

```
# (D) remove ONLY my 4 scratch upload temps; leave the pre-existing 16:42 dirs
$ rm -f /tmp/paperless/paperless-upload-2ury4j6o /tmp/paperless/paperless-upload-gjbxwt_4 \
        /tmp/paperless/paperless-upload-_keeb7sp /tmp/paperless/paperless-upload-asugjwod
$ ls -la /tmp/paperless
drwx------ 2 root root 4096 Jul 13 16:42 paperless-d4f0db24    # pre-existing, still present
drwx------ 2 root root 4096 Jul 13 16:42 paperless-dyspt444    # pre-existing, still present

# (E) remove the single isolated namespace (db+media+log together) and the env file
$ rm -rf /tmp/inv_LVzCw9Qz ; rm -f /tmp/.inv_env
rm_exit=0
```

**4. Post-teardown fingerprint — compare to step 1.** The isolated namespace and env file are gone; shared Redis still answers; and the shared `/app` runtime state is **identical** to the pre-teardown baseline (same HEAD, same document/task/log/media counts).

```
########## POST-TEARDOWN FINGERPRINT ##########
$ ls -d /tmp/inv_LVzCw9Qz 2>/dev/null || echo "isolated_namespace_removed=yes"
isolated_namespace_removed=yes
$ ls /tmp/.inv_env 2>/dev/null || echo "env_file_removed=yes"
env_file_removed=yes
$ redis-cli ping
PONG                              # redis left as found

$ git -C /app rev-parse HEAD
4a296ad1d77d188516c612e3e83e7fdc5b417172   # unchanged
shared_documents= 0               # unchanged
shared_qtasks= 17                 # unchanged
shared_log_lines=2699             # unchanged
shared_media_files=1              # unchanged
```

**5. No leftover credentials (temporary superuser removed with the namespace).** The only superuser this investigation created (`inv_df856218`, generated secret) lived **only** in the isolated DB, which is now deleted. The shared `/app` database still contains just its pre-existing accounts — the temp superuser is absent, and no reusable credential was ever written into this document:

```
$ docker exec paperless-setup bash -lc 'cd /app/src && python3 -c "
import os, django; os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); django.setup()
from django.contrib.auth.models import User
print(\"shared_users =\", sorted((u.username, u.is_superuser) for u in User.objects.all()))
print(\"inv_df856218 present in shared DB?\", User.objects.filter(username=\"inv_df856218\").exists())"'
shared_users = [('admin', True), ('consumer', False)]
inv_df856218 present in shared DB? False
```

(`admin` is the environment-setup account created before this investigation; `consumer` is created by the data migration `User.objects.create(username="consumer")` [src/documents/migrations/0019_add_consumer_user.py:L10]. Neither was created by this investigation.)

**6. Source tree unchanged — the only change is this deliverable.** No file under `src/` (nor any other reference file) was modified; the sole change in the working tree is the new documentation file.

```
$ git rev-parse --abbrev-ref HEAD
blitzy-87b095e2-5299-40e2-89c5-c0b45c0c4edc
$ git diff --stat -- src         # zero reference/source files modified
                                 # (empty output)
$ git status --porcelain -- src  # nothing untracked or modified under src/
                                 # (empty output)
```

No existing source file was modified, and the only artifact that remains is `blitzy/documentation/paperless-ngx_542221a38dff.md`. The investigation was conducted entirely through the canonical upload API in the default configuration, inside a disposable namespace that has been fully removed, and the shared environment has been restored to — in fact never diverged from — its pre-test state. ✅
