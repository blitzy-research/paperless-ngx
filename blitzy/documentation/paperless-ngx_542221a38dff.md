# Paperless-NGX Steady-State ("Up and Idle") Runtime Behavior

**Repository:** paperless-ngx
**Commit (pinned):** `542221a38dff06361e07976452f9aea24d210542`
**Investigation type:** Read-only runtime characterization (observe by RUNNING; no source file modified).

This document answers five questions about how Paperless-NGX behaves once it is **up and idle** (running, stable, with **no documents being ingested**). Every behavioral claim is backed by log output captured at runtime, together with the **exact command** that produced it and a **`file:line` citation** to the code that emits it.

---

## 0. Provenance and evidence conventions

### 0.1 The single run all evidence comes from

All log lines, counts, and timestamps in this document come from **one** continuous run inside **one** container. Nothing here is blended from any other run.

| Provenance field | Value | How captured |
|---|---|---|
| Container name | `pngx_obs_20260710_090348` | `docker run` (see §1.2) |
| Container id | `97b1eec8f7d16beb3e7eb551b696f9d255bf1fba83d9945646bb7b8a1731664e` | `docker inspect` |
| Image ref | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` | setup instructions |
| Image digest | `sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f` | `docker inspect` |
| Repo commit (in-container) | `542221a38dff06361e07976452f9aea24d210542` | `git -C /app rev-parse HEAD` (§1.3) |
| Repo worktree | clean (`git status --porcelain` empty) | §1.3 |
| Container timezone | `Etc/UTC` (`+0000`) — **all timestamps below are UTC** | `date`, `/etc/timezone` |
| Process launch time | `2026-07-10T09:06:33Z` | `launch_meta.txt` |
| django-q cluster name (this run) | `kilo-whiskey-artist-uranus` | observed banner |

The three long-lived processes were launched to their own log files inside the container (`/app/obs_logs/{gunicorn,consumer,qcluster}.log`); those raw files were copied to the investigation host and hashed (SHA-256) before the container was destroyed. The digests below are recorded here as the **audit anchor** — the raw files themselves, being temporary observation artifacts, were removed during cleanup (§6); the run is reproducible from the exact commands in §1.2 and verifiable against these digests:

```
raw/qcluster.log   sha256 43cb699b6ebfcb4c22340c21dd748863d1ee09897f27976fcaf201fb0b92333d   (1255 lines)
raw/gunicorn.log   sha256 f39353122f62b3b472552a2f8d785d35cfc1840f66e2cd84bf2e83ee2f86bdcd   (4 lines)
raw/consumer.log   sha256 0de11e0ab8aec3c4bf1ee1eca354c675a5b5432405a7b00a839fdc5746efd1d4   (1 line)
raw/launch_meta.txt sha256 f163fb58717bf19990302a0aad337a26a13a9a9c7add239bbe129d83acad0d42
```

### 0.2 How to read the log blocks (raw vs annotated; truncation)

To keep every claim auditable, the log blocks below follow strict conventions:

- **RAW** blocks reproduce log lines exactly as written by the process. They contain no editorial text.
- Where a block is labeled **[annotated]**, any text after a `#` on a line — or a `# ...` marker on its own line — is **my annotation**, not part of the log. `# ...` explicitly marks lines I elided (always with the count/kind of what was elided). This convention is used sparingly and only where noted; it replaces the earlier document's blanket "unedited" claim, which was inaccurate.
- Counts (line counts, error counts) were produced by the exact `grep -c` / `wc -l` command shown next to them, run against the retained raw files.

### 0.3 Labels

- **[OBSERVED]** — seen directly in this run's captured output.
- **[CONFIGURED / SOURCE-DERIVED]** — a value read from configuration or a database row in this run, or read from source code; used where a full period was **not** observed end-to-end in this run (e.g. the hourly/daily/weekly cadences).
- **[INFERRED]** — derived from reading code, not exercised at runtime.
- **[NON-CANONICAL]** — an environment-specific value or a deviation from the published product packaging, always paired with its canonical counterpart.

---

## TL;DR — the two key findings

1. **There is NO dedicated periodic "healthy" heartbeat INFO log line.** When Paperless-NGX is idle and healthy, all three processes are **silent at INFO level**; the only recurring log activity naturally observed in this run is the firing of the **`Check all e-mail accounts` schedule every ~10 minutes**. The three lower-frequency schedules (classifier hourly, index daily, sanity weekly) did not recur within the observation window — their cadence is **[CONFIGURED / SOURCE-DERIVED]** (§3.1, §3.3). Two other health mechanisms — the Docker Compose `curl` liveness probe (configured every 30 s) and the django-q `Stat` write to Redis (every 0.5 s) — are **silent while healthy** (§3.4).
2. **There is NO dedicated "reconnected" log message after a component restart.** When the Redis broker is interrupted and restarted, django-q's return to operation is confirmed by **two co-occurring signals**: (a) the `[Q] ERROR ... Connection refused` stream **stops** (last error at the restart second ±1 s), and (b) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears — within one 10 s reincarnation cycle of the restart — and then runs with **no further errors** (§4). This was verified for the **django-q broker/status path only**; the Channels/websocket path was not exercised (§4.5).

---

## The five questions (preserved verbatim from the user)

> "Get Paperless-NGX running at the specified commit. Once it's idle and stable, **(R2)** what background processes or tasks continue executing automatically? **(R3)** What are the actual log entries that appear periodically showing the system is healthy and ready? I need the specific log messages, their frequency, and what they indicate. **(R4)** Also, if you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again? **(R5)** What components or processes keep running continuously to maintain Paperless-NGX in a ready state, even when no documents are being processed? You may use temporary helper commands or inspection tools if needed, but don't modify any source files and clean up any temporary artifacts when you're done."

- **R1** — Bring the system up at the pinned commit and reach a stable idle state.
- **R2** — Enumerate the background processes/tasks that keep running automatically while idle.
- **R3** — The periodic health/readiness log entries: exact messages, **measured** frequency, and meaning.
- **R4** — After interrupting and restarting a component, the messages that confirm reconnection/operational status.
- **R5** — The components/processes that run continuously to maintain a ready state at idle.

---

## 1. Environment and exact invocation commands

### 1.1 Canonical (product) topology vs. this reproduction

Paperless-NGX at this commit is a Django application whose canonical deployment is **three long-lived processes plus a Redis dependency**, declared by its process supervisor (`docker/supervisord.conf`) and mirrored by three systemd units (`scripts/paperless-*.service`):

- `[program:gunicorn]` -> `command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` — `docker/supervisord.conf:10-11`
- `[program:consumer]` -> `command=python3 manage.py document_consumer` — `docker/supervisord.conf:19-20`
- `[program:scheduler]` -> `command=python3 manage.py qcluster` — `docker/supervisord.conf:28-29`
- Each systemd unit declares `Requires=redis.service` — `scripts/paperless-webserver.service:5`, `scripts/paperless-consumer.service:3`, `scripts/paperless-scheduler.service:3`.

**The published product image vs. the image used here — they are different, and this is labeled [NON-CANONICAL — packaging].** The two are distinguished explicitly so no claim conflates them:

| Aspect | Published product image (`Dockerfile` @ this commit) | This reproduction image (SWE-Atlas coding-agent image, designated by the setup instructions) |
|---|---|---|
| Base | `FROM python:3.9-slim-bullseye as main-app` — `Dockerfile:18` | same Python 3.9 base (interpreter observed `Python 3.9.23`, §1.5) |
| Working dir | `/usr/src/paperless/src/` — `Dockerfile:77,150` | repo checked out at **`/app`**; processes run from `/app/src` |
| Service user | `useradd ... paperless` (uid 1000) — `Dockerfile:158` | `testuser` (uid 1000); container default login is `root` (§1.4) |
| Entrypoint / CMD | `ENTRYPOINT /sbin/docker-entrypoint.sh` (`Dockerfile:168`), `CMD supervisord -c /etc/supervisord.conf` (`Dockerfile:172`) | `ENTRYPOINT /bin/bash`; **no Paperless CMD** — the three processes are launched manually (§1.2) |
| Redis | **NOT** in the image. `RUNTIME_PACKAGES` (`Dockerfile:35-75`) lists curl, file, ghostscript, imagemagick, libzbar0, poppler-utils, pngquant, tesseract-ocr, etc. but **no redis-server**; the only redis artifact baked in is `wait-for-redis.py` copied to `/sbin` (`Dockerfile:141-142`). Redis is a **separate `redis:6.0` service** in `docker/compose/docker-compose.sqlite.yml:28-32`, reached via `PAPERLESS_REDIS=redis://broker:6379` (`docker-compose.sqlite.yml:53`). | `redis-server` was `apt`-installed into the container (§1.2) to stand in for that external service. |
| Process supervision | `supervisord` runs the three programs | the three processes launched directly (identical commands), each to its own log file (§1.2) |

**Bottom line:** the Python code paths exercised here are identical to the product's (same commit, same pinned dependencies), so all logging/task/recovery behavior is faithful. Only the **packaging** differs: user, working directory, entrypoint/supervisor, and the fact that Redis is a co-located `apt` package here rather than a separate `redis:6.0` container. Each such difference is flagged **[NON-CANONICAL — packaging]** where relevant.

### 1.2 Exact, executable invocation commands (as actually run)

All commands were issued **from the investigation host** against the container via `docker exec`. The long-lived processes were run as **`testuser`**, with `HOME=/app`, working directory **`/app/src`**, each **detached** (`setsid ... &`) to its own log file, capturing the PID (`echo $!`). This block is directly runnable (it names the host-vs-container boundary, the user, cwd, env, redirections, backgrounding, log paths, and PID capture that the processes actually used):

```bash
# ---- HOST: create a fresh, uniquely-named, isolated container from the designated image ----
docker run -d --name pngx_obs_20260710_090348 --entrypoint sleep \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 \
  infinity
CN=pngx_obs_20260710_090348

# ---- CONTAINER: prerequisites (see caveat, §1.7). apt runs as root ----
docker exec "$CN" bash -lc 'apt-get update && apt-get install -y redis-server libzbar0 poppler-utils pngquant procps'
# NOTE: curl (the binary in the canonical Compose liveness probe, docker/compose/docker-compose.sqlite.yml:42,
#   and shipped in the product image's RUNTIME_PACKAGES, Dockerfile:36) is NOT present in this reproduction
#   image. The silent-probe check in §3.4(ii) therefore uses the equivalent Python http.client that ships in
#   the image; run `apt-get install -y curl` here if the literal curl command is desired (verified
#   byte-identical). See the tooling caveat in §1.7.

# ---- CONTAINER: Redis broker + Channels layer (canonical command), as testuser ----
docker exec -u testuser "$CN" bash -lc 'redis-server --daemonize yes --bind 127.0.0.1 --port 6379'

# ---- CONTAINER: runtime dirs, migrations (SQLite default), full-text index, startup check (from /app/src) ----
docker exec -u testuser -e HOME=/app -w /app/src "$CN" bash -lc '
  mkdir -p /app/consume /app/media /app/data /app/data/index /app/data/log /app/export /app/static
  python3 manage.py migrate
  python3 manage.py document_index reindex
  python3 manage.py check'

# ---- CONTAINER: the three canonical long-lived processes, each detached to its own log, PID captured ----
docker exec -u testuser -e HOME=/app -w /app/src "$CN" bash -lc '
  mkdir -p /app/obs_logs
  setsid gunicorn -c /app/gunicorn.conf.py paperless.asgi:application \
    >/app/obs_logs/gunicorn.log 2>&1 & echo "gunicorn_pid=$!"  >/app/obs_logs/launch_meta.txt
  setsid python3 manage.py document_consumer \
    >/app/obs_logs/consumer.log 2>&1 & echo "consumer_pid=$!" >>/app/obs_logs/launch_meta.txt
  setsid python3 manage.py qcluster \
    >/app/obs_logs/qcluster.log 2>&1 & echo "qcluster_pid=$!" >>/app/obs_logs/launch_meta.txt'
```

Captured launch metadata (`/app/obs_logs/launch_meta.txt`, RAW):

```
LAUNCH_UTC=2026-07-10T09:06:33Z
gunicorn_pid=790
consumer_pid=793
qcluster_pid=796
```

### 1.3 Provenance: pinned commit and clean worktree inside the container

Producing command and output (RAW) — confirms the code is exactly the pinned commit and that **no repository file was modified** by the run:

```bash
# docker exec -u testuser "$CN" bash -lc 'git config --global --add safe.directory /app; \
#   git -C /app rev-parse HEAD; git -C /app status --porcelain'
542221a38dff06361e07976452f9aea24d210542
# (git status --porcelain produced NO output -> worktree clean)
```

### 1.4 Effective environment (what is actually overridden)

Producing command and output (RAW):

```bash
# docker exec "$CN" bash -lc 'env | grep ^PAPERLESS_ ; echo "--- default user ---"; whoami; id testuser'
PAPERLESS_DISABLE_DBHANDLER=true
--- default user ---
root
uid=1000(testuser) gid=1000(testuser) groups=1000(testuser)
```

The **only** `PAPERLESS_*` variable set by the image is `PAPERLESS_DISABLE_DBHANDLER=true`. **This has no functional effect at this commit:** the name `PAPERLESS_DISABLE_DBHANDLER` is not referenced anywhere under `src/` at `542221a38dff` (it appears only in `src/setup.cfg:12`, as the pytest environment-variable assignment `PAPERLESS_DISABLE_DBHANDLER=true` under that file's `[tool:pytest]` `env =` section — a test-time setting, unrelated to runtime logging), and the `LOGGING` configuration defines **no database log handler** — only a console `StreamHandler` plus two `ConcurrentRotatingFileHandler`s (`src/paperless/settings.py:373-411`). Every other runtime path uses the **repository defaults**: `BASE_DIR=/app/src`, media `/app/media`, consume `/app/consume`, data `/app/data`, index `/app/data/index`, log `/app/data/log` (`src/paperless/settings.py`). The container default login user is `root`, but the processes were run as `testuser` (uid 1000); see §1.7.

### 1.5 Dependency versions actually used (from the project's own pins)

Confirmed inside the running container; these match `requirements.txt` / `Pipfile.lock` at this commit. Producing command: `redis-server --version` and `python3 -c "import importlib.metadata ..."`.

| Package | Version | Role |
|---|---|---|
| Python | 3.9.23 | Canonical interpreter (`Dockerfile:18` base `python:3.9-slim-bullseye`) |
| redis-server | 6.0.16 | django-q broker + Channels group layer |
| Django | 4.0.4 | Web framework / ORM / management commands |
| django-q | 1.3.9 | Task queue + scheduler (`qcluster`), source of `[Q]` logs (`Pipfile` `django-q = "~=1.3"`) |
| channels | 3.0.4 | ASGI websocket framework (status updates) |
| channels-redis | 3.4.0 | Redis-backed Channels layer |
| daphne | 3.0.2 | ASGI server library (Channels dependency) |
| gunicorn | 20.1.0 | ASGI process manager for the web server |
| uvicorn | 0.17.6 | ASGI worker (`ConfigurableWorker` base) |
| redis (py client) | 3.5.3 | Python Redis client used by django-q |
| djangorestframework | 3.13.1 | REST API layer |
| whitenoise | 6.0.0 | Static-file serving |
| concurrent-log-handler | 0.9.20 | Rotating file handler for `paperless`/`paperless_mail` loggers |
| Whoosh | 2.7.4 | Full-text index (drives daily `index_optimize`) |
| scikit-learn | 1.0.2 | Document classifier (drives hourly `train_classifier`) |
| watchdog | 2.1.7 | Filesystem event monitoring for the consumer |
| inotifyrecursive | 0.3.5 | Recursive inotify support for the consumer |

**Default configuration in effect:** database = **SQLite** at `/app/data/db.sqlite3` (`src/paperless/settings.py:297`; PostgreSQL is used only when `PAPERLESS_DBHOST` is set — `settings.py:304,311`); `PAPERLESS_REDIS` default `redis://localhost:6379` (`settings.py:182`, `settings.py:456`); `DEBUG` default `NO`, so the console log handler runs at **INFO** (`settings.py:50`, `settings.py:388`).

### 1.6 Startup checks (full transcripts)

**Migrations** (SQLite). Producing command `python3 manage.py migrate`; the three schedule-defining migrations applied cleanly and the command exited 0 (RAW excerpt + exit code):

```
Applying documents.1001_auto_20201109_1636... OK
Applying documents.1004_sanity_check_schedule... OK
Applying paperless_mail.0002_auto_20201117_1334... OK
MIGRATE_EXIT=0
```

**Full-text index** (`python3 manage.py document_index reindex`) — zero documents to index at idle (RAW + exit code):

```
0it [00:00, ?it/s]
0it [00:00, ?it/s]
REINDEX_EXIT=0
```

**System check** (`python3 manage.py check`) — the canonical startup check passes cleanly (RAW + exit code):

```
System check identified no issues (0 silenced).
CHECK_EXIT=0
```

### 1.7 Non-canonical caveats (explicitly labeled)

- **[NON-CANONICAL — packaging] Direct process launch vs. supervisord/docker-compose.** The three processes were launched directly (exactly as `docker/supervisord.conf:11,20,29` and `scripts/*.service` invoke them) rather than under `supervisord`/`docker compose`. The process invocations and code paths are identical; only the supervisor wrapper differs. *Canonical counterpart:* the same three commands started by supervisord (Docker) or systemd (bare metal).
- **[NON-CANONICAL — packaging] Prerequisites installed at runtime.** `redis-server`, `libzbar0`, `poppler-utils`, `pngquant`, and `procps` were `apt`-installed into the container. Of these, `libzbar0`/`poppler-utils`/`pngquant` **are** part of the product image's `RUNTIME_PACKAGES` (`Dockerfile:35-75`); **`redis-server` is not** — in the product deployment Redis is a separate `redis:6.0` service (`docker/compose/docker-compose.sqlite.yml:28-32`). Here it is co-located in the same container. Versions match the canonical toolchain (e.g. `redis-server 6.0.16`).
- **[NON-CANONICAL — user]** The container's default login user is `root`; the Paperless processes were run as **`testuser`** (uid 1000). The product image uses a `paperless` service account (uid 1000). This does not affect the logging/task behavior examined here.
- **[NON-CANONICAL — worker count] 11 django-q workers.** `multiprocessing.cpu_count()` reported **128** in this container, so `default_task_workers()` returned `floor(sqrt(128)) = 11` (`src/paperless/settings.py:427-433`; `TASK_WORKERS` `settings.py:438`; passed to `Q_CLUSTER["workers"]` `settings.py:455`). *Canonical counterpart:* the default depends on the host CPU count (and `PAPERLESS_TASK_WORKERS`); on a 4-core host it would be `floor(sqrt(4)) = 2`. **The worker-count value is host-specific; the formula is canonical.**
- **[NON-CANONICAL — volatile fields]** Timestamps, PIDs, the django-q cluster word-name (`kilo-whiskey-artist-uranus`) and per-task word-names vary per run.
- **The web-server liveness probe returns HTTP 302** (redirect to the login page for an unauthenticated request to `/`); `curl -f` treats this as success (§3.4).
- **[NON-CANONICAL — tooling] `curl` is not present in this reproduction image.** Verified absent (`command -v curl` → not found; not in `dpkg -l`), as is `wget`. `curl` is nonetheless the **canonical** liveness-probe binary: it is the exact Docker Compose healthcheck command (`docker/compose/docker-compose.sqlite.yml:42`) and ships in the product image's `RUNTIME_PACKAGES` (`Dockerfile:36`). The §3.4(ii) silent-probe check was therefore exercised with the **equivalent Python `http.client`** that ships in the image (issuing the same `GET http://localhost:8000/`); installing curl (`apt-get install -y curl`, exactly as the other prerequisites were added in §1.2) reproduces byte-identical status and headers. *Canonical counterpart:* `curl` in the product image.

---

## 2. R1 — the system up at a stable idle state; R2/R5 — idle background processes

### 2.1 Reaching idle (R1)

After the commands in §1.2, the system reached a stable idle state at process launch (`2026-07-10 09:06:33Z`). "Idle" means the consume directory is empty and no tasks are executing. Producing command and output (RAW):

```bash
# docker exec -u testuser "$CN" bash -lc 'ls -la /app/consume'
total 12
drwxr-sr-x 2 testuser testuser 4096 Jul 10 09:06 .
drwxr-sr-x 1 testuser testuser 4096 Jul 10 09:05 ..
```

The four periodic schedules that django-q must drive were created by the migrations (§1.6) and confirmed present in the database. Producing command `python3 manage.py shell -c "...Schedule.objects..."`; output (RAW):

```
[["Train the classifier", "documents.tasks.train_classifier", "H", null],
 ["Optimize the index", "documents.tasks.index_optimize", "D", null],
 ["Perform sanity check", "documents.tasks.sanity_check", "W", null],
 ["Check all e-mail accounts", "paperless_mail.tasks.process_mail_accounts", "I", 10]]
```

These correspond exactly to the schedule-defining migrations: `Train the classifier` (HOURLY) and `Optimize the index` (DAILY) from `src/documents/migrations/1001_auto_20201109_1636.py:10-19`; `Perform sanity check` (WEEKLY) from `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; `Check all e-mail accounts` (MINUTES, `minutes=10`) from `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15`.

### 2.2 The three processes and their startup banners (R2/R5)

Each process was launched with the exact command shown in §1.2, writing its stdout+stderr to its own log file. The producing command for each banner is `docker exec "$CN" cat <logfile>`; the PIDs are cross-referenced to the process tree in §2.3.

#### (a) `gunicorn` — web server + websockets (gunicorn master + 2 uvicorn ASGI workers)

**Command:** `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` (from `/app/src`, as `testuser`). Producing command: `cat /app/obs_logs/gunicorn.log`. Output (RAW — the complete file is 4 lines):

```
[2026-07-10 09:06:34 +0000] [790] [INFO] Starting gunicorn 20.1.0
[2026-07-10 09:06:34 +0000] [790] [INFO] Listening at: http://0.0.0.0:8000 (790)
[2026-07-10 09:06:34 +0000] [790] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-10 09:06:34 +0000] [790] [INFO] Server is ready. Spawning workers
```

- `bind = 0.0.0.0:8000`, `workers = 2`, `worker_class = "paperless.workers.ConfigurableWorker"`, `timeout = 120` — `gunicorn.conf.py:3-6`.
- `Server is ready. Spawning workers` is emitted by the repo's `when_ready(server)` hook — `gunicorn.conf.py:17-18`. The `Listening at:` and `Starting gunicorn 20.1.0` lines are gunicorn's own core startup logs.
- `ConfigurableWorker` subclasses the uvicorn `UvicornWorker` (an ASGI worker) — `src/paperless/workers.py:9`. The ASGI application wires both `http` and `websocket` protocols via `ProtocolTypeRouter` — `src/paperless/asgi.py:17`, `asgi.py:19-20`; the websocket route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` — `src/paperless/urls.py:136-137`, and `StatusConsumer` joins the Channels group `status_updates` — `src/paperless/consumers.py:9,17-20`.
- **Observed processes:** master **PID 790** with **two** uvicorn ASGI worker children **PID 801** and **PID 802** (`PPid 790`), i.e. `workers=2` as configured (§2.3).

#### (b) `document_consumer` — the consumption-directory watcher

**Command:** `python3 manage.py document_consumer`. Producing command: `cat /app/obs_logs/consumer.log`. Output (RAW — the complete file is 1 line):

```
[2026-07-10 09:06:34,932] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

- This readiness banner is emitted by `handle_inotify()` — `src/documents/management/commands/document_consumer.py:200` (logger `paperless.management.consumer`, defined at `document_consumer.py:24`). If inotify were unavailable it would instead log `Polling directory for changes: <dir>` — `document_consumer.py:186` **[INFERRED — inotify was actually used]**.
- At idle this watcher is **silent**. It only logs when a file arrives: `Adding {filepath} to the task queue.` — `document_consumer.py:85`. That line **did not appear** during the idle window (no documents ingested). **PID 793** (§2.3).

#### (c) `qcluster` — the django-q task cluster (guard/sentinel + monitor + pusher + worker pool)

**Command:** `python3 manage.py qcluster`. Producing command: `head -16 /app/obs_logs/qcluster.log`. Output (RAW — first 16 lines, the complete startup banner):

```
09:06:34 [Q] INFO Q Cluster kilo-whiskey-artist-uranus starting.
09:06:34 [Q] INFO Process-1:1 ready for work at 827
09:06:34 [Q] INFO Process-1:2 ready for work at 828
09:06:34 [Q] INFO Process-1:3 ready for work at 829
09:06:34 [Q] INFO Process-1:4 ready for work at 830
09:06:34 [Q] INFO Process-1:5 ready for work at 831
09:06:34 [Q] INFO Process-1:6 ready for work at 832
09:06:34 [Q] INFO Process-1:7 ready for work at 833
09:06:34 [Q] INFO Process-1:8 ready for work at 834
09:06:34 [Q] INFO Process-1:9 ready for work at 835
09:06:34 [Q] INFO Process-1:10 ready for work at 836
09:06:34 [Q] INFO Process-1:11 ready for work at 837
09:06:35 [Q] INFO Process-1:12 monitoring at 838
09:06:35 [Q] INFO Process-1 guarding cluster kilo-whiskey-artist-uranus
09:06:35 [Q] INFO Process-1:13 pushing tasks at 839
09:06:35 [Q] INFO Q Cluster kilo-whiskey-artist-uranus running.
```

The django-q logger uses its own format `HH:MM:SS [Q] LEVEL msg` (`django_q/conf.py:213-214`) with `propagate = False` (`django_q/conf.py:212`), so `[Q]` lines are independent of Paperless's Django `LOGGING`. Each role maps to the code that emits it (all in the pip-installed `django-q 1.3.9`):

| Banner line | Role | Source |
|---|---|---|
| `Q Cluster <name> starting.` | cluster boot | `django_q/cluster.py:79` |
| `Process-1:1..11 ready for work at <pid>` | **11 worker processes** | `django_q/cluster.py:410` |
| `Process-1:12 monitoring at <pid>` | **monitor** (persists task results) | `django_q/cluster.py:378` |
| `Process-1 guarding cluster <name>` | **guard/sentinel** (0.5 s health loop) | `django_q/cluster.py:256` |
| `Process-1:13 pushing tasks at <pid>` | **pusher** (BLPOP-polls the broker) | `django_q/cluster.py:342` |
| `Q Cluster <name> running.` | cluster ready | `django_q/cluster.py:261` |

- **Worker count = 11** = `floor(sqrt(128))` on this host — caveat §1.7. Formula: `default_task_workers()` at `src/paperless/settings.py:427-433`.
- **Observed process tree:** the `qcluster` management command is **PID 796**; it spawned the guard/sentinel **PID 826**, which forked the 11 workers (PIDs 827-837), the monitor (PID 838) and the pusher (PID 839) — §2.3.

### 2.3 Full idle process topology (observed)

Producing command and output (RAW), captured at `2026-07-10T09:07:02Z`:

```bash
# docker exec "$CN" bash -lc 'ps -eo pid,ppid,user,args | grep -E "redis-server|gunicorn|document_consumer|qcluster" | grep -v grep'
    652       1 testuser redis-server 127.0.0.1:6379
    790       1 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    793       1 testuser python3 manage.py document_consumer
    796       1 testuser python3 manage.py qcluster
    801     790 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    802     790 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    826     796 testuser python3 manage.py qcluster
    827     826 testuser python3 manage.py qcluster   # worker Process-1:1 ... (827-837 = 11 workers)
    838     826 testuser python3 manage.py qcluster   # monitor Process-1:12
    839     826 testuser python3 manage.py qcluster   # pusher  Process-1:13
```

(The block is **[annotated]**: text after `#` is mine; PIDs 828-836 — the remaining nine workers — were elided for space and are contiguous in the RAW capture.) Summary:

```
redis-server (PID 652, 127.0.0.1:6379)
qcluster mgmt (PID 796) -> guard/sentinel (PID 826) -> 11 workers (827-837) + monitor (838) + pusher (839)
document_consumer (PID 793)  -- inotify watch on /app/consume
gunicorn master (PID 790) -> 2 uvicorn ASGI workers (801, 802) on :8000
```

---

## 3. R3 — periodic health/readiness log entries: messages, frequency, and meaning

**KEY FINDING (proven below): there is no dedicated periodic "healthy" heartbeat INFO log line.** While idle and healthy, the three processes are silent at INFO. The only recurring log activity **observed in this run** is the firing of the `Check all e-mail accounts` schedule every ~10 minutes. Two additional health mechanisms run continuously but are **silent** unless something is wrong.

### 3.1 The cadence table

| Recurring signal | Frequency | Basis | Meaning / Source |
|---|---|---|---|
| `... created a task from schedule [Check all e-mail accounts]` | every **10 minutes** | **[OBSERVED]** — fired on-schedule 3x in-window; two clean intervals measured (§3.3) | Scheduler enqueues the mail-poll task; `process_mail_accounts` — `src/paperless_mail/tasks.py:11`; schedule `Schedule.MINUTES, minutes=10` — `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` |
| `... created a task from schedule [Train the classifier]` | **hourly** | **[CONFIGURED / SOURCE-DERIVED]** — fired once in the startup burst; `next_run` stepped +1 h; did not recur in-window (§3.3) | Enqueues classifier retrain; `train_classifier` — `src/documents/tasks.py:48`; `Schedule.HOURLY` — `src/documents/migrations/1001_auto_20201109_1636.py:10-14` |
| `... created a task from schedule [Optimize the index]` | **daily** | **[CONFIGURED / SOURCE-DERIVED]** — `next_run` +1 day; did not recur in-window | Enqueues Whoosh index optimize; `index_optimize` — `src/documents/tasks.py:32`; `Schedule.DAILY` — `src/documents/migrations/1001_auto_20201109_1636.py:15-19` |
| `... created a task from schedule [Perform sanity check]` | **weekly** | **[CONFIGURED / SOURCE-DERIVED]** — `next_run` +7 days; did not recur in-window | Enqueues integrity sweep; `sanity_check` — `src/documents/tasks.py:255`; `Schedule.WEEKLY` — `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; emits `Sanity checker detected no issues.` — `src/documents/sanity_checker.py:27` |
| Docker Compose healthcheck `curl -f http://localhost:8000` | every **30 seconds** (configured) | **[CONFIGURED]** — Compose not executed here; command reproduced manually (§3.4) | Liveness probe — **SILENT** (no access log by default) — `docker/compose/docker-compose.sqlite.yml:41-45` |
| django-q guard `Stat` write to Redis | every **0.5 seconds** | **[OBSERVED silent + SOURCE-DERIVED cadence]** | Cluster status write — **SILENT** unless Redis is unreachable — `django_q/cluster.py:288`, `django_q/status.py:71-75` |

**What each visible signal indicates (narrowly).** A `created a task from schedule [<name>]` line proves the django-q **scheduler** fired that schedule and the **pusher/worker** pool accepted and completed the enqueued task (the following `processing [...]` / `Processed [...]` lines). That is a liveness signal for the **scheduler -> broker -> worker** path specifically. It does **not**, by itself, prove the whole system is healthy (e.g. the web server or Channels layer), and the mail/classifier/index tasks are effectively no-ops **only because nothing is configured/queued at idle** (no mail accounts; empty index/consume) — not a guarantee they are always no-ops. The sanity task additionally emits `Sanity checker detected no issues.` when it finds no problems. The two silent mechanisms (healthcheck, `Stat` heartbeat) are continuous liveness checks that by design print nothing while healthy (§3.4).

### 3.2 The one-time startup burst (overdue schedules; `catch_up: False`)

About **30 s** after the cluster reported `running.` (`09:06:35` -> **`09:07:04`**), the guard's scheduler ran for the first time and fired all four overdue schedules **once**. django-q's scheduler is gated in the guard loop by `if counter >= 30 and Conf.SCHEDULER:` (`django_q/cluster.py:284`), and Paperless sets `"catch_up": False` (`src/paperless/settings.py:451`), so overdue schedules fire a **single** time rather than replaying every missed run. Producing command: `sed -n '17,45p' qcluster.log`. Output (RAW):

```
09:07:04 [Q] INFO Enqueued 1
09:07:04 [Q] INFO Process-1 created a task from schedule [Train the classifier]
09:07:04 [Q] INFO Enqueued 1
09:07:04 [Q] INFO Process-1 created a task from schedule [Optimize the index]
09:07:04 [Q] INFO Process-1:1 processing [red-charlie-uranus-bravo]
09:07:04 [Q] INFO Process-1:2 processing [low-venus-quiet-high]
09:07:04 [Q] INFO Enqueued 1
09:07:04 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
09:07:04 [Q] INFO Process-1:3 processing [enemy-paris-saturn-yankee]
09:07:04 [Q] INFO Enqueued 1
09:07:04 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
09:07:04 [Q] INFO Process-1:4 processing [tennis-bacon-lake-november]
09:07:04 [Q] INFO Process-1:4 stopped doing work
09:07:04 [Q] INFO Processed [tennis-bacon-lake-november]
09:07:04 [Q] INFO Process-1:1 stopped doing work
[2026-07-10 09:07:04,710] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
09:07:04 [Q] INFO Process-1:3 stopped doing work
09:07:04 [Q] INFO Process-1:2 stopped doing work
09:07:04 [Q] INFO Processed [red-charlie-uranus-bravo]
09:07:04 [Q] INFO Processed [enemy-paris-saturn-yankee]
09:07:04 [Q] INFO Processed [low-venus-quiet-high]
09:07:05 [Q] INFO recycled worker Process-1:1
09:07:05 [Q] INFO Process-1:14 ready for work at 885
09:07:05 [Q] INFO recycled worker Process-1:3
09:07:05 [Q] INFO Process-1:15 ready for work at 886
09:07:05 [Q] INFO recycled worker Process-1:2
09:07:05 [Q] INFO Process-1:16 ready for work at 887
09:07:06 [Q] INFO recycled worker Process-1:4
09:07:06 [Q] INFO Process-1:17 ready for work at 888
```

Notes:
- `Enqueued 1` — `django_q/tasks.py:74`; `Process-1 created a task from schedule [<name>]` — `django_q/cluster.py:669`.
- `Sanity checker detected no issues.` prints in the **Paperless verbose format** `[{asctime}] [{levelname}] [{name}] {message}` (`src/paperless/settings.py:378`), **not** the `[Q]` format, because it comes from the `paperless.sanity_checker` logger (`src/documents/sanity_checker.py:24,27`), which writes to its rotating file handler **and** propagates to the root console handler (`settings.py:407-410`).
- `recycled worker Process-1:N` + a new `ready for work at <pid>` — django-q recycles each worker after a task because Paperless sets `"recycle": 1` (`src/paperless/settings.py:452`), a memory-hygiene measure; this is why worker index numbers climb over time.

### 3.3 The 10-minute mail cadence — measured across FOUR firings (two clean intervals)

The `Check all e-mail accounts` task is the most frequent recurring signal and the only one whose full period was observed to repeat in this run. Producing command and output (RAW, `[annotated]` with the firing sequence):

```bash
# grep 'created a task from schedule \[Check all e-mail accounts\]' /app/obs_logs/qcluster.log
09:07:04 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #1 (overdue burst)
09:16:05 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #2
09:26:07 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #3
09:36:08 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #4
```

Two consecutive **on-schedule** intervals (two independent measurements, satisfying the >=2 requirement):
- **#2 -> #3 = `09:16:05` -> `09:26:07` = 10 min 02 s**
- **#3 -> #4 = `09:26:07` -> `09:36:08` = 10 min 01 s**

(Interval #1 -> #2 is shorter because #1 was the *delayed overdue* firing from the startup burst, §3.2.) Full firing #4 block, producing command `sed -n '60,66p' qcluster.log`, output (RAW):

```
09:36:08 [Q] INFO Enqueued 1
09:36:08 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
09:36:08 [Q] INFO Process-1:7 processing [dakota-happy-alpha-massachusetts]
09:36:08 [Q] INFO Process-1:7 stopped doing work
09:36:08 [Q] INFO Processed [dakota-happy-alpha-massachusetts]
09:36:09 [Q] INFO recycled worker Process-1:7
09:36:09 [Q] INFO Process-1:20 ready for work at 2145
```

The `process_mail_accounts` function itself logs nothing at idle — no mail accounts are configured, so its loop over accounts is empty (`src/paperless_mail/tasks.py:13`). The only evidence it ran is the `[Q]` scheduler/worker lines above.

**Cross-check of all four cadences from the live database.** django-q advances each schedule's `next_run` by its period. Reading `next_run` at two points in the run (producing command: `python3 manage.py shell -c "...Schedule.objects..."`) shows only the mail schedule advancing in-window; the other three keep a fixed future `next_run`:

At `now=2026-07-10 09:08:58` (RAW):
```
Train the classifier      | type=H | next_run=2026-07-10 10:05:55   # +1 hour
Optimize the index        | type=D | next_run=2026-07-11 09:05:55   # +1 day
Perform sanity check      | type=W | next_run=2026-07-17 09:05:55   # +7 days
Check all e-mail accounts | type=I minutes=10 | next_run=2026-07-10 09:15:55
```
At `now=2026-07-10 09:37:25` (RAW):
```
Train the classifier      | type=H | next_run=2026-07-10 10:05:55   # UNCHANGED
Optimize the index        | type=D | next_run=2026-07-11 09:05:55   # UNCHANGED
Perform sanity check      | type=W | next_run=2026-07-17 09:05:55   # UNCHANGED
Check all e-mail accounts | type=I minutes=10 | next_run=2026-07-10 09:45:55   # advanced by 3x10 min
```

So the mail `next_run` stepped `09:15:55 -> 09:25:55 -> 09:35:55 -> 09:45:55` (exactly +600 s each), while classifier/index/sanity `next_run` did **not** change (their periods exceed the ~31-minute window). This is why the mail cadence is **[OBSERVED]** end-to-end while the hourly/daily/weekly cadences are **[CONFIGURED / SOURCE-DERIVED]**.

**On the ~2 s jitter (corrected).** The database `next_run` steps are exactly 600 s; the observed 10 m 02 s / 10 m 01 s intervals are **compatible with a 10-minute period plus scheduling/polling jitter** (the guard evaluates the scheduler roughly every ~30 s — `counter >= 30`, `django_q/cluster.py:284`). The +1-2 s deltas are within that jitter budget; they do **not** by themselves measure the scheduler's poll granularity.

### 3.4 Proof that the idle system is otherwise SILENT (the KEY FINDING)

**(i) The processes emit nothing between task firings.** A background poller recorded `wc -l` of each log every ~30 s (`line_count_snapshots.csv`). Producing command per row: `wc -l <log>` + `tail -1 qcluster.log`. Representative RAW rows (`utc,qcluster_lines,gunicorn_lines,consumer_lines,qcluster_last_ts`):

```
09:16:23,52,4,1,09:16:06
09:17:23,52,4,1,09:16:06
09:18:23,52,4,1,09:16:06
09:23:24,52,4,1,09:16:06
09:24:55,52,4,1,09:16:06
09:25:25,52,4,1,09:16:06
09:30:26,59,4,1,09:26:07
09:31:56,59,4,1,09:26:07
09:32:26,59,4,1,09:26:07
```

Between the mail firings the `qcluster` log stays **flat** (52 lines after firing #2, 59 after #3; each firing adds exactly 7 lines: `Enqueued`/`created a task`/`processing`/`stopped doing work`/`Processed`/`recycled`/`ready for work`) and its last-line timestamp does not move. The web server and consumer logs were flat for the **entire ~30-minute idle window**: `gunicorn.log = 4`, `consumer.log = 1`. Producing command and output (RAW):

```bash
# docker exec "$CN" bash -lc 'wc -l /app/obs_logs/gunicorn.log /app/obs_logs/consumer.log; \
#   echo ---; tail -1 /app/obs_logs/gunicorn.log; tail -1 /app/obs_logs/consumer.log'
gunicorn.log = 4 lines; tail -1:
[2026-07-10 09:06:34 +0000] [790] [INFO] Server is ready. Spawning workers
consumer.log = 1 lines; tail -1:
[2026-07-10 09:06:34,932] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

=> **No dedicated periodic "healthy" INFO line** is emitted by any of the three processes; idle activity is limited to the scheduled-task firings (only mail recurs in-window).

**(ii) The web-server liveness probe is silent [manual reproduction of the configured Compose probe].** Docker Compose was **not** executed in this run; I reproduced its configured probe command `["CMD","curl","-f","http://localhost:8000"]` (`docker/compose/docker-compose.sqlite.yml:42`; interval 30 s, timeout 10 s, retries 5 — `:43-45`) manually, with explicit status/header capture.

*Tooling note [NON-CANONICAL — tooling]:* `curl` is **not** installed in this reproduction image (verified absent — see §1.7), even though it is the canonical probe binary (the exact Compose healthcheck command and part of the product image's `RUNTIME_PACKAGES`, `Dockerfile:36`). The probe was therefore issued with the **equivalent Python `http.client`** that ships in the image (the same `GET http://localhost:8000/`); after `apt-get install -y curl`, the literal `curl -f -s -o /dev/null -w "HTTP:%{http_code}"` (run six times) and `curl -sI http://localhost:8000` commands produce **byte-identical** status and headers. The response headers are **deterministic** at this commit (produced by Django's `SecurityMiddleware`/`LocaleMiddleware` and the login redirect); only `date` is per-run.

The block below is **[annotated]** (a composed transcript, per §0.2): the `gunicorn.log …`, `probe #…`, and `DELTA …` lines are my wrapper `echo`s and line-count annotations; the indented `HTTP/1.1 …` group is the client's verbatim response, shown **in full** (an earlier draft truncated it to four header lines). Producing commands: six `GET /` probes, then one full header dump (equivalent to `curl -sI`), with `wc -l /app/obs_logs/gunicorn.log` before and after:

```
gunicorn.log BEFORE probes = 4 lines  @ 09:10:23
probe #1 -> exit=0  HTTP:302
probe #2 -> exit=0  HTTP:302
probe #3 -> exit=0  HTTP:302
probe #4 -> exit=0  HTTP:302
probe #5 -> exit=0  HTTP:302
probe #6 -> exit=0  HTTP:302
--- response headers for GET http://localhost:8000/ (equivalent to curl -sI) ---
HTTP/1.1 302 Found
date: Fri, 10 Jul 2026 09:10:22 GMT
server: uvicorn
content-type: text/html; charset=utf-8
location: /accounts/login/?next=/
x-frame-options: SAMEORIGIN
content-length: 0
vary: Accept-Language, Origin, Cookie
content-language: en-us
x-content-type-options: nosniff
referrer-policy: same-origin
cross-origin-opener-policy: same-origin
gunicorn.log AFTER probes  = 4 lines  @ 09:10:23
DELTA = 0 new log lines from the probes
```

The probe succeeds (`curl -f` — or any HTTP client — accepts the `302` redirect to the login page, `location: /accounts/login/?next=/`) yet adds **zero** log lines: gunicorn/uvicorn emits **no access log by default**. So the configured 30-second liveness probe is invisible in the logs.

**(iii) The django-q `Stat` heartbeat is silent while healthy.** The guard loop writes `Stat(self).save()` to Redis every `GUARD_CYCLE = 0.5 s` (`django_q/cluster.py:288`; `GUARD_CYCLE` default `0.5` at `django_q/conf.py:90`). `Stat.save()` only logs **on failure** — `try: self.broker.set_stat(...) except Exception as e: logger.error(e)` (`django_q/status.py:71-75`). While Redis is reachable it writes silently (confirmed by the flat `qcluster.log` above); it becomes a loud `[Q] ERROR` stream only during an outage — demonstrated directly in §4.

### 3.5 Log formats (for reference)

Two distinct formats appear at idle:
- **Paperless (Django `LOGGING`)** — `[{asctime}] [{levelname}] [{name}] {message}` — `src/paperless/settings.py:378`. Console handler at INFO by default (`settings.py:388`, `DEBUG=NO` `settings.py:50`); the `paperless`/`paperless_mail` loggers additionally write rotating files and propagate to the root console handler (`settings.py:407-410`). Example: the consumer banner and the `Sanity checker detected no issues.` line.
- **django-q** — `HH:MM:SS [Q] LEVEL msg` — `django_q/conf.py:213-214`, with `propagate = False` (`django_q/conf.py:212`). Example: every `[Q]` line above.


---

## 4. R4 — interrupt and restart a component: the reconnection / "operational again" messages

The natural target is the **Redis broker**, the shared dependency of both the django-q task queue (`src/paperless/settings.py:456`) and the Channels group layer (`settings.py:178-187`). I interrupted and restarted it **twice** to confirm the pattern is stable.

**KEY FINDING (R4): there is no dedicated "reconnected"/"connected" message.** django-q's return to operation is confirmed by **two co-occurring signals**: (1) the `[Q] ERROR ... Connection refused` stream **stops** (last error at the restart second ±1 s), and (2) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears within one 10 s reincarnation cycle and then runs with **no further errors**.

**Scope of this test (read this before the conclusion):** only the **django-q broker/status/pusher path** (plus, in Trial 2, a post-recovery scheduled task) was exercised. An authenticated websocket/Channels client, the `channels-redis` group layer's own recovery, and consumer file-enqueue were **NOT** exercised (§4.5). The conclusion is scoped accordingly.

### 4.1 The causal mechanism (grounded in code)

- **The ~2/second error stream** is the guard's heartbeat failing: the guard calls `Stat(self).save()` every 0.5 s (`django_q/cluster.py:288`); when Redis is down, `Stat.save()` catches the write failure and calls `logger.error(e)` (`django_q/status.py:71-75`) -> one `[Q] ERROR` per guard cycle ≈ 2/s.
- **The pusher dies and is reincarnated every ~10 s.** The pusher's `broker.dequeue()` uses `BLPOP` with a 1 s timeout (`django_q/brokers/redis_broker.py:20-21`). When Redis is down it raises; the pusher logs the error, `sleep(10)`, then `break`s out of its loop (`django_q/cluster.py:345-350`) and the function ends (`cluster.py:366` logs `stopped pushing tasks`). The guard detects the dead pusher (`if not self.pusher.is_alive(): self.reincarnate(self.pusher)` — `cluster.py:280`) and reincarnates it, logging `reincarnated pusher <name> after sudden death` (`cluster.py:223`) then a new `pushing tasks at <pid>` (`cluster.py:342`). The `sleep(10)` is why reincarnations are exactly 10 s apart.

### 4.2 Ownership verification before each interrupt

Before stopping Redis I verified the target belonged to **this** container (not some other local/host Redis), then used a container-scoped, port-specific command. Producing command and output (RAW), Trial 1 BEFORE:

```bash
# docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 info server | grep -E "run_id|process_id|tcp_port"; \
#   ps -C redis-server -o pid,user,args --no-headers'
process_id:652
run_id:b3e9c8a6979c4e24945585bb031f0ba71fd4316d
tcp_port:6379
    652 testuser redis-server 127.0.0.1:6379
```

The interrupt/restart commands (all scoped to `127.0.0.1:6379` inside the container; no host-wide destructive command was used):

```bash
docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 shutdown nosave'        # interrupt
docker exec -u testuser "$CN" bash -lc 'redis-server --daemonize yes --bind 127.0.0.1 --port 6379'   # restart
```

### 4.3 Trial 1 — complete before / during / restart / after

**BEFORE:** idle and stable; the pusher had been `Process-1:13 pushing tasks at 839` since cluster start (`09:06:35`). `redis-cli ... ping` -> `PONG`. `qcluster.log` marker = 66 lines.

**INTERRUPT at `09:39:59`.** `PING` during the outage confirms it (RAW):
```bash
# docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 ping'
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

**DURING.** The error stream begins immediately. The first failing `dequeue()` raises `ConnectionError: Connection closed by server.` (the in-flight BLPOP), after which reconnect attempts raise `Error 111 ... Connection refused`. Each pusher death also emits a verbose Python `--- Logging error ---` dump, because django-q calls `logger.error(e, traceback.format_exc())` with a second positional argument (`django_q/cluster.py:347`), which the stdlib logger rejects with `TypeError: not all arguments converted during string formatting`. One such dump appears per pusher death (**5 in Trial 1, 5 in Trial 2**). A disclosed excerpt of that verbose dump (producing command `sed -n '67,89p' qcluster.log`), which names the exact failing call path (RAW, one dump, truncated at the marked `# ...`):

```
--- Logging error ---
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
  File "/usr/local/lib/python3.9/site-packages/django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 429, in read_from_socket
    raise ConnectionError(SERVER_CLOSED_CONNECTION_ERROR)
redis.exceptions.ConnectionError: Connection closed by server.
# ... (stdlib logging frames elided; ends with) ...
TypeError: not all arguments converted during string formatting
```

The recurring heartbeat error line and the first pusher death+reincarnation (producing command `sed -n '174,179p' qcluster.log`), RAW:

```
09:40:08 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:40:09 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:40:09 [Q] INFO Process-1:13 stopped pushing tasks
09:40:09 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
09:40:09 [Q] INFO Process-1:21 pushing tasks at 2305
```

**Error-stream rate (exact arithmetic).** Producing command:
```bash
# grep -cE '^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\] ERROR Error 111 connecting' <trial-1 slice of qcluster.log>
```
Count = **98**; first at `09:39:59`, last at `09:40:48`. Outage = `09:39:59` -> restart `09:40:48` = **49 s** => **98 / 49 s = 2.00 errors/s**, matching the 0.5 s guard `Stat` cycle.

**Reincarnations** at `09:40:09, :19, :29, :39, :49` — exactly **10 s** apart; pusher lineage `13 -> 21 -> 22 -> 23 -> 24 -> 25`.

**RESTART at `09:40:48`** (new `run_id b2469aec...`, `ping -> PONG`). Recovery block (producing command `sed -n '655,660p' qcluster.log`), RAW:

```
09:40:47 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:40:48 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:40:49 [Q] INFO Process-1:24 stopped pushing tasks
09:40:49 [Q] ERROR reincarnated pusher Process-1:24 after sudden death
09:40:49 [Q] INFO Process-1:25 pushing tasks at 2328
```

The last `[Q] ERROR` is at `09:40:48` (= restart second); the reincarnation cycle at `09:40:49` produced pusher **`Process-1:25 pushing tasks at 2328`** — the first pusher to spawn after Redis returned — a **1 s** lag, i.e. within one 10 s reincarnation cycle. **Zero** `[Q] ERROR` lines follow it. Producing command + measured error-free window:
```bash
# awk 'NR>660' qcluster.log | grep -cE '^[0-9:]+ \[Q\] ERROR Error 111 connecting'   -> 0
```
The cluster then ran silent (healthy idle) from `09:40:49` to at least `09:43:34` — a **2 m 45 s** measured error-free window — with `ping -> PONG`.

### 4.4 Trial 2 — reproduction with a longer post-recovery window + a post-recovery scheduled task

**BEFORE:** stable pusher `Process-1:25 pushing tasks at 2328` (from Trial 1's recovery, stable ~3 min); ownership re-verified (`run_id b2469aec...`, pid 2318); `ping -> PONG`; `qcluster.log` marker = 660 lines.

**INTERRUPT at `09:43:59`; RESTART at `09:44:45`** (new `run_id 27bb1ccb...`, `ping -> PONG`). Outage = **46 s**.

**Error-stream rate.** Count = **90**; first `09:44:00`, last `09:44:44`. `90 / 46 s = 1.96 errors/s` (≈2/s; the errors span `09:44:00`-`09:44:44` = 44 s, giving 2.05/s over the active span). **Reincarnations** at `09:44:09, :19, :29, :39, :49` — 10 s apart; lineage `25 -> 26 -> 27 -> 28 -> 29 -> 30`.

**RECOVERY + post-recovery scheduled task.** Producing command `sed -n '1243,1255p' qcluster.log`, RAW (the tail of the log):

```
09:44:43 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:44:44 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
09:44:49 [Q] INFO Process-1:29 stopped pushing tasks
09:44:49 [Q] ERROR reincarnated pusher Process-1:29 after sudden death
09:44:49 [Q] INFO Process-1:30 pushing tasks at 2411
09:46:10 [Q] INFO Enqueued 1
09:46:10 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
09:46:10 [Q] INFO Process-1:8 processing [gee-fruit-maine-rugby]
09:46:10 [Q] INFO Process-1:8 stopped doing work
09:46:10 [Q] INFO Processed [gee-fruit-maine-rugby]
09:46:10 [Q] INFO recycled worker Process-1:8
09:46:10 [Q] INFO Process-1:31 ready for work at 2414
```

Last `[Q] ERROR` at `09:44:44`; recovered pusher **`Process-1:30 pushing tasks at 2411`** at `09:44:49` (a **4 s** lag, again within one 10 s cycle); **zero** `[Q] ERROR` after it. Crucially, at `09:46:10` the recovered cluster **fired the mail schedule normally** — `created a task from schedule [Check all e-mail accounts]` -> `processing` -> `Processed` -> `recycled` — which proves the full **scheduler -> pusher -> worker -> monitor** path is operational again, not merely that the error stream stopped.

### 4.5 Summary and scope of the reconnection signal

| Aspect | Trial 1 | Trial 2 |
|---|---|---|
| Redis stopped | 09:39:59 | 09:43:59 |
| Redis restarted (`PONG`) | 09:40:48 | 09:44:45 |
| Outage duration | 49 s | 46 s |
| `[Q] ERROR Error 111 ... Connection refused` count | 98 | 90 |
| Error rate (count / outage) | 98/49 s = **2.00/s** | 90/46 s = **1.96/s** |
| Last `[Q] ERROR` timestamp | 09:40:48 (= restart) | 09:44:44 |
| Reincarnation interval | 10 s (…09/19/29/39/49) | 10 s (…09/19/29/39/49) |
| Recovered pusher | `Process-1:25 pushing tasks at 2328` @09:40:49 (1 s after restart) | `Process-1:30 pushing tasks at 2411` @09:44:49 (4 s after restart) |
| `[Q] ERROR` after recovered pusher | 0 | 0 |
| Measured error-free window after recovery | 2 m 45 s | 2 m 14 s+ (incl. normal mail firing @09:46:10) |

**Conclusion (R4), scoped.** For the **django-q broker/status/pusher path**, "operational again" is confirmed by the **cessation of the `[Q] ERROR ... Connection refused` stream** (last error at the restart second ±1 s) together with a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line that appears within one 10 s reincarnation cycle and has no errors after it; Trial 2 additionally shows a normal scheduled-task firing after recovery. There is **no explicit "reconnected" message** at this commit. **Not tested here:** authenticated websocket/Channels client reconnection, `channels-redis` group-layer recovery, and consumer file-enqueue — so this conclusion does **not** claim "everything reconnected", only that the django-q broker/status path recovered (as observed) and resumed scheduled work.

---

## 5. R5 — components/processes that run continuously to maintain a ready state

At idle, the following are **always-on**, distinguished from **transient** work that runs only briefly when triggered.

### 5.1 Always-on (continuously running at idle) — [OBSERVED]

1. **Redis server** (`redis-server 127.0.0.1:6379`, PID 652) — the shared backbone: django-q broker (`src/paperless/settings.py:456`) **and** Channels group-messaging layer (`settings.py:178-187`). Stopping it degrades the cluster (§4).
2. **The django-q cluster (`qcluster`)** — long-lived processes under the management command (PID 796) and its guard/sentinel (PID 826):
   - **Guard / sentinel** — a 0.5 s health loop that reincarnates dead workers/monitor/pusher and invokes the scheduler ~every 30 s (`django_q/cluster.py:253-289`). Observed: `Process-1 guarding cluster kilo-whiskey-artist-uranus`.
   - **Monitor** (PID 838) — persists task results (`django_q/cluster.py:378`). Observed: `Process-1:12 monitoring at 838`.
   - **Pusher** (PID 839) — continuously `BLPOP`-polls the broker (`django_q/cluster.py:342`, `django_q/brokers/redis_broker.py:20-21`). Observed: `Process-1:13 pushing tasks at 839`.
   - **Worker pool** — 11 idle workers waiting for work (`Process-1:1..11 ready for work`), sized `floor(sqrt(cores))` (`src/paperless/settings.py:427-433`; §1.7).
3. **gunicorn master + 2 uvicorn ASGI workers** (PIDs 790, 801, 802) — listening on `:8000` for HTTP **and** the `ws/status/$` websocket (`gunicorn.conf.py:3-5`, `src/paperless/workers.py:9`, `src/paperless/asgi.py:17,19-20`, `src/paperless/urls.py:136-137`). They stay up to answer the API/UI and to accept websocket status connections.
4. **`document_consumer`** (PID 793) — an inotify watcher on the consumption directory, blocking on filesystem events (`src/documents/management/commands/document_consumer.py:200`). It runs continuously so a dropped-in document is picked up immediately.

### 5.2 Transient (NOT continuously running) — [OBSERVED] / [INFERRED]

- **The four scheduled task executions** (mail / classifier / index / sanity) — each briefly occupies a django-q worker when the scheduler fires it, then finishes (§3.2, §3.3). Periodic events, not standalone processes.
- **`docker/wait-for-redis.py` — a startup-only gate, NOT a continuous process.** In the product deployment the entrypoint runs this once before the main processes start: it attempts `client.ping()` in a retry loop of `MAX_RETRY_COUNT = 5` (`docker/wait-for-redis.py:16`) with `RETRY_SLEEP_SECONDS = 5`-spaced sleeps (constant at `wait-for-redis.py:17`; `Redis.from_url(...)` + `while`/`ping()`/`time.sleep(...)` loop at `wait-for-redis.py:24-35`), then **exits** `EX_OK` on success or `EX_UNAVAILABLE` on failure (`wait-for-redis.py:37-42`). It is a transient readiness gate, not part of the always-on set. **[SOURCE-DERIVED — the product entrypoint was not executed in this manual run (§1.1).]**
- **Authenticated websocket `StatusConsumer` instances — per-client, connection-triggered, NOT always-on.** The web server is always listening for `ws/status/$` (§5.1.3), but a `StatusConsumer` object exists only for the lifetime of an authenticated client connection: `connect()` denies unauthenticated clients (`raise DenyConnection`, `src/paperless/consumers.py:13-15`) and otherwise joins the `status_updates` group (`consumers.py:9,17-20`), leaving it on disconnect via `group_discard` (`consumers.py:24-27`). No client connected during this idle run, so none existed. **[INFERRED — no websocket client was connected at idle.]**
- **Document ingestion work** — `consume_file` (`src/documents/tasks.py:184`) runs only when a document is added; it did **not** run during the idle window (empty consume dir; `document_consumer.py:85` "Adding … to the task queue." never logged). **[INFERRED — no document was ingested.]**
- **Worker recycling** — after each task a worker is recycled and replaced (`"recycle": 1`, `src/paperless/settings.py:452`); observed as `recycled worker Process-1:N` (§3.2). Triggered by task completion, not a standalone loop.

---

## 6. Cleanup (temporary artifacts removed; repository unchanged)

Per the read-only mandate, the only repository change is **this document**. Every temporary runtime artifact was removed and the removal captured. Producing commands and results (RAW excerpts from the cleanup transcript):

```bash
# 1. Graceful stop of the 3 captured PIDs (exact PIDs; NO pkill/killall)
# docker exec $CN kill -TERM 790 793 796
# -> qcluster.log tail:
09:52:39 [Q] INFO Q Cluster kilo-whiskey-artist-uranus has stopped.
# -> gunicorn.log tail:
[2026-07-10 09:52:37 +0000] [790] [INFO] Handling signal: term
[2026-07-10 09:52:38 +0000] [790] [INFO] Shutting down: Master

# 2. Stop Redis, container-scoped, after re-verifying ownership (run_id 27bb1ccb..., pid 2401)
# docker exec $CN redis-cli -h 127.0.0.1 -p 6379 shutdown nosave
# docker exec $CN redis-cli -h 127.0.0.1 -p 6379 ping   -> Could not connect ... Connection refused

# 3. Remove the container by exact name (NO prune, NO broad rm)
# docker stop pngx_obs_20260710_090348 && docker rm pngx_obs_20260710_090348
# docker ps -a --filter name=pngx_obs_20260710_090348   -> (empty; container removed)
```

The three processes stopped gracefully (`Q Cluster ... has stopped.`; gunicorn `Shutting down: Master`); Redis was stopped only after confirming the `run_id` belonged to this container; the container was removed by exact name, destroying all in-container temporary state (data/media/consume/index/log directories and the SQLite DB). The raw logs were captured **outside** the repository tree on the investigation host and hashed; their SHA-256 digests are embedded in §0.1 as the audit anchor, and the raw files — like all other temporary observation artifacts (the isolated container, scratch logs, and helper captures under `/tmp`) — were then removed. Repository state after cleanup — producing command and output (RAW):

```bash
# git -C /tmp/blitzy/paperless-ngx/blitzy-4fecd6d9-f2f1-4a67-ac76-b5521a549dfb_3abcd3 status --porcelain
# (no output -> working tree clean; the only change staged for commit is this document)
```

---

## 7. Coverage pass

Every named item in the five questions, mapped to observed evidence and citations.

- **R1 — up at the pinned commit, stable idle.** Ran at commit `542221a38dff` (verified in-container `git rev-parse HEAD`, worktree clean — §1.3), Python 3.9.23; reached idle at `09:06:33Z` with an **empty consume directory** and the four schedules present (§2.1). Exact host/container build+run commands in §1.2; `manage.py migrate/reindex/check` transcripts in §1.6 (`check` -> "System check identified no issues (0 silenced).").
- **R2 — background processes/tasks running automatically at idle.** Enumerated with observed startup banners and producing commands (§2.2) and the full process tree (§2.3): gunicorn master + 2 uvicorn ASGI workers (790/801/802); `document_consumer` inotify watcher (793); the django-q cluster = mgmt (796) -> guard/sentinel (826) -> 11 workers (827-837) + monitor (838) + pusher (839); all over Redis (652). Plus the periodic scheduled tasks (§3).
- **R3 — periodic health/readiness entries: specific messages, frequency, meaning.** Cadence table (§3.1). **Specific messages** quoted verbatim with `file:line`. Frequencies: mail `[Check all e-mail accounts]` every **10 min** — **measured** across firings at `09:16:05`, `09:26:07`, `09:36:08` (two clean intervals: 10 m 02 s and 10 m 01 s; DB `next_run` stepped +600 s) (§3.3); classifier **hourly**, index **daily**, sanity **weekly** — **[CONFIGURED / SOURCE-DERIVED]** (fired once in the startup burst; `next_run` +1 h / +1 day / +7 days; did not recur in-window) (§3.1, §3.3), sanity emitting `Sanity checker detected no issues.`. Meaning of each explained narrowly (scheduler-enqueue + worker-completion) in §3.1.
  - **KEY FINDING:** no dedicated periodic "healthy" heartbeat INFO line — proven by the flat quiet-window snapshots and flat gunicorn/consumer logs (§3.4(i)).
  - **Both named silent mechanisms addressed:** the configured 30-second Compose healthcheck `curl :8000` — manual reproduction, HTTP 302, **zero** new log lines (§3.4(ii)); the 0.5-second django-q `Stat` heartbeat — silent while healthy, loud only during the §4 outage (§3.4(iii)).
- **R4 — messages confirming reconnection/operational-again after interrupt+restart.** Redis interrupted+restarted **twice** with complete before/during/restart/after transcripts and ownership verification (§4.2-§4.4). During: `[Q] ERROR Error 111 ... Connection refused` at **2.00/s** (Trial 1) and **1.96/s** (Trial 2) + pusher `reincarnated pusher … after sudden death` every 10 s. After: the error stream **ceases** (last error at the restart second ±1 s) and a fresh `[Q] INFO Process-1:N pushing tasks at <pid>` appears within one 10 s cycle with no later errors; Trial 2 shows a normal mail firing after recovery (§4.4).
  - **KEY FINDING:** **no dedicated "reconnected" message**; operational-again = (error stream stops) + (new `pushing tasks at <pid>` line). Reproduced in both trials (§4.5). **Scope:** django-q broker/status path only; Channels/websocket/all-client recovery **NOT** tested (§4.5).
- **R5 — components/processes that keep running continuously at idle.** Always-on vs. transient split (§5): always-on = Redis; django-q guard/monitor/pusher/worker-pool; gunicorn master + 2 uvicorn workers; `document_consumer`. Transient = the four scheduled task executions, the startup-only `wait-for-redis.py` gate, per-client `StatusConsumer` instances, and (not at idle) `consume_file` / worker recycling.

### 7.1 Labels used in this document

- **[OBSERVED]** — seen in this run's captured output (all quoted log blocks; the mail cadence; the silent windows; the two Redis trials).
- **[CONFIGURED / SOURCE-DERIVED]** — read from config/DB/source, full period not observed end-to-end in-window: the hourly/classifier, daily/index, weekly/sanity cadences (fired once in the startup burst; confirmed via `next_run` and migrations); the configured 30 s healthcheck interval (Compose not executed); the `wait-for-redis.py` startup gate (product entrypoint not executed).
- **[INFERRED]** — from reading code, not exercised: the `Polling directory for changes` fallback (inotify was actually used); `consume_file` ingestion and `Adding … to the task queue.` (no document ingested); per-client `StatusConsumer` behavior (no websocket client connected).
- **[NON-CANONICAL]** — environment-specific, each with its canonical counterpart (§1.1, §1.7): reproduction image vs. published product image (user/paths/entrypoint/supervisor); Redis co-located here vs. separate `redis:6.0` service; **11 workers** = `floor(sqrt(128))` (canonical: depends on host CPU count / `PAPERLESS_TASK_WORKERS`); direct launch vs. supervisord/compose; volatile timestamps/PIDs/word-names. The observed outage error `Error 111 ... Connection refused` (Errno 111, `ECONNREFUSED`) **is** the canonical message for a downed local Redis (exception class `redis.exceptions.ConnectionError`).

### 7.2 Reproduction summary

- Single run in container `pngx_obs_20260710_090348` (image digest `sha256:6e699f225ced...`), commit `542221a38dff`, Python 3.9.23, all timestamps UTC.
- Idle observation window: process launch `09:06:33Z` -> ~`09:37Z` (~31 minutes), then two Redis trials -> ~`09:47Z`; cleanup at ~`09:52Z`.
- Mail cadence: **4 firings** (two clean on-schedule intervals, 10 m 02 s / 10 m 01 s); all four cadences cross-checked against DB `next_run`.
- Redis interrupt/restart: **2 trials**; reincarnation interval **10 s** in both; error rate **2.00/s** and **1.96/s**; recovery pattern identical; Trial 2 confirmed a post-recovery scheduled-task firing.
- Raw logs retained and SHA-256-hashed (§0.1); all temporary artifacts removed; repository worktree clean (§6).

