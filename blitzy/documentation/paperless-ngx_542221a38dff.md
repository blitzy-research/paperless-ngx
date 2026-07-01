# Paperless-NGX — Six Hidden Choreographies of the Backend, Explained and Proven

> An evidence-first, code-comprehension walkthrough of six backend behaviors, written
> from **observed runtime output** — not from reading the source alone.

**Commit under study:** `542221a38dff06361e07976452f9aea24d210542` (short: `542221a38dff`, the source branch `paperless-ngx_542221a38dff`).
All `file:line` references in this document point at the code **as it exists at this commit**, verified by reading each file directly.

---

## Preamble — what this document is, and how the evidence was produced

This document answers six questions about how the Paperless-NGX backend *actually behaves at runtime*:

1. **File relocation on tag change** — how a stored file physically moves when a document's tags change, the log messages during the move, and the concrete before/after on-disk paths.
2. **Rollback safety net** — whether the file-move rollback truly recovers or merely "promises to," and what happens to files when a move fails partway through.
3. **Classifier training: skip vs. retrain** — how to trigger both the instant-skip and the full-retrain scenarios, the log messages that explain each, and the hash the system uses to detect change.
4. **Duplicate detection** — how two files that "look completely different" are still rejected as duplicates, showing the actual checksums compared at the moment of judgment.
5. **Sanity checker: healthy vs. mismatch** — what the checker's output looks like when an archive is healthy versus corrupt, and the specific hash values it compares on a mismatch.
6. **Orphaned / ghost files** — whether orphaned files really linger in the media folder and what the system reports when it finds them.

Every section below is structured identically: **Mechanism** (grounded in `file:line` citations), **Trigger** (the exact code/command that produced the evidence), **Verbatim evidence** (the captured output, unedited), **Reasoning** (*why* the system behaves this way), and explicit **sub-part answers** (before/after, log message — *including "no log" where the path is silent* — and the specific hash/value).

### Environment used to capture the evidence

The behaviors were exercised against a **real, booting Django app** loading the project's own `paperless.settings`, on the following stack:

| Component | Value |
|-----------|-------|
| Python | 3.10.20 (highest version in the CI matrix — `.github/workflows/reusable-ci-backend.yml:L55`) |
| Django | 4.0.4 (exact `Pipfile.lock` pin) |
| scikit-learn | 1.0.2 (exact pin — used only by §3) |
| Database | **SQLite** — the project default when `PAPERLESS_DBHOST` is unset (`src/paperless/settings.py:L297-L300`, with the PostgreSQL branch guarded by `if os.getenv("PAPERLESS_DBHOST")` at `:L304`) |
| Media root | a throwaway temporary directory (per-scenario), set via `PAPERLESS_MEDIA_ROOT` (`src/paperless/settings.py:L61`) |
| Channel layer | in-memory (`channels.layers.InMemoryChannelLayer`) — no Redis, no PostgreSQL, no OCR engine |
| Observation harness | temporary Python scripts living **outside** the repository working tree (`/tmp/obs`), removed after capture |
| Run date | **2026-07-01** (relevant to §2 — see caveat 3 below) |

The observation scripts drove the **real code paths** (Django `post_save`/`m2m_changed` signals, `DocumentClassifier.train()`, `train_classifier()`, `Consumer.pre_check_duplicate()`, `check_sanity()`) — never reimplementations. A per-run in-memory logging handler was attached to the relevant `paperless.*` loggers to capture each `(logger_name, level, message)` triple verbatim.

### The shared observation bootstrap (imported by every trigger below)

So that each section's **Trigger** can be a complete, self-contained excerpt without repeating boilerplate, every trigger begins by importing two helpers from one shared module (`/tmp/obs/obs_common.py`, kept outside the repository and deleted after capture). This is the exact producing scaffold; each section's snippet runs *after* `boot()` and uses `LogCapture` to record the log lines quoted in that section's evidence:

```python
# /tmp/obs/obs_common.py — the shared bootstrap (lives OUTSIDE the repo; removed after capture)
import os, sys, shutil, logging

_ROOT = "/tmp/obs"                                   # fixed, human-readable media/data root
for _d in ("media", "data", "consume"):              # fresh slate each run -> deterministic PKs/paths
    p = os.path.join(_ROOT, _d)
    if os.path.isdir(p): shutil.rmtree(p)
    os.makedirs(p, exist_ok=True)
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
os.environ["PAPERLESS_DISABLE_DBHANDLER"] = "true"
os.environ["PAPERLESS_MEDIA_ROOT"] = os.path.join(_ROOT, "media")
os.environ["PAPERLESS_DATA_DIR"]   = os.path.join(_ROOT, "data")
os.environ["PAPERLESS_CONSUMPTION_DIR"] = os.path.join(_ROOT, "consume")
sys.path.insert(0, os.environ["PAPERLESS_SRC"])      # repo's src/ dir

def boot():
    import django; django.setup()
    from django.core.management import call_command
    call_command("migrate", run_syncdb=True, verbosity=0, interactive=False)  # fresh SQLite DB

class LogCapture(logging.Handler):                   # records (logger_name, levelname, message) triples
    def __init__(self, names):
        super().__init__(level=logging.DEBUG); self.records = []; self._lg = []
        for n in names:
            lg = logging.getLogger(n); lg.setLevel(logging.DEBUG); lg.addHandler(self); self._lg.append(lg)
    def emit(self, r): self.records.append((r.name, r.levelname, r.getMessage()))
    def __enter__(self): return self
    def __exit__(self, *e):
        for lg in self._lg: lg.removeHandler(self)
```

Each section's Trigger is prefaced (implicitly) by `from obs_common import boot, LogCapture; boot()` and, where the behavior writes files, `from django.conf import settings`. Invocation for every section: `PAPERLESS_SRC=<repo>/src python /tmp/obs/<script>.py`. Because `boot()` resets the media tree and SQLite DB on each run, the primary keys, paths, and (for §3) the SHA-1 digest are deterministic across runs — see the caveats below.

### The confirmed logger names (used to filter and quote captured output)

| Logger | Defined at |
|--------|-----------|
| `paperless.handlers` | `src/documents/signals/handlers.py:L27` |
| `paperless.filehandling` | `src/documents/file_handling.py:L11` |
| `paperless.classifier` | `src/documents/classifier.py:L21` |
| `paperless.tasks` | `src/documents/tasks.py:L29` |
| `paperless.consumer` | `src/documents/consumer.py:L54` (`logging_name = "paperless.consumer"`) |
| `paperless.sanity_checker` | `src/documents/sanity_checker.py:L24` (inside `log_messages()`) |

### The four reproducibility caveats (read these before trusting any literal below)

These explain *why* certain literals in the evidence are environment-specific, and are the reason this document always shows the exact input that produced each digest.

1. **Digests are input-deterministic.** The MD5 values in §4 and §5 depend on the exact file bytes hashed; the SHA-1 `data_hash` in §3 depends on the preprocessed document *content* plus the auto-matching `document_type`/`correspondent` primary keys and the sorted auto-matching `Tag` primary keys (`src/documents/classifier.py:L124-L161`). Quote *your* observed digest and show the exact input bytes/seed that produced it — which is what this document does.
2. **A fresh database is required for the small primary keys.** The document PK printed in §5 ("document 1") and §4 ("pk=1"), and the SHA-1 in §3, depend on SQLite's autoincrement state. Each scenario was run against a fresh/isolated database (rows deleted and `sqlite_sequence` reset) so PKs start at 1.
3. **The created-date literal is the *run* date.** §2's message `Document <date> <title>` embeds the document's *created* date, which for a freshly-created row equals the day the code runs — because `str(instance)` formats `self.created` (`src/documents/models.py:L212-L220`) and `created` defaults to `timezone.now` (`src/documents/models.py:L152`). The capture below was produced on **2026-07-01**, so it reads `Document 2026-07-01 …`; a run on another day prints that day's date. This is expected, not a discrepancy.
4. **Silent success/recovery is *itself* the observed truth.** A successful move (§1) emits **no** log line, and the `OSError` rollback paths (§2, scenarios B and C — including the scenario-C reversal of an already-completed rename) emit **no** log line. For these, the evidence is the filesystem/database state, and the *absence* of a log line is the finding — stated explicitly where it applies.

> **Honesty note on the SHA-1 in §3.** With the **fully-specified seed shown in §3** run against a **fresh database** (PKs starting at 1), the SHA-1 `data_hash` observed here — `84b43315ab3567f266bea8387e7d585ea6b33a9b` — is **reproducible byte-for-byte** across repeated runs at this commit (verified over three independent runs). It does **not**, however, match digests produced by *different* seeds or PK assignments (earlier observations with other seeds recorded `182885fbcaf2ee163ce2d7d096401bf2da47b766`, `b41ce39793cd61f391afe34e6e4de9b361c74447`, and `86df1500076a69ca7d63fc5ed8826adf9dc0137a`). This is *proof* of caveat 1: the specific digest is a function of the exact seed — document content plus the auto `document_type`/`correspondent`/`Tag` primary keys — so a fresh DB with this seed reproduces it, while a different seed or non-fresh DB (caveat 2) yields a different value. The same input-determinism applies to the MD5 values in §4 and §5.

---

## §1 — File relocation on tag change

> *"When a document's tags change and the filename format includes tags in the directory structure, files apparently relocate themselves. I wonder what log messages actually appear during that dance, and what the before and after paths look like in practice."*

### Mechanism

Relocation is **signal-driven**. The function `update_filename_and_move_files` is registered for **both** signals:

- `@receiver(models.signals.m2m_changed, sender=Document.tags.through)` — fires when the tag set changes (`src/documents/signals/handlers.py:L310`)
- `@receiver(models.signals.post_save, sender=Document)` — fires on every save (`src/documents/signals/handlers.py:L311`)

with the handler defined at `src/documents/signals/handlers.py:L312`. Inside, the work happens under a lock — `with FileLock(settings.MEDIA_LOCK):` (`src/documents/signals/handlers.py:L325`) — and the new name is computed via `instance.filename = generate_unique_filename(instance)` (`src/documents/signals/handlers.py:L330`). On a real change, the file is physically moved with `os.rename(old_source_path, instance.source_path)` for the original (`src/documents/signals/handlers.py:L354`) and `os.rename(old_archive_path, instance.archive_path)` for the archive (`src/documents/signals/handlers.py:L359`), and the database row is updated *without* re-saving to avoid infinite recursion: `Document.objects.filter(pk=instance.pk).update(filename=…, archive_filename=…)` (`src/documents/signals/handlers.py:L362-L365`).

The path itself is built by `generate_filename` (`src/documents/file_handling.py:L128`). The tag placeholder is produced by joining the document's tag names, **sorted alphabetically**, comma-separated, then sanitized:

```python
tag_list = pathvalidate.sanitize_filename(
    ",".join(sorted([tag.name for tag in doc.tags.all()])),
    replacement_text="-",
)
```
(`src/documents/file_handling.py:L135-L138`). The final relative path is rendered with **plain Python `str.format()`**: `path = settings.PAPERLESS_FILENAME_FORMAT.format(title=…, tag_list=…, …)` (`src/documents/file_handling.py:L161-L162`). Immediately afterward, `path = path.strip(os.sep)` (`src/documents/file_handling.py:L178`) strips any leading separator — this is *why* an **empty** `tag_list` renders `{tag_list}/{title}` as `invoice.pdf` (not `/invoice.pdf`). Finally, `source_path` is `os.path.join(settings.ORIGINALS_DIR, fname)` (`src/documents/models.py:L223-L231`), and after a move the now-empty source directory is pruned by `delete_empty_directories` (`src/documents/file_handling.py:L23`).

**A successful move emits no log line** — there is no `logger.*` call on the success path in `update_filename_and_move_files`. This is stated explicitly because the *absence* of logs is part of the answer (caveat 4).

### Trigger

Set a tag-aware format, create a `Document` with a real file on disk, then add tags to fire `m2m_changed`. This is the full, ellipsis-free producer run by `/tmp/obs/obs1_relocation.py` (which imports `boot`/`LogCapture` from the preamble bootstrap):

```python
import os
import hashlib
from pathlib import Path
from documents.models import Document, Tag

settings.PAPERLESS_FILENAME_FORMAT = "{tag_list}/{title}"
for _sub in (settings.ORIGINALS_DIR, settings.ARCHIVE_DIR, settings.THUMBNAIL_DIR):
    os.makedirs(_sub, exist_ok=True)
LOGGERS = ["paperless.handlers", "paperless.filehandling"]

# Write the real file first so it exists at originals/invoice.pdf. With 0 tags the
# format renders "invoice.pdf", so the create-time post_save recomputes the SAME
# name and does NOT move the file.
content = b"Invoice #42 - Acme Corp - total due 100.00\n"
checksum = hashlib.md5(content).hexdigest()
Path(os.path.join(settings.ORIGINALS_DIR, "invoice.pdf")).write_bytes(content)
doc = Document.objects.create(title="invoice", mime_type="application/pdf",
                              filename="invoice.pdf", checksum=checksum,
                              storage_type=Document.STORAGE_TYPE_UNENCRYPTED)
print(f"create (0 tags): filename={doc.filename!r}")
print(f"   source_path={doc.source_path}  exists={os.path.isfile(doc.source_path)}")

# +tag 'urgent' -> m2m_changed -> move (LogCapture records the empty log set)
old_path = doc.source_path
with LogCapture(LOGGERS) as cap:
    doc.tags.add(Tag.objects.create(name="urgent"))
    logs1 = list(cap.records)
doc.refresh_from_db()
print(f"+tag 'urgent':   filename={doc.filename!r}")
print(f"   source_path={doc.source_path}  exists={os.path.isfile(doc.source_path)}")
print(f"   old 'invoice.pdf' still there={os.path.isfile(old_path)}")
print(f"   logs during tag-add: {len(logs1)} -> {logs1}")

# +tag 'paid' -> m2m_changed -> move again; 'urgent' dir becomes empty -> pruned
urgent_dir = os.path.dirname(doc.source_path)
with LogCapture(LOGGERS) as cap:
    doc.tags.add(Tag.objects.create(name="paid"))
    logs2 = list(cap.records)
doc.refresh_from_db()
print(f"+tag 'paid':     filename={doc.filename!r}")
print(f"   source_path={doc.source_path}  exists={os.path.isfile(doc.source_path)}")
print(f"   empty 'urgent' dir cleaned={not os.path.isdir(urgent_dir)}")
print(f"   logs during 2nd tag-add: {len(logs2)} -> {logs2}")
```
(This mirrors the canonical move tests in `src/documents/tests/test_file_handling.py`.) The `mime_type="application/pdf"` is needed so the file extension resolves to `.pdf`. Before/after `source_path`, whether the old file still exists, the captured logs, and whether the emptied directory was pruned were all recorded.

### Verbatim evidence (captured 2026-07-01, format `"{tag_list}/{title}"`)

```
create (0 tags): filename='invoice.pdf'
   source_path=/tmp/obs/media/documents/originals/invoice.pdf  exists=True
+tag 'urgent':   filename='urgent/invoice.pdf'
   source_path=/tmp/obs/media/documents/originals/urgent/invoice.pdf  exists=True
   old 'invoice.pdf' still there=False
   logs during tag-add: 0 -> []
+tag 'paid':     filename='paid,urgent/invoice.pdf'
   source_path=/tmp/obs/media/documents/originals/paid,urgent/invoice.pdf  exists=True
   empty 'urgent' dir cleaned=True
   logs during 2nd tag-add: 0 -> []
```

### Reasoning

The `m2m_changed` signal recomputes the target path from the *current* tag set every time tags change. Because the file already exists at the old path, `os.rename` physically relocates it to the newly computed path, and the directory that was emptied by the move is pruned by `delete_empty_directories`. The two tags render as `paid,urgent` — **alphabetically sorted** and comma-joined by `",".join(sorted([tag.name for tag in doc.tags.all()]))` (`src/documents/file_handling.py:L136`), then sanitized by `pathvalidate.sanitize_filename(...)` (`:L135`) — which is why `paid` precedes `urgent` even though `urgent` was added first.

### Sub-part answers

- **Before/after paths ✓** — `originals/invoice.pdf` → `originals/urgent/invoice.pdf` → `originals/paid,urgent/invoice.pdf`. The old file is gone after each move (`old 'invoice.pdf' still there=False`).
- **Log messages during the move ✓** — **there are none.** `logs during tag-add: 0 -> []` (and again `[]` for the second tag-add). A successful move is *silent*; the proof of the move is the changed `source_path` and the vanished old file, not a log line.
- **Directory cleanup ✓** — after the second move, the now-empty `urgent/` directory is removed (`empty 'urgent' dir cleaned=True`) by `delete_empty_directories` (`src/documents/file_handling.py:L23`).

---

## §2 — Rollback safety net

> *"There's supposedly a rollback safety net if something goes wrong during a file move … what actually happens to files when a move fails partway through?"*

### Mechanism

Before moving, `validate_move` guards two failure modes (`src/documents/signals/handlers.py:L295`):

- **Source file missing** → `logger.fatal(f"Document {str(instance)}: File {old_path} has gone.")` (`src/documents/signals/handlers.py:L298`) followed by `raise CannotMoveFilesException()` (`src/documents/signals/handlers.py:L299`).
- **Target already exists** → `logger.warning(... f"Cannot rename file since target path {new_path} already exists.")` (`src/documents/signals/handlers.py:L303-L306`) and `raise CannotMoveFilesException()`.

The recovery is the `except (OSError, DatabaseError, CannotMoveFilesException):` block (`src/documents/signals/handlers.py:L367`). It reverses any partial rename — `os.rename(instance.source_path, old_source_path)` for the original (`src/documents/signals/handlers.py:L376`) and `os.rename(instance.archive_path, old_archive_path)` for the archive (`src/documents/signals/handlers.py:L379`), each guarded by an `os.path.isfile(...)` check — and then restores the in-memory fields: `instance.filename = old_filename` (`src/documents/signals/handlers.py:L393`) and `instance.archive_filename = old_archive_filename` (`src/documents/signals/handlers.py:L394`). Crucially, because the database is only written via the recursion-safe `Document.objects.filter(pk=…).update(...)` (`src/documents/signals/handlers.py:L362-L365`) *after* the renames succeed, a failure before that point leaves the **database row untouched**.

Two facts explain the observed output:

- **`logger.fatal(...)` surfaces at level `CRITICAL`.** In Python's stdlib, `Logger.fatal` is an alias for `Logger.critical`, so the "has gone" message is emitted at level name `CRITICAL` — exactly as captured below.
- **The message text embeds the created date and (once assigned) the correspondent.** `str(instance)` returns `f"{created} {self.correspondent} {self.title}"` when a correspondent *and* title are set, else `f"{created} {self.title}"` (`src/documents/models.py:L217-L220`), where `created` is the ISO date of `self.created` (`src/documents/models.py:L213-L216`). That is why the message reads `Document 2026-07-01 gonedoc` on the first attempt and `Document 2026-07-01 gone gonedoc` on the second (after the correspondent named `gone` is assigned).

### Trigger

**Three** forced failures, each a complete, ellipsis-free producer (run by `/tmp/obs/obs2_rollback.py`, which imports `boot`/`LogCapture` from the preamble bootstrap). They mirror `test_move_file_gone` (`src/documents/tests/test_file_handling.py:L738`), `test_move_file_error` (`:L759`), and — critically for the "partway through" question — `test_move_archive_error` (`:L706-L735`):

```python
import os
from pathlib import Path
from unittest import mock
from documents.models import Document, Correspondent

settings.PAPERLESS_FILENAME_FORMAT = "{correspondent}/{title}"
ORIG = settings.ORIGINALS_DIR
for _sub in (settings.ORIGINALS_DIR, settings.ARCHIVE_DIR, settings.THUMBNAIL_DIR):
    os.makedirs(_sub, exist_ok=True)
ARCH = settings.ARCHIVE_DIR
LOGGERS = ["paperless.handlers", "paperless.filehandling"]

def rel(p):                                   # media-root-relative path, for readable output
    return os.path.relpath(p, settings.MEDIA_ROOT)

# ===== [A] Missing original (mirrors test_move_file_gone, test_file_handling.py:L738) =====
# Archive exists on disk, original does NOT. validate_move finds the original gone and raises
# CannotMoveFilesException -> logger.fatal surfaces at level CRITICAL. The correspondent is
# then assigned + saved to fire the move a 2nd time (2nd CRITICAL, now with correspondent in str()).
print("[A: missing original]  (mirrors test_move_file_gone)")
Path(os.path.join(ARCH, "0000050.pdf")).touch()          # archive only; NO original
with LogCapture(LOGGERS) as cap:
    docA = Document.objects.create(
        mime_type="application/pdf", title="gonedoc",
        filename="0000050.pdf", archive_filename="0000050.pdf",
        checksum="A50", archive_checksum="B50",
    )                                                    # post_save fires -> move fails (1st CRITICAL)
    docA.correspondent = Correspondent.objects.create(name="gone")
    docA.save()                                          # fires again -> 2nd CRITICAL
    logsA = list(cap.records)
dbA = Document.objects.get(pk=docA.pk)
print(f"   DB filename unchanged={dbA.filename!r}")
print(f"   archive left in place={os.path.isfile(os.path.join(ARCH, '0000050.pdf'))}")
print(f"   logs: {logsA}")

# ===== [B] OSError during the ORIGINAL rename (mirrors test_move_file_error, :L759) =====
# Patch os.rename to raise on the original move. It raises BEFORE anything moved, so the except
# block finds nothing to reverse -> silent recovery (no log), DB row untouched.
print("[B: OSError during original rename]  (mirrors test_move_file_error)")
Path(os.path.join(ORIG, "0000060.pdf")).touch()
Path(os.path.join(ARCH, "0000060.pdf")).touch()
with mock.patch("documents.signals.handlers.os.rename") as m:
    def fake_rename_b(src, dst):
        if "originals" in str(src):
            raise OSError("forced original failure")
        os.remove(src); Path(dst).touch()
    m.side_effect = fake_rename_b
    with LogCapture(LOGGERS) as cap:
        docB = Document.objects.create(
            mime_type="application/pdf", title="bdoc",
            filename="0000060.pdf", archive_filename="0000060.pdf",
            checksum="A60", archive_checksum="B60",
        )
        logsB = list(cap.records)
    called_b = m.called
dbB = Document.objects.get(pk=docB.pk)
print(f"   os.rename called={called_b}")
print(f"   original restored in place={os.path.isfile(os.path.join(ORIG, '0000060.pdf'))}")
print(f"   archive untouched in place={os.path.isfile(os.path.join(ARCH, '0000060.pdf'))}")
print(f"   DB filename unchanged={dbB.filename!r}")
print(f"   logs: {logsB}")

# ===== [C] Archive rename FAILS *after* the ORIGINAL move SUCCEEDED — the true "partway =====
# through" case (mirrors test_move_archive_error, :L706-L735). fake_rename_c performs the ORIGINAL
# move (os.remove + touch) but raises on the ARCHIVE move; the except block then reverses the
# ALREADY-COMPLETED original move (handlers.py:L376). `seq` records the os.rename call order.
print("[C: archive rename fails AFTER original move succeeded]  (mirrors test_move_archive_error)")
Path(os.path.join(ORIG, "0000001.pdf")).touch()
Path(os.path.join(ARCH, "0000001.pdf")).touch()
seq = []
with mock.patch("documents.signals.handlers.os.rename") as m:
    def fake_rename_c(src, dst):
        which = "archive" if "archive" in str(src) else "original"
        if "archive" in str(src):
            seq.append((which, "RAISE", rel(src), rel(dst)))
            raise OSError("forced archive failure")
        os.remove(src); Path(dst).touch()
        seq.append((which, "OK", rel(src), rel(dst)))
    m.side_effect = fake_rename_c
    with LogCapture(LOGGERS) as cap:
        docC = Document.objects.create(
            mime_type="application/pdf", title="my_doc",
            filename="0000001.pdf", archive_filename="0000001.pdf",
            checksum="A01", archive_checksum="B01",
        )
        logsC = list(cap.records)
dbC = Document.objects.get(pk=docC.pk)
new_orig = os.path.join(ORIG, "none", "my_doc.pdf")
new_arch = os.path.join(ARCH, "none", "my_doc.pdf")
print("   os.rename call sequence (verb, outcome, src, dst):")
for step in seq:
    print(f"      {step}")
print(f"   original moved back to 0000001.pdf={os.path.isfile(os.path.join(ORIG, '0000001.pdf'))}")
print(f"   new original path emptied ({rel(new_orig)})={os.path.isfile(new_orig)}")
print(f"   archive untouched at 0000001.pdf={os.path.isfile(os.path.join(ARCH, '0000001.pdf'))}")
print(f"   new archive path never created ({rel(new_arch)})={os.path.isfile(new_arch)}")
print(f"   DB filename unchanged={dbC.filename!r}  archive_filename unchanged={dbC.archive_filename!r}")
print(f"   logs: {logsC}")
```
Each scenario prints, inline, whether the DB `filename`/`archive_filename` changed, whether the files stayed at (or returned to) their original locations, the `os.rename` call order (scenario C), and the captured logs — so the evidence block below is the verbatim stdout of exactly this script.

### Verbatim evidence (captured 2026-07-01, format `"{correspondent}/{title}"`)

```
[A: missing original]  (mirrors test_move_file_gone)
   DB filename unchanged='0000050.pdf'
   archive left in place=True
   logs: [('paperless.handlers', 'CRITICAL', 'Document 2026-07-01 gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.'), ('paperless.handlers', 'CRITICAL', 'Document 2026-07-01 gone gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.')]
[B: OSError during original rename]  (mirrors test_move_file_error)
   os.rename called=True
   original restored in place=True
   archive untouched in place=True
   DB filename unchanged='0000060.pdf'
   logs: []
[C: archive rename fails AFTER original move succeeded]  (mirrors test_move_archive_error)
   os.rename call sequence (verb, outcome, src, dst):
      ('original', 'OK', 'documents/originals/0000001.pdf', 'documents/originals/none/my_doc.pdf')
      ('archive', 'RAISE', 'documents/archive/0000001.pdf', 'documents/archive/none/my_doc.pdf')
      ('original', 'OK', 'documents/originals/none/my_doc.pdf', 'documents/originals/0000001.pdf')
   original moved back to 0000001.pdf=True
   new original path emptied (documents/originals/none/my_doc.pdf)=False
   archive untouched at 0000001.pdf=True
   new archive path never created (documents/archive/none/my_doc.pdf)=False
   DB filename unchanged='0000001.pdf'  archive_filename unchanged='0000001.pdf'
   logs: []
```

### Reasoning

The rollback **genuinely recovers** — it does not merely "promise to." In *all three* scenarios the database row is unchanged (`DB filename unchanged`) and the files remain at (or are returned to) their original locations. The three scenarios probe progressively deeper failure points:

- **[A] aborts *before* any rename.** `validate_move` finds the original gone and raises `CannotMoveFilesException`, which the `except` block catches (`src/documents/signals/handlers.py:L367`); no file ever moved.
- **[B] fails *on* the original rename, before it completes.** The patched `os.rename` raises `OSError` the instant it is called for the original, so — like [A] — nothing actually moved; the `except` block's `os.path.isfile`-guarded reversals are no-ops and recovery is silent.
- **[C] fails *after* the original rename has already completed** — the literal "move fails partway through" case. This is the one that actually exercises the reversal branch, and the observed `os.rename` call sequence proves it: the original moves forward (`('original', 'OK', documents/originals/0000001.pdf, documents/originals/none/my_doc.pdf)`), the archive move raises (`('archive', 'RAISE', …)`), and then the `except` block **moves the original back** (`('original', 'OK', documents/originals/none/my_doc.pdf, documents/originals/0000001.pdf)`) via `os.rename(instance.source_path, old_source_path)` (`src/documents/signals/handlers.py:L376`). The end state confirms full recovery: `original moved back to 0000001.pdf=True`, `new original path emptied=False`, `archive untouched at 0000001.pdf=True`, and both `DB filename`/`archive_filename` unchanged. The archive reversal (`:L379`) is correctly skipped because its `os.path.isfile` guard sees the archive never reached its new path. So the safety net is real, not aspirational: a completed rename **is** reversed when a later step fails.

Scenario [A] shows **two** `CRITICAL` records because the moving-save fires **twice**: once when the document is first created (via `post_save`, before the correspondent is set → `Document 2026-07-01 gonedoc`), and once on the correspondent-assignment `save()` (→ `Document 2026-07-01 gone gonedoc`). Each failed attempt logs exactly once. This is a direct consequence of `update_filename_and_move_files` being registered on `post_save` *and* `m2m_changed` (`src/documents/signals/handlers.py:L310-L311`).

### Sub-part answers

- **Does it truly recover? ✓** Yes. DB `filename`/`archive_filename` are unchanged in all three scenarios; files are restored to (or never left) their original locations.
- **What happens on partial failure (a move that fails "partway through")? ✓** — **directly observed in scenario [C].** After the original `os.rename` has *already completed*, the archive `os.rename` raises `OSError`; the `except` block then reverses the completed original move via `os.rename(instance.source_path, old_source_path)` (`src/documents/signals/handlers.py:L376`) — the captured `os.rename` sequence shows the forward move (`original OK`) *and* the reverse move (`original OK` back to `0000001.pdf`) explicitly — while the archive reversal (`:L379`) is skipped by its `os.path.isfile` guard because the archive never reached its new path. The in-memory `filename`/`archive_filename` are restored (`:L393-L394`), and the DB write never happens because it only runs *after* both renames succeed. Net observed state: `original moved back to 0000001.pdf=True`, `archive untouched at 0000001.pdf=True`, `DB filename`/`archive_filename` unchanged.
- **The exact `CRITICAL` "has gone" message ✓** — `Document 2026-07-01 gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.` at level `CRITICAL` (from `logger.fatal`, `src/documents/signals/handlers.py:L298`).
- **The silent recovery paths ✓** — scenarios [B] *and* [C] emit **no** log (`logs: []`); their proof is the filesystem/DB state (`original restored in place=True` for [B]; `original moved back to 0000001.pdf=True` for [C]; `DB filename unchanged` for both). The absence of a log is itself the finding (caveat 4).
- **Run-date caveat** — `2026-07-01` is *this run's* date (caveat 3); a run on another day prints that day's date in the message.

---

## §3 — Classifier training: skip vs. retrain

> *"Sometimes training finishes instantly, other times it takes much longer. I want to trigger both scenarios and see the actual log messages that explain why training was skipped versus why it proceeded with full retraining, including whatever hash or checksum the system uses to detect changes."*

### Mechanism

`DocumentClassifier.train()` (`src/documents/classifier.py:L115`) computes a change-detection digest with **SHA-1**: `m = hashlib.sha1()` (`src/documents/classifier.py:L124`). It iterates over every non-inbox document and feeds four things into the hash:

- the preprocessed content — `m.update(preprocessed_content.encode("utf-8"))` (`src/documents/classifier.py:L129`);
- the `document_type` PK when its matching algorithm is `MATCH_AUTO`, else `-1`, as 4 signed little-endian bytes — `m.update(y.to_bytes(4, "little", signed=True))` (`src/documents/classifier.py:L136`);
- the `correspondent` PK under the same rule (`src/documents/classifier.py:L143`);
- each **sorted** auto-matching `Tag` PK (`src/documents/classifier.py:L155`).

The digest is then `new_data_hash = m.digest()` (`src/documents/classifier.py:L161`) — **20 raw bytes** (SHA-1 output length). The instant-skip is this early return:

```python
if self.data_hash and new_data_hash == self.data_hash:
    return False
```
(`src/documents/classifier.py:L163-L164`). If the freshly computed digest equals the stored one, `train()` returns `False` immediately — *before* any model fitting. Notably, the scikit-learn imports are **lazy** — `from sklearn.feature_extraction.text import CountVectorizer` and friends live *inside* `train()` at `src/documents/classifier.py:L188-L190`, i.e. **after** the hash+skip check. This means the change-detection hash is computed with no scikit-learn involvement and is therefore **version-independent**. (A separate lazy scikit-learn import, `from sklearn.utils.multiclass import type_of_target`, appears at `src/documents/classifier.py:L274`, but that one lives inside `predict_tags()` — not `train()` — so it plays no part in the training skip/retrain decision.)

The task wrapper `train_classifier()` (`src/documents/tasks.py:L48`) is a **no-op unless** at least one `Tag`, `DocumentType`, or `Correspondent` uses automatic matching:

```python
if (
    not Tag.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    and not DocumentType.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
    and not Correspondent.objects.filter(matching_algorithm=Tag.MATCH_AUTO).exists()
):
    return
```
(`src/documents/tasks.py:L49-L55`). When training does produce a new model it logs at **INFO** and saves — `logger.info("Saving updated classifier model to {}...".format(settings.MODEL_FILE))` (`src/documents/tasks.py:L65`) then `classifier.save()` (`src/documents/tasks.py:L67`); when nothing changed it logs at **DEBUG** — `logger.debug("Training data unchanged.")` (`src/documents/tasks.py:L69`). The model path is `MODEL_FILE = os.path.join(DATA_DIR, "classification_model.pickle")` (`src/paperless/settings.py:L74`).

### Trigger

Seed at least one auto-matching object (so the task is not a no-op), plus documents with distinct content, then exercise the classifier. This mirrors `testDatasetHashing` (`src/documents/tests/test_classifier.py:L137-L142`) and `testSaveClassifier` (`:L168-L178`). **The critical detail:** the instant-skip (`train()` → `False`) is only reachable on **one** classifier instance whose `data_hash` is already populated — `DocumentClassifier.__init__` (`src/documents/classifier.py:L65`) sets `self.data_hash = None` (`:L68`), and the skip branch is `if self.data_hash and new_data_hash == self.data_hash` (`:L163-L164`). A *second, fresh* `DocumentClassifier()` would start with `data_hash = None` and therefore always return `True`; so — exactly as `testDatasetHashing` does — `train()` must be called **twice on the same instance**. This is the full, ellipsis-free producer run by the observation harness (`/tmp/obs/obs3_classifier.py`, imports `boot`/`LogCapture` from the shared bootstrap in the preamble):

```python
import os
from documents.models import Correspondent, DocumentType, Tag, Document
from documents.classifier import DocumentClassifier
from documents import tasks

# Seed: at least one MATCH_AUTO object so train_classifier() is not a no-op
c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
t1 = Tag.objects.create(name="t1", matching_algorithm=Tag.MATCH_AUTO)
dt = DocumentType.objects.create(name="dt", matching_algorithm=DocumentType.MATCH_AUTO)
doc1 = Document.objects.create(title="doc1", content="this is a document from c1",
                               correspondent=c1, document_type=dt,
                               checksum="A", mime_type="application/pdf")
doc2 = Document.objects.create(title="doc2", content="this is another document, but from c2",
                               checksum="B", mime_type="application/pdf")
doc1.tags.add(t1)

# One instance, two calls: #1 fits (True) and sets self.data_hash; #2 skips (False)
clf = DocumentClassifier()
r1 = clf.train()            # data_hash is None -> fits models -> True; sets data_hash (L247)
h = clf.data_hash           # 20 raw bytes (SHA-1)
r2 = clf.train()            # data_hash now set & equal -> early return False (L163-164)
print(f"DocumentClassifier().train() #1 -> {r1} | data_hash(sha1 hex)= {h.hex()} | len(bytes)= {len(h)}")
print(f"same instance .train() #2 (no data change) -> {r2}")

# The task wrapper: fresh(INFO) / unchanged(DEBUG) / mutated(INFO). Each call is wrapped in
# LogCapture and its records printed as the [level][logger] lines quoted in the evidence below.
LOGGERS = ["paperless.classifier", "paperless.tasks"]

print("-- tasks.train_classifier() call #1 (fresh) --")
with LogCapture(LOGGERS) as cap:
    tasks.train_classifier()            # fresh model -> INFO save line
    for n, lvl, msg in cap.records:
        print(f"   [{lvl}][{n}] {msg}")
print(f"   MODEL_FILE on disk = {os.path.isfile(settings.MODEL_FILE)} -> {settings.MODEL_FILE}")

print("-- tasks.train_classifier() call #2 (unchanged data) --")
with LogCapture(LOGGERS) as cap:
    tasks.train_classifier()            # unchanged -> DEBUG "Training data unchanged."
    for n, lvl, msg in cap.records:
        print(f"   [{lvl}][{n}] {msg}")

print("-- mutate training set (add doc3) then call #3 --")
Document.objects.create(title="doc3", content="a third completely different document about invoices",
                        checksum="C", mime_type="application/pdf")
with LogCapture(LOGGERS) as cap:
    tasks.train_classifier()            # changed -> INFO save line
    for n, lvl, msg in cap.records:
        print(f"   [{lvl}][{n}] {msg}")
```

### Verbatim evidence (captured 2026-07-01; SHA-1 reproducible on a fresh DB with the seed above)

```
DocumentClassifier().train() #1 -> True | data_hash(sha1 hex)= 84b43315ab3567f266bea8387e7d585ea6b33a9b | len(bytes)= 20
same instance .train() #2 (no data change) -> False
-- tasks.train_classifier() call #1 (fresh) --
   [DEBUG][paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
   [DEBUG][paperless.classifier] Gathering data from database...
   [DEBUG][paperless.classifier] 2 documents, 1 tag(s), 1 correspondent(s), 1 document type(s).
   [DEBUG][paperless.classifier] Vectorizing data...
   [DEBUG][paperless.classifier] Training tags classifier...
   [DEBUG][paperless.classifier] Training correspondent classifier...
   [DEBUG][paperless.classifier] Training document type classifier...
   [INFO][paperless.tasks] Saving updated classifier model to /tmp/obs/data/classification_model.pickle...
   MODEL_FILE on disk = True -> /tmp/obs/data/classification_model.pickle
-- tasks.train_classifier() call #2 (unchanged data) --
   [DEBUG][paperless.classifier] Gathering data from database...
   [DEBUG][paperless.tasks] Training data unchanged.
-- mutate training set (add doc3) then call #3 --
   [DEBUG][paperless.classifier] Gathering data from database...
   [DEBUG][paperless.classifier] 3 documents, 1 tag(s), 1 correspondent(s), 1 document type(s).
   [DEBUG][paperless.classifier] Vectorizing data...
   [DEBUG][paperless.classifier] Training tags classifier...
   [DEBUG][paperless.classifier] Training correspondent classifier...
   [DEBUG][paperless.classifier] Training document type classifier...
   [INFO][paperless.tasks] Saving updated classifier model to /tmp/obs/data/classification_model.pickle...
```

The line `DocumentClassifier().train() #1 -> True` followed by `same instance .train() #2 (no data change) -> False` is the crux: **the same `clf` object returns `True` then `False`.** The first call finds `self.data_hash is None`, fits the models, and stores the digest (`src/documents/classifier.py:L247`); the second call finds the freshly-computed digest equal to the stored one and takes the early `return False` (`:L163-L164`). The SHA-1 `data_hash` observed here is `84b43315ab3567f266bea8387e7d585ea6b33a9b` (20 bytes). Because that digest is derived only from the preprocessed content plus the auto `Correspondent`/`DocumentType`/`Tag` primary keys (`src/documents/classifier.py:L124-L161`) — all fixed by the fully-specified seed above — it **reproduces byte-for-byte** across repeated runs against a *fresh* database at this commit (verified stable over three independent runs). Per caveat 2, a *non-fresh* database assigns different PKs and therefore yields a different digest; per caveat 1, a different seed (different content/PKs) yields a different digest — which is exactly the change-detection property this section demonstrates.

### Reasoning

"Instant" training is the early `return False` (`src/documents/classifier.py:L163-L164`): when the freshly computed SHA-1 equals the stored `data_hash`, `train()` bails out before touching scikit-learn, so it finishes in microseconds. "Full retrain" happens whenever the digest differs — a new/edited/removed document, or a changed auto type/correspondent/tag — because the mismatch falls through the skip check and proceeds to vectorize and fit the models. The **INFO vs. DEBUG** split is precisely how the task announces "I saved a new model" (`Saving updated classifier model to …`) versus "nothing to do" (`Training data unchanged.`). The `2 documents, 1 tag(s), 1 correspondent(s), 1 document type(s).` line on call #1 versus `3 documents, …` on call #3 confirms the retrain saw the mutated data set.

### Sub-part answers

- **Instant-skip vs. full-retrain triggers ✓** — skip when `new_data_hash == self.data_hash` (`src/documents/classifier.py:L163-L164`); retrain on any digest change (new doc `doc3` on call #3).
- **The log message for each ✓** — retrain → **INFO** `Saving updated classifier model to /tmp/obs/data/classification_model.pickle...` (`src/documents/tasks.py:L65`); skip → **DEBUG** `Training data unchanged.` (`src/documents/tasks.py:L69`).
- **The exact hash used ✓** — a **20-byte SHA-1** `data_hash` (`src/documents/classifier.py:L124,L161`). Observed here as hex `84b43315ab3567f266bea8387e7d585ea6b33a9b`, reproducible byte-for-byte against a fresh database with the seed shown above (and attributed to *this* seed per caveats 1–2). This is **SHA-1**, distinct from the MD5 used for file integrity elsewhere.
- **The auto-matching precondition ✓** — `train_classifier()` returns immediately unless some `Tag`/`DocumentType`/`Correspondent` uses `MATCH_AUTO` (`src/documents/tasks.py:L49-L55`); the seed includes a `MATCH_AUTO` correspondent, tag, and document type so the task is not a no-op.

---

## §4 — Duplicate detection

> *"Two files that look completely different can still be rejected as duplicates, and I want to see the actual checksums being compared at that moment of judgment."*

### Mechanism

`Consumer.pre_check_duplicate()` (`src/documents/consumer.py:L102`) reads the incoming file and computes an **MD5** hex digest of its bytes:

```python
with open(self.path, "rb") as f:
    checksum = hashlib.md5(f.read()).hexdigest()
```
(`src/documents/consumer.py:L103-L104`). It then asks the database whether *any* stored document has that digest as either its original or its archive checksum:

```python
if Document.objects.filter(
    Q(checksum=checksum) | Q(archive_checksum=checksum),
).exists():
```
(`src/documents/consumer.py:L105-L107`). On a match — after optionally deleting the incoming file when `settings.CONSUMER_DELETE_DUPLICATES` is set (`src/documents/consumer.py:L108-L109`) — it calls `self._fail(MESSAGE_DOCUMENT_ALREADY_EXISTS, f"Not consuming {self.filename}: It is a duplicate.")` (`src/documents/consumer.py:L110-L112`). `_fail` (`src/documents/consumer.py:L78`) logs at **ERROR** — `self.log("error", log_message or message, …)` (`src/documents/consumer.py:L80`) — and raises `ConsumerError(f"{self.filename}: {log_message or message}")` (`src/documents/consumer.py:L81`). The constant is `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"` (`src/documents/consumer.py:L37`). Because the consumer's `logging_name = "paperless.consumer"` (`src/documents/consumer.py:L54`), the ERROR surfaces under that logger.

The decisive point: identity is judged purely on the **MD5 of the file content**, never the filename. Two files with completely different *names* but identical *bytes* therefore collide.

### Trigger

Persist a `Document` whose `checksum` is the MD5 of a known byte string, then run the pre-check against a *differently-named* file holding the *identical* bytes — mirroring `test_delete_duplicate` (`src/documents/tests/test_consumer.py:L578`) / `test_no_delete_duplicate` (`:L597`):

```python
import os
import hashlib
from unittest import mock
from documents.models import Document
from documents.consumer import Consumer, ConsumerError

content = b"Invoice #42 - Acme Corp - total due 100.00\n"
md5 = hashlib.md5(content).hexdigest()
print(f"content MD5 = {md5}")

# Stored document carries that MD5 as its checksum; NO filename -> no stray move signal.
stored = Document.objects.create(title="stored", checksum=md5, mime_type="application/pdf")
print(f"stored Document pk={stored.pk}  checksum={stored.checksum}")

# A differently-named file holding the SAME bytes:
incoming = "/tmp/incoming_copy.pdf"
with open(incoming, "wb") as f:
    f.write(content)
incoming_md5 = hashlib.md5(open(incoming, "rb").read()).hexdigest()

consumer = Consumer()
with mock.patch.object(Consumer, "_send_progress"):     # tests mock this (test_consumer.py:L290-291)
    consumer.renew_logging_group()
    consumer.path = incoming
    consumer.filename = "incoming_copy.pdf"
    matches = Document.objects.filter(checksum=incoming_md5).exists()
    print(f"incoming file={incoming}  md5={incoming_md5}  matches_stored={matches}")
    with LogCapture(["paperless.consumer"]) as cap:
        try:
            consumer.pre_check_duplicate()              # -> raises ConsumerError
        except ConsumerError as e:
            print(f"RESULT: ConsumerError raised -> {str(e)!r}")
        for n, lvl, msg in cap.records:
            print(f"   [{lvl}][{n}] {msg}")
os.remove(incoming)
```

### Verbatim evidence (fresh DB, PK=1; content `b"Invoice #42 - Acme Corp - total due 100.00\n"`)

```
content MD5 = a59701aecccc6903a4019d07477c9a51
stored Document pk=1  checksum=a59701aecccc6903a4019d07477c9a51
incoming file=/tmp/incoming_copy.pdf  md5=a59701aecccc6903a4019d07477c9a51  matches_stored=True
RESULT: ConsumerError raised -> 'incoming_copy.pdf: Not consuming incoming_copy.pdf: It is a duplicate.'
   [ERROR][paperless.consumer] Not consuming incoming_copy.pdf: It is a duplicate.
```

The MD5 `a59701aecccc6903a4019d07477c9a51` was independently verified: `python3 -c "import hashlib; print(hashlib.md5(b'Invoice #42 - Acme Corp - total due 100.00\n').hexdigest())"` prints the same value. Per caveat 1, a *different* byte string yields a different digest — quote the digest that matches your bytes.

### Reasoning

Because the "moment of judgment" is the comparison of the **content MD5** (`src/documents/consumer.py:L104`) against stored `checksum`/`archive_checksum` values (`src/documents/consumer.py:L106`), the *name* of the file is irrelevant. Renaming a file does not change its bytes, so the digest is unchanged, the `Q(checksum=…) | Q(archive_checksum=…)` filter still matches, and the incoming copy is rejected. This is exactly the "two files that look completely different" case: they *look* different only by name, but they are byte-identical, so they share one MD5 and are correctly judged duplicates.

### Sub-part answers

- **How differently-named files are still duplicates ✓** — detection uses the **content MD5** (`src/documents/consumer.py:L104`), not the filename; identical bytes → identical digest → DB match.
- **The actual checksums compared ✓** — both sides are `a59701aecccc6903a4019d07477c9a51` for the content above (`matches_stored=True`). This is the exact digest at the moment of judgment; it is byte-dependent (caveat 1).
- **The raised error + ERROR log ✓** — `ConsumerError` with message `incoming_copy.pdf: Not consuming incoming_copy.pdf: It is a duplicate.` (`src/documents/consumer.py:L81`), and the `[ERROR][paperless.consumer] Not consuming incoming_copy.pdf: It is a duplicate.` log line (`src/documents/consumer.py:L80`).

---

## §5 — Sanity checker: healthy vs. mismatch

> *"What does its output actually look like when an archive is healthy versus when something has gone wrong, and can I see the specific hash values it compares when it discovers a mismatch?"*

### Mechanism

`check_sanity()` (`src/documents/sanity_checker.py:L49`) walks `settings.MEDIA_ROOT` (`src/documents/sanity_checker.py:L53`) and, for every document, recomputes the **MD5** of the on-disk original and archive and compares each to the stored value. For the original:

```python
checksum = hashlib.md5(f.read()).hexdigest()          # L83 (inside `with doc.source_file as f:`)
# wrapped in try/except OSError/else; the else-branch does the compare (L86-L91):
if not checksum == doc.checksum:
    messages.error(
        f"Checksum mismatch of document {doc.pk}. "
        f"Stored: {doc.checksum}, actual: {checksum}.",
    )
```
(`src/documents/sanity_checker.py:L83`, `:L87-L91`; this is a source excerpt illustrating the compare, not a runnable producer.) The archive is handled the same way, comparing against `doc.archive_checksum` and producing `Checksum mismatch of archived document {doc.pk}. …` (`src/documents/sanity_checker.py:L112,L118-L124`).

Results are accumulated in a `SanityCheckMessages` object whose helpers stamp a logging level on each message: `error()` → `logging.ERROR` (40) (`src/documents/sanity_checker.py:L14-L15`), `warning()` → `logging.WARNING` (30) (`:L17-L18`), `info()` → `logging.INFO` (20) (`:L20-L21`). Emission happens in `log_messages()` (`src/documents/sanity_checker.py:L23`): if there are **no** messages it logs `logger.info("Sanity checker detected no issues.")` (`src/documents/sanity_checker.py:L27`); otherwise it logs each message at its own level via `logger.log(msg["level"], msg["message"])` (`:L30`). Convenience predicates `has_error` (`:L38-L39`) and `has_warning` (`:L41-L42`) report whether any ERROR/WARNING message is present.

### Trigger

Run `check_sanity()` twice under the fresh `/tmp/obs` media root — once where the file matches its stored checksum (HEALTHY), once where the on-disk bytes differ from the stored digest (CORRUPT). This is the full, ellipsis-free producer run by `/tmp/obs/obs5_sanity.py` (imports `boot`/`LogCapture` from the preamble bootstrap); it seeds a document with a thumbnail and non-empty content and **no** archive, so the only variable is the original's checksum — mirroring the checksum-mismatch cases in `src/documents/tests/test_sanity_check.py`:

```python
import os, hashlib
from pathlib import Path
from documents.models import Document
from documents.sanity_checker import check_sanity

for d in (settings.ORIGINALS_DIR, settings.THUMBNAIL_DIR, settings.ARCHIVE_DIR):
    os.makedirs(d, exist_ok=True)

B1 = b"ORIGINAL bytes v1"                         # the "true" original content
B2 = b"TAMPERED bytes v2 (longer, different)"     # what a corrupted file holds instead

# HEALTHY: original on disk matches the stored checksum; thumbnail present; has content; no archive.
doc = Document.objects.create(
    title="s1", content="present", checksum=hashlib.md5(B1).hexdigest(),
    filename="0000001.pdf", mime_type="application/pdf",
)
doc.refresh_from_db()
Path(doc.source_path).parent.mkdir(parents=True, exist_ok=True)
open(doc.source_path, "wb").write(B1)             # disk == what the DB believes -> match
Path(doc.thumbnail_path).parent.mkdir(parents=True, exist_ok=True)
open(doc.thumbnail_path, "wb").write(b"thumb")    # thumbnail present -> no thumbnail error
with LogCapture(["paperless.sanity_checker"]) as cap:
    m = check_sanity(); m.log_messages()
print(f"[HEALTHY] len={len(m)} has_error={m.has_error()} has_warning={m.has_warning()}  "
      f"logs={[(l,msg) for _,l,msg in cap.records]}")

# CORRUPT: the DB still records md5(B1), but the file on disk now holds B2 (different bytes).
open(doc.source_path, "wb").write(B2)
m = check_sanity()
print(f"[CORRUPT] len={len(m)} has_error={m.has_error()}  "
      f"stored={doc.checksum} actual={hashlib.md5(B2).hexdigest()}")
for msg in m:
    print(f"   [level {msg['level']}] {msg['message']}")
```

### Verbatim evidence (fresh media root; stored=`md5(b"ORIGINAL bytes v1")`, actual=`md5(b"TAMPERED bytes v2 (longer, different)")`)

```
[HEALTHY] len=0 has_error=False has_warning=False  logs=[('INFO', 'Sanity checker detected no issues.')]
[CORRUPT] len=1 has_error=True  stored=23b1bc861edb6f698460ca6a7e15f06f actual=fccd9583fa257082613b93f8cf4bbc71
   [level 40] Checksum mismatch of document 1. Stored: 23b1bc861edb6f698460ca6a7e15f06f, actual: fccd9583fa257082613b93f8cf4bbc71.
```

Both MD5s were independently verified: `md5(b"ORIGINAL bytes v1") = 23b1bc861edb6f698460ca6a7e15f06f` and `md5(b"TAMPERED bytes v2 (longer, different)") = fccd9583fa257082613b93f8cf4bbc71`. Per caveat 1, these are byte-dependent; different content yields different digests.

### Reasoning

The checker is a straightforward **fresh MD5 recomputation** compared against the stored value. When the on-disk bytes match what the database recorded, the comparison `checksum == doc.checksum` at `src/documents/sanity_checker.py:L87` is `True` (so the `if not …` branch is skipped), no message is appended, and `log_messages()` takes the empty branch and emits the single INFO line `Sanity checker detected no issues.` (`:L27`). When the bytes differ (the "tampered" file), the comparison is `False`, one ERROR message is appended naming the document PK and **both** digests (`:L89-L90`), and it is emitted at level 40. `has_error` is `True` (`:L38-L39`), and `has_warning` is `False` because the mismatch is an ERROR, not a WARNING. The level `40` printed above is `logging.ERROR` numerically.

### Sub-part answers

- **Healthy output ✓** — `Sanity checker detected no issues.` at level INFO (`src/documents/sanity_checker.py:L27`); `len=0 has_error=False has_warning=False`.
- **Unhealthy output ✓** — one ERROR (level 40): `Checksum mismatch of document 1. Stored: 23b1bc861edb6f698460ca6a7e15f06f, actual: fccd9583fa257082613b93f8cf4bbc71.` (`src/documents/sanity_checker.py:L89-L90`); `len=1 has_error=True`.
- **The specific hash values compared ✓** — stored `23b1bc861edb6f698460ca6a7e15f06f` vs. actual `fccd9583fa257082613b93f8cf4bbc71` (both **MD5**, byte-dependent per caveat 1). The document PK is `1` because of the fresh-DB reset (caveat 2).

---

## §6 — Orphaned / ghost files

> *"Whether orphaned files really linger in the media folder and what the system actually reports when it finds them."*

### Mechanism

`check_sanity()` (`src/documents/sanity_checker.py:L49`) builds the set of all files under `settings.MEDIA_ROOT` by walking it with `os.walk(settings.MEDIA_ROOT)` (`src/documents/sanity_checker.py:L53`) and removes the lock file from that candidate set (`src/documents/sanity_checker.py:L57-L59`). As it processes each document it accounts for that document's thumbnail, original, and archive (removing them from the candidate set). Whatever files remain in the set at the end are **unaccounted for**, and each is reported:

```python
for extra_file in present_files:
    messages.warning(f"Orphaned file in media dir: {extra_file}")
```
(`src/documents/sanity_checker.py:L130-L131`). Because this uses `messages.warning(...)`, the message carries level `logging.WARNING` (**30**) (`src/documents/sanity_checker.py:L17-L18`). Critically, the code **reports** the orphan — it never deletes it — so the file stays on disk.

### Trigger

Under the fresh `/tmp/obs` media root, seed one healthy `Document` (so the *only* finding is the orphan), create a file that no `Document` references, then run the checker — mirroring `test_orphaned_file` (`src/documents/tests/test_sanity_check.py:L181-L188`). This is the full, ellipsis-free producer run by `/tmp/obs/obs6_orphan.py` (imports `boot`/`LogCapture` from the preamble bootstrap):

```python
import os, hashlib
from pathlib import Path
from documents.models import Document
from documents.sanity_checker import check_sanity

for d in (settings.ORIGINALS_DIR, settings.THUMBNAIL_DIR, settings.ARCHIVE_DIR):
    os.makedirs(d, exist_ok=True)

# A healthy document so the ONLY finding is the orphan (mirrors test_orphaned_file setup).
B = b"real doc bytes"
doc = Document.objects.create(
    title="o1", content="present", checksum=hashlib.md5(B).hexdigest(),
    filename="0000001.pdf", mime_type="application/pdf",
)
doc.refresh_from_db()
Path(doc.source_path).parent.mkdir(parents=True, exist_ok=True)
open(doc.source_path, "wb").write(B)
Path(doc.thumbnail_path).parent.mkdir(parents=True, exist_ok=True)
open(doc.thumbnail_path, "wb").write(b"thumb")

orphan = Path(settings.ORIGINALS_DIR, "orphaned")
orphan.touch()                                    # a file no Document accounts for
with LogCapture(["paperless.sanity_checker"]) as cap:
    m = check_sanity(); m.log_messages()
print(f"[ORPHAN] len={len(m)} has_error={m.has_error()} has_warning={m.has_warning()}")
for i, msg in enumerate(m):
    print(f"   messages[{i}]={{'level': {msg['level']}, 'message': {msg['message']!r}}}")
for n, lvl, msg in cap.records:
    print(f"   [{lvl}] {msg}")
print(f"   orphan file still on disk={orphan.exists()}")
```

### Verbatim evidence (fresh `/tmp/obs` media root)

```
[ORPHAN] len=1 has_error=False has_warning=True
   messages[0]={'level': 30, 'message': 'Orphaned file in media dir: /tmp/obs/media/documents/originals/orphaned'}
   [WARNING] Orphaned file in media dir: /tmp/obs/media/documents/originals/orphaned
   orphan file still on disk=True
```

### Reasoning

The sanity checker's job is to **report** integrity problems, not to repair them. An unaccounted file is surfaced as a single WARNING (level 30) via `messages.warning(...)` (`src/documents/sanity_checker.py:L131`) and left exactly where it is — confirmed by `orphan file still on disk=True` after the run. So yes: orphaned files genuinely linger; the system tells you about them but does not remove them. (The message embeds the file's absolute path, whose prefix is the media root — here the fixed `/tmp/obs/media` from the preamble bootstrap; on another host the prefix differs while the intrinsic part, `documents/originals/orphaned`, is stable — caveat 2.)

### Sub-part answers

- **Do orphans linger? ✓** Yes — the file is reported, **not** removed (`orphan file still on disk=True`); the code has no unlink on this path (`src/documents/sanity_checker.py:L130-L131`).
- **What does the system report? ✓** A single **WARNING** at level **30**: `Orphaned file in media dir: /tmp/obs/media/documents/originals/orphaned` (`src/documents/sanity_checker.py:L131`); `has_warning=True`, `has_error=False`.

---

## Cross-cutting findings

These themes recur across the six behaviors and are worth stating once, explicitly.

### Hash algorithms differ by purpose — MD5 for file integrity, SHA-1 for classifier change detection

- **MD5** is used for *document/archive content integrity*: the consumer's duplicate check `hashlib.md5(f.read()).hexdigest()` (`src/documents/consumer.py:L104`) and the sanity checker's recomputations for original (`src/documents/sanity_checker.py:L83`) and archive (`:L112`).
- **SHA-1** is used for the *classifier change-detection* digest: `hashlib.sha1()` (`src/documents/classifier.py:L124`) → `new_data_hash = m.digest()` (`:L161`), a 20-byte value.

These are never interchangeable, and this document reports each with its exact digest (§4/§5 MD5 hex, §3 SHA-1 hex) rather than conflating them. Observed values: MD5 `a59701aecccc6903a4019d07477c9a51` (§4), MD5 pair `23b1bc861edb6f698460ca6a7e15f06f`/`fccd9583fa257082613b93f8cf4bbc71` (§5), SHA-1 `84b43315ab3567f266bea8387e7d585ea6b33a9b` (§3) — all tied to their exact inputs.

### Silent success/recovery is the observed truth

Two important paths emit **no log line**, so their evidence is filesystem/DB state, and the *absence* of a log is itself the finding:

- A **successful move** (§1) — `update_filename_and_move_files` has no `logger.*` call on the success path; the capture shows `logs during tag-add: 0 -> []`.
- The **`OSError` rollback** (§2, scenario B) — the `except (OSError, DatabaseError, CannotMoveFilesException)` block (`src/documents/signals/handlers.py:L367`) restores state without logging; the capture shows `logs: []`.

### Version divergence — this commit uses `str.format()`, not Jinja2 (rule-mandated note)

At **this commit**, the filename/path is built with Python `str.format()`: `path = settings.PAPERLESS_FILENAME_FORMAT.format(...)` (`src/documents/file_handling.py:L161`), consuming `str.format()` placeholders such as `{tag_list}` and `{title}`. This commit's own bundled documentation agrees: `docs/advanced_usage.rst` shows `PAPERLESS_FILENAME_FORMAT={created_year}/{correspondent}/{title}` (~L222) and describes `{tag_list}` as "A comma separated list of all tags" (~L251), `{title}` (~L252), and the `{created*}` placeholders (~L253-L256).

The **newer live online documentation** (docs.paperless-ngx.com/advanced_usage) instead describes a **Jinja2** template syntax — it states the filename formatting uses Jinja templates and shows examples with `{{ created_year }}` and `{% if … %}` conditionals, plus newer features (`PAPERLESS_FILENAME_FORMAT_REMOVE_NONE`, per-document *storage paths*) that do **not** exist at this commit. Per the source-of-truth precedence rule, **the code at this commit prevails**: what runs here is `str.format()`, and the `{tag_list}`/`{title}` placeholders in §1 are literal `str.format()` fields. (The web corroboration is supplementary background only; the code is authoritative.)

### Concurrency guard

The entire move/rollback sequence executes inside a file lock — `with FileLock(settings.MEDIA_LOCK):` (`src/documents/signals/handlers.py:L325`), where `MEDIA_LOCK = os.path.join(MEDIA_ROOT, "media.lock")` (`src/paperless/settings.py:L72`). This serializes concurrent relocations so two tag-changes cannot race on the same file.

### `logger.fatal` surfaces as `CRITICAL`

The §2 "has gone" message is emitted with `logger.fatal(...)` (`src/documents/signals/handlers.py:L298`). In Python's stdlib logging, `Logger.fatal` is an alias for `Logger.critical`, so the record's level name is `CRITICAL` — which is exactly what the capture shows (`('paperless.handlers', 'CRITICAL', 'Document … has gone.')`).

### Dynamic created-date and `__str__`

The `Document <date> <title>` text in §2 comes from `str(instance)` (`src/documents/models.py:L212-L220`): it formats `self.created` as an ISO date (`:L213-L216`) and includes the correspondent name only when both correspondent and title are set (`:L217-L218`). Since `created` defaults to `timezone.now` (`src/documents/models.py:L152`), a freshly created row's date equals the **run date** — hence `2026-07-01` in this capture.

### `LoggingMixin` / `logging_group`

Several of these code paths log through `LoggingMixin` (`src/documents/loggers.py:L5-L21`): it holds a per-run `logging_group` UUID (`:L7`, set by `renew_logging_group()` at `:L11-L12`) and its `log(level, message, …)` method attaches `extra={"group": self.logging_group}` to every record (`:L14-L21`). The consumer uses this via `logging_name = "paperless.consumer"` (`src/documents/consumer.py:L54`), which is why the §4 ERROR is grouped and emitted under that logger. Readers filtering logs by `group` should expect this `extra` context on those records.

### Management-command entry points

The classifier and sanity behaviors are reachable from the CLI: `document_create_classifier` (`docs/administration.rst:~L333`) invokes the same `train_classifier()` path exercised in §3 — its command class imports `train_classifier` (`src/documents/management/commands/document_create_classifier.py:L1-L3`) and calls it directly inside `handle()` (`src/documents/management/commands/document_create_classifier.py:L19-L20`) — and `document_sanity_checker` (`docs/administration.rst:~L408`) invokes the same `check_sanity()` path exercised in §5/§6 — its command class imports `check_sanity` (`src/documents/management/commands/document_sanity_checker.py:L1-L2`) and calls `check_sanity(...)` followed by `messages.log_messages()` inside `handle()` (`src/documents/management/commands/document_sanity_checker.py:L22-L26`). The observation harness called these code paths directly rather than through the task queue, but the logic is identical.

---

## Final coverage pass

Every distinct sub-question from the six behaviors, and each cross-cutting requirement, is accounted for:

- **§1 File relocation on tag change**
  - [x] Before/after paths — `originals/invoice.pdf` → `originals/urgent/invoice.pdf` → `originals/paid,urgent/invoice.pdf`
  - [x] Log message during the move — **none** (`logs during tag-add: 0 -> []`); a successful move is silent
  - [x] Empty-directory cleanup — `empty 'urgent' dir cleaned=True` (`src/documents/file_handling.py:L23`)
- **§2 Rollback safety net**
  - [x] Truly recovers — DB `filename`/`archive_filename` unchanged, files restored in all three scenarios ([A], [B], [C])
  - [x] Partial-failure "partway through" behavior — **observed in scenario [C]**: after the original `os.rename` completes, the archive rename raises and the completed original move is reversed (`:L376`; captured `os.rename` sequence shows forward + reverse), archive reversal (`:L379`) skipped by its `os.path.isfile` guard, in-memory fields restored (`:L393-L394`), DB never written
  - [x] Exact `CRITICAL` "has gone" message — `Document 2026-07-01 gonedoc: File …/0000050.pdf has gone.` (`:L298`)
  - [x] Silent recovery paths — scenarios [B] and [C] both emit `logs: []`; proof is filesystem/DB state
  - [x] Two-record explanation — moving-save fires on create (`gonedoc`) and on correspondent-assignment (`gone gonedoc`)
- **§3 Classifier training: skip vs. retrain**
  - [x] Instant-skip vs. full-retrain — proven on **one** instance: the same `clf` returns `True` then `False` (`train() -> False` when the digest matches the stored `data_hash`, `:L163-L164`) vs. retrain on change
  - [x] Log message per case — INFO `Saving updated classifier model to …` (`tasks.py:L65`) vs. DEBUG `Training data unchanged.` (`tasks.py:L69`)
  - [x] Exact 20-byte SHA-1 `data_hash` — observed `84b43315ab3567f266bea8387e7d585ea6b33a9b` (reproducible on a fresh DB with the §3 seed; caveats 1–2)
  - [x] Auto-matching precondition — `train_classifier()` no-op unless a `MATCH_AUTO` object exists (`tasks.py:L49-L55`)
- **§4 Duplicate detection**
  - [x] Content-MD5 (name-independent) logic — `hashlib.md5(f.read()).hexdigest()` (`consumer.py:L104`)
  - [x] Actual MD5 compared — `a59701aecccc6903a4019d07477c9a51` on both sides (`matches_stored=True`)
  - [x] `ConsumerError` + ERROR log — `incoming_copy.pdf: Not consuming incoming_copy.pdf: It is a duplicate.` (`consumer.py:L80-L81`)
- **§5 Sanity checker: healthy vs. mismatch**
  - [x] Healthy output — `Sanity checker detected no issues.` INFO (`sanity_checker.py:L27`)
  - [x] Mismatch output — ERROR (level 40) `Checksum mismatch of document 1. Stored: …, actual: ….` (`:L89-L90`)
  - [x] Stored-vs-actual MD5 pair — `23b1bc861edb6f698460ca6a7e15f06f` vs. `fccd9583fa257082613b93f8cf4bbc71`
- **§6 Orphaned / ghost files**
  - [x] Orphans linger — reported, not removed (`orphan file still on disk=True`)
  - [x] What is reported — WARNING (level 30) `Orphaned file in media dir: /tmp/obs/media/documents/originals/orphaned` (`sanity_checker.py:L131`)
- **Cross-cutting**
  - [x] MD5-vs-SHA1 distinction stated with exact digests
  - [x] Silent success/recovery noted for §1 and §2
  - [x] Jinja2-vs-`str.format()` version divergence flagged; code at this commit prevails
  - [x] `FileLock` concurrency guard (`handlers.py:L325`)
  - [x] `logger.fatal` == `CRITICAL`
  - [x] Dynamic created-date / `__str__` (`models.py:L212-L220`)
  - [x] `LoggingMixin` / `logging_group` (`loggers.py:L5-L21`)
  - [x] Management-command entry points (`document_create_classifier`, `document_sanity_checker`)
- [x] Every quoted value carries a `file:line` citation **or** is shown with the command/code that produced it.

### Reproducibility & repository cleanliness

- All observation scripts lived **outside** the repository working tree (`/tmp/obs`) and were removed after evidence capture; any byte-compiled `__pycache__` created under `src/` while importing the modules is git-ignored and is therefore never committed.
- `git status --porcelain` is clean apart from this single new documentation file; **no existing source, test, configuration, or documentation file was modified**, and no code other than this Markdown document was added — the repository is byte-for-byte unchanged aside from the addition of `blitzy/documentation/paperless-ngx_542221a38dff.md`.

### What could not be reproduced byte-for-byte (stated honestly)

Two classes of literal are genuinely environment-varying, and are called out rather than presented as universal constants:

- **The created-date literal `2026-07-01` (§2).** It equals the *run date* (`str(instance)` formats `self.created`, which defaults to `timezone.now` — `src/documents/models.py:L152,L212-L220`), so a run on another day prints that day's date (caveat 3).
- **The media-root prefix in absolute paths.** Every `source_path`/orphan path is prefixed by the media root — here the fixed `/tmp/obs/media` from the preamble bootstrap; on another host the prefix differs while the intrinsic structure (`documents/originals/…`, the file names) is stable (caveat 2).

The **exact** SHA-1 `data_hash` (§3) and the **exact** MD5 digests (§4/§5) are functions of the precise inputs — document content, PK assignment, tag/type/correspondent PKs, and file bytes (caveat 1). Given the **fully-specified seeds shown in each section**, run against a **fresh** database (PKs starting at 1) as the harness guarantees, they *are* reproducible **byte-for-byte** — each of the six triggers above was extracted verbatim from this document and re-run, and every one reproduced its quoted evidence block exactly. What they are **not** is universal across a *different* seed, a *different* PK assignment, or a *non-fresh* database — which is precisely the input-determinism (caveats 1–2) each section demonstrates. Everything else (log strings, level names, path structure, before/after behavior, silent-success/recovery) is deterministic and reproducible from the code at this commit.
