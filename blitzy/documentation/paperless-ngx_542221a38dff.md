# OCR Runtime Behavior in paperless-ngx — A Run-First Investigation

> **Repository / commit:** `paperless-ngx` @ `542221a38dff06361e07976452f9aea24d210542` (branch `paperless-ngx_542221a38dff`)
> **Nature of this document:** A _run-first, evidence-grounded_ answer. Every behavioral claim below is paired with the **exact, unedited output** that was captured while actually running the OCR ingestion pipeline, together with the command that produced it and a `file:line` citation into the source tree. Statements that were **not** directly observed at runtime are explicitly labeled **_inferred from code_**.
> **Runtime used:** the canonical **Python 3.9** stack inside the provided Docker image (`tesseract` + `ocrmypdf` present). Observations were **not** produced on the planning shell (host Python 3.13.7, which lacks `tesseract`/`ocrmypdf`).

---

## 1. Answer Summary (the short version)

The four sub-questions, answered up front; each is proven in detail in the correspondingly-named section below.

- **P1 — How do I see that OCR started, and what does the processing state look like (text-free image)?**
  Ingestion is **asynchronous**: the HTTP upload returns `Response("OK")` in ~0.1 s (before any OCR), and all OCR work happens in a **django-q background worker**. You watch OCR through two real signals: (a) the **WebSocket progress stream** broadcast to the Channels group `"status_updates"`, which walks a fixed sequence `STARTING/new_file@0%` → **`WORKING/parsing_document@20%` (the OCR-start checkpoint)** → `WORKING/generating_thumbnail@70%` → `WORKING/parse_date@90%` → `WORKING/save_document@95%` → `SUCCESS/finished@100%`; and (b) the worker's DEBUG log line **`Calling OCRmyPDF with args: {…}`** from logger `paperless.parsing.tesseract`. The document row does **not** exist during processing — `document_id` is `null` in every progress event until the terminal `SUCCESS`, which is the first event to carry the new `document_id`.

- **P2 — Does an image that already shows text skip OCR?**
  **No.** For **any image (non-PDF) input, OCR always runs.** The parser unconditionally sets `original_has_text = False` for images, so the `skip_noarchive` early-exit can never fire for an image. The pipeline shape and the progress/log signature are **identical** to the text-free case, so _you cannot tell "the image had visible text" from the output shape_. The only artifact that distinguishes "text produced by OCR" from "a page that already had a text layer" is the **ocrmypdf sidecar marker `[OCR skipped on page(s) …]`** — which appears for text-layer PDFs and **never** for images.

- **P3 — Which API fields carry OCR-generated vs. pre-existing text?**
  Both cases funnel recognized text into the **single `content` field**, and both expose a populated **`archived_file_name`**. The two responses are structurally identical (same 12 keys). **There is no field that labels text provenance** — nothing in `GET /api/documents/{id}/` says "this text came from OCR" vs. "this text pre-existed."

- **P4 — What happens when OCR is weak/incomplete?**
  The document is **still fully processed.** When no text is recovered, the parser logs `No text was found in {document_path}, the content will be empty.` and sets the content to `""`; the consume still reaches `SUCCESS/finished@100%` and a normal `Document` row is written (with an archive PDF, for an image). "Fully processed" has **no dedicated status/"processed" column** on the model — success is signalled only by _the row existing_ plus the terminal `finished@100%` event. The **only** outcome that prevents document creation is a genuine `ParseError` (→ `FAILED@100%`, no row) — and an empty-OCR result is **not** a `ParseError`.

---

## 2. Environment & Method

All observations were produced on the **canonical runtime**, not the planning shell.

### 2.1 Toolchain verification (captured)

```console
$ docker exec paperless-app python3 --version
Python 3.9.23

$ docker exec paperless-app bash -lc 'tesseract --version | head -1'
tesseract 4.1.1

$ docker exec paperless-app python3 -c "import ocrmypdf; print(ocrmypdf.__version__)"
13.4.3

# For contrast — the planning shell (NOT used for any observation):
$ python3 --version
Python 3.13.7
```

### 2.2 Pinned dependencies confirmed at runtime (match `requirements.txt`)

```console
$ docker exec paperless-app python3 -c "import django, rest_framework, channels, channels_redis, \
    redis, pdfminer, pikepdf, PIL, img2pdf, ocrmypdf, django_q; ..."
django             4.0.4     # requirements.txt:L38
djangorestframework 3.13.1   # requirements.txt:L39
django-q           (1, 3, 9) # requirements.txt:L37
channels           3.0.4     # requirements.txt:L23
channels_redis     3.4.0     # requirements.txt:L22
redis              3.5.3     # requirements.txt:L84
pdfminer.six       20220319  # requirements.txt:L64
pikepdf            5.1.1     # requirements.txt:L65
pillow             9.1.0     # requirements.txt:L66
img2pdf            0.4.4     # requirements.txt:L50
ocrmypdf           13.4.3    # requirements.txt:L60
```

### 2.3 Default OCR settings, resolved live by Django (captured)

```console
$ docker exec paperless-app python3 -c "from django.conf import settings; ..."
OCR_MODE        = 'skip'    # src/paperless/settings.py:L522
OCR_OUTPUT_TYPE = 'pdfa'    # src/paperless/settings.py:L518
OCR_CLEAN       = 'clean'   # src/paperless/settings.py:L526
OCR_LANGUAGE    = 'eng'     # src/paperless/settings.py:L514
OCR_PAGES       = 0
```

These are the _canonical defaults_; any run that overrides them is labeled **NON-CANONICAL** below. Documented semantics of the default `skip` mode: it "only performs OCR when necessary and always creates archived documents" (`docs/configuration.rst:L328-L329`), whereas `skip_noarchive` additionally avoids creating an archive when text already exists (`docs/configuration.rst:L311`).

### 2.4 How the stack was run

The full asynchronous stack was stood up so the worker and its WebSocket progress stream are genuinely observable:

- **Redis broker** (`redis:7-alpine`) — backs both django-q and the Channels channel layer.
- **Django app** — `python3 manage.py runserver 0.0.0.0:8000 --noreload` (HTTP 200 on `/api/`).
- **django-q worker** — `python3 manage.py qcluster` (`"django_q"` in `INSTALLED_APPS` — `src/paperless/settings.py:L110`; `Q_CLUSTER` config — `src/paperless/settings.py:L449`). This is the process that executes `consume_file` and performs OCR.
- **Database** — SQLite, reset to a **clean slate** (0 documents) before observation.
- **API token** — obtained via `POST /api/token/` (`username=admin`), then used as `Authorization: Token …` on every upload/GET.

**Progress capture (real, not a mock):** the browser receives progress via `StatusConsumer` (`src/paperless/consumers.py`), which does `group_add("status_updates", …)` on connect and forwards each broadcast's `event["data"]` verbatim over the WebSocket route `ws/status/` (`src/paperless/urls.py:L137`). To record the identical payloads without a browser, a small **channel-layer subscriber** joined the **same real `"status_updates"` group** via the configured Redis channel layer and logged each broadcast with timestamps. It uses the exact same Channels mechanism the app uses — it is **not** a mock or a debug hook.

**Component-level probes (labeled NON-CANONICAL):** to expose the ocrmypdf **sidecar file** (which the canonical async path deletes on cleanup) and to exercise `skip_noarchive`, two throwaway scripts called the **real** `RasterisedDocumentParser.parse(...)` directly (no mocks). All such output is explicitly labeled **_component-level probe (NON-CANONICAL)_**. Inputs were copied to `/tmp` (outside the repo tree); repository fixtures were never modified.

**Temporary-artifact hygiene:** every scratch item (scripts, scratch media, SQLite DB, media/consume dirs) lives **outside** the repository tree (under `/tmp` and inside the container). The repository working tree is left pristine except for this one document — proven in §10.

---

## 3. Pipeline Overview (the async path, with citations)

The observable OCR behavior lives entirely in the **background worker**, because the HTTP request returns before any OCR happens.

```
POST /api/documents/post_document/            src/documents/views.py:L497 (PostDocumentView.post)
   │  writes temp file; task_id = str(uuid.uuid4())          views.py:L521
   │  async_task("documents.tasks.consume_file", …)          views.py:L523-L533 (name arg L524)
   │  return Response("OK")   ← returns immediately, no OCR   views.py:L535
   ▼ (enqueued on Redis)
django-q worker → consume_file(...)           src/documents/tasks.py:L184
   │            → Consumer().try_consume_file  src/documents/tasks.py:L236
   ▼
Consumer.try_consume_file(...)                src/documents/consumer.py:L180
   ├─ STARTING / new_file            @ 0%     consumer.py:L202
   ├─ WORKING  / parsing_document    @ 20%     consumer.py:L259   ← OCR START
   │     └─ RasterisedDocumentParser.parse(…)  src/paperless_tesseract/parsers.py:L230-L327
   │           └─ log "Calling OCRmyPDF with args: {…}"        parsers.py:L260
   │           └─ ocrmypdf.ocr(**args)                          parsers.py:L261
   ├─ WORKING  / generating_thumbnail @ 70%    consumer.py:L264
   ├─ WORKING  / parse_date          @ 90%     consumer.py:L274
   ├─ WORKING  / save_document       @ 95%     consumer.py:L294
   ├─ Consumer._store(...) → Document.objects.create(content=text, …)
   │                                            consumer.py:L379-L412 (create L398-L406, content=text L400)
   └─ SUCCESS  / finished            @ 100% (with document.id)  consumer.py:L375
   (on ParseError: _fail → FAILED@100% + raise ConsumerError    consumer.py:L78-L81; caught at L278→L280)
```

**The progress payload** is emitted by `Consumer._send_progress(...)` (`consumer.py:L56-L76`) to the Channels group `"status_updates"` (`consumer.py:L73-L74`). Every payload has **exactly seven keys** (`consumer.py:L64-L72`):

```
filename, task_id, current_progress, max_progress, status, message, document_id
```

The human-readable `message` strings are module constants: `new_file` (`consumer.py:L43`), `parsing_document` (`L45`), `generating_thumbnail` (`L46`), `parse_date` (`L47`), `save_document` (`L48`), `finished` (`L49`).

---

## 4. P1 — Observing OCR start & processing state (text-free image `no-text-alpha.png`)

**Input:** `src/paperless_tesseract/tests/samples/no-text-alpha.png` — an image with no embedded text layer.

### 4.1 The upload is asynchronous (OCR has not started yet when the HTTP call returns)

**Command (observed):**

```bash
docker exec paperless-app bash -lc \
  "curl -s -o /tmp/p1_resp_body.txt \
        -w 'HTTP_CODE=%{http_code}\nTIME_TOTAL=%{time_total}s\n' \
        -H 'Authorization: Token <token>' \
        -F document=@/app/src/paperless_tesseract/tests/samples/no-text-alpha.png \
        http://localhost:8000/api/documents/post_document/"
```

**Output (observed):**

```
HTTP_CODE=200
TIME_TOTAL=0.106975s
```

```
# response body:
"OK"
```

The endpoint returns the literal string `"OK"` in ~0.1 s. OCR itself took ~16 s (see the timeline below), so the HTTP response demonstrably precedes OCR. This is the enqueue-and-return behavior at `src/documents/views.py:L523-L535` (dispatch `async_task(...)` then `return Response("OK")`).

### 4.2 "Before" state — the document row does not exist yet

**Command (observed):**

```bash
docker exec paperless-app python3 -c "from documents.models import Document; print(Document.objects.count())"
```

**Output (observed):**

```
0
```

No `Document` row exists before (and, as shown next, during) processing.

### 4.3 "During" state — the live progress stream is the OCR signal (full, unedited payloads)

This is the complete `"status_updates"` broadcast for one run, captured by the channel-layer subscriber. **All seven payload keys are present in every event**, and `task_id` is constant across the whole task:

```json
{"recv_wall": "2026-07-14 19:50:30", "recv_mono_s": 1.562, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}}
{"recv_wall": "2026-07-14 19:50:30", "recv_mono_s": 1.739, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 19:50:34", "recv_mono_s": 5.32, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}}
{"recv_wall": "2026-07-14 19:50:45", "recv_mono_s": 16.396, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}}
{"recv_wall": "2026-07-14 19:50:45", "recv_mono_s": 16.415, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}}
{"recv_wall": "2026-07-14 19:50:45", "recv_mono_s": 16.46, "payload": {"filename": "no-text-alpha.png", "task_id": "ff370244-f7c6-477b-9399-7af3ca9f008d", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

Reading this as the "processing state":

| Event | `status`   | `message`              | `current_progress` | `document_id` | Meaning / citation                                                        |
| ----- | ---------- | ---------------------- | ------------------ | ------------- | ------------------------------------------------------------------------- |
| 1     | `STARTING` | `new_file`             | `0`                | `null`        | consume accepted — `consumer.py:L202`                                     |
| 2     | `WORKING`  | `parsing_document`     | `20`               | `null`        | **OCR START** — emitted immediately before `parse()` — `consumer.py:L259` |
| 3     | `WORKING`  | `generating_thumbnail` | `70`               | `null`        | OCR finished; building thumbnail — `consumer.py:L264`                     |
| 4     | `WORKING`  | `parse_date`           | `90`               | `null`        | date parsing — `consumer.py:L274`                                         |
| 5     | `WORKING`  | `save_document`        | `95`               | `null`        | about to persist — `consumer.py:L294`                                     |
| 6     | `SUCCESS`  | `finished`             | `100`              | **`1`**       | done; **first event carrying `document_id`** — `consumer.py:L375`         |

Two decisive "during-state" facts, both **observed**:

- **The OCR-start signal is `WORKING/parsing_document@20%`.** It fires just before `RasterisedDocumentParser.parse(...)` is invoked (`consumer.py:L259` immediately precedes the `parse()` call).
- **`document_id` is `null` for every event until the terminal `SUCCESS`.** While OCR is running, there is no committed row and no id to look up; the id first appears in the `finished@100%` event (`consumer.py:L375`). This is exactly why "watching the worker/progress stream" — not "polling the documents table" — is the correct way to see in-flight OCR.

### 4.4 "During" state — the worker log proves OCR is actively running

The definitive log signal is the DEBUG line from logger `paperless.parsing.tesseract` (`parsers.py:L24`), emitted at `parsers.py:L260` right before `ocrmypdf.ocr(**args)` (`parsers.py:L261`). Full captured worker log for this run:

```
[2026-07-14 19:50:30,439] [INFO] [paperless.consumer] Consuming no-text-alpha.png
[2026-07-14 19:50:30,456] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-14 19:50:30,457] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-14 19:50:30,473] [DEBUG] [paperless.consumer] Parsing no-text-alpha.png...
[2026-07-14 19:50:30,594] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/pl/scratch/paperless-upload-hyl9yvju: 'dpi'
[2026-07-14 19:50:30,694] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 35 based on image width 297
[2026-07-14 19:50:30,694] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /tmp/pl/scratch/paperless-upload-hyl9yvju for compatibility with img2pdf
[2026-07-14 19:50:30,701] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-hyl9yvju', 'output_file': '/tmp/pl/scratch/paperless-11nws4ga/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-11nws4ga/sidecar.txt', 'image_dpi': 35}
[2026-07-14 19:50:32,524] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 19:50:32,525] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-14 19:50:32,525] [WARNING] [paperless.parsing.tesseract] Error while getting DPI from image /tmp/pl/scratch/paperless-upload-hyl9yvju: 'dpi'
[2026-07-14 19:50:32,526] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 35 based on image width 297
[2026-07-14 19:50:32,526] [DEBUG] [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-hyl9yvju', 'output_file': '/tmp/pl/scratch/paperless-11nws4ga/archive-fallback.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'force_ocr': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-11nws4ga/sidecar-fallback.txt', 'image_dpi': 35}
[2026-07-14 19:50:34,036] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 19:50:34,036] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/pl/scratch/paperless-upload-hyl9yvju, the content will be empty.
[2026-07-14 19:50:34,037] [DEBUG] [paperless.consumer] Generating thumbnail for no-text-alpha.png...
[2026-07-14 19:50:37,943] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/pl/scratch/paperless-11nws4ga/convert.png -out /tmp/pl/scratch/paperless-11nws4ga/thumb_optipng.png
[2026-07-14 19:50:45,149] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-14 19:50:45,172] [DEBUG] [paperless.consumer] Deleting file /tmp/pl/scratch/paperless-upload-hyl9yvju
[2026-07-14 19:50:45,179] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/pl/scratch/paperless-11nws4ga
[2026-07-14 19:50:45,180] [INFO] [paperless.consumer] Document 2026-07-14 no-text-alpha consumption finished
```

Notes on the concrete OCR signals visible here (all **observed**):

- **`Calling OCRmyPDF with args: {…}`** is the OCR-start log. The arg dict directly reflects the resolved settings: `'skip_text': True` (from `OCR_MODE='skip'` — the arg is built at `parsers.py:L157-L158`), `'language': 'eng'`, `'output_type': 'pdfa'`, `'clean': True`.
- For this alpha PNG the DPI probe failed, so paperless fell back to an A4-based estimate (`Estimated DPI 35 based on image width 297`) and stripped the alpha layer for `img2pdf` (`Removing alpha layer …` — `parsers.py:L191-L201`).
- The first OCR pass produced no text → a **force-OCR fallback** was attempted (`Fallback: Calling OCRmyPDF with args: {… 'force_ocr': True …}`); it also produced no text → the empty-content last-resort warning. (This empty-text outcome is the P4 case; it is detailed in §7.)

### 4.5 "After" state — the row now exists (with OCR-derived content)

**Command (observed):**

```bash
docker exec paperless-app python3 -c "from documents.models import Document as D; d=D.objects.get(pk=1); \
  print(repr(d.title), repr(d.mime_type), repr(d.content), repr(d.checksum), repr(d.archive_filename))"
```

**Output (observed):**

```
'no-text-alpha' 'image/png' '' 'a13999fabb65f07735dfcbbb001ddaa8' '0000001.pdf'
```

After consumption a `Document` row exists (`content=text` written at `consumer.py:L400`). For this text-free image the recognized `content` is empty (`''`), yet an **archive PDF was still produced** (`archive_filename='0000001.pdf'`).

### 4.6 Stability across ≥2 runs (required for the ordering claim)

The same input was consumed three times (deleting the row between runs, because identical-checksum re-uploads are blocked by the duplicate pre-check at `consumer.py:L213`). The checkpoint **ordering was identical every time**:

```
STARTING/new_file@0 → WORKING/parsing_document@20 → WORKING/generating_thumbnail@70 →
WORKING/parse_date@90 → WORKING/save_document@95 → SUCCESS/finished@100
```

The only value that changed between runs was the terminal `document_id` (auto-increment PK: `1`, then `2`, then `3`). **The event sequence is stable.** The surviving text-free document after the stability runs is **`id=3`** (used for the P3 comparison).

---

## 5. P2 — Does an image that already contains visible text skip OCR? (`simple.png`)

**Input:** `src/paperless_tesseract/tests/samples/simple.png` — a raster image that visibly contains the text "This is a test document." Compare with the text-free `no-text-alpha.png` from P1.

### 5.1 Observed: the image enters the exact same OCR pipeline (no skip)

**Command (observed):**

```bash
docker exec paperless-app bash -lc \
  "curl -s -w 'HTTP %{http_code} in %{time_total}s\n' -H 'Authorization: Token <token>' \
   -F document=@/app/src/paperless_tesseract/tests/samples/simple.png \
   http://localhost:8000/api/documents/post_document/"
```

**Output (observed):** `HTTP 200 in 0.028s`, body `"OK"` — same async behavior as P1.

Full `"status_updates"` stream (task_id `5d86172f-4ede-40e1-a2ac-d050f8db165b`) — **identical shape to the text-free case**:

```json
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",             "document_id": null}}
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 20,  "max_progress": 100, "status": "WORKING",  "message": "parsing_document",     "document_id": null}}
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 70,  "max_progress": 100, "status": "WORKING",  "message": "generating_thumbnail", "document_id": null}}
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 90,  "max_progress": 100, "status": "WORKING",  "message": "parse_date",          "document_id": null}}
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 95,  "max_progress": 100, "status": "WORKING",  "message": "save_document",        "document_id": null}}
{"payload": {"filename": "simple.png", "task_id": "5d86172f-4ede-40e1-a2ac-d050f8db165b", "current_progress": 100, "max_progress": 100, "status": "SUCCESS",  "message": "finished",             "document_id": 4}}
```

The `WORKING/parsing_document@20%` OCR-start checkpoint fires exactly as before, so **an image that already shows text does not skip OCR** — it traverses the identical pipeline. The worker OCR log confirms `ocrmypdf` was called with the same signature:

```
[2026-07-14 19:54:48,893] [INFO]  [paperless.consumer] Consuming simple.png
[2026-07-14 19:54:48,894] [DEBUG] [paperless.consumer] Detected mime type: image/png
[2026-07-14 19:54:48,895] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-14 19:54:48,909] [DEBUG] [paperless.consumer] Parsing simple.png...
[2026-07-14 19:54:49,008] [DEBUG] [paperless.parsing.tesseract] Estimated DPI 62 based on image width 517
[2026-07-14 19:54:49,009] [DEBUG] [paperless.parsing.tesseract] Detected DPI for image /tmp/pl/scratch/paperless-upload-_zfj8wzq: 72
[2026-07-14 19:54:49,009] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/tmp/pl/scratch/paperless-upload-_zfj8wzq', 'output_file': '/tmp/pl/scratch/paperless-azcpoprn/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/pl/scratch/paperless-azcpoprn/sidecar.txt', 'image_dpi': 72}
[2026-07-14 19:54:50,086] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-14 19:54:50,087] [DEBUG] [paperless.consumer] Generating thumbnail for simple.png...
[2026-07-14 19:54:50,346] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/pl/scratch/paperless-azcpoprn/convert.png -out /tmp/pl/scratch/paperless-azcpoprn/thumb_optipng.png
[2026-07-14 19:54:50,894] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-14 19:54:50,922] [DEBUG] [paperless.consumer] Deleting file /tmp/pl/scratch/paperless-upload-_zfj8wzq
[2026-07-14 19:54:50,952] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/pl/scratch/paperless-azcpoprn
[2026-07-14 19:54:50,953] [INFO]  [paperless.consumer] Document 2026-07-14 simple consumption finished
```

Two differences from the text-free run — both benign, neither changing the pipeline shape:

- `simple.png` carries a DPI, so `Detected DPI … 72` is used (vs. the A4 fallback for `no-text-alpha.png`).
- OCR **found** text, so `Using text from sidecar file` is followed straight by thumbnail generation — **no** force-OCR fallback and **no** "No text was found" warning.

The resulting row: **`id=4`**, `title='simple'`, `content='This is a test document.'`, `archive_filename='0000004.pdf'`.

### 5.2 Why images always OCR (the code reason) — _inferred from code_, corroborated by a probe

The reason is a single unconditional assignment. In `RasterisedDocumentParser.parse(...)`:

- For a **PDF**, the parser first extracts any embedded text and decides "born-digital" by length: `text_original = self.extract_text(None, document_path)` (`parsers.py:L235`) and `original_has_text = text_original and len(text_original) > 50` (`parsers.py:L236`).
- For a **non-PDF image**, the `else` branch sets `text_original = None` (`parsers.py:L238`) and **`original_has_text = False`** (`parsers.py:L239`) — **unconditionally, regardless of any text visible in the raster.**
- The only early-exit that skips `ocrmypdf` is gated on that flag: `if settings.OCR_MODE == "skip_noarchive" and original_has_text:` (`parsers.py:L241`) → log `Document has text, skipping OCRmyPDF entirely.` (`parsers.py:L242`) → `return` (`parsers.py:L244`).

Because images force `original_has_text = False`, **this early-exit can never fire for an image**, even under `skip_noarchive`. This is verified directly in §8 (component probe case c).

### 5.3 How to tell the difference after processing — the sidecar marker

Since the pipeline shape is identical, **you cannot tell "the image had visible text" from the output shape.** The distinguishing artifact lives inside ocrmypdf's **sidecar text file**, which `extract_text(...)` inspects: it treats the text as usable only when `"[OCR skipped on page" not in text` (`parsers.py:L104`), logging `Using text from sidecar file` (`parsers.py:L107`) when the marker is absent versus `Incomplete sidecar file: discarding.` (`parsers.py:L110`) when it is present.

The canonical async path deletes the sidecar on cleanup, so a **_component-level probe (NON-CANONICAL)_** called the real parser and read the sidecar before cleanup:

**Command (observed):** `PYTHONPATH=/app/src python3 /tmp/probe_sidecar.py` (calls the real `RasterisedDocumentParser.parse()` on a `/tmp` copy of `simple.png`; no mock)
**Output (observed):**

```
======================================================================
INPUT: simple.png  (mime=image/png)
  parsed text repr : 'This is a test document.'
  archive produced : True
  sidecar.txt exists : True
    content repr : 'This is a test document.\n'
    contains '[OCR skipped on page' marker : False
```

For an **image**, the sidecar holds OCR-produced text with **no** `[OCR skipped on page(s) …]` marker, so `extract_text` takes the `Using text from sidecar file` branch — i.e., **the text is OCR-generated, not pre-existing.** (The contrasting text-layer-PDF case, where the marker _is_ present, is shown in §8.)

---

## 6. P3 — Comparing the final API responses

**Goal:** compare `GET /api/documents/{id}/` for the text-free image (P1, `id=3`) and the image-with-text (P2, `id=4`) to see which fields carry OCR-generated vs. pre-existing text.

**Command (observed):**

```bash
docker exec paperless-app bash -lc \
  "curl -s -H 'Authorization: Token <token>' http://localhost:8000/api/documents/3/ | python3 -m json.tool"
docker exec paperless-app bash -lc \
  "curl -s -H 'Authorization: Token <token>' http://localhost:8000/api/documents/4/ | python3 -m json.tool"
```

**`GET /api/documents/3/` — text-free image (full, unedited body):**

```json
{
  "id": 3,
  "correspondent": null,
  "document_type": null,
  "title": "no-text-alpha",
  "content": "",
  "tags": [],
  "created": "2026-07-14T19:52:38.938309Z",
  "modified": "2026-07-14T19:52:49.108142Z",
  "added": "2026-07-14T19:52:49.087121Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-14 no-text-alpha.png",
  "archived_file_name": "2026-07-14 no-text-alpha.pdf"
}
```

**`GET /api/documents/4/` — image with visible text (full, unedited body):**

```json
{
  "id": 4,
  "correspondent": null,
  "document_type": null,
  "title": "simple",
  "content": "This is a test document.",
  "tags": [],
  "created": "2026-07-14T19:54:48Z",
  "modified": "2026-07-14T19:54:50.921894Z",
  "added": "2026-07-14T19:54:50.895417Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-14 simple.png",
  "archived_file_name": "2026-07-14 simple.pdf"
}
```

### 6.1 Field-by-field comparison

A programmatic key-set comparison confirmed both bodies expose the **same 12 keys** (`sorted(a) == sorted(b)` → `True`), and a check for any provenance/OCR-flag field returned **`False`**.

| Field                                                             | doc 3 (text-free image)          | doc 4 (image with text)      | Notes / citation                                                                                                                                                                                                                                                                                                    |
| ----------------------------------------------------------------- | -------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                                                         | `""` (empty)                     | `"This is a test document."` | **The single field that carries recognized text** — model `content` TextField `models.py:L117-L124`; serialized at `serialisers.py:L227`. In both cases the text (or its absence) is **OCR-derived**, since both inputs are images.                                                                                 |
| `archived_file_name`                                              | `"2026-07-14 no-text-alpha.pdf"` | `"2026-07-14 simple.pdf"`    | **Populated in BOTH.** `SerializerMethodField` (`serialisers.py:L208`); getter returns the archive public name when `obj.has_archive_version` else `None` (`serialisers.py:L213-L217`); `has_archive_version` ⇔ `archive_filename is not None` (`models.py:L237-L239`). Non-null ⇒ an OCR archive PDF was produced. |
| `original_file_name`                                              | `"2026-07-14 no-text-alpha.png"` | `"2026-07-14 simple.png"`    | the stored original — `serialisers.py:L233`                                                                                                                                                                                                                                                                         |
| `id`, `title`, `created`, `modified`, `added`                     | differ trivially                 | differ trivially             | per-document values                                                                                                                                                                                                                                                                                                 |
| `correspondent`, `document_type`, `tags`, `archive_serial_number` | `null` / `[]`                    | `null` / `[]`                | unset in both                                                                                                                                                                                                                                                                                                       |

### 6.2 P3 conclusions (answering "which fields show OCR-generated vs existing text")

1. **There is exactly one field for recognized text: `content`.** Both OCR-generated text (doc 4) and the empty result (doc 3) land there (`models.py:L117-L124` → `serialisers.py:L227`). For images the text is _always_ OCR-generated; there is **no separate "existing text" field**.
2. **`archived_file_name` is non-null in both**, indicating an OCR archive PDF was produced in each case (`serialisers.py:L213-L217`, `models.py:L237-L239`).
3. **No field labels provenance.** Nothing in the response says "this text came from OCR" vs. "this text pre-existed" — the two documents are indistinguishable in response shape. (The only place that distinction is ever visible is the transient ocrmypdf sidecar marker seen in §5.3 / §8, which is not surfaced by the API.)

---

## 7. P4 — Weak/incomplete OCR: what happens to the final state?

**Input driving the weak/empty case:** the text-free `no-text-alpha.png` from P1. Tesseract recovers no usable text from it even after a force-OCR fallback, so it is the natural "weak/incomplete OCR" specimen.

### 7.1 Observed: empty OCR result → empty `content`, but the document is still fully consumed

The relevant last-resort logic in `parse(...)`: after OCR, if `not self.text` (`parsers.py:L318`), then if the original had text it reuses that (`parsers.py:L319-L320`); **otherwise** it logs `No text was found in {document_path}, the content will be empty.` (`parsers.py:L322-L326`) and sets `self.text = ""` (`parsers.py:L327`). Crucially, `parse()` then **returns normally** — an empty OCR result is not an error.

The warning was captured live in the P1 worker log (§4.4):

```
[2026-07-14 19:50:34,036] [WARNING] [paperless.parsing.tesseract] No text was found in /tmp/pl/scratch/paperless-upload-hyl9yvju, the content will be empty.
```

And the run still reached the terminal success event (from the P1 stream, §4.3):

```json
{"payload": {"filename": "no-text-alpha.png", ..., "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}}
```

**Before/during/after for the weak-OCR case** (all **observed**):

- **Before:** `Document.objects.count()` = `0` — no row (§4.2).
- **During:** the full progress sequence fires (`STARTING`→…→`SUCCESS`), with `document_id=null` throughout until the end, and the worker logs the empty-content warning mid-parse (§4.3–§4.4).
- **After:** a normal row exists — `id=3`, `content=''`, `archive_filename='0000003.pdf'`. Its API body (§6, doc 3) is a completely ordinary document response.

**Answer:** Yes — a weak/incomplete (even fully empty) OCR result **still counts as fully processed.** It is reflected in saved metadata as `content=""` on an otherwise-normal row **with an archive PDF**; there is no error marker and no "partial" flag.

### 7.2 "Fully processed" has no status column — observed via model introspection

There is no dedicated OCR-status/"processed" column on the `Document` model (`models.py:L88` through ~`L210`). Confirmed live:

**Command (observed):**

```bash
docker exec paperless-app python3 -c "from documents.models import Document; \
  print([f.name for f in Document._meta.get_fields() if getattr(f,'concrete',False)])"
```

**Output (observed):**

```
['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum',
 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename',
 'archive_filename', 'archive_serial_number', 'tags']
```

A filter for any `status`/`processed`/`ocr`/`state`-like field name returned `[]` (none). So "fully processed" is **not** a stored attribute — it is signalled only by **(a) the row existing** and **(b) the terminal `SUCCESS/finished@100%` progress event** (`consumer.py:L375`).

### 7.3 The only outcome that prevents document creation: a genuine `ParseError` (observed)

An empty-OCR result must be distinguished from a **hard parse failure**. To exercise the failure path, a deliberately corrupt PDF (valid `%PDF` header + garbage body; `libmagic` classified it `application/pdf`) was uploaded through the canonical path. **Kept in `/tmp`, outside the repo tree.**

**Command (observed):** `curl -F document=@/tmp/corrupt.pdf … /api/documents/post_document/` → `HTTP 200 "OK"`.
Before: `Document.objects.count()` = `2`. Captured `"status_updates"` stream:

```json
{"recv_wall": "2026-07-14 19:58:42", "recv_mono_s": 0.668, "payload": {"filename": "corrupt.pdf", "task_id": "aa83620d-f558-4566-8dea-68003f5797da", "current_progress": 0,   "max_progress": 100, "status": "STARTING", "message": "new_file",         "document_id": null}}
{"recv_wall": "2026-07-14 19:58:42", "recv_mono_s": 0.688, "payload": {"filename": "corrupt.pdf", "task_id": "aa83620d-f558-4566-8dea-68003f5797da", "current_progress": 20,  "max_progress": 100, "status": "WORKING",  "message": "parsing_document", "document_id": null}}
{"recv_wall": "2026-07-14 19:58:42", "recv_mono_s": 1.129, "payload": {"filename": "corrupt.pdf", "task_id": "aa83620d-f558-4566-8dea-68003f5797da", "current_progress": 100, "max_progress": 100, "status": "FAILED",   "message": "InputFileError: ",  "document_id": null}}
```

After: `Document.objects.count()` = `2` (**unchanged — no row created**). The worker log showed pdfminer failing, ocrmypdf failing, the force-OCR fallback also failing, and finally:

```
[ERROR] [paperless.consumer] Error while consuming document corrupt.pdf: InputFileError:
```

This is the failure path: an unrecoverable parse error becomes a `ParseError` (`parsers.py:L310`), caught by the consumer (`consumer.py:L278`) → `_fail(...)` (`consumer.py:L280`) which sends `FAILED@100%` (`consumer.py:L79`) and raises `ConsumerError` (`consumer.py:L81`), aborting the consume so **no `Document` is written.**

### 7.4 The key P4 distinction (observed, not merely inferred)

- **Empty/weak OCR** (`NoTextFoundException` on the first pass) does **not** abort. It triggers the force-OCR fallback and, if still empty, the empty-content last resort (`self.text = ""`, `parsers.py:L327`) and a **normal return** → a document **is** created with `content=""`.
- **A genuine `ParseError`** (unrecoverable) is the **only** path that routes through `_fail` → `FAILED@100%` and prevents document creation.

So "weak or incomplete OCR" always yields a **fully-consumed document** (possibly with empty `content`); only a hard parse failure produces no document at all.

---

## 8. Supporting evidence — the digital-PDF skip contrast (why images differ from PDFs)

This section substantiates the §5 claim that images can _never_ early-exit, by contrasting them with a **text-layer PDF** that _can_. It also exhibits the literal `[OCR skipped on page(s) …]` sidecar marker.

**(a) Canonical: `simple-digital.pdf` under default `skip`.** Uploaded through the real path → `id=5`, `content='This is a test document.'`, `archive_filename='0000005.pdf'`. Its `"status_updates"` stream has the same six-checkpoint shape as every other consume:

```json
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 0,   ..., "status": "STARTING", "message": "new_file",             "document_id": null}}
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 20,  ..., "status": "WORKING",  "message": "parsing_document",     "document_id": null}}
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 70,  ..., "status": "WORKING",  "message": "generating_thumbnail", "document_id": null}}
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 90,  ..., "status": "WORKING",  "message": "parse_date",          "document_id": null}}
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 95,  ..., "status": "WORKING",  "message": "save_document",        "document_id": null}}
{"payload": {"filename": "simple-digital.pdf", "task_id": "090fffb7-a872-4dec-b80c-32b868462dec", "current_progress": 100, ..., "status": "SUCCESS",  "message": "finished",             "document_id": 5}}
```

Under the default `skip` mode, a text-layer PDF **still enters `ocrmypdf`** and **still gets an archive**, but its `content` comes from the pre-existing text layer (the OCR sidecar is discarded). This matches the community/discussion observation that, in the code, "calling OCRmyPDF is only skipped if PAPERLESS_OCR_MODE=skip_noarchive" — i.e., the default `skip` does not bypass ocrmypdf (see §10).

**(b)–(c) Component probe (NON-CANONICAL: cases (b)/(c) override `OCR_MODE='skip_noarchive'`).** The real parser was run to show the early-exit firing for a text-layer PDF but not for an image.

**Command (observed):** `PYTHONPATH=/app/src python3 /tmp/probe_skip.py` (scratch paths abbreviated as `<work>/`):

```
========================================================================
INPUT simple-digital.pdf  mime=application/pdf  OCR_MODE='skip'  [canonical default]
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file <work>/simple-digital.pdf
      LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '<work>/simple-digital.pdf', ..., 'skip_text': True, ..., 'sidecar': '/tmp/pl/scratch/paperless-tdu7jsq6/sidecar.txt'}
      LOG DEBUG paperless.parsing.tesseract: Incomplete sidecar file: discarding.
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file /tmp/pl/scratch/paperless-tdu7jsq6/archive.pdf
   -> parsed text repr : 'This is a test document.'
   -> archive produced : True  (archive_path='/tmp/pl/scratch/paperless-tdu7jsq6/archive.pdf')
   -> sidecar.txt: contains '[OCR skipped on page' = True  repr='[OCR skipped on page(s) 1]'
========================================================================
INPUT multi-page-digital.pdf  mime=application/pdf  OCR_MODE='skip_noarchive'  [NON-CANONICAL]
      LOG DEBUG paperless.parsing.tesseract: Extracted text from PDF file <work>/multi-page-digital.pdf
      LOG DEBUG paperless.parsing.tesseract: Document has text, skipping OCRmyPDF entirely.
   -> parsed text repr : 'This is a multi page document. Page 1.\n\nThis is a multi page document. Page 2.\n\nThis is a multi page document. Page 3.'
   -> archive produced : False  (archive_path=None)
========================================================================
INPUT simple.png  mime=image/png  OCR_MODE='skip_noarchive'  [NON-CANONICAL]
      LOG DEBUG paperless.parsing.tesseract: Estimated DPI 62 based on image width 517
      LOG DEBUG paperless.parsing.tesseract: Detected DPI for image <work>/simple.png: 72
      LOG DEBUG paperless.parsing.tesseract: Calling OCRmyPDF with args: {'input_file': '<work>/simple.png', ..., 'skip_text': True, ..., 'sidecar': '/tmp/pl/scratch/paperless-ugx5h0wg/sidecar.txt', 'image_dpi': 72}
      LOG DEBUG paperless.parsing.tesseract: Using text from sidecar file
   -> parsed text repr : 'This is a test document.'
   -> archive produced : True  (archive_path='/tmp/pl/scratch/paperless-ugx5h0wg/archive.pdf')
   -> sidecar.txt: contains '[OCR skipped on page' = False  repr='This is a test document.\n'
```

What this proves (all **observed**):

- **(a)** For a text-layer PDF the sidecar literally contains `'[OCR skipped on page(s) 1]'` — the exact marker `extract_text` branches on (`parsers.py:L104`) — so it logs `Incomplete sidecar file: discarding.` (`parsers.py:L110`) and takes the text from the PDF layer instead.
- **(b)** A text-layer PDF (pdfminer length > 50 ⇒ `original_has_text=True`) under `skip_noarchive` hits the early-exit: `Document has text, skipping OCRmyPDF entirely.` (`parsers.py:L242`) → **no archive** (`archive_path=None`, `parsers.py:L244`), no ocrmypdf call.
- **(c)** The **same** `skip_noarchive` mode applied to an **image** does **not** early-exit — ocrmypdf still runs and an archive is produced — because the image branch forced `original_has_text=False` (`parsers.py:L238-L239`). Its sidecar has **no** skip marker, confirming the text is OCR-generated.

**Behavioral oracle (`src/paperless_tesseract/tests/test_parser.py`)** agrees: `test_skip_noarchive_withtext` (`L357`, `multi-page-digital.pdf`) asserts `archive_path is None`; `test_skip_noarchive_notext` (`L370`, `@override_settings OCR_MODE="skip_noarchive"`, image-based `multi-page-images.pdf`) asserts the archive file exists; `test_image_simple` (`L216`) parses `simple.png` through the OCR path.

---

## 9. Coverage / Checklist Pass

Every sub-question and every named item is addressed, with its evidence anchor and `file:line`.

| Sub-question / named item                                      | Answer (one line)                                                                                                                      | Evidence anchor                  | Primary `file:line`                                               |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | ----------------------------------------------------------------- |
| **P1** — how to _see_ OCR started (text-free image)            | Watch the worker: `WORKING/parsing_document@20%` + the `Calling OCRmyPDF with args:` DEBUG log                                         | §4.3, §4.4 (full payloads + log) | `consumer.py:L259`; `parsers.py:L260`                             |
| P1 — what the **processing state** looks like                  | Fixed checkpoint sequence 0→20→70→90→95→100; `document_id=null` until the end                                                          | §4.3 (table)                     | `consumer.py:L202,L259,L264,L274,L294,L375`                       |
| P1 — **background-worker** behavior / active-OCR **signals**   | HTTP returns `"OK"` (~0.1 s) then a django-q worker runs OCR; signals = `"status_updates"` events + `paperless.parsing.tesseract` logs | §4.1, §4.3, §4.4                 | `views.py:L523-L535`; `tasks.py:L184,L236`; `consumer.py:L56-L76` |
| P1 — before / during / after                                   | before: 0 rows; during: events fire, no row/id; after: row with (empty) OCR content + archive                                          | §4.2, §4.3, §4.5                 | `consumer.py:L400`                                                |
| P1 — stability across ≥2 runs                                  | Ordering identical over 3 runs; only terminal `document_id` differs                                                                    | §4.6                             | `consumer.py:L213` (dedup)                                        |
| **P2** — does an image with text skip OCR?                     | **No** — images always OCR; identical pipeline shape                                                                                   | §5.1 (stream + log)              | `parsers.py:L238-L239,L241-L244`                                  |
| P2 — how to tell the difference afterward                      | Only via the ocrmypdf sidecar marker `[OCR skipped on page(s) …]`; absent for images                                                   | §5.3, §8(a)                      | `parsers.py:L104,L107,L110`                                       |
| **P3** — which fields carry OCR-generated vs pre-existing text | Both use the single `content` field; both expose `archived_file_name`; **no provenance field**                                         | §6 (both JSON + table)           | `serialisers.py:L227,L213-L217`; `models.py:L117-L124,L237-L239`  |
| the **`content`** field                                        | single field for recognized text (OCR or pre-existing)                                                                                 | §6.1                             | `models.py:L117-L124` → `serialisers.py:L227`                     |
| **`archived_file_name`**                                       | non-null ⇔ an OCR archive PDF exists; populated in both image cases                                                                    | §6.1                             | `serialisers.py:L213-L217`; `models.py:L237-L239`                 |
| **P4** — weak/incomplete OCR final state                       | Empty text → `content=""`; document **still fully consumed**, `SUCCESS@100%`                                                           | §7.1 (warning + SUCCESS)         | `parsers.py:L322-L327`; `consumer.py:L375`                        |
| P4 — does it count as "**fully processed**"?                   | **Yes**; signalled by the row existing + terminal `finished@100%`                                                                      | §7.1, §7.2                       | `consumer.py:L375`                                                |
| P4 — how reflected in **saved metadata**                       | `content=""` on a normal row (+ archive); **no status/"processed" column**                                                             | §7.2 (field list)                | `models.py:L88-L210`                                              |
| P4 — contrast: what prevents document creation                 | A genuine `ParseError` only → `FAILED@100%`, no row (empty OCR does **not** abort)                                                     | §7.3, §7.4                       | `parsers.py:L310`; `consumer.py:L278,L280,L79,L81`                |
| Supporting — PDF vs image skip contrast                        | text-layer PDF _can_ early-exit under `skip_noarchive`; images cannot                                                                  | §8(a)(b)(c)                      | `parsers.py:L241-L244`                                            |

**Observed vs. inferred.** Everything above marked with a captured payload/log/JSON/field-dump is **observed at runtime**. The single item labeled **_inferred from code_** is the _mechanistic explanation_ in §5.2 (the unconditional `original_has_text=False` assignment); its behavioral consequence is nonetheless **observed** in §8 case (c).

---

## 10. External corroboration (web research)

The runtime findings were cross-checked against external sources (referenced, not reproduced):

- **ocrmypdf sidecar marker.** The OCRmyPDF project confirms that `--sidecar` writes an `[OCR skipped on page(s) …]` line into the sidecar text file for pages where OCR was skipped — the exact marker `extract_text` branches on (`parsers.py:L104`). See OCRmyPDF issue #1353 (`github.com/ocrmypdf/OCRmyPDF/issues/1353`), which shows a `"[OCR skipped on page(s) 2-5]"` sidecar line — matching the observed `'[OCR skipped on page(s) 1]'` in §8(a).
- **paperless-ngx OCR-mode semantics.** The official configuration docs state the default `skip` mode performs OCR only when needed and always creates archived documents, while `skip_noarchive` additionally skips creating an archive when text already exists — corroborating `src/paperless/settings.py:L522` and `docs/configuration.rst:L311,L328-L329`. See `docs.paperless-ngx.com/configuration/`.
- **Default `skip` still calls OCRmyPDF.** A maintainer-answered discussion notes that, in the code, calling OCRmyPDF is only skipped under `skip_noarchive` — i.e., the default `skip` does **not** bypass ocrmypdf. This corroborates the P2/§8(a) observation that a text-layer PDF (and every image) still enters ocrmypdf under `skip`. See paperless-ngx discussion #2746 (`github.com/paperless-ngx/paperless-ngx/discussions/2746`).

---

## 11. Repository integrity note

The investigation was **read-only** with respect to the source tree. All observation scaffolding (scripts, scratch media, the SQLite DB, media/consume dirs) lives **outside** the repository — under `/tmp` on the host and inside the throwaway containers — and none of it is committed.

**Command (observed) — in the repository working tree:**

```bash
$ git rev-parse --abbrev-ref HEAD
blitzy-2d7c5fb9-6420-4f89-9285-98269fdc35bb

$ git status --porcelain
?? blitzy/
```

The **only** untracked path is `blitzy/` (containing this single deliverable). No existing source file was modified, created, or deleted — satisfying the hard read-only constraint. (Note: the byte-identical `/app` checkout inside the observation container shows an in-place rewrite of one alpha-image test fixture, a side effect of the parser's alpha-layer handling at `parsers.py:L191-L201`; that is confined to the disposable container and does **not** touch this repository working tree, verified by the `git status` above.)
