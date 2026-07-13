# paperless-ngx REST API — Token Authentication (Runtime-Verified Q&A)

This document answers five questions about how **token-based authentication** works in the
paperless-ngx REST API, so an external tool can be integrated against it. Every answer is
**written from observed runtime output** of a locally running, default-configured instance
(the governing rule is *run first, then write*), and every behavioral claim is grounded with a
`file:line` reference into the repository source. The source tree was treated as **read-only
evidence**; nothing in it was modified. The test user, API token, seeded documents, local
SQLite database, temporary scripts, and the throwaway Python 3.9 virtualenv used for the
investigation were all removed afterward — only this document remains.

- **Repository state:** branch `paperless-ngx_542221a38dff`, HEAD `542221a38`.
- **Framework pins (version grounding):** `django==4.0.4` (`requirements.txt:L38`),
  `djangorestframework==3.13.1` (`requirements.txt:L39`).
- **Runtime pin:** Python 3.9 — repo-root `Dockerfile:L18` → `FROM python:3.9-slim-bullseye`.

> Note on the Python-3.9 citation: the pin lives in the **repo-root `Dockerfile`**, line 18.
> There is **no** `docker/Dockerfile` in this repository (the `docker/` directory contains only
> `compose/`, `docker-entrypoint.sh`, `docker-prepare.sh`, and related scripts), so the correct
> citation is `Dockerfile:L18`, never `docker/Dockerfile`.

---

## Environment / How this was run

The instance was built and run in its **default (canonical) configuration** as a normal user.
`DEBUG` defaults to `False` because `PAPERLESS_DEBUG` was left unset —
`src/paperless/settings.py:L50` → `DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`. Observations
were captured under Django's development server with banner
`WSGIServer/0.2 CPython/3.9.25`, bound to `127.0.0.1:8123`, `ALLOWED_HOSTS=['*']`, backed by a
local SQLite database. The pagination question (Q3) was observed **at a scale of 30 seeded
documents** so the real page-size cap is demonstrated rather than asserted.

Exact build/invocation commands (canonical, normal-user flow):

```bash
# 1. Disposable Python 3.9 virtualenv (matches Dockerfile:L18 pin)
python3.9 -m venv .venv && . .venv/bin/activate
# 2. Install the repository's pinned stack
pip install -r requirements.txt        # includes django==4.0.4, djangorestframework==3.13.1
# 3. Apply migrations against local SQLite with DEBUG off (canonical)
cd src && python manage.py migrate      # PAPERLESS_DEBUG unset => DEBUG=False
# 4. Create the test user (investigation-only; removed at cleanup)
python manage.py createsuperuser --username testuser --email t@e.st  # password Testpass123
# 5. Seed 30 Document rows so pagination is observable at scale (temp script; removed at cleanup)
# 6. Run the canonical dev server used only to capture responses
python manage.py runserver 127.0.0.1:8123 --noreload
```

---

## Token issuance via the real entry point (prerequisite)

An external tool obtains a token by **POSTing credentials to `/api/token/`**, served by
`rest_framework.authtoken.views.obtain_auth_token`. This view is imported at
`src/paperless/urls.py:L26` (`from rest_framework.authtoken import views`) and wired at
`src/paperless/urls.py:L81` (`path("token/", views.obtain_auth_token)`), under the `^api/`
prefix (`src/paperless/urls.py:L38-L40`). The token was obtained through this real entry point
(not a debug hook or shell shortcut).

```console
$ curl -s -X POST http://127.0.0.1:8123/api/token/ \
    -H "Content-Type: application/json" -d '{"username":"testuser","password":"Testpass123"}'
{"token":"d8291b4bc52ac6fb24808d9b0e823590dc29a66f"}    # HTTP/1.1 200 OK
```

The returned `token` value is exactly **40 characters** long, which is the shape the
`Authorization` header must carry below.

---

## Q1 — HTTP header name and format the API expects for the token

**Direct answer:** the header name is **`Authorization`**, and the format is the literal keyword
`Token`, a single space, then the 40-character key → **`Authorization: Token <key>`**. This is
Django REST Framework's `TokenAuthentication` contract, enabled at
`src/paperless/settings.py:L120`.

Command and observed output (HTTP 200 confirms the header is accepted):

```console
$ curl -s -i -H "Authorization: Token d8291b4bc52ac6fb24808d9b0e823590dc29a66f" \
    http://127.0.0.1:8123/api/documents/
HTTP/1.1 200 OK
X-Api-Version: 2
```

**Grounding:** `src/paperless/settings.py:L120` registers
`"rest_framework.authentication.TokenAuthentication"` in
`REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` (`src/paperless/settings.py:L116-L121`).

**Cause → effect:** `HTTP 200` confirms the header is accepted; `TokenAuthentication` parses the
`Authorization: Token <key>` header, looks the key up, and authenticates the bound user. The
`X-Api-Version: 2` response header reflects `AcceptHeaderVersioning`
(`src/paperless/settings.py:L122`) with allowed versions `["1", "2"]`
(`src/paperless/settings.py:L126`); the most recent allowed version (`2`) is echoed.

---

## Q2 — complete endpoint path for listing documents

**Direct answer:** the complete endpoint path is **`/api/documents/`**.

**Grounding:** the `documents` resource is registered on a DRF `DefaultRouter`
(`src/paperless/urls.py:L29`) as `UnifiedSearchViewSet`
(`src/paperless/urls.py:L32` → `api_router.register(r"documents", UnifiedSearchViewSet)`),
and the router is mounted under the `^api/` prefix (`src/paperless/urls.py:L38-L40`). The
authenticated GET shown in Q1 returned `HTTP 200` at exactly this path.

**Cause → effect:** router registration of `r"documents"` composed with the `^api/` include
produces the collection path `/api/documents/`; the trailing slash is DRF `DefaultRouter`'s
default. (The bind address/port `127.0.0.1:8123` is only a local capture convenience and is
**not** part of the endpoint contract — the contract is the path `/api/documents/`.)

---

## Q3 — response shape: top-level fields and pagination

**Direct answer:** the response is a **paginated JSON envelope** whose top-level fields are
**`count`, `next`, `previous`, `results`**. **Pagination *is* involved** — the endpoint does
**not** dump everything at once. It uses DRF `PageNumberPagination` via the project-wide
`StandardPagination` with a default `page_size` of **25** (client-overridable via
`page_size`, up to `max_page_size=100000`).

**Grounding:** `src/paperless/views.py:L8-L11` defines `StandardPagination(PageNumberPagination)`
with `page_size = 25`, `page_size_query_param = "page_size"`, and `max_page_size = 100000`; it is
applied to the documents viewset at `src/documents/views.py:L182`
(`pagination_class = StandardPagination`).

Commands run (authenticated GETs at a scale of **30 seeded documents**):

```console
$ curl -s -H "Authorization: Token d8291b4bc52ac6fb24808d9b0e823590dc29a66f" \
    http://127.0.0.1:8123/api/documents/
$ curl -s -H "Authorization: Token d8291b4bc52ac6fb24808d9b0e823590dc29a66f" \
    "http://127.0.0.1:8123/api/documents/?page=2"
$ curl -s -H "Authorization: Token d8291b4bc52ac6fb24808d9b0e823590dc29a66f" \
    "http://127.0.0.1:8123/api/documents/?page_size=5"
```

Observed (the top-level `count`, `next`, `previous`, and `len(results)` for each request):

```text
default page -> count=30, previous=null, len(results)=25,
                next="http://127.0.0.1:8123/api/documents/?page=2"
?page=2      -> count=30, previous=".../api/documents/", len(results)=5, next=null
?page_size=5 -> count=30, len(results)=5, next=".../?page=2&page_size=5"
```

Each element of `results` carries the `DocumentSerializer` fields, in this exact order
(12 fields): `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`,
`modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name`.

**Grounding:** the field tuple is declared in the serializer's `Meta`
(`src/documents/serialisers.py:L219` → `class Meta`, `src/documents/serialisers.py:L222-L235` →
the `fields` tuple). The plain list uses `DocumentSerializer` (not the search serializer) because
no `query`/`more_like_id` query parameter is present:
`UnifiedSearchViewSet.get_serializer_class` returns `DocumentSerializer` unless
`_is_search_request()` is true (`src/documents/views.py:L382-L394`).

**Cause → effect:** with 30 documents, the default page returns the first 25 items plus a `next`
link and `previous=null`; `?page=2` returns the remaining 5 items with a `previous` link and
`next=null`; `?page_size=5` proves the page size is client-overridable. This demonstrates the
real page-size cap of 25 at scale rather than asserting it — the endpoint pages results and does
not dump everything at once.

---

## Q4 — the same request WITHOUT auth: status code and error message

**Direct answer:** the status code is **`401 Unauthorized`** and the error body is
**`{"detail":"Authentication credentials were not provided."}`**.

Command and observed output:

```console
$ curl -s -i http://127.0.0.1:8123/api/documents/
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="api"
{"detail":"Authentication credentials were not provided."}
```

**Grounding:** the documents viewset enforces `permission_classes = (IsAuthenticated,)` at
`src/documents/views.py:L183`.

**Cause → effect:** with no credentials supplied, `IsAuthenticated` denies access, yielding
`401 Unauthorized`. The `WWW-Authenticate: Basic realm="api"` challenge header appears because
`BasicAuthentication` is listed **first** in the authentication stack
(`src/paperless/settings.py:L118`) and DRF emits the first authenticator's challenge; the
`detail` text is DRF's standard `NotAuthenticated` message.

---

## Q5 — which authentication class handles token auth, and which model stores tokens

**Direct answer:** the authentication class is
**`rest_framework.authentication.TokenAuthentication`** (`src/paperless/settings.py:L120`), and
the model that stores tokens is **`rest_framework.authtoken.models.Token`**, enabled via
`"rest_framework.authtoken"` in `INSTALLED_APPS` (`src/paperless/settings.py:L108`). Its database
table is **`authtoken_token`**, which stores a 40-character `key` bound to a user.

The token model was confirmed via the Django shell (investigation-only) by piping a small
inspection snippet through `python manage.py shell`. The observed output:

```console
$ python manage.py shell < inspect_token.py
auth class: rest_framework.authentication.TokenAuthentication
class attribute .model (raw): None
resolved get_model(): rest_framework.authtoken.models.Token
db table: authtoken_token
token key length: 40
token bound to user: testuser
```

Reading that output exactly: the authenticator is
`rest_framework.authentication.TokenAuthentication`. Its class-level `model` attribute defaults to
`None`; the concrete token model is resolved through `TokenAuthentication.get_model()`, which
returns `rest_framework.authtoken.models.Token`, whose database table is `authtoken_token`. The
bound token row's `key` is 40 characters long (matching the token issued in the *Token issuance*
section) and is tied to the `testuser` account. That `TokenAuthentication` is the validating class
is also confirmed at runtime by its distinctive invalid-token message:

```console
$ curl -s -i -H "Authorization: Token wrong_token" http://127.0.0.1:8123/api/documents/
HTTP/1.1 401 Unauthorized
{"detail":"Invalid token."}
```

**Grounding:** `src/paperless/settings.py:L108` registers `"rest_framework.authtoken"` (which
provides the `Token` model and its `authtoken_token` migration/table);
`src/paperless/settings.py:L120` registers `TokenAuthentication` in the auth stack.

**Cause → effect:** DRF's `TokenAuthentication.authenticate_credentials` looks up the presented
key in the `authtoken_token` table; a non-existent key (`wrong_token`) raises
`AuthenticationFailed("Invalid token.")`, producing the distinctive
`{"detail":"Invalid token."}` body — which is **distinct** from Q4's "credentials were not
provided" message and therefore proves `TokenAuthentication` is the class doing the validation.

> The DEBUG-only shim `paperless.auth.AngularApiAuthenticationOverride`
> (`src/paperless/auth.py:L18-L33`) is appended to the authentication stack **only when `DEBUG`
> is true** (`src/paperless/settings.py:L129-L132`). Under the canonical `DEBUG=False`
> configuration it is inert and plays **no** part in token authentication.

---

## Coverage-pass checklist

Every named item from the request is answered:

- [x] Q1 header **name** answered (`Authorization`) — with observed `HTTP 200`.
- [x] Q1 header **format** answered (`Token <key>`, 40-character key) — with observed `HTTP 200`.
- [x] Q2 complete **endpoint path** answered (`/api/documents/`).
- [x] Q3 **top-level fields** answered (`count`, `next`, `previous`, `results`).
- [x] Q3 **pagination vs. dump-everything** answered (paginated; default `page_size=25`; observed at 30 documents).
- [x] Q3 **results item fields** enumerated (12 `DocumentSerializer` fields, in order).
- [x] Q4 **status code** answered (`401 Unauthorized`).
- [x] Q4 **error message** answered (`{"detail":"Authentication credentials were not provided."}`).
- [x] Q5 **authentication class** named (`rest_framework.authentication.TokenAuthentication`).
- [x] Q5 **token model** named (`rest_framework.authtoken.models.Token`; table `authtoken_token`).
- [x] Token obtained via the **real entry point** (`POST /api/token/`).
- [x] Error/secondary paths exercised (no-auth → `401`; invalid token → `401` "Invalid token.").
- [x] Cleanup performed; source tree left untouched.

---

## Non-canonical caveats

The following choices were made for local capture convenience only and do **not** affect any
answer above:

- **SQLite** was used locally (paperless's default when no external database host is configured).
  The authentication class, token model, header contract, endpoint path, status codes, and
  pagination envelope are database-agnostic, so this choice does not change any answer.
- **Bind `127.0.0.1:8123`** was chosen for capture convenience; the port is **not** part of the
  endpoint contract — the contract is the path `/api/documents/`.
- **Django's development `runserver` (`WSGIServer`)** was used, which is appropriate for observing
  application-level authentication behavior; it does not change the HTTP contract a production
  server would present.

All temporary investigation artifacts (the Python 3.9 virtualenv, local SQLite database, seed
scripts, test user, API token, and seeded documents) were removed on completion. The repository
source tree is unchanged; only this document was added.
