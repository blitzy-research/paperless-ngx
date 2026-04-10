# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a comprehensive investigative reference document** that answers a series of concrete, testable questions about the Paperless-ngx REST API authentication system — specifically targeting integration developers who need to programmatically interact with the API from external tools.

**Category:** Create new documentation

**Documentation Type:** Technical investigation and Q&A reference (API authentication guide with code-level traceability)

The user's requirements decompose into the following explicit documentation deliverables:

- **Local setup and user/token provisioning**: Document the exact commands to run Paperless locally, create a test user, and generate an API token for that user
- **Authenticated document listing**: Capture the exact HTTP header name and format the API expects for token authentication, along with the complete endpoint path for listing documents
- **Response structure analysis**: Document the JSON response shape (top-level fields), and determine whether the response is paginated or returns all results at once
- **Unauthenticated error behavior**: Capture the exact HTTP status code and error message returned when the same endpoint is accessed without authentication
- **Django code tracing**: Identify which Django REST Framework authentication class handles token authentication and which ORM model stores the tokens — tracing through the actual source code
- **Source code preservation**: No modifications to the existing codebase; test users, tokens, and temporary files are acceptable but must be cleaned up

**Implicit Documentation Needs:**

- Token acquisition flow (how to POST credentials to obtain a token before making authenticated requests)
- API versioning headers (`X-Api-Version`, `X-Version`) that appear in authenticated responses
- The full DRF authentication chain (Basic, Session, Token) and the order of evaluation
- Pagination control parameters (`page`, `page_size`) and their defaults and limits
- Error response format for invalid tokens vs. missing tokens

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — Read-only codebase policy**: The user explicitly requires: "Don't modify the codebase, creating test users, tokens, and any temporary files you need is fine, but leave the source code untouched and clean up any additional files or changes when you're done."
- **Implementation rule — SWE-AtlasQnA-Repo**: The project rule mandates creating a new markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed, placed in the `blitzy/documentation` directory. The document must provide thinking/rationale behind the answers and base all answers on the code as truth with no assumptions.
- **Evidence-based answers**: Every claim must trace to specific source files, line numbers, or live test output
- **Style preference**: Investigation-style Q&A format with command examples, response excerpts, and source code citations

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document authentication mechanics**, we will create a new markdown file `blitzy/documentation/paperless-ngx_542221a38dff.md` containing the verified answers to each question with source citations
- To **capture exact header format**, we will reference `rest_framework.authentication.TokenAuthentication` (keyword `Token`) and cite `src/paperless/settings.py` lines 116–121 where the authentication classes are configured
- To **document the endpoint path**, we will reference `src/paperless/urls.py` lines 29–32 where the `documents` viewset is registered under the `/api/` prefix
- To **describe response structure**, we will reference the `StandardPagination` class in `src/paperless/views.py` lines 8–11 and the `DocumentSerializer` in `src/documents/serialisers.py` lines 201–235
- To **document the unauthenticated error**, we will reference the `IsAuthenticated` permission class on all viewsets in `src/documents/views.py`
- To **trace the authentication class**, we will reference `rest_framework.authentication.TokenAuthentication` and the token model at `rest_framework.authtoken.models.Token` (stored in the `authtoken_token` database table)

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `/api/token/` endpoint (`src/paperless/urls.py` line 81) is the token acquisition mechanism that must be documented alongside authentication usage
- Based on code analysis: The `ApiVersionMiddleware` (`src/paperless/middleware.py`) injects `X-Api-Version` and `X-Version` headers on authenticated responses — integration developers should be aware of these
- Based on the user's goal of "integrating external tools": The token authentication method is the recommended approach, but Basic Authentication (`Authorization: Basic <base64>`) is also available and should be mentioned as an alternative
- Based on pagination behavior: `StandardPagination` defaults to `page_size=25` with a configurable `page_size` query param up to `max_page_size=100000` — the response always uses pagination wrapping even when results fit in one page

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **mature Sphinx-based documentation system** with comprehensive RST source files, but with the documentation on API authentication limited to a brief overview section within a single file.

**Documentation Framework:**
- **Generator:** Sphinx ~4.5.0 (`Pipfile` dev dependency)
- **Theme:** sphinx_rtd_theme (Read the Docs theme)
- **Configuration:** `docs/conf.py` — defines extensions (`autodoc`, `intersphinx`, `todo`, `imgmath`, `viewcode`), theme settings, and build metadata
- **Hosting:** Read the Docs (`.readthedocs.yml` — Python 3.8, Sphinx builder pointing to `docs/conf.py`)
- **Build system:** `docs/Makefile` providing standard Sphinx targets (html, epub, pdf, linkcheck, doctest)
- **Dependencies:** `docs/requirements.txt` is empty (placeholder)
- **Preview container:** `docs/Dockerfile` provides a self-contained docs build + serve pipeline on port 8000

**Existing Documentation Files (RST):**

| File | Topic | Relevance to This Task |
|------|-------|------------------------|
| `docs/api.rst` | REST API reference | **Primary** — contains the existing authentication section |
| `docs/setup.rst` | Installation and deployment | Context for running locally |
| `docs/configuration.rst` | Environment variables | Context for auth-related settings |
| `docs/extending.rst` | Development and contribution | Context for developer setup |
| `docs/usage_overview.rst` | Product model, ingestion, search | Context for document workflows |
| `docs/administration.rst` | Backups, updates, utilities | Administrative context |
| `docs/troubleshooting.rst` | Common operational failures | Error handling context |
| `docs/faq.rst` | Common support questions | User-facing Q&A |
| `docs/index.rst` | Landing page and navigation hub | TOC structure |
| `docs/advanced_usage.rst` | Advanced matching, hooks, filenames | Extended functionality |
| `docs/scanners.rst` | Scanner recommendations | Peripheral |
| `docs/screenshots.rst` | UI visual gallery | Peripheral |
| `docs/changelog.rst` | Release history | Version reference |

**Existing API Authentication Documentation (from `docs/api.rst` lines 111–145):**

The current documentation covers three authentication methods (Basic, Session, Token) in a brief section. It documents the token acquisition endpoint (`POST /api/token/`), the header format (`Authorization: Token <token>`), and mentions token management via Django admin. However, it does not provide:
- Concrete curl/HTTP examples with complete request and response bodies
- JSON response structure for paginated document listings
- Error response details for unauthenticated/unauthorized requests
- Source code traceability to the Django authentication class chain
- Pagination behavior explanation

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code related to API authentication:

- **Authentication classes**: `src/paperless/auth.py` — contains `AutoLoginMiddleware`, `AngularApiAuthenticationOverride`, `HttpRemoteUserMiddleware`
- **DRF settings**: `src/paperless/settings.py` lines 116–127 — `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` configuring `BasicAuthentication`, `SessionAuthentication`, `TokenAuthentication`
- **URL routing**: `src/paperless/urls.py` — API router registration (lines 29–35), token endpoint (line 81)
- **Views with permissions**: `src/documents/views.py` — all viewsets enforce `permission_classes = (IsAuthenticated,)`
- **Pagination**: `src/paperless/views.py` lines 8–11 — `StandardPagination` with `page_size=25`, `page_size_query_param="page_size"`, `max_page_size=100000`
- **Serializers**: `src/documents/serialisers.py` lines 201–235 — `DocumentSerializer` defining all response fields
- **Middleware**: `src/paperless/middleware.py` — `ApiVersionMiddleware` adding `X-Api-Version` and `X-Version` headers
- **Token model**: `rest_framework.authtoken.models.Token` — DRF's built-in model with fields `key`, `user`, `created`; stored in `authtoken_token` table
- **Token view**: `rest_framework.authtoken.views.ObtainAuthToken` — POST handler that validates credentials and returns `{"token": "<key>"}`

Key directories examined:
- `src/paperless/` — Django project configuration, auth, URLs, middleware
- `src/documents/` — Document app views, serializers, models, filters
- `docs/` — Existing Sphinx documentation source files

### 0.2.3 Web Search Research Conducted

No external web search was required for this documentation task. All answers are derived directly from the codebase (source of truth per project rules) and confirmed through live local execution of the Paperless-ngx application with direct API testing via `curl`.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `src/paperless/settings.py` — DRF Authentication Configuration**
- Public APIs: `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` (lines 116–121)
- Current documentation: Partial in `docs/api.rst` (mentions three auth methods)
- Documentation needed: Full enumeration of classes in evaluation order, exact class paths, activation conditions

**Module: `rest_framework.authentication.TokenAuthentication` — Token Auth Handler**
- Public APIs: `authenticate(request)`, `authenticate_credentials(key)`, `get_model()`
- Current documentation: Briefly mentioned in `docs/api.rst` header format section
- Documentation needed: Complete code trace showing keyword `"Token"`, header parsing logic, credential validation flow, token lookup via `model.objects.select_related('user').get(key=key)`

**Module: `rest_framework.authtoken.models.Token` — Token Storage Model**
- Fields: `key` (CharField, max_length=40, primary_key), `user` (OneToOneField to AUTH_USER_MODEL), `created` (DateTimeField, auto_now_add)
- Database table: `authtoken_token`
- Current documentation: Not documented in `docs/api.rst`
- Documentation needed: Model fields, table name, key generation mechanism (`binascii.hexlify(os.urandom(20))`)

**Module: `src/paperless/urls.py` — API URL Routing**
- Endpoints: `/api/documents/` (line 32), `/api/token/` (line 81), `/api/correspondents/`, `/api/document_types/`, `/api/tags/`, `/api/saved_views/`, `/api/logs/`
- Current documentation: Listed in `docs/api.rst` lines 14–20
- Documentation needed: Complete endpoint path for document listing with query parameters

**Module: `src/documents/views.py` — Document ViewSet**
- Public APIs: `UnifiedSearchViewSet` (registered as `documents` in router), inherits `DocumentViewSet`
- Permission enforcement: `permission_classes = (IsAuthenticated,)` (line 183)
- Current documentation: Response fields listed in `docs/api.rst` lines 27–39
- Documentation needed: Exact 401 response body and status code when unauthenticated

**Module: `src/paperless/views.py` — Pagination Configuration**
- Public APIs: `StandardPagination` (lines 8–11)
- Current documentation: Not explicitly documented
- Documentation needed: Default page size (25), query parameter name (`page_size`), max page size (100000), response envelope structure (`count`, `next`, `previous`, `results`)

**Module: `src/documents/serialisers.py` — Document Serializer**
- Public APIs: `DocumentSerializer` (lines 201–235)
- Fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`
- Current documentation: Documented in `docs/api.rst` lines 27–39
- Documentation needed: Confirmation of field list and types with actual response JSON

**Module: `src/paperless/middleware.py` — API Version Middleware**
- Public APIs: `ApiVersionMiddleware.__call__()` (lines 9–16)
- Response headers: `X-Api-Version` (highest allowed version), `X-Version` (app version string)
- Current documentation: Documented in `docs/api.rst` lines 276–284
- Documentation needed: Inclusion in the authenticated response header analysis

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps must be addressed:

- **No existing Q&A-style investigation document**: The user needs a self-contained markdown answering specific integration questions — this does not exist
- **Missing curl examples**: `docs/api.rst` lacks concrete, copy-paste-ready HTTP request examples for token-based authentication
- **Missing response body documentation**: The paginated response envelope (`count`, `next`, `previous`, `results`) is not documented in `docs/api.rst`
- **Missing error response documentation**: The exact `401 Unauthorized` response body (`{"detail":"Authentication credentials were not provided."}`) is not documented
- **Missing code traceability**: No existing documentation traces the authentication flow from HTTP header → `TokenAuthentication.authenticate()` → `Token.objects.get(key=key)` → user resolution
- **Missing token model documentation**: The `authtoken_token` table, `Token` model fields, and key generation mechanism are not documented
- **Missing pagination parameter documentation**: `page_size` query parameter, defaults, and limits are not documented in the API reference

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

Per the **SWE-AtlasQnA-Repo** implementation rule, the output is a single markdown document placed at a specific path. The documentation hierarchy is:

```text
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
        ├── Overview (context and objective)
        ├── Running Paperless Locally
        ├── Creating a Test User and API Token
        ├── Authenticated Request to List Documents
        ├── JSON Response Structure
        ├── Unauthenticated Request Error
        ├── Django Code Trace: Authentication
        └── Cleanup
```

The document will follow this section flow:

- **Overview**: Context about the user's goal (external tool integration) and what this document covers
- **Running Paperless Locally**: Environment setup, dependency installation, database migration, server startup commands
- **Creating a Test User and API Token**: Django management shell commands to create a superuser and generate a DRF auth token; alternative via the `/api/token/` endpoint
- **Authenticated Request to List Documents**: Exact HTTP header name (`Authorization`), format (`Token <key>`), complete endpoint path (`/api/documents/`), full curl example, response headers analysis
- **JSON Response Structure**: Top-level pagination envelope fields (`count`, `next`, `previous`, `results`), document object fields from `DocumentSerializer`, pagination behavior and query parameters
- **Unauthenticated Request Error**: HTTP 401 status code, error message body, additional error scenarios (invalid token, wrong header format)
- **Django Code Trace**: DRF authentication class chain, `TokenAuthentication` class analysis with code walkthrough, `Token` model fields and storage table, authentication flow diagram
- **Cleanup**: Commands to remove temporary data

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract authentication class configuration from `src/paperless/settings.py` lines 116–127
- Extract URL routing from `src/paperless/urls.py` lines 29–35, 81
- Extract serializer field definitions from `src/documents/serialisers.py` lines 201–235
- Extract pagination settings from `src/paperless/views.py` lines 8–11
- Extract permission enforcement from `src/documents/views.py` lines 183, 431, 459, 471, 493
- Extract token model structure from `rest_framework.authtoken.models.Token`
- Extract authentication logic from `rest_framework.authentication.TokenAuthentication`
- Generate live examples by running Paperless locally and capturing actual curl output

**Documentation Standards:**
- Markdown formatting with proper headers (# ## ### ####)
- Code examples using fenced code blocks with language identifiers
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`
- Tables for structured comparisons (auth methods, response fields)
- Thinking/rationale sections explaining why each answer is what it is

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to include:**
- Authentication flow diagram: showing the DRF authentication class chain from HTTP request to user resolution
- Token lifecycle diagram: showing token creation via `/api/token/` → storage in `authtoken_token` → usage via `Authorization: Token <key>` header → lookup and validation

**No screenshots required** — this is a pure API/code investigation document.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/paperless/settings.py`, `src/paperless/urls.py`, `src/paperless/auth.py`, `src/paperless/views.py`, `src/paperless/middleware.py`, `src/documents/views.py`, `src/documents/serialisers.py`, `rest_framework.authentication.TokenAuthentication`, `rest_framework.authtoken.models.Token`, `rest_framework.authtoken.views.ObtainAuthToken` | Complete Q&A investigation document covering API authentication header format, endpoint path, JSON response structure, unauthenticated error behavior, Django auth class tracing, and token model identification |
| `docs/api.rst` | REFERENCE | `docs/api.rst` | Used as reference for existing documentation style and coverage; **not modified** |
| `src/paperless/settings.py` | REFERENCE | `src/paperless/settings.py` | Used as source of truth for `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` configuration; **not modified** |
| `src/paperless/urls.py` | REFERENCE | `src/paperless/urls.py` | Used as source of truth for API endpoint routing; **not modified** |
| `src/documents/views.py` | REFERENCE | `src/documents/views.py` | Used as source of truth for ViewSet permission classes and document list behavior; **not modified** |
| `src/documents/serialisers.py` | REFERENCE | `src/documents/serialisers.py` | Used as source of truth for document response field definitions; **not modified** |
| `src/paperless/views.py` | REFERENCE | `src/paperless/views.py` | Used as source of truth for `StandardPagination` settings; **not modified** |
| `src/paperless/auth.py` | REFERENCE | `src/paperless/auth.py` | Used as source of truth for custom authentication middleware; **not modified** |
| `src/paperless/middleware.py` | REFERENCE | `src/paperless/middleware.py` | Used as source of truth for `ApiVersionMiddleware`; **not modified** |

### 0.5.2 New Documentation Files Detail

**File: `blitzy/documentation/paperless-ngx_542221a38dff.md`**
- **Type:** Technical investigation / Q&A reference
- **Source Code:** Multiple source files (listed in table above)
- **Sections:**
  - Overview — purpose and integration context
  - Running Paperless Locally — setup commands with environment variables, dependency install, migration, server start
  - Creating a Test User and API Token — `manage.py shell` commands to create user and token; POST to `/api/token/` alternative
  - Authenticated Request to List Documents — Header: `Authorization: Token <key>`, Endpoint: `GET /api/documents/`, full curl command, response headers (`X-Api-Version: 2`, `X-Version: 1.7.0`)
  - JSON Response Structure — top-level fields: `count` (integer), `next` (URL or null), `previous` (URL or null), `results` (array of document objects); document fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`; pagination is always present (not a dump-everything response)
  - Unauthenticated Request Error — HTTP 401, `{"detail":"Authentication credentials were not provided."}`; invalid token: HTTP 401, `{"detail":"Invalid token."}`
  - Django Code Trace — `rest_framework.authentication.TokenAuthentication` class with keyword `"Token"`, `get_model()` returning `rest_framework.authtoken.models.Token`, credential validation via `Token.objects.select_related('user').get(key=key)`, token storage in `authtoken_token` table with fields `key`, `user_id`, `created`
  - Cleanup — removal of temp directories
- **Diagrams:**
  - Authentication class chain flowchart (Mermaid)
- **Key Citations:** `src/paperless/settings.py:116-127`, `src/paperless/urls.py:29-35,81`, `src/documents/views.py:172-200`, `src/documents/serialisers.py:201-235`, `src/paperless/views.py:8-11`, `src/paperless/middleware.py:5-16`, `rest_framework/authentication.py:TokenAuthentication`, `rest_framework/authtoken/models.py:Token`

### 0.5.3 Documentation Configuration Updates

No documentation configuration changes are required. This task creates a standalone markdown file within the `blitzy/documentation/` directory and does not modify any existing Sphinx, MkDocs, or ReadTheDocs configuration.

### 0.5.4 Cross-Documentation Dependencies

- The new document references patterns and endpoint paths described in `docs/api.rst` but does not link to it
- No table of contents, index, or glossary updates are needed
- No shared content or includes are affected

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages are relevant to the documentation exercise as they define the authentication system and API behavior being documented. These are the runtime dependencies of the Paperless-ngx application itself — no additional documentation tooling packages are needed since the deliverable is a standalone markdown file.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | django | 4.0.4 | Web framework providing auth models, middleware stack, URL routing, session management |
| pip | djangorestframework | 3.13.1 | REST API framework providing `TokenAuthentication`, `BasicAuthentication`, `SessionAuthentication`, `IsAuthenticated` permission, `PageNumberPagination`, API router |
| pip | django-filter | 21.1 | Filter backends used by document viewsets |
| pip | django-cors-headers | 3.11.0 | CORS middleware in the request pipeline |
| pip | django-extensions | 3.1.5 | Django management extensions |
| pip | django-q | 1.3.9 | Task queue (background jobs) |
| pip | whitenoise | 6.0.0 | Static file serving middleware |
| pip | python-dotenv | 0.20.0 | Environment file loading for `paperless.conf` |
| pip | python-magic | 0.4.25 | MIME type detection for document uploads |
| pip | channels | 3.0.4 | ASGI/WebSocket support (affects `runserver` command) |

All versions above are the **exact pinned versions** from `requirements.txt` in the repository root. The `djangorestframework==3.13.1` package is the critical dependency, as it provides both the `rest_framework.authentication.TokenAuthentication` class and the `rest_framework.authtoken.models.Token` model that are the primary subjects of this documentation investigation.

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file at `blitzy/documentation/paperless-ngx_542221a38dff.md` is a standalone document that does not integrate into the existing Sphinx documentation tree and contains no internal cross-references that need maintenance.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the user's questions against existing documentation:**

| Question | Covered in `docs/api.rst`? | Gap |
|----------|---------------------------|-----|
| How to run Paperless locally | No (covered in `docs/setup.rst` for Docker only) | Local dev server setup not documented |
| Create test user and API token | No | No management commands documented for token creation |
| Exact HTTP header name and format | Partial (`Authorization: Token <token>`) | Missing concrete curl examples |
| Complete endpoint path for listing documents | Yes (`/api/documents/`) | Documented but missing query params |
| JSON response top-level fields | No | Pagination envelope undocumented |
| Pagination vs. dump-everything | No | `StandardPagination` behavior undocumented |
| Unauthenticated status code and error | No | 401 response body not documented |
| Django auth class name | No | Code-level tracing not in existing docs |
| Token storage model | No | `authtoken_token` table undocumented |

**Current coverage:** 1.5 out of 9 questions partially covered (approximately 17%)

**Target coverage:** 100% — every question fully answered with code citations and live verification

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question posed by the user has a dedicated section with a clear, direct answer
- Each answer includes the code-level source citation (file path and line numbers)
- Each answer includes rationale explaining why the answer is what it is (per SWE-AtlasQnA-Repo rule)
- Concrete, copy-paste-ready examples are provided for all HTTP interactions

**Accuracy validation:**
- All curl examples were tested against a live local Paperless-ngx instance
- All response bodies were captured from actual HTTP responses (not fabricated)
- All source code citations reference actual file paths and line numbers verified via `read_file`
- The authentication class chain order matches `src/paperless/settings.py` lines 117–121 exactly

**Verified live test results captured during analysis:**

| Test | Result |
|------|--------|
| Authenticated `GET /api/documents/` | HTTP 200, JSON: `{"count":0,"next":null,"previous":null,"results":[]}` |
| Unauthenticated `GET /api/documents/` | HTTP 401, JSON: `{"detail":"Authentication credentials were not provided."}` |
| Invalid token `GET /api/documents/` | HTTP 401, JSON: `{"detail":"Invalid token."}` |
| `POST /api/token/` with valid credentials | HTTP 200, JSON: `{"token":"<40-char-hex>"}` |
| Wrong header format (`Bearer` instead of `Token`) | HTTP 401, JSON: `{"detail":"Authentication credentials were not provided."}` |

**Clarity standards:**
- Technical accuracy with accessible language targeting integration developers
- Progressive disclosure: answer first, rationale and code trace second
- Consistent use of inline code formatting for endpoint paths, header names, field names

**Maintainability:**
- Source citations enable future verification if the codebase changes
- All answers are self-contained in a single markdown file

### 0.7.3 Example and Diagram Requirements

- **Minimum curl examples:** One per major question (authenticated request, unauthenticated request, token acquisition)
- **Diagram types:** Mermaid flowchart for the DRF authentication class chain
- **Code example testing:** All examples were verified against a live Paperless-ngx instance running locally during analysis
- **Visual content freshness:** N/A — no screenshots; all diagrams are generated from code analysis

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable, containing comprehensive answers to all questions posed

**Source files analyzed and cited (read-only reference):**
- `src/paperless/settings.py` — DRF authentication configuration, INSTALLED_APPS, middleware stack, database config
- `src/paperless/urls.py` — API router registration, token endpoint, URL patterns
- `src/paperless/auth.py` — Custom authentication classes (AutoLoginMiddleware, AngularApiAuthenticationOverride, HttpRemoteUserMiddleware)
- `src/paperless/views.py` — StandardPagination class definition
- `src/paperless/middleware.py` — ApiVersionMiddleware adding X-Api-Version and X-Version headers
- `src/documents/views.py` — DocumentViewSet, UnifiedSearchViewSet, permission classes, action methods
- `src/documents/serialisers.py` — DocumentSerializer field definitions
- `src/documents/models.py` — Document model definition
- `docs/api.rst` — Existing API documentation (reference for context)
- `Pipfile` — Dependency version specifications
- `requirements.txt` — Pinned dependency versions
- `rest_framework/authentication.py` — TokenAuthentication class (DRF library code)
- `rest_framework/authtoken/models.py` — Token model (DRF library code)
- `rest_framework/authtoken/views.py` — ObtainAuthToken view (DRF library code)

**Temporary artifacts created during analysis (cleaned up):**
- SQLite database at `/tmp/paperless_data/db.sqlite3` with test user and token
- Temporary directories: `/tmp/paperless_data/`, `/tmp/paperless_media/`, `/tmp/paperless_consume/`, `/tmp/paperless_log/`

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any `.py`, `.rst`, `.ts`, `.html`, `.yaml`, or other source file — per the user's explicit instruction: "leave the source code untouched"
- **Existing documentation updates**: `docs/api.rst` and other `.rst` files are not modified
- **Frontend (Angular) code**: `src-ui/` is not analyzed or documented
- **Docker/deployment configuration**: `Dockerfile`, `docker/`, compose files are not modified
- **Test file modifications**: No changes to `src/*/tests/` directories
- **Feature additions or code refactoring**: Strictly documentation-only deliverable
- **Other API endpoints**: While the document may mention other endpoints for context (correspondents, tags, etc.), deep documentation of non-document-listing endpoints is out of scope
- **Email/IMAP authentication**: `paperless_mail` authentication is not in scope
- **OCR/parser configuration**: Document processing pipeline is not in scope
- **WebSocket authentication**: While mentioned for completeness in the auth chain, the WebSocket consumer auth is not the focus

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — the deliverable is a standalone markdown file, not integrated into the Sphinx build
- **Documentation preview command:** Any markdown viewer or `cat blitzy/documentation/paperless-ngx_542221a38dff.md`
- **Diagram generation command:** N/A — Mermaid diagrams are embedded inline in the markdown using fenced code blocks; rendering happens in any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Documentation deployment command:** N/A — the file is committed directly to the repository
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every answer must reference source files with path and line numbers
- **Style guide:** SWE-AtlasQnA-Repo rule: provide thinking/rationale behind answers, base all answers on the code as truth, do not make assumptions
- **Documentation validation:** Manual review of all curl examples against live test output; verification that all file path citations resolve to actual files in the repository

### 0.9.2 Live Testing Parameters Used During Analysis

The following environment configuration was used to run Paperless-ngx locally for API testing:

| Variable | Value | Purpose |
|----------|-------|---------|
| `PAPERLESS_DATA_DIR` | `/tmp/paperless_data` | Database and index storage |
| `PAPERLESS_MEDIA_ROOT` | `/tmp/paperless_media` | Document file storage |
| `PAPERLESS_CONSUMPTION_DIR` | `/tmp/paperless_consume` | Consumption intake directory |
| `PAPERLESS_LOGGING_DIR` | `/tmp/paperless_log` | Application log directory |
| `PAPERLESS_SECRET_KEY` | `testkey123` | Django secret key for test |
| `PAPERLESS_DEBUG` | `yes` | Enable debug mode (includes channels dev server) |

Commands executed:
- `python3 manage.py migrate --skip-checks` — Run database migrations
- `python3 manage.py shell` — Create test superuser and generate DRF auth token
- `python3 manage.py runserver 0.0.0.0:8000` — Start development server
- `curl` with various headers — Test authenticated, unauthenticated, and invalid-token requests
- Cleanup: `rm -rf /tmp/paperless_data /tmp/paperless_media /tmp/paperless_consume /tmp/paperless_log`

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository** — the codebase must remain untouched. Only new files in `blitzy/documentation/` are permitted.
- **Create a new markdown document named `paperless-ngx_542221a38dff.md`** — the filename must match the source branch name exactly, as specified by the SWE-AtlasQnA-Repo rule.
- **Place the document in the `blitzy/documentation` directory** — this directory must be created if it does not exist.
- **Provide thinking/rationale behind all answers** — every answer must explain why it is correct, not just what the answer is.
- **Base all answers on the code as the source of truth** — no assumptions; every claim must be traceable to a specific file and line number in the repository or DRF library code.
- **Clean up all temporary artifacts** — test users, tokens, temporary databases, and directories created during analysis must be removed. The analysis environment was cleaned up by deleting `/tmp/paperless_data`, `/tmp/paperless_media`, `/tmp/paperless_consume`, and `/tmp/paperless_log`.
- **Leave the source code untouched** — creating test users, tokens, and temporary files is permitted during investigation, but the source files themselves must not be modified.
- **Document the exact HTTP header name and format** — the response must include the precise header string (e.g., `Authorization: Token <key>`) and the complete endpoint path (e.g., `/api/documents/`).
- **Trace through the Django code** — the document must identify the specific authentication class (`rest_framework.authentication.TokenAuthentication`) and token model (`rest_framework.authtoken.models.Token`) by tracing the code path, not by assumption.

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

**Root-level files examined:**
- `Pipfile` — Python dependency specifications with version constraints
- `requirements.txt` — Pinned dependency versions (auto-generated from Pipfile)
- `.readthedocs.yml` — ReadTheDocs build configuration (Sphinx, Python 3.8)
- `.editorconfig` — Code formatting standards
- `.pre-commit-config.yaml` — Pre-commit hook configuration

**Django project core (`src/paperless/`):**
- `src/paperless/settings.py` — Main Django settings: `INSTALLED_APPS` (line 92–111), `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` (lines 116–121), `MIDDLEWARE` stack (lines 134–146), database config (lines 297–320), pagination implicitly via DRF
- `src/paperless/urls.py` — URL routing: API router (lines 29–35), `documents` viewset (line 32), token endpoint (line 81), auth URLs (line 44–49)
- `src/paperless/auth.py` — Custom auth classes: `AutoLoginMiddleware` (lines 9–15), `AngularApiAuthenticationOverride` (lines 18–33), `HttpRemoteUserMiddleware` (lines 36–41)
- `src/paperless/views.py` — `StandardPagination` (lines 8–11): `page_size=25`, `page_size_query_param="page_size"`, `max_page_size=100000`
- `src/paperless/middleware.py` — `ApiVersionMiddleware` (lines 5–16): adds `X-Api-Version` and `X-Version` headers
- `src/paperless/version.py` — `__version__ = (1, 7, 0)`
- `src/paperless/consumers.py` — WebSocket consumer with auth check
- `src/paperless/asgi.py` — ASGI entry with ProtocolTypeRouter

**Documents app (`src/documents/`):**
- `src/documents/views.py` — `DocumentViewSet` (lines 172–200), `UnifiedSearchViewSet` (lines 377–426), `CorrespondentViewSet` (lines 115–134), `TagViewSet` (lines 137–154), `LogViewSet` (lines 429–450), `SavedViewViewSet` (lines 453–466); all enforce `permission_classes = (IsAuthenticated,)`
- `src/documents/serialisers.py` — `DocumentSerializer` (lines 201–235) defining fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`
- `src/documents/models.py` — `Document` model, `MatchingModel` base, `Correspondent`, `Tag`, `DocumentType`
- `src/documents/filters.py` — Filter sets for document queries

**Documentation (`docs/`):**
- `docs/api.rst` — Existing REST API documentation including authentication section (lines 111–145), endpoint listing (lines 14–20), document fields (lines 27–39), search (lines 147–200), file uploads (lines 226–250), versioning (lines 252–300)
- `docs/conf.py` — Sphinx configuration: project metadata, extensions, theme settings
- `docs/index.rst` — Documentation landing page and toctree
- `docs/setup.rst` — Installation and deployment documentation
- `docs/configuration.rst` — Environment variable documentation
- `docs/Makefile` — Sphinx build targets
- `docs/requirements.txt` — Empty placeholder

**Django REST Framework library code (installed packages):**
- `rest_framework/authentication.py` — `TokenAuthentication` class: keyword `"Token"`, `authenticate()` parsing `Authorization` header, `authenticate_credentials()` performing `Token.objects.select_related('user').get(key=key)`
- `rest_framework/authtoken/models.py` — `Token` model: `key` (CharField, max_length=40, primary_key), `user` (OneToOneField), `created` (DateTimeField, auto_now_add); table name `authtoken_token`; key generation via `binascii.hexlify(os.urandom(20)).decode()`
- `rest_framework/authtoken/views.py` — `ObtainAuthToken` view: POST handler accepting `username` and `password`, returning `{"token": token.key}`

**Configuration and build files:**
- `src/setup.cfg` — pytest, flake8, coverage configuration
- `src/manage.py` — Django management entry point

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 Tech Spec Sections Referenced

- **1.1 Executive Summary** — Provided project overview context (Paperless-ngx v1.7.0, GPL-3.0, community-driven)
- **6.4 Security Architecture** — Provided comprehensive authentication framework documentation including the five authentication methods, token management details, session management, and the security configuration reference table

