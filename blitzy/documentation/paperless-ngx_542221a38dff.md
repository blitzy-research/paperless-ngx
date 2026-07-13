# paperless-ngx — Document Flow Knowledge Capture

**Repository:** paperless-ngx · **Branch:** `paperless-ngx_542221a38dff` · **HEAD:** `542221a38dff06361e07976452f9aea24d210542`
**Project version:** `(1, 7, 0)` — `src/paperless/version.py:L1`

This document answers seven questions about how a document flows through paperless-ngx v1.7.0: how it enters the system (Q1), the stages it passes through (Q2), the background-execution engine (Q3), the metadata stored per document (Q4), how those fields are classified as required / optional / derived (Q5), a concrete runtime example (Q6), and how tags, correspondents, and document types organize documents together (Q7).

Every factual claim carries an exact `file:line` reference and is labeled **[observed]** (produced by running real code in the canonical runtime and captured first-hand) or **[inferred]** (read from source, not executed). Where a section header carries a label, individual source-derived sub-claims inside it are still marked **[inferred]** so the two are never conflated. This was a strictly read-only investigation; no repository file was modified — see the closing *Reproduction, determinism & cleanup* section for the `git status --porcelain` proof.

---

## Environment & Methodology

- **Canonical runtime.** The project's canonical runtime is the Docker image `python:3.9-slim-bullseye` — `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`). Every observation below was produced **inside that canonical runtime**: container `paperless-app`, **Python 3.9.23**, with each dependency at the exact version pinned in `requirements.txt` (django `4.0.4` `:L38`, django-q `1.3.9` `:L37`, channels `3.0.4` `:L23`, channels-redis `3.4.0` `:L22`, redis `3.5.3` `:L84`, scikit-learn `1.0.2` `:L88`, whoosh `2.7.4` `:L111`, watchdog `2.1.7` `:L106`, python-magic `0.4.25` `:L79`, fuzzywuzzy[speedup] `0.18.0` `:L41`, imap-tools `0.54.0` `:L49`, dateparser `1.1.1` `:L32`), plus Tesseract, `libmagic`, `python-Levenshtein`, and a live Redis at `paperless-redis:6379`.
- **No custom harness — the real settings module is used.** Every probe runs against the project's own settings via `DJANGO_SETTINGS_MODULE=paperless.settings`; there is no bespoke settings file. The only configuration is the standard `PAPERLESS_*` environment variables (below), pointing data/media/index at a throwaway `/tmp/ppinv` directory **outside** the repository checkout (bind-mounted at `/app`). The observed behavior is therefore the *real* application's behavior, not a reduced stand-in.
- **Run first, then write.** Each **[observed]** section shows the exact command and its **complete, unedited** captured output — both stdout and stderr. The application routes its own logging to `PAPERLESS_LOGGING_DIR`; the few INFO lines that reach stderr are shown in full and *corroborate* the stdout evidence (nothing is hidden). The only run-to-run variation is (a) timestamp prefixes in stderr log lines, (b) the `created` datetime that falls back to file mtime, and (c) django-q's randomly generated task id and cluster name — all flagged where they appear.
- **Two clean runs.** The whole investigation was executed twice from a freshly migrated database. All stability-sensitive values (the 16-field inventory, the md5 checksums, every matching boolean and score, the schedule rows and constants, the progress stages, the signal receiver order and effects) were **byte-identical across both runs**; the closing section shows the `diff` result.
- **What is genuinely observed vs inferred.** Q1's async convergence and Q3's engine were **executed** end-to-end (real `async_task` → Redis → a real `qcluster` worker). Q2's stage pipeline was **executed** for the text/plain parser path and its progress broadcasts captured live. Q4/Q5 (model metadata), Q6 (the finished-signal organization), and Q7 (all matching algorithms) were **executed**. The only pieces reported **[inferred]** are branches that require inputs this investigation deliberately did not fabricate — OCR/Tesseract parsing of a real scanned PDF, the barcode-splitting branch (needs a barcode-separated PDF and a non-default flag), and the ML `MATCH_AUTO` classifier prediction (needs a trained model file) — each grounded in exact `file:line` references and never asserted as an observed outcome.

### Reproducibility harness

All probes share this setup. The environment variables and per-probe clean-state reset are:

```bash
# Inside the canonical container `paperless-app` (Python 3.9.23):
export PYTHONPATH=/app/src
export DJANGO_SETTINGS_MODULE=paperless.settings
export PAPERLESS_DATA_DIR=/tmp/ppinv/data
export PAPERLESS_MEDIA_ROOT=/tmp/ppinv/media
export PAPERLESS_STATICDIR=/tmp/ppinv/static
export PAPERLESS_CONSUMPTION_DIR=/tmp/ppinv/consume
export PAPERLESS_LOGGING_DIR=/tmp/ppinv/log
export PAPERLESS_SECRET_KEY=investigation-key

# Clean state before EACH probe (fresh throwaway SQLite DB + Whoosh index, all under /tmp):
rm -rf /tmp/ppinv
mkdir -p /tmp/ppinv/{data,media,static,consume,log}
cd /app/src && python3 manage.py migrate --no-input   # applies all migrations incl. the schedule data-migrations
```

`python3 manage.py check` reports `System check identified no issues (0 silenced).` under this configuration. The six probe scripts live in `/tmp/ppscripts/` and are reproduced **in full** in their sections below. The driver that ran all six from clean state, twice, was:

```bash
run_probe() {   # reset to clean state, then run one probe capturing stdout/stderr separately
  local name="$1"; local script="$2"
  rm -rf /tmp/ppinv && mkdir -p /tmp/ppinv/{data,media,static,consume,log}
  ( cd /app/src && python3 manage.py migrate --no-input ) >/dev/null 2>&1
  ( cd /app/src && python3 "/tmp/ppscripts/$script" ) >"$OUTDIR/$name.out" 2>"$OUTDIR/$name.err"
}
```

---

## Q1 — How does a new document enter paperless-ngx? (ingestion entry points)

A new document normally enters through **one of three canonical entry points**. Two of them (REST upload, mail fetch) *write* the incoming bytes to a temporary file; the third (the directory watcher) does **not** write bytes itself — it *observes* a file that some external actor (a scanner, an `rsync`, a copy) has already placed in the consumption directory. All three then enqueue the **same** background task, `documents.tasks.consume_file`, onto the django-q broker via `async_task(...)` (from `django_q.tasks`). The async dispatch is **[observed]** end-to-end in the combined Q1/Q3 probe below; the per-entry-point wiring is **[inferred]** from the cited files.

**1. Consumption-directory watcher** — `src/documents/management/commands/document_consumer.py` — **[inferred from source]**
- Imports the task API: `from django_q.tasks import async_task` — `:L13`.
- The command watches `CONSUMPTION_DIR`. It does **not** always poll: with the default `PAPERLESS_CONSUMER_POLLING=0` (`src/paperless/settings.py:L478`) it uses **inotify** when available — `if settings.CONSUMER_POLLING == 0 and INotify:` `:L178` → `self.handle_inotify(...)` `:L179` — and only falls back to a `PollingObserver` (`:L187`, inside `handle_polling` `:L185`) when polling is configured or inotify is unavailable (`PollingObserver` imported `:L17`; `INotify` imported `:L20`).
- When a stable, already-written file is detected, `def _consume(filepath)` — `:L46` — validates it and enqueues:
  - `async_task(` — `:L86`
  - `"documents.tasks.consume_file",` — `:L87`
- A watcher **subdirectory** may additionally supply **tag** overrides derived from the folder path; it does not set title/correspondent/type overrides.

**2. REST API upload** — `src/documents/views.py` — **[inferred from source]**
- Imports `from django_q.tasks import async_task` — `:L28`.
- `class PostDocumentView(GenericAPIView)` — `:L491` — is an **authenticated, multipart** endpoint: `permission_classes = (IsAuthenticated,)` `:L493` and `parser_classes = (parsers.MultiPartParser,)` `:L495`. Its route is **`/api/documents/post_document/`** — wired in `src/paperless/urls.py`: `r"^api/"` `:L40` + `r"^documents/post_document/"` `:L57`, `PostDocumentView.as_view()` `:L58`, `name="post_document"` `:L59`.
- Its `def post(...)` — `:L497` — streams the upload into a temp file (`tempfile.NamedTemporaryFile(...)` — `:L512`) and enqueues:
  - `async_task(` — `:L523`
  - `"documents.tasks.consume_file",` — `:L524`
- REST supplies `override_filename` `:L526`, `override_title` `:L527`, `override_correspondent_id` `:L528`, `override_document_type_id` `:L529`, and `override_tag_ids` `:L530`.

**3. Mail fetch** — `src/paperless_mail/mail.py` — **[inferred from source]**
- Imports `from django_q.tasks import async_task` — `:L11`.
- It consumes only real attachments — it skips parts whose `content_disposition` is not `"attachment"` (`:L292`) and only types accepted by `is_mime_type_supported(...)` (`:L319`, imported `:L14`). For each accepted attachment it writes a temp file (`tempfile.mkstemp(...)` — `:L322`) and enqueues:
  - `async_task(` — `:L336`
  - `"documents.tasks.consume_file",` — `:L337`
- Mail supplies `override_filename` `:L339`, `override_title` `:L342`, `override_correspondent_id` `:L343`, `override_document_type_id` `:L346`, and `override_tag_ids` `:L347` from the matching mail rule.
- This path is itself driven periodically by the scheduled task `paperless_mail.tasks.process_mail_accounts` (see Q3).

**Convergence — with one qualification.** All three call the one task `def consume_file(...)` — `src/documents/tasks.py:L184`. That task does **not** unconditionally construct a `Consumer`: it first checks the barcode-splitting feature — **[inferred from source]**

- `if settings.CONSUMER_ENABLE_BARCODES:` — `src/documents/tasks.py:L195` (the flag defaults to **False** — `src/paperless/settings.py:L502-L504`, via `__get_boolean(..., default="NO")` `:L34`; confirmed **[observed]** `CONSUMER_ENABLE_BARCODES = False` in the probe environment). When enabled, it scans for separator barcodes (`scan_file_for_separating_barcodes` `:L198`) and, **if separators are found**, splits the input, copies each part back to the consumption directory (`save_to_dir` `:L210`), deletes the original (`os.unlink(path)` `:L214`), broadcasts a `SUCCESS`/`finished` payload to `"status_updates"` (`:L227`), and **returns `"File successfully split"` without ever constructing a `Consumer`** — `:L233`. The split files re-enter through the watcher as fresh inputs.
- Only on the **non-split path** (barcodes disabled, or enabled but none found) does it build the pipeline: `document = Consumer().try_consume_file(...)` — `src/documents/tasks.py:L236` — running the stages in Q2.

So the precise statement is: **in the default configuration** (`CONSUMER_ENABLE_BARCODES=False`) every entry point converges on `Consumer.try_consume_file`, and processing (Q2) and organization (Q6/Q7) are identical regardless of arrival path; with barcode splitting enabled, a separated input is short-circuited into per-document re-consumption. Per-entry-point **overrides** differ (watcher: tags only; REST/mail: title/correspondent/type/tags) and are applied by `Consumer.apply_overrides` (`src/documents/consumer.py:L414`) — see Q2/Q6.

**[observed] — the shared async dispatch actually executed.** The probe calls the exact `async_task("documents.tasks.consume_file", ...)` all three entry points call, and a real `qcluster` worker executes it (the worker log is embedded in Q3). Script `/tmp/ppscripts/q1q3_async.py`:

```python
import os, django, time
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django_q.tasks import async_task
from django_q.models import Task
from documents.models import Document

src = "/tmp/ppinv/upload.txt"
with open(src, "w") as f:
    f.write("Async ingestion test: Invoice from Globex, amount 42.00 USD\n")

# This is EXACTLY the call all three entry points make (watcher/REST/mail):
task_id = async_task("documents.tasks.consume_file", src, task_name="probe-consume")
print(f"enqueued via async_task('documents.tasks.consume_file', ...): task_id={task_id}")

doc = None
for _ in range(90):
    time.sleep(1)
    doc = Document.objects.all().first()
    if doc:
        print(f"qcluster worker created document: id={doc.id} title={doc.title!r} "
              f"mime_type={doc.mime_type} checksum={doc.checksum}")
        break
if not doc:
    print("no document created within timeout")

t = Task.objects.filter(id=task_id).first()
if t:
    print(f"django_q Task record: func={t.func} success={t.success} result={t.result!r}")
```

Command and complete output:

```bash
# a qcluster worker is running in the background (see Q3):
python3 /tmp/ppscripts/q1q3_async.py
```

stdout:

```text
enqueued via async_task('documents.tasks.consume_file', ...): task_id=30be5a0a627a42abb79f11d3b381a290
qcluster worker created document: id=1 title='upload' mime_type=text/plain checksum=4bd40d5fb22298f57c4813682df7f19c
django_q Task record: func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
```

stderr:

```text
18:05:55 [Q] INFO Enqueued 1
```

Cause → effect: the enqueued `task_id` is a django-q handle; a worker dequeued it from Redis, ran `documents.tasks.consume_file`, which (barcodes disabled) called `Consumer().try_consume_file` and created document `id=1`; the `django_q.models.Task` row records `success=True` and `result='Success. New document id 1 created'`. The `task_id` is a random uuid — the only value that changes between runs; the document `checksum` (`4bd40d5fb22298f57c4813682df7f19c`) and the `result` are stable.

---

## Q2 — What stages does a document pass through? (the consume pipeline)

**[observed for the text/plain parser path; heavy parsers inferred].** `Consumer.try_consume_file` (`src/documents/consumer.py:L180`) runs an ordered pipeline and broadcasts progress via `_send_progress` (`def` `:L56`), which builds a payload dict (`:L64-L72`) and `group_send()`s it to the `"status_updates"` channel group (`:L73-L76`). The message codes are the `MESSAGE_*` constants at `:L43-L49`. The percentages, statuses, and message codes below are **[observed]** — captured live by subscribing a channel to that group during a real consume.

Ordered stages, each with its source citation:

1. **`STARTING` / `new_file` at 0%** — `:L202` (`MESSAGE_NEW_FILE` `:L43`).
2. **Pre-checks** (no progress payload), in this real call order — `pre_check_file_exists()` `:L211` (`def` `:L95`; raises if the file is missing), then `pre_check_directories()` `:L212` (`def` `:L115`; creates scratch/thumbnail dirs), then `pre_check_duplicate()` `:L213` (`def` `:L102`; rejects when the md5 `checksum` — or `archive_checksum`, `:L106` — already exists). The order is **file-exists → directories → duplicate**, not duplicate-first.
3. **MIME detection & parser selection** (no progress payload) — the MIME type is detected with `magic.from_file(self.path, mime=True)` (`:L219`), then the parser class is chosen by `get_parser_class_for_mime_type()` (defined in `src/documents/parsers.py:L81`, imported at `consumer.py:L26`, called at `:L223`). If no parser matches the MIME type, the file is **rejected as unsupported here** — `if not parser_class:` (`:L224`) calls `self._fail(MESSAGE_UNSUPPORTED_TYPE, ...)` (`:L225`; `MESSAGE_UNSUPPORTED_TYPE` `:L44`) — **before** the `document_consumption_started` signal in the next step. MIME detection and parser selection are **[observed]** (the observed text/plain run detected the type at `:L219` and selected the text parser at `:L223`); the unsupported-type rejection branch is **[inferred]** — it was not taken on that supported-type run.
4. **`document_consumption_started` signal** fires — `:L229`.
5. **`WORKING` / `parsing_document` at 20%** — `:L259` (`MESSAGE_PARSING_DOCUMENT` `:L45`); the selected parser's `parse()` is **invoked** at `:L261` (reached only for supported types — the unsupported-type check already happened in step 3). During parsing the parser may call `progress_callback` (`def` `:L237`), which maps parser progress into the band via `p = int((current_progress / max_progress) * 50 + 20)` (`:L239`) — so parser progress spans **20%–70%**. NOTE: the code *comment* at `:L238` says "within 20 and 80", but the arithmetic (`* 50 + 20`) tops out at **70**, and the next fixed payload is 70% for thumbnailing — so the observed parse band is **20–70%**, not 20–80%.
6. **`WORKING` / `generating_thumbnail` at 70%** — `:L264` (`MESSAGE_GENERATING_THUMBNAIL` `:L46`).
7. **`WORKING` / `parse_date` at 90%** — `:L274` (`MESSAGE_PARSE_DATE` `:L47`) — emitted **only when the parser did not already return a date** (`parse_date()` called at `:L275`). This edge branch was exercised in the observed run (the text parser returns no date), which is why `parse_date` appears.
8. **`WORKING` / `save_document` at 95%** — `:L294` (`MESSAGE_SAVE_DOCUMENT` `:L48`), inside `transaction.atomic()` (`:L298`). `_store()` (`:L379`) creates the `Document`; the **`document_consumption_finished` signal fires** (`:L306`) — this triggers organization (Q6/Q7). Under `FileLock` (`:L315`) the original, thumbnail, and optional archive are written; `generate_unique_filename()` (`:L316`, defined in `file_handling.py:L81`) computes the storage name; when an archive is produced, `archive_filename` is set (`:L328`) and `archive_checksum` is computed as an md5 over the archive bytes (`:L339-L342`).
9. **`SUCCESS` / `finished` at 100%** — `:L375` (`MESSAGE_FINISHED` `:L49`), carrying the new `document.id`.

**Failure path (unified).** Every pre-check and parser failure funnels through `_fail()` (`def` `:L78`), which first broadcasts a terminal **`FAILED`** payload at 100% — `self._send_progress(100, 100, "FAILED", message)` (`:L79`) — and then raises `ConsumerError` (`:L81`). This unified failure path is **[inferred]**: no failure branch was triggered on the observed successful run, so the `FAILED` payload was not emitted during the capture.

Optional `PRE_CONSUME_SCRIPT` (`run_pre_consume_script` `:L121`) and `POST_CONSUME_SCRIPT` (`run_post_consume_script` `:L143`) run before/after when configured.

**Date precedence** — in `_store`, `created = file_info.created or date or timezone.make_aware(datetime.datetime.fromtimestamp(stats.st_mtime))` (`:L389-L393`): a date encoded in the filename wins; else the parser/`parse_date` date; else the file's mtime. In the observed text run there was no filename date and the text parser returned none, so `created` fell back to mtime — visible below.

Script `/tmp/ppscripts/q2_consume.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from documents.consumer import Consumer

# Subscribe a channel to the real 'status_updates' group BEFORE consuming so we
# capture the exact payloads Consumer._send_progress broadcasts (consumer.py:L73).
layer = get_channel_layer()
my_channel = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)("status_updates", my_channel)

src = "/tmp/ppinv/sample.txt"
with open(src, "w") as f:
    f.write("Invoice 2022 from ACME Corp, total due 199.00 EUR\n")

# Canonical entry into the pipeline (same method tasks.consume_file calls).
doc = Consumer().try_consume_file(src)
print(f"consumed document: id={doc.id} title={doc.title!r} mime_type={doc.mime_type}")
print(f"  checksum={doc.checksum}")
print(f"  content={doc.content!r}")
print(f"  filename={doc.filename}")
print(f"  created={doc.created!r}  (mtime fallback: no filename date, text parser returned no date)")
print("real _send_progress payloads broadcast on 'status_updates' during the consume:")
for _ in range(20):
    msg = async_to_sync(layer.receive)(my_channel)
    d = msg.get("data", {})
    print(f"  current={d.get('current_progress')} max={d.get('max_progress')} "
          f"status={d.get('status')} message={d.get('message')}")
    if d.get("status") == "SUCCESS":
        break
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q2_consume.py
```

stdout:

```text
consumed document: id=1 title='sample' mime_type=text/plain
  checksum=0f03d7e3739efdee9bd1235de8eb6587
  content='Invoice 2022 from ACME Corp, total due 199.00 EUR\n'
  filename=0000001.txt
  created=datetime.datetime(2026, 7, 13, 18, 5, 35, 339972, tzinfo=zoneinfo.ZoneInfo(key='UTC'))  (mtime fallback: no filename date, text parser returned no date)
real _send_progress payloads broadcast on 'status_updates' during the consume:
  current=0 max=100 status=STARTING message=new_file
  current=20 max=100 status=WORKING message=parsing_document
  current=70 max=100 status=WORKING message=generating_thumbnail
  current=90 max=100 status=WORKING message=parse_date
  current=95 max=100 status=WORKING message=save_document
  current=100 max=100 status=SUCCESS message=finished
```

stderr (the application's own INFO log, corroborating start and finish):

```text
[2026-07-13 18:05:35,364] [INFO] [paperless.consumer] Consuming sample.txt
[2026-07-13 18:05:37,910] [INFO] [paperless.consumer] Document 2026-07-13 sample consumption finished
```

The full pipeline for a real scanned PDF (Tesseract OCR via `paperless_tesseract`) is **[inferred]** from the parser selection at `consumer.py:L223` (`get_parser_class_for_mime_type`, `src/documents/parsers.py:L81`) and the `parse()` invocation at `consumer.py:L261`; the stage **order** and the progress/message codes are **[observed]** via the text/plain parser path, which exercises the identical `Consumer.try_consume_file` code.

---

## Q3 — Are there background jobs, and what runs them? (the execution engine)

**[observed engine + schedules; supervisord wiring inferred].** The background-execution engine is **django-q 1.3.9** (`requirements.txt:L37`) — **not Celery** (this version predates the project's later Celery migration; there is no Celery dependency here). It is configured by the `Q_CLUSTER` dict in `src/paperless/settings.py:L449-L457` — `"name": "paperless"` `:L450`, `"workers": TASK_WORKERS` `:L455` (`TASK_WORKERS` defined `:L438` from `PAPERLESS_TASK_WORKERS`), `"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` `:L456`. `"django_q"` is in `INSTALLED_APPS` `:L110`.

- **Worker process.** The worker is `qcluster`, launched in the canonical image by supervisord: `docker/supervisord.conf` `[program:scheduler]` `:L28`, `command=python3 manage.py qcluster` `:L29`, `user=paperless` `:L30` **[inferred from that config; that qcluster runs is observed below]**. (Separately, `[program:consumer]` `:L19-L21` runs the directory watcher `document_consumer`; `[program:gunicorn]` `:L10-L12` runs the ASGI server.)
- **Real-time progress** uses Django Channels over Redis: `CHANNEL_LAYERS` (`settings.py:L178-L182`) with `RedisChannelLayer` `:L180` and hosts from `PAPERLESS_REDIS` `:L182`; `consume_file` broadcasts to the `"status_updates"` group (`tasks.py:L227`), the same group `Consumer._send_progress` uses.
- **Ad-hoc (fire-and-forget) tasks** enqueued via `async_task`: `documents.tasks.consume_file` (`tasks.py:L184`) and `documents.tasks.bulk_update_documents` (`:L270`).
- **Scheduled tasks** are registered as django-q `Schedule` rows by data migrations. The `Schedule` type constants were confirmed **[observed]**: `HOURLY='H'`, `DAILY='D'`, `WEEKLY='W'`, `MINUTES='I'`. All four rows below are **[observed]** from a real `migrate` (task fns: `index_optimize` `tasks.py:L32`, `train_classifier` `:L48`, `sanity_check` `:L255`):

| Scheduled task | Frequency | Registered in |
|----------------|-----------|---------------|
| `documents.tasks.train_classifier` | HOURLY (`H`) | `src/documents/migrations/1001_auto_20201109_1636.py:L11` (func), `:L13` (type) |
| `documents.tasks.index_optimize` | DAILY (`D`) | `src/documents/migrations/1001_auto_20201109_1636.py:L16` (func), `:L18` (type) |
| `documents.tasks.sanity_check` | WEEKLY (`W`) | `src/documents/migrations/1004_sanity_check_schedule.py:L11-L13` |
| `paperless_mail.tasks.process_mail_accounts` | every 10 MINUTES (`I`, `minutes=10`) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L11-L14` |

Script `/tmp/ppscripts/q3_schedules.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django_q.models import Schedule

print(
    "Schedule type constants: "
    f"HOURLY={Schedule.HOURLY!r} DAILY={Schedule.DAILY!r} "
    f"WEEKLY={Schedule.WEEKLY!r} MINUTES={Schedule.MINUTES!r}"
)
print("django-q Schedule rows registered by the data migrations:")
for s in Schedule.objects.all().order_by("func"):
    print(f"  func={s.func} schedule_type={s.schedule_type} minutes={s.minutes}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q3_schedules.py
```

stdout:

```text
Schedule type constants: HOURLY='H' DAILY='D' WEEKLY='W' MINUTES='I'
django-q Schedule rows registered by the data migrations:
  func=documents.tasks.index_optimize schedule_type=D minutes=None
  func=documents.tasks.sanity_check schedule_type=W minutes=None
  func=documents.tasks.train_classifier schedule_type=H minutes=None
  func=paperless_mail.tasks.process_mail_accounts schedule_type=I minutes=10
```

stderr: *(empty)*

**[observed] end-to-end execution.** For Q1/Q3 the probe enqueued `consume_file` to the real Redis broker and a real `qcluster` worker executed it. The worker's own INFO log shows the cluster lifecycle and the task being processed (the probe's stdout/stderr are in Q1):

```bash
# started before the probe, logging to the investigation log dir:
python3 manage.py qcluster
```

```text
18:05:49 [Q] INFO Q Cluster island-tennessee-minnesota-oven starting.
18:05:49 [Q] INFO Process-1:1 ready for work at 30851
18:05:49 [Q] INFO Process-1:2 ready for work at 30852
18:05:49 [Q] INFO Process-1:3 ready for work at 30853
18:05:49 [Q] INFO Process-1:4 ready for work at 30854
18:05:49 [Q] INFO Process-1:5 ready for work at 30855
18:05:49 [Q] INFO Process-1:6 ready for work at 30856
18:05:49 [Q] INFO Process-1:7 ready for work at 30857
18:05:49 [Q] INFO Process-1:8 ready for work at 30858
18:05:49 [Q] INFO Process-1:9 ready for work at 30859
18:05:49 [Q] INFO Process-1:10 ready for work at 30860
18:05:49 [Q] INFO Process-1:11 ready for work at 30861
18:05:49 [Q] INFO Process-1:12 monitoring at 30862
18:05:49 [Q] INFO Process-1 guarding cluster island-tennessee-minnesota-oven
18:05:49 [Q] INFO Process-1:13 pushing tasks at 30863
18:05:49 [Q] INFO Q Cluster island-tennessee-minnesota-oven running.
18:05:55 [Q] INFO Process-1:1 processing [probe-consume]
[2026-07-13 18:05:55,676] [INFO] [paperless.consumer] Consuming upload.txt
[2026-07-13 18:05:56,442] [INFO] [paperless.consumer] Document 2026-07-13 upload consumption finished
18:05:56 [Q] INFO Process-1:1 stopped doing work
18:05:56 [Q] INFO Processed [probe-consume]
18:05:56 [Q] INFO Q Cluster island-tennessee-minnesota-oven stopping.
18:05:56 [Q] INFO Process-1 stopping cluster processes
18:05:57 [Q] INFO Process-1:13 stopped pushing tasks
18:05:57 [Q] INFO Process-1:2 stopped doing work
18:05:57 [Q] INFO Process-1:3 stopped doing work
18:05:57 [Q] INFO Process-1:4 stopped doing work
18:05:57 [Q] INFO Process-1:5 stopped doing work
18:05:57 [Q] INFO Process-1:6 stopped doing work
18:05:57 [Q] INFO Process-1:7 stopped doing work
18:05:57 [Q] INFO Process-1:8 stopped doing work
18:05:57 [Q] INFO Process-1:9 stopped doing work
18:05:57 [Q] INFO Process-1:10 stopped doing work
18:05:57 [Q] INFO Process-1:11 stopped doing work
18:05:58 [Q] INFO Process-1 waiting for the monitor.
18:05:58 [Q] INFO Process-1:12 stopped monitoring results
18:05:58 [Q] INFO Q Cluster island-tennessee-minnesota-oven has stopped.
```

Cause → effect: the cluster `island-tennessee-minnesota-oven` started 13 processes, `Process-1:1` picked up `[probe-consume]`, ran the real consumer (`Consuming upload.txt` → `consumption finished`), reported `Processed [probe-consume]`, and stopped cleanly. (The cluster name is randomly generated per start — the only varying value.)

---

## Q4 — What metadata fields are stored for each document?

**[observed].** The real `Document` model (`src/documents/models.py:L88`) was introspected via Django's `_meta.get_fields()` API — 16 fields. Script `/tmp/ppscripts/q4q5_fields.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.db.models.fields import NOT_PROVIDED, AutoField
from documents.models import Document, FileInfo

print("=" * 72)
print("Q4/Q5 - Document model fields via Django _meta API [observed]")
print("=" * 72)
fields = Document._meta.get_fields()
print(f"total fields reported by _meta.get_fields(): {len(fields)}")
print(f"{'field':22}{'type':18}{'null':6}{'blank':7}{'editable':9}default")
print("-" * 69)
for f in fields:
    default = getattr(f, "default", NOT_PROVIDED)
    if default is NOT_PROVIDED:
        dflt = "-"
    elif default is None:
        dflt = "None"
    elif callable(default):
        dflt = getattr(default, "__name__", str(default))
    else:
        dflt = str(default)
    print(f"{f.name:22}{type(f).__name__:18}"
          f"{str(getattr(f,'null','')):6}{str(getattr(f,'blank','')):7}"
          f"{str(getattr(f,'editable','')):9}{dflt}")

print()
print("Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):")
for f in fields:
    if isinstance(f, AutoField):
        continue
    default = getattr(f, "default", NOT_PROVIDED)
    if getattr(f, "null", None) is False and getattr(f, "blank", None) is False and default is NOT_PROVIDED:
        print(f"  - {f.name} ({type(f).__name__}, editable={getattr(f,'editable','')})")

print()
print("=" * 72)
print("Q4/Q5 edge - FileInfo.from_filename metadata derivation [observed]")
print("=" * 72)
for fn in ["invoice_acme.pdf",
           "20221101Z - Some Title.pdf",
           "20221101123045Z - Fourteen Digit.pdf",
           "just a scan.pdf"]:
    fi = FileInfo.from_filename(fn)
    print(f"from_filename({fn!r}): title={fi.title!r} created={fi.created!r}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q4q5_fields.py
```

stdout:

```text
========================================================================
Q4/Q5 - Document model fields via Django _meta API [observed]
========================================================================
total fields reported by _meta.get_fields(): 16
field                 type              null  blank  editable default
---------------------------------------------------------------------
id                    AutoField         False True   True     -
correspondent         ForeignKey        True  True   True     -
title                 CharField         False True   True     -
document_type         ForeignKey        True  True   True     -
content               TextField         False True   True     -
mime_type             CharField         False False  False    -
checksum              CharField         False False  False    -
archive_checksum      CharField         True  True   False    -
created               DateTimeField     False False  True     now
modified              DateTimeField     False True   False    -
storage_type          CharField         False False  False    unencrypted
added                 DateTimeField     False False  False    now
filename              FilePathField     True  False  False    None
archive_filename      FilePathField     True  False  False    None
archive_serial_number IntegerField      True  True   True     -
tags                  ManyToManyField   False True   True     -

Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):
  - mime_type (CharField, editable=False)
  - checksum (CharField, editable=False)

========================================================================
Q4/Q5 edge - FileInfo.from_filename metadata derivation [observed]
========================================================================
from_filename('invoice_acme.pdf'): title='invoice_acme' created=None
from_filename('20221101Z - Some Title.pdf'): title='Some Title' created=datetime.datetime(2022, 11, 1, 0, 0, tzinfo=tzlocal())
from_filename('20221101123045Z - Fourteen Digit.pdf'): title='Fourteen Digit' created=datetime.datetime(2022, 11, 1, 12, 30, 45, tzinfo=tzlocal())
from_filename('just a scan.pdf'): title='just a scan' created=None
```

stderr: *(empty)*

Field-by-field source citations (`src/documents/models.py`): `id` — the implicit `AutoField` primary key Django adds (no explicit declaration); `correspondent` `:L97`; `title` `:L106`; `document_type` `:L108`; `content` `:L117`; `mime_type` `:L126`; `tags` `:L128`; `checksum` `:L135`; `archive_checksum` `:L143`; `created` `:L152`; `modified` `:L154`; `storage_type` `:L161`; `added` `:L169`; `filename` `:L176`; `archive_filename` `:L186`; `archive_serial_number` `:L196`.

---

## Q5 — Which fields are required vs optional vs derived?

**[observed]** (same probe as Q4). The probe's *"Strictly REQUIRED"* section computes, at the model layer, which non-auto fields are `null=False AND blank=False AND` have no default — the result is exactly two: `mime_type` and `checksum`.

- **REQUIRED (model layer):** `mime_type` (`:L126`, `editable=False`) and `checksum` (`:L135`, a unique md5, `editable=False`). Nothing else is unconditionally required by the ORM.
- **`id` — implicit `AutoField` PK:** auto-generated by the database (AUTO); `editable=True` in metadata but assigned automatically, never supplied by the ingester.
- **OPTIONAL** (nullable and/or blank; no ingester obligation): `correspondent` (FK, null & blank, `:L97`), `document_type` (FK, null & blank, `:L108`), `archive_serial_number` (null & blank, `:L196`), `tags` (M2M, blank, `:L128`), and `title` (blank; stores `''` when omitted, `:L106`).
- **`content` (`:L117`) — a deliberate distinction:** it is `editable=True` at the model layer (the ORM does **not** force it), **but in the consume pipeline it is DERIVED** — `_store` assigns the parser's extracted text to it. So `content` is *optional at the model layer / derived in practice*; the two senses must not be conflated.
- **DERIVED / auto-managed** (during consume or by the ORM):
  - `checksum` — md5 of the document bytes (derived during consume, yet REQUIRED at the model layer).
  - `archive_checksum` (`:L143`) — md5 of the archive file when one is produced (else null).
  - `filename` (`:L176`) / `archive_filename` (`:L186`) — computed by `generate_unique_filename` at storage time.
  - `created` (`:L152`) — model default `timezone.now`, but during consume it is derived by the precedence `file_info.created or parser-date or mtime` (`consumer.py:L389-L393`).
  - `modified` (`:L154`) — `auto_now` (updated on every save).
  - `added` (`:L169`) — model default `timezone.now` (set once at insert).
  - `storage_type` (`:L161`) — default `'unencrypted'`.

**Date & title derivation, exercised [observed]** via `FileInfo.from_filename` (`models.py:L434`), which applies `FILENAME_PARSE_TRANSFORMS` (loop at `:L437`; regex at `:L393` accepts an 8-digit date **and** an optional extra 6 digits — i.e. 8- or 14-digit timestamps — terminated by `Z`). The probe output above shows:

- `'invoice_acme.pdf'` → `title='invoice_acme'`, `created=None` (no date pattern → the whole stem becomes the title).
- `'20221101Z - Some Title.pdf'` → 8-digit date → `created=2022-11-01 00:00`, `title='Some Title'`.
- `'20221101123045Z - Fourteen Digit.pdf'` → 14-digit date → `created=2022-11-01 12:30:45`, `title='Fourteen Digit'`.
- `'just a scan.pdf'` → `title='just a scan'`, `created=None`.

(The 8-digit form is zero-padded to 14 in `_get_created` (`def` `:L418`) by the expression `"{:0<14}Z".format(...)` at `:L420`.)

---

## Q6 — Concrete runtime example (the finished-signal organization)

**[observed].** A real `Document` was created with `content` set exactly as `_store` assigns parser text, then the **real** `document_consumption_finished` signal was fired — the same signal `Consumer` sends at `consumer.py:L306` — with `classifier=None` (pure rule-based). The six receivers connected in `DocumentsConfig.ready()` (`src/documents/apps.py:L22-L27`) run in connection order. Script `/tmp/ppscripts/q6_signal.py`:

```python
import os, django, hashlib
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.contrib.auth.models import User
from django.contrib.admin.models import LogEntry, ADDITION
from documents.models import Document, Correspondent, DocumentType, Tag, MatchingModel
from documents.signals import document_consumption_finished
from documents import index

# Prerequisites: the 'consumer' user (required by set_log_entry) + organizing entities.
User.objects.get_or_create(username="consumer")
Correspondent.objects.create(name="ACME Corp", match="acme",
                             matching_algorithm=MatchingModel.MATCH_ANY)
DocumentType.objects.create(name="Invoice", match="invoice",
                            matching_algorithm=MatchingModel.MATCH_ANY)
Tag.objects.create(name="Inbox", is_inbox_tag=True)              # match empty
Tag.objects.create(name="Paid", match="total due",
                   matching_algorithm=MatchingModel.MATCH_ALL)
Tag.objects.create(name="Travel", match="flight hotel",
                   matching_algorithm=MatchingModel.MATCH_ALL)

content = "Invoice 2022 from ACME Corp, total due 199.00 EUR"
doc = Document.objects.create(
    title="sample", content=content, mime_type="text/plain",
    checksum=hashlib.md5(content.encode()).hexdigest(),
)
print(f"document created: id={doc.id} checksum={doc.checksum}")

def state(d):
    d.refresh_from_db()
    return (f"correspondent={d.correspondent.name if d.correspondent else None} "
            f"document_type={d.document_type.name if d.document_type else None} "
            f"tags={sorted(t.name for t in d.tags.all())}")

print("BEFORE signal:", state(doc))

# Fire the REAL signal (classifier=None -> pure rule-based). send() returns
# the receivers in connection order together with their return values.
responses = document_consumption_finished.send(
    sender=None, document=doc, logging_group=None, classifier=None,
)
print("RECEIVERS fired (in connection order from apps.py:L22-L27):")
for recv, ret in responses:
    print(f"  {recv.__module__}.{recv.__name__} -> {ret!r}")

print("AFTER  signal:", state(doc))

# Receiver-specific evidence:
for le in LogEntry.objects.filter(object_id=str(doc.id)):
    print(f"set_log_entry -> LogEntry: user={le.user.username} "
          f"action_flag={le.action_flag} is_ADDITION={le.action_flag == ADDITION} "
          f"object_repr={le.object_repr!r}")

with index.open_index().searcher() as s:
    ids = sorted(int(hit["id"]) for hit in s.documents())
    print(f"add_to_index -> whoosh index doc ids: {ids}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q6_signal.py
```

stdout:

```text
document created: id=1 checksum=dbd97f5b73b9094ecd35ca4298857559
BEFORE signal: correspondent=None document_type=None tags=[]
RECEIVERS fired (in connection order from apps.py:L22-L27):
  documents.signals.handlers.add_inbox_tags -> None
  documents.signals.handlers.set_correspondent -> None
  documents.signals.handlers.set_document_type -> None
  documents.signals.handlers.set_tags -> None
  documents.signals.handlers.set_log_entry -> None
  documents.signals.handlers.add_to_index -> None
AFTER  signal: correspondent=ACME Corp document_type=Invoice tags=['Inbox', 'Paid']
set_log_entry -> LogEntry: user=consumer action_flag=1 is_ADDITION=True object_repr='2026-07-13 ACME Corp sample'
add_to_index -> whoosh index doc ids: [1]
```

stderr (the handlers' own INFO log, corroborating each assignment):

```text
[2026-07-13 18:05:42,069] [INFO] [paperless.handlers] Assigning correspondent ACME Corp to 2026-07-13 sample
[2026-07-13 18:05:42,073] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-13 ACME Corp sample
[2026-07-13 18:05:42,076] [INFO] [paperless.handlers] Tagging "2026-07-13 ACME Corp sample" with "Paid"
```

Cause → effect, receiver by receiver (with effect-line citations in `src/documents/signals/handlers.py`):

- **`add_inbox_tags`** (`:L30`) → adds every `Tag` with `is_inbox_tag=True` via `document.tags.add(*inbox_tags)` (`:L32`): added **`Inbox`**.
- **`set_correspondent`** (`:L35`) → `document.correspondent` is `None` (guard `:L47`: skip only if already set and `replace=False`), so it matched rule `acme` (MATCH_ANY) and assigned via `document.correspondent = selected` (`:L97`): **`ACME Corp`** (stderr: `Assigning correspondent ACME Corp`).
- **`set_document_type`** (`:L101`) → matched `invoice` (MATCH_ANY), assigned via `document.document_type = selected` (`:L164`): **`Invoice`** (stderr: `Assigning document type Invoice`).
- **`set_tags`** (`:L168`) → additively adds every matching tag via `document.tags.add(*relevant_tags)` (`:L230`): `total due` (MATCH_ALL) matched → **`Paid`** added; `flight hotel` (MATCH_ALL) did **not** match → `Travel` excluded (stderr: `Tagging ... with "Paid"`).
- **`set_log_entry`** (`:L413`) → writes an audit row via `LogEntry.objects.create(action_flag=ADDITION, ...)` (`:L418`), attributed to the `consumer` user: probe shows `user=consumer action_flag=1 is_ADDITION=True`.
- **`add_to_index`** (`:L428`) → indexes the document into Whoosh via `index.add_or_update_document(document)` (`:L431`; `src/documents/index.py:L118`): re-opening the index finds doc id `[1]`.

Net effect: **BEFORE** `correspondent=None document_type=None tags=[]` → **AFTER** `correspondent=ACME Corp document_type=Invoice tags=['Inbox', 'Paid']`. The document checksum `dbd97f5b73b9094ecd35ca4298857559` (md5 of the content) is stable across both runs.

---

## Q7 — How do tags, correspondents, and document types organize documents together?

**[observed matching; ML classifier boundary inferred].** Organization uses three `MatchingModel` (`src/documents/models.py:L19`) subclasses — **`Correspondent`** (`:L57`), **`Tag`** (`:L64`), and **`DocumentType`** (`:L82`). `MatchingModel` provides `match` (`:L39`), `matching_algorithm` (`:L41`), and `is_insensitive` (`:L47`, default `True`). **`Tag` adds two fields, not one:** `color` (`:L66`, default `'#a6cee3'`) **and** `is_inbox_tag` (`:L68`, default `False`).

A document links to exactly **one** correspondent (FK `:L97`) and **one** document_type (FK `:L108`), and to **many** tags (M2M `:L128`). Assignment happens automatically at consumption through the six handlers (`apps.py:L22-L27`). For each of correspondents / types / tags, the handler asks the ML classifier for a prediction when a rule uses `MATCH_AUTO`, and unions that with the rule-based `matches()` filter (`matching.py:L60`): `match_correspondents` (`:L21`), `match_document_types` (`:L34`), `match_tags` (`:L47`).

Script `/tmp/ppscripts/q7_matching.py`:

```python
import os, django, re
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from documents.models import Correspondent, MatchingModel, Tag
from documents.matching import matches, _split_match

class FakeDoc:
    content = "Invoice 2022 from ACME Corp, total due 199.00 EUR"

doc = FakeDoc()
print("=" * 72)
print("Q7 - REAL matches() across EVERY algorithm + empty-match [observed]")
print("=" * 72)
print(f"document.content = {doc.content!r}")
print()

def mk(match, algo):
    return Correspondent(name="probe", match=match,
                         matching_algorithm=algo, is_insensitive=True)

cases = [
    ("MATCH_ANY", "acme zzz", MatchingModel.MATCH_ANY),
    ("MATCH_ALL", "acme invoice", MatchingModel.MATCH_ALL),
    ("MATCH_ALL", "acme missing", MatchingModel.MATCH_ALL),
    ("MATCH_LITERAL", "ACME Corp", MatchingModel.MATCH_LITERAL),
    ("MATCH_REGEX", r"\d{3}\.\d{2}", MatchingModel.MATCH_REGEX),
    ("MATCH_FUZZY", "akme korp", MatchingModel.MATCH_FUZZY),
    ("MATCH_AUTO", "acme", MatchingModel.MATCH_AUTO),
    ("MATCH_ANY (empty)", "", MatchingModel.MATCH_ANY),
]
for label, m, algo in cases:
    print(f"{label:20} {m!r:16} -> {matches(mk(m, algo), doc)}")

print()
from fuzzywuzzy import fuzz
mm = re.sub(r"[^\w\s]", "", "akme korp").lower()
tt = re.sub(r"[^\w\s]", "", doc.content).lower()
print("MATCH_FUZZY internal score (matching.py:L130-L135), threshold >= 90:")
print(f"  fuzz.partial_ratio({mm!r}, {tt!r}) = {fuzz.partial_ratio(mm, tt)}")

print()
probe = Correspondent(name="p", match='total "flight hotel"  extra')
print("_split_match quoted-phrase behaviour (matching.py:L155-L171) [observed]:")
print(f"  match={probe.match!r} -> {_split_match(probe)}")

print()
print("MatchingModel algorithm constants + Tag defaults [observed]:")
print(f"  MATCH_ANY={MatchingModel.MATCH_ANY} MATCH_ALL={MatchingModel.MATCH_ALL} "
      f"MATCH_LITERAL={MatchingModel.MATCH_LITERAL} MATCH_REGEX={MatchingModel.MATCH_REGEX} "
      f"MATCH_FUZZY={MatchingModel.MATCH_FUZZY} MATCH_AUTO={MatchingModel.MATCH_AUTO}")
print(f"  Tag.is_inbox_tag default={Tag._meta.get_field('is_inbox_tag').default} "
      f"Tag.color default={Tag._meta.get_field('color').default!r}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q7_matching.py
```

stdout:

```text
========================================================================
Q7 - REAL matches() across EVERY algorithm + empty-match [observed]
========================================================================
document.content = 'Invoice 2022 from ACME Corp, total due 199.00 EUR'

MATCH_ANY            'acme zzz'       -> True
MATCH_ALL            'acme invoice'   -> True
MATCH_ALL            'acme missing'   -> False
MATCH_LITERAL        'ACME Corp'      -> True
MATCH_REGEX          '\\d{3}\\.\\d{2}' -> True
MATCH_FUZZY          'akme korp'      -> False
MATCH_AUTO           'acme'           -> False
MATCH_ANY (empty)    ''               -> False

MATCH_FUZZY internal score (matching.py:L130-L135), threshold >= 90:
  fuzz.partial_ratio('akme korp', 'invoice 2022 from acme corp total due 19900 eur') = 78

_split_match quoted-phrase behaviour (matching.py:L155-L171) [observed]:
  match='total "flight hotel"  extra' -> ['total', 'flight\\s+hotel', 'extra']

MatchingModel algorithm constants + Tag defaults [observed]:
  MATCH_ANY=1 MATCH_ALL=2 MATCH_LITERAL=3 MATCH_REGEX=4 MATCH_FUZZY=5 MATCH_AUTO=6
  Tag.is_inbox_tag default=False Tag.color default='#a6cee3'
```

stderr: *(empty)*

Per-algorithm behavior, all **[observed]** against `content = 'Invoice 2022 from ACME Corp, total due 199.00 EUR'` (`matches()` at `matching.py:L60`):

- **empty `match`** → `False` (guard `:L66-L67`).
- **`is_insensitive`** adds `re.IGNORECASE` (`:L69-L70`).
- **MATCH_ANY** (`:L84`): any listed word present → `True` (`'acme zzz'` → `True`).
- **MATCH_ALL** (`:L72`): all listed words present (`'acme invoice'` → `True`; `'acme missing'` → `False`).
- **MATCH_LITERAL** (`:L91`): the phrase as an escaped, `\b`-bounded, case-insensitive substring (`'ACME Corp'` → `True`).
- **MATCH_REGEX** (`:L107`): `re.search` of the pattern, catching `re.error` (`:L113`) (`'\d{3}\.\d{2}'` → `True`, finds `199.00`).
- **MATCH_FUZZY** (`:L127`): `fuzz.partial_ratio(match, text) >= 90` (`:L135`). `'akme korp'` → `False`; the probe prints the actual score = **78** (`python-Levenshtein` is installed; 78 < 90).
- **MATCH_AUTO** (`:L147`): `matches()` returns `False` (`:L147-L149`) — AUTO is **not** rule-evaluated here; it is handled solely by the ML classifier prediction. *(The classifier prediction path in `classifier.py` is **[inferred]**: it requires a trained model file this investigation did not fabricate; that `matches()` returns `False` for AUTO is **[observed]**.)*

**`_split_match`** (`:L155-L171`) **[observed]**: it tokenizes `match` with the regex `'"([^"]+)"|(\S+)'` (`:L165`), collapses internal whitespace (`normspace` `:L166`), `re.escape`-es each term, and turns an escaped space back into `\s+` (`:L169`) so a quoted phrase must appear as consecutive words. Probe: `'total "flight hotel"  extra'` → `['total', 'flight\s+hotel', 'extra']`.

**Algorithm constants [observed]:** `MATCH_ANY=1`, `MATCH_ALL=2`, `MATCH_LITERAL=3`, `MATCH_REGEX=4`, `MATCH_FUZZY=5`, `MATCH_AUTO=6`.

**Working together:** a **correspondent** answers *who* the document is from (one per document); a **document type** answers *what kind* it is (one per document); **tags** are cross-cutting labels (many per document), including the special `is_inbox_tag` `Inbox` flag used for triage. Q6 shows all three assigned automatically in a single consume: correspondent `ACME Corp`, type `Invoice`, tags `Inbox` + `Paid`.

---

## Observed vs Inferred — summary

| Area | Status | Basis |
|------|--------|-------|
| Q1 async convergence (`async_task` → Redis → `qcluster`) | **observed** | q1q3 probe + qcluster log |
| Q1 per-entry-point wiring (watcher / REST / mail) | inferred | file:line refs |
| Q1 barcode short-circuit branch | inferred | `tasks.py:L195-L236` (flag default `False` observed) |
| Q2 stage order + progress/message codes | **observed** | q2 probe (text parser path) |
| Q2 OCR/Tesseract PDF parsing | inferred | parser selection `consumer.py:L261` |
| Q3 engine = django-q; qcluster runs | **observed** | q3 probe + q1q3 + qcluster log |
| Q3 qcluster launched by supervisord | inferred | `docker/supervisord.conf:L28-L30` |
| Q3 four schedules + type constants | **observed** | q3 probe (real `migrate`) |
| Q4 16 metadata fields | **observed** | q4q5 probe |
| Q5 required / optional / derived | **observed** | q4q5 probe |
| Q5 filename date/title derivation | **observed** | q4q5 `FileInfo` probe |
| Q6 finished-signal organization | **observed** | q6 probe (real signal, 6 receivers) |
| Q7 `matches()` all algorithms + fuzzy score | **observed** | q7 probe |
| Q7 `MATCH_AUTO` classifier prediction | inferred | `classifier.py` (needs trained model) |

---

## Reproduction, determinism & cleanup

- **Runtime:** canonical container `paperless-app`, Python 3.9.23, dependencies at the `requirements.txt` pins, the real `paperless.settings`, Redis at `paperless-redis:6379`.
- **Steps:** export the `PAPERLESS_*` env (see the harness above); for each probe: `rm -rf /tmp/ppinv` → `mkdir` → `manage.py migrate` → run the named script capturing `.out`/`.err`. For Q1/Q3, start `manage.py qcluster` in the background first. All six scripts live under `/tmp/ppscripts/` and are reproduced in full above.
- **Two clean runs; stdout byte-identical** for every stability-sensitive value:

```text
# stdout byte-identical across the two clean runs:
  q3_schedules.out: identical
  q4q5_fields.out: identical
  q6_signal.out: identical
  q7_matching.out: identical
  q2_consume.out (excluding the mtime-derived created= line): identical
  q1q3_async.out (excluding the random django-q task_id): identical
# stable checksums confirmed across runs:
  q6 document checksum: checksum=dbd97f5b73b9094ecd35ca4298857559 == checksum=dbd97f5b73b9094ecd35ca4298857559
  q1q3 document checksum: checksum=4bd40d5fb22298f57c4813682df7f19c == checksum=4bd40d5fb22298f57c4813682df7f19c
```

- **Only run-to-run variance:** log timestamps (stderr), the mtime-derived `created=` line in q2, and django-q's random `task_id` / cluster name in q1q3 — none affects any reported fact.
- **Cleanup + read-only proof.** All investigation artifacts live under the container's `/tmp` (`/tmp/ppinv`, `/tmp/ppscripts`), **outside** the `/app` repository checkout, and were removed on completion:

```bash
rm -rf /tmp/ppinv /tmp/ppscripts          # inside the container; nothing under /app was touched
git status --porcelain                    # in the repository checkout:
 M blitzy/documentation/paperless-ngx_542221a38dff.md
```

`git status --porcelain` lists exactly one path — this document — and no source file. All 19 reference source files remain byte-for-byte unchanged; the only repository change is this single documentation file.
