# Paperless-ngx — How Data Moves Through Document Ingestion (Runtime-Grounded)

> This document answers, from **live runtime observation**, how a document moves through Paperless-ngx during normal ingestion: (**O1**) detection & hand-off, (**O2**) transition into parsing/classification/indexing, (**O3**) per-stage progress/completion, (**O4**) final data destination & state recording, (**O5**) duplicate tracking, and (**O6**) duplicate avoidance. Every factual claim is backed by captured output (logs, task records, WebSocket frames, database rows, on-disk files, and the Whoosh index) or is explicitly labeled **inferred** when derived from code alone. All observation flows through the **canonical watched-directory entry point** — a file dropped into `CONSUMPTION_DIR` and picked up by the running watcher/worker; no mocks, monkeypatches, debug hooks, or direct `Consumer` calls are used as evidence.

## 1. Title & Run Metadata

- **Project / commit (canonical):** `paperless-ngx` at commit `542221a38dff06361e07976452f9aea24d210542` (verified live: `git rev-parse HEAD` inside the image returned `542221a38dff06361e07976452f9aea24d210542`).
- **Source branch:** `paperless-ngx_542221a38dff` (the branch this deliverable, `blitzy/documentation/paperless-ngx_542221a38dff.md`, is named after).
- **Canonical image:** the runtime is the user-specified image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01` (Debian 11, Python 3.9). All runs used `paperless-ngx-baseline:542221a38dff`, a warmed layer built over that exact image (adding Redis, poppler-utils, migrated DB + `consumer`/`admin` users, compiled catalogs). The host (Ubuntu 25.10 / Python 3.13) is **not** used to run the stack — only the container is.
- **Runtime:** Python **3.9.23**, Django **4.0.4**, django-q **1.3.9**, Channels **3.0.4** (observed — see §1.1).
- **Task queue / broker:** django-q worker (`qcluster`) with a **Redis** broker + channel layer at `redis://localhost:6379`.
- **Run window:** the **primary** document-processing run (Appendix A.1/A.2 and the §O1–O6 transcripts) ran `2026-07-13 ~18:40:20Z` (stack up) → `2026-07-13 ~18:41:19Z` (last drop). A **verified re-run** of the identical canonical configuration on `2026-07-14 ~01:02–01:04Z` reproduced the same boundary values (§7) and supplied the finding-specific captures shown in §1.2 (ordered setup + PID-captured observer), §5.R (RGBA sibling edge), and §A.4 (safe cleanup). Each capture states which run it is from; both runs are behaviorally identical.
- **Executed as:** non-root `testuser` inside the container.
- **Scale / stability:** the happy path is run **3 times** (two `.txt` + one `.pdf`); the duplicate path is run **3 times** (two with delete OFF, one with delete ON); plus two unsupported-type edges. Boundary values are identical across repeats (§7).

### 1.1 Runtime configuration verified live (finding #7, #8)

Versions were read at runtime by importing each package inside the container (**observed**, not manifest-derived). The capture pass ran twice, hence the intentional duplicate lines:

```text
python 3.9.23
django 4.0.4
channels 3.0.4
whoosh 2.7.4
django-q 1.3.9
channels-redis 3.4.0
whoosh 2.7.4
channels 3.0.4
django 4.0.4
```

Directory/database configuration was read live from `django.conf.settings`. The **source defaults** are repo-relative (`BASE_DIR=/app/src`, so `/app/data`, `/app/media`, `/app/consume`); the `/paperless/*` paths used here are **warmed-image `PAPERLESS_*` environment overrides** (from `pl.env`), **not** application defaults (**observed**):

```text
=== SOURCE DEFAULTS (env unset) — src/paperless/settings.py:57,61-79, BASE_DIR=/app/src ===
BASE_DIR          = /app/src
default DATA_DIR       = /app/src/../data
default MEDIA_ROOT     = /app/src/../media
default CONSUMPTION_DIR= /app/src/../consume

=== EFFECTIVE VALUES (warmed-image PAPERLESS_* overrides from pl.env) ===
PAPERLESS_DATA_DIR (env)        = /paperless/data
PAPERLESS_MEDIA_ROOT (env)      = /paperless/media
PAPERLESS_CONSUMPTION_DIR (env) = /paperless/consume
effective DATA_DIR       = /paperless/data
effective MEDIA_ROOT     = /paperless/media
effective CONSUMPTION_DIR= /paperless/consume
effective DB ENGINE      = django.db.backends.sqlite3
effective DB NAME        = /paperless/data/db.sqlite3
DEBUG                    = False
CONSUMER_POLLING         = 0
CONSUMER_DELETE_DUPLICATES= False
INDEX_DIR                = /paperless/data/index
MEDIA_LOCK               = /paperless/media/media.lock
```

> **observed.** `CONSUMER_POLLING = 0` selects the inotify watcher (`src/documents/management/commands/document_consumer.py:200`); `CONSUMER_DELETE_DUPLICATES = False` is the default for the happy/duplicate-retain runs and is toggled to `True` for the delete-ON branch (§O6). The database is SQLite at `/paperless/data/db.sqlite3`.

### 1.2 Complete ordered setup + orchestrated, PID-captured, readiness-checked bring-up (finding #6, #16)

Bringing the stack up is a two-part sequence: **(A) one-time preparation** — apply migrations (the data migration creates the `consumer` user required by `set_log_entry`, `src/documents/signals/handlers.py:416`), verify/create the auth users, mint the one-time listener credential (§1.5), and generate the deterministic sample inputs — followed by **(B) launching the concurrent processes**. The full scripts are in Appendix A.3 (`prepare.sh`, `bringup.sh`, `start_asgi.sh`) and the exact copy/paste command sequence is in A.5.

The running stack is **five** concurrent long-running processes: `redis-server` (broker + channel layer), the `qcluster` django-q worker, the `document_consumer` inotify watcher, a **`tail -F` log observer**, and the `daphne` ASGI server. Every one of them either blocks or must outlive the launching shell, so each is started detached with `setsid`, **its PID is captured** to a file, and a **readiness poll** confirms it is up before proceeding (no fixed sleeps). Literal outputs (verified re-run, `2026-07-14 ~01:02Z`):

**(A) Preparation (`prepare.sh`) — migrations → users → credential → samples:**

```text
== [1/5] env loaded ==
  DATA_DIR=/paperless/data MEDIA_ROOT=/paperless/media CONSUMPTION_DIR=/paperless/consume
== [2/5] migrations ==
  Applying sessions.0001_initial... OK
unapplied_count=0
== [3/5] required auth user: consumer ==
consumer id=1 present
== [4/5] one-time random admin credential (value never shown) ==
admin one-time password set OK (value not shown)
credential stored mode: 600 (readable only by testuser)
== [5/5] deterministic sample inputs ==
pdf generated
== sample manifest (md5 / size / type) ==
  src_txt1.txt         cdbc1170025bb16ff4c9b45fabe73d54       74  ASCII text
  src_txt2.txt         7a30f529c7cdfa466193ac611bd8927a       74  ASCII text
  src_pdf.pdf          5a9d80f50fe19ce9cb2a1c6e9dca66c1     1438  PDF document, version 1.3
  src_unsupported.xyz  fbc640c19891aebe853553f1925d320d       32  ASCII text
  src_binary.txt       b3535289b2932e25650074aa6d89bf3c   147176  ELF 64-bit LSB pie executable, x86-64
  src_rgba.png         e107126c8ee9b2d8048ba689a8bb1242     7913  PNG image data, 517 x 147, 8-bit/color RGBA
  src_noalpha.png      3d3fa69e06d2069ab9d2a3bd61c930c6     6414  PNG image data, 517 x 147, 8-bit/color RGB
== prepare complete ==
```

**(B) Bring-up (`bringup.sh`: redis → qcluster → document_consumer → tail -F observer):**

```text
== [1/4] redis (broker + channel layer) ==
redis: PONG
== [2/4] qcluster (django-q worker) ==
qcluster pid=295
01:02:47 [Q] INFO Q Cluster social-violet-sad-salami starting.
== [3/4] document_consumer (inotify watcher) ==
document_consumer pid=327
[2026-07-14 01:02:48,797] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /paperless/consume
== [4/4] live log observer (tail -F, PID captured) ==
tail -F observer pid=346
tail observer alive
== bringup complete ==
```

**ASGI (`start_asgi.sh`: daphne serving the `ws/status/` WebSocket):**

```text
== [asgi] daphne (Channels ASGI serving ws/status/) ==
daphne pid=359
daphne ready: /admin/login/ http=200
[2026-07-14 01:02:50,494] [INFO] [daphne.server] Listening on TCP address 127.0.0.1:8000
== asgi ready ==
```

> **observed.** All four background processes are PID-captured — `qcluster pid=295`, `document_consumer pid=327`, `tail -F observer pid=346`, `daphne pid=359` — each written to a `*.pid` file and used for deterministic, **exact-PID** shutdown (§A.4). The `tail -F` observer is a first-class, started-and-PID-captured process (not an implicit foreground `tail`): it follows `paperless.log` into `tail.out` for the whole run. Readiness is asserted by grepping for `Q Cluster … starting`, `Using inotify …`, `tail observer alive`, and daphne's `Listening on TCP address 127.0.0.1:8000` — never a fixed sleep. These bring-up, credential, and observer captures are from the documentation's **verified re-run** of the identical canonical configuration; the primary document-processing transcripts in Appendix A.1/A.2 are from the initial canonical run (§7 confirms identical boundary values across runs).

### 1.3 observed vs inferred convention (finding #8)

- **observed** = captured directly from a running process (a log line, a task record, a WebSocket frame, a database row, an on-disk file, or a Whoosh query result during the canonical run).
- **inferred** = derived from reading the source at commit `542221a38dff` without a corresponding runtime capture. A verbatim source excerpt is not a third category: it is quoted with a full `src/...:line` citation and is the *basis* for an inferred behavioral claim, so any claim that rests on code-reading rather than a runtime capture is labelled **inferred** — a quoted source excerpt is never tagged as if it were itself a runtime observation.

These are the material **inferred** claims in this document; everything else is **observed**:

1. **O4 failure/rollback semantics** (§O4) — that the `transaction.atomic()` block rolls back only the DB writes while the media-file writes, Whoosh add, and source `unlink` are non-transactional side effects. Forcing a mid-transaction failure would require non-canonical manipulation of the pipeline, which is disallowed.
2. **The watcher-path WebSocket `task_id` origin** (§O3) — that it is a worker-minted `uuid4`, inferred from `src/documents/consumer.py:200` combined with the watcher's `task_id`-less enqueue. (The concrete UUID *values*, and the observed fact that the django-q **Task PK** differs from the WebSocket `task_id` on the watcher path, are **observed** — §7.)
3. **The causal mechanism of the RGBA edge** (§5.R) — that the parser's in-place alpha-layer rewrite is the *cause* of the double enqueue and post-parser checksum. Its **effects** (two enqueues, two `task_id`s, post-parser checksum, late duplicate rejection, and the clean no-alpha control) are all **observed** at runtime and reproduced across two runs.

Every code citation uses a full `src/...:line` path so a reader can reproduce it against this commit.

### 1.4 Documented deviations from the literal runbook (canonical-equivalent)

1. **ASGI server = daphne, not `runserver`.** `python manage.py runserver` in this image starts a plain `WSGIServer` (the Channels ASGI runserver override is not active) that does **not** upgrade the `ws/status/` handshake — a raw upgrade request is answered with `302 → /accounts/login/?next=/ws/status/` (a redirect-following client then lands on the login page, HTTP `200`), never a `101` switch. The authenticated WebSocket capture therefore uses **daphne 3.0.2** (`daphne -b 127.0.0.1 -p 8000 paperless.asgi:application`), which loads the same `paperless.asgi:application` and the same `ws/status/` route (`src/paperless/urls.py:137`), so the observed frames — and the unauthenticated negative control (§O3) — are canonical. On daphne the unauthenticated handshake is refused with HTTP `403` (daphne logs `WSREJECT`); the `runserver` `302 → login → 200` chain is retained in §O3 only as an explicitly-labeled non-canonical contrast.
2. **`curl` is absent** in the image; HTTP readiness is polled with a short Python `urllib` probe instead.
3. **Directories are `/paperless/*`** via warmed-image `PAPERLESS_*` env overrides; the source defaults are `/app/{data,media,consume}` (§1.1). This changes only paths, not behavior.

### 1.5 One-time credential handling (finding #3, #13, #14)

No password is hard-coded or printed anywhere in this document or the harness. At runtime a **one-time random** admin password is generated **in-process**, applied with `set_password(...)`, and written to a mode-`600` credential file as an `export PL_ADMIN_PW=…` line — the value is never echoed, never passed on argv, and never printed. The listener reads it **only** from the `PL_ADMIN_PW` environment variable. This is the exact, copy/paste-safe block (step [4/5] of `prepare.sh`, §A.3) — no ellipses, fully quoted:

```bash
OBS=/home/testuser/obs
OBS="$OBS" python - <<'PY'
import os, secrets
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django; django.setup()
from django.contrib.auth import get_user_model
U = get_user_model()
pw = secrets.token_urlsafe(18)                 # one-time random, stays in-process
u, _ = U.objects.get_or_create(
    username="admin", defaults={"is_staff": True, "is_superuser": True})
u.is_staff = True; u.is_superuser = True
u.set_password(pw); u.save()
path = os.path.join(os.environ["OBS"], ".adminpw")
fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC | os.O_NOFOLLOW, 0o600)
with os.fdopen(fd, "w") as f:
    f.write("export PL_ADMIN_PW=" + pw + "\n")  # written, never printed
print("admin one-time password set OK (value not shown)")
PY
chmod 600 "$OBS/.adminpw"
```

Consumers then source the file to load the variable without echoing it — `source /home/testuser/obs/.adminpw` — and the listener is launched with that variable already in its environment. Literal output:

```text
admin one-time password set OK (value not shown)
credential stored mode: 600 (readable only by testuser)
```

> **observed.** The listener (`ws_listen.py`, §A.3) sets `requests.Session.trust_env = False` for deterministic loopback auth, prints **only** HTTP status codes and an `authenticated=<bool>` — never a cookie, session id, or password. Frame output is written to an `O_CREAT|O_EXCL|O_NOFOLLOW` file (mode `600`) inside a per-run `mktemp -d` directory (mode `700`), addressing predictable-temp-path risk (CWE-377/CWE-367).

## 2. TL;DR — one-line answers

The direct answer to each thread, as **observed** for the canonical text/PDF/no-alpha inputs, with the RGBA edge (§5.R) and the one **inferred** boundary (§O4) called out in the right-hand column:

| Thread | Direct answer (observed) | Scope / edge |
|--------|--------------------------|--------------|
| **O1 — Detection & hand-off** | the `document_consumer` inotify watcher logs `Adding <path> to the task queue.` (`src/documents/management/commands/document_consumer.py:85`) and enqueues the django-q task `documents.tasks.consume_file` (`src/documents/management/commands/document_consumer.py:86`, no `task_id`); the `qcluster` worker logs `Consuming <file>`. | One enqueue per drop for text/PDF/no-alpha; a supported **RGBA** image is rewritten in place (alpha removal) and enqueues a **second** time — two tasks for one drop (§5.R). |
| **O2 — Parse/classify/index** | the worker runs `Consumer.try_consume_file`, logging `Detected mime type: …` → `Parser: …` → `Parsing …` (`src/documents/consumer.py:215-223`); after the DB row is stored, `document_consumption_finished` (`src/documents/apps.py:22-27`) fans out to matching, the admin `LogEntry`, and the Whoosh index add. | Uniform across all observed inputs. |
| **O3 — Progress/completion** | `Consumer._send_progress` (`src/documents/consumer.py:56-77`) broadcasts JSON to the `status_updates` group; a client on `ws/status/` sees `STARTING/new_file(0)` → `WORKING/parsing_document(20)` → `generating_thumbnail(70)` → `parse_date(90)` → `save_document(95)` → `SUCCESS/finished(100)` under a **single `task_id`** per text/PDF/no-alpha drop. | RGBA emits **two interleaved `task_id` streams** for one drop — one `SUCCESS`, one `FAILED` (§5.R). |
| **O4 — Final destination & state** | a `documents_document` row (MD5 `checksum`), files under `originals/` (+ `archive/`, `thumbnails/`), an admin `LogEntry` attributed to user `consumer`, a Whoosh index entry, and a `SUCCESS` frame + task result `Success. New document id N created`. | `transaction.atomic()` covers DB writes only; the media writes, the Whoosh add, and the source `unlink` are non-transactional side effects (**inferred**, §O4). |
| **O5 — Duplicate tracking** | an MD5 of the file bytes is compared against existing `Document.checksum`/`archive_checksum` before parsing (`src/documents/consumer.py:102-113`); `checksum` is a `unique=True` column (`src/documents/models.py:135`); the stored checksum equalled the **input** bytes for text/PDF/no-alpha. | the checksum is computed on `self.path` **after** parsing (`src/documents/consumer.py:397-403`), so an RGBA rewrite makes the stored checksum the **post-parser** bytes (§5.R). |
| **O6 — Duplicate avoidance** | a match logs `[ERROR] … Not consuming <file>: It is a duplicate.`, emits a `FAILED/document_already_exists` frame, records a `success=False` task, and leaves `Document`/`LogEntry`/index counts unchanged; with `CONSUMER_DELETE_DUPLICATES=True` the duplicate source is unlinked (`src/documents/consumer.py:108-109`); rejection is **pre-parse** for text/PDF/no-alpha. | for RGBA the original bytes ≠ the stored post-parser checksum, so the re-drop is **not** rejected up front — caught later as `document_already_exists` + a DB `UNIQUE constraint failed: documents_document.checksum`; the document count still does not increase (§5.R). |

## 3. End-to-end flow (observed causal order)

The causal order below was reconstructed from the timestamps in the captured `paperless.log`, worker console, and WebSocket transcript (Appendices A.1–A.2).

```mermaid
flowchart TD
    A["New file in CONSUMPTION_DIR"] --> B["document_consumer watcher (inotify)"]
    B -->|"'Adding <file> to the task queue.' (src/documents/management/commands/document_consumer.py:85)"| C["django-q async_task<br/>documents.tasks.consume_file (src/documents/management/commands/document_consumer.py:86, no task_id)"]
    C --> D["qcluster worker: Consumer.try_consume_file()<br/>task_id = uuid4() (src/documents/consumer.py:200)"]
    D --> E["pre_check_duplicate(): MD5 vs Document.checksum/archive_checksum (src/documents/consumer.py:102-113)"]
    E -->|"duplicate"| F["_fail: 'It is a duplicate.'<br/>FAILED / document_already_exists"]
    E -->|"new"| G["MIME detect -> parser dispatch -> parse/OCR (src/documents/consumer.py:215-223)"]
    G --> H["transaction.atomic (src/documents/consumer.py:298): _store() DB row + signal fan-out (LogEntry, Whoosh add)"]
    H --> I["FileLock(MEDIA_LOCK) (src/documents/consumer.py:315): write originals/archive/thumbnail"]
    I --> J["document.save() (src/documents/consumer.py:346); os.unlink(source) (src/documents/consumer.py:350)"]
    J --> K["'Document <doc> consumption finished' (src/documents/consumer.py:373)<br/>SUCCESS / finished / document_id=N (src/documents/consumer.py:375)"]
```

> **inferred (side-effect boundary).** Only the DB writes inside `transaction.atomic()` (nodes **H**) roll back on exception. The media writes (**I**), the Whoosh index add (part of **H**'s signal fan-out but a filesystem side effect), and the source `unlink` (**J**) are **not** transactionally compensated — see §O4.

## 4. Environment & methodology

- **Prerequisites satisfied before any run:** Redis broker up; migrations applied; the auth user **`consumer`** present (required by `set_log_entry`, `src/documents/signals/handlers.py:416`, which calls `User.objects.get(username="consumer")`). In this image the `consumer` user is created by migration and observed as `id=1` in every `LogEntry` (§O4).
- **Pristine before-state (observed):** before the first drop, there are **0** documents, **0** `LogEntry` rows, and the Whoosh `INDEX_DIR` does **not exist**:

```text
=== Unapplied migrations? (empty = all applied) ===
unapplied_count=0
=== DB pristine check ===
DOC_COUNT 0
LOGENTRY_COUNT 0
=== INDEX_DIR exists before any run? ===
ls: cannot access '/paperless/data/index': No such file or directory
INDEX_DIR absent
=== harness deps ===
websockets 10.3
requests 2.27.1
```

- **Snapshot side-effect disclosure (finding #18).** The state snapshotter reports the index as `INDEX_DIR_ABSENT` using `os.path.isdir(settings.INDEX_DIR)` and deliberately **does not call** `index.open_index()` before the first run, because `open_index()` **creates** `INDEX_DIR` and an empty index if absent (`src/documents/index.py:52-61`) and would therefore mutate the before-state:

```python
def open_index(recreate=False):
    try:
        if exists_in(settings.INDEX_DIR) and not recreate:
            return open_dir(settings.INDEX_DIR, schema=get_schema())
    except Exception:
        logger.exception("Error while opening the index, recreating.")

    if not os.path.isdir(settings.INDEX_DIR):
        os.makedirs(settings.INDEX_DIR, exist_ok=True)
    return create_in(settings.INDEX_DIR, get_schema())
```

> **observed + inferred.** `INDEX_DIR_ABSENT` before the first run is **observed**; that `open_index()` would create the directory is **inferred** from the code above and is precisely why the snapshotter avoids calling it until the index legitimately exists.

## 5. O1–O6 — direct answers with live evidence

### O1 — Detection & hand-off

**Direct answer.** A new file in `CONSUMPTION_DIR` is detected by the `document_consumer` **inotify** watcher, which logs the hand-off and enqueues the django-q task `documents.tasks.consume_file`; a `qcluster` worker then picks the task up and begins consuming. The three log lines that mark detection → queue → pickup (**observed**):

```text
[2026-07-13 18:40:22,365] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /paperless/consume
[2026-07-13 18:40:44,736] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_run1.txt to the task queue.
[2026-07-13 18:40:44,885] [INFO] [paperless.consumer] Consuming sample_run1.txt
```

The watcher's enqueue call passes **no `task_id`** (watcher path); it supplies only the file path, optional tag ids, and a `task_name` (**inferred** from the enqueue signature quoted below, `src/documents/management/commands/document_consumer.py:85-90`; corroborated at runtime because the django-q **Task PK** differs from the WebSocket `task_id`, §7):

```python
        logger.info(f"Adding {filepath} to the task queue.")
        async_task(
            "documents.tasks.consume_file",
            filepath,
            override_tag_ids=tag_ids if tag_ids else None,
            task_name=os.path.basename(filepath)[:100],
        )
```

**All three ingestion entry points converge on the same task (finding #19).** The watched-directory drop is the canonical one; the other two are noted only for convergence:

- **Watcher (canonical):** `src/documents/management/commands/document_consumer.py:86` — `async_task("documents.tasks.consume_file", filepath, …)` — **no `task_id`**.
- **REST upload:** `src/documents/views.py:521` sets `task_id = str(uuid.uuid4())`, then `src/documents/views.py:523` — `async_task("documents.tasks.consume_file", temp_filename, …, task_id=task_id)` — **task_id supplied at enqueue**.
- **IMAP mail:** `src/paperless_mail/mail.py:336` — `async_task("documents.tasks.consume_file", path=temp_filename, …)`.

**Hand-off actually crossing the process boundary (observed).** The worker console (`qcluster.out`) shows django-q's `Process-1:N processing [<file>]` immediately followed by the consumer's `Consuming <file>` in `paperless.log` — proving the task left the watcher and executed in the worker (leading `N:` are `grep -n` source-line numbers):

```text
17:18:40:44 [Q] INFO Process-1:1 processing [sample_run1.txt]
18:[2026-07-13 18:40:44,885] [INFO] [paperless.consumer] Consuming sample_run1.txt
26:18:40:50 [Q] INFO Process-1:2 processing [whiskey-zebra-virginia-white]
29:18:40:50 [Q] INFO Process-1:3 processing [item-three-oranges-mirror]
32:18:40:50 [Q] INFO Process-1:4 processing [beryllium-fix-oranges-friend]
35:18:40:50 [Q] INFO Process-1:5 processing [autumn-fish-lamp-whiskey]
41:18:40:50 [Q] INFO Process-1:6 processing [sample_run2.txt]
46:[2026-07-13 18:40:50,767] [INFO] [paperless.consumer] Consuming sample_run2.txt
60:18:40:56 [Q] INFO Process-1:7 processing [sample_run3.pdf]
61:[2026-07-13 18:40:56,676] [INFO] [paperless.consumer] Consuming sample_run3.pdf
68:18:41:03 [Q] INFO Process-1:8 processing [sample_dup1.txt]
86:18:41:08 [Q] INFO Process-1:9 processing [sample_dup2.txt]
```

> **observed.** django-q also schedules internal maintenance tasks with random names (e.g. `whiskey-zebra-virginia-white`); the document tasks are the ones named after the dropped file (`sample_run1.txt`, `sample_run2.txt`, `sample_run3.pdf`, …). The watcher entry point is defined at `src/documents/management/commands/document_consumer.py:54` (extension pre-check) and `src/documents/management/commands/document_consumer.py:85-86` (log + enqueue); the task itself is `documents.tasks.consume_file` (`src/documents/tasks.py:184`), which calls `Consumer().try_consume_file(...)` (`src/documents/tasks.py:236`).

> **Scope caveat (observed).** Every drop above produced exactly **one** `Adding … to the task queue.` line and **one** enqueue. This one-drop→one-enqueue relationship held for the text, PDF, and non-alpha image inputs. It does **not** hold for a supported **RGBA** image, whose parser rewrites the file in place and triggers a second enqueue for the same path; that observed edge is documented in **§5.R**.

### O2 — Transition into parsing / classification / indexing

**Direct answer.** Inside the worker, `Consumer.try_consume_file` logs an ordered sequence that marks the move from detection into parsing, and — after the row is stored — the `document_consumption_finished` signal fans out to classification/matching, the admin `LogEntry`, and the Whoosh index. The **task name** is `documents.tasks.consume_file`; the **state changes** are the `paperless.consumer` log lines below.

**Observed stage sequence — `.txt` (document id 1):**

```text
[2026-07-13 18:40:44,885] [INFO] [paperless.consumer] Consuming sample_run1.txt
[2026-07-13 18:40:44,889] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-13 18:40:44,893] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-13 18:40:44,895] [DEBUG] [paperless.consumer] Parsing sample_run1.txt...
[2026-07-13 18:40:44,896] [DEBUG] [paperless.consumer] Generating thumbnail for sample_run1.txt...
[2026-07-13 18:40:44,917] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/paperless/paperless-8n4y3jig/thumb.png -out /tmp/paperless/paperless-8n4y3jig/thumb_optipng.png
[2026-07-13 18:40:45,523] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:40:45,527] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:40:45,547] [DEBUG] [paperless.consumer] Deleting file /paperless/consume/sample_run1.txt
[2026-07-13 18:40:45,570] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/paperless/paperless-8n4y3jig
[2026-07-13 18:40:45,571] [INFO] [paperless.consumer] Document 2026-07-13 sample_run1 consumption finished
```

**Observed stage sequence — `.pdf` (document id 3), showing OCR dispatch:**

```text
[2026-07-13 18:40:56,676] [INFO] [paperless.consumer] Consuming sample_run3.pdf
[2026-07-13 18:40:56,676] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 18:40:56,679] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 18:40:56,681] [DEBUG] [paperless.consumer] Parsing sample_run3.pdf...
[2026-07-13 18:40:56,702] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /paperless/consume/sample_run3.pdf
[2026-07-13 18:40:56,771] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/paperless/consume/sample_run3.pdf', 'output_file': '/tmp/paperless/paperless-fxdyzfwb/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-fxdyzfwb/sidecar.txt'}
[2026-07-13 18:40:57,685] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 18:40:57,685] [DEBUG] [paperless.consumer] Generating thumbnail for sample_run3.pdf...
[2026-07-13 18:40:57,689] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-fxdyzfwb/archive.pdf[0] /tmp/paperless/paperless-fxdyzfwb/convert.png
[2026-07-13 18:40:57,941] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-fxdyzfwb/convert.png -out /tmp/paperless/paperless-fxdyzfwb/thumb_optipng.png
[2026-07-13 18:40:58,318] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:40:58,321] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:40:58,340] [DEBUG] [paperless.consumer] Deleting file /paperless/consume/sample_run3.pdf
[2026-07-13 18:40:58,364] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-fxdyzfwb
[2026-07-13 18:40:58,365] [INFO] [paperless.consumer] Document 2026-07-13 sample_run3 consumption finished
```

**What the log lines map to (citations):**

- `Consuming <file>` — `src/documents/consumer.py:215`.
- `Detected mime type: <t>` — `src/documents/consumer.py:221` (the value comes from `magic.from_file(..., mime=True)` at `src/documents/consumer.py:219`).
- `Parser: <ParserClass>` / `Parsing <file>…` — parser dispatch via `get_parser_class_for_mime_type` (`src/documents/consumer.py:223`); `.txt` → `TextDocumentParser`, `.pdf` → `RasterisedDocumentParser` (which calls OCRmyPDF, visible in the OCR args block above).
- `Document classification model does not exist (yet), not performing automatic matching.` — the classifier is invoked but, with no trained model present, performs no auto-matching (`paperless.classifier`). This is the **classification** stage, executed even when the model is absent.
- **Signal fan-out (classification/matching + LogEntry + index).** After `_store()`, `document_consumption_finished.send(...)` (`src/documents/consumer.py:306`) triggers the receivers connected in `src/documents/apps.py:22-27`: `add_inbox_tags`, `set_correspondent` (`src/documents/signals/handlers.py:35`), `set_document_type`, `set_tags` (`src/documents/signals/handlers.py:168`), `set_log_entry` (`src/documents/signals/handlers.py:413`), and `add_to_index` (`src/documents/signals/handlers.py:428`, which calls `src/documents/index.py:118 add_or_update_document`). The signals are declared in `src/documents/signals/__init__.py:3-5`.

> **observed + correction.** The **indexing** stage is confirmed by the post-run Whoosh queries in §O4 (a search returns the new document ids). The unsupported binary's detected MIME is **libmagic-version-dependent**: the canonical image ships **libmagic 5.39** (`file-5.39`, Debian package `1:5.39-3+deb11u1`), whose `magic.from_file(…, mime=True)` — the exact call at `src/documents/consumer.py:219` — returns `application/x-pie-executable` for the PIE-ELF sample (`/bin/ls`; verified at runtime via both `python3 -c "import magic; print(magic.from_file('/bin/ls', mime=True))"` and `file --mime-type -b /bin/ls`, deterministic across runs). Older libmagic (≤ 5.38) returns `application/x-sharedlib`, and a still-earlier draft of this document recorded `application/x-executable`; both are non-canonical for this image. The **actual observed** value in the canonical runtime is `application/x-pie-executable` (see the edge in §5.E and the log at `paperless.consumer`), corrected here.

### O3 — Per-stage progress / completion

**Direct answer.** Progress is broadcast by `Consumer._send_progress` (`src/documents/consumer.py:56-77`), which builds a JSON payload and sends it to the Channels group `status_updates`; the `StatusConsumer` WebSocket at `ws/status/` (`src/paperless/urls.py:137`) forwards each payload to connected clients. The payload shape (**observed** live in the §A.2 frames; built by the source below, `src/documents/consumer.py:56-77`):

```python
    def _send_progress(
        self,
        current_progress,
        max_progress,
        status,
        message=None,
        document_id=None,
    ):
        payload = {
            "filename": os.path.basename(self.filename) if self.filename else None,
            "task_id": self.task_id,
            "current_progress": current_progress,
            "max_progress": max_progress,
            "status": status,
            "message": message,
            "document_id": document_id,
        }
        async_to_sync(self.channel_layer.group_send)(
            "status_updates",
            {"type": "status_update", "data": payload},
        )

    def _fail(self, message, log_message=None, exc_info=None):
        self._send_progress(100, 100, "FAILED", message)
        self.log("error", log_message or message, exc_info=exc_info)
```

**Authentication evidence (observed, no secrets).** The authenticated login sequence for the canonical run (the same session that captured the frames below at `18:40:42`), printed as status codes and a boolean only:

```text
--- listener auth (no secrets) ---
LOGIN_GET /admin/login/ status=200 csrf_present=True
LOGIN_POST /admin/login/ status=302 authenticated=True
ADMIN_CHECK /admin/ status=200
AUTH_OK authenticated=True
18:40:42 WS_CONNECTED
```

> **observed.** `LOGIN_GET … 200`, `LOGIN_POST … 302 authenticated=True`, `ADMIN_CHECK /admin/ 200`, then `WS_CONNECTED` at `18:40:42` — matching the frame timestamps below. No cookie or password value is printed.

**Negative control (observed on canonical daphne).** An unauthenticated probe is refused the handshake and receives no frames:

```text
UNAUTH_RESULT rejected_http_status=403
```

daphne records the rejection on its own console as it happens (one `WSCONNECTING → WSREJECT → WSDISCONNECT` cycle per attempt):

```text
[2026-07-13 21:15:01,727] [INFO] [daphne.server] Listening on TCP address 127.0.0.1:8000
127.0.0.1:49006 - - [13/Jul/2026:21:15:13] "WSCONNECTING /ws/status/" - -
127.0.0.1:49006 - - [13/Jul/2026:21:15:13] "WSREJECT /ws/status/" - -
127.0.0.1:49006 - - [13/Jul/2026:21:15:13] "WSDISCONNECT /ws/status/" - -
```

> **observed.** On the canonical **daphne** ASGI transport an unauthenticated `ws/status/` handshake is refused with HTTP `403` (stable across three consecutive probes), and daphne's console records the `WSCONNECTING → WSREJECT → WSDISCONNECT` cycle for each attempt. This is the runtime manifestation of the server-side guard `raise DenyConnection()` at `src/paperless/consumers.py:15` — reached inside `def connect` at `src/paperless/consumers.py:13` because `_authenticated()` (`src/paperless/consumers.py:10-11`) is `False` for an anonymous scope. The rejection is **observed** (the `403` and `WSREJECT` are captured directly from the running server), not merely inferred, and no `status_updates` frame is ever delivered without an authenticated session.
>
> **Non-canonical contrast (labeled).** The Django `runserver` dev transport in this image is a plain `WSGIServer` (banner `Starting development server …`, Django 4.0.4) that does **not** upgrade `ws/status/`: a raw handshake returns `302 Found` with `Location: /accounts/login/?next=/ws/status/`, which a redirect-following client resolves to the login page (`200`). An earlier draft reported `rejected_http_status=200`; that value is the `runserver` `302 → login → 200` redirect-chain artifact — a dev-transport quirk, **not** a `101` upgrade and **not** the canonical rejection. The canonical negative-control value is daphne's `403` above.

**Observed per-stage frames — happy path (`sample_run1.txt` → document id 1):**

```text
18:40:42 WS_CONNECTED
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
```

> **observed.** The frame sequence is a fixed state machine keyed by the `MESSAGE_*` constants and `_send_progress` percentages: `STARTING/new_file(0)` (`src/documents/consumer.py:202`) → `WORKING/parsing_document(20)` → `generating_thumbnail(70)` → `parse_date(90)` → `save_document(95)` → `SUCCESS/finished(100)` (`src/documents/consumer.py:375`), with `document_id` `null` until the terminal `SUCCESS` frame carries the new id (`1`). The `StatusConsumer.status_update` handler sends each payload with `self.send(json.dumps(event["data"]))` (`src/paperless/consumers.py:29-33`).

**Task-id origin (finding #17).** Every frame above carries `task_id "cf840be1-…"`. On the **canonical watcher path this UUID is generated inside the worker** when `Consumer` processing starts — `self.task_id = task_id or str(uuid.uuid4())` (**inferred** from source, `src/documents/consumer.py:200`) — because the watcher enqueued **without** a `task_id`:

```python
        self.filename = override_filename or os.path.basename(path)
        self.override_title = override_title
        self.override_correspondent_id = override_correspondent_id
        self.override_document_type_id = override_document_type_id
        self.override_tag_ids = override_tag_ids
        self.task_id = task_id or str(uuid.uuid4())

        self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)

        # this is for grouping logging entries for this particular file
```

> **observed + inferred.** That the watcher path's `task_id` is a worker-generated UUID is **inferred** from `src/documents/consumer.py:200` combined with the watcher's `task_id`-less enqueue (`src/documents/management/commands/document_consumer.py:86`); by contrast the REST path supplies its own `task_id` at enqueue (**inferred** from `src/documents/views.py:521`). The concrete UUID value in the frames is **observed**.

> **Scope caveat (observed).** The single-`task_id` progression shown above is what an authenticated client saw for each text/PDF/no-alpha drop. A supported **RGBA** image produces **two** overlapping `task_id` streams for one drop (the in-place alpha rewrite re-enqueues the path): one stream advances to `SUCCESS/finished`, the other terminates in `FAILED`. The complete interleaved two-`task_id` transcript is captured in **§5.R**.

### O4 — Final data destination & state recording

**Direct answer.** After a successful consume, the document's data ends up in **five** durable places, and completion is recorded in **two** more (task result + WebSocket). All values below are from the canonical run (documents `1`, `2`, `3`).

**(1) Database — `documents_document` rows (observed):**

```text
===== documents_document rows =====
id=1 title='sample_run1' mime=text/plain storage=unencrypted filename=0000001.txt archive_filename=None
   checksum=cdbc1170025bb16ff4c9b45fabe73d54 archive_checksum=None
   created=2026-07-13 18:40:43.728791+00:00 added=2026-07-13 18:40:45.528112+00:00
   content='Observation happy path TXT run 1 marker BLITZYONE ts=2026-07-13T18:40:40Z\n'
id=2 title='sample_run2' mime=text/plain storage=unencrypted filename=0000002.txt archive_filename=None
   checksum=7a30f529c7cdfa466193ac611bd8927a archive_checksum=None
   created=2026-07-13 18:40:49.625847+00:00 added=2026-07-13 18:40:51.412672+00:00
   content='Observation happy path TXT run 2 marker BLITZYTWO ts=2026-07-13T18:40:40Z\n'
id=3 title='sample_run3' mime=application/pdf storage=unencrypted filename=0000003.pdf archive_filename=0000003.pdf
   checksum=5b5e4b3dfda36bbccfc3be6e670d3b31 archive_checksum=dd80cdc499dffed5d57e7961edf34229
   created=2026-07-13 18:40:55.536902+00:00 added=2026-07-13 18:40:58.321796+00:00
   content='Blitzy PDF happy path PDPFMARKER'
```

**(2) On-disk media — originals (all), plus archive & thumbnail (observed):**

```text
media files:
/paperless/media/documents/archive/0000003.pdf
/paperless/media/documents/originals/0000001.txt
/paperless/media/documents/originals/0000002.txt
/paperless/media/documents/originals/0000003.pdf
/paperless/media/documents/thumbnails/0000001.png
/paperless/media/documents/thumbnails/0000002.png
/paperless/media/documents/thumbnails/0000003.png
```

> **observed.** Text documents get an `originals/` file and a `thumbnails/` PNG; the PDF additionally gets an `archive/` PDF/A (`0000003.pdf`) and its `archive_checksum` is populated (`dd80cdc4…`), while the `.txt` rows have `archive_filename=None`.

**(3) Admin audit — `LogEntry` attributed to user `consumer` (observed):**

```text
===== admin LogEntry rows (O4 audit) =====
id=1 action_flag=1 object_id=1 object_repr='2026-07-13 sample_run1' user=consumer(id=1) content_type=documents|document action_time=2026-07-13 18:40:45.533047+00:00
id=2 action_flag=1 object_id=2 object_repr='2026-07-13 sample_run2' user=consumer(id=1) content_type=documents|document action_time=2026-07-13 18:40:51.417534+00:00
id=3 action_flag=1 object_id=3 object_repr='2026-07-13 sample_run3' user=consumer(id=1) content_type=documents|document action_time=2026-07-13 18:40:58.327354+00:00
```

> **observed.** Each row is `action_flag=1` (ADDITION), `user=consumer(id=1)`, `content_type=documents|document`, written by `set_log_entry` (`src/documents/signals/handlers.py:413-416`).

**(4) Whoosh full-text index (observed), including an honest OCR artifact:**

```text
===== Whoosh index (O4 searchability) =====
INDEX_DOCCOUNT 3
query 'BLITZYONE': hits=1 ids=[1]
query 'BLITZYTWO': hits=1 ids=[2]
query 'PDFMARKER': hits=0 ids=[]
```

The canonical snapshot searched the literal token `PDFMARKER` and got `hits=0` — **not** an indexing failure. The PDF's OCR-extracted content is `…PDPFMARKER` (see the id=3 row above: OCR read the rendered `PDFMARKER` as `PDPFMARKER`). Re-querying the **actual indexed token** confirms the PDF is indexed and searchable (**observed**):

```text
query 'PDPFMARKER' hits=1 ids=[3]
query 'Blitzy'     hits=1 ids=[3]
query 'happy'      hits=3 ids=[3, 1, 2]
query 'BLITZYONE'  hits=1 ids=[1]
```

> **observed.** `INDEX_DOCCOUNT 3` and the queries `PDPFMARKER→[3]`, `Blitzy→[3]`, `happy→[3,1,2]`, `BLITZYONE→[1]` prove all three documents are indexed. The `add_to_index` receiver (`src/documents/signals/handlers.py:428` → `src/documents/index.py:118`) is what populates it.

**(5)+(6) Completion records — task result + terminal WebSocket frame (observed):**

```text
name=sample_run1.txt func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
name=sample_run2.txt func=documents.tasks.consume_file success=True result='Success. New document id 2 created'
name=sample_run3.pdf func=documents.tasks.consume_file success=True result='Success. New document id 3 created'
```

> **observed.** django-q stores `success=True result='Success. New document id N created'` for each (the return string is built at `src/documents/tasks.py:247`); the matching terminal `SUCCESS/finished` frame carrying `document_id=N` is in §O3 / Appendix A.2. The final log line `Document <doc> consumption finished` is emitted at `src/documents/consumer.py:373`.

#### O4 correction — transactional boundaries (finding #1)

The persistence block is a single `try/except` around `with transaction.atomic():` (**inferred** from source, `src/documents/consumer.py:296-350`):

```python
        # in the system. This will be a transaction and reasonably fast.
        try:
            with transaction.atomic():

                # store the document.
                document = self._store(text=text, date=date, mime_type=mime_type)

                # If we get here, it was successful. Proceed with post-consume
                # hooks. If they fail, nothing will get changed.

                document_consumption_finished.send(
                    sender=self.__class__,
                    document=document,
                    logging_group=self.logging_group,
                    classifier=classifier,
                )

                # After everything is in the database, copy the files into
                # place. If this fails, we'll also rollback the transaction.
                with FileLock(settings.MEDIA_LOCK):
                    document.filename = generate_unique_filename(document)
                    create_source_path_directory(document.source_path)

                    self._write(document.storage_type, self.path, document.source_path)

                    self._write(
                        document.storage_type,
                        thumbnail,
                        document.thumbnail_path,
                    )

                    if archive_path and os.path.isfile(archive_path):
                        document.archive_filename = generate_unique_filename(
                            document,
                            archive_filename=True,
                        )
                        create_source_path_directory(document.archive_path)
                        self._write(
                            document.storage_type,
                            archive_path,
                            document.archive_path,
                        )

                        with open(archive_path, "rb") as f:
                            document.archive_checksum = hashlib.md5(
                                f.read(),
                            ).hexdigest()

                # Don't save with the lock active. Saving will cause the file
                # renaming logic to aquire the lock as well.
                document.save()

                # Delete the file only if it was successfully consumed
                self.log("debug", "Deleting file {}".format(self.path))
                os.unlink(self.path)
```

**What actually rolls back, and what does not (grounded in the code above):**

- `transaction.atomic()` (`src/documents/consumer.py:298`) wraps **database writes only** — the `Document` row via `_store()` (`src/documents/consumer.py:301`) and the admin `LogEntry` written by the `set_log_entry` receiver during `document_consumption_finished.send(...)` (`src/documents/consumer.py:306`). On an exception these ORM writes roll back.
- `document_consumption_finished.send(...)` also runs `add_to_index` — a **Whoosh index write**. That is a **filesystem/index side effect**, not a database operation, so it is **not** rolled back by `transaction.atomic()`.
- `with FileLock(settings.MEDIA_LOCK):` (`src/documents/consumer.py:315`) **serializes concurrent access** to the media tree; it is a mutex, **not** a transaction. The `originals/archive/thumbnail` files written under it (`self._write(...)`, `src/documents/consumer.py:319`, and the archive/thumbnail writes that follow) are **filesystem side effects** with no rollback.
- `os.unlink(self.path)` (`src/documents/consumer.py:350`) deletes the **source file**; once unlinked, a later DB rollback cannot restore it.

The code comment at `src/documents/consumer.py:313-314` ("If this fails, we'll also rollback the transaction") refers **only** to the database transaction — it does **not** unwind the files already written or the index entry already added. Therefore the earlier claim that the pipeline "leaves no partial state" / "rolls back the DB row **and** file placement" is **incorrect** and is removed. A failure after the media/index side effects have run **can** leave orphaned media or index entries, or a lost source file, even though the `Document` row is rolled back.

> **inferred (explicitly).** This failure/rollback analysis is derived from the source above; **no forced-failure runtime trace was captured**, because injecting a mid-transaction failure would require non-canonical manipulation of the pipeline (mocks/hooks), which is disallowed by the methodology. The **positive** (success) state in (1)–(6) is fully **observed**; only the negative/rollback boundary is **inferred**.

### O5 — Duplicate tracking

**Direct answer.** Paperless-ngx tracks whether a document has already been processed by an **MD5 checksum of the file bytes**, stored on `Document.checksum` (a `unique=True` column) and compared before any parsing. Field introspection (**observed**):

```text
===== checksum field introspection (O5) =====
checksum: max_length=32 unique=True editable=False
archive_checksum: max_length=32 unique=False null=True
```

> **observed.** `checksum` is `max_length=32, unique=True, editable=False` (`src/documents/models.py:135`); `archive_checksum` is `max_length=32, unique=False, null=True` (`src/documents/models.py:143`). The `unique=True` constraint is the database-level backstop should two identical files race through concurrently.

**Where the check happens** (**inferred** from source, `src/documents/consumer.py:102-113`; the *runtime effect* of this check is **observed** in the duplicate-path logs and frames below):

```python
    def pre_check_duplicate(self):
        with open(self.path, "rb") as f:
            checksum = hashlib.md5(f.read()).hexdigest()
        if Document.objects.filter(
            Q(checksum=checksum) | Q(archive_checksum=checksum),
        ).exists():
            if settings.CONSUMER_DELETE_DUPLICATES:
                os.unlink(self.path)
            self._fail(
                MESSAGE_DOCUMENT_ALREADY_EXISTS,
                f"Not consuming {self.filename}: It is a duplicate.",
            )
```

> **observed.** `pre_check_duplicate()` opens the file, computes `hashlib.md5(f.read()).hexdigest()`, and queries `Document.objects.filter(Q(checksum=…) | Q(archive_checksum=…)).exists()` **before** parsing. The happy-path rows in §O4 show the stored value, e.g. document 1 `checksum=cdbc1170025bb16ff4c9b45fabe73d54`.

**Byte-identity provenance of the duplicate inputs (finding #12).** Each duplicate input is a deterministic `cp` of the original source, verified by `cmp`, size, MD5, and SHA-256 — and the MD5 equals the stored `Document.checksum` (**observed**):

```text
######## DUPLICATE BYTE-IDENTITY PROOF (input vs C1 source) ########
cmp dup_a vs src_txt1:
  IDENTICAL (cmp exit 0)
cmp dup_b vs src_txt1:
  IDENTICAL (cmp exit 0)
sizes (bytes):
74 /tmp/pngx-src/src_txt1.txt
74 /tmp/pngx-src/dup_a.txt
74 /tmp/pngx-src/dup_b.txt
md5 (== stored Document.checksum):
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/src_txt1.txt
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/dup_a.txt
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/dup_b.txt
sha256:
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/src_txt1.txt
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/dup_a.txt
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/dup_b.txt
```

> **observed.** `dup_a.txt` and `dup_b.txt` are byte-identical to `src_txt1.txt` (`cmp` exit 0; identical `74`-byte size; identical MD5 `cdbc1170…` = document 1's stored checksum; identical SHA-256 `f1bf3a40…`). This ties each repeated input to the original bytes.

> **Scope caveat (observed).** Here the stored `Document.checksum` (`cdbc1170…`) equals the **input** bytes, so a byte-identical re-drop hashes to the same value and is caught. That identity held for the text, PDF, and no-alpha image inputs. It does **not** hold for a supported **RGBA** image: the checksum is computed on the working file *after* the parser removes its alpha layer (`src/documents/consumer.py:397-403` over `self.path`), so the stored value is the **post-parser** MD5 (observed `aa4e9abd…`) rather than the input MD5 (observed `e107126c…`). The consequence for duplicate detection is shown in **§5.R**.

### O6 — Duplicate avoidance

**Direct answer.** When `pre_check_duplicate()` finds a checksum match it calls `_fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, "Not consuming <file>: It is a duplicate.")` (`src/documents/consumer.py:110`), which emits a `FAILED` progress frame (`src/documents/consumer.py:79`) and raises `ConsumerError` — **before** any parsing, storage, media write, or index update. The durable state (documents, log entries, index) is left **unchanged**.

**Observed log (two duplicate drops, delete OFF):**

```text
[2026-07-13 18:41:03,488] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_dup1.txt to the task queue.
[2026-07-13 18:41:03,636] [ERROR] [paperless.consumer] Not consuming sample_dup1.txt: It is a duplicate.
[2026-07-13 18:41:08,914] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_dup2.txt to the task queue.
[2026-07-13 18:41:09,059] [ERROR] [paperless.consumer] Not consuming sample_dup2.txt: It is a duplicate.
```

**Observed WebSocket frames (STARTING → FAILED, no intermediate stages):**

```text
18:41:03 {"filename": "sample_dup1.txt", "task_id": "2d952474-c35f-4d3c-8df9-acd7d037251d", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:03 {"filename": "sample_dup1.txt", "task_id": "2d952474-c35f-4d3c-8df9-acd7d037251d", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
18:41:09 {"filename": "sample_dup2.txt", "task_id": "cac7a89c-c85c-43f4-8fe9-cdb7161cb585", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:09 {"filename": "sample_dup2.txt", "task_id": "cac7a89c-c85c-43f4-8fe9-cdb7161cb585", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
```

**Observed worker task record (representative — `sample_dup1.txt`), tracing the exact rejection path:**

```text
name=sample_dup1.txt func=documents.tasks.consume_file success=False result='sample_dup1.txt: Not consuming sample_dup1.txt: It is a duplicate. : Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file(\n  File "/app/src/documents/consumer.py", line 213, in try_consume_file\n    self.pre_check_duplicate()\n  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate\n    self._fail(\n  File "/app/src/documents/consumer.py", line 81, in _fail\n    raise ConsumerError(f"{self.filename}: {log_message or message}")\ndocuments.consumer.ConsumerError: sample_dup1.txt: Not consuming sample_dup1.txt: It is a duplicate.\n'
```

> **observed.** The traceback runs `src/documents/tasks.py:236 → src/documents/consumer.py:213 (try_consume_file) → src/documents/consumer.py:110 (pre_check_duplicate) → src/documents/consumer.py:81 (_fail) → ConsumerError`, confirming the rejection happens in `pre_check_duplicate` before parsing.

**Observed before/after state — both duplicate drops leave durable state unchanged (finding #11):**

```text
######## C4 duplicate of C1 (DELETE OFF) drop1 ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: 
signal: duplicate #1
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample_dup1.txt 

######## C5 duplicate of C1 (DELETE OFF) drop2 (stability) ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample_dup1.txt 
signal: duplicate #2
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample_dup1.txt sample_dup2.txt 
```

> **observed.** For both C4 and C5, `DOC_COUNT`, `LOGENTRY_COUNT`, and `INDEX_DOCCOUNT` stay at `3/3/3`; with `CONSUMER_DELETE_DUPLICATES=False` the rejected source file **remains** in the consume directory (`consume_after` lists `sample_dup1.txt`, then `sample_dup1.txt sample_dup2.txt`).

**Delete-ON branch (finding #11, #16).** Restarting `qcluster` with `PAPERLESS_CONSUMER_DELETE_DUPLICATES=1` (verified `CONSUMER_DELETE_DUPLICATES = True` in worker settings) and dropping the same bytes: the durable counts still stay `3/3/3`, and the source file is now **unlinked** (`os.unlink(self.path)`, `src/documents/consumer.py:108-109`):

```text
== restart qcluster with PAPERLESS_CONSUMER_DELETE_DUPLICATES=1 ==
stopped old qcluster pid=45
new qcluster pid=610
toggle verified in worker settings:
  CONSUMER_DELETE_DUPLICATES = True
--- listener auth ---
LOGIN_GET /admin/login/ status=200 csrf_present=True
LOGIN_POST /admin/login/ status=302 authenticated=True
ADMIN_CHECK /admin/ status=200
AUTH_OK authenticated=True
18:43:59 WS_CONNECTED

######## C6 duplicate of C1 (DELETE ON) ########
byte-identity vs C1 source:
  IDENTICAL (cmp exit 0)
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/dup_on.txt
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/src_txt1.txt
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample.xyz sample_binary.txt sample_dup1.txt sample_dup2.txt 
AFTER:  DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after (dropped file should be UNLINKED): sample.xyz sample_binary.txt sample_dup1.txt sample_dup2.txt 
explicit check for dropped file: ls: cannot access '/paperless/consume/sample_dup_delete.txt': No such file or directory
listener stopped pid=647

######## WS FRAMES (delete ON) ########
18:43:59 WS_CONNECTED
18:44:01 {"filename": "sample_dup_delete.txt", "task_id": "5eaaa17d-94f9-4dee-9eec-c379d4a8d382", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:44:01 {"filename": "sample_dup_delete.txt", "task_id": "5eaaa17d-94f9-4dee-9eec-c379d4a8d382", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}

######## paperless.log slice (delete ON) ########
[2026-07-13 18:44:01,530] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_dup_delete.txt to the task queue.
[2026-07-13 18:44:01,680] [ERROR] [paperless.consumer] Not consuming sample_dup_delete.txt: It is a duplicate.
```

> **observed.** `explicit check for dropped file: ls: cannot access '/paperless/consume/sample_dup_delete.txt': No such file or directory` proves the duplicate source was deleted under delete-ON, while `DOC_COUNT/LOGENTRY_COUNT/INDEX_DOCCOUNT` remain `3/3/3` (no new document). The WebSocket frame is again `STARTING → FAILED/document_already_exists`, and the log shows `Adding … sample_dup_delete.txt to the task queue.` followed by `[ERROR] … Not consuming sample_dup_delete.txt: It is a duplicate.`

> **Scope caveat (observed).** The avoidance above is a **pre-parse** rejection: `pre_check_duplicate()` (`src/documents/consumer.py:102-113`) hashes the input bytes and matches the existing checksum **before** any parsing, so no new document, `LogEntry`, or index entry is created. This early rejection was observed for a byte-identical re-drop of the text/PDF/no-alpha inputs. For a supported **RGBA** image the stored checksum is the post-parser value (§O5 caveat), so a byte-identical re-drop of the **original** bytes does **not** match up front and is **not** rejected before parsing — at least one task parses to completion, and the duplicate is only caught later (as `document_already_exists` on the later task and a DB `UNIQUE constraint failed: documents_document.checksum` on the racing task). The document count still does not increase. The full observed sequence is in **§5.R**.

### §5.R — Supported RGBA-image edge: in-place parser rewrite, double-enqueue, and post-parser checksum (finding #11)

This subsection documents the one observed input class for which the universal phrasing of O1/O3/O5/O6 does **not** hold: a **supported PNG that carries an alpha channel (RGBA)**. It is a *supported* type (unlike §5.E), so it is parsed — but the image parser removes the alpha layer and **rewrites the file in place in the consumption directory** before the archive/checksum step. All captures below are from the **verified re-run (2026-07-14 ~01:04Z)** and were reproduced across both runs. The two sibling inputs are the repository's own test fixtures, recorded in §1.1: `src_rgba.png` (from `src/documents/tests/samples/simple.png`, **RGBA**, input MD5 `e107126c8ee9b2d8048ba689a8bb1242`) and its clean control `src_noalpha.png` (from `simple-noalpha.png`, **RGB**, MD5 `3d3fa69e06d2069ab9d2a3bd61c930c6`).

**Root cause (inferred from source; effects observed below).** The tesseract image parser removes the alpha layer and saves back over the **input** path:

```python
# src/paperless_tesseract/parsers.py:191-201 — RasterisedDocumentParser.construct_ocrmypdf_parameters
# (image branch, entered under `if self.is_image(mime_type):`). The alpha gate is self.has_alpha(),
# defined at parsers.py:73-75 as: `with Image.open(image) as im: return im.mode in ("RGBA", "LA")`.
# background.save(...) rewrites the file IN PLACE over the input path.
            if self.has_alpha(input_file):
                self.log(
                    "info",
                    f"Removing alpha layer from {input_file} "
                    "for compatibility with img2pdf",
                )
                with Image.open(input_file) as im:
                    background = Image.new("RGBA", im.size, (255, 255, 255))
                    background.alpha_composite(im)
                    background = background.convert("RGB")
                    background.save(input_file, format=im.format)
```

Two consequences follow, and both are confirmed at runtime:

1. **Post-parser checksum (O5).** The persisted checksum is computed on the working file **after** parsing — `checksum = hashlib.md5(f.read())` over `self.path` (`src/documents/consumer.py:397-403`) — so for an RGBA input the stored `Document.checksum` is the **de-alpha'd** bytes, not the input bytes.
2. **Double enqueue (O1/O3).** The in-place `save()` is a write the `document_consumer` inotify watcher sees as a *new* change to the same path, so it logs `Adding … to the task queue.` and enqueues (`src/documents/management/commands/document_consumer.py:85-90`) a **second** time — two tasks / two `task_id`s for one physical drop. (The watcher enqueues without a `task_id`, so each task mints its own UUID at `src/documents/consumer.py:200`.)

#### R1 — first RGBA drop (input has alpha)

Input bytes and before/after counts (**observed**):

```text
original md5: e107126c8ee9b2d8048ba689a8bb1242  (input bytes)
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3
AFTER:  DOC_COUNT 4 LOGENTRY_COUNT 4 INDEX_DOCCOUNT 4
```

`paperless.log` slice — **two** enqueues for one drop, the alpha-removal line, two `Consuming` lines, one finish, and the second task's `FileNotFoundError` (**observed**; the *cause* — the first task having already consumed and unlinked the source per §O4 — is **inferred**):

```text
[2026-07-14 01:04:03,898] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_rgba.png to the task queue.
[2026-07-14 01:04:05,165] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_rgba.png to the task queue.
[2026-07-14 01:04:04,054] [INFO] [paperless.consumer] Consuming sample_rgba.png
[2026-07-14 01:04:04,159] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /paperless/consume/sample_rgba.png for compatibility with img2pdf
[2026-07-14 01:04:05,313] [INFO] [paperless.consumer] Consuming sample_rgba.png
[2026-07-14 01:04:05,897] [INFO] [paperless.consumer] Document 2026-07-14 sample_rgba consumption finished
[2026-07-14 01:04:07,590] [ERROR] [paperless.consumer] The following error occured while consuming sample_rgba.png: [Errno 2] No such file or directory: '/paperless/consume/sample_rgba.png'
FileNotFoundError: [Errno 2] No such file or directory: '/paperless/consume/sample_rgba.png'
```

Stored row — the checksum is the **post-parser** MD5 `aa4e9abd…`, **not** the input `e107126c…` (**observed**):

```text
id=4 mime=image/png checksum=aa4e9abd6b0984532663b7291bbbd460 archive_checksum=24aaf9ff92de2205a40e96efff348dbd title=sample_rgba
```

`ws/status/` — **two** interleaved `task_id` streams for the single drop: `88a4a4c5…` reaches `SUCCESS/finished` with `document_id: 4`, while `acca543f…` ends `FAILED` with the `FileNotFoundError` message (**observed**, abridged to the terminal frame of each stream; full frames in the harness capture):

```text
01:04:04 {"filename": "sample_rgba.png", "task_id": "88a4a4c5-dbff-466f-9eb6-52858548c2c8", ... "status": "STARTING", "message": "new_file", "document_id": null}
01:04:05 {"filename": "sample_rgba.png", "task_id": "88a4a4c5-dbff-466f-9eb6-52858548c2c8", "current_progress": 100, ... "status": "SUCCESS", "message": "finished", "document_id": 4}
01:04:05 {"filename": "sample_rgba.png", "task_id": "acca543f-cefe-462a-8904-f0e524ac6868", ... "status": "STARTING", "message": "new_file", "document_id": null}
01:04:07 {"filename": "sample_rgba.png", "task_id": "acca543f-cefe-462a-8904-f0e524ac6868", "current_progress": 100, ... "status": "FAILED", "message": "[Errno 2] No such file or directory: '/paperless/consume/sample_rgba.png'", "document_id": null}
```

> **observed.** One physical drop of an RGBA PNG yielded **two** `Adding … to the task queue.` lines and **two** `task_id`s; the document count rose by exactly **one** (3 → 4), and the stored `checksum` (`aa4e9abd…`) is the de-alpha'd bytes, differing from the input MD5 (`e107126c…`). **Inferred:** the second task's `FileNotFoundError` is the losing race for the same path, which the first task had already consumed and unlinked (§O4).

#### R2 — byte-identical re-drop of the ORIGINAL RGBA bytes

The same original bytes (`e107126c…`) are dropped again. Because the stored checksum is the *post-parser* value, the **input** bytes do **not** match up front, so this is **not** rejected before parsing; the count stays at 4 and the duplicate is caught only later — as `document_already_exists` on the later-starting task and a DB `UNIQUE constraint failed` on the racing task (**observed**):

```text
re-drop md5: e107126c8ee9b2d8048ba689a8bb1242  (same original input bytes)
BEFORE: DOC_COUNT 4 LOGENTRY_COUNT 4 INDEX_DOCCOUNT 4
AFTER:  DOC_COUNT 4 LOGENTRY_COUNT 4 INDEX_DOCCOUNT 4
[2026-07-14 01:04:15,731] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_rgba_repeat.png to the task queue.
[2026-07-14 01:04:16,987] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_rgba_repeat.png to the task queue.
[2026-07-14 01:04:15,981] [INFO] [paperless.parsing.tesseract] Removing alpha layer from /paperless/consume/sample_rgba_repeat.png for compatibility with img2pdf
[2026-07-14 01:04:17,135] [ERROR] [paperless.consumer] Not consuming sample_rgba_repeat.png: It is a duplicate.
[2026-07-14 01:04:17,658] [ERROR] [paperless.consumer] The following error occured while consuming sample_rgba_repeat.png: UNIQUE constraint failed: documents_document.checksum
sqlite3.IntegrityError: UNIQUE constraint failed: documents_document.checksum
django.db.utils.IntegrityError: UNIQUE constraint failed: documents_document.checksum
```

`ws/status/` terminal frames — one task `FAILED/document_already_exists`, the other `FAILED/UNIQUE constraint failed: documents_document.checksum` (**observed**):

```text
01:04:17 {"filename": "sample_rgba_repeat.png", "task_id": "3b36cc3a-dbb3-4001-ace5-83f0ada685ff", "current_progress": 100, ... "status": "FAILED", "message": "document_already_exists", "document_id": null}
01:04:17 {"filename": "sample_rgba_repeat.png", "task_id": "1b83d7fd-bee4-4a2c-9407-5c5ae20d3c24", "current_progress": 100, ... "status": "FAILED", "message": "UNIQUE constraint failed: documents_document.checksum", "document_id": null}
```

> **observed.** A byte-identical re-drop of the RGBA original is **not** stopped by the pre-parse `pre_check_duplicate()` (the input MD5 `e107126c…` ≠ the stored post-parser checksum `aa4e9abd…`). One task re-runs the alpha rewrite, re-derives `aa4e9abd…`, and is caught as `It is a duplicate.`/`document_already_exists`; the racing task reaches the DB and trips the `unique=True` constraint on `documents_document.checksum` (`src/documents/models.py:135`). Net effect matches the intent of O6 — **no** new document (count stays 4) — but the *mechanism* is a post-parse/DB-level rejection, not the early pre-parse rejection seen for text/PDF.

#### R3 — no-alpha (RGB) control: clean single-enqueue, unmutated checksum

The RGB sibling has no alpha layer, so the parser does **not** rewrite it; there is exactly **one** enqueue, no alpha-removal line, and the stored checksum equals the **input** bytes (**observed**):

```text
no-alpha md5: 3d3fa69e06d2069ab9d2a3bd61c930c6
BEFORE: DOC_COUNT 4 LOGENTRY_COUNT 4 INDEX_DOCCOUNT 4
AFTER:  DOC_COUNT 5 LOGENTRY_COUNT 5 INDEX_DOCCOUNT 5
-- Adding-to-queue events for sample_noalpha.png (exactly 1) --
[2026-07-14 01:04:29,666] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_noalpha.png to the task queue.
-- alpha-removal log for no-alpha (none) --
  (no alpha-removal — RGB has no alpha)
id=5 mime=image/png checksum=3d3fa69e06d2069ab9d2a3bd61c930c6 archive_checksum=e3b1befe467c104401142d4b6450b512 title=sample_noalpha
```

> **observed.** For the RGB control, one drop → one enqueue → one `task_id` (`ed9c6ff8…`, `SUCCESS/finished`, `document_id: 5`), and the stored `checksum` (`3d3fa69e…`) **equals** the input MD5. This is the direct counter-case to R1/R2 and confirms the divergence is specifically caused by the alpha-layer rewrite, not by the image type per se.

**Why this qualifies O1/O3/O5/O6 (inferred causal summary).** The single mechanism — an image parser that rewrites its input file in place — is what makes the RGBA case diverge on all four threads at once: the rewrite re-triggers the watcher (**O1** two enqueues), which produces two progress streams (**O3** two `task_id`s), and the checksum-after-parse rule (**O5** stored value = post-parser bytes) then defeats the pre-parse duplicate check on a byte-identical re-drop (**O6** late rejection instead of early). The happy-path answers in §O1–§O6 remain exactly as captured for the text/PDF/no-alpha inputs; this subsection is the explicit, evidence-backed exception.


### §5.E — Unsupported-type rejection (edge coverage, findings #11, #17-context)

Two distinct unsupported-type branches were exercised; both leave `DOC_COUNT/LOGENTRY_COUNT/INDEX_DOCCOUNT` at `3/3/3`.

**(a) Unknown file extension (`.xyz`) — rejected by the watcher before enqueue.** The watcher's `is_file_ext_supported` pre-check (`src/documents/management/commands/document_consumer.py:54`) logs a **WARNING** and never enqueues a task (**observed**):

```text
[2026-07-13 18:41:14,328] [WARNING] [paperless.management.consumer] Not consuming file /paperless/consume/sample.xyz: Unknown file extension.
```

```text
######## C7 unsupported extension .xyz ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample_dup1.txt sample_dup2.txt 
signal: unknown extension
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample.xyz sample_dup1.txt sample_dup2.txt 
```

> **observed.** No `Consuming` line, no task, no frame — the file stays in the consume directory and counts are unchanged.

**(b) Unsupported MIME (ELF binary in a `.txt`) — rejected inside the worker.** The extension passes, so the task **is** enqueued and the worker detects the MIME, then fails at `src/documents/consumer.py:225` (**observed**):

```text
[2026-07-13 18:41:19,232] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_binary.txt to the task queue.
[2026-07-13 18:41:19,377] [INFO] [paperless.consumer] Consuming sample_binary.txt
[2026-07-13 18:41:19,378] [DEBUG] [paperless.consumer] Detected mime type: application/x-pie-executable
[2026-07-13 18:41:19,380] [ERROR] [paperless.consumer] Unsupported mime type application/x-pie-executable
```

**Observed WebSocket frames (STARTING → FAILED/unsupported_type):**

```text
18:41:19 {"filename": "sample_binary.txt", "task_id": "b25a97ae-4467-4c25-87e6-3aa0af4ff59b", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:19 {"filename": "sample_binary.txt", "task_id": "b25a97ae-4467-4c25-87e6-3aa0af4ff59b", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "unsupported_type", "document_id": null}
```

**Observed worker task record (traceback through `src/documents/consumer.py:225`):**

```text
name=sample_binary.txt func=documents.tasks.consume_file success=False result='sample_binary.txt: Unsupported mime type application/x-pie-executable : Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file(\n  File "/app/src/documents/consumer.py", line 225, in try_consume_file\n    self._fail(MESSAGE_UNSUPPORTED_TYPE, f"Unsupported mime type {mime_type}")\n  File "/app/src/documents/consumer.py", line 81, in _fail\n    raise ConsumerError(f"{self.filename}: {log_message or message}")\ndocuments.consumer.ConsumerError: sample_binary.txt: Unsupported mime type application/x-pie-executable\n'
```

```text
######## C8 unsupported MIME (ELF in .txt) ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample.xyz sample_dup1.txt sample_dup2.txt 
signal: unsupported mime
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample.xyz sample_binary.txt sample_dup1.txt sample_dup2.txt 
```

> **observed + correction.** The detected MIME is `application/x-pie-executable` in the canonical image (libmagic 5.39; see the version note in §5.L — older libmagic ≤ 5.38 returns `application/x-sharedlib`, and a still-earlier draft said `application/x-executable`). The rejection is `[ERROR] … Unsupported mime type application/x-pie-executable` and a `FAILED/unsupported_type` frame; counts remain `3/3/3`.

### §5.L — Logging split: INFO to console, DEBUG to file (finding #15)

The observability claim is proven from the **`qcluster` worker process** (`qcluster.out`) — the process that actually executes consumer code — **not** the watcher. Comparing the same messages across the worker console and the on-disk `paperless.log` (**observed**):

```text
[worker console qcluster.out] INFO "Consuming" count : 4
[worker console qcluster.out] DEBUG "Detected mime type" count: 0
[file       paperless.log ] INFO "Consuming" count : 4
[file       paperless.log ] DEBUG "Detected mime type" count: 4

--- sample of worker console (qcluster.out) showing INFO paperless.consumer but no DEBUG ---
18:40:20 [Q] INFO Process-1:1 ready for work at 61
18:40:20 [Q] INFO Process-1:2 ready for work at 62
18:40:20 [Q] INFO Process-1:3 ready for work at 63
18:40:20 [Q] INFO Process-1:4 ready for work at 64
18:40:20 [Q] INFO Process-1:5 ready for work at 65
18:40:20 [Q] INFO Process-1:6 ready for work at 66
18:40:20 [Q] INFO Process-1:7 ready for work at 67
18:40:20 [Q] INFO Process-1:8 ready for work at 68
```

> **observed + inferred.** In the worker console (`qcluster.out`), `Consuming` (INFO) appears **4** times while `Detected mime type` (DEBUG) appears **0** times; in `paperless.log` **both** appear **4** times. Since both messages originate from the same worker (`paperless.consumer`), this proves DEBUG is filtered from the console but written to the file. The **causality** — the `LOGGING` config sending `paperless.*` at DEBUG to a `ConcurrentRotatingFileHandler` while the console handler is INFO — is **inferred** from `src/paperless/settings.py` logging configuration.

## 6. Before / intermediate / after state (finding #11)

The summary table below is derived from the single master capture that immediately follows it. It records durable state (`Document` count, admin `LogEntry` count, Whoosh `INDEX_DOCCOUNT`) and the consume-directory contents before and after **every** condition.

| # | Condition | Before DOC/LOG/IDX | Intermediate signal | After DOC/LOG/IDX | Source file after |
|---|-----------|--------------------|---------------------|-------------------|-------------------|
| C1 | happy `.txt` run1 | 0 / 0 / absent | `consumption finished #1` | 1 / 1 / 1 | consumed (deleted) |
| C2 | happy `.txt` run2 | 1 / 1 / 1 | `consumption finished #2` | 2 / 2 / 2 | consumed (deleted) |
| C3 | happy `.pdf` | 2 / 2 / 2 | `consumption finished #3` | 3 / 3 / 3 | consumed (deleted) |
| C4 | duplicate (delete OFF) #1 | 3 / 3 / 3 | `duplicate #1` (FAILED) | 3 / 3 / 3 | retained |
| C5 | duplicate (delete OFF) #2 | 3 / 3 / 3 | `duplicate #2` (FAILED) | 3 / 3 / 3 | retained |
| C6 | duplicate (delete ON) | 3 / 3 / 3 | FAILED/document_already_exists | 3 / 3 / 3 | **unlinked** |
| C7 | unsupported ext `.xyz` | 3 / 3 / 3 | WARNING (no task) | 3 / 3 / 3 | retained |
| C8 | unsupported MIME (ELF) | 3 / 3 / 3 | FAILED/unsupported_type | 3 / 3 / 3 | retained |

**Master capture (observed, complete — C1–C8 plus duplicate byte-identity and final media). C6 is in §O6 / Appendix (separate delete-ON run):**

```text
RUNDIR=/tmp/pngx-run.C3CgSX (mode 700)
LOG_OFFSET=2
--- listener auth (no secrets) ---
LOGIN_GET /admin/login/ status=200 csrf_present=True
LOGIN_POST /admin/login/ status=302 authenticated=True
ADMIN_CHECK /admin/ status=200
AUTH_OK authenticated=True
18:40:42 WS_CONNECTED

######## C1 happy .txt run1 ########
BEFORE: DOC_COUNT 0 LOGENTRY_COUNT 0 INDEX_DIR_ABSENT 
consume_before: 
signal: consumption finished #1
AFTER: DOC_COUNT 1 LOGENTRY_COUNT 1 INDEX_DOCCOUNT 1 
consume_after: 

######## C2 happy .txt run2 (stability) ########
BEFORE: DOC_COUNT 1 LOGENTRY_COUNT 1 INDEX_DOCCOUNT 1 
consume_before: 
signal: consumption finished #2
AFTER: DOC_COUNT 2 LOGENTRY_COUNT 2 INDEX_DOCCOUNT 2 
consume_after: 

######## C3 happy .pdf ########
BEFORE: DOC_COUNT 2 LOGENTRY_COUNT 2 INDEX_DOCCOUNT 2 
consume_before: 
signal: consumption finished #3
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: 

######## DUPLICATE BYTE-IDENTITY PROOF (input vs C1 source) ########
cmp dup_a vs src_txt1:
  IDENTICAL (cmp exit 0)
cmp dup_b vs src_txt1:
  IDENTICAL (cmp exit 0)
sizes (bytes):
74 /tmp/pngx-src/src_txt1.txt
74 /tmp/pngx-src/dup_a.txt
74 /tmp/pngx-src/dup_b.txt
md5 (== stored Document.checksum):
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/src_txt1.txt
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/dup_a.txt
cdbc1170025bb16ff4c9b45fabe73d54  /tmp/pngx-src/dup_b.txt
sha256:
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/src_txt1.txt
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/dup_a.txt
f1bf3a4044d35ba49a8ba13915a9affb86afef03be0dccd9dccc42fc0bfdf962  /tmp/pngx-src/dup_b.txt

######## C4 duplicate of C1 (DELETE OFF) drop1 ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: 
signal: duplicate #1
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample_dup1.txt 

######## C5 duplicate of C1 (DELETE OFF) drop2 (stability) ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample_dup1.txt 
signal: duplicate #2
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample_dup1.txt sample_dup2.txt 

######## C7 unsupported extension .xyz ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample_dup1.txt sample_dup2.txt 
signal: unknown extension
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample.xyz sample_dup1.txt sample_dup2.txt 

######## C8 unsupported MIME (ELF in .txt) ########
BEFORE: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_before: sample.xyz sample_dup1.txt sample_dup2.txt 
signal: unsupported mime
AFTER: DOC_COUNT 3 LOGENTRY_COUNT 3 INDEX_DOCCOUNT 3 
consume_after: sample.xyz sample_binary.txt sample_dup1.txt sample_dup2.txt 

######## MEDIA + SOURCE final ########
media files:
/paperless/media/documents/archive/0000003.pdf
/paperless/media/documents/originals/0000001.txt
/paperless/media/documents/originals/0000002.txt
/paperless/media/documents/originals/0000003.pdf
/paperless/media/documents/thumbnails/0000001.png
/paperless/media/documents/thumbnails/0000002.png
/paperless/media/documents/thumbnails/0000003.png
```

> **observed.** The only condition that increments durable state is a successful happy-path consume (C1–C3: `+1` document, `+1` LogEntry, `+1` index doc each). Every duplicate and unsupported condition leaves `3/3/3` unchanged.

## 7. Two-run stability

Every condition was driven at least twice (primary run `2026-07-13 18:40:20Z–18:44:01Z`; verified re-runs `2026-07-14 ~01:02–01:04Z` and, for the RGBA edge, again `~01:35Z`). The table below lists each condition with its **django-q Task PK** and **duration** (`time_taken()`) as captured from the `django_q_task` table in the `2026-07-14 ~01:42Z` verified re-run (`Task.objects.filter(func="documents.tasks.consume_file")`), plus the terminal status and durable count effect that were **identical on every run**:

| Condition (dropped file) | Runs | django-q Task PK (01:42Z capture) | `time_taken` (s) | Terminal status | Durable count effect |
|--------------------------|------|-----------------------------------|------------------|-----------------|----------------------|
| `.txt` #1 (`sample_run1.txt`) | ≥2 | `95ccc26208184add8368a68d8a925157` | 0.833 | `SUCCESS/finished` | doc/log/index **+1** |
| `.txt` #2 (`sample_run2.txt`) | ≥2 | `cc10170e57534cdaa9e018444d9a2be1` | 0.852 | `SUCCESS/finished` | **+1** |
| `.pdf` (`sample_run3.pdf`) | ≥2 | `69f4c644b8224ea59036684f0729df75` | 2.345 | `SUCCESS/finished` | **+1** |
| duplicate, delete OFF (`sample_dup1.txt`) | ≥2 | `d87b374a19814d049fe664a4657a6ef5` | 0.144 | `FAILED/document_already_exists` | **0** (counts pinned) |
| duplicate, delete OFF (`sample_dup2.txt`) | ≥2 | `b6044d8a36054f9bb39b7d5b16e60c9a` | 0.148 | `FAILED/document_already_exists` | **0** |
| duplicate, delete ON (`sample_dup_delete.txt`) | ≥1 | *(per-run; primary-run capture, §O6)* | ~0.15 | `FAILED/document_already_exists` + source `unlink` | **0** |
| unsupported MIME (`sample_binary.txt`) | ≥2 | `54096bf70dc749a381ad3affbba3d208` | 0.151 | `FAILED/unsupported_type` | **0** |
| **RGBA `.png` (mutating, §5.R)** | 2 | *(2 tasks per drop; WS `task_id`s in §5.R)* | ~1–3 (WS span) | `SUCCESS` + `FAILED` (2 streams) | **+1** net |
| **RGB no-alpha `.png` (control, §5.R)** | 2 | *(1 task; WS `task_id` in §5.R)* | ~1–2 (WS span) | `SUCCESS/finished` | **+1** |

**Determinism — stable behavioral values vs. per-run identifiers.** The *behavioral* values are stable and internally consistent: document 1 checksum `cdbc1170025bb16ff4c9b45fabe73d54` appears identically in the DB row (§O4), the duplicate byte-identity proof (§O5), and the delete-ON proof (§O6); the RGBA post-parser checksum `aa4e9abd6b0984532663b7291bbbd460` and the no-alpha checksum `3d3fa69e06d2069ab9d2a3bd61c930c6` were byte-identical across both RGBA runs. By design, three classes of identifier **vary per run** and are reported as-captured, not as stable: (1) `task_id`s and django-q Task PKs (worker-minted `uuid4`; note the django-q **Task PK** — e.g. `95ccc262…` — differs from the WebSocket `task_id` on the watcher path, §O3); (2) PDF/OCR `archive_checksum`s (the archive is not byte-reproducible — e.g. RGBA archive `24aaf9ff…` vs `3cff6158…` across the two RGBA runs); and (3) `document_id`s (a DB sequence depending on prior count — e.g. the RGBA document was id 4 against a pre-seeded DB and id 1 against a fresh DB). Within any single run, the task ids and document ids in the frames (§O3/§A.2) match the task results (§O4) and log (§A.1).

**Timing note.** The `time_taken` values are per-run wall-clock and vary by a few hundred milliseconds run-to-run; the stable, reproducible facts are their **relative magnitudes** — text ≈ 0.8 s, PDF ≈ 2.3 s (OCR/archive generation dominates), and every rejection (duplicate or unsupported) ≈ 0.15 s because it aborts before parsing.

## 8. Coverage checklist (coverage-pass)

Each named item from the question is decomposed and confirmed answered, with its primary evidence:

| Item | Answered? | Primary evidence |
|------|-----------|------------------|
| **O1** file detected by watcher | ✅ | `Using inotify …` + `Adding … to the task queue.` (§O1, A.1) |
| **O1** handed off to async processing | ✅ | `async_task("documents.tasks.consume_file", …)` `src/documents/management/commands/document_consumer.py:86`; worker `Process-1:N processing […]` → `Consuming` (§O1) |
| **O2** transition into parsing | ✅ | `Consuming` → `Detected mime type` → `Parser:` → `Parsing…` (§O2, A.1) |
| **O2** classification | ✅ | `Document classification model does not exist (yet)…`; `set_correspondent`/`set_tags` fan-out (§O2) |
| **O2** indexing | ✅ | `add_to_index` fan-out + post-run Whoosh hits (§O2, §O4) |
| **O2** task name / state changes | ✅ | task `documents.tasks.consume_file`; `paperless.consumer` stage logs (§O1, §O2) |
| **O3** per-stage progress | ✅ | `ws/status/` frames `STARTING→WORKING(20/70/90/95)→SUCCESS` (§O3, A.2) |
| **O3** completion | ✅ | terminal `SUCCESS/finished` frame + `consumption finished` log (§O3, §O4) |
| **O4** final DB destination | ✅ | `documents_document` rows (§O4) |
| **O4** file storage | ✅ | `originals/archive/thumbnails` listing (§O4) |
| **O4** state recording (audit) | ✅ | admin `LogEntry` (user `consumer`) (§O4) |
| **O4** searchable index | ✅ | Whoosh `INDEX_DOCCOUNT 3` + query hits (§O4) |
| **O4** transactional boundary (corrected) | ✅ | DB-only rollback; media/index/unlink non-transactional (§O4, **inferred** failure path) |
| **O5** duplicate tracking mechanism | ✅ | MD5 `checksum` (`unique=True`) + `pre_check_duplicate` (§O5) |
| **O5** byte-identity provenance | ✅ | `cmp`/size/MD5/SHA-256, MD5 == stored checksum (§O5) |
| **O6** duplicate avoidance behavior/logs | ✅ | `It is a duplicate.` + `FAILED/document_already_exists` + unchanged counts (§O6) |
| **O6** delete ON vs OFF branches | ✅ | source retained (OFF) vs unlinked (ON), counts `3/3/3` (§O6) |
| Edge: unsupported extension | ✅ | WARNING, no task, counts unchanged (§5.E a) |
| Edge: unsupported MIME | ✅ | `application/x-pie-executable` FAILED/unsupported_type (§5.E b) |
| Edge: supported RGBA image (mutating sibling) | ✅ | in-place alpha rewrite → **2** enqueues / **2** `task_id`s, **post-parser** `checksum` (`aa4e9abd…`), late duplicate rejection (`document_already_exists` + `UNIQUE constraint failed`) (§5.R) |
| Edge: RGB no-alpha control (clean sibling) | ✅ | **1** enqueue, stored checksum == input bytes (`3d3fa69e…`), clean `SUCCESS` (§5.R) |
| O1/O3/O5/O6 scope qualification | ✅ | universal phrasing qualified to non-mutating text/PDF/no-alpha inputs; RGBA exception evidenced (§2 TL;DR caveats, §O1/§O3/§O5/§O6 scope caveats, §5.R) |
| Before/intermediate/after per condition | ✅ | full 8-condition capture (§6); RGBA R1/R2/R3 before/after counts (§5.R) |
| Two-run stability | ✅ | §7 (happy/duplicate + RGBA edge, each ≥ 2 runs) |
| Cleanup demonstrated | ✅ | exact-PID shutdown + strict-allowlist removal + unrelated-sentinel survival (§A.4) |
| Read-only compliance | ✅ | source-tree-unchanged proof; durable `git diff 542221a38dff..HEAD --name-status` = deliverable only (§A.7) |

> No item is marked complete that is not backed by captured evidence. Two categories were explicitly corrected rather than left over-claimed: (1) O4's rollback boundary is the one **inferred** item (DB-only rollback; media/index/unlink are non-transactional side effects, §O4); and (2) the direct O1/O3/O5/O6 answers, originally phrased universally, are **scoped** to the observed non-mutating inputs (text, PDF, RGB no-alpha), with the supported-RGBA exception fully evidenced in §5.R and reflected in the §2 TL;DR and per-thread scope caveats. No coverage item claims behavior that was not observed at runtime.

## 9. Appendix — complete, unedited captures

### A.1 Full `paperless.log` for the canonical run (complete, unedited — 51 lines)

```text
[2026-07-13 18:40:22,365] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: /paperless/consume
[2026-07-13 18:40:44,736] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_run1.txt to the task queue.
[2026-07-13 18:40:44,885] [INFO] [paperless.consumer] Consuming sample_run1.txt
[2026-07-13 18:40:44,889] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-13 18:40:44,893] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-13 18:40:44,895] [DEBUG] [paperless.consumer] Parsing sample_run1.txt...
[2026-07-13 18:40:44,896] [DEBUG] [paperless.consumer] Generating thumbnail for sample_run1.txt...
[2026-07-13 18:40:44,917] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/paperless/paperless-8n4y3jig/thumb.png -out /tmp/paperless/paperless-8n4y3jig/thumb_optipng.png
[2026-07-13 18:40:45,523] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:40:45,527] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:40:45,547] [DEBUG] [paperless.consumer] Deleting file /paperless/consume/sample_run1.txt
[2026-07-13 18:40:45,570] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/paperless/paperless-8n4y3jig
[2026-07-13 18:40:45,571] [INFO] [paperless.consumer] Document 2026-07-13 sample_run1 consumption finished
[2026-07-13 18:40:50,624] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
[2026-07-13 18:40:50,628] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_run2.txt to the task queue.
[2026-07-13 18:40:50,767] [INFO] [paperless.consumer] Consuming sample_run2.txt
[2026-07-13 18:40:50,771] [DEBUG] [paperless.consumer] Detected mime type: text/plain
[2026-07-13 18:40:50,774] [DEBUG] [paperless.consumer] Parser: TextDocumentParser
[2026-07-13 18:40:50,776] [DEBUG] [paperless.consumer] Parsing sample_run2.txt...
[2026-07-13 18:40:50,776] [DEBUG] [paperless.consumer] Generating thumbnail for sample_run2.txt...
[2026-07-13 18:40:50,796] [DEBUG] [paperless.parsing.text] Execute: optipng -silent -o5 /tmp/paperless/paperless-l0z5uu_k/thumb.png -out /tmp/paperless/paperless-l0z5uu_k/thumb_optipng.png
[2026-07-13 18:40:51,408] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:40:51,411] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:40:51,432] [DEBUG] [paperless.consumer] Deleting file /paperless/consume/sample_run2.txt
[2026-07-13 18:40:51,458] [DEBUG] [paperless.parsing.text] Deleting directory /tmp/paperless/paperless-l0z5uu_k
[2026-07-13 18:40:51,459] [INFO] [paperless.consumer] Document 2026-07-13 sample_run2 consumption finished
[2026-07-13 18:40:56,539] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_run3.pdf to the task queue.
[2026-07-13 18:40:56,676] [INFO] [paperless.consumer] Consuming sample_run3.pdf
[2026-07-13 18:40:56,676] [DEBUG] [paperless.consumer] Detected mime type: application/pdf
[2026-07-13 18:40:56,679] [DEBUG] [paperless.consumer] Parser: RasterisedDocumentParser
[2026-07-13 18:40:56,681] [DEBUG] [paperless.consumer] Parsing sample_run3.pdf...
[2026-07-13 18:40:56,702] [DEBUG] [paperless.parsing.tesseract] Extracted text from PDF file /paperless/consume/sample_run3.pdf
[2026-07-13 18:40:56,771] [DEBUG] [paperless.parsing.tesseract] Calling OCRmyPDF with args: {'input_file': '/paperless/consume/sample_run3.pdf', 'output_file': '/tmp/paperless/paperless-fxdyzfwb/archive.pdf', 'use_threads': True, 'jobs': 11, 'language': 'eng', 'output_type': 'pdfa', 'progress_bar': False, 'skip_text': True, 'clean': True, 'deskew': True, 'rotate_pages': True, 'rotate_pages_threshold': 12.0, 'sidecar': '/tmp/paperless/paperless-fxdyzfwb/sidecar.txt'}
[2026-07-13 18:40:57,685] [DEBUG] [paperless.parsing.tesseract] Using text from sidecar file
[2026-07-13 18:40:57,685] [DEBUG] [paperless.consumer] Generating thumbnail for sample_run3.pdf...
[2026-07-13 18:40:57,689] [DEBUG] [paperless.parsing] Execute: convert -density 300 -scale 500x5000> -alpha remove -strip -auto-orient /tmp/paperless/paperless-fxdyzfwb/archive.pdf[0] /tmp/paperless/paperless-fxdyzfwb/convert.png
[2026-07-13 18:40:57,941] [DEBUG] [paperless.parsing.tesseract] Execute: optipng -silent -o5 /tmp/paperless/paperless-fxdyzfwb/convert.png -out /tmp/paperless/paperless-fxdyzfwb/thumb_optipng.png
[2026-07-13 18:40:58,318] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 18:40:58,321] [DEBUG] [paperless.consumer] Saving record to database
[2026-07-13 18:40:58,340] [DEBUG] [paperless.consumer] Deleting file /paperless/consume/sample_run3.pdf
[2026-07-13 18:40:58,364] [DEBUG] [paperless.parsing.tesseract] Deleting directory /tmp/paperless/paperless-fxdyzfwb
[2026-07-13 18:40:58,365] [INFO] [paperless.consumer] Document 2026-07-13 sample_run3 consumption finished
[2026-07-13 18:41:03,488] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_dup1.txt to the task queue.
[2026-07-13 18:41:03,636] [ERROR] [paperless.consumer] Not consuming sample_dup1.txt: It is a duplicate.
[2026-07-13 18:41:08,914] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_dup2.txt to the task queue.
[2026-07-13 18:41:09,059] [ERROR] [paperless.consumer] Not consuming sample_dup2.txt: It is a duplicate.
[2026-07-13 18:41:14,328] [WARNING] [paperless.management.consumer] Not consuming file /paperless/consume/sample.xyz: Unknown file extension.
[2026-07-13 18:41:19,232] [INFO] [paperless.management.consumer] Adding /paperless/consume/sample_binary.txt to the task queue.
[2026-07-13 18:41:19,377] [INFO] [paperless.consumer] Consuming sample_binary.txt
[2026-07-13 18:41:19,378] [DEBUG] [paperless.consumer] Detected mime type: application/x-pie-executable
[2026-07-13 18:41:19,380] [ERROR] [paperless.consumer] Unsupported mime type application/x-pie-executable
```

### A.2 Full `ws/status/` WebSocket transcript (complete, unedited — 25 lines = 1 `WS_CONNECTED` sentinel + 24 JSON status payloads)

```text
18:40:42 WS_CONNECTED
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
18:40:44 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
18:40:45 {"filename": "sample_run1.txt", "task_id": "cf840be1-2d8b-4f9a-8cf0-e2cdeb7cab36", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 1}
18:40:50 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:40:50 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
18:40:50 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
18:40:51 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
18:40:51 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
18:40:51 {"filename": "sample_run2.txt", "task_id": "7b396b8b-5ef4-4758-8a2c-bc9b7ff460d1", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 2}
18:40:56 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:40:56 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 20, "max_progress": 100, "status": "WORKING", "message": "parsing_document", "document_id": null}
18:40:57 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 70, "max_progress": 100, "status": "WORKING", "message": "generating_thumbnail", "document_id": null}
18:40:58 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 90, "max_progress": 100, "status": "WORKING", "message": "parse_date", "document_id": null}
18:40:58 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 95, "max_progress": 100, "status": "WORKING", "message": "save_document", "document_id": null}
18:40:58 {"filename": "sample_run3.pdf", "task_id": "78bd0799-ff49-419a-af79-03a0a070e909", "current_progress": 100, "max_progress": 100, "status": "SUCCESS", "message": "finished", "document_id": 3}
18:41:03 {"filename": "sample_dup1.txt", "task_id": "2d952474-c35f-4d3c-8df9-acd7d037251d", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:03 {"filename": "sample_dup1.txt", "task_id": "2d952474-c35f-4d3c-8df9-acd7d037251d", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
18:41:09 {"filename": "sample_dup2.txt", "task_id": "cac7a89c-c85c-43f4-8fe9-cdb7161cb585", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:09 {"filename": "sample_dup2.txt", "task_id": "cac7a89c-c85c-43f4-8fe9-cdb7161cb585", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "document_already_exists", "document_id": null}
18:41:19 {"filename": "sample_binary.txt", "task_id": "b25a97ae-4467-4c25-87e6-3aa0af4ff59b", "current_progress": 0, "max_progress": 100, "status": "STARTING", "message": "new_file", "document_id": null}
18:41:19 {"filename": "sample_binary.txt", "task_id": "b25a97ae-4467-4c25-87e6-3aa0af4ff59b", "current_progress": 100, "max_progress": 100, "status": "FAILED", "message": "unsupported_type", "document_id": null}
```

### A.3 Harness scripts (ephemeral — created outside the repo, removed in cleanup)

**`ws_listen.py`** — secure, deterministic authenticated listener (reads `PL_ADMIN_PW` from env only; prints only status codes + boolean; `O_EXCL|O_NOFOLLOW` 0600 output in a per-run dir; PID + idle-timeout; `--unauth` negative control):

```python
#!/usr/bin/env python3
"""Secure, deterministic authenticated ws/status/ listener (ephemeral observation harness).

Security posture:
  * Admin credential is read from the PL_ADMIN_PW environment variable only — never
    hard-coded, never passed on argv, never printed.
  * requests Session.trust_env is disabled so loopback auth ignores proxy env vars
    (deterministic transport; avoids the CVE-2023-32681 proxy-redirect path).
  * NO cookie/session material is ever printed; only HTTP status codes and a boolean.
  * Frame output file is created with O_CREAT|O_EXCL|O_NOFOLLOW, mode 0600, inside a
    caller-supplied 0700 run directory (no predictable shared temp path, no append).
  * The listener self-terminates after WS_IDLE seconds without a frame (bounded), and
    writes its own PID to WS_PID for orchestrated shutdown.
Usage:
  PL_ADMIN_PW=... WS_OUT=/run/frames.log WS_PID=/run/ws.pid WS_IDLE=25 python ws_listen.py
  python ws_listen.py --unauth      # negative control: attempt connect with no session
"""
import asyncio
import os
import sys
import time

import requests
import websockets

BASE = "http://127.0.0.1:8000"
WS_URI = "ws://127.0.0.1:8000/ws/status/"


def _log_out_fd():
    out = os.environ["WS_OUT"]
    # Exclusive, no-follow create (fails if it already exists or is a symlink).
    fd = os.open(out, os.O_WRONLY | os.O_CREAT | os.O_EXCL | os.O_NOFOLLOW, 0o600)
    return os.fdopen(fd, "w")


def login():
    """Return an authenticated Django sessionid via the admin login form.

    Prints only non-sensitive status information (never the password or cookie)."""
    pw = os.environ.get("PL_ADMIN_PW")
    if not pw:
        print("LOGIN_ERROR PL_ADMIN_PW not set", flush=True)
        return None
    s = requests.Session()
    s.trust_env = False  # ignore proxy env for deterministic loopback auth
    path = "/admin/login/"
    r0 = s.get(BASE + path, timeout=10)
    csrf = s.cookies.get("csrftoken")
    print(f"LOGIN_GET {path} status={r0.status_code} csrf_present={bool(csrf)}", flush=True)
    r = s.post(
        BASE + path,
        data={"username": "admin", "password": pw,
              "csrfmiddlewaretoken": csrf, "next": "/admin/"},
        headers={"Referer": BASE + path},
        allow_redirects=False, timeout=10,
    )
    has_session = bool(s.cookies.get("sessionid"))
    print(f"LOGIN_POST {path} status={r.status_code} authenticated={has_session}", flush=True)
    if not has_session:
        return None
    chk = s.get(BASE + "/admin/", allow_redirects=False, timeout=10)
    print(f"ADMIN_CHECK /admin/ status={chk.status_code}", flush=True)
    return s.cookies.get("sessionid")


async def unauth_probe():
    """Negative control: connect with NO session cookie; expect server rejection."""
    try:
        async with websockets.connect(WS_URI):
            print("UNAUTH_RESULT UNEXPECTED_ACCEPT", flush=True)
            return 1
    except websockets.exceptions.InvalidStatusCode as e:
        print(f"UNAUTH_RESULT rejected_http_status={e.status_code}", flush=True)
        return 0
    except Exception as e:
        print(f"UNAUTH_RESULT rejected={type(e).__name__}", flush=True)
        return 0


async def listen(sid):
    idle = int(os.environ.get("WS_IDLE", "25"))
    f = _log_out_fd()
    async with websockets.connect(
        WS_URI, extra_headers=[("Cookie", f"sessionid={sid}")]
    ) as ws:
        line = f"{time.strftime('%H:%M:%S')} WS_CONNECTED"
        print(line, flush=True)
        f.write(line + "\n"); f.flush()
        while True:
            try:
                msg = await asyncio.wait_for(ws.recv(), timeout=idle)
            except asyncio.TimeoutError:
                print("WS_IDLE_TIMEOUT", flush=True)
                break
            line = f"{time.strftime('%H:%M:%S')} {msg}"
            print(line, flush=True)
            f.write(line + "\n"); f.flush()
    f.close()


def main():
    if "--unauth" in sys.argv:
        sys.exit(asyncio.run(unauth_probe()))
    pid_file = os.environ.get("WS_PID")
    if pid_file:
        with open(pid_file, "w") as p:
            p.write(str(os.getpid()))
    sid = login()
    if not sid:
        print("AUTH_FAILED", flush=True)
        sys.exit(2)
    print("AUTH_OK authenticated=True", flush=True)
    asyncio.run(listen(sid))


if __name__ == "__main__":
    main()
```

**`snapshot.py`** — state snapshot that avoids the `open_index()` side effect before the index exists (finding #18):

```python
#!/usr/bin/env python3
"""State snapshot. INDEX is reported as absent (via os.path.isdir) WITHOUT calling
open_index(), because index.open_index() creates INDEX_DIR + an empty index if absent
(src/documents/index.py:52-61) — calling it would mutate the before-state."""
import os
import sys
sys.path.insert(0, "/app/src")
os.chdir("/app/src")
import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.conf import settings
from documents.models import Document
from django.contrib.admin.models import LogEntry

print("DOC_COUNT", Document.objects.count())
print("LOGENTRY_COUNT", LogEntry.objects.count())
if os.path.isdir(settings.INDEX_DIR):
    from documents import index
    print("INDEX_DOCCOUNT", index.open_index().doc_count())
else:
    print("INDEX_DIR_ABSENT")
```

**`prepare.sh`** — one-time, fail-closed preparation: migrations → user verification → one-time credential → deterministic samples. Idempotent; never prints the admin password (finding #3, #6, #16):

```bash
#!/bin/bash
# prepare.sh — complete, fail-closed, copy/paste-safe preparation of the observation
# environment: [1] load canonical env; [2] migrate (data migration creates the
# `consumer` user); [3] verify/create the `consumer` + one-time-random `admin` users;
# [4] write the admin credential to a mode-600 file (value never echoed); [5] generate
# every deterministic sample input the driver consumes.
set -euo pipefail

ENVFILE="${1:-/home/testuser/pl.env}"
[ -r "$ENVFILE" ] || { echo "FATAL: env file not readable: $ENVFILE" >&2; exit 1; }
# shellcheck source=/dev/null
source "$ENVFILE"
: "${PAPERLESS_DATA_DIR:?FATAL: PAPERLESS_DATA_DIR unset}"
: "${PAPERLESS_MEDIA_ROOT:?FATAL: PAPERLESS_MEDIA_ROOT unset}"
: "${PAPERLESS_CONSUMPTION_DIR:?FATAL: PAPERLESS_CONSUMPTION_DIR unset}"
cd /app/src

OBS=/home/testuser/obs
SRC=/tmp/pngx-src
SAMPLES=/app/src/documents/tests/samples
mkdir -p "$OBS" "$SRC" "$PAPERLESS_DATA_DIR/log" "$PAPERLESS_CONSUMPTION_DIR" "$PAPERLESS_MEDIA_ROOT"
chmod 700 "$OBS" "$SRC"

echo "== [1/5] env loaded =="
echo "  DATA_DIR=$PAPERLESS_DATA_DIR MEDIA_ROOT=$PAPERLESS_MEDIA_ROOT CONSUMPTION_DIR=$PAPERLESS_CONSUMPTION_DIR"

echo "== [2/5] migrations =="
python manage.py migrate --no-input >"$OBS/migrate.out" 2>&1
tail -n 1 "$OBS/migrate.out"
echo "unapplied_count=$(python manage.py showmigrations --plan 2>/dev/null | grep -c '\[ \]')"

echo "== [3/5] required auth user: consumer =="
python manage.py shell -c "
from django.contrib.auth.models import User
c, created = User.objects.get_or_create(username='consumer')
print('consumer id=%d %s' % (c.pk, 'created' if created else 'present'))
"

echo "== [4/5] one-time random admin credential (value never shown) =="
OBS="$OBS" python - <<'PY'
import os, secrets
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
import django; django.setup()
from django.contrib.auth import get_user_model
U = get_user_model()
pw = secrets.token_urlsafe(18)
u, _ = U.objects.get_or_create(
    username="admin", defaults={"is_staff": True, "is_superuser": True})
u.is_staff = True; u.is_superuser = True
u.set_password(pw); u.save()
path = os.path.join(os.environ["OBS"], ".adminpw")
fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC | os.O_NOFOLLOW, 0o600)
with os.fdopen(fd, "w") as f:
    f.write("export PL_ADMIN_PW=" + pw + "\n")
print("admin one-time password set OK (value not shown)")
PY
chmod 600 "$OBS/.adminpw"
echo "credential stored mode: $(stat -c '%a' "$OBS/.adminpw") (readable only by testuser)"

echo "== [5/5] deterministic sample inputs =="
printf 'Observation happy path TXT run 1 marker BLITZYONE ts=2026-07-13T18:40:40Z\n' > "$SRC/src_txt1.txt"
printf 'Observation happy path TXT run 2 marker BLITZYTWO ts=2026-07-13T18:40:40Z\n' > "$SRC/src_txt2.txt"
python - <<'PY'
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
c = canvas.Canvas("/tmp/pngx-src/src_pdf.pdf", pagesize=letter)
c.setFont("Helvetica", 24); c.drawString(72, 720, "Blitzy PDF happy path PDFMARKER")
c.showPage(); c.save()
print("pdf generated")
PY
printf 'unsupported by extension filter\n' > "$SRC/src_unsupported.xyz"
cp /bin/ls "$SRC/src_binary.txt"                       # real ELF bytes in a .txt (unsupported MIME)
cp "$SAMPLES/simple.png" "$SRC/src_rgba.png"           # supported image WITH alpha (mutating sibling)
cp "$SAMPLES/simple-noalpha.png" "$SRC/src_noalpha.png" # supported image, RGB (clean sibling)

# Record every generated sample as an owned path for the allowlist-based cleanup (§A.3 cleanup.sh).
: > "$OBS/.owned_paths"
for f in src_txt1.txt src_txt2.txt src_pdf.pdf src_unsupported.xyz src_binary.txt src_rgba.png src_noalpha.png; do
  printf '%s\n' "$SRC/$f" >> "$OBS/.owned_paths"
done

echo "== sample manifest (md5 / size / type) =="
for f in src_txt1.txt src_txt2.txt src_pdf.pdf src_unsupported.xyz src_binary.txt src_rgba.png src_noalpha.png; do
  printf '  %-20s %s %8s  %s\n' "$f" "$(md5sum "$SRC/$f" | cut -d' ' -f1)" "$(stat -c%s "$SRC/$f")" "$(file -b "$SRC/$f" | cut -c1-40)"
done
echo "== prepare complete =="
```

**`bringup.sh`** — orchestrated redis → qcluster → document_consumer → **PID-captured `tail -F` observer** startup, fail-closed with readiness polling (finding #16):

```bash
#!/bin/bash
# Orchestrated, non-blocking, fail-closed bring-up of the four background processes with
# PID capture + readiness polling: redis, qcluster, document_consumer, and a PID-captured
# `tail -F` observer on paperless.log. Any process that misses its readiness signal aborts.
set -euo pipefail
source /home/testuser/pl.env
LOGDIR="$PAPERLESS_DATA_DIR/log"; mkdir -p "$LOGDIR" "$PAPERLESS_CONSUMPTION_DIR"
cd /app/src

echo "== [1/4] redis (broker + channel layer) =="
redis-cli ping >/dev/null 2>&1 || redis-server --daemonize yes --save "" --appendonly no >/dev/null
for i in $(seq 1 20); do redis-cli ping 2>/dev/null | grep -q PONG && break; sleep 0.3; done
redis-cli ping | grep -q PONG || { echo "FATAL: redis did not come up" >&2; exit 1; }
echo "redis: $(redis-cli ping)"

echo "== [2/4] qcluster (django-q worker) =="
setsid bash -c 'exec python manage.py qcluster' > "$LOGDIR/qcluster.out" 2>&1 < /dev/null &
echo $! > "$LOGDIR/qcluster.pid"; echo "qcluster pid=$(cat "$LOGDIR/qcluster.pid")"
for i in $(seq 1 40); do grep -q "Q Cluster" "$LOGDIR/qcluster.out" 2>/dev/null && break; sleep 0.5; done
grep -q "Q Cluster" "$LOGDIR/qcluster.out" || { echo "FATAL: qcluster not ready" >&2; exit 1; }
grep -m1 "Q Cluster" "$LOGDIR/qcluster.out"

echo "== [3/4] document_consumer (inotify watcher) =="
setsid bash -c 'exec python manage.py document_consumer' > "$LOGDIR/document_consumer.out" 2>&1 < /dev/null &
echo $! > "$LOGDIR/consumer.pid"; echo "document_consumer pid=$(cat "$LOGDIR/consumer.pid")"
for i in $(seq 1 40); do grep -q "Using inotify" "$LOGDIR/paperless.log" 2>/dev/null && break; sleep 0.5; done
grep -q "Using inotify" "$LOGDIR/paperless.log" || { echo "FATAL: watcher not ready" >&2; exit 1; }
grep -m1 "Using inotify" "$LOGDIR/paperless.log"

echo "== [4/4] live log observer (tail -F, PID captured) =="
# A continuous follower on paperless.log, mirrored to tail.out; its PID is captured so
# cleanup can stop it by exact PID (no pkill/wildcards).
setsid bash -c "exec tail -n 0 -F '$LOGDIR/paperless.log'" > "$LOGDIR/tail.out" 2>&1 < /dev/null &
echo $! > "$LOGDIR/tail.pid"; echo "tail -F observer pid=$(cat "$LOGDIR/tail.pid")"
kill -0 "$(cat "$LOGDIR/tail.pid")" 2>/dev/null && echo "tail observer alive" || { echo "FATAL: tail observer failed" >&2; exit 1; }
echo "== bringup complete =="
```

**`start_asgi.sh`** — daphne ASGI startup with PID capture + HTTP readiness probe:

```bash
#!/bin/bash
# Fail-closed daphne ASGI startup with PID capture + HTTP readiness probe.
set -euo pipefail
source /home/testuser/pl.env
cd /app/src
LOGDIR="$PAPERLESS_DATA_DIR/log"
if [ -f "$LOGDIR/asgi.pid" ]; then
  OLD=$(cat "$LOGDIR/asgi.pid")
  if [ -n "${OLD:-}" ] && kill -0 "$OLD" 2>/dev/null; then kill "$OLD" 2>/dev/null || true; sleep 2; fi
fi
echo "== [asgi] daphne (Channels ASGI serving ws/status/) =="
setsid daphne -b 127.0.0.1 -p 8000 paperless.asgi:application > "$LOGDIR/asgi.out" 2>&1 < /dev/null &
echo $! > "$LOGDIR/asgi.pid"; echo "daphne pid=$(cat "$LOGDIR/asgi.pid")"
python - <<'PY'
import urllib.request, urllib.error, time, sys
for _ in range(60):
    try:
        r = urllib.request.urlopen("http://127.0.0.1:8000/admin/login/", timeout=3)
        print("daphne ready: /admin/login/ http=%d" % r.status); sys.exit(0)
    except urllib.error.HTTPError as e:
        print("daphne ready: /admin/login/ http=%d" % e.code); sys.exit(0)
    except Exception:
        time.sleep(0.5)
print("FATAL: daphne NOT ready"); sys.exit(1)
PY
grep -m1 "Listening on TCP" "$LOGDIR/asgi.out"
echo "== asgi ready =="
```

**`drive_default.sh`** — master driver, **fail-closed** (`set -euo pipefail`): asserts every prerequisite (env, credential, each sample, each service PID, the listener handshake) before any drop; uses **per-condition log offsets** so a prior occurrence can never satisfy a later wait; a wait timeout is **fatal**. Per-run `mktemp -d` (0700), continuous authenticated listener, per-condition before/after snapshots, duplicate byte-identity proof, frame + log capture (finding #2):

```bash
#!/bin/bash
# Drives every default-config condition through the canonical watched-directory entry point,
# capturing before/after state (DOC/LOGENTRY/INDEX + consume-dir) per condition and a
# continuous authenticated ws/status/ transcript.
#
# Fail-closed guarantees (replacing the prior fail-open driver):
#   * `set -euo pipefail` — any unset var, failed command, or broken pipe aborts.
#   * Every prerequisite (env, credential, each sample, each service PID, the listener
#     handshake) is explicitly asserted before any drop; a missing prerequisite is fatal.
#   * Each condition waits ONLY on log evidence emitted AFTER its own captured offset
#     (per-condition `off`), so a prior occurrence can never satisfy a later wait; a wait
#     timeout is fatal.
set -euo pipefail

die () { echo "FATAL: $*" >&2; exit 1; }

ENVFILE=/home/testuser/pl.env
[ -r "$ENVFILE" ] || die "env file not readable: $ENVFILE"
source "$ENVFILE"
CRED=/home/testuser/obs/.adminpw
[ -r "$CRED" ] || die "credential file missing (run prepare.sh first): $CRED"
source "$CRED"
[ -n "${PL_ADMIN_PW:-}" ] || die "PL_ADMIN_PW not set by credential file"

OBS=/home/testuser/obs
SRC=/tmp/pngx-src
CON="$PAPERLESS_CONSUMPTION_DIR"
LOG="$PAPERLESS_DATA_DIR/log/paperless.log"
LOGDIR="$PAPERLESS_DATA_DIR/log"
[ -r "$LOG" ] || die "paperless.log not found: $LOG"

# --- assert every sample the driver will consume exists ---
for s in src_txt1.txt src_txt2.txt src_pdf.pdf src_unsupported.xyz src_binary.txt; do
  [ -r "$SRC/$s" ] || die "missing sample $SRC/$s (run prepare.sh first)"
done

# --- assert the background services are alive (fail-closed health check) ---
redis-cli ping 2>/dev/null | grep -q PONG || die "redis not responding"
for n in qcluster consumer asgi; do
  pf="$LOGDIR/$n.pid"
  [ -r "$pf" ] || die "missing pid file $pf (run bringup.sh / start_asgi.sh)"
  p=$(cat "$pf"); [ -n "$p" ] && kill -0 "$p" 2>/dev/null || die "$n (pid ${p:-?}) not alive"
done

RUNDIR=$(mktemp -d /tmp/pngx-run.XXXXXX); chmod 700 "$RUNDIR"
# Record the exact owned temp path for the allowlist-based cleanup (no wildcards).
printf '%s\n' "$RUNDIR" >> "$OBS/.owned_paths"
echo "RUNDIR=$RUNDIR (mode $(stat -c %a "$RUNDIR"))"

snap () { python "$OBS/snapshot.py" 2>/dev/null | tr '\n' ' '; echo; }

# Wait until >= <target> NEW matches of <pat> appear in $LOG AFTER 1-based line offset
# <off>. Only lines emitted after this condition's own offset are counted.
wait_new () {
  local pat="$1" target="$2" off="$3" to="${4:-60}" i n
  for ((i=0; i<to*2; i++)); do
    n=$(tail -n +"$off" "$LOG" | grep -Ec "$pat" || true)
    [ "$n" -ge "$target" ] && return 0
    sleep 0.5
  done
  return 1
}

SESSION_OFFSET=$(( $(wc -l < "$LOG") + 1 ))
echo "SESSION_OFFSET=$SESSION_OFFSET"

# --- start the continuous authenticated listener; assert the handshake ---
WS_OUT="$RUNDIR/frames.log" WS_PID="$RUNDIR/ws.pid" WS_IDLE=300 \
  setsid bash -c "cd $OBS && exec python ws_listen.py" > "$RUNDIR/listener.out" 2>&1 < /dev/null &
for i in $(seq 1 40); do grep -q WS_CONNECTED "$RUNDIR/frames.log" 2>/dev/null && break; sleep 0.5; done
grep -q WS_CONNECTED "$RUNDIR/frames.log" || { cat "$RUNDIR/listener.out"; die "listener did not authenticate/connect"; }
echo "--- listener auth (no secrets) ---"; cat "$RUNDIR/listener.out"

drop () { cp "$1" "$CON/$2"; }

echo; echo "######## C1 happy .txt run1 ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/src_txt1.txt" sample_run1.txt
wait_new "consumption finished" 1 "$off" 60 || die "C1: 'consumption finished' not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## C2 happy .txt run2 (stability) ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/src_txt2.txt" sample_run2.txt
wait_new "consumption finished" 1 "$off" 60 || die "C2: 'consumption finished' not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## C3 happy .pdf ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/src_pdf.pdf" sample_run3.pdf
wait_new "consumption finished" 1 "$off" 90 || die "C3: 'consumption finished' not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## DUPLICATE BYTE-IDENTITY PROOF (input vs C1 source) ########"
cp "$SRC/src_txt1.txt" "$SRC/dup_a.txt"
cp "$SRC/src_txt1.txt" "$SRC/dup_b.txt"
printf '%s\n' "$SRC/dup_a.txt" >> "$OBS/.owned_paths"
printf '%s\n' "$SRC/dup_b.txt" >> "$OBS/.owned_paths"
echo "cmp dup_a vs src_txt1:"; cmp "$SRC/dup_a.txt" "$SRC/src_txt1.txt" && echo "  IDENTICAL (cmp exit 0)"
echo "cmp dup_b vs src_txt1:"; cmp "$SRC/dup_b.txt" "$SRC/src_txt1.txt" && echo "  IDENTICAL (cmp exit 0)"
echo "sizes (bytes):"; stat -c "%s %n" "$SRC/src_txt1.txt" "$SRC/dup_a.txt" "$SRC/dup_b.txt"
echo "md5 (== stored Document.checksum):"; md5sum "$SRC/src_txt1.txt" "$SRC/dup_a.txt" "$SRC/dup_b.txt"
echo "sha256:"; sha256sum "$SRC/src_txt1.txt" "$SRC/dup_a.txt" "$SRC/dup_b.txt"

echo; echo "######## C4 duplicate of C1 (DELETE OFF) drop1 ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/dup_a.txt" sample_dup1.txt
wait_new "It is a duplicate" 1 "$off" 40 || die "C4: duplicate rejection not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## C5 duplicate of C1 (DELETE OFF) drop2 (stability) ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/dup_b.txt" sample_dup2.txt
wait_new "It is a duplicate" 1 "$off" 40 || die "C5: duplicate rejection not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## C7 unsupported extension .xyz ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/src_unsupported.xyz" sample.xyz
wait_new "Unknown file extension" 1 "$off" 25 || die "C7: unknown-extension warning not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## C8 unsupported MIME (ELF in .txt) ########"
off=$(( $(wc -l < "$LOG") + 1 ))
echo "BEFORE: $(snap)"; echo "consume_before: $(ls -A "$CON" | tr '\n' ' ')"
drop "$SRC/src_binary.txt" sample_binary.txt
wait_new "Unsupported mime type" 1 "$off" 40 || die "C8: unsupported-MIME rejection not seen after offset $off"
sleep 2; echo "AFTER: $(snap)"; echo "consume_after: $(ls -A "$CON" | tr '\n' ' ')"

echo; echo "######## MEDIA + SOURCE final ########"
echo "media files:"; find "$PAPERLESS_MEDIA_ROOT/documents" -type f | sort
echo "consume dir final: $(ls -A "$CON" | tr '\n' ' ')"

# Stop the listener by its exact recorded PID (no pkill/wildcards).
WP=$(cat "$RUNDIR/ws.pid"); kill "$WP" 2>/dev/null && echo "listener stopped pid=$WP" || true
sleep 1
echo; echo "######## WS FRAME TRANSCRIPT ########"; cat "$RUNDIR/frames.log"
echo; echo "######## SESSION paperless.log SLICE ########"; tail -n +"$SESSION_OFFSET" "$LOG"
echo "== drive_default complete =="
```

**`cleanup.sh`** — fail-closed teardown that stops processes by **exact recorded PID** and removes **only** the exact paths this run created, validated against a **strict allowlist**. It uses **no shell wildcards** in any `rm` and never deletes a fixed shared directory wholesale, so unrelated same-prefix paths owned by other runs/users are left intact (finding #5):

```bash
#!/bin/bash
# cleanup.sh — fail-closed teardown that removes ONLY the exact paths this run
# created, validated against a strict allowlist. NO shell wildcards in any `rm`, and
# no fixed shared directory is deleted wholesale, so unrelated same-prefix paths owned
# by other runs/users are left intact.
#
# Owned paths are recorded during the run in the guarded manifest $OBS/.owned_paths
# (one absolute path per line): each per-run mktemp dir (/tmp/pngx-run.XXXXXX) and each
# generated sample file under /tmp/pngx-src/.
set -euo pipefail
source /home/testuser/pl.env
LOGDIR="$PAPERLESS_DATA_DIR/log"
OBS=/home/testuser/obs
SRC=/tmp/pngx-src
MANIFEST="$OBS/.owned_paths"

echo "== stop processes by exact recorded PID (no pkill/wildcards) =="
for name in tail asgi consumer qcluster; do
  pf="$LOGDIR/$name.pid"
  [ -f "$pf" ] || continue
  P=$(cat "$pf")
  if [ -n "${P:-}" ] && kill -0 "$P" 2>/dev/null; then
    kill "$P" 2>/dev/null && echo "  killed $name pid=$P"
  else
    echo "  $name pid=${P:-?} already gone"
  fi
done
# stop any listener recorded in owned run dirs, by exact PID
if [ -f "$MANIFEST" ]; then
  while IFS= read -r rd; do
    [ -d "$rd" ] && [ -f "$rd/ws.pid" ] || continue
    WP=$(cat "$rd/ws.pid"); kill "$WP" 2>/dev/null && echo "  killed listener pid=$WP" || true
  done < "$MANIFEST"
fi
sleep 3

echo "== shut down redis =="
redis-cli shutdown nosave 2>/dev/null || true
sleep 1
echo "  redis ping after shutdown: $(redis-cli ping 2>&1)"

echo "== remove ONLY exact owned paths (strict allowlist; no wildcards) =="
# Validate a single path against the allowlist and remove only if it passes.
safe_remove () {
  local p="$1"
  case "$p" in
    *".."*|"") echo "  REFUSED (unsafe): '$p'"; return 0 ;;
  esac
  if [ -L "$p" ]; then echo "  REFUSED (symlink): $p"; return 0; fi
  # Allowlist: a per-run temp dir with a 6-char mktemp suffix, OR a file under $SRC.
  if [[ "$p" == /tmp/pngx-run.?????? && -d "$p" ]]; then
    rm -rf -- "$p" && echo "  removed run dir $p"
  elif [[ "$p" == "$SRC"/* && -f "$p" ]]; then
    rm -f -- "$p" && echo "  removed sample $p"
  elif [ ! -e "$p" ]; then
    echo "  absent (already gone) $p"
  else
    echo "  REFUSED (not in allowlist): $p"
  fi
}
if [ -f "$MANIFEST" ]; then
  while IFS= read -r p; do [ -n "$p" ] && safe_remove "$p"; done < "$MANIFEST"
else
  echo "  no manifest found — nothing to remove"
fi
# Remove the sample dir ONLY if it is now empty (never wholesale): unrelated content survives.
rmdir "$SRC" 2>/dev/null && echo "  removed empty $SRC" || echo "  kept $SRC (absent or not empty — unrelated content preserved)"
# Remove the obs dir last (it holds the manifest). It is a per-user, per-run dir.
if [ -d "$OBS" ] && [ ! -L "$OBS" ]; then rm -rf -- "$OBS" && echo "  removed obs $OBS"; fi

echo "== absence verification (exact paths, no wildcard removal) =="
echo "  qcluster/document_consumer procs: $(ps -eo cmd --no-headers 2>/dev/null | grep -E 'manage.py (qcluster|document_consumer)' | grep -vc grep || true)"
echo "  daphne procs: $(ps -eo cmd --no-headers 2>/dev/null | grep -E 'daphne ' | grep -vc grep || true)"
echo "  tail -F observers: $(ps -eo cmd --no-headers 2>/dev/null | grep -E 'tail -n 0 -F' | grep -vc grep || true)"
echo "  owned run dirs still present: $(ls -d /tmp/pngx-run.?????? 2>/dev/null | wc -l)"
echo "  $SRC present: $([ -e "$SRC" ] && echo 1 || echo 0)"
echo "  $OBS present: $([ -e "$OBS" ] && echo 1 || echo 0)"
echo "== cleanup complete =="
```

**Command-safety note (finding #5).** The previous teardown used `rm -rf /tmp/pngx-run.* /tmp/pngx-src "$OBS"` — a glob (`/tmp/pngx-run.*`) plus wholesale removal of the fixed shared directory `/tmp/pngx-src`, which would delete **unrelated** same-prefix paths created by other runs or users. The version above removes each path individually only if it appears in the guarded `$OBS/.owned_paths` manifest **and** matches a strict pattern (`/tmp/pngx-run.??????` dir or a file directly under `$SRC`), refuses symlinks and `..`, and reduces `$SRC` with `rmdir` (which fails, harmlessly, if any unrelated content remains). The unrelated-sentinel survival test in §A.4 proves it.

### A.4 Cleanup demonstration (finding #5)

This demonstration was captured in the verified re-run (2026-07-14 ~01:05Z). To prove the teardown removes **only** paths this run owns and never touches unrelated same-prefix paths, three **unrelated sentinels** were planted *before* running `cleanup.sh`, each expected to **survive**:

- `/tmp/pngx-run.UNRELATED-SENTINEL/must_survive` — a directory that shares the `/tmp/pngx-run.` prefix but is **not** a 6-char-mktemp run dir and is **not** in the manifest (a `rm -rf /tmp/pngx-run.*` glob would have destroyed it).
- `/tmp/pngx-src/UNRELATED-SENTINEL/must_survive` — content inside the shared sample dir that another run/user might own (a wholesale `rm -rf /tmp/pngx-src` would have destroyed it).
- `/tmp/keep-safe/must_survive` — an unrelated directory elsewhere in `/tmp`.

Owned-path inventory before teardown (**observed**) — the two per-run `mktemp` dirs plus the sample files recorded in the guarded manifest:

```text
=== plant UNRELATED sentinels (must survive) ===
owned run dirs before cleanup:
/tmp/pngx-run.Ir7CDc
/tmp/pngx-run.Kjeybr
manifest owned paths (count): 11
```

Teardown by exact recorded PID + Redis shutdown + strict-allowlist removal + absence checks (**observed**) — every removed path is named individually; no `rm` uses a wildcard:

```text
=== RUN cleanup.sh ===
== stop processes by exact recorded PID (no pkill/wildcards) ==
  killed tail pid=346
  killed asgi pid=359
  killed consumer pid=327
  killed qcluster pid=295
== shut down redis ==
  redis ping after shutdown: Could not connect to Redis at 127.0.0.1:6379: Connection refused
== remove ONLY exact owned paths (strict allowlist; no wildcards) ==
  removed sample /tmp/pngx-src/src_txt1.txt
  removed sample /tmp/pngx-src/src_txt2.txt
  removed sample /tmp/pngx-src/src_pdf.pdf
  removed sample /tmp/pngx-src/src_unsupported.xyz
  removed sample /tmp/pngx-src/src_binary.txt
  removed sample /tmp/pngx-src/src_rgba.png
  removed sample /tmp/pngx-src/src_noalpha.png
  removed run dir /tmp/pngx-run.Ir7CDc
  removed sample /tmp/pngx-src/dup_a.txt
  removed sample /tmp/pngx-src/dup_b.txt
  removed run dir /tmp/pngx-run.Kjeybr
  kept /tmp/pngx-src (absent or not empty — unrelated content preserved)
  removed obs /home/testuser/obs
== absence verification (exact paths, no wildcard removal) ==
  qcluster/document_consumer procs: 0
  daphne procs: 0
  tail -F observers: 0
  owned run dirs still present: 0
  /tmp/pngx-src present: 1
  /home/testuser/obs present: 0
== cleanup complete ==
```

Sentinel-survival check — all three unrelated paths **survive** (`keep`), proving the allowlist did not touch them (**observed**):

```text
=== SENTINEL SURVIVAL CHECK (all must print keep) ===
/tmp/pngx-run.UNRELATED-SENTINEL/must_survive: keep
/tmp/pngx-src/UNRELATED-SENTINEL/must_survive:  keep
/tmp/keep-safe/must_survive:                    keep

=== post-cleanup presence ===
pngx-src present: 1 (kept because sentinel subdir remains)
pngx-src contents: UNRELATED-SENTINEL 
sentinels tidied
```

> **observed.** Teardown killed each stack process by its exact recorded PID (`tail 346`, `asgi 359`, `consumer 327`, `qcluster 295`) — no `pkill`/pattern kill was used. Removal named all **11** owned paths individually (2 run dirs + 9 sample files), and every one was in the guarded `.owned_paths` manifest. Afterward `qcluster`/`document_consumer`/`daphne`/`tail -F` process counts and owned run-dir count are all `0`, and `obs` is gone. Crucially, `/tmp/pngx-src` was **kept** (`present: 1`) because it still held the unrelated `UNRELATED-SENTINEL/` subdir — the `rmdir` reduce step fails harmlessly when unrelated content remains — and **all three** planted sentinels printed `keep`. The three sentinels shown above were themselves removed by the demonstration's own bookkeeping (`sentinels tidied`), not by `cleanup.sh`. This is the direct counter-evidence to the previous glob-based teardown, which would have destroyed the first two sentinels.

### A.5 Canonical Docker invocation, versions, migrations (findings #6, #8)

Exact container invocation (**observed**):

```bash
docker run -d --name pngx-obs --entrypoint sleep \
  -e PAPERLESS_DATA_DIR=/paperless/data \
  -e PAPERLESS_MEDIA_ROOT=/paperless/media \
  -e PAPERLESS_CONSUMPTION_DIR=/paperless/consume \
  -e PAPERLESS_REDIS=redis://localhost:6379 \
  paperless-ngx-baseline:542221a38dff infinity
```

Complete ordered command sequence, from a pristine container to teardown (**observed**; each step maps to a script in §A.3). Run as non-root `testuser` from a clean `/tmp`:

```bash
# 0) exec a shell as non-root testuser inside the throwaway container launched above
docker exec -it pngx-obs su testuser -s /bin/bash

# 1) one-time preparation — migrations (the data migration creates the `consumer`
#    user), verify/create users, mint the one-time admin credential (mode 600,
#    value never printed), and generate every deterministic sample input
bash /home/testuser/obs/prepare.sh /home/testuser/pl.env

# 2) bring up the background stack: redis -> qcluster -> document_consumer ->
#    PID-captured `tail -F` observer (each detached, PID-captured, readiness-polled)
bash /home/testuser/obs/bringup.sh

# 3) start the daphne ASGI server that serves the ws/status/ WebSocket
bash /home/testuser/obs/start_asgi.sh

# 4) drive every default-config condition (fail-closed; per-condition log offsets)
bash /home/testuser/obs/drive_default.sh

# 5) (edge) observe the supported-image RGBA sibling (§5.R)
bash /home/testuser/obs/drive_rgba.sh

# 6) safe teardown — exact-PID shutdown + allowlist-only temp removal (no wildcards)
bash /home/testuser/obs/cleanup.sh
```

Runtime versions (live import, **observed**) and pristine pre-run DB/index state (**observed**):

```text
python 3.9.23
django 4.0.4
channels 3.0.4
whoosh 2.7.4
django-q 1.3.9
channels-redis 3.4.0
whoosh 2.7.4
channels 3.0.4
django 4.0.4
```

```text
=== Unapplied migrations? (empty = all applied) ===
unapplied_count=0
=== DB pristine check ===
DOC_COUNT 0
LOGENTRY_COUNT 0
=== INDEX_DIR exists before any run? ===
ls: cannot access '/paperless/data/index': No such file or directory
INDEX_DIR absent
=== harness deps ===
websockets 10.3
requests 2.27.1
```

### A.6 Prior-checkpoint resolution ledger (finding #10)

The prior gate reports were not available as separate files in the workspace; this ledger records, finding-by-finding, how the 19 issues from the latest review (`cr/review`) are resolved in this revision so resolution is independently verifiable.

| # | Sev | Finding (short) | Resolution in this revision |
|---|-----|-----------------|-----------------------------|
| 1 | CRIT | O4 false atomicity | §O4 rewritten: `transaction.atomic()` = DB only; media/index/unlink non-transactional; "no partial state" removed; failure path labeled **inferred**. |
| 2 | CRIT | Edited/impossible "raw" output | All evidence blocks are byte-identical captures spliced verbatim; commentary is prose outside fences. |
| 3 | CRIT | Hard-coded admin password + cookie-prefix logging | One-time random credential via `PL_ADMIN_PW` env; harness prints only status codes + boolean; no password or cookie material anywhere (§1.5, A.3). |
| 4 | MAJ | Screenshots residue + false provenance | Screenshots removed from the workspace; no provenance claim made; literal read-only status in §A.7. |
| 5 | MAJ | Cleanup not demonstrated | §A.4 shows PID-exact shutdown, Redis shutdown, temp removal, absence checks. |
| 6 | MAJ | Commands with `...` | Full orchestration scripts + literal outputs (§1.2, A.3, A.5). |
| 7 | MAJ | `/paperless/*` mislabeled as defaults | §1.1 shows source defaults `/app/*` vs warmed-image `PAPERLESS_*` overrides. |
| 8 | MAJ | observed vs inferred inconsistency | Versions/config now live-import **observed** (§1.1, A.5); labels applied throughout. |
| 9 | MAJ | Non-deterministic listener | Per-run `mktemp -d` 0700, exclusive no-follow file, WS_IDLE timeout, PID capture (§1.5, A.3). |
| 10 | MAJ | Prior reports missing | This ledger. |
| 11 | MAJ | Edge-state coverage gaps | Full before/after DOC/LOG/IDX + source-file for all 8 conditions (§6, §O6, §5.E); the supported-RGBA edge (in-place alpha rewrite, double enqueue, post-parser checksum, late duplicate rejection) is captured with raw evidence in §5.R. |
| 12 | MAJ | Duplicate byte-identity unproven | `cmp`/size/MD5/SHA-256, MD5 == stored checksum (§O5). |
| 13 | MAJ | Auth evidence incomplete/unsafe | Non-sensitive login (200/302/200) + `WS_CONNECTED` (§O3). |
| 14 | MIN | Predictable temp path | `mktemp -d` 0700 + `O_EXCL\|O_NOFOLLOW` 0600 (§1.5, A.3). |
| 15 | MAJ | Logging proof wrong process | Compared from the **qcluster worker** console vs file (§5.L). |
| 16 | MAJ | Blocking startup commands | Orchestrated `setsid`, PID capture, readiness polling (§1.2, A.3). |
| 17 | MIN | task_id "each enqueue" wrong | §O3: watcher path UUID generated at `src/documents/consumer.py:200`; REST supplies at `src/documents/views.py:521`. |
| 18 | MIN | `open_index()` side effect | Disclosed; snapshot uses `os.path.isdir` and avoids `open_index()` (§4, A.3). |
| 19 | MIN | Shorthand citations | Full `src/...:line` throughout; three enqueue sites cited (§O1). |

### A.7 Read-only compliance proof

The source tree is unchanged — no application source, frontend, tests, manifests, or Docker files were modified. This is proven **durably** by diffing against the immutable base commit from which this branch forked, `542221a38dff06361e07976452f9aea24d210542` (the exact commit this investigation targets). Because that base object never changes, the comparison below stays reproducible no matter how many finalization commits are layered onto `HEAD` (**observed**):

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-82f385ec-6e30-44aa-a31e-642e09782a4c

$ git cat-file -t 542221a38dff
commit

$ git diff 542221a38dff..HEAD --name-status
A	blitzy/documentation/paperless-ngx_542221a38dff.md

$ git diff 542221a38dff..HEAD --stat -- src src-ui Pipfile Pipfile.lock requirements.txt Dockerfile docker
(empty output above == zero source/frontend/manifest/Docker lines changed between the base commit and HEAD)
```

> **observed.** Diffed against the fork-point base `542221a38dff`, the *entire* net change introduced on this branch is a **single added file** — the documentation deliverable `blitzy/documentation/paperless-ngx_542221a38dff.md` (status `A`). The scoped `git diff` over `src`, `src-ui`, and the dependency/Docker manifests is **empty**, re-confirming that not one source, frontend, test, dependency, or container line was changed. Anchoring the proof to the immutable base commit rather than a transient `HEAD` revision means it does **not** go stale when the deliverable is (re)committed or amended during finalization: any commit that touches only the deliverable leaves the `base..HEAD` name-status at exactly this one added path. The only investigation by-products — a disposable Redis/SQLite/media/index runtime, temporary observation scripts, and sample documents — live outside the repository or under untracked paths and therefore contribute nothing to the committed tree.
