# How Documents Flow Through paperless-ngx

> **Scope & provenance.** This is an evidence-based technical explainer of how documents move through **paperless-ngx** on branch `paperless-ngx_542221a38dff` (HEAD commit `542221a38dff06361e07976452f9aea24d210542`). Every behavioral claim below was produced by **actually building and running the system first**, then pasting the verbatim observed output next to the claim. The runtime was the **default canonical configuration**: **Python 3.9.25**, the **Redis** broker at `redis://localhost:6379`, and the default **SQLite** database (`/opt/paperless/data/db.sqlite3`). The exact commands used are listed in the [Environment & commands appendix](#7--environment--commands-appendix).
>
> **Conventions.** Code citations use `path:line` and resolve at commit `542221a38dff`. Observed output is shown in fenced blocks with the command that produced it. Statements derived only from reading source (not run) are explicitly marked **(inferred from reading)**. Values obtained from anything other than the real canonical code path are explicitly marked **non-canonical**.

---

## Table of contents

1. [Document-flow overview](#1--document-flow-overview)
2. [Q1 — How a document enters paperless-ngx (ingestion entry points)](#2--q1--how-a-document-enters-paperless-ngx-ingestion-entry-points)
3. [Q2 — Processing stages & background execution](#3--q2--processing-stages--background-execution)
4. [Q3 — Per-document metadata (required vs optional vs derived)](#4--q3--per-document-metadata-required-vs-optional-vs-derived)
5. [Q4 — Practical organization with tags, correspondents & document types](#5--q4--practical-organization-with-tags-correspondents--document-types)
6. [Coverage-pass checklist](#6--coverage-pass-checklist)
7. [Environment & commands appendix](#7--environment--commands-appendix)

---

## 1 · Document-flow overview

The single most important structural fact is **convergence**: paperless-ngx does *not* have three independent ingestion pipelines. It has **one** pipeline, fed by **three** different sources. Every real entry point ends by enqueuing the **same** background task by its dotted-path string:

```python
async_task("documents.tasks.consume_file", ...)
```

- Watched consumption folder → `src/documents/management/commands/document_consumer.py:86-87`
- REST upload → `src/documents/views.py:523-524`
- IMAP e-mail attachment → `src/paperless_mail/mail.py:336-337`

That task, `consume_file` `[src/documents/tasks.py:184]`, runs on the **Django-Q** worker cluster and calls `Consumer().try_consume_file(...)` `[src/documents/consumer.py:180]`, which performs the ordered processing stages (duplicate check → parser dispatch → OCR/text extraction → metadata derivation → **atomic** DB persistence → file storage). Persisting the row fires the `document_consumption_finished` signal `[src/documents/signals/__init__.py:4]` **inside** the same atomic block, which triggers the auto-organization handlers (correspondent, document type, tags, inbox tags) and the full-text index update.

I proved the convergence at runtime: three documents were created through the three distinct entry points, and all three produced a Django-Q `Task` row whose `func` is `documents.tasks.consume_file` (full evidence in [§2](#2--q1--how-a-document-enters-paperless-ngx-ingestion-entry-points)):

```text
func=documents.tasks.consume_file | task_name='folder_invoice.txt'  | success=True | result='Success. New document id 1 created'
func=documents.tasks.consume_file | task_name='rest_invoice.txt'    | success=True | result='Success. New document id 2 created'
func=documents.tasks.consume_file | task_name='email_invoice.txt'   | success=True | result='Success. New document id 3 created'
```

### Unified flow diagram

```mermaid
flowchart TD
    A["Watched consumption folder<br/>document_consumer.py:86-87<br/>(THE USUAL PATH)"] --> Q
    B["REST upload POST /api/documents/post_document/<br/>views.py:523-524"] --> Q
    C["IMAP e-mail attachment<br/>paperless_mail/mail.py:336-337"] --> Q
    Q["async_task('documents.tasks.consume_file', ...)<br/>enqueued on the Django-Q Redis broker"] --> W
    W["Django-Q worker cluster (manage.py qcluster)<br/>consume_file  tasks.py:184"] --> D
    D["Consumer.try_consume_file()  consumer.py:180"] --> E["Pre-checks: file exists / directories / duplicate checksum"]
    E --> F["Parser dispatch by MIME type  consumer.py:219-223"]
    F --> G["parse() → OCR / text extraction  consumer.py:261"]
    G --> H["thumbnail + get_text + get_date  consumer.py:265-272"]
    H --> I["load_classifier()  consumer.py:292"]
    I --> J["with transaction.atomic():  consumer.py:298"]
    J --> K["_store() → Document.objects.create()  consumer.py:379"]
    K --> L["document_consumption_finished.send()  consumer.py:306<br/>(fires INSIDE the atomic block)"]
    L --> M["Auto-organize (signal handlers, apps.py:22-27):<br/>add_inbox_tags · set_correspondent · set_document_type · set_tags"]
    M --> N["set_log_entry + add_to_index → Whoosh  handlers.py:413,428"]
    J --> O["_write() stores files + thumbnail  consumer.py:429"]

    S["Scheduler (same qcluster process)"] -. "HOURLY" .-> S1["train_classifier  tasks.py:48"]
    S -. "DAILY" .-> S2["index_optimize  tasks.py:32"]
    S -. "WEEKLY" .-> S3["sanity_check  tasks.py:255"]
```

The rest of this document answers each of the four question groups in detail, with the pasted runtime evidence for every claim.

---

## 2 · Q1 — How a document enters paperless-ngx (ingestion entry points)

**Short answer.** A new document *usually* enters through the **watched consumption folder** — you drop a file into a directory and paperless picks it up automatically. There are **three** canonical entry points in total (folder, REST upload, IMAP e-mail), and **all three converge on the single background task `documents.tasks.consume_file`**. There is also a fourth, **non-canonical** enqueue path (`bulk_edit.py`) that is *not* fresh ingestion.

### 2.1 The usual path — watched consumption folder

This is the typical way documents enter: a long-lived watcher process (`manage.py document_consumer`) observes a directory and enqueues each new file. The watcher startup was observed verbatim:

```text
# consumer.log (from: python manage.py document_consumer)
[2026-07-02 23:18:29,902] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /opt/paperless/consume
```

> Note: in this canonical runtime the watcher uses **inotify**, not polling, because `PAPERLESS_CONSUMER_POLLING` is `0`. The `PollingObserver(timeout=settings.CONSUMER_POLLING)` `[src/documents/management/commands/document_consumer.py:187]` branch is the fallback used only when polling is enabled.

Dropping a file triggers the enqueue. Command and observed output:

```bash
cp /tmp/obs/samples/folder_invoice.txt /opt/paperless/consume/folder_invoice.txt
```

```text
# consumer.log — the enqueue line, logged immediately before async_task(...)
[2026-07-02 23:21:35,201] [INFO] [paperless.management.consumer] Adding /opt/paperless/consume/folder_invoice.txt to the task queue.
```

That log line is emitted by `logger.info(f"Adding {filepath} to the task queue.")` `[src/documents/management/commands/document_consumer.py:85]`, immediately followed by `async_task(` `[:86]` with `"documents.tasks.consume_file"` `[:87]`. The worker then picks it up:

```text
# qcluster.log
23:21:35 [Q] INFO Enqueued 1
[2026-07-02 23:21:35,333] [INFO] [paperless.consumer] Consuming folder_invoice.txt
[2026-07-02 23:21:35,936] [INFO] [paperless.consumer] Document 2026-07-02 folder_invoice consumption finished
23:21:35 [Q] INFO Processed [folder_invoice.txt]
```

Resulting Django-Q task row:

```text
func=documents.tasks.consume_file | task_name='folder_invoice.txt' | success=True | result='Success. New document id 1 created'
```

### 2.2 REST upload — `POST /api/documents/post_document/`

The HTTP upload endpoint is `class PostDocumentView(GenericAPIView)` `[src/documents/views.py:491]`, whose `def post(self, request, ...)` `[:497]` enqueues `async_task(` `[:523]` with `"documents.tasks.consume_file"` `[:524]`. It requires authentication (`permission_classes = (IsAuthenticated,)` `[:493]`).

Command and observed response (token redacted per secret-handling policy):

```bash
curl -F "document=@/tmp/obs/samples/rest_invoice.txt" \
     -H "Authorization: Token <redacted>" \
     http://localhost:8000/api/documents/post_document/
```

```text
"OK"
HTTP_STATUS:200
```

The body is the literal string `"OK"`. This is exactly what the view returns: `return Response("OK")` `[src/documents/views.py:535]`. (The endpoint returns `"OK"`, **not** the async task id.) The uploaded payload is validated by `class PostDocumentSerializer` `[src/documents/serialisers.py:413]` in `def validate_document` `[:450]`, which detects the MIME type via `magic.from_buffer(document_data, mime=True)` `[:452]`.

Without authentication the endpoint refuses the upload (confirming the `IsAuthenticated` permission):

```text
{"detail":"Authentication credentials were not provided."}
HTTP_STATUS:401
```

Worker pickup and task row:

```text
# qcluster.log
[2026-07-02 23:22:22,032] [INFO] [paperless.consumer] Consuming rest_invoice.txt
[2026-07-02 23:22:22,...]  [INFO] [paperless.consumer] Document 2026-07-02 rest_invoice consumption finished
23:22:22 [Q] INFO Processed [rest_invoice.txt]
```

```text
func=documents.tasks.consume_file | task_name='rest_invoice.txt' | success=True | result='Success. New document id 2 created'
```

### 2.3 IMAP e-mail attachment

The e-mail path is `class MailAccountHandler(LoggingMixin)` `[src/paperless_mail/mail.py:104]`; `handle_mail_account` `[:151]` iterates a mailbox and `handle_message` `[:272]` processes each message, detecting attachment MIME via `magic.from_buffer(att.payload, mime=True)` `[:317]` and enqueuing `async_task(` `[:336]` with `"documents.tasks.consume_file"` `[:337]`. E-mail accounts/rules are configured by `MailAccount` / `MailRule` `[src/paperless_mail/models.py]`.

> **Transport labeled non-canonical.** A live IMAP server is not available in-sandbox, so the network transport was stubbed with a synthetic message object. **However, the real enqueue code path was exercised** — I drove the actual `MailAccountHandler().handle_message(msg, rule)` method (rule configured `assign_title_from=FROM_SUBJECT`, `assign_correspondent_from=FROM_NOTHING`, `attachment_type=ATTACHMENTS_ONLY`). Only the IMAP fetch/transport is synthetic; the code that builds and enqueues the task is genuine.

Observed output:

```text
[2026-07-02 23:23:35,575] [INFO] [paperless_mail] Rule obs-probe-account.obs-probe-rule: Consuming attachment email_invoice.txt from mail Email Invoice Test from billing@acme.example
HANDLE_MESSAGE_RETURN = 1
```

```text
# qcluster.log
[2026-07-02 23:23:35,704] [INFO] [paperless.consumer] Consuming email_invoice.txt
[2026-07-02 23:23:35,...]  [INFO] [paperless.consumer] Document 2026-07-02 Email Invoice Test consumption finished
23:23:36 [Q] INFO Processed [email_invoice.txt]
```

```text
func=documents.tasks.consume_file | task_name='email_invoice.txt' | success=True | result='Success. New document id 3 created'
```

The title `Email Invoice Test` came from the e-mail subject because the rule used `assign_title_from = FROM_SUBJECT`. (The temporary `MailAccount`/`MailRule` used to drive the handler were deleted afterward.)

### 2.4 Convergence proof

All three entry points produced Django-Q `Task` rows with the identical `func`, `documents.tasks.consume_file`, each returning the success string from `return "Success. New document id {} created".format(document.pk)` `[src/documents/tasks.py:247]`:

```text
func=documents.tasks.consume_file | task_name='folder_invoice.txt' | success=True | result='Success. New document id 1 created'
func=documents.tasks.consume_file | task_name='rest_invoice.txt'   | success=True | result='Success. New document id 2 created'
func=documents.tasks.consume_file | task_name='email_invoice.txt'  | success=True | result='Success. New document id 3 created'
```

### 2.5 Non-canonical enqueue path (for completeness)

`src/documents/bulk_edit.py` enqueues **`documents.tasks.bulk_update_documents`** (at lines `18`, `31`, `47`, `63`, `87`) — used for *reprocessing / bulk editing existing documents*, not for fresh ingestion. It does **not** call `consume_file`, so it is **not** a document entry point. It is listed here only to be exhaustive.

**Q1 summary:** usual = watched folder; all three canonical entry points (folder / REST / IMAP) converge on `documents.tasks.consume_file`; `bulk_update_documents` is a separate, non-ingestion path.

---

## 3 · Q2 — Processing stages & background execution

**Short answer.** Once enqueued, a file is processed by the background task `consume_file` `[src/documents/tasks.py:184]`, which runs `Consumer.try_consume_file()` `[src/documents/consumer.py:180]`. The ordered stages are: pre-checks (existence, directories, duplicate checksum) → parser dispatch by MIME → parse (OCR/text) → thumbnail + text + date → load classifier → **atomic DB persist** → `document_consumption_finished` signal (fires *inside* the atomic block) → auto-organize + full-text index → file storage. Background execution is **Django-Q** (a multiprocessing task queue), run as `manage.py qcluster`, using **Redis** as the broker. Three periodic jobs are scheduled: **train classifier (hourly), optimize index (daily), sanity check (weekly)**.

### 3.1 The ordered pipeline stages (observed)

To surface the internal stage sequence I ran the canonical `consume_file` synchronously with the `paperless` logger raised to `DEBUG`. This does **not** alter the pipeline — it only makes the pipeline's *existing* stage log lines visible. (The async convergence itself was already proven in [§2](#24--convergence-proof); this foreground run is purely to expose the ordered DEBUG lines.) Command:

```bash
python manage.py shell < /tmp/obs/pipeline_trace.py   # calls documents.tasks.consume_file(<file>)
```

Observed ordered output (document id 4 created), each line mapped to its source stage:

```text
[INFO]  [paperless.consumer] Consuming pipeline_sample.txt
[DEBUG] [paperless.consumer] Detected mime type: text/plain
[DEBUG] [paperless.consumer] Parser: TextDocumentParser
[DEBUG] [paperless.consumer] Parsing pipeline_sample.txt...
[DEBUG] [paperless.consumer] Generating thumbnail for pipeline_sample.txt...
[DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 ...
[DEBUG] [paperless.consumer] Document classification model does not exist (yet), not performing automatic matching.
[DEBUG] [paperless.consumer] Saving record to database
[DEBUG] [paperless.consumer] Deleting file /opt/paperless/tmp/pipeline_sample.txt
[INFO]  [paperless.consumer] Document 2026-07-02 pipeline_sample consumption finished
=== CONSUME_FILE RESULT: 'Success. New document id 4 created'
```

| # | Stage | Observed line | Source anchor |
|---|-------|---------------|---------------|
| 1 | Enter pipeline / progress `STARTING` | `Consuming pipeline_sample.txt` | `try_consume_file` `[consumer.py:180]`; `Consuming {filename}` `[consumer.py:215]`; progress `[consumer.py:202]` |
| 2 | Pre-checks (exists / dirs / **duplicate checksum**) | *(no error → passed)* | `pre_check_file_exists` `[:211]`, `pre_check_directories` `[:212]`, `pre_check_duplicate` `[:213]` |
| 3 | MIME detection | `Detected mime type: text/plain` | `magic.from_file(...)` `[consumer.py:219]` |
| 4 | Parser dispatch by MIME | `Parser: TextDocumentParser` | `get_parser_class_for_mime_type(mime_type)` `[consumer.py:223]` (see `src/documents/parsers.py`) |
| 5 | Parse (OCR / text extraction) | `Parsing pipeline_sample.txt...` | `document_parser.parse(...)` `[consumer.py:261]` |
| 6 | Thumbnail generation | `Generating thumbnail for ...` / `optipng ...` | `get_optimised_thumbnail(...)` `[consumer.py:265]` |
| 7 | Extract text & date | *(text/date read)* | `get_text()` `[:271]`, `get_date()` `[:272]`, `parse_date` fallback `[:275]` |
| 8 | Load ML classifier | `Document classification model does not exist (yet)...` | `load_classifier()` `[consumer.py:292]` |
| 9 | **Atomic DB persist** | `Saving record to database` | `with transaction.atomic():` `[consumer.py:298]` → `_store(...)` `[consumer.py:379]` → `Document.objects.create(...)` |
| 10 | `document_consumption_finished` signal | *(triggers §3.4 handlers)* | `document_consumption_finished.send(...)` `[consumer.py:306]` (**inside** the atomic block) |
| 11 | Store files + thumbnail | *(files written)* | `_write(...)` `[consumer.py:429]` |
| 12 | Delete source, finish | `Deleting file ...` / `... consumption finished` | `os.unlink(self.path)` `[consumer.py:350]`; finish log; `return document` `[consumer.py:377]` |
| 13 | Return success | `Success. New document id 4 created` | `return "Success. New document id {} created"...` `[tasks.py:247]` |

### 3.2 The atomicity boundary

**(inferred from reading `[src/documents/consumer.py:298,306]`)** The row creation (`_store` → `Document.objects.create`) and the `document_consumption_finished.send(...)` call both occur **inside the same** `with transaction.atomic():` block `[consumer.py:298]`. Consequently the document row plus its derived fields are committed together, and the post-persistence organization is triggered by the signal rather than by inline code. The observed `Saving record to database` line (stage 9 above) marks entry into this block.

### 3.3 Background execution technology — Django-Q

The background execution technology is **Django-Q** (`django-q==1.3.9` `[requirements.txt:37]`; `django_q` is in `INSTALLED_APPS` `[src/paperless/settings.py:110]`) — **not Celery**. It is configured by the `Q_CLUSTER` dict `[src/paperless/settings.py:449]` and run as a long-lived cluster via `python3 manage.py qcluster` `[docker/supervisord.conf:28-29]`.

Framework behavior confirmed via the official Django-Q documentation and the `Koed00/django-q` project (framework-defined, not repository-defined):

- Django-Q is a multiprocessing distributed task queue for Django; the cluster uses a pool of worker processes.
- A worker cluster is started with `python manage.py qcluster`.
- It supports multiple **brokers** (Redis, the Django ORM, SQS, etc.); paperless-ngx uses **Redis** by default.
- The cluster is composed of a **sentinel** (spawns/health-checks/reincarnates processes), a **pusher** (pulls task packages off the broker into an internal queue), **workers** (execute tasks), a **monitor** (saves results to the DB), and a **scheduler** (fires scheduled tasks).
- Crucially — <cite index="10-24,10-25">unlike Celery, Django-Q tasks don't need decorators; any importable function can be queued as a task</cite>. This is exactly why paperless enqueues by dotted-path string `async_task("documents.tasks.consume_file", ...)` rather than via a decorated task object.

Observed `qcluster` startup banner (canonical config). The random cluster codename (`seventeen-paris-violet-shade`) matches Django-Q's documented naming convention:

```text
# qcluster.log (from: python manage.py qcluster)
23:18:29 [Q] INFO Q Cluster seventeen-paris-violet-shade starting.
23:18:29 [Q] INFO Process-1:1 ready for work at 48058
...  (Process-1:1 through Process-1:11 — 11 workers)
23:18:29 [Q] INFO Process-1:11 ready for work at 48068
23:18:29 [Q] INFO Process-1:12 monitoring at 48069
23:18:29 [Q] INFO Process-1 guarding cluster seventeen-paris-violet-shade
23:18:29 [Q] INFO Process-1:13 pushing tasks at 48070
23:18:29 [Q] INFO Q Cluster seventeen-paris-violet-shade running.
```

**Worker count = 11 (observed, stable across 2 boots — see [R10 magnitude note](#r10--magnitudetiming)).** The relevant `Q_CLUSTER` values, read from the running settings:

```text
TASK_WORKERS      = 11
Q_CLUSTER.workers = 11
Q_CLUSTER.redis   = redis://localhost:6379
Q_CLUSTER.timeout = 1800
Q_CLUSTER.retry   = 1810
Q_CLUSTER.name    = paperless
```

These map to `Q_CLUSTER` `[src/paperless/settings.py:449]`: `"name": "paperless"` `[:450]`, `"catch_up": False` `[:451]`, `"recycle": 1` `[:452]`, `"retry"` `[:453]`, `"timeout": PAPERLESS_WORKER_TIMEOUT` `[:454]` (default `1800` `[:440]`), `"workers": TASK_WORKERS` `[:455]`, `"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` `[:456]`. `PAPERLESS_WORKER_RETRY` is `timeout + 10` `[:444-446]` → `1810`. The worker count is derived (when `PAPERLESS_TASK_WORKERS` is unset) as `floor(sqrt(cpu_count))`; here `multiprocessing.cpu_count()` returned `128`, and `floor(sqrt(128)) = 11`.

> Because `Q_CLUSTER["recycle"] = 1` `[settings.py:452]`, each worker is recycled after processing a task, so higher `Process-1:N` indices (e.g. `:14`, `:15`) appear later in the log as workers are replaced — this is expected and does not change the pool size of 11.

### 3.4 Signal-driven post-processing (auto-organize + index)

When the row is persisted, `document_consumption_finished = Signal()` `[src/documents/signals/__init__.py:4]` fires. Its handlers are connected in `DocumentsConfig.ready()` `[src/documents/apps.py:11]`:

```python
document_consumption_finished.connect(add_inbox_tags)      # apps.py:22
document_consumption_finished.connect(set_correspondent)   # apps.py:23
document_consumption_finished.connect(set_document_type)   # apps.py:24
document_consumption_finished.connect(set_tags)            # apps.py:25
document_consumption_finished.connect(set_log_entry)       # apps.py:26
document_consumption_finished.connect(add_to_index)        # apps.py:27
```

Two of these fire on **every** consume regardless of configuration; I proved both at runtime after consuming four documents:

- **`set_log_entry`** `[src/documents/signals/handlers.py:413]` (creates an admin `LogEntry` as the `consumer` user via `User.objects.get(username="consumer")` `[:416]`):

```text
LOGENTRY_COUNT= 4
  LogEntry: object_id=1 repr='2026-07-02 folder_invoice'   by user=consumer
  LogEntry: object_id=2 repr='2026-07-02 rest_invoice'     by user=consumer
  LogEntry: object_id=3 repr='2026-07-02 Email Invoice Test' by user=consumer
  LogEntry: object_id=4 repr='2026-07-02 pipeline_sample'  by user=consumer
```

- **`add_to_index`** `[src/documents/signals/handlers.py:428]` (adds the document to the Whoosh full-text index via `index.add_or_update_document` `[:431]`; schema in `get_schema()` `[src/documents/index.py:31]`):

```text
INDEXED_DOC_COUNT= 4
SEARCH content:acme -> hits= 4     # all four consumed docs are searchable
```

The four *conditional* organizer handlers — `add_inbox_tags` `[handlers.py:30]`, `set_correspondent` `[:35]`, `set_document_type` `[:101]`, `set_tags` `[:168]` — also fire on every consume, but assign nothing until matching organizers exist. Their **positive** assignment evidence is shown in [§5](#5--q4--practical-organization-with-tags-correspondents--document-types).

### 3.5 The three scheduled background jobs

Beyond `consume_file`, Django-Q's scheduler runs periodic jobs. I queried the live `Schedule` table:

```bash
python manage.py shell -c "from django_q.models import Schedule; [print(s.func, s.schedule_type, s.name) for s in Schedule.objects.all()]"
```

```text
func=documents.tasks.train_classifier          | schedule_type=H | name='Train the classifier'
func=documents.tasks.index_optimize            | schedule_type=D | name='Optimize the index'
func=documents.tasks.sanity_check              | schedule_type=W | name='Perform sanity check'
func=paperless_mail.tasks.process_mail_accounts | schedule_type=I | name='Check all e-mail accounts'
SCHEDULE_TYPE constants: HOURLY='H' DAILY='D' WEEKLY='W' MINUTES='I'
```

| Job | Cadence | `schedule_type` | Task fn | Seeded by |
|-----|---------|-----------------|---------|-----------|
| Train the classifier | **HOURLY** | `H` | `train_classifier` `[src/documents/tasks.py:48]` | migration `1001_auto_20201109_1636.py` |
| Optimize the index | **DAILY** | `D` | `index_optimize` `[src/documents/tasks.py:32]` | migration `1001_auto_20201109_1636.py` |
| Perform sanity check | **WEEKLY** | `W` | `sanity_check` `[src/documents/tasks.py:255]` | migration `1004_sanity_check_schedule.py` |
| Check all e-mail accounts *(bonus)* | interval (`MINUTES`) | `I` | `paperless_mail.tasks.process_mail_accounts` | (paperless_mail) |

> The three jobs the question asks about are the first three (classifier / index / sanity check). A **fourth** schedule — `process_mail_accounts` (interval type `I`) — also exists to poll IMAP accounts; it is included here for completeness.

### 3.6 Canonical process topology

The production runtime (supervisord) runs three long-lived programs plus the Redis dependency; I started all three in the canonical configuration:

| Program | Command | Anchor | Role |
|---------|---------|--------|------|
| `gunicorn` | `gunicorn -c gunicorn.conf.py paperless.asgi:application` | `[docker/supervisord.conf:10-11]` | ASGI web/API server (`:8000`) |
| `consumer` | `python3 manage.py document_consumer` | `[docker/supervisord.conf:19-20]` | consumption-folder watcher |
| `scheduler` | `python3 manage.py qcluster` | `[docker/supervisord.conf:28-29]` | Django-Q worker **and** scheduler cluster |

Redis must be reachable at startup (`[docker/wait-for-redis.py]`); confirmed with `redis-cli ping` → `PONG`. The gunicorn server was observed starting and serving:

```text
[2026-07-02 23:18:29 +0000] [48029] [INFO] Starting gunicorn 20.1.0
[2026-07-02 23:18:29 +0000] [48029] [INFO] Listening at: http://0.0.0.0:8000 (48029)
[2026-07-02 23:18:29 +0000] [48029] [INFO] Using worker: paperless.workers.ConfigurableWorker
# curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost:8000/api/  =>  HTTP 200
```

**Q2 summary:** ordered stages of `try_consume_file()` as tabled above; DB persistence is atomic and fires the auto-organize/index signal inside the transaction; background execution is **Django-Q** (`manage.py qcluster`, **Redis** broker, 11 workers observed); three scheduled jobs — **train classifier hourly, optimize index daily, sanity check weekly**.

---

## 4 · Q3 — Per-document metadata (required vs optional vs derived)

**Short answer.** The `Document` model `[src/documents/models.py:88]` stores **15 fields**. The key insight is that **nothing is strictly *user*-required**: the pipeline **derives** the essential fields (`checksum`, `mime_type`, `content`, `filename`) and **defaults** the bookkeeping fields (`created`, `added`, `modified`, `storage_type`). The organizer fields (`correspondent`, `document_type`, `tags`, `archive_serial_number`, and `title`) are **optional**. Two fields (`archive_checksum`, `archive_filename`) are **derived-conditional** — populated only when an archive version is generated.

### 4.1 Complete field table

`class Document(models.Model)` `[src/documents/models.py:88]`, with `Meta.ordering = ("-created",)` `[:208]`:

| Field | Type | `file:line` | Classification | How populated |
|-------|------|-------------|----------------|---------------|
| `correspondent` | FK → Correspondent (`blank`, `null`, `SET_NULL`) | `models.py:97-104` | **Optional** | User, or matching/classifier at consume time |
| `title` | CharField(128, `blank`, `db_index`) | `models.py:106` | **Optional** (auto-derived) | User override, else derived from filename/subject |
| `document_type` | FK → DocumentType (`blank`, `null`, `SET_NULL`) | `models.py:108-115` | **Optional** | User, or matching/classifier |
| `content` | TextField(`blank`) | `models.py:117-124` | **Derived** | Parser text/OCR output |
| `mime_type` | CharField(256, `editable=False`) | `models.py:126` | **Derived** (DB-required: NOT NULL, no default) | libmagic (`magic.from_file`) |
| `tags` | M2M → Tag (`blank`) | `models.py:128-133` | **Optional** | User, matching, or inbox tags |
| `checksum` | CharField(32, `editable=False`, `unique`) | `models.py:135-141` | **Derived** (DB-required: NOT NULL + unique) | MD5 of the original file |
| `archive_checksum` | CharField(32, `editable=False`, `blank`, `null`) | `models.py:143-150` | **Derived-conditional** | MD5 of archive file — only if an archive is produced |
| `created` | DateTimeField(`default=timezone.now`, `db_index`) | `models.py:152` | **Always populated** | Default now, or detected document date |
| `modified` | DateTimeField(`auto_now=True`, `editable=False`) | `models.py:154-159` | **Always populated** | Auto on every save |
| `storage_type` | CharField(11, `choices`, `default=unencrypted`, `editable=False`) | `models.py:161-167` | **Always populated** | Default `unencrypted` |
| `added` | DateTimeField(`default=timezone.now`, `editable=False`) | `models.py:169-174` | **Always populated** | Default now (row insert time) |
| `filename` | FilePathField(1024, `editable=False`, `default=None`, `unique`, `null`) | `models.py:176-184` | **Derived** | `generate_filename` `[src/documents/file_handling.py]` |
| `archive_filename` | FilePathField(1024, `editable=False`, `default=None`, `unique`, `null`) | `models.py:186-194` | **Derived-conditional** | Archive storage path — only if an archive is produced |
| `archive_serial_number` | IntegerField(`blank`, `null`, `unique`, `db_index`) | `models.py:196-205` | **Optional** | Explicit user action only |

**Three senses of "required":**

- **User-required:** *none.* The pipeline derives everything essential and defaults the rest, so a user can consume a bare file with zero supplied metadata.
- **DB-required (NOT NULL, must have a value to save):** `checksum` (unique, `editable=False`) and `mime_type` (`editable=False`, no default) — both always derived by the pipeline — plus the defaulted `created` / `added` / `modified` / `storage_type`.
- **Optional (nullable/blank):** `correspondent`, `document_type`, `tags`, `archive_serial_number`, and `title` (blank-able but auto-derived).

### 4.2 Observed runtime example

I fetched the real `Document pk=1` (the `folder_invoice` consumed via the watched folder in [§2.1](#21--the-usual-path--watched-consumption-folder)) and printed every field. Command:

```bash
python manage.py shell < /tmp/obs/metadata_example.py   # Document.objects.get(pk=1)
```

Verbatim output:

```text
=== RUNTIME METADATA EXAMPLE: Document pk=1 (consumed via watched folder) ===
pk                    = 1
title                 = 'folder_invoice'
correspondent_id      = None
document_type_id      = None
tags                  = []
content (len=154)      = 'INVOICE\nFrom: ACME Corporation\nInvoice Number: ACME-2024-004'...
mime_type             = 'text/plain'
checksum              = 'a3f16230fed260267c96b95dea90b1f5'
archive_checksum      = None
filename              = '0000001.txt'
archive_filename      = None
created               = 2026-07-02T23:21:34.197098+00:00
added                 = 2026-07-02T23:21:35.872032+00:00
modified              = 2026-07-02T23:21:35.890015+00:00
storage_type          = 'unencrypted'
archive_serial_number = None
```

What this example demonstrates about each class:

- **Derived (populated though never user-supplied):** `content` (154 chars extracted by `TextDocumentParser`), `mime_type = 'text/plain'` (libmagic), `checksum = 'a3f16230…'` (MD5 of the original), and `filename = '0000001.txt'` (storage path from `generate_filename`).
- **Derived-conditional → empty here:** `archive_checksum = None` and `archive_filename = None`. A `text/plain` file produces **no** archive (PDF/A) version, so these two derived fields are correctly `None`. (They would be populated for, e.g., an OCR'd PDF.)
- **Always populated (defaults):** `created`, `added`, `modified` (all timestamps around `23:21:3x`), and `storage_type = 'unencrypted'`.
- **Optional → empty here:** `correspondent_id = None`, `document_type_id = None`, `tags = []`, and `archive_serial_number = None` — this document was consumed before any organizers existed, so nothing matched. `title = 'folder_invoice'` shows the optional-but-auto-derived behavior: no user title was supplied, so it was derived from the filename stem.

**Q3 summary:** 15 stored fields; nothing is user-required; `checksum`/`mime_type`/`content`/`filename` are derived; `created`/`added`/`modified`/`storage_type` are always populated by defaults; `correspondent`/`document_type`/`tags`/`archive_serial_number`/`title` are optional; `archive_checksum`/`archive_filename` are derived-conditional (empty for the text example, as observed).

---

## 5 · Q4 — Practical organization with tags, correspondents & document types

**Short answer.** `Tag`, `Correspondent`, and `DocumentType` all inherit from a shared base, `MatchingModel` `[src/documents/models.py:19]`, which gives each of them a `match` string and a `matching_algorithm`. When a document is consumed, the signal handlers run these match rules against the document's text and **automatically assign** the correspondent, document type, and tags. There are **six** matching algorithms; the sixth (`MATCH_AUTO`) delegates to an **ML classifier**. Users then **filter and search** documents by these organizers through the REST API.

### 5.1 The shared `MatchingModel` base and the three organizers

`class MatchingModel(models.Model)` `[src/documents/models.py:19]` defines the six algorithm constants and a `matching_algorithm` field (`PositiveIntegerField`, `default=MATCH_ANY` `[:41-45]`), plus `match` `[:39]` and `is_insensitive` (`default=True` `[:47]`). The three organizers subclass it:

- `class Correspondent(MatchingModel)` `[src/documents/models.py:57]`
- `class Tag(MatchingModel)` `[src/documents/models.py:64]` (adds `color` `[:66]` and `is_inbox_tag` `[:68]`)
- `class DocumentType(MatchingModel)` `[src/documents/models.py:82]`

### 5.2 The six matching algorithms (all enumerated)

The dispatcher is `def matches(matching_model, document)` `[src/documents/matching.py:60]`. It first returns `False` if the match string is empty (`match.strip() == ""` `[:66-67]`), applies `re.IGNORECASE` when `is_insensitive` `[:69-70]`, then branches per algorithm:

| # | Constant (value) | `models.py` | Branch | Cause → effect behavior |
|---|------------------|-------------|--------|-------------------------|
| 1 | `MATCH_ANY` = **1** | `:21` | `matching.py:84` | Splits `match` into words; returns `True` as soon as **any** word matches `\b{word}\b` in the content |
| 2 | `MATCH_ALL` = **2** | `:22` | `matching.py:72` | Returns `True` only if **every** split word is present; any missing word → `False` |
| 3 | `MATCH_LITERAL` = **3** | `:23` | `matching.py:91` | Matches the **exact phrase** via `re.escape(match)` with word boundaries |
| 4 | `MATCH_REGEX` = **4** | `:24` | `matching.py:107` | Treats `match` as a **regular expression** (`re.search(re.compile(match))`); a bad regex is logged and returns `False` `[:113-117]` |
| 5 | `MATCH_FUZZY` = **5** | `:25` | `matching.py:127` | Uses `from fuzzywuzzy import fuzz` `[:128]`; strips punctuation; returns `True` iff `fuzz.partial_ratio(match, text) >= 90` `[:135]` |
| 6 | `MATCH_AUTO` = **6** | `:26` | `matching.py:147` | Returns `False` here — *"this is done elsewhere"* `[:148]`; the actual decision comes from the **ML classifier** (see §5.4) |

An unrecognized algorithm raises `NotImplementedError("Unsupported matching algorithm")` `[matching.py:151]`.

### 5.3 Observed auto-assignment (three algorithms at once)

I created three organizers with different algorithms plus an inbox tag, then consumed a document whose text triggers all of them:

```text
# organizers created (manage.py shell)
Correspondent id=1 'ACME Corporation' algo=3(MATCH_LITERAL) match='ACME Corporation'
DocumentType  id=1 'Invoice'          algo=5(MATCH_FUZZY)   match='Invoice'
Tag           id=1 'Business'         algo=1(MATCH_ANY)     match='consulting licensing'
Tag(inbox)    id=2 'Inbox'            is_inbox_tag=True      match=''
```

Dropping `q4_match.txt` (content mentions "INVOICE", "ACME Corporation", "consulting") produced these verbatim handler log lines:

```text
# qcluster.log
[2026-07-02 23:33:20,428] [INFO] [paperless.handlers] Assigning correspondent ACME Corporation to 2026-07-02 q4_match
[2026-07-02 23:33:20,432] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-02 ACME Corporation q4_match
[2026-07-02 23:33:20,433] [INFO] [paperless.handlers] Tagging "2026-07-02 ACME Corporation q4_match" with "Business"
```

The final persisted document confirms all four handlers ran:

```text
DOC pk=5 title='q4_match'
  correspondent = ACME Corporation (via MATCH_LITERAL)
  document_type = Invoice (via MATCH_FUZZY)
  tags          = ['Business', 'Inbox']
```

Cause → effect mapping:
- `set_correspondent` `[handlers.py:35]` → `ACME Corporation` matched via **MATCH_LITERAL** (exact phrase in content).
- `set_document_type` `[handlers.py:101]` → `Invoice` matched via **MATCH_FUZZY** (`partial_ratio ≥ 90`).
- `set_tags` `[handlers.py:168]` → `Business` matched via **MATCH_ANY** (the word "consulting" is present).
- `add_inbox_tags` `[handlers.py:30]` → `Inbox` added because it is an inbox tag (`is_inbox_tag=True`), regardless of content. (Its empty `match` is why it is applied by `add_inbox_tags`, not by `matches()`, which short-circuits empty matches at `[matching.py:66-67]`.)

### 5.4 The `MATCH_AUTO` ML classifier

`MATCH_AUTO` delegates to `class DocumentClassifier(object)` `[src/documents/classifier.py:60]` (persisted model, `FORMAT_VERSION = 7` `[:63]`). The wrapper functions in `matching.py` are what actually call it — this is the *"done elsewhere"* referenced above:

- `match_correspondents` `[matching.py:21]` → `classifier.predict_correspondent(...)` `[:23]`
- `match_document_types` `[matching.py:34]` → `classifier.predict_document_type(...)` `[:36]`
- `match_tags` `[matching.py:47]` → `classifier.predict_tags(...)` `[:49]`

I created a `MATCH_AUTO` document type, assigned it to two documents as training labels, and ran the classifier training task `train_classifier` `[src/documents/tasks.py:48]`. Verbatim output:

```text
[DEBUG] [paperless.classifier] Gathering data from database...
[DEBUG] [paperless.classifier] 4 documents, 0 tag(s), 0 correspondent(s), 1 document type(s).
[DEBUG] [paperless.classifier] Vectorizing data...
[DEBUG] [paperless.classifier] Training document type classifier...
[INFO]  [paperless.tasks] Saving updated classifier model to /opt/paperless/data/classification_model.pickle...
CLASSIFIER_LOADED FORMAT_VERSION = 7
```

Prediction, via `predict_document_type` `[classifier.py:262]`:

```text
PREDICT on doc1 training content -> [2] (AutoInvoice pk=2)   # classifier learned the label
predict_correspondent -> None                                # predict_correspondent [classifier.py:251]
predict_tags          -> []                                  # predict_tags [classifier.py:273]
```

The classifier trained a document-type model (uses scikit-learn `MLPClassifier`; `scikit-learn==1.0.2` `[requirements.txt:88]`), persisted it to `classification_model.pickle`, and then predicted the `AutoInvoice` document type (`pk=2`) for the training content. `predict_correspondent` returned `None` and `predict_tags` returned `[]` because no `MATCH_AUTO` correspondent or tag existed, so those sub-classifiers were not trained. (On *unseen* text the document-type prediction returned the null class with this deliberately tiny 4-sample training set — reported exactly as observed.)

### 5.5 The practical filter / search workflow

Once organizers are assigned, users retrieve documents via the REST API filter set `DocumentFilterSet` `[src/documents/filters.py:81]`. Observed queries (verbatim status + count):

```text
# curl -H "Authorization: Token <redacted>" http://localhost:8000/api/documents/?...

?correspondent__id=1   -> count=1 results=['q4_match']                 HTTP_STATUS:200   # filters.py:110
?document_type__id=1   -> count=1 results=['q4_match']                 HTTP_STATUS:200   # filters.py:115
?tags__id__all=1       -> count=1 results=['q4_match']                 HTTP_STATUS:200   # filters.py:90
?query=consulting      -> count=2 results=['q4_match','folder_invoice'] HTTP_STATUS:200  # Whoosh full-text
```

- The **structured filters** (`correspondent__id` `[filters.py:110]`, `document_type__id` `[filters.py:115]`, `tags__id__all` `[filters.py:90]`) each return **1** — only `q4_match` was auto-assigned those organizers. Related tag filters also exist: `tags__id__none` `[filters.py:92]`, `tags__id__in` `[filters.py:94]`. Organizer filter sets: `CorrespondentFilterSet` `[filters.py:18]`, `TagFilterSet` `[filters.py:24]`, `DocumentTypeFilterSet` `[filters.py:30]`.
- The **full-text search** `?query=consulting` returns **2** — both `q4_match` and `folder_invoice` contain the word "consulting" in their indexed content (Whoosh index, `[src/documents/index.py]`). This works even though `folder_invoice` has no assigned organizers, illustrating the complementary roles: structured organizers for *categorization*, full-text index for *content search*.

**Practical workflow (cause → effect):** define an organizer with a `match` rule and algorithm → on consume, the `document_consumption_finished` signal invokes `set_correspondent`/`set_document_type`/`set_tags`, which call `matching.match_*` (or the classifier for `MATCH_AUTO`) → the document acquires a correspondent, type, and tags → `add_inbox_tags` flags new documents for triage → `add_to_index` makes both content and organizers searchable → the user narrows the collection with `/api/documents/?correspondent__id=…&document_type__id=…&tags__id__all=…` or full-text `?query=…`.

**Q4 summary:** `Tag`/`Correspondent`/`DocumentType` share the `MatchingModel` base; six algorithms (ANY=1, ALL=2, LITERAL=3, REGEX=4, FUZZY=5 with threshold ≥ 90, AUTO=6); `MATCH_AUTO` uses the scikit-learn `DocumentClassifier`; assignment is signal-driven at consume time; users filter via structured API params and search via the Whoosh full-text index.

---

## 6 · Coverage-pass checklist

Each named sub-item of the four question groups, mapped to where it is answered, with an exact literal + `file:line` + the kind of evidence.

| Question sub-item | Answered in | Exact literal + `file:line` | Evidence |
|-------------------|-------------|------------------------------|----------|
| **Q1** — usual entry point | §2.1 | `Adding {filepath} to the task queue.` `document_consumer.py:85` | Observed log line |
| Q1 — folder path enqueue | §2.1 | `async_task("documents.tasks.consume_file", ...)` `document_consumer.py:86-87` | Observed enqueue + task row (id 1) |
| Q1 — REST upload | §2.2 | `PostDocumentView` `views.py:491`; `Response("OK")` `views.py:535` | Observed HTTP 200 body `"OK"`; 401 unauth |
| Q1 — REST enqueue | §2.2 | `async_task(...)` `views.py:523-524` | Observed task row (id 2) |
| Q1 — MIME validation | §2.2 | `magic.from_buffer(document_data, mime=True)` `serialisers.py:452` | (inferred from reading) |
| Q1 — IMAP e-mail | §2.3 | `handle_message` `mail.py:272`; `async_task(...)` `mail.py:336-337` | Observed handler log + task row (id 3); transport **non-canonical** |
| Q1 — convergence | §2.4 | `Success. New document id {} created` `tasks.py:247` | Observed 3 task rows, same `func` |
| Q1 — non-canonical bulk | §2.5 | `documents.tasks.bulk_update_documents` `bulk_edit.py:18,31,47,63,87` | (inferred from reading) — labeled non-canonical |
| **Q2** — pipeline entry | §3.1 | `consume_file` `tasks.py:184`; `try_consume_file` `consumer.py:180` | Observed DEBUG trace |
| Q2 — duplicate check | §3.1 | `pre_check_duplicate` `consumer.py:213` | (inferred from reading) |
| Q2 — parser dispatch | §3.1 | `get_parser_class_for_mime_type` `consumer.py:223` | Observed `Parser: TextDocumentParser` |
| Q2 — OCR/text | §3.1 | `parse(...)` `consumer.py:261` | Observed `Parsing ...` |
| Q2 — thumbnail | §3.1 | `get_optimised_thumbnail` `consumer.py:265` | Observed `Generating thumbnail ...` |
| Q2 — load classifier | §3.1 | `load_classifier()` `consumer.py:292` | Observed `...model does not exist (yet)...` |
| Q2 — atomic persist | §3.2 | `with transaction.atomic():` `consumer.py:298` | Observed `Saving record to database`; atomicity (inferred from reading) |
| Q2 — signal fire | §3.4 | `document_consumption_finished.send(...)` `consumer.py:306` | (inferred from reading) + handler effects observed |
| Q2 — background tech = **Django-Q** | §3.3 | `Q_CLUSTER` `settings.py:449`; `django-q==1.3.9` `requirements.txt:37` | Observed qcluster banner + web-confirmed |
| Q2 — qcluster command | §3.3, §3.6 | `python3 manage.py qcluster` `supervisord.conf:28-29` | Observed banner |
| Q2 — Redis broker | §3.3 | `redis://localhost:6379` `settings.py:456` | Observed `redis-cli ping → PONG` |
| Q2 — worker count | §3.3, R10 | `TASK_WORKERS` `settings.py:438,455` | Observed 11, stable ×2 |
| Q2 — no decorators | §3.3 | dotted-path enqueue | Web-confirmed framework behavior |
| Q2 — train classifier HOURLY | §3.5 | `train_classifier` `tasks.py:48`; migration `1001` | Observed `schedule_type=H` |
| Q2 — index optimize DAILY | §3.5 | `index_optimize` `tasks.py:32`; migration `1001` | Observed `schedule_type=D` |
| Q2 — sanity check WEEKLY | §3.5 | `sanity_check` `tasks.py:255`; migration `1004` | Observed `schedule_type=W` |
| Q2 — process topology | §3.6 | `supervisord.conf:10-11,19-20,28-29` | Observed 3 startup banners |
| **Q3** — 15 fields enumerated | §4.1 | `Document` `models.py:88-205` | Field table w/ `file:line` |
| Q3 — required (DB) | §4.1 | `checksum` `models.py:135`; `mime_type` `models.py:126` | Observed non-null values |
| Q3 — derived | §4.1-4.2 | `content`/`mime_type`/`checksum`/`filename` | Observed populated values |
| Q3 — optional | §4.1-4.2 | `correspondent`/`document_type`/`tags`/`asn` | Observed `None`/`[]` |
| Q3 — always populated | §4.1-4.2 | `created`/`added`/`modified`/`storage_type` | Observed timestamps + `unencrypted` |
| Q3 — runtime example | §4.2 | `Document.objects.get(pk=1)` | Observed full field dump |
| **Q4** — `MatchingModel` base | §5.1 | `MatchingModel` `models.py:19` | (inferred from reading) |
| Q4 — 3 organizers | §5.1 | `Correspondent:57`, `Tag:64`, `DocumentType:82` | Created at runtime |
| Q4 — MATCH_ANY=1 | §5.2 | `models.py:21`; branch `matching.py:84` | Observed (`Business`) |
| Q4 — MATCH_ALL=2 | §5.2 | `models.py:22`; branch `matching.py:72` | (inferred from reading) |
| Q4 — MATCH_LITERAL=3 | §5.2-5.3 | `models.py:23`; branch `matching.py:91` | Observed (`ACME Corporation`) |
| Q4 — MATCH_REGEX=4 | §5.2 | `models.py:24`; branch `matching.py:107` | (inferred from reading) |
| Q4 — MATCH_FUZZY=5 (≥90) | §5.2-5.3 | `models.py:25`; threshold `matching.py:135` | Observed (`Invoice`) |
| Q4 — MATCH_AUTO=6 | §5.2,5.4 | `models.py:26`; branch `matching.py:147` | Observed classifier train+predict |
| Q4 — ML classifier | §5.4 | `DocumentClassifier` `classifier.py:60`; `FORMAT_VERSION=7` `:63` | Observed training + `predict_document_type=[2]` |
| Q4 — API filtering | §5.5 | `DocumentFilterSet` `filters.py:81` | Observed 4 API responses (HTTP 200) |
| Q4 — full-text search | §5.5 | Whoosh `index.py` | Observed `?query=consulting → count=2` |

---

## 7 · Environment & commands appendix

### 7.1 Canonical runtime (default configuration)

- **Python:** `python --version` → `Python 3.9.25` (canonical base `FROM python:3.9-slim-bullseye as main-app` `[Dockerfile:18]`).
- **Broker:** Redis at `redis://localhost:6379` `[settings.py:456]`; `redis-cli ping` → `PONG`.
- **Database:** SQLite (default). Observed: `ENGINE=django.db.backends.sqlite3` `[settings.py:299]`, `NAME=/opt/paperless/data/db.sqlite3` `[settings.py:300]`.
- **Key pinned versions** `[requirements.txt]`: `django==4.0.4` `:38`, `django-q==1.3.9` `:37`, `djangorestframework==3.13.1` `:39`, `django-filter==21.1` `:35`, `redis==3.5.3` `:84`, `scikit-learn==1.0.2` `:88`, `watchdog==2.1.7` `:106`, `whoosh==2.7.4` `:111`, `imap-tools==0.54.0` `:49`, `ocrmypdf==13.4.3` `:60`, `channels==3.0.4` `:23`.

### 7.2 Exact commands used

```bash
# Activate canonical venv + env; run manage.py from <repo>/src
source /opt/paperless/activate.sh
cd <repo>/src

# Database (already migrated in the image → "No migrations to apply")
python manage.py migrate

# Start the three canonical long-lived processes (production topology)
python manage.py qcluster            # Django-Q worker + scheduler cluster
python manage.py document_consumer   # consumption-folder watcher
gunicorn -c <repo>/gunicorn.conf.py paperless.asgi:application   # ASGI API on :8000

# Q1 — folder path (the usual one)
cp /tmp/obs/samples/folder_invoice.txt /opt/paperless/consume/

# Q1 — REST upload
curl -F "document=@/tmp/obs/samples/rest_invoice.txt" \
     -H "Authorization: Token <redacted>" \
     http://localhost:8000/api/documents/post_document/

# Q1 — IMAP (real handle_message enqueue; synthetic transport = non-canonical)
python manage.py shell < /tmp/obs/mail_probe.py

# Q2 — surface ordered pipeline stages (synchronous, DEBUG logging)
python manage.py shell < /tmp/obs/pipeline_trace.py

# Q2 — scheduled jobs
python manage.py shell -c "from django_q.models import Schedule; [print(s.func, s.schedule_type, s.name) for s in Schedule.objects.all()]"

# Q3 — metadata runtime example
python manage.py shell < /tmp/obs/metadata_example.py

# Q4 — create organizers + classifier demo, and API filtering
python manage.py shell < /tmp/obs/make_organizers.py
cp /tmp/obs/samples/q4_match.txt /opt/paperless/consume/
python manage.py shell < /tmp/obs/classifier_demo.py
curl -H "Authorization: Token <redacted>" "http://localhost:8000/api/documents/?correspondent__id=1"
```

> All temporary scripts and sample inputs were created under `/tmp/obs` (outside the repository) and deleted after use. Every intermediate DB row created for observation lives in `/opt/paperless/data/db.sqlite3` (outside the repository). The repository working tree is unchanged apart from this document.

### 7.3 R10 — magnitude/timing

- **Worker count = 11**, observed at the scale of two full `qcluster` boots. Boot #1 (cluster `seventeen-paris-violet-shade`) and boot #2 (cluster `asparagus-maryland-utah-arkansas`) each showed initial workers `Process-1:1` … `Process-1:11` plus one monitor and one pusher → **stable across ≥2 runs**. Derivation: `floor(sqrt(multiprocessing.cpu_count()))` with `cpu_count()=128` → `floor(sqrt(128))=11`, and `PAPERLESS_TASK_WORKERS` was unset (no override).
- **Scheduled-job cadences** `H`/`D`/`W` (hourly/daily/weekly) were read directly from the persisted `Schedule` table (deterministic seed data from migrations `1001` and `1004`), not sampled over time.

### 7.4 Non-canonical / labeled values

| Value | Why labeled | Where |
|-------|-------------|-------|
| IMAP transport | No live IMAP server in-sandbox; message object synthetic. **The real `handle_message()` enqueue path was still exercised** — only the network fetch is stubbed. | §2.3 |
| DEBUG pipeline trace (synchronous foreground run) | Canonical `consume_file` invoked directly to surface existing DEBUG stage lines; async convergence itself was proven separately in §2. Not a bypass — same code path. | §3.1 |
| Classifier prediction on *unseen* text → null class | Deliberately tiny 4-sample training set; reported exactly as observed. Prediction on *training* content correctly returned `[2]`. | §5.4 |

*End of document.*
