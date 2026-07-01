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
4. **Silent success/recovery is *itself* the observed truth.** A successful move (§1) emits **no** log line, and the `OSError` rollback path (§2, scenario B) emits **no** log line. For these, the evidence is the filesystem/database state, and the *absence* of a log line is the finding — stated explicitly where it applies.

> **Honesty note on the SHA-1 in §3.** Even with a fresh database and a `generate_test_data()`-style seed (PKs starting at 1), the SHA-1 `data_hash` observed here (`182885fbcaf2ee163ce2d7d096401bf2da47b766`) does **not** match values seen in other environments (an earlier brief observed `b41ce39793cd61f391afe34e6e4de9b361c74447`; an even earlier note recorded `86df1500076a69ca7d63fc5ed8826adf9dc0137a`). This is *proof* of caveat 1: the specific digest is a function of the exact (and partly undocumented) seed — document content, PK assignment, and tag/type/correspondent PKs — and is not reproducible byte-for-byte across environments. This document reports the digest observed *here* and attributes it to *this* seed. The same reasoning applies to the MD5 values in §4 and §5.

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

Set a tag-aware format, create a `Document` with a real file on disk, then add tags to fire `m2m_changed`:

```python
settings.PAPERLESS_FILENAME_FORMAT = "{tag_list}/{title}"
doc = Document.objects.create(title="invoice", mime_type="application/pdf",
                              checksum="…", storage_type=Document.STORAGE_TYPE_UNENCRYPTED)
# write bytes to doc.source_path, set doc.filename="invoice.pdf", doc.save()
doc.tags.add(Tag.objects.create(name="urgent"))   # fires m2m_changed -> move
doc.tags.add(Tag.objects.create(name="paid"))      # fires m2m_changed -> move again
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

The `m2m_changed` signal recomputes the target path from the *current* tag set every time tags change. Because the file already exists at the old path, `os.rename` physically relocates it to the newly computed path, and the directory that was emptied by the move is pruned by `delete_empty_directories`. The two tags render as `paid,urgent` — **alphabetically sorted** (`sorted(...)` at `src/documents/file_handling.py:L137`), comma-joined, and sanitized — which is why `paid` precedes `urgent` even though `urgent` was added first.

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

Two forced failures, mirroring `test_move_file_gone` (`src/documents/tests/test_file_handling.py:L738`) and `test_move_file_error` (`:L759`):

```python
# [A] missing original: format "{correspondent}/{title}"; create the doc with ONLY
# its archive on disk (original absent), then assign a correspondent and save().
settings.PAPERLESS_FILENAME_FORMAT = "{correspondent}/{title}"
# ... create doc (filename='0000050.pdf'), write archive only, no original ...
doc.correspondent = Correspondent.objects.create(name="gone"); doc.save()   # move fails

# [B] OSError during original rename: patch os.rename to raise when moving the ORIGINAL.
@mock.patch("documents.signals.handlers.os.rename")
def scenario_b(m):
    def fake_rename(src, dst):
        if "originals" in str(src): raise OSError("forced")
        return real_rename(src, dst)
    m.side_effect = fake_rename
    doc.correspondent = Correspondent.objects.create(name="also"); doc.save()
```
Recorded: whether the DB `filename` changed, whether the files stayed at their original locations, and the captured logs.

### Verbatim evidence (captured 2026-07-01, format `"{correspondent}/{title}"`)

```
[A: missing original]
   DB filename unchanged='0000050.pdf'
   archive left in place=True
   logs: [('paperless.handlers', 'CRITICAL', 'Document 2026-07-01 gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.'), ('paperless.handlers', 'CRITICAL', 'Document 2026-07-01 gone gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.')]
[B: OSError during original rename]
   os.rename called=True
   original restored in place=True
   archive untouched in place=True
   DB filename unchanged='0000060.pdf'
   logs: []
```

### Reasoning

The rollback **genuinely recovers** — it does not merely "promise to." In *both* scenarios the database row is unchanged (`DB filename unchanged`) and the files remain at their original locations. In scenario [A] the move is aborted before any rename because `validate_move` finds the original gone and raises `CannotMoveFilesException`, which the `except` block catches (`src/documents/signals/handlers.py:L367`). In scenario [B] the *original's* `os.rename` raises `OSError` mid-move; the `except` block reverses whatever partial state exists (guarded by `os.path.isfile`) and restores the in-memory fields, so the file ends up back where it started.

Scenario [A] shows **two** `CRITICAL` records because the moving-save fires **twice**: once when the document is first created (via `post_save`, before the correspondent is set → `Document 2026-07-01 gonedoc`), and once on the correspondent-assignment `save()` (→ `Document 2026-07-01 gone gonedoc`). Each failed attempt logs exactly once. This is a direct consequence of `update_filename_and_move_files` being registered on `post_save` *and* `m2m_changed` (`src/documents/signals/handlers.py:L310-L311`).

### Sub-part answers

- **Does it truly recover? ✓** Yes. DB `filename` is unchanged in both scenarios; files are restored to (or never left) their original locations.
- **What happens on partial failure? ✓** Any partial `os.rename` is reversed (`src/documents/signals/handlers.py:L376,L379`) and the in-memory `filename`/`archive_filename` are restored (`:L393-L394`); the DB write never happens because it only runs *after* the renames succeed.
- **The exact `CRITICAL` "has gone" message ✓** — `Document 2026-07-01 gonedoc: File /tmp/obs/media/documents/originals/0000050.pdf has gone.` at level `CRITICAL` (from `logger.fatal`, `src/documents/signals/handlers.py:L298`).
- **The silent `OSError` path ✓** — scenario [B] emits **no** log (`logs: []`); its proof is the filesystem/DB state (`original restored in place=True`, `DB filename unchanged`). The absence of a log is the finding.
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
(`src/documents/classifier.py:L163-L164`). If the freshly computed digest equals the stored one, `train()` returns `False` immediately — *before* any model fitting. Notably, the scikit-learn imports are **lazy** — `from sklearn.feature_extraction.text import CountVectorizer` and friends live *inside* `train()` at `src/documents/classifier.py:L188-L190` (and `:L274`), i.e. **after** the hash+skip check. This means the change-detection hash is computed with no scikit-learn involvement and is therefore **version-independent**.

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

Seed at least one auto-matching object (so the task is not a no-op), plus documents with distinct content, then call the classifier three ways — this mirrors `testDatasetHashing` (`src/documents/tests/test_classifier.py:L137-L142`) and `testSaveClassifier` (`:L168-L178`):

```python
c1 = Correspondent.objects.create(name="c1", matching_algorithm=Correspondent.MATCH_AUTO)
t1 = Tag.objects.create(name="t1", matching_algorithm=Tag.MATCH_AUTO)
dt = DocumentType.objects.create(name="dt", matching_algorithm=DocumentType.MATCH_AUTO)
doc1 = Document.objects.create(title="doc1", content="this is a document from c1",
                               correspondent=c1, document_type=dt, checksum="A", ...)
doc2 = Document.objects.create(title="doc2", content="this is another document, but from c2", checksum="B", ...)
doc1.tags.add(t1)

DocumentClassifier().train()   # #1 -> True  (prints SHA-1 hex + byte length)
DocumentClassifier().train()   # #2 -> False (unchanged)
tasks.train_classifier()       # fresh model -> INFO save
tasks.train_classifier()       # unchanged   -> DEBUG "Training data unchanged."
# add doc3, then:
tasks.train_classifier()       # changed      -> INFO save
```

### Verbatim evidence (captured; your SHA-1 hex will differ — caveat 1)

```
DocumentClassifier().train() #1 -> True | data_hash(sha1 hex)= 182885fbcaf2ee163ce2d7d096401bf2da47b766 | len(bytes)= 20
DocumentClassifier().train() #2 (no data change) -> False
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
   [INFO][paperless.tasks] Saving updated classifier model to /tmp/obs/data/classification_model.pickle...
```

The SHA-1 `data_hash` observed **here** is `182885fbcaf2ee163ce2d7d096401bf2da47b766` (20 bytes). Per caveat 1 and the honesty note in the preamble, this exact hex is a function of *this* seed (document content + the auto `Correspondent`/`DocumentType` PKs + the sorted auto `Tag` PK) and is **not** reproducible byte-for-byte in a different environment — the whole point of the section is that the system compares *this kind* of digest, and your run will produce your own.

### Reasoning

"Instant" training is the early `return False` (`src/documents/classifier.py:L163-L164`): when the freshly computed SHA-1 equals the stored `data_hash`, `train()` bails out before touching scikit-learn, so it finishes in microseconds. "Full retrain" happens whenever the digest differs — a new/edited/removed document, or a changed auto type/correspondent/tag — because the mismatch falls through the skip check and proceeds to vectorize and fit the models. The **INFO vs. DEBUG** split is precisely how the task announces "I saved a new model" (`Saving updated classifier model to …`) versus "nothing to do" (`Training data unchanged.`). The `2 documents, 1 tag(s), 1 correspondent(s), 1 document type(s).` line on call #1 versus `3 documents, …` on call #3 confirms the retrain saw the mutated data set.

### Sub-part answers

- **Instant-skip vs. full-retrain triggers ✓** — skip when `new_data_hash == self.data_hash` (`src/documents/classifier.py:L163-L164`); retrain on any digest change (new doc `doc3` on call #3).
- **The log message for each ✓** — retrain → **INFO** `Saving updated classifier model to /tmp/obs/data/classification_model.pickle...` (`src/documents/tasks.py:L65`); skip → **DEBUG** `Training data unchanged.` (`src/documents/tasks.py:L69`).
- **The exact hash used ✓** — a **20-byte SHA-1** `data_hash` (`src/documents/classifier.py:L124,L161`). Observed here as hex `182885fbcaf2ee163ce2d7d096401bf2da47b766`; reported as *this run's* value and attributed to *this* seed (caveat 1). This is **SHA-1**, distinct from the MD5 used for file integrity elsewhere.
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
content = b"Invoice #42 - Acme Corp - total due 100.00\n"
md5 = hashlib.md5(content).hexdigest()
Document.objects.create(title="stored", checksum=md5, mime_type="application/pdf", ...)  # NO filename -> avoids a stray move signal
# write the SAME bytes to a differently-named file:
open("/tmp/incoming_copy.pdf", "wb").write(content)
consumer.path = "/tmp/incoming_copy.pdf"; consumer.filename = "incoming_copy.pdf"
consumer.pre_check_duplicate()   # -> raises ConsumerError
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
checksum = hashlib.md5(f.read()).hexdigest()
...
if not checksum == doc.checksum:
    messages.error(
        f"Checksum mismatch of document {doc.pk}. "
        f"Stored: {doc.checksum}, actual: {checksum}.",
    )
```
(`src/documents/sanity_checker.py:L83`, `:L87-L91`). The archive is handled the same way, comparing against `doc.archive_checksum` and producing `Checksum mismatch of archived document {doc.pk}. …` (`src/documents/sanity_checker.py:L112,L118-L124`).

Results are accumulated in a `SanityCheckMessages` object whose helpers stamp a logging level on each message: `error()` → `logging.ERROR` (40) (`src/documents/sanity_checker.py:L14-L15`), `warning()` → `logging.WARNING` (30) (`:L17-L18`), `info()` → `logging.INFO` (20) (`:L20-L21`). Emission happens in `log_messages()` (`src/documents/sanity_checker.py:L23`): if there are **no** messages it logs `logger.info("Sanity checker detected no issues.")` (`src/documents/sanity_checker.py:L27`); otherwise it logs each message at its own level via `logger.log(msg["level"], msg["message"])` (`:L30`). Convenience predicates `has_error` (`:L38-L39`) and `has_warning` (`:L41-L42`) report whether any ERROR/WARNING message is present.

### Trigger

Run `check_sanity()` twice under a fresh temporary media root — once where the file matches its stored checksum, once where it does not:

```python
# HEALTHY: stored checksum == md5 of the bytes actually on disk -> "no issues"
# CORRUPT: stored checksum = md5(B1) but the file on disk contains B2 (different bytes)
B1 = b"ORIGINAL bytes v1"
B2 = b"TAMPERED bytes v2 (longer, different)"
doc.checksum = hashlib.md5(B1).hexdigest()      # what the DB believes
open(doc.source_path, "wb").write(B2)            # what is actually on disk
msgs = check_sanity(); msgs.log_messages()
```
(Mirrors the checksum-mismatch cases in `src/documents/tests/test_sanity_check.py`.)

### Verbatim evidence (fresh media root; stored=`md5(b"ORIGINAL bytes v1")`, actual=`md5(b"TAMPERED bytes v2 (longer, different)")`)

```
[HEALTHY] len=0 has_error=False has_warning=False  logs=[('INFO', 'Sanity checker detected no issues.')]
[CORRUPT] len=1 has_error=True  stored=23b1bc861edb6f698460ca6a7e15f06f actual=fccd9583fa257082613b93f8cf4bbc71
   [level 40] Checksum mismatch of document 1. Stored: 23b1bc861edb6f698460ca6a7e15f06f, actual: fccd9583fa257082613b93f8cf4bbc71.
```

Both MD5s were independently verified: `md5(b"ORIGINAL bytes v1") = 23b1bc861edb6f698460ca6a7e15f06f` and `md5(b"TAMPERED bytes v2 (longer, different)") = fccd9583fa257082613b93f8cf4bbc71`. Per caveat 1, these are byte-dependent; different content yields different digests.

### Reasoning

The checker is a straightforward **fresh MD5 recomputation** compared against the stored value. When the on-disk bytes match what the database recorded, the comparison at `src/documents/sanity_checker.py:L88` is `True`, no message is appended, and `log_messages()` takes the empty branch and emits the single INFO line `Sanity checker detected no issues.` (`:L27`). When the bytes differ (the "tampered" file), the comparison is `False`, one ERROR message is appended naming the document PK and **both** digests (`:L89-L90`), and it is emitted at level 40. `has_error` is `True` (`:L38-L39`), and `has_warning` is `False` because the mismatch is an ERROR, not a WARNING. The level `40` printed above is `logging.ERROR` numerically.

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

Under a fresh temporary media root, create a file that no `Document` references, then run the checker — mirroring `test_orphaned_file` (`src/documents/tests/test_sanity_check.py:L181-L188`):

```python
from pathlib import Path
Path(originals_dir, "orphaned").touch()   # a file no Document accounts for
msgs = check_sanity(); msgs.log_messages()
# afterwards: assert Path(originals_dir, "orphaned").exists()  # still there
```

### Verbatim evidence (fresh media root `/tmp/tmpr3h26vmp`)

```
[ORPHAN] len=1 has_error=False has_warning=True
   messages[0]={'level': 30, 'message': 'Orphaned file in media dir: /tmp/tmpr3h26vmp/documents/originals/orphaned'}
   [WARNING] Orphaned file in media dir: /tmp/tmpr3h26vmp/documents/originals/orphaned
   orphan file still on disk=True
```

### Reasoning

The sanity checker's job is to **report** integrity problems, not to repair them. An unaccounted file is surfaced as a single WARNING (level 30) via `messages.warning(...)` (`src/documents/sanity_checker.py:L131`) and left exactly where it is — confirmed by `orphan file still on disk=True` after the run. So yes: orphaned files genuinely linger; the system tells you about them but does not remove them. (The concrete temp path differs per run because a fresh `mkdtemp()` directory is used — caveat 1/2.)

### Sub-part answers

- **Do orphans linger? ✓** Yes — the file is reported, **not** removed (`orphan file still on disk=True`); the code has no unlink on this path (`src/documents/sanity_checker.py:L130-L131`).
- **What does the system report? ✓** A single **WARNING** at level **30**: `Orphaned file in media dir: /tmp/tmpr3h26vmp/documents/originals/orphaned` (`src/documents/sanity_checker.py:L131`); `has_warning=True`, `has_error=False`.

---

## Cross-cutting findings

These themes recur across the six behaviors and are worth stating once, explicitly.

### Hash algorithms differ by purpose — MD5 for file integrity, SHA-1 for classifier change detection

- **MD5** is used for *document/archive content integrity*: the consumer's duplicate check `hashlib.md5(f.read()).hexdigest()` (`src/documents/consumer.py:L104`) and the sanity checker's recomputations for original (`src/documents/sanity_checker.py:L83`) and archive (`:L112`).
- **SHA-1** is used for the *classifier change-detection* digest: `hashlib.sha1()` (`src/documents/classifier.py:L124`) → `new_data_hash = m.digest()` (`:L161`), a 20-byte value.

These are never interchangeable, and this document reports each with its exact digest (§4/§5 MD5 hex, §3 SHA-1 hex) rather than conflating them. Observed values: MD5 `a59701aecccc6903a4019d07477c9a51` (§4), MD5 pair `23b1bc861edb6f698460ca6a7e15f06f`/`fccd9583fa257082613b93f8cf4bbc71` (§5), SHA-1 `182885fbcaf2ee163ce2d7d096401bf2da47b766` (§3) — all tied to their exact inputs.

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

The classifier and sanity behaviors are reachable from the CLI: `document_create_classifier` (`docs/administration.rst:~L333`) invokes the same `train_classifier()` path exercised in §3, and `document_sanity_checker` (`docs/administration.rst:~L408`) invokes the same `check_sanity()` path exercised in §5/§6. The observation harness called these code paths directly rather than through the task queue, but the logic is identical.

---

## Final coverage pass

Every distinct sub-question from the six behaviors, and each cross-cutting requirement, is accounted for:

- **§1 File relocation on tag change**
  - [x] Before/after paths — `originals/invoice.pdf` → `originals/urgent/invoice.pdf` → `originals/paid,urgent/invoice.pdf`
  - [x] Log message during the move — **none** (`logs during tag-add: 0 -> []`); a successful move is silent
  - [x] Empty-directory cleanup — `empty 'urgent' dir cleaned=True` (`src/documents/file_handling.py:L23`)
- **§2 Rollback safety net**
  - [x] Truly recovers — DB `filename` unchanged, files restored in both scenarios
  - [x] Partial-failure behavior — partial `os.rename` reversed (`:L376,L379`), in-memory fields restored (`:L393-L394`), DB never written
  - [x] Exact `CRITICAL` "has gone" message — `Document 2026-07-01 gonedoc: File …/0000050.pdf has gone.` (`:L298`)
  - [x] Silent `OSError` path — `logs: []`; proof is filesystem/DB state
  - [x] Two-record explanation — moving-save fires on create (`gonedoc`) and on correspondent-assignment (`gone gonedoc`)
- **§3 Classifier training: skip vs. retrain**
  - [x] Instant-skip vs. full-retrain — `train() -> False` when digest unchanged (`:L163-L164`) vs. retrain on change
  - [x] Log message per case — INFO `Saving updated classifier model to …` (`tasks.py:L65`) vs. DEBUG `Training data unchanged.` (`tasks.py:L69`)
  - [x] Exact 20-byte SHA-1 `data_hash` — observed `182885fbcaf2ee163ce2d7d096401bf2da47b766` (attributed to this seed; caveat 1)
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
  - [x] What is reported — WARNING (level 30) `Orphaned file in media dir: <path>` (`sanity_checker.py:L131`)
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

- All observation scripts lived **outside** the repository working tree (`/tmp/obs`) and were removed after evidence capture; any byte-compiled `__pycache__` created under `src/` during setup was cleaned up.
- `git status --porcelain` is clean apart from this single new documentation file; **no existing source, test, configuration, or documentation file was modified**, and no code other than this Markdown document was added — the repository is byte-for-byte unchanged aside from the addition of `blitzy/documentation/paperless-ngx_542221a38dff.md`.

### What could not be reproduced byte-for-byte (stated honestly)

The **exact** SHA-1 `data_hash` (§3) and the **exact** MD5 digests (§4/§5) are functions of the precise inputs — document content, PK assignment, tag/type/correspondent PKs, and file bytes. They are **not** reproducible byte-for-byte across environments (caveats 1–2). This document therefore reports the values observed in *this* run, shows the exact inputs that produced them, and marks them as seed/byte-dependent rather than universal constants. Everything else (log strings, level names, path structure, before/after behavior, silent-success/recovery) is deterministic and reproducible from the code at this commit.
