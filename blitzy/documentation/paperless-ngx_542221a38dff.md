# Paperless-ngx REST API Authentication System — Q&A Analysis

| Metadata | Value |
|---|---|
| **Source Branch** | `paperless-ngx_542221a38dff` |
| **Paperless-ngx Version** | 1.7.0 (`src/paperless/version.py` line 1: `__version__ = (1, 7, 0)`) |
| **Django Version** | 4.0.4 (from `requirements.txt`) |
| **Django REST Framework Version** | 3.13.1 (from `requirements.txt`) |
| **Investigation Type** | Read-only code analysis + live runtime verification |
| **Source Code Modified** | No — zero existing files were changed |

## Overview

This document is a comprehensive, evidence-based investigation of the Paperless-ngx REST API authentication system, with primary focus on **token-based authentication**. Every answer is derived from direct analysis of the repository source code, the installed Django REST Framework (DRF) library code, and live runtime testing against the Django development server using SQLite as the database backend. No assumptions are made — all claims trace to verifiable code and observed HTTP responses.

---

## Table of Contents

1. [Authentication Header Name and Format](#1-authentication-header-name-and-format)
2. [Complete Endpoint Path for Listing Documents](#2-complete-endpoint-path-for-listing-documents)
3. [JSON Response Structure](#3-json-response-structure)
4. [Pagination Details](#4-pagination-details)
5. [Unauthenticated Request Response](#5-unauthenticated-request-response)
6. [Invalid Token Response](#6-invalid-token-response)
7. [Authentication Class Code Trace](#7-authentication-class-code-trace)
8. [Token Model and Storage](#8-token-model-and-storage)
9. [All Configured Authentication Methods](#9-all-configured-authentication-methods)
10. [API Version Headers](#10-api-version-headers)
11. [Token Acquisition Endpoint](#11-token-acquisition-endpoint)

---

## 1. Authentication Header Name and Format

### Answer

The exact HTTP header for token authentication is:

```
Authorization: Token <40-character-hex-string>
```

**Concrete example** (from live testing):

```
Authorization: Token 13a43b8e1952bdf47a3fa48f2cbc8b73aa1279eb
```

### Evidence from Source Code

**1. DRF configuration in `src/paperless/settings.py` (lines 116–121):**

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.BasicAuthentication",
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    ...
}
```

`TokenAuthentication` is the third authentication class in the list. DRF iterates through these classes in order for each request until one successfully authenticates or all return `None`.

**2. DRF's `TokenAuthentication` class in `rest_framework/authentication.py`:**

The class defines:

```python
class TokenAuthentication(BaseAuthentication):
    keyword = 'Token'
    model = None
```

The `keyword` class attribute is `'Token'` — this is the prefix keyword expected in the `Authorization` header. The `authenticate()` method performs the following parsing:

```python
def authenticate(self, request):
    auth = get_authorization_header(request).split()

    if not auth or auth[0].lower() != self.keyword.lower().encode():
        return None

    if len(auth) == 1:
        msg = _('Invalid token header. No credentials provided.')
        raise exceptions.AuthenticationFailed(msg)
    elif len(auth) > 2:
        msg = _('Invalid token header. Token string should not contain spaces.')
        raise exceptions.AuthenticationFailed(msg)

    try:
        token = auth[1].decode()
    except UnicodeError:
        msg = _('Invalid token header. Token string should not contain invalid characters.')
        raise exceptions.AuthenticationFailed(msg)

    return self.authenticate_credentials(token)
```

The `get_authorization_header(request)` function reads the raw header:

```python
def get_authorization_header(request):
    auth = request.META.get('HTTP_AUTHORIZATION', b'')
    if isinstance(auth, str):
        auth = auth.encode(HTTP_HEADER_ENCODING)
    return auth
```

**3. Official documentation in `docs/api.rst` (lines 132–144):**

The documentation explicitly states the format for token authentication:

```
Authorization: Token <token>
```

And confirms: "POST a username and password as a form or json string to `/api/token/` and paperless will respond with a token."

### Full Code Trace

```
HTTP Request with header "Authorization: Token abc123..."
    ↓
get_authorization_header(request)
    → reads request.META['HTTP_AUTHORIZATION'] as bytestring
    → returns b'Token abc123...'
    ↓
TokenAuthentication.authenticate(self, request)
    → splits header on whitespace: auth = [b'Token', b'abc123...']
    → compares auth[0].lower() == self.keyword.lower().encode()
    → b'token' == b'token' → keyword matches
    → token = auth[1].decode() → 'abc123...'
    → calls self.authenticate_credentials('abc123...')
    ↓
TokenAuthentication.authenticate_credentials(self, key)
    → model = self.get_model() → rest_framework.authtoken.models.Token
    → token = model.objects.select_related('user').get(key=key)
    → if DoesNotExist: raises AuthenticationFailed("Invalid token.")
    → if not token.user.is_active: raises AuthenticationFailed("User inactive or deleted.")
    → returns (token.user, token)
```

### Rationale / Thinking

The keyword `Token` is hardcoded as the default in DRF's `TokenAuthentication.keyword` class attribute. The header parsing is **case-insensitive for the keyword** (the code compares `.lower()` on both sides), meaning `Authorization: token abc123` would also work. However, the **token value itself is case-sensitive** because it is used as the primary key lookup in the database (`Token.objects.get(key=key)`). The 40-character hex string is generated by `binascii.hexlify(os.urandom(20)).decode()` in the `Token.generate_key()` class method, which produces only lowercase hexadecimal characters.

**Live runtime verification confirmed**: Sending `Authorization: Token 13a43b8e1952bdf47a3fa48f2cbc8b73aa1279eb` returned HTTP 200 with the expected JSON response. Using `Authorization: Bearer <same_token>` returned HTTP 401 with `"Authentication credentials were not provided."`, confirming the keyword `Token` (not `Bearer`) is required.

---

## 2. Complete Endpoint Path for Listing Documents

### Answer

```
/api/documents/
```

### Evidence from Source Code

**1. Router registration in `src/paperless/urls.py` (lines 29–32):**

```python
api_router = DefaultRouter()
api_router.register(r"correspondents", CorrespondentViewSet)
api_router.register(r"document_types", DocumentTypeViewSet)
api_router.register(r"documents", UnifiedSearchViewSet)
```

The `documents` prefix is registered with `UnifiedSearchViewSet` on line 32.

**2. URL prefix in `src/paperless/urls.py` (lines 38–84):**

The router URLs are included under the `^api/` prefix:

```python
urlpatterns = [
    re_path(
        r"^api/",
        include(
            [
                ...
                path("token/", views.obtain_auth_token),
            ]
            + api_router.urls,
        ),
    ),
    ...
]
```

This means the `documents/` pattern from the router becomes `/api/documents/` in the full URL space.

**3. ViewSet class in `src/documents/views.py` (lines 377–426):**

```python
class UnifiedSearchViewSet(DocumentViewSet):
    ...
```

`UnifiedSearchViewSet` extends `DocumentViewSet`, which is defined at line 172:

```python
class DocumentViewSet(
    RetrieveModelMixin,
    UpdateModelMixin,
    DestroyModelMixin,
    ListModelMixin,
    GenericViewSet,
):
    model = Document
    queryset = Document.objects.all()
    serializer_class = DocumentSerializer
    pagination_class = StandardPagination
    permission_classes = (IsAuthenticated,)
    ...
```

**4. Official documentation in `docs/api.rst` (line 16):**

> `/api/documents/`: Full CRUD support, except POSTing new documents.

### Rationale / Thinking

DRF's `DefaultRouter` (instantiated at `src/paperless/urls.py` line 29) auto-generates URL patterns from registered viewsets. The `register(r"documents", UnifiedSearchViewSet)` call creates:
- `documents/` → list action (GET returns paginated list)
- `documents/<pk>/` → detail action (GET/PUT/PATCH/DELETE for individual documents)

These patterns are nested under the `^api/` regex prefix via the `re_path(r"^api/", include([...] + api_router.urls))` at line 39–84, resulting in the full path `/api/documents/`.

**Live runtime verification confirmed**: `curl http://localhost:8000/api/documents/` with a valid token returned HTTP 200 with `{"count":0,"next":null,"previous":null,"results":[]}`.

---

## 3. JSON Response Structure

### Answer

The response is a **paginated JSON envelope** with four top-level fields:

```json
{
    "count": 0,
    "next": null,
    "previous": null,
    "results": []
}
```

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `count` | integer | Total number of matching documents across all pages |
| `next` | string \| null | Fully-qualified URL to the next page of results, or `null` if on the last page |
| `previous` | string \| null | Fully-qualified URL to the previous page of results, or `null` if on the first page |
| `results` | array | Array of serialized document objects for the current page |

### Document Object Fields

From `DocumentSerializer` in `src/documents/serialisers.py` (lines 201–235):

```python
class DocumentSerializer(DynamicFieldsModelSerializer):

    correspondent = CorrespondentField(allow_null=True)
    tags = TagsField(many=True)
    document_type = DocumentTypeField(allow_null=True)

    original_file_name = SerializerMethodField()
    archived_file_name = SerializerMethodField()
    ...

    class Meta:
        model = Document
        depth = 1
        fields = (
            "id",
            "correspondent",
            "document_type",
            "title",
            "content",
            "tags",
            "created",
            "modified",
            "added",
            "archive_serial_number",
            "original_file_name",
            "archived_file_name",
        )
```

| Field | Type | Read-Only | Description |
|-------|------|-----------|-------------|
| `id` | integer | Yes | Document primary key |
| `correspondent` | integer \| null | No | Foreign key to Correspondent, or `null` |
| `document_type` | integer \| null | No | Foreign key to DocumentType, or `null` |
| `title` | string | No | Document title (max 128 characters) |
| `content` | string | No | Plain-text extracted content of the document |
| `tags` | array of integers | No | List of Tag IDs assigned to the document |
| `created` | datetime string | No | Creation date (ISO 8601 format) |
| `modified` | datetime string | Yes | Last modification date (auto-set by Django's `auto_now=True`) |
| `added` | datetime string | Yes | Date added to Paperless (auto-set by `default=timezone.now`) |
| `archive_serial_number` | integer \| null | No | Position in physical document archive |
| `original_file_name` | string | Yes | Verbose filename of original document (via `SerializerMethodField`) |
| `archived_file_name` | string \| null | Yes | Verbose filename of archived version, `null` if no archive exists |

### Evidence

- **Serializer fields**: `src/documents/serialisers.py` lines 219–235 — the `class Meta: fields` tuple explicitly lists all 12 fields.
- **Model backing**: `src/documents/models.py` lines 88–206 — the `Document` model defines all corresponding database columns.
- **Official documentation**: `docs/api.rst` lines 26–39 lists the same fields with matching descriptions.
- **Test confirmation**: `src/documents/tests/test_api.py` lines 37–65 — the `TestDocumentApi.testDocuments` test asserts `response["count"]` (line 39), `response.data["count"]` (line 58), and `response.data["results"][0]` (line 60), confirming the paginated envelope structure.

### Live Runtime Verification

```
$ curl -H "Authorization: Token <token>" http://localhost:8000/api/documents/
HTTP/1.1 200 OK
Content-Type: application/json

{"count":0,"next":null,"previous":null,"results":[]}
```

The response exactly matches the documented four-field paginated envelope structure.

### Rationale / Thinking

The four top-level fields (`count`, `next`, `previous`, `results`) are the standard pagination envelope produced by DRF's `PageNumberPagination` class. This is not a Paperless-specific design — it is the default DRF pagination response format. The `StandardPagination` class in `src/paperless/views.py` extends `PageNumberPagination` and is set as the `pagination_class` on `DocumentViewSet` (line 182), which `UnifiedSearchViewSet` inherits.

---

## 4. Pagination Details

### Answer

**Yes, the endpoint IS paginated.** It does **NOT** dump everything at once. The default page size is **25 documents per page**.

### Configuration

From `src/paperless/views.py` (lines 8–11):

```python
class StandardPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = "page_size"
    max_page_size = 100000
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| `page_size` | 25 | Default number of documents per page |
| `page_size_query_param` | `"page_size"` | Query parameter name to override page size (e.g., `?page_size=50`) |
| `max_page_size` | 100,000 | Maximum allowed page size to prevent abuse |
| `page_query_param` | `"page"` | Page navigation parameter (inherited from DRF's `PageNumberPagination` default) |

### Usage Examples

```
GET /api/documents/                     → First 25 documents
GET /api/documents/?page=2              → Documents 26–50
GET /api/documents/?page_size=10        → First 10 documents
GET /api/documents/?page=3&page_size=10 → Documents 21–30
```

### Evidence

- `src/documents/views.py` line 182: `pagination_class = StandardPagination` on `DocumentViewSet`
- `src/documents/views.py` line 377: `class UnifiedSearchViewSet(DocumentViewSet):` — inherits `StandardPagination`
- `docs/api.rst` line 158: "Pagination works exactly the same as it does for normal requests on this endpoint."
- The search example in `docs/api.rst` lines 171–196 shows the paginated envelope with `"next": "http://localhost:8000/api/documents/?page=2&query=test"`, confirming `next`/`previous` contain full URLs with query parameters preserved.

### Rationale / Thinking

DRF's `PageNumberPagination` wraps all list responses in the `{count, next, previous, results}` envelope automatically. The `next` and `previous` fields provide fully-qualified URLs for page navigation, making the API **self-describing** — clients can follow `next` links to paginate through all results without constructing URLs manually. The `StandardPagination` subclass in Paperless customizes only three parameters (page size, query param name, and max page size) while inheriting all other behavior from the DRF base class.

The `max_page_size` of 100,000 is notable — it allows clients to request very large pages if needed (e.g., `?page_size=100000`), which effectively allows "dump everything" behavior for small to medium collections, but still enforces an upper bound.

---

## 5. Unauthenticated Request Response

### Answer

- **HTTP Status Code**: `401 Unauthorized`
- **Response Body**: `{"detail":"Authentication credentials were not provided."}`
- **Response Header**: `WWW-Authenticate: Basic realm="api"`

### Evidence from Source Code

**1. Permission enforcement in `src/documents/views.py` (line 183):**

```python
class DocumentViewSet(...):
    ...
    permission_classes = (IsAuthenticated,)
```

Every API viewset in the application requires authentication. The `IsAuthenticated` permission class (from `rest_framework.permissions`) checks `request.user.is_authenticated` and denies access if the user is anonymous.

**2. DRF authentication flow:**

When no `Authorization` header is provided:
- `BasicAuthentication.authenticate()` → returns `None` (no `Authorization: Basic` header)
- `SessionAuthentication.authenticate()` → returns `None` (no session cookie)
- `TokenAuthentication.authenticate()` → returns `None` (no `Authorization: Token` header)
- All three classes return `None` → `request.user` is set to `AnonymousUser`
- `IsAuthenticated.has_permission()` → returns `False` → DRF raises `NotAuthenticated` exception

**3. The `WWW-Authenticate` header:**

DRF includes a `WWW-Authenticate` response header from the first authentication class that provides one. `BasicAuthentication.authenticate_header()` returns `'Basic realm="api"'` (the `www_authenticate_realm` attribute defaults to `'api'`), which is why the 401 response includes `WWW-Authenticate: Basic realm="api"`.

**4. The error message:**

The string `"Authentication credentials were not provided."` is DRF's default `NotAuthenticated` exception message, defined in `rest_framework/exceptions.py`.

### Live Runtime Verification

```
$ curl -D - http://localhost:8000/api/documents/
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="api"
Content-Type: application/json

{"detail":"Authentication credentials were not provided."}
```

### Rationale / Thinking

DRF's authentication and permission system follows a two-phase approach:
1. **Authentication phase**: Each configured authentication class attempts to authenticate the request. If a class returns `None`, the next class is tried. If all return `None`, the request proceeds as unauthenticated (anonymous).
2. **Permission phase**: After authentication, DRF checks permissions. `IsAuthenticated` requires `request.user.is_authenticated == True`. For anonymous users, this fails, and DRF returns a 401 response.

The `WWW-Authenticate: Basic realm="api"` header is significant — it tells HTTP clients that Basic authentication is an option. This header comes from `BasicAuthentication` because it is the first class in the authentication list that provides an `authenticate_header()` method returning a non-`None` value.

---

## 6. Invalid Token Response

### Answer

- **HTTP Status Code**: `401 Unauthorized`
- **Response Body**: `{"detail":"Invalid token."}`
- **Response Header**: `WWW-Authenticate: Basic realm="api"`

### Evidence from Source Code

From `rest_framework/authentication.py` — `TokenAuthentication.authenticate_credentials()`:

```python
def authenticate_credentials(self, key):
    model = self.get_model()
    try:
        token = model.objects.select_related('user').get(key=key)
    except model.DoesNotExist:
        raise exceptions.AuthenticationFailed(_('Invalid token.'))

    if not token.user.is_active:
        raise exceptions.AuthenticationFailed(_('User inactive or deleted.'))

    return (token.user, token)
```

When the token key does not exist in the database, `model.DoesNotExist` is raised, which is caught and converted to an `AuthenticationFailed` exception with the message `"Invalid token."`.

Additionally, if a valid token exists but the associated user has `is_active=False`, the error message would be `"User inactive or deleted."`.

### Live Runtime Verification

```
$ curl -D - -H "Authorization: Token invalidtoken1234567890abcdef12345678" http://localhost:8000/api/documents/
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="api"
Content-Type: application/json

{"detail":"Invalid token."}
```

### Rationale / Thinking

The distinction between "credentials not provided" (Section 5) and "Invalid token" (this section) is important: the first occurs when no authentication mechanism can identify the request (all return `None`), while the second occurs when `TokenAuthentication` actively rejects the token (raises `AuthenticationFailed`). In both cases the HTTP status code is 401, but the error messages differ, allowing clients to distinguish between "you forgot to send credentials" and "the credentials you sent are wrong."

---

## 7. Authentication Class Code Trace

### Answer

The authentication class that handles token authentication is:

**`rest_framework.authentication.TokenAuthentication`**

Located in the installed DRF package at `rest_framework/authentication.py`.

### Complete Code Trace

**Step 1 — Configuration Registration**

`src/paperless/settings.py` (lines 116–121):

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.BasicAuthentication",
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",   # ← Line 120
    ],
    ...
}
```

The class is registered as a **dotted string path**. DRF resolves this string to the actual class at runtime using Django's `import_string()` utility function.

**Step 2 — Request Lifecycle Entry Point**

On each API request, DRF's `APIView.initial()` calls `perform_authentication(request)`, which accesses `request.user`. This triggers the `Request.user` property, which calls `Request._authenticate()`. This method iterates through the configured authentication class instances:

```python
# In rest_framework/request.py (conceptual)
def _authenticate(self):
    for authenticator in self.authenticators:
        try:
            user_auth_tuple = authenticator.authenticate(self)
        except exceptions.APIException:
            ...
        if user_auth_tuple is not None:
            self._authenticator = authenticator
            self.user, self.auth = user_auth_tuple
            return
```

**Step 3 — Header Extraction**

`get_authorization_header(request)` in `rest_framework/authentication.py`:

```python
def get_authorization_header(request):
    auth = request.META.get('HTTP_AUTHORIZATION', b'')
    if isinstance(auth, str):
        auth = auth.encode(HTTP_HEADER_ENCODING)
    return auth
```

Django converts the HTTP `Authorization` header to `HTTP_AUTHORIZATION` in `request.META` (standard WSGI convention: prefix `HTTP_`, uppercase, hyphens become underscores).

**Step 4 — Keyword Matching**

`TokenAuthentication.authenticate(self, request)`:

```python
auth = get_authorization_header(request).split()    # e.g., [b'Token', b'abc123...']

if not auth or auth[0].lower() != self.keyword.lower().encode():
    return None    # Not a Token auth request — skip to next authenticator
```

The comparison `auth[0].lower() != self.keyword.lower().encode()` checks if the first part of the header matches `b'token'`. This is **case-insensitive** — both `Token` and `token` would match.

**Step 5 — Token Extraction and Validation**

```python
token = auth[1].decode()                          # Decode bytes to string
return self.authenticate_credentials(token)        # Look up in database
```

**Step 6 — Database Lookup**

`TokenAuthentication.authenticate_credentials(self, key)`:

```python
model = self.get_model()                           # Returns rest_framework.authtoken.models.Token
token = model.objects.select_related('user').get(key=key)    # Database query
```

- On `DoesNotExist` → raises `AuthenticationFailed("Invalid token.")`
- On success → checks `token.user.is_active`
- Returns `(token.user, token)` — the user object and the token object

### Visual Flow Diagram

```
HTTP Request
    │
    ▼
APIView.initial()
    │
    ▼
Request._authenticate()  ─── iterates through authenticators
    │
    ├─→ BasicAuthentication.authenticate()
    │       → No "Basic" keyword → returns None
    │
    ├─→ SessionAuthentication.authenticate()
    │       → No session cookie → returns None
    │
    └─→ TokenAuthentication.authenticate()
            │
            ▼
        get_authorization_header(request)
            → request.META['HTTP_AUTHORIZATION'] = b'Token abc123...'
            │
            ▼
        Split header: [b'Token', b'abc123...']
            │
            ▼
        Keyword check: b'token' == b'token' ✓
            │
            ▼
        authenticate_credentials('abc123...')
            │
            ▼
        Token.objects.select_related('user').get(key='abc123...')
            │
            ├─→ DoesNotExist: raise AuthenticationFailed("Invalid token.")
            │
            └─→ Found: check user.is_active
                    │
                    └─→ return (token.user, token)  ✓ Authenticated!
```

### Rationale / Thinking

The authentication flow is a classic **chain of responsibility** pattern. DRF iterates through all configured authentication classes in order. Each class either:
- Returns `None` (meaning "I can't authenticate this request, try the next class")
- Returns a `(user, auth)` tuple (meaning "authentication succeeded")
- Raises `AuthenticationFailed` (meaning "I recognize this request type but the credentials are invalid")

`TokenAuthentication` only engages when it detects the `Token` keyword in the `Authorization` header. If the header uses `Basic` or `Bearer` or is absent entirely, `TokenAuthentication` returns `None` and defers to the other authenticators. This design allows multiple authentication schemes to coexist on the same endpoints.

---

## 8. Token Model and Storage

### Answer

The token model is:

**`rest_framework.authtoken.models.Token`**

Located in `rest_framework/authtoken/models.py`, stored in the database table **`authtoken_token`**.

### Token Model Schema

From `rest_framework/authtoken/models.py`:

```python
class Token(models.Model):
    key = models.CharField(_("Key"), max_length=40, primary_key=True)
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, related_name='auth_token',
        on_delete=models.CASCADE, verbose_name=_("User")
    )
    created = models.DateTimeField(_("Created"), auto_now_add=True)

    class Meta:
        abstract = 'rest_framework.authtoken' not in settings.INSTALLED_APPS
        verbose_name = _("Token")
        verbose_name_plural = _("Tokens")

    def save(self, *args, **kwargs):
        if not self.key:
            self.key = self.generate_key()
        return super().save(*args, **kwargs)

    @classmethod
    def generate_key(cls):
        return binascii.hexlify(os.urandom(20)).decode()
```

### Database Table: `authtoken_token`

| Column | SQL Type | Constraints | Description |
|--------|----------|-------------|-------------|
| `key` | VARCHAR(40) | PRIMARY KEY | 40-character lowercase hexadecimal string |
| `user_id` | INTEGER | UNIQUE, FOREIGN KEY → `auth_user.id`, ON DELETE CASCADE | One-to-one relationship with the User model |
| `created` | DATETIME | NOT NULL, auto-set on creation | Timestamp when the token was created |

### Key Properties

- **One token per user**: The `OneToOneField` on `user` enforces that each user can have at most one token. Attempting to create a second token for the same user raises an `IntegrityError`.
- **Auto-generated key**: When a new `Token` is saved without a key, `generate_key()` produces a 40-character hex string from 20 random bytes (`os.urandom(20)` → `binascii.hexlify()` → 40 hex chars).
- **Cascade delete**: When a user is deleted, their token is automatically deleted (`on_delete=models.CASCADE`).
- **Abstract conditionally**: The `class Meta: abstract = 'rest_framework.authtoken' not in settings.INSTALLED_APPS` ensures the model only creates a database table when the `rest_framework.authtoken` app is installed.

### App Registration

`src/paperless/settings.py` (line 108):

```python
INSTALLED_APPS = [
    ...
    "rest_framework",
    "rest_framework.authtoken",    # ← Line 108
    ...
]
```

The `rest_framework.authtoken` app is registered in `INSTALLED_APPS`, which:
1. Makes the `Token` model concrete (not abstract)
2. Enables its database migrations (creates the `authtoken_token` table)
3. Registers the `TokenAdmin` in Django admin for token management

### Rationale / Thinking

The DRF token model uses a deliberately simple design:
- The token `key` is the primary key, enabling direct O(1) lookups by token value.
- The `OneToOneField` to `User` enforces a strict one-token-per-user policy. This is a design choice by DRF — some applications need multiple tokens per user (for different devices/clients), which would require a custom token model.
- The `select_related('user')` in `authenticate_credentials()` ensures the user is fetched in the same SQL query as the token, avoiding an N+1 query problem.
- Token generation uses `os.urandom(20)` (160 bits of entropy), which provides sufficient randomness for authentication tokens.

---

## 9. All Configured Authentication Methods

### Default Configuration

From `src/paperless/settings.py` (lines 116–121), Paperless-ngx configures **three** authentication methods by default:

| # | Class | Header Format | Use Case |
|---|-------|---------------|----------|
| 1 | `rest_framework.authentication.BasicAuthentication` | `Authorization: Basic <base64(username:password)>` | Simple scripts, CLI tools |
| 2 | `rest_framework.authentication.SessionAuthentication` | Cookie-based (Django session) | Browser-based clients, Angular SPA |
| 3 | `rest_framework.authentication.TokenAuthentication` | `Authorization: Token <key>` | API clients, automation, third-party integrations |

### Conditional Additions

**DEBUG mode** (`src/paperless/settings.py` lines 129–132):

```python
if DEBUG:
    REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"].append(
        "paperless.auth.AngularApiAuthenticationOverride",
    )
```

When `PAPERLESS_DEBUG=YES`, a fourth authentication class is appended: `paperless.auth.AngularApiAuthenticationOverride`. This class (defined in `src/paperless/auth.py` lines 18–33) auto-authenticates any request whose `Referer` header starts with `http://localhost:4200/` (the Angular development server URL), using the first staff user in the database. This is a development-only convenience — **it is never active in production**.

**HTTP Remote User** (`src/paperless/settings.py` lines 207–215):

```python
if ENABLE_HTTP_REMOTE_USER:
    MIDDLEWARE.append("paperless.auth.HttpRemoteUserMiddleware")
    AUTHENTICATION_BACKENDS = [
        "django.contrib.auth.backends.RemoteUserBackend",
        "django.contrib.auth.backends.ModelBackend",
    ]
    REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"].append(
        "rest_framework.authentication.RemoteUserAuthentication",
    )
```

When `PAPERLESS_ENABLE_HTTP_REMOTE_USER=YES`, `RemoteUserAuthentication` is appended. This enables SSO (Single Sign-On) via a reverse proxy that sets the `REMOTE_USER` HTTP header. The `HttpRemoteUserMiddleware` in `src/paperless/auth.py` (lines 36–41) reads the header name from `PAPERLESS_HTTP_REMOTE_USER_HEADER_NAME` (default: `HTTP_REMOTE_USER`).

### Custom Authentication Classes

From `src/paperless/auth.py`:

| Class | Lines | Purpose |
|-------|-------|---------|
| `AutoLoginMiddleware` | 9–15 | Django middleware (not DRF auth class). When `PAPERLESS_AUTO_LOGIN_USERNAME` is set, auto-logs in that user for every request. Activated via `settings.py` lines 195–199. |
| `AngularApiAuthenticationOverride` | 18–33 | DRF auth class for DEBUG mode only. Auto-authenticates requests from the Angular dev server (`localhost:4200`). |
| `HttpRemoteUserMiddleware` | 36–41 | Django middleware extending `RemoteUserMiddleware`. Reads SSO username from a configurable HTTP header. |

### Rationale / Thinking

The three default authentication methods serve different client types:
- **BasicAuthentication** is the simplest but least secure (credentials sent with every request, only base64-encoded, not encrypted). Suitable for testing and simple scripts, especially over HTTPS.
- **SessionAuthentication** is the most user-friendly for browser-based access, leveraging Django's built-in session framework with CSRF protection.
- **TokenAuthentication** is the recommended method for API clients — it avoids sending passwords on every request and tokens can be revoked independently of the user's password.

The conditional additions (Angular override, Remote User) demonstrate Paperless-ngx's flexibility in supporting different deployment environments while keeping the default configuration secure and simple.

---

## 10. API Version Headers

### Answer

For authenticated requests, the API includes two custom response headers:

| Header | Current Value | Description |
|--------|---------------|-------------|
| `X-Api-Version` | `2` | The latest API version supported by this Paperless-ngx instance |
| `X-Version` | `1.7.0` | The application version of the Paperless-ngx instance |

### Evidence from Source Code

From `src/paperless/middleware.py` (lines 5–16):

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

- `X-Api-Version`: Set to the **last element** of `REST_FRAMEWORK["ALLOWED_VERSIONS"]` which is `["1", "2"]` (from `src/paperless/settings.py` line 126), so the value is `"2"`.
- `X-Version`: Constructed from `version.__version__` which is `(1, 7, 0)` (from `src/paperless/version.py` line 1), joined with dots to produce `"1.7.0"`.
- These headers are **only added for authenticated requests** (`if request.user.is_authenticated`).

### Live Runtime Verification

```
$ curl -D - -H "Authorization: Token <token>" http://localhost:8000/api/documents/
...
X-Api-Version: 2
X-Version: 1.7.0
...
```

The headers were confirmed present in authenticated responses and absent in unauthenticated (401) responses.

### Rationale / Thinking

These headers serve an important purpose for API clients: they allow clients to detect the server version and API compatibility **without a separate version endpoint**. As documented in `docs/api.rst` (lines 276–284), clients should perform an authenticated request and check for these headers to determine compatibility. The headers are only present for authenticated requests because: (1) only authenticated users should need version information, and (2) the middleware runs after authentication, so `request.user.is_authenticated` is accurate at that point.

---

## 11. Token Acquisition Endpoint

### Answer

Tokens are acquired by POSTing credentials to:

```
POST /api/token/
```

### Request Format

```http
POST /api/token/ HTTP/1.1
Content-Type: application/json

{"username": "your_username", "password": "your_password"}
```

Both JSON and form-encoded data are accepted (the view configures `FormParser`, `MultiPartParser`, and `JSONParser`).

### Success Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"token": "13a43b8e1952bdf47a3fa48f2cbc8b73aa1279eb"}
```

### Error Response (Invalid Credentials)

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{"non_field_errors": ["Unable to log in with provided credentials."]}
```

### Evidence from Source Code

**1. URL registration in `src/paperless/urls.py` (line 81):**

```python
path("token/", views.obtain_auth_token),
```

Where `views` is imported from `rest_framework.authtoken` at line 26:

```python
from rest_framework.authtoken import views
```

**2. View implementation in `rest_framework/authtoken/views.py`:**

```python
class ObtainAuthToken(APIView):
    throttle_classes = ()
    permission_classes = ()
    parser_classes = (parsers.FormParser, parsers.MultiPartParser, parsers.JSONParser,)
    renderer_classes = (renderers.JSONRenderer,)
    serializer_class = AuthTokenSerializer

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        token, created = Token.objects.get_or_create(user=user)
        return Response({'token': token.key})
```

Key observations:
- `permission_classes = ()` — **no authentication required** to obtain a token (you provide credentials in the request body instead).
- `Token.objects.get_or_create(user=user)` — if the user already has a token, the existing one is returned; otherwise a new one is created.
- The response is always `{"token": "<key>"}` on success.

### Live Runtime Verification

```
$ curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username":"blitzy_test_user","password":"blitzy_test_pass_2024"}'

HTTP/1.1 200 OK
{"token":"13a43b8e1952bdf47a3fa48f2cbc8b73aa1279eb"}
```

```
$ curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username":"blitzy_test_user","password":"wrongpassword"}'

HTTP/1.1 400 Bad Request
{"non_field_errors":["Unable to log in with provided credentials."]}
```

### Rationale / Thinking

The token acquisition endpoint is designed to be the **entry point** for API token authentication. The typical flow for an API client is:

1. POST username/password to `/api/token/` to obtain a token
2. Store the token securely
3. Use `Authorization: Token <key>` on all subsequent requests
4. The token persists until explicitly deleted (tokens do not expire in the default DRF implementation)

The `get_or_create` pattern means that repeated token requests for the same user always return the **same token** — calling the endpoint multiple times does not generate new tokens. To get a new token, the existing one must be deleted first (via Django admin or direct database manipulation).

Note that the error response for invalid credentials returns **400 Bad Request** (not 401 Unauthorized). This is intentional: the `/api/token/` endpoint is an unauthenticated endpoint, so there is no concept of "unauthorized" — the request simply contains invalid form data.

---

## Summary of Findings

| Question | Answer |
|----------|--------|
| **Authentication header format** | `Authorization: Token <40-character-hex-string>` |
| **Document list endpoint** | `GET /api/documents/` |
| **Response top-level fields** | `count` (int), `next` (url\|null), `previous` (url\|null), `results` (array) |
| **Is it paginated?** | Yes — 25 documents per page by default, configurable via `?page_size=N` |
| **Unauthenticated response** | HTTP 401: `{"detail":"Authentication credentials were not provided."}` |
| **Invalid token response** | HTTP 401: `{"detail":"Invalid token."}` |
| **Authentication class** | `rest_framework.authentication.TokenAuthentication` in `rest_framework/authentication.py` |
| **Token model** | `rest_framework.authtoken.models.Token` in `rest_framework/authtoken/models.py` |
| **Token storage table** | `authtoken_token` (columns: `key` VARCHAR(40) PK, `user_id` INT UNIQUE FK, `created` DATETIME) |
| **Token acquisition** | `POST /api/token/` with `{"username":"...","password":"..."}` → `{"token":"<key>"}` |
| **All auth methods** | BasicAuthentication, SessionAuthentication, TokenAuthentication (+ conditional: AngularApiAuthOverride, RemoteUserAuthentication) |
| **API version headers** | `X-Api-Version: 2`, `X-Version: 1.7.0` (authenticated responses only) |

---

## Key File Reference Index

| File | Lines | Content |
|------|-------|---------|
| `src/paperless/settings.py` | 92–111 | `INSTALLED_APPS` with `rest_framework.authtoken` at line 108 |
| `src/paperless/settings.py` | 116–121 | `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` — three auth classes |
| `src/paperless/settings.py` | 129–132 | DEBUG-mode `AngularApiAuthenticationOverride` append |
| `src/paperless/settings.py` | 207–215 | Remote user authentication conditional |
| `src/paperless/urls.py` | 29 | `api_router = DefaultRouter()` |
| `src/paperless/urls.py` | 32 | `api_router.register(r"documents", UnifiedSearchViewSet)` |
| `src/paperless/urls.py` | 38–84 | URL patterns under `^api/` prefix |
| `src/paperless/urls.py` | 81 | `path("token/", views.obtain_auth_token)` |
| `src/paperless/views.py` | 8–11 | `StandardPagination` — page_size=25, page_size_query_param="page_size", max_page_size=100000 |
| `src/paperless/auth.py` | 1–41 | Three custom auth classes (AutoLogin, Angular override, Remote User) |
| `src/paperless/middleware.py` | 5–16 | `ApiVersionMiddleware` — X-Api-Version and X-Version headers |
| `src/paperless/version.py` | 1 | `__version__ = (1, 7, 0)` |
| `src/documents/views.py` | 172–196 | `DocumentViewSet` — queryset, serializer, pagination, permissions |
| `src/documents/views.py` | 377–426 | `UnifiedSearchViewSet` — extends DocumentViewSet, handles search |
| `src/documents/serialisers.py` | 201–235 | `DocumentSerializer` — 12 fields in Meta.fields tuple |
| `src/documents/models.py` | 88–206 | `Document` model — all field definitions |
| `src/documents/tests/test_api.py` | 28–65 | Test class confirms response structure (count, results) |
| `docs/api.rst` | 14–39 | API endpoints and document fields documentation |
| `docs/api.rst` | 111–145 | Authorization section — three auth methods, token format |
| `docs/api.rst` | 253–300 | API versioning and version headers |
| `rest_framework/authentication.py` | (library) | `TokenAuthentication` class — keyword, authenticate, authenticate_credentials |
| `rest_framework/authtoken/models.py` | (library) | `Token` model — key, user, created fields |
| `rest_framework/authtoken/views.py` | (library) | `ObtainAuthToken` — POST endpoint for token acquisition |
