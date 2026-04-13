# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **investigate, document, and deliver a comprehensive Q&A analysis of the Paperless-ngx REST API authentication system** — specifically its token-based authentication mechanism — without modifying any source code. The deliverable is a standalone markdown document placed in the `blitzy/documentation` directory.

The feature requirements, restated with enhanced clarity, are:

- **Run Paperless-ngx locally**: Stand up the Django application using SQLite (the default database backend), run migrations, and start the development server to enable live API testing.
- **Create a test user and generate an API token**: Use the Django ORM (via `manage.py` or a script) to create a temporary superuser, then use the `rest_framework.authtoken.models.Token` model to generate an authentication token for that user.
- **Make an authenticated request to list documents**: Issue an HTTP GET request to the document list endpoint with the generated token, observe the exact response shape, and document the results.
- **Document the exact HTTP header name and format**: Identify from both the DRF source code and live testing that the required header is `Authorization: Token <token_value>`.
- **Document the complete endpoint path**: Confirm the canonical endpoint path is `/api/documents/` as registered via the DRF router in `src/paperless/urls.py`.
- **Describe the JSON response structure**: Identify the top-level fields (`count`, `next`, `previous`, `results`) and confirm that the endpoint uses paginated responses powered by `StandardPagination` (page size 25).
- **Test unauthenticated access**: Repeat the same request without the `Authorization` header and document the HTTP 401 status code and the error message `{"detail":"Authentication credentials were not provided."}`.
- **Trace the Django authentication code path**: Identify `rest_framework.authentication.TokenAuthentication` as the class handling token auth, and `rest_framework.authtoken.models.Token` as the model storing tokens in the `authtoken_token` database table.
- **Leave source code untouched**: No modifications to the existing repository files; only the creation of a new markdown document in `blitzy/documentation/` is permitted. All temporary artifacts (test users, tokens, database files, runtime directories) must be cleaned up.

Implicit requirements detected:

- The investigation must cover all three authentication methods configured in `src/paperless/settings.py` (Basic, Session, and Token authentication), with primary focus on Token authentication.
- The answer must trace the full code path from the HTTP header through `get_authorization_header()` → `TokenAuthentication.authenticate()` → `TokenAuthentication.authenticate_credentials()` → `Token.objects.get(key=key)`.
- The investigation must address pagination behavior: the endpoint does not "dump everything at once" — it uses DRF's `PageNumberPagination` subclass with a default page size of 25.
- Error responses for both missing credentials (401) and invalid tokens (401 with "Invalid token.") should be covered.

### 0.1.2 Special Instructions and Constraints

- **Read-only source mandate**: The user explicitly instructs: "Don't modify the codebase." The SWE-AtlasQnA-Repo rule reinforces this: "Do not modify any existing files in the source repository."
- **Cleanup requirement**: "Clean up any additional files or changes when you're done." All temporary runtime artifacts (SQLite database, media/data/consume/static directories, test users, tokens) must be removed after testing.
- **Output format**: Per the SWE-AtlasQnA-Repo rule, the deliverable is a markdown document named `<source_branch_name>.md` placed in `blitzy/documentation/` in the destination repo.
- **Evidence-based answers**: The rule states "Do not make assumptions, base your answers on the code as the truth." All answers must cite specific source files, line numbers, and live test results.
- **Thinking/rationale**: The rule requires "Provide thinking / rationale behind the answers."

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the authentication questions**, we will trace through the DRF configuration in `src/paperless/settings.py` (lines 116–121) where `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` is defined, then follow the code path into `rest_framework.authentication.TokenAuthentication` to document the exact header format (`Authorization: Token <key>`), the keyword matching logic, and the credential lookup against `rest_framework.authtoken.models.Token`.
- To **document the API response shape**, we will examine `src/documents/views.py` where `UnifiedSearchViewSet` (registered as the `documents` router entry) extends `DocumentViewSet`, which uses `StandardPagination` from `src/paperless/views.py` (page_size=25, page_size_query_param="page_size"), and `DocumentSerializer` from `src/documents/serialisers.py` (fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`).
- To **demonstrate the behavior live**, we will install dependencies, run migrations against SQLite, start the dev server, and execute `curl` commands to capture exact HTTP headers and JSON bodies.
- To **deliver the result**, we will create a single markdown document in `blitzy/documentation/` containing all findings, code traces, and test results with full rationale.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

Since this task is a **read-only investigation** with a single markdown deliverable, the repository scope discovery focuses on identifying all files that must be **analyzed** (not modified) to answer the user's questions, plus the single new file to be created.

**Authentication & Token System Files (Primary Analysis Targets)**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `src/paperless/settings.py` (lines 92–131) | Defines `INSTALLED_APPS` (includes `rest_framework.authtoken`), `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` | Core: configures which authentication backends are active |
| `src/paperless/auth.py` | Custom authentication classes: `AutoLoginMiddleware`, `AngularApiAuthenticationOverride`, `HttpRemoteUserMiddleware` | Context: shows the full auth landscape beyond token auth |
| `src/paperless/urls.py` (line 81) | Registers `path("token/", views.obtain_auth_token)` for token acquisition | Core: the token acquisition endpoint |
| `src/paperless/middleware.py` | `ApiVersionMiddleware` — adds `X-Api-Version` and `X-Version` headers to authenticated responses | Supporting: explains additional response headers observed |
| `src/paperless/views.py` | `StandardPagination` class (page_size=25, page_size_query_param="page_size", max_page_size=100000) | Core: controls the paginated response shape |

**DRF Library Files (Installed Package Analysis)**

| Library File Path | Purpose | Relevance |
|-------------------|---------|-----------|
| `rest_framework/authentication.py` — `TokenAuthentication` class | Parses `Authorization: Token <key>` header, calls `authenticate_credentials(key)` | Core: the authentication class that processes token headers |
| `rest_framework/authentication.py` — `get_authorization_header()` | Reads `request.META['HTTP_AUTHORIZATION']` as bytestring | Core: entry point for all auth header parsing |
| `rest_framework/authtoken/models.py` — `Token` model | Stores tokens in `authtoken_token` table with fields: `key` (PK, 40-char hex), `user` (OneToOne FK), `created` | Core: the token storage model |
| `rest_framework/authtoken/views.py` — `ObtainAuthToken` view | POST endpoint that validates username/password and returns `{"token": "<key>"}` | Core: the token acquisition view |
| `rest_framework/authtoken/serializers.py` — `AuthTokenSerializer` | Validates `username` and `password` fields for token acquisition | Supporting: token request validation |

**Document API Files (Response Shape Analysis)**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `src/documents/views.py` — `DocumentViewSet` (line 172) | Base viewset: queryset, serializer, pagination, permissions (`IsAuthenticated`), filter backends | Core: controls the document list behavior |
| `src/documents/views.py` — `UnifiedSearchViewSet` (line 377) | Router-registered viewset extending `DocumentViewSet`; handles both regular list and search queries | Core: the actual viewset serving `/api/documents/` |
| `src/documents/serialisers.py` — `DocumentSerializer` (line 201) | Serializer with fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name` | Core: defines the JSON shape of each document object |
| `src/documents/models.py` — `Document` model (line 88) | ORM model defining all document fields, storage types, and relationships | Supporting: backs the serializer fields |
| `src/documents/filters.py` | `DocumentFilterSet` for query parameter filtering | Supporting: filtering capabilities on the endpoint |

**Documentation Files**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `docs/api.rst` (lines 111–145) | Official API documentation covering Authorization section with Token auth format | Core: confirms the documented header format |
| `docs/api.rst` (lines 150–230) | Search, pagination, and versioning documentation | Supporting: confirms pagination behavior |

**Test Files (Behavioral Reference)**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `src/documents/tests/test_api.py` | API test suite using `force_login` and asserting response structure (`count`, `results`) | Supporting: confirms expected response fields |
| `src/documents/tests/test_views.py` | View tests including login redirect behavior (302 to `/accounts/login/`) | Supporting: confirms unauthenticated redirect behavior |
| `src/paperless/tests/test_websockets.py` | WebSocket consumer tests with authentication gating | Context: shows auth is enforced on WebSocket connections too |

### 0.2.2 Web Search Research Conducted

No external web searches were required for this investigation. All answers were derived directly from:

- Static code analysis of the repository source files
- Static analysis of the installed `djangorestframework==3.13.1` package source code
- Live runtime testing against the Django development server with SQLite

### 0.2.3 New File Requirements

A single new file will be created as the investigation deliverable:

- **`blitzy/documentation/<source_branch_name>.md`** — A comprehensive markdown document answering all of the user's questions about Paperless-ngx API authentication, including:
  - The exact HTTP header name and format for token authentication
  - The complete endpoint path for listing documents
  - The JSON response structure with all top-level fields
  - Pagination behavior details
  - The HTTP status code and error message for unauthenticated requests
  - The Django authentication class and token model code trace
  - Full rationale and code citations for every answer

No other new files are required. All temporary runtime artifacts (database, directories, test users) are created and cleaned up during the investigation phase only.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are directly relevant to this authentication investigation exercise. All versions are sourced from `requirements.txt` (the pinned install manifest) and `Pipfile` (the dependency specification).

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| PyPI | `django` | 4.0.4 | Web framework; provides `django.contrib.auth`, `User` model, session middleware, and `AuthenticationMiddleware` |
| PyPI | `djangorestframework` | 3.13.1 | REST API framework; provides `TokenAuthentication`, `Token` model, `PageNumberPagination`, `IsAuthenticated` permission, and `obtain_auth_token` view |
| PyPI | `django-filter` | 21.1 | Query parameter filtering for API endpoints (`DjangoFilterBackend` in `DocumentViewSet`) |
| PyPI | `django-cors-headers` | 3.11.0 | CORS handling for cross-origin API requests |
| PyPI | `channels` | 3.0.4 | ASGI/WebSocket support; `AuthMiddlewareStack` for WebSocket authentication |
| PyPI | `whitenoise` | 6.0.0 | Static file serving middleware |
| PyPI | `python-dotenv` | 0.20.0 | Environment variable loading from `paperless.conf` |
| PyPI | `redis` | 3.5.3 | Redis client for Django-Q task queue and Channels layer (runtime dependency, not needed for auth investigation) |
| PyPI | `django-q` | 1.3.9 | Background task queue (not directly related to auth but required by INSTALLED_APPS) |
| PyPI | `python-magic` | 0.4.25 | MIME type detection for document uploads |
| PyPI | `whoosh` | 2.7.4 | Full-text search index engine |

### 0.3.2 Dependency Updates

No dependency updates are required for this task. The investigation is read-only and the markdown deliverable has no package dependencies.

**Import References Analyzed (Not Modified)**

The following import chains were traced during the investigation to understand the authentication flow:

- `src/paperless/urls.py` → `from rest_framework.authtoken import views` → `views.obtain_auth_token`
- `src/paperless/settings.py` → `"rest_framework.authentication.TokenAuthentication"` (string reference resolved at runtime)
- `src/documents/views.py` → `from rest_framework.permissions import IsAuthenticated`
- `src/paperless/views.py` → `from rest_framework.pagination import PageNumberPagination`
- `rest_framework/authentication.py` → `from rest_framework.authtoken.models import Token` (lazy import inside `get_model()`)

**External Reference Files Analyzed (Not Modified)**

| File | Content Analyzed |
|------|------------------|
| `Pipfile` | Confirmed `djangorestframework = "~=3.13"` and `django = "~=4.0"` |
| `requirements.txt` | Confirmed exact pins: `django==4.0.4`, `djangorestframework==3.13.1` |
| `src/setup.cfg` | Confirmed `DJANGO_SETTINGS_MODULE=paperless.settings` for pytest |
| `docs/api.rst` | Confirmed documented token auth format: `Authorization: Token <token>` |

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The authentication system involves a multi-layered integration across Django middleware, DRF authentication classes, URL routing, and view permission enforcement. The following touchpoints were traced during the investigation:

**Authentication Configuration Chain**

- `src/paperless/settings.py` (lines 116–121): The `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` list defines the three active authentication backends in order:
  - `rest_framework.authentication.BasicAuthentication` — HTTP Basic auth
  - `rest_framework.authentication.SessionAuthentication` — Cookie/session-based auth
  - `rest_framework.authentication.TokenAuthentication` — Token-based auth via `Authorization` header
- `src/paperless/settings.py` (line 108): `rest_framework.authtoken` is registered in `INSTALLED_APPS`, enabling the `Token` model and its database migration.
- `src/paperless/settings.py` (lines 129–132): In `DEBUG` mode, `AngularApiAuthenticationOverride` is appended to the authentication classes.
- `src/paperless/settings.py` (lines 207–215): When `PAPERLESS_ENABLE_HTTP_REMOTE_USER` is set, `RemoteUserAuthentication` is also appended.

**Token Acquisition Flow**

- `src/paperless/urls.py` (line 81): `path("token/", views.obtain_auth_token)` registers the token acquisition endpoint under the `/api/` prefix, making the full path `/api/token/`.
- `rest_framework/authtoken/views.py` — `ObtainAuthToken.post()`: Accepts `{"username": "...", "password": "..."}` via JSON or form data. On success, calls `Token.objects.get_or_create(user=user)` and returns `{"token": "<key>"}`.

**Token Authentication Flow (Request Lifecycle)**

- `rest_framework/authentication.py` — `get_authorization_header(request)`: Reads `request.META['HTTP_AUTHORIZATION']` as a bytestring.
- `rest_framework/authentication.py` — `TokenAuthentication.authenticate(request)`: Splits the header on whitespace; checks that the first element matches the `keyword` attribute (default: `"Token"`, case-insensitive). Extracts the token string and calls `authenticate_credentials(key)`.
- `rest_framework/authentication.py` — `TokenAuthentication.authenticate_credentials(key)`: Calls `Token.objects.select_related('user').get(key=key)`. On `DoesNotExist`, raises `AuthenticationFailed("Invalid token.")`. On success, checks `token.user.is_active` and returns `(token.user, token)`.

**Permission Enforcement**

- `src/documents/views.py` — Every API viewset explicitly sets `permission_classes = (IsAuthenticated,)`:
  - `CorrespondentViewSet` (line 125)
  - `TagViewSet` (line 151)
  - `DocumentTypeViewSet` (line 166)
  - `DocumentViewSet` (line 183) — parent of `UnifiedSearchViewSet`
  - `LogViewSet` (line 431)
  - `SavedViewViewSet` (line 459)
  - `PostDocumentView` (line 471)
  - `StatisticsView` (line 493)
  - `BulkEditView` / `BulkDownloadView` / `SelectionDataView` (line 540+)

**Pagination Integration**

- `src/paperless/views.py` (lines 8–11): `StandardPagination` extends DRF's `PageNumberPagination` with `page_size = 25`, `page_size_query_param = "page_size"`, and `max_page_size = 100000`.
- `src/documents/views.py` (line 182): `DocumentViewSet` sets `pagination_class = StandardPagination`, which wraps all list responses in the `{"count", "next", "previous", "results"}` envelope.

**Response Header Middleware**

- `src/paperless/middleware.py` (lines 5–16): `ApiVersionMiddleware` adds `X-Api-Version` and `X-Version` headers to every response for authenticated users. These headers are visible in API responses and can be used by clients for version detection.

### 0.4.2 Database/Schema Touchpoints

The token authentication system uses the `authtoken_token` table, created by the DRF migration `rest_framework/authtoken/migrations/0001_initial.py`:

| Table | Columns | Relationships |
|-------|---------|---------------|
| `authtoken_token` | `key` (VARCHAR(40), PK), `created` (DATETIME, auto), `user_id` (INT, UNIQUE FK) | One-to-one with `auth_user` table via `user_id` |
| `auth_user` | `id`, `username`, `password`, `is_active`, `is_staff`, `is_superuser`, etc. | Django's built-in User model |

No schema changes are required for this investigation task.

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this is a read-only investigation task, the execution plan consists of a single file creation plus the investigative steps that produce its content.

**Group 1 — Investigation Steps (No Source Modifications)**

- **ANALYZE**: `src/paperless/settings.py` — Extract `REST_FRAMEWORK` configuration, `INSTALLED_APPS` with `rest_framework.authtoken`, and authentication class list
- **ANALYZE**: `src/paperless/urls.py` — Confirm `/api/token/` endpoint registration and `/api/documents/` router registration
- **ANALYZE**: `src/paperless/auth.py` — Document all custom authentication classes (AutoLogin, Angular override, HTTP Remote User)
- **ANALYZE**: `src/paperless/views.py` — Extract `StandardPagination` configuration (page_size=25)
- **ANALYZE**: `src/paperless/middleware.py` — Document `ApiVersionMiddleware` behavior
- **ANALYZE**: `src/documents/views.py` — Trace `DocumentViewSet` and `UnifiedSearchViewSet` for permission classes, pagination, and serializer usage
- **ANALYZE**: `src/documents/serialisers.py` — Extract `DocumentSerializer` field list
- **ANALYZE**: `src/documents/models.py` — Cross-reference `Document` model fields with serializer
- **ANALYZE**: `rest_framework/authentication.py` — Trace `TokenAuthentication.authenticate()` and `authenticate_credentials()` methods
- **ANALYZE**: `rest_framework/authtoken/models.py` — Document `Token` model schema (key, user, created)
- **ANALYZE**: `rest_framework/authtoken/views.py` — Document `ObtainAuthToken.post()` flow
- **ANALYZE**: `docs/api.rst` — Cross-reference official documentation with code findings

**Group 2 — Runtime Testing (Temporary, Cleaned Up)**

- **EXECUTE**: Install Python dependencies (`django==4.0.4`, `djangorestframework==3.13.1`, etc.)
- **EXECUTE**: Create runtime directories (`data/`, `media/`, `static/`, `consume/`)
- **EXECUTE**: Run `python3 manage.py migrate` to create SQLite database with all tables including `authtoken_token`
- **EXECUTE**: Create test user via Django ORM: `User.objects.create_superuser(username='testuser')`
- **EXECUTE**: Generate token via DRF model: `Token.objects.get_or_create(user=user)`
- **EXECUTE**: Start Django dev server on port 8000
- **EXECUTE**: `curl` with `Authorization: Token <key>` to `/api/documents/` — capture full response
- **EXECUTE**: `curl` without auth to `/api/documents/` — capture 401 response
- **EXECUTE**: `curl` with invalid token — capture "Invalid token." response
- **EXECUTE**: `curl` with `Bearer` scheme — capture 401 "credentials not provided" response
- **CLEANUP**: Delete test token and user via ORM
- **CLEANUP**: Stop dev server
- **CLEANUP**: Remove `data/`, `media/`, `static/`, `consume/` directories

**Group 3 — Deliverable Creation**

- **CREATE**: `blitzy/documentation/<source_branch_name>.md` — Comprehensive Q&A document containing:
  - Authentication header format: `Authorization: Token <40-character-hex-string>`
  - Complete endpoint path: `/api/documents/`
  - JSON response structure: `{"count": <int>, "next": <url|null>, "previous": <url|null>, "results": [<document_objects>]}`
  - Document object fields: `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`
  - Pagination: yes, page size 25, configurable via `?page_size=N` and `?page=N`
  - Unauthenticated response: HTTP 401, `{"detail":"Authentication credentials were not provided."}`
  - Authentication class: `rest_framework.authentication.TokenAuthentication` in `rest_framework/authentication.py`
  - Token model: `rest_framework.authtoken.models.Token` stored in `authtoken_token` table
  - Full code traces with file paths, line numbers, and rationale

### 0.5.2 Implementation Approach

The implementation follows a three-phase approach:

- **Phase 1 — Static Analysis**: Establish the complete authentication architecture by reading all relevant source files. This produces the code trace answers (authentication class, token model, header format) without requiring any runtime.
- **Phase 2 — Runtime Validation**: Stand up the application temporarily to confirm the static analysis findings with live HTTP requests. This produces the exact JSON responses, HTTP status codes, and error messages.
- **Phase 3 — Deliverable Assembly**: Compile all findings into a well-structured markdown document with clear headings for each question, code citations, and rationale sections.

### 0.5.3 Key Findings Summary

The live testing confirmed the following answers to the user's questions:

- **Header name and format**: `Authorization: Token ceedc3f3a8274db44d5b3c44f1477229f62f134d` — the keyword is `Token` (case-insensitive in parsing), followed by a space, followed by the 40-character hex token key.
- **Complete endpoint path**: `/api/documents/` — registered via `api_router.register(r"documents", UnifiedSearchViewSet)` in `src/paperless/urls.py` under the `^api/` prefix.
- **JSON response top-level fields**: `count` (integer, total matching documents), `next` (URL string or null, link to next page), `previous` (URL string or null, link to previous page), `results` (array of document objects).
- **Pagination**: Yes, the endpoint is paginated. Default page size is 25. Controlled by `?page=N` and `?page_size=N` query parameters. Maximum page size is 100,000.
- **Unauthenticated response**: HTTP 401 Unauthorized with body `{"detail":"Authentication credentials were not provided."}` and header `WWW-Authenticate: Basic realm="api"`.
- **Invalid token response**: HTTP 401 Unauthorized with body `{"detail":"Invalid token."}`.
- **Authentication class**: `rest_framework.authentication.TokenAuthentication` in `rest_framework/authentication.py`.
- **Token model**: `rest_framework.authtoken.models.Token` in `rest_framework/authtoken/models.py`, stored in database table `authtoken_token`.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Files to Analyze (read-only, no modifications)**

- Authentication system: `src/paperless/auth.py`, `src/paperless/settings.py`, `src/paperless/urls.py`
- Middleware: `src/paperless/middleware.py`
- API views: `src/documents/views.py`
- Serializers: `src/documents/serialisers.py`
- Models: `src/documents/models.py`
- Pagination: `src/paperless/views.py`
- DRF internals: `rest_framework/authentication.py`, `rest_framework/authtoken/models.py`, `rest_framework/authtoken/views.py`, `rest_framework/authtoken/serializers.py`
- Documentation: `docs/api.rst`
- Test references: `src/documents/tests/test_api.py`, `src/documents/tests/test_views.py`
- Dependencies: `Pipfile`, `requirements.txt`
- Version: `src/paperless/version.py`

**File to Create**

- `blitzy/documentation/<source_branch_name>.md` — The Q&A deliverable document

**Temporary Runtime Artifacts (created and cleaned up)**

- SQLite database: `data/db.sqlite3`
- Runtime directories: `data/`, `media/`, `static/`, `consume/`, `data/log/`
- Test user: `testuser` (Django `auth_user` table entry)
- Test token: token row in `authtoken_token` table

**Investigation Topics In Scope**

- Token authentication header format and keyword
- Document list endpoint path and URL routing
- JSON response envelope structure (count, next, previous, results)
- Document object serialization fields
- Pagination configuration and behavior
- Unauthenticated request error response (HTTP 401)
- Invalid token error response (HTTP 401)
- TokenAuthentication class code trace
- Token model storage and schema
- Token acquisition endpoint (`/api/token/`) behavior
- All three configured authentication methods (Basic, Session, Token)
- ApiVersionMiddleware response headers (`X-Api-Version`, `X-Version`)

### 0.6.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any existing file in the repository
- **Frontend (Angular SPA)**: The `src-ui/` directory is not relevant to this API authentication investigation
- **Email ingestion**: `src/paperless_mail/` is unrelated to REST API token auth
- **OCR/Parser subsystems**: `src/paperless_tesseract/`, `src/paperless_text/`, `src/paperless_tika/` are not relevant
- **Document consumer pipeline**: `src/documents/consumer.py`, `src/documents/tasks.py` — not related to auth
- **Search indexing internals**: `src/documents/index.py` — only the search query routing in `UnifiedSearchViewSet` is relevant
- **Machine learning classifier**: `src/documents/classifier.py` — not related to auth
- **Docker/deployment configuration**: `Dockerfile`, `docker/`, `docker-builders/` — not needed for local dev server testing
- **CI/CD pipelines**: `.github/workflows/` — not relevant
- **Performance optimization**: No performance changes are in scope
- **Multi-user/permission models**: The investigation covers authentication only, not fine-grained authorization
- **WebSocket authentication**: While the `AuthMiddlewareStack` in `src/paperless/asgi.py` is noted for context, detailed WebSocket auth investigation is out of scope

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly emphasized by the user and the project configuration:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the question(s) posed in the prompt.
  - Provide thinking / rationale behind the answers.
  - Do not make assumptions, base your answers on the code as the truth.
  - Do not modify any existing files in the source repository.
  - Do not add any other code in the source repository (besides the above requested document).
  - Place the generated document in the `blitzy/documentation` directory in the destination repo.

- **No Source Modification**: The user explicitly states: "Don't modify the codebase, creating test users, tokens, and any temporary files you need is fine, but leave the source code untouched."

- **Cleanup Mandate**: "Clean up any additional files or changes when you're done." All temporary artifacts (database files, runtime directories, test data) must be removed after the investigation.

### 0.7.2 Investigation-Specific Requirements

- **Evidence-Based Answers**: Every answer must cite specific file paths, line numbers, and code snippets from the repository. No assumptions or generalizations without evidence.
- **Code Tracing Depth**: The user specifically asks to "trace through the Django code to find which authentication class handles token authentication and what model stores the tokens." This requires following the import chain from `settings.py` → DRF `authentication.py` → `authtoken/models.py`.
- **Exact Values Required**: The user asks for "the exact HTTP header name and format," "the complete endpoint path," and "what does the JSON response look like." Answers must provide literal, copy-pasteable values.
- **Negative Testing**: The user asks to "try that same request without auth" — the investigation must include the unauthenticated case with the exact status code and error message.
- **Response Structure Detail**: The user asks "specifically what are the top level fields, and is pagination involved or does it dump everything at once?" — the answer must enumerate all four top-level keys and explain the pagination mechanism.

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Core Authentication Files**

| File Path | Lines Analyzed | Key Findings |
|-----------|---------------|--------------|
| `src/paperless/settings.py` | 1–320 | `REST_FRAMEWORK` config (lines 116–121), `INSTALLED_APPS` with `rest_framework.authtoken` (line 108), database config (lines 297–318), security settings (lines 193–215) |
| `src/paperless/auth.py` | 1–42 (full file) | Three custom auth classes: `AutoLoginMiddleware`, `AngularApiAuthenticationOverride`, `HttpRemoteUserMiddleware` |
| `src/paperless/urls.py` | 1–146 (full file) | Token endpoint at line 81: `path("token/", views.obtain_auth_token)`; documents router at line 32: `api_router.register(r"documents", UnifiedSearchViewSet)` |
| `src/paperless/middleware.py` | 1–17 (full file) | `ApiVersionMiddleware` adds `X-Api-Version` and `X-Version` headers |
| `src/paperless/views.py` | 1–25 (full file) | `StandardPagination`: page_size=25, page_size_query_param="page_size", max_page_size=100000 |
| `src/paperless/version.py` | 1–2 (full file) | `__version__ = (1, 7, 0)` |
| `src/paperless/asgi.py` | 1–18 (full file) | `AuthMiddlewareStack` for WebSocket auth |
| `src/paperless/consumers.py` | Referenced | WebSocket consumer with authentication gating |

**Document API Files**

| File Path | Lines Analyzed | Key Findings |
|-----------|---------------|--------------|
| `src/documents/views.py` | 1–430 | `DocumentViewSet` (line 172): pagination_class, permission_classes=(IsAuthenticated,), filter_backends; `UnifiedSearchViewSet` (line 377) |
| `src/documents/serialisers.py` | 1–501 (full file) | `DocumentSerializer` (line 201): 12 fields; `DynamicFieldsModelSerializer` base class; all other serializers |
| `src/documents/models.py` | 1–220 | `Document` model (line 88) with all fields; `MatchingModel`, `Correspondent`, `Tag`, `DocumentType` |
| `src/documents/filters.py` | Referenced | `DocumentFilterSet` for query parameter filtering |

**DRF Library Files (Installed Package)**

| File Path | Key Findings |
|-----------|--------------|
| `rest_framework/authentication.py` | `TokenAuthentication` class: keyword="Token", `authenticate()` parses `Authorization` header, `authenticate_credentials()` looks up token in DB |
| `rest_framework/authtoken/models.py` | `Token` model: key (PK, 40-char hex), user (OneToOne FK), created; table name `authtoken_token` |
| `rest_framework/authtoken/views.py` | `ObtainAuthToken.post()`: validates credentials, returns `{"token": "<key>"}` |
| `rest_framework/authtoken/serializers.py` | `AuthTokenSerializer`: validates `username` and `password` fields |
| `rest_framework/authtoken/admin.py` | `TokenAdmin`: manages tokens in Django admin |
| `rest_framework/authtoken/migrations/0001_initial.py` | Creates `authtoken_token` table with key, created, user_id columns |
| `rest_framework/pagination.py` | `PageNumberPagination` base class with `page_query_param="page"` |

**Documentation and Configuration**

| File Path | Key Findings |
|-----------|--------------|
| `docs/api.rst` (lines 111–300) | Official documentation: three auth methods, token format `Authorization: Token <token>`, search/pagination behavior, API versioning |
| `Pipfile` | `django = "~=4.0"`, `djangorestframework = "~=3.13"`, all dependencies |
| `requirements.txt` | Pinned versions: `django==4.0.4`, `djangorestframework==3.13.1` |
| `src/setup.cfg` | `DJANGO_SETTINGS_MODULE=paperless.settings`, pytest/coverage config |
| `paperless.conf.example` | Environment variable reference for runtime configuration |

**Test Files Referenced**

| File Path | Key Findings |
|-----------|--------------|
| `src/documents/tests/test_api.py` (lines 1–80) | `TestDocumentApi` setUp creates superuser, uses `force_login`, asserts response structure with `count` and `results` |
| `src/documents/tests/test_views.py` (lines 1–50) | Login redirect test: unauthenticated GET to `/` returns 302 to `/accounts/login/` |

**Folders Explored**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| Root (`""`) | Level 0 | Repository root structure |
| `src/` | Level 1 | Python source tree: `manage.py`, all Django apps |
| `src/paperless/` | Level 2 | Core Django project: settings, urls, auth, middleware, views |
| `src/documents/` | Level 2 | Main document management app: views, serializers, models, tests |
| `src/paperless/tests/` | Level 3 | Tests for websockets and config checks |
| `src/documents/tests/` | Level 3 | API tests, view tests, model tests |
| `docs/` | Level 1 | Sphinx documentation including `api.rst` |

### 0.8.2 Attachments

No attachments were provided for this project.

### 0.8.3 External References

No external Figma URLs or design assets were referenced for this task. All findings are derived from the repository codebase and the installed DRF package source code.

