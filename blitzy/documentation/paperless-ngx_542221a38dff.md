# Paperless-ngx — Idle Runtime Behavior (Observed Baseline)

**Pinned commit:** `542221a38dff06361e07976452f9aea24d210542`
**Nature of this document:** a *pre-change behavioral baseline*. It answers, from **actual observed runtime output**, how Paperless-ngx behaves once fully started and sitting **idle** (zero documents being processed), *before* any code changes are made.

**How the evidence was produced.** The full stack was built and run at the pinned commit inside the provided Docker image `paperless-ngx-ready:542221a38dff` (derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`), which pins Python 3.9 — `Dockerfile:L18` `FROM python:3.9-slim-bullseye as main-app`. A separate `redis:6` container provided the broker + channel layer. Every behavioral claim below is paired with **one verbatim observed line** and the **command that produced it**, and every literal (string, config key, path, cadence, number) carries a `file:line` citation. Each quoted log line is labelled with its **sink**: `stdout`, `data/log/paperless.log`, or `data/log/mail.log`.

> **Read-only / clean-tree statement.** No existing source file was modified. All observation was performed in a throwaway container and with temporary capture files kept **outside** the repository tree. The only change introduced to the repository is this one document, committed on the working branch; every temporary artifact (the throwaway observation container and the host-side capture files) was created outside the repository tree and removed afterward. This is verified with the commands captured in §6: `git diff --name-status 542221a38dff..HEAD` shows exactly one entry — `A blitzy/documentation/paperless-ngx_542221a38dff.md` — and, with the deliverable committed, `git status --porcelain` is empty (a clean working tree).

> **Three citation corrections** (the plan's prose drifted; the runtime-verified lines below are authoritative, re-verified with `grep -n`): the Redis reconnection string is at `docker/wait-for-redis.py:L41` (not L44); `PAPERLESS_WORKER_TIMEOUT` default `1800` is at `src/paperless/settings.py:L440` (not L444); the root console handler is `src/paperless/settings.py:L407` with the `loggers` block at `L408`–`L410` (not L406–L411).

> **Citation convention.** Two kinds of `file:line` citations appear in this document. **(1) Repository files** are cited by their repo-relative path (for example `src/paperless/settings.py:L407`, `docker/wait-for-redis.py:L41`, `docker/docker-entrypoint.sh:L77`); these resolve in the source tree at the pinned commit `542221a38dff`. **(2) Installed dependency-package internals** are cited with a `django_q/` prefix (for example `django_q/cluster.py:L410`, `django_q/conf.py:L207`, `django_q/humanhash.py:L364`) — these are **not** repository files but source lines of the pinned dependency **`django-q==1.3.9`** (`requirements.txt:L37`; verified at runtime `import django_q; django_q.VERSION == (1, 3, 9)`), installed in this image at the runtime path `/usr/local/lib/python3.9/site-packages/django_q/`. django-q emits every `[Q]` log line quoted below, so its cluster/scheduler internals are cited to explain those lines; the official django-q 1.3 documentation is used only for interpretation (noted where it appears).

---

## Section 1 — O1: Setup and reaching a stable idle state

**Provenance — exact commit *and* branch of the running code.** The stack was run from a checkout on branch **`paperless-ngx_542221a38dff`** at the pinned commit **`542221a38dff06361e07976452f9aea24d210542`**, with a **clean working tree** (no tracked file modified). Command + verbatim output, captured inside the running container at `/app` (sink `stdout`):

```text
$ git rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ git rev-parse --abbrev-ref HEAD
paperless-ngx_542221a38dff
$ git status --porcelain
(no output — clean working tree)
```

`git rev-parse --abbrev-ref HEAD` returning `paperless-ngx_542221a38dff` is also what names this document (`blitzy/documentation/paperless-ngx_542221a38dff.md`); every log line, cadence, and literal quoted below was observed against exactly this commit on this branch.

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
- `migrations()` prints `docker/docker-prepare.sh:L44` `echo "Apply database migrations..."` then runs `python3 manage.py migrate` (`docker/docker-prepare.sh:L45`).

**Observed — the entrypoint banner (sink `stdout`).** Running the real entrypoint in the observation container prints the banner verbatim, then the stock `initialize()` step stops under `set -e` because this prepared image runs as `testuser` and has no `paperless` OS user (it bakes the repo at `/app` and runs as `testuser` instead of performing the stock root→`paperless` remap). Reported exactly as observed:

```text
$ docker exec pl-idle-obs bash -lc 'cd /app/src && bash /app/docker/docker-entrypoint.sh /bin/true'
Paperless-ngx docker container starting...
id: ‘paperless’: no such user
```

The banner line (`docker/docker-entrypoint.sh:L77`) is emitted before the stop; the missing-`paperless`-user abort is an artifact of this prepared image and does not affect the idle observation, which launches the three programs directly (below).

**Observed — the prepare/migration path (sink `stdout`).** Running the real prepare script reproduces the `wait_for_redis` → `migrations` sequence verbatim. (To run the stock script unchanged, two throwaway-container symlinks were created — `/usr/src/paperless → /app` and `/sbin/wait-for-redis.py → /app/docker/wait-for-redis.py` — because the image places the repo at `/app`; these are container-filesystem links, not repository changes.)

```text
$ docker exec -u testuser -w /app/src -e PAPERLESS_REDIS=redis://paperless-redis:6379 \
    pl-idle-obs bash -lc 'bash /app/docker/docker-prepare.sh'
Waiting for Redis: redis://paperless-redis:6379
Connected to Redis broker: redis://paperless-redis:6379
Apply database migrations...
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
Search index out of date. Updating...
```

This surfaces both required startup banners as **observed output**: the Redis-readiness line `Connected to Redis broker: …` (`docker/wait-for-redis.py:L41`) and the migration banner `Apply database migrations...` (`docker/docker-prepare.sh:L44`). `No migrations to apply.` confirms the baked `db.sqlite3` already carries every migration — so the four django-q `Schedule` rows already exist (proven in §1.3). A direct `python3 manage.py migrate` reproduces the same no-op:

```text
$ docker exec -u testuser -w /app/src -e PAPERLESS_REDIS=redis://paperless-redis:6379 \
    pl-idle-obs bash -lc 'python3 manage.py migrate'
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
```

`python3 manage.py migrate` is what **materializes the four django-q `Schedule` rows** (Section 3). As the observed `No migrations to apply.` above confirms, the image already carries a fully-migrated `db.sqlite3`, so migration was a no-op and the four schedules were already present (verified in §1.3). The three programs were then launched directly (the image places the repository at `/app`, not `/usr/src/paperless`, so the stock command in `docker/supervisord.conf:L11` — `gunicorn -c /usr/src/paperless/gunicorn.conf.py …` — was run against the real path `-c /app/gunicorn.conf.py`; the process identities are otherwise identical to the supervisord programs).

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

**Idle** here means: all three programs RUNNING **and** migrations applied **and** the consumption directory empty **and** the task queue drained. (Note: the image bakes two leftover sample PDFs into `/app/consume` — `patch-code-t-middle_document_0.pdf` and `patch-code-t-middle_document_1.pdf` (8156 bytes each) — which were removed before starting the consumer so the system is genuinely document-free. Removing runtime data inside a throwaway container is not a source-file change.)

**(a) All three programs RUNNING** — command + verbatim (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ps axf -o pid,ppid,user,args | grep -E "qcluster|document_consumer|gunicorn" | grep -v grep'
    115       0 testuser bash -lc gunicorn -c /app/gunicorn.conf.py paperless.asgi:application > /tmp/obs/gunicorn.log 2>&1
    122     115 testuser  \_ /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    126     122 testuser      \_ /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    130     122 testuser      \_ /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
    107       0 testuser bash -lc python3 -u manage.py document_consumer > /tmp/obs/consumer.log 2>&1
    114     107 testuser  \_ python3 -u manage.py document_consumer
     99       0 testuser bash -lc python3 -u manage.py qcluster > /tmp/obs/qcluster.log 2>&1
    106      99 testuser  \_ python3 -u manage.py qcluster
    142     106 testuser      \_ python3 -u manage.py qcluster
    144     142 testuser          \_ python3 -u manage.py qcluster
    148     142 testuser          \_ python3 -u manage.py qcluster
    ...            (13 children of PID 142 in total: 11 workers + monitor + pusher; full qcluster tree in §3.1)
```

**(b) Consumption directory empty** (sustained; `src/paperless/settings.py:L78` `CONSUMPTION_DIR`) — command + verbatim (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ls -la /app/consume'
total 12
drwxr-sr-x 1 testuser testuser 4096 Jul  1 23:53 .
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

> **On the PID values in this document.** The absolute PIDs shown here — and in §1.4a, §3.1, and §5 — come from **separate `ps`/log snapshots** captured across the investigation's multiple idle bring-ups (and, in §5, across program restarts). They are **not stable identifiers** and legitimately differ between snapshots (for example the qcluster master reads `95` in this snapshot versus `106` in the §1.4a process tree, and the sentinel `133` here versus `142` in §3.1). What is invariant across every run — and what the answers below actually rely on — is the **process structure** (one gunicorn master + 2 workers; one consumer; one qcluster master → one sentinel/guard → 11 workers + 1 monitor + 1 pusher = 13 children) and the **verbatim log strings**; both were reproduced identically on re-runs, while only the PIDs (and the random cluster display name, §3.1) changed.

### 2.2 The two external always-on services

- **Redis** — the single most important dependency: it is **both** the django-q broker (`src/paperless/settings.py:L456`) **and** the Channels layer backend `channels_redis.core.RedisChannelLayer` (`src/paperless/settings.py:L178` `CHANNEL_LAYERS`, `L180` backend, `L182` hosts, `L183` `"capacity": 2000`, `L184` `"expiry": 15`). Its liveness was shown by `PING -> True` in §1.1; Section 5 shows what happens when it is interrupted.
- **Database** (SQLite here) — persists document metadata **and** the four `Schedule` rows the scheduler reads (dumped in §1.3).

The ASGI application itself routes both protocols even while idle — `src/paperless/asgi.py:L17` `application = ProtocolTypeRouter(`, `L19` `"http": get_asgi_application()`, `L20` `"websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` — but the websocket branch is dormant with no client connected (see §3.4).

---

## Section 3 — O2: Automatic background processes/tasks while idle

### 3.1 The django-q `qcluster` process tree (what keeps executing)

The `qcluster` program is not one process; the **sentinel/guard** forks a pool of workers plus a monitor and a pusher. **Command + verbatim forest** (sink `stdout`):

```text
$ docker exec pl-idle-obs bash -lc 'ps axf -o pid,ppid,user,args | grep "manage.py qcluster" | grep -v grep | head'
     99       0 testuser bash -lc python3 -u manage.py qcluster > /tmp/obs/qcluster.log 2>&1
    106      99 testuser  \_ python3 -u manage.py qcluster
    142     106 testuser      \_ python3 -u manage.py qcluster
    144     142 testuser          \_ python3 -u manage.py qcluster
    148     142 testuser          \_ python3 -u manage.py qcluster
    151     142 testuser          \_ python3 -u manage.py qcluster
    152     142 testuser          \_ python3 -u manage.py qcluster
    153     142 testuser          \_ python3 -u manage.py qcluster
    ...            (13 children of PID 142 in total)
```

These roles are confirmed by the django-q startup lines the cluster emitted (sink `stdout`, `/tmp/obs/qcluster.log`). All `[Q]` strings are library-emitted by **django-q 1.3.9** (`requirements.txt:L37` `django-q==1.3.9`; verified at runtime: `import django_q; django_q.VERSION == (1, 3, 9)`):

```text
23:53:56 [Q] INFO Q Cluster apart-maine-zebra-charlie starting.
23:53:56 [Q] INFO Process-1:1 ready for work at 144
23:53:56 [Q] INFO Process-1:2 ready for work at 148
23:53:56 [Q] INFO Process-1:3 ready for work at 151
23:53:56 [Q] INFO Process-1:4 ready for work at 152
23:53:56 [Q] INFO Process-1:5 ready for work at 153
23:53:56 [Q] INFO Process-1:6 ready for work at 154
23:53:56 [Q] INFO Process-1:7 ready for work at 155
23:53:56 [Q] INFO Process-1:8 ready for work at 156
23:53:56 [Q] INFO Process-1:9 ready for work at 157
23:53:56 [Q] INFO Process-1:10 ready for work at 158
23:53:56 [Q] INFO Process-1:11 ready for work at 159
23:53:56 [Q] INFO Process-1:12 monitoring at 160
23:53:56 [Q] INFO Process-1 guarding cluster apart-maine-zebra-charlie
23:53:56 [Q] INFO Process-1:13 pushing tasks at 161
23:53:56 [Q] INFO Q Cluster apart-maine-zebra-charlie running.
```

Reading the tree against those lines:

- **11 worker processes** — `Process-1:1 … Process-1:11 ready for work` at observed PIDs `144, 148, 151, 152, 153, 154, 155, 156, 157, 158, 159`. This equals `TASK_WORKERS`, verified at runtime `TASK_WORKERS = 11` (`src/paperless/settings.py:L438` = `TASK_WORKERS`; `L455` `"workers": TASK_WORKERS`). django-q emits this line from `django_q/cluster.py:L410` template `"{name} ready for work at {pid}"`.
- **1 monitor** — `Process-1:12 monitoring at 160` (`django_q/cluster.py:L378`). It writes task results back to the DB/broker.
- **1 pusher** — `Process-1:13 pushing tasks at 161` (`django_q/cluster.py:L342`). It pulls queued tasks off Redis into the worker pool.
- **1 sentinel / guard** — `Process-1 guarding cluster …` (`django_q/cluster.py:L256`); this is PID `142`, the parent of the worker pool (forked from the `manage.py qcluster` master, PID `106`). The **scheduler runs inside this guard loop** — there is no separate scheduler process (`django_q/cluster.py:L253` `def guard`).
- **Cluster ready** — `Q Cluster … running.` (`django_q/cluster.py:L261`) is the completion marker.

> **Observed nuance — cluster display name is NOT `paperless`.** The startup/running lines show `apart-maine-zebra-charlie`, even though `Q_CLUSTER["name"]="paperless"` (`src/paperless/settings.py:L450`). This is not a misconfiguration: in django-q 1.3.9 the *displayed* name is `Cluster.name` → `humanize(self.cluster_id.hex)` (`django_q/cluster.py:L110`–`L111`; `django_q/humanhash.py:L364`), a **random human-readable id generated per cluster run**; the startup line uses it (`django_q/cluster.py:L79` `f"Q Cluster {self.name} starting."`) as does the running line (`django_q/cluster.py:L261`). `Q_CLUSTER["name"]="paperless"` is the *internal* cluster identifier (Redis key / stat identity), not the log display name. A second cluster start later in this run produced a *different* random name (`pip-lactose-sink-jupiter`, §5.3), confirming the behavior. Reported exactly as observed.

### 3.2 Why `[Q]` lines look different from Paperless's own log lines

django-q installs its **own** logger and formatter, which is why `[Q]` lines carry `HH:MM:SS` and a `[Q]` tag instead of Paperless's verbose format:

- `django_q/conf.py:L207` `logger = logging.getLogger("django-q")`
- `django_q/conf.py:L213`–`L214` `Formatter(fmt="%(asctime)s [Q] %(levelname)s %(message)s", datefmt="%H:%M:%S")`
- `django_q/conf.py:L216`–`L217` its own `StreamHandler`.

Paperless's own loggers instead use the verbose format `"[{asctime}] [{levelname}] [{name}] {message}"` (`src/paperless/settings.py:L378`).

### 3.3 The four periodic schedules (each named, with cadence + migration `file:line`)

All four are created by data migrations and were dumped live in §1.3. **All four were also observed executing** in the initial catch-up burst (their pre-baked `next_run` values were overdue and `catch_up=False`, so each fired exactly once shortly after cluster start) — command + verbatim (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
23:54:25 [Q] INFO Enqueued 1
23:54:25 [Q] INFO Process-1 created a task from schedule [Train the classifier]
23:54:25 [Q] INFO Process-1:1 processing [mike-sixteen-xray-steak]
23:54:25 [Q] INFO Enqueued 1
23:54:25 [Q] INFO Process-1 created a task from schedule [Optimize the index]
23:54:25 [Q] INFO Process-1:2 processing [oranges-mexico-hamper-alaska]
23:54:25 [Q] INFO Enqueued 1
23:54:25 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
23:54:25 [Q] INFO Process-1:3 processing [equal-arizona-edward-october]
23:54:25 [Q] INFO Enqueued 1
23:54:25 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
23:54:25 [Q] INFO Process-1:4 processing [pizza-queen-mars-neptune]
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
    print(str(t.started)[:19],'|',t.func,'| success='+str(t.success),'| result='+repr(t.result))"
2026-07-01 23:54:25 | documents.tasks.train_classifier           | success=True | result=None
2026-07-01 23:54:25 | documents.tasks.index_optimize             | success=True | result=None
2026-07-01 23:54:25 | documents.tasks.sanity_check               | success=True | result='No issues detected.'
2026-07-01 23:54:25 | paperless_mail.tasks.process_mail_accounts | success=True | result='No new documents were added.'
2026-07-01 23:57:26 | paperless_mail.tasks.process_mail_accounts | success=True | result='No new documents were added.'
2026-07-02 00:07:27 | paperless_mail.tasks.process_mail_accounts | success=True | result='No new documents were added.'
2026-07-02 00:17:29 | paperless_mail.tasks.process_mail_accounts | success=True | result='No new documents were added.'
```

The four rows dated `23:54:25` are the one-time catch-up burst (all four schedules); the three later `process_mail_accounts` rows (`23:57:26`, `00:07:27`, `00:17:29`) are the recurring 10-minute mail checks measured in §4. Every idle result is a success no-op.

- `train_classifier` returns `None` and logs nothing on a fresh install: it **early-returns** when no Tag/DocumentType/Correspondent has `MATCH_AUTO` (`src/documents/tasks.py:L48`, condition `L50-L52`).
- `index_optimize` returns `None`; it commits a Whoosh optimize (`src/documents/tasks.py:L32`, `writer.commit(optimize=True)` `L35`).
- `sanity_check` returns the literal `"No issues detected."` (`src/documents/tasks.py:L267`); it *also* emitted a Paperless app line (see §4).
- `process_mail_accounts` returns the literal `"No new documents were added."` (`src/paperless_mail/tasks.py:L22`) and, with **zero mail accounts configured**, never enters its loop body `for account in MailAccount.objects.all():` (`src/paperless_mail/tasks.py:L13`), so it logs nothing. The zero-account precondition is confirmed live — command + verbatim output (sink `stdout` via `manage.py shell`; the `MailAccount` model is `src/paperless_mail/models.py:L6`):

```text
$ python3 manage.py shell -c "from paperless_mail.models import MailAccount; print('MailAccount_count=', MailAccount.objects.count())"
MailAccount_count= 0
```

Because the count is `0`, the `for` loop at `src/paperless_mail/tasks.py:L13` iterates zero times, `total_new_documents` stays `0` (`:L12`), and control falls straight to the `else` branch returning `"No new documents were added."` (`:L21`–`:L22`) — no `MailAccountHandler` is ever constructed and nothing is logged.

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
23:54:26 [Q] INFO recycled worker Process-1:1
23:54:26 [Q] INFO Process-1:14 ready for work at 294
```

The worker index keeps climbing (`Process-1:14`, `:15`, …) while the **pool size stays fixed at 11** — recycling replaces a worker rather than growing the pool. Observed verbatim, the four catch-up workers were each replaced: `recycled worker Process-1:1 → Process-1:14 ready for work at 294`, `:3 → :15 at 295`, `:2 → :16 at 296`, `:4 → :17 at 297`.

---

## Section 4 — O3: Periodic "healthy/ready" log entries — message + measured frequency + meaning

### 4.1 How the frequency was measured

The idle stack was kept running from cluster start **23:53:56 to the Redis interrupt at 00:17:45** — **~24 minutes** of continuous idle observation, well over the 20-minute minimum, during which the 10-minute mail task recurred **three** times (`23:57:26`, `00:07:27`, `00:17:29` — see §4.2). Cadence was derived from the timestamps django-q prints on each `[Q]` line (`%H:%M:%S`, §3.2). The single most important empirical finding: **between scheduled task fires, `stdout` is completely silent.**

To prove this, `qcluster.log`'s line count was sampled once per minute for the whole window with an explicit background sampler (the exact command that produced the timeline, sink `stdout`):

```text
$ docker exec -d pl-idle-obs bash -lc \
    'for i in $(seq 1 44); do printf "%s -> %s lines\n" \
       "$(date +%H:%M:%S)" "$(wc -l < /tmp/obs/qcluster.log)" \
       >> /tmp/obs/linecount_samples.txt; sleep 60; done'
$ docker exec pl-idle-obs bash -lc 'cat /tmp/obs/linecount_samples.txt'
23:54:25 -> 16 lines
23:55:25 -> 45 lines
23:56:25 -> 45 lines
23:57:25 -> 45 lines
23:58:25 -> 52 lines
23:59:25 -> 52 lines
00:00:25 -> 52 lines
00:01:25 -> 52 lines
00:02:25 -> 52 lines
00:03:25 -> 52 lines
00:04:25 -> 52 lines
00:05:25 -> 52 lines
00:06:25 -> 52 lines
00:07:25 -> 52 lines
00:08:25 -> 59 lines
00:09:25 -> 59 lines
00:10:25 -> 59 lines
00:11:25 -> 59 lines
00:12:25 -> 59 lines
00:13:25 -> 59 lines
00:14:25 -> 59 lines
00:15:25 -> 59 lines
00:16:25 -> 59 lines
00:17:25 -> 59 lines
```

Reading this timeline: the count jumps `16 → 45` as the startup lines plus the one-time catch-up burst land, then sits **frozen at 45** for three minutes, jumps to **52** (the `23:57:26` mail fire appended exactly **7** lines, sampled at `23:58:25`), stays frozen for ten minutes, jumps to **59** (the `00:07:27` mail fire, +7, sampled at `00:08:25`), and again stays frozen for ten minutes. Each mail fire adds exactly 7 lines; nothing else is written between fires (samples lag a fire by up to ~60 s because sampling is at `:25` past each minute).

Consequently the **~30-second django-q scheduler poll produces no log line** when nothing is due. That poll is real but silent: the guard loop calls `scheduler()` only when its accumulator crosses 30 s — `django_q/cluster.py:L283-L286` `counter += cycle; if counter >= 30 and Conf.SCHEDULER: counter = 0; scheduler(...)`, with `cycle = Conf.GUARD_CYCLE = 0.5` (`django_q/conf.py:L90`) — and `scheduler()` only emits a line (`created a task from schedule …`, `django_q/cluster.py:L669`) when a schedule's `next_run < now`.

### 4.2 The measured periodic-log inventory

Each row's own producing command + verbatim evidence is given immediately below the table (E1–E4); no row relies on another section for its proof.

| Exact message string (verbatim) | Sink | Measured frequency | What it indicates | Evidence |
|---|---|---|---|---|
| `Process-1 created a task from schedule [Check all e-mail accounts]` (with its `Enqueued 1` / `processing` / `Processed` / `recycled worker` / `ready for work` burst) | `stdout` | **~10 min** — consecutive-recurring deltas `23:57:26→00:07:27 = 601 s` and `00:07:27→00:17:29 = 602 s` | The **only** recurring idle heartbeat on stdout; proves scheduler + pusher + worker + monitor + recycle are all alive | **E1** |
| `recycled worker Process-1:N` → `Process-1:M ready for work at <pid>` | `stdout` | **once per task execution** (`recycle=1`) | Cluster self-maintenance: a worker is replaced after each task | **E1** (in each fire block) |
| `Q Cluster <name> running.` | `stdout` | **once per cluster start** (not periodic) | Cluster reached ready state | **E2** |
| `Using inotify to watch directory for changes: /app/src/../consume` | `stdout` **and** `data/log/paperless.log` | **once at start** (not periodic), then silent | Consumer is ready and watching in inotify mode | **E3** |
| `Server is ready. Spawning workers` | `stdout` | **once per gunicorn start** (not periodic) | Web server ready | **E4** |
| (~30 s django-q scheduler poll) | — | **~30 s, but emits NO line** when nothing is due | Scheduler is alive; silent unless a schedule is due | frozen line-count timeline, §4.1 |

**E1 — the recurring 10-minute mail heartbeat.** Command listing every schedule fire (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
$ docker exec pl-idle-obs bash -lc "grep 'created a task from schedule' /tmp/obs/qcluster.log"
23:54:25 [Q] INFO Process-1 created a task from schedule [Train the classifier]
23:54:25 [Q] INFO Process-1 created a task from schedule [Optimize the index]
23:54:25 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
23:54:25 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
23:57:26 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
00:07:27 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
00:17:29 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

The four `23:54:25` lines are the one-time catch-up burst; the three `Check all e-mail accounts` lines at `23:57:26`, `00:07:27`, `00:17:29` are the recurring heartbeat — **deltas `601 s` and `602 s` ≈ 10 min** (the `23:54:25→23:57:26` gap of `181 s` is *not* the cadence: it is the phase-lock artifact discussed in §4.2.1). Two consecutive full fire blocks, verbatim (sink `stdout`), showing the whole per-fire burst including the `recycle=1` replacement:

```text
23:57:26 [Q] INFO Enqueued 1
23:57:26 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
23:57:26 [Q] INFO Process-1:5 processing [oranges-four-monkey-bacon]
23:57:26 [Q] INFO Process-1:5 stopped doing work
23:57:26 [Q] INFO Processed [oranges-four-monkey-bacon]
23:57:26 [Q] INFO recycled worker Process-1:5
23:57:26 [Q] INFO Process-1:18 ready for work at 373
```

```text
00:07:27 [Q] INFO Enqueued 1
00:07:27 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
00:07:27 [Q] INFO Process-1:6 processing [rugby-leopard-hot-emma]
00:07:27 [Q] INFO Process-1:6 stopped doing work
00:07:27 [Q] INFO Processed [rugby-leopard-hot-emma]
00:07:28 [Q] INFO recycled worker Process-1:6
00:07:28 [Q] INFO Process-1:19 ready for work at 603
```

**E2 — cluster ready marker.** Command (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
$ docker exec pl-idle-obs bash -lc "grep 'Q Cluster' /tmp/obs/qcluster.log"
23:53:56 [Q] INFO Q Cluster apart-maine-zebra-charlie starting.
23:53:56 [Q] INFO Q Cluster apart-maine-zebra-charlie running.
```

**E3 — consumer readiness.** Command (sink `data/log/paperless.log`):

```text
$ docker exec pl-idle-obs bash -lc 'grep inotify /app/data/log/paperless.log'
[2026-07-01 23:53:56,076] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

**E4 — web-server readiness.** Command (sink `stdout`, `/tmp/obs/gunicorn.log`):

```text
$ docker exec pl-idle-obs bash -lc 'cat /tmp/obs/gunicorn.log'
[2026-07-01 23:53:55 +0000] [122] [INFO] Starting gunicorn 20.1.0
[2026-07-01 23:53:55 +0000] [122] [INFO] Listening at: http://0.0.0.0:8000 (122)
[2026-07-01 23:53:55 +0000] [122] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 23:53:55 +0000] [122] [INFO] Server is ready. Spawning workers
```

#### 4.2.1 The mail cadence is phase-locked (reported exactly as observed)

The first *recurring* fire (`23:57:26`) is only `181 s` after the catch-up fire (`23:54:25`), **not** ~600 s. This is expected and is reported honestly rather than smoothed: django-q's `MINUTES` schedule advances `next_run` to a fixed minute boundary derived from the pre-baked `next_run`, so recurring fires are phase-locked to that boundary (here ≈ `:X7:2x`). The **true cadence** is therefore the delta between *consecutive recurring* fires — `601 s` and `602 s` — while the catch-up→first-recurring gap is a one-off alignment step.

### 4.3 What does NOT appear periodically within the window (reported exactly)

- **Hourly `Train the classifier`** did not recur within *this* ~24-minute window: like the mail task (§4.2.1), its `next_run` is **phase-locked to the same pre-baked `:47` minute boundary** — django-q advances it to the *next occurrence* of that boundary, **not** to catch-up + 1 h — so after the `23:54:25` catch-up fire it advances to ≈ `00:47`, which is beyond the window (idle observation ended at the Redis interrupt, `00:17:45`). (The `:47` boundary is the schedule's observed `next_run` minute — the same base that yields the mail fires at `:57`/`:07`/`:17` in §3.3; verified live by inspecting `Schedule.next_run`, which reads `…:47:18` for this HOURLY row and steps by exactly one hour after each fire.) Consequently, whether the hourly task recurs inside a ~24-minute window depends on where the catch-up lands relative to the `:47` boundary: here the `:54` catch-up left the next fire `~53 min` out, whereas a catch-up shortly *before* the boundary makes it recur within ~10 min. On this clean system its task body is silent (`result=None`, §3.3), so even when it does fire the only trace is the `[Q]` `created a task from schedule [Train the classifier]` / `processing` / `Processed` wrapper lines.
- **Daily `Optimize the index`** (`next_run 2026-07-02`) and **Weekly `Perform sanity check`** (`next_run 2026-07-08`) will not recur within any ~24-min window; their cadence is stated from the migrations (§3.3). Both were, however, observed executing once in the catch-up burst (§3.3).

### 4.4 The `paperless.*` application sinks while idle

Only two application lines were written during the whole idle window, both from `paperless.*` loggers. Command + verbatim (sink `data/log/paperless.log`):

```text
$ docker exec pl-idle-obs bash -lc 'cat /app/data/log/paperless.log'
[2026-07-01 23:53:56,076] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
[2026-07-01 23:54:25,767] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

The observed `Sanity checker detected no issues.` line is emitted by `src/documents/sanity_checker.py:L27` (`logger.info("Sanity checker detected no issues.")`); the `Using inotify to watch directory for changes: …` line is emitted by `src/documents/management/commands/document_consumer.py:L200`. Both of these **also** appear on `stdout`. This is by design: the root logger routes to the `console` handler (`src/paperless/settings.py:L407` `"root": {"handlers": ["console"]}`, StreamHandler `L389`, level `INFO` when `DEBUG` is off `L388`, verbose formatter `L390`/`L378`), while the `paperless` logger additionally writes to `paperless.log` (`src/paperless/settings.py:L409` `"paperless": {"handlers": ["file_paperless"], …}`, file at `L395`). Because `disable_existing_loggers` is `False` (`L375`) and no logger sets `propagate=False`, records reach **both** the file handler and (via propagation to root) the console.

> **Observed nuance — `mail.log` is never created at idle.** The `paperless_mail` logger → `mail.log` (`src/paperless/settings.py:L410`, file at `L402`) never fired, so the file does not exist. Command + verbatim (sink `stdout`):
>
> ```text
> $ docker exec pl-idle-obs bash -lc 'ls -la /app/data/log'
> total 16
> drwxr-sr-x 1 testuser testuser 4096 Jul  1 23:53 .
> drwxr-sr-x 1 testuser testuser 4096 Jul  2 00:17 ..
> -rw-r--r-- 1 testuser testuser    0 Jul  1 21:41 .__mail.lock
> -rw-r--r-- 1 testuser testuser    0 Jul  1 23:54 .__paperless.lock
> -rw-r--r-- 1 testuser testuser  226 Jul  1 23:54 paperless.log
> ```
>
> Note also that the mail task's logger is `getLogger("paperless.mail.tasks")` (`src/paperless_mail/tasks.py:L8`) — under the `paperless.*` hierarchy, so *if* it logged, it would land in `paperless.log`, not `mail.log`. At idle it emits nothing; `"No new documents were added."` is only a `Task.result` value (§3.3), never a log line in any sink.

---

## Section 5 — O4: Recovery after a brief interrupt/restart

The interrupt/restart steps are **reversible operational actions on running components — no code was edited.** Two kinds of perturbation were exercised: (5.1) briefly stopping the shared **Redis** service, and (5.2–5.3) restarting each of the three application programs.

### 5.1 Interrupting Redis — the broker-loss symptom

**Action + timing** (host clock): `docker stop paperless-redis` at `00:17:45`, `docker start paperless-redis` at `00:18:31` (~46 s outage).

While Redis was down, the running django-q pusher lost its broker connection. **Verbatim reaction** (sink `stdout`, `/tmp/obs/qcluster.log`):

```text
00:17:46 [Q] ERROR Error -5 connecting to paperless-redis:6379. No address associated with hostname.
00:17:55 [Q] INFO Process-1:13 stopped pushing tasks
```

(The very first loss — logged via a full traceback rather than a single `[Q]` line — was `redis.exceptions.ConnectionError: Connection closed by server.` raised from `broker.dequeue()` → `blpop` at the moment of `docker stop`; once the stopped container's DNS entry disappeared, the pusher then flooded `Error -5 connecting to paperless-redis:6379. No address associated with hostname.` roughly twice a second until Redis returned. The `stopped pushing tasks` line is from `django_q/cluster.py:L366`.)

> **Reported exactly — the `src/documents/tasks.py:L231` hint did NOT appear.** The code has `logger.warning("OSError. It could be, the broker cannot be reached.")` (`src/documents/tasks.py:L231`), but that lives inside `sanity_check`'s handler and only fires if a *document* task is running during broker loss. No document task was in flight during the interrupt, so it never triggered. Command + verbatim (sink `stdout`):
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

**The running cluster self-healed without a restart** — the guard reincarnated the dead pusher and it resumed once Redis returned. First self-heal cycle, verbatim (sink `stdout`):

```text
00:17:55 [Q] INFO Process-1:13 stopped pushing tasks
00:17:55 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
00:17:55 [Q] INFO Process-1:21 pushing tasks at 863
```

Reported exactly: the guard kept reincarnating the pusher (`Process-1:21`→`:22`→`:23`→`:24`→`:25`, fresh PIDs `863`…`871`) because each newborn also failed while Redis was still down; after `docker start paperless-redis` (`00:18:31`) the reincarnated pusher held. A fresh `Process-1:NN pushing tasks at <pid>` line (from `django_q/cluster.py:L342`) confirms the broker path is operational again.

### 5.3 Restarting the three programs — each readiness marker

Each program was stopped (via `pkill` inside the container's isolated PID namespace) and relaunched; each re-emitted its readiness marker.

**Scheduler / qcluster** — verbatim (sink `stdout`, `/tmp/obs/qcluster_restart.log`):

```text
00:18:59 [Q] INFO Q Cluster pip-lactose-sink-jupiter starting.
00:18:59 [Q] INFO Process-1:12 monitoring at 962
00:18:59 [Q] INFO Process-1 guarding cluster pip-lactose-sink-jupiter
00:18:59 [Q] INFO Process-1:13 pushing tasks at 963
00:18:59 [Q] INFO Q Cluster pip-lactose-sink-jupiter running.
```

The completion marker `Q Cluster pip-lactose-sink-jupiter running.` (`django_q/cluster.py:L261`) confirms the cluster is operational again — and its **different random name** (vs. `apart-maine-zebra-charlie`) re-confirms the §3.1 finding that the display name is a per-run humanized id, not `Q_CLUSTER["name"]`.

**Consumer** — verbatim (sink `stdout`, `/tmp/obs/consumer_restart.log`):

```text
[2026-07-02 00:18:59,593] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

Re-emits the readiness line from `src/documents/management/commands/document_consumer.py:L200`.

**Web server / gunicorn** — reported exactly as observed, in two parts: (a) the original in-run restart attempt, and (b) a clean restart that captures the readiness marker directly.

**(a) First attempt — same-PID-namespace `pkill` did not stop the master.** In the initial perturbation the `pkill` did **not** terminate the original gunicorn (its master, PID `122`, kept running and holding the listening socket), so the *relaunched* gunicorn could not bind `:8000` and gave up after retrying. Verbatim (sink `stdout`, `/tmp/obs/gunicorn_restart.log`):

```text
[2026-07-02 00:18:59 +0000] [936] [INFO] Starting gunicorn 20.1.0
[2026-07-02 00:18:59 +0000] [936] [ERROR] Connection in use: ('0.0.0.0', 8000)
[2026-07-02 00:18:59 +0000] [936] [ERROR] Retrying in 1 second.
[2026-07-02 00:19:00 +0000] [936] [ERROR] Connection in use: ('0.0.0.0', 8000)
[2026-07-02 00:19:00 +0000] [936] [ERROR] Retrying in 1 second.
[2026-07-02 00:19:04 +0000] [936] [ERROR] Can't connect to ('0.0.0.0', 8000)
```

Because the original web process never went down, the endpoint stayed continuously operational throughout — verified live after the perturbation (sink `stdout`):

```text
$ docker exec -u testuser -w /app/src pl-idle-obs bash -lc \
    "python3 -c \"import urllib.request; print('HTTP', urllib.request.urlopen('http://127.0.0.1:8000/',timeout=5).status)\""
HTTP 200
```

**(b) Clean restart — the master was fully stopped first, so gunicorn's own readiness marker was captured post-restart.** To capture the web server's `when_ready` marker directly (rather than infer continuity from `HTTP 200`), gunicorn was cleanly restarted on a fresh idle bring-up of the same pinned image/branch: the master was sent `SIGTERM`, and its exit was confirmed to have released `:8000` (`socket.connect_ex(("127.0.0.1",8000))` returned `111` = ECONNREFUSED = free) **before** relaunching. The fresh master then bound the port and emitted the `when_ready(server)` marker — `gunicorn.conf.py:L17`–`L18` `server.log.info("Server is ready. Spawning workers")`. Verbatim (sink `stdout`, `/tmp/obs/gunicorn_restart.log`):

```text
[2026-07-02 03:21:04 +0000] [213] [INFO] Starting gunicorn 20.1.0
[2026-07-02 03:21:04 +0000] [213] [INFO] Listening at: http://0.0.0.0:8000 (213)
[2026-07-02 03:21:04 +0000] [213] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-02 03:21:04 +0000] [213] [INFO] Server is ready. Spawning workers
```

The fresh master (**PID `213`**, distinct from the original) confirms this is a genuine post-restart start; immediately afterward the endpoint again answered `HTTP 200`. Its `03:21:04` timestamp is later than the `00:18`–`00:20` Redis-interrupt run above because this clean restart was a follow-up capture on a fresh idle container from the same image at branch `paperless-ngx_542221a38dff` — reported honestly rather than back-dated into the earlier run. The same `Server is ready. Spawning workers` marker also appears once at each original program start in §4.2 (evidence **E4**).

### 5.4 Proof the whole stack is operational again

The observation harness was stopped (`00:20:29`) before the next 10-minute mail fire was due (≈ `00:27`), so **no post-restart scheduled fire was captured in this run** — reported honestly rather than asserted. The end-to-end operational proof is therefore the set of readiness markers already captured above plus a drained task queue. Immediately after recovery the queue was empty — command + verbatim (sink `stdout`):

```text
$ python3 manage.py shell -c "from django_q.models import OrmQ; print('OrmQ queued now:', OrmQ.objects.count())"
OrmQ queued now: 0
```

This is the recovery signature end-to-end: **Redis reconnected (§5.2, `Connected to Redis broker: …`) → cluster running (§5.3, `Q Cluster pip-lactose-sink-jupiter running.`) → consumer watching (§5.3, `Using inotify to watch directory for changes: …`) → web ready and serving (§5.3, `Server is ready. Spawning workers` after a clean restart + `HTTP 200`) → task queue drained (`OrmQ queued now: 0`).**

---

## Section 6 — Coverage pass

Re-reading the original question and confirming every distinct sub-question and every named item is answered above:

- [x] **"what background processes or tasks continue executing automatically?"** → the django-q `qcluster` process tree (sentinel/guard + 11 workers + monitor + pusher) and the four periodic schedules — §3.1, §3.3.
- [x] **Each of the four periodic tasks by name, with cadence:**
  - [x] `Train the classifier` — HOURLY (`src/documents/migrations/1001_auto_20201109_1636.py:L10-L14`) — §3.3.
  - [x] `Optimize the index` — DAILY (`src/documents/migrations/1001_auto_20201109_1636.py:L15-L19`) — §3.3.
  - [x] `Perform sanity check` — WEEKLY (`src/documents/migrations/1004_sanity_check_schedule.py:L10-L14`) — §3.3.
  - [x] `Check all e-mail accounts` — every 10 MINUTES (`src/paperless_mail/migrations/0002_auto_20201117_1334.py:L10-L15`) — §3.3, §4.2.
- [x] **"the actual log entries that appear periodically … the specific log messages, their frequency, and what they indicate"** → the O3 table with **measured** cadences (mail deltas 601 s and 602 s ≈ 10 min; scheduler ~30 s but silent) — §4.1, §4.2, each row backed by a verbatim line with its sink.
- [x] **"if you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again?"** → the Redis interrupt + `Connected to Redis broker: …` (`docker/wait-for-redis.py:L41`), cluster self-heal (`reincarnated pusher …` → `… pushing tasks at 863`), `Q Cluster … running.`, `Using inotify to watch directory for changes: …`, `Server is ready. Spawning workers`, and the end-to-end operational proof (`HTTP 200` + drained queue) — §5.1–§5.4.
- [x] **"What components or processes keep running continuously … even when no documents are being processed?"** → the three Supervisord programs (gunicorn / consumer / scheduler) plus Redis and the database — §2.1, §2.2.
- [x] **Idle-only scope** — every quoted heartbeat is from the document-free state (empty consume dir, drained queue); no ingestion/OCR line is reported as idle behavior.
- [x] **Measured, not inferred, frequency** — the idle window was ~24 minutes (`23:53:56`→`00:17:45`); the 10-minute mail task recurred at 23:57:26 / 00:07:27 / 00:17:29 (three recurring fires; consecutive-recurring deltas 601 s and 602 s), with the catch-up→first-recurring 181 s gap reported honestly as a phase-lock artifact (§4.2.1).
- [x] **Read-only & cleanup** — no source file was modified (verified: `git diff --name-status 542221a38dff..HEAD` lists **only** this document, and no other tracked file differs from the pinned commit); every temporary artifact (the throwaway observation container and the host-side capture files) lived outside the repo tree and was removed. With the deliverable committed, `git status --porcelain` is empty. Command + verbatim output:

```text
$ git diff --name-status 542221a38dff06361e07976452f9aea24d210542..HEAD
A	blitzy/documentation/paperless-ngx_542221a38dff.md
$ git status --porcelain
(no output — working tree clean once the deliverable is committed)
```

### Anticipated-vs-observed corrections (reported exactly as observed)

- The cluster's displayed name is a **random humanized id** (`apart-maine-zebra-charlie`, later `pip-lactose-sink-jupiter`), **not** `paperless`; `Q_CLUSTER["name"]="paperless"` (`src/paperless/settings.py:L450`) is the internal identifier only — §3.1.
- `"No new documents were added."` is a **`Task.result` value**, not a stdout/file log line; `mail.log` is never created at idle — §3.3, §4.4.
- The ~30-second scheduler poll is real but **emits no log line** when nothing is due — §4.1.
- The `src/documents/tasks.py:L231` "OSError…" broker-loss hint did **not** appear (no document task was in flight during the Redis interrupt) — §5.1.

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
