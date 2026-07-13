# paperless-ngx REST API — Token Authentication (Runtime-Verified Q&A)

This document answers five questions about how **token-based authentication** works in the
paperless-ngx REST API, so an external tool can be integrated against it. Every answer is
**written from observed runtime output** of a locally running, default-configured instance
(the governing rule is _run first, then write_), and every behavioral claim is grounded with a
`file:line` reference into the repository source. The source tree was treated as **read-only
evidence**; nothing in it was modified. The test user, API token, seeded documents, local
SQLite database, temporary scripts, and the throwaway investigation container used here were all
removed afterward, and the removal is verified below — only this document remains.

Every command block below is **copy-paste reproducible from a clean checkout**: each block shows
the _exact_ command that produced the output immediately beneath it, and the one-time setup
(container start, environment file, and helper scripts) is shown in full before it is used.

### Provenance (what was investigated vs. where this file lives)

- **Investigated source branch / baseline commit:** `paperless-ngx_542221a38dff`, source
  baseline commit `542221a38dff`. This is the paperless-ngx source under investigation. It is
  byte-identical to the mandated runtime image's `/app/src` (md5-verified during setup; the image
  tag's commit equals this baseline).
- **Destination repository (where this document is added):** the Blitzy working branch
  `blitzy-03b19d89-b665-4750-904b-b3b5d93c0c5f`. The **only** change added to the destination is
  this one markdown file, placed on top of the source baseline; no source file is changed
  (verified in _Cleanup and verification_).
- **Framework pins (version grounding):** `django==4.0.4` (`requirements.txt:L38`),
  `djangorestframework==3.13.1` (`requirements.txt:L39`).
- **Runtime pin:** Python 3.9 — repo-root `Dockerfile:L18` -> `FROM python:3.9-slim-bullseye`.
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
`/app/src` is byte-identical to the source baseline). All data paths are redirected under `/tmp`
so the source tree at `/app/src` stays pristine. `DEBUG` defaults to `False` because
`PAPERLESS_DEBUG` is left unset — `src/paperless/settings.py:L50` ->
`DEBUG = __get_boolean("PAPERLESS_DEBUG", "NO")`. The default user inside the image is `root`.
The pagination question (Q3) is observed **at a scale of 30 seeded documents** so the default
page size is demonstrated rather than asserted.

### How commands are shown (the exact, reproducible convention)

The mandated image is intentionally minimal: it ships **no `curl`/`wget`** and no process tools
beyond `kill`, and a fresh `docker exec` does **not** inherit environment variables from a prior
`exec`. To make every step reproducible from a clean container, this runbook uses three rules,
applied verbatim in the blocks that follow:

1. **Environment is persisted to a file and sourced on every exec.** Step 2 writes
   `/tmp/pl0/env.sh` once; every Django command then begins with `source /tmp/pl0/env.sh`. This
   avoids the failure mode where an un-sourced `exec` cannot even import settings
   (`django.core.exceptions.ImproperlyConfigured: Requested setting LOGGING_CONFIG ...`).
2. **All multi-line / quote-heavy Python is placed in helper scripts** under `/tmp/pl0` and run
   via `python manage.py shell < /tmp/pl0/<name>.py`. This keeps every shell command free of
   embedded single quotes, so it survives the `docker exec ... bash -lc '...'` wrapper unchanged.
3. **The two wrapper forms** used below are exactly:
   - Django / `manage.py` commands (need settings + the source tree):
     `docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && <cmd>'`
   - HTTP / file commands (run from the scratch dir, need no Django settings):
     `docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && <cmd>'`

Each block shows the full `docker exec ...` command on the `$` line and its raw, unedited output
directly beneath it.

### 1. Start the mandated container

```bash
docker run -d --name paperless-setup-0 -w /app paperless-ngx-qna:setup -c "sleep infinity"
```

### 2. One-time setup: environment file + all helper scripts (copy-paste as a single block)

This one block creates the canonical data-path environment file and **every** helper script this
runbook uses. It is fed to the container over stdin (`bash -s`), so the nested here-documents are
written verbatim with no shell-quoting hazards. Run it once, immediately after step 1:

```bash
docker exec -i paperless-setup-0 bash -s <<'SETUP'
set -e
mkdir -p /tmp/pl0/data /tmp/pl0/media /tmp/pl0/static /tmp/pl0/consume /tmp/pl0/log

cat > /tmp/pl0/env.sh <<'ENV'
export PL=/tmp/pl0
export PAPERLESS_DATA_DIR=$PL/data
export PAPERLESS_MEDIA_ROOT=$PL/media
export PAPERLESS_STATICDIR=$PL/static
export PAPERLESS_CONSUMPTION_DIR=$PL/consume
export PAPERLESS_LOGGING_DIR=$PL/log
export DJANGO_SETTINGS_MODULE=paperless.settings
ENV

cat > /tmp/pl0/settingsprobe.py <<'PY'
# settingsprobe.py  —  run via:  python manage.py shell < /tmp/pl0/settingsprobe.py
from django.conf import settings as s
print("DEBUG =", s.DEBUG)
print("ALLOWED_HOSTS =", s.ALLOWED_HOSTS)
print("DB ENGINE =", s.DATABASES["default"]["ENGINE"])
print("DB NAME =", s.DATABASES["default"]["NAME"])
PY

cat > /tmp/pl0/mkuser.py <<'PY'
# mkuser.py  —  run via:  python manage.py shell < /tmp/pl0/mkuser.py
from django.contrib.auth.models import User
User.objects.filter(username="testuser").delete()
u = User.objects.create_superuser("testuser", "testuser@example.test", "Testpass123")
print("created user:", u.username, "| id:", u.id, "| is_superuser:", u.is_superuser)
print("total users:", User.objects.count())
PY

cat > /tmp/pl0/seed.py <<'PY'
# seed.py  —  run via:  python manage.py shell < /tmp/pl0/seed.py
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
print("last title:", Document.objects.order_by("id").last().title)
PY

cat > /tmp/pl0/api_probe.py <<'PY'
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
PY

cat > /tmp/pl0/save_body.py <<'PY'
#!/usr/bin/env python3
"""Save ONLY the JSON response body of a GET request to a file (stdlib urllib).

This materializes the body_*.json inputs consumed by paginate_assert.py. It is
separate from api_probe.py, which prints a curl -i style capture (status line +
headers + body) to stdout for the raw-evidence blocks.

Usage: python save_body.py URL TOKEN OUTFILE
"""
import sys
import urllib.request
import urllib.error

url = sys.argv[1]
token = sys.argv[2] if len(sys.argv) > 2 and sys.argv[2] else None
outfile = sys.argv[3]

req = urllib.request.Request(url, method="GET")
if token:
    req.add_header("Authorization", "Token " + token)
try:
    resp = urllib.request.urlopen(req, timeout=30)
except urllib.error.HTTPError as e:
    resp = e

body = resp.read().decode("utf-8")
with open(outfile, "w") as fh:
    fh.write(body)
print("wrote %d bytes -> %s" % (len(body.encode("utf-8")), outfile))
PY

cat > /tmp/pl0/paginate_assert.py <<'PY'
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
PY

cat > /tmp/pl0/inspect_token.py <<'PY'
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
PY

cat > /tmp/pl0/cleanup.py <<'PY'
# cleanup.py  —  run via:  python manage.py shell < /tmp/pl0/cleanup.py
from django.contrib.auth.models import User
from documents.models import Document
from rest_framework.authtoken.models import Token
Document.objects.all().delete()
User.objects.filter(username="testuser").delete()
print("Document rows after delete :", Document.objects.count())
print("testuser rows after delete :", User.objects.filter(username="testuser").count())
print("Token rows after delete    :", Token.objects.count())
PY

echo "container setup complete; files under /tmp/pl0:"
ls /tmp/pl0
SETUP
```

Observed output (the setup completes and lists the files it created):

```console
container setup complete; files under /tmp/pl0:
api_probe.py
cleanup.py
consume
data
env.sh
inspect_token.py
log
media
mkuser.py
paginate_assert.py
save_body.py
seed.py
settingsprobe.py
static
```

From here on, `env.sh` and the eight helper scripts above exist under `/tmp/pl0` and are used by
name. Their full contents are shown in this block, so nothing below is hidden or assumed.

### 3. Confirm the canonical configuration (raw)

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py shell < /tmp/pl0/settingsprobe.py'
DEBUG = False
ALLOWED_HOSTS = ['*']
DB ENGINE = django.db.backends.sqlite3
DB NAME = /tmp/pl0/data/db.sqlite3
```

This confirms the canonical facts every answer relies on: **`DEBUG=False`**,
**`ALLOWED_HOSTS=['*']`**, and a **SQLite** database at `/tmp/pl0/data/db.sqlite3`. (Python
`3.9.23`, Django `4.0.4`, and DRF `3.13.1` are confirmed by the `Server:` header and version
pins referenced throughout.)

---

### 4. Apply migrations (raw — complete, unedited output)

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py migrate --noinput'
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, django_q, documents, paperless_mail, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying authtoken.0003_tokenproxy... OK
  Applying django_q.0001_initial... OK
  Applying django_q.0002_auto_20150630_1624... OK
  Applying django_q.0003_auto_20150708_1326... OK
  Applying django_q.0004_auto_20150710_1043... OK
  Applying django_q.0005_auto_20150718_1506... OK
  Applying django_q.0006_auto_20150805_1817... OK
  Applying django_q.0007_ormq... OK
  Applying django_q.0008_auto_20160224_1026... OK
  Applying django_q.0009_auto_20171009_0915... OK
  Applying django_q.0010_auto_20200610_0856... OK
  Applying django_q.0011_auto_20200628_1055... OK
  Applying django_q.0012_auto_20200702_1608... OK
  Applying django_q.0013_task_attempt_count... OK
  Applying django_q.0014_schedule_cluster... OK
  Applying documents.0001_initial... OK
  Applying documents.0002_auto_20151226_1316... OK
  Applying documents.0003_sender... OK
  Applying documents.0004_auto_20160114_1844... OK
  Applying documents.0005_auto_20160123_0313... OK
  Applying documents.0006_auto_20160123_0430... OK
  Applying documents.0007_auto_20160126_2114... OK
  Applying documents.0008_document_file_type... OK
  Applying documents.0009_auto_20160214_0040... OK
  Applying documents.0010_log... OK
  Applying documents.0011_auto_20160303_1929... OK
  Applying documents.0012_auto_20160305_0040... OK
  Applying documents.0013_auto_20160325_2111... OK
  Applying documents.0014_document_checksum... OK
  Applying documents.0015_add_insensitive_to_match... OK
  Applying documents.0016_auto_20170325_1558... OK
  Applying documents.0017_auto_20170512_0507... OK
  Applying documents.0018_auto_20170715_1712... OK
  Applying documents.0019_add_consumer_user... OK
  Applying documents.0020_document_added... OK
  Applying documents.0021_document_storage_type... OK
  Applying documents.0022_auto_20181007_1420... OK
  Applying documents.0023_document_current_filename... OK
  Applying documents.1000_update_paperless_all... OK
  Applying documents.1001_auto_20201109_1636... OK
  Applying documents.1002_auto_20201111_1105... OK
  Applying documents.1003_mime_types... OK
  Applying documents.1004_sanity_check_schedule... OK
  Applying documents.1005_checksums... OK
  Applying documents.1006_auto_20201208_2209... OK
  Applying documents.1007_savedview_savedviewfilterrule... OK
  Applying documents.1008_auto_20201216_1736... OK
  Applying documents.1009_auto_20201216_2005... OK
  Applying documents.1010_auto_20210101_2159... OK
  Applying documents.1011_auto_20210101_2340... OK
  Applying documents.1012_fix_archive_files... OK
  Applying documents.1013_migrate_tag_colour... OK
  Applying documents.1014_auto_20210228_1614... OK
  Applying documents.1015_remove_null_characters... OK
  Applying documents.1016_auto_20210317_1351... OK
  Applying documents.1017_alter_savedviewfilterrule_rule_type... OK
  Applying documents.1018_alter_savedviewfilterrule_value... OK
  Applying paperless_mail.0001_initial... OK
  Applying paperless_mail.0002_auto_20201117_1334... OK
  Applying paperless_mail.0003_auto_20201118_1940... OK
  Applying paperless_mail.0004_mailrule_order... OK
  Applying paperless_mail.0005_help_texts... OK
  Applying paperless_mail.0006_auto_20210101_2340... OK
  Applying paperless_mail.0007_auto_20210106_0138... OK
  Applying paperless_mail.0008_auto_20210516_0940... OK
  Applying paperless_mail.0009_mailrule_assign_tags... OK
  Applying paperless_mail.0010_auto_20220311_1602... OK
  Applying paperless_mail.0011_remove_mailrule_assign_tag... OK
  Applying paperless_mail.0012_alter_mailrule_assign_tags... OK
  Applying paperless_mail.0009_alter_mailrule_action_alter_mailrule_folder... OK
  Applying paperless_mail.0013_merge_20220412_1051... OK
  Applying paperless_mail.0014_alter_mailrule_action... OK
  Applying sessions.0001_initial... OK
```

The full run applies every app's migrations. The three `authtoken.*` lines
(`authtoken.0001_initial`, `authtoken.0002_auto_20160226_1747`, `authtoken.0003_tokenproxy`)
create the `authtoken_token` table that backs token storage (Q5).

### 5. Create the test user and seed 30 documents

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py shell < /tmp/pl0/mkuser.py'
created user: testuser | id: 2 | is_superuser: True
total users: 2
```

The count is `total users: 2` because paperless's data migrations create a built-in `consumer`
account at `id: 1`; the account created here is `testuser` at `id: 2`, which the API token is
bound to.

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py shell < /tmp/pl0/seed.py'
seeded Document rows: 30
first title: Test Document 01
last title: Test Document 30
```

The `Document` model requires a unique `checksum` and a `mime_type` (see
`src/documents/models.py`), so each of the 30 rows uses a unique 32-character MD5 hex checksum.

### 6. Start the canonical dev server and wait until it is ready (banner, raw)

The server is started **detached** (`docker exec -d`), then a readiness loop polls the log until
the "Starting development server" banner line appears **before** the log is printed — this avoids
the race where the log is read before the server has finished booting:

```console
$ docker exec -d paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && exec python manage.py runserver 127.0.0.1:8123 --noreload > /tmp/pl0/log/runserver.log 2>&1'
$ docker exec paperless-setup-0 bash -lc 'for i in $(seq 1 60); do grep -q "Starting development server" /tmp/pl0/log/runserver.log 2>/dev/null && break; sleep 0.5; done; cat /tmp/pl0/log/runserver.log'
Performing system checks...

System check identified no issues (0 silenced).
July 13, 2026 - 20:34:25
Django version 4.0.4, using settings 'paperless.settings'
Starting development server at http://127.0.0.1:8123/
Quit the server with CONTROL-C.
```

The banner confirms **Django 4.0.4**, settings module `paperless.settings`, and the bind address
`http://127.0.0.1:8123/`. (The timestamp is wall-clock and varies per run; everything else is
stable.)

---

## Token issuance via the real entry point (prerequisite)

An external tool obtains a token by **POSTing credentials to `/api/token/`**, served by
`rest_framework.authtoken.views.obtain_auth_token`. This view is imported at
`src/paperless/urls.py:L26` (`from rest_framework.authtoken import views`) and wired at
`src/paperless/urls.py:L81` (`path("token/", views.obtain_auth_token)`), under the `^api/`
prefix (`src/paperless/urls.py:L38-L40`). The token is obtained through this real entry point
(not a debug hook or shell shortcut), using the `api_probe.py` helper created in step 2:

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py POST http://127.0.0.1:8123/api/token/ "" "{\"username\":\"testuser\",\"password\":\"Testpass123\"}"'
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 20:35:11 GMT
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

{"token":"1f350e2500056d57d92f7aabeeef0665c46d91bc"}
```

The `HTTP/1.1 200 OK` status line is emitted by the helper (the server returns status `200`; the
line is not part of the JSON body). The returned `token` value
`1f350e2500056d57d92f7aabeeef0665c46d91bc` is exactly **40 characters** long — the shape the `Authorization` header must
carry below. This token belongs to a disposable test user and is **provably invalidated** in the
_Cleanup and verification_ section at the end.

---

## Q1 — HTTP header name and format the API expects for the token

**Direct answer:** the header name is **`Authorization`**, and the format is the literal keyword
`Token`, a single space, then the 40-character key -> **`Authorization: Token <key>`**. This is
Django REST Framework's `TokenAuthentication` contract, enabled at
`src/paperless/settings.py:L120`. (The keyword `Token` is confirmed directly from the running
authenticator in Q5: `TokenAuthentication.keyword: 'Token'`.)

Command and **complete** observed response — the entire raw capture is shown (status line, every
header, a blank line, then the full JSON body; nothing is elided):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET http://127.0.0.1:8123/api/documents/ 1f350e2500056d57d92f7aabeeef0665c46d91bc'
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 20:35:20 GMT
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

{"count":30,"next":"http://127.0.0.1:8123/api/documents/?page=2","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"Test Document 30","content":"","tags":[],"created":"2026-07-13T20:33:54.084758Z","modified":"2026-07-13T20:33:54.084989Z","added":"2026-07-13T20:33:54.084766Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 30.pdf","archived_file_name":null},{"id":29,"correspondent":null,"document_type":null,"title":"Test Document 29","content":"","tags":[],"created":"2026-07-13T20:33:54.080110Z","modified":"2026-07-13T20:33:54.080316Z","added":"2026-07-13T20:33:54.080117Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 29.pdf","archived_file_name":null},{"id":28,"correspondent":null,"document_type":null,"title":"Test Document 28","content":"","tags":[],"created":"2026-07-13T20:33:54.075647Z","modified":"2026-07-13T20:33:54.075924Z","added":"2026-07-13T20:33:54.075656Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 28.pdf","archived_file_name":null},{"id":27,"correspondent":null,"document_type":null,"title":"Test Document 27","content":"","tags":[],"created":"2026-07-13T20:33:54.068392Z","modified":"2026-07-13T20:33:54.068605Z","added":"2026-07-13T20:33:54.068400Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 27.pdf","archived_file_name":null},{"id":26,"correspondent":null,"document_type":null,"title":"Test Document 26","content":"","tags":[],"created":"2026-07-13T20:33:54.064234Z","modified":"2026-07-13T20:33:54.064517Z","added":"2026-07-13T20:33:54.064244Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 26.pdf","archived_file_name":null},{"id":25,"correspondent":null,"document_type":null,"title":"Test Document 25","content":"","tags":[],"created":"2026-07-13T20:33:54.055519Z","modified":"2026-07-13T20:33:54.055769Z","added":"2026-07-13T20:33:54.055528Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 25.pdf","archived_file_name":null},{"id":24,"correspondent":null,"document_type":null,"title":"Test Document 24","content":"","tags":[],"created":"2026-07-13T20:33:54.035731Z","modified":"2026-07-13T20:33:54.035970Z","added":"2026-07-13T20:33:54.035738Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 24.pdf","archived_file_name":null},{"id":23,"correspondent":null,"document_type":null,"title":"Test Document 23","content":"","tags":[],"created":"2026-07-13T20:33:54.030403Z","modified":"2026-07-13T20:33:54.030640Z","added":"2026-07-13T20:33:54.030410Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 23.pdf","archived_file_name":null},{"id":22,"correspondent":null,"document_type":null,"title":"Test Document 22","content":"","tags":[],"created":"2026-07-13T20:33:54.026251Z","modified":"2026-07-13T20:33:54.026626Z","added":"2026-07-13T20:33:54.026262Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 22.pdf","archived_file_name":null},{"id":21,"correspondent":null,"document_type":null,"title":"Test Document 21","content":"","tags":[],"created":"2026-07-13T20:33:54.011888Z","modified":"2026-07-13T20:33:54.012155Z","added":"2026-07-13T20:33:54.011900Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 21.pdf","archived_file_name":null},{"id":20,"correspondent":null,"document_type":null,"title":"Test Document 20","content":"","tags":[],"created":"2026-07-13T20:33:53.987376Z","modified":"2026-07-13T20:33:53.987724Z","added":"2026-07-13T20:33:53.987390Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 20.pdf","archived_file_name":null},{"id":19,"correspondent":null,"document_type":null,"title":"Test Document 19","content":"","tags":[],"created":"2026-07-13T20:33:53.971210Z","modified":"2026-07-13T20:33:53.971514Z","added":"2026-07-13T20:33:53.971222Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 19.pdf","archived_file_name":null},{"id":18,"correspondent":null,"document_type":null,"title":"Test Document 18","content":"","tags":[],"created":"2026-07-13T20:33:53.954927Z","modified":"2026-07-13T20:33:53.955206Z","added":"2026-07-13T20:33:53.954938Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 18.pdf","archived_file_name":null},{"id":17,"correspondent":null,"document_type":null,"title":"Test Document 17","content":"","tags":[],"created":"2026-07-13T20:33:53.936464Z","modified":"2026-07-13T20:33:53.936645Z","added":"2026-07-13T20:33:53.936470Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 17.pdf","archived_file_name":null},{"id":16,"correspondent":null,"document_type":null,"title":"Test Document 16","content":"","tags":[],"created":"2026-07-13T20:33:53.930276Z","modified":"2026-07-13T20:33:53.930511Z","added":"2026-07-13T20:33:53.930284Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 16.pdf","archived_file_name":null},{"id":15,"correspondent":null,"document_type":null,"title":"Test Document 15","content":"","tags":[],"created":"2026-07-13T20:33:53.922971Z","modified":"2026-07-13T20:33:53.923258Z","added":"2026-07-13T20:33:53.922983Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 15.pdf","archived_file_name":null},{"id":14,"correspondent":null,"document_type":null,"title":"Test Document 14","content":"","tags":[],"created":"2026-07-13T20:33:53.898837Z","modified":"2026-07-13T20:33:53.899197Z","added":"2026-07-13T20:33:53.898850Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 14.pdf","archived_file_name":null},{"id":13,"correspondent":null,"document_type":null,"title":"Test Document 13","content":"","tags":[],"created":"2026-07-13T20:33:53.885116Z","modified":"2026-07-13T20:33:53.885382Z","added":"2026-07-13T20:33:53.885125Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 13.pdf","archived_file_name":null},{"id":12,"correspondent":null,"document_type":null,"title":"Test Document 12","content":"","tags":[],"created":"2026-07-13T20:33:53.876455Z","modified":"2026-07-13T20:33:53.876858Z","added":"2026-07-13T20:33:53.876467Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 12.pdf","archived_file_name":null},{"id":11,"correspondent":null,"document_type":null,"title":"Test Document 11","content":"","tags":[],"created":"2026-07-13T20:33:53.862255Z","modified":"2026-07-13T20:33:53.862626Z","added":"2026-07-13T20:33:53.862271Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 11.pdf","archived_file_name":null},{"id":10,"correspondent":null,"document_type":null,"title":"Test Document 10","content":"","tags":[],"created":"2026-07-13T20:33:53.846705Z","modified":"2026-07-13T20:33:53.847114Z","added":"2026-07-13T20:33:53.846720Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 10.pdf","archived_file_name":null},{"id":9,"correspondent":null,"document_type":null,"title":"Test Document 09","content":"","tags":[],"created":"2026-07-13T20:33:53.825058Z","modified":"2026-07-13T20:33:53.825422Z","added":"2026-07-13T20:33:53.825073Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 09.pdf","archived_file_name":null},{"id":8,"correspondent":null,"document_type":null,"title":"Test Document 08","content":"","tags":[],"created":"2026-07-13T20:33:53.808140Z","modified":"2026-07-13T20:33:53.808425Z","added":"2026-07-13T20:33:53.808153Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 08.pdf","archived_file_name":null},{"id":7,"correspondent":null,"document_type":null,"title":"Test Document 07","content":"","tags":[],"created":"2026-07-13T20:33:53.727654Z","modified":"2026-07-13T20:33:53.728043Z","added":"2026-07-13T20:33:53.727667Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 07.pdf","archived_file_name":null},{"id":6,"correspondent":null,"document_type":null,"title":"Test Document 06","content":"","tags":[],"created":"2026-07-13T20:33:53.706232Z","modified":"2026-07-13T20:33:53.706537Z","added":"2026-07-13T20:33:53.706244Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 06.pdf","archived_file_name":null}]}
```

The body above is the complete `Content-Length: 8364` payload: the pagination envelope followed
by 25 document objects (ids 30 down to 6). Its pretty-printed form and the full field enumeration
appear in Q3.

**Grounding:** `src/paperless/settings.py:L120` registers
`"rest_framework.authentication.TokenAuthentication"` in
`REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` (`src/paperless/settings.py:L117-L121`).

**Cause -> effect:** `HTTP 200` confirms the header is accepted; `TokenAuthentication` parses the
`Authorization: Token <key>` header, looks the key up, and authenticates the bound user.

**On the `X-Api-Version: 2` and `X-Version: 1.7.0` response headers (exact attribution).** These
two headers are added by the project's **custom `ApiVersionMiddleware`**
(`src/paperless/middleware.py:L5-L14`), registered in `MIDDLEWARE` at
`src/paperless/settings.py:L142` — **not** by DRF's `AcceptHeaderVersioning`. The middleware sets
them **only when `request.user.is_authenticated`** (`src/paperless/middleware.py:L11`); it sets
`X-Api-Version` to the **last** entry of `ALLOWED_VERSIONS` (`["1", "2"]` -> `"2"`)
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
(`src/paperless/urls.py:L32` -> `api_router.register(r"documents", UnifiedSearchViewSet)`),
and the router is mounted under the `^api/` prefix (`src/paperless/urls.py:L38-L40`). The
authenticated GET shown in Q1 returned `HTTP 200` at exactly this path.

**Cause -> effect:** router registration of `r"documents"` composed with the `^api/` include
produces the collection path `/api/documents/`; the trailing slash is DRF `DefaultRouter`'s
default. (The bind address/port `127.0.0.1:8123` is only a local capture convenience and is
**not** part of the endpoint contract — the contract is the path `/api/documents/`.)

---

## Q3 — response shape: top-level fields and pagination

**Direct answer:** the response is a **paginated JSON envelope** whose top-level fields are
**`count`, `next`, `previous`, `results`**. **Pagination _is_ involved** — the endpoint does
**not** dump everything at once. It uses DRF `PageNumberPagination` via the project-wide
`StandardPagination` with a **default `page_size` of 25** (client-overridable via the `page_size`
query parameter, up to a **maximum of `max_page_size=100000`**).

**Grounding:** `src/paperless/views.py:L8-L11` defines `StandardPagination(PageNumberPagination)`
with `page_size = 25`, `page_size_query_param = "page_size"`, and `max_page_size = 100000`; it is
applied to the documents viewset at `src/documents/views.py:L182`
(`pagination_class = StandardPagination`).

### Capture the three response bodies (saved to files)

Q3 uses three authenticated GETs at a scale of **30 seeded documents**. Each response body is
first saved to a file with the `save_body.py` helper (created in step 2) so it can be
pretty-printed and asserted below; the byte count is printed as each file is written:

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python save_body.py http://127.0.0.1:8123/api/documents/ 1f350e2500056d57d92f7aabeeef0665c46d91bc body_default.json'
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python save_body.py "http://127.0.0.1:8123/api/documents/?page=2" 1f350e2500056d57d92f7aabeeef0665c46d91bc body_page2.json'
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python save_body.py "http://127.0.0.1:8123/api/documents/?page_size=5" 1f350e2500056d57d92f7aabeeef0665c46d91bc body_pagesize5.json'
wrote 8364 bytes -> body_default.json
wrote 1736 bytes -> body_page2.json
wrote 1760 bytes -> body_pagesize5.json
```

The three files now exist on disk (their sizes match the `Content-Length` headers seen below):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && ls -l body_default.json body_page2.json body_pagesize5.json'
-rw-r--r-- 1 root root 8364 Jul 13 20:35 body_default.json
-rw-r--r-- 1 root root 1736 Jul 13 20:35 body_page2.json
-rw-r--r-- 1 root root 1760 Jul 13 20:35 body_pagesize5.json
```

### The default-page body, in full (pretty-printed)

The complete default-page body was already captured raw in Q1 (`Content-Length: 8364`). Here the
**saved `body_default.json`** is pretty-printed with `python -m json.tool` (reformatted only for
readability — same bytes, not re-fetched), showing the envelope and **all 25** `results` objects:

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python -m json.tool body_default.json'
{
    "count": 30,
    "next": "http://127.0.0.1:8123/api/documents/?page=2",
    "previous": null,
    "results": [
        {
            "id": 30,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 30",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.084758Z",
            "modified": "2026-07-13T20:33:54.084989Z",
            "added": "2026-07-13T20:33:54.084766Z",
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
            "created": "2026-07-13T20:33:54.080110Z",
            "modified": "2026-07-13T20:33:54.080316Z",
            "added": "2026-07-13T20:33:54.080117Z",
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
            "created": "2026-07-13T20:33:54.075647Z",
            "modified": "2026-07-13T20:33:54.075924Z",
            "added": "2026-07-13T20:33:54.075656Z",
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
            "created": "2026-07-13T20:33:54.068392Z",
            "modified": "2026-07-13T20:33:54.068605Z",
            "added": "2026-07-13T20:33:54.068400Z",
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
            "created": "2026-07-13T20:33:54.064234Z",
            "modified": "2026-07-13T20:33:54.064517Z",
            "added": "2026-07-13T20:33:54.064244Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 26.pdf",
            "archived_file_name": null
        },
        {
            "id": 25,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 25",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.055519Z",
            "modified": "2026-07-13T20:33:54.055769Z",
            "added": "2026-07-13T20:33:54.055528Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 25.pdf",
            "archived_file_name": null
        },
        {
            "id": 24,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 24",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.035731Z",
            "modified": "2026-07-13T20:33:54.035970Z",
            "added": "2026-07-13T20:33:54.035738Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 24.pdf",
            "archived_file_name": null
        },
        {
            "id": 23,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 23",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.030403Z",
            "modified": "2026-07-13T20:33:54.030640Z",
            "added": "2026-07-13T20:33:54.030410Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 23.pdf",
            "archived_file_name": null
        },
        {
            "id": 22,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 22",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.026251Z",
            "modified": "2026-07-13T20:33:54.026626Z",
            "added": "2026-07-13T20:33:54.026262Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 22.pdf",
            "archived_file_name": null
        },
        {
            "id": 21,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 21",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:54.011888Z",
            "modified": "2026-07-13T20:33:54.012155Z",
            "added": "2026-07-13T20:33:54.011900Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 21.pdf",
            "archived_file_name": null
        },
        {
            "id": 20,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 20",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.987376Z",
            "modified": "2026-07-13T20:33:53.987724Z",
            "added": "2026-07-13T20:33:53.987390Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 20.pdf",
            "archived_file_name": null
        },
        {
            "id": 19,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 19",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.971210Z",
            "modified": "2026-07-13T20:33:53.971514Z",
            "added": "2026-07-13T20:33:53.971222Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 19.pdf",
            "archived_file_name": null
        },
        {
            "id": 18,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 18",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.954927Z",
            "modified": "2026-07-13T20:33:53.955206Z",
            "added": "2026-07-13T20:33:53.954938Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 18.pdf",
            "archived_file_name": null
        },
        {
            "id": 17,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 17",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.936464Z",
            "modified": "2026-07-13T20:33:53.936645Z",
            "added": "2026-07-13T20:33:53.936470Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 17.pdf",
            "archived_file_name": null
        },
        {
            "id": 16,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 16",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.930276Z",
            "modified": "2026-07-13T20:33:53.930511Z",
            "added": "2026-07-13T20:33:53.930284Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 16.pdf",
            "archived_file_name": null
        },
        {
            "id": 15,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 15",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.922971Z",
            "modified": "2026-07-13T20:33:53.923258Z",
            "added": "2026-07-13T20:33:53.922983Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 15.pdf",
            "archived_file_name": null
        },
        {
            "id": 14,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 14",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.898837Z",
            "modified": "2026-07-13T20:33:53.899197Z",
            "added": "2026-07-13T20:33:53.898850Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 14.pdf",
            "archived_file_name": null
        },
        {
            "id": 13,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 13",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.885116Z",
            "modified": "2026-07-13T20:33:53.885382Z",
            "added": "2026-07-13T20:33:53.885125Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 13.pdf",
            "archived_file_name": null
        },
        {
            "id": 12,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 12",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.876455Z",
            "modified": "2026-07-13T20:33:53.876858Z",
            "added": "2026-07-13T20:33:53.876467Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 12.pdf",
            "archived_file_name": null
        },
        {
            "id": 11,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 11",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.862255Z",
            "modified": "2026-07-13T20:33:53.862626Z",
            "added": "2026-07-13T20:33:53.862271Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 11.pdf",
            "archived_file_name": null
        },
        {
            "id": 10,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 10",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.846705Z",
            "modified": "2026-07-13T20:33:53.847114Z",
            "added": "2026-07-13T20:33:53.846720Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 10.pdf",
            "archived_file_name": null
        },
        {
            "id": 9,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 09",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.825058Z",
            "modified": "2026-07-13T20:33:53.825422Z",
            "added": "2026-07-13T20:33:53.825073Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 09.pdf",
            "archived_file_name": null
        },
        {
            "id": 8,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 08",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.808140Z",
            "modified": "2026-07-13T20:33:53.808425Z",
            "added": "2026-07-13T20:33:53.808153Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 08.pdf",
            "archived_file_name": null
        },
        {
            "id": 7,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 07",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.727654Z",
            "modified": "2026-07-13T20:33:53.728043Z",
            "added": "2026-07-13T20:33:53.727667Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 07.pdf",
            "archived_file_name": null
        },
        {
            "id": 6,
            "correspondent": null,
            "document_type": null,
            "title": "Test Document 06",
            "content": "",
            "tags": [],
            "created": "2026-07-13T20:33:53.706232Z",
            "modified": "2026-07-13T20:33:53.706537Z",
            "added": "2026-07-13T20:33:53.706244Z",
            "archive_serial_number": null,
            "original_file_name": "2026-07-13 Test Document 06.pdf",
            "archived_file_name": null
        }
    ]
}
```

### The other two pages (complete raw responses)

**`?page=2`** (headers + full body — the remaining 5 of 30 documents, with a `previous` link and
`next: null`):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET "http://127.0.0.1:8123/api/documents/?page=2" 1f350e2500056d57d92f7aabeeef0665c46d91bc'
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 20:36:42 GMT
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

{"count":30,"next":null,"previous":"http://127.0.0.1:8123/api/documents/","results":[{"id":5,"correspondent":null,"document_type":null,"title":"Test Document 05","content":"","tags":[],"created":"2026-07-13T20:33:53.652431Z","modified":"2026-07-13T20:33:53.652775Z","added":"2026-07-13T20:33:53.652448Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 05.pdf","archived_file_name":null},{"id":4,"correspondent":null,"document_type":null,"title":"Test Document 04","content":"","tags":[],"created":"2026-07-13T20:33:53.475027Z","modified":"2026-07-13T20:33:53.475314Z","added":"2026-07-13T20:33:53.475036Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 04.pdf","archived_file_name":null},{"id":3,"correspondent":null,"document_type":null,"title":"Test Document 03","content":"","tags":[],"created":"2026-07-13T20:33:53.469308Z","modified":"2026-07-13T20:33:53.469572Z","added":"2026-07-13T20:33:53.469320Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 03.pdf","archived_file_name":null},{"id":2,"correspondent":null,"document_type":null,"title":"Test Document 02","content":"","tags":[],"created":"2026-07-13T20:33:53.444024Z","modified":"2026-07-13T20:33:53.444286Z","added":"2026-07-13T20:33:53.444036Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 02.pdf","archived_file_name":null},{"id":1,"correspondent":null,"document_type":null,"title":"Test Document 01","content":"","tags":[],"created":"2026-07-13T20:33:53.414482Z","modified":"2026-07-13T20:33:53.414767Z","added":"2026-07-13T20:33:53.414496Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 01.pdf","archived_file_name":null}]}
```

**`?page_size=5`** (headers + full body — the page size is client-overridable):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET "http://127.0.0.1:8123/api/documents/?page_size=5" 1f350e2500056d57d92f7aabeeef0665c46d91bc'
HTTP/1.1 200 OK
Date: Mon, 13 Jul 2026 20:36:42 GMT
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

{"count":30,"next":"http://127.0.0.1:8123/api/documents/?page=2&page_size=5","previous":null,"results":[{"id":30,"correspondent":null,"document_type":null,"title":"Test Document 30","content":"","tags":[],"created":"2026-07-13T20:33:54.084758Z","modified":"2026-07-13T20:33:54.084989Z","added":"2026-07-13T20:33:54.084766Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 30.pdf","archived_file_name":null},{"id":29,"correspondent":null,"document_type":null,"title":"Test Document 29","content":"","tags":[],"created":"2026-07-13T20:33:54.080110Z","modified":"2026-07-13T20:33:54.080316Z","added":"2026-07-13T20:33:54.080117Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 29.pdf","archived_file_name":null},{"id":28,"correspondent":null,"document_type":null,"title":"Test Document 28","content":"","tags":[],"created":"2026-07-13T20:33:54.075647Z","modified":"2026-07-13T20:33:54.075924Z","added":"2026-07-13T20:33:54.075656Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 28.pdf","archived_file_name":null},{"id":27,"correspondent":null,"document_type":null,"title":"Test Document 27","content":"","tags":[],"created":"2026-07-13T20:33:54.068392Z","modified":"2026-07-13T20:33:54.068605Z","added":"2026-07-13T20:33:54.068400Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 27.pdf","archived_file_name":null},{"id":26,"correspondent":null,"document_type":null,"title":"Test Document 26","content":"","tags":[],"created":"2026-07-13T20:33:54.064234Z","modified":"2026-07-13T20:33:54.064517Z","added":"2026-07-13T20:33:54.064244Z","archive_serial_number":null,"original_file_name":"2026-07-13 Test Document 26.pdf","archived_file_name":null}]}
```

### Derived assertions (computed from the saved bodies)

`paginate_assert.py` (from step 2) reads the three saved bodies and prints the top-level fields
and result counts:

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python paginate_assert.py'
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
(`src/documents/serialisers.py:L219` -> `class Meta`, `src/documents/serialisers.py:L222-L235` ->
the `fields` tuple). The plain list uses `DocumentSerializer` (not the search serializer) because
no `query`/`more_like_id` query parameter is present:
`UnifiedSearchViewSet.get_serializer_class` returns `DocumentSerializer` unless
`_is_search_request()` is true (`src/documents/views.py:L382-L392`).

**Cause -> effect:** with 30 documents, the default page returns the first 25 items plus a `next`
link and `previous=null`; `?page=2` returns the remaining 5 items with a `previous` link and
`next=null`; `?page_size=5` proves the page size is client-overridable. **25 is the _default_
page size, not a hard cap** — a client may request up to `max_page_size=100000`. The endpoint
pages its results and does **not** dump everything at once.

---

## Q4 — the same request WITHOUT auth: status code and error message

**Direct answer:** the status code is **`401 Unauthorized`** and the error body is
**`{"detail":"Authentication credentials were not provided."}`**.

Command and **complete** observed response (identical GET as Q1, but with no `Authorization`
header):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET http://127.0.0.1:8123/api/documents/'
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 20:36:42 GMT
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

**Cause -> effect:** with no credentials supplied, `IsAuthenticated` denies access, yielding
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

The token model is confirmed via the Django shell (investigation-only), running the
`inspect_token.py` helper created in step 2 (its full contents are shown there). Its raw output:

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py shell < /tmp/pl0/inspect_token.py'
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
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET http://127.0.0.1:8123/api/documents/ wrong_token'
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 20:36:44 GMT
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

**Cause -> effect:** DRF's `TokenAuthentication.authenticate_credentials` looks up the presented
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

All investigation artifacts are removed and the removal is verified. First, inside the container,
the seeded documents and the test user are deleted (deleting the user cascade-deletes its token)
using the `cleanup.py` helper from step 2; the counts confirm removal:

```console
$ docker exec paperless-setup-0 bash -lc 'source /tmp/pl0/env.sh && cd /app/src && python manage.py shell < /tmp/pl0/cleanup.py'
Document rows after delete : 0
testuser rows after delete : 0
Token rows after delete    : 0
```

Re-issuing the **same request with the previously valid token** now fails — proving the token is
invalidated at the API level (its user, and therefore its `authtoken_token` row, are gone):

```console
$ docker exec paperless-setup-0 bash -lc 'cd /tmp/pl0 && python api_probe.py GET http://127.0.0.1:8123/api/documents/ 1f350e2500056d57d92f7aabeeef0665c46d91bc'
HTTP/1.1 401 Unauthorized
Date: Mon, 13 Jul 2026 20:38:44 GMT
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

Then, **on the host** (managing the container and the destination repository), the entire
investigation container is removed — which destroys its SQLite database and all `/tmp/pl0` data:

```console
$ docker rm -f paperless-setup-0
paperless-setup-0
```

Its absence is verified with an explicit shell conditional whose printed line **is** the genuine
output of the command (the underlying `docker ps` filter matches nothing, so the `then` branch
runs):

```console
$ if [ -z "$(docker ps -a --filter name=paperless-setup-0 --format '{{.Names}}')" ]; then echo "paperless-setup-0: not present (removed)"; else echo "paperless-setup-0: STILL PRESENT"; fi
paperless-setup-0: not present (removed)
```

Finally, the destination repository's source tree is confirmed untouched. `git status --porcelain`
prints **nothing** (an empty working tree):

```console
$ git status --porcelain
```

The block above is intentionally empty: `git status --porcelain` produced no output, which means
the working tree is clean. Scoping the same check to the source tree is likewise empty, proving no
source file was modified by this investigation:

```console
$ git status --porcelain -- src/
```

And the only change this branch adds on top of the source baseline `542221a38dff` is this one
documentation file (nothing under `src/`):

```console
$ git diff --name-status 542221a38dff HEAD
A	blitzy/documentation/paperless-ngx_542221a38dff.md
```

The disposable test user, its 40-character token, the 30 seeded documents, the SQLite database,
the temporary helper/inspection scripts, and the container itself are all gone; the token above is
provably non-functional. No product source, configuration, dependency, test, or frontend file was
modified.

---

## Coverage-pass checklist

Every named item from the request is answered, and each box is ticked only where adjacent observed
evidence exists in this document:

- [x] Q1 header **name** answered — `Authorization` (Q1: authenticated `HTTP 200`).
- [x] Q1 header **format** answered — `Token <key>`, 40-character key (Q1 + Q5 `keyword: 'Token'` + issuance token length).
- [x] Q2 complete **endpoint path** answered — `/api/documents/` (Q2 grounding + Q1 `200`).
- [x] Q3 **top-level fields** answered — `count`, `next`, `previous`, `results` (Q3 full body + `paginate_assert.py`).
- [x] Q3 **pagination vs. dump-everything** answered — paginated; **default** `page_size=25`, **max** `100000` (Q3).
- [x] Q3 **default page** proof — `count=30`, `len(results)=25`, `previous=null`, `next=...?page=2` (Q1 raw body + Q3 pretty body + assertion).
- [x] Q3 **`?page=2`** proof — `count=30`, `len(results)=5`, `previous=.../`, `next=null` (complete raw + assertion).
- [x] Q3 **`?page_size=5`** proof — `count=30`, `len(results)=5`, `next=...?page=2&page_size=5` (complete raw + assertion).
- [x] Q3 **`results` item fields** enumerated — 12 `DocumentSerializer` fields, in order (assertion `results[0] keys`).
- [x] Q4 **status code** answered — `401 Unauthorized` (complete raw).
- [x] Q4 **error message** answered — `{"detail":"Authentication credentials were not provided."}` (complete raw).
- [x] Q5 **authentication class** named — `rest_framework.authentication.TokenAuthentication` (settings + `inspect_token.py`).
- [x] Q5 **token model** named — `rest_framework.authtoken.models.Token`; table `authtoken_token` (`inspect_token.py` raw output).
- [x] Q5 **exact model-inspection command shown** — `python manage.py shell < /tmp/pl0/inspect_token.py` (full script in step 2 + raw output).
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
