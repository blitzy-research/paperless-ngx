# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** that comprehensively answers why the Paperless-ngx document list view exhibits "haunted" pagination behavior — where documents duplicate across neighboring pages, disappear and reappear, and where the glitch appears to depend on the viewer's permission level — by tracing the exact root cause through the backend API layer, the ORM query chain, the pagination mechanism, and the frontend list-state model, grounding every conclusion in the actual source code.

**Category:** Create new documentation
**Documentation Type:** Technical investigation / Root-cause analysis document

The specific documentation requirements are:

- Identify whether the backend produces duplicate rows that are collapsed elsewhere, whether pagination (`OFFSET`/`LIMIT`) is applied before or after deduplication (`.distinct()`), or whether the ordering is quietly unstable when multiple rows tie on the primary sort key
- Trace what the API actually returns across consecutive page requests under common filter combinations, and reconcile that with what the frontend's `DocumentListViewService` expects pagination to mean
- Determine the exact condition that destabilizes the list, with rationale grounded in code analysis
- Assess the user's observation that the glitch correlates with non-admin viewers whose visibility is shaped by sharing rules
- Produce a self-contained markdown document placed at `blitzy/documentation/paperless-ngx_542221a38dff.md` that answers all of the above
- No modifications to existing repository files; temporary observation scripts may be mentioned conceptually but the repository itself must remain unchanged

### 0.1.2 Special Instructions and Constraints

- **Rule — No repository modifications:** The implementation rule `SWE-AtlasQnA-Repo` explicitly mandates: "Do not modify any existing files in the source repository." The output is a standalone markdown document only.
- **Output location:** The document must be placed at `blitzy/documentation/paperless-ngx_542221a38dff.md`.
- **Evidence standard:** "Do not make assumptions, base your answers on the code as the truth." Every conclusion must cite specific source files and line numbers.
- **Thinking and rationale:** "Provide thinking / rationale behind the answers." The document must walk through the reasoning chain, not merely state a verdict.
- **Temporary scripts:** The user allows conceptual discussion of temporary observation scripts but requires that "anything temporary should be cleaned up afterward."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To answer whether the backend produces duplicates, we will analyze `DocumentViewSet.get_queryset()` in `src/documents/views.py` (line 199), which calls `Document.objects.distinct()`, and the `TagsFilter` in `src/documents/filters.py` (lines 36–60), which applies M2M JOINs through `.filter(tags__id=tag_id)` and conditionally calls `.distinct()` only for the `in_list` variant.
- To answer whether pagination happens before or after deduplication, we will trace how DRF's `PageNumberPagination` (subclassed as `StandardPagination` in `src/paperless/views.py`, lines 8–11) interacts with the ORM queryset — specifically, that `OFFSET`/`LIMIT` is appended to the SQL after `DISTINCT` is already in the `SELECT` clause, meaning deduplication precedes slicing.
- To answer whether ordering is unstable on tied sort keys, we will examine `Document.Meta.ordering = ("-created",)` in `src/documents/models.py` (line 208) and the `ordering_fields` declaration in `DocumentViewSet` (lines 187–196), demonstrating that no secondary tiebreaker column (such as `id`) is specified, leaving the database free to return tied rows in arbitrary order across separate query executions.
- To assess the sharing-rules observation, we will document that this codebase version has **no per-document ownership or sharing model** — the only permission check is `IsAuthenticated` — and explain how the user's perception may conflate different saved-view filter configurations with permission-based visibility.
- To produce the output document, we will create `blitzy/documentation/paperless-ngx_542221a38dff.md` containing the full investigation narrative.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **The `DISTINCT` + non-deterministic ordering interaction needs explicit documentation:** The combination of `.distinct()` on the queryset and ordering by the non-unique `created` field is the primary mechanism that destabilizes pagination. This is not documented anywhere in the existing `docs/api.rst` or codebase comments.
- **The tag-filter JOIN amplification behavior needs documentation:** When `tags__id__all` applies multiple `.filter(tags__id=X)` calls, each generates a separate JOIN to the M2M `documents_document_tags` table. The queryset-level `.distinct()` prevents duplicate result rows, but it interacts with the non-deterministic ordering to produce unstable page boundaries.
- **The frontend pagination model's assumptions need documentation:** `DocumentListViewService` (in `src-ui/src/app/services/document-list-view.service.ts`) assumes that the backend returns stable, non-overlapping page slices — an assumption that breaks when the backend ordering is non-deterministic.
- **The absence of object-level permissions needs clarification:** The user believes the glitch depends on "sharing rules," but `src/documents/views.py` uses only `IsAuthenticated` for all document endpoints. The investigation must explicitly address this misconception.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a Sphinx-based documentation system with a Read the Docs theme, covering user guides, configuration, API reference, and administration, but containing no existing documentation that addresses pagination stability, sort-order determinism, or the interaction between `DISTINCT` and `OFFSET`/`LIMIT` pagination in the document list endpoint.

- **Documentation framework:** Sphinx ~4.5.0 (from `Pipfile` dev-packages)
- **Documentation generator configuration:** `docs/conf.py` — configures `sphinx_rtd_theme`, project metadata, extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`)
- **Documentation build file:** `docs/Makefile` — standard Sphinx targets (html, epub, latex, linkcheck, doctest)
- **Documentation hosting:** `.readthedocs.yml` at repository root, pointing to `docs/conf.py` and `docs/requirements.txt`
- **Existing REST API docs:** `docs/api.rst` — documents endpoints, query/filter parameters, search, upload, versioning, and authorization. Pagination is mentioned only briefly in the search section ("Pagination works exactly the same as it does for normal requests on this endpoint") with no deeper explanation.
- **Diagram tools detected:** None configured in the docs system. The tech spec uses Mermaid diagrams, but the Sphinx docs do not integrate Mermaid or PlantUML.
- **API documentation tools:** No automated API doc generation (no JSDoc, Sphinx autodoc for views, or OpenAPI/Swagger configuration detected).

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to identify code relevant to the pagination instability investigation:

- **Document list API:** `src/documents/views.py` — `DocumentViewSet` (lines 172–360), `UnifiedSearchViewSet` (lines 377–426)
- **Pagination class:** `src/paperless/views.py` — `StandardPagination` (lines 8–11), subclass of `PageNumberPagination` with `page_size=25`, `max_page_size=100000`
- **Document model ordering:** `src/documents/models.py` — `Document.Meta.ordering = ("-created",)` (line 208)
- **Filter infrastructure:** `src/documents/filters.py` — `DocumentFilterSet` (lines 81–118), `TagsFilter` (lines 36–60), `InboxFilter` (lines 63–70), `TitleContentFilter` (lines 73–78)
- **Serializers:** `src/documents/serialisers.py` — `DocumentSerializer` (lines 201–235)
- **Search index pagination:** `src/documents/index.py` — `DelayedQuery.__getitem__` (lines 203–237), Whoosh `search_page` pagination
- **URL routing:** `src/paperless/urls.py` — confirms `UnifiedSearchViewSet` is registered at `documents` endpoint (line 32)
- **DRF configuration:** `src/paperless/settings.py` — `REST_FRAMEWORK` dict (lines 116–131), no `DEFAULT_PAGINATION_CLASS` or `DEFAULT_FILTER_BACKENDS` at framework level
- **Frontend list service:** `src-ui/src/app/services/document-list-view.service.ts` — manages pagination state, sort, filters, reload cycle
- **Frontend REST service:** `src-ui/src/app/services/rest/document.service.ts` — `listFiltered()` method, `filterRulesToQueryParams()`, sort-field constants
- **Frontend base service:** `src-ui/src/app/services/rest/abstract-paperless-service.ts` — `list()` method assembling `HttpParams` (page, page_size, ordering)
- **Filter rule types:** `src-ui/src/app/data/filter-rule-type.ts` — 23 filter rule type definitions mapping frontend filter IDs to backend query parameters

**Key directories examined:**
- `src/documents/` — Core Django app (models, views, filters, serialisers, index, signals)
- `src/paperless/` — Project settings, URL routing, pagination class
- `src-ui/src/app/services/` — Angular services for document list state and REST communication
- `src-ui/src/app/data/` — TypeScript data models and filter rule type definitions
- `docs/` — Sphinx documentation source tree

**Related documentation found:**
- `docs/api.rst` — Existing API reference; does not document pagination mechanics, ordering stability, or filter interaction with pagination
- `docs/usage_overview.rst` — User-facing overview; no mention of known pagination behaviors
- `docs/troubleshooting.rst` — Troubleshooting guide; does not address list-view pagination issues

### 0.2.3 Web Search Research Conducted

No web search was required for this investigation. All findings are derived from direct code analysis. The root cause — non-deterministic ordering in SQL combined with `OFFSET`/`LIMIT` pagination — is a well-established database behavior pattern that does not require external validation. The relevant DRF pagination mechanics (`PageNumberPagination` using `Paginator` which issues `LIMIT`/`OFFSET` queries) are confirmed directly from the source code and the pinned `djangorestframework==3.13.1` dependency.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation document must trace through the following modules, each contributing a piece of the root-cause chain:

- **Module: `src/documents/views.py` — DocumentViewSet**
  - Public APIs: `get_queryset()` (line 199), `get_serializer()` (lines 201–210), inherited `list()` from `ListModelMixin`
  - Current documentation: `docs/api.rst` documents the endpoint but not its queryset construction or ordering behavior
  - Documentation needed: Explanation of how `.distinct()` is applied unconditionally, how `OrderingFilter` appends ordering, and how DRF's `PageNumberPagination` slices the result

- **Module: `src/documents/filters.py` — DocumentFilterSet and TagsFilter**
  - Public APIs: `TagsFilter.filter()` (lines 42–60), `InboxFilter.filter()` (lines 64–70), `TitleContentFilter.filter()` (lines 73–78)
  - Current documentation: Not documented beyond the DRF browsable API
  - Documentation needed: How `tags__id__all` chained `.filter(tags__id=X)` calls create multiple JOINs on the M2M table, how only `tags__id__in` explicitly calls `.distinct()`, and how the queryset-level `.distinct()` from `get_queryset()` interacts with these JOINs

- **Module: `src/documents/models.py` — Document model**
  - Key fields: `created` (DateTimeField, `db_index=True`), `Meta.ordering = ("-created",)` (line 208), `tags` (ManyToManyField)
  - Current documentation: Field descriptions in `docs/api.rst`
  - Documentation needed: Explanation of why `("-created",)` without a secondary key like `("-created", "-id")` creates non-deterministic ordering for tied timestamps

- **Module: `src/paperless/views.py` — StandardPagination**
  - Configuration: `page_size=25`, `page_size_query_param="page_size"`, `max_page_size=100000`
  - Current documentation: Not documented
  - Documentation needed: Explanation of how DRF `PageNumberPagination` translates page numbers to SQL `OFFSET`/`LIMIT`

- **Module: `src/documents/index.py` — DelayedQuery (Whoosh search pagination)**
  - Key logic: `__getitem__` (lines 203–237) uses `searcher.search_page()` with Whoosh-native pagination
  - Current documentation: `docs/api.rst` mentions search pagination briefly
  - Documentation needed: Contrast with ORM-based pagination to clarify that this separate path has its own ordering semantics

- **Module: `src-ui/src/app/services/document-list-view.service.ts` — DocumentListViewService**
  - Key logic: `reload()` (lines 133–184), `currentPage` setter (lines 231–235), `getNext()`/`getPrevious()` (lines 298–342)
  - Current documentation: Not documented externally
  - Documentation needed: How the frontend assumes stable page boundaries and how cross-page navigation relies on that assumption

- **Module: `src-ui/src/app/services/rest/abstract-paperless-service.ts` — list() method**
  - Key logic: `list()` (lines 32–58) assembles `page`, `page_size`, `ordering` query parameters
  - Current documentation: Not documented externally
  - Documentation needed: How the `ordering` parameter is constructed and passed to the backend

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps are identified:

- **Undocumented behavior — ordering stability:** No documentation anywhere in the repository explains that the default sort order `("-created",)` is non-deterministic for documents sharing the same `created` timestamp. This is the primary root cause of the observed pagination instability.
- **Undocumented behavior — `.distinct()` on every document list query:** The unconditional `.distinct()` in `get_queryset()` (line 199 of `views.py`) is not documented or explained. It exists to prevent duplicate rows from M2M JOINs but interacts with non-deterministic ordering.
- **Undocumented behavior — tag filter JOIN mechanics:** The `TagsFilter` class's behavior of chaining `.filter(tags__id=X)` (which creates multiple JOINs) versus using `.filter(tags__id__in=X)` (which creates a single JOIN) is not documented.
- **Missing investigation document:** No existing document in the repository investigates or explains the observed pagination anomalies.
- **Undocumented limitation — absence of object-level permissions:** The user's belief that visibility is "shaped by sharing rules" is contradicted by the code, which shows only `IsAuthenticated` permission classes. This gap between user expectation and codebase reality is not documented.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single self-contained markdown investigation document. Its internal structure is designed to walk the reader from symptom through mechanism to root cause, with code-grounded evidence at every step.

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── Title and Summary
        ├── 1. Observed Symptoms (restating the user's report)
        ├── 2. Architecture of the Document List Pipeline
        │   ├── 2.1 Backend: ViewSet → Queryset → Filters → Pagination
        │   ├── 2.2 Frontend: DocumentListViewService → API → UI
        │   └── 2.3 Search Path: UnifiedSearchViewSet → Whoosh
        ├── 3. Root Cause Analysis
        │   ├── 3.1 Non-Deterministic Ordering on Tied Sort Keys
        │   ├── 3.2 DISTINCT + OFFSET/LIMIT Interaction
        │   ├── 3.3 Tag Filter JOIN Amplification
        │   └── 3.4 The Sharing Rules Misconception
        ├── 4. Reproducing the Condition
        │   ├── 4.1 Minimum Conditions for Instability
        │   └── 4.2 Observation Strategy (Temporary Scripts)
        ├── 5. Conclusions and Recommendations
        └── 6. Source References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the queryset construction chain from `src/documents/views.py` line 199, tracing through `DjangoFilterBackend`, `SearchFilter`, `OrderingFilter`, and finally `StandardPagination`
- Extract the `TagsFilter.filter()` logic from `src/documents/filters.py` lines 42–60 to demonstrate JOIN behavior
- Extract `Document.Meta.ordering` from `src/documents/models.py` line 208 to demonstrate the single-key sort
- Extract `StandardPagination` configuration from `src/paperless/views.py` lines 8–11
- Extract the frontend `reload()` flow from `src-ui/src/app/services/document-list-view.service.ts` lines 133–184
- Trace the `list()` parameter assembly from `src-ui/src/app/services/rest/abstract-paperless-service.ts` lines 32–58
- Confirm absence of object-level permissions by auditing every `permission_classes` declaration in `src/documents/views.py`

**Documentation Standards:**

- Markdown formatting with hierarchical headers (`#`, `##`, `###`)
- Mermaid diagram showing the query pipeline from HTTP request through queryset to SQL
- Inline code references in the format `Source: path/to/file.py:LineNumber`
- Tables for structured comparisons (e.g., filter behaviors, sort key analysis)
- All conclusions must reference specific files and line numbers as evidence

### 0.4.3 Diagram and Visual Strategy

The investigation document will include the following Mermaid diagrams:

- **Query pipeline flowchart:** Shows the progression from `DocumentViewSet.get_queryset()` through `DjangoFilterBackend` → `OrderingFilter` → `StandardPagination`, highlighting where `.distinct()` is applied and where `OFFSET`/`LIMIT` is injected
- **Tag filter JOIN diagram:** Illustrates how `tags__id__all=6,7` generates two separate JOINs on the `documents_document_tags` M2M table, and how `.distinct()` collapses the resulting duplicates
- **Frontend-backend interaction sequence:** Shows the `DocumentListViewService.reload()` → `DocumentService.listFiltered()` → HTTP GET → DRF pipeline → SQL query → response flow, highlighting the assumption of stable page boundaries


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/views.py`, `src/documents/filters.py`, `src/documents/models.py`, `src/paperless/views.py`, `src/documents/index.py`, `src-ui/src/app/services/document-list-view.service.ts`, `src-ui/src/app/services/rest/document.service.ts`, `src-ui/src/app/services/rest/abstract-paperless-service.ts`, `src-ui/src/app/data/filter-rule-type.ts` | Complete investigative analysis answering why document list pagination is unstable: root cause (non-deterministic ordering on tied `created` timestamps without tiebreaker), mechanism (DISTINCT + OFFSET/LIMIT with non-unique ORDER BY), tag-filter JOIN amplification, absence of per-document sharing rules, and observation strategy |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Root-Cause Analysis
Source Code:
    - src/documents/views.py (DocumentViewSet.get_queryset, ordering_fields, filter_backends)
    - src/documents/filters.py (TagsFilter, DocumentFilterSet, InboxFilter)
    - src/documents/models.py (Document.Meta.ordering, tags M2M, created field)
    - src/paperless/views.py (StandardPagination configuration)
    - src/documents/index.py (DelayedQuery, Whoosh search_page)
    - src/paperless/urls.py (UnifiedSearchViewSet registered at /documents/)
    - src/paperless/settings.py (REST_FRAMEWORK configuration)
    - src-ui/src/app/services/document-list-view.service.ts (reload, pagination state)
    - src-ui/src/app/services/rest/document.service.ts (listFiltered, filterRulesToQueryParams)
    - src-ui/src/app/services/rest/abstract-paperless-service.ts (list method, ordering param)
    - src-ui/src/app/data/filter-rule-type.ts (FILTER_RULE_TYPES, filter variable mappings)
Sections:
    - Observed Symptoms (restatement of user report)
    - Architecture of the Document List Pipeline
        - Backend pipeline: ViewSet → Queryset → Filters → Ordering → Pagination
        - Frontend pipeline: DocumentListViewService → DocumentService → HTTP
        - Search path: UnifiedSearchViewSet → Whoosh index
    - Root Cause Analysis
        - Non-deterministic ordering on tied created timestamps
        - DISTINCT + OFFSET/LIMIT interaction
        - Tag filter JOIN amplification with chained .filter() calls
        - The sharing rules misconception (absence of object-level permissions)
    - Reproducing the Condition
        - Minimum conditions for instability
        - Observation strategy using temporary scripts
    - Conclusions and Recommendations
    - Source References
Diagrams:
    - Query pipeline flowchart (Mermaid)
    - Tag filter JOIN amplification diagram (Mermaid)
    - Frontend-backend pagination sequence (Mermaid)
Key Citations:
    - src/documents/views.py:199 (get_queryset with .distinct())
    - src/documents/models.py:208 (Meta.ordering = ("-created",))
    - src/documents/filters.py:42-60 (TagsFilter.filter)
    - src/paperless/views.py:8-11 (StandardPagination)
    - src-ui/src/app/services/document-list-view.service.ts:133-184 (reload)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone markdown document in `blitzy/documentation/` which is not managed by the Sphinx documentation build system. The existing `docs/conf.py`, `docs/Makefile`, and `.readthedocs.yml` remain unchanged.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content dependencies:** The output document is self-contained and does not reference or include content from other documentation files.
- **No navigation link updates:** The document is placed in `blitzy/documentation/`, outside the Sphinx `docs/` tree, so no table of contents or sidebar updates are needed.
- **Source code cross-references:** The document will reference source files by path and line number but does not hyperlink to them. All references are textual citations.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The investigation document requires no documentation tooling beyond a standard markdown renderer. The following table lists the project dependencies directly relevant to the investigation analysis, as confirmed from `Pipfile` and `requirements.txt`:

| Registry | Package Name | Version | Purpose (in context of investigation) |
|----------|--------------|---------|---------------------------------------|
| pip | django | 4.0.4 | ORM queryset construction, `DISTINCT`, `ORDER BY`, `OFFSET`/`LIMIT` SQL generation |
| pip | djangorestframework | 3.13.1 | `PageNumberPagination`, `OrderingFilter`, `DjangoFilterBackend`, viewset lifecycle |
| pip | django-filter | 21.1 | `FilterSet`, custom `Filter` subclasses (`TagsFilter`, `InboxFilter`, `TitleContentFilter`) |
| pip | whoosh | 2.7.4 | Full-text search index, `search_page` pagination, `sortedby` field ordering |
| npm | @angular/core | 13.3.4 | Frontend framework; `DocumentListViewService` and component lifecycle |
| npm | @ng-bootstrap/ng-bootstrap | (from `package.json`) | `ngb-pagination` component used for document list pagination UI |
| npm | rxjs | (from `package.json`) | Observable-based data flow for `DocumentService.listFiltered()` and `reload()` |

All versions are taken from the pinned dependency manifests (`requirements.txt` for Python, `package.json` / `package-lock.json` for Node.js). No version placeholders or assumptions are used.

### 0.6.2 Documentation Reference Updates

Not applicable. The output document is a new standalone file. No existing documentation files require link updates, and no link transformation rules apply.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The investigation document targets 100% coverage of the user's questions. The following checklist maps each user question to the specific section and evidence that must address it:

| User Question | Target Section | Source Evidence | Coverage Status |
|---------------|---------------|-----------------|-----------------|
| Is the backend producing duplicates that get collapsed somewhere later? | Root Cause Analysis §3.2 | `src/documents/views.py:199` (`.distinct()`), `src/documents/filters.py:52` (`TagsFilter` with `in_list`), `src/documents/filters.py:54-58` (`TagsFilter` without `in_list`) | Required |
| Is pagination happening before any deduplication? | Root Cause Analysis §3.2 | `src/paperless/views.py:8-11` (`StandardPagination`), DRF `PageNumberPagination` → Django `Paginator` → SQL `DISTINCT` before `LIMIT/OFFSET` | Required |
| Is the ordering quietly unstable when multiple rows tie on the primary sort key? | Root Cause Analysis §3.1 | `src/documents/models.py:208` (`Meta.ordering = ("-created",)`), `src/documents/views.py:187-196` (`ordering_fields`), absence of secondary sort key | Required |
| Does the glitch depend on what the user is allowed to see (sharing rules)? | Root Cause Analysis §3.4 | `src/documents/views.py:183` (`permission_classes = (IsAuthenticated,)`), absence of any `ObjectPermission` or owner-based filter | Required |
| What does the API actually return across consecutive page requests? | Reproducing the Condition §4.1–4.2 | `src-ui/src/app/services/rest/abstract-paperless-service.ts:32-58` (`list()` method), `src-ui/src/app/services/document-list-view.service.ts:133-184` (`reload()`) | Required |
| What does the UI think pagination means? | Architecture §2.2 | `src-ui/src/app/services/document-list-view.service.ts:73` (`currentPageSize`), `src-ui/src/app/components/document-list/document-list.component.html` (`ngb-pagination`) | Required |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must be addressed with a clear, evidence-based answer
- Every conclusion must cite at least one specific source file and line number
- The document must include a definitive identification of the root cause
- The document must address the sharing-rules observation, even if the answer is that the assumption is incorrect

**Accuracy validation:**
- All file paths and line numbers must be verified against the current repository state
- All code behavior claims must be traceable to specific source lines
- No speculative claims — if a behavior cannot be confirmed from code, it must be explicitly flagged as uncertain

**Clarity standards:**
- The document must be readable by a developer who has not previously examined the codebase
- Technical explanations must proceed from high-level architecture to specific implementation details
- Diagrams must illustrate the query pipeline and data flow, not merely decorate the document

**Maintainability:**
- All source references include file path and line number for traceability
- The document is self-contained with no external dependencies

### 0.7.3 Example and Diagram Requirements

- Minimum 3 Mermaid diagrams: query pipeline, tag-filter JOIN amplification, frontend-backend sequence
- Minimum 1 example SQL query showing the `DISTINCT ... ORDER BY created DESC LIMIT 25 OFFSET 25` pattern
- Minimum 1 example scenario showing how two documents with the same `created` timestamp can swap positions across page requests
- Minimum 1 conceptual observation script outline (curl-based or Python-based) for watching consecutive page API responses


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/paperless-ngx_542221a38dff.md` — the complete investigation document

- **Code modules analyzed (read-only, for evidence gathering):**
  - `src/documents/views.py` — `DocumentViewSet`, `UnifiedSearchViewSet`, filter/ordering/pagination configuration
  - `src/documents/filters.py` — `DocumentFilterSet`, `TagsFilter`, `InboxFilter`, `TitleContentFilter`
  - `src/documents/models.py` — `Document` model, `Meta.ordering`, field definitions, M2M tags relationship
  - `src/documents/serialisers.py` — `DocumentSerializer`, `DynamicFieldsModelSerializer`
  - `src/documents/index.py` — `DelayedQuery`, `DelayedFullTextQuery`, Whoosh search pagination
  - `src/paperless/views.py` — `StandardPagination` class
  - `src/paperless/urls.py` — URL routing, `UnifiedSearchViewSet` registration
  - `src/paperless/settings.py` — `REST_FRAMEWORK` configuration, database backend selection
  - `src-ui/src/app/services/document-list-view.service.ts` — pagination state, reload logic, page navigation
  - `src-ui/src/app/services/rest/document.service.ts` — `listFiltered()`, `filterRulesToQueryParams()`
  - `src-ui/src/app/services/rest/abstract-paperless-service.ts` — `list()` method, ordering parameter construction
  - `src-ui/src/app/data/filter-rule-type.ts` — filter rule type definitions
  - `src-ui/src/app/data/filter-rule.ts` — `FilterRule` interface, `cloneFilterRules()`, `isFullTextFilterRule()`
  - `src-ui/src/app/components/document-list/document-list.component.html` — pagination template (`ngb-pagination`)
  - `src-ui/src/app/components/document-list/document-list.component.ts` — component lifecycle, sort/filter bindings

- **Dependency manifests analyzed (read-only):**
  - `Pipfile` — Python dependency declarations
  - `requirements.txt` — Pinned Python dependency versions
  - `src-ui/package.json` — Node.js dependency declarations

- **Existing documentation analyzed (read-only):**
  - `docs/api.rst` — existing API documentation
  - `docs/conf.py` — Sphinx configuration
  - `.readthedocs.yml` — Read the Docs hosting configuration

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No changes to any existing file in the repository. The implementation rule `SWE-AtlasQnA-Repo` explicitly prohibits this.
- **Test file modifications:** No changes to test files.
- **Fix implementation:** The document identifies the root cause and recommends solutions, but does not implement any code changes (e.g., adding a secondary sort key to `Meta.ordering`).
- **Deployment configuration changes:** No changes to Docker, Supervisord, or CI/CD configurations.
- **Sphinx documentation updates:** The output is not integrated into the Sphinx `docs/` tree.
- **Frontend component changes:** No changes to Angular components, services, or templates.
- **Database schema changes:** No migrations or model modifications.
- **Persistent observation scripts:** The document may describe conceptual observation scripts, but these are not committed to the repository. Any temporary scripts used during investigation must be cleaned up.
- **Unrelated documentation:** Features not related to document list pagination (e.g., mail ingestion, OCR, barcode splitting) are out of scope.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not part of the Sphinx build.
- **Documentation preview command:** Any Markdown renderer (e.g., `grip`, VS Code preview, GitHub web UI) can render the output file.
- **Diagram generation command:** Mermaid diagrams are embedded inline in the markdown. They can be rendered by any Mermaid-aware viewer (GitHub, GitLab, Mermaid CLI `mmdc`).
- **Documentation deployment command:** Not applicable — the file is committed to `blitzy/documentation/` in the destination repository.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every technical claim must reference a source file path and line number.
- **Style guide:** The document follows the existing repository convention of clear prose with inline code references, consistent with `CONTRIBUTING.md` and `docs/api.rst` style patterns. Headers use `#`/`##`/`###` hierarchy. Code blocks use triple-backtick fencing with language identifiers.
- **Documentation validation:** Markdown lint (`markdownlint`) can be used for structural validation. Mermaid syntax can be validated with `mmdc --input file.md`.


## 0.10 Rules for Documentation

The following rules are explicitly specified by the user or derived from the project's implementation rules:

- **Do not modify any existing files in the source repository.** The output is exclusively the new markdown document at `blitzy/documentation/paperless-ngx_542221a38dff.md`. (Source: `SWE-AtlasQnA-Repo` implementation rule)
- **Provide thinking and rationale behind the answers.** The document must walk through the reasoning chain — not merely state conclusions — so the reader can follow the logic from symptom to root cause. (Source: `SWE-AtlasQnA-Repo` implementation rule)
- **Do not make assumptions; base answers on the code as the truth.** Every assertion must be grounded in a specific file and line number. If the code does not confirm a behavior, the document must say so explicitly. (Source: `SWE-AtlasQnA-Repo` implementation rule)
- **Place the generated document in the `blitzy/documentation` directory.** The filename must be `paperless-ngx_542221a38dff.md`, matching the source branch name. (Source: `SWE-AtlasQnA-Repo` implementation rule)
- **The repository itself should remain unchanged.** Temporary scripts may be discussed conceptually for observation purposes, but nothing is committed beyond the investigation document. (Source: user's explicit instruction)
- **Anything temporary should be cleaned up afterward.** If observation scripts are described, the document must note that they are ephemeral and not part of the repository state. (Source: user's explicit instruction)


## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Backend — Django/Python:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `src/documents/views.py` | `DocumentViewSet` (queryset, ordering_fields, filter_backends, pagination_class), `UnifiedSearchViewSet` (search path), permission classes (`IsAuthenticated` on all viewsets) |
| `src/documents/filters.py` | `DocumentFilterSet` (field filters), `TagsFilter` (M2M JOIN behavior with/without `.distinct()`), `InboxFilter`, `TitleContentFilter` |
| `src/documents/models.py` | `Document` model (`Meta.ordering = ("-created",)`, field definitions, `tags` M2M, `created`/`modified`/`added` DateTimeFields), `SavedView`, `SavedViewFilterRule` (23 rule types) |
| `src/documents/serialisers.py` | `DocumentSerializer` (field list), `DynamicFieldsModelSerializer` (field selection) |
| `src/documents/index.py` | `DelayedQuery.__getitem__` (Whoosh `search_page` pagination), `_get_query_sortedby` (sort field mapping), schema definition |
| `src/paperless/views.py` | `StandardPagination` (`PageNumberPagination`, `page_size=25`, `max_page_size=100000`) |
| `src/paperless/urls.py` | `UnifiedSearchViewSet` registered at `documents` endpoint, URL routing |
| `src/paperless/settings.py` | `REST_FRAMEWORK` configuration (authentication, versioning), database backend selection |
| `Pipfile` | Python dependency versions: `django ~=4.0`, `djangorestframework ~=3.13`, `django-filter ~=21.1`, `whoosh ~=2.7.4` |
| `requirements.txt` | Pinned versions: `django==4.0.4`, `djangorestframework==3.13.1`, `django-filter==21.1`, `whoosh==2.7.4` |

**Frontend — Angular/TypeScript:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `src-ui/src/app/services/document-list-view.service.ts` | `DocumentListViewService` (pagination state, `reload()` logic, page navigation, sort/filter state management, `defaultListViewState()`) |
| `src-ui/src/app/services/rest/document.service.ts` | `DocumentService` (`listFiltered()`, `filterRulesToQueryParams()`, `DOCUMENT_SORT_FIELDS` constants) |
| `src-ui/src/app/services/rest/abstract-paperless-service.ts` | `AbstractPaperlessService.list()` (HTTP parameter assembly: page, page_size, ordering) |
| `src-ui/src/app/data/filter-rule-type.ts` | `FILTER_RULE_TYPES` array (23 filter rules mapping frontend IDs to backend query parameters) |
| `src-ui/src/app/data/filter-rule.ts` | `FilterRule` interface, `cloneFilterRules()`, `isFullTextFilterRule()` |
| `src-ui/src/app/components/document-list/document-list.component.html` | Pagination template (`ngb-pagination` bound to `list.currentPage`, `list.collectionSize`) |
| `src-ui/src/app/components/document-list/document-list.component.ts` | Component lifecycle, sort event handling, filter synchronization |
| `src-ui/package.json` | Frontend dependency declarations (Angular 13.3.4, ng-bootstrap, RxJS) |

**Documentation:**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `docs/api.rst` | Existing API documentation — confirmed no mention of pagination stability or ordering determinism |
| `docs/conf.py` | Sphinx configuration — confirmed documentation framework |
| `.readthedocs.yml` | Hosting configuration — confirmed docs deployment target |
| `docs/Makefile` | Build targets — confirmed Sphinx build system |

**Folders explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| `/` (root) | Level 0 | Repository structure, build files, governance docs |
| `src/` | Level 1 | Django source tree structure, app packages |
| `src/documents/` | Level 2 | Core domain app — models, views, filters, serializers, index, signals, tests, migrations |
| `src/paperless/` | Level 2 | Project settings, URL routing, pagination class, authentication |
| `src-ui/` | Level 1 | Angular workspace, package manifests |
| `src-ui/src/app/services/` | Level 3 | Document list view service, REST services |
| `src-ui/src/app/data/` | Level 3 | Data models, filter rule types |
| `src-ui/src/app/components/document-list/` | Level 3 | Document list component (template, class, styles) |
| `docs/` | Level 1 | Sphinx documentation source tree |

### 0.11.2 Attachments

No attachments were provided for this project. No Figma URLs or design files are referenced.

### 0.11.3 External References

No external URLs, web search results, or third-party documentation were required. All analysis is derived from the repository source code.


