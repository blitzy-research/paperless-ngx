# Paperless-NGX Steady-State ("Up and Idle") Runtime Behavior

**Repository:** paperless-ngx
**Commit (pinned):** `542221a38dff06361e07976452f9aea24d210542`
**Investigation type:** Read-only runtime characterization (observe by RUNNING; no source file modified).

This document answers five questions about how Paperless-NGX behaves once it is **up and idle** (running, stable, with **no documents being ingested**). Every behavioral claim is backed by log output captured at runtime, together with the **exact command** that produced it and a **`file:line` citation** to the code that emits it.

---

## 0. Provenance and evidence conventions

### 0.1 The two independent runs all evidence comes from

Evidence in this document comes from **two fully independent runs**, each in its **own container** with its **own** freshly-migrated SQLite database, its **own** Redis server, and its **own** set of the three processes. They share nothing:

- **Run A — the primary observation run.** All topology, banners, startup burst, idle-silence, healthcheck, and the two Redis interrupt/restart trials come from Run A.
- **Run B — an independent confirmation run.** A second, separately-created container used to re-measure the mail cadence from scratch (fresh migrations, fresh Redis, fresh processes), so the ~10-minute period is confirmed **across two runs**, not merely across two adjacent intervals of one run (§3.3).

| Provenance field | Run A | Run B |
|---|---|---|
| Container name | `pngx_obsA_20260710_182631` | `pngx_obsB_20260710_182219` |
| Container id | `ecddce77da8d27699422ffca98bedae81c1a0ca7664979c2907bd6e224a6d725` | `bda883d180f1bc0c2cc103c28284adba1be6cf421fad252359d7777a60efc756` |
| Image ref | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` | (same image) |
| Image digest | `sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f` | `sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f` |
| Repo commit (in-container) | `542221a38dff06361e07976452f9aea24d210542` | `542221a38dff06361e07976452f9aea24d210542` |
| Repo worktree | clean (`git status --porcelain` empty) | clean (`git status --porcelain` empty) |
| Container timezone | `Etc/UTC` (`+0000`) — **all timestamps below are UTC** | `Etc/UTC` |
| Process launch time (`LAUNCH_UTC`) | `2026-07-10T18:27:03Z` | `2026-07-10T18:23:11Z` |
| django-q cluster name (this run) | `undress-illinois-kitten-item` | `minnesota-tennessee-nineteen-seventeen` |

**Raw transcripts are embedded, not deleted.** The three long-lived processes were launched to their own log files inside each container (`/app/obs_logs/{gunicorn,consumer,qcluster}.log`). The **complete, relevant raw transcripts are embedded inline** in §§1.6, 2.1–2.3, 3.2–3.4, and 4.2–4.4 below — **every quoted log block IS the raw file content**, reproduced exactly (with the elision convention of §0.2 applied only where explicitly marked). As **supplementary** provenance, the SHA-256 digests of the full log files (captured to the investigation host before teardown, §6) are recorded here so a reader who independently reproduces a run can confirm a byte-for-byte match:

```
Run A  qcluster.log   sha256 5d754a6f38542accbb7496ca4f0808367fed593cb247d8edf0085b0c836ff03f   (1505 lines)
Run A  gunicorn.log   sha256 16a9d16abe8c619a1639f6ae72ea050466f3d561bc921ad1dc5fdcdcc64fd56f   (4 lines)
Run A  consumer.log   sha256 d73528e6be663db0d45796cb0fad7e27f1d785f2be7d84250b7bdff041a23b60   (1 line)
Run A  launch_meta.txt sha256 ef37ee8ad15d6282ee8740bb68e084532600f8ad1c3fbea943cb0acb7eb5f928  (4 lines)
Run B  qcluster.log   sha256 aa266c614c83edb1271cffc33d5b82f973ade8dec3ee011de33a06ad6663fcd4   (74 lines, incl. graceful-stop)
Run B  gunicorn.log   sha256 6886e4311f1ee79708cba29ec3a4c9b96c43ffbeca88a23c05058d5cae5eeeff   (4 lines)
Run B  consumer.log   sha256 65f4e2c776eeb39f7cc81d6a2be858a2eac4cb90451cbc1858fed01391f5456e   (1 line)
Run B  launch_meta.txt sha256 826e8ef1741d23de8621b0e1ce837844ed89b6914452db4aa6b1542b778992c7  (4 lines)
```

### 0.2 How to read the log blocks (raw vs annotated; truncation)

To keep every claim auditable, the log blocks below follow strict conventions:

- **RAW** blocks reproduce log lines exactly as written by the process. They contain no editorial text.
- Where a block is labeled **[annotated]**, any text after a `#` on a line — or a `# ...` marker on its own line — is **my annotation**, not part of the log. `# ...` explicitly marks lines I elided (always with the count/kind of what was elided). This convention is used sparingly and only where noted.
- Counts (line counts, error counts) were produced by the exact `grep -c` / `wc -l` command shown next to them, run against the retained raw files.

### 0.3 Labels

- **[OBSERVED]** — seen directly in a run's captured output.
- **[CONFIGURED / SOURCE-DERIVED]** — a value read from configuration or a database row in this run, or read from source code; used where a full period was **not** observed end-to-end (e.g. the hourly/daily/weekly cadences).
- **[INFERRED]** — derived from reading code, not exercised at runtime.
- **[NON-CANONICAL]** — an environment-specific value or a deviation from the published product packaging, always paired with its canonical counterpart.

---

## TL;DR — the two key findings

1. **There is NO dedicated periodic "healthy" heartbeat INFO log line.** When Paperless-NGX is idle and healthy, all three processes are **silent at INFO level**; the only recurring log activity naturally observed is the firing of the **`Check all e-mail accounts` schedule every ~10 minutes** (confirmed across two independent runs, §3.3). The three lower-frequency schedules (classifier hourly, index daily, sanity weekly) did not recur within the observation window — their cadence is **[CONFIGURED / SOURCE-DERIVED]** (§3.1, §3.3). Two other health mechanisms — the Docker Compose `curl` liveness probe (configured every 30 s) and the django-q `Stat` write to Redis (every 0.5 s) — are **silent while healthy** (§3.4).
2. **There is NO dedicated "reconnected" log message after a component restart.** When the Redis broker is interrupted and restarted, the return to operation is confirmed by a **combination of signals**: (a) the `[Q] ERROR ... Connection refused` stream **stops** (last error at the restart second ±1 s); (b) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears within one 10 s reincarnation cycle; and (c) active round-trips prove the two Redis-backed subsystems work again — a **django-q broker** task (`async_task` → result `9.0`) and a **`channels-redis` group-layer** message (send→receive `MATCH True`), plus an HTTP `302` from the web server and a live consumer (§4). This was verified after **both** restarts.

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

All commands were issued **from the investigation host** against each container via `docker exec`. The long-lived processes were run as **`testuser`**, with `HOME=/app`, working directory **`/app/src`**, each **detached** (`setsid ... &`) to its own log file, capturing the PID (`echo $!`). This block is directly runnable and is shown for **Run A** (Run B is byte-identical except for the container name); it names the host-vs-container boundary, the user, cwd, env, redirections, backgrounding, log paths, and PID capture that the processes actually used:

```bash
# ---- HOST: create a fresh, uniquely-named, isolated container from the designated image ----
#   (--init installs a PID-1 init so exited grandchildren are reaped)
docker run -d --init --name pngx_obsA_20260710_182631 --entrypoint sleep \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 \
  infinity
CN=pngx_obsA_20260710_182631

# ---- CONTAINER: prerequisites (see caveat, §1.7). apt runs as root ----
docker exec "$CN" bash -lc 'apt-get update && apt-get install -y redis-server libzbar0 poppler-utils pngquant procps curl'

# ---- CONTAINER: Redis broker + Channels layer (canonical command), as testuser ----
docker exec -u testuser "$CN" bash -lc 'redis-server --daemonize yes --bind 127.0.0.1 --port 6379'

# ---- CONTAINER: runtime dirs, migrations (SQLite default), full-text index, startup check (from /app/src) ----
docker exec -u testuser -e HOME=/app -w /app/src "$CN" bash -lc '
  mkdir -p /app/consume /app/media /app/data /app/data/index /app/data/log /app/export /app/static
  python3 manage.py migrate
  python3 manage.py document_index reindex
  python3 manage.py check'

# ---- CONTAINER: the three canonical long-lived processes, each detached to its own log, PID captured ----
#   NOTE: LAUNCH_UTC is written FIRST (truncating create, '>'), then each PID is appended ('>>').
docker exec -u testuser -e HOME=/app -w /app/src "$CN" bash -lc '
  mkdir -p /app/obs_logs
  echo "LAUNCH_UTC=$(date -u +%Y-%m-%dT%H:%M:%SZ)" > /app/obs_logs/launch_meta.txt
  setsid gunicorn -c /app/gunicorn.conf.py paperless.asgi:application \
    >/app/obs_logs/gunicorn.log 2>&1 & echo "gunicorn_pid=$!" >>/app/obs_logs/launch_meta.txt
  setsid python3 manage.py document_consumer \
    >/app/obs_logs/consumer.log 2>&1 & echo "consumer_pid=$!" >>/app/obs_logs/launch_meta.txt
  setsid python3 manage.py qcluster \
    >/app/obs_logs/qcluster.log 2>&1 & echo "qcluster_pid=$!" >>/app/obs_logs/launch_meta.txt'
```

Captured launch metadata (`/app/obs_logs/launch_meta.txt`, Run A, RAW — the `echo LAUNCH_UTC` line above is what produces the first line, so the displayed 4-line file is exactly what the command writes):

```
LAUNCH_UTC=2026-07-10T18:27:03Z
gunicorn_pid=687
consumer_pid=688
qcluster_pid=689
```

For reference, Run B's `launch_meta.txt` (same command, independent container), RAW:

```
LAUNCH_UTC=2026-07-10T18:23:11Z
gunicorn_pid=708
consumer_pid=709
qcluster_pid=710
```

### 1.3 Provenance: pinned commit and clean worktree inside the container

Producing command and output (Run A, RAW) — confirms the code is exactly the pinned commit and that **no repository file was modified** by the run:

```bash
# docker exec -u testuser "$CN" bash -lc 'git config --global --add safe.directory /app; \
#   git -C /app rev-parse HEAD; git -C /app status --porcelain'
542221a38dff06361e07976452f9aea24d210542
# (git status --porcelain produced NO output -> worktree clean)
```

### 1.4 Effective environment (what is actually overridden)

Producing command and output (Run A, RAW):

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

**Migrations** (SQLite). Producing command `python3 manage.py migrate`; the three schedule-defining migrations applied cleanly (shown here in context with the other migrations in the same batch) and the command exited 0 (Run A, RAW + exit code):

```
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying django_q.0002_auto_20150630_1624... OK
  Applying documents.0002_auto_20151226_1316... OK
  Applying documents.1001_auto_20201109_1636... OK
  Applying documents.1004_sanity_check_schedule... OK
  Applying paperless_mail.0002_auto_20201117_1334... OK
MIGRATE_EXIT=0
```

**Full-text index** (`python3 manage.py document_index reindex`) — zero documents to index at idle (Run A, RAW + exit code):

```

0it [00:00, ?it/s]
0it [00:00, ?it/s]
REINDEX_EXIT=0
```

**System check** (`python3 manage.py check`) — the canonical startup check passes cleanly (Run A, RAW + exit code):

```
System check identified no issues (0 silenced).
CHECK_EXIT=0
```

### 1.7 Non-canonical caveats (explicitly labeled)

- **[NON-CANONICAL — packaging] Direct process launch vs. supervisord/docker-compose.** The three processes were launched directly (exactly as `docker/supervisord.conf:11,20,29` and `scripts/*.service` invoke them) rather than under `supervisord`/`docker compose`. The process invocations and code paths are identical; only the supervisor wrapper differs. *Canonical counterpart:* the same three commands started by supervisord (Docker) or systemd (bare metal).
- **[NON-CANONICAL — packaging] Prerequisites installed at runtime.** `redis-server`, `libzbar0`, `poppler-utils`, `pngquant`, `procps`, and `curl` were `apt`-installed into the container. Of these, `libzbar0`/`poppler-utils`/`pngquant`/`curl` **are** part of the product image's `RUNTIME_PACKAGES` (`Dockerfile:35-75`, curl at `Dockerfile:36`); **`redis-server` is not** — in the product deployment Redis is a separate `redis:6.0` service (`docker/compose/docker-compose.sqlite.yml:28-32`). Here it is co-located in the same container. Versions match the canonical toolchain (e.g. `redis-server 6.0.16`).
- **[NON-CANONICAL — user]** The container's default login user is `root`; the Paperless processes were run as **`testuser`** (uid 1000). The product image uses a `paperless` service account (uid 1000). This does not affect the logging/task behavior examined here.
- **[NON-CANONICAL — worker count] 11 django-q workers.** `multiprocessing.cpu_count()` reported **128** in this container, so `default_task_workers()` returned `floor(sqrt(128)) = 11` (`src/paperless/settings.py:427-433`; `TASK_WORKERS` `settings.py:438`; passed to `Q_CLUSTER["workers"]` `settings.py:455`). *Canonical counterpart:* the default depends on the host CPU count (and `PAPERLESS_TASK_WORKERS`); on a 4-core host it would be `floor(sqrt(4)) = 2`. **The worker-count value is host-specific; the formula is canonical.**
- **[NON-CANONICAL — volatile fields]** Timestamps, PIDs, the django-q cluster word-names (`undress-illinois-kitten-item` in Run A, `minnesota-tennessee-nineteen-seventeen` in Run B) and per-task word-names vary per run.
- **The web-server liveness probe returns HTTP 302** (redirect to the login page for an unauthenticated request to `/`); `curl -f` treats this as success (§3.4).

---

## 2. R1 — the system up at a stable idle state; R2/R5 — idle background processes

### 2.1 Reaching idle (R1)

After the commands in §1.2, the system reached a stable idle state at process launch (Run A, `2026-07-10T18:27:03Z`). "Idle" means the consume directory is empty and no tasks are executing. Producing command and output (Run A, RAW):

```bash
# docker exec -u testuser "$CN" bash -lc 'ls -la /app/consume'
total 12
drwxr-sr-x 2 testuser testuser 4096 Jul 10 18:27 .
drwxr-sr-x 1 testuser testuser 4096 Jul 10 18:27 ..
```

The four periodic schedules that django-q must drive were created by the migrations (§1.6) and confirmed present in the database. Producing command `python3 manage.py shell -c "...Schedule.objects..."`; output (Run A, RAW):

```
[["Train the classifier", "documents.tasks.train_classifier", "H", null], ["Optimize the index", "documents.tasks.index_optimize", "D", null], ["Perform sanity check", "documents.tasks.sanity_check", "W", null], ["Check all e-mail accounts", "paperless_mail.tasks.process_mail_accounts", "I", 10]]
```

These correspond exactly to the schedule-defining migrations: `Train the classifier` (HOURLY) and `Optimize the index` (DAILY) from `src/documents/migrations/1001_auto_20201109_1636.py:10-19`; `Perform sanity check` (WEEKLY) from `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; `Check all e-mail accounts` (MINUTES, `minutes=10`) from `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15`.

### 2.2 The three processes and their startup banners (R2/R5)

Each process was launched with the exact command shown in §1.2, writing its stdout+stderr to its own log file. The producing command for each banner is `docker exec "$CN" cat <logfile>`; the PIDs are cross-referenced to the process tree in §2.3. All values below are from **Run A**.

#### (a) `gunicorn` — web server + websockets (gunicorn master + 2 uvicorn ASGI workers)

**Command:** `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` (from `/app/src`, as `testuser`). Producing command: `cat /app/obs_logs/gunicorn.log`. Output (RAW — the complete file is 4 lines):

```
[2026-07-10 18:27:03 +0000] [687] [INFO] Starting gunicorn 20.1.0
[2026-07-10 18:27:03 +0000] [687] [INFO] Listening at: http://0.0.0.0:8000 (687)
[2026-07-10 18:27:03 +0000] [687] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-10 18:27:03 +0000] [687] [INFO] Server is ready. Spawning workers
```

- `bind = 0.0.0.0:8000`, `workers = 2`, `worker_class = "paperless.workers.ConfigurableWorker"`, `timeout = 120` — `gunicorn.conf.py:3-6`.
- `Server is ready. Spawning workers` is emitted by the repo's `when_ready(server)` hook — `gunicorn.conf.py:17-18`. The `Listening at:` and `Starting gunicorn 20.1.0` lines are gunicorn's own core startup logs.
- `ConfigurableWorker` subclasses the uvicorn `UvicornWorker` (an ASGI worker) — `src/paperless/workers.py:9`. The ASGI application wires both `http` and `websocket` protocols via `ProtocolTypeRouter` — `src/paperless/asgi.py:17`, `asgi.py:19-20`; the websocket route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` — `src/paperless/urls.py:136-137`, and `StatusConsumer` joins the Channels group `status_updates` — `src/paperless/consumers.py:9,17-20`.
- **Observed processes:** master **PID 687** with **two** uvicorn ASGI worker children **PID 723** and **PID 724** (`PPid 687`), i.e. `workers=2` as configured (§2.3).

#### (b) `document_consumer` — the consumption-directory watcher

**Command:** `python3 manage.py document_consumer`. Producing command: `cat /app/obs_logs/consumer.log`. Output (RAW — the complete file is 1 line):

```
[2026-07-10 18:27:04,259] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

- This readiness banner is emitted by `handle_inotify()` — `src/documents/management/commands/document_consumer.py:200` (logger `paperless.management.consumer`, defined at `document_consumer.py:24`). If inotify were unavailable it would instead log `Polling directory for changes: <dir>` — `document_consumer.py:186` **[INFERRED — inotify was actually used]**.
- At idle this watcher is **silent**. It only logs when a file arrives: `Adding {filepath} to the task queue.` — `document_consumer.py:85`. That line **did not appear** during the idle window (no documents ingested). **PID 688** (§2.3).

#### (c) `qcluster` — the django-q task cluster (guard/sentinel + monitor + pusher + worker pool)

**Command:** `python3 manage.py qcluster`. Producing command: `head -16 /app/obs_logs/qcluster.log`. Output (RAW — first 16 lines, the complete startup banner):

```
18:27:04 [Q] INFO Q Cluster undress-illinois-kitten-item starting.
18:27:04 [Q] INFO Process-1:1 ready for work at 751
18:27:04 [Q] INFO Process-1:2 ready for work at 752
18:27:04 [Q] INFO Process-1:3 ready for work at 753
18:27:04 [Q] INFO Process-1:4 ready for work at 754
18:27:04 [Q] INFO Process-1:5 ready for work at 755
18:27:04 [Q] INFO Process-1:6 ready for work at 756
18:27:04 [Q] INFO Process-1:7 ready for work at 757
18:27:04 [Q] INFO Process-1:8 ready for work at 758
18:27:04 [Q] INFO Process-1:9 ready for work at 759
18:27:04 [Q] INFO Process-1:10 ready for work at 760
18:27:04 [Q] INFO Process-1:11 ready for work at 761
18:27:04 [Q] INFO Process-1:12 monitoring at 762
18:27:04 [Q] INFO Process-1 guarding cluster undress-illinois-kitten-item
18:27:04 [Q] INFO Process-1:13 pushing tasks at 763
18:27:04 [Q] INFO Q Cluster undress-illinois-kitten-item running.
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
- **Observed process tree:** the `qcluster` management command is **PID 689**; it spawned the guard/sentinel **PID 750**, which forked the 11 workers (PIDs 751-761), the monitor (PID 762) and the pusher (PID 763) — §2.3.
- **Run B cross-check** (independent container): the identical banner appeared with cluster name `minnesota-tennessee-nineteen-seventeen`, 11 workers (PIDs 738-748), monitor (749) and pusher (750) — confirming the startup banner is stable across runs.

### 2.3 Full idle process topology (observed)

Producing command and output (Run A, RAW), captured at `2026-07-10T18:27:04Z` (pre-startup-burst, so the original 11 workers 751-761 are still present):

```bash
# docker exec "$CN" bash -lc 'ps -eo pid,ppid,user,args | grep -E "redis-server|gunicorn|document_consumer|qcluster" | grep -v grep'
    611       1 testuser redis-server 127.0.0.1:6379
    687       1 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    688       1 testuser python3 manage.py document_consumer
    689       1 testuser python3 manage.py qcluster
    723     687 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    724     687 testuser /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    750     689 testuser python3 manage.py qcluster
    751     750 testuser python3 manage.py qcluster
    752     750 testuser python3 manage.py qcluster
    753     750 testuser python3 manage.py qcluster
    754     750 testuser python3 manage.py qcluster
    755     750 testuser python3 manage.py qcluster
    756     750 testuser python3 manage.py qcluster
    757     750 testuser python3 manage.py qcluster
    758     750 testuser python3 manage.py qcluster
    759     750 testuser python3 manage.py qcluster
    760     750 testuser python3 manage.py qcluster
    761     750 testuser python3 manage.py qcluster
    762     750 testuser python3 manage.py qcluster
    763     750 testuser python3 manage.py qcluster
```

The 18 lines above are the complete, unelided `ps` capture (redis + gunicorn master + 2 gunicorn workers + consumer + qcluster mgmt + guard + 11 workers + monitor + pusher). Summary:

```
redis-server (PID 611, 127.0.0.1:6379)
qcluster mgmt (PID 689) -> guard/sentinel (PID 750) -> 11 workers (751-761) + monitor (762) + pusher (763)
document_consumer (PID 688)  -- inotify watch on /app/consume
gunicorn master (PID 687) -> 2 uvicorn ASGI workers (723, 724) on :8000
```

(Worker index numbers climb over the run as django-q recycles each worker after a task — `"recycle": 1`, `src/paperless/settings.py:452` — so a later `ps` shows higher PIDs for the workers; the *count* stays 11.)

---

## 3. R3 — periodic health/readiness log entries: messages, frequency, and meaning

**KEY FINDING (proven below): there is no dedicated periodic "healthy" heartbeat INFO log line.** While idle and healthy, the three processes are silent at INFO. The only recurring log activity **observed** is the firing of the `Check all e-mail accounts` schedule every ~10 minutes. Two additional health mechanisms run continuously but are **silent** unless something is wrong.

### 3.1 The cadence table

| Recurring signal | Frequency | Basis | Meaning / Source |
|---|---|---|---|
| `... created a task from schedule [Check all e-mail accounts]` | every **10 minutes** | **[OBSERVED]** — fired on-schedule and re-measured in a second independent run; two clean intervals in each run (§3.3) | Scheduler enqueues the mail-poll task; `process_mail_accounts` — `src/paperless_mail/tasks.py:11`; schedule `Schedule.MINUTES, minutes=10` — `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` |
| `... created a task from schedule [Train the classifier]` | **hourly** | **[CONFIGURED / SOURCE-DERIVED]** — fired once in the startup burst; `next_run` stepped +1 h; did not recur in-window (§3.3) | Enqueues classifier retrain; `train_classifier` — `src/documents/tasks.py:48`; `Schedule.HOURLY` — `src/documents/migrations/1001_auto_20201109_1636.py:10-14` |
| `... created a task from schedule [Optimize the index]` | **daily** | **[CONFIGURED / SOURCE-DERIVED]** — `next_run` +1 day; did not recur in-window | Enqueues Whoosh index optimize; `index_optimize` — `src/documents/tasks.py:32`; `Schedule.DAILY` — `src/documents/migrations/1001_auto_20201109_1636.py:15-19` |
| `... created a task from schedule [Perform sanity check]` | **weekly** | **[CONFIGURED / SOURCE-DERIVED]** — `next_run` +7 days; did not recur in-window | Enqueues integrity sweep; `sanity_check` — `src/documents/tasks.py:255`; `Schedule.WEEKLY` — `src/documents/migrations/1004_sanity_check_schedule.py:10-14`; emits `Sanity checker detected no issues.` — `src/documents/sanity_checker.py:27` |
| Docker Compose healthcheck `curl -f http://localhost:8000` | every **30 seconds** (configured) | **[CONFIGURED]** — Compose not executed here; command reproduced manually (§3.4) | Liveness probe — **SILENT** (no access log by default) — `docker/compose/docker-compose.sqlite.yml:41-45` |
| django-q guard `Stat` write to Redis | every **0.5 seconds** | **[OBSERVED silent + SOURCE-DERIVED cadence]** | Cluster status write — **SILENT** unless Redis is unreachable — `django_q/cluster.py:288`, `django_q/status.py:71-75` |

**What each visible signal indicates (narrowly).** A `created a task from schedule [<name>]` line proves the django-q **scheduler** fired that schedule and the **pusher/worker** pool accepted and completed the enqueued task (the following `processing [...]` / `Processed [...]` lines). That is a liveness signal for the **scheduler -> broker -> worker** path specifically. It does **not**, by itself, prove the whole system is healthy (e.g. the web server or Channels layer), and the mail/classifier/index tasks are effectively no-ops **only because nothing is configured/queued at idle** (no mail accounts; empty index/consume) — not a guarantee they are always no-ops. The sanity task additionally emits `Sanity checker detected no issues.` when it finds no problems. The two silent mechanisms (healthcheck, `Stat` heartbeat) are continuous liveness checks that by design print nothing while healthy (§3.4).

### 3.2 The one-time startup burst (overdue schedules; `catch_up: False`)

About **~30 s** after the cluster reported `running.` (Run A: `18:27:04` -> **`18:27:33`**), the guard's scheduler ran for the first time and fired all four overdue schedules **once**. django-q's scheduler is gated in the guard loop by `if counter >= 30 and Conf.SCHEDULER:` (`django_q/cluster.py:284`), and Paperless sets `"catch_up": False` (`src/paperless/settings.py:451`), so overdue schedules fire a **single** time rather than replaying every missed run. Producing command: `sed -n '17,45p' qcluster.log`. Output (Run A, RAW):

```
18:27:33 [Q] INFO Enqueued 1
18:27:33 [Q] INFO Process-1 created a task from schedule [Train the classifier]
18:27:33 [Q] INFO Process-1:1 processing [july-fanta-idaho-island]
18:27:33 [Q] INFO Enqueued 1
18:27:33 [Q] INFO Process-1 created a task from schedule [Optimize the index]
18:27:33 [Q] INFO Process-1:2 processing [nebraska-india-low-bacon]
18:27:33 [Q] INFO Enqueued 1
18:27:33 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
18:27:33 [Q] INFO Process-1:3 processing [ohio-nuts-comet-uniform]
18:27:33 [Q] INFO Enqueued 1
18:27:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
18:27:33 [Q] INFO Process-1:4 processing [rugby-aspen-hawaii-cold]
18:27:33 [Q] INFO Process-1:4 stopped doing work
18:27:33 [Q] INFO Processed [rugby-aspen-hawaii-cold]
18:27:34 [Q] INFO Process-1:1 stopped doing work
[2026-07-10 18:27:34,019] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
18:27:34 [Q] INFO Process-1:3 stopped doing work
18:27:34 [Q] INFO Processed [july-fanta-idaho-island]
18:27:34 [Q] INFO Process-1:2 stopped doing work
18:27:34 [Q] INFO Processed [ohio-nuts-comet-uniform]
18:27:34 [Q] INFO Processed [nebraska-india-low-bacon]
18:27:34 [Q] INFO recycled worker Process-1:1
18:27:34 [Q] INFO Process-1:14 ready for work at 869
18:27:34 [Q] INFO recycled worker Process-1:3
18:27:34 [Q] INFO Process-1:15 ready for work at 870
18:27:34 [Q] INFO recycled worker Process-1:2
18:27:34 [Q] INFO Process-1:16 ready for work at 872
18:27:35 [Q] INFO recycled worker Process-1:4
18:27:35 [Q] INFO Process-1:17 ready for work at 890
```

Notes:
- `Enqueued 1` — `django_q/tasks.py:74`; `Process-1 created a task from schedule [<name>]` — `django_q/cluster.py:669`.
- `Sanity checker detected no issues.` prints in the **Paperless verbose format** `[{asctime}] [{levelname}] [{name}] {message}` (`src/paperless/settings.py:378`), **not** the `[Q]` format, because it comes from the `paperless.sanity_checker` logger (`src/documents/sanity_checker.py:24,27`), which writes to its rotating file handler **and** propagates to the root console handler (`settings.py:407-410`).
- `recycled worker Process-1:N` + a new `ready for work at <pid>` — django-q recycles each worker after a task because Paperless sets `"recycle": 1` (`src/paperless/settings.py:452`), a memory-hygiene measure; this is why worker index numbers climb over time.

### 3.3 The 10-minute mail cadence — measured in two independent runs

The `Check all e-mail accounts` task is the most frequent recurring signal and the only one whose full period repeated within the observation window. To satisfy the "confirm the value is stable across **at least two runs**" requirement **without** conflating two adjacent intervals of one run with two runs, the cadence was measured in **two independent runs** — Run A and Run B are separate containers with separate databases, separate Redis servers, and separate cluster instances (§0.1).

**Run A** — cluster `undress-illinois-kitten-item`. Producing command and output (RAW, `[annotated]` with the firing sequence):

```bash
# grep 'created a task from schedule \[Check all e-mail accounts\]' /app/obs_logs/qcluster.log
18:27:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #1 (overdue burst)
18:37:05 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #2
18:47:06 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #3
18:57:08 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #4
```

Two consecutive **on-schedule** intervals **within Run A** (these are two adjacent intervals of the *same* run, not two runs):
- **#2 -> #3 = `18:37:05` -> `18:47:06` = 10 min 01 s (601 s)**
- **#3 -> #4 = `18:47:06` -> `18:57:08` = 10 min 02 s (602 s)**

(Interval #1 -> #2 = 572 s is shorter because #1 was the *delayed overdue* firing from the startup burst, §3.2.)

**Run B** — a **genuinely independent second run** (different container `pngx_obsB_20260710_182219`, different database, different Redis, cluster `minnesota-tennessee-nineteen-seventeen`). Producing command and output (RAW):

```bash
# grep 'created a task from schedule \[Check all e-mail accounts\]' /app/obs_logs/qcluster.log   (Run B)
18:23:42 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #1 (overdue burst)
18:33:13 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #2
18:43:14 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #3
18:53:16 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]   # #4
```

Two consecutive on-schedule intervals **within Run B**:
- **#2 -> #3 = `18:33:13` -> `18:43:14` = 10 min 01 s (601 s)**
- **#3 -> #4 = `18:43:14` -> `18:53:16` = 10 min 02 s (602 s)**

**Conclusion:** the ~10-minute mail cadence is confirmed **across two independent runs** (Run A: 601 s / 602 s; Run B: 601 s / 602 s), and within each run across two consecutive intervals. The period is stable at 600 s + a few seconds of scheduling latency.

Full firing #4 block from Run A, producing command `sed -n '60,66p' qcluster.log`, output (RAW):

```
18:57:08 [Q] INFO Enqueued 1
18:57:08 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
18:57:08 [Q] INFO Process-1:7 processing [coffee-bacon-green-green]
18:57:08 [Q] INFO Process-1:7 stopped doing work
18:57:08 [Q] INFO Processed [coffee-bacon-green-green]
18:57:08 [Q] INFO recycled worker Process-1:7
18:57:08 [Q] INFO Process-1:20 ready for work at 1849
```

The `process_mail_accounts` function itself logs nothing at idle — no mail accounts are configured, so its loop over accounts is empty (`src/paperless_mail/tasks.py:13`). The only evidence it ran is the `[Q]` scheduler/worker lines above.

**Cross-check of all four cadences from the live database (Run A).** django-q advances each schedule's `next_run` by its period. Reading `next_run` at two points in the run (producing command: `python3 manage.py shell -c "...Schedule.objects..."`) shows only the mail schedule advancing in-window; the other three keep a fixed future `next_run`:

At `now=2026-07-10 18:27:46` (RAW):
```
Train the classifier | type=H | next_run=2026-07-10 19:26:39
Optimize the index | type=D | next_run=2026-07-11 18:26:39
Perform sanity check | type=W | next_run=2026-07-17 18:26:39
Check all e-mail accounts | type=I minutes=10 | next_run=2026-07-10 18:36:40
```
At `now=2026-07-10 18:59:16` (RAW):
```
Train the classifier | type=H | next_run=2026-07-10 19:26:39   # UNCHANGED
Optimize the index | type=D | next_run=2026-07-11 18:26:39   # UNCHANGED
Perform sanity check | type=W | next_run=2026-07-17 18:26:39   # UNCHANGED
Check all e-mail accounts | type=I minutes=10 | next_run=2026-07-10 19:06:40   # advanced by 3x10 min
```

So the mail `next_run` stepped `18:36:40 -> 18:46:40 -> 18:56:40 -> 19:06:40` (exactly +600 s each), while classifier/index/sanity `next_run` did **not** change (their periods exceed the ~32-minute window). This is why the mail cadence is **[OBSERVED]** end-to-end while the hourly/daily/weekly cadences are **[CONFIGURED / SOURCE-DERIVED]**.

**On the few-second offset.** The database `next_run` steps are exactly 600 s; the observed firings land a few seconds after each `next_run` boundary (e.g. `next_run 18:36:40` -> fired `18:37:05`) because the guard evaluates the scheduler only periodically (`counter >= 30`, `django_q/cluster.py:284`). The interval **between firings** is what measures the period, and it is 601 s / 602 s ≈ 600 s in both runs.

### 3.4 Proof that the idle system is otherwise SILENT (the KEY FINDING)

**(i) The processes emit nothing between task firings.** A background poller recorded `wc -l` of each log every ~30 s (`line_count_snapshots.csv`, Run A). Producing command per row: `wc -l <log>` + `tail -1 qcluster.log`. Representative RAW rows (`utc,qcluster_lines,gunicorn_lines,consumer_lines,qcluster_last_ts`):

```
18:27:03,0,0,0,
18:27:33,16,4,1,18:27:04
18:36:03,45,4,1,18:27:35
18:36:33,45,4,1,18:27:35
18:37:03,45,4,1,18:27:35
18:37:33,52,4,1,18:37:05
18:38:03,52,4,1,18:37:05
18:39:03,52,4,1,18:37:05
18:40:33,52,4,1,18:37:05
18:41:03,52,4,1,18:37:05
```

Between the mail firings the `qcluster` log stays **flat** (45 lines after the startup burst, 52 after firing #2; each firing adds exactly 7 lines: `Enqueued`/`created a task`/`processing`/`stopped doing work`/`Processed`/`recycled`/`ready for work`) and its last-line timestamp does not move. The web server and consumer logs were flat for the **entire idle window**: `gunicorn.log = 4`, `consumer.log = 1`. Producing command and its **literal stdout** (Run A, RAW — this is exactly what `wc -l` then `tail -1` print, unmodified):

```bash
# docker exec "$CN" bash -lc 'wc -l /app/obs_logs/gunicorn.log /app/obs_logs/consumer.log; \
#   echo ---; tail -1 /app/obs_logs/gunicorn.log; tail -1 /app/obs_logs/consumer.log'
  4 /app/obs_logs/gunicorn.log
  1 /app/obs_logs/consumer.log
  5 total
---
[2026-07-10 18:27:03 +0000] [687] [INFO] Server is ready. Spawning workers
[2026-07-10 18:27:04,259] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

=> **No dedicated periodic "healthy" INFO line** is emitted by any of the three processes; idle activity is limited to the scheduled-task firings (only mail recurs in-window).

**(ii) The web-server liveness probe is silent [manual reproduction of the configured Compose probe].** Docker Compose was **not** executed in this run; I reproduced its configured probe command `["CMD","curl","-f","http://localhost:8000"]` (`docker/compose/docker-compose.sqlite.yml:42`; interval 30 s, timeout 10 s, retries 5 — `:43-45`) manually with the **literal `curl` binary** (installed with the other prerequisites, §1.2; it is the canonical probe binary — the exact Compose healthcheck command and part of the product image's `RUNTIME_PACKAGES`, `Dockerfile:36`), capturing status and headers. The response headers are **deterministic** at this commit (produced by Django's `SecurityMiddleware`/`LocaleMiddleware` and the login redirect); only `date` is per-run.

The block below is **[annotated]** (a composed transcript, per §0.2): the `gunicorn.log …`, `probe #…`, and the `--- ... ---` marker lines are my wrapper `echo`s and line-count annotations; the `HTTP/1.1 …` group is `curl`'s verbatim response, shown **in full**. Producing commands: six `curl -f -s -o /dev/null -w "HTTP:%{http_code}"` probes, then one `curl -sI http://localhost:8000`, with `wc -l /app/obs_logs/gunicorn.log` before and after:

```
gunicorn.log BEFORE probes = 4 lines  @ 18:28:14
probe #1 -> HTTP:302 exit=0
probe #2 -> HTTP:302 exit=0
probe #3 -> HTTP:302 exit=0
probe #4 -> HTTP:302 exit=0
probe #5 -> HTTP:302 exit=0
probe #6 -> HTTP:302 exit=0
--- curl -sI http://localhost:8000 (full response headers) ---
HTTP/1.1 302 Found
date: Fri, 10 Jul 2026 18:28:14 GMT
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
gunicorn.log AFTER probes  = 4 lines  @ 18:28:14
```

The probe succeeds (`curl -f` accepts the `302` redirect to the login page, `location: /accounts/login/?next=/`) yet adds **zero** log lines (before and after both show 4): gunicorn/uvicorn emits **no access log by default**. So the configured 30-second liveness probe is invisible in the logs.

**(iii) The django-q `Stat` heartbeat is silent while healthy.** The guard loop writes `Stat(self).save()` to Redis every `GUARD_CYCLE = 0.5 s` (`django_q/cluster.py:288`; `GUARD_CYCLE` default `0.5` at `django_q/conf.py:90`). `Stat.save()` only logs **on failure** — `try: self.broker.set_stat(...) except Exception as e: logger.error(e)` (`django_q/status.py:71-75`). While Redis is reachable it writes silently (confirmed by the flat `qcluster.log` above); it becomes a loud `[Q] ERROR` stream only during an outage — demonstrated directly in §4.

### 3.5 Log formats (for reference)

Two distinct formats appear at idle:
- **Paperless (Django `LOGGING`)** — `[{asctime}] [{levelname}] [{name}] {message}` — `src/paperless/settings.py:378`. Console handler at INFO by default (`settings.py:388`, `DEBUG=NO` `settings.py:50`); the `paperless`/`paperless_mail` loggers additionally write rotating files and propagate to the root console handler (`settings.py:407-410`). Example: the consumer banner and the `Sanity checker detected no issues.` line.
- **django-q** — `HH:MM:SS [Q] LEVEL msg` — `django_q/conf.py:213-214`, with `propagate = False` (`django_q/conf.py:212`). Example: every `[Q]` line above.

---

## 4. R4 — interrupt and restart a component: the reconnection / "operational again" messages

The natural target is the **Redis broker**, the shared dependency of both the django-q task queue (`src/paperless/settings.py:456`) and the Channels group layer (`settings.py:178-187`). I interrupted and restarted it **twice** (both in Run A) to confirm the pattern is stable, and after each restart I actively exercised **both** Redis-backed subsystems to prove whole-system recovery, not merely that the error stream stopped.

**KEY FINDING (R4): there is no dedicated "reconnected"/"connected" message.** The return to operation is established by a **combination of signals** (detailed in §4.5): the `[Q] ERROR ... Connection refused` stream **stops**; a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears within one 10 s reincarnation cycle; a **django-q broker round-trip** completes (`async_task("math.sqrt", 81)` → `9.0`); a **`channels-redis` group-layer round-trip** completes (`group_send` → `receive`, `MATCH True`); the web server answers `HTTP 302`; and the consumer process is still alive.

### 4.1 The causal mechanism (grounded in code)

- **The ~2/second error stream** is the guard's heartbeat failing: the guard calls `Stat(self).save()` every 0.5 s (`django_q/cluster.py:288`); when Redis is down, `Stat.save()` catches the write failure and calls `logger.error(e)` (`django_q/status.py:71-75`) -> one `[Q] ERROR` per guard cycle ≈ 2/s.
- **The pusher dies and is reincarnated every ~10 s.** The pusher's `broker.dequeue()` uses `BLPOP` with a 1 s timeout (`django_q/brokers/redis_broker.py:20-21`). When Redis is down it raises; the pusher logs the error, `sleep(10)`, then `break`s out of its loop (`django_q/cluster.py:345-350`) and the function ends (`cluster.py:366` logs `stopped pushing tasks`). The guard detects the dead pusher (`if not self.pusher.is_alive(): self.reincarnate(self.pusher)` — `cluster.py:280`) and reincarnates it, logging `reincarnated pusher <name> after sudden death` (`cluster.py:223`) then a new `pushing tasks at <pid>` (`cluster.py:342`). The `sleep(10)` is why reincarnations are exactly 10 s apart.
- **The verbose `--- Logging error ---` dump.** Each pusher death also emits a Python stdlib logging dump, because django-q calls `logger.error(e, traceback.format_exc())` with a second positional argument (`django_q/cluster.py:347`), which the stdlib logger rejects with `TypeError: not all arguments converted during string formatting`. Producing command `sed -n '67,92p' qcluster.log` (one dump, Run A, RAW — the middle stdlib frames shown in full, the dump continues past line 92):

```
--- Logging error ---
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
  File "/usr/local/lib/python3.9/site-packages/django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 1900, in blpop
    return self.execute_command('BLPOP', *keys)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 901, in execute_command
    return self.parse_response(conn, command_name, **options)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 915, in parse_response
    response = connection.read_response()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 739, in read_response
    response = self._parser.read_response()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 470, in read_response
    self.read_from_socket()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 429, in read_from_socket
    raise ConnectionError(SERVER_CLOSED_CONNECTION_ERROR)
redis.exceptions.ConnectionError: Connection closed by server.

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/logging/__init__.py", line 1083, in emit
    msg = self.format(record)
  File "/usr/local/lib/python3.9/logging/__init__.py", line 927, in format
```

### 4.2 Ownership verification before each interrupt

Before stopping Redis I verified the target belonged to **this** container (not some other local/host Redis), then used a container-scoped, port-specific command. Producing command and output (Run A, Trial 1 BEFORE, RAW):

```bash
# docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 info server | grep -E "run_id|process_id|tcp_port"; \
#   ps -C redis-server -o pid,user,args --no-headers'
process_id:611
run_id:1af892c1e2b524c5aeded9853b9f14d9f3e29098
tcp_port:6379
    611 testuser redis-server 127.0.0.1:6379
```

The interrupt/restart commands (all scoped to `127.0.0.1:6379` inside the container; no host-wide destructive command was used):

```bash
docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 shutdown nosave'        # interrupt
docker exec -u testuser "$CN" bash -lc 'redis-server --daemonize yes --bind 127.0.0.1 --port 6379'   # restart
```

The two Redis-backed subsystems were then exercised with small helper scripts run through the canonical Django entry point `python3 manage.py shell` (so real project settings — `Q_CLUSTER`, `CHANNEL_LAYERS` — are loaded):
- **Broker round-trip:** `from django_q.tasks import async_task, fetch; tid = async_task("math.sqrt", 81)`, then poll `fetch(tid)` and print `result`. A returned `9.0` proves the scheduler->broker->worker->monitor(result-persist) path is live end-to-end.
- **Channels group-layer round-trip:** `get_channel_layer()` (a `channels_redis.core.RedisChannelLayer`), `new_channel()`, `group_add("qa_group", ch)`, `group_send("qa_group", {...})`, `receive(ch)`, `group_discard(...)`. A received payload equal to the sent payload (`MATCH True`) proves the `channels-redis` layer is talking to Redis again.

### 4.3 Trial 1 — complete before / during / restart / after

**BEFORE:** idle and stable; the pusher had been `Process-1:13 pushing tasks at 763` since cluster start (`18:27:04`). `redis-cli ... ping` -> `PONG`. `qcluster.log` marker = 66 lines; Redis `run_id 1af892c1...`, `process_id 611`.

**INTERRUPT at `19:00:34`.** `PING` during the outage confirms it (RAW):
```bash
# docker exec "$CN" bash -lc 'redis-cli -h 127.0.0.1 -p 6379 ping'
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

**DURING.** The error stream begins immediately (first `[Q] ERROR` at `19:00:35`, 1 s after interrupt). The recurring heartbeat error line and the first pusher death+reincarnation (producing command `sed -n '156,178p' qcluster.log`, showing the first two error lines and the first reincarnation), RAW:

```
19:00:35 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:00:35 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:00:44 [Q] INFO Process-1:13 stopped pushing tasks
19:00:45 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
19:00:45 [Q] INFO Process-1:21 pushing tasks at 2026
```

**Error-stream rate (exact arithmetic).** Producing command:
```bash
# grep -cE '^[0-9]{2}:[0-9]{2}:[0-9]{2} \[Q\] ERROR Error 111 connecting' <trial-1 slice of qcluster.log>
```
Count = **115**; first at `19:00:35`, last at `19:01:32`. Active error span `19:00:35` -> `19:01:32` = **57 s** => **115 / 57 s = 2.02 errors/s**, matching the 0.5 s guard `Stat` cycle.

**Reincarnations** at `19:00:45, :55, 19:01:05, :15, :25, :35` — exactly **10 s** apart; pusher lineage `13 -> 21 -> 22 -> 23 -> 24 -> 25 -> 26`.

**RESTART at `19:01:32`** (new `run_id 3ff508e7af535498203259b82dfdf2fdf1f7302f`, `process_id 2075`, `ping -> PONG`). Recovery block (producing command `sed -n '771,778p' qcluster.log`), RAW:

```
19:01:30 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:01:30 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:01:31 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:01:31 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:01:32 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:01:35 [Q] INFO Process-1:25 stopped pushing tasks
19:01:35 [Q] ERROR reincarnated pusher Process-1:25 after sudden death
19:01:35 [Q] INFO Process-1:26 pushing tasks at 2095
```

The last `[Q] ERROR ... Connection refused` is at `19:01:32` (= restart second); the reincarnation cycle at `19:01:35` produced pusher **`Process-1:26 pushing tasks at 2095`** — the first pusher to spawn after Redis returned — a **3 s** lag, i.e. within one 10 s reincarnation cycle. **Zero** `[Q] ERROR ... Connection refused` lines follow it (producing command + count):
```bash
# awk 'NR>778' qcluster.log | grep -cE '^[0-9:]+ \[Q\] ERROR Error 111 connecting'   -> 0
```

**Active recovery proof (Trial 1).** After the recovered pusher, both Redis-backed subsystems were exercised (§4.2). Broker round-trip (RAW):
```
# python3 manage.py shell < broker_rt.py    (QA_TAG=cycle1-...)
19:02:09 [Q] INFO Enqueued 1
TAG cycle1-20260710T190208
TASK_ID=4e10cb781fca4e29ab9fc9cd1d85d71f
FOUND True SUCCESS True RESULT 9.0 FUNC math.sqrt
```
Channels group-layer round-trip (RAW):
```
# python3 manage.py shell < chan_rt.py      (QA_TAG=cycle1-...)
SENT {'type': 'qa.message', 'text': 'cycle1-20260710T190209-19803'}
RECEIVED {'type': 'qa.message', 'text': 'cycle1-20260710T190209-19803'}
MATCH True
```
Web server + consumer liveness (RAW):
```
# curl -f -s -o /dev/null -w "HTTP:%{http_code}" http://localhost:8000 ; ps -C ... document_consumer
HTTP:302 exit=0
consumer_alive:     688 python3 manage.py document_consumer
```
**Error-free window (Trial 1):** from the recovered pusher `19:01:35` to a wallclock re-check at `19:03:31` (= **116 s**), `grep -c '[Q] ERROR Error 111 connecting'` over that span = **0** and `ping -> PONG`; an explicit intra-window liveness endpoint is the broker round-trip completing at `19:02:09` (`RESULT 9.0`).

### 4.4 Trial 2 — reproduction (unchanged procedure) + a post-recovery scheduled task as an independent endpoint

**BEFORE:** stable pusher `Process-1:26 pushing tasks at 2095` (from Trial 1's recovery, stable ~3 min); ownership re-verified (`run_id 3ff508e7...`, `process_id 2075`); `ping -> PONG`; `qcluster.log` marker = 783 lines.

**INTERRUPT at `19:04:43`; RESTART at `19:05:39`** (new `run_id 5fdf5e4cf18d216a60413722000b011031f341b2`, `process_id 2362`, `ping -> PONG`). Outage = **56 s**.

**Error-stream rate.** Count = **110**; first `19:04:44`, last `19:05:39`; active span `19:04:44` -> `19:05:39` = 55 s => `110 / 55 s = 2.00 errors/s`. **Reincarnations** at `19:04:53, 19:05:03, :13, :23, :34, :44` — ~10 s apart.

**RECOVERY + post-recovery scheduled task.** Producing command `sed -n '1487,1503p' qcluster.log`, RAW (recovery block followed by the next naturally-scheduled mail firing):

```
19:05:37 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:05:38 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:05:38 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:05:39 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:05:44 [Q] INFO Process-1:32 stopped pushing tasks
19:05:44 [Q] ERROR reincarnated pusher Process-1:32 after sudden death
19:05:44 [Q] INFO Process-1:33 pushing tasks at 2372
19:07:09 [Q] INFO Enqueued 1
19:07:09 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:07:09 [Q] INFO Process-1:10 processing [oklahoma-hotel-vermont-network]
19:07:09 [Q] INFO Process-1:10 stopped doing work
19:07:09 [Q] INFO Processed [oklahoma-hotel-vermont-network]
```

Last `[Q] ERROR ... Connection refused` at `19:05:39`; recovered pusher **`Process-1:33 pushing tasks at 2372`** at `19:05:44` (a **5 s** lag, within one 10 s cycle); **zero** `[Q] ERROR ... Connection refused` after it.

**Active recovery proof (Trial 2).** Broker round-trip (RAW):
```
# python3 manage.py shell < broker_rt.py    (QA_TAG=cycle2-...)
19:06:14 [Q] INFO Enqueued 1
TAG cycle2-20260710T190613
TASK_ID=c7b856a0a8664a63b99fd4e1527cced8
FOUND True SUCCESS True RESULT 9.0 FUNC math.sqrt
```
Channels group-layer round-trip (RAW):
```
# python3 manage.py shell < chan_rt.py      (QA_TAG=cycle2-...)
SENT {'type': 'qa.message', 'text': 'cycle2-20260710T190614-29428'}
RECEIVED {'type': 'qa.message', 'text': 'cycle2-20260710T190614-29428'}
MATCH True
```
Web server + consumer liveness (RAW): `HTTP:302 exit=0`; consumer still alive (`688 python3 manage.py document_consumer`).

**Error-free window (Trial 2) — two explicit log endpoints.** The window runs from the recovered pusher `19:05:44` to the **next naturally-scheduled** mail firing at `19:07:09` (`created a task from schedule [Check all e-mail accounts]` -> `processing` -> `Processed`) = **85 s**, with `grep -c '[Q] ERROR Error 111 connecting'` over that span = **0** and `ping -> PONG`. Both endpoints are concrete log lines, and the closing endpoint is a **scheduler-driven** event (not one I triggered), independently proving the full `scheduler -> pusher -> worker -> monitor` path is operational again — not merely that the error stream stopped.

### 4.5 Summary and scope of the reconnection signal

| Aspect | Trial 1 | Trial 2 |
|---|---|---|
| Redis stopped (interrupt) | 19:00:34 | 19:04:43 |
| Redis restarted (`PONG`) | 19:01:32 | 19:05:39 |
| Outage duration | 58 s | 56 s |
| `[Q] ERROR Error 111 ... Connection refused` count | 115 | 110 |
| Error rate (count / active span) | 115/57 s = **2.02/s** | 110/55 s = **2.00/s** |
| Last `[Q] ERROR` timestamp | 19:01:32 (= restart) | 19:05:39 (= restart) |
| Reincarnation interval | ~10 s (…19:00:45/:55/01:05/:15/:25/:35) | ~10 s (…19:04:53/05:03/:13/:23/:34/:44) |
| Recovered pusher | `Process-1:26 pushing tasks at 2095` @19:01:35 (3 s after restart) | `Process-1:33 pushing tasks at 2372` @19:05:44 (5 s after restart) |
| `[Q] ERROR` after recovered pusher | 0 | 0 |
| django-q broker round-trip (`math.sqrt(81)`) | `RESULT 9.0` @19:02:09 | `RESULT 9.0` @19:06:14 |
| `channels-redis` group-layer round-trip | `MATCH True` | `MATCH True` |
| Web server / consumer | `HTTP 302` / alive (PID 688) | `HTTP 302` / alive (PID 688) |
| Measured error-free window after recovery | 116 s (19:01:35 -> 19:03:31; endpoint: broker RT @19:02:09) | 85 s (19:05:44 -> post-recovery mail firing @19:07:09) |

**Conclusion (R4).** There is **no explicit "reconnected" message** at this commit. "Operational again" is instead confirmed by a **combination** of signals, verified after **both** restarts: (1) the `[Q] ERROR ... Connection refused` stream **ceases** (last error at the restart second ±1 s); (2) a fresh **`[Q] INFO Process-1:N pushing tasks at <pid>`** line appears within one 10 s reincarnation cycle and has no errors after it; (3) a **django-q broker** task round-trips to `9.0`; (4) the **`channels-redis` group layer** round-trips (`MATCH True`); (5) the web server answers `HTTP 302`; and (6) the `document_consumer` process is still alive. Trial 2 additionally shows a **naturally-scheduled** mail task firing after recovery. Because the test exercised **both** Redis-backed subsystems (the task-queue broker **and** the Channels group layer) plus the web and consumer processes, the recovery evidence spans the whole idle system, not just one path. (What is still **[INFERRED]**, not observed: an *authenticated end-to-end websocket client* reconnect through the browser — the group-layer round-trip proves the Channels transport over Redis works, which is the piece that depends on Redis; the `StatusConsumer` accept/deny logic itself is unchanged code, §5.2.)

---

## 5. R5 — components/processes that run continuously to maintain a ready state

At idle, the following are **always-on**, distinguished from **transient** work that runs only briefly when triggered. PIDs are from Run A.

### 5.1 Always-on (continuously running at idle) — [OBSERVED]

1. **Redis server** (`redis-server 127.0.0.1:6379`, PID 611) — the shared backbone: django-q broker (`src/paperless/settings.py:456`) **and** Channels group-messaging layer (`settings.py:178-187`). Stopping it degrades the cluster and both Redis-backed subsystems (§4).
2. **The django-q cluster (`qcluster`)** — long-lived processes under the management command (PID 689) and its guard/sentinel (PID 750):
   - **Guard / sentinel** — a 0.5 s health loop that reincarnates dead workers/monitor/pusher and invokes the scheduler ~every 30 s (`django_q/cluster.py:253-289`). Observed: `Process-1 guarding cluster undress-illinois-kitten-item`.
   - **Monitor** (PID 762) — persists task results (`django_q/cluster.py:378`). Observed: `Process-1:12 monitoring at 762`.
   - **Pusher** (PID 763) — continuously `BLPOP`-polls the broker (`django_q/cluster.py:342`, `django_q/brokers/redis_broker.py:20-21`). Observed: `Process-1:13 pushing tasks at 763`.
   - **Worker pool** — 11 idle workers waiting for work (`Process-1:1..11 ready for work`), sized `floor(sqrt(cores))` (`src/paperless/settings.py:427-433`; §1.7).
3. **gunicorn master + 2 uvicorn ASGI workers** (PIDs 687, 723, 724) — listening on `:8000` for HTTP **and** the `ws/status/$` websocket (`gunicorn.conf.py:3-5`, `src/paperless/workers.py:9`, `src/paperless/asgi.py:17,19-20`, `src/paperless/urls.py:136-137`). They stay up to answer the API/UI and to accept websocket status connections.
4. **`document_consumer`** (PID 688) — an inotify watcher on the consumption directory, blocking on filesystem events (`src/documents/management/commands/document_consumer.py:200`). It runs continuously so a dropped-in document is picked up immediately.

### 5.2 Transient (NOT continuously running) — [OBSERVED] / [INFERRED]

- **The four scheduled task executions** (mail / classifier / index / sanity) — each briefly occupies a django-q worker when the scheduler fires it, then finishes (§3.2, §3.3). Periodic events, not standalone processes.
- **`docker/wait-for-redis.py` — a startup-only gate, NOT a continuous process.** In the product deployment the entrypoint runs this once before the main processes start: it attempts `client.ping()` in a retry loop of `MAX_RETRY_COUNT = 5` (`docker/wait-for-redis.py:16`) with `RETRY_SLEEP_SECONDS = 5`-spaced sleeps (constant at `wait-for-redis.py:17`; `Redis.from_url(...)` + `while`/`ping()`/`time.sleep(...)` loop at `wait-for-redis.py:24-35`), then **exits** `EX_OK` on success or `EX_UNAVAILABLE` on failure (`wait-for-redis.py:37-42`). It is a transient readiness gate, not part of the always-on set. **[SOURCE-DERIVED — the product entrypoint was not executed in this manual run (§1.1).]**
- **Authenticated websocket `StatusConsumer` instances — per-client, connection-triggered, NOT always-on.** The web server is always listening for `ws/status/$` (§5.1.3), but a `StatusConsumer` object exists only for the lifetime of an authenticated client connection: `connect()` denies unauthenticated clients (`raise DenyConnection`, `src/paperless/consumers.py:13-15`) and otherwise joins the `status_updates` group (`consumers.py:9,17-20`), leaving it on disconnect via `group_discard` (`consumers.py:24-27`). No browser client connected during this idle run, so none existed. (The **Channels group layer** those consumers rely on **was** exercised directly over Redis in §4.) **[INFERRED — no websocket client was connected at idle.]**
- **Document ingestion work** — `consume_file` (`src/documents/tasks.py:184`) runs only when a document is added; it did **not** run during the idle window (empty consume dir; `document_consumer.py:85` "Adding … to the task queue." never logged). **[INFERRED — no document was ingested.]**
- **Worker recycling** — after each task a worker is recycled and replaced (`"recycle": 1`, `src/paperless/settings.py:452`); observed as `recycled worker Process-1:N` (§3.2). Triggered by task completion, not a standalone loop.

---

## 6. Cleanup (temporary artifacts removed; repository unchanged)

Per the read-only mandate, the only repository change is **this document**. Every temporary runtime artifact was removed and the removal captured. The graceful-stop transcript below is from the independent run's teardown (Run B, `kill -TERM` of its three PIDs 708/709/710); Run A was torn down identically. Producing commands and results (RAW excerpts):

```bash
# 1. Graceful stop of the 3 captured PIDs (exact PIDs; NO pkill/killall)
# docker exec $CN kill -TERM 708 709 710
# -> qcluster.log tail:
19:16:30 [Q] INFO Process-1 waiting for the monitor.
19:16:30 [Q] INFO Process-1:12 stopped monitoring results
19:16:30 [Q] INFO Q Cluster minnesota-tennessee-nineteen-seventeen has stopped.
# -> gunicorn.log tail:
[2026-07-10 19:16:28 +0000] [708] [INFO] Handling signal: term
[2026-07-10 19:16:28 +0000] [708] [INFO] Shutting down: Master

# 2. Stop Redis in each container, container-scoped, after re-verifying ownership by run_id
# docker exec $CN redis-cli -h 127.0.0.1 -p 6379 shutdown nosave
# docker exec $CN redis-cli -h 127.0.0.1 -p 6379 ping   -> Could not connect ... Connection refused

# 3. Remove BOTH observation containers by exact name (NO prune, NO broad rm)
# docker stop pngx_obsA_20260710_182631 pngx_obsB_20260710_182219
# docker rm   pngx_obsA_20260710_182631 pngx_obsB_20260710_182219
# docker ps -a --filter name=pngx_obs   -> (empty; both containers removed)
```

The three processes stopped gracefully (`Q Cluster ... has stopped.`; gunicorn `Shutting down: Master`); Redis was stopped only after confirming the `run_id` belonged to that container; each container was removed by exact name, destroying all in-container temporary state (data/media/consume/index/log directories and the SQLite DB). The complete relevant raw transcripts are **embedded in this document** (§§1.6, 2–4); the full log files were also copied **outside** the repository tree to the investigation host and their SHA-256 digests recorded in §0.1 as supplementary provenance, after which those host-side files — like all other temporary observation artifacts (the two containers, scratch logs, and helper captures under `/tmp`) — were removed. Repository state after cleanup — producing command and output (RAW):

```bash
# git -C /tmp/blitzy/paperless-ngx/blitzy-4fecd6d9-f2f1-4a67-ac76-b5521a549dfb_3abcd3 status --porcelain
# (no output)
```

An empty `git status --porcelain` means the working tree has **no staged, unstaged, or untracked changes relative to HEAD** — i.e. it is clean. This document is committed as tracked content on the branch (it is part of `HEAD`), which is precisely why it does **not** appear as a pending change in `--porcelain` output.

---

## 7. Coverage pass

Every named item in the five questions, mapped to observed evidence and citations.

- **R1 — up at the pinned commit, stable idle.** Ran at commit `542221a38dff` (verified in-container `git rev-parse HEAD`, worktree clean — §1.3), Python 3.9.23; reached idle at `18:27:03Z` (Run A) with an **empty consume directory** and the four schedules present (§2.1). Exact host/container build+run commands in §1.2; `manage.py migrate/reindex/check` transcripts in §1.6 (`check` -> "System check identified no issues (0 silenced).").
- **R2 — background processes/tasks running automatically at idle.** Enumerated with observed startup banners and producing commands (§2.2) and the full process tree (§2.3): gunicorn master + 2 uvicorn ASGI workers (687/723/724); `document_consumer` inotify watcher (688); the django-q cluster = mgmt (689) -> guard/sentinel (750) -> 11 workers (751-761) + monitor (762) + pusher (763); all over Redis (611). Plus the periodic scheduled tasks (§3).
- **R3 — periodic health/readiness entries: specific messages, frequency, meaning.** Cadence table (§3.1). **Specific messages** quoted verbatim with `file:line`. Frequencies: mail `[Check all e-mail accounts]` every **10 min** — **measured across two independent runs** (Run A firings `18:37:05`/`18:47:06`/`18:57:08`; Run B firings `18:33:13`/`18:43:14`/`18:53:16`; two clean intervals of 601 s / 602 s in *each* run; DB `next_run` stepped +600 s) (§3.3); classifier **hourly**, index **daily**, sanity **weekly** — **[CONFIGURED / SOURCE-DERIVED]** (fired once in the startup burst; `next_run` +1 h / +1 day / +7 days; did not recur in-window) (§3.1, §3.3), sanity emitting `Sanity checker detected no issues.`. Meaning of each explained narrowly (scheduler-enqueue + worker-completion) in §3.1.
  - **KEY FINDING:** no dedicated periodic "healthy" heartbeat INFO line — proven by the flat quiet-window snapshots and the literal `wc -l` / `tail -1` output (§3.4(i)).
  - **Both named silent mechanisms addressed:** the configured 30-second Compose healthcheck `curl :8000` — manual reproduction with the literal `curl` binary, HTTP 302, **zero** new log lines (§3.4(ii)); the 0.5-second django-q `Stat` heartbeat — silent while healthy, loud only during the §4 outage (§3.4(iii)).
- **R4 — messages confirming reconnection/operational-again after interrupt+restart.** Redis interrupted+restarted **twice** with complete before/during/restart/after transcripts and ownership verification (§4.2-§4.4). During: `[Q] ERROR Error 111 ... Connection refused` at **2.02/s** (Trial 1) and **2.00/s** (Trial 2) + pusher `reincarnated pusher … after sudden death` every ~10 s. After: the error stream **ceases** (last error at the restart second ±1 s) and a fresh `[Q] INFO Process-1:N pushing tasks at <pid>` appears within one 10 s cycle with no later errors; **both** Redis-backed subsystems were actively re-exercised after **each** restart — a django-q broker round-trip (`RESULT 9.0`) and a `channels-redis` group-layer round-trip (`MATCH True`) — plus `HTTP 302` and a live consumer; Trial 2 also shows a naturally-scheduled mail firing after recovery (§4.4).
  - **KEY FINDING:** **no dedicated "reconnected" message**; operational-again = (error stream stops) + (new `pushing tasks at <pid>`) + (broker round-trip `9.0`) + (Channels round-trip `MATCH True`) + (`HTTP 302`) + (live consumer). Reproduced in both trials (§4.5).
- **R5 — components/processes that keep running continuously at idle.** Always-on vs. transient split (§5): always-on = Redis; django-q guard/monitor/pusher/worker-pool; gunicorn master + 2 uvicorn workers; `document_consumer`. Transient = the four scheduled task executions, the startup-only `wait-for-redis.py` gate, per-client `StatusConsumer` instances, and (not at idle) `consume_file` / worker recycling.

### 7.1 Labels used in this document

- **[OBSERVED]** — seen in a run's captured output (all quoted log blocks; the mail cadence in two runs; the silent windows; the two Redis trials with broker + Channels round-trips).
- **[CONFIGURED / SOURCE-DERIVED]** — read from config/DB/source, full period not observed end-to-end in-window: the hourly/classifier, daily/index, weekly/sanity cadences (fired once in the startup burst; confirmed via `next_run` and migrations); the configured 30 s healthcheck interval (Compose not executed); the `wait-for-redis.py` startup gate (product entrypoint not executed).
- **[INFERRED]** — from reading code, not exercised: the `Polling directory for changes` fallback (inotify was actually used); `consume_file` ingestion and `Adding … to the task queue.` (no document ingested); an authenticated end-to-end browser websocket `StatusConsumer` reconnect (no browser client connected; the underlying Channels group layer over Redis **was** exercised, §4).
- **[NON-CANONICAL]** — environment-specific, each with its canonical counterpart (§1.1, §1.7): reproduction image vs. published product image (user/paths/entrypoint/supervisor); Redis co-located here vs. separate `redis:6.0` service; **11 workers** = `floor(sqrt(128))` (canonical: depends on host CPU count / `PAPERLESS_TASK_WORKERS`); direct launch vs. supervisord/compose; volatile timestamps/PIDs/word-names. The observed outage error `Error 111 ... Connection refused` (Errno 111, `ECONNREFUSED`) **is** the canonical message for a downed local Redis (exception class `redis.exceptions.ConnectionError`).

### 7.2 Reproduction summary

- **Two independent runs**, both at commit `542221a38dff`, image digest `sha256:6e699f225ced...`, Python 3.9.23, all timestamps UTC:
  - **Run A** (`pngx_obsA_20260710_182631`) — full observation: topology, banners, startup burst, idle-silence, healthcheck, and the two Redis trials. Launch `18:27:03Z`; idle observation to ~`18:59Z`; two Redis trials ~`19:00Z`-`19:07Z`.
  - **Run B** (`pngx_obsB_20260710_182219`) — independent cadence confirmation. Launch `18:23:11Z`; four mail firings to `18:53:16`; graceful stop `19:16Z`.
- Mail cadence: confirmed **across two independent runs** — two clean on-schedule intervals per run (Run A 601 s / 602 s; Run B 601 s / 602 s); all four cadences cross-checked against DB `next_run` in Run A.
- Redis interrupt/restart: **2 trials** (both Run A); reincarnation interval ~10 s in both; error rate 2.02/s and 2.00/s; recovery pattern identical; after each restart a django-q broker round-trip (`9.0`) **and** a `channels-redis` group-layer round-trip (`MATCH True`) confirmed whole-system recovery; Trial 2 additionally confirmed a post-recovery scheduled-task firing.
- Complete relevant raw transcripts embedded inline (§§1.6, 2–4); full log files SHA-256-hashed as supplementary provenance (§0.1); all temporary artifacts removed; repository worktree clean (§6).
