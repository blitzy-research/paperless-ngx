# Why the Paperless‑ngx documents list "feels haunted": a runtime root‑cause analysis

**Branch:** `paperless-ngx_542221a38dff` · **Commit (HEAD):** `542221a38dff06361e07976452f9aea24d210542`
**Method:** strictly read‑only investigation. The relevant code paths were **built and run first**; every behavioural claim below is backed by the exact command that produced it and its **unedited output**, plus a `file:line` reference into the source. Statements that were only read (not executed) are labelled **(inferred)**. No source file was modified; this Markdown file is the only artifact added. All temporary scripts and seeded data were removed afterwards (see §13).

---

## 1. The question

A user reports that the documents list _"can feel haunted during normal browsing"_:

- with **a couple of common filters enabled**, the **same document appears twice across neighbouring pages**, or
- a document **disappears for a page and then comes back**,

…while **nobody is editing anything** and **the visible sort order looks unchanged**. It reportedly gets **stranger when the viewer is not an all‑powerful admin** and visibility is _"shaped by sharing rules."_ Three named hypotheses were posed, each of which is answered explicitly and by name below:

- **H1** — Is the backend producing **duplicates** that get collapsed somewhere later?
- **H2** — Is **pagination happening before any de‑duplication**?
- **H3** — Is the **ordering quietly unstable** when multiple rows **tie** on the primary sort key?

…plus the **non‑admin / sharing‑rules** dimension.

The request was to _"watch what the API actually returns across consecutive page requests, and line that up with what the UI thinks pagination means, until the exact condition that destabilizes the list becomes clear."_ That is exactly what this document does.

---

## 2. Direct answer (TL;DR)

**The reproducible instability is H3: the list is ordered by a _non‑unique_ column with _no unique tiebreaker_, and each page is an independent `LIMIT/OFFSET` query. The ordering is therefore not a _total_ order — among rows that tie on the sort key, the row order is arbitrary and determined by the query plan / physical state, not by anything stable. When that arbitrary tie order changes between two page fetches (which happens during ordinary operation), a tied row can land on two adjacent pages (duplicate) or fall through the gap between them (skip), even though `count` and the visible sort never change.**

Answers by name:

| Hypothesis                                            | Verdict                                                                                                                                                                                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **H1** — backend produces duplicates collapsed later? | **Partially yes.** A many‑to‑many filter JOIN _does_ fan out duplicate `Document` rows, but they are collapsed by **`SELECT DISTINCT` inside the database** (`Document.objects.distinct()`), **not** in a later Python/UI step. |
| **H2** — pagination before de‑duplication?            | **No.** The emitted SQL is `SELECT DISTINCT … ORDER BY … LIMIT 25 OFFSET N`; `DISTINCT` is part of the query **before** the page slice. De‑duplication happens _before_ `LIMIT`, not after.                                     |
| **H3** — ordering unstable on ties?                   | **Yes — this is the root cause.** There is no unique tiebreaker anywhere (`Meta.ordering = ("-created",)`, the DB `ordering_fields` default, and the Whoosh sort map all lack a unique key).                                    |

**Honest negative result on the "non‑admin / sharing rules" premise (stated plainly, first):** at this commit there are **no object‑level permissions, no document ownership, and no sharing/ACL model at all**. Every documents request is gated only by `IsAuthenticated`; there is no `owner` field on `Document`, no `django-guardian` dependency, and the only per‑user‑scoped queryset in the whole viewset module is for `SavedView`, **not** `Document`. Observed directly: an admin and a non‑admin user receive **byte‑for‑byte identical** responses (identical SHA‑256, §8). So the glitch **cannot** arise from a permission‑scoped queryset here; it is **viewer‑independent** and attributable entirely to H3. The user's intuition that it "gets stranger for non‑admins" is a red herring at this commit — the same instability is present for everyone.

**Which path actually manifests it (nuance):** the instability is _latent_ in both list paths, but the two paths behave differently at runtime:

- **Database path** (structured filters, default browse): the `ORDER BY -created` has no tiebreaker, but PostgreSQL's `SELECT DISTINCT` on the full row _accidentally_ rescues stability in the simplest plan by folding the unique `id` into the sort key. That rescue is **fragile**: adding a many‑to‑many filter (a "common filter") exposes a second, equally valid `HashAggregate` plan whose `ORDER BY` sort keeps only `created` (no `id`). The planner can even choose _different_ plans for _different pages of the same list_, so — as proven in §6.3 — the real endpoint returns doc 14 on **both** page 3 and the neighbouring page 4 while doc 16 vanishes, all with `count` constant and no re‑sorting.
- **Whoosh full‑text path** (`?query=`): there is **no rescue at all**. Results tie on relevance score, the sort map has no `id` key, and `?ordering=id` is silently ignored. A single ordinary background re‑index (which fires on every document consumption) deterministically reshuffles the tied block, reproducing both the duplicate and the skip.

The remainder of this document proves each of these with captured output.

---

## 3. Environment & exact build/invocation commands (canonical)

The system was run in its default, canonical configuration inside the provided image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`), which bakes the repository at `/app` on Python 3.9 with a **PostgreSQL** backing store (the backend on which non‑deterministic tie ordering manifests). Docker‑in‑Docker containers: `paperless-app` (app), `paperless-db` (postgres:13), `paperless-broker` (redis:6.0) on network `paperless-net`; the app publishes `0.0.0.0:8000`.

**Interpreter and dependency versions**

```text
$ docker exec paperless-app python3 --version
Python 3.9.23

$ docker exec paperless-app python3 -c 'import django,rest_framework,django_filters,whoosh,psycopg2; \
    print("Django",django.get_version()); print("DRF",rest_framework.VERSION); \
    print("django-filter",django_filters.__version__); \
    print("Whoosh",".".join(map(str,whoosh.__version__))); print("psycopg2",psycopg2.__version__.split()[0])'
Django 4.0.4
DRF 3.13.1
django-filter 21.1
Whoosh 2.7.4
psycopg2 2.9.3
```

These match the pinned versions in `requirements.txt` (`django==4.0.4` [line 38], `djangorestframework==3.13.1` [line 39], `django-filter==21.1` [line 35], `whoosh==2.7.4` [line 111], `psycopg2==2.9.3` [line 69]) and the canonical Python from `Dockerfile:18` (`FROM python:3.9-slim-bullseye as main-app`).

**Database engine in use (canonical PostgreSQL)** — confirms the code switched to the PostgreSQL engine because `PAPERLESS_DBHOST` is set (`src/paperless/settings.py:304-318`):

```text
$ docker exec paperless-app python3 manage.py shell -c \
    'from django.db import connection; \
     print("ENGINE:",connection.settings_dict["ENGINE"]); \
     print("NAME:",connection.settings_dict["NAME"],"HOST:",connection.settings_dict["HOST"]); \
     c=connection.cursor(); c.execute("select version();"); print("PG:",c.fetchone()[0][:40])'
ENGINE: django.db.backends.postgresql_psycopg2
NAME: paperless HOST: paperless-db
PG: PostgreSQL 13.23 (Debian 13.23-1.pgdg13+
```

**Server invocation (the real entry point).** The API is served by the standard dev server; every headline observation below is a real `GET /api/documents/` request routed to `UnifiedSearchViewSet` (`src/paperless/urls.py:32`). The running process was verified:

```text
# canonical run command (from the image setup):
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py migrate && \
    python3 manage.py runserver 0.0.0.0:8000 --noreload --insecure'

# migrations already applied:
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py migrate --check'
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  No migrations to apply.

# server process actually running (exact command + unedited output):
$ docker top paperless-app -eo pid,cmd
PID                 CMD
12115               sleep infinity
23775               python3 manage.py runserver 0.0.0.0:8000 --noreload --insecure
```

The API was driven from the host with `curl` against the mapped port `http://localhost:8000` (the `paperless-app` image ships without `curl`, so requests were issued from the host, which is equivalent — the port is published). Authentication used the two throwaway local users provided by the canonical image: `admin` (superuser) and `viewer` (non‑admin), via ordinary Basic auth (`-u admin:admin123` / `-u viewer:viewer123`); every command in this document uses Basic auth so each is runnable as-is. DRF enables Basic, Session, and Token authentication (`src/paperless/settings.py:117-121`).

### 3.1 Seeding the tie condition (the crux)

The defect is invisible unless duplicate sort‑key values fall on a page boundary. A throwaway script (`/tmp/seed_haunted.py`, run via `manage.py shell`, later deleted) created **12** `Document` rows that **all share one identical `created` timestamp** — because `created` is `DateTimeField(default=timezone.now, db_index=True)` and is **not unique** (`src/documents/models.py:152`), and `Document.Meta.ordering = ("-created",)` sorts by exactly this non‑unique column (`src/documents/models.py:207-208`). Each row also received one normal tag plus **two inbox tags** (to exercise the M2M fan‑out) and **identical `content`** (to force tied Whoosh relevance scores).

```text
$ docker exec -i paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/seed_haunted.py
SEED_TS 2026-07-08T04:59:34.321734+00:00
SEED_IDS [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
TAG_A_ID 1
INBOX1_ID 2 INBOX2_ID 3
DOC_COUNT 12
DISTINCT_CREATED_COUNT 1
```

`DISTINCT_CREATED_COUNT 1` confirms all 12 rows tie on `created`. The Whoosh index was then built:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py document_index reindex'
100%|██████████| 12/12 [00:00<00:00, 582.34it/s]
```

Both list paths return all 12 seeded rows through the real endpoint:

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?page_size=1"          | python3 -c "import sys,json;print('count=',json.load(sys.stdin)['count'])"
count= 12
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page_size=1" | python3 -c "import sys,json;print('count=',json.load(sys.stdin)['count'])"
count= 12
```

All page walks below use `page_size=3` so a page boundary (between positions 3/4, 6/7, 9/10) falls **inside** the 12‑row tied block. `?fields=id` slims the JSON via the serializer's dynamic‑fields projection (`src/documents/serialisers.py:22-40,201`); it does **not** change the SQL, which always selects the full row (shown in §4/§5).

---

## 4. H1 — Is the backend producing duplicates that get collapsed later?

**Verdict: Partially yes — a many‑to‑many filter JOIN fans out duplicate `Document` rows, but they are collapsed by `SELECT DISTINCT` _inside the database_ (`Document.objects.distinct()`, `src/documents/views.py:198-199`), not in a later Python/UI step.**

Every seeded document carries two inbox tags (tag ids 2 and 3, both `is_inbox_tag=True`). The `is_in_inbox=true` filter joins `Document → tags` and keeps rows whose tag has `is_inbox_tag=True` (`InboxFilter.filter` → `qs.filter(tags__is_inbox_tag=True)`, `src/documents/filters.py:63-70`) — **with no `.distinct()` of its own**. So the raw JOIN yields two rows per document. Corroborating shell (labelled corroboration — the headline is the HTTP behaviour that follows):

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
raw  = Document.objects.filter(tags__is_inbox_tag=True)
print(\"raw JOIN row count   :\", raw.count())
print(\"after .distinct()    :\", raw.distinct().count())
print(\"raw ids (with dups)  :\", sorted(raw.values_list(\"id\", flat=True)))
"'
raw JOIN row count   : 24
after .distinct()    : 12
raw ids (with dups)  : [6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15, 16, 16, 17, 17]
```

The JOIN really does fan out **24** rows (each of the 12 documents twice — once per inbox tag). Now the **real endpoint** — the headline evidence — shows the response contains **no duplicate ids**, i.e. the fan‑out was already collapsed before serialization:

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&page_size=100&fields=id" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); ids=[r['id'] for r in d['results']]; \
      print('count =', d['count']); print('n_results =', len(ids)); \
      print('has_duplicates =', len(ids)!=len(set(ids))); print('ids =', sorted(ids))"
count = 12
n_results = 12
has_duplicates = False
ids = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
```

So the duplicates the JOIN produces never reach the client — they are removed by `SELECT DISTINCT` in the database (proven by the emitted SQL in §5). The de‑duplication is **not** something the UI or a Python post‑pass does; it is intrinsic to the queryset that `DocumentViewSet.get_queryset` returns:

```text
$ sed -n '198,199p' src/documents/views.py
    def get_queryset(self):
        return Document.objects.distinct()
```

**Sibling M2M variants (exhaustive, not representative):** the fan‑out and the collapse depend on which filter branch runs.

```text
# tags__id__all (AND-of-tags): loops qs.filter(tags__id=…) per id — but single tag id here → no fan-out
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?tags__id__all=1&page_size=100&fields=id" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('tags__id__all=1 count',d['count'])"
tags__id__all=1 count 12

# tags__id__in (OR-of-tags): the in_list branch itself appends .distinct() at filter level (filters.py:52)
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?tags__id__in=2,3&page_size=100&fields=id" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('tags__id__in=2,3 count',d['count'])"
tags__id__in=2,3 count 12
```

Cause → effect, by branch:

- `InboxFilter.filter` returns `qs.filter(tags__is_inbox_tag=True)` with **no** `distinct()` (`src/documents/filters.py:63-70`) → relies entirely on the view‑level `distinct()`.
- The non‑`in_list` `TagsFilter` branch loops `qs.filter(tags__id=tag_id)` per id with **no per‑filter distinct** (`src/documents/filters.py:53-58`).
- Only the `in_list` branch adds its own `.distinct()` (`src/documents/filters.py:52`).

In all cases the client sees a de‑duplicated list of 12 — the duplicates are collapsed **in the DB**, answering H1 as _partially yes, but not "collapsed later."_

---

## 5. H2 — Is pagination happening before any de‑duplication?

**Verdict: No. The emitted SQL is `SELECT DISTINCT … ORDER BY … LIMIT 3 OFFSET N` — `DISTINCT` is part of the query and is applied _before_ the `LIMIT` slice.** De‑duplication precedes pagination, not the other way around.

The page slice is supplied by `StandardPagination` (`page_size=25` default, overridable via `page_size` query param, `max_page_size=100000`):

```text
$ sed -n '8,11p' src/paperless/views.py
class StandardPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = "page_size"
    max_page_size = 100000
```

Corroborating SQL capture (labelled corroboration): building the same queryset the viewset builds (`Document.objects.distinct()` filtered/ordered by the same backends, then sliced the way DRF slices page 2 at `page_size=3`) and printing `str(queryset.query)`:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from documents.models import Document
qs = Document.objects.distinct().order_by(\"-created\")[3:6]   # page 2, page_size=3 -> OFFSET 3 LIMIT 3
print(str(qs.query))
"'
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 3 OFFSET 3
```

(The output above is a single physical line as printed by `str(qs.query)`; it is shown verbatim. The row has 15 columns — note there is no `storage_path_id` at this commit.) The clause order is unambiguous: `SELECT DISTINCT … FROM … ORDER BY … LIMIT 3 OFFSET 3`. The `DISTINCT` is evaluated as part of producing the ordered result set; the `LIMIT/OFFSET` is applied to the already‑distinct, already‑ordered stream.

This is confirmed on the **real request path** — routing a `page=2&page_size=3` GET through the `UnifiedSearchViewSet` and capturing the actually‑executed SQL with `CaptureQueriesContext`:

```text
$ cat /tmp/h2_capture.py
from django.test import Client
from django.test.utils import CaptureQueriesContext
from django.db import connection
from django.contrib.auth.models import User
c = Client()
c.force_login(User.objects.get(username="admin"))
with CaptureQueriesContext(connection) as ctx:
    r = c.get("/api/documents/?page=2&page_size=3&fields=id")
print("HTTP", r.status_code)
for q in ctx.captured_queries:
    s = q["sql"]
    if "documents_document" in s and s.strip().upper().startswith("SELECT DISTINCT"):
        print(s)
        break

$ docker exec -i paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/h2_capture.py
HTTP 200
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 3 OFFSET 3
```

The single page query executed by the live viewset is exactly `SELECT DISTINCT … ORDER BY "created" DESC LIMIT 3 OFFSET 3` — `DISTINCT` in the `SELECT`, `LIMIT/OFFSET` last.

Cause → effect: because `.distinct()` is baked into the queryset (`src/documents/views.py:198-199`) **before** DRF's paginator slices it (`src/paperless/views.py:8-11`), pagination is applied to an already‑de‑duplicated stream. So H2's proposed mechanism — "we paginate a list that still has duplicates, then de‑dup per page" — is **not** what happens. The duplicate/skip symptom therefore cannot be explained by H2; it comes from H3 (§6).

---

## 6. H3 — Is the ordering quietly unstable when rows tie on the primary sort key? (ROOT CAUSE)

**Verdict: Yes. This is the root cause.** The list is ordered by `created` — a non‑unique column — with **no unique tiebreaker anywhere**, and each page is an independent `LIMIT/OFFSET` query. That is not a _total_ order: the relative order of rows that tie on `created` is unspecified. When that unspecified order changes between two page fetches, tied rows duplicate across a boundary or fall through the gap.

The three ingredients, each grounded:

```text
$ sed -n '152p;207,208p' src/documents/models.py
    created = models.DateTimeField(_("created"), default=timezone.now, db_index=True)
    class Meta:
        ordering = ("-created",)
```

- Primary sort key `created` is **non‑unique** (`src/documents/models.py:152`).
- Default order is `-created` with no secondary key (`src/documents/models.py:207-208`); DRF's `OrderingFilter` falls back to this `Meta.ordering` when no `?ordering=` param is sent, so **the default browse is already tie‑exposed** — the user need not choose any special sort.
- The page slice is per‑page `LIMIT/OFFSET` (`src/paperless/views.py:8-11`), so every page is a _separate_ query — a separate evaluation of that non‑total order.
- The only unique field among the orderable ones is `id`; `archive_serial_number` is `unique=True` **but nullable** (`src/documents/models.py:196-205`), so it is not a total order over all rows. `ordering_fields` (`src/documents/views.py:187-196`) offers no composite/tiebreaker default.

The crucial consequence: **whether the result is stable depends entirely on whether some unique column happens to end up in the effective sort — which is decided by the PostgreSQL query plan, not by the code.** The next three subsections show (6.1) the unfiltered path is _accidentally_ stabilized by the plan, (6.2) the same query has a second, equally valid plan that is _not_ stabilized, and (6.3) the two plans can serve _different pages of the same list_, producing the exact duplicate/skip symptom in a single quiescent snapshot.

### 6.1 The unfiltered path is _accidentally_ stabilized by the plan

On a quiescent table, repeatedly paging the unfiltered list (no `?ordering=`, `page_size=3`) is stable, and — perhaps surprisingly — comes back in `id`‑ascending order within the tie:

```text
$ for run in 1 2 3; do echo -n "run$run: "; \
    for p in 1 2 3 4; do \
      curl -s -u admin:admin123 "http://localhost:8000/api/documents/?page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print([r['id'] for r in json.load(sys.stdin)['results']],end=' ')"; \
    done; echo; done
run1: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
run2: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
run3: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
```

Why is a "non‑total order" stable here? Because `SELECT DISTINCT` over the **full row** makes PostgreSQL sort by _all_ selected columns to find duplicates, and that column list includes the unique `id`. `EXPLAIN (ANALYZE)` of the exact executed query shows `id` folded into the sort key as the **first tiebreaker** after `created` (this is the actual executed plan, not an estimate):

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.db import connection
from documents.models import Document
qs = Document.objects.distinct().order_by(\"-created\")
sql, params = qs.query.sql_with_params()
c = connection.cursor()
c.execute(sql, params); print(\"actual unfiltered id order:\", [r[0] for r in c.fetchall()])
c.execute(\"EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF, SUMMARY OFF) \"+sql, params)
print(chr(10).join(r[0] for r in c.fetchall()))
"'
actual unfiltered id order: [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
Unique (actual rows=12 loops=1)
  ->  Sort (actual rows=12 loops=1)
        Sort Key: created DESC, id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, modified, storage_type, added, filename, archive_filename, archive_serial_number
        Sort Method: quicksort  Memory: 28kB
        ->  Seq Scan on documents_document (actual rows=12 loops=1)
```

The `Sort Key` begins `created DESC, id, …`: the unique `id` is an _accidental_ tiebreaker, so the order is total and therefore stable. This is why the bug can lie dormant on a simple browse for a long time — **exactly the user's "worked fine, then haunted" experience.** **This rescue is not written anywhere in the code — it is an artifact of the plan** that PostgreSQL chose to compute `DISTINCT`, and it disappears the moment the plan changes (§6.2, §6.3) or the path changes (§9).

### 6.2 The same filtered query has a second valid plan with **no** `id` — and a different order

Enabling a "common filter" (the many‑to‑many inbox filter) adds two JOINs, and PostgreSQL then has two equally reasonable ways to compute `DISTINCT`: fold everything into one sort (`Sort → Unique`, which keeps `id`), or hash‑group to de‑duplicate and sort only by the `ORDER BY` afterwards (`HashAggregate → Sort`, which keeps only `created`). These two plans return the **same 12 rows** (`count` constant) in **different orders**. Forcing each plan on the _exact_ SQL the endpoint emits (verified to contain the two `INNER JOIN`s of the real filter path):

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.db import connection
from documents.models import Document
from documents.filters import DocumentFilterSet
fs = DocumentFilterSet({\"is_in_inbox\": \"true\"}, queryset=Document.objects.distinct())
qs = fs.qs.order_by(\"-created\")
sql, params = qs.query.sql_with_params()
c = connection.cursor()
print(\"SQL matches API path (two INNER JOINs):\", sql.count(\"INNER JOIN\")==2)
c.execute(\"SET enable_hashagg = off;\")
c.execute(sql, params); print(\"enable_hashagg=OFF order:\", [r[0] for r in c.fetchall()])
c.execute(\"EXPLAIN (COSTS OFF) \"+sql, params); p=[r[0].strip() for r in c.fetchall()]
print(\"  plan:\", \" / \".join(p[:3]))
c.execute(\"SET enable_hashagg = on;\")
c.execute(sql, params); print(\"enable_hashagg=ON  order:\", [r[0] for r in c.fetchall()])
c.execute(\"EXPLAIN (COSTS OFF) \"+sql, params); p=[r[0].strip() for r in c.fetchall()]
print(\"  plan:\", \" / \".join(p[:3]))
"'
SQL matches API path (two INNER JOINs): True
enable_hashagg=OFF order: [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
  plan: Unique / ->  Sort / Sort Key: documents_document.created DESC, documents_document.id, documents_document.correspondent_id, documents_document.title, documents_document.document_type_id, documents_document.content, documents_document.mime_type, documents_document.checksum, documents_document.archive_checksum, documents_document.modified, documents_document.storage_type, documents_document.added, documents_document.filename, documents_document.archive_filename, documents_document.archive_serial_number
enable_hashagg=ON  order: [12, 11, 10, 9, 7, 13, 8, 6, 16, 17, 15, 14]
  plan: Sort / Sort Key: documents_document.created DESC / ->  HashAggregate
```

Cause → effect, read directly off the two plans:

- `enable_hashagg=OFF` → `Unique → Sort` with `Sort Key: created DESC, id, …` → the unique `id` makes it a **total** order → `[6,7,8,…,17]`.
- `enable_hashagg=ON` → `Sort (Sort Key: created DESC only) → HashAggregate` → the `HashAggregate` does the de‑duplication and the top `Sort` orders by `created` **only**, so the 12 tied rows emerge in hash‑bucket order → `[12,11,10,9,7,13,8,6,16,17,15,14]`.

Both are correct answers to the same SQL; which one runs is a **cost decision** that shifts with table statistics (autovacuum/`ANALYZE`), row counts, PostgreSQL version, and config. The default plan on this data is the `HashAggregate` one — so the real endpoint returns the scrambled order:

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&page_size=100&fields=id" \
  | python3 -c "import sys,json;print([r['id'] for r in json.load(sys.stdin)['results']])"
[12, 11, 10, 9, 7, 13, 8, 6, 16, 17, 15, 14]
```

**Reproducibility note — this scrambled order is one representative single‑snapshot capture, not a fixed constant of the bug.** The 12‑element permutation above is the hash‑bucket order that PostgreSQL's `HashAggregate` happened to emit for _this_ seed under _this_ plan. `HashAggregate` de‑duplicates by hashing the **full row tuple**, which includes per‑row‑unique bytes this analysis never pins (`checksum`, `title`, `content`, `filename`); an independent re‑seed with the _same_ tie structure but _different_ row bytes therefore drops the rows into _different_ hash buckets and returns a _different_ order — and which of the two near‑cost‑tied plans the planner picks is itself the cost decision noted above (it shifts with table statistics, row counts, PostgreSQL version, and config). An independent re‑seed of the identical structure (12 rows sharing one `created`, same IDs 6–17, fresh row bytes) shows a _different_ order with `count` still 12:

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&page_size=100&fields=id" | python3 -c "import sys,json;d=json.load(sys.stdin);print('count',d['count']);print('order',[r['id'] for r in d['results']])"
count 12
order [11, 15, 16, 17, 8, 7, 13, 10, 12, 9, 14, 6]
```

What _is_ exactly reproducible — and what actually answers H3 — is independent of the particular sequence: the verdict (root cause), the **mechanism** (the `HashAggregate` plan drops the `id` tiebreaker, leaving a non‑total order that per‑page `LIMIT/OFFSET` then slices inconsistently), and the constant `count`. Only the _specific_ scrambled order is snapshot‑specific.

### 6.3 The haunting itself: different pages of one list, different plans (the smoking gun)

Because every page is an **independent** `LIMIT 3 OFFSET N` query (§6, `src/paperless/views.py:8-11`), and because the two plans of §6.2 are near‑cost‑tied, the planner can pick a **different plan for different offsets of the same list**. Paging the real endpoint through `?is_in_inbox=true` at `page_size=3` — repeated 3× to show it is not a fluke — produces this, stably:

```text
$ for run in 1 2 3; do echo -n "run$run: "; \
    for p in 1 2 3 4; do \
      curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; \
    done; echo; done
run1: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [17, 15, 14]
run2: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [17, 15, 14]
run3: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [17, 15, 14]
```

Read the union of the four pages: **doc 14 appears on page 3 _and again_ on the very next page, page 4 — the same document on two neighbouring pages; and doc 16 never appears at all (a skip)** — while `count` is 12 the whole time and no `?ordering=` was ever sent. This is _precisely_ the user's report: "the same document twice across neighbouring pages" (doc 14 on **adjacent** pages 3 and 4) and "a document disappears for a page" (doc 16 is gone from the browse), in a single, unchanging snapshot of the data. `EXPLAIN`ing each page's exact query shows why — the pages are served by two different plans:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.db import connection
from documents.models import Document
from documents.filters import DocumentFilterSet
c = connection.cursor()
for pg in range(4):
    off = pg*3
    fs = DocumentFilterSet({\"is_in_inbox\": \"true\"}, queryset=Document.objects.distinct())
    qs = fs.qs.order_by(\"-created\")[off:off+3]
    sql, params = qs.query.sql_with_params()
    c.execute(sql, params); ids = [r[0] for r in c.fetchall()]
    c.execute(\"EXPLAIN (COSTS OFF) \"+sql, params); plan = [r[0].strip() for r in c.fetchall()]
    via = \"HashAggregate\" if any(\"HashAggregate\" in x for x in plan) else \"Sort+Unique\"
    sk = [x for x in plan if x.startswith(\"Sort Key\")]
    print(\"page\", pg+1, \"(OFFSET %d LIMIT 3)\"%off, \"-> ids\", ids, \"| DISTINCT via\", via)
    print(\"      \", (sk[0][:70] if sk else \"(no explicit Sort node)\"))
"'
page 1 (OFFSET 0 LIMIT 3) -> ids [6, 7, 8] | DISTINCT via Sort+Unique
       Sort Key: documents_document.created DESC, documents_document.id, docu
page 2 (OFFSET 3 LIMIT 3) -> ids [9, 10, 11] | DISTINCT via Sort+Unique
       Sort Key: documents_document.created DESC, documents_document.id, docu
page 3 (OFFSET 6 LIMIT 3) -> ids [12, 13, 14] | DISTINCT via Sort+Unique
       Sort Key: documents_document.created DESC, documents_document.id, docu
page 4 (OFFSET 9 LIMIT 3) -> ids [17, 15, 14] | DISTINCT via HashAggregate
       Sort Key: documents_document.created DESC
```

Cause → effect: pages 1–3 (`OFFSET 0/3/6`) are planned as `Sort+Unique`, whose `Sort Key` includes the unique `id`, so they slice the `id`‑ordered total order `[6,7,8 | 9,10,11 | 12,13,14 | …]`. Page 4 (`OFFSET 9`) is planned as `HashAggregate`, whose top `Sort Key` is `created DESC` **only**, so it slices the _hash_ order `[12,11,10,9,7,13,8,6,16,17,15,14]` — its tail (positions 9–11) is `[17,15,14]`. The two plans encode **two different orderings**, and the paginator stitches slices from both into one browse. The result is that doc `14` — which the `id`‑ordered total order placed at the tail of page 3 (`[12,13,14]`) — reappears in the tail of the hash order on the _immediately following_ page 4, so the user meets the same document twice on two neighbouring pages; meanwhile doc `16` (which the total order would place on page 4 as part of `[15,16,17]`) is emitted by neither plan and simply vanishes from the browse. No row was edited; the sort the user sees ("newest first") never changed; only the invisible tie order differed between two of the page queries — the definition of a "haunted" list.

**Reproducibility note — the _specific_ duplicated/skipped IDs and the "neighbouring pages 3 & 4" adjacency are snapshot‑specific; the duplicate‑and‑skip _phenomenon_ is the invariant.** Which document duplicates, which is skipped, and whether the duplicate lands on adjacent or non‑adjacent pages all follow from the hash‑bucket order of §6.2, which (as shown there) varies per seed and per plan choice. On the independent re‑seed of §6.2, the identical forward browse instead duplicates rows on **non‑adjacent** pages and drops three different rows — while `count` stays 12 and no `?ordering=` is sent:

```text
$ for p in 1 2 3 4; do curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&page=$p&page_size=3&fields=id" | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; done; echo
p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [9, 14, 6]
```

Here the union duplicates doc 6 on pages **1 and 4** and doc 9 on pages **2 and 4** — **non‑adjacent** pages, not only neighbouring ones — plus doc 14 on pages 3 and 4, while docs 15, 16, 17 are skipped. So the headline walk above (doc 14 on neighbouring pages 3 & 4, doc 16 skipped) is one representative capture; a duplicate can equally surface on non‑adjacent pages, and the count of duplicated/skipped rows varies with the hash order. What is invariant — and what matches the user's report — is that whenever the forward browse crosses a tied boundary served by two different plans, at least one tied row is duplicated across a page boundary and at least one is skipped, with `count` constant and the visible sort unchanged.

### 6.4 Summary of H3

- The code provides **no unique tiebreaker** at any layer (`Meta.ordering`, the DB `ordering_fields` default, the Whoosh sort map).
- Stability is therefore **left to chance**: it depends on whether the query plan happens to include a unique column in its effective sort. On the unfiltered path the `SELECT DISTINCT`‑over‑all‑columns plan _incidentally_ includes `id` (§6.1) → **latent** bug.
- Adding a common M2M filter exposes a second, equally valid `HashAggregate` plan whose `ORDER BY` sort has **no** `id` (§6.2); the planner can even pick different plans for different pages of the same list (§6.3) → **manifest** duplicates and skips in one snapshot, `count` constant.
- The Whoosh path (§9) has **no** accidental rescue at all and manifests the same symptom from a single ordinary re‑index.

---

## 7. Control — page by a unique key (`ordering=id`)

This is a **labelled control**, not the headline. To confirm the instability is _tie‑specific_ (and not some other pagination defect), the identical page walk was repeated with `ordering=id` — a unique, total order. Pages are stable across repeated runs, with no duplicates and no skips, even with the M2M filter that reshuffles the default order:

```text
$ for run in 1 2 3; do echo -n "run$run: "; \
    for p in 1 2 3 4; do \
      curl -s -u admin:admin123 "http://localhost:8000/api/documents/?is_in_inbox=true&ordering=id&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print([r['id'] for r in json.load(sys.stdin)['results']],end=' ')"; \
    done; echo; done
run1: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
run2: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
run3: [6, 7, 8] [9, 10, 11] [12, 13, 14] [15, 16, 17]
```

Unlike §6.3, every page's plan now leads its `Sort Key` with the unique `id`, regardless of which offset or DISTINCT strategy is used:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py shell -c "
from django.db import connection
from documents.models import Document
from documents.filters import DocumentFilterSet
c=connection.cursor()
for off in (0,9):
    fs=DocumentFilterSet({\"is_in_inbox\":\"true\"},queryset=Document.objects.distinct())
    qs=fs.qs.order_by(\"id\")[off:off+3]
    sql,params=qs.query.sql_with_params()
    c.execute(sql,params); ids=[r[0] for r in c.fetchall()]
    c.execute(\"EXPLAIN (COSTS OFF) \"+sql,params); plan=[r[0].strip() for r in c.fetchall()]
    sk=[x for x in plan if x.startswith(\"Sort Key\")]
    print(\"OFFSET\",off,\"ids\",ids,\"| Sort Key:\", (sk[0][9:40] if sk else \"n/a\"))
"'
OFFSET 0 ids [6, 7, 8] | Sort Key:  documents_document.id, documen
OFFSET 9 ids [15, 16, 17] | Sort Key:  documents_document.id
```

Cause → effect: `id` is unique, so `ORDER BY id` is a **total** order; there are no ties to resolve arbitrarily, so every page boundary is deterministic and the union across pages is exactly the 12 rows with no repeats — even when one page uses `Sort+Unique` and another uses `HashAggregate`, because the leading key is `id` either way. `id` is the only always‑present unique member of `ordering_fields` (`src/documents/views.py:187-196`; `archive_serial_number` is unique but nullable, `src/documents/models.py:196-205`). This isolates the root cause to the _absence of a unique tiebreaker_ under the non‑unique default sort — i.e. H3.

---

## 8. Admin vs non‑admin (the "sharing rules" dimension)

**Verdict: there is no permission scoping at this commit; the two viewers get byte‑for‑byte identical results, so the instability is viewer‑independent.** This is stated first and plainly because the user's premise ("gets stranger for non‑admins, shaped by sharing rules") does not hold against this code.

Running the identical default browse as the superuser `admin` and the non‑admin `viewer`:

```text
$ for u in admin:admin123 viewer:viewer123; do echo -n "$u -> "; \
    curl -s -u "$u" "http://localhost:8000/api/documents/?page_size=100&fields=id" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('count',d['count'],'ids',sorted(r['id'] for r in d['results']))"; \
  done
admin:admin123 -> count 12 ids [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
viewer:viewer123 -> count 12 ids [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
```

And on the Whoosh path, identical again:

```text
$ for u in admin:admin123 viewer:viewer123; do echo -n "$u -> "; \
    curl -s -u "$u" "http://localhost:8000/api/documents/?query=haunted&page_size=100&fields=id" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('count',d['count'],'ids',sorted(r['id'] for r in d['results']))"; \
  done
admin:admin123 -> count 12 ids [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
viewer:viewer123 -> count 12 ids [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17]
```

The result _sets_ being equal is necessary but not sufficient — the user's symptom is about the **unstable page walk**, so the identical `page_size=3` walk was run as both users on both paths. Every page is byte-identical between the two identities, so the duplicate/skip distribution is identical too — the haunting is in no way modulated by who is looking. On the **DB path** (the unstable H3 walk of §6.3), both users see doc 14 on neighbouring pages 3 and 4 and doc 16 skipped:

```text
$ for u in admin:admin123 viewer:viewer123; do echo -n "$u -> "; \
    for p in 1 2 3 4; do \
      curl -s -u "$u" "http://localhost:8000/api/documents/?is_in_inbox=true&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; \
    done; echo; done
admin:admin123 -> p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [17, 15, 14]
viewer:viewer123 -> p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [17, 15, 14]
```

On the **Whoosh path**, from a quiescent index, the per-page walk is likewise identical for both identities:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py document_index reindex --no-progress-bar'   # quiescent baseline; prints nothing
$ for u in admin:admin123 viewer:viewer123; do echo -n "$u -> "; \
    for p in 1 2 3 4; do \
      curl -s -u "$u" "http://localhost:8000/api/documents/?query=haunted&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; \
    done; echo; done
admin:admin123 -> p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [15, 16, 17]
viewer:viewer123 -> p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [15, 16, 17]
```

To prove _byte-for-byte_ equality (not just the same ids), the full response bodies were hashed. Both requests use ordinary Basic auth with the two local accounts, and a **total** order (`ordering=id`) so the body is deterministic and differs only by identity; the SHA-256 of the full JSON body is identical:

```text
$ A=$(curl -s -u admin:admin123  "http://localhost:8000/api/documents/?ordering=id&page_size=100" | sha256sum | cut -d' ' -f1)
$ B=$(curl -s -u viewer:viewer123 "http://localhost:8000/api/documents/?ordering=id&page_size=100" | sha256sum | cut -d' ' -f1)
$ echo "admin  $A"; echo "viewer $B"; [ "$A" = "$B" ] && echo "IDENTICAL" || echo "DIFFERENT"
admin  591e5eeeb75088f08b3cc01f0889ba41a479e548b293b649a8ca8c3326437a12
viewer 591e5eeeb75088f08b3cc01f0889ba41a479e548b293b649a8ca8c3326437a12
IDENTICAL
```

Cause → effect, grounded in the code:

- The documents endpoint's only gate is `permission_classes = (IsAuthenticated,)` — any logged‑in user, admin or not, sees everything (`src/documents/views.py:183`).
- `get_queryset` returns `Document.objects.distinct()` with **no** `filter(owner=…)` or permission scoping (`src/documents/views.py:198-199`).
- There is **no `owner` field** on the `Document` model (the field list runs `src/documents/models.py:88-205` with no ownership/ACL field), and **no `django-guardian`** dependency in `requirements.txt`.
- The **only** per‑user‑scoped queryset anywhere in the viewset module is `SavedViewViewSet.get_queryset → SavedView.objects.filter(user=user)` (`src/documents/views.py:461-463`) — that scopes _saved filter presets_, not documents.

So the "non‑admin" angle is a **negative result**: no sharing model exists to shape visibility. The reproducible haunting is the same for everyone and is fully explained by H3. **(inferred, then confirmed):** reading the model and requirements suggested no ownership/ACL; the identical‑hash observation above confirms it at runtime.

**Reproducibility note — the literal SHA‑256 digest is a single‑snapshot value; the _equality_ is the reproducible, load‑bearing fact.** The digest `591e5eee…437a12` is taken over the entire JSON body, which embeds seed‑specific bytes (titles, checksums, `created`/`added`/`modified` timestamps, ids), so it necessarily changes on any re‑seed. What is invariant — and what actually settles the non‑admin question — is that the admin body and the viewer body hash to the **same** value as each other. An independent re‑seed confirms both halves at once — a _different_ digest, still **identical** between the two identities:

```text
$ A=$(curl -s -u admin:admin123  "http://localhost:8000/api/documents/?ordering=id&page_size=100" | sha256sum | cut -d' ' -f1)
$ B=$(curl -s -u viewer:viewer123 "http://localhost:8000/api/documents/?ordering=id&page_size=100" | sha256sum | cut -d' ' -f1)
$ echo "admin  $A"; echo "viewer $B"; [ "$A" = "$B" ] && echo "IDENTICAL" || echo "DIFFERENT"
admin  b3e2a7f73ee2d282a54401a21035e48e223037b5b044eb8865639b2a84a13b5e
viewer b3e2a7f73ee2d282a54401a21035e48e223037b5b044eb8865639b2a84a13b5e
IDENTICAL
```

The digest differs from the one above (`b3e2a7f7…` vs `591e5eee…`) because the body bytes differ per seed; the admin==viewer equality — the fact that proves viewer‑independence — holds on every re‑seed.

---

## 9. Whoosh full‑text path (`?query=`) — same instability class, no accidental rescue

The user's "a couple of common filters" can equally be a **text query**, which routes to the Whoosh full‑text path instead of the DB queryset. `UnifiedSearchViewSet._is_search_request()` fires when `query` (or `more_like_id`) is present (`src/documents/views.py:388-392`) and `filter_queryset` swaps in a `DelayedFullTextQuery` (`src/documents/views.py:394-411`), paged by `list()` opening an index searcher (`src/documents/views.py:413-426`). This path has **no accidental `id` rescue at all**, so it manifests the haunting from a single ordinary re‑index.

**All results tie on relevance** (identical seeded content → identical score `1.0`):

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page_size=100" \
  | python3 -c "import sys,json;d=json.load(sys.stdin); \
      print('count',d['count']); print('scores',sorted({round(r.get('__search_hit__',{}).get('score',1.0),3) for r in d['results']}))"
count 12
scores [1.0]
```

**No unique tiebreaker is even available on this path.** `DelayedQuery._get_query_sortedby` returns `(None, False)` when no ordering is given (→ relevance/docnum order), and its `sort_fields_map` has **no `id` key**, so even an explicit `?ordering=id` cannot produce a total order:

```text
$ sed -n '165,190p' src/documents/index.py
    def _get_query_sortedby(self):
        if "ordering" not in self.query_params:
            return None, False

        field: str = self.query_params["ordering"]

        sort_fields_map = {
            "created": "created",
            "modified": "modified",
            "added": "added",
            "title": "title",
            "correspondent__name": "correspondent",
            "document_type__name": "type",
            "archive_serial_number": "asn",
        }

        if field.startswith("-"):
            field = field[1:]
            reverse = True
        else:
            reverse = False

        if field not in sort_fields_map:
            return None, False
        else:
            return sort_fields_map[field], reverse
```

There is no `"id"` entry in that map. Each page is an independent `searcher.search_page(pagenum, pagelen, sortedby, reverse)` call (`src/documents/index.py:203-221`), so equal‑score ties order by internal Whoosh **docnum**, which changes whenever a document is re‑indexed.

**Reproducing the haunting on the real search endpoint.** A normal document consumption re-indexes the touched document via `add_to_index` (`src/documents/signals/handlers.py:428-431` → `index.add_or_update_document`, `src/documents/index.py:118-120`), wired to `document_consumption_finished` (`src/documents/apps.py:27`). Whoosh's `update_document` is delete+append, so the touched doc gets a **new, higher docnum**, moving it to the **end** of the equal-score tie block. The throwaway re-index helper makes exactly the call the signal makes:

```text
$ cat /tmp/touch_index.py
import os
from documents.models import Document
from documents import index
did = int(os.environ["TOUCH_ID"])
d = Document.objects.get(id=did)
# the EXACT call add_to_index makes on document_consumption_finished
# (signals/handlers.py:428-431 -> index.add_or_update_document -> index.py:118-120)
index.add_or_update_document(d)
print("TOUCHED", d.id)
```

**(a) On a quiescent index the same request set is perfectly stable — 0 duplicates, 0 skips over repeated runs.** Rebuild the index, then walk pages 1→4 three times, unchanged:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py document_index reindex --no-progress-bar'   # rebuild index; prints nothing
$ for run in 1 2 3; do echo -n "run$run: "; \
    for p in 1 2 3 4; do \
      curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; \
    done; echo; done
run1: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [15, 16, 17]
run2: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [15, 16, 17]
run3: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 14]  p4 [15, 16, 17]
```

So the run-to-run inconsistency on this path is **not** produced by re-issuing the same request against an unchanged index (distribution there: **0 duplicates, 0 skips**); it is produced by the **ordinary background re-index** that fires on every consume. Each such event is itself deterministic — it moves exactly the touched doc to the tie-block end — so the resulting duplicate/skip is reproducible, as the next two demonstrations show (each is repeated to confirm the post-event state is itself stable).

**(b) "The same document twice across neighbouring pages" — from ONE ordinary re-index.** The user views page 3, a background consume re-indexes one of the documents currently on page 3 (doc 14), then the user clicks _Next_ to page 4:

```text
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=3&page_size=3&fields=id" \
  | python3 -c "import sys,json;print('page3 =',[r['id'] for r in json.load(sys.stdin)['results']])"
page3 = [12, 13, 14]

$ docker exec -i -e TOUCH_ID=14 paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/touch_index.py
TOUCHED 14

$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=4&page_size=3&fields=id" \
  | python3 -c "import sys,json;print('page4 =',[r['id'] for r in json.load(sys.stdin)['results']])"
page4 = [16, 17, 14]
```

Doc **14** was on **page 3** and, after one ordinary re-index, appears again on the **immediately following page 4** — the same document on two **neighbouring** pages. Walking all four pages afterwards (repeated 2× to show the new state is itself stable) also reveals the matching skip — doc 15 slid up into page 3, which the user already passed, so it is never seen in the forward browse:

```text
$ for run in 1 2; do echo -n "run$run: "; \
    for p in 1 2 3 4; do \
      curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=$p&page_size=3&fields=id" \
      | python3 -c "import sys,json;print('p%d'%$p,[r['id'] for r in json.load(sys.stdin)['results']],end='  ')"; \
    done; echo; done
run1: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 15]  p4 [16, 17, 14]
run2: p1 [6, 7, 8]  p2 [9, 10, 11]  p3 [12, 13, 15]  p4 [16, 17, 14]
```

The faithful forward browse the user experienced is therefore `6,7,8 | 9,10,11 | 12,13,14 | 16,17,14`: doc **14** appears on adjacent pages 3 and 4 (**duplicate**), and doc **15** is never seen (**skip**) — `count` stays 12 and the "relevance" sort never changed.

**(c) "A document disappears for a page and then comes back."** Poll a _single_ page (page 3) while ordinary background consumption continues. Doc 12 starts on page 3, drops off after it is itself re-indexed, and rotates back onto page 3 as three further documents are consumed (each consume shifts the tie block by one position):

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py document_index reindex --no-progress-bar'   # rebuild index; prints nothing

# t0 — doc 12 is present on page 3:
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=3&page_size=3&fields=id" \
  | python3 -c "import sys,json;print('page3 =',[r['id'] for r in json.load(sys.stdin)['results']])"
page3 = [12, 13, 14]

# a background consume re-indexes doc 12 (exact add_to_index call):
$ docker exec -i -e TOUCH_ID=12 paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/touch_index.py
TOUCHED 12

# t1 — doc 12 has DISAPPEARED from page 3:
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=3&page_size=3&fields=id" \
  | python3 -c "import sys,json;print('page3 =',[r['id'] for r in json.load(sys.stdin)['results']])"
page3 = [13, 14, 15]

# three further ordinary consumes re-index docs 6, 7, 8:
$ for did in 6 7 8; do docker exec -i -e TOUCH_ID=$did paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/touch_index.py; done
TOUCHED 6
TOUCHED 7
TOUCHED 8

# t2 — doc 12 has COME BACK onto page 3:
$ curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&page=3&page_size=3&fields=id" \
  | python3 -c "import sys,json;print('page3 =',[r['id'] for r in json.load(sys.stdin)['results']])"
page3 = [16, 17, 12]
```

Doc **12** is on page 3 (t0), gone from page 3 (t1, right after it was re-indexed), and back on page 3 (t2, after further consumes rotate the tie block) — exactly "a document disappears for a page and then comes back," with `count` constant at 12 and the visible "relevance" sort unchanged throughout.

**`?ordering=id` is silently ignored on the search path**, so the user cannot even opt into a stable order here. Rebuild the index, re-index doc 7, then ask for `ordering=id`: doc 7 ends up **last**, not in id order — the ordering param had no effect:

```text
$ docker exec paperless-app bash -lc 'cd /app/src && python3 manage.py document_index reindex --no-progress-bar'   # rebuild index; prints nothing
$ docker exec -i -e TOUCH_ID=7 paperless-app bash -lc 'cd /app/src && python3 manage.py shell' < /tmp/touch_index.py
TOUCHED 7
$ for p in 1 2 3 4; do curl -s -u admin:admin123 "http://localhost:8000/api/documents/?query=haunted&ordering=id&page=$p&page_size=3&fields=id" \
    | python3 -c "import sys,json;print([r['id'] for r in json.load(sys.stdin)['results']],end=' ')"; done; echo
[6, 8, 9] [10, 11, 12] [13, 14, 15] [16, 17, 7]
```

(If `ordering=id` were honoured, doc 7 would sort near the front; instead it is last — confirming `sort_fields_map` has no `id` key, `src/documents/index.py:171-179`.)

Cause → effect: relevance ties + no `id` tiebreaker in `sort_fields_map` (`src/documents/index.py:171-179`) + per‑page independent `search_page` (`src/documents/index.py:203-221`) + docnum churn on re‑index (`update_document` = delete+append, `src/documents/index.py:87`) ⇒ the tied block reorders between page fetches ⇒ duplicate + skip. This is the full‑text twin of H3, and unlike §6.1 it has **no** accidental rescue, so a single background consume is enough to trigger it.

---

## 10. Backend ↔ UI correlation — why it _feels_ haunted

The backend returns a constant `count` and a stream of independently‑fetched pages; the Angular UI trusts that stream as one stable, totally‑ordered sequence and never de‑duplicates across pages. Lining the two up explains the perception exactly.

**The API contract per page** is `{count, next, previous, results[]}`; `count` is constant (12) across all requests above, so the page count and the "sort order" the user sees never change — only the _membership_ of individual pages shifts.

**The UI treats each page independently.** `document-list-view.service.ts.reload()` replaces the row list with the page's `results` and sets the paginator's total from `count`, with **no cross‑page bookkeeping or de‑duplication**:

```text
$ sed -n '149,150p' src-ui/src/app/services/document-list-view.service.ts
          activeListViewState.collectionSize = result.count
          activeListViewState.documents = result.results
```

- Default sort is `created` / reverse (`src-ui/src/app/services/document-list-view.service.ts:93-94`) — the same non‑unique key as the backend default, so the UI's out‑of‑the‑box browse rides directly on the unstable order.
- `reload()` (`src-ui/src/app/services/document-list-view.service.ts:133-155`) sets `collectionSize = result.count` and `documents = result.results` each time; it does **not** compare against previously‑seen pages, so a row repeated by the backend is rendered twice and a skipped row is simply never rendered.
- Overflowing the last page (a 404) resets to page 1 and reloads (`src-ui/src/app/services/document-list-view.service.ts:158-161`) — a benign symptom of a shrinking/shifting list, not a fix.

**The request the UI builds** is plain offset pagination: `list()` sets `page`, `page_size`, and `ordering` params (`src-ui/src/app/services/rest/abstract-paperless-service.ts:32-58`, with `ordering` from `getOrderingQueryParam`, lines 24-30); `listFiltered` adds the filter‑rule params (`src-ui/src/app/services/rest/document.service.ts:96-116`). The paginator itself is pure offset math — `<ngb-pagination [pageSize] [collectionSize] [(page)]>` (`src-ui/src/app/components/document-list/document-list.component.html:95-96`) — it asks for "page N" and renders whatever comes back.

Cause → effect: the UI assumes page N and page N+1 are consecutive, non‑overlapping windows onto one fixed ordering. When the backend's tie order shifts between those two fetches (§6.2, §9), the windows overlap (duplicate) or leave a gap (skip). Because `count` and the sort label are unchanged, nothing on screen explains it — hence "haunted."

---

## 11. Remediation (FINDING ONLY — not applied)

Per the read‑only scope, **no source file was modified**; the following is documented as a finding only.

- **Append a unique tiebreaker to force a total order.** Ordering by `("-created", "pk")` (or adding `id` to every `ORDER BY`) makes the sort total, so ties resolve deterministically and pages stop overlapping — on both the stable and the `HashAggregate` plans. This is the smallest, most direct fix for H3.
- **Or switch to DRF `CursorPagination`,** which requires a unique, unchanging ordering and paginates by an opaque cursor rather than by `OFFSET`, eliminating the boundary‑shuffle class entirely.
- **For the Whoosh path,** add a stable final tiebreaker (e.g. an `id` sort key in `sort_fields_map`, `src/documents/index.py:171-179`) so equal‑relevance ties order deterministically regardless of docnum churn.

**Why this hid for so long (inferred, from framework behaviour):** Django/DRF emit `UnorderedObjectListWarning` only when a queryset is _entirely_ unordered; ordering by a **non‑unique** column raises **no** warning. So `ORDER BY -created` looks correct, passes silently, and — thanks to the accidental `DISTINCT`‑driven `id` tiebreaker on the simplest plan (§6.1) — even behaves correctly until a plan flip or a full‑text query removes the rescue.

---

## 12. Coverage pass

| Named item                                   | Verdict                                           | Concrete observed value                                                                                                                                             | Key `file:line`                                                                                                  | Evidence (command → output)                                                                                                                                                                            | Sibling variants covered                                                                                | Causal reason                                                                                                                                                                 |
| -------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **H1** — backend duplicates collapsed later? | **Partially yes; collapsed in the DB, not later** | raw JOIN = **24** rows → `.distinct()` = **12**; API body has **0** duplicate ids                                                                                   | `src/documents/views.py:198-199`; `src/documents/filters.py:63-70,52-58`                                         | §4: shell shows 24→12; `?is_in_inbox=true` API → `has_duplicates=False`, ids 6–17                                                                                                                      | `is_in_inbox`, `tags__id__all`, `tags__id__in` branches                                                 | M2M JOIN fans out rows; `SELECT DISTINCT` in `get_queryset` collapses them in‑DB                                                                                              |
| **H2** — pagination before de‑dup?           | **No**                                            | SQL = `SELECT DISTINCT … ORDER BY "created" DESC LIMIT 3 OFFSET 3`                                                                                                  | `src/paperless/views.py:8-11`; `src/documents/views.py:198-199`                                                  | §5: `str(qs.query)` + live `CaptureQueriesContext`                                                                                                                                                     | page 1/2/3 slices                                                                                       | `.distinct()` is in the queryset **before** the paginator's `LIMIT/OFFSET`                                                                                                    |
| **H3** — unstable ordering on ties?          | **Yes — ROOT CAUSE**                              | filtered page walk `p1[6,7,8] p2[9,10,11] p3[12,13,14] p4[17,15,14]` → doc 14 duplicated (neighbouring pages 3 & 4), doc 16 skipped; `count`=12                     | `src/documents/models.py:152,207-208`; `src/paperless/views.py:8-11`; `src/documents/views.py:198-199`           | §6.1 unfiltered stable + `EXPLAIN ANALYZE` (`id` in Sort Key); §6.2 same query two plans (`enable_hashagg` off/on) → two orders; §6.3 per‑page `EXPLAIN` (pages 1‑3 Sort+Unique, page 4 HashAggregate) | tie vs no‑tie; unfiltered vs M2M‑filtered; Sort+Unique vs HashAggregate plan; per‑page boundaries       | non‑unique `created`, no tiebreaker, per‑page `LIMIT/OFFSET`; `id` appears in the sort only when the plan is Sort+Unique, so different pages/plans slice different tie orders |
| **Control** — `ordering=id`                  | **Stable** (confirms tie‑specificity)             | pages `[6,7,8][9,10,11][12,13,14][15,16,17]` identical over 3 runs, even with M2M filter; every page's Sort Key leads with `id`                                     | `src/documents/views.py:187-196`                                                                                 | §7: 3 identical runs + per‑page `EXPLAIN` Sort Key `id`                                                                                                                                                | with/without M2M filter; OFFSET 0 & 9                                                                   | `id` is unique → total order → deterministic boundaries regardless of plan                                                                                                    |
| **Non‑admin / sharing**                      | **No permission scoping; viewer‑independent**     | admin vs viewer identical ids; full‑body SHA‑256 identical `591e5eee…437a12`                                                                                        | `src/documents/views.py:183,198-199,461-463`; `src/documents/models.py:88-205`; `requirements.txt` (no guardian) | §8: paired full‑list + per‑page walks + `sha256sum` → IDENTICAL                                                                                                                                        | DB & Whoosh paths; full‑list ids, per‑page walk & full‑body hash                                        | only `IsAuthenticated`; no `owner` field; no guardian; only `SavedView` is user‑scoped                                                                                        |
| **Whoosh path** (`?query=`)                  | **Same instability class; no rescue**             | all scores `1.0`; quiescent walk 0 dup/0 skip (×3); one re‑index → doc 14 dup on neighbouring pages 3 & 4 + doc 15 skip; doc 12 disappears then returns; `count`=12 | `src/documents/index.py:165-190,203-221,87`; `src/documents/views.py:388-411`                                    | §9: quiescent walk ×3 (0/0) + neighbouring dup + disappears/comeback + `ordering=id` ignored                                                                                                           | relevance tie; quiescent vs post‑re‑index; neighbouring dup; disappears/comeback; `ordering=id` ignored | no `id` in `sort_fields_map`; per‑page `search_page`; `update_document`=delete+append churns docnum                                                                           |

**Note on the illustrative DB‑path values in this table.** The specific DB‑path figures quoted in the **H3** row (`p4[17,15,14]`; doc 14 duplicated on neighbouring pages 3 & 4; doc 16 skipped) and the **Non‑admin** row (the literal SHA‑256 `591e5eee…437a12`) are one representative single‑snapshot capture — they vary per re‑seed and per query plan, as demonstrated in §6.2, §6.3, and §8. Everything else is exactly reproducible: the verdicts, the underlying mechanism (a `HashAggregate` plan drops the `id` tiebreaker → non‑total order under per‑page `LIMIT/OFFSET`), the constant `count`, the `ordering=id` control, the admin==viewer _equality_, and the **Whoosh** row (whose neighbouring‑page duplicate is _deterministic_ — an ordinary re‑index moves the touched doc to the tie‑block end, so it reproduces byte‑for‑byte).

Every named item (H1, H2, H3, non‑admin) is answered by name with a concrete observed value, a `file:line`, captured evidence, sibling variants, and a cause→effect reason.

---

## 13. Cleanup note

This was a strictly read‑only investigation. All temporary observation scripts were written under `/tmp` (never under `blitzy/`) and were deleted after use — the seed script (`/tmp/seed_haunted.py`), the one‑document re‑index (`/tmp/touch_index.py`), the H2 capture (`/tmp/h2_capture.py`), and the various SQL/`EXPLAIN` probes. All seeded database rows (the 12 `HAUNTED…` documents and the 3 `HAUNTED…` tags) and the throwaway Whoosh index entries were removed, returning the documents table to its prior state (`count = 0`). **No existing source file was modified; this Markdown document is the only file added to the repository.**
