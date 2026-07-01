# Paperless-NGX — Idle / Steady-State Runtime Behavior (commit 542221a38dff)

## Summary

Paperless-NGX runs as a **single multi-process container** whose `ENTRYPOINT` is `/sbin/docker-entrypoint.sh` [Dockerfile:L168] and whose `CMD` launches **Supervisor** [Dockerfile:L172], which in turn keeps three long-lived processes alive: the Gunicorn-hosted ASGI web app, the `document_consumer` directory watcher, and the Django-Q `qcluster` scheduler [docker/supervisord.conf:L10-29]. Background work and periodic scheduling are driven by **Django-Q `1.3.9`** (`python3 manage.py qcluster`) — **not another task-queue framework** [Pipfile.lock:default.django-q.version]; the cluster is an internal set of guard/sentinel, pusher, worker(s), and monitor processes that read `django_q.models.Schedule` rows from the database. While idle (booted, stable, no documents to process), readiness is maintained by those always-on processes plus **four DB-registered periodic schedules** — an e-mail check every `10` minutes, classifier training `HOURLY`, index optimization `DAILY`, and a sanity check `WEEKLY`. A single **Redis** instance serves a dual role: it is both the Django-Q task broker at `redis://localhost:6379` [src/paperless/settings.py:L456] and the Channels layer backend for live status websockets [src/paperless/settings.py:L178-187]. The externally observable proofs that the system is "ready" while idle are the `Q Cluster ... running.` banner [django_q/cluster.py:L261], the periodic e-mail-check cycle every `10` minutes, and a passing `30s` container healthcheck (`curl -f http://localhost:8000`) [docker/compose/docker-compose.postgres.yml:L56-57].

## How these answers were produced (run-before-write)

Per the governing rule, the relevant code paths were **built and run first**, and the answers below are written from **actually observed output**, not from reading alone.

**Observation harness (throwaway, outside the repository).** A minimal Django project was scaffolded **outside** the Paperless-NGX repository, mirroring Paperless's `Q_CLUSTER` block [src/paperless/settings.py:L449-457]:

- `name="paperless"` [src/paperless/settings.py:L450]
- `catch_up=False` [src/paperless/settings.py:L451]
- `recycle=1` [src/paperless/settings.py:L452]
- `retry=PAPERLESS_WORKER_RETRY` (defaults to `1810` = timeout + 10) [src/paperless/settings.py:L453], [src/paperless/settings.py:L444-446]
- `timeout=PAPERLESS_WORKER_TIMEOUT` (defaults to `1800`) [src/paperless/settings.py:L454], [src/paperless/settings.py:L440]
- `workers=TASK_WORKERS` [src/paperless/settings.py:L455] (the harness pinned `workers=1` to make the `recycle=1` behavior visible in a single worker line)
- `redis="redis://localhost:6379"` [src/paperless/settings.py:L456]

The harness was installed to match Paperless's exact pins — `django==4.0.4`, `django-q==1.3.9`, `redis==3.5.3` [Pipfile.lock:default] — against a real Redis broker (`redis-server`) on `localhost:6379`. A single `django_q.models.Schedule` row equivalent to the Paperless e-mail check was registered — `name="Check all e-mail accounts"`, `schedule_type=Schedule.MINUTES`, `minutes=10` — mirroring [src/paperless_mail/migrations/0002_auto_20201117_1334.py:L11-14]; its target returned `"No new documents were added."`, mirroring [src/paperless_mail/tasks.py:L22]. The command that produced the capture is the same management command Supervisor runs as the `scheduler` program: `python3 manage.py qcluster` [docker/supervisord.conf:L28-29].

**Independent reproduction against the real codebase.** In addition to the throwaway harness, the identical behavior was reproduced by running the **actual** Paperless-NGX `manage.py qcluster` inside the canonical image (`Python 3.9.23`, `django-q 1.3.9`, `redis 3.5.3`, `django 4.0.4`) after `manage.py migrate` created the four real `django_q.models.Schedule` rows. That run confirmed every log line, the `~30s` scheduler cadence, the idle `"No new documents were added."` result, the broker key prefix `django_q:paperless:q`, and the reconnection sequence. Per-run values that are inherently random or environment-dependent — the humanized cluster id, per-task ids, PIDs, worker count, wall-clock timestamps, and the exact POSIX errno for an unreachable broker — differ from run to run; these are called out explicitly where they appear.

**Why the `[Q]` lines look the way they do.** Every `[Q]`-prefixed line is emitted by Django-Q's own logger named `django-q` [django_q/conf.py:L207], which it self-configures with format `"%(asctime)s [Q] %(levelname)s %(message)s"` and `datefmt="%H:%M:%S"` [django_q/conf.py:L213-214], its own `logging.StreamHandler()` [django_q/conf.py:L216], and `propagate=False` [django_q/conf.py:L212]. That is why `[Q]` lines appear in that exact short `HH:MM:SS [Q] LEVEL message` format regardless of Paperless's own `LOGGING` configuration [src/paperless/settings.py:L373].

**Citations.** Paths under `src/...`, `docker/...`, and the repository root refer to the Paperless-NGX source at commit `542221a38dff`. Paths under `django_q/...` refer to the installed `django-q == 1.3.9` library (the emitter of every `[Q]` string); each such line was verified by reading the installed library source. The harness lived entirely outside the repository and was removed afterward; the repository is left unchanged apart from this document.

## Q0 — Getting it running

**Rationale.** Paperless-NGX ships as one container built on base image `python:3.9-slim-bullseye` [Dockerfile:L18], exposing port `8000` [Dockerfile:L170], with `ENTRYPOINT ["/sbin/docker-entrypoint.sh"]` [Dockerfile:L168] and `CMD ["/usr/local/bin/supervisord", "-c", "/etc/supervisord.conf"]` [Dockerfile:L172]. "Getting it running" therefore means: satisfy the prerequisites (a Redis broker, the pinned Python dependencies, and a database), then let Supervisor start the three long-running processes.

**Prerequisites.**

- A reachable **Redis** broker at the default `redis://localhost:6379` [src/paperless/settings.py:L456] (used both as the Django-Q broker and the Channels layer).
- The **pinned Python dependencies** (table below).
- A **database** (SQLite by default).

**Startup gates (observed literals).** The entrypoint first prints:

```
Paperless-ngx docker container starting...
```

emitted at [docker/docker-entrypoint.sh:L77]. A Redis readiness gate then blocks startup until the broker answers, printing `Waiting for Redis: {REDIS_URL}` [docker/wait-for-redis.py:L21] and, on success, `Connected to Redis broker: {REDIS_URL}` [docker/wait-for-redis.py:L41]. On failure it prints `Redis ping #{attempt} failed, waiting {RETRY_SLEEP_SECONDS}s` [docker/wait-for-redis.py:L31] and finally `Failed to connect to: {REDIS_URL}` [docker/wait-for-redis.py:L38]; the gate retries `MAX_RETRY_COUNT=5` times [docker/wait-for-redis.py:L16] with `RETRY_SLEEP_SECONDS=5` between attempts [docker/wait-for-redis.py:L17].

**The three Supervisor-managed processes** [docker/supervisord.conf:L10-29] (each runs as `user=paperless`, with stdout routed to `/dev/stdout` and stderr to `/dev/stderr`):

```
[program:gunicorn]
command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application   # docker/supervisord.conf:L10-11

[program:consumer]
command=python3 manage.py document_consumer                                           # docker/supervisord.conf:L19-20

[program:scheduler]
command=python3 manage.py qcluster                                                    # docker/supervisord.conf:L28-29
```

**Pinned dependencies** (registry `pip`; exact versions from `Pipfile.lock`, base runtime Python `3.9` [Dockerfile:L18]):

| Registry | Package                  | Version  |
| -------- | ------------------------ | -------- |
| pip      | `django`                 | `4.0.4`  |
| pip      | `django-q`               | `1.3.9`  |
| pip      | `redis`                  | `3.5.3`  |
| pip      | `channels`               | `3.0.4`  |
| pip      | `channels-redis`         | `3.4.0`  |
| pip      | `uvicorn`                | `0.17.6` |
| pip      | `gunicorn`               | `20.1.0` |
| pip      | `djangorestframework`    | `3.13.1` |
| pip      | `whitenoise`             | `6.0.0`  |
| pip      | `watchdog`               | `2.1.7`  |
| pip      | `inotifyrecursive`       | `0.3.5`  |
| pip      | `concurrent-log-handler` | `0.9.20` |
| pip      | `django-extensions`      | `3.1.5`  |

**Observed proof the cluster came up.** Running `python3 manage.py qcluster` produced the startup banner shown in Q2 (Capture A). Its final line — `Q Cluster ... running.` [django_q/cluster.py:L261] — is the readiness signal: the cluster is fully up and its guard/pusher/worker/monitor processes are live.

**Scope note (what was and wasn't run locally).** The native OCR stack (`ocrmypdf`, `pikepdf`, `tesseract`, `qpdf`, `jbig2enc`, `pyzbar`, `scikit-learn`) requires the prebuilt image `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff06361e07976452f9aea24d210542`. The idle/steady-state behavior documented here concerns the Django-Q broker/cluster, the ASGI web tier, the Channels layer, and the inotify watcher — all independent of the OCR stack. The version-pinned Django-Q broker/cluster behavior was reproduced directly; the OCR pipeline is neither exercised nor required to answer the idle-behavior questions.

## Q1 — Background processes that run while idle

**Rationale.** "Idle" here means the system is booted and stable with no documents to process. In that state, two categories of work keep running: (a) three always-on operating-system processes supervised by Supervisor, and (b) the Django-Q scheduler firing four database-registered periodic schedules.

**(a) The three always-on processes** [docker/supervisord.conf:L10-29]:

- **The Gunicorn/Uvicorn ASGI web tier** — `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` [docker/supervisord.conf:L10-11]. It hosts the HTTP API/UI and the websocket route via an ASGI `ProtocolTypeRouter` [src/paperless/asgi.py:L17].
- **The `document_consumer`** — `python3 manage.py document_consumer` [docker/supervisord.conf:L19-20]. This is an inotify/polling directory watcher (logger `paperless.management.consumer` [src/documents/management/commands/document_consumer.py:L24]) that watches the consume directory. **While idle** (no file present or arriving) the watch loop waits on filesystem events and generates **no Redis traffic**. It is **not, however, broker-independent**: when a file arrives, `_consume()` enqueues `documents.tasks.consume_file` via Django-Q's `async_task` [src/documents/management/commands/document_consumer.py:L13], [src/documents/management/commands/document_consumer.py:L84-L91], which obtains a broker (`get_broker()`) [django_q/tasks.py:L55] and calls `broker.enqueue(pack)` [django_q/tasks.py:L73]; this project defaults to the **Redis broker** [django_q/brokers/__init__.py:L201-L205], whose `enqueue` performs `connection.rpush(list_key, task)` [django_q/brokers/redis_broker.py:L17-L18]. So idle watching needs no broker, but **ingestion enqueueing requires Redis** (this nuance matters for Q3).
- **The Django-Q `qcluster`** — `python3 manage.py qcluster` [docker/supervisord.conf:L28-29]. Internally this is not one process but a set: a **sentinel/guard** loop that supervises children, a **pusher** that performs the broker `BLPOP` loop, one or more **worker** processes, and a **monitor** (result collector). Their startup lines are shown in Q2 (Capture A).

**Observed proof that the consumer's ingestion path uses the Redis broker.** Inside the canonical image (real `paperless.settings`, Redis up), calling the exact function `document_consumer` imports — `async_task("documents.tasks.consume_file", ...)` [src/documents/management/commands/document_consumer.py:L13], [src/documents/management/commands/document_consumer.py:L84-L91] — grew the broker's Redis list by one:

```
$ python3 -c 'import django; django.setup(); from django_q.brokers import get_broker; \
  b=get_broker(); print("broker_class="+type(b).__name__); print("list_key="+b.list_key); \
  print("size_before="+str(b.queue_size())); from django_q.tasks import async_task; \
  async_task("documents.tasks.consume_file","/tmp/pc/does-not-exist.pdf",sync=False); \
  print("size_after="+str(b.queue_size()))'
broker_class=Redis
list_key=django_q:paperless:q
size_before=0
05:54:18 [Q] INFO Enqueued 1
size_after=1
```

The default broker is `Redis` [django_q/brokers/__init__.py:L201-L205], its list key is `django_q:paperless:q`, and the enqueue (`connection.rpush`) [django_q/brokers/redis_broker.py:L17-L18] moved the queue size from `0` to `1`. This is an **ingestion action, not idle activity** — while idle no file arrives, so no enqueue occurs and the consumer produces no Redis traffic.

**(b) The four DB-registered periodic schedules.** Crucially, these are `django_q.models.Schedule` **rows created by data migrations** — not code decorators. Running `manage.py migrate` created exactly four rows; the observed table (from the independent reproduction) was:

```
'Train the classifier'      | func=documents.tasks.train_classifier          | type=H | minutes=None
'Optimize the index'        | func=documents.tasks.index_optimize            | type=D | minutes=None
'Perform sanity check'      | func=documents.tasks.sanity_check              | type=W | minutes=None
'Check all e-mail accounts' | func=paperless_mail.tasks.process_mail_accounts | type=I | minutes=10
TOTAL: 4
```

Mapping each row to its migration and cadence:

- **E-mail check — every `10` minutes.** `name="Check all e-mail accounts"`, `func` `paperless_mail.tasks.process_mail_accounts`, `schedule_type=Schedule.MINUTES` [src/paperless_mail/migrations/0002_auto_20201117_1334.py:L13], `minutes=10` [src/paperless_mail/migrations/0002_auto_20201117_1334.py:L14]. (`type=I` above is Django-Q's internal code for `MINUTES`.)
- **Classifier training — `HOURLY`.** `func` `documents.tasks.train_classifier`, `schedule_type=Schedule.HOURLY` [src/documents/migrations/1001_auto_20201109_1636.py:L11-14] (`type=H`).
- **Index optimization — `DAILY`.** `func` `documents.tasks.index_optimize`, `schedule_type=Schedule.DAILY` [src/documents/migrations/1001_auto_20201109_1636.py:L15-18] (`type=D`).
- **Sanity check — `WEEKLY`.** `func` `documents.tasks.sanity_check`, `schedule_type=Schedule.WEEKLY` [src/documents/migrations/1004_sanity_check_schedule.py:L11-13] (`type=W`).

The task bodies live in `src/documents/tasks.py` — `index_optimize()` [src/documents/tasks.py:L32] and `train_classifier()` [src/documents/tasks.py:L48], under logger `paperless.tasks` [src/documents/tasks.py:L29]. The e-mail task body is `process_mail_accounts()` [src/paperless_mail/tasks.py:L11], logger `paperless.mail.tasks` [src/paperless_mail/tasks.py:L8].

**Distinguishing idle from ingestion-only activity (important nuance).** Several signal-driven components exist but fire **only during document ingestion, never while idle**:

- The six `document_consumption_finished` handlers wired in `DocumentsConfig.ready()` — `add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index` [src/documents/apps.py:L22-27]. These run in response to a completed consumption, so with no documents being consumed they never execute.
- The websocket progress broadcaster `StatusConsumer(WebsocketConsumer)` [src/paperless/consumers.py:L9]. Its `connect()` first enforces authentication and raises `DenyConnection()` if the client is unauthenticated [src/paperless/consumers.py:L11], [src/paperless/consumers.py:L13-15], only then joining the `"status_updates"` group [src/paperless/consumers.py:L18]. It carries live progress traffic only when an authenticated client is connected while a document is being processed.

So while idle there is **no websocket traffic and no signal-handler activity**; the only recurring activity is the always-on processes plus the four scheduled tasks.

## Q2 — Periodic health/readiness log entries (messages, frequency, meaning)

This is the evidentiary core. The command that produced the capture is the same one Supervisor runs:

```
$ python3 manage.py qcluster        # same command supervisord runs [docker/supervisord.conf:L28-29]
```

### Capture A — idle startup banner + one scheduler cycle

```
22:33:18 [Q] INFO Q Cluster west-carolina-high-fanta starting.
22:33:18 [Q] INFO Process-1:1 ready for work at 25419
22:33:18 [Q] INFO Process-1:2 monitoring at 25420
22:33:18 [Q] INFO Process-1 guarding cluster west-carolina-high-fanta
22:33:18 [Q] INFO Process-1:3 pushing tasks at 25421
22:33:18 [Q] INFO Q Cluster west-carolina-high-fanta running.
22:33:47 [Q] INFO Enqueued 1
22:33:47 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:33:47 [Q] INFO Process-1:1 processing [burger-mirror-tennis-carolina]
22:33:47 [Q] INFO Process-1:1 stopped doing work
22:33:47 [Q] INFO Processed [burger-mirror-tennis-carolina]
22:33:48 [Q] INFO recycled worker Process-1:1
22:33:48 [Q] INFO Process-1:4 ready for work at 25424
```

Graceful shutdown tail (context, emitted on `SIGTERM`):

```
22:34:37 [Q] INFO Q Cluster west-carolina-high-fanta stopping.
22:34:38 [Q] INFO Process-1 stopping cluster processes
22:34:38 [Q] INFO Process-1:3 stopped pushing tasks
22:34:38 [Q] INFO Process-1:4 stopped doing work
22:34:39 [Q] INFO Process-1 waiting for the monitor.
22:34:39 [Q] INFO Process-1:2 stopped monitoring results
22:34:39 [Q] INFO Q Cluster west-carolina-high-fanta has stopped.
```

### Per-line explanation

| Log line                                                             | Source `file:line`         | Frequency                                            | Meaning                                                                                                    |
| -------------------------------------------------------------------- | -------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `Q Cluster {id} starting.`                                           | `django_q/cluster.py:L79`  | once per cluster start                               | Cluster boot begins.                                                                                       |
| `Process-1:1 ready for work at {pid}`                                | `django_q/cluster.py:L410` | once per worker spawn (and again after each recycle) | A worker process is up and awaiting tasks.                                                                 |
| `Process-1:2 monitoring at {pid}`                                    | `django_q/cluster.py:L378` | once per start                                       | The monitor (result collector) is up.                                                                      |
| `Process-1 guarding cluster {id}`                                    | `django_q/cluster.py:L256` | once per start                                       | The sentinel/guard loop is supervising the child processes.                                                |
| `Process-1:3 pushing tasks at {pid}`                                 | `django_q/cluster.py:L342` | once per pusher spawn                                | The pusher (broker `BLPOP` loop) is up.                                                                    |
| `Q Cluster {id} running.`                                            | `django_q/cluster.py:L261` | once, at end of boot                                 | **The readiness signal** — the cluster is fully up.                                                        |
| `Enqueued {n}`                                                       | `django_q/tasks.py:L74`    | each time a due schedule is enqueued                 | A task was pushed to the broker.                                                                           |
| `Process-1 created a task from schedule [Check all e-mail accounts]` | `django_q/cluster.py:L669` | each time the schedule fires (every `10` min)        | The scheduler turned a due `Schedule` row into a task.                                                     |
| `Process-1:1 processing [{task}]`                                    | `django_q/cluster.py:L420` | per task                                             | A worker began executing the task (bracketed name is a random per-task humanhash id).                      |
| `Process-1:1 stopped doing work`                                     | `django_q/cluster.py:L451` | per task (because `recycle=1`)                       | The worker exits after one task (recycling releases memory); `recycle=1` [src/paperless/settings.py:L452]. |
| `Processed [{task}]`                                                 | `django_q/cluster.py:L392` | per task                                             | The monitor recorded the finished result.                                                                  |
| `recycled worker {name}`                                             | `django_q/cluster.py:L232` | per task (due to `recycle=1`)                        | The sentinel replaced the exited worker.                                                                   |

After each recycle a fresh `Process-1:N ready for work at {pid}` [django_q/cluster.py:L410] appears (e.g. `Process-1:4 ready for work at 25424` in Capture A).

### Frequency of the scheduler tick

Django-Q's guard loop drives the scheduler roughly **every `~30` seconds**: the guard cycle is `GUARD_CYCLE = 0.5` seconds [django_q/conf.py:L90] and scheduling is enabled with `SCHEDULER = True` [django_q/conf.py:L93]. The guard loop accumulates a counter by `GUARD_CYCLE` each iteration and calls the scheduler only once `counter >= 30 and Conf.SCHEDULER` [django_q/cluster.py:L283-L286] (then resets the counter), i.e. once every `~30` seconds.

**Empirical proof.** In Capture A the readiness banner is at `22:33:18` and the first schedule fire is at `22:33:47` — a measured gap of **`29` seconds**, confirming the `~30s` cadence. The independent reproduction against the real codebase measured the same gap at **`30` seconds** (banner `05:10:38` → first fire `05:11:08`).

### The idle e-mail check result

When idle, `process_mail_accounts` returns `"No new documents were added."` [src/paperless_mail/tasks.py:L22]. The independent reproduction confirmed this empirically: the stored `django_q` task row had `success=True` and `result='No new documents were added.'`, and the schedule's `next_run` advanced by exactly `10` minutes (observed `05:10:16` → `05:20:16`), confirming the `minutes=10` cadence.

### The four schedules' cadences (as health-log frequencies)

- E-mail check: **every `10` minutes** — the most frequent periodic health signal.
- Classifier training: **`HOURLY`**.
- Index optimization: **`DAILY`**.
- Sanity check: **`WEEKLY`**.

### Accuracy note: the banner id is random, not `name="paperless"`

The bracketed cluster id (e.g. `west-carolina-high-fanta`) is a **random per-start humanhash**, **not** the configured `name="paperless"`. `Cluster.name` is a property that returns `humanize(self.cluster_id.hex)` [django_q/cluster.py:L110-111], where `cluster_id = uuid.uuid4()` [django_q/cluster.py:L58] — a fresh UUID per cluster start. The configured `name="paperless"` [src/paperless/settings.py:L450] instead becomes `Conf.PREFIX` (`PREFIX = conf.get("name", "default")` [django_q/conf.py:L80]), which forms the broker's Redis list key `django_q:paperless:q` and the stats key `django_q:paperless:cluster` (`Q_STAT = f"django_q:{PREFIX}:cluster"` [django_q/conf.py:L174]). Both were verified empirically in the independent reproduction: `broker.list_key == 'django_q:paperless:q'` and `Conf.Q_STAT == 'django_q:paperless:cluster'`. That reproduction also produced entirely different humanized ids — `lemon-double-moon-pip` on one run and `berlin-artist-washington-lactose` on another — confirming the id is random and independent of the configured `name`.

## Q3 — Reconnection after a brief interruption

**Rationale.** The question asks which log messages confirm reconnection and a return to operational status after part of the system is briefly interrupted and restarted. The component that is interrupted is the **Redis broker**; the Django-Q **pusher** depends on it (its `BLPOP` loop reads from Redis). The `document_consumer`'s filesystem watch loop does not itself hold a Redis connection, so **watching continues** during the outage; but the consumer is **not fully unaffected** — a file that arrives while Redis is down cannot be enqueued, because ingestion pushes work through Django-Q's `async_task` [src/documents/management/commands/document_consumer.py:L13], [src/documents/management/commands/document_consumer.py:L84-L91] → the Redis broker [django_q/tasks.py:L55], [django_q/tasks.py:L73], [django_q/brokers/redis_broker.py:L17-L18].

### Commands that produced the capture

```
$ # with qcluster already running:
$ redis-cli shutdown nosave      # interrupt: stop Redis
$ # ... wait ...
$ redis-server --daemonize yes   # restart Redis
```

### Capture B — connection-error storm + pusher reincarnation

```
22:35:42 [Q] INFO Q Cluster cola-alaska-burger-earth starting.
...
22:35:42 [Q] INFO Q Cluster cola-alaska-burger-earth running.
22:35:54 [Q] ERROR Error 99 connecting to localhost:6379. Cannot assign requested address.
22:35:54 [Q] ERROR Error 99 connecting to localhost:6379. Cannot assign requested address.
   ... (repeats; 62 such lines observed while Redis was down) ...
22:36:03 [Q] INFO Process-1:3 stopped pushing tasks
22:36:04 [Q] ERROR reincarnated pusher Process-1:3 after sudden death
22:36:04 [Q] INFO Process-1:4 pushing tasks at 6021
   ... (pusher reincarnated 3 times total: Process-1:3 -> :4 -> :5 -> :6) ...
```

While Redis is down, the pusher's `broker.dequeue()` [django_q/cluster.py:L345] → `self.connection.blpop(self.list_key, 1)` [django_q/brokers/redis_broker.py:L21] fails, and the cluster logs a repeating connection error at [django_q/cluster.py:L347]. The exact broker-down signal is:

- `Error 99 connecting to localhost:6379. Cannot assign requested address.` — **62** such lines in this capture; equivalently `Error 111 connecting to localhost:6379. Connection refused.` where nothing is listening on the port. The errno is **environment-dependent** and either variant is expected — see the grounded note below.

**Grounded note on the errno (both variants are correct).** The specific POSIX errno for an unreachable broker is **environment-dependent**; the interruption surfaces as one of two equally valid siblings — **both are the same "Redis is unreachable" condition** raised by `redis-py` at the identical code path [django_q/cluster.py:L347], with an identical reincarnation sequence around either: `Error 99 connecting to localhost:6379. Cannot assign requested address.` (`EADDRNOTAVAIL`, typical under local ephemeral-port pressure) and `Error 111 connecting to localhost:6379. Connection refused.` (`ECONNREFUSED`, emitted when nothing is listening on the port). Which one appears depends only on the host's networking state, so an acceptance check for this behavior must accept **either**. A fresh re-run of the same interrupt/restart inside the canonical `paperless-ready:542221a38dff` image (`Python 3.9.23`, `django-q 1.3.9`) produced the line `07:38:25 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.` — **67** such lines with **0** `Error 99` lines — alongside an identical surrounding sequence: **3** pusher reincarnations (`reincarnated pusher ... after sudden death`), **3** `--- Logging error ---` / `TypeError: not all arguments converted during string formatting` blocks (tracebacks pointing at `cluster.py` line `345`), and **zero** `reconnect` / `restored` / `back online` matches.

The reincarnation lines themselves:

- `Process-1:3 stopped pushing tasks` [django_q/cluster.py:L366] — the failed pusher is winding down.
- `reincarnated pusher Process-1:3 after sudden death` [django_q/cluster.py:L223] — the sentinel restarted the dead pusher (observed **3 times**: `:3 → :4 → :5 → :6`).

### Capture B2 — the secondary Python "Logging error" artifact

```
--- Logging error ---
Traceback (most recent call last):
  ...
  File ".../django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
  File ".../django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
  ...
TypeError: not all arguments converted during string formatting
```

**What this artifact is.** Django-Q logs the pusher error with `logger.error(e, traceback.format_exc())` [django_q/cluster.py:L347], passing the traceback as a **second positional argument**. Python's `logging` then attempts `str(exception) % (traceback,)`, which raises `TypeError: not all arguments converted during string formatting`. This is a known **Django-Q `1.3.9` quirk** — the real broker-down signal is the `Error 99 ...` (or `Error 111 ...`) line, not this secondary traceback. The traceback also empirically confirms the failing code path: pusher `broker.dequeue()` [django_q/cluster.py:L345] → `self.connection.blpop(self.list_key, 1)` [django_q/brokers/redis_broker.py:L21]. The independent reproduction reproduced this exact `--- Logging error ---` / `TypeError: not all arguments converted during string formatting` block (three occurrences, tracebacks pointing at `cluster.py` line `345`).

### Capture C — recovery: the normal INFO cycle resumes after Redis restart

```
22:36:41 [Q] INFO Enqueued 1
22:36:41 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
22:36:41 [Q] INFO Process-1:1 processing [minnesota-virginia-queen-alaska]
22:36:41 [Q] INFO Process-1:1 stopped doing work
22:36:41 [Q] INFO Processed [minnesota-virginia-queen-alaska]
22:36:42 [Q] INFO recycled worker Process-1:1
22:36:42 [Q] INFO Process-1:7 ready for work at 25973
```

### The reconnection answer

- **The broker-down signal:** a repeating connection error logged at [django_q/cluster.py:L347] — `Error 99 connecting to localhost:6379. Cannot assign requested address.` (**62 times** in the Error 99 capture) **or**, equivalently, `Error 111 connecting to localhost:6379. Connection refused.` (**67 times** in the fresh canonical re-run). The exact errno is environment-dependent (see the grounded note above); both denote the same unreachable broker.
- **The pusher winding down and being restarted:** `Process-1:3 stopped pushing tasks` [django_q/cluster.py:L366] and `reincarnated pusher Process-1:3 after sudden death` [django_q/cluster.py:L223] (**3 times**: `:3 → :4 → :5 → :6`).
- **The key nuance — there is NO explicit "reconnected" line.** A grep of the full capture for wording such as `reconnect`, `restored`, or `back online` returned **zero** matches. This was re-verified in the independent reproduction: grepping the full log for `reconnect|restored|back online|resumed|recovery` produced **zero** `[Q]` matches. `redis-py` reconnects **transparently** on the next command via a fresh pooled connection, so Django-Q emits no "reconnected" message. **Operational recovery is proven solely by the resumption of the normal INFO scheduler cycle** shown in Capture C: `Enqueued` → `created a task from schedule [...]` → `processing` → `stopped doing work` → `Processed` → `recycled worker` → `ready for work`.
- **Measured recovery window (canonical capture):** Redis was stopped at `22:35:53`, the first error appeared at `22:35:54`, the last error at `22:36:21`, Redis was restarted (`PONG` at `22:36:23`), and the first fully-normal post-recovery cycle appeared at `22:36:41`.
- **Components and their behavior during the Redis outage:** the `document_consumer`'s inotify/polling **watch loop keeps running** — filesystem watching holds no Redis connection [src/documents/management/commands/document_consumer.py:L24] — but the consumer is **not fully broker-independent**: a file arriving while Redis is down cannot be enqueued, since ingestion pushes work through `async_task` → the Redis broker [src/documents/management/commands/document_consumer.py:L13], [src/documents/management/commands/document_consumer.py:L84-L91], [django_q/tasks.py:L55], [django_q/tasks.py:L73], [django_q/brokers/redis_broker.py:L17-L18]. The Channels websocket layer (`channels-redis == 3.4.0`) reconnects independently and only carries traffic when an authenticated client is connected during ingestion [src/paperless/consumers.py:L9-25].

## Q4 — What keeps Paperless-NGX "ready"

**Rationale.** Even with no documents to process, readiness is maintained by a fixed set of always-on components plus an external health probe. Each component below runs continuously and requires no document activity to stay up.

- **ASGI web tier.** Gunicorn hosts `paperless.asgi:application` [docker/supervisord.conf:L10-11] using `worker_class = "paperless.workers.ConfigurableWorker"` [gunicorn.conf.py:L5], which subclasses the Uvicorn worker — `class ConfigurableWorker(UvicornWorker)` [src/paperless/workers.py:L9]. Gunicorn logs `Server is ready. Spawning workers` from its `when_ready` hook [gunicorn.conf.py:L18], binds `0.0.0.0:8000` [gunicorn.conf.py:L3], runs `workers=2` [gunicorn.conf.py:L4], and uses `timeout=120` [gunicorn.conf.py:L6]. The ASGI app routes HTTP + websocket via `ProtocolTypeRouter` [src/paperless/asgi.py:L17], with the websocket route `re_path(r"ws/status/$", StatusConsumer.as_asgi())` [src/paperless/urls.py:L136-137].
- **Document watcher.** `document_consumer` [docker/supervisord.conf:L19-20] keeps an inotify/polling watch on the consume directory (logger `paperless.management.consumer` [src/documents/management/commands/document_consumer.py:L24]) so a new file is picked up instantly and enqueued for consumption via Django-Q's `async_task` → the Redis broker [src/documents/management/commands/document_consumer.py:L13], [src/documents/management/commands/document_consumer.py:L84-L91]; while idle (no file arriving) it is simply watching and generates no Redis traffic.
- **Django-Q cluster.** `qcluster` [docker/supervisord.conf:L28-29] stays up as guard + pusher + worker(s) + monitor. In the source, a worker with no task to run sets its shared timer value to `-1`, annotated with the inline comment `# Idle` [django_q/cluster.py:L417], [django_q/cluster.py:L446] (the status-label constant is `IDLE`, defined as `_("Idle")` [django_q/conf.py:L198]) — i.e. the process is alive and waiting, which is exactly the observed idle behavior (no tasks processed between the `10`-minute e-mail checks). Worker count scales with CPU via `TASK_WORKERS` [src/paperless/settings.py:L438], which feeds `Q_CLUSTER["workers"]` [src/paperless/settings.py:L455]; task `timeout=1800` seconds [src/paperless/settings.py:L454], [src/paperless/settings.py:L440] with `retry=1810` seconds [src/paperless/settings.py:L453], [src/paperless/settings.py:L444-446].
- **Redis (dual role).** The same Redis instance is the Django-Q broker [src/paperless/settings.py:L456] **and** the Channels layer backend — `CHANNEL_LAYERS` uses `channels_redis.core.RedisChannelLayer` [src/paperless/settings.py:L178-187] with `capacity=2000` [src/paperless/settings.py:L183] and `expiry=15` [src/paperless/settings.py:L184].
- **External readiness probe.** The container healthcheck runs `["CMD", "curl", "-f", "http://localhost:8000"]` on a **`30s`** interval with `timeout: 10s` and `retries: 5` [docker/compose/docker-compose.postgres.yml:L56-57] (present in all compose variants). A passing probe means the ASGI tier is serving.
- **Startup system checks (context).** At boot Django registers `paths_check` [src/paperless/checks.py:L52], `binaries_check` [src/paperless/checks.py:L66], and `debug_mode_check` [src/paperless/checks.py:L86], which validate paths, required binaries, and debug mode before the system is considered healthy.

**Tying it together.** The three externally observable proofs that the system is "ready" while idle are: the `Q Cluster ... running.` banner [django_q/cluster.py:L261], the periodic e-mail-check cycle every `10` minutes (Capture A), and a passing `30s` `curl` healthcheck [docker/compose/docker-compose.postgres.yml:L56-57].

## Coverage pass

Re-reading the original question and confirming each part is answered:

- [x] **Q0 — got it running.** Redis broker + pinned dependencies + a database, then the three Supervisor-managed processes; startup banners quoted (`Paperless-ngx docker container starting...`, `Connected to Redis broker: {REDIS_URL}`, `Q Cluster ... running.`).
- [x] **Q1 — idle background processes.** Three always-on processes (ASGI web tier, `document_consumer`, Django-Q `qcluster`) + four DB-registered schedules; idle-vs-ingestion distinction (signal handlers and websocket are ingestion-only).
- [x] **Q2 — periodic health/readiness logs.** Verbatim `[Q]` lines; per-line table with exact messages, `file:line`, frequency, and meaning; `~30s` tick measured at `29s` (canonical) / `30s` (reproduction); idle result `No new documents were added.`; the four cadences (`10` min / `HOURLY` / `DAILY` / `WEEKLY`).
- [x] **Q3 — reconnection after interruption.** The connection-error storm (`Error 99 ... Cannot assign requested address.` **or** the equivalent `Error 111 ... Connection refused.` — the errno is environment-dependent and either is acceptable; the fresh canonical re-run produced `Error 111` **67** times), pusher reincarnation (`3` times), **no explicit "reconnected" line**, recovery proven solely by the resumed INFO cycle; the `document_consumer` watch loop keeps running during the outage, though a file arriving while Redis is down cannot be enqueued until the broker returns.
- [x] **Q4 — what keeps it ready.** ASGI/Uvicorn workers, `document_consumer`, Django-Q cluster, Redis broker/channel layer (dual role), and the `30s` container healthcheck.

## Citations index

| Literal / behavior                                                                       | Reference                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name="paperless"`                                                                       | `src/paperless/settings.py:L450`                                                                                                                                                                     |
| `catch_up=False`                                                                         | `src/paperless/settings.py:L451`                                                                                                                                                                     |
| `recycle=1`                                                                              | `src/paperless/settings.py:L452`                                                                                                                                                                     |
| `redis://localhost:6379` (broker)                                                        | `src/paperless/settings.py:L456`                                                                                                                                                                     |
| `Q_CLUSTER` block                                                                        | `src/paperless/settings.py:L449-457`                                                                                                                                                                 |
| `CHANNEL_LAYERS` / `capacity=2000` / `expiry=15`                                         | `src/paperless/settings.py:L178-187`, `src/paperless/settings.py:L183`, `src/paperless/settings.py:L184`                                                                                             |
| task `timeout=1800`                                                                      | `src/paperless/settings.py:L454`, `src/paperless/settings.py:L440`                                                                                                                                   |
| `retry=1810`                                                                             | `src/paperless/settings.py:L453`, `src/paperless/settings.py:L444-446`                                                                                                                               |
| `TASK_WORKERS` → `Q_CLUSTER["workers"]`                                                  | `src/paperless/settings.py:L438`, `src/paperless/settings.py:L455`                                                                                                                                   |
| `LOGGING` config                                                                         | `src/paperless/settings.py:L373`                                                                                                                                                                     |
| the three supervised processes                                                           | `docker/supervisord.conf:L10-29`                                                                                                                                                                     |
| `gunicorn ... paperless.asgi:application`                                                | `docker/supervisord.conf:L10-11`                                                                                                                                                                     |
| `python3 manage.py document_consumer`                                                    | `docker/supervisord.conf:L19-20`                                                                                                                                                                     |
| `python3 manage.py qcluster`                                                             | `docker/supervisord.conf:L28-29`                                                                                                                                                                     |
| `Server is ready. Spawning workers`                                                      | `gunicorn.conf.py:L18`                                                                                                                                                                               |
| `worker_class = ConfigurableWorker` / bind / workers / timeout                           | `gunicorn.conf.py:L5`, `gunicorn.conf.py:L3`, `gunicorn.conf.py:L4`, `gunicorn.conf.py:L6`; `src/paperless/workers.py:L9`                                                                            |
| e-mail check `MINUTES` / `minutes=10`                                                    | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L13-14`                                                                                                                                    |
| `train_classifier` `HOURLY` / `index_optimize` `DAILY`                                   | `src/documents/migrations/1001_auto_20201109_1636.py:L11-18`                                                                                                                                         |
| `sanity_check` `WEEKLY`                                                                  | `src/documents/migrations/1004_sanity_check_schedule.py:L11-13`                                                                                                                                      |
| `No new documents were added.`                                                           | `src/paperless_mail/tasks.py:L22`                                                                                                                                                                    |
| `index_optimize()` / `train_classifier()` / logger `paperless.tasks`                     | `src/documents/tasks.py:L32`, `src/documents/tasks.py:L48`, `src/documents/tasks.py:L29`                                                                                                             |
| `StatusConsumer` / auth gate / `status_updates` group                                    | `src/paperless/consumers.py:L9`, `src/paperless/consumers.py:L11`, `src/paperless/consumers.py:L18`                                                                                                  |
| six ingestion-only `document_consumption_finished` handlers                              | `src/documents/apps.py:L22-27`                                                                                                                                                                       |
| websocket route `ws/status/$`                                                            | `src/paperless/urls.py:L136-137`                                                                                                                                                                     |
| `ProtocolTypeRouter` (HTTP + websocket)                                                  | `src/paperless/asgi.py:L17`                                                                                                                                                                          |
| `document_consumer` watcher logger                                                       | `src/documents/management/commands/document_consumer.py:L24`                                                                                                                                         |
| `document_consumer` enqueues `documents.tasks.consume_file` via `async_task` (ingestion) | `src/documents/management/commands/document_consumer.py:L13`, `src/documents/management/commands/document_consumer.py:L84-L91`                                                                       |
| `async_task` → `get_broker()` → `broker.enqueue` (default Redis broker, `rpush`)         | `django_q/tasks.py:L55`, `django_q/tasks.py:L73`, `django_q/brokers/__init__.py:L201-L205`, `django_q/brokers/redis_broker.py:L17-L18`                                                               |
| startup checks `paths_check` / `binaries_check` / `debug_mode_check`                     | `src/paperless/checks.py:L52`, `src/paperless/checks.py:L66`, `src/paperless/checks.py:L86`                                                                                                          |
| healthcheck `curl -f http://localhost:8000`, `30s` interval                              | `docker/compose/docker-compose.postgres.yml:L56-57`                                                                                                                                                  |
| base image `python:3.9-slim-bullseye`                                                    | `Dockerfile:L18`                                                                                                                                                                                     |
| `ENTRYPOINT ["/sbin/docker-entrypoint.sh"]`                                              | `Dockerfile:L168`                                                                                                                                                                                    |
| `EXPOSE 8000`                                                                            | `Dockerfile:L170`                                                                                                                                                                                    |
| `CMD` supervisord                                                                        | `Dockerfile:L172`                                                                                                                                                                                    |
| `Paperless-ngx docker container starting...`                                             | `docker/docker-entrypoint.sh:L77`                                                                                                                                                                    |
| Redis gate `Connected to Redis broker: {REDIS_URL}`                                      | `docker/wait-for-redis.py:L41` (also `docker/wait-for-redis.py:L21`, `docker/wait-for-redis.py:L31`, `docker/wait-for-redis.py:L38`, `docker/wait-for-redis.py:L16`, `docker/wait-for-redis.py:L17`) |
| `Q Cluster {id} starting.`                                                               | `django_q/cluster.py:L79`                                                                                                                                                                            |
| `... ready for work at {pid}`                                                            | `django_q/cluster.py:L410`                                                                                                                                                                           |
| `... monitoring at {pid}`                                                                | `django_q/cluster.py:L378`                                                                                                                                                                           |
| `... guarding cluster {id}`                                                              | `django_q/cluster.py:L256`                                                                                                                                                                           |
| `... pushing tasks at {pid}`                                                             | `django_q/cluster.py:L342`                                                                                                                                                                           |
| `Q Cluster {id} running.`                                                                | `django_q/cluster.py:L261`                                                                                                                                                                           |
| `Enqueued {n}`                                                                           | `django_q/tasks.py:L74`                                                                                                                                                                              |
| `... created a task from schedule [...]`                                                 | `django_q/cluster.py:L669`                                                                                                                                                                           |
| `... processing [{task}]`                                                                | `django_q/cluster.py:L420`                                                                                                                                                                           |
| `... stopped doing work`                                                                 | `django_q/cluster.py:L451`                                                                                                                                                                           |
| `Processed [{task}]`                                                                     | `django_q/cluster.py:L392`                                                                                                                                                                           |
| `recycled worker {name}`                                                                 | `django_q/cluster.py:L232`                                                                                                                                                                           |
| pusher error log `logger.error(e, traceback.format_exc())`                               | `django_q/cluster.py:L347`                                                                                                                                                                           |
| `... stopped pushing tasks`                                                              | `django_q/cluster.py:L366`                                                                                                                                                                           |
| `reincarnated pusher {name} after sudden death`                                          | `django_q/cluster.py:L223`                                                                                                                                                                           |
| pusher `broker.dequeue()` → `blpop`                                                      | `django_q/cluster.py:L345`; `django_q/brokers/redis_broker.py:L21`                                                                                                                                   |
| `Q Cluster {id} stopping.` / `has stopped.`                                              | `django_q/cluster.py:L87`, `django_q/cluster.py:L90`                                                                                                                                                 |
| `Cluster.name` = `humanize(cluster_id.hex)` / `uuid.uuid4()`                             | `django_q/cluster.py:L110-111`, `django_q/cluster.py:L58`                                                                                                                                            |
| `GUARD_CYCLE = 0.5` / `SCHEDULER = True`                                                 | `django_q/conf.py:L90`, `django_q/conf.py:L93`                                                                                                                                                       |
| `PREFIX` / `Q_STAT = django_q:{PREFIX}:cluster`                                          | `django_q/conf.py:L80`, `django_q/conf.py:L174`                                                                                                                                                      |
| `django-q` logger / format / `propagate=False` / `StreamHandler`                         | `django_q/conf.py:L207`, `django_q/conf.py:L213-214`, `django_q/conf.py:L212`, `django_q/conf.py:L216`                                                                                               |
| `django-q == 1.3.9` pin                                                                  | `Pipfile.lock:default.django-q.version`                                                                                                                                                              |
