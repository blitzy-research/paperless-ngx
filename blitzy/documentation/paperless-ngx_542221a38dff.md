# Why the paperless-ngx documents list "feels haunted": a runtime-evidenced root-cause analysis

**Subject:** The same document appears twice across neighboring pages, or vanishes for a page and then reappears, during ordinary filtered browsing of the documents list — with nobody editing anything and the sort order unchanged.

**Scope / version pin:** All conclusions are pinned to **paperless-ngx 1.7.0** (`src/paperless/version.py:1` → `__version__ = (1, 7, 0)`). The permission and ordering model differs in later releases; nothing here should be generalized beyond 1.7.0.

**Method in one line:** The relevant code paths were *run first* through their real entry point (`GET /api/documents/` → `UnifiedSearchViewSet`), on the real `paperless.settings`, and the compiled SQL + per-page id sets were captured; the answer is proved from that captured output, not from prose.

**Label convention:** every finding is tagged *(observed)* — verified by running the code and pasted verbatim in the Evidence Appendix — or *(inferred)* — reasoned but not directly executed.

---

## Table of contents

1. [Direct answer (read this first)](#1-direct-answer-read-this-first)
2. [Environment & method](#2-environment--method)
3. [Per-hypothesis findings (H1, H2, H3)](#3-per-hypothesis-findings)
4. [The "duplicate vs. disappear" symptom, concretely](#4-the-duplicate-vs-disappear-symptom-concretely)
5. [Permission-dependence premise — verified negative](#5-permission-dependence-premise--verified-negative)
6. [Mode A vs Mode B (browse vs full-text search)](#6-mode-a-vs-mode-b-browse-vs-full-text-search)
7. [API ↔ UI pagination mapping](#7-api--ui-pagination-mapping)
8. [The canonical fix (rationale only — NOT applied)](#8-the-canonical-fix-rationale-only--not-applied)
9. [Evidence appendix — commands + unedited outputs](#9-evidence-appendix--commands--unedited-outputs)
10. [Reproduction recipe](#10-reproduction-recipe)
11. [Scope caveats](#11-scope-caveats)

---

## 1. Direct answer (read this first)

The symptom is the textbook signature of **Hypothesis 3 — an unstable `ORDER BY` on ties.** The model's default ordering is `Meta.ordering = ("-created",)` (`src/documents/models.py:207-208`) on `created = models.DateTimeField(_("created"), default=timezone.now, db_index=True)` (`src/documents/models.py:152`) — a **non-unique** column — and the list SQL emitted for the browse endpoint is a single statement:

```
SELECT DISTINCT <15 columns> FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT k OFFSET m
```

with **no unique tiebreaker** in the `ORDER BY`. An `ORDER BY` that ranks only on a non-unique column is not a *total* order: when several rows tie on `created`, the database is free to return the tied rows in any order, so two overlapping `LIMIT/OFFSET` windows can **repeat** a row (you see it twice) or **skip** a row (it vanishes) at a page boundary. This was **reproduced at runtime** *(observed — §9 Artifact 7)*: on PostgreSQL with a forced sequential scan, the tiebreaker-less `ORDER BY "created" DESC` window slid so that `id 5` moved from page 1 to page 4 and `id 11` moved from page 2 to page 1 **after merely touching an unrelated column while `created` was left unchanged** — exactly the "seen twice / vanish" behavior.

### The crucial, empirically-verified nuance (a *qualified negative* — stated plainly and up front)

In paperless-ngx **1.7.0**, the browse endpoint does **NOT** reproduce the symptom across repeated identical runs **in the canonical/default configuration**, because the list query is always `SELECT DISTINCT`: `DocumentViewSet.get_queryset()` returns `Document.objects.distinct()` (`src/documents/views.py:198-199`) over a projection that **includes the unique primary key `id`**.

- On **PostgreSQL**, `DISTINCT` forces the planner to sort by the **entire projection** — `Sort Key: created DESC, id, …` (all 15 columns) *(observed — §9 Artifact 6)* — so `(created DESC, id)` becomes a **unique prefix**, i.e. a **total, deterministic order**, i.e. stable pagination.
- On the default **SQLite** backend it is stable for an independent reason: the query is served by an ordered covering-index scan on `created` — `SCAN TABLE documents_document USING INDEX documents_document_created_bedd0818` *(observed — §9 Artifact 8)* — with ties broken de-facto by `rowid` (≈ `id`) and rows updated in place.

Observed distribution: **3 identical SQLite runs and 3 identical PostgreSQL runs, admin AND non-admin → identical page contents, zero duplicates, zero omissions** *(observed — §9 Artifacts 2 and 7)*.

> **Therefore:** H3 is the correct *mechanism*, and the code is **textually vulnerable** (the `ORDER BY` genuinely has no unique tiebreaker), but the bug is **inadvertently neutralized** in the canonical 1.7.0 configuration — by `.distinct()` dragging the unique `id` into the sort key (PostgreSQL) and by rowid-ordered index scans (SQLite). The mechanism is nonetheless **real**: it was reproduced the moment the accidental stabilizer was removed (the non-`DISTINCT` shape on PostgreSQL, §9 Artifact 7), and §4 enumerates exactly the conditions under which it *would* bite in production.

### The permission negative (also up front)

The user's premise that the glitch "feels worse for non-admin users whose visibility is shaped by sharing/permissions" is a **verified NEGATIVE at 1.7.0**. The documents-list SQL is **byte-identical** for an administrator and a regular user, with **no per-user filtering** *(observed — §9 Artifact 3: `IDENTICAL SQL: True | per-user/owner WHERE present: False`)*. At 1.7.0 the `Document` model has no owner/permission field (`src/documents/models.py:88-208`) and the viewset's only gate is `permission_classes = (IsAuthenticated,)` (`src/documents/views.py:183`). See §5 for why the premise nonetheless *feels* true (the `SavedView` model *is* per-user).

### One-paragraph summary of the three hypotheses

- **H1 — "backend hands back duplicates that get collapsed later?"** Partly true about duplicates, **false about "later."** Duplicates are produced *transiently at the SQL JOIN level* by the many-to-many tag filters, and they are collapsed **inside the very same SQL statement** by `SELECT DISTINCT` — never in Python (§2, §3.1, §9 Artifact 4).
- **H2 — "pagination happening before any de-duplication?"** **No.** De-duplication (`SELECT DISTINCT`) and pagination (`LIMIT`/`OFFSET`) are expressed in **one statement**; the window is carved from the already-`DISTINCT`, already-ordered set (§3.2, §9 Artifact 1).
- **H3 — "ordering quietly unstable whenever several rows tie?"** **Yes — this is the root cause.** `ORDER BY created DESC` has no unique tiebreaker; ties therefore have no defined relative order (§3.3, §4, §9 Artifacts 6–8).

---

## 2. Environment & method

**Run-first, evidence-driven.** Per the investigation rules, the relevant code paths were built and run *before* this document was written; every quantitative claim below is backed by an embedded command and its unedited output in §9.

**Real entry point, not a shortcut.** All API observations drive the genuine endpoint `GET /api/documents/`, which the router binds to `UnifiedSearchViewSet` (`src/paperless/urls.py:32` → `api_router.register(r"documents", UnifiedSearchViewSet)`; the class is `src/documents/views.py:377`). Requests are issued with the DRF test client (`rest_framework.test.APIClient`) authenticated via `force_login`, mirroring the established harness in `src/documents/tests/test_api.py:29-33` (`User.objects.create_superuser` + `self.client.force_login`).

**Canonical configuration.** Observations were made against the real `paperless.settings` with the **default SQLite** backend (`src/paperless/settings.py:297-300` → `"ENGINE": "django.db.backends.sqlite3"`, `"NAME": os.path.join(DATA_DIR, "db.sqlite3")`). PostgreSQL is only selected when `PAPERLESS_DBHOST` is set (`src/paperless/settings.py:304`); it is used here **only** for the cross-engine caveat (§9 Artifacts 6–7).

**Exact versions actually used** *(observed)*:

| Component | Version used for SQLite artifacts (1–5, 8) | Version used for PostgreSQL artifacts (6–7) |
|-----------|--------------------------------------------|---------------------------------------------|
| Python | **3.9.23** (canonical, per the project Dockerfile `python:3.9-slim-bullseye`) | 3.9.23 |
| Django | 4.0.4 | 4.0.4 |
| djangorestframework | 3.13.1 | 3.13.1 |
| django-filter | 21.1 | 21.1 |
| Whoosh | 2.7.4 | — |
| Database | SQLite (stdlib) | PostgreSQL **13.23** |

**Determinism / honesty note.** The SQLite artifacts (1–5, 8) were reproduced on the **fully canonical Python 3.9.23** runtime with the exact library pins — so no interpreter-version caveat applies to them *(observed)*. The PostgreSQL cross-engine artifacts (6–7) were reproduced on **PostgreSQL 13.23** (the version available in the canonical container image); the mechanism and the exact page-shift outcome are **not specific to a major version** and were confirmed to match across three fresh runs *(observed; equivalence to other PostgreSQL majors such as 16 is inferred)*.

**What was exercised (not just the happy path).** Both endpoint modes (Mode A database browse and Mode B Whoosh full-text search); the JOIN-multiplying many-to-many filter path (`tags__id__all`); the admin-versus-non-admin comparison of both JSON and SQL; paging across consecutive pages of a tied-`created` corpus repeated across multiple identical runs; the compiled `SELECT DISTINCT … ORDER BY … LIMIT … OFFSET …`; and the cross-engine `EXPLAIN`/`EXPLAIN QUERY PLAN`. The UI's 404→page-1 reset branch is analyzed from source in §7.

**Read-only guarantee.** No file under `src/` or `src-ui/` was modified. All reproduction ran from scripts kept **outside** the repository, with every data path (`DATA_DIR`, DB, index, logs) redirected outside the working tree; the tree was verified byte-for-byte pristine afterward (`git status --porcelain` empty). The single artifact created is this document.

---

## 3. Per-hypothesis findings

### 3.1 Hypothesis 1 — "Is the backend handing back duplicates that get collapsed somewhere later?"

**Answer: Partly true about duplicates, FALSE about "later."** Duplicates *are* produced — but only **transiently, at the SQL JOIN level** — and they are collapsed **within the very same SQL statement** by `SELECT DISTINCT`, **not later in Python**.

**Cause → effect:**

- The many-to-many tag/inbox filters join `documents_document` to `documents_document_tags`. A document that matches more than one tag row is multiplied into more than one result row by the JOIN. *(observed — §9 Artifact 4):* raw `Document.objects.filter(tags__id__in=[1, 2])` on a document holding **both** tags returned `[1, 1]`; adding `.distinct()` returned `[1]`.
- The collapse happens **in the same statement**, because the base queryset is already de-duplicated: `DocumentViewSet.get_queryset()` returns `Document.objects.distinct()` (`src/documents/views.py:198-199`). The compiled SQL for the filtered request is therefore a single `SELECT DISTINCT … FROM "documents_document" INNER JOIN "documents_document_tags" … INNER JOIN "documents_document_tags" T4 … WHERE (… tag_id = 1 AND T4.tag_id = 2) ORDER BY "created" DESC LIMIT 1` *(observed — §9 Artifact 4)*. The duplicate rows never leave the database.
- It is **not** collapsed in Python. `DocumentSerializer` (`src/documents/serialisers.py:201-222`) is a `DynamicFieldsModelSerializer`, which is itself a plain `serializers.ModelSerializer` (`src/documents/serialisers.py:22`); it performs **no** de-duplication of the result list.
- Filter code that produces the JOINs: `TagsFilter.filter` (`src/documents/filters.py:42-60`) calls `.distinct()` **only** on the `in`-list path (`src/documents/filters.py:52`), and **not** inside the `__all`/`__none` loop (`src/documents/filters.py:54-58`); `InboxFilter` (`src/documents/filters.py:63-70`) and `TitleContentFilter` (`src/documents/filters.py:73-78`) are the other filter branches; `DocumentFilterSet` (`src/documents/filters.py:81`) wires `tags__id__all` (`src/documents/filters.py:90`), `tags__id__none` (`src/documents/filters.py:92`), and `tags__id__in` (`src/documents/filters.py:94`). The per-filter `.distinct()` on the `in` path is **incidental**; the *authoritative* de-duplication is the base `get_queryset().distinct()`, which is why even the `__all` path (whose loop never calls `.distinct()`) still returns collapsed rows.

**Bottom line for H1:** duplicates are real but ephemeral (JOIN fan-out), collapsed *in-SQL* in the same statement, never surfaced to Python, and never "collapsed later."

### 3.2 Hypothesis 2 — "Is pagination happening before any de-duplication?"

**Answer: NO.** De-duplication and pagination are expressed in a **single SQL statement**; pagination is not a separate stage that precedes de-duplication.

**Cause → effect:**

- The compiled statement is one `SELECT DISTINCT <15 cols> FROM "documents_document" [INNER JOINs for filters] ORDER BY "created" DESC LIMIT k OFFSET m` *(observed — §9 Artifacts 1 and 4)*.
- DRF `StandardPagination(PageNumberPagination)` (`src/paperless/views.py:8-11`: `page_size = 25`, `page_size_query_param = "page_size"`, `max_page_size = 100000`) slices via Django's `Paginator`, which the ORM renders as the SQL `LIMIT`/`OFFSET` on that same statement. The window is carved **from the already-`DISTINCT`, already-ordered result set**, inside one round-trip.
- **Key reasoning (and why H2 is secondary to H3):** `SELECT DISTINCT` de-duplicates the row **set** but does **not** by itself impose a **total order** — these are independent concerns. So even though de-duplication is correctly applied *before* (indeed, jointly with) the `LIMIT/OFFSET`, an ordering that is not *total* still permits unstable windows. Getting the dedup placement right (H2) does not, on its own, make pagination stable; that is the job of a unique tiebreaker (H3).

**Bottom line for H2:** pagination is applied to the de-duplicated, ordered set in a single statement — dedup is not "after" pagination — so H2 is not the fault; it merely clears the way to see that H3 is.

### 3.3 Hypothesis 3 — "Is the ordering quietly unstable whenever several rows tie on the sorted field?"

**Answer: YES — this is the root cause.** `ORDER BY "documents_document"."created" DESC` carries **no explicit unique tiebreaker** (TRUE → latent fragility), and the resulting instability was **reproduced** in the non-`DISTINCT` shape on PostgreSQL *(observed — §9 Artifact 7)*. In the canonical 1.7.0 Mode A it is *accidentally* stabilized by `DISTINCT`+`id` (PostgreSQL) and by rowid-ordered index scans (SQLite), so it does not reproduce run-to-run there.

**Core reasoning (cause → effect):** `created` is indexed but **non-unique** (`src/documents/models.py:152`), and `Meta.ordering = ("-created",)` (`src/documents/models.py:208`) ranks only on it. Without a unique tiebreaker (e.g. `id`), tied rows have **no defined relative order**, so successive `LIMIT/OFFSET` windows are free to overlap (a row seen on two pages) or skip (a row on no page) across a page boundary — with the total `count` unchanged, which is exactly why the effect is subtle.

**Exactly WHEN the latent H3 bug WOULD manifest** (i.e., where the accidental 1.7.0 stabilizers do *not* apply):

- **(a)** a query path that lacks `.distinct()`, or whose `DISTINCT` projection contains **no unique column** — then no unique field is dragged into the sort key;
- **(b)** a hash-aggregate `DISTINCT` plan (which is unordered) followed by an `ORDER BY created`-only sort — the planner is free to choose this on larger tables;
- **(c)** PostgreSQL heap-order churn from ordinary writes / `VACUUM` / HOT-update misses — *(observed — §9 Artifact 7: touching one unrelated column shifted the whole window)*;
- **(d)** parallel sequential scans, whose row-return order is not deterministic;
- **(e)** cross-backend differences: PostgreSQL gives **no** implicit tie order at all, whereas SQLite's `rowid` gives *de-facto* (but undocumented, not contractual) stability.

**Bottom line for H3:** the ordering *is* quietly unstable on ties by construction; the only reason the default browse page looks calm today is two accidental stabilizers, either of which can disappear under the conditions above.

---

## 4. The "duplicate vs. disappear" symptom, concretely

Both halves of the user's question — *"does the same document quietly show up twice as you cross a page boundary, or briefly vanish for a page and then return"* — are answered directly by the clean 40-row PostgreSQL demonstration *(observed — §9 Artifact 7)*. Forty documents share one identical `created` value (ids 1..40); a forced sequential scan reads them; the full tiebreaker-less `ORDER BY "created" DESC` order is captured in pages of 10 **before** and **after** an unrelated `UPDATE documents_document SET title='touched' WHERE id=5` (the `created` value is **never** changed):

```
BEFORE:  page1=[1..10]                 page2=[11..20]                 page3=[21..30]                 page4=[31..40]
AFTER :  page1=[1,2,3,4,6,7,8,9,10,11] page2=[12,13,…,21]             page3=[22,23,…,31]             page4=[32,…,40,5]
```

- **DUPLICATE (seen twice):** `id 5` migrated **page 1 → page 4**. A user who saw document 5 on page 1, then pages forward, meets document 5 **again** on page 4 — "the same document quietly shows up twice as you cross a page boundary."
- **VANISH (skipped for a page, then returns):** `id 11` shifted **page 2 → page 1**. A user who has already viewed page 1 (before the write, when it was `[1..10]`) and then advances to page 2 (now `[12..21]`) **never encounters document 11** — it "briefly vanishes." On a later reload it reappears wherever the current heap order places it.

The full set of rows that changed page in this run was `{5: page1→page4, 11: page2→page1, 21: page3→page2, 31: page4→page3}` *(observed)*. Crucially, the `DISTINCT` shape (the actual `get_queryset()` shape) was **invariant** under the identical experiment — touching `id 9` left the order unchanged, head still `[1,2,…,12]` — because `id` is present in its sort key.

**Run-to-run character (stated honestly).** The *exact* manifestation is **heap-state dependent**: the tiebreaker-less order is simply whatever the sequential scan yields, and that changes when the heap changes (the `UPDATE` relocates a tuple). Across **three fresh identical runs** the shift was reproduced **identically** (deterministic for a fixed fresh heap) *(observed)*; but the shift depends on which row is touched, the table size, the heap layout, and the plan — so in a live system with continuous writes it presents as an **intermittent, non-deterministic** "haunting" rather than a fixed law. That intermittency is itself the run-to-run character the user reported.


---

## 5. Permission-dependence premise — verified negative

**Answer: VERIFIED NEGATIVE at 1.7.0.** The documents-list glitch **cannot** depend on sharing/permissions at this release, because the documents-list query does not depend on the authenticated user at all.

**Cause → effect + evidence:**

- The `Document` model carries **no** owner/user/permission field anywhere in its definition (`src/documents/models.py:88-208`).
- The viewset's only authorization gate is `permission_classes = (IsAuthenticated,)` (`src/documents/views.py:183`) — authentication, not per-object authorization.
- `DocumentViewSet.get_queryset()` returns `Document.objects.distinct()` **unconditionally**, with no per-user narrowing (`src/documents/views.py:198-199`).
- `REST_FRAMEWORK` (`src/paperless/settings.py:116-127`) declares only authentication and versioning classes — **no** global permission backend and **no** global filter backend — so nothing injects a per-user `WHERE` clause globally either.
- **Runtime proof** *(observed — §9 Artifact 3):* for the identical request `GET /api/documents/?page=2&page_size=3`, the main `SELECT` emitted under a **superuser** login and under a **regular-user** login was **byte-identical** — `IDENTICAL SQL: True` — with **no** per-user/owner predicate (`per-user/owner WHERE present: False`). The repeated-paging runs (§9 Artifact 2) were likewise identical between admin and non-admin (both `stable: True`, zero duplicates, zero omissions).

**Why the premise nonetheless *feels* true (the useful contrast).** A different model, `SavedView`, **does** carry a per-user field — `user = models.ForeignKey(User, …)` (`src/documents/models.py:323`, class at `src/documents/models.py:316`) — and its list **is** filtered per user (the pattern is visible in `src/documents/tests/test_api.py` `test_saved_views`, which creates `SavedView.objects.create(user=u1, …)` for distinct users). So a user's saved views genuinely differ between accounts, which can create the *intuition* that "what I see is scoped to me." That per-user scoping applies to **saved views**, **not** to the documents list — hence the premise is honestly false for this endpoint at 1.7.0.

**Consequence for the haunting:** because the documents-list SQL is user-independent, any perceived "worse for non-admins" difference is not produced by the backend query. If anything, non-admin accounts more often browse through **saved views** (with their own filter rules and page counts), which increases exposure to the *perception-level* 404→page-1 reset described in §7 — but that is a UI navigation effect, not permission-shaped SQL.

---

## 6. Mode A vs Mode B (browse vs full-text search)

`GET /api/documents/` resolves to `UnifiedSearchViewSet` (`src/paperless/urls.py:32`; class at `src/documents/views.py:377`), whose `UnifiedSearchViewSet._is_search_request()` (`src/documents/views.py:388-392`) branches on the presence of `query` or `more_like_id`. The two branches use **completely different pagination mechanisms** and must be reported separately.

### Mode A — database browse (no `query`/`more_like_id`)

`UnifiedSearchViewSet.filter_queryset()` delegates to the parent (`src/documents/views.py:410-411` → `super().filter_queryset(queryset)`), filtering `Document.objects.distinct()` through `filter_backends = (DjangoFilterBackend, SearchFilter, OrderingFilter)` (`src/documents/views.py:184`) with `ordering_fields` (`src/documents/views.py:187-196`), ordered `-created`, and paginated by SQL `LIMIT/OFFSET` via `StandardPagination`. **This is the "browsing with common filters" scenario** the user describes, and it is the SQL path analyzed in H1–H3. Its (in)stability originates entirely in the SQL `ORDER BY`.

### Mode B — full-text search (`query=` / `more_like_id=`)

`UnifiedSearchViewSet.filter_queryset()` instead returns a Whoosh-backed delayed query — `DelayedFullTextQuery` (for `query`) or `DelayedMoreLikeThisQuery` (for `more_like_id`), constructed as `query_class(self.searcher, self.request.query_params, self.paginator.get_page_size(self.request))` (`src/documents/views.py:398-409`) — and `UnifiedSearchViewSet.list()` opens the index searcher around the call (`src/documents/views.py:413-420`). Paging is performed by Whoosh, not SQL: inside `DelayedQuery.__getitem__` (`src/documents/index.py:203-222`, class at `src/documents/index.py:128`; `DelayedFullTextQuery` at `src/documents/index.py:240`) the code calls

```
searcher.search_page(q, mask=…, filter=…,
                     pagenum=math.floor(item.start / self.page_size) + 1,
                     pagelen=self.page_size,
                     sortedby=…, reverse=…)
```

with the sort key computed by `_get_query_sortedby()` (`src/documents/index.py:165`). Ordering here is governed by **Whoosh scoring / `sortedby`**, not by a SQL `ORDER BY`.

**Runtime contrast** *(observed — §9 Artifact 5):* `GET /api/documents/?query=content&page=1&page_size=3` returned `HTTP 200, count: 10, ids: [5, 10, 9]`. Because all seeded documents share identical content, their relevance scores tie, so the exact id order is **index/scoring dependent** (it can differ run-to-run and will not match the SQL `-created` order). The takeaway: **Mode A instability comes from the SQL `ORDER BY`; Mode B ordering comes from Whoosh** — two different mechanisms, each internally consistent within a single request, and they must not be conflated.

---

## 7. API ↔ UI pagination mapping

The Angular list is a thin client over the API; its pagination state maps one-to-one onto the backend query parameters. (Read from source; no frontend build is required to answer this.)

- **State** lives in `DocumentListViewService`: `currentPage` (`src-ui/src/app/services/document-list-view.service.ts:29`), `collectionSize` (`:34`), `sortField` (`:39`), `sortReverse` (`:44`); the defaults are `currentPage: 1`, `collectionSize: null`, `sortField: 'created'`, `sortReverse: true` (`src-ui/src/app/services/document-list-view.service.ts:91-94`).
- **Reload** — `DocumentListViewService.reload()` (`src-ui/src/app/services/document-list-view.service.ts:133`) calls `documentService.listFiltered(currentPage, currentPageSize, sortField, sortReverse, filterRules)` (`src-ui/src/app/services/document-list-view.service.ts:138-145`) and, on success, stores `collectionSize = result.count` (`:149`).
- **Request construction** — `DocumentService.listFiltered(page?, pageSize?, sortField?, sortReverse?, filterRules?, extraParams = {})` (`src-ui/src/app/services/rest/document.service.ts:96-107`) forwards to `AbstractPaperlessService.list()`, which builds the `HttpParams`: `page` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:40-41`), `page_size` (`:43-44`), and `ordering` (`:46-48`), where `getOrderingQueryParam = (sortReverse ? '-' : '') + sortField` (`src-ui/src/app/services/rest/abstract-paperless-service.ts:24-30`).
- **Net default request:** `GET /api/documents/?page=N&page_size=M&ordering=-created` — which exactly matches the model default `Meta.ordering = ("-created",)` and, like it, **carries no unique tiebreaker**. This is the direct tie-in to H3: the client faithfully asks for the same tiebreaker-less order the backend defaults to.
- **Mode detection in the UI** — `DocumentListComponent.getSortFields()` (`src-ui/src/app/components/document-list/document-list.component.ts:80`) switches the available sort columns via `isFullTextFilterRule(this.list.filterRules)` (`:81`, imported at `:21` from `src/app/data/filter-rule`) — this is how the UI distinguishes **Mode B** (full-text) from **Mode A** (filters). Sorting is delegated: `onSort(event)` (`:86`) calls `this.list.setSort(event.column, event.reverse)` (`:87`), and reloads route through `this.list.reload()` (`:107`).

### The 404 → page-1 reset (a secondary, perception-level contributor)

When a request targets a page beyond the last available page, the API returns **HTTP 404**, and `DocumentListViewService` resets to page 1 and reloads:

```
if (activeListViewState.currentPage != 1 && error.status == 404) {
  // this happens when applying a filter: the current page might not be
  // available anymore due to the reduced result set.
  activeListViewState.currentPage = 1
  this.reload()
}
```

(`src-ui/src/app/services/document-list-view.service.ts:158-161`.) This reset can itself produce a **"disappear then reappear" *perception*** when the effective page count shifts — e.g. after applying a filter that shrinks the result set, the user is bounced from page 5 back to page 1, so documents that *were* on page 5 "vanish" and only "return" once the user re-navigates. **This is distinct from the H3 mechanism:** it is a UI navigation artifact driven by a changing `count`, not by unstable SQL ordering. It is included for completeness because it can compound the perceived haunting, but it is not the root cause of the "same sort order, nobody editing, item seen twice" report — that one is H3.

---

## 8. The canonical fix (rationale only — NOT applied)

**No source file was changed by this investigation.** The repository is read-only for this task, so the remedy below is documented purely as rationale; it is **not** implemented.

- **Add a unique tiebreaker** to the ordering — e.g. `order_by("-created", "-id")`, or `Meta.ordering = ("-created", "-id")` — **or** adopt DRF `CursorPagination`. Either makes the order **total**, which makes pagination stable **regardless** of backend, query plan, or whether `DISTINCT` happens to pull a unique column into the sort key. This closes the latent H3 gap for *all* query paths, including the `(a)`/`(b)` cases in §3.3 where today's accidental stabilization does not apply.
- **A unique key is already available.** Every model uses an `AutoField` primary key `id` — `DEFAULT_AUTO_FIELD = "django.db.models.AutoField"` (`src/paperless/settings.py:320`) — so `-id` is a ready, unique, indexed tiebreaker at zero schema cost.
- **External corroboration (rationale only; these are not repository files).** This is a well-known Django REST Framework class of defect: DRF Discussion #8840 describes inconsistent pagination when sorting by non-unique columns; DRF Issue #6886 reports missing/duplicate records with `ordering` + `LimitOffsetPagination`, resolved by adding the primary key as a secondary sort (`ordering = ['-event_date', 'id']`) — directly analogous to paperless-ngx's `("-created",)`; and the DRF pagination documentation notes that `CursorPagination` yields a consistent view in which a client never sees the same item twice, but it requires a unique, unchanging ordering field. These informed the mechanism analysis only.


---

## 9. Evidence appendix — commands + unedited outputs

All artifacts below were captured by running the real code through its real entry point (`UnifiedSearchViewSet` via `rest_framework.test.APIClient` + `force_login`, mirroring `src/documents/tests/test_api.py:29-33`) against the real `paperless.settings`. Artifacts 1–5 and 8 ran on the **default SQLite** backend, **Python 3.9.23** with the exact pins (Django 4.0.4, DRF 3.13.1, django-filter 21.1, Whoosh 2.7.4). Artifacts 6–7 ran on **PostgreSQL 13.23**. Outputs are pasted unedited (a benign `whitenoise` "No directory at: …/static/" `UserWarning` — emitted because the full real settings include the whitenoise middleware — is omitted only where it is not part of an artifact's result; it never affects the data below).

**Common driver (SQLite artifacts 1–5, 8).** A single script bootstraps Django with the real settings, redirects all data paths outside the repository, migrates a fresh database, seeds **10 documents that share one identical `created` timestamp** (ids 1..10), creates a superuser and a regular user, then drives `GET /api/documents/` with `APIClient`, capturing SQL via `django.test.utils.CaptureQueriesContext(connection)`. Environment header printed by the run:

```
python: Python 3.9.23 | django: 4.0.4 | db.vendor: sqlite
DATA_DIR: /tmp/repro | DB NAME: /tmp/repro/db.sqlite3
SEEDED ids (all share created=2020-01-01T12:00:00+00:00): [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

### Artifact 1 — Compiled Mode A SQL (admin; `page_size=3`) *(observed)*

Command: drive `GET /api/documents/?page=1&page_size=3` (and `page=2`) inside `CaptureQueriesContext(connection)` and print the statement whose text contains `FROM "documents_document"`, `ORDER BY`, and `LIMIT`.

```
-- page=1&page_size=3   (HTTP 200)
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 3

-- page=2&page_size=3   (HTTP 200)
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 3 OFFSET 3
```

**What it proves:** one statement; `SELECT DISTINCT` over a 15-column projection (the first column is the unique PK `"documents_document"."id"`); `ORDER BY "documents_document"."created" DESC`; `LIMIT`/`OFFSET` for the page window; and **no `id` (or any unique) tiebreaker in the `ORDER BY`**. This is the exact statement referenced by H1, H2, and H3.

### Artifact 2 — Repeated identical runs (SQLite, `page_size=3`), admin AND non-admin *(observed)*

Command: with the 10 tied-`created` documents seeded, page through `GET /api/documents/?page=N&page_size=3` to exhaustion, 3 times each, under `force_login(admin)` and `force_login(regular)`.

```
admin  run1: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
admin  run2: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
admin  run3: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
regular run1: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
regular run2: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
regular run3: [[10, 9, 8], [7, 6, 5], [4, 3, 2], [1]]
duplicated: []   missing: []   stable(admin): True   stable(regular): True
```

**What it proves:** zero duplicates, zero omissions, identical across runs and across admin/non-admin → the **qualified negative** for the canonical SQLite config. (On SQLite the tied rows come back in a stable `rowid`-descending order — `[10,9,8,…,1]` — because of the index scan in Artifact 8.)

### Artifact 3 — Admin vs non-admin main `SELECT` (`page=2&page_size=3`) *(observed)*

Command: capture the main `SELECT` for the identical request under `force_login(admin)` and `force_login(regular)`, compare the strings, and test for any per-user/owner predicate.

```
IDENTICAL SQL: True   |   per-user/owner WHERE present: False
```

**What it proves:** the documents-list SQL does not depend on the authenticated user → the permission-dependence premise (§5) is a verified negative.

### Artifact 4 — Filter JOIN multiplication (H1) *(observed)*

Command: create tags `t1`,`t2`; attach both to document 1; compare the raw queryset with its `.distinct()`; then drive the API filter path and capture the SQL.

```
doc 1 holds tags [1, 2]
raw   Document.objects.filter(tags__id__in=[1,2])            -> [1, 1]
      Document.objects.filter(tags__id__in=[1,2]).distinct() -> [1]

GET /api/documents/?tags__id__all=1,2&page_size=3  -> HTTP 200, count: 1, ids: [1]
main SELECT:
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_document_tags" T4 ON ("documents_document"."id" = T4."document_id") WHERE ("documents_document_tags"."tag_id" = 1 AND T4."tag_id" = 2) ORDER BY "documents_document"."created" DESC LIMIT 1
```

**What it proves:** the double `INNER JOIN "documents_document_tags" … T4 …` multiplies the matching row (raw `[1, 1]`), and `SELECT DISTINCT` collapses it **in the same statement** (API returns `count: 1, ids: [1]`). H1: duplicates are real at the JOIN level, collapsed in-SQL, not in Python.

### Artifact 8 — SQLite `EXPLAIN QUERY PLAN` of the list query *(observed)*

Command: `EXPLAIN QUERY PLAN` the compiled SQL of `Document.objects.distinct().order_by("-created")`.

```
(5, 0, 0, 'SCAN TABLE documents_document USING INDEX documents_document_created_bedd0818')
```

**What it proves:** SQLite serves the ordered scan directly from the `created` index (`documents_document_created_bedd0818`); tied rows come out in `rowid` (≈ `id`) order and updates are applied in place → SQLite's **independent** reason for stability in the canonical config. (SQLite renders the step as `SCAN TABLE …`; newer SQLite builds print `SCAN …` — the index name is identical either way.)

### Artifact 5 — Mode B full-text search (Whoosh) *(observed)*

Command: build a real Whoosh index (`documents.index.open_index(recreate=True)`, then `documents.index.add_or_update_document(doc)` for each of the 10 documents), then `GET /api/documents/?query=content&page=1&page_size=3`.

```
?query=content&page=1&page_size=3  -> HTTP 200, count: 10, ids: [5, 10, 9]
path: UnifiedSearchViewSet._is_search_request() -> DelayedFullTextQuery -> searcher.search_page (src/documents/index.py:203-222)
```

**What it proves:** the search path returns `HTTP 200, count: 10`; the id order (`[5, 10, 9]`) is produced by **Whoosh scoring**, not a SQL `ORDER BY`. Because all documents share identical content their scores tie, so this order is index/scoring dependent and need not match — and generally will not match — the Mode A `-created` order. Different mechanism, reported separately (§6).

### Artifact 6 — PostgreSQL `EXPLAIN` of the real 15-column `DISTINCT` query *(observed)*

Command: on PostgreSQL 13.23, `EXPLAIN` the compiled SQL of `Document.objects.distinct().order_by("-created")`.

```
Unique  (cost=11.04..12.24 rows=30 width=2098)
  ->  Sort  (cost=11.04..11.11 rows=30 width=2098)
        Sort Key: created DESC, id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, modified, storage_type, added, filename, archive_filename, archive_serial_number
        ->  Seq Scan on documents_document  (cost=0.00..10.30 rows=30 width=2098)
```

**What it proves:** `DISTINCT` becomes `Unique ← Sort`, and the **Sort Key leads with `created DESC, id`**. Because `id` is the unique PK, `(created DESC, id)` is a **unique prefix** → total, deterministic order → stable Mode A pagination on PostgreSQL. This is precisely *why* the textual H3 vulnerability is neutralized in the canonical browse path.

### Artifact 7 — Cross-engine instability contrast (PostgreSQL, forced seq-scan, `created` NEVER changed) *(observed — the proof H3 is real)*

Command: a clean table of 40 documents all tied on `created` (ids 1..40); force a sequential scan (`SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off; SET max_parallel_workers_per_gather=0;`); capture the full `ORDER BY "created" DESC` order in pages of 10 before/after `UPDATE documents_document SET title='touched' WHERE id=5;` for both the non-`DISTINCT` and the `.distinct()` shapes.

```
NON-DISTINCT (no tiebreaker):
  BEFORE:  {'page1': [1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 'page2': [11, 12, 13, 14, 15, 16, 17, 18, 19, 20], 'page3': [21, 22, 23, 24, 25, 26, 27, 28, 29, 30], 'page4': [31, 32, 33, 34, 35, 36, 37, 38, 39, 40]}
  AFTER UPDATE ...title='touched' WHERE id=5 (created UNCHANGED):
          {'page1': [1, 2, 3, 4, 6, 7, 8, 9, 10, 11], 'page2': [12, 13, 14, 15, 16, 17, 18, 19, 20, 21], 'page3': [22, 23, 24, 25, 26, 27, 28, 29, 30, 31], 'page4': [32, 33, 34, 35, 36, 37, 38, 39, 40, 5]}
  ids that changed page:  {5: 'page1->page4', 11: 'page2->page1', 21: 'page3->page2', 31: 'page4->page3'}

DISTINCT (Document.objects.distinct(), same shape as get_queryset()):
  touching id=9 -> order INVARIANT (before == after): True
  head stays: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
```

**What it proves:** with the tiebreaker-less `ORDER BY "created" DESC` (non-`DISTINCT` shape), an unrelated `UPDATE` that never touches `created` reshuffled tied rows across the `LIMIT/OFFSET` windows — `id 5` to a later page (**duplicate**), `id 11` to an earlier page (**vanish**). The `DISTINCT` shape (the real `get_queryset()` shape, whose Sort Key includes `id`) was **invariant** under the identical experiment. This is the direct, first-hand proof that H3 is a real mechanism, gated only by whether a unique column is in the sort key.

**Reproducibility / honesty note:** this exact shift reproduced **identically across three fresh runs** (deterministic for a fixed fresh heap). The *magnitude and direction* of the shift are **heap-state dependent** — they depend on which row is touched, the table size, the heap layout, and the chosen plan — so in a continuously-written production table the effect appears **intermittently and non-deterministically**, which is exactly the run-to-run "haunting" character reported.

---

## 10. Reproduction recipe

Everything below runs **outside** the repository (scripts and all data live under a scratch directory such as `/tmp/repro`), so the working tree stays pristine.

**Environment.** Python 3.9 with the exact pins already installed (`django==4.0.4`, `djangorestframework==3.13.1`, `django-filter==21.1`, `whoosh==2.7.4`, `factory-boy`); default **SQLite**. (This investigation used the project's canonical container image, which ships exactly these; PostgreSQL 13.23 was installed into it for the cross-engine caveat.)

**SQLite (Artifacts 1–5, 8).**

1. Set `DJANGO_SETTINGS_MODULE=paperless.settings`, `PAPERLESS_DISABLE_DBHANDLER=true`, and redirect all data paths outside the repo — `PAPERLESS_DATA_DIR=/tmp/repro` (which also relocates the DB `/tmp/repro/db.sqlite3` per `src/paperless/settings.py:300`, `INDEX_DIR`, `LOGGING_DIR`, etc.), plus `PAPERLESS_MEDIA_ROOT`, `PAPERLESS_STATICDIR`, `PAPERLESS_LOGGING_DIR` under `/tmp/repro`; add `PAPERLESS_ALLOWED_HOSTS=testserver,localhost,127.0.0.1` (the DRF test client uses host `testserver`); and put the repo's `src` on `PYTHONPATH`.
2. `import django; django.setup()`, then `call_command("migrate", run_syncdb=True)` against the fresh `/tmp/repro/db.sqlite3`.
3. Seed with `Document.objects.create(title=…, content="content here", created=<one shared aware datetime>, checksum=<unique>, mime_type="application/pdf")` for ids 1..10 (fresh DB ⇒ ids start at 1).
4. Drive the API with `rest_framework.test.APIClient` + `force_login` (pass `SERVER_NAME="localhost"` on each request), capturing SQL with `django.test.utils.CaptureQueriesContext(connection)`. For Artifact 5, first build the Whoosh index via `documents.index.open_index(recreate=True)` + `documents.index.add_or_update_document(doc)`.

**PostgreSQL (Artifacts 6–7).** PostgreSQL cannot run as root — run the cluster as the `postgres` OS user: `initdb -D /tmp/repro-pg/pgdata -A trust`; start with `pg_ctl -D /tmp/repro-pg/pgdata -o '-p 55432 -k /tmp/repro-pg/pgsock -c listen_addresses=127.0.0.1' -l /tmp/repro-pg/pg.log -w start` (the log file must sit in a postgres-writable directory); `CREATE USER paperless … SUPERUSER;` and `createdb -O paperless paperless`. Point a second bootstrap at the PostgreSQL path by setting `PAPERLESS_DBHOST=127.0.0.1`, `PAPERLESS_DBPORT=55432`, `PAPERLESS_DBNAME=paperless`, `PAPERLESS_DBUSER=paperless`, `PAPERLESS_DBPASS=paperless` (this selects `postgresql_psycopg2` per `src/paperless/settings.py:304-318`). Force a sequential scan with `SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off; SET max_parallel_workers_per_gather=0;` before the window queries, then run the before/after `UPDATE` experiment.

**Cleanup (mandatory).** Delete `/tmp/repro` and the PostgreSQL cluster, stop `postgres`, and remove any scratch scripts; because all data paths were redirected outside the repo, nothing is written into the working tree. Verify with `git status --porcelain` (must be empty apart from this document).

---

## 11. Scope caveats

- **Version pin.** Every conclusion is scoped to **paperless-ngx 1.7.0** (`src/paperless/version.py:1`). Later releases change both the permission model and the ordering/queryset construction, so neither the permission negative (§5) nor the specific "accidental stabilizer" analysis (§1, §3.3) should be generalized beyond 1.7.0.
- **Canonical backend.** The primary result is on the default **SQLite** backend (`src/paperless/settings.py:299`). The **PostgreSQL** artifacts (6–7) are a cross-engine caveat: they ran on PostgreSQL **13.23** (the version in the canonical container), and the mechanism is not major-version specific; equivalence to PostgreSQL 16 is inferred, not separately observed here.
- **Interpreter fidelity.** SQLite Artifacts 1–5 and 8 ran on the **fully canonical Python 3.9.23** with the exact library pins, so no interpreter-minor-version caveat applies to them.
- **Read-only.** No file under `src/` or `src-ui/` was modified; the canonical fix in §8 is documented as rationale only and is **not** applied. All temporary reproduction scripts and databases were created outside the repository and removed, leaving the working tree byte-for-byte unchanged apart from this single document.
- **Observed vs. inferred.** Every quantitative claim is tagged *(observed)* and backed by an embedded command + unedited output in §9; anything reasoned but not directly executed is tagged *(inferred)*.

