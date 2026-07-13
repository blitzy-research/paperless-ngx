# How OCR Behaves at Runtime in paperless-ngx — A Runtime Investigation

This document answers, from **directly observed runtime behavior**, how OCR works inside
paperless-ngx across four scenarios:

- **Q1** — Upload an image with **no embedded text**: how to see that OCR has started, what the
  document's processing state looks like while it runs, and how the background workers behave.
- **Q2** — Upload a similar image that **already contains text**: does the system skip OCR, or
  still touch the OCR pipeline, and how can you tell the difference after processing finishes?
- **Q3** — **Compare the final API responses** for both cases: which fields carry OCR-generated
  text versus pre-existing text.
- **Q4** — **Weak or incomplete OCR**: what happens to the document's final state, does it still
  count as fully processed, and how is that reflected in the saved metadata?

**Method.** Every behavioral claim below was produced by building and running paperless-ngx in its
canonical, default configuration, driving real uploads through the canonical REST entry point
`POST /api/documents/post_document/`, and capturing the raw runtime signals (WebSocket
`status_updates` frames, the `paperless.consumer` / `paperless.parsing.tesseract` logger output, the
`GET /api/documents/{id}/` JSON, the OCR sidecar bytes, and the django-q task rows). Each raw block
below is preceded by the **exact command that produced it**. Claims are labelled **[observed]** when
they come from captured runtime output and **[inferred]** when they are read from the source alone.

- **Runtime commit:** `542221a38dff06361e07976452f9aea24d210542` (verified below).
- **All `path:line` citations are repo-relative and correspond to this commit.**
- **Read-only:** no source file was modified. All observation scripts and sample uploads were
  temporary and were removed at teardown (see *Cleanup & Security Teardown*).

---

## TL;DR (each answer is proven in its section below)

- **Q1 [observed].** OCR start is visible as the `paperless.parsing.tesseract` log line
  `Calling OCRmyPDF with args:` (its full argument dict is shown verbatim under Q1) and as the WebSocket transition to
  `status="WORKING", message="parsing_document", current_progress=20`. While OCR runs, the status
  **holds** at `WORKING/parsing_document/20` — there is **no intra-OCR progress stream** — then jumps
  to `generating_thumbnail/70`. The work is executed by a **django-q `qcluster`** worker running
  `documents.tasks.consume_file`.
- **Q2 [observed].** An **image always runs OCR** (paperless hard-codes "no pre-existing text" for
  images). A **text-bearing PDF still touches OCRmyPDF**, but in the default `skip` mode OCRmyPDF
  copies the text pages verbatim and writes a sidecar marker `[OCR skipped on page(s) …]`; paperless
  detects that marker, **discards** the sidecar, and reads the text with pdfminer instead. After
  finishing you tell them apart by the log trail and the sidecar bytes (shown exactly, below).
- **Q3 [observed].** `GET /api/documents/{id}/` returns **exactly 12 fields**. The **same `content`
  field** carries the text in both cases (there is **no** provenance field distinguishing
  OCR-generated from pre-existing text), and `archived_file_name` is non-null for both because both
  produced an archive PDF.
- **Q4 [observed].** Weak/empty OCR still ends in **`SUCCESS`** — the document is stored with
  `content=""` and an archive PDF is still produced. A document only reaches **`FAILED`** through a
  different mechanism (a duplicate-checksum `IntegrityError`, shown with its full traceback).

---

## Environment & Reproduction

### The runtime that was observed

The stack is two Docker containers on a user-defined bridge network: the paperless application
container (which runs the ASGI web/WebSocket server **and** the django-q worker), and a redis
container (django-q broker + Channels layer).

**Command — containers, image digest, network, published ports [observed]:**

```
$ docker ps
CONTAINER ID   IMAGE                                                                                                            COMMAND                  CREATED             STATUS             PORTS                    NAMES
52c2d21db705   ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01   "bash -c 'sleep infi…"   About an hour ago   Up About an hour   0.0.0.0:8000->8000/tcp   paperless-app
1a20985e0446   redis:7-alpine                                                                                                   "docker-entrypoint.s…"   About an hour ago   Up About an hour   6379/tcp                 paperless-redis

$ docker inspect --format '{{index .RepoDigests 0}}' paperless-app
ghcr.io/scaleapi/swe-atlas@sha256:d4abe56dd5d1cb2632353baf06e9d80a704f147a70ac2ec15c2b78fc5fddfe15
$ docker inspect --format 'ImageID={{.Image}}' paperless-app
ImageID=sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f

$ docker network inspect paperless-net --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
paperless-redis 172.18.0.2/16
paperless-app 172.18.0.3/16
```

Redis version [observed]:

```
$ docker exec paperless-redis redis-server --version
Redis server v=7.4.9 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=b4acab9aea09546d
```

**Runtime commit matches the deliverable's citation commit [observed].** `docker exec` runs as
`root` (the image's default user), while `/app` is owned by `testuser`, so git first needs `/app`
marked as a safe directory; otherwise it aborts with `fatal: detected dubious ownership in
repository at '/app'` (exit 128). This one-time global setting also satisfies the
`git -C /app status --porcelain` check in *Cleanup & Security Teardown*:

```
$ docker exec paperless-app git config --global --add safe.directory /app
$ docker exec paperless-app git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
```

### Exact build/run recipe

The application container's PID 1 is `sleep infinity`; there is **no supervisord** in this image, so
the web server and the worker are started manually inside the container. The following commands
reproduce the stack that was observed running (the running result is verified by the `docker ps`,
`/proc`, config, and gunicorn-log captures in this section):

```
# 0. The exact image (tag as captured by `docker ps` / `docker inspect` above)
IMAGE=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01

# 1. Network + broker
docker network create paperless-net
docker run -d --name paperless-redis --network paperless-net redis:7-alpine

# 2. Application container, port published, on the same network.
#    PAPERLESS_ADMIN_PASSWORD is a secret and is intentionally not printed here; supply your own.
#    NOTE: the image ENTRYPOINT is ["/bin/bash"], so `--entrypoint bash` is REQUIRED. Without it the
#    effective command becomes `/bin/bash bash -c "sleep infinity"` and the container immediately
#    exits 126 ("/bin/bash: /bin/bash: cannot execute binary file"). With `--entrypoint bash` the
#    running COMMAND is `bash -c 'sleep infi…'`, exactly as shown by the `docker ps` capture above.
docker run -d --name paperless-app --network paperless-net -p 8000:8000 \
  --entrypoint bash \
  -e PAPERLESS_REDIS=redis://paperless-redis:6379 \
  -e PAPERLESS_ADMIN_USER=admin \
  -e PAPERLESS_ADMIN_PASSWORD="$PAPERLESS_ADMIN_PASSWORD" \
  -e PAPERLESS_ADMIN_MAIL=admin@example.com \
  -e PAPERLESS_TIME_ZONE=UTC \
  "$IMAGE" -c "sleep infinity"

# 2b. Runtime prerequisites the raw image lacks — run as root inside the container (docker exec's
#     default user is root).
#     * libzbar0 is REQUIRED: `src/documents/tasks.py:25` does `from pyzbar import pyzbar` at module
#       load, and the qcluster worker imports `documents.tasks`; without libzbar0 that import fails
#       with "ImportError: Unable to find zbar shared library", so `consume_file` can never run.
#       poppler-utils and pngquant back the PDF/image consume path.
#     * The bundled ImageMagick policy replaces the distro default, whose
#       `<policy domain="coder" rights="none" pattern="PDF" />` blocks the PDF coder that the
#       generating_thumbnail (progress 70) and archive stages need; the bundled file grants
#       `rights="read|write" pattern="PDF"`.
docker exec paperless-app bash -c 'apt-get update && apt-get install -y libzbar0 poppler-utils pngquant'
docker exec paperless-app bash -c 'cp /app/docker/imagemagick-policy.xml /etc/ImageMagick-6/policy.xml'

# 3. Prepare data dirs, apply migrations, create the superuser (run inside the container).
#    The data/media/consume dirs must exist first, or Django's system check aborts migrate with
#    "PAPERLESS_CONSUMPTION_DIR is set but doesn't exist." / "PAPERLESS_MEDIA_ROOT is set but doesn't exist."
docker exec paperless-app bash -c 'mkdir -p /app/data /app/media /app/consume /app/data/index'
docker exec paperless-app bash -c 'cd /app/src && python3 manage.py migrate'
docker exec paperless-app bash -c 'cd /app/src && python3 manage.py manage_superuser'

# 4. Start the ASGI web/WebSocket server and the django-q worker (both inside the container).
#    gunicorn's stdout/stderr is redirected to /tmp/gunicorn.log so the "Listening at" / "Using
#    worker" lines can be read back with the verification command shown below.
docker exec -d paperless-app bash -c 'cd /app/src && gunicorn -c /app/gunicorn.conf.py paperless.asgi:application > /tmp/gunicorn.log 2>&1'
docker exec -d paperless-app bash -c 'cd /app/src && python3 manage.py qcluster'
```

> **Note on `PAPERLESS_ADMIN_PASSWORD`.** The literal password is intentionally **redacted** here
> (it is a secret, not an observation). For this investigation the seeded `admin` password was
> **rotated to an ephemeral random value** before any upload, kept only in a mode-`0600` file, and
> destroyed at teardown. See *Authentication* and *Cleanup & Security Teardown*. This redaction of a
> secret is distinct from evidence: every observable value elsewhere in this document is shown in
> full.

Migrations and superuser state at the time of observation [observed]:

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py showmigrations | grep -c "\[X\]"'
92
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py showmigrations | grep -c "\[ \]"'
0
```

gunicorn is listening on `0.0.0.0:8000` with the paperless ASGI worker [observed]:

```
$ docker exec paperless-app grep -E "Listening at|Using worker" /tmp/gunicorn.log
[2026-07-13 16:25:06 +0000] [535] [INFO] Listening at: http://0.0.0.0:8000 (535)
[2026-07-13 16:25:06 +0000] [535] [INFO] Using worker: paperless.workers.ConfigurableWorker
```

### Canonical, default configuration

Configuration was read from the running process with `manage.py shell`. The producing command is
shown; the OCR mode is the canonical default `skip` (`src/paperless/settings.py:522`), and the full
`Q_CLUSTER` dict (including `catch_up: false`) is `src/paperless/settings.py:449`.

**Command [observed]:**

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py shell < /tmp/dump_config.py'
OCR_MODE= skip
DEBUG= False
channels_in_INSTALLED_APPS= False
django_q_in_INSTALLED_APPS= True
SESSION_ENGINE= django.contrib.sessions.backends.db
CONSUMER_ENABLE_BARCODES= False
OCR_LANGUAGE= eng
OCR_OUTPUT_TYPE= pdfa
OCR_CLEAN= clean
OCR_DESKEW= True
OCR_ROTATE_PAGES= True
THREADS_PER_WORKER= 11
CHANNEL_LAYERS_BACKEND= channels_redis.core.RedisChannelLayer
Q_CLUSTER= {"catch_up": false, "name": "paperless", "recycle": 1, "redis": "redis://paperless-redis:6379", "retry": 1810, "timeout": 1800, "workers": 11}
```

- `OCR_MODE=skip` — `src/paperless/settings.py:522` (`PAPERLESS_OCR_MODE` default `"skip"`).
- `Q_CLUSTER` — `src/paperless/settings.py:449` (`name` :450, `catch_up` :451, `recycle` :452,
  `retry` :453, `timeout` :454, `workers` :455, `redis` :456).
- `django_q` is the installed worker app — `src/paperless/settings.py:110`. `channels` is only
  appended to `INSTALLED_APPS` when `DEBUG` is true (`src/paperless/settings.py:113`), which is why
  `channels_in_INSTALLED_APPS=False` here even though the WebSocket works (it is served by the ASGI
  application, `src/paperless/asgi.py:20`).
- `CONSUMER_ENABLE_BARCODES=False`, so the barcode branch in `consume_file` is skipped and control
  goes straight to `Consumer().try_consume_file(...)` (`src/documents/tasks.py:236`) **[observed via
  config + log]**.

### Topology of the background workers

`ps` is **not installed** in this image, so the process list was enumerated from `/proc`. The actual
topology is: PID 1 = `sleep infinity`; a gunicorn master + workers running the ASGI app; and a
django-q `qcluster` sentinel + worker pool. **There is no `document_consumer` process** — uploads in
this investigation used the REST entry point exclusively.

**Command [observed]:**

```
$ docker exec paperless-app bash -c 'for p in $(ls /proc | grep -E "^[0-9]+$"); do cmd=$(tr "\0" " " </proc/$p/cmdline 2>/dev/null); [ -n "$cmd" ] && echo "PID $p: $cmd"; done'
PID 1: sleep infinity
PID 527: python3 manage.py qcluster
PID 535: /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID 544: /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID 547: /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
PID 562: python3 manage.py qcluster
PID 574: python3 manage.py qcluster
PID 575: python3 manage.py qcluster
PID 2417: python3 manage.py qcluster
PID 2614: python3 manage.py qcluster
PID 2811: python3 manage.py qcluster
PID 2874: python3 manage.py qcluster
PID 2946: python3 manage.py qcluster
PID 3030: python3 manage.py qcluster
PID 3072: python3 manage.py qcluster
PID 3084: python3 manage.py qcluster
PID 3085: python3 manage.py qcluster
PID 3386: python3 manage.py qcluster
PID 3678: python3 manage.py qcluster
```

(The `qcluster` worker count fluctuates because `Q_CLUSTER` has `recycle: 1`, so workers are
recycled after each task; the sentinel plus its pool total 15 at this instant, matching the count
below.)

Counts and the document_consumer check — the scan excludes its own PID (`$$`) so it cannot match
itself [observed]:

```
$ docker exec paperless-app bash -c 'self=$$; dc=0;
for p in $(ls /proc | grep -E "^[0-9]+$"); do
  [ "$p" = "$self" ] && continue
  cmd=$(tr "\0" " " </proc/$p/cmdline 2>/dev/null)
  case "$cmd" in *"manage.py document_consumer"*) dc=$((dc+1));; esac
done
[ "$dc" -eq 0 ] && echo "RESULT: no manage.py document_consumer process is running";
g=0; q=0;
for p in $(ls /proc | grep -E "^[0-9]+$"); do
  cmd=$(tr "\0" " " </proc/$p/cmdline 2>/dev/null)
  case "$cmd" in
    *gunicorn*paperless.asgi*) g=$((g+1));;
    *"manage.py qcluster"*) q=$((q+1));;
  esac
done
echo "gunicorn processes (master+workers): $g";
echo "qcluster processes (sentinel+workers): $q"'
RESULT: no manage.py document_consumer process is running
gunicorn processes (master+workers): 3
qcluster processes (sentinel+workers): 15
```

(The three gunicorn processes are the master PID 535 plus its two workers 544 and 547, matching the
`/proc` listing above; the 15 `qcluster` processes are the django-q sentinel plus its recycled
worker pool.)

- The web/WebSocket server is `gunicorn … paperless.asgi:application` — the ASGI app is defined at
  `src/paperless/asgi.py:17` (`ProtocolTypeRouter`) with the WebSocket handler behind
  `AuthMiddlewareStack` at `src/paperless/asgi.py:20`.
- The worker is django-q's `qcluster`. The upload enqueues `documents.tasks.consume_file` via
  `async_task` (`src/documents/views.py:523`), and the `qcluster` pool executes it
  (`src/documents/tasks.py:184`, `:236`).
- The `document_consumer` directory watcher is an **alternate** canonical entry point
  (`src/documents/management/commands/document_consumer.py:86` enqueues the same task); it was **not
  running** here.

### Authentication

The WebSocket at `ws/status/` (`src/paperless/urls.py:137`) is guarded by `AuthMiddlewareStack`
(`src/paperless/asgi.py:20`) and `StatusConsumer.connect`, which raises `DenyConnection` for
unauthenticated clients (`src/paperless/consumers.py:15`). Session auth uses the DB session backend
(`SESSION_ENGINE=django.contrib.sessions.backends.db`, confirmed above).

Both outcomes were exercised. The listener presents the admin `sessionid` cookie (read from a
mode-`0600` file, never from argv); an unauthenticated attempt is rejected with HTTP 403 at the
handshake.

**Command [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/ws_auth_probe.py
[17:51:15.985] ATTEMPT WITH valid admin sessionid cookie:
[17:51:16.019]   -> CONNECTED OK (handshake accepted, authenticated)
[17:51:16.020] ATTEMPT WITHOUT any session cookie:
[17:51:16.024]   -> WebSocketBadStatusException: Handshake status 403 Forbidden -+-+- {'date': 'Mon, 13 Jul 2026 17:51:16 GMT', 'server': 'Python/3.9 websockets/10.3', 'content-length': '0', 'content-type': 'text/plain', 'connection': 'close'} -+-+- b''
```

REST uploads authenticate with a DRF **token** (`Authorization: Token …`), also read from the
mode-`0600` file. The token and session were created during provisioning and revoked at teardown.

### Observation probes (temporary; source shown for auditability; removed afterward)

All probes lived **off-repo** under `/tmp/blitzy_probes/` (mode `700`) and were deleted at teardown
(*Cleanup & Security Teardown*). Their security-relevant logic is reproduced here so the method is
reviewable even though the files no longer exist:

- **`upload.py`** — drives the canonical `POST /api/documents/post_document/`. The DRF token is read
  from `/tmp/blitzy_probes/auth.env` (mode `0600`) **inside the script**, never passed on the command
  line:

  ```python
  AUTH = "/tmp/blitzy_probes/auth.env"
  URL  = "http://localhost:8000/api/documents/post_document/"
  def load(key):
      for line in open(AUTH):
          if line.startswith(key + "="):
              return line.strip().split("=", 1)[1]
      raise SystemExit(f"missing {key}")
  tok = load("TOKEN")
  r = requests.post(URL, headers={"Authorization": f"Token {tok}"},
                    files={"document": (name, fh)}, data={"title": title}, timeout=60)
  ```

- **`ws_listen.py`** — connects to `ws/status/` with the admin `sessionid` cookie (also read from the
  `0600` file) and prints a `LISTENER_START` line with a timestamp **before** any upload, then every frame
  verbatim with its receive timestamp. This is how each Q-section proves the listener was attached
  *before* the upload.

- **`sidecar_watch.py`** — runs inside the container and polls `/tmp/paperless` at 20 ms, copying the
  **exact** per-task `sidecar*.txt` / `archive*.pdf` by full path (recording source path, inode,
  size, and sha256) before the parser deletes its temp directory. It uses the concrete per-task path,
  **not** a broad glob, so there is no read-time race.

- **`api_get.py`** (full `GET /api/documents/{id}/` JSON + a field count), **`qtask.py`** (dumps
  `django_q.models.Task` rows), **`docs_admin.py`** (list/delete), **`dump_config.py`**,
  **`provision_auth.py`** (password rotation + token/session creation), **`run_scenario.sh`** (a
  single-invocation orchestrator that marks the log length, starts the sidecar watcher + WS listener,
  **waits for `LISTENER_START` before uploading**, then extracts only the new log lines and the
  sidecar captures).

### Where each runtime signal was captured

| Signal | Source | Cited definition |
|---|---|---|
| Task queued / running / finished + progress | WebSocket `status_updates` frames | `Consumer._send_progress` `src/documents/consumer.py:56` |
| OCR start + arguments | `paperless.parsing.tesseract` log | `src/paperless_tesseract/parsers.py:260` |
| Which worker ran it | django-q `Task` row | `src/documents/tasks.py:184`, `:236` |
| Final stored text / archive | `GET /api/documents/{id}/` JSON | `DocumentSerializer` `src/documents/serialisers.py:219` |
| OCR text bytes | per-task `sidecar.txt` | `src/paperless_tesseract/parsers.py:99` |

---

## The observed pipeline at a glance

```
POST /api/documents/post_document/           src/documents/views.py:497  (PostDocumentView.post)
   └─ async_task("documents.tasks.consume_file", …)   views.py:523   task_id = str(uuid.uuid4())  views.py:521
        └─ django-q qcluster worker           tasks.py:184 consume_file → tasks.py:236 Consumer().try_consume_file
             ├─ STARTING / new_file / 0                        consumer.py:202   ─┐
             ├─ WORKING / parsing_document / 20                consumer.py:259    │  WebSocket
             │     └─ RasterisedDocumentParser.parse()         parsers.py:230     │  status_updates
             │          └─ "Calling OCRmyPDF with args:"       parsers.py:260     │  frames via
             ├─ WORKING / generating_thumbnail / 70            consumer.py:264    │  _send_progress
             ├─ WORKING / parse_date / 90                      consumer.py:274    │  consumer.py:56
             ├─ WORKING / save_document / 95                   consumer.py:294    │
             │     └─ _store → Document.objects.create         consumer.py:301,398│
             └─ SUCCESS / finished / 100 (+document_id)        consumer.py:375   ─┘
```

The five `WORKING`/`SUCCESS` messages come from the module constants
`src/documents/consumer.py:43-49` (`MESSAGE_NEW_FILE`, `PARSING_DOCUMENT`, `GENERATING_THUMBNAIL`,
`PARSE_DATE`, `SAVE_DOCUMENT`, `FINISHED`). The frame payload keys are set in
`Consumer._send_progress` (`src/documents/consumer.py:56`, payload built at ~`:64-72`):
`filename, task_id, current_progress, max_progress, status, message, document_id`, and are relayed to
the browser verbatim by `StatusConsumer.status_update` → `self.send(json.dumps(event["data"]))`
(`src/paperless/consumers.py:29`, `:33`).

---

## Q1 — Upload an image with no embedded text (the pure-OCR path)

**Input:** `no-text-alpha.png` — an image with no text layer (a copy of the repository fixture
`src/paperless_tesseract/tests/samples/no-text-alpha.png`, sha256
`a0ea205979f49e4ddfb98462be55cb7c175181b90fb7da1c8a3d8614eabaf21b`, 32595 bytes). A **copy** was
uploaded so the repository fixture is never touched (see *Cleanup*). The scenario was run **twice**
with the identical bytes (deleting the resulting document between runs so the second upload is not
rejected as a duplicate): **Run 1 → document 13**, **Run 2 → document 14**.

### Command

```
# listener attaches first (prints LISTENER_START), THEN the upload runs:
$ python3 /tmp/blitzy_probes/scripts/ws_listen.py 25 &
$ python3 /tmp/blitzy_probes/scripts/upload.py /tmp/blitzy_probes/samples/no-text-alpha.png q1_notext_run1
```

### BEFORE — the upload is accepted and the task is enqueued [observed]

The REST call returns immediately with `"OK"`; the work has been handed to the worker, not done
inline. `PostDocumentView.post` returns `Response("OK")` at `src/documents/views.py:535`.

```
[upload 17:52:51.236] POST http://localhost:8000/api/documents/post_document/
file=/tmp/blitzy_probes/samples/no-text-alpha.png title=q1_notext_run1
HTTP 200
BODY: "OK"
```

### The processing state — the ordered `status_updates` frames (Run 1 → doc 13) [observed]

The listener started at `17:52:51.097`, **before** the upload at `17:52:51.236` — so the frames below
were captured from the very start of the task. The `task_id` is the upload UUID from
`src/documents/views.py:521` (`task_id = str(uuid.uuid4())`).

```
LISTENER_START 17:52:51.097 connected=ws://localhost:8000/ws/status/
[17:52:51.415] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[17:52:51.440] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[17:52:53.395] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[17:53:00.975] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[17:53:00.994] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[17:53:01.058] {"filename": "no-text-alpha.png", "task_id": "07882971-6177-451e-aea6-e7e09f382725", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 13}
```

Mapping each frame to code: `STARTING/new_file/0` = `src/documents/consumer.py:202`;
`WORKING/parsing_document/20` = `:259`; `WORKING/generating_thumbnail/70` = `:264`;
`WORKING/parse_date/90` = `:274`; `WORKING/save_document/95` = `:294`; `SUCCESS/finished/100` with
the new `document_id` = `:375`.

### DURING — OCR starts, and the start signal in the logs (Run 1) [observed]

The definitive **start-of-OCR** signal is the `paperless.parsing.tesseract` line
`Calling OCRmyPDF with args:` (`src/paperless_tesseract/parsers.py:260`). The producing command
selects run 1's log window (17:52–17:53; run 2 is at 17:55, so the windows do not overlap):

```
$ docker exec paperless-app bash -c "grep -E '2026-07-13 17:5[23]:' /app/data/log/paperless.log"
[2026-07-13 17:52:51,417] [INFO] [paperless.consumer] Consuming no-text-alpha.png
[2026-07-13 17:52:51,418] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-13 17:52:51,420] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 17:52:51,439] [DEBUG] [paperless.consumer] Parsing no-text-alpha.png...
[2026-07-13 17:52:51,530] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/paperless/paperless-upload-ggmbu6bj: 'dpi'
[2026-07-13 17:52:51,530] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 35 based on image width 297
[2026-07-13 17:52:51,531] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/paperless/paperless-upload-ggmbu6bj for compatibility with img2pdf
[2026-07-13 17:52:51,536] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-ggmbu6bj', 'output_file': '/tmp/paperless/paperless-basx4ci3/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-basx4ci3/sidecar.txt', 'image_dpi': 35}
[2026-07-13 17:52:52,500] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 17:52:52,501] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-13 17:52:52,502] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-ggmbu6bj', 'output_file': '/tmp/paperless/paperless-basx4ci3/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-basx4ci3/sidecar-fallback.txt', 'image_dpi': 35}
[2026-07-13 17:52:53,378] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/paperless/paperless-upload-ggmbu6bj, the content will be empty.
[2026-07-13 17:52:53,378] [DEBUG] [paperless.consumer] Generating thumbnail for no-text-alpha.png...
[2026-07-13 17:53:00,994] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 17:53:01,015] [DEBUG] [paperless.consumer] Deleting file /tmp/paperless/paperless-upload-ggmbu6bj
[2026-07-13 17:53:01,040] [INFO] [paperless.consumer] Document 2026-07-13 q1_notext_run1 consumption finished
```

Key facts from this trace **[observed]**:

- The image is OCR'd with `skip_text: True` (the default `skip` mode maps to `skip_text` at
  `src/paperless_tesseract/parsers.py:158`), and OCRmyPDF runs with `progress_bar: False`
  (`src/paperless_tesseract/parsers.py:152`).
- `image_dpi: 35` here is the **estimated** DPI for *this* image (`Estimated DPI 35 based on image
  width 297`). DPI is per-image, not a constant — see Q2 where the JPG is `image_dpi: 200`.
- The alpha layer is removed from the **temp copy** `/tmp/paperless/paperless-upload-ggmbu6bj`
  (`src/paperless_tesseract/parsers.py:194`), **not** from the uploaded file on disk — this is why
  uploading a copy leaves the repository fixture pristine.
- Because this particular image yields no text, the first pass produces an empty sidecar, which
  raises `NoTextFoundException` (`src/paperless_tesseract/parsers.py:267`) and triggers the
  `force_ocr` safe-fallback second invocation (`:297`). That is a Q4 concern; it is noted here only
  because it is part of the honest, unedited trace.

### The background worker that executed it [observed]

The task ran on a django-q `qcluster` worker. Note that the django-q `Task.id` (32-hex, **no**
dashes) is a **different identifier** from the WebSocket `task_id` (the dashed UUID above): they name
different layers. Cross-layer correlation therefore uses filename + timing + the resulting
`document_id`, not a shared id.

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py shell < /tmp/qtask.py'
id=9a128c9ef2794d23b4544f61546ced03 func=documents.tasks.consume_file name='no-text-alpha.png' success=True started=17:52:51 stopped=17:53:01 result='Success. New document id 13 created'
id=7698b7c6b8d6445e98e598d6a20a81fc func=documents.tasks.consume_file name='no-text-alpha.png' success=True started=17:55:21 stopped=17:55:31 result='Success. New document id 14 created'
```

### CRITICAL observed nuance — intra-OCR progress is NOT streamed [observed]

During OCR the status **holds** at `WORKING/parsing_document/20`; the next frame is
`generating_thumbnail/70`. There is no fractional progress within OCR. Measuring the gap across
**both** runs:

| Transition | Run 1 (doc 13) | Run 2 (doc 14) |
|---|---|---|
| `parsing_document/20` → `generating_thumbnail/70` (the OCR window) | 17:52:51.440 → 17:52:53.395 = **1.955 s** | 17:55:22.019 → 17:55:23.971 = **1.952 s** |
| `generating_thumbnail/70` → `parse_date/90` (thumbnail render) | 17:52:53.395 → 17:53:00.975 = **7.580 s** | 17:55:23.971 → 17:55:31.557 = **7.586 s** |

The OCR window is **stable at ~1.95 s** across two runs, and **no status frame is emitted inside it**
even though *two* OCRmyPDF invocations (the skip-text pass plus the force-OCR fallback) run during
that window. The reason **[observed + code-cited]**: OCRmyPDF is invoked with `progress_bar: False`
(`src/paperless_tesseract/parsers.py:152`) and `RasterisedDocumentParser.parse()` never calls the
`self.progress()` hook (`src/documents/parsers.py:300`), so nothing between `20` and `70` is pushed.
The longest silent gap is actually the thumbnail render (`70`→`90`, ~7.58 s), also frame-silent.

### AFTER — the document is persisted [observed]

`SUCCESS/finished/100` carries the new `document_id` (13 for run 1, 14 for run 2), and the django-q
`Task.result` is `Success. New document id 13 created`. The status transitions to `SUCCESS` at
`src/documents/consumer.py:375`.

### Reproducibility & the second run [observed]

Run 2 (doc 14) produced the identical ordered frame sequence and OCR-window timing (table above):

```
LISTENER_START 17:55:21.650 connected=ws://localhost:8000/ws/status/
[17:55:22.000] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[17:55:22.019] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[17:55:23.971] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[17:55:31.557] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[17:55:31.574] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[17:55:31.635] {"filename": "no-text-alpha.png", "task_id": "cb5737c2-3192-421c-bbb7-00f55f9fe207", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 14}
```

### The same pure-OCR path *producing* text [observed]

To show the identical image path yielding real text (rather than the empty result above), an image
that visibly contains text — `simple.png` (a copy of
`src/paperless_tesseract/tests/samples/simple.png`) — was uploaded through the same entry point
(→ document 15). The log shows `Using text from sidecar file` and **no** "content will be empty"
warning, and the captured sidecar bytes are the OCR output. The sidecar was captured by full path and
its bytes verified with Python (no `xxd`, which is absent from this image):

```
$ docker exec paperless-app python3 -c "from pathlib import Path; b=Path('/tmp/cap_q1_withtext_img/460619994_sidecar.txt').read_bytes(); print(repr(b)); print('hex ', b.hex()); print('len ', len(b))"
b'This is a test document.\n'
hex  546869732069732061207465737420646f63756d656e742e0a
len  25
```

The archive PDF was produced for this document (sidecar capture recorded
`archive.pdf`, 15900 bytes). The stored `content` for this document is `This is a test document.`
(24 chars — the trailing newline is stripped on store), confirming the sidecar is the source of the
document's text on the image path.


---

## Q2 — An image that already contains text vs. a text-bearing PDF

The question — "does an image with text skip OCR, or still touch the pipeline?" — has different
answers for an **image** and for a **PDF**, so both were exercised, each **twice** (deleting the
document between the two identical uploads):

- **Image with visible text** `simple.jpg` → run 1 doc 16 (deleted), **run 2 doc 17 (kept for Q3)**.
- **Single-page text PDF** `simple-digital.pdf` → run 1 doc 18 (deleted), **run 2 doc 19 (kept for
  Q3)**.
- **Three-page text PDF** `multi-page-digital.pdf` → run 1 doc 20, run 2 doc 21 (both deleted).

### An image ALWAYS runs OCR [observed + code-cited]

For any image input, paperless does not even look for a pre-existing text layer: `parse()` sets
`text_original = None` and `original_has_text = False` unconditionally for images
(`src/paperless_tesseract/parsers.py:238-239`); the "already has text" branch is guarded by
`if mime_type == "application/pdf":` (`src/paperless_tesseract/parsers.py:234`), with the has-text
length test (`len(text_original) > 50`) at `:236`. So `simple.jpg` is OCR'd
exactly like the no-text image — only the sidecar comes back non-empty.

**Command + log (run 2, doc 17) [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/upload.py /tmp/blitzy_probes/samples/simple.jpg q2_img_run2
[upload 17:58:45.491] POST http://localhost:8000/api/documents/post_document/
HTTP 200
BODY: "OK"

# new lines from /app/data/log/paperless.log for this task:
[2026-07-13 17:58:45,666] [INFO] [paperless.consumer] Consuming simple.jpg
[2026-07-13 17:58:45,667] [DEBUG] [paperless.consumer] Detected mime type: image/jpeg
[2026-07-13 17:58:45,669] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 17:58:45,777] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-q_xq3vg7', 'output_file': '/tmp/paperless/paperless-vvxs_gfg/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-vvxs_gfg/sidecar.txt', 'image_dpi': 200}
[2026-07-13 17:58:47,116] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 17:58:47,852] [INFO] [paperless.consumer] Document 2026-07-13 q2_img_run2 consumption finished
```

- OCRmyPDF is called with `skip_text: True` and a **real** `image_dpi: 200` for this JPG (contrast
  the no-text PNG's `image_dpi: 35` in Q1 — DPI is genuinely per-image).
- The only text-source line is `Using text from sidecar file`
  (`src/paperless_tesseract/parsers.py:107`). There is **no** pdfminer pre-extraction and **no**
  "Incomplete sidecar" discard — those are PDF-only behaviors (below).

**Sidecar bytes for the image (both runs identical) [observed]:**

```
$ docker exec paperless-app python3 -c "from pathlib import Path; b=Path('/tmp/cap_q2_img_run2/460766558_sidecar.txt').read_bytes(); print(repr(b)); print('hex ', b.hex()); print('len ', len(b))"
b'This is a test document.\n'
hex  546869732069732061207465737420646f63756d656e742e0a
len  25
```

An archive PDF was produced (`archive.pdf`, 21250 bytes → stored as media
`0000017.pdf`). The image path has **no `[OCR skipped …]` marker** because OCR really ran on the
image.

### A text PDF is COPIED, not re-OCR'd — but it still touches OCRmyPDF [observed + code-cited]

For a PDF, paperless first pdfminer-extracts the original to decide whether it already has text
(`original_has_text` when the extract length > 50, `src/paperless_tesseract/parsers.py:236`), then in
the default `skip` mode still calls OCRmyPDF with `skip_text: True`. OCRmyPDF **copies** the
text pages verbatim and writes a sidecar marker instead of OCR text. paperless notices the marker,
**discards** the sidecar (`Incomplete sidecar file: discarding.`,
`src/paperless_tesseract/parsers.py:110`), and reads the text from the archive with pdfminer.

**Command + log, in order (run 2, doc 19) [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/upload.py /tmp/blitzy_probes/samples/simple-digital.pdf q2_pdf_run2
[upload 18:00:10.872] POST http://localhost:8000/api/documents/post_document/
HTTP 200
BODY: "OK"

[2026-07-13 18:00:11,051] [INFO] [paperless.consumer] Consuming simple-digital.pdf
[2026-07-13 18:00:11,052] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 18:00:11,054] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 18:00:11,099] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-upload-cva4n4he
[2026-07-13 18:00:11,170] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-cva4n4he', 'output_file': '/tmp/paperless/paperless-u8ars4g0/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-u8ars4g0/sidecar.txt'}
[2026-07-13 18:00:11,464] [DEBUG] [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
[2026-07-13 18:00:11,469] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /tmp/paperless/paperless-u8ars4g0/archive.pdf
[2026-07-13 18:00:13,206] [INFO] [paperless.consumer] Document 2026-07-13 q2_pdf_run2 consumption finished
```

The four PDF-only discriminators appear in order **[observed]**:

1. `Extracted text from PDF file …/paperless-upload-cva4n4he` — pdfminer **pre-extract of the
   ORIGINAL** upload (used for the has-text decision).
2. `Calling OCRmyPDF …'skip_text': True…` — note there is **no `image_dpi` key** (PDF path, unlike
   the image path which always has `image_dpi`).
3. `Incomplete sidecar file: discarding.` — the sidecar held only the skip marker, so it is thrown
   away (`src/paperless_tesseract/parsers.py:104`, `:110`).
4. `Extracted text from PDF file …/archive.pdf` — pdfminer **fallback on the ARCHIVE** to get the
   text that ends up in `content`.

**Sidecar bytes for the text PDF (both runs identical) — the skip marker itself [observed]:**

```
$ docker exec paperless-app python3 -c "from pathlib import Path; b=Path('/tmp/cap_q2_pdf_run2/460771247_sidecar.txt').read_bytes(); print(repr(b)); print('hex ', b.hex()); print('len ', len(b))"
b'[OCR skipped on page(s) 1]'
hex  5b4f435220736b6970706564206f6e207061676528732920315d
len  26
```

The three-page PDF's marker enumerates the whole page range (both runs identical) [observed]:

```
$ docker exec paperless-app python3 -c "from pathlib import Path; b=Path('/tmp/cap_q2_multi_run2/460771541_sidecar.txt').read_bytes(); print(repr(b)); print('hex ', b.hex()); print('len ', len(b))"
b'[OCR skipped on page(s) 1-3]'
hex  5b4f435220736b6970706564206f6e207061676528732920312d335d
len  28
```

### How `extract_text()` turns the marker into a decision [observed + code-cited]

`extract_text()` (`src/paperless_tesseract/parsers.py:99`) reads the sidecar and, if it contains the
substring `[OCR skipped on page` (`:104`), treats it as incomplete, logs `Incomplete sidecar file:
discarding.` (`:110`), and falls back to pdfminer on the archive PDF. Otherwise it logs `Using text
from sidecar file` (`:107`) and keeps the sidecar text. This is exactly what the two byte captures
above show: the **image** sidecar has no marker (kept), the **PDF** sidecar is only the marker
(discarded).

### Post-finish differentiators — summary [observed]

| After finishing | Image with text (`simple.jpg`, doc 17) | Text PDF (`simple-digital.pdf`, doc 19) |
|---|---|---|
| Detected mime type | `image/jpeg` | `application/pdf` |
| pdfminer pre-extract of original? | no | **yes** (`Extracted text from PDF file …upload-…`) |
| OCRmyPDF called? | yes (`skip_text: True`, `image_dpi: 200`) | yes (`skip_text: True`, **no** `image_dpi`) |
| Sidecar contents (exact) | `b'This is a test document.\n'` (25 B) | `b'[OCR skipped on page(s) 1]'` (26 B) |
| Sidecar disposition | `Using text from sidecar file` (kept) | `Incomplete sidecar file: discarding.` (discarded) |
| Text source for `content` | OCR sidecar | pdfminer on the archive PDF |
| Archive PDF produced? | yes (`0000017.pdf`) | yes (`0000019.pdf`) |

**Answer to Q2 [observed].** An **image is never treated as already having text**, so it always runs
OCR (only the sidecar shows whether text was found). A **text PDF still touches OCRmyPDF**, but in
`skip` mode OCRmyPDF copies the existing text pages and emits the `[OCR skipped on page(s) …]`
marker; paperless discards that sidecar and reuses the original text via pdfminer. After processing
you distinguish them by (a) the mime type, (b) the presence of a pdfminer pre-extract line, (c) the
`image_dpi` key in the OCRmyPDF args, and most decisively (d) the sidecar bytes — OCR text vs. the
skip marker.


---

## Q3 — Compare the final API responses for both cases

The two documents kept from Q2 are compared here: **doc 17** (the OCR'd image `simple.jpg`) and
**doc 19** (the text PDF `simple-digital.pdf`). The endpoint is `GET /api/documents/{id}/`, shaped by
`DocumentSerializer` (`src/documents/serialisers.py:219`).

### The response has exactly 12 fields [observed]

`DocumentSerializer.Meta.fields` lists **exactly 12 fields**
(`src/documents/serialisers.py:222-235`): `id, correspondent, document_type, title, content, tags,
created, modified, added, archive_serial_number, original_file_name, archived_file_name`. There is
**no `storage_path`** and **no `created_date`** field on this endpoint at this commit. The probe
prints the response plus a field count/name list to prove the exact set.

**Command — doc 17 (OCR'd image) [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/api_get.py 17
GET http://localhost:8000/api/documents/17/ -> HTTP 200
{
  "added": "2026-07-13T17:58:47.807484Z",
  "archive_serial_number": null,
  "archived_file_name": "2026-07-13 q2_img_run2.pdf",
  "content": "This is a test document.",
  "correspondent": null,
  "created": "2026-07-13T17:58:45Z",
  "document_type": null,
  "id": 17,
  "modified": "2026-07-13T17:58:47.827309Z",
  "original_file_name": "2026-07-13 q2_img_run2.jpg",
  "tags": [],
  "title": "q2_img_run2"
}
FIELD_COUNT 12
FIELD_NAMES ['added', 'archive_serial_number', 'archived_file_name', 'content', 'correspondent', 'created', 'document_type', 'id', 'modified', 'original_file_name', 'tags', 'title']
```

**Command — doc 19 (text PDF) [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/api_get.py 19
GET http://localhost:8000/api/documents/19/ -> HTTP 200
{
  "added": "2026-07-13T18:00:13.159727Z",
  "archive_serial_number": null,
  "archived_file_name": "2026-07-13 q2_pdf_run2.pdf",
  "content": "This is a test document.",
  "correspondent": null,
  "created": "2026-07-13T18:00:10Z",
  "document_type": null,
  "id": 19,
  "modified": "2026-07-13T18:00:13.180320Z",
  "original_file_name": "2026-07-13 q2_pdf_run2.pdf",
  "tags": [],
  "title": "q2_pdf_run2"
}
FIELD_COUNT 12
FIELD_NAMES ['added', 'archive_serial_number', 'archived_file_name', 'content', 'correspondent', 'created', 'document_type', 'id', 'modified', 'original_file_name', 'tags', 'title']
```

### Field-by-field comparison [observed]

Both responses have the **identical 12-field schema**. The values split as follows:

| Field | doc 17 (OCR'd image) | doc 19 (text PDF) | Same value? |
|---|---|---|---|
| `id` | `17` | `19` | **differ** |
| `title` | `"q2_img_run2"` | `"q2_pdf_run2"` | **differ** |
| `content` | `"This is a test document."` | `"This is a test document."` | **SAME** |
| `correspondent` | `null` | `null` | same |
| `document_type` | `null` | `null` | same |
| `tags` | `[]` | `[]` | same |
| `archive_serial_number` | `null` | `null` | same |
| `created` | `2026-07-13T17:58:45Z` | `2026-07-13T18:00:10Z` | **differ** |
| `modified` | `2026-07-13T17:58:47.827309Z` | `2026-07-13T18:00:13.180320Z` | **differ** |
| `added` | `2026-07-13T17:58:47.807484Z` | `2026-07-13T18:00:13.159727Z` | **differ** |
| `original_file_name` | `"2026-07-13 q2_img_run2.jpg"` | `"2026-07-13 q2_pdf_run2.pdf"` | **differ** |
| `archived_file_name` | `"2026-07-13 q2_img_run2.pdf"` | `"2026-07-13 q2_pdf_run2.pdf"` | **differ** |

Which fields carry the text, and how OCR-generated vs. pre-existing text is reflected **[observed +
code-cited]**:

- **`content` is the sole text carrier for BOTH cases, and its value is identical here** because both
  fixtures contain the same phrase. This is the key finding: the API exposes **no provenance field**
  — nothing in the response says whether the text was produced by OCR (doc 17, from the OCR sidecar)
  or read from an existing text layer (doc 19, via pdfminer). `content` is
  `src/documents/serialisers.py:227`, backed by `Document.content` (`src/documents/models.py:117`).
  The difference in *where the text came from* is visible only in the **logs/sidecar** (Q2), never in
  the API payload.
- **`archived_file_name` is non-null for both** because both produced an archive PDF.
  `get_archived_file_name` returns `None` unless `has_archive_version` is true, otherwise the public
  archive filename (`src/documents/serialisers.py:213`); `has_archive_version` is true when
  `archive_filename` is set (`src/documents/models.py:238`). `original_file_name` comes from
  `get_original_file_name` → `get_public_filename()` (`src/documents/serialisers.py:210`).
- The remaining differences (`id`, `title`, the three timestamps, and the two filenames) simply
  reflect that these are two distinct documents ingested at different times from differently-named
  inputs.

**Answer to Q3 [observed].** Both cases return the same 12-field shape; the OCR-generated text and
the pre-existing text both land in the **same `content` field**, and `archived_file_name` is present
for both. There is **no field that distinguishes** OCR-generated text from pre-existing text — that
distinction exists only in the processing logs and the sidecar, not in the API response.


---

## Q4 — Weak or incomplete OCR results

**Input:** `blank_white.png` — a genuinely blank 1000×700 white image generated with Pillow
(`Image.new("RGB", (1000, 700), (255, 255, 255))`, sha256
`5eb1fdc728b8345e0068953c619155579223c382a5eeb1d44d5051e0852631c9`, 3676 bytes). It contains no text,
so OCR produces nothing. The scenario was run **twice** (run 1 → doc 22, deleted; **run 2 → doc 23,
kept**).

### DURING — the empty result triggers the force-OCR safe fallback, then the empty-content warning [observed]

When the first (skip-text) OCR pass yields no text, `parse()` raises `NoTextFoundException`
(`src/paperless_tesseract/parsers.py:267`), which is caught (`:276`) and retried once with
`force_ocr` (`Fallback: Calling OCRmyPDF …`, `:297`). When that still yields nothing, the parser logs
`No text was found …, the content will be empty` (`src/paperless_tesseract/parsers.py:324`) and sets
`self.text = ""` (`:327`).

**Command + log (run 2, doc 23) [observed]:**

```
$ python3 /tmp/blitzy_probes/scripts/upload.py /tmp/blitzy_probes/samples/blank_white.png q4_blank_run2
[upload 18:05:17.503] POST http://localhost:8000/api/documents/post_document/
HTTP 200
BODY: "OK"

[2026-07-13 18:05:18,966] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-13 18:05:18,967] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/paperless/paperless-upload-gpd9nedu', 'output_file': '/tmp/paperless/paperless-8q58kgsz/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-8q58kgsz/sidecar-fallback.txt', 'image_dpi': 120}
[2026-07-13 18:05:20,035] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/paperless/paperless-upload-gpd9nedu, the content will be empty.
```

### AFTER — consumption STILL SUCCEEDS [observed]

Despite the empty text, the task ends in **`SUCCESS`** with a real `document_id`. The listener started
before the upload; the full frame sequence:

```
LISTENER_START 18:05:17.373 connected=ws://localhost:8000/ws/status/
[18:05:17.673] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[18:05:17.693] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[18:05:20.056] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[18:05:20.529] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[18:05:20.544] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[18:05:20.614] {"filename": "blank_white.png", "task_id": "f3a44d7e-c458-48e3-b338-c5b12675dc8e", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 23}
```

The django-q task is recorded as a success [observed]:

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py shell < /tmp/qtask.py'
id=4f637b37505e4ac38ce9ac19a7c45a31 func=documents.tasks.consume_file name='blank_white.png' success=True started=18:05:17 stopped=18:05:20 result='Success. New document id 23 created'
```

### The saved metadata — empty content, but a real archive [observed]

The stored document has `content=""` (empty string) yet a **non-null** `archived_file_name` — the
archive PDF is produced regardless of whether any text was found. The response still has all 12
fields (no `storage_path`/`created_date`):

```
$ python3 /tmp/blitzy_probes/scripts/api_get.py 23
GET http://localhost:8000/api/documents/23/ -> HTTP 200
{
  "added": "2026-07-13T18:05:20.563609Z",
  "archive_serial_number": null,
  "archived_file_name": "2026-07-13 q4_blank_run2.pdf",
  "content": "",
  "correspondent": null,
  "created": "2026-07-13T18:05:17.510344Z",
  "document_type": null,
  "id": 23,
  "modified": "2026-07-13T18:05:20.592571Z",
  "original_file_name": "2026-07-13 q4_blank_run2.png",
  "tags": [],
  "title": "q4_blank_run2"
}
FIELD_COUNT 12
```

Cross-check directly in the database [observed]:

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=23); print(repr(d.content), d.archive_filename)"'
'' 0000023.pdf
```

**Answer to Q4 (final state + saved metadata) [observed].** Weak/incomplete OCR **still counts as
fully processed**: the terminal status is `SUCCESS`, a `Document` row is created, an archive PDF is
generated (`archived_file_name` non-null, `has_archive_version` true, `src/documents/models.py:238`),
and the only reflection of the weak OCR in the saved metadata is `content=""`. There is no partial or
"degraded" state — an empty OCR result is a normal success.

### Contrast — what a real FAILED looks like [observed]

Because the empty-OCR path succeeds, it is worth showing when a document *does* reach `FAILED`.
`FAILED` is emitted only by `Consumer._fail` (`src/documents/consumer.py:78-81`). Two distinct
failure mechanisms were genuinely observed by re-uploading the same bytes (a checksum collision):

**(1) EARLY — the duplicate pre-check (`document_already_exists`).** Re-uploading `blank_white.png`
while doc 23 still existed failed **immediately after `STARTING`**, before any parsing. The
`pre_check_duplicate` method MD5s the **original** upload bytes
(`hashlib.md5(...)`, `src/documents/consumer.py:104`), matches
`Q(checksum=…) | Q(archive_checksum=…)` (`:106`), and calls `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS)`
(`:111`, message constant `src/documents/consumer.py:37`):

```
LISTENER_START 18:06:00.343 connected=ws://localhost:8000/ws/status/
[18:06:00.674] {"filename": "blank_white.png", "task_id": "cc09deee-d9e3-40bc-a29c-ed59d6945cc6", "current_progress": 0,   "status": "STARTING", "message": "new_file",              "document_id": null}
[18:06:00.697] {"filename": "blank_white.png", "task_id": "cc09deee-d9e3-40bc-a29c-ed59d6945cc6", "current_progress": 100, "status": "FAILED",   "message": "document_already_exists","document_id": null}
```

**(2) LATE — a database `IntegrityError` at store time.** An **alpha** image (`no-text-alpha.png`)
behaves differently: the parser rewrites the temp file in place to remove the alpha layer
(`src/paperless_tesseract/parsers.py:194`), so the **stored** checksum differs from the original
upload's MD5. `pre_check_duplicate` therefore **misses**, the task runs the entire pipeline, and the
collision only surfaces at the final DB insert. First a seed upload succeeds
(`no-text-alpha.png` → doc 24), then an identical re-upload runs fully and fails at
`save_document/95`:

```
$ python3 /tmp/blitzy_probes/scripts/upload.py /tmp/blitzy_probes/samples/no-text-alpha.png q4_interr_dup
[upload 18:08:04.565] POST http://localhost:8000/api/documents/post_document/
HTTP 200
BODY: "OK"

# WebSocket frames — the failure is LATE, after a full parse/thumbnail/save attempt:
[18:08:04.749] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
[18:08:04.775] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
[18:08:06.786] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
[18:08:14.367] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
[18:08:14.384] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
[18:08:14.405] {"filename": "no-text-alpha.png", "task_id": "b9fb35b7-c405-48f0-a4eb-4c785c8e9ed5", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "UNIQUE constraint failed: documents_document.checksum", "document_id": null}
```

The `paperless.consumer` log records the same at 18:08:14.405 [observed]:

```
[2026-07-13 18:08:14,384] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:08:14,405] [ERROR] [paperless.consumer] The following error occured while consuming no-text-alpha.png: UNIQUE constraint failed: documents_document.checksum
```

The django-q `Task.result` holds the **full traceback**, which shows the exact exception type and
path — a `django.db.utils.IntegrityError`, **not** a `ParseError` [observed]:

The producing command dumps the failed `Task` row; its `result` holds the complete traceback,
reproduced here **unedited** [observed]:

```
$ docker exec paperless-app bash -c 'cd /app/src && python3 manage.py shell < /tmp/qtask.py'
id=668493911f0942678ed04410e94ed65c name=no-text-alpha.png success=False started=18:08:04 stopped=18:08:14
--- result ---
no-text-alpha.png: The following error occured while consuming no-text-alpha.png: UNIQUE constraint failed: documents_document.checksum : Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/utils.py", line 89, in _execute
    return self.cursor.execute(sql, params)
  File "/usr/local/lib/python3.9/site-packages/django/db/backends/sqlite3/base.py", line 477, in execute
    return Database.Cursor.execute(self, query, params)
sqlite3.IntegrityError: UNIQUE constraint failed: documents_document.checksum

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/asgiref/sync.py", line 266, in main_wrap
    raise exc_info[1]
  File "/app/src/documents/consumer.py", line 301, in try_consume_file
    document = self._store(text=text, date=date, mime_type=mime_type)
  File "/app/src/documents/consumer.py", line 398, in _store
    document = Document.objects.create(
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

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker
    res = f(*task["args"], **task["kwargs"])
  File "/app/src/documents/tasks.py", line 236, in consume_file
    document = Consumer().try_consume_file(
  File "/app/src/documents/consumer.py", line 363, in try_consume_file
    self._fail(
  File "/app/src/documents/consumer.py", line 81, in _fail
    raise ConsumerError(f"{self.filename}: {log_message or message}")
documents.consumer.ConsumerError: no-text-alpha.png: The following error occured while consuming no-text-alpha.png: UNIQUE constraint failed: documents_document.checksum
```

This traceback pins the mechanism precisely **[observed]**: the `IntegrityError` is raised inside
`_store` at `Document.objects.create` (`src/documents/consumer.py:398`, called from `:301`); it is
caught by the **generic** `except Exception` at `src/documents/consumer.py:363`, which calls `_fail`
(`:78`), which sends the `FAILED` WebSocket frame (`:79`) and raises `ConsumerError` (`:81`).

> **[inferred]** The parser can also raise `ParseError`, which is caught by the earlier
> `except ParseError` at `src/documents/consumer.py:278` and routed to the same `_fail`. That branch
> was **not** triggered in this investigation (the empty-OCR case takes the success path, and the
> duplicate case takes the `IntegrityError` path above), so the `ParseError → FAILED` route is
> labelled inferred from the code rather than observed.


---

## Methodology & Reproducibility

- **Canonical entry point.** Every scenario was driven through `POST /api/documents/post_document/`
  (`src/documents/views.py:497`), which enqueues `documents.tasks.consume_file` via `async_task`
  (`:523`). No parser was called directly; no debug hook or bypass was used.
- **Canonical, default configuration.** `PAPERLESS_OCR_MODE` was left at its default `skip`
  (`src/paperless/settings.py:522`); the full effective config is captured above. No setting was
  changed to obtain the primary observations.
- **Listener-before-upload.** For every scenario the WebSocket listener printed `LISTENER_START` with
  a timestamp *earlier* than the upload timestamp, so no frame was missed. Both timestamps are shown
  in each section.
- **Two runs each.** Every input was ingested **twice** with identical bytes (deleting the
  intervening document so the second upload is not rejected as a duplicate). The frame order was
  identical across runs, and the one timing claim (the Q1 OCR window) was stable across both runs
  (1.955 s vs. 1.952 s). Documents observed per scenario: Q1 → 13, 14 (+ text demo 15); Q2 → image
  16/17, PDF 18/19, multi-page 20/21; Q4 → blank 22/23, plus the IntegrityError seed 24 and its
  duplicate.
- **Exact bytes.** Sidecar bytes were read with Python `Path(...).read_bytes()` + `.hex()` on the
  concrete per-task file captured by the in-container watcher (recording path, inode, size, sha256).
  `xxd` is **not** installed in this image, so no `xxd` output appears anywhere in this document.
- **Cross-layer identifiers.** The WebSocket `task_id` (dashed UUID, `src/documents/views.py:521`)
  and the django-q `Task.id` (32-hex, no dashes) are different identifiers; correlation used
  filename + timing + `document_id`.
- **Observed vs. inferred.** Claims from captured output are `[observed]`; the only significant
  `[inferred]` claim is the `ParseError → FAILED` branch in Q4 (the code path exists but was not
  triggered at runtime).

## Cleanup & Security Teardown

This was a **read-only** investigation. The steps below restore the repository and destroy all
temporary state.

- **Repository unchanged (read-only rule).** No source file was modified. Uploads used **copies** of
  fixtures under `/tmp/blitzy_probes/samples/`, never the repository fixtures themselves, so the
  in-place alpha-removal that OCRmyPDF's preprocessing performs
  (`src/paperless_tesseract/parsers.py:194`) touched only temp copies. The runtime checkout's git
  tree is verified clean:

  ```
  $ docker exec paperless-app git -C /app status --porcelain
  (empty output — no modified files)
  ```

- **Documents created during the investigation were deleted** via the REST API, and the count
  returned to zero.
- **Auth material destroyed.** The seeded admin password was **rotated** to an ephemeral random value
  before any upload (never printed, kept only in a mode-`0600` file); the DRF token and DB session
  created for the probes were revoked/deleted; the `0600` credential file was removed at teardown. No
  live credential is disclosed anywhere in this document.
- **Runtime stopped and isolated.** After all observations were captured, the published service and
  its Redis broker were stopped and removed so nothing remains listening on `0.0.0.0:8000`:

  ```
  $ docker stop paperless-app paperless-redis && docker rm paperless-app paperless-redis
  paperless-app
  paperless-redis
  paperless-app
  paperless-redis
  $ docker network rm paperless-net
  paperless-net
  $ docker ps --format '{{.Names}}' | grep -i paperless || echo '(no paperless containers remain)'
  (no paperless containers remain)
  ```
- **Probe harness removed.** Everything under `/tmp/blitzy_probes/` (scripts, sample copies, captures)
  and the in-container `/tmp/*.py` helpers and `/tmp/cap_*` capture dirs were deleted after the
  observations were recorded. The only artifact that remains is this document.

## Appendix A — Observed-vs-inferred summary

| Claim | Label |
|---|---|
| OCR start signal = `Calling OCRmyPDF with args` + WS `WORKING/parsing_document/20` | observed |
| Status holds at `WORKING/20` during OCR (no intra-OCR progress); stable ~1.95 s over 2 runs | observed |
| Worker is django-q `qcluster` running `consume_file`; no `document_consumer` running | observed |
| Image always OCRs (`original_has_text=False` for images) | observed (log) + code-cited |
| Text PDF still calls OCRmyPDF; sidecar marker discarded; pdfminer used | observed |
| Exact sidecar bytes (image 25 B text; PDF 26 B marker; multi-page 28 B marker) | observed |
| API returns exactly 12 fields; no `storage_path`/`created_date` | observed |
| `content` is the sole text carrier for both cases; no provenance field | observed + code-cited |
| Weak/empty OCR → `SUCCESS`, `content=""`, archive still produced | observed |
| Duplicate re-upload → `FAILED` via `IntegrityError` (early pre-check or late `_store`) | observed |
| `ParseError → FAILED` branch (`consumer.py:278`) | **inferred** (not triggered at runtime) |

## Appendix B — Citation index (repo-relative, commit `542221a38dff`)

- `src/documents/views.py:497` `PostDocumentView.post`; `:521` `task_id = str(uuid.uuid4())`; `:523`
  `async_task("documents.tasks.consume_file", …)`; `:535` `Response("OK")`.
- `src/documents/tasks.py:184` `consume_file`; `:236` `Consumer().try_consume_file(...)`.
- `src/documents/consumer.py:37` `MESSAGE_DOCUMENT_ALREADY_EXISTS`; `:43-49` status message
  constants; `:56` `_send_progress` (payload `:64-72`); `:78-81` `_fail` (FAILED send `:79`, raise
  `ConsumerError` `:81`); `:102-111` `pre_check_duplicate` (`hashlib.md5` `:104`, `Q(...)` `:106`,
  `_fail` `:111`); `:202` `STARTING/new_file`; `:259` `WORKING/parsing_document/20`; `:261`
  `parse()`; `:264` `generating_thumbnail/70`; `:274` `parse_date/90`; `:278-280` `except ParseError
  → _fail`; `:294` `save_document/95`; `:301` `self._store(...)`; `:362-363` generic `except
  Exception → _fail`; `:375` `SUCCESS/finished/100`; `:398` `Document.objects.create`; `:400`
  `content=text`.
- `src/paperless_tesseract/parsers.py:99` `extract_text`; `:104` `[OCR skipped on page` check; `:107`
  `Using text from sidecar file`; `:110` `Incomplete sidecar file: discarding.`; `:135`
  `construct_ocrmypdf_parameters`; `:152` `progress_bar False`; `:156` `force_ocr`; `:158`
  `skip_text`; `:160` `redo_ocr`; `:194` `Removing alpha layer`; `:230` `parse()`; `:234` `if
  mime_type == "application/pdf":` branch guard; `:236` has-text length test (`len > 50`);
  `:238-239` image `text_original=None`/`original_has_text=False`; `:260` `Calling
  OCRmyPDF with args`; `:267` raise `NoTextFoundException`; `:276` `except`; `:297` `Fallback: Calling
  OCRmyPDF`; `:324` `content will be empty`; `:327` `self.text = ""`.
- `src/documents/parsers.py:300` `progress()` (unused hook); `:310` `get_archive_path`; `:342`
  `get_text`.
- `src/documents/serialisers.py:207` `original_file_name` field; `:208` `archived_file_name` field;
  `:210` `get_original_file_name`; `:213` `get_archived_file_name`; `:219` `class Meta`; `:222-235`
  the 12-field tuple; `:227` `content`.
- `src/documents/models.py:117` `content`; `:143` `archive_checksum`; `:186` `archive_filename`;
  `:238` `has_archive_version`.
- `src/paperless/settings.py:50` `DEBUG`; `:110` `django_q`; `:113` channels-appended-if-DEBUG; `:180`
  `RedisChannelLayer`; `:449` `Q_CLUSTER` (`:451` `catch_up`); `:522` `OCR_MODE` default `skip`.
- `src/paperless/consumers.py:9` `StatusConsumer`; `:15` `DenyConnection`; `:29` `status_update`;
  `:33` `self.send(json.dumps(event["data"]))`.
- `src/paperless/urls.py:136-137` `websocket_urlpatterns` + `ws/status/` route.
- `src/paperless/asgi.py:17` `ProtocolTypeRouter`; `:20` WebSocket behind `AuthMiddlewareStack`.
- `src/documents/management/commands/document_consumer.py:86-87` `async_task(
  "documents.tasks.consume_file", …)` (alternate entry point, not used here).

## Appendix C — Named-item coverage checklist

| Question | Named item | Where answered |
|---|---|---|
| Q1 | "see that OCR has started" | `Calling OCRmyPDF with args` log + WS `WORKING/parsing_document/20` |
| Q1 | "processing state while running" | ordered `status_updates` frames; holds at `WORKING/20` |
| Q1 | "background workers behave" | django-q `qcluster` runs `consume_file`; `Task` row |
| Q1 | "signals of active OCR work" | the OCRmyPDF args line; the held `WORKING/20`; the ~1.95 s window |
| Q2 | "skip OCR entirely, or still touch the pipeline" | image always OCRs; text PDF still calls OCRmyPDF |
| Q2 | "how to tell the difference after" | mime type, pdfminer pre-extract, `image_dpi` key, sidecar bytes |
| Q3 | "which fields show OCR-generated vs existing text" | both in `content`; no provenance field; 12-field schema |
| Q4 | "does it still count as fully processed" | yes — terminal `SUCCESS`, document row created |
| Q4 | "how reflected in saved metadata" | `content=""`, archive still produced (`archived_file_name` non-null) |
| Q4 | contrast: real `FAILED` | duplicate `IntegrityError` (early pre-check + late `_store`), full traceback |
