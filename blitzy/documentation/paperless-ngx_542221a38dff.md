# How data moves through Paperless‑NGX during document ingestion

*A runtime‑grounded investigation at commit `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`).*

This document answers five questions about the ingestion pipeline **from what was actually observed while the system was running**, not from reading the source alone. Every answer pairs **verbatim captured output** (with the exact command that produced it) against **`file:line` citations** into the source that produced that output. Where something is attached in code but not visible at runtime, that is called out explicitly rather than asserted.

---

## 0. Runtime that was stood up, and how evidence was captured

**Pinned runtime (observed, matches `Pipfile.lock`).** All observation used the project's Python 3.10 virtualenv, not the newer host tooling. The exact installed version of every runtime component the answers rely on was read from the venv with `importlib.metadata`, and each value is cross‑checked against its pinned `Pipfile.lock` entry:

```console
$ ./venv/bin/python --version
Python 3.10.20
$ ./venv/bin/python -c "import importlib.metadata as m
pkgs=['Django','django-q','channels','channels-redis','redis','Whoosh','scikit-learn','watchdog','inotifyrecursive','ocrmypdf','python-magic','gunicorn']
[print(f'{p}=={m.version(p)}') for p in pkgs]"
Django==4.0.4
django-q==1.3.9
channels==3.0.4
channels-redis==3.4.0
redis==3.5.3
Whoosh==2.7.4
scikit-learn==1.0.2
watchdog==2.1.7
inotifyrecursive==0.3.5
ocrmypdf==13.4.3
python-magic==0.4.25
gunicorn==20.1.0
```

Each observed version equals the pinned literal in `Pipfile.lock`: `Django==4.0.4` [`Pipfile.lock:L286`], `django-q==1.3.9` [`Pipfile.lock:L326`], `channels==3.0.4` [`Pipfile.lock:L181`], `channels-redis==3.4.0` [`Pipfile.lock:L189`], `redis==3.5.3` [`Pipfile.lock:L986`], `whoosh==2.7.4` [`Pipfile.lock:L1442`], `scikit-learn==1.0.2` [`Pipfile.lock:L1154`], `watchdog==2.1.7` [`Pipfile.lock:L1358`], `inotifyrecursive==0.3.5` [`Pipfile.lock:L522`], `ocrmypdf==13.4.3` [`Pipfile.lock:L678`], `python-magic==0.4.25` [`Pipfile.lock:L916`], `gunicorn==20.1.0` [`Pipfile.lock:L361`].

**Topology.** The documented multi‑process topology (defined in `docker/supervisord.conf`) was brought up natively: a **Redis** broker, the **django‑q** worker (`qcluster`) [`docker/supervisord.conf:L28-L29`], the filesystem **watcher** (`document_consumer`) [`docker/supervisord.conf:L19-L20`], and the **ASGI/WebSocket** server (`gunicorn … paperless.asgi:application`) [`docker/supervisord.conf:L10-L11`], over the default **SQLite** DB at `DATA_DIR/db.sqlite3` [`src/paperless/settings.py:L300`].

**Isolation (read‑only guarantee).** To leave the repository byte‑for‑byte unchanged, the runtime was pointed at throwaway directories outside the repo via environment variables (`PAPERLESS_DATA_DIR`, `PAPERLESS_MEDIA_ROOT`, `PAPERLESS_CONSUMPTION_DIR` [`src/paperless/settings.py:L66,L61,L78`]), plus `PAPERLESS_REDIS=redis://localhost:6379` and `PAPERLESS_OCR_OUTPUT_TYPE=pdf`. The consumption directory, media tree, SQLite DB, Whoosh index, and logs all lived under `/tmp/blitzy_obs/…`. All temporary scripts and test inputs were removed afterward.

**Service startup, observed.**

```console
$ setsid ../venv/bin/python manage.py qcluster > /tmp/blitzy_obs/run/qcluster.log 2>&1 &   # from src/
$ head -12 /tmp/blitzy_obs/run/qcluster.log
05:46:58 [Q] INFO Q Cluster november-sink-oklahoma-zebra starting.
05:46:58 [Q] INFO Process-1:1 ready for work at 68284
05:46:58 [Q] INFO Process-1:2 ready for work at 68285
05:46:58 [Q] INFO Process-1:3 ready for work at 68286
05:46:58 [Q] INFO Process-1:4 ready for work at 68287
05:46:58 [Q] INFO Process-1:5 ready for work at 68288
05:46:58 [Q] INFO Process-1:6 ready for work at 68289
05:46:58 [Q] INFO Process-1:7 ready for work at 68290
05:46:58 [Q] INFO Process-1:8 ready for work at 68291
05:46:58 [Q] INFO Process-1:9 ready for work at 68292
05:46:58 [Q] INFO Process-1:10 ready for work at 68293
05:46:58 [Q] INFO Process-1:11 ready for work at 68294
$ grep -E "monitoring at|guarding cluster|pushing tasks|running\." /tmp/blitzy_obs/run/qcluster.log
05:46:58 [Q] INFO Process-1:12 monitoring at 68295
05:46:58 [Q] INFO Process-1 guarding cluster november-sink-oklahoma-zebra
05:46:58 [Q] INFO Process-1:13 pushing tasks at 68296
05:46:58 [Q] INFO Q Cluster november-sink-oklahoma-zebra running.
```

(The two blocks above are the verbatim `head -12` and a `grep` of the same file. The `head -12` already shows the `Q Cluster … starting.` line followed by *every* `ready for work` worker — here `Process-1:1` through `Process-1:11` — so no worker line is elided between the two blocks; the `grep` then selects the lifecycle lines that come *after* the workers: the monitor (`Process-1:12`), the cluster guard, the task pusher (`Process-1:13`), and the final `running.` line. The number of `ready for work` workers equals the host CPU/worker count and is therefore environment‑dependent — on this host it was 11, so the monitor and pusher fall at `:12` and `:13`.)

```console
$ setsid ../venv/bin/python manage.py document_consumer > /tmp/blitzy_obs/run/consumer.log 2>&1 &   # from src/
$ cat /tmp/blitzy_obs/run/consumer.log
[2026-07-01 05:48:03,005] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /tmp/blitzy_obs/consume
```

The watcher's `"Using inotify to watch directory for changes: …"` line confirms the **inotify** watch mode was selected — the default, because `CONSUMER_POLLING` defaults to `0` [`src/paperless/settings.py:L478`] and the code takes the inotify branch when `settings.CONSUMER_POLLING == 0 and INotify` [`src/documents/management/commands/document_consumer.py:L178-L179`], whose `handle_inotify()` logs that exact message [`src/documents/management/commands/document_consumer.py:L200`]. This distinction matters for Q1 below.

```console
$ setsid ../venv/bin/gunicorn -c ../gunicorn.conf.py paperless.asgi:application > /tmp/blitzy_obs/run/gunicorn.log 2>&1 &   # from src/
$ head -4 /tmp/blitzy_obs/run/gunicorn.log
[2026-07-01 05:48:25 +0000] [69639] [INFO] Starting gunicorn 20.1.0
[2026-07-01 05:48:25 +0000] [69639] [INFO] Listening at: http://0.0.0.0:8000 (69639)
[2026-07-01 05:48:25 +0000] [69639] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-01 05:48:25 +0000] [69639] [INFO] Server is ready. Spawning workers
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/api/
HTTP 200
```

**Log format & file location.** The application log is written to `DATA_DIR/log/paperless.log` [`src/paperless/settings.py:L395`], and the `paperless` logger is configured at `DEBUG` [`src/paperless/settings.py:L409`]. The line format is `"[{asctime}] [{levelname}] [{name}] {message}"` [`src/paperless/settings.py:L378`], which matches every quoted line below.

**The six evidence channels used throughout.** (a) `paperless.log`; (b) the `qcluster` worker stdout; (c) the WebSocket `status_updates` stream — captured two ways (see Q3): a Redis channel‑layer **group tap**, and a fully **authenticated end‑to‑end WebSocket client**; (d) the SQLite `Document` row via the Django ORM; (e) on‑disk artifacts under `media/documents/{originals,archive,thumbnails}` and the Whoosh index dir; (f) the **django‑q `Task` record** (task name + success/result), which corroborates the task name independently of the logs.

**Test input.** A throwaway PNG carrying text (so OCR produces real content) was generated with ImageMagick, and its MD5 was recorded (it is quoted again in Q4/Q5):

```console
$ convert -size 1000x300 xc:white -gravity center -pointsize 40 -fill black \
    -annotate +0-40 "Blitzy Runtime Observation Document" \
    -annotate +0+40 "Ingestion pipeline evidence capture 2025" \
    /tmp/blitzy_obs/testinput/obs_demo.png
$ md5sum /tmp/blitzy_obs/testinput/obs_demo.png
f08328e0bcec1b5c0bbf70dc8b9d3d96  /tmp/blitzy_obs/testinput/obs_demo.png
```

---

## Q1 — Detection & Handoff

**Question.** When Paperless‑NGX is running and a new document appears, what observable runtime behavior shows how the document is *detected* and *handed off* for processing?

**Answer.** The `document_consumer` **filesystem watcher** detects the new file (via inotify), logs it, and **enqueues an asynchronous django‑q task named `documents.tasks.consume_file`**; the `qcluster` worker then picks it up. Two other entry points enqueue the *same* task: the REST upload endpoint and the email fetcher.

### Observed: consumption‑directory path

The file was dropped into the watched directory, and the detection/handoff lines were then captured verbatim. Because these three lines are emitted by three different processes to three different streams, each is grepped from the stream it actually lands in — the human‑readable detection line from `paperless.log`, django‑q's `Enqueued` acknowledgement from the `document_consumer` process's own stdout capture (`consumer.log`, established in §0), and the worker pickup from the `qcluster` stdout (`qcluster.log`):

```console
$ cp /tmp/blitzy_obs/testinput/obs_demo.png /tmp/blitzy_obs/consume/          # trigger
$ grep -hE "Adding .* to the task queue\." /tmp/blitzy_obs/data/log/paperless.log     # detection (paperless.log)
[2026-07-01 05:51:58,769] [INFO] [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/obs_demo.png to the task queue.
$ grep -hE "Enqueued [0-9]+$" /tmp/blitzy_obs/run/consumer.log                        # enqueue ack (document_consumer stdout)
05:51:58 [Q] INFO Enqueued 1
$ grep -hE "processing \[obs_demo" /tmp/blitzy_obs/run/qcluster.log                  # worker pickup (qcluster stdout)
05:51:58 [Q] INFO Process-1:5 processing [obs_demo.png]
```

**Why the `Enqueued 1` acknowledgement is captured from `consumer.log`, not from `paperless.log` or `qcluster.log`.** The detection line `Adding … to the task queue.` is written by the `paperless.management.consumer` logger, so it reaches `paperless.log` (the `paperless` logger is wired to the `file_paperless` handler [`src/paperless/settings.py:L409`]). The `Enqueued 1` line, by contrast, is emitted by django‑q's *own* logger the instant the task is pushed onto the broker — `logger.info(f"Enqueued {enqueue_id}")` [`django_q/tasks.py:L74`] — and that `"django-q"` logger [`django_q/conf.py:L207`] sets `logger.propagate = False` [`django_q/conf.py:L212`] and attaches a bare `logging.StreamHandler()` [`django_q/conf.py:L216`], which defaults to the *calling* process's `stderr`. Because the enqueue executes inside the `document_consumer` process (the `async_task(…)` call at [`src/documents/management/commands/document_consumer.py:L86-L91`]), that line is written to `document_consumer`'s own `stderr` — captured here in `consumer.log` — and, being non‑propagating, it never reaches the `paperless.log` file handler. (`qcluster.log` does carry `Enqueued …` lines, but only at startup, when the scheduler enqueues its periodic tasks — a different set of tasks from this drop.) The `Process-1:5 processing [obs_demo.png]` line comes from yet another process, the `qcluster` worker, which is why it appears only in `qcluster.log`.

**How the file was detected — the observed inotify path, not the polling fallback.** The runtime log `"Using inotify to watch directory for changes: …"` (captured in §0) proves the watcher took the **inotify** branch. That branch is selected when `settings.CONSUMER_POLLING == 0 and INotify` is truthy [`src/documents/management/commands/document_consumer.py:L178-L179`]; otherwise the watcher falls back to polling [`src/documents/management/commands/document_consumer.py:L180-L181`]. On the observed inotify branch, `handle_inotify()` logs that message [`src/documents/management/commands/document_consumer.py:L199-L200`], arms inotify with `inotify_flags = flags.CLOSE_WRITE | flags.MOVED_TO` [`src/documents/management/commands/document_consumer.py:L203`], and adds a watch on the directory [`src/documents/management/commands/document_consumer.py:L207`]. Its loop then reads raw inotify events via `inotify.read(timeout=1000)` [`src/documents/management/commands/document_consumer.py:L216`], reconstructs each `filepath = os.path.join(path, event.name)` [`src/documents/management/commands/document_consumer.py:L221`], and — after a debounce interval — calls the module‑level `_consume(filepath)` [`src/documents/management/commands/document_consumer.py:L230`]. The watchdog callbacks `Handler.on_created` [`src/documents/management/commands/document_consumer.py:L129-L130`] and `Handler.on_moved` [`src/documents/management/commands/document_consumer.py:L132-L133`] are **not** exercised on this path; they belong to the polling fallback, where `handle_polling()` schedules a `PollingObserver` with `Handler()` [`src/documents/management/commands/document_consumer.py:L185-L188`].

**How it was handed off.** Both branches converge on the same module‑level `_consume()`, whose logger is `"paperless.management.consumer"` [`src/documents/management/commands/document_consumer.py:L24`]. It emits the exact detection line `f"Adding {filepath} to the task queue."` [`src/documents/management/commands/document_consumer.py:L85`] — the line observed above — immediately before calling `async_task("documents.tasks.consume_file", filepath, …)` [`src/documents/management/commands/document_consumer.py:L86-L91`], where the task literal is `"documents.tasks.consume_file"` [`src/documents/management/commands/document_consumer.py:L87`] and `task_name` is set to the file's basename [`src/documents/management/commands/document_consumer.py:L90`]. In the captured output, `Enqueued 1` is django‑q acknowledging the queued task and `Process-1:5 processing [obs_demo.png]` is the `qcluster` worker dequeuing it (the bracketed name is that `task_name`).

### Corroboration: the django‑q `Task` record names the task independently of the logs

```console
$ ../venv/bin/python -c "import os,django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); from django_q.models import Task; [print(f'name={t.name!r} func={t.func} success={t.success} result={t.result!r}') for t in Task.objects.filter(func='documents.tasks.consume_file', success=True).order_by('started')]"
```
```text
name='obs_demo.png' func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
name='rest_demo.png' func=documents.tasks.consume_file success=True result='Success. New document id 2 created'
```

The first persisted record is the consumption‑directory drop; it shows `func=documents.tasks.consume_file` — the same handoff target, observed from the task queue's own database record rather than a log line. (The second row is the REST upload, discussed next.) The task's function is dispatched by django‑q's worker to `documents.tasks.consume_file` [`src/documents/tasks.py:L184`], which in turn calls `Consumer().try_consume_file(...)` [`src/documents/tasks.py:L236`].

### Observed: REST upload path

The REST endpoint returns immediately after enqueuing. The exact request that produced the observed body and status code:

```console
$ curl -sS -w "\nHTTP_STATUS:%{http_code}\n" -H "Authorization: Token <REDACTED_TOKEN>" \
       -F "document=@/tmp/blitzy_obs/testinput/rest_demo.png" \
       http://localhost:8000/api/documents/post_document/
```
```text
"OK"
HTTP_STATUS:200
```

- `PostDocumentView.post` [`src/documents/views.py:L497`] writes the upload to a `tempfile.NamedTemporaryFile(prefix="paperless-upload-", dir=settings.SCRATCH_DIR, delete=False)` [`src/documents/views.py:L512-L516`], captures its name as `temp_filename` [`src/documents/views.py:L519`], generates `task_id = str(uuid.uuid4())` [`src/documents/views.py:L521`], enqueues `async_task("documents.tasks.consume_file", temp_filename, …)` [`src/documents/views.py:L523-L533`] (task literal at [`src/documents/views.py:L524`], the `temp_filename` positional argument at [`src/documents/views.py:L525`], the `task_id` keyword at [`src/documents/views.py:L531`]), and returns `Response("OK")` [`src/documents/views.py:L535`] — hence the body `"OK"` and HTTP `200` observed above. The route is registered as `post_document` at `documents/post_document/` [`src/paperless/urls.py:L56-L59`], nested under the `^api/` parent prefix [`src/paperless/urls.py:L39-L41`]; together these form the full path `api/documents/post_document/` used in the `curl` above.
- That upload was consumed too, confirming the REST path reaches the identical task. The command below shows the worker dequeuing it, the consumer starting, and the `paperless-upload-`‑prefixed temp file (created at `src/documents/views.py:L512`) being cleaned up after storage:

```console
$ grep -hE "processing \[rest_demo|Consuming rest_demo|Deleting file /tmp/paperless/paperless-upload" \
        /tmp/blitzy_obs/run/qcluster.log /tmp/blitzy_obs/data/log/paperless.log | sort -u
```
```text
05:53:20 [Q] INFO Process-1:6 processing [rest_demo.png]
[2026-07-01 05:53:20,141] [INFO] [paperless.consumer] Consuming rest_demo.png
[2026-07-01 05:53:21,380] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-udh9kr46
```

  Its persisted `Task` record is the **second row already shown above** (`name='rest_demo.png' … result='Success. New document id 2 created'`), confirming the REST path enqueues the same `documents.tasks.consume_file` target and that the temp filename observed here (`/tmp/paperless/paperless-upload-udh9kr46`) carries the `paperless-upload-` prefix from `src/documents/views.py:L513`.

- The **email** entry point also enqueues the same task: `async_task("documents.tasks.consume_file", path=temp_filename, …)` [`src/paperless_mail/mail.py:L336-L338`] (task literal at [`src/paperless_mail/mail.py:L337`], `path=temp_filename` at [`src/paperless_mail/mail.py:L338`]). This path was **not** exercised at runtime here — no mail server was configured — so it is cited from source and explicitly *not* asserted as observed.

**Reasoning.** Detection and handoff are deliberately decoupled: the watcher (or the REST view, or the mail fetcher) only *enqueues* work and returns; the actual processing happens later in a separate `qcluster` worker process. The single, observable handoff contract across all three entry points is the django‑q task name `documents.tasks.consume_file`, which is visible both in the queue logs and in the persisted `Task` record.

---

## Q2 — Stage Transitions into Parsing / Classification / Indexing

**Question.** What log messages, task names, or state changes indicate the transition from initial detection into *parsing*, *classification*, and *indexing*?

**Answer.** The `qcluster` worker runs `documents.tasks.consume_file()` [`src/documents/tasks.py:L184`] (logger `"paperless.tasks"` [`src/documents/tasks.py:L29`]), which delegates to `Consumer().try_consume_file(...)` [`src/documents/tasks.py:L236`]. `Consumer` (logger `"paperless.consumer"` [`src/documents/consumer.py:L52-L54`]) emits a fixed sequence of named log lines as it moves through the stages, and — after the row is stored — fires the `document_consumption_finished` signal, which is what drives **classification** and **Whoosh indexing**.

### Observed: the full correlated `paperless.log` sequence for one ingestion

The command below prints exactly the log lines the consume‑directory run appended (`paperless.log` lines 4–20; lines 1–3 were the startup baseline). It is quoted verbatim and in full — no line is elided:

```console
$ sed -n '4,20p' /tmp/blitzy_obs/data/log/paperless.log
```
```text
[2026-07-01 05:51:58,769] [INFO] [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/obs_demo.png to the task queue.
[2026-07-01 05:51:58,894] [INFO] [paperless.consumer] Consuming obs_demo.png
[2026-07-01 05:51:58,894] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-01 05:51:58,895] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-01 05:51:58,897] [DEBUG] [paperless.consumer] Parsing obs_demo.png...
[2026-07-01 05:51:58,958] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/blitzy_obs/consume/obs_demo.png: 'dpi'
[2026-07-01 05:51:58,958] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
[2026-07-01 05:51:58,959] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/blitzy_obs/consume/obs_demo.png', 'output_file': '/tmp/paperless/paperless-1f_tzpu1/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdf', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-1f_tzpu1/sidecar.txt', 'image_dpi': 120}
[2026-07-01 05:51:59,732] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-01 05:51:59,732] [DEBUG] [paperless.consumer] Generating thumbnail for obs_demo.png...
[2026-07-01 05:51:59,736] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-1f_tzpu1/archive.pdf[0] /tmp/paperless/paperless-1f_tzpu1/convert.png
[2026-07-01 05:51:59,991] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-1f_tzpu1/convert.png -out /tmp/paperless/paperless-1f_tzpu1/thumb_optipng.png
[2026-07-01 05:52:01,898] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-01 05:52:01,901] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-01 05:52:01,921] [DEBUG] [paperless.consumer] Deleting file /tmp/blitzy_obs/consume/obs_demo.png
[2026-07-01 05:52:01,966] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-1f_tzpu1
[2026-07-01 05:52:01,967] [INFO] [paperless.consumer] Document 2026-07-01 obs_demo consumption finished
```

Mapping each observed line to its source (every stage is on the `paperless.consumer`/parser loggers; line numbers are exact):

| Observed line | Stage | Source |
|---|---|---|
| `Consuming obs_demo.png` | pipeline start | `self.log("info", f"Consuming {self.filename}")` [`src/documents/consumer.py:L215`] |
| `Detected mime type: image/png` | MIME detection | `self.log("debug", f"Detected mime type: {mime_type}")` [`src/documents/consumer.py:L221`] (value from `magic.from_file(...)` [`src/documents/consumer.py:L219`]) |
| `Parser: RasterisedDocumentParser` | parser selection | `self.log("debug", f"Parser: {type(document_parser).__name__}")` [`src/documents/consumer.py:L246`] |
| `Parsing obs_demo.png...` | **parsing** | `self.log("debug", "Parsing {}...".format(self.filename))` [`src/documents/consumer.py:L260`], immediately after the `parsing_document` progress milestone [`src/documents/consumer.py:L259`] |
| `paperless.parsing.tesseract … Calling OCRmyPDF with args: {… 'output_type': 'pdf' …}` | OCR parse | tesseract parser, logger `"paperless.parsing.tesseract"` [`src/paperless_tesseract/parsers.py:L24`] |
| `Generating thumbnail for obs_demo.png...` | thumbnailing | `self.log("debug", f"Generating thumbnail for {self.filename}...")` [`src/documents/consumer.py:L263`], after the `generating_thumbnail` milestone [`src/documents/consumer.py:L264`] |
| `paperless.classifier … model does not exist (yet) …` | **classifier *loading* (pre‑store)** — **not** proof that the classification handlers ran | `load_classifier()` [`src/documents/classifier.py:L30`] emits this at [`src/documents/classifier.py:L32-L35`]; it is *called* before storage at [`src/documents/consumer.py:L292`] (see the timing note below) |
| `Saving record to database` | persistence | `self.log("debug", "Saving record to database")` [`src/documents/consumer.py:L387`], run when `_store(...)` is *called* at [`src/documents/consumer.py:L301`] |
| `Document … consumption finished` | completion | `self.log("info", "Document {} consumption finished".format(document))` [`src/documents/consumer.py:L373`] |

### Timing note: classifier *loading* precedes the store; the classification *handlers* run after it

The observed timestamps make the ordering unambiguous. The `paperless.classifier` line is stamped **`05:52:01,898`** while `Saving record to database` is stamped **`05:52:01,901`** — the classifier message is emitted **~3 ms *before*** the store. That is because `load_classifier()` is *called* at [`src/documents/consumer.py:L292`], ahead of the `_store(...)` call at [`src/documents/consumer.py:L301`]. Consequently the `paperless.classifier` line proves only that the classifier was **loaded** — and, with no model on disk yet, that automatic matching was skipped [`src/documents/classifier.py:L32-L35`]. It does **not** prove that the post‑save classification handlers (`set_correspondent`/`set_document_type`/`set_tags`) executed; those are driven by a signal that fires *after* the store, as shown next.

### What drives classification and indexing: the `document_consumption_finished` signal

*After* `_store()` persists the row, the consumer fires the signal inside the same atomic transaction:

```python
# src/documents/consumer.py:L306-L311 (verbatim source)
document_consumption_finished.send(
    sender=self.__class__,
    document=document,
    logging_group=self.logging_group,
    classifier=classifier,
)
```

`DocumentsConfig.ready()` connects **six** receivers to that signal [`src/documents/apps.py:L22-L27`]:

```python
document_consumption_finished.connect(add_inbox_tags)      # src/documents/apps.py:L22
document_consumption_finished.connect(set_correspondent)   # src/documents/apps.py:L23
document_consumption_finished.connect(set_document_type)   # src/documents/apps.py:L24
document_consumption_finished.connect(set_tags)            # src/documents/apps.py:L25
document_consumption_finished.connect(set_log_entry)       # src/documents/apps.py:L26
document_consumption_finished.connect(add_to_index)        # src/documents/apps.py:L27
```

- **Classification** = `set_correspondent` [`src/documents/signals/handlers.py:L35`], `set_document_type` [`src/documents/signals/handlers.py:L101`], and `set_tags` [`src/documents/signals/handlers.py:L168`] (logger `"paperless.handlers"` [`src/documents/signals/handlers.py:L27`]). They combine the ML classifier (`src/documents/classifier.py`; persisted model `MODEL_FILE = DATA_DIR/classification_model.pickle` [`src/paperless/settings.py:L74`]) with rule matching in `src/documents/matching.py`. **These handlers produced no log line in this run:** with no trained model and no matching rules defined, each took its no‑assignment path and returned silently (a handler logs only when it actually *assigns* something, e.g. `set_correspondent`'s "Assigning correspondent …" branch). Their execution is therefore *wired* (the six `connect()` calls above) and *source‑verified*, but is **not** evidenced by the `paperless.classifier` line quoted earlier — that line is classifier *loading*, a separate pre‑store step.
- **Indexing** = `add_to_index` [`src/documents/signals/handlers.py:L428`], which calls `index.add_or_update_document(document)` [`src/documents/signals/handlers.py:L431`]; that opens a Whoosh `AsyncWriter(open_index())` (writer at [`src/documents/index.py:L66`], function at [`src/documents/index.py:L118`], logger `"paperless.index"` [`src/documents/index.py:L28`]). **This handler leaves observable state**, so it is the concrete runtime proof that the signal fired and its receivers ran: under Q4, a full‑text search of the Whoosh index returns the new document (`id=1`). The signal's effect is thus verified by the index entry — not by the classifier log.

### Corroboration: task name + success in the django‑q record

The `qcluster` stdout shows the framework side of the same run. The command below selects just the worker lines for this file's successful task (worker `Process-1:5`):

```console
$ grep -E "Process-1:5 processing \[obs_demo|Processed \[obs_demo|recycled worker Process-1:5" \
        /tmp/blitzy_obs/run/qcluster.log
```
```text
05:51:58 [Q] INFO Process-1:5 processing [obs_demo.png]
05:52:01 [Q] INFO Processed [obs_demo.png]
05:52:02 [Q] INFO recycled worker Process-1:5
```

The persisted `Task` record — the first `success=True` row already shown under Q1 — independently confirms the task name and outcome: `func=documents.tasks.consume_file`, `result='Success. New document id 1 created'`.

**Reasoning.** The stage transitions are observable as a deterministic, ordered log sequence on the `paperless.consumer` logger (Consuming → Detected mime type → Parser → Parsing → Generating thumbnail → Saving record → consumption finished), with parsing delegated to a MIME‑specific parser logger (`paperless.parsing.tesseract` here). Classification and indexing are **not** inline log steps: they are signal‑driven consequences of `document_consumption_finished`, sent *after* the store at [`src/documents/consumer.py:L306`] and wired centrally in `apps.py` [`src/documents/apps.py:L22-L27`]. Because the classification handlers had nothing to assign in this run they emitted no log, so their execution is source‑verified rather than log‑evidenced; the one signal receiver that leaves observable state — `add_to_index` — is what runtime‑proves the signal fired (the Whoosh hit under Q4). Note the `paperless.classifier` line is *not* part of this post‑store phase: it is classifier **loading**, emitted before the store (see the timing note above).

> **Edge case (barcode split).** Before normal consumption, `consume_file()` optionally scans for separator barcodes; if found it splits the file and returns the string `"File successfully split"` *instead of* consuming [`src/documents/tasks.py:L233`]. This path is disabled by default (`CONSUMER_ENABLE_BARCODES` defaults to `False` [`src/paperless/settings.py:L502-L503`], via `__get_boolean(default="NO")` [`src/paperless/settings.py:L34`]), so the normal run above went straight to `try_consume_file` [`src/documents/tasks.py:L236`].

**Edge case (unsupported type) — observed.** A file that no parser can handle is rejected by **two distinct guards**, exercised here as a targeted follow‑up. (1) The **watcher** rejects unknown *extensions* before it ever enqueues a task — `_consume()` calls `is_file_ext_supported()` [`src/documents/management/commands/document_consumer.py:L54-L55`] — so dropping a `.zip` into the consumption directory is turned away up front:

```text
[2026-07-01 08:59:45,784] [WARNING] [paperless.management.consumer] Not consuming file /tmp/blitzy_obs/consume/obs_unsupported.zip: Unknown file extension.
```

(2) The **consumer** independently rejects unsupported *MIME types* once a file actually reaches `try_consume_file()` — the non‑watcher entry points (REST upload and email) enqueue `consume_file` directly and so bypass the extension guard above. `magic.from_file(...)` detects the type [`src/documents/consumer.py:L219`] and, when `get_parser_class_for_mime_type(...)` returns no parser [`src/documents/consumer.py:L223-L224`], the consumer calls `self._fail(MESSAGE_UNSUPPORTED_TYPE, f"Unsupported mime type {mime_type}")` [`src/documents/consumer.py:L225`], where `MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` [`src/documents/consumer.py:L44`]. Enqueuing the same zip's bytes directly reproduced this second guard verbatim:

```text
[2026-07-01 09:00:03,252] [INFO] [paperless.consumer] Consuming obs_unsupported.zip
[2026-07-01 09:00:03,253] [DEBUG] [paperless.consumer] Detected mime type: application/zip
[2026-07-01 09:00:03,256] [ERROR] [paperless.consumer] Unsupported mime type application/zip
```

Because `_fail()` broadcasts a `FAILED` frame at `100/100` *before* raising `ConsumerError` [`src/documents/consumer.py:L78-L81`], the channel‑layer tap captured a payload of the same shape as the duplicate case in Q5 — the identical seven `_send_progress()` keys — but carrying `"status": "FAILED"` with `"message": "unsupported_type"`:

```text
{"type": "status_update", "data": {"filename": "obs_unsupported.zip", "task_id": "5eed493a-9762-4632-99a8-d1c3e25057d5", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "unsupported_type", "document_id": null}}
```

The django‑q `Task` row corroborates the task‑queue side: `func='documents.tasks.consume_file'`, `success=False`, with `result` beginning `obs_unsupported.zip: Unsupported mime type application/zip : Traceback (most recent call last):`. This is the very same `_fail()` mechanism that produces the duplicate `document_already_exists` frame under Q5 — only the `message` literal differs (`unsupported_type` vs. `document_already_exists`).


---

## Q3 — Progress & Completion reporting

**Question.** How does the system reflect *progress* or *completion* of each stage while the document is being processed?

**Answer.** Progress is broadcast over a **WebSocket channel‑layer group named `"status_updates"`** as structured JSON payloads at **fixed milestones**, built by `Consumer._send_progress()` [`src/documents/consumer.py:L56`]. Each payload carries `filename`, `task_id`, `current_progress`, `max_progress`, `status`, `message`, and `document_id` [`src/documents/consumer.py:L64-L72`] and is sent with `async_to_sync(self.channel_layer.group_send)("status_updates", {"type": "status_update", "data": payload})` [`src/documents/consumer.py:L73-L76`]. Browsers receive these frames through `StatusConsumer`.

### How the payloads were captured (two independent methods, each with its exact script)

Because the `ws/status/` endpoint **requires authentication** (`StatusConsumer.connect()` raises `DenyConnection()` for unauthenticated users [`src/paperless/consumers.py:L13-L15`]), two complementary temporary subscribers were used. The authentication gate itself was verified first.

**Auth gate — an unauthenticated connect is rejected before any frame is sent.** The exact temporary script and its run:

```console
$ cat /tmp/blitzy_obs/scripts/ws_unauth.py
#!/usr/bin/env python
"""Unauthenticated connect probe: confirms StatusConsumer.connect() rejects
unauthenticated clients (DenyConnection -> HTTP 403) before any frame is sent.
"""
import asyncio, websockets


async def main():
    try:
        async with websockets.connect("ws://localhost:8000/ws/status/"):
            print("connected")
    except Exception as e:
        print(f"DENIED: {type(e).__name__}: {e}")


asyncio.run(main())
$ ./venv/bin/python /tmp/blitzy_obs/scripts/ws_unauth.py
DENIED: InvalidStatusCode: server rejected WebSocket connection: HTTP 403
```

### Method 1 — channel‑layer group tap (`ws_tap.py`)

The tap script and the command that launched it (started **before** the file drop so no frame is missed):

```console
$ cat /tmp/blitzy_obs/scripts/ws_tap.py
#!/usr/bin/env python
"""Channel-layer group tap: joins the Redis-backed "status_updates" group
directly and prints every group_send envelope verbatim. Captures exactly what
Consumer._send_progress() broadcasts, independent of any browser client.
"""
import os, sys, json, asyncio, time
sys.path.insert(0, "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/src")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django
django.setup()
from channels.layers import get_channel_layer


async def main():
    layer = get_channel_layer()
    channel = await layer.new_channel()
    await layer.group_add("status_updates", channel)
    print(f"TAP joined group 'status_updates' as {channel}", flush=True)
    while True:
        msg = await layer.receive(channel)
        ts = time.strftime("%H:%M:%S")
        print(f"[TAP {ts}] {json.dumps(msg)}", flush=True)


asyncio.run(main())
$ setsid ./venv/bin/python /tmp/blitzy_obs/scripts/ws_tap.py > /tmp/blitzy_obs/run/ws_tap.out 2>&1 &
```

The tap's verbatim output for the consume‑directory ingestion (its join line plus the six milestone envelopes; later lines in the file belong to the REST upload and the duplicate re‑drop, shown under Q1/Q5):

```text
TAP joined group 'status_updates' as specific.fb9ee44629ca4e8eae63ddc6543851c8!c98370213488447aaab24fa5dc04dad5
[TAP 05:51:58] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
[TAP 05:51:58] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
[TAP 05:51:59] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}}
[TAP 05:52:00] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}}
[TAP 05:52:01] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}}
[TAP 05:52:01] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

### Method 2 — authenticated end‑to‑end WebSocket client (`ws_client.py`)

A Django `sessionid` was minted for the admin user and passed as a cookie so `AuthMiddlewareStack` authenticates the WebSocket scope; the client then prints the JSON frames a browser actually receives. The mint command, the client script, and its launch (also started **before** the drop):

```console
$ # sessionid mint run from src/ (django.setup() imports paperless.settings)
$ SID=$(../venv/bin/python -c "import os,django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from django.contrib.auth.models import User
from django.contrib.sessions.backends.db import SessionStore
u=User.objects.get(username='obsadmin'); s=SessionStore()
s['_auth_user_id']=str(u.id); s['_auth_user_backend']='django.contrib.auth.backends.ModelBackend'
s['_auth_user_hash']=u.get_session_auth_hash(); s.create(); print(s.session_key, end='')")
$ cat /tmp/blitzy_obs/scripts/ws_client.py
#!/usr/bin/env python
"""Authenticated end-to-end WebSocket client. Connects to ws://localhost:8000/ws/status/
using a Django sessionid cookie (argv[1]) so AuthMiddlewareStack marks the scope user
authenticated, then prints every JSON frame the browser would receive.
"""
import sys, asyncio, websockets

SESSIONID = sys.argv[1]
URI = "ws://localhost:8000/ws/status/"


async def main():
    async with websockets.connect(
        URI, extra_headers=[("Cookie", f"sessionid={SESSIONID}")]
    ) as ws:
        print(f"CLIENT connected authenticated to {URI}", flush=True)
        while True:
            frame = await ws.recv()
            print(frame, flush=True)


asyncio.run(main())
$ setsid ../venv/bin/python /tmp/blitzy_obs/scripts/ws_client.py "$SID" > /tmp/blitzy_obs/run/ws_client.out 2>&1 &   # from src/
```

The six verbatim frames the authenticated client received for `obs_demo.png` (later lines belong to the REST upload and the duplicate, shown under Q1/Q5):

```text
CLIENT connected authenticated to ws://localhost:8000/ws/status/
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
{"filename": "obs_demo.png", "task_id": "3a1894f0-1346-4f31-b297-ca9714d7f3e6", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
```

Both methods received identical milestone data: the tap shows each envelope as `{"type": "status_update", "data": {…}}`, while the authenticated client receives exactly the inner `data` object. Each milestone maps to a specific `_send_progress()` call site (exact literals verified against source):

| Observed `status` / `current_progress` / `message` | Source |
|---|---|
| `STARTING` `0` `new_file` | `self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)` [`src/documents/consumer.py:L202`] (`MESSAGE_NEW_FILE = "new_file"` [`src/documents/consumer.py:L43`]) |
| `WORKING` `20` `parsing_document` | `self._send_progress(20, 100, "WORKING", MESSAGE_PARSING_DOCUMENT)` [`src/documents/consumer.py:L259`] (`MESSAGE_PARSING_DOCUMENT = "parsing_document"` [`src/documents/consumer.py:L45`]) |
| `WORKING` `70` `generating_thumbnail` | `self._send_progress(70, 100, "WORKING", MESSAGE_GENERATING_THUMBNAIL)` [`src/documents/consumer.py:L264`] (`MESSAGE_GENERATING_THUMBNAIL = "generating_thumbnail"` [`src/documents/consumer.py:L46`]) |
| `WORKING` `90` `parse_date` | `self._send_progress(90, 100, "WORKING", MESSAGE_PARSE_DATE)` [`src/documents/consumer.py:L274`] — sent only when the parser returned no date (`if not date:` [`src/documents/consumer.py:L273`]) |
| `WORKING` `95` `save_document` | `self._send_progress(95, 100, "WORKING", MESSAGE_SAVE_DOCUMENT)` [`src/documents/consumer.py:L294`] (`MESSAGE_SAVE_DOCUMENT = "save_document"` [`src/documents/consumer.py:L48`]) |
| `SUCCESS` `100` `finished`, `document_id: 1` | `self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)` [`src/documents/consumer.py:L375`] (`MESSAGE_FINISHED = "finished"` [`src/documents/consumer.py:L49`]) |

Note the payload keys match the dict built at [`src/documents/consumer.py:L64-L72`] exactly, and `document_id` is `null` for every intermediate frame and becomes the real id (`1`) only in the terminal `SUCCESS` frame — because only that call passes `document.id` [`src/documents/consumer.py:L375`]. (During parsing the parser can also emit intermediate `WORKING` frames rescaled via `p = int((current_progress / max_progress) * 50 + 20)` [`src/documents/consumer.py:L237-L240`]; the six above are the fixed stage milestones.)

### Relay to browsers

`StatusConsumer` (a `WebsocketConsumer` [`src/paperless/consumers.py:L9`]) joins the `"status_updates"` group on connect [`src/paperless/consumers.py:L17-L20`] and, on each `status_update` event [`src/paperless/consumers.py:L29`], sends `self.send(json.dumps(event["data"]))` [`src/paperless/consumers.py:L33`] — which is exactly why the end‑to‑end client frames above are the JSON‑serialized inner `data` payload (whereas the tap prints the whole envelope). Routing is `ProtocolTypeRouter({… "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))})` [`src/paperless/asgi.py:L17-L21`] (the `websocket` entry at [`src/paperless/asgi.py:L20`]), and the endpoint is registered at `re_path(r"ws/status/$", StatusConsumer.as_asgi())` [`src/paperless/urls.py:L137`].

**Reasoning.** Progress reporting is a **transient, push‑based** mechanism: the worker computes coarse, fixed percentages at each stage boundary and broadcasts them to any subscribed browser via Redis + Channels. Completion of the whole pipeline is signalled by the terminal `SUCCESS`/`100`/`finished` frame that additionally carries the new `document_id`, letting the UI link straight to the stored document. As shown next (Q4), these percentages live only on the wire — they are never written to the database.


---

## Q4 — Final State & Storage

**Question.** After processing finishes, what observable evidence shows *where* the document's data ends up and how its *final state* is recorded?

**Answer.** The final state is a persisted **`Document` row** plus three **on‑disk artifacts** (original, archive, thumbnail) and one **full‑text index entry**.

> **KEY FINDING — final state has no status column.** The `Document` model has **no `status`/`state`/`processing` field of any kind**. The transient processing status (`STARTING`/`WORKING`/`SUCCESS`/`FAILED`) exists **only in the WebSocket stream** (Q3); it is never persisted. A document's "final state" is therefore represented purely by *the existence of the row and its files*, not by a status flag.

### Observed: the persisted `Document` row

```console
$ ../venv/bin/python - <<'PY'      # run from src/ ; PAPERLESS_* env exported per setup
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
title: obs_demo
mime_type: image/png
checksum: f08328e0bcec1b5c0bbf70dc8b9d3d96
archive_checksum: 45ceabdcb65c8ddd078ab78c5ed00861
filename: 0000001.png
archive_filename: 0000001.pdf
storage_type: unencrypted
created: 2026-07-01T05:51:57.763855+00:00
added: 2026-07-01T05:52:01.902200+00:00
content[:100]: 'Blitzy Runtime Observation Document\n\nIngestion pipeline evidence capture 2025'
```

Field‑by‑field, mapped to the model `class Document(models.Model)` [`src/documents/models.py:L88`]:
- `content` — the OCR‑extracted text (`TextField`, help text "The raw, text-only data of the document. This field is primarily used for searching.") [`src/documents/models.py:L117-L124`]. The observed value is exactly the text rendered into the test PNG, proving OCR populated it.
- `mime_type = image/png` [`src/documents/models.py:L126`]; `storage_type = unencrypted` [`src/documents/models.py:L161`].
- `checksum = f08328e0bcec1b5c0bbf70dc8b9d3d96` — `CharField(max_length=32, editable=False, unique=True)` [`src/documents/models.py:L135-L141`]. This is the MD5 of the **original** file (its match to the on‑disk original and its role as the dedup key are shown below and under Q5).
- `archive_checksum = 45ceabdcb65c8ddd078ab78c5ed00861` [`src/documents/models.py:L143`] — MD5 of the generated archive PDF.
- `filename = 0000001.png` [`src/documents/models.py:L176-L184`] and `archive_filename = 0000001.pdf` [`src/documents/models.py:L186-L194`] — both `FilePathField(unique=True)`.
- `created` [`src/documents/models.py:L152`] and `added` [`src/documents/models.py:L169-L174`] timestamps.

### Observed: the on‑disk artifacts

```console
$ find /tmp/blitzy_obs/media/documents -type f -name '0000001.*' -printf '%s bytes  %p\n' | sort
```
```text
10437 bytes  /tmp/blitzy_obs/media/documents/originals/0000001.png
16679 bytes  /tmp/blitzy_obs/media/documents/archive/0000001.pdf
7490 bytes  /tmp/blitzy_obs/media/documents/thumbnails/0000001.png
```

The `find` was scoped to the traced document (`0000001.*`); a second document (id=2, the REST‑upload demo exercised in Q1) is independently present on disk as `0000002.*` under the same directories. These three directories are `ORIGINALS_DIR = media/documents/originals` [`src/paperless/settings.py:L62`], `ARCHIVE_DIR = media/documents/archive` [`src/paperless/settings.py:L63`], and `THUMBNAIL_DIR = media/documents/thumbnails` [`src/paperless/settings.py:L64`]. All of this happens inside one `transaction.atomic()` block [`src/documents/consumer.py:L298`]: the row is created by `self._store(...)` [`src/documents/consumer.py:L301`], the `document_consumption_finished` signal is then sent [`src/documents/consumer.py:L306`], and only afterwards — under `FileLock` [`src/documents/consumer.py:L315`] — are the files copied into place via `self._write(...)` (original [`src/documents/consumer.py:L319`], thumbnail [`src/documents/consumer.py:L321-L325`], archive [`src/documents/consumer.py:L333-L337`]), the `archive_checksum` computed [`src/documents/consumer.py:L339-L342`], and finally `document.save()` [`src/documents/consumer.py:L346`]. Comparing MD5s confirms the stored original is byte‑identical to the input:

```console
$ md5sum /tmp/blitzy_obs/media/documents/originals/0000001.png /tmp/blitzy_obs/media/documents/archive/0000001.pdf
f08328e0bcec1b5c0bbf70dc8b9d3d96  /tmp/blitzy_obs/media/documents/originals/0000001.png
45ceabdcb65c8ddd078ab78c5ed00861  /tmp/blitzy_obs/media/documents/archive/0000001.pdf
```

The first hash, `f08328e0bcec1b5c0bbf70dc8b9d3d96`, is exactly the row's `checksum` (and equals the input file's MD5 — see Q5), confirming the stored original is byte‑for‑byte the file that was ingested; the second, `45ceabdcb65c8ddd078ab78c5ed00861`, is exactly the row's `archive_checksum` [`src/documents/models.py:L143`], the MD5 of the generated archive PDF.


### Observed: the full‑text index entry

The Whoosh index at `INDEX_DIR = DATA_DIR/index` [`src/paperless/settings.py:L73`] was updated on consumption, and the new document is searchable by its OCR content:

```console
$ ls -1 /tmp/blitzy_obs/data/index/
MAIN_6p3mqxtmc1hvwivd.seg
MAIN_WRITELOCK
MAIN_f4vcgfs5t3ftb642.seg
_MAIN_3.toc
$ ../venv/bin/python - <<'PY'      # run from src/
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from documents import index
from whoosh.qparser import QueryParser
ix = index.open_index()
with ix.searcher() as s:
    for term in ("blitzy","ingestion","observation"):
        h = s.search(QueryParser("content", ix.schema).parse(term))
        print(f"search content:'{term}' -> {len(h)} hit(s): {[(x['id'], x['title']) for x in h]}")
    print("index doc_count:", ix.doc_count())
PY
```
```text
search content:'blitzy' -> 1 hit(s): [(1, 'obs_demo')]
search content:'ingestion' -> 1 hit(s): [(1, 'obs_demo')]
search content:'observation' -> 1 hit(s): [(1, 'obs_demo')]
index doc_count: 2
```

This is the runtime proof that the `add_to_index` handler ran (Q2): each word rendered into the test image — `blitzy`, `ingestion`, `observation` — retrieves exactly the traced document's index entry (`id=1`, title `obs_demo`), so its OCR‑extracted `content` was written to the Whoosh index. `index doc_count: 2` reflects that the REST‑upload document (id=2) exercised in Q1 is indexed as well. `add_to_index` calls `index.add_or_update_document(document)` [`src/documents/signals/handlers.py:L428-L431`], which writes through the Whoosh `AsyncWriter` [`src/documents/index.py:L66`] in `add_or_update_document` [`src/documents/index.py:L118`].

### Observed: the field list confirms there is no status column

```console
$ ../venv/bin/python - <<'PY'      # run from src/
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
# src/documents/consumer.py:L102-L113 (verbatim source)
def pre_check_duplicate(self):
    with open(self.path, "rb") as f:
        checksum = hashlib.md5(f.read()).hexdigest()                 # L104
    if Document.objects.filter(
        Q(checksum=checksum) | Q(archive_checksum=checksum),         # L106
    ).exists():
        if settings.CONSUMER_DELETE_DUPLICATES:                      # L108-L109
            os.unlink(self.path)
        self._fail(                                                  # L110
            MESSAGE_DOCUMENT_ALREADY_EXISTS,                         # L111
            f"Not consuming {self.filename}: It is a duplicate.",    # L112
        )
```

with `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` [`src/documents/consumer.py:L37`]. It is invoked early in `try_consume_file()` at [`src/documents/consumer.py:L213`], and `_fail()` broadcasts `FAILED`/`100` via `_send_progress(100, 100, "FAILED", message)` [`src/documents/consumer.py:L79`] and then `raise ConsumerError(...)` [`src/documents/consumer.py:L81`].

### Observed: re‑ingesting the identical file (same bytes)

```console
$ md5sum /tmp/blitzy_obs/testinput/obs_demo.png     # identical bytes to the first ingest
f08328e0bcec1b5c0bbf70dc8b9d3d96  /tmp/blitzy_obs/testinput/obs_demo.png
$ cp /tmp/blitzy_obs/testinput/obs_demo.png /tmp/blitzy_obs/consume/        # re-drop the identical file
$ grep "task queue\|duplicate" /tmp/blitzy_obs/data/log/paperless.log | tail -2
```
```text
[2026-07-01 05:54:58,562] [INFO] [paperless.management.consumer] Adding /tmp/blitzy_obs/consume/obs_demo.png to the task queue.
[2026-07-01 05:54:58,697] [ERROR] [paperless.consumer] Not consuming obs_demo.png: It is a duplicate.
```

The `[ERROR] [paperless.consumer] Not consuming obs_demo.png: It is a duplicate.` line is the verbatim rendering of the message built at [`src/documents/consumer.py:L112`]. The file was re‑detected and enqueued (the watcher itself does not deduplicate), but consumption was rejected before any second document was created.

### Observed: the WebSocket `FAILED` payload

The two subscribers from Q3 (the authenticated end‑to‑end client `ws_client.py` and the channel‑layer tap `ws_tap.py`) were still connected, so they also captured the duplicate task's frames. That task's WebSocket `task_id` is `f3c5232e-c75f-440a-aa4c-deef27d91474` — a fresh UUID minted by the consumer at [`src/documents/consumer.py:L200`] (`self.task_id = task_id or str(uuid.uuid4())`), because the watcher enqueues without a `task_id` [`src/documents/management/commands/document_consumer.py:L86-L91`]. Extracting exactly that task's frames from each subscriber's saved output:

**Method 1 — authenticated end‑to‑end client** (plain inner `data` payloads, one per frame):

```console
$ grep "f3c5232e-c75f-440a-aa4c-deef27d91474" /tmp/blitzy_obs/run/ws_client.out
```
```text
{"filename": "obs_demo.png", "task_id": "f3c5232e-c75f-440a-aa4c-deef27d91474", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
{"filename": "obs_demo.png", "task_id": "f3c5232e-c75f-440a-aa4c-deef27d91474", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
```

**Method 2 — channel‑layer tap** (full `status_update` envelopes with a local `[TAP hh:mm:ss]` receipt‑time prefix added by the tap script):

```console
$ grep "f3c5232e-c75f-440a-aa4c-deef27d91474" /tmp/blitzy_obs/run/ws_tap.out
```
```text
[TAP 05:54:58] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "f3c5232e-c75f-440a-aa4c-deef27d91474", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
[TAP 05:54:58] {"type": "status_update", "data": {"filename": "obs_demo.png", "task_id": "f3c5232e-c75f-440a-aa4c-deef27d91474", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}}
```

Only **two** frames appear (not the six of a successful run): the initial `STARTING`/`0` frame, sent at [`src/documents/consumer.py:L202`] *before* the duplicate check runs at [`src/documents/consumer.py:L213`], followed immediately by the terminal `FAILED`/`100` frame. The exact literals `"status": "FAILED"` and `"message": "document_already_exists"` match `_fail()` → `_send_progress(100, 100, "FAILED", message)` [`src/documents/consumer.py:L79`], where `message` is `MESSAGE_DOCUMENT_ALREADY_EXISTS` (defined at [`src/documents/consumer.py:L37`], passed at [`src/documents/consumer.py:L111`]).

### Corroboration: the django‑q Failure record (clean traceback)

The task's persisted record flips to `success=False`, and its stored traceback pins the exact call chain (this is the authoritative, un‑garbled traceback):

```console
$ ../venv/bin/python - <<'PY'      # run from src/
import os, django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup()
from django_q.models import Task
t=[x for x in Task.objects.all() if not x.success][-1]
print('django-q Task.id:', t.id)
print('name=', t.name, 'func=', t.func, 'success=', t.success)
print('args=', t.args)
print(t.result)
PY
```
```text
django-q Task.id: 5c9fce9f2bb944b39c663d220ac02be4
name= obs_demo.png func= documents.tasks.consume_file success= False
args= ('/tmp/blitzy_obs/consume/obs_demo.png',)
obs_demo.png: Not consuming obs_demo.png: It is a duplicate. : Traceback (most recent call last):
  File "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/venv/lib/python3.10/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/src/documents/consumer.py", line 213, in try_consume_file
    self.pre_check_duplicate()
  File "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/src/documents/consumer.py", line 110, in pre_check_duplicate
    self._fail(
  File "/tmp/blitzy/paperless-ngx/blitzy-6ca62ab3-d3ba-4223-8376-103b21659227_57f7e5/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: obs_demo.png: Not consuming obs_demo.png: It is a duplicate.
```

This confirms, at runtime, the precise call chain: `django_q/cluster.py:432` → `tasks.py:236` (`consume_file`) → `consumer.py:213` (`pre_check_duplicate()`) → `consumer.py:110` (`self._fail(`) → `consumer.py:81` (`raise ConsumerError`). Note that this django‑q record is keyed by django‑q's own internal task id (`5c9fce9f2bb944b39c663d220ac02be4`), which is **distinct** from the WebSocket `task_id` (`f3c5232e-c75f-440a-aa4c-deef27d91474`) shown above: the WebSocket id is minted inside the consumer at [`src/documents/consumer.py:L200`], whereas django‑q assigns its own id when the task is enqueued. The two records are tied to the same event by matching `name`/`func`/`args` and the identical `ConsumerError` message, not by a shared identifier.

### Corroboration: the database `unique=True` backstop

Even if the application check were bypassed, the schema forbids a duplicate checksum. Attempting to insert a second row with an existing `checksum` raises an `IntegrityError`:

```console
$ ../venv/bin/python - <<'PY'      # run from src/
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

This is the runtime manifestation of the `checksum` field's declaration — a `CharField(max_length=32, editable=False, unique=True)` [`src/documents/models.py:L135-L141`]. (A periodic, checksum‑based integrity pass also exists in `src/documents/sanity_checker.py` as a supplementary safeguard, cited from source rather than exercised here.)

**Reasoning.** "Already processed" is tracked by content identity — the MD5 checksum of the file's bytes — not by filename or path. Deduplication is enforced in **two independent layers**: an early application‑level query in `pre_check_duplicate()` that fails the task cleanly with `document_already_exists` (so a re‑dropped file produces a `FAILED` WebSocket frame and a django‑q failure, but **no** second `Document`), and a database `UNIQUE` constraint that guarantees the invariant even if the application check is ever skipped. Matching against *both* `checksum` and `archive_checksum` [`src/documents/consumer.py:L106`] means a file identical to either a stored original or a stored archive is caught.

---

## Coverage

Re‑reading the five sub‑questions, each is explicitly answered above with verbatim runtime output and `file:line` citations:

| Sub‑question | Answered in | Signature runtime evidence |
|---|---|---|
| **Q1** Detection & handoff | Q1 | Watcher `"Adding … to the task queue."`; `qcluster` `processing […]`; REST `"OK"`/HTTP `200`; django‑q `Task.func = documents.tasks.consume_file` |
| **Q2** Transitions → parsing / classification / indexing | Q2 | `paperless.consumer` log sequence (Consuming → Detected mime type → Parser → Parsing → Generating thumbnail → Saving record → consumption finished); `load_classifier()` logged **pre‑store** (`05:52:01,898` < `05:52:01,901`); `document_consumption_finished` sent **post‑store** → 6 wired handlers; signal proven at runtime by the Whoosh index search hit (`id=1`), not by the classifier line |
| **Q3** Progress & completion | Q3 | Six WebSocket frames `STARTING 0 new_file` → `WORKING 20/70/90/95` → `SUCCESS 100 finished document_id 1`; unauthenticated connect → HTTP `403` |
| **Q4** Final state & storage | Q4 | Persisted `Document` row (id 1); `originals/archive/thumbnails` files; Whoosh search hit; **KEY FINDING: no status column** (field‑list dump) |
| **Q5** Duplicate avoidance | Q5 | `"Not consuming …: It is a duplicate."`; WebSocket `FAILED`/`document_already_exists`; django‑q `success=False` traceback; DB `UNIQUE constraint failed: documents_document.checksum` |

**Method note.** Per the investigation rule, every answer above was written from output captured while the system was running (the pinned Python 3.10.20 venv over Redis + django‑q + Channels + gunicorn/ASGI + SQLite); source `file:line` references are used only to locate the origin of each observed value. Two claims are cited from source rather than executed and are flagged as such: the **email** ingestion entry point — `async_task("documents.tasks.consume_file", path=temp_filename, …)` [`src/paperless_mail/mail.py:L336-L338`] (no mail server was configured) and the periodic `sanity_checker` integrity pass. All temporary observation scripts, test inputs, and the throwaway data directory were removed after capture, leaving the repository unchanged except for this document.
