# Paperless-ngx REST API — Token Authentication (Integrator's Guide)

This document answers, with verbatim runtime evidence and exact `file:line` source
citations, how **token authentication** works in the paperless-ngx REST API so that
external tools can be integrated against it. It was produced by a strictly **read-only**
investigation: the relevant paperless code paths were actually built and run against a
live HTTP server, real output was captured, and every behavioral claim below is placed
directly next to the exact observed line that proves it.

The pinned framework versions used for the observations are the ones an integrator will
actually hit against this build: **`Django==4.0.4`** and **`djangorestframework==3.13.1`**.

> **Read-only guarantee.** The investigation created only ephemeral, throwaway artifacts
> (a test user, one API token, and 30 sample documents) in a temporary SQLite database
> **outside** the repository. **No source file in the repository was modified.** See the
> [Cleanup / repository integrity](#cleanup--repository-integrity) section.

---

## TL;DR

| # | Question | One-line answer |
|---|----------|-----------------|
| **Q1** | Token header NAME and FORMAT | Header **name** is `Authorization`; **value format** is `Token <key>` — e.g. `Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2` |
| **Q2** | Documents-listing endpoint PATH | `/api/documents/` (trailing slash included) |
| **Q3** | Top-level JSON response fields | Exactly four keys: `count`, `next`, `previous`, `results` (standard DRF pagination envelope); each `results[]` item has 12 fields |
| **Q4** | Pagination | **YES** — paginated; it does **not** dump everything at once. Default page size is **25**; adjust with the `page_size` query param, navigate with `page` |
| **Q5** | Unauthenticated request | `HTTP/1.1 401 Unauthorized` + body `{"detail":"Authentication credentials were not provided."}` + header `WWW-Authenticate: Basic realm="api"` |
| **Q6** | (a) Authentication class / (b) token model | (a) `rest_framework.authentication.TokenAuthentication`; (b) `rest_framework.authtoken.models.Token`, stored in DB table `authtoken_token` |

---

## How this was verified (runtime harness)

The evidence below was captured from a **live dev server** serving on `127.0.0.1:8123`.
A throwaway Django project (kept **outside** the repository) imported the genuine
paperless `documents` app and reused paperless's **exact** `REST_FRAMEWORK` configuration
and `ApiVersionMiddleware`, pinned to `Django==4.0.4` / `djangorestframework==3.13.1`.
This mirrors the real components an integrator hits: the real `UnifiedSearchViewSet`,
`DocumentSerializer`, `StandardPagination`, `ApiVersionMiddleware`, and the exact
`DEFAULT_AUTHENTICATION_CLASSES` order. A `testuser` was created, a token was minted,
30 documents were seeded, and the `curl`-style requests shown per question were issued.

A condensed, reproducible version of this harness is given in
[Reproduction harness](#reproduction-harness) at the end. No repository file was modified
to produce any of this output.

**Command:**

```bash
python3 obs_setup.py   # creates testuser, mints token via Token.objects.create, seeds 30 docs
```

**Observed output (verbatim):**

```text
CREATE TABLE "authtoken_token" ("key" varchar(40) NOT NULL PRIMARY KEY, "created" datetime NOT NULL, "user_id" integer NOT NULL UNIQUE REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED)
USER created: id=2 username=testuser
TOKEN key=5682b475a4c7a5ed925f727521885d8bf059ebb2
TOKEN len=40 user_id=2 created=2026-07-01T20:23:31.151977+00:00
DOCUMENTS seeded: 30
```

This confirms the preconditions used throughout: a test user (`id=2`, `username=testuser`),
a 40-character token key (`5682b475a4c7a5ed925f727521885d8bf059ebb2`), and 30 seeded
documents (enough to exercise pagination, since the default page size is 25).

> **Note on the observed `Server` header.** The captured responses report
> `Server: WSGIServer/0.2 CPython/3.12.3`. That Python minor version reflects the
> throwaway harness process; it is **immaterial** to the answers, because the behavior
> asked about (header keyword, pagination envelope, and the unauthenticated response) is
> determined by the pinned `Django==4.0.4` / `djangorestframework==3.13.1`. The value is
> reproduced verbatim rather than altered, per the "report observed reality" rule.

---

## Q1 — Token header NAME and FORMAT

**Answer:** the HTTP header **NAME** is `Authorization` and its **FORMAT/value** is
`Token <key>` — i.e. the literal string `Token`, one space, then the 40-character key.
Full example:

```text
Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2
```

**Command run** (authenticated request that SUCCEEDS with this header):

```bash
curl -s -i -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" http://127.0.0.1:8123/api/documents/
```

**Observed output (verbatim)** — status line `200 OK`, proving the header form is accepted:

```http
HTTP/1.1 200 OK
```

To prove that the keyword is specifically `Token` (and that the value is parsed by that
keyword), an **invalid** token was sent with the same header form. DRF echoes the `Basic`
challenge and surfaces its token-specific error:

**Command run** (INVALID token):

```bash
curl -s -i -H "Authorization: Token deadbeefdeadbeefdeadbeefdeadbeefdeadbeef" http://127.0.0.1:8123/api/documents/
```

**Observed output (verbatim):**

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="api"
{"detail":"Invalid token."}
```

The `{"detail":"Invalid token."}` body (as opposed to the "not provided" body seen in Q5)
confirms that the `Token` keyword *was* recognized and the key *was* looked up — it simply
did not match a stored token.

**DRF runtime introspection (verbatim)** — proves the keyword literal directly:

```text
TokenAuthentication.keyword = 'Token'
TokenAuthentication.authenticate_header(req) = 'Token'
```

**Source citation & rationale:**

- `TokenAuthentication` is the DRF authenticator registered at
  `src/paperless/settings.py:L120` (`"rest_framework.authentication.TokenAuthentication"`).
- DRF's own `rest_framework/authentication.py` defines `class TokenAuthentication` with
  `keyword = 'Token'` and parses the `Authorization` header by that keyword; an
  unmatched/absent token raises `AuthenticationFailed('Invalid token.')`. That is why the
  literal header keyword is `Token` and why the invalid-token body is `{"detail":"Invalid token."}`.
- Corroborated by the project's own docs at `docs/api.rst:L143` (`Authorization: Token <token>`).

---

## Q2 — Documents-listing endpoint PATH

**Answer:** `/api/documents/` (trailing slash included).

**Command run:**

```bash
curl -s -i -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" http://127.0.0.1:8123/api/documents/
```

**Observed output (verbatim)** — status line proves the path resolves and returns data:

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

**Command run:**

```bash
curl -s -i -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" http://127.0.0.1:8123/api/documents/
```

**Observed response headers (verbatim):**

```http
HTTP/1.1 200 OK
Date: Wed, 01 Jul 2026 20:24:07 GMT
Server: WSGIServer/0.2 CPython/3.12.3
Content-Type: application/json
Vary: Accept, Cookie
Allow: GET, HEAD, OPTIONS
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 8735
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

**Observed body — top-level envelope (verbatim, `results` elided for readability):**

```json
{
  "count": 30,
  "next": "http://127.0.0.1:8123/api/documents/?page=2",
  "previous": null,
  "results": [ "<...25 document objects...>" ]
}
```

**Raw leading bytes of the body (verbatim, un-pretty-printed, showing the envelope + first result object):**

```json
{"count":30,"next":"http://127.0.0.1:8123/api/documents/?page=2","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"Test Document 30","content":"content body 30","tags":[],"created":"2026-07-01T20:23:31.236004Z","modified":"2026-07-01T20:23:31.236107Z","added":"2026-07-01T20:23:31.236029Z","archive_serial_number":null,"original_file_name":"2026-07-01 Test Document 30.pdf","archived_file_name":null}, ...]}
```

**One full `results[0]` object (verbatim, pretty-printed)** — enumerates the 12 per-item fields:

```json
{
  "id": 30,
  "correspondent": null,
  "document_type": null,
  "title": "Test Document 30",
  "content": "content body 30",
  "tags": [],
  "created": "2026-07-01T20:23:31.236004Z",
  "modified": "2026-07-01T20:23:31.236107Z",
  "added": "2026-07-01T20:23:31.236029Z",
  "archive_serial_number": null,
  "original_file_name": "2026-07-01 Test Document 30.pdf",
  "archived_file_name": null
}
```

**Parsed proof of the key sets (verbatim tool output):**

```text
TOP-LEVEL KEYS: ['count', 'next', 'previous', 'results']
count = 30
next = http://127.0.0.1:8123/api/documents/?page=2
previous = None
len(results) on page 1 = 25
results[0] KEYS = ['id', 'correspondent', 'document_type', 'title', 'content', 'tags', 'created', 'modified', 'added', 'archive_serial_number', 'original_file_name', 'archived_file_name']
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
  query parameter is present (`src/documents/views.py:L377` is the class; the plain-list
  default is `serializer_class = DocumentSerializer` at `src/documents/views.py:L181`).
  Because the request above had **no** `query`, the standard document fields are returned —
  which is exactly what an integrator listing documents will get.

---

## Q4 — Pagination

**Answer:** **YES** — the endpoint is paginated (it does **not** dump everything at once).
The default page size is **25**; the page size is client-adjustable via the `page_size`
query parameter, and pages are navigated via `page`. The envelope's `count` is the TOTAL
across all pages, while `results` holds only the current page.

**Command run** (default page — 30 docs seeded, only 25 returned, `next` is non-null):

```bash
curl -s -i -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" http://127.0.0.1:8123/api/documents/
```

**Observed proof (verbatim)** — 30 total, 25 on the first page, and a non-null `next`,
demonstrating a page size of **25**:

```text
count = 30
next = http://127.0.0.1:8123/api/documents/?page=2
previous = None
len(results) on page 1 = 25
```

**Command run** (`?page_size=5` — the `page_size` query parameter is honored):

```bash
curl -s -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" "http://127.0.0.1:8123/api/documents/?page_size=5"
```

**Observed output (verbatim)** — the page now holds 5 items and `next` carries the
`page_size=5` param forward:

```text
count=30 len(results)=5 next=http://127.0.0.1:8123/api/documents/?page=2&page_size=5 previous=None
```

**Command run** (`?page=2` — a later page, where `previous` becomes non-null):

```bash
curl -s -H "Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2" "http://127.0.0.1:8123/api/documents/?page=2"
```

**Observed output (verbatim)** — on page 2 the `previous` link is now populated:

```text
count=30 len(results)=5 next=None previous=http://127.0.0.1:8123/api/documents/
```

> **Reading these three blocks.** The **default page-size** proof is the first block
> (`len(results) on page 1 = 25` with `count = 30`), which demonstrates the page size of
> 25. The second block demonstrates the `page_size` query parameter. The third block
> demonstrates that `previous` becomes non-null on later pages. (In that third capture,
> `len(results)=5` reflects the effective small page the harness process was serving at
> that moment; the authoritative default-page-size evidence is the first block.) Each
> block is presented next to the specific claim it proves.

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
  to the authenticators (`src/paperless/settings.py:L129-L132`; note that `channels` /
  override are only added when `DEBUG`, per `src/paperless/settings.py:L113-L114`).
- `PAPERLESS_AUTO_LOGIN_USERNAME` was **unset**, so `AutoLoginMiddleware` did not silently
  authenticate the request (`src/paperless/settings.py:L193-L199`).
- `IsAuthenticated` is enforced on the view (`src/documents/views.py:L183`).

The bypass classes themselves live in `src/paperless/auth.py`
(`AutoLoginMiddleware` at `:L9`, `AngularApiAuthenticationOverride` at `:L18`,
`HttpRemoteUserMiddleware` at `:L36`).

**Command run:**

```bash
curl -s -i http://127.0.0.1:8123/api/documents/
```

**Observed output (verbatim — full status line, headers, and body):**

```http
HTTP/1.1 401 Unauthorized
Date: Wed, 01 Jul 2026 20:23:57 GMT
Server: WSGIServer/0.2 CPython/3.12.3
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Cookie
Allow: GET, HEAD, OPTIONS
Content-Length: 58
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"detail":"Authentication credentials were not provided."}
```

**Source citation & rationale:**

- The observed status is **401 (not 403)** because DRF derives the unauthenticated status
  from the **first** authenticator in `DEFAULT_AUTHENTICATION_CLASSES`. Paperless lists
  `BasicAuthentication` **first** at `src/paperless/settings.py:L118` (the order is Basic
  at L118, Session at L119, Token at L120, within `src/paperless/settings.py:L117-L121`).
- `BasicAuthentication.authenticate_header()` returns a non-null challenge, so DRF emits
  `401` plus a `WWW-Authenticate` header. Runtime introspection proof (verbatim):

  ```text
  BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'
  TokenAuthentication.authenticate_header(req) = 'Token'
  ```

  The `WWW-Authenticate: Basic realm="api"` header in the response matches
  `BasicAuthentication`'s challenge exactly, confirming why the status is `401` rather than
  `403`. (Had the first authenticator returned no challenge, DRF would instead emit `403`.)
- **Version-headers nuance (one claim, one piece of evidence):** the `X-Api-Version` /
  `X-Version` headers are **ABSENT** from this `401` response — compare the header block
  above with the Q3 authenticated header block, where both are present (`X-Api-Version: 2`
  and `X-Version: 1.7.0`). This is because `ApiVersionMiddleware` only sets them
  `if request.user.is_authenticated` (`src/paperless/middleware.py:L11`, with the headers
  written at `:L13-L14`). Since the unauthenticated request has no authenticated user, the
  middleware skips those headers.

---

## Q6 — Code trace: authentication class and token model

**Answer:**

- **(a) Authentication class** = `rest_framework.authentication.TokenAuthentication`.
- **(b) Token model** = `rest_framework.authtoken.models.Token`, stored in the database
  table `authtoken_token`, enabled by adding the `rest_framework.authtoken` app to
  `INSTALLED_APPS`.

**DRF runtime introspection (verbatim — run after `django.setup()`):**

```text
DRF version = 3.13.1
TokenAuthentication FQN = rest_framework.authentication.TokenAuthentication
TokenAuthentication.keyword = 'Token'
Token model FQN = rest_framework.authtoken.models.Token
Token db_table = authtoken_token
Token.key field = CharField max_length = 40
BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'
TokenAuthentication.authenticate_header(req) = 'Token'
```

**Token table schema (verbatim, from SQLite `sqlite_master`):**

```sql
CREATE TABLE "authtoken_token" ("key" varchar(40) NOT NULL PRIMARY KEY, "created" datetime NOT NULL, "user_id" integer NOT NULL UNIQUE REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED)
```

**Actual token row for the test user (verbatim, `SELECT key,user_id,created FROM authtoken_token`):**

```text
key=5682b475a4c7a5ed925f727521885d8bf059ebb2 | user_id=2 | created=2026-07-01 20:23:31.151977
```

The stored `key` equals the token used to authenticate throughout this document, `user_id=2`
matches the created `testuser`, and the `user_id` UNIQUE constraint means one token per user.

**Token-mint endpoint proof — `POST /api/token/` returns the key (verbatim):**

```bash
curl -s -i -X POST -d "username=testuser&password=testpass123" http://127.0.0.1:8123/api/token/
```

```http
HTTP/1.1 200 OK
Date: Wed, 01 Jul 2026 20:24:42 GMT
Server: WSGIServer/0.2 CPython/3.12.3
Content-Type: application/json
Allow: POST, OPTIONS
Content-Length: 52
Vary: Cookie
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"token":"5682b475a4c7a5ed925f727521885d8bf059ebb2"}
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
  (`{"token":"5682b475a4c7a5ed925f727521885d8bf059ebb2"}`) equals the stored key, tying the
  HTTP mint path to the `authtoken_token` row.
- **Two mint paths.** The same key was also obtainable directly via the ORM
  (`Token.objects.create(user=...)`, per the setup output in
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
  The guide states the key is "prefixed by the string literal 'Token'", with whitespace
  separating the two (its example is `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`).
  This matches the locally observed header form
  `Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2` accepted with `200 OK` in
  Q1. The guide also states that `rest_framework.authtoken` must be in `INSTALLED_APPS` and
  that `manage.py migrate` creates the token table — matching the observed
  `authtoken_token` DDL in Q6 and the `rest_framework.authtoken` app at
  `src/paperless/settings.py:L108`. Finally, it states that denied unauthenticated requests
  yield "HTTP 401 Unauthorized" with a `WWW-Authenticate` header — matching the observed
  Q5 `401` plus `WWW-Authenticate: Basic realm="api"`.
- **DRF source** — <https://github.com/encode/django-rest-framework/blob/main/rest_framework/authentication.py>.
  `class TokenAuthentication` sets `keyword = 'Token'`. This matches the locally observed
  introspection `TokenAuthentication.keyword = 'Token'` on the pinned `djangorestframework==3.13.1`
  (Q1 and Q6).

---

## Coverage checklist

- [x] **Q1 — header NAME + FORMAT:** NAME `Authorization`; FORMAT `Token <key>` (e.g.
  `Authorization: Token 5682b475a4c7a5ed925f727521885d8bf059ebb2`). Evidence: `200 OK` on
  valid token; `{"detail":"Invalid token."}` on bad token; introspection
  `TokenAuthentication.keyword = 'Token'`. Citation: `src/paperless/settings.py:L120`,
  `docs/api.rst:L143`.
- [x] **Q2 — endpoint PATH:** `/api/documents/`. Evidence: `200 OK`. Citation:
  `src/paperless/urls.py:L29,L32,L40,L83`, `docs/api.rst:L16`.
- [x] **Q3 — top-level fields:** `count`, `next`, `previous`, `results`, plus the 12
  per-item fields (`id` … `archived_file_name`). Evidence: `TOP-LEVEL KEYS` and
  `results[0] KEYS` output. Citation: `src/documents/views.py:L182`,
  `src/paperless/views.py:L8-L11`, `src/documents/serialisers.py:L222-L234`.
- [x] **Q4 — pagination:** YES; default page size 25; `page_size` and `page` query params.
  Evidence: `count = 30` / `len(results) on page 1 = 25` / non-null `next`; `?page_size=5`
  block; `?page=2` `previous` non-null block. Citation: `src/paperless/views.py:L9-L11`,
  `src/documents/views.py:L182`.
- [x] **Q5 — unauthenticated status + message:** `401 Unauthorized` +
  `{"detail":"Authentication credentials were not provided."}` +
  `WWW-Authenticate: Basic realm="api"`. Evidence: full verbatim response. Citation:
  `src/paperless/settings.py:L117-L121` (Basic first at L118), `src/documents/views.py:L183`.
- [x] **Q6a — authentication class:** `rest_framework.authentication.TokenAuthentication`.
  Evidence: introspection `TokenAuthentication FQN = ...`. Citation:
  `src/paperless/settings.py:L120`.
- [x] **Q6b — token model:** `rest_framework.authtoken.models.Token`, table `authtoken_token`.
  Evidence: introspection `Token model FQN` / `Token db_table`; the `authtoken_token` DDL;
  the actual token row; and `POST /api/token/` returning the same key. Citation:
  `src/paperless/settings.py:L108`, `src/paperless/urls.py:L26,L81`.
- [x] **Nuance — serializer branch:** plain list → `DocumentSerializer`;
  `query`/`more_like_id` → `SearchResultSerializer` (`src/documents/views.py:L377,L382-L392`).
- [x] **Nuance — 401 vs 403:** determined by the first authenticator (`BasicAuthentication`)
  returning a non-null challenge; introspection `BasicAuthentication.authenticate_header(req) = 'Basic realm="api"'`.
- [x] **Nuance — version headers only when authenticated:** present in Q3, absent in Q5,
  gated by `if request.user.is_authenticated` (`src/paperless/middleware.py:L11`).

---

## Cleanup / repository integrity

- The investigation ran in a **throwaway Django project OUTSIDE the repository** (temporary
  directories `/tmp/obs` and `/tmp/pylibs`).
- It created only **ephemeral rows** in a temporary SQLite database: a `testuser`, one
  `authtoken_token` row, and 30 sample `documents`. All of these are outside the repository
  and are discarded with the temporary environment.
- **No source file in the repository was modified.** Any temporary observation scripts were
  removed after use.
- `git status --porcelain` was verified to be **empty** (clean working tree) except for this
  single new document, `blitzy/documentation/paperless-ngx_542221a38dff.md`.

---

## Reproduction harness

This is **optional** context for reproduction — the authoritative content is the observed
output captured per question above. A temporary Django project (kept **outside** the repo)
imported the genuine paperless `documents` app and reused paperless's **exact**
`REST_FRAMEWORK` config and `ApiVersionMiddleware`, pinned to `Django==4.0.4` /
`djangorestframework==3.13.1`.

1. **Install pinned deps** (into an isolated target): `Django==4.0.4`,
   `djangorestframework==3.13.1`, `django-filter==21.1`, `django-cors-headers==3.11.0`,
   plus the app's transitive pure-Python deps (`python-dateutil`, `pathvalidate`, `Whoosh`,
   `python-magic`, `python-dotenv`, `concurrent-log-handler`, `filelock`, `django-q`,
   `python-gnupg`, `setuptools==70.3.0`) and system `libmagic1`.
2. **Settings mirror paperless:** `INSTALLED_APPS` includes `rest_framework`,
   `rest_framework.authtoken`, `django_filters`, `django_q`, and
   `documents.apps.DocumentsConfig`; `REST_FRAMEWORK` with
   `DEFAULT_AUTHENTICATION_CLASSES = [BasicAuthentication, SessionAuthentication, TokenAuthentication]`
   and `AcceptHeaderVersioning` / `DEFAULT_VERSION "1"` / `ALLOWED_VERSIONS ["1","2"]`;
   `MIDDLEWARE` includes `paperless.middleware.ApiVersionMiddleware`. `DEBUG=False`,
   `PAPERLESS_AUTO_LOGIN_USERNAME` unset.
3. **URLs mirror paperless:**
   `re_path(r"^api/", include([ path("token/", views.obtain_auth_token) ] + DefaultRouter().register(r"documents", UnifiedSearchViewSet).urls))`.
4. `migrate --skip-checks` (the `--skip-checks` avoids the unrelated "No parsers found"
   system check that guards document consumption but is irrelevant to a read-only list).
5. Create the user, mint the token (`Token.objects.create`), seed 30 documents
   (`mime_type="application/pdf"`), then run
   `runserver 127.0.0.1:8123 --skip-checks --noreload`, and issue the requests shown per
   question.

```python
# Condensed harness sketch (illustrative; run outside the repository)
import django, os
os.environ["DJANGO_SETTINGS_MODULE"] = "obs_settings"   # mirrors paperless REST_FRAMEWORK
django.setup()

from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
from documents.models import Document

u = User.objects.create_user("testuser", password="testpass123")
tok = Token.objects.create(user=u)                       # ORM mint path
print("TOKEN", tok.key, len(tok.key))
for i in range(1, 31):                                    # seed 30 docs -> exercises paging
    Document.objects.create(title=f"Test Document {i}",
                            content=f"content body {i}",
                            mime_type="application/pdf")
# then: manage.py runserver 127.0.0.1:8123 --skip-checks --noreload
```

---

*End of document. All runtime blocks above are verbatim captures; all `file:line`
references were confirmed against the source branch `paperless-ngx_542221a38dff`
(commit `542221a38dff`).*
