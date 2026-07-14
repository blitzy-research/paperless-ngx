# OCR Runtime Behavior in paperless-ngx — A Run-First Investigation

> **Repository / commit:** `paperless-ngx` @ `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`)
> **Nature of this document:** A _run-first, evidence-grounded_ answer. The relevant code paths were built and run first, and the real output was captured before any conclusion was drawn. Two kinds of statement appear below and are kept visibly distinct:
> - **_observed at runtime_** — paired with the exact command that produced it and the complete, unedited output (progress payloads, worker logs, API JSON, DB reads, probe output);
> - **_inferred from code_** — a mechanistic explanation derived from reading the source at the commit above, cited by `file:line`, that was *not* directly measured.
>
> Where a claim is a mechanism ("why") rather than a measurement ("what happened"), it is labeled **_inferred from code_**. Everything shown inside a captured output block is **_observed_**.
>
> **Runtime used:** the canonical **Python 3.9** stack inside the provided Docker image (`tesseract` + `ocrmypdf` present). Observations were **not** produced on the planning shell (host Python 3.13, which lacks `tesseract`/`ocrmypdf`).

---

## 1. Answer Summary (the short version)

The four sub-questions, answered up front; each is proven in detail in the correspondingly-named section below.

- **P1 — How do I see that OCR started, and what does the processing state look like (text-free image)?**
  Ingestion is **asynchronous**: the HTTP upload returns `Response("OK")` in ~0.03 s (before any OCR), and all OCR work happens in a **django-q background worker**. You watch OCR through two real signals: (a) the **WebSocket progress stream** broadcast to the Channels group `"status_updates"`, which for the observed successful image consumes walked the sequence `STARTING/new_file@0%` → **`WORKING/parsing_document@20%` (the parsing/OCR-phase checkpoint)** → `WORKING/generating_thumbnail@70%` → `WORKING/parse_date@90%` → `WORKING/save_document@95%` → `SUCCESS/finished@100%`; and (b) the worker's DEBUG log line **`Calling OCRmyPDF with args: {…}`** from logger `paperless.parsing.tesseract`, which is the definitive *engine-call* signal (it fires a fraction of a second after the 20% checkpoint, inside `parse()`). The `document_id` field is `null` in the progress payloads until the terminal `SUCCESS` — but that is a property of how the payload is built, **not** proof that no row exists during processing: an independent DB poll shows the committed row appears *after* the 95% event and just *before* the 100% event.

- **P2 — Does an image that already shows text skip OCR?**
  **No.** For **any image (non-PDF) input, OCR always runs** (_observed_ for both a text-free and a text-bearing image; the _inferred-from-code_ reason is that the parser unconditionally sets `original_has_text = False` for images, so the `skip_noarchive` early-exit can never fire for an image). The pipeline shape and the progress/log signature are **identical** to the text-free case, so _you cannot tell "the image had visible text" from the output shape_. The only artifact that distinguishes "text produced by OCR" from "a page that already had a text layer" is the **ocrmypdf sidecar marker `[OCR skipped on page(s) …]`** — observed present for a text-layer PDF and absent for images.

- **P3 — Which API fields carry OCR-generated vs. pre-existing text?**
  Both cases funnel recognized text into the **single `content` field**, and both exposed a populated **`archived_file_name`**. The two responses were structurally identical (same 12 keys). **There is no field that labels text provenance** — nothing in `GET /api/documents/{id}/` says "this text came from OCR" vs. "this text pre-existed." (`archived_file_name` being non-null means archive *metadata* is set on the row, i.e. `archive_filename is not None`; it does not encode text provenance.)

- **P4 — What happens when OCR is weak/incomplete?**
  The document is **still fully processed.** When no text is recovered, the parser logs `No text was found in {document_path}, the content will be empty.` and sets the content to `""`; the consume still reached `SUCCESS/finished@100%` and a normal `Document` row was written (with an archive PDF, for an image). "Fully processed" has **no dedicated status/"processed" column** on the model — success is signalled only by _the row existing_ plus the terminal `finished@100%` event. An empty-OCR result is **not** an error and does **not** prevent creation. It must be distinguished from a hard failure: in the observed comparison a genuine `ParseError` (corrupt PDF) produced `FAILED@100%` with **no row**. That `ParseError` is **not** the only no-row outcome, though — a duplicate upload was *also* observed to produce `FAILED@100%`/no row (failing at a pre-check, before OCR even started), and several other `_fail` paths exist in the source (enumerated in §7.4).

---

## 2. Environment & Method

All observations were produced on the **canonical runtime** inside the provided Docker image, not the planning shell.

### 2.1 Image provenance (exact, pinned)

**Command (observed):**

```bash
# base (source) image, by immutable digest
docker image inspect \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01 \
  --format '{{index .RepoDigests 0}}{{"\n"}}{{.Id}}'
# derived, ready-to-run image (setup fixes baked in) and its parent
docker image inspect paperless-ngx-ready:local --format 'derived={{.Id}} parent={{.Parent}}'
```

**Output (observed):**

```
ghcr.io/scaleapi/swe-atlas@sha256:d4abe56dd5d1cb2632353baf06e9d80a704f147a70ac2ec15c2b78fc5fddfe15
sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f
derived=sha256:241e817e0d4a3fc22cfecb25e5d9b662ea7a373df19e28a318d72ba8f787fd01 parent=sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f
```

The derived image `paperless-ngx-ready:local` is a child of the base image (matching parent digest). The in-image repository copy at `/app` is at the target commit:

```bash
$ docker exec paperless-app bash -lc 'git config --global --add safe.directory /app; cd /app && git rev-parse HEAD'
542221a38dff06361e07976452f9aea24d210542
```

### 2.2 Toolchain verification (captured)

**Command + output (observed):**

```console
$ docker exec paperless-app bash -lc 'cd /app/src && python3 --version'
Python 3.9.23

$ docker exec paperless-app bash -lc 'tesseract --version 2>&1 | head -1'
tesseract 4.1.1

$ docker exec paperless-app bash -lc 'cd /app/src && python3 -c "import ocrmypdf; print(\"ocrmypdf\", ocrmypdf.__version__)"'
ocrmypdf 13.4.3

$ docker exec paperless-app bash -lc 'gs --version 2>&1 | head -1'
9.53.3
```

### 2.3 Pinned dependencies confirmed at runtime (match `requirements.txt`)

**Command (observed) — runnable, prints the installed versions:**

```bash
docker exec paperless-app bash -lc 'cd /app/src && python3 -c "
import importlib.metadata as m
for p in [\"django\",\"djangorestframework\",\"django-q\",\"channels\",\"channels-redis\",
          \"redis\",\"pdfminer.six\",\"pikepdf\",\"pillow\",\"img2pdf\",\"ocrmypdf\"]:
    print(f\"{p:22s} {m.version(p)}\")
"'
```

**Output (observed):**

```
django                 4.0.4
djangorestframework    3.13.1
django-q               1.3.9
channels               3.0.4
channels-redis         3.4.0
redis                  3.5.3
pdfminer.six           20220319
pikepdf                5.1.1
pillow                 9.1.0
img2pdf                0.4.4
ocrmypdf               13.4.3
```

These match the pins in `requirements.txt` (e.g. `django-q==1.3.9`, `ocrmypdf==13.4.3`, `channels==3.0.4`, `pdfminer.six==20220319`).

### 2.4 Default OCR settings, resolved live by Django (captured)

**Command (observed) — runnable, initializes Django then prints resolved settings:**

```bash
docker exec paperless-app bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings python3 -c "
import django; django.setup()
from django.conf import settings
for k in [\"OCR_MODE\",\"OCR_OUTPUT_TYPE\",\"OCR_CLEAN\",\"OCR_LANGUAGE\",\"OCR_PAGES\",\"OCR_IMAGE_DPI\"]:
    print(f\"{k:16s}= {getattr(settings, k)!r}\")
"'
```

**Output (observed):**

```
OCR_MODE        = 'skip'
OCR_OUTPUT_TYPE = 'pdfa'
OCR_CLEAN       = 'clean'
OCR_LANGUAGE    = 'eng'
OCR_PAGES       = 0
OCR_IMAGE_DPI   = None
```

These are the _canonical defaults_ (`src/paperless/settings.py:L510-L526`); any run that overrides them is labeled **NON-CANONICAL** below. Documented semantics of the default `skip` mode: it "only performs OCR when necessary and always creates archived documents", whereas `skip_noarchive` additionally avoids creating an archive when text already exists (`docs/configuration.rst:§L305-L330`).

### 2.5 How the stack was run (services + health)

The full asynchronous stack was stood up so the worker and its progress stream are genuinely observable.

**Commands (observed):**

```bash
docker network create paperless-net
docker run -d --name paperless-redis --network paperless-net redis:7-alpine
docker run -d --name paperless-app --network paperless-net --user root \
  -e PAPERLESS_REDIS=redis://paperless-redis:6379 \
  -e PAPERLESS_DATA_DIR=/tmp/pl/data -e PAPERLESS_MEDIA_ROOT=/tmp/pl/media \
  -e PAPERLESS_STATICDIR=/tmp/pl/static -e PAPERLESS_CONSUMPTION_DIR=/tmp/pl/consume \
  -e PAPERLESS_SCRATCH_DIR=/tmp/pl/scratch -e PAPERLESS_LOGGING_DIR=/tmp/pl/log \
  -e PAPERLESS_SECRET_KEY=dev-key -e PAPERLESS_TIME_ZONE=UTC \
  --entrypoint bash paperless-ngx-ready:local \
  -lc 'mkdir -p /tmp/pl/{data,media,static,consume,scratch,log}; tail -f /dev/null'

docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py migrate --no-input'
docker exec paperless-app bash -lc 'cd /app/src && DJANGO_SUPERUSER_USERNAME=admin \
  DJANGO_SUPERUSER_PASSWORD=adminpass DJANGO_SUPERUSER_EMAIL=admin@example.com \
  python3 manage.py createsuperuser --noinput'
docker exec -d paperless-app bash -lc 'cd /app/src && exec python3 manage.py runserver 0.0.0.0:8000 --noreload > /tmp/rt_web.log 2>&1'
docker exec -d paperless-app bash -lc 'cd /app/src && exec python3 manage.py qcluster > /tmp/rt_qcluster.log 2>&1'
```

**Health (observed):**

```console
# qcluster worker came up (last line of /tmp/rt_qcluster.log):
[Q] INFO Q Cluster princess-butter-lake-seventeen running.

# web server (banner from /tmp/rt_web.log):
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://0.0.0.0:8000/

# API reachable:
$ TOKEN=$(docker exec paperless-app bash -lc "curl -s -X POST -d 'username=admin&password=adminpass' http://localhost:8000/api/token/ | python3 -c 'import sys,json;print(json.load(sys.stdin)[\"token\"])'")
$ docker exec paperless-app bash -lc "curl -s -o /dev/null -w 'GET /api/ -> HTTP %{http_code}\n' -H \"Authorization: Token $TOKEN\" http://localhost:8000/api/"
GET /api/ -> HTTP 200
```

- **Redis broker** (`redis:7-alpine`) — backs both django-q and the Channels channel layer.
- **django-q worker** — `python3 manage.py qcluster` (`"django_q"` in `INSTALLED_APPS` — `src/paperless/settings.py:L110`; `Q_CLUSTER` config — `src/paperless/settings.py:L449`). This is the process that executes `consume_file` and performs OCR.
- **Database** — SQLite at `/tmp/pl/data/db.sqlite3`, reset to a **clean slate** (0 documents) before observation.
- **API token** — obtained via `POST /api/token/` (`username=admin`) into a shell variable `$TOKEN`; sent as `Authorization: Token $TOKEN` on every upload/GET. The literal token value is never printed in this document.

> **Security note.** The token is kept in a shell variable and never emitted. Redis, the SQLite DB, and the HTTP/WS ports are bound only to the disposable observation containers on a private Docker network; they are not exposed to untrusted clients.

### 2.6 How progress was captured (two methods, both real)

Progress is broadcast by the worker to the Channels group `"status_updates"` over the configured Redis channel layer (`src/documents/consumer.py:L73-L74`). Two real capture methods were used; both are published in full in the **Appendix (§12)** so the exact commands, secure temp handling, and cleanup are auditable.

1. **Channel-layer subscriber (`sub.py`) — a PRIVILEGED internal probe.** It joins the *same* `"status_updates"` group the browser's `StatusConsumer` joins (`src/paperless/consumers.py:L16-L19`) via the app's configured Redis channel layer, and logs each broadcast verbatim with wall-clock + monotonic timestamps. It is the exact same Channels mechanism the app uses — **not** a mock. It **bypasses** `StatusConsumer`'s authentication gate (`src/paperless/consumers.py:L13-L15`), so it is privileged instrumentation for observation, not a method a normal WebSocket client could use.
2. **Authenticated WebSocket client (`ws_probe.py`) — the browser-equivalent path.** It connects to the real WebSocket route `ws/status/` (`src/paperless/urls.py:L137`) with a genuine logged-in session cookie and receives the same payloads the browser front-end does. This is used in §4.8 to demonstrate the practical, non-privileged observation method.

**Component-level probes (labeled NON-CANONICAL):** to expose the ocrmypdf **sidecar file** (which the canonical async path deletes on cleanup) and to exercise `skip_noarchive`, two throwaway scripts (`probe_sidecar.py`, `probe_skip.py`, §12) called the **real** `RasterisedDocumentParser.parse(...)` directly (no mocks). All such output is explicitly labeled **_component-level probe (NON-CANONICAL)_**. Every probe **copies the fixture into a private `tempfile.mkdtemp()` directory and parses only the copy**, so repository fixtures are never touched (proven in §11).

**Temporary-artifact hygiene:** every scratch item (scripts, scratch media, SQLite DB, media/consume dirs) lives **outside** the repository tree — under `/tmp` on the host and inside the disposable containers. The repository working tree is left pristine except for this one document — proven in §11.

---

## 3. Pipeline Overview (the async path, with citations)

The observable OCR behavior lives entirely in the **background worker**, because the HTTP request returns before any OCR happens. The following is **_inferred from code_** (a map of the source), and each checkpoint is **_observed_** in §4.

```text
POST /api/documents/post_document/            src/documents/views.py:L497 (PostDocumentView.post)
   │  writes temp file in SCRATCH_DIR                        views.py:L512-L518
   │  task_id = str(uuid.uuid4())                            views.py:L521
   │  async_task("documents.tasks.consume_file", …)          views.py:L523 (task_id passed in, NOT returned)
   │  return Response("OK")   ← returns immediately, no OCR   views.py:L535
   ▼ (enqueued on Redis)
django-q worker → consume_file(...)           src/documents/tasks.py:L184
   │            → Consumer().try_consume_file  src/documents/tasks.py:L236
   ▼
Consumer.try_consume_file(...)                src/documents/consumer.py:L180
   ├─ STARTING / new_file            @ 0%     consumer.py:L202
   ├─ pre-checks: file exists / duplicate / mime  consumer.py:L211-L221 (each may _fail → FAILED, no row)
   ├─ WORKING  / parsing_document    @ 20%     consumer.py:L259   ← parsing/OCR-phase checkpoint
   │     └─ RasterisedDocumentParser.parse(…)  src/paperless_tesseract/parsers.py:L230-L327
   │           └─ log "Calling OCRmyPDF with args: {…}"        parsers.py:L260   ← engine-call signal
   │           └─ ocrmypdf.ocr(**args)                          parsers.py:L261
   ├─ WORKING  / generating_thumbnail @ 70%    consumer.py:L264
   ├─ (if no date parsed) WORKING / parse_date @ 90%  consumer.py:L273-L274  ← conditional
   ├─ WORKING  / save_document       @ 95%     consumer.py:L294
   ├─ with transaction.atomic():               consumer.py:L298
   │     └─ Consumer._store(...) → Document.objects.create(content=text, …)
   │                                            consumer.py:L379-L412 (create L398, content=text)
   │     (commit occurs when the atomic block exits — the row is now visible to other connections)
   ├─ run_post_consume_script(document)        consumer.py:L371 (runs AFTER commit)
   └─ SUCCESS  / finished            @ 100% (with document.id)  consumer.py:L375
   (on ParseError during parse: _fail → FAILED@100% + raise ConsumerError  parsers.py:L310/L314; consumer.py:L78-L81)
```

**The progress payload** is emitted by `Consumer._send_progress(...)` (`consumer.py:L56-L76`) to the group `"status_updates"` (`consumer.py:L73-L74`). Every payload has **exactly seven keys** (_observed_ in every stream below):

```
filename, task_id, current_progress, max_progress, status, message, document_id
```

The human-readable `message` strings are module constants: `new_file` (`consumer.py:L43`), `parsing_document` (`L45`), `generating_thumbnail` (`L46`), `parse_date` (`L47`), `save_document` (`L48`), `finished` (`L49`).

**Two precision points (_inferred from code_, confirmed by the observed timings in §4):**

- The **20% `parsing_document`** event (`consumer.py:L259`) marks *entry into the parsing/OCR phase* — it fires immediately *before* `parse()` is called. The definitive *engine-call* signal is the `Calling OCRmyPDF with args:` DEBUG log (`parsers.py:L260`), which fires a fraction of a second later, inside `parse()`.
- The **90% `parse_date`** event is **conditional**: it is sent only when the parser did not already extract a date (`if not date:` — `consumer.py:L273-L274`). For the observed fixtures (no filename/embedded date) it fired every time; for an input that yields a date it would be skipped. A failure terminates the sequence early (observed for the corrupt PDF: `0 → 20 → FAILED`, and for a duplicate: `0 → FAILED`).

---

## 4. P1 — Observing OCR start & processing state (text-free image `no-text-alpha.png`)

**Input:** `src/paperless_tesseract/tests/samples/no-text-alpha.png` — an image with no embedded text layer.

Two complete, independent runs were captured (run A → document id 1; run B → document id 2). Run A is shown in full with its DB poll; the stability comparison is in §4.7.

### 4.1 The upload is asynchronous, and the task id is NOT returned

**Command (observed):**

```bash
docker exec paperless-app bash -lc "curl -s -o /tmp/obs/p1a_body.txt \
  -w 'HTTP_CODE=%{http_code}\nTIME_TOTAL=%{time_total}s\n' \
  -H \"Authorization: Token $TOKEN\" \
  -F document=@/app/src/paperless_tesseract/tests/samples/no-text-alpha.png \
  http://localhost:8000/api/documents/post_document/"
docker exec paperless-app bash -lc 'echo -n "body="; cat /tmp/obs/p1a_body.txt'
```

**Output (observed):**

```
HTTP_CODE=200
TIME_TOTAL=0.032244s
body="OK"
```

The endpoint returns the literal JSON string `"OK"` in ~0.03 s. OCR itself took ~10 s in this run (see §4.3), so the HTTP response demonstrably precedes OCR (enqueue-and-return at `views.py:L523-L535`).

**Important for "watching" (answering the user's real need):** the endpoint mints a `task_id` (`views.py:L521`) but **does not return it** — the body is only `"OK"` (`views.py:L535`). So a client **cannot** obtain the task id from the upload response. Correlation is done on the *progress stream* instead, where each event carries both `task_id` and `filename` (§4.3), or via the worker log. This is why "watch the worker/stream", not "read the POST response", is the correct approach.

### 4.2 "Before" state — the document row does not exist yet

**Command (observed):**

```bash
docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c \
  "from documents.models import Document; print(Document.objects.count())"'
```

**Output (observed):** `0`

### 4.3 "During" state — the live progress stream (full, unedited payloads)

This is the complete `"status_updates"` broadcast for run A, captured by the channel-layer subscriber (`sub.py`, §12). The first line is the subscriber's own readiness marker; the rest are the verbatim worker broadcasts. **All seven payload keys are present in every event**, and `task_id` is constant across the task:

```json
{"event": "subscriber_ready", "group": "status_updates"}
{"recv_wall": "2026-07-14 20:51:55.212", "recv_mono_s": 1.936, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 20:51:55.233", "recv_mono_s": 1.956, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:51:57.389", "recv_mono_s": 4.112, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}}
{"recv_wall": "2026-07-14 20:52:05.105", "recv_mono_s": 11.828, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}}
{"recv_wall": "2026-07-14 20:52:05.121", "recv_mono_s": 11.844, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:52:05.193", "recv_mono_s": 11.916, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

Reading this as the "processing state":

| Event | `status`   | `message`              | `current_progress` | `document_id` | Meaning / citation                                                        |
| ----- | ---------- | ---------------------- | ------------------ | ------------- | ------------------------------------------------------------------------- |
| 1     | `STARTING` | `new_file`             | `0`                | `null`        | consume accepted — `consumer.py:L202`                                     |
| 2     | `WORKING`  | `parsing_document`     | `20`               | `null`        | **parsing/OCR-phase entry** — emitted just before `parse()` — `consumer.py:L259` |
| 3     | `WORKING`  | `generating_thumbnail` | `70`               | `null`        | OCR finished; building thumbnail — `consumer.py:L264`                     |
| 4     | `WORKING`  | `parse_date`           | `90`               | `null`        | date parsing (conditional) — `consumer.py:L273-L274`                      |
| 5     | `WORKING`  | `save_document`        | `95`               | `null`        | about to persist — `consumer.py:L294`                                     |
| 6     | `SUCCESS`  | `finished`             | `100`              | **`1`**       | done; **first event carrying `document_id`** — `consumer.py:L375`         |

Two "during-state" facts:

- **The OCR start you can see is `WORKING/parsing_document@20%`** (_observed_). It fires just before `RasterisedDocumentParser.parse(...)` is invoked (`consumer.py:L259` immediately precedes the `parse()` call — _inferred from code_).
- **`document_id` is `null` in every event until the terminal `SUCCESS`** (_observed_). This is because `_send_progress(...)` is only *passed* a `document_id` on the final call (`consumer.py:L375`); the other calls omit it, so it defaults to `null` (_inferred from code_, `consumer.py:L56-L76`). **It is not evidence that the row is absent for the whole run** — §4.4 measures the DB directly and shows the row is committed just before the 100% event.

### 4.4 "During" state — measuring the DB directly (the 95% → commit → 100% window)

To see *when the committed row actually appears* — rather than inferring it from the payload field — an independent, **read-only** SQLite poller (`poll_db.py`, §12) sampled `COUNT(*)` and `MAX(id)` every 25 ms from a separate connection while run A executed. It prints a line only when the value changes.

**Output (observed):**

```json
{"mono_s": 0.0, "wall": "20:51:52.544", "count": 0, "max_id": null}
{"mono_s": 12.637, "wall": "20:52:05.181", "count": 1, "max_id": 1}
```

Line up the wall-clock times with the stream (§4.3) and the worker log (§4.5):

| Wall clock (this run) | Event                                                    | Source        |
| --------------------- | -------------------------------------------------------- | ------------- |
| 20:51:52 – 20:52:05.1 | DB `count = 0` (through STARTING, 20%, 70%, 90%)          | poller        |
| 20:52:05.121          | `WORKING/save_document@95%`                              | stream        |
| 20:52:05.121          | worker log `Saving record to database`                   | worker log    |
| **20:52:05.181**      | **DB `count` becomes 1 (`max_id = 1`) — committed row visible** | **poller**    |
| 20:52:05.193          | `SUCCESS/finished@100%` (first event with `document_id=1`) | stream        |

So the committed row became visible to an independent connection at **20:52:05.181** — **after** the 95% `save_document` event and **~12 ms before** the terminal `finished@100%` event. This matches the source (_inferred from code_): `_store(...)` runs `Document.objects.create(...)` (`consumer.py:L398`) inside `with transaction.atomic():` (`consumer.py:L298`); the commit happens when that block exits, *after* the 95% send (`consumer.py:L294`) and *before* the final 100% send (`consumer.py:L375`). The correct statement of the "during" state is therefore: **no row exists during the OCR/parse phase (0–70%); the row is created and committed in the persistence step around 95%; the terminal 100% event follows the commit.**

### 4.5 "During" state — the worker log proves OCR is actively running

The definitive engine-call signal is the DEBUG line from logger `paperless.parsing.tesseract` (`parsers.py:L24`), emitted at `parsers.py:L260` right before `ocrmypdf.ocr(**args)` (`parsers.py:L261`). Full captured worker log for run A (`grep`-ed to the two OCR-relevant loggers):

**Command (observed):**

```bash
docker exec paperless-app bash -lc \
  'grep -E "paperless.consumer|paperless.parsing.tesseract" /tmp/pl/log/paperless.log'
```

**Output (observed) — run A slice:**

```
[2026-07-14 20:51:55,215] [INFO] [paperless.consumer] Consuming no-text-alpha.png
[2026-07-14 20:51:55,216] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-14 20:51:55,217] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-14 20:51:55,232] [DEBUG] [paperless.consumer] Parsing no-text-alpha.png...
[2026-07-14 20:51:55,349] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/pl/scratch/paperless-upload-sqseq4pf: 'dpi'
[2026-07-14 20:51:55,350] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 35 based on image width 297
[2026-07-14 20:51:55,350] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/pl/scratch/paperless-upload-sqseq4pf for compatibility with img2pdf
[2026-07-14 20:51:55,356] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-sqseq4pf', 'output_file': '/tmp/pl/scratch/paperless-ynev8grp/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-ynev8grp/sidecar.txt', 'image_dpi': 35}
[2026-07-14 20:51:56,370] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 20:51:56,371] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-14 20:51:56,371] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/pl/scratch/paperless-upload-sqseq4pf: 'dpi'
[2026-07-14 20:51:56,371] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 35 based on image width 297
[2026-07-14 20:51:56,372] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-sqseq4pf', 'output_file': '/tmp/pl/scratch/paperless-ynev8grp/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-ynev8grp/sidecar-fallback.txt', 'image_dpi': 35}
[2026-07-14 20:51:57,372] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 20:51:57,373] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/pl/scratch/paperless-upload-sqseq4pf, the content will be empty.
[2026-07-14 20:51:57,373] [DEBUG] [paperless.consumer] Generating thumbnail for no-text-alpha.png...
[2026-07-14 20:51:57,948] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/pl/scratch/paperless-ynev8grp/convert.png -out /tmp/pl/scratch/paperless-ynev8grp/thumb_optipng.png
[2026-07-14 20:52:05,121] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-14 20:52:05,140] [DEBUG] [paperless.consumer] Deleting file /tmp/pl/scratch/paperless-upload-sqseq4pf
[2026-07-14 20:52:05,166] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/pl/scratch/paperless-ynev8grp
[2026-07-14 20:52:05,167] [INFO] [paperless.consumer] Document 2026-07-14 no-text-alpha consumption finished
```

Concrete OCR signals visible here (all **_observed_**):

- **`Calling OCRmyPDF with args: {…}`** is the engine-call signal at 20:51:55,356 — ~124 ms after the 20% checkpoint (20:51:55.233). The arg dict reflects the resolved settings: `'skip_text': True` (from `OCR_MODE='skip'`, built at `parsers.py:L157-L158`), `'language': 'eng'`, `'output_type': 'pdfa'`, `'clean': True`.
- For this alpha PNG the DPI probe failed, so paperless fell back to an A4-based estimate (`Estimated DPI 35 based on image width 297`) and stripped the alpha layer for `img2pdf` (`Removing alpha layer …` — `parsers.py:L191-L201`). This alpha strip happens on the worker's *scratch copy* (`/tmp/pl/scratch/paperless-upload-…`), not on the repository fixture.
- The first OCR pass produced no text → a **force-OCR fallback** (`Fallback: Calling OCRmyPDF with args: {… 'force_ocr': True …}`); it also produced no text → the empty-content last-resort warning. (This empty-text outcome is the P4 case; §7.)

### 4.6 "After" state — the row now exists (with OCR-derived content)

**Command (observed) — captured before any deletion:**

```bash
docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document as D
d=D.objects.get(pk=1)
print(repr(d.title), repr(d.mime_type), repr(d.content), repr(d.checksum), repr(d.archive_filename), d.has_archive_version)
"'
```

**Output (observed):**

```
'no-text-alpha' 'image/png' '' 'a13999fabb65f07735dfcbbb001ddaa8' '0000001.pdf' True
```

After consumption a `Document` row exists (`content=text` written at `consumer.py:L398-L400`). For this text-free image the recognized `content` is empty (`''`), yet archive metadata is set (`archive_filename='0000001.pdf'`, `has_archive_version=True`).

### 4.7 Stability across two complete runs (what is stable vs. what changes)

Run B consumed the **same input** again. Because storing the row a second time would collide on the unique `checksum` column (`src/documents/models.py:L135-L139`), row id 1 was first deleted via the ORM so the identical input could be re-consumed cleanly:

```bash
docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document as D; D.objects.get(pk=1).delete()"'
```

Run B's complete stream (**_observed_**):

```json
{"event": "subscriber_ready", "group": "status_updates"}
{"recv_wall": "2026-07-14 20:53:12.262", "recv_mono_s": 1.873, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 20:53:12.285", "recv_mono_s": 1.896, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:53:14.650", "recv_mono_s": 4.261, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}}
{"recv_wall": "2026-07-14 20:53:22.839", "recv_mono_s": 12.45, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}}
{"recv_wall": "2026-07-14 20:53:22.858", "recv_mono_s": 12.469, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:53:22.922", "recv_mono_s": 12.533, "payload": {"filename": "no-text-alpha.png", "task_id": "d3faf291-727f-44f8-b38e-8d0d87d1d96c", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 2}}
```

Run B's worker log confirms the identical OCR path (key lines, **_observed_**):

```
[2026-07-14 20:53:12,264] [INFO] [paperless.consumer] Consuming no-text-alpha.png
[2026-07-14 20:53:12,409] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-mfmpd7nw', 'output_file': '/tmp/pl/scratch/paperless-d_2zjawf/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-d_2zjawf/sidecar.txt', 'image_dpi': 35}
[2026-07-14 20:53:13,605] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-14 20:53:14,626] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/pl/scratch/paperless-upload-mfmpd7nw, the content will be empty.
[2026-07-14 20:53:22,906] [INFO] [paperless.consumer] Document 2026-07-14 no-text-alpha consumption finished
```

**What is stable vs. what changes across runs A and B (_observed_):**

- **Stable:** the *ordered* `(status, message, current_progress)` sequence — `STARTING/new_file/0 → WORKING/parsing_document/20 → WORKING/generating_thumbnail/70 → WORKING/parse_date/90 → WORKING/save_document/95 → SUCCESS/finished/100` — was byte-for-byte identical in both runs, as was the OCR path in the worker log.
- **Changes every run:** the `task_id` (`fdf2508d-…` vs `d3faf291-…`), the wall-clock timestamps, the per-step durations (`recv_mono_s`: e.g. the 70% step at 4.112 s vs 4.261 s; the 100% step at 11.916 s vs 12.533 s), the worker's scratch temp paths (`paperless-upload-sqseq4pf` vs `-mfmpd7nw`), and the terminal `document_id` (auto-increment PK `1` then `2`). Only the *ordering* is a stable claim; the identifiers and timings are not.

**A note on the alpha PNG's stored checksum (_observed_).** The stored `checksum` is **not** the md5 of the file you upload, because the parser rewrites the working copy to strip the alpha layer before storing:

```bash
docker exec paperless-app bash -lc '
  echo "original file md5 (RGBA, as uploaded): $(md5sum /app/src/paperless_tesseract/tests/samples/no-text-alpha.png | cut -d" " -f1)"
  cd /app/src && python3 manage.py shell -c "from documents.models import Document as D; print(\"stored checksum (RGB, after alpha removal):\", D.objects.get(pk=2).checksum)"'
```

```
original file md5 (RGBA, as uploaded): e8c17675174950020835add3f444f08c
stored checksum (RGB, after alpha removal): a13999fabb65f07735dfcbbb001ddaa8
```

So re-uploading the alpha PNG is not blocked by the duplicate *pre-check* (which hashes the uploaded original bytes — `consumer.py:L102-L104`, called at `L213` — a hash that never matches the stored RGB hash); the collision that would actually stop a second store is the unique `checksum` constraint on the row (`models.py:L139`). Deleting the row first (as above) sidesteps both.

### 4.8 How a user actually watches this (task id + WebSocket), practically

The browser front-end receives progress over a WebSocket to `ws/status/`, served by `StatusConsumer` (`src/paperless/consumers.py`) under `AuthMiddlewareStack` (`src/paperless/asgi.py:L20`). `StatusConsumer` **requires an authenticated session** — it raises `DenyConnection` otherwise (`consumers.py:L13-L15`) — and forwards each broadcast's `event["data"]` verbatim (`consumers.py:L29-L33`).

> **Runtime detail (_observed_).** With `PAPERLESS_DEBUG` unset (the default), `"channels"` is *not* added to `INSTALLED_APPS` (`src/paperless/settings.py:L113-L114`), so `manage.py runserver` runs the plain WSGI dev server and does **not** serve `ws/status/` (a plain HTTP GET to it `302`-redirects to the login page). WebSockets are served by an ASGI server; for this observation `daphne` was run against the same `paperless.asgi:application` the production image uses:
>
> ```bash
> docker exec -d paperless-app bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings exec daphne -b 0.0.0.0 -p 8001 paperless.asgi:application > /tmp/rt_daphne.log 2>&1'
> ```

**Unauthenticated connection is rejected (_observed_)** — confirms the auth gate:

```bash
docker exec paperless-app bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
  PYTHONPATH=/app/src python3 /tmp/obs/ws_probe.py ws://localhost:8001/ws/status/ noauth'
```

```
UNAUTH: connection REJECTED -> InvalidStatusCode: server rejected WebSocket connection: HTTP 403
```

**Authenticated connection (browser-equivalent) receives the live stream (_observed_).** `ws_probe.py` (§12) mints a real DB session for `admin`, connects with that `sessionid` cookie, and prints each frame; meanwhile `simple.jpg` was uploaded to drive the worker. The frames are the raw 7-key payloads `StatusConsumer` forwards (`consumers.py:L33`):

```
AUTH: connection ACCEPTED
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
WS_FRAME {"filename": "simple.jpg", "task_id": "7176f6c6-de76-4e95-8b8c-21e95685454a", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 5}
```

**Practical guidance, grounded in the above:**

- Because the POST returns only `"OK"` (no task id — §4.1), correlate by watching the stream: each frame carries `task_id` + `filename`. A browser simply displays every `status_updates` event; a script filters by the `filename` it just uploaded.
- For a **headless** operator without a browser session, the two reliable signals are **(a)** the `paperless.parsing.tesseract` worker log (the `Calling OCRmyPDF with args:` line is the engine-call marker), and **(b)** the channel-layer subscriber (`sub.py`) — but note that (b) joins the group directly on Redis and is **privileged** (it bypasses the `StatusConsumer` auth gate), so it is an instrumentation technique, not a normal client path.

---

## 5. P2 — Does an image that already contains visible text skip OCR? (`simple.png`)

**Input:** `src/paperless_tesseract/tests/samples/simple.png` — a raster image that visibly contains the text "This is a test document." Compare with the text-free `no-text-alpha.png` from P1.

### 5.1 Observed: the image enters the exact same OCR pipeline (no skip)

**Command (observed):**

```bash
docker exec paperless-app bash -lc "curl -s -w 'HTTP %{http_code} in %{time_total}s\n' \
  -H \"Authorization: Token $TOKEN\" \
  -F document=@/app/src/paperless_tesseract/tests/samples/simple.png \
  http://localhost:8000/api/documents/post_document/"
```

**Output (observed):** `"OK"HTTP 200 in 0.091726s` — same async behavior as P1.

Full `"status_updates"` stream (task_id `97963242-822f-44ec-901a-8ccb6dd78326`) — **identical shape to the text-free case** (**_observed_**):

```json
{"event": "subscriber_ready", "group": "status_updates"}
{"recv_wall": "2026-07-14 20:54:09.553", "recv_mono_s": 1.945, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 20:54:09.585", "recv_mono_s": 1.978, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:54:10.859", "recv_mono_s": 3.251, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}}
{"recv_wall": "2026-07-14 20:54:11.635", "recv_mono_s": 4.028, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}}
{"recv_wall": "2026-07-14 20:54:11.651", "recv_mono_s": 4.044, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:54:11.723", "recv_mono_s": 4.116, "payload": {"filename": "simple.png", "task_id": "97963242-822f-44ec-901a-8ccb6dd78326", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 3}}
```

The `WORKING/parsing_document@20%` checkpoint fires exactly as before, so **an image that already shows text does not skip OCR** — it traverses the identical pipeline. The worker log confirms `ocrmypdf` was called with the same signature (**_observed_**):

```
[2026-07-14 20:54:09,556] [INFO] [paperless.consumer] Consuming simple.png
[2026-07-14 20:54:09,557] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-14 20:54:09,558] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-14 20:54:09,585] [DEBUG] [paperless.consumer] Parsing simple.png...
[2026-07-14 20:54:09,709] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 62 based on image width 517
[2026-07-14 20:54:09,709] [DEBUG] [paperless.parsing.tesseract] Detected DPI for image /tmp/pl/scratch/paperless-upload-ius63oxb: 72
[2026-07-14 20:54:09,709] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-ius63oxb', 'output_file': '/tmp/pl/scratch/paperless-ymif33_v/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-ymif33_v/sidecar.txt', 'image_dpi': 72}
[2026-07-14 20:54:10,841] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 20:54:10,842] [DEBUG] [paperless.consumer] Generating thumbnail for simple.png...
[2026-07-14 20:54:11,109] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/pl/scratch/paperless-ymif33_v/convert.png -out /tmp/pl/scratch/paperless-ymif33_v/thumb_optipng.png
[2026-07-14 20:54:11,652] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-14 20:54:11,679] [DEBUG] [paperless.consumer] Deleting file /tmp/pl/scratch/paperless-upload-ius63oxb
[2026-07-14 20:54:11,706] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/pl/scratch/paperless-ymif33_v
[2026-07-14 20:54:11,707] [INFO] [paperless.consumer] Document 2026-07-14 simple consumption finished
```

Two differences from the text-free run — both benign, neither changing the pipeline shape:

- `simple.png` carries a DPI, so `Detected DPI … 72` is used (vs. the A4 fallback for `no-text-alpha.png`).
- OCR **found** text, so `Using text from sidecar file` is followed straight by thumbnail generation — **no** force-OCR fallback and **no** "No text was found" warning.

The resulting row: **`id=3`**, `title='simple'`, `content='This is a test document.'`, `archive_filename` set (§6).

### 5.2 Why images always OCR — _inferred from code_, corroborated by the §8 probe

This is a mechanism (why), so it is labeled **_inferred from code_**. In `RasterisedDocumentParser.parse(...)`:

- For a **PDF**, the parser first extracts any embedded text and decides "born-digital" by length: `text_original = self.extract_text(None, document_path)` (`parsers.py:L235`) and `original_has_text = text_original and len(text_original) > 50` (`parsers.py:L236`).
- For a **non-PDF image**, the `else` branch sets `text_original = None` (`parsers.py:L238`) and **`original_has_text = False`** (`parsers.py:L239`) — **unconditionally, regardless of any text visible in the raster.**
- The only early-exit that skips `ocrmypdf` is gated on that flag: `if settings.OCR_MODE == "skip_noarchive" and original_has_text:` (`parsers.py:L241`) → log `Document has text, skipping OCRmyPDF entirely.` (`parsers.py:L242`) → `return` (`parsers.py:L244`).

Because images force `original_has_text = False`, **this early-exit can never fire for an image**, even under `skip_noarchive`. The behavioral consequence is **_observed_** directly in §8 case (c).

### 5.3 How to tell the difference after processing — the sidecar marker

Since the pipeline shape is identical, **you cannot tell "the image had visible text" from the output shape.** The distinguishing artifact lives inside ocrmypdf's **sidecar text file**, which `extract_text(...)` inspects: it treats the text as usable only when `"[OCR skipped on page" not in text` (`parsers.py:L104`), logging `Using text from sidecar file` (`parsers.py:L107`) when the marker is absent versus `Incomplete sidecar file: discarding.` (`parsers.py:L110`) when it is present.

The canonical async path deletes the sidecar on cleanup, so a **_component-level probe (NON-CANONICAL)_** (`probe_sidecar.py`, §12) called the real parser on a **private `/tmp` copy** of `simple.png` and read the sidecar before cleanup.

**Command (observed):**

```bash
docker exec paperless-app bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
  PYTHONPATH=/app/src python3 /tmp/obs/probe_sidecar.py \
  /app/src/paperless_tesseract/tests/samples/simple.png image/png'
```

**Output (observed):**

```
======================================================================
INPUT: simple.png  (mime=image/png)
      LOG DEBUG paperless.parsing.tesseract: Estimated DPI 62 based on image width 517
      LOG DEBUG paperless.parsing.tesseract: Detected DPI for image /tmp/probe-in-zltlyks7/simple.png: 72
      LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/probe-in-zltlyks7/simple.png', 'output_file': '/tmp/pl/scratch/paperless-n1jpj_w3/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-n1jpj_w3/sidecar.txt', 'image_dpi': 72}
[2026-07-14 20:55:51,479] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
      LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
  parsed text repr : 'This is a test document.'
  archive produced : True
  sidecar.txt exists : True
    content repr : 'This is a test document.\n'
    contains '[OCR skipped on page' marker : False
      LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pl/scratch/paperless-n1jpj_w3
```

The input path is `/tmp/probe-in-zltlyks7/simple.png` (a private copy — the repository fixture was not parsed). For an **image**, the sidecar holds OCR-produced text with **no** `[OCR skipped on page(s) …]` marker, so `extract_text` takes the `Using text from sidecar file` branch — i.e., **the text is OCR-generated, not pre-existing.** (The `[ERROR] [ocrmypdf._exec.tesseract] … Error during processing.` line is emitted by ocrmypdf's *own* logger, not paperless's; it is non-fatal — text was still recovered, as the following `Using text from sidecar file` and `parsed text` lines show. The canonical §5.1 worker run of the same file did not surface it.) The contrasting text-layer-PDF case, where the marker *is* present, is in §8.

---

## 6. P3 — Comparing the final API responses

**Goal:** compare `GET /api/documents/{id}/` for the text-free image (P1, `id=2`) and the image-with-text (P2, `id=3`) to see which fields carry OCR-generated vs. pre-existing text.

**Command (observed):**

```bash
docker exec paperless-app bash -lc "curl -s -H \"Authorization: Token $TOKEN\" http://localhost:8000/api/documents/2/ | python3 -m json.tool"
docker exec paperless-app bash -lc "curl -s -H \"Authorization: Token $TOKEN\" http://localhost:8000/api/documents/3/ | python3 -m json.tool"
```

**`GET /api/documents/2/` — text-free image (full, unedited body):**

```json
{
    "id": 2,
    "correspondent": null,
    "document_type": null,
    "title": "no-text-alpha",
    "content": "",
    "tags": [],
    "created": "2026-07-14T20:53:12.408744Z",
    "modified": "2026-07-14T20:53:22.879291Z",
    "added": "2026-07-14T20:53:22.859954Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-14 no-text-alpha.png",
    "archived_file_name": "2026-07-14 no-text-alpha.pdf"
}
```

**`GET /api/documents/3/` — image with visible text (full, unedited body):**

```json
{
    "id": 3,
    "correspondent": null,
    "document_type": null,
    "title": "simple",
    "content": "This is a test document.",
    "tags": [],
    "created": "2026-07-14T20:54:09Z",
    "modified": "2026-07-14T20:54:11.678279Z",
    "added": "2026-07-14T20:54:11.653276Z",
    "archive_serial_number": null,
    "original_file_name": "2026-07-14 simple.png",
    "archived_file_name": "2026-07-14 simple.pdf"
}
```

### 6.1 Field-by-field comparison (programmatic)

**Command (observed) — fetches both and compares:**

```bash
docker exec paperless-app bash -lc "cd /app/src && TOK=$TOKEN python3 -c \"
import os, json, urllib.request
tok=os.environ['TOK']
def get(i):
    r=urllib.request.Request(f'http://localhost:8000/api/documents/{i}/', headers={'Authorization':'Token '+tok})
    return json.load(urllib.request.urlopen(r))
a,b=get(2),get(3)
print('doc2 keys == doc3 keys :', sorted(a)==sorted(b))
print('number of keys        :', len(a))
tp=[k for k in a if ('ocr' in k.lower()) or ('provenance' in k.lower()) or (k.lower() in ('is_ocr','ocr_status','text_source','processed','status'))]
print('fields naming TEXT provenance / OCR status :', tp)
print('*_file_name fields (about files, not text)  :', [k for k in a if k.endswith('file_name')])
\""
```

**Output (observed):**

```
doc2 keys == doc3 keys : True
number of keys        : 12
fields naming TEXT provenance / OCR status : []
*_file_name fields (about files, not text)  : ['original_file_name', 'archived_file_name']
```

| Field                                                             | doc 2 (text-free image)          | doc 3 (image with text)      | Notes / citation                                                                                                                                                                                                              |
| ----------------------------------------------------------------- | -------------------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                                                         | `""` (empty)                     | `"This is a test document."` | **The single field that carries recognized text** — model `content` TextField (`models.py:L117`); serialized at `serialisers.py:L227`. In both cases the text (or its absence) is **OCR-derived**, since both inputs are images. |
| `archived_file_name`                                              | `"2026-07-14 no-text-alpha.pdf"` | `"2026-07-14 simple.pdf"`    | **Populated in BOTH.** `SerializerMethodField` (`serialisers.py:L208`); `get_archived_file_name` returns the archive public name when `obj.has_archive_version` else `None` (`serialisers.py:L213-L217`); `has_archive_version` is defined as `archive_filename is not None` (`models.py:L238-L239`). See the qualification in §6.2(2). |
| `original_file_name`                                              | `"2026-07-14 no-text-alpha.png"` | `"2026-07-14 simple.png"`    | the stored original file's public name — `serialisers.py:L207,L233` (about the *file*, not text provenance)                                                                                                                    |
| `id`, `title`, `created`, `modified`, `added`                     | differ trivially                 | differ trivially             | per-document values                                                                                                                                                                                                          |
| `correspondent`, `document_type`, `tags`, `archive_serial_number` | `null` / `[]`                    | `null` / `[]`                | unset in both                                                                                                                                                                                                                |

### 6.2 P3 conclusions (answering "which fields show OCR-generated vs existing text")

1. **There is exactly one field for recognized text: `content`** (_observed_: both bodies expose it). Both the OCR-generated text (doc 3) and the empty result (doc 2) land there (`models.py:L117` → `serialisers.py:L227`). For images the text is _always_ OCR-generated; there is **no separate "existing text" field**.
2. **`archived_file_name` is non-null in both** (_observed_). Precisely (_inferred from code_): the getter returns a name exactly when `has_archive_version` is true, and `has_archive_version` is defined as `archive_filename is not None` (`models.py:L238-L239`). So a non-null value means **archive *metadata* is set on the row** for these successful image consumes — it does **not** independently prove the physical archive file currently exists on disk, nor does it label text provenance.
3. **No field labels provenance** (_observed_: the programmatic check returned `[]` for any OCR/provenance/status field name; the two `*_file_name` fields name files, not text). Nothing in the response says "this text came from OCR" vs. "this text pre-existed" — the two documents are indistinguishable in response shape. The only place that distinction is ever visible is the transient ocrmypdf sidecar marker (§5.3 / §8), which the API does not surface.

---

## 7. P4 — Weak/incomplete OCR: what happens to the final state?

**Input driving the weak/empty case:** the text-free `no-text-alpha.png` — the **same run A (document id 1)** captured in §4. Tesseract recovered no usable text from it even after a force-OCR fallback, so it is the natural "weak/incomplete OCR" specimen. The before/during/after below all come from that single run (no mixing across runs).

### 7.1 Observed: empty OCR result → empty `content`, but the document is still fully consumed

The relevant last-resort logic in `parse(...)` (_inferred from code_): after OCR, if `not self.text` (`parsers.py:L318`), then if the original had text it reuses that (`parsers.py:L319-L320`); **otherwise** it logs `No text was found in {document_path}, the content will be empty.` (`parsers.py:L322-L326`) and sets `self.text = ""` (`parsers.py:L327`). `parse()` then **returns normally** — an empty OCR result is not an error.

The warning was captured live in the run-A worker log (§4.5), **_observed_**:

```
[2026-07-14 20:51:57,373] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/pl/scratch/paperless-upload-sqseq4pf, the content will be empty.
```

And the same run still reached the terminal success event (from the run-A stream, §4.3), **_observed_**:

```json
{"recv_wall": "2026-07-14 20:52:05.193", "recv_mono_s": 11.916, "payload": {"filename": "no-text-alpha.png", "task_id": "fdf2508d-b07f-4759-8bbc-b6117d886bab", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

**Before/during/after for the weak-OCR case — all from run A (all _observed_):**

- **Before:** `Document.objects.count()` = `0` — no row (§4.2).
- **During:** the full progress sequence fires (`STARTING`→…→`SUCCESS`), with `document_id=null` throughout until the end (§4.3); the worker logs the empty-content warning mid-parse (above); and the DB poll shows the row committed only near 95% (§4.4).
- **After:** a normal row exists — `id=1`, `content=''`, `archive_filename='0000001.pdf'`, `has_archive_version=True` (§4.6).

**Answer:** Yes — a weak/incomplete (even fully empty) OCR result **still counts as fully processed.** It is reflected in saved metadata as `content=""` on an otherwise-normal row **with archive metadata set**; there is no error marker and no "partial" flag.

### 7.2 "Fully processed" has no status column — observed via model introspection

There is no dedicated OCR-status/"processed" column on the `Document` model (`models.py:L88-L210`). Confirmed live:

**Command (observed):**

```bash
docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
print([f.name for f in Document._meta.get_fields() if getattr(f,\"concrete\",False)])"'
```

**Output (observed):**

```
['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum',
 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename',
 'archive_filename', 'archive_serial_number', 'tags']
```

None of these is a `status`/`processed`/`ocr`/`state`-like field. So "fully processed" is **not** a stored attribute — it is signalled only by **(a) the row existing** and **(b) the terminal `SUCCESS/finished@100%` progress event** (`consumer.py:L375`).

### 7.3 Contrast — a hard failure produces `FAILED@100%` and no row (observed)

An empty-OCR result must be distinguished from a **hard parse failure**. To exercise the failure path, a deliberately corrupt PDF (valid `%PDF` header + garbage body) was created in a secure temp file and uploaded through the canonical path.

**Command (observed) — create the corrupt input securely:**

```bash
docker exec paperless-app bash -lc '
set -eu
work=$(mktemp -d); trap "rm -rf \"$work\"" EXIT
cp=$work/corrupt.pdf
printf "%%PDF-1.4\nthis is not a valid pdf body %s\n" "$(head -c 64 /dev/urandom | base64)" > "$cp"
echo "libmagic mime: $(file -b --mime-type "$cp")"
echo "size: $(stat -c%s "$cp") bytes"
cp "$cp" /tmp/obs/corrupt.pdf'
```

**Output (observed):**

```
libmagic mime: application/pdf
size: 128 bytes
```

**Upload + result (observed):** before `Document.objects.count()` = `2`.

```bash
docker exec paperless-app bash -lc "curl -s -w 'HTTP %{http_code}\n' -H \"Authorization: Token $TOKEN\" \
  -F document=@/tmp/obs/corrupt.pdf http://localhost:8000/api/documents/post_document/"
```

Captured `"status_updates"` stream (**_observed_**):

```json
{"event": "subscriber_ready", "group": "status_updates"}
{"recv_wall": "2026-07-14 20:55:28.799", "recv_mono_s": 1.91, "payload": {"filename": "corrupt.pdf", "task_id": "31d22d96-d611-4e52-b2d2-0fbeb2c19eb4", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 20:55:28.819", "recv_mono_s": 1.93, "payload": {"filename": "corrupt.pdf", "task_id": "31d22d96-d611-4e52-b2d2-0fbeb2c19eb4", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 20:55:29.218", "recv_mono_s": 2.329, "payload": {"filename": "corrupt.pdf", "task_id": "31d22d96-d611-4e52-b2d2-0fbeb2c19eb4", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "InputFileError: ", "document_id": null}}
```

After: `Document.objects.count()` = `2` (**unchanged — no row created**). The worker log (**_observed_**):

```
[2026-07-14 20:55:28,802] [INFO] [paperless.consumer] Consuming corrupt.pdf
[2026-07-14 20:55:29,218] [ERROR] [paperless.consumer] Error while consuming document corrupt.pdf: InputFileError: 
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError
    raise InputFileError() from e
ocrmypdf.exceptions.InputFileError
    raise ParseError(f"{e.__class__.__name__}: {str(e)}")
documents.parsers.ParseError: InputFileError: 
```

This is the failure path (_inferred from code_, matching the traceback): an unrecoverable OCR error becomes a `ParseError` (`parsers.py:L310`/`L314`), caught by the consumer (`consumer.py:L278`) → `_fail(...)` (`consumer.py:L280`) which sends `FAILED@100%` (`consumer.py:L79`) and raises `ConsumerError` (`consumer.py:L81`), aborting the consume so **no `Document` is written.** Note the sequence stopped at `20 → FAILED` (no 70/90/95), because the failure occurred during `parse()`.

### 7.4 The key P4 distinction — bounded to what was observed

- **Empty/weak OCR** (`NoTextFoundException` on the first pass) does **not** abort. It triggers the force-OCR fallback and, if still empty, the empty-content last resort (`self.text = ""`, `parsers.py:L327`) and a **normal return** → a document **is** created with `content=""`. **_Observed_** (run A, §4/§7.1).
- **A genuine `ParseError`** (unrecoverable) routes through `_fail` → `FAILED@100%` and prevents document creation. **_Observed_** (corrupt PDF, §7.3).

**`ParseError` is *not* the only outcome that prevents document creation.** A second no-row outcome was **_observed_** by re-uploading an already-consumed file (`simple.png`, id 3), which the duplicate pre-check rejects (`pre_check_duplicate` → `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS)` — `consumer.py:L110-L113`; message constant at `consumer.py:L37`) — and it fails at the pre-check, *before* the 20% parsing event:

```json
{"event": "subscriber_ready", "group": "status_updates"}
{"recv_wall": "2026-07-14 21:03:52.059", "recv_mono_s": 1.811, "payload": {"filename": "simple.png", "task_id": "94de51db-ffab-4c23-9dd2-214255cf6524", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 21:03:52.076", "recv_mono_s": 1.827, "payload": {"filename": "simple.png", "task_id": "94de51db-ffab-4c23-9dd2-214255cf6524", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}}
```

```
[2026-07-14 21:03:52,075] [ERROR] [paperless.consumer] Not consuming simple.png: It is a duplicate.
```

Count was `4` before and `4` after — no row created, and this was **not** a `ParseError` (message `document_already_exists`, `status FAILED`, and the `20%` event never fired).

Beyond these two observed cases, the source has several more `_fail` paths that also prevent creation (**_inferred from code_**): a missing file (`pre_check_file_exists` → `consumer.py:L95`), an unsupported MIME type (`_fail(MESSAGE_UNSUPPORTED_TYPE)` → `consumer.py:L225`), and a failing pre-consume script (`run_pre_consume_script` → `consumer.py:L121-L141`) — all *before* `_store`; and any exception inside the storage `transaction.atomic()` block rolls the row back (`consumer.py:L362-L367`). Conversely, the post-consume script runs *after* the row is committed (`consumer.py:L371`), so a failure there would leave a row in place. So the correct, bounded statement is: **an empty OCR result returns normally and creates a document; a hard `ParseError` (observed) and a duplicate (observed) — among other `_fail` conditions — do not.**

---

## 8. Supporting evidence — the digital-PDF skip contrast (why images differ from PDFs)

This section substantiates the §5.2 claim that images can _never_ early-exit, by contrasting them with a **text-layer PDF** that _can_. It also exhibits the literal `[OCR skipped on page(s) …]` sidecar marker. Cases (b)/(c) override `OCR_MODE='skip_noarchive'` and are therefore **NON-CANONICAL**; case (a) uses the canonical default `skip`. The full script is `probe_skip.py` (§12); it parses **private `/tmp` copies** of the fixtures.

**Command (observed):**

```bash
docker exec paperless-app bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
  PYTHONPATH=/app/src python3 /tmp/obs/probe_skip.py'
```

**Output (observed) — complete, unedited:**

```
========================================================================
INPUT simple-digital.pdf  mime=application/pdf  OCR_MODE='skip'  [canonical default]
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/probe-in-an7qbmno/simple-digital.pdf
      LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/probe-in-an7qbmno/simple-digital.pdf', 'output_file': '/tmp/pl/scratch/paperless-7yelc_4_/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-7yelc_4_/sidecar.txt'}
      LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pl/scratch/paperless-7yelc_4_/archive.pdf
   -> parsed text repr : 'This is a test document.'
   -> archive produced : True  (archive_path=/tmp/pl/scratch/paperless-7yelc_4_/archive.pdf)
   -> sidecar.txt: contains '[OCR skipped on page' = True  repr='[OCR skipped on page(s) 1]'
      LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pl/scratch/paperless-7yelc_4_
========================================================================
INPUT multi-page-digital.pdf  mime=application/pdf  OCR_MODE='skip_noarchive'  [NON-CANONICAL]
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/probe-in-o02h8czd/multi-page-digital.pdf
      LOG DEBUG paperless.parsing.tesseract: Document has text, skipping OCRmyPDF entirely.
   -> parsed text repr : 'This is a multi page document. Page 1.\n\nThis is a multi page document. Page 2.\n\nThis is a multi page document. Page 3.'
   -> archive produced : False  (archive_path=None)
      LOG DEBUG paperless.parsing.tesseract: Deleting directory /tmp/pl/scratch/paperless-vefg38ul
========================================================================
INPUT simple.png  mime=image/png  OCR_MODE='skip_noarchive'  [NON-CANONICAL]
      LOG DEBUG paperless.parsing.tesseract: Estimated DPI 62 based on image width 517
      LOG DEBUG paperless.parsing.tesseract: Detected DPI for image /tmp/probe-in-i45t_z6b/simple.png: 72
      LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '/tmp/probe-in-i45t_z6b/simple.png', 'output_file': '/tmp/pl/scratch/paperless-lhs619fr/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-lhs619fr/sidecar.txt', 'image_dpi': 72}
[2026-07-14 20:56:11,113] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
      LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
   -> parsed text repr : 'This is a test document.'
   -> archive produced : True  (archive_path=/tmp/pl/scratch/paperless-lhs619fr/archive.pdf)
   -> sidecar.txt: contains '[OCR skipped on page' = False  repr='This is a test document.\n'
```

What this proves (all **_observed_**):

- **(a)** For a text-layer PDF under the canonical `skip`, ocrmypdf **is still called** (`skip_text: True`), and the sidecar literally contains `'[OCR skipped on page(s) 1]'` — the exact marker `extract_text` branches on (`parsers.py:L104`) — so it logs `Incomplete sidecar file: discarding.` (`parsers.py:L110`) and takes the text from the PDF layer instead.
- **(b)** A text-layer PDF (pdfminer length > 50 ⇒ `original_has_text=True`) under `skip_noarchive` hits the early-exit: `Document has text, skipping OCRmyPDF entirely.` (`parsers.py:L242`) → **no archive** (`archive_path=None`), no ocrmypdf call.
- **(c)** The **same** `skip_noarchive` mode applied to an **image** does **not** early-exit — ocrmypdf still runs and an archive is produced — because the image branch forced `original_has_text=False` (`parsers.py:L238-L239`). Its sidecar has **no** skip marker, confirming the text is OCR-generated.

**Behavioral oracle (`src/paperless_tesseract/tests/test_parser.py`, _inferred from code_)** agrees: `test_skip_noarchive_withtext` (`multi-page-digital.pdf`) asserts `archive_path is None`; `test_skip_noarchive_notext` (`@override_settings OCR_MODE="skip_noarchive"`, image-based) asserts the archive file exists; `test_image_simple` parses `simple.png` through the OCR path.

---

## 9. Coverage / Checklist Pass

Every sub-question and every named item is addressed, with its evidence anchor and `file:line`.

| Sub-question / named item                                      | Answer (one line)                                                                                                                      | Evidence anchor                  | Primary `file:line`                                               |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | ----------------------------------------------------------------- |
| **P1** — how to _see_ OCR started (text-free image)            | Watch the worker: `WORKING/parsing_document@20%` (phase entry) + the `Calling OCRmyPDF with args:` DEBUG log (engine call)             | §4.3, §4.5 (full payloads + log) | `consumer.py:L259`; `parsers.py:L260`                             |
| P1 — what the **processing state** looks like                  | Observed successful sequence 0→20→70→90→95→100; `document_id` null in payloads; committed row appears ~95% (DB poll)                    | §4.3, §4.4 (table + poll)        | `consumer.py:L202,L259,L264,L274,L294,L298,L375`                  |
| P1 — **background-worker** behavior / active-OCR **signals**   | HTTP returns `"OK"` (~0.03 s, task id NOT returned) then a django-q worker runs OCR; signals = `"status_updates"` events + tesseract logs | §4.1, §4.3, §4.5, §4.8           | `views.py:L521-L535`; `tasks.py:L184,L236`; `consumer.py:L56-L76` |
| P1 — before / during / after                                   | before: 0 rows; during: events fire, row committed near 95%; after: row with (empty) OCR content + archive                             | §4.2, §4.3, §4.4, §4.6           | `consumer.py:L298,L398,L375`                                      |
| P1 — stability across ≥2 runs                                  | Two full runs; ordered status/message/progress identical; task_id, timestamps, durations, terminal `document_id` all differ           | §4.7                             | `consumer.py:L56-L76`; `models.py:L139` (dedup)                   |
| P1 — practical WebSocket observation                           | Authenticated `ws/status/` (StatusConsumer, auth-gated) receives the stream; unauth → 403                                             | §4.8                             | `consumers.py:L13-L15,L29-L33`; `urls.py:L137`; `asgi.py:L20`     |
| **P2** — does an image with text skip OCR?                     | **No** — images always OCR; identical pipeline shape (observed); reason is `original_has_text=False` (inferred)                        | §5.1 (stream + log), §5.2        | `parsers.py:L238-L239,L241-L244`                                  |
| P2 — how to tell the difference afterward                      | Only via the ocrmypdf sidecar marker `[OCR skipped on page(s) …]`; absent for images                                                   | §5.3, §8(a)                      | `parsers.py:L104,L107,L110`                                       |
| **P3** — which fields carry OCR-generated vs pre-existing text | Both use the single `content` field; both expose `archived_file_name`; **no provenance field** (observed)                             | §6 (both JSON + programmatic)    | `serialisers.py:L227,L213-L217`; `models.py:L117,L238-L239`       |
| the **`content`** field                                        | single field for recognized text (OCR or pre-existing)                                                                                 | §6.1                             | `models.py:L117` → `serialisers.py:L227`                          |
| **`archived_file_name`**                                       | non-null ⇔ `archive_filename is not None` (archive metadata set) for these successful image consumes                                   | §6.1, §6.2(2)                    | `serialisers.py:L213-L217`; `models.py:L238-L239`                 |
| **P4** — weak/incomplete OCR final state                       | Empty text → `content=""`; document **still fully consumed**, `SUCCESS@100%`                                                           | §7.1 (warning + SUCCESS)         | `parsers.py:L322-L327`; `consumer.py:L375`                        |
| P4 — does it count as "**fully processed**"?                   | **Yes**; signalled by the row existing + terminal `finished@100%`                                                                      | §7.1, §7.2                       | `consumer.py:L375`                                                |
| P4 — how reflected in **saved metadata**                       | `content=""` on a normal row (+ archive metadata); **no status/"processed" column**                                                    | §7.2 (field list)                | `models.py:L88-L210`                                              |
| P4 — what prevents document creation                           | Hard `ParseError` (observed) and duplicate (observed) → `FAILED@100%`, no row; other `_fail` paths too (inferred); empty OCR does not  | §7.3, §7.4                       | `parsers.py:L310,L314`; `consumer.py:L79,L210-L221,L280`          |
| Supporting — PDF vs image skip contrast                        | text-layer PDF _can_ early-exit under `skip_noarchive`; images cannot                                                                   | §8(a)(b)(c)                      | `parsers.py:L241-L244`                                            |

**Observed vs. inferred discipline.** Every payload/log/JSON/DB-read/probe block above is **_observed at runtime_**. The statements labeled **_inferred from code_** are the *mechanistic explanations* (the "why"): the pipeline map in §3; the `original_has_text=False` reason in §5.2; the transaction-commit ordering rationale in §4.4; the enumeration of additional `_fail` paths in §7.4; and the `test_parser.py` oracle in §8. Each such inference is grounded in a `file:line` citation and, where a behavioral consequence exists, that consequence is separately **_observed_** (e.g., §5.2's consequence is observed in §8(c); §4.4's commit ordering is observed by the DB poll).

---

## 10. External corroboration (web research)

The runtime findings were cross-checked against version-appropriate external sources. Runtime and repository evidence above is primary; these links corroborate the mechanism.

- **ocrmypdf sidecar marker** — pinned to the exact `ocrmypdf==13.4.3` source. The literal marker paperless branches on is written by ocrmypdf when a page's OCR is skipped: see [`OcrGraft`/sidecar handling in OCRmyPDF v13.4.3](https://github.com/ocrmypdf/OCRmyPDF/blob/v13.4.3/src/ocrmypdf/_pipeline.py). This is the `[OCR skipped on page(s) …]` string tested at [`parsers.py:L104` @ commit 542221a](https://github.com/paperless-ngx/paperless-ngx/blob/542221a38dff06361e07976452f9aea24d210542/src/paperless_tesseract/parsers.py#L104), and it matches the observed `'[OCR skipped on page(s) 1]'` in §8(a).
- **paperless-ngx OCR-mode semantics** — pinned to the target commit (the *current* published docs have dropped the historical `skip_noarchive` wording, so the commit-pinned source is authoritative here): [`docs/configuration.rst` @ commit 542221a](https://github.com/paperless-ngx/paperless-ngx/blob/542221a38dff06361e07976452f9aea24d210542/docs/configuration.rst) documents that the default `skip` performs OCR only when needed and always creates an archive, while `skip_noarchive` additionally skips creating an archive when text already exists — corroborating [`settings.py:L522` @ 542221a](https://github.com/paperless-ngx/paperless-ngx/blob/542221a38dff06361e07976452f9aea24d210542/src/paperless/settings.py#L522).
- **Default `skip` still calls OCRmyPDF** — corroborated directly by the commit-pinned parser: the early-exit is gated *only* on `skip_noarchive` at [`parsers.py:L241` @ 542221a](https://github.com/paperless-ngx/paperless-ngx/blob/542221a38dff06361e07976452f9aea24d210542/src/paperless_tesseract/parsers.py#L241), so the default `skip` does not bypass ocrmypdf — matching the P2/§8(a) observation.

---

## 11. Repository integrity note

The investigation was **read-only** with respect to the source tree. All observation scaffolding (scripts, scratch media, the SQLite DB, media/consume dirs) lives **outside** the repository — under `/tmp` on the host and inside the disposable containers — and none of it is committed. Repository integrity is proven in this section (§11).

**Repository fixtures were never modified (_observed_).** The canonical upload path never touches a fixture (the server copies the uploaded bytes into its own `SCRATCH_DIR` temp file — `views.py:L512-L518` — and the alpha-strip rewrites *that* scratch copy, not the fixture), and every component probe parses a private `/tmp` copy. Verified after all runs and probes, inside the observation container's `/app` checkout:

```bash
docker exec paperless-app bash -lc 'cd /app && git status --porcelain'   # (empty output)
docker exec paperless-app bash -lc 'cd /app && md5sum \
  src/paperless_tesseract/tests/samples/no-text-alpha.png \
  src/paperless_tesseract/tests/samples/simple.png \
  src/paperless_tesseract/tests/samples/simple-digital.pdf \
  src/paperless_tesseract/tests/samples/multi-page-digital.pdf'
```

```
e8c17675174950020835add3f444f08c  src/paperless_tesseract/tests/samples/no-text-alpha.png
249d1239dc39449c856dcdfbb75850c5  src/paperless_tesseract/tests/samples/simple.png
42995833e01aea9b3edee44bbfdd7ce1  src/paperless_tesseract/tests/samples/simple-digital.pdf
9c9691e51741c1f4f41a20896af31770  src/paperless_tesseract/tests/samples/multi-page-digital.pdf
```

The `git status` inside `/app` is empty and the fixture md5s are identical to their pre-run values — so **no source file (including no test fixture) was modified** in the observation environment.

**The destination repository working tree (_observed_).**

```bash
$ git rev-parse --abbrev-ref HEAD
blitzy-2d7c5fb9-6420-4f89-9285-98269fdc35bb

$ git status --porcelain
 M blitzy/documentation/paperless-ngx_542221a38dff.md
```

The only change is this single deliverable document. No existing source file was modified, created, or deleted — satisfying the hard read-only constraint.

**Temporary-artifact cleanup (_observed_).** After the investigation, all observation infrastructure was removed and the removal verified — the disposable containers, the private Docker network, **and the anonymous Redis data volume** (so no `dump.rdb` remnant is left behind):

```bash
docker rm -f paperless-app paperless-redis
docker network rm paperless-net
docker volume prune -f          # removes the anonymous redis data volume
# verification (all three should show no project artifacts):
docker ps -a         --filter name=paperless   # (no rows)
docker network ls    --filter name=paperless-net # (no rows)
docker volume ls -q                              # (project volume gone)
```

The observation scripts under `/tmp/obs` and all scratch data under `/tmp/pl` are inside those disposable containers and vanish with them; nothing observation-related remains in the repository tree or in Docker state.

---

## 12. Appendix — observation scripts (published for auditability)

These are the exact scripts used above. Each uses secure temp handling (`tempfile.mkdtemp` / `mktemp -d`), passes arguments as arrays / via `-F`/`-c` rather than shell-interpolated user data, copies trusted fixtures rather than mutating them, and cleans up with `finally`/`trap`.

**`sub.py` — channel-layer subscriber (PRIVILEGED internal probe):**

```python
import sys, json, time, datetime
import django
django.setup()                                   # DJANGO_SETTINGS_MODULE=paperless.settings
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer

GROUP = sys.argv[1] if len(sys.argv) > 1 else "status_updates"
DEADLINE = time.monotonic() + (float(sys.argv[2]) if len(sys.argv) > 2 else 40.0)

layer = get_channel_layer()
channel_name = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)(GROUP, channel_name)          # same group StatusConsumer joins
print(json.dumps({"event": "subscriber_ready", "group": GROUP}), flush=True)

t0 = time.monotonic()
try:
    while time.monotonic() < DEADLINE:
        try:
            msg = async_to_sync(layer.receive)(channel_name)
        except Exception:
            break
        payload = msg.get("data", msg)
        print(json.dumps({
            "recv_wall": datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")[:-3],
            "recv_mono_s": round(time.monotonic() - t0, 3),
            "payload": payload,
        }), flush=True)
        if isinstance(payload, dict) and payload.get("status") in ("SUCCESS", "FAILED"):
            break
finally:
    async_to_sync(layer.group_discard)(GROUP, channel_name)
```

**`poll_db.py` — independent read-only DB poller (used for the 95%→commit→100% window):**

```python
import sys, json, time, sqlite3, datetime
DB = sys.argv[1]
DURATION = float(sys.argv[2]) if len(sys.argv) > 2 else 30.0
INTERVAL = (float(sys.argv[3]) if len(sys.argv) > 3 else 30.0) / 1000.0
con = sqlite3.connect(f"file:{DB}?mode=ro", uri=True, timeout=1.0)   # read-only: never blocks writer
con.execute("PRAGMA query_only = ON;")
t0 = time.monotonic(); last = None
try:
    while time.monotonic() - t0 < DURATION:
        count, max_id = con.execute("SELECT COUNT(*), MAX(id) FROM documents_document;").fetchone()
        if (count, max_id) != last:
            print(json.dumps({"mono_s": round(time.monotonic()-t0,3),
                              "wall": datetime.datetime.now().strftime("%H:%M:%S.%f")[:-3],
                              "count": count, "max_id": max_id}), flush=True)
            last = (count, max_id)
        time.sleep(INTERVAL)
finally:
    con.close()
```

**`probe_sidecar.py` — NON-CANONICAL component probe (image sidecar):**

```python
import os, sys, uuid, shutil, logging, tempfile
import django
django.setup()
from paperless_tesseract.parsers import RasterisedDocumentParser
FIXTURE, MIME = sys.argv[1], sys.argv[2]
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("      LOG %(levelname)s %(name)s: %(message)s"))
lg = logging.getLogger("paperless.parsing.tesseract"); lg.setLevel(logging.DEBUG); lg.addHandler(h)
workdir = tempfile.mkdtemp(prefix="probe-in-"); os.chmod(workdir, 0o700)   # 0700 private temp
copy_path = os.path.join(workdir, os.path.basename(FIXTURE))
shutil.copyfile(FIXTURE, copy_path)                                        # parse a COPY, never the fixture
parser = RasterisedDocumentParser(uuid.uuid4())
try:
    print("=" * 70); print(f"INPUT: {os.path.basename(FIXTURE)}  (mime={MIME})")
    parser.parse(copy_path, MIME, os.path.basename(FIXTURE))
    print(f"  parsed text repr : {parser.get_text()!r}")
    print(f"  archive produced : {bool(parser.get_archive_path())}")
    sidecar = os.path.join(parser.tempdir, "sidecar.txt")
    print(f"  sidecar.txt exists : {os.path.isfile(sidecar)}")
    if os.path.isfile(sidecar):
        content = open(sidecar).read()
        print(f"    content repr : {content!r}")
        print(f"    contains '[OCR skipped on page' marker : {'[OCR skipped on page' in content}")
finally:
    parser.cleanup(); shutil.rmtree(workdir, ignore_errors=True)          # finally cleanup
```

**`probe_skip.py` — NON-CANONICAL component probe (skip vs skip_noarchive contrast):**

```python
import os, sys, uuid, shutil, logging, tempfile
import django
django.setup()
from django.test import override_settings
from paperless_tesseract.parsers import RasterisedDocumentParser
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("      LOG %(levelname)s %(name)s: %(message)s"))
lg = logging.getLogger("paperless.parsing.tesseract"); lg.setLevel(logging.DEBUG); lg.addHandler(h)
SAMPLES = "/app/src/paperless_tesseract/tests/samples"
CASES = [("simple-digital.pdf", "application/pdf", "skip", "[canonical default]"),
         ("multi-page-digital.pdf", "application/pdf", "skip_noarchive", "[NON-CANONICAL]"),
         ("simple.png", "image/png", "skip_noarchive", "[NON-CANONICAL]")]
for fixture, mime, mode, label in CASES:
    workdir = tempfile.mkdtemp(prefix="probe-in-"); os.chmod(workdir, 0o700)
    copy_path = os.path.join(workdir, fixture)
    shutil.copyfile(os.path.join(SAMPLES, fixture), copy_path)             # parse a COPY
    parser = RasterisedDocumentParser(uuid.uuid4())
    try:
        print("=" * 72); print(f"INPUT {fixture}  mime={mime}  OCR_MODE='{mode}'  {label}")
        with override_settings(OCR_MODE=mode):
            parser.parse(copy_path, mime, fixture)
        print(f"   -> parsed text repr : {parser.get_text()!r}")
        ap = parser.get_archive_path()
        print(f"   -> archive produced : {bool(ap)}  (archive_path={ap})")
        sc = os.path.join(parser.tempdir, "sidecar.txt")
        if os.path.isfile(sc):
            c = open(sc).read()
            print(f"   -> sidecar.txt: contains '[OCR skipped on page' = {'[OCR skipped on page' in c}  repr={c!r}")
    finally:
        parser.cleanup(); shutil.rmtree(workdir, ignore_errors=True)
```

**`ws_probe.py` — authenticated WebSocket client (browser-equivalent) and unauth check:**

```python
import sys, json, asyncio
import django
django.setup()
import websockets
URL, MODE = sys.argv[1], sys.argv[2]

def mint_sessionid(username):                    # runs in sync context, before the event loop
    from django.contrib.auth import get_user_model
    from django.contrib.sessions.backends.db import SessionStore
    user = get_user_model().objects.get(username=username)
    s = SessionStore()
    s["_auth_user_id"] = str(user.pk)
    s["_auth_user_backend"] = "django.contrib.auth.backends.ModelBackend"
    s["_auth_user_hash"] = user.get_session_auth_hash()
    s.create()
    return s.session_key

async def run_noauth():
    try:
        async with websockets.connect(URL):
            print("UNAUTH: connection ACCEPTED (unexpected)")
    except Exception as e:
        print(f"UNAUTH: connection REJECTED -> {type(e).__name__}: {e}")

async def run_auth(sid, seconds):
    async with websockets.connect(URL, extra_headers=[("Cookie", f"sessionid={sid}")]) as ws:
        print("AUTH: connection ACCEPTED")
        try:
            while True:
                frame = await asyncio.wait_for(ws.recv(), timeout=seconds)
                print("WS_FRAME " + frame)
                if json.loads(frame).get("status") in ("SUCCESS", "FAILED"):
                    break
        except asyncio.TimeoutError:
            print("AUTH: idle timeout, closing")

if MODE == "noauth":
    asyncio.run(run_noauth())
else:
    asyncio.run(run_auth(mint_sessionid(sys.argv[3]), float(sys.argv[4]) if len(sys.argv) > 4 else 30.0))
```

**Corrupt-input creation (inline, secure temp):** shown in §7.3.
