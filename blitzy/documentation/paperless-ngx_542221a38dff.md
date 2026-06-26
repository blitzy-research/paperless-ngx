# Paperless-ngx Runtime Choreography — Six Behaviors, Captured Live

This is an evidence-backed investigative report into the *hidden choreography* of six runtime
behaviors inside the Paperless-ngx core engine (`src/documents/`). For each behavior the report
pairs **genuine, captured runtime artifacts** — real absolute paths, real MD5/SHA-1 hash values,
and verbatim log lines (with logger name and level) — with the **code-level rationale** that
produces the behavior, anchored by `file:line` citations into the source tree.

The investigation is strictly **read-only** with respect to the application source. The behaviors
were exercised against a live run of the Django `documents` app; all scratch instrumentation lived
outside the repository and was deleted afterward. No product code was changed.

---

## Preamble / Environment

### Repository under study

| Field | Value |
|-------|-------|
| Project | Paperless-ngx |
| Branch | `paperless-ngx_542221a38dff` |
| Commit | `542221a38dff06361e07976452f9aea24d210542` |
| Core engine | `src/documents/` (the Django `documents` app) |

### Capture environment

- **Runtime:** Python 3.9 / Django 4.0.4, with SQLite as the default database. *(The Django
  version was verified from `requirements.txt` (`django==4.0.4`) and `Pipfile.lock`
  (`"version": "==4.0.4"`), corroborated by the runtime environment summary.)*
- **Storage:** A scratch `MEDIA_ROOT` and `DATA_DIR` were provisioned **outside** the repository
  (under `/tmp/pngx_scratch`) so that no real document store was touched.
- **Logging:** `DEBUG` was enabled on the `paperless.*` loggers so that the normally-quiet DEBUG
  breadcrumbs were observable.
- **Async vs. synchronous capture:** The classifier and sanity-check routines were invoked
  **directly in a Django shell** rather than dispatched to a background Django-Q worker. This is
  deliberate — running them inline keeps their log output attached to the foreground process so it
  can be captured, instead of being swallowed by a detached worker.
- **Source-tree integrity:** Throughout the investigation the source tree was kept
  **byte-for-byte unchanged**; `git status --porcelain` on the application source remained empty.

### CRITICAL — Hash-algorithm clarification (read this first)

Paperless-ngx uses **two distinct hash algorithms for two unrelated purposes**, and a third
algorithm is sometimes wrongly attributed to it. These must never be conflated:

| Algorithm | Purpose | Digest width | Where in code |
|-----------|---------|--------------|---------------|
| **MD5** | File integrity **and** deduplication | 16 bytes / **32 hex** | `src/documents/consumer.py:L104` (`hashlib.md5(f.read()).hexdigest()`); `src/documents/sanity_checker.py:L83` & `:L112` (MD5 recompute of original and archive). Columns `checksum` / `archive_checksum` are `CharField(max_length=32)` at `src/documents/models.py:L135-L141` & `:L143-L150` — 32 hex chars ⇒ MD5. |
| **SHA-1** | Classifier training-data **change detection only** | 20 bytes / **40 hex** | `src/documents/classifier.py:L124` (`m = hashlib.sha1()`), finalized `new_data_hash = m.digest()` at `:L161`. This answers *"should I retrain?"* — it is **not** file integrity. |
| **SHA-256** | **Explicitly rejected for this commit** | (64 hex) | A third-party wiki claims SHA-256 is used for original/archive files. This is **incorrect for commit `542221a38dff`**: the code uses `hashlib.md5` (`src/documents/consumer.py:L104`) and the columns are `max_length=32` (`src/documents/models.py:L135-L150`). A SHA-256 hex digest would be 64 characters; the 32-character columns make MD5 the only possibility. |

> The MD5 (integrity/dedup) and SHA-1 (classifier change-detection) mechanisms are completely
> independent. They are computed over different inputs, stored in different places, and serve
> different goals. Do not assume one implies the other.

### Determinism legend

Captured artifacts fall into two categories, and each artifact below is tagged accordingly:

- **STABLE** — content-derived and reproducible anywhere. The same inputs always yield the same
  value (e.g. an MD5 of identical bytes; the SHA-1 *algorithm* and its 20-byte/40-hex width; log
  message templates and logger names; the `Invoice.pdf` → `Invoice Paid.pdf` filename transform;
  the relative on-disk layout `documents/{originals,archive,thumbnails}/`).
- **ENV-SPECIFIC** — varies per run/environment (e.g. absolute paths such as those under
  `/tmp/pngx_scratch`; document primary keys; timestamps and the `2026-06-26` date prefix that
  appears in `str(document)`; and the SHA-1 `data_hash` *value*, which depends on the full
  training corpus and label primary keys).

### Loggers reference

The subsystems under study emit through named loggers. The logging mechanism itself is
`LoggingMixin.log(...)` in `src/documents/loggers.py:L14-L21`, which resolves
`logging.getLogger(self.logging_name)` and dispatches via `getattr(logger, level)(message, ...)`:

| Logger | Subsystem |
|--------|-----------|
| `paperless.handlers` | Signal handlers, including tag-driven relocation (`src/documents/signals/handlers.py:L27`) |
| `paperless.filehandling` | Filename generation / file handling |
| `paperless.matching` | Tag/correspondent/document-type matching |
| `paperless.classifier` | Classifier model build and gather steps |
| `paperless.tasks` | Task wrappers (`train_classifier`, `sanity_check`) |
| `paperless.consumer` | Document consumption and duplicate detection |
| `paperless.sanity_checker` | Sanity-check report (`src/documents/sanity_checker.py:L24`) |

---

## Q1 — Tag-Driven File Relocation

### Question

When a document's tags change and `PAPERLESS_FILENAME_FORMAT` embeds tags in the storage layout,
does the document's **original** *and* **archive** file actually get moved on disk? And what, if
anything, is logged?

### How It Was Triggered

`PAPERLESS_FILENAME_FORMAT` was set to `'{title} {tag_list}'`. A `Document` (pk=1,
`title='Invoice'`) was created with `filename='Invoice.pdf'` and `archive_filename='Invoice.pdf'`,
and real files were touched at both the original and archive paths. The relocation was then fired
by adding a tag:

```python
doc.tags.add(Tag(name='Paid', matching_algorithm=MATCH_ANY))
```

Adding the tag fires the `m2m_changed` signal on `Document.tags.through`, which invokes
`update_filename_and_move_files`.

### Captured Evidence

**BEFORE** the tag change — `filename='Invoice.pdf'`, `archive_filename='Invoice.pdf'`:

```
source_path  = /tmp/pngx_scratch/media/documents/originals/Invoice.pdf   (exists)
archive_path = /tmp/pngx_scratch/media/documents/archive/Invoice.pdf     (exists)
```

*(ENV-SPECIFIC: the absolute paths under `/tmp/pngx_scratch`; STABLE: the relative layout
`documents/originals/` and `documents/archive/`.)*

**TRIGGER:** `doc.tags.add(Tag 'Paid')`

**AFTER** the tag change — `filename='Invoice Paid.pdf'`, `archive_filename='Invoice Paid.pdf'`:

```
source_path  = /tmp/pngx_scratch/media/documents/originals/Invoice Paid.pdf   (exists=True)
archive_path = /tmp/pngx_scratch/media/documents/archive/Invoice Paid.pdf     (exists=True)
OLD original exists? False
OLD archive   exists? False
```

Both files were genuinely relocated via `os.rename`, and neither old path survives.
*(STABLE: the `'Invoice.pdf'` → `'Invoice Paid.pdf'` transform. ENV-SPECIFIC: the absolute paths.)*

**Log capture during the move: NONE.** Zero `paperless.*` lines were emitted (only `filelock`
DEBUG noise around `MEDIA_LOCK`). This is the **"silent success"** insight — *a successful
tag-driven move emits no dedicated log line.* A reader watching the logs for a "moved file"
message would wrongly conclude nothing happened; the real evidence is the before/after path delta
on disk.

For contrast, the log lines a reader *would* see when tags are assigned **during consumption**
(captured by exercising `set_tags` through the consumption-finished pipeline) are:

```
DEBUG [paperless.matching] Tag acme matched on document 2026-06-26 Receipt because it contains this string: "acme corp"
INFO [paperless.handlers] Tagging "2026-06-26 Receipt" with "acme"
```

*(ENV-SPECIFIC: the `2026-06-26` date prefix in `str(document)`.)* The nuance: this `INFO` "Tagging"
line fires only via the `document_consumption_finished` pipeline, **not** on a manual ORM
`tags.add()`. A pure ORM tag change is therefore fully silent (see Cross-cutting insight #2).

### Code Rationale

- `update_filename_and_move_files` is a `@receiver` bound to **both** `m2m_changed` (on
  `Document.tags.through`) **and** `post_save` (on `Document`) — `src/documents/signals/handlers.py:L310-L312`.
  Adding or removing a tag therefore fires the relocation logic.
- The new name is computed by `generate_unique_filename` → `generate_filename`
  (`src/documents/file_handling.py:L128-L199`), which reads `settings.PAPERLESS_FILENAME_FORMAT`
  (`:L132`, `:L161`) and supports the `{tags}` / `{tag_list}` placeholders at `:L174-L175`.
- The move is two `os.rename` calls: the original at `src/documents/signals/handlers.py:L354` and
  the archive at `:L359`, each preceded by `validate_move` (`:L295-L307`) and
  `create_source_path_directory` — a silent `os.makedirs(..., exist_ok=True)` at
  `src/documents/file_handling.py:L19-L20`.
- The before/after paths derive from the model path properties:
  `Document.source_path = ORIGINALS_DIR/<filename>` (`src/documents/models.py:L222-L231`) and
  `Document.archive_path = ARCHIVE_DIR/<archive_filename>` (`:L241-L246`).
- `PAPERLESS_FILENAME_FORMAT` defaults to `None` (`src/paperless/settings.py:L584`); the handler
  logger is `paperless.handlers` (`src/documents/signals/handlers.py:L27`).
- A CLI mass-rename path exists — `document_renamer` sends `post_save` for every document
  (`src/documents/management/commands/document_renamer.py`, `handle()` ~L26-L34) but first raises
  the root log handler to ERROR (`logging.getLogger().handlers[0].level = logging.ERROR` ~L28),
  which suppresses most messages during a bulk rename.

### Reproduction Steps

1. Set `PAPERLESS_FILENAME_FORMAT='{title} {tag_list}'`.
2. Create a `Document` with original and archive files present on disk.
3. Enable `DEBUG` on the `paperless.*` loggers.
4. Run `doc.tags.add(<tag>)`.
5. Re-read `doc.source_path` / `doc.archive_path`: confirm the **new** paths exist and the **old**
   paths do not. Observe that **no dedicated move log line** appears.

### Documentation corroboration

> *Corroboration (code remains authoritative):* The official Paperless-ngx docs note that with
> `{tag_list}` you may hit OS maximum path lengths, in which case files "retain the previous path"
> and the issue is logged — matching the invalid/over-limit fallback in
> `src/documents/file_handling.py:L181-L184`. The docs also state that changing
> `PAPERLESS_FILENAME_FORMAT` requires manually running the document renamer to move existing
> documents.

---


## Q2 — Move Rollback Safety Net

### Question

Does the file-move routine *truly* roll back on a partway failure — restoring the original files
*and* the database/in-memory state — or does it merely "promise to"?

### How It Was Triggered

A `Document` (pk=3, `filename='Statement.pdf'`) was created with both files present. A tag change
was then triggered while `documents.signals.handlers.Document.objects.filter` was patched to raise
`DatabaseError`. A source read confirms that `filter(pk=...).update(...)` at
`src/documents/signals/handlers.py:L362-L365` is the **only** `filter` call in the routine and that
it runs **after** both `os.rename` calls (original at L354, archive at L359). So both files genuinely
moved A→B, and then the database write raised `DatabaseError`, driving execution into the `except`
branch.

### Captured Evidence

**AFTER the rollback** — the in-memory attributes and on-disk files were both restored:

```
filename     restored to 'Statement.pdf'   (== before: True)
archive_filename restored to 'Statement.pdf'
original restored at OLD path = True
archive  restored at OLD path = True
stray file at NEW original path = False
stray file at NEW archive  path = False
```

The reverse renames plus the in-memory attribute restoration genuinely returned the files **and**
the attributes to their original state.

**Log capture during the rollback: NONE** — zero `paperless.*` lines. This is the **"silent revert"**
insight: *the rollback emits no dedicated log line.*

The safety net is an **invariant, not a filesystem transaction.** `validate_move` enforces a
never-overwrite rule. The FATAL path was captured separately by deleting the source file and then
triggering a move:

```
CRITICAL [paperless.handlers] Document 2026-06-26 Ghost: File /tmp/pngx_scratch/media/documents/originals/Ghost.pdf has gone.
```

Note the level: the code calls `logger.fatal(...)`, which Python renders at **CRITICAL** level (see
Cross-cutting insight #3). *(ENV-SPECIFIC: the absolute path and the `2026-06-26` prefix.)* After
this abort, `filename` stayed `'Ghost.pdf'` (unchanged) — the move aborted cleanly via
`CannotMoveFilesException`.

The deliberate inner-`except` block silently `pass`es. Its in-code justification is reproduced
**verbatim** below (the original typos "santiy" and "orignal" are preserved as they appear in the
source):

```
This is fine, since:
A: if we managed to move source from A to B, we will also manage to move it from B to A. If not, we have a serious issue that's going to get caught by the santiy checker. All files remain in place and will never be overwritten, so this is not the end of the world. B: if moving the orignal file failed, nothing has changed anyway.
```

### Code Rationale

- The database update (`filename` / `archive_filename`) persists **only after** both renames
  succeed — `src/documents/signals/handlers.py:L362-L365`.
- The `try` spans `:L326-L394`. On `except (OSError, DatabaseError, CannotMoveFilesException)` at
  `:L367`, the code reverses the renames at `:L376` (original) and `:L379` (archive), then restores
  the in-memory `instance.filename` / `instance.archive_filename` at `:L393-L394`.
- The inner `except Exception` (`:L381`) silently `pass`es (`:L390`) on the rationale in the
  verbatim comment above: the reverse should succeed if the forward did; `validate_move` guarantees
  targets are never overwritten (`:L303-L306`); and any residual inconsistency is "going to get
  caught by the santiy checker."
- `validate_move` provides the never-overwrite invariant plus a missing-source guard at
  `:L295-L307` — the FATAL `"File … has gone."` at `:L298` and the WARNING on an existing target at
  `:L303-L306`.
- **Conclusion:** the rollback **really does** restore the files (verifiable on disk) and the
  in-memory attributes, and it is **silent** (no log line). The robustness comes from the
  never-overwrite invariant and the sanity checker as a backstop — not from a transactional
  filesystem.

### Reproduction Steps

1. Create a `Document` with both files on disk and a tag-embedding `PAPERLESS_FILENAME_FORMAT`.
2. Monkeypatch the post-rename database `.update()` to raise `DatabaseError`.
3. Trigger a tag change.
4. Confirm both files are back at their original paths, no strays exist at the new paths, and
   `instance.filename` is restored — all with **no log line**.
5. Separately, delete the source file and trigger a move to capture the `CRITICAL`
   `"File … has gone."` line.

### Documentation corroboration

> *Corroboration (code remains authoritative):* The official usage docs state Paperless "will never
> overwrite" the original document, and the administration docs state the renaming logic "is robust
> and will never overwrite or delete a file" — corroborating the never-overwrite invariant plus the
> sanity checker as the backstop.

---


## Q3 — Classifier Training: Skip vs Full Retrain

### Question

Why does classifier training sometimes finish almost instantly and sometimes retrain fully? What
hash drives that decision?

### How It Was Triggered

`train_classifier()` — the function behind the `document_create_classifier` management command —
was invoked **three times**: (1) an initial build, (2) immediately again with unchanged data, and
(3) after adding a new document. *(Capture note: `scikit-learn` was installed for the run, but the
`data_hash` computation is independent of scikit-learn; pure-Python and C-library import shims used
to let `tasks.py` import lived entirely outside the repo and touched no source file.)*

### Captured Evidence

**FIRST run (build):**

```
DEBUG [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
DEBUG [paperless.classifier] Gathering data from database...
DEBUG [paperless.classifier] 7 documents, 1 tag(s), 2 correspondent(s), 0 document type(s).
DEBUG [paperless.classifier] Vectorizing data...
DEBUG [paperless.classifier] Training tags classifier...
DEBUG [paperless.classifier] Training correspondent classifier...
DEBUG [paperless.classifier] There are no document types. Not training document type classifier.
INFO [paperless.tasks] Saving updated classifier model to /tmp/pngx_scratch/data/classification_model.pickle...
```

The persisted `data_hash` was captured as a 20-byte SHA-1 digest (40 hex chars):

```
data_hash (hex) = 083b30f0691d5fdbf30a18acdc96578f364fc92c
```

Length = **20 bytes** (SHA-1 ⇒ 20 bytes / 40 hex). *(ENV-SPECIFIC value; STABLE algorithm and length.)*

**SECOND run (unchanged data)** — the **SKIP** path:

```
DEBUG [paperless.classifier] Gathering data from database...
DEBUG [paperless.tasks] Training data unchanged.
```

No "Saving updated classifier model" line was emitted.

**THIRD run (after adding a document)** — a full retrain again: the `INFO`
`Saving updated classifier model to ...` line reappeared, and the new `data_hash` changed vs. the
first (changed = **True**):

```
data_hash (hex) = ebb42adf8cf57a694a4f50389a40facee82d08cd
```

This is exactly the "instant vs. long" framing: the skip path returns immediately, while the
retrain path refits the model.

### Code Rationale

- The change-detection hash is `m = hashlib.sha1()` at `src/documents/classifier.py:L124`, updated
  with the preprocessed content (`:L129`), the document-type pk (`:L136`), the correspondent pk
  (`:L143`), and the sorted auto-tag pks (`:L155`); it is finalized as `new_data_hash = m.digest()`
  at `:L161`.
- The skip decision is `if self.data_hash and new_data_hash == self.data_hash: return False` at
  `:L163-L164`. A full train assigns `self.data_hash = new_data_hash` (`:L247`) and returns `True`
  (`:L249`). The hash is persisted via pickle in `save()` (`:L96-L113`) to `settings.MODEL_FILE`.
- The log lines live in `train_classifier` — `src/documents/tasks.py:L48-L72`: the retrain `INFO`
  "Saving updated classifier model to {}…" at `:L64-L66`; the skip `DEBUG` "Training data
  unchanged." at `:L69`; and a failure `WARNING` "Classifier error: …" at `:L72`. Note the
  early-return guard (`:L49-L55`) requires at least one auto-matching (`MATCH_AUTO`)
  Tag / DocumentType / Correspondent to exist before training proceeds.
- `MODEL_FILE` is defined at `src/paperless/settings.py:L74`. The CLI entry point is
  `src/documents/management/commands/document_create_classifier.py`, whose `handle()` calls
  `train_classifier()`.
- **Determinism nuance:** the SHA-1 `data_hash` *value* depends on the full training corpus (each
  document's preprocessed content plus its document-type / correspondent pks plus sorted auto-tag
  pks), so the exact hex is **ENV-SPECIFIC**. The **algorithm (SHA-1, 20-byte digest / 40 hex)** and
  the skip/retrain decision itself are **STABLE**.

### Reproduction Steps

1. Seed at least one auto-matched (`MATCH_AUTO`) label so training proceeds past the guard.
2. Call `train_classifier()` → observe the build steps and "Saving updated classifier model…".
3. Call it again with unchanged data → observe "Training data unchanged." (the skip path).
4. Add or modify a document, then call again → observe a fresh save and a **changed** `data_hash`.

---


## Q4 — Duplicate Detection by Checksum

### Question

How can two files that "look completely different" both be rejected as duplicates? And what are the
actual checksum values being compared at the moment of rejection?

### How It Was Triggered

An existing `Document` (pk=1) was created with **distinct original and archive bytes**. An incoming
file was then submitted that is **byte-identical to the archive rendition** — and therefore looks
completely different from the original scan. *(Capture note: the in-memory channel layer was used so
that `Consumer._send_progress` worked without Redis/ASGI — an outside-the-repo settings choice only.)*

The probe **pins the exact input bytes** so the MD5 values below are reproducible verbatim by anyone.
The two stored renditions were:

```
original (scanner) bytes  = b"%PDF-1.4\nPaperless-ngx Q4 probe -- ORIGINAL scanner rendition.\n"
archive  (OCR)     bytes  = b"%PDF-1.4\nPaperless-ngx Q4 probe -- ARCHIVE OCR rendition.\n"
```

The incoming re-uploaded file was made **byte-identical to the archive rendition** above.

### Captured Evidence

The existing document pk=1 stored two checksums — each is `hashlib.md5(<bytes above>).hexdigest()`:

```
checksum         = 2f9eaec824a91b25bf48f739ead36dd4   (MD5 of the pinned "original" scanner bytes)
archive_checksum = 8bfd1e0d69f87157526dfb84d1caf3c8   (MD5 of the pinned "archive" rendition bytes)
```

*(STABLE: both MD5 values — reproduce them with `md5(<original bytes>)` and `md5(<archive bytes>)`
using the pinned byte literals above.)*

The incoming file's MD5 (it is byte-identical to the pinned archive rendition):

```
incoming MD5 = 8bfd1e0d69f87157526dfb84d1caf3c8   → equals existing archive_checksum (True)
```

**This is the crux:** the incoming MD5 is matched against **both** the `checksum` **and** the
`archive_checksum` columns. So a byte-different *archive* re-upload still collides with an existing
document, even though it shares no bytes with that document's original scan.

**With `CONSUMER_DELETE_DUPLICATES = False`** (the default): a `ConsumerError` is raised —
`rescan_of_archive.pdf: Not consuming rescan_of_archive.pdf: It is a duplicate.` — and the incoming
file is left in place (NOT deleted = True). The log line:

```
ERROR [paperless.consumer] Not consuming rescan_of_archive.pdf: It is a duplicate.
```

**With `CONSUMER_DELETE_DUPLICATES = True`:** the same `ConsumerError` and the same `ERROR` log line
are produced, but the incoming file is deleted via `os.unlink` (deleted = True).

**SHA-256 rejection evidence:** both checksum values above are 32 hex characters (MD5), and the
`checksum` / `archive_checksum` columns are `max_length=32`. A SHA-256 hex digest would be 64
characters — so SHA-256 is impossible here.

### Code Rationale

- `pre_check_duplicate` — `src/documents/consumer.py:L102-L113`: it computes
  `checksum = hashlib.md5(f.read()).hexdigest()` at `:L104`, then queries
  `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists()` at
  `:L105-L107`. The `Q(...) | Q(...)` is precisely why an incoming MD5 can match either column.
- On a hit: if `settings.CONSUMER_DELETE_DUPLICATES` is set, the file is removed via `os.unlink` at
  `:L109`; then `_fail(...)` raises a `ConsumerError` "Not consuming {filename}: It is a duplicate."
  at `:L110-L113`. The `ConsumerError` string is formatted `f"{self.filename}: {log_message or message}"`
  (`src/documents/consumer.py:L81`) — here `log_message` is the truthy "Not consuming … It is a
  duplicate." string, so the `or message` fallback is not used — which is why the exception text
  repeats the filename:
  `rescan_of_archive.pdf: Not consuming rescan_of_archive.pdf: It is a duplicate.`
- The columns are `checksum = CharField(max_length=32, unique=True)` at
  `src/documents/models.py:L135-L141` and `archive_checksum` likewise at `:L143-L150` — the
  32-character width confirms MD5.
- `CONSUMER_DELETE_DUPLICATES` defaults to `false` (`src/paperless/settings.py:L486`). The logger is
  `paperless.consumer`.
- **Version-honesty note:** at **this** commit the message is exactly
  `Not consuming {filename}: It is a duplicate.` Newer Paperless-ngx releases enrich it (e.g.
  "It is a duplicate of &lt;title&gt; (#&lt;pk&gt;).") — but the code is the source of truth for
  `542221a38dff`, and the commit-accurate string is the unenriched one above.

### Reproduction Steps

1. Create a document whose `checksum` and `archive_checksum` are the MD5s of the **pinned bytes**
   above — i.e. `checksum = md5(b"%PDF-1.4\nPaperless-ngx Q4 probe -- ORIGINAL scanner rendition.\n")`
   = `2f9eaec824a91b25bf48f739ead36dd4` and
   `archive_checksum = md5(b"%PDF-1.4\nPaperless-ngx Q4 probe -- ARCHIVE OCR rendition.\n")`
   = `8bfd1e0d69f87157526dfb84d1caf3c8`.
2. Submit a new file whose bytes equal the pinned **archive** rendition (different from the original
   scan); its MD5 is `8bfd1e0d69f87157526dfb84d1caf3c8`, matching the `archive_checksum` column.
3. Observe the `ConsumerError` and the `paperless.consumer` `ERROR` line.
4. Toggle `CONSUMER_DELETE_DUPLICATES` to show the incoming file preserved (`False`) versus
   `os.unlink`-deleted (`True`).
5. To regenerate the exact hexes anywhere, run:
   `python -c "import hashlib; print(hashlib.md5(b'%PDF-1.4\nPaperless-ngx Q4 probe -- ARCHIVE OCR rendition.\n').hexdigest())"`
   → `8bfd1e0d69f87157526dfb84d1caf3c8` (verbatim).

### Documentation corroboration

> *Corroboration (code remains authoritative):* A Paperless-ngx maintainer explicitly states that
> duplicates are detected using MD5 and that a detected duplicate means the files are bit-for-bit
> identical — directly corroborating the code and refuting the SHA-256 claim. The docs also state
> that with the default, "when the consumer detects a duplicate document, it will not touch the
> original document … Defaults to false."

---


## Q5 — Sanity Checker: Healthy vs Hash Mismatch

### Question

What does the sanity checker report for a **healthy** archive versus a **corrupted** one — including
the specific stored-vs-actual checksum values it prints when it discovers a mismatch?

### How It Was Triggered

`check_sanity()` was run against a **freshly-reset database** (isolation matters: leftover documents
from other probes would otherwise pollute the healthy check with messages like "Original of document
N does not exist"). It was run first with a healthy document (pk=1, whose real original, archive, and
thumbnail files have MD5s matching the stored checksums), then again after **overwriting the original
file's bytes on disk** so that its MD5 no longer matched the stored value.

The probe **pins the exact input bytes** so the stored/actual MD5 values below are reproducible
verbatim by anyone. The healthy original and the corrupting bytes were:

```
healthy original bytes  = b"%PDF-1.4\nPaperless-ngx Q5 probe -- HEALTHY original bytes.\n"
corrupting (overwrite)  = b"%PDF-1.4\nPaperless-ngx Q5 probe -- CORRUPTED original bytes.\n"
```

The stored `checksum` is `md5(<healthy bytes>)`; after the original file on disk is overwritten with
the corrupting bytes, the recomputed ("actual") MD5 is `md5(<corrupting bytes>)`.

### Captured Evidence

**HEALTHY** — `has_error=False`, `has_warning=False`, message count = 0:

```
INFO [paperless.sanity_checker] Sanity checker detected no issues.
```

**CORRUPTED** — `has_error=True`, message count = 1:

```
ERROR [paperless.sanity_checker] Checksum mismatch of document 1. Stored: d0c4264fa36808d43f596ddaa74c5570, actual: 8e1dfb095c8862c46bcb61c265726482.
```

Both values are 32-hex MD5 digests *(STABLE — reproduce them with `md5(<healthy bytes>)` and
`md5(<corrupting bytes>)` using the pinned byte literals above)*:

```
Stored: d0c4264fa36808d43f596ddaa74c5570   = md5(b"%PDF-1.4\nPaperless-ngx Q5 probe -- HEALTHY original bytes.\n")
actual: 8e1dfb095c8862c46bcb61c265726482   = md5(b"%PDF-1.4\nPaperless-ngx Q5 probe -- CORRUPTED original bytes.\n")
```

*(ENV-SPECIFIC: the document pk=1 and the absolute `source_path` `…/originals/0000001.pdf`.)*

### Code Rationale

- `check_sanity` — `src/documents/sanity_checker.py:L49-L133` — recomputes MD5 for the original at
  `:L83` and for the archive at `:L112`.
- The original-mismatch message is "Checksum mismatch of document {doc.pk}. Stored: {doc.checksum},
  actual: {checksum}." at `:L88-L91`. The archive-mismatch message is "Checksum mismatch of archived
  document {doc.pk}. Stored: {doc.archive_checksum}, actual: {checksum}." at `:L119-L124`.
- The healthy "no issues" `INFO` is emitted at `:L27`; the logger is `paperless.sanity_checker`
  (`:L24`).
- The task wrapper `sanity_check` — `src/documents/tasks.py:L255-L267` — converts the outcome into a
  string or an exception: an error ⇒
  `raise SanityCheckFailedException("Sanity check failed with errors. See log.")` (`:L261`); a
  warning ⇒ "Sanity check exited with warnings. See log." (`:L263`); and a clean run ⇒
  "No issues detected." (`:L267`). The CLI entry point is
  `src/documents/management/commands/document_sanity_checker.py`.

### Reproduction Steps

1. Seed a healthy document whose original file holds the pinned **healthy bytes**
   (`b"%PDF-1.4\nPaperless-ngx Q5 probe -- HEALTHY original bytes.\n"`) and whose stored `checksum`
   is their MD5 (`d0c4264fa36808d43f596ddaa74c5570`), with matching archive/thumbnail files and
   non-empty `content`; run `check_sanity()` → "Sanity checker detected no issues."
2. Overwrite the stored original on disk with the pinned **corrupting bytes**
   (`b"%PDF-1.4\nPaperless-ngx Q5 probe -- CORRUPTED original bytes.\n"`).
3. Re-run `check_sanity()` → the `ERROR` "Checksum mismatch of document {pk}. Stored: …, actual: …"
   with both MD5 values (`Stored: d0c4264fa36808d43f596ddaa74c5570`,
   `actual: 8e1dfb095c8862c46bcb61c265726482`).
4. To regenerate the exact hexes anywhere, run:
   `python -c "import hashlib; print(hashlib.md5(b'%PDF-1.4\nPaperless-ngx Q5 probe -- CORRUPTED original bytes.\n').hexdigest())"`
   → `8e1dfb095c8862c46bcb61c265726482` (verbatim).

### Documentation corroboration

> *Corroboration (code remains authoritative):* The administration docs enumerate the sanity
> checker's scope — missing/inaccessible originals and archives, corrupted originals/archives via
> checksum comparison, and missing/inaccessible thumbnails — matching `check_sanity`.

---


## Q6 — Orphaned / Ghost Media Files

### Question

Do orphaned files genuinely linger in the media folder, and what exactly does the system report when
it finds them?

### How It Was Triggered

A stray file that belongs to **no** `Document` was planted inside `MEDIA_ROOT`, and then
`check_sanity()` was run against a fresh database.

### Captured Evidence

The planted stray file:

```
/tmp/pngx_scratch/media/documents/originals/ZZZ_orphan_ghost.pdf
```

*(ENV-SPECIFIC path.)*

Running `check_sanity()` produced `has_error=False`, `has_warning=True`, message count = 1:

```
WARNING [paperless.sanity_checker] Orphaned file in media dir: /tmp/pngx_scratch/media/documents/originals/ZZZ_orphan_ghost.pdf
```

**Conclusion:** orphans **do linger** — the sanity checker does **not** delete them — and they are
surfaced as **WARNINGS**, not errors.

### Code Rationale

- `check_sanity` walks `settings.MEDIA_ROOT` to collect every present file at
  `src/documents/sanity_checker.py:L52-L55`, removes the media lock from that set at `:L57-L59`, and
  then removes each `Document`'s matched thumbnail/source/archive path. Any leftover file is reported
  as a `WARNING` "Orphaned file in media dir: {extra_file}" at `:L130-L131`.
- `MEDIA_ROOT` is defined at `src/paperless/settings.py:L61` and `MEDIA_LOCK` at `:L72`. The logger is
  `paperless.sanity_checker`.

### Reproduction Steps

1. Reset to a clean database and media store.
2. Drop a stray file under `MEDIA_ROOT/documents/originals/`.
3. Run `check_sanity()` → the `WARNING` "Orphaned file in media dir: …"; confirm the file is still
   present afterward (it is not deleted).

### Documentation corroboration

> *Corroboration (code remains authoritative):* The administration docs list orphaned files among the
> sanity checker's reported categories, consistent with the `WARNING` surfaced by `check_sanity`.

---


## Appendix — Cross-Cutting Insights

Five empirical insights emerge across the six behaviors:

1. **Silent success & silent rollback.** Neither a successful tag-driven move (Q1) nor a rollback
   (Q2) emits a dedicated log line. A reader watching the logs for a "moved file" message would
   wrongly conclude nothing happened; the real evidence is the on-disk before/after path delta (plus
   the `set_tags` `INFO` line during consumption).
2. **The `set_tags` "Tagging …" `INFO` fires only on the consumption pipeline.** It is wired in
   `src/documents/apps.py` `ready()` to `document_consumption_finished`, **not** on a manual ORM
   `tags.add()`. By contrast, `update_filename_and_move_files` self-registers via
   `@receiver(m2m_changed)` + `@receiver(post_save)` (`src/documents/signals/handlers.py:L310-L312`),
   so it always fires on tag changes and saves.
3. **`logger.fatal()` renders at `CRITICAL` level.** Q2's "File … has gone." appears as
   `CRITICAL [paperless.handlers]` because Python maps `fatal` to `CRITICAL`.
4. **MD5 (32 hex) for integrity/dedup vs. SHA-1 (40 hex) for classifier change-detection — SHA-256
   rejected.** The 32-character `checksum` / `archive_checksum` columns prove MD5; the classifier's
   SHA-1 `data_hash` is a separate, change-detection-only mechanism.
5. **Orphans linger as WARNINGS** — they are neither errors nor auto-deletions; the sanity checker
   reports them and leaves them in place.

### Determinism summary

| Category | Artifacts |
|----------|-----------|
| **STABLE** (content-derived; reproducible anywhere) | All MD5 values from Q4 (the original `checksum` and the `archive_checksum`) and Q5 (the stored and actual checksums). Because Q4 and Q5 **pin the exact input byte literals**, each value is `hashlib.md5(<pinned bytes>).hexdigest()` and a reader reproduces it **verbatim** anywhere — see the fenced capture below, where every hex is shown alongside the literal bytes that produce it. The SHA-1 algorithm plus its 20-byte / 40-hex length. All log message templates and logger names. The filename transform `Invoice.pdf` → `Invoice Paid.pdf`. The relative layout `documents/{originals,archive,thumbnails}/`. |
| **ENV-SPECIFIC** (varies per run) | Absolute paths (here under `/tmp/pngx_scratch`); document primary keys; timestamps and the `2026-06-26` date prefix in `str(document)`; and the SHA-1 `data_hash` *values* from Q3 (they depend on the full corpus plus label primary keys, which vary per run — so unlike the MD5s these are **not** reader-reproducible verbatim and are tagged ENV-SPECIFIC). |

The specific captured hash values referenced above. The four MD5s are reproducible **verbatim**
anywhere — each is the MD5 of the pinned byte literal printed beside it (run
`hashlib.md5(<bytes>).hexdigest()`). The two SHA-1 `data_hash` values are representative
ENV-SPECIFIC captures (they depend on env-assigned label primary keys, so re-running yields
*equivalent* 40-hex digests, not these exact ones):

```
STABLE       — MD5,   Q4 original checksum:          2f9eaec824a91b25bf48f739ead36dd4  = md5(b"%PDF-1.4\nPaperless-ngx Q4 probe -- ORIGINAL scanner rendition.\n")
STABLE       — MD5,   Q4 archive_checksum:           8bfd1e0d69f87157526dfb84d1caf3c8  = md5(b"%PDF-1.4\nPaperless-ngx Q4 probe -- ARCHIVE OCR rendition.\n")
STABLE       — MD5,   Q5 stored original checksum:   d0c4264fa36808d43f596ddaa74c5570  = md5(b"%PDF-1.4\nPaperless-ngx Q5 probe -- HEALTHY original bytes.\n")
STABLE       — MD5,   Q5 actual recomputed checksum: 8e1dfb095c8862c46bcb61c265726482  = md5(b"%PDF-1.4\nPaperless-ngx Q5 probe -- CORRUPTED original bytes.\n")
ENV-SPECIFIC — SHA-1, Q3 data_hash (first build):    083b30f0691d5fdbf30a18acdc96578f364fc92c  (representative; depends on env-assigned label pks)
ENV-SPECIFIC — SHA-1, Q3 data_hash (third run):      ebb42adf8cf57a694a4f50389a40facee82d08cd  (representative; depends on env-assigned label pks)
```

---

*This document describes a strictly read-only investigation. No file in the Paperless-ngx source
tree was modified, added, or deleted; all instrumentation lived outside the repository and was
removed after evidence capture.*

