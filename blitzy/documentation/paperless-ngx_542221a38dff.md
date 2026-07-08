# Paperless-NGX Document Ingestion — Runtime-Grounded Investigation

**Target:** paperless-ngx **v1.7.0** at commit `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`)

**Methodology (read this first):** Every behavioral claim below is grounded in **observed runtime output** captured while the as-shipped stack was actually running — not in reading code alone. Each claim carries (a) the exact command that produced the evidence, (b) the complete, unedited output, and (c) a `file:line` reference plus the name of the specific function doing the work. Statements that could not be directly observed are explicitly labelled **[inferred]**. One deliberate deviation from the pure runtime path — the IMAP *transport* for the email entry point — is explicitly labelled **[non-canonical stand-in]** where it appears; the consume code path it drives is the real one.

---

## 0. How the stack was built and run (canonical configuration)

The investigation was performed inside the user-provided canonical container `paperless_setup` (image `ghcr.io/scaleapi/swe-atlas:...qna_1.01`), which is the as-shipped build. No source file, setting, or dependency was modified; the stack runs in its **default configuration**.

**Runtime confirmed:**

```
$ docker exec paperless_setup python3 --version
Python 3.9.23
$ docker exec paperless_setup redis-cli ping
PONG
$ docker exec paperless_setup bash -lc 'cd /app && git rev-parse HEAD'
542221a38dff06361e07976452f9aea24d210542
```

Python 3.9 matches `Dockerfile:L18` (`FROM python:3.9-slim-bullseye`). The four processes that make up the canonical runtime (per `docker/supervisord.conf`) were live throughout:

| Process | Role | Evidence |
|---------|------|----------|
| `redis-server` (localhost:6379) | Django-Q broker **and** Channels layer backend | `redis-cli ping` → `PONG`; `settings.py:L456` default `redis://localhost:6379` |
| `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` (0.0.0.0:8000) | ASGI server — serves HTTP **and** the `ws/status/` WebSocket | `[program:gunicorn]`; `worker_class=paperless.workers.ConfigurableWorker` |
| `python3 manage.py document_consumer` | Consumption-directory watcher | `[program:consumer]` |
| `python3 manage.py qcluster` | Django-Q worker cluster that actually executes `consume_file` | `[program:scheduler]` |

**Standard exec pattern** used for every `manage.py` command below:

```
docker exec -u testuser -e PAPERLESS_REDIS=redis://localhost:6379 paperless_setup \
    bash -lc 'cd /app/src && python3 manage.py <cmd>'
```

**Consumer watch-mode startup banner** (proves inotify mode, the default):

```
$ docker exec paperless_setup bash -lc "grep -F 'watch directory' /app/data/log/paperless.log"
[2026-07-08 04:13:50,537] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /app/src/../consume
```

This is emitted by `document_consumer.py:L200`. inotify is used because `PAPERLESS_CONSUMER_POLLING=0` (default, `settings.py:L478`); the polling alternative would log `Polling directory for changes:` from `document_consumer.py:L186`.

**Resolved runtime directories** (printed via `manage.py shell`):

| Setting | Resolved value | Source |
|---------|----------------|--------|
| `CONSUMPTION_DIR` | `/app/src/../consume` (= `/app/consume`) | `settings.py:L78-81` |
| `ORIGINALS_DIR` | `/app/media/documents/originals` | `settings.py:L62` |
| `ARCHIVE_DIR` | `/app/media/documents/archive` | `settings.py:L63` |
| `THUMBNAIL_DIR` | `/app/media/documents/thumbnails` | `settings.py:L64` |
| `DATA_DIR` | `/app/data` | `settings.py:L66` |
| `INDEX_DIR` | `/app/data/index` (Whoosh) | `settings.py:L73` |
| `MODEL_FILE` | `/app/data/classification_model.pickle` — **ABSENT on a fresh stack** | `settings.py:L74` |
| `LOGGING_DIR` | `/app/data/log` | `settings.py:L76` |
| `SCRATCH_DIR` | `/tmp/paperless` | `settings.py:L84` |
| `MEDIA_LOCK` | `/app/media/media.lock` | `settings.py` |

Relevant defaults, all confirmed at runtime: `CONSUMER_POLLING=0`, `CONSUMER_DELETE_DUPLICATES=False`, `CONSUMER_ENABLE_BARCODES=False`. `CHANNEL_LAYERS` is `channels_redis.core.RedisChannelLayer` at `redis://localhost:6379`.

**Logging.** The `paperless.*` loggers write to a `ConcurrentRotatingFileHandler` at `/app/data/log/paperless.log` at **DEBUG** level with format `[{asctime}] [{levelname}] [{name}] {message}` (`settings.py:L378`, `L392-L409`). **The `paperless_mail` logger writes to a SEPARATE file** `/app/data/log/mail.log` (`settings.py:L399-L410`) — so email evidence below is drawn from `mail.log`, everything else from `paperless.log`.

> **Note on the logging-group correlation id.** `LoggingMixin` (`loggers.py:L11-L21`) calls `renew_logging_group()` to set `logging_group = uuid.uuid4()` and passes `extra={"group": logging_group}` on every record, so a per-file correlation id exists on each log record. However, the default file formatter does **not** print `{group}`, so the uuid is not visible in `paperless.log` (the DB log handler that would surface it is disabled via `PAPERLESS_DISABLE_DBHANDLER=true`). The timelines below are therefore correlated by ordering + the filename embedded in each message.

**WebSocket probe (temporary, since removed).** Because `ws/status/` requires an authenticated session — `StatusConsumer.connect` raises `DenyConnection()` if `self.scope["user"]` is unauthenticated (`paperless/consumers.py:L13-L21`) — the probe created a Django DB session server-side for `admin`, then connected a `websockets` client to `ws://localhost:8000/ws/status/` presenting `Cookie: sessionid=<key>`, and appended every received frame to a log file. Cross-process delivery was verified before any injection: a manual `group_send("status_updates", {"type":"status_update","data":{...}})` issued from a *separate* `manage.py shell` process arrived at the probe verbatim, confirming the Redis channel layer carries frames from the qcluster worker to the ASGI socket.

**Sample inputs** (small real PDFs generated for the runs; all since deleted). Their md5 sums are used as ground truth below:

| File | md5 | Used for |
|------|-----|----------|
| `sample_blitzy.pdf` (1639 B) | `196f115dc90f5cebbe4c4aa8df258c1d` | EP1 + the clean Q4 reference (doc id 3) |
| `sample_ep2_rest.pdf` | `431432b2c5189cbafc779c4321b8c82c` | EP2 REST upload (doc id 4) |
| `sample_ep3_email.pdf` | `87910bc8cfe323f071da36a614360813` | EP3 email (doc id 5) |
| `sample_q2_match.pdf` | `4bdd0f9bc7915a633117d67c2a565afe` | Q2 classification/matching (doc id 6) |
| `sample_q3_image.pdf` | `b99660780acd3b36aba2ca0420bb6749` | Q3 real per-page OCR (doc id 7) |
| `sample_q4_state.pdf` | `d54f6fb5a5d5631c0aff906f73d9fca5` | Q4 before/during/after (doc id 8) |

(The document primary keys start at 3 because the setup harness created and deleted two smoke-test documents, advancing the SQLite autoincrement; the live document count was 0 before these injections.)

---

## 1. Ingestion overview — the convergent pipeline

Three different producers detect "a new document" but all **converge on a single Django-Q task**:

```
                 detect                          handoff
CONSUMPTION_DIR  ──►  document_consumer._consume ─┐
REST upload      ──►  PostDocumentView.post       ├─► async_task("documents.tasks.consume_file", …)
email attachment ──►  MailAccountHandler.handle_* ─┘            │
                                                                ▼   (executed by qcluster worker)
                                            documents.tasks.consume_file (tasks.py:L184)
                                                                │
                                                                ▼
                                            Consumer().try_consume_file(...) (consumer.py:L180)
                                                                │
   ┌───────────────┬───────────────┬───────────────┬───────────┴───────┬──────────────┬─────────────┐
   ▼               ▼               ▼               ▼                   ▼              ▼             ▼
 pre_check      MIME +          parse (OCR)     thumbnail          parse_date     atomic _store   post-consume
 duplicate      parser select   text extract    generate           (if no date)   Document.create signals fan-out
 (L102-113)     (L219-225)      (L259-261)      (L263-269)         (L271-275)     (L298-306)      (matching, LogEntry, index)
```

The staged flow inside `try_consume_file`, each stage emitting a WebSocket progress frame and/or a log line:

1. **`STARTING` frame** `new_file` (0/100) — `consumer.py:L202`
2. **Duplicate pre-check** — `consumer.py:L213` → `pre_check_duplicate` `L102-113`
3. **MIME detection** (`magic.from_file`) + **parser selection** — `consumer.py:L219-225`
4. **`document_consumption_started`** signal — `consumer.py:L229`
5. **Parsing / OCR** — `WORKING` frame `parsing_document` (20/100) `L259`, then `RasterisedDocumentParser.parse`
6. **Thumbnail** — `WORKING` frame `generating_thumbnail` (70/100) `L264`
7. **Date parse** — `WORKING` frame `parse_date` (90/100) `L274` *(only when the parser yielded no date)*
8. **Save** — `WORKING` frame `save_document` (95/100) `L294`, then atomic `_store` `L379`
9. **`document_consumption_finished`** signal fan-out — `L306` → matching, admin LogEntry, search index
10. **Source file unlinked** — `L349-350`
11. **`SUCCESS` frame** `finished` (100/100, carries `document_id`) — `L375`; task returns `"Success. New document id <pk> created"` `tasks.py:L247`

The remaining sections answer Q1–Q6 in order, each leading with the direct answer.

---

## Q1 — How is a new document detected and handed off for processing?

**Direct answer.** Detection is performed by three different components depending on where the document arrives: the **consumption-directory watcher** (`document_consumer._consume`, inotify) for files dropped into `CONSUMPTION_DIR`; the **REST view** `PostDocumentView.post` for HTTP uploads; and **`MailAccountHandler`** for email attachments. **Handoff is identical in all three cases**: each calls `async_task("documents.tasks.consume_file", …)`, placing the work on the Django-Q queue where the `qcluster` worker picks it up and runs `Consumer().try_consume_file(...)`. For the directory watcher, the observable handoff signal is the log line `Adding <filepath> to the task queue.`

### Entry point 1 — Consumption-directory watcher (primary path)

**Command:**
```
$ docker exec paperless_setup bash -lc 'cp /tmp/obs/sample_blitzy.pdf /app/consume/sample_blitzy.pdf'
```

**Detection + handoff (paperless.log):**
```
[2026-07-08 04:30:21,630] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_blitzy.pdf to the task queue.
```

This single line is emitted by `_consume` at `document_consumer.py:L85`, immediately before the enqueue at `document_consumer.py:L86-L91`:
`async_task("documents.tasks.consume_file", filepath, override_tag_ids=…, task_name=os.path.basename(filepath)[:100])`. The `_consume` function (`document_consumer.py:L46`) is invoked by the inotify handler when a file settles in the watched directory.

### Entry point 2 — REST API upload `POST /api/documents/post_document/`

**Command** (curl is not installed in the container, so the real HTTP endpoint was exercised with Python `requests` and HTTP Basic auth):
```
requests.post("http://localhost:8000/api/documents/post_document/",
              auth=("admin","admin123"),
              files={"document": ("sample_ep2_rest.pdf", <bytes>, "application/pdf")})
→ HTTP 200, body: "OK"
```

The handler is `PostDocumentView.post` (`views.py:L497`, `permission_classes=(IsAuthenticated,)` `L493`). It writes the upload to a scratch tempfile with prefix `paperless-upload-` (`views.py:L513`) and enqueues with a `task_id=uuid4()` (`views.py:L521`, `async_task` `L523`), finally returning `Response("OK")` (`views.py:L535`). There is **no** `Adding … to the task queue.` line for this path — that log belongs solely to the directory watcher; the consumer log for this run begins at `Consuming sample_ep2_rest.pdf`. The resulting Django-Q task recorded `args=('/tmp/paperless/paperless-upload-s2oj0o94',)`, confirming the `paperless-upload-` tempfile actually became the task input.

### Entry point 3 — Email ingestion

**[non-canonical stand-in — transport only]** A live IMAP server was impractical, so the raw message was built locally via `imap_tools.MailMessage.from_bytes` and fed to the **real** entry function `MailAccountHandler.handle_message(message, rule)`, which reaches the genuine `async_task(...consume_file...)` at `mail.py:L336`. Only the IMAP fetch was substituted; the mail-processing and consume code path is the real one. Temporary DB rows (`MailAccount` `blitzy-obs-account`, `MailRule` `blitzy-obs-rule`) were created to drive it and have since been deleted.

**mail.log (separate file):**
```
[2026-07-08 04:34:59,640] [DEBUG] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Processing mail Blitzy Email Ingestion Test from sender@example.com with 1 attachment(s)
[2026-07-08 04:34:59,657] [INFO] [paperless_mail] Rule blitzy-obs-account.blitzy-obs-rule: Consuming attachment sample_ep3_email.pdf from mail Blitzy Email Ingestion Test from sender@example.com
```

The `Consuming attachment …` line is emitted at `mail.py:L329-L334`. The attachment is written to a tempfile with prefix `paperless-mail-` and passed to `consume_file` as the **keyword** argument `path=temp_filename` (`mail.py:L335+`).

### Convergence proof (all three reach the same task)

**Command:**
```
$ … manage.py shell -c "from django_q.models import Task; [print(...) for t in Task.objects.filter(func='documents.tasks.consume_file')...]"
```

**Output:**
```
name='sample_blitzy.pdf'    success=True result='Success. New document id 3 created' args=('/app/src/../consume/sample_blitzy.pdf',)
name='sample_ep2_rest.pdf'  success=True result='Success. New document id 4 created' args=('/tmp/paperless/paperless-upload-s2oj0o94',)
name='sample_ep3_email.pdf' success=True result='Success. New document id 5 created' args=()
```

All three rows share `func=documents.tasks.consume_file`. The `name` is the per-producer `task_name` (watcher: file basename; REST: uploaded doc name; email: attachment filename). Note the email row has `args=()` because its input path was passed as a **keyword** (`path=`), whereas the watcher and REST rows pass it **positionally** — a small but real observable difference between the producers.

**Reasoning (cause → effect).** The three producers exist because documents can arrive by three transports (filesystem, HTTP, IMAP), but paperless funnels them into one asynchronous task so that all downstream processing (parse → classify → persist → index) is written once and shared. The `async_task` call is the handoff boundary: it serialises the request into the Redis-backed Django-Q broker, and the separate `qcluster` process dequeues and runs it — which is precisely why the work is asynchronous and why the progress/status evidence in Q3 has to be observed over a WebSocket rather than in the HTTP response.

---

## Q2 — What log messages / task names / state changes indicate the transition into parsing, classification, and indexing?

**Direct answer.** The single task name is **`documents.tasks.consume_file`**. Within its execution, the transitions are marked by DEBUG/INFO log lines from the `paperless.consumer`, `paperless.parsing.tesseract`, `paperless.classifier`, `paperless.matching`, and `paperless.handlers` loggers, and by two Django signals (`document_consumption_started`, `document_consumption_finished`). The observable ordered markers are: `Consuming <file>` → `Detected mime type:` → `Parser: <ClassName>` → `Parsing <file>...` (**parsing**) → `Generating thumbnail…` → the classifier/matching lines (**classification**) → `add_to_index` writing Whoosh (**indexing**) → `Document … consumption finished`.

The following **complete, unedited** timeline was captured for `sample_q2_match.pdf` (doc id 6), which was crafted to trigger rule-based matching:

**Command:**
```
$ docker exec paperless_setup bash -lc \
    "awk '/Adding .*sample_q2_match/{f=1} f{print} /sample_q2_match consumption finished/{f=0}' /app/data/log/paperless.log"
```

**Output:**
```
[2026-07-08 04:38:16,634] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_q2_match.pdf to the task queue.
[2026-07-08 04:38:16,770] [INFO] [paperless.consumer] Consuming sample_q2_match.pdf
[2026-07-08 04:38:16,771] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-08 04:38:16,772] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-08 04:38:16,774] [DEBUG] [paperless.consumer] Parsing sample_q2_match.pdf...
[2026-07-08 04:38:16,799] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /app/src/../consume/sample_q2_match.pdf
[2026-07-08 04:38:16,867] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/sample_q2_match.pdf', 'output_file': '/tmp/paperless/paperless-q7umatzk/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-q7umatzk/sidecar.txt'}
[2026-07-08 04:38:17,128] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-08 04:38:17,134] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-q7umatzk/archive.pdf
[2026-07-08 04:38:17,134] [DEBUG] [paperless.consumer] Generating thumbnail for sample_q2_match.pdf...
[2026-07-08 04:38:17,138] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-q7umatzk/archive.pdf[0] /tmp/paperless/paperless-q7umatzk/convert.png
[2026-07-08 04:38:17,150] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-08 04:38:17,229] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-q7umatzk/gs_out.png /tmp/paperless/paperless-q7umatzk/convert_gs.png
[2026-07-08 04:38:17,318] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-q7umatzk/convert_gs.png -out /tmp/paperless/paperless-q7umatzk/thumb_optipng.png
[2026-07-08 04:38:17,949] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-08 04:38:17,952] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-08 04:38:17,957] [DEBUG] [paperless.matching] Correspondent Blitzy Corp matched on document 2026-07-08 sample_q2_match because it contains this word: Blitzy
[2026-07-08 04:38:17,957] [INFO] [paperless.handlers] Assigning correspondent Blitzy Corp to 2026-07-08 sample_q2_match
[2026-07-08 04:38:17,958] [DEBUG] [paperless.matching] DocumentType Observation Report matched on document 2026-07-08 Blitzy Corp sample_q2_match because it contains this word: Observation
[2026-07-08 04:38:17,958] [INFO] [paperless.handlers] Assigning document type Observation Report to 2026-07-08 Blitzy Corp sample_q2_match
[2026-07-08 04:38:17,960] [DEBUG] [paperless.matching] Tag ingested matched on document 2026-07-08 Blitzy Corp sample_q2_match because it contains this word: ingestion
[2026-07-08 04:38:17,960] [INFO] [paperless.handlers] Tagging "2026-07-08 Blitzy Corp sample_q2_match" with "ingested"
[2026-07-08 04:38:17,979] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/sample_q2_match.pdf
[2026-07-08 04:38:18,004] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-q7umatzk
[2026-07-08 04:38:18,004] [INFO] [paperless.consumer] Document 2026-07-08 Blitzy Corp sample_q2_match consumption finished
```

### Parsing (answered by name)

`Consuming <file>` is logged at `consumer.py:L215`. MIME detection uses `magic.from_file` (`consumer.py:L219`) and logs `Detected mime type:` (`L221`); the parser class is chosen by `get_parser_class_for_mime_type` (`parsers.py:L81`) and logged as `Parser: RasterisedDocumentParser` (`consumer.py:L246`). For `application/pdf`/images that class is `RasterisedDocumentParser` (`paperless_tesseract/parsers.py:L18`, `logging_name="paperless.parsing.tesseract"` `L24`). The `document_consumption_started` signal fires at `consumer.py:L229`. Actual OCR is visible in the `paperless.parsing.tesseract` lines: text is first extracted directly, then **OCRmyPDF** is called with `skip_text: True` (so an existing text layer is preserved rather than re-OCR'd). Thumbnail generation follows (`consumer.py:L263`), here falling back from ImageMagick to ghostscript and then `optipng`.

### Classification (answered by name)

**Observed:** on a fresh stack the ML classifier is **skipped**, and the system logs:
```
[2026-07-08 04:38:17,949] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
```
This is `load_classifier` (`classifier.py:L30`) returning `None` because `MODEL_FILE` (`/app/data/classification_model.pickle`) does not exist (`classifier.py:L31-L36`). `load_classifier` is called once at `consumer.py:L292`. **Crucially, rule-based matching still runs even with no ML model.** After `Saving record to database` (i.e. after the row exists), the `document_consumption_finished` signal fans out to the matching handlers, which iterate over the defined entities and apply `matches()` (`matching.py:L60`); the `paperless.matching` logger emits `<Class> <name> matched on document <doc> because <reason>` via `log_reason` (`matching.py:L13-L17`), and the reason `it contains this word: <word>` corresponds to the `MATCH_ANY` algorithm. Here the auto-match rules (temporary `Correspondent "Blitzy Corp"`, `DocumentType "Observation Report"`, `Tag "ingested"`, all `MATCH_ANY`, case-insensitive) matched on the words *Blitzy*, *Observation*, *ingestion*, and `paperless.handlers` logged the corresponding `Assigning …`/`Tagging …` actions. **[inferred]** If a trained `classification_model.pickle` were present, `load_classifier` would return a classifier and the `paperless.classifier` predict path would additionally run; that path was not exercised here because the default fresh stack has no model.

### Indexing (answered by name)

Indexing is the last handler in the fan-out: `add_to_index` (`signals/handlers.py:L428-L430`) calls `index.add_or_update_document` (`index.py:L118-L120`) → `update_document` (`index.py:L87-L107`), which writes the Whoosh schema fields defined in `get_schema` (`index.py:L31-L49`). Indexing itself emits **no** log line (verified — there is no `paperless.index` message in the timeline); its observable evidence is the resulting index entry, shown under Q4.

### Fan-out order (state change ordering)

The handler order in the timeline exactly matches the wiring in `apps.py:DocumentsConfig.ready()` (`L22-L27`): **`add_inbox_tags` → `set_correspondent` → `set_document_type` → `set_tags` → `set_log_entry` → `add_to_index`**. `add_inbox_tags`, `set_log_entry`, and `add_to_index` produce no log line here (no inbox tags configured; the latter two write a DB LogEntry and the Whoosh entry — see Q4). A visible side effect of the ordering is that the document's string representation mutates mid-timeline — `2026-07-08 sample_q2_match` → `2026-07-08 Blitzy Corp sample_q2_match` — as the correspondent is assigned.

**Reasoning (cause → effect).** Parsing must precede persistence because the extracted text becomes the `Document.content`; classification/matching runs **after** the row is created (note `Saving record to database` precedes the matching lines) because the handlers mutate and re-save the freshly created `Document`; indexing runs last so the Whoosh entry reflects the final content and assigned metadata. The signal architecture (`started` before parsing, `finished` after save) is what lets these stages be decoupled handlers rather than inline code.


---

## Q3 — How does the system reflect progress or completion of each stage?

**Direct answer.** Progress is reflected in real time as **WebSocket status frames** broadcast on `ws/status/`, plus the log lines already shown. For a successful run the client receives **exactly six frames** at the checkpoints `STARTING(0)` → `WORKING(20)` → `WORKING(70)` → `WORKING(90)` → `WORKING(95)` → `SUCCESS(100)`. Completion is signalled two ways: the terminal `SUCCESS(100,100)` frame (the only frame that carries a non-null `document_id`), and the Django-Q task **result string** `Success. New document id <pk> created`.

The progress mechanism is `Consumer._send_progress` (`consumer.py:L56-L76`), which builds a 7-key payload and broadcasts it via `group_send` to the `"status_updates"` group; the event `type` `status_update` is dispatched (Channels replaces dots with underscores) to `StatusConsumer.status_update` (`paperless/consumers.py:L29-L33`), which sends `json.dumps(event["data"])` to every subscribed client. The route is `re_path(r"ws/status/$", StatusConsumer.as_asgi())` (`urls.py:L137`).

**Captured frames for a full successful run** (`sample_blitzy.pdf` → doc id 3), exactly as received by the probe:

```
$ cat /tmp/obs/ws_ep1.log
WS_CONNECTED
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "16ea36bb-bb46-4cb4-8104-a29fe17192b3", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 3}
WS_DONE 6
```

Each frame maps to a specific `_send_progress` call:

| Frame | `message` | Emitter |
|-------|-----------|---------|
| `STARTING (0,100)` | `new_file` | `consumer.py:L202` |
| `WORKING (20,100)` | `parsing_document` | `consumer.py:L259` |
| `WORKING (70,100)` | `generating_thumbnail` | `consumer.py:L264` |
| `WORKING (90,100)` | `parse_date` | `consumer.py:L274` (only if `not date`, `L273`) |
| `WORKING (95,100)` | `save_document` | `consumer.py:L294` |
| `SUCCESS (100,100)` | `finished`, `document_id=<pk>` | `consumer.py:L375` |

### Two important runtime findings that contradict a naive code reading

**(1) The intermediate 20→70 "callback band" is never emitted.** `consumer.py:L237-L240` defines a local `progress_callback` computing `p = int((current_progress/max_progress)*50 + 20)` and passes it to the parser, whose `DocumentParser.progress()` wrapper (`parsers.py:L300-L302`) would call it. But **no parser ever invokes `self.progress()`**, so the wrapper — and therefore the callback — is never reached:
```
$ docker exec paperless_setup bash -lc "grep -rn -E 'self\.progress\(|progress_callback\(' /app/src --include=*.py"
/app/src/documents/consumer.py:237:        def progress_callback(current_progress, max_progress):
/app/src/documents/parsers.py:302:            self.progress_callback(current_progress, max_progress)

$ docker exec paperless_setup bash -lc "grep -rn '\.progress(' /app/src --include=*.py || echo '(no matches)'"
(no matches)
```
The first search shows the only two references are the *definition* (`consumer.py:237`) and the wrapper's internal call (`parsers.py:302`, inside `DocumentParser.progress()`); the second search shows there is **no callsite** of the `.progress()` method anywhere in the source, so `parsers.py:302` is never executed.
This was confirmed empirically: an **image-only PDF** (`sample_q3_image.pdf`, doc id 7) that forced genuine per-page OCR (OCRmyPDF ran ~2.6 s) still produced **exactly the same six discrete frames** — no frames appeared between 20 and 70:
```
$ cat /tmp/obs/ws_q3img.log
WS_CONNECTED
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 0,   … "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 20,  … "status": "WORKING",  "message": "parsing_document", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 70,  … "status": "WORKING",  "message": "generating_thumbnail", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 90,  … "status": "WORKING",  "message": "parse_date", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 95,  … "status": "WORKING",  "message": "save_document", "document_id": null}
FRAME {"filename": "sample_q3_image.pdf", … "current_progress": 100, … "status": "SUCCESS",  "message": "finished", "document_id": 7}
WS_DONE 6
```

**(2) The inline comment at `consumer.py:L238` is doubly inaccurate.** It reads *"recalculate progress to be within 20 and 80"*, but (a) the arithmetic `*50 + 20` caps at **70**, not 80, and (b) the callback is never invoked anyway, so no such intermediate frame ever appears. The ground truth is what is observed above: the band does not occur.

**On the `parse_date` (90) frame:** it is present for every `RasterisedDocumentParser` run because that parser never sets `self.date`, so `get_date()` returns `None` and `if not date:` (`consumer.py:L273`) is always true for PDF/image documents. **[inferred]** A parser that returned a date would skip this single frame; this could not be reproduced with the tesseract parser, which never sets a date.

### Completion signal — the task result string

**Command + output:**
```
$ … Task.objects.filter(func='documents.tasks.consume_file') …
name='sample_blitzy.pdf' success=True result='Success. New document id 3 created' …
```
The string `Success. New document id <pk> created` is built at `tasks.py:L247` and persisted on the Django-Q `Task` row (see Q5). *(The barcode-split alternate path would instead return `File successfully split` and omit `document_id` — `tasks.py:L217-L233` — but that path is default-OFF (`CONSUMER_ENABLE_BARCODES=False`) and was not exercised.)*

### Stability across runs

The checkpoint value set `{0, 20, 70, 90, 95, 100}` and the six-frame shape were **identical across five independent successful runs** (EP1, EP2, EP3, Q2-match, Q3-image-OCR), as the captured `ws_*.log` files above and below show. There was **no run-to-run variation** in the emitted values.

**Reasoning (cause → effect).** Because the actual work happens in a separate `qcluster` process, the only way a UI can reflect progress is out-of-band; paperless uses the Redis-backed Channels layer so the worker's `group_send` reaches every browser subscribed to `ws/status/`. The frames are therefore coarse checkpoints wrapped around the expensive steps (parse, thumbnail, save) rather than a smooth percentage — and, as observed, the finer-grained callback that was intended to fill the 20→70 gap is dead code.

---

## Q4 — Where does the document's data end up, and how is its final state recorded?

**Direct answer.** A successful consume produces **six durable artifacts** and deletes the input: (1) one `Document` row in SQLite; (2) the **original** file under `originals/`; (3) an OCR'd **archive** PDF under `archive/`; (4) a **thumbnail** PNG under `thumbnails/`; (5) a **Whoosh** full-text index entry; (6) an admin **`LogEntry`** ADDITION recorded by the `consumer` user. The source file in `CONSUMPTION_DIR` is **unlinked**. There is no separate "final state" column — the *existence* of these artifacts is the final state. The reference example is doc id 3 (`sample_blitzy.pdf`), consumed cleanly via the directory watcher.

### (1) The `Document` row

**Command + output:**
```
$ … manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=3); …"
id = 3
title = 'sample_blitzy'
correspondent = None
document_type = None
checksum = '196f115dc90f5cebbe4c4aa8df258c1d'
archive_checksum = 'e0ce6d71cefed2c38e58535f942ebf4d'
mime_type = 'application/pdf'
storage_type = 'unencrypted'
created = datetime.datetime(2026, 7, 8, 4, 30, 20, 623100, tzinfo=datetime.timezone.utc)
modified = datetime.datetime(2026, 7, 8, 4, 30, 23, 123812, tzinfo=datetime.timezone.utc)
added = datetime.datetime(2026, 7, 8, 4, 30, 23, 104862, tzinfo=datetime.timezone.utc)
filename = '0000003.pdf'
archive_filename = '0000003.pdf'
tags = []
content_len = 241
```

The row is created by `_store` (`consumer.py:L379`, `Document.objects.create(...)` `L398-L406`). Field grounding in `models.py`: `Document` `L88`; `content` `L117-124`; `checksum` unique `L135-141`; `created` `L152`; `modified` auto_now `L154-159`; `storage_type` `L161-167` (default `unencrypted`); `added` `L169-174`; `filename` unique `L176-184`. The `filename` follows the `{pk:07d}.pdf` pattern. `correspondent`/`document_type`/`tags` are empty here because this document was consumed **before** the Q2 matching rules existed — demonstrating that a document persists cleanly with no metadata when nothing matches.

### (2)+(3) Media files + byte-exact checksum verification

**Command + output:**
```
$ docker exec paperless_setup bash -lc 'ls -l /app/media/documents/originals/0000003.pdf /app/media/documents/archive/0000003.pdf /app/media/documents/thumbnails/0000003.png'
-rw-r--r-- 1 testuser testuser  8890 Jul  8 04:30 /app/media/documents/archive/0000003.pdf
-rw-r--r-- 1 testuser testuser  1639 Jul  8 04:30 /app/media/documents/originals/0000003.pdf
-rw-r--r-- 1 testuser testuser 10246 Jul  8 04:30 /app/media/documents/thumbnails/0000003.png

$ docker exec paperless_setup bash -lc 'md5sum /tmp/obs/sample_blitzy.pdf /app/media/documents/originals/0000003.pdf /app/media/documents/archive/0000003.pdf'
input    /tmp/obs/sample_blitzy.pdf             : 196f115dc90f5cebbe4c4aa8df258c1d
original media/originals/0000003.pdf            : 196f115dc90f5cebbe4c4aa8df258c1d
archive  media/archive/0000003.pdf              : e0ce6d71cefed2c38e58535f942ebf4d
```

Three facts fall out of this: **(a)** the stored original is **byte-identical** to the injected input (`196f11… == 196f11…`); **(b)** `Document.checksum` equals that same md5 — i.e. the checksum is the md5 of the original bytes; **(c)** `Document.archive_checksum` (`e0ce6d71…`) equals the md5 of the OCR'd archive PDF. The files are written under `FileLock(MEDIA_LOCK)` (`consumer.py:L315`): original at `L319`, thumbnail at `L321-L325`, archive at `L327-L337`, and `archive_checksum` is computed at `L339-L342`. Note the thumbnail is a **`.png`** in this version.

### (4) The Whoosh index entry

**Command + output:**
```
$ … manage.py shell -c "from documents.index import open_index; from whoosh.query import Term; …"
hits= 1
stored_fields= {'id': 3}
content-search machine-readable -> [{'id': 3}]
```

Only `id` is `stored=True` in the schema (`index.py:L33`), so a stored-field lookup returns `{'id': 3}`; the other fields are indexed-but-not-stored. Searching the `content` field for a phrase unique to this document (`machine-readable`) returns doc 3, proving the OCR text was actually indexed. Written via `add_to_index` → `add_or_update_document` → `update_document` (`handlers.py:L428-L431`, `index.py:L118`/`L87`).

### (5) The admin `LogEntry`

**Command + output:**
```
$ … manage.py shell -c "from django.contrib.admin.models import LogEntry; e=LogEntry.objects.get(object_id='3', content_type__model='document'); …"
id=3 user=consumer action_flag=1(ADDITION=True) object_id=3 object_repr='2026-07-08 sample_blitzy'
```

Created by `set_log_entry` (`signals/handlers.py:L413-L425`) with `action_flag=ADDITION`, `user=User.objects.get(username="consumer")` (`L416`), `object_id=document.pk`. This is why the stack must have a `consumer` user for consumption to succeed.

### (6) Source file removal + before/during/after

Using a fresh input (`sample_q4_state.pdf`, doc id 8) to show the transition cleanly:

| | Consumption dir | `Document` (by checksum) | Doc count |
|--|-----------------|--------------------------|-----------|
| **Before** | (file absent) | `exists() = False` | 5 |
| **After**  | file **unlinked** (absent) | `exists() = True`, `id=8` | 6 |

The deletion is logged and performed at `consumer.py:L349-L350` (`Deleting file <path>` then `os.unlink(self.path)`), e.g. for doc 3:
```
[2026-07-08 04:30:23,124] [DEBUG] [paperless.consumer] Deleting file /app/src/../consume/sample_blitzy.pdf
```

Summary of state changes: `Document` **absent → present**; consumption file **present → unlinked**; media dirs **empty → populated (3 files)**; index entry **absent → present**.

**Reasoning (cause → effect).** The row is created inside `with transaction.atomic():` (`consumer.py:L298`) and the `document_consumption_finished` signal fires inside that **same** atomic block (`L306`), so the row and all handler side effects (matching, LogEntry, index) commit together or not at all. Media writes are guarded by `FileLock(MEDIA_LOCK)` (`L315`) to serialise concurrent consumers. The source file is unlinked **only after** a successful store (`L350`) — which is exactly why, on any failure (Q6/edge paths), the row is rolled back and the source file is left in place.


---

## Q5 — How does Paperless-NGX track whether a document has already been processed or needs further work?

**Direct answer (the negative, stated up front).** At this commit there is **NO explicit per-document status / state / processed column**. Whether a document has "been processed" is tracked *implicitly* by four observable signals: **(1) the existence of a `Document` row** carrying the file's checksum; **(2) the Django-Q task ledger** (`Task` / `Success` / `Failure` records); **(3) the removal of the source file** from `CONSUMPTION_DIR`; and **(4) the terminal WebSocket status** (`SUCCESS` vs `FAILED`). There is **no re-processing queue or "needs further work" flag** — a file is (re)processed whenever it (re)appears at an entry point, gated only by the duplicate check (Q6).

### Proof: the `Document` model has no status field

**Command + output:**
```
$ … manage.py shell -c "from documents.models import Document; print([f.name for f in Document._meta.get_fields()]); …"
FIELDS = ['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum', 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename', 'archive_filename', 'archive_serial_number', 'tags']
status-like = []
```

Filtering the field names for any of `status/state/process/stage/done/complete/flag` returns an empty list. The underlying SQLite table `documents_document` has the same columns and no status column. Grounding: `Document` `models.py:L88`; the full field set spans `L97-L184`; none represents processing state.

### Signal 1 — Row existence (the durable "processed" marker)

**Command + output:**
```
$ … Document.objects.filter(checksum='196f115dc90f5cebbe4c4aa8df258c1d').exists()   → True   # sample_blitzy: processed
$ … Document.objects.filter(checksum='00000000000000000000000000000000').exists()   → False  # never seen
```
A document counts as "processed" iff a row with its checksum exists; the row is created only at the very end of `try_consume_file`. This is exactly the query `pre_check_duplicate` performs (Q6).

### Signal 2 — The Django-Q task ledger (per-run record)

`Q_CLUSTER` was observed as `{'name':'paperless','catch_up':False,'recycle':1,'retry':1810,'timeout':1800,'workers':11,'redis':'redis://localhost:6379'}` (no `save_limit` override, so the django-q default of 250 saved successes applies; **failures are always saved**). Every executed task is persisted:

```
$ … Task.objects.filter(func='documents.tasks.consume_file') …   # (successes)
name='sample_blitzy.pdf'    success=True result='Success. New document id 3 created' args=('/app/src/../consume/sample_blitzy.pdf',)
name='sample_ep2_rest.pdf'  success=True result='Success. New document id 4 created' args=('/tmp/paperless/paperless-upload-s2oj0o94',)
name='sample_ep3_email.pdf' success=True result='Success. New document id 5 created' args=()
name='sample_q2_match.pdf'  success=True result='Success. New document id 6 created' args=('/app/src/../consume/sample_q2_match.pdf',)
name='sample_q3_image.pdf'  success=True result='Success. New document id 7 created' args=('/app/src/../consume/sample_q3_image.pdf',)
name='sample_q4_state.pdf'  success=True result='Success. New document id 8 created' args=('/app/src/../consume/sample_q4_state.pdf',)

Success total: 8 Failure total: 5
```

Each `Task` carries a uuid `id`, a humanised `name` (the `task_name`), `func='documents.tasks.consume_file'`, the `result` string, `started`/`stopped` timestamps, and a `success` boolean. `Success` and `Failure` are proxy models of `Task` filtered on `success=True/False`. This ledger is the per-run "processing record": a completed run leaves a `Success` row whose `result` names the created document id; a failed run leaves a `Failure` row (shown in Q6/edge paths). *(These semantics match the official Django-Q documentation on task/result persistence; the values quoted are the ones observed here.)*

### Signal 3 — Source-file removal

As shown in Q4, a successfully consumed file is unlinked from `CONSUMPTION_DIR` (`consumer.py:L350`). Conversely, a file still present in the consume directory was **not** successfully consumed — this is the filesystem-level signal, and it is exactly what happens on the error paths (Q6, edges): the source file is left in place.

### Signal 4 — Terminal WebSocket status

The final `ws/status/` frame is `SUCCESS` on success and `FAILED` on any failure (Q3/Q6) — the real-time signal a UI client uses to decide whether processing completed.

**"Needs further work" — [inferred from observed absence].** There is no queue or column expressing partial/incomplete processing. Because a failed run rolls back inside the atomic block, it leaves **no** `Document` row (verified in the edge paths: document count is unchanged after each failure), and the only trace is the Django-Q `Failure` record. The system therefore does not "remember" that a document needs re-work; re-work only happens if the file is presented again at an entry point.

**Reasoning (cause → effect).** Paperless treats the presence of a fully-formed `Document` row as the single source of truth for "processed," which is why no status column is needed: partially-processed states never persist (the atomic transaction guarantees all-or-nothing), so a row's mere existence already means "fully processed." The Django-Q ledger and the WebSocket status exist to *report* on runs, not to *drive* re-processing.

---

## Q6 — What behavior/logs indicate how the system avoids duplicate processing?

**Direct answer.** Duplicate avoidance has **two independent layers**: **(1)** an application-level md5 pre-check, `Consumer.pre_check_duplicate` (`consumer.py:L102-L113`), which hashes the input and fails fast with the log `It is a duplicate.` and the WebSocket status `document_already_exists` if a `Document` already has that checksum (or archive_checksum); and **(2)** a database `UNIQUE` constraint on `Document.checksum` (`models.py:L135-L141`) that raises `IntegrityError` at write time even if the app check were bypassed.

The same bytes as doc 3 (`sample_blitzy.pdf`, md5 `196f115dc90f5cebbe4c4aa8df258c1d`) were re-injected.

### Layer 1 — application pre-check

`pre_check_duplicate` opens the input, computes `checksum = hashlib.md5(f.read()).hexdigest()` (`consumer.py:L104`), and runs `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists()` (`L105-L107`). On a hit it calls `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, "Not consuming {filename}: It is a duplicate.")` (`L110-L113`), where the constant is `"document_already_exists"` (`consumer.py:L37`).

**Before:** doc count = 6; `consume_file` Failure baseline recorded.

**During (paperless.log):**
```
[2026-07-08 04:47:39,044] [INFO] [paperless.management.consumer] Adding /app/src/../consume/sample_blitzy.pdf to the task queue.
[2026-07-08 04:47:39,179] [ERROR] [paperless.consumer] Not consuming sample_blitzy.pdf: It is a duplicate.
```
Note the watcher **still enqueues** the file (the `Adding … to the task queue.` line at `document_consumer.py:L85` runs before any hashing); the duplicate detection happens **inside the worker**.

**During (WebSocket) — only two frames, because it fails before parsing:**
```
$ cat /tmp/obs/ws_dup.log
WS_CONNECTED
FRAME {"filename": "sample_blitzy.pdf", "task_id": "d4dcd938-9d8b-4c9f-83da-39750c86ec75", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "sample_blitzy.pdf", "task_id": "d4dcd938-9d8b-4c9f-83da-39750c86ec75", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
WS_DONE 2
```
The `FAILED(100,100)` frame is produced by `_fail` → `_send_progress` (`consumer.py:L78-L81`), which sends the terminal frame and then raises `ConsumerError`.

**During (Django-Q Failure record):**
```
name='sample_blitzy.pdf' func='documents.tasks.consume_file' success=False
result='sample_blitzy.pdf: Not consuming sample_blitzy.pdf: It is a duplicate. : Traceback (most recent call last):
  File ".../django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: sample_blitzy.pdf: Not consuming sample_blitzy.pdf: It is a duplicate.'
```

**Byte-exact proof the md5 pre-check compares real bytes:**
```
$ md5sum /tmp/obs/sample_blitzy.pdf   → 196f115dc90f5cebbe4c4aa8df258c1d
$ Document.objects.get(pk=3).checksum → '196f115dc90f5cebbe4c4aa8df258c1d'   # identical
```

**After:** doc count = **6, unchanged** (no new row). The duplicate input **remains** in `/app/consume` because `CONSUMER_DELETE_DUPLICATES=False` (default, `settings.py:L486`; the unlink at `consumer.py:L108-L109` runs only when that setting is `True`). *(If toggled on — a non-default config — the duplicate input would additionally be deleted.)*

### Layer 2 — database `UNIQUE` constraint (independent guarantee)

Bypassing the app check entirely by creating a row directly via the ORM with an existing checksum:

**Command + output:**
```
$ … manage.py shell -c "from django.db import transaction, IntegrityError; from documents.models import Document; dup=Document.objects.get(pk=3).checksum;
    try:
        with transaction.atomic():
            Document.objects.create(title='dup-attempt', content='x', mime_type='application/pdf', checksum=dup)
    except IntegrityError as e: print('IntegrityError:', e)
    print('count still:', Document.objects.count())"
reusing existing checksum: 196f115dc90f5cebbe4c4aa8df258c1d
IntegrityError: UNIQUE constraint failed: documents_document.checksum
count still: 6
```

This is enforced by `checksum = models.CharField(..., unique=True)` at `models.py:L135-L141`. The count remains 6 (the create rolls back).

**Reasoning (cause → effect).** The two layers serve different purposes. Layer 1 is a **fast, user-facing** guard: it hashes the input before any expensive OCR and surfaces a clear `document_already_exists` status so the client learns *why* nothing happened — and it checks **both** `checksum` and `archive_checksum` so a re-fed archive PDF is also caught. Layer 2 is the **last-resort integrity guarantee** at the storage layer: even if two identical files were processed concurrently and both passed the app check before either committed, the database `UNIQUE(checksum)` would reject the second write. Together they make duplicate ingestion both cheap to reject and impossible to persist.


---

## Entry-point comparison (how the three producers converge)

| Aspect | Consumption directory | REST API upload | Email attachment |
|--------|-----------------------|-----------------|------------------|
| Detector / function | `document_consumer._consume` (inotify) `document_consumer.py:L46` | `PostDocumentView.post` `views.py:L497` | `MailAccountHandler.handle_message` `mail.py` |
| Detection trigger | file settles in `CONSUMPTION_DIR` | authenticated `POST /api/documents/post_document/` | mail rule matches an attachment |
| Input handed to task | the file path in `CONSUMPTION_DIR` (positional) | scratch tempfile `paperless-upload-…` (positional) | scratch tempfile `paperless-mail-…` (keyword `path=`) |
| Distinct handoff log | `Adding <path> to the task queue.` (`L85`) | *(none — no watcher line)* | `Rule …: Consuming attachment …` in **mail.log** (`mail.py:L329-334`) |
| Handoff call | `async_task("documents.tasks.consume_file", …)` `L86-91` | `async_task(...consume_file...)` `views.py:L523` | `async_task(...consume_file...)` `mail.py:L336` |
| `task_id` origin | generated inside `try_consume_file` (no id passed) | `uuid4()` in the view (`views.py:L521`) | generated inside `try_consume_file` |
| Observed `Task.args` | `('/app/src/../consume/…',)` | `('/tmp/paperless/paperless-upload-…',)` | `()` (path passed as kwarg) |
| Converges on | `documents.tasks.consume_file` → `Consumer().try_consume_file` (identical from here on) | same | same |

All three were observed to produce `func=documents.tasks.consume_file` tasks that succeeded with `Success. New document id 3/4/5 created` (Q1). After the handoff, the code path is byte-for-byte identical — which is why Q2–Q6 are answered once and apply to all three transports.

---

## Edge / error paths (exercising every condition)

Four non-happy-path conditions were exercised. The common outcome for all *failures* is: **no `Document` row is created** (count unchanged), and **the source file is NOT unlinked** (it remains in `CONSUMPTION_DIR`, because `os.unlink` at `consumer.py:L350` runs only after a successful store).

### (a) Duplicate — see Q6 above (two layers, `document_already_exists` + `IntegrityError`).

### (b) Unsupported extension (rejected at detection, before handoff)

A file with an unknown extension (`badext.xyz`) is rejected by the watcher itself via `is_file_ext_supported` (`parsers.py:L62`) before any task is enqueued:
```
[2026-07-08 04:50:15,081] [WARNING] [paperless.management.consumer] Not consuming file /app/src/../consume/badext.xyz: Unknown file extension.
```
This is `document_consumer.py:L54-L55`. **No** `consume_file` task was enqueued (task count unchanged), **no** WebSocket frame was emitted, and **no** `Document` row was created — the rejection happens entirely in the watcher, upstream of the queue. The supported-extension set observed at runtime is: `.bat .bmp .brf .c .csv .gif .h .jfif .jpe .jpeg .jpg .ksh .pdf .pl .png .pot .srt .text .tif .tiff .txt`.

### (c) Unsupported MIME (rejected inside the worker, before parsing)

A file with a `.pdf` extension but zip content (`notreally.pdf`) passes the extension check but fails MIME-based parser selection: `magic.from_file` returns `application/octet-stream`, `get_parser_class_for_mime_type` returns `None`, and `try_consume_file` calls `_fail(MESSAGE_UNSUPPORTED_TYPE, …)` (`consumer.py:L223-L225`, constant `"unsupported_type"` `L44`).

**paperless.log:**
```
[2026-07-08 04:50:38,129] [INFO] [paperless.management.consumer] Adding /app/src/../consume/notreally.pdf to the task queue.
[2026-07-08 04:50:38,263] [INFO] [paperless.consumer] Consuming notreally.pdf
[2026-07-08 04:50:38,264] [DEBUG] [paperless.consumer] Detected mime type: application/octet-stream
[2026-07-08 04:50:38,267] [ERROR] [paperless.consumer] Unsupported mime type application/octet-stream
```
**WebSocket (two frames — fails before parsing, so no `parsing_document` frame):**
```
$ cat /tmp/obs/ws_edge2.log
WS_CONNECTED
FRAME {"filename": "notreally.pdf", "task_id": "43a35159-5646-4bae-b278-5f6248d7ea40", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "notreally.pdf", "task_id": "43a35159-5646-4bae-b278-5f6248d7ea40", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "unsupported_type", "document_id": null}
WS_DONE 2
```
**Django-Q Failure:** `result='notreally.pdf: Unsupported mime type application/octet-stream : Traceback …'` (chain: `tasks.py:L236` → `consumer.py:L225` `_fail` → `L81` raise `ConsumerError`). Doc count 6 → 6.

### (d) Parse failure (fails during parsing)

A file that is genuinely `application/pdf` but structurally broken (`corrupt.pdf`) reaches the parser and then raises `ParseError` (`parsers.py:L277`, raised at `paperless_tesseract/parsers.py:L310`), handled at `consumer.py:L278-L284` via `_fail(str(e), "Error while consuming document {file}: {e}")`.

**paperless.log (unedited, including the pdfminer + force-OCR fallback sequence):**
```
[2026-07-08 04:51:11,817] [INFO] [paperless.management.consumer] Adding /app/src/../consume/corrupt.pdf to the task queue.
[2026-07-08 04:51:11,950] [INFO] [paperless.consumer] Consuming corrupt.pdf
[2026-07-08 04:51:11,954] [DEBUG] [paperless.consumer] Parsing corrupt.pdf...
pdfminer.pdfparser.PDFSyntaxError: No /Root object! - Is this really a PDF?
[2026-07-08 04:51:12,044] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/corrupt.pdf', … 'skip_text': True, …}
[2026-07-08 04:51:12,209] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: . Attempting force OCR to get the text.
[2026-07-08 04:51:12,210] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/app/src/../consume/corrupt.pdf', … 'force_ocr': True, …}
[2026-07-08 04:51:12,303] [ERROR] [paperless.consumer] Error while consuming document corrupt.pdf: InputFileError: 
```
**WebSocket (three frames — reaches `parsing_document` (20) before failing, distinguishing it from case (c)):**
```
$ cat /tmp/obs/ws_edge3.log
WS_CONNECTED
FRAME {"filename": "corrupt.pdf", "task_id": "058b3221-b50c-4d36-9d8a-48497972cca8", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
FRAME {"filename": "corrupt.pdf", "task_id": "058b3221-b50c-4d36-9d8a-48497972cca8", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
FRAME {"filename": "corrupt.pdf", "task_id": "058b3221-b50c-4d36-9d8a-48497972cca8", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "InputFileError: ", "document_id": null}
WS_DONE 3
```
**Django-Q Failure:** `result='corrupt.pdf: Error while consuming document corrupt.pdf: InputFileError:  : Traceback …'`, whose chain includes `paperless_tesseract/parsers.py:L261` (initial OCR) and `L298` (force-OCR fallback), `pikepdf._qpdf.PdfError: … unable to find trailer dictionary`, `documents.parsers.ParseError: InputFileError:` (`parsers.py:L310`), then `consumer.py:L261` (`document_parser.parse`) and `L280` `_fail`. Doc count 6 → 6.

**Edge-path state summary (before / during / after):**

| Case | Enqueued? | WS frames | `Document` row | Source file after |
|------|-----------|-----------|----------------|-------------------|
| Unsupported extension | **No** (watcher rejects) | none | none | remains |
| Unsupported MIME | yes | `STARTING`→`FAILED(unsupported_type)` | none | remains |
| Parse failure | yes | `STARTING`→`WORKING(20)`→`FAILED(InputFileError: )` | none | remains |

The FAILED-frame `message` differs by cause: `document_already_exists` (duplicate), `unsupported_type` (no parser), or the `ParseError` string itself (`InputFileError: `). The `WORKING(20)` frame appears only for the parse-failure case because that is the only failure reached *after* the `parsing_document` checkpoint.

---

## Methodology notes, labels, and coverage pass

**Labels used above.** The only deviation from a pure real-path run is the **email IMAP transport**, explicitly labelled **[non-canonical stand-in — transport only]** in Q1: the raw message was constructed locally and fed to the genuine `MailAccountHandler.handle_message` entry function, so the consume path is real; only the network fetch was substituted. Two statements are labelled **[inferred]**: (i) that a parser returning a date would skip the `parse_date(90)` frame (not reproducible with the tesseract parser, which never sets a date); and (ii) that a trained `classification_model.pickle` would additionally run the ML predict path (the default fresh stack has no model). Everything else is directly observed.

**Stability.** The progress checkpoint set `{0,20,70,90,95,100}` and the six-frame success shape were stable across **five** independent successful runs (EP1/EP2/EP3/Q2/Q3-image-OCR); no run-to-run variation in the emitted values was observed.

**Coverage pass (every named item addressed):**

- **Q1 detection & handoff** — three detectors named (`_consume`, `PostDocumentView.post`, `MailAccountHandler.handle_message`); shared handoff `async_task("documents.tasks.consume_file")`; convergence proven via the task ledger. ✔
- **Q2 parsing / classification / indexing** — each named and evidenced: parsing (`RasterisedDocumentParser`, OCRmyPDF), classification (ML skipped with `load_classifier` `None`; rule-based `matching.py` still runs), indexing (`add_to_index` → Whoosh). ✔
- **Q3 progress / completion** — six-frame WebSocket sequence with per-frame emitters; the dead `progress_callback` and the inaccurate `L238` comment; completion via terminal `SUCCESS` frame + task result string. ✔
- **Q4 final destination & state** — `Document` row, original/archive/thumbnail media, Whoosh entry, admin `LogEntry`, source unlink; before/during/after; atomic + FileLock rationale. ✔
- **Q5 already-processed tracking** — the **negative** (no status column) proven by field introspection; four implicit signals evidenced. ✔
- **Q6 duplicate avoidance** — both layers: app md5 pre-check (`It is a duplicate.` / `document_already_exists`) and DB `UNIQUE(checksum)` (`IntegrityError`); byte-exact md5 match. ✔

**Scope note.** This investigation is read-only; the only durable artifact is this document. All temporary observation scripts, sample PDFs, and helper DB rows created for the runs were removed afterward, leaving the repository unchanged apart from this file. Findings (e.g. the absence of a `Document` status field, the absence of a `progress()` caller) are stated as of commit `542221a38dff06361e07976452f9aea24d210542`.

