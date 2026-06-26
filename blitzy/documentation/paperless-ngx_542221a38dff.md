# Paperless-NGX — Idle / Stable Runtime Behavior

**Commit under investigation:** `542221a38dff06361e07976452f9aea24d210542`
**Scope:** A read-only investigation of how Paperless-NGX behaves once it is **up, idle, and stable** — before any code changes. No source file was modified; the only file written is this document.

---

## Methodology & Ground Rules

This document answers five runtime questions about an idle Paperless-NGX instance. Two principles governed the work:

1. **Code is the source of truth.** Every factual claim carries an inline citation to a specific file and line range, e.g. `[gunicorn.conf.py:L17-L18]`. Nothing here rests on assumption or general Paperless folklore.
2. **The system was actually built and run.** A live stack was brought up from the provided Docker image at the target commit, allowed to idle, and then deliberately perturbed (a reversible restart of the scheduler, the Redis broker, and the web server) so that real log output could be captured first-hand. Every log snippet in this document is **verbatim captured output**, not a reconstruction.

> **Task engine note:** Paperless-NGX uses **Django-Q** (`django-q==1.3.9` `[requirements.txt:L37]`) for asynchronous and scheduled work. There is **no Celery** anywhere in the dependency set. All scheduling described below is Django-Q.

**How the live stack was assembled (exact commands).** Because the production stack needs Redis, PostgreSQL, and OCR tooling, the canonical run environment is the provided container image. Three containers were started on a shared Docker network:

```bash
# 1. Network + dependency services
docker network create paperless-idle-net
docker run -d --name paperless-broker --network paperless-idle-net redis:6.0
docker run -d --name paperless-db --network paperless-idle-net \
  -e POSTGRES_DB=paperless -e POSTGRES_USER=paperless -e POSTGRES_PASSWORD=paperless \
  postgres:13

# 2. The webserver container (provided image), wired to broker + db
docker run -d --name paperless-web --network paperless-idle-net --entrypoint tail \
  -e PAPERLESS_REDIS=redis://paperless-broker:6379 \
  -e PAPERLESS_DBHOST=paperless-db \
  -e PAPERLESS_DBUSER=paperless -e PAPERLESS_DBPASS=paperless -e PAPERLESS_DBNAME=paperless \
  -e PAPERLESS_DISABLE_DBHANDLER=true -p 8000:8000 \
  <provided-image> -f /dev/null

# 3. Inside the container: prepare + launch the three long-running processes (from /app/src)
python3 manage.py migrate
python3 manage.py collectstatic --noinput
gunicorn -c /app/gunicorn.conf.py paperless.asgi:application   # web server
python3 manage.py document_consumer                            # consumption watcher
python3 manage.py qcluster                                     # Django-Q scheduler
```

The `db` and `broker` images (`postgres:13`, `redis:6.0`) match the committed Compose topology `[docker/compose/docker-compose.postgres.yml:L31-L38]`. Two small environment adjustments were required to reach a clean idle state and are noted where relevant (an OCR system-library install, and creation of the consumption/media directories that the startup system-check requires).

---

## Section 1 — How to get Paperless-NGX running at the target commit

**What the deployment unit is.** Paperless-NGX ships as a Docker Compose stack. A single `webserver` container runs **Supervisor in the foreground** (`nodaemon=true` `[docker/supervisord.conf:L2]`), which manages exactly three long-running programs `[docker/supervisord.conf:L10-L35]`. Two sidecar services complete the stack: a `broker` running `redis:6.0` `[docker/compose/docker-compose.postgres.yml:L31-L32]` and a `db` running `postgres:13` `[docker/compose/docker-compose.postgres.yml:L37-L38]`.

**The startup sequence (what happens before "idle").** In the production image the container entrypoint prints a banner and then hands off to a preparation script:

- The entrypoint prints `Paperless-ngx docker container starting...` `[docker/docker-entrypoint.sh:L77]`, initializes directories/permissions, and execs the command `[docker/docker-entrypoint.sh:L84-L92]`.
- `docker-prepare.sh`'s `do_work()` runs a fixed sequence: `wait_for_postgres` (only when `PAPERLESS_DBHOST` is set) → `wait_for_redis` → `migrations` → `search_index` → `superuser` `[docker/docker-prepare.sh:L66-L79]`. The PostgreSQL wait is gated on `PAPERLESS_DBHOST` being set `[docker/docker-prepare.sh:L67-L69]`; migrations print `Apply database migrations...` `[docker/docker-prepare.sh:L44]`; the search index is rebuilt **only** if `data/.index_version` is missing or `!= "1"`, printing `Search index out of date. Updating...` `[docker/docker-prepare.sh:L53-L54]`.

**Redis readiness, captured live.** The Redis wait helper retries up to `MAX_RETRY_COUNT = 5` `[docker/wait-for-redis.py:L16]` and prints a connect line on success `[docker/wait-for-redis.py:L21,L41]`. Running it against the live broker produced exactly:

```text
Waiting for Redis: redis://paperless-broker:6379
Connected to Redis broker: redis://paperless-broker:6379
```

**Migrations & static assets, captured live.** `manage.py migrate` applied all migrations (including the data migrations that create the scheduled tasks — see Section 2), and `collectstatic` reported `171 static files copied to '/app/static'.` Before migrations succeed, Django's `paths_check` system check `[src/paperless/checks.py:L51-L52]` fails fast if the consumption/media directories do not exist (it emits `{} is set but doesn't exist.` `[src/paperless/checks.py:L10]`) — confirming those checks run at startup.

**Reaching idle.** Supervisor then execs and the three programs come up `[docker/supervisord.conf:L10-L29]`. The web server answers `GET /` (verified live: `HTTP 200 OK`). Note there is **no dedicated `/health` endpoint** — the catch-all route serves the SPA index `[src/paperless/urls.py:L132]`, so "healthy" simply means `/` returns `200`.

---

## Section 2 — What background processes / tasks continue executing automatically at idle

At idle, two distinct kinds of background work continue with zero user activity: **three always-on supervised processes**, and **four recurring Django-Q scheduled tasks**.

### 2.1 The three continuously-running supervised processes

All three are defined in `docker/supervisord.conf` and each pipes both stdout and stderr to `/dev/stdout` and `/dev/stderr`, which is why all three streams appear together in `docker logs` `[docker/supervisord.conf:L14-L17,L23-L26,L32-L35]`.

| Program | Command | Role at idle |
|---------|---------|--------------|
| `gunicorn` | `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` `[docker/supervisord.conf:L10-L11]` | ASGI web/WebSocket server. Binds `0.0.0.0:8000` `[gunicorn.conf.py:L3]`, `workers` default `2` `[gunicorn.conf.py:L4]`, `worker_class = paperless.workers.ConfigurableWorker` `[gunicorn.conf.py:L5]` (a `UvicornWorker` subclass `[src/paperless/workers.py:L9]`), `timeout = 120` `[gunicorn.conf.py:L6]`. |
| `consumer` | `python3 manage.py document_consumer` `[docker/supervisord.conf:L19-L20]` | Watches the consumption directory. Event-driven (inotify) and **silent at idle** after one readiness line `[src/documents/management/commands/document_consumer.py:L199-L200]`. |
| `scheduler` | `python3 manage.py qcluster` `[docker/supervisord.conf:L28-L29]` | The Django-Q task cluster — runs scheduled tasks and any queued async work. |

The Supervisor program for the scheduler is literally named **`scheduler`** `[docker/supervisord.conf:L28]` (relevant to the restart test in Section 4).

### 2.2 The recurring Django-Q scheduled tasks

Scheduled tasks are stored as **rows in the `django_q` Schedule table** — they are *data created by migrations*, not static configuration. Inspecting the live table (`Schedule.objects`) on the running system returned **four** schedules:

```text
('Train the classifier',       'documents.tasks.train_classifier',          'H')        # HOURLY
('Optimize the index',         'documents.tasks.index_optimize',            'D')        # DAILY
('Perform sanity check',       'documents.tasks.sanity_check',              'W')        # WEEKLY
('Check all e-mail accounts',  'paperless_mail.tasks.process_mail_accounts','I', min=10) # every 10 MINUTES
```

(`H`/`D`/`W`/`I` are the Django-Q `Schedule.HOURLY` / `DAILY` / `WEEKLY` / `MINUTES` type codes, confirmed live.)

| Task (func) | Schedule name | Frequency | Defined by |
|-------------|---------------|-----------|------------|
| `documents.tasks.train_classifier` | "Train the classifier" | **HOURLY** | `[src/documents/migrations/1001_auto_20201109_1636.py:L10-L14]` |
| `documents.tasks.index_optimize` | "Optimize the index" | **DAILY** | `[src/documents/migrations/1001_auto_20201109_1636.py:L15-L19]` |
| `documents.tasks.sanity_check` | "Perform sanity check" | **WEEKLY** | `[src/documents/migrations/1004_sanity_check_schedule.py:L10-L14]` |
| `paperless_mail.tasks.process_mail_accounts` | "Check all e-mail accounts" | **every 10 minutes** | `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15]` |

> **Important — the most frequent idle task.** The hourly/daily/weekly `documents.tasks` schedules are the headline three, but the live system also runs a **fourth** schedule: `paperless_mail.tasks.process_mail_accounts` fires **every 10 minutes** (`schedule_type=Schedule.MINUTES, minutes=10` `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:L13-L14]`). At idle this is the *single most frequent* recurring background task. It was confirmed by reading the live `django_q` Schedule rows, not inferred.

That the schedules are genuinely recurring was verified directly: after the cluster fired them once at startup, their `next_run` timestamps had advanced to roughly +10 min (mail), +1 h (classifier), +24 h (index), and +7 days (sanity) respectively.

---

## Section 3 — Periodic "healthy / ready" log entries: exact message, frequency, and meaning

There are three categories of periodic signal at idle. Two of them (the healthcheck and the Django-Q execution sequence) are genuinely periodic; the third category is the **deliberate quietness** of the application's own task loggers, which is itself a "healthy" signal.

### 3.1 The Docker healthcheck — `GET /` every 30 seconds

The Compose healthcheck is defined as:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000"]
  interval: 30s
  timeout: 10s
  retries: 5
```

`[docker/compose/docker-compose.postgres.yml:L55-L59]`

- **Message / probe:** an HTTP `GET /` issued by `curl -f` against `http://localhost:8000`.
- **Frequency:** every **30 seconds** (`interval: 30s`), with a 10 s timeout and 5 retries before the container is marked unhealthy.
- **Meaning:** there is **no dedicated `/health` endpoint**; readiness is simply a `2xx` from the SPA index route `[src/paperless/urls.py:L132]`. A live probe returned `HTTP 200 OK`. Each probe also surfaces as a gunicorn access log entry, so at idle gunicorn emits a `GET /` access line roughly every 30 seconds.

### 3.2 The Django-Q `[Q]` task-execution sequence — once per schedule firing

When a schedule becomes due, the Django-Q cluster logs a short, recurring sequence. This is a **Django-Q library runtime** signal (see Section 7 for the labeling note). On the live system, all four schedules happened to be due at cluster start, and the captured output for one firing was:

```text
21:28:25 [Q] INFO Enqueued 1
21:28:25 [Q] INFO Process-1 created a task from schedule [Train the classifier]
21:28:25 [Q] INFO Process-1:1 processing [speaker-lima-eight-ack]
...
21:28:25 [Q] INFO Processed [speaker-lima-eight-ack]
```

- **Messages (in order):** `Enqueued 1` → `Process-1 created a task from schedule [<schedule name>]` → `Process-1:N processing [<task name>]` → `Processed [<task name>]`.
- **A precise, verified detail about `[<name>]`:** the **"created a task from schedule"** line uses the **human-readable schedule name** (live: `[Train the classifier]`, `[Optimize the index]`, `[Perform sanity check]`, `[Check all e-mail accounts]`). The **"processing"** and **"Processed"** lines use Django-Q's **auto-generated random task name** (live: `[speaker-lima-eight-ack]`, `[nitrogen-ten-november-double]`, `[earth-colorado-jig-ohio]`, `[michigan-nevada-zebra-red]`) — *not* the schedule name.
- **Frequency:** one such sequence per schedule firing — so **every ~10 minutes** for the mail check, **hourly** for the classifier, **daily** for the index, and **weekly** for the sanity check (matching the schedules in §2.2).
- **Meaning:** a healthy cluster picked up a due schedule, ran it in a worker, and recorded the result. Between firings the cluster sits idle.

Immediately after each task, the worker is **recycled** (captured live) — because `Q_CLUSTER["recycle"] = 1` `[src/paperless/settings.py:L452]`, each worker handles exactly one task and is then replaced:

```text
21:28:26 [Q] INFO recycled worker Process-1:1
21:28:26 [Q] INFO Process-1:14 ready for work at 752
```

### 3.3 The application task loggers are (intentionally) quiet at idle

The most striking idle observation is how *little* the application logs. After capturing a full idle window, the only non-`[Q]` application log lines written were the consumer's startup lines and a single sanity-check line. This is by design:

- **`index_optimize()`** commits the Whoosh index via an `AsyncWriter` and logs **nothing** `[src/documents/tasks.py:L32-L35]`.
- **`train_classifier()`** **returns early and silently** when no auto-matching `Tag`/`DocumentType`/`Correspondent` exists — the normal state of a fresh install `[src/documents/tasks.py:L48-L55]`. (Only when it actually trains does it log INFO `Saving updated classifier model to {}...` `[src/documents/tasks.py:L64-L66]`.)
- **`process_mail_accounts()`** iterates `MailAccount.objects.all()` `[src/paperless_mail/tasks.py:L11-L22]`; with zero configured accounts (confirmed live) the loop body never runs, so it logs nothing and merely returns `"No new documents were added."` as the task result.
- **`sanity_check()`** is the one application heartbeat that *does* log at idle: when clean it emits INFO `Sanity checker detected no issues.` `[src/documents/sanity_checker.py:L26-L27]`. Captured live (note the Paperless verbose format):

```text
[2026-06-26 21:28:25,873] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

### 3.4 Two log sinks with different verbosity — and which messages land where

Paperless wires **two** logging handlers, and understanding them is essential to reading the logs correctly `[src/paperless/settings.py:L373-L412]`:

| Sink | Handler | Level | Destination |
|------|---------|-------|-------------|
| Console | `logging.StreamHandler` | `INFO` (`DEBUG` only if `PAPERLESS_DEBUG`) `[src/paperless/settings.py:L387-L390]` | stdout → `docker logs` |
| File | `ConcurrentRotatingFileHandler` | full `DEBUG` | `paperless.log` (1 MB rotation × 20 backups) `[src/paperless/settings.py:L370-L371,L392-L397]` |

The console handler is attached to the **root** logger `[src/paperless/settings.py:L407]`, while the file handler is attached only to the `paperless` (and `paperless_mail`) loggers `[src/paperless/settings.py:L408-L410]`. Because `disable_existing_loggers` is `False` `[src/paperless/settings.py:L375]`, records propagate to root. The verbose console/file format is `"[{asctime}] [{levelname}] [{name}] {message}"` `[src/paperless/settings.py:L378]`.

The three resulting routing rules were **verified live** (counting occurrences in each sink):

1. **`paperless.*` at INFO+ appears in BOTH sinks.** `Sanity checker detected no issues.` was present once in `paperless.log` and once on the scheduler's stdout stream.
2. **`paperless.*` at DEBUG is file-only.** The consumer's startup self-test line `Not consuming file ... : File has moved.` (DEBUG) appeared in `paperless.log` but **not** on the consumer's stdout — the console handler's `INFO` level filters it out.
3. **Django-Q `[Q]` lines reach `docker logs` but NOT `paperless.log`.** The `django_q` logger is *not* under the `paperless` namespace, so it only has the root console handler; a grep for `[Q]` in `paperless.log` returned **zero** matches. (The `[Q]` lines also use Django-Q's own time-only format, e.g. `21:28:25 [Q] INFO ...`, distinct from the Paperless verbose format above.)

The same DEBUG-is-file-only rule explains why the classifier's `Document classification model does not exist (yet) ...` line `[src/documents/classifier.py:L32-L35]` (logger `paperless.classifier` `[src/documents/classifier.py:L21]`, DEBUG) is visible only in `paperless.log`, never in `docker logs`.

---

## Section 4 — Reconnection / recovery log entries after a brief interrupt-and-restart

To answer "if you briefly interrupt and restart part of the system, what confirms everything reconnected," three **deliberate, reversible** perturbations were performed and the real recovery output captured. The system was returned to a healthy state afterward (final `GET /` → `HTTP 200 OK`, all three processes alive).

### 4.1 Restarting the Django-Q scheduler (`qcluster`)

The running cluster's master process was found and gracefully stopped (`kill -TERM <pid>`), producing the **graceful stop** sequence:

```text
21:33:02 [Q] INFO Q Cluster december-floor-december-autumn stopping.
21:33:03 [Q] INFO Process-1 stopping cluster processes
21:33:03 [Q] INFO Process-1:13 stopped pushing tasks
21:33:03 [Q] INFO Process-1:N stopped doing work        (one line per worker)
21:33:04 [Q] INFO Process-1 waiting for the monitor.
21:33:04 [Q] INFO Process-1:12 stopped monitoring results
21:33:04 [Q] INFO Q Cluster december-floor-december-autumn has stopped.
```

Relaunching `python3 manage.py qcluster` produced the **start → running** reconnection sequence (the line that confirms the cluster has re-attached to the broker and is operational is the final `running.`):

```text
21:33:21 [Q] INFO Q Cluster nineteen-diet-music-venus starting.
21:33:21 [Q] INFO Process-1:1 ready for work at 1027
...                                                       (Process-1:1 .. Process-1:11 = 11 workers)
21:33:21 [Q] INFO Process-1:12 monitoring at 1038
21:33:21 [Q] INFO Process-1 guarding cluster nineteen-diet-music-venus
21:33:21 [Q] INFO Process-1:13 pushing tasks at 1039
21:33:21 [Q] INFO Q Cluster nineteen-diet-music-venus running.
```

> **Observed detail:** each cluster instance receives a **new humanized name** — `december-floor-december-autumn` before the restart, `nineteen-diet-music-venus` after. The number of `ready for work` lines equals the worker count (here **11**; see Section 5 for why).

### 4.2 Restarting the Redis broker (`docker restart paperless-broker`)

This is the most informative perturbation because Redis is the broker for both Django-Q and Channels. The re-run readiness helper confirmed reconnection:

```text
Waiting for Redis: redis://paperless-broker:6379
Connected to Redis broker: redis://paperless-broker:6379
```

`[docker/wait-for-redis.py:L21,L41]`

The running cluster's pusher hit the dropped connection and Django-Q's sentinel **self-healed** it. Captured verbatim:

```text
Message: ConnectionError('Connection closed by server.')
21:33:49 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
21:33:59 [Q] INFO Process-1:13 stopped pushing tasks
21:33:59 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
21:33:59 [Q] INFO Process-1:14 pushing tasks at 1074
```

- The transient `Error -5 ... No address associated with hostname` is the brief DNS gap while the broker container is recreated.
- `reincarnated pusher Process-1:13 after sudden death` followed by a new `pushing tasks` line is the **recovery confirmation** — Django-Q's sentinel replaced the dead pusher. (This is the live wording for `recycle`/reincarnation behavior; `Q_CLUSTER["recycle"] = 1` `[src/paperless/settings.py:L452]`, `timeout = 1800` `[src/paperless/settings.py:L454]`, `retry = 1810` `[src/paperless/settings.py:L453]`.)

Full operational recovery was then **proven** by enqueuing a probe task, which the cluster picked up and completed:

```text
21:34:34 [Q] INFO Enqueued 1
21:34:34 [Q] INFO Process-1:1 processing [twelve-august-victor-three]
21:34:34 [Q] INFO recycled worker Process-1:1
21:34:34 [Q] INFO Process-1:15 ready for work at 1110
21:34:34 [Q] INFO Processed [twelve-august-victor-three]
```

### 4.3 Restarting the web server (`gunicorn`)

Gracefully restarting the gunicorn master re-emitted the readiness line from the `when_ready` hook `[gunicorn.conf.py:L17-L18]`:

```text
[2026-06-26 21:35:35 +0000] [1495] [INFO] Starting gunicorn 20.1.0
[2026-06-26 21:35:35 +0000] [1495] [INFO] Listening at: http://0.0.0.0:8000 (1495)
[2026-06-26 21:35:35 +0000] [1495] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-06-26 21:35:35 +0000] [1495] [INFO] Server is ready. Spawning workers
```

The decisive "operational again" line is **`Server is ready. Spawning workers`** `[gunicorn.conf.py:L17-L18]`.

### 4.4 A key behavioral nuance — `catch_up = False`

`Q_CLUSTER["catch_up"]` is **`False`** `[src/paperless/settings.py:L451]`. This means schedules whose firing time elapsed **while the cluster was down are NOT re-run** on restart — Django-Q simply advances to the next interval. This was consistent with the live restart: no `created a task from schedule` lines appeared on relaunch (a grep of the restart log returned zero), and the schedules' `next_run` values remained in the future. The practical implication: a brief scheduler outage silently skips any window it missed rather than producing a burst of catch-up work.

> In a real single-container deployment all three processes' stdout/stderr are funneled to `/dev/stdout` and `/dev/stderr` `[docker/supervisord.conf:L14-L35]`, **and** Supervisor logs its own `... entered RUNNING state ...` line whenever it (re)starts a managed program. In this investigation each process was launched into its own log file, which is why the snippets above are cleanly attributable to a single component.

---

## Section 5 — Components that run continuously to maintain the "ready" state

Even with zero documents being processed, the following components stay up to keep Paperless-NGX ready:

1. **The three Supervisor-managed processes** `[docker/supervisord.conf:L10-L29]`:
   - **gunicorn** (ASGI) — serves HTTP and WebSocket on `:8000` `[gunicorn.conf.py:L3]`. Confirmed live with **2 web workers** (the default `workers = 2` `[gunicorn.conf.py:L4]`).
   - **document_consumer** — the inotify watcher. The default watch mode is inotify because `CONSUMER_POLLING` defaults to `0` `[src/paperless/settings.py:L478]`; it logs `Using inotify to watch directory for changes: {directory}` once `[src/documents/management/commands/document_consumer.py:L199-L200]` and is then silent until a file event.
   - **qcluster** — the Django-Q cluster. Confirmed live with **11 task workers** plus a monitor, a pusher, and a guarding sentinel. The worker count is `TASK_WORKERS = floor(sqrt(cpu_count))` for hosts with ≥4 cores `[src/paperless/settings.py:L427-L438]`; this host has 128 cores, hence 11. (A typical 2–4 core deployment would show 2.) This is a **separate worker pool** from gunicorn's 2 web workers.

2. **Redis (`redis:6.0`) in a dual role** — it is simultaneously:
   - the **Django-Q broker** (`Q_CLUSTER["redis"]` `[src/paperless/settings.py:L449-L457]`), and
   - the **Channels layer backend** (`RedisChannelLayer`, `capacity` 2000, `expiry` 15 `[src/paperless/settings.py:L178-L187]`).

3. **PostgreSQL (`postgres:13`)** — the primary database `[docker/compose/docker-compose.postgres.yml:L37-L38]`, read/written by gunicorn (ORM) and qcluster (schedules + task results).

4. **The WebSocket / Channels layer** — ASGI requests are routed by a `ProtocolTypeRouter` that sends HTTP to the Django ASGI app and WebSocket through `AuthMiddlewareStack(URLRouter(...))` `[src/paperless/asgi.py:L17-L22]`. The `ws/status/` route `[src/paperless/urls.py:L136-L138]` is served by `StatusConsumer`, which joins the `status_updates` group on connect and relays progress events `[src/paperless/consumers.py:L9-L33]`. It performs **no logging of its own**, so it is silent at idle — but the consumer (and its Redis-backed group) remains connected and ready.

5. **Startup system checks** run before the server is allowed up — `paths_check`, `binaries_check`, and `debug_mode_check` are registered Django checks `[src/paperless/checks.py:L51-L52,L65-L66,L85-L86]`; the first of these is what enforces that the consumption/media directories exist.

---

## Section 6 — Rationale / "thinking" behind each answer

This section makes explicit the reasoning that connects the code to the conclusions above.

- **Why all three processes' logs appear together in `docker logs`.** Supervisor routes each program's stdout and stderr to `/dev/stdout` and `/dev/stderr` with `*_logfile_maxbytes=0` (no rotation) `[docker/supervisord.conf:L14-L35]`. The container's PID 1 is Supervisor (`nodaemon=true` `[docker/supervisord.conf:L2]`), so the three streams are merged into the container's log stream. In this investigation each process was instead launched into a separate file, which is why per-process attribution was possible.

- **Why Django-Q `[Q]` lines reach `docker logs` but not `paperless.log`.** The file handler is bound only to the `paperless`/`paperless_mail` loggers `[src/paperless/settings.py:L408-L410]`, whereas the console handler is on the **root** logger `[src/paperless/settings.py:L407]`. `django_q` is outside the `paperless` namespace, so its records reach only the root console handler. With `disable_existing_loggers=False` `[src/paperless/settings.py:L375]`, those records still propagate to root and thus to stdout. This was confirmed by finding **zero** `[Q]` lines in `paperless.log`.

- **Why a `paperless.*` INFO line shows up in both sinks, but DEBUG only in the file.** A `paperless.*` record hits the file handler directly (DEBUG-capable) **and** propagates to the root console handler. The console handler's level is `INFO` `[src/paperless/settings.py:L388]`, so INFO+ passes to stdout while DEBUG is filtered out — leaving DEBUG file-only. Verified live with the sanity-check INFO line (both sinks) versus the consumer DEBUG self-test line (file only).

- **Why the system is so quiet at idle.** The three "noisy-sounding" tasks are deliberately silent in their idle paths: `index_optimize` has no log calls `[src/documents/tasks.py:L32-L35]`; `train_classifier` early-returns before logging when there is nothing to match `[src/documents/tasks.py:L48-L55]`; and `process_mail_accounts` iterates an empty account set `[src/paperless_mail/tasks.py:L11-L22]`. The only application heartbeat is the weekly `sanity_check` "no issues" INFO line `[src/documents/sanity_checker.py:L26-L27]`. Therefore the *dominant* periodic signal at idle is the Django-Q `[Q]` execution sequence (most often the 10-minute mail check) plus the 30-second healthcheck `GET /`.

- **Why `[<name>]` differs between the "created from schedule" line and the "processing/Processed" lines.** Django-Q creates an ad-hoc task from the schedule and gives that task its own auto-generated humanized id; the scheduler logs the human-readable **schedule** name when *creating* the task, but the worker logs the **task** name when *processing* it. The live capture confirmed both forms in the same firing.

- **Why missed schedules don't replay after a restart.** `catch_up=False` `[src/paperless/settings.py:L451]` tells Django-Q to skip intervals that elapsed during downtime rather than back-fill them. This is the correct behavior for periodic maintenance work (re-running an hour of skipped "optimize the index" calls would be pointless), and it matched the live restart, where no catch-up tasks fired.

- **Why "healthy" is a `200` on `/` rather than a `/health` probe.** There is no health route; the catch-all URL serves the SPA index `[src/paperless/urls.py:L132]`, and the Compose healthcheck simply asserts `curl -f http://localhost:8000` succeeds `[docker/compose/docker-compose.postgres.yml:L55-L59]`. A `2xx` means gunicorn, its ASGI app, and the DB-backed index view are all responsive.

---

## Section 7 — Citations appendix

Every factual claim above maps to one of the following file locators (verified by reading each file at commit `542221a38dff`). Live log strings were captured from the running stack described in the Methodology section.

| # | Claim | File : locator |
|---|-------|----------------|
| 1 | Supervisor runs in foreground | `docker/supervisord.conf:L2` |
| 2 | Three supervised programs (gunicorn / consumer / scheduler) | `docker/supervisord.conf:L10-L29` |
| 3 | All three pipe stdout/stderr to `/dev/stdout`/`/dev/stderr` | `docker/supervisord.conf:L14-L17,L23-L26,L32-L35` |
| 4 | Entrypoint banner "Paperless-ngx docker container starting..." | `docker/docker-entrypoint.sh:L77` |
| 5 | `do_work()` order: postgres → redis → migrations → search_index → superuser | `docker/docker-prepare.sh:L66-L79` |
| 6 | "Apply database migrations..." | `docker/docker-prepare.sh:L44` |
| 7 | Conditional "Search index out of date. Updating..." | `docker/docker-prepare.sh:L53-L54` |
| 8 | Redis helper messages (Waiting / Connected / Failed; retries=5) | `docker/wait-for-redis.py:L16,L21,L38,L41` |
| 9 | broker `redis:6.0`, db `postgres:13` | `docker/compose/docker-compose.postgres.yml:L31-L32,L37-L38` |
| 10 | Healthcheck `GET /` every 30 s (timeout 10 s, retries 5) | `docker/compose/docker-compose.postgres.yml:L55-L59` |
| 11 | gunicorn bind / workers / worker_class / timeout | `gunicorn.conf.py:L3-L6` |
| 12 | gunicorn `when_ready` → "Server is ready. Spawning workers" | `gunicorn.conf.py:L17-L18` |
| 13 | `ConfigurableWorker` is a `UvicornWorker` subclass | `src/paperless/workers.py:L9` |
| 14 | `train_classifier` HOURLY, `index_optimize` DAILY | `src/documents/migrations/1001_auto_20201109_1636.py:L10-L19` |
| 15 | `sanity_check` WEEKLY | `src/documents/migrations/1004_sanity_check_schedule.py:L10-L14` |
| 16 | `process_mail_accounts` every 10 minutes (MINUTES, minutes=10) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15` |
| 17 | `index_optimize` logs nothing | `src/documents/tasks.py:L32-L35` |
| 18 | `train_classifier` early-returns silently on fresh install | `src/documents/tasks.py:L48-L55` |
| 19 | `train_classifier` INFO "Saving updated classifier model..." (when it trains) | `src/documents/tasks.py:L64-L66` |
| 20 | `process_mail_accounts` iterates accounts; returns result string | `src/paperless_mail/tasks.py:L11-L22` |
| 21 | `sanity_check` "Sanity checker detected no issues." (INFO) | `src/documents/sanity_checker.py:L26-L27` |
| 22 | classifier "model does not exist (yet)..." (DEBUG, file-only) | `src/documents/classifier.py:L21,L32-L35` |
| 23 | Consumer inotify default + readiness line | `src/documents/management/commands/document_consumer.py:L199-L200` |
| 24 | `CONSUMER_POLLING` defaults to 0 (inotify) | `src/paperless/settings.py:L478` |
| 25 | LOGGING: verbose format, console=INFO StreamHandler, file=DEBUG ConcurrentRotatingFileHandler, root=console, paperless→file | `src/paperless/settings.py:L373-L412` |
| 26 | `disable_existing_loggers=False` | `src/paperless/settings.py:L375` |
| 27 | Log rotation 1 MB × 20 backups | `src/paperless/settings.py:L370-L371` |
| 28 | `Q_CLUSTER` (catch_up=False, recycle=1, retry=1810, timeout=1800, workers, redis) | `src/paperless/settings.py:L449-L457` |
| 29 | `TASK_WORKERS = floor(sqrt(cpu_count))` for ≥4 cores | `src/paperless/settings.py:L427-L438` |
| 30 | `CHANNEL_LAYERS` RedisChannelLayer (capacity 2000, expiry 15) | `src/paperless/settings.py:L178-L187` |
| 31 | ASGI `ProtocolTypeRouter` (HTTP + WebSocket) | `src/paperless/asgi.py:L17-L22` |
| 32 | `StatusConsumer` joins `status_updates`, no logging | `src/paperless/consumers.py:L9-L33` |
| 33 | `ws/status/` route | `src/paperless/urls.py:L136-L138` |
| 34 | No `/health` route — catch-all serves SPA index | `src/paperless/urls.py:L132` |
| 35 | Startup system checks (paths/binaries/debug) | `src/paperless/checks.py:L10,L51-L52,L65-L66,L85-L86` |
| 36 | Backend runtime `python:3.9-slim-bullseye` | `Dockerfile:L18` |
| 37 | Pins: django 4.0.4, **django-q 1.3.9**, channels 3.0.4, gunicorn 20.1.0, redis 3.5.3, whoosh 2.7.4, scikit-learn 1.0.2 | `requirements.txt:L23,L37,L38,L42,L84,L88,L111` |

### Note on Django-Q `[Q]` strings (library runtime, not repository code)

Django-Q (`django-q==1.3.9` `[requirements.txt:L37]`) is **not vendored** in this repository, so its `[Q]`-prefixed log strings — the startup `starting → ready for work → monitoring → guarding cluster → pushing tasks → running` sequence, the `created a task from schedule [...] → processing [...] → Processed [...]` execution sequence, the `recycled worker` / `reincarnated pusher ... after sudden death` lines, and the `stopping → has stopped` shutdown sequence — are **library runtime output**. They were corroborated against the official Django-Q 1.3.x cluster documentation and **confirmed verbatim against the live `qcluster` output** captured during this investigation (reproduced in Sections 3 and 4). The exact mutable values in those lines (cluster name, PIDs, task ids, timestamps) vary per run; the live examples shown are representative captures, not fixed strings.
