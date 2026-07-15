# Asynchronous Background Processing During Document Ingestion in paperless-ngx

**Repository commit:** `542221a38dff` (full SHA `542221a38dff06361e07976452f9aea24d210542`).
**Subject:** how paperless-ngx performs asynchronous background work while a document is ingested,
observed on a running system and traced back to the exact source code responsible.

## 1. Framework: Django-Q 1.3.9 over a Redis broker — **not Celery**

Background processing in this version of paperless-ngx is powered by **Django-Q 1.3.9 running over a
Redis broker**. It is **not Celery.** This is the single most important fact for everything below, and
it is stated first deliberately: "async work" in a Django project is *commonly* assumed to be Celery,
and here that assumption is wrong. There is no `celery`, no `@shared_task`, no `celerybeat`, and no
`apply_async` anywhere in the ingestion path. The enqueue primitive is Django-Q's `async_task(...)`,
the worker is `python3 manage.py qcluster`, and task state lives in Django-Q's own `django_q_task`
table.

**Runtime proof — installed framework and versions** (match the `requirements.txt` pins exactly):

```console
$ docker exec pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django_q, django, redis, channels
print("django_q ", django_q.VERSION)
print("django   ", django.get_version())
print("redis    ", redis.__version__)
print("channels ", channels.__version__)
PY
django_q  (1, 3, 9)
django    4.0.4
redis     3.5.3
channels  3.0.4
```

**Runtime proof — the live broker is Redis:**

```console
$ docker exec pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.brokers import get_broker
b = get_broker()
print("broker class =", type(b).__module__ + "." + type(b).__name__)
print("broker info  =", b.info())
print("list_key     =", b.list_key)
PY
broker class = django_q.brokers.redis_broker.Redis
broker info  = Redis 7.4.9
list_key     = django_q:paperless:q
```

**Runtime proof — there is no Celery in the async ingestion source tree**, and the four import sites of
the real enqueue primitive (`async_task`) are exactly the ingestion/bulk entry points:

```console
$ docker exec pngx_qa bash -lc 'cd /app/src && \
    grep -rniE "celery|shared_task|apply_async|celerybeat" documents/ paperless/ paperless_mail/ | wc -l'
0

$ docker exec pngx_qa bash -lc 'cd /app/src && grep -rn "from django_q.tasks import async_task" documents/ paperless/ paperless_mail/'
documents/management/commands/document_consumer.py:13:from django_q.tasks import async_task
documents/views.py:28:from django_q.tasks import async_task
documents/bulk_edit.py:4:from django_q.tasks import async_task
paperless_mail/mail.py:11:from django_q.tasks import async_task
```

- The async engine is Django-Q: `django_q==1.3.9` in `requirements.txt`, and `"django_q"` is in
  `INSTALLED_APPS` at `src/paperless/settings.py:110`.
- The broker is Redis: the live broker object is `django_q.brokers.redis_broker.Redis`, configured by
  the `redis` key of `Q_CLUSTER` at `src/paperless/settings.py:456`
  (`"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")`).
- Celery is named here **only** to disambiguate terminology; it is not used and is not described below.

The end-to-end lifecycle, in one picture (every arrow is confirmed at runtime in the sections that
follow; the intermediate *pusher → internal queue → worker → result queue → monitor* hops are the
actual Django-Q 1.3.9 mechanics, corrected and demonstrated in [Q1](#3-q1--live-async-behavior),
[Q2](#4-q2--services-involved) and [Q4](#6-q4--waiting-vs-actively-processing)):

```
 enqueue site          Redis broker list          pusher proc       internal mp-queue      worker proc         result mp-queue     monitor proc
 (async_task, Q7)  --RPUSH-->  django_q:paperless:q  --BLPOP-->  task_queue.put()  -->  worker executes  --put-->  result_queue  --save-->  django_q_task row
                              (WAITING, Q3/Q4)                  (in cluster memory)      consume_file()             (in memory)            (DONE, Q5/Q6)
                                                                                              |
                                                                                              +-- group_send --> "status_updates" WebSocket (Q6)
```

## How to read this document (methodology & label conventions)

This document is **evidence-first**. Every behavioral claim is placed next to the exact command that
produced it and that command's *actual, complete output*, and is then tied to the responsible source
code by `file:line`. To keep the two kinds of statements honest, two labels are used consistently:

- **(inferred)** — a statement derived from *reading source code* rather than from observed output.
  Wherever practical the same statement is also confirmed at runtime, and the confirming observation is
  shown next to it.
- **(documentation-derived)** — a statement taken from the official Django-Q 1.3.x documentation
  (named precisely in the [Appendix](#appendix)). These are confirmed against the installed
  `django-q==1.3.9` source and/or runtime wherever possible.

Everything **not** carrying one of those two labels is backed by the adjacent observed output. Evidence
is shown complete for the signal in question; where a capture is long, the block states exactly what was
filtered and how (e.g. "state-change rows of the 0.25 s poll"), and the unfiltered source of that
capture is a file under `/tmp/obs` produced by a fully-shown command.

## Table of contents

1. [Framework: Django-Q, not Celery](#1-framework-django-q-139-over-a-redis-broker--not-celery)
2. [Environment & exact reproduction](#2-environment--exact-reproduction)
3. [Q1 — Live async behavior (end-to-end lifecycle)](#3-q1--live-async-behavior)
4. [Q2 — Services involved (process topology)](#4-q2--services-involved)
5. [Q3 — How a job appears when created](#5-q3--how-a-job-appears-when-created)
6. [Q4 — Waiting vs. actively processing](#6-q4--waiting-vs-actively-processing)
7. [Q5 — Where task state is stored](#7-q5--where-task-state-is-stored)
8. [Q6 — After-the-fact status](#8-q6--after-the-fact-status)
9. [Q7 — Enqueue origin in code](#9-q7--enqueue-origin-in-code)
10. [Scheduled / recurring jobs](#10-scheduled--recurring-jobs)
11. [Exhaustive condition coverage](#11-exhaustive-condition-coverage)
12. [Cleanup & repository integrity](#12-cleanup--repository-integrity)
13. [Appendix](#appendix)

---

## 2. Environment & exact reproduction

All observation was performed **inside a disposable Docker container** built from the canonical image
(the project targets `python:3.9-slim-bullseye`, `Dockerfile:18`). The plain host shell cannot run any
of this — it lacks the pinned dependencies — so every command below is executed inside the container
via `docker exec`. The commands in this section are the complete, self-contained recipe; anyone can
reproduce the entire investigation from them.

### 2.1 Disclosed environment posture (deviations from production, and why)

Three honest deviations from the production `docker/supervisord.conf` deployment were made, none of
which affects the asynchronous mechanism under study. They are disclosed here and never presented as
"production-exact":

1. **Least-privileged user.** Production drops each service to the unprivileged **`paperless`** user
   (`docker/supervisord.conf:12,21,30`). This image does **not** contain a `paperless` user; the only
   non-root account available is **`testuser`** (uid 1000). Therefore migrations and all three services
   are run as `testuser` — the least-privileged account available — never as root.
2. **No published host port.** The container is created with **no `-p` port mapping** and **no bind
   mounts** (`PortBindings={}`, `Mounts=[]`), so nothing is exposed to the host. The app's own
   `gunicorn.conf.py` binds `0.0.0.0:8000` *inside* the container, but with no port published that
   endpoint is reachable only within the container's network namespace (via `localhost`) and by
   `docker exec`. This is strictly safer than the production host-wide bind.
3. **Redis provenance.** The image does not bundle a Redis server, so Redis is run from the pinned
   `redis:7-alpine` image sharing the container's network namespace, which keeps the canonical
   `PAPERLESS_REDIS=redis://localhost:6379` address intact. The observed server is **Redis 7.4.9**; the
   Django-Q Redis broker uses only `RPUSH`/`BLPOP`/`LLEN` on a list, whose semantics are identical
   across Redis 6 and 7 (the broker source is shown in [Q4](#6-q4--waiting-vs-actively-processing)).

One **provenance note** (not a deviation — it makes the runtime *more* production-faithful): the
canonical image ships without three OS libraries that the project's own `Dockerfile` installs —
`libzbar0` (the shared library behind the `pyzbar` barcode reader), `poppler-utils`, and `pngquant`.
`libzbar0` is load-bearing for the async path: `documents/tasks.py:25` executes `from pyzbar import
pyzbar` at **module import time**, so a Django-Q worker that tries to run `consume_file` imports that
module and dies if the library is missing. This exact failure was observed first-hand and is shown in
[§3 (Q1)](#3-q1--live-async-behavior) as a worker-death trace; the fix is a one-line `apt-get install`
in §2.2 that restores the libraries the production `Dockerfile` already prescribes — never a source
change.

The data directories are relocated under `/tmp/pp` (writable by `testuser`) via the documented
`PAPERLESS_*` environment variables — the supported configuration mechanism, not a code change.

### 2.2 Create the disposable container and Redis

```console
$ IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01"

$ docker run -d --name pngx_qa --entrypoint sleep "$IMG" infinity
71b83e370f47f2f1911ba3d11bfae49d43b56ab186eacea7a3b20b90fd291f2e

$ docker inspect pngx_qa --format \
    'PortBindings={{json .HostConfig.PortBindings}} Mounts={{json .Mounts}}'
PortBindings={} Mounts=[]

$ docker exec pngx_qa bash -lc 'git config --global --add safe.directory /app; cd /app && git rev-parse HEAD'
542221a38dff06361e07976452f9aea24d210542

# Restore the OS libraries the project Dockerfile installs (see §2.1 provenance note).
# Without libzbar0 the qcluster worker cannot import documents.tasks and consume_file cannot run.
$ docker exec -u root pngx_qa bash -lc \
    'export DEBIAN_FRONTEND=noninteractive; apt-get update -qq && \
     apt-get install -y -qq libzbar0 poppler-utils pngquant'
Setting up libzbar0:amd64 (0.23.90-1+deb11u1) ...
Setting up poppler-utils (20.09.0-3.1+deb11u2) ...
Setting up pngquant (2.13.1-1) ...

$ docker run -d --name pngx_redis --network container:pngx_qa redis:7-alpine \
    redis-server --bind 127.0.0.1 --port 6379 --save '' --appendonly no
c9fdf9a86c76e05e0e361f8ffae8102b79ee282c2b2b398dd01ec63e91a9a63a
```

Redis is now reachable at the canonical `redis://localhost:6379` **inside** the container:

```console
$ docker exec pngx_qa bash /tmp/pp/pp.sh <<'PY'
import redis
r = redis.Redis(host="localhost", port=6379)
print("PING localhost:6379 ->", r.ping())
print("redis_version       ->", r.info().get("redis_version"))
PY
PING localhost:6379 -> True
redis_version       -> 7.4.9
```

> Note: the project ships a startup gate, `docker/wait-for-redis.py`
> (`MAX_RETRY_COUNT=5` L16, `RETRY_SLEEP_SECONDS=5` L17, `REDIS_URL` default L19), which blocks until
> `PAPERLESS_REDIS` answers before the services start — the same reachability proven above.

### 2.3 Directories, environment file, and the two disclosed helper scripts

```console
$ docker exec pngx_qa bash -lc 'mkdir -p /tmp/pp/{data,media,consume,scratch,data/index,data/log} /tmp/obs \
    && chown -R testuser:testuser /tmp/pp /tmp/obs'
```

The single environment file every command sources (`/tmp/pp/penv`):

```bash
export HOME=/tmp/pp
export TMPDIR=/tmp/pp/scratch
export DJANGO_SETTINGS_MODULE=paperless.settings
export PAPERLESS_REDIS=redis://localhost:6379
export PAPERLESS_DATA_DIR=/tmp/pp/data
export PAPERLESS_MEDIA_ROOT=/tmp/pp/media
export PAPERLESS_CONSUMPTION_DIR=/tmp/pp/consume
export PAPERLESS_SCRATCH_DIR=/tmp/pp/scratch
export PAPERLESS_LOGGING_DIR=/tmp/pp/data/log
export PYTHONPATH=/app/src
```

`/tmp/pp/pp.sh` — runs a Python snippet (read from stdin) in the canonical env as non-root `testuser`.
Used throughout for ORM/broker probes; disclosed in full so every later block is exact:

```bash
#!/bin/bash
# pp.sh — run a Python snippet (read from stdin) inside the canonical paperless-ngx
# environment as the non-root 'testuser'. Sources /tmp/pp/penv (the documented env).
exec runuser -u testuser -- bash -lc 'set -a; . /tmp/pp/penv; set +a; cd /app/src && exec python3 -'
```

`/tmp/pp/launch.sh` — starts the three canonical services (per `docker/supervisord.conf`), each
backgrounded, recording each **master PID** via `$!` for a safe, targeted shutdown later:

```bash
#!/bin/bash
set -a; . /tmp/pp/penv; set +a
cd /app/src
nohup python3 manage.py qcluster            > /tmp/obs/qcluster.log 2>&1 < /dev/null & echo $! > /tmp/obs/qcluster.pid ; disown
nohup python3 manage.py document_consumer   > /tmp/obs/consumer.log 2>&1 < /dev/null & echo $! > /tmp/obs/consumer.pid ; disown
nohup gunicorn -c /app/gunicorn.conf.py paperless.asgi:application > /tmp/obs/gunicorn.log 2>&1 < /dev/null & echo $! > /tmp/obs/gunicorn.pid ; disown
```

### 2.4 Migrate, create users (as `testuser`)

Migrations create the Django-Q tables (`django_q_task`, `django_q_schedule`, `django_q_ormq`) via the
`"django_q"` app and seed the recurring schedules and the `consumer` user (used by a signal handler,
see [Q1](#3-q1--live-async-behavior)):

```console
$ docker exec pngx_qa runuser -u testuser -- bash -lc \
    'set -a; . /tmp/pp/penv; set +a; cd /app/src && python3 manage.py migrate --no-input' \
    | grep -iE "Applying django_q|documents.0019_add_consumer_user"
  Applying django_q.0001_initial... OK
  Applying django_q.0002_auto_20150630_1624... OK
  Applying django_q.0003_auto_20150708_1326... OK
  Applying django_q.0004_auto_20150710_1043... OK
  Applying django_q.0005_auto_20150718_1506... OK
  Applying django_q.0006_auto_20150805_1817... OK
  Applying django_q.0007_ormq... OK
  Applying django_q.0008_auto_20160224_1026... OK
  Applying django_q.0009_auto_20171009_0915... OK
  Applying django_q.0010_auto_20200610_0856... OK
  Applying django_q.0011_auto_20200628_1055... OK
  Applying django_q.0012_auto_20200702_1608... OK
  Applying django_q.0013_task_attempt_count... OK
  Applying django_q.0014_schedule_cluster... OK
  Applying documents.0019_add_consumer_user... OK
```

```console
$ docker exec pngx_qa runuser -u testuser -- bash -lc \
    'set -a; . /tmp/pp/penv; set +a; cd /app/src && \
     DJANGO_SUPERUSER_PASSWORD=<REDACTED> python3 manage.py createsuperuser \
       --username admin --email admin@example.com --noinput'
Superuser created successfully.
```

### 2.5 Launch and confirm the three services

```console
$ docker exec pngx_qa runuser -u testuser -- bash /tmp/pp/launch.sh

$ docker exec pngx_qa bash -lc 'for s in qcluster consumer gunicorn; do
    p=$(cat /tmp/obs/$s.pid)
    own=$(stat -c %U /proc/$p 2>/dev/null || echo GONE)
    cmd=$(tr "\0" " " < /proc/$p/cmdline 2>/dev/null)
    echo "$s: PID $p owner=$own :: ${cmd:0:70}"
  done'
qcluster: PID 190 owner=testuser :: python3 manage.py qcluster
consumer: PID 191 owner=testuser :: python3 manage.py document_consumer
gunicorn: PID 192 owner=testuser :: /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/g

$ docker exec pngx_qa bash /tmp/pp/pp.sh <<'PY'
import urllib.request as u
print("GET /api/ ->", u.urlopen("http://localhost:8000/api/", timeout=10).status)
PY
GET /api/ -> 200
```

All three services run as `testuser`; the web endpoint answers on the container-internal `localhost`.
Every temporary helper and log lives under `/tmp` (outside the repository) and the whole container is
discarded at the end; see [Cleanup & repository integrity](#12-cleanup--repository-integrity).

> **Disclosure (worker-cluster PID used in the Q-sections).** In this session the `libzbar0` library
> (§2.1 provenance note) was installed *after* an initial launch, so the first `qcluster` main (PID 190)
> ran workers that could not import `documents.tasks`; a single, clean `SIGTERM` restart then produced
> the **stable cluster used for every observation below — python main PID `2809`, banner
> `queen-table-earth-mirror`** (`consumer` PID 191 and `gunicorn` PID 192 were untouched). In a clean
> reproduction the `apt-get` step in §2.2 precedes the launch, so no restart is needed and a single
> launch yields a stable cluster directly. The verbatim graceful stop of the live cluster is captured at
> teardown in [§12 (Cleanup)](#12-cleanup--repository-integrity).

---

## 3. Q1 — Live async behavior

**Question.** *How does asynchronous work behave in a live setup — the end-to-end lifecycle of a
background job from creation to completion when a document is ingested?*

**Answer (mechanism, cause → effect).** A document is ingested by **enqueuing a single Django-Q task**,
`documents.tasks.consume_file`, and letting the `qcluster` worker cluster run it. The lifecycle has six
observable hops, each proven below with the real command and its unedited output:

1. **Trigger → enqueue.** An entry point (here the directory watcher) calls
   `async_task("documents.tasks.consume_file", <path>, task_name=…)`, which `RPUSH`es a signed task
   package onto the Redis list `django_q:paperless:q`. *(Enqueue call sites are enumerated in
   [§9 (Q7)](#9-q7--enqueue-origin-in-code); the raw package bytes are dissected in
   [§5 (Q3)](#5-q3--how-a-job-appears-when-created).)*
2. **Broker → pusher.** The cluster's **pusher** process `BLPOP`s the package off that Redis list,
   verifies its signature, and `put`s it onto an **in-memory `multiprocessing` task queue** — Redis is
   not read by the workers directly (`django_q/cluster.py:345` `broker.dequeue()`, `:362` `task_queue.put`;
   see [§4](#4-q2--services-involved)).
3. **Worker → execute.** A **worker** process pops the package from that internal queue and calls
   `consume_file(*args)` (`documents/tasks.py:184`), which delegates to
   `Consumer().try_consume_file(...)` (`documents/tasks.py:236` → `documents/consumer.py:180`).
4. **Pipeline.** `try_consume_file` parses the file (`document_parser.parse`, `consumer.py:261`), loads
   the ML classifier once (`load_classifier`, `consumer.py:292`), persists the `Document`, then fires
   `document_consumption_finished.send(...)` (`consumer.py:306`).
5. **Post-consume handlers.** That signal synchronously runs the **six** handlers wired in
   `documents/apps.py:22-27` (inbox tags, correspondent, type, tags, admin log entry, search index).
6. **Result → persist.** The worker returns the result string; the cluster's **monitor** process writes
   one row into the `django_q_task` table and logs `Processed [<name>]`. In parallel, progress is
   broadcast over a WebSocket (`async_to_sync(get_channel_layer().group_send)("status_updates", …)`,
   `documents/tasks.py:226-227`; frames shown in [§8 (Q6)](#8-q6--after-the-fact-status)).

The rest of this section walks a single, real ingestion through those hops. The blocks below were
executed **in the order shown**; each is a snapshot at that point in the sequence (for example, §3.1's
`docs=0` count is the clean slate that exists *before* the §3.2 ingestion). Reproduced in the same
order, every command yields the output shown.

### 3.1 Setup: matching objects so every handler has an observable effect

To make all six post-consume handlers (step 5) produce a *visible* postcondition, four matching objects
were created via the ORM (the supported admin operation, not a code change). All match the literal token
`paperlessqademo` using the default `MATCH_ANY` algorithm (`documents/models.py:21,44`):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from documents.models import Tag, Correspondent, DocumentType, Document
from django_q.models import Task
print("BEFORE-setup counts:",
      "docs=", Document.objects.count(), "tasks=", Task.objects.count(),
      "correspondents=", Correspondent.objects.count(),
      "types=", DocumentType.objects.count(), "tags=", Tag.objects.count())
ANY = 1  # MATCH_ANY (documents/models.py:21)
inbox,_ = Tag.objects.get_or_create(name="QA-Inbox",
    defaults=dict(is_inbox_tag=True, matching_algorithm=ANY, match=""))
auto,_  = Tag.objects.get_or_create(name="QA-Auto",
    defaults=dict(is_inbox_tag=False, matching_algorithm=ANY, match="paperlessqademo"))
corr,_  = Correspondent.objects.get_or_create(name="QA Correspondent",
    defaults=dict(matching_algorithm=ANY, match="paperlessqademo"))
dtype,_ = DocumentType.objects.get_or_create(name="QA Type",
    defaults=dict(matching_algorithm=ANY, match="paperlessqademo"))
print("  inbox Tag id=%d is_inbox_tag=%s match=%r" % (inbox.id, inbox.is_inbox_tag, inbox.match))
print("  auto  Tag id=%d is_inbox_tag=%s match=%r" % (auto.id, auto.is_inbox_tag, auto.match))
print("  Correspondent id=%d match=%r" % (corr.id, corr.match))
print("  DocumentType  id=%d match=%r" % (dtype.id, dtype.match))
PY
BEFORE-setup counts: docs= 0 tasks= 3 correspondents= 0 types= 0 tags= 0
  inbox Tag id=1 is_inbox_tag=True match=''
  auto  Tag id=2 is_inbox_tag=False match='paperlessqademo'
  Correspondent id=1 match='paperlessqademo'
  DocumentType  id=1 match='paperlessqademo'
```

*(The `tasks= 3` rows already present are seeded scheduled jobs that the cluster ran on its own; they
are identified in [§10](#10-scheduled--recurring-jobs). `docs= 0` confirms a clean document slate.)*

### 3.2 Trigger (hop 1): drop a file through the real directory watcher

A `.txt` containing the token is moved into the consumption directory. This exercises the **canonical**
`document_consumer` entry point (not a debug hook):

```console
$ docker exec -u testuser pngx_qa bash -lc '
cat > /tmp/pp/qa2.txt <<TXT
QA demonstration document (run 2) for paperless-ngx runtime investigation.
paperlessqademo invoice from ACME Corporation.
Parsed by the paperless_text parser (text/plain), no OCR required.
TXT
ts=$(date +%Y%m%d_%H%M%S); dest="/tmp/pp/consume/qa_run2_${ts}.txt"
mv /tmp/pp/qa2.txt "$dest"; echo "dropped: $dest"
date -u +"drop_utc=%Y-%m-%dT%H:%M:%S.%NZ"'
dropped: /tmp/pp/consume/qa_run2_20260714_213727.txt
drop_utc=2026-07-14T21:37:27.518291118Z
```

The watcher detects the file and enqueues it. Its own log line (the `_consume` function,
`documents/management/commands/document_consumer.py:85`) is the observable proof of the enqueue:

```console
$ docker exec pngx_qa bash -lc "grep -n 'qa_run2' /tmp/obs/consumer.log"
4:[2026-07-14 21:37:28,518] [INFO] [paperless.management.consumer] Adding /tmp/pp/consume/qa_run2_20260714_213727.txt to the task queue.
```

### 3.3 Worker execution (hops 2–5): `consume_file` runs and the six handlers fire

The `qcluster` worker log records the whole server-side sequence — the worker picking up the task, the
`Consumer` running, three of the six handlers logging their assignment, and the completion line:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_run2_20260714_213727' /tmp/obs/qcluster.log"
17:21:37:28 [Q] INFO Process-1:1 processing [qa_run2_20260714_213727.txt]
18:[2026-07-14 21:37:28,698] [INFO] [paperless.consumer] Consuming qa_run2_20260714_213727.txt
19:[2026-07-14 21:37:29,544] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-14 qa_run2_20260714_213727
20:[2026-07-14 21:37:29,545] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-14 QA Correspondent qa_run2_20260714_213727
21:[2026-07-14 21:37:29,546] [INFO] [paperless.handlers] Tagging "2026-07-14 QA Correspondent qa_run2_20260714_213727" with "QA-Auto"
22:[2026-07-14 21:37:29,597] [INFO] [paperless.consumer] Document 2026-07-14 QA Correspondent qa_run2_20260714_213727 consumption finished
24:21:37:29 [Q] INFO Processed [qa_run2_20260714_213727.txt]
```

Reading that log against the source: `Process-1:1 processing [...]` is the worker loop
(`django_q/cluster.py:420`); `Consuming …` is `Consumer.try_consume_file` starting
(`documents/consumer.py`); the three `paperless.handlers` lines are emitted by `set_correspondent`,
`set_document_type`, and `set_tags` (`documents/signals/handlers.py`); `consumption finished` is logged
right after `document_consumption_finished.send` returns; and `Processed [<name>]` is the **monitor**
process logging success (`django_q/cluster.py:392`) after it has persisted the result row via `save_task`
(`django_q/cluster.py:384` → `Task.objects.create`, `django_q/cluster.py:506`).

**All six** post-consume handlers are verified by their database postconditions on the single resulting
document (`documents/apps.py:22-27` order shown in brackets):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from documents.models import Document
from django.contrib.admin.models import LogEntry
from django.contrib.contenttypes.models import ContentType
from documents.index import open_index
d = Document.objects.get(id=1)
print("Document.id           =", d.id)
print("[2] set_correspondent : correspondent =", d.correspondent.name if d.correspondent else None)
print("[3] set_document_type : document_type =", d.document_type.name if d.document_type else None)
tags = sorted((t.name, t.is_inbox_tag) for t in d.tags.all())
print("[1]+[4] tags (name,is_inbox_tag) =", tags)
ct = ContentType.objects.get_for_model(Document)
les = LogEntry.objects.filter(content_type=ct, object_id=str(d.id))
print("[5] set_log_entry     : LogEntry count =", les.count())
for le in les:
    print("        LogEntry user=%s action_flag=%s" % (le.user.username, le.action_flag))
with open_index().searcher() as s:
    present = any(f.get("id") == d.id for _, f in s.iter_docs())
print("[6] add_to_index      : this document present in index:", present)
PY
Document.id           = 1
[2] set_correspondent : correspondent = QA Correspondent
[3] set_document_type : document_type = QA Type
[1]+[4] tags (name,is_inbox_tag) = [('QA-Auto', False), ('QA-Inbox', True)]
[5] set_log_entry     : LogEntry count = 1
        LogEntry user=consumer action_flag=1
[6] add_to_index      : this document present in index: True
```

Mapping each observed postcondition to its handler and cause:

| # | Handler (`documents/apps.py` line) | Observed postcondition | Cause |
|---|---|---|---|
| 1 | `add_inbox_tags` (`:22`) | tag `QA-Inbox` (`is_inbox_tag=True`) on the doc | inbox tags are added unconditionally |
| 2 | `set_correspondent` (`:23`) | `correspondent = QA Correspondent` | token matched the correspondent's `match` |
| 3 | `set_document_type` (`:24`) | `document_type = QA Type` | token matched the type's `match` |
| 4 | `set_tags` (`:25`) | tag `QA-Auto` on the doc | token matched the tag's `match` |
| 5 | `set_log_entry` (`:26`) | one `LogEntry` by user `consumer`, `action_flag=1` (ADDITION) | handler looks up the seeded `consumer` user (migration `0019_add_consumer_user.py`) |
| 6 | `add_to_index` (`:27`) | doc id `1` present in the Whoosh search index | handler calls `index.add_or_update_document` |

### 3.4 Completion (hop 6): the persisted `django_q_task` row

When the worker returns, the monitor writes exactly one row. This is the durable record of the job
(full storage model in [§7 (Q5)](#7-q5--where-task-state-is-stored) / [§8 (Q6)](#8-q6--after-the-fact-status)):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(name="qa_run2_20260714_213727.txt")
print("Task.id        =", t.id)
print("Task.name      =", t.name)
print("Task.func      =", t.func)
print("Task.args      =", repr(t.args))
print("Task.success   =", t.success)
print("Task.started   =", t.started.isoformat())
print("Task.stopped   =", t.stopped.isoformat())
print("Task.time_taken=", "%.3f s" % t.time_taken())
print("Task.result    =", repr(t.result))
PY
Task.id        = 9593494164364f649ee2727659f78fbb
Task.name      = qa_run2_20260714_213727.txt
Task.func      = documents.tasks.consume_file
Task.args      = ('/tmp/pp/consume/qa_run2_20260714_213727.txt',)
Task.success   = True
Task.started   = 2026-07-14T21:37:28.519407+00:00
Task.stopped   = 2026-07-14T21:37:29.600141+00:00
Task.time_taken= 1.081 s
Task.result    = 'Success. New document id 1 created'
```

### 3.5 End-to-end timing (single calculation, from absolute timestamps)

First, what the two timestamps *mean* (grounded in the pinned Django-Q source):

- `Task.started` is stamped at **enqueue time**, not worker-start: `async_task` sets
  `task["started"] = timezone.now()` (`django_q/tasks.py:65`) *before* `broker.enqueue(...)`
  (`django_q/tasks.py:73`). (The same value travels inside the package — it is exactly the `started`
  field peeked in [§5 (Q3)](#5-q3--how-a-job-appears-when-created).)
- `Task.stopped` is stamped when the worker finishes the function: `task["stopped"] = timezone.now()`
  (`django_q/cluster.py:444`). `save_task` persists both unchanged (`django_q/cluster.py:513-514`).

Therefore `time_taken() = stopped − started = (queue-wait) + (execution)`. As a single calculation from
the absolute timestamps (rounded to 3 decimals throughout this document):

```
stopped − started = 2026-07-14T21:37:29.600141Z − 2026-07-14T21:37:28.519407Z
                  = 1.080734 s  ≈  1.081 s
```

This equals the `Task.time_taken()` value (`1.081 s`) Django-Q reports. Because the cluster was **up**
here, the queue-wait was negligible — the worker's `processing`/`Consuming` lines land at `21:37:28,698`,
only ≈ 0.18 s after the enqueue — so this `time_taken` is dominated by execution. The
**user-perceived** latency is a bit larger still: the file was dropped at `21:37:27.518` but the watcher
logged the enqueue at `21:37:28,518`, i.e. the watcher's debounce added ≈ 1.0 s before the task even
existed. That `started`-is-enqueue semantics becomes vivid in [§6 (Q4)](#6-q4--waiting-vs-actively-processing),
where a task deliberately left waiting reports a `time_taken` of **115.780 s** for ≈ 1 s of real work.

### 3.6 Edge conditions observed on the live path

The happy path above is not the only condition the lifecycle exhibits. Two non-happy conditions were
observed first-hand (fuller error/edge coverage is consolidated in
[§11](#11-exhaustive-condition-coverage)):

**(a) Non-fatal thumbnail warning (PDF).** Ingesting a text-layer PDF (generated with `reportlab`)
exercises PDF thumbnailing. paperless first tries ImageMagick `convert`, which this image blocks via its
PDF security policy; paperless catches the failure and **falls back to Ghostscript**, logging a
`WARNING`. Consumption still succeeds — the warning is *non-fatal*:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_thumb_20260714_213835|convert-im6|falling back' /tmp/obs/qcluster.log"
27:21:38:36 [Q] INFO Process-1:2 processing [qa_thumb_20260714_213835.pdf]
28:[2026-07-14 21:38:36,474] [INFO] [paperless.consumer] Consuming qa_thumb_20260714_213835.pdf
29:convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
30:convert-im6.q16: no images defined `/tmp/pp/scratch/paperless-lwwo2xiy/convert.png' @ error/convert.c/ConvertImageCommand/3229.
31:[2026-07-14 21:38:37,195] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
34:21:38:38 [Q] INFO Processed [qa_thumb_20260714_213835.pdf]
```

The fallback and the warning are exactly the code at `documents/parsers.py`: `make_thumbnail_from_pdf`
(`:187`) runs `run_convert` (`:111`); on `ParseError` it calls `make_thumbnail_from_pdf_gs_fallback`
(`:206` → `:153`), whose `logger.warning(...)` is the line seen above (`:159`). The task still records
success:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.filter(name__startswith="qa_thumb").order_by("-stopped").first()
print("id=%s success=%s result=%r time_taken=%.3fs" % (t.id, t.success, t.result, t.time_taken()))
PY
id=114a9e07bbcf4c0c953390a8a7ad5448 success=True result='Success. New document id 2 created' time_taken=1.751s
```

**(b) Worker death vs. clean failure (missing native library).** Before the `libzbar0` library was
installed (§2.1 provenance note), the very first `consume_file` attempt made the worker **die at import
time** rather than fail gracefully. This is an instructive contrast: a *dead* worker never returns a
result, so the monitor writes **no** `django_q_task` row and, with the Redis broker (no delivery
receipts), the task is simply **lost** — not retried. This trace was emitted into the live
`qcluster.log` during the first launch (before `libzbar0` was installed); because that log was truncated
when the fixed cluster was relaunched, the excerpt is retained verbatim in a preserved evidence file and
is shown here **complete and unedited**:

```console
$ docker exec pngx_qa bash -lc "cat /tmp/obs/first_launch_worker_death.log"
21:34:28 [Q] INFO Process-1:2 processing [qa_ingest_20260714_213427.txt]
Process Process-1:2:
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/pydoc.py", line 439, in safeimport
    module = __import__(path)
  File "/app/src/documents/tasks.py", line 25, in <module>
    from pyzbar import pyzbar
  File "/usr/local/lib/python3.9/site-packages/pyzbar/pyzbar.py", line 7, in <module>
    from .wrapper import (
  File "/usr/local/lib/python3.9/site-packages/pyzbar/wrapper.py", line 151, in <module>
    zbar_version = zbar_function(
  File "/usr/local/lib/python3.9/site-packages/pyzbar/wrapper.py", line 148, in zbar_function
    return prototype((fname, load_libzbar()))
  File "/usr/local/lib/python3.9/site-packages/pyzbar/wrapper.py", line 127, in load_libzbar
    libzbar, dependencies = zbar_library.load()
  File "/usr/local/lib/python3.9/site-packages/pyzbar/zbar_library.py", line 65, in load
    raise ImportError('Unable to find zbar shared library')
ImportError: Unable to find zbar shared library

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 315, in _bootstrap
    self.run()
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 108, in run
    self._target(*self._args, **self._kwargs)
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 424, in worker
    f = pydoc.locate(f)
  File "/usr/local/lib/python3.9/pydoc.py", line 1718, in locate
    nextmodule = safeimport('.'.join(parts[:n+1]), forceload)
  File "/usr/local/lib/python3.9/pydoc.py", line 454, in safeimport
    raise ErrorDuringImport(path, sys.exc_info())
pydoc.ErrorDuringImport: problem in documents.tasks - ImportError: Unable to find zbar shared library
21:34:29 [Q] ERROR reincarnated worker Process-1:2 after death
21:34:29 [Q] INFO Process-1:15 ready for work at 2261
```

The worker imports the task target lazily at run time via `pydoc.locate(f)` (`django_q/cluster.py:424`);
because `documents/tasks.py:25` imports `pyzbar` at module scope, the missing shared library propagates
as an uncaught error that kills the process. Django-Q's guard **reincarnates** the dead worker (a new
worker PID appears), but the in-flight task yields no row. After installing `libzbar0` (§2.2), the same
ingestion produced the successful row shown in §3.4. This difference — *clean failure writes a
`success=False` row; a process death writes nothing* — is revisited with a forced clean failure in
[§8 (Q6)](#8-q6--after-the-fact-status).

> **(inferred)** The "task is lost, not retried" claim for a *process death* under the Redis broker is
> inferred from the broker having no acknowledge/receipt mechanism (see the broker source in
> [§6 (Q4)](#6-q4--waiting-vs-actively-processing)); it was corroborated by observing that no new row and
> no re-processing appeared for `qa_ingest_20260714_213427.txt` after the death.

---


## 4. Q2 — Services involved

**Question.** *Which services/processes participate behind the scenes (message broker, worker cluster,
scheduler, web/API process, file watcher)?*

**Answer (mechanism, cause → effect).** Production runs **three long-running programs** under
Supervisord plus **one external service (Redis)**. Crucially, the third program is *not* a bare
scheduler — it is the **Django-Q worker cluster**, which is itself a *tree of real OS processes*, and
the scheduler is a loop *inside* that tree, not a separate process. The participants are:

| Participant | What it is | Canonical command | Runtime evidence |
|---|---|---|---|
| **Redis** | message broker **and** Channels layer (dual role) | external `redis-server` | §4.4 |
| **web/API + WebSocket** | `gunicorn` running the ASGI app `paperless.asgi:application` | `gunicorn -c …/gunicorn.conf.py paperless.asgi:application` | §4.1, §4.3 |
| **file watcher** | `document_consumer` management command | `python3 manage.py document_consumer` | §4.1 |
| **worker cluster (+ scheduler)** | `qcluster` — a process tree: 1 main + 1 Sentinel + N workers + monitor + pusher; scheduler runs inside the Sentinel | `python3 manage.py qcluster` | §4.2 |

These three programs are exactly the `docker/supervisord.conf` definitions (note the program *named*
`scheduler` runs `qcluster`):

```console
$ docker exec pngx_qa bash -lc "grep -nE '^\[program|command=|user=' /app/docker/supervisord.conf"
10:[program:gunicorn]
11:command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
12:user=paperless
19:[program:consumer]
20:command=python3 manage.py document_consumer
21:user=paperless
28:[program:scheduler]
29:command=python3 manage.py qcluster
30:user=paperless
```

### 4.1 The three live services (PID, owner, exact command line)

```console
$ docker exec pngx_qa bash -lc '
for name in qcluster consumer gunicorn; do
  pid=$(cat /tmp/obs/$name.pid)
  echo "[$name] pid=$pid comm=$(cat /proc/$pid/comm) owner=$(stat -c %U /proc/$pid)"
  echo "   cmdline=$(tr "\0" " " < /proc/$pid/cmdline)"
done'
[qcluster] pid=2809 comm=python3 owner=testuser
   cmdline=python3 manage.py qcluster
[consumer] pid=191 comm=python3 owner=testuser
   cmdline=python3 manage.py document_consumer
[gunicorn] pid=192 comm=gunicorn owner=testuser
   cmdline=/usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

### 4.2 The `qcluster` process tree — real processes, not threads

Finding-of-record: Django-Q workers are **separate operating-system processes**, each with its own PID,
not threads inside one interpreter. This is directly visible by walking `/proc` from the cluster main
down through the Sentinel to its children (the cluster used here is `queen-table-earth-mirror`, main
PID `2809`, per the §2.5 disclosure).

> **Cluster generation note.** The specific PIDs, `cluster_id`, and banner name below are a point-in-time
> snapshot of the cluster generation that was live when Q2 was observed. The **structure** (1 main + 1
> Sentinel + 11 workers + 1 monitor + 1 pusher) and the worker **count** are generation-invariant. The
> Q4 boundary demonstration in [§6](#6-q4--waiting-vs-actively-processing) deliberately stops and
> restarts the cluster, so each subsequent generation gets a fresh `cluster_id`/PIDs/banner; the
> `/proc`-walk and `Stat` commands here read the *current* pidfile, so they reproduce the same structure
> against whichever generation is live.

Walking the tree:

```console
$ docker exec pngx_qa bash -lc '
MAIN=$(cat /tmp/obs/qcluster.pid)
echo "CLUSTER MAIN pid=$MAIN comm=$(cat /proc/$MAIN/comm) ppid=$(awk "/^PPid:/{print \$2}" /proc/$MAIN/status)"
for c in /proc/[0-9]*/status; do
  p=$(awk "/^PPid:/{print \$2}" "$c"); pid=$(awk "/^Pid:/{print \$2}" "$c")
  [ "$p" = "$MAIN" ] && SENT=$pid && echo "  SENTINEL pid=$pid comm=$(cat /proc/$pid/comm) ppid=$p"
done
n=0
for c in /proc/[0-9]*/status; do
  p=$(awk "/^PPid:/{print \$2}" "$c"); pid=$(awk "/^Pid:/{print \$2}" "$c")
  if [ "$p" = "$SENT" ]; then n=$((n+1)); echo "    child#$n pid=$pid comm=$(cat /proc/$pid/comm)"; fi
done
echo "  => Sentinel has $n child processes (expect 11 workers + 1 monitor + 1 pusher = 13)"'
CLUSTER MAIN pid=2809 comm=python3 ppid=1
  SENTINEL pid=2835 comm=python3 ppid=2809
    child#1 pid=2836 comm=python3
    child#2 pid=2837 comm=python3
    child#3 pid=2838 comm=python3
    child#4 pid=2839 comm=python3
    child#5 pid=2840 comm=python3
    child#6 pid=2841 comm=python3
    child#7 pid=2842 comm=python3
    child#8 pid=2843 comm=python3
    child#9 pid=2844 comm=python3
    child#10 pid=2845 comm=python3
    child#11 pid=2846 comm=python3
    child#12 pid=2847 comm=python3
    child#13 pid=2848 comm=python3
  => Sentinel has 13 child processes (expect 11 workers + 1 monitor + 1 pusher = 13)
```

That tree maps one-to-one onto the cluster's own startup banner (each `Process-1:N` is one of the PIDs
above):

```console
$ docker exec pngx_qa bash -lc "sed -n '1,15p' /tmp/obs/qcluster.log"
21:36:49 [Q] INFO Q Cluster queen-table-earth-mirror starting.
21:36:49 [Q] INFO Process-1:1 ready for work at 2836
21:36:49 [Q] INFO Process-1:2 ready for work at 2837
21:36:49 [Q] INFO Process-1:3 ready for work at 2838
21:36:49 [Q] INFO Process-1:4 ready for work at 2839
21:36:49 [Q] INFO Process-1:5 ready for work at 2840
21:36:49 [Q] INFO Process-1:6 ready for work at 2841
21:36:49 [Q] INFO Process-1:7 ready for work at 2842
21:36:49 [Q] INFO Process-1:8 ready for work at 2843
21:36:49 [Q] INFO Process-1:9 ready for work at 2844
21:36:49 [Q] INFO Process-1:10 ready for work at 2845
21:36:49 [Q] INFO Process-1:11 ready for work at 2846
21:36:49 [Q] INFO Process-1:12 monitoring at 2847
21:36:49 [Q] INFO Process-1 guarding cluster queen-table-earth-mirror
21:36:49 [Q] INFO Process-1:13 pushing tasks at 2848
```

Reading the banner against `django_q/cluster.py`: the **main** process (`manage.py qcluster`, PID 2809)
spawns one **Sentinel** process (`Process(target=Sentinel)`, `cluster.py:68`; PID 2835). The Sentinel
spawns the **11 workers** (`Process-1:1..11`, `spawn_worker`), **1 monitor** (`Process-1:12 monitoring`,
`spawn_monitor`, `cluster.py:208`), and **1 pusher** (`Process-1:13 pushing tasks`, `spawn_pusher`,
`cluster.py:200`); it then runs the **guard** loop itself (`Process-1 guarding`, `cluster.py:253`).

**Where is the scheduler?** There is **no separate scheduler process**. The scheduler is a function
(`django_q/cluster.py:576`) that the guard loop calls *synchronously* inside the Sentinel every
`GUARD_CYCLE`-driven ~30 s. So the Supervisord program named `scheduler` is really the whole worker
cluster, and the actual scheduling work happens inside `Process-1` (the Sentinel). This is proven live
in [§10 (Scheduled jobs)](#10-scheduled--recurring-jobs), where a schedule fires with the log line
`Process-1 created a task from schedule …`.

The cluster's own status API agrees with the `/proc` tree — 11 workers, cluster name `paperless`, and
the two internal `multiprocessing` queues both empty at idle:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.monitor import Stat
from django_q.conf import Conf
for s in Stat.get_all():
    print("cluster_id=%s pid=%s status=%s workers=%d task_q=%d done_q=%d"
          % (s.cluster_id, s.pid, s.status, len(s.workers), s.task_q_size, s.done_q_size))
print("Conf.PREFIX(cluster name) =", Conf.PREFIX)
PY
cluster_id=0c7ab676-7fb5-4456-aa78-dd38719b8fe9 pid=2809 status=Idle workers=11 task_q=0 done_q=0
Conf.PREFIX(cluster name) = paperless
```

The worker count of **11** is not arbitrary: `default_task_workers()` (`paperless/settings.py:427`)
returns `floor(sqrt(cpu_count))` when there are ≥ 4 cores, and this host reports 128 cores:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import os, math
print("cpu_count=%d -> floor(sqrt)=%d" % (os.cpu_count(), math.floor(math.sqrt(os.cpu_count()))))
PY
cpu_count=128 -> floor(sqrt)=11
```

The cluster configuration these processes obey is `Q_CLUSTER` (`paperless/settings.py:449-457`):
`name="paperless"`, `catch_up=False`, `recycle=1` (a worker is recycled after each task), `timeout=1800`
s, `retry=1810` s, `workers=11`, `redis="redis://localhost:6379"`. Read back at runtime:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.conf import settings
q = settings.Q_CLUSTER
print({k: q[k] for k in ("name","catch_up","recycle","timeout","retry","workers","redis")})
PY
{'name': 'paperless', 'catch_up': False, 'recycle': 1, 'timeout': 1800, 'retry': 1810, 'workers': 11, 'redis': 'redis://localhost:6379'}
```

### 4.3 gunicorn — the web/API + WebSocket process

`gunicorn` is a master process (PID 192) with worker children serving the ASGI app (which carries both
the REST API and the Channels WebSocket routes, `paperless/asgi.py:17-20`):

```console
$ docker exec pngx_qa bash -lc '
G=$(cat /tmp/obs/gunicorn.pid); echo "gunicorn master=$G"
for c in /proc/[0-9]*/status; do
  p=$(awk "/^PPid:/{print \$2}" "$c"); pid=$(awk "/^Pid:/{print \$2}" "$c")
  [ "$p" = "$G" ] && echo "  worker pid=$pid comm=$(cat /proc/$pid/comm)"
done'
gunicorn master=192
  worker pid=194 comm=gunicorn
  worker pid=195 comm=gunicorn
```

### 4.4 Redis — one server, two roles

Redis is used simultaneously as the **Django-Q broker** (`Q_CLUSTER["redis"]`) and the **Channels
layer** backing WebSocket fan-out (`CHANNEL_LAYERS[...]["BACKEND"] =
channels_redis.core.RedisChannelLayer`, `paperless/settings.py:178-187`, which also tunes the layer with
`capacity: 2000` and `expiry: 15` at `settings.py:183-184`). Both point at the same server:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.conf import settings
print("broker  Q_CLUSTER[redis]       =", settings.Q_CLUSTER["redis"])
print("channels CHANNEL_LAYERS backend =", settings.CHANNEL_LAYERS["default"]["BACKEND"])
print("channels CHANNEL_LAYERS hosts   =", settings.CHANNEL_LAYERS["default"]["CONFIG"]["hosts"])
import redis
r = redis.Redis.from_url(settings.Q_CLUSTER["redis"])
print("PING =", r.ping(), "| redis_version =", r.info()["redis_version"])
PY
broker  Q_CLUSTER[redis]       = redis://localhost:6379
channels CHANNEL_LAYERS backend = channels_redis.core.RedisChannelLayer
channels CHANNEL_LAYERS hosts   = ['redis://localhost:6379']
PING = True | redis_version = 7.4.9
```

### 4.5 Stopping the cluster

A `qcluster` is stopped with a single `SIGTERM` to its main PID; the Sentinel drains its workers and the
monitor before exiting. Because that shutdown is part of teardown, the **actual, verbatim** graceful
stop of the live cluster (`queen-table-earth-mirror`, PID 2809) is captured at cleanup time in
[§12 (Cleanup & repository integrity)](#12-cleanup--repository-integrity).

> **(inferred)** The mapping of each banner line to a specific `spawn_*` call in `django_q/cluster.py` is
> read from that pinned source; the *existence and count* of the processes (1 main + 1 Sentinel + 11
> workers + 1 monitor + 1 pusher) is **observed** directly from `/proc` and the banner above.

---


## 5. Q3 — How a job appears when created

**Question.** *How does a background job "appear" the moment it is created — what identifiers and payload
accompany it, and where does it first land?*

**Answer (mechanism, cause → effect).** The instant a job is created, `async_task(...)` builds a plain
Python **task dict**, stamps it, **signs + pickles** it into one opaque string, and `RPUSH`es that
string onto the Redis list `django_q:paperless:q`. That is where the job first lands — as a single
element at the tail of a Redis list — and `async_task` returns the job's **id** (a 32-char hex UUID).
No database row exists yet. The identifiers/payload that accompany the job are the dict's keys:
`id`, `name`, `func`, `args`, `kwargs`, and `started` (the enqueue timestamp).

The relevant source, in order (`django_q/tasks.py`):

```console
$ docker exec pngx_qa bash -lc "sed -n '1,120p' /usr/local/lib/python3.9/site-packages/django_q/tasks.py | grep -nE 'def async_task|task\[\"started\"\] = |pack = SignedPackage|enqueue_id = broker|return task\[\"id\"\]'"
20:def async_task(func, *args, **kwargs):
65:    task["started"] = timezone.now()
69:    pack = SignedPackage.dumps(task)
73:    enqueue_id = broker.enqueue(pack)
76:    return task["id"]
```

So `task["started"]` is set at enqueue (`tasks.py:65`), the dict is serialized by `SignedPackage.dumps`
(`tasks.py:69`), pushed with `broker.enqueue` (`tasks.py:73`), and the **id is returned** (`tasks.py:76`).
`broker.enqueue` is a Redis `RPUSH` onto `django_q:{name}:q` (`django_q/brokers/redis_broker.py:15,18`).

### 5.1 The job as it actually appears on the queue (observed)

To capture the job *at the moment of creation*, the cluster was stopped first (so nothing drains the
list), then a file was dropped through the real watcher. The package then sits in Redis and can be read
**non-destructively with `LINDEX`** (peek at index 0 — it does not pop the element):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.conf import settings
from django_q.signing import SignedPackage
from django_q.models import Task
import redis
r = redis.Redis.from_url(settings.Q_CLUSTER["redis"])
key = "django_q:paperless:q"
llen = r.llen(key)
print("LLEN %s = %d  (WAITING: package sits in the Redis list)" % (key, llen))
raw = r.lindex(key, 0)                      # LINDEX = non-destructive peek at head
print("LINDEX head: type=%s length=%d bytes" % (type(raw).__name__, len(raw)))
print("first 60 raw bytes:", raw[:60])
pkg = SignedPackage.loads(raw)              # verifies HMAC signature (key=SECRET_KEY, salt=PREFIX)
print("--- decoded task package (keys) ---")
for k in sorted(pkg.keys()):
    print("  %-10s = %r" % (k, pkg[k]))
print("--- no persisted row yet for this id ---")
print("Task.objects.filter(id=%r).exists() = %s" % (pkg["id"], Task.objects.filter(id=pkg["id"]).exists()))
PY
LLEN django_q:paperless:q = 1  (WAITING: package sits in the Redis list)
LINDEX head: type=bytes length=483 bytes
first 60 raw bytes: b'gAWVOQEAAAAAAAB9lCiMAmlklIwgZjkzMjlkN2VkYjE3NDg2YWIyMDFiOWQ3'
--- decoded task package (keys) ---
  args       = ('/tmp/pp/consume/qa_wait_run1_20260714_215813.txt',)
  func       = 'documents.tasks.consume_file'
  id         = 'f9329d7edb17486ab201b9d7e9057e10'
  kwargs     = {'override_tag_ids': None}
  name       = 'qa_wait_run1_20260714_215813.txt'
  started    = datetime.datetime(2026, 7, 14, 21, 58, 14, 969374, tzinfo=datetime.timezone.utc)
--- no persisted row yet for this id ---
Task.objects.filter(id='f9329d7edb17486ab201b9d7e9057e10').exists() = False
```

Reading the decoded package field-by-field (these are the "identifiers and payload" the question asks
for):

| Field | Observed value | Meaning |
|---|---|---|
| `id` | `f9329d7edb17486ab201b9d7e9057e10` | 32-char hex UUID; the value `async_task` returns; becomes the `django_q_task` primary key |
| `func` | `documents.tasks.consume_file` | dotted path the worker will `pydoc.locate` and call |
| `args` | `('/tmp/pp/consume/qa_wait_run1_20260714_215813.txt',)` | positional args — here the file path to ingest |
| `kwargs` | `{'override_tag_ids': None}` | keyword args passed by the watcher's `async_task` call |
| `name` | `qa_wait_run1_20260714_215813.txt` | human-readable label (the watcher uses the filename) |
| `started` | `2026-07-14 21:58:14.969374+00:00` | enqueue timestamp (`tasks.py:65`); later copied verbatim into the row |

The final line confirms the crucial point: **at creation the job exists only as a Redis list element —
there is no `django_q_task` row yet** (`Task.objects.filter(id=…).exists() = False`). The row is written
only after the worker finishes (see [§7](#7-q5--where-task-state-is-stored) / [§8](#8-q6--after-the-fact-status)).

### 5.2 Why the payload is opaque bytes — the signed-pickle trust boundary

The 483-byte value is not human-readable because Django-Q **signs and pickles** every package.
`SignedPackage.dumps/loads` (`django_q/signing.py`) wrap Django's signing with a pickle serializer,
using `key=Conf.SECRET_KEY` and `salt=Conf.PREFIX` (the cluster name `paperless`):

```console
$ docker exec pngx_qa bash -lc "sed -n '1,27p' /usr/local/lib/python3.9/site-packages/django_q/signing.py"
"""Package signing."""
import pickle

from django_q import core_signing as signing
from django_q.conf import Conf

BadSignature = signing.BadSignature


class SignedPackage:
    """Wraps Django's signing module with custom Pickle serializer."""

    @staticmethod
    def dumps(obj, compressed: bool = Conf.COMPRESSED) -> str:
        return signing.dumps(
            obj,
            key=Conf.SECRET_KEY,
            salt=Conf.PREFIX,
            compress=compressed,
            serializer=PickleSerializer,
        )

    @staticmethod
    def loads(obj) -> any:
        return signing.loads(
            obj, key=Conf.SECRET_KEY, salt=Conf.PREFIX, serializer=PickleSerializer
        )
```

> **Security / trust boundary (important).** `SignedPackage.loads` first **verifies an HMAC signature**
> keyed by this deployment's Django `SECRET_KEY` (salted by the cluster prefix) and only then unpickles.
> It is therefore safe to call **only** on packages produced by the *same* canonical broker — i.e. bytes
> this very deployment signed. A blob not signed with the matching key raises `BadSignature` and is never
> unpickled. This decode must **never** be pointed at arbitrary/untrusted external bytes: unsigned pickle
> is unsafe, and the signature gate is exactly what keeps the broker's private queue a closed trust
> boundary. The peek above is legitimate because it reads *our own* queue with *our own* key.

### 5.3 Scheduled jobs appear the same way

A recurring job is not special at creation time: when a `Schedule` becomes due, the scheduler calls the
same `async_task(...)`, producing the same kind of signed package on the same Redis list. The only
difference is *who* calls `async_task` (the Sentinel's scheduler loop rather than an ingestion entry
point). This is shown live in [§10 (Scheduled jobs)](#10-scheduled--recurring-jobs).

---

## 6. Q4 — Waiting vs. actively processing

**Question.** *How does the system distinguish work that is waiting (queued/pending) from work that is
actively being processed (started/running)?*

**Answer (mechanism, cause → effect).** There are **three** distinguishable states, and the naive signal
(the Redis queue length) only tells you about the first one:

- **WAITING** — the signed package is an element in the Redis list `django_q:paperless:q`. It is counted
  by `LLEN` (which the broker exposes as `queue_size()`, `redis_broker.py:26`). No `django_q_task` row
  exists.
- **IN-CLUSTER / ACTIVELY PROCESSING** — the **pusher** has already `BLPOP`ed the package off Redis
  (`redis_broker.py:21`) and `put` it on the cluster's **internal `multiprocessing` task queue**; a
  **worker** then pops it and runs the function. During this whole span `LLEN = 0` **and no row exists
  yet**. This is the subtle part: *a zero queue length does **not** mean "done" — it can mean the task is
  internally queued or actively running.*
- **DONE** — the worker returned, the **monitor** wrote the `django_q_task` row. The durable "it
  finished" signal is the **row**, not the queue length.

The critical correction this section proves: **workers never read Redis directly.** Only the single
pusher reads Redis; everything past that point lives in an in-memory `multiprocessing` queue
(`django_q/cluster.py:345` `broker.dequeue()` → `django_q/brokers/redis_broker.py:21` `BLPOP` →
`cluster.py:362` `task_queue.put` → `:415` worker `task_queue.get`). So `LLEN=0 + no row` is ambiguous
between "internally queued" and "running", and neither is "done".

### 6.1 The poller (published in full)

To observe WAITING vs. ACTIVE vs. DONE with timestamps, a small poller samples, every 0.1 s: the Redis
`LLEN`, the cluster `Stat` (status + internal `task_q`/`done_q` sizes), and whether the **specific**
task's row exists yet — tagging every sample with a `run_id` and an `env_id`, and watching one exact
`task_id`. Its complete source (no hidden resets or filters):

```console
$ docker exec pngx_qa bash -lc "cat /tmp/pp/poll.py"
#!/usr/bin/env python3
"""Boundary poller for Q4: samples the WAITING/PROCESSING/DONE state of ONE task.
Usage: poll.py <task_id> <task_name> <run_id> <env_id> <duration_s> <interval_s>
Each sample line: run_id | env_id | iso_utc | LLEN | cluster_status | task_q | done_q | state(task_id)
  state = WAITING (LLEN>=1, no row) | IN-CLUSTER/ACTIVE (LLEN==0, no row) | DONE(success=..)
"""
import os, sys, time, django
from datetime import datetime, timezone
django.setup()
from django.conf import settings
from django_q.models import Task
from django_q.monitor import Stat
import redis

task_id, task_name, run_id, env_id = sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4]
duration, interval = float(sys.argv[5]), float(sys.argv[6])
r = redis.Redis.from_url(settings.Q_CLUSTER["redis"])
list_key = "django_q:%s:q" % settings.Q_CLUSTER["name"]

print("# run_id=%s env_id=%s watch_task_id=%s watch_name=%s" % (run_id, env_id, task_id, task_name))
print("# %-26s | %-4s | %-8s | %-4s | %-4s | %s" % ("iso_utc","LLEN","cluster","task_q","done_q","state"))
end = time.time() + duration
seen_done = 0
while time.time() < end:
    ts = datetime.now(timezone.utc).isoformat(timespec="milliseconds")
    llen = r.llen(list_key)
    stats = Stat.get_all()
    if stats:
        s = stats[0]; cstat, tq, dq = s.status, s.task_q_size, s.done_q_size
    else:
        cstat, tq, dq = "down", "-", "-"
    row = Task.objects.filter(id=task_id).first()
    if row is None:
        state = "WAITING" if llen >= 1 else "IN-CLUSTER/ACTIVE"
    else:
        state = "DONE(success=%s)" % row.success
        seen_done += 1
    print("%-28s | %-4s | %-8s | %-4s | %-4s | %s" % (ts, llen, cstat, tq, dq, state))
    sys.stdout.flush()
    if seen_done >= 3:   # capture a few DONE samples then stop
        break
    time.sleep(interval)
```

The experiment for each run: (1) stop the cluster; (2) drop a file through the watcher — with nothing
draining the list, the package **waits**; (3) peek it to learn its `task_id` (the [§5](#5-q3--how-a-job-appears-when-created)
capture); (4) start the poller watching that id; (5) start the cluster and let the task drain → run →
persist. The poller thus records the full **before / during / after** of the same, correlated task.

### 6.2 Run 1 — the complete before/during/after trace

```console
$ docker exec pngx_qa bash -lc "cat /tmp/obs/poll_run1.csv"
# run_id=run1 env_id=pngx_qa/542221a38dff/redis7.4.9 watch_task_id=f9329d7edb17486ab201b9d7e9057e10 watch_name=qa_wait_run1_20260714_215813.txt
# iso_utc                    | LLEN | cluster  | task_q | done_q | state
2026-07-14T22:00:08.081+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.191+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.293+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.395+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.497+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.599+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.701+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.803+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:08.906+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.008+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.109+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.212+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.314+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.416+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.518+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.621+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.723+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.825+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:00:09.942+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.045+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.147+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.250+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.352+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.455+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.557+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.661+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:00:10.763+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-14T22:00:10.866+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-14T22:00:10.968+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
```

The three states are unmistakable and correlated to the *same* `task_id`:

- **WAITING** `22:00:08.081 → 22:00:09.825`: `LLEN=1`, cluster `down`, no row.
- **IN-CLUSTER/ACTIVE** `22:00:09.942 → 22:00:10.661`: `LLEN=0`, cluster present, internal `task_q=0`
  and `done_q=0`, **still no row**. This is the window that a naive "is the queue empty?" check would
  wrongly report as finished.
- **DONE** from `22:00:10.763`: the row now exists with `success=True`.

That the ACTIVE window is *genuinely executing the function* (not merely idle) is confirmed by the
worker's own log for the same task, with absolute timestamps that fall inside the ACTIVE window above:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_wait_run1' /tmp/obs/qcluster.log"
84:22:00:09 [Q] INFO Process-1:1 processing [qa_wait_run1_20260714_215813.txt]
85:[2026-07-14 22:00:10,037] [INFO] [paperless.consumer] Consuming qa_wait_run1_20260714_215813.txt
86:[2026-07-14 22:00:10,687] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-14 qa_wait_run1_20260714_215813
87:[2026-07-14 22:00:10,688] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-14 QA Correspondent qa_wait_run1_20260714_215813
88:[2026-07-14 22:00:10,690] [INFO] [paperless.handlers] Tagging "2026-07-14 QA Correspondent qa_wait_run1_20260714_215813" with "QA-Auto"
89:[2026-07-14 22:00:10,745] [INFO] [paperless.consumer] Document 2026-07-14 QA Correspondent qa_wait_run1_20260714_215813 consumption finished
91:22:00:10 [Q] INFO Processed [qa_wait_run1_20260714_215813.txt]
```

Finally the persisted row, correlated by the **same id** peeked while waiting:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(id="f9329d7edb17486ab201b9d7e9057e10")
print("id=%s success=%s" % (t.id, t.success))
print("started =", t.started.isoformat())
print("stopped =", t.stopped.isoformat())
print("time_taken = %.3f s" % t.time_taken())
print("result =", repr(t.result))
PY
id=f9329d7edb17486ab201b9d7e9057e10 success=True
started = 2026-07-14T21:58:14.969374+00:00
stopped = 2026-07-14T22:00:10.749247+00:00
time_taken = 115.780 s
result = 'Success. New document id 3 created'
```

Note the `started` (`21:58:14.969374`) is exactly the enqueue timestamp peeked inside the package in
§5.1 — proving the point from §3.5 that **`started` is stamped at enqueue**. As a single calculation:

```
time_taken = stopped − started = 2026-07-14T22:00:10.749247Z − 2026-07-14T21:58:14.969374Z
           = 115.779873 s  ≈  115.780 s
```

The job's *execution* was ≈ 0.7 s (worker `Consuming` `22:00:10,037` → `consumption finished`
`22:00:10,745` = `0.708 s`); the remaining ≈ 115.07 s is pure **queue-wait** — time the package spent in
the WAITING state because the cluster was deliberately down. This is the concrete proof that
`time_taken` = queue-wait + execution.

### 6.3 Run 2 — identical experiment (two-run stability)

Repeating the *identical* procedure with a second file yields the same three-state signature:

```console
$ docker exec pngx_qa bash -lc "cat /tmp/obs/poll_run2.csv"
# run_id=run2 env_id=pngx_qa/542221a38dff/redis7.4.9 watch_task_id=d050b8651a404292807516543bc88446 watch_name=qa_wait_run2_20260714_220232.txt
# iso_utc                    | LLEN | cluster  | task_q | done_q | state
2026-07-14T22:02:49.669+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:02:51.638+00:00 | 1    | down     | -    | -    | WAITING
2026-07-14T22:02:51.740+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:02:52.561+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-14T22:02:52.664+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
```
*(abridged to the state-transition boundaries; the full 32-sample trace is in `/tmp/obs/poll_run2.csv`
— it shows 20 consecutive WAITING samples `22:02:49.669→51.638`, then 9 IN-CLUSTER/ACTIVE samples
`22:02:51.740→52.561`, then 3 DONE samples.)*

Two-run comparison (the state pattern is stable; the difference is only the deliberate queue-wait):

| Measure | Run 1 | Run 2 |
|---|---|---|
| WAITING samples (`LLEN=1`, no row) | 18 | 20 |
| IN-CLUSTER/ACTIVE samples (`LLEN=0`, no row) | 8 | 9 |
| DONE samples | 3 | 3 |
| execution (`Consuming`→`consumption finished`) | 0.708 s | 0.750 s |
| `time_taken` (enqueue→finish = wait+exec) | 115.780 s | 18.697 s |
| `success` / `result` | True / "New document id 3 created" | True / "New document id 4 created" |

The qualitative boundary (WAITING → IN-CLUSTER/ACTIVE → DONE) and the execution duration (≈ 0.7 s) are
**stable across both runs**; only `time_taken` differs, and it differs *exactly* by how long each task
was left waiting — which is expected and reinforces the `started`-is-enqueue semantics.

### 6.4 What the admin/API surfaces for each state

- **WAITING** is only visible at the broker: `queue_size()` (i.e. `LLEN`) counts it. Django-Q's ORM-broker
  "Queued tasks" admin page (`OrmQ`) is **not** available here because this deployment uses the **Redis**
  broker — so there is no admin page listing waiting tasks; the Redis list is the source of truth.
  *(This matches the Django-Q admin design; see [Appendix](#appendix).)*
- **IN-CLUSTER/ACTIVE** has no persisted representation at all; it is reflected only transiently by the
  cluster `Stat` (`task_q`/`done_q`, worker state) and the worker log.
- **DONE** is the persisted `django_q_task` row, surfaced by the "Successful/Failed tasks" admin pages
  (see [§7](#7-q5--where-task-state-is-stored) / [§8](#8-q6--after-the-fact-status)).

> **(inferred)** The statement that `queue_size()`/`LLEN` *excludes* in-progress work is grounded both in
> the broker source (`queue_size` is a plain `LLEN`, `redis_broker.py:26`, and the pusher has already
> `BLPOP`ed the element out of the list before the worker runs) and directly **observed** above: every
> IN-CLUSTER/ACTIVE sample shows `LLEN=0` while the task is provably still running.

---


## 7. Q5 — Where task state is stored

**Question.** *Where does task state ultimately get persisted (which database table / model)?*

**Answer (exact, grounded).** Task state is persisted by Django-Q into **its own** table
**`django_q_task`**, mapped by the model **`django_q.models.Task`**. paperless-ngx defines **no** custom
task model — it relies entirely on Django-Q's built-in tables, which exist because `"django_q"` is listed
in `INSTALLED_APPS` (`src/paperless/settings.py:110`) and its migrations create the tables.

### 7.1 The model → table mapping (observed)

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task, Success, Failure, Schedule, OrmQ
print("Task._meta.db_table       =", Task._meta.db_table)
print("Task._meta.app_label      =", Task._meta.app_label)
print("Task fields               =", [f.name for f in Task._meta.get_fields()])
print("Success is proxy          =", Success._meta.proxy, "| base:", Success._meta.concrete_model.__name__)
print("Failure is proxy          =", Failure._meta.proxy, "| base:", Failure._meta.concrete_model.__name__)
print("Schedule._meta.db_table   =", Schedule._meta.db_table)
print("OrmQ._meta.db_table       =", OrmQ._meta.db_table)
PY
Task._meta.db_table       = django_q_task
Task._meta.app_label      = django_q
Task fields               = ['id', 'name', 'func', 'hook', 'args', 'kwargs', 'result', 'group', 'started', 'stopped', 'success', 'attempt_count']
Success is proxy          = True | base: Task
Failure is proxy          = True | base: Task
Schedule._meta.db_table   = django_q_schedule
OrmQ._meta.db_table       = django_q_ormq
```

So the durable task record is `django_q.models.Task` → table **`django_q_task`**, app label `django_q`,
with exactly these columns: **`id, name, func, hook, args, kwargs, result, group, started, stopped,
success, attempt_count`**. `Success` and `Failure` are **proxy** models over the same `Task` table
(covered in [§8](#8-q6--after-the-fact-status)); `Schedule` (`django_q_schedule`) holds recurring jobs
(covered in [§10](#10-scheduled--recurring-jobs)); `OrmQ` (`django_q_ormq`) is the ORM-broker queue table,
unused here because this deployment uses the Redis broker.

### 7.2 There is no custom paperless task model (observed)

Every model class in `documents/models.py` is a domain model — none is task-related:

```console
$ docker exec pngx_qa bash -lc "grep -nE '^class .*\(models\.Model\)|^class .*\(MatchingModel\)' /app/src/documents/models.py"
19:class MatchingModel(models.Model):
57:class Correspondent(MatchingModel):
64:class Tag(MatchingModel):
82:class DocumentType(MatchingModel):
88:class Document(models.Model):
285:class Log(models.Model):
316:class SavedView(models.Model):
342:class SavedViewFilterRule(models.Model):

$ docker exec pngx_qa bash -lc "grep -rnE 'class .*Task.*\(models\.Model\)' /app/src/documents /app/src/paperless /app/src/paperless_mail 2>/dev/null || echo 'NO custom Task model in paperless app code'"
NO custom Task model in paperless app code
```

The domain models are `MatchingModel, Correspondent, Tag, DocumentType, Document, Log, SavedView,
SavedViewFilterRule` — confirming the convention: **paperless-ngx stores task state only in Django-Q's
tables**, never a bespoke one.

### 7.3 The tables exist because `django_q` is installed (observed)

```console
$ docker exec pngx_qa bash -lc "grep -nE '\"django_q\"' /app/src/paperless/settings.py"
110:    "django_q",

$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.db import connection
print("DB engine:", connection.vendor, "| path:", connection.settings_dict.get("NAME"))
with connection.cursor() as c:
    c.execute("SELECT sql FROM sqlite_master WHERE type='table' AND name='django_q_task';")
    print(c.fetchone()[0])
    c.execute("SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'django_q%';")
    print("django_q tables:", [r[0] for r in c.fetchall()])
PY
DB engine: sqlite | path: /tmp/pp/data/db.sqlite3
CREATE TABLE "django_q_task" ("name" varchar(100) NOT NULL, "func" varchar(256) NOT NULL, "hook" varchar(256) NULL, "args" text NULL, "kwargs" text NULL, "result" text NULL, "started" datetime NOT NULL, "stopped" datetime NOT NULL, "success" bool NOT NULL, "id" varchar(32) NOT NULL PRIMARY KEY, "group" varchar(100) NULL, "attempt_count" integer NOT NULL)
django_q tables: ['django_q_ormq', 'django_q_task', 'django_q_schedule']
```

The physical `django_q_task` table (here in the canonical default **SQLite** DB at
`/tmp/pp/data/db.sqlite3`) has `id` as the 32-char `PRIMARY KEY` — the same id `async_task` returns and
that the package carried in [§5](#5-q3--how-a-job-appears-when-created). The three Django-Q tables
present are `django_q_ormq`, `django_q_task`, `django_q_schedule`.

> **(inferred)** That the *migrations of the `django_q` app* are what physically create these tables is
> inferred from Django's standard app/migration mechanics (the app is in `INSTALLED_APPS` and `migrate`
> was run in [§2](#2-environment--exact-reproduction)); the table's **existence and exact schema** are
> **observed** above.

---

## 8. Q6 — After-the-fact status

**Question.** *After a job finishes, how can an operator determine what happened to it (result,
success/failure, timing, history)?*

**Answer (mechanism).** There are **two distinct** channels, and they must not be conflated:

- **Durable / after-the-fact:** the persisted **`django_q_task` row**. Its fields answer every part of
  the question — `success` (succeeded or not), `result` (return value on success, **full traceback** on
  failure), `started`/`stopped`/`time_taken()` (timing), and the row's mere existence + `id`/`name`/`func`
  (history). Operators read this via the Django admin **"Successful tasks"** (`Success`) and **"Failed
  tasks"** (`Failure`) pages, which are proxy views over the same table.
- **Realtime / live-only:** the Channels **WebSocket** `status_updates` stream. This is a *transient*
  progress feed the front end consumes **while** the job runs; it is **not** persisted and is gone once
  the job ends. It answers "what is happening now", not "what happened".

### 8.1 A completed **success** — the full row (observed)

Reading back the seed document's task — the successful half of the success/failure pair (the same seed
whose byte-identical duplicate is failed in §8.2):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(name="qa_seed_20260714_221848.txt")
for f in ["id","name","func","hook","group","args","kwargs","result","started","stopped","success","attempt_count"]:
    print("  %-14s = %r" % (f, getattr(t, f)))
print("  time_taken()   = %.3f s" % t.time_taken())
PY
  id             = 'aaf0ee80efc64eb2ad9496ff8bbbd43d'
  name           = 'qa_seed_20260714_221848.txt'
  func           = 'documents.tasks.consume_file'
  hook           = None
  group          = None
  args           = ('/tmp/pp/consume/qa_seed_20260714_221848.txt',)
  kwargs         = {'override_tag_ids': None}
  result         = 'Success. New document id 5 created'
  started        = datetime.datetime(2026, 7, 14, 22, 18, 50, 79230, tzinfo=datetime.timezone.utc)
  stopped        = datetime.datetime(2026, 7, 14, 22, 18, 51, 4842, tzinfo=datetime.timezone.utc)
  success        = True
  attempt_count  = 1
  time_taken()   = 0.926 s
```

Every part of the question is answered by this one row: **success** = `True`; **result** = the string the
task returned (`consume_file` returns `"Success. New document id {pk} created"`, `documents/tasks.py:247`);
**timing** = `started`/`stopped` and `time_taken() = 0.926 s`; **history/identity** = `id`, `name`, `func`,
`attempt_count = 1`.

### 8.2 A completed **failure** — the complete row with full traceback (observed)

A **real** failure was forced through the canonical watcher entry point by dropping a file whose content
is **byte-identical** to an already-ingested document (same md5), which trips the consumer's duplicate
pre-check. Its worker log line and the **complete, untruncated** failed row:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_dup_20260714_221924' /tmp/obs/qcluster.log"
165:22:19:25 [Q] INFO Process-1:6 processing [qa_dup_20260714_221924.txt]
166:[2026-07-14 22:19:25,359] [ERROR] [paperless.consumer] Not consuming qa_dup_20260714_221924.txt: It is a duplicate.
168:22:19:25 [Q] ERROR Failed [qa_dup_20260714_221924.txt] - qa_dup_20260714_221924.txt: Not consuming qa_dup_20260714_221924.txt: It is a duplicate. : Traceback (most recent call last):
179:documents.consumer.ConsumerError: qa_dup_20260714_221924.txt: Not consuming qa_dup_20260714_221924.txt: It is a duplicate.

$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task, Success, Failure
t = Task.objects.get(name="qa_dup_20260714_221924.txt")
for f in ["id","name","func","hook","group","args","kwargs","started","stopped","success","attempt_count"]:
    print("  %-14s = %r" % (f, getattr(t, f)))
print("  time_taken()   = %.3f s" % t.time_taken())
print("  --- result (COMPLETE, unedited traceback) ---")
print(t.result)
print("  --- end result ---")
print("Failure proxy contains this id? ", Failure.objects.filter(id=t.id).exists())
print("Success proxy contains this id? ", Success.objects.filter(id=t.id).exists())
PY
  id             = '1052faf67c114cbcbcfc1b2343d152dc'
  name           = 'qa_dup_20260714_221924.txt'
  func           = 'documents.tasks.consume_file'
  hook           = None
  group          = None
  args           = ('/tmp/pp/consume/qa_dup_20260714_221924.txt',)
  kwargs         = {'override_tag_ids': None}
  started        = datetime.datetime(2026, 7, 14, 22, 19, 25, 204246, tzinfo=datetime.timezone.utc)
  stopped        = datetime.datetime(2026, 7, 14, 22, 19, 25, 360779, tzinfo=datetime.timezone.utc)
  success        = False
  attempt_count  = 1
  time_taken()   = 0.157 s
  --- result (COMPLETE, unedited traceback) ---
qa_dup_20260714_221924.txt: Not consuming qa_dup_20260714_221924.txt: It is a duplicate. : Traceback (most recent call last):
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
documents.consumer.ConsumerError: qa_dup_20260714_221924.txt: Not consuming qa_dup_20260714_221924.txt: It is a duplicate.

  --- end result ---
Failure proxy contains this id?  True
Success proxy contains this id?  False
```

For a failure, `success = False` and — crucially — **`result` holds the complete Python traceback**
(here 812 characters, shown in full above; nothing is truncated). The traceback itself is a precise
after-the-fact record of *what happened and where*: the worker called the task at
`django_q/cluster.py:432` (`res = f(*task["args"], **task["kwargs"])`), which entered
`consume_file` (`documents/tasks.py:236`) → `Consumer.try_consume_file` (`consumer.py:213`) →
`pre_check_duplicate` (`consumer.py:110`) → `_fail` (`consumer.py:81`) raising
`documents.consumer.ConsumerError: … It is a duplicate.` The same row is classified into the **`Failure`**
proxy (`True`) and excluded from **`Success`** (`False`).

### 8.3 The admin surfaces: Success / Failure proxies (observed)

The two "history" admin pages are proxy models over `django_q_task`, filtered by the `success` flag:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'class SuccessManager|class FailureManager|class Success\(Task\)|class Failure\(Task\)|get_queryset\(\).filter\(success=|proxy = True' /usr/local/lib/python3.9/site-packages/django_q/models.py"
108:class SuccessManager(models.Manager):
110:        return super(SuccessManager, self).get_queryset().filter(success=True)
113:class Success(Task):
121:        proxy = True
124:class FailureManager(models.Manager):
126:        return super(FailureManager, self).get_queryset().filter(success=False)
129:class Failure(Task):
137:        proxy = True

$ docker exec pngx_qa bash -lc "grep -nE 'admin.site.register|class .*Admin|if Conf.ORM' /usr/local/lib/python3.9/site-packages/django_q/admin.py"
10:class TaskAdmin(admin.ModelAdmin):
43:class FailAdmin(admin.ModelAdmin):
62:class ScheduleAdmin(admin.ModelAdmin):
86:class QueueAdmin(admin.ModelAdmin):
107:admin.site.register(Schedule, ScheduleAdmin)
108:admin.site.register(Success, TaskAdmin)
109:admin.site.register(Failure, FailAdmin)
111:if Conf.ORM or Conf.TESTING:
112:    admin.site.register(OrmQ, QueueAdmin)
```

`SuccessManager` filters `success=True` (`models.py:110`) and `FailureManager` filters `success=False`
(`models.py:126`); both `Success` and `Failure` set `proxy = True` (`models.py:121,137`). The admin
registers **`Success`→`TaskAdmin`** ("Successful tasks") and **`Failure`→`FailAdmin`** ("Failed tasks")
unconditionally (`admin.py:108-109`), but registers **`OrmQ`→`QueueAdmin`** ("Queued tasks") **only**
`if Conf.ORM or Conf.TESTING` (`admin.py:111-112`). Observed live, that condition is false here:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.conf import Conf
from django.contrib import admin
from django_q.models import OrmQ, Success, Failure
print("Conf.ORM =", Conf.ORM, "(None => Redis broker, not ORM)")
print("OrmQ registered in admin?    ", OrmQ in admin.site._registry)
print("Success registered in admin? ", Success in admin.site._registry)
print("Failure registered in admin? ", Failure in admin.site._registry)
print("live counts -> Success:", Success.objects.count(), "Failure:", Failure.objects.count())
PY
Conf.ORM = None (None => Redis broker, not ORM)
OrmQ registered in admin?     False
Success registered in admin?  True
Failure registered in admin?  True
live counts -> Success: 15 Failure: 1
```

So with the **Redis** broker, the admin shows "Successful tasks" and "Failed tasks" but **not** "Queued
tasks" — waiting work has no admin page here (as noted in [§6.4](#64-what-the-adminapi-surfaces-for-each-state)).
The single `Failure` is precisely the duplicate forced in §8.2.

### 8.4 The realtime channel: authenticated WebSocket status stream (observed)

Distinct from the persisted row, the front end watches progress live over a WebSocket. The endpoint is
`ws/status/` (`src/paperless/urls.py:137`), routed through `AuthMiddlewareStack`
(`src/paperless/asgi.py:20`) so the Django **session cookie** authenticates the socket. `StatusConsumer`
**denies** unauthenticated connections (`_authenticated()` → `DenyConnection()`,
`src/paperless/consumers.py:10-15`). That denial is observable — an unauthenticated handshake is rejected
with **HTTP 403**:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import asyncio, websockets
async def go():
    try:
        async with websockets.connect("ws://localhost:8000/ws/status/") as ws:
            print("connected WITHOUT auth (unexpected)")
    except Exception as e:
        print("unauth connect result:", type(e).__name__, str(e)[:120])
asyncio.get_event_loop().run_until_complete(go())
PY
unauth connect result: InvalidStatusCode server rejected WebSocket connection: HTTP 403
```

With a **real session login** (the CSRF-protected `/admin/login/` flow) the handshake succeeds and the
socket receives the live `status_updates` frames for an ingestion. The complete capture script is
`/tmp/pp/ws_capture.py` (56 lines, published in [§13 Appendix](#appendix)); its full run:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export PP_USER=admin PP_PASS=<REDACTED>; cd /app/src && python3 /tmp/pp/ws_capture.py'
LOGIN GET /admin/login/  -> HTTP 200, csrftoken cookie present=True
LOGIN POST /admin/login/ -> HTTP 302 (302=success), sessionid cookie present=True
  auth cookie sent to WS handshake:  Cookie: sessionid=<REDACTED>  (real 32-char value redacted)
WS 2026-07-14T22:22:36.021+00:00 CONNECTED (HTTP 101 Switching Protocols) — authenticated accept
WS 2026-07-14T22:22:36.027+00:00 dropped /tmp/pp/consume/qa_ws_20260714_222236.txt to trigger ingestion
WS FRAME 1 2026-07-14T22:22:37.179+00:00  {"current_progress": 0, "document_id": null, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "new_file", "status": "STARTING", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS FRAME 2 2026-07-14T22:22:37.193+00:00  {"current_progress": 20, "document_id": null, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "parsing_document", "status": "WORKING", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS FRAME 3 2026-07-14T22:22:37.196+00:00  {"current_progress": 70, "document_id": null, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "generating_thumbnail", "status": "WORKING", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS FRAME 4 2026-07-14T22:22:37.878+00:00  {"current_progress": 90, "document_id": null, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "parse_date", "status": "WORKING", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS FRAME 5 2026-07-14T22:22:37.881+00:00  {"current_progress": 95, "document_id": null, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "save_document", "status": "WORKING", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS FRAME 6 2026-07-14T22:22:37.967+00:00  {"current_progress": 100, "document_id": 6, "filename": "qa_ws_20260714_222236.txt", "max_progress": 100, "message": "finished", "status": "SUCCESS", "task_id": "ca38d192-8d48-49fb-a3bd-12ec6d684538"}
WS 2026-07-14T22:22:37.967+00:00 terminal status 'SUCCESS' received; stopping
```

The six frames are exactly the `_send_progress(...)` calls in the pipeline (`documents/consumer.py`):
`STARTING` (0%, `:202`) → `WORKING` (20% parsing `:259`, 70% thumbnail `:264`, 90% parse-date `:274`,
95% save `:294`) → `SUCCESS` (100%, `document_id=6`, `:375`). Each frame is the JSON `payload` built at
`consumer.py:56-72` (`filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`,
`document_id`).

> **Key distinction (cause of common confusion).** The WebSocket frame's `task_id`
> (`ca38d192-8d48-49fb-a3bd-12ec6d684538`, a UUID) is the **front-end progress id**, *not* the
> `django_q_task` primary key (a 32-char hex like `aaf0ee80…`). The realtime stream and the durable row
> are keyed differently and serve different purposes; see [§9](#9-q7--enqueue-origin-in-code) for where the
> progress UUID is generated. **After the fact**, the authoritative record is the `django_q_task` row —
> the WebSocket frames are already gone.

---

## 9. Q7 — Enqueue origin in code

**Direct answer.** Background jobs are placed on the queue by exactly one primitive —
`django_q.tasks.async_task(<dotted-task-path>, …)` — called from **four** origins in the paperless-ngx
source. The three *ingestion* origins all enqueue the **same** task path,
`"documents.tasks.consume_file"`; the *bulk* origin enqueues `"documents.tasks.bulk_update_documents"`.
Every origin below was exercised **live** through its canonical entry point and correlated to a real
`django_q_task` row.

| # | Origin (canonical entry point) | Enqueue call site (`file:line`) | Task path enqueued |
|---|--------------------------------|----------------------------------|--------------------|
| 1 | Directory watcher (`document_consumer`) | `src/documents/management/commands/document_consumer.py:86` | `documents.tasks.consume_file` |
| 2 | REST upload (`PostDocumentView.post`) | `src/documents/views.py:523` | `documents.tasks.consume_file` |
| 3 | Mail fetch (`MailAccountHandler.handle_message`) | `src/paperless_mail/mail.py:336` | `documents.tasks.consume_file` |
| 4 | Bulk edit (five `bulk_edit` functions) | `src/documents/bulk_edit.py:18,31,47,63,87` | `documents.tasks.bulk_update_documents` |

### 9.1 Origin 1 — the directory watcher (observed in §3 and §5)

The watcher is the origin already traced end-to-end in [§3](#3-q1--live-async-behavior) (worker execution)
and [§5](#5-q3--how-a-job-appears-when-created) (raw package). The enqueue is two source lines — a log line
then the call **(source)**:

```console
$ docker exec pngx_qa sed -n '85,91p' /app/src/documents/management/commands/document_consumer.py
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
            filepath,
            override_tag_ids=tag_ids if tag_ids else None,
            task_name=os.path.basename(filepath)[:100],
        )
```

The `_consume` enqueue function (`document_consumer.py:46`) is invoked by the `watchdog`
`FileSystemEventHandler` (`Handler`, `:128`) that `Command.handle` (`:156`) installs on the consumption
directory. The matching **observed** log line — `Adding … to the task queue.` — appears in the watcher's
own output in [§3.2](#32-trigger-hop-1-drop-a-file-through-the-real-directory-watcher). Note this origin
passes **no** `task_id` kwarg, so the worker's WebSocket progress `task_id` is `None` for watcher drops
(contrast Origin 2).

### 9.2 Origin 2 — REST upload (`PostDocumentView.post`), exercised live

**Source contract (observed via `sed`).** The view requires authentication, accepts multipart, mints a
progress UUID, enqueues, and returns the string `"OK"`:

```console
$ docker exec pngx_qa sed -n '491,495p' /app/src/documents/views.py
class PostDocumentView(GenericAPIView):

    permission_classes = (IsAuthenticated,)
    serializer_class = PostDocumentSerializer
    parser_classes = (parsers.MultiPartParser,)
$ docker exec pngx_qa sed -n '521,535p' /app/src/documents/views.py
        task_id = str(uuid.uuid4())

        async_task(
            "documents.tasks.consume_file",
            temp_filename,
            override_filename=doc_name,
            override_title=title,
            override_correspondent_id=correspondent_id,
            override_document_type_id=document_type_id,
            override_tag_ids=tag_ids,
            task_id=task_id,
            task_name=os.path.basename(doc_name)[:100],
        )

        return Response("OK")
```

Anchors: `permission_classes = (IsAuthenticated,)` (`views.py:493`), `MultiPartParser` (`:495`),
`task_id = str(uuid.uuid4())` (`:521`), the enqueue (`:523`), and `return Response("OK")` (`:535`).

**Live run (session + CSRF auth; credentials redacted, mechanics intact).** The full helper is
`/tmp/pp/rest_upload.py` (published in [§13 Appendix](#appendix)); the browser-canonical authentication is
a CSRF-protected session login, then a multipart POST carrying the `sessionid` cookie and the
`X-CSRFToken` header:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export QA_ADMIN_PW=<REDACTED>; cd /app/src && python3 /tmp/pp/rest_upload.py'
STEP1 GET /accounts/login/ -> 200 ; csrftoken cookie present=True
STEP2 POST /accounts/login/ -> 302 ; sessionid cookie present=True

REQUEST  POST http://localhost:8000/api/documents/post_document/
  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>
  multipart field 'document' = (qa_rest_20260714_225256.txt, 84 bytes, text/plain)
RESPONSE status = 200
RESPONSE content-type = application/json
RESPONSE body = '"OK"'

correlating (waiting for worker to consume the REST upload)...
TASK id=3bfecaf91deb42beac17728578936669 (hex, Django-Q PK)
  func=documents.tasks.consume_file name=qa_rest_20260714_225256.txt success=True time_taken=1.243s
  result='Success. New document id 8 created'
DOCUMENT id=8 title='qa_rest_20260714_225256' content='PAPERLESS_QA_REST_UPLOAD unique content 20260714_225256 for the REST ingestion path.'
```

**Cause → effect.** The `302` on the login POST is Django's success redirect; it is what deposits the
`sessionid` cookie the session uses thereafter. The multipart POST authenticated by that cookie (plus the
CSRF header required for unsafe methods under `SessionAuthentication`) reaches `PostDocumentView.post`,
which writes the upload to a scratch temp file, calls `async_task("documents.tasks.consume_file", …)`
(`views.py:523`), and returns `Response("OK")`. The observed response body is therefore the JSON string
`"OK"` — **not** a task id.

**The response contract and the two distinct UUIDs (resolves the common mislabel).** The endpoint returns
neither identifier. There are two different UUIDs in play:

- The **Django-Q Task primary key** is generated *inside* `async_task` by `django_q`, independent of
  paperless. `async_task` calls `tag = uuid()` and sets `task["id"] = tag[1]`
  (`django_q/tasks.py:38,41`), then `return task["id"]` (`:76`). `humanhash.uuid()` returns
  `str(uuid.uuid4()).replace("-", "")` (`django_q/humanhash.py:358`) — a **32-char hex with no hyphens**.
  This is the value stored as `django_q_task.id`; the observed PK above is `3bfecaf91deb42beac17728578936669`.
- The **front-end progress UUID** is a `uuid4()` used only for the realtime stream. For a REST upload it
  is minted as `task_id = str(uuid.uuid4())` (`views.py:521`, **hyphenated**) and passed as the `task_id`
  **kwarg** to the enqueued `consume_file` (`views.py:531`). `consume_file(…, task_id=None)`
  (`documents/tasks.py:191`) forwards it to `Consumer.try_consume_file(…, task_id=task_id)`
  (`documents/tasks.py:236-244`), which sets `self.task_id = task_id or str(uuid.uuid4())`
  (`documents/consumer.py:200`); `Consumer._send_progress` (`consumer.py:56`) then emits it in every
  `status_updates` frame as `"task_id": self.task_id` (`consumer.py:66`). The
  [§8.4](#84-the-realtime-channel-authenticated-websocket-status-stream-observed) frames were captured for a
  **watcher** drop, which supplies no `task_id`, so there the Consumer generated its *own* `uuid4()` at
  `consumer.py:200` — still hyphenated (`ca38d192-8d48-49fb-a3bd-12ec6d684538`), illustrating the same
  distinction. Either way the progress id is a hyphenated `uuid4`, never the 32-char hex Task PK.

So the durable record (hex PK) and the realtime progress stream (hyphenated UUID) are keyed by **different**
identifiers, and `Response("OK")` returns neither. The REST client observes progress on the WebSocket and
the final outcome via the `django_q_task` row.

### 9.3 Origin 3 — mail fetch (`handle_message`), exercised live against a real IMAP server

This origin was **run live** (not merely source-cited). Because the canonical mail path performs a real
IMAP `LOGIN → SELECT → SEARCH → FETCH`, a **real minimal IMAP4 server** (Twisted, published in
[§13 Appendix](#appendix) as `/tmp/pp/imap_server.py`) was stood up on `127.0.0.1:10143` serving one
plaintext account whose INBOX holds one RFC822 message carrying a `.txt` attachment. paperless connects to
it with its **real** `imap_tools` client — no mock, stub, or synthetic task stand-in.

**Source contract (observed via `sed`).** `handle_message` validates the attachment's true MIME type,
writes it to a scratch temp file, logs a `Consuming attachment …` line, then enqueues:

```console
$ docker exec pngx_qa sed -n '329,349p' /app/src/paperless_mail/mail.py
                self.log(
                    "info",
                    f"Rule {rule}: "
                    f"Consuming attachment {att.filename} from mail "
                    f"{message.subject} from {message.from_}",
                )

                async_task(
                    "documents.tasks.consume_file",
                    path=temp_filename,
                    override_filename=pathvalidate.sanitize_filename(
                        att.filename,
                    ),
                    override_title=title,
                    override_correspondent_id=correspondent.id
                    if correspondent
                    else None,
                    override_document_type_id=doc_type.id if doc_type else None,
                    override_tag_ids=tag_ids,
                    task_name=att.filename[:100],
                )
```

The seeded schedule's canonical callable is the **plural** `process_mail_accounts`
(`src/paperless_mail/tasks.py:11`), which loops over every configured `MailAccount` and returns a summary
string (e.g. `'No new documents were added.'`). The reproduction below drives the **single-account
convenience wrapper** `process_mail_account` (`src/paperless_mail/tasks.py:25`) for the one QA account (it
returns `None` — see the `RETURN` line below); both converge on the identical chain →
`MailAccountHandler.handle_mail_account` (`mail.py:151`, does
`get_mailbox(...)` + `M.login(account.username, account.password)`) → `handle_mail_rule` (`mail.py:187`,
does `M.folder.set(rule.folder)` then `M.fetch(criteria=AND(**criterias), mark_seen=False, …)`,
`mail.py:222`) → `handle_message` (`mail.py:272`) → the enqueue at `mail.py:336`. For
`ImapSecurity.NONE` the client is `MailBoxUnencrypted` (`mail.py:94`).

**Live run with the cluster intentionally stopped** (so the enqueued package waits in Redis and can be
peeked non-destructively, mirroring [§5](#5-q3--how-a-job-appears-when-created)):

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    cd /app/src && python3 /tmp/pp/mail_run.py'
ACCT id=1 security=1(No encryption) host=127.0.0.1:10143 user=qauser
RULE id=1 folder=INBOX maximum_age=0 action=3(Mark as read, don't process read mails)
BEFORE  LLEN(django_q:paperless:q)=0  tasks=18  documents=6
>>> invoking single-account convenience wrapper: process_mail_account('qa_mail')
[2026-07-14 22:49:50,506] [INFO] [paperless_mail] Rule qa_mail.qa_rule: Consuming attachment qa_mail_attachment.txt from mail PaperlessQA live mail ingestion test from qa-sender@example.test
22:49:50 [Q] INFO Enqueued 1
RETURN process_mail_account = None
AFTER   LLEN(django_q:paperless:q)=1  tasks=18  documents=6
LINDEX 0 raw bytes length = 650
PACKAGE keys       = ['args', 'func', 'id', 'kwargs', 'name', 'started']
PACKAGE id         = 12b9c63a07bc4e04947b4e3ed9f48379
PACKAGE func       = documents.tasks.consume_file
PACKAGE name       = qa_mail_attachment.txt
PACKAGE args       = ()
PACKAGE kwargs.override_filename = qa_mail_attachment.txt
PACKAGE kwargs.override_title    = PaperlessQA live mail ingestion test
PACKAGE kwargs.task_name (via name/kw) = qa_mail_attachment.txt
PACKAGE kwargs.path exists       = True -> /tmp/pp/scratch/paperless-mail-qypumx9l
```

**Cause → effect (the attachment log and the raw package).** The `Consuming attachment
qa_mail_attachment.txt from mail …` line is the `self.log("info", …)` at `mail.py:329`, emitted the instant
the enqueue at `mail.py:336` fires; Django-Q's own `Enqueued 1` confirms the RPUSH, and `LLEN` moves
`0 → 1`. The peeked package (the same signed-pickle format dissected in
[§5.2](#52-why-the-payload-is-opaque-bytes--the-signed-pickle-trust-boundary)) proves the enqueued
`func` is `documents.tasks.consume_file` and carries the mail-specific kwargs — `override_filename` and
`override_title` from the message, and `path` pointing at the scratch temp file the mail handler wrote
(`/tmp/pp/scratch/paperless-mail-…`).

**Bringing the cluster back completes the job; the Task row and new Document correlate by id and time:**

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; cd /app/src && python3 -' <<'PY'
import os, django; os.environ.setdefault("DJANGO_SETTINGS_MODULE","paperless.settings"); django.setup()
from django_q.models import Task; from documents.models import Document
t=Task.objects.get(id="12b9c63a07bc4e04947b4e3ed9f48379")
print("TASK", t.id, t.func, t.name, "success="+str(t.success),
      "started="+str(t.started), "stopped="+str(t.stopped),
      "time_taken=%.3fs"%t.time_taken(), "result="+repr(t.result))
d=Document.objects.get(id=7)
print("DOC", d.id, repr(d.title), repr(d.content))
PY
TASK 12b9c63a07bc4e04947b4e3ed9f48379 documents.tasks.consume_file qa_mail_attachment.txt success=True started=2026-07-14 22:49:50.509235+00:00 stopped=2026-07-14 22:50:18.654658+00:00 time_taken=28.145s result='Success. New document id 7 created'
DOC 7 'PaperlessQA live mail ingestion test' 'PAPERLESS_QA_MAIL_ATTACHMENT unique content for the mail ingestion path.'
```

The worker log for the mail attachment (from the fresh cluster's `qcluster.log`):

```console
$ docker exec pngx_qa grep -E 'processing \[qa_mail|Consuming qa_mail|Processed \[qa_mail' /tmp/obs/qcluster.log
22:50:17 [Q] INFO Process-1:1 processing [qa_mail_attachment.txt]
[2026-07-14 22:50:17,872] [INFO] [paperless.consumer] Consuming qa_mail_attachment.txt
22:50:18 [Q] INFO Processed [qa_mail_attachment.txt]
```

**Interpreting `time_taken=28.145s`.** This is *not* 28 s of work. `Task.started` is the **enqueue**
timestamp (`22:49:50.509`, established in [§6](#6-q4--waiting-vs-actively-processing)) and `Task.stopped`
is the worker-finish timestamp (`22:50:18.654`). The package sat in Redis while the cluster was
deliberately stopped; the worker log shows the actual execution was `22:50:17 → 22:50:18` (~1 s). This
independently re-confirms the Q4 timing semantics (`time_taken` = queue-wait + execution) on a second,
different task.

**The `MARK_READ` post-consume action is observable too.** The rule's default action
(`MailAction.MARK_READ`, `models.py:150`) runs `get_rule_action(rule).post_consume(...)` in
`handle_mail_rule` (`mail.py:259`) *after* enqueue, storing `\Seen` on the processed message so it is not
re-fetched. Re-querying the mailbox with the real client shows the message is now seen:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; cd /app/src && python3 -' <<'PY'
from imap_tools import MailBoxUnencrypted, AND
with MailBoxUnencrypted("127.0.0.1",10143).login("qauser","<REDACTED>") as mb:
    print("UNSEEN now =", len(list(mb.fetch(AND(seen=False), mark_seen=False))))
    allm=list(mb.fetch(AND(all=True), mark_seen=False))
    print("ALL =", len(allm), "flags:", [list(m.flags) for m in allm])
PY
UNSEEN now = 0
ALL = 1 flags: [['\\Seen']]
```

### 9.4 Origin 4 — bulk edit, exercised live

The five bulk operations each enqueue `bulk_update_documents` **(source)**:

```console
$ docker exec pngx_qa grep -nE 'def |async_task' /app/src/documents/bulk_edit.py
4:from django_q.tasks import async_task
10:def set_correspondent(doc_ids, correspondent):
18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
23:def set_document_type(doc_ids, document_type):
31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
36:def add_tag(doc_ids, tag):
47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
52:def remove_tag(doc_ids, tag):
63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
68:def modify_tags(doc_ids, add_tags, remove_tags):
87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
92:def delete(doc_ids):
```

(Note `delete` at `:92` performs its work synchronously and does **not** enqueue — only the five above are
enqueue origins.) The canonical entry point is the REST `BulkEditView` (`views.py:469`), which requires
`IsAuthenticated` (`:471`), parses **JSON** (`:473`), dispatches `method(documents, **parameters)`
(`:485`), and returns `Response({"result": result})` (`:486`). The worker task
`bulk_update_documents(document_ids)` (`documents/tasks.py:270`) re-indexes the affected documents in
Whoosh (`index.update_document`, `:280`).

**Live run (JSON body; session + CSRF auth):**

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export QA_ADMIN_PW=<REDACTED>; cd /app/src && python3 /tmp/pp/bulk_edit.py'
SETUP tag id=3 name=qa_bulk_tag ; target documents=[8, 7]
LOGIN -> 302 ; sessionid present=True

REQUEST  POST http://localhost:8000/api/documents/bulk_edit/
  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>
  Content-Type: application/json
  body = {"documents": [8, 7], "method": "add_tag", "parameters": {"tag": 3}}
RESPONSE status = 200
RESPONSE body   = '{"result":"OK"}'

correlating (waiting for bulk_update_documents task)...
TASK id=68da20dfac30425f995562b5f883c9b4 func=documents.tasks.bulk_update_documents success=True time_taken=0.179s result=None
EFFECT documents now carrying tag 3: [8, 7]
```

**Cause → effect.** The JSON POST reaches `BulkEditView.post`, whose serializer resolves `method` to
`documents.bulk_edit.add_tag`. `add_tag` writes the tag rows and then calls
`async_task("documents.tasks.bulk_update_documents", document_ids=[8,7])` (`bulk_edit.py:47`) and returns
`"OK"` — which the view wraps as `{"result": "OK"}` (the observed body). The worker then runs
`bulk_update_documents`, whose `result` is `None` (it re-indexes rather than creating a document). The
`EFFECT` line confirms documents 8 and 7 now carry tag 3, proving the enqueued re-index actually executed.

### 9.5 Convergence and coverage

The single enqueue primitive is `django_q.tasks.async_task(<dotted-path>, *args, **kwargs)`. Whatever the
origin, the call builds a task dict, mints the hex id (`django_q/tasks.py:38,41`), signs+pickles it
(`SignedPackage.dumps`, `:69`), `RPUSH`es it onto `django_q:paperless:q` via `broker.enqueue` (`:73`), and
returns the hex id (`:76`) — the exact mechanism dissected for the watcher in
[§5](#5-q3--how-a-job-appears-when-created). The three ingestion origins converge on one worker function,
`documents.tasks.consume_file`; the bulk origin targets `documents.tasks.bulk_update_documents`.

**Condition coverage.** Every *required canonical* enqueue origin was exercised live — watcher, REST
upload, real IMAP mail fetch, and bulk edit — and each was correlated to a real `django_q_task` row, with
the REST `Response("OK")` contract and the mail `MARK_READ` post-action observed directly. The only
enqueue-adjacent variant **not** run is the barcode-separated split inside `consume_file`
(`documents/tasks.py:196-234`), which is gated off by the default
`PAPERLESS_CONSUMER_ENABLE_BARCODES=false`; per that default it is described from source and explicitly
labeled **(inferred)** in [§3.6](#36-edge-conditions-observed-on-the-live-path) rather than claimed as
observed.

---

## 10. Scheduled / recurring jobs

paperless-ngx runs four **recurring** background jobs. They are not registered in application code at
startup; they are seeded as `django_q.models.Schedule` rows by three data migrations, and the running
`qcluster` **Sentinel** fires them when they come due using the very same `async_task(...)` enqueue
primitive dissected in [§5](#5-q3--how-a-job-appears-when-created). This section shows the seeded rows,
then observes the scheduler firing one **live** (before/during/after), then measures cadence stability
across ≥2 intervals.

### 10.1 The four seeded schedules (observed)

The schedules are created by data migrations, so they exist after `manage.py migrate`
([§2.4](#24-migrate-create-users-as-testuser)) — no application code registers them at boot:

| Schedule name | `func` | Type | Interval | Seeding migration (`schedule(...)`) |
|---|---|---|---|---|
| Train the classifier | `documents.tasks.train_classifier` | `H` HOURLY | 1 h | `src/documents/migrations/1001_auto_20201109_1636.py:10` (`schedule_type=Schedule.HOURLY`, `:13`) |
| Optimize the index | `documents.tasks.index_optimize` | `D` DAILY | 1 d | `src/documents/migrations/1001_auto_20201109_1636.py:15` (`schedule_type=Schedule.DAILY`, `:18`) |
| Perform sanity check | `documents.tasks.sanity_check` | `W` WEEKLY | 1 w | `src/documents/migrations/1004_sanity_check_schedule.py:10` (`schedule_type=Schedule.WEEKLY`, `:13`) |
| Check all e-mail accounts | `paperless_mail.tasks.process_mail_accounts` | `I` MINUTES | 10 min | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10` (`schedule_type=Schedule.MINUTES`, `minutes=10`, `:13-14`) |

Observed live (this snapshot is taken *after* the firings shown in §10.2–§10.3, which is why `id=1` and
`id=4` already reflect them):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Schedule
print("Schedule rows:", Schedule.objects.count())
for s in Schedule.objects.order_by("id"):
    print(f"id={s.id} func={s.func} type={s.schedule_type} minutes={s.minutes} repeats={s.repeats} next_run={s.next_run.isoformat()} name={s.name!r}")
PY
Schedule rows: 4
id=1 func=documents.tasks.train_classifier type=H minutes=None repeats=-4 next_run=2026-07-15T00:09:00+00:00 name='Train the classifier'
id=2 func=documents.tasks.index_optimize type=D minutes=None repeats=-2 next_run=2026-07-15T21:13:15.406512+00:00 name='Optimize the index'
id=3 func=documents.tasks.sanity_check type=W minutes=None repeats=-2 next_run=2026-07-21T21:13:15.509431+00:00 name='Perform sanity check'
id=4 func=paperless_mail.tasks.process_mail_accounts type=I minutes=10 repeats=-14 next_run=2026-07-14T23:23:16.761616+00:00 name='Check all e-mail accounts'
```

The `type` column holds Django-Q's single-letter `Schedule.TYPE` codes (`H`/`D`/`W`/`I` =
HOURLY/DAILY/WEEKLY/MINUTES). Two observed facts matter:

- **`repeats` is negative** (−2, −4, −14), below the seeded value of `−1`. A seeded schedule starts at
  `repeats = -1` (Django-Q's "repeat forever" sentinel); each firing decrements it by one
  (`s.repeats += -1`, `django_q/cluster.py:648`). The magnitude is therefore a running **count of past
  firings** during this container's uptime — e.g. `id=4` has fired ≈13 times. This alone proves the
  scheduler has been active.
- **`next_run` is the next due instant.** Once `timezone.now()` passes it, the schedule becomes eligible
  on the next scheduler pass.

### 10.2 The scheduler mechanism, and a controlled live firing (observed)

**Mechanism (cause → effect).** The scheduler is not a separate OS process. It runs inside the cluster
**Sentinel** (`Process-1`): the Sentinel's `guard()` loop (`django_q/cluster.py:253`) increments a counter
each ~0.5 s cycle and, when `counter >= 30 and Conf.SCHEDULER` (`:284`), calls `scheduler(broker=self.broker)`
(`:286`) — i.e. roughly every 30 s. `scheduler()` (`:576`) selects due rows (source shown verbatim):

```console
$ docker exec pngx_qa sed -n '586,589p' /usr/local/lib/python3.9/site-packages/django_q/cluster.py
            for s in (
                Schedule.objects.select_for_update()
                .exclude(repeats=0)
                .filter(next_run__lt=timezone.now())
```

The filter is a **strict `<`** (`next_run__lt`, `:589`): a schedule fires only once its `next_run` is
strictly in the past. For each due row, the interval is applied to the **old** `next_run` and the row is
advanced to the next future slot before the task is enqueued (source verbatim):

```console
$ docker exec pngx_qa sed -n '612,640p' /usr/local/lib/python3.9/site-packages/django_q/cluster.py
                if s.schedule_type != s.ONCE:
                    next_run = arrow.get(s.next_run)
                    while True:
                        if s.schedule_type == s.MINUTES:
                            next_run = next_run.shift(minutes=+(s.minutes or 1))
                        elif s.schedule_type == s.HOURLY:
                            next_run = next_run.shift(hours=+1)
                        elif s.schedule_type == s.DAILY:
                            next_run = next_run.shift(days=+1)
                        elif s.schedule_type == s.WEEKLY:
                            next_run = next_run.shift(weeks=+1)
                        elif s.schedule_type == s.MONTHLY:
                            next_run = next_run.shift(months=+1)
                        elif s.schedule_type == s.QUARTERLY:
                            next_run = next_run.shift(months=+3)
                        elif s.schedule_type == s.YEARLY:
                            next_run = next_run.shift(years=+1)
                        elif s.schedule_type == s.CRON:
                            if not croniter:
                                raise ImportError(
                                    _(
                                        "Please install croniter to enable cron expressions"
                                    )
                                )
                            next_run = arrow.get(
                                croniter(s.cron, localtime()).get_next()
                            )
                        if Conf.CATCH_UP or next_run > arrow.utcnow():
                            break
```

Because paperless sets `catch_up=False` (Q2, [§4](#4-q2--services-involved)), the `while` loop keeps
shifting by the interval until `next_run > arrow.utcnow()` (`:639`), so a past-due schedule is advanced to
the **next future slot** and fires **once**, skipping any missed slots. It then decrements
`s.repeats += -1` (`:648`), enqueues via `s.task = django_q.tasks.async_task(s.func, *args, **kwargs)`
(`:658`), and logs `"{process name} created a task from schedule [{name}]"` (`:669`).

To observe this end-to-end without waiting an hour, I nudged the HOURLY *Train the classifier* schedule's
`next_run` into the past and watched the live Sentinel (cluster pid `5928`) react on its next pass.

**BEFORE** — `id=1` is due in the future; I set its `next_run` to a clean past instant (`23:09:00`):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.utils import timezone
from datetime import datetime, timezone as tz
from django_q.models import Schedule
s = Schedule.objects.get(id=1)
print("nudge applied at UTC now =", timezone.now().isoformat())
print("  old next_run =", s.next_run.isoformat(), "| old repeats =", s.repeats)
s.next_run = datetime(2026, 7, 14, 23, 9, 0, tzinfo=tz.utc)
s.save(update_fields=["next_run"])
s.refresh_from_db()
print("  SET next_run =", s.next_run.isoformat(), "| repeats (unchanged) =", s.repeats)
PY
nudge applied at UTC now = 2026-07-14T23:11:50.483111+00:00
  old next_run = 2026-07-14T23:13:15.405279+00:00 | old repeats = -3
  SET next_run = 2026-07-14T23:09:00+00:00 | repeats (unchanged) = -3
```

**DURING** — within one scheduler pass (~30 s later, at `23:12:20`) the live Sentinel logged the firing.
`Process-1` is the Sentinel process, confirming the scheduler runs there (not in a worker, not in a
separate OS process):

```console
$ docker exec pngx_qa grep -nE 'created a task from schedule \[Train the classifier\]' /tmp/obs/qcluster.log
51:23:12:20 [Q] INFO Process-1 created a task from schedule [Train the classifier]
```

**AFTER** — the row advanced by exactly the HOURLY interval, `repeats` decremented once, and a real
`django_q_task` row was created whose id equals the id Django-Q stored back on the schedule as `s.task`:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.utils import timezone
from django_q.models import Schedule, Task
s = Schedule.objects.get(id=1)
print("observed at UTC now =", timezone.now().isoformat())
print("  id=1 next_run =", s.next_run.isoformat())
print("  id=1 repeats  =", s.repeats)
print("  id=1 s.task   =", s.task)
t = Task.objects.filter(func="documents.tasks.train_classifier").order_by("-started").first()
print("  Task.id      =", t.id)
print("  Task.name    =", t.name)
print("  Task.started =", t.started.isoformat())
print("  Task.stopped =", t.stopped.isoformat())
print("  Task.success =", t.success)
print("  Task.result  =", repr(t.result))
PY
observed at UTC now = 2026-07-14T23:12:38.959661+00:00
  id=1 next_run = 2026-07-15T00:09:00+00:00
  id=1 repeats  = -4
  id=1 s.task   = 632d85191f7f401b969fbd4d4a34a06f
  Task.id      = 632d85191f7f401b969fbd4d4a34a06f
  Task.name    = spring-eight-wolfram-pluto
  Task.started = 2026-07-14T23:12:20.323091+00:00
  Task.stopped = 2026-07-14T23:12:21.073674+00:00
  Task.success = True
  Task.result  = None
```

Reading the AFTER state against the mechanism:

- **`next_run`: `23:09:00` → `2026-07-15T00:09:00`** — exactly `+1 h`, the HOURLY shift applied to the
  *old* `next_run` I set (not to "now"); one shift sufficed because `23:09:00 + 1 h = 00:09:00` is already
  in the future, so the `catch_up=False` loop stopped after one iteration.
- **`repeats`: `-3` → `-4`** — decremented exactly once, regardless of how far in the past the trigger was.
- **`s.task == Task.id == 632d85191f7f401b969fbd4d4a34a06f`** — the schedule stores the id of the task it
  enqueued, and that task ran to completion (`success=True`; `result=None` because `train_classifier`
  returns `None` when no retrain is warranted). The 32-char hex id is a Django-Q task id exactly as in
  [§5](#5-q3--how-a-job-appears-when-created).

This is the same enqueue → execute → persist lifecycle as an ingested document; the *only* difference is
the caller — the Sentinel's `scheduler()` rather than an ingestion entry point.

### 10.3 Cadence stability across ≥2 intervals (observed)

The MINUTES=10 *Check all e-mail accounts* schedule fires often enough to measure directly. This cluster
instance (started `22:50`) logged **three** consecutive firings — two full intervals — all attributed to
`Process-1`:

```console
$ docker exec pngx_qa grep -nE 'created a task from schedule \[Check all e-mail accounts\]' /tmp/obs/qcluster.log
32:22:53:17 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
44:23:03:19 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
58:23:13:20 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

The persisted `Task.started` timestamps give the exact inter-firing deltas across the container's whole
uptime (a longer history than one cluster instance, because the rows survive `qcluster` restarts):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
prev = None
for t in Task.objects.filter(func="paperless_mail.tasks.process_mail_accounts").order_by("started"):
    d = (t.started - prev).total_seconds() if prev else None
    print(f"{t.started.isoformat()}  delta={d if d is None else round(d, 3)}s")
    prev = t.started
PY
2026-07-14T21:15:05.672139+00:00  delta=Nones
2026-07-14T21:23:20.144112+00:00  delta=494.472s
2026-07-14T21:33:32.490845+00:00  delta=612.347s
2026-07-14T21:43:19.564116+00:00  delta=587.073s
2026-07-14T21:53:21.038685+00:00  delta=601.475s
2026-07-14T22:03:21.299874+00:00  delta=600.261s
2026-07-14T22:13:22.742022+00:00  delta=601.442s
2026-07-14T22:23:24.195348+00:00  delta=601.453s
2026-07-14T22:33:25.605871+00:00  delta=601.411s
2026-07-14T22:43:27.784459+00:00  delta=602.179s
2026-07-14T22:53:17.520422+00:00  delta=589.736s
2026-07-14T23:03:19.023166+00:00  delta=601.503s
2026-07-14T23:13:20.503028+00:00  delta=601.48s
```

**Reading the cadence.** The steady-state deltas cluster tightly at **600.3–602.2 s** — the 10-minute
interval plus up to one guard cycle of latency. The schedule's `next_run` advances by *exactly* 600 s
(`.shift(minutes=+10)`, `cluster.py:616`), but the firing happens on the first ~30 s guard pass *after*
`next_run` elapses, so successive firings inherit a near-constant phase offset of ≈1.5 s. The three shorter
deltas (494.472 s, 587.073 s, 589.736 s) each coincide with a `qcluster` restart during the investigation
(a restart shifts the guard phase); they are restart artifacts, not cadence drift. Across ≥2 consecutive
steady-state intervals the cadence is stable.

The deterministic half of this is directly visible on the schedule row: after the `23:13:20` firing,
`id=4`'s `next_run` had advanced by exactly `+10 min`:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Schedule
s = Schedule.objects.get(id=4)
print("id=4 next_run =", s.next_run.isoformat(), "| repeats =", s.repeats, "| last task =", s.task)
PY
id=4 next_run = 2026-07-14T23:23:16.761616+00:00 | repeats = -14 | last task = 472bee71d94248deb0db57776a9689f4
```

`23:13:16.761616 + 10 min = 23:23:16.761616` — the interval is applied to the stored `next_run`, so cadence
does not drift even though the *observed firing* lags by the guard latency.

### 10.4 Scheduled jobs are enqueued identically to ingestion jobs

There is no separate "scheduled-task" code path at enqueue time. As established in
[§5.3](#53-scheduled-jobs-appear-the-same-way), `scheduler()` calls the same
`django_q.tasks.async_task(s.func, …)` (`django_q/cluster.py:658`) that the ingestion entry points call
([§9](#9-q7--enqueue-origin-in-code)); it produces the same signed package on the same
`django_q:paperless:q` Redis list, is dequeued by the same workers, and persists the same kind of
`django_q_task` row. The scheduler is therefore best understood as a **fifth enqueue origin** whose caller
is the Sentinel rather than a user-facing ingestion path.

---


## 11. Exhaustive condition coverage

Rule 2 requires that **every** condition each of the seven questions implies be exercised — not just the
happy path — and that state changes be observed *before / during / after*. This section is the coverage
ledger: each row names a condition, the **canonical** entry point that exercised it, the section holding
the adjacent evidence, and whether the row is **observed** at runtime or **(inferred)** / **(documentation-derived)**
per the label conventions in [How to read this document](#how-to-read-this-document-methodology--label-conventions).
[§11.2](#112-conditions-described-from-source-not-run-under-canonical-defaults) then lists — explicitly —
the few conditions that the *default canonical configuration* does not run, so no claim of "everything was
executed" is overstated.

### 11.1 Condition matrix

| Q | Condition class | Condition exercised | Canonical trigger | Evidence | Basis |
|---|---|---|---|---|---|
| Q1 | primary | Full ingestion lifecycle: watcher → `consume_file` → persisted row | Directory drop into the consume dir | [§3](#3-q1--live-async-behavior) | observed |
| Q1 | transitional | The five hops enqueue → queue → dequeue → execute → persist | Directory drop | [§3](#3-q1--live-async-behavior), [§6](#6-q4--waiting-vs-actively-processing) | observed |
| Q1 | secondary | Six post-consumption signal handlers run (inbox tags, correspondent/type/tag matching, `Log` entry, search-index add) | Directory drop, one correlated document | [§3](#3-q1--live-async-behavior) | observed |
| Q1 | edge | Thumbnail path: ImageMagick blocked by PDF policy → Ghostscript fallback (non-fatal warning) | Text-layer PDF through the watcher | [§3](#3-q1--live-async-behavior) | observed |
| Q1 | alternate-flag | Barcode-separated split inside `consume_file` (`tasks.py:196-234`) | Not run — gated off by default `PAPERLESS_CONSUMER_ENABLE_BARCODES=false` | [§3](#3-q1--live-async-behavior), [§11.2](#112-conditions-described-from-source-not-run-under-canonical-defaults) | (inferred) |
| Q2 | primary | Three Supervisord programs (`gunicorn`, `consumer`, `scheduler`) plus Redis in its dual broker+Channels role | Launch per `docker/supervisord.conf` | [§4](#4-q2--services-involved) | observed |
| Q2 | topology | Cluster process tree: 1 Sentinel + N workers + 1 monitor + 1 pusher, all **processes** (not threads) | `qcluster` running | [§4](#4-q2--services-involved) | observed |
| Q2 | transitional | Graceful cluster stop: SIGTERM → workers drain → monitor/pusher exit | Live cluster stop at teardown | [§4](#4-q2--services-involved) | observed |
| Q3 | primary | A signed task package on the Redis list carrying id / func / args / name | Watcher drop with the cluster **down** | [§5](#5-q3--how-a-job-appears-when-created) | observed |
| Q3 | edge | The payload is opaque signed-pickle bytes; HMAC signature gate is the trust boundary | Same waiting package, peeked with `LINDEX` | [§5.2](#52-why-the-payload-is-opaque-bytes--the-signed-pickle-trust-boundary) | observed + (inferred) |
| Q3 | secondary | A scheduled job produces the *same* kind of package | Live schedule firing | [§5.3](#53-scheduled-jobs-appear-the-same-way), [§10](#10-scheduled--recurring-jobs) | observed |
| Q4 | transitional | WAITING (before) → IN-CLUSTER/ACTIVE (during) → DONE (after), keyed to one `Task.id` | `poll.py` sampling one task | [§6](#6-q4--waiting-vs-actively-processing) | observed |
| Q4 | edge | `LLEN=0` + no row is ambiguous — internally queued vs. actively running — disambiguated by same-task worker log timestamps | `poll.py` + worker log | [§6](#6-q4--waiting-vs-actively-processing) | observed + (inferred) |
| Q4 | stability | Two identical runs; WAITING/ACTIVE/DONE pattern and durations reported with variance | Repeat of the Q4 experiment | [§6](#6-q4--waiting-vs-actively-processing) | observed |
| Q5 | primary | State persists in table `django_q_task` via `django_q.models.Task` | Inspect DB after a real task | [§7](#7-q5--where-task-state-is-stored) | observed |
| Q5 | negative | **No** custom paperless task model exists (only Django-Q's tables) | Enumerate `documents` models + migrations | [§7](#7-q5--where-task-state-is-stored) | observed |
| Q6 | primary | Success row: `func`/`args`/`kwargs`/`started`/`stopped`/`time_taken`/`success`/`result` all present | A successfully consumed document | [§8](#8-q6--after-the-fact-status) | observed |
| Q6 | error | Failure row: `success=False` with the **complete, untruncated** traceback in `result` | A forced duplicate-detection failure | [§8](#8-q6--after-the-fact-status) | observed |
| Q6 | secondary | `Success` / `Failure` proxy models back the two admin pages over the same table | DB + admin | [§8](#8-q6--after-the-fact-status) | observed + (documentation-derived) |
| Q6 | transitional | Live WebSocket `status_updates` frames STARTING → WORKING → SUCCESS | Authenticated WS during a live ingest | [§8](#8-q6--after-the-fact-status) | observed |
| Q6 | edge | The "Queued tasks" (`OrmQ`) admin page is **absent** under the Redis broker | Admin under the canonical Redis config | [§8](#8-q6--after-the-fact-status) | (documentation-derived) |
| Q7 | primary | Four live enqueue origins: directory watcher, REST upload, real IMAP mail fetch, bulk edit | Each canonical entry point | [§9](#9-q7--enqueue-origin-in-code) | observed |
| Q7 | contrast | Bulk **delete** runs **synchronously** (no `async_task`) unlike the five bulk async sites | Bulk-edit module inspection | [§9](#9-q7--enqueue-origin-in-code) | observed |
| Q7 | API-contract | REST `POST /api/documents/post_document/` returns `Response("OK")`; the progress UUID is separate | REST upload | [§9](#9-q7--enqueue-origin-in-code) | observed |
| Q7 | alternate-flag | Barcode split origin (default-disabled) | Not run — see [§11.2](#112-conditions-described-from-source-not-run-under-canonical-defaults) | [§9](#9-q7--enqueue-origin-in-code) | (inferred) |
| §10 | primary | Four seeded `django_q_schedule` rows (train classifier, optimize index, sanity check, check e-mail) | `migrate` then inspect | [§10.1](#101-the-four-seeded-schedules-observed) | observed |
| §10 | transitional | Live firing before/during/after: due row → `async_task` → `next_run` advanced, `repeats` decremented | Nudged HOURLY schedule fired by the Sentinel | [§10.2](#102-the-scheduler-mechanism-and-a-controlled-live-firing-observed) | observed |
| §10 | stability | Cadence measured across ≥2 consecutive intervals of a MINUTES schedule | Natural firings of the mail schedule | [§10.3](#103-cadence-stability-across-2-intervals-observed) | observed |
| §10 | alternate-flag | `catch_up=False` → a schedule missed while the cluster was down runs **once** on restart, not once per missed slot | Observed `next_run` advancing to the next future slot | [§10.2](#102-the-scheduler-mechanism-and-a-controlled-live-firing-observed) | observed + (documentation-derived) |
| all | robustness | Worker `recycle=1` — a fresh worker process per task | `Q_CLUSTER` read back at runtime | [§4](#4-q2--services-involved) | observed |
| all | edge | Under the Redis broker (no message receipts) an in-flight task lost to a process crash or timeout is **not** re-delivered; `retry=1810`s is a safety bound | Config + broker behavior | [§4](#4-q2--services-involved), [§6](#6-q4--waiting-vs-actively-processing) | (inferred) + (documentation-derived) |

### 11.2 Conditions described from source (not run under canonical defaults)

Three conditions are **not** executed by the default canonical build; each is described from source and
labeled, and none is claimed as observed:

- **Barcode-separated splitting inside `consume_file`** (`documents/tasks.py:196-234`). The default
  canonical configuration ships `PAPERLESS_CONSUMER_ENABLE_BARCODES=false`, so this branch does not run
  during ingestion. It is described from source and labeled **(inferred)** in
  [§3](#3-q1--live-async-behavior) / [§9](#9-q7--enqueue-origin-in-code). Running it would require
  flipping a non-default flag, which the run-first canonical mandate excludes.
- **Loss of an in-flight task on process death under the Redis broker.** Deliberately crashing a worker
  mid-task is not a safe, repeatable canonical action; the "lost, not retried" behavior is therefore
  **(inferred)** from `django_q/cluster.py` and corroborated **(documentation-derived)** by the
  Django-Q *Brokers* section (the Redis broker has no message receipts). See [§6](#6-q4--waiting-vs-actively-processing).
- **The ORM-broker "Queued tasks" (`OrmQ`) admin page.** This page exists **only** with the Django ORM
  broker; the canonical deployment uses the Redis broker, so the page is legitimately absent. Its
  existence-condition is stated **(documentation-derived)** in [§8](#8-q6--after-the-fact-status), and the
  canonical Redis behavior (no such page) is what is observed.

Everything else the seven questions imply — every ingestion origin, both success and failure outcomes,
the full waiting/processing/done transition (twice), the persisted table and its absence of a custom
model, the live realtime stream, and the recurring-schedule lifecycle and cadence — was exercised **live**
through canonical entry points, with adjacent complete evidence in the sections cited above.

---

## 12. Cleanup & repository integrity

The whole investigation ran inside **two disposable containers** (`pngx_qa` for the app, `pngx_redis`
for the broker) and used only temporary scripts under the container's `/tmp/pp` and host evidence files
under `/tmp/qa_work` — **no file in the source repository was ever modified** (the read-only mandate).
Cleanup is therefore a *non-destructive* teardown: gracefully stop the services, then delete the
disposable containers. There is **no** database or file "restore" step, because nothing durable outside
the containers was touched — a deliberate contrast to a destructive "overwrite the DB then delete the
backup" approach.

### 12.1 Safe, validated service shutdown

Every service PID was captured **live** at launch via `$!` (recorded to `/tmp/obs/*.pid` by
[`launch.sh`](#132-the-remaining-helper-scripts-verbatim)); none is hard-coded. Immediately **before**
signalling, each PID's `/proc/<pid>/comm` is re-validated so a recycled PID can never be signalled by
mistake (this closes the check-then-act TOCTOU window). `SIGTERM` is sent to the `qcluster` **main**
process, which triggers Django-Q's graceful stop so the Sentinel drains and reaps its entire child tree:

```console
$ docker exec pngx_qa sh -c '<validate comm, then SIGTERM 5928 191 192 5211, then poll /proc>'
===== STEP 1: validate comm immediately before signalling (finding #16) =====
  pid=5928  comm=python3 -> will SIGTERM
  pid=191   comm=python3 -> will SIGTERM
  pid=192   comm=gunicorn -> will SIGTERM
  pid=5211  comm=python3 -> will SIGTERM

===== STEP 2: SIGTERM top-level services (qcluster main drains its own tree) =====
  SIGTERM -> 5928 (python3)
  SIGTERM -> 191 (python3)
  SIGTERM -> 192 (gunicorn)
  SIGTERM -> 5211 (python3)

===== STEP 3: wait for graceful drain + child reaping =====
  t=1s  service PIDs still alive: 18
  t=2s  service PIDs still alive: 11
  t=3s  service PIDs still alive: 4
  t=4s  service PIDs still alive: 4
  ...
  t=12s service PIDs still alive: 4
```

Within ~3 s the **14** cluster children — the Sentinel (`5945`) plus its 11 workers, 1 monitor, and 1
pusher — exit and are reaped. The **4** top-level processes that were launched with `nohup … &  disown`
finish their own shutdown but, having been reparented to PID 1, are left as **defunct (zombie)** entries
because this minimal container init does not reap them. A zombie holds **no** memory, file descriptors,
or sockets — it is only a slot in the process table — and is reaped the instant the container is removed:

```console
$ docker exec pngx_qa sh -c 'for p in 5928 191 192 5211; do awk "{print \$3}" /proc/$p/stat; done'
  pid=5928  state=Z ppid=1 comm=python3
  pid=191   state=Z ppid=1 comm=python3
  pid=192   state=Z ppid=1 comm=gunicorn
  pid=5211  state=Z ppid=1 comm=python3
```

That the services are **functionally** dead (not merely defunct shells) is confirmed by the absence of
any listening socket inside the container: parsing `/proc/net/tcp` for `LISTEN` (state `0A`) returned
**no rows** for the gunicorn port `8000` or the IMAP port `10143` — the sockets were released when the
processes terminated. (The scan of `manage.py qcluster` processes must exclude the scanning shell itself,
whose own command line contains that string — otherwise it self-matches; the per-PID `/proc` existence
check used above avoids that trap entirely.)

### 12.2 Disposable-container discard (reaps the zombies, frees everything)

Because the entire runtime — app, worker cluster, watcher, IMAP test server, Redis broker, and the
SQLite DB / media / consume directories under the container's `/tmp` — lived inside the two disposable
containers, the correct and complete cleanup is to **delete the containers**. This atomically reaps the
four zombies and frees every resource; no host or repository state needs restoring:

```console
$ docker rm -f pngx_qa pngx_redis
pngx_qa
pngx_redis

$ docker ps -a --format '{{.Names}}' | grep -E '^pngx_(qa|redis)$' || echo "neither pngx_qa nor pngx_redis exists"
neither pngx_qa nor pngx_redis exists

$ ps -eo pid,comm,args | grep -E 'imap_server\.py|manage\.py qcluster|paperless\.asgi' | grep -v grep || echo "no host service processes"
no host service processes

$ ss -ltnp 2>/dev/null | grep -E ':10143|:8000' || echo "no host listeners on 10143/8000"
no host listeners on 10143/8000
```

No container, no process, and no port remains. Note there was never a **published** host port to leak in
the first place: `pngx_qa` was created with **no** `-p` mapping and driven purely by `docker exec`, so
gunicorn's `0.0.0.0:8000` and the IMAP `10143` were reachable only from inside the container — never on
the host (see [§2.1](#21-disclosed-environment-posture-deviations-from-production-and-why)).

### 12.3 The repository is byte-for-byte unchanged except the single document

Finally, the sole-file proof. This block was captured **during authoring**, while the answer document was
still an uncommitted working-tree modification; at that point the working tree differed from `HEAD` in
**exactly one** path — the answer document — and the `blitzy/screenshots/` directory that the earlier
review flagged is gone:

```console
$ git rev-parse --abbrev-ref HEAD && git rev-parse HEAD
blitzy-a0535bb1-52e0-488f-9eaf-09f99118e2cf
d68342db3699ba4fdd25ca341e7cf33c2067ccd2

$ git status --porcelain
 M blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff --name-only
blitzy/documentation/paperless-ngx_542221a38dff.md

$ test -e blitzy/screenshots && echo "screenshots STILL present" || echo "blitzy/screenshots does not exist"
blitzy/screenshots does not exist

$ find blitzy -type f
blitzy/documentation/paperless-ngx_542221a38dff.md

$ git status --porcelain | grep '^??' || echo "no untracked files"
no untracked files
```

> **Authoring-time snapshot.** The `HEAD` shown above (`d68342db3…`) is the mid-authoring commit at the
> moment of capture; it advances by one commit each time this document is revised, so a later reader will
> observe a different `HEAD`. Likewise, `git status --porcelain` shows the document as a pending ` M` only
> while it is uncommitted — once the document is committed it becomes tracked and `git status --porcelain`
> is **empty**. The durable, always-reproducible invariant is therefore not the exact `HEAD` hash but the
> *delta from the base commit*: `git diff --name-status 542221a38dff..HEAD` yields exactly one line —
> `A blitzy/documentation/paperless-ngx_542221a38dff.md` — with **no source file modified**.

The only change this task introduces to the repository is the addition/modification of this one Markdown
document; every temporary artifact used to produce it lived outside the tracked tree (in the now-deleted
containers and in host scratch under `/tmp`) and leaves **no trace in the repository** — the sole,
load-bearing integrity guarantee. (Host authoring scratch under `/tmp` is not part of the repository or
this deliverable and is unrelated to the tracked-tree cleanliness proven above.)

---

<a id="appendix"></a>

## 13. Appendix

Every temporary artifact used to observe the behaviors in this document is a **script**, published here
inline and **verbatim** so that each command shown in §1–§10 is reproducible byte-for-byte. Nothing in this
appendix modifies the repository: these scripts lived **outside** the tracked tree — under the container's
`/tmp/pp/`, which is discarded when the disposable containers are removed (see
[§12](#12-cleanup--repository-integrity)), and in a host authoring-scratch directory (`/tmp/qa_work/`) that
is not part of the repository or this deliverable. The single load-bearing integrity guarantee is that the
tracked tree is left byte-for-byte unchanged except for this one document. No credential ever appears in the
repository or in this published document: the admin/test password and the mailbox password are shown as
`<REDACTED>` throughout. Most helpers read the real value from an environment variable at run time — e.g.
`QA_ADMIN_PW`, `PP_PASS` (see `ws_capture.py`, `rest_upload.py`, and `bulk_edit.py` in
[§13.2](#132-the-remaining-helper-scripts-verbatim)); the throwaway mail helper `mail_run.py` instead used a
local, disposable test password inline (shown redacted in §13.2). In every case the value is a local-only,
disposable test credential created solely for this investigation — it is embedded in no repository file and
appears nowhere in this deliverable.

### 13.1 Complete helper-script index

Nine helper scripts were used. Four are already published verbatim earlier in the document (at the point
where they were first needed); the remaining five are published in [§13.2](#132-the-remaining-helper-scripts-verbatim)
directly below. This table is the single consolidated index:

| Script | Lines | Published verbatim in | Purpose (section it serves) |
|---|---:|---|---|
| `penv` | 10 | [§2.3](#23-directories-environment-file-and-the-two-disclosed-helper-scripts) | The exact environment variables sourced by **every** command (redis URL, data/media/consume dirs, `DJANGO_SETTINGS_MODULE`, `PYTHONPATH`) |
| `pp.sh` | 4 | [§2.3](#23-directories-environment-file-and-the-two-disclosed-helper-scripts) | Runs a Python snippet (from stdin) inside the canonical env as non-root `testuser` — the idiom behind every ORM/broker probe |
| `launch.sh` | 8 | [§2.3](#23-directories-environment-file-and-the-two-disclosed-helper-scripts) | Starts the three canonical services (`qcluster`, `document_consumer`, `gunicorn`) per `docker/supervisord.conf`, recording each master PID via `$!` |
| `poll.py` | 42 | [§6](#6-q4--waiting-vs-actively-processing) | The Q4 boundary poller — samples Redis `LLEN`, cluster `Stat`, and the specific `Task` row to prove WAITING → PROCESSING → DONE |
| `ws_capture.py` | 56 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q6 authenticated WebSocket status capture (real session login → `sessionid` cookie → `ws/status/`) |
| `imap_server.py` | 180 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 real minimal IMAP4 server (Twisted) — one plaintext account, one RFC822 message with a `.txt` attachment |
| `rest_upload.py` | 56 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 REST upload driver — browser-canonical session + CSRF auth against `POST /api/documents/post_document/` |
| `mail_run.py` | 70 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 mail driver — creates a `MailAccount`/`MailRule` and invokes the single-account convenience wrapper `process_mail_account` (same `handle_mail_account`→`handle_message` chain as the seeded plural `process_mail_accounts`) to fetch |
| `bulk_edit.py` | 47 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 bulk-edit driver — session + CSRF auth against `POST /api/documents/bulk_edit/` (JSON body) |

Total: **473 lines** across nine scripts. `penv`, `pp.sh`, and `launch.sh` appear in
[§2.3](#23-directories-environment-file-and-the-two-disclosed-helper-scripts); `poll.py` appears in
[§6](#6-q4--waiting-vs-actively-processing); the five below complete the set.

### 13.2 The remaining helper scripts (verbatim)

The five scripts referenced by forward-links elsewhere in the document are reproduced here exactly as run
(only the credential literals are replaced by `<REDACTED>`; line counts are otherwise unchanged from the
files under `/tmp/pp/`).

#### `ws_capture.py` — Q6 authenticated WebSocket status capture ([§8](#8-q6--after-the-fact-status))

```python
#!/usr/bin/env python3
"""Disclosed WebSocket status capture for Q6.
Flow: (1) real Django session login via /admin/login/ (CSRF) -> sessionid cookie;
      (2) open ws://localhost:8000/ws/status/ carrying that cookie (AuthMiddlewareStack
          reads it -> scope['user'] -> StatusConsumer.connect accepts);
      (3) drop a .txt through the watched consume dir to trigger consume_file;
      (4) print every status_updates frame as it arrives, with an arrival timestamp.
Credentials come from env PP_USER / PP_PASS; the sessionid value is redacted in output.
"""
import os, sys, json, asyncio, subprocess
from datetime import datetime, timezone
import requests, websockets

BASE = "http://localhost:8000"
USER = os.environ["PP_USER"]
PASS = os.environ["PP_PASS"]

def iso():
    return datetime.now(timezone.utc).isoformat(timespec="milliseconds")

# (1) Real session login (CSRF-protected Django admin login form)
s = requests.Session()
g = s.get(BASE + "/admin/login/")
csrf = s.cookies.get("csrftoken")
print("LOGIN GET /admin/login/  -> HTTP %d, csrftoken cookie present=%s" % (g.status_code, bool(csrf)))
p = s.post(BASE + "/admin/login/",
           data={"username": USER, "password": PASS,
                 "csrfmiddlewaretoken": csrf, "next": "/admin/"},
           headers={"Referer": BASE + "/admin/login/"}, allow_redirects=False)
sid = s.cookies.get("sessionid")
print("LOGIN POST /admin/login/ -> HTTP %d (302=success), sessionid cookie present=%s" % (p.status_code, bool(sid)))
print("  auth cookie sent to WS handshake:  Cookie: sessionid=<REDACTED>  (real 32-char value redacted)")

async def go():
    ck = "sessionid=%s; csrftoken=%s" % (sid, csrf)
    async with websockets.connect(BASE.replace("http", "ws") + "/ws/status/",
                                  extra_headers={"Cookie": ck}) as ws:
        print("WS %s CONNECTED (HTTP 101 Switching Protocols) — authenticated accept" % iso())
        ts = datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
        fn = "/tmp/pp/consume/qa_ws_%s.txt" % ts
        subprocess.run(["bash", "-lc", "echo 'PAPERLESS_QA_WS_%s live status stream demo' > %s" % (ts, fn)], check=True)
        print("WS %s dropped %s to trigger ingestion" % (iso(), fn))
        n = 0
        while True:
            try:
                raw = await asyncio.wait_for(ws.recv(), timeout=20)
            except asyncio.TimeoutError:
                print("WS %s (no more frames for 20s; stopping)" % iso())
                break
            n += 1
            frame = json.loads(raw)
            print("WS FRAME %d %s  %s" % (n, iso(), json.dumps(frame, sort_keys=True)))
            if frame.get("status") in ("SUCCESS", "FAILED"):
                print("WS %s terminal status '%s' received; stopping" % (iso(), frame.get("status")))
                break
asyncio.get_event_loop().run_until_complete(go())
```

#### `imap_server.py` — Q7 real minimal IMAP4 server, Twisted ([§9](#9-q7--enqueue-origin-in-code))

```python
#!/usr/bin/env python3
"""A REAL, minimal IMAP4 server (Twisted) for exercising paperless-ngx mail ingestion.
It serves ONE plaintext account with an INBOX holding ONE real RFC822 message that
carries a .txt attachment. No mocks: paperless connects to it with its real imap_tools
client over a real TCP socket. Supports LOGIN, SELECT, SEARCH, FETCH (BODY.PEEK[]),
STORE (\\Seen for the MARK_READ post-consume action), LOGOUT.
Usage: imap_server.py <port> <user> <password>
"""
import sys, io, email
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.utils import formatdate
from zope.interface import implementer
from twisted.internet import reactor, protocol
from twisted.mail import imap4
from twisted.cred import portal, checkers
from twisted.python import log

PORT = int(sys.argv[1]); USER = sys.argv[2].encode(); PASS = sys.argv[3].encode()

def build_message():
    m = MIMEMultipart()
    m["Subject"] = "PaperlessQA live mail ingestion test"
    m["From"] = "qa-sender@example.test"
    m["To"] = "paperless@example.test"
    m["Date"] = formatdate(localtime=False)
    m.attach(MIMEText("Body: this message carries one text attachment for paperless.", "plain"))
    att = MIMEText("PAPERLESS_QA_MAIL_ATTACHMENT unique content for the mail ingestion path.", "plain")
    att.add_header("Content-Disposition", "attachment", filename="qa_mail_attachment.txt")
    m.attach(att)
    return m.as_bytes().replace(b"\n", b"\r\n")

RFC822 = build_message()

@implementer(imap4.IMessagePart)
class Part:
    def __init__(self, mimepart):
        self.mp = mimepart
    def getHeaders(self, negate, *names):
        names = [n.upper() for n in names]
        out = {}
        for k, v in self.mp.items():
            if (negate ^ (k.upper() in names)) or not names:
                out[k.lower()] = v
        return out
    def getBodyFile(self):
        payload = self.mp.get_payload(decode=False)
        if isinstance(payload, list):
            payload = self.mp.as_string()
        return io.BytesIO(payload.encode() if isinstance(payload, str) else payload)
    def getSize(self):
        return len(self.mp.as_bytes())
    def isMultipart(self):
        return self.mp.is_multipart()
    def getSubPart(self, part):
        return Part(self.mp.get_payload()[part])

@implementer(imap4.IMessage)
class Message:
    def __init__(self, uid, flags, rfc822):
        self.uid = uid
        self.flags = list(flags)
        self._raw = rfc822
        self.msg = email.message_from_bytes(rfc822)
    def getUID(self): return self.uid
    def getFlags(self): return self.flags
    def getInternalDate(self): return b"01-Jan-2026 00:00:00 +0000"
    def getHeaders(self, negate, *names):
        names = [n.upper() for n in names]
        out = {}
        for k, v in self.msg.items():
            if not names or (negate ^ (k.upper() in names)):
                out[k.lower()] = v
        return out
    def getBodyFile(self):
        body = self._raw.split(b"\r\n\r\n", 1)[1]
        return io.BytesIO(body)
    def getSize(self): return len(self._raw)
    def isMultipart(self): return self.msg.is_multipart()
    def getSubPart(self, part): return Part(self.msg.get_payload()[part])

@implementer(imap4.IMailbox)
class Mailbox:
    def __init__(self):
        self.messages = [Message(1, [], RFC822)]
        self.listeners = []
    def getUIDValidity(self): return 1
    def getUIDNext(self): return len(self.messages) + 1
    def getUID(self, num): return num
    def getMessageCount(self): return len(self.messages)
    def getRecentCount(self): return 0
    def getUnseenCount(self): return sum(1 for m in self.messages if r'\Seen' not in m.flags)
    def isWriteable(self): return True
    def getHierarchicalDelimiter(self): return "."
    def getFlags(self): return [r'\Seen', r'\Answered', r'\Flagged', r'\Deleted', r'\Draft', r'\Recent']
    def getMessage(self, num): return self.messages[num - 1]
    def requestStatus(self, names): return imap4.statusRequestHelper(self, names)
    def addListener(self, l): self.listeners.append(l)
    def removeListener(self, l): self.listeners.remove(l)
    def _resolve_last(self, messages, uid):
        # A "1:*" range parses to a MessageSet with an open-ended high (None) and
        # its `.last` UNSET (the getter returns a sentinel, not None). Twisted then
        # raises "last value not set" on any membership/iteration test. Detect the
        # open range via the public `ranges` list and resolve '*' to the highest id.
        needs = any(hi is None for (_lo, hi) in getattr(messages, 'ranges', []))
        if needs:
            messages.last = (max((m.uid for m in self.messages), default=0)
                             if uid else self.getMessageCount())
    def store(self, messages, flags, mode, uid):
        self._resolve_last(messages, uid)
        out = {}
        for i, m in enumerate(self.messages, 1):
            key = m.uid if uid else i
            if key not in messages:
                continue
            if mode == 1:
                for f in flags:
                    if f not in m.flags: m.flags.append(f)
            elif mode == -1:
                for f in flags:
                    if f in m.flags: m.flags.remove(f)
            else:
                m.flags = list(flags)
            out[i] = m.flags
        return out
    def fetch(self, messages, uid):
        self._resolve_last(messages, uid)
        result = []
        for i, m in enumerate(self.messages, 1):
            key = m.uid if uid else i
            if key in messages:
                result.append((i, m))
        return result
    def expunge(self):
        removed = [i + 1 for i, m in enumerate(self.messages) if r'\Deleted' in m.flags]
        self.messages = [m for m in self.messages if r'\Deleted' not in m.flags]
        return removed
    def destroy(self): pass

@implementer(portal.IRealm)
class Realm:
    def __init__(self): self.mailbox = Mailbox()
    def requestAvatar(self, avatarId, mind, *interfaces):
        if imap4.IAccount in interfaces:
            return imap4.IAccount, Account(self.mailbox), lambda: None
        raise NotImplementedError()

@implementer(imap4.IAccount)
class Account:
    def __init__(self, mailbox): self.mailbox = mailbox
    def listMailboxes(self, ref, wildcard): return [("INBOX", self.mailbox)]
    def select(self, name, rw=True): return self.mailbox
    def isSubscribed(self, name): return True
    def create(self, path): return False
    def delete(self, path): return False
    def rename(self, o, n): return False
    def subscribe(self, name): return True
    def unsubscribe(self, name): return True

class IMAPFactory(protocol.Factory):
    def __init__(self, portal_): self.portal = portal_
    def buildProtocol(self, addr):
        p = imap4.IMAP4Server()
        p.portal = self.portal
        p.factory = self
        return p

def main():
    log.startLogging(sys.stderr)
    r = Realm()
    p = portal.Portal(r)
    checker = checkers.InMemoryUsernamePasswordDatabaseDontUse()
    checker.addUser(USER, PASS)
    p.registerChecker(checker)
    reactor.listenTCP(PORT, IMAPFactory(p), interface="127.0.0.1")
    print("IMAP server listening on 127.0.0.1:%d user=%s" % (PORT, USER.decode()), file=sys.stderr)
    reactor.run()

if __name__ == "__main__":
    main()
```

#### `rest_upload.py` — Q7 REST upload driver, session + CSRF ([§9](#9-q7--enqueue-origin-in-code))

```python
import os, sys, time, io, json, requests, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")

BASE = "http://localhost:8000"
USER = "admin"; PW = os.environ.get("QA_ADMIN_PW", "<REDACTED>")

s = requests.Session()

# --- Session + CSRF auth (browser-canonical) ---------------------------------
# 1) GET the login page to obtain the csrftoken cookie
r0 = s.get(f"{BASE}/accounts/login/")
csrf = s.cookies.get("csrftoken")
print("STEP1 GET /accounts/login/ -> %d ; csrftoken cookie present=%s" % (r0.status_code, bool(csrf)))
# 2) POST credentials with the CSRF token (Referer required by Django CSRF)
r1 = s.post(f"{BASE}/accounts/login/",
            data={"username": USER, "password": PW, "csrfmiddlewaretoken": csrf, "next": "/"},
            headers={"Referer": f"{BASE}/accounts/login/"}, allow_redirects=False)
print("STEP2 POST /accounts/login/ -> %d ; sessionid cookie present=%s"
      % (r1.status_code, bool(s.cookies.get("sessionid"))))

# 3) POST the document multipart with the session cookie + X-CSRFToken header
csrf = s.cookies.get("csrftoken")
uniq = time.strftime("%Y%m%d_%H%M%S")
content = ("PAPERLESS_QA_REST_UPLOAD unique content %s for the REST ingestion path." % uniq).encode()
fname = "qa_rest_%s.txt" % uniq
files = {"document": (fname, io.BytesIO(content), "text/plain")}
print("\nREQUEST  POST %s/api/documents/post_document/" % BASE)
print("  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>")
print("  multipart field 'document' = (%s, %d bytes, text/plain)" % (fname, len(content)))
r2 = s.post(f"{BASE}/api/documents/post_document/",
            files=files,
            headers={"X-CSRFToken": csrf, "Referer": f"{BASE}/"})
print("RESPONSE status =", r2.status_code)
print("RESPONSE content-type =", r2.headers.get("Content-Type"))
print("RESPONSE body =", repr(r2.text))

# --- correlate: wait for worker, then show the resulting Task + Document ------
django.setup()
from django_q.models import Task
from documents.models import Document
before_docs = Document.objects.count()
print("\ncorrelating (waiting for worker to consume the REST upload)...")
target=None
for i in range(60):
    t = Task.objects.filter(name__startswith="qa_rest_%s" % uniq).order_by("-started").first()
    if t and t.stopped:
        target=t; break
    time.sleep(1)
if target:
    print("TASK id=%s (hex, Django-Q PK)" % target.id)
    print("  func=%s name=%s success=%s time_taken=%.3fs" % (target.func, target.name, target.success, target.time_taken()))
    print("  result=%r" % target.result)
    d=Document.objects.order_by("-id").first()
    print("DOCUMENT id=%s title=%r content=%r" % (d.id, d.title, d.content))
else:
    print("no task row matched within timeout")
```

#### `mail_run.py` — Q7 mail account/rule setup + fetch driver ([§9](#9-q7--enqueue-origin-in-code))

```python
import os, sys, logging, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()

# Stream the real paperless.mail flow to stdout so the "Consuming attachment ..." line is captured as evidence.
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("MAILLOG %(levelname)s %(name)s: %(message)s"))
lg = logging.getLogger("paperless.mail"); lg.addHandler(h); lg.setLevel(logging.DEBUG)

import redis
from django_q.models import Task
from documents.models import Document
from paperless_mail.models import MailAccount, MailRule
from paperless_mail.tasks import process_mail_account

r = redis.Redis(host="localhost", port=6379)
QKEY = "django_q:paperless:q"

acct, _ = MailAccount.objects.get_or_create(
    name="qa_mail",
    defaults=dict(imap_server="127.0.0.1", imap_port=10143,
                  imap_security=MailAccount.ImapSecurity.NONE,
                  username="qauser", password="<REDACTED>", character_set="US-ASCII"))
# ensure canonical values even if it already existed
acct.imap_server="127.0.0.1"; acct.imap_port=10143
acct.imap_security=MailAccount.ImapSecurity.NONE
acct.username="qauser"; acct.password="<REDACTED>"; acct.character_set="US-ASCII"; acct.save()

rule, _ = MailRule.objects.get_or_create(
    name="qa_rule", account=acct,
    defaults=dict(folder="INBOX", maximum_age=0,
                  action=MailRule.MailAction.MARK_READ,
                  attachment_type=MailRule.AttachmentProcessing.ATTACHMENTS_ONLY,
                  assign_title_from=MailRule.TitleSource.FROM_SUBJECT,
                  assign_correspondent_from=MailRule.CorrespondentSource.FROM_NOTHING))
rule.folder="INBOX"; rule.maximum_age=0; rule.action=MailRule.MailAction.MARK_READ; rule.save()

print("ACCT id=%s security=%s(%s) host=%s:%s user=%s" % (
    acct.id, acct.imap_security, acct.get_imap_security_display(),
    acct.imap_server, acct.imap_port, acct.username))
print("RULE id=%s folder=%s maximum_age=%s action=%s(%s)" % (
    rule.id, rule.folder, rule.maximum_age, rule.action, rule.get_action_display()))

print("BEFORE  LLEN(%s)=%d  tasks=%d  documents=%d"
      % (QKEY, r.llen(QKEY), Task.objects.count(), Document.objects.count()))

print(">>> invoking single-account convenience wrapper: process_mail_account('qa_mail')")
ret = process_mail_account("qa_mail")
print("RETURN process_mail_account =", repr(ret))

print("AFTER   LLEN(%s)=%d  tasks=%d  documents=%d"
      % (QKEY, r.llen(QKEY), Task.objects.count(), Document.objects.count()))

# Peek (non-destructive) the WAITING signed package the mail async_task produced.
raw = r.lindex(QKEY, 0)
print("LINDEX 0 raw bytes length =", (len(raw) if raw else None))
if raw:
    from django_q.signing import SignedPackage
    pkg = SignedPackage.loads(raw)
    print("PACKAGE keys       =", sorted(pkg.keys()))
    print("PACKAGE id         =", pkg.get("id"))
    print("PACKAGE func       =", pkg.get("func"))
    print("PACKAGE name       =", pkg.get("name"))
    print("PACKAGE args       =", pkg.get("args"))
    kw = dict(pkg.get("kwargs") or {})
    # show the mail-specific override kwargs
    print("PACKAGE kwargs.override_filename =", kw.get("override_filename"))
    print("PACKAGE kwargs.override_title    =", kw.get("override_title"))
    print("PACKAGE kwargs.task_name (via name/kw) =", kw.get("task_name", pkg.get("name")))
    print("PACKAGE kwargs.path exists       =", "path" in kw, "->", kw.get("path"))
```

#### `bulk_edit.py` — Q7 bulk-edit driver, JSON body ([§9](#9-q7--enqueue-origin-in-code))

```python
import os, sys, time, json, requests, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from documents.models import Tag, Document
from django_q.models import Task

BASE="http://localhost:8000"; USER="admin"; PW=os.environ.get("QA_ADMIN_PW","<REDACTED>")

# setup: ensure a tag exists to add (test data; the ENTRY POINT under test is the REST bulk_edit endpoint)
tag,_=Tag.objects.get_or_create(name="qa_bulk_tag", defaults={"color":"#a6cee3"})
docs=list(Document.objects.order_by("-id").values_list("id",flat=True)[:2])
print("SETUP tag id=%s name=%s ; target documents=%s" % (tag.id, tag.name, docs))
tasks_before=Task.objects.filter(func="documents.tasks.bulk_update_documents").count()

s=requests.Session()
s.get(f"{BASE}/accounts/login/"); csrf=s.cookies.get("csrftoken")
r1=s.post(f"{BASE}/accounts/login/",
          data={"username":USER,"password":PW,"csrfmiddlewaretoken":csrf,"next":"/"},
          headers={"Referer":f"{BASE}/accounts/login/"}, allow_redirects=False)
print("LOGIN -> %d ; sessionid present=%s" % (r1.status_code, bool(s.cookies.get("sessionid"))))
csrf=s.cookies.get("csrftoken")

body={"documents":docs,"method":"add_tag","parameters":{"tag":tag.id}}
print("\nREQUEST  POST %s/api/documents/bulk_edit/" % BASE)
print("  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>")
print("  Content-Type: application/json")
print("  body =", json.dumps(body))
r2=s.post(f"{BASE}/api/documents/bulk_edit/", json=body,
          headers={"X-CSRFToken":csrf,"Referer":f"{BASE}/","Content-Type":"application/json"})
print("RESPONSE status =", r2.status_code)
print("RESPONSE body   =", repr(r2.text))

print("\ncorrelating (waiting for bulk_update_documents task)...")
target=None
for i in range(30):
    t=Task.objects.filter(func="documents.tasks.bulk_update_documents").order_by("-started").first()
    if t and t.stopped and Task.objects.filter(func="documents.tasks.bulk_update_documents").count()>tasks_before:
        target=t; break
    time.sleep(1)
if target:
    print("TASK id=%s func=%s success=%s time_taken=%.3fs result=%r"
          % (target.id, target.func, target.success, target.time_taken(), target.result))
    # confirm the tag was actually applied (effect of the bulk edit)
    applied=[d.id for d in Document.objects.filter(id__in=docs, tags__id=tag.id)]
    print("EFFECT documents now carrying tag %s: %s" % (tag.id, applied))
else:
    print("no bulk_update_documents task appeared within timeout")
```

### 13.3 Django-Q 1.3.x documentation cross-reference (research)

The queued-vs-running distinction (Q4) and the result-storage model (Q5/Q6) are implemented **inside the
`django-q` dependency**, not in the paperless-ngx source tree. Every runtime value in this document was
therefore verified against the installed Django-Q source on disk
(`/usr/local/lib/python3.9/site-packages/django_q/…`, cited by `file:line`); this subsection additionally
names the official Django-Q documentation sections that describe each mechanism, so the reader can confirm
the intended design behind the observed behavior.

**Documentation version vs. installed source (a nuance worth stating).** The official Django-Q manual at
`django-q.readthedocs.io` currently renders as **release 1.3.6** — its downloadable PDF is titled
"Django Q Documentation, Release 1.3.6" — whereas the version pinned and actually running here is
**django-q 1.3.9** (`requirements.txt`). 1.3.9 is a small maintenance increment over 1.3.6 that does not
change the broker/queue/result model exercised by Q4–Q6. Because the prose and the running code can in
principle diverge, this document treats the **installed 1.3.9 source as ground truth** for every value and
uses the docs only to name the authoritative mechanism. (Labeled **(documentation-derived)** wherever a claim
rests on the manual rather than on observed 1.3.9 output.)

**Section map — official docs → the behavior observed in this document:**

- **Brokers.** Describes the Redis broker's `enqueue`, which sends a task package to the broker queue and
  returns a tracking id (the Q3/Q7 enqueue primitive, [§5](#5-q3--how-a-job-appears-when-created)/[§9](#9-q7--enqueue-origin-in-code)),
  and `queue_size()`, documented as "the amount of messages in the broker's queue" — i.e. the **WAITING**
  count observed at the boundary in [§6](#6-q4--waiting-vs-actively-processing). The same section states the
  default Redis broker does not support message receipts, so an in-flight task lost to a crash or a worker
  timeout is **not** re-delivered; this is the documented basis for treating `retry=1810`s
  ([§4](#4-q2--services-involved)) as a safety bound rather than an active redelivery timer under the Redis
  broker. **(documentation-derived)**, and consistent with the observed `django_q/brokers/redis_broker.py`.
- **Architecture** (subsections *Signed Tasks · Broker · Pusher · Worker · Monitor · Sentinel · Timeouts ·
  Scheduler*). This is the design description of the exact multi-process topology dissected at runtime in
  [§3](#3-q1--live-async-behavior)/[§4](#4-q2--services-involved) and observed against
  `django_q/cluster.py` — the **Sentinel** (`Process-1`) that spawns the **Pusher**, **Worker**(s), and
  **Monitor**, and hosts the **Scheduler** on its guard loop.
- **Admin** (subsections *Successful tasks · Failed tasks · Scheduled tasks · Queued tasks*). Grounds the
  Q6 result surfaces ([§8](#8-q6--after-the-fact-status)): "Successful tasks" uses the `Success` proxy and
  "Failed tasks" the `Failure` proxy over the same `django_q_task` table. The docs explicitly note the
  **Queued tasks** admin page is available **only** under the Django ORM broker — which is precisely why it
  is **absent** in this deployment (Redis broker), a fact this document confirms rather than assumes.
- **Schedules** (*Schedule*, *Missed schedules*) together with the **Configuration** key `catch_up`. These
  describe the recurring-job model and the documented rule that, with `catch_up=False`, schedules missed
  while the cluster was down "run only once" on restart before normal cadence resumes — the exact behavior
  observed for the four seeded schedules in [§10](#10-scheduled--recurring-jobs).
- **Tasks** (`async_task()`) and **Monitor** (`Stat`). `async_task()` is the enqueue call named at every
  origin in [§9](#9-q7--enqueue-origin-in-code); `Stat` is the live per-cluster status object the Q4 poller
  reads ([§6](#6-q4--waiting-vs-actively-processing)). Observed against `django_q/tasks.py` and
  `django_q/monitor.py`.

In short: the official 1.3.6 documentation names the mechanisms (enqueue → broker queue → worker → monitor →
`django_q_task`; Success/Failure proxies; ORM-only Queued-tasks admin; `catch_up=False` single-shot
catch-up), and the installed **1.3.9** source — cited throughout by `file:line` — is what each observed
value in §1–§10 was actually measured against.
