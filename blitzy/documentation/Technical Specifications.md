# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative reference document** (`paperless-ngx_542221a38dff.md`) that comprehensively answers a series of interconnected questions about how the Paperless-NGX search index synchronization mechanism works at runtime, with all answers grounded exclusively in analysis of the source code.

- **Category:** Create new documentation
- **Documentation Type:** Technical investigation / Q&A analysis document

The user seeks deep, code-backed answers to the following questions:

- **Q1 – API-driven title change and search visibility:** When a document's title is changed through the REST API, does the updated title appear in Whoosh search results immediately within the same HTTP request cycle, or does a background worker process the index update asynchronously?
- **Q2 – Background worker involvement during API updates:** What happens in the Django-Q (`qcluster`) background task worker process when a document is modified via the API — does a task get enqueued, or does the index update complete entirely within the API request?
- **Q3 – Direct SQL modification and index staleness:** If a document's title is changed directly in the database via raw SQL (bypassing Django ORM signals), does the search index remain stale? Can the document be found by the new title?
- **Q4 – Forced index reconciliation:** Is there a mechanism to force the system to reconcile the Whoosh index with the current database state, and what observable effect does this have on the worker logs and task queue?
- **Q5 – Index corruption and self-healing:** If the Whoosh index files are deleted or corrupted while documents exist in the database, what recovery mechanism exists, how long does reconstruction take for a small document set, and can the system self-heal or is manual intervention required?

### 0.1.2 Special Instructions and Constraints

- **No source code modifications:** The user explicitly requires that no existing source files in the repository be modified. The deliverable is exclusively a new markdown document.
- **Test artifact cleanup:** Any temporary test documents created for observation must be cleaned up when finished.
- **Implementation rule compliance:** Per the project rule `SWE-AtlasQnA-Repo`, the document must be placed in the `blitzy/documentation` directory and named `paperless-ngx_542221a38dff.md`.
- **Evidence-based answers:** All answers must be derived from the source code with rationale provided — no assumptions permitted.
- **Thinking/rationale required:** The document must provide the reasoning chain behind each answer, tracing the code path from API endpoint through signal handlers and index writers.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer Q1** (API title update → search visibility), we will trace the code path from `DocumentViewSet.update()` in `src/documents/views.py` (line 212) through `index.add_or_update_document()` in `src/documents/index.py` (line 118), documenting that the Whoosh index update uses `AsyncWriter` but executes synchronously within the HTTP request handler, not via background task.
- To **answer Q2** (background worker involvement), we will analyze the `DocumentViewSet.update()` method to confirm it calls `index.add_or_update_document()` directly (not via `async_task`), contrasting this with the `bulk_edit.py` bulk operations which do enqueue `async_task("documents.tasks.bulk_update_documents", ...)`.
- To **answer Q3** (direct SQL bypass), we will document that Django's `post_save` signal and the explicit `index.add_or_update_document()` call in the view are the only index update triggers — raw SQL bypasses both, leaving the Whoosh index stale.
- To **answer Q4** (forced reconciliation), we will document the `manage.py document_index reindex` command from `src/documents/management/commands/document_index.py`, which calls `index_reindex()` in `src/documents/tasks.py` (line 38), recreating the entire Whoosh index from all `Document` objects.
- To **answer Q5** (index corruption/self-healing), we will document the `open_index()` function in `src/documents/index.py` (line 52) which catches exceptions during index open and recreates the index, plus the `docker/docker-prepare.sh` startup script which checks an index version file and triggers reindex on version mismatch.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Signal handler registration context:** The `add_to_index` handler in `src/documents/signals/handlers.py` (line 428) is only connected to the `document_consumption_finished` signal (not `post_save`), meaning it only fires during initial document ingestion — not during API-driven updates. This distinction is critical and must be documented.
- **Whoosh AsyncWriter semantics:** The `AsyncWriter` used in `src/documents/index.py` (line 66) is a Whoosh concurrency mechanism (not an async-task mechanism). It operates synchronously within the calling thread but uses file locking internally for safe concurrent writes. This must be clarified in the document.
- **Contrast between API update and bulk edit paths:** The API `update()` path synchronously updates the index, while `bulk_edit.py` operations use `async_task` for deferred index updates. This architectural difference needs documentation.
- **Startup-time index version tracking:** The `docker/docker-prepare.sh` script (line 49-57) maintains an `.index_version` sentinel file and triggers a full reindex when the version changes — this is the primary self-healing mechanism and must be documented.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with comprehensive coverage of user guides, administration, API reference, and configuration, hosted on Read the Docs.

- **Documentation framework:** Sphinx ~4.5.0 (from `Pipfile` dev-packages)
- **Documentation generator configuration:** `docs/conf.py` — uses `sphinx_rtd_theme`, enables autodoc, intersphinx, todo, imgmath, and viewcode extensions
- **Documentation source format:** reStructuredText (`.rst`) files in the `docs/` directory
- **Documentation hosting:** Read the Docs, configured via `.readthedocs.yml` (Python 3.8 build, Sphinx builder)
- **Documentation build mechanism:** `docs/Makefile` (Sphinx standard targets: html, epub, latexpdf, linkcheck, doctest) and `docs/Dockerfile` (containerized build + HTTP server on port 8000)
- **Existing documentation files:**

| File | Content Domain | Relevance to This Task |
|------|---------------|----------------------|
| `docs/administration.rst` | Backups, updates, index management, utilities | **High** — contains `document_index` reindex/optimize reference |
| `docs/api.rst` | REST API endpoints, search query parameters | **High** — documents search endpoint and full-text query syntax |
| `docs/usage_overview.rst` | Ingestion, search workflow, recommended patterns | **High** — documents Whoosh query language and search behavior |
| `docs/configuration.rst` | All environment variables and runtime settings | **Medium** — references `PAPERLESS_TASK_WORKERS` and index/data paths |
| `docs/extending.rst` | Development setup, parser extensions | **Low** — not directly about search index sync |
| `docs/setup.rst` | Installation, migration, deployment modes | **Low** — deployment-level information only |
| `docs/troubleshooting.rst` | Common operational failures and fixes | **Medium** — potential index issues |

### 0.2.2 Repository Code Analysis for Documentation

The following code files form the basis of all answers in the documentation deliverable:

**Primary search/index code analyzed:**
- `src/documents/index.py` — Whoosh schema, `open_index()`, `open_index_writer()`, `update_document()`, `add_or_update_document()`, `remove_document_from_index()`, `DelayedFullTextQuery`, `DelayedMoreLikeThisQuery`, `autocomplete()`
- `src/documents/tasks.py` — `index_reindex()` (full rebuild), `index_optimize()`, `bulk_update_documents()`, `consume_file()`
- `src/documents/views.py` — `DocumentViewSet.update()` (line 212-217, synchronous index update), `UnifiedSearchViewSet` (search query routing), `PostDocumentView` (async ingestion via `async_task`)
- `src/documents/management/commands/document_index.py` — CLI command wrapping `index_reindex()` and `index_optimize()`

**Signal and lifecycle code analyzed:**
- `src/documents/signals/__init__.py` — Defines `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` signals
- `src/documents/signals/handlers.py` — `add_to_index()` (line 428), `update_filename_and_move_files()` (post_save handler for file moves), `cleanup_document_deletion()` (post_delete handler)
- `src/documents/apps.py` — `DocumentsConfig.ready()` connects `add_to_index` to `document_consumption_finished` only
- `src/documents/consumer.py` — `Consumer.try_consume_file()` fires `document_consumption_finished` signal which triggers `add_to_index`

**Bulk operation code analyzed:**
- `src/documents/bulk_edit.py` — All bulk operations (`set_correspondent`, `set_document_type`, `add_tag`, `remove_tag`, `modify_tags`, `delete`) — each queues `async_task("documents.tasks.bulk_update_documents", ...)` for deferred index updates
- `src/documents/admin.py` — `DocumentAdmin.save_model()` directly calls `index.add_or_update_document()`, `delete_model()` calls `index.remove_document_from_index()`

**Infrastructure and configuration code analyzed:**
- `src/paperless/settings.py` — `INDEX_DIR` (line 73: `os.path.join(DATA_DIR, "index")`), `Q_CLUSTER` configuration (lines 449-457), `DATA_DIR`, `TASK_WORKERS`
- `docker/supervisord.conf` — Three supervised processes: gunicorn, document_consumer, qcluster (scheduler)
- `docker/docker-prepare.sh` — Startup index version check and conditional reindex (lines 49-57)

### 0.2.3 Web Search Research Conducted

No web search is required for this documentation task. All answers are derived exclusively from source code analysis, per the user's explicit instruction to base answers on "the code as the truth." The Whoosh library's `AsyncWriter` semantics are well-documented in the codebase's usage patterns and do not require external verification to answer the user's specific questions.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The deliverable document must trace through and explain the following modules in detail:

- **Module: `src/documents/index.py`**
  - Public APIs: `get_schema()`, `open_index(recreate)`, `open_index_writer(optimize)`, `open_index_searcher()`, `update_document(writer, doc)`, `remove_document(writer, doc)`, `remove_document_by_id(writer, doc_id)`, `add_or_update_document(document)`, `remove_document_from_index(document)`, `autocomplete(ix, term, limit)`, `DelayedFullTextQuery`, `DelayedMoreLikeThisQuery`
  - Current documentation: Existing `docs/administration.rst` mentions `document_index reindex/optimize` commands briefly. No detailed explanation of the synchronous update mechanism exists.
  - Documentation needed: Detailed explanation of synchronous vs asynchronous index update paths, `AsyncWriter` semantics, and the Whoosh schema fields

- **Module: `src/documents/views.py` — `DocumentViewSet`**
  - Endpoints: `PUT /api/documents/<pk>/` (update), `DELETE /api/documents/<pk>/` (destroy)
  - Current documentation: `docs/api.rst` documents the endpoint fields and search parameters but does not describe the index update behavior during PUT operations
  - Documentation needed: Explanation that `update()` (line 212) calls `index.add_or_update_document()` synchronously after `super().update()` returns

- **Module: `src/documents/tasks.py`**
  - Public APIs: `index_reindex(progress_bar_disable)`, `index_optimize()`, `bulk_update_documents(document_ids)`, `consume_file(...)`
  - Current documentation: `docs/administration.rst` (lines 337-358) references the CLI command but lacks explanation of what happens internally
  - Documentation needed: Explanation of full rebuild via `open_index(recreate=True)` plus iteration over all Document objects

- **Module: `src/documents/signals/handlers.py`**
  - Public APIs: `add_to_index(sender, document, **kwargs)`, `update_filename_and_move_files(sender, instance, **kwargs)`, `cleanup_document_deletion(sender, instance, using, **kwargs)`
  - Current documentation: None — signal handler behavior is not documented anywhere
  - Documentation needed: Explanation that `add_to_index` connects only to `document_consumption_finished` (not `post_save`), meaning it is NOT triggered by API updates

- **Module: `src/documents/bulk_edit.py`**
  - Public APIs: `set_correspondent()`, `set_document_type()`, `add_tag()`, `remove_tag()`, `modify_tags()`, `delete()`
  - Current documentation: Not documented regarding index update behavior
  - Documentation needed: Explanation that bulk operations enqueue `async_task` for deferred index updates (contrast with synchronous API update path)

- **Module: `docker/docker-prepare.sh`**
  - Public APIs: `search_index()` function (lines 49-57)
  - Current documentation: Not explicitly documented
  - Documentation needed: Explanation of the index version sentinel file and startup-time conditional reindex

- **Module: `src/documents/management/commands/document_index.py`**
  - Public APIs: `Command.handle()` with `reindex` and `optimize` subcommands
  - Current documentation: Briefly mentioned in `docs/administration.rst` (lines 348-358)
  - Documentation needed: Full explanation of what reindex does internally (recreates index schema, iterates all documents, writes each to Whoosh)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist that the new document must address:

- **Undocumented behavior:** The synchronous index update path in `DocumentViewSet.update()` — nowhere in existing documentation does it explain that API PUT requests update the Whoosh index within the HTTP request cycle
- **Undocumented behavior:** The distinction between the `document_consumption_finished` signal (ingestion-time indexing) and the direct `index.add_or_update_document()` call (API-time indexing)
- **Undocumented behavior:** The consequence of bypassing Django ORM — no existing documentation warns that raw SQL changes will leave the search index stale
- **Undocumented behavior:** The container startup index version check in `docker/docker-prepare.sh` and its role as a partial self-healing mechanism
- **Undocumented behavior:** The `open_index()` function's exception handler that recreates a corrupted index on next access (line 56-57 in `index.py`)
- **Missing architectural explanation:** No existing document provides a unified view of all the code paths that can trigger a Whoosh index update (API update, document consumption, bulk edit, admin save, CLI reindex)

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single markdown document placed in the `blitzy/documentation/` directory. Its internal structure follows the Q&A investigation format required by the implementation rules:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── Overview (purpose and scope of investigation)
        ├── Architecture Context (Whoosh index, Django-Q, signals)
        ├── Q1: API Title Change and Search Visibility
        │   ├── Answer with code path trace
        │   └── Rationale with source citations
        ├── Q2: Background Worker Behavior During API Updates
        │   ├── Answer contrasting sync vs async paths
        │   └── Rationale with signal registration analysis
        ├── Q3: Direct SQL Modification and Index Staleness
        │   ├── Answer explaining ORM signal bypass
        │   └── Rationale with Django signal mechanics
        ├── Q4: Forced Index Reconciliation
        │   ├── Answer documenting document_index reindex
        │   └── Rationale with task function analysis
        ├── Q5: Index Corruption and Self-Healing
        │   ├── Answer documenting recovery mechanisms
        │   └── Rationale with startup script and error handling
        └── Summary (consolidated findings table)
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract index update code paths from `src/documents/views.py`, `src/documents/index.py`, `src/documents/signals/handlers.py`, and `src/documents/tasks.py` using direct code reading
- Extract signal registration patterns from `src/documents/apps.py` to prove which signals trigger indexing
- Extract startup behavior from `docker/docker-prepare.sh` for self-healing analysis
- Extract error recovery patterns from `src/documents/index.py` `open_index()` exception handler

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Source citations as inline references in the format `Source: /path/to/file.py:LineNumber`
- Short code snippets (2-3 lines) to illustrate key decision points in the code
- Tables for summarizing findings and comparing code paths
- Mermaid diagrams for visualizing the index update flow paths

### 0.4.3 Diagram and Visual Strategy

The document must include the following Mermaid diagrams:

- **Index Update Paths Diagram:** A flowchart showing all code paths that trigger Whoosh index updates — API update (synchronous), document consumption (signal-driven), bulk edit (async task), admin save (synchronous), CLI reindex (batch)
- **API Update Sequence Diagram:** A sequence diagram showing the exact call chain from `DocumentViewSet.update()` → `super().update()` → `index.add_or_update_document()` → `open_index_writer()` → `AsyncWriter.update_document()` → `AsyncWriter.commit()`
- **Direct SQL vs API Comparison Diagram:** A side-by-side flow showing what happens when a title is changed via API (both DB and index updated) vs. raw SQL (only DB updated, index stale)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/index.py`, `src/documents/views.py`, `src/documents/tasks.py`, `src/documents/signals/handlers.py`, `src/documents/apps.py`, `src/documents/bulk_edit.py`, `src/documents/consumer.py`, `src/documents/admin.py`, `src/documents/management/commands/document_index.py`, `docker/docker-prepare.sh`, `src/paperless/settings.py` | Comprehensive Q&A document answering all five questions about Whoosh search index synchronization, with code-path traces, rationale, and Mermaid diagrams |

**No existing files are modified.** This is strictly a CREATE operation for a single new file, per the `SWE-AtlasQnA-Repo` implementation rule that prohibits modification of existing repository files.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Q&A Reference
Source Code:
  - src/documents/index.py (Whoosh index operations, schema, writers)
  - src/documents/views.py:212-217 (DocumentViewSet.update with sync index call)
  - src/documents/views.py:377-426 (UnifiedSearchViewSet search flow)
  - src/documents/tasks.py:32-46 (index_optimize, index_reindex)
  - src/documents/tasks.py:270-280 (bulk_update_documents)
  - src/documents/signals/handlers.py:428-431 (add_to_index handler)
  - src/documents/apps.py:11-29 (signal registration in ready())
  - src/documents/consumer.py:306-311 (consumption signal emission)
  - src/documents/bulk_edit.py (all bulk ops enqueue async_task)
  - src/documents/admin.py:85-89 (admin save_model with index update)
  - src/documents/management/commands/document_index.py (CLI reindex/optimize)
  - docker/docker-prepare.sh:49-57 (startup index version check)
  - src/paperless/settings.py:73 (INDEX_DIR), 449-457 (Q_CLUSTER)
Sections:
  - Overview (scope, purpose, architecture context)
  - Architecture Context (Whoosh engine, Django-Q workers, signal system)
  - Q1: API Title Change and Search Visibility
  - Q2: Background Worker Behavior During API Updates
  - Q3: Direct SQL Modification and Index Staleness
  - Q4: Forced Index Reconciliation
  - Q5: Index Corruption and Self-Healing
  - Summary Table (all findings consolidated)
Diagrams:
  - Mermaid flowchart: All index update code paths
  - Mermaid sequence diagram: API update → index update call chain
  - Mermaid comparison diagram: API path vs direct SQL path
Key Citations:
  src/documents/index.py, src/documents/views.py, src/documents/tasks.py,
  src/documents/signals/handlers.py, src/documents/apps.py,
  src/documents/consumer.py, src/documents/bulk_edit.py,
  src/documents/admin.py, src/documents/management/commands/document_index.py,
  docker/docker-prepare.sh, src/paperless/settings.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The deliverable is placed in the `blitzy/documentation/` directory, which is a project-specific output directory outside the Sphinx documentation build tree. No changes to `docs/conf.py`, `.readthedocs.yml`, or `docs/Makefile` are needed.

### 0.5.4 Cross-Documentation Dependencies

- **No navigation link updates required:** The new file is standalone and does not integrate into the existing Sphinx documentation tree
- **No table-of-contents updates:** The `docs/index.rst` toctree is not modified
- **No index/glossary updates:** The file is a self-contained Q&A reference document

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to this documentation exercise as they are the runtime components whose behavior the document must describe:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | whoosh | ~2.7.4 | Whoosh full-text search engine — the core search index library whose synchronization behavior is being documented |
| pip | django | ~4.0 | Django web framework providing ORM signals (`post_save`, `post_delete`, `m2m_changed`) that the index hooks into |
| pip | django-q | ~1.3 | Django-Q task queue — manages background worker processes (`qcluster`) and `async_task` dispatching for bulk operations |
| pip | djangorestframework | ~3.13 | Django REST Framework — provides the `UpdateModelMixin` and `DestroyModelMixin` that `DocumentViewSet` extends |
| pip | redis | * | Redis client — message broker for Django-Q task queue and Django Channels layer |
| pip | channels | ~3.0 | Django Channels — WebSocket support for real-time status updates during document consumption |
| pip | channels-redis | * | Redis backend for Django Channels layer |
| pip | sphinx | ~4.5.0 | Sphinx documentation generator — existing documentation build tool (not directly used for this deliverable) |
| pip | sphinx_rtd_theme | * | Read the Docs theme for Sphinx (existing documentation infrastructure) |

**Note:** The deliverable document itself (`paperless-ngx_542221a38dff.md`) is plain Markdown and does not require any build tools or documentation generators. The packages listed above are the runtime dependencies whose behavior is being investigated and documented.

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file is a standalone Q&A document that does not integrate into the existing Sphinx-based documentation tree. No internal link rewriting or cross-reference updates are needed.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions addressed:** 5/5 (100%) — all five distinct questions about search index synchronization are answered
- **Code paths traced:** 6/6 (100%) — all index update code paths are documented:
  - API `DocumentViewSet.update()` → synchronous `index.add_or_update_document()`
  - Document consumption → `document_consumption_finished` signal → `add_to_index` handler
  - Bulk edit operations → `async_task("documents.tasks.bulk_update_documents")`
  - Django admin → `DocumentAdmin.save_model()` → `index.add_or_update_document()`
  - CLI → `manage.py document_index reindex` → `index_reindex()` full rebuild
  - Startup → `docker/docker-prepare.sh` `search_index()` → conditional reindex
- **Source files cited:** 11/11 (100%) — every source file contributing to the answers is referenced with line numbers
- **Diagrams included:** 3 Mermaid diagrams covering index update paths, API sequence, and SQL bypass comparison

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Each of the five questions has a direct answer supported by code-path evidence
- Each answer includes the rationale/thinking behind the conclusion
- All claims reference specific source files and line numbers
- Architectural context is provided before diving into individual answers

**Accuracy validation:**
- All code references have been verified by reading the actual source files via `read_file`
- Signal registration verified in `src/documents/apps.py` — `add_to_index` connects only to `document_consumption_finished`
- Synchronous index update verified in `src/documents/views.py` line 216 — `index.add_or_update_document()` is called directly, not via `async_task`
- Reindex mechanism verified in `src/documents/tasks.py` line 38 — `index_reindex()` passes `recreate=True` to `open_index()`
- Startup self-healing verified in `docker/docker-prepare.sh` lines 49-57 — version sentinel file gates reindex

**Clarity standards:**
- Technical accuracy with accessible language suitable for a developer onboarding to the codebase
- Progressive disclosure: architecture context first, then specific question answers
- Consistent terminology: "Whoosh index" (not "search index" or "full-text index" interchangeably without definition), "Django-Q worker" (not "background process" without context)

**Maintainability:**
- Source citations with file paths and line numbers enable future verification
- Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum code snippets per answer:** 1-2 short snippets (2-3 lines each) showing the critical decision point
- **Diagram types:** Mermaid flowcharts and sequence diagrams
- **Visual content:** 3 Mermaid diagrams as specified in Section 0.4.3

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable

- **Source code analyzed for documentation content (read-only — not modified):**
  - `src/documents/index.py` — Whoosh index schema, open/write/search/update/remove operations
  - `src/documents/views.py` — `DocumentViewSet.update()`, `DocumentViewSet.destroy()`, `UnifiedSearchViewSet`
  - `src/documents/tasks.py` — `index_reindex()`, `index_optimize()`, `bulk_update_documents()`, `consume_file()`
  - `src/documents/signals/__init__.py` — Signal definitions
  - `src/documents/signals/handlers.py` — `add_to_index()`, `update_filename_and_move_files()`, `cleanup_document_deletion()`
  - `src/documents/apps.py` — Signal handler registration in `ready()`
  - `src/documents/consumer.py` — `Consumer.try_consume_file()` and signal emission
  - `src/documents/bulk_edit.py` — Bulk operations with async task queuing
  - `src/documents/admin.py` — `DocumentAdmin.save_model()` and `delete_model()`
  - `src/documents/management/commands/document_index.py` — CLI reindex/optimize command
  - `src/documents/models.py` — Document model definition and field schema
  - `src/documents/serialisers.py` — `DocumentSerializer` field set
  - `src/paperless/settings.py` — `INDEX_DIR`, `Q_CLUSTER`, `DATA_DIR` configuration
  - `docker/docker-prepare.sh` — Startup index version check and conditional reindex
  - `docker/supervisord.conf` — Supervised processes (gunicorn, consumer, qcluster)

- **Existing documentation referenced (read-only — not modified):**
  - `docs/administration.rst` — Existing index management documentation
  - `docs/api.rst` — Existing API search documentation
  - `docs/usage_overview.rst` — Existing search usage documentation
  - `docs/conf.py` — Sphinx configuration for context

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository are modified, per both the user's explicit instruction and the `SWE-AtlasQnA-Repo` implementation rule
- **Test file modifications:** No test files are created or modified
- **Existing documentation updates:** No changes to `docs/*.rst` files, `docs/conf.py`, or `.readthedocs.yml`
- **Feature additions or code refactoring:** No changes to application behavior
- **Deployment configuration changes:** No changes to `docker/`, `Dockerfile`, or compose files
- **Frontend analysis:** The Angular SPA in `src-ui/` is not analyzed as the user's questions are entirely backend-focused
- **Email ingestion analysis:** The `src/paperless_mail/` subsystem is not relevant to the search index synchronization questions
- **OCR/parser analysis:** The `src/paperless_tesseract/`, `src/paperless_text/`, and `src/paperless_tika/` parsers are not relevant beyond their role in feeding content to the consumer pipeline

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file that does not require a build step
- **Documentation preview command:** Any Markdown renderer (e.g., grip, GitHub web preview, VS Code Markdown preview)
- **Diagram generation:** Mermaid diagrams are embedded inline in the Markdown using mermaid code fences — they render natively on GitHub and most modern Markdown viewers
- **Default format:** Markdown (.md) with Mermaid diagrams
- **Citation requirement:** Every technical claim must reference a source file path and line number
- **Style guide:** Follow the SWE-AtlasQnA-Repo rule: provide thinking/rationale behind answers, base answers on the code as truth, do not make assumptions
- **Documentation validation:** Manual review for factual accuracy against source code; no automated linting required

### 0.9.2 File Placement and Naming

- **Output directory:** `blitzy/documentation/` (within the destination repository)
- **Filename:** `paperless-ngx_542221a38dff.md` (matching the source branch name per implementation rule)
- **Directory creation:** The `blitzy/documentation/` directory must be created if it does not already exist

## 0.10 Rules for Documentation

The following rules are explicitly derived from user instructions and the `SWE-AtlasQnA-Repo` implementation rule:

- **Do not modify any existing files in the source repository.** The only file operation permitted is creating the new markdown document at `blitzy/documentation/paperless-ngx_542221a38dff.md`.
- **Do not make assumptions — base all answers on the code as the truth.** Every answer must trace a specific code path through identified source files with line number citations. Speculation about behavior not evidenced in the code is not permitted.
- **Provide thinking and rationale behind the answers.** Each answer must include the logical reasoning chain — why the code behaves as it does, what mechanisms are in play, and what evidence supports the conclusion.
- **Place the generated document in the `blitzy/documentation` directory** in the destination repository, named `paperless-ngx_542221a38dff.md` (matching the source branch name).
- **Clean up test artifacts when finished.** If any temporary test documents are created for observation during the investigation, they must be removed before the task is complete. The deliverable document itself is the only persistent artifact.
- **No source code modifications** — not even documentation comments or docstrings within existing source files.
- **Comprehensive answers** — each of the five user questions must receive a complete, self-contained answer that a developer new to the codebase can follow without needing to read additional files.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Root-level files:**
- `Pipfile` — Python dependency manifest (Whoosh ~2.7.4, Django ~4.0, Django-Q ~1.3, DRF ~3.13, Sphinx ~4.5.0)
- `requirements.txt` — Pinned Python dependency versions (auto-generated from Pipfile)
- `.readthedocs.yml` — Read the Docs build configuration (Sphinx, Python 3.8)
- `gunicorn.conf.py` — ASGI server configuration
- `paperless.conf.example` — Environment variable documentation

**Core backend source:**
- `src/documents/index.py` — Whoosh search index: schema definition, open/create, writer/searcher context managers, update/remove operations, DelayedFullTextQuery, DelayedMoreLikeThisQuery, autocomplete
- `src/documents/views.py` — REST API views: DocumentViewSet (update at line 212, destroy at line 219), UnifiedSearchViewSet (search at line 377), PostDocumentView (upload at line 491), BulkEditView, SearchAutoCompleteView
- `src/documents/tasks.py` — Background tasks: index_reindex (line 38), index_optimize (line 32), bulk_update_documents (line 270), consume_file (line 184)
- `src/documents/signals/__init__.py` — Signal definitions: document_consumption_started, document_consumption_finished, document_consumer_declaration
- `src/documents/signals/handlers.py` — Signal handlers: add_to_index (line 428), update_filename_and_move_files (line 312), cleanup_document_deletion (line 233), set_correspondent, set_document_type, set_tags, set_log_entry, add_inbox_tags
- `src/documents/apps.py` — App configuration: DocumentsConfig.ready() connecting signal handlers (lines 11-29)
- `src/documents/consumer.py` — Document ingestion pipeline: Consumer.try_consume_file() emitting document_consumption_finished signal (line 306)
- `src/documents/bulk_edit.py` — Bulk operations: set_correspondent, set_document_type, add_tag, remove_tag, modify_tags, delete — all using async_task for deferred index updates
- `src/documents/admin.py` — Django admin: DocumentAdmin.save_model (line 85), delete_model (line 79), delete_queryset (line 70) — all with synchronous index updates
- `src/documents/models.py` — Document model: title (max 128, db_index=True), content (TextField), correspondent (FK), document_type (FK), tags (M2M), checksum (MD5), timestamps, filename, archive_filename
- `src/documents/serialisers.py` — DocumentSerializer field set and DynamicFieldsModelSerializer
- `src/documents/management/commands/document_index.py` — CLI command: reindex and optimize subcommands wrapping tasks.index_reindex() and tasks.index_optimize()
- `src/documents/filters.py` — DocumentFilterSet for API filtering

**Settings and configuration:**
- `src/paperless/settings.py` — INDEX_DIR (line 73: DATA_DIR/index), Q_CLUSTER (lines 449-457: redis broker, catch_up=False, recycle=1), DATA_DIR, MEDIA_ROOT, TASK_WORKERS

**Infrastructure:**
- `docker/docker-prepare.sh` — Container startup: search_index() function (lines 49-57) with index version sentinel file and conditional reindex
- `docker/supervisord.conf` — Process supervision: gunicorn (ASGI server), document_consumer (directory watcher), qcluster (Django-Q task worker)

**Existing documentation:**
- `docs/administration.rst` — "Managing the document search index" section (lines 337-358)
- `docs/api.rst` — REST API documentation including search endpoints and full-text query parameters (lines 147-214)
- `docs/usage_overview.rst` — Search usage documentation including Whoosh query language reference (lines 268-322)
- `docs/conf.py` — Sphinx configuration: project "Paperless-ngx", sphinx_rtd_theme, extensions

**Folder structure explored:**
- Root (`/`) — Repository overview and top-level files
- `src/` — Python source tree with all Django apps
- `src/documents/` — Core documents app with all submodules
- `src/documents/signals/` — Signal definitions and handlers
- `src/documents/management/` — Management command namespace
- `src/documents/management/commands/` — All CLI commands including document_index
- `src/paperless/` — Core project package (settings, URLs, ASGI)
- `docs/` — Sphinx documentation workspace
- `docker/` — Container runtime assets

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 External URLs

No external URLs were referenced in the user's requirements. All investigation is based entirely on repository source code analysis.

