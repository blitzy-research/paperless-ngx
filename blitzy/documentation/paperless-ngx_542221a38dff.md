# paperless-ngx — Runtime Behavior of Document Ingestion

**Branch:** `paperless-ngx_542221a38dff` · **Commit (HEAD):** `542221a38dff06361e07976452f9aea24d210542`

This document answers three questions about how **paperless-ngx** behaves at runtime while it ingests a document:

- **O1 — The ingestion pipeline.** When a test PDF is submitted, *which services participate* and *what ordered sequence of events appears in the logs* as the document moves through the system?
- **O2 — Classifier retraining.** Does the machine-learning classifier retrain automatically on **every** upload, or **only under certain conditions**? What log messages distinguish "training is happening" from "the classifier is idle"?
- **O3 — Storage & database.** After a document finishes processing, *where does it land on disk*, what is the *default directory structure and filename pattern*, and *which database tables receive new rows*?

> Every answer below was produced **run-first**: the full stack was built and run inside the mandated Docker image, a real PDF was pushed through it, and the **actual output was captured**. Quoted log lines, filenames, table names, HTTP responses, and measured values are **verbatim** from that run. Each factual claim carries a `file:line` citation to this commit's source. Where a value could not be produced at runtime, it is called out explicitly.

---

## Methodology (run-first, read-only)

- **Read-only:** No file in the source tree was modified. The only artifact added to the repository is this document. All source references below are **read-only citations**.
- **Environment:** The host cannot run paperless-ngx (Python 3.13, no Redis/tesseract/scikit-learn). All observation was done inside the provided image `paperless-ngx-ready:latest` (derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...`), which ships **Python 3.9** ([`Dockerfile:L18`](../../Dockerfile) → `FROM python:3.9-slim-bullseye`) and the OCR binaries `tesseract-ocr` (+eng/deu/fra/ita/spa) ([`Dockerfile:L63-L68`](../../Dockerfile)), `ghostscript` ([`Dockerfile:L41`](../../Dockerfile)) and `imagemagick` ([`Dockerfile:L45`](../../Dockerfile)).
- **Default configuration:** `PAPERLESS_FILENAME_FORMAT` was left **unset** so O3 documents out-of-the-box behavior; its default is `None` ([`src/paperless/settings.py:L584`](../../src/paperless/settings.py)).
- **DEBUG visible:** `PAPERLESS_DEBUG=true` so `paperless.consumer`/`paperless.classifier` DEBUG lines surface. Mechanically, `DEBUG` defaults `False` ([`settings.py:L50`](../../src/paperless/settings.py)); the `paperless` logger is level `DEBUG` writing to a file handler ([`settings.py:L409`](../../src/paperless/settings.py)) → `data/log/paperless.log`, so DEBUG lines land there regardless, while the console shows DEBUG only when `PAPERLESS_DEBUG=true` ([`settings.py:L388`](../../src/paperless/settings.py)).

Verification that we ran the intended commit, in the intended runtime:

```console
$ docker exec pngx-investigate python3 --version
Python 3.9.23
$ docker exec pngx-investigate git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ docker exec pngx-investigate bash -lc 'echo "PAPERLESS_FILENAME_FORMAT=[${PAPERLESS_FILENAME_FORMAT}]"'
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

All four were confirmed live:

```console
$ docker exec pngx-investigate bash -lc "ps aux | grep -E 'qcluster|document_consumer|gunicorn|redis-server' | grep -v grep"
root   44  redis-server *:6379
root   80  python3 manage.py qcluster
root   81  python3 manage.py document_consumer
root   82  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
...
```

Startup "ready" lines (verbatim):

```text
# data/log/gunicorn.log
[2026-07-01 04:54:33 +0000] [82] [INFO] Starting gunicorn 20.1.0
[2026-07-01 04:54:33 +0000] [82] [INFO] Listening at: http://0.0.0.0:8000 (82)
[2026-07-01 04:54:33 +0000] [82] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 04:54:33 +0000] [82] [INFO] Server is ready. Spawning workers

# data/log/qcluster.log  (note the [Q] django-q banner — NOT Celery)
04:54:34 [Q] INFO Q Cluster florida-ack-salami-earth starting.
04:54:34 [Q] INFO Process-1:1 ready for work at 111
04:54:34 [Q] INFO Q Cluster florida-ack-salami-earth running.

# data/log/consumer.log
[2026-07-01 04:54:34,416] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

The API answered on port 8000 (basic auth `admin:admin`):

```console
$ # GET /api/  (via python urllib; curl is not in the image)
HTTP_STATUS: 200
BODY_HEAD: {"correspondents":"http://localhost:8000/api/correspondents/","document_types":...,"documents":"http://localhost:8000/api/documents/","logs":...,"tags":...}
```

A Django user named **`consumer`** must exist, because the `set_log_entry` consumption handler calls `User.objects.get(username="consumer")` ([`src/documents/signals/handlers.py:L416`](../../src/documents/signals/handlers.py)); if absent, the consume transaction fails. Both users were present:

```console
$ python3 manage.py shell -c "from django.contrib.auth.models import User; print(list(User.objects.values_list('username', flat=True)))"
USERS: ['admin', 'consumer']
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

> **This is django-q, not Celery.** The worker log prints `Q Cluster florida-ack-salami-earth running.` and every task line is prefixed `[Q]` (quoted above and below).

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
$ docker exec pngx-investigate bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test.pdf'
```

The watcher enqueued it (from `data/log/consumer.log`, logger `paperless.management.consumer`):

```text
[2026-07-01 04:56:20,200] [INFO] [paperless.management.consumer] Adding /app/src/../consume/o1_test.pdf to the task queue.
04:56:20 [Q] INFO Enqueued 1
```

**Entry point 2 — `POST /api/documents/post_document/` (drives web → queue).** A multipart upload (field name `document`) with basic auth returned, **verbatim**:

```text
HTTP_STATUS: 200
RESPONSE_BODY: '"OK"'
```

That matches `return Response("OK")` ([`views.py:L535`](../../src/documents/views.py)). The worker then processed it (from `data/log/qcluster.log`):

```text
04:57:56 [Q] INFO Process-1:6 processing [o1_post.pdf]
...
04:58:00 [Q] INFO Processed [o1_post.pdf]
04:58:00 [Q] INFO recycled worker Process-1:6
```

Both tasks' return values (stored in the django-q task table) confirm success and the created document ids — matching `"Success. New document id {} created"` ([`tasks.py:L247`](../../src/documents/tasks.py)):

```console
$ python3 manage.py shell -c "from django_q.models import Task; [print(t.func,'| success:',t.success,'| result:',repr(t.result)) for t in Task.objects.filter(func='documents.tasks.consume_file').order_by('started')]"
documents.tasks.consume_file | success: True | result: 'Success. New document id 1 created'
documents.tasks.consume_file | success: True | result: 'Success. New document id 2 created'
```

## O1(c) The ordered log sequence (verbatim transcript)

Below is the **actual** `data/log/paperless.log` window for the consume-dir document (`o1_test.pdf`). The POST document (`o1_post.pdf`) produced the identical ordering.

```text
[2026-07-01 04:56:20,200] [INFO]  [paperless.management.consumer] Adding /app/src/../consume/o1_test.pdf to the task queue.
[2026-07-01 04:56:20,313] [INFO]  [paperless.consumer] Consuming o1_test.pdf
[2026-07-01 04:56:20,313] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-01 04:56:20,316] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 04:56:20,318] [DEBUG] [paperless.consumer] Parsing o1_test.pdf...
[2026-07-01 04:56:20,340] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /app/src/../consume/o1_test.pdf
[2026-07-01 04:56:20,472] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '.../o1_test.pdf', 'output_file': '.../archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '.../sidecar.txt'}
[2026-07-01 04:56:20,747] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-01 04:56:20,752] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-w8utc308/archive.pdf
[2026-07-01 04:56:20,752] [DEBUG] [paperless.consumer] Generating thumbnail for o1_test.pdf...
[2026-07-01 04:56:20,756] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient .../archive.pdf[0] .../convert.png
[2026-07-01 04:56:21,443] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 .../convert.png -out .../thumb_optipng.png
[2026-07-01 04:56:24,252] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 04:56:24,255] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 04:56:24,274] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/o1_test.pdf
[2026-07-01 04:56:24,297] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-w8utc308
[2026-07-01 04:56:24,298] [INFO]  [paperless.consumer] Document 2026-07-01 o1_test consumption finished
```

Mapping each observed line to its source (all in `Consumer.try_consume_file()`, [`src/documents/consumer.py:L180+`](../../src/documents/consumer.py), logger `paperless.consumer` unless noted):

| Observed line | Source `file:line` |
|---|---|
| `Adding ... to the task queue.` (logger `paperless.management.consumer`) | [`document_consumer.py:L85`](../../src/documents/management/commands/document_consumer.py) |
| `Consuming o1_test.pdf` | [`consumer.py:L215`](../../src/documents/consumer.py) |
| `Detected mime type: application/pdf` | [`consumer.py:L221`](../../src/documents/consumer.py) |
| `Parser: RasterisedDocumentParser` | [`consumer.py:L246`](../../src/documents/consumer.py) |
| `Parsing o1_test.pdf...` | [`consumer.py:L260`](../../src/documents/consumer.py) |
| `Extracted text from PDF file ...` / `Calling OCRmyPDF with args: ...` | [`src/paperless_tesseract/parsers.py:L122`](../../src/paperless_tesseract/parsers.py), [`parsers.py:L260`](../../src/paperless_tesseract/parsers.py) |
| `Generating thumbnail for o1_test.pdf...` | [`consumer.py:L263`](../../src/documents/consumer.py) |
| `Document classification model does not exist (yet)...` (logger `paperless.classifier`) | [`classifier.py:L33`](../../src/documents/classifier.py) via `load_classifier()` at [`consumer.py:L292`](../../src/documents/consumer.py) |
| `Saving record to database` | [`consumer.py:L387`](../../src/documents/consumer.py) (inside `_store()`, called at L301) |
| `Deleting file /app/src/../consume/o1_test.pdf` | [`consumer.py:L349`](../../src/documents/consumer.py) (`os.unlink(self.path)` at L350) |
| `Document 2026-07-01 o1_test consumption finished` | [`consumer.py:L373`](../../src/documents/consumer.py) |

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
[2026-07-01 04:59:12,097] [INFO]  [paperless.management.consumer] Adding /app/src/../consume/o1_test_dup.pdf to the task queue.
[2026-07-01 04:59:12,213] [ERROR] [paperless.consumer] Not consuming o1_test_dup.pdf: It is a duplicate.
```

That message is `"Not consuming {self.filename}: It is a duplicate."` ([`consumer.py:L112`](../../src/documents/consumer.py)), keyed by `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` ([`consumer.py:L37`](../../src/documents/consumer.py)). The django-q task recorded `success=False`, and the document count stayed at 2 (the duplicate was not stored):

```console
$ python3 manage.py shell -c "from django_q.models import Task; t=Task.objects.filter(func='documents.tasks.consume_file').last(); print('success:', t.success, '| result:', repr(t.result)[:80])"
success: False | result: 'o1_test_dup.pdf: Not consuming o1_test_dup.pdf: It is a duplicate. : Traceback ...'
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
func: documents.tasks.train_classifier          | name: 'Train the classifier'    | schedule_type: H
func: documents.tasks.index_optimize            | name: 'Optimize the index'       | schedule_type: D
func: documents.tasks.sanity_check              | name: 'Perform sanity check'     | schedule_type: W
```

**Uploading does not retrain.** Across both O1 uploads (pk 1 and pk 2) the *only* classifier line was the "no model yet" DEBUG — there is no "Saving updated classifier model" or "Training tags classifier" per upload:

```text
[2026-07-01 04:56:24,252] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```

That line is `"Document classification model does not exist (yet), not performing automatic matching."` ([`src/documents/classifier.py:L33`](../../src/documents/classifier.py)), emitted by `load_classifier()` ([`classifier.py:L30`](../../src/documents/classifier.py)) called during consumption at [`consumer.py:L292`](../../src/documents/consumer.py).

### The two gates

Even when the scheduled task (or the manual trigger `python3 manage.py document_create_classifier` → `train_classifier()` [`src/documents/management/commands/document_create_classifier.py:L20`](../../src/documents/management/commands/document_create_classifier.py)) runs, training is gated twice:

- **Gate 1 — is any auto-matcher configured?** `train_classifier()` returns *silently* if no `Tag`/`DocumentType`/`Correspondent` uses `MATCH_AUTO` ([`src/documents/tasks.py:L49-L55`](../../src/documents/tasks.py); `MATCH_AUTO = 6` [`src/documents/models.py:L26`](../../src/documents/models.py)). Observed: with zero auto-matchers, the command exits 0 and logs **nothing**:

```console
$ python3 manage.py document_create_classifier ; echo "exit=$?"
exit=0
# → no new paperless.classifier / paperless.tasks lines at all (silent early return)
```

- **Gate 2 — did the training data change?** `DocumentClassifier.train()` ([`classifier.py:L115`](../../src/documents/classifier.py)) builds a SHA1 over the **non-inbox** documents' preprocessed content + auto-labels — `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` ([`classifier.py:L125-L127`](../../src/documents/classifier.py)) — and returns `False` when the hash is unchanged: `if self.data_hash and new_data_hash == self.data_hash: return False` ([`classifier.py:L163-L164`](../../src/documents/classifier.py)).

## O2(b) The two distinguishing log strings (both captured verbatim)

**TRAINING HAPPENED.** After creating a `MATCH_AUTO` tag and assigning it to a non-inbox document, the manual trigger produced (from `data/log/paperless.log`):

```text
[2026-07-01 05:02:04,143] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:02:04,144] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-01 05:02:04,146] [DEBUG] [paperless.classifier] 2 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-01 05:02:04,591] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-01 05:02:04,592] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-01 05:02:04,609] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-01 05:02:04,610] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-01 05:02:04,610] [INFO]  [paperless.tasks] Saving updated classifier model to /app/src/../data/classification_model.pickle...
```

The decisive line is the INFO **`Saving updated classifier model to {settings.MODEL_FILE}...`** ([`src/documents/tasks.py:L64-L66`](../../src/documents/tasks.py)), which fires only when `classifier.train()` returned `True` ([`classifier.py:L249`](../../src/documents/classifier.py)); it is immediately followed by `classifier.save()` ([`tasks.py:L67`](../../src/documents/tasks.py)). The intermediate DEBUGs are `"Gathering data from database..."` ([`classifier.py:L123`](../../src/documents/classifier.py)), `"Vectorizing data..."` ([`classifier.py:L193`](../../src/documents/classifier.py)), and `"Training tags classifier..."` ([`classifier.py:L203`](../../src/documents/classifier.py)).

**CLASSIFIER IDLE.** Re-running the same trigger immediately, with the data unchanged, produced:

```text
[2026-07-01 05:02:23,136] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-01 05:02:23,140] [DEBUG] [paperless.tasks] Training data unchanged.
```

The decisive line is the DEBUG **`Training data unchanged.`** ([`src/documents/tasks.py:L69`](../../src/documents/tasks.py)), reached because `train()` returned `False` via the hash gate ([`classifier.py:L163-L164`](../../src/documents/classifier.py)).

| State | Log string | Level / logger | Source |
|---|---|---|---|
| **Training happened** | `Saving updated classifier model to {MODEL_FILE}...` | INFO `paperless.tasks` | [`tasks.py:L64-L66`](../../src/documents/tasks.py) |
| **Classifier idle (unchanged)** | `Training data unchanged.` | DEBUG `paperless.tasks` | [`tasks.py:L69`](../../src/documents/tasks.py) |
| **Idle (no auto-matchers)** | *(silent — no log)* | — | [`tasks.py:L49-L55`](../../src/documents/tasks.py) |
| **No model at match time** | `Document classification model does not exist (yet)...` | DEBUG `paperless.classifier` | [`classifier.py:L33`](../../src/documents/classifier.py) |

## O2(c) Model persistence facts (version-skew guard)

The model is a **plain pickle** at `MODEL_FILE = DATA_DIR/classification_model.pickle` ([`settings.py:L74`](../../src/paperless/settings.py)) with `FORMAT_VERSION = 7` ([`classifier.py:L63`](../../src/documents/classifier.py)) and **no HMAC**. Verified by reading the on-disk header:

```console
$ ls -la /app/data/classification_model.pickle
-rw-r--r-- 1 root testuser 126530 Jul  1 05:02 /app/data/classification_model.pickle
$ python3 -c "import pickle; f=open('/app/data/classification_model.pickle','rb'); print('schema_version:', pickle.load(f)); h=pickle.load(f); print('data_hash:', type(h).__name__, len(h), 'bytes')"
schema_version: 7
data_hash: bytes 20 bytes
```

The first pickled value is `7` (the `FORMAT_VERSION`), the second is the 20-byte SHA1 `data_hash` — i.e. a bare pickle stream, no signed/HMAC wrapper.

## O2(d) Rationale (the *why*)

The classifier is deliberately decoupled from upload: it runs on a **fixed HOURLY schedule** and, when it runs, it (1) does nothing unless at least one label uses automatic matching (Gate 1), and (2) recomputes a content+label **hash** and skips training when that hash is unchanged (Gate 2). So an upload only affects training at the *next scheduled run*, and *only if* it altered the non-inbox training set. This is corroborated by the project docs, which state that paperless "periodically (default: once each hour) checks for changes" and that "the Auto matching algorithm only takes documents into account which are NOT placed in your inbox" ([`docs/advanced_usage.rst:L76-L79`](../../docs/advanced_usage.rst)).

---


# O3 — On-disk location, default directory structure & filename pattern, and DB tables

## O3(a) Where it lands on disk & O3(b) the default directory structure

With default configuration, files are written under `MEDIA_ROOT/documents/...`. Live settings values:

```console
$ python3 -c "... print(settings.MEDIA_ROOT, settings.ORIGINALS_DIR, settings.DATABASES['default']['NAME'], settings.DATABASES['default']['ENGINE'])"
MEDIA_ROOT:     /app/src/../media          -> normalized /app/media
ORIGINALS_DIR:  /app/media/documents/originals
DATABASES NAME: /app/src/../data/db.sqlite3
ENGINE:         django.db.backends.sqlite3
```

These come from `MEDIA_ROOT` ([`settings.py:L61`](../../src/paperless/settings.py)), `ORIGINALS_DIR` ([`settings.py:L62`](../../src/paperless/settings.py)), `ARCHIVE_DIR` ([`settings.py:L63`](../../src/paperless/settings.py)), `THUMBNAIL_DIR` ([`settings.py:L64`](../../src/paperless/settings.py)), and the SQLite `NAME` at `DATA_DIR/db.sqlite3` ([`settings.py:L300`](../../src/paperless/settings.py)).

A real directory listing after ingesting four documents (`find media/documents -type f | sort`, verbatim):

```text
media/documents/archive/0000001.pdf
media/documents/archive/0000002.pdf
media/documents/archive/0000003.pdf
media/documents/archive/0000004.pdf
media/documents/originals/0000001.pdf
media/documents/originals/0000002.pdf
media/documents/originals/0000003.pdf
media/documents/originals/0000004.pdf
media/documents/thumbnails/0000001.png
media/documents/thumbnails/0000002.png
media/documents/thumbnails/0000003.png
media/documents/thumbnails/0000004.png
```

So one ingested PDF produces up to three artifacts:

- **original** → `media/documents/originals/0000001.pdf` (`Document.source_path` [`src/documents/models.py:L223`](../../src/documents/models.py))
- **archive** (the OCR'd PDF/A sidecar) → `media/documents/archive/0000001.pdf` (`Document.archive_path` [`models.py:L242`](../../src/documents/models.py))
- **thumbnail** → `media/documents/thumbnails/0000001.png` (`Document.thumbnail_path` [`models.py:L273`](../../src/documents/models.py))

`ls -l` shows real sizes (original = the input bytes; archive = OCR'd PDF/A; thumbnail = PNG):

```text
media/documents/originals:   0000001.pdf 1494   0000002.pdf 1499
media/documents/archive:     0000001.pdf 7918   0000002.pdf 8263
media/documents/thumbnails:  0000001.png 7612   0000002.png 7788
```

The ORM agrees for pk 1:

```console
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); print(d.filename, d.source_path, d.archive_path, d.thumbnail_path, d.storage_type, d.mime_type)"
filename:       0000001.pdf
source_path:    /app/src/../media/documents/originals/0000001.pdf
archive_path:   /app/src/../media/documents/archive/0000001.pdf
thumbnail_path: /app/src/../media/documents/thumbnails/0000001.png
storage_type:   unencrypted
mime_type:      application/pdf
```

Two other filesystem sinks (not database tables): file writes are serialized by a `FileLock` on `media/media.lock` ([`settings.py:L72`](../../src/paperless/settings.py)) — visible in the listing — and the Whoosh full-text index lives at `DATA_DIR/index` ([`settings.py:L73`](../../src/paperless/settings.py)), updated by the `add_to_index` handler ([`handlers.py:L428`](../../src/documents/signals/handlers.py)):

```console
$ ls data/index
MAIN_68oo6yijvl71liu0.seg   MAIN_WRITELOCK   MAIN_ig17bc1vafrc52sq.seg   _MAIN_3.toc
```

## O3(c) The default filename pattern: `0000001.pdf`

The first document (pk 1) is stored as **`0000001.pdf`** — a **zero-padded 7-digit primary key** plus the file extension. This is produced by `generate_filename()` ([`src/documents/file_handling.py:L128`](../../src/documents/file_handling.py)):

```python
counter_str = f"_{counter:02}" if counter else ""          # file_handling.py:L186
filetype_str = ".pdf" if archive_filename else doc.file_type  # file_handling.py:L188
if len(path) > 0:
    filename = f"{path}{counter_str}{filetype_str}"
else:
    filename = f"{doc.pk:07}{counter_str}{filetype_str}"   # file_handling.py:L193  -> "0000001.pdf"
```

**Rationale:** the directory portion `path` is only populated when `settings.PAPERLESS_FILENAME_FORMAT is not None` ([`file_handling.py:L132`](../../src/documents/file_handling.py)). Because that setting is unset (default `None`, [`settings.py:L584`](../../src/paperless/settings.py)), `path` stays `""` and the `else` branch runs: `f"{doc.pk:07}{filetype_str}"` → `0000001.pdf`. `counter_str` adds `_01`, `_02`, … only when two documents would resolve to the same name. The observed run confirms the sequence `0000001.pdf`, `0000002.pdf`, `0000003.pdf`, `0000004.pdf` = pk 1..4. The upstream config docs corroborate the default: `PAPERLESS_FILENAME_FORMAT` "Default is none, which disables this feature." ([`docs/configuration.rst:L108-L112`](../../docs/configuration.rst)).

## O3(d) Which database tables receive new rows

The default database is **SQLite** at `data/db.sqlite3` ([`settings.py:L300`](../../src/paperless/settings.py)). A temporary script snapshotted per-table `COUNT(*)` (over all `sqlite_master` tables) **before** and **after** ingesting a document.

**Diff #1 — ingest one document (no tag applied), verbatim:**

```text
TABLE                                        BEFORE    AFTER   DELTA
----------------------------------------------------------------------
django_admin_log                                  2        3      +1
django_q_task                                     7        9      +2
documents_document                                2        3      +1
```

**Diff #2 — define an inbox tag, then ingest one document (tag auto-applied), verbatim:**

```text
TABLE                                        BEFORE    AFTER   DELTA
----------------------------------------------------------------------
django_admin_log                                  3        4      +1
django_q_task                                     9       10      +1
documents_document                                3        4      +1
documents_document_tags                           1        2      +1
```

Interpreting the observed diffs:

| Table | Δ per ingestion | Why | Source |
|---|---|---|---|
| **`documents_document`** | **+1 (always)** | one row per document (`id`/pk was 3 then 4 in the two diffs) | model `Document` [`models.py:L88`](../../src/documents/models.py) |
| **`django_admin_log`** | **+1 (always)** | `set_log_entry` creates a `LogEntry` `ADDITION` as user `consumer` | [`handlers.py:L413-L425`](../../src/documents/signals/handlers.py) |
| **`django_q_task`** | **+1 per consumed file** | the completed `consume_file` task record (django-q) | [`tasks.py:L184`](../../src/documents/tasks.py) |
| **`documents_document_tags`** | **+1 (conditional)** | tag M2M join — only when a tag is applied; observed when the inbox tag was auto-assigned | M2M [`models.py:L128`](../../src/documents/models.py), `add_inbox_tags` |

> **Note on `django_q_task = +2` in Diff #1.** Only **one** of those rows is from ingestion (`result: 'Success. New document id 3 created'`); the other is an unrelated scheduled task (`paperless_mail.tasks.process_mail_accounts`, `result: 'No new documents were added.'`) that happened to fire during the wait window. Diff #2 shows the clean **+1** ingestion delta.

**Conditional / not observed in this run (flagged explicitly):**

- `documents_correspondent`, `documents_tag`, `documents_documenttype` — gain rows **only when a new label is created**, which did not happen through ingestion in these diffs.
- `documents_log` (paperless's own `Log` model [`models.py:L285`](../../src/documents/models.py)) — **stayed at 0** in both diffs (not written during a normal PDF consume here).
- `django_q_schedule` — **unchanged**; schedules are created once at migration time, not per ingestion.

Finally, the abstract base model `MatchingModel` ([`models.py:L19`](../../src/documents/models.py), `abstract = True` [`models.py:L50`](../../src/documents/models.py)) has **no table of its own** — confirmed by listing `documents_*` tables:

```console
$ sqlite3 db.sqlite3 "SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'documents_%'"
documents_correspondent   documents_document   documents_document_tags   documents_documenttype
documents_log   documents_savedview   documents_savedviewfilterrule   documents_tag
# → no documents_matchingmodel table
```

---


# How this was observed (exact commands)

Every quoted value above came from the following reproducible steps, run inside the container `pngx-investigate` (image `paperless-ngx-ready:latest`, Python 3.9.23, HEAD `542221a38dff...`). Temporary scripts and test PDFs listed here were created outside the tracked repository tree (in `/tmp`) and removed afterward.

```bash
# 1. Start a fresh container (empty DB → first document gets pk=1) and the full stack (DEBUG on, default config)
docker run -d --name pngx-investigate --entrypoint sleep paperless-ngx-ready:latest infinity
docker exec pngx-investigate /usr/local/bin/paperless-start.sh
#   → redis-server --daemonize + manage.py migrate + collectstatic + createsuperuser(admin/admin)
#     + nohup qcluster + nohup document_consumer + nohup gunicorn(:8000);  logs in /app/data/log/

# 2. Sanity checks
docker exec pngx-investigate python3 --version                 # Python 3.9.23
docker exec pngx-investigate git -C /app rev-parse HEAD        # 542221a38dff06361e07976452f9aea24d210542
docker exec pngx-investigate bash -lc 'ps aux | grep -E "qcluster|document_consumer|gunicorn|redis-server" | grep -v grep'

# 3. O1 — generate a small PDF (reportlab), submit BOTH ways, tail the logs
docker exec pngx-investigate bash -lc 'python3 -c "from reportlab.pdfgen import canvas; ..."'  # → /tmp/o1_test.pdf (1494 bytes)
docker exec pngx-investigate bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test.pdf'           # consume-dir path
#   POST /api/documents/post_document/ (multipart field "document", basic auth admin:admin) via python urllib → HTTP 200, body "OK"
docker exec pngx-investigate bash -lc 'tail -n +3 /app/data/log/paperless.log'                 # ordered consumer sequence
docker exec pngx-investigate bash -lc 'cat /app/data/log/qcluster.log'                          # django-q worker lifecycle
docker exec pngx-investigate bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test_dup.pdf'        # duplicate edge case

# 4. O2 — schedule + force TRAIN then IDLE branches
docker exec pngx-investigate bash -lc 'python3 manage.py shell -c "from django_q.models import Schedule; ..."'
docker exec pngx-investigate bash -lc 'python3 manage.py document_create_classifier'            # Gate 1: silent (no auto-matcher)
#   create Tag(matching_algorithm=MATCH_AUTO), assign to non-inbox doc pk=1, then:
docker exec pngx-investigate bash -lc 'PAPERLESS_DEBUG=true python3 manage.py document_create_classifier'  # TRAIN → "Saving updated classifier model to ..."
docker exec pngx-investigate bash -lc 'PAPERLESS_DEBUG=true python3 manage.py document_create_classifier'  # IDLE  → "Training data unchanged."
docker exec pngx-investigate bash -lc 'python3 -c "import pickle; f=open(\"/app/data/classification_model.pickle\",\"rb\"); print(pickle.load(f)); print(len(pickle.load(f)))"'  # 7, 20

# 5. O3 — list media, before/after DB row-count diff
docker exec pngx-investigate bash -lc 'find /app/media/documents -type f | sort'
docker cp /tmp/dbdiff.py pngx-investigate:/tmp/dbdiff.py
docker exec pngx-investigate bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_before.json'
#   ingest one document (reportlab PDF → consume dir), wait, then:
docker exec pngx-investigate bash -lc 'python3 /tmp/dbdiff.py snapshot /tmp/snap_after.json'
docker exec pngx-investigate bash -lc 'python3 /tmp/dbdiff.py diff /tmp/snap_before.json /tmp/snap_after.json'
```

The `dbdiff.py` snapshot logic (temporary; removed after use):

```python
import sqlite3, json
con = sqlite3.connect("file:/app/data/db.sqlite3?mode=ro", uri=True, timeout=30)
tables = [r[0] for r in con.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
counts = {t: con.execute(f'SELECT COUNT(*) FROM "{t}"').fetchone()[0] for t in tables}
# ...write counts to JSON; a second run diffs before vs after and prints only tables whose count changed.
```

---

# Coverage pass (every sub-question answered)

| Question | Sub-part | Answered? | Where |
|---|---|---|---|
| **O1** | Which services/processes participate | ✅ | *The running stack*; O1(a): `document_consumer`, `gunicorn`, **Redis**, `qcluster` (django-q), parser subsystem, + DB/media/Whoosh/WebSocket sinks — tied to `docker/supervisord.conf` |
| **O1** | Ordered sequence of log events | ✅ | O1(c): verbatim `paperless.log` transcript with a line-by-line `file:line` mapping into `Consumer.try_consume_file()` |
| **O1** | Both entry points demonstrated | ✅ | O1(b): consume-dir (`Adding ... to the task queue.`) and `POST /api/documents/post_document/` (**HTTP 200**, body `"OK"`) |
| **O1** | Edge cases | ✅ | O1(d): duplicate → `Not consuming ...: It is a duplicate.`, task `success=False`; unsupported-type path noted as **not observed** |
| **O1** | Celery vs django-q | ✅ | Stated **django-q, not Celery** with the `[Q]` banner and `Q_CLUSTER`/`django_q` citations |
| **O2** | Retrain on every upload, or conditional? | ✅ | O2(a): **NOT per upload** — scheduled `Schedule.HOURLY`, doubly gated (Gate 1 `MATCH_AUTO`; Gate 2 SHA1 data-hash) |
| **O2** | "Training happening" vs "idle" log strings | ✅ | O2(b): INFO `Saving updated classifier model to ...` vs DEBUG `Training data unchanged.` (both captured verbatim), + silent Gate 1 and `does not exist (yet)` |
| **O2** | Rationale | ✅ | O2(d): schedule + two gates; corroborated by `docs/advanced_usage.rst` |
| **O2** | Model persistence (version-skew) | ✅ | O2(c): plain pickle, `FORMAT_VERSION = 7`, 20-byte SHA1 hash, **no HMAC** |
| **O3** | Where on disk | ✅ | O3(a): `media/documents/{originals,archive,thumbnails}/` under `MEDIA_ROOT=/app/media` |
| **O3** | Default directory structure | ✅ | O3(a/b): quoted `find` listing of the three subdirectories + `media.lock` + Whoosh `data/index` |
| **O3** | Default filename pattern | ✅ | O3(c): **`0000001.pdf`** = `{doc.pk:07}` + extension, quoted `ls`; rationale from `generate_filename()` with `PAPERLESS_FILENAME_FORMAT` unset |
| **O3** | Which DB tables receive new rows | ✅ | O3(d): quoted before/after diff — always `documents_document`, `django_admin_log`, `django_q_task`; conditional `documents_document_tags` (observed); others flagged as conditional/not-observed; abstract `MatchingModel` has no table |

**Items explicitly flagged as not verified at runtime:** the unsupported-MIME-type rejection path (a valid PDF was used); `documents_log`, `documents_correspondent`/`documents_tag`/`documents_documenttype`, and `django_q_schedule` did not gain rows in the observed ingestion diffs and are marked conditional. Everything else in this document is grounded in the quoted, command-attributed runtime output above and in this commit's source at `542221a38dff06361e07976452f9aea24d210542`.

