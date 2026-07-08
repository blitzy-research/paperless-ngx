# How Paperless‑NGX Processes a Document End‑to‑End — A Runtime‑Grounded Trace

*From a file landing in the consumption directory, through parsing, classification, and full‑text indexing — answered with real observed evidence (captured log lines, real Redis broker contents, real database rows) plus exact `file:line` code citations.*

---

## Environment

All values below were produced by **building and running Paperless‑NGX in its default, canonical configuration** as a normal user would, and then observing the live system. Every behavioural claim is backed by (a) the actual command that produced it, (b) the actual output, and (c) an exact `file:line` citation into the source tree.

| Property | Value | Evidence |
|----------|-------|----------|
| Application | **Paperless‑NGX v1.7.0** | `[src/paperless/version.py:1]` → `__version__ = (1, 7, 0)` |
| Runtime | **Python 3.9** (container base `python:3.9-slim-bullseye`; observed interpreter `Python 3.9.23`) | matches the project `Dockerfile` base |
| Database | **SQLite** at `DATA_DIR/db.sqlite3` (default) | `[src/paperless/settings.py:299-300]`; PostgreSQL only if `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:304-311]` |
| Message broker + channel layer | **Redis** | Q_CLUSTER broker `[src/paperless/settings.py:449-456]`; Channels layer `[src/paperless/settings.py:178-182]` |
| Queuing framework | **Django‑Q `1.3.9`** — cluster name `paperless`, worker timeout `1800`s | `[requirements.txt:37]`, `[src/paperless/settings.py:449-454]`; registered in `INSTALLED_APPS` as `"django_q"` `[src/paperless/settings.py:110]` |
| Full‑text search | **Whoosh `2.7.4`** | `[requirements.txt:111]`, `[src/documents/index.py:31]` |
| Parser used for the test file | **`TextDocumentParser`** (plain text, weight 10) | `[src/paperless_text/parsers.py:12]`, `[src/paperless_text/signals.py:10-12]` |

> **Read‑only investigation.** The source repository is treated as strictly read‑only. The **only** artifact created by this task is this markdown document. During observation, all runtime data (database, media, consumption directory, logs, scratch) was relocated to scratch directories **outside** the source tree via `PAPERLESS_DATA_DIR`, `PAPERLESS_MEDIA_ROOT`, `PAPERLESS_CONSUMPTION_DIR`, `PAPERLESS_LOGGING_DIR`, and `PAPERLESS_SCRATCH_DIR` (→ `/work/...`), the repo was mounted read‑only, and `PYTHONDONTWRITEBYTECODE=1` was set. `git status --porcelain` on the source repository returns **empty** (see the Reproduction & Cleanup appendix).

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
# 1. Canonical Python 3.9 runtime (matches Dockerfile base FROM python:3.9-slim-bullseye)
docker run -d --name pngx -v <repo>:/src:ro -e PYTHONDONTWRITEBYTECODE=1 -e PYTHONUNBUFFERED=1 \
  python:3.9-slim-bullseye sleep infinity           # observed: Python 3.9.23

# 2. System deps (normal-user prerequisites)
apt-get install -y libmagic1 redis-server fonts-liberation libzbar0 tesseract-ocr gnupg optipng

# 3. Python deps (pinned)
pip install -r requirements.txt      # django==4.0.4, django-q==1.3.9, redis==3.5.3, channels==3.0.4,
                                      # channels-redis==3.4.0, whoosh==2.7.4, watchdog==2.1.7,
                                      # inotifyrecursive==0.3.5, python-magic==0.4.25, concurrent-log-handler==0.9.20, ...

# 4. Canonical environment (data relocated OUT of the source tree to keep it pristine)
export PAPERLESS_REDIS=redis://localhost:6379
export PAPERLESS_DATA_DIR=/work/data
export PAPERLESS_MEDIA_ROOT=/work/media
export PAPERLESS_CONSUMPTION_DIR=/work/consume
export PAPERLESS_LOGGING_DIR=/work/log
export PAPERLESS_SCRATCH_DIR=/work/scratch
export PAPERLESS_THUMBNAIL_FONT_NAME=/usr/share/fonts/truetype/liberation/LiberationSerif-Regular.ttf
export PAPERLESS_TIME_ZONE=UTC

# 5. Start Redis
redis-server --daemonize yes --save "" --appendonly no
redis-cli ping                                       # observed: PONG

# 6. Apply migrations (creates all backing tables + seeds scheduled tasks)
cd /src/src && python3 manage.py migrate              # observed: exit 0, all migrations applied

# 7. Launch the canonical processes (as in docker/supervisord.conf)
python3 manage.py document_consumer                   # watcher
python3 manage.py qcluster                            # Django-Q worker/scheduler
# gunicorn -c gunicorn.conf.py paperless.asgi:application   # web (not required to observe consumption)
```

### Observed results

- `redis-cli ping` → `PONG`.
- `manage.py migrate` → **exit 0**; applied migrations for `admin / auth / authtoken / contenttypes / django_q / documents[0001..1018] / paperless_mail / sessions`. Only **non‑fatal WARNINGS** appeared — `Paperless can't find convert` and `optipng` — emitted by `binaries_check` `[src/paperless/checks.py]` at Warning level.
- `migrate` created the backing tables **`documents_document`**, **`documents_log`**, **`django_q_task`**, **`django_q_schedule`**, **`django_admin_log`**, and **`auth_user`** (verified present in `/work/data/db.sqlite3`).
- `migrate` **seeded the scheduled Django‑Q tasks** — rows in **`django_q_schedule`** `(func, name, schedule_type)`:

```
('documents.tasks.train_classifier',            'Train the classifier',        'H')   # HOURLY
('documents.tasks.index_optimize',              'Optimize the index',          'D')   # DAILY
('documents.tasks.sanity_check',                'Perform sanity check',        'W')   # WEEKLY
('paperless_mail.tasks.process_mail_accounts',  'Check all e-mail accounts',   'I')   # minutes (mail app)
```

  These are seeded by migrations `[src/documents/migrations/1001_auto_20201109_1636.py]` (train_classifier=HOURLY, index_optimize=DAILY) and `[src/documents/migrations/1004_sanity_check_schedule.py]` (sanity_check=WEEKLY).
- Migration `[src/documents/migrations/0019_add_consumer_user.py:10]` created the **`consumer`** user (row `(1, 'consumer')`) via `User.objects.create(username="consumer")` — this is the user that `set_log_entry` later attributes the admin audit entry to.
- **Baseline before any file:** `documents_document` count `0`, `django_q_task` count `0`, `documents_log` count `0`, broker `LLEN django_q:paperless:q` = `0`.

> **Runtime‑prerequisite note (accuracy).** Even a plain `.txt` requires the **`optipng`** binary: the text parser's thumbnail generation runs `optipng -silent -o5 ...`. Without `optipng`, the `consume_file` task **fails before the document row is written** — this was observed directly: an initial run failed with `[Errno 2] No such file or directory: 'optipng'` and left `documents_document` at `0`. `optipng` is part of the canonical binary set checked by `binaries_check` `[src/paperless/checks.py]`.

---

## R2 — Drop a Test File and Follow It (the real entry point)

**Answer.** The document is ingested through the **real entry point** — writing a file into `PAPERLESS_CONSUMPTION_DIR` — **not** the REST upload endpoint and **not** a direct `async_task` call. A plain‑text file routes to the lightest parser, **`TextDocumentParser`** in the `paperless_text` app, which declares `text/plain` → `.txt` with **weight 10** `[src/paperless_text/signals.py:10-12]`. That declaration is registered by connecting to the `document_consumer_declaration` signal in the app's `ready()` `[src/paperless_text/apps.py:11-13]`, and parser dispatch picks the **highest‑weight** declaration in `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]` by iterating `document_consumer_declaration.send(None)` `[src/documents/parsers.py:87]`.

### Command + file used

```bash
printf "Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n" \
  > /work/consume/report_2022.txt
md5sum /work/consume/report_2022.txt      # 0da8a96bf7f3a377f2acba04d2800b51
wc -c   /work/consume/report_2022.txt      # 100
```

**Observed:** the file is **100 bytes** with md5 **`0da8a96bf7f3a377f2acba04d2800b51`**. This exact md5 reappears verbatim as the database **`checksum`** field in R5 — the document's checksum is the md5 hexdigest of the ingested file's bytes `[src/documents/consumer.py:103-104]`.

---

## R3 — Log Messages and Triggered Tasks

**Answer.** All pipeline loggers live under the `paperless.*` namespace. Per the `LOGGING` configuration `[src/paperless/settings.py:373-412]`, the `paperless` logger writes to a rotating file handler (`paperless.log`) `[src/paperless/settings.py:392-395,409]` and propagates to the root `console` `StreamHandler` `[src/paperless/settings.py:387-390,407]`, so every line below appears **both** on stdout and in `paperless.log`. The `verbose` formatter is `[{asctime}] [{levelname}] [{name}] {message}` `[src/paperless/settings.py:377-379]`.

### Ordered log stream — captured verbatim from `/work/log/paperless.log`

```
[2026-07-08 03:23:33,230] [INFO] [paperless.management.consumer] Adding /work/consume/report_2022.txt to the task queue.
[2026-07-08 03:23:33,371] [INFO] [paperless.consumer] Consuming report_2022.txt
[2026-07-08 03:23:33,374] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-08 03:23:33,378] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-08 03:23:33,381] [DEBUG] [paperless.consumer] Parsing report_2022.txt...
[2026-07-08 03:23:33,381] [DEBUG] [paperless.consumer] Generating thumbnail for report_2022.txt...
[2026-07-08 03:23:33,405] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /work/scratch/paperless-t8ccitgq/thumb.png -out /work/scratch/paperless-t8ccitgq/thumb_optipng.png
[2026-07-08 03:23:35,749] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 03:23:35,752] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 03:23:35,769] [DEBUG] [paperless.consumer] Deleting file /work/consume/report_2022.txt
[2026-07-08 03:23:35,794] [DEBUG] [paperless.parsing.text] Deleting directory /work/scratch/paperless-t8ccitgq
[2026-07-08 03:23:35,794] [INFO] [paperless.consumer] Document 2026-07-08 report_2022 consumption finished
```

### Per‑line `file:line` mapping

| Log line | Logger | Source |
|----------|--------|--------|
| `Adding {filepath} to the task queue.` | `paperless.management.consumer` | `[src/documents/management/commands/document_consumer.py:85]` (logger defined `[:24]`) |
| `Consuming {filename}` | `paperless.consumer` | `[src/documents/consumer.py:215]` (`logging_name` set `[:54]`) |
| `Detected mime type: {mime}` | `paperless.consumer` | `[src/documents/consumer.py:221]` (`magic.from_file(...)` `[:219]`) |
| `Parser: {ParserClass}` | `paperless.consumer` | `[src/documents/consumer.py:246]` (parser selected via `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]`) |
| `Parsing {}...` | `paperless.consumer` | `[src/documents/consumer.py:260]` |
| `Generating thumbnail for {}...` | `paperless.consumer` | `[src/documents/consumer.py:263]` |
| `Execute: optipng ...` | `paperless.parsing.text` | text parser via base `DocumentParser.run_command` `[src/documents/parsers.py:333]`; parser `logging_name` `[src/paperless_text/parsers.py:17]` |
| `Document classification model does not exist (yet), not performing automatic matching.` | `paperless.classifier` | `[src/documents/classifier.py:33]` — on a fresh install `load_classifier()` `[:30]` returns `None` before importing scikit‑learn |
| `Saving record to database` | `paperless.consumer` | `[src/documents/consumer.py:387]` — immediately before `Document.objects.create(...)` `[:398]` |
| `Deleting file {}` | `paperless.consumer` | `[src/documents/consumer.py]` (source file removed after storage) |
| `Deleting directory {}` | `paperless.parsing.text` | base `DocumentParser` cleanup `[src/documents/parsers.py:349]` |
| `Document {} consumption finished` | `paperless.consumer` | `[src/documents/consumer.py:373]` — `{}` is the document's `__str__` = `"{created} {title}"` (no correspondent) `[src/documents/models.py:212-219]`, hence `2026-07-08 report_2022` |

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

- `documents.tasks.train_classifier` — HOURLY `[src/documents/tasks.py:48]`, seeded `[src/documents/migrations/1001_auto_20201109_1636.py]`
- `documents.tasks.index_optimize` — DAILY `[src/documents/tasks.py:32]`, seeded `[src/documents/migrations/1001_auto_20201109_1636.py]`
- `documents.tasks.sanity_check` — WEEKLY `[src/documents/tasks.py:255]`, seeded `[src/documents/migrations/1004_sanity_check_schedule.py]`

Observed after `qcluster` startup: `train_classifier` → success; `index_optimize` → success; `sanity_check` → success with result `No issues detected.`; and `paperless_mail.tasks.process_mail_accounts` → success. These are **not** part of the inline consumption pipeline.


---

## R4 — Inspect the Queued Task on the Broker (transitional states)

**Answer.** The queued task lives on a **Redis list** named **`django_q:paperless:q`**. Django‑Q constructs this key as `f"django_q:{PREFIX}:q"` where `PREFIX` is the cluster name **`paperless`** `[src/paperless/settings.py:449-450]`. Confirmed against the installed framework source `[django_q/brokers/redis_broker.py:14-15]`:

```python
def __init__(self, list_key: str = Conf.PREFIX):
    super(Redis, self).__init__(list_key=f"django_q:{list_key}:q")
```

Enqueue is a Redis **`rpush`** `[django_q/brokers/redis_broker.py:17-18]` and dequeue is a blocking **`blpop`** `[django_q/brokers/redis_broker.py:20-23]`.

### Transitional states (before / during / after)

```bash
# BEFORE (nothing queued)
redis-cli KEYS "*"                       # (empty)
redis-cli LLEN django_q:paperless:q      # 0

# DURING (document_consumer up, qcluster NOT yet started)
redis-cli KEYS "*"                       # "django_q:paperless:q"   (the only key)
redis-cli LLEN django_q:paperless:q      # 1

# AFTER (qcluster started -> worker blpop drains the list)
redis-cli LLEN django_q:paperless:q      # 0
```

The broker list therefore transitions **empty → one signed package → drained**.

### Raw queued payload (observed)

The queued entry is a **441‑byte ASCII string** in the form `<base64(pickle)>:<base62‑timestamp>:<HMAC‑signature>` produced by Django's `TimestampSigner`. Head of the raw value (via `redis-cli --no-raw LINDEX django_q:paperless:q 0`):

```
gAWVGQEAAAAAAAB9lCiMAmlklIwgODllMmY5YjQzNGY4NDk2ZTk5ZmY4YzhkOTc1YThkNTGUjARuYW1llIwRdGVzdF9kb2N1bWVudC50eHSUjARmdW5jlIwc...:1whIpz:w...
```

The `g` prefix is base64 of the pickle opcode `\x80\x04` (pickle `HIGHEST_PROTOCOL` = protocol 4 on Python 3.9). It is **not** zlib‑compressed: Django‑Q's `Conf.COMPRESSED` defaults to `False` `[django_q/conf.py:116]` and Paperless does not enable compression.

### Decoded package (observed)

Decoding in Django context via `django_q.signing.SignedPackage.loads(raw)`:

```
Conf.PREFIX (salt) = 'paperless'      # == cluster name  [src/paperless/settings.py:450]
Conf.SECRET_KEY    = <set, len 50>    # HMAC key = Django SECRET_KEY
type(decoded)      = dict
decoded.keys()     = ['args', 'func', 'id', 'kwargs', 'name', 'started']
decoded['func']    = 'documents.tasks.consume_file'
decoded['args']    = ('/work/consume/test_document.txt',)
decoded['kwargs']  = {'override_tag_ids': None}
decoded['name']    = 'test_document.txt'
decoded['id']      = '89e2f9b434f8496e99ff8c8d975a8d51'
```

> **Note on the file name in this capture.** This broker‑introspection snapshot was taken during a run that used a file named `test_document.txt` (hence the decoded `args`/`name` show that file and the id `89e2f9b434f8496e99ff8c8d975a8d51`). It is a self‑consistent capture and is distinct from the end‑to‑end database trace in R2/R5, which used `report_2022.txt`. The **structure** — a signed `dict` with keys `args/func/id/kwargs/name/started` and `func == documents.tasks.consume_file` — is identical regardless of which file is dropped.

### Commands used

```bash
redis-cli LLEN django_q:paperless:q
redis-cli --no-raw LINDEX django_q:paperless:q 0
# decode:
python3 manage.py shell   # then: from django_q.signing import SignedPackage; SignedPackage.loads(raw_bytes)
```

**Conclusion.** The broker entry is a **pickled‑then‑HMAC‑signed task dictionary** produced by Django‑Q's `SignedPackage.dumps` `[django_q/signing.py:14]`, which pickles the dict with `pickle.dumps(obj, protocol=pickle.HIGHEST_PROTOCOL)` (via `PickleSerializer` `[django_q/signing.py:35]`) and then signs it with `django_q.core_signing.dumps(..., key=Conf.SECRET_KEY, salt=Conf.PREFIX)` `[django_q/signing.py:14-19]` (the module aliases `from django_q import core_signing as signing` `[django_q/signing.py:4]`). Only a cluster sharing the same `SECRET_KEY` **and** cluster name (`paperless`) can unpack it. The dict's keys exactly match the task package built by `async_task` (see R6).

---

## R5 — Query the Database After Processing

Queries were run via the Django ORM (`manage.py shell`). The document record, the parsed‑metadata fields, and the task‑execution history are shown below with the exact observed values.

### (a) The document record — table `documents_document` `[src/documents/models.py:88]`

```
id         = 1
title      = 'report_2022'
content    = 'Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n'
mime_type  = 'text/plain'
checksum   = '0da8a96bf7f3a377f2acba04d2800b51'     # == md5 of the dropped file (see R2)
created    = 2026-07-08 03:23:32.228026+00:00
modified   = 2026-07-08 03:23:35.769228+00:00
storage_type = 'unencrypted'
```

These parsed‑metadata fields are written by `Document.objects.create(...)` in `[src/documents/consumer.py:398]`. `content` is the full parsed text produced by `TextDocumentParser`; `checksum` is the md5 hexdigest of the file bytes `[src/documents/consumer.py:103-104]`; `title` defaults to the filename stem.

### (b) Task‑execution history — table `django_q_task` (Django‑Q `Task` model `[django_q/models.py:20]`)

```
name    = 'report_2022.txt'
func    = 'documents.tasks.consume_file'
success = True
started = 2026-07-08 03:23:33.231577+00:00
stopped = 2026-07-08 03:23:35.797238+00:00
result  = 'Success. New document id 1 created'
id      = '316b3c718f114c118f2c22c0c5139122'
```

Django‑Q's result monitor persists **both** successes and failures here. The `result` string `Success. New document id {} created` is produced by `consume_file` `[src/documents/tasks.py:247]`. Failed runs (e.g. the duplicate/unsupported cases below, or the pre‑`optipng` failure) are stored with `success = False` and the traceback/error text as `result` (demonstrated in the Secondary Conditions section).

### (c) Admin audit entry — table `django_admin_log`, written by `set_log_entry` `[src/documents/signals/handlers.py:413]`

```
action_flag  = 1            # ADDITION
user         = 'consumer'   # the user auto-created by migration 0019
object_id    = 1
object_repr  = '2026-07-08 report_2022'
content_type = 'documents | document'
```

### (d) Accuracy nuance — `documents_log` is defined but NOT written by the consumption path

The `documents_log` table / `Log` model `[src/documents/models.py:285]` is **defined but not written** by the v1.7.0 consumption path. Observed: `Log.objects.count()` = **`0`** after successful consumption. The reason is that the `LOGGING` configuration `[src/paperless/settings.py:373-412]` has **no database handler** — only a console `StreamHandler` `[:387-390]`, a `ConcurrentRotatingFileHandler` writing `paperless.log` `[:392-395]`, and a mail file handler `[:400-402]`. Processing log lines therefore flow to **stdout and `paperless.log`** (via the `LoggingMixin.log` group‑UUID convention `[src/documents/loggers.py:14]`, which attaches `extra={"group": self.logging_group}` `[:21]`), never to `documents_log`. This is reported as **observed**, not assumed.

### (e) Full‑text index (ties R5 to indexing)

After consumption, `add_to_index` `[src/documents/signals/handlers.py:428]` → `index.add_or_update_document` `[src/documents/index.py:118]` (schema `[src/documents/index.py:31]`) created Whoosh index files under `/work/data/index/`:

```
MAIN_2kz6tag92ezi8xcv.seg    _MAIN_2.toc    MAIN_WRITELOCK
```


---

## R6 — Trace the Codepath and Name the Framework

**Codepath (each step named with its `file:line`):**

**1. The `document_consumer` management command** `[src/documents/management/commands/document_consumer.py]` watches `PAPERLESS_CONSUMPTION_DIR`. Its `handle()` `[:156]` chooses the watcher:

- `if settings.CONSUMER_POLLING == 0 and INotify:` `[:178]` → **inotify** via `handle_inotify` `[:199]`, which logs `Using inotify to watch directory for changes:` `[:200]` and watches with flags `CLOSE_WRITE | MOVED_TO` `[:203]`;
- otherwise **polling** via `handle_polling` `[:185]`, which logs `Polling directory for changes:` `[:186]` and uses a watchdog `PollingObserver` `[:187]`.

Filesystem events dispatch to `Handler.on_created` `[:129]` / `Handler.on_moved` `[:132]`.

**2. Those handlers call the module‑level `_consume(filepath)`** `[:46]`, which:

- skips directories/ignored paths `[:47]`;
- guards against moved files `[:50-52]`;
- applies the **extension pre‑filter** `if not is_file_ext_supported(...)` `[:54]` (from `[src/documents/parsers.py:62]`), logging `Not consuming file {}: Unknown file extension.` `[:55]` and returning if the extension is unsupported;
- then logs **`Adding {filepath} to the task queue.`** `[:85]` and **submits the task**:

```python
async_task(
    "documents.tasks.consume_file",
    filepath,
    override_tag_ids=tag_ids if tag_ids else None,
    task_name=os.path.basename(filepath)[:100],
)   # src/documents/management/commands/document_consumer.py:86-91
```

**3. `async_task`** (Django‑Q, `[django_q/tasks.py:20]`) is the function that **constructs the task object**. In the installed `django-q==1.3.9`, it:

- builds the task `dict` `{"id": ..., "name": ..., "func": func, "args": args}` `[django_q/tasks.py:40-46]`;
- sets `task["kwargs"] = keywords` `[django_q/tasks.py:64]` and `task["started"] = timezone.now()` `[django_q/tasks.py:65]`;
- signs it with `pack = SignedPackage.dumps(task)` `[django_q/tasks.py:69]`;
- enqueues it via `enqueue_id = broker.enqueue(pack)` `[django_q/tasks.py:73]` and logs `Enqueued {enqueue_id}` `[django_q/tasks.py:74]`.

This exactly explains the decoded broker dict in R4 (keys `args / func / id / kwargs / name / started`).

**4. The `qcluster` worker** pops the package (`blpop` `[django_q/brokers/redis_broker.py:20-23]`), verifies the HMAC signature and unpickles it, then runs **`documents.tasks.consume_file`** `[src/documents/tasks.py:184]`, which constructs `Consumer().try_consume_file(...)` `[src/documents/consumer.py:180]` — the rest of the pipeline traced in R3.

### The precise answer to "which part constructs the task, and which framework dispatches it"

- **What constructs the task object:** Paperless's `_consume` calls `async_task(...)` `[src/documents/management/commands/document_consumer.py:86-91]`; the **`async_task` function itself builds the task `dict`** `[django_q/tasks.py:40-46]`, then signs `[django_q/tasks.py:69]` and enqueues `[django_q/tasks.py:73]` it.
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

A second happy file `report_2023.txt` (md5 `882152fa1db0eb45890bfdc43ad4fb9b`, which became `documents_document` id `2`) produced a log sequence **byte‑for‑byte identical** to the first run — only the filename, timestamps, and the scratch‑dir hash differ. Ordering is stable.

```
[2026-07-08 03:28:31,727] [INFO] [paperless.management.consumer] Adding /work/consume/report_2023.txt to the task queue.
[2026-07-08 03:28:31,868] [INFO] [paperless.consumer] Consuming report_2023.txt
[2026-07-08 03:28:31,872] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-08 03:28:31,875] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-08 03:28:31,877] [DEBUG] [paperless.consumer] Parsing report_2023.txt...
[2026-07-08 03:28:31,877] [DEBUG] [paperless.consumer] Generating thumbnail for report_2023.txt...
[2026-07-08 03:28:31,900] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /work/scratch/paperless-niuj5945/thumb.png -out /work/scratch/paperless-niuj5945/thumb_optipng.png
[2026-07-08 03:28:34,288] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 03:28:34,291] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 03:28:34,310] [DEBUG] [paperless.consumer] Deleting file /work/consume/report_2023.txt
[2026-07-08 03:28:34,334] [DEBUG] [paperless.parsing.text] Deleting directory /work/scratch/paperless-niuj5945
[2026-07-08 03:28:34,334] [INFO] [paperless.consumer] Document 2026-07-08 report_2023 consumption finished
```

### 2. Duplicate file (pre‑existing checksum)

Re‑creating the exact content of an already‑consumed document (same md5 `882152fa1db0eb45890bfdc43ad4fb9b`) triggers `pre_check_duplicate` `[src/documents/consumer.py:213]` (definition `[:102]`), which runs **before** the `Consuming` log `[:215]`:

```
[2026-07-08 03:28:53,451] [INFO]  [paperless.management.consumer] Adding /work/consume/duplicate_of_2023.txt to the task queue.
[2026-07-08 03:28:53,597] [ERROR] [paperless.consumer] Not consuming duplicate_of_2023.txt: It is a duplicate.
```

The worker records the task as failed:

```
documents.consumer.ConsumerError: duplicate_of_2023.txt: Not consuming duplicate_of_2023.txt: It is a duplicate.
```

The message constant is `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` `[src/documents/consumer.py:37]`; `_fail(...)` `[:78]` raises `ConsumerError`, which Django‑Q records as `success = False` in `django_q_task`. Note that **no `Consuming`/mime lines appear** because the duplicate check precedes them.

### 3. Unsupported MIME — two distinct rejection paths

**(a) Watcher‑level (unknown extension).** Dropping `archive.zip` is rejected **before queuing** — no task is created:

```
[2026-07-08 03:29:23,077] [WARNING] [paperless.management.consumer] Not consuming file /work/consume/archive.zip: Unknown file extension.
```

This is the `is_file_ext_supported` gate `[src/documents/management/commands/document_consumer.py:54-55]` (predicate defined `[src/documents/parsers.py:62]`).

**(b) Consumer‑level (supported extension, unsupported content).** A file with a `.txt` extension but binary content (512 zero bytes → `magic` detects `application/octet-stream`, which has no parser) passes the extension filter, is queued, and fails at the MIME check:

```
[2026-07-08 03:30:32,596] [INFO]  [paperless.management.consumer] Adding /work/consume/unsupported.txt to the task queue.
[2026-07-08 03:30:32,739] [INFO]  [paperless.consumer] Consuming unsupported.txt
[2026-07-08 03:30:32,740] [DEBUG] [paperless.consumer] Detected mime type: application/octet-stream
[2026-07-08 03:30:32,742] [ERROR] [paperless.consumer] Unsupported mime type application/octet-stream
```

Here `get_parser_class_for_mime_type` returns `None` `[src/documents/consumer.py:223]` → `_fail(MESSAGE_UNSUPPORTED_TYPE, ...)` `[:225]`, with constant `MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` `[:44]`. This too is recorded as `success = False` in `django_q_task`, with the traceback stored as the `result`.

### 4. Polling vs. inotify watcher modes

The default is **inotify** (`CONSUMER_POLLING == 0` `[src/documents/management/commands/document_consumer.py:178]`). Restarting the consumer with `PAPERLESS_CONSUMER_POLLING=2` switches to **polling**. The distinguishing startup lines:

```
# inotify (default):
[INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /work/consume      # :200
# polling (PAPERLESS_CONSUMER_POLLING=2):
[INFO] [paperless.management.consumer] Polling directory for changes: /work/consume                      # :186
```

Under polling, a distinct debounce line appears before queuing:

```
[2026-07-08 03:31:35,777] [DEBUG] [paperless.management.consumer] Waiting for file /work/consume/report_poll.txt to remain unmodified
[2026-07-08 03:31:40,778] [INFO]  [paperless.management.consumer] Adding /work/consume/report_poll.txt to the task queue.
...
[2026-07-08 03:31:43,326] [INFO]  [paperless.consumer] Document 2026-07-08 report_poll consumption finished
```

This is `_consume_wait_unmodified` `[src/documents/management/commands/document_consumer.py:99]`, which logs `Waiting for file ... to remain unmodified` `[:103]`, loops `settings.CONSUMER_POLLING_RETRY_COUNT` `[:107]` times sleeping `settings.CONSUMER_POLLING_DELAY` `[:122]` between tries, and logs a timeout error `[:125]` if the file never settles. The processing order **after detection** is identical to inotify; only the detection mechanism and the debounce differ. (`report_poll.txt` md5 `2c617f0ca4bd3f6da3e3ca80918b56a8` became id `3`.) Also observed: on startup the watcher **re‑scans pre‑existing files** in the directory, so files left behind by earlier failures (which are not deleted) are re‑queued or re‑warned.

### 5. Barcode‑split branch (code‑level; off by default)

When `PAPERLESS_CONSUMER_ENABLE_BARCODES` is set, `consume_file` `[src/documents/tasks.py:184]` takes the branch at `if settings.CONSUMER_ENABLE_BARCODES:` `[:195]`, calling `scan_file_for_separating_barcodes` `[:96]` (which uses `barcode_reader` `[:75]`); on a split it returns `"File successfully split"` `[:233]`. This flag is **off by default**, so the canonical `.txt` path skips this block. This branch is cited **from source** and was **not** exercised at runtime (it requires a PDF carrying a separator barcode plus the `pyzbar`/`zbar` tooling); it is labelled here as a documented secondary/edge path.


---

## Design Patterns and Accuracy Nuances

- **Observer pattern for detection.** A watchdog `PollingObserver` `[src/documents/management/commands/document_consumer.py:187]` or an inotify observer dispatches filesystem events to `Handler.on_created` `[:129]` / `Handler.on_moved` `[:132]`.
- **Signal‑based plugin registration for parsers.** Each parser app connects its declaration to `document_consumer_declaration` in its `apps.ready()` (e.g. `[src/paperless_text/apps.py:11-13]`); dispatch picks the **highest‑weight** declaration in `get_parser_class_for_mime_type` `[src/documents/parsers.py:81]` by iterating `document_consumer_declaration.send(None)` `[:87]`. Observed weights: **Text = 10** `[src/paperless_text/signals.py:10]`, **Tika = 10** `[src/paperless_tika/signals.py:10]`, **Tesseract = 0** `[src/paperless_tesseract/signals.py:10]`.
- **Signed‑package task serialization.** The queued payload is a **pickled + Django‑signed** `dict` produced by `SignedPackage` `[django_q/signing.py:10]`, with **salt = cluster name `paperless`** and **key = Django `SECRET_KEY`** `[django_q/signing.py:14-19]`.
- **Synchronous post‑consume hooks.** Classification/matching/indexing run **inside** `consume_file` via the `document_consumption_finished` signal `[src/documents/apps.py:22-27]`, **not** as separate queued tasks.
- **`documents_log` defined‑but‑unwritten.** The `Log` model / `documents_log` table `[src/documents/models.py:285]` is defined but never populated by the v1.7.0 consumption path (no DB logging handler `[src/paperless/settings.py:373-412]`); `Log.objects.count()` observed to be `0`.
- **`optipng` is a required binary even for `.txt`.** Thumbnail optimisation runs `optipng`; its absence fails `consume_file` before the document row is written (observed).

### Security & performance notes (observed / cited)

- Task packages are **HMAC‑signed** with the Django `SECRET_KEY` (salt = cluster name `paperless` `[src/paperless/settings.py:449-450]`), so only clusters sharing both can unpack a package.
- The default Redis broker provides **at‑most‑once** delivery without message receipts (enqueue `rpush` / dequeue `blpop` `[django_q/brokers/redis_broker.py:17-23]`).
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
# Runtime + deps (see R1 for the full list)
redis-server --daemonize yes --save "" --appendonly no && redis-cli ping    # PONG
cd /src/src && python3 manage.py migrate                                     # exit 0

# Observe the broker BEFORE
redis-cli LLEN django_q:paperless:q                                          # 0

# Start the watcher, drop the file (real entry point)
python3 manage.py document_consumer &
printf "Paperless NGX runtime trace test.\nAcme Corporation invoice.\nDated 2022-04-01. Amount due 42.00 USD.\n" > /work/consume/report_2022.txt
md5sum /work/consume/report_2022.txt                                         # 0da8a96bf7f3a377f2acba04d2800b51

# Observe the broker DURING (task queued, worker not yet running)
redis-cli LLEN django_q:paperless:q                                          # 1
redis-cli --no-raw LINDEX django_q:paperless:q 0                             # <signed pickle package>

# Decode the queued package
python3 manage.py shell   # from django_q.signing import SignedPackage; SignedPackage.loads(raw_bytes)

# Start the worker; observe the broker AFTER
python3 manage.py qcluster &
redis-cli LLEN django_q:paperless:q                                          # 0

# Query the database after processing
python3 manage.py shell   # Document.objects.get(); Task.objects.get(...); LogEntry.objects.get(); Log.objects.count()  # -> 0
ls /work/data/index/                                                         # Whoosh index files
```

### Cleanup / verification performed

The source repository was mounted **read‑only**, all runtime data lived in container scratch under `/work`, `PYTHONDONTWRITEBYTECODE=1` prevented `.pyc` writes into the tree, and the container was removed afterward. All temporary observation scripts and test files were removed. **`git status --porcelain` on the source repository returns empty** — the tree is byte‑for‑byte unchanged; the only new artifact is this markdown document.

> The behaviours reported above are **stable and reproducible**: an independent re‑run of the exact `.txt` ingestion path (isolated data/broker outside the repo) reproduced the same ordered log stream, the same broker list key `django_q:paperless:q` with the identical `empty → 1 → drained` transition, the same decoded task‑dict shape (`func = documents.tasks.consume_file`, `kwargs = {'override_tag_ids': None}`, salt `paperless`), the same `checksum == md5` relationship, the same `django_q_task` result string form (`Success. New document id N created`), `Log.objects.count() == 0`, and the same Whoosh index file set — with only the inherently variable values (timestamps, absolute paths, scratch‑dir hashes, task/document ids) differing between runs.

---

## Coverage Pass — every part of the question, answered

| # | Requirement | Where answered | Key observed value(s) |
|---|-------------|----------------|-----------------------|
| **R1** | Bring up the full runtime (Redis, DB, qcluster, document_consumer, gunicorn) | R1 | `redis-cli ping → PONG`; `migrate` exit 0; seeded `django_q_schedule`; `consumer` user `(1,'consumer')`; baseline counts 0 |
| **R2** | Drop a test file into the consumption dir and follow it (real entry point) | R2 | `report_2022.txt`, 100 bytes, md5 `0da8a96bf7f3a377f2acba04d2800b51`; routes to `TextDocumentParser` (weight 10) |
| **R3** | Enumerate log messages + triggered tasks | R3 | full ordered stream (12 lines) verbatim; ingestion task `documents.tasks.consume_file` `[tasks.py:184]`; inline 6 handlers vs separate scheduled tasks |
| **R4** | Inspect the queued task on the broker | R4 | list `django_q:paperless:q`; `LLEN` 0→1→0; 441‑byte signed pickle; decoded `dict` `{args,func,id,kwargs,name,started}`; salt `paperless` |
| **R5** | Query the DB after processing (record, parsed fields, task history) | R5 | `documents_document` row (`checksum == md5`); `django_q_task` `Success. New document id 1 created`; `django_admin_log` ADDITION by `consumer`; `documents_log` count `0`; Whoosh files |
| **R6** | Trace the codepath + name the framework | R6 | `document_consumer._consume` → `async_task` (builds+signs+enqueues) → `qcluster` runs `consume_file`; framework = **Django‑Q + Redis**, not Celery |
| **S1** | Stable ordering ≥2 runs | Secondary §1 | byte‑for‑byte identical ordering (id 1 and id 2) |
| **S2** | Duplicate file | Secondary §2 | `Not consuming ...: It is a duplicate.`; task `success = False` |
| **S3** | Unsupported MIME (2 paths) | Secondary §3 | watcher‑level `Unknown file extension.` (no task); consumer‑level `Unsupported mime type application/octet-stream` |
| **S4** | Polling vs inotify | Secondary §4 | `Using inotify ...` (default) vs `Polling directory ...` + `Waiting for file ... to remain unmodified` |
| **S5** | Barcode‑split branch | Secondary §5 | cited from source (`tasks.py:195`, `:96`, `:233`); off by default, not run |

**Named mechanisms/functions addressed:** `document_consumer` command, `Handler.on_created/on_moved`, `_consume`, `is_file_ext_supported`, `async_task`, `SignedPackage.dumps/loads`, `PickleSerializer`, Redis `rpush`/`blpop`, `qcluster`, `consume_file`, `Consumer.try_consume_file`, `pre_check_duplicate`, `get_parser_class_for_mime_type`, `TextDocumentParser`, `document_consumption_finished` and its six handlers (`add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index`), `match_correspondents/document_types/tags`, `index.add_or_update_document`, `LoggingMixin.log`, `Document` / `Log` models, the `Task` model, and the scheduled tasks `train_classifier` / `index_optimize` / `sanity_check`.
