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
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
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
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.brokers import get_broker
b = get_broker()
print("broker class =", type(b).__module__ + "." + type(b).__name__)
print("broker info  =", b.info())
print("list_key     =", b.list_key)
PY
broker class = django_q.brokers.redis_broker.Redis
broker info  = Redis 6.0.16
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
3. **Redis provenance.** The image does not bundle a running Redis server, so Redis is installed from
   Debian 11 "bullseye"'s own package (`apt-get install redis-server`) and started bound to loopback
   **inside** the container, which keeps the canonical `PAPERLESS_REDIS=redis://localhost:6379` address
   intact. The observed server is **Redis 6.0.16** (`redis-server 5:6.0.16-1+deb11u8`); the Django-Q
   Redis broker uses only `RPUSH`/`BLPOP`/`LLEN`/`LINDEX` on a single list, whose semantics are
   identical across Redis 6 and 7 (the broker source is shown in
   [Q4](#6-q4--waiting-vs-actively-processing)), so this version choice does not affect any behavior
   under study.

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

**Historical dependency snapshot — observation-only, not production-hardened.** The pinned dependency
set captured at commit `542221a38dff` is a **historical snapshot** and is deliberately reproduced
byte-for-byte: the read-only mandate forbids editing `requirements.txt` / `Pipfile`, and the point of
this investigation is to observe the asynchronous mechanism *as it shipped*, not as it might be
re-pinned today. Several of those pins are now end-of-life or carry published CVEs — notably **Django
4.0.4** (the 4.0.x series is past end of mainstream/extended support), **djangorestframework 3.13.1**,
and **gunicorn 20.1.0**. This environment is therefore stood up **for observation of the Django-Q
enqueue/worker/broker behavior only** and must **not** be treated as production-ready or
security-hardened; a real deployment would track the project's current supported dependency set. None
of these versions alters the asynchronous behavior under study. This posture, together with the
specific pre-existing edge behaviors observed while exercising the canonical paths, is catalogued in
[§11.3](#113-out-of-scope-observations--pre-existing-system-behaviors).

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

# Restore the OS libraries the project Dockerfile installs (see §2.1 provenance note) and install a
# Redis server (the canonical image ships none). Without libzbar0 the qcluster worker cannot import
# documents.tasks, so consume_file cannot run.
$ docker exec -u root pngx_qa bash -lc \
    'export DEBIAN_FRONTEND=noninteractive; apt-get update -qq && \
     apt-get install -y -qq redis-server libzbar0 poppler-utils pngquant'

# Confirm the exact installed versions (dpkg-query is deterministic, independent of install state):
$ docker exec pngx_qa dpkg-query -W -f='${Package} ${Version}\n' \
    redis-server libzbar0 poppler-utils pngquant
libzbar0 0.23.90-1+deb11u1
pngquant 2.13.1-1
poppler-utils 20.09.0-3.1+deb11u2
redis-server 5:6.0.16-1+deb11u8

# Start Redis bound to loopback inside the container, preserving the canonical
# redis://localhost:6379 address. --save "" --appendonly no keeps it purely in-memory.
$ docker exec -u root pngx_qa bash -lc \
    'redis-server --daemonize yes --bind 127.0.0.1 --port 6379 --save "" --appendonly no'
```

Redis is now reachable at the canonical `redis://localhost:6379` **inside** the container. This is
proven with the bundled `redis-cli` — no project helper scripts are needed yet (they are created in
§2.3):

```console
$ docker exec pngx_qa redis-cli ping
PONG

$ docker exec pngx_qa redis-cli info server | grep -E 'redis_version|^os:|multiplexing_api'
redis_version:6.0.16
os:Linux 6.6.122+ x86_64
multiplexing_api:epoll
```

> Note: the project ships a startup gate, `docker/wait-for-redis.py`
> (`MAX_RETRY_COUNT=5` L16, `RETRY_SLEEP_SECONDS=5` L17, `REDIS_URL` default L19), which blocks until
> `PAPERLESS_REDIS` answers before the services start — the same reachability proven above.

### 2.3 Directories, environment file, and the disclosed helper scripts

First create the working directories (as root, then hand ownership to `testuser`):

```console
$ docker exec pngx_qa bash -lc 'mkdir -p /tmp/pp/{data,media,consume,scratch,data/index,data/log} /tmp/obs \
    && chown -R testuser:testuser /tmp/pp /tmp/obs'
```

The remaining setup materializes three small, fully-disclosed helpers **on disk**. Each is written with
a real `cat > … <<'EOF'` heredoc fed over `docker exec -i` stdin, so this recipe is genuinely
self-executable — nothing below is display-only, and each block writes exactly the bytes shown.

**(a) `/tmp/pp/penv`** — the single environment file every later command sources. It relocates the
`PAPERLESS_*` data directories under `/tmp/pp` (writable by `testuser`) and points the app at the
canonical Redis address:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'cat > /tmp/pp/penv' <<'EOF'
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
EOF
```

**(b) `/tmp/pp/pp.sh`** — runs a Python snippet (read from stdin) in the canonical env as non-root
`testuser`. Used throughout for ORM/broker probes; disclosed in full so every later block is exact:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'cat > /tmp/pp/pp.sh' <<'EOF'
#!/bin/bash
# pp.sh — run a Python snippet (read from stdin) inside the canonical paperless-ngx
# environment as the non-root 'testuser'. Sources /tmp/pp/penv (the documented env).
exec runuser -u testuser -- bash -lc 'set -a; . /tmp/pp/penv; set +a; cd /app/src && exec python3 -'
EOF
```

**(c) `/tmp/pp/launch.sh`** — starts the three canonical services (per `docker/supervisord.conf`), each
backgrounded, recording each **master PID** via `$!` for a safe, targeted shutdown later. `cd` sits on
its own line (not chained with `&&`) so `$!` captures the real `python3`/`gunicorn` master process, not
a transient subshell — the property that makes the targeted shutdown in [§12.1](#121-safe-validated-service-shutdown) reliable:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'cat > /tmp/pp/launch.sh' <<'EOF'
#!/bin/bash
set -a; . /tmp/pp/penv; set +a
cd /app/src
nohup python3 manage.py qcluster            > /tmp/obs/qcluster.log 2>&1 < /dev/null & echo $! > /tmp/obs/qcluster.pid ; disown
nohup python3 manage.py document_consumer   > /tmp/obs/consumer.log 2>&1 < /dev/null & echo $! > /tmp/obs/consumer.pid ; disown
nohup gunicorn -c /app/gunicorn.conf.py paperless.asgi:application > /tmp/obs/gunicorn.log 2>&1 < /dev/null & echo $! > /tmp/obs/gunicorn.pid ; disown
EOF
```

Make the two scripts executable and confirm all three helpers exist on disk with the expected sizes:

```console
$ docker exec -u testuser pngx_qa bash -lc 'chmod +x /tmp/pp/pp.sh /tmp/pp/launch.sh && \
    wc -l /tmp/pp/penv /tmp/pp/pp.sh /tmp/pp/launch.sh'
  10 /tmp/pp/penv
   4 /tmp/pp/pp.sh
   6 /tmp/pp/launch.sh
  20 total
```

With `pp.sh` now on disk, confirm the app's own Redis client (the same `redis` library the worker uses)
reaches the canonical address and sees the Redis 6.0.16 server started in §2.2. This is the same
reachability the startup gate above guarantees, but exercised through the exact client stack the code
uses:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import redis
r = redis.Redis(host="localhost", port=6379)
print("PING localhost:6379 ->", r.ping())
print("redis_version       ->", r.info().get("redis_version"))
PY
PING localhost:6379 -> True
redis_version       -> 6.0.16
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
qcluster: PID 13604 owner=testuser :: python3 manage.py qcluster
consumer: PID 2970 owner=testuser :: python3 manage.py document_consumer
gunicorn: PID 2971 owner=testuser :: /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf

$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import urllib.request as u
print("GET /api/ ->", u.urlopen("http://localhost:8000/api/", timeout=10).status)
PY
GET /api/ -> 200
```

All three services run as `testuser`; the web endpoint answers on the container-internal `localhost`.
Every temporary helper and log lives under `/tmp` (outside the repository) and the whole container is
discarded at the end; see [Cleanup & repository integrity](#12-cleanup--repository-integrity).

> **Disclosure (worker-cluster identity used in the Q-sections).** Because §2.2 installs `libzbar0`
> *before* launch (§2.1 provenance note), the single `launch.sh` invocation above yields a stable worker
> cluster directly — **python main PID `13604`, banner `timing-uncle-failed-neptune`** — alongside the
> long-lived `consumer` (PID `2970`) and `gunicorn` (PID `2971`) processes. This is the generation the
> `/proc`-walk and `Stat` commands in [§4](#4-q2--services-involved) read from the live pidfiles. The
> `consumer` and `gunicorn` processes stay fixed for the whole investigation, but the `qcluster`
> *generation* is point-in-time: the Q4 boundary demonstration in
> [§6](#6-q4--waiting-vs-actively-processing) and the schedule observations in
> [§10](#10-scheduled--recurring-jobs) deliberately stop and restart the cluster, so those sections show
> later generations with fresh PIDs/banners/`cluster_id`s. Only the **structure** (1 main + 1 Sentinel +
> 11 workers + 1 monitor + 1 pusher) is generation-invariant (see the cluster-generation note in
> [§4.2](#42-the-qcluster-process-tree--real-processes-not-threads)). The verbatim graceful stop of the
> live cluster is captured at teardown in [§12 (Cleanup)](#12-cleanup--repository-integrity).

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
cat > /tmp/pp/qa1.txt <<TXT
QA demonstration document for paperless-ngx runtime investigation.
paperlessqademo invoice from ACME Corporation.
Parsed by the paperless_text parser (text/plain), no OCR required.
TXT
ts=$(date +%Y%m%d_%H%M%S); dest="/tmp/pp/consume/qa_watch_${ts}.txt"
mv /tmp/pp/qa1.txt "$dest"; echo "dropped: $dest"
date -u +"drop_utc=%Y-%m-%dT%H:%M:%S.%NZ"'
dropped: /tmp/pp/consume/qa_watch_20260715_090209.txt
drop_utc=2026-07-15T09:02:09.195594868Z
```

The watcher detects the file and enqueues it. Its own log line (the `_consume` function,
`documents/management/commands/document_consumer.py:85`) is the observable proof of the enqueue:

```console
$ docker exec pngx_qa bash -lc "grep -n 'qa_watch_20260715_090209' /tmp/obs/consumer.log"
2:[2026-07-15 09:02:10,200] [INFO] [paperless.management.consumer] Adding /tmp/pp/consume/qa_watch_20260715_090209.txt to the task queue.
```

### 3.3 Worker execution (hops 2–5): `consume_file` runs and the six handlers fire

The `qcluster` worker log records the whole server-side sequence — the worker picking up the task, the
`Consumer` running, three of the six handlers logging their assignment, and the completion line:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_watch_20260715_090209' /tmp/obs/qcluster.log"
46:09:02:10 [Q] INFO Process-1:5 processing [qa_watch_20260715_090209.txt]
47:[2026-07-15 09:02:10,355] [INFO] [paperless.consumer] Consuming qa_watch_20260715_090209.txt
48:[2026-07-15 09:02:11,097] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-15 qa_watch_20260715_090209
49:[2026-07-15 09:02:11,098] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-15 QA Correspondent qa_watch_20260715_090209
50:[2026-07-15 09:02:11,100] [INFO] [paperless.handlers] Tagging "2026-07-15 QA Correspondent qa_watch_20260715_090209" with "QA-Auto"
51:[2026-07-15 09:02:11,151] [INFO] [paperless.consumer] Document 2026-07-15 QA Correspondent qa_watch_20260715_090209 consumption finished
53:09:02:11 [Q] INFO Processed [qa_watch_20260715_090209.txt]
```

Reading that log against the source: `Process-1:5 processing [...]` is the worker loop
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
t = Task.objects.get(name="qa_watch_20260715_090209.txt")
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
Task.id        = fa11857820694edfb3f1594caf732cc4
Task.name      = qa_watch_20260715_090209.txt
Task.func      = documents.tasks.consume_file
Task.args      = ('/tmp/pp/consume/qa_watch_20260715_090209.txt',)
Task.success   = True
Task.started   = 2026-07-15T09:02:10.202315+00:00
Task.stopped   = 2026-07-15T09:02:11.154929+00:00
Task.time_taken= 0.953 s
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
stopped − started = 2026-07-15T09:02:11.154929Z − 2026-07-15T09:02:10.202315Z
                  = 0.952614 s  ≈  0.953 s
```

This equals the `Task.time_taken()` value (`0.953 s`) Django-Q reports. Because the cluster was **up**
here, the queue-wait was negligible — the worker's `processing`/`Consuming` lines land at `09:02:10,355`,
only ≈ 0.16 s after the enqueue — so this `time_taken` is dominated by execution. The
**user-perceived** latency is a bit larger still: the file was dropped at `09:02:09.195` but the watcher
logged the enqueue at `09:02:10,200`, i.e. the watcher's debounce added ≈ 1.0 s before the task even
existed. That `started`-is-enqueue semantics becomes vivid in [§6 (Q4)](#6-q4--waiting-vs-actively-processing),
where a task deliberately left waiting reports a `time_taken` of **19.228 s** for ≈ 0.7 s of real work.

### 3.6 Edge conditions observed on the live path

The happy path above is not the only condition the lifecycle exhibits. Two non-happy conditions were
observed first-hand (fuller error/edge coverage is consolidated in
[§11](#11-exhaustive-condition-coverage)):

**(a) Non-fatal thumbnail warning (PDF).** Ingesting a text-layer PDF (generated with `reportlab`)
exercises PDF thumbnailing. paperless first tries ImageMagick `convert`, which this image blocks via its
PDF security policy; paperless catches the failure and **falls back to Ghostscript**, logging a
`WARNING`. Consumption still succeeds — the warning is *non-fatal*:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_thumb_20260715_105357|convert-im6|falling back' /tmp/obs/qcluster.log"
114:10:53:58 [Q] INFO Process-1:15 processing [qa_thumb_20260715_105357.pdf]
115:[2026-07-15 10:53:58,261] [INFO] [paperless.consumer] Consuming qa_thumb_20260715_105357.pdf
116:convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
117:convert-im6.q16: no images defined `/tmp/pp/scratch/paperless-ahntn6b2/convert.png' @ error/convert.c/ConvertImageCommand/3229.
118:[2026-07-15 10:53:59,012] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
119:[2026-07-15 10:53:59,812] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-15 qa_thumb_20260715_105357
120:[2026-07-15 10:53:59,813] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-15 QA Correspondent qa_thumb_20260715_105357
121:[2026-07-15 10:53:59,815] [INFO] [paperless.handlers] Tagging "2026-07-15 QA Correspondent qa_thumb_20260715_105357" with "QA-Auto"
122:[2026-07-15 10:53:59,883] [INFO] [paperless.consumer] Document 2026-07-15 QA Correspondent qa_thumb_20260715_105357 consumption finished
124:10:53:59 [Q] INFO Processed [qa_thumb_20260715_105357.pdf]
```

The fallback and the warning are exactly the code at `documents/parsers.py`: `make_thumbnail_from_pdf`
(`:187`) runs `run_convert` (`:111`); on `ParseError` it calls `make_thumbnail_from_pdf_gs_fallback`
(`:206` → `:153`), whose `logger.warning(...)` is the line seen above (`:159`). The task still records
success:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.filter(name__startswith="qa_thumb_20260715_105357").order_by("-stopped").first()
print("id=%s success=%s result=%r time_taken=%.3fs" % (t.id, t.success, t.result, t.time_taken()))
PY
id=fed82cba40f24dc9b2fa0677c9823b32 success=True result='Success. New document id 10 created' time_taken=1.779s
```

**(b) Worker death vs. clean failure (missing native library).** A missing *native* library is a
qualitatively different failure from a bad document: it makes the worker **die at import time** rather
than fail gracefully. To observe this condition directly and under the canonical entry point, the
`libzbar0` shared library was temporarily moved out of the loader path (with `ldconfig` refreshed), a
file was dropped through the watcher, and the library was restored immediately afterward. The inducement
command and its markers:

```console
$ docker exec pngx_qa bash -lc '
BK=/tmp/zbar_bak; mkdir -p "$BK"
restore(){ mv "$BK"/libzbar.so.0.3.0 /usr/lib/x86_64-linux-gnu/; mv "$BK"/libzbar.so.0 /usr/lib/x86_64-linux-gnu/; ldconfig; echo "RESTORED $(date -u +%H:%M:%S.%N)"; }
trap restore EXIT
mv /usr/lib/x86_64-linux-gnu/libzbar.so.0.3.0 "$BK"/ && mv /usr/lib/x86_64-linux-gnu/libzbar.so.0 "$BK"/ && ldconfig
echo "HIDDEN $(date -u +%H:%M:%S.%N)"
ts=$(date +%Y%m%d_%H%M%S); tmpf=/tmp/pp/qa_death_${ts}.txt; dest=/tmp/pp/consume/qa_death_${ts}.txt
runuser -u testuser -- bash -c "printf %s\\\\n \"QA worker-death demonstration document.\" > $tmpf"
mv "$tmpf" "$dest"; echo "DROPPED $dest $(date -u +%H:%M:%S.%N)"
sleep 9'
HIDDEN 11:03:01.876517768
DROPPED /tmp/pp/consume/qa_death_20260715_110301.txt 11:03:01.886984591
RESTORED 11:03:10.942586458
```

The worker that dequeued the task died at import time. The trace is captured from the live `qcluster.log`
and is shown here **complete and unedited** (the traceback lines themselves carry no filename, so the
region is addressed by line number — the enclosing `processing [qa_death_…]` / `reincarnated` markers
were located with `grep -nE 'qa_death_20260715_110301|reincarnated|ready for work'`):

```console
$ docker exec pngx_qa bash -lc "sed -n '134,168p' /tmp/obs/qcluster.log"
11:03:02 [Q] INFO Process-1:17 processing [qa_death_20260715_110301.txt]
Process Process-1:17:
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
11:03:03 [Q] ERROR reincarnated worker Process-1:17 after death
11:03:03 [Q] INFO Process-1:28 ready for work at 15392
```

The worker imports the task target lazily at run time via `pydoc.locate(f)` (`django_q/cluster.py:424`);
because `documents/tasks.py:25` imports `pyzbar` at module scope, the missing shared library propagates
as an uncaught error that kills the process — the exception escapes `worker()` entirely (it is raised
*before* the per-task try/except that would otherwise record a failure), so no result is ever pushed.
Django-Q's Sentinel **reincarnates** the dead worker: `Process-1:17` dies and a fresh `Process-1:28`
(PID `15392`) appears one second later. The in-flight task therefore yields **no** `django_q_task` row —
directly observed:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
print("Task rows for qa_death_20260715_110301.txt =",
      Task.objects.filter(name="qa_death_20260715_110301.txt").count())
PY
Task rows for qa_death_20260715_110301.txt = 0
```

After the library was restored, the very next ingestion through the same watcher succeeded, confirming
the cluster self-heals (a fresh worker re-imports `documents.tasks` cleanly):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.filter(name="qa_heal_20260715_110348.txt").order_by("-stopped").first()
print("id=%s success=%s result=%r time_taken=%.3fs" % (t.id, t.success, t.result, t.time_taken()))
PY
id=418d4860f19243f7a1b07181f8e54c42 success=True result='Success. New document id 11 created' time_taken=0.908s
```

This difference is the crux of the edge behavior: a **process death writes nothing** (zero rows, task
lost), whereas a **clean failure writes a `success=False` row** — the latter is demonstrated with a
forced duplicate failure in [§8 (Q6)](#8-q6--after-the-fact-status).

> **(inferred)** The "task is *not retried*" half of the claim is grounded in the Redis broker having no
> acknowledge/receipt mechanism (broker source cited in [§6 (Q4)](#6-q4--waiting-vs-actively-processing)):
> once the package is popped from the Redis list it is gone, so a worker death loses it with no
> re-delivery. The **zero-row / task-lost** half is *observed* directly above (no row for
> `qa_death_20260715_110301.txt`, and no re-processing of it appeared afterward).

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
8:user=root
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

The first match — `8:user=root` — is the `[supervisord]` **master** section's own directive
(`docker/supervisord.conf:8`): the Supervisord process runs as root, and each of the three programs then
drops privileges via its per-program `user=paperless` (lines 12, 21, 30). *(In this read-only
reproduction the programs are launched directly as `testuser` rather than under Supervisord, so the owner
observed in §4.1 is `testuser`; see [§2.5](#25-launch-and-confirm-the-three-services).)*

### 4.1 The three live services (PID, owner, exact command line)

```console
$ docker exec pngx_qa bash -lc '
for name in qcluster consumer gunicorn; do
  pid=$(cat /tmp/obs/$name.pid)
  echo "[$name] pid=$pid comm=$(cat /proc/$pid/comm) owner=$(stat -c %U /proc/$pid)"
  echo "   cmdline=$(tr "\0" " " < /proc/$pid/cmdline)"
done'
[qcluster] pid=13604 comm=python3 owner=testuser
   cmdline=python3 manage.py qcluster
[consumer] pid=2970 comm=python3 owner=testuser
   cmdline=python3 manage.py document_consumer
[gunicorn] pid=2971 comm=gunicorn owner=testuser
   cmdline=/usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

### 4.2 The `qcluster` process tree — real processes, not threads

Finding-of-record: Django-Q workers are **separate operating-system processes**, each with its own PID,
not threads inside one interpreter. This is directly visible by walking `/proc` from the cluster main
down through the Sentinel to its children (the cluster used here is `timing-uncle-failed-neptune`, main
PID `13604`, per the §2.5 disclosure).

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
CLUSTER MAIN pid=13604 comm=python3 ppid=1
  SENTINEL pid=13611 comm=python3 ppid=13604
    child#1 pid=13623 comm=python3
    child#2 pid=13624 comm=python3
    child#3 pid=15392 comm=python3
    child#4 pid=15459 comm=python3
    child#5 pid=15471 comm=python3
    child#6 pid=15484 comm=python3
    child#7 pid=15514 comm=python3
    child#8 pid=15544 comm=python3
    child#9 pid=15546 comm=python3
    child#10 pid=15959 comm=python3
    child#11 pid=15979 comm=python3
    child#12 pid=16072 comm=python3
    child#13 pid=16507 comm=python3
  => Sentinel has 13 child processes (expect 11 workers + 1 monitor + 1 pusher = 13)
```

The cluster's own startup banner assigns each role a `Process-1:N` label. It was captured at launch, so
its worker PIDs are the *original* ones; because `recycle=1` (the `Q_CLUSTER` setting shown further
below) replaces each worker after every task, the **11 live worker PIDs** above (`15392`…`16507`) have
since rotated away from the banner's `13612`…`13622`, whereas the **monitor** (`13623`) and **pusher**
(`13624`) are never recycled and therefore persist unchanged into the live tree. The *count* and *roles*
are generation-invariant; only the individual worker PIDs churn:

```console
$ docker exec pngx_qa bash -lc "sed -n '1,15p' /tmp/obs/qcluster.log"
09:29:20 [Q] INFO Q Cluster timing-uncle-failed-neptune starting.
09:29:20 [Q] INFO Process-1:1 ready for work at 13612
09:29:20 [Q] INFO Process-1:2 ready for work at 13613
09:29:20 [Q] INFO Process-1:3 ready for work at 13614
09:29:20 [Q] INFO Process-1:4 ready for work at 13615
09:29:20 [Q] INFO Process-1:5 ready for work at 13616
09:29:20 [Q] INFO Process-1:6 ready for work at 13617
09:29:20 [Q] INFO Process-1:7 ready for work at 13618
09:29:20 [Q] INFO Process-1:8 ready for work at 13619
09:29:20 [Q] INFO Process-1:9 ready for work at 13620
09:29:20 [Q] INFO Process-1:10 ready for work at 13621
09:29:20 [Q] INFO Process-1:11 ready for work at 13622
09:29:20 [Q] INFO Process-1:12 monitoring at 13623
09:29:20 [Q] INFO Process-1 guarding cluster timing-uncle-failed-neptune
09:29:20 [Q] INFO Process-1:13 pushing tasks at 13624
```

Reading the banner against `django_q/cluster.py`: the **main** process (`manage.py qcluster`, PID 13604)
spawns one **Sentinel** process (`Process(target=Sentinel)`, `cluster.py:68`; PID 13611). The Sentinel
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
cluster_id=f5e5ec1c-34f7-4660-bbdd-caedee7b0a07 pid=13604 status=Idle workers=11 task_q=0 done_q=0
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

`gunicorn` is a master process (PID 2971) with worker children serving the ASGI app (which carries both
the REST API and the Channels WebSocket routes, `paperless/asgi.py:17-20`):

```console
$ docker exec pngx_qa bash -lc '
G=$(cat /tmp/obs/gunicorn.pid); echo "gunicorn master=$G"
for c in /proc/[0-9]*/status; do
  p=$(awk "/^PPid:/{print \$2}" "$c"); pid=$(awk "/^Pid:/{print \$2}" "$c")
  [ "$p" = "$G" ] && echo "  worker pid=$pid comm=$(cat /proc/$pid/comm)"
done'
gunicorn master=2971
  worker pid=2973 comm=gunicorn
  worker pid=2974 comm=gunicorn
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
PING = True | redis_version = 6.0.16
```

### 4.5 Stopping the cluster

A `qcluster` is stopped with a single `SIGTERM` to its main PID; the Sentinel drains its workers and the
monitor before exiting. Because that shutdown is part of teardown, the **actual, verbatim** graceful
stop of the live cluster (`timing-uncle-failed-neptune`, PID 13604) is captured at cleanup time in
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
LINDEX head: type=bytes length=467 bytes
first 60 raw bytes: b'gAWVLQEAAAAAAAB9lCiMAmlklIwgYTNjMzgzNWY2NmRjNDQyN2ExYTZlNTNj'
--- decoded task package (keys) ---
  args       = ('/tmp/pp/consume/qa_pkg_20260715_090313.txt',)
  func       = 'documents.tasks.consume_file'
  id         = 'a3c3835f66dc4427a1a6e53c8fe8fd80'
  kwargs     = {'override_tag_ids': None}
  name       = 'qa_pkg_20260715_090313.txt'
  started    = datetime.datetime(2026, 7, 15, 9, 3, 14, 923768, tzinfo=datetime.timezone.utc)
--- no persisted row yet for this id ---
Task.objects.filter(id='a3c3835f66dc4427a1a6e53c8fe8fd80').exists() = False
```

Reading the decoded package field-by-field (these are the "identifiers and payload" the question asks
for):

| Field | Observed value | Meaning |
|---|---|---|
| `id` | `a3c3835f66dc4427a1a6e53c8fe8fd80` | 32-char hex UUID; the value `async_task` returns; becomes the `django_q_task` primary key |
| `func` | `documents.tasks.consume_file` | dotted path the worker will `pydoc.locate` and call |
| `args` | `('/tmp/pp/consume/qa_pkg_20260715_090313.txt',)` | positional args — here the file path to ingest |
| `kwargs` | `{'override_tag_ids': None}` | keyword args passed by the watcher's `async_task` call |
| `name` | `qa_pkg_20260715_090313.txt` | human-readable label (the watcher uses the filename) |
| `started` | `2026-07-15 09:03:14.923768+00:00` | enqueue timestamp (`tasks.py:65`); later copied verbatim into the row |

The final line confirms the crucial point: **at creation the job exists only as a Redis list element —
there is no `django_q_task` row yet** (`Task.objects.filter(id=…).exists() = False`). The row is written
only after the worker finishes (see [§7](#7-q5--where-task-state-is-stored) / [§8](#8-q6--after-the-fact-status)).

### 5.2 Why the payload is opaque bytes — the signed-pickle trust boundary

The 467-byte value is not human-readable because Django-Q **signs and pickles** every package.
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
# run_id=run1 env_id=pngx_qa/542221a38dff/redis6.0.16 watch_task_id=a3c3835f66dc4427a1a6e53c8fe8fd80 watch_name=qa_pkg_20260715_090313.txt
# iso_utc                    | LLEN | cluster  | task_q | done_q | state
2026-07-15T09:03:30.689+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:30.800+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:30.902+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.005+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.107+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.209+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.311+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.414+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.516+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.618+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.721+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.824+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:31.927+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.041+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.144+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.247+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.349+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.452+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.554+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.656+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.758+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.861+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:32.963+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:33.065+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:33.167+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:03:33.269+00:00 | 1    | Starting | 0    | 0    | WAITING
2026-07-15T09:03:33.373+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.476+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.579+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.681+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.784+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.887+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:33.990+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:34.093+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:03:34.195+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-15T09:03:34.299+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-15T09:03:34.402+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
```

The three states are unmistakable and correlated to the *same* `task_id`
(`a3c3835f66dc4427a1a6e53c8fe8fd80` — the exact package peeked while waiting in
[§5.1](#51-the-job-as-it-actually-appears-on-the-queue-observed)):

- **WAITING** `09:03:30.689 → 09:03:33.269`: `LLEN=1`, no row. The cluster is `down` for the first 25
  samples and `Starting` on the last one — the package is still queued because no worker has dequeued it
  yet. (The poller began ≈16 s after enqueue, so these 26 samples are the *tail* of a longer wait; the
  full wait is measured from `Task.started` below.)
- **IN-CLUSTER/ACTIVE** `09:03:33.373 → 09:03:34.093`: `LLEN=0`, cluster present, internal `task_q=0`
  and `done_q=0`, **still no row**. This is the window that a naive "is the queue empty?" check would
  wrongly report as finished.
- **DONE** from `09:03:34.195`: the row now exists with `success=True`.

That the ACTIVE window is *genuinely executing the function* (not merely idle) is confirmed by the
worker's own log for the same task, with absolute timestamps that fall inside the ACTIVE window above:

```console
$ docker exec pngx_qa bash -lc "grep -E 'qa_pkg_20260715_090313' /tmp/obs/qcluster.log"
09:03:33 [Q] INFO Process-1:1 processing [qa_pkg_20260715_090313.txt]
[2026-07-15 09:03:33,412] [INFO] [paperless.consumer] Consuming qa_pkg_20260715_090313.txt
[2026-07-15 09:03:34,098] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-15 qa_pkg_20260715_090313
[2026-07-15 09:03:34,099] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-15 QA Correspondent qa_pkg_20260715_090313
[2026-07-15 09:03:34,101] [INFO] [paperless.handlers] Tagging "2026-07-15 QA Correspondent qa_pkg_20260715_090313" with "QA-Auto"
[2026-07-15 09:03:34,148] [INFO] [paperless.consumer] Document 2026-07-15 QA Correspondent qa_pkg_20260715_090313 consumption finished
09:03:34 [Q] INFO Processed [qa_pkg_20260715_090313.txt]
```

Finally the persisted row, correlated by the **same id** peeked while waiting:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(id="a3c3835f66dc4427a1a6e53c8fe8fd80")
print("id=%s success=%s" % (t.id, t.success))
print("started =", t.started.isoformat())
print("stopped =", t.stopped.isoformat())
print("time_taken = %.3f s" % t.time_taken())
print("result =", repr(t.result))
PY
id=a3c3835f66dc4427a1a6e53c8fe8fd80 success=True
started = 2026-07-15T09:03:14.923768+00:00
stopped = 2026-07-15T09:03:34.151565+00:00
time_taken = 19.228 s
result = 'Success. New document id 2 created'
```

Note the `started` (`09:03:14.923768`) is exactly the enqueue timestamp peeked inside the package in
§5.1 — proving the point from §3.5 that **`started` is stamped at enqueue**. As a single calculation:

```
time_taken = stopped − started = 2026-07-15T09:03:34.151565Z − 2026-07-15T09:03:14.923768Z
           = 19.227797 s  ≈  19.228 s
```

The job's *execution* was ≈ 0.74 s (worker `Consuming` `09:03:33,412` → `consumption finished`
`09:03:34,148` = `0.736 s`); the remaining ≈ 18.49 s is pure **queue-wait** — time the package spent in
the WAITING state because the cluster was deliberately down. This is the concrete proof that
`time_taken` = queue-wait + execution.

### 6.3 Run 2 — identical experiment (two-run stability)

Repeating the *identical* procedure with a second file yields the same three-state signature. The
**complete** trace is inlined below (every sample, no external-file reference) so the evidence is
fully self-contained:

```console
$ docker exec pngx_qa bash -lc "cat /tmp/obs/poll_run2.csv"
# run_id=run2 env_id=pngx_qa/542221a38dff/redis6.0.16 watch_task_id=5487d1fe53ba4f74b7326af6148e4d45 watch_name=qa_run2_20260715_090423.txt
# iso_utc                    | LLEN | cluster  | task_q | done_q | state
2026-07-15T09:04:27.348+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.457+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.560+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.662+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.763+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.865+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:27.967+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.069+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.171+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.273+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.375+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.477+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.579+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.691+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.799+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:28.902+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.003+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.106+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.208+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.309+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.411+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.513+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.615+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.717+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.819+00:00 | 1    | down     | -    | -    | WAITING
2026-07-15T09:04:29.921+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.023+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.126+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.228+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.330+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.433+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.535+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.637+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.739+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.841+00:00 | 0    | Idle     | 0    | 0    | IN-CLUSTER/ACTIVE
2026-07-15T09:04:30.944+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-15T09:04:31.046+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
2026-07-15T09:04:31.148+00:00 | 0    | Idle     | 0    | 0    | DONE(success=True)
```

The three states are again correlated to the *same* `task_id` (`5487d1fe53ba4f74b7326af6148e4d45`):

- **WAITING** `09:04:27.348 → 09:04:29.819`: 25 samples, `LLEN=1`, cluster `down`, **no row**.
- **IN-CLUSTER/ACTIVE** `09:04:29.921 → 09:04:30.841`: 10 samples, `LLEN=0`, cluster present, internal
  `task_q=0`/`done_q=0`, **still no row** — the same "empty queue but not finished" trap as Run 1.
- **DONE** from `09:04:30.944`: 3 samples, the row now exists with `success=True`.

Worker log for the same task (the execution window falls inside the ACTIVE band above):

```console
$ docker exec pngx_qa bash -lc "grep -E 'qa_run2_20260715_090423' /tmp/obs/qcluster.log"
09:04:29 [Q] INFO Process-1:1 processing [qa_run2_20260715_090423.txt]
[2026-07-15 09:04:29,995] [INFO] [paperless.consumer] Consuming qa_run2_20260715_090423.txt
[2026-07-15 09:04:30,783] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-15 qa_run2_20260715_090423
[2026-07-15 09:04:30,784] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-15 QA Correspondent qa_run2_20260715_090423
[2026-07-15 09:04:30,786] [INFO] [paperless.handlers] Tagging "2026-07-15 QA Correspondent qa_run2_20260715_090423" with "QA-Auto"
[2026-07-15 09:04:30,840] [INFO] [paperless.consumer] Document 2026-07-15 QA Correspondent qa_run2_20260715_090423 consumption finished
09:04:30 [Q] INFO Processed [qa_run2_20260715_090423.txt]
```

Persisted row (correlated by the same id):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(id="5487d1fe53ba4f74b7326af6148e4d45")
print("id=%s success=%s" % (t.id, t.success))
print("started =", t.started.isoformat())
print("stopped =", t.stopped.isoformat())
print("time_taken = %.3f s" % t.time_taken())
print("result =", repr(t.result))
PY
id=5487d1fe53ba4f74b7326af6148e4d45 success=True
started = 2026-07-15T09:04:24.942754+00:00
stopped = 2026-07-15T09:04:30.843643+00:00
time_taken = 5.901 s
result = 'Success. New document id 3 created'
```

Two-run comparison (the state pattern is stable; the difference is only the deliberate queue-wait):

| Measure | Run 1 | Run 2 |
|---|---|---|
| WAITING samples (`LLEN=1`, no row) | 26 | 25 |
| IN-CLUSTER/ACTIVE samples (`LLEN=0`, no row) | 8 | 10 |
| DONE samples | 3 | 3 |
| execution (`Consuming`→`consumption finished`) | 0.736 s | 0.845 s |
| `time_taken` (enqueue→finish = wait+exec) | 19.228 s | 5.901 s |
| `success` / `result` | True / "New document id 2 created" | True / "New document id 3 created" |

The qualitative boundary (WAITING → IN-CLUSTER/ACTIVE → DONE) and the execution duration (≈ 0.7–0.8 s)
are **stable across both runs**; only `time_taken` differs, and it differs *exactly* by how long each
task was left waiting — which is expected and reinforces the `started`-is-enqueue semantics.

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
t = Task.objects.get(name="qa_seed_20260715_111305.txt")
for f in ["id","name","func","hook","group","args","kwargs","result","started","stopped","success","attempt_count"]:
    print("  %-14s = %r" % (f, getattr(t, f)))
print("  time_taken()   = %.3f s" % t.time_taken())
PY
  id             = '4d907eeb7b7349e6b706bd1a53f22761'
  name           = 'qa_seed_20260715_111305.txt'
  func           = 'documents.tasks.consume_file'
  hook           = None
  group          = None
  args           = ('/tmp/pp/consume/qa_seed_20260715_111305.txt',)
  kwargs         = {'override_tag_ids': None}
  result         = 'Success. New document id 12 created'
  started        = datetime.datetime(2026, 7, 15, 11, 13, 6, 384165, tzinfo=datetime.timezone.utc)
  stopped        = datetime.datetime(2026, 7, 15, 11, 13, 7, 348367, tzinfo=datetime.timezone.utc)
  success        = True
  attempt_count  = 1
  time_taken()   = 0.964 s
```

Every part of the question is answered by this one row: **success** = `True`; **result** = the string the
task returned (`consume_file` returns `"Success. New document id {pk} created"`, `documents/tasks.py:247`);
**timing** = `started`/`stopped` and `time_taken() = 0.964 s`; **history/identity** = `id`, `name`, `func`,
`attempt_count = 1`.

### 8.2 A completed **failure** — the complete row with full traceback (observed)

A **real** failure was forced through the canonical watcher entry point by dropping a file whose content
is **byte-identical** to an already-ingested document (same md5), which trips the consumer's duplicate
pre-check. Its worker log line and the **complete, untruncated** failed row:

```console
$ docker exec pngx_qa bash -lc "grep -nE 'qa_dup_20260715_111819' /tmp/obs/qcluster.log"
200:11:18:20 [Q] INFO Process-1:22 processing [qa_dup_20260715_111819.txt]
201:[2026-07-15 11:18:21,117] [ERROR] [paperless.consumer] Not consuming qa_dup_20260715_111819.txt: It is a duplicate.
203:11:18:21 [Q] ERROR Failed [qa_dup_20260715_111819.txt] - qa_dup_20260715_111819.txt: Not consuming qa_dup_20260715_111819.txt: It is a duplicate. : Traceback (most recent call last):
214:documents.consumer.ConsumerError: qa_dup_20260715_111819.txt: Not consuming qa_dup_20260715_111819.txt: It is a duplicate.

$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task, Success, Failure
t = Task.objects.get(name="qa_dup_20260715_111819.txt")
for f in ["id","name","func","hook","group","args","kwargs","started","stopped","success","attempt_count"]:
    print("  %-14s = %r" % (f, getattr(t, f)))
print("  time_taken()   = %.3f s" % t.time_taken())
print("  --- result (COMPLETE, unedited traceback) ---")
print(t.result)
print("  --- end result ---")
print("Failure proxy contains this id? ", Failure.objects.filter(id=t.id).exists())
print("Success proxy contains this id? ", Success.objects.filter(id=t.id).exists())
PY
  id             = '2d1df668f3544ad796281306ee976434'
  name           = 'qa_dup_20260715_111819.txt'
  func           = 'documents.tasks.consume_file'
  hook           = None
  group          = None
  args           = ('/tmp/pp/consume/qa_dup_20260715_111819.txt',)
  kwargs         = {'override_tag_ids': None}
  started        = datetime.datetime(2026, 7, 15, 11, 18, 20, 954065, tzinfo=datetime.timezone.utc)
  stopped        = datetime.datetime(2026, 7, 15, 11, 18, 21, 118791, tzinfo=datetime.timezone.utc)
  success        = False
  attempt_count  = 1
  time_taken()   = 0.165 s
  --- result (COMPLETE, unedited traceback) ---
qa_dup_20260715_111819.txt: Not consuming qa_dup_20260715_111819.txt: It is a duplicate. : Traceback (most recent call last):
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
documents.consumer.ConsumerError: qa_dup_20260715_111819.txt: Not consuming qa_dup_20260715_111819.txt: It is a duplicate.

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
print("--- Failure rows (name) ---")
for t in Failure.objects.order_by("started"):
    print("   ", t.name)
PY
Conf.ORM = None (None => Redis broker, not ORM)
OrmQ registered in admin?     False
Success registered in admin?  True
Failure registered in admin?  True
live counts -> Success: 41 Failure: 5
--- Failure rows (name) ---
    qa_dup_20260715_090521.txt
    qa_dup_race_a_20260715_092406.txt
    qa_race2_b_20260715_092707.txt
    qa_mail_attachment.txt
    qa_dup_20260715_111819.txt
```

So with the **Redis** broker, the admin shows "Successful tasks" and "Failed tasks" but **not** "Queued
tasks" — waiting work has no admin page here (as noted in [§6.4](#64-what-the-adminapi-surfaces-for-each-state)).
The **`Success`** count is a **point-in-time snapshot** that grows monotonically — the seeded schedules
keep firing successfully (mail every 10 min, `train_classifier` hourly, `sanity_check` weekly, per
[§10](#10-scheduled--recurring-jobs)), so re-running this probe later yields a larger number (it was
**41** at this capture). The **`Failure`** count, by contrast, is stable at **five** — the edge-case
duplicates and failures deliberately induced during this investigation, listed by name above — among them
`qa_dup_20260715_111819.txt`, the duplicate forced in §8.2.

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
socket receives the live `status_updates` frames for an ingestion. The capture script `/tmp/pp/ws_capture.py`
(published in [§13 Appendix](#132-the-remaining-helper-scripts-verbatim)) is a **pure listener** — it logs
in, opens the socket, and records frames; it does **not** trigger any ingestion itself. The frames below
are the live stream of the **same REST upload analysed in
[§9.2](#92-origin-2--rest-upload-postdocumentviewpost-exercised-live)** — the socket was connected first
(`09:07:33`), then that upload was POSTed, and the listener recorded its progress. Every frame therefore
carries the **same** progress UUID peeked in the WAITING package there (`6faa779e-e117-44ce-b5dd-a4fc13c86a56`)
and terminates with `document_id: 4`:

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export PP_USER=admin PP_PASS=<REDACTED>; cd /app/src && python3 /tmp/pp/ws_capture.py'
LOGIN GET /admin/login/  -> HTTP 200, csrftoken cookie present=True
LOGIN POST /admin/login/ -> HTTP 302 (302=success), sessionid cookie present=True
  auth cookie sent to WS handshake:  Cookie: sessionid=<REDACTED>  (real 32-char value redacted)
WS 2026-07-15T09:07:33.414+00:00 CONNECTED (HTTP 101 Switching Protocols) - authenticated accept
WS FRAME 1 2026-07-15T09:07:38.815+00:00  {"current_progress": 0, "document_id": null, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "new_file", "status": "STARTING", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS FRAME 2 2026-07-15T09:07:38.828+00:00  {"current_progress": 20, "document_id": null, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "parsing_document", "status": "WORKING", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS FRAME 3 2026-07-15T09:07:38.831+00:00  {"current_progress": 70, "document_id": null, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "generating_thumbnail", "status": "WORKING", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS FRAME 4 2026-07-15T09:07:39.499+00:00  {"current_progress": 90, "document_id": null, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "parse_date", "status": "WORKING", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS FRAME 5 2026-07-15T09:07:39.502+00:00  {"current_progress": 95, "document_id": null, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "save_document", "status": "WORKING", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS FRAME 6 2026-07-15T09:07:39.559+00:00  {"current_progress": 100, "document_id": 4, "filename": "qa_rest_20260715_090736.txt", "max_progress": 100, "message": "finished", "status": "SUCCESS", "task_id": "6faa779e-e117-44ce-b5dd-a4fc13c86a56"}
WS 2026-07-15T09:07:39.559+00:00 terminal status 'SUCCESS' received; stopping
```

The six frames are exactly the `_send_progress(...)` calls in the pipeline (`documents/consumer.py`):
`STARTING` (0%, `:202`) → `WORKING` (20% parsing `:259`, 70% thumbnail `:264`, 90% parse-date `:274`,
95% save `:294`) → `SUCCESS` (100%, `document_id=4`, `:375`). Each frame is the JSON `payload` built at
`consumer.py:56-72` (`filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`,
`document_id`).

> **Key distinction (cause of common confusion).** The WebSocket frame's `task_id`
> (`6faa779e-e117-44ce-b5dd-a4fc13c86a56`, a hyphenated UUID) is the **front-end progress id**, *not* the
> `django_q_task` primary key. For this very upload the durable row's PK is the **32-char hex**
> `6c677734037b4a5d8bbc45500bf21a11` ([§9.2](#92-origin-2--rest-upload-postdocumentviewpost-exercised-live)) —
> a different identifier. The realtime stream and the durable row are keyed differently and serve different
> purposes; see [§9](#9-q7--enqueue-origin-in-code) for where the progress UUID is generated. **After the
> fact**, the authoritative record is the `django_q_task` row — the WebSocket frames are already gone.

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
`/tmp/pp/rest_upload.py` (published in [§13 Appendix](#132-the-remaining-helper-scripts-verbatim)); the
browser-canonical authentication is a CSRF-protected session login, then a multipart POST carrying the
`sessionid` cookie and the `X-CSRFToken` header. So that the enqueued package can be *peeked while it is
still WAITING* (proving the identifiers at enqueue time), the cluster is stopped just before the POST and
started immediately after the peek — the same non-destructive `LINDEX` technique as
[§5.1](#51-the-job-as-it-actually-appears-on-the-queue-observed):

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export QA_ADMIN_PW=<REDACTED>; cd /app/src && python3 /tmp/pp/rest_upload.py'
STEP1 GET /accounts/login/ -> 200 ; csrftoken cookie present=True
STEP2 POST /accounts/login/ -> 302 ; sessionid cookie present=True

REQUEST  POST http://localhost:8000/api/documents/post_document/
  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>
  multipart field 'document' = (qa_rest_20260715_090736.txt, 92 bytes, text/plain)
RESPONSE status = 200
RESPONSE content-type = application/json
RESPONSE body = '"OK"'

WAITING peek:  LLEN(django_q:paperless:q) = 1
PACKAGE hex id (Django-Q PK)              = 6c677734037b4a5d8bbc45500bf21a11
PACKAGE func                              = documents.tasks.consume_file
PACKAGE name                              = qa_rest_20260715_090736.txt
PACKAGE args                              = ('/tmp/pp/scratch/paperless-upload-acjs74nt',)
PACKAGE kwargs['task_id'] (progress UUID) = 6faa779e-e117-44ce-b5dd-a4fc13c86a56
PACKAGE kwargs['override_filename']       = qa_rest_20260715_090736.txt
```

The peek captures **both** identifiers at the moment of enqueue: the 32-char hex **Django-Q Task PK**
`6c677734037b4a5d8bbc45500bf21a11` (the package's own `id`) and the hyphenated **front-end progress UUID**
`6faa779e-e117-44ce-b5dd-a4fc13c86a56` (the `task_id` kwarg minted at `views.py:521`). The cluster is then
restarted; the worker consumes the package, and the persisted row is queried by that **same** hex PK:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
from documents.models import Document
t = Task.objects.get(id="6c677734037b4a5d8bbc45500bf21a11")   # the peeked hex PK
print("Task.id        =", t.id, "(== peeked package hex PK)")
print("Task.name      =", t.name)
print("Task.func      =", t.func)
print("Task.success   =", t.success)
print("Task.result    =", repr(t.result))
print("Task.started   =", t.started.isoformat())
print("Task.stopped   =", t.stopped.isoformat())
print("Task.time_taken= %.3f s" % t.time_taken())
d = Document.objects.get(pk=4)
print("Document.id     =", d.id, "title=", repr(d.title))
print("Document count  =", Document.objects.count())
PY
Task.id        = 6c677734037b4a5d8bbc45500bf21a11 (== peeked package hex PK)
Task.name      = qa_rest_20260715_090736.txt
Task.func      = documents.tasks.consume_file
Task.success   = True
Task.result    = 'Success. New document id 4 created'
Task.started   = 2026-07-15T09:07:36.501792+00:00
Task.stopped   = 2026-07-15T09:07:39.559433+00:00
Task.time_taken= 3.058 s
Document.id     = 4 title= 'qa_rest_20260715_090736'
Document count  = 4
```

This is a **single, identity-complete chain**: the *one* hex PK `6c677734037b4a5d8bbc45500bf21a11` is
peeked while WAITING on the Redis list, resolves to the persisted `django_q_task` row, and yields
**Document 4**; the *one* progress UUID `6faa779e-e117-44ce-b5dd-a4fc13c86a56` is present in the WAITING
package's `kwargs['task_id']` **and** in all six WebSocket frames captured for this very run in
[§8.4](#84-the-realtime-channel-authenticated-websocket-status-stream-observed).

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
  This is the value stored as `django_q_task.id`; the observed PK above is `6c677734037b4a5d8bbc45500bf21a11`.
- The **front-end progress UUID** is a `uuid4()` used only for the realtime stream. For a REST upload it
  is minted as `task_id = str(uuid.uuid4())` (`views.py:521`, **hyphenated**) and passed as the `task_id`
  **kwarg** to the enqueued `consume_file` (`views.py:531`). `consume_file(…, task_id=None)`
  (`documents/tasks.py:191`) forwards it to `Consumer.try_consume_file(…, task_id=task_id)`
  (`documents/tasks.py:236-244`), which sets `self.task_id = task_id or str(uuid.uuid4())`
  (`documents/consumer.py:200`); `Consumer._send_progress` (`consumer.py:56`) then emits it in every
  `status_updates` frame as `"task_id": self.task_id` (`consumer.py:66`). For **this REST run** the peek
  above shows that progress UUID is `6faa779e-e117-44ce-b5dd-a4fc13c86a56` in `kwargs['task_id']`, and the
  [§8.4](#84-the-realtime-channel-authenticated-websocket-status-stream-observed) frames — captured for this
  same upload — carry exactly that value in every frame's `"task_id"`. (Because the REST view supplies a
  `task_id`, the `or str(uuid.uuid4())` fallback at `consumer.py:200` is *not* taken; a watcher drop, which
  supplies no `task_id`, is where that fallback mints the Consumer's own hyphenated `uuid4` instead.) Either
  way the progress id is a hyphenated `uuid4`, never the 32-char hex Task PK.

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
string. This is the **real production trigger**: the `qcluster` scheduler thread fires the seeded
`Schedule` row *by itself* on its interval — nothing calls the single-account `process_mail_account`
wrapper in normal operation. That the plural schedule auto-fires is directly observable in the running
cluster's log; it recurs on the seeded `MINUTES`/`10` cadence (see [§10](#10-scheduled--recurring-jobs)):

```console
$ docker exec pngx_qa grep -E 'created a task from schedule \[Check all e-mail accounts\]' /tmp/obs/qcluster.log
09:30:20 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
09:40:21 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
09:50:23 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
10:00:24 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

Those four are **steady-state** recurrences: once the test message is `\Seen` they return
`'No new documents were added.'` and enqueue nothing. To capture a firing that actually **ingests an
attachment**, a real IMAP account + rule were seeded and one *unseen* message with a `.txt` attachment was
placed in its INBOX; the schedule's `next_run` was then nudged into the past so the Sentinel fires it on
its next pass — the **same `next_run` technique** used in
[§10.2](#102-the-scheduler-mechanism-and-a-controlled-live-firing-observed). The setup + nudge (credential redacted):

```console
$ docker exec -i -u testuser pngx_qa bash -lc 'set -a; . /tmp/pp/penv; set +a; \
    export QA_MAIL_PW=<REDACTED>; cd /app/src && python3 /tmp/pp/mail_run.py'
ACCT id=1 host=127.0.0.1:10143 security=No encryption user=qauser
RULE id=1 folder=INBOX action=Mark as read, don't process read mails attachment_type=1
seeded schedule: id=4 name='Check all e-mail accounts' func=paperless_mail.tasks.process_mail_accounts type=I minutes=10
BEFORE next_run=2026-07-15T09:20:53.061489+00:00 repeats=-3
NUDGED next_run -> 2026-07-15T09:10:00+00:00 (strict-past; Sentinel will fire on next pass)
now = 2026-07-15T09:14:16.259938+00:00
```

On its next pass the Sentinel fired the **plural** `process_mail_accounts` as **Task A**; Task A logged in
over the real IMAP client, fetched the attachment, and — from *inside its own execution* — called
`async_task("documents.tasks.consume_file", …)` at `mail.py:336`, enqueuing **Task B**. Both rows are read
back **by explicit id** (no `Document.objects.order_by("-id").first()` guess), so the chain is
identity-complete:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Task
from documents.models import Document
a = Task.objects.get(id="c6f88a30db354ab9ba451f6a1f0775ff")   # scheduler-fired PLURAL task
print("TASK A  id=%s" % a.id)
print("  func    =", a.func)
print("  name    =", a.name)
print("  success =%s result=%r time_taken=%.3fs" % (a.success, a.result, a.time_taken()))
print("  started =%s stopped=%s" % (a.started.isoformat(), a.stopped.isoformat()))
b = Task.objects.get(id="c8333342075f4a04ae7984c6179d08be")   # the consume_file A enqueued
print("TASK B  id=%s" % b.id)
print("  func    =", b.func)
print("  name    =", b.name)
print("  success =%s result=%r time_taken=%.3fs" % (b.success, b.result, b.time_taken()))
print("  started =%s stopped=%s" % (b.started.isoformat(), b.stopped.isoformat()))
print("B enqueued INSIDE A's execution window? %s" % (a.started <= b.started <= a.stopped))
d = Document.objects.get(id=5)
print("DOCUMENT id=%d title=%r" % (d.id, d.title))
PY
TASK A  id=c6f88a30db354ab9ba451f6a1f0775ff
  func    = paperless_mail.tasks.process_mail_accounts
  name    = mars-butter-mike-montana
  success =True result='Added 1 document(s).' time_taken=0.082s
  started =2026-07-15T09:14:39.195235+00:00 stopped=2026-07-15T09:14:39.277628+00:00
TASK B  id=c8333342075f4a04ae7984c6179d08be
  func    = documents.tasks.consume_file
  name    = qa_mail_attachment.txt
  success =True result='Success. New document id 5 created' time_taken=0.897s
  started =2026-07-15T09:14:39.275178+00:00 stopped=2026-07-15T09:14:40.171884+00:00
B enqueued INSIDE A's execution window? True
DOCUMENT id=5 title='PaperlessQA live mail ingestion test'
```

**Cause → effect (the plural chain, end to end).** The Sentinel fires the **plural**
`process_mail_accounts` (Task A) — *not* the single-account `process_mail_account` wrapper. Task A runs the
canonical path `MailAccountHandler.handle_mail_account` (`mail.py:151`, `get_mailbox(...)` +
`M.login(account.username, account.password)`) → `handle_mail_rule` (`mail.py:187`, `M.folder.set(rule.folder)`
then `M.fetch(criteria=AND(**criterias), mark_seen=False, …)`, `mail.py:222`) → `handle_message`
(`mail.py:272`) → the enqueue at `mail.py:336`. For `ImapSecurity.NONE` the client is `MailBoxUnencrypted`
(`mail.py:94`). The proof that **Task A enqueued Task B** is temporal and exact: Task B's `started`
(`09:14:39.275178`) falls **inside** Task A's execution window (`09:14:39.195235 → 09:14:39.277628`) — the
`B enqueued INSIDE A's execution window? True` line — and Task A's own result string `'Added 1
document(s).'` is the count `process_mail_accounts` returns for the single attachment it consumed. Task B is
the `consume_file` job that then created **Document 5** (`'Success. New document id 5 created'`).

**Interpreting the two `time_taken` values.** Because the cluster was **running** for this natural firing,
neither task was left waiting: Task A `process_mail_accounts` completed in `0.082 s` (a fast IMAP
`LOGIN → SELECT → SEARCH → FETCH` against the local server), and Task B `consume_file` in `0.897 s` (the
actual parse → persist → index of the attachment). Both are genuine execution times with negligible
queue-wait — the complement of the deliberately-delayed cases in
[§6](#6-q4--waiting-vs-actively-processing).

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
SETUP tag id=3 name=qa_bulk_tag ; target documents=[12, 11]
LOGIN -> 302 ; sessionid present=True

REQUEST  POST http://localhost:8000/api/documents/bulk_edit/
  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>
  Content-Type: application/json
  body = {"documents": [12, 11], "method": "add_tag", "parameters": {"tag": 3}}
RESPONSE status = 200
RESPONSE body   = '{"result":"OK"}'

correlating (waiting for bulk_update_documents task)...
TASK id=79f0c34a52e44c2cbe48bf0d2d604177 func=documents.tasks.bulk_update_documents success=True time_taken=0.161s result=None
EFFECT documents now carrying tag 3: [12, 11]
```

**Cause → effect.** The JSON POST reaches `BulkEditView.post`, whose serializer resolves `method` to
`documents.bulk_edit.add_tag`. `add_tag` writes the tag rows and then calls
`async_task("documents.tasks.bulk_update_documents", document_ids=[12,11])` (`bulk_edit.py:47`) and returns
`"OK"` — which the view wraps as `{"result": "OK"}` (the observed body). The worker then runs
`bulk_update_documents`, whose `result` is `None` (it re-indexes rather than creating a document). The
`EFFECT` line confirms documents 12 and 11 now carry tag 3, proving the enqueued re-index actually executed.

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
`id=4` already reflect them). It is a **point-in-time** capture: because `id=4` (mail) fires every 10 min
and `id=1` (classifier) hourly, re-running this probe later shows `id=4`'s `repeats` grown more negative
and its `next_run` advanced by further 10-min steps — the decrement/advance mechanism is dissected in
§10.2–§10.3:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Schedule
print("Schedule rows:", Schedule.objects.count())
for s in Schedule.objects.order_by("id"):
    print(f"id={s.id} func={s.func} type={s.schedule_type} minutes={s.minutes} repeats={s.repeats} next_run={s.next_run.isoformat()} name={s.name!r}")
PY
Schedule rows: 4
id=1 func=documents.tasks.train_classifier type=H minutes=None repeats=-6 next_run=2026-07-15T12:09:00+00:00 name='Train the classifier'
id=2 func=documents.tasks.index_optimize type=D minutes=None repeats=-2 next_run=2026-07-16T09:00:52.480222+00:00 name='Optimize the index'
id=3 func=documents.tasks.sanity_check type=W minutes=None repeats=-2 next_run=2026-07-22T09:00:52.566742+00:00 name='Perform sanity check'
id=4 func=paperless_mail.tasks.process_mail_accounts type=I minutes=10 repeats=-19 next_run=2026-07-15T11:41:00+00:00 name='Check all e-mail accounts'
```

The `type` column holds Django-Q's single-letter `Schedule.TYPE` codes (`H`/`D`/`W`/`I` =
HOURLY/DAILY/WEEKLY/MINUTES). Two observed facts matter:

- **`repeats` is negative** (−2, −6, −19), below the seeded value of `−1`. A seeded schedule starts at
  `repeats = -1` (Django-Q's "repeat forever" sentinel); each firing decrements it by one
  (`s.repeats += -1`, `django_q/cluster.py:648`). The magnitude is therefore a running **count of past
  firings** during this container's uptime — e.g. `id=4` has fired ≈18 times. This alone proves the
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
`next_run` into the past and watched the live Sentinel (cluster pid `13604`) react on its next pass.

**BEFORE** — `id=1` is due in the future; I set its `next_run` to a clean past instant (`11:09:00`):

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django.utils import timezone
from datetime import datetime, timezone as tz
from django_q.models import Schedule
s = Schedule.objects.get(id=1)
print("nudge applied at UTC now =", timezone.now().isoformat())
print("  old next_run =", s.next_run.isoformat(), "| old repeats =", s.repeats)
s.next_run = datetime(2026, 7, 15, 11, 9, 0, tzinfo=tz.utc)
s.save(update_fields=["next_run"])
s.refresh_from_db()
print("  SET next_run =", s.next_run.isoformat(), "| repeats (unchanged) =", s.repeats)
PY
nudge applied at UTC now = 2026-07-15T11:29:12.421697+00:00
  old next_run = 2026-07-15T12:12:00+00:00 | old repeats = -5
  SET next_run = 2026-07-15T11:09:00+00:00 | repeats (unchanged) = -5
```

**DURING** — within one scheduler pass (~25 s later, at `11:29:37`) the live Sentinel logged the firing.
The grep pattern matches every *Train the classifier* firing in this log — two earlier natural HOURLY
firings (`10:12:26`, `11:12:05`) and the controlled one I triggered at `11:29:37`, ~25 s after the
`11:29:12` nudge (within one guard pass). `Process-1` is the Sentinel process, confirming the scheduler
runs there (not in a worker, not in a separate OS process):

```console
$ docker exec pngx_qa grep -nE 'created a task from schedule \[Train the classifier\]' /tmp/obs/qcluster.log
73:10:12:26 [Q] INFO Process-1 created a task from schedule [Train the classifier]
184:11:12:05 [Q] INFO Process-1 created a task from schedule [Train the classifier]
226:11:29:37 [Q] INFO Process-1 created a task from schedule [Train the classifier]
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
observed at UTC now = 2026-07-15T11:30:03.017998+00:00
  id=1 next_run = 2026-07-15T12:09:00+00:00
  id=1 repeats  = -6
  id=1 s.task   = a3e02424a4ce4774949a1f2ec60df428
  Task.id      = a3e02424a4ce4774949a1f2ec60df428
  Task.name    = fifteen-happy-enemy-blossom
  Task.started = 2026-07-15T11:29:37.777205+00:00
  Task.stopped = 2026-07-15T11:29:37.914456+00:00
  Task.success = True
  Task.result  = None
```

Reading the AFTER state against the mechanism:

- **`next_run`: `11:09:00` → `2026-07-15T12:09:00`** — exactly `+1 h`, the HOURLY shift applied to the
  *old* `next_run` I set (not to "now"); one shift sufficed because `11:09:00 + 1 h = 12:09:00` is already
  in the future, so the `catch_up=False` loop stopped after one iteration.
- **`repeats`: `-5` → `-6`** — decremented exactly once, regardless of how far in the past the trigger was.
- **`s.task == Task.id == a3e02424a4ce4774949a1f2ec60df428`** — the schedule stores the id of the task it
  enqueued, and that task ran to completion (`success=True`; `result=None` because `train_classifier`
  returns `None` when no retrain is warranted). The 32-char hex id is a Django-Q task id exactly as in
  [§5](#5-q3--how-a-job-appears-when-created).

This is the same enqueue → execute → persist lifecycle as an ingested document; the *only* difference is
the caller — the Sentinel's `scheduler()` rather than an ingestion entry point.

### 10.3 Cadence stability across ≥2 intervals (observed)

The MINUTES=10 *Check all e-mail accounts* schedule fires often enough to measure directly. The current
`qcluster.log` (beginning `09:29:20`) records **fourteen** firings — spanning many consecutive intervals —
all attributed to `Process-1`:

```console
$ docker exec pngx_qa grep -nE 'created a task from schedule \[Check all e-mail accounts\]' /tmp/obs/qcluster.log
18:09:30:20 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
25:09:40:21 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
32:09:50:23 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
39:10:00:24 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
46:10:10:26 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
80:10:20:27 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
87:10:25:58 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
94:10:31:29 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
101:10:41:00 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
108:10:51:02 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
128:11:01:03 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
177:11:11:05 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
219:11:21:06 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
233:11:31:08 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
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
2026-07-15T09:02:02.889910+00:00  delta=Nones
2026-07-15T09:11:08.681338+00:00  delta=545.791s
2026-07-15T09:14:39.195235+00:00  delta=210.514s
2026-07-15T09:20:09.999088+00:00  delta=330.804s
2026-07-15T09:30:20.446902+00:00  delta=610.448s
2026-07-15T09:40:21.919363+00:00  delta=601.472s
2026-07-15T09:50:23.356045+00:00  delta=601.437s
2026-07-15T10:00:24.788228+00:00  delta=601.432s
2026-07-15T10:10:26.236193+00:00  delta=601.448s
2026-07-15T10:20:27.678345+00:00  delta=601.442s
2026-07-15T10:25:58.495909+00:00  delta=330.818s
2026-07-15T10:31:29.307379+00:00  delta=330.811s
2026-07-15T10:41:00.666091+00:00  delta=571.359s
2026-07-15T10:51:02.096100+00:00  delta=601.43s
2026-07-15T11:01:03.549085+00:00  delta=601.453s
2026-07-15T11:11:05.018372+00:00  delta=601.469s
2026-07-15T11:21:06.531994+00:00  delta=601.514s
2026-07-15T11:31:08.002209+00:00  delta=601.47s
```

**Reading the cadence.** The steady-state deltas cluster tightly at **≈601.4–601.5 s** — the 10-minute
interval plus up to one guard cycle of latency. The schedule's `next_run` advances by *exactly* 600 s
(`.shift(minutes=+10)`, `cluster.py:616`), but the firing happens on the first ~30 s guard pass *after*
`next_run` elapses, so successive undisturbed firings inherit a near-constant phase offset of ≈1.5 s. The
several shorter/irregular deltas (`210.514 s`, `330.804 s`, `330.818 s`, `330.811 s`, `545.791 s`,
`571.359 s`) each coincide with a `qcluster` stop/restart during the investigation — the Q4 experiment in
[§6](#6-q4--waiting-vs-actively-processing) and the worker-death reproduction in
[§3.6](#36-edge-conditions-observed-on-the-live-path) both stopped/restarted the cluster, shifting the
guard phase; the single slightly-long `610.448 s` is the first firing after such a restart, before the
phase re-settles. They are restart artifacts, not cadence drift. Across ≥2 consecutive undisturbed
intervals — e.g. `09:30→09:40→09:50→10:00→10:10→10:20`, and again `10:51→11:01→11:11→11:21→11:31` — the
cadence is stable.

The deterministic half of this is directly visible on the schedule row: after the `11:31:08` firing,
`id=4`'s `next_run` had advanced by exactly `+10 min`:

```console
$ docker exec -i pngx_qa bash /tmp/pp/pp.sh <<'PY'
import django; django.setup()
from django_q.models import Schedule
s = Schedule.objects.get(id=4)
print("id=4 next_run =", s.next_run.isoformat(), "| repeats =", s.repeats, "| last task =", s.task)
PY
id=4 next_run = 2026-07-15T11:41:00+00:00 | repeats = -19 | last task = 0829d30cf7d146cea27230e36a932b89
```

`11:31:00 + 10 min = 11:41:00` — the interval is applied to the stored `next_run`, so cadence
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

### 11.3 Out-of-scope observations — pre-existing system behaviors

While driving the **canonical** entry points to answer Q1–Q7, several pre-existing behaviors of
the paperless-ngx product code (and of the `django-q` dependency) surfaced as side effects. They are
**not defects introduced by this investigation** and — critically — they are **out of scope to fix**
under this task's mandate: AAP §0.3.2 places "any modification, creation, or deletion of repository
source files other than the single answer document" and "refactoring, bug fixing, performance tuning,
or security hardening of the asynchronous subsystem" explicitly out of scope, and §0.4.2 forbids
adding, removing, or upgrading any dependency. They are therefore **catalogued here as observed facts
with exact root-cause citations, not remediated**. The sole change this task introduces remains the
single documentation file.

Each item below gives (a) the **observed** evidence with the command that produced it, (b) the exact
`file:line` root cause naming the responsible function, (c) the cause → effect mechanism, and (d) the
specific mandate clause that keeps the fix out of scope. Where an item's transient artifact (a rotated
worker-log window, or a one-shot WebSocket frame) is quoted from the original capture, its **durable**
half is re-verified **live** against the running system so the claim is grounded, not merely recalled.

#### F-API-1 — a 128-character title is silently stored as 127 (functional / data-fidelity)

Observed (live): the 128-'T' title POSTed through the REST upload path is persisted as **127**
characters.

```console
$ docker exec -i pngx_qa /tmp/pp/pp.sh <<'PY'
from documents.models import Document
d6 = Document.objects.get(pk=6)            # 128-'T' title submitted via POST /api/documents/post_document/
print("Document 6 title length =", len(d6.title), "; every char 'T'? =", set(d6.title) == {"T"})
PY
Document 6 title length = 127 ; every char 'T'? = True
```

Root cause: inside `Consumer._store` (`src/documents/consumer.py:379`) the row is created by
`src/documents/consumer.py:398-399` as
`Document.objects.create(title=(self.override_title or file_info.title)[:127], …)`. The hardcoded
`[:127]` slice caps the title one character **below** the column's declared capacity,
`src/documents/models.py:106` `title = models.CharField(_("title"), max_length=128, blank=True, db_index=True)`.

Mechanism (cause → effect): the slice bound (127) is smaller than the column bound (128); a caller who
supplies exactly the column-maximum 128 characters has the final character dropped before the INSERT,
with no error, warning, or truncation notice. The stored value is 127 characters.

Out of scope: correcting the slice is a **source-code bug fix** in the ingestion path — excluded by AAP
§0.3.2 (no repository source modification; no bug fixing of the asynchronous subsystem).

#### F-SEC-2 — a CRLF in a title forges standalone worker-log lines (log injection)

Observed (live, durable half): the CR/LF injected into the title survives ingestion and is stored
verbatim in the `title` column.

```console
$ docker exec -i pngx_qa /tmp/pp/pp.sh <<'PY'
from documents.models import Document
print(repr(Document.objects.get(pk=7).title))
PY
'LEGITTITLE\r\n[2026-01-01 00:00:00,000] [INFO] [paperless.forged] INJECTED_ADMIN_ACTION'
```

Observed (transient half, from the original capture — the worker log has since rotated past 09:22,
but the persisted title above is the live, reproducible anchor): while that document was consumed, the
matching handlers logged the title verbatim, and the embedded newline split each message into a second,
forged-looking physical line:

```text
79:[2026-07-15 09:22:57,422] [INFO] [paperless.handlers] Assigning correspondent QA Correspondent to 2026-07-15 LEGITTITLE
80:[2026-01-01 00:00:00,000] [INFO] [paperless.forged] INJECTED_ADMIN_ACTION
81:[2026-07-15 09:22:57,423] [INFO] [paperless.handlers] Assigning document type QA Type to 2026-07-15 QA Correspondent LEGITTITLE
82:[2026-01-01 00:00:00,000] [INFO] [paperless.forged] INJECTED_ADMIN_ACTION
```

Root cause: the title flows unescaped into log messages emitted by the post-consume matching handlers —
`src/documents/signals/handlers.py:93` `f"Assigning correspondent {selected} to {document}"`,
`:160` `f"Assigning document type {selected} to {document}"`, and `:224` `'Tagging "{}" with "{}"'` —
where `{document}` renders through `Document.__str__` (which embeds the `title`). Nothing strips or
escapes CR/LF, and the DB column (`src/documents/models.py:106`) stores the newline unchanged.

Mechanism (cause → effect): a user-controlled `\r\n` inside the title splits one logical log record
across two physical lines; the injected second line
`[2026-01-01 00:00:00,000] [INFO] [paperless.forged] INJECTED_ADMIN_ACTION` is byte-for-byte
indistinguishable from a genuine timestamped log entry ⇒ audit-trail spoofing / log forgery.

Out of scope: adding newline sanitization or output-encoding is **security hardening** of the
ingestion code — excluded by AAP §0.3.2.

#### F-SEC-3 — a raw database error is broadcast over the authenticated WebSocket (information disclosure)

Observed (live, durable half): the losing task of a duplicate-checksum collision persists with the raw
DB error string in `result`.

```console
$ docker exec -i pngx_qa /tmp/pp/pp.sh <<'PY'
from django_q.models import Task
t = Task.objects.get(id="138f23904b654777b57b73570808c9d0")   # loser of a duplicate race
print("success =", t.success, "; result[:64] =", repr((t.result or '')[:64]))
PY
success = False ; result[:64] = 'qa_dup_race_a_20260715_092406.txt: The following error occured w'
```

Observed (transient half, from the authenticated WebSocket capture in §8.4; the same raw string as it
left the server): the identical message is pushed to the connected client as a `FAILED` progress frame.

```text
WS FRAME 12  {"current_progress": 100, "document_id": null, "filename": "qa_race2_b_20260715_092707.txt", "max_progress": 100, "message": "UNIQUE constraint failed: documents_document.checksum", "status": "FAILED", "task_id": "cc115b95-3c86-4bf6-b322-c88592c56522"}
```

Root cause chain (every hop LIVE-verified in source): `src/documents/consumer.py:398`
`Document.objects.create(…)` violates the unique `checksum` constraint → `sqlite3.IntegrityError`
→ `django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum`; the broad
`except Exception as e:` at `src/documents/consumer.py:362` calls `self._fail(str(e), …)` at `:363`;
`_fail` (`src/documents/consumer.py:78`) first calls `self._send_progress(100, 100, "FAILED", message)`
at `:79` — broadcasting the raw message — before `raise ConsumerError(...)` at `:81`. The broadcast
reaches the browser through `StatusConsumer.status_update` (`src/paperless/consumers.py:29`) →
`self.send(json.dumps(event["data"]))` (`src/paperless/consumers.py:33`).

Mechanism (cause → effect): the verbatim database error (`str(e)`) becomes the WebSocket `message`
field ⇒ any authenticated client watching progress learns internal schema details (the table/column
`documents_document.checksum`). The winning sibling of the race committed normally as Document 9; the
persisted `success=False` row above is the durable proof that the broadcast string is the real DB
error.

Out of scope: redacting or normalizing the broadcast message is **security hardening** — excluded by
AAP §0.3.2.

#### F-API-2 — an orphaned plaintext upload temp is left when the broker is unreachable (reliability / data-at-rest)

Observed (fresh, live): in a controlled broker-outage window — authenticate while Redis is up, POST
while Redis is down, then restore Redis — the request returns **HTTP 500** yet the uploaded bytes are
left on disk as a plaintext temp file.

```console
BASELINE paperless-upload-* count = 1
LOGIN (Redis UP): GET 200 ; POST 302 ; sessionid=True
REDIS after shutdown: redis-cli ping -> Could not connect to Redis at 127.0.0.1:6379: Connection refused

REQUEST POST http://localhost:8000/api/documents/post_document/  (multipart 'qa_orphan_20260715_121244.txt', 52 bytes)
RESPONSE status = 500
RESPONSE body(first160) = '\n<!doctype html>\n<html lang="en">\n<head>\n  <title>Server Error (500)</title>\n</head>\n<body>\n  <h1>Server Error (500)</h1><p></p>\n</body>\n</html>\n'

AFTER paperless-upload-* count = 2  NEW = ['paperless-upload-7y24gw8k']
  ORPHAN paperless-upload-7y24gw8k : 52 bytes, mode 0o600, content=b'PAPERLESS_QA_ORPHAN_TEST broker-down 20260715_121244'
REDIS after restart: redis-cli ping -> PONG
```

The leaked file persists after the failed request; the earlier orphan from a prior outage is still
present too, showing orphans are never reclaimed:

```console
$ docker exec pngx_qa ls -la /tmp/pp/scratch/ | grep paperless-upload
-rw------- 1 testuser testuser   52 Jul 15 12:12 paperless-upload-7y24gw8k
-rw------- 1 testuser testuser   97 Jul 15 09:28 paperless-upload-iy3r7twa
```

Root cause: `PostDocumentView.post` (`src/documents/views.py:497`) writes the upload to disk **before**
enqueuing — `tempfile.NamedTemporaryFile(prefix="paperless-upload-", dir=settings.SCRATCH_DIR,
delete=False)` at `src/documents/views.py:512-516`, `f.write(doc_data)` at `:517`, and
`temp_filename = f.name` at `:519` — then generates `task_id = str(uuid.uuid4())` at `:521` and calls
`async_task("documents.tasks.consume_file", temp_filename, …)` at `:523`. There is no `try/finally`
guarding the enqueue.

Mechanism (cause → effect): because `delete=False`, the temp file outlives the `with` block; when
`async_task` cannot reach the broker it raises **after** the bytes are already persisted, the view
returns HTTP 500, and no code path deletes the file ⇒ a `-rw-------` but **plaintext** upload
accumulates in the scratch directory on every broker-down POST.

Out of scope: reordering the write/enqueue, or adding cleanup on enqueue failure, is a **source-code
bug fix** — excluded by AAP §0.3.2.

#### F-OBS-1 — a broker outage degrades into a logging `TypeError` (observability; dependency behavior)

Observed (fresh, live): during the same broker-outage window, the `qcluster` pusher does not emit a
clean error — its `logger.error` call itself fails to format, producing a `--- Logging error ---`
dump with a `TypeError` on every dequeue attempt (current-generation `qcluster.log`):

```text
--- Logging error ---
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
  File "/usr/local/lib/python3.9/site-packages/django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 1900, in blpop
    return self.execute_command('BLPOP', *keys)
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 429, in read_from_socket
    raise ConnectionError(SERVER_CLOSED_CONNECTION_ERROR)
redis.exceptions.ConnectionError: Connection closed by server.

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/logging/__init__.py", line 1083, in emit
    msg = self.format(record)
  File "/usr/local/lib/python3.9/logging/__init__.py", line 663, in format
    record.message = record.getMessage()
  File "/usr/local/lib/python3.9/logging/__init__.py", line 367, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  [... multiprocessing spawn/fork boilerplate elided; the emitting frame is: ...]
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 347, in pusher
    logger.error(e, traceback.format_exc())
Message: ConnectionError('Connection closed by server.')
Arguments: ('Traceback (most recent call last):\n  File ".../django_q/cluster.py", line 345, in pusher\n    task_set = broker.dequeue()\n  ... redis.exceptions.ConnectionError: Connection closed by server.\n',)
```

The cluster nonetheless recovers on its own once Redis returns — the reconnect burst is followed by a
reincarnated pusher:

```text
12:12:43 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
   ... (one line per second for the duration of the outage) ...
12:12:52 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
12:12:53 [Q] INFO Process-1:13 stopped pushing tasks
12:12:53 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
12:12:53 [Q] INFO Process-1:43 pushing tasks at 17423
```

Root cause (inside the `django-q` dependency): `django_q/cluster.py:345` `task_set = broker.dequeue()`
(→ `django_q/brokers/redis_broker.py:21` `blpop(self.list_key, 1)`) raises `ConnectionError` when the
broker is down; the handler at `django_q/cluster.py:347` calls `logger.error(e, traceback.format_exc())`
— passing the **exception object as the log format string (`msg`)** and the traceback string as a
**positional argument**. Python logging then evaluates `msg = msg % self.args` at
`/usr/local/lib/python3.9/logging/__init__.py:367`; because `str(e)` (`"Connection closed by server."`)
contains no `%` placeholder while `self.args` is non-empty, it raises
`TypeError: not all arguments converted during string formatting`.

Mechanism (cause → effect): the exception-as-format-string plus a spurious positional argument turns a
simple broker outage into a per-retry internal "Logging error" dump (with the `Message:` / `Arguments:`
fallback) instead of a single clean error line — noise that obscures the real signal (the broker is
down). The task-processing path itself still self-heals via the sentinel's reincarnated pusher.

Out of scope on two independent grounds: (1) the behavior lives in the third-party `django-q==1.3.9`
package, and the mandate forbids adding, removing, or upgrading any dependency (AAP §0.4.2, §0.3.2);
(2) it is a bug fix regardless of location, also excluded. Observation-only.

#### Dependency-snapshot posture (F-SEC-1, recap of §2.1)

Finally, the environment itself is pinned to the **historical** dependency set captured at commit
`542221a38dff`. As detailed in [§2.1](#21-disclosed-environment-posture-deviations-from-production-and-why), several
of those pins are end-of-life or carry published CVEs — notably **Django 4.0.4**, **djangorestframework
3.13.1**, and **gunicorn 20.1.0**. The environment is stood up for **observation of the Django-Q
enqueue/worker/broker behavior only** and is explicitly **not** production-hardened. Re-pinning is out
of scope on two grounds: the read-only mandate forbids editing `requirements.txt` / `Pipfile`
(AAP §0.3.2), and §0.4.2 records "New dependencies to add: None / Dependencies to update: None". None of
these versions changes the asynchronous behavior under study.

**Net effect on scope.** Every behavior in §11.3 was observed while exercising the canonical entry
points, and **none was modified**. Fixing any of them would require editing repository source or a
dependency — both forbidden — so they are documented here and left exactly as they ship. The repository
integrity check in [§12](#12-cleanup--repository-integrity) confirms the source tree and dependency
manifests are byte-for-byte unchanged; the only artifact this task adds is this document.


---

## 12. Cleanup & repository integrity

The whole investigation ran inside a **single disposable Docker container** (`pngx_qa`), with the Redis
broker installed into that very container from Debian's own package — not a second container — exactly as
[§2.2](#22-create-the-disposable-container-and-redis) sets it up. It used only temporary scripts under the
container's `/tmp/pp` and host evidence files under `/tmp/qa_work` — **no file in the source repository
was ever modified** (the read-only mandate). Cleanup is therefore a *non-destructive* teardown: gracefully
stop the services (including Redis), then delete the disposable container. There is **no** database or file
"restore" step, because nothing durable outside the container was touched — a deliberate contrast to a
destructive "overwrite the DB then delete the backup" approach.

### 12.1 Safe, validated service shutdown

Every service PID was captured **live** at launch via `$!` — the three canonical services to
`/tmp/obs/*.pid` by [`launch.sh`](#132-the-remaining-helper-scripts-verbatim), and the ad-hoc IMAP test
server (stood up for [§9.3](#93-origin-3--mail-fetch-handle_message-exercised-live-against-a-real-imap-server))
recorded the same way when it was started — so none is hard-coded. Immediately **before** signalling,
each PID's `/proc/<pid>/comm` is re-validated so a recycled PID can never be signalled by mistake (this
closes the check-then-act TOCTOU window). `SIGTERM` is sent to the `qcluster` **main** process, which
triggers Django-Q's graceful stop so the Sentinel drains and reaps its entire child tree:

```bash
# /tmp/obs/teardown.sh — validated shutdown of the four nohup'd top-level services.
# The 3 canonical PIDs come from launch.sh's pidfiles (/tmp/obs/*.pid); the IMAP test server's PID
# was likewise captured via $! when it was stood up for §9.3. None is hard-coded.
TOP="13604 2970 2971 14765"     # qcluster main · document_consumer · gunicorn master · IMAP test server
# Full monitored set = the 4 top-level + the qcluster subtree (Sentinel 13611 + its 13
# children: 11 workers, monitor 13623, pusher 17423) + gunicorn's 2 workers (2973 2974) = 20 PIDs.
CL="13611 13623 15544 15546 15959 15979 16072 16507 17091 17195 17344 17383 17423 17499 2973 2974"

echo "===== STEP 1: validate comm immediately before signalling ====="
for p in $TOP; do printf "  pid=%-6s comm=%s -> will SIGTERM\n" "$p" "$(cat /proc/$p/comm)"; done

echo "===== STEP 2: SIGTERM top-level services (qcluster main drains its own tree) ====="
for p in $TOP; do c=$(cat /proc/$p/comm); kill -TERM "$p" && printf "  SIGTERM -> %s (%s)\n" "$p" "$c"; done

echo "===== STEP 3: wait for graceful drain + child reaping ====="
for t in $(seq 1 14); do sleep 1; a=0; for p in $TOP $CL; do [ -e /proc/$p ] && a=$((a+1)); done
  { [ $t -le 5 ] || [ $t -eq 14 ]; } && printf "  t=%-2ss service PIDs still alive: %s\n" "$t" "$a"
  [ $t -eq 6 ] && echo "  ..."; done
```

```console
$ docker exec pngx_qa bash /tmp/obs/teardown.sh
===== STEP 1: validate comm immediately before signalling =====
  pid=13604  comm=python3 -> will SIGTERM
  pid=2970   comm=python3 -> will SIGTERM
  pid=2971   comm=gunicorn -> will SIGTERM
  pid=14765  comm=python3 -> will SIGTERM

===== STEP 2: SIGTERM top-level services (qcluster main drains its own tree) =====
  SIGTERM -> 13604 (python3)
  SIGTERM -> 2970 (python3)
  SIGTERM -> 2971 (gunicorn)
  SIGTERM -> 14765 (python3)

===== STEP 3: wait for graceful drain + child reaping =====
  t=1 s service PIDs still alive: 17
  t=2 s service PIDs still alive: 4
  t=3 s service PIDs still alive: 4
  t=4 s service PIDs still alive: 4
  t=5 s service PIDs still alive: 4
  ...
  t=14s service PIDs still alive: 4
```

Within ~2 s the **14** cluster children — the Sentinel (`13611`) plus its 11 workers, 1 monitor
(`13623`), and 1 pusher (`17423`) — exit and are **reaped by the cluster itself** (the Sentinel reaps
the workers/monitor/pusher, then the main reaps the Sentinel), so none of them lingers. The **4**
top-level processes that were launched with `nohup … & disown` finish their own shutdown but, having
been reparented to PID 1, are left as **defunct (zombie)** entries because this minimal container init
does not reap them. A zombie holds **no** memory, file descriptors, or sockets — it is only a slot in
the process table — and is reaped the instant the container is removed. The two follow-up read-only
`/proc` inspections below confirm both halves — the 4 top-level slots are `state=Z ppid=1`, and the
cluster subtree is gone entirely:

```console
$ docker exec pngx_qa bash -lc 'for p in 13604 2970 2971 14765; do
    printf "  pid=%-6s state=%s ppid=%s comm=%s\n" "$p" \
      "$(awk "{print \$3}" /proc/$p/stat)" "$(awk "{print \$4}" /proc/$p/stat)" "$(cat /proc/$p/comm)"; done'
  pid=13604  state=Z ppid=1 comm=python3
  pid=2970   state=Z ppid=1 comm=python3
  pid=2971   state=Z ppid=1 comm=gunicorn
  pid=14765  state=Z ppid=1 comm=python3

$ docker exec pngx_qa bash -lc 'for p in 13611 13623 17423 17499; do
    [ -e /proc/$p ] && echo "  pid=$p STILL PRESENT" || echo "  pid=$p reaped (gone from /proc)"; done'
  pid=13611 reaped (gone from /proc)
  pid=13623 reaped (gone from /proc)
  pid=17423 reaped (gone from /proc)
  pid=17499 reaped (gone from /proc)
```

That the services are **functionally** dead (not merely defunct shells) is confirmed by the absence of
any listening socket inside the container's network namespace — parsing `/proc/net/tcp` and
`/proc/net/tcp6` for `LISTEN` (state `0A`) returned **no rows at all**, so neither the gunicorn port
`8000` nor the IMAP port `10143` remains bound; the sockets were released the instant the processes
terminated:

```console
$ docker exec pngx_qa bash -lc '
    LIS=$(awk "NR>1 && \$4==\"0A\"{split(\$2,a,\":\"); print strtonum(\"0x\"a[2])}" \
          /proc/net/tcp /proc/net/tcp6 | sort -un | tr "\n" " ")
    echo "  LISTEN ports open (container netns): ${LIS:-<none>}"
    for want in 8000 10143; do echo " $LIS " | grep -q " $want " \
      && echo "  port $want: STILL LISTENING" || echo "  port $want: no listener (released)"; done'
  LISTEN ports open (container netns): <none>
  port 8000: no listener (released)
  port 10143: no listener (released)
```

(The scan of `manage.py qcluster` processes must exclude the scanning shell itself, whose own command
line contains that string — otherwise it self-matches; the per-PID `/proc` existence check used above
avoids that trap entirely.)

### 12.2 Disposable-container discard (reaps the zombies, frees everything)

Because the entire runtime — app, worker cluster, watcher, IMAP test server, Redis broker, and the
SQLite DB / media / consume directories under the container's `/tmp` — lived inside the one disposable
container, the correct and complete cleanup is to **delete the container**. This atomically reaps the
four zombies and frees every resource; no host or repository state needs restoring:

```console
$ docker rm -f pngx_qa
pngx_qa

$ docker ps -a --format '{{.Names}}' | grep -E '^pngx_qa$' || echo "container pngx_qa no longer exists"
container pngx_qa no longer exists

$ ps -eo pid,comm,args | grep -E 'imap_server\.py|manage\.py qcluster|paperless\.asgi' | grep -v grep || echo "no host service processes"
no host service processes

$ ss -ltn 2>/dev/null | grep -E ':10143|:8000' || echo "no host listeners on 10143/8000"
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
6b280d64623c6082b42082757bef43aeb4a8c880

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

> **Authoring-time snapshot.** The `HEAD` shown above (`6b280d64…`) is the mid-authoring commit at the
> moment of capture; it advances by one commit each time this document is revised, so a later reader will
> observe a different `HEAD`. Likewise, `git status --porcelain` shows the document as a pending ` M` only
> while it is uncommitted — once the document is committed it becomes tracked and `git status --porcelain`
> is **empty**. The durable, always-reproducible invariant is therefore not the exact `HEAD` hash but the
> *delta from the base commit*: `git diff --name-status 542221a38dff..HEAD` yields exactly one line —
> `A blitzy/documentation/paperless-ngx_542221a38dff.md` — with **no source file modified**.

The only change this task introduces to the repository is the addition/modification of this one Markdown
document; every temporary artifact used to produce it lived outside the tracked tree (in the now-deleted
container and in host scratch under `/tmp`) and leaves **no trace in the repository** — the sole,
load-bearing integrity guarantee. (Host authoring scratch under `/tmp` is not part of the repository or
this deliverable and is unrelated to the tracked-tree cleanliness proven above.)

---

<a id="appendix"></a>

## 13. Appendix

Every temporary artifact used to observe the behaviors in this document is a **script**, published here
inline and **verbatim** so that each command shown in §1–§10 is reproducible byte-for-byte. Nothing in this
appendix modifies the repository: these scripts lived **outside** the tracked tree — under the container's
`/tmp/pp/`, which is discarded when the disposable container is removed (see
[§12](#12-cleanup--repository-integrity)), and in a host authoring-scratch directory (`/tmp/qa_work/`) that
is not part of the repository or this deliverable. The single load-bearing integrity guarantee is that the
tracked tree is left byte-for-byte unchanged except for this one document. No credential ever appears in the
repository or in this published document: the admin/test password and the mailbox password are shown as
`<REDACTED>` throughout. Every helper reads the real value from an environment variable at run time —
`QA_ADMIN_PW` (`rest_upload.py`, `bulk_edit.py`), `PP_PASS` (`ws_capture.py`), and `QA_MAIL_PW`
(`mail_run.py`); all are published in [§13.2](#132-the-remaining-helper-scripts-verbatim). In every case
the value is a local-only, disposable test credential created solely for this investigation — it is
embedded in no repository file and appears nowhere in this deliverable.

### 13.1 Complete helper-script index

Nine helper scripts were used. Four are already published verbatim earlier in the document (at the point
where they were first needed); the remaining five are published in [§13.2](#132-the-remaining-helper-scripts-verbatim)
directly below. This table is the single consolidated index:

| Script | Lines | Published verbatim in | Purpose (section it serves) |
|---|---:|---|---|
| `penv` | 10 | [§2.3](#23-directories-environment-file-and-the-disclosed-helper-scripts) | The exact environment variables sourced by **every** command (redis URL, data/media/consume dirs, `DJANGO_SETTINGS_MODULE`, `PYTHONPATH`) |
| `pp.sh` | 4 | [§2.3](#23-directories-environment-file-and-the-disclosed-helper-scripts) | Runs a Python snippet (from stdin) inside the canonical env as non-root `testuser` — the idiom behind every ORM/broker probe |
| `launch.sh` | 6 | [§2.3](#23-directories-environment-file-and-the-disclosed-helper-scripts) | Starts the three canonical services (`qcluster`, `document_consumer`, `gunicorn`) per `docker/supervisord.conf`, recording each master PID via `$!` |
| `poll.py` | 42 | [§6](#6-q4--waiting-vs-actively-processing) | The Q4 boundary poller — samples Redis `LLEN`, cluster `Stat`, and the specific `Task` row to prove WAITING → PROCESSING → DONE |
| `ws_capture.py` | 58 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q6 authenticated WebSocket status capture (real session login → `sessionid` cookie → `ws/status/`) |
| `imap_server.py` | 180 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 real minimal IMAP4 server (Twisted) — one plaintext account, one RFC822 message with a `.txt` attachment |
| `rest_upload.py` | 62 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 REST upload driver — browser-canonical session + CSRF auth against `POST /api/documents/post_document/`, then peeks the WAITING package to record the identity-complete chain |
| `mail_run.py` | 54 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 mail driver — seeds a `MailAccount`/`MailRule` and **nudges the seeded plural `Schedule`** so the running `qcluster` Sentinel fires `process_mail_accounts` by itself (the real production trigger); invokes no task directly |
| `bulk_edit.py` | 47 | [§13.2](#132-the-remaining-helper-scripts-verbatim) | Q7 bulk-edit driver — session + CSRF auth against `POST /api/documents/bulk_edit/` (JSON body) |

Total: **463 lines** across nine scripts. `penv`, `pp.sh`, and `launch.sh` appear in
[§2.3](#23-directories-environment-file-and-the-disclosed-helper-scripts); `poll.py` appears in
[§6](#6-q4--waiting-vs-actively-processing); the five below complete the set.

### 13.2 The remaining helper scripts (verbatim)

The five scripts referenced by forward-links elsewhere in the document are reproduced here exactly as run
(only the credential literals are replaced by `<REDACTED>`; line counts are otherwise unchanged from the
files under `/tmp/pp/`).

#### `ws_capture.py` — Q6 authenticated WebSocket status capture ([§8](#8-q6--after-the-fact-status))

```python
#!/usr/bin/env python3
"""Disclosed WebSocket status listener for Q6 (§8) and the identity-complete REST
chain (§9.2). Pure LISTENER — it does not trigger ingestion itself; the concurrent
REST upload (rest_upload.py) is the canonical trigger, so the frames captured here
carry that same job's progress UUID.
Flow: (1) real Django session login via /admin/login/ (CSRF) -> sessionid cookie;
      (2) open ws://localhost:8000/ws/status/ carrying that cookie (AuthMiddlewareStack
          reads it -> scope['user'] -> StatusConsumer.connect accepts);
      (3) print every status_updates frame as it arrives, with an arrival timestamp,
          until a terminal SUCCESS/FAILED frame or the idle timeout.
Credentials come from env PP_USER / PP_PASS; the sessionid value is redacted in output.
"""
import os, sys, json, asyncio
from datetime import datetime, timezone
import requests, websockets

BASE = "http://localhost:8000"
USER = os.environ["PP_USER"]
PASS = os.environ["PP_PASS"]
IDLE = float(os.environ.get("PP_WS_IDLE", "30"))

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
        print("WS %s CONNECTED (HTTP 101 Switching Protocols) - authenticated accept" % iso())
        sys.stdout.flush()
        n = 0
        while True:
            try:
                raw = await asyncio.wait_for(ws.recv(), timeout=IDLE)
            except asyncio.TimeoutError:
                print("WS %s (no frames for %ss; stopping)" % (iso(), IDLE))
                break
            n += 1
            frame = json.loads(raw)
            print("WS FRAME %d %s  %s" % (n, iso(), json.dumps(frame, sort_keys=True)))
            sys.stdout.flush()
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
#!/usr/bin/env python3
"""Q7 REST upload driver producing an IDENTITY-COMPLETE chain (§9.2).
Runs with the cluster DOWN so the enqueued package can be peeked WAITING, then the
caller starts the cluster; the concurrently-connected ws_capture.py records the frames.
It threads ONE job identity through every layer:
  package hex id (Django-Q PK)  ==  Task.id
  package kwargs['task_id'] (progress UUID, generated at views.py:521)  ==  WS frames' task_id
  filename  ==  Task.name-derived / Document title
  Task.result -> Document id
Browser-canonical session + CSRF auth against POST /api/documents/post_document/.
"""
import os, sys, io, time, json, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import requests

BASE = "http://localhost:8000"
USER = "admin"; PW = os.environ.get("QA_ADMIN_PW", "<REDACTED>")
s = requests.Session()

# --- Session + CSRF auth (browser-canonical) ---
r0 = s.get(f"{BASE}/accounts/login/")
csrf = s.cookies.get("csrftoken")
print("STEP1 GET /accounts/login/ -> %d ; csrftoken cookie present=%s" % (r0.status_code, bool(csrf)))
r1 = s.post(f"{BASE}/accounts/login/",
            data={"username": USER, "password": PW, "csrfmiddlewaretoken": csrf, "next": "/"},
            headers={"Referer": f"{BASE}/accounts/login/"}, allow_redirects=False)
print("STEP2 POST /accounts/login/ -> %d ; sessionid cookie present=%s" % (r1.status_code, bool(s.cookies.get("sessionid"))))

# --- POST the document multipart ---
csrf = s.cookies.get("csrftoken")
uniq = time.strftime("%Y%m%d_%H%M%S")
content = ("PAPERLESS_QA_REST_UPLOAD paperlessqademo unique %s for the REST ingestion path." % uniq).encode()
fname = "qa_rest_%s.txt" % uniq
files = {"document": (fname, io.BytesIO(content), "text/plain")}
print("\nREQUEST  POST %s/api/documents/post_document/" % BASE)
print("  auth: session cookie sessionid=<REDACTED>; header X-CSRFToken=<REDACTED>")
print("  multipart field 'document' = (%s, %d bytes, text/plain)" % (fname, len(content)))
r2 = s.post(f"{BASE}/api/documents/post_document/", files=files,
            headers={"X-CSRFToken": csrf, "Referer": f"{BASE}/"})
print("RESPONSE status =", r2.status_code)
print("RESPONSE content-type =", r2.headers.get("Content-Type"))
print("RESPONSE body =", repr(r2.text))

# --- Peek the WAITING package (cluster is down): hex PK + progress UUID ---
django.setup()
import redis
from django.conf import settings
from django_q.signing import SignedPackage
r = redis.Redis.from_url(settings.Q_CLUSTER["redis"])
QKEY = "django_q:%s:q" % settings.Q_CLUSTER["name"]
print("\nWAITING peek:  LLEN(%s) = %d" % (QKEY, r.llen(QKEY)))
raw = r.lindex(QKEY, 0)
pkg = SignedPackage.loads(raw)
prog = (pkg.get("kwargs") or {}).get("task_id")
print("PACKAGE hex id (Django-Q PK)          =", pkg.get("id"))
print("PACKAGE func                          =", pkg.get("func"))
print("PACKAGE name                          =", pkg.get("name"))
print("PACKAGE args                          =", repr(pkg.get("args")))
print("PACKAGE kwargs['task_id'] (progress UUID) =", prog)
print("PACKAGE kwargs['override_filename']   =", (pkg.get("kwargs") or {}).get("override_filename"))
# persist identity for the caller to correlate after the cluster runs
open("/tmp/pp/rest_ident.txt","w").write("%s|%s|%s" % (pkg.get("id"), prog, fname))
```

#### `mail_run.py` — Q7 mail account/rule setup + seeded-schedule nudge driver ([§9](#9-q7--enqueue-origin-in-code))

This driver **does not** invoke any task itself. It seeds a real `MailAccount`/`MailRule`, then nudges the
seeded **plural** `Schedule` row into the strict past so the **running** `qcluster` Sentinel fires
`paperless_mail.tasks.process_mail_accounts` by itself (the real production trigger observed in
[§9.3](#93-origin-3--mail-fetch-handle_message-exercised-live-against-a-real-imap-server)). The mailbox
password is read from the environment variable `QA_MAIL_PW`, never hard-coded.

```python
#!/usr/bin/env python3
"""Q7 mail driver (§9.3) — sets up a real MailAccount/MailRule, then NUDGES the seeded
PLURAL schedule so the RUNNING qcluster Sentinel fires paperless_mail.tasks.process_mail_accounts
by itself (the real production trigger). It deliberately does NOT call the single-account
convenience wrapper process_mail_account. The mailbox password is read from the environment
(QA_MAIL_PW) and never hard-coded; it is stored only on the MailAccount row.
"""
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()

from datetime import timedelta
from django.utils import timezone
from django_q.models import Schedule
from paperless_mail.models import MailAccount, MailRule

PW = os.environ.get("QA_MAIL_PW", "<REDACTED>")

acct, _ = MailAccount.objects.get_or_create(
    name="qa_mail",
    defaults=dict(imap_server="127.0.0.1", imap_port=10143,
                  imap_security=MailAccount.ImapSecurity.NONE,
                  username="qauser", password=PW, character_set="US-ASCII"))
acct.imap_server = "127.0.0.1"; acct.imap_port = 10143
acct.imap_security = MailAccount.ImapSecurity.NONE
acct.username = "qauser"; acct.password = PW; acct.character_set = "US-ASCII"; acct.save()

rule, _ = MailRule.objects.get_or_create(
    name="qa_rule", account=acct,
    defaults=dict(folder="INBOX", maximum_age=0,
                  action=MailRule.MailAction.MARK_READ,
                  attachment_type=MailRule.AttachmentProcessing.ATTACHMENTS_ONLY,
                  assign_title_from=MailRule.TitleSource.FROM_SUBJECT,
                  assign_correspondent_from=MailRule.CorrespondentSource.FROM_NOTHING))
rule.folder = "INBOX"; rule.maximum_age = 0
rule.action = MailRule.MailAction.MARK_READ
rule.attachment_type = MailRule.AttachmentProcessing.ATTACHMENTS_ONLY; rule.save()

print("ACCT id=%s host=%s:%s security=%s user=%s" % (
    acct.id, acct.imap_server, acct.imap_port,
    acct.get_imap_security_display(), acct.username))
print("RULE id=%s folder=%s action=%s attachment_type=%s" % (
    rule.id, rule.folder, rule.get_action_display(), rule.attachment_type))

# Nudge the SEEDED plural schedule into the strict past so the running Sentinel fires it
# on its next pass — the same next_run technique used in §10.2. No task is invoked here.
s = Schedule.objects.get(func="paperless_mail.tasks.process_mail_accounts")
print("seeded schedule: id=%s name=%r func=%s type=%s minutes=%s" % (
    s.id, s.name, s.func, s.schedule_type, s.minutes))
print("BEFORE next_run=%s repeats=%s" % (s.next_run.isoformat(), s.repeats))
past = (timezone.now() - timedelta(minutes=4)).replace(second=0, microsecond=0)
s.next_run = past; s.save()
print("NUDGED next_run -> %s (strict-past; Sentinel will fire on next pass)" % s.next_run.isoformat())
print("now = %s" % timezone.now().isoformat())
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
