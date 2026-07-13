# Paperless-ngx Runtime Investigation — Document Processing Pipeline

**Commit:** `542221a38dff06361e07976452f9aea24d210542`
**Branch:** `paperless-ngx_542221a38dff`
**Scope:** A strictly **read-only**, **run-first** investigation of how paperless-ngx ingests a
document, when its machine-learning classifier retrains, and where processed data lands on disk and
in the database. Every behavioral claim below is backed by **actual captured runtime output** shown
next to the exact command that produced it; every code-level claim carries a `file:line` citation
naming the specific function/method/class. Statements that are derived from reading code rather than
observed at runtime are explicitly labeled **(inferred)**.

> **Task queue is django-q, not Celery.** Asynchronous work is dispatched with
> `django_q.tasks.async_task` and scheduled work with `django_q.tasks.schedule`; both run in the
> `qcluster` process. There is **no** `src/paperless/celery.py` at this commit (verified:
> `ls src/paperless/celery.py` → *No such file or directory*).
>
> **Version boundary.** The classifier here is the `FORMAT_VERSION = 7` variant
> (`src/documents/classifier.py:63`). It has **no HMAC signing, no `StoragePath` classifier, and no
> NLTK stemming**; those are features of *newer* paperless-ngx releases and are out of scope for this
> commit.

---

## Table of Contents

1. [How the environment was built and run](#1-how-the-environment-was-built-and-run)
2. [Proof the full stack was running before any document was submitted](#2-proof-the-full-stack-was-running-before-any-document-was-submitted)
3. [Q1 — The ingestion pipeline: which services are involved and the ordered log sequence](#3-q1--the-ingestion-pipeline)
4. [Q2 — Classifier retraining: conditions and the distinguishing log messages](#4-q2--classifier-retraining)
5. [Q3 — On-disk layout and which database tables receive new rows](#5-q3--on-disk-layout-and-database-tables)
6. [Corroborating research (validates, does not replace, the runtime evidence)](#6-corroborating-research)
7. [Coverage confirmation](#7-coverage-confirmation)
8. [Cleanup note](#8-cleanup-note)

---

## 1. How the environment was built and run

The stack was run from the canonical Docker image specified in the setup instructions
(`ghcr.io/scaleapi/swe-atlas:...paperless-ngx...542221a38dff...`), whose main application stage is
built `FROM python:3.9-slim-bullseye` (`Dockerfile:18`). Two containers were used: the paperless-ngx
application container (`pngx`) and a **Redis 6.0** broker container (`pngx-redis`), mirroring
`docker/compose/docker-compose.sqlite.yml:28-29` (`broker: image: redis:6.0`). The three long-running
programs match `docker/supervisord.conf` exactly:

| Service | supervisord program | Command | Citation |
|---|---|---|---|
| Web server (ASGI) | `[program:gunicorn]` | `gunicorn -c … paperless.asgi:application` | `docker/supervisord.conf:10-11` |
| Consume-dir watcher | `[program:consumer]` | `python3 manage.py document_consumer` | `docker/supervisord.conf:19-20` |
| Worker + scheduler | `[program:scheduler]` | `python3 manage.py qcluster` | `docker/supervisord.conf:28-29` |

The image does not ship supervisor, so the three programs were launched manually (out-of-repo helper
scripts), which is behaviorally identical to the supervisord definitions above. The commands below are
the **exact, literal, runnable** contents of the two helper scripts — there are no elisions or
placeholders. Three deliberate hardening choices are called out inline and are the only departures from
a naive bring-up; each is disclosed here so the reader can reproduce or relax it knowingly:

1. **Services run as non-root UID/GID 1000** — this *matches* the canonical `user=paperless` in every
   `docker/supervisord.conf` program, and UID 1000 is the `paperless` user created by `Dockerfile:157-159`
   (in this image UID 1000 is `testuser`, the same uid/gid). Running non-root avoids root-owned runtime
   artifacts.
2. **The web port is published to loopback only** (`127.0.0.1:8000`), not `0.0.0.0`. gunicorn still
   binds `0.0.0.0:8000` *inside* the container (its canonical `gunicorn.conf.py` value); only the host
   publish is restricted.
3. **A fresh, throwaway, test-only `PAPERLESS_SECRET_KEY` is injected** (generated with
   `python3 -c "import secrets; print(secrets.token_hex(24))"`). It is not a production credential and is
   discarded with the container.

```bash
# ============================= bring-up.sh (out-of-repo helper) =============================
#!/bin/bash
set -eu
IMAGE="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01"
TEST_SECRET="2b58f96d8b4840982a3512502dd8d0929450bf8c17acf383"   # throwaway, test-only

docker network create pngx-net
docker run -d --name pngx-redis --network pngx-net redis:6.0     # broker: redis:6.0
docker run -d --name pngx --network pngx-net \
    -e PAPERLESS_REDIS=redis://pngx-redis:6379 \
    -e PAPERLESS_TIME_ZONE=UTC \
    -e PAPERLESS_SECRET_KEY="$TEST_SECRET" \
    -p 127.0.0.1:8000:8000 \
    --entrypoint /bin/bash "$IMAGE" -c "sleep infinity"

# canonical RUNTIME_PACKAGES missing from the image, plus the PDF policy fix:
docker exec pngx bash -lc 'export DEBIAN_FRONTEND=noninteractive; apt-get update -qq && \
    apt-get install -y --no-install-recommends libzbar0 poppler-utils'
docker exec pngx bash -lc 'cp /app/docker/imagemagick-policy.xml /etc/ImageMagick-6/policy.xml'

# runtime dirs owned by UID 1000 so the services can run non-root:
docker exec pngx bash -lc 'mkdir -p /app/data /app/media /app/consume /app/logs /tmp/paperless && \
    chown -R 1000:1000 /app/data /app/media /app/consume /app/logs /tmp/paperless'

# schema + a fresh test-only superuser (admin/admin), run as UID 1000:
docker exec -u 1000:1000 -w /app/src pngx bash -lc 'python3 manage.py migrate --no-input'
docker exec -u 1000:1000 -e DJANGO_SUPERUSER_PASSWORD=admin -w /app/src pngx bash -lc \
    'python3 manage.py createsuperuser --no-input --username admin --email admin@example.com'

# ========================= start-services.sh (run INSIDE the container) =====================
#!/bin/bash
# launched via: docker exec -u 1000:1000 pngx bash /tmp/start-services.sh
set -u
cd /app/src
setsid bash -c 'gunicorn -c /app/gunicorn.conf.py paperless.asgi:application \
    >> /app/logs/gunicorn.log 2>&1' < /dev/null &
setsid bash -c 'python3 manage.py document_consumer >> /app/logs/consumer.log 2>&1' < /dev/null &
setsid bash -c 'python3 manage.py qcluster          >> /app/logs/qcluster.log 2>&1' < /dev/null &
```

Notes on faithfulness:

- `libzbar0` is required because `src/documents/tasks.py:25` imports `from pyzbar import pyzbar` at
  module load time; without the native `zbar` library, **every** django-q task fails to import. This
  is a canonical runtime dependency of the image, not a behavior change.
- `docker/supervisord.conf:11` references `/usr/src/paperless/gunicorn.conf.py`; because the source is
  baked at `/app` in this image, the runtime used `/app/gunicorn.conf.py`. The server, args, and ASGI
  application (`paperless.asgi:application`) are identical.
- **Configuration is canonical (default):** `PAPERLESS_FILENAME_FORMAT` is unset
  (`src/paperless/settings.py:584`), the database is the default **SQLite** at `DATA_DIR/db.sqlite3`
  (`src/paperless/settings.py:300`), and `PAPERLESS_DEBUG` is unset so `settings.DEBUG` is `False`
  (`src/paperless/settings.py:50`) — all confirmed at runtime in §2. All backend commands below are
  issued as the non-root service user with `docker exec -u 1000:1000 -w /app/src pngx …`; the injected
  `PAPERLESS_SECRET_KEY` affects neither the filename format, the database, nor `DEBUG`.

**The logging surface used for observation.** paperless-ngx configures a dedicated file handler for
the `paperless` logger:

- `"paperless"` logger → `handlers=["file_paperless"]`, `level DEBUG` (`src/paperless/settings.py:409`)
- `file_paperless` writes to `LOGGING_DIR/paperless.log` (`src/paperless/settings.py:395`), i.e.
  `/app/data/log/paperless.log`
- the root console handler is `"DEBUG" if DEBUG else "INFO"` (`src/paperless/settings.py:388`); with
  the canonical `DEBUG=False`, the console (captured to `/app/logs/qcluster.log`) shows **INFO+** while
  `/app/data/log/paperless.log` shows the **full DEBUG** stream.

Because the `paperless` logger is at DEBUG by default, the DEBUG-level "idle" training line is visible
in the canonical configuration with **no config change** — satisfying the default-config requirement.
The log format is `"[{asctime}] [{levelname}] [{name}] {message}"` (`src/paperless/settings.py:377-379`),
so each line below is attributed to a service by its `[name]` (logger namespace) and `[levelname]`.

---

## 2. Proof the full stack was running before any document was submitted

**Both containers up** (`pngx` published to loopback only):

```text
$ docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}  {{.Ports}}'
pngx        ghcr.io/scaleapi/swe-atlas:...paperless-ngx...qna_1.01  Up 8 minutes  127.0.0.1:8000->8000/tcp
pngx-redis  redis:6.0                                               Up 8 minutes  6379/tcp
```

**All three services live, running as the non-root service user (UID 1000).** The image ships no `ps`,
so the process table was read directly from `/proc` by a small out-of-repo helper. The helper resolves
each process's **real UID**, **sorts by PID**, and **explicitly excludes the transient inspection
process** (its own PID and its `docker exec` shell) so no observer artifact is mistaken for a service.
Its complete source (cleaned up on completion, per §8):

```python
# proc_snapshot.py — walks /proc, resolves real UID, sorts by PID, excludes self + exec-shell
import os
self_pid = os.getpid()
try:
    ppid = os.getppid()
except Exception:
    ppid = -1
rows = []
for name in sorted(os.listdir('/proc'), key=lambda x: int(x) if x.isdigit() else 1<<30):
    if not name.isdigit():
        continue
    pid = int(name)
    if pid in (self_pid, ppid):                      # exclude the inspector + its exec shell
        continue
    try:
        with open(f'/proc/{pid}/status') as f:
            st = f.read()
        ruid = next(l for l in st.splitlines() if l.startswith('Uid:')).split()[1]
        with open(f'/proc/{pid}/cmdline','rb') as f:
            cmd = f.read().replace(b'\x00', b' ').strip().decode('utf-8','replace')
    except (FileNotFoundError, ProcessLookupError, StopIteration, PermissionError):
        continue
    rows.append((pid, ruid, cmd))
print(f"{'PID':>6}  {'UID':>5}  CMDLINE")
for pid, ruid, cmd in rows:
    print(f"{pid:>6}  {ruid:>5}  {cmd[:140]}")
print(f"\n(excluded transient inspection pids: self={self_pid}, exec-shell={ppid})")
```

```text
$ docker exec -u 1000:1000 pngx python3 /tmp/proc_snapshot.py
   PID    UID  CMDLINE
     1      0  sleep infinity
   467   1000  bash -c gunicorn -c /app/gunicorn.conf.py paperless.asgi:application \
  >> /app/logs/gunicorn.log 2>&1
   468   1000  bash -c python3 manage.py document_consumer \
  >> /app/logs/consumer.log 2>&1
   469   1000  bash -c python3 manage.py qcluster \
  >> /app/logs/qcluster.log 2>&1
   471   1000  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
   472   1000  python3 manage.py qcluster
   473   1000  python3 manage.py document_consumer
   475   1000  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
   476   1000  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
   500   1000  python3 manage.py qcluster
   505   1000  python3 manage.py qcluster
   506   1000  python3 manage.py qcluster
   507   1000  python3 manage.py qcluster
   508   1000  python3 manage.py qcluster
   509   1000  python3 manage.py qcluster
   510   1000  python3 manage.py qcluster
   511   1000  python3 manage.py qcluster
   512   1000  python3 manage.py qcluster
   513   1000  python3 manage.py qcluster
   571   1000  python3 manage.py qcluster
   572   1000  python3 manage.py qcluster
   573   1000  python3 manage.py qcluster
   574   1000  python3 manage.py qcluster

(excluded transient inspection pids: self=903, exec-shell=0)
```

**Reading the snapshot.** `PID 1` is the container's init (`sleep infinity`, the `--entrypoint /bin/bash
-c "sleep infinity"` process) and is the **only** process owned by root (`UID 0`); it is the container
placeholder, not a paperless service. PIDs `467/468/469` are the three `setsid bash -c` wrappers from
`start-services.sh` (their cmdlines carry the embedded newline from the script's line-continuation,
shown faithfully). `PID 471` is the gunicorn master with workers `475`/`476`; `PID 473` is the single
`document_consumer`; `PID 472` plus `500–513` and `571–574` are the django-q `qcluster` worker pool
(django-q recycles workers, so these pids change over time). **Every paperless service process runs as
UID 1000**, matching the canonical `user=paperless` in `docker/supervisord.conf`.

**Redis broker live** (the broker that transports every django-q task between producer and worker).
Liveness was proven both directly against the broker container and from *inside* the app container over
the `pngx-net` network, using the runtime `PAPERLESS_REDIS` URL:

```text
$ docker exec pngx-redis redis-cli PING
PONG
$ docker exec pngx bash -lc 'python3 -c "import redis,os; print(redis.from_url(os.environ[\"PAPERLESS_REDIS\"]).ping())"'
True
```

**Cause → effect:** the app resolves its broker from `PAPERLESS_REDIS=redis://pngx-redis:6379` into the
`redis` key of `Q_CLUSTER` (`src/paperless/settings.py:449,456`); the `True` from the app container is
what guarantees the enqueue → worker hand-off in §3 can actually happen.

**Web server answering** (the container lacks `curl`; the request is issued from the host, where port
8000 is published to loopback only):

```text
$ curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/api/
200
```

**Canonical SQLite database and broker wiring** (via Django settings at runtime, read as UID 1000):

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from django.conf import settings
print('ENGINE                    :', settings.DATABASES['default']['ENGINE'])
print('NAME                      :', settings.DATABASES['default']['NAME'])
print('PAPERLESS_FILENAME_FORMAT :', repr(settings.PAPERLESS_FILENAME_FORMAT))
print('settings.DEBUG            :', settings.DEBUG)
print('Q_CLUSTER redis           :', settings.Q_CLUSTER.get('redis'))"

ENGINE                    : django.db.backends.sqlite3
NAME                      : /app/src/../data/db.sqlite3
PAPERLESS_FILENAME_FORMAT : None
settings.DEBUG            : False
Q_CLUSTER redis           : redis://pngx-redis:6379
```

This confirms the default database (`src/paperless/settings.py:300`), the unset default filename
format (`src/paperless/settings.py:584`), canonical `DEBUG=False` (`src/paperless/settings.py:50`), and
the Redis broker key of `Q_CLUSTER` (`src/paperless/settings.py:449,456`).

**The classifier-training schedule already exists** (seeded by migration
`src/documents/migrations/1001_auto_20201109_1636.py:10-14`). This is the single most important piece
of evidence for Q2: training is a **scheduled** task, not something ingestion runs.

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from django_q.models import Schedule
for s in Schedule.objects.order_by('id'):
    print('id=%s  func=%-42s name=%-27s type=%s' % (s.id, s.func, repr(s.name), s.schedule_type))
print('Schedule.HOURLY =', repr(Schedule.HOURLY))"

id=1  func=documents.tasks.train_classifier           name='Train the classifier'      type=H
id=2  func=documents.tasks.index_optimize             name='Optimize the index'        type=D
id=3  func=documents.tasks.sanity_check               name='Perform sanity check'      type=W
id=4  func=paperless_mail.tasks.process_mail_accounts name='Check all e-mail accounts' type=I
Schedule.HOURLY = 'H'
```

**Cause → effect:** migration `1001_auto_20201109_1636.py:10-14` calls
`schedule("documents.tasks.train_classifier", name="Train the classifier", schedule_type=Schedule.HOURLY)`,
which is why a row with `func=documents.tasks.train_classifier` and `type=H` (HOURLY) exists at runtime.
Because this schedule — not the consumption task — is what drives training, **uploading a document never
trains the classifier by itself** (proven in §4).

**The scheduler was observed firing autonomously.** Within a minute of cluster start, django-q created
and ran the scheduled training task on its own — captured unedited from `/app/logs/qcluster.log`:

```text
18:13:03 [Q] INFO Enqueued 1
18:13:03 [Q] INFO Process-1 created a task from schedule [Train the classifier]
18:13:03 [Q] INFO Process-1:1 processing [vermont-sixteen-august-india]
```

Crucially, this scheduled run produced **no** `paperless.tasks` training line in
`/app/data/log/paperless.log`, because on the fresh install there is no `MATCH_AUTO` entity — the guard
branch (§4.3) returns before logging. This is the first live proof that a running scheduler alone does
not train; the conditions in §4 must be met.


---

## 3. Q1 — The ingestion pipeline

> *"Once the application is up and all of its services are running, I want to submit a test PDF and
> observe how the ingestion pipeline actually behaves. As the document moves through the system, which
> services are involved, and what sequence of events shows up in the logs?"*

### 3.1 Which services are involved

A single PDF submission is handled by a chain of cooperating services. The entry point only *enqueues*
work; the actual processing happens in a separate worker process:

| Order | Service (process) | Role | Logger namespace seen |
|---|---|---|---|
| 1 | **gunicorn** (web) *or* **document_consumer** (watcher) | Accepts the document and enqueues a django-q task | `[Q]` / `paperless.management.consumer` |
| 2 | **Redis** broker | Transports the task from producer to worker | — |
| 3 | **qcluster** (django-q worker) | Dequeues and executes `documents.tasks.consume_file` | `[Q]` |
| 4 | `Consumer.try_consume_file()` (inside qcluster) | Parses/OCRs, thumbnails, stores the `Document` | `paperless.consumer`, `paperless.parsing[.tesseract]`, `paperless.classifier` |
| 5 | Post-consumption signal handlers (inside qcluster) | Inbox tags, matching, admin log entry, search index | `paperless.handlers` |
| 6 | SQLite DB + media dirs + Whoosh index | Durable storage of row, files, and full-text index | — |

**Attribution rule:** each log line is attributed by its `[name]`:
`paperless.management.consumer` = the directory watcher; `[Q]` = django-q (enqueue/worker);
`paperless.consumer` = `Consumer` (`src/documents/consumer.py:54`, `logging_name = "paperless.consumer"`);
`paperless.parsing[.tesseract]` = the PDF parser/OCR; `paperless.classifier` = classifier load;
`paperless.handlers` = the post-consumption handlers (`src/documents/signals/handlers.py:27`).

The progress arrow is deliberately drawn to a **Channels** participant, not back to the HTTP client:
the REST client already received `"OK"` and disconnected; live progress is published to the Channels
`status_updates` group over Redis (`consumer.py:73-74`) for WebSocket subscribers (the SPA), so it never
reaches the `curl` caller and never appears in the logs (see §3.4).

```mermaid
sequenceDiagram
    participant Client
    participant Web as gunicorn (web)
    participant Watch as document_consumer
    participant Redis as Redis broker
    participant Worker as qcluster (django-q)
    participant Consumer as Consumer.try_consume_file
    participant Handlers as 6 finished-signal handlers
    participant Store as SQLite + media + Whoosh
    participant Channels as Channels status_updates (WebSocket)

    alt REST upload (used here)
        Client->>Web: POST /api/documents/post_document/  [views.py:497]
        Web->>Redis: async_task("documents.tasks.consume_file")  [views.py:523]
        Web-->>Client: HTTP 200 "OK"  [views.py:535]
    else Consume directory
        Client->>Watch: drop file into CONSUMPTION_DIR
        Watch->>Redis: async_task("documents.tasks.consume_file")  [document_consumer.py:85-86]
    end
    Redis->>Worker: deliver consume_file
    Worker->>Consumer: try_consume_file(path)  [tasks.py:236 -> consumer.py:180]
    Consumer->>Consumer: mime detect, parse/OCR, thumbnail, date
    Consumer->>Store: transaction.atomic() -> _store() saves Document  [consumer.py:298,398]
    Consumer->>Handlers: document_consumption_finished.send()  [consumer.py:306]
    Handlers->>Store: inbox tag, matches, admin LogEntry, Whoosh index
    Consumer->>Store: move original/archive + rename under FileLock  [consumer.py:315-316]
    Consumer->>Redis: _send_progress(...) -> group_send("status_updates")  [consumer.py:73-74]
    Redis->>Channels: 0->100 SUCCESS (WebSocket only; never sent to the REST client, never logged)
```

### 3.2 Submitting a test PDF through a real entry point

A synthetic, text-bearing PDF was generated by a temporary helper **outside** the repository tree
(`reportlab`); its complete source (cleaned up on completion, per §8) is inlined here:

```python
# make_pdf.py — temporary helper (outside the repo tree; removed in §8). Full source, inlined.
# Generates a small, single-page, text-bearing PDF so the consumer's parse/OCR stage has real
# text to extract. Usage: python3 make_pdf.py <output.pdf> "<text drawn on the page>"
import sys
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4

out_path = sys.argv[1]                                   # e.g. q1_ingestion_trace.pdf
text = sys.argv[2] if len(sys.argv) > 2 else "Paperless-ngx ingestion test document"

c = canvas.Canvas(out_path, pagesize=A4)                 # default A4 page
c.setFont("Helvetica", 12)                               # built-in font (not embedded)
c.drawString(72, 760, text)                              # caller-supplied label line
c.drawString(72, 730, "Paperless-ngx runtime investigation - synthetic test document.")
c.save()                                                 # writes the single-page PDF
```

The generated PDF was then submitted through the canonical REST endpoint
`POST /api/documents/post_document/` (`src/documents/views.py:491` `class PostDocumentView`). Because the
endpoint requires authentication (`permission_classes = (IsAuthenticated,)`, `views.py:493`), a DRF token
was first obtained through the canonical token endpoint; the exact commands (no placeholders — the token
value is carried in a shell variable and only its length is echoed) were:

```text
$ TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/token/ \
             -d "username=admin&password=admin" \
         | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
$ echo "token length: ${#TOKEN}"
token length: 40

$ curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Token $TOKEN" \
       -F "document=@q1_ingestion_trace.pdf" \
       -F "title=Q1 Ingestion Trace" \
       http://127.0.0.1:8000/api/documents/post_document/
"OK"
HTTP 200
```

**Cause → effect:** `PostDocumentView` is a `GenericAPIView` gated by `IsAuthenticated` (`views.py:493`)
that accepts multipart uploads via `parser_classes = (parsers.MultiPartParser,)` (`views.py:495`). Its
`post()` (`views.py:497`) validates the payload with `serializer.is_valid(raise_exception=True)`
(`views.py:500`), writes the bytes to a `paperless-upload-` scratch file under `SCRATCH_DIR`
(`views.py:512-517`), mints a task id `task_id = str(uuid.uuid4())` (`views.py:521`), calls
`async_task("documents.tasks.consume_file", …, task_id=task_id, task_name=…)` (`views.py:531`, `async_task`
imported at `:28`), and returns `Response("OK")` (`views.py:535`) — HTTP **200** with the body literally
`"OK"`. The `uuid4` `task_id` is threaded through to **both** subsystems: passing it as `task_id=` makes
it the django-q task record's id (stored dash-stripped — the successful ingestion's row is
`id=16bd776d8f844dc8a3db53a87ccbc853` in `django_q_task`, §5.3), and `Consumer` reuses the *same* value as
its Channels progress correlation id
(`consumer.py:200`, echoed in the progress payload at `:66`). It is simply **not** returned in the
response body. Handing the task to Redis is exactly why the sequence that follows spans multiple services.

### 3.3 The ordered log sequence (complete, unedited)

To guarantee the blocks below are **complete and unedited** — not hand-selected, reordered, or
composited — each log's byte length was recorded immediately **before** the upload
(`wc -c < <logfile>`), and the block shown is exactly the new bytes appended during the single
submission (`tail -c +$((offset+1)) <logfile>`). Nothing between the pre-upload offset and end-of-file
is omitted.

**Producer side — the web server enqueues** (`/app/logs/gunicorn.log` delta):

```text
18:26:00 [Q] INFO Enqueued 1
```

**Worker side — django-q dequeues and runs the task** (`/app/logs/qcluster.log` delta). Note that
only the **INFO** `paperless.consumer` lines surface here, because the console/root handler is INFO
when `DEBUG=False` (`src/paperless/settings.py:388`):

```text
18:26:00 [Q] INFO Process-1:6 processing [q1_ingestion_trace.pdf]
[2026-07-13 18:26:00,267] [INFO] [paperless.consumer] Consuming q1_ingestion_trace.pdf
[2026-07-13 18:26:02,583] [INFO] [paperless.consumer] Document 2026-07-13 Q1 Ingestion Trace consumption finished
18:26:02 [Q] INFO Process-1:6 stopped doing work
18:26:02 [Q] INFO Processed [q1_ingestion_trace.pdf]
18:26:02 [Q] INFO recycled worker Process-1:6
18:26:02 [Q] INFO Process-1:19 ready for work at 1136
```

**The full DEBUG consumer sequence** (`/app/data/log/paperless.log` delta — the canonical DEBUG
surface described in §1). This is the complete, unedited block for the single submission:

```text
[2026-07-13 18:26:00,267] [INFO] [paperless.consumer] Consuming q1_ingestion_trace.pdf
[2026-07-13 18:26:00,268] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 18:26:00,271] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 18:26:00,284] [DEBUG] [paperless.consumer] Parsing q1_ingestion_trace.pdf...
[2026-07-13 18:26:00,315] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-a11g9fwy
[2026-07-13 18:26:00,396] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-a11g9fwy', 'output_file': '/tmp/paperless/paperless-5jousnnt/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-5jousnnt/sidecar.txt'}
[2026-07-13 18:26:00,711] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-13 18:26:00,717] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-5jousnnt/archive.pdf
[2026-07-13 18:26:00,717] [DEBUG] [paperless.consumer] Generating thumbnail for q1_ingestion_trace.pdf...
[2026-07-13 18:26:00,736] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-5jousnnt/archive.pdf[0] /tmp/paperless/paperless-5jousnnt/convert.png
[2026-07-13 18:26:01,408] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-5jousnnt/convert.png -out /tmp/paperless/paperless-5jousnnt/thumb_optipng.png
[2026-07-13 18:26:02,538] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:26:02,555] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:26:02,575] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-a11g9fwy
[2026-07-13 18:26:02,582] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-5jousnnt
[2026-07-13 18:26:02,583] [INFO] [paperless.consumer] Document 2026-07-13 Q1 Ingestion Trace consumption finished
```


### 3.4 Per-line attribution and cause → effect

Each captured line maps to a specific service and code location. `Consumer.try_consume_file()` is the
orchestrator (`src/documents/consumer.py:180`); it inherits `LoggingMixin` and logs under
`paperless.consumer` (`src/documents/consumer.py:52-54`).

| Captured line | Service / logger | Emitting code | Why it fires (cause → effect) |
|---|---|---|---|
| `[Q] INFO Enqueued 1` | gunicorn → django-q | `views.py:523` | `async_task(...)` pushes the task to Redis; django-q logs the enqueue |
| `[Q] … processing [q1_ingestion_trace.pdf]` | qcluster | django-q worker | worker dequeues the task named after the file's basename (`task_name`, `views.py:531`) |
| `Consuming q1_ingestion_trace.pdf` (INFO) | `paperless.consumer` | `consumer.py:215` | first user-visible line after the existence/dir/MD5-duplicate pre-checks (`:211-213`) |
| `Detected mime type: application/pdf` | `paperless.consumer` | `consumer.py:221` | `magic.from_file(...)` MIME sniff selects a parser |
| `Parser: RasterisedDocumentParser` | `paperless.consumer` | `consumer.py:246` | mime→parser dispatch chose the raster/PDF parser |
| `Parsing q1_ingestion_trace.pdf...` | `paperless.consumer` | `consumer.py:260` | parse stage begins (progress → 20, `:259`) |
| `Calling OCRmyPDF with args: {…}` | `paperless.parsing.tesseract` | PDF parser | OCR runs even for text PDFs with `skip_text=True`, producing a **PDF/A archive** |
| `Generating thumbnail for q1_ingestion_trace.pdf...` | `paperless.consumer` | `consumer.py:263` | thumbnail stage (progress → 70, `:264`) |
| `Execute: convert … convert.png` / `optipng …` | `paperless.parsing[.tesseract]` | thumbnailer | ImageMagick renders page 1, optipng compresses it |
| `Document classification model does not exist (yet)…` | `paperless.classifier` | `classifier.py:32-36` via `load_classifier()` (`consumer.py:292`) | no model exists on a fresh install, so **no automatic matching** happens |
| `Saving record to database` | `paperless.consumer` | `consumer.py:387` (in `_store`, `:379`) | the `Document` row is created inside `transaction.atomic()` (`consumer.py:298`, `Document.objects.create` `:398`) |
| `Deleting file …paperless-upload-a11g9fwy` | `paperless.consumer` | `consumer.py:349` | the scratch upload copy is removed after the row is saved |
| `Document … consumption finished` (INFO) | `paperless.consumer` | `consumer.py:373` | end of `try_consume_file` (post-consume script `:371`, progress → 100 SUCCESS `:375`) |

**The silent stages between these lines (source-derived).** Several pipeline stages run but emit **no
dedicated log line** on a stock install; they are therefore invisible in the capture above and are listed
here as **(source-derived)** so the sequence is not mistaken for the whole story:

| Silent stage | Code | Why it is silent here |
|---|---|---|
| `document_consumption_started` signal fan-out | `consumer.py:229` | notifies listeners ("about to do work"); no listener logs on a stock install |
| pre-consume script hook | `consumer.py:235` (`run_pre_consume_script`, `:121`) | no `PAPERLESS_PRE_CONSUME_SCRIPT` configured → returns without logging |
| text extraction | `consumer.py:271` (`document_parser.get_text()`) | the parser logs its own OCR lines; the `get_text()` call itself has no line |
| date detection | `consumer.py:272-275` (`get_date()` / `parse_date`) | only logs (progress 90, `MESSAGE_PARSE_DATE`) when no date is embedded |
| macOS "shadow" (`._`) file cleanup | `consumer.py:358-360` | only logs if a `._name` shadow file exists (none for a REST upload) |
| post-consume script hook | `consumer.py:371` (`run_post_consume_script`, `:143`) | no `PAPERLESS_POST_CONSUME_SCRIPT` configured → returns without logging |

**On WebSocket progress (source-derived).** The `Consumer` also emits progress updates
`_send_progress(0→20→70→90→95→100)` (`consumer.py:202,259,264,274,294,375`). Each call ends in
`async_to_sync(self.channel_layer.group_send)("status_updates", …)` (`consumer.py:73-74`), i.e. the
updates are published to the Channels `status_updates` group over Redis for **WebSocket** subscribers —
they are **not** returned to the `curl`/REST caller (which already received `"OK"`) and are **not**
written to any log. This is labeled source-derived because the progress path was read from the code, not
captured in the logs (by design, nothing logs it).

### 3.5 The six post-consumption handlers

After the `Document` is stored, the consumer fans out via
`document_consumption_finished.send(...)` (`src/documents/consumer.py:306`). Exactly **six** handlers
are connected, in this order, in `src/documents/apps.py:22-27`:

```python
document_consumption_finished.connect(add_inbox_tags)       # apps.py:22
document_consumption_finished.connect(set_correspondent)    # apps.py:23
document_consumption_finished.connect(set_document_type)    # apps.py:24
document_consumption_finished.connect(set_tags)             # apps.py:25
document_consumption_finished.connect(set_log_entry)        # apps.py:26
document_consumption_finished.connect(add_to_index)         # apps.py:27
```

They live in `src/documents/signals/handlers.py` under logger `paperless.handlers`
(`handlers.py:27`): `add_inbox_tags` (`:30`), `set_correspondent` (`:35`), `set_document_type`
(`:101`), `set_tags` (`:168`), `set_log_entry` (`:413`), `add_to_index` (`:428`). On a fresh install
with no matching rules the handlers run silently, so their execution was proven by their **side
effects**:

```text
# set_log_entry (handlers.py:413) -> a django_admin_log row written as the 'consumer' user:
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from django.contrib.admin.models import LogEntry
le = LogEntry.objects.latest('id')
print(le.user.username, le.action_flag, repr(le.object_repr), le.content_type)"
consumer 1 '2026-07-13 Q1 Ingestion Trace' documents | document        # action_flag 1 = ADDITION

# add_to_index (handlers.py:428) -> the document is now in the Whoosh full-text index:
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from documents import index
from whoosh.qparser import QueryParser
ix = index.open_index()
with ix.searcher() as s:
    print('Whoosh hits for id:1 ->', len(s.search(QueryParser('id', ix.schema).parse('1'))))"
Whoosh hits for id:1 -> 1

# add_inbox_tags (handlers.py:30) -> no-op here, because no inbox tag is seeded (see §4.5):
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from documents.models import Tag, Document
print('inbox tags:', Tag.objects.filter(is_inbox_tag=True).count(), '| tags on doc:', Document.objects.get().tags.count())"
inbox tags: 0 | tags on doc: 0
```

**Cause → effect:** `set_log_entry` calls `LogEntry.objects.create(...)` as the user `consumer`
(`handlers.py:416,418`), which is why a `django_admin_log` ADDITION row (`action_flag=1`) appears with
`object_repr='2026-07-13 Q1 Ingestion Trace'`; `add_to_index` updates the Whoosh index, which is why
searching the index for the new document's id (`id:1`) returns exactly one hit; `add_inbox_tags` is a
no-op because there is no inbox tag to apply (0 inbox tags, 0 tags on the document). `set_correspondent`,
`set_document_type`, and `set_tags` ran but matched nothing (no rules defined), so they made no changes.

**First file placement vs. later renames (a common point of confusion).** The **first** time the
files are written and named happens *inside the consumer*: under `with FileLock(settings.MEDIA_LOCK):`
(`consumer.py:315`) it sets `document.filename = generate_unique_filename(document)` (`consumer.py:316`)
and writes the original/thumbnail/archive. A **separate** handler,
`update_filename_and_move_files` (`handlers.py:312`, also under a `FileLock` at `:325`), is a
`post_save` handler that renames/moves files **later**, when a document's metadata changes (e.g. you
edit its correspondent). It is *not* what places the file during initial ingestion.

### 3.6 The duplicate-rejection path (a re-upload of the same bytes)

To show what happens when the *same* file is submitted again, the identical PDF was re-posted through
the same REST endpoint. The entry point still returns `"OK"` (it only *enqueues*; it does not inspect
content), but the **worker** rejects the task. Captured unedited (byte-offset deltas of
`/app/logs/qcluster.log` and `/app/data/log/paperless.log`):

```text
$ curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Token $TOKEN" \
       -F "document=@q1_ingestion_trace.pdf" -F "title=Q1 Ingestion Trace (duplicate)" \
       http://127.0.0.1:8000/api/documents/post_document/
"OK"
HTTP 200

# qcluster.log delta:
18:27:59 [Q] INFO Process-1:7 processing [q1_ingestion_trace.pdf]
[2026-07-13 18:28:00,115] [ERROR] [paperless.consumer] Not consuming q1_ingestion_trace.pdf: It is a duplicate.
18:28:00 [Q] INFO Process-1:7 stopped doing work
18:28:00 [Q] ERROR Failed [q1_ingestion_trace.pdf] - q1_ingestion_trace.pdf: Not consuming q1_ingestion_trace.pdf: It is a duplicate. : Traceback (most recent call last):
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
documents.consumer.ConsumerError: q1_ingestion_trace.pdf: Not consuming q1_ingestion_trace.pdf: It is a duplicate.

# paperless.log delta:
[2026-07-13 18:28:00,115] [ERROR] [paperless.consumer] Not consuming q1_ingestion_trace.pdf: It is a duplicate.
```

**Cause → effect:** `pre_check_duplicate()` (`consumer.py:102`) computes the file's MD5
(`hashlib.md5(...)`, `:104`) and queries `Document.objects.filter(Q(checksum=checksum) |
Q(archive_checksum=checksum))` (`:106`); the match makes it call `_fail(...)` (`:110`), which raises
`ConsumerError` (`:81`). django-q records the task as `Failed [...]`. This is the **canonical
sequential** rejection path (the pre-check catches the duplicate *before* any DB write). There is also a
distinct **concurrent-race** variant: if two identical files are consumed simultaneously, both may pass
`pre_check_duplicate` before either commits, and the second `Document.objects.create(...)`
(`consumer.py:398`, inside `transaction.atomic()` `:298`) then trips the database's uniqueness on
`documents_document.checksum` — surfacing as `UNIQUE constraint failed: documents_document.checksum`
rather than the friendly message above (**source-derived** — the race was not forced at runtime; the
sequential path shown here is the one an ordinary re-upload hits).

**Q1 answer in one sentence.** A REST upload (gunicorn) or a consume-directory drop
(`document_consumer`) enqueues `documents.tasks.consume_file` to Redis; the `qcluster` django-q worker
runs `Consumer.try_consume_file()`, which logs the ordered `paperless.consumer` sequence
(*Consuming → Detected mime type → Parser → Parsing → OCR/thumbnail → classifier check → Saving record
to database → consumption finished*), stores the `Document` transactionally, fans out to six
`paperless.handlers` signal handlers, and moves/renames the files — after which the worker logs
`Processed [q1_ingestion_trace.pdf]`.


---

## 4. Q2 — Classifier retraining

> *"I also want to upload a few more documents and watch how the system reacts after each one. Does
> the machine learning classifier retrain automatically on every upload, or only under certain
> conditions, and what log messages make it clear when training is happening versus when the
> classifier stays idle?"*

**Short answer:** No — the classifier does **not** retrain on every upload. **Consumption ≠ training.**
Training is a **separate, scheduled (hourly), doubly-conditional** task. It runs only when (a) at least
one Tag / DocumentType / Correspondent uses the `MATCH_AUTO` algorithm, **and** (b) the training data
has actually changed since the last run. The distinguishing log lines are
`Saving updated classifier model to …` (trained) versus `Training data unchanged.` (idle).

### 4.1 Consumption never trains the classifier (upload ≠ train)

To test the "on every upload" hypothesis directly, several **additional** distinct PDFs were submitted
through the canonical REST entry point, and after **each** upload two things were checked: the bounded
`paperless.log` delta for that upload (grepped for any training/task line), and whether the model file
yet exists. The harness (run on the Docker host; it reuses the `make_pdf.py` helper whose source is
inlined in §3.2) and its complete, unedited output:

```text
$ TOKEN=$(cat /tmp/pngx-investigation/evidence/.drf_token)
$ PLOG=/app/data/log/paperless.log ; MODEL=/app/data/classification_model.pickle
$ for n in 2 3 4; do
>   docker exec -u 1000:1000 pngx bash -lc "python3 /tmp/make_pdf.py /tmp/pngx-test/q2_doc_${n}.pdf 'Q2 Multi-Upload Doc ${n}'"
>   docker cp pngx:/tmp/pngx-test/q2_doc_${n}.pdf /tmp/pngx-investigation/q2_doc_${n}.pdf
>   POFF=$(docker exec pngx bash -lc "wc -c < $PLOG")                          # byte offset BEFORE
>   curl -s -w " [HTTP %{http_code}]" -H "Authorization: Token $TOKEN" \
>        -F "document=@/tmp/pngx-investigation/q2_doc_${n}.pdf" \
>        http://127.0.0.1:8000/api/documents/post_document/                    # REAL entry point
>   # (wait until "consumption finished" appears in the delta, then:)
>   docker exec pngx bash -lc "tail -c +$((POFF+1)) $PLOG" \
>        | grep -E "paperless.tasks|Saving updated classifier|Training data unchanged|Gathering data"
>   docker exec pngx bash -lc "test -f $MODEL && echo EXISTS || echo ABSENT"
> done

########## UPLOAD #2 ##########
REST response: "OK" [HTTP 200]
--- paperless.log delta grep for training/tasks (expect NONE) ---
    (NONE — no training/classifier-task line in this upload's delta)
--- model file after upload #2: ABSENT

########## UPLOAD #3 ##########
REST response: "OK" [HTTP 200]
--- paperless.log delta grep for training/tasks (expect NONE) ---
    (NONE — no training/classifier-task line in this upload's delta)
--- model file after upload #3: ABSENT

########## UPLOAD #4 ##########
REST response: "OK" [HTTP 200]
--- paperless.log delta grep for training/tasks (expect NONE) ---
    (NONE — no training/classifier-task line in this upload's delta)
--- model file after upload #4: ABSENT
```

*(The `##########`, `---`, and `(NONE …)` lines are the harness's own `echo`/`tee` annotations; the
`"OK" [HTTP 200]` and `ABSENT` tokens are the live REST response and file-test result.)* Across three
additional uploads (documents `pk=2,3,4`, on top of the `pk=1` document from §3), **not one** produced
a `paperless.tasks` or `Gathering data` line, and the model file remained **ABSENT** after every one.

The single classifier-related line that *does* appear during a consumption is a per-document
*classification attempt*, not training. Filtering one consumption's complete `paperless.log` delta to
classifier lines shows exactly one:

```text
$ docker exec pngx bash -lc "tail -c +$((POFF+1)) $PLOG" | grep -i classif
    [2026-07-13 18:37:50,345] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```

**Cause → effect:** during consumption the consumer calls `load_classifier()`
(`src/documents/consumer.py:292` → `src/documents/classifier.py:30`) to *suggest* a
correspondent/type/tags for the incoming document; because no model file exists it logs
"…model does not exist (yet)…" (`classifier.py:32-36`) and returns `None`. That is a read/suggest step,
not training. Training is an entirely separate task, `documents.tasks.train_classifier`
(`src/documents/tasks.py:48`), which `consume_file` (`src/documents/tasks.py:184`, calling
`try_consume_file` at `:236`) **never** invokes. Therefore uploading documents — one or many — cannot
by itself produce a training log or a model file.

### 4.2 The training task and its (hourly) schedule

`train_classifier()` is registered to run **hourly** by
`src/documents/migrations/1001_auto_20201109_1636.py:10-14`
(`schedule("documents.tasks.train_classifier", …, schedule_type=Schedule.HOURLY)`), which is the
`id=1 … type=H` row shown in §2. To exercise the *same code path* deterministically (rather than
waiting for the top of the hour), the investigation used the canonical manual trigger,
`manage.py document_create_classifier`, whose `handle()` (`document_create_classifier.py:19`) calls the
identical `train_classifier()` (`:20`). This is the real training entry point, not a bypass.

The full body of the task (`src/documents/tasks.py:48-72`) defines exactly the branches exercised below:

```python
def train_classifier():                                                  # tasks.py:48
    if (
        not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
        and not DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
        and not Correspondent.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    ):
        return                                                           # tasks.py:55  (GUARD)
    classifier = load_classifier()                                       # tasks.py:57
    if not classifier:
        classifier = DocumentClassifier()
    try:
        if classifier.train():                                           # tasks.py:63
            logger.info(
                "Saving updated classifier model to {}...".format(settings.MODEL_FILE),  # tasks.py:65
            )
            classifier.save()                                            # tasks.py:67
        else:
            logger.debug("Training data unchanged.")                     # tasks.py:69  (IDLE)
    except Exception as e:
        logger.warning("Classifier error: " + str(e))                    # tasks.py:72  (ERROR)
```

### 4.3 All four outcomes, captured live

`train_classifier()` has exactly four possible outcomes; **all four were exercised at runtime** through
the real code path and are shown below with complete, unedited output.

**How these were captured (honest chronology).** On the fresh stack the branches were driven in this
order: the **TRAINED** run first (after configuring one `MATCH_AUTO` correspondent), then three **IDLE**
runs over unchanged data, then the **GUARD** branch (by temporarily removing the `MATCH_AUTO` flag),
then the **ERROR** branch (by temporarily excluding every document from the training set). Two facts
make each branch independent of capture order: the guard short-circuits *before* `load_classifier()`
(`tasks.py:55`), and the error raises *before* the hash gate (`classifier.py:159`, ahead of
`:161-164`) — so a pre-existing model file cannot change either outcome's log output. Every state change
used to reach the guard and error branches was **non-destructive and reversed** (no document was ever
deleted); see the note in each branch. All captures use the byte-offset delta method from §3.3 (`wc -c`
before, `tail -c +OFFSET` after) for exact, complete new bytes.

**Branch — TRAINED.** A `MATCH_AUTO` correspondent is created and assigned to one document. This ORM
setup is a **non-canonical precondition** (it stands in for the point-and-click a user performs in the
web UI); the *training trigger itself* remains the canonical task. Then the canonical manual trigger is
run:

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "\
from documents.models import Correspondent, Document, MatchingModel; \
c,created=Correspondent.objects.get_or_create(name='ACME Corporation', defaults={'matching_algorithm':MatchingModel.MATCH_AUTO,'match':'acme'}); \
print('Correspondent:', repr(c.name), 'pk=', c.pk, 'matching_algorithm=', c.matching_algorithm, '(6=MATCH_AUTO)'); \
d=Document.objects.order_by('pk').first(); d.correspondent=c; d.save(); print('Assigned correspondent to Document pk=', d.pk)"
Correspondent: 'ACME Corporation' pk= 2 matching_algorithm= 6 (6=MATCH_AUTO)
Assigned correspondent to Document pk= 1

$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py document_create_classifier   # canonical trigger
[2026-07-13 18:39:31,036] [INFO] [paperless.tasks] Saving updated classifier model to /app/src/../data/classification_model.pickle...
```

Full `paperless.log` (DEBUG) delta for this run:

```text
[2026-07-13 18:39:30,537] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:39:30,538] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 18:39:30,543] [DEBUG] [paperless.classifier] 5 documents, 0 tag(s), 1 correspondent(s), 0 document type(s).
[2026-07-13 18:39:30,960] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-13 18:39:30,961] [DEBUG] [paperless.classifier] There are no tags. Not training tags classifier.
[2026-07-13 18:39:30,962] [DEBUG] [paperless.classifier] Training correspondent classifier...
[2026-07-13 18:39:31,035] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-13 18:39:31,036] [INFO] [paperless.tasks] Saving updated classifier model to /app/src/../data/classification_model.pickle...
```

The model file appears immediately afterward:

```text
$ docker exec pngx ls -l /app/data/classification_model.pickle
-rw-r--r-- 1 testuser testuser 159935 Jul 13 18:39 /app/data/classification_model.pickle
```

**Cause → effect:** the guard passes (one `MATCH_AUTO` correspondent). `load_classifier()` finds no
model file yet and returns `None` (hence the first DEBUG line, `classifier.py:32-36`), so
`train_classifier` constructs a fresh `DocumentClassifier` whose `self.data_hash = None`
(`classifier.py:68`). `train()` gathers the 5 non-inbox documents and — because there is 1 correspondent
but 0 auto tags/types — trains only the correspondent sub-classifier (`classifier.py:226`; the tag/type
branches log "There are no …"). This is the **first** run, so the hash gate `if self.data_hash and …`
(`classifier.py:163-164`) is `False` *regardless of the data* (see §4.4); `train()` returns `True`,
`tasks.py:63` takes the `if` branch, logs the **INFO** line at `tasks.py:64-65`, and `classifier.save()`
(`:67`) writes the model — a plain pickle of 159,935 bytes owned by the non-root `testuser` (UID 1000).

**Branch — IDLE / SKIPPED (stable across 3 unchanged-data runs).** With nothing changed since the
TRAINED run, the canonical trigger was run three more times:

```text
$ MODEL=/app/data/classification_model.pickle
$ docker exec pngx md5sum $MODEL          # before the idle runs
7701284ace3065d7ac4b4eddcaedfbe0  /app/data/classification_model.pickle

----- IDLE RUN #1 -----   ($ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py document_create_classifier)
  console (INFO): (empty)
  paperless.log delta:
    [2026-07-13 18:39:58,855] [DEBUG] [paperless.classifier] Gathering data from database...
    [2026-07-13 18:39:58,861] [DEBUG] [paperless.tasks] Training data unchanged.
----- IDLE RUN #2 -----
  console (INFO): (empty)
  paperless.log delta:
    [2026-07-13 18:39:59,458] [DEBUG] [paperless.management.consumer] Not consuming file /app/src/../consume/__paperless_write_test_2112__: File has moved.
    [2026-07-13 18:40:01,024] [DEBUG] [paperless.classifier] Gathering data from database...
    [2026-07-13 18:40:01,030] [DEBUG] [paperless.tasks] Training data unchanged.
----- IDLE RUN #3 -----
  console (INFO): (empty)
  paperless.log delta:
    [2026-07-13 18:40:01,659] [DEBUG] [paperless.management.consumer] Not consuming file /app/src/../consume/__paperless_write_test_2268__: File has moved.
    [2026-07-13 18:40:03,306] [DEBUG] [paperless.classifier] Gathering data from database...
    [2026-07-13 18:40:03,311] [DEBUG] [paperless.tasks] Training data unchanged.

$ docker exec pngx md5sum $MODEL          # after the idle runs — identical, model not rewritten
7701284ace3065d7ac4b4eddcaedfbe0  /app/data/classification_model.pickle
```

The `console` is empty because `Training data unchanged.` is a **DEBUG** line while the console handler
is INFO (`src/paperless/settings.py:388`, `"level": "DEBUG" if DEBUG else "INFO"` with `DEBUG` defaulting
to `NO` at `:50`); it is nonetheless visible in the canonical `/app/data/log/paperless.log` because the
`paperless` logger writes to the file handler at DEBUG (`src/paperless/settings.py:409`) — **no
configuration change was needed** to observe it. The model's MD5 is **identical** before and after all
three runs, confirming a skipped run does not rewrite the model.

*Attribution note — the interleaved `__paperless_write_test_NNNN__` lines.* These are **not** emitted by
the training task. Each `manage.py` invocation first runs Django's system checks, and `path_check()`
writes a probe file named `__paperless_write_test_{os.getpid()}__` into the consume directory
(`src/paperless/checks.py:29`) to verify it is writable, then deletes it. The separate
`document_consumer` watcher notices the file appear-then-vanish and logs
`Not consuming file …: File has moved.` (`src/documents/management/commands/document_consumer.py:51`).
`NNNN` is therefore the PID of that management-command process (here `2112`, `2268`); the lines are shown
unedited for fidelity and are incidental to training.

**Branch — GUARD (no log at all; bounded proof).** To reach the guard on the running stack, the ACME
correspondent's algorithm was temporarily flipped away from `MATCH_AUTO` (and restored immediately
after), so that **zero** `MATCH_AUTO` entities exist:

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "\
from documents.models import Tag,DocumentType,Correspondent,MatchingModel; \
c=Correspondent.objects.get(name='ACME Corporation'); c.matching_algorithm=MatchingModel.MATCH_ANY; c.save(); \
print('MATCH_AUTO entities now:', Tag.objects.filter(matching_algorithm=6).count()+DocumentType.objects.filter(matching_algorithm=6).count()+Correspondent.objects.filter(matching_algorithm=6).count())"
MATCH_AUTO entities now: 0

$ docker exec pngx bash -lc "wc -l < /app/data/log/paperless.log"     # line-count BEFORE
107
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py document_create_classifier
                                        # <- console: (empty)
$ docker exec pngx bash -lc "wc -l < /app/data/log/paperless.log"     # line-count AFTER
107
# byte-offset delta for the run:
                                        # <- (ZERO new bytes — no lines emitted at all)
```

**Cause → effect:** `MATCH_AUTO = 6` (`src/documents/models.py:26`); with the flag removed, the guard's
three `.exists()` checks are all false, so `train_classifier` `return`s at `tasks.py:55` — before
`load_classifier()`, before any `Gathering data` line, and before writing anything at all. The
line-count is unchanged (**107 → 107**) and the byte-offset delta is **empty**: bounded, definitive proof
that a stock install with no "Auto" rule logs *nothing* about training. The `MATCH_AUTO` flag was then
restored so the correspondent is auto-matching again.

**Branch — ERROR (`MATCH_AUTO` entity exists, but no training data).** This branch was reached
**without deleting anything**: a temporary inbox tag was applied to all documents so the training query
`…exclude(tags__is_inbox_tag=True)` (`classifier.py:125-127`) gathers **zero** documents. (The tag was
removed afterward and the restored state re-verified.)

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "\
from documents.models import Tag, Document; \
t,_=Tag.objects.get_or_create(name='__q2_tmp_inbox__', defaults={'is_inbox_tag':True}); \
[d.tags.add(t) for d in Document.objects.all()]; \
print('eligible (non-inbox) documents:', Document.objects.exclude(tags__is_inbox_tag=True).count())"
eligible (non-inbox) documents: 0

$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py document_create_classifier
[2026-07-13 18:43:15,779] [WARNING] [paperless.tasks] Classifier error: No training data available.
# paperless.log delta:
[2026-07-13 18:43:15,777] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 18:43:15,779] [WARNING] [paperless.tasks] Classifier error: No training data available.
```

**Cause → effect:** the guard passes (ACME is `MATCH_AUTO` again), so `train()` runs, but the
inbox-exclusion leaves it with no documents; `if not data:` raises
`ValueError("No training data available.")` (`src/documents/classifier.py:159`) — note this is *before*
the hash gate at `:161-164`, so a pre-existing model is irrelevant. The `except` at `tasks.py:71`
catches it and logs the WARNING at `tasks.py:72`. The model file was **not** modified (its MD5 stayed
`7701284a…`), and removing the temporary tag restored 5 eligible documents; a follow-up run then logged
`Training data unchanged.` again — confirming the state was fully restored.

**The `Gathering data from database…` line is not an outcome signal.** It appears in the TRAINED, IDLE
**and** ERROR deltas above because `train()` logs it at its *start* (`classifier.py:123`), before it
knows what will happen. The **outcome** is given solely by the terminal `paperless.tasks` line
(`Saving updated classifier model …` / `Training data unchanged.` / `Classifier error: …`) — or, in the
guard case, by the total absence of any line.

### 4.4 The SHA-1 hash gate (why "idle" happens)

`DocumentClassifier.train()` (`src/documents/classifier.py:115`) is what decides trained-vs-idle:

- it logs `Gathering data from database...` (`classifier.py:123`) and builds a SHA-1 over the training
  data — `m = hashlib.sha1()` (`classifier.py:124`) — iterating
  `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`classifier.py:125-127`);
- if there is no data it raises `ValueError("No training data available.")` (`classifier.py:159`);
- otherwise it computes `new_data_hash = m.digest()` (`classifier.py:161`) and applies the gate
  `if self.data_hash and new_data_hash == self.data_hash: return False` (`classifier.py:163-164`).

Because a freshly constructed classifier initializes `self.data_hash = None`
(`src/documents/classifier.py:68`), the gate is `False` on the **first** run with data, so it trains
and stores the new hash (`self.data_hash = new_data_hash`, `classifier.py:247`). On the **second** run
over identical data, the freshly loaded model carries that stored hash, `new_data_hash` matches, the
gate returns `False`, and the task logs `Training data unchanged.` **Cause → effect:** identical
training data ⇒ identical SHA-1 digest ⇒ `train()` returns `False` ⇒ the idle log line.

### 4.5 The inbox-exclusion edge case (specific to this commit)

The training query explicitly excludes inbox-tagged documents
(`exclude(tags__is_inbox_tag=True)`, `src/documents/classifier.py:125-127`). Critically, **at this
commit no default inbox tag is seeded**: `is_inbox_tag` exists only as a schema field added by
`src/documents/migrations/1000_update_paperless_all.py:34,36`, and there is no fixtures directory to
seed one:

```text
$ ls src/documents/fixtures/
ls: cannot access 'src/documents/fixtures/': No such file or directory
```

**Cause → effect:** with zero inbox tags (`Tag.objects.filter(is_inbox_tag=True).count()` → `0`, shown
in §3.5), a newly consumed document is **not** excluded from the training set. That is why the TRAINED
branch in §4.3 was reachable: all **five** freshly-consumed documents were eligible (none inbox-excluded)
once a `MATCH_AUTO` entity existed. It is also why the ERROR branch in §4.3 had to *manufacture*
inbox-exclusion (temporarily inbox-tagging every document) to drive the training set to zero.

### 4.6 Frequency and stability

Training was driven through the real `train_classifier()` code path **seven** times, all shown in §4.3:
one TRAINED run, three consecutive IDLE runs over unchanged data, one GUARD run (no `MATCH_AUTO`
entity), one ERROR run (a `MATCH_AUTO` entity but no eligible training data, reached by inbox-exclusion
— never by deleting documents), and one further IDLE run after the temporary state was restored.
Separately, §4.1 established that the document uploads (five in total, across §3 and §4.1) produced
**no** training at all, and §2/§4.2 showed the hourly scheduler autonomously firing the same task on the
fresh database. The IDLE outcome was **stable and reproducible** across every unchanged-data run — the
three consecutive runs in §4.3 plus the post-restore run — confirming the SHA-1 gate is deterministic.
This directly answers the "on every upload vs. only under certain conditions" question: **only under
certain conditions**, and on a **schedule** (hourly), never as a side effect of an upload.

### 4.7 Distinguishing log lines — trained vs. idle vs. error vs. guard

All four rows below were captured live in §4.3.

| Outcome | Log line (verbatim) | Logger | Level | Emitted when |
|---|---|---|---|---|
| **TRAINED** | `Saving updated classifier model to …` | `paperless.tasks` | INFO | `classifier.train()` returned `True` (`tasks.py:63-65`, then `save()` `:67`) |
| **IDLE / SKIPPED** | `Training data unchanged.` | `paperless.tasks` | DEBUG | SHA-1 hash gate hit (`tasks.py:69`; gate `classifier.py:163-164`) |
| **ERROR** | `Classifier error: …` | `paperless.tasks` | WARNING | `train()` raised, e.g. no data (`tasks.py:72`; `ValueError` `classifier.py:159`) |
| **(guard)** | *(no log at all)* | — | — | no `MATCH_AUTO` Tag/DocumentType/Correspondent (`tasks.py:49-55`) |

A `paperless.classifier` companion line, `Gathering data from database...` (`classifier.py:123`),
precedes the trained, idle **and** error outcomes alike — it marks that `train()` *started*, not what it
decided. The outcome is therefore read from the **terminal `paperless.tasks` line**, not from this
prefix: `Gathering …` → `Saving updated classifier model …` is a **trained** run; `Gathering …` →
`Training data unchanged.` is an **idle** run; `Gathering …` → `Classifier error: …` is an **error**
run; and *no line at all* is the **guard** case.

### 4.8 Version caveat

This is the `FORMAT_VERSION = 7` classifier (`src/documents/classifier.py:63`). The model written in
§4.3 is a **plain pickle** (159,935 bytes): `save()` (`src/documents/classifier.py:96-113`) simply
`pickle.dump`s the format version followed by the data hash, vectorizer, and the three sub-classifiers,
and `load()` (`:76-94`) `pickle.load`s them back after a bare version-integer check — there is **no HMAC
signature, no `StoragePath` classifier, and no NLTK stemming** at this commit. Newer paperless-ngx
releases add those; they are out of scope here.


---

## 5. Q3 — On-disk layout and database tables

> *"After a document finishes processing, I want to see where it ends up on disk, what directory
> structure and filename pattern paperless-ngx uses by default, and which database tables receive new
> rows as part of ingestion."*

### 5.1 The default media tree

Every consumed **OCR-processed (PDF) document** produces exactly **three** files, split across three
fixed sub-directories of the media tree. The count is **parser-conditional**: `originals/` and
`thumbnails/` are always written, but the `archive/` PDF/A is produced only when the parser performs OCR,
so a text-only document (no OCR) yields just **two** files — demonstrated at runtime under *"Conditional
archive"* below. All five documents in this investigation are PDFs, so each produced three files; listing
the tree after the five consumptions (pks 1–5):

```text
$ docker exec pngx bash -lc "find /app/media/documents -type f | sort"
/app/media/documents/archive/0000001.pdf
/app/media/documents/archive/0000002.pdf
/app/media/documents/archive/0000003.pdf
/app/media/documents/archive/0000004.pdf
/app/media/documents/archive/0000005.pdf
/app/media/documents/originals/0000001.pdf
/app/media/documents/originals/0000002.pdf
/app/media/documents/originals/0000003.pdf
/app/media/documents/originals/0000004.pdf
/app/media/documents/originals/0000005.pdf
/app/media/documents/thumbnails/0000001.png
/app/media/documents/thumbnails/0000002.png
/app/media/documents/thumbnails/0000003.png
/app/media/documents/thumbnails/0000004.png
/app/media/documents/thumbnails/0000005.png

$ for d in originals archive thumbnails; do echo "== /app/media/documents/$d =="; docker exec pngx ls -l /app/media/documents/$d; done
== /app/media/documents/originals ==
total 20
-rw-r--r-- 1 testuser testuser 1587 Jul 13 18:26 0000001.pdf
-rw-r--r-- 1 testuser testuser 1594 Jul 13 18:36 0000002.pdf
-rw-r--r-- 1 testuser testuser 1594 Jul 13 18:36 0000003.pdf
-rw-r--r-- 1 testuser testuser 1594 Jul 13 18:36 0000004.pdf
-rw-r--r-- 1 testuser testuser 1594 Jul 13 18:37 0000005.pdf
== /app/media/documents/archive ==
total 40
-rw-r--r-- 1 testuser testuser 7823 Jul 13 18:26 0000001.pdf
-rw-r--r-- 1 testuser testuser 7991 Jul 13 18:36 0000002.pdf
-rw-r--r-- 1 testuser testuser 8111 Jul 13 18:36 0000003.pdf
-rw-r--r-- 1 testuser testuser 8038 Jul 13 18:36 0000004.pdf
-rw-r--r-- 1 testuser testuser 8082 Jul 13 18:37 0000005.pdf
== /app/media/documents/thumbnails ==
total 60
-rw-r--r-- 1 testuser testuser 10313 Jul 13 18:26 0000001.png
-rw-r--r-- 1 testuser testuser 10643 Jul 13 18:36 0000002.png
-rw-r--r-- 1 testuser testuser 10665 Jul 13 18:36 0000003.png
-rw-r--r-- 1 testuser testuser 10624 Jul 13 18:36 0000004.png
-rw-r--r-- 1 testuser testuser 10659 Jul 13 18:37 0000005.png
```

**Cause → effect:** these three locations are the defaults defined in `src/paperless/settings.py`:
`ORIGINALS_DIR = MEDIA_ROOT/documents/originals` (`:62`), `ARCHIVE_DIR = MEDIA_ROOT/documents/archive`
(`:63`), and `THUMBNAIL_DIR = MEDIA_ROOT/documents/thumbnails` (`:64`). Inside a single
`with FileLock(settings.MEDIA_LOCK):` block (`consumer.py:315`), the consumer writes the raw uploaded
PDF to originals (`consumer.py:319`), the thumbnail PNG to thumbnails (`consumer.py:321-325`), and the
OCRmyPDF-produced **PDF/A** to archive (`consumer.py:327-337`). Taking `pk=1` as the example, the size
difference — original 1,587 bytes vs. archive 7,823 bytes — reflects that the archive is the OCR'd PDF/A
copy (with an embedded text layer and PDF/A metadata), not the original. All files are owned by the
non-root `testuser` (UID 1000), matching the services' run user. (The classifier model, when it exists,
lives *outside* this tree at `MODEL_FILE = DATA_DIR/classification_model.pickle`,
`src/paperless/settings.py:74`.)

**Conditional archive — a non-PDF (text) document produces only two files (captured).** The three-file
result above is specific to OCR-processed PDFs; the file count is **parser-conditional**. In the
consumer's placement block the original (`src/documents/consumer.py:319`) and the thumbnail
(`consumer.py:321-325`) are written **unconditionally**, but the archive copy is written **only inside a
guard** — `if archive_path and os.path.isfile(archive_path):` (`consumer.py:327`). Whether a parser
yields an `archive_path` is parser-specific: the base `DocumentParser` initialises
`self.archive_path = None` (`src/documents/parsers.py:295`); the raster/PDF parser overwrites it with the
OCRmyPDF-produced PDF/A, while the plain-text `TextDocumentParser` (`src/paperless_text/parsers.py`,
handles `.txt`/`.md`/`.csv`) never assigns it — it only implements `get_thumbnail()` and `parse()`. A
text document therefore keeps `archive_path = None`, the guard is skipped, no `archive/` file is written,
and `Document.has_archive_version` (`src/documents/models.py:238-239`,
`return self.archive_filename is not None`) is `False`.

This was verified at runtime in a separate clean stack (fresh database), consuming — through the **same**
canonical REST entry point used for the PDF — a PDF (→ `pk=1`) and then a plain-text `.txt` file
(→ `pk=2`), so the two parsers contrast side by side:

```text
$ TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/token/ \
             -d "username=admin&password=admin" \
         | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# (1) canonical REST upload of a PDF
$ curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Token $TOKEN" \
       -F "document=@qa_control.pdf" -F "title=QA Control PDF" \
       http://127.0.0.1:8000/api/documents/post_document/
"OK"
HTTP 200

# (2) canonical REST upload of a plain-text file (same endpoint)
$ curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Token $TOKEN" \
       -F "document=@qa_textonly.txt" -F "title=QA Text Only" \
       http://127.0.0.1:8000/api/documents/post_document/
"OK"
HTTP 200
```

The text document is dispatched to `TextDocumentParser`; its complete, ordered `paperless.log` window has
**no** `Calling OCRmyPDF` / archive-extraction line that the PDF path emits (contrast §3.3–§3.4) — its
only `Execute:` line is the thumbnail's `optipng`:

```text
$ docker exec pngx sed -n "/Consuming qa_textonly.txt/,/QA Text Only consumption finished/p" /app/data/log/paperless.log
[2026-07-13 21:28:03,532] [INFO] [paperless.consumer] Consuming qa_textonly.txt
[2026-07-13 21:28:03,537] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-13 21:28:03,541] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-13 21:28:03,555] [DEBUG] [paperless.consumer] Parsing qa_textonly.txt...
[2026-07-13 21:28:03,556] [DEBUG] [paperless.consumer] Generating thumbnail for qa_textonly.txt...
[2026-07-13 21:28:03,594] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/paperless/paperless-xhv3oj8o/thumb.png -out /tmp/paperless/paperless-xhv3oj8o/thumb_optipng.png
[2026-07-13 21:28:04,303] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 21:28:04,317] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 21:28:04,344] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-iqemmxvs
[2026-07-13 21:28:04,348] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/paperless/paperless-xhv3oj8o
[2026-07-13 21:28:04,349] [INFO] [paperless.consumer] Document 2026-07-13 QA Text Only consumption finished
```

The resulting database attributes and the on-disk file count differ by parser exactly as predicted — the
PDF has an archive and three files; the text document has none and two files:

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from documents.models import Document
for d in Document.objects.order_by('pk'):
    print(f'pk={d.pk}  mime_type={d.mime_type}  has_archive_version={d.has_archive_version}  filename={d.filename!r}  archive_filename={d.archive_filename!r}')"
pk=1  mime_type=application/pdf  has_archive_version=True  filename='0000001.pdf'  archive_filename='0000001.pdf'
pk=2  mime_type=text/plain  has_archive_version=False  filename='0000002.txt'  archive_filename=None

$ docker exec pngx bash -lc 'for f in 0000001 0000002; do echo "== $f =="; find /app/media/documents -type f -name "$f.*" | sort; done'
== 0000001 ==
/app/media/documents/archive/0000001.pdf
/app/media/documents/originals/0000001.pdf
/app/media/documents/thumbnails/0000001.png
== 0000002 ==
/app/media/documents/originals/0000002.txt
/app/media/documents/thumbnails/0000002.png
```

**Cause → effect:** the PDF (`pk=1`) took the OCR path, so its parser produced a PDF/A and the
`archive_path` guard (`consumer.py:327`) fired → **three** files. The text file (`pk=2`) took the
`TextDocumentParser` path, which leaves `archive_path = None`, so the guard was skipped → **two** files
(`originals/` + `thumbnails/`) with `has_archive_version = False`. The "three files" figure is therefore
the **PDF/OCR case**, not a universal invariant: non-OCR document types omit the `archive/` artifact and
produce two files. For the user's actual input (a test PDF) and all five of this investigation's own
consumptions (pks 1–5, all PDFs), the three-file layout shown above holds exactly.

### 5.2 The default filename pattern (zero-padded 7-digit id)

Each document's stored filename is its zero-padded, 7-digit primary key. A real read-only query over all
five documents shows the pattern exactly (this replaces an earlier elided snippet — the command below is
literal and runnable):

```text
$ docker exec -u 1000:1000 -w /app/src pngx python3 manage.py shell -c "
from documents.models import Document
for d in Document.objects.order_by('pk'):
    print(f'pk={d.pk}  filename={d.filename!r}  archive_filename={d.archive_filename!r}  storage_type={d.storage_type}')"
pk=1  filename='0000001.pdf'  archive_filename='0000001.pdf'  storage_type=unencrypted
pk=2  filename='0000002.pdf'  archive_filename='0000002.pdf'  storage_type=unencrypted
pk=3  filename='0000003.pdf'  archive_filename='0000003.pdf'  storage_type=unencrypted
pk=4  filename='0000004.pdf'  archive_filename='0000004.pdf'  storage_type=unencrypted
pk=5  filename='0000005.pdf'  archive_filename='0000005.pdf'  storage_type=unencrypted
```

**Cause → effect:** because `PAPERLESS_FILENAME_FORMAT` is unset
(`src/paperless/settings.py:584`, `PAPERLESS_FILENAME_FORMAT = os.getenv("PAPERLESS_FILENAME_FORMAT")`
→ `None`, shown in §2), `generate_filename()` (`src/documents/file_handling.py:128`) takes its default
branch: with no format string the intermediate `path` is empty, so `len(path) > 0` (`:190`) is false and
it returns `filename = f"{doc.pk:07}{counter_str}{filetype_str}"` (`src/documents/file_handling.py:193`).
For `pk = 1` with no collision (`counter_str = ""`) and a PDF (`filetype_str = ".pdf"`), this evaluates
to `0000001.pdf`; the sequence continues `0000002.pdf … 0000005.pdf` for the subsequent primary keys,
which is exactly what the listing in §5.1 shows.

**Collision handling (inferred).** When two documents would resolve to the same filename,
`generate_unique_filename()` (`src/documents/file_handling.py:81`) increments a counter and
`generate_filename()` appends it via `counter_str = f"_{counter:02}" if counter else ""`
(`src/documents/file_handling.py:186`), yielding `_01`, `_02`, etc. This is labeled *inferred*: under
the default pk-based scheme every id is already unique, so a collision is not naturally triggered by
normal ingestion; the behavior is read from the code and corroborated by the official docs (§6).

### 5.3 Which database tables receive new rows

To answer this exhaustively (rather than guessing at a few tables), row counts of **every** table in the
database were snapshotted immediately before and after a single REST ingestion, using a strictly
read-only helper. The helper opens the SQLite file with `mode=ro`, so it physically cannot create,
modify, or delete anything — the read-only claim is verifiable from its source:

```python
# db_rowcounts.py — temporary read-only helper (removed in §8). Full source, inlined:
import sqlite3, sys
# Canonical DB path resolved from settings NAME in §2: /app/data/db.sqlite3
DB = "/app/data/db.sqlite3"
# mode=ro => strictly read-only connection (cannot create/modify/delete anything).
con = sqlite3.connect(f"file:{DB}?mode=ro", uri=True)
cur = con.cursor()
cur.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
tables = [r[0] for r in cur.fetchall()]
for t in tables:
    cur.execute(f'SELECT COUNT(*) FROM "{t}"')
    print(f"{cur.fetchone()[0]:>8}  {t}")
con.close()
```

The complete **before** snapshot (pristine DB, no documents yet) and **after** snapshot (immediately
after the single REST ingestion of `q1_ingestion_trace.pdf`, pk=1) — every table, nothing omitted:

```text
$ docker exec pngx python3 /tmp/db_rowcounts.py        # BEFORE
       0  auth_group
       0  auth_group_permissions
      88  auth_permission
       2  auth_user
       0  auth_user_groups
       0  auth_user_user_permissions
       0  authtoken_token
       0  django_admin_log
      22  django_content_type
      92  django_migrations
       0  django_q_ormq
       4  django_q_schedule
       5  django_q_task
       0  django_session
       0  documents_correspondent
       0  documents_document
       0  documents_document_tags
       0  documents_documenttype
       0  documents_log
       0  documents_savedview
       0  documents_savedviewfilterrule
       0  documents_tag
       0  paperless_mail_mailaccount
       0  paperless_mail_mailrule
       0  paperless_mail_mailrule_assign_tags
      16  sqlite_sequence

$ docker exec pngx python3 /tmp/db_rowcounts.py        # AFTER
       0  auth_group
       0  auth_group_permissions
      88  auth_permission
       2  auth_user
       0  auth_user_groups
       0  auth_user_user_permissions
       1  authtoken_token
       1  django_admin_log
      22  django_content_type
      92  django_migrations
       0  django_q_ormq
       4  django_q_schedule
       6  django_q_task
       0  django_session
       0  documents_correspondent
       1  documents_document
       0  documents_document_tags
       0  documents_documenttype
       0  documents_log
       0  documents_savedview
       0  documents_savedviewfilterrule
       0  documents_tag
       0  paperless_mail_mailaccount
       0  paperless_mail_mailrule
       0  paperless_mail_mailrule_assign_tags
      16  sqlite_sequence
```

Diffing the two snapshots yields exactly **four** tables with a higher count — but one of them is an
artifact of the investigation, not of ingestion:

```text
changed tables (after − before):
  authtoken_token       before=0 after=1 delta=+1
  django_admin_log      before=0 after=1 delta=+1
  django_q_task         before=5 after=6 delta=+1
  documents_document    before=0 after=1 delta=+1
```

**Honest disclosure:** `authtoken_token +1` is **not** caused by ingestion — it is the DRF auth token
created by the investigation's own `POST /api/token/` call (§3.2) to authenticate the upload. Excluding
that, **three** tables gain a row as a direct result of one ingestion: `documents_document`,
`django_admin_log`, and `django_q_task`.

The exact new rows in those three tables (queried with the same `mode=ro` connection):

```text
--- documents_document WHERE id=1 ---
                    id = 1
                 title = 'Q1 Ingestion Trace'
              filename = '0000001.pdf'
      archive_filename = '0000001.pdf'
              checksum = '4fa09dfa80cb242b08018ee91a96ca96'
      archive_checksum = '6e3fb715c5a5ae7cae666e050d03c25f'
          storage_type = 'unencrypted'
             mime_type = 'application/pdf'
      correspondent_id = 2
      document_type_id = None
               created = '2026-07-13 18:26:00.069640'
                 added = '2026-07-13 18:26:02.556712'
--- django_admin_log WHERE object_id='1' ---
           action_time = '2026-07-13 18:26:02.562000'
               user_id = 1
       content_type_id = 6
             object_id = '1'
           object_repr = '2026-07-13 Q1 Ingestion Trace'
           action_flag = 1
        change_message = ''
--- auth_user (to resolve user_id=1) ---
    user id=1 username='consumer'
    user id=2 username='admin'
--- django_q_task WHERE id='16bd776d8f844dc8a3db53a87ccbc853' ---
                    id = '16bd776d8f844dc8a3db53a87ccbc853'
                  name = 'q1_ingestion_trace.pdf'
                  func = 'documents.tasks.consume_file'
               success = 1
               started = '2026-07-13 18:26:00.071157'
               stopped = '2026-07-13 18:26:02.608841'
```

*Caveat on `documents_document.correspondent_id`:* the value `2` shown above is **not** what ingestion
wrote — at consumption time this column was `NULL`. It was set to the ACME correspondent (pk=2) later,
during the Q2 TRAINED-branch setup (§4.3). Every other field above (`filename`, both checksums,
`storage_type`, `mime_type`, `created`, `added`) is exactly as ingestion persisted it.

| Table | Δ | The new row | Cause (code) |
|---|---|---|---|
| `documents_document` | +1 | the `Document` record (`id=1`, `filename=0000001.pdf`, checksums, `storage_type=unencrypted`) | `_store()` creates it inside `transaction.atomic()` (`consumer.py:298`) |
| `django_admin_log` | +1 | an admin `LogEntry`, `action_flag=1` (ADDITION), `object_repr='2026-07-13 Q1 Ingestion Trace'`, authored by the `consumer` user (`user_id=1`) | `set_log_entry` handler (`handlers.py:413`) fetches `User…username="consumer"` (`:416`) then `LogEntry.objects.create(...)` (`:418`) |
| `django_q_task` | +1 | the finished `consume_file` task result (`func=documents.tasks.consume_file`, `success=1`) | the django-q worker persists each finished task in its ORM result backend, sized by `Q_CLUSTER` (`settings.py:449-456`) |

**The relevant zero-change candidates** (present in both snapshots at the same count, so *not* written by
ingestion) confirm the answer is complete rather than cherry-picked:

- `documents_document_tags` `0 → 0` — the Document↔Tag M2M (`src/documents/models.py:128`) gains nothing
  because on a fresh install there is no matching or inbox tag to apply (consistent with §3.5 and §4.5).
- `documents_correspondent` `0 → 0`, `documents_documenttype` `0 → 0`, `documents_tag` `0 → 0`,
  `documents_log` `0 → 0` — ingestion creates no matcher entities or document-log rows on its own.
- `django_q_ormq` `0 → 0` — the django-q *broker queue* table stays empty because the queued task is
  delivered through **Redis**, not the ORM broker, in this configuration.
- `django_q_schedule` `4 → 4` — the four periodic schedules (including hourly `train_classifier`, §2)
  are unchanged by a document upload.
- `sqlite_sequence` `16 → 16` — no *row* is added; only the stored `documents_document` counter *value*
  increments (this is how the next pk is allocated). A value change is not a row-count change, so it does
  not surface in the row-count diff (inferred from SQLite autoincrement semantics).

**Cause → effect:** one ingestion persists its `Document` row inside `transaction.atomic()`
(`consumer.py:298`), the finished-signal `set_log_entry` handler writes the admin `LogEntry` as the
`consumer` user (`handlers.py:413-418`), and the django-q worker records the completed `consume_file`
task — three rows in three tables. The database is the default SQLite file at `DATA_DIR/db.sqlite3`
(`src/paperless/settings.py:300`).


---

## 6. Corroborating research

The following external sources **validate** — they do not replace — the runtime evidence captured in
§§2–5. Every behavioral conclusion in this document was observed directly at runtime; the sources below
are cited only to confirm that what was observed matches the project's own documentation and the
experience of other users from this commit's era. Where a source describes a *newer* release, the
difference is flagged explicitly against commit `542221a38dff`.

### 6.1 Official documentation

- **Hourly auto-retraining.** The official *Advanced Topics* page states that Paperless checks for
  changes and retrains automatically `(default: once each hour)`. This corroborates the observed
  `django_q_schedule` row (`id=1 … func=documents.tasks.train_classifier … type=H`, §2) and the
  "consumption ≠ training" result (§4.1).
  Source: *Advanced Topics — Paperless-ngx* — <https://docs.paperless-ngx.com/advanced_usage/>
- **Inbox exclusion from the training set.** The same page states that the Auto matching algorithm only
  takes into account documents that are **not** placed in the inbox. This corroborates the
  `.exclude(tags__is_inbox_tag=True)` query used by `DocumentClassifier.train()`
  (`src/documents/classifier.py:125-127`) and the manufactured edge case in §4.5.
  Source: *Advanced Topics — Paperless-ngx* — <https://docs.paperless-ngx.com/advanced_usage/>
- **Default filename = internal document id.** The *FAQ* states that "by default, paperless uses the
  internal ID of each document as its filename." This corroborates the observed `0000001.pdf` → `pk=1`
  mapping (§5.2) and the default branch of `generate_filename()`
  (`src/documents/file_handling.py:190-193`).
  Source: *FAQs — Paperless-ngx* — <https://docs.paperless-ngx.com/faq/>
- **Archive PDF/A stored alongside the unmodified original.** The *Administration* page states that
  Paperless stores archived PDF/A documents "alongside your original documents," derived from originals
  that are always kept unmodified. This corroborates the `originals/` + `archive/` split observed in
  §5.1.
  Source: *Administration — Paperless-ngx* — <https://docs.paperless-ngx.com/administration/>

### 6.2 Community corroboration (subordinate to the runtime evidence)

Real user logs from this commit's era (2022–2023) independently show the **exact** log strings this
investigation captured at runtime in §4.3 — the same logger namespaces (`paperless.classifier`,
`paperless.tasks`) and the same messages — confirming the observed behavior is canonical and not an
artifact of this environment. These threads are corroboration only; the primary evidence remains the
runtime capture in §§3–5.

- **The "idle / unchanged" skip sequence** — `[DEBUG] [paperless.classifier] Gathering data from
  database...` immediately followed by `[DEBUG] [paperless.tasks] Training data unchanged.` — appears in
  a real user's log in *Discussion #1809* (Oct 2022), which also shows the per-consumption
  `[DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing
  automatic matching.` line observed in §4.1; the same skip pair also appears in *Discussion #2472*
  (Jan 2023).
  Sources: *Discussion #1809 — "Document classification model does not exist (yet), not performing
  automatic matching."* — <https://github.com/paperless-ngx/paperless-ngx/discussions/1809> ·
  *Discussion #2472 — "Documents not consumed, stay queued, logs not helpful"* —
  <https://github.com/paperless-ngx/paperless-ngx/discussions/2472>
- **The "trained / saving" sequence** — the classifier's `N documents, N tag(s), …` →
  `Vectorizing data...` → `Training … classifier...` steps followed by
  `[INFO] [paperless.tasks] Saving updated classifier model to …classification_model.pickle...`, and
  then hourly `Gathering data from database...` → `Training data unchanged.` skips — appears in a real
  user's log in *Issue #3531* (Jun 2023). This corroborates both the trained-branch and the
  idle-branch sequences captured in §4.3.
  Source: *Issue #3531 — "[BUG] Documents not being processed anymore"* —
  <https://github.com/paperless-ngx/paperless-ngx/issues/3531>
- **`_01` / `_02` filename de-duplication.** The *paperless-ng 1.5.0* changelog documents that the
  filename formatter will "append `_01`, `_02`, etc when it detects duplicate filenames." This
  corroborates the **inferred** collision-suffix behavior in §5.2 (`generate_unique_filename()`
  `src/documents/file_handling.py:81`; `counter_str` `:186`). Note the same changelog entry says the
  formatter no longer embeds the document id — but that applies to the *custom* `PAPERLESS_FILENAME_FORMAT`
  scheme; with the format **unset** (the default at this commit) the filename remains the zero-padded
  pk, exactly as the FAQ states and as observed in §5.2.
  Source: *Changelog — Paperless-ng 1.5.0* —
  <https://paperless-ngx.readthedocs.io/en/ng-1.5.0/changelog.html>
- **OCRmyPDF → PDF/A alongside originals.** The same changelog's OCRmyPDF-integration entry documents
  that OCRmyPDF produces archived PDF/A versions and that Paperless stores those archived versions
  alongside the originals — corroborating the `archive/` tree observed beside `originals/` in §5.1.
  Source: *Changelog — Paperless-ng 1.5.0* —
  <https://paperless-ngx.readthedocs.io/en/ng-1.5.0/changelog.html>

### 6.3 Version boundary (reaffirmed)

Several search results describe features of *newer* paperless-ngx that **do not exist at commit
`542221a38dff`** and were therefore **not** used as evidence for any claim in this document:

- **Newer scheduling mechanism.** Current docs expose a crontab-style
  `PAPERLESS_CLASSIFIER_TRAINING_SCHEDULE` (default `5 */1 * * *`). At this commit there is no such
  setting: the hourly cadence comes from the django-q `Schedule.HOURLY` row registered by migration
  `src/documents/migrations/1001_auto_20201109_1636.py` (§4.2), which is precisely the
  `django_q_schedule` row observed in §2. *(Source: Configuration — Paperless-ngx —
  <https://docs.paperless-ngx.com/configuration/> — describes a newer release.)*
- **Newer classifier internals.** Third-party write-ups describe an HMAC-signed model, a `StoragePath`
  classifier, and NLTK stemming. None are present here: the classifier is `FORMAT_VERSION = 7`
  (`src/documents/classifier.py:63`) with plain `pickle` load/save (`:76-113`), no HMAC, no
  `StoragePath` classifier, and no NLTK stemming. *(Source: DeepWiki — Document Classification —
  <https://deepwiki.com/paperless-ngx/paperless-ngx/4.3-document-classification> — describes a newer
  release.)*
- **A newer regression.** A 2024 discussion (#8132, v2.13.x) reports the classifier retraining *every*
  hour even with unchanged data — the opposite of the SHA-1-gated skip observed here (§4.3/§4.4). That
  is a later-version behavior change and is out of scope for this commit.
- **Task system.** The task and scheduler layer at this commit is **django-q** (`qcluster`), never
  Celery.

Because online tracebacks for paperless reference *different* source line numbers than this commit,
**every** `file:line` citation in this document was verified live against the source at `542221a38dff`
rather than copied from documentation, and the
`Gathering data from database... → Training data unchanged.` skip was observed directly at runtime
(§4.3) — so the community threads above serve strictly as corroboration, never as the primary evidence.

---

## 7. Coverage confirmation

Every sub-question and named item from the prompt is answered below, mapped to the section that answers
it and to the **specific evidence** behind it. The final column states the **evidence type** honestly,
so nothing is over-claimed:

- `captured` = the exact command and its complete, unedited runtime output are shown;
- `bounded` = proven by a runtime measurement (line counts / byte-offset deltas) rather than a raw block —
  used only where the correct behavior is the *absence* of a log line, so there is no block to show;
- `source-derived` = read from the code at this commit and labeled as such in-line (not observed in a log);
- `non-canonical` = a precondition created via the ORM rather than the user-facing path (labeled in-line);
- `inferred` = not exercised at runtime, deduced from code + documentation.

| # | Sub-ask | Answered in | Evidence behind it | Evidence type |
|---|---|---|---|---|
| Q1a | Which services are involved | §3.1 | `/proc` process list + Redis `PING` (§2); per-line logger attribution (§3.4) | captured |
| Q1b | The ordered sequence of events in the logs | §3.3 + §3.4 | complete byte-offset deltas of `gunicorn.log`, `qcluster.log`, `paperless.log` | captured |
| Q1c | The silent (no-log) pipeline stages | §3.4 (silent-stage table) | derived from `consumer.py` (started signal, pre-consume, `get_text`, date parsing, shadow cleanup, post-consume) | source-derived |
| Q1d | Post-consumption fan-out (the six handlers) | §3.5 | `django_admin_log` row (user `consumer`), Whoosh index hit, inbox-tag count | captured |
| Q1e | Progress / status delivery | §3.1 diagram + §3.4 note | routed to the Channels `status_updates` group over WebSocket, not to the HTTP client | source-derived |
| Q1f | Duplicate re-upload behavior (sequential) | §3.6 | worker `Not consuming …: It is a duplicate.` + django-q task `Failed` | captured |
| Q1f′ | Concurrent-race duplicate variant | §3.6 | `UNIQUE constraint failed: documents_document.checksum` — not forced at runtime | source-derived |
| Q2a | Does it retrain on *every* upload? (no) | §4.1 | several REST uploads (`OK` / HTTP 200), each with a bounded `paperless.log` delta showing **no** training line and the model file **absent** | captured |
| Q2b | Only under certain conditions? | §4.2–§4.6 | hourly `django_q_schedule` row (§2); MATCH_AUTO guard; SHA-1 hash gate | captured |
| Q2c | Guard branch (no MATCH_AUTO entity present) | §4.3 Branch 1 | training task triggered with zero MATCH_AUTO entities; `paperless.log` line-count `107 → 107`, zero-byte delta (guard returns before emitting anything) | bounded |
| Q2d | "Training happening" log message | §4.3 Branch 2, §4.7 | `[INFO] [paperless.tasks] Saving updated classifier model to …` + 159 935-byte model file written | captured |
| Q2e | Training precondition (a MATCH_AUTO entity) | §4.3 Branch 2 | `Correspondent 'ACME Corporation'` (`MATCH_AUTO`) created via ORM and assigned to a document | non-canonical |
| Q2f | "Classifier idle" log message | §4.3 Branch 3, §4.7 | `[DEBUG] [paperless.tasks] Training data unchanged.`, stable across repeated runs (identical model MD5 `7701284a…`) | captured |
| Q2g | Error branch (MATCH_AUTO set but no eligible data) | §4.3 Branch 4 | `[WARNING] [paperless.tasks] Classifier error: No training data available.` — reached non-destructively via inbox exclusion, state restored | captured |
| Q3a | Where the document ends up on disk | §5.1 | `find` / `ls -l` of `/app/media/documents/{originals,archive,thumbnails}` | captured |
| Q3b | Default directory structure | §5.1 | the three fixed subdirectories; `originals/` + `thumbnails/` always populated, `archive/` only for OCR-processed (PDF) documents | captured |
| Q3c | Default filename pattern | §5.2 | ORM listing tying `pk=1…5` → `0000001.pdf … 0000005.pdf` | captured |
| Q3c′ | Filename collision suffix (`_01`, `_02`) | §5.2 | `counter_str` in `generate_unique_filename()`; not triggered under the default pk scheme | inferred |
| Q3d | Which DB tables receive new rows | §5.3 | exhaustive 26-table before/after snapshot via a read-only (`mode=ro`) helper, plus the exact changed rows | captured |

**Honest scope of the evidence.** All three questions are answered from runtime observation, and every
code-level claim carries a `file:line` citation verified against the source at commit `542221a38dff`.
Behavioral claims are backed by the exact command and its complete, unedited output **except** for the
items explicitly labeled otherwise, both in the table above and in-line where they appear, namely:

- the **guard** branch (Q2c) is proven by a *bounded* line-count / byte-delta rather than a raw block,
  because the correct behavior is that **no** log line is emitted — there is nothing to print;
- the **silent Q1 stages** (Q1c) and the **WebSocket / Channels progress delivery** (Q1e) are
  *source-derived* from `consumer.py` and the signal wiring, not read from a log line;
- the **training precondition** — the `ACME` `MATCH_AUTO` correspondent (Q2e) — is a *non-canonical*
  ORM setup used only to make the conditional training branch reachable, not the user-facing web path;
- the **concurrent-race** duplicate variant (Q1f′) is *source-derived*: only the sequential
  duplicate-rejection path was forced at runtime;
- the **filename collision suffix** (`_01`/`_02`, Q3c′) and the **`sqlite_sequence` value increment**
  are *inferred* from the code and corroborated by the docs in §6, not exercised at runtime.

Nothing outside these clearly labeled items is presented as observed without its captured command and
output.

---

## 8. Cleanup note

Cleanup was **executed and evidenced**, not merely asserted — the exact commands and their complete,
unedited output are shown below, all run from the Docker-in-Docker host. Because the `pngx` container
had **no bind mounts and no named volumes** (verified with
`docker inspect pngx --format '{{range .Mounts}}...{{end}}'`, which returned nothing), every piece of
runtime state — the SQLite database, the media tree, and the classifier model — lived inside the
container's writable layer and was destroyed entirely when the container was removed (§8.3).

### 8.1 Restore pre-test state and prove the schedule is quiet

The only persistent test entity capable of keeping the hourly scheduler active was the `MATCH_AUTO`
`Correspondent 'ACME Corporation'` created in §4.3. Removing it via the canonical ORM dropped the
MATCH_AUTO entity count to zero; a subsequent real training trigger then appended **zero bytes** to
`paperless.log` — proving the guard branch (`src/documents/tasks.py:49-55`) now returns before emitting
any log line, i.e. the schedule is quiet.

```console
$ docker exec -w /app/src pngx python3 manage.py shell -c \
    "from documents.models import Correspondent, DocumentType, Tag; \
     n,_=Correspondent.objects.filter(name='ACME Corporation').delete(); \
     print('deleted rows (correspondent + cascade M2M/FK unset):', n); \
     A=6; \
     print('MATCH_AUTO AFTER -> Correspondent:', Correspondent.objects.filter(matching_algorithm=A).count(), \
           'DocumentType:', DocumentType.objects.filter(matching_algorithm=A).count(), \
           'Tag:', Tag.objects.filter(matching_algorithm=A).count())"
deleted rows (correspondent + cascade M2M/FK unset): 1
MATCH_AUTO AFTER -> Correspondent: 0 DocumentType: 0 Tag: 0

$ docker exec pngx sh -c 'wc -c < /app/data/log/paperless.log'     # byte offset BEFORE trigger
15508
$ docker exec -w /app/src pngx python3 manage.py document_create_classifier
$ docker exec pngx sh -c 'wc -c < /app/data/log/paperless.log'     # byte offset AFTER trigger
15508
# offset before = 15508 ; offset after = 15508 ; delta = 0 bytes
# grep of the (empty) delta for any training signature:
#   (paperless.tasks | Saving updated classifier | Training data unchanged | Gathering data | Classifier error)
#   >>> NO training line emitted — the schedule is QUIET <<<
```

The generated model file was then removed, completing the pre-test restoration:

```console
$ docker exec pngx sh -c 'wc -c < /app/data/classification_model.pickle'   # before
159935
$ docker exec pngx rm -f /app/data/classification_model.pickle
$ docker exec pngx sh -c 'test -f /app/data/classification_model.pickle && echo yes || echo no'
no
```

### 8.2 Remove every investigation-owned scratch file (container + host)

```console
# --- container-internal scratch (test PDFs, helper scripts, the start-services copy,
#     and the consumer's residual upload temp) ---
$ docker exec pngx sh -c 'rm -rf /tmp/pngx-test; \
      rm -f /tmp/db_rowcounts.py /tmp/make_pdf.py /tmp/proc_snapshot.py /tmp/start-services.sh; \
      rm -rf /tmp/paperless'
# absence checks inside the container:
/tmp/pngx-test                        -> ABSENT
/tmp/*.py  (helper scripts)           -> ABSENT
/tmp/start-services.sh                -> ABSENT
/tmp/paperless  (paperless-upload-*)  -> ABSENT
find /tmp -name '*.pdf'               -> NONE
find /tmp -name 'paperless-upload-*'  -> NONE

# --- host-side scratch (scripts, evidence captures, test PDFs, section backups) ---
$ rm -rf /tmp/pngx-investigation
$ ls -la /tmp/pngx-investigation   -> ABSENT
$ ls -la /tmp/paperless            -> ABSENT
$ ls -la /tmp/pngx-run             -> ABSENT
```

The `/root/pngx-run/{bring-up.sh,start-services.sh}` scripts are the environment's own provisioning
scripts (created before this investigation, outside the repository); they are not investigation-owned
and were left untouched.

### 8.3 Dispose the runtime

```console
$ docker rm -f pngx pngx-redis
pngx
pngx-redis
$ docker network rm pngx-net
pngx-net
$ docker ps -a --format '{{.Names}}' | grep -iE 'pngx|paperless|redis'   -> NONE (all removed)
$ docker network ls --format '{{.Name}}' | grep -E 'pngx-net'            -> NONE (removed)
$ docker volume ls | grep -iE 'pngx|paperless'                           -> NONE (no residual volumes)
```

Because the container held no volumes, its removal wiped the database, media tree, and any generated
model in a single step — no runtime state survives.

### 8.4 Read-only proof — the repository is unchanged except the single deliverable

```console
$ git status --porcelain
 M blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --name-status 542221a38dff          # working tree vs the pristine source commit
A	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --stat 542221a38dff
 blitzy/documentation/paperless-ngx_542221a38dff.md | 1435 ++++++++++++++++++++
 1 file changed, 1435 insertions(+)
# NOTE (self-reference): this --stat was captured at the moment the cleanup evidence
# was gathered, when the document stood at 1435 lines — i.e. BEFORE this §8 cleanup
# section was itself appended. Re-running the command now reports a larger count (the
# file is longer by exactly the lines of this section); the insertion count is the one
# figure here that necessarily changes as this section is written and is therefore not
# re-captured. The invariant facts a reviewer relies on are unaffected: exactly ONE
# file changed, ADDED (status "A") relative to the pristine 542221a38dff baseline, with
# zero modifications to any pre-existing file — confirmed by the `git status --porcelain`
# and `git diff --name-status` output above, both of which are independent of length.

$ git status --porcelain --untracked-files=all | grep '^??'    # untracked files?
(no output — nothing untracked)

$ find . -not -path './.git/*' \( -name 'classification_model*' -o -name 'db.sqlite3' \
      -o -name 'q1_*.pdf' -o -name 'q2_doc_*.pdf' -o -name '*.heapsnapshot' \)
(no output — none found)

# runtime dirs are container-only and never existed in the repo tree:
media, data, documents/originals, src/media, src/data   -> all absent
```

A repository-wide search for `*.pickle` matches exactly one path —
`./src/documents/tests/data/model.pickle` — which is a **pre-existing tracked test fixture** shipped in
the source at commit `542221a38dff` (it appears **unchanged** in the baseline diff above, never as an
addition), **not** an artifact of this investigation. No generated `classification_model.pickle`,
database, media file, log, or test document was written into the repository.

**Conclusion.** The **only** change to the repository is this single new file,
`blitzy/documentation/paperless-ngx_542221a38dff.md` — added relative to the pristine source, as the
baseline `git diff --name-status 542221a38dff` proves. No existing source, configuration, dependency,
or test file was modified, added, or deleted; the runtime and every investigation-owned scratch artifact
have been disposed of and their absence verified. The source repository is left unchanged, exactly as
required by the read-only mandate of this investigation.
