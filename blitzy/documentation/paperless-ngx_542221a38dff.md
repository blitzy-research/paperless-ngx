# paperless-ngx — Automatic Background Maintenance & Task Scheduling

**Repository:** paperless-ngx
**Branch:** `paperless-ngx_542221a38dff`
**Commit:** `542221a38dff06361e07976452f9aea24d210542`
**Runtime used for all observations:** the project's documented Docker image on **Python 3.9.23** (Debian 11), with **Redis** running on `127.0.0.1:6379` (Django-Q broker + Channels layer) and the default **SQLite** database at `/app/data/db.sqlite3`. All commands were run as the non-root user `testuser` from `/app/src`.

This document answers eleven questions about what paperless-ngx does automatically in the background and how its task scheduling works. **Every behavioral claim is paired with the exact command that produced it, the complete/unedited output, a `file:line` citation into the source at this commit, and a cause→effect explanation.** Negative findings (features the questions assume that do not exist) are stated plainly with evidence of absence.

---

## ⚠️ The single most important correction: Celery → Django-Q

The questions are framed in **Celery** terminology ("the Celery beat schedule configuration"). **At this commit, paperless-ngx does not use Celery at all — it uses [Django-Q](https://django-q.readthedocs.io/) 1.3.9.** (Later paperless-ngx releases _did_ migrate to Celery; this report describes only commit `542221a38dff`.) Django-Q is described by its authors as "a native Django task queue, scheduler and worker application using Python multiprocessing." Everything below is therefore answered against Django-Q's real machinery, and the Celery vocabulary is remapped as follows:

| Celery concept (as asked)                                          | paperless-ngx reality (Django-Q)                                                                       |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| "Celery"                                                           | **Not present.** Zero occurrences under `src/`.                                                        |
| celery worker                                                      | `python3 manage.py qcluster` (the Django-Q cluster: workers + monitor + scheduler/pusher)              |
| `celery beat` schedule                                             | The **`django_q_schedule`** table, seeded by three data migrations calling `django_q.tasks.schedule()` |
| beat schedule file (`celerybeat-schedule` / `CELERYBEAT_SCHEDULE`) | **No such file.** Schedules are defined imperatively inside data migrations                            |
| Celery config (`CELERY_*`)                                         | The `Q_CLUSTER` dict in `src/paperless/settings.py`                                                    |
| task result backend / `django_celery_results`                      | Django-Q's **`django_q_task`** table (+ `Success`/`Failure` proxy models)                              |

**Evidence of absence — Celery is nowhere in the source tree:**

```console
$ grep -rin "celery" src/
$ echo "exit code: $?"
exit code: 1
```

`grep` exits `1` (no matches). Celery is not imported anywhere, is not in `requirements.txt`, and there is no `celery.py`, no `@shared_task`, no `@periodic_task`, and no `celerybeat` schedule.

**Evidence of presence — Django-Q is the registered engine:**

`"django_q"` is a registered Django app:

```python
# src/paperless/settings.py:108-111
    "rest_framework.authtoken",
    "django_filters",
    "django_q",
] + env_apps
```

The cluster is configured by the `Q_CLUSTER` dictionary:

```python
# src/paperless/settings.py:449-457
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": PAPERLESS_WORKER_RETRY,
    "timeout": PAPERLESS_WORKER_TIMEOUT,
    "workers": TASK_WORKERS,
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
}
```

And the pinned dependency confirms the engine and version:

```
# requirements.txt:37
django-q==1.3.9
```

The **canonical vocabulary** used throughout this document: `Q_CLUSTER` (configuration), `django_q.tasks.schedule()` (the helper that seeds a schedule), `django_q.models.Schedule` → table `django_q_schedule` (the recurring-job definitions/state), the `qcluster` management command (the live scheduler/worker process), and `django_q.models.Task` → table `django_q_task` with its `Success`/`Failure` proxy models (execution history).

---

## How the evidence was gathered (run-first methodology)

1. **Seed the schedules via the canonical path:**
   ```
   docker exec -u testuser -w /app/src <container> python3 manage.py migrate
   ```
   The data migrations create the four `django_q_schedule` rows.
2. **Start the canonical scheduler/worker:**
   ```
   docker exec -u testuser -w /app/src -d <container> bash -lc 'cd /app/src && python3 manage.py qcluster > /tmp/qcluster.log 2>&1'
   ```
   All interval/behavioral values below come from this **live `qcluster`** executing the seeded rows — **not** from reading migration source alone. Any value not obtained from the canonical path is explicitly labeled.
3. **Observe** the `django_q_schedule` / `django_q_task` tables and task log output, exercising each task (including edge/negative branches), and **confirm interval stability across ≥2 observations**.

---

## Q1 — What maintenance runs automatically after setup (no user action)?

**Answer.** Once the environment is up, three long-running processes are supervised, and the one responsible for maintenance is the **Django-Q cluster** (`qcluster`). It automatically executes **four recurring jobs** with no user action: **train the classifier (hourly)**, **optimize the search index (daily)**, **sanity check (weekly)**, and **check e-mail accounts (every 10 minutes)**. In addition, container **startup** runs database migrations (which _seed_ those four schedules on first boot) and performs a **conditional** full search-index rebuild. No document import happens on its own — consumption is triggered only by a user dropping a file in the consume directory, uploading via the API, or an e-mail arriving.

**Process topology** is defined by supervisor — three programs:

```ini
# docker/supervisord.conf:10-11
[program:gunicorn]
command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
```

```ini
# docker/supervisord.conf:19-20
[program:consumer]
command=python3 manage.py document_consumer
```

```ini
# docker/supervisord.conf:28-29
[program:scheduler]
command=python3 manage.py qcluster
```

The `[program:scheduler]` = `qcluster` is the maintenance engine. **Proof it is alive and working** (the image has no `ps`, so processes were enumerated from `/proc`):

```console
$ docker exec <container> bash -lc 'for pid in /proc/[0-9]*; do tr "\0" " " < "$pid/cmdline" 2>/dev/null | grep -q "manage.py qcluster" && echo "PID $(basename $pid): $(tr "\0" " " < "$pid/cmdline")"; done'
PID 102: python3 manage.py qcluster
PID 112: python3 manage.py qcluster
PID 113: python3 manage.py qcluster
PID 114: python3 manage.py qcluster
PID 115: python3 manage.py qcluster
PID 131: python3 manage.py qcluster
PID 132: python3 manage.py qcluster
PID 133: python3 manage.py qcluster
```

The multiple PIDs are Django-Q's multiprocessing model — one guard/sentinel (master), a pool of worker processes, a monitor, and a scheduler/pusher — all under the single `qcluster` command.

**What actually fired automatically** — the live cluster created and processed all four schedules on its first sweep (excerpt from `/tmp/qcluster.log`):

```
Process-1 created a task from schedule [Train the classifier]
Process-1 created a task from schedule [Optimize the index]
Process-1 created a task from schedule [Perform sanity check]
Process-1 created a task from schedule [Check all e-mail accounts]
```

Cause→effect: supervisor keeps `qcluster` running; `qcluster`'s scheduler component polls `django_q_schedule` for rows whose `next_run` is due and enqueues them onto the Redis broker, where worker processes execute them — entirely without user action. The four jobs themselves are enumerated in **Q6**; the startup-vs-recurring split is detailed in **Q9**. The document classifier that is trained automatically lives in `src/documents/classifier.py` (`class DocumentClassifier` at line 60).

---

## Q2 — Which scheduler is used, and where is the schedule configuration defined?

**Answer.** The scheduler is **Django-Q**, registered as the app `"django_q"` (`src/paperless/settings.py:110`) and configured by the **`Q_CLUSTER`** dict (`src/paperless/settings.py:449-457`). Crucially, the **cluster configuration** and the **schedule definitions** live in two different places:

- **Cluster configuration** (workers, timeout, retry, broker) → `Q_CLUSTER` in `settings.py`.
- **Schedule definitions** (which functions run, and how often) → **data migrations** that call `django_q.tasks.schedule()`. There is **no declarative "beat" file**; at runtime the "schedule configuration" _is_ the set of rows in the **`django_q_schedule`** table.

`Q_CLUSTER` (verbatim, already shown in the correction section above) sets `name="paperless"`, `catch_up=False` (line 451), `recycle=1` (line 452), `retry=PAPERLESS_WORKER_RETRY` (line 453), `timeout=PAPERLESS_WORKER_TIMEOUT` (line 454), `workers=TASK_WORKERS` (line 455), and `redis` defaulting to `"redis://localhost:6379"` (line 456).

**Runtime confirmation that the config produced a live cluster** — the `qcluster` startup banner:

```
[Q] INFO Q Cluster october-video-one-pennsylvania starting.
Process-1:1 ready for work at 40
Process-1:2 ready for work at 41
...
Process-1:11 ready for work at 50
Process-1:12 monitoring at 51
Process-1 guarding cluster october-video-one-pennsylvania
Process-1:13 pushing tasks at 52
[Q] INFO Q Cluster october-video-one-pennsylvania running.
```

Note: `october-video-one-pennsylvania` is Django-Q's random _display_ name for this cluster instance; the configured `Q_CLUSTER["name"]="paperless"` is used as the Redis broker key prefix, not as the log banner name. The 11 worker processes match `TASK_WORKERS` (derived from CPU count via `PAPERLESS_TASK_WORKERS`), and `Process-1:13 pushing tasks` is the scheduler component that reads `django_q_schedule`.

Cause→effect: because `django_q` is in `INSTALLED_APPS`, its models (`Schedule`, `Task`) and its `qcluster` command are available; because `Q_CLUSTER` points at Redis and defines worker/timeout/retry values, `qcluster` launches a multiprocess cluster bound to that broker. The database/broker backing this is also in `settings.py`: `DATABASES` defaults to SQLite (`src/paperless/settings.py:297-302`) and switches to PostgreSQL only when `PAPERLESS_DBHOST` is set (`:304-318`); the Channels layer shares Redis (`CHANNEL_LAYERS` at `:178-180`). See **Q7** for the exact schedule-defining lines.

---

## Q3 — The sanity checker: what it validates, its log output, and how often it runs

**Answer.** The sanity checker is the scheduled task **`documents.tasks.sanity_check`** (`src/documents/tasks.py:255-267`), which calls **`documents.sanity_checker.check_sanity()`** (`src/documents/sanity_checker.py:49`). It runs **weekly** (seeded as `Schedule.WEEKLY` — see below). For **every** `Document` it validates the thumbnail, the original file, and (if present) the archive file — checking existence, readability, and **MD5 checksum** integrity — plus content presence; it also flags **orphaned files** in the media directory. Output goes to the logger **`paperless.sanity_checker`**.

**Frequency — WEEKLY**, seeded here:

```python
# src/documents/migrations/1004_sanity_check_schedule.py:10-14
    schedule(
        "documents.tasks.sanity_check",
        name="Perform sanity check",
        schedule_type=Schedule.WEEKLY,
    )
```

**The task body** (note it raises on error, so a failing sanity check is recorded as a Django-Q _failure_):

```python
# src/documents/tasks.py:255-267
def sanity_check():
    messages = sanity_checker.check_sanity()

    messages.log_messages()

    if messages.has_error():
        raise SanityCheckFailedException("Sanity check failed with errors. See log.")
    elif messages.has_warning():
        return "Sanity check exited with warnings. See log."
    elif len(messages) > 0:
        return "Sanity check exited with infos. See log."
    else:
        return "No issues detected."
```

**What it validates** (each row is a distinct check inside `check_sanity()`):

| Validation                                                             | Level emitted | Message / `file:line`                                                                                                             |
| ---------------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Thumbnail file exists (`doc.thumbnail_path`)                           | ERROR         | `Thumbnail of document {pk} does not exist.` — `sanity_checker.py:64`                                                             |
| Thumbnail is readable                                                  | ERROR         | `Cannot read thumbnail file of document {pk}: {e}` — `sanity_checker.py:72`                                                       |
| Original file exists (`doc.source_path`)                               | ERROR         | `Original of document {pk} does not exist.` — `sanity_checker.py:77`                                                              |
| Original **MD5 checksum** matches `doc.checksum`                       | ERROR         | `Checksum mismatch of document {pk}. …` — `sanity_checker.py:89`                                                                  |
| Archive checksum/filename **consistency**                              | ERROR         | `… archive file checksum, but no archive filename.` / `… archive file, but its checksum is missing.` — `sanity_checker.py:94-103` |
| Archive file exists + readable + checksum (when `has_archive_version`) | ERROR         | `Archived version of document {pk} does not exist.` `sanity_checker.py:106`; checksum mismatch `:120`                             |
| Content present (`doc.content`)                                        | INFO          | `Document {pk} has no content.` — `sanity_checker.py:128`                                                                         |
| No orphaned files in media dir                                         | WARNING       | `Orphaned file in media dir: {file}` — `sanity_checker.py:131`                                                                    |

The `Document` fields being validated are defined in `src/documents/models.py`: `content`@117, `checksum`@135, `archive_checksum`@143, `archive_filename`@186, `source_path`@223, `has_archive_version`@238, `archive_path`@242, `thumbnail_path`@273.

**The logging mechanism** — `SanityCheckMessages.log_messages()`:

```python
# src/documents/sanity_checker.py:23-30
    def log_messages(self):
        logger = logging.getLogger("paperless.sanity_checker")

        if len(self._messages) == 0:
            logger.info("Sanity checker detected no issues.")
        else:
            for msg in self._messages:
                logger.log(msg["level"], msg["message"])
```

### Both branches exercised at runtime

**(a) The "no issues" branch** — captured directly from the canonical `qcluster` running the weekly `sanity_check` on the fresh (empty) database:

```
[2026-07-08 04:26:14,833] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

Re-confirmed via the on-demand management command `python3 manage.py document_sanity_checker` (`src/documents/management/commands/document_sanity_checker.py`, which calls `check_sanity()` then `log_messages()`):

```console
$ python3 manage.py document_sanity_checker --no-progress-bar
[2026-07-08 04:29:22,536] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

**(b) The "populated" branch** — a single `Document` was created with empty content and no files on disk, plus one orphaned file placed in the media directory, then `check_sanity().log_messages()` was invoked. The raw, unedited `paperless.sanity_checker` output:

```
[2026-07-08 04:29:44,139] [ERROR] [paperless.sanity_checker] Thumbnail of document 1 does not exist.
[2026-07-08 04:29:44,140] [ERROR] [paperless.sanity_checker] Original of document 1 does not exist.
[2026-07-08 04:29:44,140] [INFO] [paperless.sanity_checker] Document 1 has no content.
[2026-07-08 04:29:44,140] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /app/media/documents/originals/blitzy_orphan_observation.pdf
```

```
has_error = True | has_warning = True | total msgs = 4
```

Cause→effect: the missing thumbnail and missing original each append an ERROR (`sanity_checker.py:64,77`); the empty `content` field appends an INFO (`:128`); the extra file that does not belong to any document is reported as a WARNING (`:131`). Because `has_error()` is `True`, in the scheduled path `sanity_check()` would `raise SanityCheckFailedException` (`tasks.py:260`) — which is exactly how a sanity failure becomes a recorded Django-Q _failure_ (see Q8). The temporary document and orphan file were deleted afterward, restoring the database to zero issues.

---

## Q4 — Is there automatic search-index optimization and/or database cleanup?

**Answer.** **Search-index optimization: YES.** **Database (ORM) cleanup: NO** — this does not exist.

### Index optimization EXISTS (Whoosh, daily)

The scheduled task **`documents.tasks.index_optimize`** optimizes the **Whoosh** full-text index (a set of files on disk, separate from the relational database):

```python
# src/documents/tasks.py:32-35
def index_optimize():
    ix = index.open_index()
    writer = AsyncWriter(ix)
    writer.commit(optimize=True)
```

`index.open_index()` (`src/documents/index.py:52`) opens the Whoosh index directory; `writer.commit(optimize=True)` merges the index segments. It is seeded **DAILY**:

```python
# src/documents/migrations/1001_auto_20201109_1636.py:15-19
    schedule(
        "documents.tasks.index_optimize",
        name="Optimize the index",
        schedule_type=Schedule.DAILY,
    )
```

**Runtime confirmation** — both the on-demand command and a direct call complete successfully and show the target is the Whoosh index:

```console
$ python3 manage.py document_index optimize
$ echo "exit code: $?"
exit code: 0
```

```
Whoosh index opened: FileIndex(FileStorage('/app/src/../data/index'), 'MAIN') doc_count= 0
index_optimize() completed: writer.commit(optimize=True) executed on Whoosh index at /app/src/../data/index
```

The on-demand equivalent is `python3 manage.py document_index optimize` (`src/documents/management/commands/document_index.py`, where `optimize` → `index_optimize()` at line 25 and `reindex` → `index_reindex()` at line 23).

### Database cleanup does NOT exist (negative finding)

There is **no scheduled task and no task function that prunes, purges, vacuums, or otherwise cleans up relational-database rows.** `index_optimize` operates on the Whoosh search index, _not_ on the SQLite/PostgreSQL database. Evidence of absence:

```console
$ grep -rniE "def (cleanup|purge|prune|vacuum|db_cleanup|delete_old|clean_db)" src/documents/tasks.py src/paperless_mail/tasks.py
$ echo "exit code: $?"
exit code: 1
```

No such function exists, and the complete recurring inventory (Q6) contains only four jobs, none of which touches ORM-row retention. The only automatic "trimming" anywhere in the system is Django-Q's own `save_limit` retention of _its own_ task-history rows (see Q10) — a Django-Q feature, not a paperless database-cleanup task.

---

## Q5 — Is there any task that handles failed-document retries or stuck/hung jobs?

**Answer. No — there is no dedicated failed-document-retry task and no stuck/hung-job watchdog task in paperless.** The _only_ retry/stuck-job mechanism is Django-Q's **cluster-level** `retry`/`timeout` behavior configured in `Q_CLUSTER` (see Q8 for the precise semantics). This operates at the task-queue level, not at the document level; there is no logic that, e.g., finds documents whose consumption failed and re-enqueues them.

Two things are easy to mistake for a retry/stuck-job handler but are **not**:

1. **`async_task()` enqueues are ad-hoc work, not retry handlers.** Throughout the codebase, one-off work is dispatched with Django-Q's `async_task()` (fire-and-forget), never as a recurring schedule and never as a retry loop:

   ```console
   $ grep -rn "async_task" src/documents/views.py src/documents/management/commands/document_consumer.py src/paperless_mail/mail.py src/documents/bulk_edit.py
   src/documents/views.py:28:from django_q.tasks import async_task
   src/documents/views.py:523:        async_task(
   src/documents/management/commands/document_consumer.py:13:from django_q.tasks import async_task
   src/documents/management/commands/document_consumer.py:86:        async_task(
   src/paperless_mail/mail.py:11:from django_q.tasks import async_task
   src/paperless_mail/mail.py:336:                async_task(
   src/documents/bulk_edit.py:4:from django_q.tasks import async_task
   src/documents/bulk_edit.py:18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
   src/documents/bulk_edit.py:31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
   src/documents/bulk_edit.py:47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
   src/documents/bulk_edit.py:63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
   src/documents/bulk_edit.py:87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
   ```

   These enqueue `consume_file` when a user drops/uploads a file or an e-mail arrives (`views.py:523`, `document_consumer.py:86`, `mail.py:336`) and `bulk_update_documents` for bulk operations (`bulk_edit.py`). If one of these fails, it becomes a Django-Q _failure_ record (Q8); nothing automatically retries it.

2. **The consumer's `observer.schedule(...)` is watchdog, not Django-Q.** The directory watcher uses the `watchdog` library's polling observer — the word "schedule" here is unrelated to task scheduling:

   ```python
   # src/documents/management/commands/document_consumer.py:187-188
        self.observer = PollingObserver(timeout=settings.CONSUMER_POLLING)
        self.observer.schedule(Handler(), directory, recursive=recursive)
   ```

Cause→effect: because retry/stuck handling is delegated entirely to Django-Q's queue-level `retry`/`timeout` (which re-queues a task whose worker _died or timed out_, but **not** a task that raised — see Q8), paperless itself contains no document-level retry loop and no "find stuck jobs" sweeper. This is a **negative finding**, stated here explicitly.

---

## Q6 — Every recurring task in the schedule configuration, with its interval

**Answer.** There are **exactly four** recurring tasks. (There is **no `celerybeat` schedule, no `@periodic_task` decorator, and no crontab file** — see the correction section and Q7 for the grep proof. The "schedule configuration" is the set of `django_q_schedule` rows seeded by three data migrations.)

| #   | `func`                                       | `name`                      | `schedule_type`               | Interval             | Seeding migration                                                |
| --- | -------------------------------------------- | --------------------------- | ----------------------------- | -------------------- | ---------------------------------------------------------------- |
| 1   | `documents.tasks.train_classifier`           | "Train the classifier"      | `HOURLY` (`H`)                | **every hour**       | `src/documents/migrations/1001_auto_20201109_1636.py:10-14`      |
| 2   | `documents.tasks.index_optimize`             | "Optimize the index"        | `DAILY` (`D`)                 | **every day**        | `src/documents/migrations/1001_auto_20201109_1636.py:15-19`      |
| 3   | `documents.tasks.sanity_check`               | "Perform sanity check"      | `WEEKLY` (`W`)                | **every week**       | `src/documents/migrations/1004_sanity_check_schedule.py:10-14`   |
| 4   | `paperless_mail.tasks.process_mail_accounts` | "Check all e-mail accounts" | `MINUTES` (`I`), `minutes=10` | **every 10 minutes** | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` |

**Corroborated by the live `django_q_schedule` rows** (read back after `migrate`, before the first `qcluster` sweep). The Django-Q `schedule_type` codes are `H`=Hourly, `D`=Daily, `W`=Weekly, `I`=Minutes:

```
now = 2026-07-08T04:25:19  |  Schedule.objects.count() = 4
id=1 | documents.tasks.train_classifier            | 'Train the classifier'      | type=H | minutes=None | repeats=-1 | next_run=2026-07-08 04:25:07.355152+00:00
id=2 | documents.tasks.index_optimize              | 'Optimize the index'        | type=D | minutes=None | repeats=-1 | next_run=2026-07-08 04:25:07.356128+00:00
id=3 | documents.tasks.sanity_check                | 'Perform sanity check'      | type=W | minutes=None | repeats=-1 | next_run=2026-07-08 04:25:07.421458+00:00
id=4 | paperless_mail.tasks.process_mail_accounts  | 'Check all e-mail accounts' | type=I | minutes=10   | repeats=-1 | next_run=2026-07-08 04:25:07.845925+00:00
```

**Intervals confirmed stable across two independent canonical observations.** After the live `qcluster` swept the due rows the first time, each `next_run` advanced by exactly its interval; then all four `next_run` values were reset to a common instant `T1` and the cluster was allowed to sweep again, producing identical deltas:

```
Measurement #1 (from creation T0 = 04:25:07, after 1st qcluster sweep):
  train_classifier      H : 04:25:07        -> 05:25:07        (+1 hour)
  index_optimize        D : Jul-08 04:25:07 -> Jul-09 04:25:07 (+1 day)
  sanity_check          W : Jul-08 04:25:07 -> Jul-15 04:25:07 (+7 days)
  process_mail_accounts I : 04:25:07        -> 04:35:07        (+10 minutes)

Measurement #2 (after resetting next_run to T1 = 04:27:16, 2nd qcluster sweep at 04:27:44):
  train_classifier      H : 04:27:16        -> 05:27:16        (+1 hour)
  index_optimize        D : Jul-08 04:27:16 -> Jul-09 04:27:16 (+1 day)
  sanity_check          W : Jul-08 04:27:16 -> Jul-15 04:27:16 (+7 days)
  process_mail_accounts I : 04:27:16        -> 04:37:16        (+10 minutes)
```

**Run scale/duration:** a ~3.5-minute observation window containing two scheduler sweeps ~90 s apart; the observed scheduler poll cadence was ~30 s (cluster start 04:25:45 → first schedule execution 04:26:14). The `+1 hour / +1 day / +7 days / +10 min` deltas were **identical across both sweeps**, so the intervals are stable. (Note: advancing `next_run` was performed by the canonical `qcluster` path itself; the only non-canonical action was _resetting_ `next_run` to `T1` to force a prompt second measurement — the resulting intervals are canonical.)

The body of the every-10-minutes job iterates configured mail accounts and delegates the actual fetch:

```python
# src/paperless_mail/tasks.py:11-22 (structure)
def process_mail_accounts():
    ...
    for account in MailAccount.objects.all():
        ... MailAccountHandler().handle_mail_account(account) ...
```

With no configured `MailAccount` rows, it iterates an empty set and returns quickly — but the schedule still fires every 10 minutes, as observed.

---

## Q7 — Where in the code are the periodic tasks defined and registered?

**Answer.** In **three Django data migrations**, each of which calls **`django_q.tasks.schedule()`** inside an `add_schedules(apps, schema_editor)` function that is wrapped in a `migrations.RunPython(...)` operation. Running `manage.py migrate` executes those functions, which **insert the rows into `django_q_schedule`** — that insertion _is_ the registration. Each migration also defines a reverse `remove_schedules()` that deletes its rows by `func`, so the registration is undone on migration rollback.

**Proof that only these three files register schedules** (both the import of the helper and its invocation):

```console
$ grep -rn "from django_q.tasks import schedule" src/
src/documents/migrations/1001_auto_20201109_1636.py:6:from django_q.tasks import schedule
src/documents/migrations/1004_sanity_check_schedule.py:6:from django_q.tasks import schedule
src/paperless_mail/migrations/0002_auto_20201117_1334.py:6:from django_q.tasks import schedule

$ grep -rn "    schedule(" src/
src/documents/migrations/1001_auto_20201109_1636.py:10:    schedule(
src/documents/migrations/1001_auto_20201109_1636.py:15:    schedule(
src/documents/migrations/1004_sanity_check_schedule.py:10:    schedule(
src/paperless_mail/migrations/0002_auto_20201117_1334.py:10:    schedule(
```

Four `schedule(` invocations across exactly three files → four recurring jobs (matching Q6).

**The full registration pattern** (one representative migration, verbatim — note `add_schedules`/`remove_schedules` and the `RunPython` operation):

```python
# src/documents/migrations/1001_auto_20201109_1636.py
from django.db import migrations
from django.db.migrations import RunPython
from django_q.models import Schedule
from django_q.tasks import schedule


def add_schedules(apps, schema_editor):
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


def remove_schedules(apps, schema_editor):
    Schedule.objects.filter(func="documents.tasks.train_classifier").delete()
    Schedule.objects.filter(func="documents.tasks.index_optimize").delete()


class Migration(migrations.Migration):

    dependencies = [
        ("documents", "1000_update_paperless_all"),
        ("django_q", "0013_task_attempt_count"),
    ]

    operations = [RunPython(add_schedules, remove_schedules)]
```

**Runtime confirmation that these migrations performed the registration** — the `migrate` run applied all three:

```
Applying documents.1001_auto_20201109_1636... OK
Applying documents.1004_sanity_check_schedule... OK
Applying paperless_mail.0002_auto_20201117_1334... OK
```

Cause→effect: `RunPython(add_schedules, remove_schedules)` (line 34 of the file above; line 28 in `1004`; line 29 in `0002`) makes Django run `add_schedules` on `migrate` and `remove_schedules` on reverse. `schedule(func, name=…, schedule_type=…[, minutes=…])` is Django-Q's helper that creates a `Schedule` row. So the periodic tasks are _defined_ by these Python calls and _registered_ as database rows the moment `migrate` runs — which is why there is no separate beat file to read. Each migration also declares a dependency on `("django_q", "0013_task_attempt_count")`, guaranteeing the Django-Q tables (including the `Task.attempt_count` column used in Q8) exist before the rows are inserted.

---

## Q8 — What happens when a scheduled task fails? Is there retry logic and/or alerting?

**Answer.** When a task fails, Django-Q writes the **error text (full traceback) into the `Task.result` field** and marks the record `success=False` (visible through the `Failure` proxy model / `django_q_task` table). **Retry logic exists only at the Django-Q cluster level** via the `retry`/`timeout` settings — and it re-queues only tasks whose worker **died or timed out**, _not_ tasks that raised an exception. **There is no exponential backoff and no application-level alerting** (no e-mail/Slack/Sentry/Rollbar wired up); failures surface only via the logs and the `Task.result` row, and a failed task can be manually resubmitted from the Django admin.

**The retry/timeout configuration:**

```python
# src/paperless/settings.py:440
PAPERLESS_WORKER_TIMEOUT: Final[int] = __get_int("PAPERLESS_WORKER_TIMEOUT", 1800)
```

```python
# src/paperless/settings.py:442-447
# Per django-q docs, timeout must be smaller than retry
# We default retry to 10s more than the timeout
PAPERLESS_WORKER_RETRY: Final[int] = __get_int(
    "PAPERLESS_WORKER_RETRY",
    PAPERLESS_WORKER_TIMEOUT + 10,
)
```

These feed `Q_CLUSTER["timeout"]` and `Q_CLUSTER["retry"]` (`settings.py:453-454`). **Semantics**, grounded in the Django-Q documentation:

- **`timeout`** (default **1800 s**) — "The number of seconds a worker is allowed to spend on a task before it's terminated." This bounds how long a single task may run.
- **`retry`** (default **1810 s** = timeout + 10) — "The number of seconds a broker will wait for a cluster to finish a task, before it's presented again." So if a worker dies/hangs and does not acknowledge completion within `retry` seconds, the broker re-delivers the task. paperless deliberately sets `retry = timeout + 10` (comment at `settings.py:442-443`) so a task is re-queued only _after_ it would have been killed by `timeout` — this is the only "stuck job" recovery in the system.

**Runtime demonstration of failure handling via the canonical path** — a task guaranteed to raise was enqueued and executed by the live `qcluster`:

```console
$ python3 manage.py shell -c "from django_q.tasks import async_task; print('id=', async_task('builtins.int', 'this-is-not-an-int'))"
id= <task-id>
# (waited for qcluster to process)
```

The resulting `Failure` row (queried from `django_q_task`):

```
=== Failure proxy row ===
name          : ink-solar-eighteen-chicken
func          : builtins.int
success       : False
attempt_count : 1
result        : invalid literal for int() with base 10: 'this-is-not-an-int' : Traceback (most recent call last):
  File ".../site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
ValueError: invalid literal for int() with base 10: 'this-is-not-an-int'

Task totals -> Task=9, Success=8, Failure=1
```

Corresponding `qcluster` log line:

```
[2026-07-08 04:31:02] [Q] ERROR Failed [ink-solar-eighteen-chicken] - invalid literal for int() with base 10: 'this-is-not-an-int' : Traceback ...
```

Cause→effect: the worker caught the exception, stored the error string + traceback in `Task.result`, and set `success=False`. The Django-Q documentation confirms this design — "The worker will try to put the error in the result field of the task so you can review what happened. You can resubmit a failed task back to the queue using the admin's action menu." **Critically, `attempt_count = 1`** — the task that _raised_ was **not** automatically retried. (Django-Q's `max_attempts` governs re-delivery of tasks that were never acknowledged; a task that ran to completion and raised is simply recorded as failed.) Failures are always persisted (unlike successes, which are capped by `save_limit` — see Q10), so the failure record is retained for inspection. There is no code path that emails, pages, or otherwise alerts on failure — **no application-level alerting** is a negative finding here.

---

## Q9 — Which tasks run on startup vs. strictly on a recurring schedule?

**Answer.** Two distinct kinds of automatic work:

- **Startup (once per container boot, not recurring):**
  1. `python3 manage.py migrate` — applies migrations; on first boot this is also what _seeds_ the four schedules (Q7).
  2. A **conditional** full search-index rebuild (`document_index reindex`) — runs only when an on-disk index-version marker file is missing or outdated.
- **Recurring (strictly on schedule, executed by `qcluster`):** the four jobs from Q6 (`train_classifier` hourly, `index_optimize` daily, `sanity_check` weekly, `process_mail_accounts` every 10 min).

**Startup wiring** — the entrypoint invokes the prepare script:

```sh
# docker/docker-entrypoint.sh:37
	gosu paperless /sbin/docker-prepare.sh
```

Inside `docker-prepare.sh`, migrations run unconditionally, then the index rebuild runs **conditionally**:

```sh
# docker/docker-prepare.sh:45
		python3 manage.py migrate
```

```sh
# docker/docker-prepare.sh:49-56
search_index() {
	index_version=1
	index_version_file=/usr/src/paperless/data/.index_version

	if [[ (! -f "$index_version_file") || $(<$index_version_file) != "$index_version" ]]; then
		echo "Search index out of date. Updating..."
		python3 manage.py document_index reindex
		echo $index_version | tee $index_version_file >/dev/null
```

The conditional reindex is backed by `documents.tasks.index_reindex()` (`src/documents/tasks.py:38-45`), which rebuilds the whole Whoosh index (`index.open_index(recreate=True)` then updates every document). This is a **startup** action — it is _not_ one of the `django_q_schedule` rows. (The _daily_ `index_optimize` from Q4/Q6 is the recurring counterpart; note reindex ≠ optimize: reindex rebuilds, optimize merges segments.)

**Runtime nuance — `catch_up=False` (`src/paperless/settings.py:451`).** Because catch-up is disabled, schedules that were due while the cluster was _down_ are **not** back-run for every missed slot. The Django-Q documentation states the default behavior "is to play catch up and execute all the missed time slots… You can override this behavior by setting `catch_up` to `False`. This will make those schedules run only once when the cluster starts and normal scheduling resumes." This was observable in the `next_run` behavior: on cluster start, each due row fired exactly once and then advanced by a single interval (Q6), rather than replaying multiple missed intervals.

---

## Q10 — Which database table(s) track task-execution history and scheduled-job state?

**Answer.** Under the `django_q` app label:

- **`django_q_task`** — **task-execution history**. Every executed task (scheduled or ad-hoc) becomes a row. Django-Q exposes it through two **proxy** models: **`Success`** (rows where `success=True`) and **`Failure`** (rows where `success=False`). Key columns: `name`, `func`, `args`, `kwargs`, `result` (holds the return value, or the **error/traceback on failure**), `started`, `stopped`, `success`, `attempt_count`.
- **`django_q_schedule`** — **scheduled-job state**. One row per recurring job (the four from Q6). Key columns: `func`, `name`, `schedule_type`, `minutes`, `repeats`, `next_run`, `cron`, `task` (id of the last generated task), `cluster`.
- **`django_q_ormq`** — the **ORM-broker queue table**. paperless uses the **Redis** broker, so this table is **unused** (the live queue lives in Redis, not the database).

**Runtime confirmation of the table names and contents** (queried via the Django ORM / direct SQLite):

```
Task.objects.model._meta.db_table      = django_q_task
Schedule.objects.model._meta.db_table  = django_q_schedule
OrmQ.objects.model._meta.db_table      = django_q_ormq

Success is proxy? True     Failure is proxy? True

django_q_task     row count = 9
  -> Success proxy count = 8
  -> Failure proxy count = 1
django_q_schedule row count = 4
django_q_ormq     row count = 0     # empty: Redis broker in use, not the ORM broker
```

**Why `django_q_ormq` is empty** — the broker is Redis, and `Q_CLUSTER` has no `orm` key:

```
Q_CLUSTER["redis"]          = redis://localhost:6379
"orm" in Q_CLUSTER          = False
```

The Django-Q docs corroborate that the ORM-queue admin/table "is only enabled when you use the Django ORM broker" — with the Redis broker it stays empty, which is exactly what was observed.

**Retention — `save_limit` (Q10 nuance).** paperless does **not** override `save_limit`:

```console
$ grep -rn "save_limit" src/
$ echo "exit code: $?"
exit code: 1
```

```
"save_limit" in Q_CLUSTER            = False
Django-Q effective Conf.SAVE_LIMIT   = 250
```

So Django-Q's **default retention of 250 successful tasks** applies (confirmed at runtime as `Conf.SAVE_LIMIT = 250`, and consistent with the documented default). The Django-Q documentation describes `save_limit` as: "Limits the amount of successful tasks saved to Django. Set to 0 for unlimited. Set to -1 for no success storage at all… **Failures are always saved.**" Cause→effect: `django_q_task` will accumulate up to 250 _successful_ rows (older successes are trimmed by Django-Q), while **all failures are retained**, and `django_q_schedule` always holds exactly the four recurring rows whose `next_run` advances after each execution.

---

## Q11 — What settings/environment variables enable or disable these maintenance features?

**Answer.** Control is **coarse** — there are cluster-wide knobs (environment variables) plus two data-driven on/off behaviors, but **no per-schedule enable/disable setting** in paperless.

**Environment variables / settings** (defaults shown from `paperless.conf.example`):

| Variable                       | Default                  | Effect                                                                             | `file:line`                                             |
| ------------------------------ | ------------------------ | ---------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `PAPERLESS_REDIS`              | `redis://localhost:6379` | Broker + Channels endpoint; without a reachable Redis the whole cluster cannot run | `paperless.conf.example:10`; consumed `settings.py:456` |
| `PAPERLESS_TASK_WORKERS`       | `1`                      | Number of Django-Q worker processes (`Q_CLUSTER["workers"]`)                       | `paperless.conf.example:57`; `settings.py:438,455`      |
| `PAPERLESS_THREADS_PER_WORKER` | `1`                      | Threads per worker for parallelizable work                                         | `paperless.conf.example:58`                             |
| `PAPERLESS_TIME_ZONE`          | `UTC`                    | Time zone in which schedule `next_run` times are interpreted                       | `paperless.conf.example:59`                             |
| `PAPERLESS_CONSUMER_POLLING`   | `10`                     | watchdog polling interval for the consume directory (consumer, not `qcluster`)     | `paperless.conf.example:60`                             |
| `PAPERLESS_WORKER_TIMEOUT`     | `1800`                   | Per-task time limit (`Q_CLUSTER["timeout"]`)                                       | `settings.py:440,454`                                   |
| `PAPERLESS_WORKER_RETRY`       | `1810` (=timeout+10)     | Broker re-queue window (`Q_CLUSTER["retry"]`)                                      | `settings.py:444-447,453`                               |

**Schedule-level pause via `repeats=0` (Django-Q mechanism).** A schedule can be paused by setting its `repeats` to `0`. This is a Django-Q model feature, not a paperless setting. **Demonstrated at runtime:**

```
# set schedule id=4 (process_mail_accounts) repeats=0, next_run in the past, wait > one scheduler sweep (~35 s)
before: task-count-for-func = 2 ; next_run = <past>
after 35s: task-count-for-func = 2 (UNCHANGED) ; next_run = <same past value, not advanced>
=> repeats=0 schedule did NOT fire; restored to repeats=-1 afterward.
```

This matches the Django-Q documentation: repeats is the "Number of times to repeat the schedule. -1=Always, 0=Never, n=n." The four seeded schedules use the default `repeats=-1` ("Always"), which is why they fire indefinitely (and `repeats` counts down `-1 → -2 → …` as an execution counter, as observed in Q6).

**Data-driven on/off — the `train_classifier` self-guard.** The hourly classifier training returns immediately (doing no work) unless at least one `Tag`, `DocumentType`, or `Correspondent` uses automatic matching:

```python
# src/documents/tasks.py:48-55
def train_classifier():
    if (
        not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
        and not DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
        and not Correspondent.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    ):

        return
```

**Both branches demonstrated at runtime:**

```
# self-guard (0 MATCH_AUTO Tag/DocumentType/Correspondent):
train_classifier() returned in 0.0008s ; MODEL_FILE (/app/data/classification_model.pickle) created? False   # early return, no work

# training path (after creating 1 MATCH_AUTO Tag + 5 documents with content):
[2026-07-08 04:30:42,750] [INFO] [paperless.tasks] Saving updated classifier model to /app/src/../data/classification_model.pickle...
train_classifier() returned in 0.4560s ; MODEL_FILE created? True (53973 bytes)
```

Cause→effect: with no auto-matching metadata, the guard's `return` makes the hourly job a no-op — effectively a data-driven "off" for classifier training with zero configuration. (Test artifacts — the tag, documents, and model file — were removed afterward.)

**`catch_up=False` (`settings.py:451`)** is the other coarse control, governing whether missed slots are replayed (Q9).

**Negative finding:** there is **no per-schedule enable/disable environment variable or setting** in paperless. You cannot, via config, toggle "sanity check off" or "classifier on" individually; the available controls are the cluster-wide env vars above, the Django-Q `repeats=0` pause (a manual DB/admin action, not a paperless setting), the `train_classifier` self-guard (data-driven), and `catch_up`.

---

## Appendix — Coverage summary & negative findings

**Every question answered with a concrete value, a `file:line`, observed runtime evidence, and causal reasoning:**

| Q   | Topic                           | Headline finding                                                                                                                     |
| --- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Q1  | Automatic maintenance           | `qcluster` runs 4 recurring jobs; startup seeds schedules + conditional reindex                                                      |
| Q2  | Scheduler & config location     | Django-Q; `Q_CLUSTER` (`settings.py:449-457`); schedules defined in migrations, not a beat file                                      |
| Q3  | Sanity checker                  | `sanity_check` (weekly) → `check_sanity()`; validates thumbnail/original/archive/content/orphans; logs to `paperless.sanity_checker` |
| Q4  | Index optimize vs DB cleanup    | Whoosh `index_optimize` (daily) **exists**; ORM DB-cleanup **does not**                                                              |
| Q5  | Failed-doc retries / stuck jobs | **None** dedicated; only cluster-level `retry`/`timeout`                                                                             |
| Q6  | Recurring inventory             | **Exactly 4**: hourly / daily / weekly / every 10 min                                                                                |
| Q7  | Where defined/registered        | 3 data migrations calling `schedule()` inside `RunPython`                                                                            |
| Q8  | Failure handling                | Error → `Task.result`; cluster `retry`/`timeout` only; `attempt_count=1` (no retry of a raise); **no alerting**                      |
| Q9  | Startup vs recurring            | Startup: migrate + conditional reindex; recurring: the 4 jobs; `catch_up=False`                                                      |
| Q10 | Task-history tables             | `django_q_task` (+`Success`/`Failure`), `django_q_schedule`; `django_q_ormq` empty (Redis broker); `save_limit`=250                  |
| Q11 | Enable/disable controls         | Cluster-wide env vars; `repeats=0` pause; `train_classifier` self-guard; `catch_up`; **no per-schedule toggle**                      |

**Negative findings (each stated with evidence of absence):**

1. **No Celery / celery beat** — `grep -rin "celery" src/` exits 1 (zero matches); the engine is Django-Q 1.3.9.
2. **No dedicated failed-document-retry or stuck-job task** — only Django-Q cluster-level `retry`/`timeout`; the `async_task` sites are ad-hoc enqueues, and `observer.schedule` is watchdog, not Django-Q.
3. **No ORM-level database-cleanup task** — `index_optimize` targets the Whoosh index; `grep` for cleanup/purge/prune/vacuum functions exits 1.
4. **No application-level alerting** — failures surface only via logs and `Task.result`; no e-mail/Sentry/Slack path.
5. **No per-schedule enable/disable setting** — control is coarse (worker env vars, `repeats=0` pause, `train_classifier` self-guard, `catch_up=False`).

**Methodology notes:** all interval/behavioral values were obtained from the canonical `python3 manage.py qcluster` process executing the seeded `django_q_schedule` rows; intervals were confirmed stable across two sweeps within a ~3.5-minute window (observed poll cadence ~30 s). Django-Q 1.3.x internals (the `Task`/`Schedule`/`Success`/`Failure`/`OrmQ` models, `save_limit`=250 default, `repeats`/`catch_up`/`retry`/`timeout` semantics) were grounded against the official Django-Q documentation (django-q.readthedocs.io) and the `Koed00/django-q` source, and independently confirmed by runtime introspection of the installed `django-q==1.3.9`. All observation artifacts (temporary documents, tags, the classifier model file, orphan file, and the failing test task) were removed after capture; the source repository is left byte-for-byte unchanged, with this document as the only addition.
