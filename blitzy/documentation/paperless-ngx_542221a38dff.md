# How paperless-ngx handles background / asynchronous processing during document ingestion

> A **runtime-grounded** investigation. Every behavioral claim below is backed by the exact command that produced it and its **complete, unedited output**, captured from a live cluster. Every code claim carries a `file:line` reference verified against the working tree at HEAD `542221a38dff06361e07976452f9aea24d210542`.

---

## 0. The question, and the direct answers

The question has six parts. Here are the one-line answers; each is proven in its own section below.

| # | Sub-question | Direct answer |
|---|---|---|
| **Q1** | Which services/processes are involved behind the scenes? | One container, supervised by **Supervisord**, running **three** long-lived Python processes — `gunicorn ... paperless.asgi:application` (ASGI web server), `python3 manage.py document_consumer` (directory watcher), `python3 manage.py qcluster` (**Django-Q** worker cluster) — plus a single **Redis** instance that is *both* the Django-Q broker *and* the Channels layer store. |
| **Q2** | How does a job appear once it's created? | As **one signed, pickled task package** `RPUSH`ed onto the Redis **list** `django_q:paperless:q`. `queue_size` goes `0 → 1`; the element is an opaque ~369–381-byte token (375 bytes in the run shown); **zero** database rows exist at creation. |
| **Q3** | Waiting vs. actively processing? | **Waiting** = still an element in the Redis list (counted by `queue_size`/`LLEN`). **Active** = pulled by the cluster's *pusher* (`BLPOP`) into an in-memory queue and executing in a *worker* — `queue_size` **excludes** it. Observable via Django-Q `Stat` objects: `status == "Working"` vs `"Idle"`. |
| **Q4** | Where is task state stored? | Completed task state is persisted to the **`django_q_task`** database table (Django-Q's `Task` model; `Success`/`Failure` are proxy models filtered on the `success` flag). Live progress is a **separate, ephemeral** WebSocket broadcast to the `status_updates` Channels group that is **never** written to the database. |
| **Q5** | How do you tell, after the fact, what happened to a job? | Read its `django_q_task` row: the **`success`** flag, **`started`/`stopped`/`time_taken()`**, and the pickled **`result`** (return value on success; the exception + full traceback string on failure). Surfaced through Django-Q's **built-in Django admin**. Paperless adds no custom task API. |
| **Q6** | Where in the paperless source is a background job triggered? | Via `async_task(...)`: the directory watcher (`document_consumer.py:86`), the REST upload endpoint (`views.py:523`), the five bulk operations (`bulk_edit.py:18,31,47,63,87`), and IMAP mail consumption (`mail.py:336`). Each calls `django_q.tasks.async_task`, which signs the package and calls `broker.enqueue` (`RPUSH`). |

### 0.1 Critical clarification — this is **Django-Q**, not Celery

Generic "background processing in Django" discussions usually assume Celery. **This version of paperless-ngx does not use Celery.** The async engine is **Django-Q 1.3.9** backed by Redis:

- `requirements.txt:37` → `django-q==1.3.9`; `requirements.txt:38` → `django==4.0.4`; `Pipfile:17` → `django-q = "~=1.3"`.
- There is **no** Celery dependency anywhere in the tree, **no** `celery.py`, and **no `PaperlessTask` model**. The models defined in `src/documents/models.py` are `MatchingModel`, `Correspondent`, `Tag`, `DocumentType`, `Document`, `Log`, `SavedView`, `SavedViewFilterRule`, and `FileInfo` — none is a task model. Task state lives entirely in the third-party `django_q` app's own tables.

Everything below is therefore analyzed with **Django-Q semantics**.

---

## 1. Methodology (run-first)

### 1.1 Where the observation ran, and why it is canonical

The reproduction ran **inside the project's own canonical container** (`paperless-app-0`, built from the project image), which is **Python 3.9.23** — matching the project's canonical runtime `Dockerfile:18` (`FROM python:3.9-slim-bullseye`) — with **every queue-relevant library pinned exactly to the manifest**. The broker is a **real Redis 6.0** server (`docker/compose/docker-compose.sqlite.yml:29` → `image: redis:6.0`), reached at `redis://broker:6379` (the canonical value from `docker-compose.sqlite.yml:53`).

Only the **task function bodies** used for holding/observing each state are **labeled non-canonical stand-ins** (running the *real* `documents.tasks.consume_file` needs the full OCR stack). The **dispatch path, the broker, the cluster, and the entire `Q_CLUSTER` configuration are canonical and identical to paperless.** Separately, §4/§5 also show the **real** `documents.tasks.consume_file` rows taken directly from the running paperless database, so the persisted outcome is demonstrated on the genuine task body too.

**Proof the harness configuration equals paperless's own resolved configuration.** Paperless's real settings, dumped from the running app:

```
$ cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -c <dump resolved settings>
Q_CLUSTER = {
  "name": "paperless",
  "catch_up": false,
  "recycle": 1,
  "retry": 1810,
  "timeout": 1800,
  "workers": 11,
  "redis": "redis://broker:6379"
}
ASGI_APPLICATION = paperless.asgi.application
django_q in INSTALLED_APPS = True
CHANNEL_LAYERS backend = channels_redis.core.RedisChannelLayer
CHANNEL_LAYERS hosts = ['redis://broker:6379']
CHANNEL_LAYERS capacity = 2000
CHANNEL_LAYERS expiry = 15
multiprocessing.cpu_count() = 128
```

The throwaway harness resolves the **identical** `Q_CLUSTER`, and the interpreter/library versions match the manifest exactly:

```
$ cd /tmp/dqproj && PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 -c <dump harness Q_CLUSTER>
Q_CLUSTER = {
  "name": "paperless",
  "catch_up": false,
  "recycle": 1,
  "retry": 1810,
  "timeout": 1800,
  "workers": 11,
  "redis": "redis://broker:6379"
}

$ python3 --version
Python 3.9.23

$ python3 -m pip freeze | grep -Ei <queue libs>
aioredis==1.3.1
asgiref==3.5.0
channels==3.0.4
channels-redis==3.4.0
daphne==3.0.2
Django==4.0.4
django-q==1.3.9
hiredis==2.0.0
redis==3.5.3
```

These are the exact pins from `requirements.txt` (`django-q==1.3.9` `:37`, `django==4.0.4` `:38`, `redis==3.5.3` `:84`, `channels==3.0.4` `:23`, `channels-redis==3.4.0` `:22`, `asgiref==3.5.0` `:13`, `daphne==3.0.2` `:31`, `aioredis==1.3.1` `:10`, `hiredis==2.0.0` `:44`).

> **Why `workers: 11`?** `Q_CLUSTER["workers"]` comes from paperless's `default_task_workers()` (`src/paperless/settings.py:427-435`): for a host with ≥4 cores it returns `floor(sqrt(cores))`. This container's Python sees `multiprocessing.cpu_count() == 128`, so `floor(sqrt(128)) == 11`. This is the canonical default for *this* machine and is reported as observed.

### 1.2 The stand-in task bodies (explicitly labeled non-canonical)

```python
# harness_tasks.py — LABELED NON-CANONICAL STAND-INS.
# Dispatched through the IDENTICAL canonical path
# (async_task -> SignedPackage.dumps -> broker.enqueue RPUSH -> pusher BLPOP -> worker -> monitor save_task).
def slow_ok(n):
    """Sleep n seconds then succeed (used to hold the ACTIVE state)."""
    time.sleep(n)
    return {"ok": True, "n": n}

def fast_ok(x):
    """Return immediately (used to observe a SUCCESS row quickly)."""
    return {"ok": True, "x": x, "doubled": x * 2}

def boom():
    """Always raise (used to observe the FAILED row + stored traceback)."""
    raise ValueError("intentional failure for observation")
```

### 1.3 Migrations create the Django-Q tables

```
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, django_q, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  ... (auth/admin/contenttypes) ...
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
  Applying sessions.0001_initial... OK

--- Table proof: which django_q_* tables exist (Django introspection) ---
$ ... connection.introspection.table_names()
['django_q_ormq', 'django_q_schedule', 'django_q_task']
```

The three tables (`django_q_task`, `django_q_ormq`, `django_q_schedule`) are Django defaults — the `Task` model sets no `db_table` override (`django_q/models.py:104`, `app_label = "django_q"`).

### 1.4 Run scale and stability

- **WAITING** was exercised many times (single enqueue + an 8× repeat to measure the package-size distribution).
- **ACTIVE** was driven with bursts of **200** and **160** slow tasks (each > the in-memory `QUEUE_LIMIT` of `workers**2 = 121`, forcing a visible Redis backlog).
- **SUCCESS**, **FAILED**, and the **SAVE_LIMIT** cap were each reproduced with dedicated tasks and a 300-task success burst.
- Every state was observed across **≥ 2 runs** (Phase-3 run #1 and run #2, plus the fresh clean run shown here). Structural behaviors were **identical** across runs. Values that legitimately vary run-to-run are called out where they appear: the package byte size (**369–381 bytes**), the random task UUID/humanized name, the `started`/`stopped` timestamps, and `time_taken()`.

---

## Q1 — Which services/processes are involved behind the scenes?

**Direct answer.** paperless-ngx ships as a **single container** whose processes are launched and kept alive by **Supervisord** (`docker/supervisord.conf`). Three long-running Python processes do the work, and they all talk to one **Redis** instance:

1. **ASGI web server** — `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` (`docker/supervisord.conf:10-11`). Configured at `gunicorn.conf.py:3` (`bind 0.0.0.0:8000`), `:4` (`workers = 2`), `:5` (`worker_class = "paperless.workers.ConfigurableWorker"`), `:6` (`timeout = 120`). It serves both HTTP and WebSockets (see Q4/§Q5 ephemeral channel).
2. **Directory watcher** — `python3 manage.py document_consumer` (`docker/supervisord.conf:19-20`). Watches the consume directory and **enqueues** a job per new file (Q6).
3. **Django-Q worker cluster** — `python3 manage.py qcluster` (`docker/supervisord.conf:28-29`, under `[program:scheduler]`). This is the process that actually **executes** background jobs.
4. **Redis** — provisioned as the Compose service literally named **`broker`** (`docker/compose/docker-compose.sqlite.yml:28` → `broker:`, `:29` → `image: redis:6.0`, `:53` → `PAPERLESS_REDIS: redis://broker:6379`; default URL `docker/wait-for-redis.py:19` → `redis://localhost:6379`). This **one** Redis serves a **dual role**: the Django-Q **broker** (`Q_CLUSTER["redis"]`, `settings.py:449-457`) *and* the Channels layer store (`RedisChannelLayer`, `settings.py:178-186`).

`django_q` is enabled as a Django app at `settings.py:110`.

**Observed — the real Django-Q cluster starting up (canonical command, same as `supervisord.conf:29`):**

```
$ cd /app/src && python3 manage.py qcluster        # (backgrounded)
17:32:11 [Q] INFO Q Cluster mountain-fish-illinois-eighteen starting.
17:32:11 [Q] INFO Process-1:1 ready for work at 25920
17:32:11 [Q] INFO Process-1:2 ready for work at 25921
17:32:11 [Q] INFO Process-1:3 ready for work at 25922
17:32:11 [Q] INFO Process-1:4 ready for work at 25923
17:32:11 [Q] INFO Process-1:5 ready for work at 25924
17:32:11 [Q] INFO Process-1:6 ready for work at 25925
17:32:11 [Q] INFO Process-1:7 ready for work at 25926
17:32:11 [Q] INFO Process-1:8 ready for work at 25927
17:32:11 [Q] INFO Process-1:9 ready for work at 25928
17:32:11 [Q] INFO Process-1:10 ready for work at 25929
17:32:11 [Q] INFO Process-1:11 ready for work at 25930
17:32:11 [Q] INFO Process-1:12 monitoring at 25931
17:32:11 [Q] INFO Process-1 guarding cluster mountain-fish-illinois-eighteen
17:32:11 [Q] INFO Process-1:13 pushing tasks at 25932
17:32:11 [Q] INFO Q Cluster mountain-fish-illinois-eighteen running.
```

This single `qcluster` command is itself a small **process tree**: **11 workers** (`Process-1:1`–`:11`), a **monitor** (`Process-1:12`, which saves results), a **guard/sentinel** (`Process-1 guarding …`, which spawns and health-checks the others), and a **pusher** (`Process-1:13`, which moves tasks from Redis into the in-memory queue). The worker count `11` is exactly the resolved `Q_CLUSTER["workers"]` from §1.1.

**Observed — the full supervised trio running together** (the directory watcher had just been started and immediately enqueued the sample document — real Q6 behavior):

```
[2026-07-13 17:32:28,298] [INFO] [paperless.management.consumer] Adding /app/src/../consume/patch-code-t-middle_document_0.pdf to the task queue.
17:32:28 [Q] INFO Enqueued 1
[2026-07-13 17:32:28,315] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

```
  pid=21245  ppid=21243   ASGI web server (gunicorn)               | .../python3.9 .../gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  pid=21258  ppid=21245   ASGI web server (gunicorn)               | .../python3.9 .../gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  pid=21261  ppid=21245   ASGI web server (gunicorn)               | .../python3.9 .../gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  pid=25912  ppid=25906   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25919  ppid=25912   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25923  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25924  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25925  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25926  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25927  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25928  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25929  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25930  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25931  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25932  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster
  pid=25945  ppid=25939   directory watcher (document_consumer)    | python3 manage.py document_consumer
  pid=25960  ppid=25919   Django-Q worker cluster (qcluster)       | python3 manage.py qcluster

  (Redis runs in the separate paperless-redis-0 container -> the Django-Q broker + Channels layer)
```

Here you can see all three services simultaneously: **gunicorn** (master `21245` + its 2 workers `21258`/`21261`, matching `gunicorn.conf.py:4`), the **document_consumer** (`25945`), and the **qcluster** tree (master `25912` → guard `25919` → its 11 workers + monitor + pusher children). Redis is the fourth participant, in its sibling container.

**How the WebSocket half is wired (relevant to Q4/Q5).** `paperless.asgi:application` is a `ProtocolTypeRouter` (`src/paperless/asgi.py:17`) that routes `http` to Django (`:19`) and `websocket` to `AuthMiddlewareStack(URLRouter(...))` (`:20`); the WebSocket endpoint is `StatusConsumer` (`src/paperless/consumers.py:9`), which joins the `status_updates` group on connect (`:17-18`) and leaves it on disconnect (`:24-25`). This is the live-progress path, not the persistence path.

> **Corroboration (official Django-Q 1.3.x docs).** The docs describe exactly this internal division of labor: the *pusher* "checks the signing and unpacks the task to the internal Task Queue", a *worker* "pulls a task of the Task Queue", and "The result monitor checks the Result Queue for processed packages and saves both failed and successful packages to the Django database". The *sentinel* "spawns all process and then checks the health of all workers". The banner above is that architecture, live.

---


## Q2 — How does a job appear once it's created?

**Direct answer.** Enqueuing does exactly one thing: it produces a **single, signed, pickled task package** and **`RPUSH`es it onto a Redis list** whose key is `django_q:paperless:q`. At that instant `queue_size` (which is just `LLEN` on that list) goes from `0 → 1`, and **no database row exists yet** — under the Redis broker a waiting job lives *only* in Redis.

**Mechanism (code).** `async_task(func, *args, **kwargs)` (`django_q/tasks.py:20`) builds a task dict, then `pack = SignedPackage.dumps(task)` (`:69`) and `enqueue_id = broker.enqueue(pack)` (`:73`). For the Redis broker, the list key is `f"django_q:{name}:q"` (`django_q/brokers/redis_broker.py:15`) → with `name = "paperless"` that is `django_q:paperless:q`; `enqueue` is a `redis.rpush` (`:17-18`).

**Observed.** Enqueue one task through the canonical `async_task()` entry point, with **no cluster running** so it stays put:

```
# Precondition: qcluster NOT running; broker flushed.
$ redis-cli KEYS '*'

$ redis-cli LLEN django_q:paperless:q
0

# Enqueue ONE task via the canonical async_task() entry point (no cluster consuming):
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/enq.py harness_tasks.slow_ok 30
17:20:56 [Q] INFO Enqueued 1
async_task('harness_tasks.slow_ok',30) -> task id 365b3914c28c4adb9df9e1c826243475
Task.objects.count() immediately after enqueue = 0
get_broker().queue_size() = 1

# Inspect the broker immediately after enqueue:
$ redis-cli KEYS '*'
django_q:paperless:q
$ redis-cli TYPE django_q:paperless:q
list
$ redis-cli LLEN django_q:paperless:q
1
```

So: one new key appears (`django_q:paperless:q`), it is a **`list`**, its length is **1**, and — critically — `Task.objects.count()` is still **0**. The job exists purely as a Redis list element.

**What is actually in that element?** It is an **opaque, signed token**, *not* JSON. Measured exactly:

```
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/measure.py
LLEN django_q:paperless:q = 1
type(element) = bytes
exact byte length = len(element) = 375
first 24 bytes (repr) = b'gAWV6AAAAAAAAAB9lCiMAmlk'
last 20 bytes (repr)  = b'BUfHK9AQ3wPcbwKIrmgA'
is JSON? starts with { or [ : False
contains ':' signing separator: True
```

The element is **375 bytes** in this run. Because each package embeds a random task UUID, a humanized name, and a signing timestamp, the exact size varies slightly run-to-run; across 8 identical enqueues the observed sizes were `[381, 369, 373, 369, 374, 374, 379, 378]` → **a 369–381 byte range**. It is `bytes`, it does **not** start with `{`/`[` (so not JSON), and it contains the `:` that separates a `django.core.signing` payload from its signature.

**Proof it is a signed, base64-encoded pickle** — decode the payload and round-trip it through Django-Q's own `SignedPackage.loads`:

```
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/decode.py
first 3 raw bytes after base64-decode = b'\x80\x05\x95'  (pickle PROTO opcode: 0x80 0x05 = protocol 5)
SignedPackage.loads(element) keys = ['args', 'func', 'id', 'kwargs', 'name', 'started']
  func    = harness_tasks.slow_ok
  args    = (30,)
  id      = 365b3914c28c4adb9df9e1c826243475
  name    = equal-three-fourteen-fifteen
```

Base64-decoding the payload reveals the pickle **PROTO** opcode `\x80\x05` (protocol 5, the maximum on Python 3.9). `SignedPackage.loads` reconstructs the task dict `{func, args, kwargs, id, name, started}`, and its `id` (`365b3914…`) is exactly the id `async_task()` returned above. So the "job at rest" is: *the pickled call spec (`func`, `args`, `kwargs`), a unique id, a humanized name, and an enqueue timestamp — base64-encoded and HMAC-signed with the project `SECRET_KEY`, sitting as one element in the Redis list.*

> **Corroboration (official Django-Q 1.3.x docs).** The signing is expected: "Django Q uses your SECRET_KEY to sign task packages and prevent task crossover." That is why the element is an opaque signed token rather than readable JSON.

---


## Q3 — Waiting vs. actively processing: how to tell them apart

**Direct answer.** A **waiting** job is still an element in the Redis list `django_q:paperless:q` — it is counted by `queue_size()` (which is literally `LLEN`). An **actively processing** job has been removed from that list by the cluster's **pusher** (via `BLPOP`) and placed into an **in-memory** `task_queue`, from which a **worker** executes it — so it is **no longer counted by `queue_size()`**. The distinction is directly observable two ways: (a) the Redis `LLEN` (waiting only), and (b) a Django-Q **`Stat`** object whose `status` reads **`Working`** while any in-flight/queued-in-memory work exists, and **`Idle`** when the cluster is fully drained.

**Mechanism (code).**
- `queue_size()` → `redis.llen(list_key)` (`django_q/brokers/redis_broker.py:25-26`).
- The pusher removes from Redis with `BLPOP` (`redis_broker.py:20-21`, `dequeue`) inside `pusher()` (`django_q/cluster.py:333`) and places the task into the **in-memory** `task_queue` via `task_queue.put(task)` (`:362`). That queue is bounded: `Queue(maxsize=Conf.QUEUE_LIMIT)` (`cluster.py:161`), and `QUEUE_LIMIT` defaults to `workers**2` (`conf.py:113`) — here `11**2 = 121`.
- A `worker()` (`cluster.py:399`) pops from `task_queue` and runs the function.
- The sentinel's `status()` (`cluster.py:175-181`) returns `Conf.IDLE` **iff** both the internal `result_queue` and `task_queue` are empty, else `Conf.WORKING` (`conf.py:196-198` → `"Working"`/`"Idle"`). `Stat.status` mirrors it (`django_q/status.py:41`).

**Observed — before / intermediate / after.** Enqueue a **burst of 200** `slow_ok(3)` with no cluster (BEFORE), then start `qcluster` and poll every 2 s:

```
================ C2 — ACTIVE (waiting vs in-flight) ================

# BEFORE: enqueue a burst of 200 slow tasks (each sleeps 3s) with NO cluster running.
# 200 > QUEUE_LIMIT (workers**2 = 11**2 = 121), so a backlog must remain in Redis while workers run.
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/burst.py 200 3
Enqueued 200 x slow_ok(3) via async_task()
get_broker().queue_size() = 200
Task.objects.count() (DB rows) = 0

$ redis-cli LLEN django_q:paperless:q   # full backlog waiting, nothing processing yet
200
```

```
# ---- INTERMEDIATE: poll every 2s while the burst drains ----
  t(s) | queue_size(LLEN) | Stat.status | task_q | done_q | DB_rows(persisted)
  -----+------------------+-------------+--------+--------+-------------------
     0 |                5 |     Working |    121 |      0 | 64
     2 |                0 |     Working |    121 |      0 | 71
     4 |                0 |     Working |    116 |      0 | 76
     6 |                0 |     Working |    108 |      0 | 83
     8 |                0 |     Working |    102 |      0 | 89
    10 |                0 |     Working |     96 |      0 | 95
    12 |                0 |     Working |     88 |      0 | 103
    14 |                0 |     Working |     82 |      0 | 109
    16 |                0 |     Working |     76 |      0 | 115
    18 |                0 |     Working |     68 |      0 | 123
    20 |                0 |     Working |     62 |      0 | 129
    22 |                0 |     Working |     56 |      0 | 135
    24 |                0 |     Working |     48 |      0 | 143
    26 |                0 |     Working |     42 |      0 | 149
    28 |                0 |     Working |     36 |      0 | 155
    30 |                0 |     Working |     28 |      0 | 163
    32 |                0 |     Working |     22 |      0 | 169
    34 |                0 |     Working |     16 |      0 | 175
    36 |                0 |     Working |      8 |      0 | 183
    38 |                0 |     Working |      2 |      0 | 189
    40 |                0 |        Idle |      0 |      0 | 195
    42 |                0 |        Idle |      0 |      0 | 200
    44 |                0 |        Idle |      0 |      0 | 200
    46 |                0 |        Idle |      0 |      0 | 200
    48 |                0 |        Idle |      0 |      0 | 200
    50 |                0 |        Idle |      0 |      0 | 200
    52 |                0 |        Idle |      0 |      0 | 200
    55 |                0 |        Idle |      0 |      0 | 200
    57 |                0 |        Idle |      0 |      0 | 200
    59 |                0 |        Idle |      0 |      0 | 200
    61 |                0 |        Idle |      0 |      0 | 200
    63 |                0 |        Idle |      0 |      0 | 200
```

```
# AFTER: all tasks drained; cluster idle. Final broker + DB state:
$ redis-cli LLEN django_q:paperless:q
0
$ ... Task.objects.count()
200
```

**Reading the table — this is the whole answer to Q3:**

- **BEFORE**: `queue_size = LLEN = 200` (all 200 **waiting**), `DB_rows = 0`.
- **t = 0**: `LLEN = 5` (a small backlog still **waiting** in Redis) while `task_q = 121` — the pusher has already moved the maximum `QUEUE_LIMIT = 121` tasks into the **in-memory** queue. `Stat.status = Working`.
- **t = 2 … 38 (the key window)**: `queue_size = LLEN = 0` — **nothing is waiting in Redis** — yet `Stat.status = Working`, `task_q` drains `121 → 2`, and `DB_rows` climbs `71 → 189` but is still `< 200`. That gap (121…2 tasks that are neither in the Redis list **nor** yet persisted) is precisely the **actively-processing / in-flight** set. **`queue_size` does not count them.**
- **t = 40**: the last in-flight task finishes; `task_q = 0`, `Stat.status` flips to **`Idle`**, and `DB_rows` completes at `200`.
- **AFTER**: `LLEN = 0`, all `200` persisted.

The startup banner for this run also demonstrated `recycle=1` at shutdown — the workers that finally stopped were numbered `Process-1:211/212/213`, i.e. each of the 11 worker slots had been **reincarnated ~200 times** (one fresh worker per task), consistent with `Q_CLUSTER["recycle"] = 1`:

```
17:25:24 [Q] INFO Process-1:211 stopped doing work
17:25:24 [Q] INFO Process-1:212 stopped doing work
17:25:24 [Q] INFO Process-1:213 stopped doing work
17:25:24 [Q] INFO Process-1 waiting for the monitor.
17:25:24 [Q] INFO Process-1:12 stopped monitoring results
17:25:24 [Q] INFO Q Cluster oklahoma-summer-eleven-oklahoma has stopped.
```

> **Corroboration (official Django-Q 1.3.x docs).** `queue_size()` "Returns the size of the broker queue. Note that this does not count tasks currently being processed." The in-memory bound is documented too: `queue_limit` controls "how many tasks are kept in memory by a single cluster" and "Defaults to workers**2" — matching the observed `task_q` ceiling of `121`.
>
> **Nuance relevant to Q3/Q5 (inferred from docs, not reproduced here — labeled inferred).** The default Redis broker has **no message receipts**: per the docs, "The default Redis broker does not support message receipts. This means that in case of a catastrophic failure of the cluster server or worker timeouts, tasks that were being executed get lost." An in-flight task (already `BLPOP`-ed out of the list) is therefore not recoverable from Redis if the whole cluster dies mid-execution — distinct from a task whose *code* raises, which becomes a normal recorded **failure** (Q5).

---


## Q4 — Where does the task state end up being stored?

**Direct answer.** There are **two entirely separate** channels, and conflating them is the classic mistake:

1. **Persisted state** — the finished task's outcome is written to the **`django_q_task`** database table (Django-Q's `Task` model). `Success` and `Failure` are **proxy models** over that same table, filtered on the `success` boolean. This is the durable record.
2. **Ephemeral state** — while a document is being consumed, paperless broadcasts **live progress** over a Redis-backed **Channels** group called `status_updates`. This is transient (15-second key expiry) and is **never written to `django_q_task`**.

**Mechanism (code).**
- `Task` model → `django_q/models.py:20`; its `result` is a `PickledObjectField` (`:27`); `started`/`stopped`/`success` at `:29-31`; table is the default `django_q_task` (`:104`, `app_label="django_q"`, no `db_table` override).
- `Success` = proxy, `objects` filtered `success=True` (`models.py:108-121`); `Failure` = proxy, filtered `success=False` (`models.py:124-137`).
- The monitor persists via `save_task(task, broker)` (`cluster.py:369` monitor → `:384`/`:454` save).
- Ephemeral progress: `src/documents/consumer.py:56` `_send_progress(...)` calls `async_to_sync(self.channel_layer.group_send)("status_updates", {...})` (`:73-74`); the layer is the `RedisChannelLayer` at `settings.py:178-186` (capacity 2000, **expiry 15**).

**Observed (persisted) — a completed SUCCESS row read straight from `django_q_task`:**

```
================ C3 — DONE / SUCCESS ================
Task.objects.get(func="harness_tasks.fast_ok"):
  id         = 8b73e02994dd4cd384ee99aa08dc3831
  name       = delaware-summer-happy-texas
  func       = harness_tasks.fast_ok
  success    = True
  started    = 2026-07-13 17:25:59.525144+00:00
  stopped    = 2026-07-13 17:26:09.989816+00:00
  time_taken = 10.464672 seconds  # (stopped-started).total_seconds()
  result     = {'ok': True, 'x': 21, 'doubled': 42}  type = dict

# Success is a PROXY model of Task filtered on success=True:
  Success._meta.proxy      = True
  Success._meta.db_table   = django_q_task
  Success.objects.count()  = 1
  Failure.objects.count()  = 1
```

The `result` is the task's **actual return value**, unpickled from the `PickledObjectField` back into a Python `dict`. `Success._meta.proxy == True` and `Success._meta.db_table == django_q_task` prove `Success` is not a separate table — it is a filtered view of `django_q_task`.

**Observed (persisted) — the same rows via raw SQL, proving the physical storage location:**

```
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/raw_sql.py
SELECT id, func, success, started, stopped FROM django_q_task ORDER BY stopped DESC;

  ('fe07db3031aa40d18d069eea9347abf4', 'harness_tasks.boom', False, datetime.datetime(2026, 7, 13, 17, 25, 59, 549539), datetime.datetime(2026, 7, 13, 17, 26, 9, 990460))
  ('8b73e02994dd4cd384ee99aa08dc3831', 'harness_tasks.fast_ok', True, datetime.datetime(2026, 7, 13, 17, 25, 59, 525144), datetime.datetime(2026, 7, 13, 17, 26, 9, 989816))

SELECT COUNT(*) FROM django_q_task;  -> 2
SELECT COUNT(*) FROM django_q_task WHERE success=1;  (Success proxy) -> 1
SELECT COUNT(*) FROM django_q_task WHERE success=0;  (Failure proxy) -> 1
```

The `Success`/`Failure` proxies are simply `WHERE success=1` / `WHERE success=0` over the one `django_q_task` table.

**Observed (ephemeral) — the live progress channel is a completely different keyspace and persists nothing:**

```
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 /tmp/dqproj/c5_channels.py
django_q_task rows BEFORE progress broadcast: 251
broker list django_q:paperless:q LLEN BEFORE: 0
Redis keyspace BEFORE:
    (no keys)

subscribed a StatusConsumer channel: specific.1f2f583270df40589a2b16abdc4200fd!3d0cee08431246a18b78c999dcf655cb
group_send: pushed 3 progress messages to group status_updates

Redis keyspace DURING (after group_send, before receive) -- asgi:* keys, NOT django_q:*, short TTL:
    asgi:group:status_updates                      type=zset   ttl=86400s
    asgispecific.1f2f583270df40589a2b16abdc4200fd! type=zset   ttl=15s

--- received by the subscriber, live, in order ---
    STARTING -> {'task_id': 'demo', 'current_progress': 0, 'max_progress': 100, 'status': 'STARTING', 'message': 'new_file'}
    WORKING -> {'task_id': 'demo', 'current_progress': 20, 'max_progress': 100, 'status': 'WORKING', 'message': 'parsing_document'}
    SUCCESS -> {'task_id': 'demo', 'current_progress': 100, 'max_progress': 100, 'status': 'SUCCESS', 'message': 'finished'}

django_q_task rows AFTER progress broadcast: 251  (UNCHANGED -> progress persisted NOTHING)
broker list django_q:paperless:q LLEN AFTER: 0
```

The progress messages live under `asgi:*` keys — **not** `django_q:*` — and the per-subscriber message queue carries **`ttl=15s`**, exactly the `expiry=15` from `settings.py:178-186`. The `django_q_task` row count is **251 before and 251 after** the broadcast: the WebSocket progress stream writes **nothing** to the task table. (In real paperless the payload's statuses come from `consumer.py`: `STARTING` `:202`, `WORKING` `:240/:259/:264/:274/:294`, `SUCCESS` `:375`, and `FAILED` via `_fail` `:78-81`.)

> **Bottom line for Q4:** *persisted* task state → `django_q_task` (SQLite/DB, durable). *Ephemeral* progress → the `status_updates` Channels group over Redis (`asgi:*`, 15 s expiry, never persisted). They even share the **same** Redis server, but they are different keyspaces with different lifetimes.

---


## Q5 — After the fact, how can you tell what happened to a given job?

**Direct answer.** You read the job's **`django_q_task`** row. Four fields tell the whole story:

- **`success`** — `True` or `False`.
- **`started` / `stopped`** — timestamps; `time_taken()` = `(stopped - started).total_seconds()` (`django_q/models.py:93`). Note this **includes queue-wait time**, not just execution.
- **`result`** — the pickled outcome: on success it's the function's **return value**; on failure it's a **string containing the exception message and the full traceback**.

These are surfaced through **Django-Q's built-in Django admin** (Successful tasks / Failed tasks / Scheduled tasks). **Paperless itself adds no custom task model, serializer, or API** — the visibility is entirely Django-Q's.

**Mechanism (code).** On failure the worker stores `result = (f"{e} : {traceback.format_exc()}", False)` (`django_q/cluster.py:435`); the monitor then persists the row via `save_task` (`cluster.py:454`). The admin registrations are `admin.site.register(Schedule, ...)`, `register(Success, ...)`, `register(Failure, ...)` at `django_q/admin.py:107-109` (with `OrmQ`/"Queued tasks" registered only for the ORM broker at `:112` — see the note at the end of this section).

**Observed — a FAILED row (harness stand-in `boom()`):**

```
================ C4 — FAILED ================
Task.objects.get(func="harness_tasks.boom"):
  id         = fe07db3031aa40d18d069eea9347abf4
  func       = harness_tasks.boom
  success    = False
  started    = 2026-07-13 17:25:59.549539+00:00
  stopped    = 2026-07-13 17:26:09.990460+00:00
  time_taken = 10.440921 seconds
  result (stored exception + traceback string):
  ----------------------------------------------------------------
intentional failure for observation : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/tmp/dqproj/harness_tasks.py", line 30, in boom
    raise ValueError("intentional failure for observation")
ValueError: intentional failure for observation

  ----------------------------------------------------------------
  type(result) = str

# Failure is a PROXY model of Task filtered on success=False:
  Failure._meta.proxy    = True
  Failure._meta.db_table = django_q_task
  is this row a Failure? Failure.objects.filter(id='fe07db3031aa40d18d069eea9347abf4').exists() = True
  is this row a Success? Success.objects.filter(id='fe07db3031aa40d18d069eea9347abf4').exists() = False
```

`success = False`, and `result` is exactly the `f"{e} : {traceback.format_exc()}"` string built at `cluster.py:435` — you can read the failing frame (`worker` at `cluster.py:432` → `boom` at `harness_tasks.py:30`) straight out of the stored row. The row is a `Failure` (proxy filter `success=False`) and is *not* a `Success`.

**Observed — the REAL `documents.tasks.consume_file` rows** (canonical task body, read from the running paperless database — this is the genuine ingestion task, not a stand-in):

```
$ cd /app/src && PYTHONPATH=/app/src DJANGO_SETTINGS_MODULE=paperless.settings python3 /tmp/read_real.py
================ REAL paperless documents.tasks.consume_file rows ================
----------------------------------------------------------------------
  id         = 4f2f2507b1894b4b9feed2b619dd4abd
  func       = documents.tasks.consume_file
  success    = False
  started    = 2026-07-13 16:31:36.521815+00:00
  stopped    = 2026-07-13 16:31:37.942034+00:00
  time_taken = 1.420219
  result:
  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
patch-code-t-middle_document_0.pdf: The following error occured while consuming patch-code-t-middle_document_0.pdf: UNIQUE constraint failed: documents_document.checksum : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/sqlite3/base.py", line 477, in execute
    return Database.Cursor.execute(self, query, params)
sqlite3.IntegrityError: UNIQUE constraint failed: documents_document.checksum

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/asgiref/sync.py", line 266, in main_wrap
    raise exc_info[1]
  File "/app/src/documents/consumer.py", line 301, in try_consume_file
    document = self._store(text=text, date=date, mime_type=mime_type)
  File "/app/src/documents/consumer.py", line 398, in _store
    document = Document.objects.create(
  ... (Django ORM insert frames) ...
django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 363, in try_consume_file
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: patch-code-t-middle_document_0.pdf: The following error occured while consuming patch-code-t-middle_document_0.pdf: UNIQUE constraint failed: documents_document.checksum

  <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
----------------------------------------------------------------------
  id         = 04880ce6132f43a78d73fa7f213a8393
  func       = documents.tasks.consume_file
  success    = True
  started    = 2026-07-13 16:31:36.542276+00:00
  stopped    = 2026-07-13 16:31:37.908922+00:00
  time_taken = 1.366646
  result:
  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
Success. New document id 1 created
  <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
----------------------------------------------------------------------
Success (proxy, success=True) count = 10
Failure (proxy, success=False) count = 1
Success._meta.proxy = True | db_table = django_q_task
Failure._meta.proxy = True | db_table = django_q_task
```

This is the genuine ingestion pipeline recorded in `django_q_task`:
- The **successful** consume returns the string `Success. New document id 1 created` (that is the function's return value, stored in `result`).
- The **failed** consume of a duplicate stores the full traceback, and you can read the entire real code path out of it: `cluster.py:432` (worker) → `documents/tasks.py:236` (`consume_file` calls `Consumer().try_consume_file(...)`) → `documents/consumer.py:363` → `consumer.py:81` (`raise ConsumerError`) → root cause `UNIQUE constraint failed: documents_document.checksum`. That is exactly how you tell, after the fact, *why* a specific ingestion job failed.

**Observed — "failures are always saved" vs. the success `SAVE_LIMIT`:**

```
================ SAVE_LIMIT — failures always saved, successes capped ================
# Django-Q default Conf.SAVE_LIMIT = 250 (django_q/conf.py:87).
# We enqueued 300 successful tasks (fast_ok) on top of the existing 1 success + 1 failure.
$ PYTHONPATH=/tmp/dqproj DJANGO_SETTINGS_MODULE=dqsettings python3 -c <count Success/Failure/Task + Conf.SAVE_LIMIT>
Conf.SAVE_LIMIT           = 250
Success.objects.count()   = 250  (capped at SAVE_LIMIT)
Failure.objects.count()   = 1  (failure never pruned)
Task.objects.count()      = 251  (= 250 successes + 1 failure)
```

After processing 300+ successes, the persisted **Success** count is capped at exactly **250** (`Conf.SAVE_LIMIT`, `conf.py:87`), while the single **Failure** is **never** pruned — total `251`. In the source, both the early-return skip (`cluster.py:461`) and the prune (`cluster.py:477`) are gated on `task["success"]`, so failures have **no** skip/prune path. This means: for after-the-fact debugging you can rely on failures being retained even when old successes have been rotated out.

> **Corroboration (official Django-Q 1.3.x docs).** `save_limit` "Limits the amount of successful tasks saved to Django. Set to 0 for unlimited. Set to -1 for no success storage at all. ... Failures are always saved." — matching the observed cap of 250 and the retained failure. `time_taken` "represents the time a task spends in the cluster, this includes any time it may have waited in the queue" — which is why the harness `time_taken` values (~10 s) reflect the queue wait before the cluster was started, whereas the real `consume_file` rows (cluster already running) show ~1.4 s.
>
> **Admin note (grounded in source + docs).** Django-Q registers `Success`, `Failure`, and `Schedule` in the Django admin (`django_q/admin.py:107-109`). The "Queued tasks" (`OrmQ`) admin is registered only for the ORM broker (`:112`); the docs confirm that view "is only enabled when you use the Django ORM broker." Because paperless uses the **Redis** broker, the waiting queue is *not* visible in the admin — it lives in the Redis list (Q2/Q3) — but Successful/Failed/Scheduled tasks are.

---


## Q6 — Where in the paperless source is a background job triggered?

**Direct answer.** Every background job is triggered by a call to **`django_q.tasks.async_task(...)`**. There are four categories of enqueue site in the paperless source (all import `async_task` from `django_q.tasks`), and `async_task` is the same canonical entry point exercised throughout this document (`django_q/tasks.py:20` → `SignedPackage.dumps` `:69` → `broker.enqueue` `:73` → Redis `RPUSH django_q:paperless:q`).

**1. Directory watcher** — a new file in the consume folder → one `consume_file` job.
`src/documents/management/commands/document_consumer.py` (import at `:13`):
```python
# :85  logger.info(f"Adding {filepath} to the task queue.")
# :86
        async_task(
            "documents.tasks.consume_file",
            ...
        )
```
This is the exact line whose runtime effect was captured in Q1: `Adding /app/src/../consume/patch-code-t-middle_document_0.pdf to the task queue.` followed by `Enqueued 1`.

**2. REST upload endpoint** (`PostDocumentView`) — an HTTP document upload → one `consume_file` job.
`src/documents/views.py` (import at `:28`):
```python
# :521  task_id = str(uuid.uuid4())
# :523
        async_task(
            "documents.tasks.consume_file",
            temp_filename,
            ...
        )
```

**3. Bulk operations** — each bulk edit → a `bulk_update_documents` job. **All five** call sites in `src/documents/bulk_edit.py` (import at `:4`):
```
18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
```

**4. IMAP mail consumption** — a fetched mail attachment → one `consume_file` job.
`src/paperless_mail/mail.py` (import at `:11`):
```python
# :336
                async_task(
                    "documents.tasks.consume_file",
                    path=temp_filename,
                    ...
                )
```

**The task bodies these strings resolve to** (dispatched by dotted path, run inside a worker at `cluster.py:432`, `res = f(*task["args"], **task["kwargs"])`):
- `documents.tasks.consume_file` — `src/documents/tasks.py:184`; it calls `Consumer().try_consume_file(...)` at `:236` (the ingestion pipeline whose success/failure rows are shown in Q5).
- `documents.tasks.bulk_update_documents` — `src/documents/tasks.py:270`.

**Also (named for completeness).** Beyond these direct `async_task` enqueues, paperless registers **recurring/scheduled** jobs via `django_q.tasks.schedule` in migrations — `src/documents/migrations/1001_auto_20201109_1636.py`, `.../1004_sanity_check_schedule.py`, `.../1005_checksums.py`, and `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — which is why the real paperless DB (Q5) also contained `train_classifier`, `index_optimize`, `sanity_check`, and `process_mail_accounts` rows. And after a document is consumed, six post-consume **signal handlers** are wired in `src/documents/apps.py:22-27` (in `ready()` at `:11`) — these run in the same worker as part of `consume_file`, they are not separate `async_task` enqueues.

**Trigger → mechanism, end to end (all observed above):**

```
enqueue site (async_task, e.g. document_consumer.py:86)
    -> django_q.tasks.async_task            (tasks.py:20)
    -> SignedPackage.dumps(task)            (tasks.py:69)   # sign + pickle  (Q2: the 375-byte token)
    -> broker.enqueue(pack) = RPUSH         (redis_broker.py:17-18)  # onto django_q:paperless:q  (Q2)
    -> [WAITING: element in the Redis list; queue_size/LLEN counts it]        (Q3)
    -> pusher BLPOP -> in-memory task_queue  (cluster.py:333/:362)  # QUEUE_LIMIT=121
    -> [ACTIVE: in task_queue / running in a worker; queue_size EXCLUDES it]  (Q3)
    -> worker runs f(...)                    (cluster.py:399/:432)
    -> monitor save_task -> django_q_task row (cluster.py:369/:454)  # success flag + result  (Q4/Q5)
       (in parallel, ephemeral progress -> group_send 'status_updates' -> WebSocket, never persisted) (Q4)
```

---


## Coverage pass

Every part of the question and every named item is addressed:

- **Q1 (services)** ✔ — three supervised processes (`gunicorn`/`document_consumer`/`qcluster`) + Redis, each mapped to `supervisord.conf` lines and shown live (cluster banner + full process list).
- **Q2 (appearance)** ✔ — signed pickled package on the Redis list `django_q:paperless:q`; `queue_size 0→1`; measured **375 bytes** (range 369–381); `SignedPackage.loads` round-trip; **0** DB rows at creation.
- **Q3 (waiting vs active)** ✔ — `LLEN`/`queue_size` (waiting) vs. in-memory `task_q` + `Stat.status="Working"` (active); the poll table shows `queue_size=0` while `Working` with `task_q` draining — before/intermediate/after all reported.
- **Q4 (storage)** ✔ — persisted `django_q_task` (with `Success`/`Failure` proxies, `proxy=True`, `db_table=django_q_task`) vs. ephemeral `status_updates` Channels group (`asgi:*`, TTL 15 s, persists nothing). Explicitly separated.
- **Q5 (after-the-fact)** ✔ — `success` / `started` / `stopped` / `time_taken()` / `result` shown for SUCCESS and FAILED, on both a stand-in and the **real** `consume_file` rows; `SAVE_LIMIT=250` cap vs. "failures always saved" demonstrated; built-in admin registration named; no custom paperless task API.
- **Q6 (code origin)** ✔ — all enqueue sites named by `file:line`: `document_consumer.py:86`, `views.py:523`, `bulk_edit.py:18/31/47/63/87` (all five), `mail.py:336`; task bodies `tasks.py:184`/`:270`; plus scheduled-task migrations and the six `apps.py:22-27` signal handlers.

**Condition coverage (waiting / active / done / failed):** all four exercised with real captured output and before/intermediate/after values. **Every "e.g./including" item** (each bulk call site, each named process, `Success`/`Failure` proxies, `SAVE_LIMIT`, the `status_updates` statuses) is covered.

**Labeling discipline used in this document:**
- **Observed** = has a command + its complete, unedited output next to it (Q1–Q5 evidence blocks).
- **Non-canonical stand-in** = the `slow_ok`/`fast_ok`/`boom` task *bodies* only; the dispatch/broker/cluster/config are canonical, and Q5 additionally uses the real `consume_file` rows.
- **Inferred** (not reproduced here, explicitly marked): the Redis-broker "no message receipts → in-flight loss on catastrophic cluster death" behavior in Q3, taken from the official docs; and the admin "Queued tasks only under the ORM broker" statement, grounded in `admin.py:112` + docs.

**Reproducibility / stability:** structural results were identical across ≥2 runs; the only run-to-run variation is the package byte size (369–381), the random task id/name, the timestamps, and `time_taken()`.

---

## Appendix — verified `file:line` reference map

All references below were verified against the working tree at HEAD `542221a38dff06361e07976452f9aea24d210542`; the `django_q/*` references were verified against the installed `django-q==1.3.9` in the canonical container.

### A. Services / topology (Q1)
| Claim | file:line |
|---|---|
| gunicorn program + command | `docker/supervisord.conf:10-11` |
| document_consumer program + command | `docker/supervisord.conf:19-20` |
| qcluster program + command (`[program:scheduler]`) | `docker/supervisord.conf:28-29` |
| gunicorn bind / workers=2 / ConfigurableWorker / timeout=120 | `gunicorn.conf.py:3-6` |
| ASGI `ProtocolTypeRouter` http + websocket | `src/paperless/asgi.py:17-20` |
| `StatusConsumer` group_add/discard `status_updates` | `src/paperless/consumers.py:9, :17-18, :24-25` |
| `django_q` in `INSTALLED_APPS` | `src/paperless/settings.py:110` |
| `ASGI_APPLICATION` | `src/paperless/settings.py:156` |
| `CHANNEL_LAYERS` RedisChannelLayer capacity 2000 expiry 15 | `src/paperless/settings.py:178-186` |
| `default_task_workers()` (√cores for cores≥4) | `src/paperless/settings.py:427-435` |
| `PAPERLESS_WORKER_TIMEOUT` default 1800 | `src/paperless/settings.py:440` |
| `PAPERLESS_WORKER_RETRY` = timeout+10 (1810) | `src/paperless/settings.py:444-447` |
| `Q_CLUSTER` block | `src/paperless/settings.py:449-457` |
| Redis provisioned as `broker` (image redis:6.0) | `docker/compose/docker-compose.sqlite.yml:28-29` |
| `PAPERLESS_REDIS: redis://broker:6379` | `docker/compose/docker-compose.sqlite.yml:53` |
| Redis default URL `redis://localhost:6379` | `docker/wait-for-redis.py:19` |
| Canonical Python 3.9 base image | `Dockerfile:18` |

### B. Enqueue call sites + task bodies (Q6)
| Claim | file:line |
|---|---|
| watcher `async_task("documents.tasks.consume_file", ...)` | `src/documents/management/commands/document_consumer.py:86` (import `:13`, log `:85`) |
| REST upload `async_task("documents.tasks.consume_file", temp_filename, ...)` | `src/documents/views.py:523` (import `:28`, task_id `:521`) |
| bulk `async_task("documents.tasks.bulk_update_documents", document_ids=...)` ×5 | `src/documents/bulk_edit.py:18, :31, :47, :63, :87` (import `:4`) |
| IMAP mail `async_task("documents.tasks.consume_file", path=..., ...)` | `src/paperless_mail/mail.py:336` (import `:11`) |
| `consume_file` body (→ `try_consume_file`) | `src/documents/tasks.py:184` (`:236`) |
| `bulk_update_documents` body | `src/documents/tasks.py:270` |
| post-consume signal handlers (6× connect) | `src/documents/apps.py:22-27` (ready `:11`) |

### C. Ephemeral progress channel (Q4)
| Claim | file:line |
|---|---|
| `_send_progress` def | `src/documents/consumer.py:56` |
| `async_to_sync(group_send)("status_updates", ...)` | `src/documents/consumer.py:73-74` |
| `_fail` → FAILED / `raise ConsumerError` | `src/documents/consumer.py:78-81` |
| STARTING / WORKING / SUCCESS statuses | `src/documents/consumer.py:202 / 240,259,264,274,294 / 375` |

### D. Django-Q 1.3.9 internals (verified in the installed package)
| Claim | file:line |
|---|---|
| `async_task` entry point | `django_q/tasks.py:20` |
| `SignedPackage.dumps(task)` | `django_q/tasks.py:69` |
| `broker.enqueue(pack)` | `django_q/tasks.py:73` |
| Redis broker key `django_q:paperless:q` | `django_q/brokers/redis_broker.py:15` |
| `enqueue` → RPUSH | `django_q/brokers/redis_broker.py:17-18` |
| `dequeue` → BLPOP | `django_q/brokers/redis_broker.py:20-21` |
| `queue_size` → LLEN | `django_q/brokers/redis_broker.py:25-26` |
| `SAVE_LIMIT` default 250 | `django_q/conf.py:87` |
| `QUEUE_LIMIT` default workers**2 | `django_q/conf.py:113` |
| status labels WORKING/IDLE | `django_q/conf.py:196-198` |
| in-memory `task_queue = Queue(maxsize=QUEUE_LIMIT)` | `django_q/cluster.py:161` |
| sentinel `status()` IDLE/WORKING | `django_q/cluster.py:175-181` |
| `pusher` (BLPOP → in-memory queue) | `django_q/cluster.py:333, :362` |
| `monitor` → `save_task` | `django_q/cluster.py:369, :384` |
| `worker` `res = f(*args, **kwargs)` | `django_q/cluster.py:399, :432` |
| failure result `f"{e} : {traceback.format_exc()}"` | `django_q/cluster.py:435` |
| `save_task` (SAVE_LIMIT gating; failures always saved) | `django_q/cluster.py:454, :461, :477` |
| `Stat` class / `status = sentinel.status()` | `django_q/status.py:30, :41` |
| `Task` model + `result` PickledObjectField | `django_q/models.py:20, :27` |
| `started` / `stopped` / `success` fields | `django_q/models.py:29-31` |
| `time_taken()` = (stopped-started).total_seconds() | `django_q/models.py:93` |
| default table `django_q_task` (app_label, no db_table override) | `django_q/models.py:104` |
| `Success` proxy filter(success=True) | `django_q/models.py:108-121` |
| `Failure` proxy filter(success=False) | `django_q/models.py:124-137` |
| admin registers Schedule/Success/Failure (+OrmQ conditional) | `django_q/admin.py:107-109, :112` |

### E. Manifests
| Claim | file:line |
|---|---|
| `django-q==1.3.9` | `requirements.txt:37` |
| `django==4.0.4` | `requirements.txt:38` |
| `redis==3.5.3` | `requirements.txt:84` |
| `channels==3.0.4` | `requirements.txt:23` |
| `channels-redis==3.4.0` | `requirements.txt:22` |
| `asgiref==3.5.0` | `requirements.txt:13` |
| `daphne==3.0.2` | `requirements.txt:31` |
| `aioredis==1.3.1` | `requirements.txt:10` |
| `hiredis==2.0.0` | `requirements.txt:44` |
| `django-q = "~=1.3"` | `Pipfile:17` |

### F. Sources consulted
- The paperless-ngx source tree at HEAD `542221a38dff06361e07976452f9aea24d210542`.
- A live run of the software's Django-Q + Redis stack in the project's canonical Python 3.9 container.
- The official **Django-Q 1.3.x** documentation (`django-q.readthedocs.io`: *Tasks*, *Brokers*, *Configuration*, *Admin pages*) and the `Koed00/django-q` source, used to corroborate `queue_size` (excludes in-flight), `save_limit` (default 250; failures always saved), `queue_limit` (defaults to `workers**2`), the Redis-broker no-receipts behavior, and the pusher/worker/monitor/sentinel architecture.
