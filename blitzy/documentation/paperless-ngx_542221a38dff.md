# paperless-ngx v1.7.0 — Automatic Background Maintenance & Scheduling: An Evidence-Based Investigation

> **Scope of this document.** This is an investigative, read‑only analysis of how **paperless‑ngx v1.7.0** performs automatic background maintenance. It answers eleven specific questions about the project's scheduled/periodic tasks. Every behavioral claim about paperless‑ngx is grounded in an exact `file:line` citation against the source tree, and every claim about the *framework* that runs those tasks is attributed to the Django‑Q 1.3 documentation. No source file was modified to produce this document — the repository was read as the authoritative source of truth.

---

## ⚠️ Headline finding (read this first): the scheduler is **Django‑Q**, not Celery

The questions that motivated this investigation are framed around **Celery** and **"Celery beat."** That mental model does not match this codebase. **paperless‑ngx v1.7.0 has no Celery anywhere.** Its background task queue *and* its periodic scheduler are both provided by **Django‑Q 1.3.9**, a Redis‑backed, Django‑native task queue. The remainder of this document therefore translates every Celery‑oriented question onto its Django‑Q equivalent and answers against the real mechanism.

**Evidence for the headline finding:**

| Claim | Evidence (`file:line`) |
|---|---|
| This is paperless‑ngx **v1.7.0** | `src/paperless/version.py:1` → `__version__ = (1, 7, 0)` |
| The task queue + scheduler is **Django‑Q 1.3.9** | `requirements.txt:37` → `django-q==1.3.9`; `Pipfile:17` → `django-q = "~=1.3"` |
| `django_q` is an installed Django app | `src/paperless/settings.py:110` → `"django_q",` in `INSTALLED_APPS` |
| There is **no Celery application object** | `src/paperless/__init__.py:1-4` imports only `binaries_check` / `paths_check` — no `Celery(...)` app is created at package import |
| There is **no Celery dependency and no `CELERY_BEAT_SCHEDULE`** | A full‑text search for "celery" across all `.py` files returns **zero** matches; `requirements.txt` and `Pipfile` declare no Celery package |

**Thinking / reasoning.** A Celery project is identifiable by three fingerprints: a `celery` dependency, a `Celery(...)` application object (conventionally in a `celery.py` / package `__init__.py`), and a `CELERY_BEAT_SCHEDULE` (or `@periodic_task`/`@shared_task` beat entries) describing periodic work. **None of the three is present here.** Conversely, all three Django‑Q fingerprints *are* present: the pinned dependency (`django-q==1.3.9`), the registered app (`"django_q"` in `INSTALLED_APPS`), and a `Q_CLUSTER` configuration dictionary (`src/paperless/settings.py:449-457`). Because "code is truth," the only defensible conclusion is that the user's Celery premise is incorrect for this version, and the correct subject of every question is Django‑Q. This is the single most important correction in this document, so it leads everything that follows.

### Celery concept → Django‑Q reality (one‑to‑one mapping)

| Celery concept (the user's mental model) | Django‑Q reality in paperless‑ngx v1.7.0 |
|---|---|
| Celery app / `celery.py` | No Celery app exists; Django‑Q is configured via the `Q_CLUSTER` dict — `src/paperless/settings.py:449-457` |
| `celery worker` process | `python3 manage.py qcluster` (worker **and** scheduler), supervised as `[program:scheduler]` — `docker/supervisord.conf:28-29` |
| `celery beat` daemon | The Django‑Q scheduler loop is built **into** the same `qcluster` process — there is **no separate beat process** |
| `CELERY_BEAT_SCHEDULE` dict | `django_q.models.Schedule` rows created inside **data migrations** (not a static settings dict) |
| `@periodic_task` / beat entries | `schedule(...)` calls wrapped in `migrations.RunPython(add_schedules, remove_schedules)` |
| Celery result backend (e.g. `django_celery_results`) | Django‑Q's own ORM tables: `django_q_task` and `django_q_schedule` |
| `task_annotations` retries / `time_limit` | Django‑Q `retry` / `timeout` keys in `Q_CLUSTER` — `src/paperless/settings.py:453-454` |
| Flower / Celery‑events alerting | **None configured.** Django‑Q supports pluggable Sentry/Rollbar error reporters only when an `error_reporter` key is set — paperless sets none |

A quick note on a naming nuance that will otherwise confuse anyone grepping the codebase: the Supervisor *program* is called `scheduler`, but the *command* it runs is `qcluster` (`docker/supervisord.conf:28-29`). There is no program literally named "celery" or "beat."

---

## §1. How to set up the development environment (the runtime stack & run model)

**Answer.** paperless‑ngx v1.7.0 is a **Django 4.0.4** application running on **Python 3.9**, backed by a **Redis** broker, with background work handled by **Django‑Q 1.3.9**. In its canonical container form, three long‑running processes are supervised by **Supervisord**: the web server (`gunicorn`), the filesystem intake worker (`document_consumer`), and the Django‑Q cluster (`qcluster`, which is the worker pool *and* the periodic scheduler). On container startup, `manage.py migrate` runs first and materializes the scheduled jobs into the database.

**Evidence (`file:line`):**

- Python version: `Dockerfile:18` → `FROM python:3.9-slim-bullseye as main-app`.
- Framework & queue pins (`requirements.txt`): `django==4.0.4` (L38), `django-q==1.3.9` (L37), `redis==3.5.3` (L84), `channels==3.0.4` (L23), `channels-redis==3.4.0` (L22), `django-picklefield==3.0.1` (L36, used by Django‑Q to serialize task args/results), `djangorestframework==3.13.1` (L39), `gunicorn==20.1.0` (L42), `scikit-learn==1.0.2` (L88), `whoosh==2.7.4` (L111).
- Redis is a hard runtime dependency: `docker/wait-for-redis.py:19` reads `os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` and blocks startup until Redis answers; `paperless.conf.example:10` documents `PAPERLESS_REDIS`.
- Supervisord run model (`docker/supervisord.conf`): `[program:gunicorn]` at L10‑11 (`gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application`); `[program:consumer]` at L19‑20 (`python3 manage.py document_consumer`); `[program:scheduler]` at L28‑29 (`python3 manage.py qcluster`).
- Web server config: `gunicorn.conf.py` sets `bind` (L3), `workers` (L4), and `worker_class = "paperless.workers.ConfigurableWorker"` (L5).
- Startup migration: `docker/docker-prepare.sh:45` → `python3 manage.py migrate` (run under a file lock), which creates the `Schedule` rows.

**How to run it for behavioral verification.** Build/run the provided Docker image and observe the `[program:scheduler]` (`qcluster`) logs; or, for a host run, start Redis, then `python manage.py migrate` followed by `python manage.py qcluster` from `src/`. Either path lets you watch the Django‑Q scheduler boot and enqueue due jobs. (See §12 for the verification method and artifact‑cleanup commitment.)

**Thinking / reasoning.** "Set up the dev environment" is really a question about *what has to be present for maintenance tasks to run at all*. The decisive facts are (1) the Python/Django/Django‑Q versions, because they fix the scheduling semantics; (2) Redis, because Django‑Q cannot broker tasks without it — `wait-for-redis.py` blocking startup proves the dependency is non‑optional; and (3) the `qcluster` process, because per Django‑Q's own documentation tasks/schedules are only processed when a worker cluster is running. The Supervisord file is the authoritative description of the run model, and `docker-prepare.sh` ordering (`migrate` before the cluster) explains why schedules exist in the database before the scheduler ever looks for them.

---

## §2. Where is the schedule configuration defined?

**Answer.** Two distinct things, neither of which is a Celery beat dictionary:

1. **The cluster/scheduler itself is configured** by the `Q_CLUSTER` dictionary in `src/paperless/settings.py:449-457`. This sets the cluster name, the catch‑up policy, worker recycling, the retry/timeout window, the worker count, and the Redis URL.
2. **The recurring jobs are *registered as data*** — `django_q.models.Schedule` rows created inside **Django data migrations**, not declared in a settings dictionary. There is **no `CELERY_BEAT_SCHEDULE`** anywhere in the project.

**Evidence (`file:line`):** The `Q_CLUSTER` block, verbatim by line:

```python
# src/paperless/settings.py:449-457
Q_CLUSTER = {
    "name": "paperless",                                        # L450
    "catch_up": False,                                          # L451
    "recycle": 1,                                               # L452
    "retry": PAPERLESS_WORKER_RETRY,                            # L453
    "timeout": PAPERLESS_WORKER_TIMEOUT,                        # L454
    "workers": TASK_WORKERS,                                    # L455
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),  # L456
}
```

The worker tunables feeding that block live just above it: `TASK_WORKERS` (L438), `PAPERLESS_WORKER_TIMEOUT` defaulting to `1800` (L440), an explanatory comment that "timeout must be smaller than retry" (L442‑443), and `PAPERLESS_WORKER_RETRY` defaulting to `PAPERLESS_WORKER_TIMEOUT + 10` (L444‑447). The recurring jobs themselves are registered in three data migrations (enumerated in §6 and §7).

**Thinking / reasoning.** The user asked for "the schedule config." In a Celery project that would be one dictionary; here the answer is necessarily two‑part because Django‑Q splits *cluster configuration* (static, in settings) from *schedule definitions* (dynamic rows, created by migrations and thereafter editable at runtime via the Django admin). The absence of `CELERY_BEAT_SCHEDULE` is itself an answer: I searched the settings module and the whole `.py` tree and found no such dictionary and no `celery` import, which is exactly why the question must be redirected to `Q_CLUSTER` plus the migration‑created `Schedule` rows.

---

## §3. The sanity checker: what it validates, its log output, and how often it runs

**Answer.** The sanity checker is `check_sanity()` in `src/documents/sanity_checker.py`. It walks the media tree and, for **every** `Document`, verifies that the expected files exist, are readable, and match their stored checksums, reporting problems at three severities (error / warning / info) through a dedicated logger. It runs **WEEKLY** as a scheduled Django‑Q job, and the scheduled wrapper raises an exception if any *error*‑level problems were found. It can also be invoked manually via a management command.

**Evidence (`file:line`):**

- `check_sanity(progress=False)` — `src/documents/sanity_checker.py:49-133`. It builds the set of present files by walking `settings.MEDIA_ROOT` (L53‑55), removes the media lock file (`media.lock`) from that set (L57‑59), then per `Document` validates:
  - **thumbnail** existence and readability (L62‑72),
  - **original file** existence and an **MD5 checksum** match (L74‑91),
  - **archive file** checksum/filename field pairing, archive‑file existence and readability, and an **archive MD5 checksum** match (L93‑124),
  - flags **empty `content`** as an info message (L127‑128),
  - and finally reports any files left unreferenced as **orphan warnings** (L130‑131).
- Logging: `class SanityCheckMessages` — `src/documents/sanity_checker.py:10-42` — collects messages via `error()` (L14‑15), `warning()` (L17‑18), and `info()` (L20‑21); `log_messages()` (L23‑30) emits each through the `paperless.sanity_checker` logger (defined at L24), mapping severity to `logger.error` / `logger.warning` / `logger.info`; `has_error()` (L38‑39) and `has_warning()` (L41‑42) summarize the outcome.
- The scheduled wrapper: `sanity_check()` — `src/documents/tasks.py:255-267` — calls `sanity_checker.check_sanity()`, then `messages.log_messages()`, and **raises `SanityCheckFailedException("Sanity check failed with errors. See log.")`** (L260‑261) when `messages.has_error()`; otherwise it returns a short warning/info/"No issues detected." summary string. `SanityCheckFailedException` is defined at `src/documents/sanity_checker.py:45-46`.
- Cadence: **WEEKLY**, registered at `src/documents/migrations/1004_sanity_check_schedule.py:10-14`.
- Manual entry point: `src/documents/management/commands/document_sanity_checker.py:22-26` calls `check_sanity(progress=...)` then `messages.log_messages()`.
- Corroborating docs (secondary): `docs/administration.rst:390-403` lists the issue categories, and L408 documents the `document_sanity_checker` command.

**Thinking / reasoning.** Three sub‑questions had to be separated. *What it validates* comes straight from the per‑document branches in `check_sanity()` (thumbnail, original+checksum, archive+checksum, content, orphans). *Its log output* is the severity‑tagged emission in `SanityCheckMessages.log_messages()` through the `paperless.sanity_checker` logger — the severities matter because they drive whether the scheduled task fails. *How often* is fixed by the WEEKLY `Schedule` row in migration 1004. Note the deliberate design: the *library* function `check_sanity()` only collects and logs; the *scheduled wrapper* `sanity_check()` is what escalates errors into a raised exception (so a weekly run with real corruption is recorded as a failed task), while warnings/info do not fail the run. I verified the function's real signature is `check_sanity(progress=False)` (not a `progress_bar_disable` parameter), and use that accurate name here.

---

## §4. Is there automatic index optimization or database cleanup?

**Answer.** **Index optimization: yes** — a **DAILY** scheduled task, `index_optimize`, optimizes the Whoosh full‑text search index. **Database cleanup: no** — there is **no separate scheduled database‑cleanup task** in v1.7.0. I searched all three schedule‑creating migrations, the per‑app `tasks.py` modules, and the `Q_CLUSTER` config, and none registers a database‑pruning/cleanup job. (Django‑Q does retain its own task results, but that retention is governed by the framework's `save_limit`, which paperless does not override — see §10.)

**Evidence (`file:line`):**

- DAILY schedule registration: `src/documents/migrations/1001_auto_20201109_1636.py:15-19` registers `documents.tasks.index_optimize` with name "Optimize the index" and `schedule_type=Schedule.DAILY`.
- The task: `index_optimize()` — `src/documents/tasks.py:32-35`:

  ```python
  # src/documents/tasks.py:32-35
  def index_optimize():
      ix = index.open_index()
      writer = AsyncWriter(ix)
      writer.commit(optimize=True)
  ```

- A *separate* Whoosh helper that the scheduled task does **not** use: `open_index_writer(optimize=False)` is a context manager at `src/documents/index.py:64-74` whose `finally` clause calls `writer.commit(optimize=optimize)` (L74); `open_index(recreate=False)` is at L52‑61. The DAILY `index_optimize` task does **not** call `open_index_writer`; as shown in the code block above (`src/documents/tasks.py:32-35`) it inlines its own `open_index()`, `AsyncWriter(ix)`, and `writer.commit(optimize=True)` sequence directly.
- Corroborating docs (secondary): `docs/administration.rst:355-358` states index optimization "is regularly invoked by the task scheduler."

**Thinking / reasoning.** This is a two‑part question and both parts must be answered from evidence, including the negative. The positive half is direct: a DAILY `Schedule` row points at `index_optimize`, which commits the Whoosh writer with `optimize=True` — the canonical Whoosh "compact/merge segments" operation. The negative half ("database cleanup") is an *absence* claim, so I justify it by what I searched: the four `Schedule` rows (§6), the task modules, and the cluster config — none of them schedules a DB‑cleanup routine. The closest thing to "cleanup" is Django‑Q trimming its own successful‑task history via `save_limit`, but that is framework housekeeping inside `django_q_task`, not an application maintenance task, and paperless does not configure it.

---

## §5. Is there a task that handles failed‑document retries or stuck jobs?

**Answer.** **No — there is no dedicated application‑level retry or stuck‑job recovery task.** paperless‑ngx v1.7.0 does not define a task that scans for failed documents and re‑queues them, nor one that detects and restarts hung jobs. Instead, "stuck" behavior is bounded entirely by **Django‑Q's `timeout` and `retry` settings** in `Q_CLUSTER`. There is no application code that implements retry logic on top of the framework.

**Evidence (`file:line`):**

- The only knobs governing this are framework settings: `Q_CLUSTER["timeout"] = PAPERLESS_WORKER_TIMEOUT` and `Q_CLUSTER["retry"] = PAPERLESS_WORKER_RETRY` — `src/paperless/settings.py:453-454`, with the tunables defined at L438‑447 (`PAPERLESS_WORKER_TIMEOUT` default 1800, `PAPERLESS_WORKER_RETRY` default `timeout + 10`).
- No schedule registers a retry/stuck‑job task: the four `Schedule` rows (§6) are `train_classifier`, `index_optimize`, `sanity_check`, and `process_mail_accounts` — none is a retry/recovery job.
- No custom task model exists to track or drive retries (§10): `src/documents/models.py` defines no task table.

**Framework behavior (Django‑Q 1.3; installed dependency source `django_q/conf.py:125-135`).** In Django‑Q, `timeout` bounds **worker execution**: it is the number of seconds to wait for a worker to finish a task before that worker is terminated and reincarnated — the installed source documents it at `django_q/conf.py:125-126` as "Number of seconds to wait for a worker to finish." `retry` is the **broker acknowledgement / re‑delivery interval**: the number of seconds the broker waits for acknowledgement before re‑presenting a task — the installed source documents it at `django_q/conf.py:133-135` as "Number of seconds to wait for acknowledgement before retrying a task." The framework requires `timeout < retry` (enforced by the check at `django_q/conf.py:138-142`, which warns "Set retry larger than timeout"); if `retry` is smaller than the timeout (or than the task's real duration), the broker can re‑present a task that is still running — potentially producing duplicate runs. This is precisely why paperless defaults `retry = timeout + 10` (`src/paperless/settings.py:444-447`), mirroring the in‑code comment at L442‑443 that "timeout must be smaller than retry."

**Thinking / reasoning.** The honest answer is an absence backed by evidence. I confirmed there is no retry/stuck‑job task by enumerating every scheduled job (none qualifies) and confirming there is no custom task model that could store retry state. What *does* exist is generic, framework‑level resilience: if a worker takes longer than `timeout`, the broker may re‑present the task after `retry`. paperless tunes those two numbers so the re‑present window is always slightly larger than the timeout, avoiding accidental duplicate execution. So the system's answer to "stuck job" is "the broker's timeout/retry window," not a bespoke watchdog task. (See §8 for what happens to a task that genuinely fails.)

---

## §6. ALL recurring tasks and their intervals (the "Celery beat schedule," reframed)

**Answer.** There are **exactly four** recurring tasks. They are not entries in a `CELERY_BEAT_SCHEDULE` dict — they are `django_q.models.Schedule` rows created by three data migrations. Reframed onto the user's question, here is the complete inventory with intervals:

| Task (dotted path) | Schedule name | Interval (`schedule_type`) | Registered in (`file:line`) |
|---|---|---|---|
| `documents.tasks.train_classifier` | "Train the classifier" | **HOURLY** | `src/documents/migrations/1001_auto_20201109_1636.py:10-14` |
| `documents.tasks.index_optimize` | "Optimize the index" | **DAILY** | `src/documents/migrations/1001_auto_20201109_1636.py:15-19` |
| `documents.tasks.sanity_check` | "Perform sanity check" | **WEEKLY** | `src/documents/migrations/1004_sanity_check_schedule.py:10-14` |
| `paperless_mail.tasks.process_mail_accounts` | "Check all e-mail accounts" | **every 10 minutes** (`Schedule.MINUTES`, `minutes=10`) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` |

**Evidence (`file:line`):**

- `train_classifier` (HOURLY) and `index_optimize` (DAILY) are both created in `src/documents/migrations/1001_auto_20201109_1636.py` — the `train_classifier` block at L10‑14 (`schedule_type=Schedule.HOURLY`) and the `index_optimize` block at L15‑19 (`schedule_type=Schedule.DAILY`).
- `sanity_check` (WEEKLY) is created in `src/documents/migrations/1004_sanity_check_schedule.py:10-14` (`schedule_type=Schedule.WEEKLY`).
- `process_mail_accounts` (every 10 minutes) is created in `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15`, with `schedule_type=Schedule.MINUTES` (L13) and `minutes=10` (L14).
- Corroborating docs (secondary): `docs/administration.rst:416` states "Paperless automatically fetches your e-mail every 10 minutes by default."

**Django‑Q `schedule_type` codes (framework reference, `django_q/models.py:163-171`).** Django‑Q stores each schedule's cadence as a single‑character `schedule_type` code. The complete set defined by the installed Django‑Q 1.3.9 is:

| Code | `schedule_type` constant | Meaning |
|---|---|---|
| `O` | `Schedule.ONCE` | Run once |
| `I` | `Schedule.MINUTES` | Every N minutes (uses the `minutes` field) |
| `H` | `Schedule.HOURLY` | Hourly |
| `D` | `Schedule.DAILY` | Daily |
| `W` | `Schedule.WEEKLY` | Weekly |
| `M` | `Schedule.MONTHLY` | Monthly |
| `Q` | `Schedule.QUARTERLY` | Quarterly |
| `Y` | `Schedule.YEARLY` | Yearly |
| `C` | `Schedule.CRON` | Cron expression |

paperless‑ngx uses exactly four of these: `process_mail_accounts` is `I` (`MINUTES`, every 10 minutes), `train_classifier` is `H` (`HOURLY`), `index_optimize` is `D` (`DAILY`), and `sanity_check` is `W` (`WEEKLY`).

**Thinking / reasoning.** This is the literal "list ALL recurring tasks with intervals" question, and the only faithful way to answer it is to read the rows that the scheduler actually acts on. Because Django‑Q persists schedules as database rows seeded by migrations, the authoritative list is the union of the `schedule(...)` calls across the three migrations — not a settings dictionary. I cross‑checked the count: three migrations create four rows (migration 1001 creates two), and no other imports or calls of `django_q.tasks.schedule(...)` register recurring Django‑Q schedules anywhere in the tree. (The only other `schedule(...)` in the source is the unrelated watchdog call `self.observer.schedule(...)` at `src/documents/management/commands/document_consumer.py:188`, which registers a filesystem‑polling observer for the consumption directory — not a Django‑Q periodic task.) The `MINUTES` + `minutes=10` pair is the Django‑Q idiom for "every N minutes," which is why the mail check is the most frequent job.

---

## §7. Where are periodic tasks defined and registered?

**Answer.** Definition and registration are deliberately separate concerns:

- **Definition** (the work each task performs) lives in per‑app `tasks.py` modules — `src/documents/tasks.py` and `src/paperless_mail/tasks.py`.
- **Registration** (turning a function into a recurring schedule) happens inside **data migrations**, each wrapping `schedule(...)` in `migrations.RunPython(add_schedules, remove_schedules)`.

This is distinct from *event‑driven* dispatch — work that is enqueued on demand via `async_task(...)` rather than on a clock (covered in §9).

**Evidence (`file:line`):**

- Task definitions:
  - `src/documents/tasks.py` — module logger `paperless.tasks` (L29); `index_optimize` (L32‑35); `index_reindex` (L38‑45); `train_classifier` (L48‑72); `sanity_check` (L255‑267).
  - `src/paperless_mail/tasks.py` — module logger `paperless.mail.tasks` (L8); `process_mail_accounts()` (L11‑22), which iterates `MailAccount.objects.all()` (L13) and processes each account (L15).
- Registration migrations (all three import `from django_q.models import Schedule` and `from django_q.tasks import schedule`, define `add_schedules`/`remove_schedules`, depend on `("django_q", "0013_task_attempt_count")`, and expose `operations = [migrations.RunPython(add_schedules, remove_schedules)]`):
  - `src/documents/migrations/1001_auto_20201109_1636.py` — `add_schedules` body L9‑19, reverse `remove_schedules` deletes by `func` at L22‑24, django_q dependency at L31, `operations` at L34.
  - `src/documents/migrations/1004_sanity_check_schedule.py` — `add_schedules` L9‑14, `remove_schedules` L17‑18, django_q dependency at L25, `operations` at L28.
  - `src/paperless_mail/migrations/0002_auto_20201117_1334.py` — `add_schedules` L9‑15, `remove_schedules` L18‑19, django_q dependency at L26, `operations` at L29.
- Event‑driven (non‑scheduled) dispatch, for contrast: `src/documents/views.py` imports `async_task` (L28) and calls `async_task("documents.tasks.consume_file", ...)` (L523); `src/paperless_mail/mail.py` imports `async_task` (L11) and calls it (L336) to consume fetched attachments.

**Thinking / reasoning.** "Where are they defined and registered" has two answers because Django‑Q decouples the two. The function bodies are ordinary Python in `tasks.py`; what makes four of them *recurring* is the existence of `Schedule` rows, and those rows are created by migrations rather than declared in settings. Using `RunPython` migrations is a notable design choice: it makes schedule registration **migration‑managed and reversible** — the forward function creates the named schedules and the reverse function deletes them by `func`. Registration runs once because an already‑applied Django migration is not re‑run in the normal migration flow; the `schedule(...)` helper itself is **not** idempotent (`django_q/tasks.py:106-108` raises `IntegrityError` if a schedule with the same name already exists), so it is Django's applied‑migration tracking, not the helper, that prevents duplicate rows. I deliberately contrast this with `async_task(...)` so the reader does not mistake every `tasks.py` function for a scheduled job: `consume_file`, for instance, is dispatched on an event (an upload or a fetched e‑mail), never on a clock.

---

## §8. What happens if a scheduled task fails — retry logic or alerting?

**Answer.** When a scheduled task fails, Django‑Q **persists the failure** (including the traceback) to its task table, and `timeout`/`retry` bound how long a task may run before being re‑presented. There is **no alerting** — no Sentry, Rollbar, or e‑mail error reporter is configured. Two paperless tasks behave notably and oppositely with respect to failure: `train_classifier` **swallows its own exceptions** (so a training error never marks the task failed), while `sanity_check` **deliberately raises** when it finds error‑level problems (so a corrupt library *is* recorded as a failed task).

**Evidence (`file:line`):**

- `train_classifier()` — `src/documents/tasks.py:48-72` — wraps its training body in `try/except` and at L71‑72 does `except Exception as e: logger.warning("Classifier error: " + str(e))`. It does **not** re‑raise, so a training failure is logged at WARNING but the Django‑Q task still completes "successfully."
- `sanity_check()` — `src/documents/tasks.py:255-267` — raises `SanityCheckFailedException("Sanity check failed with errors. See log.")` at L260‑261 when `messages.has_error()`, so genuine corruption surfaces as a failed task.
- No alerting is configured: the `Q_CLUSTER` dict (`src/paperless/settings.py:449-457`) contains **no `error_reporter` key**. Failure information therefore goes only to the `django_q_task` table and the logs.

**Framework behavior (Django‑Q 1.3 documentation).** Per the Django‑Q 1.3 documentation, **"Failures are always saved"** — failed tasks (with their tracebacks) are written to the `django_q_task` table regardless of the `save_limit` setting (which only bounds retention of *successful* results). Alerting is **opt‑in**: Django‑Q exposes a pluggable `error_reporter` mechanism (e.g. Sentry or Rollbar) that activates only when an `error_reporter` key is supplied inside `Q_CLUSTER` (plus the corresponding extra installed). Because paperless supplies none, no external alert is emitted on failure.

**Thinking / reasoning.** The question bundles two things — retry and alerting — and the evidence cleanly separates them. *Retry*: there is no application retry logic; the only mechanism is the broker's `timeout`/`retry` window (§5), and failures are recorded by the framework rather than re‑driven by paperless. *Alerting*: this is an absence I can prove structurally — Django‑Q only alerts when `error_reporter` is set, and that key is simply not in paperless's `Q_CLUSTER`. The two contrasting task behaviors are the most reasoning‑rich detail: `train_classifier` intentionally degrades gracefully (a model that cannot be trained this hour should not generate a noisy failure), whereas `sanity_check` intentionally escalates (silent data corruption would be far worse than a failed task row). Both are explicit choices visible in the code, not framework defaults.

---

## §9. Are there tasks that run on startup vs strictly on schedule?

**Answer.** Yes — three categories must be distinguished:

1. **Startup work** (runs once when the container boots): `manage.py migrate`, which *materializes* the `Schedule` rows, and the signal‑handler wiring in `DocumentsConfig.ready()`.
2. **Scheduled work** (runs only when the `qcluster` scheduler fires it): the four recurring tasks from §6.
3. **Event‑driven work** (runs on demand, neither at startup nor on a clock): `async_task(...)` enqueues such as document consumption.

Importantly, `DocumentsConfig.ready()` does **not** run any maintenance task — it only *connects signal handlers*, which then fire on the `document_consumption_finished` event, i.e. it is event‑driven, not scheduled.

**Evidence (`file:line`):**

- Startup migration that creates the schedules: `docker/docker-prepare.sh:45` → `python3 manage.py migrate`.
- Startup signal wiring: `src/documents/apps.py:11-29` — `DocumentsConfig.ready()` connects six handlers to the `document_consumption_finished` signal (e.g. inbox‑tag, correspondent, document‑type, tag, log‑entry, and add‑to‑index handlers, L22‑27). This runs at app‑load but only *registers* handlers; the handlers fire on document‑consumption events.
- No Celery app is created at import time: `src/paperless/__init__.py:1-4` imports only `binaries_check` / `paths_check` (and lists them in `__all__`).
- Event‑driven enqueues (not startup, not scheduled): `src/documents/views.py:523` (`async_task("documents.tasks.consume_file", ...)`) and `src/paperless_mail/mail.py:336`.
- The recurring tasks run only under the cluster: the `[program:scheduler]` process runs `python3 manage.py qcluster` (`docker/supervisord.conf:28-29`).

**Thinking / reasoning.** The trap in this question is conflating "code that runs at startup" with "scheduled maintenance." The only startup actions relevant to maintenance are (a) `migrate`, which seeds the schedule rows so they *exist* before the scheduler looks for them, and (b) `AppConfig.ready()`, which is Django's standard startup hook — but here it merely *wires signals*, so it is event‑driven rather than a maintenance job. The recurring tasks themselves never run "at startup"; they run when the Django‑Q scheduler loop inside `qcluster` determines a row is due. And the `async_task(...)` calls are a third, separate mode: they are triggered by user/system events (an upload, an e‑mail fetch), proving that not every entry in `tasks.py` is on the scheduler. Separating these three modes is what makes the answer precise.

---

## §10. Which database table tracks task execution history or scheduled‑job state?

**Answer.** Task execution history and scheduled‑job state live **only in Django‑Q's own ORM tables** — there is **no custom paperless task model** in v1.7.0. The two relevant tables are:

- **`django_q_task`** — one row per executed task, including successes and **failures with tracebacks** (the execution *history*).
- **`django_q_schedule`** — one row per recurring schedule, tracking its `next_run`, the id of the last task it spawned, and whether that last run succeeded (the scheduled‑job *state*).

**Evidence (`file:line`):**

- `src/documents/models.py` defines these model classes and **no others**: `MatchingModel` (L19), `Correspondent` (L57), `Tag` (L64), `DocumentType` (L82), `Document` (L88), `Log` (L285), `SavedView` (L316), `SavedViewFilterRule` (L342), `FileInfo` (L386). **None of them tracks task execution or schedule state** — so task history necessarily resides in Django‑Q's tables. (The only `...Task` identifiers anywhere in the tree are in test files, not in `models.py`.)
- The schedules that populate `django_q_schedule` are the four rows from §6; the tasks that populate `django_q_task` are every execution the `qcluster` worker performs.

**Framework behavior (Django‑Q 1.3 documentation).** Per the Django‑Q 1.3 documentation, the framework stores results in `django_q_task` and schedules in `django_q_schedule`; **failures are always saved** to `django_q_task` regardless of `save_limit` (which bounds only successful‑result retention). This is the substrate for "task execution history" even though paperless adds no model of its own.

**Sample read‑only verification queries (SQLite/PostgreSQL):**

```sql
-- Current scheduled jobs and when each will next run
SELECT func, schedule_type, next_run FROM django_q_schedule;

-- Most recent task executions, newest first (success flag + timing)
SELECT func, success, started, stopped
FROM django_q_task
ORDER BY stopped DESC
LIMIT 20;
```

**Thinking / reasoning.** The question presumes a table tracking task execution. The correct response is to (1) point at the real tables and (2) *prove* the absence of a custom one, because a later paperless release does introduce a custom `PaperlessTask` model and it would be easy to import that anachronism. I prove the absence by enumerating every class in `models.py` and showing none is task‑related. The two sample queries are intentionally read‑only `SELECT`s so they can be run during behavioral verification without mutating state — `django_q_schedule` answers "what is scheduled and when next," and `django_q_task` answers "what actually ran and did it succeed," which together are exactly the "execution history / scheduled‑job state" the question asks about.

---

## §11. What controls whether these maintenance features are enabled or disabled?

**Answer.** Control is split across three levels, and notably **individual schedules cannot be toggled by an environment flag**:

1. **Cluster‑wide environment variables** tune *how* the workers run (count, threads, timeout/retry, broker), but do not enable/disable individual jobs.
2. **The `Schedule` rows themselves** determine *which* recurring jobs exist; once a migration has materialized them they exist and run, and the way to disable one is to **edit or delete it via the Django admin** (the `Schedule` model is registered in the Django admin by Django‑Q itself — framework behavior; installed dependency source `django_q/admin.py:107`: `admin.site.register(Schedule, ScheduleAdmin)`) — not via a config flag.
3. **One task self‑disables in code**: `train_classifier` returns early when no tag, document type, or correspondent uses auto‑matching, so it effectively turns itself off when there is nothing to learn.

**Evidence (`file:line`):**

- Worker/broker environment variables (corroborated by docs): `PAPERLESS_TASK_WORKERS` (`docs/configuration.rst:517`), `PAPERLESS_THREADS_PER_WORKER` (L523), `PAPERLESS_WORKER_TIMEOUT` (L564), `PAPERLESS_WORKER_RETRY` (L569), and `PAPERLESS_REDIS` (`paperless.conf.example:10`). These feed the `Q_CLUSTER` tunables at `src/paperless/settings.py:438-447` and `449-457`.
- Schedules exist as data once migrated (§6) — there is no `PAPERLESS_*` flag that conditionally creates or skips a `Schedule` row; the migrations always create all four.
- `train_classifier` self‑disable guard: `src/documents/tasks.py:48-55` returns early unless `Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()` (or the equivalent for `DocumentType` / `Correspondent`) is true.

**Thinking / reasoning.** The user expects a feature flag per maintenance feature, as a Celery‑beat project might offer. The evidence says otherwise: the `PAPERLESS_*` variables are *cluster* tunables (how many workers, how long before timeout/retry, which Redis), not per‑schedule switches. The genuine "enable/disable" surface for an individual job is the `Schedule` row, which lives in the database and is editable through the Django admin — a runtime control, not a deploy‑time flag. The one code‑level exception is `train_classifier`'s self‑disable guard, which is a conditional *within* the task rather than an external toggle; it is worth calling out because it explains why the HOURLY classifier job can be a near no‑op on a fresh install with no auto‑matching configured.

---

## §12. Rationale & verification appendix

### 12.1 How each answer was verified

- **Static source inspection (authoritative).** Every behavioral claim above was traced to a specific `file:line` on the source branch and re‑opened to confirm the cited lines contain what is claimed. The pre‑supplied citations were checked byte‑for‑byte; one was corrected (`check_sanity`'s real parameter is `progress=False`, not `progress_bar_disable=...`) and the accurate name is used throughout §3.
- **The "no Celery" assertion** was verified by full‑text search: `grep -ri celery --include=*.py src/` returns nothing, and `grep -i celery requirements.txt Pipfile` returns nothing. The Django‑Q fingerprints (`django-q==1.3.9`, `"django_q"` in `INSTALLED_APPS`, the `Q_CLUSTER` dict) are all present.
- **Framework semantics** (catch‑up policy, the `timeout < retry` rule, "failures are always saved," the pluggable `error_reporter`, and the `qcluster` requirement) were validated against the **Django‑Q 1.3 documentation**, since those behaviors live in the dependency and are not visible in the repository source. Each is tied above to the activating paperless config line.
- **Optional non‑destructive runtime observation.** The answers can be confirmed at runtime by booting the system (Redis + `manage.py migrate` + `manage.py qcluster`, or the provided Docker image), watching the `[program:scheduler]` / `qcluster` logs as it schedules jobs, and running the read‑only `SELECT` queries from §10 against `django_q_schedule` / `django_q_task`. These observations are read‑only; per the governing constraint, any test artifacts created during such observation (queued test tasks, scratch rows, temporary files) are removed afterward, and **no source file is modified**.

### 12.2 The Django‑Q `catch_up=False` consequence

Per the Django‑Q 1.3 documentation, the scheduler's default is to "play catch up" after downtime — replaying every missed slot until it is current. paperless sets `catch_up: False` (`src/paperless/settings.py:451`), which overrides that: a schedule that was missed during an outage runs **once** when the cluster restarts and then resumes normal timing. Practical consequence: after a multi‑day outage, the WEEKLY sanity check is **not** replayed multiple times — it simply runs once and the next run is scheduled normally. The conservative `recycle: 1` (L452) likewise reflects memory hygiene for OCR‑heavy work: per the Django‑Q docs, `recycle` is the number of tasks a worker handles before being recycled (default 500), so `1` recycles a worker after **every** task.

### 12.3 Capabilities the questions assumed that do **not** exist in v1.7.0

Each of these is a first‑class answer — an evidence‑backed *absence*, not silence:

| Assumed capability | Reality in v1.7.0 | Evidence |
|---|---|---|
| Celery / "Celery beat" schedule | Does not exist — the scheduler is Django‑Q | No "celery" in any `.py`; no Celery in `requirements.txt`/`Pipfile`; `Q_CLUSTER` at `src/paperless/settings.py:449-457` |
| A dedicated failed‑retry / stuck‑job task | Does not exist — bounded only by Django‑Q `timeout`/`retry` | No such `Schedule` row (§6); `src/paperless/settings.py:453-454` |
| Alerting / error reporting on failure | Not configured — no `error_reporter` in `Q_CLUSTER` | `src/paperless/settings.py:449-457` |
| A custom task‑history model | Does not exist — history lives in `django_q_task` / `django_q_schedule` | `src/documents/models.py` (no task model among its classes) |
| A scheduled database‑cleanup task | Does not exist (only the DAILY Whoosh `index_optimize`) | §6 schedule inventory; `src/documents/tasks.py:32-35` |

### 12.4 Manual entry points (for completeness)

Three of the maintenance routines can also be triggered by hand via Django management commands, independent of the scheduler:

- `src/documents/management/commands/document_index.py:20-25` — `reindex` / `optimize` (inside `transaction.atomic()`), calling `index_reindex()` / `index_optimize()`.
- `src/documents/management/commands/document_sanity_checker.py:22-26` — runs `check_sanity(progress=...)` then `messages.log_messages()`.
- `src/documents/management/commands/document_create_classifier.py:20` — runs `train_classifier()` (the same function the HOURLY schedule invokes).

**Thinking / reasoning.** The verification appendix exists to make the whole document falsifiable: a reader can re‑open every cited line, re‑run the two greps, and (optionally) boot the cluster and query the two tables to watch the conclusions hold. The "absences" table is deliberate — investigative honesty requires stating what was searched for and not found, with the same citation rigor as the positive findings. The manual entry points are included because they show the same task functions are reachable both on a schedule and by hand, which reinforces that "scheduled" is a property of the `Schedule` row, not of the function itself.
