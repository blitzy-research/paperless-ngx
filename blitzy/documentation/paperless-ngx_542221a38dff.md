# How OCR Behaves at Runtime in paperless-ngx — A Runtime-Grounded Investigation

**Branch:** `paperless-ngx_542221a38dff` · **HEAD:** `542221a38` · **Method:** every claim below was produced by *building, running, and observing* paperless-ngx in its default configuration and capturing the real output. Each behavioral claim is tagged **[observed-at-runtime]** or **[inferred-from-reading]**, shows the exact command and its complete unedited output, and cites the implementing `file:line`.

---

## TL;DR (the direct plain-reading answers)

- **Q1 — How can I see that OCR started, and what does the processing state look like?**
  There is **no per-document "processing state" field** in paperless-ngx. `class Document` has no status/state column (`src/documents/models.py:88`), and there is no `PaperlessTask` model at this commit. While OCR runs, the live state exists in exactly three places: the **WebSocket stream** at `ws/status/`, the **django-q task record**, and the **`paperless.consumer` log**. You first see OCR "start" as the `STARTING`→`WORKING/parsing_document` WebSocket payloads plus the debug log line `Calling OCRmyPDF with args: {…}` (`src/paperless_tesseract/parsers.py:260`). The signal that *truly* indicates active OCR work is the window between the `WORKING 20% parsing_document` and `WORKING 70% generating_thumbnail` payloads, during which a real `ocrmypdf`/`tesseract`/`unpaper`/`gs` subprocess tree is running. **[observed-at-runtime]**

- **Q2 — Does an image with text skip OCR, or still touch the pipeline?**
  In the default `skip` mode, **the OCR pipeline is *touched* for every document** — image *and* text-layer PDF — because paperless always invokes `ocrmypdf.ocr(**args)` with `skip_text=True` (`src/paperless_tesseract/parsers.py:158,261`). A raster **image never has an embedded text layer**, so OCR is *always* performed on images. The **only** case that skips OCRmyPDF *entirely* is the non-default `skip_noarchive` mode applied to a PDF whose embedded text exceeds 50 characters (`src/paperless_tesseract/parsers.py:241`). **[observed-at-runtime]**

- **Q3 — Which API fields show OCR-generated vs pre-existing text?**
  **Neither — the REST response alone cannot cleanly distinguish them in the default mode.** Both provenances land in the *same* `content` field, and the serializer exposes no provenance field (`src/documents/serialisers.py:222-235`). In default `skip`, both an image and a text-layer PDF produce an archive, so `archived_file_name` is populated for both. Byte-level provenance (the `checksum` / `archive_checksum` columns) is **DB-only** and not exposed by REST. **[observed-at-runtime]**

- **Q4 — What happens on weak/incomplete OCR?**
  The document is **still fully processed**. "Fully processed" is binary here: it means *a `Document` row exists*. Even when OCR finds nothing, the WebSocket stream ends in `SUCCESS/finished`, a `Document` is created with `content = ""` (`src/paperless_tesseract/parsers.py:327` → `src/documents/consumer.py:398`), the `checksum` is still set, and an archive file is still written. There is no partial-success state. **[observed-at-runtime]**

The remainder of this document proves each of these with captured evidence.

> **⚠️ One correction to a common assumption, surfaced up front (see Q1 §"The true active-OCR signal"):** the interpolated `WORKING 20→70` progress payloads that one might expect from the `progress_callback` are **never emitted for OCR documents**. `RasterisedDocumentParser.parse()` never calls the callback, so the observed stream jumps directly from `WORKING 20 parsing_document` to `WORKING 70 generating_thumbnail`. This is an observed fact, documented below with evidence.

---

## Environment & how to reproduce

### Versions (canonical container)

The investigation ran inside the user-supplied canonical Docker image (Python 3.9 backend + full OCR toolchain), which matches the pins in `requirements.txt` at this commit.

**Command:**
```bash
python3 --version
ocrmypdf --version
tesseract --version 2>&1 | head -1
gs --version
unpaper --version
redis-server --version
```

**Complete output [observed-at-runtime]:**
```
Python 3.9.23
ocrmypdf 13.4.3
tesseract 4.1.1
ghostscript 9.53.3
unpaper 6.1
Redis server v=6.0.16
```

Python package pins consumed (from `requirements.txt`, confirmed installed): `ocrmypdf==13.4.3`, `Django==4.0.4`, `djangorestframework==3.13.1`, `django-q==1.3.9` (**this is the background worker — not Celery**), `channels==3.0.4`, `channels-redis==3.4.0`, `redis==3.5.3`, `pikepdf==5.1.1`, `img2pdf==0.4.4`, `pdf2image==1.16.0`, `python-magic==0.4.25`, `scikit-learn==1.0.2`, `daphne==3.0.2`.

Notes on toolchain provenance:
- `qpdf` **10.1.0** is the version actually on `PATH` in the running container (the `.build-config.json` reference file names `10.6.3`; I report the *observed* running version). **[observed-at-runtime]**
- `jbig2` is **not invocable as a CLI** on `PATH` in this container (jbig2enc is an optional monochrome-compression helper; its absence does not affect any Q1–Q4 path). **[observed-at-runtime]**

### Default configuration (what a normal user gets)

**Command** (read the settings the running process actually sees, with no `PAPERLESS_OCR_MODE` set):
```bash
cd /app/src && python manage.py shell -c "
from django.conf import settings
print('OCR_MODE        =', repr(settings.OCR_MODE))
print('OCR_OUTPUT_TYPE =', repr(settings.OCR_OUTPUT_TYPE))
print('OCR_LANGUAGE    =', repr(settings.OCR_LANGUAGE))
print('ASGI_APPLICATION=', repr(settings.ASGI_APPLICATION))
"
```

**Complete output [observed-at-runtime]:**
```
OCR_MODE        = 'skip'
OCR_OUTPUT_TYPE = 'pdfa'
OCR_LANGUAGE    = 'eng'
ASGI_APPLICATION= 'paperless.asgi.application'
```

This matches the defaults defined at `src/paperless/settings.py:522` (`OCR_MODE` → `"skip"`), `:518` (`OCR_OUTPUT_TYPE` → `"pdfa"`), `:514` (`OCR_LANGUAGE` → `"eng"`), and `:156` (`ASGI_APPLICATION`). It is also the documented default in `paperless.conf.example:43` (`#PAPERLESS_OCR_MODE=skip`). Unless a scenario is **explicitly labeled** as a non-default mode run, everything below uses this default (`OCR_MODE="skip"`). **[inferred-from-reading + observed-at-runtime]**

### Standing up the async stack

Because "state while running" lives only in live surfaces, the investigation runs **redis + django-q qcluster + an ASGI server** concurrently. Runtime directories are pointed at `/tmp` so that no runtime artifact touches the repository.

```bash
# Runtime dirs + broker outside the repo tree; OCR_MODE intentionally UNSET => default "skip"
export PAPERLESS_REDIS="redis://localhost:6379"
export PAPERLESS_DATA_DIR=/tmp/pl/data
export PAPERLESS_MEDIA_ROOT=/tmp/pl/media
export PAPERLESS_CONSUMPTION_DIR=/tmp/pl/consume
export PAPERLESS_STATICDIR=/tmp/pl/static
export PAPERLESS_SECRET_KEY=insecure-observation-key
export PAPERLESS_TIME_ZONE=UTC
mkdir -p /tmp/pl/data /tmp/pl/media /tmp/pl/consume /tmp/pl/static

redis-server --daemonize yes                    # broker + channel layer ; redis-cli ping => PONG
cd /app/src && python manage.py migrate          # sqlite at $PAPERLESS_DATA_DIR/db.sqlite3

DJANGO_SUPERUSER_PASSWORD=admin \
  python manage.py createsuperuser --noinput --username admin --email admin@example.com
python manage.py drf_create_token admin          # prints a REST token (redacted here as <TOKEN>)

python manage.py qcluster > /tmp/pl/qcluster.log 2>&1 &     # django-q worker cluster
daphne -b 127.0.0.1 -p 8000 paperless.asgi:application \
  > /tmp/pl/asgi.log 2>&1 &                                  # ASGI/WebSocket server
```

The `qcluster` worker cluster reports (this is django-q, not Celery):

**Command:** `head -17 /tmp/pl/qcluster.log`

**Complete output [observed-at-runtime]:**
```
15:48:37 [Q] INFO Q Cluster failed-king-music-summer starting.
15:48:37 [Q] INFO Process-1:1 ready for work at 1804
15:48:37 [Q] INFO Process-1:2 ready for work at 1805
15:48:37 [Q] INFO Process-1:3 ready for work at 1806
15:48:37 [Q] INFO Process-1:4 ready for work at 1807
15:48:37 [Q] INFO Process-1:5 ready for work at 1808
15:48:37 [Q] INFO Process-1:6 ready for work at 1809
15:48:37 [Q] INFO Process-1:7 ready for work at 1810
15:48:37 [Q] INFO Process-1:8 ready for work at 1811
15:48:37 [Q] INFO Process-1:9 ready for work at 1812
15:48:37 [Q] INFO Process-1:10 ready for work at 1813
15:48:37 [Q] INFO Process-1:11 ready for work at 1814
15:48:37 [Q] INFO Process-1:12 monitoring at 1815
15:48:37 [Q] INFO Process-1 guarding cluster failed-king-music-summer
15:48:37 [Q] INFO Process-1:13 pushing tasks at 1816
15:48:37 [Q] INFO Q Cluster failed-king-music-summer running.
```

The "before" state, captured after the stack was up and before any upload:

**Command:** `curl -s -H "Authorization: Token <TOKEN>" http://127.0.0.1:8000/api/documents/`

**Complete output [observed-at-runtime]:**
```
{"count":0,"next":null,"previous":null,"results":[]}
```

### Authentication used for observation

- **REST** uses DRF token auth: `Authorization: Token <TOKEN>` (minted with `manage.py drf_create_token`). The real upload endpoint is `POST /api/documents/post_document/` (`src/documents/views.py:491`).
- **WebSocket** `ws/status/` requires an authenticated session, because `StatusConsumer.connect()` raises `DenyConnection()` unless `self.scope["user"].is_authenticated` (`src/paperless/consumers.py:11-14`). To subscribe a listener, a DB session was minted for the `admin` superuser and its `sessionid` cookie presented on the handshake. **This is equivalent to a normal browser login; the `StatusConsumer` auth check and channel-layer relay are the real code path being exercised.** Proof the auth is real: a valid `sessionid` connects and is accepted; a bogus `sessionid` is rejected at the handshake. **[observed-at-runtime]**

All observation helpers (WebSocket listener, session minter, ORM reader, upload driver) were written under `/tmp` and deleted afterward; the repository's only tracked change is this document.

---

## Q1 — Image with no embedded text: seeing OCR start, the live state, worker behavior, and the true active-OCR signal

**Input:** `no_text.png` — a raster PNG containing shapes but no rendered text (MIME `image/png`, detected by python-magic during consumption).

### Direct answer

You see OCR "start" as two nearly-simultaneous WebSocket payloads — `STARTING 0% new_file` then `WORKING 20% parsing_document` — immediately followed by the worker-log line `Calling OCRmyPDF with args: {…}`. The document's **processing state does not live in the `Document` row** (that row does not exist yet); it lives only in (a) the `ws/status/` stream, (b) the django-q task record, and (c) the `paperless.consumer` log. The worker is a **django-q `qcluster`** process that runs `consume_file` → `Consumer.try_consume_file()` → `RasterisedDocumentParser.parse()` → the `ocrmypdf` subprocess. The signal that *truly* indicates active OCR is the interval between the `parsing_document (20%)` and `generating_thumbnail (70%)` payloads, during which a live `tesseract`/`unpaper`/`ghostscript` subprocess tree exists. Everything at 70% and beyond (thumbnail, date parse, save) is **post-OCR** and is not OCR work. **[observed-at-runtime]**

### 1a. The ordered WebSocket sequence (before / during / after)

A listener was connected to `ws://127.0.0.1:8000/ws/status/` **before** the upload, then the file was posted to the real endpoint:

```bash
# listener already running in the background, writing timestamped payloads
curl -s -H "Authorization: Token <TOKEN>" \
     -F "document=@/tmp/pl/inputs/no_text.png" \
     http://127.0.0.1:8000/api/documents/post_document/
```

**Complete WebSocket capture [observed-at-runtime]:**
```
[connected 2026-07-07T15:38:31.495122] ws://127.0.0.1:8000/ws/status/
[2026-07-07T15:38:33.045511] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[2026-07-07T15:38:33.053289] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[2026-07-07T15:38:35.755701] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[2026-07-07T15:38:37.550009] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[2026-07-07T15:38:37.551871] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[2026-07-07T15:38:37.600458] {"filename": "no_text.png", "task_id": "3bd909af-bd6b-428d-8457-7ffcc4d6a34e", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
[closed 2026-07-07T15:38:56.571964]
```

**Interpretation and grounding.** The payload shape `{filename, task_id, current_progress, max_progress, status, message, document_id}` is built by `Consumer._send_progress()` (`src/documents/consumer.py:56`) and relayed to the browser by `StatusConsumer.status_update()` (`src/paperless/consumers.py:29-33`). The ordered emissions map exactly to source:

| Order | status | %/100 | message | Emitted at |
|-------|--------|-------|---------|-----------|
| 1 | `STARTING` | 0 | `new_file` | `src/documents/consumer.py:202` |
| 2 | `WORKING` | 20 | `parsing_document` | `src/documents/consumer.py:259` |
| 3 | `WORKING` | 70 | `generating_thumbnail` | `src/documents/consumer.py:264` |
| 4 | `WORKING` | 90 | `parse_date` (only if no date was found in metadata) | `src/documents/consumer.py:274` |
| 5 | `WORKING` | 95 | `save_document` | `src/documents/consumer.py:294` |
| 6 | `SUCCESS` | 100 | `finished` (+ `document_id`) | `src/documents/consumer.py:375` |

The message strings are the module-level constants `MESSAGE_NEW_FILE="new_file"` (`:43`), `MESSAGE_PARSING_DOCUMENT` (`:45`), `MESSAGE_GENERATING_THUMBNAIL` (`:46`), `MESSAGE_PARSE_DATE` (`:47`), `MESSAGE_SAVE_DOCUMENT` (`:48`), `MESSAGE_FINISHED` (`:49`). **[observed-at-runtime, grounded]**

Crucially, **`document_id` is `null` in every payload until the final `SUCCESS`**, where it becomes `1`. This is the cause→effect proof that there is no queryable document during processing: `document_id` is only known after `Document.objects.create()` runs inside `_store()` (`src/documents/consumer.py:398`), which is why the id first appears in the `finished` payload (`src/documents/consumer.py:375`). **[observed-at-runtime]**

### 1b. There is no per-document processing-state field

**Command** (grep the model's field block for any status-like column):
```bash
grep -nE "status|state|processing" src/documents/models.py | head
```
**Complete output [observed-at-runtime]:** *(no matching field in `class Document`; the term does not occur as a model field between the class at `:88` and the end of the field block)* — the command returns nothing for a status/state field.

`class Document` is defined at `src/documents/models.py:88`; its persisted columns include `content` (`:117`), `mime_type` (`:126`), `checksum` (`:135`), `archive_checksum` (`:143`), and `archive_filename` (`:186`), plus the `has_archive_version` property (`:238`) — but **no** status/state/processing field. There is also **no `PaperlessTask` model** anywhere in `src/` at this commit (a repository-wide grep returns nothing). **Therefore the "processing state" the question asks about does not exist as a database field; it exists only as the live signals shown here.** **[observed-at-runtime, grounded]**

The second live surface — the **django-q task record** — carries the finished/failed state that no `Document` field does:

**Command:**
```bash
cd /app/src && python manage.py shell -c "
from django_q.models import Task
qs=Task.objects.filter(func='documents.tasks.consume_file').order_by('started')
print('consume_file Task rows:', qs.count())
for t in qs[:4]:
    print('  name=',t.name,'| success=',t.success,'| started=',t.started.strftime('%H:%M:%S.%f')[:-3],'| stopped=',t.stopped.strftime('%H:%M:%S.%f')[:-3])
"
```
**Complete output [observed-at-runtime]:**
```
consume_file Task rows: 12
  name= no_text.png | success= True | started= 15:38:32.900 | stopped= 15:38:37.599
  name= no_text.png | success= False | started= 15:42:02.342 | stopped= 15:42:02.489
  name= no_text_1.png | success= True | started= 15:43:15.011 | stopped= 15:43:19.681
  name= text_image.png | success= True | started= 15:44:02.298 | stopped= 15:44:07.056
```

The first row is the `no_text.png` consume: `success=True`, elapsed ≈ **4.7 s** (15:38:32.900 → 15:38:37.599), which brackets the WebSocket sequence above. This `django_q.models.Task` table (redis-backed broker + result store) is the durable worker-state surface that stands in for the missing document status field. **[observed-at-runtime]**

### 1c. Worker behavior and the OCR invocation (the `paperless.consumer` log)

The definitive proof that OCR was invoked — and *which mode flag* was used — is the debug line emitted immediately before `ocrmypdf.ocr(**args)` (`src/paperless_tesseract/parsers.py:260-261`). The full consume trace for `no_text.png`:

**Command:** `sed -n '2,23p' /tmp/pl/data/log/paperless.log`

**Complete output [observed-at-runtime]:**
```
[2026-07-07 15:38:33,047] [INFO] [paperless.consumer] Consuming no_text.png
[2026-07-07 15:38:33,047] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-07 15:38:33,050] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-07 15:38:33,052] [DEBUG] [paperless.consumer] Parsing no_text.png...
[2026-07-07 15:38:33,143] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-5bgk4uap: 'dpi'
[2026-07-07 15:38:33,143] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-07 15:38:33,143] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-5bgk4uap', 'output_file': '/tmp/paperless/paperless-onor509t/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-onor509t/sidecar.txt', 'image_dpi': 120}
[2026-07-07 15:38:34,490] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-07 15:38:34,491] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-07 15:38:34,491] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-5bgk4uap: 'dpi'
[2026-07-07 15:38:34,491] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-07 15:38:34,491] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-5bgk4uap', 'output_file': '/tmp/paperless/paperless-onor509t/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-onor509t/sidecar-fallback.txt', 'image_dpi': 120}
[2026-07-07 15:38:35,750] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-07 15:38:35,751] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/paperless/paperless-upload-5bgk4uap, the content will be empty.
[2026-07-07 15:38:35,751] [DEBUG] [paperless.consumer] Generating thumbnail for no_text.png...
[2026-07-07 15:38:35,754] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-onor509t/archive.pdf[0] /tmp/paperless/paperless-onor509t/convert.png
[2026-07-07 15:38:36,184] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-onor509t/convert.png -out /tmp/paperless/paperless-onor509t/thumb_optipng.png
[2026-07-07 15:38:37,549] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-07 15:38:37,551] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-07 15:38:37,571] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-5bgk4uap
[2026-07-07 15:38:37,596] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-onor509t
[2026-07-07 15:38:37,597] [INFO] [paperless.consumer] Document 2026-07-07 no_text consumption finished
```

**Interpretation and grounding.**
- `Detected mime type: image/png` → `Parser: RasterisedDocumentParser` shows parser dispatch (`get_parser_class_for_mime_type` in `src/documents/parsers.py`).
- `Calling OCRmyPDF with args: {… 'skip_text': True …}` is emitted at `src/paperless_tesseract/parsers.py:260`, and `skip_text=True` is the flag chosen for the default `skip` mode by `construct_ocrmypdf_parameters()` (`:158`). This single line is the definitive "OCR is actively running" proof, and it names the mode.
- Because `no_text.png` is a blank image, OCR finds nothing → `NoTextFoundException` (`:267`) → the fallback re-invokes OCRmyPDF with `force_ocr=True` (`:297-298`, `construct_ocrmypdf_parameters(safe_fallback=True)` → `:156`) → still empty → `No text was found …, the content will be empty.` (`:322-327`). (This blank-input fallback chain is examined in detail under Q4.) **[observed-at-runtime, grounded]**

Note the log records `image_dpi: 120` and `jobs: 11` — real runtime-derived values, not defaults invented here.

### 1d. The true active-OCR signal: live subprocesses during the 20→70 window

To show what "active OCR" *is*, a `ps` sampler recorded the worker's child processes during the interval between the `parsing_document (20%)` and `generating_thumbnail (70%)` payloads (captured on the stability re-run `no_text_1.png`, whose sequence is identical):

**Command (sampler):** `while :; do ps -eo pid,rss,comm,args | grep -E 'tesseract|unpaper|ghostscript|gs |ocrmypdf' | grep -v grep; sleep 0.25; done`

**Complete output [observed-at-runtime]:**
```
[15:43:15.299]
   1101 24108 tesseract       tesseract --list-langs
[15:43:15.556]
   1117 29352 tesseract       tesseract -l osd --psm 0 /tmp/ocrmypdf.io._8rvm9vo/000001_rasterize_preview.jpg stdout
[15:43:15.813]
   1125 25764 tesseract       tesseract -l eng --psm 2 /tmp/ocrmypdf.io._8rvm9vo/000001_rasterize.png stdout
[15:43:16.071]
   1133  7600 unpaper         unpaper -v --dpi 120.0 --layout none --mask-scan-size 100 --no-border-align --no-mask-center --no-grayfilter --no-blackfilter --no-deskew /tmp/tmpaaj7sbot/input.pnm /tmp/tmpaaj7sbot/output.ppm
[15:43:16.328]
   1140 33440 tesseract       tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io._8rvm9vo/000001_ocr.png /tmp/ocrmypdf.io._8rvm9vo/000001_ocr_tess pdf txt
[15:43:16.842]
   1163 34592 tesseract       tesseract -l osd --psm 0 /tmp/ocrmypdf.io.y3dy_kog/000001_rasterize_preview.jpg stdout
[15:43:17.099]
   1171 29232 tesseract       tesseract -l eng --psm 2 /tmp/ocrmypdf.io.y3dy_kog/000001_rasterize.png stdout
[15:43:17.356]
   1179 11052 unpaper         unpaper -v --dpi 120.0 --layout none --mask-scan-size 100 --no-border-align --no-mask-center --no-grayfilter --no-blackfilter --no-deskew /tmp/tmpg6q611te/input.pnm /tmp/tmpg6q611te/output.ppm
[15:43:17.614]
   1186 33624 tesseract       tesseract -l eng -c textonly_pdf=1 /tmp/ocrmypdf.io.y3dy_kog/000001_ocr.png /tmp/ocrmypdf.io.y3dy_kog/000001_ocr_tess pdf txt
[15:43:18.128]
   1206 31588 gs              gs -sstdout=%stderr -dQUIET -dSAFER -dBATCH -dNOPAUSE -dNOPROMPT -dMaxBitmap=500000000 -dAlignToPixels=0 -dGridFitTT=2 -sDEVICE=pngalpha -dTextAlphaBits=4 -dGraphicsAlphaBits=4 -r300x300 -dFirstPage=1 -dLastPage=1 -sOutputFile=/tmp/magick-... -f/tmp/magick-... -f/tmp/magick-...
```

**Interpretation and grounding.** During the 20→70 window the worker spawns the real OCR toolchain: `tesseract --list-langs` (engine probe), `tesseract … --psm 0 …rasterize_preview` (orientation detection), `tesseract … --psm 2 …rasterize` (layout), `unpaper` (page cleaning, because `clean=True` in the args), and the actual OCR pass `tesseract -l eng -c textonly_pdf=1 …000001_ocr.png` — then `gs` (ghostscript) for the PDF/A archive. **Two distinct `ocrmypdf.io.*` temp roots** appear (`_8rvm9vo` and `y3dy_kog`), which are the two OCR passes seen in the log: the initial `skip_text` attempt and the `force_ocr` fallback. This subprocess tree — bounded by the `parsing_document` and `generating_thumbnail` payloads — **is** the active-OCR signal. **[observed-at-runtime, grounded]**

By contrast, the `generating_thumbnail (70%)` step runs `convert` + `optipng` (thumbnailing), `parse_date (90%)` parses a date from the filename/text, and `save_document (95%)` writes the DB row — **none of these are OCR**. So the honest answer to "which signal truly indicates active OCR work" is: *the `parsing_document → generating_thumbnail` interval plus the `Calling OCRmyPDF with args` log line plus the running tesseract/ocrmypdf subprocess* — not the later thumbnail/date/save steps.

### 1e. ⚠️ Correction: the interpolated 20→70 progress band is never emitted for OCR documents

One might expect a stream of intermediate `WORKING` payloads between 20% and 70%, produced by `progress_callback` (`src/documents/consumer.py:237-240`), which recalculates page progress via `p = int((current_progress / max_progress) * 50 + 20)`. **Observed reality: those intermediate payloads never appear for OCR documents** — every captured stream jumps straight from `WORKING 20 parsing_document` to `WORKING 70 generating_thumbnail` (see 1a, and the stability run below).

**Cause (grounded).** `RasterisedDocumentParser.parse()` **never calls** the progress callback. A grep of `src/paperless_tesseract/parsers.py` shows the only occurrence of "progress" is the OCRmyPDF argument `"progress_bar": False` (`:152`) — an argument value, *not* a call. The callback is defined (`consumer.py:237`), passed into the parser constructor (`consumer.py` parser instantiation), and stored, but the OCR parser body simply never invokes `self.progress(...)`. Hence no interpolated payloads are produced. **[observed-at-runtime + inferred-from-reading]** This is the one place where the pipeline's *emitted* behavior diverges from a plausible reading of the code, so it is called out explicitly.

### 1f. The FAILED path (edge case)

Re-uploading the *identical* `no_text.png` exercises the failure branch. The WebSocket stream:

**Complete output [observed-at-runtime]:**
```
[connected 2026-07-07T15:42:00.948051] ws://127.0.0.1:8000/ws/status/
[2026-07-07T15:42:02.484641] {"filename": "no_text.png", "task_id": "c3765424-ba8b-4c02-9c99-88244e05580e", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[2026-07-07T15:42:02.488707] {"filename": "no_text.png", "task_id": "c3765424-ba8b-4c02-9c99-88244e05580e", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
```
and the corresponding log line:
```
[2026-07-07 15:42:02,488] [ERROR] [paperless.consumer] Not consuming no_text.png: It is a duplicate.
```

**Interpretation and grounding.** Deduplication happens by checksum *before* parsing, so the stream goes `STARTING` → `FAILED` with message `document_already_exists`, emitted by `Consumer._fail()` (`src/documents/consumer.py:78`). The matching django-q Task row is the `success=False` entry at 15:42:02 seen in 1b. So the `FAILED` status is the fourth possible terminal state alongside `STARTING`/`WORKING`/`SUCCESS`. **[observed-at-runtime, grounded]**

### 1g. Stability (run-to-run)

The Q1 scenario was run at least twice. The stability re-run used `no_text_1.png`:

**Complete output [observed-at-runtime]:**
```
[connected 2026-07-07T15:43:13.612293] ws://127.0.0.1:8000/ws/status/
[2026-07-07T15:43:15.156609] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[2026-07-07T15:43:15.164906] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[2026-07-07T15:43:17.870854] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[2026-07-07T15:43:19.629101] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[2026-07-07T15:43:19.631132] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[2026-07-07T15:43:19.682341] {"filename": "no_text_1.png", "task_id": "3d6f35fe-8938-49f8-b8ed-08a4d6ba1852", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 2}
```
The payload sequence (statuses, percentages, messages, and the `document_id=null`-until-`SUCCESS` behavior) is **identical** to the first run; only the timestamps, `task_id`, and final `document_id` differ. The full pipeline call chain is `documents.tasks.consume_file` (`src/documents/tasks.py:184`) → `Consumer().try_consume_file(...)` (`src/documents/tasks.py:236`) → `RasterisedDocumentParser.parse()`. **[observed-at-runtime]**

---

## Q2 — A similar image that already contains text: skip, or still touch the pipeline?

### Direct answer (and an important fidelity reconciliation)

**In the default `skip` mode, the OCR pipeline is *touched* for every document — the `ocrmypdf.ocr()` subprocess runs even when the input already has text.** It never skips OCRmyPDF *entirely* in the default mode.

The user's phrase "an image that already contains text" needs one honest correction to be answerable: **a raster image (PNG/JPG) has no embedded text layer at all.** In `RasterisedDocumentParser.parse()`, the image branch hard-codes `original_has_text = False` (`src/paperless_tesseract/parsers.py:239`), so OCR is *always* performed on images regardless of what they visually depict. The place where "already has text → skip" can actually manifest is a **PDF that carries an embedded text layer**. This section therefore exercises **both**: (2a) an image that visibly contains words, and (2b/2c) a text-layer PDF. **[observed-at-runtime, grounded]**

The mode→flag decision is made by `construct_ocrmypdf_parameters()` (`src/paperless_tesseract/parsers.py:155-160`):
```python
if settings.OCR_MODE == "force" or safe_fallback:
    ocrmypdf_args["force_ocr"] = True
elif settings.OCR_MODE in ["skip", "skip_noarchive"]:
    ocrmypdf_args["skip_text"] = True
elif settings.OCR_MODE == "redo":
    ocrmypdf_args["redo_ocr"] = True
```

### 2a. An image that visibly contains text (`text_image.png`, default `skip`)

`text_image.png` renders legible words ("INVOICE 2024-0042", "Acme Corporation Limited", "Total amount due: 1234.56 USD", "Payment terms net thirty days"). Consume trace:

**Command:** `sed -n '/Consuming text_image.png$/,/text_image consumption finished/p' /tmp/pl/data/log/paperless.log`

**Complete output [observed-at-runtime]:**
```
[2026-07-07 15:44:02,444] [INFO] [paperless.consumer] Consuming text_image.png
[2026-07-07 15:44:02,444] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-07 15:44:02,447] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-07 15:44:02,449] [DEBUG] [paperless.consumer] Parsing text_image.png...
[2026-07-07 15:44:02,541] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-ovesht9p: 'dpi'
[2026-07-07 15:44:02,541] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 169 based on image width 1400
[2026-07-07 15:44:02,541] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-ovesht9p', 'output_file': '/tmp/paperless/paperless-mqujgftn/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-mqujgftn/sidecar.txt', 'image_dpi': 169}
[2026-07-07 15:44:04,222] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-07 15:44:04,223] [DEBUG] [paperless.consumer] Generating thumbnail for text_image.png...
[2026-07-07 15:44:04,227] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-mqujgftn/archive.pdf[0] /tmp/paperless/paperless-mqujgftn/convert.png
[2026-07-07 15:44:04,562] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-mqujgftn/convert.png -out /tmp/paperless/paperless-mqujgftn/thumb_optipng.png
[2026-07-07 15:44:07,006] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-07 15:44:07,009] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-07 15:44:07,028] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-ovesht9p
[2026-07-07 15:44:07,053] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-mqujgftn
[2026-07-07 15:44:07,054] [INFO] [paperless.consumer] Document 2026-07-07 text_image consumption finished
```

**Interpretation and grounding.** OCRmyPDF is invoked with `skip_text: True` — **the pipeline is touched, and OCR actually runs** (`src/paperless_tesseract/parsers.py:261`). Unlike the blank image, there is *no* `NoTextFoundException` and *no* force fallback, because tesseract found the rendered words. `Using text from sidecar file` (`:107`) is logged — meaning the text came from the OCRmyPDF sidecar, i.e. it is **OCR-generated**. The resulting content (confirmed via ORM in Q3) is `INVOICE 2024-0042\n\nAcme Corporation Limited\nTotal amount due: 1234.56 USD\n\nPayment terms net thirty days` (104 chars). This proves the claim: **images are always OCR'd, even when they "already contain text" visually**, because the image branch sets `original_has_text=False` (`:239`). **[observed-at-runtime, grounded]**

### 2b. A text-layer PDF (`text_layer.pdf`, default `skip`) — pipeline touched, OCR skipped per-page

`text_layer.pdf` is a born-digital PDF with a real embedded text layer of 236 characters (measured with the exact code path `original_has_text` uses — `post_process_text(pdfminer.extract_text(...))` — giving `len=236 > 50 → True`). Consume trace:

**Complete output [observed-at-runtime]:**
```
[2026-07-07 15:44:40,196] [INFO] [paperless.consumer] Consuming text_layer.pdf
[2026-07-07 15:44:40,197] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-07 15:44:40,200] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-07 15:44:40,202] [DEBUG] [paperless.consumer] Parsing text_layer.pdf...
[2026-07-07 15:44:40,228] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-r24m_wkv
[2026-07-07 15:44:40,297] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-r24m_wkv', 'output_file': '/tmp/paperless/paperless-7848swc2/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-7848swc2/sidecar.txt'}
[2026-07-07 15:44:40,558] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-07 15:44:40,564] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-7848swc2/archive.pdf
[2026-07-07 15:44:40,564] [DEBUG] [paperless.consumer] Generating thumbnail for text_layer.pdf...
[2026-07-07 15:44:40,567] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-7848swc2/archive.pdf[0] /tmp/paperless/paperless-7848swc2/convert.png
[2026-07-07 15:44:41,224] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-7848swc2/convert.png -out /tmp/paperless/paperless-7848swc2/thumb_optipng.png
[2026-07-07 15:44:44,227] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-07 15:44:44,230] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-07 15:44:44,250] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-r24m_wkv
[2026-07-07 15:44:44,275] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-7848swc2
[2026-07-07 15:44:44,275] [INFO] [paperless.consumer] Document 2026-07-07 text_layer consumption finished
```

**Interpretation and grounding — this trace is the heart of Q2/Q3.** Four events, in order:
1. `Extracted text from PDF file …upload…` — before OCR, `parse()` runs pdfminer to compute `text_original` and `original_has_text = len > 50` (`src/paperless_tesseract/parsers.py:234-236`).
2. `Calling OCRmyPDF with args: {… 'skip_text': True …}` — **OCRmyPDF is still invoked (pipeline touched)** (`:261`). Note there is no `image_dpi` key here because the input is a PDF, not an image.
3. `Incomplete sidecar file: discarding.` — the sidecar OCRmyPDF wrote contained the marker `[OCR skipped on page …]`, so `extract_text()` discards it (`:104`, `:110`).
4. `Extracted text from PDF file …archive.pdf` — with the sidecar discarded, paperless re-extracts the embedded text from the produced archive with pdfminer (`:119`).

So for a text-layer PDF, OCRmyPDF runs but **performs no OCR on the already-text page** (it copies the page through), the sidecar comes back empty-for-that-page, and paperless falls back to the PDF's own embedded text. The stored `content` is therefore the **pre-existing** text (236 chars), not OCR output. This is the mechanism that later makes Q3's provenance question answerable only via logs/DB. **[observed-at-runtime, grounded]**

### 2c. The ONLY true skip: `skip_noarchive` + embedded text > 50 chars (NON-DEFAULT — labeled)

> **Non-default run.** The following was executed with `PAPERLESS_OCR_MODE=skip_noarchive` (the qcluster worker was restarted with that environment variable). Every other setting is default. This is the *only* configuration in which OCRmyPDF is not invoked at all.

Consuming `text_layer_1.pdf` (embedded text 241 chars > 50) under `skip_noarchive`:

**Complete output [observed-at-runtime]:**
```
[2026-07-07 15:46:34,152] [INFO] [paperless.consumer] Consuming text_layer_1.pdf
[2026-07-07 15:46:34,152] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-07 15:46:34,155] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-07 15:46:34,157] [DEBUG] [paperless.consumer] Parsing text_layer_1.pdf...
[2026-07-07 15:46:34,182] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-l3truo0c
[2026-07-07 15:46:34,182] [DEBUG] [paperless.parsing.tesseract] Document has text, skipping OCRmyPDF entirely.
[2026-07-07 15:46:34,182] [DEBUG] [paperless.consumer] Generating thumbnail for text_layer_1.pdf...
[2026-07-07 15:46:34,185] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-upload-l3truo0c[0] /tmp/paperless/paperless-rfggkxdl/convert.png
[2026-07-07 15:46:34,843] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-rfggkxdl/convert.png -out /tmp/paperless/paperless-rfggkxdl/thumb_optipng.png
[2026-07-07 15:46:37,812] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-07 15:46:37,815] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-07 15:46:37,834] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-l3truo0c
[2026-07-07 15:46:37,859] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-rfggkxdl
[2026-07-07 15:46:37,860] [INFO] [paperless.consumer] Document 2026-07-07 text_layer_1 consumption finished
```

**Interpretation and grounding.** After extracting the embedded text, `parse()` hits the early return `if settings.OCR_MODE == "skip_noarchive" and original_has_text:` (`src/paperless_tesseract/parsers.py:241`), logs `Document has text, skipping OCRmyPDF entirely.` (`:242`), sets `self.text = text_original`, and returns. **There is NO `Calling OCRmyPDF with args` line — OCRmyPDF is never invoked, and no archive is produced.** Notice the thumbnail here is generated from the *original upload* (`…paperless-upload-l3truo0c[0]`), not from an `archive.pdf`, confirming no archive was written. This is the single case that "skips OCR entirely." **[observed-at-runtime, grounded]**

### 2d. The 50-character boundary (demonstrated at the boundary, under `skip_noarchive`)

The gate for the early return is `original_has_text = text_original and len(text_original) > 50` (`src/paperless_tesseract/parsers.py:236`). Two PDFs were crafted to straddle it: `boundary_le50.pdf` (29 embedded chars, ≤ 50) and `boundary_gt50.pdf` (74 embedded chars, > 50), both consumed under `skip_noarchive`.

**`boundary_le50.pdf` (29 chars → `original_has_text=False` → OCR runs) [observed-at-runtime]:**
```
[2026-07-07 15:47:19,314] [INFO] [paperless.consumer] Consuming boundary_le50.pdf
...
[2026-07-07 15:47:19,341] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-ys8lkaic
[2026-07-07 15:47:19,409] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-ys8lkaic', 'output_file': '/tmp/paperless/paperless-g3rvz51_/archive.pdf', ... 'skip_text': True, ... 'sidecar': '/tmp/paperless/paperless-g3rvz51_/sidecar.txt'}
[2026-07-07 15:47:19,676] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-07 15:47:19,680] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-g3rvz51_/archive.pdf
...
[2026-07-07 15:47:21,348] [INFO] [paperless.consumer] Document 2026-07-07 boundary_le50 consumption finished
```

**`boundary_gt50.pdf` (74 chars → `original_has_text=True` → OCRmyPDF skipped entirely) [observed-at-runtime]:**
```
[2026-07-07 15:47:41,153] [INFO] [paperless.consumer] Consuming boundary_gt50.pdf
...
[2026-07-07 15:47:41,181] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-mtu_an2e
[2026-07-07 15:47:41,181] [DEBUG] [paperless.parsing.tesseract] Document has text, skipping OCRmyPDF entirely.
[2026-07-07 15:47:41,183] [DEBUG] [paperless.parsing] Execute: convert ... /tmp/paperless/paperless-upload-mtu_an2e[0] ...
...
[2026-07-07 15:47:42,869] [INFO] [paperless.consumer] Document 2026-07-07 boundary_gt50 consumption finished
```

**Interpretation.** The only difference between the two inputs is 29 vs 74 characters of embedded text, and that difference flips the behavior exactly at the `> 50` gate (`:236`): the 29-char PDF still invokes OCRmyPDF (`skip_text=True`) and gets an archive, while the 74-char PDF takes the `skipping OCRmyPDF entirely` early return (`:242`) and gets no archive. This is the boundary demonstrated at the boundary. **[observed-at-runtime, grounded]**

### 2e. How to tell the difference after processing finishes

- **Under default `skip`:** both an image and a text-layer PDF produce an archive, so `archived_file_name` is populated for both (see Q3). The clean way to tell provenance apart *after the fact* is the **worker log**: `Using text from sidecar file` (`:107`) ⇒ the content is **OCR-generated** (2a); `Incomplete sidecar file: discarding.` (`:110`) followed by `Extracted text from PDF file …archive.pdf` (`:119`) ⇒ the content is **pre-existing** embedded text (2b). Alternatively, the DB `checksum` vs `archive_checksum` columns give byte-level provenance (Q3).
- **Under `skip_noarchive`:** the after-the-fact signal is cleanly REST-visible — `archived_file_name == null` (no archive was produced), because OCRmyPDF was skipped entirely (2c).

**[observed-at-runtime, grounded]**

---

## Q3 — Comparing the final API responses: which fields show OCR-generated vs pre-existing text?

### Direct answer (lead with the honest, potentially counter-intuitive result)

**In the default `skip` mode, the REST response alone does NOT cleanly distinguish OCR-generated text from pre-existing text.** Both provenances are written to the **same `content` field**, and `DocumentSerializer` exposes **no provenance field**. In default mode both an image and a text-layer PDF produce an archive, so `archived_file_name` is populated for *both*. The byte-level provenance signals — the `checksum` and `archive_checksum` columns — are **not serialized** and are visible only via the DB/ORM. The only weak REST-visible hint is `original_file_name`'s extension (`.png` vs `.pdf`). **[observed-at-runtime, grounded]**

### 3a. The two full REST responses side by side

**Command (image case, doc 3):** `curl -s -H "Authorization: Token <TOKEN>" http://127.0.0.1:8000/api/documents/3/ | python -m json.tool`

**Complete output [observed-at-runtime]:**
```json
{
    "id": 3,
    "correspondent": null,
    "document_type": null,
    "title": "text_image",
    "content": "INVOICE 2024-0042\n\nAcme Corporation Limited\nTotal amount due: 1234.56 USD\n\nPayment terms net thirty days",
    "tags": [],
    "created": "2026-07-07T15:44:02Z",
    "modified": "2026-07-07T15:44:07.027807Z",
    "added": "2026-07-07T15:44:07.010625Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-07 text_image.png",
    "archived_file_name": "2026-07-07 text_image.pdf"
}
```

**Command (text-layer PDF case, doc 4):** `curl -s -H "Authorization: Token <TOKEN>" http://127.0.0.1:8000/api/documents/4/ | python -m json.tool`

**Complete output [observed-at-runtime]:**
```json
{
    "id": 4,
    "correspondent": null,
    "document_type": null,
    "title": "text_layer",
    "content": "This is a born-digital PDF with a real embedded text layer.\n\nIt was produced by reportlab, not by scanning or OCR.\n\nThe paperless OCR pipeline should detect this pre-existing text.\n\nInvoice number 2024-0042 for Acme Corporation Limited.",
    "tags": [],
    "created": "2026-07-07T15:44:40.048710Z",
    "modified": "2026-07-07T15:44:44.249913Z",
    "added": "2026-07-07T15:44:44.231405Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-07 text_layer.pdf",
    "archived_file_name": "2026-07-07 text_layer.pdf"
}
```

### 3b. Field-by-field comparison

| Field | Image (doc 3, OCR-generated) | Text-layer PDF (doc 4, pre-existing) | Distinguishes provenance? |
|-------|------------------------------|--------------------------------------|---------------------------|
| `id` | `3` | `4` | no |
| `content` | 104-char OCR output | 236-char embedded text | **no — same field for both** |
| `original_file_name` | `2026-07-07 text_image.png` | `2026-07-07 text_layer.pdf` | weak hint only (extension) |
| `archived_file_name` | `2026-07-07 text_image.pdf` | `2026-07-07 text_layer.pdf` | **no — populated for both** |
| `correspondent`/`document_type`/`tags`/`archive_serial_number` | null/empty | null/empty | no |
| `created`/`modified`/`added` | timestamps | timestamps | no |

Both responses carry the **identical serializer field set**, which is exactly `DocumentSerializer.Meta.fields` (`src/documents/serialisers.py:222-235`):
```python
fields = (
    "id", "correspondent", "document_type", "title", "content", "tags",
    "created", "modified", "added", "archive_serial_number",
    "original_file_name", "archived_file_name",
)
```
The text — whether produced by OCR (doc 3) or lifted from the PDF's embedded layer (doc 4) — lands in the single `content` field. This is because `_store()` writes `Document.objects.create(content=text, …)` (`src/documents/consumer.py:398`) regardless of how `parse()` obtained `text`. There is no field recording *how* the text was obtained. **[observed-at-runtime, grounded]**

`archived_file_name` is a `SerializerMethodField` (`src/documents/serialisers.py:208`) whose getter returns the archive filename when `obj.has_archive_version` is true, else `None` (`:213-217`). In default `skip`, **both** documents have an archive, so both return a filename — hence this field cannot separate the two provenances in the default mode. **[observed-at-runtime, grounded]**

### 3c. Byte-level provenance is DB-only (not in REST)

The columns that *would* distinguish provenance byte-for-byte — `checksum` (of the original bytes) and `archive_checksum` (of the OCR/normalized archive) — are **absent from the serializer**. They can only be read from the DB:

**Command:**
```bash
cd /app/src && python manage.py shell -c "
from documents.models import Document
for i in [3,4]:
    d=Document.objects.get(pk=i)
    print('doc',d.pk, d.title,'| checksum=',d.checksum,'| archive_checksum=',d.archive_checksum,'| archive_filename=',d.archive_filename,'| has_archive_version=',d.has_archive_version)
"
```
**Complete output [observed-at-runtime]:**
```
doc 3 text_image | checksum= 39243b9c86ef6c4adb027a5ba7380d41 | archive_checksum= a7685c047521f84d40f68997d3cd6aae | archive_filename= 0000003.pdf | has_archive_version= True
doc 4 text_layer | checksum= 258ae5cbc096627a4a5c02d1c8f0b503 | archive_checksum= de3dc988c324939a97991e7c52ac875a | archive_filename= 0000004.pdf | has_archive_version= True
```

`checksum` is defined at `src/documents/models.py:135`, `archive_checksum` at `:143`, `archive_filename` at `:186` — none of which appear in `DocumentSerializer.Meta.fields`. So even though both documents *do* have distinct checksums, **you cannot see them through the REST API**; you must query the ORM/admin. **[observed-at-runtime, grounded]**

> **Byte-sensitivity note.** The checksums above are MD5 hashes computed by the running system over the exact stored bytes (`hashlib.md5(...)` in `_store()`, `src/documents/consumer.py`). They are reported verbatim as emitted. Comparing them against externally-recomputed hashes was out of scope for a provenance *field* question; treated here strictly as observed values that differ per document.

### 3d. The clean REST-visible distinction exists only under `skip_noarchive`

The one configuration in which the REST response *does* cleanly signal "no OCR archive" is the non-default `skip_noarchive` mode. Doc 5 (the `skip_noarchive` text-layer PDF from Q2c) returns `archived_file_name: null`:

**Command:** `curl -s -H "Authorization: Token <TOKEN>" http://127.0.0.1:8000/api/documents/5/ | python -m json.tool`

**Complete output [observed-at-runtime]:**
```json
{
    "id": 5,
    "correspondent": null,
    "document_type": null,
    "title": "text_layer_1",
    "content": "This is a born-digital PDF with a real embedded text layer.\n\nIt was produced by reportlab, not by scanning or OCR.\n\nThe paperless OCR pipeline should detect this pre-existing text.\n\nInvoice number 2024-0042 for Acme Corporation Limited (v1).",
    "tags": [],
    "created": "2026-07-07T15:46:34.002938Z",
    "modified": "2026-07-07T15:46:37.834056Z",
    "added": "2026-07-07T15:46:37.816397Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-07 text_layer_1.pdf",
    "archived_file_name": null
}
```

Here `archived_file_name` is `null` because `has_archive_version` is false (no archive was produced when OCRmyPDF was skipped entirely). This is the only clean, REST-only "this text is pre-existing, not OCR-generated" signal — and it requires the non-default `skip_noarchive` mode. **[observed-at-runtime, grounded]**

---

## Q4 — Weak or incomplete OCR: what happens to the final state?

### Direct answer

**The document is still fully processed.** When OCR produces weak or empty results, the WebSocket stream still ends in `SUCCESS/finished`, a `Document` row is still created, and processing is considered complete. "Fully processed" is **binary**: it means *a `Document` row exists*. There is no partial-success or degraded state. The weak result is reflected in the saved metadata as an **empty `content` string** (`content = ""`); the `checksum` is still set, and — for an image input — an archive file is still written. **[observed-at-runtime, grounded]**

### 4a. The final state is `SUCCESS` (a document is created)

**Input:** `blank.png` — a near-blank solid image that yields no OCR text. WebSocket stream:

**Complete output [observed-at-runtime]:**
```
[connected 2026-07-07T15:48:44.655725] ws://127.0.0.1:8000/ws/status/
[2026-07-07T15:48:46.194810] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[2026-07-07T15:48:46.202421] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[2026-07-07T15:48:48.499140] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[2026-07-07T15:48:48.966959] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[2026-07-07T15:48:48.970115] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[2026-07-07T15:48:49.022642] {"filename": "blank.png", "task_id": "4daef87d-c6e8-40ba-8f25-47f49ea00611", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 8}
```

Despite finding no text, the stream terminates in `SUCCESS` with `document_id: 8` — identical in shape to the healthy Q1 sequence. The empty-OCR outcome does **not** produce `FAILED`. **[observed-at-runtime]**

### 4b. The fallback chain that leads to empty content (cause → effect)

**Command:** `sed -n '/Consuming blank.png$/,/Document 2026-07-07 blank consumption finished/p' /tmp/pl/data/log/paperless.log`

**Complete output [observed-at-runtime]:**
```
[2026-07-07 15:48:46,196] [INFO] [paperless.consumer] Consuming blank.png
[2026-07-07 15:48:46,197] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-07 15:48:46,199] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-07 15:48:46,202] [DEBUG] [paperless.consumer] Parsing blank.png...
[2026-07-07 15:48:46,292] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-fkbwzmly: 'dpi'
[2026-07-07 15:48:46,292] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-07 15:48:46,292] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-fkbwzmly', 'output_file': '/tmp/paperless/paperless-ufwb1von/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-ufwb1von/sidecar.txt', 'image_dpi': 120}
[2026-07-07 15:48:47,419] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-07 15:48:47,419] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-07 15:48:47,420] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-fkbwzmly: 'dpi'
[2026-07-07 15:48:47,420] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-07 15:48:47,420] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-fkbwzmly', 'output_file': '/tmp/paperless/paperless-ufwb1von/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-ufwb1von/sidecar-fallback.txt', 'image_dpi': 120}
[2026-07-07 15:48:48,494] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-07 15:48:48,494] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/paperless/paperless-upload-fkbwzmly, the content will be empty.
[2026-07-07 15:48:48,494] [DEBUG] [paperless.consumer] Generating thumbnail for blank.png...
[2026-07-07 15:48:48,498] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-ufwb1von/archive.pdf[0] /tmp/paperless/paperless-ufwb1von/convert.png
[2026-07-07 15:48:48,897] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-ufwb1von/convert.png -out /tmp/paperless/paperless-ufwb1von/thumb_optipng.png
[2026-07-07 15:48:48,966] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-07 15:48:48,969] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-07 15:48:48,992] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-fkbwzmly
[2026-07-07 15:48:49,018] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-ufwb1von
[2026-07-07 15:48:49,018] [INFO] [paperless.consumer] Document 2026-07-07 blank consumption finished
```

**Cause → effect, grounded in source.** The content is empty *because*:
1. First OCR pass with `skip_text=True` finds nothing → `self.text` is empty → `raise NoTextFoundException("No text was found in the original document")` (`src/paperless_tesseract/parsers.py:267`).
2. That exception is caught and paperless retries with `safe_fallback=True`, which maps to `force_ocr=True` (`:156`), logged as `Fallback: Calling OCRmyPDF with args: {… 'force_ocr': True …}` (`:297-298`). This corresponds to OCRmyPDF's `--force-ocr` (rasterize every page and re-OCR).
3. The forced pass *also* finds nothing → the last-resort branch runs: since there is no `original_has_text` for an image (`:319`), it logs `No text was found in …, the content will be empty.` (`:322-325`) and sets `self.text = ""` (`:327`).
4. Consumption then continues normally to thumbnail → save. **The empty text is a valid result, not an error.**

**[observed-at-runtime, grounded]**

### 4c. The saved metadata reflects the weak result

**REST response [observed-at-runtime]:** `curl -s -H "Authorization: Token <TOKEN>" http://127.0.0.1:8000/api/documents/8/ | python -m json.tool`
```json
{
    "id": 8,
    "correspondent": null,
    "document_type": null,
    "title": "blank",
    "content": "",
    "tags": [],
    "created": "2026-07-07T15:48:46.055357Z",
    "modified": "2026-07-07T15:48:48.991739Z",
    "added": "2026-07-07T15:48:48.970811Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-07 blank.png",
    "archived_file_name": "2026-07-07 blank.pdf"
}
```

**DB/ORM row [observed-at-runtime]:**
```
doc 8 | blank
  mime_type       : image/png
  content(repr)   : '' len= 0
  checksum        : ad930d863dafac88369d8f2edfec63c5
  archive_checksum: ff261bbe71973d85dc3071f675ff744b
  archive_filename: 0000008.pdf
  has_archive_ver : True
```

**Interpretation and grounding.**
- `content` is the empty string `""` (REST) / `'' len=0` (ORM) — this is the saved reflection of the weak OCR result, exactly the value assigned at `src/paperless_tesseract/parsers.py:327`.
- The document is nonetheless a **complete, first-class record**: `checksum` is set (`ad930d86…`), and — because this is an *image* input — the first (non-forced) `archive.pdf` was produced, so `has_archive_version=True` and `archived_file_name` is populated (`2026-07-07 blank.pdf`). The `self.archive_path` was assigned from the first pass (`src/paperless_tesseract/parsers.py:263`) before the `NoTextFoundException`, and the fallback deliberately does *not* overwrite it, so the archive survives even though the text is empty. **[observed-at-runtime, grounded]**
- "Fully processed" therefore equals "a `Document` row exists": the `Document.objects.create()` at `src/documents/consumer.py:398` ran inside `transaction.atomic()` (`:298`), and the `SUCCESS/finished` payload (`:375`) confirms completion. There is no separate "processed / partially-processed / failed-OCR" status because — as established in Q1 — no such field exists on the model. **[observed-at-runtime, grounded]**

### 4d. Stability

The weak-OCR scenario was re-run with `blank_1.png`; the WebSocket sequence again ended in `SUCCESS/finished`, the same `NoTextFoundException → force fallback → "the content will be empty."` log markers appeared, and the resulting document again had `content == ""` with `checksum` set and an archive present. The behavior is stable run-to-run. **[observed-at-runtime]**

---

## Web corroboration — OCRmyPDF option semantics (official docs)

The observed behavior was cross-checked against the official OCRmyPDF documentation (`ocrmypdf.readthedocs.io`), reading the version family closest to the pinned `ocrmypdf==13.4.3`. Short attributed excerpts:

- **`--skip-text`** (paperless `skip`/`skip_noarchive` → `skip_text=True`): the docs state that with `--skip-text`, "no OCR will be performed on pages that already have text" and "The page will be copied to the output" (OCRmyPDF docs, *Advanced features*). This corroborates Q2b: a text-layer PDF still invokes OCRmyPDF (pipeline touched) but the already-text page is copied through — so the sidecar has no text for that page.
- **Sidecar behavior**: the docs state that "If the document contains pages that already have text, that text will not appear in the sidecar" (OCRmyPDF docs, *Cookbook*). This directly explains the mechanism in `extract_text()` — the `[OCR skipped on page` marker check (`src/paperless_tesseract/parsers.py:104`) and the pdfminer fallback (`:110`, `:119`).
- **`--force-ocr`** (paperless `force` and the safe-fallback retry → `force_ocr=True`): the docs state that with `--force-ocr`, "all pages will be rasterized to images, discarding any hidden OCR text" (OCRmyPDF docs, *Advanced features*). This corroborates the Q1/Q4 fallback pass (`:297-298`); paperless intentionally does not adopt the forced archive as the primary one.
- **`--redo-ocr`** (paperless `redo` → `redo_ocr=True`): the docs describe that invisible "text is categorized as either visible or invisible" and "Invisible text (OCR) is stripped out" then re-OCR'd (OCRmyPDF docs, *Advanced features*) — matching paperless's `redo` mapping (`:160`). (Not exercised at runtime here; listed for completeness of the mode map.)
- **Default (no mode flag)**: the docs note that normally "OCRmyPDF will exit with an error if asked to modify a file with OCR" (OCRmyPDF docs, *Cookbook*). This is *why* paperless always passes exactly one mode flag in `construct_ocrmypdf_parameters()` (`:155-160`).
- **`--output-type pdfa`**: produces PDF/A archival output, matching paperless's `OCR_OUTPUT_TYPE="pdfa"` (`src/paperless/settings.py:518`) and the observed `'output_type': 'pdfa'` in every `Calling OCRmyPDF with args` dict.

All observed runtime behavior is consistent with the official documentation.

---

## Coverage / verification appendix

Each named item from the four questions, confirmed with a concrete value, a `file:line`, a cause→effect reason, and captured runtime output:

- **[✓] Every start signal and its %/status.** `STARTING 0 new_file` (`consumer.py:202`), `WORKING 20 parsing_document` (`:259`), `WORKING 70 generating_thumbnail` (`:264`), `WORKING 90 parse_date` (`:274`), `WORKING 95 save_document` (`:294`), `SUCCESS 100 finished` (`:375`); plus the `FAILED 100 document_already_exists` path (`_fail`, `:78`). Evidence: Q1 §1a, §1f.
- **[✓] No per-document processing-state field; no `PaperlessTask`.** `class Document` at `models.py:88` has no status field; live state lives in the `ws/status/` stream, the `django_q.models.Task` record, and `paperless.consumer` logs. Evidence: Q1 §1b.
- **[✓] Worker/`qcluster` behavior and the call chain.** `consume_file` (`tasks.py:184`) → `try_consume_file` (`tasks.py:236`) → `RasterisedDocumentParser.parse()` → `ocrmypdf`. Evidence: Environment (cluster startup), Q1 §1b (Task row), §1d (subprocesses).
- **[✓] The signal that truly indicates active OCR.** The `parsing_document(20)→generating_thumbnail(70)` window + the `Calling OCRmyPDF with args` log (`parsers.py:260`) + the running tesseract/unpaper/gs subprocess tree; thumbnail/date/save are post-OCR. Evidence: Q1 §1c, §1d.
- **[✓] The 20→70 interpolation is never emitted for OCR docs.** `parse()` never calls `progress_callback`; only `"progress_bar": False` (`parsers.py:152`) appears. Evidence: Q1 §1e.
- **[✓] Image-vs-text-layer-PDF reconciliation.** Images set `original_has_text=False` (`parsers.py:239`) → always OCR'd; the skip behavior is a PDF phenomenon. Evidence: Q2 §2a, §2b.
- **[✓] Skip-vs-touch with the exact flag.** Default `skip` ⇒ `skip_text=True` (`parsers.py:158`), pipeline touched; the only entire skip is `skip_noarchive`+>50 chars ⇒ `Document has text, skipping OCRmyPDF entirely.` (`:241-242`). Evidence: Q2 §2b, §2c.
- **[✓] The 50-character boundary demonstrated at the boundary.** 29 chars → OCR runs; 74 chars → early return; gate at `parsers.py:236`. Evidence: Q2 §2d.
- **[✓] Field-by-field API comparison.** Identical serializer fields (`serialisers.py:222-235`); same `content` field; no provenance field; `archived_file_name` populated for both in default mode; checksums are DB-only. Evidence: Q3 §3a–§3c.
- **[✓] The sidecar/pdfminer provenance mechanism.** `[OCR skipped on page` marker check (`parsers.py:104`) → discard sidecar (`:110`) → pdfminer fallback (`:119`). Evidence: Q2 §2b; corroborated by OCRmyPDF docs.
- **[✓] Honest "no clean REST-only distinction in default mode."** Led with in the Q3 direct answer; clean signal (`archived_file_name==null`) only under `skip_noarchive`. Evidence: Q3 §3d.
- **[✓] Weak-OCR "still SUCCESS / empty content".** `NoTextFoundException` (`parsers.py:267`) → force retry (`:297`) → `self.text=""` (`:327`); `SUCCESS/finished` with `content=""`. Evidence: Q4 §4a–§4c.
- **[✓] "Fully processed = a `Document` row exists" (binary).** `Document.objects.create()` (`consumer.py:398`) inside `transaction.atomic()` (`:298`); no partial state. Evidence: Q4 §4c.
- **[✓] Saved-metadata reflection of weak OCR.** `content=""`, `checksum` set, archive present (`has_archive_version=True`). Evidence: Q4 §4c.
- **[✓] Web-corroborated OCRmyPDF semantics cited.** See the corroboration section.
- **[✓] Run-to-run stability.** Q1/Q2a/Q2b/Q4 each run ≥2×; sequences and final states identical (OCR text was byte-identical across the two `text_image` runs). Evidence: Q1 §1g, Q4 §4d.

### Terminal-state summary (all observed documents)

| doc | input | kind | mode | `content` len | archive? | terminal WS status |
|-----|-------|------|------|---------------|----------|--------------------|
| 1 | no_text.png | image | skip (default) | 0 | yes | SUCCESS |
| 2 | no_text_1.png | image | skip (default) | 0 | yes | SUCCESS |
| 3 | text_image.png | image | skip (default) | 104 (OCR) | yes | SUCCESS |
| 4 | text_layer.pdf | pdf | skip (default) | 236 (pre-existing) | yes | SUCCESS |
| 5 | text_layer_1.pdf | pdf | skip_noarchive (non-default) | 241 (pre-existing) | **no** | SUCCESS |
| 6 | boundary_le50.pdf | pdf | skip_noarchive (non-default) | 29 (OCR ran, ≤50) | yes | SUCCESS |
| 7 | boundary_gt50.pdf | pdf | skip_noarchive (non-default) | 74 (skipped, >50) | **no** | SUCCESS |
| 8 | blank.png | image | skip (default) | 0 (empty) | yes | SUCCESS |
| — | no_text.png (dup) | image | skip (default) | — | — | **FAILED** (duplicate) |

*(Documents 9–11 are the stability re-runs of docs 3, 4, and 8; each reproduced its counterpart's sequence and final state.)*

---

## Method notes and honesty labels

- **Real entry point.** Every consumption was driven through `POST /api/documents/post_document/` (`src/documents/views.py:491`, `async_task` enqueue at `:523`), which returns the literal body `"OK"` (`:535`) and does **not** return the document id — the id was correlated via the WebSocket `task_id → document_id` payload. No debug hooks or bypasses were used.
- **Session for the WebSocket** was minted directly for the `admin` superuser and presented as a `sessionid` cookie. This is **equivalent to a normal login**; the exercised code path (`StatusConsumer` authentication at `src/paperless/consumers.py:11-14` and the channel-layer relay at `:29-33`) is the real one. Labeled here so it is not mistaken for the primary behavior under test.
- **Labels.** Statements are marked **[observed-at-runtime]** where backed by captured output, and **[inferred-from-reading]** where derived from source without a direct runtime capture (notably the `progress_callback` non-invocation reasoning in Q1 §1e, which is additionally supported by the observed absence of intermediate payloads).
- **No value was rounded or adjusted.** All checksums, character counts, DPIs, timestamps, and `task_id`/`document_id` values are reproduced exactly as emitted.
- **`<TOKEN>`** in the commands above is a redacted placeholder for the local, throwaway DRF token minted for this investigation; it is intentionally not written into this document.
