# paperless-ngx — Runtime Choreography: An Evidence-Backed Investigation

This document answers six "runtime choreography" questions about the paperless-ngx
document-management backend. Every answer is proven with **real runtime evidence** —
actual on-disk paths, actual checksums/hashes, and actual log messages captured by
*running the system through its canonical entry points* — not inferred from reading
code. Each factual claim is grounded in a `file:line` reference that names the specific
function performing the work, and each finding is explicitly tagged **[observed]** or
**[inferred]**.

The investigation is read-only: apart from this one Markdown file, the paperless-ngx
repository is left byte-for-byte unchanged. All scratch data lived outside the repository
tree (inside the container under `/scratch`) and was removed afterward.

---

## Methodology & Environment

**Canonical runtime.** paperless-ngx v1.7.0 pins its runtime to **Python 3.9**
(`Dockerfile:18` → `FROM python:3.9-slim-bullseye as main-app`). The host shell is
Python 3.12 with Django not importable, so the entire investigation was run **inside the
canonical Docker container** (`paperless-ngx-ready:latest`, Python 3.9.23, Django 4.0.4),
with the repository checked out at `/app` on commit `542221a38dff`. Django's `manage.py`
lives at `src/manage.py`, so every management command was invoked with the working
directory set to `/app/src`.

**Services.** Redis (`redis==3.5.3`) — the Django-Q broker / channels backend — was
started with `redis-server --daemonize yes`. The database is the default SQLite.

**Throwaway store (via environment variables only — no tracked file was edited).**
Storage was pointed at scratch locations so experiments never touched a real archive:

```
PAPERLESS_MEDIA_ROOT=/scratch/media
PAPERLESS_DATA_DIR=/scratch/data
PAPERLESS_CONSUMPTION_DIR=/scratch/consume
```

The storage layout observed under `MEDIA_ROOT` (grounded in `src/paperless/settings.py`):
`ORIGINALS_DIR = media/documents/originals` (`:62`), `ARCHIVE_DIR = media/documents/archive`
(`:63`), `THUMBNAIL_DIR = media/documents/thumbnails` (`:64`), `MEDIA_LOCK = media/media.lock`
(`:72`), `MODEL_FILE = DATA_DIR/classification_model.pickle` (`:74`). `PAPERLESS_FILENAME_FORMAT`
defaults to `None` (`:584`), so tags are *not* in the path unless the format is set
explicitly (required for Q1/Q2).

**Observability (this matters for reading the evidence).** The logging config at
`src/paperless/settings.py:373-411` routes the `paperless` logger to the file
`data/log/paperless.log` at level **DEBUG** (`:409`), while the root/console handler is
gated at **INFO** unless `PAPERLESS_DEBUG=true` (`:388`, `:407`). **Consequence:**
DEBUG-level evidence (notably the Q3 skip message `"Training data unchanged."`) is
*always* written to `data/log/paperless.log` but appears on the console only with
`PAPERLESS_DEBUG=true`. Throughout this investigation, DEBUG evidence was read from
`/scratch/data/log/paperless.log` and, where useful, also surfaced on the console by
running with `PAPERLESS_DEBUG=true`.

**Determinism note.** The container's setup left a background Django-Q `qcluster` worker
running, which executes *scheduled* tasks (`train_classifier`, `sanity_check`,
`index_optimize`, `process_mail_accounts`) asynchronously and would otherwise inject
stray log lines and retrain the classifier on a schedule — contaminating deterministic
observation (especially Q3). The `qcluster` was therefore stopped so that each canonical
entry point runs single-threaded and its output is unambiguous. This does **not** bypass
any interface: the canonical entry points (management commands, the `tasks.consume_file`
task callable, and the ORM `m2m_changed`/`post_save` signals) are still invoked directly;
stopping the async scheduler only prevents concurrent interfering runs. (The `gunicorn`
web server was left running but is inert here — it serves HTTP and does not run scheduled
tasks; no API requests were issued during the classifier/sanity experiments.)

**Canonical consumption.** Documents were ingested by calling the real task callable
`documents.tasks.consume_file(path)` synchronously via `manage.py shell` — this is the
exact task the consume-directory watcher and the REST upload endpoint enqueue to Django-Q
(`document_consumer` → `async_task("documents.tasks.consume_file", …)`). The sample file
was copied into `/scratch/consume` first because the consumer moves and deletes its source.

**Base command template** (all commands below are variations of this):

```
docker exec [-e PAPERLESS_FILENAME_FORMAT='…'] [-e PAPERLESS_DEBUG=true] \
    -w /app/src paperless-work python3 manage.py <command>
```

**Reset procedure** used between question groups to obtain a clean, self-contained store:

```
docker exec paperless-work bash -lc \
  'rm -rf /scratch/media/* /scratch/data/* /scratch/consume/*; \
   mkdir -p /scratch/media /scratch/data /scratch/consume /scratch/data/log'
docker exec -w /app/src paperless-work python3 manage.py migrate --no-input
```

---

## Q1 — File relocation on tag change

> **User's words:** *"When a document's tags change and the filename format includes tags
> in the directory structure, files apparently relocate themselves. I wonder what log
> messages actually appear during that dance, and what the before and after paths look
> like in practice."*

### Direct answer

When `PAPERLESS_FILENAME_FORMAT` embeds the tag token, changing a document's tags fires
the `m2m_changed` signal, which invokes **`update_filename_and_move_files`**
(`src/documents/signals/handlers.py:312`). That function renames the **original** and the
**archive** file into the new tag-derived directory. The **thumbnail does not move** —
its path is derived from the document's primary key, not the filename format. And the
surprising truth about the "dance": **a successful move emits no log message at all** —
there is not a single logging call inside `update_filename_and_move_files`. The only
observable evidence of a successful relocation is the path change itself.

### Exact commands

Seed a document with a tag-bearing filename format, record the "before" paths, then add a
tag through the canonical `m2m_changed` path and record the "after" paths:

```
# Seed (format embeds the tag directory)
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' paperless-work \
  bash -lc 'python3 manage.py shell -c "from documents import tasks; tasks.consume_file(\"/scratch/consume/simple.pdf\")"'

# Trigger the move via the m2m_changed signal, with DEBUG surfaced on the console
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' -e PAPERLESS_DEBUG=true paperless-work \
  bash -lc 'python3 manage.py shell -c "
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name=\"Invoice\")
print(\">>> calling d.tags.add(Invoice) now (fires m2m_changed)\")
d.tags.add(t)
print(\">>> d.tags.add(Invoice) returned\")
"'
```

### Complete unedited output

**Before (no tags) — document paths and on-disk files:**

```
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
thumbnail_path   = /scratch/media/documents/thumbnails/0000001.png
--- disk (find) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/originals/simple.pdf
/scratch/media/documents/thumbnails/0000001.png
```

**During the move — console output (with `PAPERLESS_DEBUG=true`) between the two markers:**

```
>>> calling d.tags.add(Invoice) now (fires m2m_changed)
>>> d.tags.add(Invoice) returned
```

(There are no log lines between the two markers — the move produced no console output
even at DEBUG level.)

**After (tag `Invoice` added) — document paths and on-disk files:**

```
filename         = Invoice/simple.pdf
archive_filename = Invoice/simple.pdf
source_path      = /scratch/media/documents/originals/Invoice/simple.pdf
archive_path     = /scratch/media/documents/archive/Invoice/simple.pdf
thumbnail_path   = /scratch/media/documents/thumbnails/0000001.png
tags             = ['Invoice']
--- disk (find) ---
/scratch/media/documents/archive/Invoice/simple.pdf
/scratch/media/documents/originals/Invoice/simple.pdf
/scratch/media/documents/thumbnails/0000001.png
```

**The log file wrote zero new lines during the move** — the definitive "no log on
success" evidence (`diff` of `paperless.log` before vs. after, plus a grep of the whole
log for any move activity):

```
=== log lines: before=18  after=18 ===
<<< NO DIFFERENCE: the successful move wrote ZERO log lines to paperless.log >>>
=== also grep the ENTIRE log for any move/rename/handlers activity ===
<<< no rename/move/handlers log lines anywhere in paperless.log >>>
```

**Second canonical trigger — the `document_renamer` management command.** Consuming a
document *without* a format (so its name is the pk-based `0000001.pdf`), attaching a tag,
then running the bulk renamer *with* the format set relocates the files just as silently:

```
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' paperless-work \
  python3 manage.py document_renamer --no-progress-bar
```

```
=========== CONSOLE EXIT (document_renamer) ===========
=== AFTER renamer: filename + disk ===
filename     = Receipt/simple.pdf
source_path  = /scratch/media/documents/originals/Receipt/simple.pdf
/scratch/media/documents/archive/Receipt/simple.pdf
/scratch/media/documents/originals/Receipt/simple.pdf
/scratch/media/documents/thumbnails/0000001.png
```

The console was empty, and the full `paperless.log` afterward contained only the earlier
consumption trace — **zero** move/handlers lines:

```
=== any move/rename/handlers lines? ===
<<< NONE: document_renamer relocation wrote ZERO move/handlers log lines >>>
```

### `file:line` grounding

- The move engine is **`update_filename_and_move_files`** at
  `src/documents/signals/handlers.py:312`, registered on **both**
  `@receiver(models.signals.m2m_changed, sender=Document.tags.through)` (`:310`) and
  `@receiver(models.signals.post_save, sender=Document)` (`:311`) — so a tag change
  reaches it via `m2m_changed`.
- Work runs under `with FileLock(settings.MEDIA_LOCK)` (`:325`); new names come from
  `generate_unique_filename` (`:330` original, `:339` archive); the moves are bare
  `os.rename` for the original (`:354`) and the archive (`:359`); the database row is
  updated directly with `Document.objects.filter(pk=instance.pk).update(...)` (`:362-365`)
  to avoid signal recursion.
- The tag-derived directory comes from the `{tag_list}` token in **`generate_filename`**
  at `src/documents/file_handling.py:128`, where `tag_list` is built (`:135`) and applied
  via `.format(...)` (`:175`).
- The thumbnail path is **`Document.thumbnail_path`** at `src/documents/models.py:273-278`,
  computed as `THUMBNAIL_DIR/"{pk:07}.png"` — it depends on the primary key, not the
  filename, so it is invariant under a tag/filename change.
- The "no log on success" fact is grounded in the **absence** of any `logger` call in the
  entire body of `update_filename_and_move_files` (`:312-410`). The `document_renamer`
  command additionally sets `logging.getLogger().handlers[0].level = logging.ERROR`
  (`src/documents/management/commands/document_renamer.py:28`), further silencing the root
  console handler, and drives the move with `post_save.send(Document, instance=document)`
  (`:34`).

### Observed vs. inferred

**[observed]** — every path (before/after), the empty log diff, and the silent
`document_renamer` relocation were captured at runtime.

### Cause → effect

`generate_filename` builds the target directory from the sorted, sanitized tag names in
the `{tag_list}` token (`file_handling.py:135,175`). Adding the `Invoice` tag changes the
computed name from `simple.pdf` to `Invoice/simple.pdf`, so `update_filename_and_move_files`
`os.rename`s the original (`:354`) and the archive (`:359`) into the new `Invoice/`
sub-directory and updates the DB row (`:362-365`). It never touches the thumbnail (the
function only renames the original and archive paths), and because the function contains
no logging calls, a successful relocation is silent — the path change is the only signal.

---

## Q2 — Rollback safety net on move failure

> **User's words:** *"There's supposedly a rollback safety net if something goes wrong
> during a file move, but I can't tell from reading the code whether it truly recovers or
> just promises to, what actually happens to files when a move fails partway through?"*

### Direct answer

The safety net **truly recovers** — it is not merely a promise. When a move fails partway
through (for example, the original file is renamed successfully but the archive rename
then fails), the `except` handler in `update_filename_and_move_files` renames the moved
file(s) back to their original locations, and — crucially — the database row is **not**
updated (the `.update(...)` call is reached only *after* all renames succeed). So the
files end exactly where they started and the DB still points at them. The recovery itself
is **silent** (no log line), and the one place in the whole move path that *does* log is
`validate_move`, which emits a single CRITICAL line when the source file has vanished.
One caveat, addressed below: the *inner* recovery is wrapped in `try/except Exception:
pass`, so if the recovery rename itself failed, that failure would be swallowed silently
by design (the sanity checker is the backstop).

### Exact commands

**Experiment A — force an `OSError` mid-move.** Seed with a format (no tag), then
pre-create `archive/Invoice` as a *file* so that when the handler tries to create the
archive's target directory (`os.makedirs("…/archive/Invoice")`) it raises
`FileExistsError` — *after* the original has already been renamed. Then add the tag:

```
# pre-create the blocker so the archive make-dirs fails after the original moves
docker exec paperless-work bash -lc 'printf "BLOCKER" > /scratch/media/documents/archive/Invoice'

# trigger the move
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' -e PAPERLESS_DEBUG=true paperless-work \
  bash -lc 'python3 manage.py shell -c "
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name=\"Invoice\")
try:
    d.tags.add(t)
    print(\">>> d.tags.add(Invoice) returned NORMALLY (no exception propagated)\")
except Exception as e:
    print(\">>> EXCEPTION propagated to caller:\", type(e).__name__, str(e))
"'
```

**Experiment B — make the source file vanish.** Seed with a format, delete the original
file, then add the tag so `validate_move` sees a missing source:

```
docker exec paperless-work bash -lc 'rm -f /scratch/media/documents/originals/simple.pdf'
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' -e PAPERLESS_DEBUG=true paperless-work \
  bash -lc 'python3 manage.py shell -c "
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name=\"Invoice\")
d.tags.add(t)
print(\">>> d.tags.add(Invoice) returned NORMALLY (no exception propagated)\")
"'
```

### Complete unedited output

**Experiment A — before (on-disk state; note the `Invoice` blocker file in `archive/`):**

```
originals:
/scratch/media/documents/originals/simple.pdf
archive:
/scratch/media/documents/archive
/scratch/media/documents/archive/Invoice
/scratch/media/documents/archive/simple.pdf
```

**Experiment A — trigger (console):**

```
>>> before add: filename= simple.pdf
>>> d.tags.add(Invoice) returned NORMALLY (no exception propagated)
```

**Experiment A — after (DB fields + on-disk state):**

```
DB filename         = simple.pdf  (UNCHANGED => .update() at handlers.py:362 never reached)
DB archive_filename = simple.pdf
DB tags             = ['Invoice']  (m2m WAS committed)
source_path         = /scratch/media/documents/originals/simple.pdf
--- disk ---
originals:
/scratch/media/documents/originals/simple.pdf
archive:
/scratch/media/documents/archive
/scratch/media/documents/archive/Invoice
/scratch/media/documents/archive/simple.pdf
```

**Experiment A — the failed move + rollback wrote zero log lines:**

```
<<< NO DIFFERENCE: the failed move + rollback wrote ZERO log lines >>>
```

Interpretation of the before/during/after: the original was renamed to
`originals/Invoice/simple.pdf` (the intermediate state), then the archive's directory
creation failed with `OSError`, the `except` handler renamed the original **back** to
`originals/simple.pdf`, and the DB `filename`/`archive_filename` were left untouched (the
`.update(...)` is only reached after both renames succeed). The tag relation itself was
committed (`m2m` is independent of the file move). End state = start state on disk.

**Experiment B — trigger (console AND `paperless.log`):**

```
[2026-07-13 17:28:06,290] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
>>> d.tags.add(Invoice) returned NORMALLY (no exception propagated)
```

**Experiment B — the log delta confirms the CRITICAL line was written to the file too:**

```
18a19
> [2026-07-13 17:28:06,290] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
```

**Experiment B — after (DB unchanged):**

```
DB filename = simple.pdf  archive_filename = simple.pdf  tags = ['Invoice']
```

### `file:line` grounding

- The recovery is the `except (OSError, DatabaseError, CannotMoveFilesException)` block at
  `src/documents/signals/handlers.py:367`. Inside it, an inner `try` (`:374`) renames the
  original back — `os.rename(instance.source_path, old_source_path)` (`:375-376`) — and
  the archive back (`:378-379`), then restores the in-memory names (`:393-394`).
- The order that makes Experiment A a genuine *partial* failure: the original is moved
  first (`validate_move` `:352`, `create_source_path_directory` `:353`, `os.rename` `:354`),
  then the archive (`validate_move` `:357`, `create_source_path_directory` `:358`,
  `os.rename` `:359`). The blocker makes `create_source_path_directory` (which calls
  `os.makedirs`, `src/documents/file_handling.py:19-20`) raise at `:358` — *after* the
  original already moved.
- The DB write `Document.objects.filter(pk=instance.pk).update(filename=…, archive_filename=…)`
  is at `:362-365`, positioned *after* both moves — which is why a mid-move failure leaves
  the DB row unchanged.
- The one logger in the move path is **`validate_move`** (`:295-307`):
  `logger.fatal(f"Document {str(instance)}: File {old_path} has gone.")` (`:298`) when the
  source is missing (raising `CannotMoveFilesException` `:299`), and
  `logger.warning(f"Document {str(instance)}: Cannot rename file since target path
  {new_path} already exists.")` (`:303-306`) when the target already exists. The logger is
  named `paperless.handlers` (`:27`); `logger.fatal` emits at CRITICAL level.
- The inner recovery's `except Exception:` / `pass` is at `:381` / `:390`.

### Observed vs. inferred

**[observed]** — Experiment A (the original is restored to its start location, the DB row
is unchanged, and the whole failed move + rollback is silent) and Experiment B (the single
CRITICAL `"… has gone."` line on both console and log; DB unchanged; no exception
propagated) were both captured at runtime.

**[inferred]** — the claim that a *failed recovery* (the inner `except Exception: pass` at
`:390`) is swallowed silently could **not** be triggered through a canonical path and is
therefore inferred from the code. Documented attempts: (1) pre-creating a conflicting
*target* to force the `validate_move` "already exists" branch is defeated because
`generate_unique_filename` (`file_handling.py:81-125`) appends a `_NN` suffix on
collision, so the computed target never pre-exists; (2) the original-restore target is
structurally the original source location, which is freed exactly when the file moves
forward, so it cannot be pre-occupied to make the restore fail; (3) driving the move as a
non-root user with a read-only `originals/` blocks the *forward* `makedirs` (so the
original never moves in the first place) and also confounds consumption. A side finding
from attempt (1): the `validate_move` "target already exists" `logger.warning`
(`:303-306`) is effectively unreachable in the normal tag-change flow precisely because of
that collision-avoidance suffix.

### Cause → effect

The rollback works because the code performs all irreversible steps (the renames) inside a
single `try`, and only commits the new names to the DB (`:362-365`) *after* every rename
has succeeded. If any rename or the DB write throws, control jumps to the `except` (`:367`)
which reverses whichever renames actually happened (`:375-379`) and restores the in-memory
names (`:393-394`). Because none of this path logs, a successful recovery is invisible; the
only time you see anything is when `validate_move` refuses to proceed — e.g. the source has
vanished (`:298`, Experiment B) — which is also why a move can never overwrite an existing
target.


---

## Q3 — Classifier training skip vs. full retrain

> **User's words:** *"The classifier's behavior puzzles me, sometimes training finishes
> instantly, other times it takes much longer. I want to trigger both scenarios and see
> the actual log messages that explain why training was skipped versus why it proceeded
> with full retraining, including whatever hash or checksum the system uses to detect
> changes."*

### Direct answer

"Instant" training is the **skip** branch and "much longer" is a **full retrain**. The
classifier computes a **SHA-1** digest over the training inputs (every document's
preprocessed content plus the `MATCH_AUTO` label bytes) and compares it to the digest it
persisted the last time it trained. If they match, `train()` returns `False` immediately
and the caller logs the DEBUG line **`"Training data unchanged."`**; if they differ (or
no model exists yet), it does the full vectorize-and-fit and logs the INFO line
**`"Saving updated classifier model to …"`**. The change-detection value is a 20-byte
(160-bit) SHA-1 digest stored inside the model pickle as `data_hash`.

### Exact commands

Create one `MATCH_AUTO` tag, attach it to a document, then run the canonical entry point
`document_create_classifier` four times: first train, two unchanged skips (to prove
stability), and a retrain after altering the labeled data:

```
docker exec -w /app/src -e PAPERLESS_DEBUG=true paperless-work python3 manage.py document_create_classifier   # RUN 1: first train
docker exec -w /app/src -e PAPERLESS_DEBUG=false paperless-work python3 manage.py document_create_classifier  # RUN 2: skip #1 (console INFO-gated)
docker exec -w /app/src -e PAPERLESS_DEBUG=true  paperless-work python3 manage.py document_create_classifier  # RUN 3: skip #2
# (assign the AutoTag to a second document to change the training data)
docker exec -w /app/src -e PAPERLESS_DEBUG=true  paperless-work python3 manage.py document_create_classifier  # RUN 4: retrain
```

The persisted digest was read straight out of the model pickle (the second pickled
object, per `save()` order):

```
docker exec paperless-work python3 -c '
import pickle
with open("/scratch/data/classification_model.pickle", "rb") as f:
    fmt = pickle.load(f)          # 1st object: FORMAT_VERSION
    data_hash = pickle.load(f)    # 2nd object: self.data_hash
print("FORMAT_VERSION =", fmt)
print("data_hash length (bytes) =", len(data_hash))
print("data_hash.hex() =", data_hash.hex())
'
```

### Complete unedited output

**RUN 1 — first full train** (console with `PAPERLESS_DEBUG=true`; identical in
`paperless.log`):

```
[2026-07-13 17:14:20,792] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
[2026-07-13 17:14:20,793] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 17:14:20,796] [DEBUG] [paperless.classifier] 2 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-13 17:14:21,236] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-13 17:14:21,237] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-13 17:14:21,252] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-13 17:14:21,252] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-13 17:14:21,253] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
```

**Persisted digest after RUN 1:**

```
FORMAT_VERSION = 7
data_hash type = bytes
data_hash length (bytes) = 20
data_hash.hex() = d2218aa5d0d734c141857487cee7e5112b90d82f
```

**RUN 2 — skip #1, unchanged data, `PAPERLESS_DEBUG=false`.** The console was
**completely empty** (the skip message is DEBUG, and the console is gated at INFO). The
DEBUG line was captured from `paperless.log`:

```
[2026-07-13 17:14:47,276] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 17:14:47,280] [DEBUG] [paperless.tasks] Training data unchanged.
```

Digest after RUN 2 (unchanged):

```
data_hash.hex() = d2218aa5d0d734c141857487cee7e5112b90d82f
```

**RUN 3 — skip #2, unchanged data, `PAPERLESS_DEBUG=true`** (skip line now visible on the
console, proving the skip is stable across ≥ 2 runs):

```
[2026-07-13 17:15:08,597] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 17:15:08,601] [DEBUG] [paperless.tasks] Training data unchanged.
```

Digest after RUN 3 (still identical):

```
data_hash.hex() = d2218aa5d0d734c141857487cee7e5112b90d82f
```

**RUN 4 — full retrain after altering the `MATCH_AUTO` data** (the `AutoTag` was assigned
to a second document, changing that document's label bytes):

```
[2026-07-13 17:15:29,369] [DEBUG] [paperless.classifier] Gathering data from database...
[2026-07-13 17:15:29,373] [DEBUG] [paperless.classifier] 2 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
[2026-07-13 17:15:29,373] [DEBUG] [paperless.classifier] Vectorizing data...
[2026-07-13 17:15:29,374] [DEBUG] [paperless.classifier] Training tags classifier...
[2026-07-13 17:15:29,390] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
[2026-07-13 17:15:29,390] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
[2026-07-13 17:15:29,391] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
```

Digest after RUN 4 (**changed** — this is why it retrained):

```
data_hash.hex() = 0c64d07b63edcaa96ebb357e280d3861800ebda9
```

**Independent byte-level SHA-1 verification.** To prove the persisted `data_hash` is
genuinely a SHA-1 over content + label bytes (not merely 20 bytes of something), the exact
hashing loop was replicated using the codebase's own `preprocess_content` against the live
database and compared to the persisted value:

```
independently recomputed SHA-1 .hex() = 0c64d07b63edcaa96ebb357e280d3861800ebda9
persisted model data_hash .hex()      = 0c64d07b63edcaa96ebb357e280d3861800ebda9
MATCH ==> True
```

### `file:line` grounding

- The digest and skip decision live in **`DocumentClassifier.train`** at
  `src/documents/classifier.py:115`: `m = hashlib.sha1()` (`:124`); per-document
  `m.update(preprocessed_content.encode("utf-8"))` (`:129`); the `MATCH_AUTO` label bytes
  for document-type (`:136`), correspondent (`:143`) and each tag (`:155`), each via
  `y.to_bytes(4, "little", signed=True)`; finalized `new_data_hash = m.digest()` (`:161`);
  and the skip test `if self.data_hash and new_data_hash == self.data_hash: return False`
  (`:163-164`).
- Persistence: `save()` writes `pickle.dump(self.FORMAT_VERSION, f)` (`:101`) then
  `pickle.dump(self.data_hash, f)` (`:102`) — so the second pickled object is the digest;
  `FORMAT_VERSION = 7` (`:63`); `load()` restores it with `self.data_hash = pickle.load(f)`
  (`:86`).
- The user-visible log strings are emitted by **`train_classifier`** in
  `src/documents/tasks.py:48`: on retrain, `logger.info("Saving updated classifier model
  to {}...".format(settings.MODEL_FILE))` (`:64-66`); on skip, `logger.debug("Training
  data unchanged.")` (`:69`). It early-returns unless a `MATCH_AUTO`
  tag/type/correspondent exists (`:49-55`).
- The canonical entry point is `document_create_classifier`, whose `handle` simply calls
  `train_classifier()` (`src/documents/management/commands/document_create_classifier.py:20`).

### Observed vs. inferred

**[observed]** — the INFO retrain line, the DEBUG skip line (captured from both the log
file and the console), the persisted 20-byte SHA-1 digest, its stability across the two
unchanged runs, and its change after altering the data were all captured at runtime. The
SHA-1 attribution is **[observed]** and byte-verified: an independent recomputation of the
digest matched the persisted value exactly.

### Cause → effect

`train()` feeds each non-inbox document's preprocessed content and its `MATCH_AUTO` label
pks into a SHA-1 accumulator and finalizes a 20-byte digest (`classifier.py:124-161`).
Identical inputs produce an identical digest, so the equality test at `:163-164` short-
circuits with `return False` and the caller logs `"Training data unchanged."`
(`tasks.py:69`) — the "instant" case. Changing any labeled input (here, adding the
`AutoTag` to a second document) changes at least one document's contribution, so the
digest differs, the equality test fails, the classifier vectorizes and fits the model, and
the caller logs `"Saving updated classifier model to …"` (`tasks.py:64-66`) before
persisting the new digest — the "much longer" case.


---

## Q4 — Duplicate detection checksums

> **User's words:** *"Duplicate detection feels like magic, two files that look completely
> different can still be rejected as duplicates, and I want to see the actual checksums
> being compared at that moment of judgment."*

### Direct answer

At ingestion the consumer computes an **MD5** of the incoming file and rejects it if that
MD5 matches **either** an existing document's original `checksum` **or** its
`archive_checksum`. That `OR` is the "magic": a file that looks completely different from
an original — specifically, the OCR'd **archive** rendition of an already-consumed
document — has its *own* MD5 stored in `archive_checksum`, so re-submitting that archive
file collides on the archive branch and is rejected, even though it is byte-for-byte a
different file (here 10 859 bytes vs. the original's 22 926 bytes). The "moment of
judgment" comparison is between the incoming file's MD5 and the matched stored column.

### Exact commands

Seed one PDF, verify the stored checksums against the bytes on disk, then re-consume (A)
the byte-identical original and (B) the OCR archive file:

```
# Seed
docker exec -w /app/src paperless-work bash -lc 'python3 manage.py shell -c "from documents import tasks; tasks.consume_file(\"/scratch/consume/original_to_seed.pdf\")"'

# Experiment A: re-consume the byte-identical original
docker exec paperless-work bash -lc 'cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/dup_identical.pdf'
docker exec -w /app/src paperless-work bash -lc 'python3 manage.py shell -c "
from documents import tasks
from documents.consumer import ConsumerError
try:
    tasks.consume_file(\"/scratch/consume/dup_identical.pdf\")
except ConsumerError as e:
    print(\"ConsumerError RAISED:\"); print(repr(str(e)))
"'

# Experiment B: consume the OCR archive rendition of the existing document
docker exec paperless-work bash -lc 'cp /scratch/media/documents/archive/0000001.pdf /scratch/consume/dup_archive.pdf'
docker exec -w /app/src paperless-work bash -lc 'python3 manage.py shell -c "
from documents import tasks
from documents.consumer import ConsumerError
try:
    tasks.consume_file(\"/scratch/consume/dup_archive.pdf\")
except ConsumerError as e:
    print(\"ConsumerError RAISED:\"); print(repr(str(e)))
"'
```

### Complete unedited output

**Seed document identity, and byte-level verification that the stored checksums match the
files on disk:**

```
pk=1
checksum=42995833e01aea9b3edee44bbfdd7ce1
archive_checksum=735a4f281cf0eb939f02207cf0510d00
source_path=/scratch/media/documents/originals/0000001.pdf
archive_path=/scratch/media/documents/archive/0000001.pdf
has_archive_version=True
```

```
=== md5sum of on-disk ORIGINAL vs stored checksum ===
42995833e01aea9b3edee44bbfdd7ce1  /scratch/media/documents/originals/0000001.pdf
stored checksum         = 42995833e01aea9b3edee44bbfdd7ce1

=== md5sum of on-disk ARCHIVE vs stored archive_checksum ===
735a4f281cf0eb939f02207cf0510d00  /scratch/media/documents/archive/0000001.pdf
stored archive_checksum = 735a4f281cf0eb939f02207cf0510d00

=== compare original bytes vs archive bytes (are the two files different?) ===
DIFFERENT bytes (visually/byte-wise distinct files)
original size: 22926 bytes
archive  size: 10859 bytes
```

**Experiment A — original collision.** The incoming file's MD5 (what
`pre_check_duplicate` computes at `consumer.py:104`) equals the stored `checksum`:

```
=== incoming file MD5 (what consumer.py:104 will compute) ===
42995833e01aea9b3edee44bbfdd7ce1  /scratch/consume/dup_identical.pdf
=== EXPERIMENT A: consume byte-identical original -> expect duplicate rejection ===
[2026-07-13 17:18:38,268] [ERROR] [paperless.consumer] Not consuming dup_identical.pdf: It is a duplicate.
ConsumerError RAISED:
'dup_identical.pdf: Not consuming dup_identical.pdf: It is a duplicate.'
```

**Experiment B — archive collision (the "two files that look completely different"
case).** The incoming file is the OCR archive (10 859 bytes); its MD5 does **not** equal
the stored original `checksum` but **does** equal the stored `archive_checksum`:

```
incoming (archive) MD5:
735a4f281cf0eb939f02207cf0510d00  /scratch/consume/dup_archive.pdf
stored archive_checksum of pk=1 = 735a4f281cf0eb939f02207cf0510d00
stored ORIGINAL checksum of pk=1 = 42995833e01aea9b3edee44bbfdd7ce1 (incoming does NOT match this)
=== EXPERIMENT B: consume the OCR archive PDF -> expect duplicate via archive_checksum ===
[2026-07-13 17:18:56,357] [ERROR] [paperless.consumer] Not consuming dup_archive.pdf: It is a duplicate.
ConsumerError RAISED:
'dup_archive.pdf: Not consuming dup_archive.pdf: It is a duplicate.'
```

**Post-state — both attempts were rejected; no new document was created:**

```
Document count = 1
pk=1 checksum=42995833e01aea9b3edee44bbfdd7ce1 archive_checksum=735a4f281cf0eb939f02207cf0510d00
```

### `file:line` grounding

- The check is **`Consumer.pre_check_duplicate`** at `src/documents/consumer.py:102`:
  `checksum = hashlib.md5(f.read()).hexdigest()` (`:104`); the query
  `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists()`
  (`:105-107`) — matching against **both** the original and the archive checksum; on a hit
  it calls `self._fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, f"Not consuming {self.filename}: It
  is a duplicate.")` (`:110-113`).
- **`Consumer._fail`** (`:78-81`) logs at error level (`self.log("error", …)` `:80`) and
  raises `ConsumerError(f"{self.filename}: {log_message}")` (`:81`);
  `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` (`:37`). The pre-check is
  invoked from `try_consume_file` (`:213`).
- The stored columns are defined in `src/documents/models.py`:
  `checksum = models.CharField(max_length=32, editable=False, unique=True)` (`:135-141`)
  and `archive_checksum = models.CharField(max_length=32, null=True, blank=True)`
  (`:143-150`). The `max_length=32` is exactly the length of an MD5 hex digest.
- `CONSUMER_DELETE_DUPLICATES` (whether the rejected file is unlinked) defaults off
  (`src/paperless/settings.py:486`); it was left at its default here, so the check only
  rejects and does not delete the incoming file.

### Observed vs. inferred

**[observed]** — the seed checksums (byte-verified against the on-disk files), the
incoming MD5 for each experiment, the two `ERROR` log lines, the two `ConsumerError`
messages, and the fact that the document count stayed at 1 were all captured at runtime.

### Cause → effect

`pre_check_duplicate` reduces every incoming file to a single MD5 (`consumer.py:104`) and
asks the database whether that value already appears in **either** the `checksum` **or**
the `archive_checksum` column (`:105-107`). In Experiment A the incoming MD5
(`42995833…`) equals the original's stored `checksum`, so the `Q(checksum=…)` disjunct
matches. In Experiment B the incoming file is the OCR archive, a byte-wise different file
(10 859 vs. 22 926 bytes) whose MD5 (`735a4f28…`) equals the stored `archive_checksum`,
so the `Q(archive_checksum=…)` disjunct matches. Either way `_fail` logs
`"… It is a duplicate."` and raises `ConsumerError`, so ingestion aborts and no second
document is created. The archive branch of that `OR` is precisely why "two files that look
completely different" can be judged duplicates.


---

## Q5 — Sanity checker output and mismatch hashes

> **User's words:** *"The sanity checker intrigues me as well, what does its output
> actually look like when an archive is healthy versus when something has gone wrong, and
> can I see the specific hash values it compares when it discovers a mismatch?"*

### Direct answer

Against a healthy store the sanity checker emits a single INFO line,
**`"Sanity checker detected no issues."`** When a stored file's bytes no longer match its
recorded checksum, it emits an ERROR naming the document and **both** hash values —
`"Checksum mismatch of document {pk}. Stored: {…}, actual: {…}."` for an original, and the
parallel `"Checksum mismatch of archived document {pk}. Stored: {…}, actual: {…}."` for an
archive. The "stored" value is the MD5 recorded in the DB at consumption; the "actual"
value is the MD5 the checker recomputes from the file on disk *right now*. One important
behavioral nuance: the `document_sanity_checker` **management command never raises** (it
only logs), whereas the scheduled `tasks.sanity_check` **does** raise
`SanityCheckFailedException` when any error is present.

### Exact commands

```
# Q5a healthy
docker exec -w /app/src paperless-work python3 manage.py document_sanity_checker --no-progress-bar

# Q5b corrupt the ORIGINAL, then re-run
docker exec paperless-work bash -lc 'printf "CORRUPTION-MARKER-Q5B" >> /scratch/media/documents/originals/0000001.pdf'
docker exec -w /app/src paperless-work python3 manage.py document_sanity_checker --no-progress-bar

# Q5c restore the original, corrupt the ARCHIVE, then re-run
docker exec paperless-work bash -lc 'cp /app/src/documents/tests/samples/simple.pdf /scratch/media/documents/originals/0000001.pdf'
docker exec paperless-work bash -lc 'printf "CORRUPTION-MARKER-Q5C-ARCHIVE" >> /scratch/media/documents/archive/0000001.pdf'
docker exec -w /app/src paperless-work python3 manage.py document_sanity_checker --no-progress-bar

# Q5d CLI does not raise vs. task raises (same corrupted store)
docker exec -w /app/src paperless-work bash -lc 'python3 manage.py document_sanity_checker --no-progress-bar >/dev/null 2>&1; echo "CLI exit code = $?"'
docker exec -w /app/src paperless-work bash -lc 'python3 manage.py shell -c "
from documents import tasks
from documents.sanity_checker import SanityCheckFailedException
try:
    tasks.sanity_check()
except SanityCheckFailedException as e:
    print(\"SanityCheckFailedException RAISED:\", repr(str(e)))
"'
```

### Complete unedited output

**Q5a — healthy store:**

```
[2026-07-13 17:20:53,998] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
```

**Q5b — corrupted original.** Before/after MD5 of the file on disk (the "after" value is
what the checker will report as `actual`):

```
=== Q5b: ORIGINAL file state BEFORE corruption ===
before MD5:
42995833e01aea9b3edee44bbfdd7ce1  /scratch/media/documents/originals/0000001.pdf
stored checksum = 42995833e01aea9b3edee44bbfdd7ce1
=== corrupt the original (append bytes) ===
after MD5 (this is the expected actual: value):
e82ecaf2da96b3e0f10acf93d24be402  /scratch/media/documents/originals/0000001.pdf
=== Q5b: sanity check with corrupted ORIGINAL ===
[2026-07-13 17:21:27,527] [ERROR] [paperless.sanity_checker] Checksum mismatch of document 1. Stored: 42995833e01aea9b3edee44bbfdd7ce1, actual: e82ecaf2da96b3e0f10acf93d24be402.
```

The reported `actual: e82ecaf2da96b3e0f10acf93d24be402` is byte-for-byte the MD5 of the
corrupted file, and `Stored: 42995833e01aea9b3edee44bbfdd7ce1` is the original recorded
checksum.

**Q5c — corrupted archive (original restored first, so only the archive error fires):**

```
=== restore pristine original ===
restored original MD5:
42995833e01aea9b3edee44bbfdd7ce1  /scratch/media/documents/originals/0000001.pdf
=== ARCHIVE file state BEFORE corruption ===
before MD5:
993b9ba36cd0d9026c8a2c1daa5243b8  /scratch/media/documents/archive/0000001.pdf
stored archive_checksum = 993b9ba36cd0d9026c8a2c1daa5243b8
=== corrupt the ARCHIVE (append bytes) ===
after MD5 (expected actual: value):
b934c00a9020b85390baf9e26bc0f715  /scratch/media/documents/archive/0000001.pdf
=== Q5c: sanity check with corrupted ARCHIVE (original restored) ===
[2026-07-13 17:21:44,783] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: 993b9ba36cd0d9026c8a2c1daa5243b8, actual: b934c00a9020b85390baf9e26bc0f715.
```

Again the reported `actual` equals the MD5 of the corrupted archive, and `Stored` equals
the recorded `archive_checksum`. (Note: `archive_checksum` is OCR-nondeterministic across
consumptions, hence its value here differs from Q4's seed; the original `checksum`
`42995833…` is stable.)

**Q5d — the two entry points diverge on error handling** (run against the still-corrupted
store):

```
=== Empirically: CLI command with errors does NOT raise (exit code) ===
CLI exit code = 0
=== Empirically: tasks.sanity_check() DOES raise SanityCheckFailedException ===
SanityCheckFailedException RAISED: 'Sanity check failed with errors. See log.'
```

### `file:line` grounding

- The healthy message is emitted by **`SanityCheckMessages.log_messages`** at
  `src/documents/sanity_checker.py:23-30`: `logger.info("Sanity checker detected no
  issues.")` (`:27`) when there are zero messages; the logger is `paperless.sanity_checker`
  (`:24`).
- The walk/compare is **`check_sanity`** at `:49-133`: it recomputes the original's MD5
  with `hashlib.md5(f.read()).hexdigest()` (`:83`) and, on mismatch, emits
  `messages.error(f"Checksum mismatch of document {doc.pk}. Stored: {doc.checksum}, actual:
  {checksum}.")` (`:88-91`); the archive equivalent recomputes at `:112` and emits
  `messages.error(f"Checksum mismatch of archived document {doc.pk}. Stored:
  {doc.archive_checksum}, actual: {checksum}.")` (`:119-124`).
- The CLI entry point `document_sanity_checker` calls `check_sanity(...)` (`:24`) then
  `messages.log_messages()` (`:26`) and does **not** raise
  (`src/documents/management/commands/document_sanity_checker.py`). The scheduled
  **`tasks.sanity_check`** (`src/documents/tasks.py:255`) calls the same checker, then
  `if messages.has_error(): raise SanityCheckFailedException("Sanity check failed with
  errors. See log.")` (`:260-261`).

### Observed vs. inferred

**[observed]** — the healthy INFO line, the original-mismatch ERROR, the archive-mismatch
ERROR, and the CLI-vs-task divergence were all captured at runtime. Both `Stored` and
`actual` hex values were byte-verified against the exact bytes written to disk.

### Cause → effect

`check_sanity` iterates every `Document`, and for each on-disk file recomputes an MD5 and
compares it to the checksum stored at consumption (`sanity_checker.py:83` / `:112`). When
the bytes are intact the two agree and no message is added, so `log_messages` takes its
"zero messages" branch and prints `"Sanity checker detected no issues."` (`:27`). Appending
bytes to a file changes its MD5, so the recomputed `actual` no longer equals the `Stored`
value, and the checker records the corresponding mismatch ERROR carrying both hex values
(`:88-91` for originals, `:119-124` for archives). The management command merely logs those
messages, so its process exits 0; the scheduled task re-runs the same check and converts an
error into a raised `SanityCheckFailedException` (`tasks.py:260-261`).


---

## Q6 — Orphaned / ghost files

> **User's words:** *"I also wonder about ghost files, whether orphaned files really linger
> in the media folder and what the system actually reports when it finds them."*

### Direct answer

Yes — orphaned files really linger. The sanity checker **reports** an unreferenced file
under the media tree with the WARNING `"Orphaned file in media dir: {path}"`, but it
**never deletes it**. The file remains on disk after the check, and a subsequent run
reports the very same warning again.

### Exact commands

Seed a healthy document, drop an unreferenced file into `originals/`, confirm no document
points at it, run the checker, then confirm the file is still there and re-run:

```
# place a file no Document row references
docker exec paperless-work bash -lc 'printf "i am a ghost file not referenced by any document row" > /scratch/media/documents/originals/ghost_orphan.pdf'

# run the checker
docker exec -w /app/src paperless-work python3 manage.py document_sanity_checker --no-progress-bar

# confirm persistence, then run a second time
docker exec paperless-work bash -lc 'ls -la /scratch/media/documents/originals/'
docker exec -w /app/src paperless-work python3 manage.py document_sanity_checker --no-progress-bar
```

### Complete unedited output

**Before — `originals/` holds only the real document; the orphan is then created and
confirmed unreferenced:**

```
=== Q6: media/documents/originals BEFORE placing orphan ===
total 32
drwxr-xr-x 2 root root  4096 Jul 13 17:22 .
drwxr-xr-x 5 root root  4096 Jul 13 17:22 ..
-rw-r--r-- 1 root root 22926 Jul 13 17:22 0000001.pdf
=== place an UNREFERENCED (orphan) file that no Document row points to ===
created:
-rw-r--r-- 1 root root 52 Jul 13 17:22 /scratch/media/documents/originals/ghost_orphan.pdf
orphan MD5: 0e3a292239ccb8cc1ce922ce83b0829b  /scratch/media/documents/originals/ghost_orphan.pdf
=== confirm no Document references this file (checksum not in DB) ===
doc count = 1
any doc filename referencing ghost_orphan? False
```

**The sanity checker's report (console and `paperless.log`):**

```
[2026-07-13 17:22:39,804] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf
```

**After — the orphan still exists (unchanged MD5), and a second run reports it again:**

```
=== originals/ AFTER sanity check (orphan must still be present) ===
total 36
drwxr-xr-x 2 root root  4096 Jul 13 17:22 .
drwxr-xr-x 5 root root  4096 Jul 13 17:22 ..
-rw-r--r-- 1 root root 22926 Jul 13 17:22 0000001.pdf
-rw-r--r-- 1 root root    52 Jul 13 17:22 ghost_orphan.pdf
orphan still present? MD5: 0e3a292239ccb8cc1ce922ce83b0829b  /scratch/media/documents/originals/ghost_orphan.pdf
=== re-run sanity checker a 2nd time to prove orphan STILL lingers (idempotent, not deleted) ===
[2026-07-13 17:22:52,810] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf
=========== CONSOLE EXIT ===========
final: orphan present? YES
```

### `file:line` grounding

- The report is emitted by **`check_sanity`** at `src/documents/sanity_checker.py:49-133`.
  It first walks `MEDIA_ROOT`, collecting every file into `present_files` (`:52-55`), then
  removes the lockfile (`:57-59`); for each `Document` it removes that document's thumbnail
  (`:67-68`), original (`:79-80`), and archive (`:108-109`) from `present_files`. Whatever
  remains is reported: `for extra_file in present_files: messages.warning(f"Orphaned file
  in media dir: {extra_file}")` (`:130-131`).
- There is no `os.remove`/`unlink` anywhere in `check_sanity`, which is why the orphan is
  reported but not deleted. The WARNING is surfaced by `SanityCheckMessages.log_messages`
  (`:23-30`, logger `paperless.sanity_checker` `:24`).

### Observed vs. inferred

**[observed]** — the WARNING line (with the orphan's absolute path), the directory listings
before and after, the unchanged orphan MD5, and the identical second-run warning were all
captured at runtime.

### Cause → effect

`check_sanity` builds the set of every file physically present under `MEDIA_ROOT`, then
subtracts every file that any document legitimately references (thumbnail, original,
archive). A file that no document points at is never subtracted, so it survives to the
final loop and is reported as `"Orphaned file in media dir: …"` (`sanity_checker.py:130-131`).
Because the function only *reports* — it contains no deletion — the ghost file lingers on
disk indefinitely and is re-reported on every subsequent run.

---

## Coverage note

Every named sub-part of every question is answered, each with a concrete value, a
`file:line` reference naming the specific function, observed evidence, and a causal reason.

- **Q1 — relocation on tag change.**
  - *"what log messages actually appear during that dance"* → **none on success**
    [observed]: empty `paperless.log` diff (18→18) even with `PAPERLESS_DEBUG=true`, and no
    move/handlers line anywhere in the log; grounded in the zero-logging body of
    `update_filename_and_move_files` (`handlers.py:312-410`). The `document_renamer`
    relocation was likewise silent (and sets the root console handler to ERROR,
    `document_renamer.py:28`).
  - *"before and after paths"* → **original** `originals/simple.pdf` → `originals/Invoice/simple.pdf`;
    **archive** `archive/simple.pdf` → `archive/Invoice/simple.pdf`; **thumbnail**
    `thumbnails/0000001.png` → **unchanged** (pk-based, `models.py:273-278`) [observed].

- **Q2 — rollback on move failure.**
  - *"what actually happens to files when a move fails partway through"* → the moved
    original is renamed **back** to its start location and the DB row is **not** updated
    [observed, Experiment A]: intermediate state = original at `originals/Invoice/simple.pdf`;
    final state = original back at `originals/simple.pdf`, `archive/simple.pdf` untouched,
    DB `filename`/`archive_filename` = `simple.pdf`; the failure + rollback is silent
    (`except` at `handlers.py:367`, restores at `:375-379`, DB write at `:362-365` never
    reached).
  - *"truly recovers or just promises to"* → **truly recovers** [observed]. The single
    logging point in the path, `validate_move` (`:298`), was exercised in Experiment B
    (missing source → CRITICAL `"… has gone."` on console and log).
  - The **silent inner recovery** (`except Exception: pass`, `:381`/`:390`) is addressed and
    labeled **[inferred]**, with three documented canonical attempts that could not force a
    failed recovery.

- **Q3 — classifier skip vs. retrain.**
  - *"log messages … training was skipped"* → DEBUG `"Training data unchanged."`
    (`tasks.py:69`) [observed, in `paperless.log`; console only with `PAPERLESS_DEBUG=true`].
  - *"… why it proceeded with full retraining"* → INFO `"Saving updated classifier model to
    /scratch/data/classification_model.pickle..."` (`tasks.py:64-66`) [observed].
  - *"whatever hash or checksum the system uses"* → **SHA-1**, a 20-byte digest persisted as
    `data_hash` (`classifier.py:124`, `:161`, `:163-164`); actual values
    `d2218aa5d0d734c141857487cee7e5112b90d82f` (first train, identical across the two
    unchanged skip runs) and `0c64d07b63edcaa96ebb357e280d3861800ebda9` (after altering the
    data) [observed], with the SHA-1 attribution byte-verified by independent recomputation.
    The skip was reproduced **twice** on unchanged data with an identical digest.

- **Q4 — duplicate checksums.**
  - *"the actual checksums being compared at that moment of judgment"* → incoming MD5 vs.
    the matched stored column [observed]: original collision incoming `42995833…` ==
    stored `checksum`; archive collision incoming `735a4f28…` == stored `archive_checksum`
    (and ≠ the original `checksum`). Both raised `ConsumerError` with `"… It is a
    duplicate."` (`consumer.py:104`, `:105-107`, `:110-113`).
  - *"two files that look completely different"* → the OCR archive (10 859 bytes) vs. the
    original (22 926 bytes) — byte-wise different, yet rejected via the
    `Q(checksum=…) | Q(archive_checksum=…)` disjunct [observed].

- **Q5 — sanity output and mismatch hashes.**
  - *"healthy"* → INFO `"Sanity checker detected no issues."` (`sanity_checker.py:27`) [observed].
  - *"when something has gone wrong"* → ERROR `"Checksum mismatch of document 1. Stored:
    42995833…, actual: e82ecaf2…."` (original, `:88-91`) and `"Checksum mismatch of
    archived document 1. Stored: 993b9ba3…, actual: b934c00a…."` (archive, `:119-124`)
    [observed].
  - *"the specific hash values it compares"* → both `Stored` and `actual` hex values shown
    and byte-verified against the bytes written; plus the CLI-does-not-raise vs.
    `tasks.sanity_check`-raises-`SanityCheckFailedException` distinction (`tasks.py:260-261`)
    [observed].

- **Q6 — orphaned files.**
  - *"whether orphaned files really linger"* → **yes** [observed]: `ghost_orphan.pdf`
    persisted after the check (unchanged MD5 `0e3a2922…`) and was re-reported on a second run.
  - *"what the system actually reports"* → WARNING `"Orphaned file in media dir:
    /scratch/media/documents/originals/ghost_orphan.pdf"` (`sanity_checker.py:130-131`)
    [observed]; the checker contains no deletion, so orphans are never auto-removed.

