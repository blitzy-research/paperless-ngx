# Why the paperless-ngx documents list "feels haunted": duplicates and disappearances across neighboring pages

**Source baseline commit:** `542221a38` — the paperless-ngx commit under investigation (baseline `HEAD` before this document was added).
**Investigation type:** read-only, run-first, evidence-grounded Q&A.
**Deliverable:** this document only. No existing source file was modified (complete cleanup and integrity proof in §12). This answer is the sole net-new file; once committed it becomes the branch `HEAD` as a `docs:` commit atop the baseline, so the pre-commit snapshot in §12 shows `HEAD = 542221a38` with this file still untracked.

---

## Evidence labels (used consistently throughout)

Every substantive claim carries exactly one of these labels, and the §11 ledger reconciles each back to its section:

- **[OBSERVED — HTTP]** — captured through the **real authenticated HTTP API** `GET /api/documents/` (the canonical entry point). This is the primary class of evidence.
- **[OBSERVED — runtime]** — captured at runtime through a real, non-HTTP path that *supports* an HTTP observation: the exact SQL Django emitted for a live routed request (from PostgreSQL statement logs), `EXPLAIN`/`EXPLAIN ANALYZE` on the live database, an ORM measurement at runtime, or a read-only replay on an isolated copy of the live database. Clearly secondary/supporting.
- **[STATICALLY VERIFIED]** — a code fact with a `file:line` reference and the exact class/method named; confirmed against the byte-identical source in the running container.
- **[INFERRED]** — a code- or documentation-derived expectation **not** reproduced at runtime here; labeled as such.

---

## 1. Question restated

The user's report, verbatim:

> "the documents list can feel haunted during normal browsing, where paging through the documents view with a couple of common filters enabled can make the same document show up twice across neighboring pages or disappear for a page and then come back even though nobody is editing anything and the sort order looks unchanged. It gets stranger when the viewer is not an all powerful admin and visibility is shaped by sharing rules, because the glitch seems to depend on what the user is allowed to see rather than what exists."

The user asked us to watch what `GET /api/documents/` actually returns across consecutive page requests and line that up with what the UI thinks pagination means, until the exact condition that destabilizes the list becomes clear. Three named hypotheses must each be answered **by name**:

- **H1 — Backend duplicates collapsed later:** *"Is the backend producing duplicates that get collapsed somewhere later?"*
- **H2 — Pagination before de-duplication:** *"is pagination happening before any de-duplication?"*
- **H3 — Unstable ordering on ties:** *"or is the ordering quietly unstable when multiple rows tie on the primary sort key?"*

Plus the **authorization premise**: does visibility "shaped by sharing rules" (object-level permissions) amplify the glitch at this commit?

---

## 2. TL;DR — direct answers

- **H3 — Unstable ordering on ties is the true root-cause *mechanism*, but at this commit the symptom does NOT reproduce during ordinary paged browsing on either default backend. [OBSERVED — HTTP negative + STATICALLY VERIFIED mechanism + OBSERVED — runtime demonstration]**
  The list is ordered by the **non-unique** `Document.Meta.ordering = ("-created",)` (`src/documents/models.py:L208`) over a `created` field that is indexed but **not unique** and has **no secondary tiebreaker** (`src/documents/models.py:L152`) — the exact precondition for tie instability. However, driving the real HTTP API page-by-page over an unchanged, tie-heavy dataset was **12/12 clean on default SQLite and 12/12 clean on PostgreSQL** (§4.2, §4.5), both unfiltered and with a common filter enabled. The reason is a subtle interaction that we confirmed at runtime: on **SQLite** both independent page requests happen to use the *same* query plan, so they resolve the tie identically (§4.3–§4.4); on **PostgreSQL** the `Document.objects.distinct()` in `get_queryset()` forces a sort over **all** output columns — which includes the unique `id` — producing a **total order** `(created DESC, id, …)` that is deterministic even under forced parallelism and `VACUUM FULL` (§4.5). The H3 mechanism is therefore **real but latent** here: it surfaces only when the two independent page requests resolve ties under *different* orders, which we demonstrated with a controlled, read-only plan divergence on an isolated copy of the database — producing exactly **id `66`–`70` duplicated and id `51`–`55` missing with `count` constant** (§4.4). This is reported honestly per the "reproduce, don't stabilize" rule: we did not manufacture a stabilized variant and call the behavior deterministic, nor did we mutate the live database to force the artifact.

- **H2 — Pagination does NOT happen before de-duplication; the framing is refuted. [OBSERVED — HTTP]**
  `DocumentViewSet.get_queryset()` returns `Document.objects.distinct()` (`src/documents/views.py:L198-199`), so `DISTINCT` is part of the **base queryset**. The SQL Django actually emitted for a live routed page-2 request (captured from PostgreSQL statement logs, §6) is a single statement `SELECT DISTINCT … ORDER BY "created" DESC LIMIT 15 OFFSET 25`, and `count` is `SELECT COUNT(*) FROM (SELECT DISTINCT …) subquery`. `DISTINCT` is applied to the row source **within the same statement** that carries `LIMIT/OFFSET` — de-duplication **precedes** the page slice.

- **H1 — Yes, the backend can produce duplicate rows (many-to-many JOIN fan-out), collapsed at the base queryset, not by the filters. [OBSERVED — HTTP + runtime]**
  A tag filter that matches two tags on one document JOINs the M2M tag tables and emits duplicate rows before de-duplication. Measured at runtime: the filter `?tags__name__icontains=invoice` produced **36 raw JOIN rows for 30 distinct documents**; the HTTP endpoint returned **30** (§5). The collapsing step is `get_queryset()`'s `Document.objects.distinct()` (`src/documents/views.py:L198-199`) — the individual filters emit no `DISTINCT`. So H1's "collapsed somewhere later" is accurate, but the collapse happens **before** pagination (§6), which is why fan-out is not the paging artifact.

- **Authorization premise — does NOT reproduce; the instability is permission-independent here. [OBSERVED — HTTP + STATICALLY VERIFIED]**
  Every documents endpoint enforces only `permission_classes = (IsAuthenticated,)` (`src/documents/views.py:L183`); there are **no object-level permissions** and **no `django-guardian`** dependency at this commit (§7). Running the identical two-page sweep as a **superuser** and as a **non-staff** user returned **identical** `count` and identical page-1/page-2 id sets on both backends, with an empty symmetric difference (§7). The "sharing rules shape what you see" scenario is not present in this code; the list behavior depends on tie-ordering, not on who is looking.

**One-line summary:** the ordering by a non-unique `-created` with no tiebreaker is the latent mechanism the user intuited (H3), but at commit `542221a38` the `.distinct()` side effect on PostgreSQL and single-plan stability on SQLite keep it from surfacing during ordinary browsing; H1 (fan-out) and H2 (dedup-before-pagination) are real but are *not* the cause; and the "sharing rules" premise does not exist in this code.

---

## 3. Environment & exact commands (canonical build → run → seed → observe)

All behavioral evidence was captured through the **real authenticated HTTP API** `GET /api/documents/`, served by Django's `runserver` (entry point `src/manage.py`), in the project's **default SQLite** configuration; a **PostgreSQL 13** production backend was additionally stood up for the tie-break comparison. Non-HTTP captures (routed SQL, `EXPLAIN`, ORM counts, isolated-copy replay) are labeled **[OBSERVED — runtime]** and never substitute for an HTTP observation.

### 3.1 Canonical runtime and versions [OBSERVED — runtime]

The canonical container image and interpreter/framework versions (run-first: the stack was started and exercised *before* any of this write-up):

```
$ docker inspect --format '{{.Config.Image}}' paperless-app
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01

$ docker exec paperless-app bash -lc 'python --version'
Python 3.9.23

$ docker exec paperless-app bash -lc "cd src && python -c 'import django,rest_framework,django_filters; print(\"django\",django.get_version()); print(\"drf\",rest_framework.VERSION); print(\"django_filter\",django_filters.VERSION)'"
django 4.0.4
drf 3.13.1
django_filter (21, 1)
```

These match the pinned dependencies in `requirements.txt` (django 4.0.4, djangorestframework 3.13.1, django-filter 21.1). The repository root is bind-mounted into the container as its working directory, so every command below uses **repository-relative paths** (e.g. `cd src`, `data/db.sqlite3`); no absolute container path is relied upon.

### 3.2 Default database is SQLite [STATICALLY VERIFIED + OBSERVED — runtime]

```
$ sed -n '299,302p' src/paperless/settings.py
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": os.path.join(DATA_DIR, "db.sqlite3"),
    },
}
```

No `PAPERLESS_DBHOST` is set by default, so the PostgreSQL branch (`src/paperless/settings.py:L304-318`) is not taken. Confirmed at runtime:

```
$ docker exec paperless-app bash -lc "cd src && python -c 'import sys,os; sys.path.insert(0,os.getcwd()); os.environ.setdefault(\"DJANGO_SETTINGS_MODULE\",\"paperless.settings\"); import django; django.setup(); from django.conf import settings; d=settings.DATABASES[\"default\"]; print(\"ENGINE=\"+d[\"ENGINE\"])'"
ENGINE=django.db.backends.sqlite3
```

The SQLite database file is the repository-relative `data/db.sqlite3`, which — together with `media/`, `*.log`, and `src-ui/dist/` — is git-ignored (`.gitignore`), so all runtime/seed state is outside the tracked repository (verified in §12).

### 3.3 The canonical entry point / invocation [OBSERVED — runtime]

The documents list is served by the real viewset through `runserver`. For the observations, the server was bound to **loopback only** (`127.0.0.1`) inside the container so nothing was exposed off-host, and reached via `docker exec` from inside the container:

```
# SQLite (default backend), loopback-only, no autoreload:
$ docker exec -d paperless-app bash -lc 'cd src && PAPERLESS_DISABLE_DBHANDLER=true python manage.py runserver 127.0.0.1:8000 --noreload'

# PostgreSQL (production backend) on a second port, same code path:
$ docker exec -d -e PAPERLESS_DBHOST=paperless-postgres paperless-app bash -lc 'cd src && PAPERLESS_DISABLE_DBHANDLER=true python manage.py runserver 127.0.0.1:8001 --noreload'
```

> `PAPERLESS_DISABLE_DBHANDLER=true` disables only the database-backed *logging* handler; it does not alter the documents API code path. `curl` is not installed in the container, so HTTP requests were issued with Python's standard-library `urllib` (real HTTP over the loopback socket to `runserver`).

### 3.4 `/api/documents/` routing, ordering, pagination, and authentication [STATICALLY VERIFIED]

The list endpoint is registered to `UnifiedSearchViewSet`, which subclasses `DocumentViewSet` and does **not** override the queryset, pagination, filter backends, or permissions — so the default (non-search) list inherits `DocumentViewSet`'s behavior exactly (the routed SQL in §6 confirms this empirically):

```
$ grep -n 'register(r"documents"' src/paperless/urls.py
32:api_router.register(r"documents", UnifiedSearchViewSet)

$ grep -n 'class UnifiedSearchViewSet\|class DocumentViewSet\|pagination_class = StandardPagination\|permission_classes = (IsAuthenticated,)\|def get_queryset\|Document.objects.distinct' src/documents/views.py
172:class DocumentViewSet(
182:    pagination_class = StandardPagination
183:    permission_classes = (IsAuthenticated,)
184:    filter_backends = (DjangoFilterBackend, SearchFilter, OrderingFilter)
198:    def get_queryset(self):
199:        return Document.objects.distinct()
377:class UnifiedSearchViewSet(DocumentViewSet):
```

- Default order: `Document.Meta.ordering = ("-created",)` (`src/documents/models.py:L208`); `created = models.DateTimeField(_("created"), default=timezone.now, db_index=True)` — indexed, **not** unique, no tiebreaker (`src/documents/models.py:L152`). `id` is an allowed ordering field (`src/documents/views.py:L188`) and an auto-increment integer PK (`DEFAULT_AUTO_FIELD`, `src/paperless/settings.py:L320`) — a natural monotonic tiebreaker the default ordering does not use.
- Pagination: `StandardPagination(PageNumberPagination)` with `page_size = 25` (`src/paperless/views.py:L8-11`) — offset (`LIMIT/OFFSET`) slicing.
- Authentication: `REST_FRAMEWORK` (`src/paperless/settings.py:L116`) enables `BasicAuthentication`; without credentials the endpoint returns `401` (i.e. `IsAuthenticated`). Trimmed to the auth-relevant response details only:

```
$ docker exec paperless-app bash -lc 'cd src && python -c "
import urllib.request, urllib.error, json
try:
    urllib.request.urlopen(\"http://127.0.0.1:8000/api/documents/?page=1\", timeout=15)
except urllib.error.HTTPError as e:
    print(\"HTTP status:\", e.code, e.reason)
    print(\"WWW-Authenticate:\", e.headers.get(\"WWW-Authenticate\"))
    print(\"body:\", json.load(e))
"'
HTTP status: 401 Unauthorized
WWW-Authenticate: Basic realm="api"
body: {'detail': 'Authentication credentials were not provided.'}
```

### 3.5 Tie-inducing seed dataset [OBSERVED — runtime]

The failure only manifests when tied `created` values straddle a page boundary, so the dataset was seeded (through the real app, on the git-ignored database) larger than one page (`page_size = 25`) with deliberate ties, plus **disposable** local users. The seed creates **40 documents**: 20 at timestamp `D0 = 2026-07-13 12:00` (newest) and 20 at `D1 = 2026-07-12 12:00`, so the `D1` tie block straddles the 25/26 boundary. Tags: `invoice` on **30** documents (deliberately > 25, so *filtered* browsing also spans a page boundary), `invoice-paid` on 6, `inbox` on 12. Disposable users: `probe_admin` (superuser) and `probe_viewer` (non-staff), with an obviously-disposable password; both are **deleted during cleanup** (§12).

```
$ docker exec paperless-app bash -lc 'cd src && python manage.py shell -c "
from documents.models import Document, Tag
from django.contrib.auth.models import User
from django.db.models import Count
print(\"total docs:\", Document.objects.count())
print(\"created distribution:\", list(Document.objects.values(\"created\").annotate(n=Count(\"id\")).order_by(\"-created\")))
for t in Tag.objects.all(): print(\"  tag\", repr(t.name), \"(id=%d)\"%t.id, \"on\", t.documents.count(), \"docs\")
"'
total docs: 40
created distribution: [{'created': datetime.datetime(2026, 7, 13, 12, 0, tzinfo=datetime.timezone.utc), 'n': 20}, {'created': datetime.datetime(2026, 7, 12, 12, 0, tzinfo=datetime.timezone.utc), 'n': 20}]
  tag 'invoice' (id=4) on 30 docs
  tag 'invoice-paid' (id=5) on 6 docs
  tag 'inbox' (id=6) on 12 docs
```

With `page_size = 25` and default `-created` ordering, page 1 = the 20 `D0` documents + 5 of the 20 `D1` documents; page 2 = the remaining 15 `D1` documents. The boundary at position 25/26 lands **inside the `D1` tie block** — exactly the condition H3 needs. *Which* of the tied `D1` rows land on page 1 versus page 2 is unspecified by SQL, and is the crux of H3.

---

## 4. Evidence for H3 — unstable ordering on ties

### 4.1 The effective order is the model default `-created` (non-unique, no tiebreaker) [STATICALLY VERIFIED]

The request omits `?ordering`, and `DocumentViewSet` sets no default `ordering` attribute, so `Document.Meta.ordering = ("-created",)` (`src/documents/models.py:L208`) governs. `created` is `models.DateTimeField(_("created"), default=timezone.now, db_index=True)` — indexed but **not** unique, with no secondary key (`src/documents/models.py:L152`). This is the precondition for H3: a sort key on which multiple rows can tie, with nothing to break the tie deterministically.

### 4.2 Repeated identical sweeps on default SQLite are stable — unfiltered AND filtered (honest negative) [OBSERVED — HTTP]

The sweep harness (`sweep.py`, reproduced in §A) fetches the authoritative universe once via `page_size=100000` (`StandardPagination.max_page_size`, no `LIMIT/OFFSET` boundary), then performs N **independent, unchanged** consecutive-page sweeps at `page_size=25` (following `next` until exhausted — exactly what the Angular page-number client does), and per sweep computes duplicates (an id on two pages), gaps (a universe id on no page), and extras. **Unfiltered**, 12 identical sweeps on the default SQLite backend:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8000 paperless-app bash -lc 'cd src && python /tmp/sweep.py "page_size=25&ordering=-created" 12 "SQLite-unfiltered-default-created"'
=== SWEEP LABEL: SQLite-unfiltered-default-created ===
query: page_size=25&ordering=-created
universe: count=40 n_ids=40 ids=[50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 70, 69, 68, 67, 66, 65, 64, 63, 62, 61, 60, 59, 58, 57, 56, 55, 54, 53, 52, 51]
sweep 01: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 02: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 03: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 04: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 05: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 06: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 07: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 08: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 09: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 10: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 11: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 12: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
DISTRIBUTION: 12/12 clean, 0/12 artifact
```

**With a common filter enabled** — the `invoice` tag, which is on 30 documents (> 25, so the filtered result spans a page boundary too), 12 identical sweeps:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8000 paperless-app bash -lc 'cd src && python /tmp/sweep.py "page_size=25&ordering=-created&tags__id__all=4" 12 "SQLite-filtered-invoice-tags__id__all"'
=== SWEEP LABEL: SQLite-filtered-invoice-tags__id__all ===
query: page_size=25&ordering=-created&tags__id__all=4
universe: count=30 n_ids=30 ids=[31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65]
sweep 01: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 02: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 03: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 04: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 05: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 06: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 07: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 08: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 09: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 10: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 11: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 12: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
DISTRIBUTION: 12/12 clean, 0/12 artifact
```

**Distribution: 12/12 clean unfiltered and 12/12 clean filtered.** On the default, single-process, static SQLite backend the artifact does **not** reproduce by mere repetition — even with a common filter enabled and ties straddling the boundary. This is reported honestly; §4.3–§4.4 show precisely *why* it is only coincidentally stable and how the underlying order is not guaranteed.

### 4.3 Why SQLite is coincidentally stable — and why the tie order is NOT guaranteed [OBSERVED — runtime]

Both independent page requests are satisfied by the **same** plan (an index walk over `documents_document_created_bedd0818`), so they resolve the tie identically. Crucially — unlike PostgreSQL (§4.5) — SQLite's `DISTINCT` implementation for this query does **not** inject a total order over all columns: the effective tie order is a *plan artifact*. Replaying the **real** all-15-column `SELECT DISTINCT … ORDER BY created DESC` query two ways on an **isolated copy** of the live database (see §4.4 for the zero-mutation method) shows two different, equally-valid tie orders:

```
$ docker exec paperless-app bash -lc 'cd src && python /tmp/mechanism_distinct_sqlite.py'
=== EXPLAIN QUERY PLAN: real DISTINCT query, PLAN A (default) ===
   (9, 0, 0, 'SCAN TABLE documents_document USING INDEX documents_document_created_bedd0818')

=== EXPLAIN QUERY PLAN: real DISTINCT query, PLAN B (NOT INDEXED) ===
   (8, 0, 0, 'SCAN TABLE documents_document')
   (34, 0, 0, 'USE TEMP B-TREE FOR ORDER BY')

full order PLAN A (default): first10= [50, 49, 48, 47, 46, 45, 44, 43, 42, 41]  tie-boundary[20:30]= [70, 69, 68, 67, 66, 65, 64, 63, 62, 61]
full order PLAN B (NOTIDX ): first10= [31, 32, 33, 34, 35, 36, 37, 38, 39, 40]  tie-boundary[20:30]= [51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
PLAN A == PLAN B full order?  False
```

PLAN A (index walk) returns each tie block in descending `id` order (the index B-tree carries the rowid `= id` as an implicit trailing key); PLAN B (forced scan + temp-B-tree sort) returns ascending `id`. Both are valid results for `ORDER BY created DESC` because SQL imposes no order within a tie. Because ordinary browsing lets **both** pages pick PLAN A, they agree — hence §4.2 is clean. The instability only appears if the two independent page requests pick *different* plans.

### 4.4 The H3 artifact, demonstrated read-only on an isolated copy (controlled plan divergence) [OBSERVED — runtime]

To exhibit the artifact H3 predicts — two independent page queries resolving the tie under different orders — without mutating the live database or its schema, the live SQLite file was **copied** to an isolated scratch path and the real DISTINCT query replayed under PLAN A for "page 1" and PLAN B for "page 2". The divergence is induced purely by the read-only `NOT INDEXED` **query hint** (not a `DROP INDEX`), so there is **no schema mutation and no write to the live database**. The live database checksum is captured before and after to prove it:

```
$ docker exec paperless-app bash -lc '
  LIVE=data/db.sqlite3
  echo -n "live DB sha256 BEFORE: "; sha256sum "$LIVE" | cut -d" " -f1
  cp "$LIVE" /tmp/db_copy.sqlite3
  echo "isolated copy created at /tmp/db_copy.sqlite3 (size: $(stat -c%s /tmp/db_copy.sqlite3) bytes)"
  echo -n "live DB sha256 AFTER : "; sha256sum "$LIVE" | cut -d" " -f1
'
live DB sha256 BEFORE: 41cc808f7ceea56f5d8bd9539ca661c0f10fd17b7acda75997a96a6d6b415cf1
isolated copy created at /tmp/db_copy.sqlite3 (size: 339968 bytes)
live DB sha256 AFTER : 41cc808f7ceea56f5d8bd9539ca661c0f10fd17b7acda75997a96a6d6b415cf1
```

Simulating offset paging (page 1 = rows `[0:25]`, page 2 = rows `[25:50]`) under each plan, then the **cross-plan** case (page 1 under PLAN A, page 2 under PLAN B) over the identical, unchanged 40-row copy:

```
single-plan A dups/missing: ([], [])
single-plan B dups/missing: ([], [])
cross-plan (A page1, B page2) dups/missing: ([66, 67, 68, 69, 70], [51, 52, 53, 54, 55])
```

**Result [OBSERVED — runtime]:** when page 1 and page 2 resolve the `created` tie under different orders, ids `66`–`70` appear on **both** pages (duplicates) and ids `51`–`55` appear on **neither** (gaps), while `count` stays `40` — precisely the user's "same document twice / disappears then returns / total unchanged" symptom. This is the H3 mechanism, demonstrated without any mutation of the repository or the live database. It is **not** reached during ordinary browsing at this commit because (SQLite) both pages use one plan and (PostgreSQL) the order is total (§4.5); hence H3 is the real but **latent** root cause.

### 4.5 PostgreSQL production backend — also stable, because `.distinct()` injects a total order [OBSERVED — HTTP + runtime]

A PostgreSQL 13 backend was stood up (the production path, selected when `PAPERLESS_DBHOST` is set, `src/paperless/settings.py:L304-318`), migrated, seeded with the identical dataset, and driven through the same real HTTP API on port 8001. **Unfiltered**, 12 identical sweeps:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8001 paperless-app bash -lc 'cd src && python /tmp/sweep.py "page_size=25&ordering=-created" 12 "PostgreSQL-unfiltered-default-created"'
=== SWEEP LABEL: PostgreSQL-unfiltered-default-created ===
query: page_size=25&ordering=-created
universe: count=40 n_ids=40 ids=[4041, 4042, 4043, 4044, 4045, 4046, 4047, 4048, 4049, 4050, 4051, 4052, 4053, 4054, 4055, 4056, 4057, 4058, 4059, 4060, 4061, 4062, 4063, 4064, 4065, 4066, 4067, 4068, 4069, 4070, 4071, 4072, 4073, 4074, 4075, 4076, 4077, 4078, 4079, 4080]
sweep 01: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 02: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 03: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 04: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 05: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 06: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 07: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 08: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 09: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 10: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 11: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
sweep 12: count=40 pagesizes={1: 25, 2: 15} dups=[] missing=[] extra=[] CLEAN
DISTRIBUTION: 12/12 clean, 0/12 artifact
```

Filtered (`invoice` tag id=4, 30 docs > 25) was likewise **12/12 clean**:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8001 paperless-app bash -lc 'cd src && python /tmp/sweep.py "page_size=25&ordering=-created&tags__id__all=4" 12 "PostgreSQL-filtered-invoice-tags__id__all"'
=== SWEEP LABEL: PostgreSQL-filtered-invoice-tags__id__all ===
query: page_size=25&ordering=-created&tags__id__all=4
universe: count=30 n_ids=30 ids=[4041, 4042, 4043, 4044, 4045, 4046, 4047, 4048, 4049, 4050, 4051, 4052, 4053, 4054, 4055, 4061, 4062, 4063, 4064, 4065, 4066, 4067, 4068, 4069, 4070, 4071, 4072, 4073, 4074, 4075]
sweep 01: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 02: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 03: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 04: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 05: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 06: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 07: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 08: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 09: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 10: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 11: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
sweep 12: count=30 pagesizes={1: 25, 2: 5} dups=[] missing=[] extra=[] CLEAN
DISTRIBUTION: 12/12 clean, 0/12 artifact
```
 The reason is decisive and specific to this codebase: because `get_queryset()` applies `.distinct()`, the emitted query is `SELECT DISTINCT <all 15 columns> … ORDER BY created DESC`, and PostgreSQL's sort-based `DISTINCT` must sort by **every output column** — which includes the unique `id`. The `EXPLAIN` shows the unique `id` as the **second** sort key, so `(created DESC, id, …)` is already a **total order**:

```
$ docker exec paperless-postgres psql -U paperless -d paperless -c "EXPLAIN SELECT DISTINCT <15 cols> FROM documents_document ORDER BY created DESC LIMIT 25 OFFSET 25;"
                                                                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=76.46..77.06 rows=15 width=1234)
   ->  Unique  (cost=75.46..77.06 rows=40 width=1234)
         ->  Sort  (cost=75.46..75.56 rows=40 width=1234)
               Sort Key: created DESC, id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, modified, storage_type, added, filename, archive_filename, archive_serial_number
               ->  Seq Scan on documents_document  (cost=0.00..74.40 rows=40 width=1234)
(5 rows)
```

> In the `psql` commands above and below, `<15 cols>` denotes exactly the 15 `documents_document` columns enumerated verbatim in the routed `SELECT DISTINCT …` of §6 (`id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, created, modified, storage_type, added, filename, archive_filename, archive_serial_number`); the full list appears in every plan's `Sort Key` line, so the plan output is complete and unedited.

Because `id` is unique, `(created DESC, id)` breaks every `created` tie deterministically, so independent page requests cannot disagree. This holds even under the most adversarial conditions we could force. **Forcing genuine 4-worker parallel execution** (on a 4000-document dataset with two large tie blocks; GUCs applied one-per-statement) still preserves the total order via `Gather Merge`, and a 15-rep tie-boundary probe deep inside a tie block found **0/15** artifacts:

```
$ docker exec paperless-postgres psql -U paperless -d paperless -c "EXPLAIN ANALYZE SELECT DISTINCT <15 cols> FROM documents_document ORDER BY created DESC LIMIT 25 OFFSET 1000;"
                                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=186.12..187.43 rows=25 width=1226) (actual time=4.702..6.331 rows=25 loops=1)
   ->  Unique  (cost=133.89..342.83 rows=4000 width=1226) (actual time=4.389..6.303 rows=1025 loops=1)
         ->  Gather Merge  (cost=133.89..192.83 rows=4000 width=1226) (actual time=4.388..6.100 rows=1025 loops=1)
               Workers Planned: 4
               Workers Launched: 4
               ->  Sort  (cost=133.83..136.33 rows=1000 width=1226) (actual time=0.509..0.517 rows=205 loops=5)
                     Sort Key: created DESC, id, correspondent_id, title, document_type_id, content, mime_type, checksum, archive_checksum, modified, storage_type, added, filename, archive_filename, archive_serial_number
                     Sort Method: quicksort  Memory: 659kB
                     Worker 0:  Sort Method: quicksort  Memory: 25kB
                     Worker 1:  Sort Method: quicksort  Memory: 25kB
                     Worker 2:  Sort Method: quicksort  Memory: 25kB
                     Worker 3:  Sort Method: quicksort  Memory: 25kB
                     ->  Parallel Seq Scan on documents_document  (cost=0.00..84.00 rows=1000 width=1226) (actual time=0.003..0.131 rows=800 loops=5)
 Planning Time: 0.839 ms
 Execution Time: 6.467 ms
(15 rows)

$ docker exec -e PROBE_BASE=http://127.0.0.1:8001 paperless-app bash -lc 'cd src && python /tmp/boundary_probe.py "page_size=25&ordering=-created" 40 15 "PG-forced-parallel-4workers"'
=== BOUNDARY PROBE: PG-forced-parallel-4workers ===
query=page_size=25&ordering=-created pages=40/41 reps=15
rep 01: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 02: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 03: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 04: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 05: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 06: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 07: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 08: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 09: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 10: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 11: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 12: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 13: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 14: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
rep 15: count=4000 |pN|=25 |pN+1|=25 boundary_overlap(dups)=[] clean
distinct page40 id-sets across 15 reps: 1 (1 => stable, >1 => run-to-run variation)
distinct page41 id-sets across 15 reps: 1
DISTRIBUTION: 0/15 reps showed a boundary duplicate
```

**Running `VACUUM FULL` between the two page fetches** (heap reorganization — the realistic "nobody is editing anything" maintenance trigger — while still forced-parallel) was also **0/8**:

```
=== VACUUM-between-pages probe pages=40/41 reps=8 (forced-parallel) ===
rep 01: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 02: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 03: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 04: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 05: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 06: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 07: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
rep 08: |pN|=25 |pN+1|=25 boundary_overlap=[] clean
DISTRIBUTION: 0/8 showed a boundary duplicate
```

The total order `(created DESC, id, …)` is a logical property independent of physical heap layout, so even `VACUUM FULL` cannot destabilize it. **This means the `.distinct()` added to collapse H1 fan-out (§5) has the side effect of stabilizing pagination on PostgreSQL** — an important, non-obvious finding: at this commit, the H3 mechanism is latent on the production backend too.

---


## 5. Evidence for H1 — backend duplicates via many-to-many JOIN fan-out, collapsed at the base queryset

The tag filters in `DocumentFilterSet` (`src/documents/filters.py:L81`) JOIN the many-to-many tag tables. Some paths add no `.distinct()` of their own: `InboxFilter` → `qs.filter(tags__is_inbox_tag=True)` (`src/documents/filters.py:L66`) and the `TagsFilter` non-`in_list` loop → `qs.filter(tags__id=tag_id)` (`src/documents/filters.py:L58`); only the `in_list` branch self-dedups with `.distinct()` (`src/documents/filters.py:L52`). A document matching two tags therefore produces two rows in the pre-de-duplication JOIN.

### 5.1 Fan-out measured at runtime, collapse confirmed over HTTP [OBSERVED — runtime + HTTP]

The filter `?tags__name__icontains=invoice` matches **both** the `invoice` tag (30 docs) and the `invoice-paid` tag (6 docs); the 6 documents carrying both tags fan out. Measuring the raw pre-`.distinct()` JOIN versus the de-duplicated count at runtime, with the compiled SQL:

```
$ docker exec paperless-app bash -lc 'cd src && python /tmp/h1_fanout.py'
filter: tags__name__icontains='invoice'
pre-distinct JOIN row count (fan-out): 36
distinct document count: 30
duplicate rows collapsed by .distinct(): 6

--- compiled SQL, pre-distinct (fan-out) ---
SELECT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_tag" ON ("documents_document_tags"."tag_id" = "documents_tag"."id") WHERE "documents_tag"."name" LIKE %invoice% ESCAPE '\' ORDER BY "documents_document"."created" DESC

--- compiled SQL, with .distinct() (as DocumentViewSet.get_queryset does) ---
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" INNER JOIN "documents_document_tags" ON ("documents_document"."id" = "documents_document_tags"."document_id") INNER JOIN "documents_tag" ON ("documents_document_tags"."tag_id" = "documents_tag"."id") WHERE "documents_tag"."name" LIKE %invoice% ESCAPE '\' ORDER BY "documents_document"."created" DESC
```

The **canonical HTTP** result is de-duplicated to the distinct document count:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8000 paperless-app bash -lc 'cd src && python3 -c "
import urllib.request, base64, json, os
url = os.environ[\"PROBE_BASE\"] + \"/api/documents/?tags__name__icontains=invoice&page_size=25\"
req = urllib.request.Request(url)
req.add_header(\"Authorization\", \"Basic \" + base64.b64encode(b\"probe_admin:probe-disposable-pw\").decode())
d = json.load(urllib.request.urlopen(req, timeout=30))
print(\"HTTP count (collapsed):\", d[\"count\"], \"| n_results page1:\", len(d[\"results\"]))
"'
HTTP count (collapsed): 30 | n_results page1: 25
```

**36 raw JOIN rows for 30 distinct documents → 30 over HTTP.** The filter's own SQL carries no `DISTINCT`; the collapse happens at the base queryset.

### 5.2 Answer to H1 — the collapse is `get_queryset().distinct()` [OBSERVED — runtime + HTTP]

Yes: the backend query **can** produce duplicate rows via M2M JOIN fan-out (36 → 30). Those duplicates are collapsed by the base queryset's `Document.objects.distinct()` in `DocumentViewSet.get_queryset()` (`src/documents/views.py:L198-199`) — **not** by the individual filters (their emitted SQL has no `DISTINCT`). So H1's phrasing ("duplicates that get collapsed somewhere later") is accurate; but that collapse occurs at the base queryset, **before** pagination (§6), which is why fan-out is not what destabilizes paging.

---

## 6. Evidence for H2 — de-duplication precedes pagination (framing refuted) [OBSERVED — HTTP]

Because `.distinct()` lives in `get_queryset()` (`src/documents/views.py:L198-199`), it is part of the base queryset, and `StandardPagination` (`src/documents/views.py:L182`; defined `src/paperless/views.py:L8-11`) slices that already-`DISTINCT` queryset. Rather than a non-routed `APIRequestFactory` call, we captured the SQL Django **actually emitted for a live routed request** by enabling PostgreSQL statement logging and issuing a real authenticated `GET /api/documents/?page=2` through the HTTP API:

```
$ docker exec paperless-postgres psql -U paperless -d paperless -c "ALTER SYSTEM SET log_statement='all'; SELECT pg_reload_conf();"
$ docker exec -e PROBE_BASE=http://127.0.0.1:8001 paperless-app bash -lc 'cd src && python3 -c "
import urllib.request, base64, json, os
url = os.environ[\"PROBE_BASE\"] + \"/api/documents/?page=2&page_size=25&ordering=-created\"
req = urllib.request.Request(url)
req.add_header(\"Authorization\", \"Basic \" + base64.b64encode(b\"probe_admin:probe-disposable-pw\").decode())
d = json.load(urllib.request.urlopen(req, timeout=30))
print(\"live page2 count:\", d[\"count\"], \"n_results:\", len(d[\"results\"]))
"'
live page2 count: 40 n_results: 15
$ docker logs paperless-postgres   # sliced to the routed statements for that request
```

The two statements Django ran for the page-2 envelope (complete, unedited):

```sql
-- page-slice query (DISTINCT and LIMIT/OFFSET in the SAME statement):
SELECT DISTINCT "documents_document"."id", "documents_document"."correspondent_id", "documents_document"."title", "documents_document"."document_type_id", "documents_document"."content", "documents_document"."mime_type", "documents_document"."checksum", "documents_document"."archive_checksum", "documents_document"."created", "documents_document"."modified", "documents_document"."storage_type", "documents_document"."added", "documents_document"."filename", "documents_document"."archive_filename", "documents_document"."archive_serial_number" FROM "documents_document" ORDER BY "documents_document"."created" DESC LIMIT 15 OFFSET 25

-- count query (count is COUNT(*) over the DISTINCT subquery):
SELECT COUNT(*) FROM (SELECT DISTINCT "documents_document"."id" AS "col1", "documents_document"."correspondent_id" AS "col2", "documents_document"."title" AS "col3", "documents_document"."document_type_id" AS "col4", "documents_document"."content" AS "col5", "documents_document"."mime_type" AS "col6", "documents_document"."checksum" AS "col7", "documents_document"."archive_checksum" AS "col8", "documents_document"."created" AS "col9", "documents_document"."modified" AS "col10", "documents_document"."storage_type" AS "col11", "documents_document"."added" AS "col12", "documents_document"."filename" AS "col13", "documents_document"."archive_filename" AS "col14", "documents_document"."archive_serial_number" AS "col15" FROM "documents_document") subquery
```

> Provenance: this SQL is **[OBSERVED — runtime]** as the exact statements PostgreSQL logged while serving a real, routed HTTP request through the registered `UnifiedSearchViewSet`/`DocumentViewSet` stack — not an `APIRequestFactory` reconstruction. (`LIMIT 15` rather than `25` on page 2 is Django's `Paginator` clamping the last page's slice to the remaining `40 − 25 = 15` rows.)

**Answer to H2 [OBSERVED — HTTP]:** de-duplication **precedes** pagination. `DISTINCT` is applied to the row source **within the same statement** that carries `LIMIT/OFFSET`, and `count` is `COUNT(*)` over the `DISTINCT` subquery (hence the constant total). The "pagination happening before any de-duplication" framing is **refuted**: there is no window in which un-de-duplicated rows are paginated. This is also *why* fixing H1-style fan-out cannot fix a paging artifact — the fan-out is gone before the slice; what the default ordering still lacks is a unique tiebreaker (H3).

---

## 7. Authorization verification — the "sharing rules" premise does not reproduce here [OBSERVED — HTTP + STATICALLY VERIFIED]

At this commit the only gate on every documents endpoint is `permission_classes = (IsAuthenticated,)` (`src/documents/views.py:L183`); the `REST_FRAMEWORK` block (`src/paperless/settings.py:L116`) sets no default permission class and no object-level permission backend, and `django-guardian` is not installed. To verify empirically rather than assert, we ran the **identical two-page sweep** (fetching page 1 **and** page 2 for each user — the earlier version of this document contained a loop that never fetched page 2) as the superuser and as the non-staff user, on **both** backends.

Probe user attributes [OBSERVED — runtime]:

```
$ docker exec paperless-app bash -lc 'cd src && python /tmp/user_attrs.py'
probe_admin: is_superuser=True is_staff=True is_active=True groups=[] user_perms=[]
probe_viewer: is_superuser=False is_staff=False is_active=True groups=[] user_perms=[]
```

Side-by-side page sweep — SQLite (port 8000) and PostgreSQL (port 8001) [OBSERVED — HTTP]:

```
$ docker exec -e PROBE_BASE=http://127.0.0.1:8000 paperless-app bash -lc 'cd src && python /tmp/auth_compare.py'   # SQLite
superuser  page1 status/count: 200 40 | page2: 200 40
non-staff  page1 status/count: 200 40 | page2: 200 40
superuser total ids (p1+p2): 40 distinct: 40
non-staff total ids (p1+p2): 40 distinct: 40
id sets IDENTICAL between superuser and non-staff? True
symmetric difference (docs visible to one but not the other): []
anonymous page1 HTTP status: 401 (401 => authentication required)

$ docker exec -e PROBE_BASE=http://127.0.0.1:8001 paperless-app bash -lc 'cd src && python /tmp/auth_compare.py'   # PostgreSQL
superuser  page1 status/count: 200 40 | page2: 200 40
non-staff  page1 status/count: 200 40 | page2: 200 40
id sets IDENTICAL between superuser and non-staff? True
symmetric difference (docs visible to one but not the other): []
anonymous page1 HTTP status: 401 (401 => authentication required)
```

Absence of an object-level permission backend [STATICALLY VERIFIED + OBSERVED — runtime]:

```
$ docker exec paperless-app bash -lc 'pip show django-guardian 2>&1 | head -1'
WARNING: Package(s) not found: django-guardian
$ docker exec paperless-app bash -lc 'grep -rn "guardian\|ObjectPermission\|get_objects_for_user\|DjangoObjectPermissions" src/paperless/settings.py src/documents/views.py || echo "no guardian/object-permission references"'
no guardian/object-permission references
```

**Answer to the premise [OBSERVED — HTTP]:** the superuser and the non-staff user receive **identical** `count` and identical page-1/page-2 contents (empty symmetric difference) on both backends; anonymous access is `401`. The list behavior is **permission-independent** at this commit; the "sharing rules shape what you see" scenario **does not reproduce** because that feature is not present in this code. (In the wild, a per-user filter that changes the *membership* of a tied block would change *where* the boundary falls within a tie — but the destabilizing element is still the tie order, H3, not an authorization check.)

---

## 8. UI correlation — why offset math over an unstable order would become visible

The Angular client models a page **number**, not a cursor, and assumes a single fixed global order:

- `src-ui/src/app/data/results.ts:L1-5` declares `interface Results<T> { count: number; results: T[] }` — **no `next`/`previous`**. The client does not follow cursor links; it navigates by page number. (The DRF JSON does include `next`/`previous`, per `docs/api.rst`, but the Angular type ignores them.)
- `src-ui/src/app/services/rest/abstract-paperless-service.ts` `list()` sets the query params `page` (`L41`), `page_size` (`L44`), and `ordering` (`L48`).
- `src-ui/src/app/services/document-list-view.service.ts` defaults the documents list to `sortField: 'created'` (`L93`) and `sortReverse: true` (`L94`) → effective `ordering=-created`, the **same non-unique key** as the model default; page size comes from user settings (`currentPageSize`, `L73`); `reload()` calls `listFiltered(currentPage, currentPageSize, sortField, sortReverse, filterRules)` (`L138-145`), which delegates to `list(...)` (`document.service.ts:L104`), and stores `collectionSize = result.count` (`L149`).
- The UI's **total-page calculation** is explicit: `getLastPage(): number { return Math.ceil(this.collectionSize / this.currentPageSize) }` (`src-ui/src/app/services/document-list-view.service.ts:L276-277`) — i.e. `ceil(count / page_size)`. The template binds this model to the pager: `<ngb-pagination [pageSize]="list.currentPageSize" [collectionSize]="list.collectionSize" [(page)]="list.currentPage" [maxSize]="5" [rotate]="true">` (`src-ui/src/app/components/document-list/document-list.component.html:L95-96`), which independently computes the page count from `collectionSize`/`pageSize`.
- On any page beyond the first, a `404` triggers a **reset to page 1**: `if (activeListViewState.currentPage != 1 && error.status == 404)` (`L158`) → `currentPage = 1` (`L160`) → `this.reload()` (`L161`).

**Mechanism in UI terms:** each page navigation is an **independent** offset query (`LIMIT page_size OFFSET (page-1)*page_size`) over a global order the UI assumes is stable, with total pages computed as `ceil(count / page_size)`. *If* that order were unstable across ties (the H3 mechanism), the offset arithmetic would silently show the same `id` on two adjacent pages and omit another, with `count` (and the computed page total) unchanged — matching the report. At this commit the backend order is *not* unstable during ordinary browsing (§4), so the UI does not surface the artifact here; the UI is the surface on which the latent mechanism *would* become visible if a plan divergence (SQLite) or a non-total ordering were introduced. Full-text **search** is a separate path — "Results are always sorted by search score" (`docs/api.rst:L162`) — so it is not governed by the default `-created` ordering (search scores can themselves tie; search-score tie stability was not separately tested and is out of scope).

**Observed side effect of the `404 → page 1` reset — a persistent `undefined: I` error banner [OBSERVED — HTTP + UI; mechanism INFERRED; orthogonal to the H1/H2/H3 thesis; reference-only]:** the reset path in the bullet above (`L158` → `L160` → `L161`) is reached whenever `currentPage` falls out of range — not only when a filter shrinks the result set, but equally on a **page-size increase** that reduces the total page count and on a **plain (unfiltered) load of a persisted out-of-range page** — and exercising it directly through the real client surfaces a **pre-existing frontend defect** that is *independent* of the ordering/pagination question this document answers. The banner is **not** transient: once set it stays on screen — blanking the whole document grid while the header still shows the correct `count` — until the user takes an action that issues a fresh page‑1 request (reset filters, change the query or sort, or navigate to a valid page); because `currentPage` is persisted, a plain browser reload of the out-of-range page re-triggers the same `404` and re-displays the banner rather than clearing it. It is recorded here because it is reached through the very same `404`-on-a-stale-page path §8 describes.

*Reproduction (canonical HTTP API + real Angular client; `admin`, default SQLite; 40 seeded documents, `page_size = 25`):* open Documents **page 2**, then enable a tag filter whose result is a single page — e.g. `invoice-paid` (6 documents). The client re-requests the list while `currentPage` is still `2`, which is now out of range. The **identical** banner is reached with **no filter active**, confirming the trigger is any out-of-range `currentPage` rather than filtering specifically: (a) increasing the page size while on a page that no longer exists (e.g. `page_size` 25 → 100 on a 60‑document set — the header still reads `60 documents` with **no** `(filtered)` suffix and `Reset filters` disabled), or (b) a plain load of a **persisted** out-of-range page (header e.g. `8 documents`, again unfiltered with `Reset filters` disabled).

*OBSERVED (HTTP) — captured from the browser network log:* the stale page-2 request is issued **multiple times concurrently**, each returns `404`, and the reset request for page 1 then succeeds:

```
GET /api/documents/?page=2&page_size=25&ordering=-created                  → 200      (on page 2, unfiltered)
GET /api/documents/?page=2&page_size=25&ordering=-created&tags__id__all=5  → 404  ×3  (page 2 now out of range)
GET /api/documents/?page=1&page_size=25&ordering=-created&tags__id__all=5  → 200      (reset-to-page-1 succeeds)
```

Each `404` body is the DRF pagination default `{"detail":"Invalid page."}` (`content-length: 26`, `content-type: application/json`, `x-version: 1.7.0`).

*OBSERVED (UI):* despite the page-1 request returning `200`, a full-width red banner reads **`Error while loading documents: undefined: I`** over an **empty** document grid, while the header still reports the correct count (`6 documents (filtered)`) and the pager has collapsed to a single page.

*INFERRED (mechanism, from source):* `reload()` clears `this.error = null` at its start (`src-ui/src/app/services/document-list-view.service.ts:L135`), but the concurrent multi-fetch means a **late** `404` can arrive *after* the reset has already set `currentPage = 1`. That late `404` fails the reset guard `if (activeListViewState.currentPage != 1 && error.status == 404)` (`L158`) and falls into the `else` branch (`L162-181`), which composes the message by looking the response key up in `DOCUMENT_SORT_FIELDS` (`src-ui/src/app/services/rest/document.service.ts:L16-24`). That list has no `detail` entry, so `DOCUMENT_SORT_FIELDS.find((f) => f.field == fieldName)?.name` (`L173`) is `undefined` → the `"undefined"` prefix; the value `error.error['detail']` is the string `"Invalid page."`, and indexing `[0]` (`fieldError[0]`, `L174`) yields `"I"`, producing `this.error = "undefined: I"` (`L180`). The banner is therefore an artifact of the multi-fetch race on the reset path, not of the list order. It **persists** because `reload()` clears `this.error` only at its **start** (`L135`) and this `else` branch does not itself re-issue a request — so once the late `404` sets the message, nothing clears it until a later user-initiated `reload()` succeeds; a plain reload of the same persisted out-of-range page merely re-enters the race and re-sets it.

*Disposition:* this is a **pre-existing** defect in `document-list-view.service.ts` at this commit; it does not touch the `-created` ordering that is the subject of this document and it neither confirms nor contradicts H1/H2/H3. Per the read-only scope of this investigation (§12), **no fix was applied.** The standard remediation would be to (a) treat non-field response keys (like `detail`) as a plain message in the `else` branch instead of formatting them as `field: value`, and (b) suppress or coalesce error state from superseded/aborted list requests so a late `404` cannot overwrite a successful reset. Recorded **for reference only.**

---


## 9. Database note — SQLite and PostgreSQL both stable canonically at this commit

- **SQLite (default backend) [OBSERVED — HTTP + runtime].** Repeated identical sweeps were 12/12 clean unfiltered and filtered (§4.2). Both independent page requests use the same index-walk plan (`SCAN TABLE documents_document USING INDEX documents_document_created_bedd0818`, §4.3), so they resolve the tie identically. SQLite's `DISTINCT` for this query does **not** inject a total order; the tie order is a *plan artifact* (index walk → descending `id`; forced sort → ascending `id`, §4.3). The order within a tie is therefore not guaranteed — it is only *coincidentally* stable because ordinary browsing keeps both pages on the same plan. A cross-plan divergence produces the artifact (§4.4).
- **PostgreSQL (production backend) [OBSERVED — HTTP + runtime].** Selected when `PAPERLESS_DBHOST` is set (`postgresql_psycopg2`, `src/paperless/settings.py:L310`, branch `L304-318`). Repeated identical sweeps were 12/12 clean unfiltered and filtered (§4.5). PostgreSQL documents that the order of rows tied on all `ORDER BY` expressions is unspecified and must not be relied upon (see References); a *bare* `ORDER BY created` would indeed be free to differ per plan/heap/parallelism. **However**, the query this application actually issues is `SELECT DISTINCT <all 15 columns> … ORDER BY created DESC`, and PostgreSQL's sort-based `DISTINCT` sorts by every output column, placing the unique `id` as the second sort key. That makes the effective order the **total** order `(created DESC, id, …)`, which we observed to be deterministic even under forced 4-worker parallelism and `VACUUM FULL` between page fetches (§4.5). No probabilistic "the norm / at least as likely" claim is made: the actual observed distribution on PostgreSQL at this commit is **0 artifacts** across every sweep and adversarial probe.

The prior version of this document asserted the artifact was "the norm on PostgreSQL" and "at least as likely" in production; that comparative-likelihood language was unsupported and is retracted. The supported statements are (a) PostgreSQL guarantees no order among rows tied on the ordering expressions, and (b) at this commit the `.distinct()` in `get_queryset()` incidentally converts the effective ordering into a total order, which is why the observed PostgreSQL distribution is 0 artifacts.

---

## 10. Reference-only remediation (described, **NOT applied**)

> **Reference-only. No source file was modified as part of this investigation** (see §12). These are the standard fixes for the documented Django/DRF failure mode, offered for the reader's benefit; they are also what would make the H3 mechanism impossible rather than merely latent.

The root-cause mechanism (H3) is an ordering with no unique final key. Any of the following removes the ambiguity for *all* code paths and backends:

1. **Append a unique tiebreaker to the model ordering** (smallest change): `ordering = ("-created", "-id")` in `Document.Meta` (`src/documents/models.py:L207-208`). `id` is a monotonic integer PK (`DEFAULT_AUTO_FIELD`, `src/paperless/settings.py:L320`) and an allowed ordering field (`src/documents/views.py:L188`). This guarantees a total order regardless of query plan, backend, `.distinct()`, or `?ordering=` override.
2. **A stable-ordering filter** that appends `pk` whenever the requested ordering is not already unique — mirroring Django admin's `_get_deterministic_ordering`, which appends the primary key for exactly this reason (Django #17198). This also fixes explicit `?ordering=` requests, not just the default.
3. **Add `.distinct()`/dedup discipline** to the fan-out filters (e.g. `InboxFilter`, `src/documents/filters.py:L63-66`) for tidiness. Note this addresses **H1**, not the paging mechanism — the base `get_queryset().distinct()` already collapses fan-out before pagination (§6).
4. **Switch the list to `CursorPagination`.** Per DRF's documentation, `CursorPagination` guarantees a client "will never see the same item twice" while paging, but it **requires an ordering that is an unchanging, unique or nearly-unique, and indexed field (or fields)** — e.g. a monotonically increasing `created`/`id` — and it replaces page-number navigation with opaque cursors. It does not "automatically order on a unique field"; the developer must supply a suitable ordering. Adopting it would require the Angular `Results<T>` page-number model (`src-ui/src/app/data/results.ts:L1-5`, `getLastPage()` at `document-list-view.service.ts:L276-277`) to follow `next`/`previous` cursor links instead of computing `ceil(count / page_size)`.

Options 1 and 2 are the minimal, backend-independent fixes; both were corroborated by the confirmation that a total order (as PostgreSQL's `.distinct()` already produces, §4.5) yields 0 artifacts.

---

## 11. Observed-vs-Inferred ledger

Categories are consistent with the labels defined at the top and reconcile to the section that establishes each claim.

| # | Claim | Status | Basis |
|---|-------|--------|-------|
| 1 | Default ordering is `-created`, non-unique, no tiebreaker | STATICALLY VERIFIED | `src/documents/models.py:L152,L208`; no `ordering` on the viewset |
| 2 | `/api/documents/` routes to `UnifiedSearchViewSet(DocumentViewSet)`, inheriting queryset/pagination/perms | STATICALLY VERIFIED | `src/paperless/urls.py:L32`; `src/documents/views.py:L172,L377`; routed SQL §6 |
| 3 | Pagination is offset-based, `page_size=25` | STATICALLY VERIFIED | `StandardPagination`, `src/paperless/views.py:L8-11` |
| 4 | 12/12 identical sweeps clean on default SQLite, unfiltered AND filtered (invoice, 30>25) | OBSERVED — HTTP | §4.2 |
| 5 | SQLite serves the order via one plan for both pages; `DISTINCT` does not inject a total order (tie order is a plan artifact) | OBSERVED — runtime | `EXPLAIN QUERY PLAN` + PLAN A≠PLAN B, §4.3 |
| 6 | Cross-plan divergence on an isolated copy yields dups `[66-70]`, gaps `[51-55]`, `count` constant; live DB byte-identical before/after | OBSERVED — runtime | §4.4 (sha256 `41cc808f…` unchanged) |
| 7 | 12/12 identical sweeps clean on PostgreSQL, unfiltered AND filtered | OBSERVED — HTTP | §4.5 |
| 8 | `.distinct()` makes the PostgreSQL order total `(created DESC, id, …)`; stable under forced 4-worker parallel (0/15) and `VACUUM FULL` (0/8) | OBSERVED — HTTP + runtime | `EXPLAIN`/`EXPLAIN ANALYZE` + probes, §4.5 |
| 9 | H3 mechanism is real but latent at this commit (surfaces only under plan divergence / a non-total order) | OBSERVED — runtime + INFERRED | §4.3–§4.5 synthesis |
| 10 | Filters fan out (36 raw → 30 distinct); collapsed by base `.distinct()` | OBSERVED — runtime + HTTP | §5.1, §5.2 |
| 11 | De-dup precedes pagination (`DISTINCT … LIMIT/OFFSET` one statement; `COUNT(*)` over DISTINCT subquery) | OBSERVED — HTTP (routed SQL) | §6 |
| 12 | Superuser and non-staff see identical `count`/pages (empty symmetric difference); anonymous `401` | OBSERVED — HTTP | §7 |
| 13 | Only `IsAuthenticated`; no object-level perms; no `django-guardian` | STATICALLY VERIFIED + OBSERVED — runtime | `src/documents/views.py:L183`; `pip show`/grep §7 |
| 14 | Angular navigates by page number; total pages `= ceil(count/page_size)` | STATICALLY VERIFIED | `results.ts:L1-5`; `abstract-paperless-service.ts:L41,44,48`; `document-list-view.service.ts:L93-94,L140-143,L149,L158-161,L276-277`; template `L95-96` |
| 15 | PostgreSQL guarantees no order among rows tied on the ordering expressions | STATICALLY VERIFIED (external docs) + INFERRED for the bare-ORDER-BY case | PostgreSQL docs (References); not reached here because `.distinct()` makes the order total |
| 16 | Any action that leaves `currentPage` out of range (filter shrink, page-size increase, or a persisted out-of-range page on a plain load) triggers the `404 → page 1` reset; a late `404` on that path renders a persistent `undefined: I` banner over an empty grid while `count` stays correct (pre-existing frontend defect, orthogonal to H1/H2/H3) | OBSERVED — HTTP + UI; mechanism INFERRED | §8 note; network log (3×`404` on stale page + reset `200`), `{"detail":"Invalid page."}`; `document-list-view.service.ts:L135,L158,L162-181`; `document.service.ts:L16-24` |

---

## 12. Repository-integrity proof and complete cleanup

All observation tooling was temporary and confined to scratch paths and the git-ignored runtime trees (`data/`, `media/`, `*.log`, `src-ui/dist/`). No existing repository file was modified; the only net-new path is this answer document. The complete, captured cleanup:

```
### [1] SQLite live DB sha256 immediately before cleanup mutations
41cc808f7ceea56f5d8bd9539ca661c0f10fd17b7acda75997a96a6d6b415cf1
    (== the pre-experiment checksum -> all HTTP sweeps, the mechanism demo, H1, and auth were READ-ONLY)

### [2] Stop the two investigation runservers (SQLite 127.0.0.1:8000, PostgreSQL 127.0.0.1:8001)
    (ps/pkill are absent in the container; processes were located via /proc/<pid>/cmdline and stopped with the kill builtin)
confirmed: no runserver processes remain
port 8000: closed (URLError)
port 8001: closed (URLError)

### [3] Delete disposable probe users from the SQLite runtime DB
deleted probe user rows: 2
remaining users: ['admin', 'consumer', 'viewer']

### [4] Remove the PostgreSQL investigation container + its anonymous volume
paperless-postgres removed
confirmed: paperless-postgres gone

### [5] Remove container-side temp artifacts (probe scripts, JSON captures, isolated DB copy)
  (no probe artifacts remain)

### [6] Remove host-side probe scripts

### [7] Repository integrity — git state (tracked files only; runtime data is git-ignored)
-- git status --porcelain --
    (empty — no tracked file modified except the net-new answer document, shown below)
-- branch --
blitzy-e58dd14a-bc36-4f2f-912f-3f73de6974de
-- gitignore coverage of runtime artifacts --
  ignored: data/db.sqlite3
  ignored: media
  ignored: src-ui/dist
```

The 12 read-only source references cited in this document were verified byte-identical to the baseline throughout (`git diff --quiet HEAD -- <file>` for each of `src/documents/views.py`, `src/paperless/views.py`, `src/documents/models.py`, `src/documents/filters.py`, `src/paperless/settings.py`, `src/paperless/urls.py`, `src/manage.py`, `src-ui/src/app/data/results.ts`, `src-ui/src/app/services/rest/abstract-paperless-service.ts`, `src-ui/src/app/services/document-list-view.service.ts`, `src-ui/src/app/components/document-list/document-list.component.html`, `docs/api.rst` — all unmodified). The provided-environment containers `paperless-app` and `paperless-redis` were left running; only the `paperless-postgres` container this investigation added was removed.

**Baseline vs. delivery snapshot [OBSERVED — runtime].** During the investigation the working tree is clean and `HEAD` is the investigated **source baseline** `542221a38`, with this answer document still untracked:

```
$ git rev-parse --short HEAD
542221a38
$ git status --porcelain -uall
?? blitzy/documentation/paperless-ngx_542221a38dff.md
$ git diff --name-only HEAD
        (empty — no tracked file was modified)
```

This is the **pre-commit snapshot**; committing this document (a `docs:` commit) advances `HEAD` to that new commit atop `542221a38`, at which point the file is tracked and the working tree is again clean. No remediation was applied.

---

## References (external, authoritative)

- Django ticket #34251 — non-deterministic ordering on non-unique columns leading to inconsistent pagination: https://code.djangoproject.com/ticket/34251
- Django ticket #17198 — admin appends the primary key to non-unique ordering for a deterministic total order (`_get_deterministic_ordering`): https://code.djangoproject.com/ticket/17198
- Django REST Framework issue #6886 — "Missing/duplicate records when using `ordering` and `LimitOffsetPagination`," resolved by adding a unique tiebreaker (`['-event_date', 'id']`): https://github.com/encode/django-rest-framework/issues/6886
- Django REST Framework pagination docs — `CursorPagination` guarantees no repeated items but requires an unchanging, unique/nearly-unique, indexed ordering: https://www.django-rest-framework.org/api-guide/pagination/
- PostgreSQL — `ORDER BY`: order of rows tied on the sort expressions is unspecified and must not be relied upon: https://www.postgresql.org/docs/current/queries-order.html
- PostgreSQL — Indexes and `ORDER BY` (B-tree ordering; table TID used as a tiebreaker): https://www.postgresql.org/docs/current/indexes-ordering.html
- SQLite — the `rowid` and its role in table B-trees (implicit trailing key): https://www.sqlite.org/rowidtable.html

---

## Appendix A — observation harnesses (temporary; removed at cleanup)

The core sweep harness `sweep.py` (fetches the universe via `page_size=100000`, then N independent consecutive-page sweeps at `page_size=25`, computing per-sweep duplicates/gaps/extras and the distribution):

```python
import urllib.request, base64, json, sys, os
from collections import Counter
BASE = os.environ.get("PROBE_BASE", "http://127.0.0.1:8000") + "/api/documents/"
AUTH = base64.b64encode(b"probe_admin:probe-disposable-pw").decode()

def fetch(url):
    req = urllib.request.Request(url); req.add_header("Authorization", "Basic " + AUTH)
    with urllib.request.urlopen(url=req, timeout=60) as r:
        return json.load(r)

def full_universe(query):
    parts = [p for p in query.split("&") if not p.startswith("page_size=") and not p.startswith("page=")]
    d = fetch(f"{BASE}?{'&'.join(parts + ['page_size=100000'])}")
    return d["count"], [x["id"] for x in d["results"]]

def one_sweep(query):
    parts = [p for p in query.split("&") if not p.startswith("page=")]
    q = "&".join(parts); pages = {}; page = 1; count = None
    while True:
        d = fetch(f"{BASE}?{q}&page={page}"); count = d["count"]
        pages[page] = [x["id"] for x in d["results"]]
        if not d.get("next"): break
        page += 1
    return count, pages

def analyze(U, count, pages):
    allids = [i for ids in pages.values() for i in ids]; c = Counter(allids)
    dups = sorted([i for i, n in c.items() if n > 1])
    missing = sorted(set(U) - set(allids)); extra = sorted(set(allids) - set(U))
    return dups, missing, extra, {p: len(ids) for p, ids in pages.items()}

query, n, label = sys.argv[1], int(sys.argv[2]), sys.argv[3]
ucount, U = full_universe(query)
print(f"=== SWEEP LABEL: {label} ===\nquery: {query}\nuniverse: count={ucount} n_ids={len(U)} ids={U}")
clean = 0
for s in range(1, n + 1):
    count, pages = one_sweep(query); dups, missing, extra, ps = analyze(U, count, pages)
    ok = (not dups) and (not missing) and (not extra) and (count == ucount); clean += 1 if ok else 0
    print(f"sweep {s:02d}: count={count} pagesizes={ps} dups={dups} missing={missing} extra={extra} {'CLEAN' if ok else 'ARTIFACT'}")
print(f"DISTRIBUTION: {clean}/{n} clean, {n-clean}/{n} artifact")
```

The remaining harnesses — `seed_dataset.py` (tie-inducing seed), `mechanism_distinct_sqlite.py` (read-only isolated-copy plan-divergence demo), `h1_fanout.py`, `auth_compare.py` (two-page superuser-vs-non-staff comparison), `boundary_probe.py` (parallel tie-boundary probe), and the `VACUUM FULL`-between-pages probe — followed the same shape (authenticated `urllib` HTTP against the loopback `runserver`, or read-only SQL on an isolated copy). All were removed at cleanup (§12); the repository retains only this document.

