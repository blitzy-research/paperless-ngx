# Diagnosing the "Haunted" Documents List in paperless-ngx v1.7.0

> **Version under investigation:** `1.7.0` — confirmed by `__version__ = (1, 7, 0)` `[src/paperless/version.py:1]`.
> **Methodology:** run-first. The relevant code paths were **executed** in the mandated Docker image before this answer was written; every runtime claim below is placed directly next to the verbatim output line that produced it. Temporary observation scripts lived **outside** the repository (`/tmp/obs`) and were removed; the only file added to the repository is this document.

---

## Section 1 — The question, restated, and the root cause in one paragraph

**Restated symptom.** With a couple of common filters enabled (e.g. tag filters, or the inbox filter), paging through the documents list misbehaves even though nobody is editing anything and the chosen sort order never changes: the **same document appears on two neighbouring pages**, or a document **vanishes from one page and reappears on another**. The user proposes three hypotheses (H1, H2, H3, preserved verbatim in Section 3) and adds that the effect "gets stranger" for non-admin users whose visibility is supposedly shaped by sharing rules.

**Root cause, in one paragraph.** The documents API uses **offset / page-number pagination** — `StandardPagination` with `page_size = 25` `[src/paperless/views.py:8-11]` — layered on top of the **non-unique** default sort key `Document.Meta.ordering = ("-created",)` `[src/documents/models.py:207-208]`, which carries **no unique tiebreaker**. When common **tag many-to-many filters** are active (`tags__id__in` / `tags__id__all` / the inbox filter) `[src/documents/filters.py:52,54-58,63-70]`, the JOIN against the tag table **multiplies** rows, and `Document.objects.distinct()` `[src/documents/views.py:198-199]` is the point where those duplicates are collapsed. Because rows that **tie on `created`** have a **database-defined (unstable) order** across separate `LIMIT/OFFSET` page queries, adjacent pages can **overlap** (the same document appears twice) or **gap** (a document is skipped for a page). The Angular frontend trusts the server-provided `count` `[src-ui/src/app/data/results.ts:1-5]` and performs **no client-side de-duplication** `[src-ui/src/app/services/document-list-view.service.ts:133-161]`, so it faithfully mirrors whatever instability the backend emits. In short: **the true root cause is H3** (unstable ordering on tied sort keys), while **H1 is confirmed** (M2M JOIN duplicates, collapsed by `.distinct()`) and **H2 is only partially right** (in the ORM path de-duplication happens *inside* the same SQL statement that is sliced, so it does **not** happen "before" pagination).

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

All evidence below was captured by running the exact query shape (`Document.objects.distinct()`, `Meta.ordering = ("-created",)`, tag M2M JOIN) against a tie-heavy dataset — five documents that all share the same `created` timestamp, with `doc1` carrying two matching tags. The standalone model's table is named `obsapp_document`; **in production the table is `documents_document`**, but the SQL **shape** is identical (see the methodology note in Section 6).

### H1 — *"producing duplicates that get collapsed somewhere later."* → **CONFIRMED**

The tag M2M filters JOIN the tag table and multiply rows. The `in_list` branch applies `.distinct()` `[src/documents/filters.py:52]`, the else-branch chains one `filter(tags__id=…)` per tag `[src/documents/filters.py:54-58]`, and `InboxFilter` returns `qs.filter(tags__is_inbox_tag=True)` with **no `.distinct()`** `[src/documents/filters.py:63-70]`. The collapse point for the queryset is `Document.objects.distinct()` `[src/documents/views.py:198-199]`. Observed:

```text
rows WITHOUT distinct : ['doc1', 'doc2', 'doc1', 'doc3']
rows WITH   distinct : ['doc1', 'doc2', 'doc3']
count() WITHOUT distinct: 4 | count() WITH distinct: 3
```

`doc1` matches two tags, so the JOIN multiplies it to **4** rows; `.distinct()` collapses the result to **3**. So the backend **does** generate duplicate rows that are collapsed later — exactly H1.

### H2 — *"pagination happening before any de-duplication."* → **In the ORM path, NO** (dedup is inside the sliced statement); **nuance for other paths**

In the ORM path the generated SQL shows `SELECT DISTINCT` and `LIMIT` co-occurring in a **single** statement, so de-duplication is part of the very query that is then sliced — pagination does **not** happen "before" de-duplication here:

```text
SELECT DISTINCT "obsapp_document"."id", "obsapp_document"."title", "obsapp_document"."created" FROM "obsapp_document" ORDER BY "obsapp_document"."created" DESC LIMIT 3
```

Two honest nuances:
- **Whoosh full-text path:** there is no SQL `DISTINCT` at all; each page is an independent `search_page` call `[src/documents/index.py:203-237]`, so ORM de-duplication is simply not a concern there.
- **No-`distinct()` filter branches:** `InboxFilter` `[src/documents/filters.py:63-70]` (and the chaining all-tags branch `[src/documents/filters.py:54-58]`) do **not** de-duplicate at all. In those branches H2's premise is closest to reality, because there is no de-duplication to precede — the JOIN-inflated rows are paginated directly (demonstrated live in Section 6, Scenario B).

So H2's premise is only partially correct, and only meaningfully applies to the no-`distinct()` branches — not to the main `Document.objects.distinct()` browse queryset.

### H3 — *"ordering quietly unstable when rows tie on the primary sort key."* → **CONFIRMED — this is the true root cause**

The generated SQL orders by `created DESC` with **no unique tiebreaker** (same evidence line as H2 above: `ORDER BY "obsapp_document"."created" DESC`). With tied `created` values the order among tied rows is **database-defined** and therefore **unstable across separate `LIMIT/OFFSET` page queries**. Paging the un-distinct JOIN at `page_size=2` reproduces the user's exact symptom:

```text
full ordered JOIN result (no distinct): ['doc1', 'doc2', 'doc1', 'doc3']
page1 (offset0,limit2): ['doc1', 'doc2']
page2 (offset2,limit2): ['doc1', 'doc3']
SAME doc on BOTH neighbouring pages: ['doc1']
```

When a tied / JOIN-multiplied row straddles a page boundary, the same document (`doc1`) lands on **both** neighbouring pages — "the same document shows up twice across neighbouring pages." Symmetrically, another document is pushed past the boundary and can be **skipped** for a page — the "disappears for a page then reappears" half of the symptom.

**Direct demonstration that the order is genuinely unstable (not just theoretically).** Re-running the identical query shape produced a **different** tied-row interleaving while the deterministic facts (count `4→3`, `doc1` duplicated) stayed the same:

```text
rows WITHOUT distinct : ['doc1', 'doc2', 'doc3', 'doc1']
```

Same dataset, same `ORDER BY created DESC`, different row order among the ties. That is precisely why the glitch "feels haunted": `ORDER BY created DESC` guarantees **nothing** about the relative order of rows that tie on `created`.

---

## Section 4 — The permission / "sharing rules" angle (honest finding)

The user's premise is that the anomaly depends on what a non-admin user is *allowed* to see. **Reported exactly as observed, this premise does not hold in v1.7.0: there is no object-level / per-user document access control in this version.**

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

- `AutoLoginMiddleware` resolves an identity via `User.objects.get(username=settings.AUTO_LOGIN_USERNAME)` `[src/paperless/auth.py:12]`, and `AngularApiAuthenticationOverride` falls back to `User.objects.filter(is_staff=True).first()` `[src/paperless/auth.py:29]`. Both establish *who the user is*; **neither filters the documents queryset by user or role.**
- A repository-wide search found **no `django-guardian`**, **no `get_objects_for_user` / `has_perms` object-permission checks**, and **no `owner` field** on the `Document` model.

**Conclusion:** in paperless-ngx v1.7.0, admin vs non-admin does **not** change the returned document set — the glitch is **pagination instability, not visibility**. The reason it can *feel* "stranger" for some users is incidental: different users have different active filters and tag sets, which changes how many tied / JOIN-multiplied rows straddle a page boundary, and therefore how often the instability surfaces. It is not per-user access control.

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

The following uses the **actual** `StandardPagination` imported from the repository (`from paperless.views import StandardPagination`), exercised via DRF `APIRequestFactory` with `ordering=-created`, `page_size=2`. Confirmation that the real class was loaded:

```text
Imported REAL StandardPagination from repo: page_size= 25 | page_size_query_param= 'page_size' | max_page_size= 100000
```

### Scenario A — browse path with `Document.objects.distinct()` (the `DocumentViewSet` shape)

```text
GET /api/documents/?page=1&page_size=2&ordering=-created -> {"count": 5, "next": "http://testserver/api/documents/?ordering=-created&page=2&page_size=2", "previous": null, "results": [{"id": 1, "title": "doc1", ...}, {"id": 2, "title": "doc2", ...}]}
GET /api/documents/?page=2&page_size=2&ordering=-created -> {"count": 5, "next": "http://testserver/api/documents/?ordering=-created&page=3&page_size=2", "previous": "http://testserver/api/documents/?ordering=-created&page_size=2", "results": [{"id": 3, "title": "doc3", ...}, {"id": 4, "title": "doc4", ...}]}
server count field: 5
```

Note the server **does** return `next`/`previous`, but the frontend `Results<T>` ignores them and relies solely on `count` for its page math `[src-ui/src/app/data/results.ts:1-5]`.

### Scenario B — inbox-style tag filter WITHOUT `.distinct()` (mirrors `InboxFilter` `[src/documents/filters.py:63-70]`)

```text
GET page=1 -> {"count": 4, "next": "http://testserver/api/documents/?ordering=-created&page=2&page_size=2", "previous": null, "results": [{"id": 1, "title": "doc1", ...}, {"id": 2, "title": "doc2", ...}]}
GET page=2 -> {"count": 4, "next": null, "previous": "http://testserver/api/documents/?ordering=-created&page_size=2", "results": [{"id": 1, "title": "doc1", ...}, {"id": 3, "title": "doc3", ...}]}
page1 titles: ['doc1', 'doc2'] | page2 titles: ['doc1', 'doc3']
SAME document id on BOTH neighbouring pages: [1]
server count field (inflated by JOIN, no distinct): 4
```

This is the **live HTTP reproduction of the user's exact symptom**: document `id=1` (`doc1`) appears on **both** page 1 and page 2. Also note `count: 4` is **inflated** by the JOIN — there are only **3** distinct matching documents — because in this no-`distinct()` branch the count is computed on the un-distinct'd queryset.

**Honest note on intermittency (a second manifestation of the same instability).** The placement of the duplicate depends on the DB-defined tied-row order and is **not** controlled by the application. In a separate run of the same Scenario B, both copies of `doc1` landed on the *same* page instead, so no cross-page overlap appeared that time:

```text
page1 titles: ['doc1', 'doc1'] | page2 titles: ['doc2', 'doc3']
SAME document id on BOTH neighbouring pages: []
```

Same code, same data, different tie order — sometimes a **cross-page duplicate**, sometimes a **same-page duplicate**, and correspondingly a skipped document elsewhere. This is why the behaviour is intermittent and "feels haunted."

**Methodology note (stated honestly).** The pinned stack was installed and run **outside** the repository, purely for observation; the repository itself is unchanged (`git status --porcelain` remained empty throughout, and the repo was mounted read-only for the live-API run). The observation ran inside the user-mandated Docker image, where the interpreter and dependencies were measured as `python 3.9.23 | django 4.0.4` (with `djangorestframework==3.13.1`, `django-filter==21.1`). Because Python `3.9.23` is within Django `4.0.4`'s officially supported range (Python ≤ 3.10), the stack loaded cleanly and **no** version-compatibility gap applied in the canonical container. (The upstream plan had referenced a separate host at Python `3.12.3`; the authoritative observation reported here used the mandated container at `3.9.23`.) The query-shape reproduction used a **minimal standalone model** whose table is `obsapp_document`; the production table is `documents_document`, but the SQL **shape** — `SELECT DISTINCT … ORDER BY created DESC LIMIT/OFFSET` — is identical, and Scenario A/B additionally exercised the **real** `StandardPagination` class at the HTTP layer. The same behaviour would be observed by paging the live `/api/documents/?page=N&…` endpoint against a tie-heavy dataset and quoting the JSON `count`/`results` across consecutive pages.

---

## Section 7 — Corroborating best-practice references and remedies (recommendations only — not applied)

External sources independently confirm that offset pagination over a non-unique sort key without a unique tiebreaker is the recognised cause of cross-page duplicates and skips (summarised in my own words to respect source copyright):

- **Django REST Framework — pagination docs.** DRF notes that cursor pagination requires, in its words, `"a unique, unchanging ordering of items"` — a guarantee that offset / page-number styles (such as paperless-ngx's `StandardPagination`) do not provide.
- **DRF GitHub discussion #8840.** Describes the same `OrderingFilter` + `PageNumber`/`LimitOffset` pagination combination producing non-deterministic order on non-unique columns, with duplicates or missing rows at page borders; it observes that the Django admin mitigates this by inserting the primary key into the ordering.
- **DRF GitHub issue #6886.** A near-identical symptom — correct total `count`, yet rows skipped and duplicated across pages — resolved by appending a unique tiebreaker to the ordering.
- **Django ticket #34251.** Proposes warning when a total, deterministic ordering (a PK or unique field) is absent, because some backends — including PostgreSQL — produce non-deterministic order on non-unique columns.
- **General SQL pagination guidance (e.g. PlanetScale).** Ordering by a non-unique column is non-deterministic; appending a unique column (such as `id`) yields deterministic order, and keyset / cursor pagination is preferred for large datasets.

**Converging remedy (documented, not implemented).** Append a unique tiebreaker such as `("-created", "pk")`, and/or adopt cursor/keyset pagination. The observed contrast shows how a tiebreaker turns the partial order into a **total** order that is stable across successive `LIMIT/OFFSET` pages:

```text
SQL WITHOUT tiebreaker: SELECT DISTINCT ... ORDER BY "obsapp_document"."created" DESC LIMIT 3
SQL WITH    tiebreaker: SELECT DISTINCT ... ORDER BY "obsapp_document"."created" DESC, "obsapp_document"."id" ASC LIMIT 3
```

**Database backends.** paperless-ngx defaults to **SQLite** (`db.sqlite3`) `[src/paperless/settings.py:299-300]`, switching to **PostgreSQL** when `PAPERLESS_DBHOST` is set `[src/paperless/settings.py:304-311]`. **Both** leave the order of tied rows unspecified without a tiebreaker — so the fix is backend-independent. (Note that SQLite may *happen* to return a stable order for repeated *identical* queries, but the `ORDER BY` still guarantees nothing across different `LIMIT/OFFSET` slices or after data changes, as the second Scenario B manifestation in Section 6 demonstrates.)

> These remedies are **recommendations only**. In keeping with the read-only scope of this investigation, **no source file was modified** — the sole change to the repository is this document.

---

## Section 8 — Closing coverage pass

Every named item in the question is addressed:

- **H1 — "producing duplicates that get collapsed somewhere later."** ✔ Confirmed: the tag M2M JOIN multiplies rows (4 rows) and `Document.objects.distinct()` collapses them (3 rows) `[src/documents/filters.py:52,54-58,63-70]` `[src/documents/views.py:198-199]`.
- **H2 — "pagination happening before any de-duplication."** ✔ Answered: in the ORM browse path, `SELECT DISTINCT` and `LIMIT` are in one statement, so dedup precedes the slice (not "before"); the premise only fits the no-`distinct()` branches (e.g. `InboxFilter`) and does not apply to the Whoosh path.
- **H3 — "ordering quietly unstable when rows tie on the primary sort key."** ✔ Confirmed and identified as the **true root cause**: `ORDER BY created DESC` has no unique tiebreaker `[src/documents/models.py:207-208]`, so tied-row order is DB-defined and unstable → cross-page duplicates / skips (reproduced live).
- **Permission / "sharing rules" angle.** ✔ Honest finding: v1.7.0 has **no object-level ACL**; only `IsAuthenticated` guards the endpoint `[src/documents/views.py:183]`; admin vs non-admin does not change the document set. The symptom is pagination instability, not visibility.
- **Both pagination paths.** ✔ ORM browse path (`Document.objects.distinct()` + `StandardPagination` `LIMIT/OFFSET`) and Whoosh full-text path (`DelayedQuery.__getitem__` → `search_page` per page) `[src/documents/views.py:394-411]` `[src/documents/index.py:203-237]`.
- **UI correlation.** ✔ The frontend trusts server `count`, models no `next`/`previous`, does no client-side de-duplication, defaults to non-unique `created` DESC, and exposes **no `id` sort** (`ordering_fields` has `id` at `[src/documents/views.py:188]`, but `DOCUMENT_SORT_FIELDS` omits it `[src-ui/src/app/services/rest/document.service.ts:16-24]`), so it cannot stabilise the order and mirrors backend instability.
- **Remedies.** ✔ Presented as recommendations only (unique tiebreaker `("-created", "pk")`; cursor/keyset pagination), corroborated by external sources; no source file modified.
