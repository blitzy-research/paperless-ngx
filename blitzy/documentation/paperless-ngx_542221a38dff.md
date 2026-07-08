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

**Exact build/run environment and invocation.** Everything was run inside the user‑specified canonical Docker container (AAP §0.8.1), image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`, running as the container named `paperless-ngx-setup-0`, where Python 3.9, all pinned dependencies, and Redis are already provisioned and the repository is bind‑mounted at `/app`. Django management commands run from `/app/src` (where `manage.py` sets `DJANGO_SETTINGS_MODULE=paperless.settings`); source‑file inspections (`grep`, `ls`) run from the repository root `/app`. Every command was executed inside the container via `docker exec`. Here is one fully‑formed invocation, shown in its entirety with its real output, to make the exact pattern concrete:

```
$ docker exec paperless-ngx-setup-0 bash -lc 'cd /app/src && python --version'
Python 3.9.23
```

**Convention for the code blocks below.** So the identical `docker exec paperless-ngx-setup-0 bash -lc '…'` wrapper need not be repeated on every line, each block below shows the *inner* command — the part after `cd /app/src &&` (or after `cd /app &&` for source‑file inspections such as `grep`/`ls`) — immediately followed by its complete, unedited output. Every inner command was run through exactly the wrapper shown above.

**Verified dependency versions (live capture):**

```
$ python -c "import django,whoosh,django_q,rest_framework,tqdm,redis,channels; print('django',django.get_version()); print('whoosh',whoosh.__version__); print('django_q',django_q.VERSION); print('drf',rest_framework.VERSION); print('tqdm',tqdm.__version__); print('redis',redis.__version__); print('channels',channels.__version__)"
django 4.0.4
whoosh (2, 7, 4)
django_q (1, 3, 9)
drf 3.13.1
tqdm 4.64.0
redis 3.5.3
channels 3.0.4
```

These match the pins in `requirements.txt`: `whoosh==2.7.4` (L111), `django==4.0.4` (L38), `django-q==1.3.9` (L37), `djangorestframework==3.13.1` (L39), `tqdm==4.64.0` (L97), `redis==3.5.3` (L84).

**Running services (broker + worker + web/API).** Redis is the Django‑Q broker; the worker is `python manage.py qcluster` (its stdout is captured to `/tmp/qcluster.log`); the web/API layer is `gunicorn -c /app/gunicorn.conf.py paperless.asgi:application` on `0.0.0.0:8000`. The complete process listing below shows Redis (PID 454), the qcluster master (PID 6255, parent PID 0), its sentinel (PID 6269, child of 6255) and its recycling worker pool (the PIDs whose parent is 6269 — Django‑Q sets `recycle: 1` in `Q_CLUSTER`, `src/paperless/settings.py:L452`, so worker processes are recycled and their PIDs churn), plus the gunicorn master (PID 11467, parent PID 0) and its two workers (PIDs 11477/11478, children of 11467):

```
$ redis-cli ping
PONG
$ ps -eo pid,ppid,cmd | grep -E "manage.py qcluster|gunicorn|redis-server" | grep -v grep
    454       1 redis-server *:6379
   6255       0 python manage.py qcluster
   6269    6255 python manage.py qcluster
   6281    6269 python manage.py qcluster
   6282    6269 python manage.py qcluster
   6347    6269 python manage.py qcluster
   6348    6269 python manage.py qcluster
   7006    6269 python manage.py qcluster
  11467       0 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  11477   11467 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  11478   11467 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
  11578    6269 python manage.py qcluster
  11810    6269 python manage.py qcluster
  11870    6269 python manage.py qcluster
  12082    6269 python manage.py qcluster
  12130    6269 python manage.py qcluster
  12148    6269 python manage.py qcluster
  12149    6269 python manage.py qcluster
  12231    6269 python manage.py qcluster
```

**Process baseline (these services were started by the canonical setup, before this investigation began).** Their start times all precede the experiments below (which ran from ~05:23 UTC onward), which is the evidence that the investigation started no service of its own (see the cleanup pass in §7):

```
$ date -u +"%Y-%m-%dT%H:%M:%SZ"
2026-07-08T05:23:57Z
$ ps -o pid,lstart,cmd -p 454,6255,11467
    PID                  STARTED CMD
    454 Wed Jul  8 04:04:05 2026 redis-server *:6379
   6255 Wed Jul  8 04:13:11 2026 python manage.py qcluster
  11467 Wed Jul  8 04:21:30 2026 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
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
SEEDED_IDS [5, 6, 7]
TITLES [(5, 'seedqonealpha'), (6, 'seedqtwobravo'), (7, 'seedqthreecharlie')]
```

- **Document count used throughout: 3** (ids **5, 6, 7**; the SQLite primary‑key sequence continues past ids used and freed by earlier setup activity). Doc **5** (`seedqonealpha`) is the Q1 edit target; doc **6** (`seedqtwobravo`) is the Q2 raw‑SQL target; doc **7** is filler. Search terms are distinctive lowercase single tokens so the Whoosh standard analyzer indexes them cleanly and searches are unambiguous.

**API authentication.** The REST API requires authentication (`REST_FRAMEWORK` Basic/Session/Token auth, `src/paperless/settings.py:L116-L120`). The commands below use HTTP Basic auth as `admin:paperless` — a local, throwaway superuser created by the canonical setup for this ephemeral, network‑isolated container (per the environment notes; paperless needs no external credentials). It is shown verbatim only so the commands are exactly reproducible; it is not a real or production secret.

**Baseline index population and the read path (proves the index is populated and search works):**

```
$ python manage.py document_index reindex

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 754.55it/s]

$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqonealpha" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":5,"correspondent":null,"document_type":null,"title":"seedqonealpha","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:24:18.958991Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqonealpha.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

(The `document_index reindex` progress bar is emitted by tqdm on **stderr**; the blocks in this document show the command's combined stdout+stderr as it appears in the terminal.)

The `"__search_hit__"` marker on the result confirms the response came through the Whoosh search read path (`UnifiedSearchViewSet.list()` → `open_index_searcher()`), not a plain database listing.

**Baseline background‑task state.** Django‑Q records each *completed* task in the `Task` table (`django_q.models.Task`), and that history accumulates for the lifetime of the long‑running worker. To make the "was a job enqueued for this operation?" observations self‑contained and start from a known zero, the completed‑task history was cleared once here (this is Django‑Q observability data — `Task` rows in the git‑ignored SQLite DB — not any indexing code path). The authoritative signal used throughout is then the **function‑filtered** count `Task.objects.filter(func="documents.tasks.bulk_update_documents").count()`, which is immune to the unrelated periodic mail task (`paperless_mail.tasks.process_mail_accounts`) that otherwise adds noise to the `Task` table and the `qcluster` log:

```
$ echo "from django_q.models import Task; print('DELETED', Task.objects.all().delete()); print('TASK_TOTAL_NOW', Task.objects.count())" | python manage.py shell
DELETED (13, {'django_q.Task': 13})
TASK_TOTAL_NOW 0
$ echo "from django_q.models import Task; print('TASK_TOTAL', Task.objects.count()); print('BULK_UPDATE_TASKS', Task.objects.filter(func='documents.tasks.bulk_update_documents').count())" | python manage.py shell
TASK_TOTAL 0
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

### 3.1 Evidence — Run 1 (title → `quniqxaa`), fully command‑grounded

Each edit cycle is driven by a small wrapper script (a temporary observation artifact under `/tmp`, removed in §7) that makes the worker/queue evidence command‑grounded: it (a) records the `qcluster.log` line count and the function‑filtered `bulk_update_documents` task count *before* the edit, (b) issues the real `PATCH` and the immediate search — each printing its **full response body** plus curl's `time_total`, and (c) records the line/task counts *after* and prints the exact new `qcluster.log` lines via `sed -n "$((before+1)),${after}p"`, so "no worker activity" is proven by an actually‑empty command output rather than asserted. The wrapper is shown once, then invoked as `<doc_id> <new_title>` per run:

```
$ cat /tmp/qna_q1_cycle.sh
#!/bin/bash
# Usage: qna_q1_cycle.sh <doc_id> <new_title>
# Brackets a single-document API title edit with worker-log line counts and
# a function-filtered Django-Q task count so we can prove whether a job was picked up.
LOG=/tmp/qcluster.log
ID="$1"; TITLE="$2"
FUNC=documents.tasks.bulk_update_documents
cd /app/src

before=$(wc -l < "$LOG")
bulk_before=$(echo "from django_q.models import Task; print(Task.objects.filter(func='$FUNC').count())" | python manage.py shell 2>/dev/null)
echo "BEFORE: qcluster.log lines=$before ; bulk_update_documents tasks=$bulk_before"

echo "--- PATCH /api/documents/$ID/ {\"title\":\"$TITLE\"} ---"
curl -sS -u admin:paperless -X PATCH "http://localhost:8000/api/documents/$ID/" \
     -H "Content-Type: application/json" -d "{\"title\":\"$TITLE\"}" \
     -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"

echo "--- SEARCH ?query=$TITLE (issued immediately after the PATCH) ---"
curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=$TITLE" \
     -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"

after=$(wc -l < "$LOG")
bulk_after=$(echo "from django_q.models import Task; print(Task.objects.filter(func='$FUNC').count())" | python manage.py shell 2>/dev/null)
echo "AFTER:  qcluster.log lines=$after ; bulk_update_documents tasks=$bulk_after"
echo "--- NEW qcluster.log lines, range [$((before+1))..$after] via: sed -n \"$((before+1)),${after}p\" $LOG  (empty output = no worker activity) ---"
sed -n "$((before+1)),${after}p" "$LOG"
echo "--- (end of new qcluster.log lines) ---"
```

```
$ bash /tmp/qna_q1_cycle.sh 5 quniqxaa
BEFORE: qcluster.log lines=113 ; bulk_update_documents tasks=0
--- PATCH /api/documents/5/ {"title":"quniqxaa"} ---
{"id":5,"correspondent":null,"document_type":null,"title":"quniqxaa","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:10.809611Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxaa.pdf","archived_file_name":null}
HTTP 200  time_total=0.148940s
--- SEARCH ?query=quniqxaa (issued immediately after the PATCH) ---
{"count":1,"next":null,"previous":null,"results":[{"id":5,"correspondent":null,"document_type":null,"title":"quniqxaa","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:10.809611Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxaa.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200  time_total=0.109304s
AFTER:  qcluster.log lines=113 ; bulk_update_documents tasks=0
--- NEW qcluster.log lines, range [114..113] via: sed -n "114,113p" /tmp/qcluster.log  (empty output = no worker activity) ---
--- (end of new qcluster.log lines) ---
```

**Reading of the evidence:** The `PATCH` returned **HTTP 200** with the serialized document showing `"title":"quniqxaa"`, in **0.149 s**. The *immediate* search returned **`"count":1`** with the edited document (id 5, new title, `__search_hit__`), in **0.109 s** — the write was visible on the very next request. Across the window the worker log stayed at **113 lines** and the `bulk_update_documents` count stayed at **0**; the new‑lines command `sed -n "114,113p" /tmp/qcluster.log` produced **no output at all** (its range start 114 is past the end 113), which is the command‑grounded proof that the worker logged nothing and picked up no job for the single‑document edit.

### 3.2 Evidence — latency stability across runs 2 and 3 (same wrapper, full output)

```
$ bash /tmp/qna_q1_cycle.sh 5 quniqxbb
BEFORE: qcluster.log lines=113 ; bulk_update_documents tasks=0
--- PATCH /api/documents/5/ {"title":"quniqxbb"} ---
{"id":5,"correspondent":null,"document_type":null,"title":"quniqxbb","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:12.886944Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxbb.pdf","archived_file_name":null}
HTTP 200  time_total=0.148883s
--- SEARCH ?query=quniqxbb (issued immediately after the PATCH) ---
{"count":1,"next":null,"previous":null,"results":[{"id":5,"correspondent":null,"document_type":null,"title":"quniqxbb","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:12.886944Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxbb.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200  time_total=0.110046s
AFTER:  qcluster.log lines=113 ; bulk_update_documents tasks=0
--- NEW qcluster.log lines, range [114..113] via: sed -n "114,113p" /tmp/qcluster.log  (empty output = no worker activity) ---
--- (end of new qcluster.log lines) ---

$ bash /tmp/qna_q1_cycle.sh 5 quniqxcc
BEFORE: qcluster.log lines=113 ; bulk_update_documents tasks=0
--- PATCH /api/documents/5/ {"title":"quniqxcc"} ---
{"id":5,"correspondent":null,"document_type":null,"title":"quniqxcc","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:15.013933Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxcc.pdf","archived_file_name":null}
HTTP 200  time_total=0.159764s
--- SEARCH ?query=quniqxcc (issued immediately after the PATCH) ---
{"count":1,"next":null,"previous":null,"results":[{"id":5,"correspondent":null,"document_type":null,"title":"quniqxcc","content":"alpha bravo charlie content one","tags":[],"created":"2026-07-08T05:24:18.957325Z","modified":"2026-07-08T05:27:15.013933Z","added":"2026-07-08T05:24:18.957329Z","archive_serial_number":null,"original_file_name":"2026-07-08 quniqxcc.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200  time_total=0.109949s
AFTER:  qcluster.log lines=113 ; bulk_update_documents tasks=0
--- NEW qcluster.log lines, range [114..113] via: sed -n "114,113p" /tmp/qcluster.log  (empty output = no worker activity) ---
--- (end of new qcluster.log lines) ---
```

**Stability (3 documents, 3 runs).** PATCH latency: **0.149 s / 0.149 s / 0.160 s**; immediate‑search latency: **0.109 s / 0.110 s / 0.110 s**. Every run: search **count = 1**, worker log **113 → 113**, `bulk_update_documents` **0 → 0**, and the `sed -n "114,113p"` new‑lines command emits **nothing**. The "immediately visible, no job" behavior is stable across runs; latency is consistently well under a second.

### 3.3 Async contrast — a bulk edit DOES enqueue a Django‑Q job

A bulk edit routes through `src/documents/bulk_edit.py`, whose operations dispatch a `documents.tasks.bulk_update_documents` async task via a call to `async_task` (at `:L18, :L31, :L47, :L63, :L87`). `modify_tags` sets `affected_docs` to *all* requested ids unconditionally, so it reliably enqueues a job (complete function, verbatim):

```
# src/documents/bulk_edit.py:L68-L89  (modify_tags → async_task)
def modify_tags(doc_ids, add_tags, remove_tags):
    qs = Document.objects.filter(id__in=doc_ids)
    affected_docs = [doc.id for doc in qs]

    DocumentTagRelationship = Document.tags.through

    DocumentTagRelationship.objects.filter(
        document_id__in=affected_docs,
        tag_id__in=remove_tags,
    ).delete()

    DocumentTagRelationship.objects.bulk_create(
        [
            DocumentTagRelationship(document_id=doc, tag_id=tag)
            for (doc, tag) in itertools.product(affected_docs, add_tags)
        ],
        ignore_conflicts=True,
    )

    async_task("documents.tasks.bulk_update_documents", document_ids=affected_docs)

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

The bulk edit is exercised with the same bracketing wrapper (temporary `/tmp` artifact, removed in §7), which additionally prints the last `bulk_update_documents` task row:

```
$ cat /tmp/qna_q1_bulk.sh
#!/bin/bash
# Async contrast: a bulk edit dispatches a Django-Q job (unlike the single-document edit).
LOG=/tmp/qcluster.log
FUNC=documents.tasks.bulk_update_documents
cd /app/src
before=$(wc -l < "$LOG")
bulk_before=$(echo "from django_q.models import Task; print(Task.objects.filter(func='$FUNC').count())" | python manage.py shell 2>/dev/null)
echo "BEFORE: qcluster.log lines=$before ; bulk_update_documents tasks=$bulk_before"
echo "--- POST /api/documents/bulk_edit/ method=modify_tags add_tags=[2] on doc 5 ---"
curl -sS -u admin:paperless -X POST "http://localhost:8000/api/documents/bulk_edit/" \
     -H "Content-Type: application/json" \
     -d '{"documents":[5],"method":"modify_tags","parameters":{"add_tags":[2],"remove_tags":[]}}' \
     -w "\nHTTP %{http_code}  time_total=%{time_total}s\n"
echo "--- sleep 5 (allow the Django-Q worker to process the queued job) ---"
sleep 5
after=$(wc -l < "$LOG")
bulk_after=$(echo "from django_q.models import Task; print(Task.objects.filter(func='$FUNC').count())" | python manage.py shell 2>/dev/null)
echo "AFTER:  qcluster.log lines=$after ; bulk_update_documents tasks=$bulk_after"
echo "--- last bulk_update_documents Task (func, name, success, started) ---"
echo "from django_q.models import Task; t=Task.objects.filter(func='documents.tasks.bulk_update_documents').order_by('-started').first(); print((t.func, t.name, t.success, str(t.started)))" | python manage.py shell 2>/dev/null
echo "--- NEW qcluster.log lines, range [$((before+1))..$after] via: sed -n \"$((before+1)),${after}p\" $LOG ---"
sed -n "$((before+1)),${after}p" "$LOG"
echo "--- (end of new qcluster.log lines) ---"

$ bash /tmp/qna_q1_bulk.sh
BEFORE: qcluster.log lines=113 ; bulk_update_documents tasks=0
--- POST /api/documents/bulk_edit/ method=modify_tags add_tags=[2] on doc 5 ---
{"result":"OK"}
HTTP 200  time_total=0.117422s
--- sleep 5 (allow the Django-Q worker to process the queued job) ---
AFTER:  qcluster.log lines=125 ; bulk_update_documents tasks=1
--- last bulk_update_documents Task (func, name, success, started) ---
('documents.tasks.bulk_update_documents', 'oklahoma-yellow-nitrogen-aspen', True, '2026-07-08 05:27:56.122318+00:00')
--- NEW qcluster.log lines, range [114..125] via: sed -n "114,125p" /tmp/qcluster.log ---
05:27:55 [Q] INFO Enqueued 1
05:27:55 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
05:27:55 [Q] INFO Process-1:16 processing [failed-carbon-artist-west]
05:27:55 [Q] INFO Process-1:16 stopped doing work
05:27:55 [Q] INFO Processed [failed-carbon-artist-west]
05:27:56 [Q] INFO recycled worker Process-1:16
05:27:56 [Q] INFO Process-1:27 ready for work at 12458
05:27:56 [Q] INFO Process-1:17 processing [oklahoma-yellow-nitrogen-aspen]
05:27:56 [Q] INFO Process-1:17 stopped doing work
05:27:56 [Q] INFO Processed [oklahoma-yellow-nitrogen-aspen]
05:27:56 [Q] INFO recycled worker Process-1:17
05:27:56 [Q] INFO Process-1:28 ready for work at 12462
```

**Reading of the evidence:** The bulk edit returned `{"result":"OK"}` and, unlike the single edit, the worker **picked up a job**: the `bulk_update_documents` count went **0 → 1**, the last such `Task` is `('documents.tasks.bulk_update_documents', 'oklahoma-yellow-nitrogen-aspen', True, '2026-07-08 05:27:56.122318+00:00')` (success), and the worker log grew **113 → 125**. The `sed -n "114,125p"` extraction shows the real new lines: the bulk edit's `Enqueued 1` followed by the worker `processing [oklahoma-yellow-nitrogen-aspen]` and `Processed [oklahoma-yellow-nitrogen-aspen]` — the direct contrast to the single‑document edit, which produced no lines at all. (The interleaved `[Check all e-mail accounts]` / `[failed-carbon-artist-west]` lines are the **unrelated** periodic mail schedule `paperless_mail.tasks.process_mail_accounts` that happened to fire in the same window; the function‑filtered `bulk_update_documents` count `0 → 1` isolates our signal from that noise, which is exactly why that filtered count — not the raw log — is the authoritative "was a job enqueued?" metric.)

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

Confirmed at runtime by enumerating the live signal receivers — a genuine observation of the dispatcher, not merely a reading of the source. `add_to_index` is **not** among the `post_save` receivers for `Document`, and appears **only** among the `document_consumption_finished` receivers, so a metadata edit (which fires `post_save`) triggers no indexing signal at all:

```
$ python manage.py shell <<'PY'
from django.db.models.signals import post_save
from documents.models import Document
from documents.signals import document_consumption_finished
ps = [f.__qualname__ for f in post_save._live_receivers(Document)]
dc = [f.__qualname__ for f in document_consumption_finished._live_receivers(Document)]
print('post_save[sender=Document]         :', ps)
print('document_consumption_finished      :', dc)
print('add_to_index fires on post_save?   :', 'add_to_index' in ps)
print('add_to_index fires on consumption? :', 'add_to_index' in dc)
PY
post_save[sender=Document]         : ['update_filename_and_move_files']
document_consumption_finished      : ['add_inbox_tags', 'set_correspondent', 'set_document_type', 'set_tags', 'set_log_entry', 'add_to_index']
add_to_index fires on post_save?   : False
add_to_index fires on consumption? : True
```

The runtime enumeration above, together with the wiring quoted from `apps.py:L27`, **observes** that no indexing signal is wired to fire on a metadata edit; the code at `src/documents/views.py:L216` (quoted in the DIRECT ANSWER above) is what performs the in‑request write. The narrower attribution that the **view handler specifically** — rather than any signal — is what indexes the edit is labeled **inferred** (from the code structure + this runtime receiver enumeration + the observed synchronous, no‑job behavior in §3.1 and §3.2), rather than proven by instrumenting the signal path itself: the observed timing alone is consistent with either a view‑handler write or a hypothetical *synchronous* `post_save` receiver (Django `post_save` fires inside the request, not as a Django‑Q job), so distinguishing the two rests on the code and the receiver wiring shown here. This mirrors the labeling of the parallel read‑derived claim in §4.4.

This is why the raw‑SQL path in Q2 (which fires no signal at all) leaves the index untouched.

---

## 4. Q2 — raw SQL bypass: stale index and forced reconciliation

**DIRECT ANSWER.** Updating a document's title *directly in the database via raw SQL* (not through the API/ORM) leaves the Whoosh index **stale**: a search for the **new** title returns **nothing (MISS)**, while a search for the **old** title still **HITS** the document. The system does **not** auto‑reconcile. To force reconciliation you run **`python manage.py document_index reindex`**, which executes **in the command's own process (NO Django‑Q job)** and prints a **tqdm** progress bar as its visible activity. After reindex the new title HITS and the old title MISSES.

**Why it goes stale.** A raw SQL `UPDATE` bypasses Django's ORM entirely, so it fires no `post_save` signal, and `Document` has **no `save()` override** that would touch the index. The class is defined at `src/documents/models.py:L88` and its `title` field at `:L106`; a search for any `save()` override on the model returns nothing (grep exit status `1` = no match), so `index.update_document()` is never reached and the on‑disk index keeps the old tokens:

```
$ grep -nE "def save" src/documents/models.py
$ echo "grep_exit=$?"
grep_exit=1
$ sed -n "88p;106p" src/documents/models.py
class Document(models.Model):
    title = models.CharField(_("title"), max_length=128, blank=True, db_index=True)
```

### 4.1 Evidence — before: index and DB agree

Doc 6 was seeded with title `seedqtwobravo` (§2), and at the start of this experiment the index and DB agree on it — a search for `seedqtwobravo` returns the document:

```
$ echo "from documents.models import Document; d=Document.objects.get(id=6); print('DOC6_DB_TITLE', repr(d.title))" | python manage.py shell
DOC6_DB_TITLE 'seedqtwobravo'
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":6,"correspondent":null,"document_type":null,"title":"seedqtwobravo","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T05:24:18.987685Z","modified":"2026-07-08T05:24:18.987936Z","added":"2026-07-08T05:24:18.987689Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqtwobravo.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

### 4.2 Evidence — the raw SQL UPDATE (bypassing the ORM), then the stale index

The update was issued through the raw DB cursor (`django.db.connection.cursor()`), which executes SQL directly with no ORM `save()` and no signal. Script `/tmp/qna_rawsql.py` (a temporary observation artifact, removed in §7), shown and executed:

```
$ cat /tmp/qna_rawsql.py
from django.db import connection
c = connection.cursor()
c.execute("UPDATE documents_document SET title=%s WHERE id=%s", ["rawsqlnewzz", 6])
print("ROWCOUNT", c.rowcount)
c.execute("SELECT id, title FROM documents_document WHERE id=%s", [6])
print("RAW_SELECT", c.fetchone())
$ python manage.py shell < /tmp/qna_rawsql.py
ROWCOUNT 1
RAW_SELECT (6, 'rawsqlnewzz')
```

Now the index is stale — the **new** title MISSES, the **old** title still HITS:

```
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":6,"correspondent":null,"document_type":null,"title":"rawsqlnewzz","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T05:24:18.987685Z","modified":"2026-07-08T05:24:18.987936Z","added":"2026-07-08T05:24:18.987689Z","archive_serial_number":null,"original_file_name":"2026-07-08 rawsqlnewzz.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
```

**Reading of the evidence (the divergence in one screen).** The raw SQL changed the database (`ROWCOUNT 1`, `RAW_SELECT (6, 'rawsqlnewzz')`). Yet a search for the **new** title `rawsqlnewzz` returns **`"count":0`** — the index has no such token. A search for the **old** title `seedqtwobravo` still returns **`"count":1`** — the index still maps that stale token to doc 6. Note the smoking gun: that stale hit's serialized `"title"` is **`"rawsqlnewzz"`** (the current DB value), because the search *matches* on the stale Whoosh index but the API *serializes* the row from the database. You can only *find* the document by its obsolete indexed title, while its actual title is the new one — a textbook index/DB divergence.

### 4.3 Evidence — forced reconciliation via `document_index reindex` (in‑process, no job)

The reconciliation entry point is a synchronous management command wrapped in a DB transaction; it calls `index_reindex()` directly and dispatches no `async_task`:

```
# src/documents/management/commands/document_index.py:L11-L25  (add_arguments + handle, verbatim)
    def add_arguments(self, parser):
        parser.add_argument("command", choices=["reindex", "optimize"])
        parser.add_argument(
            "--no-progress-bar",
            default=False,
            action="store_true",
            help="If set, the progress bar will not be shown",
        )

    def handle(self, *args, **options):
        with transaction.atomic():
            if options["command"] == "reindex":
                index_reindex(progress_bar_disable=options["no_progress_bar"])
            elif options["command"] == "optimize":
                index_optimize()

# src/documents/tasks.py:L38-L45  (index_reindex — rebuilds from Document.objects.all(), tqdm bar)
def index_reindex(progress_bar_disable=False):
    documents = Document.objects.all()

    ix = index.open_index(recreate=True)

    with AsyncWriter(ix) as writer:
        for document in tqdm.tqdm(documents, disable=progress_bar_disable):
            index.update_document(writer, document)
```

The reindex is run through the same bracketing wrapper (temporary `/tmp` artifact, removed in §7), which records the worker‑log line count and `Task` total before and after and prints the exact new log lines via `sed`, so "no job was picked up" is proven by an actually‑empty command output:

```
$ cat /tmp/qna_q2_reindex.sh
#!/bin/bash
# Reconciliation: document_index reindex runs in-process (no Django-Q job).
LOG=/tmp/qcluster.log
cd /app/src
before=$(wc -l < "$LOG")
total_before=$(echo "from django_q.models import Task; print(Task.objects.count())" | python manage.py shell 2>/dev/null)
echo "BEFORE: qcluster.log lines=$before ; Task total=$total_before"
echo "--- python manage.py document_index reindex   (stdout+stderr combined; tqdm on stderr) ---"
python manage.py document_index reindex 2>&1
after=$(wc -l < "$LOG")
total_after=$(echo "from django_q.models import Task; print(Task.objects.count())" | python manage.py shell 2>/dev/null)
echo "AFTER:  qcluster.log lines=$after ; Task total=$total_after"
echo "--- NEW qcluster.log lines, range [$((before+1))..$after] via: sed -n \"$((before+1)),${after}p\" $LOG  (empty = reindex ran in-process, NOT via Django-Q) ---"
sed -n "$((before+1)),${after}p" "$LOG"
echo "--- (end of new qcluster.log lines) ---"

$ bash /tmp/qna_q2_reindex.sh
BEFORE: qcluster.log lines=132 ; Task total=3
--- python manage.py document_index reindex   (stdout+stderr combined; tqdm on stderr) ---

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 727.04it/s]
AFTER:  qcluster.log lines=132 ; Task total=3
--- NEW qcluster.log lines, range [133..132] via: sed -n "133,132p" /tmp/qcluster.log  (empty = reindex ran in-process, NOT via Django-Q) ---
--- (end of new qcluster.log lines) ---
```

After the reindex the **new** title HITS and the **old** title MISSES:

```
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":6,"correspondent":null,"document_type":null,"title":"rawsqlnewzz","content":"delta echo foxtrot content two","tags":[],"created":"2026-07-08T05:24:18.987685Z","modified":"2026-07-08T05:24:18.987936Z","added":"2026-07-08T05:24:18.987689Z","archive_serial_number":null,"original_file_name":"2026-07-08 rawsqlnewzz.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqtwobravo" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200
```

**Reading of the evidence:** `document_index reindex` printed a **tqdm** bar — the intermediate `0%|          | 0/3 [00:00<?, ?it/s]` line followed by the final `100%|██████████| 3/3 [00:00<00:00, 727.04it/s]` — as its only visible activity. Afterward the **new** title `rawsqlnewzz` HITS (`"count":1`, id 6) and the **old** title `seedqtwobravo` MISSES (`"count":0`) — the index is reconciled with the database. The wrapper's own `wc -l` bracket shows the worker log **unchanged at 132 lines** and the `Task` total **unchanged at 3**, and the command‑grounded new‑lines extraction `sed -n "133,132p" /tmp/qcluster.log` produced **no output at all** (its range start 133 is past the end 132) — the proof that the command ran **in the invoking process, not via Django‑Q**. (The absolute log count is higher than Q1's 125 because the periodic mail schedule keeps appending lines between experiments; each experiment's own before/after window — not the absolute number — is what proves its claim.) The `Task` table's function breakdown confirms there is no reindex entry:

```
$ echo "from django_q.models import Task; from collections import Counter; print(dict(Counter(t.func for t in Task.objects.all())))" | python manage.py shell
{'paperless_mail.tasks.process_mail_accounts': 2, 'documents.tasks.bulk_update_documents': 1}
```

The single `documents.tasks.bulk_update_documents` row is the one from the Q1 async contrast; the two `paperless_mail.tasks.process_mail_accounts` rows are the unrelated periodic mail schedule; there is **no** `reindex`/`index_reindex` function anywhere in the queue — reindex is never a queued task. (`Task total=3` above equals `2 + 1`, matching this breakdown.)

### 4.4 The sanity check does not reconcile the index (**inferred**)

`src/documents/sanity_checker.py` contains **no** `index`/`whoosh`/`reindex` references. A case‑insensitive grep for those tokens across the file returns nothing (grep exit status `1` = no match), while the file itself exists and is non‑empty (4861 bytes) — so the empty result is a real "no matches," not a missing file:

```
$ grep -niE "index|whoosh|reindex" src/documents/sanity_checker.py
$ echo "grep_exit=$?"
grep_exit=1
$ ls -l src/documents/sanity_checker.py | awk '{print $5, $9}'
4861 src/documents/sanity_checker.py
```

Combined with the observation above that a manual `document_index reindex` was required to fix the stale state, this indicates the weekly `sanity_check` does not repair a stale/missing index. This is labeled **inferred** (from reading the source + the observed need for a manual reindex) rather than directly observed by running the sanity check.

---

## 5. Q3 — index corruption/deletion and rebuild: self‑heal vs. manual intervention

**DIRECT ANSWER.** If the index is deleted or corrupted while documents still exist in the database, the index **structure self‑heals but the data does not**. Opening the index recreates an **empty** structure automatically (via `create_in`), so searches return **nothing** until you rebuild. On a *corrupt* (as opposed to merely missing) index, `open_index()` logs exactly **`Error while opening the index, recreating.`** and then recreates the empty structure. Repopulation is **always manual** — `python manage.py document_index reindex` — because there is **no scheduled reindex** anywhere in the system. For the 3‑document set, the rebuild wall‑clock was **stable at ≈1.35 s (1.345–1.384 s) across three runs** (the pure indexing throughput is ≈700–730 docs/s; the wall‑clock is dominated by process/Django startup), and the **tqdm** bar is the visible processing activity.

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
drwxr-sr-x 2 root root  4096 Jul  8 05:45 .
drwxr-sr-x 4 root root  4096 Jul  8 05:47 ..
-rwxr-xr-x 1 root root     0 Jul  8 05:30 MAIN_WRITELOCK
-rw-r--r-- 1 root root 14839 Jul  8 05:45 MAIN_paqpg6kvnyejmjnk.seg
-rw-r--r-- 1 root root  4396 Jul  8 05:45 _MAIN_1.toc
$ rm -rf /app/data/index/*
$ ls -la /app/data/index                            # immediately after delete: empty
total 8
drwxr-sr-x 2 root root 4096 Jul  8 05:51 .
drwxr-sr-x 4 root root 4096 Jul  8 05:47 ..
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=rawsqlnewzz" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqthreecharlie" -w "\nHTTP %{http_code}\n"
{"count":0,"next":null,"previous":null,"results":[]}
HTTP 200
$ ls -la /app/data/index                            # after a search: empty structure recreated
total 12
drwxr-sr-x 2 root root 4096 Jul  8 05:51 .
drwxr-sr-x 4 root root 4096 Jul  8 05:47 ..
-rw-r--r-- 1 root root 4064 Jul  8 05:51 _MAIN_0.toc
```

**Reading of the evidence:** After deleting the index contents, the directory holds only `.` and `..` (`total 8`); searching returns **`"count":0`** for every query even though the three documents still exist in the database. The first `open_index()` call (via the search read path) recreated an **empty** structure — note the new `_MAIN_0.toc` (4064 bytes) but **no `.seg` data file** (the 14839‑byte `MAIN_paqpg6kvnyejmjnk.seg` is gone). This is the structure‑only self‑heal (`create_in`, `src/documents/index.py:L61`); the pure‑deletion case takes the non‑exception branch (`exists_in()` is False), so it recreates *silently* with no error log.

### 5.2 Evidence — corruption case: the exact `Error while opening the index, recreating.` log line

The index was repopulated, then its table‑of‑contents file was overwritten with garbage, then `documents.index.open_index()` was called so its internal `try/except` fires. A handler was attached to the `paperless.index` logger (`src/documents/index.py:L28` → `logger = logging.getLogger("paperless.index")`) to capture the line:

```
$ python manage.py document_index reindex          # repopulate first

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 685.23it/s]
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
[2026-07-08 05:51:46,010] [ERROR] [paperless.index] Error while opening the index, recreating.
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

**Reading of the evidence:** With a corrupt table‑of‑contents, `exists_in(settings.INDEX_DIR)` — called *inside* `open_index()`'s `try` at `src/documents/index.py:L54` — raised `whoosh.index.IndexError: Index was created on different architecture: saved int = 71, this computer = 4`. `open_index()`'s `except` branch (`:L56-L57`) caught it and logged the exact line **`Error while opening the index, recreating.`** (shown twice: once via the handler added for capture, prefixed `CAPTURED[paperless.index ERROR]`, and once via the application's own configured handler, prefixed `[2026-07-08 05:51:46,010] [ERROR] [paperless.index]`). It then recreated an empty structure — `OPEN_RESULT_doc_count: 0` confirms the rebuilt index holds **no data**. So corruption self‑heals the *structure* (and logs a clear error), but the *data* is gone until a manual reindex.

### 5.3 Evidence — rebuild wall‑clock timing, stable across three runs (3 documents)

```
$ echo "from documents.models import Document; print(Document.objects.count())" | python manage.py shell    # document count
3
$ rm -rf /app/data/index/* && { time python manage.py document_index reindex ; }   # RUN 1

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 701.47it/s]

real	0m1.384s
user	0m1.229s
sys	0m0.156s

$ rm -rf /app/data/index/* && { time python manage.py document_index reindex ; }   # RUN 2

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 730.46it/s]

real	0m1.345s
user	0m1.190s
sys	0m0.158s

$ rm -rf /app/data/index/* && { time python manage.py document_index reindex ; }   # RUN 3

  0%|          | 0/3 [00:00<?, ?it/s]
100%|██████████| 3/3 [00:00<00:00, 713.28it/s]

real	0m1.375s
user	0m1.222s
sys	0m0.156s
```

**Reading of the evidence (timing).** Document count = **3**. Rebuild wall‑clock (`real`): **1.384 s / 1.345 s / 1.375 s** — **stable** (spread ≈ 3%). The tqdm bar (the visible processing activity) reports ≈**700–730 docs/s** (701.47 / 730.46 / 713.28 it/s), i.e. the actual indexing of 3 documents takes on the order of **4 ms**; the ≈1.35 s command wall‑clock is dominated by Python interpreter + Django app startup, not by the indexing work. For this small set the reconstruction is effectively near‑instant, and the value is reproducible across runs.

### 5.4 Evidence — documents searchable again after rebuild

```
$ curl -sS -u admin:paperless "http://localhost:8000/api/documents/?query=seedqthreecharlie" -w "\nHTTP %{http_code}\n"
{"count":1,"next":null,"previous":null,"results":[{"id":7,"correspondent":null,"document_type":null,"title":"seedqthreecharlie","content":"golf hotel india content three","tags":[],"created":"2026-07-08T05:24:19.012229Z","modified":"2026-07-08T05:24:19.012455Z","added":"2026-07-08T05:24:19.012232Z","archive_serial_number":null,"original_file_name":"2026-07-08 seedqthreecharlie.pdf","archived_file_name":null,"__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
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
| Q1‑a | Does an API title edit appear immediately or need background processing? | **Immediately, synchronously**; searchable in the same request (search count=1) with search latency ≤ 0.11 s | `views.py:L212-L217`, `index.py:L64-L74`, `:L118-L120` | §3.1 |
| Q1‑b | Is any Django‑Q job picked up for the single edit? | **No** — worker log 113→113, `bulk_update_documents` 0→0, no new log lines | `views.py:L212-L217` (no `async_task`) | §3.1–§3.2 |
| Q1‑c | Watch the worker: does a single edit produce a task? | **No task** for the edit (3 runs; the `sed` new‑lines window is empty) | `views.py:L212-L217` (no `async_task`; cf. `bulk_edit.py:L87`) | §3.1–§3.2 |
| Q1‑d | Latency, ≥2 runs, stable? | PATCH 0.149/0.149/0.160 s; search 0.109/0.110/0.110 s — **stable, sub‑second** (3 docs, 3 runs) | runtime measurement; path `views.py:L212-L217` + `:L413-L425` | §3.1–§3.2 |
| Q1‑e | Asynchronous contrast | **Bulk edit DOES enqueue a job** — `bulk_update_documents` 0→1, task `oklahoma-yellow-nitrogen-aspen` success, worker log 113→125 | `bulk_edit.py:L87`, `tasks.py:L270-L280` | §3.3 |
| Q1‑f | Are signals the mechanism? | **No** — runtime receiver enumeration shows `post_save[Document]` = `[update_filename_and_move_files]` (moves files only) and `add_to_index` fires only on `document_consumption_finished`; the narrower attribution to the view handler specifically is **inferred** | `handlers.py:L310-L312`, `:L428-L431`, `apps.py:L27` | §3.4 |
| Q2‑a | Does raw SQL update make the new title searchable? | **No — stale index; new title MISS (count:0)** | `models.py:L88`,`:L106` (no `save()` override) | §4.2 |
| Q2‑b | Old title after raw SQL? | **Still HITS (count:1)**; hit serializes the new DB title → index/DB divergence | `index.py:L240-L254` (matching), serializer reads DB | §4.2 |
| Q2‑c | Is there a way to force reconciliation? | **Yes — `python manage.py document_index reindex`** | `document_index.py:L20-L25`, `tasks.py:L38-L45` | §4.3 |
| Q2‑d | Effect on worker logs / task queue during reconciliation? | **None** — runs in‑process; log 132→132, Task 3→3, no reindex task in queue; **tqdm** is the visible activity | `document_index.py:L21` (`transaction.atomic`), no `async_task` | §4.3 |
| Q2‑e | Does the sanity check reconcile the index? | **No** (**inferred**: `sanity_checker.py` has no index refs; manual reindex was required) | `sanity_checker.py` (no index refs) | §4.4 |
| Q3‑a | Behavior when index is deleted? | Empty structure silently recreated on open; **searches return nothing** | `index.py:L52-L61`, `:L61` (`create_in`) | §5.1 |
| Q3‑b | Behavior when index is corrupted? | Logs exactly **`Error while opening the index, recreating.`**, recreates empty (doc_count 0) | `index.py:L56-L57`, `:L61` | §5.2 |
| Q3‑c | Recovery mechanism | **Manual `document_index reindex`** repopulates from `Document.objects.all()` | `tasks.py:L38-L45` | §5.3–§5.4 |
| Q3‑d | Rebuild duration for a small set, ≥2 runs, stable? | **1.384/1.345/1.375 s** wall‑clock for **3 docs** — stable (spread ≈3%); indexing ≈700–730 docs/s | runtime measurement; path `tasks.py:L38-L45` (`:L44` loop) | §5.3 |
| Q3‑e | Visible processing activity during rebuild | The **tqdm** progress bar — e.g. `100%|██████████| 3/3 [00:00<00:00, 701.47it/s]` (rate 701.47/730.46/713.28 it/s across the 3 runs) | `tasks.py:L44` | §5.3 |
| Q3‑f | Searchable after rebuild? | **Yes** — `?query=seedqthreecharlie` → count:1 (id 7) | runtime observation; path `views.py:L413-L425` (search read) | §5.4 |
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

**Delete the temporary `Document` records (ids 5, 6, 7) and the temporary `Tag`, restoring the empty baseline:**

```
$ echo "from documents.models import Document, Tag; print('DOCS_DELETED', Document.objects.filter(id__in=[5,6,7]).delete()); print('TAG_DELETED', Tag.objects.filter(name='qnatmptag').delete()); print('DOC_COUNT_NOW', Document.objects.count()); print('TAG_COUNT_NOW', Tag.objects.count())" | python manage.py shell
DOCS_DELETED (4, {'documents.Document_tags': 1, 'documents.Document': 3})
TAG_DELETED (1, {'documents.Tag': 1})
DOC_COUNT_NOW 0
TAG_COUNT_NOW 0
```

**Restore a consistent (now empty, 0‑document) index via a final `document_index reindex`:**

```
$ rm -rf /app/data/index/* && python manage.py document_index reindex

0it [00:00, ?it/s]
0it [00:00, ?it/s]
$ ls -la /app/data/index
total 12
drwxr-sr-x 2 root root 4096 Jul  8 05:59 .
drwxr-sr-x 4 root root 4096 Jul  8 05:59 ..
-rwxr-xr-x 1 root root    0 Jul  8 05:59 MAIN_WRITELOCK
-rw-r--r-- 1 root root 4064 Jul  8 05:59 _MAIN_1.toc
```

The restored index (`MAIN_WRITELOCK` + `_MAIN_1.toc`, 0 documents — the `tqdm` bar reads `0it [00:00, ?it/s]` because there are no documents to index) matches the original 0‑document state the environment started in.

**Remove the temporary observation scripts, and confirm none remain:**

```
$ rm -rf /tmp/qna_seed.py /tmp/qna_q1_cycle.sh /tmp/qna_q1_bulk.sh /tmp/qna_rawsql.py /tmp/qna_q2_reindex.sh /tmp/qna_q2_full.sh /tmp/qna_corrupt2.py /tmp/qna_q3_full.sh /tmp/qna_evidence
$ ls -la /tmp | grep -c qna    # 0 = all investigation scripts removed
0
```

**On the running services and their logs (why they are left in place).** The Redis, Django‑Q `qcluster`, and gunicorn services were **started by the environment setup, not by this investigation** — their PIDs and start times are unchanged from the setup baseline and predate every experiment above (all of which ran between 05:24 and 05:59):

```
$ date -u
Wed Jul  8 05:59:57 UTC 2026
$ ps -o pid,lstart,cmd -p 454,6255,11467
    PID                  STARTED CMD
    454 Wed Jul  8 04:04:05 2026 redis-server *:6379
   6255 Wed Jul  8 04:13:11 2026 python manage.py qcluster
  11467 Wed Jul  8 04:21:30 2026 /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /app/gunicorn.conf.py paperless.asgi:application
```

Because the investigation **started no service of its own**, the correct "leave as found" action is to keep these pre‑existing services running. Their stdout logs `/tmp/qcluster.log` and `/tmp/gunicorn.log` are **service‑owned** files — the live stdout of those still‑running setup processes (the `qcluster` log, for instance, keeps growing as the periodic mail schedule fires) — not artifacts this investigation created, so they are left in place with their owning services. The only `/tmp` files this investigation created were the `qna_*` observation scripts, and those are removed above (the `grep -c qna` count is `0`).

**Final repository state — the sole changed path in the working tree is this document; nothing else under tracked source changed:**

```
$ git status --porcelain --untracked-files=all
 M blitzy/documentation/paperless-ngx_542221a38dff.md
$ git check-ignore data/index data/db.sqlite3
data/index
data/db.sqlite3
```

`git status --porcelain` reports exactly one changed path — this document (`M` = a modification to the already‑tracked deliverable, which is then committed) — and nothing else. The runtime state the experiments touched (`data/index`, `data/db.sqlite3`) is confirmed git‑ignored by `git check-ignore`, so it never appears as a repository change. This confirms the read‑only‑source constraint was honored: no existing source file was modified, added, or deleted; the sole persistent artifact is this document, `blitzy/documentation/paperless-ngx_542221a38dff.md`.
