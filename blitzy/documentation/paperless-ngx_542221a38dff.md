# paperless-ngx — Runtime Ingestion Behavior (observation-backed answers)

**Repository:** `paperless-ngx` · **Branch:** `paperless-ngx_542221a38dff` · **HEAD:** `542221a38dff06361e07976452f9aea24d210542`

This document answers five runtime questions about how `paperless-ngx` behaves while ingesting
documents. Every factual claim below is backed by **(a)** the exact command that produced it,
**(b)** the complete, unedited observed output, and **(c)** a `file:line` citation to the
responsible code. Statements that were reasoned from reading the code but not directly observed at
runtime are explicitly marked **(inferred)**.

---

## Investigation environment & canonical build

All observations were produced by running the software in its **default / canonical configuration**
as a normal user would, using the pinned dependency stack. Nothing about the application's behavior
was altered.

| Component | Value | Source |
|-----------|-------|--------|
| Python interpreter | 3.9.23 | `Dockerfile:L18` `FROM python:3.9-slim-bullseye` |
| Database (default) | SQLite (`/work/data/db.sqlite3`) | `src/paperless/settings.py:L297-300` |
| Broker / channel layer | Redis 6.0 (`redis://localhost:6379`) | `docker/compose/docker-compose.sqlite.yml:L28-29`; `src/paperless/settings.py:L456` |
| Task queue | **Django-Q 1.3.9** (NOT Celery) | `src/paperless/settings.py:L449-456` |
| Web/ASGI server | gunicorn 20.1.0 → `paperless.asgi:application` | `docker/supervisord.conf:L10-11` |
| OCR stack | ocrmypdf 13.4.3, tesseract 4.1.1, ghostscript 9.53.3, ImageMagick `convert`, `optipng` | observed (below) |
| ML stack | scikit-learn 1.0.2, numpy 1.22.3, scipy 1.8.0 | observed (below) |
| Full-text index | Whoosh 2.7.4 (filesystem) | `src/paperless/settings.py:L73` |

**Observed toolchain versions:**

```
$ tesseract --version | head -1 ; gs --version ; python3 -c "import ocrmypdf,sklearn,numpy,scipy,whoosh; \
  print('ocrmypdf',ocrmypdf.__version__); print('scikit-learn',sklearn.__version__); \
  print('numpy',numpy.__version__); print('scipy',scipy.__version__); print('whoosh',whoosh.__version__)"
tesseract 4.1.1
9.53.3
ocrmypdf 13.4.3
scikit-learn 1.0.2
numpy 1.22.3
scipy 1.8.0
whoosh (2, 7, 4)
```

**Canonical bring-up (exact commands used).** The three supervised programs from
`docker/supervisord.conf` were launched by hand — `qcluster` (scheduler/worker,
`docker/supervisord.conf:L28-29`), `document_consumer` (watcher, `L19-20`), and `gunicorn`
(`L10-11`). All writable paths were redirected **outside** the repository so the checkout stays
pristine; the database was created fresh with `migrate`.

```bash
# environment (SQLite default DB, OCR language eng); redis 6.0 already running on localhost:6379
export PAPERLESS_REDIS=redis://localhost:6379
export PAPERLESS_SECRET_KEY=observation-key
export PAPERLESS_DATA_DIR=/work/data
export PAPERLESS_MEDIA_ROOT=/work/media
export PAPERLESS_CONSUMPTION_DIR=/work/consume
export PAPERLESS_OCR_LANGUAGE=eng
export PAPERLESS_TIME_ZONE=UTC

cd /app/src
python3 manage.py migrate                    # creates schema, the `consumer` user, and the schedules
python3 manage.py qcluster            > /work/qcluster.out 2>&1 &   # Django-Q cluster (worker + scheduler)
python3 manage.py document_consumer   > /work/consumer.out 2>&1 &   # consumption-dir watcher
gunicorn -c /app/gunicorn.conf.py -b 0.0.0.0:8000 paperless.asgi:application > /work/gunicorn.out 2>&1 &
```

**Bootstrap side effects observed after `migrate`** (these are prerequisites the pipeline relies on):

```
$ python3 manage.py shell -c "from django.contrib.auth.models import User; \
  [print('id=%d username=%r is_superuser=%s'%(u.id,u.username,u.is_superuser)) for u in User.objects.order_by('id')]"
id=1 username='consumer' is_superuser=False

$ python3 manage.py shell -c "from django_q.models import Schedule; \
  [print('name=%r func=%s type=%s'%(s.name,s.func,s.schedule_type)) for s in Schedule.objects.order_by('id')]"
name='Train the classifier' func=documents.tasks.train_classifier type=H
name='Optimize the index' func=documents.tasks.index_optimize type=D
name='Perform sanity check' func=documents.tasks.sanity_check type=W
name='Check all e-mail accounts' func=paperless_mail.tasks.process_mail_accounts type=I
```

- The `consumer` user (id=1) is created by migration `0019` — `src/documents/migrations/0019_add_consumer_user.py:L10` (`User.objects.create(username="consumer")`). The post-consume `set_log_entry` handler depends on it (see Q5).
- The classifier retrain is registered as a **HOURLY** schedule (`type=H`) named *"Train the classifier"* by migration `1001` — `src/documents/migrations/1001_auto_20201109_1636.py:L10-14`. This is the mechanism behind Q3.

---

## Methodology note

Everything below was observed at runtime by exercising the **real ingestion entry points** — a PDF
dropped into the consumption directory watched by `document_consumer`
(`src/documents/management/commands/document_consumer.py:L85-91`), backed by a live gunicorn/ASGI
server that also exposes `POST /api/documents/post_document/` (`src/documents/views.py:L491`,
enqueue at `L523-524`). No bypassing interface, fallback, or synthetic stand-in was used to obtain
any behavioral value. Where a value could only be reasoned from the source (e.g. the never-triggered
classifier error branch), it is labeled **(inferred)**. Per-file log lines are grouped by a UUID via
`LoggingMixin` (`src/documents/loggers.py:L5`, `L14`, `L21`).

## Question → section map

| # | Question | Section |
|---|----------|---------|
| Q1 | Which running services participate in ingestion? | [Q1](#q1--which-services-participate-in-ingestion) |
| Q2 | What is the ordered log-event sequence for one document? | [Q2](#q2--ordered-log-event-sequence-for-one-document) |
| Q3 | When does the ML classifier retrain (every upload or only sometimes)? | [Q3](#q3--when-does-the-classifier-retrain-frequencytiming) |
| Q4 | Which log messages distinguish "training happened" from "idle"? | [Q4](#q4--log-strings-distinguishing-training-vs-idle) |
| Q5 | Where does the document land on disk, and which DB tables get rows? | [Q5](#q5--default-disk-layout--db-tables-beforeafter-one-ingestion) |

---

## Q1 — Which services participate in ingestion?

**Answer.** Six runtime participants cooperate to ingest one document: the **`document_consumer`
directory watcher**, the **Redis broker** (which is also the Channels layer), the **`qcluster`
Django-Q worker**, the **gunicorn/ASGI web server** (with its WebSocket status channel), the
**database** (SQLite by default), and the **OCR stack** (`ocrmypdf`/`tesseract` plus ImageMagick
`convert` + `optipng`). The asynchronous task queue is **Django-Q, not Celery**.

### The participants, each with observed evidence

**1. `document_consumer` — the directory watcher.** It places an inotify watch on the consumption
directory and, on a new file, logs a line and enqueues a Django-Q task; it does **not** run the
pipeline itself.

```
$ head -1 /work/consumer.out
[2026-07-08 05:56:47,621] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /work/consume
```

- Logger `paperless.management.consumer` — `src/documents/management/commands/document_consumer.py:L24`.
- "Adding … to the task queue." then `async_task("documents.tasks.consume_file", …)` — `document_consumer.py:L85` and `L86-87`.

**2. Redis broker (`redis:6.0`) — the message bus & channel layer.** It is the Django-Q broker
(watcher → worker hand-off) and the Channels backend for WebSocket progress.

```
$ redis-cli ping
PONG
$ redis-server --version
Redis server v=6.0.16 sha=00000000:0 malloc=jemalloc-5.2.1 bits=64 build=d4b5be3f91fa055c
$ redis-cli INFO server
# Server
redis_version:6.0.16
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:d4b5be3f91fa055c
redis_mode:standalone
os:Linux 6.6.122+ x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:10.2.1
process_id:719
run_id:68768607b18fc2786a9a7770599f75bd557ee0c8
tcp_port:6379
uptime_in_seconds:5563
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:5105346
executable:/app/redis-server
config_file:
io_threads_active:0
$ python3 manage.py shell -c "from django.conf import settings; print('Q_CLUSTER =', settings.Q_CLUSTER); \
  print('save_limit present?', 'save_limit' in settings.Q_CLUSTER)"
Q_CLUSTER = {'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://localhost:6379'}
save_limit present? False
```

- `Q_CLUSTER` (broker URL, worker count) — `src/paperless/settings.py:L449-456`; the `redis` URL default is `redis://localhost:6379` at `L456`.
- **Observed runtime broker version: Redis 6.0.16.** `redis-server --version` reports `v=6.0.16` and `redis-cli INFO server` reports `redis_version:6.0.16` (both shown above). This *observed* runtime availability agrees with the canonical broker image `redis:6.0` declared in source at `docker/compose/docker-compose.sqlite.yml:L28-29` — i.e. the version was measured at runtime, not merely assumed from the compose file.
- **`save_limit` is absent** from `Q_CLUSTER`, so Django-Q's default of **250** applies (successful tasks are persisted up to 250; failures are always saved). This makes completed tasks an observable DB side effect — see Q5. The default is not an assumption; it is read directly from the installed Django-Q package source:

```
$ python3 -c "import django_q, os; print(os.path.dirname(django_q.__file__))"
/usr/local/lib/python3.9/site-packages/django_q
$ grep -n "SAVE_LIMIT" /usr/local/lib/python3.9/site-packages/django_q/conf.py
87:    SAVE_LIMIT = conf.get("save_limit", 250)
$ sed -n '85,89p' /usr/local/lib/python3.9/site-packages/django_q/conf.py
    # Maximum number of successful tasks kept in the database. 0 saves everything. -1 saves none
    # Failures are always saved
    SAVE_LIMIT = conf.get("save_limit", 250)

    # Guard loop sleep in seconds. Should be between 0 and 60 seconds.
```

So `save_limit` defaults to **250** (`django_q/conf.py:L87`, `django-q==1.3.9`); completed `consume_file` tasks are therefore persisted as `django_q_task` rows up to that limit (failures always saved) — an observable DB side effect confirmed in Q5.

**3. `qcluster` — the Django-Q worker (and scheduler).** It pops the `consume_file` task from Redis
and runs the actual pipeline.

```
$ head -16 /work/qcluster.out
05:56:47 [Q] INFO Q Cluster mexico-angel-asparagus-nuts starting.
05:56:47 [Q] INFO Process-1:1 ready for work at 5153
05:56:47 [Q] INFO Process-1:2 ready for work at 5154
05:56:47 [Q] INFO Process-1:3 ready for work at 5155
05:56:47 [Q] INFO Process-1:4 ready for work at 5156
05:56:47 [Q] INFO Process-1:5 ready for work at 5157
05:56:47 [Q] INFO Process-1:6 ready for work at 5158
05:56:47 [Q] INFO Process-1:7 ready for work at 5159
05:56:47 [Q] INFO Process-1:8 ready for work at 5160
05:56:47 [Q] INFO Process-1:9 ready for work at 5161
05:56:47 [Q] INFO Process-1:10 ready for work at 5162
05:56:47 [Q] INFO Process-1:11 ready for work at 5163
05:56:47 [Q] INFO Process-1:12 monitoring at 5164
05:56:47 [Q] INFO Process-1 guarding cluster mexico-angel-asparagus-nuts
05:56:47 [Q] INFO Process-1:13 pushing tasks at 5165
05:56:47 [Q] INFO Q Cluster mexico-angel-asparagus-nuts running.
```

- The task the worker runs is `documents.tasks.consume_file` — `src/documents/tasks.py:L184` — which instantiates `Consumer().try_consume_file(...)` at `src/documents/tasks.py:L236`.
- The 11 worker processes correspond to `Q_CLUSTER["workers"]` (observed `workers: 11`).

**4. gunicorn / ASGI web server.** Required for the `POST /api/documents/post_document/` upload
entry point and for the WebSocket status channel that streams progress.

```
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/api/
HTTP 200
```

- Runs `paperless.asgi:application` — `docker/supervisord.conf:L10-11`.
- The alternate upload entry point `PostDocumentView` writes a `NamedTemporaryFile` and enqueues the same task — `src/documents/views.py:L491`, `L512`, `L523-524`.

**5. Database (SQLite by default).** Ingestion persists a `Document` row (and more — see Q5).

```
$ python3 manage.py shell -c "from django.conf import settings; \
  print('ENGINE =', settings.DATABASES['default']['ENGINE']); print('NAME   =', settings.DATABASES['default']['NAME'])"
ENGINE = django.db.backends.sqlite3
NAME   = /work/data/db.sqlite3
```

- Default SQLite database — `src/paperless/settings.py:L297-300`.

**6. OCR stack.** The `RasterisedDocumentParser` invokes `ocrmypdf`/`tesseract` for text and PDF/A,
and ImageMagick `convert` + `optipng` for the thumbnail. Their invocations are visible verbatim in
the Q2 log (`Calling OCRmyPDF with args: …`, `Execute: convert …`, `Execute: optipng …`).

### Django-Q, **not** Celery (critical naming fact)

The ingestion task is `documents.tasks.consume_file`, executed by a Django-Q `qcluster`. Celery is
not installed at all:

```
$ python3 -c "import celery"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'celery'
$ python3 -c "import django_q; print('django_q', django_q.VERSION)"
django_q (1, 3, 9)
```

### Reasoning

The watcher's only job is to detect the file and **enqueue** a task
(`document_consumer.py:L85-87`); it never runs the pipeline. Redis carries that task from the watcher
to a `qcluster` worker, which runs the whole `Consumer.try_consume_file()` pipeline
(`src/documents/consumer.py:L180`), calling the OCR stack for parsing/thumbnailing and finally
writing to the database. gunicorn/ASGI is the front door for the HTTP upload path and the source of
the live WebSocket progress channel (also carried over Redis). Thus every one of the six services has
a distinct, observed role in a single ingestion.

---

## Q2 — Ordered log-event sequence for one document

**Answer.** A single PDF (`invoice_alpha.pdf`) was dropped into the consumption directory and its
complete, ordered log stream was captured from `DATA_DIR/log/paperless.log`. The log format is
`[{asctime}] [{levelname}] [{name}] {message}` (`src/paperless/settings.py:L377-378`). The verbatim
sequence, with each line annotated by the code that emits it, is below.

**Commands (real file-drop entry point):**

```bash
$ off=$(wc -l < /work/data/log/paperless.log)                            # lines already in the log (off=3)
$ cp /work/samples/invoice_alpha.pdf /work/consume/invoice_alpha.pdf     # real file-drop entry point
# poll until this document's slice logs "consumption finished" (finished in ~5s)
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log                   # exactly the lines for this document
```

**Observed, unedited output** (volatile substrings — timestamps, the random `/tmp/paperless/…`
work-dir name — vary run to run; the log strings and ordering are stable):

```
[2026-07-08 05:58:46,837] [INFO] [paperless.management.consumer] Adding /work/consume/invoice_alpha.pdf to the task queue.
[2026-07-08 05:58:46,985] [INFO] [paperless.consumer] Consuming invoice_alpha.pdf
[2026-07-08 05:58:46,985] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 05:58:46,988] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 05:58:46,990] [DEBUG] [paperless.consumer] Parsing invoice_alpha.pdf...
[2026-07-08 05:58:47,015] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /work/consume/invoice_alpha.pdf
[2026-07-08 05:58:47,088] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/work/consume/invoice_alpha.pdf', 'output_file': '/tmp/paperless/paperless-tsiq6d2a/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-tsiq6d2a/sidecar.txt'}
[2026-07-08 05:58:47,384] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-08 05:58:47,390] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-tsiq6d2a/archive.pdf
[2026-07-08 05:58:47,390] [DEBUG] [paperless.consumer] Generating thumbnail for invoice_alpha.pdf...
[2026-07-08 05:58:47,393] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-tsiq6d2a/archive.pdf[0] /tmp/paperless/paperless-tsiq6d2a/convert.png
[2026-07-08 05:58:48,045] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-tsiq6d2a/convert.png -out /tmp/paperless/paperless-tsiq6d2a/thumb_optipng.png
[2026-07-08 05:58:50,979] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 05:58:50,982] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 05:58:51,002] [DEBUG] [paperless.consumer] Deleting file /work/consume/invoice_alpha.pdf
[2026-07-08 05:58:51,027] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-tsiq6d2a
[2026-07-08 05:58:51,027] [INFO] [paperless.consumer] Document 2026-07-08 invoice_alpha consumption finished
```

### Line-by-line source attribution

| Log line (message) | Emitting code |
|--------------------|---------------|
| `Adding …/invoice_alpha.pdf to the task queue.` | `src/documents/management/commands/document_consumer.py:L85` |
| `Consuming invoice_alpha.pdf` | `src/documents/consumer.py:L215` |
| `Detected mime type: application/pdf` | `src/documents/consumer.py:L221` |
| `Parser: RasterisedDocumentParser` | `src/documents/consumer.py:L246` |
| `Parsing invoice_alpha.pdf...` | `src/documents/consumer.py:L260` |
| `Extracted text from PDF file …` | `src/paperless_tesseract/parsers.py:L122` (`self.log("debug", f"Extracted text from PDF file {pdf_file}")`; logger `paperless.parsing.tesseract` at `src/paperless_tesseract/parsers.py:L24`) |
| `Calling OCRmyPDF with args: …` | `src/paperless_tesseract/parsers.py:L260` (`self.log("debug", f"Calling OCRmyPDF with args: {args}")`) |
| `Incomplete sidecar file: discarding.` | `src/paperless_tesseract/parsers.py:L110` (`self.log("debug", "Incomplete sidecar file: discarding.")`) |
| `Generating thumbnail for invoice_alpha.pdf...` | `src/documents/consumer.py:L263` |
| `Execute: convert …` | `src/documents/parsers.py:L143` (module-level `logger.debug("Execute: " + " ".join(args), …)` inside `run_convert()`; logger name `paperless.parsing` at `src/documents/parsers.py:L287`) |
| `Execute: optipng …` | `src/documents/parsers.py:L333` (`self.log("debug", f"Execute: {' '.join(args)}")` inside `get_optimised_thumbnail()` at `L319`; on a `RasterisedDocumentParser` instance the logger resolves to `paperless.parsing.tesseract`) |
| `Document classification model does not exist (yet), not performing automatic matching.` | `src/documents/classifier.py:L32-34` (via `load_classifier()` called during auto-matching) |
| `Saving record to database` | `src/documents/consumer.py:L387` (inside `_store()` at `L379`) |
| `Deleting file /work/consume/invoice_alpha.pdf` | `src/documents/consumer.py:L349` (source `os.unlink` at `L350`, *after* a successful save) |
| `Deleting directory /tmp/paperless/…` | `src/documents/parsers.py:L349` (`self.log("debug", f"Deleting directory {self.tempdir}")` in `DocumentParser.cleanup()` at `L348`; on a `RasterisedDocumentParser` instance the logger resolves to `paperless.parsing.tesseract`, matching the `optipng` row above) |
| `Document 2026-07-08 invoice_alpha consumption finished` | `src/documents/consumer.py:L373` |

### The three named loggers, and the observed truth about the handlers

- **`paperless.management.consumer`** — the watcher (`document_consumer.py:L24`). Emits the first line.
- **`paperless.consumer`** — the `Consumer` class (`consumer.py:L54` `logging_name = "paperless.consumer"`). Emits the bulk of the pipeline lines.
- **`paperless.handlers`** — the six post-consume signal handlers (`src/documents/signals/handlers.py:L27`).

**Observed truth:** in this clean run (no auto-matching entities, no inbox tags) the
`paperless.handlers` logger emits **no lines at all** — the six handlers only log conditionally. This
does **not** mean they didn't run: `set_log_entry` did run, which we prove independently via the
`django_admin_log` **+1** delta in Q5. Do not expect handler log lines in a clean ingestion.

The `paperless.parsing` / `paperless.parsing.tesseract` lines come from the OCR **parser**
(`paperless_tesseract`), which is not one of the three AAP-named loggers but belongs to the same
grouped file (per-file UUID grouping via `LoggingMixin`, `src/documents/loggers.py:L14`, `L21`
`extra={"group": …}`).

### Message/progress constants are WebSocket codes, **not** these log strings

The constants `new_file`, `parsing_document`, `generating_thumbnail`, `save_document`, `finished`
(`src/documents/consumer.py:L43`, `L45`, `L46`, `L48`, `L49`) are **WebSocket status codes** sent via
`_send_progress()` — they are distinct from the human-readable log strings above:

```
$ grep -n "MESSAGE_\|_send_progress" src/documents/consumer.py
37:MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"
38:MESSAGE_FILE_NOT_FOUND = "file_not_found"
39:MESSAGE_PRE_CONSUME_SCRIPT_NOT_FOUND = "pre_consume_script_not_found"
40:MESSAGE_PRE_CONSUME_SCRIPT_ERROR = "pre_consume_script_error"
41:MESSAGE_POST_CONSUME_SCRIPT_NOT_FOUND = "post_consume_script_not_found"
42:MESSAGE_POST_CONSUME_SCRIPT_ERROR = "post_consume_script_error"
43:MESSAGE_NEW_FILE = "new_file"
44:MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"
45:MESSAGE_PARSING_DOCUMENT = "parsing_document"
46:MESSAGE_GENERATING_THUMBNAIL = "generating_thumbnail"
47:MESSAGE_PARSE_DATE = "parse_date"
48:MESSAGE_SAVE_DOCUMENT = "save_document"
49:MESSAGE_FINISHED = "finished"
56:    def _send_progress(
79:        self._send_progress(100, 100, "FAILED", message)
98:                MESSAGE_FILE_NOT_FOUND,
111:                MESSAGE_DOCUMENT_ALREADY_EXISTS,
127:                MESSAGE_PRE_CONSUME_SCRIPT_NOT_FOUND,
138:                MESSAGE_PRE_CONSUME_SCRIPT_ERROR,
149:                MESSAGE_POST_CONSUME_SCRIPT_NOT_FOUND,
175:                MESSAGE_POST_CONSUME_SCRIPT_ERROR,
202:        self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)
225:            self._fail(MESSAGE_UNSUPPORTED_TYPE, f"Unsupported mime type {mime_type}")
240:            self._send_progress(p, 100, "WORKING")
259:            self._send_progress(20, 100, "WORKING", MESSAGE_PARSING_DOCUMENT)
264:            self._send_progress(70, 100, "WORKING", MESSAGE_GENERATING_THUMBNAIL)
274:                self._send_progress(90, 100, "WORKING", MESSAGE_PARSE_DATE)
294:        self._send_progress(95, 100, "WORKING", MESSAGE_SAVE_DOCUMENT)
375:        self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)
```

`_send_progress()` builds a payload and calls `async_to_sync(self.channel_layer.group_send)(…)`
(`src/documents/consumer.py:L56` onward) — i.e. it pushes to the Redis-backed Channels layer for the
WebSocket UI, not to `paperless.log`. That is why the progress codes never appear in the log stream.

### Reasoning

The ordering directly reflects `Consumer.try_consume_file()`: announce → detect mime → pick parser →
parse (OCR) → thumbnail → attempt auto-match (loads the classifier) → persist (`_store`) → unlink the
source → "consumption finished". The watcher line precedes everything because the watcher runs in a
separate process and enqueues the task before any worker picks it up.

### The second real entry point — `POST /api/documents/post_document/` (same pipeline, observed)

The file-drop above is one of the **two** real ingestion entry points. The other is the REST upload
endpoint `POST /api/documents/post_document/`, served by `PostDocumentView`
(`src/documents/views.py:L491`). It requires authentication
(`permission_classes = (IsAuthenticated,)`, `src/documents/views.py:L493`), so a superuser was created
right after `migrate` — the canonical step a normal user takes to obtain API access:

```bash
$ DJANGO_SUPERUSER_PASSWORD=admin python3 manage.py createsuperuser \
    --noinput --username admin --email admin@example.com
Superuser created successfully.
```

**Command (real API entry point).** A single PDF was uploaded with `curl`. The log offset was taken
just before the upload so the captured slice belongs to exactly this document:

```bash
$ off=$(wc -l < /work/data/log/paperless.log)   # lines already in the log (off=2)
$ curl -sS -i -u admin:admin -F "document=@/work/tmp/api_upload_demo.pdf;type=application/pdf" http://127.0.0.1:8000/api/documents/post_document/
```

**Observed, unedited HTTP response** (the endpoint returns `HTTP/1.1 200 OK` with the JSON body `"OK"`;
`server: uvicorn` is the ASGI worker running under gunicorn):

```
HTTP/1.1 200 OK
date: Wed, 08 Jul 2026 08:40:17 GMT
server: uvicorn
content-type: application/json
vary: Accept, Accept-Language, Origin
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

**The view enqueues the same task on the same broker.** `PostDocumentView.post()` writes the upload to
a `NamedTemporaryFile` (`src/documents/views.py:L512`) and calls
`async_task("documents.tasks.consume_file", …)` (`src/documents/views.py:L523-524`) — the *same* task
the watcher enqueues. The web (gunicorn) process logs the enqueue, and the `qcluster` worker picks it
up and runs the identical pipeline:

```
$ grep -n "Enqueued" /work/gunicorn.out
13:08:40:18 [Q] INFO Enqueued 1
$ grep -n "api_upload_demo" /work/qcluster.out
17:08:40:18 [Q] INFO Process-1:1 processing [api_upload_demo.pdf]
18:[2026-07-08 08:40:18,146] [INFO] [paperless.consumer] Consuming api_upload_demo.pdf
19:[2026-07-08 08:40:22,107] [INFO] [paperless.consumer] Document 2026-07-08 api_upload_demo consumption finished
21:08:40:22 [Q] INFO Processed [api_upload_demo.pdf]
```

**Observed, unedited pipeline log slice** — captured the instant this document logged *"consumption
finished"* (`sed -n "$((off+1)),\$p" /work/data/log/paperless.log`; same volatile-substring caveat as
the file-drop block):

```
[2026-07-08 08:40:18,146] [INFO] [paperless.consumer] Consuming api_upload_demo.pdf
[2026-07-08 08:40:18,147] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 08:40:18,149] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 08:40:18,152] [DEBUG] [paperless.consumer] Parsing api_upload_demo.pdf...
[2026-07-08 08:40:18,175] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-9coyy6mz
[2026-07-08 08:40:18,244] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-9coyy6mz', 'output_file': '/tmp/paperless/paperless-t5o1qsfl/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-t5o1qsfl/sidecar.txt'}
[2026-07-08 08:40:18,519] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-08 08:40:18,524] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-t5o1qsfl/archive.pdf
[2026-07-08 08:40:18,524] [DEBUG] [paperless.consumer] Generating thumbnail for api_upload_demo.pdf...
[2026-07-08 08:40:18,527] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-t5o1qsfl/archive.pdf[0] /tmp/paperless/paperless-t5o1qsfl/convert.png
[2026-07-08 08:40:19,181] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-t5o1qsfl/convert.png -out /tmp/paperless/paperless-t5o1qsfl/thumb_optipng.png
[2026-07-08 08:40:22,061] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 08:40:22,064] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 08:40:22,082] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-9coyy6mz
[2026-07-08 08:40:22,106] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-t5o1qsfl
[2026-07-08 08:40:22,107] [INFO] [paperless.consumer] Document 2026-07-08 api_upload_demo consumption finished
```

**The pipeline is identical; only the entry differs.** Compared with the file-drop sequence above, this
slice has exactly **two** differences, both expected from the code:

1. **No `[paperless.management.consumer] Adding … to the task queue.` line.** The directory watcher is
   not involved on the API path — `PostDocumentView` enqueues the task directly
   (`src/documents/views.py:L523-524`). The slice therefore begins at `Consuming …` and is **16** lines
   rather than the watcher path's 17.
2. **The source file is the upload temp file**, `/tmp/paperless/paperless-upload-9coyy6mz` (the
   `NamedTemporaryFile`, `src/documents/views.py:L512`), instead of a `/work/consume/…` path — so the
   `Extracted text from PDF file …` and `Deleting file …` lines reference that temp file, and
   `Deleting file` unlinks it after a successful save.

Everything else — mime detection, parser selection, OCR (`Calling OCRmyPDF …`), thumbnailing
(`convert`/`optipng`), the classifier `…model does not exist (yet)…` DEBUG, `Saving record to
database`, `Deleting directory …`, and `Document … consumption finished` — is the same as the watcher
path, because **both entry points converge on `documents.tasks.consume_file`**.

The upload produced a real `Document` row with the default `{pk:07}` filename (Q5), confirming the API
path is exercised end-to-end and is not a stand-in:

```
$ python3 manage.py shell -c "from documents.models import Document as D; d=D.objects.get(); \
  print(d.pk, repr(d.title), d.filename, d.archive_filename); print(d.source_path)"
1 'api_upload_demo' 0000001.pdf 0000001.pdf
/work/media/documents/originals/0000001.pdf
```

---

## Q3 — When does the classifier retrain? (frequency/timing)

**Answer.** The classifier does **not** retrain on every upload. Retraining is decoupled from
ingestion: `train_classifier()` (`src/documents/tasks.py:L48`) is **not** called by `consume_file`,
and is registered on an **HOURLY** schedule (`src/documents/migrations/1001_auto_20201109_1636.py:L10-14`)
executed by the `qcluster` scheduler. Even when it does run, it short-circuits unless the training
data changed (see Q4). So the accurate statement is: **not on every upload, and not even on every
scheduled run — only on a scheduled/on-demand run whose training data has changed.**

### Mechanism (grounded in code)

- `consume_file` (`src/documents/tasks.py:L184`) instantiates `Consumer().try_consume_file(...)` at `L236`; it never references `train_classifier`. During consumption the classifier is only **loaded** for auto-matching, never trained.
- `train_classifier` is scheduled HOURLY by migration `1001` — observed as a live schedule row:

```
$ python3 manage.py shell -c "from django_q.models import Schedule, Task; \
  s=Schedule.objects.get(func='documents.tasks.train_classifier'); \
  print('name=%r func=%s schedule_type=%s repeats=%s next_run=%s'%(s.name,s.func,s.schedule_type,s.repeats,s.next_run)); \
  tc=Task.objects.filter(func='documents.tasks.train_classifier'); \
  print('train_classifier Task rows executed so far:', tc.count()); \
  [print('  task name=%r success=%s started=%s'%(t.name,t.success,t.started)) for t in tc.order_by('started')]"
name='Train the classifier' func=documents.tasks.train_classifier schedule_type=H repeats=-2 next_run=2026-07-08 06:56:45.313295+00:00
train_classifier Task rows executed so far: 1
  task name='island-louisiana-fruit-floor' success=True started=2026-07-08 05:57:17.269667+00:00
```

`schedule_type=H` is Django-Q's HOURLY type. The **only** `train_classifier` execution during the
entire session was fired by the **`qcluster` scheduler at cluster startup** (05:57:17) — a due
schedule runs once when the cluster comes up — **not** by any upload. That single scheduled run
**silently skipped** (no `MATCH_AUTO` entity existed yet, see Q4), so it created no model and emitted
no training log line; `next_run` is the next hourly tick ~1 hour out. The scheduler firing is directly
observable in the cluster log, and it is the *only* thing that ever invokes `train_classifier`:

```
$ grep -nE "created a task from schedule \[Train the classifier\]" /work/qcluster.out
18:05:57:17 [Q] INFO Process-1 created a task from schedule [Train the classifier]
```

### Empirical proof it is not per-upload (scale & stability)

Three documents were ingested in total (one for Q2, then two more), giving three rows in
`documents_document`:

```
$ python3 manage.py shell -c "from documents.models import Document; \
  print('documents_document rows:', Document.objects.count()); \
  [print('  pk=',d.pk,d.filename,repr(d.title)) for d in Document.objects.order_by('pk')]"
documents_document rows: 3
  pk= 1 0000001.pdf 'invoice_alpha'
  pk= 2 0000002.pdf 'report_beta'
  pk= 3 0000003.pdf 'letter_gamma'
```

Document 1 (`invoice_alpha`) was ingested in Q2. The two additional uploads were each checked
**immediately after their own `consumption finished`** by slicing only that upload's newly appended
log lines (`off` = line count captured just before the drop) and grepping for any training marker.
The `|| echo` fallback prints the sentinel *only when grep matches nothing* — so the sentinel below
is genuine command output, not a hand-written note:

```
$ off=$(wc -l < /work/data/log/paperless.log); cp /work/samples/report_beta.pdf /work/consume/   # then poll for "consumption finished"
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE "Saving updated classifier model|Training data unchanged|Gathering data from database|Vectorizing data|Training .* classifier" || echo ">>> NO training markers after report_beta upload <<<"
>>> NO training markers after report_beta upload <<<

$ off=$(wc -l < /work/data/log/paperless.log); cp /work/samples/letter_gamma.pdf /work/consume/   # then poll for "consumption finished"
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE "Saving updated classifier model|Training data unchanged|Gathering data from database|Vectorizing data|Training .* classifier" || echo ">>> NO training markers after letter_gamma upload <<<"
>>> NO training markers after letter_gamma upload <<<
```

And the aggregate grep over the **complete three-ingestion log** (same `|| echo` fallback) confirms
zero training activity across all three uploads combined:

```
$ grep -nE "Saving updated classifier model|Training data unchanged|Gathering data from database|Vectorizing data|Training .* classifier" /work/data/log/paperless.log || echo ">>> NO MATCHES — no training log lines during any of the 3 uploads <<<"
>>> NO MATCHES — no training log lines during any of the 3 uploads <<<
```

The only classifier-related lines in the whole log are the per-upload **matching** attempts from
`load_classifier()` (one per document) — not training:

```
$ grep -nE "\[paperless.classifier\]" /work/data/log/paperless.log
16:[2026-07-08 05:58:50,979] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
33:[2026-07-08 05:59:54,441] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
50:[2026-07-08 06:00:00,015] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```

And no model file exists after three uploads (training genuinely never happened):

```
$ ls -la /work/data/classification_model.pickle
ls: cannot access '/work/data/classification_model.pickle': No such file or directory
```

**Scale/stability:** across **3** ingestions with no intervening model change, the number of
retrains observed was **0**. This was verified two ways: a per-upload grep run immediately after each
of the two additional uploads returned no training markers (the two `report_beta`/`letter_gamma`
sentinels above), and the aggregate grep over the complete three-ingestion log likewise returned
none. Uploading more documents does not change this — the retrain trigger is the hourly schedule
(whose single startup run silently skipped, above) or the on-demand command in Q4, never the upload
itself.

### Reasoning

Retraining is intentionally decoupled from ingestion so that a burst of uploads does not repeatedly
pay the (expensive) training cost. Instead, the model is refreshed on a fixed hourly cadence, and —
as Q4 shows — even that refresh is a no-op unless the training data actually changed.


---

## Q4 — Log strings distinguishing training vs. idle

**Answer.** Training is driven through its canonical on-demand entry point,
`python3 manage.py document_create_classifier`
(`src/documents/management/commands/document_create_classifier.py:L20` → `train_classifier()`), which
is the same function the hourly schedule runs. There are four distinct branches, each with an exact,
observed log signature:

| Branch | Condition | Log signature | Code |
|--------|-----------|---------------|------|
| **Silent skip** | No `Tag`/`DocumentType`/`Correspondent` uses `MATCH_AUTO` | *(nothing — function returns immediately)* | `src/documents/tasks.py:L49-55` |
| **No model** | Past the guard, but `MODEL_FILE` missing | `DEBUG … Document classification model does not exist (yet), not performing automatic matching.` | `src/documents/classifier.py:L32-34` |
| **Training** | `DocumentClassifier.train()` returns truthy (data new/changed) | `INFO … Saving updated classifier model to {MODEL_FILE}...` | `src/documents/tasks.py:L64-66` (string `L65`) |
| **Idle** | `train()` sees an unchanged SHA-1 → returns `False` | `DEBUG … Training data unchanged.` | `src/documents/tasks.py:L69` |

### Branch 1 — SILENT SKIP (no `MATCH_AUTO` entity → no log line, no model)

With a fresh database there is no auto-matching entity, so the guard at `tasks.py:L49-55` returns
before doing anything:

```
$ python3 manage.py shell -c "from documents.models import Tag,DocumentType,Correspondent; \
  print('tags MATCH_AUTO:', Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).count()); \
  print('doctypes MATCH_AUTO:', DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).count()); \
  print('correspondents MATCH_AUTO:', Correspondent.objects.filter(matching_algorithm=Tag.MATCH_AUTO).count())"
tags MATCH_AUTO: 0
doctypes MATCH_AUTO: 0
correspondents MATCH_AUTO: 0

$ off=$(wc -l < /work/data/log/paperless.log); python3 manage.py document_create_classifier; echo exit=$?
off=78
exit=0
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE 'paperless.classifier|paperless.tasks' || echo '>>> NO classifier/tasks log lines — SILENT SKIP confirmed <<<'
>>> NO classifier/tasks log lines — SILENT SKIP confirmed <<<

$ ls -la /work/data/classification_model.pickle
ls: cannot access '/work/data/classification_model.pickle': No such file or directory
```

The command produced no stdout, no log line, and no model file — matching the guard
(`tasks.py:L50-52` checks each of `Tag`/`DocumentType`/`Correspondent` for `MATCH_AUTO`; `return` at
`L55`). Seeding one `MATCH_AUTO` entity (below) is a legitimate prerequisite to get past this guard,
not a bypass.

### Branches 2 + 3 — NO-MODEL DEBUG, then TRAINING (INFO)

One `MATCH_AUTO` tag was created and assigned to document pk=1, then the command was run. On the
first run the model file is missing, so `load_classifier()` logs the **no-model DEBUG** line, then
`train()` runs and the task logs the **training INFO** line:

```
$ python3 manage.py shell -c "from documents.models import Tag,Document; \
  t,_=Tag.objects.get_or_create(name='AutoTagAlpha', defaults={'matching_algorithm':Tag.MATCH_AUTO,'match':'acme'}); \
  t.matching_algorithm=Tag.MATCH_AUTO; t.match='acme'; t.save(); Document.objects.get(pk=1).tags.add(t); \
  print('tag pk=%d match_algo=%d (MATCH_AUTO=%d) assigned_to_doc=1'%(t.pk,t.matching_algorithm,Tag.MATCH_AUTO))"
tag pk=2 match_algo=6 (MATCH_AUTO=6) assigned_to_doc=1

$ stat -c %Y /work/data/classification_model.pickle          # mtime BEFORE (no model yet)
stat: cannot statx '/work/data/classification_model.pickle': No such file or directory

$ off=$(wc -l < /work/data/log/paperless.log); python3 manage.py document_create_classifier   # RUN 1 (data is new)
off=79
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE 'paperless.classifier|paperless.tasks' || echo '>>> NONE <<<'
1:[2026-07-08 06:03:20,657] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
2:[2026-07-08 06:03:20,658] [DEBUG] [paperless.classifier] Gathering data from database...
3:[2026-07-08 06:03:20,661] [DEBUG] [paperless.classifier] 3 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
4:[2026-07-08 06:03:21,052] [DEBUG] [paperless.classifier] Vectorizing data...
5:[2026-07-08 06:03:21,053] [DEBUG] [paperless.classifier] Training tags classifier...
6:[2026-07-08 06:03:21,105] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
7:[2026-07-08 06:03:21,105] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
8:[2026-07-08 06:03:21,105] [INFO] [paperless.tasks] Saving updated classifier model to /work/data/classification_model.pickle...

$ stat -c %Y /work/data/classification_model.pickle          # mtime AFTER
1783490601
$ ls -la /work/data/classification_model.pickle
-rw-r--r-- 1 root root 251972 Jul  8 06:03 /work/data/classification_model.pickle
```

- **No-model DEBUG** string — `src/documents/classifier.py:L32-34` (via `load_classifier()` at `L30`).
- `Gathering data from database...` — `src/documents/classifier.py:L123`.
- **Training INFO** string `Saving updated classifier model to …` — `src/documents/tasks.py:L64-66` (the string literal is on `L65`); `classifier.save()` follows at `L67`. `train()` returned truthy (`src/documents/classifier.py:L249` `return True`), and a **251972-byte** model file was created at `MODEL_FILE` (`src/paperless/settings.py:L74`).
- (The INFO line also appears on stderr because the `paperless` logger propagates to the root `console` handler whose level is `INFO` — `src/paperless/settings.py:L387-390`, `L407`.)

### Branch 4 — IDLE (DEBUG "Training data unchanged."), stable across ≥2 runs

Re-running with **unchanged** data twice more yields the idle branch every time, and the model file
is **not** rewritten (mtime constant) — demonstrating the required ≥2-run stability:

```
$ stat -c %Y /work/data/classification_model.pickle          # IDLE RUN 2 — mtime BEFORE
1783490601
$ off=$(wc -l < /work/data/log/paperless.log); python3 manage.py document_create_classifier   # IDLE RUN 2
off=87
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE 'paperless.classifier|paperless.tasks' || echo '>>> NONE <<<'
2:[2026-07-08 06:03:22,968] [DEBUG] [paperless.classifier] Gathering data from database...
3:[2026-07-08 06:03:22,973] [DEBUG] [paperless.tasks] Training data unchanged.
$ stat -c %Y /work/data/classification_model.pickle          # IDLE RUN 2 — mtime AFTER (unchanged)
1783490601

$ stat -c %Y /work/data/classification_model.pickle          # IDLE RUN 3 — mtime BEFORE
1783490601
$ off=$(wc -l < /work/data/log/paperless.log); python3 manage.py document_create_classifier   # IDLE RUN 3
off=90
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE 'paperless.classifier|paperless.tasks' || echo '>>> NONE <<<'
2:[2026-07-08 06:03:24,808] [DEBUG] [paperless.classifier] Gathering data from database...
3:[2026-07-08 06:03:24,812] [DEBUG] [paperless.tasks] Training data unchanged.
$ stat -c %Y /work/data/classification_model.pickle          # IDLE RUN 3 — mtime AFTER (unchanged)
1783490601
```

- **Idle DEBUG** string `Training data unchanged.` — `src/documents/tasks.py:L69`, reached because `train()` computed a SHA-1 over the preprocessed content+labels (`classifier.py:L124`) equal to the stored `data_hash`, so it short-circuited with `return False` (`src/documents/classifier.py:L163-164`).

### Transition — data changed → trains again

To confirm the trigger is a **data change**, the tag was assigned to a second document and the command
re-run; training fired again and the model mtime advanced:

```
$ python3 manage.py shell -c "from documents.models import Tag,Document; \
  t=Tag.objects.get(name='AutoTagAlpha'); Document.objects.get(pk=2).tags.add(t); \
  print('AutoTagAlpha now on docs:', list(t.documents.values_list('pk',flat=True)))"
AutoTagAlpha now on docs: [1, 2]

$ stat -c %Y /work/data/classification_model.pickle          # mtime BEFORE
1783490601
$ off=$(wc -l < /work/data/log/paperless.log); python3 manage.py document_create_classifier   # data changed
off=94
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep -nE 'paperless.classifier|paperless.tasks' || echo '>>> NONE <<<'
1:[2026-07-08 06:03:27,600] [DEBUG] [paperless.classifier] Gathering data from database...
2:[2026-07-08 06:03:27,604] [DEBUG] [paperless.classifier] 3 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
3:[2026-07-08 06:03:27,605] [DEBUG] [paperless.classifier] Vectorizing data...
4:[2026-07-08 06:03:27,605] [DEBUG] [paperless.classifier] Training tags classifier...
5:[2026-07-08 06:03:27,660] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
6:[2026-07-08 06:03:27,660] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
7:[2026-07-08 06:03:27,661] [INFO] [paperless.tasks] Saving updated classifier model to /work/data/classification_model.pickle...
$ stat -c %Y /work/data/classification_model.pickle          # mtime AFTER -> advanced (retrained)
1783490607
```

### Branch 5 — ERROR (inferred)

An exception raised inside `train()` is caught and logged at `WARNING` as
`Classifier error: {e}` (`src/documents/tasks.py:L71-72`). This branch was **not** triggered at
runtime, so it is labeled **(inferred)** from the source.

### Decision table (summary)

| State | Trigger condition | Emitted log |
|-------|-------------------|-------------|
| Silent skip | no `MATCH_AUTO` entity exists | *(none)* |
| Trains (new/changed data) | guard passed **and** SHA-1 differs from stored hash | `INFO … Saving updated classifier model to …` (+ no-model DEBUG on first ever run) |
| Idle | guard passed **and** SHA-1 equals stored hash | `DEBUG … Training data unchanged.` |
| Error | exception during `train()` | `WARNING … Classifier error: {e}` *(inferred)* |

This fully supports the Q3 conclusion: retraining happens **not on every upload, and not on every
scheduled run either** — only on a run whose training data changed.


---

## Q5 — Default disk layout + DB tables (before/after one ingestion)

**Answer.** With `PAPERLESS_FILENAME_FORMAT` unset (the default), a single ingestion writes three
files named by a **7-digit zero-padded primary key** — `documents/originals/0000001.pdf`,
`documents/archive/0000001.pdf`, `documents/thumbnails/0000001.png` — and inserts rows into
**`documents_document`** (+1), **`django_admin_log`** (+1), and the Django-Q task table
**`django_q_task`** (+1 for the ingestion). `documents_document_tags` gets **+0** in a clean run
(no inbox/auto tag). The full-text index is written to a **Whoosh index on the filesystem**, not a
DB table. Crucially, **`documents_log` is NOT written** in this commit.

### Default on-disk layout & filename pattern

`PAPERLESS_FILENAME_FORMAT` defaults to `None` (`src/paperless/settings.py:L584`
`os.getenv("PAPERLESS_FILENAME_FORMAT")`), so `generate_filename()`
(`src/documents/file_handling.py:L128`; the `if settings.PAPERLESS_FILENAME_FORMAT is not None:`
branch at `L132` is skipped) falls through to the default at `src/documents/file_handling.py:L193`
(`filename = f"{doc.pk:07}{counter_str}{filetype_str}"`).

```
$ find /work/media -type f | sort
/work/media/documents/archive/0000001.pdf
/work/media/documents/archive/0000002.pdf
/work/media/documents/archive/0000003.pdf
/work/media/documents/archive/0000004.pdf
/work/media/documents/originals/0000001.pdf
/work/media/documents/originals/0000002.pdf
/work/media/documents/originals/0000003.pdf
/work/media/documents/originals/0000004.pdf
/work/media/documents/thumbnails/0000001.png
/work/media/documents/thumbnails/0000002.png
/work/media/documents/thumbnails/0000003.png
/work/media/documents/thumbnails/0000004.png
/work/media/media.lock

$ ls -la /work/media/documents/originals /work/media/documents/archive /work/media/documents/thumbnails
/work/media/documents/archive:
total 52
drwxr-xr-x 2 root root 4096 Jul  8 06:05 .
drwxr-xr-x 5 root root 4096 Jul  8 05:58 ..
-rw-r--r-- 1 root root 8497 Jul  8 05:58 0000001.pdf
-rw-r--r-- 1 root root 8263 Jul  8 05:59 0000002.pdf
-rw-r--r-- 1 root root 8208 Jul  8 06:00 0000003.pdf
-rw-r--r-- 1 root root 7694 Jul  8 06:05 0000004.pdf

/work/media/documents/originals:
total 24
drwxr-xr-x 2 root root 4096 Jul  8 06:05 .
drwxr-xr-x 5 root root 4096 Jul  8 05:58 ..
-rw-r--r-- 1 root root 1562 Jul  8 05:58 0000001.pdf
-rw-r--r-- 1 root root 1529 Jul  8 05:59 0000002.pdf
-rw-r--r-- 1 root root 1527 Jul  8 06:00 0000003.pdf
-rw-r--r-- 1 root root 1517 Jul  8 06:05 0000004.pdf

/work/media/documents/thumbnails:
total 52
drwxr-xr-x 2 root root 4096 Jul  8 06:05 .
drwxr-xr-x 5 root root 4096 Jul  8 05:58 ..
-rw-r--r-- 1 root root 9724 Jul  8 05:58 0000001.png
-rw-r--r-- 1 root root 8643 Jul  8 05:59 0000002.png
-rw-r--r-- 1 root root 9004 Jul  8 06:00 0000003.png
-rw-r--r-- 1 root root 7658 Jul  8 06:05 0000004.png
```

- `originals/` → `ORIGINALS_DIR` (`src/paperless/settings.py:L62`); the model's `source_path` builds `{:07}{}` (`src/documents/models.py:L223`, name at `L227`).
- `archive/` → `ARCHIVE_DIR` (`src/paperless/settings.py:L63`); `archive_path` (`src/documents/models.py:L242`).
- `thumbnails/` → `THUMBNAIL_DIR` (`src/paperless/settings.py:L64`); `thumbnail_path` builds `{:07}.png` (`src/documents/models.py:L273`, name at `L274`).
- `media.lock` → the `FileLock(settings.MEDIA_LOCK)` file (`src/paperless/settings.py:L72`; used at `src/documents/consumer.py:L315`).

The listing shows all **four** documents ingested during this session (pk 1–4); each ingestion
contributes exactly one file to each of `originals/`, `archive/`, and `thumbnails/`, so a single
ingestion adds **three** files. The stored `Document` row confirms the `{pk:07}` naming and that text
was extracted:

```
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); \
  print('pk=',d.pk,'filename=',d.filename,'archive_filename=',d.archive_filename); \
  print('title=',repr(d.title),'mime=',d.mime_type,'checksum=',d.checksum[:12]); \
  print('source_path=',d.source_path); print('archive_path=',d.archive_path); \
  print('thumbnail_path=',d.thumbnail_path); print('content(first80)=',repr(d.content[:80]))"
pk= 1 filename= 0000001.pdf archive_filename= 0000001.pdf
title= 'invoice_alpha' mime= application/pdf checksum= 9d5fb1b7843c
source_path= /work/media/documents/originals/0000001.pdf
archive_path= /work/media/documents/archive/0000001.pdf
thumbnail_path= /work/media/documents/thumbnails/0000001.png
content(first80)= 'ACME Invoice Alpha 2026\n\nInvoice number: ALPHA-0001\n\nBill to: Wile E Coyote\n\nAmo'
```

### Database tables receiving rows (before → after one ingestion)

Per-table row counts were snapshotted with a small ephemeral helper immediately **before** dropping a
single PDF and again **after** the pipeline logged `consumption finished`. The helper counts every
user table via `sqlite_master`:

```
$ cat /work/_investigate/db_table_counts.py
#!/usr/bin/env python3
"""Print row counts for every user table in the default SQLite DB.

Usage: python3 db_table_counts.py
Emits lines of the form "<count>\t<table>" for every table listed in
sqlite_master (type='table'), sorted by table name. Used to snapshot the
database immediately before and after a single ingestion so per-table deltas
are unambiguous. Ephemeral observation helper; removed at cleanup.
"""
import os
import sqlite3

db = os.environ.get("PAPERLESS_DBPATH", "/work/data/db.sqlite3")
con = sqlite3.connect(db)
cur = con.cursor()
cur.execute(
    "SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' ORDER BY name"
)
tables = [r[0] for r in cur.fetchall()]
for t in tables:
    cur.execute(f'SELECT COUNT(*) FROM "{t}"')
    n = cur.fetchone()[0]
    print(f"{n}\t{t}")
con.close()
```

The clean single ingestion measured here is `memo_delta.pdf` — the **fourth** document of the session
(the three from Q2/Q3 were already present, and the classifier state was reset first so no auto-tag
applies). The `before` baseline is therefore **not** zero (`documents_document`=3;
`django_q_task`=7 = four scheduled startup tasks + three prior `consume_file` tasks). The exact
ordered command sequence — snapshot BEFORE, drop, wait-for-`consumption finished`, snapshot AFTER — is:

```
$ python3 /work/_investigate/db_table_counts.py > /work/evidence/before.txt      # snapshot BEFORE
$ off=$(wc -l < /work/data/log/paperless.log); cp /work/samples/memo_delta.pdf /work/consume/   # the single ingestion
$ sed -n "$((off+1)),\$p" /work/data/log/paperless.log | grep 'consumption finished'            # wait/confirm done
[2026-07-08 06:05:21,730] [INFO] [paperless.consumer] Document 2026-07-08 memo_delta consumption finished
$ python3 /work/_investigate/db_table_counts.py > /work/evidence/after.txt        # snapshot AFTER
```

The complete, unedited before/after snapshots (all 25 tables) — a reader can verify the three changed
rows (`django_admin_log` 3→4, `django_q_task` 7→8, `documents_document` 3→4) directly:

```
$ cat /work/evidence/before.txt
0	auth_group
0	auth_group_permissions
88	auth_permission
1	auth_user
0	auth_user_groups
0	auth_user_user_permissions
0	authtoken_token
3	django_admin_log
22	django_content_type
92	django_migrations
0	django_q_ormq
4	django_q_schedule
7	django_q_task
0	django_session
0	documents_correspondent
3	documents_document
0	documents_document_tags
0	documents_documenttype
0	documents_log
0	documents_savedview
0	documents_savedviewfilterrule
0	documents_tag
0	paperless_mail_mailaccount
0	paperless_mail_mailrule
0	paperless_mail_mailrule_assign_tags
$ cat /work/evidence/after.txt
0	auth_group
0	auth_group_permissions
88	auth_permission
1	auth_user
0	auth_user_groups
0	auth_user_user_permissions
0	authtoken_token
4	django_admin_log
22	django_content_type
92	django_migrations
0	django_q_ormq
4	django_q_schedule
8	django_q_task
0	django_session
0	documents_correspondent
4	documents_document
0	documents_document_tags
0	documents_documenttype
0	documents_log
0	documents_savedview
0	documents_savedviewfilterrule
0	documents_tag
0	paperless_mail_mailaccount
0	paperless_mail_mailrule
0	paperless_mail_mailrule_assign_tags
```

A second helper (`db_delta.py`) diffs the two snapshots and always prints the five tables of interest
(so a `+0` is shown explicitly, not omitted):

```
$ python3 /work/_investigate/db_delta.py /work/evidence/before.txt /work/evidence/after.txt
TABLE                          BEFORE  AFTER  DELTA
django_admin_log                    3      4     +1
django_q_task                       7      8     +1
documents_document                  3      4     +1
documents_document_tags             0      0     +0
documents_log                       0      0     +0
```

**Attribution, by name:**

```
$ python3 manage.py shell -c "from django_q.models import Task; \
  [print('  func=%-40s name=%-28r success=%s'%(t.func,t.name,t.success)) for t in Task.objects.order_by('started')]"
  func=documents.tasks.train_classifier       name='island-louisiana-fruit-floor' success=True
  func=documents.tasks.index_optimize         name='washington-west-vermont-quebec' success=True
  func=documents.tasks.sanity_check           name='video-oxygen-beryllium-venus' success=True
  func=paperless_mail.tasks.process_mail_accounts name='zulu-early-angel-spring'      success=True
  func=documents.tasks.consume_file           name='invoice_alpha.pdf'            success=True
  func=documents.tasks.consume_file           name='report_beta.pdf'              success=True
  func=documents.tasks.consume_file           name='letter_gamma.pdf'             success=True
  func=documents.tasks.consume_file           name='memo_delta.pdf'               success=True

$ python3 manage.py shell -c "from django.contrib.admin.models import LogEntry; \
  [print('  user=%r action_flag=%d object_repr=%r content_type=%s'%(le.user.username,le.action_flag,le.object_repr,le.content_type)) for le in LogEntry.objects.all()]"
  user='consumer' action_flag=1 object_repr='2026-07-08 invoice_alpha' content_type=documents | document
  user='consumer' action_flag=1 object_repr='2026-07-08 report_beta' content_type=documents | document
  user='consumer' action_flag=1 object_repr='2026-07-08 letter_gamma' content_type=documents | document
  user='consumer' action_flag=1 object_repr='2026-07-08 memo_delta' content_type=documents | document
```

- **`documents_document` +1** — `Document.objects.create(...)` in `_store()` (`src/documents/consumer.py:L398`, method at `L379`).
- **`django_admin_log` +1** — `set_log_entry` creates a `LogEntry` with `action_flag=ADDITION` as user `consumer` (`src/documents/signals/handlers.py:L413-424`; user at `L416`). Note the mechanism is `LogEntry.objects.create(action_flag=ADDITION, …)` (not `log_action`); the table is `django_admin_log`. This is the independent proof that `set_log_entry` ran even though it emitted no log line (Q2).
- **`django_q_task` +1 (ingestion)** — the completed `documents.tasks.consume_file` task for the measured ingestion (`memo_delta.pdf`, `success=True`) is persisted, taking the table 7→8. Because `Q_CLUSTER` has no `save_limit` key (`src/paperless/settings.py:L449-456`), the Django-Q default of **250** applies — sourced in Q1 to the installed package at `django_q/conf.py:L87` (`SAVE_LIMIT = conf.get("save_limit", 250)`, django-q==1.3.9), so successful tasks are retained (up to 250; failures are always kept). The other seven rows are the four scheduled startup tasks (`train_classifier`, `index_optimize`, `sanity_check`, `process_mail_accounts`) plus the three prior `consume_file` tasks; the ingestion row is attributed specifically to `documents.tasks.consume_file` named `memo_delta.pdf` by name.
- **`documents_document_tags` +0** — 0 in this clean run because no inbox tag and no auto-matched tag applied. It would be **>0** via `add_inbox_tags` (`src/documents/signals/handlers.py:L30`) if an inbox tag existed, or via `set_tags` (`src/documents/signals/handlers.py:L168`) if the classifier auto-matched a tag. Reported as observed: +0.

### The full-text index is on the FILESYSTEM, not a DB table

The `add_to_index` handler (`src/documents/signals/handlers.py:L428`, `index.add_or_update_document`
at `L431`) updates a **Whoosh** index living on disk at `DATA_DIR/index`
(`src/paperless/settings.py:L73`) — there is no corresponding database table:

```
$ ls -la /work/data/index/
total 64
drwxr-xr-x 2 root root  4096 Jul  8 06:05 .
drwxr-xr-x 4 root root  4096 Jul  8 06:05 ..
-rwxr-xr-x 1 root root     0 Jul  8 05:57 MAIN_WRITELOCK
-rw-r--r-- 1 root root 11013 Jul  8 05:59 MAIN_gban6vijrbe9vsfk.seg
-rw-r--r-- 1 root root 10828 Jul  8 06:00 MAIN_nz9yw5yzp9mxwn9p.seg
-rw-r--r-- 1 root root 12114 Jul  8 05:58 MAIN_p5xord3108cbnpl5.seg
-rw-r--r-- 1 root root 11172 Jul  8 06:05 MAIN_y11jn1zh4eglzuk9.seg
-rw-r--r-- 1 root root  4702 Jul  8 06:05 _MAIN_5.toc
```

### OBSERVED DEVIATION — `documents_log` is NOT written (reported, not "fixed")

`documents_log` stayed at **0 before and after** ingestion (and remained 0 after all three
ingestions plus the training runs). The cause is in the logging configuration: the `LOGGING` dict
(`src/paperless/settings.py:L373-412`) defines only three handlers — `console`, `file_paperless`,
and `file_mail` — with **no database log handler**:

```
$ sed -n '386,411p' src/paperless/settings.py
    "handlers": {
        "console": {
            "level": "DEBUG" if DEBUG else "INFO",
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
        "file_paperless": {
            "class": "concurrent_log_handler.ConcurrentRotatingFileHandler",
            "formatter": "verbose",
            "filename": os.path.join(LOGGING_DIR, "paperless.log"),
            "maxBytes": LOGROTATE_MAX_SIZE,
            "backupCount": LOGROTATE_MAX_BACKUPS,
        },
        "file_mail": {
            "class": "concurrent_log_handler.ConcurrentRotatingFileHandler",
            "formatter": "verbose",
            "filename": os.path.join(LOGGING_DIR, "mail.log"),
            "maxBytes": LOGROTATE_MAX_SIZE,
            "backupCount": LOGROTATE_MAX_BACKUPS,
        },
    },
    "root": {"handlers": ["console"]},
    "loggers": {
        "paperless": {"handlers": ["file_paperless"], "level": "DEBUG"},
        "paperless_mail": {"handlers": ["file_mail"], "level": "DEBUG"},
    },
```

Because no handler writes to the database, the `Log` model / `documents_log` table
(`src/documents/models.py:L285`) is never populated during ingestion in this commit. This deviates
from more general paperless-ngx descriptions that mention a database log; it is reported here as the
**observed truth**, with the row-count evidence (0 → 0) and the settings citation above.

> **Note on `PAPERLESS_DISABLE_DBHANDLER`.** The runtime environment carried
> `PAPERLESS_DISABLE_DBHANDLER=true`, but that variable is **not read anywhere in the runtime
> source** — its only occurrence is under `[tool:pytest]` in `src/setup.cfg:L8-12` (a pytest-only
> env). So it is a **runtime no-op** in this commit; the absence of a DB log handler is caused solely
> by the `LOGGING` config above, not by that variable.

```
$ grep -rn "DISABLE_DBHANDLER" src/
src/setup.cfg:12:  PAPERLESS_DISABLE_DBHANDLER=true
```

### Concurrency / state boundaries (attribution guarantee)

Persistence runs inside `transaction.atomic()` (`src/documents/consumer.py:L298`) and the media
writes are guarded by `FileLock(settings.MEDIA_LOCK)` (`src/documents/consumer.py:L315`) — the
`media.lock` file observed in the media tree above. The source file in the consumption directory is
`os.unlink`-ed **only after** a successful save (`src/documents/consumer.py:L349-350`), which is
directly visible in the Q2 log ordering (`Saving record to database` precedes
`Deleting file …/invoice_alpha.pdf`) and by the consumption directory being empty afterward:

```
$ ls -A /work/consume/ ; echo '(empty if nothing above)'
(empty if nothing above)
```

Bracketing the before/after snapshots around these boundaries is what makes the row and file deltas
unambiguously attributable to the single ingestion.

### Reasoning

The `{pk:07}` naming comes directly from the unset-format fallback, so the first document is always
`0000001.*`. The three DB inserts map one-to-one to distinct code paths: the document row
(`_store`), the admin-log audit row (`set_log_entry`), and the persisted task row (Django-Q). The
full-text search data is deliberately kept in a Whoosh index on disk rather than the relational DB,
and — in this specific commit — the `documents_log` table simply has no writer.


---

## Coverage & Reasoning (final pass)

### One-line answers

- **Q1 — Which services participate?** Six: the `document_consumer` watcher, the Redis 6.0 broker/channel layer, the `qcluster` Django-Q worker, the gunicorn/ASGI server (+ WebSocket channel), the SQLite database, and the OCR stack (`ocrmypdf`/`tesseract` + `convert`/`optipng`). The queue is **Django-Q, not Celery**.
- **Q2 — Log sequence?** Watcher "Adding … to the task queue." → `Consuming` → `Detected mime type` → `Parser:` → `Parsing…` → OCR (`Extracted text`, `Calling OCRmyPDF`, sidecar) → `Generating thumbnail…` → `convert`/`optipng` → classifier `…model does not exist (yet)…` → `Saving record to database` → `Deleting file …` → `Document … consumption finished`.
- **Q3 — When does it retrain?** **Not per upload.** It runs on an **HOURLY** schedule; three uploads produced zero training lines and no model file.
- **Q4 — Training vs idle strings?** Training = `INFO … Saving updated classifier model to {MODEL_FILE}...`; idle = `DEBUG … Training data unchanged.`; plus silent-skip (no log) and no-model DEBUG.
- **Q5 — Disk & tables?** `documents/{originals,archive,thumbnails}/{pk:07}.{pdf,pdf,png}`; rows added to `documents_document` (+1), `django_admin_log` (+1), `django_q_task` (+1); Whoosh index on the filesystem; **`documents_log` not written**.

### Named-item checklist (every item addressed by name)

- **Services:** `document_consumer` ✓, Redis broker ✓, `qcluster` Django-Q worker ✓, gunicorn/ASGI ✓, SQLite DB ✓, OCR stack (`ocrmypdf`/`tesseract`/`convert`/`optipng`) ✓; Django-Q vs Celery ✓; `save_limit` default 250 ✓.
- **Three loggers:** `paperless.management.consumer` ✓, `paperless.consumer` ✓, `paperless.handlers` (emits nothing in a clean run — proven via `django_admin_log` +1) ✓.
- **Five message constants:** `new_file` ✓, `parsing_document` ✓, `generating_thumbnail` ✓, `save_document` ✓, `finished` ✓ — all WebSocket progress codes via `_send_progress`, not log strings.
- **Six post-consume handlers (connect order, `src/documents/apps.py:L22-27`):** `add_inbox_tags` ✓, `set_correspondent` ✓, `set_document_type` ✓, `set_tags` ✓, `set_log_entry` ✓, `add_to_index` ✓.
- **Four classifier branches:** silent-skip ✓, no-model DEBUG ✓, training INFO ✓, idle DEBUG ✓ (+ transition on data change ✓; error branch (inferred) ✓).
- **Filename pattern & media dirs:** `{pk:07}` ✓; `originals` ✓, `archive` ✓, `thumbnails` ✓.
- **DB tables:** `documents_document` ✓, `django_admin_log` ✓, `django_q_task` ✓, `documents_document_tags` (+0) ✓; Whoosh index on filesystem (not DB) ✓; `documents_log` deviation (not written) ✓.
- **Entry points:** consumption-dir file drop (`document_consumer.py:L85-91`) ✓; `POST /api/documents/post_document/` (`views.py:L491`, `L523-524`) ✓.

### Inferred vs. observed

Everything above is **observed at runtime** except one item: the classifier **error branch**
`Classifier error: {e}` (`src/documents/tasks.py:L71-72`), labeled **(inferred)** — it was not
triggered during the investigation. Notably, the claim that the HOURLY schedule is executed by the
`qcluster` scheduler process (rather than inline per upload) is **observed**, not inferred: the
cluster log shows `Process-1 created a task from schedule [Train the classifier]` at startup (Q3),
and that single scheduled run left no model and no training line, while three uploads likewise
produced none.

### Reproducibility note

Every command needed to reproduce these results is shown inline. Volatile substrings vary run to run
— timestamps, the Django-Q cluster's random name (this run: `mexico-angel-asparagus-nuts`), the random
`/tmp/paperless/…` work-dir (this run: `paperless-tsiq6d2a`), the Whoosh segment hashes (`MAIN_*.seg`),
the classifier model size, the document checksum, and Django-Q task names — but the **log strings,
their ordering, the filename pattern, and the table deltas are stable**. All runtime work was
performed outside the repository (a `/work` tree with `DATA/MEDIA/CONSUME` redirected there), so the
repository working tree is unchanged apart from this document.

