# Paperless-NGX Idle (Steady-State) Runtime Behavior — Investigation Report

**Repository:** paperless-ngx
**Commit:** `542221a38dff06361e07976452f9aea24d210542`
**Branch:** `paperless-ngx_542221a38dff`
**Application version:** `1.7.0` — `__version__ = (1, 7, 0)` [src/paperless/version.py:L1] (there is no startup version banner in this release; the version is exposed only through the `X-Version` HTTP response header set by `ApiVersionMiddleware` [src/paperless/middleware.py:L14]).
**Task engine:** `django-q` **1.3.9** — `django-q==1.3.9` [requirements.txt:L37]. **This commit predates the project's migration to Celery.** The background engine is therefore the `qcluster` process; there is no `celery.py`, no Celery beat, and no Celery worker. Nothing in this report should be read as describing Celery.

---

## 1. Summary

This report answers, empirically and by name, five questions about what Paperless-NGX does when it is **idle** — running and ready, but with **no documents in the consumption directory and no OCR pipeline executing**:

- **Q1 — Startup:** the canonical boot sequence and the point at which the system is "ready."
- **Q2 — Idle background work:** which background processes/tasks keep executing automatically once idle.
- **Q3 — Periodic health/ready log entries:** the specific periodic log messages, their frequency, and what they indicate.
- **Q4 — Interrupt/restart recovery:** the specific log messages that confirm reconnection/operational-again after a component is briefly interrupted.
- **Q5 — Continuously-running components:** what keeps running continuously to maintain readiness even when no documents are being processed.

The investigation was **run-first**: the system was built/run in its canonical configuration, left to idle for **> 30 minutes**, and its **actual, unedited log output** was captured. Every behavioral claim below is presented with **(a)** the captured output, **(b)** the exact command that produced it, and **(c)** a `[path:line]` citation to the code that explains it. Per-run identifiers (cluster display name, task IDs, PIDs, worker count) are presented as **variable**. Statements are explicitly labeled **observed-at-runtime** vs **inferred-from-code**.

### 1.1 TL;DR

When Paperless-NGX is idle at this commit, exactly **three supervised long-running processes** stay alive — `gunicorn` (the ASGI web server on port 8000), `document_consumer` (a watchdog file-watcher), and `qcluster` (the `django-q` cluster) — backed by two always-on infrastructure dependencies, the **Redis broker** and the **SQLite database**. The only *automatic* background work is the **four seeded `django-q` schedules**: a mail check every **10 minutes**, classifier training **hourly**, index optimization **daily**, and a sanity check **weekly**. While idle the system is remarkably quiet: the `document_consumer` prints one watch line at startup and then is silent; `qcluster` is silent between schedule firings; and the container healthcheck (`curl -f http://localhost:8000` every 30 s) passes **without emitting any application log line** in the canonical configuration. The genuinely periodic *log* heartbeat is the **10-minute mail check** in the `qcluster` log. When the Redis broker is briefly stopped and restarted, `django-q` emits **no literal "reconnected" message**; instead recovery is confirmed by a composite signature — the connection errors **cease**, the sentinel **reincarnates the pusher** (`reincarnated pusher Process-1:N after sudden death` → new `pushing tasks at <pid>`), and the next scheduled task **enqueues normally** again.

---

## 2. Environment & Method

### 2.1 Canonical runtime

All values in this report were gathered from the **canonical Docker runtime** specified by the project setup: the image
`andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542`
(sourced from `ghcr.io/scaleapi/swe-atlas:...paperless-ngx...`), with the repository checked out at commit `542221a38dff`. The runtime baseline was verified to match the project's canonical pins:

```
$ docker exec paperless-app bash -lc 'python3 --version; \
    python3 -c "import django_q; print(\"django_q\", django_q.VERSION)"'
Python 3.9.23
django_q (1, 3, 9)

$ docker exec paperless-broker redis-server --version
Redis server v=6.0.20 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=dbdcb1f5eaf1bc2

$ docker exec paperless-app bash -lc 'for b in convert optipng curl gs tesseract pdftoppm pngquant; \
    do printf "%s -> " "$b"; command -v "$b" || echo MISSING; done'
convert   -> /usr/bin/convert
optipng   -> /usr/bin/optipng
curl      -> /usr/bin/curl
gs        -> /usr/bin/gs
tesseract -> /usr/bin/tesseract
pdftoppm  -> /usr/bin/pdftoppm
pngquant  -> /usr/bin/pngquant
```

This is **canonical** (**observed**): Python **3.9** and Redis **6.0** both match the shipped defaults, and every OCR/imaging binary and Python module (`pdf2image`, `pyzbar`) is present — so scheduled tasks execute normally with **no** missing-binary warnings. There is therefore no `pdf2image`-import worker death and no "Paperless can't find convert/optipng" artifact in this run (those non-canonical artifacts are discussed in §8).

### 2.2 Topology

Two containers run on a shared user-defined Docker network `paperless-net`:

```
$ docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
paperless-app     ghcr.io/scaleapi/swe-atlas:...qna_1.01     Up ...
paperless-broker  redis:6.0                                  Up ...
```

- **`paperless-app`** runs the three canonical long-running programs (`gunicorn`, `document_consumer`, `qcluster`) and the SQLite database at `/app/data/db.sqlite3`.
- **`paperless-broker`** is the canonical `redis:6.0` broker [docker/compose/docker-compose.sqlite.yml:L29].
- The app reaches the broker at `PAPERLESS_REDIS=redis://paperless-broker:6379` (a per-deployment hostname; the in-repo default is `redis://localhost:6379` [src/paperless/settings.py:L456]).

### 2.3 Observation method & discipline

- **Idle definition:** no files in the consumption directory, no OCR pipeline running; `MailAccount.objects.count() == 0` (verified in §4).
- **Duration:** the cluster was observed continuously from **19:21:30** to **> 19:51:48** (> 30 minutes), long enough to capture the 30-second healthcheck cadence many times and the 10-minute mail-check cadence across **three** clean intervals.
- **Cadence measurement:** every periodic value is derived from **timestamped** log lines and confirmed stable across **≥ 2** occurrences.
- **Hourly/daily/weekly schedules** are reported **from schedule configuration** (the seeding migrations), because they cannot fire inside a short window; this is stated explicitly wherever it applies.
- **Cross-run stability:** the startup sequence and worker count were confirmed identical across **two** independent cluster boots (§3.4).
- **Read-only mandate:** no source file was created, edited, or deleted. Temporary observation scripts lived under an in-container scratch dir and host `/tmp/blitzy_evidence/` (both outside the repository working tree) and were removed afterward; the clean `git status --porcelain` is shown in §10.

> **Timestamp note.** `django-q` renders its log lines in a short `HH:MM:SS [Q] LEVEL message` form (local process time), while Paperless's own loggers render `[YYYY-MM-DD HH:MM:SS,mmm] [LEVEL] [name] message` per the `LOGGING` `verbose` format `"[{asctime}] [{levelname}] [{name}] {message}"` [src/paperless/settings.py:L378]. Both forms appear verbatim below; the message body is the stable, library-canonical part.

---

## 3. Q1 — Startup: the canonical boot sequence and when the system is "ready"

### 3.1 The boot chain (inferred-from-code, with the ready signals observed below)

The canonical container entrypoint drives the boot in a fixed order:

1. **Entrypoint** `docker/docker-entrypoint.sh` prints `"Paperless-ngx docker container starting..."` [docker/docker-entrypoint.sh:L77], runs `initialize()` [docker/docker-entrypoint.sh:L84] (which invokes `gosu paperless /sbin/docker-prepare.sh` [docker/docker-entrypoint.sh:L37]), then `exec "$@"` [docker/docker-entrypoint.sh:L91]. Because the `CMD` starts with `/`, it is exec'd directly rather than treated as a `manage.py` subcommand [docker/docker-entrypoint.sh:L86].
2. **`CMD`** is `["/usr/local/bin/supervisord","-c","/etc/supervisord.conf"]` [Dockerfile:L172]; `ENTRYPOINT ["/sbin/docker-entrypoint.sh"]` [Dockerfile:L168]; `EXPOSE 8000` [Dockerfile:L170]; base image `python:3.9-slim-bullseye` [Dockerfile:L18]. **There is no `HEALTHCHECK` directive in the Dockerfile** (verified) — the healthcheck lives only in docker-compose (see §5.1).
3. **`docker-prepare.sh`** is the readiness sequence (inferred-from-code):
   - PostgreSQL `pg_isready` wait — **only when `PAPERLESS_DBHOST` is set** [docker/docker-prepare.sh:L67-69], i.e. skipped for the default SQLite backend.
   - Redis wait via `python3 /sbin/wait-for-redis.py` [docker/docker-prepare.sh:L33].
   - flock-guarded `echo "Apply database migrations..."` [docker/docker-prepare.sh:L44] then `python3 manage.py migrate` [docker/docker-prepare.sh:L45].
   - `echo "Search index out of date. Updating..."` [docker/docker-prepare.sh:L54] then `python3 manage.py document_index reindex` [docker/docker-prepare.sh:L55].
   - superuser bootstrap [docker/docker-prepare.sh:L60-63].
4. **Redis wait strings** (inferred-from-code) in `docker/wait-for-redis.py`: prints `"Waiting for Redis: {REDIS_URL}"` [docker/wait-for-redis.py:L21]; retries `MAX_RETRY_COUNT = 5` [docker/wait-for-redis.py:L16] × `RETRY_SLEEP_SECONDS = 5` [docker/wait-for-redis.py:L17]; on success prints `"Connected to Redis broker: {REDIS_URL}"` [docker/wait-for-redis.py:L41]; on failure prints `"Failed to connect to: {REDIS_URL}"` [docker/wait-for-redis.py:L38] and `sys.exit(os.EX_UNAVAILABLE)` [docker/wait-for-redis.py:L39].
5. **`supervisord` launches exactly three long-running programs** [docker/supervisord.conf]: `[program:gunicorn]` → `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` [docker/supervisord.conf:L10-11]; `[program:consumer]` → `python3 manage.py document_consumer` [docker/supervisord.conf:L19-20]; `[program:scheduler]` → `python3 manage.py qcluster` [docker/supervisord.conf:L28-29]. Each program logs to `/dev/stdout` + `/dev/stderr`.

### 3.2 gunicorn "ready" signal (observed)

Command that produced the output:

```
$ docker exec paperless-app bash -lc 'cat /tmp/gunicorn.log'
[2026-07-14 19:21:30 +0000] [107] [INFO] Starting gunicorn 20.1.0
[2026-07-14 19:21:30 +0000] [107] [INFO] Listening at: http://0.0.0.0:8000 (107)
[2026-07-14 19:21:30 +0000] [107] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-14 19:21:30 +0000] [107] [INFO] Server is ready. Spawning workers
```

- `gunicorn.conf.py` binds `0.0.0.0:8000` [gunicorn.conf.py:L3], defaults to `workers = 2` [gunicorn.conf.py:L4], uses `worker_class = "paperless.workers.ConfigurableWorker"` [gunicorn.conf.py:L5] (which is `class ConfigurableWorker(UvicornWorker)` [src/paperless/workers.py:L9]), and sets `timeout = 120` [gunicorn.conf.py:L6].
- **The readiness marker is `"Server is ready. Spawning workers"`**, emitted by the `when_ready` hook `server.log.info("Server is ready. Spawning workers")` [gunicorn.conf.py:L18]. After this line the REST API/ASGI app is accepting connections on port 8000 (confirmed independently: `curl` to `:8000` returns HTTP 302, and `/api/` returns HTTP 200 — see §5).

### 3.3 document_consumer "ready" signal (observed)

```
$ docker exec paperless-app bash -lc 'cat /tmp/consumer.log'
[2026-07-14 19:21:30,629] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

- The consumer logs its watch line **once** at startup and is then **silent while idle**. The **inotify** path was active in this run, emitting `"Using inotify to watch directory for changes: {directory}"` [src/documents/management/commands/document_consumer.py:L200]. (The alternative is the polling path `"Polling directory for changes: {directory}"` [src/documents/management/commands/document_consumer.py:L186] using `PollingObserver(timeout=settings.CONSUMER_POLLING)` [src/documents/management/commands/document_consumer.py:L187].) The enqueue line `"Adding {filepath} to the task queue."` [src/documents/management/commands/document_consumer.py:L85] fires only when a file actually arrives, so it does **not** appear in an idle capture.

### 3.4 qcluster (`django-q`) startup sequence (observed, cross-run stable)

```
$ docker exec paperless-app bash -lc 'cat /tmp/qcluster.log'   # baseline run #1
19:21:30 [Q] INFO Q Cluster kilo-blue-snake-yankee starting.
19:21:30 [Q] INFO Process-1:1 ready for work at 158
19:21:30 [Q] INFO Process-1:2 ready for work at 159
19:21:30 [Q] INFO Process-1:3 ready for work at 160
19:21:30 [Q] INFO Process-1:4 ready for work at 161
19:21:30 [Q] INFO Process-1:5 ready for work at 162
19:21:30 [Q] INFO Process-1:6 ready for work at 163
19:21:30 [Q] INFO Process-1:7 ready for work at 164
19:21:30 [Q] INFO Process-1:8 ready for work at 165
19:21:30 [Q] INFO Process-1:9 ready for work at 166
19:21:30 [Q] INFO Process-1:10 ready for work at 167
19:21:30 [Q] INFO Process-1:11 ready for work at 168
19:21:30 [Q] INFO Process-1:12 monitoring at 169
19:21:30 [Q] INFO Process-1 guarding cluster kilo-blue-snake-yankee
19:21:30 [Q] INFO Process-1:13 pushing tasks at 170
19:21:30 [Q] INFO Q Cluster kilo-blue-snake-yankee running.
```

The **readiness marker is the final line, `"Q Cluster <name> running."`**. The bootstrap comprises, in order: **N worker processes** each logging `Process-1:N ready for work at <pid>`; one **monitor** (`Process-1:12 monitoring at <pid>`); the **sentinel/guard** (`Process-1 guarding cluster <name>`); and the **pusher/scheduler** (`Process-1:13 pushing tasks at <pid>`). These `django-q` strings are **library-canonical** (validated against `django-q` docs and the Koed00/django-q issue tracker), not Paperless-specific.

**Cluster-name nuance (cause → effect).** The config sets `Q_CLUSTER["name"] = "paperless"` [src/paperless/settings.py:L450], but the **runtime display name** in the `"Q Cluster ... running."` line is a **randomly generated humanized name** — here `kilo-blue-snake-yankee`. The config value `"paperless"` functions as the broker/queue key and signing salt; it does **not** appear in the running-banner line. The display name therefore must be treated as a **per-run variable identifier** (`<cluster-name>`).

**Worker count (host-dependent, variable).** `TASK_WORKERS` [src/paperless/settings.py:L438] defaults to `default_task_workers()` [src/paperless/settings.py:L427], which computes `available_cores = max(multiprocessing.cpu_count(), 1)` [src/paperless/settings.py:L429], returns that count if `< 4` [src/paperless/settings.py:L431-432], else `max(math.floor(math.sqrt(available_cores)), 1)` [src/paperless/settings.py:L433]. In this container `os.cpu_count() == 128`, so `floor(sqrt(128)) = 11` — **exactly the 11 workers observed** (`Process-1:1` … `Process-1:11`). Report this as **host-dependent** (e.g. 2 workers at 4 cores, 3 at 9 cores, 11 at 128 cores).

**Cross-run stability (observed).** A second, independently launched cluster produced an identical structure with a different random name — confirming the sequence and the worker count are stable across runs:

```
$ docker exec paperless-app bash -lc 'cd /app/src && nohup python3 manage.py qcluster \
    > /tmp/blitzy_investigation/qcluster_run2.log 2>&1 & sleep 7; \
    cat /tmp/blitzy_investigation/qcluster_run2.log'   # controlled run #2
19:44:29 [Q] INFO Q Cluster maine-lamp-wolfram-eleven starting.
19:44:29 [Q] INFO Process-1:1 ready for work at 742
...
19:44:29 [Q] INFO Process-1:11 ready for work at 752
19:44:29 [Q] INFO Process-1:12 monitoring at 753
19:44:29 [Q] INFO Process-1 guarding cluster maine-lamp-wolfram-eleven
19:44:29 [Q] INFO Process-1:13 pushing tasks at 754
19:44:29 [Q] INFO Q Cluster maine-lamp-wolfram-eleven running.
```

Run #1 name `kilo-blue-snake-yankee` vs run #2 `maine-lamp-wolfram-eleven`; **both 11 workers**, identical line sequence. (Stopping run #2 additionally exercised the graceful **stop procedure** — `Process-1 waiting for the monitor.` / `stopped monitoring results` / `Q Cluster <name> has stopped.` — which is the clean-shutdown counterpart to the startup banner.)


---

## 4. Q2 — Idle background work: the four seeded `django-q` schedules

Once idle, the **only automatic background work** is the four `django-q` schedules driven by the `qcluster` pusher. Each is seeded by a migration (all importing `from django_q.models import Schedule` and `from django_q.tasks import schedule`) and stored as a `Schedule` row in the (SQLite) database.

| Schedule name | Function | Cadence | Type code | Seed migration |
|---|---|---|---|---|
| "Train the classifier" | `documents.tasks.train_classifier` | **HOURLY** | `H` | src/documents/migrations/1001_auto_20201109_1636.py:L10-14 |
| "Optimize the index" | `documents.tasks.index_optimize` | **DAILY** | `D` | src/documents/migrations/1001_auto_20201109_1636.py:L15-19 |
| "Perform sanity check" | `documents.tasks.sanity_check` | **WEEKLY** | `W` | src/documents/migrations/1004_sanity_check_schedule.py:L10-14 |
| "Check all e-mail accounts" | `paperless_mail.tasks.process_mail_accounts` | **every 10 MINUTES** | `I`, `minutes=10` | src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-15 |

Task bodies (citations): `index_optimize()` [src/documents/tasks.py:L32], `train_classifier()` [src/documents/tasks.py:L48], `sanity_check()` [src/documents/tasks.py:L255]; the mail task `process_mail_accounts()` [src/paperless_mail/tasks.py:L11] iterates `MailAccount.objects.all()` [src/paperless_mail/tasks.py:L13].

### 4.1 The seeded schedule rows (observed)

Enumerated with a temporary, read-only management shell (helper removed afterward):

```
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c \
  "from django_q.models import Schedule; import json; \
   print(json.dumps(list(Schedule.objects.values_list(\"name\",\"func\",\"schedule_type\",\"minutes\",\"repeats\").order_by(\"id\")), default=str)); \
   print(\"COUNT=\", Schedule.objects.count())"'
["Train the classifier",       "documents.tasks.train_classifier",          "H", null, -2]
["Optimize the index",         "documents.tasks.index_optimize",            "D", null, -2]
["Perform sanity check",       "documents.tasks.sanity_check",              "W", null, -2]
["Check all e-mail accounts",  "paperless_mail.tasks.process_mail_accounts","I", 10,   -3]
COUNT= 4
```

The type codes are `django-q`'s `Schedule` constants: `H`=hourly, `D`=daily, `W`=weekly, `I`=minutes (with `minutes=10`). Negative `repeats` (`-2`/`-3`) means "repeat forever." **COUNT = 4** confirms these are the complete set of automatic background tasks in an idle system.

### 4.2 The only firing observed in the window: the 10-minute mail check (observed)

In a > 30-minute window, only the **10-minute** schedule fires repeatedly. The hourly/daily/weekly schedules are therefore **reported-from-config** (the migrations above), **not observed-firing** — they would next fire well outside the observation window. (The one exception is the **startup catch-up burst** at 19:22:00, discussed in §5.3, where all four fired once because their initial `next_run` was already in the past and `Q_CLUSTER["catch_up"] = False` [src/paperless/settings.py:L451] causes a single immediate run rather than back-filling.)

A representative steady-state (non-startup) mail firing:

```
$ docker exec paperless-app bash -lc 'grep -E "^19:41" /tmp/qcluster.log'
19:41:33 [Q] INFO Enqueued 1
19:41:33 [Q] INFO Process-1:6 processing [thirteen-lion-minnesota-romeo]
19:41:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:41:33 [Q] INFO Process-1:6 stopped doing work
19:41:33 [Q] INFO Processed [thirteen-lion-minnesota-romeo]
19:41:33 [Q] INFO recycled worker Process-1:6
19:41:33 [Q] INFO Process-1:19 ready for work at 645
```

Mechanism (cause → effect): the pusher `created a task from schedule [Check all e-mail accounts]` and `Enqueued 1` onto the Redis list; a worker picked it up (`Process-1:6 processing [<task-id>]`), ran it, and logged `Processed [<task-id>]`. The worker was then **recycled** (`recycled worker Process-1:6` → a fresh `Process-1:19 ready for work at <pid>`) because `Q_CLUSTER["recycle"] = 1` [src/paperless/settings.py:L452] recycles a worker after every task. Task IDs (`thirteen-lion-minnesota-romeo`) are **randomly generated humanized strings**, i.e. per-run variable.

### 4.3 Idle mail-task result: "No new documents were added." (observed)

With no mail accounts configured, the mail task returns the else-branch string `"No new documents were added."` [src/paperless_mail/tasks.py:L22]. This surfaces as the **`django-q` task result** recorded in the database (not as a `qcluster` log line):

```
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c \
  "from django_q.models import Task; \
   [print(t.started, t.success, repr(t.result)) for t in \
    Task.objects.filter(func=\"paperless_mail.tasks.process_mail_accounts\").order_by(\"-started\")[:3]]; \
   from paperless_mail.models import MailAccount; print(\"MailAccount count =\", MailAccount.objects.count())"'
2026-07-14 19:41:33.275107+00:00 True 'No new documents were added.'
2026-07-14 19:31:31.856888+00:00 True 'No new documents were added.'
2026-07-14 19:22:00.427514+00:00 True 'No new documents were added.'
MailAccount count = 0
```

`MailAccount count = 0` confirms the idle precondition, and every mail-check run is `success=True` with result `'No new documents were added.'` — exactly the else-branch at [src/paperless_mail/tasks.py:L22].


---

## 5. Q3 — Periodic health/ready log entries: specific strings, frequency, and meaning

There are two distinct periodic idle signals. **Only one produces a log line** in the canonical configuration; the other is a periodic readiness *probe* that is silent at the application-log level. Both are documented below with measured cadence.

### 5.1 The container healthcheck — every 30 s, **compose-only, and silent in application logs**

**Definition (from config).** The healthcheck is declared in docker-compose, **not** in the Dockerfile:

```yaml
# docker/compose/docker-compose.sqlite.yml
healthcheck:                                              # L41
  test: ["CMD", "curl", "-f", "http://localhost:8000"]    # L42
  interval: 30s                                           # L43
  timeout: 10s                                            # L44
  retries: 5                                              # L45
```

`curl -f http://localhost:8000`, `interval 30s` [docker/compose/docker-compose.sqlite.yml:L41-45], against the `redis:6.0` broker service [docker/compose/docker-compose.sqlite.yml:L29]. **Critical caveat:** there is **no `HEALTHCHECK` in the `Dockerfile`** (verified), so this 30 s probe fires automatically **only** under docker-compose. In this dev container the running image has `Healthcheck = null` (verified via `docker inspect`), so the probe was **reproduced manually** (below) and is labeled as such.

**Cadence (observed, manual reproduction).** A loop issuing the exact probe command every 30 s:

```
$ docker exec paperless-app bash -lc 'cat /tmp/blitzy_investigation/healthcheck_capture.log'
[2026-07-14 19:40:05] round=1  curl_exit=0 http_status=302
[2026-07-14 19:40:35] round=2  curl_exit=0 http_status=302
[2026-07-14 19:41:05] round=3  curl_exit=0 http_status=302
[2026-07-14 19:41:35] round=4  curl_exit=0 http_status=302
[2026-07-14 19:42:05] round=5  curl_exit=0 http_status=302
[2026-07-14 19:42:35] round=6  curl_exit=0 http_status=302
[2026-07-14 19:43:05] round=7  curl_exit=0 http_status=302
[2026-07-14 19:43:36] round=8  curl_exit=0 http_status=302
[2026-07-14 19:44:06] round=9  curl_exit=0 http_status=302
[2026-07-14 19:44:36] round=10 curl_exit=0 http_status=302
[2026-07-14 19:45:06] round=11 curl_exit=0 http_status=302
[2026-07-14 19:45:36] round=12 curl_exit=0 http_status=302
[2026-07-14 19:46:06] round=13 curl_exit=0 http_status=302
[2026-07-14 19:46:36] round=14 curl_exit=0 http_status=302
```

**Frequency:** consecutive deltas are **30 s** (19:40:05 → 19:40:35 → 19:41:05 …), stable across all 14 rounds. **Meaning:** every 30 s Docker confirms the web server is alive; `curl -f` treats the app's `302` redirect (to the login page) as success (`curl_exit=0`), so the container is marked **healthy**.

**What it does NOT do (observed — corrects a common assumption):** the healthcheck produces **no application log line**. Across all 14 probes (plus extra manual hits to `/` and `/api/`), `gunicorn.log` never grew beyond its four startup lines:

```
$ docker exec paperless-app bash -lc 'wc -l /tmp/gunicorn.log'
4 /tmp/gunicorn.log      # still only the startup banner; zero access lines
```

**Mechanism (observed + inferred).** At runtime the access loggers have no handler and Paperless configures only its own loggers:

```
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c \
  "import logging; lg=logging.getLogger(\"uvicorn.access\"); \
   print(\"uvicorn.access level=\", logging.getLevelName(lg.level), \"handlers=\", lg.handlers, \"propagate=\", lg.propagate)"'
uvicorn.access level= NOTSET handlers= [] propagate= True
```

`gunicorn.conf.py` sets **no `accesslog`**; the `uvicorn.access`/`gunicorn.access` loggers have **no handlers of their own** (observed above); and Paperless's `LOGGING` dict configures handlers only for the `paperless` and `paperless_mail` loggers plus a root `console`/file handler set (`disable_existing_loggers = False`) with the `verbose` format [src/paperless/settings.py:L373-412] — no access-log handler anywhere. **Consequently the 30 s healthcheck is a periodic readiness *probe* (HTTP-response + Docker health status), not a periodic log message.** Under docker-compose its result is observable as the container's `healthy` status; it is *not* visible in Paperless's logs.

### 5.2 The scheduler firing the 10-minute mail check — the genuine periodic *log* heartbeat (observed)

The one periodic **log** entry in an idle system is the `qcluster` pusher firing the every-10-minute mail check. Measured across the observation window:

```
$ docker exec paperless-app bash -lc 'grep "created a task from schedule \[Check all e-mail accounts\]" /tmp/qcluster.log'
19:22:00 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # startup catch-up
19:31:31 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:41:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:51:47 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # post-Redis-restart (see §6)
```

**Frequency (measured, ≥ 2 clean occurrences):** steady-state deltas are **~10 minutes** — 19:31:31 → 19:41:33 = **10 m 02 s**, and 19:41:33 → 19:51:47 = **10 m 14 s**. The few-second jitter comes from the pusher's sub-minute poll granularity (the `django-q` docs state the scheduler "checks for any scheduled tasks that should be starting" roughly twice a minute). **Meaning:** the scheduler is alive and firing due schedules; each firing writes `Enqueued <n>`, `Process-1 created a task from schedule [<name>]`, and `Process-1:N processing [<task-id>]` (full example in §4.2). These `django-q` strings are **library-canonical**.

### 5.3 The startup catch-up burst (observed) — why all four fired once at 19:22:00

At the first pusher poll (~30 s after boot) every schedule whose seeded `next_run` was already in the past fired once:

```
$ docker exec paperless-app bash -lc 'grep -E "^19:22:00 .*(created a task|Enqueued|Sanity)" /tmp/qcluster.log'
19:22:00 [Q] INFO Enqueued 1
19:22:00 [Q] INFO Process-1 created a task from schedule [Train the classifier]
19:22:00 [Q] INFO Enqueued 1
19:22:00 [Q] INFO Process-1 created a task from schedule [Optimize the index]
19:22:00 [Q] INFO Enqueued 1
19:22:00 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
19:22:00 [Q] INFO Enqueued 1
19:22:00 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

The weekly sanity check that ran in this burst logged its result via the Paperless logger (note the different `[{asctime}]` format):

```
[2026-07-14 19:22:00,574] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

This burst is a **one-time boot event**, not a periodic signal; after it, only the 10-minute mail check recurs within the window.

### 5.4 Idle means quiet — "silent between firings" (observed)

Between firings, `qcluster` emits **nothing**. Two consecutive silent gaps prove it:

```
$ docker exec paperless-app bash -lc 'grep -E "^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\]" /tmp/qcluster.log | \
    awk "\$1>\"19:22:01\" && \$1<\"19:31:31\"" | wc -l'
0     # zero [Q] lines for ~9.5 min (19:22:01 -> 19:31:31)
$ docker exec paperless-app bash -lc 'grep -E "^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\]" /tmp/qcluster.log | \
    awk "\$1>\"19:31:32\" && \$1<\"19:41:33\"" | wc -l'
0     # zero [Q] lines for ~10 min (19:31:32 -> 19:41:33)
```

**Summary of Q3:** in an idle canonical deployment the periodic signals are (1) a **30 s healthcheck** that keeps the container `healthy` but is **silent in application logs**, and (2) a **10-minute mail-check** firing that *is* the periodic log heartbeat. The `document_consumer` and the between-firing `qcluster` contribute **no** periodic idle log chatter.


---

## 6. Q4 — Interrupt/restart recovery: stopping and restarting the Redis broker

**Why Redis is the right interrupt target.** Redis has a **dual role**: it is the `django-q` broker (`"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` [src/paperless/settings.py:L456]) **and** the `channels_redis` websocket channel-layer backend (`"BACKEND": "channels_redis.core.RedisChannelLayer"` [src/paperless/settings.py:L180], `"hosts": [os.getenv("PAPERLESS_REDIS", ...)]` [src/paperless/settings.py:L182]). Interrupting Redis therefore directly exercises the reconnection path of the background engine.

The experiment captured all three states required — **before**, **during**, and **after** — by stopping and restarting the broker **container** while the canonical `qcluster` ran idle.

### 6.1 Before (observed)

```
$ docker ps --filter name=paperless-broker --format '{{.Names}} {{.Status}}'
paperless-broker Up 42 minutes
$ docker exec paperless-app bash -lc '[ -r /proc/170/cmdline ] && echo "pusher PID 170 ALIVE"'
pusher PID 170 ALIVE
```

The pusher was `Process-1:13` at **PID 170** (from the startup banner in §3.4), broker healthy.

### 6.2 During the outage (observed) — the actual error string

```
$ date '+%F %T'; docker stop paperless-broker
2026-07-14 19:45:54
paperless-broker            # stopped; broker container Exited (0)
```

Within ~1 s the pusher began logging connection failures, repeating for the whole ~2-minute outage:

```
$ docker exec paperless-app bash -lc 'grep -E "^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\] ERROR Error" /tmp/qcluster.log | head'
19:45:55 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
19:45:55 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
19:45:56 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
...
$ docker exec paperless-app bash -lc 'grep -c "Error -5 connecting" /tmp/qcluster.log'
203
```

**Actual observed outage error:** `Error -5 connecting to paperless-broker:6379. No address associated with hostname.` — a **DNS-resolution failure** (`socket.gaierror: [Errno -5] No address associated with hostname`, wrapped by `redis.exceptions.ConnectionError`). **Cause → effect (and a deliberate note on deviation from the predicted string):** because the canonical broker is addressed by its **Docker hostname** `paperless-broker` (not `localhost`), `docker stop` removes the container's DNS record from the `paperless-net` network, so the client fails at **name resolution** (errno -5) rather than at the socket. This is distinct from the two other outage modes: a **"Connection refused"** error would appear if the hostname still resolved but the port were closed (e.g. the redis process killed while the container/network stayed up), and **"Error 99 … Cannot assign requested address"** is the `localhost` ephemeral-port-exhaustion signature from very rapid retries. We report the string we actually observed; the reconnection *semantics* (below) are identical regardless of which outage error appears.

> `django-q` 1.3.9 quirk (observed): interleaved with the concise error lines are Python `--- Logging error --- TypeError: not all arguments converted during string formatting` blocks. This is a **library** behavior — `logger.error(e, traceback.format_exc())` at `django_q/cluster.py:347` passes the traceback as a `%`-format argument — not a Paperless issue. The concise `[Q] ERROR ...` line is still emitted correctly.

**The pusher is repeatedly reincarnated during the outage (observed).** The sentinel/guard detects the failing pusher and reincarnates it roughly every 10 s; the original pusher is the first to die:

```
$ docker exec paperless-app bash -lc 'grep "reincarnated pusher" /tmp/qcluster.log'
19:46:04 [Q] ERROR reincarnated pusher Process-1:13 after sudden death   # <- original pusher (PID 170)
19:46:15 [Q] ERROR reincarnated pusher Process-1:20 after sudden death
19:46:25 [Q] ERROR reincarnated pusher Process-1:21 after sudden death
19:46:37 [Q] ERROR reincarnated pusher Process-1:22 after sudden death
19:46:47 [Q] ERROR reincarnated pusher Process-1:23 after sudden death
19:46:57 [Q] ERROR reincarnated pusher Process-1:24 after sudden death
19:47:07 [Q] ERROR reincarnated pusher Process-1:25 after sudden death
19:47:17 [Q] ERROR reincarnated pusher Process-1:26 after sudden death
19:47:28 [Q] ERROR reincarnated pusher Process-1:27 after sudden death
19:47:38 [Q] ERROR reincarnated pusher Process-1:28 after sudden death
19:47:49 [Q] ERROR reincarnated pusher Process-1:29 after sudden death
19:47:59 [Q] ERROR reincarnated pusher Process-1:30 after sudden death
```

Each reincarnation is a three-line signature — `Process-1:N stopped pushing tasks` → `reincarnated pusher Process-1:N after sudden death` → new `Process-1:M pushing tasks at <new-pid>`:

```
19:46:47 [Q] INFO  Process-1:23 stopped pushing tasks
19:46:47 [Q] ERROR reincarnated pusher Process-1:23 after sudden death
19:46:47 [Q] INFO  Process-1:24 pushing tasks at 826
```

**Cause → effect:** the sentinel "checks the health of all workers, including the pusher and the monitor … in case of a sudden death or timeout it will reincarnate the failing processes" (library-canonical mechanism). While Redis is unreachable each freshly-reincarnated pusher immediately fails again, so the reincarnation repeats. The `Process-1:N` counter climbs (13 → 20 → 21 → … → 31; labels 14–19 were consumed earlier by worker recycling in §4.2/§5.3), and each new pusher gets a **new PID** (e.g. 826, 827, … 858, 859) — the PID change is the concrete evidence of reincarnation. Present `N` and the PIDs as **variable**.

### 6.3 After the restart (observed) — the recovery signature

```
$ date '+%F %T'; docker start paperless-broker
2026-07-14 19:47:49
paperless-broker            # broker back Up
```

```
$ docker exec paperless-app bash -lc 'grep -E "^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\]" /tmp/qcluster.log | awk "\$1>=\"19:47:48\""'
19:47:48 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
19:47:48 [Q] INFO  Process-1:29 stopped pushing tasks
19:47:49 [Q] ERROR reincarnated pusher Process-1:29 after sudden death
19:47:49 [Q] INFO  Process-1:30 pushing tasks at 858
19:47:49 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.   # last error
19:47:59 [Q] INFO  Process-1:30 stopped pushing tasks
19:47:59 [Q] ERROR reincarnated pusher Process-1:30 after sudden death
19:47:59 [Q] INFO  Process-1:31 pushing tasks at 859       # <- stable post-recovery pusher
```

Two independent confirmations that the broker is reachable and the final pusher survived:

```
$ docker exec paperless-app bash -lc '[ -r /proc/859/cmdline ] && echo "PID 859 ALIVE (stable pusher)"'
PID 859 ALIVE (stable pusher)
$ docker exec paperless-app bash -lc 'cd /app/src && python3 -c \
  "import redis,os; print(\"PING ->\", redis.from_url(os.getenv(\"PAPERLESS_REDIS\")).ping())"'
PING -> True
```

**Full operational recovery (observed) — resumed enqueue.** The definitive "operational again" proof is that the **next scheduled task fires normally**:

```
$ docker exec paperless-app bash -lc 'grep -E "^19:51" /tmp/qcluster.log'
19:51:47 [Q] INFO Enqueued 1
19:51:47 [Q] INFO Process-1:7 processing [neptune-harry-emma-spaghetti]
19:51:47 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:51:47 [Q] INFO Process-1:7 stopped doing work
19:51:47 [Q] INFO recycled worker Process-1:7
19:51:47 [Q] INFO Process-1:32 ready for work at 906
19:51:48 [Q] INFO Processed [neptune-harry-emma-spaghetti]
```

(The corresponding `django-q` task result was `success=True`, `'No new documents were added.'`, i.e. normal idle behavior fully restored.)

### 6.4 What "reconnected" looks like here (explicit)

**`django-q` emits no literal "reconnected" (or "reconnecting"/"connection restored") string** — verified: `grep -iE "reconnect|reconnected|restored" /tmp/qcluster.log` returns nothing. **Recovery must be inferred from a composite, observed signature:**

1. the repeated `Error -5 connecting to paperless-broker:6379 …` lines **stop** (last one at 19:47:49, right when the broker returned) — **cessation of errors**;
2. a final `reincarnated pusher Process-1:N after sudden death` is followed by a `Process-1:M pushing tasks at <pid>` that **stays alive** (PID 859) — **a surviving pusher**;
3. the next due schedule **enqueues and processes normally** (`Enqueued 1` / `created a task from schedule [Check all e-mail accounts]` / `Processed [<id>]`) — **resumed work**.

**Observed vs inferred:** the error lines, reincarnation lines, surviving-PID, and resumed enqueue are all **observed-at-runtime**. The conclusion "everything is reconnected and operational again" is **inferred** from the *combination* of (1)+(2)+(3) — there is no single explicit all-clear message.

### 6.5 The web server rode through the outage (observed) — Redis is not needed to serve HTTP

The 30 s healthcheck loop of §5.1 was running across the outage window (broker down 19:45:59–19:47:49). Rounds **13 (19:46:06)** and **14 (19:46:36)** fell squarely inside the outage and still returned `http_status=302 curl_exit=0`. **Cause → effect:** Redis backs only the `django-q` broker and the websocket channel layer; plain HTTP request serving does not touch Redis, so `gunicorn` stays **healthy** even while Redis is down. Only background scheduling and websocket delivery are affected during a Redis outage.


---

## 7. Q5 — Components/processes that keep running continuously to maintain readiness

Even with no documents processing, the following stay up continuously. The three application processes were confirmed alive via their in-container cmdlines (`/proc/<pid>/cmdline`); the infrastructure components via `docker ps` and a Redis `PING`.

### 7.1 The three supervised long-running processes

1. **`gunicorn` — the ASGI web server.** Serves the Django REST API and the ASGI app on `0.0.0.0:8000` [gunicorn.conf.py:L3]; it is the target of the healthcheck; readiness marker `"Server is ready. Spawning workers"` [gunicorn.conf.py:L18]. Supervised as `[program:gunicorn]` [docker/supervisord.conf:L10-11]. It also **hosts the websocket/Channels layer** (§7.3). Cause → effect: it must stay up so the UI/API and the healthcheck endpoint remain reachable at all times. **Observed** continuously (HTTP 302 on every probe, including during the Redis outage).

2. **`document_consumer` — the file watcher.** Runs a `watchdog` observer over the consumption directory; imports `from watchdog.observers.polling import PollingObserver` [src/documents/management/commands/document_consumer.py:L17]. It logs its watch line **once** — here `"Using inotify to watch directory for changes: {directory}"` [src/documents/management/commands/document_consumer.py:L200] — and is then **silent while idle**. The enqueue line `"Adding {filepath} to the task queue."` [src/documents/management/commands/document_consumer.py:L85] (which calls `async_task("documents.tasks.consume_file", …)` [src/documents/management/commands/document_consumer.py:L86]) fires **only when a file arrives**, so it does **not** appear in an idle capture. Supervised as `[program:consumer]` [docker/supervisord.conf:L19-20]. Cause → effect: it stays up so that a newly-dropped document is picked up instantly.

3. **`qcluster` — the `django-q` cluster (the only always-on scheduled-task engine).** Comprises a **sentinel/guard**, a **monitor**, a **pusher/scheduler**, and **N workers** (`N = floor(sqrt(cores))`, here **11** — §3.4). The pusher polls sub-minute and fires the four schedules (§4) at their intervals, but is **silent between firings** (§5.4). Supervised as `[program:scheduler]` [docker/supervisord.conf:L28-29]; configured by `Q_CLUSTER` [src/paperless/settings.py:L449-457]. Cause → effect: it stays up so the periodic maintenance tasks (mail/classifier/index/sanity) keep running automatically.

### 7.2 Infrastructure that must stay up

4. **Redis broker (`redis:6.0`).** Dual role — `django-q` broker [src/paperless/settings.py:L456] **and** `channels_redis` channel layer [src/paperless/settings.py:L180-182]. It is a **hard prerequisite**: while it is down, background scheduling and websocket delivery stop (as demonstrated in §6), though HTTP serving continues. Canonical version is `redis:6.0` across the compose files [docker/compose/docker-compose.sqlite.yml:L29]. **Observed** continuously (broker `Up`; `PING -> True`).

5. **Database (SQLite default).** Stores the `django-q` `Schedule` rows and task results (both enumerated live in §4). Migrations are applied at startup — `"Apply database migrations..."` [docker/docker-prepare.sh:L44] → `manage.py migrate` [docker/docker-prepare.sh:L45]. Cause → effect: without it the scheduler has no schedule definitions to fire and nowhere to record results. It is not a separate process (embedded), but it is continuously required.

### 7.3 The websocket / Channels layer (part of gunicorn's ASGI app, not a separate process)

6. **Channels websocket layer.** `src/paperless/asgi.py` builds a `ProtocolTypeRouter` [src/paperless/asgi.py:L17] multiplexing `"http": get_asgi_application()` [src/paperless/asgi.py:L19] and `"websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` [src/paperless/asgi.py:L20]. The websocket `StatusConsumer` [src/paperless/consumers.py:L9] joins the `status_updates` group on connect (`async_to_sync(self.channel_layer.group_add)("status_updates", self.channel_name)` [src/paperless/consumers.py:L17-19]). Cause → effect: this keeps the server **ready to push** live processing-status updates to the Angular UI even while idle; delivery rides on the Redis channel layer (§7.4). It runs **inside** the gunicorn ASGI worker — it is not a fourth process. (The Angular UI itself is out of scope beyond this.)

### 7.4 `supervisord` — the mechanism, not a readiness signal

7. **`supervisord`** supervises and auto-restarts the three programs [docker/supervisord.conf; docker/docker-entrypoint.sh:L84-93]. It is the **mechanism** that keeps the components alive (restarting any that crash); it is **not itself a readiness signal**. **Environmental note (non-canonical):** in the dev image used here the three programs were launched **directly** rather than under `supervisord` (see §8); this does not change *what* runs continuously, only *how* it is supervised.

### 7.5 Supporting always-loaded state (inferred-from-code)

Signal wiring happens at app-ready: `DocumentsConfig.ready()` [src/documents/apps.py:L11] connects the `document_consumption_finished` handlers [src/documents/apps.py:L22-27]. These are only exercised during ingestion, but they are part of the always-loaded application state that keeps the system ready to process a document the instant one appears.

**Q5 summary:** the continuously-running set is **`gunicorn` + `document_consumer` + `qcluster`** (the three supervised processes), plus the **Redis broker** and the **SQLite database**, with the **Channels websocket layer** living inside gunicorn and **`supervisord`** as the supervising mechanism.


---

## 8. Canonical vs. non-canonical appendix

All *behavioral* values above were gathered in the **canonical** runtime (Python **3.9.23**, Redis **6.0.20**, `django-q` **1.3.9**, all OCR/imaging binaries and Python modules present — §2.1). No missing-binary warnings occurred, and no worker died from a missing module. The following are **environmental deltas** of the specific dev container used, disclosed for completeness. **None changes what runs or the log strings**; they affect only *how* the processes were supervised/addressed.

| Item | Canonical default | This run | Impact on the findings |
|---|---|---|---|
| Process supervision | `supervisord` launches the 3 programs [docker/supervisord.conf; Dockerfile:L172] | 3 programs launched **directly** (no `supervisord` in the dev image) | None on *what* runs or its logs; only the supervising mechanism differs (§7.4) |
| Broker address | in-repo default `redis://localhost:6379` [src/paperless/settings.py:L456] | `PAPERLESS_REDIS=redis://paperless-broker:6379` (compose-style hostname) | Causes the outage error to be a **DNS** error (`Error -5 … No address associated with hostname`) rather than the `localhost` "Error 99" / a "Connection refused" (§6.2). Recovery semantics identical |
| Container healthcheck | compose `curl -f http://localhost:8000` @30 s [docker/compose/docker-compose.sqlite.yml:L41-45] | dev image has `Healthcheck = null` (no `HEALTHCHECK` in Dockerfile) | The 30 s probe was **reproduced manually** and labeled as such (§5.1) |
| DB log handler | `paperless.log` handler active | `PAPERLESS_DISABLE_DBHANDLER=true` (dev delta) | Does not affect the console/stdout `[Q]` and `[paperless.*]` lines quoted here |

**Non-canonical artifacts explicitly NOT observed here (would appear only in a broken/minimal environment):** startup warnings `"Paperless can't find convert"` / `"optipng"` (missing ImageMagick/optipng); and workers dying with `"reincarnated worker Process-1:N after death"` caused by `ModuleNotFoundError: No module named 'pdf2image'` at the module-load import `from pdf2image import convert_from_path` [src/documents/tasks.py:L23]. Because the canonical image bundles these, none of those artifacts appeared — and they must **not** be presented as default behavior. (Note: the `reincarnated worker … after death` string is the *worker* analogue of the *pusher* reincarnation in §6; in this run reincarnations were caused by the deliberate Redis outage, not by a missing module.)

Environmental deltas to keep in mind if comparing with other setups: **Python 3.12 vs canonical 3.9** and **Redis 7.x vs canonical 6.0** are the usual non-canonical substitutions — neither applies here (this run is 3.9 / 6.0).

---

## 9. Final coverage pass

Every sub-question and every named item is addressed, with observed output + producing command + `[path:line]` citation.

| Question / named item | Where answered | Observed / from-config |
|---|---|---|
| **Q1 Startup / ready** | §3 | observed |
| — boot chain (entrypoint → prepare → supervisord) | §3.1 | inferred-from-code |
| — Redis wait "Connected to Redis broker" | §3.1 | from-config (citations) |
| — gunicorn "Server is ready. Spawning workers" | §3.2 | observed |
| — consumer inotify watch line | §3.3 | observed |
| — qcluster "Q Cluster `<name>` running." + full sequence | §3.4 | observed |
| — cluster display-name vs config `"paperless"` | §3.4 | observed + inferred |
| — worker count = `floor(sqrt(cores))` = 11 | §3.4 | observed + from-config |
| — cross-run stability (2 runs) | §3.4 | observed |
| **Q2 Idle background work** | §4 | observed + from-config |
| — "Train the classifier" (train_classifier, HOURLY) | §4 table, §4.1 | from-config (schedule row observed) |
| — "Optimize the index" (index_optimize, DAILY) | §4 table, §4.1 | from-config (schedule row observed) |
| — "Perform sanity check" (sanity_check, WEEKLY) | §4 table, §4.1 | from-config (schedule row observed; ran once in startup burst §5.3) |
| — "Check all e-mail accounts" (process_mail_accounts, 10 MIN) | §4 table, §4.2 | observed firing |
| — idle mail result "No new documents were added." | §4.3 | observed |
| **Q3 Periodic health/ready logs** | §5 | observed |
| — 30 s healthcheck (`curl -f :8000`), compose-only, silent-in-logs | §5.1 | observed (cadence) + from-config (definition) |
| — 10-min mail-check firing (the log heartbeat), strings + cadence | §5.2 | observed |
| — startup catch-up burst (one-time) | §5.3 | observed |
| — "silent between firings" | §5.4 | observed |
| **Q4 Interrupt/restart recovery** | §6 | observed |
| — before (pusher PID) | §6.1 | observed |
| — during: actual error `Error -5 … No address associated with hostname` | §6.2 | observed |
| — pusher reincarnation `reincarnated pusher Process-1:N after sudden death` | §6.2 | observed |
| — after: error cessation + stable pusher + PING | §6.3 | observed |
| — resumed enqueue (next mail check) | §6.3 | observed |
| — no literal "reconnected" string (composite signature) | §6.4 | observed + inferred |
| — web server survives outage (HTTP 302 during) | §6.5 | observed |
| **Q5 Continuously-running components** | §7 | observed + from-config |
| — gunicorn (web server) | §7.1 | observed |
| — document_consumer (file watcher, silent idle) | §7.1 | observed |
| — qcluster (sentinel + monitor + pusher + N workers) | §7.1 | observed |
| — Redis broker (dual role) | §7.2 | observed |
| — SQLite database (Schedule rows + results) | §7.2 | observed |
| — Channels websocket layer (in gunicorn) | §7.3 | from-config |
| — supervisord (mechanism, not readiness) | §7.4 | from-config + delta noted |
| **`django-q`, not Celery** | Header, §1 | from-config [requirements.txt:L37] |
| **Canonical vs non-canonical labeling** | §2.1, §8 | observed |

---

## 10. Integrity — read-only mandate honored

No source file was created, edited, or deleted. The only change to the repository working tree is the new deliverable; no tracked file differs from `HEAD`:

```
$ git diff --stat HEAD
                     # (empty — zero tracked files changed)

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/paperless-ngx_542221a38dff.md
```

The single untracked entry is this report. (`.gitignore` contains no `blitzy` reference, so the deliverable is tracked by git; creating it created the `blitzy/` and `blitzy/documentation/` directories, the only permitted filesystem side effect.)

**Temporary observation artifacts** were kept **outside** the repository working tree — an in-container scratch dir (`/tmp/blitzy_investigation/`) and a host scratch dir (`/tmp/blitzy_evidence/`) — and were removed during finalization; none was ever inside the checkout, so none could affect the read-only source tree. The Redis broker and the baseline processes were left healthy and recovered after the interrupt/restart experiment (§6.3).

---

### Provenance note on log strings

The `django-q` log strings quoted here (`Q Cluster <name> running.`, `Process-1:N ready for work at <pid>`, `pushing tasks at <pid>`, `Enqueued <n>`, `Process-1 created a task from schedule [<name>]`, `Process-1:N processing [<task-id>]`, `reincarnated pusher Process-1:N after sudden death`, `reincarnated worker Process-1:N after death`) were cross-checked against the `django-q` documentation and the Koed00/django-q issue tracker and confirmed to be **library-canonical** (emitted by `django-q` itself), and each was then **confirmed to appear verbatim in the captured output** above. The randomly-generated humanized cluster **display names** and **task IDs** vary per run and are presented as variable placeholders throughout.
