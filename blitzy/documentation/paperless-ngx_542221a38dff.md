# Paperless-NGX — Idle / Stable Runtime Behavior

**Commit under investigation:** `542221a38dff06361e07976452f9aea24d210542`
**Scope:** A read-only investigation of how Paperless-NGX behaves once it is **up, idle, and stable** — before any code changes. No source file was modified; the only file written is this document.

---

## Methodology & Ground Rules

This document answers five runtime questions about an idle Paperless-NGX instance. Two principles governed the work:

1. **Code is the source of truth.** Every factual claim carries an inline citation to a specific file and line range, e.g. `[gunicorn.conf.py:L17-L18]`. Nothing here rests on assumption or general Paperless folklore.
2. **The system was actually built and run.** A live stack was brought up from the provided Docker image at the target commit, allowed to idle, and then deliberately perturbed (a reversible restart of the scheduler, the Redis broker, and the web server) so that real log output could be captured first-hand.

> **Task engine note:** Paperless-NGX uses **Django-Q** (`django-q==1.3.9` `[requirements.txt:L37]`) for asynchronous and scheduled work. There is **no Celery** anywhere in the dependency set `[requirements.txt:L1-L113]`. All scheduling described below is Django-Q.

### How evidence is labelled in this document

To keep the three kinds of evidence cleanly separated (a requirement of an audit-grade answer), every block is tagged:

- **`[file:Lx-Ly]`** — a citation to repository source/configuration at this commit. These are the authoritative facts.
- **`(captured live)`** — verbatim output from the running stack described below. Where a block was shortened for length it is tagged **`(captured live — abbreviated)`** and the omitted lines are described explicitly; no block silently elides content. Mutable values (timestamps, PIDs, Django-Q cluster names, random task ids) vary per run and are shown as captured.
- **`(probe / command)`** — a command that was *issued* (e.g. an HTTP request or a `kill`), as opposed to a line the application *emitted*. A probe is not an application log line.
- **`(Django-Q library runtime)`** — strings emitted by the Django-Q library (not vendored in this repo), corroborated against the official Django-Q 1.3.x documentation **and** confirmed verbatim against the live `qcluster` output reproduced here.

### How the live stack was assembled (exact, reproducible commands)

Because the production stack needs Redis, PostgreSQL, and OCR tooling, the canonical run environment is the provided container image. **Credentials are supplied through the shell environment and are never hard-coded into commands** — set them once before running:

```bash
# Credentials are injected via the environment, not written into commands.
# (For this stack the local-dev value happens to be the well-known compose default
#  defined in docker/compose/docker-compose.postgres.yml:L43-L45; substitute your own.)
export POSTGRES_PASSWORD="<choose-a-value>"
export PAPERLESS_DBPASS="${POSTGRES_PASSWORD}"
```

Three containers were started on a shared Docker network. The webserver image is the exact image used in this investigation:

```bash
# 1. Network + dependency services
docker network create paperless-idle-net
docker run -d --name paperless-broker --network paperless-idle-net redis:6.0
docker run -d --name paperless-db --network paperless-idle-net \
  -e POSTGRES_DB=paperless -e POSTGRES_USER=paperless -e POSTGRES_PASSWORD="${POSTGRES_PASSWORD}" \
  postgres:13

# 2. The webserver container (the exact provided image), wired to broker + db
docker run -d --name paperless-web --network paperless-idle-net --entrypoint tail \
  -e PAPERLESS_REDIS=redis://paperless-broker:6379 \
  -e PAPERLESS_DBHOST=paperless-db \
  -e PAPERLESS_DBUSER=paperless -e PAPERLESS_DBPASS="${PAPERLESS_DBPASS}" -e PAPERLESS_DBNAME=paperless \
  -e PAPERLESS_DISABLE_DBHANDLER=true -p 8000:8000 \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 -f /dev/null
```

The `db` and `broker` images (`postgres:13`, `redis:6.0`) match the committed Compose topology `[docker/compose/docker-compose.postgres.yml:L31-L38]`. Two **transient** helpers were applied inside the running container to reach a clean idle state — neither modifies any source file, and both were torn down with the container afterwards:

```bash
# (transient) OCR system libraries that documents.tasks imports at module load
docker exec paperless-web bash -lc 'apt-get update -qq && apt-get install -y --no-install-recommends libzbar0 poppler-utils'
# (transient) create the consumption/media/data dirs the startup paths_check requires (see Section 1)
docker exec paperless-web bash -lc 'mkdir -p consume media data'   # relative to the repo root
```

One-time prep and the three long-running processes are then launched from the repository's `src/` directory. The gunicorn config lives at the repository root, referenced relatively as `../gunicorn.conf.py` (in the production image Supervisor references it by its absolute install path `[docker/supervisord.conf:L11]`):

```bash
# one-time prep (run from the repo's src/ directory)
python3 manage.py migrate
python3 manage.py collectstatic --noinput
python3 manage.py document_index reindex          # build the search index, as docker-prepare.sh does [docker/docker-prepare.sh:L53-L57]

# the three long-running processes (run from src/)
gunicorn -c ../gunicorn.conf.py paperless.asgi:application   # web server (config at repo root)
python3 manage.py document_consumer                          # consumption watcher
python3 manage.py qcluster                                   # Django-Q scheduler
```

In the production image these same three programs are launched and supervised automatically (see Sections 4 and 5) `[docker/supervisord.conf:L10-L35]`.

---

## Section 1 — How to get Paperless-NGX running at the target commit

**What the deployment unit is.** Paperless-NGX ships as a Docker Compose stack. A single `webserver` container runs **Supervisor in the foreground** (`nodaemon=true` `[docker/supervisord.conf:L2]`), which manages exactly three long-running programs `[docker/supervisord.conf:L10-L35]`. Two sidecar services complete the stack: a `broker` running `redis:6.0` `[docker/compose/docker-compose.postgres.yml:L31-L32]` and a `db` running `postgres:13` `[docker/compose/docker-compose.postgres.yml:L37-L38]`.

**The startup sequence (what happens before "idle").** In the production image the container entrypoint prints a banner and then hands off to a preparation script:

- The entrypoint prints `Paperless-ngx docker container starting...` `[docker/docker-entrypoint.sh:L77]`, initializes directories/permissions, and execs the command `[docker/docker-entrypoint.sh:L84-L92]`.
- `docker-prepare.sh`'s `do_work()` runs a fixed sequence: `wait_for_postgres` (only when `PAPERLESS_DBHOST` is set `[docker/docker-prepare.sh:L67-L69]`) → `wait_for_redis` → `migrations` → `search_index` → `superuser` `[docker/docker-prepare.sh:L66-L79]`. Migrations print `Apply database migrations...` `[docker/docker-prepare.sh:L44]`; the search index is rebuilt **only** if `data/.index_version` is missing or `!= "1"`, printing `Search index out of date. Updating...` `[docker/docker-prepare.sh:L53-L54]`.

**Redis readiness, captured live.** The Redis wait helper retries up to `MAX_RETRY_COUNT = 5` `[docker/wait-for-redis.py:L16]` and prints a connect line on success `[docker/wait-for-redis.py:L21,L41]`. Running it against the live broker produced exactly `(captured live)`:

```text
Waiting for Redis: redis://paperless-broker:6379
Connected to Redis broker: redis://paperless-broker:6379
```

**Startup system checks run first — captured live.** Before migrations succeed, Django's `paths_check` system check `[src/paperless/checks.py:L51-L52]` fails fast if the consumption/media directories do not exist, emitting the template `{} is set but doesn't exist.` `[src/paperless/checks.py:L10]`. On a first run this was observed verbatim `(captured live)`, which is direct proof the check executes at startup:

```text
SystemCheckError: System check identified some issues:
ERRORS:
?: PAPERLESS_CONSUMPTION_DIR is set but doesn't exist.
?: PAPERLESS_MEDIA_ROOT is set but doesn't exist.
```

After creating those directories (the transient helper above), `manage.py migrate` applied all migrations — including the data migrations that create the scheduled tasks (see Section 2) — and `collectstatic` reported `171 static files copied` to the configured static root `(captured live)`.

**Reaching idle.** Supervisor then execs and the three programs come up `[docker/supervisord.conf:L10-L29]`. The web server answers `GET /`; there is **no dedicated `/health` endpoint** — the catch-all route serves the SPA index behind `login_required` `[src/paperless/urls.py:L132]`. A bare unauthenticated `GET /` therefore returns **`302 Found` → `/accounts/login/?next=/`**, and returns **`200 OK`** once the redirect is followed `(captured live)`. Both outcomes count as "up" for the healthcheck (Section 3.1).

---

## Section 2 — What background processes / tasks continue executing automatically at idle

At idle, two distinct kinds of background work continue with zero user activity: **three always-on supervised processes**, and **four recurring Django-Q scheduled tasks**.

### 2.1 The three continuously-running supervised processes

All three are defined in `docker/supervisord.conf` and each pipes both stdout and stderr to `/dev/stdout` and `/dev/stderr`, which is why all three streams appear together in `docker logs` `[docker/supervisord.conf:L14-L17,L23-L26,L32-L35]`.

| Program | Command | Role at idle |
|---------|---------|--------------|
| `gunicorn` | `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` `[docker/supervisord.conf:L10-L11]` | ASGI web/WebSocket server. Binds `0.0.0.0:8000` `[gunicorn.conf.py:L3]`, `workers` default `2` `[gunicorn.conf.py:L4]`, `worker_class = paperless.workers.ConfigurableWorker` `[gunicorn.conf.py:L5]` (a `UvicornWorker` subclass `[src/paperless/workers.py:L9]`), `timeout = 120` `[gunicorn.conf.py:L6]`. |
| `consumer` | `python3 manage.py document_consumer` `[docker/supervisord.conf:L19-L20]` | Watches the consumption directory. Event-driven (inotify) and **silent at idle** after one readiness line `[src/documents/management/commands/document_consumer.py:L200]`. |
| `scheduler` | `python3 manage.py qcluster` `[docker/supervisord.conf:L28-L29]` | The Django-Q task cluster — runs scheduled tasks and any queued async work. |

The Supervisor program for the scheduler is literally named **`scheduler`** `[docker/supervisord.conf:L28]` (relevant to the restart test in Section 4).

### 2.2 The recurring Django-Q scheduled tasks

Scheduled tasks are stored as **rows in the `django_q` Schedule table** — they are *data created by migrations*, not static configuration. Inspecting the live table (`Schedule.objects`) on the running system returned **four** schedules `(captured live)`:

```text
name='Train the classifier'      func=documents.tasks.train_classifier          type=H              # HOURLY
name='Optimize the index'        func=documents.tasks.index_optimize            type=D              # DAILY
name='Perform sanity check'      func=documents.tasks.sanity_check              type=W              # WEEKLY
name='Check all e-mail accounts' func=paperless_mail.tasks.process_mail_accounts type=I  minutes=10  # every 10 MINUTES
```

(`H`/`D`/`W`/`I` are the Django-Q `Schedule.HOURLY` / `DAILY` / `WEEKLY` / `MINUTES` type codes; the `minutes=10` value is the schedule's stored interval.)

| Task (func) | Schedule name | Frequency | Defined by |
|-------------|---------------|-----------|------------|
| `documents.tasks.train_classifier` | "Train the classifier" | **HOURLY** | `[src/documents/migrations/1001_auto_20201109_1636.py:L10-L14]` |
| `documents.tasks.index_optimize` | "Optimize the index" | **DAILY** | `[src/documents/migrations/1001_auto_20201109_1636.py:L15-L19]` |
| `documents.tasks.sanity_check` | "Perform sanity check" | **WEEKLY** | `[src/documents/migrations/1004_sanity_check_schedule.py:L10-L14]` |
| `paperless_mail.tasks.process_mail_accounts` | "Check all e-mail accounts" | **every 10 minutes** | `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15]` |

> **The most frequent idle task.** The hourly/daily/weekly `documents.tasks` schedules are the headline three, but the live system also runs a **fourth** schedule: `paperless_mail.tasks.process_mail_accounts` fires **every 10 minutes** (`schedule_type=Schedule.MINUTES, minutes=10` `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:L13-L14]`). At idle this is the *single most frequent* recurring background task. It was confirmed by reading the live `django_q` Schedule rows, not inferred `(captured live)`.

That the schedules are genuinely recurring was verified directly `(captured live)`: all four were due at cluster start (their `next_run` lay in the past), they fired once shortly after the cluster came up, and their `next_run` timestamps then advanced by exactly their interval — `+10 min` (mail), `+1 h` (classifier), `+1 day` (index), and `+7 days` (sanity). This `next_run` advance is the recurrence mechanism; the intervals match the migration definitions cited above.

---

## Section 3 — Periodic "healthy / ready" log entries: exact message, frequency, and meaning

There are three categories of periodic signal at idle. One is an external probe (the healthcheck); one is a genuinely periodic emitted log sequence (the Django-Q execution triple); and the third is the **deliberate quietness** of the application's own task loggers, which is itself a "healthy" signal.

### 3.1 The Docker healthcheck — an external `GET /` probe every 30 seconds (not an emitted log line)

The Compose healthcheck is defined as `[docker/compose/docker-compose.postgres.yml:L55-L59]`:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000"]
  interval: 30s
  timeout: 10s
  retries: 5
```

- **What it is (probe / command):** an external HTTP `GET /` issued by Docker via `curl -f` against `http://localhost:8000` every **30 seconds**, with a 10 s timeout and 5 retries before the container is marked unhealthy `[docker/compose/docker-compose.postgres.yml:L57-L59]`.
- **It does NOT emit an application log line — verified live.** `gunicorn.conf.py` configures **no** access log (it defines only `bind`, `workers`, `worker_class`, `timeout`, and lifecycle hooks `[gunicorn.conf.py:L1-L18]`). Empirically, after issuing a `GET /` against the running server, gunicorn's stdout/stderr stream remained at exactly its four startup lines — **no access/`GET /` line was produced** `(captured live)`. The healthcheck is therefore a *probe*, observable as a container health state, **not** a periodic log entry.
- **What "healthy" means here:** `curl -f` fails only on HTTP status ≥ 400, so the `302 Found` → `/accounts/login/` redirect that a bare `GET /` returns is treated as success `(captured live)`; there is no dedicated `/health` route `[src/paperless/urls.py:L132]`. A passing probe means gunicorn, its ASGI app, and the DB-backed index view are all responsive.

### 3.2 The Django-Q `[Q]` task-execution sequence — once per schedule firing

When a schedule becomes due, the Django-Q cluster logs a short, recurring sequence `(Django-Q library runtime)`. All four schedules were due at cluster start and each fired once; one complete firing was captured verbatim `(captured live)`:

```text
22:18:01 [Q] INFO Enqueued 1
22:18:01 [Q] INFO Process-1 created a task from schedule [Train the classifier]
22:18:01 [Q] INFO Process-1:1 processing [comet-potato-oranges-lake]
22:18:01 [Q] INFO Processed [comet-potato-oranges-lake]
```

- **Messages (in order):** `Enqueued 1` → `Process-1 created a task from schedule [<schedule name>]` → `Process-1:<n> processing [<task name>]` → `Processed [<task name>]`.
- **A precise, verified detail about `[<name>]`:** the **"created a task from schedule"** line uses the **human-readable schedule name** (live: `[Train the classifier]`, `[Optimize the index]`, `[Perform sanity check]`, `[Check all e-mail accounts]`). The **"processing"** and **"Processed"** lines use Django-Q's **auto-generated random task name** (live: `[comet-potato-oranges-lake]`, `[failed-kentucky-avocado-nine]`, `[three-nitrogen-burger-diet]`, `[oklahoma-wisconsin-september-network]`) — *not* the schedule name `(captured live)`.
- **Frequency:** one such sequence per schedule firing — so **every ~10 minutes** for the mail check, **hourly** for the classifier, **daily** for the index, and **weekly** for the sanity check (matching the schedules in §2.2).
- **Meaning:** a healthy cluster picked up a due schedule, ran it in a worker, and recorded the result. Between firings the cluster sits idle.

Immediately after each task the worker is **recycled** `(captured live)` — because `Q_CLUSTER["recycle"] = 1` `[src/paperless/settings.py:L452]`, each worker handles exactly one task and is then replaced:

```text
22:18:01 [Q] INFO recycled worker Process-1:1
22:18:01 [Q] INFO Process-1:14 ready for work at 561
```

### 3.3 The application task loggers are (intentionally) quiet at idle

The most striking idle observation is how *little* the application logs. Across a full idle window the only non-`[Q]` application log lines written were the consumer's startup lines and a single sanity-check line `(captured live)`. This is by design:

- **`index_optimize()`** commits the Whoosh index via an `AsyncWriter` and logs **nothing** `[src/documents/tasks.py:L32-L35]`.
- **`train_classifier()`** **returns early and silently** when no auto-matching `Tag`/`DocumentType`/`Correspondent` exists — the normal state of a fresh install `[src/documents/tasks.py:L48-L55]`. (Only when it actually trains does it log INFO `Saving updated classifier model to {}...` `[src/documents/tasks.py:L64-L66]`.)
- **`process_mail_accounts()`** iterates `MailAccount.objects.all()` `[src/paperless_mail/tasks.py:L11-L22]`; with zero configured accounts (confirmed live `(captured live)`) the loop body never runs, so it logs nothing and merely returns `"No new documents were added."` as the task result.
- **`sanity_check()`** is the one application heartbeat that *does* log at idle: when clean it emits INFO `Sanity checker detected no issues.` `[src/documents/sanity_checker.py:L26-L27]`. Captured verbatim (note the Paperless verbose format) `(captured live)`:

```text
[2026-06-26 22:18:01,489] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

### 3.4 Two log sinks with different verbosity — and which messages land where

Paperless wires **two** logging handlers, and understanding them is essential to reading the logs correctly `[src/paperless/settings.py:L373-L412]`:

| Sink | Handler | Level | Destination |
|------|---------|-------|-------------|
| Console | `logging.StreamHandler` | `INFO` (`DEBUG` only if `PAPERLESS_DEBUG`) `[src/paperless/settings.py:L387-L390]` | stdout → `docker logs` |
| File | `ConcurrentRotatingFileHandler` | full `DEBUG` | `data/log/paperless.log` (1 MB rotation × 20 backups) `[src/paperless/settings.py:L370-L371,L392-L397]` |

The console handler is attached to the **root** logger `[src/paperless/settings.py:L407]`, while the file handler is attached only to the `paperless` (and `paperless_mail`) loggers `[src/paperless/settings.py:L408-L410]`. Because `disable_existing_loggers` is `False` `[src/paperless/settings.py:L375]`, records propagate to root. The verbose console/file format is `"[{asctime}] [{levelname}] [{name}] {message}"` `[src/paperless/settings.py:L378]`.

The three resulting routing rules were **verified live** by counting occurrences in each sink `(captured live)`:

1. **`paperless.*` at INFO+ appears in BOTH sinks.** `Sanity checker detected no issues.` was present once in `paperless.log` **and** once on the scheduler's stdout stream (one match in each).
2. **`paperless.*` at DEBUG is file-only.** The consumer's startup self-test line (logger `paperless.management.consumer`, DEBUG `[src/documents/management/commands/document_consumer.py:L51]`) appeared in `paperless.log` but **not** on the consumer's stdout — the console handler's `INFO` level filters it out. Captured verbatim (path rendered as its settings variable to keep this document container-path-neutral) `(captured live)`:

   ```text
   [2026-06-26 22:17:32,653] [DEBUG] [paperless.management.consumer] Not consuming file <CONSUMPTION_DIR>/__paperless_write_test_475__: File has moved.
   ```

3. **Django-Q `[Q]` lines reach `docker logs` but NOT `paperless.log`.** The `django_q` logger is outside the `paperless` namespace and uses its own time-only format (`21:28:25 [Q] INFO ...`), distinct from the Paperless verbose format above; the file handler is bound only to the `paperless`/`paperless_mail` loggers `[src/paperless/settings.py:L408-L410]`, so `[Q]` records only reach the process stdout/stderr. A grep for `[Q]` in `paperless.log` returned **zero** matches `(captured live)`.

The same DEBUG-is-file-only rule explains why the classifier's `Document classification model does not exist (yet) ...` line `[src/documents/classifier.py:L32-L35]` (logger `paperless.classifier` `[src/documents/classifier.py:L21]`, DEBUG) is visible only in `paperless.log`, never in `docker logs`.

---

## Section 4 — Reconnection / recovery log entries after a brief interrupt-and-restart

To answer "if you briefly interrupt and restart part of the system, what confirms everything reconnected," four **deliberate, reversible** perturbations were performed and the real recovery output captured. The system was returned to a healthy state afterward (final `GET /` → `200 OK`, all processes alive — see §4.5) `(captured live)`.

### Exact commands used for every perturbation

These are the precise, sanitized commands issued. Process IDs were located by scanning `/proc/<pid>/cmdline` (the image ships no `ps`); each `<…-pid>` below is the master process of that program `(probe / command)`:

```bash
# (4.1) Django-Q scheduler: graceful stop, then relaunch (run from src/)
kill -TERM <qcluster-master-pid>                                   # SIGTERM = graceful cluster stop
python3 manage.py qcluster                                         # relaunch

# (4.2) Redis broker: restart the container, then re-verify connectivity
docker restart paperless-broker
python3 ../docker/wait-for-redis.py                                # re-run the repo's readiness helper

# (4.3) Web server: graceful stop, then relaunch (run from src/)
kill -TERM <gunicorn-master-pid>
gunicorn -c ../gunicorn.conf.py paperless.asgi:application

# (4.4) Supervisor-managed restart (when running under Supervisor)
supervisorctl restart scheduler                                    # program name per docker/supervisord.conf:L28
```

### 4.1 Restarting the Django-Q scheduler (`qcluster`)

Sending `SIGTERM` to the cluster master produced the **graceful stop** sequence. Captured verbatim and complete (`(captured live)`; there are eleven `stopped doing work` lines, one per worker):

```text
22:20:11 [Q] INFO Q Cluster papa-lactose-chicken-timing stopping.
22:20:11 [Q] INFO Process-1 stopping cluster processes
22:20:11 [Q] INFO Process-1:13 stopped pushing tasks
22:20:11 [Q] INFO Process-1:5 stopped doing work
22:20:11 [Q] INFO Process-1:6 stopped doing work
22:20:11 [Q] INFO Process-1:7 stopped doing work
22:20:11 [Q] INFO Process-1:8 stopped doing work
22:20:11 [Q] INFO Process-1:9 stopped doing work
22:20:11 [Q] INFO Process-1:11 stopped doing work
22:20:11 [Q] INFO Process-1:10 stopped doing work
22:20:11 [Q] INFO Process-1:14 stopped doing work
22:20:11 [Q] INFO Process-1:15 stopped doing work
22:20:11 [Q] INFO Process-1:16 stopped doing work
22:20:11 [Q] INFO Process-1:17 stopped doing work
22:20:12 [Q] INFO Process-1 waiting for the monitor.
22:20:12 [Q] INFO Process-1:12 stopped monitoring results
22:20:12 [Q] INFO Q Cluster papa-lactose-chicken-timing has stopped.
```

Relaunching produced the **start → running** reconnection sequence. The decisive "operational again" line is the final `running.` Captured verbatim and complete (eleven `ready for work` lines, one per worker) `(captured live)`:

```text
22:20:28 [Q] INFO Q Cluster vegan-three-high-lima starting.
22:20:28 [Q] INFO Process-1:1 ready for work at 915
22:20:28 [Q] INFO Process-1:2 ready for work at 916
22:20:28 [Q] INFO Process-1:3 ready for work at 917
22:20:28 [Q] INFO Process-1:4 ready for work at 918
22:20:28 [Q] INFO Process-1:5 ready for work at 919
22:20:28 [Q] INFO Process-1:6 ready for work at 920
22:20:28 [Q] INFO Process-1:7 ready for work at 921
22:20:28 [Q] INFO Process-1:8 ready for work at 922
22:20:28 [Q] INFO Process-1:9 ready for work at 923
22:20:28 [Q] INFO Process-1:10 ready for work at 924
22:20:28 [Q] INFO Process-1:11 ready for work at 925
22:20:28 [Q] INFO Process-1:12 monitoring at 926
22:20:28 [Q] INFO Process-1 guarding cluster vegan-three-high-lima
22:20:28 [Q] INFO Process-1:13 pushing tasks at 927
22:20:28 [Q] INFO Q Cluster vegan-three-high-lima running.
```

> **Observed detail:** each cluster instance receives a **new humanized name** — `papa-lactose-chicken-timing` before the restart, `vegan-three-high-lima` after `(captured live)`. The number of `ready for work` lines equals the worker count (here **11**; see Section 5 for why). No `created a task from schedule` lines appeared on relaunch (see §4.5).

### 4.2 Restarting the Redis broker (`docker restart paperless-broker`)

This is the most informative perturbation because Redis is the broker for both Django-Q and Channels. The re-run readiness helper confirmed reconnection `(captured live)`:

```text
Waiting for Redis: redis://paperless-broker:6379
Connected to Redis broker: redis://paperless-broker:6379
```

`[docker/wait-for-redis.py:L21,L41]`

The running cluster's pusher hit the dropped connection and Django-Q's sentinel **self-healed** it. The decisive recovery lines were captured verbatim (`(captured live)`; Django-Q also dumps a Python `ConnectionError` traceback, which is library noise and is omitted here):

```text
22:20:49 [Q] ERROR Error -5 connecting to paperless-broker:6379. No address associated with hostname.
22:20:59 [Q] INFO Process-1:13 stopped pushing tasks
22:20:59 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
22:20:59 [Q] INFO Process-1:14 pushing tasks at 945
```

- The transient `Error -5 ... No address associated with hostname` is the brief DNS gap while the broker container is recreated.
- `reincarnated pusher Process-1:13 after sudden death` followed by a new `pushing tasks` line is the **recovery confirmation** — Django-Q's sentinel replaced the dead pusher `(Django-Q library runtime)`. (`Q_CLUSTER["recycle"] = 1` `[src/paperless/settings.py:L452]`, `timeout = 1800` `[src/paperless/settings.py:L454]`, `retry = 1810` `[src/paperless/settings.py:L453]`.)

Full operational recovery was then **proven** by enqueuing a probe task, which the cluster picked up and completed `(captured live)`:

```text
22:21:27 [Q] INFO Enqueued 1
22:21:27 [Q] INFO Process-1:1 processing [uniform-texas-cat-foxtrot]
22:21:27 [Q] INFO Processed [uniform-texas-cat-foxtrot]
22:21:27 [Q] INFO recycled worker Process-1:1
22:21:27 [Q] INFO Process-1:15 ready for work at 985
```

### 4.3 Restarting the web server (`gunicorn`)

Gracefully stopping and relaunching the gunicorn master re-emitted the readiness line from the `when_ready` hook `[gunicorn.conf.py:L17-L18]`. Captured verbatim and complete `(captured live)`:

```text
[2026-06-26 22:21:53 +0000] [1024] [INFO] Starting gunicorn 20.1.0
[2026-06-26 22:21:53 +0000] [1024] [INFO] Listening at: http://0.0.0.0:8000 (1024)
[2026-06-26 22:21:53 +0000] [1024] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-06-26 22:21:53 +0000] [1024] [INFO] Server is ready. Spawning workers
```

The decisive "operational again" line is **`Server is ready. Spawning workers`** `[gunicorn.conf.py:L17-L18]`.

### 4.4 Supervisor-managed restart — the consumer readiness line and `entered RUNNING state`

The manual restarts in §4.1–§4.3 launched each process directly, which is why their snippets are cleanly attributable to a single component; that method does **not** exercise Supervisor. In the production image, however, all three programs run under Supervisor `[docker/supervisord.conf:L10-L35]`, which emits its own lifecycle lines and **restarts any program that exits**. To capture those exact lines, Supervisor was run over the same three programs (`gunicorn`, `consumer`, `scheduler`) and the output captured verbatim.

**Consumer readiness line.** On every (re)start the consumer prints exactly one readiness line and is then silent until a file event `[src/documents/management/commands/document_consumer.py:L200]`. Captured verbatim (directory rendered as its settings variable to keep this document container-path-neutral) `(captured live)`:

```text
[2026-06-26 22:22:38,151] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: <CONSUMPTION_DIR>
```

(The default watch mode is inotify because `CONSUMER_POLLING` defaults to `0` `[src/paperless/settings.py:L478]`; in polling mode the line would instead read `Polling directory for changes: ...` `[src/documents/management/commands/document_consumer.py:L186]`.)

**Supervisor lifecycle lines.** Supervisor's own log shows each program being spawned and then confirmed up. Captured verbatim — the `entered RUNNING state` line is Supervisor's "operational again" confirmation `(captured live)`:

```text
2026-06-26 22:22:36,996 INFO spawned: 'consumer' with pid 1124
2026-06-26 22:22:36,998 INFO spawned: 'gunicorn' with pid 1125
2026-06-26 22:22:37,000 INFO spawned: 'scheduler' with pid 1126
2026-06-26 22:22:38,151 INFO success: consumer entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-06-26 22:22:38,152 INFO success: gunicorn entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-06-26 22:22:38,152 INFO success: scheduler entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

A **Supervisor-managed restart** of one program (`supervisorctl restart scheduler` — program name per `[docker/supervisord.conf:L28]`) produced the stop → respawn → ready sequence, again ending in `entered RUNNING state` `(captured live)`:

```text
2026-06-26 22:22:54,887 INFO waiting for scheduler to stop
2026-06-26 22:22:56,050 INFO stopped: scheduler (exit status 0)
2026-06-26 22:22:56,052 INFO spawned: 'scheduler' with pid 1202
2026-06-26 22:22:57,054 INFO success: scheduler entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

- **Meaning / frequency:** these Supervisor lines appear **once per program (re)start**. In steady idle they are not periodic — they fire only when Supervisor starts the stack or when a managed program exits/crashes and Supervisor restarts it. `entered RUNNING state` is the authoritative "this program is operational again" signal at the process-supervision layer.

### 4.5 A key behavioral nuance — `catch_up = False`

`Q_CLUSTER["catch_up"]` is **`False`** `[src/paperless/settings.py:L451]`. This means schedules whose firing time elapsed **while the cluster was down are NOT re-run** on restart — Django-Q simply advances to the next interval. This was consistent with the live restart: no `created a task from schedule` lines appeared on relaunch (a grep of the relaunch log returned zero), and the schedules' `next_run` values remained in the future `(captured live)`. The practical implication: a brief scheduler outage silently skips any window it missed rather than producing a burst of catch-up work.

---

## Section 5 — Components that run continuously to maintain the "ready" state

Even with zero documents being processed, the following components stay up to keep Paperless-NGX ready:

1. **The three Supervisor-managed processes** `[docker/supervisord.conf:L10-L29]`:
   - **gunicorn** (ASGI) — serves HTTP and WebSocket on `:8000` `[gunicorn.conf.py:L3]`. Confirmed live with **2 web workers** (one master + two worker processes), matching the default `workers = 2` `[gunicorn.conf.py:L4]` `(captured live)`.
   - **document_consumer** — the inotify watcher. The default watch mode is inotify because `CONSUMER_POLLING` defaults to `0` `[src/paperless/settings.py:L478]`; it logs `Using inotify to watch directory for changes: <CONSUMPTION_DIR>` once `[src/documents/management/commands/document_consumer.py:L200]` and is then silent until a file event.
   - **qcluster** — the Django-Q cluster. Confirmed live with **11 task workers** plus a monitor, a pusher, and a guarding sentinel `(captured live)`. The worker count is `TASK_WORKERS = floor(sqrt(cpu_count))` for hosts with ≥4 cores `[src/paperless/settings.py:L427-L438]`; this host has 128 cores, hence 11. (A typical 2–4 core deployment would show 2.) This is a **separate worker pool** from gunicorn's 2 web workers.

2. **Redis (`redis:6.0`) in a dual role** — it is simultaneously:
   - the **Django-Q broker** (`Q_CLUSTER["redis"]` `[src/paperless/settings.py:L449-L457]`), and
   - the **Channels layer backend** (`RedisChannelLayer`, `capacity` 2000, `expiry` 15 `[src/paperless/settings.py:L178-L187]`).

3. **PostgreSQL (`postgres:13`)** — the primary database `[docker/compose/docker-compose.postgres.yml:L37-L38]`, read/written by gunicorn (ORM) and qcluster (schedules + task results).

4. **The WebSocket / Channels layer** — ASGI requests are routed by a `ProtocolTypeRouter` that sends HTTP to the Django ASGI app and WebSocket through `AuthMiddlewareStack(URLRouter(...))` `[src/paperless/asgi.py:L17-L22]`. The `ws/status/` route `[src/paperless/urls.py:L136-L138]` is served by `StatusConsumer`, which joins the `status_updates` group on connect and relays progress events `[src/paperless/consumers.py:L9-L33]`. It performs **no logging of its own**, so it is silent at idle — but the consumer (and its Redis-backed group) remains connected and ready.

5. **Startup system checks** run before the server is allowed up — `paths_check`, `binaries_check`, and `debug_mode_check` are registered Django checks `[src/paperless/checks.py:L51-L52,L65-L66,L85-L86]`; the first of these is what enforces that the consumption/media directories exist (proven live in Section 1).

---

## Section 6 — Rationale / "thinking" behind each answer

This section makes explicit the reasoning that connects the code to the conclusions above.

- **Why all three processes' logs appear together in `docker logs`.** Supervisor routes each program's stdout and stderr to `/dev/stdout` and `/dev/stderr` with `*_logfile_maxbytes=0` (no rotation) `[docker/supervisord.conf:L14-L35]`. The container's PID 1 is Supervisor (`nodaemon=true` `[docker/supervisord.conf:L2]`), so the three streams are merged into the container's log stream. In this investigation each process was instead launched into a separate file, which is why per-process attribution was possible.

- **Why Django-Q `[Q]` lines reach `docker logs` but not `paperless.log`.** The file handler is bound only to the `paperless`/`paperless_mail` loggers `[src/paperless/settings.py:L408-L410]`. `django_q` is outside the `paperless` namespace and uses its own time-only log format, so its records never reach the `paperless` file handler — they surface only on the process stdout/stderr (→ `docker logs` in production). This was confirmed by finding **zero** `[Q]` lines in `paperless.log` `(captured live)`.

- **Why a `paperless.*` INFO line shows up in both sinks, but DEBUG only in the file.** A `paperless.*` record hits the file handler directly (DEBUG-capable) **and** propagates to the root console handler `[src/paperless/settings.py:L407-L409]`. The console handler's level is `INFO` `[src/paperless/settings.py:L388]`, so INFO+ passes to stdout while DEBUG is filtered out — leaving DEBUG file-only. Verified live with the sanity-check INFO line (both sinks) versus the consumer DEBUG self-test line (file only) `(captured live)`.

- **Why the system is so quiet at idle.** The three "noisy-sounding" tasks are deliberately silent in their idle paths: `index_optimize` has no log calls `[src/documents/tasks.py:L32-L35]`; `train_classifier` early-returns before logging when there is nothing to match `[src/documents/tasks.py:L48-L55]`; and `process_mail_accounts` iterates an empty account set `[src/paperless_mail/tasks.py:L11-L22]`. The only application heartbeat is the weekly `sanity_check` "no issues" INFO line `[src/documents/sanity_checker.py:L26-L27]`. Therefore the *dominant* emitted periodic signal at idle is the Django-Q `[Q]` execution sequence (most often the 10-minute mail check); the 30-second healthcheck is an external probe, not an emitted log line (Section 3.1).

- **Why `[<name>]` differs between the "created from schedule" line and the "processing/Processed" lines.** Django-Q creates an ad-hoc task from the schedule and gives that task its own auto-generated humanized id; the scheduler logs the human-readable **schedule** name when *creating* the task, but the worker logs the **task** name when *processing* it. The live capture confirmed both forms in the same firing `(captured live)`.

- **Why missed schedules don't replay after a restart.** `catch_up=False` `[src/paperless/settings.py:L451]` tells Django-Q to skip intervals that elapsed during downtime rather than back-fill them. This is the correct behavior for periodic maintenance work (re-running an hour of skipped "optimize the index" calls would be pointless), and it matched the live restart, where no catch-up tasks fired `(captured live)`.

- **Why "healthy" is a probe on `/` rather than a `/health` endpoint.** There is no health route; the catch-all URL serves the SPA index behind `login_required` `[src/paperless/urls.py:L132]`, and the Compose healthcheck simply asserts `curl -f http://localhost:8000` succeeds `[docker/compose/docker-compose.postgres.yml:L55-L59]`. Because `curl -f` only fails on status ≥ 400, the `302 Found` → login redirect counts as healthy `(captured live)`; the probe asserts responsiveness of gunicorn, its ASGI app, and the DB-backed view, without emitting any application log line.

---

## Section 7 — Citations appendix

Every factual claim above maps to one of the following file locators (verified by reading each file at commit `542221a38dff`). Blocks tagged `(captured live)` were captured from the running stack described in the Methodology section; blocks tagged `(Django-Q library runtime)` are library output (see the note at the end).

| # | Claim | File : locator |
|---|-------|----------------|
| 1 | Supervisor runs in foreground | `docker/supervisord.conf:L2` |
| 2 | Three supervised programs (gunicorn / consumer / scheduler) | `docker/supervisord.conf:L10-L29` |
| 3 | All three pipe stdout/stderr to `/dev/stdout`/`/dev/stderr` | `docker/supervisord.conf:L14-L17,L23-L26,L32-L35` |
| 4 | Scheduler program is named `scheduler` | `docker/supervisord.conf:L28` |
| 5 | Production Supervisor references gunicorn config by absolute path | `docker/supervisord.conf:L11` |
| 6 | Entrypoint banner "Paperless-ngx docker container starting..." | `docker/docker-entrypoint.sh:L77` |
| 7 | Entrypoint init + exec | `docker/docker-entrypoint.sh:L84-L92` |
| 8 | `do_work()` order: postgres → redis → migrations → search_index → superuser | `docker/docker-prepare.sh:L66-L79` |
| 9 | `wait_for_postgres` gated on `PAPERLESS_DBHOST` | `docker/docker-prepare.sh:L67-L69` |
| 10 | "Apply database migrations..." | `docker/docker-prepare.sh:L44` |
| 11 | Conditional "Search index out of date. Updating..." + reindex | `docker/docker-prepare.sh:L53-L57` |
| 12 | Redis helper messages (Waiting / Connected; retries=5) | `docker/wait-for-redis.py:L16,L21,L41` |
| 13 | broker `redis:6.0`, db `postgres:13` | `docker/compose/docker-compose.postgres.yml:L31-L32,L37-L38` |
| 14 | Compose DB credential defaults (env-injected at runtime) | `docker/compose/docker-compose.postgres.yml:L43-L45` |
| 15 | Healthcheck `curl -f` GET `/` every 30 s (timeout 10 s, retries 5) | `docker/compose/docker-compose.postgres.yml:L55-L59` |
| 16 | gunicorn bind / workers / worker_class / timeout; no access-log config | `gunicorn.conf.py:L1-L6` |
| 17 | gunicorn `when_ready` → "Server is ready. Spawning workers" | `gunicorn.conf.py:L17-L18` |
| 18 | `ConfigurableWorker` is a `UvicornWorker` subclass | `src/paperless/workers.py:L9` |
| 19 | `train_classifier` HOURLY, `index_optimize` DAILY | `src/documents/migrations/1001_auto_20201109_1636.py:L10-L19` |
| 20 | `sanity_check` WEEKLY | `src/documents/migrations/1004_sanity_check_schedule.py:L10-L14` |
| 21 | `process_mail_accounts` every 10 minutes (MINUTES, minutes=10) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15` |
| 22 | `index_optimize` logs nothing | `src/documents/tasks.py:L32-L35` |
| 23 | `train_classifier` early-returns silently on fresh install | `src/documents/tasks.py:L48-L55` |
| 24 | `train_classifier` INFO "Saving updated classifier model..." (when it trains) | `src/documents/tasks.py:L64-L66` |
| 25 | `process_mail_accounts` iterates accounts; returns result string | `src/paperless_mail/tasks.py:L11-L22` |
| 26 | `sanity_check` "Sanity checker detected no issues." (INFO) | `src/documents/sanity_checker.py:L26-L27` |
| 27 | classifier "model does not exist (yet)..." (DEBUG, file-only) | `src/documents/classifier.py:L21,L32-L35` |
| 28 | Consumer inotify readiness line | `src/documents/management/commands/document_consumer.py:L200` |
| 29 | Consumer polling-mode line | `src/documents/management/commands/document_consumer.py:L186` |
| 30 | Consumer DEBUG "File has moved" self-test line | `src/documents/management/commands/document_consumer.py:L51` |
| 31 | `CONSUMER_POLLING` defaults to 0 (inotify) | `src/paperless/settings.py:L478` |
| 32 | LOGGING: verbose format, console=INFO StreamHandler, file=DEBUG ConcurrentRotatingFileHandler, root=console, paperless→file | `src/paperless/settings.py:L373-L412` |
| 33 | Console handler level INFO (DEBUG if PAPERLESS_DEBUG) | `src/paperless/settings.py:L387-L390` |
| 34 | Root=console handler; paperless/paperless_mail→file handlers | `src/paperless/settings.py:L407-L410` |
| 35 | `disable_existing_loggers=False` | `src/paperless/settings.py:L375` |
| 36 | Log rotation 1 MB × 20 backups | `src/paperless/settings.py:L370-L371` |
| 37 | `Q_CLUSTER` (catch_up=False, recycle=1, retry=1810, timeout=1800, workers, redis) | `src/paperless/settings.py:L449-L457` |
| 38 | `TASK_WORKERS = floor(sqrt(cpu_count))` for ≥4 cores | `src/paperless/settings.py:L427-L438` |
| 39 | `CHANNEL_LAYERS` RedisChannelLayer (capacity 2000, expiry 15) | `src/paperless/settings.py:L178-L187` |
| 40 | ASGI `ProtocolTypeRouter` (HTTP + WebSocket) | `src/paperless/asgi.py:L17-L22` |
| 41 | `StatusConsumer` joins `status_updates`, no logging | `src/paperless/consumers.py:L9-L33` |
| 42 | `ws/status/` route | `src/paperless/urls.py:L136-L138` |
| 43 | No `/health` route — catch-all serves SPA index behind login_required | `src/paperless/urls.py:L132` |
| 44 | Startup system checks (paths/binaries/debug) + "is set but doesn't exist" template | `src/paperless/checks.py:L10,L51-L52,L65-L66,L85-L86` |
| 45 | Backend runtime `python:3.9-slim-bullseye` | `Dockerfile:L18` |
| 46 | Pins: django 4.0.4, **django-q 1.3.9**, channels 3.0.4, gunicorn 20.1.0, redis 3.5.3, scikit-learn 1.0.2, whoosh 2.7.4 | `requirements.txt:L23,L37,L38,L42,L84,L88,L111` |

### Live-evidence index (what was captured from the running stack)

The following claims are grounded in `(captured live)` output reproduced in the body above, from the single run described in the Methodology section (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`, wall-clock window `2026-06-26 22:17–22:22 UTC`):

- Redis readiness lines and the startup `paths_check` failure (Section 1).
- `collectstatic` reporting `171 static files copied`, and `GET /` returning `302 → /accounts/login/` (`200` when followed) (Sections 1, 3.1).
- The four `django_q` Schedule rows and their `next_run` advance (Section 2.2).
- The absence of any gunicorn access-log line after a `GET /` (Section 3.1).
- The `[Q]` execution triple with schedule-name vs random-task-name, worker recycling, and the sanity-check INFO line (Sections 3.2–3.3).
- The two-sink routing counts: `[Q]` absent from `paperless.log`; sanity line in both sinks; consumer DEBUG file-only (Section 3.4).
- The qcluster stop/start sequences, the Redis self-heal (`reincarnated pusher … after sudden death`) and probe-task recovery, the gunicorn restart, and the Supervisor `entered RUNNING state` lines (Section 4).
- The live worker counts (2 gunicorn workers; 11 Django-Q workers) (Section 5).

### Note on Django-Q `[Q]` strings (library runtime, not repository code)

Django-Q (`django-q==1.3.9` `[requirements.txt:L37]`) is **not vendored** in this repository, so its `[Q]`-prefixed log strings — the startup `starting → ready for work → monitoring → guarding cluster → pushing tasks → running` sequence, the `created a task from schedule [...] → processing [...] → Processed [...]` execution sequence, the `recycled worker` / `reincarnated pusher ... after sudden death` lines, and the `stopping → has stopped` shutdown sequence — are **library runtime output**. They were corroborated against the official Django-Q 1.3.x cluster documentation **and** confirmed verbatim against the live `qcluster` output captured during this investigation (reproduced in Sections 3 and 4). The exact mutable values in those lines (cluster name, PIDs, task ids, timestamps) vary per run; the live examples shown are representative captures, not fixed strings.
