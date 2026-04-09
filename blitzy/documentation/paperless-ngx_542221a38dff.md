# Paperless-ngx REST API Authentication Investigation

This document is a comprehensive, evidence-based investigation of the **Paperless-ngx REST API authentication system**, targeted at integration developers who need to programmatically interact with the API from external tools (scripts, automation platforms, third-party applications).

Every answer in this document is **verified against the actual codebase** — the source code is treated as the single source of truth. No assumptions are made; every claim traces to a specific source file and line number.

**Version context:** Paperless-ngx **v1.7.0**
`Source: src/paperless/version.py:1` — `__version__ = (1, 7, 0)`

---

## 1. Running Paperless Locally

To test the API, you first need a running Paperless-ngx instance. The following steps set up a minimal local development server suitable for API testing.

### Environment Variables

| Variable | Purpose | Default (from settings.py) |
|----------|---------|---------------------------|
| `PAPERLESS_DATA_DIR` | Database and search index storage | `<BASE_DIR>/../data` (`Source: src/paperless/settings.py:66`) |
| `PAPERLESS_MEDIA_ROOT` | Document file storage | `<BASE_DIR>/../media` (`Source: src/paperless/settings.py:61`) |
| `PAPERLESS_CONSUMPTION_DIR` | Consumption intake directory | `<BASE_DIR>/../consume` (`Source: src/paperless/settings.py:78-81`) |
| `PAPERLESS_LOGGING_DIR` | Application log directory | `<DATA_DIR>/log` (`Source: src/paperless/settings.py:76`) |
| `PAPERLESS_SECRET_KEY` | Django secret key for cryptographic signing | Built-in fallback (insecure — should be overridden) (`Source: src/paperless/settings.py:260-263`) |
| `PAPERLESS_DEBUG` | Enable debug mode (optional) | `"NO"` (`Source: src/paperless/settings.py:50`) |

### Step-by-Step Commands

```bash
# 1. Navigate to the src/ directory where manage.py lives
cd src

# 2. Set required environment variables
export PAPERLESS_DATA_DIR=/tmp/paperless_data
export PAPERLESS_MEDIA_ROOT=/tmp/paperless_media
export PAPERLESS_CONSUMPTION_DIR=/tmp/paperless_consume
export PAPERLESS_LOGGING_DIR=/tmp/paperless_log
export PAPERLESS_SECRET_KEY=testkey123

# 3. Create required directories
mkdir -p $PAPERLESS_DATA_DIR $PAPERLESS_MEDIA_ROOT $PAPERLESS_CONSUMPTION_DIR $PAPERLESS_LOGGING_DIR

# 4. Run database migrations
python3 manage.py migrate --skip-checks

# 5. Start the development server
python3 manage.py runserver 0.0.0.0:8000
```

### Thinking/Rationale

The environment variables above are derived directly from `src/paperless/settings.py`:

- **`PAPERLESS_DATA_DIR`** is read at line 66: `DATA_DIR = os.getenv("PAPERLESS_DATA_DIR", os.path.join(BASE_DIR, "..", "data"))`. This directory stores the SQLite database (`db.sqlite3`) and the search index.
- **`PAPERLESS_MEDIA_ROOT`** is read at line 61: `MEDIA_ROOT = os.getenv("PAPERLESS_MEDIA_ROOT", os.path.join(BASE_DIR, "..", "media"))`. This stores document originals, archive versions, and thumbnails.
- **`PAPERLESS_CONSUMPTION_DIR`** is read at lines 78–81: `CONSUMPTION_DIR = os.getenv("PAPERLESS_CONSUMPTION_DIR", os.path.join(BASE_DIR, "..", "consume"))`. This is the directory Paperless watches for new documents to ingest.
- **`PAPERLESS_LOGGING_DIR`** is read at line 76: `LOGGING_DIR = os.getenv("PAPERLESS_LOGGING_DIR", os.path.join(DATA_DIR, "log"))`. Application log files are written here.
- **`PAPERLESS_SECRET_KEY`** is Django's `SECRET_KEY`, used for cryptographic signing (session cookies, CSRF tokens, password hashing). The codebase provides a built-in fallback default (`"e11fl1oa-*ytql8p)(06fbj4ukrlo+n7k&q5+$1md7i+mge=ee"` at `src/paperless/settings.py:260-263`), so the key is technically not required. However, this default is **insecure for production** — any deployment accessible beyond a closed network should override it with a unique, random value.
- **`PAPERLESS_DEBUG`** is read at line 50: `DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`. Setting this to `"yes"` enables Django debug mode, which also conditionally adds the `AngularApiAuthenticationOverride` class (`Source: src/paperless/settings.py:129-132`).

The **`--skip-checks`** flag on the `migrate` command is recommended because Django system checks (in `src/paperless/checks.py`) validate OCR binaries (`tesseract`, `optipng`, etc.) and other optional dependencies that are not needed for API testing. Without this flag, migration may fail if these binaries are not installed.

The `manage.py` entry point is located at `src/manage.py` (`Source: src/manage.py`), which is why we `cd src` first.

---

## 2. Creating a Test User and API Token

Before making authenticated API requests, you need a user account and an API token.

### Method 1: Django Management Shell

```bash
cd src
python3 manage.py shell
```

```python
>>> from django.contrib.auth.models import User
>>> user = User.objects.create_superuser('testuser', 'test@example.com', 'testpassword')
>>> from rest_framework.authtoken.models import Token
>>> token = Token.objects.create(user=user)
>>> print(token.key)
# Output: a 40-character hex string, e.g. "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
```

### Method 2: POST to the Token Endpoint

Once a user exists, you can obtain a token via HTTP:

```bash
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpassword"}'
```

**Response (HTTP 200):**

```json
{"token":"a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"}
```

The token is a **40-character hexadecimal string**.

### Thinking/Rationale

- The **token endpoint** is registered at `src/paperless/urls.py` line 81:
  ```python
  path("token/", views.obtain_auth_token)
  ```
  where `views` is imported from `rest_framework.authtoken` at line 26:
  ```python
  from rest_framework.authtoken import views
  ```
  `Source: src/paperless/urls.py:26,81`

- **`obtain_auth_token`** is an instance of `rest_framework.authtoken.views.ObtainAuthToken`. This view accepts a POST request with `username` and `password` fields (as JSON or form data), validates the credentials against Django's authentication system, and returns `{"token": token.key}`. If the user does not yet have a token, one is created automatically.

- The **`Token` model** is `rest_framework.authtoken.models.Token`. It is enabled in the application via `INSTALLED_APPS` at `src/paperless/settings.py` line 108:
  ```python
  "rest_framework.authtoken",
  ```
  `Source: src/paperless/settings.py:108`

  This entry registers the `authtoken` app, which provides the database migration for the `authtoken_token` table and the Django admin interface for managing tokens.

- The **token key** is generated via `binascii.hexlify(os.urandom(20)).decode()`, which produces a 40-character hex string (20 random bytes → 40 hex characters). This is defined in the `Token.generate_key()` method within the DRF library code at `rest_framework/authtoken/models.py`.

- Each user can have **at most one token** — the `user` field on the `Token` model is a `OneToOneField`, enforcing a database-level uniqueness constraint.

---

## 3. Authenticated Request to List Documents

### Header Format

| Component | Value |
|-----------|-------|
| **Header name** | `Authorization` |
| **Header format** | `Token <key>` |
| **Keyword** | `Token` (NOT `Bearer`) |

The keyword is literally the string `Token`. This is defined by the `rest_framework.authentication.TokenAuthentication` class, which sets the class attribute `keyword = 'Token'`.

### Endpoint Path

**`GET /api/documents/`**

`Source: src/paperless/urls.py:32` — `api_router.register(r"documents", UnifiedSearchViewSet)` within the `r"^api/"` prefix at line 40.

### Full curl Example

```bash
curl -s http://localhost:8000/api/documents/ \
  -H "Authorization: Token a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
```

### Response Headers

An authenticated response includes two custom version headers injected by the `ApiVersionMiddleware`:

| Header | Value | Source |
|--------|-------|--------|
| `X-Api-Version` | `2` | The last element of `ALLOWED_VERSIONS = ["1", "2"]` (`Source: src/paperless/settings.py:126`) |
| `X-Version` | `1.7.0` | Formatted from `__version__ = (1, 7, 0)` (`Source: src/paperless/version.py:1`) |

`Source: src/paperless/middleware.py:12-14`

### Verified Live Test Result

| Test | Result |
|------|--------|
| Authenticated `GET /api/documents/` | HTTP 200, `{"count":0,"next":null,"previous":null,"results":[]}` |

### Thinking/Rationale

- **URL routing chain**: `src/paperless/urls.py` line 40 wraps all API routes under `r"^api/"` using `re_path`. The `DefaultRouter` instance at line 29 registers the `documents` viewset at line 32, making the full path `/api/documents/`. The router automatically generates list (GET), retrieve (GET /:id/), update (PUT /:id/), and destroy (DELETE /:id/) routes for the viewset. Note: `DocumentViewSet` does **not** include `CreateModelMixin` (`src/documents/views.py:172-178`), so there is no POST (create) route on `/api/documents/` — document uploads use the separate `/api/documents/post_document/` endpoint.
  `Source: src/paperless/urls.py:29-35,40`

- **`ApiVersionMiddleware`** at `src/paperless/middleware.py` lines 5–16 adds version headers **only when the user is authenticated** (line 11: `if request.user.is_authenticated`). For unauthenticated requests, these headers are absent.
  ```python
  class ApiVersionMiddleware:
      def __init__(self, get_response):
          self.get_response = get_response

      def __call__(self, request):
          response = self.get_response(request)
          if request.user.is_authenticated:
              versions = settings.REST_FRAMEWORK["ALLOWED_VERSIONS"]
              response["X-Api-Version"] = versions[len(versions) - 1]
              response["X-Version"] = ".".join([str(_) for _ in version.__version__])
          return response
  ```
  `Source: src/paperless/middleware.py:5-16`

- The middleware is registered in the `MIDDLEWARE` stack at `src/paperless/settings.py` line 142:
  ```python
  "paperless.middleware.ApiVersionMiddleware",
  ```
  `Source: src/paperless/settings.py:142`

- The **`UnifiedSearchViewSet`** at `src/documents/views.py` line 377 extends `DocumentViewSet` (line 172) which sets:
  - `serializer_class = DocumentSerializer` (line 181)
  - `pagination_class = StandardPagination` (line 182)
  - `permission_classes = (IsAuthenticated,)` (line 183)
  `Source: src/documents/views.py:172-183,377`

---

## 4. JSON Response Structure

### Top-Level Pagination Envelope

Every response from `/api/documents/` is wrapped in a pagination envelope:

```json
{
  "count": 42,
  "next": "http://localhost:8000/api/documents/?page=2",
  "previous": null,
  "results": [
    { "...document object..." },
    { "...document object..." }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `count` | integer | Total number of documents matching the query |
| `next` | URL string or `null` | Link to the next page, `null` if on the last page |
| `previous` | URL string or `null` | Link to the previous page, `null` if on the first page |
| `results` | array | Array of document objects for the current page |

`Source: src/paperless/views.py:8-11` — `StandardPagination` extends `PageNumberPagination` from DRF, which produces this envelope structure.

### Document Object Fields

Each document object in the `results` array contains these fields, defined by `DocumentSerializer`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Primary key |
| `correspondent` | integer or `null` | Foreign key to correspondent |
| `document_type` | integer or `null` | Foreign key to document type |
| `title` | string | Document title |
| `content` | string | Extracted text content (OCR or parsed) |
| `tags` | array of integers | Array of tag IDs associated with this document |
| `created` | datetime string | Document creation date (ISO 8601 format) |
| `modified` | datetime string | Last modified date (ISO 8601 format) |
| `added` | datetime string | Date the document was added to Paperless (ISO 8601 format) |
| `archive_serial_number` | integer or `null` | Archive serial number (ASN) |
| `original_file_name` | string | Original uploaded filename |
| `archived_file_name` | string or `null` | Archived version filename (`null` if no archive version exists) |

`Source: src/documents/serialisers.py:201-235` — `DocumentSerializer` class with `Meta.fields` tuple at lines 222–235.

### Pagination Behavior

**The response is ALWAYS paginated** — it never returns all results at once in a flat array, even if all results fit on a single page.

| Parameter | Value | Source |
|-----------|-------|--------|
| Default `page_size` | `25` | `src/paperless/views.py:9` |
| Query parameter name | `page_size` | `src/paperless/views.py:10` — `page_size_query_param = "page_size"` |
| Maximum page size | `100000` | `src/paperless/views.py:11` — `max_page_size = 100000` |

**Navigation examples:**

- `GET /api/documents/?page=2` — second page of results
- `GET /api/documents/?page_size=50` — 50 results per page
- `GET /api/documents/?page=3&page_size=10` — third page, 10 results per page

### Example Response: Empty Database

```json
{
  "count": 0,
  "next": null,
  "previous": null,
  "results": []
}
```

Even with zero documents, the pagination envelope (`count`, `next`, `previous`, `results`) is still present.

### Thinking/Rationale

- **`StandardPagination`** inherits from `rest_framework.pagination.PageNumberPagination` (`Source: src/paperless/views.py:5` — import, line 8 — class definition):
  ```python
  from rest_framework.pagination import PageNumberPagination

  class StandardPagination(PageNumberPagination):
      page_size = 25
      page_size_query_param = "page_size"
      max_page_size = 100000
  ```
  `Source: src/paperless/views.py:5,8-11`

- **`DocumentViewSet`** at `src/documents/views.py` line 182 sets `pagination_class = StandardPagination`, importing it from `paperless.views` at line 32.
  `Source: src/documents/views.py:32,182`

- **`UnifiedSearchViewSet`** at `src/documents/views.py` line 377 extends `DocumentViewSet` and inherits the `pagination_class` setting. It does not override pagination behavior for non-search requests.
  `Source: src/documents/views.py:377`

- DRF's `PageNumberPagination` **always** wraps results in the `{count, next, previous, results}` envelope. There is no configuration to disable this wrapping — the envelope structure is part of the `get_paginated_response()` method in the base class. This means the API response is **never** a flat JSON array of documents.

---

## 5. Unauthenticated Request Error

### Scenario 1: No Authentication Header

```bash
curl -s http://localhost:8000/api/documents/
```

**Response:**

- **HTTP Status Code:** `401 Unauthorized`
- **Response Body:**

```json
{"detail":"Authentication credentials were not provided."}
```

### Scenario 2: Invalid Token

```bash
curl -s http://localhost:8000/api/documents/ \
  -H "Authorization: Token invalidtoken123"
```

**Response:**

- **HTTP Status Code:** `401 Unauthorized`
- **Response Body:**

```json
{"detail":"Invalid token."}
```

### Scenario 3: Wrong Header Keyword (`Bearer` Instead of `Token`)

```bash
curl -s http://localhost:8000/api/documents/ \
  -H "Authorization: Bearer a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
```

**Response:**

- **HTTP Status Code:** `401 Unauthorized`
- **Response Body:**

```json
{"detail":"Authentication credentials were not provided."}
```

`TokenAuthentication` only recognizes the keyword `Token`. When `Bearer` is used, the token authentication class returns `None` (no match), and since no other authentication class matches either, DRF treats the request as unauthenticated.

### Verified Live Test Results

| Test | Result |
|------|--------|
| No auth header | HTTP 401, `{"detail":"Authentication credentials were not provided."}` |
| Invalid token | HTTP 401, `{"detail":"Invalid token."}` |
| Wrong keyword (`Bearer`) | HTTP 401, `{"detail":"Authentication credentials were not provided."}` |

### Thinking/Rationale

- **All viewsets enforce `permission_classes = (IsAuthenticated,)`**. The `IsAuthenticated` permission class from DRF checks that `request.user` is authenticated. Every API viewset in Paperless-ngx sets this permission:
  - `DocumentViewSet` at `src/documents/views.py` line 183
  - `CorrespondentViewSet` at `src/documents/views.py` line 125
  - `TagViewSet` at `src/documents/views.py` line 151
  - `DocumentTypeViewSet` at `src/documents/views.py` line 166
  - `LogViewSet` at `src/documents/views.py` line 431
  - `SavedViewViewSet` at `src/documents/views.py` line 459
  - `BulkEditView` at `src/documents/views.py` line 471
  - `PostDocumentView` at `src/documents/views.py` line 493

- When **no authentication class succeeds** (all return `None`), DRF marks the request as unauthenticated. The `IsAuthenticated` permission class then denies the request, and DRF returns HTTP 401 with the message `"Authentication credentials were not provided."`. This is the standard DRF behavior defined in `rest_framework.exceptions`.

- When a **token is found but invalid**, the `TokenAuthentication.authenticate_credentials()` method raises `AuthenticationFailed("Invalid token.")`. This exception is caught by DRF's exception handler and returned as HTTP 401 with `{"detail":"Invalid token."}`.

- The key distinction between "credentials not provided" and "invalid token" is:
  - **"Not provided"**: The `Authorization` header is missing or uses an unrecognized keyword — the authentication class returns `None` (no attempt to validate)
  - **"Invalid token"**: The `Authorization` header uses the correct `Token` keyword, but the token value does not match any record in the database — the authentication class raises an exception

---

## 6. Django Code Trace: Authentication Class and Token Model

### Authentication Class Chain

The DRF authentication classes are configured at `src/paperless/settings.py` lines 116–121:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.BasicAuthentication",
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    # ...
}
```

`Source: src/paperless/settings.py:116-121`

DRF evaluates these classes **in order** for each incoming request:

| Order | Class | Mechanism | Header Format |
|-------|-------|-----------|---------------|
| 1 | `rest_framework.authentication.BasicAuthentication` | HTTP Basic Auth | `Authorization: Basic <base64(username:password)>` |
| 2 | `rest_framework.authentication.SessionAuthentication` | Django session cookie | `Cookie: sessionid=<session_key>` |
| 3 | `rest_framework.authentication.TokenAuthentication` | Token-based auth | `Authorization: Token <key>` |

DRF tries each class in sequence. The first class to return a `(user, auth)` tuple "wins" and the user is authenticated. If a class returns `None`, DRF moves to the next class. If all classes return `None`, the request is treated as unauthenticated.

**Note on DEBUG mode:** When `PAPERLESS_DEBUG=yes`, a fourth authentication class is appended:

```python
if DEBUG:
    REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"].append(
        "paperless.auth.AngularApiAuthenticationOverride",
    )
```

`Source: src/paperless/settings.py:129-132`

This class auto-authenticates requests originating from the Angular development server at `localhost:4200` and is **never active in production**.

### `TokenAuthentication` Deep Dive

**Class:** `rest_framework.authentication.TokenAuthentication`

This is the DRF authentication class that handles token-based authentication. Here is how it processes a request:

#### The `keyword` Attribute

```python
keyword = 'Token'
```

This class attribute is what determines that the header must be `Authorization: Token <key>` and NOT `Authorization: Bearer <key>`. The keyword is compared case-insensitively when parsing the header.

#### The `authenticate(request)` Method

1. Reads the `Authorization` header from the request
2. Splits the header value on whitespace to get `[keyword, token_key]`
3. Checks that the first part matches `self.keyword` (case-insensitive comparison)
4. If the keyword does not match, returns `None` (allowing the next auth class to try)
5. If the keyword matches, calls `self.authenticate_credentials(token_key)` to validate the token

#### The `authenticate_credentials(key)` Method

1. Gets the token model via `self.get_model()`, which returns `rest_framework.authtoken.models.Token`
2. Attempts to look up the token in the database:
   ```python
   model.objects.select_related('user').get(key=key)
   ```
3. If the token is not found, raises `AuthenticationFailed("Invalid token.")`
4. If the token is found, checks that `token.user.is_active` is `True`
5. If the user is inactive, raises `AuthenticationFailed("User inactive or deleted.")`
6. If all checks pass, returns the tuple `(token.user, token)`

### Token Model

**Model:** `rest_framework.authtoken.models.Token`

**Database table:** `authtoken_token`

| Field | Type | Description |
|-------|------|-------------|
| `key` | `CharField(max_length=40)` | Primary key — the token string itself |
| `user` | `OneToOneField(AUTH_USER_MODEL)` | The user this token belongs to |
| `created` | `DateTimeField(auto_now_add=True)` | Timestamp of when the token was created |

**Key facts:**

- **Key generation:** `binascii.hexlify(os.urandom(20)).decode()` — generates 20 random bytes, hex-encodes them into a 40-character string
- **One token per user:** The `user` field is a `OneToOneField`, meaning each user can have at most one token. Attempting to create a second token for the same user will raise an `IntegrityError`
- **Token enabled via INSTALLED_APPS:** The `"rest_framework.authtoken"` entry at `src/paperless/settings.py` line 108 registers the app, enabling its database migration (which creates the `authtoken_token` table) and its Django admin interface

`Source: src/paperless/settings.py:108`

### Authentication Flow Diagram

```mermaid
flowchart TD
    A[HTTP Request] --> B{Authorization Header?}
    B -->|Yes| C[BasicAuthentication]
    B -->|No| D[SessionAuthentication]
    C -->|"Basic ..." keyword?| C1[Decode base64 credentials]
    C -->|No match| D
    C1 -->|Valid| AUTH[User Authenticated ✓]
    C1 -->|Invalid| FAIL1[401 Invalid credentials]
    D -->|Session cookie?| D1[Validate session]
    D -->|No session| E[TokenAuthentication]
    D1 -->|Valid| AUTH
    D1 -->|Invalid| E
    E -->|"Token ..." keyword?| E1["Token.objects.get(key=key)"]
    E -->|No match| NOAUTH[No Authentication]
    E1 -->|Found & active| AUTH
    E1 -->|Not found| FAIL2["401 Invalid token."]
    NOAUTH --> F{IsAuthenticated permission?}
    F -->|Required| DENIED["401 Credentials not provided"]
    AUTH --> G[Process Request → 200 OK]
```

> **Note:** This diagram is a simplified representation. In practice, DRF evaluates **all** authentication classes sequentially — each class independently examines the request and returns either a `(user, auth)` tuple or `None`. The decision nodes above illustrate the *effective* behavior, not the literal control flow within DRF's authentication loop.

### Thinking/Rationale

- The **`DEFAULT_AUTHENTICATION_CLASSES`** list at `src/paperless/settings.py` lines 117–121 defines the exact order in which DRF evaluates authentication. This is the definitive configuration — DRF reads this list at startup and creates authentication class instances for each viewset.

- The **`TokenAuthentication` class** is part of the `djangorestframework` package (version 3.13.1 per `requirements.txt`). It is not a custom Paperless-ngx class — it is Django REST Framework's built-in token authentication implementation.

- The **token model** is registered via `INSTALLED_APPS` at `src/paperless/settings.py` line 108. Without this entry, the `authtoken_token` table would not be created during migrations, and token authentication would fail at runtime.

- **Answer to "Which Django REST Framework authentication class handles token auth?":**
  `rest_framework.authentication.TokenAuthentication`

- **Answer to "Which ORM model stores the tokens?":**
  `rest_framework.authtoken.models.Token`, stored in database table `authtoken_token`

---

## 7. Additional: Custom Authentication in Paperless-ngx

Beyond the three standard DRF authentication classes, Paperless-ngx defines custom authentication middleware in `src/paperless/auth.py`:

### `AutoLoginMiddleware` (lines 9–15)

```python
class AutoLoginMiddleware(MiddlewareMixin):
    def process_request(self, request):
        try:
            request.user = User.objects.get(username=settings.AUTO_LOGIN_USERNAME)
            auth.login(request, request.user)
        except User.DoesNotExist:
            pass
```

`Source: src/paperless/auth.py:9-15`

- **Purpose:** Automatically logs in a preconfigured user for every request
- **Activation:** Only added to the `MIDDLEWARE` stack when `PAPERLESS_AUTO_LOGIN_USERNAME` is set (`Source: src/paperless/settings.py:193-200`)
- **Use case:** Single-user deployments where authentication is not needed (e.g., home NAS setups behind a VPN)

### `AngularApiAuthenticationOverride` (lines 18–33)

```python
class AngularApiAuthenticationOverride(authentication.BaseAuthentication):
    def authenticate(self, request):
        if (
            settings.DEBUG
            and "Referer" in request.headers
            and request.headers["Referer"].startswith("http://localhost:4200/")
        ):
            user = User.objects.filter(is_staff=True).first()
            return (user, None)
        else:
            return None
```

`Source: src/paperless/auth.py:18-33`

- **Purpose:** Auto-authenticates requests from the Angular development server during development
- **Activation:** Only appended to `DEFAULT_AUTHENTICATION_CLASSES` when `DEBUG=True` (`Source: src/paperless/settings.py:129-132`)
- **Security:** This class is **never active in production** and only works for requests with a `Referer` header starting with `http://localhost:4200/`

### `HttpRemoteUserMiddleware` (lines 36–41)

```python
class HttpRemoteUserMiddleware(RemoteUserMiddleware):
    header = settings.HTTP_REMOTE_USER_HEADER_NAME
```

`Source: src/paperless/auth.py:36-41`

- **Purpose:** Enables Single Sign-On (SSO) via the `HTTP_REMOTE_USER` header (or a custom header name)
- **Activation:** Only added to `MIDDLEWARE` when `PAPERLESS_ENABLE_HTTP_REMOTE_USER=true` (`Source: src/paperless/settings.py:207-208`)
- **Use case:** Reverse proxy authentication (e.g., Authelia, Authentik, or Apache `mod_auth`)

### Thinking/Rationale

These custom classes are **supplementary** to the three main DRF authentication classes (`BasicAuthentication`, `SessionAuthentication`, `TokenAuthentication`). They are conditionally activated based on environment variables and are not part of the default authentication chain.

- **Why document these?** Integration developers may encounter unexpected authentication behavior if these middleware classes are enabled in their target Paperless-ngx instance. Understanding that `AutoLoginMiddleware` bypasses all authentication (when configured) and that `AngularApiAuthenticationOverride` only applies in debug mode prevents confusion during API integration testing.
- **Code as truth:** Each class's activation condition was traced directly to the `if` guards in `src/paperless/settings.py` (lines 193–200 for AutoLogin, lines 129–132 for AngularOverride, lines 207–208 for HttpRemoteUser). These are the only code paths that add extra authentication classes or middleware to the stack.

---

## 8. Alternative Authentication Methods

While token authentication is the recommended method for external tool integration, Paperless-ngx supports two additional authentication methods.

### Basic Authentication

**Header:** `Authorization: Basic <base64-encoded username:password>`

```bash
# Example: username "testuser", password "testpassword"
# Base64 of "testuser:testpassword" = "dGVzdHVzZXI6dGVzdHBhc3N3b3Jk"
curl -s http://localhost:8000/api/documents/ \
  -H "Authorization: Basic dGVzdHVzZXI6dGVzdHBhc3N3b3Jk"

# Or using curl's built-in basic auth:
curl -s -u testuser:testpassword http://localhost:8000/api/documents/
```

`Source: src/paperless/settings.py:118` — `"rest_framework.authentication.BasicAuthentication"`

**Pros:** Simple, no token management needed.
**Cons:** Credentials are sent with every request (base64 is NOT encryption); not recommended for production use over unencrypted connections.

### Session Authentication

Session authentication works via Django's session framework. When you log in through the Paperless-ngx web interface, a session cookie (`sessionid`) is set in your browser, and subsequent API requests are automatically authenticated.

`Source: src/paperless/settings.py:119` — `"rest_framework.authentication.SessionAuthentication"`

**Pros:** Seamless for browser-based interactions.
**Cons:** Requires CSRF token handling for non-GET requests; not practical for external scripts or automation tools.

### Recommendation

**Token authentication is the recommended method for external tool integration** because:
1. The token is sent via a standard HTTP header — no cookie management needed
2. Tokens can be revoked independently without changing the user's password
3. No CSRF token handling required
4. The token is long-lived and can be stored securely in configuration files

### Thinking/Rationale

- **Why cover alternative methods?** The `DEFAULT_AUTHENTICATION_CLASSES` list at `src/paperless/settings.py:117-121` configures all three methods as active simultaneously. An integration developer's HTTP client might inadvertently use Basic or Session auth (e.g., browser-based testing with cookies), so understanding the full authentication surface helps avoid subtle bugs.
- **Why recommend Token auth specifically?** Basic Authentication transmits credentials with every request (base64 encoding is reversible, not encryption). Session Authentication requires CSRF token management for state-changing requests (`POST`, `PUT`, `DELETE`), making it impractical for headless scripts. Token Authentication avoids both issues — a single `Authorization: Token <key>` header is stateless, revocable, and does not expose the user's password.
- **Code as truth:** The class paths `rest_framework.authentication.BasicAuthentication` (line 118) and `rest_framework.authentication.SessionAuthentication` (line 119) are verified directly from the DRF configuration in `src/paperless/settings.py`.

---

## 9. Cleanup

After testing, remove the temporary directories and data:

```bash
# Remove temporary directories created for local testing
rm -rf /tmp/paperless_data /tmp/paperless_media /tmp/paperless_consume /tmp/paperless_log
```

If you are running against a persistent database (not `/tmp/`), remember to:

1. **Delete the test user:**

   ```bash
   python3 manage.py shell
   ```

   ```python
   >>> from django.contrib.auth.models import User
   >>> User.objects.filter(username='testuser').delete()
   ```

   (Deleting the user automatically deletes the associated token due to the `CASCADE` foreign key.)

2. **Or revoke just the token:**

   ```bash
   python3 manage.py shell
   ```

   ```python
   >>> from rest_framework.authtoken.models import Token
   >>> Token.objects.filter(user__username='testuser').delete()
   ```

---

## 10. Quick Reference Summary

| Question | Answer |
|----------|--------|
| HTTP header name | `Authorization` |
| Header format | `Token <40-char-hex-key>` |
| Document listing endpoint | `GET /api/documents/` |
| Token acquisition endpoint | `POST /api/token/` |
| Response format | Paginated: `{count, next, previous, results}` |
| Default page size | `25` |
| Max page size | `100000` |
| Page size query parameter | `page_size` |
| Unauthenticated status code | HTTP `401` |
| Unauthenticated error message | `{"detail":"Authentication credentials were not provided."}` |
| Invalid token error message | `{"detail":"Invalid token."}` |
| DRF authentication class | `rest_framework.authentication.TokenAuthentication` |
| Token ORM model | `rest_framework.authtoken.models.Token` |
| Token database table | `authtoken_token` |
| Token length | 40 characters (hex-encoded 20 bytes) |
| Token generation | `binascii.hexlify(os.urandom(20)).decode()` |
| Version header (API) | `X-Api-Version: 2` |
| Version header (App) | `X-Version: 1.7.0` |

### All Verified Live Test Results

| Test | Result |
|------|--------|
| Authenticated `GET /api/documents/` | HTTP 200, `{"count":0,"next":null,"previous":null,"results":[]}` |
| Unauthenticated `GET /api/documents/` | HTTP 401, `{"detail":"Authentication credentials were not provided."}` |
| Invalid token `GET /api/documents/` | HTTP 401, `{"detail":"Invalid token."}` |
| `POST /api/token/` with valid credentials | HTTP 200, `{"token":"<40-char-hex>"}` |
| Wrong header format (`Bearer` instead of `Token`) | HTTP 401, `{"detail":"Authentication credentials were not provided."}` |

### Key Source Code References

| # | Reference | File and Lines |
|---|-----------|----------------|
| 1 | DRF Auth Config | `src/paperless/settings.py:116-121` |
| 2 | Debug Auth Addition | `src/paperless/settings.py:129-132` |
| 3 | INSTALLED_APPS authtoken | `src/paperless/settings.py:108` |
| 4 | API Router | `src/paperless/urls.py:29-35` |
| 5 | Documents Endpoint | `src/paperless/urls.py:32` |
| 6 | API URL Prefix | `src/paperless/urls.py:40` |
| 7 | Token Endpoint | `src/paperless/urls.py:81` |
| 8 | StandardPagination | `src/paperless/views.py:8-11` |
| 9 | ApiVersionMiddleware | `src/paperless/middleware.py:5-16` |
| 10 | Version | `src/paperless/version.py:1` |
| 11 | DocumentViewSet | `src/documents/views.py:172-200` |
| 12 | UnifiedSearchViewSet | `src/documents/views.py:377-426` |
| 13 | CorrespondentViewSet permissions | `src/documents/views.py:125` |
| 14 | TagViewSet permissions | `src/documents/views.py:151` |
| 15 | DocumentTypeViewSet permissions | `src/documents/views.py:166` |
| 16 | LogViewSet permissions | `src/documents/views.py:431` |
| 17 | SavedViewViewSet permissions | `src/documents/views.py:459` |
| 18 | BulkEditView permissions | `src/documents/views.py:471` |
| 19 | PostDocumentView permissions | `src/documents/views.py:493` |
| 20 | DocumentSerializer fields | `src/documents/serialisers.py:201-235` |
| 21 | Custom Auth Classes | `src/paperless/auth.py:9-15,18-33,36-41` |
| 22 | MIDDLEWARE stack | `src/paperless/settings.py:134-146` |
