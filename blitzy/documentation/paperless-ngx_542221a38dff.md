# paperless-ngx — Document Flow Knowledge Capture

**Repository:** paperless-ngx · **Branch:** `paperless-ngx_542221a38dff` · **HEAD:** `542221a38dff06361e07976452f9aea24d210542`
**Project version:** `(1, 7, 0)` — `src/paperless/version.py:L1`

This document answers seven questions about how a document flows through paperless-ngx v1.7.0: how it enters the system (Q1), the stages it passes through (Q2), the background-execution engine (Q3), the metadata stored per document (Q4), how those fields are classified as required / optional / derived (Q5), a concrete runtime example (Q6), and how tags, correspondents, and document types organize documents together (Q7). Every factual claim carries an exact `file:line` reference and is labeled **[observed]** (produced by running real code and captured first-hand) or **[inferred]** (read from source, not executed here). This was a strictly read-only investigation; no repository file was modified.

---

## Environment & Methodology

- **Canonical runtime used for observation.** The project's canonical runtime is the Docker image `python:3.9-slim-bullseye` — `Dockerfile:L18` (`FROM python:3.9-slim-bullseye as main-app`). The investigation was performed **inside that canonical runtime**: container `paperless-app`, **Python 3.9.23**, with every dependency at the exact version pinned in `requirements.txt` (django `4.0.4` `:L38`, django-q `1.3.9` `:L37`, channels `3.0.4` `:L23`, channels-redis `3.4.0` `:L22`, redis `3.5.3` `:L84`, scikit-learn `1.0.2` `:L88`, whoosh `2.7.4` `:L111`, watchdog `2.1.7` `:L106`, python-magic `0.4.25` `:L79`, fuzzywuzzy[speedup] `0.18.0` `:L41`, imap-tools `0.54.0` `:L49`, dateparser `1.1.1` `:L32`).
- **Isolated harness, outside the repository.** A minimal Django settings harness plus two observation scripts were placed in the container's `/tmp` (never inside the repository checkout, which is bind-mounted at `/app`). The harness declares only `django_q` + the real `documents` app, points the classifier `MODEL_FILE` at a non-existent path (so `load_classifier()` returns `None`, exercising the pure **rule-based** organization path), and uses a throwaway SQLite database in `/tmp`. This loads the **real** `documents` models and signal handlers unchanged.
- **Run first, then write.** Each **[observed]** section below shows the exact command and its **unedited** captured output. All stability-sensitive values (the 16-field count, the md5 checksum, every matching boolean, the schedule constants) were confirmed **identical across two runs** (`diff` reported no differences).
- **What is inferred and why.** Q1 (the three ingestion entry points end-to-end) and Q2 (the full multi-stage consume pipeline: OCR/Tesseract, thumbnailing, real parsers, and the Redis-backed `qcluster` worker), the broker/worker half of Q3, and the ML `MATCH_AUTO` classifier path of Q7 are labeled **[inferred from source]**. Executing those requires orchestrating the full asynchronous consume with real scanned inputs, running `qcluster` workers, and Tesseract OCR — beyond the scope of a read-only investigation. They are grounded in exact `file:line` references and are never asserted as observed outcomes. The `document_consumption_finished` step of Q2 is, however, demonstrated live in Q6.
- **Read-only discipline.** All harness scripts and the throwaway database/index lived outside the repository and were deleted on completion; the checkout is byte-for-byte unchanged apart from this one document.

---

## Q1 — How does a new document enter paperless-ngx? (ingestion entry points) — [inferred from source]

A new document normally enters through **one of three canonical entry points**. Each one writes the incoming bytes to a file and then enqueues the **same** background task, `documents.tasks.consume_file`, onto the django-q broker via `async_task(...)` (imported as `from django_q.tasks import async_task`). None of these three paths was executed here (each requires the full asynchronous runtime); all claims are read from source.

**1. Consumption-directory watcher** — `src/documents/management/commands/document_consumer.py`
- Imports the task API: `from django_q.tasks import async_task` — `:L13`.
- The management command watches `CONSUMPTION_DIR` (via watchdog's `PollingObserver`, imported `:L17`; directory default wired at `:L150`). When a stable file appears, `def _consume(filepath)` — `:L46` — validates it and enqueues:
  - `async_task(` — `:L86`
  - `"documents.tasks.consume_file",` — `:L87`

**2. REST API upload** — `src/documents/views.py`
- Imports `from django_q.tasks import async_task` — `:L28`.
- `class PostDocumentView(GenericAPIView)` — `:L491`; its `def post(...)` — `:L497` — streams the uploaded file into a temporary file (`tempfile.NamedTemporaryFile(...)` — `:L512`) and enqueues:
  - `async_task(` — `:L523`
  - `"documents.tasks.consume_file",` — `:L524`

**3. Mail fetch** — `src/paperless_mail/mail.py`
- Imports `from django_q.tasks import async_task` — `:L11`.
- For each processed mail attachment it writes a temp file (`tempfile.mkstemp(...)` — `:L322`) and enqueues:
  - `async_task(` — `:L336`
  - `"documents.tasks.consume_file",` — `:L337`
- This handler is itself driven periodically by the scheduled task `paperless_mail.tasks.process_mail_accounts` (see Q3).

**Convergence point.** All three enqueue the one task `def consume_file(...)` — `src/documents/tasks.py:L184`. That task constructs a `Consumer` and calls `Consumer().try_consume_file(...)` — `src/documents/tasks.py:L236` — which runs the pipeline described in Q2. Because every entry point converges here, the processing behavior (Q2) and the organization behavior (Q6/Q7) are identical regardless of how the document arrived.

---

## Q2 — What stages does a document pass through before it is fully processed? — [inferred from source]

The pipeline lives in `Consumer.try_consume_file(...)` — `src/documents/consumer.py:L180`. Throughout, progress is broadcast over WebSockets via `Consumer._send_progress(current, max, status, message)` — `src/documents/consumer.py:L56` (see Q3 for the channel). The status message constants are defined at `src/documents/consumer.py:L44-L49` (e.g. `MESSAGE_FINISHED = "finished"` — `:L49`). A complete end-to-end run (real OCR/Tesseract, thumbnailing, and the Redis-backed worker) requires the full Docker runtime and was not executed here; the stages and their exact lines are read from source. The stages, in order:

1. **STARTING (0%).** `self._send_progress(0, 100, "STARTING", MESSAGE_NEW_FILE)` — `:L202`. The overridable inputs (`override_filename`, `override_title`, `override_correspondent_id`, `override_document_type_id`, `override_tag_ids`, `task_id`) are captured from the signature at `:L180-L189`; `self.filename` becomes `override_filename or os.path.basename(path)` — `:L195`.
2. **Pre-checks.** `pre_check_file_exists()` — `:L95`; `pre_check_duplicate()` (rejects a file whose md5 checksum already exists) — `:L102`; `pre_check_directories()` (ensures the scratch/media directories exist) — `:L115`.
3. **MIME detection and parser selection.** `mime_type = magic.from_file(self.path, mime=True)` — `:L219` (python-magic). `parser_class = get_parser_class_for_mime_type(mime_type)` — `:L223`; if none is found, `if not parser_class:` — `:L224` — the file is rejected as an unsupported type. Parser resolution helpers live in `src/documents/parsers.py`: `is_mime_type_supported` `:L43`, `get_parser_class_for_mime_type` `:L81`, and the parser's `parse(...)` method `:L307`.
4. **`document_consumption_started` signal.** `document_consumption_started.send(...)` — `:L229`.
5. **Optional pre-consume script.** `self.run_pre_consume_script()` — `:L235` (runs `PRE_CONSUME_SCRIPT` if configured).
6. **Parsing (20%).** `self._send_progress(20, 100, "WORKING", MESSAGE_PARSING_DOCUMENT)` — `:L259`, then `document_parser.parse(self.path, mime_type, self.filename)` — `:L261`. (During parsing the parser reports fine-grained progress through a callback rescaled into the 20–80% band — `:L237-L240`.)
7. **Thumbnail (70%).** `self._send_progress(70, 100, "WORKING", MESSAGE_GENERATING_THUMBNAIL)` — `:L264`, then `get_optimised_thumbnail(...)` — `:L265`.
8. **Text and date extraction.** `text = document_parser.get_text()` — `:L271`; `date = document_parser.get_date()` — `:L272`.
9. **Missing-date branch (edge condition).** `if not date:` — `:L273` → `self._send_progress(90, 100, "WORKING", MESSAGE_PARSE_DATE)` — `:L274` → `date = parse_date(self.filename, text)` — `:L275`. This fallback (using `src/documents/parsers.py:parse_date` `:L212`, which relies on `dateparser`) runs **only** when the parser did not itself return a date. The archive path is then obtained: `archive_path = document_parser.get_archive_path()` — `:L276`.
10. **Classifier prepared.** `classifier = load_classifier()` — `:L292` (`src/documents/classifier.py:load_classifier` `:L30`). When no trained model file exists this returns `None`, and organization proceeds purely rule-based (this is exactly the condition demonstrated in Q6).
11. **Save (95%), inside a transaction.** `self._send_progress(95, 100, "WORKING", MESSAGE_SAVE_DOCUMENT)` — `:L294`; `with transaction.atomic():` — `:L298`.
    - `document = self._store(text=text, date=date, mime_type=mime_type)` — `:L301` (`Consumer._store` `:L379`) creates the `Document` row. Explicit consumption-time overrides are applied by `Consumer.apply_overrides(document)` — `:L414`, which sets `correspondent` / `document_type` / `tags` from any `override_*_id` values passed in (these take precedence over automatic organization).
    - `document_consumption_finished.send(sender=..., document=document, logging_group=..., classifier=classifier)` — `:L306` — fires the organization signal (Q6/Q7). **This exact signal is demonstrated live in Q6.**
    - `with FileLock(settings.MEDIA_LOCK):` — `:L315` (filelock). Inside the lock: `document.filename = generate_unique_filename(document)` — `:L316` (`src/documents/file_handling.py:generate_unique_filename` `:L81`, backed by `generate_filename` `:L128`); the original and thumbnail are written; if an archive was produced, `document.archive_filename` is assigned and `document.archive_checksum = hashlib.md5(...).hexdigest()` is computed — `:L338-L340`.
    - `document.save()` — `:L346`; the consumed source file is removed with `os.unlink(self.path)` — `:L350`.
12. **Cleanup + optional post-consume script.** `document_parser.cleanup()` runs in `finally` — `:L369`; then `self.run_post_consume_script(document)` — `:L371` (runs `POST_CONSUME_SCRIPT` if configured).
13. **SUCCESS (100%).** `self._send_progress(100, 100, "SUCCESS", MESSAGE_FINISHED, document.id)` — `:L375`; the method returns the `Document` — `:L377`.

After this pipeline the document is fully processed: stored with a unique filename, indexed and organized (Q6/Q7), and available through the API/UI.

---

## Q3 — Are there background jobs, and what runs them? — [observed + inferred]

**The background-execution engine is django-q (version `1.3.9`), *not* Celery.** This version of paperless-ngx predates the project's later migration to Celery; the evidence is the pinned dependency `django-q==1.3.9` — `requirements.txt:L37` — together with the `Q_CLUSTER` configuration block and the `qcluster` worker process.

**Configuration** — `src/paperless/settings.py`:
- `django_q` is an installed app: `"django_q",` — `:L110`.
- The task cluster is configured by `Q_CLUSTER` — `:L449` — with `"name": "paperless"` `:L450`, `"workers": TASK_WORKERS` `:L455`, and a Redis broker `"redis": os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")` `:L456`. `TASK_WORKERS` is defined at `:L438`. The worker process that consumes these tasks is started with `python3 manage.py qcluster`.
- Real-time consumption progress (emitted by `Consumer._send_progress`, Q2) is broadcast over Django Channels: `CHANNEL_LAYERS` — `:L178-L182` — uses `"BACKEND": "channels_redis.core.RedisChannelLayer"` `:L180` with `"hosts": [os.getenv("PAPERLESS_REDIS", "redis://localhost:6379")]` `:L182`. The channel group name is `"status_updates"` (used at `src/documents/consumer.py:L74` and `src/documents/tasks.py:L227`).

**Ad-hoc (on-demand) tasks** in `src/documents/tasks.py`: `consume_file` `:L184` (the ingestion task from Q1), `bulk_update_documents` `:L270`, plus the scheduled-task functions `index_optimize` `:L32`, `train_classifier` `:L48`, and `sanity_check` `:L255`.

**Scheduled tasks.** These are registered as django-q `Schedule` rows by data migrations that call `django_q.tasks.schedule(...)`:

| Scheduled task | Frequency | Registered in |
|----------------|-----------|---------------|
| `documents.tasks.train_classifier` | HOURLY | `src/documents/migrations/1001_auto_20201109_1636.py` (func `:L11`, `schedule_type=Schedule.HOURLY` `:L13`) |
| `documents.tasks.index_optimize` | DAILY | `src/documents/migrations/1001_auto_20201109_1636.py` (func `:L16`, `schedule_type=Schedule.DAILY` `:L18`) |
| `documents.tasks.sanity_check` | WEEKLY | `src/documents/migrations/1004_sanity_check_schedule.py` (func `:L11`, `schedule_type=Schedule.WEEKLY` `:L13`) |
| `paperless_mail.tasks.process_mail_accounts` | every 10 MINUTES | `src/paperless_mail/migrations/0002_auto_20201117_1334.py` (func `:L11`, `schedule_type=Schedule.MINUTES` `:L13`, `minutes=10` `:L14`) |

**[observed] — schedule type constants.** Command (run inside the canonical container, Python 3.9.23):

```bash
PYTHONPATH=/app/src:/tmp DJANGO_SETTINGS_MODULE=ppsettings python3 -c \
  "import django; django.setup(); from django_q.models import Schedule; \
   print(Schedule.HOURLY, Schedule.DAILY, Schedule.WEEKLY, Schedule.MINUTES)"
```

Output:

```text
H D W I
```

So `HOURLY='H'`, `DAILY='D'`, `WEEKLY='W'`, `MINUTES='I'`.

**[observed] — the `Schedule` rows the migrations actually create.** After running `migrate` on a throwaway database with only the `documents` app installed, the harness enumerated `django_q.models.Schedule`. Command:

```bash
PYTHONPATH=/app/src:/tmp DJANGO_SETTINGS_MODULE=ppsettings python3 /tmp/q6_signal.py
```

Relevant unedited output (rows ordered by `func` for a deterministic listing):

```text
Q3 [observed] django-q Schedule rows created by migrations:
  func=documents.tasks.index_optimize schedule_type=D minutes=None
  func=documents.tasks.sanity_check schedule_type=W minutes=None
  func=documents.tasks.train_classifier schedule_type=H minutes=None

Q3 [observed] Schedule type constants:
  HOURLY='H' DAILY='D' WEEKLY='W' MINUTES='I'
```

Cause → effect: applying the `documents` migrations registered exactly three schedules — `index_optimize` (`D`), `sanity_check` (`W`), and `train_classifier` (`H`) — matching the table above. `paperless_mail.tasks.process_mail_accounts` (`schedule_type='I'`, `minutes=10`) is **not** in this listing only because the `paperless_mail` app was outside the minimal harness's installed apps; its schedule is confirmed from the migration source cited above, and the constant `MINUTES='I'` it uses is observed here directly.

**[inferred]** The Redis-backed broker (`redis://localhost:6379`) and the `qcluster` worker actually executing these tasks end-to-end require the full Docker runtime and were not run here.

---

## Q4 — What metadata fields are stored for each document? — [observed]

The fields were read from the **real** `Document` model — `src/documents/models.py:L88` — at runtime through Django's `_meta` API (not by eyeballing declarations). Command:

```bash
PYTHONPATH=/app/src:/tmp DJANGO_SETTINGS_MODULE=ppsettings python3 /tmp/introspect.py
```

Unedited output (field-inventory section):

```text
========================================================================
Q4/Q5 - Document model fields via Django _meta API [observed]
========================================================================
total fields reported by _meta.get_fields(): 16
field                 type              null  blank  editable default
---------------------------------------------------------------------
id                    AutoField         False True   True     —
correspondent         ForeignKey        True  True   True     —
title                 CharField         False True   True     —
document_type         ForeignKey        True  True   True     —
content               TextField         False True   True     —
mime_type             CharField         False False  False    —
checksum              CharField         False False  False    —
archive_checksum      CharField         True  True   False    —
created               DateTimeField     False False  True     now
modified              DateTimeField     False True   False    —
storage_type          CharField         False False  False    unencrypted
added                 DateTimeField     False False  False    now
filename              FilePathField     True  False  False    None
archive_filename      FilePathField     True  False  False    None
archive_serial_number IntegerField      True  True   True     —
tags                  ManyToManyField   False True   True     —

Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):
  - mime_type (CharField, editable=False)
  - checksum (CharField, editable=False)
```

**The model stores 16 fields** — the implicit `id` primary key plus 15 declared fields. Each field with its declaration line in `src/documents/models.py`:

| # | Field | Type | Declared at | Purpose |
|---|-------|------|-------------|---------|
| 1 | `id` | AutoField | (implicit PK) | Primary key |
| 2 | `correspondent` | ForeignKey → Correspondent | `:L97` | Who the document is from/to |
| 3 | `title` | CharField(128) | `:L106` | Human-readable title |
| 4 | `document_type` | ForeignKey → DocumentType | `:L108` | Kind of document |
| 5 | `content` | TextField | `:L117` | Extracted full text (for search) |
| 6 | `mime_type` | CharField(256) | `:L126` | Original MIME type |
| 7 | `checksum` | CharField(32), unique | `:L135` | md5 of the original file |
| 8 | `archive_checksum` | CharField(32) | `:L143` | md5 of the archived (PDF/A) file |
| 9 | `created` | DateTimeField | `:L152` | Document date |
| 10 | `modified` | DateTimeField (`auto_now`) | `:L154` | Last-modified timestamp |
| 11 | `storage_type` | CharField(11) | `:L161` | `unencrypted` or `gpg` |
| 12 | `added` | DateTimeField | `:L169` | When paperless ingested it |
| 13 | `filename` | FilePathField(1024), unique | `:L176` | Current original filename in storage |
| 14 | `archive_filename` | FilePathField(1024), unique | `:L186` | Current archive filename in storage |
| 15 | `archive_serial_number` | IntegerField, unique | `:L196` | Physical-archive position (ASN) |
| 16 | `tags` | ManyToManyField → Tag | `:L128` | Zero or more tags |

---

## Q5 — Which fields are required vs optional vs derived? — [observed]

Using the same `_meta` introspection (the `null` / `blank` / `editable` / `default` columns above), the harness computed which fields are strictly required at the model layer — those that are **not null**, **not blank**, have **no default**, and are not the auto primary key. The observed result:

```text
Strictly REQUIRED at model layer (not null & not blank & no default & not AutoField):
  - mime_type (CharField, editable=False)
  - checksum (CharField, editable=False)
```

Only **two** fields are strictly required. Classifying every field by the observed attributes:

| Classification | Fields (with declaration line) | Why (observed attributes) |
|----------------|-------------------------------|---------------------------|
| **REQUIRED** | `mime_type` `:L126`, `checksum` `:L135` | `null=False`, `blank=False`, no default; both `editable=False` (set by the consumer, not the user). `checksum` is additionally `unique`. |
| **OPTIONAL** | `correspondent` `:L97`, `document_type` `:L108`, `title` `:L106`, `archive_serial_number` `:L196`, `tags` `:L128` | FKs are `null=True, blank=True`; `title` is `blank=True` (stores `''` when unset); `archive_serial_number` is `null=True, blank=True`; `tags` (M2M) is `blank=True`. |
| **DERIVED** (computed by the consumer/parser, `editable=False`) | `content` `:L117`, `archive_checksum` `:L143`, `filename` `:L176`, `archive_filename` `:L186` | `content` is the parser's extracted text; `archive_checksum` is the md5 of the produced archive; `filename` / `archive_filename` (`default=None`, `null=True`) are assigned during storage by `generate_unique_filename`. |
| **DEFAULT / AUTO** | `created` `:L152`, `added` `:L169`, `storage_type` `:L161`, `modified` `:L154` | `created` and `added` default to `timezone.now` (shown as `now`); `storage_type` defaults to `unencrypted`; `modified` uses `auto_now=True` (`editable=False`) — note `_meta` shows no `.default` for it because `auto_now` is a distinct mechanism, hence the `—` in the output. |

> Note: `content` and `title` are `null=False` but `blank=True`, so they are *stored* even when empty (as `''`), not omitted — they are optional/derived for the user but always present in the row.

**[observed] — metadata derivation edge (missing-date / default naming).** `FileInfo.from_filename` — `src/documents/models.py` (class `FileInfo` at `:L386`, classmethod `from_filename` at `:L434`) — derives `title` and `created` from a filename. Same command (`/tmp/introspect.py`); unedited output:

```text
========================================================================
Q4/Q5 edge - FileInfo.from_filename metadata derivation [observed]
========================================================================
from_filename('invoice_acme.pdf'): title='invoice_acme' created=None
from_filename('20221101Z - Some Title.pdf'): title='Some Title' created=datetime.datetime(2022, 11, 1, 0, 0, tzinfo=tzlocal())
from_filename('just a scan.pdf'): title='just a scan' created=None
```

Cause → effect: with default naming the whole filename stem becomes the `title` and `created` is `None` (e.g. `invoice_acme.pdf` → `title='invoice_acme'`, `created=None`; `just a scan.pdf` → `title='just a scan'`, `created=None`). Because `created` is `None`, the consumer falls back to the parser-extracted date or, failing that, `parse_date` (Q2's missing-date branch at `consumer.py:L273-L275`). Only the special `YYYYMMDDZ - Title` pattern is parsed into a real `created` datetime (`20221101Z - Some Title.pdf` → `title='Some Title'`, `created=2022-11-01 00:00`).


---

## Q6 — Concrete runtime example — [observed]

This exercises the **real** organization path end-to-end by firing the actual `document_consumption_finished` signal — the same signal `Consumer` sends at `src/documents/consumer.py:L306` — with the six handlers connected in `DocumentsConfig.ready()`:

```text
document_consumption_finished.connect(add_inbox_tags)     # src/documents/apps.py:L22
document_consumption_finished.connect(set_correspondent)  # :L23
document_consumption_finished.connect(set_document_type)  # :L24
document_consumption_finished.connect(set_tags)           # :L25
document_consumption_finished.connect(set_log_entry)      # :L26
document_consumption_finished.connect(add_to_index)       # :L27
```

The harness seeded a `consumer` user and five organizing entities, created a `Document` whose `content` is set exactly as `Consumer._store` assigns parser text, printed the before-state, fired the signal (no `classifier` kwarg → `None` → **pure rule-based**, exactly the fresh-instance case from Q2 step 10), and printed the after-state. Command:

```bash
PYTHONPATH=/app/src:/tmp DJANGO_SETTINGS_MODULE=ppsettings python3 /tmp/q6_signal.py
```

Seeded entities: `Correspondent(name="ACME Corp", match="acme", MATCH_ANY)`; `DocumentType(name="Invoice", match="invoice", MATCH_ANY)`; `Tag(name="Inbox", is_inbox_tag=True)`; `Tag(name="Paid", match="total due", MATCH_ALL)`; `Tag(name="Travel", match="flight hotel", MATCH_ALL)`. Document content: `"Invoice 2022 from ACME Corp, total due 199.00 EUR"`.

Unedited captured output:

```text
document created: id=1 checksum=dbd97f5b73b9094ecd35ca4298857559
BEFORE signal: correspondent=None document_type=None tags=[]
AFTER  signal: correspondent=ACME Corp document_type=Invoice tags=['Inbox', 'Paid']
set_log_entry -> LogEntry rows for doc=1 user=consumer action_flag=1
add_to_index -> whoosh index doc ids: [1]
```

The checksum `dbd97f5b73b9094ecd35ca4298857559` is the md5 of the exact content string (independently cross-checked: `python3 -c "import hashlib; print(hashlib.md5(b'Invoice 2022 from ACME Corp, total due 199.00 EUR').hexdigest())"` → `dbd97f5b73b9094ecd35ca4298857559`).

Cause → effect — naming each handler and its source line in `src/documents/signals/handlers.py`:

- **`add_inbox_tags`** — `:L30` — added `Inbox` because that tag has `is_inbox_tag=True` (it does `Tag.objects.filter(is_inbox_tag=True)` and adds them unconditionally, regardless of any match rule).
- **`set_correspondent`** — `:L35` — matched the rule `match="acme"` (MATCH_ANY) against the content and set `correspondent = ACME Corp`.
- **`set_document_type`** — `:L101` — matched `match="invoice"` (MATCH_ANY) and set `document_type = Invoice`.
- **`set_tags`** — `:L168` — matched `Paid` (`match="total due"`, MATCH_ALL — both words present) and added it; `Travel` (`match="flight hotel"`, MATCH_ALL) did **not** match and was excluded. Final tag set: `['Inbox', 'Paid']`.
- **`set_log_entry`** — `:L413` — wrote a Django admin `LogEntry` with `action_flag=1` (ADDITION) as the `consumer` user (observed: `LogEntry rows for doc=1 user=consumer action_flag=1`).
- **`add_to_index`** — `:L428` — indexed the document into the Whoosh full-text index (observed: `whoosh index doc ids: [1]`).

The before/after transition (`correspondent=None, document_type=None, tags=[]` → `correspondent=ACME Corp, document_type=Invoice, tags=['Inbox','Paid']`) is the concrete, observed demonstration of automatic organization at consumption time.

---

## Q7 — How do tags, correspondents, and document types organize documents together? — [observed + inferred]

Organization is built on one abstract base, `MatchingModel` — `src/documents/models.py:L19` — and its three concrete subclasses. A document links to **one** correspondent (FK), **one** document type (FK), and **many** tags (M2M) — see the Q4 field table (`correspondent` `:L97`, `document_type` `:L108`, `tags` `:L128`).

**The base `MatchingModel`** (`:L19`) defines the matching contract shared by all three entities:
- Algorithm constants: `MATCH_ANY = 1` `:L21`, `MATCH_ALL = 2` `:L22`, `MATCH_LITERAL = 3` `:L23`, `MATCH_REGEX = 4` `:L24`, `MATCH_FUZZY = 5` `:L25`, `MATCH_AUTO = 6` `:L26`.
- `match = models.CharField(...)` `:L39` (the pattern text); `matching_algorithm = models.PositiveIntegerField(..., default=MATCH_ANY)` `:L41` (default at `:L44`); `is_insensitive = models.BooleanField(..., default=True)` `:L47`.

> Citation note: the `match` field is at `:L39` (line 21 is `MATCH_ANY = 1`).

**The three concrete entities** (each named individually):
- **Correspondent** — `class Correspondent(MatchingModel)` — `:L57`. One per document (FK, optional). Represents the party a document is from/to.
- **Document type** — `class DocumentType(MatchingModel)` — `:L82`. One per document (FK, optional). Represents the kind of document (e.g. "Invoice").
- **Tag** — `class Tag(MatchingModel)` — `:L64`. Many per document (M2M). Adds one field beyond the base: `is_inbox_tag = models.BooleanField(..., default=False)` — `:L68` (default confirmed `False` below).

**Automatic assignment at consumption.** The six handlers connected in `src/documents/apps.py:L22-L27` (see Q6) drive assignment. For correspondents, document types, and tags the work is done by `match_correspondents` `:L21`, `match_document_types` `:L34`, and `match_tags` `:L47` in `src/documents/matching.py`. Each **combines** two sources: an ML classifier prediction (used **only** when a `MATCH_AUTO` rule exists) and the rule-based filter `matches(matching_model, document)` — `src/documents/matching.py:L60`. For example `match_correspondents` returns `filter(lambda o: matches(o, document) or o.pk == pred_id, ...)` — the rule filter OR the predicted id.

**[observed] — every matching algorithm, plus the empty-match edge.** The harness exercised the real `matches()` against the content `"Invoice 2022 from ACME Corp, total due 199.00 EUR"`. Command:

```bash
PYTHONPATH=/app/src:/tmp DJANGO_SETTINGS_MODULE=ppsettings python3 /tmp/introspect.py
```

Unedited output:

```text
========================================================================
Q7 - REAL matches() across EVERY algorithm + empty-match [observed]
========================================================================
document.content = 'Invoice 2022 from ACME Corp, total due 199.00 EUR'

MATCH_ANY 'acme zzz'       -> True
MATCH_ALL 'acme invoice'   -> True
MATCH_ALL 'acme missing'   -> False
MATCH_LITERAL 'ACME Corp'  -> True
MATCH_REGEX '\d{3}\.\d{2}' -> True
MATCH_FUZZY 'akme korp'    -> False
MATCH_AUTO 'acme'          -> False
MATCH_ANY '' (empty)       -> False

MatchingModel algorithm constants [observed]:
  MATCH_ANY=1 MATCH_ALL=2 MATCH_LITERAL=3 MATCH_REGEX=4 MATCH_FUZZY=5 MATCH_AUTO=6
  Tag.is_inbox_tag default: False
```

Each result tied to its branch in `src/documents/matching.py`:

- **Empty match** → `False`. `if matching_model.match.strip() == "":` `:L66` → `return False` `:L67`. (This is why the `Inbox` tag in Q6, whose `match` is empty, is never assigned by rule matching — it is added by `add_inbox_tags` instead.)
- **Case-insensitivity.** `if matching_model.is_insensitive:` `:L69` sets `{"flags": re.IGNORECASE}` `:L70`, applied to all regex-based algorithms below.
- **MATCH_ALL** (`:L72`) → `'acme invoice'` **True**, `'acme missing'` **False**. Every whitespace-split word (via `_split_match` `:L155`) must be found as `\bword\b`; the loop returns `False` on the first missing word.
- **MATCH_ANY** (`:L84`) → `'acme zzz'` **True**. Returns `True` on the first word found (`acme`); the unrelated `zzz` is irrelevant.
- **MATCH_LITERAL** (`:L91`) → `'ACME Corp'` **True**. Searches `rf"\b{re.escape(matching_model.match)}\b"` `:L94`, so the whole string is matched as an escaped, word-bounded, case-insensitive substring.
- **MATCH_REGEX** (`:L107`) → `'\d{3}\.\d{2}'` **True**. The pattern is compiled and `re.search`ed; it finds `199.00`. Invalid patterns are caught: `except re.error:` `:L113` → `return False`.
- **MATCH_FUZZY** (`:L127`) → `'akme korp'` **False**. Uses `fuzz.partial_ratio(match, text) >= 90` `:L135`; the misspelling scores below the 90 threshold. (On this canonical image the `fuzzywuzzy[speedup]` extra is installed, so no "slow pure-python SequenceMatcher" warning is emitted; the result is unaffected.)
- **MATCH_AUTO** (`:L147`) → **False**. `matches()` deliberately returns `False` here — `# this is done elsewhere.` `:L148` / `return False` `:L149` — because automatic matching is delegated to the ML classifier, not to `matches()`.

**How the three entities work together — addressing each by name:**

1. **Tags — the Inbox flow.** `add_inbox_tags` (`handlers.py:L30`) always adds every `is_inbox_tag=True` tag (default `is_inbox_tag=False`, confirmed observed above) to a newly consumed document, independent of any `match` rule. This is why, in Q6, `Inbox` appears even though its `match` is empty (which `matches()` returns `False` for). Other tags are assigned by `set_tags` (`handlers.py:L168`) via `match_tags` — a document can carry **many** tags (`Paid` matched, `Travel` did not).
2. **Correspondents — one per document.** `set_correspondent` (`handlers.py:L35`) assigns at most one via `match_correspondents`; in Q6 the `acme` (MATCH_ANY) rule selected `ACME Corp`.
3. **Document types — one per document.** `set_document_type` (`handlers.py:L101`) assigns at most one via `match_document_types`; in Q6 the `invoice` (MATCH_ANY) rule selected `Invoice`.
4. **Rule-based vs ML-based.** All three entities share the same dual mechanism: the rule-based `matches()` filter (observed above for every algorithm) **or** an ML prediction when a `MATCH_AUTO` rule is configured.

**[inferred]** The `MATCH_AUTO` → classifier path uses `src/documents/classifier.py` (`load_classifier` `:L30`, `predict_correspondent` `:L251`, `predict_document_type` `:L262`, `predict_tags` `:L273`) with a scikit-learn `1.0.2` model. On a fresh instance no trained model file exists, so `load_classifier()` returns `None` and only rule-based matching runs (exactly the path observed in Q6/Q7). Exercising a real trained-model prediction requires the full Docker runtime and was not run here.

The concrete, observed demonstration of all of the above working together is the Q6 result: `correspondent=ACME Corp`, `document_type=Invoice`, `tags=['Inbox', 'Paid']`.

---

## Observed vs Inferred — summary

| Question / claim | Label | Basis |
|------------------|-------|-------|
| Q1 — three ingestion entry points converge on `consume_file` | [inferred from source] | End-to-end async ingestion (watcher/REST/mail → django-q) not run; grounded in `file:line`. |
| Q2 — full `try_consume_file` pipeline (OCR, thumbnail, storage) | [inferred from source] | Real OCR/Tesseract + `qcluster` require the Docker runtime; the `document_consumption_finished` step is shown live in Q6. |
| Q3 — engine is django-q; `Schedule` constants (`H/D/W/I`) and rows | [observed] | Run on canonical Python 3.9.23 (constants + migrated-DB rows). |
| Q3 — Redis broker + `qcluster` executing tasks end-to-end | [inferred from source] | Requires the full runtime. |
| Q4 — 16 `Document` fields | [observed] | Django `_meta` introspection of the real model. |
| Q5 — required (`mime_type`, `checksum`) vs optional/derived/default | [observed] | Same `_meta` attributes; plus `FileInfo.from_filename` derivation edge. |
| Q6 — organization signal before/after, LogEntry, Whoosh index | [observed] | Real `document_consumption_finished` fired with the six real handlers. |
| Q7 — every matching algorithm + empty-match; entity cardinality | [observed] | Real `matches()` across all algorithms. |
| Q7 — `MATCH_AUTO` → scikit-learn classifier path | [inferred from source] | No trained model on a fresh instance; requires the Docker runtime. |

**Reproduction & cleanup.** All observations were produced inside the canonical container (`paperless-app`, Python 3.9.23) using a minimal, isolated Django settings harness and two scripts (`introspect.py`, `q6_signal.py`) placed in the container's `/tmp`, with a throwaway SQLite database and Whoosh index also under `/tmp` — all **outside** the repository checkout. Every stability-sensitive value (the 16-field count, the checksum `dbd97f5b73b9094ecd35ca4298857559`, every matching boolean, and the schedule constants) was confirmed identical across two runs. All temporary scripts, the throwaway database, and the index were deleted on completion; the repository is unchanged apart from this single document.

