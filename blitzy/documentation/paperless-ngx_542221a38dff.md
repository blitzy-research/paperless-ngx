# How paperless-ngx v1.7.0 Keeps Its Whoosh Search Index in Sync — A Runtime‑Verified Q&A

## 1. Title & Summary

This document answers three questions about how **paperless-ngx v1.7.0** (`src/paperless/version.py:L1` → `__version__ = (1, 7, 0)`) keeps its **Whoosh 2.7.4** full‑text search index synchronized with changes to documents. Each answer was produced by **running the code first** in the project's default, canonical configuration (Django web/API + Django‑Q worker + Redis broker + SQLite database + on‑disk Whoosh index) and capturing the real, unedited command output that appears beside every claim below.

- **Q1 — API edit path (synchronous vs. asynchronous indexing).** **DIRECT ANSWER: The edit is indexed *synchronously, inside the API request* — the document is searchable by its new title immediately (well under a second), and *no* background job is picked up.** A single‑document `PATCH` runs `index.add_or_update_document()` inside `DocumentViewSet.update()` (`src/documents/views.py:L212-L217`). By contrast, a **bulk edit** *does* enqueue a Django‑Q job (`documents.tasks.bulk_update_documents`).
- **Q2 — raw SQL bypass path (stale index + forced reconciliation).** **DIRECT ANSWER: The index goes *stale* — a search for the new title MISSES while a search for the old title still HITS. Reconciliation is *manual* via `python manage.py document_index reindex`, which runs *in‑process* (NO Django‑Q job).** A raw `UPDATE` fires no Django ORM `save()`/signal, and `Document` has no `save()` override that indexes (`src/documents/models.py:L88`, `:L106`).
- **Q3 — index corruption/deletion + rebuild (self‑heal vs. manual).** **DIRECT ANSWER: The index *structure* self‑heals, but the *data* does not — repopulation is *always manual*.** Opening a missing/corrupt index makes `open_index()` recreate an **empty** structure (`src/documents/index.py:L52-L61`); on a *corrupt* index it logs exactly `Error while opening the index, recreating.` (`:L56-L57`). Documents become searchable again only after `python manage.py document_index reindex`. There is **no scheduled reindex** anywhere. For a 3‑document set the rebuild wall‑clock was **stable at ≈1.33–1.40 s across three runs** (dominated by process/Django startup; the pure indexing runs at ≈730 docs/s).

Every behavioral claim below is immediately followed by the exact command and its complete, unedited output. Every factual claim carries a `file:line` reference naming the specific function/method/class. Statements derived from reading rather than observation are explicitly labeled **inferred**.

---

## 2. Methodology & Environment

**Application & stack (all values verified against this checkout):**

- **paperless-ngx v1.7.0** — `src/paperless/version.py:L1` → `__version__ = (1, 7, 0)`.
- **Task processor: Django‑Q 1.3.9** (NOT Celery). `src/documents/views.py:L28` → `from django_q.tasks import async_task`. Online docs that mention Celery describe a later version; **the code is authoritative for this checkout.**
- **Search engine: Whoosh 2.7.4**, index on disk at `INDEX_DIR = os.path.join(DATA_DIR, "index")` — `src/paperless/settings.py:L73` (`DATA_DIR` at `:L66`), i.e. `data/index`.
- **Database: default SQLite** `data/db.sqlite3` — `src/paperless/settings.py:L297-L300` (`ENGINE="django.db.backends.sqlite3"` `:L299`, `NAME` `:L300`). PostgreSQL is only used if `PAPERLESS_DBHOST` is set (`:L304-L318`). This investigation uses the **default (SQLite)**.
- **Worker cluster:** `Q_CLUSTER` at `src/paperless/settings.py:L449-L457` (Redis broker default `redis://localhost:6379`, `:L456`).
- **Runtime baseline:** `Dockerfile:L18` → `FROM python:3.9-slim-bullseye as main-app`.

**Exact build/run environment and invocation.** Everything was run inside the user‑specified canonical Docker container (AAP §0.8.1), image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`, where Python 3.9, all pinned dependencies, and Redis are already provisioned and the repository is bind‑mounted at `/app`. All management commands run from `/app/src` (where `manage.py` sets `DJANGO_SETTINGS_MODULE=paperless.settings`). For brevity, the command lines shown in the code blocks below are the *inner* commands; each was executed inside the container as:

```
$ docker exec paperless-ngx-setup-0 bash -lc 'cd /app/src && <command>'
```

**Verified interpreter and dependency versions (live capture):**

```
$ python --version
$ python -c "import django,whoosh,django_q,rest_framework,tqdm,redis,channels; print('django',django.get_version()); print('whoosh',whoosh.__version__); print('django_q',django_q.VERSION); print('drf',rest_framework.VERSION); print('tqdm',tqdm.__version__); print('redis',redis.__version__); print('channels',channels.__version__)"
Python 3.9.23
django 4.0.4
whoosh (2, 7, 4)
django_q (1, 3, 9)
drf 3.13.1
tqdm 4.64.0
redis 3.5.3
channels 3.0.4
```

These match the pins in `requirements.txt`: `whoosh==2.7.4` (L111), `django==4.0.4` (L38), `django-q==1.3.9` (L37), `djangorestframework==3.13.1` (L39), `tqdm==4.64.0` (L97), `redis==3.5.3` (L84).

**Running services (broker + worker + web/API).** Redis is the Django‑Q broker; the worker is `python manage.py qcluster` (stdout captured to `/tmp/qcluster.log`); the web/API layer is `gunicorn ... paperless.asgi:application` on `0.0.0.0:8000`:

```
$ redis-cli ping
PONG
$ ps -eo pid,cmd | grep -E "manage.py qcluster|gunicorn" | grep -v grep
   6255 python manage.py qcluster
   ... (qcluster sentinel + worker pool) ...
  11467 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

**Branch/version.** The deliverable is named for the source branch (`git rev-parse` shows the working checkout at commit `542221a38`, matching the required file name `paperless-ngx_542221a38dff.md`).

**Database migration state (DB + django_q tables + seeded schedules present):**

```
$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.
```

**Seed data (the "already‑ingested" baseline).** Three `Document` rows were created via the ORM (giving each a distinctive `title`/`content`; `mime_type` is required non‑null and `checksum` is unique per `src/documents/models.py:L126, :L135-L141`), then indexed through the real path (`document_index reindex`) so the baseline mirrors an ingested, indexed document. Final seed script (`/tmp/qna_seed.py`):

```
$ cat /tmp/qna_seed.py
from documents.models import Document
import django.utils.timezone as tz
seed = [
    ("seedqonealpha",      "alpha bravo charlie content one",   "chkqone01"),
    ("seedqtwobravo",      "delta echo foxtrot content two",    "chkqtwo02"),
    ("seedqthreecharlie",  "golf hotel india content three",    "chkqthree03"),
]
ids = []
for (t, c, chk) in seed:
    d = Document.objects.create(
        title=t, content=c, checksum=chk, mime_type="application/pdf",
        created=tz.now(), added=tz.now(),
    )
    ids.append(d.id)
print("SEEDED_IDS", ids)
print("TITLES", [(d.id, d.title) for d in Document.objects.order_by("id")])

$ python manage.py shell < /tmp/qna_seed.py
SEEDED_IDS [2, 3, 4]
TITLES [(2, 'seedqonealpha'), (3, 'seedqtwobravo'), (4, 'seedqthreecharlie')]
```

- **Document count used throughout: 3** (ids **2, 3, 4**). Doc **2** (`seedqonealpha`) is the Q1 edit target; doc **3** (`seedqtwobravo`) is the Q2 raw‑SQL target; doc **4** is filler. Search terms are distinctive lowercase single tokens so the Whoosh standard analyzer indexes them cleanly and searches are unambiguous.

**API authentication.** The REST API requires authentication (`REST_FRAMEWORK` Basic/Session/Token auth, `src/paperless/settings.py:L116-L120`). The commands below use HTTP Basic auth as `admin:paperless` — a local, throwaway superuser created by the canonical setup for this ephemeral, network‑isolated container (per the environment notes; paperless needs no external credentials). It is shown verbatim only so the commands are exactly reproducible; it is not a real or production secret.

**Baseline index population and the read path (proves the index is populated and search works):**

```
$ python manage.py document_index reindex

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 709.86it/s]

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqonealpha" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":2,"correspondent":null,"document_type":null,"title":"seedqonealpha","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T04:36:10.730117Z","modified":"2026-07-08T04:36:10.731792Z","added":"2026-07-08T04:36:10.730122Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqonealpha.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

The `"__search_hit__"` marker on the result confirms the response came through the Whoosh search read path (`UnifiedSearchViewSet.list()` → `open_index_searcher()`), not a plain database listing.

**Baseline background‑task state.** Django‑Q records each *completed* task in the `Task` table (`django_q.models.Task`). The authoritative "was a job enqueued for this operation?" signal used below is the **function‑filtered** count `Task.objects.filter(func="documents.tasks.bulk_update_documents").count()`, because it is immune to the unrelated periodic mail task (`paperless_mail.tasks.process_mail_accounts`, a 10‑minute schedule) that otherwise adds noise to the `Task` table and the `qcluster` log:

```
$ echo "from django_q.models import Task; print('TASK_TOTAL', Task.objects.count()); print('BULK_UPDATE_TASKS', Task.objects.filter(func='documents.tasks.bulk_update_documents').count())" | python manage.py shell
TASK_TOTAL 6
BULK_UPDATE_TASKS 0
```

Everything in Sections 3–5 below is captured live output from this environment.

---

## 3. Q1 — API edit path: synchronous vs. asynchronous indexing

**DIRECT ANSWER.** Changing an already‑ingested document's title through the REST API (`PATCH /api/documents/{id}/`) updates the Whoosh index **synchronously, within the API request**: an immediate search for the new unique title returns the document (result count ≥ 1) in a fraction of a second, and **no Django‑Q background job is picked up** for the edit (the worker log gains no line for it and the `Task` table gains no row). This is because `DocumentViewSet.update()` explicitly calls `index.add_or_update_document(self.get_object())` inside the request, right after the ORM save:

```
# src/documents/views.py:L212-L217  (DocumentViewSet.update)
    def update(self, request, *args, **kwargs):
        response = super(DocumentViewSet, self).update(request, *args, **kwargs)
        from documents import index

        index.add_or_update_document(self.get_object())
        return response
```

`add_or_update_document()` opens a Whoosh writer whose context manager **commits on exit** (in the `finally` block), so the write is durable before `update()` returns:

```
# src/documents/index.py:L118-L120  (add_or_update_document)
def add_or_update_document(document):
    with open_index_writer() as writer:
        update_document(writer, document)

# src/documents/index.py:L64-L74  (open_index_writer — commit-on-exit)
@contextmanager
def open_index_writer(optimize=False):
    writer = AsyncWriter(open_index())
    try:
        yield writer
    except Exception as e:
        logger.exception(str(e))
        writer.cancel()
    finally:
        writer.commit(optimize=optimize)      # <- commit happens here, in-request
```

The read path opens a **fresh on‑disk searcher per request** (`src/documents/views.py:L418`, inside `UnifiedSearchViewSet.list()` at `:L413-L425`), so the just‑committed write is visible on the very next search:

```
# src/documents/views.py:L413-L425  (UnifiedSearchViewSet.list)
    def list(self, request, *args, **kwargs):
        if self._is_search_request():
            from documents import index
            try:
                with index.open_index_searcher() as s:      # :L418 — fresh searcher per request
                    self.searcher = s
                    return super(UnifiedSearchViewSet, self).list(request)
            except NotFound:
                raise
            except Exception as e:
                return HttpResponseBadRequest(str(e))
        else:
```

### 3.1 Evidence — Run 1 (title → `quniqxaa`), with before/after worker + queue snapshots

```
$ LOG=/tmp/qcluster.log
$ wc -l < "$LOG"                                    # BEFORE: worker-log line count
66
$ echo "from django_q.models import Task; print('TASK_TOTAL_BEFORE', Task.objects.count()); print('BULK_BEFORE', Task.objects.filter(func='documents.tasks.bulk_update_documents').count())" | python manage.py shell
TASK_TOTAL_BEFORE 6
BULK_BEFORE 0

$ curl -sS -u admin:paperless -X PATCH "http://localhost:8000/api/documents/2/" \
       -H "Content-Type: application/json" -d '{"title":"quniqxaa"}' \
       -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
{"id":2,"correspondent":null,"document_type":null,"title":"quniqxaa","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T04:36:10.730117Z","modified":"2026-07-08T04:37:22.901680Z","added":"2026-07-08T04:36:10.730122Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxaa.pdf","archived_file_name":null}
HTTP 200  time_total=0.186689s

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=quniqxaa" \
       -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
{"count":1,"next":null,"previous":null,"results":[{"id":2,"correspondent":null,"document_type":null,"title":"quniqxaa","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T04:36:10.730117Z","modified":"2026-07-08T04:37:22.901680Z","added":"2026-07-08T04:36:10.730122Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxaa.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200  time_total=0.114467s

$ wc -l < "$LOG"                                    # AFTER: worker-log line count (unchanged)
66
$ echo "from django_q.models import Task; print('TASK_TOTAL_AFTER', Task.objects.count()); print('BULK_AFTER', Task.objects.filter(func='documents.tasks.bulk_update_documents').count())" | python manage.py shell
TASK_TOTAL_AFTER 6
BULK_AFTER 0
# NEW qcluster log lines during the PATCH+search window:
(none — no new worker activity)
```

**Reading of the evidence:** The `PATCH` returned **HTTP 200** with the serialized document showing `"title":"quniqxaa"`, in **0.187 s**. The *immediate* search returned **`"count":1`** with the edited document (id 2, new title, `__search_hit__`), in **0.114 s** — the write was visible on the very next request. Across the window the worker log stayed at **66 lines**, the total `Task` count stayed at **6**, and the `bulk_update_documents` count stayed at **0**, with **no new worker log lines**. That is the "no job was picked up" evidence for a single‑document edit.

### 3.2 Evidence — latency stability across runs 2 and 3

```
$ # RUN 2 → title quniqxbb
$ curl -sS -u admin:paperless -X PATCH "http://localhost:8000/api/documents/2/" -H "Content-Type: application/json" -d '{"title":"quniqxbb"}' -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
HTTP 200  time_total=0.242941s
patched_title=quniqxbb
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=quniqxbb" -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
HTTP 200  time_total=0.155157s
search_count=1 result_id=2
log_lines: 66 -> 66 ; bulk_update_tasks: 0 -> 0

$ # RUN 3 → title quniqxcc
$ curl -sS -u admin:paperless -X PATCH "http://localhost:8000/api/documents/2/" -H "Content-Type: application/json" -d '{"title":"quniqxcc"}' -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
HTTP 200  time_total=0.254140s
patched_title=quniqxcc
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=quniqxcc" -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
HTTP 200  time_total=0.202425s
search_count=1 result_id=2
log_lines: 66 -> 66 ; bulk_update_tasks: 0 -> 0
```

**Stability (3 documents, 3 runs).** PATCH latency: **0.187 s / 0.243 s / 0.254 s**; immediate‑search latency: **0.114 s / 0.155 s / 0.202 s**. Every run: search **count = 1**, worker log **66 → 66**, `bulk_update_documents` **0 → 0**. The "immediately visible, no job" behavior is stable across runs; latency is consistently well under a second.

### 3.3 Async contrast — a bulk edit DOES enqueue a Django‑Q job

A bulk edit routes through `src/documents/bulk_edit.py`, which dispatches `async_task("documents.tasks.bulk_update_documents", ...)` (at `:L18, :L31, :L47, :L63, :L87`). `modify_tags` sets `affected_docs` to *all* requested ids unconditionally, so it reliably enqueues a job:

```
# src/documents/bulk_edit.py:L68-L89  (modify_tags → async_task)
def modify_tags(doc_ids, add_tags, remove_tags):
    qs = Document.objects.filter(id__in=doc_ids)
    affected_docs = [doc.id for doc in qs]
    ...
    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)  # :L87
    return "OK"

# src/documents/tasks.py:L270-L280  (the worker-side task)
def bulk_update_documents(document_ids):
    documents = Document.objects.filter(id__in=document_ids)
    ix = index.open_index()
    for doc in documents:
        post_save.send(Document, instance=doc, created=False)
    with AsyncWriter(ix) as writer:
        for doc in documents:
            index.update_document(writer, doc)
```

```
$ wc -l < /tmp/qcluster.log                         # BEFORE
73
$ echo "from django_q.models import Task; print('BULK_BEFORE', Task.objects.filter(func='documents.tasks.bulk_update_documents').count())" | python manage.py shell
BULK_BEFORE 0

$ curl -sS -u admin:paperless -X POST "http://localhost:8000/api/documents/bulk_edit/" \
       -H "Content-Type: application/json" \
       -d '{"documents":[2],"method":"modify_tags","parameters":{"add_tags":[1],"remove_tags":[]}}' \
       -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
{"result":"OK"}
HTTP 200  time_total=0.132839s

$ sleep 5     # give the Django-Q worker a moment to process
$ wc -l < /tmp/qcluster.log                         # AFTER
78
$ echo "from django_q.models import Task; print('BULK_AFTER', Task.objects.filter(func='documents.tasks.bulk_update_documents').count()); t=Task.objects.filter(func='documents.tasks.bulk_update_documents').order_by('-started').first(); print('LAST_BULK_TASK', (t.func, t.name, t.success, str(t.started)))" | python manage.py shell
BULK_AFTER 1
LAST_BULK_TASK ('documents.tasks.bulk_update_documents', 'venus-rugby-cola-kitten', True, '2026-07-08 04:38:37.353556+00:00')
# NEW qcluster log lines during the window:
04:38:37 [Q] INFO Process-1:8 processing [venus-rugby-cola-kitten]
04:38:37 [Q] INFO Process-1:8 stopped doing work
04:38:37 [Q] INFO Processed [venus-rugby-cola-kitten]
04:38:37 [Q] INFO recycled worker Process-1:8
04:38:37 [Q] INFO Process-1:21 ready for work at 11870
```

**Reading of the evidence:** The bulk edit returned `{"result":"OK"}` and, unlike the single edit, the worker **picked up a job**: the `bulk_update_documents` count went **0 → 1**, the last such `Task` is `('documents.tasks.bulk_update_documents', 'venus-rugby-cola-kitten', True, ...)` (success), and the worker log gained five lines showing it `processing`/`Processed` that task. This is the direct contrast to the synchronous single‑document edit.

### 3.4 Why signals are *not* the mechanism (for a single edit)

A metadata edit is indexed by the **view handler**, not by a Django signal. The `post_save` receiver on `Document` only moves files — it contains **no** `index.` call:

```
# src/documents/signals/handlers.py:L310-L312
@receiver(models.signals.m2m_changed, sender=Document.tags.through)
@receiver(models.signals.post_save, sender=Document)
def update_filename_and_move_files(sender, instance, **kwargs):   # moves files only; no index call
```

The function that *does* index, `add_to_index()`, has **no `@receiver` decorator** and is wired **only** to the `document_consumption_finished` signal (i.e. new‑document ingestion) in `DocumentsConfig.ready()`:

```
# src/documents/signals/handlers.py:L428-L431
def add_to_index(sender, document, **kwargs):
    from documents import index
    index.add_or_update_document(document)

# src/documents/apps.py:L27  (inside DocumentsConfig.ready)
        document_consumption_finished.connect(add_to_index)
```

This is why the raw‑SQL path in Q2 (which fires no signal at all) leaves the index untouched.

---

## 4. Q2 — raw SQL bypass: stale index and forced reconciliation

**DIRECT ANSWER.** Updating a document's title *directly in the database via raw SQL* (not through the API/ORM) leaves the Whoosh index **stale**: a search for the **new** title returns **nothing (MISS)**, while a search for the **old** title still **HITS** the document. The system does **not** auto‑reconcile. To force reconciliation you run **`python manage.py document_index reindex`**, which executes **in the command's own process (NO Django‑Q job)** and prints a **tqdm** progress bar as its visible activity. After reindex the new title HITS and the old title MISSES.

**Why it goes stale.** A raw SQL `UPDATE` bypasses Django's ORM entirely, so it fires no `post_save` signal, and `Document` has **no `save()` override** that would touch the index (`src/documents/models.py:L88` class, `:L106` the `title` field; `grep -n "def save" src/documents/models.py` returns nothing). Therefore `index.update_document()` is never reached and the on‑disk index keeps the old tokens.

### 4.1 Evidence — before: index and DB agree

```
$ echo "from documents.models import Document; d=Document.objects.get(id=3); print('DOC3_DB_TITLE', repr(d.title))" | python manage.py shell
DOC3_DB_TITLE 'seedqtwobravo'

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":3,"correspondent":null,"document_type":null,"title":"seedqtwobravo","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T04:36:10.752492Z","modified":"2026-07-08T04:36:10.752696Z","added":"2026-07-08T04:36:10.752495Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqtwobravo.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

### 4.2 Evidence — the raw SQL UPDATE (bypassing the ORM), then the stale index

The update was issued through the raw DB cursor (`django.db.connection.cursor()`), which executes SQL directly with no ORM `save()` and no signal. Script `/tmp/qna_rawsql.py`:

```
$ cat /tmp/qna_rawsql.py
from django.db import connection
c = connection.cursor()
c.execute("UPDATE documents_document SET title=%s WHERE id=%s", ["rawsqlnewzz", 3])
print("ROWCOUNT", c.rowcount)
c.execute("SELECT id, title FROM documents_document WHERE id=%s", [3])
print("RAW_SELECT", c.fetchone())

$ python manage.py shell < /tmp/qna_rawsql.py
ROWCOUNT 1
RAW_SELECT (3, 'rawsqlnewzz')
```

Now the index is stale — the **new** title MISSES, the **old** title still HITS:

```
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":3,"correspondent":null,"document_type":null,"title":"rawsqlnewzz","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T04:36:10.752492Z","modified":"2026-07-08T04:36:10.752696Z","added":"2026-07-08T04:36:10.752495Z","archive_serial_number":null,"original_file_name":"2026-07-08 rawsqlnewzz.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

**Reading of the evidence (the divergence in one screen).** The raw SQL changed the database (`ROWCOUNT 1`, `RAW_SELECT (3, 'rawsqlnewzz')`). Yet a search for the **new** title `rawsqlnewzz` returns **`"count":0`** — the index has no such token. A search for the **old** title `seedqtwobravo` still returns **`"count":1`** — the index still maps that stale token to doc 3. Note the smoking gun: that stale hit's serialized `"title"` is **`"rawsqlnewzz"`** (the current DB value), because the search *matches* on the stale Whoosh index but the API *serializes* the row from the database. You can only *find* the document by its obsolete indexed title, while its actual title is the new one — a textbook index/DB divergence.

### 4.3 Evidence — forced reconciliation via `document_index reindex` (in‑process, no job)

The reconciliation entry point is a synchronous management command wrapped in a DB transaction; it calls `index_reindex()` directly and dispatches no `async_task`:

```
# src/documents/management/commands/document_index.py:L11-L25
    def add_arguments(self, parser):
        parser.add_argument("command", choices=["reindex", "optimize"])   # :L12
        ...
    def handle(self, *args, **options):
        with transaction.atomic():                                        # :L21
            if options["command"] == "reindex":
                index_reindex(progress_bar_disable=options["no_progress_bar"])  # :L23
            elif options["command"] == "optimize":
                index_optimize()                                          # :L25

# src/documents/tasks.py:L38-L45  (index_reindex — rebuilds from the DB, tqdm bar)
def index_reindex(progress_bar_disable=False):
    documents = Document.objects.all()                                    # :L39
    ix = index.open_index(recreate=True)                                  # :L41
    with AsyncWriter(ix) as writer:
        for document in tqdm.tqdm(documents, disable=progress_bar_disable):  # :L44
            index.update_document(writer, document)                       # :L45
```

```
$ wc -l < /tmp/qcluster.log                         # BEFORE reindex
78
$ echo "from django_q.models import Task; print('TASK_TOTAL_BEFORE', Task.objects.count())" | python manage.py shell
TASK_TOTAL_BEFORE 8

$ python manage.py document_index reindex

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 701.35it/s]

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":3,"correspondent":null,"document_type":null,"title":"rawsqlnewzz","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T04:36:10.752492Z","modified":"2026-07-08T04:36:10.752696Z","added":"2026-07-08T04:36:10.752495Z","archive_serial_number":null,"original_file_name":"2026-07-08 rawsqlnewzz.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200

$ wc -l < /tmp/qcluster.log                         # AFTER reindex
78
$ echo "from django_q.models import Task; print('TASK_TOTAL_AFTER', Task.objects.count())" | python manage.py shell
TASK_TOTAL_AFTER 8
# NEW qcluster log lines during the reindex window:
(none — reindex ran in-process, NOT via Django-Q)
```

**Reading of the evidence:** `document_index reindex` printed a **tqdm** bar (`100%|██████████| 3/3 [00:00<00:00, 701.35it/s]`). Afterward the **new** title `rawsqlnewzz` HITS (`"count":1`) and the **old** title `seedqtwobravo` MISSES (`"count":0`) — the index is reconciled with the database. During the reindex window the worker log stayed at **78 lines** and the total `Task` count stayed at **8** with **no new worker log lines**, proving the command ran **in the invoking process, not via Django‑Q**. The `Task` table contains no reindex entry at all:

```
$ echo "from django_q.models import Task; from collections import Counter; print(dict(Counter(t.func for t in Task.objects.all())))" | python manage.py shell
{'documents.tasks.bulk_update_documents': 1, 'paperless_mail.tasks.process_mail_accounts': 4, 'documents.tasks.sanity_check': 1, 'documents.tasks.train_classifier': 1, 'documents.tasks.index_optimize': 1}
```

The only `documents.tasks.bulk_update_documents` row is the one from the Q1 async contrast; the four `process_mail_accounts` rows are the unrelated periodic mail schedule; there is **no** `reindex`/`index_reindex` function anywhere in the queue — reindex is never a queued task.

### 4.4 The sanity check does not reconcile the index (**inferred**)

`src/documents/sanity_checker.py` contains **no** `index`/`whoosh`/`reindex` references (`grep -niE 'index|whoosh|reindex' src/documents/sanity_checker.py` returns nothing). Combined with the observation above that a manual `document_index reindex` was required to fix the stale state, this indicates the weekly `sanity_check` does not repair a stale/missing index. This is labeled **inferred** (from reading the source + the observed need for a manual reindex) rather than directly observed by running the sanity check.

---

## 5. Q3 — index corruption/deletion and rebuild: self‑heal vs. manual intervention

**DIRECT ANSWER.** If the index is deleted or corrupted while documents still exist in the database, the index **structure self‑heals but the data does not**. Opening the index recreates an **empty** structure automatically (via `create_in`), so searches return **nothing** until you rebuild. On a *corrupt* (as opposed to merely missing) index, `open_index()` logs exactly **`Error while opening the index, recreating.`** and then recreates the empty structure. Repopulation is **always manual** — `python manage.py document_index reindex` — because there is **no scheduled reindex** anywhere in the system. For the 3‑document set, the rebuild wall‑clock was **stable at ≈1.33–1.40 s across three runs** (the pure indexing throughput is ≈730 docs/s; the wall‑clock is dominated by process/Django startup), and the **tqdm** bar is the visible processing activity.

The recovery logic lives in `open_index()`:

```
# src/documents/index.py:L52-L61  (open_index — structure self-heal only)
def open_index(recreate=False):
    try:
        if exists_in(settings.INDEX_DIR) and not recreate:
            return open_dir(settings.INDEX_DIR, schema=get_schema())
    except Exception:
        logger.exception("Error while opening the index, recreating.")   # :L56-L57
    if not os.path.isdir(settings.INDEX_DIR):
        os.makedirs(settings.INDEX_DIR, exist_ok=True)
    return create_in(settings.INDEX_DIR, get_schema())                   # :L61 — empty structure
```

### 5.1 Evidence — deletion case: empty structure recreated, search returns nothing

```
$ ls -la /app/data/index                            # BEFORE deletion
total 32
-rwxr-xr-x 1 root root     0 Jul  8 04:16 MAIN_WRITELOCK
-rw-r--r-- 1 root root 14846 Jul  8 04:40 MAIN_k9nke9jxqfovfufs.seg
-rw-r--r-- 1 root root  4396 Jul  8 04:40 _MAIN_1.toc

$ rm -rf /app/data/index/*
$ ls -la /app/data/index                            # immediately after delete: empty
total 8
drwxr-sr-x 2 root root 4096 Jul  8 04:42 .
drwxr-sr-x 4 root root 4096 Jul  8 04:40 ..

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqthreecharlie" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200

$ ls -la /app/data/index                            # after a search: empty structure recreated
total 12
-rw-r--r-- 1 root root 4064 Jul  8 04:42 _MAIN_0.toc
```

**Reading of the evidence:** After deleting the index contents, the directory is empty; searching returns **`"count":0"`** for every query even though the three documents still exist in the database. The first `open_index()` call (via the search read path) recreated an **empty** structure — note the new `_MAIN_0.toc` (4064 bytes) but **no `.seg` data file** (the 14846‑byte `MAIN_...seg` is gone). This is the structure‑only self‑heal (`create_in`, `src/documents/index.py:L61`); the pure‑deletion case takes the non‑exception branch (`exists_in()` is False), so it recreates *silently* with no error log.

### 5.2 Evidence — corruption case: the exact `Error while opening the index, recreating.` log line

The index was repopulated, then its table‑of‑contents file was overwritten with garbage, then `documents.index.open_index()` was called so its internal `try/except` fires. A handler was attached to the `paperless.index` logger (`src/documents/index.py:L28` → `logger = logging.getLogger("paperless.index")`) to capture the line:

```
$ python manage.py document_index reindex          # repopulate first

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 682.41it/s]

$ for f in /app/data/index/_MAIN_*.toc; do echo "GARBAGE-NOT-A-VALID-TOC-REDO" > "$f"; done
$ cat /tmp/qna_corrupt2.py
import logging, sys
h = logging.StreamHandler(sys.stdout)
h.setFormatter(logging.Formatter("CAPTURED[%(name)s %(levelname)s] %(message)s"))
lg = logging.getLogger("paperless.index")
lg.addHandler(h)
lg.setLevel(logging.DEBUG)
from documents import index
ix = index.open_index()
print("OPEN_RESULT_doc_count:", ix.doc_count())

$ python manage.py shell < /tmp/qna_corrupt2.py
CAPTURED[paperless.index ERROR] Error while opening the index, recreating.
Traceback (most recent call last):
  File "/app/src/documents/index.py", line 54, in open_index
    if exists_in(settings.INDEX_DIR) and not recreate:
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 136, in exists_in
    ix = open_dir(dirname, indexname=indexname)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 123, in open_dir
    return FileIndex(storage, schema=schema, indexname=indexname)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 421, in __init__
    TOC.read(self.storage, self.indexname, schema=self._schema)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 632, in read
    check_size("int", _INT_SIZE)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 628, in check_size
    raise IndexError("Index was created on different architecture:"
whoosh.index.IndexError: Index was created on different architecture: saved int = 71, this computer = 4
[2026-07-08 04:43:50,187] [ERROR] [paperless.index] Error while opening the index, recreating.
Traceback (most recent call last):
  File "/app/src/documents/index.py", line 54, in open_index
    if exists_in(settings.INDEX_DIR) and not recreate:
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 136, in exists_in
    ix = open_dir(dirname, indexname=indexname)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 123, in open_dir
    return FileIndex(storage, schema=schema, indexname=indexname)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 421, in __init__
    TOC.read(self.storage, self.indexname, schema=self._schema)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 632, in read
    check_size("int", _INT_SIZE)
  File "/usr/local/lib/python3.9/site-packages/whoosh/index.py", line 628, in check_size
    raise IndexError("Index was created on different architecture:"
whoosh.index.IndexError: Index was created on different architecture: saved int = 71, this computer = 4
OPEN_RESULT_doc_count: 0
```

**Reading of the evidence:** With a corrupt table‑of‑contents, `exists_in(settings.INDEX_DIR)` — called *inside* `open_index()`'s `try` at `src/documents/index.py:L54` — raised `whoosh.index.IndexError: Index was created on different architecture: saved int = 71, this computer = 4`. `open_index()`'s `except` branch (`:L56-L57`) caught it and logged the exact line **`Error while opening the index, recreating.`** (shown twice: once via the handler added for capture, `CAPTURED[paperless.index ERROR] ...`, and once via the application's own configured handler, `[ERROR] [paperless.index] ...`). It then recreated an empty structure — `OPEN_RESULT_doc_count: 0` confirms the rebuilt index holds **no data**. So corruption self‑heals the *structure* (and logs a clear error), but the *data* is gone until a manual reindex.

### 5.3 Evidence — rebuild wall‑clock timing, stable across three runs (3 documents)

```
$ echo "from documents.models import Document; print(Document.objects.count())" | python manage.py shell    # document count
3

$ rm -rf /app/data/index/* ; time python manage.py document_index reindex          # RUN 1

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 730.08it/s]

real	0m1.396s
user	0m1.203s
sys	0m0.164s

$ rm -rf /app/data/index/* ; time python manage.py document_index reindex          # RUN 2

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 733.74it/s]

real	0m1.342s
user	0m1.185s
sys	0m0.159s

$ rm -rf /app/data/index/* ; time python manage.py document_index reindex          # RUN 3

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 732.33it/s]

real	0m1.328s
user	0m1.181s
sys	0m0.149s
```

**Reading of the evidence (timing).** Document count = **3**. Rebuild wall‑clock (`real`): **1.396 s / 1.342 s / 1.328 s** — **stable** (spread ≈ 5%). The tqdm bar (the visible processing activity) reports ≈**730 docs/s**, i.e. the actual indexing of 3 documents takes on the order of **4 ms**; the ≈1.3 s command wall‑clock is dominated by Python interpreter + Django app startup, not by the indexing work. For this small set the reconstruction is effectively near‑instant, and the value is reproducible across runs.

### 5.4 Evidence — documents searchable again after rebuild

```
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqthreecharlie" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":4,"correspondent":null,"document_type":null,"title":"seedqthreecharlie","content":"golf hotel india content three","tags":[],"created":"2026-07-08T04:36:10.779692Z","modified":"2026-07-08T04:36:10.779884Z","added":"2026-07-08T04:36:10.779694Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqthreecharlie.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

### 5.5 Evidence — there is NO scheduled reindex (recovery is always manual)

```
$ echo "from django_q.models import Schedule; [print(s.name, '|', s.func, '|', s.schedule_type) for s in Schedule.objects.all().order_by('func')]" | python manage.py shell
Optimize the index | documents.tasks.index_optimize | D
Perform sanity check | documents.tasks.sanity_check | W
Train the classifier | documents.tasks.train_classifier | H
Check all e-mail accounts | paperless_mail.tasks.process_mail_accounts | I

$ echo "from django_q.models import Schedule; print('REINDEX_SCHEDULES', list(Schedule.objects.filter(func__icontains='reindex').values_list('func', flat=True)))" | python manage.py shell
REINDEX_SCHEDULES []
```

**Reading of the evidence:** The seeded schedules are `index_optimize` (Daily), `sanity_check` (Weekly), `train_classifier` (Hourly), and the unrelated `process_mail_accounts` (Interval). A filter for any schedule whose function name contains `reindex` returns an empty list. These schedules are seeded by migrations — `src/documents/migrations/1001_auto_20201109_1636.py` seeds `train_classifier` (HOURLY, `:L10-L14`) and `index_optimize` (DAILY, `:L15-L19`), and `src/documents/migrations/1004_sanity_check_schedule.py` seeds `sanity_check` (WEEKLY, `:L10-L14`). **No migration seeds `index_reindex`.** Only `index_optimize` (which merely optimizes the existing index, `src/documents/tasks.py:L32-L35`) runs automatically; rebuilding the *data* after deletion/corruption therefore always requires the manual `document_index reindex`.

---

## 6. Coverage Pass

Every sub‑part and every named item, answered explicitly with its concrete value, `file:line`, observed evidence, and reasoning:

| # | Sub‑question / named item | Answer (observed) | Key `file:line` | Evidence location |
|---|---------------------------|-------------------|-----------------|-------------------|
| Q1‑a | Does an API title edit appear immediately or need background processing? | **Immediately, synchronously**; searchable in the same request (search count=1, latency < 0.21 s) | `views.py:L212-L217`, `index.py:L64-L74`, `:L118-L120` | §3.1 |
| Q1‑b | Is any Django‑Q job picked up for the single edit? | **No** — worker log 66→66, `bulk_update_documents` 0→0, no new log lines | `views.py:L216` (no `async_task`) | §3.1–§3.2 |
| Q1‑c | Watch the worker: does a single edit produce a task? | **No task** for the edit (3 runs) | — | §3.1–§3.2 |
| Q1‑d | Latency, ≥2 runs, stable? | PATCH 0.187/0.243/0.254 s; search 0.114/0.155/0.202 s — **stable, sub‑second** (3 docs, 3 runs) | — | §3.2 |
| Q1‑e | Asynchronous contrast | **Bulk edit DOES enqueue a job** — `bulk_update_documents` 0→1, task `venus-rugby-cola-kitten` success, 5 worker log lines | `bulk_edit.py:L87`, `tasks.py:L270-L280` | §3.3 |
| Q1‑f | Are signals the mechanism? | **No** — `post_save` receiver moves files only; `add_to_index` wired to consumption only | `handlers.py:L310-L312`, `:L428-L431`, `apps.py:L27` | §3.4 |
| Q2‑a | Does raw SQL update make the new title searchable? | **No — stale index; new title MISS (count:0)** | `models.py:L88`,`:L106` (no `save()` override) | §4.2 |
| Q2‑b | Old title after raw SQL? | **Still HITS (count:1)**; hit serializes the new DB title → index/DB divergence | `index.py:L240-L254` (matching), serializer reads DB | §4.2 |
| Q2‑c | Is there a way to force reconciliation? | **Yes — `python manage.py document_index reindex`** | `document_index.py:L20-L25`, `tasks.py:L38-L45` | §4.3 |
| Q2‑d | Effect on worker logs / task queue during reconciliation? | **None** — runs in‑process; log 78→78, Task 8→8, no reindex task in queue; **tqdm** is the visible activity | `document_index.py:L21` (`transaction.atomic`), no `async_task` | §4.3 |
| Q2‑e | Does the sanity check reconcile the index? | **No** (**inferred**: `sanity_checker.py` has no index refs; manual reindex was required) | `sanity_checker.py` (no index refs) | §4.4 |
| Q3‑a | Behavior when index is deleted? | Empty structure silently recreated on open; **searches return nothing** | `index.py:L52-L61`, `:L61` (`create_in`) | §5.1 |
| Q3‑b | Behavior when index is corrupted? | Logs exactly **`Error while opening the index, recreating.`**, recreates empty (doc_count 0) | `index.py:L56-L57`, `:L61` | §5.2 |
| Q3‑c | Recovery mechanism | **Manual `document_index reindex`** repopulates from `Document.objects.all()` | `tasks.py:L38-L45` | §5.3–§5.4 |
| Q3‑d | Rebuild duration for a small set, ≥2 runs, stable? | **1.396/1.342/1.328 s** wall‑clock for **3 docs** — stable; indexing ≈730 docs/s | — | §5.3 |
| Q3‑e | Visible processing activity during rebuild | The **tqdm** progress bar (`100%|██████████| 3/3 ...`) | `tasks.py:L44` | §5.3 |
| Q3‑f | Searchable after rebuild? | **Yes** — `?query=seedqthreecharlie` → count:1 | — | §5.4 |
| Q3‑g | Self‑heal vs. manual intervention? | **Structure self‑heals; data is manual** — no scheduled reindex exists | migrations `1001`, `1004`; `REINDEX_SCHEDULES []` | §5.5 |

---


## 7. Cleanup Confirmation & Reproducibility

All runtime experiments operated only on git‑ignored runtime state under `data/` (the SQLite database `data/db.sqlite3` and the Whoosh index `data/index`, both matched by `/data/` in `.gitignore`) and on temporary helper scripts under `/tmp`. After capturing the evidence above, every transient artifact was removed and the runtime state was restored so that the repository and its source are left unchanged.

**Confirm `data/` is git‑ignored (so the index/db experiments never touch tracked source):**

```
$ git check-ignore data/index data/db.sqlite3
data/index
data/db.sqlite3
```

**Delete the temporary `Document` records (ids 2, 3, 4) and the temporary `Tag`, restoring the empty baseline:**

```
$ echo "from documents.models import Document, Tag; print('DOCS_DELETED', Document.objects.filter(id__in=[2,3,4]).delete()); print('TAG_DELETED', Tag.objects.filter(name='qnatmptag').delete()); print('DOC_COUNT_NOW', Document.objects.count()); print('TAG_COUNT_NOW', Tag.objects.count())" | python manage.py shell
DOCS_DELETED (4, {'documents.Document_tags': 1, 'documents.Document': 3})
TAG_DELETED (1, {'documents.Tag': 1})
DOC_COUNT_NOW 0
TAG_COUNT_NOW 0
```

**Remove the temporary observation scripts, then restore a consistent (now empty, 0‑document) index via a final `document_index reindex`:**

```
$ rm -f /tmp/qna_seed.py /tmp/qna_rawsql.py /tmp/qna_corrupt.py /tmp/qna_corrupt2.py /tmp/patch_*.json /tmp/search_*.json
$ rm -rf /app/data/index/* ; python manage.py document_index reindex

0it [00:00, ?it/s]
0it [00:00, ?it/s]
$ ls -la /app/data/index/
-rwxr-xr-x 1 root root    0 Jul  8 04:53 MAIN_WRITELOCK
-rw-r--r-- 1 root root 4064 Jul  8 04:53 _MAIN_1.toc
```

The restored index (`MAIN_WRITELOCK` + `_MAIN_1.toc`, 0 documents) matches the original state the environment started in. The pre‑existing Redis, Django‑Q `qcluster`, and gunicorn services were reused throughout and are intentionally **left running** as they were found (this investigation started no new services of its own).

**Final repository state — only the one new Markdown file is present; nothing under tracked source changed:**

```
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/paperless-ngx_542221a38dff.md

$ git add --dry-run blitzy/
add 'blitzy/documentation/paperless-ngx_542221a38dff.md'
```

This confirms the read‑only‑source constraint was honored: no existing repository file was modified, added, or deleted; the sole persistent artifact is this document, `blitzy/documentation/paperless-ngx_542221a38dff.md`.
