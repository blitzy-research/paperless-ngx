# Why the paperless-ngx documents list looks "haunted": duplicates across pages, disappearing rows, and the "sharing rules" premise — an evidence-backed root-cause investigation

> **Deliverable note.** This is a read-only, *run-first* investigation. The relevant code paths were built and run in the canonical container, temporary observation scripts were executed against the **real** `GET /api/documents/` endpoint, and this answer is written from the captured output. No source file was modified; the only change to the repository is this document (see §10.2). Every behavioral claim is placed next to the exact command and its complete, un-abbreviated output; every code claim carries an exact `file:line` citation. Statements that were only reasoned from reading (not run) are explicitly labelled **(inferred)**. All credentials in commands/output are **redacted** — placeholders such as `<redacted>` stand in for the real dev password; the redaction never removes the proof that authentication succeeded.

---

## TL;DR — what was actually observed

- **H1 — "backend duplicates collapsed later?"** *Partly true, but not the cause.* The join-producing filters **do** materialize duplicate rows (observed raw counts 10 and 20 vs. distinct 5 and 10), but they are collapsed by `SELECT DISTINCT` **inside** the query, **before** the page slice. Duplicates therefore never survive into a page. See §3.
- **H2 — "pagination applied before de-duplication?"** *False.* The emitted SQL shows filtering + `SELECT DISTINCT` (+ `ORDER BY`) in the inner query and `LIMIT`/`OFFSET` applied **last**. De-duplication happens first, the slice happens after. See §4.
- **H3 — "ordering quietly unstable on ties?"** *This is the real latent defect, but it did **not**, by itself, reproduce the symptom on a static dataset in this build.* The code's `ORDER BY "created" DESC` carries **no unique tiebreaker**, so the relative order of rows tied on `created` is **unspecified / implementation-dependent** — observed to differ across engine and query plan. **However**, driving the real endpoint over a **static** tied dataset (nobody editing) produced **stable** pages on canonical **SQLite** *and* on **PostgreSQL**, because `get_queryset()`'s `.distinct()` over *all* columns inadvertently pulls the unique `id` into the sort key. See §5.
- **What the user actually sees was reproduced through the real endpoint as offset-pagination over a *shifting* result set** — a document ingested or removed by the **background consumer** (not the user editing the rows they are viewing) between two page requests makes a document appear on two neighbouring pages, or disappear. H3's missing tiebreaker **amplifies** this. See §6.
- **The "sharing rules / what the user is allowed to see" premise is false at this commit.** There is no per-document access-control layer (no django-guardian, no `owner` field, `IsAuthenticated` only); a non-superuser receives the **identical** rows in the **identical** order as the superuser. The only genuinely user-dependent variable is per-user `SavedView` **sort** settings, which merely pick a *different but equally tie-prone* `ORDER BY`. See §7.

**The one-line fix (recommended, not applied):** append the primary key as a tiebreaker — `ORDER BY "created" DESC, "id" DESC` — and/or move to keyset/cursor pagination. See §10.1.

---

## 1. The question, decomposed

The user reports that while browsing the documents list normally (a couple of common filters, a chosen sort order that "looks unchanged", **no** full-text search box query, and **nobody editing data**), the same document can appear on two neighbouring pages, or a document can vanish from one page and reappear later. The effect is said to feel worse for **non-admin** users whose visibility is "shaped by sharing rules." Three named hypotheses must be resolved:

- **H1** — Backend produces duplicate rows that get collapsed (de-duplicated) further down the pipeline.
- **H2** — Pagination is applied **before** any de-duplication.
- **H3** — Ordering is quietly **unstable** when rows tie on the primary sort key.

Plus a premise to test rather than assume: does what-you-see depend on **per-user sharing rules / permissions**?

Every named mechanism the question implicates is addressed by name with a cause → effect explanation and re-checked in the final coverage pass (§10): the filter variants `tags__id__in`, `tags__id__all`, the `InboxFilter` true-branch, `TitleContentFilter`; the queryset-level `Document.objects.distinct()`; `StandardPagination` (`LIMIT`/`OFFSET`); the default `ORDER BY -created` and the `ordering_fields` whitelist; the `DISTINCT` + related-column edge case; the real entry point `UnifiedSearchViewSet` and its dual code path; the Whoosh full-text sibling; the frontend pagination contract; and the permission stack.

---

## 2. Environment and exact commands (canonical, default configuration)

### 2.1 Runtime

All values below were produced in the **canonical** container (`paperless-qna-0`), in the software's **default** configuration, unless a block is explicitly labelled **[PostgreSQL]** (a non-canonical *database backend*, PostgreSQL 13, run inside the *same* canonical Python 3.9 container for the H3 engine contrast in §5.5).

- Python **3.9.23** (canonical); the default database backend is **SQLite** (`django.db.backends.sqlite3`, `src/paperless/settings.py:L299`), used unless `PAPERLESS_DBHOST` is set (`src/paperless/settings.py:L304`).
- The version constant surfaces as the `X-Version: 1.7.0` response header (see §2.4).

Server started (canonical SQLite), verbatim banner from the live process log:

```text
$ cd /app/src && python manage.py runserver 0.0.0.0:8000 --noreload
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

### 2.2 The real entry point

`GET /api/documents/` is served by **`UnifiedSearchViewSet`**, not `DocumentViewSet` in isolation:

```python
# src/paperless/urls.py:L32
api_router.register(r"documents", UnifiedSearchViewSet)
```

`UnifiedSearchViewSet` subclasses `DocumentViewSet` (`src/documents/views.py:L377`). All observations below hit `/api/documents/` so the real routing, filtering, ordering, de-duplication, and pagination all execute.

### 2.3 The dual code path (why "normal browsing" is the DB path)

`UnifiedSearchViewSet` only takes the Whoosh full-text branch when a `query` or `more_like_id` parameter is present:

```python
# src/documents/views.py:L388-L392
def _is_search_request(self):
    return (
        "query" in self.request.query_params
        or "more_like_id" in self.request.query_params
    )
```

With no such parameter (the user's scenario), it falls through to the base database path:

```python
# src/documents/views.py:L410-L411
        else:
            return super(UnifiedSearchViewSet, self).filter_queryset(queryset)
```

So "normal browsing with a couple of filters and a sort" is the **database** path. The Whoosh path is enumerated for completeness in §9 and is **(inferred)** only.

### 2.4 How observations were taken (real endpoint, not a bypass)

Two complementary harnesses were used, both routed through the real URL `/api/documents/`:

1. **Live HTTP** — a real `runserver` process; requests issued with Python `requests` against `http://127.0.0.1:8000/api/documents/` and header `Accept: application/json; version=2`, and `curl` from the host for the raw status line + headers. This is the canonical real entry point.
2. **SQL capture** — `django.test.Client().get("/api/documents/?page=1&page_size=25&ordering=-created")` (and the other URLs shown below), which resolves the **same URLconf → `UnifiedSearchViewSet`** (it is *not* a bypass of the view; it is standard Django URL resolution) wrapped in `CaptureQueriesContext` so the **exact emitted SQL** can be printed in full.

Verbatim response status line and headers from the real endpoint (host `curl`; credentials redacted in the command; the response itself contains **no** cookies/CSRF/token/authorization headers, so it is shown in full):

```text
$ curl -sS -D - -o /dev/null -u admin:<redacted> \
    -H 'Accept: application/json; version=2' \
    'http://localhost:8000/api/documents/?page=1&page_size=25&ordering=-created'
HTTP/1.1 200 OK
Date: Fri, 03 Jul 2026 00:16:30 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 7824
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

`HTTP/1.1 200 OK` with `-u admin:<redacted>` confirms the superuser authenticated. `X-Api-Version: 2` confirms the versioned Accept header took effect; `X-Version: 1.7.0` is the running build.

### 2.5 Seed (enough rows tied on `created` to span multiple pages at `page_size = 25`)

60 documents were created all sharing an identical `created` timestamp (`2020-01-01T00:00:00Z`); tags `alpha`,`beta` were attached to 10 docs and inbox tags `inboxA`,`inboxB` to 5, so the tag filters can join-multiply rows. The seed mutates only the **git-ignored** dev database. Verbatim:

```text
$ PYTHONPATH=/app/src python /tmp/blz/seed.py
ENGINE= sqlite
SEED_TOTAL_DOCS= 60
SEED_TIED_ON_CREATED= 60
TAG_IDS alpha,beta,inboxA,inboxB= 1 2 3 4
MIN_DOC_ID= 1 MAX_DOC_ID= 60 PAD= 0
```

`SEED_TIED_ON_CREATED= 60` confirms all 60 rows tie on `created`; ids run 1–60 (independently confirmed by the `ordering=id` walk in §5.2, which returns exactly `[1..25]`, `[26..50]`, `[51..60]`). At `page_size = 25` that is three pages (25 + 25 + 10).

---

## 3. H1 — the join filters *do* make duplicates, but `DISTINCT` collapses them **before** the slice

**Question:** does the browsing query materialize duplicate rows, and where are they collapsed? **Answer:** duplicates are produced by the many-to-many joins, and they are collapsed by `DISTINCT` in the inner query — so they never reach a page. Below, each named filter variant is exercised through the real endpoint and its **full** emitted SQL is shown (each `SELECT DISTINCT` lists all 15 `documents_document` columns; nothing is abbreviated). Each list request also emits one `documents_tag` prefetch query per returned row (DRF `prefetch_related('tags')`); those are identical in shape except for the trailing `document_id`, so one representative line is shown verbatim at the end of §3.5 rather than repeated per row.

### 3.1 `tags__id__in` — one join + the filter's **own** `.distinct()`

Code:

```python
# src/documents/filters.py:L51-L52
        if self.in_list:
            qs = qs.filter(tags__id__in=tag_ids).distinct()
```

Observed (raw rows would be 20; endpoint returns `count=10`):

```text
$ GET /api/documents/?ordering=id&tags__id__in=1,2
HTTP_STATUS=200 COUNT=10
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") WHERE "documents_document_tags"."tag_id" IN (1, 2)) subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") WHERE "documents_document_tags"."tag_id" IN (1, 2) ORDER BY "documents_document"."id" ASC LIMIT 10
```

**Cause → effect:** the `INNER JOIN documents_document_tags` yields one row per (document, matching-tag) pair (a document with both tag 1 and tag 2 appears twice), and `SELECT DISTINCT` folds those back to one row **before** the `LIMIT`. Ten distinct documents are returned.

### 3.2 `tags__id__all` — repeated joins, relies on the queryset-level `DISTINCT`

Code:

```python
# src/documents/filters.py:L54-L58
        else:
            for tag_id in tag_ids:
                if self.exclude:
                    qs = qs.exclude(tags__id=tag_id)
                else:
                    qs = qs.filter(tags__id=tag_id)
```

Observed (two joins — note the aliased `T4` — one AND-combination per document):

```text
$ GET /api/documents/?ordering=id&tags__id__all=1,2
HTTP_STATUS=200 COUNT=10
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_document_tags" T4 ON ("documents_document"."id" = T4."document_id") WHERE ("documents_document_tags"."tag_id" = 1 AND T4."tag_id" = 2)) subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_document_tags" T4 ON ("documents_document"."id" = T4."document_id") WHERE ("documents_document_tags"."tag_id" = 1 AND T4."tag_id" = 2) ORDER BY "documents_document"."id" ASC LIMIT 10
```

**Cause → effect:** each additional `tags__id=<tag_id>` clause adds another join (here the aliased `T4`), which can multiply rows; `DISTINCT` is what keeps the result to one row per document.

### 3.3 `InboxFilter` true-branch — **no local `.distinct()`** (collapse depends on the queryset)

Code (note: **no** `.distinct()` here, unlike `TagsFilter.in_list`):

```python
# src/documents/filters.py:L63-L66
class InboxFilter(Filter):
    def filter(self, qs, value):
        if value == "true":
            return qs.filter(tags__is_inbox_tag=True)
```

Observed (raw rows would be 10; endpoint returns `count=5`):

```text
$ GET /api/documents/?ordering=id&is_in_inbox=true
HTTP_STATUS=200 COUNT=5
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_tag" ON ("documents_document_tags"."tag_id" = "documents_tag"."id") WHERE "documents_tag"."is_inbox_tag") subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_tag" ON ("documents_document_tags"."tag_id" = "documents_tag"."id") WHERE "documents_tag"."is_inbox_tag" ORDER BY "documents_document"."id" ASC LIMIT 5
```

**Cause → effect:** `is_in_inbox=true` joins through to `documents_tag` and, because a document can carry more than one inbox tag, multiplies rows (raw 10). The `SELECT DISTINCT` here comes **only** from the queryset-level `Document.objects.distinct()` (§3.5), not from the filter — this is the latent fragility the question's H1 hints at: `InboxFilter` relies entirely on the shared `.distinct()`.

### 3.4 `TitleContentFilter` — same-table OR, **no join**, no multiplication

Code:

```python
# src/documents/filters.py:L73-L78
class TitleContentFilter(Filter):
    def filter(self, qs, value):
        if value:
            return qs.filter(Q(title__icontains=value) | Q(content__icontains=value))
        else:
            return qs
```

Observed (an OR over two columns of the **same** table — no join, so no row multiplication):

```text
$ GET /api/documents/?ordering=id&title_content=tied-00
HTTP_STATUS=200 COUNT=10
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document" WHERE ("documents_document"."title" LIKE '%tied-00%' ESCAPE '\' OR "documents_document"."content" LIKE '%tied-00%' ESCAPE '\')) subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" WHERE ("documents_document"."title" LIKE '%tied-00%' ESCAPE '\' OR "documents_document"."content" LIKE '%tied-00%' ESCAPE '\') ORDER BY "documents_document"."id" ASC LIMIT 10
```

**Cause → effect:** because there is no join, no row is duplicated; `TitleContentFilter` cannot contribute duplicate rows.

### 3.5 The queryset-level `Document.objects.distinct()` and raw-vs-distinct proof

```python
# src/documents/views.py:L198-L199
    def get_queryset(self):
        return Document.objects.distinct()
```

That `.distinct()` is the shared collapse relied on by the join filters (especially `InboxFilter`, §3.3). Raw-vs-distinct counts, run through the ORM, verbatim:

```text
$ Document.objects.filter(tags__is_inbox_tag=True).count()
RAW_INBOX= 10
$ Document.objects.filter(tags__is_inbox_tag=True).distinct().count()
DIST_INBOX= 5
$ Document.objects.filter(tags__id__in=[1,2]).count()
RAW_IN_1_2= 20
$ Document.objects.filter(tags__id__in=[1,2]).distinct().count()
DIST_IN_1_2= 10
```

The representative per-row tag prefetch query emitted by the serializer (one such line per returned row, differing only in the trailing `document_id`; shown here for `document_id = 60`):

```text
SQL: SELECT "documents_tag"."id", "documents_tag"."name", "documents_tag"."match", "documents_tag"."matching_algorithm", "documents_tag"."is_insensitive", "documents_tag"."color", "documents_tag"."is_inbox_tag" FROM "documents_tag" INNER JOIN "documents_document_tags" ON ("documents_tag"."id" = "documents_document_tags"."tag_id") WHERE "documents_document_tags"."document_id" = 60
```

**H1 verdict:** duplicates are real at the join level (raw 10/20) but collapsed to distinct rows (5/10) **inside** the query. Since the collapse is in the inner query (§4), duplicates cannot survive into a page. H1 is therefore **not** the cause of the same document appearing on two pages.

---

## 4. H2 — order of operations: filter + `DISTINCT` first, `LIMIT`/`OFFSET` last

**Question:** is pagination applied *before* de-duplication? **Answer: no.** The full emitted SQL for the default browse (the user's scenario, `ordering=-created`) shows two statements per page — a `COUNT(*)` over the de-duplicated subquery, and the paginated fetch where `DISTINCT` and `ORDER BY` are in the body and `LIMIT`/`OFFSET` is appended last.

### 4.1 Page 1 (verbatim, full)

```text
$ GET /api/documents/?page=1&page_size=25&ordering=-created
HTTP_STATUS=200 COUNT=60
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document") subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 25
```

### 4.2 Page 2 (verbatim, full) — only `OFFSET 25` is added

```text
$ GET /api/documents/?page=2&page_size=25&ordering=-created
HTTP_STATUS=200 COUNT=60
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document") subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 25 OFFSET 25
```

**Cause → effect:** the inner `SELECT DISTINCT` over all 15 columns followed by `ORDER BY "documents_document"."created" DESC` (shown verbatim above) is the whole inner computation; `LIMIT 25` (page 1) / `LIMIT 25 OFFSET 25` (page 2) is appended **after** it. The de-duplication is done, then the window is sliced. The pagination is `StandardPagination`:

```python
# src/paperless/views.py:L8-L11
class StandardPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = "page_size"
    max_page_size = 100000
```

wired at `pagination_class = StandardPagination` (`src/documents/views.py:L182`). This is `LIMIT`/`OFFSET` (page-number) pagination — the family sensitive to unstable tie order (§5). **H2 is false**: de-duplication precedes the slice.

### 4.3 The `DISTINCT` + related-column ordering edge case (`correspondent__name`)

The whitelist of orderable columns is:

```python
# src/documents/views.py:L187-L196
    ordering_fields = (
        "id",
        "title",
        "correspondent__name",
        "document_type__name",
        "created",
        "modified",
        "added",
        "archive_serial_number",
    )
```

When ordering targets a **related** column, Django must add that column to the `SELECT DISTINCT` list (you cannot order by a column that is not selected under `DISTINCT`). Observed verbatim — note the trailing `"documents_correspondent"."name"` pulled into the distinct list and the `LEFT OUTER JOIN`:

```text
$ GET /api/documents/?page=1&page_size=25&ordering=correspondent__name
HTTP_STATUS=200 COUNT=60
SQL: SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document") subquery
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number", "documents_correspondent"."name" FROM "documents_document" LEFT OUTER JOIN "documents_correspondent" ON ("documents_document"."correspondent_id" = "documents_correspondent"."id") ORDER BY "documents_correspondent"."name" ASC LIMIT 25
```

**Cause → effect:** adding `"documents_correspondent"."name"` to `SELECT DISTINCT` means de-duplication now considers that extra column. For a **to-one** relation like `correspondent` this is harmless here, but for a **to-many** ordering target combined with a join-multiplying filter it would weaken de-duplication (distinct now includes the related column, so multiplied rows can differ on it and survive). This is a real edge case of `DISTINCT`-plus-related-ordering and is the mechanism the question gestures at; here `count=60` (unchanged) because `correspondent` is to-one and unset.

---

## 5. H3 — the ordering is unstable on ties (latent), and why the static case still came out stable

**Question:** is the ordering quietly unstable when rows tie on the primary key? **Answer: the *specification* is unstable — the code's `ORDER BY` has no unique tiebreaker, so tie order is undefined and was observed to differ across engine/plan — but on a *static* dataset through the real endpoint the pages came out stable on both engines**, because `.distinct()` over all columns inadvertently supplies a tiebreaker. This section proves both halves with observed output.

### 5.1 The default order carries no unique tiebreaker

```python
# src/documents/models.py:L207-L208
    class Meta:
        ordering = ("-created",)
```

with `created` tie-prone and indexed:

```python
# src/documents/models.py:L152
    created = models.DateTimeField(_("created"), default=timezone.now, db_index=True)
```

Of the whitelisted `ordering_fields` (§4.3), only `id` is guaranteed unique; `created`, `title`, `correspondent__name`, `document_type__name`, `modified`, `added`, and `archive_serial_number` can all tie. The emitted clause for the default browse is exactly `ORDER BY "documents_document"."created" DESC` (§4.1/§4.2) — **no `id` (or other unique) tiebreaker is appended**. When many rows share one `created`, their relative order is left to the database.

### 5.2 Contrast: `ordering=id` (a unique key) is perfectly stable

The `ordering=id` fetch emits a unique-key sort (full query, verbatim):

```text
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."id" ASC LIMIT 25
```

and its page-walk is disjoint and complete:

```text
########## ordering=id (admin) [SQLite, unique key contrast]
page 1: count=60 n=25 ids=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25]
page 2: count=60 n=25 ids=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
page 3: count=60 n=10 ids=[51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
total_ids_returned=60 unique=60 DUPLICATES=[] OMISSIONS=[]
```

**Cause → effect:** a unique key makes the order total, so `OFFSET` slices a fixed sequence — no duplicates, no gaps.

### 5.3 The tie order is genuinely unspecified (same rows, different order per sort/engine)

The very same 60 rows come back in **different** tie orders depending on which tie-prone column is sorted (all captured at `page_size=60`, SQLite):

```text
########## Tie-order is arbitrary (full 60-row union per ordering) [SQLite]
CMD: GET /api/documents/?page=1&page_size=60&ordering=-created
ids=[60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
CMD: GET /api/documents/?page=1&page_size=60&ordering=correspondent__name
ids=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
CMD: GET /api/documents/?page=1&page_size=60&ordering=document_type__name
ids=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
```

**Cause → effect:** with `-created` the tie block emerges in `id`-descending order (the first id is `60` and the last is `1`, per the verbatim array above); with the all-`NULL` related columns `correspondent__name` / `document_type__name` the tie block emerges in `id`-ascending order (first id `1`, last id `60`). Same rows, different order — decided by scan/plan, not by any unique key in the `ORDER BY`. The order is therefore **unspecified**. (This corroborates the standard database guidance that `OFFSET` over a non-unique sort can duplicate or skip rows, and that the remedy is a unique tiebreaker such as the primary key.)

### 5.4 Canonical SQLite, static dataset, two full page-walks — **stable** (observed)

Two complete page-walks of the default browse (nobody editing between or during them) returned **identical** pages with **no** duplicates and **no** omissions:

```text
########## PASS 1 ordering=-created (admin) [SQLite]
page 1: count=60 n=25 ids=[60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36]
page 2: count=60 n=25 ids=[35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11]
page 3: count=60 n=10 ids=[10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
total_ids_returned=60 unique=60 DUPLICATES=[] OMISSIONS=[]

########## PASS 2 ordering=-created (admin) [SQLite]
page 1: count=60 n=25 ids=[60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36]
page 2: count=60 n=25 ids=[35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11]
page 3: count=60 n=10 ids=[10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
total_ids_returned=60 unique=60 DUPLICATES=[] OMISSIONS=[]
```

**Observed truth (stated exactly as measured, even though H3 was the leading hypothesis):** on canonical **SQLite**, the static tied dataset paginates **stably** across two passes — `DUPLICATES=[] OMISSIONS=[]` both times. The pure H3 tie-instability does **not**, on its own, produce the haunted anomaly here. Why: although the *specified* `ORDER BY` is `created DESC` only, SQLite's single deterministic plan over static data yields a repeatable tie order, so `OFFSET` slices a repeatable sequence.

### 5.5 PostgreSQL (non-canonical **backend**, actually run) — also stable, and the EXPLAIN shows why

To test whether a different engine/plan exposes the latent instability, PostgreSQL **13.23** was installed and run **inside the same canonical Python 3.9 container** (this is a non-canonical *database backend*; the default remains SQLite). Version, verbatim:

```text
$ /usr/lib/postgresql/13/bin/postgres --version
postgres (PostgreSQL) 13.23 (Debian 13.23-0+deb11u4)
```

Django was pointed at it (`PAPERLESS_DBHOST` set), confirmed at runtime as `RUNTIME_ENGINE= django.db.backends.postgresql_psycopg2`; 2000 rows all tied on `created` were seeded; and the table was forced onto a **4-worker parallel sequential-scan** plan via `ALTER DATABASE paperless SET` for each of `max_parallel_workers_per_gather = 4`, `min_parallel_table_scan_size = 0`, `parallel_setup_cost = 0`, `parallel_tuple_cost = 0`, `enable_indexscan = off`, and `enable_bitmapscan = off` — the configuration most likely to expose nondeterministic tie order.

**The real query’s plan and its 6× repeat window (verbatim):**

```text
########## REAL distinct query (page 2 window)
SQL: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 25 OFFSET 25
exec 1 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
exec 2 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
exec 3 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
exec 4 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
exec 5 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
exec 6 window(offset25,limit25)=[26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
ALL_6_IDENTICAL=True
PLAN:
  Limit
    ->  Unique
          ->  Gather Merge
                Workers Planned: 4
                ->  Sort
                      Sort Key: created DESC, id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, modified, storage_type, added, filename, archive_filename, archive_serial_number
                      ->  Parallel Seq Scan on documents_document
```

**Cause → effect (the key finding):** because `.distinct()` (§3.5) is applied over **all** selected columns, PostgreSQL's `Unique` step forces the `Sort Key` to include **every** column — including the unique **`id`**. That accidentally makes the sort **total**, so even a 4-worker parallel scan yields a deterministic window (`ALL_6_IDENTICAL=True`). The `.distinct()` intended for de-duplication *inadvertently* supplies the very tiebreaker H3 says is missing — which is why the real endpoint is stable for static data on PostgreSQL too.

**Proof the *specified* order (without that accident) is only latently stable** — the pure `SELECT "documents_document"."id" FROM "documents_document" ORDER BY "documents_document"."created" DESC` (no distinct, so the `Sort Key` is `created DESC` only; full query in the block below) returns a **different** window and, critically, one that is *not* `[26..50]`:

```text
########## NO-tiebreaker query id-only (page 2 window)
SQL: SELECT "documents_document"."id" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 25 OFFSET 25
exec 1 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
exec 2 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
exec 3 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
exec 4 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
exec 5 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
exec 6 window(offset25,limit25)=[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]
ALL_6_IDENTICAL=True
PLAN:
  Limit
    ->  Gather Merge
          Workers Planned: 4
          ->  Sort
                Sort Key: created DESC
                ->  Parallel Seq Scan on documents_document
```

**Cause → effect:** with the `Sort Key` reduced to `created DESC`, the page-2 window (shown verbatim above) is `[27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 1]` — an **arbitrary** slice in which id `1` lands at the end of page 2 while id `26` fell onto page 1, differing from the real query's `[26..50]`. It is *repeatable* only because the data and plan are fixed (`ALL_6_IDENTICAL=True`); the specific membership is decided by the plan, not by any key. This is exactly H3's latent hazard: the moment the accidental `id`-in-sort-key goes away (a stats-driven `HashAggregate` de-dup plan that sorts on `created` alone, a different engine or version, a different worker count), the page boundaries move.

**Live real-endpoint PostgreSQL page-walks — stable across two passes and five identical repeats:**

```text
########## PASS 1 ordering=-created (admin) [PostgreSQL, page_size=500]
page 1: n=500 first5=[1, 2, 3, 4, 5] last5=[496, 497, 498, 499, 500]
page 2: n=500 first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
page 3: n=500 first5=[1001, 1002, 1003, 1004, 1005] last5=[1496, 1497, 1498, 1499, 1500]
page 4: n=500 first5=[1501, 1502, 1503, 1504, 1505] last5=[1996, 1997, 1998, 1999, 2000]
count=2000 total_returned=2000 unique=2000 DUPLICATES=[] N_OMISSIONS=0

########## PASS 2 ordering=-created (admin) [PostgreSQL, page_size=500]
page 1: n=500 first5=[1, 2, 3, 4, 5] last5=[496, 497, 498, 499, 500]
page 2: n=500 first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
page 3: n=500 first5=[1001, 1002, 1003, 1004, 1005] last5=[1496, 1497, 1498, 1499, 1500]
page 4: n=500 first5=[1501, 1502, 1503, 1504, 1505] last5=[1996, 1997, 1998, 1999, 2000]
count=2000 total_returned=2000 unique=2000 DUPLICATES=[] N_OMISSIONS=0

########## Repeated IDENTICAL request, NO data edits (page=2, ordering=-created) [PostgreSQL]
req 1: first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
req 2: first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
req 3: first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
req 4: first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
req 5: first5=[501, 502, 503, 504, 505] last5=[996, 997, 998, 999, 1000]
ALL_5_IDENTICAL=True
```

**H3 verdict (observed):** the *specification* is unstable — the code's `ORDER BY` has **no unique tiebreaker**, and the same rows demonstrably order differently across sort column, engine, and plan (§5.3, §5.5). But on a **static** dataset through the real endpoint, pages were **stable** on both canonical SQLite and PostgreSQL because `.distinct()` over all columns accidentally supplies `id` in the sort key. So **H3 is a real *latent* defect and an amplifier, not the demonstrated static no-edit trigger in this build.** The trigger that actually reproduces the user's symptom is in §6.

---

## 6. What actually makes documents duplicate/disappear "with nobody editing": background result-set shift (reproduced through the real endpoint)

Since a static dataset paginates stably (§5.4/§5.5), the reproducible cause of the user's symptom is **offset pagination over a result set that changes between two independent per-page requests** — and in paperless the thing that changes it "without the user editing" is the **background consumer** ingesting a newly-scanned document or removing one (delete/merge). This is "no user edit" in exactly the sense the user means: they are not editing the rows they are viewing, and their sort looks unchanged, yet the list is haunted.

Both halves were reproduced through the **real** `GET /api/documents/` endpoint on canonical SQLite. Verbatim:

### 6.1 Duplicate — one document ingested between page 1 and page 2

```text
ENGINE= sqlite
### DUPLICATE demo — background ingestion of ONE new doc between page1 and page2 [SQLite] ###
CMD page1 BEFORE:  GET /api/documents/?page=1&page_size=25&ordering=-created
page1_BEFORE = [60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36]
BACKGROUND INGEST: Document.objects.create(created=now) -> id=61 (consumer activity, not a user edit)
CMD page2 AFTER:   GET /api/documents/?page=2&page_size=25&ordering=-created
page2_AFTER  = [36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12]
DUPLICATE ids shown on BOTH neighboring pages = [36]
```

**Cause → effect:** the new document (`id=61`, `created=now`) sorts to the very top, shifting every row down one position. `OFFSET 25` now lands one row earlier in the sequence, so `id=36` — the last row of page 1 — reappears as the first row of page 2. The user sees the same document twice, having edited nothing.

### 6.2 Omission — one page-1 document removed before page 2

```text
### OMISSION demo — ONE doc removed between page1 and page2 [SQLite] ###
CMD page1 BEFORE:  GET /api/documents/?page=1&page_size=25&ordering=-created
page1_BEFORE = [60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36]
page2_BEFORE = [35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11]
REMOVE doc id=60 (a page-1 row; e.g. deleted/merged in the background)
CMD page2 AFTER:   GET /api/documents/?page=2&page_size=25&ordering=-created
page2_AFTER  = [34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10]
OMITTED ids (were page2's head, pulled into already-viewed page1 tail) = [35]
```

**Cause → effect:** removing a page-1 row shifts everything up one position, so what would have been page 2's head (`id=35`) is pulled into the (already-viewed) tail of page 1's window; on the next page-2 request it is skipped. The user never sees `id=35` — a document "disappeared," again with no edit on their part.

### 6.3 How H3 amplifies this

The two demos above happen with **any** `LIMIT`/`OFFSET` pagination when the result set shifts, even with a perfect total order. H3 makes it **worse**: because rows tied on `created` have no fixed relative order (§5.3/§5.5), a single insert/delete — or any plan change — can reshuffle an **entire tie block**, not just the one neighbouring row, turning a one-row slip into a scattered set of duplicates and gaps. The missing tiebreaker (`src/documents/models.py:L207-L208`; whitelist `src/documents/views.py:L187-L196` where only `id` is unique) is the amplifier; the result-set shift is the trigger.

---

## 7. The permission premise (R5): there is no per-document "sharing" at this commit

### 7.1 No object-permission layer exists (runtime dump)

The following was dumped at runtime through the app's own settings/models:

```text
INSTALLED_APPS = ['whitenoise.runserver_nostatic', 'django.contrib.auth', 'django.contrib.contenttypes', 'django.contrib.sessions', 'django.contrib.messages', 'django.contrib.staticfiles', 'corsheaders', 'django_extensions', 'paperless', 'documents.apps.DocumentsConfig', 'paperless_tesseract.apps.PaperlessTesseractConfig', 'paperless_text.apps.PaperlessTextConfig', 'paperless_mail.apps.PaperlessMailConfig', 'django.contrib.admin', 'rest_framework', 'rest_framework.authtoken', 'django_filters', 'django_q']
ANY_GUARDIAN = False
DOCUMENT_FIELDS = ['id', 'correspondent', 'title', 'document_type', 'content', 'mime_type', 'checksum', 'archive_checksum', 'created', 'modified', 'storage_type', 'added', 'filename', 'archive_filename', 'archive_serial_number', 'tags']
HAS_OWNER = False
HAS_SHARED = False
DocumentViewSet.permission_classes      = (<class 'rest_framework.permissions.IsAuthenticated'>,)
UnifiedSearchViewSet.permission_classes = (<class 'rest_framework.permissions.IsAuthenticated'>,)
GET_QUERYSET_SOURCE:
    def get_queryset(self):
        return Document.objects.distinct()

USERS = [('consumer', False), ('admin', True), ('temp_viewer', False)]
temp_viewer is_superuser=False is_staff=False is_active=True
```

- `ANY_GUARDIAN = False` — django-guardian is not installed (`INSTALLED_APPS`, `src/paperless/settings.py:L92-L111`).
- `HAS_OWNER = False`, `HAS_SHARED = False` — the `Document` model has no `owner`/`shared` field (`src/documents/models.py:L88-L206`).
- Both viewsets require only authentication (`src/documents/views.py:L183`): `permission_classes = (IsAuthenticated,)`.
- `get_queryset()` is user-independent (`src/documents/views.py:L198-L199`): `return Document.objects.distinct()` — the same queryset for every authenticated user.

### 7.2 A non-superuser sees the identical rows in identical order (full arrays + equality)

The same default page-walk was driven as the non-superuser `temp_viewer` (`is_superuser=False`; authenticated → `HTTP 200`) and as `admin`, and the complete union arrays were compared deterministically. Verbatim (no abbreviation):

```text
########## R5 permission comparison admin vs temp_viewer, ordering=-created [SQLite]
admin_count=60
admin_union_ids=[60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
temp_viewer_count=60
temp_viewer_union_ids=[60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
SAME_COUNT=True
SAME_SET=True
IDENTICAL_ORDER=True
```

**Cause → effect:** because no code path filters `Document` rows by the requesting user, the non-admin receives **exactly** the same 60 documents in the same order as the superuser — `SAME_COUNT=True`, `SAME_SET=True`, `IDENTICAL_ORDER=True`. There is **no** row-level "sharing" and no per-user visibility of the document set at this commit.

### 7.3 What the perceived per-user behaviour actually is

The genuine user-dependent variable is **per-user `SavedView` sort persistence**: `user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name=_("user"))` (`src/documents/models.py:L323`), `sort_field` (`src/documents/models.py:L333`), `sort_reverse` (`src/documents/models.py:L339`), on the model class `class SavedView(models.Model)` at `src/documents/models.py:L316`.

**Cause → effect:** two users can have different saved sorts (e.g. one on `created`, another on `correspondent__name`). As §5.3 shows, those pick **different but equally tie-prone** `ORDER BY` clauses over the **same** row set — so different users experience *different arbitrary orderings* (and, under result-set shift, different duplicate/disappear patterns), which *feels* like "it's worse for some users." It is **sort-driven instability, not a filtered/ACL'd row set.** The "sharing rules" premise does not hold at commit `542221a38`.

---

## 8. Frontend pagination contract alignment (R4)

The frontend (under `src-ui/`, Angular/TypeScript) is **reference-only** here (Node is not installed in this backend investigation; the citations below are static reads and are labelled **(inferred)** where they describe runtime behaviour not exercised in this environment). It explains *why* an unstable backend order — or a shifted result set — becomes a visible duplicate/disappearance in the UI.

- **Each page is an independent HTTP request.** `AbstractPaperlessService.getOrderingQueryParam` returns `(sortReverse ? '-' : '') + sortField` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:L24-L26`); the `list()` method (declared at `src-ui/src/app/services/rest/abstract-paperless-service.ts:L32`) sets `page` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:L41`), `page_size` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:L44`), and `ordering` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:L48`) as query params. There is **no** snapshot/cursor tying the pages together. **(inferred)**
- **Default sort → `ordering=-created`.** `currentPage: number` (`src-ui/src/app/services/document-list-view.service.ts:L29`); defaults `sortField: 'created'` (`src-ui/src/app/services/document-list-view.service.ts:L93`), `sortReverse: true` (`src-ui/src/app/services/document-list-view.service.ts:L94`). The `reload(onFinish?)` method (`src-ui/src/app/services/document-list-view.service.ts:L133`) performs the per-page request and sets `collectionSize = result.count` (`src-ui/src/app/services/document-list-view.service.ts:L149`); on paging past the last page it resets: `if (activeListViewState.currentPage != 1 && error.status == 404)` (`src-ui/src/app/services/document-list-view.service.ts:L158`) → `currentPage = 1` (`src-ui/src/app/services/document-list-view.service.ts:L160`) → `this.reload()` (`src-ui/src/app/services/document-list-view.service.ts:L161`). The list component drives it via `this.list.reload()` (`src-ui/src/app/components/document-list/document-list.component.ts:L107`, `src-ui/src/app/components/document-list/document-list.component.ts:L126`, `src-ui/src/app/components/document-list/document-list.component.ts:L164`, `src-ui/src/app/components/document-list/document-list.component.ts:L197`). **(inferred)**
- **The UI derives only the *page count* from `count`** — `Results<T> { count: number; results: T[] }` (`src-ui/src/app/data/results.ts:L1-L4`). It does **not** pin row identity across pages, so if the backend returns an overlapping/gapped window (§6) the UI faithfully renders the duplicate/disappearance. **(inferred)**
- **Reinforcement of H3 — the UI cannot self-heal.** `DOCUMENT_SORT_FIELDS` (`src-ui/src/app/services/rest/document.service.ts:L16-L24`) offers exactly `archive_serial_number`, `correspondent__name`, `title`, `document_type__name`, `created`, `added`, `modified` — it **does not include `id`**; `DOCUMENT_SORT_FIELDS_FULLTEXT` merely adds `score` (`src-ui/src/app/services/rest/document.service.ts:L26-L32`, the search path). So a UI user can only pick a **tie-prone, non-unique** sort; the one guaranteed-unique tiebreaker (`id`, present in the backend `ordering_fields`, `src/documents/views.py:L187-L196`) is **not selectable from the UI**, so the user cannot stabilize their own pages. **(inferred)**

**Cause → effect:** independent per-page requests + `ordering=-created` (no unique tiebreaker) + a `count`-only page model ⇒ whenever the backend's window shifts between those requests (result-set change, or a tie-order change), the UI shows the same document twice or drops one — precisely the reported symptom.

---

## 9. The Whoosh full-text sibling path (enumerated for completeness — OUT of the user's scenario)

The user's scenario has **no** search-box query, so it never enters the full-text branch; that branch is enumerated only so the answer is exhaustive. **(inferred — read only; not exercised, since it is outside the user's no-query scenario.)**

- It is taken **only** when `query` or `more_like_id` is present (`_is_search_request`, `src/documents/views.py:L388-L392`). `UnifiedSearchViewSet.get_serializer_class` then switches to `SearchResultSerializer` (`src/documents/views.py:L382-L386`), which is defined at `src/documents/views.py:L362` (a subclass of `DocumentSerializer`), **not** in `serialisers.py`.
- Its pagination is Whoosh's own `search_page`, not SQL `LIMIT`/`OFFSET` — `DelayedQuery.__getitem__` (`src/documents/index.py:L203`) calls (`src/documents/index.py:L210-L218`):
  ```python
  page: ResultsPage = self.searcher.search_page(
      q,
      mask=mask,
      filter=self._get_query_filter(),
      pagenum=math.floor(item.start / self.page_size) + 1,
      pagelen=self.page_size,
      sortedby=sortedby,
      reverse=reverse,
  )
  ```
- **Cause → effect (relevance ties, a different animal):** here "ties" are equal **relevance scores**, not equal column values, and paging is by `pagenum`/`pagelen` over the Whoosh result set. It is a distinct mechanism and **not** the reported browsing defect.

---

## 10. Final coverage pass

Every item the question names, addressed with cause → effect and grounded in code (`file:line`) and/or observed output:

| # | Named item | Where addressed | Verdict (cause → effect) |
|---|------------|-----------------|--------------------------|
| 1 | **H1** — backend duplicates collapsed later | §3 | Real, but not the cause: joins multiply rows (raw 10/20) and `SELECT DISTINCT` (`src/documents/views.py:L198-L199`) collapses them **before** the slice → duplicates never reach a page. |
| 2 | **H2** — pagination before de-duplication | §4 | **False**: emitted SQL puts filter/`DISTINCT`/`ORDER BY` inner and `LIMIT`/`OFFSET` last → de-dup first, then slice. |
| 3 | **H3** — unstable ordering on ties | §5, §6.3 | **Latent defect + amplifier, not the static trigger here**: `ORDER BY created DESC` has **no unique tiebreaker** so tie order is unspecified (differs by sort/engine/plan), but `.distinct()` over all columns accidentally adds `id` to the sort key → static pages came out **stable** on SQLite and PostgreSQL. |
| 4 | `TagsFilter` `tags__id__in` | §3.1 | One M2M join + own `.distinct()` (`src/documents/filters.py:L51-L52`); collapses `count=10` (raw 20). |
| 5 | `TagsFilter` `tags__id__all` | §3.2 | Repeated joins (aliased `T4`) (`src/documents/filters.py:L54-L58`); relies on queryset `DISTINCT`; `count=10`. |
| 6 | `InboxFilter` true-branch | §3.3 | `qs.filter(tags__is_inbox_tag=True)` (`src/documents/filters.py:L65-L66`), **no local `.distinct()`**; collapses `count=5` (raw 10) via queryset `DISTINCT` only. |
| 7 | `TitleContentFilter` | §3.4 | OR over same-table `title`/`content` (`src/documents/filters.py:L73-L78`); **no join**, no multiplication. |
| 8 | `get_queryset().distinct()` | §3.5, §5.5 | `Document.objects.distinct()` (`src/documents/views.py:L198-L199`) collapses join multiplication and, over all columns, accidentally supplies `id` in the sort key. |
| 9 | `DISTINCT` + related-column ordering | §4.3 | `correspondent__name`/`document_type__name` (`src/documents/views.py:L187-L196`) pulled into `SELECT DISTINCT` (verbatim SQL); would weaken de-dup for a to-many ordering target. |
| 10 | Tie on `created` | §2.5, §5.1, §5.3 | 60 rows seeded at `created=2020-01-01T00:00:00Z` (`src/documents/models.py:L152`); `Meta.ordering=("-created",)` (`src/documents/models.py:L207-L208`). |
| 11 | Page beyond last → DRF `NotFound` / UI reset to page 1 | §8 | UI resets on `error.status == 404` → `currentPage = 1` → `this.reload()` (`src-ui/src/app/services/document-list-view.service.ts:L158-L161`). |
| 12 | `NULL` / equal `created` values | §5.3 | All-equal `created` (and all-`NULL` related columns) observed to yield arbitrary DESC vs ASC tie order. |
| 13 | `page_size` boundary at exactly 25 | §4, §5.4 | `StandardPagination.page_size = 25` (`src/paperless/views.py:L8-L11`); pages 25/25/10 over 60 rows; `LIMIT 25 [OFFSET 25]` verbatim. |
| 14 | Real entry point `UnifiedSearchViewSet` | §2.2, §2.4 | `api_router.register(r"documents", UnifiedSearchViewSet)` (`src/paperless/urls.py:L32`); all evidence via `GET /api/documents/`. |
| 15 | Dual code path `_is_search_request` | §2.3, §9 | true only for `query`/`more_like_id` (`src/documents/views.py:L388-L392`); else `super().filter_queryset` (`src/documents/views.py:L410-L411`) = DB path. |
| 16 | `StandardPagination` (`LIMIT`/`OFFSET`) | §4 | `PageNumberPagination` subclass; `page_size=25`, `max_page_size=100000` (`src/paperless/views.py:L8-L11`). |
| 17 | Permission premise (R5) | §7 | **False**: no guardian, no `owner`, `IsAuthenticated` only, identical non-admin rows; perception attributed to per-user `SavedView` sort (`src/documents/models.py:L323`, `src/documents/models.py:L333`, `src/documents/models.py:L339`). |
| 18 | Frontend contract (R4) | §8 | Independent per-page requests; `ordering=-created` default; `DOCUMENT_SORT_FIELDS` excludes `id` (`src-ui/src/app/services/rest/document.service.ts:L16-L24`). |
| 19 | Whoosh sibling | §9 | `search_page` relevance-score paging (`src/documents/index.py:L203`, `src/documents/index.py:L210-L218`); out of scenario. |
| 20 | Engine dependence / SQLite vs PostgreSQL | §5.4, §5.5 | SQLite static walk stable (observed); PostgreSQL 13.23 (non-canonical backend) static walk **also** stable (observed) — EXPLAIN shows `.distinct()` supplies `id` in the `Sort Key`; the no-tiebreaker query is only latently stable. |
| 21 | The actual no-user-edit trigger | §6 | Reproduced through the real endpoint: background ingestion/removal shifts the result set under `LIMIT`/`OFFSET` → duplicate `[36]` / omission `[35]`. |

### 10.1 Recommendations (described only — **not** applied; this is a read-only investigation)

1. **Append a unique tiebreaker to the ordering** — e.g. `ORDER BY "created" DESC, "id" DESC`. This makes the order **total** so it no longer depends on the accidental `.distinct()`-supplies-`id` behaviour (§5.5) or on engine/plan, and it minimises the duplicate window under result-set shift (§6). Apply at `Document.Meta.ordering` and/or in the ordering backend.
2. **Prefer keyset/cursor pagination** over `LIMIT`/`OFFSET` for large, frequently-changing lists — it is immune to the background result-set-shift trigger (§6), which no tiebreaker can fully eliminate for offset pagination.
3. **Expose `id` (or a stable proxy) as a UI sort option** (`DOCUMENT_SORT_FIELDS`, `src-ui/src/app/services/rest/document.service.ts:L16-L24`) so users are not confined to tie-prone columns.
4. **Give `InboxFilter` its own `.distinct()`** (or otherwise not rely solely on the queryset-level `DISTINCT`, `src/documents/filters.py:L65-L66`) to remove the latent fragility in §3.3.

### 10.2 Clean-repo guarantee (verbatim `git status`)

All temporary observation scripts lived under the container's `/tmp` and the host `/tmp` — never in the repository tree; the `runserver` processes were stopped; the seeded data lived only in the **git-ignored** dev database (`data/db.sqlite3`) and the PostgreSQL cluster's own data directory (`/var/lib/postgresql/13/main`, outside the repository mount). The only change to the repository is this document.

**Authoritative per-file proof** — expanding untracked files individually with `--untracked-files=all` shows the single untracked path is exactly this answer document:

```text
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/paperless-ngx_542221a38dff.md
```

The same is confirmed by a pathspec-scoped status:

```text
$ git status --porcelain -- blitzy/documentation/paperless-ngx_542221a38dff.md
?? blitzy/documentation/paperless-ngx_542221a38dff.md
```

For completeness: the **default** `git status --porcelain` **collapses** the wholly-new top-level `blitzy/` directory into a single entry, because Git does not descend into an entirely-untracked directory:

```text
$ git status --porcelain
?? blitzy/
```

There are **no** tracked modifications, and the only file physically present under `blitzy/` is this document:

```text
$ git diff --stat
$ find blitzy -type f
blitzy/documentation/paperless-ngx_542221a38dff.md
$ find blitzy -type d
blitzy
blitzy/documentation
```

No existing source file was modified, created, or deleted; no dependency or configuration file was touched. Branch `blitzy-0fc6e9a3-9036-45e1-a62a-761d59ff2fdb`, HEAD `542221a38`, no submodules (`.gitmodules` absent).
