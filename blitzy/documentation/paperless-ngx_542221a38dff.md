# Paperless-NGX Document Ingestion — Runtime-Grounded Investigation

**Target:** paperless-ngx **v1.7.0** at commit `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`)

**Methodology (read this first):** Every behavioral claim below is grounded in **observed runtime output** captured while the as-shipped stack was actually running — not in reading code alone. Each claim carries (a) the exact command that produced the evidence, (b) the complete, unedited output, and (c) a repo-relative `file:line` reference plus the name of the specific function doing the work. Statements that could not be directly observed are explicitly labelled **[inferred]**. For the email entry point the **real paperless IMAP client transport** was exercised (imap_tools `MailBox.login().fetch()` over a live TCP socket); the only substitution is the mail *server*, which was a local minimal RFC3501 server — this is labelled **[local mail server — real transport]** where it appears.

---

## 0. How the stack was built and run (canonical configuration)

The investigation ran the as-shipped build in its **default configuration** inside the user-provided canonical image. A fresh, reproducible instance was launched from the committed durable image and started with the shipped launcher; no source file, setting, or dependency was modified.

**Exact build/run commands:**

```
$ docker run -d --name pl paperless-ngx-ready:latest -c "sleep infinity"
$ docker exec pl /usr/local/bin/start-paperless.sh
```

**Complete output of `start-paperless.sh`** (starts redis, applies migrations idempotently, launches the three supervised processes, checks the API):

```
Starting redis-server...
PONG
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
Stack started. Processes:
     17 root     redis-server *:6379
     42 testuser python3 manage.py qcluster
     45 testuser python3 manage.py document_consumer
     48 testuser gunicorn -c /app/gunicorn.conf.py paperless.asgi:application (+ workers 51,52)
     ... 11 qcluster workers ...
API check:
200
```

**Runtime confirmed:**

```
$ docker exec pl python3 --version
Python 3.9.23
$ docker exec pl redis-cli ping
PONG
$ docker exec pl bash -lc 'cd /app && git rev-parse HEAD'
542221a38dff06361e07976452f9aea24d210542
```

Python 3.9 matches `Dockerfile:L18` (`FROM python:3.9-slim-bullseye`). The four processes that make up the canonical runtime (per `docker/supervisord.conf`) were live throughout:

| Process | Role | Evidence |
|---------|------|----------|
| `redis-server` (localhost:6379) | Django-Q broker **and** Channels layer backend | `redis-cli ping` → `PONG`; `src/paperless/settings.py:L456` default `redis://localhost:6379` |
| `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` (0.0.0.0:8000) | ASGI server — serves HTTP **and** the `ws/status/` WebSocket | `docker/supervisord.conf` `[program:gunicorn]`; `worker_class=paperless.workers.ConfigurableWorker` |
| `python3 manage.py document_consumer` | Consumption-directory watcher | `docker/supervisord.conf` `[program:consumer]` |
| `python3 manage.py qcluster` | Django-Q worker cluster that actually executes `consume_file` | `docker/supervisord.conf` `[program:scheduler]` |

**Per-process startup banners** (captured from each process's stdout file):

```
$ docker exec pl bash -lc 'cat /app/data/log/qcluster.out'
05:32:36 [Q] INFO Q Cluster hamper-bakerloo-nine-low starting.
05:32:36 [Q] INFO Process-1:1..11 ready for work
05:32:36 [Q] INFO Process-1:12 monitoring at 88
05:32:36 [Q] INFO Process-1 guarding cluster hamper-bakerloo-nine-low
05:32:36 [Q] INFO Process-1:13 pushing tasks at 89
05:32:36 [Q] INFO Q Cluster hamper-bakerloo-nine-low running.

$ docker exec pl bash -lc 'cat /app/data/log/gunicorn.out'
[2026-07-08 05:32:36 +0000] [48] [INFO] Starting gunicorn 20.1.0
[2026-07-08 05:32:36 +0000] [48] [INFO] Listening at: http://0.0.0.0:8000 (48)
[2026-07-08 05:32:36 +0000] [48] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-08 05:32:36 +0000] [48] [INFO] Server is ready. Spawning workers
```

**Consumer watch-mode startup banner** (proves inotify mode, the default):

```
$ docker exec pl bash -lc "grep -F 'watch directory' /app/data/log/paperless.log"
[2026-07-08 05:32:36,837] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

This is emitted by `Command.handle_inotify` at `src/documents/management/commands/document_consumer.py:L200`. inotify is used because `PAPERLESS_CONSUMER_POLLING=0` (default, `src/paperless/settings.py:L478`); the polling alternative would log `Polling directory for changes:` from `src/documents/management/commands/document_consumer.py:L186`.

**Resolved runtime directories** (printed via `manage.py shell`, exactly as the running process resolved them):

| Setting | Resolved value | Source |
|---------|----------------|--------|
| `CONSUMPTION_DIR` | `/app/src/../consume` (= `/app/consume`) | `src/paperless/settings.py:L78-L81` |
| `ORIGINALS_DIR` | `/app/src/../media/documents/originals` | `src/paperless/settings.py:L62` |
| `ARCHIVE_DIR` | `/app/src/../media/documents/archive` | `src/paperless/settings.py:L63` |
| `THUMBNAIL_DIR` | `/app/src/../media/documents/thumbnails` | `src/paperless/settings.py:L64` |
| `DATA_DIR` | `/app/src/../data` | `src/paperless/settings.py:L66` |
| `INDEX_DIR` | `/app/src/../data/index` (Whoosh) | `src/paperless/settings.py:L73` |
| `MODEL_FILE` | `/app/src/../data/classification_model.pickle` — **ABSENT on a fresh stack** (`exists=False`) | `src/paperless/settings.py:L74` |
| `SCRATCH_DIR` | `/tmp/paperless` | `src/paperless/settings.py:L84` |
| `MEDIA_LOCK` | `/app/src/../media/media.lock` | `src/paperless/settings.py` |

Relevant defaults, all confirmed at runtime via `manage.py shell`: `CONSUMER_POLLING=0`, `CONSUMER_DELETE_DUPLICATES=False`, `CONSUMER_ENABLE_BARCODES=False`. `Q_CLUSTER` resolved to `{'name':'paperless','catch_up':False,'recycle':1,'retry':1810,'timeout':1800,'workers':11,'redis':'redis://localhost:6379'}`. `CHANNEL_LAYERS` is `channels_redis.core.RedisChannelLayer` at `['redis://localhost:6379']` (`src/paperless/settings.py:L182`).

**Logging.** The `paperless.*` loggers write to a `ConcurrentRotatingFileHandler` at `/app/data/log/paperless.log` at **DEBUG** level with format `[{asctime}] [{levelname}] [{name}] {message}` (`src/paperless/settings.py:L373-L411`). **The `paperless_mail` logger writes to a SEPARATE file** `/app/data/log/mail.log` — so email evidence below is drawn from `mail.log`, everything else from `paperless.log`.

> **Note on the logging-group correlation id.** `LoggingMixin` (`src/documents/loggers.py:L11-L21`) calls `renew_logging_group()` to set `logging_group = uuid.uuid4()` and passes `extra={"group": logging_group}` on every record, so a per-file correlation id exists on each log record. However, the default file formatter does **not** print `{group}`, so the uuid is not visible in `paperless.log` (the DB log handler that would surface it is disabled via `PAPERLESS_DISABLE_DBHANDLER=true`). The timelines below are therefore correlated by ordering + the filename embedded in each message.

**WebSocket probe (temporary, since removed).** Because `ws/status/` requires an authenticated session — `StatusConsumer.connect` raises `DenyConnection()` if `self.scope["user"]` is unauthenticated (`src/paperless/consumers.py:L13-L21`) — the probe created a Django DB session server-side for `admin` (no password used; the session was minted directly through `SessionStore`), then connected a `websockets` client to `ws://localhost:8000/ws/status/` presenting `Cookie: sessionid=<key>`, and appended every received frame to a log file. Cross-process delivery was verified before any injection: a manual `group_send("status_updates", {"type":"status_update","data":{...}})` issued from a *separate* `manage.py shell` process arrived at the probe verbatim, confirming the Redis channel layer carries frames from the qcluster worker to the ASGI socket.

**Sample inputs** (small real PDFs generated for the runs; all since deleted). Their md5 sums (raw `md5sum` output) are used as ground truth below:

```
$ docker exec pl bash -lc 'md5sum /tmp/obs/sample_blitzy.pdf /tmp/obs/sample_ep2_rest.pdf /tmp/obs/sample_q2_match.pdf /tmp/obs/sample_q3_image.pdf /tmp/obs/sample_q4_state.pdf /tmp/obs/sample_ep3_email.pdf'
76d05cddde0212a2865112f55b01c2e7  /tmp/obs/sample_blitzy.pdf
08b18d2ad7e1371aea876dd7d20d3915  /tmp/obs/sample_ep2_rest.pdf
82fa4a567aab43bf1eb979c80b26f1f5  /tmp/obs/sample_q2_match.pdf
d858f8a09de6c201d954a233796e320c  /tmp/obs/sample_q3_image.pdf
185b3d9a2a4b9b858a86481c1ed5c08a  /tmp/obs/sample_q4_state.pdf
ddab23481a0b009068e7b381912d9001  /tmp/obs/sample_ep3_email.pdf
```

| File | Size (B) | Used for | Resulting doc id |
|------|----------|----------|------------------|
| `sample_blitzy.pdf` | 1547 | EP1 (consumption dir) + the clean Q4 reference | 3 |
| `sample_ep2_rest.pdf` | 1495 | EP2 REST upload | 4 |
| `sample_q2_match.pdf` | 1526 | Q2 classification/matching | 5 |
| `sample_q3_image.pdf` | 13057 | Q3 real per-page OCR (image-only) | 6 |
| `sample_q4_state.pdf` | 1494 | Q4 before/during/after | 7 |
| `sample_ep3_email.pdf` | 1490 | EP3 email (real IMAP transport) | 8 |

The document primary keys start at **3** because the setup harness created and deleted two smoke-test documents, advancing the SQLite autoincrement (the `documents_document` sequence was observed at 2 on the fresh stack); the live document count was **0** before these injections.

---

## 1. Ingestion overview — the convergent pipeline

Three different producers detect "a new document" but all **converge on a single Django-Q task**:

```
                 detect                          handoff
CONSUMPTION_DIR  ──►  document_consumer._consume ─┐
REST upload      ──►  PostDocumentView.post       ├─► async_task("documents.tasks.consume_file", …)
email attachment ──►  MailAccountHandler.handle_* ─┘            │
                                                                ▼   (executed by qcluster worker)
                                        documents.tasks.consume_file (src/documents/tasks.py:L184)
                                                                │
                                                                ▼
                                Consumer().try_consume_file(...) (src/documents/consumer.py:L180)
                                                                │
   ┌───────────────┬───────────────┬───────────────┬───────────┴───────┬──────────────┬─────────────┐
   ▼               ▼               ▼               ▼                   ▼              ▼             ▼
 pre_check      MIME +          parse (OCR)     thumbnail          parse_date     atomic _store   post-consume
 duplicate      parser select   text extract    generate           (if no date)   Document.create signals fan-out
 (L102-L113)    (L219-L225)     (L259-L261)     (L263-L264)        (L273-L274)    (L298-L306)     (matching, LogEntry, index)
```

The staged flow inside `try_consume_file`, each stage emitting a WebSocket progress frame and/or a log line (all in `src/documents/consumer.py`):

1. **`STARTING` frame** `new_file` (0/100) — `L202`
2. **Duplicate pre-check** — `L213` → `pre_check_duplicate` `L102-L113`
3. **MIME detection** (`magic.from_file`) + **parser selection** — `L219-L225`
4. **`document_consumption_started`** signal — `L229`
5. **Parsing / OCR** — `WORKING` frame `parsing_document` (20/100) `L259`, then `RasterisedDocumentParser.parse`
6. **Thumbnail** — `WORKING` frame `generating_thumbnail` (70/100) `L264`
7. **Date parse** — `WORKING` frame `parse_date` (90/100) `L274` *(only when the parser yielded no date)*
8. **Save** — `WORKING` frame `save_document` (95/100) `L294`, then atomic `_store` `L379`
9. **`document_consumption_finished`** signal fan-out — `L306` → matching, admin LogEntry, search index
10. **Source file unlinked** — `L349-L350`
11. **`SUCCESS` frame** `finished` (100/100, carries `document_id`) — `L375`; task returns `"Success. New document id <pk> created"` `src/documents/tasks.py:L247`

The remaining sections answer Q1–Q6 in order, each leading with the direct answer.

---

## Q1 — How is a new document detected and handed off for processing?

**Direct answer.** Detection is performed by three different components depending on where the document arrives: the **consumption-directory watcher** (`_consume`, inotify) for files dropped into `CONSUMPTION_DIR`; the **REST view** `PostDocumentView.post` for HTTP uploads; and **`MailAccountHandler`** for email attachments. **Handoff is identical in all three cases**: each calls `async_task("documents.tasks.consume_file", …)`, placing the work on the Django-Q queue where the `qcluster` worker picks it up and runs `Consumer().try_consume_file(...)`. For the directory watcher, the observable handoff signal is the log line `Adding <filepath> to the task queue.`

### Entry point 1 — Consumption-directory watcher (primary path)

**Command:**
```
$ docker exec pl bash -lc 'cp /tmp/obs/sample_blitzy.pdf /app/consume/sample_blitzy.pdf'
```

**Detection + handoff (paperless.log):**
```
[2026-07-08 05:39:05,901] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_blitzy.pdf to the task queue.
```

This single line is emitted by `_consume` at `src/documents/management/commands/document_consumer.py:L85`, immediately before the enqueue at `L86-L91`:
`async_task("documents.tasks.consume_file", filepath, override_tag_ids=…, task_name=os.path.basename(filepath)[:100])`. The `_consume` function (`src/documents/management/commands/document_consumer.py:L46`) is invoked by the inotify handler when a file settles in the watched directory. The resulting Django-Q task recorded `args=('/app/src/../consume/sample_blitzy.pdf',)`, confirming the path was passed **positionally**.

### Entry point 2 — REST API upload `POST /api/documents/post_document/`

curl is not installed in the container, so the real HTTP endpoint was exercised with Python `requests`. **No password appears in any command**: a DRF auth token was minted at runtime through the ORM and passed via an environment variable, then used in an `Authorization: Token …` header (`TokenAuthentication` is enabled alongside Session/Basic auth).

**Commands:**
```
$ export PL_TOKEN=$(docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl \
    bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
u=User.objects.get(username=\"admin\")
tok,_=Token.objects.get_or_create(user=u)
print(tok.key)"')
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 -e PL_TOKEN="$PL_TOKEN" pl \
    bash -lc 'cd /app/src && python3 /tmp/obs_ep2_rest.py'
```

The harness `obs_ep2_rest.py` (temporary, since removed) reads the token from the environment — no credential literal is embedded:
```python
import os, requests
TOKEN = os.environ["PL_TOKEN"]  # runtime token; not embedded in this script
url = "http://localhost:8000/api/documents/post_document/"
with open("/tmp/obs/sample_ep2_rest.pdf", "rb") as fh:
    r = requests.post(
        url,
        headers={"Authorization": f"Token {TOKEN}"},
        files={"document": ("sample_ep2_rest.pdf", fh, "application/pdf")},
    )
print("HTTP", r.status_code)
print("BODY", repr(r.text))
```

**Output:**
```
HTTP 200
BODY '"OK"'
```

The handler is `PostDocumentView.post` (`src/documents/views.py:L497`, `permission_classes=(IsAuthenticated,)` `L493`). It writes the upload to a scratch tempfile with prefix `paperless-upload-` (`src/documents/views.py:L513`) and enqueues with a `task_id=uuid4()` (`src/documents/views.py:L521`, `async_task` `L523`), finally returning `Response("OK")` (`src/documents/views.py:L535`). There is **no** `Adding … to the task queue.` line for this path — that log belongs solely to the directory watcher; the consumer log for this run begins at `Consuming sample_ep2_rest.pdf`. The resulting Django-Q task recorded `args=('/tmp/paperless/paperless-upload-1ncu_py5',)`, confirming the `paperless-upload-` tempfile became the task input.

### Entry point 3 — Email ingestion **[local mail server — real transport]**

The **real paperless IMAP client path** was driven end-to-end: the default scheduled task `process_mail_accounts()` (`src/paperless_mail/tasks.py:L11`) iterates `MailAccount.objects.all()` and calls `MailAccountHandler().handle_mail_account(account)` (`src/paperless_mail/mail.py:L151`), which opens a mailbox with `get_mailbox().login().fetch()` (imap_tools `MailBox`) over a **live TCP socket** and issues genuine `LOGIN` / `SELECT INBOX` / `SEARCH (UNSEEN)` / `FETCH` / post-consume `UID STORE`+`EXPUNGE` commands. The only substitution is the mail **server**: a local minimal RFC3501 server served one unseen message carrying the attachment (temporary, since removed). Temporary DB rows drove it: `MailAccount` `blitzy-obs-account` (`host=127.0.0.1 port=1143 imap_security=NONE`), `MailRule` `blitzy-obs-rule` (`folder=INBOX action=MARK_READ title_source=FROM_SUBJECT correspondent_source=FROM_NOTHING`) — both deleted afterward.

**Command:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl \
    bash -lc 'cd /app/src && python3 manage.py shell -c "from paperless_mail.tasks import process_mail_accounts; print(process_mail_accounts())"'
Added 1 document(s).
```

**mail.log (complete, unedited — the real IMAP transport):**
```
[2026-07-08 05:46:41,063] [DEBUG] [paperless_mail] Processing mail account blitzy-obs-account
[2026-07-08 05:46:41,152] [DEBUG] [paperless_mail] Account blitzy-obs-account: Processing 1 rule(s)
[2026-07-08 05:46:41,153] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Selecting folder INBOX
[2026-07-08 05:46:41,195] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Searching folder with criteria (UNSEEN)
[2026-07-08 05:46:41,277] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Processing mail Blitzy Email Ingestion Test from sender@example.com with 1 attachment(s)
[2026-07-08 05:46:41,280] [INFO] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Consuming attachment sample_ep3_email.pdf from mail Blitzy Email Ingestion Test from sender@example.com
[2026-07-08 05:46:41,282] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Processed 1 matching mail(s)
[2026-07-08 05:46:41,282] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Running mail actions on 1 mails
```

The `Consuming attachment …` line is emitted at `src/paperless_mail/mail.py:L331-L332`. The attachment is written to a tempfile with prefix `paperless-mail-` (`src/paperless_mail/mail.py:L323`) and passed to `consume_file` as the **keyword** argument `path=temp_filename` (`src/paperless_mail/mail.py:L336-L338`).

### Convergence proof (all three reach the same task)

**Command:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Task
for t in Task.objects.filter(func=\"documents.tasks.consume_file\").order_by(\"started\"):
    print(\"name=\"+repr(t.name), \"success=\"+str(t.success), \"result=\"+repr(t.result), \"args=\"+repr(t.args))"'
```

**Output:**
```
name='sample_blitzy.pdf' success=True result='Success. New document id 3 created' args=('/app/src/../consume/sample_blitzy.pdf',)
name='sample_ep2_rest.pdf' success=True result='Success. New document id 4 created' args=('/tmp/paperless/paperless-upload-1ncu_py5',)
name='sample_q2_match.pdf' success=True result='Success. New document id 5 created' args=('/app/src/../consume/sample_q2_match.pdf',)
name='sample_q3_image.pdf' success=True result='Success. New document id 6 created' args=('/app/src/../consume/sample_q3_image.pdf',)
name='sample_q4_state.pdf' success=True result='Success. New document id 7 created' args=('/app/src/../consume/sample_q4_state.pdf',)
name='sample_ep3_email.pdf' success=True result='Success. New document id 8 created' args=()
```

All six rows share `func=documents.tasks.consume_file`. The `name` is the per-producer `task_name` (watcher: file basename; REST: uploaded doc name; email: attachment filename). Note the email row has `args=()` because its input path was passed as a **keyword** (`path=`), whereas the watcher and REST rows pass it **positionally** — a small but real observable difference between the producers. The full email task record confirms the keyword handoff:
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Task
t=[x for x in Task.objects.filter(func=\"documents.tasks.consume_file\") if x.result==\"Success. New document id 8 created\"][0]
print(\"args=\"+repr(t.args)); print(\"kwargs=\"+repr(t.kwargs))"'
args=()
kwargs={'path': '/tmp/paperless/paperless-mail-lpfmhgzy', 'override_filename': 'sample_ep3_email.pdf', 'override_title': 'Blitzy Email Ingestion Test', 'override_correspondent_id': None, 'override_document_type_id': None, 'override_tag_ids': []}
```

**Reasoning (cause → effect).** The three producers exist because documents can arrive by three transports (filesystem, HTTP, IMAP), but paperless funnels them into one asynchronous task so that all downstream processing (parse → classify → persist → index) is written once and shared. The `async_task` call is the handoff boundary: it serialises the request into the Redis-backed Django-Q broker, and the separate `qcluster` process dequeues and runs it — which is precisely why the work is asynchronous and why the progress/status evidence in Q3 has to be observed over a WebSocket rather than in the HTTP response.

---

## Q2 — What log messages / task names / state changes indicate the transition into parsing, classification, and indexing?

**Direct answer.** The single task name is **`documents.tasks.consume_file`**. Within its execution, the transitions are marked by DEBUG/INFO log lines from the `paperless.consumer`, `paperless.parsing.tesseract`, `paperless.classifier`, `paperless.matching`, and `paperless.handlers` loggers, and by two Django signals (`document_consumption_started`, `document_consumption_finished`). The observable ordered markers are: `Consuming <file>` → `Detected mime type:` → `Parser: <ClassName>` → `Parsing <file>...` (**parsing**) → `Generating thumbnail…` → the classifier/matching lines (**classification**) → `add_to_index` writing Whoosh (**indexing**) → `Document … consumption finished`.

The following **complete, unedited** timeline was captured for `sample_q2_match.pdf` (doc id 5), which was crafted to trigger rule-based matching:

**Command:**
```
$ docker exec pl bash -lc "awk '/Adding .*sample_q2_match/{f=1} f{print} /sample_q2_match consumption finished/{f=0}' /app/data/log/paperless.log"
```

**Output:**
```
[2026-07-08 05:42:15,003] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_q2_match.pdf to the task queue.
[2026-07-08 05:42:15,141] [INFO] [paperless.consumer] Consuming sample_q2_match.pdf
[2026-07-08 05:42:15,142] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 05:42:15,144] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 05:42:15,146] [DEBUG] [paperless.consumer] Parsing sample_q2_match.pdf...
[2026-07-08 05:42:15,168] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /app/src/../consume/sample_q2_match.pdf
[2026-07-08 05:42:15,238] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/sample_q2_match.pdf', 'output_file': '/tmp/paperless/paperless-8b_ky8bn/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-8b_ky8bn/sidecar.txt'}
[2026-07-08 05:42:15,969] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-08 05:42:15,974] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-8b_ky8bn/archive.pdf
[2026-07-08 05:42:15,974] [DEBUG] [paperless.consumer] Generating thumbnail for sample_q2_match.pdf...
[2026-07-08 05:42:15,978] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-8b_ky8bn/archive.pdf[0] /tmp/paperless/paperless-8b_ky8bn/convert.png
[2026-07-08 05:42:15,995] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 05:42:16,083] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-8b_ky8bn/gs_out.png /tmp/paperless/paperless-8b_ky8bn/convert_gs.png
[2026-07-08 05:42:16,173] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-8b_ky8bn/convert_gs.png -out /tmp/paperless/paperless-8b_ky8bn/thumb_optipng.png
[2026-07-08 05:42:16,241] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 05:42:16,244] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 05:42:16,262] [DEBUG] [paperless.matching] Correspondent Blitzy Corp matched on document 2026-07-08 sample_q2_match because it contains this word: Blitzy
[2026-07-08 05:42:16,262] [INFO] [paperless.handlers] Assigning correspondent Blitzy Corp to 2026-07-08 sample_q2_match
[2026-07-08 05:42:16,263] [DEBUG] [paperless.matching] DocumentType Observation Report matched on document 2026-07-08 Blitzy Corp sample_q2_match because it contains this word: Observation
[2026-07-08 05:42:16,264] [INFO] [paperless.handlers] Assigning document type Observation Report to 2026-07-08 Blitzy Corp sample_q2_match
[2026-07-08 05:42:16,265] [DEBUG] [paperless.matching] Tag ingested matched on document 2026-07-08 Blitzy Corp sample_q2_match because it contains this word: ingestion
[2026-07-08 05:42:16,265] [INFO] [paperless.handlers] Tagging "2026-07-08 Blitzy Corp sample_q2_match" with "ingested"
[2026-07-08 05:42:16,285] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/sample_q2_match.pdf
[2026-07-08 05:42:16,317] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-8b_ky8bn
[2026-07-08 05:42:16,318] [INFO] [paperless.consumer] Document 2026-07-08 Blitzy Corp sample_q2_match consumption finished
```

### Parsing (answered by name)

`Consuming <file>` is logged at `src/documents/consumer.py:L215`. MIME detection uses `magic.from_file` (`src/documents/consumer.py:L219`) and logs `Detected mime type:` (`L221`); the parser class is chosen by `get_parser_class_for_mime_type` (`src/documents/parsers.py:L81`) and logged as `Parser: RasterisedDocumentParser` (`src/documents/consumer.py:L246`). For `application/pdf`/images that class is `RasterisedDocumentParser` (`src/paperless_tesseract/parsers.py:L18`, `logging_name="paperless.parsing.tesseract"` `L24`). The `document_consumption_started` signal fires at `src/documents/consumer.py:L229`. Actual OCR is visible in the `paperless.parsing.tesseract` lines: text is first extracted directly, then **OCRmyPDF** is called with `skip_text: True` (so an existing text layer is preserved rather than re-OCR'd). Thumbnail generation follows (`src/documents/consumer.py:L263`), here falling back from ImageMagick to ghostscript and then `optipng`.

### Classification (answered by name)

**Observed:** on a fresh stack the ML classifier is **skipped**, and the system logs (from the timeline above):
```
[2026-07-08 05:42:16,241] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```
This is `load_classifier` (`src/documents/classifier.py:L30`) returning `None` because `MODEL_FILE` (`/app/src/../data/classification_model.pickle`) does not exist (`src/documents/classifier.py:L31-L33`). **Crucially, rule-based matching still runs even with no ML model.** After `Saving record to database` (i.e. after the row exists), the `document_consumption_finished` signal fans out to the matching handlers, which iterate over the defined entities and apply `matches()` (`src/documents/matching.py:L60`); the `paperless.matching` logger emits `<Class> <name> matched on document <doc> because <reason>` via `log_reason` (`src/documents/matching.py:L13`), and the reason `it contains this word: <word>` corresponds to the `MATCH_ANY` algorithm (`src/documents/matching.py:L84-L87`). Here the auto-match rules (temporary `Correspondent "Blitzy Corp"`, `DocumentType "Observation Report"`, `Tag "ingested"`, all `MATCH_ANY`, case-insensitive) matched on the words *Blitzy*, *Observation*, *ingestion*, and `paperless.handlers` logged the corresponding `Assigning …`/`Tagging …` actions. **[inferred]** If a trained `classification_model.pickle` were present, `load_classifier` would return a classifier and the `paperless.classifier` predict path would additionally run; that path was not exercised here because the default fresh stack has no model.

### Indexing (answered by name)

Indexing is the last handler in the fan-out: `add_to_index` (`src/documents/signals/handlers.py:L428-L430`) calls `index.add_or_update_document` (`src/documents/index.py:L118-L120`) → `update_document` (`src/documents/index.py:L87-L107`), which writes the Whoosh schema fields defined in `get_schema` (`src/documents/index.py:L31-L49`). Indexing itself emits **no** log line (verified — there is no `paperless.index` message in the timeline); its observable evidence is the resulting index entry, shown under Q4.

### Fan-out order (state change ordering)

The handler order in the timeline exactly matches the wiring in `DocumentsConfig.ready()` (`src/documents/apps.py:L22-L27`): **`add_inbox_tags` → `set_correspondent` → `set_document_type` → `set_tags` → `set_log_entry` → `add_to_index`**. `add_inbox_tags`, `set_log_entry`, and `add_to_index` produce no log line here (no inbox tags configured; the latter two write a DB LogEntry and the Whoosh entry — see Q4). A visible side effect of the ordering is that the document's string representation mutates mid-timeline — `2026-07-08 sample_q2_match` → `2026-07-08 Blitzy Corp sample_q2_match` — as the correspondent is assigned.

**Reasoning (cause → effect).** Parsing must precede persistence because the extracted text becomes the `Document.content`; classification/matching runs **after** the row is created (note `Saving record to database` precedes the matching lines) because the handlers mutate and re-save the freshly created `Document`; indexing runs last so the Whoosh entry reflects the final content and assigned metadata. The signal architecture (`started` before parsing, `finished` after save) is what lets these stages be decoupled handlers rather than inline code.

---

## Q3 — How does the system reflect progress or completion of each stage?

**Direct answer.** Progress is reflected in real time as **WebSocket status frames** broadcast on `ws/status/`, plus the log lines already shown. For a successful run the client receives **exactly six frames** at the checkpoints `STARTING(0)` → `WORKING(20)` → `WORKING(70)` → `WORKING(90)` → `WORKING(95)` → `SUCCESS(100)`. Completion is signalled two ways: the terminal `SUCCESS(100,100)` frame (the only frame that carries a non-null `document_id`), and the Django-Q task **result string** `Success. New document id <pk> created`.

The progress mechanism is `Consumer._send_progress` (`src/documents/consumer.py:L56-L76`), which builds a 7-key payload and broadcasts it via `group_send` to the `"status_updates"` group; the event `type` `status_update` is dispatched (Channels replaces dots with underscores) to `StatusConsumer.status_update` (`src/paperless/consumers.py:L29-L33`), which sends `json.dumps(event["data"])` to every subscribed client. The route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` (`src/paperless/urls.py:L137`).

**Captured frames for a full successful run** (`sample_blitzy.pdf` → doc id 3), exactly as received by the probe:

```
$ docker exec pl bash -lc 'cat /tmp/ws_ep1.log'
WS_CONNECTED
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "e76e24ac-5c70-41b2-b17f-7b489ed68271", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 3}
WS_DONE 6
```

Each frame maps to a specific `_send_progress` call (all in `src/documents/consumer.py`):

| Frame | `message` | Emitter |
|-------|-----------|---------|
| `STARTING (0,100)` | `new_file` | `L202` |
| `WORKING (20,100)` | `parsing_document` | `L259` |
| `WORKING (70,100)` | `generating_thumbnail` | `L264` |
| `WORKING (90,100)` | `parse_date` | `L274` (only if `not date`, `L273`) |
| `WORKING (95,100)` | `save_document` | `L294` |
| `SUCCESS (100,100)` | `finished`, `document_id=<pk>` | `L375` |

### Two important runtime findings that contradict a naive code reading

**(1) The intermediate 20→70 "callback band" is never emitted.** `src/documents/consumer.py:L237-L240` defines a local `progress_callback` computing `p = int((current_progress/max_progress)*50 + 20)` and passes it to the parser, whose `DocumentParser.progress()` wrapper (`src/documents/parsers.py:L300-L302`) would call it. But **no parser ever invokes `self.progress()`**, so the wrapper — and therefore the callback — is never reached:
```
$ docker exec pl bash -lc "grep -rn -E 'def progress_callback|self\.progress_callback\(' /app/src --include=*.py"
/app/src/documents/consumer.py:237:        def progress_callback(current_progress, max_progress):
/app/src/documents/parsers.py:302:            self.progress_callback(current_progress, max_progress)

$ docker exec pl bash -lc "grep -rn '\.progress(' /app/src --include=*.py || echo '(no matches)'"
(no matches)
```
The first search shows the only two references are the *definition* (`src/documents/consumer.py:237`) and the wrapper's internal call (`src/documents/parsers.py:302`, inside `DocumentParser.progress()`); the second search shows there is **no callsite** of the `.progress()` method anywhere in the source, so `src/documents/parsers.py:302` is never executed.

This was confirmed empirically: an **image-only PDF** (`sample_q3_image.pdf`, doc id 6) that forced genuine per-page OCR (OCRmyPDF ran ~2.2 s) still produced **exactly the same six discrete frames** — no frames appeared between 20 and 70:
```
$ docker exec pl bash -lc 'cat /tmp/ws_q3.log'
WS_CONNECTED
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", "task_id": "5fef7b2c-db7f-4106-aa17-9f9d34920192", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 6}
WS_DONE 6
```

**(2) The inline comment at `src/documents/consumer.py:L238` is doubly inaccurate.** It reads *"recalculate progress to be within 20 and 80"*, but (a) the arithmetic `*50 + 20` caps at **70**, not 80, and (b) the callback is never invoked anyway, so no such intermediate frame ever appears. The ground truth is what is observed above: the band does not occur.

**On the `parse_date` (90) frame:** it is present for every `RasterisedDocumentParser` run because that parser never sets `self.date`, so `get_date()` returns `None` and `if not date:` (`src/documents/consumer.py:L273`) is always true for PDF/image documents. **[inferred]** A parser that returned a date would skip this single frame; this could not be reproduced with the tesseract parser, which never sets a date.

### Completion signal — the task result string

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Task
t=[x for x in Task.objects.filter(func=\"documents.tasks.consume_file\") if x.result==\"Success. New document id 3 created\"][0]
print(repr(t.result))"'
'Success. New document id 3 created'
```
The string `Success. New document id <pk> created` is built at `src/documents/tasks.py:L247` and persisted on the Django-Q `Task` row (see Q5). *(The barcode-split alternate path would instead return `File successfully split` — `src/documents/tasks.py:L233` — but that path is default-OFF (`CONSUMER_ENABLE_BARCODES=False`) and was not exercised.)*

### Stability across runs

The checkpoint value set `{0, 20, 70, 90, 95, 100}` and the six-frame shape were **identical across six independent successful runs** (EP1, EP2, Q2-match, Q3-image-OCR, Q4-state, EP3-email), as the captured `ws_*.log` files show. There was **no run-to-run variation** in the emitted values.

**Reasoning (cause → effect).** Because the actual work happens in a separate `qcluster` process, the only way a UI can reflect progress is out-of-band; paperless uses the Redis-backed Channels layer so the worker's `group_send` reaches every browser subscribed to `ws/status/`. The frames are therefore coarse checkpoints wrapped around the expensive steps (parse, thumbnail, save) rather than a smooth percentage — and, as observed, the finer-grained callback that was intended to fill the 20→70 gap is dead code.

---

## Q4 — Where does the document's data end up, and how is its final state recorded?

**Direct answer.** A successful consume produces **six durable artifacts** and deletes the input: (1) one `Document` row in SQLite; (2) the **original** file under `originals/`; (3) an OCR'd **archive** PDF under `archive/`; (4) a **thumbnail** PNG under `thumbnails/`; (5) a **Whoosh** full-text index entry; (6) an admin **`LogEntry`** ADDITION recorded by the `consumer` user. The source file in `CONSUMPTION_DIR` is **unlinked**. There is no separate "final state" column — the *existence* of these artifacts is the final state. The reference example is doc id 3 (`sample_blitzy.pdf`), consumed cleanly via the directory watcher.

### (1) The `Document` row

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
d=Document.objects.get(pk=3)
for fld in [\"id\",\"title\",\"correspondent\",\"document_type\",\"checksum\",\"archive_checksum\",\"mime_type\",\"storage_type\",\"created\",\"modified\",\"added\",\"filename\",\"archive_filename\"]:
    print(fld, \"=\", repr(getattr(d, fld)))
print(\"tags =\", list(d.tags.all()))
print(\"content_len =\", len(d.content))"'
id = 3
title = 'sample_blitzy'
correspondent = None
document_type = None
checksum = '76d05cddde0212a2865112f55b01c2e7'
archive_checksum = 'f64a05a4a82de5fecca1ab06d6835211'
mime_type = 'application/pdf'
storage_type = 'unencrypted'
created = datetime.datetime(2026, 7, 8, 5, 39, 4, 891334, tzinfo=datetime.timezone.utc)
modified = datetime.datetime(2026, 7, 8, 5, 39, 7, 944401, tzinfo=datetime.timezone.utc)
added = datetime.datetime(2026, 7, 8, 5, 39, 7, 925495, tzinfo=datetime.timezone.utc)
filename = '0000003.pdf'
archive_filename = '0000003.pdf'
tags = []
content_len = 154
```

The row is created by `_store` (`src/documents/consumer.py:L379`, `Document.objects.create(...)` `L398`). Field grounding in `src/documents/models.py`: `Document` `L88`; `content` `L117`; `checksum` unique `L135`; `created` `L152`; `modified` `L154`; `storage_type` `L161` (default `unencrypted`); `added` `L169`; `filename` unique `L176`. The `filename` follows the `{pk:07d}.pdf` pattern. `correspondent`/`document_type`/`tags` are empty here because this document was consumed **before** the Q2 matching rules existed — demonstrating that a document persists cleanly with no metadata when nothing matches.

### (2)+(3) Media files + byte-exact checksum verification

**Command + output:**
```
$ docker exec pl bash -lc 'ls -l /app/media/documents/originals/0000003.pdf /app/media/documents/archive/0000003.pdf /app/media/documents/thumbnails/0000003.png'
-rw-r--r-- 1 testuser testuser 7605 Jul  8 05:39 /app/media/documents/archive/0000003.pdf
-rw-r--r-- 1 testuser testuser 1547 Jul  8 05:39 /app/media/documents/originals/0000003.pdf
-rw-r--r-- 1 testuser testuser 5870 Jul  8 05:39 /app/media/documents/thumbnails/0000003.png

$ docker exec pl bash -lc 'md5sum /tmp/obs/sample_blitzy.pdf /app/media/documents/originals/0000003.pdf /app/media/documents/archive/0000003.pdf'
76d05cddde0212a2865112f55b01c2e7  /tmp/obs/sample_blitzy.pdf
76d05cddde0212a2865112f55b01c2e7  /app/media/documents/originals/0000003.pdf
f64a05a4a82de5fecca1ab06d6835211  /app/media/documents/archive/0000003.pdf
```

Three facts fall out of this raw `md5sum` output: **(a)** the stored original is **byte-identical** to the injected input (both `76d05cddde0212a2865112f55b01c2e7`); **(b)** `Document.checksum` (shown above) equals that same md5 — i.e. the checksum is the md5 of the original bytes; **(c)** `Document.archive_checksum` (`f64a05a4a82de5fecca1ab06d6835211`) equals the md5 of the OCR'd archive PDF. The files are written under `FileLock(MEDIA_LOCK)` (`src/documents/consumer.py:L315`). Note the thumbnail is a **`.png`** in this version.

### (4) The Whoosh index entry

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.index import open_index
from whoosh.query import Term
from whoosh import qparser
ix=open_index()
with ix.searcher() as s:
    r=s.search(Term(\"id\",\"3\"))
    print(\"id:3 hits=\"+str(len(r)), \"stored_fields=\"+str(dict(r[0])))
    qp=qparser.QueryParser(\"content\", ix.schema)
    r2=s.search(qp.parse(\"machine-readable\"))
    print(\"content:machine-readable hits=\"+str(len(r2)), \"ids=\"+str([dict(h)[\"id\"] for h in r2]))"'
id:3 hits=1 stored_fields={'id': 3}
content:machine-readable hits=1 ids=[3]
```

Only `id` is `stored=True` in the schema (`src/documents/index.py:L33`), so a stored-field lookup returns `{'id': 3}`; the other fields are indexed-but-not-stored. Searching the `content` field for a phrase unique to this document (`machine-readable`) returns doc 3, proving the OCR text was actually indexed. Written via `add_to_index` → `add_or_update_document` → `update_document` (`src/documents/signals/handlers.py:L428`, `src/documents/index.py:L118`/`L87`).

### (5) The admin `LogEntry`

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.contrib.admin.models import LogEntry
e=LogEntry.objects.get(object_id=\"3\", content_type__model=\"document\")
print(\"LogEntry id=\"+str(e.id), \"user=\"+e.user.username, \"action_flag=\"+str(e.action_flag), \"is_addition=\"+str(e.is_addition()), \"object_repr=\"+repr(e.object_repr))"'
LogEntry id=3 user=consumer action_flag=1 is_addition=True object_repr='2026-07-08 sample_blitzy'
```

Created by `set_log_entry` (`src/documents/signals/handlers.py:L413-L425`) with `action_flag=ADDITION` (`L419`), `user=User.objects.get(username="consumer")` (`L416`), `object_id=document.pk`. This is why the stack must have a `consumer` user for consumption to succeed.

### (6) Source file removal + before/during/after (raw command output)

Using a fresh input (`sample_q4_state.pdf`, doc id 7) to show the transition cleanly. Each line below is the raw output of the labelled command.

**Before injection:**
```
$ docker exec pl bash -lc 'ls -l /app/consume/'
total 0
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
print(\"count=\"+str(Document.objects.count()))
print(\"exists=\"+str(Document.objects.filter(checksum=\"185b3d9a2a4b9b858a86481c1ed5c08a\").exists()))"'
count=4
exists=False
```

**During (file present in the consume dir, immediately after `cp`):**
```
$ docker exec pl bash -lc 'cp /tmp/obs/sample_q4_state.pdf /app/consume/sample_q4_state.pdf; ls -l /app/consume/'
total 4
-rw-r--r-- 1 root testuser 1494 Jul  8 05:43 sample_q4_state.pdf
```

**After (consume complete):**
```
$ docker exec pl bash -lc 'ls -l /app/consume/'
total 0
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
print(\"count=\"+str(Document.objects.count()))
d=Document.objects.filter(checksum=\"185b3d9a2a4b9b858a86481c1ed5c08a\").first()
print(\"exists=\"+str(d is not None), \"id=\"+str(d.id if d else None))"'
count=5
exists=True id=7
```

The deletion is logged and performed at `src/documents/consumer.py:L349-L350` (`Deleting file <path>` then `os.unlink(self.path)`):
```
$ docker exec pl bash -lc "grep -F 'Deleting file /app/src/../consume/sample_q4_state.pdf' /app/data/log/paperless.log"
[2026-07-08 05:43:26,655] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/sample_q4_state.pdf
```

Summary of state changes: `Document` **absent → present** (`exists() False → True`, `id=7`); consumption file **present → unlinked** (`total 4 → total 0`); doc count **4 → 5**; media dirs **empty → populated**; index entry **absent → present**.

**Reasoning (cause → effect).** The row is created inside `with transaction.atomic():` (`src/documents/consumer.py:L298`) and the `document_consumption_finished` signal fires inside that **same** atomic block (`L306`), so the row and all handler side effects (matching, LogEntry, index) commit together or not at all. Media writes are guarded by `FileLock(MEDIA_LOCK)` (`L315`) to serialise concurrent consumers. The source file is unlinked **only after** a successful store (`L350`) — which is exactly why, on any failure (Q6/edge paths), the row is rolled back and the source file is left in place.

---

## Q5 — How does Paperless-NGX track whether a document has already been processed or needs further work?

**Direct answer (the negative, stated up front).** At this commit there is **NO explicit per-document status / state / processed column**. Whether a document has "been processed" is tracked *implicitly* by four observable signals: **(1) the existence of a `Document` row** carrying the file's checksum; **(2) the Django-Q task ledger** (`Task` / `Success` / `Failure` records); **(3) the removal of the source file** from `CONSUMPTION_DIR`; and **(4) the terminal WebSocket status** (`SUCCESS` vs `FAILED`). There is **no re-processing queue or "needs further work" flag** — a file is (re)processed whenever it (re)appears at an entry point, gated only by the duplicate check (Q6).

### Proof: the `Document` model has no status field

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
names=[f.name for f in Document._meta.get_fields()]
print(names)
print(\"status-like:\", [n for n in names if any(k in n.lower() for k in (\"status\",\"state\",\"process\",\"stage\",\"phase\",\"done\",\"complete\"))])"'
['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum', 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename', 'archive_filename', 'archive_serial_number', 'tags']
status-like: []
```

Filtering the field names for any of `status/state/process/stage/phase/done/complete` returns an empty list. The underlying SQLite table `documents_document` has the same 15 concrete columns (verified via `connection.introspection.get_table_description`: `id, title, content, created, modified, correspondent_id, checksum, added, storage_type, archive_serial_number, document_type_id, mime_type, archive_checksum, archive_filename, filename`) and no status column. Grounding: `Document` `src/documents/models.py:L88`; none of its fields represents processing state.

### Signal 1 — Row existence (the durable "processed" marker)

**Command + output:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
print(\"doc3 checksum exists:\", Document.objects.filter(checksum=\"76d05cddde0212a2865112f55b01c2e7\").exists())
print(\"absent checksum exists:\", Document.objects.filter(checksum=\"00000000000000000000000000000000\").exists())"'
doc3 checksum exists: True
absent checksum exists: False
```
A document counts as "processed" iff a row with its checksum exists; the row is created only at the very end of `try_consume_file`. This is exactly the query `pre_check_duplicate` performs (Q6).

### Signal 2 — The Django-Q task ledger (per-run record)

`Q_CLUSTER` was observed as `{'name':'paperless','catch_up':False,'recycle':1,'retry':1810,'timeout':1800,'workers':11,'redis':'redis://localhost:6379'}` (no `save_limit` override, so the django-q default of 250 saved successes applies; **failures are always saved**). Every executed task is persisted:

```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Task, Success, Failure
print(\"Task total=\"+str(Task.objects.count()), \"Success total=\"+str(Success.objects.count()), \"Failure total=\"+str(Failure.objects.count()))
print(\"consume_file Success count=\"+str(Success.objects.filter(func=\"documents.tasks.consume_file\").count()))
for t in Success.objects.filter(func=\"documents.tasks.consume_file\").order_by(\"started\"):
    print(t.id, \"|\", t.name, \"|\", t.started.isoformat(), \"|\", t.stopped.isoformat(), \"|\", t.success, \"|\", repr(t.result))"'
Task total=10 Success total=10 Failure total=0
consume_file Success count=6
93095348693a482180c284bb23e70543 | sample_blitzy.pdf | 2026-07-08T05:39:05.903277+00:00 | 2026-07-08T05:39:07.994251+00:00 | True | 'Success. New document id 3 created'
e5f89c9ff005409c99174e31a9147706 | sample_ep2_rest.pdf | 2026-07-08T05:40:46.561485+00:00 | 2026-07-08T05:40:47.893944+00:00 | True | 'Success. New document id 4 created'
df0fd99f1fdf46fb8ec1237ccf3325d4 | sample_q2_match.pdf | 2026-07-08T05:42:15.005204+00:00 | 2026-07-08T05:42:16.334119+00:00 | True | 'Success. New document id 5 created'
7a24ef50759a4bc88bfbfa8d7015d3ed | sample_q3_image.pdf | 2026-07-08T05:42:38.333245+00:00 | 2026-07-08T05:42:41.685248+00:00 | True | 'Success. New document id 6 created'
ab9e3d08a58a455f8f041c3a1af19d70 | sample_q4_state.pdf | 2026-07-08T05:43:25.384890+00:00 | 2026-07-08T05:43:26.677055+00:00 | True | 'Success. New document id 7 created'
927b159b64a34b3caaa970503943cc42 | sample_ep3_email.pdf | 2026-07-08T05:46:41.281596+00:00 | 2026-07-08T05:46:42.615515+00:00 | True | 'Success. New document id 8 created'
```

(The above was captured before the Q6/edge failure runs; those add `Failure` rows, shown next.) Each `Task` carries a uuid `id`, a humanised `name` (the `task_name`), `func='documents.tasks.consume_file'`, the `result` string, `started`/`stopped` timestamps, and a `success` boolean. `Success` and `Failure` are proxy models of `Task` filtered on `success=True/False`. This ledger is the per-run "processing record": a completed run leaves a `Success` row whose `result` names the created document id; a failed run leaves a `Failure` row (shown in Q6/edge paths). *(These semantics match the official Django-Q documentation on task/result persistence; the values quoted are the ones observed here.)*

### Signal 3 — Source-file removal

As shown in Q4, a successfully consumed file is unlinked from `CONSUMPTION_DIR` (`src/documents/consumer.py:L350`). Conversely, a file still present in the consume directory was **not** successfully consumed — this is the filesystem-level signal, and it is exactly what happens on the error paths (Q6, edges): the source file is left in place.

### Signal 4 — Terminal WebSocket status

The final `ws/status/` frame is `SUCCESS` on success and `FAILED` on any failure (Q3/Q6) — the real-time signal a UI client uses to decide whether processing completed.

**"Needs further work" — [inferred from observed absence].** There is no queue or column expressing partial/incomplete processing. Because a failed run rolls back inside the atomic block, it leaves **no** `Document` row (verified in the edge paths: document count is unchanged after each failure), and the only trace is the Django-Q `Failure` record. The system therefore does not "remember" that a document needs re-work; re-work only happens if the file is presented again at an entry point.

**Reasoning (cause → effect).** Paperless treats the presence of a fully-formed `Document` row as the single source of truth for "processed," which is why no status column is needed: partially-processed states never persist (the atomic transaction guarantees all-or-nothing), so a row's mere existence already means "fully processed." The Django-Q ledger and the WebSocket status exist to *report* on runs, not to *drive* re-processing.

---

## Q6 — What behavior/logs indicate how the system avoids duplicate processing?

**Direct answer.** Duplicate avoidance has **two independent layers**: **(1)** an application-level md5 pre-check, `Consumer.pre_check_duplicate` (`src/documents/consumer.py:L102-L113`), which hashes the input and fails fast with the log `It is a duplicate.` and the WebSocket status `document_already_exists` if a `Document` already has that checksum (or archive_checksum); and **(2)** a database `UNIQUE` constraint on `Document.checksum` (`src/documents/models.py:L135`) that raises `IntegrityError` at write time even if the app check were bypassed.

The same bytes as doc 3 (`sample_blitzy.pdf`) were re-injected.

### Layer 1 — application pre-check

`pre_check_duplicate` opens the input, computes `checksum = hashlib.md5(f.read()).hexdigest()` (`src/documents/consumer.py:L104`), and runs `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists()` (`L105-L107`). On a hit it calls `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, "Not consuming {filename}: It is a duplicate.")` (`L110-L113`), where the constant is `"document_already_exists"` (`src/documents/consumer.py:L37`).

**Before (raw):**
```
$ docker exec pl bash -lc 'ls -l /app/consume/'
total 0
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
from django_q.models import Failure
print(\"doc_count=\"+str(Document.objects.count()), \"failure_count=\"+str(Failure.objects.count()))"'
doc_count=6 failure_count=0
```

**Inject:**
```
$ docker exec pl bash -lc 'cp /tmp/obs/sample_blitzy.pdf /app/consume/sample_blitzy.pdf'
```

**During (paperless.log):**
```
[2026-07-08 05:51:55,549] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_blitzy.pdf to the task queue.
[2026-07-08 05:51:55,690] [ERROR] [paperless.consumer] Not consuming sample_blitzy.pdf: It is a duplicate.
```
Note the watcher **still enqueues** the file (the `Adding … to the task queue.` line at `src/documents/management/commands/document_consumer.py:L85` runs before any hashing); the duplicate detection happens **inside the worker**.

**During (WebSocket) — only two frames, because it fails before parsing:**
```
$ docker exec pl bash -lc 'cat /tmp/ws_dup.log'
WS_CONNECTED
FRAME {"filename": "sample_blitzy.pdf", "task_id": "0d38dba8-3506-4d6b-a611-e4b8979ee72a", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "0d38dba8-3506-4d6b-a611-e4b8979ee72a", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
WS_DONE 2
```
The `FAILED(100,100)` frame is produced by `_fail` → `_send_progress` (`src/documents/consumer.py:L78-L81`), which sends the terminal frame and then raises `ConsumerError`.

**During (Django-Q Failure record — complete, unedited):**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Failure
t=Failure.objects.filter(func=\"documents.tasks.consume_file\", name=\"sample_blitzy.pdf\").order_by(\"-started\").first()
print(\"id=\"+t.id, \"success=\"+str(t.success), \"args=\"+repr(t.args)); print(t.result)"'
id=a943dbe3be7c4864aa98144474d5dd95 success=False args=('/app/src/../consume/sample_blitzy.pdf',)
sample_blitzy.pdf: Not consuming sample_blitzy.pdf: It is a duplicate. : Traceback (most recent call last):
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
documents.consumer.ConsumerError: sample_blitzy.pdf: Not consuming sample_blitzy.pdf: It is a duplicate.
```

**Byte-exact proof the md5 pre-check compares real bytes (raw output):**
```
$ docker exec pl bash -lc 'md5sum /app/consume/sample_blitzy.pdf'
76d05cddde0212a2865112f55b01c2e7  /app/consume/sample_blitzy.pdf
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "from documents.models import Document; print(Document.objects.get(pk=3).checksum)"'
76d05cddde0212a2865112f55b01c2e7
```

**After (raw):**
```
$ docker exec pl bash -lc 'ls -l /app/consume/'
total 4
-rw-r--r-- 1 root testuser 1547 Jul  8 05:51 sample_blitzy.pdf
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
from django_q.models import Failure
print(\"doc_count=\"+str(Document.objects.count()), \"failure_count=\"+str(Failure.objects.count()))"'
doc_count=6 failure_count=1
```
Doc count is **6, unchanged** (no new row); the `Failure` ledger gained one row. The duplicate input **remains** in `/app/consume` because `CONSUMER_DELETE_DUPLICATES=False` (default, `src/paperless/settings.py:L486`; the unlink at `src/documents/consumer.py:L108-L109` runs only when that setting is `True`). *(If toggled on — a non-default config — the duplicate input would additionally be deleted.)*

### Layer 2 — database `UNIQUE` constraint (independent guarantee)

Bypassing the app check entirely by creating a row directly via the ORM with an existing checksum:

**Command + output (complete, unedited traceback).** The probe was written to a file so its line numbers are stable and inspectable, then piped to `manage.py shell`:
```
$ cat > /tmp/dup_probe.py <<'PY'
import traceback
from django.db import IntegrityError, transaction
from documents.models import Document

before = Document.objects.count()
print("doc_count BEFORE =", before)
dup = Document.objects.get(pk=3).checksum
try:
    with transaction.atomic():
        Document.objects.create(
            title="blitzy-dup-constraint-probe",
            content="probe",
            checksum=dup,
            mime_type="application/pdf",
        )
except IntegrityError:
    print(traceback.format_exc())
after = Document.objects.count()
print("doc_count AFTER =", after, "(unchanged =", before == after, ")")
PY
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell < /tmp/dup_probe.py'
doc_count BEFORE = 6
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/sqlite3/base.py", line 477, in execute
    return Database.Cursor.execute(self, query, params)
sqlite3.IntegrityError: UNIQUE constraint failed: documents_document.checksum

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "<string>", line 10, in <module>
  File "/usr/local/lib/python3.9/site-packages/django/db/models/manager.py", line 85, in manager_method
    return getattr(self.get_queryset(), name)(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/django/db/models/query.py", line 514, in create
    obj.save(force_insert=True, using=self.db)
  File "/usr/local/lib/python3.9/site-packages/django/db/models/base.py", line 806, in save
    self.save_base(
  File "/usr/local/lib/python3.9/site-packages/django/db/models/base.py", line 857, in save_base
    updated = self._save_table(
  File "/usr/local/lib/python3.9/site-packages/django/db/models/base.py", line 1000, in _save_table
    results = self._do_insert(
  File "/usr/local/lib/python3.9/site-packages/django/db/models/base.py", line 1041, in _do_insert
    return manager._insert(
  File "/usr/local/lib/python3.9/site-packages/django/db/models/manager.py", line 85, in manager_method
    return getattr(self.get_queryset(), name)(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/django/db/models/query.py", line 1434, in _insert
    return query.get_compiler(using=using).execute_sql(returning_fields)
  File "/usr/local/lib/python3.9/site-packages/django/db/models/sql/compiler.py", line 1621, in execute_sql
    cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 67, in execute
    return self._execute_with_wrappers(
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 80, in _execute_with_wrappers
    return executor(sql, params, many, context)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/utils.py", line 91, in __exit__
    raise dj_exc_value.with_traceback(traceback) from exc_value
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/sqlite3/base.py", line 477, in execute
    return Database.Cursor.execute(self, query, params)
django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum

doc_count AFTER = 6 (unchanged = True )
```

This is enforced by `checksum = models.CharField(..., unique=True)` at `src/documents/models.py:L135`. Both the underlying `sqlite3.IntegrityError` and the wrapping `django.db.utils.IntegrityError` name `documents_document.checksum`; the count remains 6 (the create rolls back inside the `transaction.atomic()` block).

**Reasoning (cause → effect).** The two layers serve different purposes. Layer 1 is a **fast, user-facing** guard: it hashes the input before any expensive OCR and surfaces a clear `document_already_exists` status so the client learns *why* nothing happened — and it checks **both** `checksum` and `archive_checksum` so a re-fed archive PDF is also caught. Layer 2 is the **last-resort integrity guarantee** at the storage layer: even if two identical files were processed concurrently and both passed the app check before either committed, the database `UNIQUE(checksum)` would reject the second write. Together they make duplicate ingestion both cheap to reject and impossible to persist.

---

## Entry-point comparison (how the three producers converge)

| Aspect | Consumption directory | REST API upload | Email attachment |
|--------|-----------------------|-----------------|------------------|
| Detector / function | `_consume` (inotify) `src/documents/management/commands/document_consumer.py:L46` | `PostDocumentView.post` `src/documents/views.py:L497` | `MailAccountHandler.handle_mail_account` `src/paperless_mail/mail.py:L151` |
| Detection trigger | file settles in `CONSUMPTION_DIR` | authenticated `POST /api/documents/post_document/` | mail rule matches an attachment |
| Input handed to task | the file path in `CONSUMPTION_DIR` (positional) | scratch tempfile `paperless-upload-…` (positional) | scratch tempfile `paperless-mail-…` (keyword `path=`) |
| Distinct handoff log | `Adding <path> to the task queue.` (`src/documents/management/commands/document_consumer.py:L85`) | *(none — no watcher line)* | `Rule …: Consuming attachment …` in **mail.log** (`src/paperless_mail/mail.py:L331-L332`) |
| Handoff call | `async_task("documents.tasks.consume_file", …)` (`src/documents/management/commands/document_consumer.py:L86-L91`) | `async_task("documents.tasks.consume_file", …)` (`src/documents/views.py:L523`) | `async_task("documents.tasks.consume_file", path=…)` (`src/paperless_mail/mail.py:L336-L338`) |
| `task_id` origin | generated inside `try_consume_file` (no id passed) | `uuid4()` in the view (`src/documents/views.py:L521`) | generated inside `try_consume_file` |
| Observed `Task.args` | `('/app/src/../consume/…',)` | `('/tmp/paperless/paperless-upload-1ncu_py5',)` | `()` (path passed as kwarg) |
| Observed doc id | 3 | 4 | 8 |
| Converges on | `documents.tasks.consume_file` → `Consumer().try_consume_file` (identical from here on) | same | same |

All three were observed to produce `func=documents.tasks.consume_file` tasks that succeeded with `Success. New document id 3/4/8 created` respectively (Q1). After the handoff, the code path is identical — which is why Q2–Q6 are answered once and apply to all three transports. (The REST `task_id` uuid4 additionally flows into the WebSocket frames: for doc 4 the observed WS `task_id` was `689b5494-3366-48fd-8c1f-a43888252872`, generated by the view at `src/documents/views.py:L521`; the watcher and email `task_id`s are instead generated inside `try_consume_file`.)

---

## Edge / error paths (exercising every condition)

Four non-happy-path conditions were exercised. The common outcome for all *failures* is: **no `Document` row is created** (count unchanged), and **the source file is NOT unlinked** (it remains in `CONSUMPTION_DIR`, because `os.unlink` at `src/documents/consumer.py:L350` runs only after a successful store).

### (a) Duplicate — see Q6 above (two layers, `document_already_exists` + `IntegrityError`).

### (b) Unsupported extension (rejected at detection, before handoff)

A file with an unknown extension (`sample_badext.xyz`) is rejected by the watcher itself via `is_file_ext_supported` (`src/documents/parsers.py:L62`) before any task is enqueued.

**Before:**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
from django_q.models import Task
print(\"doc_count=\"+str(Document.objects.count()), \"consume_file_task_count=\"+str(Task.objects.filter(func=\"documents.tasks.consume_file\").count()))"'
doc_count=6 consume_file_task_count=7
```

**Inject + watcher log:**
```
$ docker exec pl bash -lc 'cp /tmp/obs/sample_badext.xyz /app/consume/sample_badext.xyz'
[2026-07-08 05:54:46,757] [WARNING] [paperless.management.consumer] Not consuming file /app/src/../consume/sample_badext.xyz: Unknown file extension.
```

**After (task count unchanged — nothing enqueued):**
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
from django_q.models import Task
print(\"doc_count=\"+str(Document.objects.count()), \"consume_file_task_count=\"+str(Task.objects.filter(func=\"documents.tasks.consume_file\").count()))"'
doc_count=6 consume_file_task_count=7
$ docker exec pl bash -lc 'ls -l /app/consume/'
total 4
-rw-r--r-- 1 root testuser 50 Jul  8 05:54 sample_badext.xyz
```

This rejection is `src/documents/management/commands/document_consumer.py:L54-L55`. **No** `consume_file` task was enqueued (count unchanged at 7), **no** WebSocket frame was emitted, and **no** `Document` row was created — the rejection happens entirely in the watcher, upstream of the queue; the file is left untouched. The supported-extension set observed at runtime is: `.bat .bmp .brf .c .csv .gif .h .jfif .jpe .jpeg .jpg .ksh .pdf .pl .png .pot .srt .text .tif .tiff .txt`.

### (c) Unsupported MIME (rejected inside the worker, before parsing)

A file with a `.pdf` extension but random-binary content (`sample_unsupported.pdf`, md5 `9c340f8304e4f171de1f57b99f4b7e5a`) passes the extension check but fails MIME-based parser selection: `magic.from_file` returns `application/octet-stream`, `get_parser_class_for_mime_type` returns `None`, and `try_consume_file` calls `_fail(MESSAGE_UNSUPPORTED_TYPE, …)` (`src/documents/consumer.py:L223-L225`, constant `"unsupported_type"` `L44`).

**paperless.log (complete):**
```
[2026-07-08 05:55:15,266] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_unsupported.pdf to the task queue.
[2026-07-08 05:55:15,402] [INFO] [paperless.consumer] Consuming sample_unsupported.pdf
[2026-07-08 05:55:15,404] [DEBUG] [paperless.consumer] Detected mime type: application/octet-stream
[2026-07-08 05:55:15,406] [ERROR] [paperless.consumer] Unsupported mime type application/octet-stream
```
**WebSocket (two frames — fails before parsing, so no `parsing_document` frame):**
```
$ docker exec pl bash -lc 'cat /tmp/ws_edgec.log'
WS_CONNECTED
FRAME {"filename": "sample_unsupported.pdf", "task_id": "07bfedf5-c94a-4fdb-83f7-11ef856bbafd", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_unsupported.pdf", "task_id": "07bfedf5-c94a-4fdb-83f7-11ef856bbafd", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "unsupported_type", "document_id": null}
WS_DONE 2
```
**Django-Q Failure (complete, unedited):**
```
id=37dd5f286f42440c9c68a5106741a7e5 name='sample_unsupported.pdf' success=False
args=('/app/src/../consume/sample_unsupported.pdf',)
sample_unsupported.pdf: Unsupported mime type application/octet-stream : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 225, in try_consume_file
    self._fail(MESSAGE_UNSUPPORTED_TYPE, f"Unsupported mime type {mime_type}")
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: sample_unsupported.pdf: Unsupported mime type application/octet-stream
```
Doc count 6 → 6 (unchanged); the source file remains in `/app/consume`.

### (d) Parse failure (fails during parsing)

A file that libmagic reports as `application/pdf` (a `%PDF` header followed by garbage, `sample_corrupt.pdf`, md5 `15814d1eea261f1b43d940ce19963a11`) reaches the parser and then raises `ParseError` (`src/documents/parsers.py:L277`, raised at `src/paperless_tesseract/parsers.py:L310`), handled at `src/documents/consumer.py:L278-L284` via `_fail`.

**paperless.log (complete, unedited — including the pdfminer failure and the skip_text → force_ocr fallback):**
```
[2026-07-08 05:56:04,604] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_corrupt.pdf to the task queue.
[2026-07-08 05:56:04,745] [INFO] [paperless.consumer] Consuming sample_corrupt.pdf
[2026-07-08 05:56:04,745] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 05:56:04,746] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 05:56:04,749] [DEBUG] [paperless.consumer] Parsing sample_corrupt.pdf...
[2026-07-08 05:56:04,768] [WARNING] [paperless.parsing.tesseract] Error while getting text from PDF document with pdfminer.six
Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 120, in extract_text
    stripped = post_process_text(pdfminer_extract_text(pdf_file))
  File "/usr/local/lib/python3.9/site-packages/pdfminer/high_level.py", line 157, in extract_text
    for page in PDFPage.get_pages(
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfpage.py", line 151, in get_pages
    doc = PDFDocument(parser, password=password, caching=caching)
  File "/usr/local/lib/python3.9/site-packages/pdfminer/pdfdocument.py", line 752, in __init__
    raise PDFSyntaxError("No /Root object! - Is this really a PDF?")
pdfminer.pdfparser.PDFSyntaxError: No /Root object! - Is this really a PDF?
[2026-07-08 05:56:04,845] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/sample_corrupt.pdf', 'output_file': '/tmp/paperless/paperless-lpjmjubv/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-lpjmjubv/sidecar.txt'}
[2026-07-08 05:56:05,012] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: . Attempting force OCR to get the text.
[2026-07-08 05:56:05,012] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/sample_corrupt.pdf', 'output_file': '/tmp/paperless/paperless-lpjmjubv/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-lpjmjubv/sidecar-fallback.txt'}
[2026-07-08 05:56:05,105] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-lpjmjubv
[2026-07-08 05:56:05,109] [ERROR] [paperless.consumer] Error while consuming document sample_corrupt.pdf: InputFileError: 
```
**WebSocket (three frames — reaches `parsing_document` (20) before failing, distinguishing it from case (c)):**
```
$ docker exec pl bash -lc 'cat /tmp/ws_edged.log'
WS_CONNECTED
FRAME {"filename": "sample_corrupt.pdf", "task_id": "03ddad37-89d1-4184-9ca4-391e21be2cb2", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_corrupt.pdf", "task_id": "03ddad37-89d1-4184-9ca4-391e21be2cb2", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
FRAME {"filename": "sample_corrupt.pdf", "task_id": "03ddad37-89d1-4184-9ca4-391e21be2cb2", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "InputFileError: ", "document_id": null}
WS_DONE 3
```
The `message` is the literal `"InputFileError: "` — a **trailing space is part of the byte-exact output** and is preserved here unmodified. It arises because `RasterisedDocumentParser.parse` raises `ParseError(f"{e.__class__.__name__}: {str(e)}")` (`src/paperless_tesseract/parsers.py:L310`) where the underlying `ocrmypdf.exceptions.InputFileError` has an empty `str(e)`, yielding `"InputFileError: "` (class name, colon, space, empty message). The same trailing space therefore appears in the `ERROR` log line and in the Django-Q `result` below.

**Django-Q Failure (complete, unedited — the full chained traceback).** Query:
```
$ docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 pl bash -lc 'cd /app/src && python3 manage.py shell -c "
from django_q.models import Failure
t=Failure.objects.filter(func=\"documents.tasks.consume_file\", name=\"sample_corrupt.pdf\").order_by(\"-started\").first()
print(\"id=\"+t.id, \"name=\"+repr(t.name), \"success=\"+str(t.success)); print(\"args=\"+repr(t.args)); print(t.result)"'
id=4911535ab9a641858d644706d7ab87d3 name='sample_corrupt.pdf' success=False
args=('/app/src/../consume/sample_corrupt.pdf',)
sample_corrupt.pdf: Error while consuming document sample_corrupt.pdf: InputFileError:  : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 163, in get_pdfinfo
    return PdfInfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/pdfinfo/info.py", line 901, in __init__
    with Pdf.open(infile) as pdf:
  File "/usr/local/lib/python3.9/site-packages/pikepdf/_methods.py", line 923, in open
    pdf = Pdf._open(
pikepdf._qpdf.PdfError: /tmp/ocrmypdf.io.vmzy91s1/origin.pdf: unable to find trailer dictionary while recovering damaged file

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 261, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 370, in run_pipeline
    pdfinfo = get_pdfinfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 174, in get_pdfinfo
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 163, in get_pdfinfo
    return PdfInfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/pdfinfo/info.py", line 901, in __init__
    with Pdf.open(infile) as pdf:
  File "/usr/local/lib/python3.9/site-packages/pikepdf/_methods.py", line 923, in open
    pdf = Pdf._open(
pikepdf._qpdf.PdfError: /tmp/ocrmypdf.io.kozygyre/origin.pdf: unable to find trailer dictionary while recovering damaged file

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/src/paperless_tesseract/parsers.py", line 298, in parse
    ocrmypdf.ocr(**args)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/api.py", line 337, in ocr
    return run_pipeline(options=options, plugin_manager=plugin_manager, api=True)
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_sync.py", line 370, in run_pipeline
    pdfinfo = get_pdfinfo(
  File "/usr/local/lib/python3.9/site-packages/ocrmypdf/_pipeline.py", line 174, in get_pdfinfo
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/asgiref/sync.py", line 266, in main_wrap
    raise exc_info[1]
  File "/app/src/documents/consumer.py", line 261, in try_consume_file
    document_parser.parse(self.path, mime_type, self.filename)
  File "/app/src/paperless_tesseract/parsers.py", line 310, in parse
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: InputFileError: 

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 280, in try_consume_file
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: sample_corrupt.pdf: Error while consuming document sample_corrupt.pdf: InputFileError: 
```
Doc count 6 → 6 (unchanged); the corrupt source file remains in `/app/consume`. The chain shows the two OCR attempts (initial `skip_text` at `src/paperless_tesseract/parsers.py:L261`, force-OCR fallback at `L298`), the underlying `pikepdf._qpdf.PdfError` (`unable to find trailer dictionary while recovering damaged file`), the `documents.parsers.ParseError: InputFileError:` raised at `src/paperless_tesseract/parsers.py:L310`, and finally `_fail` at `src/documents/consumer.py:L280`.

### Edge-path state summary (before / during / after)

| Case | Enqueued? | WS frames | `Document` row | Source file after | Failure ledger |
|------|-----------|-----------|----------------|-------------------|----------------|
| Unsupported extension | **No** (watcher rejects) | none | none created (count 6→6) | remains | +0 (no task) |
| Unsupported MIME | yes | `STARTING`→`FAILED(unsupported_type)` | none created (count 6→6) | remains | +1 |
| Parse failure | yes | `STARTING`→`WORKING(20)`→`FAILED(InputFileError: )` | none created (count 6→6) | remains | +1 |

The FAILED-frame `message` differs by cause: `document_already_exists` (duplicate), `unsupported_type` (no parser), or the `ParseError` string itself (`InputFileError: `). The `WORKING(20)` frame appears only for the parse-failure case because that is the only failure reached *after* the `parsing_document` checkpoint.

---

## Methodology notes, labels, and coverage pass

**Labels used above.** For the email entry point the **real paperless IMAP client transport** was exercised end-to-end (`get_mailbox().login().fetch()` over a live TCP socket, with genuine `SELECT`/`SEARCH (UNSEEN)`/`FETCH`), labelled **[local mail server — real transport]** in Q1; only the mail *server* was a local minimal RFC3501 server. Two statements are labelled **[inferred]**: (i) that a parser returning a date would skip the `parse_date(90)` frame (not reproducible with the tesseract parser, which never sets a date); and (ii) that a trained `classification_model.pickle` would additionally run the ML predict path (the default fresh stack has no model). Everything else is directly observed.

**Stability.** The progress checkpoint set `{0,20,70,90,95,100}` and the six-frame success shape were stable across **six** independent successful runs (EP1/EP2/Q2/Q3-image-OCR/Q4-state/EP3-email); no run-to-run variation in the emitted values was observed.

**Observation-harness artifact (disclosed).** The local minimal IMAP server does not persist the `\Seen` flag, so while it and the temporary `MailAccount` were left running, the scheduled `process_mail_accounts` re-fetched the same message once and produced a second `sample_ep3_email.pdf` duplicate `Failure`. This is a **test-harness artifact, not paperless behavior**; the IMAP server was stopped and the `MailAccount`/`MailRule` deleted. Each edge-path before/after delta above was measured independently and is unaffected.

**Coverage pass (every named item addressed):**

- **Q1 detection & handoff** — three detectors named (`_consume`, `PostDocumentView.post`, `MailAccountHandler.handle_mail_account`); shared handoff `async_task("documents.tasks.consume_file")`; convergence proven via the task ledger (6 rows). ✔
- **Q2 parsing / classification / indexing** — each named and evidenced: parsing (`RasterisedDocumentParser`, OCRmyPDF), classification (ML skipped with `load_classifier` `None`; rule-based `matching.py` still runs), indexing (`add_to_index` → Whoosh). ✔
- **Q3 progress / completion** — six-frame WebSocket sequence with per-frame emitters; the dead `progress_callback` and the inaccurate `L238` comment; completion via terminal `SUCCESS` frame + task result string. ✔
- **Q4 final destination & state** — `Document` row, original/archive/thumbnail media, Whoosh entry, admin `LogEntry`, source unlink; raw before/during/after; atomic + FileLock rationale. ✔
- **Q5 already-processed tracking** — the **negative** (no status column) proven by field introspection; four implicit signals evidenced. ✔
- **Q6 duplicate avoidance** — both layers: app md5 pre-check (`It is a duplicate.` / `document_already_exists`) and DB `UNIQUE(checksum)` (`IntegrityError`); byte-exact md5 match. ✔

**Scope note.** This investigation is read-only; the only durable artifact is this document. All temporary observation scripts, sample PDFs, and helper DB rows created for the runs were removed afterward, leaving the repository unchanged apart from this file. Findings (e.g. the absence of a `Document` status field, the absence of a `progress()` caller) are stated as of commit `542221a38dff06361e07976452f9aea24d210542`.
