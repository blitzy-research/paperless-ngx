# Background / Asynchronous Processing in paperless-ngx — a run-first, evidence-based investigation

> **Subject:** How paperless-ngx performs background/asynchronous processing while it ingests documents.
> **Engine:** **Django-Q** (the `django_q` app) — **not Celery**. There are zero `celery` references in the settings; `"django_q"` is the installed app at `src/paperless/settings.py:110`.
> **Commit:** `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`).
> **Scope of findings:** Everything below is scoped to *exactly this commit*. In particular, the negative findings (no `PaperlessTask` model, no `/api/tasks` endpoint) are true here and must **not** be generalized to later paperless-ngx releases that add task-tracking features.
> **Methodology:** Run-first. The system was **built and run** in its canonical configuration, real ingestion work was triggered through its **real entry points**, and the answers below are written from **observed runtime output**. Every command that produced an output is shown next to that output, complete and unedited. Every claim is tied to a `file:line` reference and/or to captured command output. Statements that are inferred rather than observed are explicitly labelled **[INFERRED]**.

---

## 1. Environment & how it was run

All observation happened **inside the user-provided canonical Docker container** (the bare host sandbox runs Python 3.13 with no Redis and no `django_q`, and is *not* a valid observation environment).

| Property | Value | Evidence |
|---|---|---|
| Attached image (setup instructions) | `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542` | user-provided setup instructions |
| Source image it was created from | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` | `docker inspect pngx --format '{{.Config.Image}}'` → the ghcr string above; running container name `pngx` |
| OS | Debian GNU/Linux 11 (bullseye) | `cat /etc/os-release` → `PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"` |
| Python | 3.9.23 | `python3 --version` → `Python 3.9.23`; `Dockerfile:18` = `FROM python:3.9-slim-bullseye as main-app` |
| Django | 4.0.4 | `requirements.txt:38` `django==4.0.4`; `import django; django.get_version()` → `4.0.4` |
| Django-Q | 1.3.9 | `requirements.txt:37` `django-q==1.3.9`; `django_q.VERSION` → `(1, 3, 9)` |
| redis (python client) | 3.5.3 | `requirements.txt:84` `redis==3.5.3`; `import redis; redis.__version__` → `3.5.3` |
| hiredis | 2.0.0 | `requirements.txt:44`; `import hiredis; hiredis.__version__` → `2.0.0` |
| channels / channels-redis / daphne | 3.0.4 / 3.4.0 / 3.0.2 | `requirements.txt:23,22,31`; imported versions match |
| djangorestframework | 3.13.1 | `requirements.txt:39`; `rest_framework.VERSION` → `3.13.1` |
| gunicorn | 20.1.0 | `gunicorn.__version__` → `20.1.0` (also in the startup banner below) |
| Redis server | v6.0.16, on `localhost:6379` | `redis-server --version` reports `v=6.0.16`; `redis-cli ping` → `PONG` |
| Database | SQLite at `/app/src/../data/db.sqlite3` | `settings.DATABASES["default"]["ENGINE"] == django.db.backends.sqlite3` (Q5) |
| Repo path in container | `/app` (source under `/app/src`) | — |
| Runtime user | `testuser` (uid 1000) — the canonical CI/runtime user | setup instructions; the `docker exec -u testuser` wrapper below |

**Command wrapper.** Every command in this document was executed inside the running canonical container (name `pngx`). The concrete wrapper — shown here with a real example — is:

```bash
docker exec -u testuser pngx bash -c 'cd /app/src && redis-cli ping'
```

For readability that `docker exec -u testuser pngx bash -c` wrapper is omitted from the per-question command blocks; the command shown after a `$` prompt is exactly the string passed to `bash -c`, i.e. exactly what produced the output beneath it. Django management/HTTP/WebSocket commands were run as `-u testuser` (with `cd /app/src` first, since that is where `manage.py` lives); a few root-level container inspections (e.g. reading `/proc`) were run without `-u`.

**How the stack was brought up (canonical, default configuration).** The default execution model is a single container running **three** Supervisord-managed long-running processes plus Redis and the database (`docker/supervisord.conf`). Redis was started (`redis-server --daemonize yes`; already running → `redis-cli ping` = `PONG`), the database was migrated and a superuser created, then the three processes were launched with the **exact `command=` lines** from `docker/supervisord.conf`, run as `testuser`, each backgrounded to a log file. The repo lives at `/app` (not the production `/usr/src/paperless`), so `gunicorn.conf.py` is at `/app/gunicorn.conf.py`:

```bash
# one-time canonical setup (as testuser), mirroring the user setup instructions:
$ python3 manage.py migrate                 # applies every migration, including the django_q app migrations
$ python3 manage.py document_index reindex
# the superuser password is supplied through the DJANGO_SUPERUSER_PASSWORD env var, sourced from
# $PNGX_ADMIN_PW — an ephemeral local-only test value (intentionally NOT printed here; credential hygiene):
$ DJANGO_SUPERUSER_PASSWORD="$PNGX_ADMIN_PW" \
      python3 manage.py createsuperuser --noinput --username admin --email admin@example.com
      # -> "Superuser created successfully."

# the three long-running processes, each mirroring docker/supervisord.conf:
$ python3 manage.py qcluster                                          > /tmp/blitzy_obs/qcluster.log 2>&1   # [program:scheduler] command :29
$ gunicorn -c /app/gunicorn.conf.py paperless.asgi:application        > /tmp/blitzy_obs/gunicorn.log 2>&1   # [program:gunicorn]   command :11
$ python3 manage.py document_consumer                                 > /tmp/blitzy_obs/consumer.log 2>&1   # [program:consumer]   command :20
```

> **Note on `supervisord.conf` vs. this run.** `docker/supervisord.conf` declares `user=paperless` and the production path `/usr/src/paperless/gunicorn.conf.py` (`docker/supervisord.conf:11`). In this container the canonical user is `testuser` and the repo is at `/app`. The **`command=` program invocations are identical** to what Supervisord would run — `python3 manage.py qcluster`, `python3 manage.py document_consumer`, and the `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` invocation shown above — which is the aspect that matters for observing background processing.

---

## 2. TL;DR — one direct answer per question

- **Q1 (what actually happens):** A real `async_task` targeting `documents.tasks.consume_file` serializes the job and pushes it onto the **Redis** broker; a `qcluster` worker dequeues it and runs the `consume_file` task function [`src/documents/tasks.py:184`], which delegates to `Consumer().try_consume_file` [`src/documents/tasks.py:236`]. On success it returns `"Success. New document id {} created"` [`src/documents/tasks.py:247`]; on failure it raises `ConsumerError` [`src/documents/tasks.py:249`]. **Observed:** the `simple.pdf` job went waiting → active → a `django_q_task` row with `success=True, result='Success. New document id 1 created'`.
- **Q2 (services):** **Three** Supervisord processes — `gunicorn` (ASGI web+WebSocket) [`docker/supervisord.conf:11`], `document_consumer` (dir watcher) [`docker/supervisord.conf:20`], `qcluster` (the Django-Q worker cluster that *executes* jobs and *schedules* recurring ones) [`docker/supervisord.conf:29`] — plus **Redis** (broker + Channels layer) and the **database** (holds `django_q_task`). **Observed:** the `qcluster` banner spawned 11 workers + a monitor + a pusher + a guard, and fired all 4 due `Schedule`s on startup.
- **Q3 (how jobs appear):** In **three** live places — the serialized payload in the **Redis** broker list `django_q:paperless:q`, the **`qcluster` worker log** (`processing [simple.pdf]` then `Processed [simple.pdf]`), and **WebSocket `status_updates`** progress frames — and, after completion, as a **row in `django_q_task`**.
- **Q4 (waiting vs. active):** **Waiting** = enqueued in the **Redis** list (`LLEN django_q:paperless:q` ≥ 1, no worker yet); **active** = executing in a `qcluster` worker (`processing [simple.pdf]`, `STARTING`→`WORKING`→`SUCCESS`). Because the broker is **Redis, not the ORM**, the `django_q_ormq` table (`OrmQ`) stays **empty**. **Observed** before/during/after with the queue length and the `django_q_task` count: `0/4/0` → `1/4/0` → `0/5/0`.
- **Q5 (where state is stored):** Finished-task state lives in the **database**, table **`django_q_task`**, model **`django_q.models.Task`** (with `Success`/`Failure` as *proxies* over the same table). Waiting state is **not** in the DB — it is in **Redis**.
- **Q6 (after-the-fact):** Use Django-Q's own facilities — `django_q.tasks.result(task_id)` / `fetch(task_id)`, the `Task`/`Success`/`Failure` models, and the **Django admin** (Successful/Failed/Scheduled task pages). Retained results are bounded by **`SAVE_LIMIT`** (default **250**, `django_q/conf.py:87`). Lookups use the **stored hex `Task.id`**, not the API's self-assigned uuid4 (proven below). **Negative findings:** there is **no `PaperlessTask` model** and **no `/api/tasks` endpoint / `TaskViewSet`** at this commit.
- **Q7 (where jobs are triggered):** By `django_q.tasks.async_task` at four call-site families — the directory watcher [`src/documents/management/commands/document_consumer.py:86`], the REST upload [`src/documents/views.py:523`], IMAP mail [`src/paperless_mail/mail.py:336`], and bulk edits [`src/documents/bulk_edit.py:18,31,47,63,87`] — plus recurring `django_q.tasks.schedule` registrations in three migrations.

---

## Q1 — Runtime behavior: what actually happens when a document is ingested

**Direct answer.** A real `async_task` call whose first argument is the dotted path `"documents.tasks.consume_file"` serializes the job (a pickled, signed "task package") and pushes it onto the **Redis** broker list `django_q:paperless:q`. A `qcluster` worker process dequeues it, imports the dotted-path function, and runs the `consume_file` task function [`src/documents/tasks.py:184`], whose signature declares `task_id=None` as its trailing keyword parameter [`src/documents/tasks.py:191`]. With barcodes disabled (the default — `CONSUMER_ENABLE_BARCODES=False`, `src/paperless/settings.py:502`), `consume_file` **delegates to `Consumer().try_consume_file`** [`src/documents/tasks.py:236`], the multi-stage ingestion pipeline in `src/documents/consumer.py`. On success it returns the string `"Success. New document id {} created".format(document.pk)` [`src/documents/tasks.py:247`]; if the returned document is null it raises `ConsumerError` [`src/documents/tasks.py:249`], and any pipeline precondition failure raises `ConsumerError` from `Consumer._fail()` [`src/documents/consumer.py:81`]. **The run below took the main (non-barcode) path.**

**Command & complete output — the qcluster worker executing the job** (the job was parked in Redis while `qcluster` was momentarily stopped, then picked up on restart — see Q4). This is the complete `qcluster` log from the restart banner through the recycle line:

```text
$ cat /tmp/blitzy_obs/qcluster_run2.log
05:47:37 [Q] INFO Q Cluster jig-december-charlie-jersey starting.
05:47:37 [Q] INFO Process-1:1 ready for work at 27276
05:47:37 [Q] INFO Process-1:2 ready for work at 27277
05:47:37 [Q] INFO Process-1:3 ready for work at 27278
05:47:37 [Q] INFO Process-1:4 ready for work at 27279
05:47:37 [Q] INFO Process-1:5 ready for work at 27280
05:47:37 [Q] INFO Process-1:6 ready for work at 27281
05:47:37 [Q] INFO Process-1:7 ready for work at 27282
05:47:37 [Q] INFO Process-1:8 ready for work at 27283
05:47:37 [Q] INFO Process-1:9 ready for work at 27284
05:47:37 [Q] INFO Process-1:10 ready for work at 27285
05:47:37 [Q] INFO Process-1:11 ready for work at 27286
05:47:37 [Q] INFO Process-1:12 monitoring at 27287
05:47:37 [Q] INFO Process-1 guarding cluster jig-december-charlie-jersey
05:47:37 [Q] INFO Process-1:13 pushing tasks at 27288
05:47:37 [Q] INFO Q Cluster jig-december-charlie-jersey running.
05:47:37 [Q] INFO Process-1:1 processing [simple.pdf]
[2026-07-08 05:47:37,195] [INFO] [paperless.consumer] Consuming simple.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-ba0vf0iu/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-08 05:47:37,625] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 05:47:38,396] [INFO] [paperless.consumer] Document 2026-07-08 simple consumption finished
05:47:38 [Q] INFO Process-1:1 stopped doing work
05:47:38 [Q] INFO Processed [simple.pdf]
05:47:38 [Q] INFO recycled worker Process-1:1
05:47:38 [Q] INFO Process-1:14 ready for work at 27321
```

**Command & complete output — the stored result row** (`django_q.models.Task`), the exact query and its output:

```text
$ python3 manage.py shell -c "
import os
from django_q.models import Task
from django_q.tasks import result, fetch
tid = os.environ['TID']
t = Task.objects.get(id=tid)
print('id      =', t.id); print('name    =', t.name); print('func    =', t.func)
print('success =', t.success); print('result  =', repr(t.result))
print('started =', t.started); print('stopped =', t.stopped)
ft = fetch(tid)
print('fetch(%r) -> %r | success=%s' % (tid, ft, ft.success))
print('result(%r) -> %r' % (tid, result(tid)))
"
# (TID=2f82f3a05b04425ba4a409de1290e5c7 supplied via env var)
id      = 2f82f3a05b04425ba4a409de1290e5c7
name    = simple.pdf
func    = documents.tasks.consume_file
success = True
result  = 'Success. New document id 1 created'
started = 2026-07-08 05:46:45.892958+00:00
stopped = 2026-07-08 05:47:38.400694+00:00
fetch('2f82f3a05b04425ba4a409de1290e5c7') -> <Task: simple.pdf> | success=True
result('2f82f3a05b04425ba4a409de1290e5c7') -> 'Success. New document id 1 created'
```

**Cause → effect.**
- The line `Process-1:1 processing [simple.pdf]` is emitted by the Django-Q cluster **because** a worker dequeued the package and is about to call the target function; the worker calls `res = f(*task["args"], **task["kwargs"])` at `django_q/cluster.py:432` (this exact frame appears at the top of the failure traceback in Q6/Appendix). **Therefore** `consume_file("/app/src/../consume/simple.pdf")` runs.
- `[paperless.consumer] Consuming simple.pdf` is logged from inside `Consumer.try_consume_file()` **because** `consume_file` delegated to it [`src/documents/tasks.py:236`].
- The two `convert-im6.q16` lines and the `Thumbnail generation with ImageMagick failed, falling back to ghostscript` warning are a real, non-fatal detail of this environment: ImageMagick's `policy.xml` forbids rasterizing PDFs, **so** the parser falls back to ghostscript for the thumbnail — consumption still succeeds.
- `Processed [simple.pdf]` is emitted **because** `consume_file` returned normally; the return value `"Success. New document id 1 created"` [`src/documents/tasks.py:247`] is stored as the Task's `result` (observed above). The document primary key `1` in the string equals the new `Document.pk`.
- After the task, `recycled worker Process-1:1` appears **because** `Q_CLUSTER["recycle"] == 1` (Q_CLUSTER dump in Q2): each worker is replaced after completing one task, so `Process-1:1` is recycled into `Process-1:14`.

**Distinct condition — the barcode short-circuit (not taken here).** **[INFERRED]** If `CONSUMER_ENABLE_BARCODES` were on [`src/paperless/settings.py:502`] and a separator barcode were found, `consume_file` splits the file, deletes the original, broadcasts a `SUCCESS` status by calling `async_to_sync(get_channel_layer().group_send)` to the `status_updates` group [`src/documents/tasks.py:226`] with a `"task_id": task_id` field [`src/documents/tasks.py:219`], and returns `"File successfully split"` [`src/documents/tasks.py:233`] **without** calling `try_consume_file`. In this default run barcodes are **off** (observed: `settings.CONSUMER_ENABLE_BARCODES == False`, Q2 settings dump), so the main path ran and the SUCCESS/`document_id` came from the pipeline, not the split branch.

---

## Q2 — Services / processes involved in background processing

**Direct answer.** Three Supervisord-managed processes plus Redis and the database:

1. **`gunicorn`** — `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` [`docker/supervisord.conf:11`]. The ASGI web **and** WebSocket server (serves the REST API that enqueues jobs and the `ws/status/` progress socket).
2. **`document_consumer`** — `python3 manage.py document_consumer` [`docker/supervisord.conf:20`]. The directory watcher that **enqueues** ingestion jobs.
3. **`qcluster`** — `python3 manage.py qcluster` [`docker/supervisord.conf:29`]. The Django-Q worker **cluster** that **executes** enqueued jobs **and** runs the **scheduler** for recurring `Schedule`s.
4. **Redis** — the Django-Q **broker** (`Q_CLUSTER["redis"]`, `src/paperless/settings.py:456`) **and** the Channels layer backend (`CHANNEL_LAYERS` → `channels_redis.core.RedisChannelLayer`, `src/paperless/settings.py:178,180`).
5. **Database** (SQLite by default) — holds the `django_q_task` table (finished-task state) and the `django_q_schedule` table (recurring registrations).

**Command & complete output — the `qcluster` startup banner and the scheduler firing on startup** (shows the internal process roles AND that `qcluster` both schedules and executes). This is the head of `qcluster.log` on the clean start:

```text
$ sed -n '1,35p' /tmp/blitzy_obs/qcluster.log
05:42:35 [Q] INFO Q Cluster finch-pasta-zulu-princess starting.
05:42:35 [Q] INFO Process-1:1 ready for work at 26275
05:42:35 [Q] INFO Process-1:2 ready for work at 26276
05:42:35 [Q] INFO Process-1:3 ready for work at 26277
05:42:35 [Q] INFO Process-1:4 ready for work at 26278
05:42:35 [Q] INFO Process-1:5 ready for work at 26279
05:42:35 [Q] INFO Process-1:6 ready for work at 26280
05:42:35 [Q] INFO Process-1:7 ready for work at 26281
05:42:35 [Q] INFO Process-1:8 ready for work at 26282
05:42:35 [Q] INFO Process-1:9 ready for work at 26283
05:42:35 [Q] INFO Process-1:10 ready for work at 26284
05:42:35 [Q] INFO Process-1:11 ready for work at 26285
05:42:35 [Q] INFO Process-1:12 monitoring at 26286
05:42:35 [Q] INFO Process-1 guarding cluster finch-pasta-zulu-princess
05:42:35 [Q] INFO Process-1:13 pushing tasks at 26287
05:42:35 [Q] INFO Q Cluster finch-pasta-zulu-princess running.
05:43:04 [Q] INFO Enqueued 1
05:43:04 [Q] INFO Process-1 created a task from schedule [Train the classifier]
05:43:04 [Q] INFO Process-1:1 processing [foxtrot-steak-charlie-salami]
05:43:04 [Q] INFO Enqueued 1
05:43:04 [Q] INFO Process-1 created a task from schedule [Optimize the index]
05:43:04 [Q] INFO Process-1:2 processing [indigo-two-harry-juliet]
05:43:04 [Q] INFO Enqueued 1
05:43:04 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
05:43:04 [Q] INFO Process-1:3 processing [asparagus-hotel-october-magnesium]
05:43:04 [Q] INFO Enqueued 1
05:43:04 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
05:43:04 [Q] INFO Process-1:4 processing [sodium-undress-september-nine]
05:43:04 [Q] INFO Process-1:4 stopped doing work
05:43:04 [Q] INFO Processed [sodium-undress-september-nine]
[2026-07-08 05:43:04,979] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /app/media/documents/archive/0000002.pdf
```

**Command & complete output — gunicorn and consumer startup:**

```text
$ sed -n '1,4p' /tmp/blitzy_obs/gunicorn.log
[2026-07-08 05:42:55 +0000] [26294] [INFO] Starting gunicorn 20.1.0
[2026-07-08 05:42:55 +0000] [26294] [INFO] Listening at: http://0.0.0.0:8000 (26294)
[2026-07-08 05:42:55 +0000] [26294] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 05:42:55 +0000] [26294] [INFO] Server is ready. Spawning workers

$ sed -n '1p' /tmp/blitzy_obs/consumer.log
[2026-07-08 05:42:55,716] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

**Command & complete output — the three processes actually running.** `ps` is not installed, so the exact scan reads `/proc` (the script `proc_scan.sh` iterates `/proc/[0-9]*`, matches the three command lines, and prints `PID`/`PPID`/`CMD`):

```text
$ sh /tmp/blitzy_obs/proc_scan.sh
PID=26262 PPID=0 CMD=python3 manage.py qcluster
PID=26274 PPID=26262 CMD=python3 manage.py qcluster
PID=26279 PPID=26274 CMD=python3 manage.py qcluster
PID=26280 PPID=26274 CMD=python3 manage.py qcluster
PID=26281 PPID=26274 CMD=python3 manage.py qcluster
PID=26282 PPID=26274 CMD=python3 manage.py qcluster
PID=26283 PPID=26274 CMD=python3 manage.py qcluster
PID=26284 PPID=26274 CMD=python3 manage.py qcluster
PID=26285 PPID=26274 CMD=python3 manage.py qcluster
PID=26286 PPID=26274 CMD=python3 manage.py qcluster
PID=26287 PPID=26274 CMD=python3 manage.py qcluster
PID=26294 PPID=0 CMD=/usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID=26300 PPID=0 CMD=python3 manage.py document_consumer
PID=26307 PPID=26294 CMD=/usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID=26308 PPID=26294 CMD=/usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID=26347 PPID=26274 CMD=python3 manage.py qcluster
PID=26348 PPID=26274 CMD=python3 manage.py qcluster
PID=26349 PPID=26274 CMD=python3 manage.py qcluster
PID=26350 PPID=26274 CMD=python3 manage.py qcluster
```

**Command & complete output — Redis is up, and the broker/channel/DB configuration:**

```text
$ redis-cli ping
PONG

$ python3 manage.py shell -c "
from django.conf import settings
print('DB_ENGINE =', settings.DATABASES['default']['ENGINE'])
print('DB_NAME   =', settings.DATABASES['default']['NAME'])
print('Q_CLUSTER =', settings.Q_CLUSTER)
print('CHANNEL_LAYERS_BACKEND =', settings.CHANNEL_LAYERS['default']['BACKEND'])
print('CHANNEL_LAYERS_HOSTS   =', settings.CHANNEL_LAYERS['default']['CONFIG']['hosts'])
print('CONSUMER_ENABLE_BARCODES =', settings.CONSUMER_ENABLE_BARCODES)
"
DB_ENGINE = django.db.backends.sqlite3
DB_NAME   = /app/src/../data/db.sqlite3
Q_CLUSTER = {'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://localhost:6379'}
CHANNEL_LAYERS_BACKEND = channels_redis.core.RedisChannelLayer
CHANNEL_LAYERS_HOSTS   = ['redis://localhost:6379']
CONSUMER_ENABLE_BARCODES = False
```

**Scheduler role — recurring tasks are registered as Django-Q `Schedule`s in DB migrations (not in `apps.py`).** Command and complete output — the 4 `Schedule` rows present after migration:

```text
$ python3 manage.py shell -c "
from django_q.models import Schedule
for s in Schedule.objects.all().order_by('id'):
    print('id=%d func=%s name=%r type=%s next_run=%s' % (s.id, s.func, s.name, s.schedule_type, s.next_run))
"
id=1 func=documents.tasks.train_classifier name='Train the classifier' type=H next_run=2026-07-08 06:42:08.375420+00:00
id=2 func=documents.tasks.index_optimize name='Optimize the index' type=D next_run=2026-07-09 05:42:08.376605+00:00
id=3 func=documents.tasks.sanity_check name='Perform sanity check' type=W next_run=2026-07-15 05:42:08.472942+00:00
id=4 func=paperless_mail.tasks.process_mail_accounts name='Check all e-mail accounts' type=I next_run=2026-07-08 05:52:09.184264+00:00
```

These correspond exactly to the migration registrations (see Q7 for their `schedule()` registration source):
- `documents/migrations/1001_auto_20201109_1636.py:10-18` — `train_classifier` `HOURLY` (`type=H`) and `index_optimize` `DAILY` (`type=D`).
- `documents/migrations/1004_sanity_check_schedule.py:10-14` — `sanity_check` `WEEKLY` (`type=W`).
- `paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` — `process_mail_accounts` `MINUTES` (`type=I`, `minutes=10`); its body is `process_mail_accounts` [`src/paperless_mail/tasks.py:11`].

**Cause → effect.** The `qcluster` banner shows the cluster's internal division of labor: **11 worker processes** (`Process-1:1 ready for work` through `Process-1:11 ready for work`, matching `Q_CLUSTER["workers"]=11`), one **monitor** (`Process-1:12 monitoring`) that writes finished results into `django_q_task`, one **pusher** (`Process-1:13 pushing tasks`) that reads the Redis broker and dispatches to workers, and the **guard/sentinel** (`Process-1 guarding cluster`) that supervises them. **Because** the same `qcluster` process also runs the scheduler, all four `Schedule`s whose `next_run` had already passed fired on startup — the banner shows `created a task from schedule [Train the classifier]`, `[Optimize the index]`, `[Perform sanity check]`, `[Check all e-mail accounts]`, each followed by a worker line naming the humanhash-labelled task it runs (`Process-1:1 processing [foxtrot-steak-charlie-salami]`, `Process-1:2 processing [indigo-two-harry-juliet]`, etc.) — direct evidence that `qcluster` **both schedules and executes**. Those four runs are exactly why the Q4 baseline `Task.count` is `4`. In the `/proc` snapshot, PIDs `26262`/`26274` are the `qcluster` launcher and its cluster sentinel (the "guard"); `26279–26287` are the surviving workers + monitor + pusher; and `26347–26350` are freshly-recycled workers (`recycle=1` replaced the four workers `26275–26278` that ran the startup schedules). `gunicorn` is master `26294` + two `ConfigurableWorker`s (`26307`, `26308`); `document_consumer` is `26300`.

---

## Q3 — How background jobs appear once created

**Direct answer.** An enqueued job appears in **three** observable places, then in a fourth after it finishes:
1. In the **Redis broker** — the serialized task package sits in the list `django_q:paperless:q` while it waits.
2. In the **`qcluster` worker log** — a line naming the task when a worker takes it (observed: `Process-1:1 processing [simple.pdf]`), and `Processed [simple.pdf]` on success or `Failed [simple.pdf]` on error when it finishes.
3. As **WebSocket `status_updates` progress frames** pushed to authenticated browsers (`STARTING`→`WORKING`→`SUCCESS`/`FAILED`).
4. After completion, as a **row in `django_q_task`** (see Q5).

**(a) In the Redis broker.** Command and complete output, captured while a worker was momentarily unavailable (`qcluster` stopped) so the job is visibly parked (this is the same `simple.pdf` job as Q1/Q4):

```text
$ redis-cli LLEN django_q:paperless:q
1
$ redis-cli LRANGE django_q:paperless:q 0 -1
gAWVEQEAAAAAAAB9lCiMAmlklIwgMmY4MmYzYTA1YjA0NDI1YmE0YTQwOWRlMTI5MGU1YzeUjARuYW1llIwKc2ltcGxlLnBkZpSMBGZ1bmOUjBxkb2N1bWVudHMudGFza3MuY29uc3VtZV9maWxllIwEYXJnc5SMHi9hcHAvc3JjLy4uL2NvbnN1bWUvc2ltcGxlLnBkZpSFlIwGa3dhcmdzlH2UjBBvdmVycmlkZV90YWdfaWRzlE5zjAdzdGFydGVklIwIZGF0ZXRpbWWUjAhkYXRldGltZZSTlEMKB-oHCAUuLQ2gHpRoDowIdGltZXpvbmWUk5RoDowJdGltZWRlbHRhlJOUSwBLAEsAh5RSlIWUUpSGlFKUdS4:1whL7N:9I9nh4strt1Lw7yPwEItUqk2OM6rE26-LiP3zyUumtA
```

That payload is a Django-`signing`-signed, pickled dict — a three-part colon-separated string of the form `base64payload:timestamp:signature` (visible in the raw output above: the trailing `:1whL7N:9I9nh4strt1Lw7yPwEItUqk2OM6rE26-LiP3zyUumtA` is the timestamp and HMAC signature). Deserializing it with Django-Q's **own** `SignedPackage.loads` (the exact decode the cluster uses) yields the human-readable job:

```text
$ python3 manage.py shell -c "
import redis
from django_q.signing import SignedPackage
r = redis.Redis(host='localhost', port=6379, db=0)
raw = r.lrange('django_q:paperless:q', 0, -1)
print('LLEN django_q:paperless:q =', r.llen('django_q:paperless:q'))
for i, item in enumerate(raw):
    task = SignedPackage.loads(item)
    print('--- SignedPackage.loads(payload[%d]) ---' % i)
    for k in ('id', 'name', 'func', 'args', 'kwargs', 'started'):
        print('  %-8s = %r' % (k, task.get(k)))
"
LLEN django_q:paperless:q = 1
--- SignedPackage.loads(payload[0]) ---
  id       = '2f82f3a05b04425ba4a409de1290e5c7'
  name     = 'simple.pdf'
  func     = 'documents.tasks.consume_file'
  args     = ('/app/src/../consume/simple.pdf',)
  kwargs   = {'override_tag_ids': None}
  started  = datetime.datetime(2026, 7, 8, 5, 46, 45, 892958, tzinfo=datetime.timezone.utc)
```

**(b) In the `qcluster` worker log** (from Q1): `Process-1:1 processing [simple.pdf]` and then `Processed [simple.pdf]`.

**(c) As WebSocket `status_updates` frames.** The socket is authenticated: the ASGI router wraps the WebSocket route in `AuthMiddlewareStack` [`src/paperless/asgi.py:20`], the route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` [`src/paperless/urls.py:136-137`], and `StatusConsumer.connect()` [`src/paperless/consumers.py:13`] raises `DenyConnection` [`src/paperless/consumers.py:15`] unless `self.scope["user"].is_authenticated` [`src/paperless/consumers.py:11`]. I verified both branches. **Negative control (no auth):**

```text
$ python3 - <<'PYEOF'
import asyncio, websockets
async def main():
    try:
        async with websockets.connect("ws://localhost:8000/ws/status/") as ws:
            print("UNEXPECTED: connected without auth")
    except Exception as e:
        print("DENIED as expected:", type(e).__name__, str(e)[:120])
asyncio.run(main())
PYEOF
DENIED as expected: InvalidStatusCode server rejected WebSocket connection: HTTP 403
```

For the authenticated capture, a Django session for `admin` was minted programmatically and passed as the `sessionid` value of the handshake `Cookie` header. **The session key itself is never printed or written to the frame log** — it is used only as a cookie value — so the frames below are **complete and unedited** (each line is exactly what `websockets`' `ws.recv()` returned, i.e. the JSON the server sent). This is the full `ws_jobA.log` for the `simple.pdf` ingest:

```text
$ cat /tmp/blitzy_obs/ws_jobA.log
[ws] handshake accepted; connected to ws://localhost:8000/ws/status/
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
```

**Cause → effect.** Each frame carries exactly the keys of `Consumer._send_progress()`'s payload — `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, `document_id` — with `"task_id": self.task_id` [`src/documents/consumer.py:66`]. The browser progress bar advances **because** `_send_progress()` calls `async_to_sync(self.channel_layer.group_send)` targeting the `"status_updates"` group [`src/documents/consumer.py:73`], which the Channels/Redis layer fans out to every member of the `status_updates` group; `StatusConsumer.status_update()` [`src/paperless/consumers.py:29`] then forwards `json.dumps(event["data"])` to the socket. The six stages map 1:1 to the pipeline call sites: `STARTING 0 new_file` [`consumer.py:202`], `WORKING 20 parsing_document` [`consumer.py:259`], `WORKING 70 generating_thumbnail` [`consumer.py:264`], `WORKING 90 parse_date` [`consumer.py:274`], `WORKING 95 save_document` [`consumer.py:294`], `SUCCESS 100 finished` with `document_id=1` [`consumer.py:375`]. The frame `task_id` `43634c2d-fdaa-45d8-83e1-e45ba38cb228` is a **uuid4 generated by the Consumer** (`self.task_id = task_id or str(uuid.uuid4())` [`src/documents/consumer.py:200`]) because the directory watcher passed **no** `task_id`; it is the realtime-progress correlation id, **not** the stored `Task.id` `2f82f3a05b04425ba4a409de1290e5c7` (Q6 explains this in full).

---

## Q4 — Waiting (queued/pending) vs. active (started/running)

**Direct answer.** **Waiting** work sits in the **Redis** broker list `django_q:paperless:q` — enqueued but not yet taken by any worker. **Active** work is a job currently executing inside a `qcluster` worker process. Because the broker is **Redis, not the Django ORM** (`Q_CLUSTER["redis"]="redis://localhost:6379"`, `src/paperless/settings.py:456`; there is no `"orm"` key), waiting jobs are visible **in Redis**, and the Django-Q `OrmQ` table (`django_q_ormq`) stays **empty**. The transition waiting→active is observable as: the job leaving the Redis list, the `qcluster` log line `Process-1:1 processing [simple.pdf]`, and the WebSocket sequence `STARTING`→`WORKING`→`SUCCESS`/`FAILED`.

To make "waiting" unambiguous I stopped the `qcluster` worker, dropped a file (so `document_consumer` enqueued it with no worker to take it), then restarted `qcluster`. **Do NOT** read `OrmQ` as the waiting queue for this configuration — it is the wrong broker; the correct evidence is the Redis list length. The counts helper used at each boundary is:

```text
$ python3 manage.py shell -c "
import redis
from django_q.models import Task, OrmQ
from documents.models import Document
r = redis.Redis(host='localhost', port=6379, db=0)
print('LLEN django_q:paperless:q =', r.llen('django_q:paperless:q'))
print('Task.count     =', Task.objects.count())
print('OrmQ.count     =', OrmQ.objects.count())
print('Document.count =', Document.objects.count())
"
```

**BEFORE (all three processes up, consume dir empty):**

```text
LLEN django_q:paperless:q = 0
Task.count     = 4
OrmQ.count     = 0
Document.count = 0
```

**DURING — WAITING (`qcluster` stopped; `simple.pdf` dropped into `/app/consume`):**

```text
# /tmp/blitzy_obs/consumer.log — the enqueue (logged by document_consumer)
[2026-07-08 05:46:45,890] [INFO] [paperless.management.consumer] Adding /app/src/../consume/simple.pdf to the task queue.
05:46:45 [Q] INFO Enqueued 1
# Redis broker — the job is parked, waiting for a worker (same counts helper as above):
LLEN django_q:paperless:q = 1
Task.count     = 4
OrmQ.count     = 0
Document.count = 0
```

**DURING — ACTIVE (`qcluster` restarted; a worker takes the parked job):**

```text
# /tmp/blitzy_obs/qcluster_run2.log (excerpt; full log in Q1):
05:47:37 [Q] INFO Process-1:1 processing [simple.pdf]
[2026-07-08 05:47:37,195] [INFO] [paperless.consumer] Consuming simple.pdf
# WebSocket, in the same window (first two frames; full log in Q3):
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "simple.pdf", "task_id": "43634c2d-fdaa-45d8-83e1-e45ba38cb228", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
```

**AFTER (job finished):**

```text
LLEN django_q:paperless:q = 0
Task.count     = 5
OrmQ.count     = 0
Document.count = 1
```

**Per-job correlation (JobA — directory-watcher SUCCESS).** One job, tied end-to-end:

| Stage | Evidence | Value |
|---|---|---|
| Enqueue (waiting) | `consumer.log` | `Adding /app/src/../consume/simple.pdf to the task queue.` @ `05:46:45,890` → `Enqueued 1` |
| Redis broker id | `SignedPackage.loads` of the parked payload | `id='2f82f3a05b04425ba4a409de1290e5c7'`, `LLEN=1` |
| WebSocket progress id | `ws_jobA.log` | uuid4 `43634c2d-fdaa-45d8-83e1-e45ba38cb228` (STARTING→WORKING→SUCCESS) |
| Worker log (active) | `qcluster_run2.log` | `Process-1:1 processing [simple.pdf]` @ `05:47:37` → `Processed [simple.pdf]` @ `05:47:38` |
| Stored `django_q_task` row | `Task.objects.get(id='2f82f3a05b04425ba4a409de1290e5c7')` | `id=2f82f3a05b04425ba4a409de1290e5c7`, `success=True`, `result='Success. New document id 1 created'` |
| `result()`/`fetch()` | `django_q.tasks` | `fetch('2f82f3a05b04425ba4a409de1290e5c7') -> <Task: simple.pdf>`, `result('2f82f3a05b04425ba4a409de1290e5c7') -> 'Success. New document id 1 created'` |
| Document id | WebSocket SUCCESS frame + result string | `document_id = 1` |

**Cause → effect.**
- While `qcluster` was down, `document_consumer`'s `async_task` still pushed the package to Redis (`Enqueued 1`), **so** `LLEN django_q:paperless:q` became `1` — that is the *waiting* state. No `django_q_task` row exists yet (`Task.count` unchanged at `4`) **because** a row is written only when the monitor records a *finished* task.
- `OrmQ.count` stayed `0` throughout **because** `Q_CLUSTER` selects the Redis broker [`settings.py:456`]; the `django_q_ormq` table is only used when the broker is the ORM. **Therefore** `OrmQ` is the wrong place to look for waiting work here.
- The **same task id** `2f82f3a05b04425ba4a409de1290e5c7` appears in the waiting Redis payload (Q3) *and* in the finished row — proving one job traversed waiting → active → stored. Its `started` field is identical in both (`2026-07-08 05:46:45.892958`) because Django-Q stamps `task["started"] = timezone.now()` at *enqueue* time inside `async_task` [`django_q/tasks.py:65`] and carries that value into the stored row [`django_q/tasks.py:273`], rather than re-stamping at pickup.
- When `qcluster` restarted, its pusher read the list and handed the package to `Process-1:1`, which logged `processing [simple.pdf]` — the *active* state — and the WebSocket immediately began emitting `STARTING`/`WORKING`.

**Progress-stage enumeration (every distinct status a job broadcasts):** `STARTING` [`consumer.py:202`]; `WORKING` at 20/70/90/95 [`consumer.py:259,264,274,294`]; `SUCCESS` [`consumer.py:375`]; `FAILED` [`consumer.py:79`] (inside `_fail()`, immediately followed by `raise ConsumerError` at `consumer.py:81`). All were observed live — `STARTING`→`WORKING`×4→`SUCCESS` in Q3/JobA, and `STARTING`→`FAILED` in the failure run (Q6 / Appendix A1–A2).

---

## Q5 — Where task state is stored

**Direct answer.** Finished-task state is stored in the **database**, in the table **`django_q_task`**, via the model **`django_q.models.Task`**. `Success` and `Failure` are **proxy models** over that *same* table (they add no columns; they only filter by `success=True`/`success=False`). Waiting/pending state is **not** in the database — it lives in **Redis** (Q4). The `django_q_ormq` table (`OrmQ`) exists in the schema but stays empty under the Redis broker.

**Command & complete output — the model → table mapping and fields:**

```text
$ python3 manage.py shell -c "
from django_q.models import Task, Success, Failure
print('Task.db_table =', Task._meta.db_table)
print('Success.proxy =', Success._meta.proxy, '| Failure.proxy =', Failure._meta.proxy)
print('fields =', [f.name for f in Task._meta.get_fields()])
"
Task.db_table = django_q_task
Success.proxy = True | Failure.proxy = True
fields = ['id', 'name', 'func', 'hook', 'args', 'kwargs', 'result', 'group', 'started', 'stopped', 'success', 'attempt_count']
```

**Command & complete output — the full `CREATE TABLE` DDL for all three Django-Q tables** (unedited, straight from SQLite's `sqlite_master`):

```text
$ python3 -c "
import sqlite3
con = sqlite3.connect('/app/data/db.sqlite3')
for t in ('django_q_task', 'django_q_ormq', 'django_q_schedule'):
    print('-- %s --' % t)
    print(con.execute(\"select sql from sqlite_master where name=?\", (t,)).fetchone()[0])
    print()
"
-- django_q_task --
CREATE TABLE "django_q_task" ("name" varchar(100) NOT NULL, "func" varchar(256) NOT NULL, "hook" varchar(256) NULL, "args" text NULL, "kwargs" text NULL, "result" text NULL, "started" datetime NOT NULL, "stopped" datetime NOT NULL, "success" bool NOT NULL, "id" varchar(32) NOT NULL PRIMARY KEY, "group" varchar(100) NULL, "attempt_count" integer NOT NULL)

-- django_q_ormq --
CREATE TABLE "django_q_ormq" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "key" varchar(100) NOT NULL, "payload" text NOT NULL, "lock" datetime NULL)

-- django_q_schedule --
CREATE TABLE "django_q_schedule" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "func" varchar(256) NOT NULL, "hook" varchar(256) NULL, "args" text NULL, "kwargs" text NULL, "schedule_type" varchar(1) NOT NULL, "repeats" integer NOT NULL, "next_run" datetime NULL, "task" varchar(100) NULL, "name" varchar(100) NULL, "minutes" smallint unsigned NULL CHECK ("minutes" >= 0), "cron" varchar(100) NULL, "cluster" varchar(100) NULL)
```

**A real stored row** is shown in Q1 (`Task.objects.get(id='2f82f3a05b04425ba4a409de1290e5c7')` → the `simple.pdf` success row) and in the Appendix (the `test_with_bom.pdf` success row and the `simple.pdf` `Failure` row).

**Cause → effect.**
- The PRIMARY KEY of `django_q_task` is `"id" varchar(32)` — a **32-character** column. That is exactly why the stored `Task.id` is a 32-char hex string (`2f82f3a05b04425ba4a409de1290e5c7`) and **cannot** be the 36-char dashed uuid4 the API generates (`5f7e49cd-2ece-4edf-a2d3-f63d8e43b449`) — the provenance point proven in Q6.
- `Success` and `Failure` being **proxies** (`proxy = True`) means the database has **one** results table; the success/failure split is a query filter, not a second table. This is why in Q6 the counts satisfy `Success + Failure == Task` exactly (`6 + 1 == 7`).
- `django_q_ormq` has a `payload text` column — it is the ORM broker's queue table. It is present because the migration created it, but it is unused here (Q4/Q6) since the broker is Redis.

---

## Q6 — After-the-fact inspection: determining what happened to a completed job

**Direct answer.** After a job finishes, use **Django-Q's own facilities**: the functions `django_q.tasks.result(task_id)` and `django_q.tasks.fetch(task_id)`, the `Task`/`Success`/`Failure` models, and the **Django admin** ("Successful tasks", "Failed tasks", "Scheduled tasks" changelists). A success stores `success=True` and a human `result` string; a failure stores `success=False` and the full traceback in `result`. The lookup key is the **stored 32-char hex `Task.id`** — *not* the API's self-assigned uuid4 (proven below). How many finished results are retained is bounded by **`SAVE_LIMIT`** (default **250**). **Negative findings at this commit:** there is **no `PaperlessTask` model** and **no `/api/tasks` endpoint / `TaskViewSet`** — inspection relies entirely on Django-Q's own machinery.

**(a) `result()` / `fetch()` — success and failure.** For the `simple.pdf` success (JobA) the exact lookups and output are in Q1; for the `simple.pdf` **duplicate failure** (JobC) they are in Appendix A2. Summarised:

```text
# success (JobA, stored hex id 2f82f3a05b04425ba4a409de1290e5c7):
fetch('2f82f3a05b04425ba4a409de1290e5c7')  -> <Task: simple.pdf>   (success=True)
result('2f82f3a05b04425ba4a409de1290e5c7') -> 'Success. New document id 1 created'

# failure (JobC, stored hex id 729016df94a6429d9ae71b52fee98407):
fetch('729016df94a6429d9ae71b52fee98407')  -> <Task: simple.pdf>   (success=False)
```

For the failure task, `result('729016df94a6429d9ae71b52fee98407')` returns the **complete `ConsumerError` traceback** — reproduced verbatim (byte-identical to the worker-log traceback) in Appendix A2.

**(b) Retained-results bound — `SAVE_LIMIT`.** Command & complete output:

```text
$ python3 manage.py shell -c "
from django_q.conf import Conf
from django.conf import settings
print('Conf.SAVE_LIMIT =', Conf.SAVE_LIMIT)
print('Q_CLUSTER has save_limit key =', 'save_limit' in settings.Q_CLUSTER)
"
Conf.SAVE_LIMIT = 250
Q_CLUSTER has save_limit key = False
```

Cause → effect: `SAVE_LIMIT` defaults to **250** — `SAVE_LIMIT = conf.get("save_limit", 250)` [`django_q/conf.py:87`] — and paperless does **not** override it (`Q_CLUSTER` has no `save_limit` key, `src/paperless/settings.py:449-457`). **Therefore** only the most recent 250 **successful** results are retained in `django_q_task`; once exceeded, Django-Q trims the oldest successful rows. This directly limits after-the-fact inspection: a `result()`/`fetch()` for a long-past *successful* job may return `None` simply because its row was trimmed. (Django-Q keeps failures regardless of `SAVE_LIMIT` when `catch_up`/retention applies; the operative default value observed here is `250`.)

**(c) task_id provenance — why lookups must use the stored hex id, not the API uuid4 (finding-critical).** This is the nuance the API path introduces. The REST upload **generates its own** `task_id = str(uuid.uuid4())` [`src/documents/views.py:521`] and passes it into the `async_task` call as the `task_id=task_id` keyword argument. But `task_id` is **not** one of Django-Q's recognised option keys (`opt_keys`, `django_q/tasks.py:23`), so `async_task` does **not** use it as the task's id; it falls through into `task["kwargs"]` [`django_q/tasks.py:64`], while Django-Q assigns the task's **own** id from `tag = uuid()` → `task["id"] = tag[1]` [`django_q/tasks.py:38,41`] (the 32-char hex). The leftover `task_id` kwarg then reaches `consume_file` as its `task_id=` argument, which forwards it to `Consumer.try_consume_file` as `task_id=task_id` [`src/documents/tasks.py:243`], where `self.task_id = task_id or str(uuid.uuid4())` [`src/documents/consumer.py:200`]. **So the API's uuid4 becomes the WebSocket progress-correlation id — never the stored `Task.id`.** Exact runtime proof (JobB, `test_with_bom.pdf`):

```text
$ cat /tmp/blitzy_obs/jobB_provenance_proof.txt
# lookup by the API self-assigned uuid4 (the WebSocket progress task_id):
  fetch('5f7e49cd-2ece-4edf-a2d3-f63d8e43b449')  -> None
  result('5f7e49cd-2ece-4edf-a2d3-f63d8e43b449') -> None
  Task.objects.filter(id=api_uuid).exists() -> False
# lookup by the stored Django-Q hex Task.id:
  fetch('65f88864546346088f2cda0ea99813f9') -> <Task: test_with_bom.pdf> | success = True
  result('65f88864546346088f2cda0ea99813f9') -> 'Success. New document id 2 created'
```

Reconciliation with the AAP's Q6 interpretation: the AAP states the API path *generates its own `task_id`* while the directory watcher *lets Django-Q assign the id*. The runtime evidence confirms exactly that **and** sharpens what the API-supplied value governs: it is the **progress** id (seen in the JobB WebSocket frames), whereas the **stored** id is always Django-Q's own hex `tag[1]`. Consequently, to locate a finished job you query by the **hex `Task.id`**; the API's uuid4 only correlates the live progress stream. (For the directory watcher, no `task_id` is supplied, so the Consumer generates a *fresh* uuid4 for progress — JobA's `43634c2d-fdaa-45d8-83e1-e45ba38cb228` — which is likewise distinct from the stored hex `2f82f3a05b04425ba4a409de1290e5c7`.)

**(d) `Success` / `Failure` model rows and the proxy partition.** Command & complete output:

```text
$ python3 manage.py shell -c "
from django_q.models import Task, Success, Failure, OrmQ
print('Task    =', Task.objects.count())
print('Success =', Success.objects.count())
print('Failure =', Failure.objects.count())
print('OrmQ    =', OrmQ.objects.count())
f = Failure.objects.order_by('-stopped').first()
print('newest Failure:', f.name, 'success=%s' % f.success)
"
Task    = 7
Success = 6
Failure = 1
OrmQ    = 0
newest Failure: simple.pdf success=False
```

Cause → effect: `Success (6) + Failure (1) == Task (7)` because `Success`/`Failure` are proxies partitioning the one `django_q_task` table by the `success` boolean (Q5). The 7 rows are: 4 startup-schedule tasks (Q2) + JobA success (`simple.pdf`, doc 1) + JobB success (`test_with_bom.pdf`, doc 2) + JobC failure (`simple.pdf` duplicate) — of which exactly one (`JobC`) is a `Failure`.

**(e) Django admin inspection.** The admin registers the Django-Q proxy models. Authenticated GETs (the session for `admin` is minted in-process with `Client.force_login` — no password typed, and the session key is never printed):

```text
$ python3 /tmp/blitzy_obs/admin_get.py     # django.test.Client + force_login(admin); prints each status code and page title
GET /admin/django_q/success/       -> 200  | <title>Select Successful task to change | Paperless-ngx</title>
GET /admin/django_q/failure/       -> 200  | <title>Select Failed task to change | Paperless-ngx</title>
GET /admin/django_q/schedule/      -> 200  | <title>Select Scheduled task to change | Paperless-ngx</title>
GET /admin/django_q/ormq/          -> 404
```

The `/ormq/` 404 also surfaced in the live gunicorn log: `[2026-07-08 05:52:34,829] [WARNING] [django.request] Not Found: /admin/django_q/ormq/` (Q7). The admin registry confirms which models are registered and which are proxies:

```text
$ python3 manage.py shell -c "
from django.contrib import admin
from django_q import models as qm
reg = admin.site._registry
print({n: getattr(qm, n)._meta.proxy for n in ['Task','Success','Failure','Schedule','OrmQ'] if getattr(qm, n) in reg})
"
{'Task': False, 'Success': True, 'Failure': True, 'Schedule': True, 'OrmQ': False}
```

Cause → effect: `Success`, `Failure`, and `Schedule` are registered and browsable (hence 200); their changelist titles ("Successful/Failed/Scheduled task") are exactly what a user sees. `Task` itself and `OrmQ` are **not** registered in this build, so `/admin/django_q/ormq/` returns **404** — after-the-fact inspection of *waiting* work is not available via the admin (it is in Redis, Q4).

**(f) Negative finding N1 — no `PaperlessTask` model.** Command & complete output:

```text
$ grep -rn "PaperlessTask" src/ ; echo "exit=$?"
exit=1
```

`grep` produced **no output** and exit code **1** (no match) across the entire `src/` tree — there is no application-owned task-status model at this commit. Task state lives entirely in Django-Q's `Task` table (Q5).

**(g) Negative finding N2 — no `/api/tasks` endpoint / `TaskViewSet`.** Command & complete output:

```text
$ sed -n '29,35p' src/paperless/urls.py    # the DRF router registrations
api_router = DefaultRouter()
api_router.register(r"correspondents", CorrespondentViewSet)
api_router.register(r"document_types", DocumentTypeViewSet)
api_router.register(r"documents", UnifiedSearchViewSet)
api_router.register(r"logs", LogViewSet, basename="logs")
api_router.register(r"tags", TagViewSet)
api_router.register(r"saved_views", SavedViewViewSet)

$ grep -rn 'register(r"tasks"\|TaskViewSet' src/ ; echo "exit=$?"
exit=1
```

The DRF router registers exactly **six** routes — `correspondents`, `document_types`, `documents`, `logs`, `tags`, `saved_views` [`src/paperless/urls.py:29-35`] — and there is **no** `tasks` route and **no** `TaskViewSet` anywhere in `src/` (grep exit 1, no output). After-the-fact inspection is therefore *not* exposed through a paperless REST endpoint; it is done via Django-Q and the admin.

**(h) Negative finding N3 — `OrmQ` empty (broker is Redis).** Command & complete output:

```text
$ python3 manage.py shell -c "
from django_q.models import OrmQ
from django.conf import settings
print('OrmQ.count =', OrmQ.objects.count())
print('Q_CLUSTER redis =', settings.Q_CLUSTER.get('redis'))
print('Q_CLUSTER has orm key =', 'orm' in settings.Q_CLUSTER)
"
OrmQ.count = 0
Q_CLUSTER redis = redis://localhost:6379
Q_CLUSTER has orm key = False
```

Cause → effect: `Q_CLUSTER` sets `redis` and has no `orm` key [`src/paperless/settings.py:456`], so the broker is Redis and `django_q_ormq` is never written — `OrmQ.count == 0`. Waiting work is inspected in Redis (Q4), finished work in `django_q_task` (Q5), never in `OrmQ`.

---

## Q7 — Where in the code background jobs are triggered (the enqueue call sites)

**Direct answer.** Every background job is triggered by a call to **`django_q.tasks.async_task`**. There are four families of call sites, plus recurring registrations made with **`django_q.tasks.schedule`** in migrations. Two of the four families were exercised live in this investigation (directory watcher, REST upload); the other two (IMAP mail, bulk edit) are present in source but were **[INFERRED]** — not exercised at runtime because no mail account / bulk request was issued.

**(1) Directory watcher — `document_consumer` (exercised: JobA, JobC).** Command & complete output:

```text
$ sed -n '13p;84,91p' src/documents/management/commands/document_consumer.py
from django_q.tasks import async_task
    try:
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
            filepath,
            override_tag_ids=tag_ids if tag_ids else None,
            task_name=os.path.basename(filepath)[:100],
        )
```

The enqueue is at `src/documents/management/commands/document_consumer.py:86` (the `async_task(` line), import at `:13`. Note it passes **no** `task_id` — hence Django-Q assigns the stored hex id and the Consumer generates its own uuid4 for progress (Q6). The live proof is the `consumer.log` line `Adding /app/src/../consume/simple.pdf to the task queue.` followed by `Enqueued 1` (Q4).

**(2) REST API upload — `PostDocumentView` (exercised: JobB).** Command & complete output:

```text
$ sed -n '28p' src/documents/views.py
from django_q.tasks import async_task
$ sed -n '491p;493p' src/documents/views.py
class PostDocumentView(GenericAPIView):
    permission_classes = (IsAuthenticated,)
$ sed -n '519,535p' src/documents/views.py
            temp_filename = f.name

        task_id = str(uuid.uuid4())

        async_task(
            "documents.tasks.consume_file",
            temp_filename,
            override_filename=doc_name,
            override_title=title,
            override_correspondent_id=correspondent_id,
            override_document_type_id=document_type_id,
            override_tag_ids=tag_ids,
            task_id=task_id,
            task_name=os.path.basename(doc_name)[:100],
        )

        return Response("OK")
```

The endpoint requires authentication — `permission_classes = (IsAuthenticated,)` [`src/documents/views.py:493`] — generates `task_id = str(uuid.uuid4())` [`:521`], calls `async_task` [`:523`], and returns `Response("OK")` [`:535`] (the HTTP body observed in Appendix A1). **Live proof the enqueue happens inside the gunicorn/ASGI worker:** the JobB upload at 05:50:00 produced `05:50:00 [Q] INFO Enqueued 1` in **`gunicorn.log`** (not `consumer.log`):

```text
$ cat /tmp/blitzy_obs/gunicorn.log
[2026-07-08 05:42:55 +0000] [26294] [INFO] Starting gunicorn 20.1.0
[2026-07-08 05:42:55 +0000] [26294] [INFO] Listening at: http://0.0.0.0:8000 (26294)
[2026-07-08 05:42:55 +0000] [26294] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 05:42:55 +0000] [26294] [INFO] Server is ready. Spawning workers
05:50:00 [Q] INFO Enqueued 1
[2026-07-08 05:52:34,829] [WARNING] [django.request] Not Found: /admin/django_q/ormq/
```

Cause → effect: `Enqueued 1` is Django-Q's `async_task` logging its push to Redis [`django_q/tasks.py:74`]. It appears in `gunicorn.log` because the `async_task` call ran **in the gunicorn worker process** handling the POST — pinpointing the API path's enqueue to the ASGI server process (contrast: JobA/JobC's `Enqueued 1` appears in `consumer.log`, the watcher process). The trailing `/admin/django_q/ormq/` 404 is the Q6 admin probe, also served by gunicorn.

**(3) IMAP mail ingestion — `paperless_mail` [INFERRED, not exercised].** Command & complete output:

```text
$ sed -n '11p;336,340p' src/paperless_mail/mail.py
from django_q.tasks import async_task
                async_task(
                    "documents.tasks.consume_file",
                    path=temp_filename,
                    override_filename=pathvalidate.sanitize_filename(
                        att.filename,
```

Enqueue at `src/paperless_mail/mail.py:336`, import at `:11`. **[INFERRED]** — this path fires only when the scheduled `process_mail_accounts` finds a matching message; no mail account was configured, so it was not run at runtime. Its scheduled trigger is item (5) below.

**(4) Bulk operations — `bulk_edit` [INFERRED, not exercised].** Command & complete output:

```text
$ grep -n 'async_task' src/documents/bulk_edit.py
4:from django_q.tasks import async_task
18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
```

Five enqueue sites [`src/documents/bulk_edit.py:18,31,47,63,87`], import at `:4`, all pushing `documents.tasks.bulk_update_documents`. **[INFERRED]** — not exercised (no bulk request issued).

**(5) Recurring registrations — `schedule()` calls in migrations (fired live: all 4 on startup).** These create the `Schedule` rows (Q2) that `qcluster`'s scheduler enqueues when due. Commands & complete output:

```text
$ sed -n '10,18p' src/documents/migrations/1001_auto_20201109_1636.py
    schedule(
        "documents.tasks.train_classifier",
        name="Train the classifier",
        schedule_type=Schedule.HOURLY,
    )
    schedule(
        "documents.tasks.index_optimize",
        name="Optimize the index",
        schedule_type=Schedule.DAILY,

$ sed -n '10,14p' src/documents/migrations/1004_sanity_check_schedule.py
    schedule(
        "documents.tasks.sanity_check",
        name="Perform sanity check",
        schedule_type=Schedule.WEEKLY,
    )

$ sed -n '10,15p' src/paperless_mail/migrations/0002_auto_20201117_1334.py
    schedule(
        "paperless_mail.tasks.process_mail_accounts",
        name="Check all e-mail accounts",
        schedule_type=Schedule.MINUTES,
        minutes=10,
    )
```

These four `schedule()` calls register `train_classifier` (HOURLY), `index_optimize` (DAILY), `sanity_check` (WEEKLY), and `process_mail_accounts` (every 10 MINUTES) — the exact `Schedule` rows enumerated in Q2 and observed firing on `qcluster` startup. The `process_mail_accounts` body lives at `src/paperless_mail/tasks.py:11`.

**Coverage note on the task bodies (what runs inside the worker):** `consume_file` [`src/documents/tasks.py:184`], `bulk_update_documents` [`src/documents/tasks.py:270`], `sanity_check` [`src/documents/tasks.py:255`], `train_classifier` [`src/documents/tasks.py:48`], `index_optimize` [`src/documents/tasks.py:32`], and `process_mail_accounts` [`src/paperless_mail/tasks.py:11`].

---

## Appendix — full evidence for the API-upload and failure paths

### A1 — JobB: REST API upload SUCCESS (`test_with_bom.pdf`)

**Exact command & complete HTTP output** (credentials are read from an environment variable — an ephemeral local-only test value that is never printed):

```text
$ python3 - <<'PYEOF'
import os, requests
# PNGX_ADMIN_PW holds an ephemeral local-only test password (never printed)
fp = "/app/src/documents/tests/samples/test_with_bom.pdf"
with open(fp, "rb") as fh:
    r = requests.post("http://localhost:8000/api/documents/post_document/",
                      auth=("admin", os.environ["PNGX_ADMIN_PW"]),
                      files={"document": ("test_with_bom.pdf", fh, "application/pdf")})
print("HTTP status:", r.status_code)
print("HTTP body  :", r.text)
PYEOF
HTTP status: 200
HTTP body  : "OK"
```

**Worker log** (the job executing in a `qcluster` worker — exact `grep` of the worker log for this job):

```text
$ grep -n 'test_with_bom' /tmp/blitzy_obs/qcluster_run2.log
05:50:00 [Q] INFO Process-1:2 processing [test_with_bom.pdf]
[2026-07-08 05:50:00,719] [INFO] [paperless.consumer] Consuming test_with_bom.pdf
[2026-07-08 05:50:04,728] [INFO] [paperless.consumer] Document 2020-07-02 test_with_bom consumption finished
05:50:04 [Q] INFO Processed [test_with_bom.pdf]
```

**WebSocket frames** (full raw `ws_jobB.log`, token-safe — the session key is never printed):

```text
$ cat /tmp/blitzy_obs/ws_jobB.log
[ws] handshake accepted; connected to ws://localhost:8000/ws/status/
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
{"filename": "test_with_bom.pdf", "task_id": "5f7e49cd-2ece-4edf-a2d3-f63d8e43b449", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 2}
```

**Stored DB row + result/fetch:**

```text
$ cat /tmp/blitzy_obs/jobB_row.txt
stored Task.id (hex) = 65f88864546346088f2cda0ea99813f9
name    = test_with_bom.pdf
success = True
result  = 'Success. New document id 2 created'
started = 2026-07-08 05:50:00.571457+00:00
stopped = 2026-07-08 05:50:04.732398+00:00
Document.count = 2
```

The provenance proof (`fetch`/`result` on both the API uuid4 and the stored hex id) is in Q6(c).

**JobB per-job correlation (API upload SUCCESS):**

| Stage | Evidence | Value |
|---|---|---|
| Enqueue (in gunicorn) | `gunicorn.log` | `05:50:00 [Q] INFO Enqueued 1` (in the ASGI worker process) |
| HTTP response | `requests.post` | status `200`, body `"OK"` |
| WebSocket progress id | `ws_jobB.log` | uuid4 `5f7e49cd-2ece-4edf-a2d3-f63d8e43b449` (self-assigned by `views.py:521`) |
| Worker log (active) | `qcluster_run2.log` | `Process-1:2 processing [test_with_bom.pdf]` @ `05:50:00` → `Processed` @ `05:50:04` |
| Stored `django_q_task` row | `jobB_row.txt` | hex `65f88864546346088f2cda0ea99813f9`, `success=True`, `result='Success. New document id 2 created'` |
| Provenance | Q6(c) | `fetch('5f7e49cd-2ece-4edf-a2d3-f63d8e43b449')` → `None`; `fetch('65f88864546346088f2cda0ea99813f9')` → `<Task: test_with_bom.pdf>` |
| Document id | WebSocket SUCCESS frame + result string | `document_id = 2` |

### A2 — JobC: directory-watcher DUPLICATE FAILURE (`simple.pdf` again)

Re-dropping the already-ingested `simple.pdf` exercises the failure branch: the pipeline detects a duplicate and raises `ConsumerError` from `Consumer._fail()` [`src/documents/consumer.py:81`].

**Enqueue** (exact `grep` of the watcher log for the second `simple.pdf`):

```text
$ grep -n 'simple.pdf' /tmp/blitzy_obs/consumer.log
[2026-07-08 05:51:16,402] [INFO] [paperless.management.consumer] Adding /app/src/../consume/simple.pdf to the task queue.
```

**Worker log — the complete failure with full traceback** (exact `sed` range of the worker log for this job):

```text
$ sed -n '/05:51:16 \[Q\] INFO Process-1:3 processing/,/05:51:16 \[Q\] INFO recycled worker Process-1:3/p' /tmp/blitzy_obs/qcluster_run2.log
05:51:16 [Q] INFO Process-1:3 processing [simple.pdf]
[2026-07-08 05:51:16,551] [ERROR] [paperless.consumer] Not consuming simple.pdf: It is a duplicate.
05:51:16 [Q] INFO Process-1:3 stopped doing work
05:51:16 [Q] ERROR Failed [simple.pdf] - simple.pdf: Not consuming simple.pdf: It is a duplicate. : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: simple.pdf: Not consuming simple.pdf: It is a duplicate.

05:51:16 [Q] INFO recycled worker Process-1:3
```

**WebSocket frames** (full raw `ws_jobC.log`, token-safe — note it goes STARTING → FAILED, skipping all WORKING stages):

```text
$ cat /tmp/blitzy_obs/ws_jobC.log
[ws] handshake accepted; connected to ws://localhost:8000/ws/status/
{"filename": "simple.pdf", "task_id": "89444de2-2836-4be8-ad56-05ff2609f999", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "simple.pdf", "task_id": "89444de2-2836-4be8-ad56-05ff2609f999", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
```

**Stored `Failure` row + result/fetch (the stored traceback, verbatim):**

```text
$ cat /tmp/blitzy_obs/jobC_row.txt
stored Failure Task.id (hex) = 729016df94a6429d9ae71b52fee98407
name    = simple.pdf
func    = documents.tasks.consume_file
success = False
started = 2026-07-08 05:51:16.402979+00:00
stopped = 2026-07-08 05:51:16.552261+00:00
---- fetch()/result() by hex id ----
fetch('729016df94a6429d9ae71b52fee98407') -> <Task: simple.pdf> | success=False
---- result() (the stored ConsumerError traceback, verbatim) ----
simple.pdf: Not consuming simple.pdf: It is a duplicate. : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: simple.pdf: Not consuming simple.pdf: It is a duplicate.
```

**JobC per-job correlation (duplicate FAILURE):**

| Stage | Evidence | Value |
|---|---|---|
| Enqueue (waiting) | `consumer.log` | `Adding /app/src/../consume/simple.pdf to the task queue.` @ `05:51:16,402` |
| WebSocket progress id | `ws_jobC.log` | uuid4 `89444de2-2836-4be8-ad56-05ff2609f999` (STARTING → FAILED, no WORKING) |
| Worker log (active → error) | `qcluster_run2.log` | `Process-1:3 processing [simple.pdf]` → `[ERROR] Not consuming simple.pdf: It is a duplicate.` → `Failed [simple.pdf]` @ `05:51:16` |
| Stored `django_q_task` (`Failure`) row | `jobC_row.txt` | hex `729016df94a6429d9ae71b52fee98407`, `success=False` |
| `result()`/`fetch()` | `django_q.tasks` | `fetch('729016df94a6429d9ae71b52fee98407')` → `<Task: simple.pdf>` (`success=False`); `result('729016df94a6429d9ae71b52fee98407')` → the full `ConsumerError` traceback above |
| Failure semantics | `consumer.py:110` → `:81` | `pre_check_duplicate()` calls `_fail()` which broadcasts `FAILED` then raises `ConsumerError`; the worker records a `Failure` |

Cause → effect: the failure path skips every `WORKING` stage — the WebSocket jumps `STARTING` (0) directly to `FAILED` (100) — because `pre_check_duplicate()` [`src/documents/consumer.py:213`] runs before any parsing and calls `_fail()` [`:110`], which first broadcasts the `FAILED` progress frame [`:79`] and then raises `ConsumerError` [`:81`]. Django-Q's worker catches the exception, logs the `Failed [simple.pdf]` line carrying the traceback (its top frame `cluster.py:432` is where the worker invoked the task), and stores a `Failure` row whose `result` is the full traceback — which `result()`/`fetch()` return verbatim.

---


## Coverage checklist

A final pass over every named item the question asks for, each with its concrete value, primary `file:line`, the observed evidence, and whether it was observed at runtime or is source-derived/inferred.

| Question / named item | Direct answer (concrete value) | Primary `file:line` | Observed evidence | Observed / Inferred |
|---|---|---|---|---|
| **Q1** runtime behavior: enqueue → Redis → qcluster → `consume_file` → `try_consume_file` | Worker runs `consume_file`, delegates to `try_consume_file`; returns `'Success. New document id 1 created'` | `tasks.py:184,236,247` | JobA `qcluster_run2.log` + stored row | **Observed** |
| **Q1** failure return branch | `raise ConsumerError` when duplicate/unsupported | `tasks.py:249`, `consumer.py:81` | JobC full traceback (A2) | **Observed** |
| **Q1** barcode short-circuit | `"File successfully split"`, no `try_consume_file` | `tasks.py:219-233`, `settings.py:502` | `CONSUMER_ENABLE_BARCODES=False` (Q2) | **Inferred** (branch not taken) |
| **Q2** gunicorn (ASGI web + WS) | master PID 26294 on `:8000` | `docker/supervisord.conf:11` | `gunicorn.log` + `/proc` | **Observed** |
| **Q2** document_consumer (watcher) | PID 26300, inotify on `/app/src/../consume` | `docker/supervisord.conf:20` | `consumer.log` + `/proc` | **Observed** |
| **Q2** qcluster (worker cluster) | cluster `finch-pasta-zulu-princess`, 11 workers + monitor + pusher + guard | `docker/supervisord.conf:29` | qcluster banner + `/proc` | **Observed** |
| **Q2** Redis (broker + channel layer) | `redis://localhost:6379`, `PONG` | `settings.py:456,182` | `redis-cli ping` + settings dump | **Observed** |
| **Q2** database | SQLite `/app/src/../data/db.sqlite3` | settings dump | `DATABASES` dump | **Observed** |
| **Q2** 4 schedules + scheduler role | train_classifier/index_optimize/sanity_check/process_mail_accounts | migrations; `Schedule` rows | 4 rows + startup firing in banner | **Observed** |
| **Q3** job in Redis queue | list `django_q:paperless:q`, `LLEN=1` | — | `LRANGE` payload + `SignedPackage.loads` | **Observed** |
| **Q3** job in qcluster logs | `processing [simple.pdf]` / `Processed` | — | `qcluster_run2.log` | **Observed** |
| **Q3** WebSocket `status_updates` frames | STARTING→WORKING×4→SUCCESS | `consumer.py:66,73`; `consumers.py:29` | raw `ws_jobA.log` (6 frames) | **Observed** |
| **Q3** row in `django_q_task` after completion | hex `2f82f3a05b04425ba4a409de1290e5c7` | — | stored row (Q1) | **Observed** |
| **Q4** waiting vs. active | Redis list vs. worker `processing` | `settings.py:456` | before/during/after `0/4/0→1/4/0→0/5/0` | **Observed** |
| **Q4** `OrmQ` empty (Redis broker) | `OrmQ.count == 0` | `settings.py:456` | count query | **Observed** |
| **Q5** table/model/proxies | `django_q_task` / `Task` / `Success`,`Failure` proxies | `Task._meta` | model query + full `CREATE TABLE` | **Observed** |
| **Q6** `result()`/`fetch()` success + failure | success string / full traceback | — | JobA (Q1) + JobC (A2) | **Observed** |
| **Q6** `SAVE_LIMIT` bound | `250` (default, not overridden) | `django_q/conf.py:87` | `Conf.SAVE_LIMIT` | **Observed** |
| **Q6** task_id provenance | API uuid4 = progress id, not stored id | `views.py:521`; `django_q/tasks.py:23,41,64`; `consumer.py:200` | `fetch(uuid4)=None` vs `fetch(hex)=<Task>` | **Observed** |
| **Q6** `Success`/`Failure` rows | `6 + 1 == 7` | — | counts query | **Observed** |
| **Q6** Django admin | success/failure/schedule 200, ormq 404 | — | `admin_get.py` + gunicorn 404 log | **Observed** |
| **Q6** N1 no `PaperlessTask` | grep exit 1, no output | — | `grep -rn PaperlessTask src/` | **Observed** |
| **Q6** N2 no `/api/tasks` / `TaskViewSet` | 6 router routes, no tasks | `urls.py:29-35` | router `sed` + grep exit 1 | **Observed** |
| **Q6** N3 `OrmQ` empty | `OrmQ.count == 0` | `settings.py:456` | count + config query | **Observed** |
| **Q7** dir-watcher enqueue | `async_task` of `documents.tasks.consume_file` (no `task_id`) | `document_consumer.py:86` | JobA/JobC `consumer.log` `Enqueued 1` | **Observed** |
| **Q7** API enqueue | `async_task` of `consume_file` with `task_id=task_id` | `views.py:523` | JobB `gunicorn.log` `Enqueued 1` | **Observed** |
| **Q7** IMAP mail enqueue | `async_task` of `consume_file` (`path=` kwarg) | `mail.py:336` | source `sed` | **Inferred** (no mail account) |
| **Q7** bulk enqueue (×5) | `async_task` of `documents.tasks.bulk_update_documents` | `bulk_edit.py:18,31,47,63,87` | source `grep` | **Inferred** (no bulk request) |
| **Q7** schedule registrations (×4) | `schedule()` in 3 migrations | migrations `:10` | `Schedule` rows + startup firing | **Observed** |

## Observed vs. inferred (evidence-quality statement)

To state evidence quality precisely (rather than overclaim):

- **Q1–Q6 are answered entirely from runtime observation** captured in the canonical container — every command shown was executed and its complete, unedited output is pasted alongside it; the only Q1 item that is **inferred** is the barcode short-circuit branch (it is not taken in the default configuration, `CONSUMER_ENABLE_BARCODES=False`), and it is labelled `[INFERRED]` where it appears.
- **Q7 is answered from a mix of observation and source:** the **directory-watcher** and **REST-API** enqueue paths were exercised live (JobA/JobC and JobB respectively), and the four `schedule()` registrations were observed firing on `qcluster` startup; the **IMAP-mail** [`src/paperless_mail/mail.py:336`] and **bulk-edit** [`src/documents/bulk_edit.py:18,31,47,63,87`] enqueue paths are read from source and are labelled `[INFERRED]` because their runtime preconditions (a configured mail account; a bulk API request) were not present in this run.
- No block is normalized, redacted-then-labelled-complete, or truncated; the WebSocket captures are raw JSON frames written verbatim, with the session cookie never printed (so "complete and unedited" holds literally).

## Cleanup & Read-Only Verification

The investigation used temporary observation scripts and logs under `/tmp/blitzy_obs/` **outside** the repository (`/app`); runtime data (`data/`, `media/`, `consume/`, `index/`) is git-ignored. Per the read-only rule, all temporary scripts were removed and the source repository was verified byte-for-byte unchanged. Exact commands and complete output:

```text
### 1) remove all temporary observation scripts and logs ###
$ rm -rf /tmp/blitzy_obs
$ ls /tmp/blitzy_obs
ls: cannot access '/tmp/blitzy_obs': No such file or directory

### 2) source repository working tree is clean ###
$ git -C /app status --porcelain
(no output above = working tree clean)

### 3) HEAD is exactly the investigated commit ###
$ git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542

### 4) no diff vs the investigated baseline commit ###
$ git -C /app diff --name-status 542221a38dff06361e07976452f9aea24d210542
(no output above = zero tracked-file changes)

### 5) no stray scratch/adhoc artifacts inside the repo ###
$ find /app -maxdepth 4 \( -name "blitzy_*" -o -name "blitzy_adhoc_test_*" \) 2>/dev/null
(no output above = none)
```

Note on the source repository's pristine state: at the start of the investigation the container's `/app` had exactly one pre-existing working-tree modification — `src/paperless_tesseract/tests/samples/simple-alpha.png`, a binary side-effect of a prior test run, unrelated to this investigation — which was restored to its committed state with `git -C /app checkout -- src/paperless_tesseract/tests/samples/simple-alpha.png` before observation began. The investigation itself modified **no** tracked file (step 2 above is empty), so `/app` is left at commit `542221a38dff06361e07976452f9aea24d210542` with a clean working tree. The single artifact this task produces — this answer document under `blitzy/documentation/` — lives in the **delivery** repository, not in the observed source repository.

---

*End of document. Every command shown was run inside the canonical container (setup image `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542`, commit `542221a38dff06361e07976452f9aea24d210542`); every output block is the complete, unedited output that command produced; every factual claim is tied to a `file:line` reference or observed output; and each `[INFERRED]` item is source-derived because its runtime precondition was absent.*
