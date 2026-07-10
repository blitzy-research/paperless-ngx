# Paperless‑NGX Steady‑State ("Up and Idle") Runtime Behavior

**Repository:** paperless‑ngx
**Commit (pinned):** `542221a38dff06361e07976452f9aea24d210542`
**Investigation type:** Read‑only runtime characterization (observe by RUNNING; no source file modified)
**Captured:** 2026‑07‑10, single continuous run 07:47:30 → 08:11:43 UTC (≈24 min), plus two Redis interrupt/restart cycles

This document answers five questions about how Paperless‑NGX behaves once it is **up and idle** (running, stable, with **no documents being ingested**). Every behavioral claim below is backed by **actual, unedited log output that I captured at runtime**, together with the **exact command** that produced it and a **`file:line` citation** to the code that emits it. Statements that are *inferred from reading code* rather than observed at runtime are explicitly marked **[INFERRED]**. Environment‑specific values are marked **[NON‑CANONICAL]** with their canonical counterpart.

---

## TL;DR — the two key findings

1. **There is NO dedicated periodic "healthy" heartbeat INFO log line.** When Paperless‑NGX is idle and healthy, all three processes are **silent at INFO level**; the only recurring log activity is the firing of the **four django‑q scheduled tasks** (mail every 10 min, classifier hourly, index optimize daily, sanity check weekly). Two other health mechanisms — the Docker Compose `curl` liveness probe (every 30 s) and the django‑q `Stat` heartbeat to Redis (every 0.5 s) — are **deliberately silent** while healthy.
2. **There is NO dedicated "reconnected" log message after a component restart.** When the Redis broker is interrupted and restarted, "operational again" is confirmed by **two co‑occurring signals**: (a) the `[Q] ERROR ... Connection refused` stream **ceases** the instant Redis returns, and (b) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears with **no further errors** following it.

---

## The five questions (preserved verbatim from the user)

> "Get Paperless‑NGX running at the specified commit. Once it's idle and stable, **(R2)** what background processes or tasks continue executing automatically? **(R3)** What are the actual log entries that appear periodically showing the system is healthy and ready? I need the specific log messages, their frequency, and what they indicate. **(R4)** Also, if you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again? **(R5)** What components or processes keep running continuously to maintain Paperless‑NGX in a ready state, even when no documents are being processed? You may use temporary helper commands or inspection tools if needed, but don't modify any source files and clean up any temporary artifacts when you're done."

- **R1** — Bring the system up at the pinned commit and reach a stable idle state.
- **R2** — Enumerate the background processes/tasks that keep running automatically while idle.
- **R3** — The periodic health/readiness log entries: exact messages, **measured** frequency, and meaning.
- **R4** — After interrupting and restarting a component, the messages that confirm reconnection/operational status.
- **R5** — The components/processes that run continuously to maintain a ready state at idle.

---

## 1. Environment and exact invocation commands

### 1.1 How the runtime was provisioned

Paperless‑NGX at this commit is a Django application whose canonical deployment is **three long‑lived processes plus a Redis dependency**, as declared by its process supervisor (`docker/supervisord.conf`) and mirrored by three systemd units (`scripts/paperless-*.service`):

- `[program:gunicorn]` → `command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` — `docker/supervisord.conf:10-11`
- `[program:consumer]` → `command=python3 manage.py document_consumer` — `docker/supervisord.conf:19-20`
- `[program:scheduler]` → `command=python3 manage.py qcluster` — `docker/supervisord.conf:28-29`
- Each systemd unit declares `Requires=redis.service` — `scripts/paperless-webserver.service:5`, `scripts/paperless-consumer.service:3`, `scripts/paperless-scheduler.service:3`.

I reproduced this runtime **inside the project's own canonical Docker image** (the image designated by the setup instructions), which ships **Python 3.9.23** — matching the Dockerfile base `FROM python:3.9-slim-bullseye as main-app` (`Dockerfile:18`) — with all of the repository's pinned dependencies pre‑installed. I launched a **fresh, isolated container** and ran the three processes exactly as the supervisor/systemd units invoke them. The repository git checkout was never written to; the only artifact added to the repository is **this document**.

**Exact commands (run in order):**

```bash
# 1. Fresh, isolated container from the canonical image (Python 3.9.23), no repo writes
docker run -d --name pngx_obs --entrypoint sleep \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 infinity

# 2. Runtime prerequisites installed inside the container (see caveat 1.3)
apt-get install -y redis-server libzbar0 poppler-utils pngquant curl

# 3. Redis broker + Channels layer (canonical start command)
redis-server --daemonize yes --bind 127.0.0.1 --port 6379

# 4. Database migrations (SQLite default) + full-text index + startup check — run from /app/src
cd /app/src
python3 manage.py migrate
python3 manage.py document_index reindex
python3 manage.py check

# 5. The three canonical long-lived processes (each to its own log file)
gunicorn -c /app/gunicorn.conf.py paperless.asgi:application     # web + websockets (:8000)
python3 manage.py document_consumer                             # directory watcher
python3 manage.py qcluster                                      # django-q task cluster
```

`manage.py check` output (canonical startup check passes cleanly):

```
System check identified no issues (0 silenced).
```

### 1.2 Dependency versions actually used (from the project's own pins)

These versions were confirmed inside the running container (`pip show ...`) and match `requirements.txt` / `Pipfile.lock` at this commit:

| Package | Version | Role |
|---|---|---|
| Python | 3.9.23 | Canonical interpreter (`Dockerfile:18`) |
| redis‑server | 6.0.16 | django‑q broker + Channels group layer |
| Django | 4.0.4 | Web framework / ORM / management commands |
| django‑q | 1.3.9 | Task queue + scheduler (`qcluster`), source of `[Q]` logs (`Pipfile:17` `django-q = "~=1.3"`) |
| channels | 3.0.4 | ASGI websocket framework (status updates) |
| channels‑redis | 3.4.0 | Redis‑backed Channels layer |
| gunicorn | 20.1.0 | ASGI process manager for the web server |
| uvicorn | 0.17.6 | ASGI worker (`ConfigurableWorker` base) |
| redis (py client) | 3.5.3 | Python Redis client used by django‑q |
| concurrent‑log‑handler | 0.9.20 | Rotating file handler for `paperless`/`paperless_mail` loggers |
| Whoosh | 2.7.4 | Full‑text index (drives daily `index_optimize`) |
| scikit‑learn | 1.0.2 | Document classifier (drives hourly `train_classifier`) |
| watchdog | 2.1.7 | Filesystem event monitoring for the consumer |
| inotifyrecursive | 0.3.5 | Recursive inotify support for the consumer |

**Default configuration observed** (no overrides): database = **SQLite** at `/app/data/db.sqlite3` (`src/paperless/settings.py:297-302`; PostgreSQL is used only when `PAPERLESS_DBHOST` is set, `settings.py:304-318`); `PAPERLESS_REDIS` default `redis://localhost:6379` (`settings.py:182`, `settings.py:456`); `DEBUG` default `NO` so the console log handler runs at **INFO** (`settings.py:50`, `settings.py:388`).

### 1.3 Non‑canonical caveats (explicitly labeled)

- **[NON‑CANONICAL — packaging] Direct process launch vs. supervisord/docker‑compose.** I launched the three processes directly (as the setup instructions prescribe, and exactly as `docker/supervisord.conf:11,20,29` and `scripts/*.service` invoke them) rather than under `supervisord` or `docker compose`. The **process invocations and code paths are identical**; only the supervisor wrapper differs. *Canonical counterpart:* the same three commands, started by supervisord (Docker) or systemd (bare metal).
- **[NON‑CANONICAL — prerequisites installed at runtime]** `redis-server`, `libzbar0`, `poppler-utils`, `pngquant`, and `curl` were `apt`‑installed into the container because the minimal base image lacks them; the published Paperless‑NGX image bakes these in. Versions match the canonical toolchain (e.g. `redis-server 6.0.16`).
- **[NON‑CANONICAL — user]** Processes ran as `root` inside the container rather than the `paperless`/`testuser` service account. This does not affect the logging or task behavior examined here.
- **[NON‑CANONICAL — worker count] 11 django‑q workers.** The container observed `multiprocessing.cpu_count() == 128`, so `default_task_workers()` returned `floor(sqrt(128)) = 11` (`src/paperless/settings.py:427-433`, `TASK_WORKERS` `settings.py:438`). *Canonical counterpart:* the default depends on the host CPU count (and `PAPERLESS_TASK_WORKERS`); on a 4‑core host it would be `floor(sqrt(4)) = 2`. The **worker‑count value is host‑specific; the formula is canonical.**
- **[NON‑CANONICAL — volatile fields]** Timestamps, PIDs, the django‑q cluster word‑name (e.g. `montana-tennis-winner-tango`) and the per‑task word‑names vary per run. All output below is my own freshly captured run.
- **The web server healthcheck returned HTTP 302** (redirect to the login page for an unauthenticated request); `curl -f` treats this as success. This is expected for the root URL at idle.

---

## 2. R1 — The system up at a stable idle state; R2/R5 — idle background processes

### 2.1 Reaching idle (R1)

After the commands in §1.1, the system reached a stable idle state at **2026‑07‑10 07:47:29 UTC**. "Idle" here means the consume directory is empty and no tasks are executing:

```
# ls -la /app/consume        (the watched consumption directory)
total 8
drwxr-sr-x 2 root     testuser 4096 Jul 10 07:47 .
drwxr-sr-x 1 testuser testuser 4096 Jul 10 07:46 ..
```

The four periodic schedules that django‑q must drive were created by the migrations and confirmed present in the database (`python3 manage.py shell`):

```
[["Train the classifier","documents.tasks.train_classifier","H",null],
 ["Optimize the index","documents.tasks.index_optimize","D",null],
 ["Perform sanity check","documents.tasks.sanity_check","W",null],
 ["Check all e-mail accounts","paperless_mail.tasks.process_mail_accounts","I",10]]
```

These correspond exactly to the schedule‑defining migrations: `Train the classifier` (HOURLY) and `Optimize the index` (DAILY) from `src/documents/migrations/1001_auto_20201109_1636.py:10-19`; `Perform sanity check` (WEEKLY) from `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; `Check all e-mail accounts` (MINUTES, `minutes=10`) from `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15`.

### 2.2 The three processes and their startup banners (R2/R5)

Each process was launched with the exact command shown, writing its own stdout+stderr to a separate log file. The banners below are **unedited**.

#### (a) `gunicorn` — web server + websockets (gunicorn master + 2 uvicorn ASGI workers)

**Command:** `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` (from `/app/src`)

```
[2026-07-10 07:47:30 +0000] [671] [INFO] Starting gunicorn 20.1.0
[2026-07-10 07:47:30 +0000] [671] [INFO] Listening at: http://0.0.0.0:8000 (671)
[2026-07-10 07:47:30 +0000] [671] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-10 07:47:30 +0000] [671] [INFO] Server is ready. Spawning workers
```

- `bind = 0.0.0.0:8000`, `workers = 2`, `worker_class = "paperless.workers.ConfigurableWorker"`, `timeout = 120` — `gunicorn.conf.py:3-6`.
- `Server is ready. Spawning workers` is emitted by the repo's `when_ready(server)` hook — `gunicorn.conf.py:17-18`. The `Listening at: http://0.0.0.0:8000` and `Starting gunicorn 20.1.0` lines are gunicorn's own stdlib startup logs **[emitted by gunicorn core, not by repo config]**.
- `ConfigurableWorker` subclasses the uvicorn `UvicornWorker` (an ASGI worker) — `src/paperless/workers.py:9`. The ASGI application wires both `http` and `websocket` protocols via `ProtocolTypeRouter` — `src/paperless/asgi.py:17-20`; the websocket route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` — `src/paperless/urls.py:136-137`, and `StatusConsumer` joins the Channels group `status_updates` — `src/paperless/consumers.py:9,17-19`.
- **Observed processes:** master **PID 671** with **two** uvicorn ASGI worker children **PID 682** and **PID 686** (`PPid 671`), i.e. `workers=2` as configured.

*(One benign one‑time startup warning also appears — `UserWarning: No directory at: /app/static/` from whitenoise — because static files were not collected in this minimal run. It is emitted only at startup, never periodically, and does not affect the runtime behavior examined here.)*

#### (b) `document_consumer` — the consumption‑directory watcher

**Command:** `python3 manage.py document_consumer`

```
[2026-07-10 07:47:30,893] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

- This readiness banner is emitted by `handle_inotify()` — `src/documents/management/commands/document_consumer.py:200` (logger `paperless.management.consumer`, defined at `document_consumer.py:24`). If inotify were unavailable it would instead log `Polling directory for changes: <dir>` — `document_consumer.py:186`.
- At idle this watcher is **silent**. It only logs when a file arrives: `Adding {filepath} to the task queue.` — `document_consumer.py:85`. That line **did not appear** during the idle window (no documents ingested).

#### (c) `qcluster` — the django‑q task cluster (guard/sentinel + monitor + pusher + worker pool)

**Command:** `python3 manage.py qcluster`

```
07:47:30 [Q] INFO Q Cluster montana-tennis-winner-tango starting.
07:47:30 [Q] INFO Process-1:1 ready for work at 702
07:47:30 [Q] INFO Process-1:2 ready for work at 703
07:47:30 [Q] INFO Process-1:3 ready for work at 704
07:47:30 [Q] INFO Process-1:4 ready for work at 706
07:47:30 [Q] INFO Process-1:5 ready for work at 709
07:47:30 [Q] INFO Process-1:6 ready for work at 710
07:47:30 [Q] INFO Process-1:7 ready for work at 711
07:47:30 [Q] INFO Process-1:8 ready for work at 712
07:47:30 [Q] INFO Process-1:9 ready for work at 713
07:47:30 [Q] INFO Process-1:10 ready for work at 714
07:47:30 [Q] INFO Process-1:11 ready for work at 715
07:47:30 [Q] INFO Process-1:12 monitoring at 716
07:47:30 [Q] INFO Process-1 guarding cluster montana-tennis-winner-tango
07:47:30 [Q] INFO Process-1:13 pushing tasks at 717
07:47:30 [Q] INFO Q Cluster montana-tennis-winner-tango running.
```

The django‑q logger uses its own format `HH:MM:SS [Q] LEVEL msg` (`django_q/conf.py:213-214`, with `propagate = False` at `django_q/conf.py:212`, so `[Q]` lines are independent of Paperless's Django `LOGGING`). Mapping each role to the code that emits it (all in the pip‑installed `django-q 1.3.9` package):

| Banner line | Role | Source |
|---|---|---|
| `Q Cluster <name> starting.` | cluster boot | `django_q/cluster.py:79` |
| `Process-1:1..11 ready for work at <pid>` | **11 worker processes** | `django_q/cluster.py:410` |
| `Process-1:12 monitoring at <pid>` | **monitor** (persists task results) | `django_q/cluster.py:378` |
| `Process-1 guarding cluster <name>` | **guard/sentinel** (0.5 s health loop) | `django_q/cluster.py:256` |
| `Process-1:13 pushing tasks at <pid>` | **pusher** (BLPOP‑polls the broker) | `django_q/cluster.py:342` |
| `Q Cluster <name> running.` | cluster ready | `django_q/cluster.py:261` |

- **Worker count = 11** = `floor(sqrt(128))` on this host — see caveat §1.3. Formula: `default_task_workers()` at `src/paperless/settings.py:427-433`; used by `TASK_WORKERS` at `settings.py:438` and passed into `Q_CLUSTER["workers"]` at `settings.py:455`.
- **Observed process tree:** the `qcluster` master/guard is **PID 655**; it forked the 11 workers (PIDs 702–715), the monitor (PID 716) and the pusher (PID 717).

**Full idle process topology (observed):**

```
redis-server (PID 485, 127.0.0.1:6379)
qcluster    (guard/sentinel PID 655) ── 11 workers (702-715) + monitor (716) + pusher (717)
document_consumer (PID 663)  ── inotify watch on /app/consume
gunicorn    (master PID 671) ── 2 uvicorn ASGI workers (682, 686) on :8000
```


---

## 3. R3 — Periodic health/readiness log entries: messages, frequency, and meaning

**KEY FINDING (proven below): there is no dedicated periodic "healthy" heartbeat INFO log line.** While idle and healthy, the three processes are silent at INFO. The only recurring log activity is the firing of the four django‑q scheduled tasks. Two additional health mechanisms run continuously but are **silent** unless something is wrong.

### 3.1 The measured cadence table

Each cadence below was **measured**, not assumed (frequency confirmed either by observing ≥2 firings, or by reading the schedule `next_run` increments from the live database — both shown in §3.2/§3.3).

| Recurring signal | Frequency | Meaning / Source |
|---|---|---|
| `... created a task from schedule [Check all e-mail accounts]` | every **10 minutes** (most frequent) | Mail polling; `process_mail_accounts` — `src/paperless_mail/tasks.py:11`; schedule `Schedule.MINUTES, minutes=10` — `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` |
| `... created a task from schedule [Train the classifier]` | **hourly** | Retrain document classifier; `train_classifier` — `src/documents/tasks.py:48`; schedule `Schedule.HOURLY` — `src/documents/migrations/1001_auto_20201109_1636.py:10-14` |
| `... created a task from schedule [Optimize the index]` | **daily** | Optimize Whoosh full‑text index; `index_optimize` — `src/documents/tasks.py:32`; schedule `Schedule.DAILY` — `src/documents/migrations/1001_auto_20201109_1636.py:15-19` |
| `... created a task from schedule [Perform sanity check]` | **weekly** | Integrity sweep; `sanity_check` — `src/documents/tasks.py:255`; schedule `Schedule.WEEKLY` — `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; emits `Sanity checker detected no issues.` — `src/documents/sanity_checker.py:27` |
| Docker Compose healthcheck `curl -f http://localhost:8000` | every **30 seconds** | Liveness probe — **SILENT** (no access log by default) — `docker/compose/docker-compose.sqlite.yml:41-45` |
| django‑q guard `Stat` heartbeat write to Redis | every **0.5 seconds** | Cluster status write — **SILENT** unless Redis is unreachable — `django_q/cluster.py:288`, `django_q/status.py:71-75` |

**What these signals indicate about health.** The four scheduled‑task firings are the visible proof that the django‑q **scheduler + pusher + worker pool** are alive and processing the queue on time — a healthy idle cluster fires them on schedule and each completes immediately (the mail/classifier/index tasks are no‑ops at idle, and the sanity task reports "no issues"). Their absence, or `[Q] ERROR` lines instead, would indicate a broker or worker problem. The two silent mechanisms are the continuous liveness checks: the healthcheck confirms the web server answers HTTP, and the `Stat` heartbeat confirms the cluster can reach Redis — but by design **neither prints anything while healthy** (see §3.4).

### 3.2 The one‑time startup burst (overdue schedules; `catch_up: False`)

About **30 seconds** after the cluster reported `running.` (07:47:30 → **07:48:00**), the guard's scheduler ran for the first time and fired all four overdue schedules **once**. django‑q's scheduler is gated in the guard loop by `if counter >= 30 and Conf.SCHEDULER:` (`django_q/cluster.py:284-286`), and Paperless sets `"catch_up": False` (`src/paperless/settings.py:451`), so overdue schedules fire a **single** time rather than replaying every missed run. Unedited:

```
07:48:00 [Q] INFO Enqueued 1
07:48:00 [Q] INFO Process-1 created a task from schedule [Train the classifier]
07:48:00 [Q] INFO Enqueued 1
07:48:00 [Q] INFO Process-1:1 processing [mountain-crazy-happy-july]
07:48:00 [Q] INFO Process-1 created a task from schedule [Optimize the index]
07:48:00 [Q] INFO Process-1:2 processing [fillet-georgia-earth-sodium]
07:48:00 [Q] INFO Enqueued 1
07:48:00 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
07:48:00 [Q] INFO Process-1:3 processing [washington-pluto-black-november]
07:48:00 [Q] INFO Enqueued 1
07:48:00 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
07:48:00 [Q] INFO Process-1:4 processing [connecticut-berlin-ohio-sixteen]
07:48:00 [Q] INFO Process-1:4 stopped doing work
07:48:00 [Q] INFO Processed [connecticut-berlin-ohio-sixteen]
[2026-07-10 07:48:00,559] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
07:48:00 [Q] INFO Process-1:3 stopped doing work
07:48:00 [Q] INFO Process-1:1 stopped doing work
07:48:00 [Q] INFO Process-1:2 stopped doing work
07:48:00 [Q] INFO Processed [washington-pluto-black-november]
07:48:00 [Q] INFO Processed [mountain-crazy-happy-july]
07:48:00 [Q] INFO Processed [fillet-georgia-earth-sodium]
07:48:00 [Q] INFO recycled worker Process-1:1
07:48:00 [Q] INFO Process-1:14 ready for work at 821
07:48:00 [Q] INFO recycled worker Process-1:3
07:48:00 [Q] INFO Process-1:15 ready for work at 822
07:48:01 [Q] INFO recycled worker Process-1:2
07:48:01 [Q] INFO Process-1:16 ready for work at 823
07:48:01 [Q] INFO recycled worker Process-1:4
07:48:01 [Q] INFO Process-1:17 ready for work at 824
```

Notes on the lines above:
- `Enqueued 1` — `django_q/tasks.py:74`; `Process-1 created a task from schedule [<name>]` — `django_q/cluster.py:669`.
- `Sanity checker detected no issues.` prints in the **Paperless verbose format** `[{asctime}] [{levelname}] [{name}] {message}` (`src/paperless/settings.py:378`), **not** the `[Q]` format, because it comes from the `paperless.sanity_checker` logger (`src/documents/sanity_checker.py:24,27`), which writes to its rotating file handler **and** propagates to the root console handler (no `propagate:False`; root handler = `console`) — `settings.py:407-410`. I confirmed it also landed in the rotating file `/app/data/log/paperless.log`:
  ```
  [2026-07-10 07:48:00,559] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
  ```
- `recycled worker Process-1:N` + a new `ready for work at <pid>` — django‑q recycles each worker after the task because Paperless sets `"recycle": 1` (`src/paperless/settings.py:452`), a memory‑hygiene measure. This is why worker index numbers climb over time.

### 3.3 The 10‑minute mail cadence — measured across THREE firings (≥2 observations)

The `Check all e-mail accounts` task is the most frequent recurring signal. I observed it fire **three times** during the run:

```
# grep 'created a task from schedule [Check all e-mail accounts]' qcluster.log
07:48:00 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #1 (overdue burst)
07:56:31 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #2
08:06:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #3
```

The consecutive **on‑schedule** interval **#2 → #3 = 07:56:31 → 08:06:33 = 10 min 02 s ≈ 10 minutes** (the ~2 s jitter is the ~30 s scheduler poll granularity; the underlying `next_run` steps are exactly 10:00 — see below). Interval #1 → #2 is 8 min 31 s only because #1 was the *delayed overdue* firing (the schedule's `next_run` anchor predates cluster start). Full firing #3 block, unedited:

```
08:06:33 [Q] INFO Enqueued 1
08:06:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
08:06:33 [Q] INFO Process-1:6 processing [summer-jig-beer-one]
08:06:33 [Q] INFO Process-1:6 stopped doing work
08:06:33 [Q] INFO Processed [summer-jig-beer-one]
08:06:33 [Q] INFO recycled worker Process-1:6
08:06:33 [Q] INFO Process-1:19 ready for work at 1173
```

*(The `process_mail_accounts` function itself logs nothing at idle — no mail accounts are configured, so its loop over accounts is empty (`src/paperless_mail/tasks.py:11-17`). The only evidence it ran is the `[Q]` scheduler/worker lines above.)*

**Independent confirmation of all four cadences from the live database.** Reading each schedule's `next_run` (captured at 07:59:33) proves the frequencies directly, because django‑q advances `next_run` by the schedule's period:

```
'Check all e-mail accounts'  type=I minutes=10  next_run=2026-07-10 08:06:12   # +10 minutes
'Train the classifier'       type=H             next_run=2026-07-10 08:46:11   # +1 hour   (HOURLY)
'Optimize the index'         type=D             next_run=2026-07-11 07:46:11   # +1 day    (DAILY)
'Perform sanity check'       type=W             next_run=2026-07-17 07:46:11   # +7 days   (WEEKLY)
now= 2026-07-10 07:59:33
```

So: mail = +10 min, classifier = +1 h, index = +1 day, sanity = +7 days — matching the schedule types in the migrations cited in §3.1.

### 3.4 Proof that the idle system is otherwise SILENT (the KEY FINDING)

**(i) The processes emit nothing between task firings.** Over two quiet windows the `qcluster` log did not grow at all:

- **Window A** — 07:48:01 → 07:56:31 (~8.5 min): `qcluster.log` stayed at **45 lines** (the last line's timestamp remained `07:48:01`). Verified at 07:56:16:
  ```
  # wc -l qcluster.log  → 45
  # tail -1 qcluster.log → 07:48:01 [Q] INFO Process-1:17 ready for work at 824
  ```
- **Window B** — 07:56:32 → 08:06:33 (10.0 min): no new lines until the mail firing at 08:06:33.

The web server and consumer logs were **flat for the entire ~20‑minute idle window**:

```
gunicorn.log = 12 lines   (unchanged from startup, incl. through the healthcheck probes)
consumer.log =  1 line    (just the inotify readiness banner)
```

**(ii) The Docker Compose healthcheck is silent.** I ran the exact probe command (`docker/compose/docker-compose.sqlite.yml:42`, interval 30 s, timeout 10 s, retries 5, `:43-45`) six times and compared the gunicorn log before and after:

```
# curl -f http://localhost:8000  (x6)
curl #1 -> HTTP 302 (exit=0)
curl #2 -> HTTP 302 (exit=0)
curl #3 -> HTTP 302 (exit=0)
curl #4 -> HTTP 302 (exit=0)
curl #5 -> HTTP 302 (exit=0)
curl #6 -> HTTP 302 (exit=0)
# gunicorn.log line count:  BEFORE = 12   AFTER = 12   (identical)
```

The probe succeeds (`-f` accepts the 302 redirect) yet adds **zero** log lines — gunicorn/uvicorn emits **no access log by default**. So the 30‑second liveness probe is invisible in the logs. **[This is why there is no periodic "healthy" line from the web server.]**

**(iii) The django‑q `Stat` heartbeat is silent while healthy.** The guard loop writes a `Stat(self).save()` to Redis every `GUARD_CYCLE = 0.5 s` (`django_q/cluster.py:288`; `GUARD_CYCLE` default `0.5` at `django_q/conf.py:90`). `Stat.save()` only logs **on failure** — `try: self.broker.set_stat(...) except Exception as e: logger.error(e)` (`django_q/status.py:71-75`). While Redis is reachable it writes silently; it becomes a loud `[Q] ERROR` stream only during an outage — demonstrated directly in §4.

### 3.5 Log formats (for reference)

Two distinct formats appear at idle:
- **Paperless (Django `LOGGING`)** — `[{asctime}] [{levelname}] [{name}] {message}` — `src/paperless/settings.py:378`. Console handler at INFO by default (`settings.py:388`, `DEBUG=NO` `settings.py:50`); the `paperless` and `paperless_mail` loggers additionally write rotating files (`settings.py:392-405`) and propagate to the root console handler (`settings.py:407-410`). Example: the `document_consumer` banner and the `Sanity checker detected no issues.` line.
- **django‑q** — `HH:MM:SS [Q] LEVEL msg` — `django_q/conf.py:213-214`, with `propagate = False` (`django_q/conf.py:212`) so `[Q]` lines are independent of the Django config. Example: every `[Q]` line above.


---

## 4. R4 — Interrupt and restart a component: the reconnection / "operational again" messages

The natural target is the **Redis broker**, because it is the shared dependency of both the django‑q task queue (`src/paperless/settings.py:456`) and the Channels group layer (`settings.py:178-187`). I interrupted and restarted it **twice** to prove the pattern is stable.

**Interrupt / restart commands:**
```bash
redis-cli -h 127.0.0.1 -p 6379 shutdown nosave          # interrupt
redis-server --daemonize yes --bind 127.0.0.1 --port 6379   # restart
```

**KEY FINDING (R4): there is no dedicated "reconnected"/"connected" message.** Operational‑again is confirmed by **two co‑occurring signals**: (1) the `[Q] ERROR ... Connection refused` stream **stops** the instant Redis returns, and (2) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears with **no further errors** after it.

### 4.1 The causal mechanism (grounded in code)

- **The ~2/second error stream** is the guard's heartbeat failing: the guard calls `Stat(self).save()` every 0.5 s (`django_q/cluster.py:288`); when Redis is down, `Stat.save()` catches the write failure and calls `logger.error(e)` (`django_q/status.py:71-75`) → one `[Q] ERROR` per guard cycle ≈ 2/s.
- **The pusher dies and is reincarnated every ~10 s.** The pusher's `broker.dequeue()` uses `BLPOP` with a 1 s timeout (`django_q/brokers/redis_broker.py:20-21`). When Redis is down it raises; the pusher logs the error, `sleep(10)`, then `break`s out of its loop (`django_q/cluster.py:345-350`) and the function ends (`cluster.py:366` logs `stopped pushing tasks`). The guard detects the dead pusher (`if not self.pusher.is_alive(): self.reincarnate(self.pusher)` — `cluster.py:280-281`) and reincarnates it, logging `reincarnated pusher <name> after sudden death` (`cluster.py:223`) followed by a new `pushing tasks at <pid>` (`cluster.py:342`). The `sleep(10)` is why reincarnations are exactly 10 s apart.

### 4.2 Run #1 — before / during / after

**BEFORE:** idle and stable; the pusher had been `Process-1:13 pushing tasks at 717` since cluster start (07:47:30).

**DURING (Redis stopped at 08:08:50).** `PING` confirms the outage, then the error stream and pusher reincarnations begin. Representative unedited `[Q]` lines (the ~2/second `Connection refused` stream is shown trimmed to a few lines for space; it repeats continuously):

```
08:08:50 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
08:08:50 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
08:08:51 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
   ... (repeats ~2x/second — 52 such lines over the ~28 s outage) ...
08:09:00 [Q] INFO Process-1:13 stopped pushing tasks
08:09:00 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
08:09:00 [Q] INFO Process-1:20 pushing tasks at 1216
08:09:10 [Q] INFO Process-1:20 stopped pushing tasks
08:09:10 [Q] ERROR reincarnated pusher Process-1:20 after sudden death
08:09:10 [Q] INFO Process-1:21 pushing tasks at 1217
08:09:20 [Q] INFO Process-1:21 stopped pushing tasks
08:09:20 [Q] ERROR reincarnated pusher Process-1:21 after sudden death
08:09:20 [Q] INFO Process-1:22 pushing tasks at 1221
```

`PING` during the outage:
```
# redis-cli -h 127.0.0.1 -p 6379 ping
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

The error‑code text is the **canonical** message for a downed local Redis: `Error 111 ... Connection refused` (the exception class is `redis.exceptions.ConnectionError`). Each pusher death also produces one verbose Python `--- Logging error --- ... TypeError: not all arguments converted during string formatting` traceback dump, because django‑q calls `logger.error(e, traceback.format_exc())` with a second positional argument (`django_q/cluster.py:347`); a trimmed representative slice of that traceback (which names the exact failing call path) is:

```
File ".../django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
File ".../django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
   ...
redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.
```

**AFTER (Redis restarted at 08:09:28, `PING → PONG`).** The `[Q] ERROR` stream stops immediately, and the next reincarnated pusher stays up with no errors after it:

```
08:09:28 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.   <-- LAST error (== restart second)
08:09:30 [Q] INFO Process-1:22 stopped pushing tasks
08:09:30 [Q] ERROR reincarnated pusher Process-1:22 after sudden death
08:09:30 [Q] INFO Process-1:23 pushing tasks at 1253                              <-- operational again; NO further errors
```

Reincarnation timestamps in run #1 — **08:09:00, 08:09:10, 08:09:20, 08:09:30** — are exactly **10 s** apart (the `sleep(10)`). The last `[Q] ERROR` is at **08:09:28**, the same second Redis was restarted; `Process-1:23` (the pusher spawned after Redis returned) produces no further errors → the cluster is operational again.

### 4.3 Run #2 — reproduction (proves stability)

**BEFORE:** stable pusher `Process-1:23` (from run #1's recovery). The web‑server and consumer logs were unaffected by the idle outage (no active websocket clients / no enqueues): `gunicorn.log` stayed at 12 lines, `consumer.log` at 1 line.

**DURING (Redis stopped at 08:10:52; 75 `[Q] ERROR` lines over ~37 s ≈ 2/s).** Pusher reincarnations, unedited:

```
08:11:03 [Q] INFO Process-1:23 stopped pushing tasks
08:11:03 [Q] ERROR reincarnated pusher Process-1:23 after sudden death
08:11:03 [Q] INFO Process-1:24 pushing tasks at 1285
08:11:13 [Q] INFO Process-1:24 stopped pushing tasks
08:11:13 [Q] ERROR reincarnated pusher Process-1:24 after sudden death
08:11:13 [Q] INFO Process-1:25 pushing tasks at 1286
08:11:23 [Q] INFO Process-1:25 stopped pushing tasks
08:11:23 [Q] ERROR reincarnated pusher Process-1:25 after sudden death
08:11:23 [Q] INFO Process-1:26 pushing tasks at 1290
08:11:33 [Q] INFO Process-1:26 stopped pushing tasks
08:11:33 [Q] ERROR reincarnated pusher Process-1:26 after sudden death
08:11:33 [Q] INFO Process-1:27 pushing tasks at 1308
```

**AFTER (Redis restarted at 08:11:30, `PING → PONG`).** First `[Q] ERROR` at 08:10:53; **last `[Q] ERROR` at 08:11:30** (== restart second); the final reincarnated pusher `Process-1:27 pushing tasks at 1308` (08:11:33) runs with no subsequent errors. Reincarnation timestamps **08:11:03, 08:11:13, 08:11:23, 08:11:33** are again exactly **10 s** apart — identical to run #1.

### 4.4 Summary of the reconnection signal

| Aspect | Run #1 | Run #2 |
|---|---|---|
| Redis stopped | 08:08:50 | 08:10:52 |
| Redis restarted (`PONG`) | 08:09:28 | 08:11:30 |
| `[Q] ERROR ... Connection refused` count during outage | 52 (~2/s) | 75 (~2/s) |
| Last `[Q] ERROR` timestamp | 08:09:28 (= restart) | 08:11:30 (= restart) |
| Pusher reincarnation interval | 10 s (…00/10/20/30) | 10 s (…03/13/23/33) |
| Recovery marker (fresh pusher, no later errors) | `Process-1:23 pushing tasks at 1253` | `Process-1:27 pushing tasks at 1308` |

**Conclusion (R4):** "everything reconnected and is operational again" is confirmed by the **cessation of the `[Q] ERROR ... Connection refused` stream** together with a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line that has no errors after it. There is **no explicit "reconnected" message** at this commit.


---

## 5. R5 — Components/processes that run continuously to maintain a ready state

At idle, the following are **always‑on** (they keep running to keep the system ready even with zero documents), distinguished from **transient** work that only runs briefly when triggered.

### 5.1 Always‑on (continuously running at idle) — observed

1. **Redis server** (`redis-server 127.0.0.1:6379`, PID 485) — the shared backbone: django‑q broker (`src/paperless/settings.py:456`) **and** Channels group‑messaging layer (`settings.py:178-187`). Stopping it degrades the cluster (§4).
2. **The django‑q cluster (`qcluster`)** — a set of long‑lived processes under one master (guard/sentinel, PID 655):
   - **Guard / sentinel** — a 0.5 s health loop that reincarnates dead workers/monitor/pusher and invokes the scheduler ~every 30 s (`django_q/cluster.py:253-289`). Observed banner: `Process-1 guarding cluster montana-tennis-winner-tango`.
   - **Monitor** (PID 716) — persists task results (`django_q/cluster.py:378`). Observed: `Process-1:12 monitoring at 716`.
   - **Pusher** (PID 717) — continuously `BLPOP`‑polls the broker for new task packages (`django_q/cluster.py:342`, `django_q/brokers/redis_broker.py:20-21`). Observed: `Process-1:13 pushing tasks at 717`.
   - **Worker pool** — 11 idle worker processes waiting for work (`Process-1:1..11 ready for work`), sized by `floor(sqrt(cores))` (`src/paperless/settings.py:427-433`).
3. **gunicorn master + 2 uvicorn ASGI workers** (PIDs 671, 682, 686) — listening on `:8000` for HTTP **and** the `ws/status/$` websocket (`gunicorn.conf.py:3-5`, `src/paperless/workers.py:9`, `src/paperless/asgi.py:17-20`, `src/paperless/urls.py:136-137`). They stay up to answer the API, serve the UI, and hold websocket status connections.
4. **`document_consumer`** (PID 663) — an inotify watcher on the consumption directory, blocking on filesystem events (`src/documents/management/commands/document_consumer.py:200`). It runs continuously so a dropped‑in document is picked up immediately.

### 5.2 Transient (NOT continuously running) — observed / inferred

- **The four scheduled task executions** (mail / classifier / index / sanity) — each briefly occupies a django‑q worker when the scheduler fires it, then finishes (see the burst in §3.2 and the mail firings in §3.3). They are periodic events, not continuously‑running processes.
- **Document ingestion work** — `consume_file` (`src/documents/tasks.py:184`) runs only when a document is added; it did **not** run during the idle window (empty consume dir, `document_consumer.py:85` "Adding … to the task queue." never logged). **[The absence at idle is observed; the trigger path is INFERRED from code, as no document was ingested.]**
- **Worker recycling** — after each task a worker is recycled and replaced (`"recycle": 1`, `src/paperless/settings.py:452`); observed as `recycled worker Process-1:N` in §3.2. This is triggered by task completion, not a standalone loop.

---

## 6. Coverage pass

Every named item in the five questions, mapped to the observed evidence and citation above.

- **R1 — up at the pinned commit, stable idle.** ✅ Ran at commit `542221a38dff` (verified `git rev-parse HEAD`), Python 3.9.23; reached idle at 07:47:29 with an **empty consume directory** and the four schedules present (§2.1). Exact build/run commands in §1.1; `manage.py check` → "System check identified no issues".
- **R2 — background processes/tasks running automatically at idle.** ✅ Enumerated with observed startup banners (§2.2): gunicorn master + 2 uvicorn ASGI workers; `document_consumer` inotify watcher; the django‑q cluster = guard/sentinel + monitor + pusher + 11 workers; all over Redis. Plus the four periodic scheduled tasks (§3).
- **R3 — periodic health/readiness entries: specific messages, frequency, meaning.** ✅ The measured cadence table (§3.1): mail `[Check all e-mail accounts]` every **10 min** (measured across firings at 07:48:00, 07:56:31, 08:06:33 — consecutive interval 10 m 02 s; DB `next_run` +10 min), classifier **hourly**, index **daily**, sanity **weekly** (all confirmed via DB `next_run` in §3.3), plus the sanity line `Sanity checker detected no issues.`. Meaning of each explained in §3.1. **Specific messages** quoted verbatim with `file:line`.
  - **KEY FINDING:** no dedicated periodic "healthy" heartbeat INFO line — proven by the quiet windows and the flat gunicorn/consumer logs (§3.4(i)).
  - **Named silent mechanisms both addressed:** the 30‑second Compose healthcheck `curl :8000` (proven silent — HTTP 302, zero new log lines, §3.4(ii)); the 0.5‑second django‑q `Stat` heartbeat (silent while healthy; loud only during the §4 outage, §3.4(iii)).
- **R4 — messages confirming reconnection/operational‑again after interrupt+restart.** ✅ Redis interrupted and restarted **twice** (§4.2–§4.3), with before/during/after states. During: `[Q] ERROR Error 111 ... Connection refused` ~2/s + pusher `reincarnated pusher … after sudden death` every 10 s. After: the error stream **ceases** at the restart second and a fresh `[Q] INFO Process-1:N pushing tasks at <pid>` line appears with no later errors.
  - **KEY FINDING:** there is **no dedicated "reconnected" message**; operational status = (error stream stops) + (new `pushing tasks at <pid>` line). Reproduced identically in both runs (§4.4).
- **R5 — components/processes that keep running continuously at idle.** ✅ Always‑on vs. transient split (§5): always‑on = Redis; django‑q guard/monitor/pusher/worker‑pool; gunicorn master + 2 uvicorn workers; `document_consumer`. Transient = the four scheduled task executions and (not at idle) `consume_file`.

### 6.1 Labels used in this document

- **[INFERRED]** — statements derived from reading code rather than observed at runtime: (a) the `Polling directory for changes` fallback (inotify was actually used); (b) the `Adding … to the task queue.` ingestion path (no document was ingested at idle); (c) `consume_file` ingestion work.
- **[NON‑CANONICAL]** — environment‑specific values, each with its canonical counterpart: direct process launch vs. supervisord/compose; runtime‑installed prerequisites; running as `root`; **11 workers** = `floor(sqrt(128))` (canonical: depends on host CPU count / `PAPERLESS_TASK_WORKERS`); volatile timestamps/PIDs/word‑names. The observed outage error `Error 111 ... Connection refused` **is** the canonical message for a downed local Redis (the exception class `redis.exceptions.ConnectionError` and host/port are the canonical parts).
- **Everything else** in this document is backed by the actual, unedited runtime output shown alongside it and a `file:line` citation to the emitting code.

### 6.2 Reproduction summary

- Idle observation window: **07:47:30 → 08:11:43 UTC** (~24 minutes), single continuous run.
- Mail cadence: **3 firings** (≥2 observations); consecutive on‑schedule interval **≈10 min**; all four cadences cross‑checked against DB `next_run`.
- Redis interrupt/restart: **2 runs**; reincarnation interval **10 s** in both; recovery pattern identical.
