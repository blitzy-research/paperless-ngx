# How Paperless‑NGX Processes a Document End‑to‑End — A Runtime‑Grounded Trace

*From a file landing in the consumption directory, through parsing, classification, and full‑text indexing — answered with real observed evidence (captured log lines, real Redis broker contents, real database rows) plus exact `file:line` code citations.*

---

## Environment

All values below were produced by **building and running Paperless‑NGX in its default, canonical configuration** as a normal user would, and then observing the live system. Every behavioural claim is backed by (a) the actual command that produced it, (b) the actual output, and (c) an exact `file:line` citation into the source tree.

| Property | Value | Evidence |
|----------|-------|----------|
| Application | **Paperless‑NGX v1.7.0** | `[src/paperless/version.py:1]` → `__version__ = (1, 7, 0)` |
| Runtime | **Python 3.9** (container base `python:3.9-slim-bullseye`; observed interpreter `Python 3.9.25`) | `[Dockerfile:18]` → `FROM python:3.9-slim-bullseye as main-app`; observed interpreter `Python 3.9.25` matches this base |
| Database | **SQLite** at `DATA_DIR/db.sqlite3` (default) | `[src/paperless/settings.py:299-300]`; PostgreSQL only if `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:304-311]` |
| Message broker + channel layer | **Redis** | Q_CLUSTER broker `[src/paperless/settings.py:449-456]`; Channels layer `[src/paperless/settings.py:178-182]` |
| Queuing framework | **Django‑Q `1.3.9`** — cluster name `paperless`, worker timeout `1800`s | `[requirements.txt:37]`, `[src/paperless/settings.py:449-454]`; registered in `INSTALLED_APPS` as `"django_q"` `[src/paperless/settings.py:110]` |
| Full‑text search | **Whoosh `2.7.4`** | `[requirements.txt:111]`, `[src/documents/index.py:31]` |
| Parser used for the test file | **`TextDocumentParser`** (plain text, weight 10) | `[src/paperless_text/parsers.py:12]`, `[src/paperless_text/signals.py:10-12]` |

> **Read‑only investigation.** The source repository is treated as strictly read‑only. The **only** artifact created by this task is this markdown document. During observation, all runtime data (database, media, consumption directory, logs, scratch) was relocated to scratch directories **outside** the source tree via `PAPERLESS_DATA_DIR`, `PAPERLESS_MEDIA_ROOT`, `PAPERLESS_CONSUMPTION_DIR`, `PAPERLESS_LOGGING_DIR`, and `PAPERLESS_SCRATCH_DIR` (→ `/tmp/pngx-work/...`), the Python venv lived **outside** the repo (`/opt/paperless-venv`), and `PYTHONDONTWRITEBYTECODE=1` was set to prevent `.pyc` writes into the tree. `git status --porcelain` on the source repository returns **empty** (see the Reproduction & Cleanup appendix).

---

## Executive Summary

When a file is dropped into `PAPERLESS_CONSUMPTION_DIR`, the following happens:

1. A long‑running **`document_consumer`** management command watches the directory (via **inotify** by default, or a **watchdog `PollingObserver`** when polling is enabled). On a filesystem event it calls the module‑level function **`_consume(filepath)`**, which logs **`Adding {filepath} to the task queue.`** and submits a task.
2. The task is submitted by calling **`async_task("documents.tasks.consume_file", ...)`** — a function from the **Django‑Q** framework. Django‑Q **constructs the task object** (a Python `dict`), **pickles and HMAC‑signs** it with `SignedPackage`, and **`rpush`es** it onto a Redis list named **`django_q:paperless:q`**.
3. A separate **`qcluster`** worker process **`blpop`s** the package off the list, verifies the signature, unpickles it, and runs **`documents.tasks.consume_file`**, which drives `Consumer.try_consume_file(...)`.
4. Inside that single task, the file's MIME type is detected, the correct parser is selected (a plain `.txt` → `TextDocumentParser`), the text is parsed, a thumbnail is generated, and the document row is written to **`documents_document`**.
5. At the end of consumption, the **`document_consumption_finished`** signal fires **synchronously inside the same task**, invoking six handlers that perform classification/matching (`set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags`), write an admin audit entry (`set_log_entry` → `django_admin_log`), and update the **Whoosh** full‑text index (`add_to_index`).
6. Django‑Q's result monitor records the task's execution history — success or failure — in the **`django_q_task`** table.

Classification and full‑text indexing therefore run **inline within `consume_file`**, *not* as separate queued tasks. The only genuinely separate Django‑Q tasks are the **scheduled** ones seeded at migration time: `train_classifier` (hourly), `index_optimize` (daily), and `sanity_check` (weekly). The queuing framework is unambiguously **Django‑Q with a Redis broker — not Celery** (there are **zero** `celery`/`kombu` references anywhere in the dependency manifests).

---

## R1 — Bring Up the Full Runtime

**Answer.** The canonical runtime is **Redis** (broker + channel layer), the **SQLite** database, and **three long‑running processes** launched exactly as the container's process manager does in `docker/supervisord.conf`:

| Process | Command | Citation |
|---------|---------|----------|
| Web / ASGI | `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` | `[docker/supervisord.conf:11]` (`[program:gunicorn]` `[:10]`) |
| Directory watcher | `python3 manage.py document_consumer` | `[docker/supervisord.conf:20]` (`[program:consumer]` `[:19]`) |
| Django‑Q worker / scheduler | `python3 manage.py qcluster` | `[docker/supervisord.conf:29]` (`[program:scheduler]` `[:28]`) |

### Exact bring‑up commands used

```bash
# Canonical Python 3.9 runtime: a venv with the pinned deps installed
# (mirrors the container base FROM python:3.9-slim-bullseye). Observed interpreter: Python 3.9.25.
source /opt/paperless-venv/bin/activate
python --version                                      # observed: Python 3.9.25

# The Python deps are the pinned set from requirements.txt, already installed in the venv, e.g.:
#   django==4.0.4, django-q==1.3.9, redis==3.5.3, channels==3.0.4, channels-redis==3.4.0,
#   whoosh==2.7.4, watchdog==2.1.7, inotifyrecursive==0.3.5, python-magic==0.4.25, scikit-learn==1.0.2, ...
# System binaries present in this run: redis-server, libmagic1, optipng, convert (ImageMagick), tesseract.

# Canonical environment (all runtime data relocated OUT of the read-only source tree)
export REPO=/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9
export PAPERLESS_REDIS=redis://localhost:6379
export PAPERLESS_DATA_DIR=/tmp/pngx-work/data
export PAPERLESS_MEDIA_ROOT=/tmp/pngx-work/media
export PAPERLESS_CONSUMPTION_DIR=/tmp/pngx-work/consume
export PAPERLESS_LOGGING_DIR=/tmp/pngx-work/log
export PAPERLESS_SCRATCH_DIR=/tmp/pngx-work/scratch
export PAPERLESS_TIME_ZONE=UTC
export PYTHONDONTWRITEBYTECODE=1

# 1. Start Redis (Django-Q broker + Channels layer) and verify
redis-server --daemonize yes                          # (already running in this environment)
redis-cli ping                                        # observed: PONG

# 2. Apply migrations (creates all backing tables + seeds the scheduled tasks)
cd "$REPO/src" && python manage.py migrate            # observed: exit 0, all migrations applied

# 3. Launch ALL THREE canonical long-running processes — exactly as docker/supervisord.conf.
#    Each is run from $REPO/src (required, else "ModuleNotFoundError: paperless").
gunicorn -c "$REPO/gunicorn.conf.py" paperless.asgi:application   # web / ASGI  [docker/supervisord.conf:11]
python manage.py document_consumer                                # watcher     [docker/supervisord.conf:20]
python manage.py qcluster                                         # Django-Q    [docker/supervisord.conf:29]
```

### Observed results

The **complete, unedited** output of the canonical `manage.py migrate` command (producing command shown, then its full stdout; the command returned exit code `0`):

```bash
$ cd "$REPO/src" && python manage.py migrate
```

```
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
```

- `redis-cli ping` → `PONG`.
- `manage.py migrate` → **exit 0**; applied migrations for `admin / auth / authtoken / contenttypes / django_q / documents[0001..1018] / paperless_mail / sessions`. In this run **no system‑check warnings appeared**, because all three binaries that `binaries_check` inspects — `settings.CONVERT_BINARY`, `settings.OPTIPNG_BINARY`, and `"tesseract"` `[src/paperless/checks.py:75]` — were present on `PATH`. (When one is missing, `binaries_check` `[src/paperless/checks.py:66-82]` appends a non‑fatal `Warning(...)` `[src/paperless/checks.py:80]` such as `Paperless can't find convert`.)
- `migrate` created the backing tables **`documents_document`**, **`documents_log`**, **`django_q_task`**, **`django_q_schedule`**, **`django_admin_log`**, and **`auth_user`** (verified present in `/tmp/pngx-work/data/db.sqlite3`).
- `migrate` **seeded the scheduled Django‑Q tasks** — the complete `django_q_schedule` rows `(func, name, schedule_type)`, produced by `python manage.py shell -c "from django_q.models import Schedule; [print((s.func, s.name, s.schedule_type)) for s in Schedule.objects.order_by('id')]"`:

```
('documents.tasks.train_classifier', 'Train the classifier', 'H')
('documents.tasks.index_optimize', 'Optimize the index', 'D')
('documents.tasks.sanity_check', 'Perform sanity check', 'W')
('paperless_mail.tasks.process_mail_accounts', 'Check all e-mail accounts', 'I')
```

  These are seeded by migrations `[src/documents/migrations/1001_auto_20201109_1636.py:10-19]` (`train_classifier`=HOURLY `[:10-14]`, `index_optimize`=DAILY `[:15-19]`) and `[src/documents/migrations/1004_sanity_check_schedule.py:10-14]` (`sanity_check`=WEEKLY); the `process_mail_accounts` schedule belongs to the mail app.
- Migration `[src/documents/migrations/0019_add_consumer_user.py:10]` created the **`consumer`** user (row `(1, 'consumer')`) via `User.objects.create(username="consumer")` — this is the user that `set_log_entry` later attributes the admin audit entry to.
- **Baseline before any file** (`Document.objects.count()`, `Task.objects.count()`, `Log.objects.count()`, `redis-cli LLEN`): `documents_document` = `0`, `django_q_task` = `0`, `documents_log` = `0`, broker `LLEN django_q:paperless:q` = `0`.
- **Web / ASGI process (gunicorn) — launched and health‑verified.** The exact command (run from `$REPO/src`) and its observed startup log:

```
$ gunicorn -c /tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/gunicorn.conf.py paperless.asgi:application
[2026-07-08 07:51:38 +0000] [121151] [INFO] Starting gunicorn 20.1.0
[2026-07-08 07:51:38 +0000] [121151] [INFO] Listening at: http://0.0.0.0:8000 (121151)
[2026-07-08 07:51:38 +0000] [121151] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 07:51:38 +0000] [121151] [INFO] Server is ready. Spawning workers
```

  Health checks against the live server (master PID `121151` + 2 workers):

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/admin/login/
200
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/api/
200
```

  The bind address `0.0.0.0:8000` and worker class `paperless.workers.ConfigurableWorker` come from `gunicorn.conf.py` at the repo root. This is the same web process the container's `[program:gunicorn]` launches `[docker/supervisord.conf:11]`.

> **Runtime‑prerequisite note (accuracy).** Even a plain `.txt` exercises the **`optipng`** binary. The text parser's own `get_thumbnail` renders the PNG with **PIL only** `[src/paperless_text/parsers.py:19-38]`, but the consumer calls `get_optimised_thumbnail` `[src/documents/consumer.py:265]`, whose base implementation `[src/documents/parsers.py:319-338]` runs `settings.OPTIPNG_BINARY` `[src/documents/parsers.py:325]` via `subprocess.Popen(args).wait()` `[src/documents/parsers.py:335]` when `OPTIMIZE_THUMBNAILS` is true (the default `[src/paperless/settings.py:508]`), emitting the `Execute: optipng -silent -o5 ...` line `[src/documents/parsers.py:333]` seen in R3. `optipng` is also one of the three binaries `binaries_check` inspects `[src/paperless/checks.py:75]`. In this canonical run `optipng` was **present** (`/usr/bin/optipng`), so the `consume_file` task completed and wrote the document row (id `1`). Were `optipng` absent or non‑zero, `get_optimised_thumbnail` raises `ParseError("Optipng failed …")` `[src/documents/parsers.py:336]` before the row is stored — a `.txt` is therefore not entirely tool‑free despite routing to the lightest parser.

---

## R2 — Drop a Test File and Follow It (the real entry point)

**Answer.** The document is ingested through the **real entry point** — writing a file into `PAPERLESS_CONSUMPTION_DIR` — **not** the REST upload endpoint and **not** a direct `async_task` call. A plain‑text file routes to the lightest parser, **`TextDocumentParser`** in the `paperless_text` app, which declares `text/plain` → `.txt` with **weight 10** `[src/paperless_text/signals.py:10-12]`. That declaration is registered by connecting to the `document_consumer_declaration` signal in the app's `ready()` `[src/paperless_text/apps.py:11-13]`, and parser dispatch picks the **highest‑weight** declaration in `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]` by iterating `document_consumer_declaration.send(None)` `[src/documents/parsers.py:87]`.

### Command + file used

```bash
printf "Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n" \
  > /tmp/pngx-work/consume/report_2022.txt
md5sum /tmp/pngx-work/consume/report_2022.txt      # 0da8a96bf7f3a377f2acba04d2800b51
wc -c   /tmp/pngx-work/consume/report_2022.txt      # 100
```

**Observed:** the file is **100 bytes** with md5 **`0da8a96bf7f3a377f2acba04d2800b51`**. This exact md5 reappears verbatim as the database **`checksum`** field in R5 — the document's checksum is the md5 hexdigest of the ingested file's bytes `[src/documents/consumer.py:103-104]`.

---

## R3 — Log Messages and Triggered Tasks

**Answer.** All pipeline loggers live under the `paperless.*` namespace. Per the `LOGGING` configuration `[src/paperless/settings.py:373-412]`, the `paperless` logger writes to a rotating file handler (`paperless.log`) `[src/paperless/settings.py:392-395,409]` and propagates to the root `console` `StreamHandler` `[src/paperless/settings.py:387-390,407]`, so every line below appears **both** on stdout and in `paperless.log`. The `verbose` formatter is `[{asctime}] [{levelname}] [{name}] {message}` `[src/paperless/settings.py:377-379]`.

### Ordered log stream — captured verbatim from `/tmp/pngx-work/log/paperless.log`

Producing command (complete, unedited slice for the R2 file `report_2022.txt`):

```bash
$ sed -n '2,14p' /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:51:44,399] [INFO] [paperless.management.consumer] Adding /tmp/pngx-work/consume/report_2022.txt to the task queue.
[2026-07-08 07:52:03,656] [INFO] [paperless.consumer] Consuming report_2022.txt
[2026-07-08 07:52:03,660] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-08 07:52:03,663] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-08 07:52:03,665] [DEBUG] [paperless.consumer] Parsing report_2022.txt...
[2026-07-08 07:52:03,666] [DEBUG] [paperless.consumer] Generating thumbnail for report_2022.txt...
[2026-07-08 07:52:03,688] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/pngx-work/scratch/paperless-q_s8b4s2/thumb.png -out /tmp/pngx-work/scratch/paperless-q_s8b4s2/thumb_optipng.png
[2026-07-08 07:52:04,479] [DEBUG] [paperless.management.consumer] Not consuming file /tmp/pngx-work/consume/__paperless_write_test_121205__: File has moved.
[2026-07-08 07:52:05,639] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 07:52:05,642] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 07:52:05,659] [DEBUG] [paperless.consumer] Deleting file /tmp/pngx-work/consume/report_2022.txt
[2026-07-08 07:52:05,705] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/pngx-work/scratch/paperless-q_s8b4s2
[2026-07-08 07:52:05,705] [INFO] [paperless.consumer] Document 2026-07-08 report_2022 consumption finished
```

Two **real** artifacts appear in this verbatim slice and are explained rather than edited out:

- The **≈19 s gap** between `Adding …` (`07:51:44`) and `Consuming …` (`07:52:03`) is deliberate: `qcluster` (the worker) was started only *after* the broker was snapshotted for R4, so the task sat enqueued on the Redis list in the interim (see the `LLEN 0 → 1 → 0` transition in R4). With `qcluster` already running, the `Consuming` line follows within ≈127 ms — see run 2 (`report_2023`) in Secondary Conditions.
- The line `Not consuming file …/__paperless_write_test_121205__: File has moved.` is **not** part of `report_2022`'s consumption. It is the watcher briefly seeing the transient write‑probe file that `path_check` `[src/paperless/checks.py:19]` creates as `__paperless_write_test_{os.getpid()}__` `[src/paperless/checks.py:27-30]` (via `open(test_file, "w")` `[src/paperless/checks.py:32]`) and then removes `[src/paperless/checks.py:45-46]` to verify the consumption directory is writable; `121205` is the `qcluster` PID. It exits early through the same `_consume` guard that logs `Not consuming file {}: File has moved.` `[src/documents/management/commands/document_consumer.py:51]`.

### Per‑line `file:line` mapping

| Log line | Logger | Source |
|----------|--------|--------|
| `Adding {filepath} to the task queue.` | `paperless.management.consumer` | `[src/documents/management/commands/document_consumer.py:85]` (logger defined `[:24]`) |
| `Consuming {filename}` | `paperless.consumer` | `[src/documents/consumer.py:215]` (`logging_name` set `[:54]`) |
| `Detected mime type: {mime}` | `paperless.consumer` | `[src/documents/consumer.py:221]` (`magic.from_file(...)` `[:219]`) |
| `Parser: {ParserClass}` | `paperless.consumer` | `[src/documents/consumer.py:246]` (parser selected via `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]`) |
| `Parsing {}...` | `paperless.consumer` | `[src/documents/consumer.py:260]` |
| `Generating thumbnail for {}...` | `paperless.consumer` | `[src/documents/consumer.py:263]` |
| `Execute: optipng ...` | `paperless.parsing.text` | base `DocumentParser.get_optimised_thumbnail` `[src/documents/parsers.py:333]` (called by the consumer `[src/documents/consumer.py:265]`); logs via the parser instance's `logging_name` `[src/paperless_text/parsers.py:17]` |
| `Not consuming file …/__paperless_write_test_<pid>__: File has moved.` | `paperless.management.consumer` | transient write‑probe from `path_check` `[src/paperless/checks.py:27-30]`, caught by the `_consume` guard `[src/documents/management/commands/document_consumer.py:51]` — **not** part of the document's consumption (see note above) |
| `Document classification model does not exist (yet), not performing automatic matching.` | `paperless.classifier` | `[src/documents/classifier.py:33]` — on a fresh install `load_classifier()` `[:30]` returns `None` before importing scikit‑learn |
| `Saving record to database` | `paperless.consumer` | `[src/documents/consumer.py:387]` — immediately before `Document.objects.create(...)` `[:398]` |
| `Deleting file {}` | `paperless.consumer` | `[src/documents/consumer.py:349]` (source file removed after storage) |
| `Deleting directory {}` | `paperless.parsing.text` | base `DocumentParser` cleanup `[src/documents/parsers.py:349]` |
| `Document {} consumption finished` | `paperless.consumer` | `[src/documents/consumer.py:373]` — `{}` is the document's `__str__` = `f"{created} {title}"` (no correspondent) `[src/documents/models.py:212-220]`, hence `2026-07-08 report_2022` |

> **Accuracy note on two thumbnail lines.** `Generating thumbnail for {}...` is emitted by the **consumer** (logger `paperless.consumer`, `[src/documents/consumer.py:263]`), while the subsequent `Execute: optipng ...` line is emitted by the **text parser** (logger `paperless.parsing.text`). The log stream above shows this distinction directly in the `[name]` field.

### The ingestion task (named exactly)

The task the watcher submits is **`documents.tasks.consume_file`** `[src/documents/tasks.py:184]`.

### Inline classification/indexing vs. separate scheduled tasks

**Classification, matching, and full‑text indexing run INLINE inside the same `consume_file` task** — they are **not** separate queued tasks. They are driven by the **`document_consumption_finished`** signal `[src/documents/signals/__init__.py:4]`, which is sent from `[src/documents/consumer.py:306]` and wired in `[src/documents/apps.py:22-27]` to **six handlers, in this order** (handler logger `paperless.handlers` `[src/documents/signals/handlers.py:27]`):

1. `add_inbox_tags` `[src/documents/signals/handlers.py:30]`
2. `set_correspondent` `[src/documents/signals/handlers.py:35]`
3. `set_document_type` `[src/documents/signals/handlers.py:101]`
4. `set_tags` `[src/documents/signals/handlers.py:168]`
5. `set_log_entry` `[src/documents/signals/handlers.py:413]`
6. `add_to_index` `[src/documents/signals/handlers.py:428]`

> **Fresh‑install nuance (observed).** On a fresh install (no correspondents/types/tags/matching rules, no trained classifier, no inbox tag), the four matching handlers assign nothing, so their INFO logs (`Assigning correspondent ...`, `Assigning document type ...`, `Tagging ...`) do **not** appear in the stream — the only classification‑related line is the `paperless.classifier ... not performing automatic matching` line. `set_log_entry` still writes the `django_admin_log` ADDITION and `add_to_index` still updates the Whoosh index — both **silently** (no INFO line). Matching itself is implemented in `match_correspondents` `[src/documents/matching.py:21]`, `match_document_types` `[:34]`, and `match_tags` `[:47]`, supporting ANY/ALL/LITERAL/REGEX/FUZZY/AUTO algorithms `[src/documents/models.py:21-26]`.

**Separate SCHEDULED Django‑Q tasks (distinct from `consume_file`).** When `qcluster` starts, the seeded schedules fire on their own cadence and appear as their **own `django_q_task` rows**:

- `documents.tasks.train_classifier` `[src/documents/tasks.py:48]` — seeded **HOURLY** `[src/documents/migrations/1001_auto_20201109_1636.py:10-14]`
- `documents.tasks.index_optimize` `[src/documents/tasks.py:32]` — seeded **DAILY** `[src/documents/migrations/1001_auto_20201109_1636.py:15-19]`
- `documents.tasks.sanity_check` `[src/documents/tasks.py:255]` — seeded **WEEKLY** `[src/documents/migrations/1004_sanity_check_schedule.py:10-14]`

These were observed firing as their own `django_q_task` rows after `qcluster` startup (producing command + complete output):

```bash
$ python manage.py shell -c "
from django_q.models import Task
for fn in ['documents.tasks.train_classifier','documents.tasks.index_optimize',
           'documents.tasks.sanity_check','paperless_mail.tasks.process_mail_accounts']:
    for t in Task.objects.filter(func=fn).order_by('started'):
        print('%-45s success=%s result=%r' % (t.func, t.success, t.result))"
documents.tasks.train_classifier              success=True result=None
documents.tasks.index_optimize                success=True result=None
documents.tasks.sanity_check                  success=True result='No issues detected.'
paperless_mail.tasks.process_mail_accounts    success=True result='No new documents were added.'
```

`train_classifier`/`index_optimize` return `None` (nothing to train/optimise on a fresh index); `sanity_check` returns `'No issues detected.'`; the interval‑scheduled `process_mail_accounts` (mail app) returns `'No new documents were added.'` each tick. All are **separate scheduled tasks**, **not** part of the inline consumption pipeline that runs inside `consume_file`.


---

## R4 — Inspect the Queued Task on the Broker (transitional states)

> **Citation provenance (read this once).** Paths beginning with **`src/...`** are **repository** files. **Django‑Q is a pinned pip dependency** (`django-q==1.3.9` `[requirements.txt:37]`), **not** part of this repository's tree; its internals are therefore cited as **`[installed: django_q/<file>:line]`**, meaning the installed package at `/opt/paperless-venv/lib/python3.9/site-packages/django_q/`. Every such reference is additionally corroborated by observed runtime output shown alongside it.

**Answer.** The queued task lives on a **Redis list** named **`django_q:paperless:q`**. Django‑Q constructs this key as `f"django_q:{list_key}:q"`, where `list_key` defaults to `Conf.PREFIX` — the cluster name **`paperless`** `[src/paperless/settings.py:449-450]`. Confirmed against the installed framework source `[installed: django_q/brokers/redis_broker.py:13-15]`:

```python
class Redis(Broker):
    def __init__(self, list_key: str = Conf.PREFIX):
        super(Redis, self).__init__(list_key=f"django_q:{list_key}:q")
```

Enqueue is a Redis **`rpush`** `[installed: django_q/brokers/redis_broker.py:17-18]` and dequeue is a blocking **`blpop`** `[installed: django_q/brokers/redis_broker.py:20-23]`.

### Transitional states (before / during / after) — observed

Producing commands and their exact output at each point:

```
# BEFORE (nothing queued)
$ redis-cli KEYS '*'
(empty)
$ redis-cli LLEN django_q:paperless:q
0

# DURING (document_consumer up, report_2022.txt dropped, qcluster NOT yet started)
$ redis-cli KEYS '*'
django_q:paperless:q
$ redis-cli LLEN django_q:paperless:q
1

# AFTER (qcluster started -> worker blpop drains the list)
$ redis-cli LLEN django_q:paperless:q
0
```

The broker list therefore transitions **empty → one signed package → drained**. At the DURING point the consumer had logged `Adding /tmp/pngx-work/consume/report_2022.txt to the task queue.` and Django‑Q logged `INFO Enqueued 1`.

### Raw queued payload (observed, complete — not truncated)

The queued entry is a **447‑byte ASCII string** in the form `<urlsafe‑base64(pickle)>:<base62‑timestamp>:<HMAC‑signature>` produced by Django‑Q's signing framework (a `TimestampSigner`; note the `-` in the body confirms the URL‑safe base64 alphabet). The **complete** raw value, via `redis-cli --no-raw LINDEX django_q:paperless:q 0`:

```
"gAWVHgEAAAAAAAB9lCiMAmlklIwgZjEwODM4NGM1YjVhNGU5MTk4MDJjYTViM2UyYjJhZDmUjARuYW1llIwPcmVwb3J0XzIwMjIudHh0lIwEZnVuY5SMHGRvY3VtZW50cy50YXNrcy5jb25zdW1lX2ZpbGWUjARhcmdzlIwmL3RtcC9wbmd4LXdvcmsvY29uc3VtZS9yZXBvcnRfMjAyMi50eHSUhZSMBmt3YXJnc5R9lIwQb3ZlcnJpZGVfdGFnX2lkc5ROc4wHc3RhcnRlZJSMCGRhdGV0aW1llIwIZGF0ZXRpbWWUk5RDCgfqBwgHMywGHX-UaA6MCHRpbWV6b25llJOUaA6MCXRpbWVkZWx0YZSTlEsASwBLAIeUUpSFlFKUhpRSlHUu:1whN4K:GrnTy65RGfk8xJaN2PoHchuVjlg48Yzgu0er8bnvGQk"
```

**Pickle protocol — measured, not assumed.** The `gAWV` prefix decodes to the pickle **PROTO** opcode `\x80` followed by protocol byte `\x05` (and the start of a `FRAME` opcode `\x95`), i.e. **pickle protocol 5** — which is `pickle.HIGHEST_PROTOCOL` on Python 3.9. Verified directly against the interpreter **and** the actual payload body:

```
$ python -c "import pickle, base64; print('HIGHEST_PROTOCOL =', pickle.HIGHEST_PROTOCOL); print('b64decode(gAWV) =', base64.b64decode('gAWV'))"
HIGHEST_PROTOCOL = 5
b64decode(gAWV) = b'\x80\x05\x95'
```

It is **not** zlib‑compressed: Django‑Q's `Conf.COMPRESSED` defaults to `False` `[installed: django_q/conf.py:116]` and Paperless does not enable compression.

### Decoded package (observed)

Decoding in Django context via `django_q.signing.SignedPackage.loads(raw)` (producing command shown under "Commands used" below):

```
python                    : 3.9.25
pickle.HIGHEST_PROTOCOL   : 5
base64.b64decode('gAWV')  : b'\x80\x05\x95'
pickled body first 3 bytes: b'\x80\x05\x95'   # \x80 = PROTO opcode, 0x05 = protocol 5
Conf.PREFIX (salt)        : 'paperless'        # == cluster name  [src/paperless/settings.py:450]
Conf.COMPRESSED           : False
Conf.SECRET_KEY           : <set, len 50>      # HMAC key = Django SECRET_KEY
type(decoded)             : dict
decoded.keys()            : ['args', 'func', 'id', 'kwargs', 'name', 'started']
decoded['func']           : 'documents.tasks.consume_file'
decoded['args']           : ('/tmp/pngx-work/consume/report_2022.txt',)
decoded['kwargs']         : {'override_tag_ids': None}
decoded['name']           : 'report_2022.txt'
decoded['id']             : 'f108384c5b5a4e919802ca5b3e2b2ad9'
decoded['started']        : datetime.datetime(2026, 7, 8, 7, 51, 44, 400767, tzinfo=datetime.timezone.utc)
```

> **End‑to‑end traceability (observed).** This broker snapshot used the **same** `report_2022.txt` as the R2/R5 database trace. The decoded task **`id` = `f108384c5b5a4e919802ca5b3e2b2ad9`** reappears verbatim as the **`django_q_task.id`** in R5(b), and `decoded['args']` points at the same consumption‑dir path — so the queued package and the persisted task‑history row are demonstrably the **same task**. The **structure** — a signed `dict` with keys `args/func/id/kwargs/name/started` and `func == documents.tasks.consume_file` — is identical regardless of which file is dropped.

### Commands used

```bash
redis-cli LLEN django_q:paperless:q
redis-cli --no-raw LINDEX django_q:paperless:q 0
# decode inside the Django shell (so Conf.SECRET_KEY / Conf.PREFIX are loaded):
python manage.py shell   # then:
#   import redis, base64, pickle
#   from django_q.signing import SignedPackage
#   from django_q.conf import Conf
#   raw = redis.Redis('localhost', 6379).lindex('django_q:paperless:q', 0)
#   print(pickle.HIGHEST_PROTOCOL, base64.b64decode(b'gAWV'))
#   print(Conf.PREFIX, Conf.COMPRESSED, len(Conf.SECRET_KEY))
#   print(SignedPackage.loads(raw))
```

**Conclusion.** The broker entry is a **pickled‑then‑HMAC‑signed task dictionary** produced by Django‑Q's `SignedPackage.dumps` `[installed: django_q/signing.py:13-21]`, which pickles the dict with `pickle.dumps(obj, protocol=pickle.HIGHEST_PROTOCOL)` — protocol **5** on Python 3.9 — via `PickleSerializer` `[installed: django_q/signing.py:34-35]`, and then signs it with `signing.dumps(..., key=Conf.SECRET_KEY, salt=Conf.PREFIX, serializer=PickleSerializer)` `[installed: django_q/signing.py:15-21]` (the module aliases `from django_q import core_signing as signing` `[installed: django_q/signing.py:4]`). Only a cluster sharing the same `SECRET_KEY` **and** cluster name (`paperless`) can unpack it. The dict's keys exactly match the task package built by `async_task` (see R6).

---

## R5 — Query the Database After Processing

Queries were run via the Django ORM (`manage.py shell`). The document record, the parsed‑metadata fields, and the task‑execution history are shown below with the exact observed values.

### (a) The document record — table `documents_document` `[src/documents/models.py:88]`

Producing command + complete output:

```bash
$ python manage.py shell -c "
from documents.models import Document
d = Document.objects.get(id=1)
for f in ['id','title','content','mime_type','checksum','archive_checksum',
          'created','modified','added','storage_type','filename']:
    print(f + ' = ' + repr(getattr(d, f)))"
id = 1
title = 'report_2022'
content = 'Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n'
mime_type = 'text/plain'
checksum = '0da8a96bf7f3a377f2acba04d2800b51'
archive_checksum = None
created = datetime.datetime(2026, 7, 8, 7, 51, 43, 393898, tzinfo=datetime.timezone.utc)
modified = datetime.datetime(2026, 7, 8, 7, 52, 5, 659008, tzinfo=datetime.timezone.utc)
added = datetime.datetime(2026, 7, 8, 7, 52, 5, 642889, tzinfo=datetime.timezone.utc)
storage_type = 'unencrypted'
filename = '0000001.txt'
```

These parsed‑metadata fields are written by `Document.objects.create(...)` in `[src/documents/consumer.py:398-404]`. `content` is the full parsed text produced by `TextDocumentParser` (its `parse()` reads the file directly `[src/paperless_text/parsers.py:40-42]`); `checksum` (`0da8a96bf7f3a377f2acba04d2800b51`) is the md5 hexdigest of the file bytes computed at `[src/documents/consumer.py:402]` and **equals the md5 of the dropped file** from R2; `title` defaults to the filename stem `[src/documents/consumer.py:399]`; `mime_type` is the detected type from R3.

### (b) Task‑execution history — table `django_q_task` (Django‑Q `Task` model `[installed: django_q/models.py:20]`)

Producing command + complete output:

```bash
$ python manage.py shell -c "
from django_q.models import Task
t = Task.objects.get(name='report_2022.txt')
for f in ['id','name','func','args','success','result','started','stopped']:
    print(f + ' = ' + repr(getattr(t, f)))"
id = 'f108384c5b5a4e919802ca5b3e2b2ad9'
name = 'report_2022.txt'
func = 'documents.tasks.consume_file'
args = ('/tmp/pngx-work/consume/report_2022.txt',)
success = True
result = 'Success. New document id 1 created'
started = datetime.datetime(2026, 7, 8, 7, 51, 44, 400767, tzinfo=datetime.timezone.utc)
stopped = datetime.datetime(2026, 7, 8, 7, 52, 5, 708215, tzinfo=datetime.timezone.utc)
```

**End‑to‑end traceability:** this task `id` **`f108384c5b5a4e919802ca5b3e2b2ad9`** is byte‑for‑byte the same `id` decoded from the Redis broker package in R4, and its `result` names the `documents_document` row (`id 1`) from (a) — the queued package, the executed task, and the stored document are one and the same unit of work. Django‑Q's result monitor persists **both** successes and failures here. The `result` string `Success. New document id {} created` is produced by `consume_file` `[src/documents/tasks.py:247]`. Failed runs (the duplicate/unsupported cases) are stored with `success = False` and the traceback/error text as `result` — demonstrated with real rows in the Secondary Conditions section.

### (c) Admin audit entry — table `django_admin_log`, written by `set_log_entry` `[src/documents/signals/handlers.py:413]`

Producing command + complete output (one ADDITION row per consumed document):

```bash
$ python manage.py shell -c "
from django.contrib.admin.models import LogEntry
for e in LogEntry.objects.order_by('id'):
    print('action_flag=%s user=%s content_type=%s object_id=%s object_repr=%s'
          % (e.action_flag, e.user.username, e.content_type, e.object_id, e.object_repr))"
action_flag=1 user=consumer content_type=documents | document object_id=1 object_repr=2026-07-08 report_2022
action_flag=1 user=consumer content_type=documents | document object_id=2 object_repr=2026-07-08 report_2023
action_flag=1 user=consumer content_type=documents | document object_id=3 object_repr=2026-07-08 report_poll
```

`action_flag = 1` is Django's `ADDITION`; the acting `user` is **`consumer`**, auto‑created by migration `[src/documents/migrations/0019_add_consumer_user.py:10]`; `object_repr` is the document's `__str__` (`f"{created} {title}"` `[src/documents/models.py:220]`). `set_log_entry` `[src/documents/signals/handlers.py:413]` writes it via `LogEntry.objects.create(action_flag=ADDITION, …, object_repr=document.__str__())` `[src/documents/signals/handlers.py:418-425]`, using the `consumer` user `[:416]`.

### (d) Accuracy nuance — `documents_log` is defined but NOT written by the consumption path

The `documents_log` table / `Log` model `[src/documents/models.py:285]` is **defined but not written** by the v1.7.0 consumption path. Observed: `Log.objects.count()` = **`0`** after successful consumption. The reason is that the `LOGGING` configuration `[src/paperless/settings.py:373-412]` has **no database handler** — only a console `StreamHandler` `[:387-390]`, a `ConcurrentRotatingFileHandler` writing `paperless.log` `[:392-395]`, and a mail file handler `[:400-402]`. Processing log lines therefore flow to **stdout and `paperless.log`** (via the `LoggingMixin.log` group‑UUID convention `[src/documents/loggers.py:14]`, which attaches `extra={"group": self.logging_group}` `[:21]`), never to `documents_log`. This is reported as **observed**, not assumed.

### (e) Full‑text index (ties R5 to indexing)

After consumption, `add_to_index` `[src/documents/signals/handlers.py:428]` → `index.add_or_update_document` `[src/documents/index.py:118]` (schema `[src/documents/index.py:31]`) maintains the Whoosh index under `/tmp/pngx-work/data/index/`. Producing command + complete output (listing taken in **this run** after all three documents were consumed). The exact set of `.seg` segment files and the `.toc` filename are internal Whoosh details that vary run‑to‑run with buffered writes, flushes, and segment merges — so the number of `.seg` files does **not** necessarily equal the document count (here, three consumed documents left **two** `.seg` segments):

```bash
$ ls -la /tmp/pngx-work/data/index/
total 44
drwxr-xr-x 2 root root  4096 Jul  8 07:52 .
drwxr-xr-x 3 root root  4096 Jul  8 07:52 ..
-rwxr-xr-x 1 root root     0 Jul  8 07:52 MAIN_WRITELOCK
-rw-r--r-- 1 root root 11715 Jul  8 07:52 MAIN_hkqcavnvt7n33d57.seg
-rw-r--r-- 1 root root 14480 Jul  8 07:52 MAIN_l50ljmbjyz6r6jbh.seg
-rw-r--r-- 1 root root  4486 Jul  8 07:52 _MAIN_4.toc
```


---

## R6 — Trace the Codepath and Name the Framework

**Codepath (each step named with its `file:line`):**

**1. The `document_consumer` management command** `[src/documents/management/commands/document_consumer.py]` watches `PAPERLESS_CONSUMPTION_DIR`. Its `handle()` `[:156]` first performs a one‑time synchronous scan of any pre‑existing files (`os.scandir` → `_consume` `[:172-173]`, or recursive `os.walk` → `_consume` `[:167-170]`), then chooses a watcher `[:178-181]`. **The two watcher modes reach `_consume` by different routes — this distinction matters:**

- **inotify (default, `CONSUMER_POLLING == 0`)** `[:178]` → `handle_inotify` `[:199]` logs `Using inotify to watch directory for changes:` `[:200]`, watches with flags `CLOSE_WRITE | MOVED_TO` `[:203]`, and runs its **own** debounce loop (`inotify_debounce = 0.5` `[:211]`); once a file has been quiet past the debounce it calls the module‑level **`_consume(filepath)` directly** `[:230]`. This path does **not** use the `Handler` class.
- **polling (`PAPERLESS_CONSUMER_POLLING` > 0)** `[:181]` → `handle_polling` `[:185]` logs `Polling directory for changes:` `[:186]` and schedules a watchdog **`PollingObserver`** `[:187]` with a `Handler()` `[:188]`. Filesystem events dispatch to **`Handler.on_created`** `[:129]` / **`Handler.on_moved`** `[:132]`, each of which starts a **`Thread` targeting `_consume_wait_unmodified`** `[:130]` / `[:133]`. `_consume_wait_unmodified(file)` `[:99]` logs `Waiting for file {} to remain unmodified` `[:103]`, polls the file's `mtime`/`size` up to `CONSUMER_POLLING_RETRY_COUNT` times `[:107]` (sleeping `CONSUMER_POLLING_DELAY` between tries `[:122]`), and **only once the file is stable calls `_consume(file)`** `[:118]` — or logs a timeout `[:125]` if it never settles.

So both routes converge on the module‑level **`_consume(filepath)`** `[:46]`, but via different mechanisms: **inotify** calls it directly from its debounce loop `[:230]`; **polling** reaches it via `Handler` `[:129/:132]` → a `Thread` `[:130/:133]` → `_consume_wait_unmodified` `[:99]` → `_consume` `[:118]`.

**2. `_consume(filepath)`** `[:46]` then:

- skips directories/ignored paths `[:47]`;
- returns early if the file has moved, logging `Not consuming file {}: File has moved.` `[:51]`;
- applies the **extension pre‑filter** `if not is_file_ext_supported(...)` `[:54]` (predicate in `[src/documents/parsers.py:62]`), logging `Not consuming file {}: Unknown file extension.` `[:55]` and returning if the extension is unsupported;
- then logs **`Adding {filepath} to the task queue.`** `[:85]` and **submits the task**:

```python
async_task(
    "documents.tasks.consume_file",
    filepath,
    override_tag_ids=tag_ids if tag_ids else None,
    task_name=os.path.basename(filepath)[:100],
)   # src/documents/management/commands/document_consumer.py:86-91
```

**3. `async_task`** (Django‑Q, `[installed: django_q/tasks.py:20]`) is the function that **constructs the task object**. In the installed `django-q==1.3.9`, it:

- builds the task `dict` `{"id": ..., "name": ..., "func": func, "args": args}` `[installed: django_q/tasks.py:40-47]`;
- sets `task["kwargs"] = keywords` `[installed: django_q/tasks.py:64]` and `task["started"] = timezone.now()` `[installed: django_q/tasks.py:65]`;
- signs it with `pack = SignedPackage.dumps(task)` `[installed: django_q/tasks.py:69]`;
- enqueues it via `enqueue_id = broker.enqueue(pack)` `[installed: django_q/tasks.py:73]` and logs `Enqueued {enqueue_id}` `[installed: django_q/tasks.py:74]` (the `INFO Enqueued 1` line observed in R4).

This exactly explains the decoded broker dict in R4 (keys `args / func / id / kwargs / name / started`).

**4. The `qcluster` worker** pops the package (`blpop` `[installed: django_q/brokers/redis_broker.py:20-23]`), verifies the HMAC signature and unpickles it, then runs **`documents.tasks.consume_file`** `[src/documents/tasks.py:184]`, which (absent barcodes) constructs `Consumer().try_consume_file(...)` `[src/documents/tasks.py:236]` → `[src/documents/consumer.py:180]` — the rest of the pipeline traced in R3.

### The precise answer to "which part constructs the task, and which framework dispatches it"

- **What constructs the task object:** Paperless's `_consume` calls `async_task(...)` `[src/documents/management/commands/document_consumer.py:86-91]`; the **`async_task` function itself builds the task `dict`** `[installed: django_q/tasks.py:40-47]`, then signs `[installed: django_q/tasks.py:69]` and enqueues `[installed: django_q/tasks.py:73]` it.
- **Which queuing framework dispatches it:** **Django‑Q** (`django-q==1.3.9` `[requirements.txt:37]`; registered as `"django_q"` in `INSTALLED_APPS` `[src/paperless/settings.py:110]`) with a **Redis broker** `[src/paperless/settings.py:449-456]`.

### It is Django‑Q — explicitly NOT Celery

There are **zero** `celery`/`kombu` references in **either** dependency manifest:

```bash
$ grep -inE 'celery|kombu' requirements.txt   # (no matches)
$ grep -inE 'celery|kombu' Pipfile            # (no matches)
```

The queuing stack is Django‑Q + Redis end to end.


---

## Secondary Conditions (every condition the question implies)

Beyond the primary happy path, the following secondary/error/edge/alternate‑flag/transitional conditions were exercised and observed.

### 1. Stable log ordering across ≥2 runs

A second happy file `report_2023.txt` (md5 `41a6465cc9c6ae26bf82fef3e87aadce`, which became `documents_document` id `2`) produced a log sequence whose ordering is **identical** to the first run — only the filename, timestamps, and the scratch‑dir hash differ. This run's `Adding → Consuming` gap is ≈127 ms (`07:52:08.999 → 07:52:09.126`) because `qcluster` was already running — confirming the ≈19 s gap in R3 was purely the deliberate broker‑snapshot delay.

**Input file — producing command + md5/size:**

```bash
$ printf 'Paperless NGX runtime trace test.\nBeta Corporation invoice.\nDated 2023-04-01. Amount due 84.00 USD.\n' \
    > /tmp/pngx-work/consume/report_2023.txt
$ md5sum /tmp/pngx-work/consume/report_2023.txt
41a6465cc9c6ae26bf82fef3e87aadce  /tmp/pngx-work/consume/report_2023.txt
$ wc -c   /tmp/pngx-work/consume/report_2023.txt
100 /tmp/pngx-work/consume/report_2023.txt
```

**Log‑extraction command + complete, unedited output** — the contiguous `Adding … → … consumption finished` window for `report_2023.txt`:

```bash
$ awk '/Adding .*report_2023\.txt to the task queue/{f=1} f{print} /report_2023 consumption finished/{exit}' \
    /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:52:08,999] [INFO] [paperless.management.consumer] Adding /tmp/pngx-work/consume/report_2023.txt to the task queue.
[2026-07-08 07:52:09,126] [INFO] [paperless.consumer] Consuming report_2023.txt
[2026-07-08 07:52:09,130] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-08 07:52:09,132] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-08 07:52:09,135] [DEBUG] [paperless.consumer] Parsing report_2023.txt...
[2026-07-08 07:52:09,135] [DEBUG] [paperless.consumer] Generating thumbnail for report_2023.txt...
[2026-07-08 07:52:09,156] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/pngx-work/scratch/paperless-fj3zwo9f/thumb.png -out /tmp/pngx-work/scratch/paperless-fj3zwo9f/thumb_optipng.png
[2026-07-08 07:52:11,049] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 07:52:11,052] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 07:52:11,068] [DEBUG] [paperless.consumer] Deleting file /tmp/pngx-work/consume/report_2023.txt
[2026-07-08 07:52:11,113] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/pngx-work/scratch/paperless-fj3zwo9f
[2026-07-08 07:52:11,114] [INFO] [paperless.consumer] Document 2026-07-08 report_2023 consumption finished
```

**Normalisation + diff — producing commands + complete output.** To prove the ordering is identical (not merely similar), each run's `Adding … → … consumption finished` window is normalised — the transient write‑probe line is dropped and the three volatile tokens (timestamp, file stem, scratch‑dir hash) are masked — then compared with `diff`:

```bash
$ normalize() {
    grep -v '__paperless_write_test_' "$1" \
    | sed -E \
        -e 's/^\[[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9:,]+\]/[TS]/' \
        -e 's/report_202[0-9]/DOC/g' \
        -e 's#paperless-[a-z0-9_]+#paperless-HASH#g'
  }
$ normalize run1_report_2022.window > run1.norm   # the R3 Adding→finished window (write-probe line dropped)
$ normalize run2_report_2023.window > run2.norm   # the window shown just above
$ diff run1.norm run2.norm && echo "IDENTICAL"
IDENTICAL
```

Both normalised windows are **12 lines** and compare **byte‑for‑byte equal**, confirming stable log ordering across the two runs.

### 2. Duplicate file (pre‑existing checksum)

Re‑creating the exact content of an already‑consumed document — `duplicate_of_2023.txt` carries the **same** content (hence same md5 `41a6465cc9c6ae26bf82fef3e87aadce`) as `report_2023.txt` from condition 1 — triggers `pre_check_duplicate` `[src/documents/consumer.py:213]` (definition `[:102]`, checksum compare `[:104-107]`), which runs **before** the `Consuming` log `[:215]`.

**Input file — producing command + md5** (identical bytes to `report_2023.txt`, hence the identical checksum that trips the duplicate check):

```bash
$ printf 'Paperless NGX runtime trace test.\nBeta Corporation invoice.\nDated 2023-04-01. Amount due 84.00 USD.\n' \
    > /tmp/pngx-work/consume/duplicate_of_2023.txt
$ md5sum /tmp/pngx-work/consume/duplicate_of_2023.txt
41a6465cc9c6ae26bf82fef3e87aadce  /tmp/pngx-work/consume/duplicate_of_2023.txt
```

**Log‑extraction command + complete, unedited output:**

```bash
$ awk '/Adding .*duplicate_of_2023\.txt to the task queue/{f=1} f{print} /It is a duplicate/{exit}' \
    /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:52:15,025] [INFO] [paperless.management.consumer] Adding /tmp/pngx-work/consume/duplicate_of_2023.txt to the task queue.
[2026-07-08 07:52:15,156] [ERROR] [paperless.consumer] Not consuming duplicate_of_2023.txt: It is a duplicate.
```

The worker records the task as **`success = False`** in `django_q_task`, with the full traceback as the `result` (producing command + complete output):

```bash
$ python manage.py shell -c "
from django_q.models import Task
t = Task.objects.filter(name='duplicate_of_2023.txt', success=False).order_by('started').first()
print('id      =', t.id)
print('name    =', t.name)
print('func    =', t.func)
print('success =', t.success)
print('result  ='); print(t.result)"
id      = 9452db958bd54e3480196210bfdbd051
name    = duplicate_of_2023.txt
func    = documents.tasks.consume_file
success = False
result  =
duplicate_of_2023.txt: Not consuming duplicate_of_2023.txt: It is a duplicate. : Traceback (most recent call last):
  File "/opt/paperless-venv/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: duplicate_of_2023.txt: Not consuming duplicate_of_2023.txt: It is a duplicate.
```

The traceback confirms the exact codepath: `consume_file` `[src/documents/tasks.py:236]` → `try_consume_file` → `pre_check_duplicate` `[src/documents/consumer.py:213]` → `_fail` at `[:110]` → `raise ConsumerError` `[:81]`. The message constant is `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` `[src/documents/consumer.py:37]`. Note that **no `Consuming`/mime lines appear** because the duplicate check precedes them.

### 3. Unsupported MIME — two distinct rejection paths

**(a) Watcher‑level (unknown extension).** Dropping `archive.zip` is rejected **before queuing** — no task is created.

**Input file — producing command** (`archive.zip` is never parsed; the watcher rejects the unknown extension before any content is read, so the bytes are immaterial):

```bash
$ printf 'zip payload\n' > /tmp/pngx-work/consume/archive.zip
```

**Log‑extraction command + complete, unedited output** (the watcher logged the rejection **twice** — once when the file was dropped, and again when the consumer was later restarted into polling mode for §4 and re‑scanned the still‑present file):

```bash
$ grep -F 'archive.zip' /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:52:19,449] [WARNING] [paperless.management.consumer] Not consuming file /tmp/pngx-work/consume/archive.zip: Unknown file extension.
[2026-07-08 07:52:32,825] [WARNING] [paperless.management.consumer] Not consuming file /tmp/pngx-work/consume/archive.zip: Unknown file extension.
```

This is the `is_file_ext_supported` gate `[src/documents/management/commands/document_consumer.py:54-55]` (predicate defined `[src/documents/parsers.py:62]`). Because the rejection happens in `_consume` **before** `async_task`, **no `django_q_task` row is created** for `archive.zip`.

**Verification query — no task row, task count unchanged** (producing commands + complete output, run immediately before and after the drop):

```bash
$ python manage.py shell -c "from django_q.models import Task; print('TASKS_BEFORE=%d' % Task.objects.count())"
TASKS_BEFORE=3
$ printf 'zip payload\n' > /tmp/pngx-work/consume/archive.zip     # (the drop shown above)
$ python manage.py shell -c "from django_q.models import Task; print('exists', Task.objects.filter(name='archive.zip').exists()); print('TASKS_AFTER=%d' % Task.objects.count())"
exists False
TASKS_AFTER=3
```

At the moment this condition ran, exactly three task rows existed — the two successful consumptions (`report_2022`, `report_2023`) plus the one failed `duplicate_of_2023`. Dropping `archive.zip` left that total **unchanged at 3** and produced **no** row named `archive.zip`, confirming the watcher‑level rejection never reaches the queue.

**(b) Consumer‑level (supported extension, unsupported content).** A file with a `.txt` extension but binary content (512 zero bytes → `magic` detects `application/octet-stream`, which has no parser) passes the extension filter, is queued, and fails at the MIME check.

**Input file — producing command + md5/size** (512 zero bytes; `magic` reports `application/octet-stream`, for which no parser is declared):

```bash
$ head -c 512 /dev/zero > /tmp/pngx-work/consume/unsupported.txt
$ md5sum /tmp/pngx-work/consume/unsupported.txt
bf619eac0cdf3f68d496ea9344137e8b  /tmp/pngx-work/consume/unsupported.txt
$ wc -c   /tmp/pngx-work/consume/unsupported.txt
512 /tmp/pngx-work/consume/unsupported.txt
```

**Log‑extraction command + complete, unedited output:**

```bash
$ awk '/Adding .*unsupported\.txt to the task queue/{f=1} f{print} /Unsupported mime type/{exit}' \
    /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:52:26,284] [INFO] [paperless.management.consumer] Adding /tmp/pngx-work/consume/unsupported.txt to the task queue.
[2026-07-08 07:52:26,418] [INFO] [paperless.consumer] Consuming unsupported.txt
[2026-07-08 07:52:26,420] [DEBUG] [paperless.consumer] Detected mime type: application/octet-stream
[2026-07-08 07:52:26,422] [ERROR] [paperless.consumer] Unsupported mime type application/octet-stream
```

Here `get_parser_class_for_mime_type` returns `None` `[src/documents/consumer.py:223]` → `_fail(MESSAGE_UNSUPPORTED_TYPE, ...)` `[:225]`, with constant `MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` `[:44]`. Because the file **was** queued (its `.txt` extension passed the watcher gate), the worker records a `success = False` row (producing command + complete output):

```bash
$ python manage.py shell -c "
from django_q.models import Task
t = Task.objects.filter(name='unsupported.txt', success=False).order_by('started').first()
print('id      =', t.id)
print('name    =', t.name)
print('func    =', t.func)
print('success =', t.success)
print('result  ='); print(t.result)"
id      = 529932eb97be4ddf8b030fd95f4462fb
name    = unsupported.txt
func    = documents.tasks.consume_file
success = False
result  =
unsupported.txt: Unsupported mime type application/octet-stream : Traceback (most recent call last):
  File "/opt/paperless-venv/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/consumer.py", line 225, in try_consume_file
    self._fail(MESSAGE_UNSUPPORTED_TYPE, f"Unsupported mime type {mime_type}")
  File "/tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: unsupported.txt: Unsupported mime type application/octet-stream
```

The traceback confirms the codepath differs from the duplicate case: it fails at `try_consume_file` `[src/documents/consumer.py:225]` (the `_fail(MESSAGE_UNSUPPORTED_TYPE, …)` call), not in `pre_check_duplicate`. This is the key contrast with the watcher‑level rejection in 3(a), which never reaches the worker at all.

### 4. Polling vs. inotify watcher modes

The default is **inotify** (`CONSUMER_POLLING == 0` `[src/documents/management/commands/document_consumer.py:178]`). To switch to **polling**, the consumer is stopped and relaunched with `PAPERLESS_CONSUMER_POLLING=2` — the exact restart commands (run from `$REPO/src`):

```bash
# stop the inotify consumer started earlier for R2/R3
$ kill "$CONSUMER_PID"
# relaunch the same management command with polling enabled (2-second interval)
$ export PAPERLESS_CONSUMER_POLLING=2
$ python manage.py document_consumer >> /tmp/pngx-work/log/consumer.stdout 2>&1 &
```

The two modes emit distinguishing startup lines:

```
# inotify (default) startup line:
[2026-07-08 07:51:42,910] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /tmp/pngx-work/consume    # :200
# polling (PAPERLESS_CONSUMER_POLLING=2) startup line:
[2026-07-08 07:52:32,825] [INFO] [paperless.management.consumer] Polling directory for changes: /tmp/pngx-work/consume                   # :186
```

Under polling, a distinct debounce line (`Waiting for file … to remain unmodified`) appears **before** queuing.

**Input file — producing command + md5/size** (a 99‑byte plaintext memo, distinct from the earlier files):

```bash
$ printf 'Paperless NGX polling trace test.\nDelta Corporation memo.\nDated 2025-07-04. Amount due 168.00 USD.\n' \
    > /tmp/pngx-work/consume/report_poll.txt
$ md5sum /tmp/pngx-work/consume/report_poll.txt
f40860d7e08bc811354e40fbeb719c38  /tmp/pngx-work/consume/report_poll.txt
$ wc -c   /tmp/pngx-work/consume/report_poll.txt
99 /tmp/pngx-work/consume/report_poll.txt
```

**Log‑extraction command + complete, unedited output** (no ellipsis) — the `Waiting` line at `07:52:36` precedes the `Adding` line at `07:52:41`, ≈5 s later, reflecting the `PAPERLESS_CONSUMER_POLLING=2` debounce:

```bash
$ awk '/Waiting for file .*report_poll\.txt to remain unmodified/{f=1} f{print} /report_poll consumption finished/{exit}' \
    /tmp/pngx-work/log/paperless.log
```

```
[2026-07-08 07:52:36,827] [DEBUG] [paperless.management.consumer] Waiting for file /tmp/pngx-work/consume/report_poll.txt to remain unmodified
[2026-07-08 07:52:41,833] [INFO] [paperless.management.consumer] Adding /tmp/pngx-work/consume/report_poll.txt to the task queue.
[2026-07-08 07:52:41,963] [INFO] [paperless.consumer] Consuming report_poll.txt
[2026-07-08 07:52:41,967] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-08 07:52:41,970] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-08 07:52:41,971] [DEBUG] [paperless.consumer] Parsing report_poll.txt...
[2026-07-08 07:52:41,972] [DEBUG] [paperless.consumer] Generating thumbnail for report_poll.txt...
[2026-07-08 07:52:41,993] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/pngx-work/scratch/paperless-r5wssogt/thumb.png -out /tmp/pngx-work/scratch/paperless-r5wssogt/thumb_optipng.png
[2026-07-08 07:52:43,909] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 07:52:43,911] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 07:52:43,928] [DEBUG] [paperless.consumer] Deleting file /tmp/pngx-work/consume/report_poll.txt
[2026-07-08 07:52:43,972] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/pngx-work/scratch/paperless-r5wssogt
[2026-07-08 07:52:43,973] [INFO] [paperless.consumer] Document 2026-07-08 report_poll consumption finished
```

The `Waiting` line is emitted by `_consume_wait_unmodified` `[src/documents/management/commands/document_consumer.py:99]` (`Waiting for file … to remain unmodified` `[:103]`), which loops `settings.CONSUMER_POLLING_RETRY_COUNT` `[:107]` times sleeping `settings.CONSUMER_POLLING_DELAY` `[:122]` between tries, and logs a timeout error `[:125]` if the file never settles. The processing order **after detection** is identical to inotify; only the detection mechanism and the debounce differ. (`report_poll.txt` md5 `f40860d7e08bc811354e40fbeb719c38` became `documents_document` id `3`.) Also observed: on restart into polling mode the watcher **re‑scans pre‑existing files** via the `handle()` initial scan `[:167-173]`, so `duplicate_of_2023.txt` and `unsupported.txt` (left behind because failed files are **not** deleted) are re‑detected — which is why the `django_q_task` failure counts above show each name twice.

### 5. Barcode‑split branch (code‑level; off by default)

When `PAPERLESS_CONSUMER_ENABLE_BARCODES` is set, `consume_file` `[src/documents/tasks.py:184]` takes the branch at `if settings.CONSUMER_ENABLE_BARCODES:` `[:195]`, calling `scan_file_for_separating_barcodes` `[:96]` (which uses `barcode_reader` `[:75]`); on a split it returns `"File successfully split"` `[:233]`. This flag is **off by default**, so the canonical `.txt` path skips this block. This branch is cited **from source** and was **not** exercised at runtime (it requires a PDF carrying a separator barcode plus the `pyzbar`/`zbar` tooling); it is labelled here as a documented secondary/edge path.


---

## Design Patterns and Accuracy Nuances

- **Observer pattern for detection (two distinct mechanisms).** The **polling** mode schedules a watchdog `PollingObserver` `[src/documents/management/commands/document_consumer.py:187]` bound to a `Handler()` `[:188]`; filesystem events dispatch to `Handler.on_created` `[:129]` / `Handler.on_moved` `[:132]`, each of which spawns a `Thread` targeting `_consume_wait_unmodified` `[:130/:133]` that debounces and then calls `_consume` `[:118]`. The **inotify** mode does **not** use the `Handler` class at all — `handle_inotify` `[:199]` runs its own debounce loop `[:211]` and calls the module‑level `_consume(filepath)` directly `[:230]` (full analysis in R6).
- **Signal‑based plugin registration for parsers.** Each parser app connects its declaration to `document_consumer_declaration` in its `apps.ready()` (e.g. `[src/paperless_text/apps.py:11-13]`); dispatch picks the **highest‑weight** declaration in `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]` by iterating `document_consumer_declaration.send(None)` `[:87]`. Observed weights: **Text = 10** `[src/paperless_text/signals.py:10]`, **Tika = 10** `[src/paperless_tika/signals.py:10]`, **Tesseract = 0** `[src/paperless_tesseract/signals.py:10]`.
- **Signed‑package task serialization.** The queued payload is a **pickled + Django‑signed** `dict` produced by `SignedPackage` `[installed: django_q/signing.py:10]`, with **salt = cluster name `paperless`** and **key = Django `SECRET_KEY`** `[installed: django_q/signing.py:15-21]`.
- **Synchronous post‑consume hooks.** Classification/matching/indexing run **inside** `consume_file` via the `document_consumption_finished` signal `[src/documents/apps.py:22-27]`, **not** as separate queued tasks.
- **`documents_log` defined‑but‑unwritten.** The `Log` model / `documents_log` table `[src/documents/models.py:285]` is defined but never populated by the v1.7.0 consumption path (no DB logging handler `[src/paperless/settings.py:373-412]`); `Log.objects.count()` observed to be `0`.
- **`.txt` still exercises the `optipng` binary (thumbnail optimisation).** The text parser's own `get_thumbnail` renders the PNG with **PIL only** — no external binary `[src/paperless_text/parsers.py:19-38]`. However, the consumer calls `get_optimised_thumbnail` `[src/documents/consumer.py:265]`, and the base `DocumentParser.get_optimised_thumbnail` `[src/documents/parsers.py:319-338]` then runs `settings.OPTIPNG_BINARY` `[src/documents/parsers.py:325]` via `subprocess.Popen(args).wait()` `[src/documents/parsers.py:335]` whenever `OPTIMIZE_THUMBNAILS` is true — the default `[src/paperless/settings.py:508]` — emitting `Execute: optipng -silent -o5 ...` `[src/documents/parsers.py:333]` (the exact line seen in R3). In **this** canonical run `optipng` was present (`/usr/bin/optipng`), so that step succeeded and the document row (id `1`) was written. Were `optipng` absent or non‑zero, `get_optimised_thumbnail` raises `ParseError("Optipng failed …")` `[src/documents/parsers.py:336]`, failing `consume_file` before `_store` writes the row — so a `.txt` is not entirely tool‑free despite routing to the lightest parser.

### Security & performance notes (observed / cited)

- Task packages are **HMAC‑signed** with the Django `SECRET_KEY` (salt = cluster name `paperless` `[src/paperless/settings.py:449-450]`), so only clusters sharing both can unpack a package.
- The default Redis broker provides **at‑most‑once** delivery without message receipts (enqueue `rpush` / dequeue `blpop` `[installed: django_q/brokers/redis_broker.py:17-23]`).
- The Django‑Q worker **timeout is 1800 seconds** `[src/paperless/settings.py:440,454]`.

---

## Reproduction & Cleanup Appendix

### Canonical service commands (from `docker/supervisord.conf`)

```bash
gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application   # web/ASGI   [docker/supervisord.conf:11]
python3 manage.py document_consumer                                           # watcher    [docker/supervisord.conf:20]
python3 manage.py qcluster                                                    # Django-Q   [docker/supervisord.conf:29]
```

### Full command list to reproduce the trace

```bash
# Runtime + deps (see R1 for the full list). Redis was already running (setup); verify:
redis-cli ping                                                               # PONG
export PAPERLESS_REDIS=redis://localhost:6379
export PAPERLESS_DATA_DIR=/tmp/pngx-work/data PAPERLESS_CONSUMPTION_DIR=/tmp/pngx-work/consume
export PAPERLESS_LOGGING_DIR=/tmp/pngx-work/log PAPERLESS_SCRATCH_DIR=/tmp/pngx-work/scratch
source /opt/paperless-venv/bin/activate
cd /tmp/blitzy/paperless-ngx/blitzy-02766774-d432-4287-a066-d2b57627175a_b181c9/src
python manage.py migrate                                                     # exit 0

# Observe the broker BEFORE
redis-cli LLEN django_q:paperless:q                                          # 0

# Start the watcher, drop the file (real entry point)
python manage.py document_consumer &
printf "Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n" > /tmp/pngx-work/consume/report_2022.txt
md5sum /tmp/pngx-work/consume/report_2022.txt                                # 0da8a96bf7f3a377f2acba04d2800b51

# Observe the broker DURING (task queued, worker not yet running)
redis-cli LLEN django_q:paperless:q                                          # 1
redis-cli --no-raw LINDEX django_q:paperless:q 0                             # complete 447-byte value shown in R4

# Decode the queued package
python manage.py shell    # from django_q.signing import SignedPackage; SignedPackage.loads(raw)  (full output in R4)

# Start the worker; observe the broker AFTER
python manage.py qcluster &
redis-cli LLEN django_q:paperless:q                                          # 0

# Query the database after processing (full outputs in R5)
python manage.py shell    # Document.objects.get(id=1); Task.objects.get(name='report_2022.txt'); LogEntry.objects.all(); Log.objects.count()  # -> 0
ls /tmp/pngx-work/data/index/                                                # Whoosh index files
```

### Cleanup / verification performed

The investigation ran the pinned dependencies from a virtual environment **outside** the repository (`/opt/paperless-venv`), and **all** runtime data (`PAPERLESS_DATA_DIR`, `PAPERLESS_CONSUMPTION_DIR`, `PAPERLESS_LOGGING_DIR`, `PAPERLESS_SCRATCH_DIR`, `PAPERLESS_MEDIA_ROOT`) was directed to scratch under `/tmp/pngx-work`, also outside the repository — so no SQLite database, Whoosh index, media, or log file was ever written into the source tree. All temporary observation scripts and test files were removed afterward. **`git status --porcelain` on the source repository returns empty** apart from this one new markdown document — the source tree is byte‑for‑byte unchanged.

> The behaviours reported above are **stable and reproducible**: an independent re‑run of the exact `.txt` ingestion path (isolated data/broker outside the repo) reproduced the same ordered log stream, the same broker list key `django_q:paperless:q` with the identical `empty → 1 → drained` transition, the same decoded task‑dict shape (`func = documents.tasks.consume_file`, `kwargs = {'override_tag_ids': None}`, salt `paperless`), the same `checksum == md5` relationship, the same `django_q_task` result string form (`Success. New document id N created`), `Log.objects.count() == 0`, and the same Whoosh index file set — with only the inherently variable values (timestamps, absolute paths, scratch‑dir hashes, task/document ids) differing between runs.

---

## Coverage Pass — every part of the question, answered

| # | Requirement | Where answered | Key observed value(s) |
|---|-------------|----------------|-----------------------|
| **R1** | Bring up the full runtime (Redis, DB, qcluster, document_consumer, gunicorn) | R1 | `redis-cli ping → PONG`; `migrate` exit 0; seeded `django_q_schedule`; `consumer` user `(1,'consumer')`; baseline counts 0 |
| **R2** | Drop a test file into the consumption dir and follow it (real entry point) | R2 | `report_2022.txt`, 100 bytes, md5 `0da8a96bf7f3a377f2acba04d2800b51`; routes to `TextDocumentParser` (weight 10) |
| **R3** | Enumerate log messages + triggered tasks | R3 | full ordered stream (12 lines) verbatim; ingestion task `documents.tasks.consume_file` `[src/documents/tasks.py:184]`; inline 6 handlers vs separate scheduled tasks |
| **R4** | Inspect the queued task on the broker | R4 | list `django_q:paperless:q`; `LLEN` 0→1→0; 447‑byte signed pickle; decoded `dict` `{args,func,id,kwargs,name,started}`; salt `paperless` |
| **R5** | Query the DB after processing (record, parsed fields, task history) | R5 | `documents_document` row (`checksum == md5`); `django_q_task` `Success. New document id 1 created`; `django_admin_log` ADDITION by `consumer`; `documents_log` count `0`; Whoosh files |
| **R6** | Trace the codepath + name the framework | R6 | `document_consumer._consume` → `async_task` (builds+signs+enqueues) → `qcluster` runs `consume_file`; framework = **Django‑Q + Redis**, not Celery |
| **S1** | Stable ordering ≥2 runs | Secondary §1 | byte‑for‑byte identical ordering (id 1 and id 2) |
| **S2** | Duplicate file | Secondary §2 | `Not consuming ...: It is a duplicate.`; task `success = False` |
| **S3** | Unsupported MIME (2 paths) | Secondary §3 | watcher‑level `Unknown file extension.` (no task); consumer‑level `Unsupported mime type application/octet-stream` |
| **S4** | Polling vs inotify | Secondary §4 | `Using inotify ...` (default) vs `Polling directory ...` + `Waiting for file ... to remain unmodified` |
| **S5** | Barcode‑split branch | Secondary §5 | cited from source (`src/documents/tasks.py:195`, `:96`, `:233`); off by default, not run |

**Named mechanisms/functions addressed:** `document_consumer` command, `Handler.on_created/on_moved`, `_consume`, `is_file_ext_supported`, `async_task`, `SignedPackage.dumps/loads`, `PickleSerializer`, Redis `rpush`/`blpop`, `qcluster`, `consume_file`, `Consumer.try_consume_file`, `pre_check_duplicate`, `get_parser_class_for_mime_type`, `TextDocumentParser`, `document_consumption_finished` and its six handlers (`add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index`), `match_correspondents/document_types/tags`, `index.add_or_update_document`, `LoggingMixin.log`, `Document` / `Log` models, the `Task` model, and the scheduled tasks `train_classifier` / `index_optimize` / `sanity_check`.
