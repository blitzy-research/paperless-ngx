# Asynchronous Background Processing During Document Ingestion in paperless-ngx

**A runtime-grounded investigation** — paperless-ngx @ commit `542221a38dff` (full SHA `542221a38dff06361e07976452f9aea24d210542`).

This document answers seven questions about how paperless-ngx runs background work during document
ingestion. It is **evidence-first**: every behavioral claim is placed next to the exact command that
produced it and that command's *actual, complete, unedited* output — captured live from a running
system — and is then correlated back to the specific source code (`file:line`) responsible.

Statements that are derived from *reading* code or third-party documentation rather than from observed
runtime output are explicitly labelled **(inferred)** or **(documentation-derived)**; wherever possible
such statements are also confirmed at runtime and the confirming observation is shown.

---

## Table of contents

1. [TL;DR — the framework framing (Django-Q, not Celery)](#1-tldr--the-framework-framing-django-q-not-celery)
2. [Environment & how to reproduce](#2-environment--how-to-reproduce)
3. [Q1 — Live async behavior (the end-to-end job lifecycle)](#q1--live-async-behavior-the-end-to-end-job-lifecycle)
4. [Q2 — Services involved (what runs behind the scenes)](#q2--services-involved-what-runs-behind-the-scenes)
5. [Q3 — How a job appears the moment it is created](#q3--how-a-job-appears-the-moment-it-is-created)
6. [Q4 — Waiting vs. actively processing (demonstrated at the boundary)](#q4--waiting-vs-actively-processing-demonstrated-at-the-boundary)
7. [Q5 — Where task state is ultimately stored](#q5--where-task-state-is-ultimately-stored)
8. [Q6 — After-the-fact status (what happened to a finished job)](#q6--after-the-fact-status-what-happened-to-a-finished-job)
9. [Q7 — Enqueue origin in code (the exact call sites)](#q7--enqueue-origin-in-code-the-exact-call-sites)
10. [Scheduled / recurring jobs](#scheduled--recurring-jobs)
11. [Exhaustive condition coverage](#exhaustive-condition-coverage)
12. [Cleanup & repository integrity](#cleanup--repository-integrity)
13. [Appendix — anchor table, code excerpts & Django-Q doc confirmations](#appendix)

---

## 1. TL;DR — the framework framing (Django-Q, **not** Celery)

Background processing in this version of paperless-ngx is powered by **Django-Q 1.3.9 running over a
Redis broker** — **not Celery**. This is the single most important fact for interpreting everything
below. "Async work" in a Django project is *commonly* assumed to be Celery; here that assumption is
**wrong**. There is no `celery`, no `@shared_task`, no `celerybeat`, and no `apply_async` anywhere in
the ingestion path; the enqueue primitive is Django-Q's `async_task(...)`, the worker is
`python3 manage.py qcluster`, and task state lives in Django-Q's own `django_q_task` table.

The lifecycle, in one picture (each arrow confirmed at runtime in the sections below):

```
 enqueue site                         Redis broker list            qcluster worker            django_q_task
 (async_task, Q7)   --rpush-->   "django_q:paperless:q"  --blpop-->  consume_file()  --monitor-->  Task row
                                    (WAITING, Q3/Q4)               (PROCESSING, Q4)            (DONE, Q5/Q6)
                                                                        |
                                                                        +--group_send-->  "status_updates" WebSocket (Q6)
```

**Runtime proof of the framework and versions** (all match the `requirements.txt` pins exactly):

```console
$ docker exec pngx bash -lc 'cd /app/src && python3 -c "import django_q, django, redis, channels; \
    print(\"django_q\", django_q.VERSION); print(\"django\", django.get_version()); \
    print(\"redis\", redis.__version__); print(\"channels\", channels.__version__)"'
django_q (1, 3, 9)
django 4.0.4
redis 3.5.3
channels 3.0.4
```

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -c "
import django; django.setup()
from django_q.brokers import get_broker
b = get_broker()
print(\"broker class =\", type(b).__module__ + \".\" + type(b).__name__)
print(\"broker info  =\", b.info())"'
broker class = django_q.brokers.redis_broker.Redis
broker info  = Redis 6.0.16
```

- The async engine is Django-Q: `django_q==1.3.9` in `requirements.txt`, and `"django_q"` is in
  `INSTALLED_APPS` at `src/paperless/settings.py:110`.
- The broker is Redis: the live broker object is `django_q.brokers.redis_broker.Redis`, configured by
  the `redis` key of `Q_CLUSTER` at `src/paperless/settings.py:456`
  (`"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")`).
- Celery is referenced here **only** to disambiguate terminology and is *not* used by this version.

---

## 2. Environment & how to reproduce

All observation was performed **inside the canonical Docker image** (the project targets
`python:3.9-slim-bullseye`, `Dockerfile:18`). The plain host shell cannot run any of this — it lacks the
pinned dependencies (`import django_q` raises `ModuleNotFoundError` there) — so every command below is
run inside the container (named `pngx`) via `docker exec`.

**Canonical interpreter & container:**

```console
$ docker exec pngx bash -lc 'python3 --version; whoami; echo "workdir: $(pwd)"'
Python 3.9.23
root
workdir: /app
```

**Redis (the Django-Q broker *and* the Channels layer)** is reachable at `PAPERLESS_REDIS`
(default `redis://localhost:6379`). The project ships a startup gate, `docker/wait-for-redis.py`
(`MAX_RETRY_COUNT=5` L16, `RETRY_SLEEP_SECONDS=5` L17, `REDIS_URL` default L19); running it confirms the
dependency:

```console
$ docker exec pngx bash -lc 'redis-cli ping'
PONG

$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
    PYTHONPATH=/app/src python3 /app/docker/wait-for-redis.py'
Waiting for Redis: redis://localhost:6379
Connected to Redis broker: redis://localhost:6379
```

**Apply migrations first.** This creates the Django-Q tables (`django_q_task`, `django_q_schedule`,
`django_q_ormq`) via the `"django_q"` app and *seeds* the recurring schedules through three data
migrations (see [Scheduled / recurring jobs](#scheduled--recurring-jobs)). Captured from a clean
database:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
    python3 manage.py migrate --no-input 2>&1 | grep -iE "Applying django_q"'
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
```

**Launch the three canonical services exactly as `docker/supervisord.conf` defines them**, each in the
**background** (never a foreground/watch process), with logs redirected to files under `/tmp`:

```bash
# from /app/src, with DJANGO_SETTINGS_MODULE=paperless.settings and PAPERLESS_REDIS set
nohup python3 manage.py qcluster                                  > /tmp/obs/qcluster.log 2>&1 &  # worker cluster + scheduler
nohup python3 manage.py document_consumer                         > /tmp/obs/consumer.log 2>&1 &  # directory watcher
nohup gunicorn -c /app/gunicorn.conf.py paperless.asgi:application > /tmp/obs/gunicorn.log 2>&1 &  # ASGI web/REST + WebSocket
```

> Note: `docker/supervisord.conf:11` references `gunicorn -c /usr/src/paperless/gunicorn.conf.py …`;
> in this image the config lives at `/app/gunicorn.conf.py` (bind `0.0.0.0:8000`, `workers=2`,
> `worker_class="paperless.workers.ConfigurableWorker"`), so the canonical invocation uses that path.
> `ConfigurableWorker` subclasses `uvicorn.workers.UvicornWorker` (uvicorn 0.17.6), which is what lets
> gunicorn serve the WebSocket endpoint over ASGI.

Confirming all three are up before exercising any entry point:

```console
$ docker exec pngx bash -lc 'tail -1 /tmp/obs/gunicorn.log; tail -1 /tmp/obs/consumer.log'
[2026-07-14 19:46:42 +0000] [28540] [INFO] Server is ready. Spawning workers
[2026-07-14 19:46:43,452] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume

$ docker exec pngx bash -lc 'python3 -c "import urllib.request as u; print(\"HTTP\", u.urlopen(\"http://localhost:8000/api/\", timeout=10).status)"'
HTTP 200
```

Every temporary helper script used below lives under `/tmp` (outside the repository tree) and is removed
afterward; see [Cleanup & repository integrity](#cleanup--repository-integrity).

---

## Q1 — Live async behavior (the end-to-end job lifecycle)

**Short answer.** When a document is ingested, an enqueue site calls
`django_q.tasks.async_task("documents.tasks.consume_file", …)`. That serializes a *task package* and
`rpush`es it onto a Redis list. A worker process in the `qcluster` cluster `blpop`s the package and
executes **`consume_file`** (`src/documents/tasks.py:184`), which delegates to
**`Consumer().try_consume_file(...)`** (`src/documents/tasks.py:236`). That method runs the multi-stage
pipeline (parse → OCR → persist → index), fires post-consumption signal handlers, and returns a success
string. A monitor process then writes the outcome to a `django_q_task` row.

**Mechanism (cause → effect), with citations.**

- The worker task is `def consume_file(path, override_filename=None, override_title=None,
  override_correspondent_id=None, override_document_type_id=None, override_tag_ids=None, task_id=None)`
  at `src/documents/tasks.py:184`.
- It delegates to `document = Consumer().try_consume_file(path, …, task_id=task_id)` at
  `src/documents/tasks.py:236`. The `Consumer` class is `src/documents/consumer.py:52`;
  `try_consume_file` begins at `src/documents/consumer.py:180`.
- On success it returns `"Success. New document id {} created".format(document.pk)`
  (`src/documents/tasks.py:247`); if the returned document is `None` it raises `ConsumerError`
  (`src/documents/tasks.py:249`; `ConsumerError` is defined at `src/documents/consumer.py:33`).
- **Six** post-consumption signal handlers run *inside the worker* after the document is stored.
  `DocumentsConfig.ready` (`src/documents/apps.py:11`) connects the `document_consumption_finished`
  signal — in this exact order — to `add_inbox_tags` (L22), `set_correspondent` (L23),
  `set_document_type` (L24), `set_tags` (L25), `set_log_entry` (L26), and `add_to_index` (L27).

**Observed evidence.** A real PDF was dropped into the consumption directory (the canonical directory-
watcher entry point). The worker log shows the complete lifecycle — task received → `consume_file`
executing → consumption finished:

```console
$ docker exec pngx bash -lc 'grep -E "running\.|processing|Consuming|consumption finished" /tmp/obs/qcluster3.log'
19:54:58 [Q] INFO Q Cluster low-skylark-fish-helium running.
19:54:58 [Q] INFO Process-1:1 processing [q4_demo.pdf]
[2026-07-14 19:54:58,424] [INFO] [paperless.consumer] Consuming q4_demo.pdf
[2026-07-14 19:54:59,745] [INFO] [paperless.consumer] Document 2026-07-14 q4_demo consumption finished
```

The persisted result row confirms the exact success string from `tasks.py:247` and that a `Document`
was created:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(func=\"documents.tasks.consume_file\", name=\"q4_demo.pdf\")
print(\"success =\", t.success)
print(\"result  =\", repr(t.result))"'
success = True
result  = 'Success. New document id 2 created'
```

**Rationale.** The three ingestion entry points (Q7) all enqueue the *same* dotted task path,
`documents.tasks.consume_file`, so the entire ingestion lifecycle converges on this one worker function.
Everything downstream of the `blpop` — parsing, OCR, thumbnailing, persistence, indexing, and the six
signal handlers — executes synchronously *within the worker process*, off the web request/response path.

> **Observed but non-fatal.** During PDF ingestion the worker logs an ImageMagick policy warning
> (`convert-im6.q16: attempt to perform an operation not allowed by the security policy 'PDF'`) while
> generating a thumbnail. The task still finished with `success=True` and the document was created — the
> thumbnail step is non-fatal to `consume_file`.

---

## Q2 — Services involved (what runs behind the scenes)

**Short answer.** Three long-running processes (defined by `docker/supervisord.conf`) plus one external
dependency, **Redis**, which plays a *dual* role. The three processes are: the ASGI web process
(`gunicorn … paperless.asgi:application`), the directory watcher
(`python3 manage.py document_consumer`), and the Django-Q worker cluster
(`python3 manage.py qcluster`, which also runs the scheduler thread).

**Mechanism (cause → effect), with citations.**

- `docker/supervisord.conf` declares exactly three programs:
  - `[program:gunicorn]` (L10) → `command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` (L11) — web/REST + WebSocket.
  - `[program:consumer]` (L19) → `command=python3 manage.py document_consumer` (L20) — the directory watcher.
  - `[program:scheduler]` (L28) → `command=python3 manage.py qcluster` (L29) — the Django-Q worker cluster **and** the scheduler thread.
- **Redis dual role:**
  1. Django-Q **broker** — `Q_CLUSTER["redis"]` at `src/paperless/settings.py:456`
     (dict `Q_CLUSTER` spans L449–L457: `name="paperless"` L450, `catch_up=False` L451, `recycle=1` L452,
     `retry` L453, `timeout` L454, `workers` L455).
  2. Channels **layer** — `CHANNEL_LAYERS` at `src/paperless/settings.py:178` uses
     `"BACKEND": "channels_redis.core.RedisChannelLayer"` (L180), `"hosts": [PAPERLESS_REDIS]` (L182),
     `"capacity": 2000` (L183), `"expiry": 15` (L184).
- `"django_q"` in `INSTALLED_APPS` (`src/paperless/settings.py:110`) is what creates the `django_q_*`
  tables. Worker count derives from CPU count via `default_task_workers()`
  (`src/paperless/settings.py:427`) feeding `TASK_WORKERS` (`src/paperless/settings.py:438`).

**Observed evidence.** The `qcluster` startup banner enumerates the worker pool and the auxiliary
threads. This machine reports 128 CPUs, and `default_task_workers()` computes `floor(sqrt(128)) = 11`,
so **11 worker processes** are spawned, plus a monitor, a "pushing tasks" thread, and a guard:

```console
$ docker exec pngx bash -lc 'sed -n "1,16p" /tmp/obs/qcluster.log'
19:46:43 [Q] INFO Q Cluster nine-muppet-oklahoma-magazine starting.
19:46:43 [Q] INFO Process-1:1 ready for work at 28568
19:46:43 [Q] INFO Process-1:2 ready for work at 28569
19:46:43 [Q] INFO Process-1:3 ready for work at 28570
19:46:43 [Q] INFO Process-1:4 ready for work at 28571
19:46:43 [Q] INFO Process-1:5 ready for work at 28572
19:46:43 [Q] INFO Process-1:6 ready for work at 28573
19:46:43 [Q] INFO Process-1:7 ready for work at 28574
19:46:43 [Q] INFO Process-1:8 ready for work at 28576
19:46:43 [Q] INFO Process-1:9 ready for work at 28577
19:46:43 [Q] INFO Process-1:10 ready for work at 28578
19:46:43 [Q] INFO Process-1:11 ready for work at 28579
19:46:43 [Q] INFO Process-1:12 monitoring at 28580
19:46:43 [Q] INFO Process-1 guarding cluster nine-muppet-oklahoma-magazine
19:46:43 [Q] INFO Process-1:13 pushing tasks at 28581
19:46:43 [Q] INFO Q Cluster nine-muppet-oklahoma-magazine running.
```

The derivation of the worker count is confirmed directly:

```console
$ docker exec pngx bash -lc 'cd /app/src && python3 -c "import multiprocessing, math; c=multiprocessing.cpu_count(); \
    print(\"cpu_count =\", c, \"| floor(sqrt(cpu)) =\", max(math.floor(math.sqrt(c)),1))"'
cpu_count = 128 | floor(sqrt(cpu)) = 11

$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -c "
import django; django.setup()
from django_q.conf import Conf
print(\"Conf.WORKERS =\", Conf.WORKERS, \"| Conf.TIMEOUT =\", Conf.TIMEOUT, \"| Conf.RETRY =\", Conf.RETRY)"'
Conf.WORKERS = 11 | Conf.TIMEOUT = 1800 | Conf.RETRY = 1810
```

**Rationale & a naming nuance.** The banner name (`nine-muppet-oklahoma-magazine`) is **not** the
configured cluster name — it is a per-boot random identifier, `humanize(self.cluster_id.hex)` where
`self.cluster_id = uuid.uuid4()` (`django_q/cluster.py:58` and `:111`). It therefore *changes on every
boot* (observed across three boots: `nine-muppet-oklahoma-magazine`, `echo-undress-ohio-vegan`,
`low-skylark-fish-helium`). The **stable** configured name is the Redis key prefix `paperless`
(`Q_CLUSTER["name"]`), which appears in the broker's list key `django_q:paperless:q` and the cluster
stats key `django_q:paperless:cluster`. The worker count (11) is deterministic given the CPU count and
was **stable across all three boots** (see [Exhaustive condition coverage](#exhaustive-condition-coverage)).


---

## Q3 — How a job appears the moment it is created

**Short answer.** Enqueuing calls `async_task("documents.tasks.consume_file", …)`, which builds a
**task package** — a dict containing an `id` (a uuid4 hex string), a human-readable `name`, the dotted
`func` path, the positional `args`, the `kwargs`, and a `started` timestamp — signs+pickles it, and
`rpush`es it onto the Redis list `django_q:paperless:q`. That is where the job first lands; it sits
there until a worker pops it.

**Mechanism (cause → effect), with citations.**

- The uniform enqueue primitive is `django_q.tasks.async_task(<dotted-task-path>, …)`. The Redis broker
  implements enqueue as a `rpush` onto its list key (see the source in Q4).
- For the directory watcher, a human-readable name is attached via
  `task_name=os.path.basename(filepath)[:100]` (`src/documents/management/commands/document_consumer.py:90`);
  the log line immediately preceding the call is `"Adding {filepath} to the task queue."`
  (`document_consumer.py:85`).
- The returned/stored `id` is the tracking id later used to find the result row.

**Observed evidence.** With the worker cluster momentarily stopped (so the package cannot be drained),
a file was dropped into the consumption directory. The watcher logged the enqueue, and the raw package
was then **peeked non-destructively** off the front of the Redis list (`LINDEX 0`, which does *not*
remove it) and decoded with Django-Q's `SignedPackage.loads`:

```console
$ docker exec pngx bash -lc 'grep -F "task queue" /tmp/obs/consumer.log | tail -1'
[2026-07-14 19:54:50,160] [INFO] [paperless.management.consumer] Adding /app/src/../consume/q4_demo.pdf to the task queue.
```

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import redis
from django_q.signing import SignedPackage
r = redis.Redis(host=\"localhost\", port=6379); KEY=b\"django_q:paperless:q\"
print(\"LLEN(waiting) =\", r.llen(KEY))
raw = r.lindex(KEY, 0)                 # PEEK front of list WITHOUT removing
print(\"raw package bytes length =\", len(raw))
print(\"raw package head (repr)  =\", repr(raw[:64]))
pkg = SignedPackage.loads(raw)         # decode the signed+pickled task package
print(\"decoded task package keys =\", sorted(pkg.keys()))
for k in (\"id\",\"name\",\"func\",\"args\",\"kwargs\"): print(\"  %-7s=\" % k, pkg.get(k))"'
LLEN(waiting) = 1
raw package bytes length = 433
raw package head (repr)  = b'gAWVEwEAAAAAAAB9lCiMAmlklIwgM2IzNzg3ZjNhN2UwNDBmNTlmYzE2MTExMTI0'
decoded task package keys = ['args', 'func', 'id', 'kwargs', 'name', 'started']
  id     = 3b3787f3a7e040f59fc1611112481358
  name   = q4_demo.pdf
  func   = documents.tasks.consume_file
  args   = ('/app/src/../consume/q4_demo.pdf',)
  kwargs = {'override_tag_ids': None}
```

**Rationale.** The decoded package is precisely what the enqueue site sent: the dotted function path
(`documents.tasks.consume_file`), the file path as the sole positional arg, and the watcher's
`override_tag_ids` kwarg. The `id` (`3b3787f3…`) is the tracking id; the same value becomes the
`django_q_task.id` of the eventual result row (confirmed in Q4). The package is opaque bytes on the
wire (a signed pickle, hence the `gAW…` base64-looking head); decoding it requires Django-Q's signing
key, which is why the peek uses `SignedPackage.loads`.

---

## Q4 — Waiting vs. actively processing (demonstrated at the boundary)

This is the most important *observed* distinction, and it is demonstrated at the boundary with
before/during/after evidence — not asserted. There are three states:

- **Waiting** — the task package is a serialized entry sitting in the **Redis list** (the broker queue).
  Django-Q's `queue_size()` counts these and **explicitly excludes tasks currently being processed**
  (documentation-derived; confirmed at runtime below). No `django_q_task` row exists yet.
- **Processing** — a worker has **popped** the package off the Redis list into the cluster's memory; it
  is no longer in the list and **still has no result row**.
- **Done** — the monitor writes a `django_q_task` row, classified `Success` or `Failure`.

**Mechanism (cause → effect), grounded in the Redis broker source.** The measurement `queue_size()`
and the raw `redis-cli LLEN` are literally the *same* operation, and the queue is FIFO
(`rpush` to enqueue, `blpop` to dequeue):

```console
$ docker exec pngx bash -lc 'DQ=$(python3 -c "import django_q,os;print(os.path.dirname(django_q.__file__))"); \
    grep -nE "def queue_size|def enqueue|def dequeue|llen|rpush|blpop|list_key" $DQ/brokers/redis_broker.py'
14:    def __init__(self, list_key: str = Conf.PREFIX):
15:        super(Redis, self).__init__(list_key=f"django_q:{list_key}:q")
17:    def enqueue(self, task):
18:        return self.connection.rpush(self.list_key, task)
20:    def dequeue(self):
21:        task = self.connection.blpop(self.list_key, 1)
25:    def queue_size(self):
26:        return self.connection.llen(self.list_key)
```

So `list_key = django_q:paperless:q` (because `Conf.PREFIX == "paperless"`), enqueue is `rpush`
(right/tail), dequeue is `blpop` (blocking left/head pop) → **FIFO**, and `queue_size()` is exactly
`LLEN(list_key)`.

**Observed evidence — a continuous poll across the boundary.** A background poller sampled
`LLEN(django_q:paperless:q)` (waiting) and `Task.objects.count()` (done) every 0.25 s. Sequence: the
worker cluster was stopped; a file was dropped (the watcher enqueued it → **waiting**); then the cluster
was restarted (a worker dequeued it → **processing**; then finished → **done**). The full trace,
condensed to its state-change rows (the raw trace has one line every 0.25 s):

```console
$ docker exec pngx bash -lc "awk 'NR==1{print;next}{k=\$2\" \"\$3; if(k!=p){print;p=k}}' /tmp/obs/q4_poll.log"
elapsed_s  LLEN(waiting)  task_rows(done)
     0.01              0                0       <-- BEFORE:     queue empty, no result row
     2.27              1                0       <-- WAITING:    package in Redis list, cluster down, no row
    10.32              0                0       <-- PROCESSING: worker popped it (LLEN back to 0), not yet persisted
    11.82              0                1       <-- DONE:        monitor wrote the django_q_task row
```

Reading the trace: for ~8 s (elapsed 2.27 → 10.07) the task is **waiting** (`LLEN=1`, `rows=0`); for
~1.5 s (10.32 → 11.57, after the cluster restart) it is **processing** — neither in the queue
(`LLEN=0`) nor persisted (`rows=0`); from 11.82 s on it is **done** (`rows=1`). The corroborating
`queue_size()` reads the same list length, and equals 0 when idle:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django, time; django.setup()
from django_q.brokers import get_broker
b = get_broker()
print(\"queue_size() read#1 =\", b.queue_size()); time.sleep(0.5)
print(\"queue_size() read#2 =\", b.queue_size())"; redis-cli llen django_q:paperless:q'
queue_size() read#1 = 0
queue_size() read#2 = 0
0
```

Confirming the documentation claim that `queue_size()` **excludes in-progress work**: in the trace, the
"processing" window shows `LLEN=0` while the task is running (worker log `processing [q4_demo.pdf]`) but
before any row is written — the running task is counted by *neither* `queue_size()` *nor* the
`django_q_task` table. The corresponding result row (written at the "done" boundary) proves the id
matches the waiting package's id from Q3 (`3b3787f3…`) and that `time_taken` **includes queue wait**:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(func=\"documents.tasks.consume_file\", name=\"q4_demo.pdf\")
print(\"id         =\", t.id)
print(\"started    =\", t.started)
print(\"stopped    =\", t.stopped)
print(\"time_taken =\", t.time_taken())
print(\"success    =\", t.success)"'
id         = 3b3787f3a7e040f59fc1611112481358
started    = 2026-07-14 19:54:50.160770+00:00
stopped    = 2026-07-14 19:54:59.748011+00:00
time_taken = 9.587241
success    = True
```

Note `started` (19:54:50.160) equals the enqueue instant (the watcher logged "Adding … to the task
queue." at 19:54:50.160), and `time_taken=9.587 s` spans the ~7 s the task spent waiting (cluster down)
plus the ~2 s of actual work — consistent with Django-Q's definition that "time taken represents the
time a task spends in the cluster, this includes any time it may have waited in the queue."

**Timing bounds.** `PAPERLESS_WORKER_TIMEOUT=1800` (`src/paperless/settings.py:440`) and
`PAPERLESS_WORKER_RETRY = timeout + 10 = 1810` (`src/paperless/settings.py:444-446`; confirmed at
runtime `Conf.TIMEOUT=1800`, `Conf.RETRY=1810` in Q2). The **Redis broker does not support message
receipts** (documentation-derived: "The default Redis broker does not support message receipts …
tasks that were being executed get lost" on crash/timeout). Django-Q only actively *re-delivers* on
brokers that support receipts, so under the canonical Redis broker the `retry` timer is effectively a
**safety bound**, not an active re-delivery mechanism **(inferred** from the broker's lack of receipts;
a forced re-delivery was not reproduced because it would require killing a worker mid-task, which is not
a canonical ingestion condition).


---

## Q5 — Where task state is ultimately stored

**Short answer.** In Django-Q's own **`django_q_task`** table, mapped by the model
**`django_q.models.Task`**. paperless-ngx (this version) defines **no** custom task-state model — it
relies entirely on Django-Q's built-in tables.

**Mechanism (cause → effect), with citations.**

- The `"django_q"` app (`src/paperless/settings.py:110`) owns the schema; its migrations
  (`django_q.0001…0014`, applied in [§2](#2-environment--how-to-reproduce)) create `django_q_task`,
  `django_q_schedule`, and `django_q_ormq`.
- **Proof of absence of a custom model:** `src/documents/models.py` contains only the domain models
  `MatchingModel` (L19), `Correspondent` (L57), `Tag` (L64), `DocumentType` (L82), `Document` (L88),
  `Log` (L285), `SavedView` (L316), `SavedViewFilterRule` (L342), plus the non-model helper class
  `FileInfo` (L386). None is task/queue/schedule related.

**Observed evidence.** The model→table mapping and the proxy relationships are read directly from the
live ORM, and a real result row is shown:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task, Schedule, Success, Failure, OrmQ
print(\"Task._meta.db_table     =\", Task._meta.db_table)
print(\"Schedule._meta.db_table =\", Schedule._meta.db_table)
print(\"OrmQ._meta.db_table     =\", OrmQ._meta.db_table)
print(\"Success proxy of        =\", Success._meta.proxy, Success._meta.concrete_model.__name__)
print(\"Failure proxy of        =\", Failure._meta.proxy, Failure._meta.concrete_model.__name__)"'
Task._meta.db_table     = django_q_task
Schedule._meta.db_table = django_q_schedule
OrmQ._meta.db_table     = django_q_ormq
Success proxy of        = True Task
Failure proxy of        = True Task
```

Proof that `src/documents/models.py` declares no task-related model (zero matches for the relevant
keywords, while the ordinary domain models are all present):

```console
$ docker exec pngx bash -lc 'grep -icE "task|celery|django_q|class .*Queue|Schedule" /app/src/documents/models.py'
0
$ docker exec pngx bash -lc 'grep -nE "^class " /app/src/documents/models.py'
19:class MatchingModel(models.Model):
57:class Correspondent(MatchingModel):
64:class Tag(MatchingModel):
82:class DocumentType(MatchingModel):
88:class Document(models.Model):
285:class Log(models.Model):
316:class SavedView(models.Model):
342:class SavedViewFilterRule(models.Model):
386:class FileInfo:
```

**Rationale.** Because state lives exclusively in `django_q_task`, an operator inspects job outcomes
through Django-Q's model (`django_q.models.Task`) and its admin, never through a paperless-specific
table. This is the decisive fact for Q6.

---

## Q6 — After-the-fact status (what happened to a finished job)

**Short answer.** Durable outcome is a row in `django_q_task` (`django_q.models.Task`) carrying
`func`, `args`, `kwargs`, `result`, `started`, `stopped`, `time_taken`, and a boolean `success`. The
`Success` and `Failure` **proxy models** (filtered on `success=True`/`success=False`) back the
"Successful tasks" / "Failed tasks" admin pages. Separately and *ephemerally*, live progress streams
over the Channels WebSocket group `status_updates` while the task runs. These two channels are distinct
and must not be conflated.

**Mechanism (cause → effect), with citations.**

- **Durable row.** Written by the monitor to `django_q_task`. Django-Q's `Task` exposes `func`,
  `args`, `kwargs`, `result` ("Contains the error if any occur"), `started`, `stopped`, `time_taken`
  (`stopped − started`), and `success` (documentation-derived; confirmed by the rows below).
- **Proxy models & admin.** `Success` and `Failure` are proxies of `Task` (confirmed in Q5). The
  "Queued tasks" (`OrmQ`) admin view is **ORM-broker-only** — Django-Q registers it as
  `if Conf.ORM or Conf.TESTING: admin.site.register(OrmQ, QueueAdmin)` (documentation-derived) — so it
  is **absent** under this Redis deployment.
- **Live stream.** `consume_file` and the `Consumer._send_progress` path
  (`src/documents/consumer.py:56`) broadcast to the `status_updates` group via
  `async_to_sync(get_channel_layer().group_send)("status_updates", {"type":"status_update","data":payload})`
  (payload built at `consumer.py:64-72`; broadcast at `consumer.py:73-76`; the barcode-split branch also
  broadcasts at `src/documents/tasks.py:226-227`). `StatusConsumer` (`src/paperless/consumers.py:9`)
  subscribes via `group_add("status_updates", self.channel_name)` (L17-19) and forwards each event in
  `status_update` (L29) with `self.send(json.dumps(event["data"]))` (L33). Routing is
  `ProtocolTypeRouter({... "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))})`
  (`src/paperless/asgi.py:17,20`); the URL is `re_path(r"ws/status/$", StatusConsumer.as_asgi())`
  (`src/paperless/urls.py:137`).

**Observed evidence — durable row (success case).** All `Task` fields for a real ingestion:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(func=\"documents.tasks.consume_file\", name=\"rest_upload_demo.png\")
for k in (\"id\",\"name\",\"func\",\"args\",\"kwargs\",\"started\",\"stopped\"):
    print(\"  %-10s=\" % k, getattr(t, k))
print(\"  time_taken=\", t.time_taken())
print(\"  success   =\", t.success)
print(\"  result    =\", repr(t.result))"'
  id        = 3d398e7b32174f988e00c915545b335b
  name      = rest_upload_demo.png
  func      = documents.tasks.consume_file
  args      = ('/tmp/paperless/paperless-upload-thndsqc9',)
  kwargs    = {'override_filename': 'rest_upload_demo.png', 'override_title': None, 'override_correspondent_id': None, 'override_document_type_id': None, 'override_tag_ids': None, 'task_id': '80504422-9471-47b3-8e30-02ffd6d7c2d7'}
  started   = 2026-07-14 19:57:20.795205+00:00
  stopped   = 2026-07-14 19:57:22.643564+00:00
  time_taken= 1.848359
  success   = True
  result    = 'Success. New document id 3 created'
```

**Observed evidence — proxy counts and OrmQ absence.** `Success`/`Failure` counts partition the table,
and the "Queued tasks" (`OrmQ`) admin is **not** registered under the Redis broker:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django.contrib import admin
from django_q.models import Task, Schedule, Success, Failure, OrmQ
from django_q.conf import Conf
reg = admin.site._registry
print(\"Success registered? \", Success in reg)
print(\"Failure registered? \", Failure in reg)
print(\"Schedule registered?\", Schedule in reg)
print(\"OrmQ registered?    \", OrmQ in reg, \"<-- Queued tasks admin ABSENT under Redis broker\")
print(\"Conf.ORM =\", Conf.ORM, \"| Conf.TESTING =\", Conf.TESTING)
print(\"django_q_ormq rows  =\", OrmQ.objects.count())
print(\"Success count =\", Success.objects.count(), \"| Failure count =\", Failure.objects.count())"'
Success registered?  True
Failure registered?  True
Schedule registered? True
OrmQ registered?     False <-- Queued tasks admin ABSENT under Redis broker
Conf.ORM = None | Conf.TESTING = False
django_q_ormq rows  = 0
Success count = 3 | Failure count = 1
```

**Observed evidence — the "Successful tasks" admin page (visual corroboration).** The screenshot
`blitzy/screenshots/django_q_successful_tasks.png` (captured during an **earlier** setup/probe run — the
task names differ from the run documented above, so it is shown only as structural corroboration, not as
the same rows) renders the Django admin at **Home › Django Q › Successful tasks**. Its left navigation
lists the three Django-Q admin sections — **Failed tasks, Scheduled tasks, Successful tasks** — and
notably **no "Queued tasks" entry** (consistent with the `OrmQ`-absent result above). The changelist
columns are exactly the `Task` fields **NAME, FUNC, STARTED, STOPPED, TIME TAKEN, GROUP**, and the seven
rows include `documents.tasks.consume_file` ingestions alongside the four scheduled-job firings
(`index_optimize`/"Optimize the index", `train_classifier`/"Train the classifier",
`sanity_check`/"Perform sanity check", `paperless_mail.tasks.process_mail_accounts`/"Check all e-mail
accounts"), where the `GROUP` column shows the originating schedule name. This is the admin surface
backed by the `Success` proxy model.

**Observed evidence — live WebSocket stream (distinct from the durable row).** An authenticated client
connected to `ws://localhost:8000/ws/status/` (Django session cookie; `StatusConsumer` denies
unauthenticated connections at `consumers.py:13-15`) and captured the frames streamed while a real
document was ingested:

```console
$ docker exec pngx bash -lc 'cd /app/src && python3 /tmp/obs/ws_capture.py'
login POST status = 302 | sessionid obtained = True
WS CONNECTED to ws://localhost:8000/ws/status/
[trigger] REST upload ws_demo.jpg -> HTTP 200 "OK"
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
WS FRAME: {"filename": "ws_demo.jpg", "task_id": "19af0c11-246c-4126-8c99-9fea00a923c0", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 4}
[capture window closed] total status_updates frames = 6
```

The six frames map one-to-one to the `_send_progress` calls in the pipeline: STARTING(0) at
`consumer.py:202`, WORKING(20 parsing) at `:259`, WORKING(70 thumbnail) at `:264`, WORKING(90 date) at
`:274`, WORKING(95 save) at `:294`, and SUCCESS(100, with `document_id`) at `:375`.

**Rationale & the two-id nuance.** The **live** stream and the **durable** row are different channels
keyed by different identifiers. Each WebSocket frame carries `task_id = "19af0c11-…"` — the `uuid4()`
generated at the enqueue site (`views.py:521`) and passed as the `task_id` *kwarg*. That is **not** the
Django-Q `Task.id`. The durable row for this same ingestion shows both, side by side:

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(func=\"documents.tasks.consume_file\", name=\"ws_demo.jpg\")
print(\"django-q Task.id =\", t.id)
print(\"task_id kwarg    =\", t.kwargs.get(\"task_id\"), \"(== the task_id in the WebSocket frames)\")
print(\"result           =\", repr(t.result))"'
django-q Task.id = d5af144bde8c4e758e91220e28046818
task_id kwarg    = 19af0c11-246c-4126-8c99-9fea00a923c0 (== the task_id in the WebSocket frames)
result           = 'Success. New document id 4 created'
```

So the front end tracks *live* progress by the `uuid4` correlation id (`task_id` kwarg), while the
*durable* history is queried by the Django-Q `Task.id`. Conflating the two would mis-answer both Q4 and
Q6.

---

## Q7 — Enqueue origin in code (the exact call sites)

**Short answer.** Every background job is triggered by `django_q.tasks.async_task(<dotted-path>, …)`.
For ingestion the dotted path is always `documents.tasks.consume_file`, called from **three** entry
points — the directory watcher, the REST upload view, and the mail handler — which all **converge on
that one worker function**. Bulk operations call `documents.tasks.bulk_update_documents` from **five**
sites in `bulk_edit.py`. The exact call sites:

| Entry point | Function | Enqueue call site | Task path |
|---|---|---|---|
| Directory watcher | `_consume` (`document_consumer.py:46`) | `async_task(...)` at `document_consumer.py:86` (preceded by the "Adding … to the task queue." log at `:85`; `task_name` at `:90`) | `documents.tasks.consume_file` |
| REST upload | `PostDocumentView.post` (`views.py:497`, class `:491`) | `task_id = str(uuid.uuid4())` at `views.py:521`, then `async_task(...)` at `views.py:523-524` | `documents.tasks.consume_file` |
| Mail fetch | `handle_message` (`mail.py:272`) | `async_task(...)` at `mail.py:336-337` (`task_name` at `:348`) | `documents.tasks.consume_file` |
| Bulk edit ×5 | `set_correspondent` (`:10`), `set_document_type` (`:23`), `add_tag` (`:36`), `remove_tag` (`:52`), `modify_tags` (`:68`) | `async_task(...)` at `bulk_edit.py:18, 31, 47, 63, 87` | `documents.tasks.bulk_update_documents` |

`bulk_edit.delete()` (`bulk_edit.py:92`) deletes documents and removes them from the index but does
**not** enqueue a task.

**Observed evidence — directory watcher (exercised live).** The enqueue log line, and the resulting
`django_q_task` row proving this call site reaches `consume_file` (see Q1/Q3/Q4 for the full lifecycle):

```console
$ docker exec pngx bash -lc 'grep -F "task queue" /tmp/obs/consumer.log | tail -1'
[2026-07-14 19:54:50,160] [INFO] [paperless.management.consumer] Adding /app/src/../consume/q4_demo.pdf to the task queue.
```

**Observed evidence — REST upload (exercised live).** A real `POST` to the canonical endpoint, and the
resulting task:

```console
$ docker exec pngx bash -lc 'cd /app/src && python3 /tmp/obs/rest_upload.py'
POST http://localhost:8000/api/documents/post_document/
HTTP status = 200
response body (returned task_id) = '"OK"'

$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(name=\"rest_upload_demo.png\")
print(\"func   =\", t.func)
print(\"args   =\", t.args, \"(temp file written by PostDocumentView.post into SCRATCH_DIR)\")
print(\"result =\", repr(t.result))"'
func   = documents.tasks.consume_file
args   = ('/tmp/paperless/paperless-upload-thndsqc9',)
result = 'Success. New document id 3 created'
```

**Observed evidence — bulk edit (exercised live).** A real `POST` to `/api/documents/bulk_edit/` with
method `set_correspondent`, and the resulting `bulk_update_documents` task:

```console
$ docker exec pngx bash -lc 'cd /app/src && python3 /tmp/obs/bulk_edit_test.py'
correspondent id = 1 | document ids = [3, 2]
POST http://localhost:8000/api/documents/bulk_edit/
HTTP status = 200
response body = '{"result":"OK"}'

$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
t = Task.objects.get(func=\"documents.tasks.bulk_update_documents\")
print(\"func   =\", t.func)
print(\"name   =\", t.name, \"(auto-generated: bulk_edit does not pass task_name)\")
print(\"kwargs =\", t.kwargs, \"(matches bulk_edit.py:18 document_ids=affected_docs)\")
print(\"success=\", t.success)"'
func   = documents.tasks.bulk_update_documents
name   = vegan-yellow-mississippi-mexico (auto-generated: bulk_edit does not pass task_name)
kwargs = {'document_ids': [3, 2]}
success= True
```

**Mail fetch — code-cited (inferred, not exercised live).** The mail path enqueues identically at
`src/paperless_mail/mail.py:336-337` (`async_task("documents.tasks.consume_file", …)`, `task_name` at
`:348`) inside `handle_message` (`mail.py:272`). Exercising it end-to-end requires a live IMAP server,
which is outside the canonical ingestion setup available here; the call site is therefore **cited from
source** and its live firing is labelled **(inferred)**. It nonetheless converges on the same task path
as the other two ingestion entry points.

**Rationale.** Three distinct triggers → one worker function (`documents.tasks.consume_file`). This
convergence is why the lifecycle in Q1 is identical regardless of how a document enters the system, and
why a single `django_q_task` row schema (Q5/Q6) suffices to record every ingestion outcome.


---

## Scheduled / recurring jobs

Beyond ingestion, paperless-ngx runs recurring background jobs. These are **not** registered in
application code — they are **seeded into `django_q_schedule` by data migrations** (`RunPython`), then
enqueued by the `qcluster` scheduler thread on their cadence.

**Mechanism (cause → effect), with citations.** Four schedules are seeded by three migrations:

- `process_mail_accounts` — **MINUTES** (every 10) — `src/paperless_mail/migrations/0002_auto_20201117_1334.py`
  (`schedule("paperless_mail.tasks.process_mail_accounts", name="Check all e-mail accounts",
  schedule_type=Schedule.MINUTES, minutes=10)`).
- `train_classifier` — **HOURLY** — and `index_optimize` — **DAILY** —
  `src/documents/migrations/1001_auto_20201109_1636.py`.
- `sanity_check` — **WEEKLY** — `src/documents/migrations/1004_sanity_check_schedule.py`.

**Observed evidence — the seeded rows** (the `schedule_type` codes are Django-Q's:
`I`=Minutes, `H`=Hourly, `D`=Daily, `W`=Weekly; negative `repeats` denotes an unbounded recurring
schedule):

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Schedule
for s in Schedule.objects.all().order_by(\"id\"):
    print(f\"id={s.id} func={s.func!r} name={s.name!r} type={s.schedule_type} minutes={s.minutes} next_run={s.next_run}\")"'
id=1 func='documents.tasks.train_classifier' name='Train the classifier' type=H minutes=None next_run=2026-07-14 20:46:01.181878+00:00
id=2 func='documents.tasks.index_optimize' name='Optimize the index' type=D minutes=None next_run=2026-07-15 19:46:01.183212+00:00
id=3 func='documents.tasks.sanity_check' name='Perform sanity check' type=W minutes=None next_run=2026-07-21 19:46:01.572896+00:00
id=4 func='paperless_mail.tasks.process_mail_accounts' name='Check all e-mail accounts' type=I minutes=10 next_run=2026-07-14 20:16:02.730397+00:00
```

**Observed evidence — the scheduler thread enqueues them.** At cluster startup the "pushing tasks"
thread created a task from *each* of the four schedules (from the boot-1 worker log):

```console
$ docker exec pngx bash -lc 'grep -oE "created a task from schedule \[[^]]*\]" /tmp/obs/qcluster.log | sort -u'
created a task from schedule [Check all e-mail accounts]
created a task from schedule [Optimize the index]
created a task from schedule [Perform sanity check]
created a task from schedule [Train the classifier]
```

**Observed evidence — recurrence on cadence.** `process_mail_accounts` (MINUTES=10) fired repeatedly;
two successive firings are ~10 minutes apart, matching its cadence (this is *observed*, not inferred):

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task
prev=None
for t in Task.objects.filter(func=\"paperless_mail.tasks.process_mail_accounts\").order_by(\"started\"):
    gap = (t.started - prev).total_seconds() if prev else None
    print(f\"started={t.started}  gap_since_prev={gap}\")
    prev=t.started"'
started=2026-07-14 19:56:27.984183+00:00  gap_since_prev=None
started=2026-07-14 20:06:29.532740+00:00  gap_since_prev=601.548557
```

**Rationale.** The scheduler thread inside `qcluster` polls `django_q_schedule` for rows whose
`next_run <= now`, enqueues the associated task (which then flows through the same
waiting→processing→done lifecycle as any other task), and advances `next_run` by the cadence — which is
why the `next_run` values above are all in the future (hourly/daily/weekly/+10 min) after the schedules
have already fired at least once. The hourly/daily/weekly jobs did not re-fire during the observation
window because their next cadence had not yet arrived; their seeded rows and boot firing are shown
above, and their re-firing is a direct consequence of the same mechanism **(inferred** for the specific
future timestamps, which lie beyond the observation window).

---

## Exhaustive condition coverage

The questions imply more than the happy path. Every condition below was exercised through a **canonical**
entry point (no mocks, no stand-ins) and observed.

### Primary (success)

Covered throughout Q1/Q3/Q4/Q6/Q7: normal documents ingested end-to-end via the directory watcher
(`q4_demo.pdf` → document 2), the REST upload (`rest_upload_demo.png` → document 3), and again during the
WebSocket capture (`ws_demo.jpg` → document 4), each producing a `success=True` row with
`result="Success. New document id N created"`.

### Error / failure (real, not mocked)

A **duplicate** was forced through the canonical directory watcher: the same content was dropped a
second time, so `Consumer.pre_check_duplicate` (`src/documents/consumer.py:102`) rejected it via `_fail`
(`:110` → `:81 raise ConsumerError`). The worker log and the persisted `Failure` row:

```console
$ docker exec pngx bash -lc 'grep -E "processing \[dup_demo|Not consuming|Failed \[dup_demo" /tmp/obs/qcluster3.log'
19:57:58 [Q] INFO Process-1:4 processing [dup_demo.pdf]
[2026-07-14 19:57:58,855] [ERROR] [paperless.consumer] Not consuming dup_demo.pdf: It is a duplicate.
19:57:58 [Q] ERROR Failed [dup_demo.pdf] - dup_demo.pdf: Not consuming dup_demo.pdf: It is a duplicate. : Traceback (most recent call last):
```

```console
$ docker exec pngx bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src python3 -c "
import django; django.setup()
from django_q.models import Task, Failure
t = Task.objects.get(name=\"dup_demo.pdf\")
print(\"success    =\", t.success)
print(\"time_taken =\", t.time_taken())
print(\"result     =\", repr(t.result)[:300], \"...\")
print(\"Failure count =\", Failure.objects.count())"'
success    = False
time_taken = 0.181926
result     = 'dup_demo.pdf: Not consuming dup_demo.pdf: It is a duplicate. : Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file( ...
Failure count = 1
```

The failure is a real `ConsumerError`: `success=False`, and the **full traceback is captured in
`Task.result`**. That traceback even confirms the worker's call mechanism from observed output —
`django_q/cluster.py:432` runs `res = f(*task["args"], **task["kwargs"])`, which calls
`tasks.py:236 consume_file` → `consumer.py:213 pre_check_duplicate` → `consumer.py:110 _fail` →
`consumer.py:81 raise ConsumerError`. This matches the Django-Q documentation that "Failures are always
saved" and that "the worker will try to put the error in the result field."

### Transitional (Q4)

The waiting → processing → done boundary is demonstrated with the before/during/after poll trace in
[Q4](#q4--waiting-vs-actively-processing-demonstrated-at-the-boundary).

### Scheduled

The seeded `django_q_schedule` rows, the boot-time firing of all four, and the observed 10-minute
recurrence of `process_mail_accounts` are shown in
[Scheduled / recurring jobs](#scheduled--recurring-jobs).

### Alternate flag / variant (barcode split)

`consume_file` has an alternate branch, gated on `settings.CONSUMER_ENABLE_BARCODES`, that splits a
document on separator barcodes and returns `"File successfully split"` after broadcasting a `SUCCESS`
progress payload (`src/documents/tasks.py:226-227`). It was **not** exercised because that flag is
disabled in the default canonical configuration; it is cited from source and labelled **(inferred)** as
a variant of the same lifecycle.

### Stability across ≥2 runs

The worker count (a magnitude value) was **stable at 11 across three cluster boots**, whose per-boot
random names differ while the worker pool does not:

```console
$ docker exec pngx bash -lc 'for f in qcluster qcluster2 qcluster3; do \
    echo "-- $f --"; grep -E "starting\.|ready for work" /tmp/obs/$f.log | \
    sed -nE "s/.*(Q Cluster .* starting\.).*/\1/p; s/.*(Process-1:11) ready for work.*/\1 present/p"; done'
-- qcluster --
Q Cluster nine-muppet-oklahoma-magazine starting.
Process-1:11 present
-- qcluster2 --
Q Cluster echo-undress-ohio-vegan starting.
Process-1:11 present
-- qcluster3 --
Q Cluster low-skylark-fish-helium starting.
Process-1:11 present
```

Idle `queue_size()` was likewise stable at `0` across repeated reads (Q4). Per-task `time_taken` varies
by task type and by queue-wait time (e.g. ~9.6 s for a PDF that waited ~7 s in the queue, ~1.8 s for an
immediately-processed REST image, ~0.17 s for a bulk re-index, ~0.18 s for the fast duplicate failure);
that variance is expected and is a property of the work performed, not of the cluster sizing.


---

## Cleanup & repository integrity

The investigation created **no** files inside the repository other than this one document. Every
observation helper was a throwaway script kept **under `/tmp`** — outside the repository tree — in two
locations: `/tmp/obs` inside the canonical container (where the services ran) and `/tmp/obs_host` on the
host (working copies of the scripts, plus copies of the captured logs). Nothing under `/tmp` is part of
the repository.

**Temporary artifacts that were created (and then removed).** Container `/tmp/obs` (18 items):
the orchestration/observation scripts `start_services.sh`, `killtree.sh`, `q4_poller.py`,
`peek_package.py`, `q4_run.sh`, `rest_upload.py`, `bulk_edit_test.py`, `ws_capture.py`; the log captures
`migrate.log`, `qcluster.log`, `qcluster2.log`, `qcluster3.log`, `consumer.log`, `gunicorn.log`,
`q4_poll.log`, `q4_poller.out`, `pids.txt`; and `db.sqlite3.bak` (a backup of the container's ephemeral
SQLite DB taken before migrating). Host `/tmp/obs_host` (15 files): the same scripts plus a `captured/`
copy of the logs.

**Step 1 — stop every background service (graceful `SIGTERM` to the exact master PIDs; never by name).**

```console
$ docker exec pngx bash -lc 'for pid in 32205 28539 28540; do kill -TERM $pid && echo "SIGTERM -> $pid"; done'
SIGTERM -> 32205
SIGTERM -> 28539
SIGTERM -> 28540

# qcluster drained and shut down cleanly (tail of /tmp/obs/qcluster3.log):
20:14:36 [Q] INFO Process-1 waiting for the monitor.
20:14:36 [Q] INFO Process-1:12 stopped monitoring results
20:14:36 [Q] INFO Q Cluster low-skylark-fish-helium has stopped.

# gunicorn master shut down cleanly (tail of /tmp/obs/gunicorn.log):
[2026-07-14 20:14:34 +0000] [28540] [INFO] Handling signal: term
[2026-07-14 20:14:34 +0000] [28540] [INFO] Shutting down: Master
```

A precise post-shutdown scan (excluding the scanning shell itself) confirms **no** service process
remains — no watch-mode or interactive process was left running:

```console
$ docker exec pngx bash -lc 'mypid=$$; found=0
for d in /proc/[0-9]*; do p=${d#/proc/}; [ "$p" = "$mypid" ] && continue
  cmd=$(tr "\0" " " < /proc/$p/cmdline 2>/dev/null)
  case "$cmd" in *"manage.py qcluster"*|*"manage.py document_consumer"*|*"gunicorn -c"*)
    case "$cmd" in bash*) : ;; *) echo "STILL RUNNING $p : $cmd"; found=$((found+1));; esac;; esac
done; echo "real_service_procs_remaining=$found"'
real_service_procs_remaining=0
```

**Step 2 — remove the scratch directories** (and restore the container's ephemeral DB to its
pre-investigation baseline first):

```console
$ docker exec pngx bash -lc 'cp -f /tmp/obs/db.sqlite3.bak /app/data/db.sqlite3 && rm -rf /tmp/obs && [ -d /tmp/obs ] && echo STILL || echo "REMOVED (absent)"'
REMOVED (absent)

$ rm -rf /tmp/obs_host && { [ -d /tmp/obs_host ] && echo STILL || echo "REMOVED (absent)"; }
REMOVED (absent)
```

**Step 3 — verify the repository is unchanged except for this one document.** No tracked file was
modified (`git diff` is empty), and the only untracked additions are under `blitzy/`:

```console
$ git -C /tmp/blitzy/paperless-ngx/blitzy-a0535bb1-52e0-488f-9eaf-09f99118e2cf_9a2c37 diff --stat
$        # (empty — zero tracked files changed)

$ git -C /tmp/blitzy/paperless-ngx/blitzy-a0535bb1-52e0-488f-9eaf-09f99118e2cf_9a2c37 status --porcelain -uall
?? blitzy/documentation/paperless-ngx_542221a38dff.md
?? blitzy/screenshots/admin_dashboard_logged_in.png
?? blitzy/screenshots/admin_login_page.png
?? blitzy/screenshots/django_q_successful_tasks.png
```

The single deliverable is `blitzy/documentation/paperless-ngx_542221a38dff.md`. The three `.png` files
under `blitzy/screenshots/` are **pre-existing** evidence artifacts produced during environment setup
(not by this investigation); one of them, `django_q_successful_tasks.png`, is referenced in
[Q6](#q6--after-the-fact-status-what-happened-to-a-finished-job) as it visually corroborates the
Django-Q "Successful tasks" admin page. No source file anywhere in the tree was created, modified, or
deleted.

> **Note on the runtime sandbox.** All observation ran inside the throwaway container `pngx`, whose
> `/app` is a *separate* checkout used only as an execution sandbox. Its runtime data directories
> (`/app/data`, `/app/media`, `/app/consume`) and its SQLite DB are ephemeral container state, distinct
> from the read-only repository working tree that holds this deliverable; the container is discarded
> after the investigation. The repository working tree — the artifact that matters — is byte-for-byte
> unchanged except for this document.


---

## Appendix

### A.1 — Consolidated source-of-truth anchor table (verified against the working tree @ `542221a38dff`)

Every anchor below was re-confirmed with `grep`/`sed` against the checked-out source at write time.

| Topic | File:Line | What it shows |
|---|---|---|
| Main worker task | `src/documents/tasks.py:184` | `def consume_file(...)` |
| WS broadcast (barcode branch) | `src/documents/tasks.py:226` | `async_to_sync(get_channel_layer().group_send)(...)` |
| Delegation to pipeline | `src/documents/tasks.py:236` | `Consumer().try_consume_file(...)` |
| Success return | `src/documents/tasks.py:247` | `"Success. New document id {} created"` |
| Failure | `src/documents/tasks.py:249` | `raise ConsumerError(...)` |
| Pipeline entry | `src/documents/consumer.py:52,180` | `class Consumer(LoggingMixin)`, `try_consume_file` |
| Progress broadcast | `src/documents/consumer.py:56` | `_send_progress(...)` |
| Duplicate rejection | `src/documents/consumer.py:102,110,81` | `pre_check_duplicate` → `_fail` → `raise ConsumerError` |
| Watcher enqueue | `src/documents/management/commands/document_consumer.py:46,85,86,90` | `_consume` → log → `async_task("documents.tasks.consume_file", …, task_name=…)` |
| REST enqueue | `src/documents/views.py:491,497,521,523` | `PostDocumentView` → `post` → `task_id=str(uuid.uuid4())` → `async_task(...)` |
| Mail enqueue | `src/paperless_mail/mail.py:272,336,348` | `handle_message` → `async_task(...)` → `task_name=att.filename[:100]` |
| Bulk enqueue (×5) | `src/documents/bulk_edit.py:18,31,47,63,87` | `async_task("documents.tasks.bulk_update_documents", …)` |
| Bulk delete (no enqueue) | `src/documents/bulk_edit.py:92` | `def delete(doc_ids)` — no `async_task` |
| Post-consume signals (×6) | `src/documents/apps.py:22-27` | `add_inbox_tags, set_correspondent, set_document_type, set_tags, set_log_entry, add_to_index` |
| No custom task model | `src/documents/models.py:19,57,64,82,88,285,316,342,386` | only domain models + `FileInfo`; zero task refs |
| `django_q` app | `src/paperless/settings.py:110` | creates `django_q_*` tables |
| Channels layer | `src/paperless/settings.py:178,180` | `RedisChannelLayer` |
| Worker count | `src/paperless/settings.py:427,438` | `default_task_workers()` → `TASK_WORKERS` |
| Timeout / retry | `src/paperless/settings.py:440,444-446` | `1800` / `1810` (= timeout+10) |
| Q_CLUSTER | `src/paperless/settings.py:449-457` | name=`paperless`, catch_up=False, recycle=1, retry, timeout, workers, redis |
| ASGI routing | `src/paperless/asgi.py:17,20` | `ProtocolTypeRouter` / `AuthMiddlewareStack(URLRouter(...))` |
| WS consumer | `src/paperless/consumers.py:9,17,29,33` | `StatusConsumer`, `group_add("status_updates", …)`, `status_update`, `self.send(...)` |
| WS route | `src/paperless/urls.py:137` | `re_path(r"ws/status/$", StatusConsumer.as_asgi())` |
| Supervisord programs | `docker/supervisord.conf:10-11,19-20,28-29` | gunicorn / consumer / scheduler(qcluster) |
| Redis gate | `docker/wait-for-redis.py:16,17,19` | retry count/sleep, `PAPERLESS_REDIS` default |
| Schedule: mail | `src/paperless_mail/migrations/0002_auto_20201117_1334.py` | `process_mail_accounts` MINUTES |
| Schedule: classifier/index | `src/documents/migrations/1001_auto_20201109_1636.py` | `train_classifier` HOURLY + `index_optimize` DAILY |
| Schedule: sanity | `src/documents/migrations/1004_sanity_check_schedule.py` | `sanity_check` WEEKLY |

### A.2 — Key cited code excerpts (short)

**Uniform enqueue primitive — REST upload** (`src/documents/views.py:521-531`):

```python
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
```

**Uniform enqueue primitive — directory watcher** (`src/documents/management/commands/document_consumer.py:85-90`):

```python
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
            filepath,
            override_tag_ids=list(tag_ids) if tag_ids else None,
            task_name=os.path.basename(filepath)[:100],
        )
```

**Broker semantics that define waiting-vs-processing (Q4)** — Django-Q's Redis broker
(`django_q/brokers/redis_broker.py`, from the pinned `django-q==1.3.9` in the container):

```python
class Redis(Broker):
    def __init__(self, list_key: str = Conf.PREFIX):
        super(Redis, self).__init__(list_key=f"django_q:{list_key}:q")

    def enqueue(self, task):
        return self.connection.rpush(self.list_key, task)      # append -> WAITING grows

    def dequeue(self):
        task = self.connection.blpop(self.list_key, 1)         # pop -> WAITING shrinks, PROCESSING begins
        if task:
            return [(None, task[1])]

    def queue_size(self):
        return self.connection.llen(self.list_key)             # counts only WAITING (list length)
```

This is why `redis-cli LLEN django_q:paperless:q` and `broker.queue_size()` return the identical
number, why the queue drains FIFO (`rpush` tail-append + `blpop` head-pop), and why a task being
processed is counted by neither (it has been popped off the list and no `django_q_task` row exists yet).

**Cluster configuration** (`src/paperless/settings.py:449-457`):

```python
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": PAPERLESS_WORKER_RETRY,     # 1810
    "timeout": PAPERLESS_WORKER_TIMEOUT, # 1800
    "workers": TASK_WORKERS,             # floor(sqrt(cpu_count)) -> 11 observed
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
}
```

### A.3 — Django-Q 1.3.x documentation facts, each with its runtime confirmation

The queued-vs-running distinction and the result-storage model live inside the `django-q` dependency,
so they were first taken from the official Django-Q 1.3.x documentation (`django-q.readthedocs.io`) and
then **confirmed at runtime** in this investigation:

| Documentation-derived fact | Where it was confirmed at runtime |
|---|---|
| `async_task(func, …)` sends a task package to the broker and returns a tracking **task id** | Q3 — task ids observed on the queue (e.g. `3b3787f3a7e040f59fc1611112481358`) and in `Task.id` |
| A worker cluster started via `qcluster` is required for any task to run | Q1/Q4 — with `qcluster` down, the package sat in Redis (`LLEN=1`); it executed only once `qcluster` ran |
| `queue_size()` counts waiting tasks and **excludes** tasks being processed | Q4 — `queue_size()`/`LLEN` = 1 while WAITING, dropped to 0 while PROCESSING (still no row) |
| Redis broker: `enqueue`=`rpush`, `dequeue`=`blpop`, `queue_size`=`llen` on `django_q:<prefix>:q` | A.2 source + Q3/Q4 `LLEN`/`LRANGE` on `django_q:paperless:q` |
| Completed tasks are saved to the **`Task`** model (`django_q_task`) with `func`/`args`/`kwargs`/`result`/`started`/`stopped`/`time_taken`/`success` | Q5/Q6 — real rows shown with every field populated |
| **`Success`/`Failure`** are proxy models filtered on `success=True`/`success=False` | Q5/Q6 — `Success._meta.proxy=True` (base `Task`); counts `Success=3`, `Failure=1` |
| Failures are saved with the exception captured in `result` | Failure case — `dup_demo.pdf`: `success=False`, full traceback string in `Task.result` |
| The **`OrmQ`** "Queued tasks" admin is registered only when `Conf.ORM` or `Conf.TESTING` | Q6 — `OrmQ registered=False`, `Conf.ORM=None`, `Conf.TESTING=False` under the Redis broker |

### A.4 — Exact environment & invocation (canonical)

- **Image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` (Debian 11, **Python 3.9.23**, WorkingDir `/app`, full source @ `542221a38dff`), run detached as container `pngx`.
- **Broker/layer:** `redis-server 6.0.16`, `redis-server --daemonize yes --bind 127.0.0.1 --port 6379`; `PAPERLESS_REDIS=redis://localhost:6379`.
- **Migrate:** `cd /app/src && python3 manage.py migrate --no-input` (creates `django_q_task`/`django_q_schedule`/`django_q_ormq`; seeds 4 schedules).
- **Services (each backgrounded, per `docker/supervisord.conf`):** `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application`; `python3 manage.py document_consumer`; `python3 manage.py qcluster`.
- **Pinned versions confirmed in-image:** `django-q==1.3.9`, `django==4.0.4`, `redis==3.5.3`, `hiredis==2.0.0`, `channels==3.0.4`, `channels-redis==3.4.0`, `djangorestframework==3.13.1`, `watchdog==2.1.7`, `gunicorn==20.1.0`, `asgiref==3.5.0`.

---

*End of runtime-grounded investigation for paperless-ngx @ `542221a38dff`. Every behavioral claim above
is paired with the command that produced it and its unedited output; statements not directly observed
are labelled **(inferred)**. Background processing is Django-Q 1.3.9 over a Redis broker — not Celery.*
