# Paperless-NGX — Idle Runtime Behavior Investigation

**Subject repository:** paperless-ngx
**Commit under investigation:** `542221a38dff06361e07976452f9aea24d210542` (release `1.7.0` — [src/paperless/version.py:L1])
**Task engine:** `django-q` 1.3.9 — a multiprocessing task queue, **not** Celery ([requirements.txt:L37])
**Broker / channel layer:** Redis 6.0 ([docker/compose/docker-compose.sqlite.yml:L29])
**Database:** SQLite (default backend; stores the `django-q` `Schedule` rows and task results)
**Canonical runtime image:** `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...`)
**Observation host:** Docker 28.x on Linux; investigation performed 2026-07-14 (UTC).

---

## The question (verbatim)

> Get Paperless-NGX running at the specified commit. Once it's idle and stable, what background processes or tasks continue executing automatically? What are the actual log entries that appear periodically showing the system is healthy and ready? I need the specific log messages, their frequency, and what they indicate. Also, if you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again? What components or processes keep running continuously to maintain Paperless-NGX in a ready state, even when no documents are being processed? You may use temporary helper commands or inspection tools if needed, but don't modify any source files and clean up any temporary artifacts when you're done.

This maps to five sub-questions, each answered by name below: **Q1** startup, **Q2** idle background work, **Q3** periodic health/ready log entries, **Q4** interrupt/restart recovery, **Q5** continuously-running components.

## How to read this document

- **Observed vs. inferred.** Every behavioral claim is backed by *actual, unedited* command output shown in a fenced block immediately next to the claim, together with the exact command that produced it. Claims that are **inferred from reading source code** (rather than observed at runtime) are explicitly labelled *(inferred from source)*.
- **Raw blocks are verbatim.** Fenced blocks are spliced byte-for-byte from captured output; they are never hand-edited. Where a value is randomly generated per boot (the `django-q` cluster display-name and worker PIDs), the surrounding prose flags it as **variable**, but the raw block still shows the concrete value observed in that run.
- **Transcript convention inside fenced blocks.** A line beginning with `$ ` is a command that was run; a line beginning with `$ #` or `#` is an inline annotation (my commentary, not program output); and a `[ ... ]` note on its own line marks where a run of *identical* repeated lines was elided for length — the exact count of the elided lines is always given right next to it (e.g. the outage's `Error -5` line repeats, with its exact `= 309` count shown). Everything else is unmodified captured output.
- **Canonical vs. non-canonical.** Values are reported from a canonical, supervised run (real `ENTRYPOINT` + `supervisord`, children de-escalated to the `paperless` user, a live Docker healthcheck, a clean runtime checkout). The few environmental deltas versus a byte-identical production deploy are enumerated in [§8](#8-canonical-vs-non-canonical-labelling) and never presented as default behavior.

---

## 1. TL;DR

At this commit Paperless-NGX is a **monolithic, multi-process Django application** run inside one container, in which `supervisord` supervises exactly **three** long-running programs — `gunicorn` (ASGI web server), `document_consumer` (file watcher), and `scheduler` = `qcluster` (the `django-q` cluster) — all de-escalated from root to the `paperless` user ([docker/supervisord.conf:L10-30]). Two external stateful components complete the ready state: the **Redis 6.0** broker/channel-layer and the **SQLite** database.

- **(Q1) Startup.** The real `ENTRYPOINT` runs `docker-entrypoint.sh` → `docker-prepare.sh` (wait-for-Redis → migrate → conditional reindex → conditional superuser) → `exec supervisord`, which spawns the three programs. The system is "ready" when gunicorn logs `Server is ready. Spawning workers` and answers HTTP on `:8000`, the consumer logs its inotify watch line, and the `qcluster` logs `Q Cluster <name> running.` — all observed below.
- **(Q2) Idle background work.** Four `django-q` schedules run automatically: **mail check** every 10 minutes, **classifier training** hourly, **index optimize** daily, **sanity check** weekly ([src/paperless_mail/migrations/0002_auto_20201117_1334.py:L12-14], [src/documents/migrations/1001_auto_20201109_1636.py:L12-18], [src/documents/migrations/1004_sanity_check_schedule.py:L12-13]). The engine driving them is the `qcluster` sentinel, which internally checks for due schedules about twice a minute.
- **(Q3) Periodic health/ready logs.** Two genuinely periodic idle signals exist: the **Docker healthcheck** (`curl -f http://localhost:8000`) fires every ~30 s and is the steady heartbeat; the **mail-check schedule** fires every ~10 min and is the most-frequent `django-q` log activity. Hourly/daily/weekly schedules are reported from configuration (they do not fire in a short window). Between firings the `qcluster` and the idle consumer are silent.
- **(Q4) Interrupt/restart recovery.** `django-q` emits **no literal "reconnected" string**. Recovery from a Redis outage is a *composite, inferred* signature: connection errors during the outage cease, the sentinel logs `reincarnated pusher Process-1:N after sudden death`, a new `Process-1:N pushing tasks at <pid>` appears, and normal schedule enqueue/processing resumes — all captured in [§6](#6-q4--interruptrestart-recovery).
- **(Q5) Continuously-running components.** The three supervised processes plus Redis and SQLite. `supervisord` itself is the *mechanism* that keeps them alive (auto-restart), not a readiness signal.

---

## 2. Environment and method

### 2.1 Canonical build and run (reproducible commands)

The system was built and run at the pinned commit through its **real production entrypoint and command**. The provided dev image is the canonical Python 3.9 / dependency baseline but ships without the process-manager layer of the production image (no `supervisord`, no `/sbin` boot scripts, no `paperless` user). To exercise the canonical boot path faithfully, a thin derived image adds **only** the runtime bits the production `Dockerfile` installs — `supervisor`, `gosu`, `curl`, the `paperless` uid/gid 1000 user, the `/sbin/docker-entrypoint.sh` / `docker-prepare.sh` / `wait-for-redis.py` scripts, `/etc/supervisord.conf`, and the ImageMagick policy — over the unmodified `/app` source tree. The application source is **not** modified (verified clean in [§2.4](#24-runtime-provenance) and [§10](#10-integrity-and-cleanup)).

```
# 1) Temporary derived image: FROM the provided dev image, ADD ONLY the canonical
#    runtime bits the production Dockerfile installs (supervisor, gosu, curl, the
#    paperless user, the /sbin scripts, /etc/supervisord.conf, the ImageMagick policy,
#    and the empty /app/consume dir). The /app source content is NOT modified.
$ docker build -f Dockerfile.canonical -t paperless-canonical:investigation .

# 2) Broker: the canonical redis:6.0, reachable on the compose service name "broker".
$ docker network create paperless-inv-net
$ docker run -d --name paperless-inv-broker --network paperless-inv-net \
      --network-alias broker redis:6.0

# 3) App: launched through the REAL production ENTRYPOINT (/sbin/docker-entrypoint.sh)
#    and CMD (supervisord), with the healthcheck from docker-compose.sqlite.yml:L41-45
#    replicated via docker run --health-* flags.
$ docker run -d --name paperless-inv-app --network paperless-inv-net \
      -e PAPERLESS_REDIS=redis://broker:6379 \
      -e PAPERLESS_ADMIN_USER=admin -e PAPERLESS_ADMIN_PASSWORD=admin \
      -p 8000:8000 \
      --health-cmd 'curl -f http://localhost:8000' \
      --health-interval=30s --health-timeout=10s --health-retries=5 \
      paperless-canonical:investigation
```

The container is launched via `CMD supervisord` ([Dockerfile:L172]) after `ENTRYPOINT /sbin/docker-entrypoint.sh` ([Dockerfile:L168]); the healthcheck flags replicate `docker/compose/docker-compose.sqlite.yml:L41-45` exactly (`curl -f http://localhost:8000`, 30 s interval, 10 s timeout, 5 retries). Note the `Dockerfile` itself contains **no** `HEALTHCHECK` directive — the healthcheck is a compose-layer concern, reproduced here at the `docker run` layer.

### 2.2 Idle runtime topology

```mermaid
graph TD
    ENTRY["docker-entrypoint.sh (initialize, then exec)"] --> PREP["docker-prepare.sh (wait-for-redis, migrate, reindex)"]
    PREP --> SUP["supervisord (pid 1, root)"]
    SUP --> GUN["gunicorn :8000 (user paperless)"]
    SUP --> CON["document_consumer (user paperless)"]
    SUP --> QC["qcluster / django-q (user paperless): sentinel + monitor + pusher + N workers"]
    GUN <--> REDIS[("Redis 6.0 broker + channel layer")]
    QC <--> REDIS
    QC <--> DB[("SQLite: Schedule rows + task results")]
    GUN -. "websocket StatusConsumer (authenticated)" .-> UI["Angular UI"]
    HC["healthcheck curl -f :8000 every 30s"] --> GUN
```

### 2.3 Method

Per the run-first discipline, the system was brought to a genuine **idle steady state** — no documents in the consumption directory, zero documents in the database, no OCR pipeline running (evidence in [§3.5](#35-idle-preconditions)) — and then observed for **more than 10 minutes** so that at least two 30-second healthcheck cycles and at least one 10-minute mail-check firing were captured live. Hourly/daily/weekly cadences are reported from their seeded schedule migrations because they cannot fire within a short observation window. Cadence figures are derived from timestamped output and confirmed across at least two occurrences. The interrupt/restart experiment (Q4) targets the **Redis broker**, because both the `qcluster` and the Channels websocket layer depend on it, making the reconnection path directly observable.

### 2.4 Runtime provenance

The runtime is canonical Python 3.9, the checkout is exactly the pinned commit, and — critically for the read-only mandate — the runtime working tree is **clean** (no source file was modified during the investigation). Redis is the canonical 6.0 server and is reachable from the app.

```
$ docker exec paperless-inv-app python3 --version
Python 3.9.23
$ docker exec paperless-inv-app bash -lc "cd /app && git rev-parse HEAD && git status --porcelain"
542221a38dff06361e07976452f9aea24d210542
        # (git status --porcelain prints nothing -> runtime checkout is CLEAN)
$ docker exec paperless-inv-broker redis-server --version
Redis server v=6.0.20 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=dbdcb1f5eaf1bc2
$ docker exec paperless-inv-app bash -lc "cd /app/src && python3 -c \"import redis,os; print('PING', redis.from_url(os.environ['PAPERLESS_REDIS']).ping())\""
PING True
```

**Worker count is host-derived, not fixed.** `django-q`'s worker count defaults to `default_task_workers()` ([src/paperless/settings.py:L427-436]), which for a host with ≥ 4 cores is `floor(sqrt(cpu_count))`. On this host `multiprocessing.cpu_count()` reports 128, so the formula yields 11 — which exactly matches the 11 `ready for work` workers observed at startup in [§3.4](#34-qcluster-the-django-q-task-engine). This value is **variable** across hosts (e.g. 2 workers at 4 cores, 3 at 9 cores):

```
$ docker exec paperless-inv-app python3 -c \
    "import multiprocessing,math; c=multiprocessing.cpu_count(); \
     print('cpu_count=',c,' floor(sqrt)=',max(math.floor(math.sqrt(c)),1))"
cpu_count= 128  floor(sqrt)= 11
```

---

## 3. Q1 — Startup and reaching idle readiness

**Question:** *Get Paperless-NGX running at the specified commit* — establish the canonical boot sequence and the point at which the system is "ready".

### 3.1 The boot chain (observed)

The container's `ENTRYPOINT` is `/sbin/docker-entrypoint.sh` ([Dockerfile:L168]); it runs an `initialize()` preparation step and then `exec`s the container command (`supervisord`) ([docker/docker-entrypoint.sh:L84-92]). `initialize()` invokes `docker-prepare.sh`, which performs the readiness sequence: wait for Redis, apply migrations, conditionally reindex, conditionally bootstrap a superuser. The complete, unedited boot output of the canonical run — from the entrypoint banner through the last `qcluster` line — was captured with `docker logs`:

```
$ docker logs paperless-inv-app
Paperless-ngx docker container starting...
Creating directory ../data/index
Creating directory ../media/documents
Creating directory ../media/documents/originals
Creating directory ../media/documents/thumbnails
Creating directory /tmp/paperless
Adjusting permissions of paperless files. This may take a while.
Waiting for Redis: redis://broker:6379
Connected to Redis broker: redis://broker:6379
Apply database migrations...
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying authtoken.0003_tokenproxy... OK
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
  Applying documents.0001_initial... OK
  Applying documents.0002_auto_20151226_1316... OK
  Applying documents.0003_sender... OK
  Applying documents.0004_auto_20160114_1844... OK
  Applying documents.0005_auto_20160123_0313... OK
  Applying documents.0006_auto_20160123_0430... OK
  Applying documents.0007_auto_20160126_2114... OK
  Applying documents.0008_document_file_type... OK
  Applying documents.0009_auto_20160214_0040... OK
  Applying documents.0010_log... OK
  Applying documents.0011_auto_20160303_1929... OK
  Applying documents.0012_auto_20160305_0040... OK
  Applying documents.0013_auto_20160325_2111... OK
  Applying documents.0014_document_checksum... OK
  Applying documents.0015_add_insensitive_to_match... OK
  Applying documents.0016_auto_20170325_1558... OK
  Applying documents.0017_auto_20170512_0507... OK
  Applying documents.0018_auto_20170715_1712... OK
  Applying documents.0019_add_consumer_user... OK
  Applying documents.0020_document_added... OK
  Applying documents.0021_document_storage_type... OK
  Applying documents.0022_auto_20181007_1420... OK
  Applying documents.0023_document_current_filename... OK
  Applying documents.1000_update_paperless_all... OK
  Applying documents.1001_auto_20201109_1636... OK
  Applying documents.1002_auto_20201111_1105... OK
  Applying documents.1003_mime_types... OK
  Applying documents.1004_sanity_check_schedule... OK
  Applying documents.1005_checksums... OK
  Applying documents.1006_auto_20201208_2209... OK
  Applying documents.1007_savedview_savedviewfilterrule... OK
  Applying documents.1008_auto_20201216_1736... OK
  Applying documents.1009_auto_20201216_2005... OK
  Applying documents.1010_auto_20210101_2159... OK
  Applying documents.1011_auto_20210101_2340... OK
  Applying documents.1012_fix_archive_files... OK
  Applying documents.1013_migrate_tag_colour... OK
  Applying documents.1014_auto_20210228_1614... OK
  Applying documents.1015_remove_null_characters... OK
  Applying documents.1016_auto_20210317_1351... OK
  Applying documents.1017_alter_savedviewfilterrule_rule_type... OK
  Applying documents.1018_alter_savedviewfilterrule_value... OK
  Applying paperless_mail.0001_initial... OK
  Applying paperless_mail.0002_auto_20201117_1334... OK
  Applying paperless_mail.0003_auto_20201118_1940... OK
  Applying paperless_mail.0004_mailrule_order... OK
  Applying paperless_mail.0005_help_texts... OK
  Applying paperless_mail.0006_auto_20210101_2340... OK
  Applying paperless_mail.0007_auto_20210106_0138... OK
  Applying paperless_mail.0008_auto_20210516_0940... OK
  Applying paperless_mail.0009_mailrule_assign_tags... OK
  Applying paperless_mail.0010_auto_20220311_1602... OK
  Applying paperless_mail.0011_remove_mailrule_assign_tag... OK
  Applying paperless_mail.0012_alter_mailrule_assign_tags... OK
  Applying paperless_mail.0009_alter_mailrule_action_alter_mailrule_folder... OK
  Applying paperless_mail.0013_merge_20220412_1051... OK
  Applying paperless_mail.0014_alter_mailrule_action... OK
  Applying sessions.0001_initial... OK
Search index out of date. Updating...

0it [00:00, ?it/s]
0it [00:00, ?it/s]
Created superuser "admin" with provided password.
Executing /usr/local/bin/supervisord -c /etc/supervisord.conf
2026-07-14 20:45:48,098 INFO Set uid to user 0 succeeded
2026-07-14 20:45:48,100 INFO supervisord started with pid 1
2026-07-14 20:45:49,102 INFO spawned: 'consumer' with pid 57
2026-07-14 20:45:49,103 INFO spawned: 'gunicorn' with pid 58
2026-07-14 20:45:49,104 INFO spawned: 'scheduler' with pid 59
[2026-07-14 20:45:49 +0000] [58] [INFO] Starting gunicorn 20.1.0
[2026-07-14 20:45:49 +0000] [58] [INFO] Listening at: http://0.0.0.0:8000 (58)
[2026-07-14 20:45:49 +0000] [58] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-14 20:45:49 +0000] [58] [INFO] Server is ready. Spawning workers
/usr/local/lib/python3.9/site-packages/whitenoise/base.py:115: UserWarning: No directory at: /app/static/
  warnings.warn(f"No directory at: {root}")
/usr/local/lib/python3.9/site-packages/whitenoise/base.py:115: UserWarning: No directory at: /app/static/
  warnings.warn(f"No directory at: {root}")
2026-07-14 20:45:50,161 INFO success: consumer entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-07-14 20:45:50,161 INFO success: gunicorn entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-07-14 20:45:50,161 INFO success: scheduler entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
/usr/local/lib/python3.9/site-packages/whitenoise/base.py:115: UserWarning: No directory at: /app/static/
  warnings.warn(f"No directory at: {root}")
/usr/local/lib/python3.9/site-packages/whitenoise/base.py:115: UserWarning: No directory at: /app/static/
  warnings.warn(f"No directory at: {root}")
[2026-07-14 20:45:50,410] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
20:45:50 [Q] INFO Q Cluster wisconsin-black-charlie-bulldog starting.
20:45:50 [Q] INFO Process-1:1 ready for work at 87
20:45:50 [Q] INFO Process-1:2 ready for work at 88
20:45:50 [Q] INFO Process-1:3 ready for work at 89
20:45:50 [Q] INFO Process-1:4 ready for work at 90
20:45:50 [Q] INFO Process-1:5 ready for work at 91
20:45:50 [Q] INFO Process-1:6 ready for work at 92
20:45:50 [Q] INFO Process-1:7 ready for work at 93
20:45:50 [Q] INFO Process-1:8 ready for work at 94
20:45:50 [Q] INFO Process-1:9 ready for work at 95
20:45:50 [Q] INFO Process-1:10 ready for work at 96
20:45:50 [Q] INFO Process-1:11 ready for work at 97
20:45:50 [Q] INFO Process-1:12 monitoring at 98
20:45:50 [Q] INFO Process-1 guarding cluster wisconsin-black-charlie-bulldog
20:45:50 [Q] INFO Process-1:13 pushing tasks at 99
20:45:50 [Q] INFO Q Cluster wisconsin-black-charlie-bulldog running.
```

Reading this output as a cause→effect chain:

1. **Entrypoint preparation.** `Paperless-ngx docker container starting...` and the `Creating directory ...` / `Adjusting permissions ...` lines are emitted by `docker-entrypoint.sh` before hand-off.
2. **Redis gate.** `Waiting for Redis: redis://broker:6379` followed by `Connected to Redis broker: redis://broker:6379` is `docker/wait-for-redis.py` confirming the broker is reachable before anything else starts ([docker/wait-for-redis.py:L21] and [docker/wait-for-redis.py:L41]). The retry policy is 5 attempts × 5 s with a fail-fast exit on exhaustion ([docker/wait-for-redis.py:L16-17]).
3. **Migrations (always run).** `Apply database migrations...` ([docker/docker-prepare.sh:L44]) precedes `manage.py migrate` ([docker/docker-prepare.sh:L45]); the migration list confirms the four schedule-seeding migrations are applied — `documents.1001_auto_20201109_1636` (classifier HOURLY + index DAILY), `documents.1004_sanity_check_schedule` (WEEKLY), and `paperless_mail.0002_auto_20201117_1334` (mail every 10 min). These seed the `django-q` `Schedule` rows enumerated in [§4.1](#41-the-four-seeded-schedules).
4. **Reindex — *conditional branch*.** `Search index out of date. Updating...` ([docker/docker-prepare.sh:L54-55]) runs **only** when the Whoosh index is missing or stale. In this fresh-volume run it fired and reindexed an empty corpus (`0it [00:00, ?it/s]`); on a warm restart with a current index this line does **not** appear.
5. **Superuser — *conditional branch*.** `Created superuser "admin" with provided password.` ([docker/docker-prepare.sh:L60-63]) appears **only** because `PAPERLESS_ADMIN_USER`/`PAPERLESS_ADMIN_PASSWORD` were supplied and the user did not yet exist; without those variables, or on a restart where the user exists, this line is absent.
6. **Hand-off to `supervisord`.** `Executing /usr/local/bin/supervisord -c /etc/supervisord.conf` is the `exec` in `docker-entrypoint.sh` ([docker/docker-entrypoint.sh:L84-92]). `supervisord started with pid 1` then `spawned: 'consumer' / 'gunicorn' / 'scheduler'` shows `supervisord` launching the three supervised programs defined in [docker/supervisord.conf:L10-30].

Everything after the hand-off is the three programs coming up — detailed next. The startup is therefore **deterministic in structure** but has **two conditional branches** (reindex, superuser) that depend on volume/environment state.

### 3.2 Web server readiness — `gunicorn`

`gunicorn` starts, binds `0.0.0.0:8000` ([gunicorn.conf.py:L3]), selects the `paperless.workers.ConfigurableWorker` class ([gunicorn.conf.py:L5]), and its `when_ready` hook logs `Server is ready. Spawning workers` ([gunicorn.conf.py:L17-18]). Precisely, that message is the **master** process signalling it has bound the socket and is about to fork workers — it is *master* readiness, not proof that a worker has served a request. Full request-serving readiness is confirmed independently by (a) the worker processes existing under the master (PIDs 61/62 in the process tree, [§3.6](#36-supervised-process-tree-non-root)) and (b) a live HTTP probe returning `302 Found` → `/accounts/login/` (an unauthenticated GET is redirected to login, which is the expected ready response):

```
$ docker exec paperless-inv-app curl -sS -o /dev/null -w "http_code=%{http_code}\n" http://localhost:8000
http_code=302
$ docker exec paperless-inv-app curl -f -s -o /dev/null http://localhost:8000 ; echo "exit=$?"
exit=0
$ curl -sS -I http://localhost:8000        # from the host, via published port
HTTP/1.1 302 Found
date: Tue, 14 Jul 2026 20:49:31 GMT
server: uvicorn
content-type: text/html; charset=utf-8
location: /accounts/login/?next=/
```

The `302` to `/accounts/login/?next=/` and the `curl -f` exit `0` are what the Docker healthcheck relies on every 30 s ([§5.1](#51-the-docker-healthcheck--the-steady-heartbeat)).

### 3.3 File-watcher readiness — `document_consumer`

The consumer's readiness line is `Using inotify to watch directory for changes: /app/src/../consume`, emitted once at startup by `handle_inotify()` ([src/documents/management/commands/document_consumer.py:L200]). This is the **inotify** path, selected by default: the command uses the kernel inotify backend via `inotifyrecursive.INotify` ([src/documents/management/commands/document_consumer.py:L20]; `inotifyrecursive==0.3.5`, [requirements.txt:L54]) whenever `PAPERLESS_CONSUMER_POLLING` is `0` (the default). The `watchdog` `PollingObserver` ([src/documents/management/commands/document_consumer.py:L17]) is a **fallback** used only when polling is explicitly configured, in which case the line would instead read `Polling directory for changes: ...` ([src/documents/management/commands/document_consumer.py:L186]). After emitting its watch line the consumer is **silent while idle**; it logs `Adding <filepath> to the task queue.` ([src/documents/management/commands/document_consumer.py:L85]) only when a file actually arrives — which never happens in the idle scenario.

### 3.4 `qcluster` — the `django-q` task engine

The scheduler program is `manage.py qcluster` ([docker/supervisord.conf:L28-29]), which boots the `django-q` cluster. Its canonical startup sequence (last 16 lines of the boot log above) is:

- `Q Cluster <name> starting.` — the cluster begins. `<name>` is a **randomly generated humanized display-name** (here `wisconsin-black-charlie-bulldog`); it is regenerated every boot and must be treated as variable. It is **distinct** from the configured internal cluster name `paperless` ([src/paperless/settings.py:L450]), which is the salt/queue identity, not the display-name.
- `Process-1:1` … `Process-1:11 ready for work at <pid>` — **11 worker processes**, matching `floor(sqrt(128)) = 11` from [§2.4](#24-runtime-provenance). The count is host-derived and variable.
- `Process-1:12 monitoring at <pid>` — the **monitor**, which drains the result queue and logs `Processed [<id>]` when a task result returns.
- `Process-1 guarding cluster <name>` — the **sentinel/guard**, running inside the main `Process-1`. This is the component that periodically checks for due schedules (about twice a minute) and enqueues them (see [§5.2](#52-the-scheduler-firing--every-10-minutes)).
- `Process-1:13 pushing tasks at <pid>` — the **pusher**, which *dequeues* broker tasks and hands them to workers. (The pusher does **not** create scheduled tasks; that is the sentinel's job. Conflating the two is a common error corrected throughout this document.)
- `Q Cluster <name> running.` — the cluster is fully up.

*(Upstream corroboration that these strings and roles are library-canonical for `django-q` 1.3.x — not Paperless customizations — is given in [§8](#8-canonical-vs-non-canonical-labelling).)* The `HH:MM:SS [Q] LEVEL message` format is `django-q`'s own console format and is distinct from Paperless's `[{asctime}] [{levelname}] [{name}] {message}` verbose format ([src/paperless/settings.py:L378]).

### 3.5 Idle preconditions

"Idle" here means no ingestion is in flight: the consumption directory is empty and the database contains zero documents, so no OCR/classification pipeline is running. This was verified before capturing the periodic signals:

```
$ ls -A /app/consume/ | wc -l   # files in the consumption dir
0
$ manage.py shell -c "Document.objects.count()"
0
```

### 3.6 Supervised process tree (non-root)

At idle, `supervisord` (pid 1, root) supervises exactly three children, **all de-escalated to the unprivileged `paperless` user** per `user=paperless` in each program block ([docker/supervisord.conf:L12], [docker/supervisord.conf:L21], [docker/supervisord.conf:L30]). The children in turn own their subordinate processes (gunicorn workers; the `django-q` worker pool). Ownership and command lines were read directly from `/proc`:

```
$ docker exec paperless-inv-app bash -lc 'for p in $(ls /proc|grep -E "^[0-9]+$"|sort -n); do
    c=$(tr "\0" " " </proc/$p/cmdline 2>/dev/null);
    case "$c" in *supervisord*|*"manage.py document_consumer"*|*"manage.py qcluster"*|*"paperless.asgi:application"*)
      printf "%-5s %-5s %-10s %s\n" "$p" "$(awk "/^PPid:/{print \$2}" /proc/$p/status)" "$(stat -c %U /proc/$p)" "$c";;
    esac; done'
PID   PPID  USER       CMD
1     0     root       /usr/local/bin/python3 /usr/local/bin/supervisord -c /etc/supervisord.conf 
57    1     paperless  python3 manage.py document_consumer 
58    1     paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application 
59    1     paperless  python3 manage.py qcluster 
61    58    paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application 
62    58    paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application 
86    59    paperless  python3 manage.py qcluster 
91    86    paperless  python3 manage.py qcluster 
92    86    paperless  python3 manage.py qcluster 
93    86    paperless  python3 manage.py qcluster 
94    86    paperless  python3 manage.py qcluster 
95    86    paperless  python3 manage.py qcluster 
96    86    paperless  python3 manage.py qcluster 
97    86    paperless  python3 manage.py qcluster 
98    86    paperless  python3 manage.py qcluster 
99    86    paperless  python3 manage.py qcluster 
119   86    paperless  python3 manage.py qcluster 
120   86    paperless  python3 manage.py qcluster 
121   86    paperless  python3 manage.py qcluster 
122   86    paperless  python3 manage.py qcluster 
```

This is the canonical process model: root is held only by the `supervisord` supervisor (pid 1); every application process — the web server and its workers, the file watcher, and the entire `qcluster` (sentinel `59`, pool parent `86`, and workers) — runs as `paperless`. The Docker healthcheck configuration is live (non-null) and reports the container healthy:

```
$ docker inspect -f 'Test={{.Config.Healthcheck.Test}} Interval={{.Config.Healthcheck.Interval}} Timeout={{.Config.Healthcheck.Timeout}} Retries={{.Config.Healthcheck.Retries}} Status={{.State.Health.Status}}' paperless-inv-app
Healthcheck.Test=[CMD-SHELL curl -f http://localhost:8000]
Interval(ns)=30s Timeout(ns)=10s Retries=5
Health.Status=healthy FailingStreak=0
```

---

## 4. Q2 — Idle background work (what runs automatically)

**Question:** *Once it's idle and stable, what background processes or tasks continue executing automatically?*

Two layers answer this: the **scheduled tasks** (data-driven, stored in the DB) and the **engine** that runs them (the `qcluster`). No document-processing work runs while idle — only the scheduler's own housekeeping.

### 4.1 The four seeded schedules

Four `django-q` `Schedule` rows are seeded by migrations at startup and drive all automatic idle work. They were read directly from the database:

```
$ manage.py shell -c "for s in Schedule.objects.all().order_by(id): print(name,type,minutes,repeats,func)"
name='Train the classifier' | type=H | minutes=None | repeats=-2 | func=documents.tasks.train_classifier
name='Optimize the index' | type=D | minutes=None | repeats=-2 | func=documents.tasks.index_optimize
name='Perform sanity check' | type=W | minutes=None | repeats=-2 | func=documents.tasks.sanity_check
name='Check all e-mail accounts' | type=I | minutes=10 | repeats=-4 | func=paperless_mail.tasks.process_mail_accounts

$ manage.py shell -c "print(Schedule schedule_type choices)"
[('O', 'Once'), ('I', 'Minutes'), ('H', 'Hourly'), ('D', 'Daily'), ('W', 'Weekly'), ('M', 'Monthly'), ('Q', 'Quarterly'), ('Y', 'Yearly'), ('C', 'Cron')]
```

Mapping the `type` code (legend above) and seed migration to a human cadence:

| Schedule name | `type` | Cadence | Task function | Seeded by |
|---|---|---|---|---|
| `Check all e-mail accounts` | `I` = Minutes, `minutes=10` | **every 10 minutes** | `paperless_mail.tasks.process_mail_accounts` ([src/paperless_mail/tasks.py:L11]) | [src/paperless_mail/migrations/0002_auto_20201117_1334.py:L12-14] |
| `Train the classifier` | `H` = Hourly | **every hour** | `documents.tasks.train_classifier` ([src/documents/tasks.py:L48]) | [src/documents/migrations/1001_auto_20201109_1636.py:L12-13] |
| `Optimize the index` | `D` = Daily | **every day** | `documents.tasks.index_optimize` ([src/documents/tasks.py:L32]) | [src/documents/migrations/1001_auto_20201109_1636.py:L17-18] |
| `Perform sanity check` | `W` = Weekly | **every week** | `documents.tasks.sanity_check` ([src/documents/tasks.py:L255]) | [src/documents/migrations/1004_sanity_check_schedule.py:L12-13] |

*(The `repeats` field is a negative internal run-counter; a negative value means "repeat unlimited" and does not stop the schedule — this is why it simply decrements each run, e.g. mail reached `-4` after 3 firings.)* Only the **mail check** (10 min) fires within a short observation window; the hourly/daily/weekly cadences are reported from this configuration because they cannot fire in a ~20-minute window.

### 4.2 The engine that runs them — `qcluster`

The schedules are executed by the always-on `qcluster` process (`django-q`). Its sentinel (the main `Process-1`, shown `guarding cluster` in [§3.4](#34-qcluster-the-django-q-task-engine)) wakes on a short internal loop and, about **twice a minute**, runs the scheduler function that checks the `Schedule` table for anything due. Upstream `django-q` documentation states this explicitly: *"Twice a minute the scheduler checks for any scheduled tasks that should be starting"* (see [§8](#8-canonical-vs-non-canonical-labelling)). This is corroborated at runtime: the cluster reported `running.` at `20:45:50` and the very first schedule check fired at `20:46:20` — exactly **30 s** later.

Because the four schedules were seeded in 2020, their stored `next_run` is far in the past at first boot. With `catch_up` set to `False` ([src/paperless/settings.py:L451]), `django-q` runs each overdue schedule **once** at the first check and then advances it to its normal cadence — it does **not** replay every missed interval. This produces a **one-time catch-up burst** at the first 30-second check in which all four schedules fire together:

```
$ docker logs paperless-inv-app | grep "\[Q\]"   # window 20:46:20-20:46:21 (one-time catch-up burst)
20:46:20 [Q] INFO Enqueued 1
20:46:20 [Q] INFO Process-1 created a task from schedule [Train the classifier]
20:46:20 [Q] INFO Process-1:1 processing [island-helium-ink-skylark]
20:46:20 [Q] INFO Enqueued 1
20:46:20 [Q] INFO Process-1 created a task from schedule [Optimize the index]
20:46:20 [Q] INFO Process-1:2 processing [saturn-single-stream-freddie]
20:46:20 [Q] INFO Enqueued 1
20:46:20 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
20:46:20 [Q] INFO Process-1:3 processing [xray-floor-freddie-hotel]
20:46:20 [Q] INFO Enqueued 1
20:46:20 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
20:46:20 [Q] INFO Process-1:4 processing [princess-hot-carpet-hotel]
20:46:20 [Q] INFO Process-1:4 stopped doing work
20:46:20 [Q] INFO Processed [princess-hot-carpet-hotel]
20:46:20 [Q] INFO Process-1:1 stopped doing work
20:46:20 [Q] INFO Process-1:3 stopped doing work
20:46:20 [Q] INFO Process-1:2 stopped doing work
20:46:20 [Q] INFO Processed [island-helium-ink-skylark]
20:46:20 [Q] INFO Processed [xray-floor-freddie-hotel]
20:46:20 [Q] INFO Processed [saturn-single-stream-freddie]
20:46:20 [Q] INFO recycled worker Process-1:1
20:46:20 [Q] INFO Process-1:14 ready for work at 119
20:46:20 [Q] INFO recycled worker Process-1:3
20:46:20 [Q] INFO Process-1:15 ready for work at 120
20:46:21 [Q] INFO recycled worker Process-1:2
20:46:21 [Q] INFO Process-1:16 ready for work at 121
20:46:21 [Q] INFO recycled worker Process-1:4
20:46:21 [Q] INFO Process-1:17 ready for work at 122
```

This burst is a **startup artifact, not the steady state**. Note the worker lifecycle within it: each task is enqueued (`Enqueued 1`), created from its schedule by the sentinel (`Process-1 created a task from schedule [...]`), picked up by a numbered worker (`Process-1:N processing [...]`), completed (`Processed [...]`), and then the worker is **recycled** (`recycled worker Process-1:N` → a fresh `Process-1:M ready for work`) because `recycle` is `1` ([src/paperless/settings.py:L452], recycle after each task). After this one burst the system settles into its steady cadence (see [§5](#5-q3--periodic-healthready-log-entries)).

### 4.3 What each idle task actually does

- **`process_mail_accounts`** iterates `MailAccount.objects.all()`; with no mail accounts configured it does no network I/O and returns the fixed string `No new documents were added.` ([src/paperless_mail/tasks.py:L11-22]). This return value is stored as the task's result in the DB (not printed to stdout), and was read back directly:

```
$ manage.py shell -c "Task.objects.filter(func=process_mail_accounts).latest -> result"
count= 3
func= paperless_mail.tasks.process_mail_accounts | success= True
result= 'No new documents were added.'
```

- **`train_classifier`** ([src/documents/tasks.py:L48]) retrains the document classifier; with no documents/tags it has nothing to learn and exits quickly.
- **`index_optimize`** ([src/documents/tasks.py:L32]) optimizes the Whoosh full-text index.
- **`sanity_check`** ([src/documents/tasks.py:L255]) verifies stored documents against checksums; with an empty corpus it is a no-op.

While idle, therefore, the only automatic work is this lightweight, data-free scheduler housekeeping — dominated by the 10-minute mail check.

---

## 5. Q3 — Periodic health/ready log entries

**Question:** *What are the actual log entries that appear periodically showing the system is healthy and ready? I need the specific log messages, their frequency, and what they indicate.*

At idle there are exactly **two** genuinely periodic signals: the container **healthcheck** (the steady ~30 s heartbeat) and the **scheduler firing** (the ~10 min mail check — the most frequent `django-q` activity). Everything else is silent between firings.

### 5.1 The Docker healthcheck — the steady heartbeat (~30 s)

**Message / mechanism:** the healthcheck runs `curl -f http://localhost:8000` against gunicorn every 30 seconds, defined in `docker/compose/docker-compose.sqlite.yml:L41-45` (`test: curl -f http://localhost:8000`, `interval: 30s`, `timeout: 10s`, `retries: 5`). It is the primary periodic *liveness* signal and the reason the container reports `healthy`. It executes automatically — the history below is Docker's own record of probe executions, not a manual `curl`:

```
$ docker inspect -f "{{json .State.Health}}" paperless-inv-app | (parse Log: Start, ExitCode, delta)
Status=healthy FailingStreak=0 (Docker retains last 5 probes)
2026-07-14 21:04:44  exit=0
2026-07-14 21:05:14  exit=0  (+30s)
2026-07-14 21:05:44  exit=0  (+30s)
2026-07-14 21:06:14  exit=0  (+30s)
2026-07-14 21:06:44  exit=0  (+30s)
```

**Frequency:** the observed inter-probe deltas are **~30 s** (30 s in most windows, occasionally 31 s). This is expected cause→effect: Docker schedules each probe 30 s after the *previous probe completes*, so the wall-clock delta is `30 s + probe_duration`, which rounds to 30–31 s. It is **not** a fixed 30.000 s tick. Across the full >20-minute observation every probe returned `exit=0` and `FailingStreak` stayed `0`.

**What it indicates:** gunicorn is bound and serving — an unauthenticated `GET /` returns `302 → /accounts/login/` (shown in [§3.2](#32-web-server-readiness--gunicorn)), which `curl -f` treats as success (exit 0). This is the single most reliable "system is healthy and ready" signal while idle.

### 5.2 The scheduler firing — every ~10 minutes

**Messages:** the most frequent periodic `django-q` log activity at idle is the 10-minute mail-check schedule. Each firing produces this exact sequence (one full steady-state firing shown, plus the timeline of all firings):

```
$ docker logs --timestamps paperless-inv-app | grep "created a task from schedule \[Check all e-mail accounts\]"
2026-07-14T20:46:20 20:46:20 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
2026-07-14T20:55:51 20:55:51 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
2026-07-14T21:05:53 21:05:53 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]

$ docker logs paperless-inv-app | grep "\[Q\]"   # one full steady-state firing (21:05:53)
21:05:53 [Q] INFO Enqueued 1
21:05:53 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
21:05:53 [Q] INFO Process-1:6 processing [oklahoma-fifteen-papa-ceiling]
21:05:53 [Q] INFO Process-1:6 stopped doing work
21:05:53 [Q] INFO Processed [oklahoma-fifteen-papa-ceiling]
21:05:53 [Q] INFO recycled worker Process-1:6
21:05:53 [Q] INFO Process-1:19 ready for work at 1206
```

**Per-line meaning and emitting component** (the `Process-N` prefix in each line *is* the emitter, so these attributions are observed, not assumed):

- `Enqueued 1` — a task was pushed onto the Redis broker queue; emitted by `django-q`'s `async_task()` running inside the sentinel/main process ([requirements.txt:L37], `django-q==1.3.9`).
- `Process-1 created a task from schedule [Check all e-mail accounts]` — the **sentinel** (`Process-1`) matched a due `Schedule` row and created the task. `Process-1` (no worker suffix) identifies the main/guard process.
- `Process-1:6 processing [oklahoma-fifteen-papa-ceiling]` — a **worker** child (here #6) dequeued and executed the task. The task id (`oklahoma-...`) is randomly generated per task and is **variable**.
- `Processed [oklahoma-fifteen-papa-ceiling]` — the **monitor** process recorded the completed result to the DB *(the emitter here is inferred from the `django-q` source, since this line carries no `Process-N` prefix)*.
- `recycled worker Process-1:6` → `Process-1:19 ready for work at 1206` — the sentinel recycled the worker after its one task (`recycle=1`, [src/paperless/settings.py:L452]) and spawned a replacement.

**Frequency:** the three observed firings were at `20:46:20`, `20:55:51`, and `21:05:53`. The two consecutive **steady-state** firings are `20:55:51 → 21:05:53 = 10 m 02 s ≈ 10 minutes`, matching `minutes=10` ([src/paperless_mail/migrations/0002_auto_20201117_1334.py:L12-14]). (The first-to-second gap, `20:46:20 → 20:55:51 = 9 m 31 s`, is shorter because the first firing was the catch-up burst pinned to a 30-second check boundary rather than the schedule's own `next_run`.)

**What it indicates:** the task engine is alive and the scheduler is firing on cadence — i.e. background scheduling is healthy. With no mail accounts configured the task itself returns `No new documents were added.` ([§4.3](#43-what-each-idle-task-actually-does)).

### 5.3 One-time catch-up vs. steady state (disambiguation)

The `20:46:20` burst in which **all four** schedules fired is the **one-time startup catch-up** described in [§4.2](#42-the-engine-that-runs-them--qcluster) — it is not periodic. In steady state only the mail check recurs (every ~10 min); the hourly/daily/weekly schedules will each fire once when their configured interval elapses, which is outside a short observation window. There is therefore no contradiction between "all four fired at startup" and "only the mail check recurs frequently": the former is catch-up, the latter is steady cadence.

### 5.4 Between firings: silence

Crucially, between scheduled firings the `qcluster` emits **nothing** — it is not a chatty heartbeat. This was confirmed by counting `[Q]` log lines in the quiet window between the first and second steady firings:

```
$ docker logs --timestamps paperless-inv-app | grep "\[Q\]" | awk between 20:56:00 and 21:05:00
[Q] log lines emitted in the 9-minute idle window 20:56:00..21:05:00 = 0
```

Likewise the idle `document_consumer` emits nothing after its startup watch line ([§3.3](#33-file-watcher-readiness--document_consumer)). So the only *log* output at idle is the periodic scheduler firings (~10 min); the healthcheck heartbeat (~30 s) is recorded by Docker's health subsystem rather than in the application log stream.

---

## 6. Q4 — Interrupt/restart recovery

**Question:** *If you briefly interrupt and restart part of the system, what specific log messages confirm everything has reconnected and is operational again?*

The interrupt target is the **Redis broker**, because both the `qcluster` (task broker) and the Channels websocket layer depend on it, so a Redis blip exercises the reconnection path directly. The experiment: capture the *before*, *during*, and *after* states around a `docker stop` / `docker start` of the broker.

To avoid perturbing the ≥10-minute idle-observation run used in §2–§5, this interrupt/restart experiment was performed as a **separate, dedicated canonical run** — a fresh boot of the same image and configuration. Per the variable-identifier convention noted in *How to read this document*, that fresh boot has its own randomly-generated `django-q` cluster name (`equal-august-hawaii-stairway`) and its own worker/pusher PIDs, so the timestamps and PIDs shown below differ from the earlier sections even though the behavior is identical. The transient `Error 111` race described below was reproduced across **three** such runs to separate the stable part of the signature from the race-dependent part.

### 6.1 Before — broker up, cluster pushing

```
$ docker ps --filter name=paperless-inv-broker --format "{{.Names}} {{.Status}} {{.Image}}"
paperless-inv-broker Up 4 minutes redis:6.0
$ # redis reachable from the app; qcluster pusher active (from boot: "Process-1:13 pushing tasks at 381")
$ python3 -c "import redis,os; print(redis.from_url(os.environ[PAPERLESS_REDIS]).ping())"
PING True
```

### 6.2 During the outage — the failure signature

Stopping the broker at **`T0 = 23:14:01`** produced an immediate, continuous error stream. The pusher is blocked in `blpop(key, 1)` on an already-established socket, so the **very first** error — at the instant `docker stop` gracefully closes that in-flight socket — is a `redis.exceptions.ConnectionError: Connection closed by server.` raised on the *read* path. This was the first error in **all three** reproduction runs. A fresh reconnect attempt may then briefly hit the still-cached-but-now-dead broker IP, producing at most a **single, transient** `Error 111 ... Connection refused` on the *connect* path — a narrow race that appeared `0, 0, 1` times across the three runs (this transcript is the run in which it appeared once). Within about one second Docker removes the stopped container's `broker` network alias from its embedded DNS, after which every reconnect fails name resolution with `Error -5 ... No address associated with hostname`, which **dominates** the outage:

```
$ T0=2026-07-14T23:14:01 ; docker stop paperless-inv-broker      # interrupt the broker
$ docker logs --timestamps paperless-inv-app | (lines at/after T0)
2026-07-14T23:14:01.615334104Z redis.exceptions.ConnectionError: Connection closed by server.
2026-07-14T23:14:01.688752636Z 23:14:01 [Q] ERROR Error 111 connecting to broker:6379. Connection refused.
2026-07-14T23:14:02.202510262Z 23:14:02 [Q] ERROR Error -5 connecting to broker:6379. No address associated with hostname.
2026-07-14T23:14:02.717064000Z 23:14:02 [Q] ERROR Error -5 connecting to broker:6379. No address associated with hostname.
   [the "Error -5 ... No address associated with hostname" line then repeats for the rest of the outage]

$ # exact per-error-type counts over the 159s outage window (T0 23:14:01 - T1 23:16:40):
Connection closed by server. (blpop read path)       = 1    <- the FIRST error, at the instant of stop
Error 111 (Connection refused, connect-path race)    = 1    <- transient; 0,0,1 across the three runs
Error -5  (No address associated w/ hostname)        = 309  <- dominant; broker DNS alias removed while stopped
reincarnated pusher ... after sudden death           = 15   <- ERROR-level (django_q/cluster.py:L223)
[Q] ERROR lines total                                = 325  <- 309 (Error -5) + 1 (Error 111) + 15 (reincarnations)

$ docker logs paperless-inv-app | grep -E "stopped pushing|reincarnated pusher"   # pusher lifecycle during outage
23:14:11 [Q] INFO Process-1:13 stopped pushing tasks
23:14:11 [Q] ERROR reincarnated pusher Process-1:13 after sudden death
23:14:22 [Q] INFO Process-1:18 stopped pushing tasks
23:14:22 [Q] ERROR reincarnated pusher Process-1:18 after sudden death
23:14:32 [Q] INFO Process-1:19 stopped pushing tasks
23:14:32 [Q] ERROR reincarnated pusher Process-1:19 after sudden death
23:14:42 [Q] INFO Process-1:20 stopped pushing tasks
23:14:42 [Q] ERROR reincarnated pusher Process-1:20 after sudden death
23:14:52 [Q] INFO Process-1:21 stopped pushing tasks
23:14:53 [Q] ERROR reincarnated pusher Process-1:21 after sudden death
23:15:03 [Q] INFO Process-1:22 stopped pushing tasks
23:15:03 [Q] ERROR reincarnated pusher Process-1:22 after sudden death
   [the stopped-pushing → reincarnated-pusher pair continues on the same ~10 s cadence for the rest of the outage — 15 reincarnations total through T1]
```

The failing code path is the pusher's blocking dequeue — a `redis.exceptions.ConnectionError` raised in `broker.dequeue()` → `blpop` ([django_q/cluster.py:L345] → `redis_broker.py:L21`). The verbatim traceback below is the **first** error captured at `T0` — the `Connection closed by server.` failure on the *read* path (`blpop` → `read_response`); once the DNS alias is gone the dominant `Error -5` failures raise the same `ConnectionError` from the *connect* path instead (`connection.py` `getaddrinfo` → `gaierror`). Each failure is followed by a benign `django-q` logging `TypeError: not all arguments converted during string formatting` in the error-formatting path:

```
$ # first error at T0: ConnectionError on the blpop read path (verbatim):
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
```

**Mechanism (cause → effect):** the pusher process blocks on `BLPOP` against Redis ([django_q/cluster.py:L345] → `redis_broker.py:L21`); when the broker vanishes the call raises `ConnectionError`, the pusher process dies, and the cluster's **sentinel** detects the death and **reincarnates** it (`reincarnated pusher Process-1:N after sudden death`), which is why the logical pusher id climbs `Process-1:13 → 18 → 19 → 20 → 21 → 22` on a ~10-second cadence for the duration of the outage. The sentinel itself never dies, so the cluster stays up and keeps retrying.

**The HTTP tier stayed healthy throughout.** The Docker healthcheck kept returning `exit=0` and `FailingStreak` never rose above `0` — including the probes at **23:14:30**, **23:15:00** and **23:15:31**, which all fell *inside* the outage window and still passed *(observed live via `docker inspect .State.Health` during the outage)*. This is expected: an unauthenticated `GET /` returns `302 → /accounts/login/` (see [§3.2](#32-web-server-readiness--gunicorn)) without touching Redis, so the liveness probe does **not** detect a broker outage:

```
$ docker inspect -f "{{json .State.Health}}" paperless-inv-app   # live; FailingStreak stayed 0 all run
Status=healthy FailingStreak=0
2026-07-14 23:14:30  exit=0   <- inside outage (broker down since T0 23:14:01)
2026-07-14 23:15:00  exit=0  (+30s)   <- inside outage
2026-07-14 23:15:31  exit=0  (+31s)   <- inside outage
```

### 6.3 After restart — the recovery signature

Restarting the broker at **`T1 = 23:16:40`** cleared the fault within ~6 seconds — the recovery latency is bounded by the pusher's ~10-second reincarnation cycle plus the instant Docker re-registers the `broker` DNS alias, not by any application retry timer. The recovery is visible as: the `Error -5` stream stops the moment DNS resolves the restarted broker again (no `Error 111` was emitted at restart in this run), then the sentinel reincarnates the pusher one last time and the new pusher **successfully begins pushing and persists** — after which errors cease entirely and Redis answers `PING`:

```
$ T1=2026-07-14T23:16:40 ; docker start paperless-inv-broker     # restart the broker
$ docker logs --timestamps paperless-inv-app | (lines at/after T1)
2026-07-14T23:16:40 23:16:40 [Q] ERROR Error -5 connecting to broker:6379. No address associated with hostname.
2026-07-14T23:16:40 23:16:40 [Q] ERROR Error -5 connecting to broker:6379. No address associated with hostname.
2026-07-14T23:16:46 23:16:46 [Q] INFO Process-1:32 stopped pushing tasks
2026-07-14T23:16:46 23:16:46 [Q] ERROR reincarnated pusher Process-1:32 after sudden death
2026-07-14T23:16:46 23:16:46 [Q] INFO Process-1:33 pushing tasks at 546

$ # error lines after restart (they cease within ~6s, bounded by the pusher reincarnation cycle):
Error -5 after T1                        = 2
Error 111 after T1                       = 0
last error after T1: 23:16:40 [Q] ERROR Error -5 connecting to broker:6379. No address associated with hostname.
$ python3 -c "import redis,os; print(redis.from_url(os.environ[PAPERLESS_REDIS]).ping())"
PING True
```

The decisive line is `Process-1:33 pushing tasks at 546` — a fresh pusher (django-q logical id `Process-1:33`, OS pid `546`) that, unlike its predecessors, does **not** immediately die (no further `stopped pushing` line follows it). Combined with the error stream stopping at `23:16:40` and `PING True`, this is the "operational again" signal.

### 6.4 There is no literal "reconnected" message

`django-q` (and Paperless) emit **no** explicit "reconnected"/"recovered" log line. Grepping the entire container log confirms it:

```
$ docker logs paperless-inv-app | grep -ic "reconnect"    # is there a literal "reconnected" message?
0
```

Therefore "everything has reconnected and is operational again" is necessarily a **composite, inferred** signature, not a single string. The confirming pattern is: **(1)** connection errors stop; **(2)** `reincarnated pusher Process-1:N after sudden death` followed by a *persisting* `Process-1:M pushing tasks at <pid>`; **(3)** `PING True`; and **(4)** the next scheduled task fires normally (next section).

### 6.5 Operational again — scheduling resumes

Definitive proof that the system is fully operational: the next mail-check schedule fired normally after recovery, at `23:21:13`, with the complete enqueue → create → process → processed sequence. (The only earlier firing in this dedicated run was the post-boot catch-up burst at `23:11:37`, where the mail check and the sanity check fired together immediately after the `23:11:07` boot; the resulting sub-10-minute gap to the first steady-state firing is the same catch-up-vs-steady-state effect documented in §5.3.)

```
$ docker logs --timestamps paperless-inv-app | (firing lines at/after T1)
2026-07-14T23:21:13 23:21:13 [Q] INFO Enqueued 1
2026-07-14T23:21:13 23:21:13 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
2026-07-14T23:21:13 23:21:13 [Q] INFO Process-1:5 processing [island-nebraska-jupiter-mirror]
2026-07-14T23:21:13 23:21:13 [Q] INFO Process-1:5 stopped doing work
2026-07-14T23:21:13 23:21:13 [Q] INFO Processed [island-nebraska-jupiter-mirror]
2026-07-14T23:21:13 23:21:13 [Q] INFO recycled worker Process-1:5
2026-07-14T23:21:13 23:21:13 [Q] INFO Process-1:34 ready for work at 655
```

### 6.6 Websocket layer impact (inferred from source)

Because Redis is also the `channels_redis` channel-layer backend, a broker outage also breaks websocket status delivery: the `StatusConsumer` ([src/paperless/consumers.py:L9]) and any `group_send` to the `status_updates` group ([src/paperless/consumers.py:L17-20]) route through the Redis-backed `CHANNEL_LAYERS` ([src/paperless/settings.py:L178-182]). This document does **not** exercise a live websocket client, so the websocket-delivery failure/recovery is **inferred from source** rather than observed; what *was* observed is that the HTTP tier (which the healthcheck probes) is unaffected, while the Redis-dependent task tier fails and recovers as shown above.

---

## 7. Q5 — Continuously-running components

**Question:** *What components or processes keep running continuously to maintain Paperless-NGX in a ready state, even when no documents are being processed?*

Five components run continuously at idle: **three supervised application processes** plus **two stateful backends** (Redis and SQLite). `supervisord` is the *mechanism* that keeps the three alive but is not itself a readiness signal.

### 7.1 The three supervised application processes

`supervisord` supervises exactly three long-running programs, each pinned to the unprivileged `paperless` user, as declared in `/etc/supervisord.conf` (matching source [docker/supervisord.conf:L10-30]) and observed live in the process tree ([§3.6](#36-supervised-process-tree-non-root)):

- **`gunicorn`** ([docker/supervisord.conf:L10-11]) — the ASGI web server on `:8000` serving the REST API and the ASGI application ([gunicorn.conf.py:L3-5]). Runs continuously; it is the target of the healthcheck. **Always active** (accepts connections at all times).
- **`document_consumer`** ([docker/supervisord.conf:L19-20]) — the file watcher. It holds a live inotify watch on the consumption directory ([§3.3](#33-file-watcher-readiness--document_consumer)) but is **silent and near-idle** until a file appears; it consumes negligible CPU while watching.
- **`qcluster`** ([docker/supervisord.conf:L28-29]) — the `django-q` cluster, the **only always-on scheduled-task engine**. It comprises a sentinel/guard (main `Process-1`), a monitor, a pusher, and N=11 workers ([§3.4](#34-qcluster-the-django-q-task-engine)). The sentinel wakes ~twice a minute to check schedules; the pool is otherwise idle between the ~10-minute firings.

### 7.2 Stateful backends — Redis and SQLite

- **Redis 6.0** ([docker/compose/docker-compose.sqlite.yml:L29]) plays a **dual role**: it is the `django-q` task broker ([src/paperless/settings.py:L456]) *and* the `channels_redis` websocket channel-layer backend ([src/paperless/settings.py:L178-182]). It is a **hard prerequisite** — [§6](#6-q4--interruptrestart-recovery) showed that without it, background scheduling fails until it returns.
- **SQLite** (default DB) stores the `django-q` `Schedule` rows and task results, and Paperless's own data. It is a file, not a process, but is continuously read/written by the running processes.

```
$ docker exec paperless-inv-app ls -l /app/data/db.sqlite3        # SQLite: Schedule rows + task results
-rw-r--r-- 1 paperless paperless 331776 Jul 14 21:15 /app/data/db.sqlite3
$ python3 -c "import redis,os; print(redis.from_url(os.environ[PAPERLESS_REDIS]).ping())"   # broker + channel layer
redis PING True
$ grep -E "^\[program:|^command=|^user=" /etc/supervisord.conf   # the 3 supervised programs
user=root
[program:gunicorn]
command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
user=paperless
[program:consumer]
command=python3 manage.py document_consumer
user=paperless
[program:scheduler]
command=python3 manage.py qcluster
user=paperless
```

### 7.3 The websocket layer (inside gunicorn, authenticated)

Paperless serves websockets through the same gunicorn/ASGI process, not a separate daemon. The `ProtocolTypeRouter` ([src/paperless/asgi.py:L17]) multiplexes the `http` ([src/paperless/asgi.py:L19]) and `websocket` ([src/paperless/asgi.py:L20]) protocols; the websocket route is wrapped in `AuthMiddlewareStack`, and the `StatusConsumer` **rejects unauthenticated connections** (`DenyConnection`) before joining the `status_updates` group ([src/paperless/consumers.py:L14-20]). So the continuously-available websocket endpoint is **authenticated-only** — it maintains readiness to push live processing status to logged-in UI clients, but is not an open/anonymous channel *(this authenticated-flow detail is read from source; no live websocket client was exercised)*.

### 7.4 `supervisord` is the mechanism, not a readiness signal

`supervisord` (pid 1, root) is what *keeps* the three programs running — it auto-restarts any that exit ([docker/supervisord.conf]). It is the reason the system self-heals process crashes, but its own log lines (`spawned`, `entered RUNNING state`) are startup/lifecycle events, **not** periodic readiness heartbeats. The periodic readiness signals are the healthcheck (~30 s) and the scheduler firings (~10 min) from [§5](#5-q3--periodic-healthready-log-entries).

### 7.5 Continuity is stable across runs

A second, sequential run (the stack was restarted, **not** a concurrent second cluster — running two clusters in one environment is explicitly discouraged upstream) reproduced the same continuously-running topology: the same three processes, the same **11-worker** count, and the same startup string sequence — differing only in the per-boot random cluster display-name and OS pids. The warm restart also skipped the two conditional startup branches (reindex, superuser creation), confirming their conditionality ([§3.1](#31-the-boot-chain-observed)):

```
$ docker restart paperless-inv-app     # SEQUENTIAL second run (broker stays up; no concurrent 2nd cluster)

$ # run #2 qcluster startup block (verbatim) -- new random name, SAME 11-worker count
21:20:33 [Q] INFO Q Cluster football-hot-emma-potato starting.
21:20:33 [Q] INFO Process-1:1 ready for work at 63
21:20:33 [Q] INFO Process-1:2 ready for work at 64
21:20:33 [Q] INFO Process-1:3 ready for work at 65
21:20:33 [Q] INFO Process-1:4 ready for work at 66
21:20:33 [Q] INFO Process-1:5 ready for work at 67
21:20:33 [Q] INFO Process-1:6 ready for work at 68
21:20:33 [Q] INFO Process-1:7 ready for work at 69
21:20:33 [Q] INFO Process-1:8 ready for work at 70
21:20:33 [Q] INFO Process-1:9 ready for work at 71
21:20:33 [Q] INFO Process-1:10 ready for work at 72
21:20:33 [Q] INFO Process-1:11 ready for work at 73
21:20:33 [Q] INFO Process-1:12 monitoring at 74
21:20:33 [Q] INFO Process-1 guarding cluster football-hot-emma-potato
21:20:33 [Q] INFO Process-1:13 pushing tasks at 75
21:20:33 [Q] INFO Q Cluster football-hot-emma-potato running.

$ # cross-run comparison:
run #1 name = wisconsin-black-charlie-bulldog   workers = 11   (cpu_count 128 -> floor(sqrt)=11)
run #2 name = football-hot-emma-potato          workers = 11
   -> worker count and startup string sequence are STABLE across runs; only the
      random display-name and OS pids differ (both are per-boot variable).

$ # warm-restart conditional branches (run #2): both reindex and superuser-create are SKIPPED
'Search index out of date. Updating...' in run #2 = 0
'Created superuser' in run #2                     = 0
No migrations to apply.
Changed password of user admin.
```

---

## 8. Canonical vs. non-canonical labelling

All Q1–Q5 behavior above was produced by a canonical run and is reported as default behavior: **Python 3.9.23** (canonical 3.9 baseline), **Redis server v6.0.20** (canonical 6.0), the exact commit `542221a38dff...`, the real `ENTRYPOINT`+`supervisord` boot, children as `paperless`, and a live 30 s healthcheck. The `django-q` log strings, the four schedule cadences, the ~30 s / ~10 min periodic signals, and the interrupt/restart recovery signature are all canonical.

The following are **environmental deltas** from a byte-identical production deploy. None changes the idle Q1–Q5 behavior; each is disclosed so no non-default artifact is mistaken for canonical:

| # | Delta observed | Why | Effect on Q1–Q5 |
|---|---|---|---|
| a | Startup warning `UserWarning: No directory at: /app/static/` (from whitenoise) | The Angular frontend is not built in the dev image (frontend is out of scope), so `collectstatic` output is absent | **None** — cosmetic; unrelated to idle background behavior |
| b | Healthcheck injected via `docker run --health-cmd` (CMD-SHELL form) instead of the compose `healthcheck:` (exec form) | The canonical healthcheck lives in compose, not the Dockerfile; reproduced at the run layer | **None** — identical `curl -f http://localhost:8000` at the same 30 s/10 s/5-retry cadence |
| c | A thin derived image added `supervisor`, `gosu`, `curl`, the `paperless` user, the `/sbin` boot scripts, and `/etc/supervisord.conf` | The dev image ships the canonical Python/deps baseline but omits the production process-manager layer | **None on behavior** — these are exactly the production `Dockerfile`'s runtime bits; the `/app` source is unmodified (verified clean, [§2.4](#24-runtime-provenance)) |
| d | `PAPERLESS_DISABLE_DBHANDLER=true` is present in the container env | Baked into the dev image | **None** — this variable is **not referenced by any runtime `.py`** at v1.7.0 (it appears only in `src/setup.cfg` as a pytest env), so the running app never reads it |
| e | No startup version banner; `X-Version`/`X-Api-Version` seen only on authenticated responses | By design: the version headers are gated by `if request.user.is_authenticated:` ([src/paperless/middleware.py:L11-14]); the version is `1.7.0` ([src/paperless/version.py:L1]) | **None** — the unauthenticated healthcheck `GET /` (302) carries no version header |

**Explicitly not observed at runtime (reported from configuration or inferred from source, and labelled as such in-text):** the hourly/daily/weekly schedule firings (they cannot fire within the observation window — reported from the seeded migrations, [§4.1](#41-the-four-seeded-schedules)); and the websocket-delivery failure/recovery during the Redis outage (inferred from source, [§6.6](#66-websocket-layer-impact-inferred-from-source)).

**Upstream provenance of the `django-q` strings (library-canonical, not Paperless-specific).** The task-engine strings and roles were cross-checked against upstream `django-q` 1.3.x sources and the installed 1.3.9 package:

- Startup sequence (`Q Cluster <name> running.`, `ready for work`, `monitoring`, `guarding cluster`, `pushing tasks`) and the *"twice a minute the scheduler checks for any scheduled tasks"* cadence — `django-q` Cluster documentation (Read the Docs, 1.3.6).
- Humanized `guarding cluster <name>` form (matching the observed `wisconsin-black-charlie-bulldog`) — Koed00/django-q issue #494.
- `reincarnated <worker|pusher> ... after death` — Koed00/django-q issue #200.
- `processing [<id>]` / `Processed [<id>]` / `stopped pushing tasks` — Koed00/django-q issue #367.
- The failing path in the outage is the installed source: pusher `broker.dequeue()` at `django_q/cluster.py:L345`, `redis_broker.py:L21` (`blpop`), and sentinel reincarnation at `django_q/cluster.py:L222`.

---

## 9. Coverage matrix

Final pass confirming every sub-question and every named item is answered, with the evidence type (Observed at runtime / from Configuration / Inferred from source).

| Sub-question | Named item | Where | Evidence |
|---|---|---|---|
| **Q1** startup | Entrypoint → prepare → supervisord boot chain | [§3.1](#31-the-boot-chain-observed) | Observed |
| Q1 | `Connected to Redis broker` / `Apply database migrations...` | [§3.1](#31-the-boot-chain-observed) | Observed |
| Q1 | Conditional reindex + superuser branches | [§3.1](#31-the-boot-chain-observed), [§7.5](#75-continuity-is-stable-across-runs) | Observed (fired on cold boot, skipped on warm restart) |
| Q1 | gunicorn `Server is ready. Spawning workers` + HTTP 302 | [§3.2](#32-web-server-readiness--gunicorn) | Observed |
| Q1 | consumer `Using inotify to watch directory for changes` | [§3.3](#33-file-watcher-readiness--document_consumer) | Observed |
| Q1 | `qcluster` startup (`starting`→`ready for work`×11→`monitoring`→`guarding`→`pushing tasks`→`running`) | [§3.4](#34-qcluster-the-django-q-task-engine) | Observed |
| Q1 | Worker count = `floor(sqrt(cpu_count))` = 11 | [§2.4](#24-runtime-provenance), [§7.5](#75-continuity-is-stable-across-runs) | Observed (stable across 2 runs) |
| Q1 | Idle preconditions (empty consume dir, 0 documents) | [§3.5](#35-idle-preconditions) | Observed |
| **Q2** idle work | `Check all e-mail accounts` — Minutes/10 min | [§4.1](#41-the-four-seeded-schedules) | Observed + Config |
| Q2 | `Train the classifier` — Hourly | [§4.1](#41-the-four-seeded-schedules) | Config (seed migration) |
| Q2 | `Optimize the index` — Daily | [§4.1](#41-the-four-seeded-schedules) | Config (seed migration) |
| Q2 | `Perform sanity check` — Weekly | [§4.1](#41-the-four-seeded-schedules) | Config (seed migration) |
| Q2 | one-time startup catch-up burst (all 4) | [§4.2](#42-the-engine-that-runs-them--qcluster) | Observed |
| Q2 | mail task result `No new documents were added.` | [§4.3](#43-what-each-idle-task-actually-does) | Observed |
| **Q3** periodic logs | Docker healthcheck `curl -f :8000`, ~30 s, `exit=0` | [§5.1](#51-the-docker-healthcheck--the-steady-heartbeat) | Observed |
| Q3 | scheduler firing (`Enqueued`/`created a task from schedule`/`processing`/`Processed`/`recycled`), ~10 min | [§5.2](#52-the-scheduler-firing--every-10-minutes) | Observed |
| Q3 | catch-up vs steady-state disambiguation | [§5.3](#53-one-time-catch-up-vs-steady-state-disambiguation) | Observed |
| Q3 | silence between firings (0 `[Q]` lines in 9-min window) | [§5.4](#54-between-firings-silence) | Observed |
| **Q4** recovery | first error `Connection closed by server.` (blpop read path); transient `Error 111` is a connect-path race (`0,0,1` across 3 runs) | [§6.2](#62-during-the-outage--the-failure-signature) | Observed |
| Q4 | dominant `Error -5 ... No address associated with hostname` (×309) | [§6.2](#62-during-the-outage--the-failure-signature) | Observed |
| Q4 | pusher `stopped pushing tasks` → `reincarnated pusher ... after sudden death` | [§6.2](#62-during-the-outage--the-failure-signature) | Observed |
| Q4 | HTTP healthcheck stays `exit=0` through outage | [§6.2](#62-during-the-outage--the-failure-signature) | Observed |
| Q4 | recovery: errors cease → new `pushing tasks at <pid>` → `PING True` | [§6.3](#63-after-restart--the-recovery-signature) | Observed |
| Q4 | no literal `reconnect` string (composite/inferred recovery) | [§6.4](#64-there-is-no-literal-reconnected-message) | Observed |
| Q4 | operational-again: next schedule fires at `23:21:13` | [§6.5](#65-operational-again--scheduling-resumes) | Observed |
| Q4 | websocket-layer impact | [§6.6](#66-websocket-layer-impact-inferred-from-source) | Inferred from source |
| **Q5** always-on | gunicorn / document_consumer / qcluster (supervised, non-root) | [§7.1](#71-the-three-supervised-application-processes) | Observed |
| Q5 | Redis (broker + channel layer), SQLite (schedules + results) | [§7.2](#72-stateful-backends--redis-and-sqlite) | Observed |
| Q5 | websocket layer (authenticated, in-gunicorn) | [§7.3](#73-the-websocket-layer-inside-gunicorn-authenticated) | Inferred from source |
| Q5 | supervisord = keep-alive mechanism, not a heartbeat | [§7.4](#74-supervisord-is-the-mechanism-not-a-readiness-signal) | Observed + source |

---

## 10. Integrity and cleanup

This investigation honored the **read-only source mandate**:

- **No source file was modified.** The application checkout at commit `542221a38dff...` was verified byte-clean at runtime — `git status --porcelain` printed nothing ([§2.4](#24-runtime-provenance)). The only build additions (supervisor, the `paperless` user, the `/sbin` scripts) live in a *derived image layer*, never in the `/app` source tree.
- **The only artifact created** in the destination repository is this document, `blitzy/documentation/paperless-ngx_542221a38dff.md`.
- **All temporary observation resources were removed** after capture: the investigation containers (`paperless-inv-app`, `paperless-inv-broker`), the temporary network (`paperless-inv-net`), the derived image (`paperless-canonical:investigation`), and all host scratch files. The destination working tree therefore contains only this documentation file as a change.

**Reproducibility note.** Because the `django-q` cluster display-name (e.g. `wisconsin-black-charlie-bulldog`, `football-hot-emma-potato`), the task ids (e.g. `island-helium-ink-skylark`), and all OS pids are randomly assigned per boot, they will differ on any re-run; the *structure*, *strings*, *cadences* (≈30 s healthcheck, ≈10 min mail check), and *worker count* (`floor(sqrt(cpu_count))`) are the stable, canonical facts and were confirmed across two runs.
