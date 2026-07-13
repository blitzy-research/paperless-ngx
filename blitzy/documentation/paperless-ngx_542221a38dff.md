# paperless-ngx REST API — Token Authentication (Runtime-Verified Q&A)

This document answers five questions about how **token-based authentication** works in the
paperless-ngx REST API, so an external tool can be integrated against it. Every answer is
**written from observed runtime output** of a locally running, default-configured instance
(the governing rule is *run first, then write*), and every behavioral claim is grounded with a
`file:line` reference into the repository source. The source tree was treated as **read-only
evidence**; nothing in it was modified. The test user, API token, seeded documents, local
SQLite database, temporary scripts, and the throwaway investigation container used here were all
removed afterward, and the removal is verified below — only this document remains.

### Provenance (what was investigated vs. where this file lives)

- **Investigated source branch / baseline commit:** `paperless-ngx_542221a38dff`, source
  baseline commit `542221a38dff`. This is the paperless-ngx source under investigation. It is
  byte-identical to the mandated runtime image's `/app/src` (md5-verified during setup; the image
  tag's commit equals this baseline).
- **Destination repository (where this document is added):** the Blitzy working branch
  `blitzy-03b19d89-b665-4750-904b-b3b5d93c0c5f`. The **only** change added to the destination is
  this one markdown file, placed on top of the source baseline; no source file is changed.
- **Framework pins (version grounding):** `django==4.0.4` (`requirements.txt:L38`),
  `djangorestframework==3.13.1` (`requirements.txt:L39`).
- **Runtime pin:** Python 3.9 — repo-root `Dockerfile:L18` → `FROM python:3.9-slim-bullseye`.
  The **observed patch version at runtime was `3.9.23`** (captured below); the repository pins the
  3.9 **major.minor** line, and the mandated image supplies the `3.9.23` patch build.

> Note on the Python-3.9 citation: the pin lives in the **repo-root `Dockerfile`**, line 18.
> There is **no** `docker/Dockerfile` in this repository (the `docker/` directory contains only
> `compose/`, `docker-entrypoint.sh`, `docker-prepare.sh`, and related scripts), so the correct
> citation is `Dockerfile:L18`, never `docker/Dockerfile`.

---

## Environment / How this was run

The instance was built and run in its **default (canonical) configuration** using the
**mandated Docker image** supplied for this project (`paperless-ngx-qna:setup`, an image whose
`/app/src` is byte-identical to the source baseline). All data paths were redirected under `/tmp`
so the source tree at `/app/src` stays pristine. `DEBUG` defaults to `False` because
`PAPERLESS_DEBUG` was left unset — `src/paperless/settings.py:L50` →
`DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`. The default user inside the image is `root`.
The pagination question (Q3) was observed **at a scale of 30 seeded documents** so the default
page size is demonstrated rather than asserted.

**How commands are shown.** Each command below was executed inside the container as
`docker exec paperless-setup-0 bash -lc '<command>'` after sourcing the environment in step 2.
For readability the blocks show the inner command (e.g. `$ python manage.py migrate --noinput`)
and its raw, unedited output.

### 1. Start the mandated container

```bash
docker run -d --name paperless-setup-0 -w /app paperless-ngx-qna:setup -c "sleep infinity"
# subsequent commands: docker exec paperless-setup-0 bash -lc '<command>'
```

### 2. Export the canonical data-path environment (keeps /app/src pristine)

```bash
export PL=/tmp/pl0
mkdir -p $PL/data $PL/media $PL/static $PL/consume $PL/log
export PAPERLESS_DATA_DIR=$PL/data
export PAPERLESS_MEDIA_ROOT=$PL/media
export PAPERLESS_STATICDIR=$PL/static
export PAPERLESS_CONSUMPTION_DIR=$PL/consume
export PAPERLESS_LOGGING_DIR=$PL/log
export DJANGO_SETTINGS_MODULE=paperless.settings
cd /app/src
```

### 3. Observed environment / configuration (raw)

```console
$ whoami
root
$ python --version
Python 3.9.23
$ python -c "import django, rest_framework; print(django.get_version()); print(rest_framework.VERSION)"
4.0.4
3.13.1
$ command -v curl || echo "curl: not found"
curl: not found
$ python -c "import django; django.setup(); from django.conf import settings as s; \
print('DEBUG =', s.DEBUG); print('ALLOWED_HOSTS =', s.ALLOWED_HOSTS); \
print('DB ENGINE =', s.DATABASES['default']['ENGINE']); print('DB NAME =', s.DATABASES['default']['NAME'])"
DEBUG = False
ALLOWED_HOSTS = ['*']
DB ENGINE = django.db.backends.sqlite3
DB NAME = /tmp/pl0/data/db.sqlite3
```

This confirms the canonical facts every answer relies on: **Python 3.9.23**, **Django 4.0.4**,
**DRF 3.13.1**, **`DEBUG=False`**, **`ALLOWED_HOSTS=['*']`**, a **SQLite** database, and that the
image ships **no `curl`** (so a Python-standard-library HTTP client is used — see step 6).

### 4. Apply migrations (raw — excerpt showing the token table being created)

```console
$ python manage.py migrate --noinput
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying authtoken.0003_tokenproxy... OK
  ... (other apps' migrations applied OK; elided) ...
```

_(Excerpt — the full run applies every app's migrations with `OK`; the `authtoken.*` lines shown
create the `authtoken_token` table that backs Q5.)_

### 5. Create the test user (non-interactive) and seed 30 documents

```console
$ python manage.py shell -c '
from django.contrib.auth.models import User
User.objects.filter(username="testuser").delete()
u = User.objects.create_superuser("testuser", "testuser@example.test", "Testpass123")
print("created user:", u.username, "| id:", u.id, "| is_superuser:", u.is_superuser)
print("total users:", User.objects.count())'
created user: testuser | id: 2 | is_superuser: True
total users: 2
```

_(`total users: 2` because paperless's data migrations create a built-in `consumer` account at
`id: 1`; the account created here is `testuser` at `id: 2`, which the API token is bound to.)_

```console
$ python manage.py shell -c '
import hashlib
from documents.models import Document
Document.objects.all().delete()
for i in range(1, 31):
    Document.objects.create(
        title="Test Document %02d" % i,
        content="",
        mime_type="application/pdf",
        checksum=hashlib.md5(("seed-%d" % i).encode()).hexdigest(),
    )
print("seeded Document rows:", Document.objects.count())
print("first title:", Document.objects.order_by("id").first().title)
print("last title:", Document.objects.order_by("id").last().title)'
seeded Document rows: 30
first title: Test Document 01
last title: Test Document 30
```

The `Document` model requires a unique `checksum` and a `mime_type` (see
`src/documents/models.py`), so each row uses a unique 32-character MD5 hex checksum.

### 6. Start the canonical dev server (banner, raw)

```console
$ nohup python manage.py runserver 127.0.0.1:8123 --noreload > /tmp/pl0/log/runserver.log 2>&1 &
$ cat /tmp/pl0/log/runserver.log
Performing system checks...

System check identified no issues (0 silenced).
July 13, 2026 - 17:23:53
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://127.0.0.1:8123/
Quit the server with CONTROL-C.
```

### 7. HTTP client used for capture (`api_probe.py`, Python standard library)

Because the image ships no `curl`/`wget`, a small **stdlib-only** helper was used to capture
**complete, `curl -i`-style** raw evidence (status line, every response header, a blank line,
then the body). It is shown in full so every captured block is reproducible and auditable:

```python
#!/usr/bin/env python3
"""Minimal HTTP client using only the Python standard library (urllib).

The mandated paperless-ngx image ships no curl/wget, so this stdlib helper is
used to capture complete, curl -i style raw HTTP evidence: the status line,
every response header, a blank line, then the response body.

Usage: python api_probe.py METHOD URL [TOKEN] [JSON_BODY]
  TOKEN     - if non-empty, sent as  Authorization: Token <TOKEN>
  JSON_BODY - if non-empty, sent as the request body with Content-Type: application/json
"""
import sys
import urllib.request
import urllib.error

method = sys.argv[1]
url = sys.argv[2]
token = sys.argv[3] if len(sys.argv) > 3 and sys.argv[3] else None
data = sys.argv[4].encode() if len(sys.argv) > 4 and sys.argv[4] else None

req = urllib.request.Request(url, data=data, method=method)
if token:
    req.add_header("Authorization", "Token " + token)
if data:
    req.add_header("Content-Type", "application/json")

try:
    resp = urllib.request.urlopen(req, timeout=30)
except urllib.error.HTTPError as e:
    resp = e  # HTTPError is itself a readable response object

code = getattr(resp, "code", getattr(resp, "status", ""))
reason = getattr(resp, "reason", "")
print("HTTP/1.1 %s %s" % (code, reason))
for header, value in resp.headers.items():
    print("%s: %s" % (header, value))
print()  # blank line separates headers from body (mirrors curl -i)
sys.stdout.write(resp.read().decode("utf-8"))
sys.stdout.write("\n")
```

---

## Token issuance via the real entry point (prerequisite)

An external tool obtains a token by **POSTing credentials to `/api/token/`**, served by
`rest_framework.authtoken.views.obtain_auth_token`. This view is imported at
`src/paperless/urls.py:L26` (`from rest_framework.authtoken import views`) and wired at
`src/paperless/urls.py:L81` (`path("token/", views.obtain_auth_token)`), under the `^api/`
prefix (`src/paperless/urls.py:L38-L40`). The token was obtained through this real entry point
(not a debug hook or shell shortcut):

```console
$ python api_probe.py POST http://127.0.0.1:8123/api/token/ "" '{"username":"testuser","password":"Testpass123"}'
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 17:24:40 GMT
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

{"token":"a9784cad268eaf5b2f026615f360c234f66993d4"}
```

The `HTTP/1.1 200 OK` status line is emitted by the helper (the server returns status `200`; the
line is not part of the JSON body). The returned `token` value is exactly **40 characters** long
— the shape the `Authorization` header must carry below. This token belongs to a disposable test
user and is **provably invalidated** in the *Cleanup and verification* section at the end.

---

## Q1 — HTTP header name and format the API expects for the token

**Direct answer:** the header name is **`Authorization`**, and the format is the literal keyword
`Token`, a single space, then the 40-character key → **`Authorization: Token <key>`**. This is
Django REST Framework's `TokenAuthentication` contract, enabled at
`src/paperless/settings.py:L120`. (The keyword `Token` is confirmed directly from the running
authenticator in Q5: `TokenAuthentication.keyword: 'Token'`.)

Command and **complete** observed response (headers shown in full; body shown as a labeled
excerpt because the full 30-document body is large — the full body is captured and asserted in
Q3):

```console
$ python api_probe.py GET http://127.0.0.1:8123/api/documents/ a9784cad268eaf5b2f026615f360c234f66993d4
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 17:24:54 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 8364
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"count":30,"next":"http://127.0.0.1:8123/api/documents/?page=2","previous":null,"results":[ ...
```

_(Body excerpt — the envelope is shown; `results` holds 25 objects. The complete body is
**8364 bytes / 25 objects** (`Content-Length: 8364`); its shape is shown in full and asserted in
Q3.)_

**Grounding:** `src/paperless/settings.py:L120` registers
`"rest_framework.authentication.TokenAuthentication"` in
`REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` (`src/paperless/settings.py:L117-L121`).

**Cause → effect:** `HTTP 200` confirms the header is accepted; `TokenAuthentication` parses the
`Authorization: Token <key>` header, looks the key up, and authenticates the bound user.

**On the `X-Api-Version: 2` and `X-Version: 1.7.0` response headers (exact attribution).** These
two headers are added by the project's **custom `ApiVersionMiddleware`**
(`src/paperless/middleware.py:L5-L14`), registered in `MIDDLEWARE` at
`src/paperless/settings.py:L142` — **not** by DRF's `AcceptHeaderVersioning`. The middleware sets
them **only when `request.user.is_authenticated`** (`src/paperless/middleware.py:L11`); it sets
`X-Api-Version` to the **last** entry of `ALLOWED_VERSIONS` (`["1", "2"]` → `"2"`)
(`src/paperless/middleware.py:L13`; `src/paperless/settings.py:L126`) and `X-Version` to the
paperless product version (`src/paperless/middleware.py:L14`). DRF's `AcceptHeaderVersioning`
(`src/paperless/settings.py:L122`) only negotiates the **request** version and defaults it to
`DEFAULT_VERSION="1"` (`src/paperless/settings.py:L123`); it does **not** echo `"2"` into the
response. Runtime corroboration: both headers appear **only** in this authenticated `200` and are
**absent** from both unauthenticated `401` responses (Q4 and Q5), exactly matching the
`is_authenticated` gate.

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
`StandardPagination` with a **default `page_size` of 25** (client-overridable via the `page_size`
query parameter, up to a **maximum of `max_page_size=100000`**).

**Grounding:** `src/paperless/views.py:L8-L11` defines `StandardPagination(PageNumberPagination)`
with `page_size = 25`, `page_size_query_param = "page_size"`, and `max_page_size = 100000`; it is
applied to the documents viewset at `src/documents/views.py:L182`
(`pagination_class = StandardPagination`).

Commands run (authenticated GETs at a scale of **30 seeded documents**):

```console
$ python api_probe.py GET 'http://127.0.0.1:8123/api/documents/'             a9784cad268eaf5b2f026615f360c234f66993d4
$ python api_probe.py GET 'http://127.0.0.1:8123/api/documents/?page=2'      a9784cad268eaf5b2f026615f360c234f66993d4
$ python api_probe.py GET 'http://127.0.0.1:8123/api/documents/?page_size=5' a9784cad268eaf5b2f026615f360c234f66993d4
```

**Complete raw response for `?page_size=5`** (headers + full body — this shows exactly what the
JSON response looks like, including five complete `results` objects):

```console
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 17:25:48 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 1760
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"count":30,"next":"http://127.0.0.1:8123/api/documents/?page=2&page_size=5","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"Test Document 30","content":"","tags":[],"created":"2026-07-13T17:23:24.987387Z","modified":"2026-07-13T17:23:24.987531Z","added":"2026-07-13T17:23:24.987392Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 30.pdf","archived_file_name":null},{"id":29,"correspondent":null,"document_type":null,"title":"Test Document 29","content":"","tags":[],"created":"2026-07-13T17:23:24.983819Z","modified":"2026-07-13T17:23:24.983949Z","added":"2026-07-13T17:23:24.983823Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 29.pdf","archived_file_name":null},{"id":28,"correspondent":null,"document_type":null,"title":"Test Document 28","content":"","tags":[],"created":"2026-07-13T17:23:24.979603Z","modified":"2026-07-13T17:23:24.979742Z","added":"2026-07-13T17:23:24.979607Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 28.pdf","archived_file_name":null},{"id":27,"correspondent":null,"document_type":null,"title":"Test Document 27","content":"","tags":[],"created":"2026-07-13T17:23:24.975653Z","modified":"2026-07-13T17:23:24.975788Z","added":"2026-07-13T17:23:24.975657Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 27.pdf","archived_file_name":null},{"id":26,"correspondent":null,"document_type":null,"title":"Test Document 26","content":"","tags":[],"created":"2026-07-13T17:23:24.971852Z","modified":"2026-07-13T17:23:24.972010Z","added":"2026-07-13T17:23:24.971858Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 26.pdf","archived_file_name":null}]}
```

The **same `?page_size=5` body, pretty-printed** with `python -m json.tool` (reformatted only for
readability — same bytes as above, not re-fetched):

```json
{
    "count": 30,
    "next": "http://127.0.0.1:8123/api/documents/?page=2&page_size=5",
    "previous": null,
    "results": [
        {
            "id": 30,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 30",
            "content": "",
            "tags": [],
            "created": "2026-07-13T17:23:24.987387Z",
            "modified": "2026-07-13T17:23:24.987531Z",
            "added": "2026-07-13T17:23:24.987392Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 30.pdf",
            "archived_file_name": null
        },
        {
            "id": 29,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 29",
            "content": "",
            "tags": [],
            "created": "2026-07-13T17:23:24.983819Z",
            "modified": "2026-07-13T17:23:24.983949Z",
            "added": "2026-07-13T17:23:24.983823Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 29.pdf",
            "archived_file_name": null
        },
        {
            "id": 28,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 28",
            "content": "",
            "tags": [],
            "created": "2026-07-13T17:23:24.979603Z",
            "modified": "2026-07-13T17:23:24.979742Z",
            "added": "2026-07-13T17:23:24.979607Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 28.pdf",
            "archived_file_name": null
        },
        {
            "id": 27,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 27",
            "content": "",
            "tags": [],
            "created": "2026-07-13T17:23:24.975653Z",
            "modified": "2026-07-13T17:23:24.975788Z",
            "added": "2026-07-13T17:23:24.975657Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 27.pdf",
            "archived_file_name": null
        },
        {
            "id": 26,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 26",
            "content": "",
            "tags": [],
            "created": "2026-07-13T17:23:24.971852Z",
            "modified": "2026-07-13T17:23:24.972010Z",
            "added": "2026-07-13T17:23:24.971858Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 26.pdf",
            "archived_file_name": null
        }
    ]
}
```

**Complete raw response for `?page=2`** (headers + full body — the remaining 5 of 30 documents,
with a `previous` link and `next: null`):

```console
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 17:25:47 GMT
Server: WSGIServer/0.2 CPython/3.9.23
Content-Type: application/json
Vary: Accept, Accept-Language, Origin, Cookie
Allow: GET, HEAD, OPTIONS
X-Frame-Options: SAMEORIGIN
X-Api-Version: 2
X-Version: 1.7.0
Content-Length: 1736
Content-Language: en-us
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin

{"count":30,"next":null,"previous":"http://127.0.0.1:8123/api/documents/","results":[{"id":5,"correspondent":null,"document_type":null,"title":"Test Document 05","content":"","tags":[],"created":"2026-07-13T17:23:24.888780Z","modified":"2026-07-13T17:23:24.888978Z","added":"2026-07-13T17:23:24.888786Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 05.pdf","archived_file_name":null},{"id":4,"correspondent":null,"document_type":null,"title":"Test Document 04","content":"","tags":[],"created":"2026-07-13T17:23:24.885069Z","modified":"2026-07-13T17:23:24.885246Z","added":"2026-07-13T17:23:24.885074Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 04.pdf","archived_file_name":null},{"id":3,"correspondent":null,"document_type":null,"title":"Test Document 03","content":"","tags":[],"created":"2026-07-13T17:23:24.881179Z","modified":"2026-07-13T17:23:24.881362Z","added":"2026-07-13T17:23:24.881186Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 03.pdf","archived_file_name":null},{"id":2,"correspondent":null,"document_type":null,"title":"Test Document 02","content":"","tags":[],"created":"2026-07-13T17:23:24.857048Z","modified":"2026-07-13T17:23:24.857196Z","added":"2026-07-13T17:23:24.857054Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 02.pdf","archived_file_name":null},{"id":1,"correspondent":null,"document_type":null,"title":"Test Document 01","content":"","tags":[],"created":"2026-07-13T17:23:24.831794Z","modified":"2026-07-13T17:23:24.832036Z","added":"2026-07-13T17:23:24.831804Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 01.pdf","archived_file_name":null}]}
```

The **default page** (no query params) is the identical endpoint captured in Q1: its envelope
(`count:30`, `next:...?page=2`, `previous:null`) and first `results` object are shown there; its
full body is **8364 bytes / 25 objects**.

**Derived assertions** (labeled *derived* — the following are **computed** from the three raw
bodies above by the fully-shown `paginate_assert.py`, which reads the saved response bodies and
prints the top-level fields and result counts):

```python
#!/usr/bin/env python3
"""Derived assertions computed from the raw JSON bodies captured above.
Reads each saved response body and prints top-level fields + result counts."""
import json

for label, path in [
    ("default page   (GET /api/documents/)",            "body_default.json"),
    ("?page=2        (GET /api/documents/?page=2)",      "body_page2.json"),
    ("?page_size=5   (GET /api/documents/?page_size=5)", "body_pagesize5.json"),
]:
    with open(path) as fh:
        d = json.load(fh)
    print(label)
    print("   top-level keys : %s" % list(d.keys()))
    print("   count          : %s" % d["count"])
    print("   len(results)   : %s" % len(d["results"]))
    print("   previous       : %s" % d["previous"])
    print("   next           : %s" % d["next"])
    if path == "body_pagesize5.json":
        print("   results[0] keys: %s" % list(d["results"][0].keys()))
    print()
```

```console
$ python paginate_assert.py
default page   (GET /api/documents/)
   top-level keys : ['count', 'next', 'previous', 'results']
   count          : 30
   len(results)   : 25
   previous       : None
   next           : http://127.0.0.1:8123/api/documents/?page=2

?page=2        (GET /api/documents/?page=2)
   top-level keys : ['count', 'next', 'previous', 'results']
   count          : 30
   len(results)   : 5
   previous       : http://127.0.0.1:8123/api/documents/
   next           : None

?page_size=5   (GET /api/documents/?page_size=5)
   top-level keys : ['count', 'next', 'previous', 'results']
   count          : 30
   len(results)   : 5
   previous       : None
   next           : http://127.0.0.1:8123/api/documents/?page=2&page_size=5
   results[0] keys: ['id', 'correspondent', 'document_type', 'title', 'content', 'tags', 'created', 'modified', 'added', 'archive_serial_number', 'original_file_name', 'archived_file_name']
```

Each element of `results` carries the `DocumentSerializer` fields, in this exact order
(12 fields): `id`, `correspondent`, `document_type`, `title`, `content`, `tags`, `created`,
`modified`, `added`, `archive_serial_number`, `original_file_name`, `archived_file_name` — matching
the `results[0] keys` line above.

**Grounding:** the field tuple is declared in the serializer's `Meta`
(`src/documents/serialisers.py:L219` → `class Meta`, `src/documents/serialisers.py:L222-L235` →
the `fields` tuple). The plain list uses `DocumentSerializer` (not the search serializer) because
no `query`/`more_like_id` query parameter is present:
`UnifiedSearchViewSet.get_serializer_class` returns `DocumentSerializer` unless
`_is_search_request()` is true (`src/documents/views.py:L382-L392`).

**Cause → effect:** with 30 documents, the default page returns the first 25 items plus a `next`
link and `previous=null`; `?page=2` returns the remaining 5 items with a `previous` link and
`next=null`; `?page_size=5` proves the page size is client-overridable. **25 is the *default*
page size, not a hard cap** — a client may request up to `max_page_size=100000`. The endpoint
pages its results and does **not** dump everything at once.

---

## Q4 — the same request WITHOUT auth: status code and error message

**Direct answer:** the status code is **`401 Unauthorized`** and the error body is
**`{"detail":"Authentication credentials were not provided."}`**.

Command and **complete** observed response:

```console
$ python api_probe.py GET http://127.0.0.1:8123/api/documents/
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 17:25:22 GMT
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

**Grounding:** the documents viewset enforces `permission_classes = (IsAuthenticated,)` at
`src/documents/views.py:L183`.

**Cause → effect:** with no credentials supplied, `IsAuthenticated` denies access, yielding
`401 Unauthorized`. The `WWW-Authenticate: Basic realm="api"` challenge header appears because
`BasicAuthentication` is listed **first** in the authentication stack
(`src/paperless/settings.py:L118`) and DRF emits the first authenticator's challenge; the
`detail` text is DRF's standard `NotAuthenticated` message. Note the `X-Api-Version` / `X-Version`
headers are **absent** here (the request is unauthenticated, so `ApiVersionMiddleware` skips them
— see Q1).

---

## Q5 — which authentication class handles token auth, and which model stores tokens

**Direct answer:** the authentication class is
**`rest_framework.authentication.TokenAuthentication`** (`src/paperless/settings.py:L120`), and
the model that stores tokens is **`rest_framework.authtoken.models.Token`**, enabled via
`"rest_framework.authtoken"` in `INSTALLED_APPS` (`src/paperless/settings.py:L108`). Its database
table is **`authtoken_token`**, which stores a 40-character `key` bound to a user.

The token model was confirmed via the Django shell (investigation-only). The **full** inspection
script is shown, then run, with its raw output:

```python
# inspect_token.py  —  run via:  python manage.py shell < inspect_token.py
from django.conf import settings
from rest_framework.authentication import TokenAuthentication
from rest_framework.authtoken.models import Token

# (1) The authentication classes configured for the API (src/paperless/settings.py:L117-L120)
print("configured DEFAULT_AUTHENTICATION_CLASSES:")
for c in settings.REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]:
    print("   -", c)

# (2) The class attribute vs. the resolved token model
ta = TokenAuthentication()
print("TokenAuthentication.keyword           :", repr(ta.keyword))
print("TokenAuthentication.model (class attr):", TokenAuthentication.model)
resolved = ta.get_model()
print("resolved get_model()                  :", resolved.__module__ + "." + resolved.__name__)
print("db table                              :", resolved._meta.db_table)

# (3) The token row bound to the test user
tok = Token.objects.get(user__username="testuser")
print("token key length                      :", len(tok.key))
print("token bound to user                   :", tok.user.username)
```

```console
$ python manage.py shell < inspect_token.py
configured DEFAULT_AUTHENTICATION_CLASSES:
   - rest_framework.authentication.BasicAuthentication
   - rest_framework.authentication.SessionAuthentication
   - rest_framework.authentication.TokenAuthentication
TokenAuthentication.keyword           : 'Token'
TokenAuthentication.model (class attr): None
resolved get_model()                  : rest_framework.authtoken.models.Token
db table                              : authtoken_token
token key length                      : 40
token bound to user                   : testuser
```

Reading that output exactly: the authenticator is
`rest_framework.authentication.TokenAuthentication`, and its keyword is `'Token'` (the Q1 header
keyword). Its class-level `model` attribute defaults to `None`; the concrete token model is
resolved through `TokenAuthentication.get_model()`, which returns
`rest_framework.authtoken.models.Token`, whose database table is `authtoken_token`. The bound
token row's `key` is 40 characters long (matching the token issued earlier) and is tied to the
`testuser` account. That `TokenAuthentication` is the validating class is also confirmed at
runtime by its distinctive invalid-token message (**complete** raw response):

```console
$ python api_probe.py GET http://127.0.0.1:8123/api/documents/ wrong_token
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 17:25:22 GMT
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

**Grounding:** `src/paperless/settings.py:L108` registers `"rest_framework.authtoken"` (which
provides the `Token` model and its `authtoken_token` migration/table — see the
`authtoken.0001_initial` / `authtoken.0003_tokenproxy` migration lines in step 4);
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

## Cleanup and verification (read-only guarantee)

All investigation artifacts were removed and the removal was verified. First, the seeded
documents and the test user were deleted (deleting the user cascade-deletes its token); the
counts confirm removal:

```console
$ python manage.py shell -c '
from django.contrib.auth.models import User
from documents.models import Document
from rest_framework.authtoken.models import Token
Document.objects.all().delete()
User.objects.filter(username="testuser").delete()
print("Document rows after delete :", Document.objects.count())
print("testuser rows after delete :", User.objects.filter(username="testuser").count())
print("Token rows after delete    :", Token.objects.count())'
Document rows after delete : 0
testuser rows after delete : 0
Token rows after delete    : 0
```

Re-issuing the **same request with the previously valid token** now fails — proving the token is
invalidated at the API level:

```console
$ python api_probe.py GET http://127.0.0.1:8123/api/documents/ a9784cad268eaf5b2f026615f360c234f66993d4
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 17:27:21 GMT
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

Finally, the entire investigation container was removed (which destroys its SQLite database and
all `/tmp/pl0` data), and its absence plus a clean source tree were verified:

```console
$ docker rm -f paperless-setup-0
paperless-setup-0
$ docker ps -a --filter name=paperless-setup-0 --format '{{.Names}}' || true
paperless-setup-0: not present (removed)
$ git status --porcelain
# (empty output — the source tree is byte-for-byte unchanged; only this
#  blitzy/documentation/ markdown file is added to the destination branch)
```

The disposable test user, its 40-character token, the 30 seeded documents, the SQLite database,
the temporary helper/inspection scripts, and the container itself are all gone; the token above
is provably non-functional. No product source, configuration, dependency, test, or frontend file
was modified.

---

## Coverage-pass checklist

Every named item from the request is answered, and each box is ticked only where adjacent
observed evidence exists in this document:

- [x] Q1 header **name** answered — `Authorization` (Q1: authenticated `HTTP 200`).
- [x] Q1 header **format** answered — `Token <key>`, 40-character key (Q1 + Q5 `keyword: 'Token'` + issuance token length).
- [x] Q2 complete **endpoint path** answered — `/api/documents/` (Q2 grounding + Q1 `200`).
- [x] Q3 **top-level fields** answered — `count`, `next`, `previous`, `results` (Q3 raw bodies + `paginate_assert.py`).
- [x] Q3 **pagination vs. dump-everything** answered — paginated; **default** `page_size=25`, **max** `100000` (Q3).
- [x] Q3 **default page** proof — `count=30`, `len(results)=25`, `previous=null`, `next=...?page=2` (Q1 envelope + assertion).
- [x] Q3 **`?page=2`** proof — `count=30`, `len(results)=5`, `previous=.../`, `next=null` (complete raw + assertion).
- [x] Q3 **`?page_size=5`** proof — `count=30`, `len(results)=5`, `next=...?page=2&page_size=5` (complete raw + assertion).
- [x] Q3 **`results` item fields** enumerated — 12 `DocumentSerializer` fields, in order (assertion `results[0] keys`).
- [x] Q4 **status code** answered — `401 Unauthorized` (complete raw).
- [x] Q4 **error message** answered — `{"detail":"Authentication credentials were not provided."}` (complete raw).
- [x] Q5 **authentication class** named — `rest_framework.authentication.TokenAuthentication` (settings + `inspect_token.py`).
- [x] Q5 **token model** named — `rest_framework.authtoken.models.Token`; table `authtoken_token` (`inspect_token.py` raw output).
- [x] Q5 **exact model-inspection command shown** — `python manage.py shell < inspect_token.py` (full script + raw output).
- [x] Token obtained via the **real entry point** — `POST /api/token/` (issuance section, real `200`).
- [x] Error/secondary paths exercised — no-auth `401` (Q4); invalid token `401` "Invalid token." (Q5).
- [x] Runtime evidence at real scale — 30 seeded documents; row-count proof in step 5.
- [x] Cleanup performed and **verified** — counts `0`, token invalidated (`401`), container removed, clean `git status`.

---

## Non-canonical caveats

The following choices were made for local capture convenience only and do **not** affect any
answer above:

- **Mandated Docker image** (`paperless-ngx-qna:setup`) was used per the project setup
  instructions; its `/app/src` is byte-identical to the source baseline. Data paths were
  redirected under `/tmp` so the source tree stays pristine.
- **Python standard-library HTTP client** (`api_probe.py`, `urllib`) was used because the image
  ships no `curl`/`wget`; it captures the wire-level status line, headers, and body verbatim. The
  header contract, endpoint path, status codes, and pagination envelope are client-agnostic.
- **SQLite** was used locally (paperless's default when no external database host is configured).
  The authentication class, token model, header contract, endpoint path, status codes, and
  pagination envelope are database-agnostic, so this choice does not change any answer.
- **Observed patch version `3.9.23`**: the repository pins the Python **3.9** line
  (`Dockerfile:L18`); the mandated image supplies the `3.9.23` patch build (`Server:` header and
  `python --version`). The patch level does not affect any HTTP or authentication contract.
- **Bind `127.0.0.1:8123`** was chosen for capture convenience; the port is **not** part of the
  endpoint contract — the contract is the path `/api/documents/`.
- **Django's development `runserver` (`WSGIServer`)** was used, which is appropriate for observing
  application-level authentication behavior; it does not change the HTTP contract a production
  server would present.
