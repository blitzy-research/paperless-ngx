# Background / Asynchronous Processing in paperless-ngx — a run-first, evidence-based investigation

> **Subject:** How paperless-ngx performs background/asynchronous processing while it ingests documents.
> **Engine:** **Django-Q** (the `django_q` app) — **not Celery**. There are zero `celery` references in the settings; `"django_q"` is the installed app at `src/paperless/settings.py:110`.
> **Commit:** `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`).
> **Scope of findings:** Everything below is scoped to *exactly this commit*. In particular, the negative findings (no `PaperlessTask` model, no `/api/tasks` endpoint) are true here and must **not** be generalized to later paperless-ngx releases that add task-tracking features.
> **Methodology:** Run-first. The system was **built and run** in its canonical configuration, real ingestion work was triggered through its **real entry points**, and the answers below are written from **observed runtime output**. Every claim is tied to a `file:line` reference and/or to captured command output. Statements that are inferred rather than observed are explicitly labelled **[INFERRED]**.

---

## 1. Environment & how it was run

All observation happened **inside the user-provided canonical Docker container** (the bare host sandbox runs Python 3.13 with no Redis and no `django_q`, and is *not* a valid observation environment).

| Property | Value | Evidence |
|---|---|---|
| Container image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` | `docker ps` (container id `f2b765873095`, name `pngx`) |
| OS | Debian GNU/Linux 11 (bullseye) | `/etc/os-release` in container |
| Python | 3.9.23 | canonical runtime; `Dockerfile:18` = `FROM python:3.9-slim-bullseye as main-app` |
| Django | 4.0.4 | `requirements.txt:38` `django==4.0.4`; `python3 -c "import django"` |
| Django-Q | 1.3.9 | `requirements.txt:37` `django-q==1.3.9`; `django_q.VERSION == (1, 3, 9)` |
| redis (python client) | 3.5.3 | `requirements.txt:84` `redis==3.5.3` |
| hiredis | 2.0.0 | `requirements.txt:44` |
| channels / channels-redis / daphne | 3.0.4 / 3.4.0 / 3.0.2 | `requirements.txt:23,22,31` |
| djangorestframework | 3.13.1 | `requirements.txt:39` |
| Redis server | v6.0.16, on `localhost:6379` | `redis-server --version`; `redis-cli ping` → `PONG` |
| Database | SQLite at `/app/data/db.sqlite3` | `settings.DATABASES["default"]["ENGINE"] == django.db.backends.sqlite3` |
| Repo path in container | `/app` (source under `/app/src`) | — |
| Runtime user | `testuser` (uid 1000) — the canonical CI/runtime user | setup instructions |

**How the stack was brought up.** The default execution model is a single container running **three** Supervisord-managed long-running processes plus Redis and the database (`docker/supervisord.conf`). I reproduced the exact `command=` lines from `docker/supervisord.conf`, run as `testuser`, backgrounded with logs to files (the repo lives at `/app`, not the production `/usr/src/paperless`, so `gunicorn.conf.py` is at `/app/gunicorn.conf.py`):

```bash
# Redis was already running (redis-cli ping -> PONG). The three processes, each mirroring docker/supervisord.conf:
docker exec -u testuser -d f2b765873095 bash -c 'cd /app/src && nohup python3 manage.py qcluster                                        > /tmp/blitzy_obs/qcluster.log 2>&1'   # [program:scheduler] command :29
docker exec -u testuser -d f2b765873095 bash -c 'cd /app/src && nohup gunicorn -c /app/gunicorn.conf.py paperless.asgi:application       > /tmp/blitzy_obs/gunicorn.log 2>&1'   # [program:gunicorn]   command :11
docker exec -u testuser -d f2b765873095 bash -c 'cd /app/src && nohup python3 manage.py document_consumer                               > /tmp/blitzy_obs/consumer.log 2>&1'   # [program:consumer]   command :20
```

All commands shown in this document were executed inside the container via `docker exec [-u testuser] f2b765873095 bash -c '<command>'`. For readability the `docker exec … bash -c` wrapper is omitted from the per-question command blocks below; the `<command>` shown is what actually produced the output.

> **Note on `supervisord.conf` vs. this run.** `docker/supervisord.conf` declares `user=paperless` and the production path `/usr/src/paperless/gunicorn.conf.py` (`docker/supervisord.conf:11`). In this container the canonical user is `testuser` and the repo is at `/app`. The **`command=` program invocations are identical** to what Supervisord would run — `python3 manage.py qcluster`, `python3 manage.py document_consumer`, and `gunicorn … paperless.asgi:application` — which is the aspect that matters for observing background processing.

---

## 2. TL;DR — one direct answer per question

- **Q1 (what actually happens):** A real `async_task("documents.tasks.consume_file", …)` serializes the job and pushes it onto the **Redis** broker; a `qcluster` worker dequeues it and runs `consume_file(...)` [`src/documents/tasks.py:184`], which delegates to `Consumer().try_consume_file(...)` [`src/documents/tasks.py:236`]. On success it returns `"Success. New document id {} created"` [`src/documents/tasks.py:247`]; on failure it raises `ConsumerError` [`src/documents/tasks.py:249`]. **Observed:** the job went waiting → active → a `django_q_task` row with `success=True, result='Success. New document id 2 created'`.
- **Q2 (services):** **Three** Supervisord processes — `gunicorn` (ASGI web+WebSocket) [`docker/supervisord.conf:11`], `document_consumer` (dir watcher) [`docker/supervisord.conf:20`], `qcluster` (the Django-Q worker cluster that *executes* jobs and *schedules* recurring ones) [`docker/supervisord.conf:29`] — plus **Redis** (broker + Channels layer) and the **database** (holds `django_q_task`). **Observed:** the `qcluster` banner spawned 11 workers + a monitor + a pusher + a guard.
- **Q3 (how jobs appear):** In **three** places — the serialized payload in the **Redis** broker list `django_q:paperless:q`, the **`qcluster` worker log** (`processing […]` / `Processed […]`), and **WebSocket `status_updates`** progress frames — and, after completion, as a **row in `django_q_task`**.
- **Q4 (waiting vs. active):** **Waiting** = enqueued in the **Redis** list (`LLEN django_q:paperless:q` ≥ 1, no worker yet); **active** = executing in a `qcluster` worker (`processing […]`, `STARTING`→`WORKING`→`SUCCESS`). Because the broker is **Redis, not the ORM**, the `django_q_ormq` table (`OrmQ`) stays **empty**. **Observed** before/during/after with the queue length and the `django_q_task` count.
- **Q5 (where state is stored):** Finished-task state lives in the **database**, table **`django_q_task`**, model **`django_q.models.Task`** (with `Success`/`Failure` as *proxies* over the same table). Waiting state is **not** in the DB — it is in **Redis**.
- **Q6 (after-the-fact):** Use Django-Q's own facilities — `django_q.tasks.result(task_id)` / `fetch(task_id)`, the `Task`/`Success`/`Failure` models, and the **Django admin** (Successful/Failed/Scheduled task pages). Success → `success=True`, `result="Success. New document id N created"`; failure → a `Failure` row (`success=False`) whose `result` holds the `ConsumerError` traceback. **Negative findings:** there is **no `PaperlessTask` model** and **no `/api/tasks` endpoint / `TaskViewSet`** at this commit.
- **Q7 (where jobs are triggered):** By `django_q.tasks.async_task(...)` at four call-site families — the directory watcher [`src/documents/management/commands/document_consumer.py:86`], the REST upload [`src/documents/views.py:523`], IMAP mail [`src/paperless_mail/mail.py:336`], and bulk edits [`src/documents/bulk_edit.py:18,31,47,63,87`] — plus recurring `django_q.tasks.schedule(...)` registrations in three migrations.

---

## Q1 — Runtime behavior: what actually happens when a document is ingested

**Direct answer.** A real `async_task("documents.tasks.consume_file", …)` call serializes the job (a pickled, signed "task package") and pushes it onto the **Redis** broker list `django_q:paperless:q`. A `qcluster` worker process dequeues it, imports the dotted-path function, and runs `consume_file(path, …, task_id=None)` [`src/documents/tasks.py:184`]. With barcodes disabled (the default — `CONSUMER_ENABLE_BARCODES=False`, `src/paperless/settings.py:502`), `consume_file` **delegates to `Consumer().try_consume_file(...)`** [`src/documents/tasks.py:236`], the multi-stage ingestion pipeline in `src/documents/consumer.py:180`. On success it returns the string `"Success. New document id {} created".format(document.pk)` [`src/documents/tasks.py:247`]; if the returned document is null it raises `ConsumerError` [`src/documents/tasks.py:249`], and any pipeline precondition failure raises `ConsumerError` from `Consumer._fail()` [`src/documents/consumer.py:81`]. **The run below took the main (non-barcode) path.**

**Command & complete output (the job going waiting → active → stored).** qcluster was momentarily stopped so the enqueue is visible before a worker takes it (see Q4); here is the worker log once qcluster was restarted and picked the job up, plus the resulting row:

```text
# tail of /tmp/blitzy_obs/qcluster.log after the worker picked up simple.pdf
04:41:55 [Q] INFO Q Cluster saturn-harry-crazy-failed running.
04:41:55 [Q] INFO Process-1:1 processing [simple.pdf]
[2026-07-08 04:41:55,922] [INFO] [paperless.consumer] Consuming simple.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-_9shwp_m/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-08 04:41:56,358] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 04:41:57,175] [INFO] [paperless.consumer] Document 2026-07-08 simple consumption finished
04:41:57 [Q] INFO Process-1:1 stopped doing work
04:41:57 [Q] INFO Processed [simple.pdf]
04:41:57 [Q] INFO recycled worker Process-1:1
04:41:57 [Q] INFO Process-1:14 ready for work at 23652
```

```text
# The stored result row (django_q.models.Task), queried via manage.py shell -- see Q5 for the exact query
  id      = ed11253d19304d44b95f52b89e493451
  name    = simple.pdf
  func    = documents.tasks.consume_file
  success = True
  result  = 'Success. New document id 2 created'
  started = 2026-07-08 04:41:17.495739+00:00
  stopped = 2026-07-08 04:41:57.178480+00:00
```

**Cause → effect.**
- The worker line `Process-1:1 processing [simple.pdf]` is emitted by the Django-Q cluster **because** a worker dequeued the package and is about to call the target function; the worker calls `f(*task["args"], **task["kwargs"])` at `django_q/cluster.py:432` (observed in the failure traceback in Q4). **Therefore** `consume_file("/app/src/../consume/simple.pdf")` runs.
- `[paperless.consumer] Consuming simple.pdf` is logged from inside `Consumer.try_consume_file()` **because** `consume_file` delegated to it [`src/documents/tasks.py:236`].
- The two `convert-im6.q16 …` lines and the `Thumbnail generation with ImageMagick failed, falling back to ghostscript` warning are a real, non-fatal detail of this environment: ImageMagick's `policy.xml` forbids rasterizing PDFs, **so** the parser falls back to ghostscript for the thumbnail — consumption still succeeds.
- `Processed [simple.pdf]` is emitted **because** `consume_file` returned normally; the return value `"Success. New document id 2 created"` [`src/documents/tasks.py:247`] is stored as the Task's `result` (observed above). The document primary key `2` in the string equals the new `Document.pk`.

**Distinct condition — the barcode short-circuit (not taken here).** If `CONSUMER_ENABLE_BARCODES` were on [`src/paperless/settings.py:502`] and a separator barcode were found, `consume_file` splits the file, deletes the original, broadcasts a `SUCCESS` status via `async_to_sync(get_channel_layer().group_send)("status_updates", …)` [`src/documents/tasks.py:226`], and returns `"File successfully split"` [`src/documents/tasks.py:233`] **without** calling `try_consume_file`. In this default run barcodes are **off** (confirmed: `settings.CONSUMER_ENABLE_BARCODES == False`), so the main path ran and the SUCCESS/`document_id` came from the pipeline, not the split branch.

---

## Q2 — Services / processes involved in background processing

**Direct answer.** Three Supervisord-managed processes plus Redis and the database:

1. **`gunicorn`** — `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` [`docker/supervisord.conf:11`]. The ASGI web **and** WebSocket server (serves the REST API that enqueues jobs and the `ws/status/` progress socket).
2. **`document_consumer`** — `python3 manage.py document_consumer` [`docker/supervisord.conf:20`]. The directory watcher that **enqueues** ingestion jobs.
3. **`qcluster`** — `python3 manage.py qcluster` [`docker/supervisord.conf:29`]. The Django-Q worker **cluster** that **executes** enqueued jobs **and** runs the **scheduler** for recurring `Schedule`s.
4. **Redis** — the Django-Q **broker** (`Q_CLUSTER["redis"]`, `src/paperless/settings.py:456`) **and** the Channels layer backend (`CHANNEL_LAYERS` → `channels_redis.core.RedisChannelLayer`, `src/paperless/settings.py:178`).
5. **Database** (SQLite by default) — holds the `django_q_task` table (finished-task state) and the `django_q_schedule` table (recurring registrations).

**Command & complete output — the `qcluster` startup banner** (shows the internal process roles):

```text
# head of /tmp/blitzy_obs/qcluster.log on a clean start
04:36:21 [Q] INFO Q Cluster arkansas-apart-oregon-leopard starting.
04:36:21 [Q] INFO Process-1:1 ready for work at 23094
04:36:21 [Q] INFO Process-1:2 ready for work at 23095
04:36:21 [Q] INFO Process-1:3 ready for work at 23096
04:36:21 [Q] INFO Process-1:4 ready for work at 23097
04:36:21 [Q] INFO Process-1:5 ready for work at 23098
04:36:21 [Q] INFO Process-1:6 ready for work at 23099
04:36:21 [Q] INFO Process-1:7 ready for work at 23100
04:36:21 [Q] INFO Process-1:8 ready for work at 23101
04:36:21 [Q] INFO Process-1:9 ready for work at 23102
04:36:21 [Q] INFO Process-1:10 ready for work at 23103
04:36:21 [Q] INFO Process-1:11 ready for work at 23104
04:36:21 [Q] INFO Process-1:12 monitoring at 23105
04:36:21 [Q] INFO Process-1 guarding cluster arkansas-apart-oregon-leopard
04:36:21 [Q] INFO Process-1:13 pushing tasks at 23106
04:36:21 [Q] INFO Q Cluster arkansas-apart-oregon-leopard running.
```

```text
# gunicorn + consumer startup (their log files)
[2026-07-08 04:36:29 +0000] [23107] [INFO] Starting gunicorn 20.1.0
[2026-07-08 04:36:29 +0000] [23107] [INFO] Listening at: http://0.0.0.0:8000 (23107)
[2026-07-08 04:36:29 +0000] [23107] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 04:36:29 +0000] [23107] [INFO] Server is ready. Spawning workers
[2026-07-08 04:36:29,301] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

**Command & complete output — the three processes actually running** (via `/proc`, since `ps` is not installed):

```text
[document_consumer] PID 23113
[gunicorn]        PID 23107   (master) + worker PIDs 23127, 23128, …
[qcluster]        PID 23081   (guard/sentinel) + workers 23094-23104 + monitor 23105 + pusher 23106
```

**Command & complete output — Redis is the broker/channel backend, DB backend:**

```text
$ redis-cli ping
PONG
$ python3 -c "…print settings…"
DB ENGINE        = django.db.backends.sqlite3
DB NAME          = /app/src/../data/db.sqlite3
Q_CLUSTER        = {'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://localhost:6379'}
```

**Scheduler role — recurring tasks are registered as Django-Q `Schedule`s in DB migrations (not in `apps.py`).** Observed 4 `Schedule` rows after migration:

```text
# Schedule.objects.all()
  id=1 func=documents.tasks.train_classifier          name='Train the classifier'      type=H  next_run=2026-07-08 05:03:30
  id=2 func=documents.tasks.index_optimize            name='Optimize the index'        type=D  next_run=2026-07-09 04:03:30
  id=3 func=documents.tasks.sanity_check              name='Perform sanity check'      type=W  next_run=2026-07-15 04:03:30
  id=4 func=paperless_mail.tasks.process_mail_accounts name='Check all e-mail accounts' type=I  next_run=2026-07-08 04:33:32
```

These correspond exactly to the migration registrations:
- `documents/migrations/1001_auto_20201109_1636.py:10-14` — `schedule("documents.tasks.train_classifier", …, schedule_type=Schedule.HOURLY)`; and `:15-19` — `index_optimize` `DAILY`.
- `documents/migrations/1004_sanity_check_schedule.py:10-14` — `sanity_check` `WEEKLY`.
- `paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` — `process_mail_accounts` `MINUTES` (`minutes=10`); its body is `process_mail_accounts` [`src/paperless_mail/tasks.py:11`].

**Cause → effect.** The `qcluster` banner shows the cluster's internal division of labor: **11 worker processes** (`Process-1:1 … 11 ready for work`, matching `Q_CLUSTER["workers"]=11`), one **monitor** (`Process-1:12 monitoring`) that writes finished results into `django_q_task`, one **pusher** (`Process-1:13 pushing tasks`) that reads the Redis broker and dispatches to workers, and the **guard/sentinel** (`Process-1 guarding cluster`) that supervises them. **Because** the same `qcluster` process also runs the scheduler, the `Schedule` whose `next_run` (id=4, 04:33:32) had already passed fired on its own during the run and produced an extra finished Task — direct evidence that `qcluster` both executes and schedules.

---


## Q3 — How background jobs appear once created

**Direct answer.** An enqueued job appears in **three** observable places, then in a fourth after it finishes:
1. In the **Redis broker** — the serialized task package sits in the list `django_q:paperless:q` while it waits.
2. In the **`qcluster` worker log** — `processing […]` when a worker takes it, `Processed […]` / `Failed […]` when it finishes.
3. As **WebSocket `status_updates` progress frames** pushed to connected browsers (`STARTING`→`WORKING`→`SUCCESS`/`FAILED`).
4. After completion, as a **row in `django_q_task`** (see Q5).

**(a) In the Redis broker.** Command and complete output, captured while a worker was momentarily unavailable (qcluster stopped) so the job is visibly parked:

```text
$ redis-cli LLEN django_q:paperless:q
1
$ redis-cli LRANGE django_q:paperless:q 0 -1
gAWVEQEAAAAAAAB9lCiMAmlklIwgZWQxMTI1M2QxOTMwNGQ0NGI5NWY1MmI4OWU0OTM0NTGUjARuYW1llIwKc2ltcGxlLnBkZpSMBGZ1bmOUjBxkb2N1bWVudHMudGFza3MuY29uc3VtZV9maWxllIwEYXJnc5SMHi9hcHAvc3JjLy4uL2NvbnN1bWUvc2ltcGxlLnBkZpSFlIwGa3dhcmdzlH2UjBBvdmVycmlkZV90YWdfaWRzlE5zjAdzdGFydGVklIwIZGF0ZXRpbWWUjAhkYXRldGltZZSTlEMKB-oHCAQpEQeQe5RoDowIdGltZXpvbmWUk5RoDowJdGltZWRlbHRhlJOUSwBLAEsAh5RSlIWUUpSGlFKUdS4:1whK61:D9VICZ3UOecOlbSMkkCApGzanyNEZAYDKx3V-zkUGOc
```

That payload is a Django-`signing`-signed, pickled dict (`<b64-pickle>:<timestamp>:<signature>`). Decoding the pickle part yields the human-readable job:

```text
# base64.urlsafe_b64decode(payload_before_first_colon) -> pickle.loads(...)
Deserialized Django-Q task package (the WAITING job in Redis):
  'id': 'ed11253d19304d44b95f52b89e493451'
  'name': 'simple.pdf'
  'func': 'documents.tasks.consume_file'
  'args': ('/app/src/../consume/simple.pdf',)
  'kwargs': {'override_tag_ids': None}
  'started': datetime.datetime(2026, 7, 8, 4, 41, 17, 495739, tzinfo=datetime.timezone.utc)
```

**(b) In the `qcluster` worker log** (from Q1): `Process-1:1 processing [simple.pdf]` … `Processed [simple.pdf]`.

**(c) As WebSocket `status_updates` frames.** An authenticated client was connected to `ws://localhost:8000/ws/status/` (a session cookie was minted for `admin`, since `StatusConsumer.connect()` raises `DenyConnection` for unauthenticated clients — `src/paperless/consumers.py:14-15`). Complete, unedited frames received during the `simple.pdf` ingest:

```text
[ws] cookie=sessionid=<REDACTED_SESSION_ID>   # ephemeral local admin session token, redacted; the frames below are the evidence
[ws] connecting to ws://localhost:8000/ws/status/
[ws] CONNECTED (handshake accepted)
[ws 04:41:55] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",            "document_id": null}
[ws 04:41:55] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 20,  "max_progress": 100, "status": "WORKING",  "message": "parsing_document",     "document_id": null}
[ws 04:41:56] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 70,  "max_progress": 100, "status": "WORKING",  "message": "generating_thumbnail", "document_id": null}
[ws 04:41:57] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 90,  "max_progress": 100, "status": "WORKING",  "message": "parse_date",          "document_id": null}
[ws 04:41:57] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 95,  "max_progress": 100, "status": "WORKING",  "message": "save_document",        "document_id": null}
[ws 04:41:57] FRAME: {"filename": "simple.pdf", "task_id": "2e8f9812-162c-4042-889c-bfbdcccb6bb5", "current_progress": 100, "max_progress": 100, "status": "SUCCESS",  "message": "finished",            "document_id": 2}
```

**Cause → effect.** Each frame carries exactly the keys of `Consumer._send_progress()`'s payload — `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, `document_id` [`src/documents/consumer.py:64-72`]. The browser progress bar advances **because** `_send_progress()` calls `async_to_sync(self.channel_layer.group_send)("status_updates", {"type": "status_update", "data": payload})` [`src/documents/consumer.py:73`], which the Channels/Redis layer fans out to every member of the `status_updates` group; `StatusConsumer.status_update()` then forwards `json.dumps(event["data"])` to the socket [`src/paperless/consumers.py:33`]. The six stages map 1:1 to the pipeline call sites: `STARTING` [`consumer.py:202`], `WORKING 20 parsing_document` [`consumer.py:259`], `WORKING 70 generating_thumbnail` [`consumer.py:264`], `WORKING 90 parse_date` [`consumer.py:274`], `WORKING 95 save_document` [`consumer.py:294`], `SUCCESS 100 finished` with `document_id=2` [`consumer.py:375`].

---

## Q4 — Waiting (queued/pending) vs. active (started/running)

**Direct answer.** **Waiting** work sits in the **Redis** broker list `django_q:paperless:q` — enqueued but not yet taken by any worker. **Active** work is a job currently executing inside a `qcluster` worker process. Because the broker is **Redis, not the Django ORM** (`Q_CLUSTER["redis"]="redis://localhost:6379"`, `src/paperless/settings.py:456`; there is no `"orm"` key), waiting jobs are visible **in Redis**, and the Django-Q `OrmQ` table (`django_q_ormq`) stays **empty**. The transition waiting→active is observable as: the job leaving the Redis list, the `qcluster` log line `processing […]`, and the WebSocket sequence `STARTING`→`WORKING`→`SUCCESS`/`FAILED`.

To make "waiting" unambiguous I stopped the `qcluster` worker, dropped a file (so `document_consumer` enqueued it with no worker to take it), then restarted `qcluster`. **Do NOT** read `OrmQ` as the waiting queue for this configuration — it is the wrong broker; the correct evidence is the Redis list length.

**BEFORE (all three processes up, consume dir empty):**

```text
$ python3 -c "…counts…"
Task.count   = 9
OrmQ.count   = 0
Document.count = 1
$ redis-cli LLEN django_q:paperless:q
0
```

**DURING — WAITING (qcluster stopped; `simple.pdf` dropped into `/app/consume`):**

```text
# /tmp/blitzy_obs/consumer.log — the enqueue
[2026-07-08 04:41:17,493] [INFO] [paperless.management.consumer] Adding /app/src/../consume/simple.pdf to the task queue.
04:41:17 [Q] INFO Enqueued 1
# Redis broker — the job is parked, waiting for a worker
$ redis-cli LLEN django_q:paperless:q
1
# DB is UNCHANGED, and OrmQ is STILL 0 (waiting lives in Redis, not the ORM)
Task.count   = 9
OrmQ.count   = 0
Document.count = 1
```

**DURING — ACTIVE (qcluster restarted; a worker takes the parked job):**

```text
04:41:55 [Q] INFO Process-1:1 processing [simple.pdf]        <- now ACTIVE inside a worker
[2026-07-08 04:41:55,922] [INFO] [paperless.consumer] Consuming simple.pdf
# WebSocket, in the same window:
[ws 04:41:55] FRAME: {… "status": "STARTING", "message": "new_file" …}
[ws 04:41:55] FRAME: {… "status": "WORKING",  "message": "parsing_document" …}
```

**AFTER (job finished):**

```text
$ redis-cli LLEN django_q:paperless:q
0
Task.count     = 10          # +1: the finished row appeared
OrmQ.count     = 0           # still empty
Document.count = 2           # +1: the new document
# newest django_q_task row:
  id=ed11253d19304d44b95f52b89e493451  name=simple.pdf  func=documents.tasks.consume_file
  success=True  result='Success. New document id 2 created'
[ws 04:41:57] FRAME: {… "status": "SUCCESS", "message": "finished", "document_id": 2}
```

**Cause → effect.**
- While `qcluster` was down, `document_consumer`'s `async_task` still pushed the package to Redis (`Enqueued 1`), **so** `LLEN django_q:paperless:q` became `1` — that is the *waiting* state. No `django_q_task` row exists yet (`Task.count` unchanged at 9) **because** a row is written only when the monitor records a *finished* task.
- `OrmQ.count` stayed `0` throughout **because** `Q_CLUSTER` selects the Redis broker [`settings.py:456`]; the `django_q_ormq` table is only used when the broker is the ORM. **Therefore** `OrmQ` is the wrong place to look for waiting work here.
- The **same task id** `ed11253d19304d44b95f52b89e493451` appears in the waiting Redis payload (Q3) *and* in the finished row — proving one job traversed waiting → active → stored.
- When `qcluster` restarted, its pusher read the list and handed the package to `Process-1:1`, which logged `processing [simple.pdf]` — the *active* state — and the WebSocket immediately began emitting `STARTING`/`WORKING`.

**Progress-stage enumeration (every distinct status a job broadcasts):** `STARTING` [`consumer.py:202`]; `WORKING` at 20/70/90/95 [`consumer.py:259,264,274,294`]; `SUCCESS` [`consumer.py:375`]; `FAILED` [`consumer.py:79`] (immediately followed by `raise ConsumerError` at `consumer.py:81`). All six were observed live (STARTING→…→SUCCESS in Q3; STARTING→FAILED in Q6's failure run).

---


## Q5 — Where task state is stored

**Direct answer.** *Finished*-task state is stored in the **database**, in the table **`django_q_task`**, modeled by **`django_q.models.Task`** (with `Success` and `Failure` as **proxy models** over that same table). In this deployment the database is **SQLite** at `/app/data/db.sqlite3` (`DATABASES["default"]["ENGINE"] = django.db.backends.sqlite3`). *Waiting/queued* state is **not** in the database — it lives in **Redis** (the configured Django-Q broker, `Q_CLUSTER["redis"]="redis://localhost:6379"`, `src/paperless/settings.py:456`). Realtime *progress* state is transient and lives only on the `status_updates` Channels group (Redis-backed), never persisted.

**Observed table name** (via `manage.py shell`):

```text
$ python3 manage.py shell -c "from django_q.models import Task, Success, Failure; \
print('Task.db_table =', Task._meta.db_table); \
print('Success.proxy =', Success._meta.proxy, '| Failure.proxy =', Failure._meta.proxy); \
print('fields =', [f.name for f in Task._meta.fields])"
Task.db_table = django_q_task
Success.proxy = True | Failure.proxy = True
fields = ['id', 'name', 'func', 'hook', 'args', 'kwargs', 'result', 'group', 'started', 'stopped', 'success', 'attempt_count']
```

**Authoritative schema** (read directly from the SQLite file with Python's `sqlite3`, because the `sqlite3` CLI and `manage.py dbshell` are not available in the container):

```text
$ python3 -c "import sqlite3; c=sqlite3.connect('/app/data/db.sqlite3'); \
print(c.execute(\"SELECT sql FROM sqlite_master WHERE name='django_q_task'\").fetchone()[0])"
CREATE TABLE "django_q_task" ("name" varchar(100) NOT NULL, "func" varchar(256) NOT NULL, "hook" varchar(256) NULL, "args" text NULL, "kwargs" text NULL, "result" text NULL, "started" datetime NOT NULL, "stopped" datetime NOT NULL, "success" bool NOT NULL, "id" varchar(32) NOT NULL PRIMARY KEY, "group" varchar(100) NULL, "attempt_count" integer NOT NULL)
```

**A real stored row** — the `simple.pdf` success from Q1/Q4:

```text
$ python3 -c "import sqlite3; c=sqlite3.connect('/app/data/db.sqlite3'); \
print(c.execute('SELECT id,name,func,success,result FROM django_q_task ORDER BY stopped DESC LIMIT 1').fetchone())"
('ed11253d19304d44b95f52b89e493451', 'simple.pdf', 'documents.tasks.consume_file', 1, 'Success. New document id 2 created')
```

For contrast, the ORM-broker queue table and the schedule table also exist but are used differently:

```text
django_q_ormq  : CREATE TABLE (... "key" varchar(100), "payload" text, "lock" datetime NULL)  -- ORM-broker queue; EMPTY here
django_q_schedule: CREATE TABLE (... "func", "schedule_type" varchar(1), "next_run", "task", "name", "minutes", "cron", "cluster")
```

**Cause → effect.** The `id` column is `varchar(32)` **because** Django-Q stores the task id as a 32-char hex (`uuid().hex`), which is why every observed `Task.id` (e.g. `ed11253d…`, `f3f7dd8f…`) is 32 hex chars with no dashes. The `success` boolean column is what lets `Success` and `Failure` be **proxy** models over the **same** `django_q_task` table — `Success.proxy == True` and `Failure.proxy == True` above — partitioning rows by `success=True`/`success=False` (proven numerically in Q6: 11 + 2 = 13). The `result` column is `text` **because** it holds either the success string `"Success. New document id N created"` [`src/documents/tasks.py:247`] or, on failure, the full `ConsumerError` traceback (Q6). Waiting work is absent from all of these tables **because** `Q_CLUSTER` has no `"orm"` key and points at Redis [`settings.py:456`]; the row for a job materializes **only after** the monitor process records its completion — this is exactly why `Task.count` did not change while the job sat waiting in Q4.

---


## Q6 — After-the-fact inspection: how to tell what happened to a completed job

**Direct answer.** After a job finishes you determine its outcome from **Django-Q's own facilities** — there is **no paperless-owned task model and no paperless tasks API at this commit**. The tools are: `django_q.tasks.result(task_id)` and `django_q.tasks.fetch(task_id)`; the `Task` / `Success` / `Failure` ORM models; and the **Django admin**, where Django-Q registers changelists for *Successful tasks*, *Failed tasks*, and *Scheduled tasks*. A **success** yields `Task.success == True` with `result == "Success. New document id N created"` [`src/documents/tasks.py:247`]; a **failure** yields a `Failure` row (`success == False`) whose `result` holds the full `ConsumerError` traceback.

**(a) `result()` and `fetch()`** (from `manage.py shell`) — success job from Q1/Q4 (`document_id 3`, the API upload):

```text
$ python3 manage.py shell -c "from django_q.tasks import result, fetch; \
t = fetch('f3f7dd8f009a4beaa7207badf06cdd19'); \
print('fetch ->', t, '| success =', t.success); \
print('result ->', repr(result('f3f7dd8f009a4beaa7207badf06cdd19'))); \
print('result(missing) ->', repr(result('00000000000000000000000000000000')))"
fetch -> <Task: test_with_bom.pdf> | success = True
result -> 'Success. New document id 3 created'
result(missing) -> None
```

`result()` returns `None` for an id that has not run (or does not match); it can optionally block for a bounded time. The failed duplicate job (Q6 failure path) looked up the same way:

```text
$ python3 manage.py shell -c "from django_q.tasks import result, fetch; \
t = fetch('5f54b5f6cc314815a2d90cc95d92293e'); \
print('fetch ->', t, '| success =', t.success); \
print('result ->', repr(result('5f54b5f6cc314815a2d90cc95d92293e')))"
fetch -> <Task: simple.pdf> | success = False
result -> 'Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file(\n  File "/app/src/documents/consumer.py", line 213, in try_consume_file\n    self.pre_check_duplicate()\n  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate\n    self._fail(\n  File "/app/src/documents/consumer.py", line 81, in _fail\n    raise ConsumerError(f"{self.filename}: {log_message or message}")\ndocuments.consumer.ConsumerError: simple.pdf: Not consuming simple.pdf: It is a duplicate.\n'
```

**(b) ORM: `Success` / `Failure` partition the table.** Command and complete output:

```text
$ python3 manage.py shell -c "from django_q.models import Task, Success, Failure, OrmQ; \
print('Task   =', Task.objects.count()); \
print('Success=', Success.objects.count()); \
print('Failure=', Failure.objects.count()); \
print('OrmQ   =', OrmQ.objects.count()); \
print('newest Failure:', Failure.objects.first().name, Failure.objects.first().success)"
Task   = 13
Success= 11
Failure= 2
OrmQ   = 0
newest Failure: simple.pdf False
```

`11 + 2 == 13` — this is the runtime proof that `Success` and `Failure` are proxy models over the one `django_q_task` table, split by the `success` boolean (Q5).

**(c) Django admin.** With an authenticated `admin` session, Django-Q's registered changelists respond:

```text
$ # GET each admin page with a minted admin sessionid cookie (requests)
GET /admin/django_q/success/   -> 200   ("Select Successful task to change"; body lists test_with_bom.pdf and simple.pdf)
GET /admin/django_q/failure/   -> 200   ("Select Failed task to change")
GET /admin/django_q/schedule/  -> 200   ("Select Scheduled task to change")
GET /admin/django_q/ormq/      -> 404   (Not Found)

$ python3 manage.py shell -c "from django.contrib import admin; from django_q.models import Task, Success, Failure, Schedule, OrmQ; \
print({m.__name__: (m in admin.site._registry) for m in [Task, Success, Failure, Schedule, OrmQ]})"
{'Task': False, 'Success': True, 'Failure': True, 'Schedule': True, 'OrmQ': False}
```

The `ormq` admin returns **404** and `OrmQ` is unregistered **because** `django_q`'s admin registers the Queued-Tasks (OrmQ) page only when the broker is the ORM (`Conf.ORM` is set); here the broker is Redis, so it is absent — extra corroboration of Q4/Q5.

**(d) `task_id` provenance (must be included).** The two live entry points differ in how a job's id is assigned, which changes how you *find* the job afterward:

```text
# API upload — views.py generates its OWN uuid4 BEFORE enqueue:
src/documents/views.py:521   task_id = str(uuid.uuid4())
src/documents/views.py:523-533   async_task("documents.tasks.consume_file", temp_filename, ... task_id=task_id, task_name=...)
# Directory watcher — NO task_id passed:
src/documents/management/commands/document_consumer.py:86   async_task("documents.tasks.consume_file", filepath, override_tag_ids=..., task_name=...)
```

Observed consequence — the self-assigned uuid4 is **NOT** the stored `Task.id`. Grounded in the Django-Q 1.3.9 `async_task` source (`/usr/local/lib/python3.9/site-packages/django_q/tasks.py`): `opt_keys` (`:23-35`) does not list `task_id`, so it is **not** treated as an option; `tag = uuid()` (`:38`) and `task["id"] = tag[1]` (`:41`) means Django-Q **always** generates the stored id itself; the leftover `task_id` flows into `task["kwargs"]` (`:64`) and is passed to the *function* `consume_file(task_id=...)`, becoming `Consumer.task_id` — the WebSocket progress-correlation id.

| Path | HTTP/response | self-assigned uuid4 (progress `task_id`) | stored Django-Q `Task.id` (hex) | differ? |
|------|---------------|------------------------------------------|---------------------------------|---------|
| API upload | `200`, body `"OK"` | `d86abe24-b7a7-4dae-b2c6-3afc73c6d8b3` | `f3f7dd8f009a4beaa7207badf06cdd19` | **yes** |
| Dir watcher | (no HTTP) | `2e8f9812-162c-4042-889c-bfbdcccb6bb5` | `ed11253d19304d44b95f52b89e493451` | **yes** |

So to look a job up with `result()`/`fetch()` you use the **hex `Task.id`**, located by name/time (or from `async_task`'s return value — which **neither** call site captures). This refines the naive "the API caller already knows the id" idea: the HTTP body is only `"OK"`, and the uuid4 the caller made governs only the realtime progress channel, not the stored record.

### Negative findings (VERIFIED EMPTY at commit `542221a38dff`; report, do not "fix")

**N1 — No `PaperlessTask` model.** Repo-wide grep returns nothing (exit code 1):

```text
$ cd /app/src && grep -rn "PaperlessTask" . ; echo "exit=$?"
exit=1
```

**N2 — No `/api/tasks` endpoint and no `TaskViewSet`.** The DRF `DefaultRouter` registers only six routes; there is no `tasks` route:

```text
$ sed -n '29,35p' src/paperless/urls.py
api_router = DefaultRouter()
api_router.register(r"correspondents", CorrespondentViewSet)
api_router.register(r"document_types", DocumentTypeViewSet)
api_router.register(r"documents", UnifiedSearchViewSet)
api_router.register(r"logs", LogViewSet, basename="logs")
api_router.register(r"tags", TagViewSet)
api_router.register(r"saved_views", SavedViewViewSet)

$ cd /app/src && grep -rn 'register(r"tasks"\|TaskViewSet' . ; echo "exit=$?"
exit=1
```

**N3 — `OrmQ` table stays empty (broker is Redis, not ORM).**

```text
$ python3 manage.py shell -c "from django_q.models import OrmQ; from django.conf import settings; \
print('OrmQ.count =', OrmQ.objects.count()); \
print('Q_CLUSTER redis =', settings.Q_CLUSTER.get('redis')); \
print('Q_CLUSTER has orm key =', 'orm' in settings.Q_CLUSTER)"
OrmQ.count = 0
Q_CLUSTER redis = redis://localhost:6379
Q_CLUSTER has orm key = False
```

These three findings are scoped to **exactly** commit `542221a38dff06361e07976452f9aea24d210542` and must **not** be generalized to later paperless-ngx releases that add a `PaperlessTask` model / `/api/tasks` API.

**Cause → effect.** After-the-fact inspection relies entirely on Django-Q **because** paperless (at this commit) defines no task-status model and exposes no tasks endpoint (N1, N2). `result()`/`fetch()` read from `django_q_task` (Q5); `Success`/`Failure` are proxies partitioning that table by `success`; the admin surfaces them as *Successful*/*Failed*/*Scheduled* changelists. Waiting work is invisible to all of these **because** it never enters the ORM — it sits in Redis until a worker consumes it (N3, Q4).

---


## Q7 — Code origin of triggers: where jobs are enqueued, and the code that sends them to the queue

**Direct answer.** Jobs are sent to the queue by **`django_q.tasks.async_task(...)`** calls. Every ingestion job is `async_task("documents.tasks.consume_file", ...)`; bulk operations enqueue `async_task("documents.tasks.bulk_update_documents", ...)`. Recurring jobs are *registered* (not enqueued at call time) by **`django_q.tasks.schedule(...)`** inside DB migrations. Task functions are referenced by their **dotted string path**, not imported at the call site — so the code that "sends to the queue" is the `async_task`/`schedule` call, and Django-Q resolves and runs the function later inside a `qcluster` worker.

**(1) Directory watcher — the primary ingestion entry point.** `src/documents/management/commands/document_consumer.py` (import `:13`, call `:86`, block `:84-91`):

```text
$ sed -n '13p' src/documents/management/commands/document_consumer.py
from django_q.tasks import async_task
$ sed -n '84,91p' src/documents/management/commands/document_consumer.py
    try:
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
            filepath,
            override_tag_ids=tag_ids if tag_ids else None,
            task_name=os.path.basename(filepath)[:100],
        )
```

No `task_id` is passed → Django-Q assigns the stored `Task.id`. This was the enqueue **observed live** in Q4 (`Adding … to the task queue.` → `Enqueued 1`).

**(2) REST API upload — `POST /api/documents/post_document/`.** `src/documents/views.py` (import `:28`, `task_id` `:521`, call `:523-533`, response `:535`):

```text
$ sed -n '28p' src/documents/views.py
from django_q.tasks import async_task
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

This path self-generates `task_id` (Q6 provenance). This was the enqueue **observed live** in Q1 SUCCESS #2 (`200`, body `"OK"`).

**(3) IMAP mail ingestion.** `src/paperless_mail/mail.py` (import `:11`, call `:336`):

```text
$ sed -n '11p' src/paperless_mail/mail.py
from django_q.tasks import async_task
$ sed -n '336,340p' src/paperless_mail/mail.py
                async_task(
                    "documents.tasks.consume_file",
                    path=temp_filename,
                    override_filename=pathvalidate.sanitize_filename(
                        att.filename,
```

*(Not exercised live — no IMAP mail account is configured in the default container; identified by reading. Labeled **inferred-from-source** for the enqueue mechanics, but the target function `consume_file` is the same one exercised in Q1.)*

**(4) Bulk operations — five call sites.** `src/documents/bulk_edit.py` (import `:4`; calls `:18,:31,:47,:63,:87`):

```text
$ grep -n 'async_task' src/documents/bulk_edit.py
4:from django_q.tasks import async_task
18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
```

*(All five enqueue `documents.tasks.bulk_update_documents` [`src/documents/tasks.py:270`]. Not exercised live; identified by reading — labeled **inferred-from-source**.)*

**(5) Recurring/scheduled registrations — `django_q.tasks.schedule(...)` in DB migrations.** These do not call `async_task`; they create `Schedule` rows that the `qcluster` scheduler later turns into enqueued jobs. All four rows were **observed** in the DB (Q2). The registrations:

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
    )
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

The mail schedule's task body is `process_mail_accounts` [`src/paperless_mail/tasks.py:11`]. **Observed** proof that the scheduler is live: during the Q2 baseline, a `process_mail_accounts` run fired on its own (its `next_run` had passed), incrementing `Task.count` from 8 to 9 and producing the `Success` row `No new documents were added.` seen in Q6(b).

**Cause → effect.** Each `async_task(...)` call **serializes** the target dotted path + args/kwargs into a package and **pushes it onto the Redis list** `django_q:paperless:q` (the waiting state, Q4); the `Enqueued 1` log line comes from Django-Q's own `async_task` [`django_q/tasks.py:74`]. A `qcluster` worker later **imports and calls** the dotted function — for ingestion that is `consume_file` [`src/documents/tasks.py:184`], reached via `res = f(*task["args"], **task["kwargs"])` [`django_q/cluster.py:432`], which is exactly the frame at the top of the Q6 failure traceback. `schedule(...)` differs: it writes a `Schedule` row, and the `qcluster` sentinel enqueues a fresh `async_task` each time `next_run` arrives — which is why recurring jobs appear in `django_q_task` even though no application code called `async_task` for them.

---


## Evidence appendix (longer raw captures)

**A1 — The full duplicate-failure worker traceback** (the complete text stored verbatim in the `Failure` row's `result`, `success=False`), captured from `/tmp/blitzy_obs/qcluster.log`:

```text
04:48:14 [Q] INFO Process-1:4 processing [simple.pdf]
[2026-07-08 04:48:14,473] [ERROR] [paperless.consumer] Not consuming simple.pdf: It is a duplicate.
04:48:14 [Q] ERROR Failed [simple.pdf] - simple.pdf: Not consuming simple.pdf: It is a duplicate. : Traceback (most recent call last):
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
04:48:14 [Q] INFO recycled worker Process-1:4
```

**A2 — The two WebSocket frames of the failure path** (STARTING then FAILED, skipping every WORKING stage because the duplicate check fires immediately after STARTING):

```text
[ws] FRAME: {"filename": "simple.pdf", "task_id": "6b2a79cc-2113-484c-b81a-6c6073602d1f", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",                "document_id": null}
[ws] FRAME: {"filename": "simple.pdf", "task_id": "6b2a79cc-2113-484c-b81a-6c6073602d1f", "current_progress": 100, "max_progress": 100, "status": "FAILED",   "message": "document_already_exists", "document_id": null}
```

`FAILED` is emitted by `Consumer._fail()` → `_send_progress(100, 100, "FAILED", message)` [`src/documents/consumer.py:79`] with `message = MESSAGE_DOCUMENT_ALREADY_EXISTS` [`src/documents/consumer.py:37`], immediately before `raise ConsumerError` [`src/documents/consumer.py:81`].

**A3 — The API-upload success (SUCCESS #2), complete request/response and worker lines:**

```text
$ python3 -c "import requests; r=requests.post('http://localhost:8000/api/documents/post_document/', \
auth=('admin','admin'), files={'document': ('test_with_bom.pdf', open('.../test_with_bom.pdf','rb'), 'application/pdf')}); \
print('status', r.status_code); print('body', r.text)"
status 200
body "OK"

# qcluster.log
04:44:39 [Q] INFO Process-1:3 processing [test_with_bom.pdf]
[2026-07-08 04:44:44,xxx] [INFO] [paperless.consumer] Document 2020-07-02 test_with_bom consumption finished
04:44:44 [Q] INFO Processed [test_with_bom.pdf]

# stored row
  id=f3f7dd8f009a4beaa7207badf06cdd19  name=test_with_bom.pdf  success=True  result='Success. New document id 3 created'
```

---

## Coverage checklist

Every question, its named items, the observed evidence, and the primary `file:line`.

| Question / named item | Direct answer | Observed evidence | Primary `file:line` |
|---|---|---|---|
| **Q1** runtime behavior | `async_task`→Redis→`qcluster` worker→`consume_file`→`try_consume_file`; returns success string | worker log `processing/Processed [simple.pdf]`; stored row `result='Success. New document id 2 created'` | `tasks.py:184,236,247` |
| Q1 barcode short-circuit condition | not taken (barcodes off by default) | `settings.CONSUMER_ENABLE_BARCODES == False` | `settings.py:502`; `tasks.py:226,233` |
| Q1 failure branch | raises `ConsumerError` | duplicate traceback (A1) | `tasks.py:249`; `consumer.py:81` |
| **Q2** gunicorn | ASGI web + WebSocket | `Listening at: http://0.0.0.0:8000` | `supervisord.conf:11` |
| Q2 document_consumer | dir watcher, enqueues | `Using inotify to watch directory …/consume` | `supervisord.conf:20` |
| Q2 qcluster | executes + schedules | banner: 11 workers + monitor + pusher + guard | `supervisord.conf:29` |
| Q2 Redis | broker + Channels layer | `redis-cli ping → PONG`; `Q_CLUSTER['redis']` | `settings.py:456,178` |
| Q2 database | holds `django_q_task` | `ENGINE=…sqlite3` | `settings.py` DATABASES |
| Q2 scheduler / 4 Schedules | HOURLY/DAILY/WEEKLY/10-min | 4 `Schedule` rows; mail schedule fired live | migrations `1001`,`1004`,`0002`; `paperless_mail/tasks.py:11` |
| **Q3** Redis broker appearance | serialized payload in `django_q:paperless:q` | `LLEN=1`; decoded pickle dict | `settings.py:456` |
| Q3 worker log appearance | `processing […]`/`Processed […]` | qcluster.log | `django_q/cluster.py:432` |
| Q3 WebSocket frames | 6-stage progress JSON | STARTING→WORKING×4→SUCCESS | `consumer.py:56-73`; `consumers.py:33` |
| Q3 payload keys | filename,task_id,current/max_progress,status,message,document_id | frames captured | `consumer.py:64-72` |
| **Q4** waiting | in Redis list, `OrmQ` empty | `LLEN=1` while `Task.count` unchanged, `OrmQ=0` | `settings.py:456` |
| Q4 active | in `qcluster` worker | `Process-1:1 processing […]` | `cluster.py:432` |
| Q4 before/during/after | queue+table counts at each boundary | 0/9/0 → 1/9/0 → 0/10/0 | — |
| Q4 progress stages | STARTING/WORKING/SUCCESS/FAILED | all observed live | `consumer.py:202,259,264,274,294,375,79` |
| **Q5** table + model | `django_q_task`, `django_q.models.Task` | `Task._meta.db_table`; `CREATE TABLE` dump | `consumer/settings` + schema |
| Q5 Success/Failure proxies | proxy over same table, split by `success` | `proxy=True`; `11+2=13` | — |
| Q5 waiting not in DB | in Redis | `OrmQ.count=0` | `settings.py:456` |
| **Q6** `result()`/`fetch()` | read stored result / Task | success + failure lookups | `django_q.tasks` |
| Q6 admin pages | Successful/Failed/Scheduled 200; ormq 404 | GET results + registry dict | — |
| Q6 task_id provenance | API self-assigns uuid4 (progress id) ≠ hex `Task.id`; watcher passes none | both ids captured, differ | `views.py:521`; `document_consumer.py:86`; `django_q/tasks.py:38,41,64` |
| Q6 no `PaperlessTask` | grep empty (exit 1) | N1 | — |
| Q6 no `/api/tasks`/`TaskViewSet` | router has 6 routes, no tasks | N2 | `urls.py:29-35` |
| Q6 `OrmQ` empty | broker is Redis | N3 | `settings.py:456` |
| **Q7** dir watcher enqueue | `async_task("…consume_file", …)` | live enqueue (Q4) | `document_consumer.py:86` |
| Q7 API enqueue | `async_task(..., task_id=…)` | live upload (Q1 #2) | `views.py:523-533` |
| Q7 mail enqueue | `async_task("…consume_file", path=…)` | read (not run; no mail account) | `mail.py:336` |
| Q7 bulk enqueue ×5 | `async_task("…bulk_update_documents", …)` | grep of 5 sites | `bulk_edit.py:18,31,47,63,87` |
| Q7 scheduled registrations | `schedule(...)` in 3 migrations | 4 Schedule rows | `1001`,`1004`,`0002` |

---

## Inferred-vs-observed callouts

Everything in Q1–Q6 is **observed at runtime** (commands + output shown). The following are the only statements **inferred from source rather than exercised live**, and they are labelled as such in-line above:

- **[INFERRED] Q7(3) IMAP mail enqueue** (`src/paperless_mail/mail.py:336`) — not exercised because no IMAP mail account is configured in the default container. The `async_task("documents.tasks.consume_file", path=…)` call was confirmed by reading; the *target* function `consume_file` is the same one exercised live in Q1.
- **[INFERRED] Q7(4) bulk-operation enqueues** (`src/documents/bulk_edit.py:18,31,47,63,87`) — the five `async_task("documents.tasks.bulk_update_documents", …)` sites were confirmed by grep; the bulk path was not driven end-to-end. The enqueue mechanism (`async_task` → Redis → `qcluster`) is identical to the observed ingestion path.
- **[INFERRED] Q1 barcode short-circuit** (`src/documents/tasks.py:226,233`) — not taken, because `CONSUMER_ENABLE_BARCODES == False` was **observed**. The branch's behavior (`"File successfully split"`, SUCCESS broadcast) is read from source; the observed run took the main `try_consume_file` path.

All task ids, counts, log lines, WebSocket frames, schema dumps, admin responses, and negative-finding command results elsewhere in this document are **observed** output captured inside the canonical container.
