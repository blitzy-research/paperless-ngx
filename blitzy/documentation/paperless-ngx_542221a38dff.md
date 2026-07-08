# Paperless-NGX Backend Runtime Behavior Investigation

This document answers six questions about the runtime behavior of the Paperless-NGX
document-management backend (the Django application under `src/`). It is a **run-first
investigation**: every behavior was exercised through its **real entry point** against the
canonical `paperless.settings` configuration, and the **unedited** output — log lines,
before/after filesystem paths, MD5/SHA-1 hashes, and error text — was captured and is embedded
verbatim next to each claim. Every factual statement carries a `file:line` reference and names
the specific function or method that performs the work, and each item is labeled **observed**
(shown in a captured output block) or **inferred** (reasoned directly from the cited source).

The six questions map to concrete code paths in the `src/documents/` package:

| # | Question topic | Primary code path |
|---|----------------|-------------------|
| Q1 | File relocation on tag change | `update_filename_and_move_files` — `src/documents/signals/handlers.py:312` |
| Q2 | Rollback safety net on move failure | rollback `except` block — `src/documents/signals/handlers.py:367` |
| Q3 | Classifier training (instant vs. long) | `DocumentClassifier.train` — `src/documents/classifier.py:115`; `train_classifier` — `src/documents/tasks.py:48` |
| Q4 | Duplicate detection | `Consumer.pre_check_duplicate` — `src/documents/consumer.py:102` |
| Q5 | Sanity checker (healthy vs. mismatch) | `check_sanity` — `src/documents/sanity_checker.py:49` |
| Q6 | Orphaned / ghost files | `check_sanity` orphan sweep — `src/documents/sanity_checker.py:131` |

---

## Methodology

**Real entry points.** Each behavior was driven through the same interface production code uses,
never a bypassing shim:

- **Q1** — a tag edit that fires Django's `m2m_changed` signal (and document creation firing
  `post_save`), both received by `update_filename_and_move_files` (`src/documents/signals/handlers.py:310-312`).
- **Q3** — `DocumentClassifier.train()` (`src/documents/classifier.py:115`) directly, and the
  canonical task `train_classifier()` (`src/documents/tasks.py:48`) invoked by the
  `document_create_classifier` management command (`src/documents/management/commands/document_create_classifier.py:20`).
- **Q4** — the consumer's `try_consume_file` (`src/documents/consumer.py:180`), which calls
  `pre_check_duplicate` (`src/documents/consumer.py:102`) at `src/documents/consumer.py:213`.
- **Q5/Q6** — `check_sanity()` (`src/documents/sanity_checker.py:49`) and the canonical task
  `sanity_check()` (`src/documents/tasks.py:255`), invoked by the `document_sanity_checker`
  management command (`src/documents/management/commands/document_sanity_checker.py:24-26`).

**Log capture strategy.** An in-memory `logging.Handler` was attached to the `"paperless"`
logger; its child loggers (`paperless.classifier`, `paperless.consumer`, `paperless.sanity_checker`,
`paperless.tasks`) propagate to it. Records were rendered with the canonical **verbose** format
`[{asctime}] [{levelname}] [{name}] {message}` defined at `src/paperless/settings.py:378`. The
`"paperless"` logger is configured at **DEBUG** level and routes to `paperless.log` via a
`ConcurrentRotatingFileHandler` (`src/paperless/settings.py:393`, `:395`, and the logger
declaration `"paperless": {"handlers": ["file_paperless"], "level": "DEBUG"}` at
`src/paperless/settings.py:409`), whereas the root logger's console handler surfaces only INFO+
(`src/paperless/settings.py:407`). Consequently, **DEBUG-only lines** such as
`Training data unchanged.` appear only in `paperless.log`, not on the console. Pytest's `caplog`
was deliberately avoided because Paperless reconfigures logging via `dictConfig`, which can
detach `caplog` from the records emitted by application code.

**Hashing distinction (this matters for Q3/Q4/Q5).** Two different digest algorithms are in play:

- **MD5** (`hashlib.md5`) governs duplicate detection (`src/documents/consumer.py:104`), the
  stored document `checksum` / `archive_checksum`, and every sanity-checker comparison
  (`src/documents/sanity_checker.py:83`, `:112`).
- **SHA-1** (`hashlib.sha1`) governs **only** the classifier training `data_hash`
  (`src/documents/classifier.py:124`).

**Runtime commands.** Every observation script was executed as follows (shown once; individual
questions below name the specific script actions):

```bash
source /tmp/pngx-venv/bin/activate
export PAPERLESS_MEDIA_ROOT=/tmp/pngx-media
export PAPERLESS_DATA_DIR=/tmp/pngx-data
export PAPERLESS_SCRATCH_DIR=/tmp/pngx-scratch
export DJANGO_SETTINGS_MODULE=paperless.settings
export PYTHONPATH=<repo>/src
python <observation_script>.py
```

The scratch SQLite database lived at `DATA_DIR/db.sqlite3`; media lived under
`MEDIA_ROOT/documents/{originals,archive,thumbnails}` (directory layout from
`src/paperless/settings.py:62`).

---

## Environment (canonical, disclosed)

The behaviors were exercised on the **canonical toolchain**: **Python 3.9** with
**`scikit-learn==1.0.2`**, matching the Dockerfile base `python:3.9-slim-bullseye`. Concretely, the
interpreter was **Python 3.9.25** (built from source via `pyenv`, because the host OS default is a
newer Python), and **every Python dependency was installed at its canonical pin** taken from the
project's `requirements.txt`. The runtime versions verified via `pip freeze` were:

| Package | Canonical pin | Used in investigation |
|---------|---------------|-----------------------|
| Python | 3.9 | 3.9.25 |
| scikit-learn | 1.0.2 | 1.0.2 |
| numpy | 1.22.3 | 1.22.3 |
| scipy | 1.8.0 | 1.8.0 |
| Django | 4.0.4 | 4.0.4 |
| channels | 3.0.4 | 3.0.4 |
| channels-redis | 3.4.0 | 3.4.0 |
| filelock | 3.6.0 | 3.6.0 |
| concurrent-log-handler | 0.9.20 | 0.9.20 |
| Whoosh | 2.7.4 | 2.7.4 |
| python-magic | 0.4.25 | 0.4.25 |
| pikepdf | 5.1.1 | 5.1.1 |
| pillow | 9.1.0 | 9.1.0 |
| djangorestframework | 3.13.1 | 3.13.1 |
| django-q | 1.3.9 | 1.3.9 |
| pathvalidate | 2.5.0 | 2.5.0 |
| python-dateutil | 2.8.2 | 2.8.2 |

The scratch runtime (virtualenv `/tmp/pngx-venv`, the SQLite database, and the media tree) lived
entirely under `/tmp`, and a local Redis served the default `CHANNEL_LAYERS` / `Q_CLUSTER`
backends. The only tools installed on top of the canonical pins were the test runners
(`pytest`, `pytest-django`, `factory-boy`) — not part of `requirements.txt` and **not imported by
any of the six code paths** — used solely to run the project's own test suite as a cross-check. No
behavior-relevant package deviated from its canonical pin; the only differences from the canonical
Debian-bullseye image are OS-level (host distribution and system libraries such as Ghostscript),
none of which are touched by the six code paths investigated here.

**The reported values are additionally version-agnostic.** The captured filesystem paths,
MD5/SHA-1 digests, and log strings depend only on Paperless-NGX's own source code and the input
bytes — not on the numeric library versions. Two distinct grounds support this, and they are *not*
the same kind of thing:

- The **MD5 fixtures** (`42995833e01aea9b3edee44bbfdd7ce1` original and
  `62acb0bcbfbcaa62ca6ad3668e4e404b` archive, Q4/Q5) are the project's own **test-fixture
  constants**, defined literally in the source at `src/documents/tests/test_sanity_check.py:105-106`
  and reproduced byte-for-byte at runtime.
- The **classifier SHA-1 `data_hash`** (`b41ce39793cd61f391afe34e6e4de9b361c74447`, Q3) is **not** a
  literal source constant; it is **derived at runtime** by the application's own `hashlib.sha1`
  (`src/documents/classifier.py:124`, `:161`) over the deterministic canonical fixture data
  (`generate_test_data`) — content plus auto-classification labels — and was reproduced
  **identically across ≥2 runs** on the canonical toolchain.

Because none of these digests is produced by a version-pinned library artifact, they are stable
across any supported interpreter.

---

## Q1 — File relocation on tag change

> *"When a document's tags change and the filename format (PAPERLESS_FILENAME_FORMAT) includes tags in the directory structure, files relocate. What log messages appear during the move, and what do the before/after paths look like?"*

### Mechanism (inferred from source)

The relocation is performed by `update_filename_and_move_files`
(`src/documents/signals/handlers.py:312`), which is decorated with **both**
`@receiver(m2m_changed, sender=Document.tags.through)` (`src/documents/signals/handlers.py:310`)
and `@receiver(post_save, sender=Document)` (`src/documents/signals/handlers.py:311`). Editing a
document's tags fires `m2m_changed`; creating/saving a document fires `post_save`. Either signal
runs the same handler.

Inside the handler (under `FileLock(settings.MEDIA_LOCK)`), the new name is computed by
`generate_unique_filename` → `generate_filename` (`src/documents/file_handling.py:128`). The
`{tag_list}` placeholder is rendered at `src/documents/file_handling.py:135-138` as:

```python
tag_list = pathvalidate.sanitize_filename(
    ",".join(sorted([tag.name for tag in doc.tags.all()])),
    replacement_text="-",
)
```

With format `"{tag_list}/{title}"` and **no** tags, `tag_list=""`, so the rendered path
`"/Invoice2023"` is trimmed by `path.strip(os.sep)` (`src/documents/file_handling.py:178`) to
`"Invoice2023"`. Adding the tag `Bank` makes `tag_list="Bank"`, producing `"Bank/Invoice2023"`.
The physical move is a single `os.rename(old_source_path, instance.source_path)`
(`src/documents/signals/handlers.py:354`); `create_source_path_directory`
(`src/documents/file_handling.py:19`) creates the new `Bank/` directory beforehand, and
`delete_empty_directories` (`src/documents/file_handling.py:23`) prunes the emptied old directory
afterward. The `source_path` itself is `os.path.join(settings.ORIGINALS_DIR, fname)`
(`src/documents/models.py:223`, returning at `:231`).

### Key finding (observed): the successful move is SILENT

There are **no** `logger.*` calls anywhere in the successful move body
(`src/documents/signals/handlers.py:312-395`). The only observable of a successful relocation is
the **path change itself** — the captured log buffer is empty (`[]`) on both create and tag-add.
The pre-move validator `validate_move` (`src/documents/signals/handlers.py:295`) *can* log
`logger.fatal` `"Document …: File … has gone."` (`src/documents/signals/handlers.py:298`) or
`logger.warning` `"Document …: Cannot rename file since target path … already exists."`
(`src/documents/signals/handlers.py:303`), but only on a **pre-move validation failure** — never
on a normal successful move.

The familiar INFO line `Tagging "…" with "…"` is emitted by `set_tags` during document
**consumption**, not on a manual tag edit, so it does not appear here (inferred). As a secondary
entry point, the `document_renamer` management command deliberately raises the root handler level
to ERROR (`logging.getLogger().handlers[0].level = logging.ERROR` at
`src/documents/management/commands/document_renamer.py:28`) before re-sending `post_save` per
document (`src/documents/management/commands/document_renamer.py:34`), which suppresses any
INFO/DEBUG move context.

### Command

```text
PAPERLESS_FILENAME_FORMAT="{tag_list}/{title}"
# seed a real PDF at originals/0000001.pdf
Document.objects.create(title="Invoice2023", filename="0000001.pdf", mime_type="application/pdf", …)   # fires post_save
doc.tags.add(Tag 'Bank')                                                                                # fires m2m_changed
```

### Observed output

```text
AFTER CREATE (post_save):
  doc.filename   = 'Invoice2023.pdf'
  source_path    = /tmp/pngx-media/documents/originals/Invoice2023.pdf  (exists=True)
  logs so far    = []

AFTER doc.tags.add(Tag 'Bank') (m2m_changed):
  BEFORE source_path = /tmp/pngx-media/documents/originals/Invoice2023.pdf | existed: True
  AFTER  filename    = 'Bank/Invoice2023.pdf'
  AFTER  source_path = /tmp/pngx-media/documents/originals/Bank/Invoice2023.pdf  (exists=True)
  old path still exists = False
  CAPTURED LOG LINES  = []

MEDIA TREE under originals/:
    Bank/Invoice2023.pdf
```

### Cause → effect

- **Before:** `/tmp/pngx-media/documents/originals/Invoice2023.pdf`
- **After (tag `Bank` added):** `/tmp/pngx-media/documents/originals/Bank/Invoice2023.pdf`

Adding the tag changed `{tag_list}` from `""` to `"Bank"`, so `generate_filename` produced a new
relative path `Bank/Invoice2023.pdf`; `os.rename` (`src/documents/signals/handlers.py:354`)
physically moved the file into the freshly created `Bank/` subdirectory and the old path no
longer exists (`old path still exists = False`). **Zero log lines** are emitted on success — the
move is silent, so the observable answer to "what log messages appear during the move" is: **none
on the success path**; the evidence of the move is the before/after path change.

---

## Q2 — Rollback safety net on move failure

> *"There is supposedly a rollback safety net if a file move fails partway through. Does it truly recover or just promise to? What actually happens to files when a move fails partway?"*

### Mechanism (inferred from source)

The forward moves are `os.rename` for the original (`src/documents/signals/handlers.py:354`) and
then the archive (`src/documents/signals/handlers.py:359`). If any of these (or the subsequent DB
update) fails, control enters the
`except (OSError, DatabaseError, CannotMoveFilesException)` block
(`src/documents/signals/handlers.py:367`). That block performs a **best-effort reverse**
`os.rename` for the original (`src/documents/signals/handlers.py:376`) and for the archive
(`src/documents/signals/handlers.py:379`), each guarded by an `os.path.isfile(...)` check, and
the whole reverse attempt is wrapped in an inner `except Exception: pass`
(`src/documents/signals/handlers.py:381`). Finally, it restores the in-memory instance
attributes `instance.filename = old_filename` and
`instance.archive_filename = old_archive_filename`
(`src/documents/signals/handlers.py:393-394`).

**This is NOT a database transaction.** The recovery is a best-effort reverse `os.rename` plus an
attribute reset — there is no atomic rollback and the inner `except Exception: pass` swallows any
failure of the reverse move. The source's own comment (`src/documents/signals/handlers.py:382-389`)
explains the reasoning: if the forward move from A→B succeeded, the reverse B→A is expected to
succeed too; if the forward move failed, "nothing has changed anyway."

### Answer to both parts

- **Does it truly recover, or just promise to?** In the mid-move failure case it **truly
  recovers at the file level** — the observed evidence below shows both files back at their origin
  and the instance attributes restored, with no log line emitted.
- **What actually happens to files when a move fails partway?** The completed portion of the move
  is physically undone by a reverse `os.rename`; the files end up back where they started. But
  because the recovery is best-effort / non-transactional (inner `except Exception: pass`), this
  is documented as a **finding**, not something remediated here. A second **finding** is that the
  recovery is *file-level only*: the empty destination directory created by
  `create_source_path_directory` (`src/documents/signals/handlers.py:353`, `:358`) before the
  failed rename is never removed and lingers on disk (demonstrated per-scenario below).

The reproduction mirrors `test_move_file_error`
(`src/documents/tests/test_file_handling.py:759`), which patches
`documents.signals.handlers.os.rename`. Two scenarios were run.

### Q2a — canonical mirror (original `os.rename` raises first)

**Command:** `fake_rename` raises `OSError` when `"original" in src`; drive via
`Document.objects.create(..., archive_filename="0000002.pdf", checksum="A", archive_checksum="B", correspondent=ACME)`
under `PAPERLESS_FILENAME_FORMAT="{correspondent}/{title}"`.

```text
os.rename called: True call count: 1
rename attempts: ['documents/originals/0000002.pdf -> documents/originals/ACME/other_doc.pdf']
AFTER failed move + rollback:
  original file back at ORIGINALS_DIR/0000002.pdf : True
  archive  file back at ARCHIVE_DIR/0000002.pdf   : True
  doc.filename (restored) = '0000002.pdf'
  doc.archive_filename    = '0000002.pdf'
  CAPTURED LOG LINES = []
  originals/ACME/ exists: True | empty: True
  archive/ACME/   exists: False | empty: False

FULL MEDIA TREE (documents/, [DIR]=directory, [FILE]=file):
    documents/archive/  [DIR]
    documents/archive/0000002.pdf  [FILE]
    documents/originals/  [DIR]
    documents/originals/0000002.pdf  [FILE]
    documents/originals/ACME/  [DIR]
    documents/thumbnails/  [DIR]
```

**Explanation:** the **first** `os.rename` (the original, `src/documents/signals/handlers.py:354`)
raises, so the file itself never moves — no destination *file* is created. The reverse branch is a
no-op (its `os.path.isfile(instance.source_path)` guard at
`src/documents/signals/handlers.py:375` is false because the file never left origin), and both
files remain at origin — the "nothing has changed anyway" case. Attributes are restored; no log
line is emitted.

Note the same directory side-effect documented for Q2b: `create_source_path_directory`
(`os.makedirs(os.path.dirname(source_path), exist_ok=True)`,
`src/documents/file_handling.py:19-20`) runs at `src/documents/signals/handlers.py:353` — *before*
the raising rename at `:354` — so the empty destination directory `documents/originals/ACME/` is
physically created and lingers (`exists: True | empty: True` above). Because the original move
raised first, control jumps straight to the `except` block before the archive move
(`src/documents/signals/handlers.py:356-359`) is ever reached, so `documents/archive/ACME/` is
never created (`exists: False`). The empty-directory cleanup (`src/documents/signals/handlers.py:396-410`)
again targets only the *old* directories and is skipped because the old file is present. So even in
this "nothing moved" case, one empty destination directory is left behind.

### Q2b — mid-move sibling (original moves for real, archive raises → reverse-rename fires)

**Command:** `fake_rename` performs a genuine move when `"original" in src` but raises `OSError`
when `"archive" in src`.

```text
os.rename called: True call count: 3
rename attempts: ['documents/originals/0000002.pdf -> documents/originals/ACME/other_doc.pdf', 'documents/archive/0000002.pdf -> documents/archive/ACME/other_doc.pdf', 'documents/originals/ACME/other_doc.pdf -> documents/originals/0000002.pdf']
AFTER partial move + rollback:
  original file back at ORIGINALS_DIR/0000002.pdf : True
  archive  file back at ARCHIVE_DIR/0000002.pdf   : True
  doc.filename (restored) = '0000002.pdf'
  doc.archive_filename    = '0000002.pdf'
  CAPTURED LOG LINES = []

  originals/ACME/ exists: True | empty: True
  archive/ACME/   exists: True | empty: True

FULL MEDIA TREE (documents/, [DIR]=directory, [FILE]=file):
    documents/archive/  [DIR]
    documents/archive/0000002.pdf  [FILE]
    documents/archive/ACME/  [DIR]
    documents/originals/  [DIR]
    documents/originals/0000002.pdf  [FILE]
    documents/originals/ACME/  [DIR]
    documents/thumbnails/  [DIR]
```

The listing above is the complete, unedited output of the re-run (stable across two runs);
the walk enumerates directories as well as files so lingering empty subdirectories are visible.

**Explanation:** here the original move (`src/documents/signals/handlers.py:354`) genuinely
completed (rename attempt #1), then the archive move (`src/documents/signals/handlers.py:359`)
raised (attempt #2). The **third** rename, `ACME/other_doc.pdf -> originals/0000002.pdf`, is the
reverse `os.rename` at `src/documents/signals/handlers.py:376` physically undoing the completed
original move. Both **files** end back at origin and the instance attributes are restored
(`src/documents/signals/handlers.py:393-394`), and **no log line** is emitted. This is the case
that actually exercises the safety net — and the files recover.

**Finding — the recovery is file-level only; the empty destination directories linger.** As the
FULL MEDIA TREE above shows, after rollback the two empty subdirectories
`documents/originals/ACME/` and `documents/archive/ACME/` remain on disk
(`exists: True | empty: True` for both, observed identically on both runs). The forward path
created them *before* renaming:
`create_source_path_directory` calls `os.makedirs(os.path.dirname(source_path), exist_ok=True)`
(`src/documents/file_handling.py:19-20`), invoked at `src/documents/signals/handlers.py:353`
(originals) and `:358` (archive) — and the mocked `os.rename` does not intercept `os.makedirs`,
so the directories are physically created regardless of whether the subsequent rename succeeds.
The rollback block (`src/documents/signals/handlers.py:367-390`) only reverse-renames the *files*
(`:376`, `:379`); it never removes the directories it created. The empty-directory cleanup
(`src/documents/signals/handlers.py:396-410`) calls `delete_empty_directories` only on the *old*
directories `os.path.dirname(old_source_path)` / `os.path.dirname(old_archive_path)` — never on
the new `ACME/` destinations — and only when the old file is absent (`:398`, `:404-406`); because
rollback restored the old files, that guard is false and cleanup is skipped entirely. Net result:
the safety net recovers the **files** but leaves two empty `ACME/` directories behind — the
recovery is not directory-complete.

---

## Q3 — Classifier training: instant vs. long

> *"Sometimes training finishes instantly, other times it takes longer. Trigger BOTH scenarios and capture the actual log messages explaining why training was SKIPPED vs why it proceeded with FULL retraining, including whatever hash/checksum detects changes."*

### Mechanism (inferred from source)

`DocumentClassifier.train()` (`src/documents/classifier.py:115`) computes a **SHA-1** digest over
the training data. It initializes `m = hashlib.sha1()` (`src/documents/classifier.py:124`) and,
for each non-inbox document (ordered by `pk`), updates it with **four** inputs: (1) the document's
preprocessed **content** bytes — `m.update(preprocessed_content.encode("utf-8"))`
(`src/documents/classifier.py:129`); (2) the auto-matched **document-type** label —
`m.update(y.to_bytes(4, "little", signed=True))` (`src/documents/classifier.py:136`), where
`y = document_type.pk` when the type's matching algorithm is `MATCH_AUTO`, else `-1`; (3) the
auto-matched **correspondent** label — same 4-byte signed little-endian encoding
(`src/documents/classifier.py:143`, same `pk`-or-`-1` rule); and (4) each auto-matched **tag** pk —
`m.update(tag.to_bytes(4, "little", signed=True))` (`src/documents/classifier.py:154-155`). It then
takes `new_data_hash = m.digest()` (`src/documents/classifier.py:161`). The digest therefore covers
**both the document content and the auto-classification labels/metadata**, not content alone. The
**skip** decision is:

```python
if self.data_hash and new_data_hash == self.data_hash:
    return False
```

at `src/documents/classifier.py:163-164` — this is the **"instant"** case (the data has not
changed since the last training, so training returns early). Otherwise the method logs
`Vectorizing data...` (`src/documents/classifier.py:193`), trains three MLP classifiers
(`Training tags classifier...` `:203`, `Training correspondent classifier...` `:226`,
`Training document type classifier...` `:237`), sets `self.data_hash = new_data_hash`
(`src/documents/classifier.py:247`), and `return True` (`src/documents/classifier.py:249`) — the
**"long" / FULL retrain** case. The on-disk model schema is pinned by `FORMAT_VERSION = 7`
(`src/documents/classifier.py:63`).

The canonical entry point `train_classifier()` (`src/documents/tasks.py:48`) wraps this: on a
`True` return it logs INFO `Saving updated classifier model to {MODEL_FILE}...`
(`src/documents/tasks.py:64-66`) and calls `classifier.save()` (`src/documents/tasks.py:67`); on a
`False` return it logs DEBUG `Training data unchanged.` (`src/documents/tasks.py:69`).
`MODEL_FILE` is `DATA_DIR/classification_model.pickle` (`src/paperless/settings.py:74`). The
management entry point is `document_create_classifier`
(`src/documents/management/commands/document_create_classifier.py:20`).

### Byte-exact hash + stability (observed)

The SHA-1 `data_hash` observed is `b41ce39793cd61f391afe34e6e4de9b361c74447` (20 bytes), and it
was **stable across ≥2 runs**. Because the digest is computed by the application's own
`hashlib.sha1` over deterministic inputs — the preprocessed document content **plus** the
auto-classification labels (document-type, correspondent, and tag pks;
`src/documents/classifier.py:129,136,143,154-155`) — and never over any library-versioned
artifact, its exact, repeated match on the canonical toolchain confirms the value is
**version-agnostic**: it depends only on the fixture data and the hashing algorithm, not on the
scikit-learn build.
The reproduction pattern mirrors the project's own test
(`assertTrue(self.classifier.train())` / `assertFalse(self.classifier.train())` at
`src/documents/tests/test_classifier.py:141-142`).

**Fixture:** the canonical `generate_test_data` (2 non-inbox documents, after excluding the
inbox-tagged document), which yields the counts line
`2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).` (`src/documents/classifier.py:179`).

### Command

Call `DocumentClassifier().train()` twice, then a fresh `DocumentClassifier().train()` to confirm
stability, then `train_classifier()` twice.

### Observed output — direct `train()` calls

```text
FIRST train() -> True ; data_hash (SHA-1, 20 bytes) = b41ce39793cd61f391afe34e6e4de9b361c74447
  [DEBUG] [paperless.classifier] Gathering data from database...
  [DEBUG] [paperless.classifier] 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
  [DEBUG] [paperless.classifier] Vectorizing data...
  [DEBUG] [paperless.classifier] Training tags classifier...
  [DEBUG] [paperless.classifier] Training correspondent classifier...
  [DEBUG] [paperless.classifier] Training document type classifier...

SECOND train() -> False ; captured lines:
  [DEBUG] [paperless.classifier] Gathering data from database...

STABILITY: fresh DocumentClassifier().train() -> True ; data_hash = b41ce39793cd61f391afe34e6e4de9b361c74447
  data_hash stable across 2 runs: True
```

### Observed output — canonical task entry point `train_classifier()`

```text
TASK train_classifier() FIRST call captured lines:
  [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
  [DEBUG] [paperless.classifier] Gathering data from database...
  [DEBUG] [paperless.classifier] 2 documents, 2 tag(s), 1 correspondent(s), 1 document type(s).
  [DEBUG] [paperless.classifier] Vectorizing data...
  [DEBUG] [paperless.classifier] Training tags classifier...
  [DEBUG] [paperless.classifier] Training correspondent classifier...
  [DEBUG] [paperless.classifier] Training document type classifier...
  [INFO] [paperless.tasks] Saving updated classifier model to /tmp/pngx-data/classification_model.pickle...
  MODEL_FILE now exists: True

TASK train_classifier() SECOND call captured lines:
  [DEBUG] [paperless.classifier] Gathering data from database...
  [DEBUG] [paperless.tasks] Training data unchanged.
```

### Observed output — what the hash covers (label sensitivity, content held byte-identical)

To confirm directly that the digest covers **labels** and not only content, exactly one
document-type **label** was changed (`doc1.document_type` from `dt` to `dt2`) while the document
**content was held byte-identical**; a fresh `DocumentClassifier().train()` was then run. The
`data_hash` changed, and restoring the label reproduced the original digest exactly:

```text
FIRST train() -> True ; data_hash (SHA-1) = b41ce39793cd61f391afe34e6e4de9b361c74447
  content(doc1) = 'this is a document from c1'
SECOND train() on unchanged data -> False (expect False = instant skip)

AFTER label-only change (document_type dt -> dt2), content unchanged:
  content(doc1) still = 'this is a document from c1'
  content byte-identical to before: True
  train() -> True ; data_hash (SHA-1) = 2495f526b4a3b05ce19067ec826407941885ed51
  H2 == H1 ? False  -> label change DID change the hash: True

AFTER restoring label (document_type -> dt):
  data_hash (SHA-1) = b41ce39793cd61f391afe34e6e4de9b361c74447 ; H3 == H1 ? True
```

This is direct evidence that a metadata/label change alone — with content unchanged — yields a
different digest (`2495f526b4a3b05ce19067ec826407941885ed51` ≠
`b41ce39793cd61f391afe34e6e4de9b361c74447`) and therefore forces a FULL retrain; the change is
driven by the `document_type` label update hashed at `src/documents/classifier.py:136`.

### Cause → effect

- **"Instant" (SKIPPED):** on the second call, the recomputed SHA-1 `data_hash` equals the stored
  one, so `train()` takes the early `return False` at `src/documents/classifier.py:163-164`. Only
  `Gathering data from database...` is logged; via the task, `Training data unchanged.`
  (`src/documents/tasks.py:69`, DEBUG — hence only in `paperless.log`) is emitted. No
  vectorization or MLP training runs — this is why it finishes instantly.
- **"Long" (FULL retrain):** on the first call the stored hash is empty/different, so the method
  proceeds through vectorization and three MLP trainings, sets the new `data_hash`, and returns
  `True`; the task then logs the INFO `Saving updated classifier model to
  /tmp/pngx-data/classification_model.pickle...` and persists the model. The extra
  `Document classification model does not exist (yet), …` DEBUG on the first task call comes from
  `load_classifier()` (`src/documents/classifier.py:30`) because no model file existed yet. A full
  retrain also fires on any *later* call whenever the recomputed digest differs — which happens not
  only when document **content** changes but also when the auto-classification **labels/metadata**
  change (document-type, correspondent, or auto-tag assignments), as demonstrated above where a
  label-only change flipped the digest.

The hash that "detects changes" is therefore the **SHA-1 `data_hash`**
(`src/documents/classifier.py:124`, `:161-164`), computed over the preprocessed content **and** the
auto-classification labels/metadata (`:129,136,143,154-155`): identical content *and* identical
labels → identical digest → skip; any change to content **or** to those labels/metadata → different
digest → full retrain.

---

## Q4 — Duplicate detection

> *"Two files that look completely different can still be rejected as duplicates. Show the actual checksums being compared at the moment of judgment."*

### Mechanism (inferred from source)

`Consumer.pre_check_duplicate` (`src/documents/consumer.py:102`) computes an **MD5** over the
incoming file's bytes:

```python
checksum = hashlib.md5(f.read()).hexdigest()
```

at `src/documents/consumer.py:104`, then rejects the file if any stored document matches on
either checksum column:

```python
if Document.objects.filter(
    Q(checksum=checksum) | Q(archive_checksum=checksum),
).exists():
```

at `src/documents/consumer.py:105-107`. This check runs **before any parsing**:
`try_consume_file` (`src/documents/consumer.py:180`) calls `self.pre_check_duplicate()` at
`src/documents/consumer.py:213`, which is *before* MIME detection via
`magic.from_file(...)` (`src/documents/consumer.py:219`) and parser selection
(`src/documents/consumer.py:223`). On a match, `_fail`
(`src/documents/consumer.py:78`) raises `ConsumerError(f"{self.filename}: {log_message}")`
(`src/documents/consumer.py:81`) with the message `Not consuming {filename}: It is a duplicate.`
(`src/documents/consumer.py:112`); if `CONSUMER_DELETE_DUPLICATES` (`src/paperless/settings.py:486`)
is set, `os.unlink(self.path)` (`src/documents/consumer.py:109`) runs first. The ERROR is logged
via `LoggingMixin.log` (`src/documents/loggers.py:14-21`) under `logging_name = "paperless.consumer"`
(`src/documents/consumer.py:54`). Detection is therefore **content-based** — the incoming
filename is irrelevant.

**Methodology honesty.** The WebSocket progress side-channel `Consumer._send_progress` was patched
to a no-op exactly as the canonical `test_consumer.py` harness does
(`src/documents/tests/test_consumer.py:289-291`); `pre_check_duplicate`,
`try_consume_file`, and `_fail` all ran through their real code path. `channels-redis==3.4.0`
(the canonical pin) was installed so `Consumer.__init__` (`src/documents/consumer.py:83`) can call
`get_channel_layer()` (`src/documents/consumer.py:93`) and import the production `CHANNEL_LAYERS`
backend (`RedisChannelLayer`, `src/paperless/settings.py:178-180`); no live Redis is exercised
because `group_send` never runs in these scenarios.

### Command

Drive `Consumer().try_consume_file(<path>)` for the duplicate cases (incoming = `simple.pdf`, MD5
`42995833e01aea9b3edee44bbfdd7ce1`); for the non-duplicate case call `pre_check_duplicate()`
directly on a genuinely different file.

### Observed output

```text
Q4a EXACT CHECKSUM MATCH:
  stored Document.checksum      = 42995833e01aea9b3edee44bbfdd7ce1
  incoming md5(sample.pdf)      = 42995833e01aea9b3edee44bbfdd7ce1
  ConsumerError: 'sample.pdf: Not consuming sample.pdf: It is a duplicate.'
  captured logs: ['[ERROR] [paperless.consumer] Not consuming sample.pdf: It is a duplicate.']

Q4b RENAMED-IDENTICAL FILE (content-based detection):
  incoming md5(totally_different_name.pdf) = 42995833e01aea9b3edee44bbfdd7ce1
  ConsumerError: 'totally_different_name.pdf: Not consuming totally_different_name.pdf: It is a duplicate.'
  captured logs: ['[ERROR] [paperless.consumer] Not consuming totally_different_name.pdf: It is a duplicate.']

Q4c ARCHIVE_CHECKSUM OR-ARM MATCH:
  stored Document.checksum         = some_other_original_checksum
  stored Document.archive_checksum = 42995833e01aea9b3edee44bbfdd7ce1
  incoming md5(sample2.pdf)        = 42995833e01aea9b3edee44bbfdd7ce1
  ConsumerError: 'sample2.pdf: Not consuming sample2.pdf: It is a duplicate.'
  captured logs: ['[ERROR] [paperless.consumer] Not consuming sample2.pdf: It is a duplicate.']

Q4d GENUINELY DIFFERENT CONTENT:
  incoming md5(genuinely_different.pdf) = a2158a25cbc0f4f6307f0bfedf35f7cf
  (stored checksum on file = 42995833e01aea9b3edee44bbfdd7ce1 )
  pre_check_duplicate() returned: None (None => no duplicate, no raise)
  captured logs: []

Q4e CONSUMER_DELETE_DUPLICATES=True:
  incoming file exists before: True
  ConsumerError: 'to_delete.pdf: Not consuming to_delete.pdf: It is a duplicate.'
  incoming file exists AFTER : False (False => deleted by os.unlink)
  captured logs: ['[ERROR] [paperless.consumer] Not consuming to_delete.pdf: It is a duplicate.']
```

### The moment of judgment

The judgment compares the incoming file's MD5 `42995833e01aea9b3edee44bbfdd7ce1` against every
document's `checksum` **OR** `archive_checksum` (`src/documents/consumer.py:105-107`), before any
parsing:

- **Q4a** — an exact `checksum` match is rejected.
- **Q4b** — a byte-identical file under a **completely different name**
  (`totally_different_name.pdf`) is still rejected, proving detection is **content-based**: the
  same MD5 (`42995833e01aea9b3edee44bbfdd7ce1`) triggers the rejection regardless of filename.
  This is the direct answer to "two files that look completely different can still be rejected."
- **Q4c** — the match can occur on the `archive_checksum` OR-arm: the incoming original's MD5
  equals a stored document's `archive_checksum`, and it is rejected.
- **Q4d** — a genuinely different file (MD5 `a2158a25cbc0f4f6307f0bfedf35f7cf`) is **not** flagged;
  `pre_check_duplicate()` returns `None` and raises nothing.
- **Q4e** — with `CONSUMER_DELETE_DUPLICATES=True`, the incoming file is `os.unlink`-ed
  (`src/documents/consumer.py:109`) before the `ConsumerError` is raised, so it no longer exists
  afterward.

**Note (honesty):** the Q4d file holds a **fixed, documented byte sequence** chosen only to differ
from the sample — `b"This is a genuinely different document, not a duplicate.\n"` — whose MD5 is
`a2158a25cbc0f4f6307f0bfedf35f7cf`. That value is simply "not any stored checksum"; the salient
point is content-difference → no match, and the exact bytes make it byte-for-byte reproducible.
The stored/incoming values in Q4a–Q4c are the canonical sample MD5
`42995833e01aea9b3edee44bbfdd7ce1`.

---

## Q5 — Sanity checker: healthy vs. mismatch

> *"What does its output look like when an archive is healthy vs when something is wrong? Show specific hash values it compares when it discovers a mismatch."*

### Mechanism (inferred from source)

`check_sanity()` (`src/documents/sanity_checker.py:49`) iterates every document and computes an
**MD5** of the original file (`checksum = hashlib.md5(f.read()).hexdigest()` at
`src/documents/sanity_checker.py:83`) and of the archive file
(`src/documents/sanity_checker.py:112`), comparing each to the stored `checksum` /
`archive_checksum`. On an original mismatch it records an ERROR message:

```python
messages.error(
    f"Checksum mismatch of document {doc.pk}. "
    f"Stored: {doc.checksum}, actual: {checksum}.",
)
```

at `src/documents/sanity_checker.py:88-91`. `SanityCheckMessages.log_messages`
(`src/documents/sanity_checker.py:23`) emits INFO `Sanity checker detected no issues.`
(`src/documents/sanity_checker.py:27`) when there are 0 messages, otherwise logs each message at
its own level. The canonical task `sanity_check()` (`src/documents/tasks.py:255`) then raises
`SanityCheckFailedException("Sanity check failed with errors. See log.")`
(`src/documents/tasks.py:261`) if there are errors, returns
`"Sanity check exited with warnings. See log."` (`src/documents/tasks.py:263`) if there are
warnings, and returns `"No issues detected."` (`src/documents/tasks.py:267`) when clean. The
management entry point is `document_sanity_checker`
(`src/documents/management/commands/document_sanity_checker.py:24-26`).

**Fixture:** the canonical `make_test_data` (`src/documents/tests/test_sanity_check.py:68`) using
real sample files — stored original MD5 `42995833e01aea9b3edee44bbfdd7ce1`
(`src/documents/tests/test_sanity_check.py:105`) and stored archive MD5
`62acb0bcbfbcaa62ca6ad3668e4e404b` (`src/documents/tests/test_sanity_check.py:106`).

### Command

Run `check_sanity()` + `log_messages()` + `sanity_check()` on a healthy fixture, then corrupt the
original file in place and re-run.

### Observed output

```text
[HEALTHY]
original md5 = 42995833e01aea9b3edee44bbfdd7ce1 | stored checksum        = 42995833e01aea9b3edee44bbfdd7ce1
archive  md5 = 62acb0bcbfbcaa62ca6ad3668e4e404b | stored archive_checksum = 62acb0bcbfbcaa62ca6ad3668e4e404b
len(messages) = 0 | has_error = False | has_warning = False
log_messages() emitted: ['[INFO] [paperless.sanity_checker] Sanity checker detected no issues.']
tasks.sanity_check() returned: 'No issues detected.'

[CHECKSUM MISMATCH — corrupted original]
stored checksum      = 42995833e01aea9b3edee44bbfdd7ce1
actual md5 (on disk) = 50e989d22f252fb3a2b1c060bec1a76b
len(messages) = 1 | has_error = True
log_messages() emitted:
  [ERROR] [paperless.sanity_checker] Checksum mismatch of document 1. Stored: 42995833e01aea9b3edee44bbfdd7ce1, actual: 50e989d22f252fb3a2b1c060bec1a76b.
tasks.sanity_check() raised SanityCheckFailedException: 'Sanity check failed with errors. See log.'
```

### Cause → effect

- **Healthy:** every on-disk MD5 equals the stored checksum, so there are **0 messages**;
  `log_messages()` emits the single INFO `Sanity checker detected no issues.`
  (`src/documents/sanity_checker.py:27`), and `sanity_check()` returns `'No issues detected.'`
  (`src/documents/tasks.py:267`).
- **Mismatch:** after corrupting the original, its on-disk MD5 becomes
  `50e989d22f252fb3a2b1c060bec1a76b`, which no longer equals the stored
  `42995833e01aea9b3edee44bbfdd7ce1`. The ERROR message names **both** hashes — the **stored** MD5
  and the **actual** on-disk MD5 — exactly as templated at `src/documents/sanity_checker.py:88-91`,
  and `sanity_check()` raises `SanityCheckFailedException` (`src/documents/tasks.py:261`).

**Note (honesty):** the "actual" hash `50e989d22f252fb3a2b1c060bec1a76b` is the MD5 of the **fixed,
documented** corrupt bytes written over the original —
`b"corrupted sanity-check bytes (fixed for reproducibility)\n"` — so it is byte-for-byte
reproducible; the stored checksum `42995833e01aea9b3edee44bbfdd7ce1` is the canonical fixture
value. The mechanism that matters is `MD5(original)` vs. the stored `checksum` at
`src/documents/sanity_checker.py:83-91`.

---

## Q6 — Orphaned / ghost files

> *"Do orphaned files linger in the media folder, and what does the system report when it finds them?"*

### Mechanism (inferred from source)

`check_sanity()` (`src/documents/sanity_checker.py:49`) walks `MEDIA_ROOT` into a **list** of
present files — `present_files = []` populated by `os.walk` + `.append(...)`
(`src/documents/sanity_checker.py:52-55`) — then for each document removes its thumbnail, original,
and archive paths from that list (`present_files.remove(...)`, `:67`, `:80`, `:109`).
Any file **left over** (present on disk but referenced by no document) is reported as a WARNING:

```python
for extra_file in present_files:
    messages.warning(f"Orphaned file in media dir: {extra_file}")
```

at `src/documents/sanity_checker.py:130-131`. The task `sanity_check()`
(`src/documents/tasks.py:255`) then returns `"Sanity check exited with warnings. See log."`
(`src/documents/tasks.py:263`) because there is a warning but no error.

### Key finding (observed): orphans are reported, never deleted

The sanity checker only **reports** orphaned files — it does not remove them. The extra file
persists on disk and surfaces solely as a WARNING. The project's own test
`test_orphaned_file` (`src/documents/tests/test_sanity_check.py:181`, asserting the
`Orphaned file in media dir` message at `:188`) exercises the same path.

### Command

Add an unreferenced file to the originals directory of a healthy fixture, run `check_sanity()` +
`log_messages()` + `sanity_check()`, then check whether the file still exists.

### Observed output

```text
orphan created at: /tmp/pngx-media/documents/originals/orphaned_extra_file.pdf | exists before: True
len(messages) = 1 | has_error = False | has_warning = True
log_messages() emitted:
  [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /tmp/pngx-media/documents/originals/orphaned_extra_file.pdf
tasks.sanity_check() returned: 'Sanity check exited with warnings. See log.'
orphan file STILL exists AFTER sanity check: True (True => reported, never deleted)
```

### Cause → effect

Yes — orphaned files **linger**. The unreferenced file remained in the present-files list after all
document paths were removed, so `check_sanity()` emitted the WARNING
`Orphaned file in media dir: /tmp/pngx-media/documents/originals/orphaned_extra_file.pdf`
(`src/documents/sanity_checker.py:131`) naming the exact path, and `sanity_check()` returned
`'Sanity check exited with warnings. See log.'` (`src/documents/tasks.py:263`). The final check
`orphan file STILL exists AFTER sanity check: True` confirms the file is **never removed** by the
sanity checker — it is reported only.

---

## Cross-cutting findings

These are **observations / findings** about how the system behaves — not defects proposed for
remediation here (the investigation is strictly read-only).

1. **MD5 vs. SHA-1 split.** Duplicate detection (`src/documents/consumer.py:104`), the stored
   `checksum` / `archive_checksum`, and the sanity checker
   (`src/documents/sanity_checker.py:83`, `:112`) all use **MD5**. The classifier training
   `data_hash` is the sole **SHA-1** consumer (`src/documents/classifier.py:124`). Answering
   Q3/Q4/Q5 correctly hinges on this distinction.
2. **Silent success-path move logging (Q1).** `update_filename_and_move_files`
   (`src/documents/signals/handlers.py:312-395`) emits **no** log line on a successful relocation;
   the only observable is the before/after path change. Log output appears only on the pre-move
   validation failures in `validate_move` (`src/documents/signals/handlers.py:298`, `:303`).
3. **Best-effort / non-transactional rollback (Q2).** The recovery is a reverse `os.rename`
   (`src/documents/signals/handlers.py:376`, `:379`) plus an attribute reset (`:393-394`) wrapped
   in `except Exception: pass` (`:381`). It genuinely restores the files in the mid-move failure
   case, but it is not a database transaction.
4. **Content-based duplicate detection before parsing (Q4).** `pre_check_duplicate`
   (`src/documents/consumer.py:102`) is called at `src/documents/consumer.py:213`, before MIME
   detection (`:219`) and parsing (`:223`); the incoming filename is irrelevant to the judgment.
5. **Orphans reported, not deleted (Q6).** `check_sanity()` only warns about orphaned files
   (`src/documents/sanity_checker.py:131`); they persist on disk.

---

## Coverage Pass

Every question and every named sub-item is addressed:

- **Q1 — File relocation on tag change.** ✅ *Log messages during the move:* **none on success**
  (silent move; `src/documents/signals/handlers.py:312-395`). ✅ *Before/after paths shown:*
  `.../originals/Invoice2023.pdf` → `.../originals/Bank/Invoice2023.pdf`; old path no longer
  exists.
- **Q2 — Rollback on move failure.** ✅ *Does it truly recover?* Yes at the **file level** —
  best-effort but effective in the mid-move case. ✅ *What happens to files?* Files are restored to
  origin (Q2a original-raises case; Q2b reverse-rename fires), attributes restored, no log line.
  Two findings documented: the recovery is **non-transactional**, and it is **file-level only** —
  empty destination directories (`ACME/`) created before the failed rename linger.
- **Q3 — Classifier instant vs. long.** ✅ *Both scenarios triggered:* full retrain (`train()` →
  `True`) and instant skip (`train()` → `False`). ✅ *SKIP vs. FULL log messages captured.*
  ✅ *Hash that detects changes shown:* SHA-1 `data_hash = b41ce39793cd61f391afe34e6e4de9b361c74447`,
  stable across ≥2 runs.
- **Q4 — Duplicate detection.** ✅ *Actual checksums at the moment of judgment shown:* incoming MD5
  `42995833e01aea9b3edee44bbfdd7ce1` vs. stored `checksum` / `archive_checksum`. ✅ *Content-based*
  (Q4b renamed-identical). ✅ *archive_checksum OR-arm* (Q4c). ✅ *Non-duplicate* (Q4d,
  `a2158a25cbc0f4f6307f0bfedf35f7cf`, returns `None`). ✅ *delete-duplicates* (Q4e).
- **Q5 — Sanity checker healthy vs. mismatch.** ✅ *Healthy output:* 0 messages + INFO + `'No
  issues detected.'`. ✅ *Mismatch output with specific hashes:* stored
  `42995833e01aea9b3edee44bbfdd7ce1` vs. actual `50e989d22f252fb3a2b1c060bec1a76b`; raises
  `SanityCheckFailedException`.
- **Q6 — Orphaned files.** ✅ *Do they linger?* Yes. ✅ *What is reported?* WARNING
  `Orphaned file in media dir: …` and `sanity_check()` returns the warnings string; file is never
  deleted.

---

## Reproducibility note

All runtime scaffolding — the `/tmp/pngx-venv` virtual environment, the scratch SQLite database,
the media tree, and the temporary observation scripts — lived entirely under `/tmp`, **outside the
repository**, and is investigation-only. The temporary observation scripts and the scratch
database/media tree were removed after the evidence above was captured; the shared virtual
environment is host scaffolding and is not part of the deliverable. The source repository contains
no change other than this document (`git status --porcelain` is empty once it is committed). The
only artifact added to the repository is this document.

---

## Reference files (read-only source of truth)

The following files were consulted read-only and are cited above by `file:line`; none were
modified:

- **Behavior implementations:** `src/documents/signals/handlers.py`,
  `src/documents/file_handling.py`, `src/documents/classifier.py`, `src/documents/tasks.py`,
  `src/documents/sanity_checker.py`, `src/documents/consumer.py`.
- **Support:** `src/documents/models.py`, `src/documents/loggers.py`,
  `src/paperless/settings.py`.
- **Entry points:** `src/documents/management/commands/document_create_classifier.py`,
  `src/documents/management/commands/document_sanity_checker.py`,
  `src/documents/management/commands/document_renamer.py`.
- **Reproduction recipes (tests):** `src/documents/tests/test_file_handling.py`,
  `src/documents/tests/test_classifier.py`, `src/documents/tests/test_consumer.py`,
  `src/documents/tests/test_sanity_check.py`.
