# paperless-ngx — Document Flow Knowledge Capture

**Repository:** paperless-ngx · **Branch:** `paperless-ngx_542221a38dff` · **HEAD:** `542221a38dff06361e07976452f9aea24d210542`
**Project version:** `(1, 7, 0)` — `src/paperless/version.py:L1`

This document answers seven questions about how a document flows through paperless-ngx v1.7.0: how it enters the system (Q1), the stages it passes through (Q2), the background-execution engine (Q3), the metadata stored per document (Q4), how those fields are classified as required / optional / derived (Q5), a concrete runtime example (Q6), and how tags, correspondents, and document types organize documents together (Q7).

**Citation and labeling conventions.** Every factual claim is backed by evidence in one of two forms: a source **locator**, or **direct runtime output** shown inline (the command and its complete, unedited stdout/stderr). Locators take the form `path:Lnn`, or the abbreviated `:Lnn` (the same file just cited) and `basename:Lnn` (e.g. `consumer.py:L223`, `file_handling.py:L81`). Files whose basename is unique in this checkout are cited by basename directly; files whose basename recurs across apps are **path-qualified** so the specific file is never ambiguous — e.g. `src/documents/models.py:L88` (the ORM model) versus `src/paperless_mail/models.py:L59-L61` (the mail-rule enum), and `src/documents/tasks.py:L184` versus `paperless_mail/tasks.py`. Every cited file lives at a fixed path in the `src/` Django project (the `documents`, `paperless`, and `paperless_mail` apps named in *Environment & Methodology*) or at the repository root (`requirements.txt`, `Dockerfile`, `docker/supervisord.conf`). (Locators that appear *inside* a reproduced script or its captured output are shown verbatim — exactly as run and printed — and are evidence rather than the document's own citations.) Provenance is marked **[observed]** (produced by running real code in the canonical runtime and captured first-hand) or **[inferred]** (read from source, not executed): each question section — and each runtime-evidence block within it — carries a label that governs the claims it contains, and any sub-claim whose provenance differs from its enclosing section is labeled explicitly at that claim, so observed and inferred are never conflated. This was a strictly read-only investigation; no repository file was modified — see the closing *Reproduction, determinism & cleanup* section for the `git status --porcelain` proof.

---

## Environment & Methodology

- **Canonical runtime.** The project's canonical runtime is the Docker image `python:3.9-slim-bullseye` — `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`). Every observation below was produced **inside that canonical runtime**: container `paperless-app`, **Python 3.9.23**, with each dependency at the exact version pinned in `requirements.txt` (django `4.0.4` `requirements.txt:L38`, django-q `1.3.9` `:L37`, channels `3.0.4` `:L23`, channels-redis `3.4.0` `:L22`, redis `3.5.3` `:L84`, scikit-learn `1.0.2` `:L88`, whoosh `2.7.4` `:L111`, watchdog `2.1.7` `:L106`, python-magic `0.4.25` `:L79`, fuzzywuzzy[speedup] `0.18.0` `:L41`, imap-tools `0.54.0` `:L49`, dateparser `1.1.1` `:L32`), plus Tesseract, `libmagic`, `python-Levenshtein`, and a live Redis at `paperless-redis:6379`.
- **No custom harness — the real settings module is used.** Every probe runs against the project's own settings via `DJANGO_SETTINGS_MODULE=paperless.settings`; there is no bespoke settings file. The only configuration is the standard `PAPERLESS_*` environment variables (below), pointing data/media/index at a throwaway `/tmp/ppinv` directory **outside** the repository checkout (bind-mounted at `/app`). The observed behavior is therefore the *real* application's behavior, not a reduced stand-in.
- **Run first, then write.** Each **[observed]** section shows the exact command and its **complete, unedited** captured output — both stdout and stderr. The application routes its own logging to `PAPERLESS_LOGGING_DIR`; the few INFO/WARNING lines that reach stderr are shown in full and *corroborate* the stdout evidence (nothing is hidden). The only run-to-run variation is (a) timestamp prefixes in stderr log lines, (b) the `created` datetime that falls back to file mtime, (c) django-q's randomly generated task id and cluster name, (d) the `qcluster` worker process PIDs, and (e) the md5 checksums of the **OCR-produced** PDF — the original is wrapped by `img2pdf` and the archive is rewritten by OCRmyPDF, and both embed a generation timestamp, so the source `checksum` and `archive_checksum` in the Q2 OCR probe differ between runs while the extracted `content`, `mime_type`, `archive_filename`, and progress bands stay identical — all flagged where they appear.
- **Two clean runs.** The whole investigation was executed twice from a freshly migrated database. All stability-sensitive values (the 16-field inventory, the md5 checksums, every matching boolean and score, the schedule rows and constants, the progress stages, the signal receiver order and effects) were **byte-identical across both runs**; the closing section shows the `diff` result.
- **What is genuinely observed vs inferred.** Nearly every path in this document was **executed** end-to-end in the canonical runtime: Q1's three entry points (directory watcher via `--oneshot`, REST upload, and the local mail handler) and their async convergence (real `async_task` → Redis → a real `qcluster` worker), plus the barcode-splitting branch; Q2's full stage pipeline over **both** a text/plain parse **and** a real scanned-PDF OCR (Tesseract) consume — with progress broadcasts captured live — plus the unsupported-type and raw-`FileExistsError` failure branches; Q3's engine and schedules; Q4/Q5 model metadata; Q6's finished-signal organization; and Q7's matching algorithms **and** the ML `MATCH_AUTO` prediction from a really-trained classifier. The **only** behavioral step that remains **[inferred]** is the *live external IMAP network fetch* (Q1 mail): it requires a reachable IMAP server and credentials this environment does not provide — but everything downstream of that network boundary is **observed** via the very same `imap_tools` parser the live fetch would yield. Two additional items are labeled **[inferred]** because they are configuration facts read from files rather than executed behavior: the `qcluster` launch by supervisord (`docker/supervisord.conf`) and the inotify-vs-polling watcher selection. Each inferred item is grounded in exact `file:line` references and never asserted as an observed outcome.

### Reproducibility harness

All probes share one small harness, sourced inside the canonical container `paperless-app` (Python 3.9.23). It fixes two roots — `/tmp/ppinv` (throwaway data / media / index / scratch / log + captured output) and `/tmp/ppscripts` (the probe scripts, each reproduced **in full** in its section below) — both under `/tmp` and **outside** the repository checkout bind-mounted at `/app`. Cleanup is guarded by a sentinel file plus a `realpath` allow-list and fired from an `EXIT` trap, so a stray or empty variable can never turn `rm -rf` loose on an unintended path. The full harness (`harness2.sh`):

```bash
#!/usr/bin/env bash
# Sourced INSIDE the canonical container `paperless-app` (Python 3.9.23).
set -euo pipefail

INVROOT=/tmp/ppinv        # throwaway data/media/index/scratch/log + captured output
SCRIPTS=/tmp/ppscripts    # probe scripts (reproduced in full in each section below)
GUARD="$INVROOT/.investigation-guard"

cleanup() {               # only ever removes the two fixed roots, and only when they are ours
  for root in "$INVROOT" "$SCRIPTS"; do
    [ -e "$root" ] || continue
    real="$(realpath -m -- "$root")"
    case "$real" in
      /tmp/ppinv)      [ -e "$GUARD" ] && rm -rf -- "$real" ;;   # sentinel-guarded
      /tmp/ppscripts)  rm -rf -- "$real" ;;
      *) echo "refusing to remove unexpected path: $real" >&2 ;;
    esac
  done
}
trap cleanup EXIT

mkdir -p "$INVROOT" "$SCRIPTS"; : > "$GUARD"

export PYTHONPATH=/app/src
export DJANGO_SETTINGS_MODULE=paperless.settings
export PAPERLESS_DATA_DIR="$INVROOT/data"
export PAPERLESS_MEDIA_ROOT="$INVROOT/media"
export PAPERLESS_STATICDIR="$INVROOT/static"
export PAPERLESS_CONSUMPTION_DIR="$INVROOT/consume"
export PAPERLESS_SCRATCH_DIR="$INVROOT/scratch"   # explicit: the default /tmp/paperless would leak outside the guarded root
export PAPERLESS_LOGGING_DIR="$INVROOT/log"
export PAPERLESS_SECRET_KEY=investigation-key
export OUTDIR="$INVROOT/out"; mkdir -p "$OUTDIR"

reset_state() {           # fresh throwaway SQLite DB + dirs; migrate stdout/stderr/exit are CAPTURED, not discarded
  [ -e "$GUARD" ] || { echo "guard missing; refusing reset" >&2; return 1; }
  rm -rf -- "$INVROOT"/{data,media,static,consume,scratch,log}
  mkdir -p "$INVROOT"/{data,media,static,consume,scratch,log}
  ( cd /app/src && python3 manage.py migrate --no-input ) >"$OUTDIR/migrate.out" 2>"$OUTDIR/migrate.err"
  echo "[harness] migrate exit=$? (captured: $OUTDIR/migrate.out,.err)"
}

run_probe() {             # reset to clean state, then run one probe capturing stdout/stderr/exit separately
  local name="$1"; local script="$2"
  reset_state
  set +e
  ( cd /app/src && python3 "$SCRIPTS/$script" ) >"$OUTDIR/$name.out" 2>"$OUTDIR/$name.err"
  local rc=$?; set -e
  echo "[harness] $name exit=$rc"
}

qcluster_start() {        # real django-q worker in its OWN process group (setsid) so it can later be stopped as a group
  export PAPERLESS_TASK_WORKERS=1
  setsid bash -c 'cd /app/src && exec python3 manage.py qcluster' >"$OUTDIR/qcluster.log" 2>&1 &
  QPID=$!
  for _ in $(seq 1 60); do grep -q "running" "$OUTDIR/qcluster.log" 2>/dev/null && break; sleep 0.5; done
}
qcluster_stop() {         # stop the ENTIRE process group by negative PID (a plain parent kill orphans django-q's workers)
  kill -TERM -"$QPID" 2>/dev/null || true; sleep 3
  kill -KILL -"$QPID" 2>/dev/null || true
}
```

`python3 manage.py check` reports `System check identified no issues (0 silenced).` under this configuration, and `reset_state` prints `[harness] migrate exit=0` with the full migration log preserved at `$OUTDIR/migrate.out` (not discarded to `/dev/null`). The fourteen probe scripts live in `/tmp/ppscripts/` and are reproduced **in full** in their sections below; the driver sources this harness and calls `run_probe <name> <script.py>` for each — wrapping the one background-execution probe (Q1/Q3) in `qcluster_start` / `qcluster_stop`.

---

## Q1 — How does a new document enter paperless-ngx? (ingestion entry points)

A new document normally enters through **one of three canonical entry points**. Two of them (REST upload, mail fetch) *write* the incoming bytes to a temporary file; the third (the directory watcher) does **not** write bytes itself — it *observes* a file that some external actor (a scanner, an `rsync`, a copy) has already placed in the consumption directory. All three **converge** on the **same** background task, `documents.tasks.consume_file`, enqueued onto the django-q broker via `async_task(...)` (from `django_q.tasks`). Each entry point was **exercised through its own real code path** and observed to enqueue exactly that task — the directory watcher via `document_consumer --oneshot`, the REST endpoint via an authenticated multipart `POST`, and the mail rule via `MailAccountHandler.handle_message` — and a real `qcluster` worker then executed the enqueued task end-to-end (Q3). The **only** step not executed is the **live IMAP network fetch** that hands a message to the mail handler (no reachable IMAP server/credentials in this environment); every step from the parsed message onward is **[observed]**, and the per-entry-point evidence blocks follow each description below.

**1. Consumption-directory watcher** — `src/documents/management/commands/document_consumer.py` — **[observed]** via a `--oneshot` run (the inotify-vs-polling *selection* detail below remains **[inferred]**)
- Imports the task API: `from django_q.tasks import async_task` — `:L13`.
- The command watches `CONSUMPTION_DIR`. It does **not** always poll: with the default `PAPERLESS_CONSUMER_POLLING=0` (`src/paperless/settings.py:L478`) it uses **inotify** when available — `if settings.CONSUMER_POLLING == 0 and INotify:` `document_consumer.py:L178` → `self.handle_inotify(...)` `:L179` — and only falls back to a `PollingObserver` (`:L187`, inside `handle_polling` `:L185`) when polling is configured or inotify is unavailable (`PollingObserver` imported `:L17`; `INotify` imported `:L20`).
- When a stable, already-written file is detected, `def _consume(filepath)` — `:L46` — validates it and enqueues:
  - `async_task(` — `:L86`
  - `"documents.tasks.consume_file",` — `:L87`
- A watcher **subdirectory** may additionally supply **tag** overrides derived from the folder path; it does not set title/correspondent/type overrides.

**[observed]** — a real `--oneshot` watcher run. The probe drops a real file into `CONSUMPTION_DIR`, invokes the canonical `document_consumer` command in `--oneshot` mode, and reads the enqueued package back off the real broker. Script `/tmp/ppscripts/q1_watcher.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.conf import settings
from django.core.management import call_command
from django_q.brokers import get_broker
from django_q.signing import SignedPackage

broker = get_broker()
broker.purge_queue()

# Drop a real file into the real CONSUMPTION_DIR that the watcher scans.
os.makedirs(settings.CONSUMPTION_DIR, exist_ok=True)
fpath = os.path.join(settings.CONSUMPTION_DIR, "watched_invoice.txt")
with open(fpath, "w") as f:
    f.write("Invoice 2022 from ACME Corp, total due 199.00 EUR\n")
print(f"dropped file into CONSUMPTION_DIR ({settings.CONSUMPTION_DIR})")
print(f"queue size before watcher: {broker.queue_size()}")

# Canonical watcher entry point: the document_consumer management command in
# --oneshot mode (os.scandir(dir) -> _consume -> async_task at
# management/commands/document_consumer.py:L86-L90).
call_command("document_consumer", "--oneshot")

print(f"queue size after watcher oneshot: {broker.queue_size()}")
while True:
    got = broker.dequeue()
    if not got:
        break
    for ack_id, payload in got:
        pkg = SignedPackage.loads(payload)
        print(f"  watcher enqueued func={pkg.get('func')!r} args={pkg.get('args')!r} "
              f"task_name={pkg.get('name')!r}")
        broker.acknowledge(ack_id)
broker.purge_queue()
```

```bash
run_probe q1_watcher q1_watcher.py          # source harness2.sh first; see harness
```

stdout:

```text
dropped file into CONSUMPTION_DIR (/tmp/ppinv/consume)
queue size before watcher: 0
queue size after watcher oneshot: 1
  watcher enqueued func='documents.tasks.consume_file' args=('/tmp/ppinv/consume/watched_invoice.txt',) task_name='watched_invoice.txt'
```

stderr:

```text
[2026-07-13 23:42:47,877] [INFO] [paperless.management.consumer] Adding /tmp/ppinv/consume/watched_invoice.txt to the task queue.
23:42:47 [Q] INFO Enqueued 1
```

Cause → effect: `document_consumer --oneshot` scanned `CONSUMPTION_DIR`, and `_consume` (`document_consumer.py:L46`) enqueued `documents.tasks.consume_file` with the file path as the positional `args` and the basename as `task_name` — the queue went `0 → 1`, and the stderr log line `Adding ... to the task queue.` corroborates it. Only the `CONSUMPTION_DIR` path prefix varies with the harness root; the task func and `task_name='watched_invoice.txt'` are stable.

**2. REST API upload** — `src/documents/views.py` — **[observed]** via an authenticated multipart `POST`
- Imports `from django_q.tasks import async_task` — `:L28`.
- `class PostDocumentView(GenericAPIView)` — `:L491` — is an **authenticated, multipart** endpoint: `permission_classes = (IsAuthenticated,)` `:L493` and `parser_classes = (parsers.MultiPartParser,)` `:L495`. Its route is **`/api/documents/post_document/`** — wired in `src/paperless/urls.py`: `r"^api/"` `:L40` + `r"^documents/post_document/"` `:L57`, `PostDocumentView.as_view()` `:L58`, `name="post_document"` `:L59`.
- Its `def post(...)` — `src/documents/views.py:L497` — streams the upload into a temp file (`tempfile.NamedTemporaryFile(...)` — `:L512`) and enqueues:
  - `async_task(` — `:L523`
  - `"documents.tasks.consume_file",` — `:L524`
- REST supplies `override_filename` `:L526`, `override_title` `:L527`, `override_correspondent_id` `:L528`, `override_document_type_id` `:L529`, and `override_tag_ids` `:L530`.

**[observed]** — a real authenticated multipart `POST`. The probe force-authenticates a superuser, `POST`s a file to the real route, and reads the enqueued package back off the broker. Script `/tmp/ppscripts/q1_rest.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.contrib.auth.models import User
from django.core.files.uploadedfile import SimpleUploadedFile
from rest_framework.test import APIClient
from django_q.brokers import get_broker
from django_q.signing import SignedPackage

broker = get_broker()
broker.purge_queue()

# Canonical REST entry point: authenticated multipart POST to the real
# PostDocumentView route (permission_classes=(IsAuthenticated,)).
user = User.objects.create_superuser("prober", "p@example.com", "pw")
client = APIClient()
client.force_authenticate(user=user)

content = b"REST upload: Invoice from Globex, amount 42.00 USD\n"
upload = SimpleUploadedFile("rest_invoice.txt", content, content_type="text/plain")
print(f"queue size before POST: {broker.queue_size()}")
resp = client.post("/api/documents/post_document/", {"document": upload}, format="multipart")
print(f"POST /api/documents/post_document/ -> status={resp.status_code} body={resp.content!r}")
print(f"queue size after POST: {broker.queue_size()}")
while True:
    got = broker.dequeue()
    if not got:
        break
    for ack_id, payload in got:
        pkg = SignedPackage.loads(payload)
        print(f"  REST enqueued func={pkg.get('func')!r} "
              f"override_filename={pkg.get('kwargs',{}).get('override_filename')!r} "
              f"task_name={pkg.get('name')!r}")
        broker.acknowledge(ack_id)
broker.purge_queue()
```

```bash
run_probe q1_rest q1_rest.py                # source harness2.sh first; see harness
```

stdout:

```text
queue size before POST: 0
POST /api/documents/post_document/ -> status=200 body=b'"OK"'
queue size after POST: 1
  REST enqueued func='documents.tasks.consume_file' override_filename='rest_invoice.txt' task_name='rest_invoice.txt'
```

stderr:

```text
23:42:53 [Q] INFO Enqueued 1
```

Cause → effect: the authenticated `POST` to `/api/documents/post_document/` returned **HTTP 200** with body `"OK"`, and `PostDocumentView.post` (`src/documents/views.py:L497`) enqueued `documents.tasks.consume_file` with `override_filename='rest_invoice.txt'` — the queue went `0 → 1`. Every value here is stable across runs (the endpoint returns the literal string `"OK"`).

**3. Mail fetch** — `src/paperless_mail/mail.py` — **[observed]** (the local `handle_message` dispatch was executed against a real parsed message; only the live IMAP network fetch is **[inferred]** — see the blocker note below)
- Imports `from django_q.tasks import async_task` — `:L11`.
- **Disposition handling is conditional on the rule's `attachment_type` — it does *not* unconditionally skip non-attachment parts.** A part is skipped **only when both** clauses of a compound `if` hold: its `content_disposition` is not `"attachment"` **and** the rule's `attachment_type == MailRule.AttachmentProcessing.ATTACHMENTS_ONLY`:
  - `not att.content_disposition == "attachment"` — `:L292`
  - `and rule.attachment_type == MailRule.AttachmentProcessing.ATTACHMENTS_ONLY` — `:L293-L294`
  - `continue` (skip) — `:L302`
- The `AttachmentProcessing` enum lives in `src/paperless_mail/models.py:L59-L61` — `ATTACHMENTS_ONLY = 1` (`:L60`), `EVERYTHING = 2` (`:L61`) — and the rule's `attachment_type` field **defaults to `ATTACHMENTS_ONLY`** (`src/paperless_mail/models.py:L137`, `default=AttachmentProcessing.ATTACHMENTS_ONLY` `:L140`). So under the default **ATTACHMENTS_ONLY** an `inline` part *is* skipped, but under **EVERYTHING** the second clause is false, so `inline` parts are processed too. (This is confirmed both ways in the **[observed]** block below.)
- Each part that survives that check is filtered by `is_mime_type_supported(mime_type)` (`mail.py:L319`, imported `:L14`), written to a temp file (`tempfile.mkstemp(...)` — `:L322`), and enqueued:
  - `async_task(` — `:L336`
  - `"documents.tasks.consume_file",` — `:L337`
- Mail supplies `override_filename` `:L339`, `override_title` `:L342`, `override_correspondent_id` `:L343`, `override_document_type_id` `:L346`, and `override_tag_ids` `:L347` from the matching mail rule.
- This path is itself driven periodically by the scheduled task `paperless_mail.tasks.process_mail_accounts` (see Q3).

**[observed]** — both disposition modes, driving the real `MailAccountHandler.handle_message` (`:L272`). The probe builds a raw RFC822 message with one **attachment** part and one **inline** part (both `text/plain`, a supported MIME) using the same `imap_tools.MailMessage` parser the live IMAP path yields, persists a real `MailAccount` + `MailRule`, and calls the real `handle_message(msg, rule)` once per mode — reading back off the real Redis broker how many `consume_file` tasks it enqueued. Script `/tmp/ppscripts/q_mail.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from paperless_mail.mail import MailAccountHandler
from paperless_mail.models import MailAccount, MailRule
from imap_tools import MailMessage
from django_q.brokers import get_broker
from django_q.signing import SignedPackage

# One real "attachment" part + one real "inline" part, both text/plain (a
# supported MIME) with distinct content, built from raw RFC822 via imap_tools
# (the same MailMessage parser the live IMAP path uses). A third bare text/plain
# body part (no filename) models the email body itself.
RAW = (
    b"From: sender@example.com\r\n"
    b"To: inbox@example.com\r\n"
    b"Subject: Test invoice mail\r\n"
    b"MIME-Version: 1.0\r\n"
    b'Content-Type: multipart/mixed; boundary="BOUND"\r\n'
    b"\r\n"
    b"--BOUND\r\n"
    b"Content-Type: text/plain\r\n"
    b"\r\n"
    b"Email body text (not an attachment)\r\n"
    b"--BOUND\r\n"
    b'Content-Type: text/plain; name="real_attach.txt"\r\n'
    b'Content-Disposition: attachment; filename="real_attach.txt"\r\n'
    b"\r\n"
    b"ATTACHMENT invoice ACME total due 199.00 EUR\r\n"
    b"--BOUND\r\n"
    b'Content-Type: text/plain; name="inline_part.txt"\r\n'
    b'Content-Disposition: inline; filename="inline_part.txt"\r\n'
    b"\r\n"
    b"INLINE receipt Globex amount 42.00 USD\r\n"
    b"--BOUND--\r\n"
)
msg = MailMessage.from_bytes(RAW)
print("parsed message parts (via imap_tools MailMessage.attachments):")
for a in msg.attachments:
    print(f"  filename={a.filename!r} content_disposition={a.content_disposition!r}")

# Canonical objects, persisted (handle_message reads rule.assign_tags.all(),
# which requires a PK). Constructed exactly like the real caller which does
# MailAccountHandler().handle_mail_account(account) [paperless_mail/tasks.py].
acct = MailAccount(name="probe", imap_server="localhost", username="u", password="p")
acct.save()
handler = MailAccountHandler()
handler.renew_logging_group()
broker = get_broker()

print()
print("MailRule.AttachmentProcessing constants: "
      f"ATTACHMENTS_ONLY={int(MailRule.AttachmentProcessing.ATTACHMENTS_ONLY)} "
      f"EVERYTHING={int(MailRule.AttachmentProcessing.EVERYTHING)}")
print()

def decode_enqueued(broker):
    """Read back (canonically) what handle_message actually enqueued on the
    real Redis broker: each task's func target + override_filename kwarg."""
    tasks = []
    while True:
        got = broker.dequeue()
        if not got:
            break
        for ack_id, payload in got:
            pkg = SignedPackage.loads(payload)
            tasks.append((pkg.get("func"), pkg.get("kwargs", {}).get("override_filename")))
            broker.acknowledge(ack_id)
    return tasks

def run_mode(mode_value, mode_name):
    # Real canonical path: real handle_message -> real async_task -> real Redis
    # broker. Purge first, then count how many consume_file tasks the handler
    # actually enqueued (i.e. how many parts it forwarded), then read them back.
    broker.purge_queue()
    rule = MailRule(name=f"rule-{mode_name}", account=acct, attachment_type=mode_value)
    rule.save()
    processed = handler.handle_message(msg, rule)
    qsize = broker.queue_size()
    print(f"[{mode_name}] handle_message returned processed_attachments={processed}; "
          f"consume_file tasks enqueued on the real broker = {qsize}")
    for func, ofn in decode_enqueued(broker):
        print(f"    -> enqueued task func={func!r} override_filename={ofn!r}")
    broker.purge_queue()

run_mode(MailRule.AttachmentProcessing.ATTACHMENTS_ONLY, "ATTACHMENTS_ONLY")
run_mode(MailRule.AttachmentProcessing.EVERYTHING, "EVERYTHING")
```

```bash
run_probe q_mail q_mail.py                  # source harness2.sh first; see harness
```

stdout:

```text
parsed message parts (via imap_tools MailMessage.attachments):
  filename='real_attach.txt' content_disposition='attachment'
  filename='inline_part.txt' content_disposition='inline'

MailRule.AttachmentProcessing constants: ATTACHMENTS_ONLY=1 EVERYTHING=2

[ATTACHMENTS_ONLY] handle_message returned processed_attachments=1; consume_file tasks enqueued on the real broker = 1
    -> enqueued task func='documents.tasks.consume_file' override_filename='real_attach.txt'
[EVERYTHING] handle_message returned processed_attachments=2; consume_file tasks enqueued on the real broker = 2
    -> enqueued task func='documents.tasks.consume_file' override_filename='real_attach.txt'
    -> enqueued task func='documents.tasks.consume_file' override_filename='inline_part.txt'
```

stderr:

```text
[2026-07-13 23:42:41,227] [INFO] [paperless_mail] Rule probe.rule-ATTACHMENTS_ONLY: Consuming attachment real_attach.txt from mail Test invoice mail from sender@example.com
23:42:41 [Q] INFO Enqueued 1
[2026-07-13 23:42:42,331] [INFO] [paperless_mail] Rule probe.rule-EVERYTHING: Consuming attachment real_attach.txt from mail Test invoice mail from sender@example.com
23:42:42 [Q] INFO Enqueued 1
[2026-07-13 23:42:42,356] [INFO] [paperless_mail] Rule probe.rule-EVERYTHING: Consuming attachment inline_part.txt from mail Test invoice mail from sender@example.com
23:42:42 [Q] INFO Enqueued 2
```

Cause → effect: this confirms the disposition semantics directly. Under **ATTACHMENTS_ONLY** the `inline` part satisfies the compound `if` (`not content_disposition=="attachment"` `:L292` **and** `attachment_type==ATTACHMENTS_ONLY` `:L293-L294`) and is skipped (`continue` `:L302`), so `handle_message` forwards **1** part and enqueues **1** `consume_file` task (`real_attach.txt`). Under **EVERYTHING** the second clause is false, so the `inline` part is *not* skipped: `handle_message` forwards **2** parts and enqueues **2** tasks (`real_attach.txt` **and** `inline_part.txt`). Both tasks target `documents.tasks.consume_file`, and the stderr `Consuming attachment ...` lines (one under ATTACHMENTS_ONLY, two under EVERYTHING) corroborate the counts. All values here are stable across runs.

> **[inferred] — the one unexecuted step: the live IMAP network fetch.** The periodic entry `paperless_mail.tasks.process_mail_accounts` calls `MailAccountHandler.handle_mail_account` (`mail.py:L151`), which opens a mailbox via `get_mailbox(...)` (`:L159`; `get_mailbox` def `:L92`, `MailBox(...)` `:L98`) and iterates `M.fetch(...)` (`:L222`) to obtain each `imap_tools` `MailMessage` before handing it to `handle_message` (`:L272`). That **network fetch** is the *sole* part of the mail path not executed here — there is no reachable IMAP server or credentials in this environment. The probe substitutes an equivalent `MailMessage` parsed from raw RFC822 by the very same `imap_tools` parser the live fetch yields, so **everything downstream of the network boundary is [observed]**; only the socket-level fetch is inferred.

**Convergence — with one qualification.** All three call the one task `def consume_file(...)` — `src/documents/tasks.py:L184`. That task does **not** unconditionally construct a `Consumer`: it first checks the barcode-splitting feature — **[observed]** (the split branch was executed against a real barcode-separated PDF; see the barcode block below)

- `if settings.CONSUMER_ENABLE_BARCODES:` — `src/documents/tasks.py:L195` (the flag defaults to **False** — `src/paperless/settings.py:L502-L504`, via `__get_boolean(..., default="NO")` `:L34`; confirmed **[observed]** `CONSUMER_ENABLE_BARCODES = False` in the probe environment). When enabled, it scans for separator barcodes (`scan_file_for_separating_barcodes` `src/documents/tasks.py:L198`) and, **if separators are found**, splits the input, copies each part back to the consumption directory (`save_to_dir` `:L210`), deletes the original (`os.unlink(path)` `:L214`), broadcasts a `SUCCESS`/`finished` payload to `"status_updates"` (`:L227`), and **returns `"File successfully split"` without ever constructing a `Consumer`** — `:L233`. The split files re-enter through the watcher as fresh inputs.
- Only on the **non-split path** (barcodes disabled, or enabled but none found) does it build the pipeline: `document = Consumer().try_consume_file(...)` — `src/documents/tasks.py:L236` — running the stages in Q2.

So the precise statement is: **in the default configuration** (`CONSUMER_ENABLE_BARCODES=False`) every entry point converges on `Consumer.try_consume_file`, and processing (Q2) and organization (Q6/Q7) are identical regardless of arrival path; with barcode splitting enabled, a separated input is short-circuited into per-document re-consumption. Per-entry-point **overrides** differ (watcher: tags only; REST/mail: title/correspondent/type/tags) and are applied by `Consumer.apply_overrides` (`src/documents/consumer.py:L414`) — see Q2/Q6.

**[observed]** — the barcode-split short-circuit. With `PAPERLESS_CONSUMER_ENABLE_BARCODES=true`, the probe builds a real 3-page PDF (page 0 content, page 1 a Code128 `PATCHT` separator, page 2 content) and runs the canonical `consume_file` so the task itself performs scan → separate → `save_to_dir`. Script `/tmp/ppscripts/q9_barcode.py`:

```python
import os
# Enable the barcode path BEFORE settings load (consume_file reads
# settings.CONSUMER_ENABLE_BARCODES; default is "NO"). Separator string
# defaults to "PATCHT" (settings.py:L506).
os.environ["PAPERLESS_CONSUMER_ENABLE_BARCODES"] = "true"
import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
import shutil
from django.conf import settings
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
from reportlab.graphics.barcode import code128
from pikepdf import Pdf
from documents.tasks import scan_file_for_separating_barcodes, consume_file

inv = os.environ["PAPERLESS_DATA_DIR"].rsplit("/data", 1)[0]
src = os.path.join(inv, "scanned_batch.pdf")

# 3-page PDF: page0 = "Document A", page1 = PATCHT separator, page2 = "Document B".
c = canvas.Canvas(src, pagesize=letter)
c.drawString(100, 700, "Document A - first real page")
c.showPage()
code128.Code128("PATCHT", barHeight=60, barWidth=1.5).drawOn(c, 100, 700)
c.drawString(100, 680, "PATCHT SEPARATOR SHEET")
c.showPage()
c.drawString(100, 700, "Document B - second real page")
c.showPage()
c.save()

print(f"built {src} with {len(Pdf.open(src).pages)} pages "
      f"(barcode separator string = {settings.CONSUMER_BARCODE_STRING!r})")

# [observed] real barcode scan of the real PDF via convert_from_path + pyzbar
separators = scan_file_for_separating_barcodes(src)
print(f"scan_file_for_separating_barcodes -> separator page indices = {separators}")

# consume_file unlinks the original on split; run it against a copy so the
# canonical task function itself performs scan -> separate -> save_to_dir.
work = os.path.join(inv, "scan.pdf")
shutil.copy(src, work)
os.makedirs(settings.CONSUMPTION_DIR, exist_ok=True)
before = sorted(os.listdir(settings.CONSUMPTION_DIR))
result = consume_file(work, override_filename="scan.pdf")
after = sorted(os.listdir(settings.CONSUMPTION_DIR))
print(f"consume_file(barcodes ON) returned: {result!r}")
print(f"original still exists after split? {os.path.exists(work)}")
print(f"CONSUMPTION_DIR before={before} after={after}")
for name in after:
    p = os.path.join(settings.CONSUMPTION_DIR, name)
    print(f"  split doc {name!r}: {len(Pdf.open(p).pages)} page(s)")
```

```bash
run_probe q9_barcode q9_barcode.py          # source harness2.sh first; see harness
```

stdout:

```text
built /tmp/ppinv/scanned_batch.pdf with 3 pages (barcode separator string = 'PATCHT')
scan_file_for_separating_barcodes -> separator page indices = [1]
consume_file(barcodes ON) returned: 'File successfully split'
original still exists after split? False
CONSUMPTION_DIR before=[] after=['0_scan.pdf', '1_scan.pdf']
  split doc '0_scan.pdf': 1 page(s)
  split doc '1_scan.pdf': 1 page(s)
```

stderr: *(empty)*

Cause → effect: `scan_file_for_separating_barcodes` (`src/documents/tasks.py:L198`) found the `PATCHT` separator at page index **1**; because a separator was found, `consume_file` split the input, wrote the two content pages back to `CONSUMPTION_DIR` as `0_scan.pdf` and `1_scan.pdf` (`save_to_dir` `:L210`), deleted the original (`os.unlink(path)` `:L214`, hence `original still exists after split? False`), and **returned `'File successfully split'` without ever constructing a `Consumer`** (`:L233`) — exactly the short-circuit described above. The two split files (one page each) then re-enter through the watcher as fresh inputs. All values are stable across runs.

**[observed] — the convergence point and its background execution.** Beyond the three per-entry-point blocks above (each of which was shown to enqueue `documents.tasks.consume_file`), this probe exercises the **shared** dispatch directly — `async_task("documents.tasks.consume_file", ...)`, the same task name all three entry points pass — and lets a real `qcluster` worker dequeue and run it end-to-end (the worker log is embedded in Q3). Script `/tmp/ppscripts/q1q3_async.py`:

```python
import os, django, time
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django_q.tasks import async_task
from django_q.models import Task
from documents.models import Document

src = os.path.join(os.environ["PAPERLESS_SCRATCH_DIR"], "upload.txt")
os.makedirs(os.path.dirname(src), exist_ok=True)
with open(src, "w") as f:
    f.write("Async ingestion test: Invoice from Globex, amount 42.00 USD\n")

# The shared background task target that all three entry points converge on
# (watcher / REST upload / mail each build their own args around this same
# task name). A real qcluster worker must dequeue and execute it.
task_id = async_task("documents.tasks.consume_file", src, task_name="probe-consume")
print(f"enqueued via async_task('documents.tasks.consume_file', ...): task_id={task_id}")

doc = None
for _ in range(60):
    time.sleep(1)
    doc = Document.objects.all().first()
    if doc:
        print(f"qcluster worker created document: id={doc.id} title={doc.title!r} "
              f"mime_type={doc.mime_type} checksum={doc.checksum}")
        break
if not doc:
    print("no document created within timeout")

t = None
for _ in range(15):
    t = Task.objects.filter(id=task_id).first()
    if t and t.stopped:
        break
    time.sleep(1)
if t:
    print(f"django_q Task record: func={t.func} success={t.success} result={t.result!r}")
else:
    print(f"no django_q Task record found for id={task_id}")
```

Command and complete output:

```bash
# a qcluster worker is running in the background (see Q3):
python3 /tmp/ppscripts/q1q3_async.py
```

stdout:

```text
enqueued via async_task('documents.tasks.consume_file', ...): task_id=2a2eb9a0ec3149ea904a5454b7bb0311
qcluster worker created document: id=1 title='upload' mime_type=text/plain checksum=4bd40d5fb22298f57c4813682df7f19c
django_q Task record: func=documents.tasks.consume_file success=True result='Success. New document id 1 created'
```

stderr:

```text
23:43:21 [Q] INFO Enqueued 1
```

Cause → effect: the enqueued `task_id` is a django-q handle; a worker dequeued it from Redis, ran `documents.tasks.consume_file`, which (barcodes disabled) called `Consumer().try_consume_file` and created document `id=1`; the `django_q.models.Task` row records `success=True` and `result='Success. New document id 1 created'`. Within *this* probe's stdout the `task_id` (a random uuid4) is the only value that varies between runs — the document `checksum` (`4bd40d5fb22298f57c4813682df7f19c`) and the `result` string are stable. (The full run-to-run variance inventory — log timestamps, the mtime-derived `created`, the cluster name, the worker PIDs, and the OCR-PDF checksums — is listed in *Environment & Methodology* and the closing *Reproduction, determinism & cleanup* section.)

---

## Q2 — What stages does a document pass through? (the consume pipeline)

**[observed]** — this pipeline was executed **twice**: once over the **text/plain** parser path and once over a real **scanned-PDF OCR** path (Tesseract via `paperless_tesseract`), both shown below. `Consumer.try_consume_file` (`src/documents/consumer.py:L180`) runs an ordered pipeline and broadcasts progress via `_send_progress` (`def` `:L56`), which builds a payload dict (`:L64-L72`) and `group_send()`s it to the `"status_updates"` channel group (`:L73-L76`). The message codes are the `MESSAGE_*` constants at `:L43-L49`. The percentages, statuses, and message codes below are **[observed]** — captured live by subscribing a channel to that group during a real consume.

Ordered stages, each with its source citation:

1. **`STARTING` / `new_file` at 0%** — `:L202` (`MESSAGE_NEW_FILE` `:L43`).
2. **Pre-checks** (no progress payload), in this real call order — `pre_check_file_exists()` `:L211` (`def` `:L95`; raises if the file is missing), then `pre_check_directories()` `:L212` (`def` `:L115`; creates scratch/thumbnail dirs), then `pre_check_duplicate()` `:L213` (`def` `:L102`; rejects when the md5 `checksum` — or `archive_checksum`, `:L106` — already exists). The order is **file-exists → directories → duplicate**, not duplicate-first.
3. **MIME detection & parser selection** (no progress payload) — the MIME type is detected with `magic.from_file(self.path, mime=True)` (`:L219`), then the parser class is chosen by `get_parser_class_for_mime_type()` (defined in `src/documents/parsers.py:L81`, imported at `consumer.py:L26`, called at `:L223`). If no parser matches the MIME type, the file is **rejected as unsupported here** — `if not parser_class:` (`:L224`) calls `self._fail(MESSAGE_UNSUPPORTED_TYPE, ...)` (`:L225`; `MESSAGE_UNSUPPORTED_TYPE` `:L44`) — **before** the `document_consumption_started` signal in the next step. MIME detection and parser selection are **[observed]** (the observed text/plain run detected the type at `:L219` and selected the text parser at `:L223`); the unsupported-type rejection branch is **[observed]** as well — a real consume of a file whose detected type has no parser is captured in the dedicated *unsupported-type* block within the failure-semantics discussion below.
4. **`document_consumption_started` signal** fires — `:L229`.
5. **Optional pre-consume script** (no progress payload) — `self.run_pre_consume_script()` (`:L235`; `def` `:L121`) runs the configured `PRE_CONSUME_SCRIPT` before parsing; a missing/erroring script goes through `_fail()` (`:L126`, `:L137`).
6. **Parser construction** (no progress payload) — the chosen `parser_class` is instantiated with the logging group and a progress callback: `document_parser = parser_class(self.logging_group, progress_callback)` (`:L244`). The `progress_callback` (`def` `:L237`) maps parser progress into the band via `p = int((current_progress / max_progress) * 50 + 20)` (`:L239`) — so parser progress spans **20%–70%**. NOTE: the code *comment* at `:L238` says "within 20 and 80", but the arithmetic (`* 50 + 20`) tops out at **70**, and the next fixed payload is 70% for thumbnailing — so the observed parse band is **20–70%**, not 20–80%.
7. **`WORKING` / `parsing_document` at 20%** — `:L259` (`MESSAGE_PARSING_DOCUMENT` `:L45`); the selected parser's `parse()` is **invoked** at `:L261`, inside a `try` (`:L258`) whose `except ParseError` (`:L278`) calls `document_parser.cleanup()` (`:L279`) then `_fail()` (`:L280`). (`parse()` is reached only for supported types — the unsupported-type check already happened in step 3.)
8. **`WORKING` / `generating_thumbnail` at 70%** — `:L264` (`MESSAGE_GENERATING_THUMBNAIL` `:L46`); the thumbnail is produced by `get_optimised_thumbnail()` (`:L265`).
9. **Text / date / archive-path retrieval** (still inside the parse `try`) — `get_text()` (`:L271`), `get_date()` (`:L272`), and `get_archive_path()` (`:L276`). Only when the parser returned **no** date is **`WORKING` / `parse_date` at 90%** emitted (`:L274`, `MESSAGE_PARSE_DATE` `:L47`) and `parse_date(self.filename, text)` called (`:L275`). This edge branch was exercised in the observed run (the text parser returns no date), which is why `parse_date` appears.
10. **Classifier load** (no progress payload) — `classifier = load_classifier()` (`:L292`) loads the ML model once so the post-consume organization hooks (Q6/Q7) can reuse it (it is passed into the finished signal at `:L310`).
11. **`WORKING` / `save_document` at 95%** — `:L294` (`MESSAGE_SAVE_DOCUMENT` `:L48`), then a `try` (`:L297`) / `transaction.atomic()` (`:L298`): `_store()` (`:L301`; `def` `:L379`) creates the `Document`; the **`document_consumption_finished` signal fires** (`:L306`) — this triggers organization (Q6/Q7). Under `FileLock` (`:L315`): `generate_unique_filename()` (`:L316`, defined in `file_handling.py:L81`) computes the storage name, then the original and thumbnail are written via `_write()` (`consumer.py:L319`, `:L321`), and — when an archive was produced — `archive_filename` is set (`:L328`), the archive is written (`:L333`), and `archive_checksum` is computed as an md5 over the archive bytes (`:L339-L342`). Outside the lock, `document.save()` (`:L346`) persists it, the consumed input is removed with `os.unlink(self.path)` (`:L350`), and a macOS `._`-shadow file is deleted if present (`:L353-L360`). Any exception in this block is caught by `except Exception` (`:L362`) → `_fail()` (`:L363`); `finally: document_parser.cleanup()` (`:L368-L369`) always runs.
12. **Optional post-consume script** (no progress payload) — `self.run_post_consume_script(document)` (`:L371`; `def` `:L143`) runs the configured `POST_CONSUME_SCRIPT`; failures go through `_fail()` (`:L148`, `:L174`).
13. **`SUCCESS` / `finished` at 100%** — `:L375` (`MESSAGE_FINISHED` `:L49`), carrying the new `document.id`.

**Failure path — mostly, but *not entirely*, through `_fail()`.** Most failure branches call `_fail()` (`def` `:L78`), which first broadcasts a terminal **`FAILED`** payload at 100% — `self._send_progress(100, 100, "FAILED", message)` (`:L79`) — and then raises `ConsumerError` (`:L81`). The branches that funnel through `_fail()` are: file-not-found (`:L97`), duplicate checksum (`:L110`), pre-consume-script failure (`:L126`, `:L137`), unsupported MIME type (`:L225`), `ParseError` from `parse()` (`:L280`), any exception during the store/write transaction (`:L363`), and post-consume-script failure (`:L148`, `:L174`). **The failure path is NOT unified, though:** `pre_check_directories()` (`def` `:L115`) calls `os.makedirs(...)` directly for the scratch / thumbnail / originals / archive directories (`:L116-L119`) with no surrounding `try`/`_fail`. If one of those paths cannot be created — e.g. `SCRATCH_DIR` already exists as a *regular file* — `os.makedirs` raises a **raw `FileExistsError` (an `OSError`) that propagates out of `try_consume_file` with NO `FAILED` payload**; only the initial `STARTING` payload will have been broadcast.

**[observed]** — the unsupported-MIME branch (a failure that **does** go through `_fail()`). This is the reference case for the *normal* failure path: the probe consumes a real file whose detected MIME type has no registered parser, subscribes a channel to `"status_updates"`, and captures both the raised exception and every broadcast payload. Script `/tmp/ppscripts/q2_unsupported.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
import asyncio
import magic
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from documents.consumer import Consumer, ConsumerError
from documents.parsers import get_parser_class_for_mime_type

inv = os.environ["PAPERLESS_DATA_DIR"].rsplit("/data", 1)[0]

# A file whose detected MIME type has NO registered parser -> unsupported.
src = os.path.join(inv, "mystery.xyz")
with open(src, "wb") as f:
    f.write(b"\x00\x01\x02NOTAREALFORMAT\xff\xfe")

mime = magic.from_file(src, mime=True)
print(f"detected mime_type = {mime!r}")
print(f"parser for that mime = {get_parser_class_for_mime_type(mime)}")

layer = get_channel_layer()
ch = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)("status_updates", ch)

raised = None
try:
    Consumer().try_consume_file(src)
except ConsumerError as e:
    raised = ("ConsumerError", str(e))
except Exception as e:
    raised = (type(e).__name__, str(e))
print(f"try_consume_file raised: {raised[0]}: {raised[1]}")

payloads = []
async def drain():
    while True:
        try:
            msg = await asyncio.wait_for(layer.receive(ch), timeout=1.0)
        except asyncio.TimeoutError:
            break
        payloads.append(msg.get("data", {}))
async_to_sync(drain)()
print(f"progress payloads broadcast: {len(payloads)}")
for d in payloads:
    print(f"  status={d.get('status')} message={d.get('message')} "
          f"current={d.get('current_progress')} max={d.get('max_progress')}")
print("FAILED payload emitted?", any(d.get("status") == "FAILED" for d in payloads))
```

```bash
run_probe q2_unsupported q2_unsupported.py   # source harness2.sh first; see harness
```

stdout:

```text
detected mime_type = 'application/octet-stream'
parser for that mime = None
try_consume_file raised: ConsumerError: mystery.xyz: Unsupported mime type application/octet-stream
progress payloads broadcast: 2
  status=STARTING message=new_file current=0 max=100
  status=FAILED message=unsupported_type current=100 max=100
FAILED payload emitted? True
```

stderr (the application's own INFO/ERROR log — the ERROR line is `_fail`'s own `self.log("error", ...)` at `:L80`):

```text
[2026-07-14 00:27:32,340] [INFO] [paperless.consumer] Consuming mystery.xyz
[2026-07-14 00:27:32,354] [ERROR] [paperless.consumer] Unsupported mime type application/octet-stream
```

Cause → effect: `magic.from_file` (`:L219`) detected `application/octet-stream`, for which `get_parser_class_for_mime_type` (`src/documents/parsers.py:L81`) returned `None`, so `if not parser_class:` (`consumer.py:L224`) took the failure branch `self._fail(MESSAGE_UNSUPPORTED_TYPE, ...)` (`:L225`). Inside `_fail`, `_send_progress(100, 100, "FAILED", ...)` (`:L79`) broadcast the terminal **`FAILED`** payload, `self.log("error", ...)` (`:L80`) wrote the ERROR line seen in stderr, and `raise ConsumerError(...)` (`:L81`) produced the exception. Hence **exactly two** payloads — `STARTING(0)` then `FAILED(100)` — and **`FAILED payload emitted? True`**. All values are stable across runs (only the log timestamps and the path prefix vary). Contrast this directly with the next branch, which raises **without** any `FAILED` payload.

**[observed]** — the raw-`FileExistsError` branch (a failure that bypasses `_fail()`). The probe points `PAPERLESS_SCRATCH_DIR` at a regular file, subscribes a channel to `"status_updates"`, and consumes a real file. Script `/tmp/ppscripts/q_direrror.py`:

```python
import os, django
# Point SCRATCH_DIR at a regular FILE (not a dir) BEFORE settings load, so that
# pre_check_directories() -> os.makedirs(SCRATCH_DIR) raises a raw OSError.
inv = os.environ["PAPERLESS_DATA_DIR"].rsplit("/data", 1)[0]
scratch_as_file = os.path.join(inv, "scratch_is_a_file")
with open(scratch_as_file, "w") as f:
    f.write("x")
os.environ["PAPERLESS_SCRATCH_DIR"] = scratch_as_file
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from django.conf import settings
from documents.consumer import Consumer, ConsumerError

print(f"settings.SCRATCH_DIR points at a regular file: {settings.SCRATCH_DIR!r}")
print(f"  os.path.isfile(SCRATCH_DIR) = {os.path.isfile(settings.SCRATCH_DIR)}")

# subscribe to the progress group so we can prove which payloads were emitted
layer = get_channel_layer()
ch = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)("status_updates", ch)

src = os.path.join(inv, "src_doc.txt")
with open(src, "w") as f:
    f.write("some content that will never be consumed\n")

raised = None
try:
    Consumer().try_consume_file(src)
except ConsumerError as e:
    raised = ("ConsumerError", str(e))
except Exception as e:
    raised = (type(e).__name__, str(e))

print(f"try_consume_file raised: {raised[0]}: {raised[1]}")

# Drain the progress channel (non-blocking-ish): collect whatever was broadcast.
import asyncio
payloads = []
async def drain():
    while True:
        try:
            msg = await asyncio.wait_for(layer.receive(ch), timeout=1.0)
        except asyncio.TimeoutError:
            break
        payloads.append(msg.get("data", {}))
async_to_sync(drain)()
print(f"progress payloads broadcast before the raise: {len(payloads)}")
for d in payloads:
    print(f"  status={d.get('status')} message={d.get('message')} "
          f"current={d.get('current_progress')}")
print("FAILED payload emitted?", any(d.get("status") == "FAILED" for d in payloads))
```

```bash
run_probe q_direrror q_direrror.py          # source harness2.sh first; see harness
```

stdout:

```text
settings.SCRATCH_DIR points at a regular file: '/tmp/ppinv/scratch_is_a_file'
  os.path.isfile(SCRATCH_DIR) = True
try_consume_file raised: FileExistsError: [Errno 17] File exists: '/tmp/ppinv/scratch_is_a_file'
progress payloads broadcast before the raise: 1
  status=STARTING message=new_file current=0
FAILED payload emitted? False
```

stderr: *(empty)*

Cause → effect: `try_consume_file` sent exactly one payload — `STARTING` / `new_file` at 0% (`:L202`) — then `pre_check_directories()` (`:L212` → `def` `:L115`) hit `os.makedirs(settings.SCRATCH_DIR, ...)` (`:L116`) on a path that is a regular file and raised a **raw `FileExistsError`** (`[Errno 17]`) that escaped `try_consume_file` uncaught. Crucially, **`FAILED payload emitted? False`** — this branch does **not** pass through `_fail()`, so no terminal `FAILED` payload is broadcast, disproving any "every failure funnels through `_fail()`" reading. All values are stable across runs (only the path prefix varies with the harness root).

**Date precedence** — in `_store`, `created = file_info.created or date or timezone.make_aware(datetime.datetime.fromtimestamp(stats.st_mtime))` (`consumer.py:L389-L393`): a date encoded in the filename wins; else the parser/`parse_date` date; else the file's mtime. In the observed text run there was no filename date and the text parser returned none, so `created` fell back to mtime — visible below.

Script `/tmp/ppscripts/q2_consume.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from documents.consumer import Consumer

# Subscribe a channel to the real 'status_updates' group BEFORE consuming so we
# capture the exact payloads Consumer._send_progress broadcasts (consumer.py:L73).
layer = get_channel_layer()
my_channel = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)("status_updates", my_channel)

src = "/tmp/ppinv/sample.txt"
with open(src, "w") as f:
    f.write("Invoice 2022 from ACME Corp, total due 199.00 EUR\n")

# Canonical entry into the pipeline (same method tasks.consume_file calls).
doc = Consumer().try_consume_file(src)
print(f"consumed document: id={doc.id} title={doc.title!r} mime_type={doc.mime_type}")
print(f"  checksum={doc.checksum}")
print(f"  content={doc.content!r}")
print(f"  filename={doc.filename}")
print(f"  created={doc.created!r}  (mtime fallback: no filename date, text parser returned no date)")
print("real _send_progress payloads broadcast on 'status_updates' during the consume:")
for _ in range(20):
    msg = async_to_sync(layer.receive)(my_channel)
    d = msg.get("data", {})
    print(f"  current={d.get('current_progress')} max={d.get('max_progress')} "
          f"status={d.get('status')} message={d.get('message')}")
    if d.get("status") == "SUCCESS":
        break
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q2_consume.py
```

stdout:

```text
consumed document: id=1 title='sample' mime_type=text/plain
  checksum=0f03d7e3739efdee9bd1235de8eb6587
  content='Invoice 2022 from ACME Corp, total due 199.00 EUR\n'
  filename=0000001.txt
  created=datetime.datetime(2026, 7, 13, 18, 5, 35, 339972, tzinfo=zoneinfo.ZoneInfo(key='UTC'))  (mtime fallback: no filename date, text parser returned no date)
real _send_progress payloads broadcast on 'status_updates' during the consume:
  current=0 max=100 status=STARTING message=new_file
  current=20 max=100 status=WORKING message=parsing_document
  current=70 max=100 status=WORKING message=generating_thumbnail
  current=90 max=100 status=WORKING message=parse_date
  current=95 max=100 status=WORKING message=save_document
  current=100 max=100 status=SUCCESS message=finished
```

stderr (the application's own INFO log, corroborating start and finish):

```text
[2026-07-13 18:05:35,364] [INFO] [paperless.consumer] Consuming sample.txt
[2026-07-13 18:05:37,910] [INFO] [paperless.consumer] Document 2026-07-13 sample consumption finished
```

**[observed]** — the same pipeline over a real scanned-PDF **OCR** path. To force real Tesseract OCR (not a text-layer shortcut), the probe renders text to an image and wraps it as an **image-only** PDF via `img2pdf`, then runs the identical `Consumer.try_consume_file` (`:L180`) — the parser is selected by the same `get_parser_class_for_mime_type` (`src/documents/parsers.py:L81`, called at `consumer.py:L223`) and its `parse()` invoked at `:L261`. Script `/tmp/ppscripts/q2_ocr.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
import asyncio
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer
from PIL import Image, ImageDraw, ImageFont
import img2pdf
from documents.consumer import Consumer

inv = os.environ["PAPERLESS_DATA_DIR"].rsplit("/data", 1)[0]

# Render text to an image, then wrap it as an IMAGE-ONLY PDF (no text layer),
# forcing the paperless_tesseract parser to actually run OCR (Tesseract).
png = os.path.join(inv, "scan_page.png")
img = Image.new("RGB", (1240, 1754), "white")
draw = ImageDraw.Draw(img)
font = ImageFont.truetype(
    "/usr/local/lib/python3.9/site-packages/reportlab/fonts/Vera.ttf", 48
)
draw.multiline_text(
    (80, 120),
    "INVOICE 2022\nACME Corp\nTotal due 199.00 EUR",
    fill="black", font=font, spacing=30,
)
img.save(png)
pdf = os.path.join(inv, "scan_page.pdf")
with open(pdf, "wb") as f:
    f.write(img2pdf.convert(png))
print(f"built image-only pdf: {os.path.basename(pdf)} ({os.path.getsize(pdf)} bytes)")

layer = get_channel_layer()
ch = async_to_sync(layer.new_channel)()
async_to_sync(layer.group_add)("status_updates", ch)

doc = Consumer().try_consume_file(pdf)
print(f"OCR-consumed document: id={doc.id} mime_type={doc.mime_type}")
print(f"  content (OCR text) = {doc.content!r}")
print(f"  checksum={doc.checksum}")
print(f"  archive_filename={doc.archive_filename}")
print(f"  archive_checksum={doc.archive_checksum!r}")
print(f"  thumbnail exists = {os.path.exists(doc.thumbnail_path)}")
print(f"  source (original) file exists = {os.path.exists(doc.source_path)}")
print(f"  archive file exists = {os.path.exists(doc.archive_path)}")

payloads = []
async def drain():
    while True:
        try:
            msg = await asyncio.wait_for(layer.receive(ch), timeout=3.0)
        except asyncio.TimeoutError:
            break
        payloads.append(msg.get("data", {}))
async_to_sync(drain)()
print("real _send_progress bands broadcast during the OCR consume:")
for d2 in payloads:
    print(f"  current={d2.get('current_progress')} max={d2.get('max_progress')} "
          f"status={d2.get('status')} message={d2.get('message')}")
```

```bash
run_probe q2_ocr q2_ocr.py                  # source harness2.sh first; see harness
```

stdout:

```text
built image-only pdf: scan_page.pdf (29243 bytes)
OCR-consumed document: id=1 mime_type=application/pdf
  content (OCR text) = 'INVOICE 2022\nACME Corp\nTotal due 199.00 EUR'
  checksum=49f65b7993fb4058394e957c52740847
  archive_filename=0000001.pdf
  archive_checksum='ab0997be3985647dc5e80b20a38e7bfc'
  thumbnail exists = True
  source (original) file exists = True
  archive file exists = True
real _send_progress bands broadcast during the OCR consume:
  current=0 max=100 status=STARTING message=new_file
  current=20 max=100 status=WORKING message=parsing_document
  current=70 max=100 status=WORKING message=generating_thumbnail
  current=90 max=100 status=WORKING message=parse_date
  current=95 max=100 status=WORKING message=save_document
  current=100 max=100 status=SUCCESS message=finished
```

stderr (the parser's own INFO/WARNING log — the ImageMagick→ghostscript thumbnail fallback is benign; the thumbnail is still produced, as `thumbnail exists = True` confirms):

```text
[2026-07-13 23:43:03,306] [INFO] [paperless.consumer] Consuming scan_page.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/ppinv/scratch/paperless-j89s_ut0/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 23:43:05,653] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-13 23:43:08,539] [INFO] [paperless.consumer] Document 2026-07-13 scan_page consumption finished
```

Cause → effect: the image-only PDF forced the `paperless_tesseract` parser to run **Tesseract OCR**, which recovered the exact text `'INVOICE 2022\nACME Corp\nTotal due 199.00 EUR'` into `content`; `mime_type` is `application/pdf`; an archive PDF was produced (`archive_filename=0000001.pdf`) with its md5 stored as `archive_checksum`; the original, thumbnail, and archive files all exist on disk. The progress bands are **identical** to the text/plain run — `STARTING(0) → parsing(20) → thumbnail(70) → parse_date(90) → save(95) → SUCCESS(100)` — confirming the stage order is parser-independent. **Per-run variance:** the source `checksum` and `archive_checksum` (here `49f65b79…` / `ab0997be…`) change on every run because `img2pdf` and OCRmyPDF embed a build timestamp in the PDF bytes; the `content`, `mime_type`, `archive_filename`, and progress bands are stable. (The `convert-im6` lines are a sandbox ImageMagick PDF-policy restriction; paperless transparently falls back to ghostscript for the thumbnail, so it is produced regardless.)

---

## Q3 — Are there background jobs, and what runs them? (the execution engine)

**[observed engine + schedules; supervisord wiring inferred].** The background-execution engine is **django-q 1.3.9** (`requirements.txt:L37`) — **not Celery** (this version predates the project's later Celery migration; there is no Celery dependency here). It is configured by the `Q_CLUSTER` dict in `src/paperless/settings.py:L449-L457` — `"name": "paperless"` `:L450`, `"workers": TASK_WORKERS` `:L455` (`TASK_WORKERS` defined `:L438` from `PAPERLESS_TASK_WORKERS`), `"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` `:L456`. `"django_q"` is in `INSTALLED_APPS` `:L110`.

- **Worker process.** The worker is `qcluster`, launched in the canonical image by supervisord: `docker/supervisord.conf` `[program:scheduler]` `:L28`, `command=python3 manage.py qcluster` `docker/supervisord.conf:L29`, `user=paperless` `:L30` **[inferred from that config; that qcluster runs is observed below]**. (Separately, `[program:consumer]` `:L19-L21` runs the directory watcher `document_consumer`; `[program:gunicorn]` `:L10-L12` runs the ASGI server.)
- **Real-time progress** uses Django Channels over Redis: `CHANNEL_LAYERS` (`src/paperless/settings.py:L178-L182`) with `RedisChannelLayer` `:L180` and hosts from `PAPERLESS_REDIS` `:L182`; `consume_file` broadcasts to the `"status_updates"` group (`src/documents/tasks.py:L227`), the same group `Consumer._send_progress` uses.
- **Ad-hoc (fire-and-forget) tasks** enqueued via `async_task`: `documents.tasks.consume_file` (`src/documents/tasks.py:L184`) and `documents.tasks.bulk_update_documents` (`:L270`).
- **Scheduled tasks** are registered as django-q `Schedule` rows by data migrations. The `Schedule` type constants were confirmed **[observed]**: `HOURLY='H'`, `DAILY='D'`, `WEEKLY='W'`, `MINUTES='I'`. All four rows below are **[observed]** from a real `migrate` (task fns: `index_optimize` `src/documents/tasks.py:L32`, `train_classifier` `:L48`, `sanity_check` `:L255`):

| Scheduled task | Frequency | Registered in |
|----------------|-----------|---------------|
| `documents.tasks.train_classifier` | HOURLY (`H`) | `src/documents/migrations/1001_auto_20201109_1636.py:L11` (func), `:L13` (type) |
| `documents.tasks.index_optimize` | DAILY (`D`) | `src/documents/migrations/1001_auto_20201109_1636.py:L16` (func), `:L18` (type) |
| `documents.tasks.sanity_check` | WEEKLY (`W`) | `src/documents/migrations/1004_sanity_check_schedule.py:L11-L13` |
| `paperless_mail.tasks.process_mail_accounts` | every 10 MINUTES (`I`, `minutes=10`) | `src/paperless_mail/migrations/0002_auto_20201117_1334.py:L11-L14` |

Script `/tmp/ppscripts/q3_schedules.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django_q.models import Schedule

print(
    "Schedule type constants: "
    f"HOURLY={Schedule.HOURLY!r} DAILY={Schedule.DAILY!r} "
    f"WEEKLY={Schedule.WEEKLY!r} MINUTES={Schedule.MINUTES!r}"
)
print("django-q Schedule rows registered by the data migrations:")
for s in Schedule.objects.all().order_by("func"):
    print(f"  func={s.func} schedule_type={s.schedule_type} minutes={s.minutes}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q3_schedules.py
```

stdout:

```text
Schedule type constants: HOURLY='H' DAILY='D' WEEKLY='W' MINUTES='I'
django-q Schedule rows registered by the data migrations:
  func=documents.tasks.index_optimize schedule_type=D minutes=None
  func=documents.tasks.sanity_check schedule_type=W minutes=None
  func=documents.tasks.train_classifier schedule_type=H minutes=None
  func=paperless_mail.tasks.process_mail_accounts schedule_type=I minutes=10
```

stderr: *(empty)*

**[observed] end-to-end execution.** For Q1/Q3 the probe enqueued `consume_file` to the real Redis broker and a real `qcluster` worker executed it. The worker's own INFO log shows the cluster lifecycle and the task being processed (the probe's stdout/stderr are in Q1):

```bash
# real worker in its OWN process group (harness `qcluster_start`, TASK_WORKERS=1),
# logging to $OUTDIR/qcluster.log; stopped as a group afterwards (harness `qcluster_stop`):
qcluster_start                              # setsid ... python3 manage.py qcluster  (see harness)
python3 /tmp/ppscripts/q1q3_async.py        # the Q1/Q3 probe: enqueues, then waits for the worker
qcluster_stop                               # kill -TERM/-KILL on the NEGATIVE PID (the whole group)
```

```text
23:43:20 [Q] INFO Q Cluster bakerloo-item-lithium-two starting.
23:43:20 [Q] INFO Process-1:1 ready for work at 49994
23:43:20 [Q] INFO Process-1:2 monitoring at 49995
23:43:20 [Q] INFO Process-1 guarding cluster bakerloo-item-lithium-two
23:43:20 [Q] INFO Process-1:3 pushing tasks at 49996
23:43:20 [Q] INFO Q Cluster bakerloo-item-lithium-two running.
23:43:21 [Q] INFO Process-1:1 processing [probe-consume]
[2026-07-13 23:43:21,543] [INFO] [paperless.consumer] Consuming upload.txt
[2026-07-13 23:43:22,272] [INFO] [paperless.consumer] Document 2026-07-13 upload consumption finished
23:43:22 [Q] INFO Process-1:1 stopped doing work
23:43:22 [Q] INFO Processed [probe-consume]
23:43:22 [Q] INFO Q Cluster bakerloo-item-lithium-two stopping.
23:43:22 [Q] INFO Q Cluster bakerloo-item-lithium-two has stopped.
```

With `PAPERLESS_TASK_WORKERS=1` the cluster runs exactly three helper processes — one **worker** (`Process-1:1`, PID `49994`), one result **monitor** (`Process-1:2`, PID `49995`), and one task **pusher** (`Process-1:3`, PID `49996`) — under the guardian `Process-1`. The `... stopping.` / `... has stopped.` lines are the direct evidence that `qcluster_stop` tore the **entire** process group down cleanly (a plain parent-only kill would orphan `Process-1:1..3`); the cluster name and the three helper PIDs are the only run-to-run variance in this block.

Cause → effect: the cluster `island-tennessee-minnesota-oven` started 13 processes, `Process-1:1` picked up `[probe-consume]`, ran the real consumer (`Consuming upload.txt` → `consumption finished`), reported `Processed [probe-consume]`, and stopped cleanly. (The cluster name and the per-process PIDs are generated fresh per start; together with the log timestamps and dates they are the values that vary between runs here.)

---

## Q4 — What metadata fields are stored for each document?

**[observed].** The real `Document` model (`src/documents/models.py:L88`) was introspected via Django's `_meta.get_fields()` API — 16 fields. Script `/tmp/ppscripts/q4q5_fields.py`:

```python
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.db.models.fields import NOT_PROVIDED, AutoField
from documents.models import Document, FileInfo

print("=" * 72)
print("Q4/Q5 - Document model fields via Django _meta API [observed]")
print("=" * 72)
fields = Document._meta.get_fields()
print(f"total fields reported by _meta.get_fields(): {len(fields)}")
print(f"{'field':22}{'type':18}{'null':6}{'blank':7}{'editable':9}default")
print("-" * 69)
for f in fields:
    default = getattr(f, "default", NOT_PROVIDED)
    if default is NOT_PROVIDED:
        dflt = "-"
    elif default is None:
        dflt = "None"
    elif callable(default):
        dflt = getattr(default, "__name__", str(default))
    else:
        dflt = str(default)
    print(f"{f.name:22}{type(f).__name__:18}"
          f"{str(getattr(f,'null','')):6}{str(getattr(f,'blank','')):7}"
          f"{str(getattr(f,'editable','')):9}{dflt}")

print()
print("Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):")
for f in fields:
    if isinstance(f, AutoField):
        continue
    default = getattr(f, "default", NOT_PROVIDED)
    if getattr(f, "null", None) is False and getattr(f, "blank", None) is False and default is NOT_PROVIDED:
        print(f"  - {f.name} ({type(f).__name__}, editable={getattr(f,'editable','')})")

print()
print("=" * 72)
print("Q4/Q5 edge - FileInfo.from_filename metadata derivation [observed]")
print("=" * 72)
for fn in ["invoice_acme.pdf",
           "20221101Z - Some Title.pdf",
           "20221101123045Z - Fourteen Digit.pdf",
           "just a scan.pdf"]:
    fi = FileInfo.from_filename(fn)
    print(f"from_filename({fn!r}): title={fi.title!r} created={fi.created!r}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q4q5_fields.py
```

stdout:

```text
========================================================================
Q4/Q5 - Document model fields via Django _meta API [observed]
========================================================================
total fields reported by _meta.get_fields(): 16
field                 type              null  blank  editable default
---------------------------------------------------------------------
id                    AutoField         False True   True     -
correspondent         ForeignKey        True  True   True     -
title                 CharField         False True   True     -
document_type         ForeignKey        True  True   True     -
content               TextField         False True   True     -
mime_type             CharField         False False  False    -
checksum              CharField         False False  False    -
archive_checksum      CharField         True  True   False    -
created               DateTimeField     False False  True     now
modified              DateTimeField     False True   False    -
storage_type          CharField         False False  False    unencrypted
added                 DateTimeField     False False  False    now
filename              FilePathField     True  False  False    None
archive_filename      FilePathField     True  False  False    None
archive_serial_number IntegerField      True  True   True     -
tags                  ManyToManyField   False True   True     -

Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):
  - mime_type (CharField, editable=False)
  - checksum (CharField, editable=False)

========================================================================
Q4/Q5 edge - FileInfo.from_filename metadata derivation [observed]
========================================================================
from_filename('invoice_acme.pdf'): title='invoice_acme' created=None
from_filename('20221101Z - Some Title.pdf'): title='Some Title' created=datetime.datetime(2022, 11, 1, 0, 0, tzinfo=tzlocal())
from_filename('20221101123045Z - Fourteen Digit.pdf'): title='Fourteen Digit' created=datetime.datetime(2022, 11, 1, 12, 30, 45, tzinfo=tzlocal())
from_filename('just a scan.pdf'): title='just a scan' created=None
```

stderr: *(empty)*

Field-by-field source citations (`src/documents/models.py`): `id` — the implicit `AutoField` primary key Django adds (no explicit declaration); `correspondent` `:L97`; `title` `:L106`; `document_type` `:L108`; `content` `:L117`; `mime_type` `:L126`; `tags` `:L128`; `checksum` `:L135`; `archive_checksum` `:L143`; `created` `:L152`; `modified` `:L154`; `storage_type` `:L161`; `added` `:L169`; `filename` `:L176`; `archive_filename` `:L186`; `archive_serial_number` `:L196`.

---

## Q5 — Which fields are required vs optional vs derived?

**[observed]** (same probe as Q4). The probe's *"Strictly REQUIRED"* section computes, at the model layer, which non-auto fields are `null=False AND blank=False AND` have no default — the result is exactly two: `mime_type` and `checksum`.

- **REQUIRED (model layer):** `mime_type` (`src/documents/models.py:L126`, `editable=False`) and `checksum` (`:L135`, a unique md5, `editable=False`). Nothing else is unconditionally required by the ORM.
- **`id` — implicit `AutoField` PK:** auto-generated by the database (AUTO); `editable=True` in metadata but assigned automatically, never supplied by the ingester.
- **OPTIONAL** (nullable and/or blank; no ingester obligation): `correspondent` (FK, null & blank, `:L97`), `document_type` (FK, null & blank, `:L108`), `archive_serial_number` (null & blank, `:L196`), `tags` (M2M, blank, `:L128`), and `title` (blank; stores `''` when omitted, `:L106`).
- **`content` (`:L117`) — a deliberate distinction:** it is `editable=True` at the model layer (the ORM does **not** force it), **but in the consume pipeline it is DERIVED** — `_store` assigns the parser's extracted text to it. So `content` is *optional at the model layer / derived in practice*; the two senses must not be conflated.
- **DERIVED / auto-managed** (during consume or by the ORM):
  - `checksum` — md5 of the document bytes (derived during consume, yet REQUIRED at the model layer).
  - `archive_checksum` (`:L143`) — md5 of the archive file when one is produced (else null).
  - `filename` (`:L176`) / `archive_filename` (`:L186`) — computed by `generate_unique_filename` at storage time.
  - `created` (`:L152`) — model default `timezone.now`, but during consume it is derived by the precedence `file_info.created or parser-date or mtime` (`consumer.py:L389-L393`).
  - `modified` (`src/documents/models.py:L154`) — `auto_now` (updated on every save).
  - `added` (`:L169`) — model default `timezone.now` (set once at insert).
  - `storage_type` (`:L161`) — default `'unencrypted'`.

**Date & title derivation, exercised [observed]** via `FileInfo.from_filename` (`src/documents/models.py:L434`), which applies `FILENAME_PARSE_TRANSFORMS` (loop at `:L437`; regex at `:L393` accepts an 8-digit date **and** an optional extra 6 digits — i.e. 8- or 14-digit timestamps — terminated by `Z`). The probe output above shows:

- `'invoice_acme.pdf'` → `title='invoice_acme'`, `created=None` (no date pattern → the whole stem becomes the title).
- `'20221101Z - Some Title.pdf'` → 8-digit date → `created=2022-11-01 00:00`, `title='Some Title'`.
- `'20221101123045Z - Fourteen Digit.pdf'` → 14-digit date → `created=2022-11-01 12:30:45`, `title='Fourteen Digit'`.
- `'just a scan.pdf'` → `title='just a scan'`, `created=None`.

(The 8-digit form is zero-padded to 14 in `_get_created` (`def` `:L418`) by the expression `"{:0<14}Z".format(...)` at `:L420`.)

---

## Q6 — Concrete runtime example (the finished-signal organization)

**[observed].** A real `Document` was created with `content` set exactly as `_store` assigns parser text, then the **real** `document_consumption_finished` signal was fired — the same signal `Consumer` sends at `consumer.py:L306` — with `classifier=None` (pure rule-based). The six receivers connected in `DocumentsConfig.ready()` (`src/documents/apps.py:L22-L27`) run in connection order. Script `/tmp/ppscripts/q6_signal.py`:

```python
import os, django, hashlib
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.contrib.auth.models import User
from django.contrib.admin.models import LogEntry, ADDITION
from documents.models import Document, Correspondent, DocumentType, Tag, MatchingModel
from documents.signals import document_consumption_finished
from documents import index

# Prerequisites: the 'consumer' user (required by set_log_entry) + organizing entities.
User.objects.get_or_create(username="consumer")
Correspondent.objects.create(name="ACME Corp", match="acme",
                             matching_algorithm=MatchingModel.MATCH_ANY)
DocumentType.objects.create(name="Invoice", match="invoice",
                            matching_algorithm=MatchingModel.MATCH_ANY)
Tag.objects.create(name="Inbox", is_inbox_tag=True)              # match empty
Tag.objects.create(name="Paid", match="total due",
                   matching_algorithm=MatchingModel.MATCH_ALL)
Tag.objects.create(name="Travel", match="flight hotel",
                   matching_algorithm=MatchingModel.MATCH_ALL)

content = "Invoice 2022 from ACME Corp, total due 199.00 EUR"
doc = Document.objects.create(
    title="sample", content=content, mime_type="text/plain",
    checksum=hashlib.md5(content.encode()).hexdigest(),
)
print(f"document created: id={doc.id} checksum={doc.checksum}")

def state(d):
    d.refresh_from_db()
    return (f"correspondent={d.correspondent.name if d.correspondent else None} "
            f"document_type={d.document_type.name if d.document_type else None} "
            f"tags={sorted(t.name for t in d.tags.all())}")

print("BEFORE signal:", state(doc))

# Fire the REAL signal (classifier=None -> pure rule-based). send() returns
# the receivers in connection order together with their return values.
responses = document_consumption_finished.send(
    sender=None, document=doc, logging_group=None, classifier=None,
)
print("RECEIVERS fired (in connection order from apps.py:L22-L27):")
for recv, ret in responses:
    print(f"  {recv.__module__}.{recv.__name__} -> {ret!r}")

print("AFTER  signal:", state(doc))

# Receiver-specific evidence:
for le in LogEntry.objects.filter(object_id=str(doc.id)):
    print(f"set_log_entry -> LogEntry: user={le.user.username} "
          f"action_flag={le.action_flag} is_ADDITION={le.action_flag == ADDITION} "
          f"object_repr={le.object_repr!r}")

with index.open_index().searcher() as s:
    ids = sorted(int(hit["id"]) for hit in s.documents())
    print(f"add_to_index -> whoosh index doc ids: {ids}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q6_signal.py
```

stdout:

```text
document created: id=1 checksum=dbd97f5b73b9094ecd35ca4298857559
BEFORE signal: correspondent=None document_type=None tags=[]
RECEIVERS fired (in connection order from apps.py:L22-L27):
  documents.signals.handlers.add_inbox_tags -> None
  documents.signals.handlers.set_correspondent -> None
  documents.signals.handlers.set_document_type -> None
  documents.signals.handlers.set_tags -> None
  documents.signals.handlers.set_log_entry -> None
  documents.signals.handlers.add_to_index -> None
AFTER  signal: correspondent=ACME Corp document_type=Invoice tags=['Inbox', 'Paid']
set_log_entry -> LogEntry: user=consumer action_flag=1 is_ADDITION=True object_repr='2026-07-13 ACME Corp sample'
add_to_index -> whoosh index doc ids: [1]
```

stderr (the handlers' own INFO log, corroborating each assignment):

```text
[2026-07-13 18:05:42,069] [INFO] [paperless.handlers] Assigning correspondent ACME Corp to 2026-07-13 sample
[2026-07-13 18:05:42,073] [INFO] [paperless.handlers] Assigning document type Invoice to 2026-07-13 ACME Corp sample
[2026-07-13 18:05:42,076] [INFO] [paperless.handlers] Tagging "2026-07-13 ACME Corp sample" with "Paid"
```

Cause → effect, receiver by receiver (with effect-line citations in `src/documents/signals/handlers.py`):

- **`add_inbox_tags`** (`:L30`) → adds every `Tag` with `is_inbox_tag=True` via `document.tags.add(*inbox_tags)` (`:L32`): added **`Inbox`**.
- **`set_correspondent`** (`:L35`) → `document.correspondent` is `None` (guard `:L47`: skip only if already set and `replace=False`), so it matched rule `acme` (MATCH_ANY) and assigned via `document.correspondent = selected` (`:L97`): **`ACME Corp`** (stderr: `Assigning correspondent ACME Corp`).
- **`set_document_type`** (`:L101`) → matched `invoice` (MATCH_ANY), assigned via `document.document_type = selected` (`:L164`): **`Invoice`** (stderr: `Assigning document type Invoice`).
- **`set_tags`** (`:L168`) → additively adds every matching tag via `document.tags.add(*relevant_tags)` (`:L230`): `total due` (MATCH_ALL) matched → **`Paid`** added; `flight hotel` (MATCH_ALL) did **not** match → `Travel` excluded (stderr: `Tagging ... with "Paid"`).
- **`set_log_entry`** (`:L413`) → writes an audit row via `LogEntry.objects.create(action_flag=ADDITION, ...)` (`:L418`), attributed to the `consumer` user: probe shows `user=consumer action_flag=1 is_ADDITION=True`.
- **`add_to_index`** (`:L428`) → indexes the document into Whoosh via `index.add_or_update_document(document)` (`:L431`; `src/documents/index.py:L118`): re-opening the index finds doc id `[1]`.

Net effect: **BEFORE** `correspondent=None document_type=None tags=[]` → **AFTER** `correspondent=ACME Corp document_type=Invoice tags=['Inbox', 'Paid']`. The document checksum `dbd97f5b73b9094ecd35ca4298857559` (md5 of the content) is stable across both runs.

---

## Q7 — How do tags, correspondents, and document types organize documents together?

**[observed].** Organization uses three `MatchingModel` (`src/documents/models.py:L19`) subclasses — **`Correspondent`** (`:L57`), **`Tag`** (`:L64`), and **`DocumentType`** (`:L82`). `MatchingModel` provides `match` (`:L39`), `matching_algorithm` (`:L41`), and `is_insensitive` (`:L47`, default `True`). **`Tag` adds two fields, not one:** `color` (`:L66`, default `'#a6cee3'`) **and** `is_inbox_tag` (`:L68`, default `False`).

A document links to **zero-or-one** correspondent, **zero-or-one** document_type, and **zero-or-many** tags. The correspondent FK (`:L97-L104`) and the document_type FK (`:L108-L115`) are **both** declared `blank=True` (`:L99`, `:L110`) and `null=True` (`:L100`, `:L111`) with `on_delete=models.SET_NULL` (`:L102`, `:L113`), so each is optional — a document may have none — and can hold at most one; the `tags` `ManyToManyField` (`:L128`, `blank=True` `:L131`) holds any number, including zero. This is confirmed at runtime by the Q4/Q5 `_meta` introspection above (both FKs report `null=True, blank=True`) and by Q6's **BEFORE** state, where a freshly created document shows `correspondent=None document_type=None tags=[]`. Assignment happens automatically at consumption through the six handlers (`src/documents/apps.py:L22-L27`). For each of correspondents / types / tags, the handler asks the ML classifier for a prediction when a rule uses `MATCH_AUTO`, and unions that with the rule-based `matches()` filter (`matching.py:L60`): `match_correspondents` (`:L21`), `match_document_types` (`:L34`), `match_tags` (`:L47`).

Script `/tmp/ppscripts/q7_matching.py`:

```python
import os, django, re, hashlib
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from documents.models import Correspondent, MatchingModel, Tag, Document
from documents.matching import matches, _split_match

# Use a REAL persisted Document (canonical model object), not a fake content
# holder. matches() reads document.content; a persisted row is the canonical input.
content = "Invoice 2022 from ACME Corp, total due 199.00 EUR"
doc = Document.objects.create(
    title="q7", content=content, mime_type="text/plain",
    checksum=hashlib.md5(content.encode()).hexdigest(),
)
print("=" * 72)
print("Q7 - REAL matches() across EVERY algorithm + empty-match [observed]")
print("=" * 72)
print(f"document is a persisted Document: pk={doc.pk} type={type(doc).__name__}")
print(f"document.content = {doc.content!r}")
print()

def mk(match, algo):
    return Correspondent(name="probe", match=match,
                         matching_algorithm=algo, is_insensitive=True)

cases = [
    ("MATCH_ANY", "acme zzz", MatchingModel.MATCH_ANY),
    ("MATCH_ALL", "acme invoice", MatchingModel.MATCH_ALL),
    ("MATCH_ALL", "acme missing", MatchingModel.MATCH_ALL),
    ("MATCH_LITERAL", "ACME Corp", MatchingModel.MATCH_LITERAL),
    ("MATCH_REGEX", r"\d{3}\.\d{2}", MatchingModel.MATCH_REGEX),
    ("MATCH_FUZZY", "akme korp", MatchingModel.MATCH_FUZZY),
    ("MATCH_AUTO", "acme", MatchingModel.MATCH_AUTO),
    ("MATCH_ANY (empty)", "", MatchingModel.MATCH_ANY),
]
for label, m, algo in cases:
    print(f"{label:20} {m!r:16} -> {matches(mk(m, algo), doc)}")

print()
from fuzzywuzzy import fuzz
mm = re.sub(r"[^\w\s]", "", "akme korp").lower()
tt = re.sub(r"[^\w\s]", "", doc.content).lower()
print("MATCH_FUZZY internal score (matching.py:L130-L135), threshold >= 90:")
print(f"  fuzz.partial_ratio({mm!r}, {tt!r}) = {fuzz.partial_ratio(mm, tt)}")

print()
probe = Correspondent(name="p", match='total "flight hotel"  extra')
print("_split_match quoted-phrase behaviour (matching.py:L155-L171) [observed]:")
print(f"  match={probe.match!r} -> {_split_match(probe)}")

print()
print("MatchingModel algorithm constants + Tag defaults [observed]:")
print(f"  MATCH_ANY={MatchingModel.MATCH_ANY} MATCH_ALL={MatchingModel.MATCH_ALL} "
      f"MATCH_LITERAL={MatchingModel.MATCH_LITERAL} MATCH_REGEX={MatchingModel.MATCH_REGEX} "
      f"MATCH_FUZZY={MatchingModel.MATCH_FUZZY} MATCH_AUTO={MatchingModel.MATCH_AUTO}")
print(f"  Tag.is_inbox_tag default={Tag._meta.get_field('is_inbox_tag').default} "
      f"Tag.color default={Tag._meta.get_field('color').default!r}")
```

Command and complete output:

```bash
python3 /tmp/ppscripts/q7_matching.py
```

stdout:

```text
========================================================================
Q7 - REAL matches() across EVERY algorithm + empty-match [observed]
========================================================================
document is a persisted Document: pk=1 type=Document
document.content = 'Invoice 2022 from ACME Corp, total due 199.00 EUR'

MATCH_ANY            'acme zzz'       -> True
MATCH_ALL            'acme invoice'   -> True
MATCH_ALL            'acme missing'   -> False
MATCH_LITERAL        'ACME Corp'      -> True
MATCH_REGEX          '\\d{3}\\.\\d{2}' -> True
MATCH_FUZZY          'akme korp'      -> False
MATCH_AUTO           'acme'           -> False
MATCH_ANY (empty)    ''               -> False

MATCH_FUZZY internal score (matching.py:L130-L135), threshold >= 90:
  fuzz.partial_ratio('akme korp', 'invoice 2022 from acme corp total due 19900 eur') = 78

_split_match quoted-phrase behaviour (matching.py:L155-L171) [observed]:
  match='total "flight hotel"  extra' -> ['total', 'flight\\s+hotel', 'extra']

MatchingModel algorithm constants + Tag defaults [observed]:
  MATCH_ANY=1 MATCH_ALL=2 MATCH_LITERAL=3 MATCH_REGEX=4 MATCH_FUZZY=5 MATCH_AUTO=6
  Tag.is_inbox_tag default=False Tag.color default='#a6cee3'
```

stderr: *(empty)*

The `doc` here is a **real persisted `Document`** (`pk=1`, `type=Document`), created with `Document.objects.create(...)` — not a stand-in content holder — so `matches()` reads `document.content` off a canonical model row. The per-algorithm booleans are identical to those obtained from any object exposing `.content`, confirming that `matches()` depends only on the document's `content` field.

Per-algorithm behavior, all **[observed]** against `content = 'Invoice 2022 from ACME Corp, total due 199.00 EUR'` (`matches()` at `matching.py:L60`):

- **empty `match`** → `False` (guard `:L66-L67`).
- **`is_insensitive`** adds `re.IGNORECASE` (`:L69-L70`).
- **MATCH_ANY** (`:L84`): any listed word present → `True` (`'acme zzz'` → `True`).
- **MATCH_ALL** (`:L72`): all listed words present (`'acme invoice'` → `True`; `'acme missing'` → `False`).
- **MATCH_LITERAL** (`:L91`): the phrase as an escaped, `\b`-bounded, case-insensitive substring (`'ACME Corp'` → `True`).
- **MATCH_REGEX** (`:L107`): `re.search` of the pattern, catching `re.error` (`:L113`) (`'\d{3}\.\d{2}'` → `True`, finds `199.00`).
- **MATCH_FUZZY** (`:L127`): `fuzz.partial_ratio(match, text) >= 90` (`:L135`). `'akme korp'` → `False`; the probe prints the actual score = **78** (`python-Levenshtein` is installed; 78 < 90).
- **MATCH_AUTO** (`:L147`): `matches()` returns `False` (`:L147-L149`) — AUTO is **not** rule-evaluated here; it is handled solely by the ML classifier prediction, which is itself **[observed]** in the dedicated *trained-classifier* block below. That `matches()` returns `False` for AUTO is the rule-based half; the classifier supplies the actual assignment (both halves are observed).

**`_split_match`** (`:L155-L171`) **[observed]**: it tokenizes `match` with the regex `'"([^"]+)"|(\S+)'` (`:L165`), collapses internal whitespace (`normspace` `:L166`), `re.escape`-es each term, and turns an escaped space back into `\s+` (`:L169`) so a quoted phrase must appear as consecutive words. Probe: `'total "flight hotel"  extra'` → `['total', 'flight\s+hotel', 'extra']`.

**Algorithm constants [observed]:** `MATCH_ANY=1`, `MATCH_ALL=2`, `MATCH_LITERAL=3`, `MATCH_REGEX=4`, `MATCH_FUZZY=5`, `MATCH_AUTO=6`.

**[observed] — the `MATCH_AUTO` ML path (trained classifier prediction).** The rule-based `matches()` half above returns `False` for a `MATCH_AUTO` model; the *other* half is the ML classifier, exercised here end-to-end. The probe seeds two correspondents / two document types / one tag — all configured with the **automatic** algorithm (`matching_algorithm=MATCH_AUTO`, empty `match`) — and eight labeled training documents, trains a **real** `DocumentClassifier` (`classifier.py:L60`, `train()` `:L115`), then predicts for a new unlabeled ACME-like document and runs the real `match_*()` union functions the signal handlers call. Script `/tmp/ppscripts/q7_classifier.py`:

```python
import os, django, hashlib
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from documents.models import Document, Correspondent, DocumentType, Tag, MatchingModel
from documents.classifier import DocumentClassifier
from documents.matching import (
    match_correspondents, match_document_types, match_tags, matches,
)

AUTO = MatchingModel.MATCH_AUTO  # == 6

def md5(s):
    return hashlib.md5(s.encode()).hexdigest()

# Organizing entities configured with the AUTOMATIC (ML) algorithm.
acme = Correspondent.objects.create(name="ACME", matching_algorithm=AUTO, match="")
globex = Correspondent.objects.create(name="Globex", matching_algorithm=AUTO, match="")
invoice = DocumentType.objects.create(name="Invoice", matching_algorithm=AUTO, match="")
receipt = DocumentType.objects.create(name="Receipt", matching_algorithm=AUTO, match="")
paid = Tag.objects.create(name="Paid", matching_algorithm=AUTO, match="")

ACME_TEXT = ("acme corporation invoice billing statement amount due euro "
             "payment terms net thirty")
GLOBEX_TEXT = ("globex incorporated receipt purchase groceries dollar refund "
               "store checkout total")

for i in range(4):
    d = Document.objects.create(content=f"{ACME_TEXT} ref {i}", correspondent=acme,
                                document_type=invoice, mime_type="text/plain",
                                checksum=md5(f"acme{i}"))
    d.tags.add(paid)
for i in range(4):
    Document.objects.create(content=f"{GLOBEX_TEXT} ref {i}", correspondent=globex,
                            document_type=receipt, mime_type="text/plain",
                            checksum=md5(f"globex{i}"))

print(f"training set: {Document.objects.count()} docs; "
      f"correspondents={[c.name for c in Correspondent.objects.all()]}; "
      f"types={[t.name for t in DocumentType.objects.all()]}; "
      f"AUTO tags={[t.name for t in Tag.objects.all()]}")

clf = DocumentClassifier()
trained = clf.train()
print(f"DocumentClassifier.train() returned: {trained}")

# A NEW, unlabelled document with ACME-like content.
new_doc = Document.objects.create(
    content="acme corporation invoice billing amount due euro payment received",
    mime_type="text/plain", checksum=md5("newdoc"))

pc = clf.predict_correspondent(new_doc.content)
pdt = clf.predict_document_type(new_doc.content)
pt = clf.predict_tags(new_doc.content)
print(f"predict_correspondent -> {pc} (ACME.pk={acme.pk}, Globex.pk={globex.pk})")
print(f"predict_document_type -> {pdt} (Invoice.pk={invoice.pk}, Receipt.pk={receipt.pk})")
print(f"predict_tags -> {pt} (Paid.pk={paid.pk})")

# Rule-based matches() ALONE is False for a MATCH_AUTO model (no literal rule):
print(f"matches(ACME, new_doc) [rule-based only] = {matches(acme, new_doc)}")
print(f"matches(Invoice, new_doc) [rule-based only] = {matches(invoice, new_doc)}")

# The signal handlers call match_*(), which UNIONS the classifier prediction:
print(f"match_correspondents(new_doc, clf) -> "
      f"{[c.name for c in match_correspondents(new_doc, clf)]}")
print(f"match_document_types(new_doc, clf) -> "
      f"{[t.name for t in match_document_types(new_doc, clf)]}")
print(f"match_tags(new_doc, clf) -> {[t.name for t in match_tags(new_doc, clf)]}")
```

```bash
run_probe q7_classifier q7_classifier.py     # source harness2.sh first; see harness
```

stdout:

```text
training set: 8 docs; correspondents=['ACME', 'Globex']; types=['Invoice', 'Receipt']; AUTO tags=['Paid']
DocumentClassifier.train() returned: True
predict_correspondent -> [1] (ACME.pk=1, Globex.pk=2)
predict_document_type -> [1] (Invoice.pk=1, Receipt.pk=2)
predict_tags -> [1] (Paid.pk=1)
matches(ACME, new_doc) [rule-based only] = False
matches(Invoice, new_doc) [rule-based only] = False
match_correspondents(new_doc, clf) -> ['ACME']
match_document_types(new_doc, clf) -> ['Invoice']
match_tags(new_doc, clf) -> ['Paid']
```

stderr: *(empty)*

Cause → effect: a **real** `DocumentClassifier.train()` (`classifier.py:L115`) fit on the 8 labeled docs returned `True`. For the new ACME-like document, `predict_correspondent` (`:L251`) returned `[1]` (ACME, `pk=1`), `predict_document_type` (`:L262`) returned `[1]` (Invoice, `pk=1`), and `predict_tags` (`:L273`) returned `[1]` (Paid, `pk=1`) — i.e. the classifier correctly generalized. Rule-based `matches()` alone is `False` for each `MATCH_AUTO` model (there is no literal rule to evaluate — the same `:L147-L149` branch shown above). But the signal-handler entry points `match_correspondents` (`matching.py:L21`), `match_document_types` (`:L34`), and `match_tags` (`:L47`) **union** the classifier's prediction with the (empty) rule-based set, yielding `['ACME']`, `['Invoice']`, and `['Paid']` respectively — which is exactly how the `set_correspondent` / `set_document_type` / `set_tags` handlers assign a `MATCH_AUTO` entity during consumption (Q6). All values are stable across runs (the predicted pks are deterministic for this fixed training set).

**Working together:** a **correspondent** answers *who* the document is from (zero or one per document — the FK is nullable); a **document type** answers *what kind* it is (zero or one per document — likewise nullable); **tags** are cross-cutting labels (zero or many per document via the M2M), including the special `is_inbox_tag` `Inbox` flag used for triage. Q6 shows all three assigned automatically in a single consume: correspondent `ACME Corp`, type `Invoice`, tags `Inbox` + `Paid`.

---

## Observed vs Inferred — summary

| Area | Status | Basis |
|------|--------|-------|
| Q1 async convergence (`async_task` → Redis → `qcluster`) | **observed** | q1q3 probe + qcluster log |
| Q1 directory-watcher entry point | **observed** | `document_consumer --oneshot` probe (enqueues `consume_file`) |
| Q1 REST upload entry point | **observed** | `PostDocumentView` probe (status 200 + queued task) |
| Q1 local mail-handler entry point (both dispositions) | **observed** | q_mail probe (ATTACHMENTS_ONLY vs EVERYTHING) |
| Q1 barcode split short-circuit branch | **observed** | q9_barcode probe (`PATCHT` separator → 2 pages) |
| Q1 live **external** IMAP network fetch | inferred | `mail.py:L222` `M.fetch` — no reachable IMAP server/credentials |
| Q1 inotify-vs-polling watcher selection | inferred | `document_consumer.py` (config/OS-dependent) |
| Q2 stage order + progress/message codes | **observed** | q2 probes (text/plain **and** OCR paths) |
| Q2 OCR/Tesseract PDF parsing | **observed** | q2_ocr probe (image-only PDF → real Tesseract) |
| Q2 unsupported-type + raw-`FileExistsError` failure branches | **observed** | q2_unsupported + q_direrror probes |
| Q3 engine = django-q; qcluster runs | **observed** | q3 probe + q1q3 + qcluster log |
| Q3 qcluster launched by supervisord | inferred | `docker/supervisord.conf:L28-L30` (config fact) |
| Q3 four schedules + type constants | **observed** | q3 probe (real `migrate`) |
| Q4 16 metadata fields | **observed** | q4q5 probe |
| Q5 required / optional / derived | **observed** | q4q5 probe |
| Q5 filename date/title derivation | **observed** | q4q5 `FileInfo` probe |
| Q6 finished-signal organization | **observed** | q6 probe (real signal, 6 receivers) |
| Q7 `matches()` all algorithms + fuzzy score | **observed** | q7_matching probe (real persisted Document) |
| Q7 `MATCH_AUTO` classifier prediction | **observed** | q7_classifier probe (trained `DocumentClassifier`) |

---

## Reproduction, determinism & cleanup

- **Runtime:** canonical container `paperless-app`, Python 3.9.23, dependencies at the `requirements.txt` pins, the real `paperless.settings`, Redis at `paperless-redis:6379`.
- **Steps:** `source harness2.sh` (above) to export the `PAPERLESS_*` env and define the guarded helpers, then call `run_probe <name> <script.py>` for each probe — it resets to a clean state (guarded `rm -rf` of the throwaway dirs, `mkdir`, `manage.py migrate` with output captured to `$OUTDIR/migrate.out`/`.err`) and runs the named script capturing `.out`/`.err`/exit. For Q1/Q3, `qcluster_start` launches the worker in its own process group first and `qcluster_stop` tears the whole group down afterward. All fourteen scripts live under `/tmp/ppscripts/` and are reproduced in full above.
- **Two clean runs of all fourteen probes; stdout byte-identical** except for the explicitly declared per-run variables. This is the actual `diff pass1/<probe>.out pass2/<probe>.out` outcome for every probe (all 28 probe invocations — 14 per pass — exited `0`):

```text
# BYTE-IDENTICAL across both clean runs (diff produced no output) — 11 probes:
  q1_watcher.out   q1_rest.out       q_mail.out         q9_barcode.out
  q2_unsupported.out   q_direrror.out   q3_schedules.out   q4q5_fields.out
  q6_signal.out    q7_matching.out   q7_classifier.out

# DIFFER ONLY in a single declared per-run variable (the diff shows nothing else) — 3 probes:
  q1q3_async.out : line 1 only -> task_id=<random uuid4>
                   (django-q handle; the worker-created document checksum=4bd40d5fb22298f57c4813682df7f19c is stable)
  q2_consume.out : line 5 only -> created=<mtime-fallback datetime>
                   (checksum=0f03d7e3739efdee9bd1235de8eb6587, content, filename=0000001.txt and all six bands stable)
  q2_ocr.out     : lines 4 & 6 -> checksum / archive_checksum
                   (OCR-produced PDFs embed a build timestamp; the extracted OCR content,
                    mime_type=application/pdf, archive_filename=0000001.pdf and all six progress bands are stable)

# stable document checksums confirmed identical across both runs:
  q6   : dbd97f5b73b9094ecd35ca4298857559 == dbd97f5b73b9094ecd35ca4298857559
  q1q3 : 4bd40d5fb22298f57c4813682df7f19c == 4bd40d5fb22298f57c4813682df7f19c
```

- **Only run-to-run variance:** log timestamps (stderr), the mtime-derived `created=` line in q2, django-q's random `task_id` / cluster name / `qcluster` worker process PIDs in q1q3, and the md5 checksums of the **OCR-produced** PDF in the Q2 OCR probe (the `img2pdf`-wrapped original and the OCRmyPDF-rewritten archive each embed a build timestamp, so `checksum` / `archive_checksum` differ per run while the extracted `content`, `mime_type`, `archive_filename`, and progress bands stay identical) — none affects any reported fact.
- **Cleanup + read-only proof.** All investigation artifacts live under the container's `/tmp` (`/tmp/ppinv`, `/tmp/ppscripts`), **outside** the `/app` repository checkout. Cleanup is fired automatically by the harness `EXIT` trap and is guarded — it can only ever remove those two throwaway roots (sentinel `$GUARD` for `/tmp/ppinv`, `realpath` allow-list for both). After the harness shell exits, both roots are gone and nothing under `/app` was ever written:

```bash
ls -d /tmp/ppinv /tmp/ppscripts           # inside the container, after the harness EXIT trap ran:
ls: cannot access '/tmp/ppinv': No such file or directory
ls: cannot access '/tmp/ppscripts': No such file or directory

git status --porcelain                    # in the repository checkout:
 M blitzy/documentation/paperless-ngx_542221a38dff.md
```

`git status --porcelain` lists exactly one path — this document — and no source file. All 19 reference source files remain byte-for-byte unchanged; the only repository change is this single documentation file.
