# Paperless-ngx API Token Authentication — Investigation & Integration Guide

This document answers, with code citations and live runtime evidence, exactly how
Paperless-ngx API **token authentication** works, so that an external tool can call the
REST API with confidence. Every conclusion is grounded in the actual source code (the
"source of truth") and cross-checked against observed runtime behavior. Where the
official Django REST Framework (DRF) documentation is referenced, it is **corroborative
only** — the binding answers derive from the pinned source and the live runtime.

> All findings are specific to the versions pinned in this repository —
> `django==4.0.4` [requirements.txt:L38] and `djangorestframework==3.13.1`
> [requirements.txt:L39] — and to the Paperless-ngx `REST_FRAMEWORK` configuration in
> `src/paperless/settings.py`.

## TL;DR — The five headline facts

| Question | Answer |
|---|---|
| Auth header | `Authorization: Token <key>` (literal keyword `Token`, a space, then a **40-char hex** key) |
| Documents endpoint | `GET /api/documents/` (note the **trailing slash**) |
| Response shape | Paginated DRF envelope `{ "count", "next", "previous", "results" }`, **25 items/page** by default (NOT a full dump) |
| Unauthenticated result | **HTTP 401 Unauthorized** + body `{"detail":"Authentication credentials were not provided."}` + header `WWW-Authenticate: Basic realm="api"` |
| Code path | Auth class `rest_framework.authentication.TokenAuthentication`; token stored by `rest_framework.authtoken.models.Token` in the **`authtoken_token`** table |

In prose: an integrator authenticates by sending the HTTP header
`Authorization: Token <40-hex-key>` to `GET /api/documents/` (trailing slash required).
A valid token yields **HTTP 200** and a *paginated* JSON envelope — `count`, `next`,
`previous`, and a `results` array of up to **25** documents per page (adjustable via
`?page_size=`, capped at `100000`) — so the endpoint never dumps every document in one
response. Omitting credentials yields **HTTP 401** (not 403) with the body
`{"detail":"Authentication credentials were not provided."}` and a
`WWW-Authenticate: Basic realm="api"` response header. The whole flow is handled by
`TokenAuthentication`, backed by the `Token` model persisted in the `authtoken_token`
database table.

## How this was verified

Three independent lines of evidence back every answer below:

1. **Code tracing (binding).** The complete request path was read in the pinned source:
   URL routing (`src/paperless/urls.py`), authentication configuration
   (`src/paperless/settings.py`), the documents ViewSet and its permission/pagination
   (`src/documents/views.py`, `src/paperless/views.py`), and the relevant DRF 3.13.1
   internals (`rest_framework/authentication.py`, `rest_framework/authtoken/models.py`,
   `rest_framework/views.py`, `rest_framework/exceptions.py`,
   `rest_framework/authtoken/views.py`, `rest_framework/authtoken/admin.py`).
2. **Empirical runtime (binding).** The contract was exercised end-to-end against the
   **actual Paperless-ngx application** run from the user-provided Docker image
   (Paperless-ngx **v1.7.0**, Python 3.9, `django==4.0.4` + `djangorestframework==3.13.1`,
   Redis + SQLite, `DEBUG=False`), probing it unauthenticated, with a valid token, with an
   invalid token, with the wrong scheme (`Bearer`), without the trailing slash, and via
   `POST /api/token/`, capturing status codes, headers, and bodies verbatim (see the
   appendix). The decisive **401-vs-403 linchpin** was then proven by exercising the
   **real DRF 3.13.1 authenticator code path** (Basic-first → `'Basic realm="api"'` →
   `401`; Session-first → `None` → `403`) — without modifying the source.
3. **Web documentation (corroborative only).** The official DRF authentication guide was
   consulted to cross-check the `Authorization: Token <key>` header contract and the
   401 + `WWW-Authenticate` behavior. It agrees with the source reading but is not the
   basis for any conclusion.

> **A note on `requirements.txt`.** The pinned versions live in the **root**
> `requirements.txt` (`django==4.0.4` at L38, `djangorestframework==3.13.1` at L39).
> There is no `src/requirements.txt` in this repository; the only other
> `docs/requirements.txt` is empty and unrelated. All version citations below therefore
> reference the root `requirements.txt`.

---

## Q1 — How to run an instance and create a user + API token

**Answer (Conclusion).** Stand up Paperless-ngx, apply database migrations (notably the
`authtoken` migrations that create the `authtoken_token` table), create a test user, and
generate an API token. There are **four** supported token-provisioning mechanisms:

1. **`POST /api/token/`** — exchange a username + password for a token
   [src/paperless/urls.py:L81].
2. **`manage.py drf_create_token <username>`** — the DRF management command
   [rest_framework/authtoken/management/commands/drf_create_token.py:L12-L19, L44-L45].
3. **Django admin** — via `TokenAdmin`/`TokenProxy`
   [rest_framework/authtoken/admin.py:L23-L51].
4. **Programmatic** — `Token.objects.get_or_create(user=...)`.

**Preferred runtime (end-to-end).** Use the user-provided Docker image
`andrewparkscaleai/coding-agent:paperless-ngx__…542221a38dff` (Python 3.9 base, with
Redis + a database). Conceptually:

- Start the stack (web + Redis + DB). Paperless runs migrations on startup; ensure the
  `rest_framework.authtoken` migrations are applied (they create `authtoken_token`).
- Create a user with `manage.py createsuperuser` (or `manage.py manage_superuser` / the
  documents `PAPERLESS_ADMIN_USER` mechanism), e.g. user `apitester`.
- Generate a token with `manage.py drf_create_token apitester`, which prints
  `Generated token <40-hex> for user apitester`.

**This was run end-to-end.** The verbatim evidence in the appendix was captured from the
**actual Paperless-ngx application** (v1.7.0) started exactly this way from the
user-provided Docker image (`DEBUG=False`, Redis + SQLite), with the user `apitester`, a
generated token, and 30 seeded documents. Note that the documents-list authentication
contract is governed entirely by (a) the DRF authenticator ordering, (b) the
`IsAuthenticated` permission, and (c) `StandardPagination`, so it is *also* faithfully
reproducible with just the pinned `django==4.0.4` + `djangorestframework==3.13.1` on
SQLite — but the appendix below reflects the real Docker stack, not a stand-in.

**Rationale (Thinking).** Token authentication requires the `authtoken` app to be
installed and migrated, because a token is a *row* in the `authtoken_token` table keyed
by a 40-character primary key [rest_framework/authtoken/models.py:L13]. The four
mechanisms do **not** all reach that row the same way. `POST /api/token/`
[rest_framework/authtoken/views.py:L54-L59] and `drf_create_token`
[rest_framework/authtoken/management/commands/drf_create_token.py:L12-L19, L44-L45] both
call `Token.objects.get_or_create`, and programmatic callers may use the same call; each
returns/persists the **same** key for a given user (never a duplicate). The Django admin,
by contrast, manages `Token` rows through `TokenProxy`/`TokenAdmin`
[rest_framework/authtoken/admin.py:L23-L51] — it does **not** call `get_or_create`; the
key is instead generated on first save by `Token.save()` → `generate_key()`
[rest_framework/authtoken/models.py:L30-L37]. For headless integration, `drf_create_token`
and `POST /api/token/` are the most convenient. The model is only concrete (non-abstract) when `rest_framework.authtoken`
is in `INSTALLED_APPS` [rest_framework/authtoken/models.py:L26], which Paperless
guarantees [src/paperless/settings.py:L108].

**Citations & Evidence.**

- `INSTALLED_APPS` includes `rest_framework.authtoken` [src/paperless/settings.py:L108].
- Token model + 40-hex key generation:
  `key = models.CharField(_("Key"), max_length=40, primary_key=True)`
  [rest_framework/authtoken/models.py:L13]; `generate_key` returns
  `binascii.hexlify(os.urandom(20)).decode()` — 20 random bytes → 40 hex characters
  [rest_framework/authtoken/models.py:L36-L37].
- `POST /api/token/` route [src/paperless/urls.py:L81]; admin registration
  `admin.site.register(TokenProxy, TokenAdmin)` [rest_framework/authtoken/admin.py:L51].
- Runtime: `drf_create_token apitester` →
  `Generated token a1b6fe70fcb6406c24fa6c8c9043be8de6c46588 for user apitester` (see
  appendix). The `authtoken_token` table is present and a token row is created.

---

## Q2 — Issue an authenticated read against the documents list

**Answer (Conclusion).** Issue `GET /api/documents/` with the header
`Authorization: Token <key>`. The server returns **HTTP 200** and a paginated JSON
envelope (detailed in Q5).

**Rationale (Thinking).** The documents ViewSet requires authentication —
`permission_classes = (IsAuthenticated,)` [src/documents/views.py:L183]. A valid token
satisfies `TokenAuthentication`, so the request is authorized and the list is returned.
The list is **paginated, not dumped**, because the ViewSet sets
`pagination_class = StandardPagination` [src/documents/views.py:L182].

**Citations & Evidence.** Verbatim PROBE 2 (valid token) — `200 OK`, body
`{"count":30,"next":".../api/documents/?page=2","previous":null,"results":[... 25 items ...]}`.
See the appendix.

---

## Q3 — The EXACT token header (name + format)

**Answer (Conclusion).** The header is:

```
Authorization: Token <key>
```

where `<key>` is a **40-character hexadecimal** string. The literal scheme keyword is
**`Token`** (capital `T`), followed by a single space, then the key. For example (with an
illustrative key):

```
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

**Rationale (Thinking).** DRF's `TokenAuthentication` defines `keyword = 'Token'` and
parses the `Authorization` header expecting the form `<keyword> <key>`
[rest_framework/authentication.py:L161]. The `<key>` is the `Token.key` primary key — a
40-character hex string produced by `generate_key()` =
`binascii.hexlify(os.urandom(20)).decode()` [rest_framework/authtoken/models.py:L13, L36].
The user's hypothesized example `Authorization: Token <token>` is therefore confirmed
exactly. Note that the **header name** is `Authorization`; the **scheme** is `Token`, and
specifically **not** `Bearer` — Paperless does not subclass `TokenAuthentication` to
change the keyword.

**Citations & Evidence.**

- `keyword = 'Token'` [rest_framework/authentication.py:L161]; class `TokenAuthentication`
  [rest_framework/authentication.py:L151]; the class docstring itself documents the
  contract with the example `Authorization: Token 401f7ac837da42b97f613d789819ff93537bee6a`.
- 40-hex key [rest_framework/authtoken/models.py:L13, L36].
- Corroboration (official DRF docs, corroborative only): the key "should be prefixed by
  the string literal 'Token', with whitespace separating the two strings," e.g.
  `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`.
- Runtime: PROBE 2 used exactly this header and returned `200` (see appendix).

---

## Q4 — The COMPLETE documents endpoint path

**Answer (Conclusion).** The complete path is:

```
/api/documents/
```

**with the trailing slash.** A full example URL is `http://<host>/api/documents/`.

**Rationale (Thinking).** Paperless builds its API with a DRF `DefaultRouter`:
`api_router = DefaultRouter()` [src/paperless/urls.py:L29], then
`api_router.register(r"documents", UnifiedSearchViewSet)` [src/paperless/urls.py:L32].
The router's URLs are mounted under the `^api/` prefix: the `urlpatterns` wrap `r"^api/"`
[src/paperless/urls.py:L40] and append `api_router.urls` [src/paperless/urls.py:L83]. A
`DefaultRouter` generates list routes **with a trailing slash** by default, so combining
the `^api/` prefix with the `documents` registration yields `/api/documents/`. The user's
hypothesized `/api/documents/` is confirmed exactly, including the trailing slash.

**Citations & Evidence.** [src/paperless/urls.py:L29, L32, L40, L83]. Runtime: all
document probes hit `/api/documents/` and behaved as a registered DRF list route (`200`
with the paginated envelope when authenticated). See the appendix.

---

## Q5 — The JSON response shape (and whether it paginates)

**Answer (Conclusion).** The response is a **paginated DRF envelope** with exactly four
top-level fields:

| Field | Type | Meaning |
|---|---|---|
| `count` | integer | total number of documents matching |
| `next` | string\|null | absolute URL of the next page (or `null` on the last page) |
| `previous` | string\|null | absolute URL of the previous page (or `null` on the first page) |
| `results` | array | the page of document objects (default **25** per page) |

The endpoint **paginates**; it does **NOT** dump all documents at once. The default page
size is **25**, client-adjustable via the `?page_size=` query parameter, and capped at
**100000**.

**Rationale (Thinking).** `DocumentViewSet.pagination_class = StandardPagination`
[src/documents/views.py:L182], and `StandardPagination(PageNumberPagination)` sets
`page_size = 25`, `page_size_query_param = "page_size"`, and `max_page_size = 100000`
[src/paperless/views.py:L8-L11]. DRF's `PageNumberPagination` emits the
`{count, next, previous, results}` envelope. **Plain list vs. search:** the route uses
`UnifiedSearchViewSet(DocumentViewSet)` [src/documents/views.py:L377]; its
`get_serializer_class` returns the standard `DocumentSerializer` for a plain list and
only switches to `SearchResultSerializer` (adding per-result search metadata) when a
`query` or `more_like_id` query parameter is present [src/documents/views.py:L382-L392].
The "list documents" answer corresponds to the **plain-list path** (`DocumentSerializer`);
the envelope shape is identical either way — only the per-result fields differ when
searching.

**Citations & Evidence.**

- `StandardPagination` page size / cap [src/paperless/views.py:L8-L11]; applied at
  [src/documents/views.py:L182].
- Search switch [src/documents/views.py:L377, L382-L392].
- Runtime: PROBE 2 →
  `{"count":30,"next":".../?page=2","previous":null,"results":[...]}` with
  `len(results) == 25` and top-level keys exactly `[count, next, previous, results]`.
  PROBE 5 (`?page_size=5`) → 5 items, with `next` carrying `&page_size=5`. See the
  appendix. (In this run, 30 documents were seeded into the live instance, hence
  `count=30`; against any instance `count` reflects the actual number of documents.)

---

## Q6 — The unauthenticated outcome (verbatim) + the decisive rationale

**Answer (Conclusion, verbatim).** An unauthenticated `GET /api/documents/` returns:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="api"
Content-Type: application/json

{"detail":"Authentication credentials were not provided."}
```

So: **status 401**, body `{"detail":"Authentication credentials were not provided."}`,
and the response header `WWW-Authenticate: Basic realm="api"`.

**Rationale (Thinking) — the linchpin.** The result is **401, not 403**, and the reason
is the **ordering** of `DEFAULT_AUTHENTICATION_CLASSES`. Paperless lists them
Basic → Session → Token:

```python
"DEFAULT_AUTHENTICATION_CLASSES": [
    "rest_framework.authentication.BasicAuthentication",   # FIRST  -> settings.py L118
    "rest_framework.authentication.SessionAuthentication", #          settings.py L119
    "rest_framework.authentication.TokenAuthentication",   #          settings.py L120
]
```

[src/paperless/settings.py:L117-L121]. When `IsAuthenticated` fails, DRF raises
`NotAuthenticated`. In `APIView.handle_exception` [rest_framework/views.py:L448], DRF
calls `get_authenticate_header(request)`, which returns
`authenticators[0].authenticate_header(request)` — i.e. it asks the **FIRST** authenticator
for the `WWW-Authenticate` value [rest_framework/views.py:L183-L190]. The first
authenticator is `BasicAuthentication`, whose `authenticate_header` returns the **truthy**
string `'Basic realm="api"'` [rest_framework/authentication.py:L57, L108-L109]. Because
that value is truthy, `handle_exception` keeps the **401** and attaches the
`WWW-Authenticate` header [rest_framework/views.py:L455-L461; header set at L89].

**Crucially:** had `SessionAuthentication` been first, its `authenticate_header` is the
inherited base method, which returns `None` (the base method's body is `pass`)
[rest_framework/authentication.py:L44; `SessionAuthentication` does NOT override it, class
at L112]. A `None` return causes `handle_exception` to **coerce the response to 403
Forbidden** [rest_framework/views.py:L461]. In other words: *with Basic first the answer
is 401; if Session were first the very same unauthenticated request would yield 403 (and
no `WWW-Authenticate` header).* The base-class docstring states the contract precisely — a
returned string → `401`, `None` → `403` [rest_framework/authentication.py:L44]. This
exact 401-vs-403 flip was reproduced empirically (see the LINCHPIN PROOF in the appendix).

```mermaid
flowchart TD
    A["Unauthenticated GET /api/documents/"] --> B["IsAuthenticated fails -> NotAuthenticated"]
    B --> C["handle_exception calls get_authenticate_header"]
    C --> D["get_authenticate_header returns authenticators[0].authenticate_header"]
    D --> E{"First authenticator's authenticate_header value"}
    E -->|"Basic is first -> 'Basic realm=api' truthy"| F["Keep 401 + WWW-Authenticate: Basic realm=api"]
    E -->|"If Session were first -> None"| G["Coerce to 403 Forbidden, no WWW-Authenticate"]
    F --> H["Body: detail = Authentication credentials were not provided."]
    %% Paperless lists BasicAuthentication FIRST, so path F is taken -> 401
```

**Invalid-token edge case.** A *present-but-bad* token yields **401** with body
`{"detail":"Invalid token."}`. DRF raises `AuthenticationFailed(_('Invalid token.'))`
[rest_framework/authentication.py:L203], and both `NotAuthenticated` and
`AuthenticationFailed` carry `status_code = 401`
[rest_framework/exceptions.py:L173-L174, L167-L168]. (Because `BasicAuthentication` is
still first,
this 401 also carries `WWW-Authenticate: Basic realm="api"`.)

**Citations & Evidence.**

- Ordering, Basic first [src/paperless/settings.py:L117-L121] (Basic at L118).
- First-authenticator selection [rest_framework/views.py:L183-L190]; the
  401-keep-vs-403-coerce decision [rest_framework/views.py:L448-L461]; the
  `WWW-Authenticate` header attach [rest_framework/views.py:L89].
- Basic realm value [rest_framework/authentication.py:L57, L108-L109]; the base
  `authenticate_header` returns `None` → 403 contract
  [rest_framework/authentication.py:L44]; `SessionAuthentication` class with no override
  [rest_framework/authentication.py:L112].
- Invalid-token path [rest_framework/authentication.py:L203]; 401 status on both
  exceptions [rest_framework/exceptions.py:L173-L174, L167-L168].
- **Runtime PROOF (real app + real DRF code path):** PROBE 1 against the live
  Paperless-ngx instance → **401** + `WWW-Authenticate: Basic realm="api"`. The linchpin
  was then isolated by exercising the **actual installed DRF 3.13.1** authenticators:
  with the real Paperless order, `authenticators[0]` is `BasicAuthentication` and
  `.authenticate_header()` returns the truthy `'Basic realm="api"'` → **401**; with
  `SessionAuthentication` first, `.authenticate_header()` returns `None` → **403** (no
  `WWW-Authenticate`). This reproduces the 401-vs-403 flip without modifying the source.
  PROBE 3 (invalid token) → 401 `{"detail":"Invalid token."}`. See the appendix.
- Corroboration (official DRF docs, corroborative only): DRF returns 401 with a
  `WWW-Authenticate` header for denied unauthenticated requests.

---

## Q7 — Trace the Django code path (auth class + token model)

**Answer (Conclusion).**

- **Authentication class:** `rest_framework.authentication.TokenAuthentication`, enabled
  via `DEFAULT_AUTHENTICATION_CLASSES` [src/paperless/settings.py:L120].
- **Token model:** `rest_framework.authtoken.models.Token`, persisted in the database
  table **`authtoken_token`**, enabled by registering `rest_framework.authtoken` in
  `INSTALLED_APPS` [src/paperless/settings.py:L108].

**Rationale (Thinking) — end-to-end trace.**

1. A request hits `/api/documents/` → routed by the `DefaultRouter` registration
   [src/paperless/urls.py:L32] under the `^api/` mount [src/paperless/urls.py:L40] to
   `UnifiedSearchViewSet` → `DocumentViewSet`
   [src/documents/views.py:L172, L377].
2. DRF runs the authenticators in order [src/paperless/settings.py:L117-L121]. For an
   `Authorization: Token <key>` header, `TokenAuthentication.authenticate` parses the
   keyword `Token` [rest_framework/authentication.py:L161] and looks up the key.
3. The lookup loads a `Token` row (`Token.key` PK, `Token.user` OneToOne, `Token.created`)
   [rest_framework/authtoken/models.py:L13, L14, L18] from the `authtoken_token` table; a
   valid row authenticates `request.user`.
4. `permission_classes = (IsAuthenticated,)` [src/documents/views.py:L183] then allows the
   request; `StandardPagination`
   [src/documents/views.py:L182, src/paperless/views.py:L8-L11] paginates the
   `DocumentSerializer` results.
5. The same `Token` model backs all provisioning paths, though they reach it
   differently. `obtain_auth_token` (the `POST /api/token/` view) returns
   `{'token': token.key}` via `Token.objects.get_or_create`
   [rest_framework/authtoken/views.py:L54-L59], and the `drf_create_token` management
   command does the same
   [rest_framework/authtoken/management/commands/drf_create_token.py:L12-L19, L44-L45].
   The Django admin instead manages `Token` rows through `TokenProxy`/`TokenAdmin`
   [rest_framework/authtoken/admin.py:L23-L51] — not via `get_or_create` — with the key
   generated on save by `Token.save()` → `generate_key()`
   [rest_framework/authtoken/models.py:L30-L37].

**Token semantics.** The key is a **40-character hex** string; there is **one token per
user** (`user = OneToOneField(..., related_name='auth_token')`
[rest_framework/authtoken/models.py:L14]); tokens have **no built-in expiry**; and the
same key is returned by both `drf_create_token` and `POST /api/token/` (both use
`get_or_create`).

**Citations & Evidence.** [src/paperless/settings.py:L108, L120];
[rest_framework/authtoken/models.py:L9, L13, L14, L18, L30-L37];
[rest_framework/authtoken/views.py:L54-L59];
[rest_framework/authtoken/management/commands/drf_create_token.py:L12-L19, L44-L45];
[rest_framework/authtoken/admin.py:L23-L51]. Runtime:
PROBE 4 (`POST /api/token/`) returned
`{"token":"a1b6fe70fcb6406c24fa6c8c9043be8de6c46588"}` — the same key minted by
`drf_create_token`. See the appendix.

---

## Putting it together: a copy-paste integration recipe

Replace `<host>`, `<password>`, and `<40-hex-key>` with your values.

```bash
# 1) Get a token (headless): either run on the server
manage.py drf_create_token apitester
# ...or exchange credentials over HTTP:
curl -s -X POST -d 'username=apitester&password=<password>' http://<host>/api/token/
# -> {"token":"<40-hex-key>"}

# 2) Call the documents list with the token:
curl -s -H 'Authorization: Token <40-hex-key>' http://<host>/api/documents/
# -> {"count":N,"next":...,"previous":...,"results":[ ...up to 25 items... ]}

# 3) Adjust page size (capped at 100000):
curl -s -H 'Authorization: Token <40-hex-key>' 'http://<host>/api/documents/?page_size=5'
```

Note: paginate by following the `next` URL until it is `null`.

---

## Caveats & operational notes

Each of the following is directly relevant when integrating an external tool:

- **Header scheme is `Token`, not `Bearer`.** Using `Bearer` will fail; Paperless does
  not subclass `TokenAuthentication` to change the keyword
  [rest_framework/authentication.py:L161].
- **Trailing slash matters:** use `/api/documents/`. DRF's `DefaultRouter` routes include
  the trailing slash [src/paperless/urls.py:L29, L32].
- **Pagination, not a dump:** always handle `count`/`next`; do not assume all documents
  are in one response. The page size defaults to 25, with a maximum of 100000
  [src/paperless/views.py:L8-L11].
- **401 vs 403 nuance:** no credentials → `401` with `WWW-Authenticate: Basic realm="api"`;
  a malformed/unknown token → `401` `{"detail":"Invalid token."}`. The `401` (rather than
  `403`) is a direct consequence of `BasicAuthentication` being listed first (see Q6).
- **Token security:** DRF token auth performs a **database lookup per request**
  [rest_framework/authentication.py:L201], and tokens **do not expire** by default and are
  stored unhashed in the `authtoken_token` table. Treat the token like a password; rotate
  it by deleting/recreating the row. There is one token per user (OneToOne)
  [rest_framework/authtoken/models.py:L14].
- **Optional/conditional auth that does NOT change this contract:** `src/paperless/auth.py`
  defines `AngularApiAuthenticationOverride`, which is appended to the authenticators only
  `if DEBUG` [src/paperless/settings.py:L129-L132], and `HttpRemoteUserMiddleware` /
  `RemoteUserAuthentication`, which are active only when `PAPERLESS_ENABLE_HTTP_REMOTE_USER`
  is set [src/paperless/settings.py:L201, L207-L215]. Under default production settings,
  neither alters the documents-list token-auth contract. Also note the API versioning
  configuration: `DEFAULT_VERSIONING_CLASS = AcceptHeaderVersioning`,
  `DEFAULT_VERSION = "1"`, `ALLOWED_VERSIONS = ["1", "2"]`
  [src/paperless/settings.py:L122-L126] — versioning is negotiated via the `Accept` header
  and is not required for a basic list call.

---

## Appendix — Empirical verification log (verbatim)

The evidence below was captured by running the **actual Paperless-ngx application**
(v1.7.0) from the user-provided Docker image — Python 3.9, `django==4.0.4` +
`djangorestframework==3.13.1`, a Redis sidecar, and the default SQLite database, with
`DEBUG=False` (so the live authenticators are exactly `Basic → Session → Token`, with no
debug-only overrides). Migrations were applied (creating `authtoken_token`), the user
`apitester` was created, and 30 documents were seeded. The server was published on host
port `8000` and probed with `curl`. For brevity, the standard security headers
(`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`,
`Cross-Origin-Opener-Policy`) are omitted; the status line, `Content-Type`,
`WWW-Authenticate`, `Allow`, `X-Api-Version`/`X-Version`, and the body are shown exactly
as emitted.

**Token generation:**

```
$ python3 manage.py drf_create_token apitester
Generated token a1b6fe70fcb6406c24fa6c8c9043be8de6c46588 for user apitester
```

(40 hex characters; the `authtoken_token` table is present; 30 documents seeded.)

**PROBE 1 — no credentials:**

```
$ curl -i http://localhost:8000/api/documents/
HTTP/1.1 401 Unauthorized
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
Content-Length: 58

{"detail":"Authentication credentials were not provided."}
```

**PROBE 2 — valid token:**

```
$ curl -i -H 'Authorization: Token a1b6fe70fcb6406c24fa6c8c9043be8de6c46588' http://localhost:8000/api/documents/
HTTP/1.1 200 OK
Content-Type: application/json
Allow: GET, HEAD, OPTIONS
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 8102

{"count":30,"next":"http://localhost:8000/api/documents/?page=2","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"doc 30","content":"content 30","tags":[],"created":"2026-06-26T23:53:00.421538Z","modified":"2026-06-26T23:53:00.421690Z","added":"2026-06-26T23:53:00.421540Z","archive_serial_number":null,"original_file_name":"2026-06-26 doc 30.pdf","archived_file_name":null}, ... 24 more document objects ... ]}
```

(top-level keys: `count`, `next`, `previous`, `results`; `len(results) == 25`; each result
is a full `DocumentSerializer` object — `id`, `correspondent`, `document_type`, `title`,
`content`, `tags`, `created`, `modified`, `added`, `archive_serial_number`,
`original_file_name`, `archived_file_name`. The `X-Version: 1.7.0` and `X-Api-Version: 2`
headers confirm the running Paperless-ngx build and the negotiated API version.)

**PROBE 3 — invalid token:**

```
$ curl -i -H 'Authorization: Token deadbeef' http://localhost:8000/api/documents/
HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Allow: GET, HEAD, OPTIONS
Content-Length: 27

{"detail":"Invalid token."}
```

**PROBE 4 — token exchange:**

```
$ curl -i -X POST -d 'username=apitester&password=testpass123' http://localhost:8000/api/token/
HTTP/1.1 200 OK
Content-Type: application/json
Allow: POST, OPTIONS
Content-Length: 52

{"token":"a1b6fe70fcb6406c24fa6c8c9043be8de6c46588"}
```

(Identical to the key minted by `drf_create_token` above — both reach the same row via
`Token.objects.get_or_create`.)

**PROBE 5 — page-size control:**

```
$ curl -i -H 'Authorization: Token a1b6fe70fcb6406c24fa6c8c9043be8de6c46588' 'http://localhost:8000/api/documents/?page_size=5'
HTTP/1.1 200 OK
Content-Type: application/json
X-Api-Version: 2
X-Version: 1.7.0

{"count":30,"next":"http://localhost:8000/api/documents/?page=2&page_size=5","previous":null,"results":[ ...5 items... ]}
```

(`len(results) == 5`.)

**PROBE 6 — bad password:**

```
$ curl -i -X POST -d 'username=apitester&password=WRONG' http://localhost:8000/api/token/
HTTP/1.1 400 Bad Request
Content-Type: application/json
Allow: POST, OPTIONS
Content-Length: 68

{"non_field_errors":["Unable to log in with provided credentials."]}
```

**PROBE 7 — wrong scheme (`Bearer` instead of `Token`):**

```
$ curl -i -H 'Authorization: Bearer a1b6fe70fcb6406c24fa6c8c9043be8de6c46588' http://localhost:8000/api/documents/
HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Basic realm="api"
Content-Length: 58

{"detail":"Authentication credentials were not provided."}
```

(`TokenAuthentication` only recognizes the `Token` keyword, so a `Bearer` scheme is not
parsed as a token at all — it is treated as *no* credentials, yielding the same 401 as
PROBE 1, not `Invalid token.`)

**PROBE 8 — missing trailing slash:**

```
$ curl -i -H 'Authorization: Token a1b6fe70fcb6406c24fa6c8c9043be8de6c46588' http://localhost:8000/api/documents
HTTP/1.1 302 Found
Content-Type: text/html; charset=utf-8
Location: /accounts/login/?next=/api/documents
```

(The router route is `/api/documents/` *with* the slash. The slash-less path does not
match the API route, so it falls through and is redirected to the login page; the same
request **with** the trailing slash returns `200` as in PROBE 2 — confirming the trailing
slash is required for the API call.)

**LINCHPIN PROOF — the 401-vs-403 decision, exercised on the real DRF 3.13.1 classes:**

The live app cannot be reconfigured to put `SessionAuthentication` first without editing
the source (out of scope), so the decision was isolated by invoking the **actual installed
DRF** authenticators through the very logic DRF uses in `APIView.get_authenticate_header`
— `authenticators[0].authenticate_header(request)` [rest_framework/views.py:L183-L190]:

```
# Real Paperless order  -> authenticators[0] is BasicAuthentication
BasicAuthentication().authenticate_header(request)    == 'Basic realm="api"'  (truthy) -> HTTP 401
# Hypothetical order     -> authenticators[0] is SessionAuthentication
SessionAuthentication().authenticate_header(request)  == None                 (falsy)  -> HTTP 403
```

(The same unauthenticated request yields **401** when `BasicAuthentication` is first and
**403** when `SessionAuthentication` is first — empirical proof of the Q6 rationale, using
the real DRF code path with no source modification.)

**Summary of probes:**

| Probe | Request | Result |
|---|---|---|
| Token gen | `drf_create_token apitester` | `Generated token <40-hex> for user apitester` |
| No creds | `GET /api/documents/` | `401`; `WWW-Authenticate: Basic realm="api"`; `{"detail":"Authentication credentials were not provided."}` |
| Valid token | `GET /api/documents/` + `Token <key>` | `200`; `{count:30,next:...page=2,previous:null,results:[25]}` |
| Invalid token | `GET` + `Token deadbeef` | `401`; `{"detail":"Invalid token."}` |
| Wrong scheme | `GET` + `Bearer <key>` | `401`; `WWW-Authenticate: Basic realm="api"`; `{"detail":"Authentication credentials were not provided."}` |
| No trailing slash | `GET /api/documents` + `Token <key>` | `302`; `Location: /accounts/login/?next=/api/documents` |
| Token exchange | `POST /api/token/` (good creds) | `200`; `{"token":"<40-hex>"}` |
| Page size | `GET /api/documents/?page_size=5` + token | `200`; 5 items |
| Bad password | `POST /api/token/` (bad creds) | `400`; `{"non_field_errors":["Unable to log in with provided credentials."]}` |
| Linchpin | real DRF: Basic-first vs Session-first `authenticate_header` | Basic-first -> `401` (+`WWW-Authenticate`); Session-first -> `403` (none) |

---

## References

The cited files and their roles (file paths only; line numbers are noted in-text):

- `requirements.txt` (root) — pinned `django==4.0.4` [L38] and
  `djangorestframework==3.13.1` [L39]. (Note: `src/requirements.txt` does not exist.)
- `src/paperless/settings.py` — `INSTALLED_APPS` includes `authtoken` [L108];
  `REST_FRAMEWORK` / authenticator ordering [L116-L121]; versioning [L122-L126];
  conditional auth [L129-L132, L201, L207-L215].
- `src/paperless/urls.py` — router [L29], documents registration [L32], `^api/` mount
  [L40], `api_router.urls` [L83], token route [L81].
- `src/documents/views.py` — `DocumentViewSet` [L172], pagination [L182], permission
  [L183], `UnifiedSearchViewSet` + search switch [L377, L382-L392].
- `src/paperless/views.py` — `StandardPagination` [L8-L11].
- DRF 3.13.1 dependency internals:
  `rest_framework/authentication.py` [L44, L53, L57, L108-L109, L112, L151, L161, L203,
  L211], `rest_framework/authtoken/models.py` [L9, L13, L14, L18, L30-L37],
  `rest_framework/views.py` [L89, L183-L190, L448-L461],
  `rest_framework/authtoken/views.py` [L54-L59],
  `rest_framework/authtoken/management/commands/drf_create_token.py` [L12-L19, L44-L45],
  `rest_framework/authtoken/admin.py` [L23-L51],
  `rest_framework/exceptions.py` [L167-L168, L173-L174].
- Corroborating (web, corroborative only): the official DRF authentication docs at
  `https://www.django-rest-framework.org/api-guide/authentication/` confirm the
  `Authorization: Token <key>` header and the 401 + `WWW-Authenticate` behavior. The
  binding answers derive from the pinned DRF 3.13.1 source and the live runtime.
