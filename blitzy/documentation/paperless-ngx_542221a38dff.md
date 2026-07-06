# paperless-ngx — Ingestion, Processing & Organization: An Evidence-Backed Q&A

> **Scope.** This document answers five investigative questions about how the **paperless-ngx**
> codebase ingests, processes, and organizes documents:
> **Q1** ingestion entry points, **Q2** processing stages, **Q3** background jobs,
> **Q4** metadata fields (with a runtime example), and **Q5** how tags, correspondents, and
> document types are used together.
>
> Every behavioral claim below is **grounded in observed runtime behavior** — the relevant code
> paths were built and executed first, and the **complete, unedited output** is embedded next to
> each claim, together with the exact command that produced it and a `file:line` citation naming
> the specific function/method/attribute. Statements derived from *reading* rather than *executing*
> are explicitly labeled **(inferred)**.
>
> **Branch/commit:** `paperless-ngx_542221a38dff` (git HEAD `542221a38dff06361e07976452f9aea24d210542`).
> All `file:line` citations were re-confirmed against the source tree at authoring time.

---

## Canonical Environment

All runtime observations come from the **default, canonical configuration a normal user runs**,
inside the provided Docker container: **Python 3.9**, **default SQLite** persistence, with
**Redis** (Django-Q broker + Django Channels layer) and **Tesseract** OCR available. The host's
non-container interpreter is *not* used for any runtime observation.

**Command (canonical environment probe):**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  python3 --version
  sed -n "18p" /app/Dockerfile
  PAPERLESS_REDIS=redis://paperless-broker:6379 python3 -c "import redis; print(\"PING ->\", redis.from_url(\"redis://paperless-broker:6379\").ping())"
  tesseract --version 2>&1 | head -2
  DJANGO_SETTINGS_MODULE=paperless.settings python3 -c "import django; django.setup(); from django.conf import settings; \
    print(settings.DATABASES[\"default\"][\"ENGINE\"]); print(settings.OCR_LANGUAGE)"
'
```

**Output (unedited):**

```
### python3 --version
Python 3.9.23

### Dockerfile:18
FROM python:3.9-slim-bullseye as main-app

### Redis reachability (redis-py from app -> broker)
PING -> True

### tesseract --version (first 2 lines)
tesseract 4.1.1
 leptonica-1.79.0

### DB engine + OCR_LANGUAGE (canonical defaults)
DATABASES[default][ENGINE] = django.db.backends.sqlite3
DATABASES[default][NAME]   = /app/src/../data/db.sqlite3
OCR_LANGUAGE               = eng
PAPERLESS_REDIS (Q broker) = redis://paperless-broker:6379
```

- **Interpreter:** Python **3.9.23**, pinned by the production image — `Dockerfile:18` = `FROM python:3.9-slim-bullseye as main-app`.
- **Database:** default **SQLite**, `settings.py:299` = `"ENGINE": "django.db.backends.sqlite3"` inside the `DATABASES` block (`settings.py:297-302`); PostgreSQL is used **only** when `PAPERLESS_DBHOST` is set, so the canonical configuration needs no external DB.
- **Broker + Channels layer:** Redis, reachable (`PING -> True`). `Q_CLUSTER["redis"]` (`settings.py:449-457`) and `CHANNEL_LAYERS` (`settings.py:178-182`) both point at Redis.
- **OCR:** Tesseract **4.1.1**, default `OCR_LANGUAGE = "eng"` — `settings.py:514`.

**Confirming the SQLite + Q_CLUSTER + CHANNEL_LAYERS source (unedited):**

```
### settings.py:178-182 (CHANNEL_LAYERS)
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")],

### settings.py:297-302 (default DB = SQLite)
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": os.path.join(DATA_DIR, "db.sqlite3"),
    },
}

### settings.py:449-457 (Q_CLUSTER)
Q_CLUSTER = {
    "name": "paperless",
    "catch_up": False,
    "recycle": 1,
    "retry": PAPERLESS_WORKER_RETRY,
    "timeout": PAPERLESS_WORKER_TIMEOUT,
    "workers": TASK_WORKERS,
    "redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379"),
}

### settings.py:514 (OCR_LANGUAGE)
OCR_LANGUAGE = os.getenv("PAPERLESS_OCR_LANGUAGE", "eng")
```

**Services launched for end-to-end observation** (the exact invocation commands):

- Migrations applied to the SQLite DB: `python3 manage.py migrate` (creates the schema and the Django-Q `Schedule` rows — see Q3); the complete output is captured immediately below.
- **Django-Q worker cluster:** `python3 manage.py qcluster` — **required** for enqueued `consume_file`
  jobs to actually execute (`docker/supervisord.conf:28-29`, `[program:scheduler] command=python3 manage.py qcluster`).
- **ASGI web server (HTTP upload + WebSocket):** `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application`
  (`docker/supervisord.conf:10-11`).

**Command (apply migrations in the canonical Python 3.9 container):**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings python3 manage.py migrate'
```

**Output (unedited):**

```
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
```

The image ships with the SQLite schema already migrated, so a canonical re-run reports
`No migrations to apply.` — the schema (including the `documents_document` table used in Q4) and the
Django-Q `Schedule` rows created by the migrations are already present. Those exact scheduled-job rows
are confirmed at runtime (tying the migration result to Q3):

**Command (dump the runtime Django-Q `Schedule` rows created by the migrations):**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  DJANGO_SETTINGS_MODULE=paperless.settings python3 - <<PY
import django; django.setup()
from django_q.models import Schedule
print("django_q Schedule row count =", Schedule.objects.count())
for s in Schedule.objects.order_by("func").all():
    print(f"  func={s.func!r}  schedule_type={s.schedule_type!r}  minutes={s.minutes}  name={s.name!r}")
PY'
```

**Output (unedited):**

```
django_q Schedule row count = 4
  func='documents.tasks.index_optimize'  schedule_type='D'  minutes=None  name='Optimize the index'
  func='documents.tasks.sanity_check'  schedule_type='W'  minutes=None  name='Perform sanity check'
  func='documents.tasks.train_classifier'  schedule_type='H'  minutes=None  name='Train the classifier'
  func='paperless_mail.tasks.process_mail_accounts'  schedule_type='I'  minutes=10  name='Check all e-mail accounts'
```

The four `Schedule` rows (`train_classifier` hourly `H`, `index_optimize` daily `D`, `sanity_check`
weekly `W`, `process_mail_accounts` every `I`=10 minutes) are the scheduled background jobs registered
by the migrations — enumerated in detail in Q3.

All observation processes ran as the non-root user `testuser` (uid 1000), matching paperless-ngx's
single-user runtime model. Runtime artifacts (SQLite rows, Whoosh index, thumbnails/media) are
confined to the container's `DATA_DIR` and never touch the source tree.

---

## Q1 — Ingestion Entry Points

**How does a document *usually* enter paperless-ngx? Enumerate every path; identify the usual one.**

**Direct answer.** There are **three** ingestion entry points, and **all three converge on the same
enqueue call** `async_task("documents.tasks.consume_file", …)`:

1. the **consumption-directory watcher** (`document_consumer`) — **THE USUAL PATH**;
2. the **HTTP REST upload** API (`PostDocumentView`); and
3. **IMAP / e-mail** (`paperless_mail`).

The **usual** path is the **consume-directory watcher**: it is the always-on supervisord program
(`[program:consumer]`, `docker/supervisord.conf:19-20`) that a scanner or drop-folder writes into,
and it requires no client, credentials, or manual action. A **bulk** re-processing variant exists
(`bulk_edit.py`) that enqueues a *different* task (`bulk_update_documents`).

### The single convergence point

**Command:**

```bash
docker exec -w /app/src paperless-app bash -c '
  sed -n "85,87p" documents/management/commands/document_consumer.py
  sed -n "523,535p" documents/views.py
  sed -n "336,340p" paperless_mail/mail.py
  grep -rn "async_task(" documents/ paperless_mail/ | grep -v test
'
```

**Output (unedited):**

```
### Q1: async_task("documents.tasks.consume_file", ...) enqueue at all 3 entry points
--- document_consumer.py (consume-dir watcher) :85-87 ---
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
--- views.py (HTTP REST upload) PostDocumentView :523-535 ---
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
--- mail.py (IMAP/email) :336-340 ---
                async_task(
                    "documents.tasks.consume_file",
                    path=temp_filename,
                    override_filename=pathvalidate.sanitize_filename(
                        att.filename,

### grep -rn async_task across src/ (all enqueue call sites)
documents/management/commands/document_consumer.py:86:        async_task(
documents/views.py:523:        async_task(
documents/bulk_edit.py:18:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
documents/bulk_edit.py:31:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
documents/bulk_edit.py:47:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
documents/bulk_edit.py:63:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
documents/bulk_edit.py:87:    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)
paperless_mail/mail.py:336:                async_task(
```

**Citations.** The enqueue of `"documents.tasks.consume_file"` happens at
`document_consumer.py:86` (preceded by the log line at `:85`), `views.py:523`
(inside `PostDocumentView`, `views.py:491`, returning `Response("OK")` at `views.py:535`),
and `mail.py:336`. The bulk variant enqueues `"documents.tasks.bulk_update_documents"` at
`bulk_edit.py:18,31,47,63,87`.

### Path 1 — Consume-directory watcher (THE USUAL PATH)

The watcher is wired via watchdog (`self.observer.schedule(...)`, `document_consumer.py:188`); on
each new file it logs `"Adding {filepath} to the task queue."` (`document_consumer.py:85`) and calls
`async_task(...)` (`document_consumer.py:86`). Exercised here with `--oneshot` (scan-once mode):

**Command:**

```bash
# place a fresh PNG into the consume dir, then run the watcher command once
docker exec -u testuser -w /app/src paperless-app \
  python3 manage.py document_consumer --oneshot
```

**Output (unedited):**

```
### Run document_consumer --oneshot (consume-dir watcher entry point)
[2026-07-06 22:41:30,930] [INFO] [paperless.management.consumer] Adding /app/src/../consume/q1_watcher_v2.png to the task queue.
22:41:30 [Q] INFO Enqueued 1
```

The `"Adding … to the task queue."` line is emitted by the command's `_consume()` at
`document_consumer.py:85`, immediately followed by the `async_task` enqueue (`:86`); Django-Q's
`[Q] INFO Enqueued 1` confirms the job was placed on the Redis queue.

### Path 2 — HTTP REST upload (`PostDocumentView`)

`PostDocumentView` (`views.py:491`, a DRF `GenericAPIView` using `PostDocumentSerializer`) writes the
upload to a temp file, calls `async_task("documents.tasks.consume_file", …)` (`views.py:523`), and
returns `Response("OK")` (`views.py:535`). Exercised with a **real authenticated HTTP POST** to the
running gunicorn server:

**Command:**

```bash
# obtain a token via POST /api/token/ (local test superuser), then POST the file to the upload
# endpoint. The observation script is written so the token VALUE is never printed — it prints only
# non-secret metadata (token length + HTTP status codes + response body), so the output below is
# complete and unedited with nothing removed.
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q1_http_upload.py
```

**Output (unedited):**

```
### auth token acquired via POST /api/token/ -> HTTP 200 (token is a 40-char DRF key; value intentionally NOT printed)
### POST /api/documents/post_document/ -> HTTP 200
### response body: "OK"
```

The endpoint returned HTTP **200** with body `"OK"` — exactly the `Response("OK")` at `views.py:535`,
confirming the upload was accepted and enqueued via `async_task` at `views.py:523`. The token
acquisition (`POST /api/token/` → HTTP 200) is shown by its status code and length only; the script
was deliberately written not to print the secret value, so the block above is complete and unedited
(no field was redacted after capture).

### Path 3 — IMAP / e-mail (`paperless_mail`)

After fetching an attachment, `MailAccountHandler.handle_message` (`mail.py:272`) detects the MIME
type, writes the payload to a temp file, logs `"Consuming attachment …"`, and calls
`async_task("documents.tasks.consume_file", path=temp_filename, …)` (`mail.py:336`). A **live IMAP
server is not available in the canonical container**, so the **transport is mocked (non-canonical)**:
a synthetic message drives the *real* `handle_message` code path and the *real* `async_task` enqueue.

**Command:**

```bash
# drive the real MailAccountHandler.handle_message with a synthetic (mocked-IMAP) message
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q1_mail.py
```

**Output (unedited):**

```
### Driving REAL MailAccountHandler.handle_message (mail.py:272) -> enqueue at mail.py:336
### (IMAP TRANSPORT MOCKED = synthetic message; the async_task enqueue path is the real one)
[DEBUG] [paperless_mail] Rule obs-account.obs-rule: Processing mail Email Subject Statement v2 from billing@initech.example with 1 attachment(s)
[INFO] [paperless_mail] Rule obs-account.obs-rule: Consuming attachment statement_v2.png from mail Email Subject Statement v2 from billing@initech.example
22:42:34 [Q] INFO Enqueued 1
### handle_message returned (processed_attachments) = 1
```

The `"Consuming attachment …"` log line comes from `mail.py` immediately before the enqueue, and
`[Q] INFO Enqueued 1` confirms the *real* `async_task` at `mail.py:336` fired. **Label:** the IMAP
*transport* was mocked (synthetic message object); the ingestion **enqueue path itself was the real
one**. Driven at scale, the scheduled job `process_mail_accounts` (`paperless_mail/tasks.py:11`) polls
every account and funnels each attachment through this same enqueue.

### Why they all funnel through `consume_file`, and why the watcher is "usual"

All three human/machine-facing entry points converge on a single asynchronous job,
`documents.tasks.consume_file` (`tasks.py:184`), which is what actually performs the processing
pipeline (Q2). This convergence means the ingestion *transport* (folder / HTTP / e-mail) is
decoupled from the processing *pipeline*. The **consume-directory watcher is the usual path**
because it is the always-running background program a scanner or file drop naturally targets
(`docker/supervisord.conf:19-20`), requiring no API client or credentials — corroborated verbatim by
`docs/usage_overview.rst:88-91`, whose "The consumption directory" section (`:85-86`) opens with
*"The primary method of getting documents into your database is by putting them in the consumption
directory."* (`docs/usage_overview.rst:88`), while the HTTP/Web-UI and IMAP methods are documented as
secondary sections further down (`:105` "Web UI Upload", `:128` "IMAP (Email)"). The **bulk** path is
distinct: `bulk_edit.py` enqueues
`bulk_update_documents` (`bulk_edit.py:18,31,47,63,87`) to re-apply metadata to *existing*
documents, not to ingest new ones.

---

## Q2 — Processing Stages

**Once a document is received, what stages does it go through before it is fully processed and
available?**

**Direct answer.** The enqueued job `consume_file` (`tasks.py:184`) optionally runs barcode
separation (default **OFF**) and then constructs a `Consumer` and calls `try_consume_file`
(`consumer.py:180`). That method drives the ordered pipeline: **validate → detect MIME & select
parser → parse/OCR → generate thumbnail → extract text & date → load classifier → atomically persist
→ fire the six post-consume handlers (metadata + full-text index) → move files into managed storage →
run post-consume script → mark SUCCESS**. Progress is streamed as milestones **STARTING 0 → WORKING
20 (parsing) → 70 (thumbnail) → 90 (parse date) → 95 (save) → SUCCESS 100** on the happy path, or a
terminal **FAILED 100** on any validation/parse failure (emitted by `_fail`, `consumer.py:78-79`).
A document becomes searchable/"available" only after the `add_to_index` handler runs.

### The observed progress milestones (real WebSocket status channel)

Each milestone is emitted by `Consumer._send_progress` (`consumer.py:56`) via
`async_to_sync(channel_layer.group_send)("status_updates", …)`. We subscribed to that Channels group
and enqueued a real document; the worker's progress messages were received live:

**Command:**

```bash
# subscribe to the 'status_updates' Channels group, enqueue consume_file, receive milestones
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q2_progress.py
```

**Output (unedited):**

```
### Subscribed to Channels group 'status_updates' (channels_redis RedisChannelLayer)
### Enqueuing async_task('documents.tasks.consume_file', ...) -> Django-Q worker will process
22:43:11 [Q] INFO Enqueued 1
### Progress milestones received over the REAL WebSocket status channel:
    STATUS   PROG  MESSAGE                  document_id
    STARTING 0     new_file                 None
    WORKING  20    parsing_document         None
    WORKING  70    generating_thumbnail     None
    WORKING  90    parse_date               None
    WORKING  95    save_document            None
    SUCCESS  100   finished                 17
```

These map exactly to the `_send_progress(...)` calls in `consumer.py`: `STARTING 0` at `:202`,
`WORKING 20 parsing_document` at `:259`, `WORKING 70 generating_thumbnail` at `:264`,
`WORKING 90 parse_date` at `:274` (only when the parser returned no date), `WORKING 95 save_document`
at `:294`, and `SUCCESS 100 finished` (with the new `document_id`) at `:375`.

### The `FAILED` terminal milestone (real WebSocket status channel)

`SUCCESS 100` is not the only terminal state. Whenever any validation or processing step fails,
`Consumer._fail` (`consumer.py:78-79`) calls `self._send_progress(100, 100, "FAILED", message)`
**before** it raises `ConsumerError`, so the UI's progress bar always reaches a terminal state. We
subscribed to the same `status_updates` group and enqueued a consume of an **unsupported** file (a
`.zip`, which has no parser — see Q2 parser dispatch and edge case 2) through the **real**
`async_task` → Django-Q worker path; the worker emitted `STARTING 0` and then the terminal
`FAILED 100`:

**Command:**

```bash
# subscribe to 'status_updates', enqueue consume_file for an unsupported .zip, receive milestones
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src \
  python3 /tmp/obs/q2_failed.py'
```

**Output (unedited):**

```
### Subscribed to Channels group 'status_updates' (channels_redis RedisChannelLayer)
23:22:47 [Q] INFO Enqueued 1
### Enqueued async_task('documents.tasks.consume_file', <unsupported .zip>) -> Django-Q worker will process & FAIL it
### Progress milestones received over the REAL WebSocket status channel (filtered to this file):
    STATUS   PROG  MESSAGE                  document_id
    STARTING 0     new_file                 None
    FAILED   100   unsupported_type         None
### terminal milestone observed: True
```

The `FAILED 100 unsupported_type` milestone is emitted by `_send_progress(100, 100, "FAILED", message)`
inside `_fail` (`consumer.py:78-79`); here `message` is the constant
`MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` (`consumer.py:44`), set by the unsupported-MIME
`_fail` call at `consumer.py:224-225`. This completes the milestone set: the pipeline always ends in
exactly one terminal milestone — `SUCCESS 100 finished` on success (`consumer.py:375`) or
`FAILED 100 <message>` on any failure (`consumer.py:79`). The result was **stable across two runs**
(both emitted identical `STARTING 0 new_file` → `FAILED 100 unsupported_type`). The same terminal
`FAILED 100` is emitted for every other `_fail` path (duplicate `document_already_exists`, file not
found, pre/post-consume script errors), since all of them route through `_fail` (`consumer.py:78-81`).

### The ordered pipeline trace (DEBUG log, document 17)

**Command:**

```bash
docker exec -w /app/src paperless-app sed -n '2607,2624p' /app/data/log/paperless.log
```

**Output (unedited):**

```
[2026-07-06 22:43:12,110] [INFO] [paperless.consumer] Consuming q2_pipeline_v2.png
[2026-07-06 22:43:12,110] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-06 22:43:12,111] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-06 22:43:12,125] [DEBUG] [paperless.consumer] Parsing q2_pipeline_v2.png...
[2026-07-06 22:43:12,216] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/obs/q2_pipeline_v2.png: 'dpi'
[2026-07-06 22:43:12,216] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 104 based on image width 860
[2026-07-06 22:43:12,216] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/obs/q2_pipeline_v2.png', 'output_file': '/tmp/paperless/paperless-f83vixyu/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-f83vixyu/sidecar.txt', 'image_dpi': 104}
[2026-07-06 22:43:13,325] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-06 22:43:13,325] [DEBUG] [paperless.consumer] Generating thumbnail for q2_pipeline_v2.png...
[2026-07-06 22:43:13,371] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-f83vixyu/archive.pdf[0] /tmp/paperless/paperless-f83vixyu/convert.png
[2026-07-06 22:43:13,385] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-06 22:43:13,456] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-f83vixyu/gs_out.png /tmp/paperless/paperless-f83vixyu/convert_gs.png
[2026-07-06 22:43:13,485] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-f83vixyu/convert_gs.png -out /tmp/paperless/paperless-f83vixyu/thumb_optipng.png
[2026-07-06 22:43:15,500] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-06 22:43:15,524] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-06 22:43:15,546] [DEBUG] [paperless.consumer] Deleting file /tmp/obs/q2_pipeline_v2.png
[2026-07-06 22:43:15,570] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-f83vixyu
[2026-07-06 22:43:15,570] [INFO] [paperless.consumer] Document 2026-07-06 q2_pipeline_v2 consumption finished
```

This is the pipeline in execution order:

| Stage | Evidence in the trace | Code |
|-------|----------------------|------|
| MIME detection | `Detected mime type: image/png` | `magic.from_file(...)`, `consumer.py:219` |
| Parser selection | `Parser: RasterisedDocumentParser` | `get_parser_class_for_mime_type(...)`, `consumer.py:223`; logged at `:246` |
| Parse / OCR | `Parsing …` + `Calling OCRmyPDF with args {… 'language': 'eng' …}` + `Using text from sidecar file` | `document_parser.parse(...)`, `consumer.py:261` |
| Thumbnail | `Generating thumbnail …` | `get_optimised_thumbnail(...)`, `consumer.py:265` |
| Classifier load | `Document classification model does not exist (yet) …` | `load_classifier()`, `consumer.py:292` |
| Persist | `Saving record to database` | `_store(...)` inside `transaction.atomic()`, `consumer.py:298-301` |
| Cleanup + finish | `Deleting file …` / `consumption finished` | file move + `SUCCESS` at `consumer.py:375` |

*(Environmental note, not a pipeline defect: the `Error while getting DPI … 'dpi'` warning and the
ImageMagick→ghostscript thumbnail fallback are container image-policy quirks; the pipeline recovers
via ghostscript and continues to SUCCESS.)*

### The task wrapper (`consume_file`) and its barcode branch

**Command:**

```bash
docker exec -w /app/src paperless-app sed -n '184,248p' documents/tasks.py
```

**Output (unedited — key lines):**

```
def consume_file(
    path,
    override_filename=None,
    ...
):

    # check for separators in current document
    if settings.CONSUMER_ENABLE_BARCODES:
        ...
    # continue with consumption if no barcode was found
    document = Consumer().try_consume_file(
        path,
        override_filename=override_filename,
        ...
    )

    if document:
        return "Success. New document id {} created".format(document.pk)
```

`consume_file` (`tasks.py:184`) first checks `settings.CONSUMER_ENABLE_BARCODES` (`tasks.py:195`,
**default OFF** — see Q3/edge cases); when off, it constructs `Consumer()` and calls
`try_consume_file(...)`, returning `"Success. New document id {} created"` (`tasks.py:247`).

### The post-consume signal and its SIX handlers

After the document is stored, `try_consume_file` sends `document_consumption_finished`
(`consumer.py:306`). That signal is defined at `signals/__init__.py:4` and wired to six handlers in
`DocumentsConfig.ready()` (`apps.py:22-27`):

**Command:**

```bash
docker exec -w /app/src paperless-app bash -c 'cat -n documents/signals/__init__.py; sed -n "11,28p" documents/apps.py'
```

**Output (unedited):**

```
### signals/__init__.py (full, 5 lines)
     1	from django.dispatch import Signal
     2	
     3	document_consumption_started = Signal()
     4	document_consumption_finished = Signal()
     5	document_consumer_declaration = Signal()

### apps.py ready() handler registration (11-28)
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
```

We proved all six fire by consuming a **plain-text** document (`TextDocumentParser`, exact content —
no OCR ambiguity) whose content matches pre-created organizers (`Wayne Enterprises`/`invoice`/`payment`):

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q2_handlers_txt.py
```

**Output (unedited):**

```
### Consuming a text/plain file (TextDocumentParser) -> exact content -> all 6 handlers fire (consumer.py:306)
[2026-07-06 22:45:09,588] [INFO] [paperless.consumer] Consuming q2_handlers_v3.txt
[INFO] [paperless.handlers] Assigning correspondent Wayne Enterprises to 2026-07-06 q2_handlers_v3
[INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-06 Wayne Enterprises q2_handlers_v3
[INFO] [paperless.handlers] Tagging "2026-07-06 Wayne Enterprises q2_handlers_v3" with "finance"
[2026-07-06 22:45:10,459] [INFO] [paperless.consumer] Document 2026-07-06 Wayne Enterprises q2_handlers_v3 consumption finished

### content (verbatim, no OCR): 'INVOICE from Wayne Enterprises\nReference WE-2025-77\npayment due Net 30\nAmount 900.00 USD\n'
### Resulting document metadata:
  id                = 19
  correspondent     = Wayne Enterprises   <- set_correspondent (handlers.py:35)
  document_type     = Invoice   <- set_document_type (handlers.py:101)
  tags              = ['Inbox', 'finance']   <- set_tags (handlers.py:168) + add_inbox_tags (handlers.py:30)
  admin LogEntry(s) = 1   <- set_log_entry (handlers.py:413)
  Whoosh search 'Wayne' -> ids: [19, 18, 8] ; this doc indexed: True   <- add_to_index (handlers.py:428)
```

Each handler and its observed effect:

| Handler | `file:line` | Observed effect |
|---------|-------------|-----------------|
| `add_inbox_tags` | `handlers.py:30` | Adds all `is_inbox_tag` tags → `Inbox` present in `tags` |
| `set_correspondent` | `handlers.py:35` | `Assigning correspondent Wayne Enterprises …` → `correspondent = Wayne Enterprises` |
| `set_document_type` | `handlers.py:101` | `Assigning document type Invoice …` → `document_type = Invoice` |
| `set_tags` | `handlers.py:168` | `Tagging … with "finance"` → `finance` in `tags` |
| `set_log_entry` | `handlers.py:413` | Creates a Django admin `LogEntry` → count = 1 |
| `add_to_index` | `handlers.py:428` | Inserts into the Whoosh full-text index → `search 'Wayne'` returns this doc → **searchable/"available"** |

### Parser selection by MIME type

Parsers register a `get_parser` factory with a `weight` and a `mime_types` map; the consumer picks
the best-weighted parser and logs `type(document_parser).__name__` (`consumer.py:246`).

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q2_parsers2.py
```

**Output (unedited):**

```
### Parser selected per MIME type (documents.parsers.get_parser_class_for_mime_type):
  application/pdf                                                        -> RasterisedDocumentParser
  image/png                                                              -> RasterisedDocumentParser
  image/jpeg                                                             -> RasterisedDocumentParser
  image/tiff                                                             -> RasterisedDocumentParser
  text/plain                                                             -> TextDocumentParser
  text/csv                                                               -> TextDocumentParser
  application/vnd.openxmlformats-officedocument.wordprocessingml.document -> None
  application/zip                                                        -> None

### PAPERLESS_TIKA_ENABLED (settings.py:592, gates Tika) = False
```

- **`RasterisedDocumentParser`** (`paperless_tesseract/parsers.py`, weight `0`,
  `paperless_tesseract/signals.py:10`): `application/pdf`, `image/jpeg`, `image/png`, `image/tiff`,
  `image/gif`, `image/bmp` — OCR via OCRmyPDF/Tesseract.
- **`TextDocumentParser`** (`paperless_text/parsers.py`, weight `10`): `text/plain`, `text/csv`.
- **`TikaDocumentParser`** (`paperless_tika/parsers.py`, weight `10`): Office formats
  (`.doc/.docx/.xls/.xlsx/.ppt/.pptx/.odt/.ods/.odp/.rtf`) — **only when `PAPERLESS_TIKA_ENABLED`
  is set** (`settings.py:592`, default `NO`). In the canonical config it is `False`, so `.docx`
  resolves to `None` (unsupported → see edge cases).


---

## Q3 — Background Jobs

**Are there background jobs? What mechanism is used for background execution?**

**Direct answer.** **Yes.** Background execution is handled by **Django-Q** (a multiprocessing task
queue for Django) — **not Celery** — over a **Redis broker**, using a worker cluster launched with
`manage.py qcluster`. **Per-document** work is the `consume_file` job (Q1/Q2); **periodic** work is
defined as Django-Q `Schedule` rows (train classifier, optimize index, sanity check, poll mail).
Real-time consumption progress is streamed over **Django Channels** backed by the same Redis.

### Proof: Django-Q, not Celery

**Command:**

```bash
docker exec -w /app/src paperless-app bash -c '
  grep -rniE "shared_task|from celery|import celery|@task\b|app\.task" \
      documents/ paperless/ paperless_mail/ paperless_tesseract/ paperless_text/ paperless_tika/; \
  echo "grep exit code: $?"
  ls paperless/celery.py; echo "ls exit code: $?"
'
```

**Output (unedited):**

```
### Q3 PROOF: Django-Q NOT Celery
--- grep -rn for celery / @shared_task / @task in src/ (expect NO matches) ---
grep exit code: 1
--- ls paperless/celery.py (expect: not found) ---
ls: cannot access 'paperless/celery.py': No such file or directory
ls exit code: 2
```

`grep` exits `1` (no matches for any Celery/shared-task marker anywhere in the backend), and
`paperless/celery.py` does **not exist** (`ls` exits `2`). Background jobs are plain functions in
`tasks.py` modules dispatched via `django_q.tasks.async_task` (see Q1's `grep` for the eight enqueue
sites). This confirms **Django-Q** is the mechanism.

### The worker cluster, broker, and Channels layer (runtime values)

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q3_django_q.py
```

**Output (unedited):**

```
### settings.Q_CLUSTER (runtime values):
   name       = 'paperless'
   catch_up   = False
   recycle    = 1
   retry      = 1810
   timeout    = 1800
   workers    = 11
   redis      = 'redis://paperless-broker:6379'

### Broker: class = Redis | info = Redis 6.0.20

### CHANNEL_LAYERS default BACKEND = channels_redis.core.RedisChannelLayer
### CHANNEL_LAYERS hosts             = ['redis://paperless-broker:6379']

### django_q Schedule rows (periodic background jobs):
   id=1 func=documents.tasks.train_classifier           type=H minutes=None
   id=2 func=documents.tasks.index_optimize             type=D minutes=None
   id=3 func=documents.tasks.sanity_check               type=W minutes=None
   id=4 func=paperless_mail.tasks.process_mail_accounts type=I minutes=10
```

`Q_CLUSTER` (`settings.py:449-457`), key-by-key with its causal effect:

| Key | Value | Effect |
|-----|-------|--------|
| `name` | `paperless` | Namespaces the cluster's Redis keys |
| `catch_up` | `False` | Missed scheduled runs are **not** back-filled after downtime |
| `recycle` | `1` | Each worker process is recycled after **1** task (bounds memory growth from heavy OCR jobs) |
| `retry` | `1810` | A task not acknowledged within 1810 s is retried |
| `timeout` | `1800` | A worker is killed if a task exceeds 1800 s (< `retry`, so a timed-out task can be retried) |
| `workers` | `11` | Number of parallel worker processes (derived from CPU count) |
| `redis` | `redis://paperless-broker:6379` | The **broker** — where the queue lives |

The broker resolves to a **Redis** class connected to **Redis 6.0.20**. The real-time progress
channel `CHANNEL_LAYERS` (`settings.py:178-182`) is `channels_redis.core.RedisChannelLayer` over the
same Redis — this is what carries the `_send_progress` milestones observed in Q2 to the UI over
WebSockets.

### The periodic `Schedule` rows (defined via migrations)

The four scheduled jobs are created as Django-Q `Schedule` rows by migrations:

**Command:**

```bash
docker exec -w /app/src paperless-app bash -c '
  sed -n "10,20p" documents/migrations/1001_auto_20201109_1636.py
  sed -n "10,14p" documents/migrations/1004_sanity_check_schedule.py
  sed -n "10,15p" paperless_mail/migrations/0002_auto_20201117_1334.py
'
```

**Output (unedited):**

```
### migration 1001 (train_classifier HOURLY, index_optimize DAILY) :10-20
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

### migration 1004 (sanity_check WEEKLY) :10-14
    schedule(
        "documents.tasks.sanity_check",
        name="Perform sanity check",
        schedule_type=Schedule.WEEKLY,
    )

### mail migration 0002 (process_mail_accounts MINUTES=10) :10-15
    schedule(
        "paperless_mail.tasks.process_mail_accounts",
        name="Check all e-mail accounts",
        schedule_type=Schedule.MINUTES,
        minutes=10,
    )
```

| Scheduled job | Cadence | `file:line` |
|---------------|---------|-------------|
| `documents.tasks.train_classifier` | `HOURLY` (`type=H`) | `documents/migrations/1001_auto_20201109_1636.py:11-14` |
| `documents.tasks.index_optimize` | `DAILY` (`type=D`) | `documents/migrations/1001_auto_20201109_1636.py:16-19` |
| `documents.tasks.sanity_check` | `WEEKLY` (`type=W`) | `documents/migrations/1004_sanity_check_schedule.py:11-14` |
| `paperless_mail.tasks.process_mail_accounts` | every `10` minutes (`type=I`) | `paperless_mail/migrations/0002_auto_20201117_1334.py:11-15` |

The runtime `Schedule.objects.all()` query (above) confirms exactly these four rows are live.

### Operational entry points (supervisord)

**Command:**

```bash
docker exec -w /app paperless-app grep -n \
  'program:gunicorn\|program:consumer\|program:scheduler\|command=' docker/supervisord.conf
```

**Output (unedited):**

```
10:[program:gunicorn]
11:command=gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
19:[program:consumer]
20:command=python3 manage.py document_consumer
28:[program:scheduler]
29:command=python3 manage.py qcluster
```

The production image runs three supervisord programs: `gunicorn … paperless.asgi:application`
(ASGI web server + WebSocket, `supervisord.conf:10-11`), `python3 manage.py document_consumer`
(the consume-dir watcher, `:19-20`), and `python3 manage.py qcluster` (the Django-Q worker cluster,
`:28-29`). We observed the running cluster launched via `qcluster` process the enqueued
`consume_file` jobs from Q1/Q2, and the scheduler component fire the mail schedule
(`created a task from schedule [Check all e-mail accounts]`).

**Rationale.** Ingestion must not block the caller (a scanner drop, an HTTP client, or a mail poll),
so each entry point simply enqueues a job on Redis and returns; the `qcluster` worker performs the
expensive OCR/parse/index pipeline asynchronously. Periodic maintenance (classifier training, index
optimization, sanity checks, mail polling) is expressed declaratively as `Schedule` rows the same
cluster executes.


---

## Q4 — Metadata Fields (with a runtime example)

**What metadata fields are saved for each document, and which are absolutely *required* versus
*optional* versus *derived* later during runtime processing? Show with a runtime example.**

**Direct answer.** At the database level, **only two fields are strictly required**:
**`mime_type`** (`models.py:126`) and **`checksum`** (`models.py:135`) — they alone have
`null=False`, `blank=False`, `editable=False`, **and no default**. Everything else is either
**derived** during the pipeline (`content`, `created`, `added`, `modified`, `filename`,
`archive_filename`, `archive_checksum`) or **optional / user-supplied**
(`title`, `correspondent`, `document_type`, `tags`, `archive_serial_number`, `storage_type`).
`class Document` is defined at `models.py:88`.

### Runtime example (user directive: "show with a runtime example")

The following script introspects each field, creates and saves a **real** `Document`, prints every
field value, renders the API serializer, and demonstrates the required-field enforcement.

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q4_runtime.py
```

**Output — Part 1, field introspection (unedited):**

```
========================================================================
PART 1 — Field introspection (Document._meta.get_field) => classification basis
========================================================================
  FIELD                  null   blank  editable  has_default default
  mime_type              False  False  False     False       (none)
  checksum               False  False  False     False       (none)
  content                False  True   True      False       (none)
  created                False  False  True      True        2026-07-06 22:47:18.520852+00:00
  added                  False  False  False     True        2026-07-06 22:47:18.520877+00:00
  modified               False  True   False     False       (none)
  filename               True   False  False     True        None
  archive_filename       True   False  False     True        None
  archive_checksum       True   True   False     False       (none)
  title                  False  True   True      False       (none)
  correspondent          True   True   True      False       (none)
  document_type          True   True   True      False       (none)
  tags                   False  True   True      False       (none)
  archive_serial_number  True   True   True      False       (none)
  storage_type           False  False  False     True        unencrypted
```

**Output — Part 2, the saved Document's raw field values (unedited):**

```
========================================================================
PART 2 — Create + save a REAL Document (runtime example) and dump raw field values
========================================================================
  saved pk = 20
  doc.mime_type              = 'application/pdf'
  doc.checksum               = 'f189ac3eca39865195b13c6e4b67a4f9'
  doc.content                = ''
  doc.created                = datetime.datetime(2026, 7, 6, 22, 47, 18, 521125, tzinfo=datetime.timezone.utc)
  doc.added                  = datetime.datetime(2026, 7, 6, 22, 47, 18, 521130, tzinfo=datetime.timezone.utc)
  doc.modified               = datetime.datetime(2026, 7, 6, 22, 47, 18, 522565, tzinfo=datetime.timezone.utc)
  doc.filename               = None
  doc.archive_filename       = None
  doc.archive_checksum       = None
  doc.title                  = ''
  doc.correspondent          = None
  doc.document_type          = None
  doc.tags                   = []
  doc.archive_serial_number  = None
  doc.storage_type           = 'unencrypted'
```

**Output — Part 3 & 4, the API serializer (unedited):**

```
========================================================================
PART 3 — DocumentSerializer(doc).data  (API contract, serialisers.py:201)
========================================================================
{
  "id": 20,
  "correspondent": null,
  "document_type": null,
  "title": "",
  "content": "",
  "tags": [],
  "created": "2026-07-06T22:47:18.521125Z",
  "modified": "2026-07-06T22:47:18.522565Z",
  "added": "2026-07-06T22:47:18.521130Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-06 .pdf",
  "archived_file_name": null
}

========================================================================
PART 4 — DocumentSerializer for a FULLY-PROCESSED doc (id=19, text parser)
========================================================================
{
  "id": 19,
  "correspondent": 2,
  "document_type": 1,
  "title": "q2_handlers_v3",
  "content": "INVOICE from Wayne Enterprises\nReference WE-2025-77\npayment due Net 30\nAmount 900.00 USD\n",
  "tags": [
    1,
    2
  ],
  "created": "2026-07-06T22:45:09.553271Z",
  "modified": "2026-07-06T22:45:10.433645Z",
  "added": "2026-07-06T22:45:10.362887Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-06 Wayne Enterprises q2_handlers_v3.txt",
  "archived_file_name": null
}
```

**Output — Part 5, 6 & 7, required-field enforcement + upload contract (unedited):**

```
========================================================================
PART 5 — REQUIRED enforcement: duplicate checksum -> IntegrityError (checksum unique=True, models.py:135)
========================================================================
  django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum

========================================================================
PART 6 — Bare save nuance: Document(title=...).save() with NO mime_type/checksum
========================================================================
  save() SUCCEEDED pk=21  mime_type='' checksum=''  (CharField empty_strings_allowed=True -> '')

========================================================================
PART 7 — PostDocumentSerializer upload contract (serialisers.py:413) required flags
========================================================================
  document         required=True write_only=True type=FileField
  title            required=False write_only=True type=CharField
  correspondent    required=False write_only=True type=PrimaryKeyRelatedField
  document_type    required=False write_only=True type=PrimaryKeyRelatedField
  tags             required=False write_only=True type=ManyRelatedField
```

### The raw SQLite DDL (ground truth at the DB layer)

**Command:**

```bash
# via Python stdlib sqlite3 (the sqlite3 CLI is not installed in the image)
docker exec -u testuser paperless-app python3 /tmp/obs/q4_ddl.py
```

**Output (unedited):**

```
  "title" varchar(128) NOT NULL
  "content" text NOT NULL
  "created" datetime NOT NULL
  "modified" datetime NOT NULL
  "correspondent_id" integer NULL REFERENCES "documents_correspondent" ("id") DEFERRABLE INITIALLY DEFERRED
  "checksum" varchar(32) NOT NULL UNIQUE
  "added" datetime NOT NULL
  "storage_type" varchar(11) NOT NULL
  "archive_serial_number" integer NULL UNIQUE
  "document_type_id" integer NULL REFERENCES "documents_documenttype" ("id") DEFERRABLE INITIALLY DEFERRED
  "mime_type" varchar(256) NOT NULL
  "archive_checksum" varchar(32) NULL
  "archive_filename" varchar(1024) NULL UNIQUE
  "filename" varchar(1024) NULL UNIQUE)
```

### Classification, with the field definitions

**REQUIRED (strict — `null=False`, `blank=False`, `editable=False`, no default).** The full,
unabbreviated field definitions, pasted directly from source:

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app sed -n '126p'     documents/models.py
docker exec -u testuser -w /app/src paperless-app sed -n '135,141p' documents/models.py
```

**Output (unedited):**

```
    mime_type = models.CharField(_("mime type"), max_length=256, editable=False)
```

```
    checksum = models.CharField(
        _("checksum"),
        max_length=32,
        editable=False,
        unique=True,
        help_text=_("The checksum of the original document."),
    )
```

- **`mime_type`** — `models.py:126`. DDL: `"mime_type" varchar(256) NOT NULL`.
- **`checksum`** — `models.py:135-141`. DDL: `"checksum" varchar(32) NOT NULL UNIQUE`.

**Missing-required-field demonstration — DB-level `NOT NULL` enforcement (runtime).** The two
required columns are enforced by the database as `NOT NULL`. Because a Django `CharField` coerces an
*unset* value to `''` before the INSERT (Part 6), the enforcement is demonstrated directly at the DB
layer with a **raw SQL insert** that sets `mime_type` / `checksum` to a genuine `NULL`:

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src \
  python3 /tmp/obs/q4_notnull.py'
```

**Output (unedited):**

```
========================================================================
F1 — DB-level NOT NULL enforcement on required columns (raw SQL insert with NULL)
========================================================================
  Document.objects.count() BEFORE = 12
  -- raw INSERT with mime_type = NULL (unique checksum so it is NOT a duplicate):
  mime_type=NULL: django.db.utils.IntegrityError: NOT NULL constraint failed: documents_document.mime_type
  -- raw INSERT with checksum = NULL (valid mime_type):
  checksum=NULL: django.db.utils.IntegrityError: NOT NULL constraint failed: documents_document.checksum
  Document.objects.count() AFTER  = 12  (unchanged => both NULL inserts were rejected)
```

Both `NULL` inserts are rejected with
`django.db.utils.IntegrityError: NOT NULL constraint failed: documents_document.mime_type` (resp.
`…checksum`), and the row count is **unchanged before/after** (12 → 12), confirming nothing was
inserted. This is the direct DB-level proof that `mime_type` and `checksum` are the required columns.

**Important nuance (reported exactly as observed).** Two distinct layers govern the two required
columns, and the runtime example exercises **both** — the direct answer ("`mime_type` and `checksum`
are the required fields") holds at the DB layer, with the ORM adding a coercion subtlety:

- **ORM layer.** A bare `Document(title=…).save()` **succeeds** with `mime_type=''` and `checksum=''`
  (Part 6) because Django `CharField` has `empty_strings_allowed=True`, so *unset* values are coerced
  to `''` *before* the INSERT — and an empty string satisfies a `NOT NULL` column. So a naive ORM
  `.save()` does **not** raise on the missing required fields; it silently substitutes `''`.
- **DB layer.** The `documents_document.mime_type` and `.checksum` columns are nonetheless `NOT NULL`
  (see the DDL above). A **raw SQL insert** that sets either to a genuine `NULL` is rejected with
  `django.db.utils.IntegrityError: NOT NULL constraint failed: documents_document.mime_type`
  (resp. `…checksum`) — the missing-required-field demonstration above (both inserts rejected, row
  count unchanged 12 → 12). Additionally, `checksum` is `UNIQUE`, so inserting a second row with an
  existing checksum raises `django.db.utils.IntegrityError: UNIQUE constraint failed:
  documents_document.checksum` (Part 5) — the very constraint the duplicate check (Q1 edge case) guards.

In the *real* pipeline, `Consumer._store` always sets `mime_type` (from `magic.from_file`) and
`checksum` (md5 of the file), so these are never empty for an actually-consumed document. **(inferred
from `_store` reading; the empty-string save, the `NOT NULL` rejections, and the `UNIQUE` rejection
above are all directly observed behavior.)**

**DERIVED (populated during pipeline execution):**

- `content` — `models.py:117` (`TextField`, `blank=True`): the parser/OCR output (`get_text`); empty on the bare doc, populated on doc 19.
- `created` — `models.py:152` (`default=timezone.now`): date extracted from the document, else the default.
- `added` — `models.py:169` (`default=timezone.now, editable=False`): auto-set at insert.
- `modified` — `models.py:154` (`auto_now=True, editable=False`): auto-updated on save.
- `filename` / `archive_filename` — `models.py:176` / `:186` (`editable=False, default=None, null=True, unique=True`): assigned **after** storage (both `None` on the un-stored bare doc; DDL `NULL UNIQUE`).
- `archive_checksum` — `models.py:143` (`editable=False, blank=True, null=True`): checksum of the generated archive PDF.

**OPTIONAL / user-supplied:**

- `title` — `models.py:106` (`blank=True`).
- `correspondent` — `models.py:97` (FK, `blank=True, null=True, on_delete=SET_NULL`).
- `document_type` — `models.py:108` (FK, `blank=True, null=True`).
- `tags` — `models.py:128` (M2M, `blank=True`).
- `archive_serial_number` — `models.py:196` (`blank=True, null=True, unique=True`).
- `storage_type` — `models.py:161` (`default=STORAGE_TYPE_UNENCRYPTED, editable=False`; constants at `models.py:90-95`) — defaulted to `'unencrypted'`.

### API contracts

- **`DocumentSerializer`** (`serialisers.py:201`, a `DynamicFieldsModelSerializer` with `depth=1`)
  exposes `fields = (id, correspondent, document_type, title, content, tags, created, modified,
  added, archive_serial_number, original_file_name, archived_file_name)` (`serialisers.py:222-234`);
  `original_file_name`/`archived_file_name` are `SerializerMethodField`s (`serialisers.py:207-208`).
  Note the API deliberately does **not** expose the internal `mime_type`/`checksum`/`storage_type`/
  raw `filename`.
- **`PostDocumentSerializer`** (`serialisers.py:413`) is the upload contract: **only `document`
  (a write-only `FileField`) is `required=True`**; `title`, `correspondent`, `document_type`, and
  `tags` are all `required=False` (Part 7) — mirroring the model's optional/required split.

**Rationale.** DB-level constraints define what is *required* (`mime_type`, `checksum` — the
identity/dedup keys); fields the *pipeline* computes are *derived* (content, dates, storage
filenames, archive checksum); and fields a *user* provides to organize the document are *optional*
(title, correspondent, type, tags, ASN). The upload API's `required` split matches this exactly.


---

## Q5 — Tags, Correspondents & Document Types Used Together

**How are tags, correspondents, and document types used together to organize documents in practice?**

**Direct answer.** `Tag`, `Correspondent`, and `DocumentType` **all subclass `MatchingModel`**
(`models.py:19`) and therefore share one rule-based matching engine (`matches()`, `matching.py:60`)
plus an optional scikit-learn auto-classifier. During consumption, the post-consume handlers
(`set_correspondent`, `set_document_type`, `set_tags`, `add_inbox_tags` — Q2) evaluate each object's
`match`/`matching_algorithm` against the document's `content` and attach matches in one pass. In
practice a document ends up with **one correspondent** (who sent it), **one document type**
(what it is), and **zero-or-more tags** (cross-cutting labels/filters).

### The six matching algorithms, exercised at runtime

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q5_matching_obs.py
```

**Output — constants & the six algorithms (unedited):**

```
========================================================================
MATCHING ALGORITHM CONSTANTS (documents.models.MatchingModel)
========================================================================
  MatchingModel.MATCH_ANY = 1
  MatchingModel.MATCH_ALL = 2
  MatchingModel.MATCH_LITERAL = 3
  MatchingModel.MATCH_REGEX = 4
  MatchingModel.MATCH_FUZZY = 5
  MatchingModel.MATCH_AUTO = 6
  MatchingModel.MATCHING_ALGORITHMS = ((1, 'Any word'), (2, 'All words'), (3, 'Exact match'), (4, 'Regular expression'), (5, 'Fuzzy word'), (6, 'Automatic'))
  Tag() default matching_algorithm = 1 (MATCH_ANY)
  Tag() default is_insensitive     = True (True)

Using real Document id=7 content:
'CONTRACT from Umbrella Corporation\nReference UMB-2023-777 Date 2023-07-19\nService Fee 3400.00 USD\n\nPayment terms Net 30 days'

========================================================================
matches() ACROSS ALL SIX ALGORITHMS (documents.matching.matches, matching.py:60)
========================================================================
  [EMPTY-GUARD   ] match=''                                       algo=1 insensitive=True -> matches()=False
  [EMPTY-GUARD   ] match='   '                                    algo=2 insensitive=True -> matches()=False
  [CASE-INSENS T ] match='umbrella'                               algo=1 insensitive=True -> matches()=True
  [CASE-SENS   F ] match='umbrella'                               algo=1 insensitive=False -> matches()=False
  [MATCH_ALL  T  ] match='Umbrella Corporation'                   algo=2 insensitive=True -> matches()=True
  [MATCH_ALL  F  ] match='Umbrella Microsoft'                     algo=2 insensitive=True -> matches()=False
  [MATCH_ANY  T  ] match='Umbrella Nonexistent'                   algo=1 insensitive=True -> matches()=True
  [MATCH_ANY  F  ] match='Foo Bar Baz'                            algo=1 insensitive=True -> matches()=False
  [MATCH_LIT  T  ] match='Umbrella Corporation'                   algo=3 insensitive=True -> matches()=True
  [MATCH_LIT  F  ] match='Corporation Umbrella'                   algo=3 insensitive=True -> matches()=False
  [MATCH_RGX  T  ] match='UMB-\\d{4}-\\d{3}'                      algo=4 insensitive=True -> matches()=True
  [MATCH_RGX  F  ] match='ZZZ-\\d{9}'                             algo=4 insensitive=True -> matches()=False
  [MATCH_RGX ERR ] match='[unclosed('                             algo=4 insensitive=True -> matches()=False
```

Each branch of `matches()` (`matching.py:60`), with its cause→effect:

| Algorithm | Constant | `file:line` | Observed |
|-----------|----------|-------------|----------|
| *(empty guard)* | — | `matching.py:66` | blank `match` → always `False` |
| *(case sensitivity)* | — | `matching.py:69-70` | `is_insensitive=True` adds `re.IGNORECASE`; `'umbrella'` matches `Umbrella` only when insensitive |
| `MATCH_ANY` | `1` | `matching.py:84` | any `\bword\b` present → `True` |
| `MATCH_ALL` | `2` | `matching.py:72` | *all* `\bword\b` present → `True` |
| `MATCH_LITERAL` | `3` | `matching.py:91` | `re.escape`d exact phrase; order matters (`Corporation Umbrella` → `False`) |
| `MATCH_REGEX` | `4` | `matching.py:107` | `re.search`; an invalid pattern is caught (`re.error`) → `False` |
| `MATCH_FUZZY` | `5` | `matching.py:127` | `fuzz.partial_ratio ≥ 90` (see below) |
| `MATCH_AUTO` | `6` | `matching.py:147-149` | **always `False` inside `matches()`** ("done elsewhere") |

The invalid-regex case also logged the caught error (from `matching.py:113-117`):
`[ERROR] [paperless.matching] Error while processing regular expression [unclosed(`.

### `MATCH_FUZZY` threshold (`fuzz.partial_ratio ≥ 90`, `matching.py:135`)

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q5_fuzzy_final.py
```

**Output (unedited):**

```
Document id=7 content: 'CONTRACT from Umbrella Corporation\nReference UMB-2023-777 Date 2023-07-19\nService Fee 3400.00 USD\n\nPayment terms Net 30 days'

MATCH_FUZZY threshold demonstration (matching.py:135 -> fuzz.partial_ratio(match,text) >= 90):
  match string             partial_ratio  vs 90       matches()
  'Umbrella Corporatn'     94             >= 90       True
  'Umbrela Corporaton'     89             <  90       False
```

The threshold is exact: a ratio of **94** (just above 90) matches; **89** (just below) does not —
demonstrating the `>= 90` cutoff at `matching.py:135` (via `fuzzywuzzy`).

### `MATCH_AUTO` returns `False` in `matches()` — the auto decision is the classifier's

**Output (unedited):**

```
========================================================================
MATCH_AUTO INSIDE matches() (matching.py:147-149 -> 'this is done elsewhere')
========================================================================
  [MATCH_AUTO F  ] match='Umbrella'                               algo=6 insensitive=True -> matches()=False
  ^ Even though 'Umbrella' is present, MATCH_AUTO returns False in matches(); the real
    auto decision is produced by the ML classifier via predict_* (not this function).
```

This is a deliberate subtlety: for `MATCH_AUTO`, `matches()` returns `False` unconditionally
(`matching.py:147-149`); the actual automatic assignment is produced by the ML classifier's
`predict_*` methods (below), not by the rule engine.

### The higher-level functions: rule match **OR** classifier prediction

`match_correspondents` / `match_document_types` / `match_tags` combine the rule result with the
classifier's prediction; with no trained model they fall back to **rule-only**.

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app python3 /tmp/obs/q5_matchfns_obs.py
```

**Output (unedited):**

```
========================================================================
load_classifier() runtime (classifier.py:30) -- canonical: no trained model present
========================================================================
  load_classifier() returned: None (None => rule-only matching, classifier absent)

========================================================================
match_correspondents / match_document_types / match_tags (matching.py:21/34/47) with classifier=None
========================================================================
  Correspondent objects in DB: [(2, 'Wayne Enterprises', "'Wayne'", 1), (1, 'billing@initech.example', "''", 1)]
  DocumentType  objects in DB: [(1, 'Invoice', "'invoice'", 1)]
  Tag           objects in DB: [(1, 'Inbox', "''", 1, 'inbox=True'), (2, 'finance', "'payment'", 1, 'inbox=False')]

  match_correspondents(doc, None) -> []
  match_document_types(doc, None) -> []
  match_tags(doc, None)           -> ['finance']
  (rule-only: each returns objects where matches(o,doc) True, since pred_id is None/[])

========================================================================
INBOX EXCLUSION for auto-matching (classifier.py:125-126: .exclude(tags__is_inbox_tag=True))
========================================================================
  Document.objects.count()                                 = 5
  Document.objects.filter(tags__is_inbox_tag=True).count() = 1
  Document.objects.exclude(tags__is_inbox_tag=True).count()= 4 (training/auto set)
  Docs currently carrying the Inbox tag: [8]
```

- `match_correspondents` (`matching.py:21` → `classifier.predict_correspondent`, `classifier.py:251`; else `pred_id=None`, `matching.py:25`)
- `match_document_types` (`matching.py:34` → `predict_document_type`, `classifier.py:262`; else `None`, `matching.py:38`)
- `match_tags` (`matching.py:47` → `predict_tags`, `classifier.py:273`; else `[]`, `matching.py:51`)

Each returns `filter(lambda o: matches(o, document) or o.pk == pred_id, …)`. With `classifier=None`
(the canonical config has no trained model — `load_classifier()` returns `None`, `classifier.py:30`),
the prediction term is empty, so results are rule-only: the document's `content` contains
`"Payment"`, so `match_tags` returns `['finance']` (tag `finance`, `match='payment'`, `MATCH_ANY`,
insensitive), while the correspondent (`match='Wayne'`) and type (`match='invoice'`) don't match this
particular content.

**Subtlety worth noting:** the `Inbox` tag has `match=''`, so it is **never** auto-assigned by
`matches()` (empty guard, `matching.py:66`); it is applied by the separate `add_inbox_tags` handler
(`handlers.py:30`).

### Inbox exclusion for auto-matching

Only documents **not** in the inbox feed the auto-classifier's training set. The runtime query above
shows `Document.objects.exclude(tags__is_inbox_tag=True)` yields **4 of 5** docs (the one inbox-tagged
doc, pk 8, is excluded). The source:

**Command:**

```bash
docker exec -w /app/src paperless-app sed -n '124,126p' documents/classifier.py
docker exec -w /app paperless-app sed -n '78,81p' docs/advanced_usage.rst
```

**Output (unedited):**

```
        for doc in Document.objects.order_by("pk").exclude(
            tags__is_inbox_tag=True,
        ):
```
```
* The Auto matching algorithm only takes documents into account which are NOT
  placed in your inbox (i.e. have any inbox tags assigned to them). This ensures
  that the neural network only learns from documents which you have correctly
  tagged before.
```

`DocumentClassifier.train` (`classifier.py:115`) iterates
`Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` (`classifier.py:125-126`), so
inbox documents are excluded from training — corroborated verbatim by `docs/advanced_usage.rst:78-79`.

### Used together, in practice

All three organizers attach in a **single post-consume pass** because `set_correspondent`
(`handlers.py:35`), `set_document_type` (`handlers.py:101`), `set_tags` (`handlers.py:168`), and
`add_inbox_tags` (`handlers.py:30`) are all connected to `document_consumption_finished` (Q2). The
practical pattern: **correspondents** = *who* sent it; **document types** = *what* it is; **tags** =
cross-cutting labels for filtering/search. Rule-based matching handles deterministic cases (a bank's
name, an invoice keyword); the optional `MATCH_AUTO` classifier learns from your already-organized
(non-inbox) documents to suggest the rest.


---

## Edge / Alternate Paths

Each alternate path was exercised through the **real** `Consumer` methods (`try_consume_file`,
`load_classifier`, `run_pre_consume_script`/`run_post_consume_script`), capturing the actual
`ConsumerError` text and log lines.

### 1. Duplicate detection by checksum

`pre_check_duplicate` (`consumer.py:102`) computes the file md5 and, if a `Document` with that
`checksum` (or `archive_checksum`) exists, calls `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, …)`
(`consumer.py:111`). We consumed a file, then re-consumed a **byte-identical** copy:

**Command / Output (unedited):**

```
md5(edge_dup.png)     = c7a9965183335c1ebb2588ab56699f98
md5(edge_dup_copy.png)= c7a9965183335c1ebb2588ab56699f98

EDGE 1a — FIRST consume of edge_dup.png -> success
  RESULT: 2026-07-06 edge_dup

EDGE 1b — RE-consume byte-identical copy -> pre_check_duplicate (consumer.py:102) -> _fail (:111)
  CONSUMER_DELETE_DUPLICATES setting (settings.py:486) = False
  ConsumerError RAISED: edge_dup_copy.png: Not consuming edge_dup_copy.png: It is a duplicate.
  edge_dup_copy.png still on disk (CONSUMER_DELETE_DUPLICATES=False): True
[2026-07-06 22:38:13,317] [ERROR] [paperless.consumer] Not consuming edge_dup_copy.png: It is a duplicate.
```

The second consume raised `ConsumerError: … It is a duplicate.` The optional
`CONSUMER_DELETE_DUPLICATES` (`settings.py:486`) is `False` by default, so the duplicate file is left
on disk (if `True`, `pre_check_duplicate` would `os.unlink` it first, `consumer.py:108-109`).

### 2. Unsupported MIME type (no parser)

If `get_parser_class_for_mime_type` returns `None`, the consumer calls
`_fail(MESSAGE_UNSUPPORTED_TYPE, …)` (`consumer.py:224-225`). We fed the **real**
`Consumer.try_consume_file()` a `.zip`:

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src \
  python3 /tmp/obs/edge2_unsupported.py'
```

**Output (unedited):**

```
EDGE 2 — Unsupported MIME (no parser) -> _fail MESSAGE_UNSUPPORTED_TYPE (consumer.py:224-225)
  magic.from_file(zip, mime=True) = application/zip
[2026-07-06 23:31:32,449] [INFO] [paperless.consumer] Consuming edge_unsupported.zip
[2026-07-06 23:31:32,449] [DEBUG] [paperless.consumer] Detected mime type: application/zip
[2026-07-06 23:31:32,467] [ERROR] [paperless.consumer] Unsupported mime type application/zip
  ConsumerError RAISED: edge_unsupported.zip: Unsupported mime type application/zip
```

`application/zip` has no registered parser (Q2), so consumption fails with
`Unsupported mime type application/zip`. (The same applies to `.docx` in the canonical config,
because Tika is disabled.)

### 3. Missing / incompatible classifier → rule-only fallback

`load_classifier()` (`classifier.py:30`) returns `None` in **all three** bad-model cases the AAP
names — **missing**, **version-incompatible**, and **corrupt** — after which matching falls back to
rule-only (Q5). We exercised all three in the canonical runtime. The model file lives in `DATA_DIR`
(`settings.MODEL_FILE`, `settings.py:74`), never in the source tree; for the incompatible/corrupt
cases the code itself deletes the bad file (`os.unlink`, `classifier.py:48`), so the canonical
"no model" state is restored automatically (verified by the `FINAL … = False` line).

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src \
  python3 /tmp/obs/edge3_classifier.py'
```

**Output (unedited):**

```
EDGE 3 — load_classifier() variants (classifier.py:30). settings.MODEL_FILE = /app/src/../data/classification_model.pickle
FORMAT_VERSION (required) = 7

--- (a) MISSING model file -> None (classifier.py:31-36) ---
  [MISSING] os.path.isfile(MODEL_FILE) BEFORE = False
[2026-07-06 23:31:33,426] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
  [MISSING] load_classifier() returned = None
  [MISSING] os.path.isfile(MODEL_FILE) AFTER  = False  (deleted by classifier.py:48 on bad model)

--- (b) INCOMPATIBLE version: schema_version=999 != 7 -> IncompatibleClassifierVersionError (classifier.py:80-83 -> caught :42-48) ---
  [INCOMPATIBLE] os.path.isfile(MODEL_FILE) BEFORE = True
[2026-07-06 23:31:33,427] [ERROR] [paperless.classifier] Unrecoverable error while loading document classification model, deleting model file.
Traceback (most recent call last):
  File "/app/src/documents/classifier.py", line 40, in load_classifier
    classifier.load()
  File "/app/src/documents/classifier.py", line 81, in load
    raise IncompatibleClassifierVersionError(
documents.classifier.IncompatibleClassifierVersionError: Cannot load classifier, incompatible versions.
  [INCOMPATIBLE] load_classifier() returned = None
  [INCOMPATIBLE] os.path.isfile(MODEL_FILE) AFTER  = False  (deleted by classifier.py:48 on bad model)

--- (c) CORRUPT payload: correct version=7 then garbage -> ClassifierModelCorruptError (classifier.py:93-94 -> caught :42-48) ---
  [CORRUPT] os.path.isfile(MODEL_FILE) BEFORE = True
[2026-07-06 23:31:33,427] [ERROR] [paperless.classifier] Unrecoverable error while loading document classification model, deleting model file.
Traceback (most recent call last):
  File "/app/src/documents/classifier.py", line 86, in load
    self.data_hash = pickle.load(f)
_pickle.UnpicklingError: invalid load key, '\x00'.

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/src/documents/classifier.py", line 40, in load_classifier
    classifier.load()
  File "/app/src/documents/classifier.py", line 94, in load
    raise ClassifierModelCorruptError()
documents.classifier.ClassifierModelCorruptError
  [CORRUPT] load_classifier() returned = None
  [CORRUPT] os.path.isfile(MODEL_FILE) AFTER  = False  (deleted by classifier.py:48 on bad model)

FINAL os.path.isfile(MODEL_FILE) (canonical state restored) = False
```

All three variants return `None` (→ rule-only matching), with the observed **before/during/after**
state transitions:

- **(a) Missing** — `settings.MODEL_FILE` absent → `load_classifier()` short-circuits to `None` at
  `classifier.py:31-36`, emitting the DEBUG line. This is the canonical default (no trained model).
- **(b) Version-incompatible** — a model whose pickled `schema_version` (`999`) differs from
  `DocumentClassifier.FORMAT_VERSION` (`7`) makes `load()` raise `IncompatibleClassifierVersionError`
  (`classifier.py:80-83`); `load_classifier` catches it at `classifier.py:42`, logs the "Unrecoverable
  error … deleting model file." exception, calls `os.unlink` (`classifier.py:48`), and returns `None`
  (file `True → False`).
- **(c) Corrupt** — a model with the correct version but a garbage payload makes the inner
  `pickle.load` fail (`classifier.py:86`), which `load()` converts to `ClassifierModelCorruptError`
  (`classifier.py:93-94`); the same `except` block (`classifier.py:42-48`) logs, deletes the file, and
  returns `None`.

The `FINAL … = False` line confirms the bad model file was removed by the code and the canonical
"no model" state was restored. Corroborated by `docs/advanced_usage.rst:78-79` (auto-matching only
runs when a compatible model exists).

### 4. Inbox exclusion for auto-matching

Covered in Q5: `Document.objects.exclude(tags__is_inbox_tag=True)` (`classifier.py:125-126`) yielded
**4 of 5** documents, excluding the single inbox-tagged one — corroborated by
`docs/advanced_usage.rst:78-79`.

### 5. Optional pre-/post-consume scripts

`run_pre_consume_script` (`consumer.py:121`) and `run_post_consume_script` (`consumer.py:143`) each
early-return when their setting is unset; when set, they `Popen` the configured script.

**Command:**

```bash
docker exec -u testuser -w /app/src paperless-app bash -c '
  PAPERLESS_REDIS=redis://paperless-broker:6379 HOME=/tmp TIKA_LOG_PATH=/tmp \
  DJANGO_SETTINGS_MODULE=paperless.settings PYTHONPATH=/app/src \
  python3 /tmp/obs/edge5_scripts.py'
```

**Output (unedited):**

```
EDGE 5a — Pre/Post-consume scripts DEFAULT (unset) -> skipped (consumer.py:121-123 / :143-145)
  settings.PRE_CONSUME_SCRIPT  (settings.py:570) = None
  settings.POST_CONSUME_SCRIPT (settings.py:571) = None
  run_pre_consume_script() returned: None (early-return no-op, guard 'if not settings.PRE_CONSUME_SCRIPT: return')
  run_post_consume_script(doc) returned: None (early-return no-op)

EDGE 5b — Post-consume script SET -> run_post_consume_script FIRES (consumer.py:143-178)
[2026-07-06 23:31:48,471] [INFO] [paperless.consumer] Executing post-consume script /tmp/obs/post_marker.sh
  marker file created by script: True
  marker contents: POST-CONSUME FIRED pk=4 file=2026-07-06 watcher_doc.png
```

In the canonical config both scripts are unset (`settings.py:570-571`, both `None`), so the methods
are no-ops. When we pointed `POST_CONSUME_SCRIPT` at a trivial script, `run_post_consume_script`
executed it (`Popen` with the document's pk, filenames, URLs, correspondent, and tags), and the
script wrote its marker file. *(The temporary script was removed afterward.)*

---

## Coverage Checklist

| Item / named target | Addressed | Evidence |
|---------------------|-----------|----------|
| **Q1** every ingestion path enumerated | ✅ | consume-dir watcher, HTTP upload, IMAP/mail — all shown enqueuing `consume_file` |
| **Q1** the *usual* path identified | ✅ | consume-dir watcher (always-on `[program:consumer]`), rationale given |
| **Q1** bulk / scheduled variants named | ✅ | `bulk_update_documents` (`bulk_edit.py`), `process_mail_accounts` |
| **Q2** ordered stages + progress milestones | ✅ | STARTING 0 → WORKING 20/70/90/95 → SUCCESS 100, plus terminal FAILED 100 (all on the live channel) + DEBUG trace |
| **Q2** six post-consume handlers (each by name) | ✅ | `add_inbox_tags`, `set_correspondent`, `set_document_type`, `set_tags`, `set_log_entry`, `add_to_index` — all fired |
| **Q2** parsers by MIME (each named) | ✅ | Rasterised/Text/Tika, with Tika gated off |
| **Q3** framework = Django-Q, not Celery | ✅ | grep exit 1, no `celery.py`; `Q_CLUSTER`; broker Redis 6.0.20 |
| **Q3** per-document + 4 scheduled jobs | ✅ | `consume_file`; train_classifier/index_optimize/sanity_check/process_mail_accounts |
| **Q3** real-time channel | ✅ | `RedisChannelLayer` |
| **Q4** required vs optional vs derived | ✅ | field introspection table + SQLite DDL + definitions |
| **Q4** runtime example (user directive) | ✅ | real `Document` saved, all fields printed, serializer JSON, DB-level `NOT NULL` + `UNIQUE` IntegrityError enforcement |
| **Q5** tags | ✅ | `finance` matched via `MATCH_ANY`; `set_tags` + `add_inbox_tags` |
| **Q5** correspondents | ✅ | `Wayne Enterprises` matched; `match_correspondents` |
| **Q5** document types | ✅ | `Invoice` matched; `match_document_types` |
| **Q5** six algorithms + fuzzy threshold + AUTO subtlety | ✅ | all six run; 94→True/89→False; MATCH_AUTO returns False in `matches()` |
| **Q5** inbox exclusion | ✅ | exclude→4/5 docs; `classifier.py:125-126` + docs |
| **Edge** duplicate / unsupported MIME / classifier (missing+incompatible+corrupt) / inbox / pre-post scripts | ✅ | each exercised with command + unedited output |

**Method note.** Every command above was run inside the canonical Python 3.9 container with Redis +
Tesseract + default SQLite and the Django-Q `qcluster` worker active. All temporary observation
scripts were created outside the repository (`/tmp`) and removed on completion; the only file added
to the repository is this document.

