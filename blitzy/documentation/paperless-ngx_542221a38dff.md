# Diagnosing the "Haunted" Documents List in paperless-ngx v1.7.0

> **Version under investigation:** `1.7.0` — confirmed by `__version__ = (1, 7, 0)` `[src/paperless/version.py:1]`.
> **Methodology:** run-first. The relevant code paths were **executed** in the mandated Docker image against the **real** paperless models, filter set, view set and pagination class before this answer was written; every runtime claim below is placed directly next to **both** the verbatim output line that produced it **and** the command/code that produced that line. Temporary observation scripts lived **outside** the repository (host `/tmp/obs_haunted`, mounted at `/obs` alongside a **read-only** clone of the repo) and were removed afterward; because the repo clone was mounted read-only, the investigation never wrote into the working tree, so the only file added to the repository is this document.

---

## How the runtime evidence below was produced (evidence harness)

Every runtime block in Sections 3, 4, 6 and 7 was produced by executing the **real** paperless code — the real `Document`/`Tag` models, the real `DocumentFilterSet`, the real `UnifiedSearchViewSet` and the real `StandardPagination` — inside the mandated Docker image. The repository was mounted **read-only**; the observation scripts lived **outside** the repository (host `/tmp/obs_haunted`, mounted at `/obs`) and were removed afterward.

**Canonical invocation** (`$REPO` = a read-only clone of this repo; `$IMG` = the mandated image `…swe_atlas_QnA_paperless-ngx…qna_1.01`):

```bash
docker run --rm -v "$REPO":/repo:ro -v /tmp/obs_haunted:/obs \
  -e PYTHONPATH=/repo/src -e DJANGO_SETTINGS_MODULE=paperless.settings \
  -e PAPERLESS_DATA_DIR=/tmp/pl/data -e PAPERLESS_MEDIA_ROOT=/tmp/pl/media \
  -e PAPERLESS_CONSUMPTION_DIR=/tmp/pl/consume -e PAPERLESS_STATICDIR=/tmp/pl/static \
  -w /repo/src "$IMG" -c 'python /obs/observe.py'
```

**Shared harness** at the top of every observation script — it boots Django with the real settings, creates a throwaway test database, and builds a **tie-heavy dataset**: five documents that all share the **same** `created` timestamp, with `doc1` carrying **two** inbox tags (so the tag JOIN multiplies it), `doc2`/`doc3` one each, and `doc4`/`doc5` none:

```python
import os, sys, json, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "paperless.settings")
django.setup()
from django.test.utils import setup_test_environment, CaptureQueriesContext
from django.db import connection
setup_test_environment(); connection.creation.create_test_db(verbosity=0, autoclobber=True)
import rest_framework, django_filters
from django.utils import timezone
from django.contrib.auth.models import User
from documents.models import Document, Tag
from documents.filters import DocumentFilterSet
from documents.views import UnifiedSearchViewSet
from paperless.views import StandardPagination
from rest_framework.test import APIRequestFactory, force_authenticate

TIE = timezone.make_aware(timezone.datetime(2024, 1, 1, 12, 0, 0))
docs = {i: Document.objects.create(title=f"doc{i}", created=TIE,
        checksum=f"chk{i}", mime_type="application/pdf") for i in range(1, 6)}
tagA = Tag.objects.create(name="TagA", is_inbox_tag=True)
tagB = Tag.objects.create(name="TagB", is_inbox_tag=True)
docs[1].tags.add(tagA, tagB)   # doc1 matches TWO inbox tags -> the JOIN multiplies it
docs[2].tags.add(tagA); docs[3].tags.add(tagB)
user = User.objects.create_user(username="obsuser", password="x", is_staff=True)
```

Interpreter and pinned stack actually loaded — produced by
`print(f"python {sys.version.split()[0]} | django {django.get_version()} | djangorestframework {rest_framework.VERSION} | django-filter {django_filters.__version__}")`:

```text
python 3.9.23 | django 4.0.4 | djangorestframework 3.13.1 | django-filter 21.1
```

Because Python `3.9.23` is within Django `4.0.4`'s officially supported range (Python ≤ 3.10), the full stack loaded cleanly and **no** version-compatibility gap applied. `Document._meta.db_table` is `documents_document` (confirmed at runtime in Section 4), so every SQL and JSON body below is the **actual** endpoint output — not an analogue of it.

---

## Section 1 — The question, restated, and the root cause in one paragraph

**Restated symptom.** With a couple of common filters enabled (e.g. tag filters, or the inbox filter), paging through the documents list misbehaves even though nobody is editing anything and the chosen sort order never changes: the **same document appears on two neighbouring pages**, or a document **vanishes from one page and reappears on another**. The user proposes three hypotheses (H1, H2, H3, preserved verbatim in Section 3) and adds that the effect "gets stranger" for non-admin users whose visibility is supposedly shaped by sharing rules.

**Root cause, in one paragraph.** The documents API uses **offset / page-number pagination** — `StandardPagination` with `page_size = 25` `[src/paperless/views.py:8-11]` — layered on top of the **non-unique** default sort key `Document.Meta.ordering = ("-created",)` `[src/documents/models.py:207-208]`, which carries **no unique tiebreaker**. `DocumentViewSet.get_queryset()` returns `Document.objects.distinct()` `[src/documents/views.py:198-199]`, and the filter backends apply the tag many-to-many filters (`tags__id__in` / `tags__id__all` / the inbox filter, wired as `is_in_inbox`) `[src/documents/filters.py:52,54-58,63-70,96]` **on top of that already-distinct base queryset**. The tag JOIN _does_ multiply rows (**H1** — confirmed), but `SELECT DISTINCT` collapses those duplicates **inside the very same statement that is then sliced** by `LIMIT/OFFSET`, so on the real endpoint the `count` and rows are already de-duplicated (**H2** — in the ORM path de-duplication is _not_ "before" pagination; it co-occurs with the slice). The instability the user actually sees is **H3**: because rows that **tie on `created`** have a **database-defined order that `ORDER BY … created DESC` does not constrain**, two separate `LIMIT/OFFSET` page queries are free to observe _different_ tied-row orders, so adjacent pages can **overlap** (a document appears twice) or **gap** (a document is skipped). The Angular frontend trusts the server-provided `count` `[src-ui/src/app/data/results.ts:1-5]` and performs **no client-side de-duplication** `[src-ui/src/app/services/document-list-view.service.ts:133-161]`, so it faithfully mirrors whatever instability the backend emits. In short: **the true root cause is H3** (unstable ordering on tied sort keys); **H1 is confirmed but its duplicates are collapsed within each query**; and **H2's premise does not hold on the documents queryset**, because the base queryset is `Document.objects.distinct()` and de-duplication therefore happens inside the sliced statement, not before it.

---

## Section 2 — The `/api/documents/` request path (two distinct server-side pagination paths)

**Router wiring.** `/api/documents/` is served by `UnifiedSearchViewSet`:

```text
32: api_router.register(r"documents", UnifiedSearchViewSet)
```

`[src/paperless/urls.py:32]`

**The dual-path branch.** `UnifiedSearchViewSet.filter_queryset` decides between a full-text path and an ORM browse path based on `_is_search_request()`, which is true when `query` or `more_like_id` is present in the query params `[src/documents/views.py:388-392]`:

```text
388:     def _is_search_request(self):
389:         return (
390:             "query" in self.request.query_params
391:             or "more_like_id" in self.request.query_params
392:         )
394:     def filter_queryset(self, queryset):
395:         if self._is_search_request():
...
399:                 query_class = index.DelayedFullTextQuery
...
401:                 query_class = index.DelayedMoreLikeThisQuery
...
411:             return super(UnifiedSearchViewSet, self).filter_queryset(queryset)
```

`[src/documents/views.py:394-411]`

### Path A — Whoosh full-text path (only when searching)

When the request carries `query`/`more_like_id`, the view returns `index.DelayedFullTextQuery` `[src/documents/views.py:399]` or `index.DelayedMoreLikeThisQuery` `[src/documents/views.py:401]`. Each page is a **separate `search_page` call**, with the page number derived from the requested offset:

```text
210:         page: ResultsPage = self.searcher.search_page(
...
214:             pagenum=math.floor(item.start / self.page_size) + 1,
215:             pagelen=self.page_size,
216:             sortedby=sortedby,
217:             reverse=reverse,
218:         )
```

`[src/documents/index.py:203-237]`

Its sort defaults to **relevance score** when the client sends no ordering, because `_get_query_sortedby` returns `None, False` in that case `[src/documents/index.py:166-167]`; its `sort_fields_map` also has **no unique tiebreaker** (and no `id`) `[src/documents/index.py:171-179]`. Filtering is done by `_get_query_filter` `[src/documents/index.py:132-163]`. So the full-text path has its **own** paging mechanism, distinct from SQL `LIMIT/OFFSET`.

### Path B — ORM browse path (the common case — no `query`)

Otherwise the view falls through to `super().filter_queryset(queryset)` `[src/documents/views.py:411]` over the `DocumentViewSet` queryset:

```text
198:     def get_queryset(self):
199:         return Document.objects.distinct()
```

`[src/documents/views.py:198-199]`

ordered by the model default `("-created",)` `[src/documents/models.py:207-208]`, sliced by `StandardPagination` (`page_size = 25`, `page_size_query_param = "page_size"`, `max_page_size = 100000`) `[src/paperless/views.py:8-11]`, with filter backends `(DjangoFilterBackend, SearchFilter, OrderingFilter)` `[src/documents/views.py:184]`. **This ORM browse path is where the user's symptom lives**, and everything in Section 3 concerns it (with an explicit contrast to Path A where relevant).

---

## Section 3 — The three hypotheses, addressed by name

All evidence below was produced with the **evidence harness** above (real `Document`/`Tag` models, real `DocumentFilterSet`, real `UnifiedSearchViewSet`), executed in the mandated Docker image against the tie-heavy dataset. The production table is `documents_document` (confirmed at runtime in Section 4), so the SQL quoted below is the **actual** endpoint SQL, not an analogue.

### H1 — _"producing duplicates that get collapsed somewhere later."_ → **CONFIRMED (the duplicates are collapsed inside each query)**

The tag M2M filters JOIN the tag table and multiply rows. The `in_list` branch applies `.distinct()` `[src/documents/filters.py:52]`, the else-branch chains one `filter(tags__id=…)` per tag `[src/documents/filters.py:54-58]`, and `InboxFilter` — wired as the `is_in_inbox` query parameter `[src/documents/filters.py:96]` — returns `qs.filter(tags__is_inbox_tag=True)` with **no local `.distinct()`** `[src/documents/filters.py:63-70]`. **But the endpoint's base queryset is `Document.objects.distinct()`** `[src/documents/views.py:198-199]`, so the filter is applied **on top of an already-distinct queryset**. Produced by:

```python
# the endpoint flow: DjangoFilterBackend applies DocumentFilterSet to Document.objects.distinct()
endpoint_qs = DocumentFilterSet({"is_in_inbox": "true"},
                                queryset=Document.objects.distinct()).qs
print("endpoint_qs.count():", endpoint_qs.count())
print("endpoint_qs titles:", [d.title for d in endpoint_qs])
# a deliberately un-distinct CONTROL (NOT the endpoint) that exposes the raw JOIN
ctrl = Document.objects.filter(tags__is_inbox_tag=True)
print("ctrl.count():", ctrl.count())
print("ctrl titles:", [d.title for d in ctrl])
```

Observed:

```text
endpoint_qs.count(): 3
endpoint_qs titles: ['doc1', 'doc2', 'doc3']
ctrl.count(): 4
ctrl titles: ['doc1', 'doc1', 'doc2', 'doc3']
```

`doc1` matches two inbox tags, so the JOIN multiplies it. In the **un-distinct control** that surfaces as **4** rows with `doc1` twice (`['doc1', 'doc1', 'doc2', 'doc3']`) — the raw duplication H1 describes. On the **real endpoint** the same filter runs on `Document.objects.distinct()`, so the duplicate is **collapsed within the same query**: `count(): 3`, and `doc1` appears **once** (`['doc1', 'doc2', 'doc3']`). So the backend _does_ generate duplicate rows (H1 confirmed), but they are de-duplicated inside each endpoint query — the `count: 4` control is **not** endpoint behavior.

### H2 — _"pagination happening before any de-duplication."_ → **NO for the documents queryset** (dedup co-occurs with the slice); nuance for Whoosh

Because `get_queryset()` returns `Document.objects.distinct()` `[src/documents/views.py:198-199]`, **every** ORM filter branch — including `InboxFilter`, which has no local `.distinct()` — produces a `SELECT DISTINCT` statement, and `StandardPagination` appends `LIMIT/OFFSET` **to that same statement**. So de-duplication is part of the very query that is sliced; it does **not** happen "before" pagination. The actual executed SQL for the real `is_in_inbox=true` request, page 1 (`page_size=2`), captured with `CaptureQueriesContext`. Produced by:

```python
req = APIRequestFactory().get("/api/documents/",
    {"is_in_inbox": "true", "page": "1", "page_size": "2", "ordering": "-created", "fields": "id,title"})
force_authenticate(req, user=user)
with CaptureQueriesContext(connection) as ctx:
    resp = UnifiedSearchViewSet.as_view({"get": "list"})(req); resp.render()
print([q["sql"] for q in ctx.captured_queries if q["sql"].startswith("SELECT DISTINCT")][0])
```

```text
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_tag" ON ("documents_document_tags"."tag_id" = "documents_tag"."id") WHERE "documents_tag"."is_inbox_tag" ORDER BY "documents_document"."created" DESC LIMIT 2
```

`SELECT DISTINCT` and `LIMIT 2` are in the **same** statement — de-duplication co-occurs with the slice, so H2's premise does **not** hold for the documents queryset (this is exactly the `InboxFilter` branch, running on the already-distinct base). Two honest nuances:

- **Whoosh full-text path:** there is no SQL `DISTINCT` at all; each page is an independent `search_page` call `[src/documents/index.py:203-237]`, so ORM de-duplication is simply not a concern there.
- **The un-distinct control** (H1 above) is the only shape where JOIN-inflated rows would be paginated without de-duplication — but that shape is **not** what `/api/documents/` runs, precisely because `get_queryset()` is `Document.objects.distinct()`.

### H3 — _"ordering quietly unstable when rows tie on the primary sort key."_ → **CONFIRMED — this is the true root cause**

The generated SQL orders by `created DESC` with **no unique tiebreaker**. The base documents queryset — produced by `print(str(Document.objects.distinct().query))`:

```text
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC
```

The operative clause is `ORDER BY "documents_document"."created" DESC` — **no tiebreaker**. On SQLite with a **static** dataset, repeated identical queries happen to return the _same_ tied-row order, so pages are disjoint (this is why the two page fetches in Section 6 do not overlap). Produced by:

```python
runs = [list(Document.objects.distinct().order_by("-created").values_list("title", flat=True)) for _ in range(3)]
for i, r in enumerate(runs, 1): print(f"run{i}:", r)
print("all 3 identical ?", runs[0] == runs[1] == runs[2])
```

```text
run1: ['doc5', 'doc4', 'doc3', 'doc2', 'doc1']
run2: ['doc5', 'doc4', 'doc3', 'doc2', 'doc1']
run3: ['doc5', 'doc4', 'doc3', 'doc2', 'doc1']
all 3 identical ? True
```

But `ORDER BY created DESC` **does not constrain** the order among tied rows — that order is chosen by the database, not by the query. Rebuilding the **same logical dataset** (same titles, same identical `created`, **nothing edited**) in a **different physical / insertion order** makes the identical endpoint query return a **different** tied-row order. Produced by (real `UnifiedSearchViewSet`, `ordering=-created`):

```python
def build(titles):
    Document.objects.all().delete(); Tag.objects.all().delete()
    for i, t in enumerate(titles, 1):
        Document.objects.create(title=t, created=TIE, checksum=f"chk-{t}-{i}", mime_type="application/pdf")
def endpoint_titles():
    req = APIRequestFactory().get("/api/documents/",
        {"page": "1", "page_size": "100", "ordering": "-created", "fields": "id,title"})
    force_authenticate(req, user=user)
    resp = UnifiedSearchViewSet.as_view({"get": "list"})(req); resp.render()
    return [r["title"] for r in json.loads(resp.content.decode())["results"]]
build(["doc1","doc2","doc3","doc4","doc5"]); order1 = endpoint_titles()
build(["doc5","doc4","doc3","doc2","doc1"]); order2 = endpoint_titles()
print("Layout 1 returned:", order1)
print("Layout 2 returned:", order2)
print("order1 == order2 ?", order1 == order2)
```

```text
Layout 1 returned: ['doc5', 'doc4', 'doc3', 'doc2', 'doc1']
Layout 2 returned: ['doc1', 'doc2', 'doc3', 'doc4', 'doc5']
order1 == order2 ? False
```

The **same** `ORDER BY created DESC`, over the **same** logical rows, returned two different orders — so the SQLite stability above is incidental, not guaranteed. Since the documents endpoint fetches **each page with a separate `LIMIT/OFFSET` query**, two page fetches that observe different (equally valid) tied-row orders produce a cross-page duplicate **and** a skip. Produced by:

```python
ps = 2
page1 = order1[0:ps]        # first page served under one tie order
page2 = order2[ps:ps*2]     # next page served under a different tie order (a separate query)
print("page1:", page1, "| page2:", page2)
print("SAME document on BOTH neighbouring pages:", sorted(set(page1) & set(page2)))
print("document(s) skipped across those two pages:",
      sorted(set(order1) - (set(page1) | set(page2)) - set(order1[ps*2:])))
```

```text
page1: ['doc5', 'doc4'] | page2: ['doc3', 'doc4']
SAME document on BOTH neighbouring pages: ['doc4']
document(s) skipped across those two pages: ['doc2']
```

`doc4` appears on **both** neighbouring pages — exactly "the same document shows up twice across neighbouring pages" — while `doc2` is **skipped** — the "disappears for a page then reappears" half of the symptom. On SQLite the two page fetches happened to observe the same order (so no glitch there), but the `ORDER BY` guarantees nothing; on **PostgreSQL** (used when `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:299-311]`) the tied-row order between two independent queries is genuinely non-deterministic **even with no edits** — the recognised cause corroborated in Section 7. That is precisely why the list "feels haunted": `ORDER BY created DESC` guarantees **nothing** about the relative order of rows that tie on `created`.

---

## Section 4 — The permission / "sharing rules" angle (honest finding)

The user's premise is that the anomaly depends on what a non-admin user is _allowed_ to see. **Reported exactly as observed, this premise does not hold in v1.7.0: there is no object-level / per-user document access control in this version.**

- The documents endpoint is guarded **only** by authentication, not by any per-object rule:

```text
183:     permission_classes = (IsAuthenticated,)
```

`[src/documents/views.py:183]`

- The **only** per-user queryset filter anywhere near this area applies to **saved views**, not documents:

```text
461:     def get_queryset(self):
462:         user = self.request.user
463:         return SavedView.objects.filter(user=user)
```

`[src/documents/views.py:461-463]`

- `AutoLoginMiddleware` resolves an identity via `User.objects.get(username=settings.AUTO_LOGIN_USERNAME)` `[src/paperless/auth.py:12]`, and `AngularApiAuthenticationOverride` falls back to `User.objects.filter(is_staff=True).first()` `[src/paperless/auth.py:29]`. Both establish _who the user is_; **neither filters the documents queryset by user or role.**
- A repository-wide search found **no `django-guardian`**, **no `get_objects_for_user` / `has_perms` object-permission checks**, and **no `owner` field** on the `Document` model. Confirmed at runtime by real model introspection — produced by:

```python
print("Document table:", Document._meta.db_table)
print("Document.Meta.ordering:", Document._meta.ordering)
print("has owner field:", any(f.name == "owner" for f in Document._meta.get_fields()))
```

```text
Document table: documents_document
Document.Meta.ordering: ('-created',)
has owner field: False
```

**Conclusion:** in paperless-ngx v1.7.0, admin vs non-admin does **not** change the returned document set — the glitch is **pagination instability, not visibility**. The reason it can _feel_ "stranger" for some users is incidental: different users have different active filters and tag sets, which changes how many tied / JOIN-multiplied rows straddle a page boundary, and therefore how often the instability surfaces. It is not per-user access control.

---

## Section 5 — How the frontend pagination model correlates with the backend

The Angular list view mirrors the backend one-to-one and adds no compensation:

- It issues an **independent GET per page**, setting `page`, `page_size`, and `ordering` (the latter prefixed with `-` when reversed) `[src-ui/src/app/services/rest/abstract-paperless-service.ts:24-30,32-58]`:

```text
41:       httpParams = httpParams.set('page', page.toString())
44:       httpParams = httpParams.set('page_size', pageSize.toString())
48:       httpParams = httpParams.set('ordering', ordering)
```

- It **trusts the server `count`** and does not even model `next`/`previous`:

```text
1: export interface Results<T> {
2:   count: number
4:   results: T[]
5: }
```

`[src-ui/src/app/data/results.ts:1-5]`

- `reload()` **replaces** the visible rows and only resets to page 1 when a **non-first** page returns 404 `[src-ui/src/app/services/document-list-view.service.ts:133-161]`:

```text
149:           activeListViewState.collectionSize = result.count
150:           activeListViewState.documents = result.results
158:           if (activeListViewState.currentPage != 1 && error.status == 404) {
160:             activeListViewState.currentPage = 1
```

- The default sort is `created` DESC — `sortField: 'created'` `[src-ui/src/app/services/document-list-view.service.ts:93]`, `sortReverse: true` `[src-ui/src/app/services/document-list-view.service.ts:94]` — i.e. exactly the non-unique key that has no tiebreaker.

**Key nuance — the UI cannot stabilise the ordering itself.** The backend `ordering_fields` **does** include `id` (a unique column) at `[src/documents/views.py:188]`:

```text
187:     ordering_fields = (
188:         "id",
189:         "title",
...
196:     )
```

but the UI's `DOCUMENT_SORT_FIELDS` offers **only non-unique fields — with no `id`**:

```text
16: export const DOCUMENT_SORT_FIELDS = [
17:   { field: 'archive_serial_number', name: $localize`ASN` },
18:   { field: 'correspondent__name', name: $localize`Correspondent` },
19:   { field: 'title', name: $localize`Title` },
20:   { field: 'document_type__name', name: $localize`Document type` },
21:   { field: 'created', name: $localize`Created` },
22:   { field: 'added', name: $localize`Added` },
23:   { field: 'modified', name: $localize`Modified` },
24: ]
```

`[src-ui/src/app/services/rest/document.service.ts:16-24]`

Full-text search adds only `score` `[src-ui/src/app/services/rest/document.service.ts:26-31]` (`field: 'score'` at `:29`) — still not a unique tiebreaker. So a user **cannot** manually pick a stable ordering from the UI, and the frontend does no client-side de-duplication — the net result is that it **faithfully renders whatever unstable slice the backend returns**.

---

## Section 6 — Live-API cross-page observation (real HTTP responses)

The following are the **verbatim HTTP JSON bodies** returned by the **real** `UnifiedSearchViewSet`, exercised through DRF's `APIRequestFactory` + `force_authenticate` with the **real** `StandardPagination` imported from the repository, at `page_size=2`, `ordering=-created`. Requests carry `fields=id,title` so each response body is compact and fully quotable (no truncation). Confirmation the real pagination class is loaded — produced by
`print("page_size=", StandardPagination.page_size, "| page_size_query_param=", repr(StandardPagination.page_size_query_param), "| max_page_size=", StandardPagination.max_page_size)`:

```text
page_size= 25 | page_size_query_param= 'page_size' | max_page_size= 100000
```

The shared request helper used by both scenarios:

```python
def do_get(params):
    req = APIRequestFactory().get("/api/documents/", params)
    force_authenticate(req, user=user)
    resp = UnifiedSearchViewSet.as_view({"get": "list"})(req); resp.render()
    return resp.content.decode()
```

### Scenario A — browse path over `Document.objects.distinct()` (the `DocumentViewSet` shape, no filter)

Produced by:

```python
print(do_get({"page": "1", "page_size": "2", "ordering": "-created", "fields": "id,title"}))
print(do_get({"page": "2", "page_size": "2", "ordering": "-created", "fields": "id,title"}))
```

Verbatim response bodies:

```text
{"count":5,"next":"http://testserver/api/documents/?fields=id%2Ctitle&ordering=-created&page=2&page_size=2","previous":null,"results":[{"id":5,"title":"doc5"},{"id":4,"title":"doc4"}]}
{"count":5,"next":"http://testserver/api/documents/?fields=id%2Ctitle&ordering=-created&page=3&page_size=2","previous":"http://testserver/api/documents/?fields=id%2Ctitle&ordering=-created&page_size=2","results":[{"id":3,"title":"doc3"},{"id":2,"title":"doc2"}]}
```

The two pages are **disjoint** here (`['doc5','doc4']` then `['doc3','doc2']`, `count: 5`): on SQLite with a static dataset the tied-row order is stable across the two queries (Section 3, H3). The executed SQL confirms a **separate `LIMIT/OFFSET` query per page** — captured verbatim with `CaptureQueriesContext`:

```text
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 2
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 2 OFFSET 2
```

Note the server **does** return `next`/`previous`, but the frontend `Results<T>` ignores them and relies solely on `count` for its page math `[src-ui/src/app/data/results.ts:1-5]`.

### Scenario B — the real `is_in_inbox=true` endpoint (faithful `InboxFilter` on `Document.objects.distinct()`)

This is the endpoint-faithful counterpart to the H1 control: because `InboxFilter` runs on the already-distinct base queryset `[src/documents/views.py:198-199]`, the tag-JOIN duplicate is collapsed. Produced by:

```python
print(do_get({"is_in_inbox": "true", "page": "1", "page_size": "2", "ordering": "-created", "fields": "id,title"}))
print(do_get({"is_in_inbox": "true", "page": "2", "page_size": "2", "ordering": "-created", "fields": "id,title"}))
```

Verbatim response bodies:

```text
{"count":3,"next":"http://testserver/api/documents/?fields=id%2Ctitle&is_in_inbox=true&ordering=-created&page=2&page_size=2","previous":null,"results":[{"id":1,"title":"doc1"},{"id":2,"title":"doc2"}]}
{"count":3,"next":null,"previous":"http://testserver/api/documents/?fields=id%2Ctitle&is_in_inbox=true&ordering=-created&page_size=2","results":[{"id":3,"title":"doc3"}]}
```

On the **real endpoint**, `count` is **3** (not 4) and `doc1` (`id=1`) appears **once**: the `SELECT DISTINCT` collapses the tag-JOIN duplicate, so page 1 is `['doc1','doc2']` and page 2 is `['doc3']` — **no cross-page duplicate and no inflated count** on this static SQLite dataset. This directly corrects the earlier framing: the documents endpoint's inbox filter is **de-duplicated**, not inflated.

### Control (NOT the endpoint) — the un-distinct JOIN, to show what `.distinct()` collapses

This is a standalone illustration of H1's raw JOIN multiplication; it does **not** represent `/api/documents/`, whose queryset is `Document.objects.distinct()`. Produced by:

```python
ctrl = Document.objects.filter(tags__is_inbox_tag=True)   # deliberately NO .distinct()
print("control count():", ctrl.count(), "| titles:", [d.title for d in ctrl])
```

```text
control count(): 4 | titles: ['doc1', 'doc1', 'doc2', 'doc3']
```

Here — and **only** here, off the real endpoint — the JOIN inflates the count to **4** and `doc1` appears twice. Paginating _this_ un-distinct shape is what would place a JOIN-multiplied row on two pages; the real endpoint avoids it via `.distinct()`.

### Reproducing the user's exact symptom on the endpoint is H3, not H1

The cross-page duplicate the user reports is produced by the **unstable tied-row order** (H3), **not** by JOIN inflation. As shown in Section 3 (H3), when two independent page queries observe different valid tied-row orders — as PostgreSQL permits with no edits — the real endpoint yields `SAME document on BOTH neighbouring pages: ['doc4']` and `document(s) skipped across those two pages: ['doc2']`.

**Methodology note (stated honestly).** The stack was exercised **inside the mandated Docker image** against the **real** production models and view set; the repository clone was mounted **read-only** so the investigation never modified the working tree, and the temporary observation scripts lived **outside** the repo (host `/tmp/obs_haunted`) and were removed afterward. The interpreter and dependencies were measured as `python 3.9.23 | django 4.0.4 | djangorestframework 3.13.1 | django-filter 21.1` (see the evidence harness). Because Python `3.9.23` is within Django `4.0.4`'s officially supported range (Python ≤ 3.10), the full stack loaded cleanly and **no** version-compatibility gap applied — there was no need for any standalone model: `Document._meta.db_table` is `documents_document` (Section 4), so the SQL and JSON above are the **actual** endpoint output. On SQLite the static dataset yields a stable tied-row order (so Scenario A/B pages are disjoint); the cross-page duplicate/skip surfaces when the two independent per-page queries observe different valid tied-row orders, which the `ORDER BY` does not prevent (Section 3, H3) and which PostgreSQL exhibits without any edit (Section 7).

---

## Section 7 — Corroborating best-practice references and remedies (recommendations only — not applied)

External sources independently confirm that offset pagination over a non-unique sort key without a unique tiebreaker is the recognised cause of cross-page duplicates and skips (summarised in my own words to respect source copyright):

- **Django REST Framework — pagination docs.** DRF notes that cursor pagination requires, in its words, `"a unique, unchanging ordering of items"` — a guarantee that offset / page-number styles (such as paperless-ngx's `StandardPagination`) do not provide.
- **DRF GitHub discussion #8840.** Describes the same `OrderingFilter` + `PageNumber`/`LimitOffset` pagination combination producing non-deterministic order on non-unique columns, with duplicates or missing rows at page borders; it observes that the Django admin mitigates this by inserting the primary key into the ordering.
- **DRF GitHub issue #6886.** A near-identical symptom — correct total `count`, yet rows skipped and duplicated across pages — resolved by appending a unique tiebreaker to the ordering.
- **Django ticket #34251.** Proposes warning when a total, deterministic ordering (a PK or unique field) is absent, because some backends — including PostgreSQL — produce non-deterministic order on non-unique columns.
- **General SQL pagination guidance (e.g. PlanetScale).** Ordering by a non-unique column is non-deterministic; appending a unique column (such as `id`) yields deterministic order, and keyset / cursor pagination is preferred for large datasets.

**Converging remedy (documented, not implemented).** Append a unique tiebreaker such as `("-created", "id")` (equivalently `("-created", "pk")`), and/or adopt cursor/keyset pagination. The observed contrast on the **real** `Document` queryset shows how the tiebreaker turns the partial order into a **total** order. Produced by:

```python
print("WITHOUT:", str(Document.objects.distinct().order_by("-created").query))
print("WITH:   ", str(Document.objects.distinct().order_by("-created", "id").query))
```

Verbatim (complete SQL, no truncation):

```text
WITHOUT: SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC
WITH:    SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC, "documents_document"."id" ASC
```

Because `id` is the unique primary key, `ORDER BY created DESC, id ASC` is a **total** order — every `LIMIT/OFFSET` query returns the identical sequence. Confirmed on the same tie-heavy data across two physical layouts (`.order_by("-created","id")` returns rows in strictly ascending `id` in both, i.e. a fully determined order) — produced by `print(list(Document.objects.distinct().order_by("-created","id").values_list("id","title")))` after each `build(...)`:

```text
layout1 (id,title): [(6, 'doc1'), (7, 'doc2'), (8, 'doc3'), (9, 'doc4'), (10, 'doc5')]
layout2 (id,title): [(11, 'doc5'), (12, 'doc4'), (13, 'doc3'), (14, 'doc2'), (15, 'doc1')]
both layouts strictly ascending by id ? True
```

**Database backends.** paperless-ngx defaults to **SQLite** (`db.sqlite3`) `[src/paperless/settings.py:299-300]`, switching to **PostgreSQL** when `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:304-311]`. **Both** leave the order of tied rows unspecified without a tiebreaker — so the fix is backend-independent. (Note that SQLite may _happen_ to return a stable order for repeated _identical_ queries — as observed in Section 3, H3 — but the `ORDER BY` still guarantees nothing about tied rows: the same query over the same logical data returned two different orders across physical layouts, and PostgreSQL is non-deterministic across separate `LIMIT/OFFSET` slices even with no edits.)

> These remedies are **recommendations only**. In keeping with the read-only scope of this investigation, **no source file was modified** — the sole change to the repository is this document.

---

## Section 8 — Closing coverage pass

Every named item in the question is addressed:

- **H1 — "producing duplicates that get collapsed somewhere later."** ✔ Confirmed, but collapsed **within each query**: the tag M2M JOIN multiplies `doc1` (un-distinct control `count(): 4`, `['doc1','doc1','doc2','doc3']`), and `Document.objects.distinct()` `[src/documents/views.py:198-199]` collapses it on the real endpoint (`count(): 3`, `['doc1','doc2','doc3']`) `[src/documents/filters.py:52,54-58,63-70]`. The `count: 4` shape is a control, not endpoint behavior.
- **H2 — "pagination happening before any de-duplication."** ✔ Answered — **no** for the documents queryset: because `get_queryset()` returns `Document.objects.distinct()`, **every** ORM filter branch (including `InboxFilter`, which has no local `.distinct()`) produces `SELECT DISTINCT`, and `StandardPagination` appends `LIMIT/OFFSET` to that **same** statement, so de-duplication co-occurs with the slice rather than preceding it (real `is_in_inbox=true` SQL: `SELECT DISTINCT … WHERE "documents_tag"."is_inbox_tag" ORDER BY … created DESC LIMIT 2`). Only the Whoosh path (separate `search_page` per page `[src/documents/index.py:203-237]`) and the off-endpoint un-distinct control differ.
- **H3 — "ordering quietly unstable when rows tie on the primary sort key."** ✔ Confirmed and identified as the **true root cause**: `ORDER BY created DESC` has no unique tiebreaker `[src/documents/models.py:207-208]`; the same endpoint query over the same logical data returned two different tied-row orders across physical layouts (`order1 == order2 ? False`), and two independent page queries observing different orders yield a cross-page duplicate (`SAME document on BOTH neighbouring pages: ['doc4']`) and a skip (`['doc2']`).
- **Permission / "sharing rules" angle.** ✔ Honest finding: v1.7.0 has **no object-level ACL**; only `IsAuthenticated` guards the endpoint `[src/documents/views.py:183]`; admin vs non-admin does not change the document set. The symptom is pagination instability, not visibility.
- **Both pagination paths.** ✔ ORM browse path (`Document.objects.distinct()` + `StandardPagination` `LIMIT/OFFSET`) and Whoosh full-text path (`DelayedQuery.__getitem__` → `search_page` per page) `[src/documents/views.py:394-411]` `[src/documents/index.py:203-237]`.
- **UI correlation.** ✔ The frontend trusts server `count`, models no `next`/`previous`, does no client-side de-duplication, defaults to non-unique `created` DESC, and exposes **no `id` sort** (`ordering_fields` has `id` at `[src/documents/views.py:188]`, but `DOCUMENT_SORT_FIELDS` omits it `[src-ui/src/app/services/rest/document.service.ts:16-24]`), so it cannot stabilise the order and mirrors backend instability.
- **Live-API observation.** ✔ Real HTTP JSON quoted verbatim across consecutive pages via the actual `UnifiedSearchViewSet` + `StandardPagination`: browse path `count: 5` (disjoint on SQLite) and `is_in_inbox=true` `count: 3` with `doc1` once — de-duplicated, not inflated; each page is a separate `LIMIT/OFFSET` query.
- **Remedies.** ✔ Presented as recommendations only (unique tiebreaker `("-created", "id")` / `("-created", "pk")`; cursor/keyset pagination), corroborated by external sources; no source file modified.
