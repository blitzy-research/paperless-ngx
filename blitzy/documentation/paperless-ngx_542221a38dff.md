# How paperless-ngx handles **image** uploads through OCR — a runtime‑grounded Q&A

This document answers four questions about how the **paperless-ngx** ingestion pipeline behaves when an **image file** (PNG/JPEG/…) is uploaded and processed by OCR. Every factual claim is backed by **either** an exact source citation (`file:line`) **or** verbatim runtime output that was captured by actually building and running the code inside the pinned Docker image. Where a claim could not be verified at runtime, it is flagged explicitly.

> **Read‑only investigation.** No source file was modified. The only artifact committed is this Markdown document. All observation scripts and fixtures lived in a container‑only scratch directory (`/root/ocr_probe/`, outside the repository bind‑mount) and were deleted afterward.

## The four questions (restated verbatim)

- **Q1 — Image with no embedded text.** When an image that carries no embedded/visible text is uploaded, how can an observer *see* that OCR has started? What does the document's processing state look like *while* OCR is running? How do the background workers behave during this phase, and what concrete signals indicate that OCR work is actively in progress?
- **Q2 — Similar image that already contains text.** When a similar image that already contains text is uploaded, does the system *skip* OCR entirely, or does it still *touch* the OCR pipeline in some way? How can the difference be told *after* processing has finished?
- **Q3 — Comparing final API responses.** When the two cases above are compared via their final API responses, which fields surface OCR‑generated text versus pre‑existing text?
- **Q4 — Weak/incomplete OCR results.** When OCR produces weak or incomplete results, what happens to the document's final state? Does the document still count as fully processed, and how is that outcome reflected in the saved metadata?

---

## 0. Environment & reproduction preamble

### 0.1 Branch, HEAD, and a note on the deliverable filename

The deliverable is named after the **source branch under investigation**, `paperless-ngx_542221a38dff`, per the task rule (R1). The git checkout used for authoring is on the Blitzy working branch, but the HEAD commit is the one the questions target.

```bash
$ git rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542
$ git branch --show-current
blitzy-c7f7fb43-5921-4fad-b843-22d5e5677639
```

> **Discrepancy noted (R1).** The working checkout's current branch is the mechanical Blitzy branch `blitzy-c7f7fb43-5921-4fad-b843-22d5e5677639`, not `paperless-ngx_542221a38dff`. The file is nonetheless named `paperless-ngx_542221a38dff.md` because that is the source branch the investigation targets and the exact deliverable path assigned by the task. HEAD (`542221a38dff06361e07976452f9aea24d210542`) matches the target commit.

### 0.2 Toolchain versions (verbatim banners)

All runtime observation was performed **inside the provided Docker container** (`paperless-qna`, built from `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff…`), which ships the OCR runtime and Redis. The host authoring shell has no OCR toolchain and was **not** used to run the pipeline.

```bash
$ docker exec paperless-qna python --version
Python 3.9.23
$ docker exec paperless-qna ocrmypdf --version
13.4.3
$ docker exec paperless-qna tesseract --version
tesseract 4.1.1
 leptonica-1.79.0
$ docker exec paperless-qna gs --version
9.53.3
$ docker exec paperless-qna redis-cli ping
PONG
```

Pinned Python packages (verbatim from `pip show`, salient lines):

```text
Name: ocrmypdf            Version: 13.4.3
Name: pikepdf             Version: 5.1.1
Name: Pillow              Version: 9.1.0
Name: img2pdf             Version: 0.4.4
Name: pdf2image           Version: 1.16.0
Name: django-q            Version: 1.3.9
Name: channels            Version: 3.0.4
Name: channels-redis      Version: 3.4.0
Name: djangorestframework Version: 3.13.1
Name: Django              Version: 4.0.4
Name: reportlab           Version: 3.6.9
```

These match the versions pinned in the repository (§0.6 of the plan), so the observations below reflect the intended runtime, not newer releases.

### 0.3 OCR configuration actually in effect (printed at runtime)

```bash
$ docker exec paperless-qna bash -lc 'cd /app/src && DJANGO_SETTINGS_MODULE=paperless.settings \
    python -c "import os,django; django.setup(); from django.conf import settings; \
    print(\"OCR_MODE        =\", repr(settings.OCR_MODE)); \
    print(\"OCR_OUTPUT_TYPE =\", repr(settings.OCR_OUTPUT_TYPE)); \
    print(\"OCR_LANGUAGE    =\", repr(settings.OCR_LANGUAGE)); \
    print(\"OCR_IMAGE_DPI   =\", repr(settings.OCR_IMAGE_DPI)); \
    print(\"OMP_THREAD_LIMIT=\", repr(os.environ.get(\"OMP_THREAD_LIMIT\")))"'
OCR_MODE        = 'skip'
OCR_OUTPUT_TYPE = 'pdfa'
OCR_LANGUAGE    = 'eng'
OCR_IMAGE_DPI   = None
OMP_THREAD_LIMIT= '1'
```

These correspond to the source defaults:

- `OCR_MODE = os.getenv("PAPERLESS_OCR_MODE", "skip")` — `src/paperless/settings.py:522`
- `OCR_OUTPUT_TYPE = os.getenv("PAPERLESS_OCR_OUTPUT_TYPE", "pdfa")` — `src/paperless/settings.py:518`
- `OCR_LANGUAGE = os.getenv("PAPERLESS_OCR_LANGUAGE", "eng")` — `src/paperless/settings.py:514`
- `os.environ["OMP_THREAD_LIMIT"] = "1"` — `src/paperless/settings.py:31` (also re‑asserted by the parser at `src/paperless_tesseract/parsers.py:232`)

> **`OCR_IMAGE_DPI = None` consequence.** Because the global image‑DPI override is unset, each fixture image **embeds** its own DPI so `parse()` does not raise the missing‑DPI `ParseError` (`src/paperless_tesseract/parsers.py:211-215`).

### 0.4 Fixtures used

Four fixtures were generated with Pillow 9.1.0 / reportlab 3.6.9 in the container‑only scratch dir. Their pixels may *depict* text, but a raster image **never** carries an embedded text *layer* — that distinction is the crux of Q2.

| Fixture | File | Kind | Purpose | Size |
|---|---|---|---|---|
| **A** | `fixtureA_notext.png` | PNG 1000×700, DPI≈150 | image with **no legible text** (shapes only) | 5741 B |
| **B** | `fixtureB_withtext.png` | PNG 1000×700, DPI≈150 | similar image with **rendered text pixels** ("PAPERLESS OCR", "Invoice 12345") | 5573 B |
| **C** | `fixtureC_textlayer.pdf` | PDF | genuine **embedded text‑layer** PDF (>50 chars) — the *contrast* case | 1592 B |
| **D** | `fixtureD_blank.png` | PNG 1000×700, DPI≈150 | **blank** image — forces the empty‑content path for Q4 | 3697 B |

### 0.5 How the pipeline was driven (two complementary strategies)

- **Strategy 2 (direct driver)** — a temporary script called `RasterisedDocumentParser.parse()` and `Consumer.try_consume_file()` directly under `DJANGO_SETTINGS_MODULE=paperless.settings`, capturing `self.text`, `self.archive_path`, the sidecar, worker logs, and the real Redis WebSocket frames.
- **Strategy 1 (HTTP + worker)** — a Django test `Client` issued a real `POST /api/documents/post_document/` (capturing the HTTP body and the enqueued `task_id`), and the Django‑Q task `documents.tasks.consume_file` was executed to capture the worker's return string and log lines.

A real `channels_redis` channel layer was used (`CHANNEL_LAYERS` backend `channels_redis.core.RedisChannelLayer`), so the progress frames quoted below are the **actual** frames relayed to the `status_updates` group — not mock payloads.

---

## Q1 — Image with no embedded text: how to *see* OCR start, the in‑progress state, worker behavior, and active‑OCR signals

**Fixture: A (`fixtureA_notext.png`).** This question has four distinct sub‑parts, answered in turn.

### Q1(a) — How can you *see* that OCR has started?

Two observable signals mark the start, one at the HTTP edge and one on the real‑time status channel.

**1. The upload endpoint returns immediately with the literal body `"OK"` and mints a `task_id` UUID.** The upload is handled by `post_document`, which generates a UUID, enqueues the async task, and returns `Response("OK")`:

- `task_id = str(uuid.uuid4())` — `src/documents/views.py:521`
- `async_task("documents.tasks.consume_file", …, task_id=task_id, …)` — `src/documents/views.py:523-533`
- `return Response("OK")` — `src/documents/views.py:535`

Captured verbatim (real `POST` via Django test `Client`, with the enqueue intercepted to read the generated id):

```text
### POST /api/documents/post_document/
HTTP status_code = 200
Content-Type = 'application/json'
response .content = b'"OK"'
response .json()  = 'OK'
[intercepted] async_task func = 'documents.tasks.consume_file'
[intercepted] async_task task_id (views.py:521 uuid4) = 'b8404260-8b59-4678-8958-f07ac3182ad0'
[intercepted] async_task task_name = 'fixtureA_notext.png'
```

So the very first externally visible sign that ingestion (and, for an image, OCR) has been kicked off is the **`"OK"`** body plus a freshly generated **`task_id`** (a UUID such as `b8404260-8b59-4678-8958-f07ac3182ad0`). The response is JSON (`Content-Type: application/json`) and the body is the JSON string `"OK"` (`b'"OK"'` on the wire).

**2. A `STARTING` progress frame appears on the WebSocket status channel.** The consumer's very first action after assigning the task id is to broadcast a `STARTING` frame:

- `self.task_id = task_id or str(uuid.uuid4())` — `src/documents/consumer.py:200`
- `self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)` — `src/documents/consumer.py:202`, where `MESSAGE_NEW_FILE = "new_file"` — `src/documents/consumer.py:43`

Those frames are relayed to browsers over the authenticated WebSocket route `ws/status/$`:

- `re_path(r"ws/status/$", StatusConsumer.as_asgi())` — `src/paperless/urls.py:137`
- ASGI wiring `"websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns))` — `src/paperless/asgi.py:20`
- `StatusConsumer.status_update()` forwards each event with `self.send(json.dumps(event["data"]))` — `src/paperless/consumers.py:33`

Captured verbatim (first frame drained from the real `status_updates` Redis group for Fixture A, `task_id` fixed to a sentinel for correlation):

```json
{"current_progress": 0, "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "new_file", "status": "STARTING", "task_id": "11111111-1111-1111-1111-111111111111"}
```

The literal `"status": "STARTING"` with `"message": "new_file"` at `current_progress: 0` is the on‑channel "OCR job has started" signal.

### Q1(b) — What does the in‑progress state look like *while* OCR is running?

**There is no persistent "processing status" to read.** The `Document` model has **no** status/state/processing column (enumerated in Q4 below). The in‑progress state exists **only** as the transient sequence of WebSocket frames (and the worker log lines). The full ordered frame sequence actually observed for Fixture A was **six discrete milestone frames**:

```json
{"current_progress": 0,   "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "new_file",             "status": "STARTING", "task_id": "11111111-1111-1111-1111-111111111111"}
{"current_progress": 20,  "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "parsing_document",     "status": "WORKING",  "task_id": "11111111-1111-1111-1111-111111111111"}
{"current_progress": 70,  "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "generating_thumbnail", "status": "WORKING",  "task_id": "11111111-1111-1111-1111-111111111111"}
{"current_progress": 90,  "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "parse_date",           "status": "WORKING",  "task_id": "11111111-1111-1111-1111-111111111111"}
{"current_progress": 95,  "document_id": null, "filename": "fixtureA_notext.png", "max_progress": 100, "message": "save_document",         "status": "WORKING",  "task_id": "11111111-1111-1111-1111-111111111111"}
{"current_progress": 100, "document_id": 4,    "filename": "fixtureA_notext.png", "max_progress": 100, "message": "finished",             "status": "SUCCESS",  "task_id": "11111111-1111-1111-1111-111111111111"}
```

The milestone frames are emitted at these exact source lines:

| Frame | Source | Constant |
|---|---|---|
| `0/100 STARTING new_file` | `src/documents/consumer.py:202` | `MESSAGE_NEW_FILE="new_file"` `consumer.py:43` |
| `20/100 WORKING parsing_document` | `src/documents/consumer.py:259` | `MESSAGE_PARSING_DOCUMENT="parsing_document"` `consumer.py:45` |
| `70/100 WORKING generating_thumbnail` | `src/documents/consumer.py:264` | `MESSAGE_GENERATING_THUMBNAIL="generating_thumbnail"` `consumer.py:46` |
| `90/100 WORKING parse_date` | `src/documents/consumer.py:274` | `MESSAGE_PARSE_DATE="parse_date"` `consumer.py:47` |
| `95/100 WORKING save_document` | `src/documents/consumer.py:294` | `MESSAGE_SAVE_DOCUMENT="save_document"` `consumer.py:48` |
| `100/100 SUCCESS finished` (+`document_id`) | `src/documents/consumer.py:375` | `MESSAGE_FINISHED="finished"` `consumer.py:49` |

The frame payload shape — the exact seven keys — is built by `_send_progress()`:

```text
payload = {
    "filename": …,          # consumer.py:65
    "task_id": self.task_id,# consumer.py:66
    "current_progress": …,  # consumer.py:67
    "max_progress": …,      # consumer.py:68
    "status": status,       # consumer.py:69
    "message": message,     # consumer.py:70
    "document_id": document_id, # consumer.py:71
}
async_to_sync(self.channel_layer.group_send)("status_updates", {"type": "status_update", "data": payload})  # consumer.py:73-76
```

**Honest finding about the "20–70 band" mid‑parse frames.** The consumer defines a `progress_callback` intended to map a parser's fine‑grained progress into a band:

```python
def progress_callback(current_progress, max_progress):
    # recalculate progress to be within 20 and 80        # consumer.py:238  (comment says 20 and 80)
    p = int((current_progress / max_progress) * 50 + 20)  # consumer.py:239  (formula -> spans 20..70)
    self._send_progress(p, 100, "WORKING")                # consumer.py:240
```

There are **two** things to reconcile here, and I verified both at runtime:

1. **Comment vs. formula mismatch.** The comment at `consumer.py:238` says "within 20 and 80", but the formula at `consumer.py:239` is `int((current/max) * 50 + 20)`, whose range is `20` (ratio 0) to `70` (ratio 1.0) — i.e., the **true band is 20–70, not 20–80**.
2. **For images the callback never fires at all.** `RasterisedDocumentParser` never invokes `self.progress()`/`progress_callback` (the base `progress()` hook at `src/documents/parsers.py:300` is not called anywhere on the tesseract path). Empirically, **no** frame with a `current_progress` strictly between 20 and 70 was ever emitted for an image — the in‑progress state is *exactly* the six discrete milestones above. So the 20–70 band is dead code for image OCR in this build; the mid‑parse experience is the jump `20 → 70`.

This is a case where reading the code alone (the comment) would mislead; running it shows the real behavior.

### Q1(c) — How do the background workers behave during this phase?

The upload does **not** run OCR in the web request. It enqueues a Django‑Q task that the `qcluster` worker executes off‑request:

- Enqueue: `async_task("documents.tasks.consume_file", …)` — `src/documents/views.py:523-533`
- Worker task entry: `def consume_file(…)` — `src/documents/tasks.py:184`
- It calls `Consumer().try_consume_file(...)` (`src/documents/tasks.py:236`) and on success returns the string `"Success. New document id {} created"` — `src/documents/tasks.py:247`.

Captured verbatim — running the real `documents.tasks.consume_file` on Fixture A (worker logger `paperless.consumer`, DEBUG handler forced) — shows the ordered worker log lines and the task's return value:

```text
CAPTURE INFO [paperless.consumer] Consuming fixtureA_notext.png
CAPTURE DEBUG [paperless.consumer] Detected mime type: image/png
CAPTURE DEBUG [paperless.consumer] Parser: RasterisedDocumentParser
CAPTURE DEBUG [paperless.consumer] Parsing fixtureA_notext.png...
[2026-07-01 04:51:16,553] [ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
CAPTURE DEBUG [paperless.consumer] Generating thumbnail for fixtureA_notext.png...
CAPTURE DEBUG [paperless.consumer] Saving record to database
CAPTURE DEBUG [paperless.consumer] Deleting file /root/ocr_probe/consume/worker_fixtureA.png
CAPTURE INFO [paperless.consumer] Document 2026-07-01 fixtureA_notext consumption finished
RETURN VALUE of consume_file(): 'Success. New document id 7 created'
```

These lines map to:

- `self.log("info", f"Consuming {self.filename}")` — `src/documents/consumer.py:215`
- `self.log("debug", f"Detected mime type: {mime_type}")` — `src/documents/consumer.py:221`
- `self.log("debug", f"Parser: {type(document_parser).__name__}")` — `src/documents/consumer.py:246` → prints `RasterisedDocumentParser` (the image parser was selected)
- `self.log("info", "Document {} consumption finished".format(document))` — `src/documents/consumer.py:373`
- return `"Success. New document id 7 created"` — `src/documents/tasks.py:247`

> The `[ERROR] [ocrmypdf._exec.tesseract] [tesseract] Error during processing.` line is a **non‑fatal** Tesseract stderr artifact emitted for a page with little/no recognizable text. It did **not** abort the run: the consumer continued to thumbnailing, saved the record, and reported "consumption finished". This is called out so the reader is not misled by the word "Error".

**Worker behavior summary:** the web process returns instantly (`"OK"`); a separate `qcluster` worker picks up `consume_file`, selects `RasterisedDocumentParser` for the image MIME type, runs OCR, persists the row, deletes the working copy of the input, and returns a success string. Progress is broadcast to the `status_updates` group throughout.

### Q1(d) — What concrete signals indicate OCR work is *actively* in progress?

The most direct "OCR is running right now" signals come from the parser itself, immediately around the `ocrmypdf.ocr()` call:

- The debug line `f"Calling OCRmyPDF with args: {args}"` — `src/paperless_tesseract/parsers.py:260` — printed immediately before `ocrmypdf.ocr(**args)` — `src/paperless_tesseract/parsers.py:261`.
- The single‑thread Tesseract setting `os.environ["OMP_THREAD_LIMIT"] = "1"` — `src/paperless_tesseract/parsers.py:232` (one core per page).

Captured verbatim from the direct parser driver on Fixture A:

```text
detected mime_type = 'image/png'
original_has_text = False  (text_original len=0)
DEBUG [paperless.parsing.tesseract] Estimated DPI 120 based on image width 1000
>>> ocrmypdf.ocr() INVOKED (call #1); skip_text=True force_ocr=None redo_ocr=None output_type='pdfa' language='eng' image_dpi=150 sidecar=sidecar.txt
DEBUG [paperless.parsing.tesseract] Detected DPI for image /root/ocr_probe/fixtures/fixtureA_notext.png: 150
DEBUG [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/root/ocr_probe/fixtures/fixtureA_notext.png', 'output_file': '/tmp/paperless/paperless-6f98lbxd/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-6f98lbxd/sidecar.txt', 'image_dpi': 150}
[after parse] OMP_THREAD_LIMIT = '1'
[after parse] ocrmypdf.ocr calls for this case = 1
```

Concrete active‑OCR indicators, all confirmed at runtime:

- The exact debug line **`Calling OCRmyPDF with args: {…}`** (`parsers.py:260`) with a runtime `args` dict containing `'skip_text': True`, `'language': 'eng'`, `'output_type': 'pdfa'`, `'sidecar': '…/sidecar.txt'`, and `'image_dpi': 150`.
- **`OMP_THREAD_LIMIT = '1'`** in the process environment (`parsers.py:232`) — the Tesseract concurrency clamp that only exists because OCR is being run.
- The wrapper counter shows **`ocrmypdf.ocr calls for this case = 1`** — OCRmyPDF was actually invoked exactly once.
- Tesseract's own stderr (e.g., the `[ocrmypdf._exec.tesseract]` lines) appears while the engine is executing.

---


## Q2 — Similar image that already contains text: does OCR get *skipped*, or is the pipeline still *touched*? And how can you tell afterward?

**Fixtures: A (`fixtureA_notext.png`) vs. B (`fixtureB_withtext.png`); PDF fixture C for contrast.**

### Q2(a) — Skip vs. touch: OCR is **never skipped for an image**

The decisive fact is in `parse()`. It branches on MIME type. Only for a **PDF** does it probe for an embedded text layer; for **anything else (every image)** it hard‑codes "no text":

```python
def parse(self, document_path, mime_type, file_name=None):          # parsers.py:230
    os.environ["OMP_THREAD_LIMIT"] = "1"                            # parsers.py:232
    if mime_type == "application/pdf":                             # parsers.py:234
        text_original = self.extract_text(None, document_path)     # parsers.py:235
        original_has_text = text_original and len(text_original) > 50  # parsers.py:236
    else:                                                          # parsers.py:237
        text_original = None                                       # parsers.py:238
        original_has_text = False                                  # parsers.py:239
    if settings.OCR_MODE == "skip_noarchive" and original_has_text:# parsers.py:241
        self.log("debug", "Document has text, skipping OCRmyPDF entirely.")  # parsers.py:242
        self.text = text_original                                  # parsers.py:243
        return                                                     # parsers.py:244
```

Two independent reasons make the "skip OCRmyPDF entirely" early‑return (`parsers.py:241-244`) **impossible for an image**:

1. `original_has_text` is unconditionally `False` for a non‑PDF (`parsers.py:238-239`), and
2. the early‑return additionally requires `OCR_MODE == "skip_noarchive"`, whereas the running default is `OCR_MODE == "skip"` (see §0.3).

Therefore control always reaches `ocrmypdf.ocr(**args)` (`parsers.py:261`) for an image. I proved this empirically by printing `original_has_text` for both images and counting the `ocrmypdf.ocr` invocations:

```text
### Fixture A (image, no text)
original_has_text = False  (text_original len=0)
skip-OCRmyPDF-entirely early-return fires? False (needs OCR_MODE=='skip_noarchive' AND original_has_text; OCR_MODE='skip')
>>> ocrmypdf.ocr() INVOKED (call #1); skip_text=True …
[after parse] ocrmypdf.ocr calls for this case = 1

### Fixture B (image, rendered text)
original_has_text = False  (text_original len=0)
skip-OCRmyPDF-entirely early-return fires? False (needs OCR_MODE=='skip_noarchive' AND original_has_text; OCR_MODE='skip')
>>> ocrmypdf.ocr() INVOKED (call #2); skip_text=True …
[after parse] ocrmypdf.ocr calls for this case = 1
```

**Answer to "skip vs. touch":** the system does **not** skip OCR for the text‑bearing image; it still fully **touches** the OCR pipeline. Both A and B report `original_has_text = False` and both reach `ocrmypdf.ocr()` exactly once. The right way to frame the difference between the two images is therefore a difference in the **result** (what text OCR extracts), **not** whether OCR ran.

### Q2(b) — The mode→flag mapping, and why `skip_text=True` still OCRs an image

The default `OCR_MODE = "skip"` sets the OCRmyPDF flag `skip_text=True`:

```python
if settings.OCR_MODE == "force" or safe_fallback:      # parsers.py:155
    ocrmypdf_args["force_ocr"] = True                  # parsers.py:156
elif settings.OCR_MODE in ["skip", "skip_noarchive"]:  # parsers.py:157
    ocrmypdf_args["skip_text"] = True                  # parsers.py:158
elif settings.OCR_MODE == "redo":                      # parsers.py:159
    ocrmypdf_args["redo_ocr"] = True                   # parsers.py:160
else:                                                   # parsers.py:161
    raise ParseError(f"Invalid ocr mode: {settings.OCR_MODE}")  # parsers.py:162
```

The runtime `args` dict (quoted in Q1(d) and again below) confirms `'skip_text': True` for both images.

**Corroboration from the official OCRmyPDF documentation** (paraphrased; source URLs below). Per the OCRmyPDF "Advanced features" page, when `--mode skip` (a.k.a. `--skip-text`) is used, no OCR is performed on pages that **already have text**, and such a page is copied straight to the output. The docs state it plainly: <cite index="1-2">"If --mode skip (or --skip-text) is issued, then no image processing or OCR will be performed on pages that already have text."</cite> A raster image, however, is converted to a **single PDF page with no text layer**, so `skip_text` has *nothing to skip* and the image is OCR'd regardless. This maps one‑to‑one to the in‑repo mapping at `parsers.py:155-162`:

- `OCR_MODE "force"` → `force_ocr=True` (`parsers.py:156`) — rasterize & OCR every page unconditionally.
- `OCR_MODE "skip"`/`"skip_noarchive"` → `skip_text=True` (`parsers.py:158`) — skip only pages that already have text.
- `OCR_MODE "redo"` → `redo_ocr=True` (`parsers.py:160`) — strip existing invisible OCR text and re‑OCR.

Sources (official OCRmyPDF docs): `https://ocrmypdf.readthedocs.io/en/latest/advanced.html` (mode semantics) and `https://ocrmypdf.readthedocs.io/en/latest/cookbook.html` (sidecar behavior). The rendered docs are for a newer OCRmyPDF (17.x); the mode/sidecar semantics are stable and match the pinned **13.4.3** behavior observed here at runtime.

### Q2(c) — How to tell *after processing* that OCR actually ran on the image

Two durable, post‑hoc signals confirm OCR executed on each image.

**Signal 1 — the sidecar has no "OCR skipped" marker.** `extract_text()` reads the sidecar text file and only trusts it when the skip marker is **absent**:

```python
if "[OCR skipped on page" not in text:           # parsers.py:104
    self.log("debug", "Using text from sidecar file")  # parsers.py:107
    return post_process_text(text)
else:
    self.log("debug", "Incomplete sidecar file: discarding.")  # parsers.py:110
```

Per the OCRmyPDF Cookbook, the sidecar contains only the text of pages that were actually OCR'd; pages that already had text are excluded — <cite index="2-21,2-22">"The sidecar file contains the OCR text found by OCRmyPDF. If the document contains pages that already have text, that text will not appear in the sidecar."</cite> This is exactly why the parser inspects the sidecar for the `"[OCR skipped on page"` marker. Captured sidecar contents:

```text
# Fixture A (no-text image)
sidecar len=16; contains '[OCR skipped on page' -> False
sidecar repr: ' \n\n \n\n \n\n \n\nOO)\n'

# Fixture B (text-in-pixels image)
sidecar len=26; contains '[OCR skipped on page' -> False
sidecar repr: ' \n\n \n\n \n\n \n\nInvoice 12245\n'
```

For **both** images the marker is **absent** (`-> False`) and the parser logged `Using text from sidecar file` — meaning OCR was performed on the page. Fixture B's sidecar even contains the recognized text `Invoice 12245`.

**Signal 2 — an OCR archive PDF/A was produced.** On success the parser records `self.archive_path = archive_path` (`parsers.py:263`) and the consumer persists it into `document.archive_filename` (`src/documents/consumer.py:327-331`). Captured archive facts:

```text
# Fixture A
archive_path=/tmp/paperless/paperless-6f98lbxd/archive.pdf size=15331B PDF/A part='2' conformance='B'
# Fixture B
archive_path=/tmp/paperless/paperless-nscjn_dm/archive.pdf size=15339B PDF/A part='2' conformance='B'
```

Both images produced a **PDF/A‑2B** archive (consistent with `OCR_OUTPUT_TYPE="pdfa"` — `src/paperless/settings.py:518`), sized ~15 KB. The presence of that archive is the durable proof that the OCR pipeline ran and produced an output.

### Q2(d) — Contrast: a real text‑layer **PDF** *does* take the genuine skip path

To make the "skip is a PDF‑only path" claim concrete, Fixture C is a real PDF with an embedded text layer (>50 chars). For it, `parse()` takes the PDF branch and `original_has_text` becomes `True`:

```text
### Fixture C (PDF, real text layer)
detected mime_type = 'application/pdf'
original_has_text = True  (text_original len=207)
skip-OCRmyPDF-entirely early-return fires? False (needs OCR_MODE=='skip_noarchive' AND original_has_text; OCR_MODE='skip')
>>> ocrmypdf.ocr() INVOKED (call #3); skip_text=True … image_dpi=None
DEBUG [paperless.parsing.tesseract] Incomplete sidecar file: discarding.
self.text (len=207): 'This is a genuine embedded PDF text layer for skip_text contrast.\n\n…'
sidecar len=26; contains '[OCR skipped on page' -> True
sidecar repr: '[OCR skipped on page(s) 1]'
```

Note the differences from the images:

- `original_has_text = True` (`parsers.py:236`) because the PDF page already had a text layer of length 207 (> 50).
- The sidecar now **contains** the `[OCR skipped on page(s) 1]` marker (`-> True`), so the parser logged `Incomplete sidecar file: discarding.` (`parsers.py:110`) and fell back to reading the text from the archive.
- `image_dpi` is `None` for a PDF (images pass `image_dpi=150`).

> Even here OCRmyPDF was still *invoked* (call #3) — because the default mode is `"skip"`, not `"skip_noarchive"`, the "skip OCRmyPDF entirely" early‑return did not fire. What the PDF demonstrates is the *page‑level* skip inside OCRmyPDF (the sidecar marker), which an image can never trigger. Had `OCR_MODE` been `"skip_noarchive"`, the PDF (and only the PDF, since `original_has_text` must be true) would have short‑circuited at `parsers.py:241-244` without calling OCRmyPDF at all.

**Answer to "how can the difference be told after processing":** compare (i) the **sidecar marker** — absent for images (OCR ran), present for the text‑layer PDF (page skipped); and (ii) the **extracted `content`** — Fixture A yields the weak `OO)`, Fixture B yields the recognized `Invoice 12245`. Both images produced an archive PDF/A, so the archive's existence proves OCR ran but does *not* by itself distinguish A from B; the distinguishing artifact is the recognized text landing in `content`.

---


## Q3 — Comparing the final API responses: which fields surface OCR‑generated text vs. pre‑existing text?

**Fixtures: A (`fixtureA_notext.png`, doc id 4) vs. B (`fixtureB_withtext.png`, doc id 5).**

I issued a real authenticated `GET /api/documents/{id}/` for both documents (Django test `Client`, superuser). Both returned `HTTP 200 application/json`. Verbatim responses:

```json
### GET /api/documents/4/   (A_notext)   HTTP 200  Content-Type: application/json
{
  "added": "2026-07-01T04:48:48.807178Z",
  "archive_serial_number": null,
  "archived_file_name": "2026-07-01 fixtureA_notext.pdf",
  "content": "OO)",
  "correspondent": null,
  "created": "2026-07-01T04:48:46.477233Z",
  "document_type": null,
  "id": 4,
  "modified": "2026-07-01T04:48:48.886906Z",
  "original_file_name": "2026-07-01 fixtureA_notext.png",
  "tags": [],
  "title": "fixtureA_notext"
}
```

```json
### GET /api/documents/5/   (B_withtext)   HTTP 200  Content-Type: application/json
{
  "added": "2026-07-01T04:48:51.022330Z",
  "archive_serial_number": null,
  "archived_file_name": "2026-07-01 fixtureB_withtext.pdf",
  "content": "Invoice 12245",
  "correspondent": null,
  "created": "2026-07-01T04:48:48.894239Z",
  "document_type": null,
  "id": 5,
  "modified": "2026-07-01T04:48:51.035645Z",
  "original_file_name": "2026-07-01 fixtureB_withtext.png",
  "tags": [],
  "title": "fixtureB_withtext"
}
```

Both payloads contain exactly the twelve fields declared in `DocumentSerializer.Meta.fields` — `src/documents/serialisers.py:222-235`:

```text
id, correspondent, document_type, title, content, tags,
created, modified, added, archive_serial_number,
original_file_name, archived_file_name
```

### Q3(a) — The single field that surfaces OCR text: `content`

**There is exactly one field that carries the recognized text, and it is `content` — for *both* provenances.** The serializer exposes the model's `content` field directly (`serialisers.py:227`), which is defined as:

```python
content = models.TextField(_("content"), blank=True, help_text=…)   # models.py:117-124
```

Side‑by‑side:

| Field | Fixture A (no‑text image) | Fixture B (text‑in‑pixels image) |
|---|---|---|
| `content` | `"OO)"` (weak OCR output) | `"Invoice 12245"` (recognized text) |
| `original_file_name` | `"2026-07-01 fixtureA_notext.png"` | `"2026-07-01 fixtureB_withtext.png"` |
| `archived_file_name` | `"2026-07-01 fixtureA_notext.pdf"` | `"2026-07-01 fixtureB_withtext.pdf"` |

Both `content` values arrived via the **same** OCR path (both images had `original_has_text = False` and both invoked `ocrmypdf.ocr()` — see Q2). **There is no field that labels text as "OCR‑derived" versus "pre‑existing".** In this codebase, for images, *all* extracted text is OCR‑derived and it always lands in `content`. Even the contrast PDF (Fixture C), whose text pre‑existed as an embedded layer, would surface that text through the very same `content` field. So the answer to "which fields surface OCR‑generated text vs. pre‑existing text" is: **the same single field, `content`** — the API does not distinguish provenance.

The base accessor backing `content` is `get_text()` → `self.text` (initial state `None`) — `src/documents/parsers.py:296,342-343`.

### Q3(b) — The durable discriminator: `archived_file_name` / `has_archive_version`

What the API *does* durably signal is whether an **OCR‑produced archive** exists. The serializer computes `archived_file_name` from the model's `has_archive_version` property:

```python
original_file_name = SerializerMethodField()      # serialisers.py:207
archived_file_name = SerializerMethodField()       # serialisers.py:208
def get_original_file_name(self, obj):             # serialisers.py:210
    return obj.get_public_filename()               # serialisers.py:211
def get_archived_file_name(self, obj):             # serialisers.py:213
    if obj.has_archive_version:                    # serialisers.py:214
        return obj.get_public_filename(archive=True)   # serialisers.py:215
    else:
        return None                                # serialisers.py:217
```

```python
@property
def has_archive_version(self):                     # models.py:238
    return self.archive_filename is not None        # models.py:239
```

- `original_file_name` is the uploaded image name (ends in `.png`).
- `archived_file_name` is non‑`None` **iff** an archive was produced (`archive_filename is not None`), and it ends in `.pdf` (the OCR PDF/A).

In both image cases `archived_file_name` is **non‑null** (`…fixtureA_notext.pdf`, `…fixtureB_withtext.pdf`), because — as Q2 showed — an OCR archive PDF/A is produced for every image regardless of how much text was recognized. **Crucial nuance:** `archived_file_name` therefore discriminates *"an OCR archive exists"*, **not** *"the text is OCR‑derived"*. It is null only when no archive was generated at all. It cannot be used to tell A from B (both are non‑null); the only field that differs meaningfully between A and B is `content`.

**Answer to Q3:** OCR‑generated text and pre‑existing text both surface in the **single `content` field** (`serialisers.py:227`; `models.py:117-124`) with no provenance label; the fields that indicate an OCR *archive* was produced are `archived_file_name`/`has_archive_version` (`serialisers.py:213-217`; `models.py:238-239`), which are non‑null for both images and thus mark "archive exists," not "text came from OCR."

---


## Q4 — Weak/incomplete OCR: what is the final state, does it still count as "fully processed," and how is that reflected in the saved metadata?

**Fixture: D (`fixtureD_blank.png`) — a blank image that forces the empty‑content path.** (Fixture A produced weak‑but‑non‑empty `OO)`; the truly empty case needs a blank image.)

### Q4(a) — The fallback chain, and the terminal outcome for weak/empty OCR

When primary OCR yields no text, the parser does **not** immediately give up; it retries once with a "safe fallback" (`force_ocr`) before settling on empty content. Traced with the direct driver on Fixture D:

```text
### Fixture D (blank image): /root/ocr_probe/fixtures/fixtureD_blank.png  mime='image/png'
DEBUG [paperless.parsing.tesseract] Calling OCRmyPDF with args: {… 'skip_text': True, … 'image_dpi': 150}
ERROR [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
DEBUG [paperless.parsing.tesseract] Using text from sidecar file
WARNING [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
DEBUG [paperless.parsing.tesseract] Fallback: Calling OCRmyPDF with args: {… 'force_ocr': True, … 'image_dpi': 150}
ERROR [ocrmypdf._exec.tesseract] [tesseract] Error during processing.
RESULT self.text == '' -> True   repr=''   len=0
RESULT archive_path set -> '/tmp/paperless/paperless-akmx_ela/archive.pdf'
DEBUG [paperless.parsing.tesseract] Using text from sidecar file
WARNING [paperless.parsing.tesseract] No text was found in /root/ocr_probe/fixtures/fixtureD_blank.png, the content will be empty.
RESULT archive on disk -> True
```

Step by step, with source anchors:

1. **Primary OCR** runs with `skip_text=True` (`parsers.py:260-261`); the sidecar yields nothing.
2. The empty result trips the guard `if not self.text: raise NoTextFoundException("No text was found in the original document")` — `src/paperless_tesseract/parsers.py:266-267`.
3. The parser catches `(NoTextFoundException, InputFileError)` (`parsers.py:276`) and logs the warning `Encountered an error while running OCR: … Attempting force OCR to get the text.` — `src/paperless_tesseract/parsers.py:277-281` — then rebuilds args with `safe_fallback=True` (`parsers.py:293`), which maps to `force_ocr=True` (`parsers.py:155-156`). The verbatim `Fallback: Calling OCRmyPDF with args: {… 'force_ocr': True, …}` line comes from `src/paperless_tesseract/parsers.py:297`.
4. Text is *still* absent, and there was no original text, so the last‑resort branch logs the warning and sets empty content:

```python
if not self.text:                                   # parsers.py:316
    if original_has_text:                           # parsers.py:319
        self.text = text_original                   # parsers.py:320
    else:
        self.log(
            "warning",
            f"No text was found in {document_path}, the content will "   # parsers.py:324
            f"be empty.",                                                # parsers.py:325
        )
        self.text = ""                              # parsers.py:327
```

The warning captured verbatim — `No text was found in /root/ocr_probe/fixtures/fixtureD_blank.png, the content will be empty.` — matches `parsers.py:322-327`, and `self.text == '' -> True (repr='', len=0)` confirms `self.text = ""` (`parsers.py:327`).

> Note the archive path reported is the **primary** `archive.pdf`, not the `archive-fallback.pdf`. The fallback run OCRs into a separate temp file, but per the parser's logic the archive kept for persistence is the primary one; `RESULT archive on disk -> True` confirms an archive PDF was produced even for the blank image.

### Q4(b) — Weak/empty OCR is a **SUCCESS**, not a failure — the row is still created and the terminal `SUCCESS` frame is emitted

Empty content is **not** an error path. When run through the real `Consumer.try_consume_file()`, Fixture D still created a `Document` row and emitted the same terminal `SUCCESS` frame as the text‑bearing fixtures. Captured verbatim:

```text
### CONSUME D_blank (task_id=44444444-4444-4444-4444-444444444444)
[2026-07-01 04:48:51,045] [INFO] [paperless.consumer] Consuming fixtureD_blank.png
[2026-07-01 04:48:52,096] [WARNING] [paperless.parsing.tesseract] Encountered an error while running OCR: No text was found in the original document. Attempting force OCR to get the text.
[2026-07-01 04:48:53,136] [WARNING] [paperless.parsing.tesseract] No text was found in /root/ocr_probe/consume/fixtureD_blank.png, the content will be empty.
[2026-07-01 04:48:53,535] [INFO] [paperless.consumer] Document 2026-07-01 fixtureD_blank consumption finished
[row] id=6 content='' (len=0)
[row] archive_filename='0000006.pdf' has_archive_version=True
[row] checksum='c813d23c7c12a40925a23f34e1a0187c' mime_type='image/png' title='fixtureD_blank'
```

And its final frame (identical shape to A/B, only the ids differ):

```json
{"current_progress": 100, "document_id": 6, "filename": "fixtureD_blank.png", "max_progress": 100, "message": "finished", "status": "SUCCESS", "task_id": "44444444-4444-4444-4444-444444444444"}
```

The row is created by `_store()`:

```python
document = Document.objects.create(     # consumer.py:398
    title=(self.override_title or file_info.title)[:127],
    content=text,                       # consumer.py:400  (here text == "")
    mime_type=mime_type,
    checksum=hashlib.md5(f.read()).hexdigest(),
    created=created,
    modified=created,
    storage_type=storage_type,
)                                       # consumer.py:406
```

and the terminal frame by `self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)` — `src/documents/consumer.py:375`.

**So the final state of a weak/empty‑OCR image is:** a fully created `Document` row with `content=''` (`len=0`), a **non‑null** `archive_filename` (`0000006.pdf`, `has_archive_version=True`), a real `checksum` and `mime_type`, and a terminal `SUCCESS`/`finished` frame carrying the new `document_id`. It is text‑poor, but fully processed.

### Q4(c) — "Fully processed" is *operational*, not a stored flag (there is **no status column**)

There is **no** status/state/processing field anywhere on the `Document` model. Enumerated at runtime from the source, the model's persisted fields are:

```text
correspondent (models.py:97)       title (models.py:106)          document_type (models.py:108)
content (models.py:117)            mime_type (models.py:126)      tags (models.py:128)
checksum (models.py:135)           archive_checksum (models.py:143)  created (models.py:152)
modified (models.py:154)           storage_type (models.py:161)   added (models.py:169)
filename (models.py:176)           archive_filename (models.py:186)  archive_serial_number (models.py:196)
```

A grep for `status`/`state`/`processing` across the `Document` class body (`models.py:88-244`) returns **nothing** — there is no processing‑status column. (`storage_type` at `models.py:161` encodes unencrypted vs. GPG storage, not OCR/processing progress.)

Therefore **"fully processed" must be defined operationally**, not by reading a stored flag. The two observable criteria are:

1. **A row exists** in the documents table (`Document.objects.create(...)` — `consumer.py:398-406`), and
2. **The terminal `SUCCESS`/`finished` frame was emitted** (`consumer.py:375`), accompanied by the log line `Document … consumption finished` (`consumer.py:373`).

Fixture D satisfies both, so a text‑poor document is nonetheless "fully processed."

### Q4(d) — Contrast with genuine FAILURES

Weak/empty OCR (a `SUCCESS`) is distinct from real failures, which route through `_fail()` and broadcast a `"FAILED"` status:

```python
def _fail(self, message, log_message=None, exc_info=None):   # consumer.py:78
    self._send_progress(100, 100, "FAILED", message)         # consumer.py:79
```

Genuine failure triggers include:

- **Unsupported MIME type** — `_fail(..., MESSAGE_UNSUPPORTED_TYPE)` where `MESSAGE_UNSUPPORTED_TYPE = "unsupported_type"` (`consumer.py:44`), raised in the mime‑type guard around `src/documents/consumer.py:221-225`.
- **Duplicate checksum** — the pre‑consume duplicate check (`pre_check_duplicate`, invoked at `src/documents/consumer.py:213`) fails the file before parsing.
- **Exhausted OCR fallback** — if even `force_ocr` cannot proceed, the parser raises `ParseError` (in the fallback `except` around `src/paperless_tesseract/parsers.py:300-310`), which the consumer turns into a `_fail(...)`.

In all those cases the terminal frame carries `"status": "FAILED"` (`consumer.py:79`) and **no** `Document` row is created — the opposite of the weak‑OCR `SUCCESS` outcome, where the row exists and the frame says `"SUCCESS"`.

**Answer to Q4:** a weak/incomplete OCR result leaves `self.text = ""` (`parsers.py:327`) after a `skip_text`→`force_ocr` fallback attempt (`parsers.py:266-298`); the consumer still creates the `Document` row with `content=''` and a non‑null `archive_filename` (`consumer.py:398-406`) and emits the terminal `100/100 SUCCESS finished` frame (`consumer.py:375`). It counts as fully processed **operationally** — there is no status column (`models.py:88-244`), so "fully processed" = *row exists + `SUCCESS` frame emitted*. This is categorically different from `_fail()`/`"FAILED"` outcomes (`consumer.py:78-79`) for duplicates, unsupported types, or an exhausted OCR fallback.

---


## Coverage pass — every sub‑part re‑checked

Re‑reading each question and confirming each distinct sub‑part is explicitly answered:

**Q1 — Image with no embedded text**
- [x] *How can you see OCR has started?* → HTTP body `"OK"` + minted `task_id` UUID (`views.py:521,535`), and the `0/100 STARTING new_file` WebSocket frame (`consumer.py:202`). — Q1(a)
- [x] *What does the in‑progress state look like?* → the six discrete milestone frames `0/20/70/90/95/100` (`consumer.py:202,259,264,274,294,375`); no persisted status; plus the honest finding that the 20–70 `progress_callback` band never fires for images (comment `consumer.py:238` vs formula `consumer.py:239`). — Q1(b)
- [x] *How do the workers behave?* → web returns instantly; `qcluster` runs `consume_file` (`tasks.py:184`) → `try_consume_file` (`tasks.py:236`) → returns `"Success. New document id 7 created"` (`tasks.py:247`); worker logs `Consuming`/`Detected mime type`/`Parser: RasterisedDocumentParser`/`consumption finished`. — Q1(c)
- [x] *What signals show OCR actively in progress?* → `Calling OCRmyPDF with args: {…}` (`parsers.py:260`) with `skip_text=True`, `image_dpi=150`; `OMP_THREAD_LIMIT='1'` (`parsers.py:232`); `ocrmypdf.ocr` call count = 1. — Q1(d)

**Q2 — Similar image that already contains text**
- [x] *Skip or still touch?* → still touches; `original_has_text=False` for both images (`parsers.py:238-239`), both reach `ocrmypdf.ocr()` (`parsers.py:261`); early‑return needs `skip_noarchive` + text (`parsers.py:241-244`), impossible for an image. — Q2(a)
- [x] *Mode→flag mapping + why `skip_text` still OCRs an image* → `OCR_MODE "skip"`→`skip_text=True` (`parsers.py:157-158`); OCRmyPDF docs corroborate skip applies only to pages that already have text. — Q2(b)
- [x] *How to tell after processing?* → sidecar `"[OCR skipped on page"` marker **absent** for both images (`parsers.py:104`), and a PDF/A archive produced (~15 KB) (`parsers.py:263`); result differs (`OO)` vs `Invoice 12245`). — Q2(c)
- [x] *PDF contrast (genuine skip path)* → Fixture C `original_has_text=True` (`parsers.py:236`), sidecar marker **present** `[OCR skipped on page(s) 1]`. — Q2(d)

**Q3 — Comparing final API responses**
- [x] *Which field surfaces OCR‑generated text?* → `content` (`serialisers.py:227`; `models.py:117-124`): A=`"OO)"`, B=`"Invoice 12245"`. — Q3(a)
- [x] *Which field surfaces pre‑existing text?* → the **same** `content` field; the API attaches no provenance label. — Q3(a)
- [x] *Durable discriminator* → `archived_file_name`/`has_archive_version` (`serialisers.py:213-217`; `models.py:238-239`) — non‑null for both images (marks "archive exists," not "OCR‑derived"). — Q3(b)

**Q4 — Weak/incomplete OCR**
- [x] *What happens to the final state?* → fallback `skip_text`→`force_ocr` (`parsers.py:266-298`), then `self.text = ""` with the warning `No text was found …, the content will be empty.` (`parsers.py:322-327`). — Q4(a)
- [x] *Does it still count as fully processed?* → yes; row created with `content=''` (`consumer.py:398-406`) + terminal `SUCCESS/finished` frame (`consumer.py:375`). — Q4(b)
- [x] *How is that reflected in saved metadata?* → `content=''`, `archive_filename='0000006.pdf'`, `has_archive_version=True`, real `checksum`/`mime_type`; **no status column** exists (`models.py:88-244`), so "fully processed" is operational (row + `SUCCESS`). — Q4(c)
- [x] *Contrast with failures* → `_fail()` broadcasts `"FAILED"` (`consumer.py:78-79`) for unsupported type (`consumer.py:221-225`), duplicate (`consumer.py:213`), or exhausted fallback `ParseError` (`parsers.py:300-310`); no row created. — Q4(d)

## Anything not verifiable at runtime (flagged per R4)

- **Live browser WebSocket handshake on `ws/status/$`.** The progress frames quoted here are the **actual** payloads relayed to the `status_updates` Redis group (captured with a real `channels_redis` subscriber), which is exactly what `StatusConsumer.status_update()` forwards via `self.send(json.dumps(event["data"]))` (`src/paperless/consumers.py:33`). I did **not** additionally drive a browser through the authenticated WebSocket upgrade (`src/paperless/asgi.py:20`; auth gate `src/paperless/consumers.py:10-21`); the relay code path is cited but the socket‑level handshake itself was not exercised. The frame *contents* are verbatim and authoritative.
- **`task_id` value is randomized per run.** The captured HTTP upload minted `task_id = 'b8404260-8b59-4678-8958-f07ac3182ad0'` (`views.py:521`); this UUID differs on every run by construction. In the consumer/WebSocket captures I injected fixed sentinel task ids (`1111…`, `2222…`, `4444…`) purely to correlate frames to fixtures — the *shape* and *sequence* are authoritative, the specific id strings are illustrative.
- **Document ids differ between runs.** The direct‑consumer run produced ids 4/5/6 (Fixtures A/B/D) and the separate worker run produced id 7 (Fixture A again). Ids are assigned by the DB sequence and are not stable across runs; the invariant is that a row is created and its id is echoed in the terminal frame's `document_id`.
- **Exact OCR text for a "no‑text" image is engine‑dependent.** Fixture A yielded `content='OO)'` and Fixture B yielded `content='Invoice 12245'` (a mis‑read of the drawn `Invoice 12345`) under Tesseract 4.1.1. These exact strings depend on the image content and the pinned engine version and would vary with different pixels or a different Tesseract build; the *structural* claims (weak vs. recognized; both via the same `content` field) are what is authoritative.
- **OCRmyPDF documentation version.** The official docs currently render for OCRmyPDF 17.x, while the pinned runtime is 13.4.3. The `skip_text`/`redo_ocr`/`force_ocr` and sidecar semantics are stable across these versions and are directly corroborated by the 13.4.3 runtime behavior captured above (e.g., the `[OCR skipped on page(s) 1]` sidecar marker for the text‑layer PDF).

## Source references

- Repository (HEAD `542221a38dff06361e07976452f9aea24d210542`) — all `file:line` citations above.
- OCRmyPDF official documentation — Advanced features: `https://ocrmypdf.readthedocs.io/en/latest/advanced.html` (mode semantics `skip`/`force`/`redo`).
- OCRmyPDF official documentation — Cookbook: `https://ocrmypdf.readthedocs.io/en/latest/cookbook.html` (sidecar contains only OCR'd‑page text).
