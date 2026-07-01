# How data moves through Paperless‑NGX during document ingestion

*A runtime‑grounded investigation at commit `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`).*

This document answers five questions about the ingestion pipeline **from what was actually observed while the system was running**, not from reading the source alone. Every answer pairs **verbatim captured output** (with the exact command that produced it) against **`file:line` citations** into the source that produced that output. Where something is attached in code but not visible at runtime, that is called out explicitly rather than asserted.

---

## 0. Runtime that was stood up, and how evidence was captured

**Pinned runtime (observed, matches `Pipfile.lock`).** All observation used the project's Python 3.10 virtualenv, not the newer host tooling.

```console
$ ./venv/bin/python --version
Python 3.10.20
$ ./venv/bin/python -c "import django,django_q,channels; print('django',django.get_version()); print('django_q',django_q.VERSION); print('channels',channels.__version__)"
django 4.0.4
django_q (1, 3, 9)
channels 3.0.4
```

Key locked versions confirmed present in the venv: `Django==4.0.4`, `django-q==1.3.9`, `channels==3.0.4`, `channels-redis==3.4.0`, `redis==3.5.3`, `Whoosh==2.7.4`, `scikit-learn==1.0.2`, `watchdog==2.1.7`, `inotifyrecursive==0.3.5`, `ocrmypdf==13.4.3`, `python-magic==0.4.25`, `gunicorn==20.1.0`.

**Topology.** The documented multi‑process topology (defined in `docker/supervisord.conf`) was brought up natively: a **Redis** broker, the **django‑q** worker (`qcluster`) [`docker/supervisord.conf:L28-L29`], the filesystem **watcher** (`document_consumer`) [`docker/supervisord.conf:L19-L20`], and the **ASGI/WebSocket** server (`gunicorn … paperless.asgi:application`) [`docker/supervisord.conf:L10-L11`], over the default **SQLite** DB at `DATA_DIR/db.sqlite3` [`src/paperless/settings.py:L300`].

**Isolation (read‑only guarantee).** To leave the repository byte‑for‑byte unchanged, the runtime was pointed at throwaway directories outside the repo via environment variables (`PAPERLESS_DATA_DIR`, `PAPERLESS_MEDIA_ROOT`, `PAPERLESS_CONSUMPTION_DIR` [`src/paperless/settings.py:L66,L61,L78`]), plus `PAPERLESS_REDIS=redis://localhost:6379` and `PAPERLESS_OCR_OUTPUT_TYPE=pdf`. The consumption directory, media tree, SQLite DB, Whoosh index, and logs all lived under `/tmp/blitzy_obs/…`. All temporary scripts and test inputs were removed afterward.

**Service startup, observed.**

```console
$ setsid ./venv/bin/python manage.py qcluster > qcluster.log 2>&1 &   # from src/
$ cat qcluster.log
04:53:40 [Q] INFO Q Cluster fanta-red-ceiling-red starting.
04:53:40 [Q] INFO Process-1:1 ready for work at 32227
04:53:40 [Q] INFO Process-1:2 ready for work at 32228
04:53:40 [Q] INFO Process-1:12 monitoring at 32238
04:53:40 [Q] INFO Process-1 guarding cluster fanta-red-ceiling-red
04:53:40 [Q] INFO Process-1:13 pushing tasks at 32239
04:53:40 [Q] INFO Q Cluster fanta-red-ceiling-red running.
```

```console
$ setsid ./venv/bin/python manage.py document_consumer > consumer.log 2>&1 &   # from src/
$ cat consumer.log
[2026-07-01 04:54:04,827] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /tmp/blitzy_obs/consume
```

The watcher's `"Using inotify to watch directory for changes: …"` line confirms the **inotify** watch mode was selected — the default, because `CONSUMER_POLLING` defaults to `0` [`src/paperless/settings.py:L478`] and the code takes the inotify branch when `CONSUMER_POLLING == 0` [`src/documents/management/commands/document_consumer.py:L178-L179`, message at `:L200`].

```console
$ setsid ./venv/bin/gunicorn -c ../gunicorn.conf.py paperless.asgi:application > gunicorn.log 2>&1 &   # from src/
$ head -4 gunicorn.log
[2026-07-01 04:54:29 +0000] [32793] [INFO] Starting gunicorn 20.1.0
[2026-07-01 04:54:29 +0000] [32793] [INFO] Listening at: http://0.0.0.0:8000 (32793)
[2026-07-01 04:54:29 +0000] [32793] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 04:54:29 +0000] [32793] [INFO] Server is ready. Spawning workers
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/api/
HTTP 200
```

**Log format & file location.** The application log is written to `DATA_DIR/log/paperless.log` [`src/paperless/settings.py:L395`], and the `paperless` logger is configured at `DEBUG` [`src/paperless/settings.py:L409`]. The line format is `"[{asctime}] [{levelname}] [{name}] {message}"` [`src/paperless/settings.py:L378`], which matches every quoted line below.

**The six evidence channels used throughout.** (a) `paperless.log`; (b) the `qcluster` worker stdout; (c) the WebSocket `status_updates` stream — captured two ways (see Q3): a Redis channel‑layer **group tap**, and a fully **authenticated end‑to‑end WebSocket client**; (d) the SQLite `Document` row via the Django ORM; (e) on‑disk artifacts under `media/documents/{originals,archive,thumbnails}` and the Whoosh index dir; (f) the **django‑q `Task` record** (task name + success/result), which corroborates the task name independently of the logs.

**Test input.** A throwaway PNG carrying text (so OCR produces content) was used:

```console
$ md5sum /tmp/blitzy_obs/testinput/blitzy_ingest_probe.png
df40b0750d5e2365e5adf7928e0c5eb5  /tmp/blitzy_obs/testinput/blitzy_ingest_probe.png
```

---

## Q1 — Detection & Handoff

**Question.** When Paperless‑NGX is running and a new document appears, what observable runtime behavior shows how the document is *detected* and *handed off* for processing?

**Answer.** The `document_consumer` **filesystem watcher** detects the new file (via inotify), logs it, and **enqueues an asynchronous django‑q task named `documents.tasks.consume_file`**; the `qcluster` worker then picks it up. Two other entry points enqueue the *same* task: the REST upload endpoint and the email fetcher.

### Observed: consumption‑directory path

Dropping the file into the watched directory produced this, verbatim, in `paperless.log` (and mirrored on the watcher's stdout):

```console
$ cp blitzy_ingest_probe.png /tmp/blitzy_obs/consume/          # trigger
$ grep -n "task queue\|Enqueued\|processing \[" paperless.log qcluster.log
```
```text
[2026-07-01 04:59:38,060] [INFO] [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/blitzy_ingest_probe.png to the task queue.
04:59:38 [Q] INFO Enqueued 1
04:59:38 [Q] INFO Process-1:5 processing [blitzy_ingest_probe.png]
```

- The watcher's logger is `"paperless.management.consumer"` [`src/documents/management/commands/document_consumer.py:L24`]. The exact detection line `f"Adding {filepath} to the task queue."` is emitted at [`src/documents/management/commands/document_consumer.py:L85`], immediately before it calls `async_task("documents.tasks.consume_file", filepath, …)` [`:L86-L91`]. The task literal is the string `"documents.tasks.consume_file"` [`:L87`].
- inotify is armed with the flags `flags.CLOSE_WRITE | flags.MOVED_TO` [`:L203`]; the filesystem events are handled by `Handler.on_created` [`:L129`] and `Handler.on_moved` [`:L132`].
- `Enqueued 1` is django‑q acknowledging the queued task; `Process-1:5 processing [blitzy_ingest_probe.png]` is the `qcluster` worker dequeuing it (the bracketed name is the task's `task_name`, set to the file's basename at [`:L90`]).

### Corroboration: the django‑q `Task` record names the task independently of the logs

```console
$ ./venv/bin/python -c "import django,os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); \
from django_q.models import Task; \
[print(f'name={t.name!r} func={t.func} success={t.success} result={t.result!r}') for t in Task.objects.all()]"
```
```text
name='blitzy_ingest_probe.png' func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
```

The persisted record shows `func=documents.tasks.consume_file` — the same handoff target, observed from the task queue's own database record rather than a log line.

### Observed: REST upload path

The REST endpoint returns immediately after enqueuing:

```console
$ curl -sS -w "\nHTTP_STATUS:%{http_code}\n" -H "Authorization: Token <REDACTED_TOKEN>" \
       -F "document=@blitzy_rest_probe.png" http://localhost:8000/api/documents/post_document/
```
```text
"OK"
HTTP_STATUS:200
```

- `PostDocumentView.post` [`src/documents/views.py:L497`] writes the upload to a `tempfile.NamedTemporaryFile(prefix="paperless-upload-", …)` [`:L512`], generates `task_id = str(uuid.uuid4())` [`:L521`], enqueues `async_task("documents.tasks.consume_file", temp_filename, …)` [`:L523-L533`], and returns `Response("OK")` — hence the body `"OK"` and HTTP `200` observed above [`:L535`]. The route is registered as `post_document` under `api/documents/post_document/` [`src/paperless/urls.py:L57-L59`].
- That upload was consumed too, confirming the REST path reaches the identical task:

```text
05:02:22 [Q] INFO Process-1:6 processing [blitzy_rest_probe.png]
[2026-07-01 05:02:22,907] [INFO] [paperless.consumer] Consuming blitzy_rest_probe.png
# django-q Task record:
name='blitzy_rest_probe.png' func=documents.tasks.consume_file success=True result='Success. New document id 2 created'
```

- The **email** entry point also enqueues the same task: `async_task("documents.tasks.consume_file", path=temp_filename, …)` [`src/paperless_mail/mail.py:L336-L337`]. (Not exercised at runtime here — no mail server was configured — so it is cited from source, not asserted as observed.)

**Reasoning.** Detection and handoff are deliberately decoupled: the watcher (or the REST view, or the mail fetcher) only *enqueues* work and returns; the actual processing happens later in a separate `qcluster` worker process. The single, observable handoff contract across all three entry points is the django‑q task name `documents.tasks.consume_file`, which is visible both in the queue logs and in the persisted `Task` record.

---

## Q2 — Stage Transitions into Parsing / Classification / Indexing

**Question.** What log messages, task names, or state changes indicate the transition from initial detection into *parsing*, *classification*, and *indexing*?

**Answer.** The `qcluster` worker runs `documents.tasks.consume_file()` [`src/documents/tasks.py:L184`] (logger `"paperless.tasks"` [`:L29`]), which delegates to `Consumer().try_consume_file(...)` [`src/documents/tasks.py:L236`]. `Consumer` (logger `"paperless.consumer"` [`src/documents/consumer.py:L52-L54`]) emits a fixed sequence of named log lines as it moves through the stages, and — after the row is stored — fires the `document_consumption_finished` signal, which is what drives **classification** and **Whoosh indexing**.

### Observed: the full correlated `paperless.log` sequence for one ingestion

```console
$ sed -n '3,$p' /tmp/blitzy_obs/data/log/paperless.log   # lines added by the run
```
```text
[2026-07-01 04:59:38,060] [INFO] [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/blitzy_ingest_probe.png to the task queue.
[2026-07-01 04:59:38,179] [INFO] [paperless.consumer] Consuming blitzy_ingest_probe.png
[2026-07-01 04:59:38,180] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-01 04:59:38,180] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 04:59:38,183] [DEBUG] [paperless.consumer] Parsing blitzy_ingest_probe.png...
[2026-07-01 04:59:38,241] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/blitzy_obs/consume/blitzy_ingest_probe.png: 'dpi'
[2026-07-01 04:59:38,242] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-01 04:59:38,242] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '…', 'output_file': '…/archive.pdf', … 'output_type': 'pdf', … 'sidecar': '…/sidecar.txt', 'image_dpi': 120}
[2026-07-01 04:59:39,049] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-01 04:59:39,049] [DEBUG] [paperless.consumer] Generating thumbnail for blitzy_ingest_probe.png...
[2026-07-01 04:59:39,052] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> … archive.pdf[0] … convert.png
[2026-07-01 04:59:39,274] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 … convert.png -out … thumb_optipng.png
[2026-07-01 04:59:39,622] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 04:59:39,625] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 04:59:39,641] [DEBUG] [paperless.consumer] Deleting file /tmp/blitzy_obs/consume/blitzy_ingest_probe.png
[2026-07-01 04:59:39,648] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-f3gu70k6
[2026-07-01 04:59:39,649] [INFO] [paperless.consumer] Document 2026-07-01 blitzy_ingest_probe consumption finished
```

Mapping each observed line to its source:

| Observed line | Stage | Source |
|---|---|---|
| `Consuming blitzy_ingest_probe.png` | pipeline start | `self.log("info", f"Consuming {self.filename}")` [`src/documents/consumer.py:L215`] |
| `Detected mime type: image/png` | MIME detection | `self.log("debug", f"Detected mime type: {mime_type}")` [`:L221`] (value from `magic.from_file` [`:L219`]) |
| `Parser: RasterisedDocumentParser` | parser selection | [`:L246`] |
| `Parsing blitzy_ingest_probe.png...` | **parsing** | [`:L260`], immediately after the `parsing_document` progress milestone [`:L259`] |
| `paperless.parsing.tesseract … OCRmyPDF … output_type: 'pdf'` | OCR parse | tesseract parser, logger `"paperless.parsing.tesseract"` [`src/paperless_tesseract/parsers.py:L24`] |
| `Generating thumbnail …` | thumbnailing | [`:L263`], after the `generating_thumbnail` milestone [`:L264`] |
| `paperless.classifier … model does not exist (yet) …` | **classification** | `src/documents/classifier.py:L33` |
| `Saving record to database` | persistence | `src/documents/consumer.py:L387` (inside `_store`) |
| `Document … consumption finished` | completion | `self.log("info", "Document {} consumption finished".format(document))` [`:L373`] |

### What drives classification and indexing: the `document_consumption_finished` signal

After `_store()` persists the row, the consumer fires the signal inside the same atomic transaction:

```text
document_consumption_finished.send(sender=…, document=document, logging_group=…, classifier=classifier)   # src/documents/consumer.py:L306
```

`DocumentsConfig.ready()` connects **six** receivers to that signal [`src/documents/apps.py:L22-L27`]:

```python
document_consumption_finished.connect(add_inbox_tags)      # src/documents/apps.py:L22
document_consumption_finished.connect(set_correspondent)   # :L23
document_consumption_finished.connect(set_document_type)   # :L24
document_consumption_finished.connect(set_tags)            # :L25
document_consumption_finished.connect(set_log_entry)       # :L26
document_consumption_finished.connect(add_to_index)        # :L27
```

- **Classification** = `set_correspondent`/`set_document_type`/`set_tags` (handlers `src/documents/signals/handlers.py:L35,L101,L168`; logger `"paperless.handlers"` [`:L27`]). They combine the ML classifier `src/documents/classifier.py` (persisted model `MODEL_FILE = DATA_DIR/classification_model.pickle` [`src/paperless/settings.py:L74`]) with rule matching in `src/documents/matching.py`. The observed `paperless.classifier` line proves this stage executed; because no classifier model existed yet, the automatic‑matching branch logged that it was skipped [`src/documents/classifier.py:L33`] — the transition still ran, it simply had no model to apply.
- **Indexing** = `add_to_index` [`src/documents/signals/handlers.py:L428`], which calls `index.add_or_update_document(document)` [`:L431`]; that opens a Whoosh `AsyncWriter(open_index())` [`src/documents/index.py:L118-L120`, writer at `:L66`] (logger `"paperless.index"` [`:L28`]). Runtime proof that indexing actually happened is under Q4 (a search of the index returns the new document).

### Corroboration: task name + success in the django‑q record

The `qcluster` stdout shows the framework side of the same run, and the persisted `Task` record confirms the task name and outcome:

```text
04:59:39 [Q] INFO Processed [blitzy_ingest_probe.png]
04:59:39 [Q] INFO recycled worker Process-1:5
# django-q Task: func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
```

**Reasoning.** The stage transitions are observable as a deterministic, ordered log sequence on the `paperless.consumer` logger (Consuming → Detected mime type → Parser → Parsing → Generating thumbnail → Saving record → consumption finished), with parsing delegated to a MIME‑specific parser logger (`paperless.parsing.tesseract` here). Classification and indexing are not inline steps but **signal‑driven consequences** of `document_consumption_finished` [`:L306`], which is why they appear *after* “Saving record to database” and are wired centrally in `apps.py`.

> **Edge case (barcode split).** Before normal consumption, `consume_file()` optionally scans for separator barcodes; if found it splits the file and returns the string `"File successfully split"` *instead of* consuming [`src/documents/tasks.py:L233`]. This path is disabled by default (`CONSUMER_ENABLE_BARCODES` defaults to `False` [`src/paperless/settings.py:L502-L503`], via `__get_boolean(default="NO")` [`:L34`]), so the normal run above went straight to `try_consume_file` [`:L236`].


---

## Q3 — Progress & Completion reporting

**Question.** How does the system reflect *progress* or *completion* of each stage while the document is being processed?

**Answer.** Progress is broadcast over a **WebSocket channel‑layer group named `"status_updates"`** as structured JSON payloads at **fixed milestones**, built by `Consumer._send_progress()` [`src/documents/consumer.py:L56`]. Each payload carries `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, and `document_id` [`:L64-L72`] and is sent with `async_to_sync(self.channel_layer.group_send)("status_updates", {"type": "status_update", "data": payload})` [`:L73-L76`]. Browsers receive these frames through `StatusConsumer`.

### How the payloads were captured (two methods, flagged)

Because the `ws/status/` endpoint **requires authentication** (`StatusConsumer.connect()` raises `DenyConnection()` for unauthenticated users [`src/paperless/consumers.py:L13-L15`]), two complementary temporary subscribers were used:

1. **Channel‑layer group tap** — a script that joined the Redis‑backed `"status_updates"` group directly and printed each `group_send` envelope. This captures exactly what `_send_progress()` broadcasts.
2. **Authenticated end‑to‑end WebSocket client** — logged into the Django admin to obtain a `sessionid` cookie, connected to `ws://localhost:8000/ws/status/`, and printed the JSON frames the browser actually receives.

The authentication requirement itself was verified: an **unauthenticated** connect is rejected before any frame is sent.

```console
$ ./venv/bin/python - <<'PY'   # unauthenticated connect
import asyncio, websockets
async def m():
    try:
        async with websockets.connect("ws://localhost:8000/ws/status/"): print("connected")
    except Exception as e: print(f"DENIED: {type(e).__name__}: {e}")
asyncio.run(m())
PY
DENIED: InvalidStatusCode: server rejected WebSocket connection: HTTP 403
```

### Observed: the six milestone payloads (authenticated end‑to‑end WebSocket frames)

These are the verbatim frames received by the authenticated client during the successful ingestion of `blitzy_ingest_probe.png`:

```text
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",            "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 20,  "max_progress": 100, "status": "WORKING",  "message": "parsing_document",    "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 70,  "max_progress": 100, "status": "WORKING",  "message": "generating_thumbnail", "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 90,  "max_progress": 100, "status": "WORKING",  "message": "parse_date",          "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 95,  "max_progress": 100, "status": "WORKING",  "message": "save_document",        "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 100, "max_progress": 100, "status": "SUCCESS",  "message": "finished",            "document_id": 1}
```

The **channel‑layer tap** captured the identical data wrapped in the transport envelope `{"type": "status_update", "data": {…}}` — e.g. the final frame:

```text
[TAP 04:59:39] {"type": "status_update", "data": {"filename": "blitzy_ingest_probe.png", "task_id": "33fd6ff6-a1ff-4799-aea2-b73dc96ac8d8", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

Each milestone maps to a specific `_send_progress()` call site (exact literals verified against source):

| Observed `status` / `current_progress` / `message` | Source |
|---|---|
| `STARTING` `0` `new_file` | `self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)` [`src/documents/consumer.py:L202`] (`MESSAGE_NEW_FILE = "new_file"` [`:L43`]) |
| `WORKING` `20` `parsing_document` | [`:L259`] (`MESSAGE_PARSING_DOCUMENT = "parsing_document"` [`:L45`]) |
| `WORKING` `70` `generating_thumbnail` | [`:L264`] (`MESSAGE_GENERATING_THUMBNAIL = "generating_thumbnail"` [`:L46`]) |
| `WORKING` `90` `parse_date` | [`:L274`] — sent only when the parser returned no date (`if not date:` [`:L273`]) |
| `WORKING` `95` `save_document` | [`:L294`] (`MESSAGE_SAVE_DOCUMENT = "save_document"` [`:L48`]) |
| `SUCCESS` `100` `finished`, `document_id: 1` | `self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)` [`:L375`] |

Note the payload keys match the dict built at [`:L64-L72`] exactly, and `document_id` is `null` for every intermediate frame and becomes the real id (`1`) only in the terminal `SUCCESS` frame — because only that call passes `document.id` [`:L375`]. (During parsing the parser can also emit intermediate `WORKING` frames rescaled into the 20–70 band via `int((current_progress / max_progress) * 50 + 20)` [`:L237-L240`]; the six above are the fixed stage milestones.)

### Relay to browsers

`StatusConsumer` (a `WebsocketConsumer` [`src/paperless/consumers.py:L9`]) joins the `"status_updates"` group on connect [`:L17-L20`] and, on each `status_update` event, sends `self.send(json.dumps(event["data"]))` [`:L33`] — which is exactly why the end‑to‑end frames above are the JSON‑serialized `data` payload. Routing is `ProtocolTypeRouter({… "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))})` [`src/paperless/asgi.py:L17-L20`], and the endpoint is registered at `re_path(r"ws/status/$", StatusConsumer.as_asgi())` [`src/paperless/urls.py:L137`].

**Reasoning.** Progress reporting is a **transient, push‑based** mechanism: the worker computes coarse, fixed percentages at each stage boundary and broadcasts them to any subscribed browser via Redis + Channels. Completion of the whole pipeline is signalled by the terminal `SUCCESS`/`100`/`finished` frame that additionally carries the new `document_id`, letting the UI link straight to the stored document. As shown next (Q4), these percentages live only on the wire — they are never written to the database.


---

## Q4 — Final State & Storage

**Question.** After processing finishes, what observable evidence shows *where* the document's data ends up and how its *final state* is recorded?

**Answer.** The final state is a persisted **`Document` row** plus three **on‑disk artifacts** (original, archive, thumbnail) and one **full‑text index entry**.

> **KEY FINDING — final state has no status column.** The `Document` model has **no `status`/`state`/`processing` field of any kind**. The transient processing status (`STARTING`/`WORKING`/`SUCCESS`/`FAILED`) exists **only in the WebSocket stream** (Q3); it is never persisted. A document's "final state" is therefore represented purely by *the existence of the row and its files*, not by a status flag.

### Observed: the persisted `Document` row

```console
$ ./venv/bin/python - <<'PY'
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from documents.models import Document
d = Document.objects.get(id=1)
for k in ("id","title","mime_type","checksum","archive_checksum","filename","archive_filename","storage_type"):
    print(f"{k}: {getattr(d,k)}")
print("created:", d.created.isoformat()); print("added:", d.added.isoformat())
print("content[:100]:", repr((d.content or '')[:100]))
PY
```
```text
id: 1
title: blitzy_ingest_probe
mime_type: image/png
checksum: df40b0750d5e2365e5adf7928e0c5eb5
archive_checksum: b114e5905b3f34fc9c934def4f4cb4c7
filename: 0000001.png
archive_filename: 0000001.pdf
storage_type: unencrypted
created: 2026-07-01T04:59:37.055019+00:00
added: 2026-07-01T04:59:39.625942+00:00
content[:100]: 'Blitzy Runtime Observation Sample\nPaperless-NGX ingestion probe document\nHello World Alpha Bravo Charl'
```

Field‑by‑field, mapped to the model `class Document(models.Model)` [`src/documents/models.py:L88`]:
- `content` — the OCR‑extracted text (`TextField`, help text "The raw, text-only data of the document. This field is primarily used for searching.") [`:L117-L124`]. The observed value is exactly the text rendered into the test PNG, proving OCR populated it.
- `mime_type = image/png` [`:L126`]; `storage_type = unencrypted` [`:L161`].
- `checksum = df40b0750d5e2365e5adf7928e0c5eb5` — `CharField(max_length=32, editable=False, unique=True)` [`:L135-L141`]. This is the MD5 of the **original** file (see the match under Q5), and it is the deduplication key.
- `archive_checksum = b114e5905b3f34fc9c934def4f4cb4c7` [`:L143`] — MD5 of the generated archive PDF.
- `filename = 0000001.png` [`:L176-L184`] and `archive_filename = 0000001.pdf` [`:L186-L194`] — both `FilePathField(unique=True)`.
- `created` [`:L152`] and `added` [`:L169-L174`] timestamps.

### Observed: the on‑disk artifacts

```console
$ find /tmp/blitzy_obs/media/documents -type f -printf '%s bytes  %p\n' | sort
```
```text
18438 bytes  /tmp/blitzy_obs/media/documents/archive/0000001.pdf
12136 bytes  /tmp/blitzy_obs/media/documents/originals/0000001.png
7130 bytes  /tmp/blitzy_obs/media/documents/thumbnails/0000001.png
```

These three directories are `ORIGINALS_DIR = media/documents/originals` [`src/paperless/settings.py:L62`], `ARCHIVE_DIR = media/documents/archive` [`:L63`], and `THUMBNAIL_DIR = media/documents/thumbnails` [`:L64`]. The files are written inside the atomic block of `try_consume_file()` after the signal, via `self._write(...)` and `document.save()` [`src/documents/consumer.py:L315-L346`]. The stored original is byte‑identical to the input (its MD5 equals the row's `checksum`):

```console
$ md5sum /tmp/blitzy_obs/media/documents/originals/0000001.png /tmp/blitzy_obs/media/documents/archive/0000001.pdf
df40b0750d5e2365e5adf7928e0c5eb5  …/originals/0000001.png     # == Document.checksum
b114e5905b3f34fc9c934def4f4cb4c7  …/archive/0000001.pdf       # == Document.archive_checksum
```

### Observed: the full‑text index entry

The Whoosh index at `INDEX_DIR = DATA_DIR/index` [`src/paperless/settings.py:L73`] was updated on consumption, and the new document is searchable by its OCR content:

```console
$ ls /tmp/blitzy_obs/data/index/
MAIN_WRITELOCK  MAIN_fgnwuzkd7uxez0pz.seg  _MAIN_2.toc
$ ./venv/bin/python - <<'PY'
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from documents import index
from whoosh.qparser import QueryParser
ix = index.open_index()
with ix.searcher() as s:
    for term in ("bravo","blitzy","paperless"):
        h = s.search(QueryParser("content", ix.schema).parse(term))
        print(f"search content:'{term}' -> {len(h)} hit(s): {[(x['id'], x['title']) for x in h]}")
    print("index doc_count:", ix.doc_count())
PY
```
```text
search content:'bravo' -> 1 hit(s): [(1, 'blitzy_ingest_probe')]
search content:'blitzy' -> 1 hit(s): [(1, 'blitzy_ingest_probe')]
search content:'paperless' -> 1 hit(s): [(1, 'blitzy_ingest_probe')]
index doc_count: 1
```

This is the runtime proof that the `add_to_index` handler ran (Q2): the OCR'd content is retrievable from the Whoosh index, and the document count is `1`.

### Observed: the field list confirms there is no status column

```console
$ ./venv/bin/python - <<'PY'
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from documents.models import Document
names = [f.name for f in Document._meta.get_fields()]
print(names)
print("has 'status':", "status" in names, "| 'state':", "state" in names, "| 'processing':", "processing" in names)
PY
```
```text
['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum', 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename', 'archive_filename', 'archive_serial_number', 'tags']
has 'status': False | 'state': False | 'processing': False
```

**Reasoning.** "Where the data ends up" is answered concretely: the row in SQLite (`data/db.sqlite3`), the original + archive + thumbnail files under `media/documents/…`, and the Whoosh index under `data/index`. "How the final state is recorded" is answered by the *absence* of any status field: success is implied by the row existing with a populated `checksum`, `content`, `filename`/`archive_filename`, and `added` timestamp. The live status values seen in Q3 are ephemeral UI signalling only — confirmed by the field‑list dump above.


---

## Q5 — Duplicate Avoidance

**Question.** How does Paperless‑NGX track whether a document has *already been processed* and avoid *duplicate processing*?

**Answer.** Before consuming, the consumer computes the file's **MD5 checksum** and checks whether any existing document already has that value as its `checksum` **or** `archive_checksum`; if so it **fails fast** with the message `document_already_exists` and never creates a second document. The database additionally enforces a `unique=True` constraint on `checksum` as a storage‑layer backstop.

### The application‑level check

`pre_check_duplicate()` [`src/documents/consumer.py:L102`] does:

```python
checksum = hashlib.md5(f.read()).hexdigest()                              # :L104
if Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists():   # :L105-L107
    …
    self._fail(MESSAGE_DOCUMENT_ALREADY_EXISTS,                            # :L110-L112
               f"Not consuming {self.filename}: It is a duplicate.")
```

with `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` [`:L37`]. It is invoked early in `try_consume_file()` at [`:L213`], and `_fail()` broadcasts `FAILED`/`100` via `_send_progress(100, 100, "FAILED", message)` [`:L79`] and then `raise ConsumerError(...)` [`:L81`].

### Observed: re‑ingesting the identical file (same bytes)

```console
$ md5sum /tmp/blitzy_obs/testinput/blitzy_ingest_probe.png     # identical bytes to the first ingest
df40b0750d5e2365e5adf7928e0c5eb5  …/blitzy_ingest_probe.png
$ cp …/blitzy_ingest_probe.png /tmp/blitzy_obs/consume/        # re-drop
$ grep -n "task queue\|duplicate" /tmp/blitzy_obs/data/log/paperless.log | tail -2
```
```text
[2026-07-01 05:04:54,430] [INFO]  [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/blitzy_ingest_probe.png to the task queue.
[2026-07-01 05:04:54,557] [ERROR] [paperless.consumer] Not consuming blitzy_ingest_probe.png: It is a duplicate.
```

The `[ERROR] … Not consuming blitzy_ingest_probe.png: It is a duplicate.` line is the verbatim message from [`src/documents/consumer.py:L112`]. The file was re‑detected and enqueued (the watcher does not itself dedup), but consumption was rejected.

### Observed: the WebSocket `FAILED` payload

Both subscribers captured the same two frames for the duplicate task (`task_id bc52ff68-b962-4f76-a4a9-f3d81187a257`): the initial `STARTING` frame (sent at [`:L202`] *before* the dedup check), then the terminal `FAILED` frame:

```text
{"filename": "blitzy_ingest_probe.png", "task_id": "bc52ff68-b962-4f76-a4a9-f3d81187a257", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",                "document_id": null}
{"filename": "blitzy_ingest_probe.png", "task_id": "bc52ff68-b962-4f76-a4a9-f3d81187a257", "current_progress": 100, "max_progress": 100, "status": "FAILED",   "message": "document_already_exists", "document_id": null}
```

The exact literals `"status": "FAILED"` and `"message": "document_already_exists"` match `_fail()` → `_send_progress(100, 100, "FAILED", message)` [`:L79`] with `message = MESSAGE_DOCUMENT_ALREADY_EXISTS` [`:L37,L111`].

### Corroboration: the django‑q Failure record (clean traceback)

The task's persisted record flips to `success=False`, and its stored traceback pins the exact call chain (this is the authoritative, un‑garbled traceback):

```console
$ ./venv/bin/python -c "import django,os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); \
from django_q.models import Task; t=[x for x in Task.objects.all() if not x.success][-1]; \
print('name=',t.name,'func=',t.func,'success=',t.success); print(t.result)"
```
```text
name= blitzy_ingest_probe.png func= documents.tasks.consume_file success= False
blitzy_ingest_probe.png: Not consuming blitzy_ingest_probe.png: It is a duplicate. : Traceback (most recent call last):
  File ".../django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File ".../src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File ".../src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File ".../src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File ".../src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: blitzy_ingest_probe.png: Not consuming blitzy_ingest_probe.png: It is a duplicate.
```

This confirms, at runtime, the precise lines: `tasks.py:236` → `consumer.py:213` (`pre_check_duplicate()`) → `consumer.py:110` (`self._fail(`) → `consumer.py:81` (`raise ConsumerError`).

### Corroboration: the database `unique=True` backstop

Even if the application check were bypassed, the schema forbids a duplicate checksum. Attempting to insert a second row with an existing `checksum` raises an `IntegrityError`:

```console
$ ./venv/bin/python - <<'PY'
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from django.db import transaction, IntegrityError
from documents.models import Document
existing = Document.objects.first()
try:
    with transaction.atomic():
        Document.objects.create(title="dup-attempt", content="x", mime_type="image/png",
                                checksum=existing.checksum, storage_type="unencrypted")
    print("UNEXPECTED: insert succeeded")
except IntegrityError as e:
    print("IntegrityError:", str(e).strip())
PY
```
```text
IntegrityError: UNIQUE constraint failed: documents_document.checksum
```

This is the runtime manifestation of `checksum = models.CharField(… unique=True …)` [`src/documents/models.py:L135-L141`]. (A periodic, checksum‑based integrity pass also exists in `src/documents/sanity_checker.py` as a supplementary safeguard, cited from source.)

**Reasoning.** "Already processed" is tracked by content identity — the MD5 checksum of the file's bytes — not by filename or path. Deduplication is enforced in **two independent layers**: an early application‑level query in `pre_check_duplicate()` that fails the task cleanly with `document_already_exists` (so a re‑dropped file produces a `FAILED` WebSocket frame and a django‑q failure, but **no** second `Document`), and a database `UNIQUE` constraint that guarantees the invariant even if the application check is ever skipped. Matching against *both* `checksum` and `archive_checksum` [`:L106`] means a file identical to either a stored original or a stored archive is caught.

---

## Coverage

Re‑reading the five sub‑questions, each is explicitly answered above with verbatim runtime output and `file:line` citations:

| Sub‑question | Answered in | Signature runtime evidence |
|---|---|---|
| **Q1** Detection & handoff | Q1 | Watcher `"Adding … to the task queue."`; `qcluster` `processing […]`; REST `"OK"`/HTTP `200`; django‑q `Task.func = documents.tasks.consume_file` |
| **Q2** Transitions → parsing / classification / indexing | Q2 | `paperless.consumer` log sequence (Consuming → Detected mime type → Parser → Parsing → Generating thumbnail → Saving record → consumption finished); `document_consumption_finished` → 6 handlers; `paperless.classifier` line; index search hit |
| **Q3** Progress & completion | Q3 | Six WebSocket frames `STARTING 0 new_file` → `WORKING 20/70/90/95` → `SUCCESS 100 finished document_id 1`; unauthenticated connect → HTTP `403` |
| **Q4** Final state & storage | Q4 | Persisted `Document` row (id 1); `originals/archive/thumbnails` files; Whoosh search hit; **KEY FINDING: no status column** (field‑list dump) |
| **Q5** Duplicate avoidance | Q5 | `"Not consuming …: It is a duplicate."`; WebSocket `FAILED`/`document_already_exists`; django‑q `success=False` traceback; DB `UNIQUE constraint failed: documents_document.checksum` |

**Method note.** Per the investigation rule, every answer above was written from output captured while the system was running (the pinned Python 3.10.20 venv over Redis + django‑q + Channels + gunicorn/ASGI + SQLite); source `file:line` references are used only to locate the origin of each observed value. Two claims are cited from source rather than executed and are flagged as such: the **email** ingestion entry point [`src/paperless_mail/mail.py:L336-L337`] (no mail server was configured) and the periodic `sanity_checker` integrity pass. All temporary observation scripts, test inputs, and the throwaway data directory were removed after capture, leaving the repository unchanged except for this document.
