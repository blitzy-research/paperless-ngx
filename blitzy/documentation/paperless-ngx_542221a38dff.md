# paperless-ngx — Runtime Behavior of Document Ingestion

**Branch:** `paperless-ngx_542221a38dff` · **Commit (HEAD):** `542221a38dff06361e07976452f9aea24d210542`

This document answers three questions about how **paperless-ngx** behaves at runtime while it ingests a document:

- **O1 — The ingestion pipeline.** When a test PDF is submitted, *which services participate* and *what ordered sequence of events appears in the logs* as the document moves through the system?
- **O2 — Classifier retraining.** Does the machine-learning classifier retrain automatically on **every** upload, or **only under certain conditions**? What log messages distinguish "training is happening" from "the classifier is idle"?
- **O3 — Storage & database.** After a document finishes processing, *where does it land on disk*, what is the *default directory structure and filename pattern*, and *which database tables receive new rows*?

> Every answer below was produced **run-first**: the full stack was built and run inside the mandated Docker image, real PDFs were pushed through it, and the **actual output was captured**. Quoted log lines, filenames, table names, HTTP responses, and measured values are reproduced **verbatim** from that run — every code fence attributed to observed output is an exact copy of the bytes the system emitted, and where a fence shows a bounded *window* of a log file the window is contiguous (nothing is elided **inside** it; the first/last lines are the true start/end of the slice). Each factual claim carries a `file:line` citation to this commit's source. Where a value could not be produced at runtime, it is called out explicitly. The exact command or script that produced each quoted value is given inline and, collectively, in the closing **"How this was observed"** section.

---

## Methodology (run-first, read-only)

- **Read-only:** No file in the source tree was modified. The only artifact added to the repository is this document. All source references below are **read-only citations**. Every temporary script and test PDF used for observation was created outside the tracked tree (inside the container's `/tmp`) and removed afterward.
- **Environment:** The host cannot run paperless-ngx (Python 3.13, no Redis/tesseract/scikit-learn). All observation was done inside a fresh container named `pngx-fix` (image `paperless-ngx-ready:latest`, derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...`), which ships **Python 3.9** ([`Dockerfile:L18`](../../Dockerfile) → `FROM python:3.9-slim-bullseye`) and the OCR binaries `tesseract-ocr` (+eng/deu/fra/ita/spa) ([`Dockerfile:L63-L68`](../../Dockerfile)), `ghostscript` ([`Dockerfile:L41`](../../Dockerfile)) and `imagemagick` ([`Dockerfile:L45`](../../Dockerfile)).
- **Default configuration:** `PAPERLESS_FILENAME_FORMAT` was left **unset** so O3 documents out-of-the-box behavior; its default is `None` ([`src/paperless/settings.py:L584`](../../src/paperless/settings.py)).
- **DEBUG visible:** `PAPERLESS_DEBUG=true` so `paperless.consumer`/`paperless.classifier` DEBUG lines surface. Mechanically, `DEBUG` defaults `False` ([`settings.py:L50`](../../src/paperless/settings.py)); the `paperless` logger is level `DEBUG` writing to a file handler ([`settings.py:L409`](../../src/paperless/settings.py)) → `data/log/paperless.log`, so DEBUG lines land there regardless, while the console (root) shows DEBUG only when `PAPERLESS_DEBUG=true` ([`settings.py:L388`](../../src/paperless/settings.py)).

Verification that we ran the intended commit, in the intended runtime:

```console
$ docker exec pngx-fix python3 --version
Python 3.9.23
$ docker exec pngx-fix git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ docker exec pngx-fix bash -lc 'echo "PAPERLESS_FILENAME_FORMAT=[${PAPERLESS_FILENAME_FORMAT}]"'
PAPERLESS_FILENAME_FORMAT=[]
```

## ⚠️ Version-skew guard (ground everything in THIS commit)

Current upstream paperless-ngx docs describe a **Celery** task processor, a `PAPERLESS_TRAIN_TASK_CRON` setting, a `storage_path` classifier, and HMAC-signed model files. **None of those apply at this commit.** This commit uses:

- **django-q** (not Celery) — `"django_q"` in `INSTALLED_APPS` ([`settings.py:L110`](../../src/paperless/settings.py)) and a `Q_CLUSTER` dict ([`settings.py:L449`](../../src/paperless/settings.py)). The worker log literally prints a django-q banner (`[Q]`, `Q Cluster ... running.`), quoted below.
- A **`Schedule.HOURLY`** training schedule registered by a migration ([`src/documents/migrations/1001_auto_20201109_1636.py:L10-L14`](../../src/documents/migrations/1001_auto_20201109_1636.py)).
- A **plain-pickle** classifier with `FORMAT_VERSION = 7` ([`src/documents/classifier.py:L63`](../../src/documents/classifier.py)) and **no HMAC** — verified below by reading the model file header.

---

## The running stack (services / processes)

The runtime is a monolithic Django app split into three supervisor-managed processes plus a Redis broker, defined in [`docker/supervisord.conf`](../../docker/supervisord.conf):

| Process | Command | `supervisord.conf` | Role |
|---|---|---|---|
| **gunicorn** | `gunicorn -c ...gunicorn.conf.py paperless.asgi:application` | `[program:gunicorn]` L10–L11 | ASGI web UI + REST API + WebSocket status feed |
| **document_consumer** | `python3 manage.py document_consumer` | `[program:consumer]` L19–L20 | Watches the consume directory (inotify) and enqueues `consume_file` |
| **scheduler / qcluster** | `python3 manage.py qcluster` | `[program:scheduler]` L28–L29 | django-q worker: runs queued `consume_file` **and** scheduled `train_classifier` |
| **Redis** | `redis-server` (6.0) | (broker) | django-q broker **and** Channels layer for WebSocket progress |

All four were confirmed live. Listing the four **master** processes (parent PID 1) is exact and complete — nothing is elided:

```console
$ docker exec pngx-fix bash -lc "ps -eo pid,ppid,cmd --sort=pid | grep -E 'redis-server|manage.py qcluster|manage.py document_consumer|paperless.asgi:application' | grep -v grep | awk '\$2==1'"
     56       1 redis-server *:6379
     92       1 python3 manage.py qcluster
     93       1 python3 manage.py document_consumer
     94       1 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

`qcluster` and `gunicorn` additionally fork worker subprocesses; counting **every** matching process (masters + forks) is likewise exact and complete:

```console
$ docker exec pngx-fix bash -lc "ps -eo cmd | grep -E 'redis-server|manage.py qcluster|manage.py document_consumer|paperless.asgi:application' | grep -v grep | sort | uniq -c"
      3 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
      1 python3 manage.py document_consumer
     15 python3 manage.py qcluster
      1 redis-server *:6379
```

(1 gunicorn master + 2 workers; 1 `document_consumer`; 1 `qcluster` master + its monitor/pusher/worker pool = 15; 1 `redis-server`.)

Startup "ready" lines (verbatim):

```text
# data/log/gunicorn.log
[2026-07-01 05:36:20 +0000] [94] [INFO] Starting gunicorn 20.1.0
[2026-07-01 05:36:20 +0000] [94] [INFO] Listening at: http://0.0.0.0:8000 (94)
[2026-07-01 05:36:20 +0000] [94] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 05:36:20 +0000] [94] [INFO] Server is ready. Spawning workers

# data/log/qcluster.log  (contiguous slice from "starting." to "running."; note the [Q] django-q banner — NOT Celery)
05:36:21 [Q] INFO Q Cluster potato-maryland-london-one starting.
05:36:21 [Q] INFO Process-1:1 ready for work at 123
05:36:21 [Q] INFO Process-1:2 ready for work at 124
05:36:21 [Q] INFO Process-1:3 ready for work at 125
05:36:21 [Q] INFO Process-1:4 ready for work at 126
05:36:21 [Q] INFO Process-1:5 ready for work at 127
05:36:21 [Q] INFO Process-1:6 ready for work at 128
05:36:21 [Q] INFO Process-1:7 ready for work at 129
05:36:21 [Q] INFO Process-1:8 ready for work at 130
05:36:21 [Q] INFO Process-1:9 ready for work at 131
05:36:21 [Q] INFO Process-1:10 ready for work at 132
05:36:21 [Q] INFO Process-1:11 ready for work at 133
05:36:21 [Q] INFO Process-1:12 monitoring at 134
05:36:21 [Q] INFO Process-1 guarding cluster potato-maryland-london-one
05:36:21 [Q] INFO Process-1:13 pushing tasks at 135
05:36:21 [Q] INFO Q Cluster potato-maryland-london-one running.

# data/log/consumer.log
[2026-07-01 05:36:21,560] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

The API answered on port 8000 (basic auth `admin:admin`). The response body is shown in full (no truncation):

```console
$ # GET /api/  (via python urllib; curl is not in the image)
HTTP_STATUS: 200
BODY: {"correspondents":"http://localhost:8000/api/correspondents/","document_types":"http://localhost:8000/api/document_types/","documents":"http://localhost:8000/api/documents/","logs":"http://localhost:8000/api/logs/","tags":"http://localhost:8000/api/tags/","saved_views":"http://localhost:8000/api/saved_views/"}
```

A Django user named **`consumer`** must exist, because the `set_log_entry` consumption handler calls `User.objects.get(username="consumer")` ([`src/documents/signals/handlers.py:L416`](../../src/documents/signals/handlers.py)); if absent, the consume transaction fails. Both users were present:

```console
$ python3 manage.py shell -c "from django.contrib.auth.models import User; print(sorted(User.objects.values_list('username', flat=True)))"
['admin', 'consumer']
```

The log format is the `verbose` formatter `[{asctime}] [{levelname}] [{name}] {message}` ([`settings.py:L378`](../../src/paperless/settings.py)), visible in every quoted line above.

---

# O1 — The ingestion pipeline: services involved & ordered log sequence

## O1(a) Which services participate

A submitted PDF flows through **all four** runtime pieces plus the parser subsystem:

1. **`document_consumer`** (watcher) — if the file arrives via the consume directory, an inotify watcher detects it and enqueues work. Import `from django_q.tasks import async_task` ([`src/documents/management/commands/document_consumer.py:L13`](../../src/documents/management/commands/document_consumer.py)); it logs `"Adding {filepath} to the task queue."` ([`document_consumer.py:L85`](../../src/documents/management/commands/document_consumer.py)) then `async_task("documents.tasks.consume_file", ...)` ([`document_consumer.py:L86`](../../src/documents/management/commands/document_consumer.py)).
2. **`gunicorn`** (web/API) — if the file arrives via HTTP, `PostDocumentView.post()` ([`src/documents/views.py:L491,L497`](../../src/documents/views.py)) writes the upload to `SCRATCH_DIR` and calls `async_task("documents.tasks.consume_file", ...)` ([`views.py:L523`](../../src/documents/views.py)).
3. **Redis** — the django-q broker that carries the `consume_file` task from either producer to the worker (`Q_CLUSTER` [`settings.py:L449`](../../src/paperless/settings.py); `PAPERLESS_REDIS` default `redis://localhost:6379` [`settings.py:L456`](../../src/paperless/settings.py)). Redis is also the Channels layer for WebSocket progress.
4. **`qcluster`** (django-q worker) — dequeues and runs `consume_file` ([`src/documents/tasks.py:L184`](../../src/documents/tasks.py)), which constructs `Consumer().try_consume_file(...)` and returns `"Success. New document id {} created"` ([`tasks.py:L247`](../../src/documents/tasks.py)).
5. **Parser subsystem** — for a PDF the parser is `RasterisedDocumentParser` (from `paperless_tesseract`), which shells out to **ocrmypdf/tesseract/ghostscript/imagemagick**.
6. **Sinks** — the SQLite database, the media filesystem, the Whoosh full-text index, and the WebSocket `status_updates` group.

> **This is django-q, not Celery.** The worker log prints `Q Cluster potato-maryland-london-one running.` and every task line is prefixed `[Q]` (quoted above and below).

Inside `Consumer.try_consume_file()`, after the DB transaction, the `document_consumption_finished` signal fans out to **six** handlers connected in [`src/documents/apps.py:L22-L27`](../../src/documents/apps.py):

```python
document_consumption_finished.connect(add_inbox_tags)
document_consumption_finished.connect(set_correspondent)
document_consumption_finished.connect(set_document_type)
document_consumption_finished.connect(set_tags)
document_consumption_finished.connect(set_log_entry)
document_consumption_finished.connect(add_to_index)
```

The three domain signals are defined in [`src/documents/signals/__init__.py:L3-L5`](../../src/documents/signals/__init__.py): `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration`.

## O1(b) The two ingestion entry points (both demonstrated)

**Entry point 1 — consume directory (drives the watcher).**

```console
$ docker exec pngx-fix bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test.pdf'
```

The watcher enqueued it (from `data/log/consumer.log`, logger `paperless.management.consumer`):

```text
[2026-07-01 05:38:13,283] [INFO] [paperless.management.consumer] Adding /app/src/../consume/o1_test.pdf to the task queue.
05:38:13 [Q] INFO Enqueued 1
```

**Entry point 2 — `POST /api/documents/post_document/` (drives web → queue).** A multipart upload (field name `document`) with basic auth `admin:admin`, produced by the `post_document.py` script (shown in full in *How this was observed*), returned **verbatim**:

```console
$ docker exec pngx-fix bash -lc 'cd /app/src && python3 /tmp/post_document.py /tmp/o1_post.pdf'
HTTP_STATUS: 200
RESPONSE_BODY: '"OK"'
```

That matches `return Response("OK")` ([`views.py:L535`](../../src/documents/views.py)). The worker then processed it. Here is the **complete, contiguous** `data/log/qcluster.log` window for the POST document (`o1_post.pdf`) — the `[Q]` worker lifecycle interleaved with the `paperless.consumer` sequence, from `processing [...]` through `recycled worker`, with nothing elided:

```text
05:39:19 [Q] INFO Process-1:6 processing [o1_post.pdf]
[2026-07-01 05:39:19,669] [INFO] [paperless.consumer] Consuming o1_post.pdf
[2026-07-01 05:39:19,670] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-01 05:39:19,672] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 05:39:19,675] [DEBUG] [paperless.consumer] Parsing o1_post.pdf...
[2026-07-01 05:39:19,696] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-wa794xa5
[2026-07-01 05:39:19,828] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-wa794xa5', 'output_file': '/tmp/paperless/paperless-nrf48dhi/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-nrf48dhi/sidecar.txt'}
[2026-07-01 05:39:20,096] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-01 05:39:20,101] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-nrf48dhi/archive.pdf
[2026-07-01 05:39:20,101] [DEBUG] [paperless.consumer] Generating thumbnail for o1_post.pdf...
[2026-07-01 05:39:20,104] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-nrf48dhi/archive.pdf[0] /tmp/paperless/paperless-nrf48dhi/convert.png
[2026-07-01 05:39:20,774] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-nrf48dhi/convert.png -out /tmp/paperless/paperless-nrf48dhi/thumb_optipng.png
[2026-07-01 05:39:23,729] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:39:23,732] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 05:39:23,752] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-wa794xa5
[2026-07-01 05:39:23,777] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-nrf48dhi
[2026-07-01 05:39:23,778] [INFO] [paperless.consumer] Document 2026-07-01 o1_post consumption finished
05:39:23 [Q] INFO Process-1:6 stopped doing work
05:39:23 [Q] INFO Processed [o1_post.pdf]
05:39:24 [Q] INFO recycled worker Process-1:6
```

Note the difference between the two entry points visible in the log: the consume-dir file is read straight from `/app/src/../consume/o1_test.pdf`, whereas the POST file is first written to a `SCRATCH_DIR` temp file (`/tmp/paperless/paperless-upload-wa794xa5`) by `PostDocumentView.post()` ([`views.py:L510-L521`](../../src/documents/views.py)) and deleted at the end.

Both tasks' return values (stored in the django-q task table) confirm success and the created document ids — matching `"Success. New document id {} created"` ([`tasks.py:L247`](../../src/documents/tasks.py)):

```console
$ python3 manage.py shell -c "from django_q.models import Task; [print(t.func,'| success:',t.success,'| result:',repr(t.result)) for t in Task.objects.filter(func='documents.tasks.consume_file').order_by('started')]"
documents.tasks.consume_file | success: True | result: 'Success. New document id 1 created'
documents.tasks.consume_file | success: True | result: 'Success. New document id 2 created'
```

## O1(c) The ordered log sequence — one file, three synchronized log windows (verbatim)

This is the ordered sequence for a **single** consume-directory ingestion, `o1_test.pdf` (which became document **pk 1** → `0000001.pdf`). Three log files are shown as **synchronized windows** for the same file and same timestamps; none contains an ellipsis.

**Window 1 — the watcher** (`data/log/consumer.log`, the `document_consumer` process's own stdout):

```text
[2026-07-01 05:38:13,283] [INFO] [paperless.management.consumer] Adding /app/src/../consume/o1_test.pdf to the task queue.
05:38:13 [Q] INFO Enqueued 1
```

**Window 2 — the qcluster worker** (`data/log/qcluster.log`). This is the **complete, contiguous** worker transcript: the `[Q]` markers `processing [o1_test.pdf]` → task execution → `Processed [o1_test.pdf]` → `recycled worker`, with the `paperless.consumer` sequence interleaved (see the note after the table for why the consumer lines appear here):

```text
05:38:13 [Q] INFO Process-1:5 processing [o1_test.pdf]
[2026-07-01 05:38:13,392] [INFO] [paperless.consumer] Consuming o1_test.pdf
[2026-07-01 05:38:13,393] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-01 05:38:13,396] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 05:38:13,398] [DEBUG] [paperless.consumer] Parsing o1_test.pdf...
[2026-07-01 05:38:13,418] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /app/src/../consume/o1_test.pdf
[2026-07-01 05:38:13,540] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/o1_test.pdf', 'output_file': '/tmp/paperless/paperless-mnnsogr_/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-mnnsogr_/sidecar.txt'}
[2026-07-01 05:38:13,812] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-01 05:38:13,817] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-mnnsogr_/archive.pdf
[2026-07-01 05:38:13,817] [DEBUG] [paperless.consumer] Generating thumbnail for o1_test.pdf...
[2026-07-01 05:38:13,820] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-mnnsogr_/archive.pdf[0] /tmp/paperless/paperless-mnnsogr_/convert.png
[2026-07-01 05:38:14,484] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-mnnsogr_/convert.png -out /tmp/paperless/paperless-mnnsogr_/thumb_optipng.png
[2026-07-01 05:38:17,401] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:38:17,404] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 05:38:17,423] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/o1_test.pdf
[2026-07-01 05:38:17,451] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-mnnsogr_
[2026-07-01 05:38:17,451] [INFO] [paperless.consumer] Document 2026-07-01 o1_test consumption finished
05:38:17 [Q] INFO Process-1:5 stopped doing work
05:38:17 [Q] INFO Processed [o1_test.pdf]
05:38:17 [Q] INFO recycled worker Process-1:5
05:38:17 [Q] INFO Process-1:18 ready for work at 395
```

**Window 3 — the dedicated file log** (`data/log/paperless.log`, the `file_paperless` handler). Same file, same timestamps; the `paperless.*` records only (no `[Q]` markers, since those are django-q's own stdout, not the `paperless` logger):

```text
[2026-07-01 05:38:13,283] [INFO] [paperless.management.consumer] Adding /app/src/../consume/o1_test.pdf to the task queue.
[2026-07-01 05:38:13,392] [INFO] [paperless.consumer] Consuming o1_test.pdf
[2026-07-01 05:38:13,393] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-01 05:38:13,396] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 05:38:13,398] [DEBUG] [paperless.consumer] Parsing o1_test.pdf...
[2026-07-01 05:38:13,418] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /app/src/../consume/o1_test.pdf
[2026-07-01 05:38:13,540] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/o1_test.pdf', 'output_file': '/tmp/paperless/paperless-mnnsogr_/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-mnnsogr_/sidecar.txt'}
[2026-07-01 05:38:13,812] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-01 05:38:13,817] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-mnnsogr_/archive.pdf
[2026-07-01 05:38:13,817] [DEBUG] [paperless.consumer] Generating thumbnail for o1_test.pdf...
[2026-07-01 05:38:13,820] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-mnnsogr_/archive.pdf[0] /tmp/paperless/paperless-mnnsogr_/convert.png
[2026-07-01 05:38:14,484] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-mnnsogr_/convert.png -out /tmp/paperless/paperless-mnnsogr_/thumb_optipng.png
[2026-07-01 05:38:17,401] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:38:17,404] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 05:38:17,423] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/o1_test.pdf
[2026-07-01 05:38:17,451] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-mnnsogr_
[2026-07-01 05:38:17,451] [INFO] [paperless.consumer] Document 2026-07-01 o1_test consumption finished
```

Mapping each observed line to its source (all within `Consumer.try_consume_file()`, defined at [`src/documents/consumer.py:L180`](../../src/documents/consumer.py) and spanning L180–L376, logger `paperless.consumer` unless noted):

| Observed line | Source `file:line` |
|---|---|
| `Adding ... to the task queue.` (logger `paperless.management.consumer`) | [`document_consumer.py:L85`](../../src/documents/management/commands/document_consumer.py) |
| `Consuming o1_test.pdf` | [`consumer.py:L215`](../../src/documents/consumer.py) |
| `Detected mime type: application/pdf` | [`consumer.py:L221`](../../src/documents/consumer.py) |
| `Parser: RasterisedDocumentParser` | [`consumer.py:L246`](../../src/documents/consumer.py) |
| `Parsing o1_test.pdf...` | [`consumer.py:L260`](../../src/documents/consumer.py) |
| `Extracted text from PDF file ...` / `Calling OCRmyPDF with args: ...` | [`src/paperless_tesseract/parsers.py`](../../src/paperless_tesseract/parsers.py) |
| `Generating thumbnail for o1_test.pdf...` | [`consumer.py:L263`](../../src/documents/consumer.py) |
| `Document classification model does not exist (yet), not performing automatic matching.` (logger `paperless.classifier`) | [`classifier.py:L33`](../../src/documents/classifier.py) via `load_classifier()` at [`consumer.py:L292`](../../src/documents/consumer.py) |
| `Saving record to database` | [`consumer.py:L387`](../../src/documents/consumer.py) (inside `_store()`) |
| `Deleting file /app/src/../consume/o1_test.pdf` | [`consumer.py:L349`](../../src/documents/consumer.py) |
| `Document 2026-07-01 o1_test consumption finished` | [`consumer.py:L373`](../../src/documents/consumer.py) |
| `processing [o1_test.pdf]` / `Processed [o1_test.pdf]` / `recycled worker` (logger `[Q]`) | django-q worker lifecycle (`Q_CLUSTER` [`settings.py:L449`](../../src/paperless/settings.py)) |

**Why the `paperless.consumer` lines appear in `qcluster.log`.** The `paperless` logger's records are handled by `file_paperless` (→ `paperless.log`) and, because Django logging propagation is on by default, also bubble up to the **root** logger whose handler is the `console` `StreamHandler` at level `DEBUG` when `PAPERLESS_DEBUG=true` ([`settings.py:L385-L409`](../../src/paperless/settings.py)). Since the consume task executes inside the `qcluster` worker process, that console stream is the worker's stdout — captured in `qcluster.log`. That is why Window 2 shows the `[Q]` markers and the `paperless.consumer` sequence together, tied to the same file and timestamps.

### Important: the numeric progress steps are WebSocket-only, not in the log

The pipeline also emits progress steps **`0/100 STARTING`**, `20/100`, `70/100`, `90/100`, `95/100`, **`100/100 SUCCESS`**. These are **not** written to `paperless.log` — they are pushed to the Channels WebSocket group `"status_updates"` by `_send_progress()`, which calls `async_to_sync(self.channel_layer.group_send)("status_updates", ...)` ([`src/documents/consumer.py:L56-L77`](../../src/documents/consumer.py)). Only `self.log(...)` calls appear in the file log. The progress step source lines are, for completeness:

| Progress | Source | Message const |
|---|---|---|
| `0/100 STARTING` `new_file` | [`consumer.py:L202`](../../src/documents/consumer.py) | `MESSAGE_NEW_FILE = "new_file"` ([`consumer.py:L43`](../../src/documents/consumer.py)) |
| `20/100 WORKING` | [`consumer.py:L259`](../../src/documents/consumer.py) | parsing |
| `70/100 WORKING` | [`consumer.py:L264`](../../src/documents/consumer.py) | thumbnail |
| `90/100 WORKING` `parse_date` | [`consumer.py:L274`](../../src/documents/consumer.py) | date parsing |
| `95/100 WORKING` `save_document` | [`consumer.py:L294`](../../src/documents/consumer.py) | before `transaction.atomic()` at L298 |
| `100/100 SUCCESS` `finished` | [`consumer.py:L375`](../../src/documents/consumer.py) | `MESSAGE_FINISHED = "finished"` ([`consumer.py:L49`](../../src/documents/consumer.py)) |

Within the atomic block, `document_consumption_finished.send(...)` fires ([`consumer.py:L306`](../../src/documents/consumer.py)), then files are renamed under `FileLock(settings.MEDIA_LOCK)` via `generate_unique_filename(document)` ([`consumer.py:L315-L316`](../../src/documents/consumer.py)).

## O1(d) Edge case observed — duplicate rejection

Re-submitting the **same content** (checksum already present as pk 1) is rejected. From `data/log/paperless.log`:

```text
[2026-07-01 05:39:50,293] [INFO] [paperless.management.consumer] Adding /app/src/../consume/o1_test_dup.pdf to the task queue.
[2026-07-01 05:39:50,409] [ERROR] [paperless.consumer] Not consuming o1_test_dup.pdf: It is a duplicate.
```

That message is `"Not consuming {self.filename}: It is a duplicate."` ([`consumer.py:L112`](../../src/documents/consumer.py)), keyed by `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` ([`consumer.py:L37`](../../src/documents/consumer.py)). The django-q task recorded `success=False`, and the document count stayed at 2 (the duplicate was not stored). The task's `result` field holds the **complete** failure traceback — reproduced here in full (verbatim, nothing elided):

```console
$ python3 manage.py shell -c "from django_q.models import Task; t=Task.objects.get(func='documents.tasks.consume_file', name='o1_test_dup.pdf', success=False); print('success:', t.success); print(t.result)"
success: False
o1_test_dup.pdf: Not consuming o1_test_dup.pdf: It is a duplicate. : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: o1_test_dup.pdf: Not consuming o1_test_dup.pdf: It is a duplicate.
```

```console
$ python3 manage.py shell -c "from documents.models import Document; print('count:', Document.objects.count())"
count: 2
```

> The related unsupported-type path (`MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` [`consumer.py:L44`](../../src/documents/consumer.py), message `"Unsupported mime type {mime_type}"` [`consumer.py:L225`](../../src/documents/consumer.py)) was **not triggered** in this run because a valid PDF was used; it is noted here for completeness but was not observed.

---

# O2 — Does the classifier retrain on every upload, or only under conditions?

## O2(a) Answer: NOT on every upload — it is scheduled (HOURLY) and doubly gated

Training is a **scheduled** task, not an upload hook. The schedule is registered by a migration ([`src/documents/migrations/1001_auto_20201109_1636.py:L10-L14`](../../src/documents/migrations/1001_auto_20201109_1636.py)):

```python
schedule(
    "documents.tasks.train_classifier",
    name="Train the classifier",
    schedule_type=Schedule.HOURLY,
)
```

Confirmed live in the django-q `Schedule` table (`schedule_type='H'` is HOURLY):

```console
$ python3 manage.py shell -c "from django_q.models import Schedule; [print('func:',s.func,'| name:',repr(s.name),'| schedule_type:',s.schedule_type) for s in Schedule.objects.all()]"
func: paperless_mail.tasks.process_mail_accounts | name: 'Check all e-mail accounts' | schedule_type: I
func: documents.tasks.train_classifier | name: 'Train the classifier' | schedule_type: H
func: documents.tasks.index_optimize | name: 'Optimize the index' | schedule_type: D
func: documents.tasks.sanity_check | name: 'Perform sanity check' | schedule_type: W
```

**Uploading does not retrain.** Across both O1 uploads (pk 1 and pk 2) the *only* classifier line was the "no model yet" DEBUG — there is no "Saving updated classifier model" and no "Training tags classifier" per upload. Grepping the entire consumer log for classifier/training activity during the two ingestions returns exactly two lines (one per upload), and nothing else:

```console
$ grep -E 'paperless.classifier|Saving updated classifier|Training tags classifier|Training data unchanged' data/log/paperless.log
[2026-07-01 05:38:17,401] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:39:23,729] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```

That line is `"Document classification model does not exist (yet), not performing automatic matching."` ([`src/documents/classifier.py:L33`](../../src/documents/classifier.py)), emitted by `load_classifier()` ([`classifier.py:L30`](../../src/documents/classifier.py)) called during consumption at [`consumer.py:L292`](../../src/documents/consumer.py) — i.e. the consume path only *reads* a model to auto-match; it never trains one.

### The two gates

Even when the scheduled task (or the manual trigger `python3 manage.py document_create_classifier` → `train_classifier()` [`src/documents/management/commands/document_create_classifier.py:L20`](../../src/documents/management/commands/document_create_classifier.py)) runs, training is gated twice:

- **Gate 1 — is any auto-matcher configured?** `train_classifier()` returns *silently* if no `Tag`/`DocumentType`/`Correspondent` uses `MATCH_AUTO` ([`src/documents/tasks.py:L49-L55`](../../src/documents/tasks.py); `MATCH_AUTO = 6` [`src/documents/models.py:L26`](../../src/documents/models.py)). Observed: with zero auto-matchers, the command exits 0 and adds **no** classifier/tasks log lines at all:

```console
$ python3 manage.py document_create_classifier ; echo "exit=$?"
exit=0
$ # paperless.log line count was identical before and after (37 → 37): silent early return, no new classifier/paperless.tasks lines.
```

- **Gate 2 — did the training data change?** `DocumentClassifier.train()` ([`classifier.py:L115`](../../src/documents/classifier.py)) builds a SHA1 over the **non-inbox** documents' preprocessed content + auto-labels — `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` ([`classifier.py:L125-L127`](../../src/documents/classifier.py)) — and returns `False` when the hash is unchanged: `if self.data_hash and new_data_hash == self.data_hash: return False` ([`classifier.py:L163-L164`](../../src/documents/classifier.py)).

## O2(b) The two distinguishing log strings (both captured verbatim)

**TRAINING HAPPENED.** After creating a `MATCH_AUTO` tag and assigning it to a non-inbox document (pk 1), the manual trigger produced (from `data/log/paperless.log`):

```text
[2026-07-01 05:42:13,930] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:42:13,931] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-01 05:42:13,934] [DEBUG] [paperless.classifier] 2 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-01 05:42:14,375] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-01 05:42:14,376] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-01 05:42:14,393] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-01 05:42:14,393] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-01 05:42:14,394] [INFO] [paperless.tasks] Saving updated classifier model to /app/src/../data/classification_model.pickle...
```

The decisive line is the INFO **`Saving updated classifier model to {settings.MODEL_FILE}...`** ([`src/documents/tasks.py:L64-L66`](../../src/documents/tasks.py)), which fires only when `classifier.train()` returned `True` ([`classifier.py:L249`](../../src/documents/classifier.py)); it is immediately followed by `classifier.save()` ([`tasks.py:L67`](../../src/documents/tasks.py)). The intermediate DEBUGs are `"Gathering data from database..."` ([`classifier.py:L123`](../../src/documents/classifier.py)), `"Vectorizing data..."` ([`classifier.py:L193`](../../src/documents/classifier.py)), and `"Training tags classifier..."` ([`classifier.py:L203`](../../src/documents/classifier.py)).

**CLASSIFIER IDLE.** Re-running the same trigger immediately, with the data unchanged, produced (note: no "does not exist yet" line this time — the model now exists and is loaded, then the hash gate short-circuits):

```text
[2026-07-01 05:42:25,293] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-01 05:42:25,297] [DEBUG] [paperless.tasks] Training data unchanged.
```

The decisive line is the DEBUG **`Training data unchanged.`** ([`src/documents/tasks.py:L69`](../../src/documents/tasks.py)), reached because `train()` returned `False` via the hash gate ([`classifier.py:L163-L164`](../../src/documents/classifier.py)).

| State | Log string | Level / logger | Source |
|---|---|---|---|
| **Training happened** | `Saving updated classifier model to {MODEL_FILE}...` | INFO `paperless.tasks` | [`tasks.py:L64-L66`](../../src/documents/tasks.py) |
| **Classifier idle (unchanged)** | `Training data unchanged.` | DEBUG `paperless.tasks` | [`tasks.py:L69`](../../src/documents/tasks.py) |
| **Idle (no auto-matchers)** | *(silent — no log)* | — | [`tasks.py:L49-L55`](../../src/documents/tasks.py) |
| **No model at match time** | `Document classification model does not exist (yet), not performing automatic matching.` | DEBUG `paperless.classifier` | [`classifier.py:L33`](../../src/documents/classifier.py) |

## O2(c) Model persistence facts (version-skew guard)

The model is a **plain pickle** at `MODEL_FILE = DATA_DIR/classification_model.pickle` ([`settings.py:L74`](../../src/paperless/settings.py)) with `FORMAT_VERSION = 7` ([`classifier.py:L63`](../../src/documents/classifier.py)) and **no HMAC**. It is written by two bare `pickle.dump` calls — first the version, then the hash ([`classifier.py:L101-L102`](../../src/documents/classifier.py)). Verified by reading the on-disk header:

```console
$ ls -la /app/data/classification_model.pickle
-rw-r--r-- 1 root testuser 145789 Jul  1 05:42 /app/data/classification_model.pickle
$ python3 -c "import pickle; f=open('/app/data/classification_model.pickle','rb'); v=pickle.load(f); print('schema_version:', v); h=pickle.load(f); print('data_hash type:', type(h).__name__, 'len:', len(h), 'bytes'); print('data_hash hex:', h.hex())"
schema_version: 7
data_hash type: bytes len: 20 bytes
data_hash hex: 4c85794518b605f336c76592e79aa8ebfd459da6
```

The first pickled value is `7` (the `FORMAT_VERSION`), the second is the 20-byte SHA1 `data_hash` — i.e. a bare pickle stream, no signed/HMAC wrapper.

## O2(d) Rationale (the *why*)

The classifier is deliberately decoupled from upload: it runs on a **fixed HOURLY schedule** and, when it runs, it (1) does nothing unless at least one label uses automatic matching (Gate 1), and (2) recomputes a content+label **hash** and skips training when that hash is unchanged (Gate 2). So an upload only affects training at the *next scheduled run*, and *only if* it altered the non-inbox training set. This is corroborated by the project docs, which state that paperless "periodically (default: once each hour) checks for changes" and that "the Auto matching algorithm only takes documents into account which are NOT placed in your inbox" ([`docs/advanced_usage.rst:L76-L79`](../../docs/advanced_usage.rst)).

---

# O3 — On-disk location, default directory structure & filename pattern, and DB tables

## O3(a) Where it lands on disk & O3(b) the default directory structure

With default configuration, files are written under `MEDIA_ROOT/documents/...`. Live settings values:

```console
$ python3 -c "import os,django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); from django.conf import settings as s; print('MEDIA_ROOT:', s.MEDIA_ROOT); print('ORIGINALS_DIR:', s.ORIGINALS_DIR); print('ARCHIVE_DIR:', s.ARCHIVE_DIR); print('THUMBNAIL_DIR:', s.THUMBNAIL_DIR); print('DB NAME:', s.DATABASES['default']['NAME']); print('DB ENGINE:', s.DATABASES['default']['ENGINE']); print('PAPERLESS_FILENAME_FORMAT:', repr(s.PAPERLESS_FILENAME_FORMAT))"
MEDIA_ROOT: /app/src/../media
ORIGINALS_DIR: /app/src/../media/documents/originals
ARCHIVE_DIR: /app/src/../media/documents/archive
THUMBNAIL_DIR: /app/src/../media/documents/thumbnails
DB NAME: /app/src/../data/db.sqlite3
DB ENGINE: django.db.backends.sqlite3
PAPERLESS_FILENAME_FORMAT: None
```

These come from `MEDIA_ROOT` ([`settings.py:L61`](../../src/paperless/settings.py)), `ORIGINALS_DIR` ([`settings.py:L62`](../../src/paperless/settings.py)), `ARCHIVE_DIR` ([`settings.py:L63`](../../src/paperless/settings.py)), `THUMBNAIL_DIR` ([`settings.py:L64`](../../src/paperless/settings.py)), and the SQLite `NAME` at `DATA_DIR/db.sqlite3` ([`settings.py:L300`](../../src/paperless/settings.py)). `/app/src/../media` normalizes to `/app/media`.

A real directory listing after ingesting six documents (`find media/documents -type f | sort`, verbatim):

```text
media/documents/archive/0000001.pdf
media/documents/archive/0000002.pdf
media/documents/archive/0000003.pdf
media/documents/archive/0000004.pdf
media/documents/archive/0000005.pdf
media/documents/archive/0000006.pdf
media/documents/originals/0000001.pdf
media/documents/originals/0000002.pdf
media/documents/originals/0000003.pdf
media/documents/originals/0000004.pdf
media/documents/originals/0000005.pdf
media/documents/originals/0000006.pdf
media/documents/thumbnails/0000001.png
media/documents/thumbnails/0000002.png
media/documents/thumbnails/0000003.png
media/documents/thumbnails/0000004.png
media/documents/thumbnails/0000005.png
media/documents/thumbnails/0000006.png
```

So one ingested PDF produces up to three artifacts:

- **original** → `media/documents/originals/0000001.pdf` (`Document.source_path` [`src/documents/models.py:L223`](../../src/documents/models.py))
- **archive** (the OCR'd PDF/A) → `media/documents/archive/0000001.pdf` (`Document.archive_path` [`models.py:L242`](../../src/documents/models.py))
- **thumbnail** → `media/documents/thumbnails/0000001.png` (`Document.thumbnail_path` [`models.py:L273`](../../src/documents/models.py))

`ls -l` shows real sizes (original = the input bytes; archive = OCR'd PDF/A; thumbnail = PNG). The complete, verbatim listing for all six ingested documents (`0000001`–`0000006`), with nothing elided:

```console
$ ls -l media/documents/*/
media/documents/archive/:
total 52
-rw-r--r-- 1 root testuser 8137 Jul  1 05:38 0000001.pdf
-rw-r--r-- 1 root testuser 8405 Jul  1 05:39 0000002.pdf
-rw-r--r-- 1 root testuser 7921 Jul  1 05:43 0000003.pdf
-rw-r--r-- 1 root testuser 8002 Jul  1 05:47 0000004.pdf
-rw-r--r-- 1 root testuser 8141 Jul  1 05:47 0000005.pdf
-rw-r--r-- 1 root testuser 8133 Jul  1 05:48 0000006.pdf

media/documents/originals/:
total 24
-rw-r--r-- 1 root testuser 1522 Jul  1 05:38 0000001.pdf
-rw-r--r-- 1 root testuser 1533 Jul  1 05:39 0000002.pdf
-rw-r--r-- 1 root testuser 1513 Jul  1 05:43 0000003.pdf
-rw-r--r-- 1 root testuser 1521 Jul  1 05:47 0000004.pdf
-rw-r--r-- 1 root testuser 1522 Jul  1 05:47 0000005.pdf
-rw-r--r-- 1 root testuser 1522 Jul  1 05:48 0000006.pdf

media/documents/thumbnails/:
total 60
-rw-r--r-- 1 root testuser 8645 Jul  1 05:38 0000001.png
-rw-r--r-- 1 root testuser 9516 Jul  1 05:39 0000002.png
-rw-r--r-- 1 root testuser 7942 Jul  1 05:43 0000003.png
-rw-r--r-- 1 root testuser 7999 Jul  1 05:47 0000004.png
-rw-r--r-- 1 root testuser 8116 Jul  1 05:47 0000005.png
-rw-r--r-- 1 root testuser 8387 Jul  1 05:48 0000006.png
```

The ORM agrees for pk 1:

```console
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); print('filename:', d.filename); print('source_path:', d.source_path); print('archive_path:', d.archive_path); print('thumbnail_path:', d.thumbnail_path); print('storage_type:', d.storage_type); print('mime_type:', d.mime_type)"
filename: 0000001.pdf
source_path: /app/src/../media/documents/originals/0000001.pdf
archive_path: /app/src/../media/documents/archive/0000001.pdf
thumbnail_path: /app/src/../media/documents/thumbnails/0000001.png
storage_type: unencrypted
mime_type: application/pdf
```

Two other filesystem sinks (not database tables): file writes are serialized by a `FileLock` on `media/media.lock` ([`settings.py:L72`](../../src/paperless/settings.py)) and the Whoosh full-text index lives at `DATA_DIR/index` ([`settings.py:L73`](../../src/paperless/settings.py)), updated by the `add_to_index` handler ([`handlers.py:L428`](../../src/documents/signals/handlers.py)):

```console
$ ls data/index
MAIN_21e7p16s55cxtrq2.seg
MAIN_WRITELOCK
_MAIN_7.toc
```

## O3(c) The default filename pattern: `0000001.pdf`

The first document (pk 1) is stored as **`0000001.pdf`** — a **zero-padded 7-digit primary key** plus the file extension. This is produced by `generate_filename()` ([`src/documents/file_handling.py:L128`](../../src/documents/file_handling.py)):

```python
counter_str = f"_{counter:02}" if counter else ""          # file_handling.py:L186
filetype_str = ".pdf" if archive_filename else doc.file_type  # file_handling.py:L188
if len(path) > 0:
    filename = f"{path}{counter_str}{filetype_str}"           # file_handling.py:L191
else:
    filename = f"{doc.pk:07}{counter_str}{filetype_str}"      # file_handling.py:L193  -> "0000001.pdf"
```

**Rationale:** the directory portion `path` is only populated when `settings.PAPERLESS_FILENAME_FORMAT is not None` ([`file_handling.py:L132`](../../src/documents/file_handling.py)). Because that setting is unset (default `None`, [`settings.py:L584`](../../src/paperless/settings.py)), `path` stays `""` and the `else` branch runs: `f"{doc.pk:07}{filetype_str}"` → `0000001.pdf`. `counter_str` adds `_01`, `_02`, … only when two documents would resolve to the same name. The observed run confirms the sequence `0000001.pdf`, `0000002.pdf`, … `0000006.pdf` = pk 1..6. The upstream config docs corroborate the default: `PAPERLESS_FILENAME_FORMAT` "Default is none, which disables this feature." ([`docs/configuration.rst:L108-L112`](../../docs/configuration.rst)).

## O3(d) Which database tables receive new rows

The default database is **SQLite** at `data/db.sqlite3` ([`settings.py:L300`](../../src/paperless/settings.py)). A temporary script, `dbdiff.py` (shown in full in *How this was observed*), snapshots per-table `COUNT(*)` (over all `sqlite_master` tables) **before** and **after** ingesting a document and prints only the tables whose count changed. Because O3 documents **default** out-of-the-box behavior — and a fresh install ships **no** classifier model (see O2) — the transiently-trained model from O2 was removed before these diffs, so ingestion performs no automatic tag-matching (the default state). The two scheduled-task windows (the interval mail task fires every 10 minutes) were avoided, so each `django_q_task` delta is exactly the single `consume_file` row.

**Diff #1 — ingest one document with no inbox tag defined (no tag applied), verbatim:**

```text
TABLE                                      BEFORE    AFTER   DELTA
----------------------------------------------------------------------
django_admin_log                                3        4      +1
django_q_task                                   9       10      +1
documents_document                              3        4      +1
```

**Diff #2 — define an inbox tag, then ingest one document (tag auto-applied by `add_inbox_tags`), verbatim:**

```text
TABLE                                      BEFORE    AFTER   DELTA
----------------------------------------------------------------------
django_admin_log                                5        6      +1
django_q_task                                  11       12      +1
documents_document                              5        6      +1
documents_document_tags                         2        3      +1
```

The tag delta in Diff #2 is confirmed on the document itself (Diff #1's document has none):

```console
$ python3 manage.py shell -c "from documents.models import Document; print('doc4 tags:', list(Document.objects.get(pk=4).tags.values_list('name', flat=True))); print('doc6 tags:', list(Document.objects.get(pk=6).tags.values_list('name', flat=True)))"
doc4 tags: []
doc6 tags: ['Inbox']
```

Interpreting the observed diffs:

| Table | Δ per ingestion | Why | Source |
|---|---|---|---|
| **`documents_document`** | **+1 (always)** | one row per document (`id`/pk was 4 then 6 in the two diffs) | model `Document` [`models.py:L88`](../../src/documents/models.py) |
| **`django_admin_log`** | **+1 (always)** | `set_log_entry` creates a `LogEntry` `ADDITION` as user `consumer` | [`handlers.py:L413-L425`](../../src/documents/signals/handlers.py) |
| **`django_q_task`** | **+1 per consumed file** | the completed `consume_file` task record (django-q) | [`tasks.py:L184`](../../src/documents/tasks.py) |
| **`documents_document_tags`** | **+1 (conditional)** | tag M2M join — only when a tag is applied; observed when the inbox tag was auto-assigned by `add_inbox_tags` | M2M [`models.py:L128`](../../src/documents/models.py) |

**Conditional / not observed in this run (flagged explicitly):**

- `documents_correspondent`, `documents_tag`, `documents_documenttype` — gain rows **only when a new label is created** (as was done manually to set up O2/O3), **not** as a side effect of ingesting a document. Across both diffs (`snap_before1` → `snap_after2`) their counts were unchanged: `documents_correspondent 0→0`, `documents_documenttype 0→0`.
- `documents_log` (paperless's own `Log` model [`models.py:L285`](../../src/documents/models.py)) — **stayed at 0** in both diffs (not written during a normal PDF consume here).
- `django_q_schedule` — **unchanged (4→4)**; schedules are created once at migration time, not per ingestion.

Finally, the abstract base model `MatchingModel` ([`models.py:L19`](../../src/documents/models.py), `abstract = True` [`models.py:L50`](../../src/documents/models.py)) has **no table of its own** — confirmed by listing the `documents_*` tables (via Python's `sqlite3` module, since the `sqlite3` CLI is not present in the image):

```console
$ python3 -c "import sqlite3; con=sqlite3.connect('file:/app/data/db.sqlite3?mode=ro', uri=True); print('\n'.join(r[0] for r in con.execute(\"SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'documents_%' ORDER BY name\")))"
documents_correspondent
documents_document
documents_document_tags
documents_documenttype
documents_log
documents_savedview
documents_savedviewfilterrule
documents_tag
```

There is **no `documents_matchingmodel` table** — the abstract base contributes its columns (`match`, `matching_algorithm`, `is_insensitive`) to each concrete table (`documents_tag`, `documents_correspondent`, `documents_documenttype`) instead.

---

# How this was observed (exact commands)

Every quoted value above came from the following reproducible steps, run inside the container `pngx-fix` (image `paperless-ngx-ready:latest`, Python 3.9.23, HEAD `542221a38dff...`). Temporary scripts and test PDFs listed here were created inside the container's `/tmp` (outside the tracked repository tree) and removed afterward; the repository is left unchanged.

### 1. Start a fresh container (empty DB → first document gets pk=1) and the full stack

```bash
docker run -d --name pngx-fix --entrypoint sleep paperless-ngx-ready:latest infinity
docker exec pngx-fix /usr/local/bin/paperless-start.sh
#   paperless-start.sh (from the image) sets PAPERLESS_DEBUG=true and runs, in order:
#     redis-server --daemonize yes ; manage.py migrate ; manage.py collectstatic ;
#     createsuperuser admin/admin ; then, backgrounded:
#       nohup manage.py qcluster           > /app/data/log/qcluster.log 2>&1 &
#       nohup manage.py document_consumer  > /app/data/log/consumer.log 2>&1 &
#       nohup gunicorn -c gunicorn.conf.py paperless.asgi:application > /app/data/log/gunicorn.log 2>&1 &
```

### 2. Sanity checks

```bash
docker exec pngx-fix python3 --version                 # Python 3.9.23
docker exec pngx-fix git -C /app rev-parse HEAD        # 542221a38dff06361e07976452f9aea24d210542
docker exec pngx-fix bash -lc "ps -eo pid,ppid,cmd --sort=pid | grep -E 'redis-server|manage.py qcluster|manage.py document_consumer|paperless.asgi:application' | grep -v grep | awk '\$2==1'"
```

### 3. O1 — generate a small PDF, submit BOTH ways, tail the logs

The test PDF is generated with reportlab (exact command; produced `/tmp/o1_test.pdf`, **1522 bytes** as reported by `ls -l`):

```bash
docker exec pngx-fix bash -lc 'python3 -c "from reportlab.pdfgen import canvas; c=canvas.Canvas(\"/tmp/o1_test.pdf\"); c.drawString(72,720,\"Paperless-ngx runtime ingestion investigation - O1 test document.\"); c.drawString(72,700,\"Invoice ACME Corporation 2026-07-01 total 42.00 USD.\"); c.save()"'

# Entry point 1 — consume directory:
docker exec pngx-fix bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test.pdf'

# Entry point 2 — a distinct PDF (so it is not a duplicate) then POST it:
docker exec pngx-fix bash -lc 'python3 -c "from reportlab.pdfgen import canvas; c=canvas.Canvas(\"/tmp/o1_post.pdf\"); c.drawString(72,720,\"Paperless-ngx runtime ingestion investigation - O1 POST document.\"); c.drawString(72,700,\"Receipt Globex Inc 2026-07-02 total 99.50 USD reference POST-2.\"); c.save()"'
docker exec pngx-fix bash -lc 'cd /app/src && python3 /tmp/post_document.py /tmp/o1_post.pdf'   # HTTP 200, body "OK"

# Read the log windows (each sliced from the line position recorded just before ingestion):
docker exec pngx-fix bash -lc 'cat /app/data/log/consumer.log'    # watcher: "Adding ... to the task queue."
docker exec pngx-fix bash -lc 'cat /app/data/log/qcluster.log'    # [Q] worker lifecycle + consumer sequence
docker exec pngx-fix bash -lc 'cat /app/data/log/paperless.log'   # dedicated file log (file_paperless handler)

# Duplicate edge case (same content as pk 1):
docker exec pngx-fix bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test_dup.pdf'
```

The complete `post_document.py` (stdlib only — the image ships no `curl`):

```python
#!/usr/bin/env python3
"""Multipart POST to /api/documents/post_document/ using only the stdlib
(the image ships no curl). Field name "document" per DocumentSerializer;
HTTP Basic auth admin:admin. Prints the HTTP status and response body."""
import base64
import sys
import urllib.request
import uuid

url = "http://localhost:8000/api/documents/post_document/"
path = sys.argv[1]
filename = path.rsplit("/", 1)[-1]

with open(path, "rb") as fh:
    file_bytes = fh.read()

boundary = uuid.uuid4().hex
pre = (
    f"--{boundary}\r\n"
    f'Content-Disposition: form-data; name="document"; filename="{filename}"\r\n'
    f"Content-Type: application/pdf\r\n\r\n"
).encode()
post = f"\r\n--{boundary}--\r\n".encode()
body = pre + file_bytes + post

req = urllib.request.Request(url, data=body, method="POST")
req.add_header("Content-Type", f"multipart/form-data; boundary={boundary}")
req.add_header(
    "Authorization",
    "Basic " + base64.b64encode(b"admin:admin").decode(),
)
with urllib.request.urlopen(req) as resp:
    print("HTTP_STATUS:", resp.status)
    print("RESPONSE_BODY:", repr(resp.read().decode()))
```

### 4. O2 — schedule query, then force the TRAIN and IDLE branches

```bash
# The scheduled training entry (django-q Schedule table):
docker exec pngx-fix bash -lc "cd /app/src && python3 manage.py shell -c \"from django_q.models import Schedule; [print('func:',s.func,'| name:',repr(s.name),'| schedule_type:',s.schedule_type) for s in Schedule.objects.all()]\""

# Gate 1 (no auto-matcher configured): silent early return
docker exec pngx-fix bash -lc 'cd /app/src && python3 manage.py document_create_classifier ; echo "exit=$?"'

# Create + assign a MATCH_AUTO tag to a non-inbox document, then train (make_auto_tag.py below):
docker cp make_auto_tag.py pngx-fix:/tmp/make_auto_tag.py
docker exec pngx-fix bash -lc 'cd /app/src && python3 manage.py shell < /tmp/make_auto_tag.py'
docker exec pngx-fix bash -lc 'cd /app/src && PAPERLESS_DEBUG=true python3 manage.py document_create_classifier'  # TRAIN -> "Saving updated classifier model to ..."
docker exec pngx-fix bash -lc 'cd /app/src && PAPERLESS_DEBUG=true python3 manage.py document_create_classifier'  # IDLE  -> "Training data unchanged."

# Read the plain-pickle header (schema version, then the 20-byte SHA1 hash):
docker exec pngx-fix bash -lc "python3 -c \"import pickle; f=open('/app/data/classification_model.pickle','rb'); v=pickle.load(f); print('schema_version:', v); h=pickle.load(f); print('data_hash type:', type(h).__name__, 'len:', len(h), 'bytes'); print('data_hash hex:', h.hex())\""
```

The complete `make_auto_tag.py` (exact code that created and assigned the auto-matching tag):

```python
from documents.models import Document, Tag

# Create a tag that uses automatic (ML) matching: matching_algorithm = MATCH_AUTO (6)
tag, created = Tag.objects.get_or_create(
    name="auto-invoice",
    defaults={"matching_algorithm": Tag.MATCH_AUTO, "match": "invoice"},
)
tag.matching_algorithm = Tag.MATCH_AUTO
tag.save()

# Assign it to a NON-inbox document (pk=1) so it enters the training set
doc = Document.objects.get(pk=1)
doc.tags.add(tag)

print("tag pk:", tag.pk, "| name:", tag.name,
      "| matching_algorithm:", tag.matching_algorithm,
      "| MATCH_AUTO const:", Tag.MATCH_AUTO, "| created:", created)
print("doc", doc.pk, "tags:", list(doc.tags.values_list("name", flat=True)))
print("auto-matchers -> tags:",
      Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).count())
```

Its observed output:

```text
tag pk: 1 | name: auto-invoice | matching_algorithm: 6 | MATCH_AUTO const: 6 | created: True
doc 1 tags: ['auto-invoice']
auto-matchers -> tags: 1
```

### 5. O3 — list media, then run the before/after DB row-count diff

```bash
docker exec pngx-fix bash -lc 'cd /app && find media/documents -type f | sort'
docker exec pngx-fix bash -lc 'cd /app && ls -l media/documents/*/'   # per-file sizes (originals/archive/thumbnails)

# Return to the default no-model state so ingestion does no auto-matching (default behavior):
docker exec pngx-fix bash -lc 'rm -f /app/data/classification_model.pickle'

# Diff #1 — no inbox tag defined -> ingest one document (no tag applied):
docker cp dbdiff.py pngx-fix:/tmp/dbdiff.py
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_before1.json'
docker exec pngx-fix bash -lc 'python3 -c "from reportlab.pdfgen import canvas; c=canvas.Canvas(\"/tmp/o3_doc4.pdf\"); c.drawString(72,720,\"O3 diff document four - distinct delta echo foxtrot golf.\"); c.drawString(72,700,\"Purchase order Umbrella 2026-07-04 total 12.34 ref FOUR.\"); c.save()"'
docker exec pngx-fix bash -lc 'cp /tmp/o3_doc4.pdf /app/consume/o3_doc4.pdf'   # wait for "consumption finished"
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_after1.json'
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py diff /tmp/snap_before1.json /tmp/snap_after1.json'

# Diff #2 — create an inbox tag, then ingest one document (tag auto-applied):
docker exec pngx-fix bash -lc "cd /app/src && python3 manage.py shell -c \"from documents.models import Tag; t,c=Tag.objects.get_or_create(name='Inbox', defaults={'is_inbox_tag':True}); t.is_inbox_tag=True; t.save(); print('inbox tag pk:', t.pk, '| is_inbox_tag:', t.is_inbox_tag, '| created:', c)\""
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_before2.json'
docker exec pngx-fix bash -lc 'python3 -c "from reportlab.pdfgen import canvas; c=canvas.Canvas(\"/tmp/o3_doc6.pdf\"); c.drawString(72,720,\"O3 diff document six - unique lima mike november oscar.\"); c.drawString(72,700,\"Agreement Stark Industries 2026-07-06 total 88.88 ref SIX.\"); c.save()"'
docker exec pngx-fix bash -lc 'cp /tmp/o3_doc6.pdf /app/consume/o3_doc6.pdf'   # wait for "consumption finished"
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_after2.json'
docker exec pngx-fix bash -lc 'python3 /tmp/dbdiff.py diff /tmp/snap_before2.json /tmp/snap_after2.json'
```

The complete `dbdiff.py` (temporary; removed after use — it opens the DB read-only and never writes):

```python
#!/usr/bin/env python3
"""Snapshot per-table COUNT(*) for every table in the live SQLite DB, or diff
two snapshots and print only the tables whose row count changed.

Usage:
    python3 dbdiff.py snapshot <out.json>
    python3 dbdiff.py diff <before.json> <after.json>
"""
import json
import sqlite3
import sys

DB_URI = "file:/app/data/db.sqlite3?mode=ro"  # read-only, never writes


def snapshot(out_path):
    con = sqlite3.connect(DB_URI, uri=True, timeout=30)
    tables = [
        r[0]
        for r in con.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name",
        )
    ]
    counts = {t: con.execute(f'SELECT COUNT(*) FROM "{t}"').fetchone()[0] for t in tables}
    con.close()
    with open(out_path, "w") as fh:
        json.dump(counts, fh)


def diff(before_path, after_path):
    with open(before_path) as fh:
        before = json.load(fh)
    with open(after_path) as fh:
        after = json.load(fh)
    keys = sorted(set(before) | set(after))
    print(f"{'TABLE':<40} {'BEFORE':>8} {'AFTER':>8} {'DELTA':>7}")
    print("-" * 70)
    for k in keys:
        b = before.get(k, 0)
        a = after.get(k, 0)
        if a != b:
            print(f"{k:<40} {b:>8} {a:>8} {a - b:>+7}")


if __name__ == "__main__":
    if sys.argv[1] == "snapshot":
        snapshot(sys.argv[2])
    elif sys.argv[1] == "diff":
        diff(sys.argv[2], sys.argv[3])
```

### 6. Cleanup

All temporary inputs and scripts live under the container's `/tmp` (and `/app/consume`, `/app/media`, `/app/data` inside the ephemeral container) — none of it is inside the tracked repository working tree. After observation the container is discarded, leaving the repository unchanged except for this single documentation file.

---

# Coverage pass (every sub-question answered)

| Question | Sub-part | Answered? | Where |
|---|---|---|---|
| **O1** | Which services/processes participate | ✅ | *The running stack*; O1(a): `document_consumer`, `gunicorn`, **Redis**, `qcluster` (django-q), parser subsystem, + DB/media/Whoosh/WebSocket sinks — tied to `docker/supervisord.conf`, with the four masters quoted from `ps` |
| **O1** | Ordered sequence of log events | ✅ | O1(c): three synchronized, contiguous windows (watcher `consumer.log`, worker `qcluster.log`, file `paperless.log`) for the **same** file `o1_test.pdf`, with a line-by-line `file:line` mapping into `Consumer.try_consume_file()` |
| **O1** | Watcher **and** qcluster worker transcript, same file | ✅ | O1(c) Windows 1 & 2: `Adding ... to the task queue.` (watcher) and the contiguous `processing [o1_test.pdf]` → … → `Processed [o1_test.pdf]` → `recycled worker` (qcluster), same timestamps, no ellipsis |
| **O1** | Both entry points demonstrated | ✅ | O1(b): consume-dir (`Adding ... to the task queue.`) and `POST /api/documents/post_document/` (**HTTP 200**, body `"OK"`) with the exact producing script |
| **O1** | Edge cases | ✅ | O1(d): duplicate → `Not consuming ...: It is a duplicate.`, task `success=False` with the **full** traceback; unsupported-type path noted as **not observed** |
| **O1** | Celery vs django-q | ✅ | Stated **django-q, not Celery** with the `[Q]` banner (`Q Cluster potato-maryland-london-one running.`) and `Q_CLUSTER`/`django_q` citations |
| **O2** | Retrain on every upload, or conditional? | ✅ | O2(a): **NOT per upload** — scheduled `Schedule.HOURLY`, doubly gated (Gate 1 `MATCH_AUTO`; Gate 2 SHA1 data-hash); proven by the grep showing only two "does not exist (yet)" lines across both uploads |
| **O2** | "Training happening" vs "idle" log strings | ✅ | O2(b): INFO `Saving updated classifier model to ...` vs DEBUG `Training data unchanged.` (both captured verbatim), + silent Gate 1 and `does not exist (yet)` |
| **O2** | Rationale | ✅ | O2(d): schedule + two gates; corroborated by `docs/advanced_usage.rst` |
| **O2** | Model persistence (version-skew) | ✅ | O2(c): plain pickle, `FORMAT_VERSION = 7`, 20-byte SHA1 hash (`4c85…`), **no HMAC** |
| **O3** | Where on disk | ✅ | O3(a): `media/documents/{originals,archive,thumbnails}/` under `MEDIA_ROOT=/app/media` |
| **O3** | Default directory structure | ✅ | O3(a/b): quoted `find` listing of the three subdirectories + `media.lock` + Whoosh `data/index` |
| **O3** | Default filename pattern | ✅ | O3(c): **`0000001.pdf`** = `{doc.pk:07}` + extension, quoted `ls`; rationale from `generate_filename()` with `PAPERLESS_FILENAME_FORMAT` unset |
| **O3** | Which DB tables receive new rows | ✅ | O3(d): quoted before/after diffs — always `documents_document`, `django_admin_log`, `django_q_task`; conditional `documents_document_tags` (observed via inbox tag); others flagged as conditional/not-observed; abstract `MatchingModel` has no table |

**Items explicitly flagged as not verified at runtime:** the unsupported-MIME-type rejection path (a valid PDF was used); `documents_log`, `documents_correspondent`/`documents_tag`/`documents_documenttype`, and `django_q_schedule` did not gain rows as a side effect of ingestion in the observed diffs and are marked conditional. Everything else in this document is grounded in the quoted, command-attributed runtime output above and in this commit's source at `542221a38dff06361e07976452f9aea24d210542`.
