# paperless-ngx — Background Maintenance & Periodic Scheduling: A Runtime Investigation

- **Repository:** paperless-ngx
- **Commit under evaluation:** `542221a38dff06361e07976452f9aea24d210542`
- **Investigation type:** Read-only. No source file was modified; the only artifact produced is this document. Every temporary observation script and throwaway runtime (SQLite DB rows, staged media, Redis data) was removed afterward.
- **Evidence discipline:** Every behavioral claim below is paired with (a) the exact command run, (b) the complete, unedited output observed, and (c) `file:line` citations. Output blocks are explicitly labeled **Observed** (produced at runtime) or **Inferred** (derived from reading source, when a value could not be produced at runtime). All line numbers were re-verified against the checked-out source at the commit above with `grep -n` / `sed -n` at author time. The only cosmetic edit applied to captured output is that the absolute repository-root path prefix (`/tmp/blitzy/paperless-ngx/blitzy-…`) is abbreviated to `<repo>` for readability; no other content is altered or truncated.

---

## How the runtime environment was brought up (default, canonical config)

The investigation booted the application exactly as a normal operator would, in its default configuration (SQLite database, local Redis broker, Django Q cluster). The exact commands, run from the repository, were:

```bash
# 1. Redis broker (Django Q's live queue lives in Redis). Started as an observation aid.
redis-server --daemonize yes --save '' --appendonly no
redis-cli ping                       # -> PONG

# 2. Apply migrations from src/ — this is what SEEDS the four django_q Schedule rows.
cd src
../venv/bin/python manage.py migrate

# 3. Launch the scheduler + worker (the "task scheduler" process).
PAPERLESS_TASK_WORKERS=1 ../venv/bin/python manage.py qcluster
```

- **Runtime:** Python 3.9.25 (`venv/`), the exact pinned dependencies from `requirements.txt` (`django==4.0.4`, `django-q==1.3.9`, `redis==3.5.3`).
- **No `PAPERLESS_*` environment variables were required** to import settings and run — `SECRET_KEY` has a default and the database defaults to SQLite.

**Observed** — settings resolved at runtime (confirms SQLite default, Redis broker, and the full `Q_CLUSTER`):

```text
DATA_DIR       = <repo>/src/../data
DB ENGINE      = django.db.backends.sqlite3
DB NAME        = <repo>/src/../data/db.sqlite3
MEDIA_ROOT     = <repo>/src/../media
Q_CLUSTER keys = ['catch_up', 'name', 'recycle', 'redis', 'retry', 'timeout', 'workers']
Q_CLUSTER      = {'name': 'paperless', 'catch_up': False, 'recycle': 1, 'retry': 1810, 'timeout': 1800, 'workers': 11, 'redis': 'redis://localhost:6379'}
TASK_WORKERS   = 11
django_q in INSTALLED_APPS = True
```

The database defaults to SQLite at `DATA_DIR/db.sqlite3`, switching to PostgreSQL only when `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:297-311]`. The scheduler process is `python3 manage.py qcluster` `[docker/supervisord.conf:28-29]`.

---

## Section 1 — Leading correction: there is **no Celery**; the scheduler is **Django Q**

The prompt asks about "the Celery beat schedule." **This premise does not hold for this codebase.** At the commit under evaluation, **Celery is entirely absent** and the task scheduler is **Django Q (`django-q==1.3.9`)**. Every "Celery beat" question below is therefore reframed onto the real Django Q implementation.

**Observed** — repository-wide search of the dependency manifests finds no Celery of any kind:

```bash
$ grep -rin "celery" requirements.txt Pipfile Pipfile.lock
$ echo "grep exit=$?"
grep exit=1
```

A non-zero (`1`) exit with no printed lines means **zero matches** — there is no `celery` or `django-celery-*` entry anywhere in `requirements.txt`, `Pipfile`, or `Pipfile.lock`.

**Observed** — the scheduler that *is* present:

```bash
$ grep -n "django-q==" requirements.txt ; grep -n "django-q" Pipfile
37:django-q==1.3.9
17:django-q = "~=1.3"
```

Mapping of the false premise onto reality:

| Prompt term (Celery vocabulary) | Actual mechanism in paperless-ngx (Django Q) | Citation |
|---|---|---|
| "task scheduler" | The `qcluster` process (`python3 manage.py qcluster`) | `[docker/supervisord.conf:28-29]` |
| "Celery beat schedule" | `django_q` `Schedule` rows seeded by data migrations | `[src/documents/migrations/1001_auto_20201109_1636.py:9-19]` |
| "task execution history table" | `django_q_task` relational table (+ `Success`/`Failure` proxies) | `[src/paperless/settings.py:449-457]` |
| "scheduled job state table" | `django_q_schedule` relational table | (observed below) |
| live task queue | Redis (`Q_CLUSTER["redis"]`) | `[src/paperless/settings.py:456]` |

`"django_q"` is registered in `INSTALLED_APPS` `[src/paperless/settings.py:110]` and the cluster is configured by the `Q_CLUSTER` dict `[src/paperless/settings.py:449-457]`.

---

## Section 2 — Full recurring-task inventory (exactly FOUR tasks; verified exhaustive)

There are **exactly four** recurring tasks. Each is registered imperatively inside a Django **data migration** using `from django_q.tasks import schedule`, wrapped in `RunPython(add_schedules, remove_schedules)`.

| # | Task (`func`) | `schedule_type` | Interval | Target function | Registration site |
|---|---|---|---|---|---|
| 1 | `documents.tasks.train_classifier` | `Schedule.HOURLY` (`H`) | every hour | `[src/documents/tasks.py:48-72]` | `[src/documents/migrations/1001_auto_20201109_1636.py:10-14]` |
| 2 | `documents.tasks.index_optimize` | `Schedule.DAILY` (`D`) | every day | `[src/documents/tasks.py:32-35]` | `[src/documents/migrations/1001_auto_20201109_1636.py:15-19]` |
| 3 | `documents.tasks.sanity_check` | `Schedule.WEEKLY` (`W`) | every week | `[src/documents/tasks.py:255-267]` | `[src/documents/migrations/1004_sanity_check_schedule.py:10-14]` |
| 4 | `paperless_mail.tasks.process_mail_accounts` | `Schedule.MINUTES` (`I`), `minutes=10` | every 10 minutes | `[src/paperless_mail/tasks.py:11-22]` | `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15]` |

### 2.1 The four rows as they actually exist in the database

**Observed** — immediately after `manage.py migrate`, querying the `django_q_schedule` table via the ORM:

```bash
$ ../venv/bin/python manage.py shell -c "
from django_q.models import Schedule
rows = Schedule.objects.all().order_by('id')
print('TOTAL Schedule rows =', rows.count())
for s in rows:
    print(f'id={s.id} | func={s.func} | name={s.name!r} | schedule_type={s.schedule_type} | minutes={s.minutes} | repeats={s.repeats} | next_run={s.next_run}')
from django_q.models import Schedule as S
print('Schedule type constants: MINUTES=%r HOURLY=%r DAILY=%r WEEKLY=%r MONTHLY=%r' % (S.MINUTES,S.HOURLY,S.DAILY,S.WEEKLY,S.MONTHLY))
"
```
```text
TOTAL Schedule rows = 4
----------------------------------------------------------------------------------------------------
id=1 | func=documents.tasks.train_classifier | name='Train the classifier' | schedule_type=H | minutes=None | repeats=-1 | next_run=2026-07-13 16:48:32.062102+00:00
id=2 | func=documents.tasks.index_optimize | name='Optimize the index' | schedule_type=D | minutes=None | repeats=-1 | next_run=2026-07-13 16:48:32.063025+00:00
id=3 | func=documents.tasks.sanity_check | name='Perform sanity check' | schedule_type=W | minutes=None | repeats=-1 | next_run=2026-07-13 16:48:32.128172+00:00
id=4 | func=paperless_mail.tasks.process_mail_accounts | name='Check all e-mail accounts' | schedule_type=I | minutes=10 | repeats=-1 | next_run=2026-07-13 16:48:32.541098+00:00
----------------------------------------------------------------------------------------------------
Schedule type constants: MINUTES='I' HOURLY='H' DAILY='D' WEEKLY='W' MONTHLY='M'
```

`repeats=-1` means the schedule repeats indefinitely. The stored `schedule_type` codes map to `H`=HOURLY, `D`=DAILY, `W`=WEEKLY, `I`=MINUTES (with `minutes=10`).

### 2.2 The intervals, proven by how `next_run` advances

The exact intervals are demonstrated by letting the cluster run the due schedules once and observing how each `next_run` is pushed forward (see Section 7 for the `catch_up=False` mechanics):

**Observed** — `next_run` before vs. after a cluster run:

```text
HOURLY  train_classifier      : 2026-07-13 16:48:32  ->  2026-07-13 17:48:32   (+1 hour)
DAILY   index_optimize        : 2026-07-13 16:48:32  ->  2026-07-14 16:48:32   (+1 day)
WEEKLY  sanity_check          : 2026-07-13 16:48:32  ->  2026-07-20 16:48:32   (+7 days)
MINUTES process_mail_accounts : 2026-07-13 16:48:32  ->  2026-07-13 16:58:32   (+10 minutes)
```

### 2.3 Exhaustiveness proof

**Observed** — a repository-wide search shows the Django Q `schedule()` wrapper is called in exactly three migration files (four calls total):

```bash
$ grep -rn "from django_q.tasks import schedule\|^\s*schedule(" src/ --include=*.py
src/paperless_mail/migrations/0002_auto_20201117_1334.py:6:from django_q.tasks import schedule
src/paperless_mail/migrations/0002_auto_20201117_1334.py:10:    schedule(
src/documents/migrations/1001_auto_20201109_1636.py:6:from django_q.tasks import schedule
src/documents/migrations/1001_auto_20201109_1636.py:10:    schedule(
src/documents/migrations/1001_auto_20201109_1636.py:15:    schedule(
src/documents/migrations/1004_sanity_check_schedule.py:6:from django_q.tasks import schedule
src/documents/migrations/1004_sanity_check_schedule.py:10:    schedule(
```

The only other `.schedule(` call in the backend is a **watchdog filesystem observer**, not a Django Q schedule:

```bash
$ grep -rn "schedule(" src/ --include=*.py | grep -v migrations | grep -v test
src/documents/management/commands/document_consumer.py:188:        self.observer.schedule(Handler(), directory, recursive=recursive)
```

### 2.4 The registration convention

**Observed** — the full text of migration `1001` (registers tasks 1 and 2):

```python
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

Each of the three migrations declares a dependency on `("django_q", "0013_task_attempt_count")` `[src/documents/migrations/1001_auto_20201109_1636.py:31]`, `[src/documents/migrations/1004_sanity_check_schedule.py:25]`, `[src/paperless_mail/migrations/0002_auto_20201117_1334.py:26]` — guaranteeing the `django_q` tables exist before the schedule rows are inserted. The `remove_schedules` reverse-operation deletes the rows on migration rollback.

---

## Section 3 — The sanity checker: what it validates, its log output, and how often it runs

**Where it lives:** `check_sanity()` `[src/documents/sanity_checker.py:49-133]`, the `SanityCheckMessages` container `[src/documents/sanity_checker.py:10-42]`, and `SanityCheckFailedException` `[src/documents/sanity_checker.py:45-46]`. All log output uses the logger `"paperless.sanity_checker"` `[src/documents/sanity_checker.py:24]`.

### 3.1 What it validates

`check_sanity()` first walks `settings.MEDIA_ROOT` collecting every file present `[src/documents/sanity_checker.py:52-55]` (removing the media lockfile `[src/documents/sanity_checker.py:57-59]`), then iterates every `Document` `[src/documents/sanity_checker.py:61]` and checks, per document:

| Check | Severity | Line |
|---|---|---|
| Thumbnail file does not exist | **error** | `[src/documents/sanity_checker.py:64]` |
| Thumbnail file cannot be read | **error** | `[src/documents/sanity_checker.py:72]` |
| Original file does not exist | **error** | `[src/documents/sanity_checker.py:77]` |
| Original file cannot be read | **error** | `[src/documents/sanity_checker.py:85]` |
| Original checksum (MD5) mismatch | **error** | `[src/documents/sanity_checker.py:89]` |
| Archive checksum present but no archive filename | **error** | `[src/documents/sanity_checker.py:96]` |
| Archive filename present but no checksum | **error** | `[src/documents/sanity_checker.py:101]` |
| Archived version does not exist | **error** | `[src/documents/sanity_checker.py:106]` |
| Archive file cannot be read | **error** | `[src/documents/sanity_checker.py:115]` |
| Archive checksum mismatch | **error** | `[src/documents/sanity_checker.py:120]` |
| Document has no `content` | **info** | `[src/documents/sanity_checker.py:128]` |
| Orphaned file left over in the media dir | **warning** | `[src/documents/sanity_checker.py:131]` |

After the loop, any media file not "claimed" by a document is reported as an orphaned-file **warning** `[src/documents/sanity_checker.py:130-131]`.

### 3.2 Log output — all three severities plus the clean case

`log_messages()` `[src/documents/sanity_checker.py:23-30]` logs each message at its severity, or — when there are no messages — logs exactly one line: `"Sanity checker detected no issues."` `[src/documents/sanity_checker.py:27]`.

Each severity was produced at runtime by staging throwaway documents/media (torn down afterward). **Observed:**

**Clean** (empty database, no orphaned files):
```text
[2026-07-13 16:55:27,897] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

**Info** (a document with empty `content`, but valid original + thumbnail files present):
```text
[2026-07-13 16:55:26,375] [INFO] [paperless.sanity_checker] Document 2 has no content.
```

**Warning** (an orphaned file in the media directory, no documents):
```text
[2026-07-13 16:55:06,776] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: <repo>/media/documents/originals/orphan_blitzy.pdf
```

**Error** (a document row whose referenced files do not exist on disk):
```text
[2026-07-13 16:54:43,389] [ERROR] [paperless.sanity_checker] Thumbnail of document 1 does not exist.
[2026-07-13 16:54:43,389] [ERROR] [paperless.sanity_checker] Original of document 1 does not exist.
```

### 3.3 Two modes: the command LOGS (never raises); the scheduled task RAISES on errors

This is a critical behavioral distinction.

- **Command mode** — `python3 manage.py document_sanity_checker` calls `check_sanity(...)` then `messages.log_messages()` `[src/documents/management/commands/document_sanity_checker.py:22-26]`. It **only logs** — it never raises, and exits `0` even when errors are present.
- **Scheduled-task mode** — `documents.tasks.sanity_check()` `[src/documents/tasks.py:255-267]` calls `check_sanity()` and `log_messages()`, then **raises `SanityCheckFailedException`** when `messages.has_error()` `[src/documents/tasks.py:260-261]`; otherwise it returns a status string (`"Sanity check exited with warnings. See log."` on warnings `[src/documents/tasks.py:262-263]`, `"Sanity check exited with infos. See log."` on infos `[src/documents/tasks.py:264-265]`, or `"No issues detected."` when clean `[src/documents/tasks.py:266-267]`).

**Observed** — the same error condition, run both ways:

Command mode (logs, then exits `0`):
```bash
$ ../venv/bin/python manage.py document_sanity_checker ; echo "command exit=$?"
100%|██████████| 1/1 [00:00<00:00, 17119.61it/s]
[2026-07-13 16:54:43,389] [ERROR] [paperless.sanity_checker] Thumbnail of document 1 does not exist.
[2026-07-13 16:54:43,389] [ERROR] [paperless.sanity_checker] Original of document 1 does not exist.
command exit=0
```

Scheduled-task mode (logs, then RAISES):
```bash
$ ../venv/bin/python manage.py shell -c "
from documents.tasks import sanity_check
try:
    print('RETURNED:', repr(sanity_check()))
except Exception as e:
    print('RAISED:', type(e).__module__ + '.' + type(e).__name__, '->', e)
"
[2026-07-13 16:54:43,892] [ERROR] [paperless.sanity_checker] Thumbnail of document 1 does not exist.
[2026-07-13 16:54:43,893] [ERROR] [paperless.sanity_checker] Original of document 1 does not exist.
RAISED: documents.sanity_checker.SanityCheckFailedException -> Sanity check failed with errors. See log.
```

For the non-error severities, the scheduled task returns a string rather than raising. **Observed:**
```text
# warning condition -> RETURNED: 'Sanity check exited with warnings. See log.'
# info condition    -> RETURNED: 'Sanity check exited with infos. See log.'
# clean condition    -> RETURNED: 'No issues detected.'
```

### 3.4 How often it runs

The sanity check runs **weekly**. Its schedule row is seeded as `schedule_type=Schedule.WEEKLY` `[src/documents/migrations/1004_sanity_check_schedule.py:10-14]`, and the live `django_q_schedule` row confirms it (Section 2.1: `id=3 ... schedule_type=W`). When the cluster ran it via the scheduler, its `next_run` advanced by exactly 7 days (Section 2.2), and the scheduled run emitted the clean line through the scheduler:

**Observed** — the scheduled dispatch of `sanity_check` on an empty database (captured in the `qcluster` log):
```text
16:49:52 [Q] INFO Process-1:5 processing [saturn-tennis-harry-indigo]
[2026-07-13 16:49:52,234] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
16:49:52 [Q] INFO Processed [saturn-tennis-harry-indigo]
```

---

## Section 4 — Index optimization and database cleanup

### 4.1 Automatic index optimization — **YES, it exists**

Search-index optimization is a real, automatic, **daily** task. `index_optimize()` opens the Whoosh index and commits with `optimize=True` `[src/documents/tasks.py:32-35]`:

```python
def index_optimize():
    ix = index.open_index()
    writer = AsyncWriter(ix)
    writer.commit(optimize=True)
```

It is scheduled `DAILY` `[src/documents/migrations/1001_auto_20201109_1636.py:15-19]` (see `id=2 ... schedule_type=D` in Section 2.1). Its real entry point is `manage.py document_index optimize`, which calls `index_optimize()` `[src/documents/management/commands/document_index.py:24-25]`.

**Observed** — run through the real command, twice (stable):
```bash
$ ../venv/bin/python manage.py document_index optimize ; echo "exit=$?"
exit=0
$ ../venv/bin/python manage.py document_index optimize ; echo "exit=$?"
exit=0
```
`index_optimize` produces no stdout; it optimizes the index and exits `0`. It was also observed running via the scheduler (Section 6, task `beer-two-golf-vegan`, `success=True`).

**Related but NOT scheduled:** `index_reindex()` `[src/documents/tasks.py:38-45]` rebuilds the whole index (`recreate=True`). It is reachable via `manage.py document_index reindex` `[src/documents/management/commands/document_index.py:22-23]` but has **no** `Schedule` row — it runs only at startup, conditionally (Section 8).

**Observed** — `reindex` on the empty database (tqdm bar over 0 documents):
```bash
$ ../venv/bin/python manage.py document_index reindex ; echo "exit=$?"
0it [00:00, ?it/s]
0it [00:00, ?it/s]
exit=0
```

### 4.2 Database cleanup — **NO dedicated task**

There is **no dedicated database-cleanup task** among the scheduled work. This is **Observed** via the exhaustive schedule enumeration (Section 2.1 / 2.3): the four — and only four — recurring tasks are `train_classifier`, `index_optimize`, `sanity_check`, and `process_mail_accounts`. None of them purges, vacuums, or prunes database rows.

The closest built-in bounding is a Django Q **library** behavior, not a paperless task: successful task records are capped by `save_limit` (default 250) — `django_q`'s own `save_task()` deletes the oldest `Success` row once the cap is exceeded. paperless does **not** configure `save_limit`, so the library default applies. This prunes only the `django_q_task` success history; it does not clean any paperless application data, and failures are never pruned (see Section 7).

---

## Section 5 — Failed-document retries / stuck-processing-job handling

**There is no dedicated failed-document-retry task and no stuck-job reaper.** This is **Observed** via the exhaustive schedule list (Section 2.3): none of the four scheduled tasks retries failed documents or reaps stuck processing jobs.

**Do not confuse this with the consumer's file-read readiness retry.** The document consumer retries *reading a file that is not yet fully written to disk* — a low-level I/O readiness loop, not a scheduled failed-job reaper:

**Observed** — the consumer's retry constants and loop:
```bash
$ grep -n "os_error_retry_count\|os_error_retry_wait\|while (read_try_count" src/documents/management/commands/document_consumer.py
59:    os_error_retry_count: Final[int] = 50
60:    os_error_retry_wait: Final[float] = 0.01
65:    while (read_try_count < os_error_retry_count) and not file_open_ok:
```

`os_error_retry_count = 50` `[src/documents/management/commands/document_consumer.py:59]` bounds retries of a file *read* (used at `[src/documents/management/commands/document_consumer.py:65]` and `[src/documents/management/commands/document_consumer.py:73]`), each spaced by `os_error_retry_wait = 0.01`s `[src/documents/management/commands/document_consumer.py:60]`. This governs newly-arriving files in the consume directory — it is unrelated to Django Q scheduled-task failures.

---

## Section 6 — End-to-end trace: definition → registration → execution → persistence

**Definition** — the task functions:
- `src/documents/tasks.py` (module logger `"paperless.tasks"` `[src/documents/tasks.py:29]`): `index_optimize()` `[32-35]`, `index_reindex()` `[38-45]`, `train_classifier()` `[48-72]`, `sanity_check()` `[255-267]`.
- `src/paperless_mail/tasks.py` (module logger `"paperless.mail.tasks"` `[src/paperless_mail/tasks.py:8]`): `process_mail_accounts()` `[11-22]`.

**Registration** — the three data migrations seed `django_q_schedule` rows during `migrate`:
- `1001_auto_20201109_1636.py` → `train_classifier` (HOURLY), `index_optimize` (DAILY).
- `1004_sanity_check_schedule.py` → `sanity_check` (WEEKLY).
- `0002_auto_20201117_1334.py` (paperless_mail) → `process_mail_accounts` (MINUTES=10).

**Observed** — the schedule-seeding migrations applying during `manage.py migrate`:
```text
  Applying django_q.0013_task_attempt_count... OK
  Applying django_q.0014_schedule_cluster... OK
  ...
  Applying documents.1001_auto_20201109_1636... OK
  ...
  Applying documents.1004_sanity_check_schedule... OK
  ...
  Applying paperless_mail.0002_auto_20201117_1334... OK
```

**Execution** — a single `qcluster` process performs both worker execution and schedule dispatch, backed by Redis. It is configured by `Q_CLUSTER` `[src/paperless/settings.py:449-457]`, with `"django_q"` in `INSTALLED_APPS` `[src/paperless/settings.py:110]`, and is launched as the supervisord `scheduler` program `[docker/supervisord.conf:28-29]`.

**Observed** — the complete, unedited `qcluster` startup and its first scheduler cycle (all four schedules were due after seeding, so all four fired):
```text
16:49:21 [Q] INFO Q Cluster vegan-beryllium-carolina-skylark starting.
16:49:21 [Q] INFO Process-1:1 ready for work at 31408
16:49:21 [Q] INFO Process-1 guarding cluster vegan-beryllium-carolina-skylark
16:49:21 [Q] INFO Process-1:2 monitoring at 31409
16:49:21 [Q] INFO Process-1:3 pushing tasks at 31410
16:49:21 [Q] INFO Q Cluster vegan-beryllium-carolina-skylark running.
16:49:51 [Q] INFO Enqueued 1
16:49:51 [Q] INFO Process-1 created a task from schedule [Train the classifier]
16:49:51 [Q] INFO Enqueued 1
16:49:51 [Q] INFO Process-1 created a task from schedule [Optimize the index]
16:49:51 [Q] INFO Process-1:1 processing [pennsylvania-queen-kentucky-mirror]
16:49:51 [Q] INFO Enqueued 1
16:49:51 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
16:49:51 [Q] INFO Enqueued 1
16:49:51 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
16:49:51 [Q] INFO Process-1:1 stopped doing work
16:49:51 [Q] INFO Processed [pennsylvania-queen-kentucky-mirror]
16:49:51 [Q] INFO recycled worker Process-1:1
16:49:51 [Q] INFO Process-1:4 ready for work at 31414
16:49:51 [Q] INFO Process-1:4 processing [beer-two-golf-vegan]
16:49:51 [Q] INFO Process-1:4 stopped doing work
16:49:51 [Q] INFO Processed [beer-two-golf-vegan]
16:49:52 [Q] INFO recycled worker Process-1:4
16:49:52 [Q] INFO Process-1:5 ready for work at 31417
16:49:52 [Q] INFO Process-1:5 processing [saturn-tennis-harry-indigo]
[2026-07-13 16:49:52,234] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
16:49:52 [Q] INFO Process-1:5 stopped doing work
16:49:52 [Q] INFO Processed [saturn-tennis-harry-indigo]
16:49:52 [Q] INFO recycled worker Process-1:5
16:49:52 [Q] INFO Process-1:6 ready for work at 31421
16:49:52 [Q] INFO Process-1:6 processing [equal-enemy-saturn-low]
16:49:52 [Q] INFO Process-1:6 stopped doing work
16:49:52 [Q] INFO Processed [equal-enemy-saturn-low]
16:49:53 [Q] INFO recycled worker Process-1:6
16:49:53 [Q] INFO Process-1:7 ready for work at 31423
16:50:05 [Q] INFO Q Cluster vegan-beryllium-carolina-skylark stopping.
16:50:06 [Q] INFO Process-1 stopping cluster processes
16:50:07 [Q] INFO Process-1:3 stopped pushing tasks
16:50:07 [Q] INFO Process-1:7 stopped doing work
16:50:07 [Q] INFO Process-1 waiting for the monitor.
16:50:07 [Q] INFO Process-1:2 stopped monitoring results
16:50:07 [Q] INFO Q Cluster vegan-beryllium-carolina-skylark has stopped.
```

The banner shows the cluster's four internal processes: the **worker** (`ready for work`), the **guard/sentinel** (`guarding cluster`), the **monitor** (`monitoring`, saves results), and the **pusher** (`pushing tasks`, pulls from the broker). `recycled worker` after every task reflects `recycle: 1` `[src/paperless/settings.py:452]`.

**Persistence** — task/schedule state lives in the relational database (`django_q_schedule`, `django_q_task`), while the live queue is in Redis. The results of the four scheduled runs were persisted as `Success` rows with the exact return values from the source functions:

**Observed** — `django_q_task` after the run:
```text
Task total   = 4
Success total= 4
Failure total= 0
------------------------------------------------------------------------------------------
name=pennsylvania-queen-kentucky-mirror | func=documents.tasks.train_classifier | success=True | attempt_count=1
   result=None
name=beer-two-golf-vegan | func=documents.tasks.index_optimize | success=True | attempt_count=1
   result=None
name=saturn-tennis-harry-indigo | func=documents.tasks.sanity_check | success=True | attempt_count=1
   result='No issues detected.'
name=equal-enemy-saturn-low | func=paperless_mail.tasks.process_mail_accounts | success=True | attempt_count=1
   result='No new documents were added.'
```

The `sanity_check` result `'No issues detected.'` matches `[src/documents/tasks.py:266-267]`; the `process_mail_accounts` result `'No new documents were added.'` matches `[src/paperless_mail/tasks.py:21-22]` (there were zero configured mail accounts).

---

## Section 7 — What happens when a scheduled task fails (retry? alerting?)

This section corrects a common misconception and reports the **observed** behavior for this exact version + broker.

### 7.1 The configuration

`Q_CLUSTER` `[src/paperless/settings.py:449-457]` sets `catch_up: False` `[451]`, `recycle: 1` `[452]`, `retry: PAPERLESS_WORKER_RETRY` `[453]`, `timeout: PAPERLESS_WORKER_TIMEOUT` `[454]`. The defaults are `PAPERLESS_WORKER_TIMEOUT = 1800` `[src/paperless/settings.py:440]` and `PAPERLESS_WORKER_RETRY = PAPERLESS_WORKER_TIMEOUT + 10` `[src/paperless/settings.py:444-446]`, with the in-source rationale:

```python
# Per django-q docs, timeout must be smaller than retry
# We default retry to 10s more than the timeout
```

**Observed** — the resolved values and, importantly, the *absence* of certain keys:
```text
Q_CLUSTER keys = ['catch_up', 'name', 'recycle', 'redis', 'retry', 'timeout', 'workers']
error_reporter present?  False
max_attempts present?    False
ack_failures present?    False
Conf.ERROR_REPORTER = {} (empty dict => no alerting)
Conf.MAX_ATTEMPTS   = 0 (0 => no attempt cap)
Conf.ACK_FAILURES   = False (False => failures not acknowledged)
Conf.RETRY          = 1810
Conf.TIMEOUT        = 1800
Conf.CATCH_UP       = False
```

### 7.2 Is there retry logic? — Observed: a failed task runs **once** and is **not** re-run

**Correction of a common assumption:** `max_attempts` **does exist** in the installed `django-q==1.3.9` (`MAX_ATTEMPTS = conf.get("max_attempts", 0)` in `django_q/conf.py:180`) — it is *not* exclusive to the `django-q2` fork. However, paperless does not set it, so it defaults to `0` (no cap).

The decisive factor for retry behavior is the **broker**. paperless uses the plain **Redis list broker**, whose `dequeue()` uses `blpop` and returns an `ack_id` of `None` (`django_q/brokers/redis_broker.py`), and whose base `acknowledge()`/`fail()` methods are no-ops (`django_q/brokers/__init__.py:69-80`). Because `blpop` atomically removes the task from the list, the Redis broker provides **no delivery receipts and therefore no redelivery**. The `retry` setting is documented to "only work[] with brokers that guarantee delivery" (`django_q/conf.py:133-134`); with the Redis list broker it is effectively inert.

The net effect: **a scheduled task that raises runs exactly once, is recorded as a Failure, and is not automatically retried.** This was confirmed by forcing a failure through the real broker/worker dispatch (`async_task` of a stdlib function that raises), with `qcluster` running.

**Observed** — the forced failure in the `qcluster` log (note the single "processing" line — it is never re-processed, even though the cluster kept running for ~35 s afterward):
```text
16:56:48 [Q] INFO Process-1:1 processing [blitzy-forced-failure]
16:56:48 [Q] INFO Process-1:1 stopped doing work
16:56:48 [Q] ERROR Failed [blitzy-forced-failure] - Expecting value: line 1 column 1 (char 0) : Traceback (most recent call last):
  File ".../django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File ".../json/__init__.py", line 346, in loads
    return _default_decoder.decode(s)
  File ".../json/decoder.py", line 337, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
  File ".../json/decoder.py", line 355, in raw_decode
    raise JSONDecodeError("Expecting value", s, err.value) from None
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

**Observed** — the persisted `Failure` record (a failed task is always saved), and proof it exists exactly once with `attempt_count=1`:
```text
FAILURE:
  id            = e58ba0a703b0413ebdcda1933c32e049
  name          = blitzy-forced-failure
  func          = json.loads
  success       = False
  attempt_count = 1
  result (first 200 chars) = 'Expecting value: line 1 column 1 (char 0) : Traceback (most recent call last):\n  File ".../django_q/cluster.py", ...'

rows named blitzy-forced-failure = 1 (1 => ran once, NOT retried)
  attempt_count = 1 | success = False
```

### 7.3 `retry` vs. `timeout` — what they actually mean here

- **`timeout` (1800 s)** — the maximum seconds a worker may spend on a task before the sentinel terminates ("reincarnates") it. This applies regardless of broker.
- **`retry` (1810 s)** — a **broker redelivery** timeout: seconds a delivery-receipt-capable broker waits before re-presenting an unacknowledged task. Its only role in this configuration is to satisfy the library's `timeout < retry` invariant; with the Redis list broker there is no redelivery to trigger. This is an at-least-once **redelivery** concept, **not** a "retry N times on failure" counter.

### 7.4 Missed schedules — `catch_up=False`

With `catch_up: False` `[src/paperless/settings.py:451]`, schedules missed while the cluster was down are **not** replayed slot-by-slot; each runs **once** at restart and then resumes normal future scheduling. This was observed directly: after seeding, all four `next_run` values were in the past; on cluster start each ran once and its `next_run` advanced by exactly one interval into the future (Section 2.2), rather than firing repeatedly to "catch up."

### 7.5 Is there alerting? — Observed: **no**

There is **no alerting**. `Q_CLUSTER` configures no `error_reporter`, so `Conf.ERROR_REPORTER == {}` (Section 7.1). In the worker, the error reporter is invoked only `if error_reporter:` (`django_q/cluster.py`), which is falsy here. A failed task is therefore **only** persisted as a `Failure` row — there is no email, webhook, Rollbar, or Sentry notification on failure.

---

## Section 8 — Tasks that run at startup vs. strictly on schedule

### 8.1 Startup activities

- **`migrate` seeds the schedule rows.** The four `django_q_schedule` rows come into existence only when migrations run, which the container does at startup via `migrations()` → `python3 manage.py migrate` `[docker/docker-prepare.sh:38,45]`. This is a startup activity that *creates* the schedules; it does not *execute* the tasks.
- **A conditional startup reindex.** `search_index()` runs `python3 manage.py document_index reindex` **only if** the stored index-version file is missing or stale `[docker/docker-prepare.sh:49-56]`:

```bash
$ grep -n "index_version\|document_index reindex\|search_index" docker/docker-prepare.sh
49:search_index() {
50:	index_version=1
51:	index_version_file=/usr/src/paperless/data/.index_version
53:	if [[ (! -f "$index_version_file") || $(<$index_version_file) != "$index_version" ]]; then
55:		python3 manage.py document_index reindex
56:		echo $index_version | tee $index_version_file >/dev/null
```

So `index_reindex` is a **startup-only, conditional** operation — never a `Schedule` row (contrast with the daily `index_optimize`).

### 8.2 `apps.py` `ready()` wires signal handlers only — no scheduled task

`DocumentsConfig.ready()` `[src/documents/apps.py:11-29]` connects **only six document-consumption signal handlers** and triggers no scheduled task:

**Observed** — the entire `ready()`:
```python
    def ready(self):
        from .signals import document_consumption_finished
        from .signals.handlers import (
            add_inbox_tags,
            set_log_entry,
            set_correspondent,
            set_document_type,
            set_tags,
            add_to_index,
        )

        document_consumption_finished.connect(add_inbox_tags)
        document_consumption_finished.connect(set_correspondent)
        document_consumption_finished.connect(set_document_type)
        document_consumption_finished.connect(set_tags)
        document_consumption_finished.connect(set_log_entry)
        document_consumption_finished.connect(add_to_index)

        AppConfig.ready(self)
```

`PaperlessMailConfig` has **no** `ready()` hook at all `[src/paperless_mail/apps.py:1-7]`.

### 8.3 Strictly scheduled

The four `Schedule` rows fire **only** via the `qcluster` scheduler loop — never at import or process start. This was observed directly: on the second cluster run, only the schedule whose `next_run` had been forced into the past fired; the three future-dated schedules did **not** fire.

**Observed** — second `qcluster` run (only the due mail schedule fires; note start at 16:51:03, fire at 16:51:33 = +30 s scheduler cadence):
```text
16:51:03 [Q] INFO Q Cluster quiet-solar-south-four starting.
16:51:03 [Q] INFO Process-1:1 ready for work at 31959
16:51:03 [Q] INFO Process-1:2 monitoring at 31960
16:51:03 [Q] INFO Process-1 guarding cluster quiet-solar-south-four
16:51:03 [Q] INFO Process-1:3 pushing tasks at 31961
16:51:03 [Q] INFO Q Cluster quiet-solar-south-four running.
16:51:33 [Q] INFO Enqueued 1
16:51:33 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
16:51:33 [Q] INFO Process-1:1 processing [ceiling-lion-salami-green]
16:51:33 [Q] INFO Process-1:1 stopped doing work
16:51:33 [Q] INFO Processed [ceiling-lion-salami-green]
16:51:34 [Q] INFO recycled worker Process-1:1
16:51:34 [Q] INFO Process-1:4 ready for work at 31964
16:51:43 [Q] INFO Q Cluster quiet-solar-south-four stopping.
```

The ~30 s scheduler cadence was stable across all three cluster runs performed (start → first schedule pickup was +30 s each time).

---

## Section 9 — Database tables tracking task history / scheduled-job state

**Yes — two relational tables track this state** (the live queue is separate, in Redis). Django Q creates three tables; paperless uses two of them meaningfully:

**Observed** — the `django_q_*` tables and their row counts after `migrate` + cluster runs + the forced failure:
```bash
$ ../venv/bin/python -c "
import sqlite3
c = sqlite3.connect('../data/db.sqlite3')
for (name,) in c.execute(\"SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'django_q%' ORDER BY name\"):
    n = c.execute(f'SELECT count(*) FROM \"{name}\"').fetchone()[0]
    print(f'  {name}: {n} rows')
"
  django_q_ormq: 0 rows
  django_q_schedule: 4 rows
  django_q_task: 7 rows
```

- **`django_q_schedule`** — **scheduled-job state**: one row per recurring task, holding `func`, `name`, `schedule_type`, `minutes`, `repeats`, and `next_run` (the four rows in Section 2.1).
- **`django_q_task`** — **task execution history**: one row per executed task. Django Q exposes two proxy models over it — `Success` (rows where `success=True`) and `Failure` (rows where `success=False`).
- **`django_q_ormq`** — the ORM-broker queue table; **unused** here because the broker is Redis (0 rows throughout).

**Observed** — the execution history, summarized by function (6 successes + the 1 forced failure = 7):
```text
Task=7  Success=6  Failure=1
by func:
  documents.tasks.index_optimize                success=True  count=1
  documents.tasks.sanity_check                  success=True  count=1
  documents.tasks.train_classifier              success=True  count=1
  json.loads                                    success=False count=1
  paperless_mail.tasks.process_mail_accounts    success=True  count=3
```

The schedule count was stable at 4 across repeated queries; the task counts are the exact cumulative total of the observed runs.

---

## Section 10 — What controls whether these maintenance features are enabled/disabled

There is **no single on/off switch** for "maintenance." The schedules exist once migrations have run, and they execute whenever a `qcluster` process is running. The relevant `PAPERLESS_*` settings and documented flags:

| Control | Setting / effect | Citation |
|---|---|---|
| `PAPERLESS_TASK_WORKERS` | → `TASK_WORKERS` (worker count; default via `default_task_workers()`) | `[src/paperless/settings.py:427,438]`; documented `[paperless.conf.example:57]` |
| `PAPERLESS_THREADS_PER_WORKER` | → `THREADS_PER_WORKER` | `[src/paperless/settings.py:460,469-471]`; documented `[paperless.conf.example:58]` |
| `PAPERLESS_WORKER_TIMEOUT` | worker task timeout (default 1800 s) → `Q_CLUSTER["timeout"]` | `[src/paperless/settings.py:440,454]` |
| `PAPERLESS_WORKER_RETRY` | broker redelivery timeout (default timeout+10) → `Q_CLUSTER["retry"]` | `[src/paperless/settings.py:444-446,453]` |
| `PAPERLESS_REDIS` | broker/queue location → `Q_CLUSTER["redis"]` | `[src/paperless/settings.py:456]`; documented `[paperless.conf.example:10]` |
| `PAPERLESS_DBHOST` | switches DB from SQLite to PostgreSQL | `[src/paperless/settings.py:304-311]` |
| `PAPERLESS_CONSUMER_POLLING` | consumer polling interval (separate consumer, not the scheduler) | `[paperless.conf.example:60]` |

**Observed** — the documented maintenance/worker flags in `paperless.conf.example`:
```bash
$ grep -n "PAPERLESS_REDIS=\|PAPERLESS_TASK_WORKERS\|PAPERLESS_THREADS_PER_WORKER\|PAPERLESS_CONSUMER_POLLING\|PAPERLESS_CONSUMER_RECURSIVE\|PAPERLESS_CONSUMER_DELETE_DUPLICATES" paperless.conf.example
10:#PAPERLESS_REDIS=redis://localhost:6379
57:#PAPERLESS_TASK_WORKERS=1
58:#PAPERLESS_THREADS_PER_WORKER=1
60:#PAPERLESS_CONSUMER_POLLING=10
61:#PAPERLESS_CONSUMER_DELETE_DUPLICATES=false
62:#PAPERLESS_CONSUMER_RECURSIVE=false
```

**Effectively disabling the scheduled maintenance** (Observed reasoning, from the runtime facts above): the scheduler only runs when the `qcluster` process is running `[docker/supervisord.conf:28-29]`, so **not launching `qcluster`** (or setting `PAPERLESS_TASK_WORKERS=0`, which yields `Q_CLUSTER["workers"]=0`) stops the recurring tasks from executing. There is no per-task enable flag in settings — a task is disabled only by deleting/altering its `django_q_schedule` row. Labeled **Inferred** where it depends on operator action not exercised here; the "scheduler runs only under `qcluster`" fact is **Observed** (Section 8.3).

### Logging routing (where maintenance output lands)

The logging config `[src/paperless/settings.py:373-411]` routes output as follows: the **root** logger writes to the console `[src/paperless/settings.py:407]`; the named logger **`paperless`** → `paperless.log` and **`paperless_mail`** → `mail.log`, both at `DEBUG` `[src/paperless/settings.py:409-410]`:

```bash
$ grep -n '"root"\|"paperless":\|"paperless_mail":\|paperless.log\|mail.log' src/paperless/settings.py
395:            "filename": os.path.join(LOGGING_DIR, "paperless.log"),
402:            "filename": os.path.join(LOGGING_DIR, "mail.log"),
407:    "root": {"handlers": ["console"]},
409:        "paperless": {"handlers": ["file_paperless"], "level": "DEBUG"},
410:        "paperless_mail": {"handlers": ["file_mail"], "level": "DEBUG"},
```

The maintenance loggers — `paperless.tasks` `[src/documents/tasks.py:29]`, `paperless.sanity_checker` `[src/documents/sanity_checker.py:24]`, and `paperless.mail.tasks` `[src/paperless_mail/tasks.py:8]` — are children of `paperless`/`paperless_mail`, so their output lands in `paperless.log` / `mail.log` respectively (and, when run under `qcluster`, on the console as seen throughout this document).

---

## Coverage summary

| User question | Answered in | One-line answer |
|---|---|---|
| Set up the dev environment | "How the runtime environment was brought up" | Redis + `migrate` + `qcluster`, default SQLite config |
| Periodic tasks registered + where schedule is defined | §2, §6 | 4 tasks, seeded by 3 data migrations into `django_q_schedule` |
| Sanity checker: validates / logs / frequency | §3 | Validates thumbnails/originals/archive/content/orphans; logs via `paperless.sanity_checker`; **weekly** |
| Automatic index optimization? DB cleanup? | §4 | Index optimize = **yes, daily**; DB cleanup = **no dedicated task** |
| Failed-doc-retry / stuck-job task? | §5 | **No** such task (consumer's 50-retry file-read is unrelated) |
| "Celery beat schedule" — list recurring tasks + intervals | §1, §2 | **No Celery**; Django Q: HOURLY, DAILY, WEEKLY, MINUTES=10 |
| Trace definition + registration | §6 | tasks.py → migrations → `Q_CLUSTER`/`qcluster` → DB/Redis |
| Failure: retry logic? alerting? | §7 | Runs once, saved as Failure, **no retry**, **no alerting**; `catch_up=False` |
| Startup vs strictly scheduled | §8 | `migrate` seeds; conditional startup reindex; `ready()` = signals only; 4 schedules strictly scheduled |
| DB table for task history / job state | §9 | `django_q_schedule` (state) + `django_q_task` (history); `django_q_ormq` unused |
| What enables/disables these features | §10 | No single switch; `PAPERLESS_*` worker/redis flags; runs only under `qcluster` |

**Investigation integrity:** all observations were produced at commit `542221a38dff06361e07976452f9aea24d210542` in the default configuration (SQLite + local Redis + Django Q). Frequency/timing claims (the four intervals; the ~30 s scheduler cadence) were confirmed stable across at least two runs. All temporary observation scripts and throwaway runtime state were removed; the source tree is unchanged apart from this document.

