# Paperless-ngx — Idle Runtime Behavior (Observed Baseline)

**Pinned commit:** `542221a38dff06361e07976452f9aea24d210542`
**Nature of this document:** a *pre-change behavioral baseline*. It answers, from **actual observed runtime output**, how Paperless-ngx behaves once fully started and sitting **idle** (zero documents being processed), *before* any code changes are made.

**How the evidence was produced.** The full stack was built and run at the pinned commit inside the provided Docker image `paperless-ngx-ready:542221a38dff` (derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`), which pins Python 3.9 — `Dockerfile:L18` `FROM python:3.9-slim-bullseye as main-app`. A separate `redis:6` container provided the broker + channel layer. Every behavioral claim below is paired with **one verbatim observed line** and the **command that produced it**, and every literal (string, config key, path, cadence, number) carries a `file:line` citation. Each quoted log line is labelled with its **sink**: `stdout`, `data/log/paperless.log`, or `data/log/mail.log`.

> **Read-only / clean-tree statement.** No existing source file was modified. All observation was performed in a throwaway container and with temporary capture files kept **outside** the repository tree. The only file added to the repository is this document. `git status --porcelain` at the end shows exactly one new entry: `?? blitzy/documentation/paperless-ngx_542221a38dff.md`.

> **Three citation corrections** (the plan's prose drifted; the runtime-verified lines below are authoritative, re-verified with `grep -n`): the Redis reconnection string is at `docker/wait-for-redis.py:L41` (not L44); `PAPERLESS_WORKER_TIMEOUT` default `1800` is at `src/paperless/settings.py:L440` (not L444); the root console handler is `src/paperless/settings.py:L407` with the `loggers` block at `L408`–`L410` (not L406–L411).

---

## Section 1 — O1: Setup and reaching a stable idle state

### 1.1 What "the system" is, and what must be up first

Paperless-ngx runs as **three long-lived application processes** supervised together, plus **two external services** that must already be up. The container image declares the supervisor entrypoint:

- `Dockerfile:L168` `ENTRYPOINT ["/sbin/docker-entrypoint.sh"]`
- `Dockerfile:L170` `EXPOSE 8000`
- `Dockerfile:L172` `CMD ["/usr/local/bin/supervisord", "-c", "/etc/supervisord.conf"]`

Two prerequisites must be reachable before the three app processes can reach idle:

- **Redis** — serves **both** the django-q task broker (`src/paperless/settings.py:L456` `"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")`) **and** the Channels WebSocket channel layer (`src/paperless/settings.py:L182` `"hosts": [os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")]`).
- **Database** — SQLite at `DATA_DIR/db.sqlite3` by default (PostgreSQL only when `PAPERLESS_DBHOST` is set — `docker/docker-prepare.sh:L11` `host="${PAPERLESS_DBHOST:=localhost}"`, branch at `docker/docker-prepare.sh:L67`). This run used the default **SQLite**, so the postgres branch was skipped and `do_work()` went straight to `wait_for_redis` (`docker/docker-prepare.sh:L71`).

**Redis reachability (command + verbatim result), sink `stdout`:**

```text
$ docker exec pl-idle-obs bash -lc 'python3 -c "import redis,os; u=os.environ.get(\"PAPERLESS_REDIS\"); r=redis.from_url(u); print(\"PING ->\", r.ping())"'
PING -> True
```

`PAPERLESS_REDIS` was `redis://paperless-redis:6379` in the environment (the image bakes it in).

### 1.2 The documented startup sequence (what the entrypoint prints)

On a normal container boot the entrypoint prints a banner and then runs the prepare script:

- `docker/docker-entrypoint.sh:L77` `echo "Paperless-ngx docker container starting..."`
- `docker/docker-prepare.sh:L66` `do_work()` runs, in order: DBHOST check (`L67`) → `wait_for_redis` (`L71`) → `migrations` (`L73`) → `search_index` (`L75`) → `superuser` (`L77`); invoked at `L81`.
- `migrations()` prints `docker/docker-prepare.sh:L44` `echo "Apply database migrations..."`.

`python3 manage.py migrate` is what **materializes the four django-q `Schedule` rows** (Section 3). In this environment the image already carries a migrated `db.sqlite3`, so migrations were a no-op; the four schedules were verified present (below). The three programs were then launched directly (the image places the repository at `/app`, not `/usr/src/paperless`, so the stock command in `docker/supervisord.conf:L11` — `gunicorn -c /usr/src/paperless/gunicorn.conf.py …` — was run against the real path `-c /app/gunicorn.conf.py`; the process identities are otherwise identical to the supervisord programs).

**Launch commands (workdir `/app/src`, user `testuser`), stdout captured to `/tmp/obs/*.log` (outside the repo):**

```text
python3 -u manage.py qcluster
python3 -u manage.py document_consumer
gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

### 1.3 Proof the four periodic schedules exist after migrate

**Command + verbatim output**, sink `stdout` (`manage.py shell`):

```text
$ python3 manage.py shell -c "from django_q.models import Schedule
for s in Schedule.objects.all().order_by('id'):
    print(repr(s.name),'|',s.func,'|',s.schedule_type,'|','minutes='+str(s.minutes))
print('TOTAL SCHEDULES:', Schedule.objects.count())"
'Train the classifier' | documents.tasks.train_classifier | H | minutes=None
'Optimize the index' | documents.tasks.index_optimize | D | minutes=None
'Perform sanity check' | documents.tasks.sanity_check | W | minutes=None
'Check all e-mail accounts' | paperless_mail.tasks.process_mail_accounts | I | minutes=10
TOTAL SCHEDULES: 4
```

The schedule-type codes are django-q's: `H`=HOURLY, `D`=DAILY, `W`=WEEKLY, `I`=MINUTES (the mail row also sets `minutes=10`). These map one-to-one to the migrations cited in Section 3.

### 1.4 Definition of "idle" and the three proofs

**Idle** here means: all three programs RUNNING **and** migrations applied **and** the consumption directory empty **and** the task queue drained. (Note: the image shipped with two leftover PDFs baked into `/app/consume`; they were removed before starting the consumer so the system is genuinely document-free. Removing runtime data inside a throwaway container is not a source-file change.)

**(a) All three programs RUNNING** — command + verbatim (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ps -eo pid,ppid,user,args | grep -E "gunicorn|manage.py" | grep -v grep'
     95       0 testuser python3 -u manage.py qcluster
    102       0 testuser python3 -u manage.py document_consumer
    109       0 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    119     109 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    124     109 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

**(b) Consumption directory empty** (sustained; `src/paperless/settings.py:L78` `CONSUMPTION_DIR`) — command + verbatim (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ls -la /app/consume'
total 12
drwxr-sr-x 1 testuser testuser 4096 Jul  1 22:08 .
drwxr-sr-x 1 testuser testuser 4096 Jul  1 21:23 ..
```

**(c) Task queue drained** — command + verbatim (sink `stdout`):

```text
$ python3 manage.py shell -c "from django_q.models import OrmQ; print('OrmQ queued:', OrmQ.objects.count())"
OrmQ queued: 0
```

**Consumer mode = inotify** (default, because `PAPERLESS_CONSUMER_POLLING` defaults to `0` — `src/paperless/settings.py:L478` `CONSUMER_POLLING = int(os.getenv("PAPERLESS_CONSUMER_POLLING", 0))`). This is confirmed by the readiness line quoted in Section 4.

**Web server actually serving on :8000** — command + verbatim (sink `stdout`):

```text
$ python3 -c "import socket,urllib.request; s=socket.socket(); print('TCP :8000 ->', s.connect_ex(('127.0.0.1',8000))==0); print('HTTP', urllib.request.urlopen('http://127.0.0.1:8000/',timeout=5).status)"
TCP :8000 -> True
HTTP 200
```

---

## Section 2 — O5: Components/processes that run continuously to keep the system ready

Even with no documents being processed, the "always-on set" is **three Supervisord-managed programs** plus **two external services**. Supervisord itself runs in the foreground (`docker/supervisord.conf:L2` `nodaemon=true`, `docker/supervisord.conf:L7` `loglevel=info`) and each program's stdout/stderr is wired to the container's streams (`docker/supervisord.conf:L14` `stdout_logfile=/dev/stdout`), which is *why* the django-q lines below surface on container `stdout`.

### 2.1 The three continuously-running programs

| Program | Command | `file:line` | Role | Observed PIDs |
|---|---|---|---|---|
| `[program:gunicorn]` | `gunicorn -c …/gunicorn.conf.py paperless.asgi:application` | `docker/supervisord.conf:L10`, cmd `L11` | ASGI/HTTP web server | master `109` → workers `119`, `124` |
| `[program:consumer]` | `python3 manage.py document_consumer` | `docker/supervisord.conf:L19`, cmd `L20` | consumption-directory watcher | `102` |
| `[program:scheduler]` | `python3 manage.py qcluster` | `docker/supervisord.conf:L28`, cmd `L29` | django-q task cluster + scheduler | `95` → sentinel `133` → 13 children |

The gunicorn program is configured by `gunicorn.conf.py`: `bind` `L3` (`0.0.0.0:8000`), `workers` `L4` (default `2`), `worker_class="paperless.workers.ConfigurableWorker"` `L5`, `timeout=120` `L6`.

**All three running at steady state** — command + verbatim (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ps -eo pid,args | grep -E "manage.py qcluster|manage.py document_consumer|gunicorn -c" | grep -v grep | head -3'
     95 python3 -u manage.py qcluster
    102 python3 -u manage.py document_consumer
    109 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

### 2.2 The two external always-on services

- **Redis** — the single most important dependency: it is **both** the django-q broker (`src/paperless/settings.py:L456`) **and** the Channels layer backend `channels_redis.core.RedisChannelLayer` (`src/paperless/settings.py:L178` `CHANNEL_LAYERS`, `L180` backend, `L182` hosts, `L183` `"capacity": 2000`, `L184` `"expiry": 15`). Its liveness was shown by `PING -> True` in §1.1; Section 5 shows what happens when it is interrupted.
- **Database** (SQLite here) — persists document metadata **and** the four `Schedule` rows the scheduler reads (dumped in §1.3).

The ASGI application itself routes both protocols even while idle — `src/paperless/asgi.py:L17` `application = ProtocolTypeRouter(`, `L19` `"http": get_asgi_application()`, `L20` `"websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` — but the websocket branch is dormant with no client connected (see §3.4).

---

## Section 3 — O2: Automatic background processes/tasks while idle

### 3.1 The django-q `qcluster` process tree (what keeps executing)

The `qcluster` program is not one process; the **sentinel/guard** forks a pool of workers plus a monitor and a pusher. **Command + verbatim forest** (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ps axf -o pid,ppid,user,args | grep qcluster | head'
     95       0 testuser python3 -u manage.py qcluster
    133      95 testuser  \_ python3 -u manage.py qcluster
    146     133 testuser      \_ python3 -u manage.py qcluster
    147     133 testuser      \_ python3 -u manage.py qcluster
    148     133 testuser      \_ python3 -u manage.py qcluster
    149     133 testuser      \_ python3 -u manage.py qcluster
    150     133 testuser      \_ python3 -u manage.py qcluster
    151     133 testuser      \_ python3 -u manage.py qcluster
    ...            (13 children of PID 133 in total)
```

These roles are confirmed by the django-q startup lines the cluster emitted (sink `stdout`, `/tmp/obs/qcluster.log`). All `[Q]` strings are library-emitted by **django-q 1.3.9** (`requirements.txt:L37` `django-q==1.3.9`; verified at runtime: `import django_q; django_q.VERSION == (1, 3, 9)`):

```text
22:08:58 [Q] INFO Q Cluster hotel-social-mississippi-cardinal starting.
22:08:58 [Q] INFO Process-1:1 ready for work at 139
22:08:58 [Q] INFO Process-1:2 ready for work at 140
22:08:58 [Q] INFO Process-1:3 ready for work at 141
22:08:58 [Q] INFO Process-1:4 ready for work at 142
22:08:58 [Q] INFO Process-1:5 ready for work at 143
22:08:58 [Q] INFO Process-1:6 ready for work at 144
22:08:58 [Q] INFO Process-1:7 ready for work at 145
22:08:58 [Q] INFO Process-1:8 ready for work at 146
22:08:58 [Q] INFO Process-1:9 ready for work at 147
22:08:58 [Q] INFO Process-1:10 ready for work at 148
22:08:58 [Q] INFO Process-1:11 ready for work at 149
22:08:58 [Q] INFO Process-1:12 monitoring at 150
22:08:58 [Q] INFO Process-1 guarding cluster hotel-social-mississippi-cardinal
22:08:58 [Q] INFO Process-1:13 pushing tasks at 151
22:08:58 [Q] INFO Q Cluster hotel-social-mississippi-cardinal running.
```

Reading the tree against those lines:

- **11 worker processes** — `Process-1:1 … Process-1:11 ready for work at 139…149`. This equals `TASK_WORKERS`, verified at runtime `TASK_WORKERS = 11` (`src/paperless/settings.py:L438` = `TASK_WORKERS`; `L455` `"workers": TASK_WORKERS`). django-q emits this line from `cluster.py:L410` template `"{name} ready for work at {pid}"`.
- **1 monitor** — `Process-1:12 monitoring at 150` (`cluster.py:L378`). It writes task results back to the DB/broker.
- **1 pusher** — `Process-1:13 pushing tasks at 151` (`cluster.py:L342`). It pulls queued tasks off Redis into the worker pool.
- **1 sentinel / guard** — `Process-1 guarding cluster …` (`cluster.py:L256`); this is PID `133`, the parent of the pool. The **scheduler runs inside this guard loop** — there is no separate scheduler process (`cluster.py:L262` `def guard`).
- **Cluster ready** — `Q Cluster … running.` (`cluster.py:L261`) is the completion marker.

> **Observed nuance — cluster display name is NOT `paperless`.** The startup/running lines show `hotel-social-mississippi-cardinal`, even though `Q_CLUSTER["name"]="paperless"` (`src/paperless/settings.py:L450`). This is not a misconfiguration: in django-q 1.3.9 the *displayed* name is `Cluster.name` → `humanize(self.cluster_id.hex)` (`cluster.py:L110`–`L111`; `humanhash.py:L364`), a **random human-readable id generated per cluster run**; the startup line uses it (`cluster.py:L79` `f"Q Cluster {self.name} starting."`) as does the running line (`cluster.py:L261`). `Q_CLUSTER["name"]="paperless"` is the *internal* cluster identifier (Redis key / stat identity), not the log display name. A second cluster start later in this run produced a *different* random name (`edward-berlin-stairway-hot`, §5.3), confirming the behavior. Reported exactly as observed.

### 3.2 Why `[Q]` lines look different from Paperless's own log lines

django-q installs its **own** logger and formatter, which is why `[Q]` lines carry `HH:MM:SS` and a `[Q]` tag instead of Paperless's verbose format:

- `django_q/conf.py:L207` `logger = logging.getLogger("django-q")`
- `django_q/conf.py:L213`–`L214` `Formatter(fmt="%(asctime)s [Q] %(levelname)s %(message)s", datefmt="%H:%M:%S")`
- `django_q/conf.py:L216`–`L217` its own `StreamHandler`.

Paperless's own loggers instead use the verbose format `"[{asctime}] [{levelname}] [{name}] {message}"` (`src/paperless/settings.py:L378`).

### 3.3 The four periodic schedules (each named, with cadence + migration `file:line`)

All four are created by data migrations and were dumped live in §1.3. **All four were also observed executing** in the initial catch-up burst (their pre-baked `next_run` values were overdue and `catch_up=False`, so each fired exactly once shortly after cluster start) — command + verbatim (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
22:09:28 [Q] INFO Enqueued 1
22:09:28 [Q] INFO Process-1 created a task from schedule [Train the classifier]
22:09:28 [Q] INFO Process-1:1 processing [foxtrot-ohio-black-illinois]
22:09:28 [Q] INFO Enqueued 1
22:09:28 [Q] INFO Process-1 created a task from schedule [Optimize the index]
22:09:28 [Q] INFO Process-1:2 processing [oxygen-romeo-comet-sweet]
22:09:28 [Q] INFO Enqueued 1
22:09:28 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
22:09:28 [Q] INFO Process-1:3 processing [item-indigo-music-victor]
22:09:28 [Q] INFO Enqueued 1
22:09:28 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:09:28 [Q] INFO Process-1:4 processing [moon-comet-yellow-aspen]
```

| # | Schedule name (verbatim) | Task function | Cadence | Migration `file:line` |
|---|---|---|---|---|
| 1 | `Train the classifier` | `documents.tasks.train_classifier` | **HOURLY** (`Schedule.HOURLY`) | `src/documents/migrations/1001_auto_20201109_1636.py:L10-L14` |
| 2 | `Optimize the index` | `documents.tasks.index_optimize` | **DAILY** (`Schedule.DAILY`) | `src/documents/migrations/1001_auto_20201109_1636.py:L15-L19` |
| 3 | `Perform sanity check` | `documents.tasks.sanity_check` | **WEEKLY** (`Schedule.WEEKLY`) | `src/documents/migrations/1004_sanity_check_schedule.py:L10-L14` |
| 4 | `Check all e-mail accounts` | `paperless_mail.tasks.process_mail_accounts` | **every 10 MINUTES** (`Schedule.MINUTES`, `minutes=10`) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15` |

**What each does at idle (verified from `Task.result` rows, sink `stdout` via `manage.py shell`):**

```text
$ python3 manage.py shell -c "from django_q.models import Task
for t in Task.objects.all().order_by('started'):
    print(t.func,'| success='+str(t.success),'| result='+repr(t.result))"
documents.tasks.train_classifier              | success=True | result=None
documents.tasks.index_optimize                | success=True | result=None
documents.tasks.sanity_check                  | success=True | result='No issues detected.'
paperless_mail.tasks.process_mail_accounts    | success=True | result='No new documents were added.'
```

- `train_classifier` returns `None` and logs nothing on a fresh install: it **early-returns** when no Tag/DocumentType/Correspondent has `MATCH_AUTO` (`src/documents/tasks.py:L48`, condition `L50-L52`).
- `index_optimize` returns `None`; it commits a Whoosh optimize (`src/documents/tasks.py:L32`, `writer.commit(optimize=True)` `L35`).
- `sanity_check` returns the literal `"No issues detected."` (`src/documents/tasks.py:L267`); it *also* emitted a Paperless app line (see §4).
- `process_mail_accounts` returns the literal `"No new documents were added."` (`src/paperless_mail/tasks.py:L22`) and, with zero mail accounts configured, never enters its loop body (`src/paperless_mail/tasks.py:L11`), so it logs nothing.

### 3.4 Always-on but idle-silent watchers

- **`document_consumer`** sits in its inotify wait loop `inotify.read(timeout=1000)` (`src/documents/management/commands/document_consumer.py:L216`) and produces no further output after its readiness line (§4). Logger name `paperless.management.consumer` (`:L24`).
- **Channels `StatusConsumer`** (`src/paperless/consumers.py:L9`) only acts on `connect` (`L13`), `disconnect` (`L23`), or `status_update` (`L29`) for the group `"status_updates"` (`L18`). With **no browser client connected**, it is dormant. Verified: Redis holds no `asgi:*` group keys — command + verbatim (sink `stdout`):

```text
$ python3 -c "import redis; r=redis.from_url('redis://paperless-redis:6379'); print('asgi:* keys ->', [k.decode() for k in r.keys('asgi*')] or '(none)')"
asgi:* keys -> (none)
```

### 3.5 Worker recycling (`recycle=1`) is part of the idle self-maintenance

`Q_CLUSTER` sets `"recycle": 1` (`src/paperless/settings.py:L452`), so **a worker is recycled after each task it runs**. Verified live `Q_CLUSTER` (sink `stdout`):

```text
$ python3 manage.py shell -c "from django.conf import settings; print(settings.Q_CLUSTER)"
{'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://paperless-redis:6379'}
```

This corresponds to `catch_up: False` (`L451`), `name: "paperless"` (`L450`), `timeout` (`L454` = `PAPERLESS_WORKER_TIMEOUT` default `1800`, `src/paperless/settings.py:L440`), and `workers` (`L455` = `TASK_WORKERS`, `L438`). The recycle is visible immediately after the catch-up burst (sink `stdout`):

```text
22:09:28 [Q] INFO recycled worker Process-1:1
22:09:28 [Q] INFO Process-1:14 ready for work at 176
```

The worker index keeps climbing (`Process-1:14`, `:15`, …) while the **pool size stays fixed at 11** — recycling replaces a worker rather than growing the pool.


---

## Section 4 — O3: Periodic "healthy/ready" log entries — message + measured frequency + meaning

### 4.1 How the frequency was measured

The idle stack was kept running from **22:08:57 to ~22:48** (~**39 minutes**, well over the 20-minute minimum). Cadence was derived from the timestamps django-q prints on each `[Q]` line (`%H:%M:%S`, §3.2). The single most important empirical finding: **between scheduled task fires, `stdout` is completely silent.** The `qcluster.log` line count was frozen for the entire gap between fires:

```text
# line count of /tmp/obs/qcluster.log, sampled over time (sink stdout)
22:09:29  -> 45 lines   (after startup + catch-up burst)
22:12:37  -> 45 lines   (no change: 3 min silent)
22:17:29  -> 52 lines   (mail fire #1 appended 7 lines)
22:23:40  -> 52 lines   (no change: 6 min silent)
22:27:31  -> 59 lines   (mail fire #2)
22:34:15  -> 59 lines   (no change)
22:37:32  -> 66 lines   (mail fire #3)
```

Consequently the **~30-second django-q scheduler poll produces no log line** when nothing is due. That poll is real but silent: the guard loop calls `scheduler()` only when its accumulator crosses 30 s — `cluster.py:L283-L286` `counter += cycle; if counter >= 30 and Conf.SCHEDULER: counter = 0; scheduler(...)`, with `cycle = Conf.GUARD_CYCLE = 0.5` (`django_q/conf.py:L90`) — and `scheduler()` only emits a line (`created a task from schedule …`, `cluster.py:L669`) when a schedule's `next_run < now`.

### 4.2 The measured periodic-log inventory

| Exact message string (verbatim) | Sink | Measured frequency | What it indicates | Evidence (verbatim line + command) |
|---|---|---|---|---|
| `Process-1 created a task from schedule [Check all e-mail accounts]` (plus the `Enqueued 1` / `processing` / `Processed` / `recycled worker` / `ready for work` burst) | `stdout` | **~10 min** — deltas `22:17:29→22:27:31 = 602 s` and `22:27:31→22:37:32 = 601 s` | The 10-minute mail check ran; it is the **only** recurring idle heartbeat on stdout, and proves scheduler + pusher + worker + monitor + recycle are all alive | two fire blocks + `grep` command below in §4.2 |
| `recycled worker Process-1:N` → `Process-1:M ready for work at <pid>` | `stdout` | **once per task execution** (`recycle=1`) | Cluster self-maintenance: worker replaced after each task | fire blocks below (§4.2); also §3.5 |
| `Q Cluster <name> running.` | `stdout` | **once per cluster start** (not periodic) | Cluster reached ready state | startup block in §3.1; restart block in §5.3 |
| `Using inotify to watch directory for changes: /app/src/../consume` | `stdout` **and** `data/log/paperless.log` | **once at start** (not periodic), then silent | Consumer is ready and watching in inotify mode | `paperless.log` dump in §4.4; restart in §5.3 |
| `Server is ready. Spawning workers` | `stdout` | **once per gunicorn start** (not periodic) | Web server ready | gunicorn block in §5.3 (`gunicorn.conf.py:L18`) |
| (~30 s django-q scheduler poll) | — | **~30 s, but emits NO line** when nothing is due | Scheduler is alive; silent unless a schedule is due | frozen line-count timeline in §4.1 |

**Evidence — two consecutive recurring mail fires (measured ~10 min apart), sink `stdout` (`/tmp/obs/qcluster.log`):**

```text
22:17:29 [Q] INFO Enqueued 1
22:17:29 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:17:29 [Q] INFO Process-1:5 processing [grey-connecticut-bacon-december]
22:17:29 [Q] INFO Process-1:5 stopped doing work
22:17:29 [Q] INFO Processed [grey-connecticut-bacon-december]
22:17:30 [Q] INFO recycled worker Process-1:5
22:17:30 [Q] INFO Process-1:18 ready for work at 378
```

```text
22:27:31 [Q] INFO Enqueued 1
22:27:31 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:27:31 [Q] INFO Process-1:6 processing [october-vegan-hawaii-kilo]
22:27:31 [Q] INFO Process-1:6 stopped doing work
22:27:31 [Q] INFO Processed [october-vegan-hawaii-kilo]
22:27:31 [Q] INFO recycled worker Process-1:6
22:27:31 [Q] INFO Process-1:19 ready for work at 429
```

**Command used to extract every mail-fire timestamp** (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc "grep 'created a task from schedule' /tmp/obs/qcluster.log"
22:09:28 [Q] INFO Process-1 created a task from schedule [Train the classifier]
22:09:28 [Q] INFO Process-1 created a task from schedule [Optimize the index]
22:09:28 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
22:09:28 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:17:29 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:27:31 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:37:32 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

### 4.3 What does NOT appear periodically within the window (reported exactly)

- **Hourly `Train the classifier`** did not recur inside the first ~20 min (its `next_run` after the catch-up fire was `22:47:18`). It **was** observed firing later at `22:47:36` (§5.4). On this clean system its task body is silent (`result=None`), so the only trace is the `[Q]` `created a task from schedule [Train the classifier]` / `processing` / `Processed` wrapper lines.
- **Daily `Optimize the index`** (`next_run 2026-07-02`) and **Weekly `Perform sanity check`** (`next_run 2026-07-08`) will not recur within any ~20-min window; their cadence is stated from the migrations (§3.3). Both were, however, observed executing once in the catch-up burst (§3.3).

### 4.4 The `paperless.*` application sinks while idle

Only two application lines were written during the whole idle window, both from `paperless.*` loggers. Command + verbatim (sink `data/log/paperless.log`):

```text
$ docker exec pl-idle-obs bash -lc 'cat /app/data/log/paperless.log'
[2026-07-01 22:08:58,736] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
[2026-07-01 22:09:28,429] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

Both of these **also** appear on `stdout`. This is by design: the root logger routes to the `console` handler (`src/paperless/settings.py:L407` `"root": {"handlers": ["console"]}`, StreamHandler `L389`, level `INFO` when `DEBUG` is off `L388`, verbose formatter `L390`/`L378`), while the `paperless` logger additionally writes to `paperless.log` (`src/paperless/settings.py:L409` `"paperless": {"handlers": ["file_paperless"], …}`, file at `L395`). Because `disable_existing_loggers` is `False` (`L375`) and no logger sets `propagate=False`, records reach **both** the file handler and (via propagation to root) the console.

> **Observed nuance — `mail.log` is never created at idle.** The `paperless_mail` logger → `mail.log` (`src/paperless/settings.py:L410`, file at `L402`) never fired, so the file does not exist. Command + verbatim (sink `stdout`):
>
> ```text
> $ docker exec pl-idle-obs bash -lc 'ls -la /app/data/log'
> -rw-r--r-- 1 testuser testuser    0 Jul  1 21:41 .__mail.lock
> -rw-r--r-- 1 testuser testuser    0 Jul  1 22:09 .__paperless.lock
> -rw-r--r-- 1 testuser testuser  226 Jul  1 22:09 paperless.log
> ```
>
> Note also that the mail task's logger is `getLogger("paperless.mail.tasks")` (`src/paperless_mail/tasks.py:L8`) — under the `paperless.*` hierarchy, so *if* it logged, it would land in `paperless.log`, not `mail.log`. At idle it emits nothing; `"No new documents were added."` is only a `Task.result` value (§3.3), never a log line in any sink.

---

## Section 5 — O4: Recovery after a brief interrupt/restart

The interrupt/restart steps are **reversible operational actions on running components — no code was edited.** Two kinds of perturbation were exercised: (5.1) briefly stopping the shared **Redis** service, and (5.2–5.3) restarting each of the three application programs.

### 5.1 Interrupting Redis — the broker-loss symptom

**Action + timing** (host clock): `docker stop paperless-redis` at `22:42:16`, `docker start paperless-redis` at `22:43:03` (~47 s outage).

While Redis was down, the running django-q pusher lost its broker connection. **Verbatim reaction** (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
22:42:26 [Q] INFO Process-1:13 stopped pushing tasks
```

(The full traceback showed `redis.exceptions.ConnectionError: Connection closed by server.` raised from `broker.dequeue()` → `blpop`, and once the container's DNS entry disappeared, `Error -5 connecting to paperless-redis:6379. No address associated with hostname.` The `stopped pushing tasks` line is from `cluster.py:L366`.)

> **Reported exactly — the `documents/tasks.py:L231` hint did NOT appear.** The code has `logger.warning("OSError. It could be, the broker cannot be reached.")` (`src/documents/tasks.py:L231`), but that lives inside `sanity_check`'s handler and only fires if a *document* task is running during broker loss. No document task was in flight during the interrupt, so it never triggered. Command + verbatim (sink `stdout`):
>
> ```text
> $ docker exec pl-idle-obs bash -lc 'grep -rn "the broker cannot be reached" /tmp/obs/*.log /app/data/log/*.log || echo ABSENT'
> ABSENT
> ```

### 5.2 Redis reconnection marker

After `docker start paperless-redis`, the canonical readiness check (`docker/wait-for-redis.py`, exactly as the entrypoint uses it) reconnected. **Command + verbatim** (sink `stdout`):

```text
$ docker exec -u testuser -w /app/src pl-idle-obs bash -lc 'python3 /app/docker/wait-for-redis.py'
Waiting for Redis: redis://paperless-redis:6379
Connected to Redis broker: redis://paperless-redis:6379
```

- `"Waiting for Redis: {REDIS_URL}"` — `docker/wait-for-redis.py:L21`
- `"Connected to Redis broker: {REDIS_URL}"` — **`docker/wait-for-redis.py:L41`** (corrected line; the plan text said L44). Related constants: `MAX_RETRY_COUNT=5` (`:L16`), `RETRY_SLEEP_SECONDS=5` (`:L17`); failure paths `"Redis ping #{attempt} failed, waiting {RETRY_SLEEP_SECONDS}s"` (`:L31`) and `"Failed to connect to: {REDIS_URL}"` (`:L38`).

**The running cluster self-healed without a restart** — the guard reincarnated the dead pusher and it resumed once Redis returned. Verbatim (sink `stdout`):

```text
22:43:05 [Q] INFO Process-1:21 stopped pushing tasks
22:43:06 [Q] ERROR reincarnated pusher Process-1:21 after sudden death
22:43:06 [Q] INFO Process-1:22 pushing tasks at 642
```

`Process-1:22 pushing tasks at 642` (a fresh pusher PID) confirms the broker path is operational again.

### 5.3 Restarting the three programs — each readiness marker

Each program was stopped (via `pkill` inside the container's isolated PID namespace) and relaunched; each re-emitted its readiness marker.

**Scheduler / qcluster** — verbatim (sink `stdout`, `/tmp/obs/qcluster_restart.log`):

```text
22:44:06 [Q] INFO Q Cluster edward-berlin-stairway-hot starting.
22:44:06 [Q] INFO Process-1:12 monitoring at 706
22:44:06 [Q] INFO Process-1 guarding cluster edward-berlin-stairway-hot
22:44:06 [Q] INFO Process-1:13 pushing tasks at 707
22:44:06 [Q] INFO Q Cluster edward-berlin-stairway-hot running.
```

The completion marker `Q Cluster edward-berlin-stairway-hot running.` (`cluster.py:L261`) confirms the cluster is operational again — and its **different random name** (vs. `hotel-social-mississippi-cardinal`) re-confirms the §3.1 finding that the display name is a per-run humanized id, not `Q_CLUSTER["name"]`.

**Consumer** — verbatim (sink `stdout`, `/tmp/obs/consumer_restart.log`):

```text
[2026-07-01 22:44:29,350] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

Re-emits the readiness line from `src/documents/management/commands/document_consumer.py:L200`.

**Web server / gunicorn** — verbatim (sink `stdout`, `/tmp/obs/gunicorn_restart.log`):

```text
[2026-07-01 22:44:38 +0000] [766] [INFO] Starting gunicorn 20.1.0
[2026-07-01 22:44:38 +0000] [766] [INFO] Listening at: http://0.0.0.0:8000 (766)
[2026-07-01 22:44:38 +0000] [766] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 22:44:38 +0000] [766] [INFO] Server is ready. Spawning workers
```

`Server is ready. Spawning workers` is the `when_ready(server)` hook — `gunicorn.conf.py:L17`–`L18`.

### 5.4 Proof the whole stack is operational again

At the first ~30 s scheduler poll after the restarts (and after the Redis blip), **both** then-due schedules fired and completed — verbatim (sink `stdout`, `/tmp/obs/qcluster_restart.log`):

```text
22:47:36 [Q] INFO Enqueued 1
22:47:36 [Q] INFO Process-1 created a task from schedule [Train the classifier]
22:47:36 [Q] INFO Process-1:1 processing [east-golf-twenty-floor]
22:47:36 [Q] INFO Enqueued 1
22:47:36 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:47:36 [Q] INFO Process-1:2 processing [diet-edward-vermont-music]
22:47:36 [Q] INFO Processed [diet-edward-vermont-music]
22:47:36 [Q] INFO Processed [east-golf-twenty-floor]
```

The queue drained back to zero afterward — command + verbatim (sink `stdout`):

```text
$ python3 manage.py shell -c "from django_q.models import OrmQ; print('OrmQ queued now:', OrmQ.objects.count())"
OrmQ queued now: 0
```

This is the recovery signature end-to-end: **Redis reconnected → cluster running → consumer watching → gunicorn serving → scheduled tasks firing and draining again.**


---

## Section 6 — Coverage pass

Re-reading the original question and confirming every distinct sub-question and every named item is answered above:

- [x] **"what background processes or tasks continue executing automatically?"** → the django-q `qcluster` process tree (sentinel/guard + 11 workers + monitor + pusher) and the four periodic schedules — §3.1, §3.3.
- [x] **Each of the four periodic tasks by name, with cadence:**
  - [x] `Train the classifier` — HOURLY (`src/documents/migrations/1001_auto_20201109_1636.py:L10-L14`) — §3.3.
  - [x] `Optimize the index` — DAILY (`src/documents/migrations/1001_auto_20201109_1636.py:L15-L19`) — §3.3.
  - [x] `Perform sanity check` — WEEKLY (`src/documents/migrations/1004_sanity_check_schedule.py:L10-L14`) — §3.3.
  - [x] `Check all e-mail accounts` — every 10 MINUTES (`src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15`) — §3.3, §4.2.
- [x] **"the actual log entries that appear periodically … the specific log messages, their frequency, and what they indicate"** → the O3 table with **measured** cadences (mail burst deltas 602 s and 601 s ≈ 10 min; scheduler ~30 s but silent) — §4.1, §4.2, each row backed by a verbatim line with its sink.
- [x] **"if you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again?"** → the Redis interrupt + `Connected to Redis broker: …` (`docker/wait-for-redis.py:L41`), cluster self-heal (`reincarnated pusher …` → `… pushing tasks at 642`), `Q Cluster … running.`, `Using inotify to watch directory for changes: …`, `Server is ready. Spawning workers`, and the end-to-end task-firing proof — §5.1–§5.4.
- [x] **"What components or processes keep running continuously … even when no documents are being processed?"** → the three Supervisord programs (gunicorn / consumer / scheduler) plus Redis and the database — §2.1, §2.2.
- [x] **Idle-only scope** — every quoted heartbeat is from the document-free state (empty consume dir, drained queue); no ingestion/OCR line is reported as idle behavior.
- [x] **Measured, not inferred, frequency** — the run exceeded ~39 minutes; the 10-minute mail task recurred at 22:17:29 / 22:27:31 / 22:37:32 (three recurring fires, two clean ~600 s deltas).
- [x] **Read-only & cleanup** — no source file was modified; all temporary capture lived outside the repo tree; the working tree ends clean with only this new document.

### Anticipated-vs-observed corrections (reported exactly as observed)

- The cluster's displayed name is a **random humanized id** (`hotel-social-mississippi-cardinal`, later `edward-berlin-stairway-hot`), **not** `paperless`; `Q_CLUSTER["name"]="paperless"` (`src/paperless/settings.py:L450`) is the internal identifier only — §3.1.
- `"No new documents were added."` is a **`Task.result` value**, not a stdout/file log line; `mail.log` is never created at idle — §3.3, §4.4.
- The ~30-second scheduler poll is real but **emits no log line** when nothing is due — §4.1.
- The `documents/tasks.py:L231` "OSError…" broker-loss hint did **not** appear (no document task was in flight during the Redis interrupt) — §5.1.

---

## Appendix — Version pins and environment (for reproducibility)

Observed at commit `542221a38dff06361e07976452f9aea24d210542`; base image `python:3.9-slim-bullseye` (`Dockerfile:L18`). Runtime pins (from `requirements.txt`, verified imported at runtime):

| Package | Version | `requirements.txt` |
|---|---|---|
| django | 4.0.4 | `L38` |
| django-q | 1.3.9 | `L37` |
| channels | 3.0.4 | `L23` |
| channels-redis | 3.4.0 | `L22` |
| daphne | 3.0.2 | `L31` |
| redis (py client) | 3.5.3 | `L84` |
| gunicorn | 20.1.0 | `L42` |
| watchdog | 2.1.7 | `L106` |
| inotifyrecursive | 0.3.5 | `L54` |
| whitenoise | 6.0.0 | `L110` |
| whoosh | 2.7.4 | `L111` |

**External reference (for interpretation only, not observed output):** the django-q 1.3 official documentation — used to interpret cluster statuses (Starting/Idle/Working/Stopping/Stopped) and that the scheduler checks for due tasks roughly twice a minute. All `[Q]` strings quoted above are captured verbatim from this run, emitted by `django-q==1.3.9`.
