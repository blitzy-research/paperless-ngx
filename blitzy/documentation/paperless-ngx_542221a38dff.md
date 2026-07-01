# Paperless-ngx REST API — Token Authentication (Integrator's Guide)

This document answers, with verbatim runtime evidence and exact `file:line` source
citations, how **token authentication** works in the paperless-ngx REST API so that
external tools can be integrated against it. It was produced by a strictly **read-only**
investigation: the genuine paperless code was actually built and run as a **live HTTP
server**, real output was captured, and every behavioral claim below is placed directly
next to the **exact command** that produced it and the **verbatim output** it emitted.

The pinned framework versions used for the observations are the ones an integrator will
actually hit against this build: **`Django==4.0.4`** and **`djangorestframework==3.13.1`**,
on **Python 3.9.23** — the exact runtime of this project (see the `Server` header and the
DRF introspection below).

> **Read-only guarantee.** The investigation ran the real paperless application inside a
> **disposable Docker container**. It created only ephemeral rows in the container's
> throwaway SQLite database (`/app/data/db.sqlite3`): a test user, one API token, and 30
> sample documents. **No file in the repository was modified**, and all temporary
> observation scripts (`/tmp/obs/*.py` inside the container) were removed afterward. See the
> [Cleanup / repository integrity](#cleanup--repository-integrity) section.

---

## TL;DR

| # | Question | One-line answer |
|---|----------|-----------------|
| **Q1** | Token header NAME and FORMAT | Header **name** is `Authorization`; **value format** is `Token <key>` — e.g. `Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3` |
| **Q2** | Documents-listing endpoint PATH | `/api/documents/` (trailing slash included) |
| **Q3** | Top-level JSON response fields | Exactly four keys: `count`, `next`, `previous`, `results` (standard DRF pagination envelope); each `results[]` item has 12 fields |
| **Q4** | Pagination | **YES** — paginated; it does **not** dump everything at once. Default page size is **25**; adjust with the `page_size` query param, navigate with `page` |
| **Q5** | Unauthenticated request | `HTTP/1.1 401 Unauthorized` + body `{"detail":"Authentication credentials were not provided."}` + header `WWW-Authenticate: Basic realm="api"` |
| **Q6** | (a) Authentication class / (b) token model | (a) `rest_framework.authentication.TokenAuthentication`; (b) `rest_framework.authtoken.models.Token`, stored in DB table `authtoken_token` |

---

## How this was verified (runtime harness)

The evidence below was captured by running the **real paperless application** as a live dev
server on `127.0.0.1:8000`, inside the project's own Docker container (Python 3.9.23,
`Django==4.0.4`, `djangorestframework==3.13.1`). This exercises the exact components an
integrator hits: the real `UnifiedSearchViewSet`, `DocumentSerializer`, `StandardPagination`,
`ApiVersionMiddleware`, and the exact `DEFAULT_AUTHENTICATION_CLASSES` order.

Because `curl`/`wget` are **not** installed in the container, HTTP requests were issued with
a tiny raw-socket client, `raw.py` (its full source is in the
[Reproduction harness](#reproduction-harness)); it prints the **verbatim** raw HTTP/1.1
response the server emits. Small `requests`-based scripts (`parse.py`, `page.py`, `obj.py`)
parse the JSON body, and `python manage.py shell -c` was used for source introspection and
SQL. **Every output block below is immediately preceded by the exact command that produced
it.**

**Command run** (bring up the environment — from the paperless source root `/app/src`):

```bash
# 1) migrate: creates auth_user, authtoken_token, and the documents tables
python manage.py migrate

# 2) create the test user, mint a token via the ORM, seed 30 documents
python manage.py shell -c "
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
from documents.models import Document
import hashlib
u, _ = User.objects.get_or_create(username='testuser'); u.set_password('testpass123'); u.save()
tok, _ = Token.objects.get_or_create(user=u)
print('USER id=%s username=%s' % (u.id, u.username))
print('TOKEN key=%s len=%s user_id=%s created=%s' % (tok.key, len(tok.key), tok.user_id, tok.created.isoformat()))
for i in range(1, 31):
    Document.objects.get_or_create(title='Test Document %d' % i, defaults=dict(content='content body %d' % i, mime_type='application/pdf', checksum=hashlib.md5(('doc%d' % i).encode()).hexdigest()))
print('DOCUMENTS count=%s' % Document.objects.count())
"

# 3) start the real paperless dev server (DEBUG=False and PAPERLESS_AUTO_LOGIN_USERNAME unset — both defaults)
python manage.py runserver 127.0.0.1:8000 --noreload --insecure
```

**Observed output (verbatim)** — the seed step:

```text
USER id=2 username=testuser
TOKEN key=1e80755d969c5bbd8bf61516e74ff81085d115a3 len=40 user_id=2 created=2026-07-01T22:22:33.658654+00:00
DOCUMENTS count=30
```

**Observed output (verbatim)** — the server boot (proves it is the genuine paperless app on
`Django 4.0.4`, and that no `--skip-checks` was needed):

```text
Performing system checks...

System check identified no issues (0 silenced).
July 01, 2026 - 22:23:20
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

This establishes the preconditions used throughout: a test user (`id=2`, `username=testuser`),
a 40-character token key (`1e80755d969c5bbd8bf61516e74ff81085d115a3`), and 30 seeded
documents — enough to exercise pagination, since the default page size is 25.

---

## Q1 — Token header NAME and FORMAT

**Answer:** the HTTP header **NAME** is `Authorization` and its **FORMAT/value** is
`Token <key>` — i.e. the literal string `Token`, one space, then the 40-character key.
Full example:

```text
Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Command run** (authenticated request that SUCCEEDS with this header; `--status` prints only
the status line):

```bash
python /tmp/obs/raw.py GET /api/documents/ "Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3" --status
```

**Observed output (verbatim)** — `200 OK`, proving the header form is accepted:

```http
HTTP/1.1 200 OK
```

To prove the keyword is specifically `Token` (and that a well-formed but **unmatched** key is
rejected as an *invalid token*), an invalid key was sent with the identical header form:

**Command run** (INVALID token key — 40 hex chars, well-formed `Token <key>` header):

```bash
python /tmp/obs/raw.py GET /api/documents/ "Authorization: Token deadbeefdeadbeefdeadbeefdeadbeefdeadbeef"
```

**Observed output (verbatim):**

```http
HTTP/1.1 401 Unauthorized
Date: Wed, 01 Jul 2026 22:23:36 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
Content-Length: 27
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Invalid token."}
```

The `{"detail":"Invalid token."}` body confirms the `Token` keyword *was* recognized and the
key *was* looked up — it simply did not match a stored token.

**Command run** (prove the keyword literal directly, via runtime introspection):

```bash
python manage.py shell -c "from rest_framework.authentication import TokenAuthentication as T; print('TokenAuthentication.keyword =', repr(T.keyword)); print('TokenAuthentication.authenticate_header(None) =', repr(T().authenticate_header(None)))"
```

**Observed output (verbatim):**

```text
TokenAuthentication.keyword = 'Token'
TokenAuthentication.authenticate_header(None) = 'Token'
```

**Command run** (distinguish an **invalid token key** from an **absent header** — see the
rationale; uses DRF's `APIRequestFactory` to call the authenticator directly):

```bash
python manage.py shell -c "
from rest_framework.authentication import TokenAuthentication
from rest_framework.test import APIRequestFactory
rf = APIRequestFactory()
print('absent Authorization header ->', TokenAuthentication().authenticate(rf.get('/api/documents/')))
print('wrong keyword (Bearer)      ->', TokenAuthentication().authenticate(rf.get('/api/documents/', HTTP_AUTHORIZATION='Bearer xyz')))
try:
    TokenAuthentication().authenticate(rf.get('/api/documents/', HTTP_AUTHORIZATION='Token deadbeefdeadbeefdeadbeefdeadbeefdeadbeef'))
except Exception as e:
    print('invalid token key           -> raises', type(e).__name__ + ':', repr(str(e)))
"
```

**Observed output (verbatim):**

```text
absent Authorization header -> None
wrong keyword (Bearer)      -> None
invalid token key           -> raises AuthenticationFailed: 'Invalid token.'
```

**Source citation & rationale:**

- `TokenAuthentication` is the DRF authenticator registered at
  `src/paperless/settings.py:L120` (`"rest_framework.authentication.TokenAuthentication"`),
  and its keyword is the literal `Token` (introspection above:
  `TokenAuthentication.keyword = 'Token'`).
- **Invalid vs. absent — an important distinction (do not conflate them):**
  - An **invalid/unmatched token key** — a *well-formed* `Authorization: Token <key>` header
    whose key is not in the database — reaches
    `TokenAuthentication.authenticate_credentials()`, which raises
    `AuthenticationFailed('Invalid token.')`. That is the observed
    `{"detail":"Invalid token."}` `401` body above.
  - An **absent `Authorization` header** (or a header with a *different* keyword such as
    `Bearer`) causes `TokenAuthentication.authenticate()` to **return `None`** — it does
    **not** raise. (Runtime proof above: both `absent ... -> None` and
    `wrong keyword (Bearer) -> None`.) No authenticator then succeeds, so the `IsAuthenticated`
    permission denies the request with the **different** body
    `{"detail":"Authentication credentials were not provided."}` seen in
    [Q5](#q5--unauthenticated-request-status-code-and-error-message).
  - This matches DRF's own `rest_framework/authentication.py`, where
    `TokenAuthentication.authenticate()` returns `None` when the header is missing or the
    keyword does not match, and only `authenticate_credentials()` raises
    `AuthenticationFailed('Invalid token.')` for an unmatched key.
- Corroborated by the project's own docs at `docs/api.rst:L143`
  (``Authorization: Token <token>``).

---

## Q2 — Documents-listing endpoint PATH

**Answer:** `/api/documents/` (trailing slash included).

**Command run:**

```bash
python /tmp/obs/raw.py GET /api/documents/ "Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3" --status
```

**Observed output (verbatim)** — the status line proves the path resolves and returns data:

```http
HTTP/1.1 200 OK
```

**Source citation & rationale:**

- Router registration: `api_router.register(r"documents", UnifiedSearchViewSet)` at
  `src/paperless/urls.py:L32`, where `api_router = DefaultRouter()` at
  `src/paperless/urls.py:L29`. A DRF `DefaultRouter` maps the registered prefix
  `documents` to the collection route with a trailing slash.
- The router is mounted under the `r"^api/"` prefix at `src/paperless/urls.py:L40` via
  `+ api_router.urls` at `src/paperless/urls.py:L83`. Together these yield the collection
  path `/api/documents/`.
- Corroborated by `docs/api.rst:L16` (``/api/documents/``).

---

## Q3 — Top-level JSON response fields

**Answer:** the top-level JSON object has exactly **four** keys — `count`, `next`,
`previous`, `results` — the standard DRF pagination envelope. Each element of `results[]`
is a document object with these **12** fields: `id`, `correspondent`, `document_type`,
`title`, `content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`,
`original_file_name`, `archived_file_name`.

**Command run** (capture the response **headers** verbatim; `--headers` prints the status
line and headers only):

```bash
python /tmp/obs/raw.py GET /api/documents/ "Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3" --headers
```

**Observed output (verbatim):**

```http
HTTP/1.1 200 OK
Date: Wed, 01 Jul 2026 22:24:05 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 8727
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

**Command run** (capture the genuine **first 480 bytes** of the response body; `--body`
prints the body only, and `head -c 480` truncates to exactly 480 bytes — this is a real,
tool-produced prefix, not a hand-edited excerpt):

```bash
python /tmp/obs/raw.py GET /api/documents/ "Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3" --body | head -c 480
```

**Observed output (verbatim — exactly 480 bytes; it therefore ends mid-token inside the
second result object, which is expected of a byte truncation):**

```text
{"count":30,"next":"http://127.0.0.1:8000/api/documents/?page=2","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"Test Document 30","content":"content body 30","tags":[],"created":"2026-07-01T22:22:33.741082Z","modified":"2026-07-01T22:22:33.741195Z","added":"2026-07-01T22:22:33.741086Z","archive_serial_number":null,"original_file_name":"2026-07-01 Test Document 30.pdf","archived_file_name":null},{"id":29,"correspondent":null,"document_ty
```

**Command run** (parse the JSON and print the exact top-level and item key sets):

```bash
python /tmp/obs/parse.py http://127.0.0.1:8000/api/documents/ 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Observed output (verbatim):**

```text
TOP-LEVEL KEYS: ['count', 'next', 'previous', 'results']
count = 30
next = http://127.0.0.1:8000/api/documents/?page=2
previous = None
len(results) on this page = 25
results[0] KEYS = ['id', 'correspondent', 'document_type', 'title', 'content', 'tags', 'created', 'modified', 'added', 'archive_serial_number', 'original_file_name', 'archived_file_name']
```

**Command run** (dump one full `results[0]` object exactly as `json.dumps(..., indent=2)`
produces it — enumerates the 12 per-item fields):

```bash
python /tmp/obs/obj.py http://127.0.0.1:8000/api/documents/ 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Observed output (verbatim):**

```json
{
  "id": 30,
  "correspondent": null,
  "document_type": null,
  "title": "Test Document 30",
  "content": "content body 30",
  "tags": [],
  "created": "2026-07-01T22:22:33.741082Z",
  "modified": "2026-07-01T22:22:33.741195Z",
  "added": "2026-07-01T22:22:33.741086Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-01 Test Document 30.pdf",
  "archived_file_name": null
}
```

**Source citation & rationale:**

- The four envelope keys come from DRF's default `PageNumberPagination` response: the
  viewset binds `pagination_class = StandardPagination` at `src/documents/views.py:L182`,
  and `StandardPagination` (`src/paperless/views.py:L8-L11`) does **not** override
  `get_paginated_response`, so DRF's default `{count, next, previous, results}` envelope
  applies.
- The per-item 12 fields come verbatim from `DocumentSerializer.Meta.fields` at
  `src/documents/serialisers.py:L222-L234` (the `fields = (...)` tuple; `id` at L223 …
  `archived_file_name` at L234).
- **Serializer-branch nuance:** `UnifiedSearchViewSet.get_serializer_class`
  (`src/documents/views.py:L382-L392`) returns `DocumentSerializer` for a **plain**
  listing and only switches to `SearchResultSerializer` when a `query` or `more_like_id`
  query parameter is present (the class is defined at `src/documents/views.py:L377`; the
  plain-list default is `serializer_class = DocumentSerializer` at
  `src/documents/views.py:L181`). Because the request above had **no** `query`, the standard
  document fields are returned — which is exactly what an integrator listing documents gets.

---

## Q4 — Pagination

**Answer:** **YES** — the endpoint is paginated (it does **not** dump everything at once).
The default page size is **25**; the page size is client-adjustable via the `page_size`
query parameter, and pages are navigated via `page`. The envelope's `count` is the TOTAL
across all pages, while `results` holds only the current page.

**Command run** (default page — 30 docs seeded, `page.py` prints a one-line summary):

```bash
python /tmp/obs/page.py http://127.0.0.1:8000/api/documents/ 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Observed output (verbatim)** — 30 total, **25** on the first page, non-null `next`,
demonstrating the default page size of **25** (it does not dump all 30):

```text
count=30 len(results)=25 next=http://127.0.0.1:8000/api/documents/?page=2 previous=None
```

**Command run** (`?page_size=5` — prove the `page_size` query parameter is honored):

```bash
python /tmp/obs/page.py "http://127.0.0.1:8000/api/documents/?page_size=5" 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Observed output (verbatim)** — the page now holds 5 items and `next` carries the
`page_size=5` param forward:

```text
count=30 len(results)=5 next=http://127.0.0.1:8000/api/documents/?page=2&page_size=5 previous=None
```

**Command run** (`?page=2` — a later page **at the default page size**, where `previous`
becomes non-null):

```bash
python /tmp/obs/page.py "http://127.0.0.1:8000/api/documents/?page=2" 1e80755d969c5bbd8bf61516e74ff81085d115a3
```

**Observed output (verbatim):**

```text
count=30 len(results)=5 next=None previous=http://127.0.0.1:8000/api/documents/
```

> **Reading these three blocks.** The **default page-size** proof is the first block
> (`len(results)=25` with `count=30` and a non-null `next`), demonstrating the page size of
> 25. The second block demonstrates the `page_size` query parameter (page shrinks to 5, and
> `next` carries `page_size=5` forward). The third block demonstrates that `previous` becomes
> non-null on a later page. Note that the third request carries **no** `page_size` parameter,
> and its `previous` link (`http://127.0.0.1:8000/api/documents/`) also carries **no**
> `page_size`: with the **default page size of 25** and a **total `count` of 30**, page 2
> naturally contains the **remaining 5** documents (25 on page 1 + 5 on page 2 = 30), and
> `next=None` because it is the last page. This is expected and further demonstrates
> pagination. Each block is presented next to the specific claim it proves.

**Source citation & rationale:**

- `StandardPagination(PageNumberPagination)` sets `page_size = 25`
  (`src/paperless/views.py:L9`), `page_size_query_param = "page_size"`
  (`src/paperless/views.py:L10`), and `max_page_size = 100000`
  (`src/paperless/views.py:L11`).
- It is bound to the viewset at `src/documents/views.py:L182`.
- Because 30 > 25, the first page returns 25 items with a non-null `next` and `count = 30`,
  proving paging rather than a full dump. Corroborated by `docs/api.rst:L171-L176`.

---

## Q5 — Unauthenticated request: status code and error message

**Answer:** re-issuing the identical `GET /api/documents/` with **no** `Authorization`
header returns **`HTTP/1.1 401 Unauthorized`**, a `WWW-Authenticate: Basic realm="api"`
challenge header, and the JSON body `{"detail":"Authentication credentials were not provided."}`.

**Validity precondition.** This test is only meaningful because the authentication
*bypasses* were confirmed inactive:

- `DEBUG=False`, so the DEBUG-gated `AngularApiAuthenticationOverride` was **not** appended
  to the authenticators (`src/paperless/settings.py:L129-L132`; DEBUG-only apps/overrides are
  added only under `if DEBUG:` per `src/paperless/settings.py:L113-L114`).
- `PAPERLESS_AUTO_LOGIN_USERNAME` was **unset**, so `AutoLoginMiddleware` did not silently
  authenticate the request (`src/paperless/settings.py:L193-L199`).
- `IsAuthenticated` is enforced on the view (`src/documents/views.py:L183`).

The bypass classes themselves live in `src/paperless/auth.py`
(`AutoLoginMiddleware` at `:L9`, `AngularApiAuthenticationOverride` at `:L18`,
`HttpRemoteUserMiddleware` at `:L36`).

**Command run:**

```bash
python /tmp/obs/raw.py GET /api/documents/
```

**Observed output (verbatim — full status line, headers, and body):**

```http
HTTP/1.1 401 Unauthorized
Date: Wed, 01 Jul 2026 22:24:39 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
Content-Length: 58
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Authentication credentials were not provided."}
```

**Command run** (why `401` and not `403` — introspect the first authenticator's challenge):

```bash
python manage.py shell -c "
from rest_framework.authentication import BasicAuthentication, TokenAuthentication
from rest_framework.test import APIRequestFactory
req = APIRequestFactory().get('/api/documents/')
print('BasicAuthentication.authenticate_header(req) =', repr(BasicAuthentication().authenticate_header(req)))
print('TokenAuthentication.authenticate_header(req) =', repr(TokenAuthentication().authenticate_header(req)))
"
```

**Observed output (verbatim):**

```text
BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'
TokenAuthentication.authenticate_header(req) = 'Token'
```

**Source citation & rationale:**

- The observed status is **401 (not 403)** because DRF derives the unauthenticated status
  from the **first** authenticator in `DEFAULT_AUTHENTICATION_CLASSES`. Paperless lists
  `BasicAuthentication` **first** at `src/paperless/settings.py:L118` (the order is Basic
  at L118, Session at L119, Token at L120, within `src/paperless/settings.py:L117-L121`).
- `BasicAuthentication.authenticate_header()` returns the non-null challenge
  `Basic realm="api"` (introspection above), so DRF emits `401` plus a matching
  `WWW-Authenticate: Basic realm="api"` header — which is exactly the header observed in the
  response. (Had the first authenticator returned no challenge, DRF would instead emit `403`.)
- **This is the "absent header" path from [Q1](#q1--token-header-name-and-format):** with no
  `Authorization` header, `TokenAuthentication.authenticate()` returns `None` (it does not
  raise `Invalid token.`), so the request reaches `IsAuthenticated`, which produces the
  `{"detail":"Authentication credentials were not provided."}` body seen above.
- **Version-headers nuance (one claim, one piece of evidence):** the `X-Api-Version` /
  `X-Version` headers are **ABSENT** from this `401` response — compare the header block
  above with the [Q3](#q3--top-level-json-response-fields) authenticated header block, where
  both are present (`X-Api-Version: 2` and `X-Version: 1.7.0`). This is because
  `ApiVersionMiddleware` only sets them `if request.user.is_authenticated`
  (`src/paperless/middleware.py:L11`, with the headers written at `:L13-L14`). Since the
  unauthenticated request has no authenticated user, the middleware skips those headers.

---

## Q6 — Code trace: authentication class and token model

**Answer:**

- **(a) Authentication class** = `rest_framework.authentication.TokenAuthentication`.
- **(b) Token model** = `rest_framework.authtoken.models.Token`, stored in the database
  table `authtoken_token`, enabled by adding the `rest_framework.authtoken` app to
  `INSTALLED_APPS`.

**Command run** (DRF runtime introspection, after `manage.py shell` performs `django.setup()`):

```bash
python manage.py shell -c "
import rest_framework
from rest_framework.authentication import TokenAuthentication
from rest_framework.authtoken.models import Token
print('DRF version =', rest_framework.VERSION)
print('TokenAuthentication FQN =', TokenAuthentication.__module__ + '.' + TokenAuthentication.__name__)
print('TokenAuthentication.keyword =', repr(TokenAuthentication.keyword))
print('Token model FQN =', Token.__module__ + '.' + Token.__name__)
print('Token db_table =', Token._meta.db_table)
kf = Token._meta.get_field('key')
print('Token.key field =', type(kf).__name__, 'max_length =', kf.max_length)
"
```

**Observed output (verbatim):**

```text
DRF version = 3.13.1
TokenAuthentication FQN = rest_framework.authentication.TokenAuthentication
TokenAuthentication.keyword = 'Token'
Token model FQN = rest_framework.authtoken.models.Token
Token db_table = authtoken_token
Token.key field = CharField max_length = 40
```

**Command run** (the token table's actual schema, straight from SQLite's `sqlite_master`):

```bash
python manage.py shell -c "
from django.db import connection
with connection.cursor() as c:
    c.execute(\"SELECT sql FROM sqlite_master WHERE type='table' AND name='authtoken_token'\")
    print(c.fetchone()[0])
"
```

**Observed output (verbatim):**

```sql
CREATE TABLE "authtoken_token" ("key" varchar(40) NOT NULL PRIMARY KEY, "created" datetime NOT NULL, "user_id" integer NOT NULL UNIQUE REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED)
```

**Command run** (the actual token row created for the test user):

```bash
python manage.py shell -c "
from django.db import connection
with connection.cursor() as c:
    c.execute('SELECT key, user_id, created FROM authtoken_token')
    for row in c.fetchall():
        print('key=%s | user_id=%s | created=%s' % row)
"
```

**Observed output (verbatim):**

```text
key=1e80755d969c5bbd8bf61516e74ff81085d115a3 | user_id=2 | created=2026-07-01 22:22:33.658654
```

The stored `key` equals the token used to authenticate throughout this document, `user_id=2`
matches the created `testuser`, and the `user_id` UNIQUE constraint (see the DDL) means one
token per user.

**Command run** (the HTTP token-mint endpoint — `POST /api/token/` returns the key):

```bash
python /tmp/obs/raw.py POST /api/token/ --data "username=testuser&password=testpass123"
```

**Observed output (verbatim):**

```http
HTTP/1.1 200 OK
Date: Wed, 01 Jul 2026 22:24:58 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Allow: POST, OPTIONS
X-Frame-Options: SAMEORIGIN
Content-Length: 52
Vary: Accept-Language, Origin, Cookie
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"token":"1e80755d969c5bbd8bf61516e74ff81085d115a3"}
```

**Source citation & rationale:**

- The authentication class is registered at `src/paperless/settings.py:L120`
  (`"rest_framework.authentication.TokenAuthentication"`).
- The token model is backed by the `"rest_framework.authtoken"` app at
  `src/paperless/settings.py:L108`; that app's migrations create the `authtoken_token`
  table (confirmed by the DDL above). The `key` is a 40-char primary key, and `user_id`
  is UNIQUE (one token per user).
- The token-mint endpoint is `path("token/", views.obtain_auth_token)` at
  `src/paperless/urls.py:L81`, where `views` is `rest_framework.authtoken.views` imported
  at `src/paperless/urls.py:L26`. The returned key
  (`{"token":"1e80755d969c5bbd8bf61516e74ff81085d115a3"}`) equals the stored key, tying the
  HTTP mint path to the `authtoken_token` row.
- **Two mint paths.** The same key was also created directly via the ORM
  (`Token.objects.get_or_create(user=...)`, per the seed output in
  [How this was verified](#how-this-was-verified-runtime-harness)). Both the HTTP endpoint
  (`POST /api/token/`) and the ORM yield a key usable in the `Authorization: Token <key>`
  header.

---

## Contextual details

- **Version headers and their gate.** Authenticated responses carry `X-Api-Version: 2`
  and `X-Version: 1.7.0` (see the Q3 header block). These are added by `ApiVersionMiddleware`
  (`src/paperless/middleware.py:L5`): `X-Api-Version` is the last entry of `ALLOWED_VERSIONS`
  (`src/paperless/middleware.py:L13`) and `X-Version` is the paperless version string
  `1.7.0` (`src/paperless/middleware.py:L14`, derived from `paperless.version.__version__ = (1, 7, 0)`).
  They appear **only** on authenticated responses because of the
  `if request.user.is_authenticated` gate (`src/paperless/middleware.py:L11`) — which is
  exactly why they are absent from the Q5 `401` response.
- **API versioning config.** `AcceptHeaderVersioning` with `DEFAULT_VERSION = "1"` and
  `ALLOWED_VERSIONS = ["1", "2"]` (`src/paperless/settings.py:L122-L126`). This explains
  why `X-Api-Version` is `2` — the last allowed version.
- **Invocation / pagination pattern reference.** The existing test-suite pattern
  `"/api/documents/?query=content&page=N&page_size=10"`
  (`src/documents/tests/test_api.py:L456,L466,L486,L488`) mirrors how the endpoint is
  paged and searched, and matches the `page` / `page_size` parameters demonstrated in Q4.
- **Entry point.** `src/manage.py` (with `DJANGO_SETTINGS_MODULE="paperless.settings"`) is
  the Django entrypoint used for migrations, user creation, and the token-minting /
  inspection shell.

---

## External corroboration (Django REST Framework docs & source)

The observed behavior matches Django REST Framework's authoritative documentation and
source. The corroboration is tied to the locally observed evidence below (not merely a
restatement of the docs).

- **DRF authentication guide** — <https://www.django-rest-framework.org/api-guide/authentication/>.
  The guide states the key is prefixed by the string literal `Token`, with whitespace
  separating the two (its example is `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`).
  This matches the locally observed header form
  `Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3` accepted with `200 OK` in
  Q1. The guide also states that `rest_framework.authtoken` must be in `INSTALLED_APPS` and
  that `manage.py migrate` creates the token table — matching the observed
  `authtoken_token` DDL in Q6 and the `rest_framework.authtoken` app at
  `src/paperless/settings.py:L108`. Finally, it states that denied unauthenticated requests
  yield "HTTP 401 Unauthorized" with a `WWW-Authenticate` header — matching the observed
  Q5 `401` plus `WWW-Authenticate: Basic realm="api"`.
- **DRF source** — <https://github.com/encode/django-rest-framework/blob/main/rest_framework/authentication.py>.
  `class TokenAuthentication` sets `keyword = 'Token'`; its `authenticate()` returns `None`
  when the header is missing or the keyword does not match, and only
  `authenticate_credentials()` raises `AuthenticationFailed('Invalid token.')` for an
  unmatched key. This matches the locally observed introspection
  (`TokenAuthentication.keyword = 'Token'`) and the absent-vs-invalid demonstration in Q1 on
  the pinned `djangorestframework==3.13.1`.

---

## Coverage checklist

- [x] **Q1 — header NAME + FORMAT:** NAME `Authorization`; FORMAT `Token <key>` (e.g.
  `Authorization: Token 1e80755d969c5bbd8bf61516e74ff81085d115a3`). Evidence: `200 OK` on
  valid token; `{"detail":"Invalid token."}` on a bad token key; introspection
  `TokenAuthentication.keyword = 'Token'`; and the absent-vs-invalid demo
  (`absent ... -> None`, `invalid token key -> raises AuthenticationFailed: 'Invalid token.'`).
  Citation: `src/paperless/settings.py:L120`, `docs/api.rst:L143`.
- [x] **Q2 — endpoint PATH:** `/api/documents/`. Evidence: `200 OK`. Citation:
  `src/paperless/urls.py:L29,L32,L40,L83`, `docs/api.rst:L16`.
- [x] **Q3 — top-level fields:** `count`, `next`, `previous`, `results`, plus the 12
  per-item fields (`id` … `archived_file_name`). Evidence: `--headers` block, genuine
  `head -c 480` body prefix, `TOP-LEVEL KEYS` / `results[0] KEYS` parse output, and the full
  `results[0]` object. Citation: `src/documents/views.py:L182`,
  `src/paperless/views.py:L8-L11`, `src/documents/serialisers.py:L222-L234`.
- [x] **Q4 — pagination:** YES; default page size 25; `page_size` and `page` query params.
  Evidence: `count=30` / `len(results)=25` / non-null `next`; `?page_size=5` block;
  `?page=2` block (remaining 5, `next=None`, `previous` non-null). Citation:
  `src/paperless/views.py:L9-L11`, `src/documents/views.py:L182`.
- [x] **Q5 — unauthenticated status + message:** `401 Unauthorized` +
  `{"detail":"Authentication credentials were not provided."}` +
  `WWW-Authenticate: Basic realm="api"`. Evidence: full verbatim response;
  `BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'`. Citation:
  `src/paperless/settings.py:L117-L121` (Basic first at L118), `src/documents/views.py:L183`.
- [x] **Q6a — authentication class:** `rest_framework.authentication.TokenAuthentication`.
  Evidence: introspection `TokenAuthentication FQN = ...`. Citation:
  `src/paperless/settings.py:L120`.
- [x] **Q6b — token model:** `rest_framework.authtoken.models.Token`, table `authtoken_token`.
  Evidence: introspection `Token model FQN` / `Token db_table`; the `authtoken_token` DDL;
  the actual token row; and `POST /api/token/` returning the same key. Citation:
  `src/paperless/settings.py:L108`, `src/paperless/urls.py:L26,L81`.
- [x] **Nuance — invalid token key vs. absent header:** invalid key →
  `AuthenticationFailed('Invalid token.')` (`{"detail":"Invalid token."}`); absent header →
  `authenticate()` returns `None` → `IsAuthenticated` →
  `{"detail":"Authentication credentials were not provided."}`. Evidence: Q1 demo output.
- [x] **Nuance — serializer branch:** plain list → `DocumentSerializer`;
  `query`/`more_like_id` → `SearchResultSerializer` (`src/documents/views.py:L377,L382-L392`).
- [x] **Nuance — 401 vs 403:** determined by the first authenticator (`BasicAuthentication`)
  returning a non-null challenge; introspection `BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'`.
- [x] **Nuance — version headers only when authenticated:** present in Q3, absent in Q5,
  gated by `if request.user.is_authenticated` (`src/paperless/middleware.py:L11`).

---

## Cleanup / repository integrity

- The investigation ran the real paperless application inside a **disposable Docker
  container**; all runtime state lives in the container's throwaway SQLite database
  (`/app/data/db.sqlite3`) and is discarded when the container is removed.
- It created only **ephemeral rows**: a `testuser`, one `authtoken_token` row, and 30 sample
  `documents`. None of these are in the repository.
- The temporary observation scripts used to produce the output above
  (`/tmp/obs/raw.py`, `parse.py`, `page.py`, `obj.py`) live **inside the container**, outside
  the repository, and were removed after use.
- **No file in the repository was modified.** `git status --porcelain` on the host repository
  is empty except for this single new document,
  `blitzy/documentation/paperless-ngx_542221a38dff.md`.

---

## Reproduction harness

This section makes every command above fully reproducible. Run everything **inside the
project's Docker container** (Python 3.9.23, `Django==4.0.4`, `djangorestframework==3.13.1`),
from the paperless source root `/app/src`, with `DEBUG=False` and
`PAPERLESS_AUTO_LOGIN_USERNAME` unset (both are defaults).

**1. Bring up the environment** (migrate, seed the user/token/30 docs, start the server) —
exactly the commands shown in [How this was verified](#how-this-was-verified-runtime-harness).

**2. The observation scripts** (written under `/tmp/obs/` in the container — outside the
repository — and deleted afterward). Because `curl`/`wget` are not installed, `raw.py` is a
minimal raw-socket HTTP client that prints the verbatim raw response:

```python
# /tmp/obs/raw.py — minimal raw-HTTP client; modes: --status | --headers | --body | (full)
# Usage: raw.py METHOD PATH [HEADER ...] [--data BODY]
import socket, sys

def main():
    args = sys.argv[1:]
    mode, data, positional = "full", None, []
    i = 0
    while i < len(args):
        a = args[i]
        if a in ("--status", "--headers", "--body"):
            mode = a[2:]
        elif a == "--data":
            i += 1; data = args[i]
        else:
            positional.append(a)
        i += 1
    method, path = positional[0], positional[1]
    headers = positional[2:]
    lines = [f"{method} {path} HTTP/1.1", "Host: 127.0.0.1:8000", "Connection: close"]
    body = b""
    if data is not None:
        body = data.encode()
        lines.append("Content-Type: application/x-www-form-urlencoded")
        lines.append(f"Content-Length: {len(body)}")
    lines += headers
    raw = ("\r\n".join(lines) + "\r\n\r\n").encode() + body
    s = socket.create_connection(("127.0.0.1", 8000))
    s.sendall(raw)
    buf = b""
    while True:
        c = s.recv(4096)
        if not c:
            break
        buf += c
    s.close()
    text = buf.decode("latin-1")
    sep = text.find("\r\n\r\n")
    head = text[:sep] if sep != -1 else text
    resp_body = text[sep + 4:] if sep != -1 else ""
    if mode == "status":
        sys.stdout.write(head.split("\r\n")[0] + "\n")
    elif mode == "headers":
        sys.stdout.write(head.replace("\r\n", "\n") + "\n")
    elif mode == "body":
        sys.stdout.write(resp_body)
    else:
        sys.stdout.write(text.replace("\r\n", "\n"))

main()
```

```python
# /tmp/obs/parse.py — print the parsed top-level + item key sets
import sys, requests
url, token = sys.argv[1], sys.argv[2]
j = requests.get(url, headers={"Authorization": f"Token {token}"}).json()
print("TOP-LEVEL KEYS:", list(j.keys()))
print("count =", j["count"])
print("next =", j["next"])
print("previous =", j["previous"])
print("len(results) on this page =", len(j["results"]))
print("results[0] KEYS =", list(j["results"][0].keys()))
```

```python
# /tmp/obs/page.py — one-line pagination summary
import sys, requests
url, token = sys.argv[1], sys.argv[2]
j = requests.get(url, headers={"Authorization": f"Token {token}"}).json()
print(f"count={j['count']} len(results)={len(j['results'])} next={j['next']} previous={j['previous']}")
```

```python
# /tmp/obs/obj.py — pretty-print results[0] exactly as json.dumps produces it
import sys, json, requests
url, token = sys.argv[1], sys.argv[2]
r = requests.get(url, headers={"Authorization": f"Token {token}"})
print(json.dumps(r.json()["results"][0], indent=2))
```

**3. Issue the requests** shown per question above (`raw.py`, `parse.py`, `page.py`,
`obj.py`, and the `manage.py shell -c` introspection/SQL commands). Every output block in this
document is the verbatim result of the command printed immediately above it.

---

*End of document. All runtime blocks above are verbatim captures from the live in-container
server (`Server: WSGIServer/0.2 CPython/3.9.23`, `Django 4.0.4`,
`djangorestframework 3.13.1`); all `file:line` references were confirmed against the source
branch `paperless-ngx_542221a38dff` (commit `542221a38dff`).*
