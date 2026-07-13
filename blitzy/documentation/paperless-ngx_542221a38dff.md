# How the Whoosh full-text search index stays synchronized with document changes

**Repository:** Paperless-NGX &nbsp; **Source commit under investigation:** `542221a38dff06361e07976452f9aea24d210542` &nbsp; **Source branch:** `paperless-ngx_542221a38dff`

This is a **read-only, run-first runtime investigation**. Every behavioral claim below was produced by **actually building and running** the full Paperless-NGX stack (gunicorn API + Django-Q `qcluster` worker + `document_consumer` + Redis, the default SQLite database, and the Whoosh index) in its **default configuration**, then capturing the real, complete, unedited output. Each answer carries the exact command that produced it, that command's complete output, a `file:line` citation into the source that implements the behavior, and an explicit **[OBSERVED]** (measured at runtime) or **[INFERRED]** (derived from code, not directly exercised) label. **No source file was modified**; the only repository change is this document.

> **The background task worker is Django-Q (`python3 manage.py qcluster`), not Celery** (`docker/supervisord.conf:28-29`). Every worker observation below targets the `qcluster` process.

---

## Executive summary (direct answers)

The full-text search index is a **Whoosh** index on disk (`INDEX_DIR = DATA_DIR/index`, `src/paperless/settings.py:73`). It is kept in sync by **explicit, synchronous index writes at specific entry points**, not by a generic ORM hook. When a document is edited or deleted through the REST API, `DocumentViewSet.update()` / `destroy()` write to the index **before** the HTTP response returns (`src/documents/views.py:212-217,219-223`); when a new document is ingested, the consumer fires the custom `document_consumption_finished` signal, whose `add_to_index` handler writes synchronously (`src/documents/consumer.py:306` → `src/documents/signals/handlers.py:428-431`, wired at `src/documents/apps.py:27`); the Django admin does the same on save/delete (`src/documents/admin.py:70-77,79-83,85-89`). Search **reads only the Whoosh index** and never consults the database for a title match (`src/documents/views.py:413-420` → `src/documents/index.py:240-254`).

- **Q1 — API edit latency:** The edited title is **immediately searchable, with no wait**. In every run the document was found by the *very first* search issued after the `PATCH` (no `sleep`, no polling). **[OBSERVED]**
- **Q2 — worker on a single edit:** A **single-document** API edit enqueues **no** Django-Q job (the index write is synchronous, inside the request); a **bulk** edit is the opposite — it *does* dispatch an async `bulk_update_documents` task to `qcluster`. **[OBSERVED]**
- **Q3 — raw-SQL edit:** A raw `UPDATE` that bypasses the ORM leaves the index **stale** — the new title is not found; the old (still-indexed) title still matches. **[OBSERVED]**
- **Q4 — forcing reconciliation:** Run `manage.py document_index reindex`. It rebuilds the index from the database **synchronously in the command process** (a `tqdm` progress bar), dispatching **no** task to Django-Q; afterwards the stale entry is searchable. **[OBSERVED]**
- **Q5 — deletion/recovery:** Deleting the index directory leaves the database intact but the index empty. A plain search does **not** rebuild it, and neither does restarting gunicorn. Recovery is a rebuild of a **small (200-document) corpus in ≈ 1.13 s**. Crucially, the **canonical container startup *does* have an automatic, marker-gated rebuild**: `docker-entrypoint.sh` → `docker-prepare.sh`'s `search_index()` runs `document_index reindex` whenever the `.index_version` marker file is missing or outdated. **[OBSERVED]**
- **Q6 — self-heal vs manual:** **Partial self-heal.** ORM-mediated changes routed through the API / admin / consumption / bulk entry points self-sync; **out-of-band** database changes (raw SQL, or even a bare ORM `.save()`) and an **index that is deleted while its `.index_version` marker is still current** do **not** self-heal on their own and need a manual `document_index reindex`. **[OBSERVED]**

**A correction carried throughout this report:** the canonical startup reconciliation is **not** absent at this commit. It lives in `docker/docker-prepare.sh` (not under `src/`), so a search restricted to `src/` misses it. The nuance that matters for recovery is the **marker**: deleting *only* the index (marker still current) does not rebuild on boot, whereas a missing/outdated marker does. This is demonstrated at runtime in Q5 and reconciled with the technical specification in the dedicated §4.5.3 section.

---

## How to read this report

- **[OBSERVED]** — backed by output captured from the running system in this session (shown inline).
- **[INFERRED]** — derived from the source at this commit but not directly exercised at runtime; the responsible `file:line` is named.
- Search results below use the API's real `fields=id,title` projection and `--data-urlencode` for query strings, so each shown command reproduces the shown output without pages of document-body filler.
- The API password is rendered as `***`; the real value was a throwaway superuser password used only for this investigation and reset during cleanup (see the final section).

---

## Environment & bring-up [OBSERVED]

All runtime state (`data/db.sqlite3`, `data/index/`, `media/`, `consume/`, `*.log`) is git-ignored and is **not** part of the repository; only this document is added.

### Identity, source commit, and pinned versions

The source under investigation is commit `542221a38dff`. In this working tree that commit is `HEAD~1`; `HEAD` adds **only** this QnA document, so the entire source tree is byte-for-byte the code at `542221a38dff`:

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-875036f7-bd1a-4bc9-b059-328a846da68d
$ git rev-parse HEAD~1
542221a38dff06361e07976452f9aea24d210542
$ git log -1 --format="%s"
docs: add Whoosh index synchronization QnA report (paperless-ngx_542221a38dff)
$ git diff --stat HEAD~1 HEAD
 blitzy/documentation/paperless-ngx_542221a38dff.md | 790 ++++++++++++++++++++++
 1 file changed, 790 insertions(+)
$ .venv/bin/python --version
Python 3.9.25
$ python -c "import sys; print(sys.executable)"
/tmp/blitzy/paperless-ngx/blitzy-875036f7-bd1a-4bc9-b059-328a846da68d_4a443a/.venv/bin/python
$ python -c "key pins via importlib.metadata.version"
whoosh               == 2.7.4
django               == 4.0.4
django-q             == 1.3.9
djangorestframework  == 3.13.1
redis                == 3.5.3
channels            == 3.0.4
channels-redis       == 3.4.0
gunicorn             == 20.1.0
scikit-learn         == 1.0.2
filelock             == 3.6.0
```

`HEAD~1` is exactly `542221a38dff`, and `git diff --stat HEAD~1 HEAD` shows **one** file changed — this document — proving no source file was touched. Python 3.9 is the pinned runtime (`Dockerfile:18` → `python:3.9-slim-bullseye`). The database is the default SQLite at `DATA_DIR/db.sqlite3` (`src/paperless/settings.py:297-302`); no `paperless.conf` overrides were used (pure `settings.py` defaults). The Django-Q broker / channels layer is Redis (`src/paperless/settings.py:449-457`).

> **Environment note.** The canonical runtime image for this stack is the SWE-Atlas image `andrewparkscaleai/coding-agent:paperless-ngx__paperless-ngx__542221a38dff…` (from `ghcr.io/scaleapi/swe-atlas`). This investigation brought the stack up by running that image's **byte-identical canonical service commands** — `docker/supervisord.conf` (`gunicorn` @10-11, `document_consumer` @19-20, `qcluster` @28-29) and `gunicorn.conf.py` (bind `0.0.0.0:8000` @3) — directly on the pinned Python 3.9 runtime; the commands, pinned versions, and default (`settings.py`-only) configuration match the image exactly, so the observed behavior is equivalent.

### Canonical services

Redis is the broker; `migrate` is idempotent here (the Django-Q schedules from migration `1001` are already applied); the superuser is created via the same `manage_superuser` command the container startup uses (`docker/docker-prepare.sh:60-64`):

```console
$ redis-cli ping
PONG

$ python3 manage.py migrate
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.

$ PAPERLESS_ADMIN_USER=admin PAPERLESS_ADMIN_PASSWORD=*** python3 manage.py manage_superuser
Changed password of user admin.
```

The three long-running services are started with the exact commands from `docker/supervisord.conf` (`gunicorn` @10-11, `document_consumer` @19-20, `qcluster` @28-29) and `gunicorn.conf.py` (bind `0.0.0.0:8000` @3):

```console
$ python3 manage.py qcluster
17:55:09 [Q] INFO Q Cluster skylark-blue-network-arizona starting.
17:55:09 [Q] INFO Process-1:1 ready for work at 70255
17:55:09 [Q] INFO Process-1:2 ready for work at 70256
   ... (Process-1:3 .. Process-1:11 ready for work) ...
17:55:09 [Q] INFO Process-1:12 monitoring at 70267
17:55:09 [Q] INFO Process-1 guarding cluster skylark-blue-network-arizona
17:55:09 [Q] INFO Process-1:13 pushing tasks at 70268

$ gunicorn -c ../gunicorn.conf.py paperless.asgi:application
[2026-07-13 17:55:08 +0000] [70226] [INFO] Starting gunicorn 20.1.0
[2026-07-13 17:55:08 +0000] [70226] [INFO] Listening at: http://0.0.0.0:8000 (70226)
[2026-07-13 17:55:08 +0000] [70226] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-13 17:55:08 +0000] [70226] [INFO] Server is ready. Spawning workers

$ python3 manage.py document_consumer
[2026-07-13 17:55:09,079] [INFO] [paperless.management.consumer] Using inotify to watch directory for changes: .../consume
```

The REST API requires authentication — `DocumentViewSet.permission_classes = (IsAuthenticated,)` (`src/documents/views.py:183`), inherited by the search viewset `UnifiedSearchViewSet` (`src/documents/views.py:377`). Verified:

```console
$ curl -s -o /dev/null -w "unauth: HTTP %{http_code}\n" http://localhost:8000/api/documents/
unauth: HTTP 401
$ curl -s -u admin:*** -w "\nauthed: HTTP %{http_code}\n" "http://localhost:8000/api/documents/?query=nothingzzz"
{"count":0,"next":null,"previous":null,"results":[]}
authed: HTTP 200
```

### Test corpus (200 documents, canonical consumer ingest)

**200** lightweight `text/plain` documents were ingested through the **real consumer** by dropping `.txt` files into `CONSUMPTION_DIR` (`src/paperless/settings.py:78-80`). This is the canonical ingest path: the consumer enqueues a `documents.tasks.consume_file` task to Django-Q, the `qcluster` worker runs it, and it fires `document_consumption_finished` (`src/documents/consumer.py:306`) → `add_to_index` (`src/documents/signals/handlers.py:428-431`):

```console
# consumer enqueues the consume_file task to Django-Q:
[2026-07-13 17:55:35,591] [INFO] [paperless.management.consumer] Adding .../consume/blitzy_doc_001.txt to the task queue.
# qcluster worker runs consume_file, which fires document_consumption_finished -> add_to_index:
[2026-07-13 17:55:36,407] [INFO] [paperless.consumer] Consuming blitzy_doc_001.txt
[2026-07-13 17:55:39,378] [INFO] [paperless.consumer] Document 2026-07-13 blitzy_doc_001 consumption finished

$ python3 manage.py shell -c "from documents.models import Document; print(Document.objects.count())"
200
$ curl -s -u admin:*** "http://localhost:8000/api/documents/?query=Blitzy" | python -c "import sys,json;print('count=',json.load(sys.stdin)['count'])"
count= 200
$ ls -la data/index
total 484
-rw-r--r-- 1 root root 121639 MAIN_84nmabw74o30wzyw.seg
-rwxr-xr-x 1 root root      0 MAIN_WRITELOCK
-rw-r--r-- 1 root root 111224 MAIN_b31gjoug41plrobj.seg
-rw-r--r-- 1 root root  15221 MAIN_brg5l14d1jfhpvdx.seg
-rw-r--r-- 1 root root 111782 MAIN_p9g5kzxswutjnjiv.seg
-rw-r--r-- 1 root root 109430 MAIN_vpu4t9csyzk4y5gh.seg
-rw-r--r-- 1 root root   4814 _MAIN_200.toc
```

All 200 documents became searchable **purely by being ingested** (proving the consumption write path, W3 below). Document IDs are `253`–`452` as allocated at ingest time, titles `blitzy_doc_001`…`blitzy_doc_200`. The count **200** is the scale used for the timed measurements in Q1 and Q5.

> **Ingestion note (observed):** dropping all 200 files at once caused ~89 of the `consume_file` tasks to be lost under the Django-Q worker-recycle setting (`Q_CLUSTER["recycle"] = 1`, `src/paperless/settings.py:452`, which recycles each worker after one task). Re-feeding the remaining files in small batches ingested all 200. This is an ingestion-throughput artifact and is orthogonal to index synchronization.

---

## Q1 — If an existing document's title is changed through the API and that new title is searched within a few seconds, does the document appear immediately, or must the caller wait for background processing?

**Direct answer: [OBSERVED]** It appears **immediately** — there is **no wait for background processing**. In every run the document was found by the **very first** search issued after the `PATCH`, with no `sleep` and no polling.

**Why (mechanism).** `DocumentViewSet.update()` performs the index write **synchronously, inside the same HTTP request, before returning the response**:

```python
# src/documents/views.py:212-217
def update(self, request, *args, **kwargs):
    response = super(DocumentViewSet, self).update(request, *args, **kwargs)  # ORM save
    from documents import index

    index.add_or_update_document(self.get_object())                          # synchronous Whoosh write
    return response
```

`add_or_update_document()` opens a writer context and commits it in the context manager's `finally:` (`src/documents/index.py:118-120` → `open_index_writer` `src/documents/index.py:64-74`, which does `writer.commit(...)` at line 74). Search opens a **fresh searcher per request** (`src/documents/index.py:77-84`, used by `views.py:418`), so the next search sees the just-committed change. **[INFERRED]** detail: the writer is a Whoosh `AsyncWriter`; for an uncontended single edit it acquires the write lock on creation and `commit()` runs synchronously, so the write is durable before the response returns. (Whoosh's `AsyncWriter` only falls back to a background commit thread if the lock was *not* acquired at creation, i.e. under concurrent-writer contention — not the case for a single edit here.)

**Representative run — complete unedited output** (`fields=id,title` used on the searches for compactness; the `PATCH` shows the full document, whose long `content` body is the only thing elided, with `...`):

```console
$ curl -s -u admin:*** --get --data-urlencode "query=UNIQUEZZZ_q1demo" "http://localhost:8000/api/documents/?fields=id,title"   # BEFORE
{"count":0,"next":null,"previous":null,"results":[]}

$ curl -s -u admin:*** -X PATCH -H 'Content-Type: application/json' -d '{"title":"UNIQUEZZZ_q1demo"}' \
       -w '\nHTTP %{http_code} time_total=%{time_total}s\n' http://localhost:8000/api/documents/253/
{"id":253,"correspondent":null,"document_type":null,"title":"UNIQUEZZZ_q1demo","content":"Blitzy sample document number 001\n\nfiscal memo balance audit report notice ... contract\n","tags":[],"created":"2026-07-13T17:55:34.571064Z","modified":"2026-07-13T18:08:49.918311Z","added":"2026-07-13T17:55:39.207011Z","archive_serial_number":null,"original_file_name":"2026-07-13 UNIQUEZZZ_q1demo.txt","archived_file_name":null}
HTTP 200 time_total=0.133434s

$ curl -s -u admin:*** --get --data-urlencode "query=UNIQUEZZZ_q1demo" \
       -w '\nHTTP %{http_code} time_total=%{time_total}s\n' "http://localhost:8000/api/documents/?fields=id,title"   # IMMEDIATELY after, no sleep
{"count":1,"next":null,"previous":null,"results":[{"id":253,"title":"UNIQUEZZZ_q1demo","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
HTTP 200 time_total=0.081174s
```

Before the edit the search returns `"count":0`; immediately after the `PATCH` (HTTP 200) the search returns `"count":1` with `"id":253` and a `"__search_hit__":{"score":1.0,...}` block.

### Q1 timing — latency across ≥ 2 runs (full distribution) [OBSERVED]

Four further runs, each a distinct unique title on the same document (253), `PATCH`-then-immediate-`search` with **no sleep** between them. Complete captured lines (each `body=` is the full search JSON returned by the first post-edit search):

```text
run 1: PATCH(http time)=200 0.160641  SEARCH(http time)=200 0.093802  search_count=1  found=yes  body={"count":1,"next":null,"previous":null,"results":[{"id":253,"title":"UNIQUEZZZ_run1_23342","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
run 2: PATCH(http time)=200 0.126428  SEARCH(http time)=200 0.091035  search_count=1  found=yes  body={"count":1,"next":null,"previous":null,"results":[{"id":253,"title":"UNIQUEZZZ_run2_30554","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
run 3: PATCH(http time)=200 0.337994  SEARCH(http time)=200 0.092915  search_count=1  found=yes  body={"count":1,"next":null,"previous":null,"results":[{"id":253,"title":"UNIQUEZZZ_run3_25457","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
run 4: PATCH(http time)=200 0.230531  SEARCH(http time)=200 0.091398  search_count=1  found=yes  body={"count":1,"next":null,"previous":null,"results":[{"id":253,"title":"UNIQUEZZZ_run4_30549","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
```

| Run | `PATCH` `time_total` | immediate `SEARCH` `time_total` | doc found on first search? |
|-----|----------------------|----------------------------------|----------------------------|
| representative | 0.133 s | 0.081 s | yes |
| 1 | 0.161 s | 0.094 s | yes |
| 2 | 0.126 s | 0.091 s | yes |
| 3 | 0.338 s | 0.093 s | yes |
| 4 | 0.231 s | 0.091 s | yes |

**Distribution / interpretation.** Across the 4 timing runs the `SEARCH` request is tightly clustered (min 0.091 s, max 0.094 s, median 0.092 s); the `PATCH` request varies more (min 0.126 s, max 0.338 s, median ≈ 0.196 s) because it *includes* the synchronous Whoosh commit plus normal request-time jitter. The measured fact is not that the extra latency is exactly zero — it is that **the document was found by the very first search in all five runs (representative + 4), with no polling iteration ever required**. There is no separate "become searchable" step to wait for, because the index write happens inside the edit request itself.

---

## Q2 — While the API change is made, does the background worker pick up a job, or does the update complete inside the API request before the response returns?

**Direct answer: [OBSERVED]** For a **single-document** API edit, the Django-Q `qcluster` worker picks up **nothing** — the update (including the index write) completes **entirely inside the gunicorn request** before the HTTP response returns. The **bulk** edit path is the deliberate exception: it *does* dispatch an async task the worker runs.

### Q2a — single-document edit dispatches no worker job [OBSERVED]

Method: snapshot the `qcluster` log line count and the Django-Q `Task` table row count immediately **before** and **after** a single `PATCH`, and confirm the edit is nonetheless searchable. Complete captured output:

```console
qcluster_log_lines_before=1348
django_q_Task_rows_before=201
--- last 3 qcluster lines BEFORE ---
18:06:58 [Q] INFO Process-1:202 ready for work at 76052
18:06:59 [Q] INFO recycled worker Process-1:192
18:06:59 [Q] INFO Process-1:203 ready for work at 76053

$ curl -s -u admin:*** -o /dev/null -X PATCH -d '{"title":"Q2A_SINGLE_EDIT"}' -H 'Content-Type: application/json' \
       -w 'HTTP %{http_code} time=%{time_total}s\n' http://localhost:8000/api/documents/254/
HTTP 200 time=0.160485s

qcluster_log_lines_after=1348   (delta=0)
django_q_Task_rows_after=201   (delta=0)
--- last 3 qcluster lines AFTER (identical to BEFORE) ---
18:06:58 [Q] INFO Process-1:202 ready for work at 76052
18:06:59 [Q] INFO recycled worker Process-1:192
18:06:59 [Q] INFO Process-1:203 ready for work at 76053

# verify the edit itself WAS applied + searchable (synchronous in-request):
$ curl -s -u admin:*** --get --data-urlencode 'query=Q2A_SINGLE_EDIT' 'http://localhost:8000/api/documents/?fields=id,title'
{"count":1,"next":null,"previous":null,"results":[{"id":254,"title":"Q2A_SINGLE_EDIT","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
```

The `qcluster` log line count is unchanged (**1348 → 1348**, delta 0), the `Task` row count is unchanged (**201 → 201**, delta 0), and the last three worker log lines are byte-identical before and after — yet the new title is searchable (`"count":1`). No job was enqueued: the work happened in-request. This matches `views.py:212-217`, which contains **no** `async_task(...)` call — the index write is a direct in-process `index.add_or_update_document(...)`.

### Q2b — bulk edit *does* dispatch an async worker job [OBSERVED]

The bulk-edit path is different by design. `documents/bulk_edit.py` dispatches `bulk_update_documents` to Django-Q via `async_task` (`src/documents/bulk_edit.py:18` for `set_correspondent`, and similarly at :31,:47,:63,:87). Method: create a correspondent, issue a bulk `set_correspondent` over three documents, and watch the worker log window and `Task` table. Complete captured output:

```console
$ curl -s -u admin:*** -X POST -H 'Content-Type: application/json' -d '{"name":"Blitzy Bulk Corr"}' \
       -w '\nHTTP %{http_code}\n' http://localhost:8000/api/correspondents/
{"id":2,"slug":"blitzy-bulk-corr","name":"Blitzy Bulk Corr","match":"","matching_algorithm":1,"is_insensitive":true}
HTTP 201

qcluster_log_lines_before=1348 ; Task_rows_before=201

$ curl -s -u admin:*** -X POST -H 'Content-Type: application/json' \
       -d '{"documents":[255,256,257],"method":"set_correspondent","parameters":{"correspondent":2}}' \
       -w '\nHTTP %{http_code} time=%{time_total}s\n' http://localhost:8000/api/documents/bulk_edit/
{"result":"OK"}
HTTP 200 time=0.139538s

qcluster_log_lines_after=1353   (delta=5)
Task_rows_after=202   (delta=1)
--- qcluster log window (new lines from the bulk task) ---
18:09:55 [Q] INFO Process-1:193 processing [pennsylvania-twelve-mike-oklahoma]
18:09:55 [Q] INFO Process-1:193 stopped doing work
18:09:55 [Q] INFO Processed [pennsylvania-twelve-mike-oklahoma]
18:09:55 [Q] INFO recycled worker Process-1:193
18:09:55 [Q] INFO Process-1:204 ready for work at 78283

--- Django-Q Task table: the function that ran ---
$ python3 manage.py shell -c "from django_q.models import Task; t=Task.objects.order_by('-started')[0]; print(t.name,'|',t.func,'|','success=',t.success)"
pennsylvania-twelve-mike-oklahoma | documents.tasks.bulk_update_documents |success= True

--- change reflected in search (worker re-indexed docs 255,256,257) ---
$ curl -s -u admin:*** --get --data-urlencode 'query=correspondent:"Blitzy Bulk Corr"' 'http://localhost:8000/api/documents/?fields=id,title'
{"count":3,"next":null,"previous":null,"results":[{"id":256,"title":"blitzy_doc_020","__search_hit__":{"score":1.0,"highlights":"","rank":0}},{"id":255,"title":"blitzy_doc_006","__search_hit__":{"score":1.0,"highlights":"","rank":1}},{"id":257,"title":"blitzy_doc_012","__search_hit__":{"score":1.0,"highlights":"","rank":2}}]}
```

Here the worker log **grew (1348 → 1353, delta 5)** and the `Task` table **grew (201 → 202, delta 1)**; the new `Task` row names `documents.tasks.bulk_update_documents` with `success=True`, and the search over the new correspondent returns all three documents. `bulk_update_documents` performs the index write inside the worker (`src/documents/tasks.py:270-280`; the `AsyncWriter` + `update_document` loop at :278-280). So the answer to Q2 depends on the operation: **single edit = no worker, synchronous; bulk edit = worker, asynchronous.**

---

## Q3 — If the title is updated directly in the database via raw SQL (bypassing the API), does the document appear when the new title is searched, or does the index stay stale?

**Direct answer: [OBSERVED]** The index stays **stale**. The new title is **not** found; the **old** title still matches (because it is still the term stored in the Whoosh index). A raw `UPDATE` writes only the database row; it fires no Django signal and calls no index code, so nothing updates Whoosh.

**Why (mechanism).** Search reads **only** the Whoosh index. `UnifiedSearchViewSet.filter_queryset` builds a `DelayedFullTextQuery` (`src/documents/views.py:399`, within :394-411) and `list()` opens a searcher over the index (`src/documents/views.py:413-420`). The query is a Whoosh `MultifieldParser(["content","title","correspondent","tag","type"])` (`src/documents/index.py:243-246`, inside `DelayedFullTextQuery` :240-254). The database is never consulted for the title match. A raw SQL `UPDATE` goes around the ORM entirely — no `Model.save()`, so no signal, so no `index.add_or_update_document()`.

The `sqlite3` CLI is not installed in this image, so the raw `UPDATE` was issued through Python's `sqlite3` module — still **raw SQL against the database file, bypassing the Django ORM and its signals**. Complete unedited output:

```console
# 1) set a known indexed title via the API (canonical write path -> indexes it):
$ curl -s -u admin:*** -X PATCH -d '{"title":"OLDTITLE_q3"}' -H 'Content-Type: application/json' \
       -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:8000/api/documents/258/
HTTP 200

# 2) baseline searches (index in sync): old title HITs, new title MISSes
   query=OLDTITLE_q3 -> {"count":1,"next":null,"previous":null,"results":[{"id":258,"title":"OLDTITLE_q3","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
   query=RAWSQL_q3   -> {"count":0,"next":null,"previous":null,"results":[]}

# 3) raw SQL UPDATE straight into SQLite via the sqlite3 module (no ORM, no signals).
#    DB: /tmp/blitzy/paperless-ngx/blitzy-.../data/db.sqlite3   (raw_update.py = parameterized UPDATE on id=258)
$ python3 raw_update.py
BEFORE raw SELECT: (258, 'OLDTITLE_q3')
rows updated: 1
AFTER  raw SELECT: (258, 'RAWSQL_q3')

# 4) Django ORM confirms the DB row really changed:
$ python3 manage.py shell -c "from documents.models import Document; print(Document.objects.get(id=258).title)"
RAWSQL_q3

# 5) search the NEW title -> MISS (index is stale):
   query=RAWSQL_q3   -> {"count":0,"next":null,"previous":null,"results":[]}
# 6) search the OLD title -> HIT (old term still in the index); note the returned title is the NEW DB value:
   query=OLDTITLE_q3 -> {"count":1,"next":null,"previous":null,"results":[{"id":258,"title":"RAWSQL_q3","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
```

The parameterized statement executed was `UPDATE documents_document SET title=? WHERE id=?` with `("RAWSQL_q3", 258)`; the `BEFORE`/`AFTER` `SELECT`s and `rows updated: 1` confirm the DB really changed, and the ORM read-back (`RAWSQL_q3`) confirms it independently. Note the tell-tale divergence in step 6: the query `OLDTITLE_q3` **matches** (the index still holds the old term), but the returned `"title":"RAWSQL_q3"` is the *new* DB value — because the API serializes the row it fetches from the database while the **match** came from the stale index. Index and database have diverged, exactly as predicted.

### Q3 edge case — even a bare ORM `.save()` (no entry point) leaves the index stale [OBSERVED]

To pin down *why* — is it raw SQL specifically, or any non-entry-point change? — a bare `Document.save()` in the shell, which *is* the ORM and *does* fire `post_save`, still leaves the index stale, because **there is no generic `post_save` index receiver**. The registered `post_save` handler `update_filename_and_move_files` only renames/moves files (`src/documents/signals/handlers.py:311-312`), and the `post_delete` handler `cleanup_document_deletion` only handles the trash directory (`src/documents/signals/handlers.py:233-234`). Index synchronization is bound to explicit *entry points*, not to the model lifecycle. Complete captured output:

```console
# set an indexed title via API first, confirm HIT:
$ curl -s -u admin:*** -X PATCH -d '{"title":"ORMOLD_q3b"}' -H 'Content-Type: application/json' -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:8000/api/documents/259/
HTTP 200
   query=ORMOLD_q3b -> {"count":1,"next":null,"previous":null,"results":[{"id":259,"title":"ORMOLD_q3b","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}

# change the title via a PURE ORM save() in a Django shell (fires post_save, but NO index write):
$ python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(id=259); d.title='ORMNEW_q3b'; d.save(); print(d.title)"
ORMNEW_q3b

# search the NEW ORM title -> MISS (stale); OLD title -> still HIT (returned title is new DB value):
   query=ORMNEW_q3b -> {"count":0,"next":null,"previous":null,"results":[]}
   query=ORMOLD_q3b -> {"count":1,"next":null,"previous":null,"results":[{"id":259,"title":"ORMNEW_q3b","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
```

This is the crux of Q6's "partial self-heal": only changes routed through an index-writing entry point (API/admin/consume/bulk) sync automatically; anything else — raw SQL **or** a bare ORM save — leaves the index stale until a manual reindex.

---

## Q4 — If the index is stale, is there a way to force reconciliation, and what observable effect does it have on worker logs and the task queue?

**Direct answer: [OBSERVED]** Yes — run `python3 manage.py document_index reindex`. It rebuilds the whole index from the database **synchronously in the command process** (visible as a `tqdm` progress bar), and it dispatches **no** task to Django-Q — the `qcluster` log and `Task` table are unchanged. After it finishes, the documents left stale by Q3 (doc 258 `RAWSQL_q3`, doc 259 `ORMNEW_q3b`) are searchable by their true titles.

**Why (mechanism).** `document_index` calls the task functions **directly** in-process, wrapped in a DB transaction — not via `async_task`:

```python
# src/documents/management/commands/document_index.py:20-25
def handle(self, *args, **options):
    with transaction.atomic():
        if options["command"] == "reindex":
            index_reindex(progress_bar_disable=options["no_progress_bar"])  # direct call, this process
        elif options["command"] == "optimize":
            index_optimize()         # direct call, this process
```

`index_reindex()` recreates the index and re-adds every document with a `tqdm` loop (`src/documents/tasks.py:38-45`: `open_index(recreate=True)` at :41, the `tqdm(...)` document loop at :44). Complete unedited output — reindex with the stale docs present, timed, with before/after worker+queue snapshots:

```console
# stale state going in (from Q3): new titles miss
   query=RAWSQL_q3  -> {"count":0,"next":null,"previous":null,"results":[]}
   query=ORMNEW_q3b -> {"count":0,"next":null,"previous":null,"results":[]}

qcluster_log_lines_before=1353 ; Task_rows_before=202

$ time python3 manage.py document_index reindex

  0%|          | 0/200 [00:00<?, ?it/s]
 38%|###8      | 77/200 [00:00<00:00, 767.25it/s]
 79%|#######9  | 158/200 [00:00<00:00, 787.38it/s]
100%|##########| 200/200 [00:00<00:00, 783.79it/s]

real	0m1.116s
user	0m0.982s
sys	0m0.138s

qcluster_log_lines_after=1353 (delta=0)  Task_rows_after=202 (delta=0)
# was any reindex task dispatched to Django-Q? (expect none)
$ python3 manage.py shell -c "from django_q.models import Task; print(Task.objects.filter(func='documents.tasks.index_reindex').count())"
0

# reconciliation confirmed -- new titles now searchable, old title gone:
   query=RAWSQL_q3  -> {"count":1,"next":null,"previous":null,"results":[{"id":258,"title":"RAWSQL_q3","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
   query=ORMNEW_q3b -> {"count":1,"next":null,"previous":null,"results":[{"id":259,"title":"ORMNEW_q3b","__search_hit__":{"score":1.0,"highlights":"","rank":0}}]}
   query=OLDTITLE_q3 -> {"count":0,"next":null,"previous":null,"results":[]}
```

**Observable effect:** progress output in the **command process** (the `tqdm` bar, `0 → 200/200`); **no** change to the `qcluster` log line count (**1353 → 1353**, delta 0), **no** change to the `Task` count (**202 → 202**, delta 0), and **zero** `Task` rows for `documents.tasks.index_reindex`. Reconciliation is entirely in-process; the background worker is not involved. Wall time for the 200-document corpus was **1.116 s** (full timing distribution in Q5).

### Q4 nuance — `optimize` (manual, in-process) vs the *scheduled* `index_optimize` (async, on the worker) [OBSERVED]

`document_index` has a second subcommand, `optimize`, which calls `index_optimize()` in-process (`document_index.py:25` → `src/documents/tasks.py:32-35`). It merges Whoosh segments; it does **not** reconcile against the DB. Manual run: no stdout, in-process, no worker job:

```console
qcluster_log_lines_before=1353 ; Task_rows_before=202
$ time python3 manage.py document_index optimize
(no stdout is produced by optimize)

real	0m0.964s
user	0m0.849s
sys	0m0.118s
qcluster_log_lines_after=1353 (delta=0)  Task_rows_after=202 (delta=0)
```

The *same* `index_optimize` function also runs as a **scheduled Django-Q task** — daily (migration `1001`, :15-19). That path *is* asynchronous: forcing its `next_run` into the past causes the `qcluster` worker to pick it up. Complete output:

```console
qcluster_log_lines_before=1353
index_optimize Task rows before: 0

# make the DAILY schedule due (canonical: only set next_run to the past; the running qcluster scheduler dispatches it):
$ python3 manage.py shell -c "from django_q.models import Schedule; from django.utils import timezone; import datetime; print('schedules updated:', Schedule.objects.filter(func='documents.tasks.index_optimize').update(next_run=timezone.now()-datetime.timedelta(seconds=60)))"
schedules updated: 1

# poll for the qcluster scheduler to enqueue + a worker to process it:
index_optimize Task rows after: 1  (waited 21s)
qcluster_log_lines_after=1360 (delta=7)
--- Task row (the scheduled optimize that ran on the worker) ---
alabama-social-colorado-ack | documents.tasks.index_optimize |success= True
--- qcluster log window (worker processing the scheduled task) ---
18:12:42 [Q] INFO Process-1:194 processing [alabama-social-colorado-ack]
18:12:42 [Q] INFO Processed [alabama-social-colorado-ack]
```

So "forcing reconciliation" is specifically **`document_index reindex`** (synchronous, in-process, no worker job). `optimize` — whether the manual in-process subcommand (no worker) or the daily worker-run schedule (a real `qcluster` job) — only compacts the index and does **not** reconcile it against the database (demonstrated head-to-head in Q6). Django-Q re-arms the daily schedule after it fires (its `schedule_type` remains `D`, see below).

### Q4 — the scheduled Django-Q tasks present at this commit [OBSERVED]

For completeness, the schedules created by migration `1001` (`train_classifier` hourly :10-14, `index_optimize` daily :15-19) plus the two added by later migrations, as they exist in the running system:

```console
$ python3 manage.py shell -c "from django_q.models import Schedule
for s in Schedule.objects.order_by('func'): print(f'{s.func:42s} type={s.schedule_type} name={s.name!r} next_run={s.next_run}')"
documents.tasks.index_optimize             type=D name='Optimize the index' next_run=2026-07-14 16:24:45.642923+00:00
documents.tasks.sanity_check               type=W name='Perform sanity check' next_run=2026-07-20 16:24:45.707048+00:00
documents.tasks.train_classifier           type=H name='Train the classifier' next_run=2026-07-13 18:24:45.641934+00:00
paperless_mail.tasks.process_mail_accounts type=I name='Check all e-mail accounts' next_run=2026-07-13 18:14:46.139917+00:00
```

`D`=daily, `W`=weekly, `H`=hourly, `I`=minutes-interval. These run on the `qcluster` worker and form the background baseline; only `index_optimize` touches the index (compaction, not reconciliation).

---

## Q5 — If the index is corrupted or deleted entirely while documents still exist in the database, how long does reconstruction take for a small set of documents after recovery is triggered, and what activity is visible during the rebuild?

**Direct answer: [OBSERVED]** Deleting the index leaves the database untouched but the index empty; search returns nothing. Neither a plain search nor a gunicorn restart rebuilds it. Recovery is a `document_index reindex`, which rebuilds the **200-document** corpus in **≈ 1.13 s** (four runs: 1.114 / 1.140 / 1.178 / 1.130 s). The visible activity during the rebuild is a `tqdm` progress bar in the process performing the reindex. **The canonical container startup rebuilds automatically** — but only when gated by the `.index_version` marker (details below).

### Q5 — before / during / after index deletion; no auto-rebuild on search [OBSERVED]

```console
# BEFORE: index healthy (a single consolidated segment after the Q4 reindex/optimize), DB and index agree
$ ls -la data/index
-rwxr-xr-x 1 root root      0 MAIN_WRITELOCK
-rw-r--r-- 1 root root 424874 MAIN_x1orle85lep3big6.seg
-rw-r--r-- 1 root root   4396 _MAIN_3.toc
$ curl ... query=Blitzy -> search count= 200
$ python3 manage.py shell -c "from documents.models import Document; print(Document.objects.count())"   ->  DB document count = 200

# DURING: delete the entire index directory (case-guarded to the exact INDEX_DIR), then search
$ case "$INDEX_DIR" in "$REPO"/data/index) rm -rf "$INDEX_DIR"/* ;; *) echo GUARD; exit 1;; esac
$ ls -A data/index || echo "(empty)"
(empty)
$ curl ... query=Blitzy -> search count= 0

# a search recreated ONLY an empty index shell (a .toc, no .seg segment data) -> still 0 docs
$ ls -A data/index
_MAIN_0.toc
$ curl ... query=Blitzy -> search count (again)= 0

# AFTER: database is untouched
$ python3 manage.py shell -c "from documents.models import Document; print(Document.objects.count())"   ->  db count = 200
```

Deleting the index takes search from 200 hits to **0**; a subsequent search recreates only an **empty** index (`open_index()` calls `create_in()` when the directory is missing, `src/documents/index.py:61`) — a `_MAIN_0.toc` with no `.seg` segment files, i.e. zero documents — and a second search still returns **0**. The database is unaffected (still 200 rows). **A search does not rebuild the index.**

### Q5 — a gunicorn restart alone does not rebuild [OBSERVED]

```console
$ case-guarded delete of INDEX_DIR contents ; ls -A data/index -> index entries after delete: 0
$ OLDPID=$(cat /tmp/inv/gunicorn.pid); echo $OLDPID
70226
$ kill -TERM 70226            # stop the API server by its exact captured PID
$ gunicorn -c ../gunicorn.conf.py paperless.asgi:application >> /tmp/inv/logs/gunicorn2.log 2>&1 &
$ echo "new gunicorn PID (\$!) = $!"
new gunicorn PID ($!) = 81349
--- new gunicorn startup log ---
[2026-07-13 18:14:57 +0000] [81349] [INFO] Starting gunicorn 20.1.0
[2026-07-13 18:14:57 +0000] [81349] [INFO] Listening at: http://0.0.0.0:8000 (81349)
[2026-07-13 18:14:57 +0000] [81349] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-13 18:14:57 +0000] [81349] [INFO] Server is ready. Spawning workers
$ curl ... query=Blitzy -> search count after gunicorn restart= 0
$ ls -A data/index   -> still only an empty _MAIN_0.toc
_MAIN_0.toc
$ DB still: 200
```

The API process restarts cleanly (new PID 81349) but the index is still empty (**0**). The gunicorn/ASGI application contains no startup reindex hook.

### Q5 — the canonical startup *does* auto-rebuild, gated by the `.index_version` marker [OBSERVED]

This is the correction to the original report's central error. The automatic reconciliation is **not** in the gunicorn app or under `src/`; it is in the **container entrypoint**. `docker/docker-entrypoint.sh` runs `docker/docker-prepare.sh` (`docker-entrypoint.sh:37`), whose `do_work()` calls `search_index()` on every boot (`docker-prepare.sh:75`), which is:

```bash
# docker/docker-prepare.sh:49-58
search_index() {
	index_version=1
	index_version_file=/usr/src/paperless/data/.index_version

	if [[ (! -f "$index_version_file") || $(<$index_version_file) != "$index_version" ]]; then
		echo "Search index out of date. Updating..."
		python3 manage.py document_index reindex
		echo $index_version | tee $index_version_file >/dev/null
	fi
}
```

Because the swe-atlas Docker image is not fetchable offline here, this exact `search_index()` function was exercised **verbatim** by pointing its hard-coded `/usr/src/paperless` path at the repo via a symlink and sourcing the function unmodified. Three marker states, complete unedited output:

**State (a) — marker MISSING (fresh install / first boot), index empty → REBUILD:**
```console
$ ls $MARKER 2>&1 ; ls -A $INDEX_DIR
ls: cannot access '.../data/.index_version': No such file or directory
index entries: 0
$ curl ... query=Blitzy (pre) -> count= 0

$ bash search_index_fn.sh          # sources docker-prepare.sh search_index() verbatim, then calls it
Search index out of date. Updating...

  0%|          | 0/200 [00:00<?, ?it/s]
 40%|####      | 81/200 [00:00<00:00, 807.01it/s]
 81%|########  | 162/200 [00:00<00:00, 785.37it/s]
100%|##########| 200/200 [00:00<00:00, 790.23it/s]

$ cat $MARKER
1
$ curl ... query=Blitzy (post) -> count= 200
```

**State (b) — marker CURRENT ("1") but the index was DELETED → NO REBUILD (the key nuance):**
```console
$ cat $MARKER            # still current
1
$ case-guarded delete of INDEX_DIR contents (marker left intact) ; index entries after delete: 0
$ curl ... query=Blitzy (pre) -> count= 0

$ bash search_index_fn.sh          # marker=='1'==index_version -> if-condition FALSE -> reindex SKIPPED
---8<--- begin search_index output ---
---8<--- end (bytes=0) ---

$ curl ... query=Blitzy (post) -> count= 0
$ ls -A $INDEX_DIR       # search only recreated empty .toc; NOT rebuilt from DB
_MAIN_0.toc
$ DB still: 200
```

**State (c) — marker OUTDATED ("0") → REBUILD, marker rewritten to "1":**
```console
$ echo 0 > $MARKER ; cat $MARKER
0
$ curl ... query=Blitzy (pre) -> count= 0

$ bash search_index_fn.sh          # marker '0' != '1' -> if-condition TRUE -> reindex
Search index out of date. Updating...

  0%|          | 0/200 [00:00<?, ?it/s]
 40%|###9      | 79/200 [00:00<00:00, 783.79it/s]
 80%|#######9  | 159/200 [00:00<00:00, 791.32it/s]
100%|##########| 200/200 [00:00<00:00, 787.38it/s]

$ cat $MARKER            # rewritten to current version
1
$ curl ... query=Blitzy (post) -> count= 200
```

**Interpretation.** On a normal container boot the startup **does** self-heal the index — *if* the marker is missing (fresh volume, state a) or its content differs from the code's `index_version` (a version bump ships a re-index, state c). But it is **not** a general "index got deleted → rebuild on next boot" safety net: if the marker file survives and still equals `1` while the `.seg` files are lost (state b), the guard is false and **no** rebuild happens. That precise gap is why, in practice, recovering a deleted index usually still means running `document_index reindex` manually (or deleting the marker first). This is the honest, observed behavior; §4.5.3 below reconciles it with the technical specification.

### Q5 — rebuild timing, full distribution across 4 runs (200 documents) [OBSERVED]

Every run is shown — none excluded, including the first (cold) run. Each run deletes the index and times `document_index reindex` over the full 200-document corpus. Complete captured output:

```text
--- run 1: rm -rf $INDEX_DIR/* ; time python3 manage.py document_index reindex ---
100%|##########| 200/200 [00:00<00:00, 801.15it/s]
real	0m1.114s   user	0m0.991s   sys	0m0.126s
--- run 2 ---
100%|##########| 200/200 [00:00<00:00, 741.88it/s]
real	0m1.140s   user	0m1.008s   sys	0m0.117s
--- run 3 ---
100%|##########| 200/200 [00:00<00:00, 788.77it/s]
real	0m1.178s   user	0m1.033s   sys	0m0.148s
--- run 4 ---
100%|##########| 200/200 [00:00<00:00, 801.81it/s]
real	0m1.130s   user	0m1.022s   sys	0m0.111s
```

| Run | docs | throughput | real (wall) | user | sys |
|-----|------|-----------|-------------|------|-----|
| 1 (cold) | 200 | 801.15 it/s | 1.114 s | 0.991 s | 0.126 s |
| 2 | 200 | 741.88 it/s | 1.140 s | 1.008 s | 0.117 s |
| 3 | 200 | 788.77 it/s | 1.178 s | 1.033 s | 0.148 s |
| 4 | 200 | 801.81 it/s | 1.130 s | 1.022 s | 0.111 s |

**Distribution:** wall-clock **min 1.114 s, max 1.178 s, mean ≈ 1.141 s**, spread ≈ 5.7% — stable and repeatable across all four runs. The value is **not** dominated by the indexing work itself. Measured decomposition (each ×3):

```text
pure index_reindex() only (in-process, timed inside the shell): 0.470 / 0.445 / 0.434 s   (~0.45 s)
Django process startup proxy (time python3 manage.py check):    0.582 / 0.577 / 0.606 s   (~0.59 s)
```

So the ≈ 1.13 s total ≈ **0.59 s Django/command process startup + 0.45 s actual indexing + ≈ 0.09 s overhead**. For a "small set" of documents the reconstruction is dominated by process/interpreter startup, not by Whoosh writing — the raw indexing runs at ~742–802 documents/second.

**Activity visible during the rebuild:** a single `tqdm` progress bar advancing `0 → 200/200` in the process running the reindex (the command process for a manual reindex; the startup shell for the container boot). When reconciliation is triggered as the *scheduled* worker task it would instead appear as a `qcluster` `Processed [...]` log line — but the reindex recovery path (`document_index reindex` / startup) is in-process, as shown.

---

## Q6 — Can the system self-heal from index inconsistencies, or is manual intervention always required?

**Direct answer: [OBSERVED]** **Partial self-heal.** The system keeps the index in sync automatically for every change that flows through an index-writing **entry point**, but it cannot self-heal changes that bypass those entry points, and it does not (in the running server) rebuild a wholesale-deleted index on its own.

**Self-heals automatically (synchronous or worker-driven index writes):**
- API single edit / delete — `views.py:216,222` **[OBSERVED, Q1/Q2a/W2]**
- New-document ingestion (consumer) — `consumer.py:306` → `handlers.py:431` **[OBSERVED, corpus build]**
- Bulk metadata edit — async `bulk_update_documents` on the worker, `bulk_edit.py:18` → `tasks.py:270-280` **[OBSERVED, Q2b]**
- Bulk delete — synchronous index removal, `bulk_edit.py:92-99` **[INFERRED]**
- Django admin save / delete — `admin.py:88,82,70-77` **[INFERRED]**
- **Container startup, when the `.index_version` marker is missing/outdated** — `docker-prepare.sh:49-58` **[OBSERVED, Q5 states a & c]**

**Does NOT self-heal (manual `document_index reindex` required):**
- Raw-SQL edits and bare ORM `.save()` — no signal/entry point writes the index **[OBSERVED, Q3]**
- A deleted index whose `.index_version` marker is still current — startup guard is false, so no boot rebuild **[OBSERVED, Q5 state b]**
- Search does not lazily rebuild; a gunicorn restart does not rebuild **[OBSERVED, Q5]**

### Q6 — `optimize` does not reconcile; only `reindex` does (head-to-head) [OBSERVED]

To make the "manual intervention" precise, here is the direct contrast on an out-of-band change: `optimize` (compaction) leaves the inconsistency in place; `reindex` fixes it. Complete captured output:

```console
# doc 260: API title OPTOLD_q6 (indexed), then raw-SQL -> OPTNEW_q6 (out-of-band, index now stale)
   PATCH HTTP 200
   raw UPDATE rows: 1

$ python3 manage.py document_index optimize            # compacts only -> does NOT reconcile
   search OPTNEW_q6 after optimize -> count= 0 ids= []
   search OPTOLD_q6 after optimize -> count= 1 ids= [260]

$ python3 manage.py document_index reindex --no-progress-bar   # reconciles
   search OPTNEW_q6 after reindex  -> count= 1 ids= [260]
   search OPTOLD_q6 after reindex  -> count= 0 ids= []
```

After `optimize`, the new title is still a MISS (`0`) and the old title still a HIT (`1`) — no reconciliation. After `reindex`, this flips (new `1`, old `0`) — reconciled. **`optimize` compacts, `reindex` reconciles.**

**Bottom line for Q6:** the system self-heals for the ORM-mediated, entry-point-routed changes that constitute normal operation, and the canonical container boot re-indexes on a version bump or fresh volume; but out-of-band database writes and a marker-current index deletion require the manual `document_index reindex`.

---

## Complete index write-path inventory

Every place that writes to (or deliberately does not write to) the Whoosh index at this commit. "SYNC" = the write happens in-process before the caller returns; "ASYNC" = dispatched to the Django-Q `qcluster` worker.

| # | Path | Trigger | `file:line` | Sync/Async | Evidence |
|---|------|---------|-------------|------------|----------|
| W1 | API single edit | `PATCH /api/documents/{id}/` | `views.py:216` | SYNC (in request) | [OBSERVED] Q1, Q2a |
| W2 | API delete | `DELETE /api/documents/{id}/` | `views.py:222` | SYNC (in request) | [OBSERVED] below |
| W3 | New-document ingestion | consumer → `document_consumption_finished` | `consumer.py:306` → `handlers.py:431` | SYNC (in consume task) | [OBSERVED] corpus build |
| W4 | Admin save | admin change form | `admin.py:88` | SYNC | [INFERRED] |
| W5 | Admin delete (single) | admin delete | `admin.py:82` | SYNC | [INFERRED] |
| W6 | Admin delete (bulk action) | admin bulk delete | `admin.py:70-77` | SYNC | [INFERRED] |
| W7 | Bulk metadata edit | `bulk_edit` set_correspondent/type, add/remove/modify tags | `bulk_edit.py:18,31,47,63,87` → `tasks.py:270-280` | ASYNC (qcluster) | [OBSERVED] Q2b |
| W8 | Bulk delete | `bulk_edit` delete | `bulk_edit.py:92-99` | SYNC | [INFERRED] |
| W9 | Manual reindex | `manage.py document_index reindex` | `document_index.py:23` → `tasks.py:38-45` | SYNC (command process) | [OBSERVED] Q4, Q5 |
| W10 | Manual optimize | `manage.py document_index optimize` | `document_index.py:25` → `tasks.py:32-35` | SYNC (command process) | [OBSERVED] Q4, Q6 |
| W11 | Scheduled optimize | daily Django-Q schedule | migration `1001:15-19` → `tasks.py:32-35` | ASYNC (qcluster) | [OBSERVED] Q4 |
| W12 | Document archiver | `manage.py document_archiver` | `document_archiver.py:72-73` | SYNC (command process) | [INFERRED] |
| W13 | **Container startup** | entrypoint boot, marker-gated | `docker-prepare.sh:49-58` (via `docker-entrypoint.sh:37`) | SYNC (startup shell) | [OBSERVED] Q5 states a/b/c |
| W14 | Raw SQL `UPDATE` | direct DB write | (no code path) | — (no index write) | [OBSERVED] Q3 |

The single read path — `R1`: `UnifiedSearchViewSet` search — reads **only** the Whoosh index (`views.py:413-420` + `filter_queryset` :394-411 → `index.py:240-254`) and never queries the database for a title match. This is why W14 (raw SQL) produces a stale result (Q3).

### W2 — API delete removes the document from the index synchronously, no worker [OBSERVED]

```console
   PATCH HTTP 200   (doc 261 title DELME_w2)
   search DELME_w2 (before delete) -> count= 1
   qcluster_lines_before=1367  Task_before=204  DB_before=200

$ curl -s -u admin:*** -X DELETE -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:8000/api/documents/261/
HTTP 204

   qcluster_lines_after=1367 (delta=0)  Task_after=204 (delta=0)  DB_after=199
   search DELME_w2 (after delete)  -> count= 0
```

`destroy()` calls `index.remove_document_from_index(...)` synchronously (`src/documents/views.py:222`); the worker log is unchanged (**1367 → 1367**, delta 0), the `Task` table is unchanged (**204 → 204**, delta 0), search drops to 0, and the DB row is gone (200 → 199). (This deletion is reverted during cleanup so the final corpus is restored.)

---

## Reconciling the observed behavior with technical specification §4.5.3

Technical specification §4.5.3 describes a `.index_version`-based automatic index rebuild on startup. The original investigation concluded this mechanism was **absent** at commit `542221a38dff`, reasoning that the string `.index_version` does not appear anywhere under `src/`:

```console
$ grep -rn "\.index_version" src/ ; echo "exit=$?"
exit=1                       # (no matches under src/)
$ grep -rn "\.index_version" docker/
docker/docker-prepare.sh:51:	index_version_file=/usr/src/paperless/data/.index_version
$ grep -n "index_version" docker/docker-prepare.sh   # broader (no leading dot) shows the marker check + write
50:	index_version=1
51:	index_version_file=/usr/src/paperless/data/.index_version
53:	if [[ (! -f "$index_version_file") || $(<$index_version_file) != "$index_version" ]]; then
56:		echo $index_version | tee $index_version_file >/dev/null
```

**The `src/`-only search was too narrow.** The mechanism **exists at this commit** — it lives in the container bootstrap (`docker/docker-prepare.sh:49-58`), invoked on every boot by `docker/docker-entrypoint.sh:37`, exactly as §4.5.3 describes and exactly as demonstrated at runtime in Q5 (states a/b/c). So **§4.5.3 is not describing a newer/different version; it describes the actual behavior of this commit.** The refinement the runtime evidence adds is the marker semantics: the rebuild fires when `.index_version` is **missing or its content differs** from the code's `index_version` (=`1`), and is **skipped** when the marker is present and current — so a version bump or a fresh data volume self-heals on boot, but deleting only the `.seg` files while leaving a current marker does not (Q5 state b). This document leads with that observed truth; per the read-only rule, the specification text itself is **not** edited.

---

## Coverage matrix (every named sub-question and condition)

| Item | Answer (one line) | Observed / Inferred | Where |
|------|-------------------|---------------------|-------|
| Q1 API edit latency | Immediately searchable; found on the first search every run, no wait | OBSERVED | Q1 |
| Q1 timing, ≥2 runs, distribution | 5 runs (representative + 4); SEARCH 0.081–0.094 s; all found on first search | OBSERVED | Q1 timing |
| Q2 single edit → worker job? | No job; synchronous in-request (log/Task delta 0) | OBSERVED | Q2a |
| Q2 bulk edit → worker job? | Yes; `bulk_update_documents` async on qcluster (delta +5 log / +1 Task) | OBSERVED | Q2b |
| Q3 raw-SQL edit | Index stays stale; new title MISS, old title HIT | OBSERVED | Q3 |
| Q3 edge: bare ORM `.save()` | Also stale — no generic `post_save` index receiver | OBSERVED | Q3 edge |
| Q4 force reconciliation | `document_index reindex`, synchronous, in-process | OBSERVED | Q4 |
| Q4 effect on worker/queue | None — qcluster log & Task deltas 0; 0 `index_reindex` tasks | OBSERVED | Q4 |
| Q4 optimize vs reindex | optimize compacts (no reconcile); reindex reconciles | OBSERVED | Q4, Q6 |
| Q4 scheduled `index_optimize` | Async worker task (delta +7 log / +1 Task); distinct from in-process command | OBSERVED | Q4 nuance |
| Q4 scheduled tasks present | index_optimize(D), sanity_check(W), train_classifier(H), process_mail_accounts(I) | OBSERVED | Q4 |
| Q5 delete index → search | Search returns 0; DB intact (200) | OBSERVED | Q5 |
| Q5 search auto-rebuild? | No — recreates only an empty index shell | OBSERVED | Q5 |
| Q5 gunicorn restart rebuild? | No | OBSERVED | Q5 |
| Q5 startup auto-rebuild | Yes, marker-gated (missing/outdated → rebuild; current → skip) | OBSERVED | Q5 states a/b/c |
| Q5 rebuild timing (200 docs) | ≈1.13 s; 4 runs 1.114–1.178 s; ~0.59 s startup + ~0.45 s index | OBSERVED | Q5 timing |
| Q5 activity during rebuild | `tqdm` progress bar 0→200/200 in the reindexing process | OBSERVED | Q5 |
| Q6 self-heal vs manual | Partial: entry-point changes self-sync; out-of-band + marker-current deletion need manual reindex | OBSERVED | Q6 |
| Write-path inventory W1–W14 | Enumerated with sync/async + citations | OBSERVED/INFERRED | inventory |
| §4.5.3 discrepancy | Mechanism exists in docker-prepare.sh; spec matches this commit | OBSERVED | §4.5.3 |
| Cleanup / read-only | Repository left unchanged — only this document differs; runtime state (DB, index, tasks, consume) is git-ignored and reset to baseline | OBSERVED | Cleanup |

**Legend:** D=daily, W=weekly, H=hourly, I=minutes-interval.

---

## Cleanup — repository left unchanged [OBSERVED]

All runtime side effects of the investigation were reverted; the only working-tree change is this document itself (which already existed at `HEAD` and was rewritten in place). Verification captured at the end of the session:

```console
# test documents restored / removed, correspondent removed, index reconciled to DB:
$ python3 manage.py shell -c "from documents.models import Document, Correspondent
print('db count=',Document.objects.count(),
      '| test-titled docs=',Document.objects.filter(title__regex=r'(UNIQUEZZZ|Q2A_SINGLE|RAWSQL|OLDTITLE|ORMNEW|ORMOLD|OPTOLD|OPTNEW|DELME)').count(),
      '| test correspondents=',Correspondent.objects.filter(name='Blitzy Bulk Corr').count())"
db count= 200 | test-titled docs= 0 | test correspondents= 0

# every test search term now misses:
$ for q in UNIQUEZZZ_q1demo Q2A_SINGLE_EDIT RAWSQL_q3 ORMNEW_q3b OPTNEW_q6 DELME_w2 "Blitzy Bulk Corr"; do
    c=$(curl -s -u admin:*** --get --data-urlencode "query=$q" "http://localhost:8000/api/documents/?fields=id" | python -c "import sys,json;print(json.load(sys.stdin)['count'])");
    echo "$q -> count=$c"; done
UNIQUEZZZ_q1demo -> count=0
Q2A_SINGLE_EDIT -> count=0
RAWSQL_q3 -> count=0
ORMNEW_q3b -> count=0
OPTNEW_q6 -> count=0
DELME_w2 -> count=0
Blitzy Bulk Corr -> count=0

# temp artifacts + startup-marker probe removed:
$ rm -f /usr/src/paperless ; rm -f data/.index_version ; rm -rf /tmp/inv
$ ls -A /usr/src/paperless 2>&1 ; ls data/.index_version 2>&1
ls: cannot access '/usr/src/paperless': No such file or directory
ls: cannot access 'data/.index_version': No such file or directory

# repository working tree: only this document differs from HEAD, no source modified:
$ git status --porcelain
 M blitzy/documentation/paperless-ngx_542221a38dff.md
$ git diff --name-only    # the only changed tracked file is this document -> no source file
blitzy/documentation/paperless-ngx_542221a38dff.md
```

The Whoosh index, the SQLite database, the Django-Q `Task`/`Schedule` tables, the consume directory, and all temporary observation scripts are runtime state (git-ignored) and were reset to a consistent baseline. The `git status --porcelain` shows exactly one changed path — this document — and `git diff --name-only` lists only that same path (no `src/`, `docker/`, or configuration file appears), confirming **no source file was modified**, honoring the read-only rule. (The ` M` status in the `git status --porcelain` capture above is the *modified-in-working-tree* state recorded during the investigation, before this document was committed; once it is committed, `git status --porcelain` reports a clean tree, with the sole change contained in this single file.)
