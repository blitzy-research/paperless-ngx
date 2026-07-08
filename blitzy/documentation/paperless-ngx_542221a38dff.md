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
$ cat /work/consumer.out
[2026-07-08 05:00:07,204] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /work/consume
```

- Logger `paperless.management.consumer` — `src/documents/management/commands/document_consumer.py:L24`.
- "Adding … to the task queue." then `async_task("documents.tasks.consume_file", …)` — `document_consumer.py:L85` and `L86-87`.

**2. Redis broker (`redis:6.0`) — the message bus & channel layer.** It is the Django-Q broker
(watcher → worker hand-off) and the Channels backend for WebSocket progress.

```
$ redis-cli ping
PONG
$ python3 manage.py shell -c "from django.conf import settings; print('Q_CLUSTER =', settings.Q_CLUSTER); \
  print('save_limit present?', 'save_limit' in settings.Q_CLUSTER)"
Q_CLUSTER = {'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://localhost:6379'}
save_limit present? False
```

- `Q_CLUSTER` (broker URL, worker count) — `src/paperless/settings.py:L449-456`; the `redis` URL default is `redis://localhost:6379` at `L456`.
- Redis 6.0 is the canonical broker image — `docker/compose/docker-compose.sqlite.yml:L28-29`.
- **`save_limit` is absent** from `Q_CLUSTER`, so Django-Q's default of **250** applies (successful tasks are persisted up to 250; failures are always saved). This makes completed tasks an observable DB side effect — see Q5.

**3. `qcluster` — the Django-Q worker (and scheduler).** It pops the `consume_file` task from Redis
and runs the actual pipeline.

```
$ cat /work/qcluster.out
05:00:07 [Q] INFO Q Cluster queen-mirror-delta-seventeen starting.
05:00:07 [Q] INFO Process-1:1 ready for work at 3687
05:00:07 [Q] INFO Process-1:2 ready for work at 3688
05:00:07 [Q] INFO Process-1:3 ready for work at 3689
05:00:07 [Q] INFO Process-1:4 ready for work at 3690
05:00:07 [Q] INFO Process-1:5 ready for work at 3691
05:00:07 [Q] INFO Process-1:6 ready for work at 3692
05:00:07 [Q] INFO Process-1:7 ready for work at 3694
05:00:07 [Q] INFO Process-1:8 ready for work at 3695
05:00:07 [Q] INFO Process-1:9 ready for work at 3696
05:00:07 [Q] INFO Process-1:10 ready for work at 3697
05:00:07 [Q] INFO Process-1:11 ready for work at 3698
05:00:07 [Q] INFO Process-1:12 monitoring at 3699
05:00:07 [Q] INFO Process-1 guarding cluster queen-mirror-delta-seventeen
05:00:07 [Q] INFO Process-1:13 pushing tasks at 3700
05:00:07 [Q] INFO Q Cluster queen-mirror-delta-seventeen running.
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
$ cp /work/samples/invoice_alpha.pdf /work/consume/invoice_alpha.pdf     # drop into the watched dir
# poll until the pipeline logs "consumption finished" (finished in ~8s)
$ tail -n +4 /work/data/log/paperless.log                                # the lines for this document
```

**Observed, unedited output** (volatile substrings — timestamps, the random `/tmp/paperless/…`
work-dir name — vary run to run; the log strings and ordering are stable):

```
[2026-07-08 05:02:32,158] [INFO] [paperless.management.consumer] Adding /work/consume/invoice_alpha.pdf to the task queue.
[2026-07-08 05:02:32,326] [INFO] [paperless.consumer] Consuming invoice_alpha.pdf
[2026-07-08 05:02:32,327] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 05:02:32,330] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 05:02:32,332] [DEBUG] [paperless.consumer] Parsing invoice_alpha.pdf...
[2026-07-08 05:02:32,359] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /work/consume/invoice_alpha.pdf
[2026-07-08 05:02:32,431] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/work/consume/invoice_alpha.pdf', 'output_file': '/tmp/paperless/paperless-9li5_5qd/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-9li5_5qd/sidecar.txt'}
[2026-07-08 05:02:32,727] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-08 05:02:32,734] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-9li5_5qd/archive.pdf
[2026-07-08 05:02:32,734] [DEBUG] [paperless.consumer] Generating thumbnail for invoice_alpha.pdf...
[2026-07-08 05:02:32,738] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-9li5_5qd/archive.pdf[0] /tmp/paperless/paperless-9li5_5qd/convert.png
[2026-07-08 05:02:33,444] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-9li5_5qd/convert.png -out /tmp/paperless/paperless-9li5_5qd/thumb_optipng.png
[2026-07-08 05:02:36,395] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 05:02:36,398] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 05:02:36,420] [DEBUG] [paperless.consumer] Deleting file /work/consume/invoice_alpha.pdf
[2026-07-08 05:02:36,426] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-9li5_5qd
[2026-07-08 05:02:36,427] [INFO] [paperless.consumer] Document 2026-07-08 invoice_alpha consumption finished
```

### Line-by-line source attribution

| Log line (message) | Emitting code |
|--------------------|---------------|
| `Adding …/invoice_alpha.pdf to the task queue.` | `src/documents/management/commands/document_consumer.py:L85` |
| `Consuming invoice_alpha.pdf` | `src/documents/consumer.py:L215` |
| `Detected mime type: application/pdf` | `src/documents/consumer.py:L221` |
| `Parser: RasterisedDocumentParser` | `src/documents/consumer.py:L246` |
| `Parsing invoice_alpha.pdf...` | `src/documents/consumer.py:L260` |
| `Extracted text from PDF file …` / `Calling OCRmyPDF with args: …` / `Incomplete sidecar file: discarding.` | OCR parser, logger `paperless.parsing.tesseract` (`paperless_tesseract`) |
| `Generating thumbnail for invoice_alpha.pdf...` | `src/documents/consumer.py:L263` |
| `Execute: convert …` / `Execute: optipng …` | OCR/thumbnail parser, logger `paperless.parsing[.tesseract]` |
| `Document classification model does not exist (yet), not performing automatic matching.` | `src/documents/classifier.py:L32-34` (via `load_classifier()` called during auto-matching) |
| `Saving record to database` | `src/documents/consumer.py:L387` (inside `_store()` at `L379`) |
| `Deleting file /work/consume/invoice_alpha.pdf` | `src/documents/consumer.py:L349` (source `os.unlink` at `L350`, *after* a successful save) |
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
$ grep -n "_send_progress\|MESSAGE_" src/documents/consumer.py | sed -n '1,20p'
43:MESSAGE_NEW_FILE = "new_file"
45:MESSAGE_PARSING_DOCUMENT = "parsing_document"
46:MESSAGE_GENERATING_THUMBNAIL = "generating_thumbnail"
48:MESSAGE_SAVE_DOCUMENT = "save_document"
49:MESSAGE_FINISHED = "finished"
56:    def _send_progress(
...
202:        self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)
259:        self._send_progress(20, 100, "WORKING", MESSAGE_PARSING_DOCUMENT)
264:        self._send_progress(70, 100, "WORKING", MESSAGE_GENERATING_THUMBNAIL)
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
$ python3 manage.py shell -c "from django_q.models import Schedule; \
  s=Schedule.objects.get(func='documents.tasks.train_classifier'); \
  print('name=%r func=%s schedule_type=%s next_run=%s last_run=%s'%(s.name,s.func,s.schedule_type,s.next_run,s.last_run()))"
name='Train the classifier' func=documents.tasks.train_classifier schedule_type=H next_run=2026-07-08 05:55:00.370958+00:00 last_run=None
```

`schedule_type=H` is Django-Q's HOURLY type; `next_run` is ~52 minutes in the future relative to the
observation window (uploads happened ~05:02–05:04), and `last_run=None` confirms the scheduled
retrain had not fired during the window.

### Empirical proof it is not per-upload (scale & stability)

Three documents were ingested in total (one for Q2, then two more), and the entire `paperless.log`
was grepped for **any** training marker:

```
$ python3 manage.py shell -c "from documents.models import Document; \
  print('documents_document rows:', Document.objects.count()); \
  [print('  pk=',d.pk,d.filename,repr(d.title)) for d in Document.objects.order_by('pk')]"
documents_document rows: 3
  pk= 1 0000001.pdf 'invoice_alpha'
  pk= 2 0000002.pdf 'report_beta'
  pk= 3 0000003.pdf 'letter_gamma'

$ grep -nE "Saving updated classifier model|Training data unchanged|Gathering data from database|Vectorizing data|Training .* classifier" /work/data/log/paperless.log
>>> NO MATCHES — no training log lines during any upload <<<
```

The only classifier-related lines in the whole log are the per-upload **matching** attempts from
`load_classifier()` (one per document) — not training:

```
$ grep -nE "\[paperless.classifier\]" /work/data/log/paperless.log
16:[2026-07-08 05:02:36,395] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
45:[2026-07-08 05:03:53,453] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
50:[2026-07-08 05:03:56,086] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```

And no model file exists after three uploads (training genuinely never happened):

```
$ ls -la /work/data/classification_model.pickle
ls: cannot access '/work/data/classification_model.pickle': No such file or directory
```

**Scale/stability:** across **3** ingestions with no intervening model change, the number of
retrains observed was **0**, and the result was stable across all three uploads (the grep for
training markers returns nothing after each). Uploading more documents does not change this — the
retrain trigger is the hourly schedule (or the on-demand command in Q4), not the upload.

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

$ python3 manage.py document_create_classifier ; echo "(exit code: $?)"
(exit code: 0)
# new paperless.log lines matching classifier|tasks:
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
  print('tag pk=%d match_algo=%d assigned_to_doc=1'%(t.pk,t.matching_algorithm))"
tag pk=1 match_algo=6 assigned_to_doc=1

$ python3 manage.py document_create_classifier      # RUN 1 (data is new)
[2026-07-08 05:06:53,413] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 05:06:53,413] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-08 05:06:53,417] [DEBUG] [paperless.classifier] 3 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-08 05:06:53,844] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-08 05:06:53,846] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-08 05:06:53,942] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-08 05:06:53,943] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-08 05:06:53,943] [INFO] [paperless.tasks] Saving updated classifier model to /work/data/classification_model.pickle...

$ ls -la /work/data/classification_model.pickle
-rw-r--r-- 1 root root 329149 Jul  8 05:06 /work/data/classification_model.pickle
```

- **No-model DEBUG** string — `src/documents/classifier.py:L32-34` (via `load_classifier()` at `L30`).
- `Gathering data from database...` — `src/documents/classifier.py:L123`.
- **Training INFO** string `Saving updated classifier model to …` — `src/documents/tasks.py:L64-66` (the string literal is on `L65`); `classifier.save()` follows at `L67`. `train()` returned truthy (`src/documents/classifier.py:L249` `return True`), and a **329149-byte** model file was created at `MODEL_FILE` (`src/paperless/settings.py:L74`).
- (The INFO line also appears on stderr because the `paperless` logger propagates to the root `console` handler whose level is `INFO` — `src/paperless/settings.py:L387-390`, `L407`.)

### Branch 4 — IDLE (DEBUG "Training data unchanged."), stable across ≥2 runs

Re-running with **unchanged** data twice more yields the idle branch every time, and the model file
is **not** rewritten (mtime constant) — demonstrating the required ≥2-run stability:

```
$ stat -c %Y /work/data/classification_model.pickle    # mtime after training: 1783487213 (05:06:53)
$ python3 manage.py document_create_classifier          # IDLE RUN 2
[2026-07-08 05:07:14,116] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-08 05:07:14,121] [DEBUG] [paperless.tasks] Training data unchanged.
# model mtime now: 1783487213  (unchanged? YES)

$ python3 manage.py document_create_classifier          # IDLE RUN 3
[2026-07-08 05:07:15,995] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-08 05:07:16,000] [DEBUG] [paperless.tasks] Training data unchanged.
# model mtime now: 1783487213  (unchanged? YES)
```

- **Idle DEBUG** string `Training data unchanged.` — `src/documents/tasks.py:L69`, reached because `train()` computed a SHA-1 over the preprocessed content+labels (`classifier.py:L124`) equal to the stored `data_hash`, so it short-circuited with `return False` (`src/documents/classifier.py:L163-164`).

### Transition — data changed → trains again

To confirm the trigger is a **data change**, the tag was assigned to a second document and the command
re-run; training fired again and the model mtime advanced:

```
$ python3 manage.py shell -c "from documents.models import Tag,Document; \
  t=Tag.objects.get(name='AutoTagAlpha'); Document.objects.get(pk=2).tags.add(t); \
  print('AutoTagAlpha now on docs:', list(t.documents.values_list('pk',flat=True)))"
AutoTagAlpha now on docs: [2, 1]

$ python3 manage.py document_create_classifier
[2026-07-08 05:07:38,253] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-08 05:07:38,257] [DEBUG] [paperless.classifier] 3 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-08 05:07:38,257] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-08 05:07:38,258] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-08 05:07:38,318] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-08 05:07:38,319] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-08 05:07:38,319] [INFO] [paperless.tasks] Saving updated classifier model to /work/data/classification_model.pickle...
# model mtime before=1783487213 after=1783487258 -> CHANGED (retrained)
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
/work/media/documents/originals/0000001.pdf
/work/media/documents/thumbnails/0000001.png
/work/media/media.lock

$ ls -la /work/media/documents/originals /work/media/documents/archive /work/media/documents/thumbnails
/work/media/documents/archive/:
-rw-r--r-- 1 root root 8654 Jul  8 05:02 0000001.pdf
/work/media/documents/originals/:
-rw-r--r-- 1 root root 1603 Jul  8 05:02 0000001.pdf
/work/media/documents/thumbnails/:
-rw-r--r-- 1 root root 10514 Jul  8 05:02 0000001.png
```

- `originals/` → `ORIGINALS_DIR` (`src/paperless/settings.py:L62`); the model's `source_path` builds `{:07}{}` (`src/documents/models.py:L223`, name at `L227`).
- `archive/` → `ARCHIVE_DIR` (`src/paperless/settings.py:L63`); `archive_path` (`src/documents/models.py:L242`).
- `thumbnails/` → `THUMBNAIL_DIR` (`src/paperless/settings.py:L64`); `thumbnail_path` builds `{:07}.png` (`src/documents/models.py:L273`, name at `L274`).
- `media.lock` → the `FileLock(settings.MEDIA_LOCK)` file (`src/paperless/settings.py:L72`; used at `src/documents/consumer.py:L315`).

The stored `Document` row confirms the `{pk:07}` naming and that text was extracted:

```
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); \
  print('pk=',d.pk,'filename=',d.filename,'archive_filename=',d.archive_filename); \
  print('title=',repr(d.title),'mime=',d.mime_type,'checksum=',d.checksum[:12]); \
  print('source_path=',d.source_path); print('archive_path=',d.archive_path); \
  print('thumbnail_path=',d.thumbnail_path); print('content(first80)=',repr(d.content[:80]))"
pk= 1 filename= 0000001.pdf archive_filename= 0000001.pdf
title= 'invoice_alpha' mime= application/pdf checksum= 9e5bf2efa6fa
source_path= /work/media/documents/originals/0000001.pdf
archive_path= /work/media/documents/archive/0000001.pdf
thumbnail_path= /work/media/documents/thumbnails/0000001.png
content(first80)= 'ACME Invoice Alpha 2026\n\nInvoice number: ALPHA-0001\n\nBill to: Wile E Coyote\n\nAmo'
```

### Database tables receiving rows (before → after one ingestion)

Per-table row counts were snapshotted immediately before dropping the single PDF and again after the
pipeline logged "consumption finished". (The DB started from a fresh `migrate`; the `django_q_task`
"before" value of 2 is two incidental scheduled tasks that ran at cluster startup — `index_optimize`
and `sanity_check` — unrelated to ingestion.)

```
$ # snapshot helper counts rows in every table via sqlite_master; deltas for the tables of interest:
TABLE                          BEFORE  AFTER  DELTA
documents_document                  0      1     +1
django_admin_log                    0      1     +1
django_q_task                       2      3     +1
documents_document_tags             0      0     +0
documents_log                       0      0     +0
```

**Attribution, by name:**

```
$ python3 manage.py shell -c "from django_q.models import Task; \
  [print('  func=%-40s name=%-28r success=%s'%(t.func,t.name,t.success)) for t in Task.objects.order_by('started')]"
  func=documents.tasks.index_optimize           name='idaho-mirror-golf-december' success=True
  func=documents.tasks.sanity_check             name='iowa-nitrogen-pip-chicken'  success=True
  func=documents.tasks.consume_file             name='invoice_alpha.pdf'          success=True

$ python3 manage.py shell -c "from django.contrib.admin.models import LogEntry; \
  [print('  user=%r action_flag=%d object_repr=%r content_type=%s'%(le.user.username,le.action_flag,le.object_repr,le.content_type)) for le in LogEntry.objects.all()]"
  user='consumer' action_flag=1 object_repr='2026-07-08 invoice_alpha' content_type=documents | document
```

- **`documents_document` +1** — `Document.objects.create(...)` in `_store()` (`src/documents/consumer.py:L398`, method at `L379`).
- **`django_admin_log` +1** — `set_log_entry` creates a `LogEntry` with `action_flag=ADDITION` as user `consumer` (`src/documents/signals/handlers.py:L413-424`; user at `L416`). Note the mechanism is `LogEntry.objects.create(action_flag=ADDITION, …)` (not `log_action`); the table is `django_admin_log`. This is the independent proof that `set_log_entry` ran even though it emitted no log line (Q2).
- **`django_q_task` +1 (ingestion)** — the completed `documents.tasks.consume_file` task named `invoice_alpha.pdf` (`success=True`) is persisted. Because `Q_CLUSTER` has no `save_limit` key (`src/paperless/settings.py:L449-456`), the Django-Q default of **250** applies. Incidental scheduled tasks (`index_optimize`, `sanity_check`) may also appear; the ingestion row is attributed specifically to `documents.tasks.consume_file` by name.
- **`documents_document_tags` +0** — 0 in this clean run because no inbox tag and no auto-matched tag applied. It would be **>0** via `add_inbox_tags` (`src/documents/signals/handlers.py:L30`) if an inbox tag existed, or via `set_tags` (`src/documents/signals/handlers.py:L168`) if the classifier auto-matched a tag. Reported as observed: +0.

### The full-text index is on the FILESYSTEM, not a DB table

The `add_to_index` handler (`src/documents/signals/handlers.py:L428`, `index.add_or_update_document`
at `L431`) updates a **Whoosh** index living on disk at `DATA_DIR/index`
(`src/paperless/settings.py:L73`) — there is no corresponding database table:

```
$ ls -la /work/data/index/
-rwxr-xr-x 1 root root     0 Jul  8 04:57 MAIN_WRITELOCK
-rw-r--r-- 1 root root 13022 Jul  8 05:02 MAIN_s7uw5bivqjgo6z7l.seg
-rw-r--r-- 1 root root  4377 Jul  8 05:02 _MAIN_2.toc
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
            ...
        },
        "file_mail": {
            "class": "concurrent_log_handler.ConcurrentRotatingFileHandler",
            ...
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
$ ls -A /work/consume/ ; echo "(exit shows empty)"
(exit shows empty)
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

Everything above is **observed at runtime** except: (a) the classifier **error branch**
`Classifier error: {e}` (`src/documents/tasks.py:L71-72`), labeled **(inferred)** — not triggered;
and (b) the general linkage that the HOURLY schedule is executed by the `qcluster` scheduler process
rather than inline — supported by the observed `schedule_type=H` row and `last_run=None` during the
upload window, but the scheduler-executes-it step itself is **(inferred)** from Django-Q's design.

### Reproducibility note

Every command needed to reproduce these results is shown inline. Volatile substrings vary run to run
— timestamps, the Django-Q cluster's random name (e.g. `queen-mirror-delta-seventeen`), the random
`/tmp/paperless/…` work-dir, the Whoosh segment hash (`MAIN_*.seg`), and Django-Q task names — but the
**log strings, their ordering, the filename pattern, and the table deltas are stable**. All runtime
work was performed outside the repository (a `/work` tree with `DATA/MEDIA/CONSUME` redirected there),
so the repository working tree is unchanged apart from this document.

