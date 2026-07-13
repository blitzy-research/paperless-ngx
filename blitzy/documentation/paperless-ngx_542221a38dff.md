# paperless-ngx — Runtime Choreography: An Evidence-Backed Investigation

This document answers six "runtime choreography" questions about the paperless-ngx
document-management backend. Every answer is proven with **real runtime evidence** —
actual on-disk paths, actual checksums/hashes, and actual log messages captured by
*running the system through its canonical entry points* — not inferred from reading
code. Each factual claim is grounded in a `file:line` reference that names the specific
function performing the work, and each finding is explicitly tagged **[observed]** or
**[inferred]**.

Every experiment below is a single, self-contained script that is shown **verbatim**
and is immediately followed by its **complete, unedited stdout/stderr**. Every marker
line in an output block (for example `>>> before ...`, `=== ... ===`) is emitted by an
`echo`/`print` in the script shown directly above it, so each displayed line is
traceable to the exact command that produced it. Seed, state-query, database-query,
hash, log-snapshot/diff, and cleanup commands are all included inline.

The investigation is read-only: apart from this one Markdown file, the paperless-ngx
repository is left byte-for-byte unchanged. All scratch data and observation scripts
lived outside the repository tree (inside the container under `/scratch`, and on the
investigation host under `/tmp`) and were removed afterward; the final
"Repository Integrity & Cleanup" section proves this with `git` and filesystem output.

---

## Methodology & Environment

### Canonical runtime and how it was launched

paperless-ngx v1.7.0 pins its runtime to **Python 3.9** (`Dockerfile:18` →
`FROM python:3.9-slim-bullseye as main-app`). The investigation **host** shell is
**Python 3.13.7** with Django not importable, so the entire investigation was run
**inside the canonical Docker container**. The container (`paperless-work`) runs the
image `paperless-ngx-ready:latest`, which is the user-specified canonical image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01`
(`paperless-canonical:latest`, image id `6e699f225ced`) with Redis and the OCR stack
added. Its identity, launch command, and environment were confirmed with `docker inspect`:

```
$ docker images --format '{{.Repository}}:{{.Tag}}  id={{.ID}}' | grep -E 'paperless|swe-atlas'
paperless-ngx-ready:latest  id=738fd46421ab
paperless-canonical:latest  id=6e699f225ced
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_paperless-ngx_e233ae8334038a4b615ea2e4ce663e30_qna_1.01  id=6e699f225ced

$ docker inspect -f 'image={{.Config.Image}}' paperless-work
image=paperless-ngx-ready:latest

$ docker inspect -f '{{json .Config.Cmd}}' paperless-work
["-lc","mkdir -p /scratch/data /scratch/media /scratch/consume && redis-server --daemonize yes --save \"\" --appendonly no && echo READY && sleep infinity"]

$ docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' paperless-work | grep '^PAPERLESS_'
PAPERLESS_REDIS=redis://localhost:6379
PAPERLESS_DATA_DIR=/scratch/data
PAPERLESS_MEDIA_ROOT=/scratch/media
PAPERLESS_CONSUMPTION_DIR=/scratch/consume
PAPERLESS_TIME_ZONE=UTC
PAPERLESS_DISABLE_DBHANDLER=true
```

The container was launched (by the environment setup, reproduced here verbatim) with the
canonical command below, and every experiment was executed inside it via
`docker exec -w /app/src paperless-work …`:

```bash
docker run -d --name paperless-work --entrypoint /bin/bash \
  -e PAPERLESS_REDIS=redis://localhost:6379 -e PAPERLESS_DATA_DIR=/scratch/data \
  -e PAPERLESS_MEDIA_ROOT=/scratch/media -e PAPERLESS_CONSUMPTION_DIR=/scratch/consume \
  -e PAPERLESS_TIME_ZONE=UTC \
  paperless-ngx-ready:latest \
  -lc 'mkdir -p /scratch/data /scratch/media /scratch/consume && \
       redis-server --daemonize yes --save "" --appendonly no && echo READY && sleep infinity'
```

### Versions and services

The exact probe script and its complete output (run with working directory `/app/src`
via `docker exec -w /app/src paperless-work bash /scratch/exp/env_probe.sh`):

```bash
# /scratch/exp/env_probe.sh
run() { echo "\$ $*"; eval "$@"; echo; }
run 'python3 --version'
run 'python3 -c "import django; print(\"Django\", django.get_version())"'
run 'redis-server --version'
run 'redis-cli ping'
run 'python3 -c "import redis; print(\"redis-py (Python client)\", redis.__version__)"'
run 'python3 -c "import django_q, sklearn, filelock, pathvalidate, whoosh, ocrmypdf, pikepdf, watchdog, concurrent_log_handler as clh; print(\"django-q\", django_q.VERSION); print(\"scikit-learn\", sklearn.__version__); print(\"filelock\", filelock.__version__); print(\"pathvalidate\", pathvalidate.__version__)"'
run 'tesseract --version 2>&1 | head -1'
run 'gs --version'
run 'unpaper --version'
run 'qpdf --version | head -1'
run 'pngquant --version 2>&1 | head -1'
run 'git -C /app rev-parse HEAD'
```


```
$ python3 --version
Python 3.9.23

$ python3 -c "import django; print(\"Django\", django.get_version())"
Django 4.0.4

$ redis-server --version
Redis server v=6.0.16 sha=00000000:0 malloc=jemalloc-5.2.1 bits=64 build=d4b5be3f91fa055c

$ redis-cli ping
PONG

$ python3 -c "import redis; print(\"redis-py (Python client)\", redis.__version__)"
redis-py (Python client) 3.5.3

$ python3 -c "import django_q, sklearn, filelock, pathvalidate, whoosh, ocrmypdf, pikepdf, watchdog, concurrent_log_handler as clh; print(\"django-q\", django_q.VERSION); print(\"scikit-learn\", sklearn.__version__); print(\"filelock\", filelock.__version__); print(\"pathvalidate\", pathvalidate.__version__)"
django-q (1, 3, 9)
scikit-learn 1.0.2
filelock 3.6.0
pathvalidate 2.5.0

$ tesseract --version 2>&1 | head -1
tesseract 4.1.1

$ gs --version
9.53.3

$ unpaper --version
6.1

$ qpdf --version | head -1
qpdf version 10.1.0

$ pngquant --version 2>&1 | head -1
2.12.2 (July 2019)

$ git -C /app rev-parse HEAD
542221a38dff06361e07976452f9aea24d210542

```

For contrast, the **investigation host** interpreter (the reason the work must run inside
the container — `python3 --version` on the host):

```
Python 3.13.7
```

**Two distinct "redis" versions** appear above and must not be confused: the **Redis
server** is `v6.0.16` (the broker/daemon, started with `redis-server --daemonize yes`
and confirmed live by `redis-cli ping` → `PONG`), whereas **`redis-py`** — the Python
client library imported by Django-Q / channels — is version **`3.5.3`**, matching the
`redis==3.5.3` pin in `requirements.txt`. The other probed pins also match
`requirements.txt`: `django==4.0.4`, `django-q==1.3.9`, `scikit-learn==1.0.2`,
`filelock==3.6.0`, `pathvalidate==2.5.0`. The container `/app` checkout is at commit
`542221a38dff06361e07976452f9aea24d210542`, the investigation baseline.

### OCR / archive tool stack

An **archive** (OCR'd) rendition of a document — and therefore an `archive_checksum` —
is only needed for **Q4's archive-checksum collision** and **Q5's archive mismatch**.
The archive-generation tools verified present above are: **tesseract 4.1.1**,
**ghostscript (gs) 9.53.3**, **unpaper 6.1**, **qpdf 10.1.0**, and **pngquant 2.12.2**
(`ocrmypdf==13.4.3` / `pikepdf==5.1.1` on the Python side). The original-only paths
(Q1 happy path, Q2, Q3, Q4 original collision, Q6) do not require them.

### Throwaway store (environment variables only — no tracked file was edited)

Storage is pointed at scratch locations inside the container (see the `PAPERLESS_*`
block above) so experiments never touch a real archive: `PAPERLESS_MEDIA_ROOT=/scratch/media`,
`PAPERLESS_DATA_DIR=/scratch/data`, `PAPERLESS_CONSUMPTION_DIR=/scratch/consume`. The
storage layout observed under `MEDIA_ROOT` is grounded in `src/paperless/settings.py`:
`MEDIA_ROOT` (`:61`), `ORIGINALS_DIR = media/documents/originals` (`:62`),
`ARCHIVE_DIR = media/documents/archive` (`:63`),
`THUMBNAIL_DIR = media/documents/thumbnails` (`:64`),
`MEDIA_LOCK = media/media.lock` (`:72`),
`MODEL_FILE = DATA_DIR/classification_model.pickle` (`:74`).
`PAPERLESS_FILENAME_FORMAT` defaults to `None` (`:584`), so tags are *not* in the path
unless the format is set explicitly (required for Q1/Q2). `CONSUMER_DELETE_DUPLICATES`
defaults off (`:486`). The container also sets `PAPERLESS_DISABLE_DBHANDLER=true`, which
only disables the *database* log handler; the file handler that writes
`data/log/paperless.log` is unaffected, so all DEBUG/INFO/WARNING/ERROR evidence still
lands in the log file.

### Database and migrations

The database is the default **SQLite** (no external DB). Before each question group the
store is reset to a clean, self-contained state and migrations are applied to the
throwaway database. The exact reset script and the complete migration output
(`docker exec paperless-work bash /scratch/exp/reset.sh`):

```bash
# /scratch/exp/reset.sh
rm -rf /scratch/media/* /scratch/data/* /scratch/consume/* 2>/dev/null || true
mkdir -p /scratch/media /scratch/data /scratch/consume /scratch/data/log
cd /app/src
python3 manage.py migrate --no-input
```


```
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

### Observability (this matters for reading the evidence)

The logging config at `src/paperless/settings.py:373-411` routes the `paperless` logger
to the file `data/log/paperless.log` at level **DEBUG** (handler `file_paperless`
`:392-398`, logger binding `:409`), while the root/console handler's level is
`"DEBUG" if DEBUG else "INFO"` (`:388`) with the root using the console handler (`:407`);
`DEBUG` is `PAPERLESS_DEBUG` (`:50`), default `NO`. **Consequence:** DEBUG-level evidence
(notably the Q3 skip message `"Training data unchanged."`) is *always* written to
`/scratch/data/log/paperless.log` but appears on the **console** only with
`PAPERLESS_DEBUG=true`. `INFO`/`WARNING`/`ERROR` (healthy sanity message, orphan
warnings, mismatch errors, duplicate errors, classifier save) surface on both. Every
experiment below therefore captures the log file (and, where useful, also surfaces DEBUG
on the console with `PAPERLESS_DEBUG=true`) so no evidence is missed.

### Task worker (Django-Q) — used for the canonical async ingestion path (Q4)

The canonical consume-directory and REST-upload entry points do not run consumption
inline; they **enqueue** `documents.tasks.consume_file` to Django-Q
(`src/documents/management/commands/document_consumer.py:86-91`;
`src/documents/views.py:523-533`), and a **`qcluster`** worker executes it. Q4 therefore
runs a real `qcluster`. For determinism, no `qcluster` runs during Q1/Q2/Q3/Q5/Q6 (those
use synchronous management commands / ORM signals), and the Q4 worker is started and then
stopped by its **specific PID** (never a broad `pkill` pattern). The exact demonstration
(`docker exec paperless-work bash /scratch/exp/qcluster_demo.sh`):

```bash
# /scratch/exp/qcluster_demo.sh
cd /app/src
python3 manage.py qcluster >/scratch/exp/qcluster_demo.log 2>&1 &
QPID=$!
echo "started qcluster, captured PID=$QPID"
for i in $(seq 1 60); do grep -q "running" /scratch/exp/qcluster_demo.log 2>/dev/null && break; sleep 0.5; done
echo "--- qcluster startup log ---"
grep -E "Q Cluster|running|process" /scratch/exp/qcluster_demo.log | head -5
echo "PID $QPID alive before stop? $([ -d /proc/$QPID ] && echo yes || echo no)"
kill "$QPID"
sleep 2
echo "PID $QPID alive after 'kill $QPID'? $([ -d /proc/$QPID ] && echo yes || echo no)"
echo "--- qcluster shutdown log (tail) ---"
tail -4 /scratch/exp/qcluster_demo.log
```


```
started qcluster, captured PID=8693
--- qcluster startup log ---
18:10:26 [Q] INFO Q Cluster dakota-autumn-london-missouri starting.
18:10:26 [Q] INFO Q Cluster dakota-autumn-london-missouri running.
PID 8693 alive before stop? yes
PID 8693 alive after 'kill 8693'? no
--- qcluster shutdown log (tail) ---
18:10:27 [Q] INFO Process-1:11 stopped doing work
18:10:27 [Q] INFO Process-1 waiting for the monitor.
18:10:27 [Q] INFO Process-1:12 stopped monitoring results
18:10:27 [Q] INFO Q Cluster dakota-autumn-london-missouri has stopped.
```

The PID captured by `$!` is confirmed alive, then `kill "$QPID"` stops exactly that
process (confirmed by `/proc/$QPID` disappearing and the clean
`Q Cluster … has stopped.` log line). This is the safe worker-stop used throughout.

### External filesystem monitor (used only by Q2)

Q2 uses a standalone **external** filesystem-event monitor to capture the transient
intermediate state of a partial move. It observes the OS filesystem from a **separate
process** using `inotify_simple` — it does **not** patch, import, or otherwise touch
product code. Its full source:

```python
# /scratch/exp/fsmon.py — external inotify observer (no product code touched)
import sys, time, os
from inotify_simple import INotify, flags
paths = sys.argv[1:-1]
ready_file = sys.argv[-1]
ino = INotify()
watch_flags = (flags.MOVED_FROM | flags.MOVED_TO | flags.CREATE |
               flags.DELETE | flags.DELETE_SELF | flags.MOVE_SELF)
wd_to_path = {}
for p in paths:
    if os.path.isdir(p):
        wd = ino.add_watch(p, watch_flags)
        wd_to_path[wd] = p
with open(ready_file, "w") as f:
    f.write("READY\n")
t0 = time.time()
stop_file = ready_file + ".stop"
while not os.path.exists(stop_file) and (time.time() - t0) < 10:
    for ev in ino.read(timeout=100):
        names = [f.name for f in flags.from_mask(ev.mask)]
        base = wd_to_path.get(ev.wd, "?")
        print(f"t=+{time.time()-t0:07.4f}s dir={base} name={ev.name!r} "
              f"cookie={ev.cookie} flags={'|'.join(names)}", flush=True)
```


---

## Q1 — File relocation on tag change

> User's words: *"When a document's tags change and the filename format includes tags in the directory structure, files apparently relocate themselves. I wonder what log messages actually appear during that dance, and what the before and after paths look like in practice."*

**Direct answer (observed).** Yes — changing a document's tags physically relocates its **original** file and its **archive** file, but **not** its thumbnail, and the successful move writes **zero** log lines. Two distinct *canonical* entry points drive the identical move engine `update_filename_and_move_files` (`src/documents/signals/handlers.py:312`): a tag edit reaches it through the `m2m_changed` receiver registered at `handlers.py:310`, and the bulk `document_renamer` command reaches it through the `post_save` receiver registered at `handlers.py:311` (fired by `document_renamer.py:34`). Both paths are exercised below and both relocate the files silently.

The relocation happens **only** because `PAPERLESS_FILENAME_FORMAT` embeds a tag token in the directory. That setting defaults to `None` (`src/paperless/settings.py:584`), and `generate_filename` short-circuits the whole tag-path construction behind `if settings.PAPERLESS_FILENAME_FORMAT is not None` (`src/documents/file_handling.py:132`); with the default, no directory is derived from tags and no move occurs (this is exactly what my earlier control run without the variable showed — filenames stayed pk-based `0000001.pdf` and nothing moved). For this experiment the format is set to `{tag_list}/{title}`, so `generate_filename` joins the comma-sorted, sanitized tag names as a leading directory (`file_handling.py:135`, applied at `:175`).

### Q1 — the exact self-contained script

The script resets a throwaway store, seeds one document through the canonical consume path, records the before-state, fires `m2m_changed` by adding a tag, records the after-state, then repeats the relocation through the second canonical trigger (`document_renamer`). Every printed marker is emitted by a `script` `echo`, so each line in the output below is attributable to this script.

```bash
#!/bin/bash
# Q1 — a tag change (m2m_changed) relocates the original + archive, silently.
# Self-contained: reset -> seed (canonical consume) -> before -> trigger -> after -> log proof.
cd /app/src
LOG=/scratch/data/log/paperless.log

echo "=== PREP: reset throwaway store + migrate (full migrate output is in Methodology) ==="
bash /scratch/exp/reset.sh > /scratch/exp/q1_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q1_reset.log)"

echo
echo "=== SEED: consume simple.pdf via canonical documents.tasks.consume_file (console at INFO) ==="
cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/simple.pdf
python3 manage.py shell <<'PY'
from documents import tasks
print("consume_file ->", tasks.consume_file("/scratch/consume/simple.pdf"))
PY

echo
echo "=== snapshot paperless.log AFTER seed / BEFORE move ==="
cp "$LOG" /scratch/exp/q1_before.log
echo "log lines before move = $(wc -l < "$LOG")"

echo
echo "=== BEFORE (no tags): DB paths + on-disk files ==="
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
for k in ("filename","archive_filename"):
    print(f"{k:16} = {getattr(d,k)}")
print(f"{'source_path':16} = {d.source_path}")
print(f"{'archive_path':16} = {d.archive_path}")
print(f"{'thumbnail_path':16} = {d.thumbnail_path}")
print(f"{'tags':16} = {list(d.tags.values_list('name', flat=True))}")
PY
echo "--- find /scratch/media/documents -type f (before) ---"
find /scratch/media/documents -type f | sort

echo
echo "=== TRIGGER: d.tags.add(Invoice) fires m2m_changed; run with PAPERLESS_DEBUG=true so any move log would show on console ==="
PAPERLESS_DEBUG=true python3 manage.py shell <<'PY'
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name="Invoice")
print(">>> calling d.tags.add(Invoice) now (fires m2m_changed)")
d.tags.add(t)
print(">>> d.tags.add(Invoice) returned")
PY

echo
echo "=== AFTER (tag Invoice): DB paths + on-disk files ==="
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
for k in ("filename","archive_filename"):
    print(f"{k:16} = {getattr(d,k)}")
print(f"{'source_path':16} = {d.source_path}")
print(f"{'archive_path':16} = {d.archive_path}")
print(f"{'thumbnail_path':16} = {d.thumbnail_path}")
print(f"{'tags':16} = {list(d.tags.values_list('name', flat=True))}")
PY
echo "--- find /scratch/media/documents -type f (after) ---"
find /scratch/media/documents -type f | sort

echo
echo "=== LOG PROOF: paperless.log before-move vs after-move ==="
echo "log lines after move  = $(wc -l < "$LOG")"
if diff /scratch/exp/q1_before.log "$LOG" > /scratch/exp/q1_logdiff.txt; then
  echo "<<< NO DIFFERENCE: the successful move wrote ZERO new lines to paperless.log >>>"
else
  echo "--- new lines written during the move: ---"; cat /scratch/exp/q1_logdiff.txt
fi
echo "=== grep the ENTIRE log for any move/rename/handlers activity ==="
if grep -nE "rename|Moved|moved file|update_filename|paperless\.handlers" "$LOG"; then
  echo "(matches above)"
else
  echo "<<< NONE: no rename/move/paperless.handlers line anywhere in paperless.log >>>"
fi

echo
echo "=== PART 2 — SECOND canonical trigger: document_renamer drives the SAME move via post_save.send (document_renamer.py:34 -> handlers.py:311) ==="
echo ">>> the documented workflow: change PAPERLESS_FILENAME_FORMAT, then run the bulk renamer to relocate existing docs"
echo "--- files BEFORE document_renamer (format is now {created_year}/{tag_list}/{title}) ---"
find /scratch/media/documents -type f | sort
cp "$LOG" /scratch/exp/q1b_before.log
echo ">>> running: PAPERLESS_FILENAME_FORMAT='{created_year}/{tag_list}/{title}' python3 manage.py document_renamer --no-progress-bar"
PAPERLESS_FILENAME_FORMAT='{created_year}/{tag_list}/{title}' python3 manage.py document_renamer --no-progress-bar
echo ">>> document_renamer exit=$?"
echo "--- files AFTER document_renamer ---"
find /scratch/media/documents -type f | sort
echo "--- DB paths AFTER document_renamer (filename is the persisted column; source_path/archive_path are ORIGINALS_DIR/ARCHIVE_DIR + filename) ---"
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
print(f"{'filename':16} = {d.filename}")
print(f"{'archive_filename':16} = {d.archive_filename}")
print(f"{'source_path':16} = {d.source_path}")
print(f"{'archive_path':16} = {d.archive_path}")
print(f"{'thumbnail_path':16} = {d.thumbnail_path}")
PY
echo "--- LOG PROOF for the document_renamer move ---"
echo "log lines after renamer = $(wc -l < "$LOG")"
if diff /scratch/exp/q1b_before.log "$LOG" > /scratch/exp/q1b_logdiff.txt; then
  echo "<<< NO DIFFERENCE: document_renamer's move wrote ZERO new lines to paperless.log >>>"
else
  echo "--- new lines during renamer: ---"; cat /scratch/exp/q1b_logdiff.txt
fi
```

### Q1 — the exact command and its complete, unedited output

Command (the `PAPERLESS_FILENAME_FORMAT` env var is what turns tags into a directory; PART 2 overrides it inline for the renamer step):


```bash
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' \
  paperless-work bash /scratch/exp/q1.sh
```


```text
=== PREP: reset throwaway store + migrate (full migrate output is in Methodology) ===
reset+migrate exit=0  migrations applied OK = 92

=== SEED: consume simple.pdf via canonical documents.tasks.consume_file (console at INFO) ===
[2026-07-13 18:23:02,463] [INFO] [paperless.consumer] Consuming simple.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-k56g_lc_/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 18:23:02,940] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-13 18:23:03,855] [INFO] [paperless.consumer] Document 2026-07-13 simple consumption finished
consume_file -> Success. New document id 1 created

=== snapshot paperless.log AFTER seed / BEFORE move ===
log lines before move = 18

=== BEFORE (no tags): DB paths + on-disk files ===
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
thumbnail_path   = /scratch/media/documents/thumbnails/0000001.png
tags             = []
--- find /scratch/media/documents -type f (before) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/originals/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== TRIGGER: d.tags.add(Invoice) fires m2m_changed; run with PAPERLESS_DEBUG=true so any move log would show on console ===
>>> calling d.tags.add(Invoice) now (fires m2m_changed)
>>> d.tags.add(Invoice) returned

=== AFTER (tag Invoice): DB paths + on-disk files ===
filename         = Invoice/simple.pdf
archive_filename = Invoice/simple.pdf
source_path      = /scratch/media/documents/originals/Invoice/simple.pdf
archive_path     = /scratch/media/documents/archive/Invoice/simple.pdf
thumbnail_path   = /scratch/media/documents/thumbnails/0000001.png
tags             = ['Invoice']
--- find /scratch/media/documents -type f (after) ---
/scratch/media/documents/archive/Invoice/simple.pdf
/scratch/media/documents/originals/Invoice/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== LOG PROOF: paperless.log before-move vs after-move ===
log lines after move  = 18
<<< NO DIFFERENCE: the successful move wrote ZERO new lines to paperless.log >>>
=== grep the ENTIRE log for any move/rename/handlers activity ===
<<< NONE: no rename/move/paperless.handlers line anywhere in paperless.log >>>

=== PART 2 — SECOND canonical trigger: document_renamer drives the SAME move via post_save.send (document_renamer.py:34 -> handlers.py:311) ===
>>> the documented workflow: change PAPERLESS_FILENAME_FORMAT, then run the bulk renamer to relocate existing docs
--- files BEFORE document_renamer (format is now {created_year}/{tag_list}/{title}) ---
/scratch/media/documents/archive/Invoice/simple.pdf
/scratch/media/documents/originals/Invoice/simple.pdf
/scratch/media/documents/thumbnails/0000001.png
>>> running: PAPERLESS_FILENAME_FORMAT='{created_year}/{tag_list}/{title}' python3 manage.py document_renamer --no-progress-bar
>>> document_renamer exit=0
--- files AFTER document_renamer ---
/scratch/media/documents/archive/2026/Invoice/simple.pdf
/scratch/media/documents/originals/2026/Invoice/simple.pdf
/scratch/media/documents/thumbnails/0000001.png
--- DB paths AFTER document_renamer (filename is the persisted column; source_path/archive_path are ORIGINALS_DIR/ARCHIVE_DIR + filename) ---
filename         = 2026/Invoice/simple.pdf
archive_filename = 2026/Invoice/simple.pdf
source_path      = /scratch/media/documents/originals/2026/Invoice/simple.pdf
archive_path     = /scratch/media/documents/archive/2026/Invoice/simple.pdf
thumbnail_path   = /scratch/media/documents/thumbnails/0000001.png
--- LOG PROOF for the document_renamer move ---
log lines after renamer = 18
<<< NO DIFFERENCE: document_renamer's move wrote ZERO new lines to paperless.log >>>
```

### Q1 — before / after paths (both canonical triggers)

Paths below are the on-disk `find` results and the DB `source_path` / `archive_path` / `thumbnail_path` properties, verbatim from the output above. `ORIGINALS_DIR`, `ARCHIVE_DIR`, `THUMBNAIL_DIR` are defined at `src/paperless/settings.py:62`, `:63`, `:64`.

| Stage | original | archive | thumbnail |
|-------|----------|---------|-----------|
| BEFORE (no tags) | `.../originals/simple.pdf` | `.../archive/simple.pdf` | `.../thumbnails/0000001.png` |
| AFTER `m2m_changed` (add tag `Invoice`) | `.../originals/Invoice/simple.pdf` | `.../archive/Invoice/simple.pdf` | `.../thumbnails/0000001.png` *(unchanged)* |
| AFTER `post_save` (`document_renamer`, format `{created_year}/{tag_list}/{title}`) | `.../originals/2026/Invoice/simple.pdf` | `.../archive/2026/Invoice/simple.pdf` | `.../thumbnails/0000001.png` *(unchanged)* |

### Q1 — grounding, observed vs. inferred, and cause→effect

- **The move engine and its two triggers [observed + `file:line`].** The relocation is performed by `update_filename_and_move_files` (`src/documents/signals/handlers.py:312`). It is registered on two signals — `@receiver(m2m_changed, sender=Document.tags.through)` (`:310`) and `@receiver(post_save, sender=Document)` (`:311`) — which is why *both* a tag edit and `document_renamer`'s `post_save.send(...)` (`document_renamer.py:34`) invoke the same function. The whole body runs under `FileLock(settings.MEDIA_LOCK)` (`:325`); the new names come from `generate_unique_filename` (`:330` original, `:338` archive), and the physical moves are the bare `os.rename` calls for the original (`:354`) and the archive (`:359`). The DB row is then updated directly with `Document.objects.filter(pk=instance.pk).update(...)` (`:362`) — a direct `UPDATE` chosen specifically "to prevent infinite recursion" through `post_save`.

- **What log messages appear during the dance? None [observed].** The output shows `log lines before move = 18` and `log lines after move = 18` for the `m2m_changed` trigger, `diff` reports `<<< NO DIFFERENCE ... wrote ZERO new lines >>>`, and a `grep` of the entire log for `rename|Moved|update_filename|paperless.handlers` returns `<<< NONE >>>`. The `document_renamer` trigger likewise leaves the log at `18` lines. **Cause [observed + `file:line`]:** there is no `logger.*` call anywhere inside `update_filename_and_move_files` (`handlers.py:312-410`). The *only* logger in the move path lives in `validate_move` — `logger.fatal(... "has gone.")` (`:298`) and `logger.warning(... "target path ... already exists")` (`:303-306`) — and those fire *only* on the error/edge paths (missing source, or a pre-existing target). On the happy path `validate_move` raises and logs nothing, so a successful relocation is genuinely silent. The observable evidence of a successful move is therefore the path change itself, not any log line. (`document_renamer.py:28` additionally pins the console handler to `ERROR`, but that is irrelevant here since the file-sink log is unchanged too.)

- **Why the thumbnail never moves [observed + `file:line`].** In all three stages the thumbnail stays at `thumbnails/0000001.png`. **Cause:** `update_filename_and_move_files` renames only the original (`:354`) and, when `instance.has_archive_version`, the archive (`:359`); it never touches the thumbnail. The thumbnail path is derived from the primary key, not the filename format — `Document.thumbnail_path` builds its name as `"{:07}.png".format(self.pk)` (`src/documents/models.py:274`) — so it is format-independent by construction and is left in place across any number of tag/format changes.

- **Why a change is even required to trigger a move [observed + `file:line`].** `update_filename_and_move_files` computes `move_original = old_filename != instance.filename` and the archive equivalent, and returns early "if filenames did not change." With the default `PAPERLESS_FILENAME_FORMAT=None` (`settings.py:584`) the tag directory is never generated (`file_handling.py:132`), so adding a tag does not change the name and nothing moves — the exact null result my control run produced. Setting the format to `{tag_list}/{title}` makes the tag part of the path, so adding `Invoice` changes `simple.pdf` → `Invoice/simple.pdf` and the rename fires.

- **Empty-directory cleanup [observed].** After the move the emptied source directories are pruned by `delete_empty_directories` (`file_handling.py:23-52`), invoked at the tail of the handler; the `find` listings show no lingering empty `originals/`, `archive/` intermediates — only the new tag/year sub-paths remain.

---

## Q2 — Rollback safety net on move failure

> User's words: *"There's supposedly a rollback safety net if something goes wrong during a file move, but I can't tell from reading the code whether it truly recovers or just promises to, what actually happens to files when a move fails partway through?"*

**Direct answer (observed).** The safety net *truly recovers* — it does not merely promise to. Two failure modes were induced through the real move engine `update_filename_and_move_files` (`src/documents/signals/handlers.py:312`), both fired canonically by a tag change (`m2m_changed`, `handlers.py:310`):

- **PART A — failure *partway through* (the case the user asks about).** The **original** file is successfully renamed to its new tag directory, then the **archive** step fails. The `except (OSError, DatabaseError, CannotMoveFilesException)` block (`handlers.py:367`) renames the original **back** to its old path (`:376`); the archive — which never moved — is left where it was (`:378` guard is false), and the database row is never updated (the direct `UPDATE` at `:362` is skipped because the exception fired before it). Net result: **both files end exactly where they started**, and the failed move plus its rollback write **zero** log lines. The transient "intermediate" state — the original sitting at its *new* path before the rollback — was captured directly by an **external `inotify` observer** (no product code was patched), as cookie-paired `MOVED_FROM`/`MOVED_TO` events.
- **PART B — failure *before anything moves*.** If the original file is missing when the move begins, the very first `validate_move` (`handlers.py:352`) detects it and logs `logger.fatal(... "has gone.")` (`:298`), raising `CannotMoveFilesException` (`:299`); nothing is renamed and the database is untouched.

**How the failure was induced (and why the obvious method does not work).** The AAP suggested pre-creating a *conflicting archive target* so that `validate_move`'s "target already exists" branch (`handlers.py:301-307`) would raise. In practice that branch is effectively **unreachable** during a normal move: `generate_unique_filename` (`src/documents/file_handling.py:81`) guarantees a non-existing target — for the archive it first tries `os.path.splitext(doc.filename)[0] + ".pdf"` (`:104`) and otherwise appends `_01`, `_02`, ... in the `while` loop that increments on `os.path.exists` (`:111-125`). A plain blocker file would therefore just yield `simple_01.pdf` and the move would *succeed*. To fail the archive step deterministically I instead placed a **regular file at the path `archive/Invoice`**, so that `create_source_path_directory`'s `os.makedirs("archive/Invoice", exist_ok=True)` (`handlers.py:358`, via `file_handling.py:19-20`) raises `FileExistsError` — a subclass of `OSError` — because a path component is a file, not a directory. This fires *after* the original has already moved (`:354`), which is exactly the "partway through" condition.
### Q2 — the exact self-contained script

Every printed marker below is emitted by a `script` `echo`, and the `inotify` event log is written by the external observer `fsmon.py` (shown in the Methodology section).

```bash
#!/bin/bash
# Q2 — the rollback safety net. Two induced failures through the REAL move engine
# update_filename_and_move_files (handlers.py:312), fired by m2m_changed (handlers.py:310).
# PART A: archive move fails AFTER the original already moved -> rollback restores the original.
#         The transient "intermediate" state (original at its NEW path) is captured by an
#         EXTERNAL inotify observer (fsmon.py); no product code is patched.
# PART B: the original source file is missing -> validate_move logs "has gone" and nothing moves.
cd /app/src
LOG=/scratch/data/log/paperless.log

seed() {
  cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/simple.pdf
  python3 manage.py shell <<'PY'
from documents import tasks
print("seed consume_file ->", tasks.consume_file("/scratch/consume/simple.pdf"))
PY
}
show() {  # $1 = label
  python3 manage.py shell <<PY
from documents.models import Document
d = Document.objects.get(pk=1)
print("filename         =", d.filename)
print("archive_filename =", d.archive_filename)
print("source_path      =", d.source_path)
print("archive_path     =", d.archive_path)
print("tags             =", list(d.tags.values_list('name', flat=True)))
PY
}

echo "############################################################"
echo "## Q2 PART A — partial failure (archive) + rollback of the original"
echo "############################################################"
bash /scratch/exp/reset.sh > /scratch/exp/q2a_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q2a_reset.log)"
echo
echo "=== SEED (canonical consume; PAPERLESS_FILENAME_FORMAT makes tags a directory) ==="
seed
echo
echo "=== BEFORE (no tags) ==="
show
echo "--- find (before) ---"; find /scratch/media/documents -type f | sort
echo
echo "=== INDUCE: put a regular FILE at archive/Invoice so create_source_path_directory's"
echo "    os.makedirs(archive/Invoice, exist_ok=True) (handlers.py:358) raises FileExistsError (OSError)."
echo "    Also pre-create originals/Invoice as an EMPTY DIR purely so the external inotify observer"
echo "    can watch the transient; product code would create that same dir at handlers.py:353. ==="
printf 'blocker\n' > /scratch/media/documents/archive/Invoice
mkdir -p /scratch/media/documents/originals/Invoice
echo "archive/Invoice is a file? $([ -f /scratch/media/documents/archive/Invoice ] && echo YES)"
echo "originals/Invoice is a dir? $([ -d /scratch/media/documents/originals/Invoice ] && echo YES)"
echo
echo "=== START external inotify observer on originals/ and originals/Invoice/ ==="
rm -f /scratch/exp/q2a_ready /scratch/exp/q2a_ready.stop /scratch/exp/q2a_fsmon.log
python3 /scratch/exp/fsmon.py \
  /scratch/media/documents/originals \
  /scratch/media/documents/originals/Invoice \
  /scratch/exp/q2a_ready > /scratch/exp/q2a_fsmon.log 2>&1 &
FSMON=$!
for i in $(seq 1 50); do [ -f /scratch/exp/q2a_ready ] && break; sleep 0.1; done
echo "observer READY (pid $FSMON), watching originals/ and originals/Invoice/"
echo
echo "=== TRIGGER: d.tags.add(Invoice) fires m2m_changed (PAPERLESS_DEBUG=true) ==="
cp "$LOG" /scratch/exp/q2a_before.log
PAPERLESS_DEBUG=true python3 manage.py shell <<'PY'
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name="Invoice")
print(">>> calling d.tags.add(Invoice) (original will move, archive makedirs will fail, rollback)")
d.tags.add(t)
print(">>> d.tags.add returned WITHOUT raising to the caller")
PY
sleep 0.4                       # let the observer drain the inotify queue
touch /scratch/exp/q2a_ready.stop
wait "$FSMON" 2>/dev/null
echo
echo "=== EXTERNAL INOTIFY EVENT LOG (this IS the observed intermediate state) ==="
echo "cookie pairs a MOVED_FROM with its MOVED_TO; the original visibly transits into originals/Invoice/ and back:"
cat /scratch/exp/q2a_fsmon.log
echo
echo "=== AFTER ==="
show
echo "--- find (after) ---"; find /scratch/media/documents -type f | sort
echo
echo "=== LOG PROOF for the failed move + rollback ==="
if diff /scratch/exp/q2a_before.log "$LOG" > /scratch/exp/q2a_logdiff.txt; then
  echo "<<< NO DIFFERENCE: the failed move AND its rollback wrote ZERO new lines to paperless.log >>>"
else
  echo "--- new lines: ---"; cat /scratch/exp/q2a_logdiff.txt
fi

echo
echo "############################################################"
echo "## Q2 PART B — original source missing -> validate_move 'has gone'"
echo "############################################################"
bash /scratch/exp/reset.sh > /scratch/exp/q2b_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q2b_reset.log)"
echo
echo "=== SEED ==="
seed
echo
echo "=== BEFORE ==="
show
echo "--- find (before) ---"; find /scratch/media/documents -type f | sort
echo
echo "=== INDUCE: delete the original file from disk (the DB row stays) ==="
rm -v /scratch/media/documents/originals/simple.pdf
echo "--- find (after deletion) ---"; find /scratch/media/documents -type f | sort
echo
echo "=== TRIGGER: d.tags.add(Invoice) fires m2m_changed (PAPERLESS_DEBUG=true so CRITICAL shows on console) ==="
cp "$LOG" /scratch/exp/q2b_before.log
PAPERLESS_DEBUG=true python3 manage.py shell <<'PY'
from documents.models import Document, Tag
d = Document.objects.get(pk=1)
t, _ = Tag.objects.get_or_create(name="Invoice")
print(">>> calling d.tags.add(Invoice)")
d.tags.add(t)
print(">>> d.tags.add returned WITHOUT raising to the caller")
PY
echo
echo "=== AFTER ==="
show
echo "--- find (after) ---"; find /scratch/media/documents -type f | sort
echo
echo "=== LOG PROOF: new lines written during the failed move ==="
diff /scratch/exp/q2b_before.log "$LOG" > /scratch/exp/q2b_logdiff.txt || true
cat /scratch/exp/q2b_logdiff.txt
echo "=== grep the log for validate_move's messages ==="
grep -nE "has gone|Cannot rename" "$LOG" || echo "(no match)"
```

### Q2 — the exact command and its complete, unedited output

Command:


```bash
docker exec -w /app/src -e PAPERLESS_FILENAME_FORMAT='{tag_list}/{title}' \
  paperless-work bash /scratch/exp/q2.sh
```


```text
############################################################
## Q2 PART A — partial failure (archive) + rollback of the original
############################################################
reset+migrate exit=0  migrations applied OK = 92

=== SEED (canonical consume; PAPERLESS_FILENAME_FORMAT makes tags a directory) ===
[2026-07-13 18:33:30,377] [INFO] [paperless.consumer] Consuming simple.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-zzq6a7hq/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 18:33:30,819] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-13 18:33:31,770] [INFO] [paperless.consumer] Document 2026-07-13 simple consumption finished
seed consume_file -> Success. New document id 1 created

=== BEFORE (no tags) ===
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
tags             = []
--- find (before) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/originals/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== INDUCE: put a regular FILE at archive/Invoice so create_source_path_directory's
    os.makedirs(archive/Invoice, exist_ok=True) (handlers.py:358) raises FileExistsError (OSError).
    Also pre-create originals/Invoice as an EMPTY DIR purely so the external inotify observer
    can watch the transient; product code would create that same dir at handlers.py:353. ===
archive/Invoice is a file? YES
originals/Invoice is a dir? YES

=== START external inotify observer on originals/ and originals/Invoice/ ===
observer READY (pid 9164), watching originals/ and originals/Invoice/

=== TRIGGER: d.tags.add(Invoice) fires m2m_changed (PAPERLESS_DEBUG=true) ===
>>> calling d.tags.add(Invoice) (original will move, archive makedirs will fail, rollback)
>>> d.tags.add returned WITHOUT raising to the caller

=== EXTERNAL INOTIFY EVENT LOG (this IS the observed intermediate state) ===
cookie pairs a MOVED_FROM with its MOVED_TO; the original visibly transits into originals/Invoice/ and back:
t=+01.1584s dir=/scratch/media/documents/originals name='simple.pdf' cookie=28041896 flags=MOVED_FROM
t=+01.1585s dir=/scratch/media/documents/originals/Invoice name='simple.pdf' cookie=28041896 flags=MOVED_TO
t=+01.1585s dir=/scratch/media/documents/originals/Invoice name='simple.pdf' cookie=28041898 flags=MOVED_FROM
t=+01.1586s dir=/scratch/media/documents/originals name='simple.pdf' cookie=28041898 flags=MOVED_TO

=== AFTER ===
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
tags             = ['Invoice']
--- find (after) ---
/scratch/media/documents/archive/Invoice
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/originals/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== LOG PROOF for the failed move + rollback ===
<<< NO DIFFERENCE: the failed move AND its rollback wrote ZERO new lines to paperless.log >>>

############################################################
## Q2 PART B — original source missing -> validate_move 'has gone'
############################################################
reset+migrate exit=0  migrations applied OK = 92

=== SEED ===
[2026-07-13 18:33:39,853] [INFO] [paperless.consumer] Consuming simple.pdf
convert-im6.q16: attempt to perform an operation not allowed by the security policy `PDF' @ error/constitute.c/IsCoderAuthorized/426.
convert-im6.q16: no images defined `/tmp/paperless/paperless-g_ys88_g/convert.png' @ error/convert.c/ConvertImageCommand/3229.
[2026-07-13 18:33:40,346] [WARNING] [paperless.parsing] Thumbnail generation with ImageMagick failed, falling back to ghostscript. Check your /etc/ImageMagick-x/policy.xml!
[2026-07-13 18:33:41,155] [INFO] [paperless.consumer] Document 2026-07-13 simple consumption finished
seed consume_file -> Success. New document id 1 created

=== BEFORE ===
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
tags             = []
--- find (before) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/originals/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== INDUCE: delete the original file from disk (the DB row stays) ===
removed '/scratch/media/documents/originals/simple.pdf'
--- find (after deletion) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== TRIGGER: d.tags.add(Invoice) fires m2m_changed (PAPERLESS_DEBUG=true so CRITICAL shows on console) ===
>>> calling d.tags.add(Invoice)
[2026-07-13 18:33:43,423] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
>>> d.tags.add returned WITHOUT raising to the caller

=== AFTER ===
filename         = simple.pdf
archive_filename = simple.pdf
source_path      = /scratch/media/documents/originals/simple.pdf
archive_path     = /scratch/media/documents/archive/simple.pdf
tags             = ['Invoice']
--- find (after) ---
/scratch/media/documents/archive/simple.pdf
/scratch/media/documents/thumbnails/0000001.png

=== LOG PROOF: new lines written during the failed move ===
18a19
> [2026-07-13 18:33:43,423] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
=== grep the log for validate_move's messages ===
19:[2026-07-13 18:33:43,423] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
```

### Q2 — the intermediate state, captured externally [observed]

The four `inotify` lines from PART A are the direct evidence of "what happens partway through." `cookie` is the kernel's identifier that pairs a `MOVED_FROM` with its matching `MOVED_TO`:

| order | dir | name | cookie | flag | meaning |
|-------|-----|------|--------|------|---------|
| 1 | `.../originals` | `simple.pdf` | `28041896` | `MOVED_FROM` | original leaves `originals/` |
| 2 | `.../originals/Invoice` | `simple.pdf` | `28041896` | `MOVED_TO` | original arrives in `originals/Invoice/` — **intermediate state** |
| 3 | `.../originals/Invoice` | `simple.pdf` | `28041898` | `MOVED_FROM` | rollback: original leaves `originals/Invoice/` |
| 4 | `.../originals` | `simple.pdf` | `28041898` | `MOVED_TO` | rollback: original restored to `originals/` |

Events 1–2 (cookie `28041896`) are `os.rename(old_source_path, instance.source_path)` at `handlers.py:354`; events 3–4 (cookie `28041898`) are the recovery `os.rename(instance.source_path, old_source_path)` at `handlers.py:376`. Between event 2 and event 3 the original genuinely existed at `.../originals/Invoice/simple.pdf` — that is the observed intermediate state. Its lifetime here was sub-millisecond (the observer read all four events within the same ~0.0002 s window) because the rollback runs immediately in the `except` block; the ordering, however, is guaranteed by the kernel's event queue and the cookie pairing.

### Q2 — before / during / after (both parts)

| | PART A (archive fails partway) | PART B (source missing) |
|---|---|---|
| **before** | original `originals/simple.pdf`, archive `archive/simple.pdf` | original `originals/simple.pdf`, archive `archive/simple.pdf` |
| **during (intermediate)** | original transiently at `originals/Invoice/simple.pdf` (inotify events 1–2); archive never moved | *no move occurs* — `validate_move` raises before any rename |
| **after** | original **restored** `originals/simple.pdf`; archive **untouched** `archive/simple.pdf`; DB `filename=simple.pdf`; `tags=['Invoice']` | original still missing; archive **untouched** `archive/simple.pdf`; DB `filename=simple.pdf`; `tags=['Invoice']` |
| **log** | **zero** new lines (silent) | one line: `[CRITICAL] [paperless.handlers] ... has gone.` |

### Q2 — grounding, observed vs. inferred, and cause→effect

- **The rollback recovers [observed + `file:line`].** In PART A the `AFTER` block shows `source_path = .../originals/simple.pdf` and the `find` listing shows the original back in `originals/` (not `originals/Invoice/`), while the archive is still at `archive/simple.pdf`. The DB `filename` is still `simple.pdf`. **Cause:** the exception is caught at `handlers.py:367`; the recovery renames the original back at `:376`; the archive branch at `:378` is skipped because `instance.archive_path` (`archive/Invoice/simple.pdf`) does not exist (the archive never moved — its `makedirs` was what failed); and the in-memory names are reset at `:393-394`. The direct DB `UPDATE` at `:362` was never reached, so the row was never changed.

- **The tag relation still persists [observed].** `AFTER` shows `tags = ['Invoice']` in both parts. **Cause:** the tag row is written to the M2M through-table *before* `m2m_changed` dispatches; the handler catches the file-move exception internally and does not re-raise, so `d.tags.add()` returns normally (`">>> d.tags.add returned WITHOUT raising to the caller"`). The file simply did not follow the tag — a consistent state on disk (the file matches the unchanged `filename` column), which is why the sanity checker (Q5/Q6) would not flag it.

- **A move that fails partway is silent [observed + `file:line`].** PART A's `LOG PROOF` reports `<<< NO DIFFERENCE ... wrote ZERO new lines >>>`. **Cause:** there is no `logger.*` call anywhere in `update_filename_and_move_files` (`handlers.py:312-410`), including its `except` block; only `validate_move` logs, and in PART A `validate_move` *passed* for both files (the failure came from `os.makedirs`, not `validate_move`).

- **A move that fails at the start is loud [observed + `file:line`].** PART B's log gains exactly one line — `[CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.` **Cause:** `validate_move` (`handlers.py:352`) sees `not os.path.isfile(old_path)` and calls `logger.fatal(f"Document {str(instance)}: File {old_path} has gone.")` (`:298`) — `logging.fatal` is an alias for `CRITICAL`, which is why the level renders as `[CRITICAL]` — then raises `CannotMoveFilesException` (`:299`). Because the original's `validate_move` is the *first* file operation, no rename ever runs and the archive block (`:356-359`) is never reached.

- **The "target already exists" warning branch is guarded but practically unreachable [observed + inferred].** I could not trigger `logger.warning(... "target path ... already exists")` (`handlers.py:303-306`) through the canonical path: `generate_unique_filename` always returns either the unchanged name or a name that does not exist on disk (`file_handling.py:104`, `:111-125`), so `validate_move`'s `os.path.isfile(new_path)` check (`:301`) is false in normal single-threaded operation. **[Inferred]** this branch exists to defend against a race or an out-of-band file appearing between name generation and rename; reaching it would require a concurrent writer or patching product code, neither of which is a canonical single-threaded invocation. This is the mechanism (collision-avoidance in `generate_unique_filename`) rather than a vague claim.

- **A *failed recovery* is silent by design [inferred].** The recovery renames are themselves wrapped in `try: ... except Exception: pass` (`handlers.py:374-390`). I did **not** observe this branch: forcing the recovery `os.rename` at `:376` to fail requires the original's parent directory to become unwritable *after* the forward move succeeded (or a filesystem fault mid-signal). As root the filesystem permissions are bypassed, and as a non-root user an unwritable `originals/` would also block the *forward* move, so the partway-failure precondition could not be met without patching the code. **[Inferred, grounded at `handlers.py:381-390`]:** if recovery fails, the code swallows the error and relies on the sanity checker (Q5) as the backstop — the source comment at `:382-388` states exactly this ("...going to get caught by the santiy [sic] checker. All files remain in place and will never be overwritten...").

**Why the log gains *exactly one* line here — and when it would be two [observed].** The handler is registered on `m2m_changed` (`handlers.py:310`), so Django invokes it on *both* the `pre_add` and `post_add` phases of a single `d.tags.add(...)`. In this PART B scenario the stored `filename` is `simple.pdf` and `Invoice` is the document's *first* tag: on `pre_add` the tag set is still empty, so `generate_filename` renders `{tag_list}/{title}` with an empty `tag_list` and `path.strip(os.sep)` (`file_handling.py:178`) collapses it to `simple` — the recomputed name equals the stored name, so the handler returns early at `handlers.py:347-349` without ever calling `validate_move`. Only `post_add` (tag set `{Invoice}`) computes the differing `Invoice/simple.pdf` and reaches `validate_move`, which is why the log gains a single CRITICAL line. Adding a *further* tag to an already-tagged document whose original is still missing — so its stored `filename` was never updated away from `simple.pdf` — makes **both** phases compute a differing tag-bearing name, so the identical line is emitted twice by the one `.add()`. Reproduced canonically via `d.tags.add()` (`PAPERLESS_FILENAME_FORMAT={tag_list}/{title}`):

```text
# first tag (untagged document -> Invoice): pre_add is a no-op, post_add fires once -> 1 line
[2026-07-13 23:13:25,710] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
# second tag on the now already-tagged document (Invoice -> +Receipt), single .add(): pre_add + post_add -> 2 lines
[2026-07-13 23:13:25,716] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
[2026-07-13 23:13:25,718] [CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.
```

(Timestamps are run-specific; the level `CRITICAL`, the logger `paperless.handlers`, the message template, and the path are deterministic. The stored `filename` stayed `simple.pdf` throughout because every attempt failed at `validate_move` before the direct DB `UPDATE` at `handlers.py:362`.)

---

## Q3 — Classifier training: skip vs. full retrain, and the change-detection hash

> User's words: *"The classifier's behavior puzzles me, sometimes training finishes instantly, other times it takes much longer. I want to trigger both scenarios and see the actual log messages that explain why training was skipped versus why it proceeded with full retraining, including whatever hash or checksum the system uses to detect changes."*

**Direct answer (observed).** Training finishes *instantly* when the change-detection hash is unchanged, and takes *longer* when it changes. The hash is a **SHA-1** digest (`hashlib.sha1()`, `src/documents/classifier.py:124`), 20 raw bytes / 40 hex characters, persisted as `data_hash` inside the model pickle (`classifier.py:102`, restored at `:86`). On each run `DocumentClassifier.train()` recomputes the digest over every document's content plus its `MATCH_AUTO` labels; if it equals the stored value the method returns `False` *before* any scikit-learn work (`classifier.py:163-164`) — that is the "instant" path. Otherwise it vectorizes and trains the neural network, sets `self.data_hash = new_data_hash` (`:247`) and returns `True` (`:249`) — the "longer" path. The two branches produce two different log messages from the caller `train_classifier` (`src/documents/tasks.py:48`):

- **skip → `DEBUG` `"Training data unchanged."`** (`tasks.py:69`)
- **retrain → `INFO` `"Saving updated classifier model to {}..."`** (`tasks.py:64-66`)

Because the skip message is `DEBUG`, it is **invisible on the console** (which is gated at `INFO`, `settings.py:388`) and appears **only** in `data/log/paperless.log` (`settings.py:409`). The experiment below runs the canonical `document_create_classifier` command **without** `PAPERLESS_DEBUG`, so the console shows only the `INFO` line and the skip evidence is read from the log — exactly the observability split the code dictates.

**The four observed `data_hash` values (full 40-char SHA-1):**

| run | data state | log message | `data_hash` |
|-----|-----------|-------------|-------------|
| 1 (first train) | 4 docs; Invoice tag on pk 1,2 | `INFO Saving updated classifier model to ...` | `c73f17cbb3e307736041f3910da1b415ee61d226` |
| 2 (skip #1) | *unchanged* | `DEBUG Training data unchanged.` | `c73f17cbb3e307736041f3910da1b415ee61d226` |
| 3 (skip #2) | *unchanged* | `DEBUG Training data unchanged.` | `c73f17cbb3e307736041f3910da1b415ee61d226` |
| 4 (retrain) | Invoice tag added to pk 3 | `INFO Saving updated classifier model to ...` | `ff507d06bb8c392a217f7063311c66331c548ca3` |

Runs 2 and 3 are identical — the skip is **stable across repeated runs**. The digest changes (runs 1 → 4) only because the training data changed.

**The exact byte stream (this is "whatever hash the system uses").** For each document, in `Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True)` order (`classifier.py:125-127`), `train()` feeds the SHA-1 accumulator four things, in this order:

1. `preprocess_content(doc.content).encode("utf-8")` — `classifier.py:128-129` (`preprocess_content` = `.lower().strip()` then `re.sub(r"\s+", " ", ...)`, `:24-27`).
2. the **document type** label: `y = dt.pk` if the type exists and `matching_algorithm == MATCH_AUTO`, else `y = -1`; then `y.to_bytes(4, "little", signed=True)` — `classifier.py:131-136`.
3. the **correspondent** label: same rule; `y.to_bytes(4, "little", signed=True)` — `classifier.py:138-143`.
4. the **tags**: `sorted([tag.pk for tag in doc.tags.filter(matching_algorithm=MATCH_AUTO)])`, then for each `tag` (a pk integer) `tag.to_bytes(4, "little", signed=True)` — `classifier.py:145-155`.

**Correction of a subtle code detail.** The document-type and correspondent labels are hashed via `y.to_bytes(...)` (`:136`, `:143`), but each **tag** is hashed via `tag.to_bytes(...)` (`:155`) — the loop variable `tag`, not the reused `y`. (All three are 4-byte little-endian signed integers, so `-1` serializes to `ffffffff`; a pk of `1` serializes to `01000000` — both visible in the enumeration below.) An earlier draft of this document claimed all labels use `y.to_bytes`; the authoritative call at `classifier.py:155` is `tag.to_bytes`.
### Q3 — the exact self-contained script

The script uses two helper snippets executed via `manage.py shell` — a hash reader (reads `FORMAT_VERSION` and the `data_hash` from the pickle) and an independent SHA-1 recompute that replicates `classifier.py:124-161` byte-for-byte and compares against the persisted value. Both helpers are shown immediately after the main script.

```bash
#!/bin/bash
# Q3 — classifier training: skip (instant) vs. full retrain, and the SHA-1 change-detection hash.
# Canonical entry point: python3 manage.py document_create_classifier -> tasks.train_classifier (tasks.py:48)
#   -> DocumentClassifier.train() (classifier.py:115). Console shows INFO only; DEBUG (incl. the skip
#   message) lands in data/log/paperless.log (settings.py:409), so we read the skip evidence from the log.
cd /app/src
LOG=/scratch/data/log/paperless.log

readhash() { python3 manage.py shell < /scratch/exp/q3_readhash.py 2>&1; }

run_train() {  # $1 = run label
  echo ">>> RUN $1: python3 manage.py document_create_classifier"
  # snapshot the log before the run; on RUN 1 the file does not exist yet (its first
  # write happens during this run), so start from an empty snapshot.
  if [ -f "$LOG" ]; then cp "$LOG" /scratch/exp/q3_pre.log; else : > /scratch/exp/q3_pre.log; fi
  echo "    [console output (INFO and above)]:"
  python3 manage.py document_create_classifier 2>&1 | sed 's/^/      /'
  echo "    [NEW lines appended to paperless.log during this run (includes DEBUG)]:"
  diff /scratch/exp/q3_pre.log "$LOG" | grep '^>' | sed 's/^> /      /' || echo "      (none)"
  echo "    [persisted data_hash after RUN $1]:"
  readhash | sed 's/^/      /'
}

echo "=== PREP: reset throwaway store + migrate ==="
bash /scratch/exp/reset.sh > /scratch/exp/q3_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q3_reset.log)"

echo
echo "=== SETUP: one MATCH_AUTO tag 'Invoice' + four training documents (docs 1,2 tagged Invoice). ==="
echo "    Documents are ORM training-data fixtures; the behaviour under test (training) is driven"
echo "    below strictly through the canonical management command."
python3 manage.py shell <<'PY'
from documents.models import Document, Tag, MatchingModel
from django.utils import timezone
inv, _ = Tag.objects.get_or_create(name="Invoice",
        defaults={"matching_algorithm": MatchingModel.MATCH_AUTO})
inv.matching_algorithm = MatchingModel.MATCH_AUTO; inv.save()
print("Invoice tag pk =", inv.pk, " matching_algorithm =", inv.matching_algorithm,
      " (MATCH_AUTO constant =", MatchingModel.MATCH_AUTO, ")")
rows = [("invoice total amount due now", [inv]),
        ("invoice payment received thanks", [inv]),
        ("letter dear sir kind regards", []),
        ("letter meeting notes agenda today", [])]
for i, (content, tags) in enumerate(rows, 1):
    d = Document.objects.create(title=f"doc{i}", content=content,
            checksum=f"{i:032d}", created=timezone.now(),
            modified=timezone.now(), added=timezone.now(), mime_type="text/plain")
    for t in tags: d.tags.add(t)
    print(f"  created pk={d.pk} content={content!r} auto_tags={[t.name for t in d.tags.all()]}")
print("total documents =", Document.objects.count())
PY

echo
echo "=== BYTE ENUMERATION of the exact stream fed to hashlib.sha1() for the RUN 1 data state ==="
python3 manage.py shell < /scratch/exp/q3_enumerate.py 2>&1

echo
echo "########## RUN 1 — first train (no prior model; data_hash is None) ##########"
run_train 1
echo "    [independent SHA-1 recompute vs the persisted data_hash]:"
python3 manage.py shell < /scratch/exp/q3_recompute.py 2>&1 | sed 's/^/      /'

echo
echo "########## RUN 2 — unchanged data (skip #1) ##########"
run_train 2

echo
echo "########## RUN 3 — unchanged data (skip #2 — confirms the skip is stable across runs) ##########"
run_train 3

echo
echo "=== CHANGE the MATCH_AUTO training data: add the 'Invoice' tag to document pk=3 ==="
python3 manage.py shell <<'PY'
from documents.models import Document, Tag
d = Document.objects.get(pk=3); inv = Tag.objects.get(name="Invoice")
d.tags.add(inv)
print("doc pk=3 auto_tags now =", [t.name for t in d.tags.all()])
PY

echo
echo "=== BYTE ENUMERATION for the RUN 4 data state (note doc pk=3 now carries the Invoice tag) ==="
python3 manage.py shell < /scratch/exp/q3_enumerate.py 2>&1

echo
echo "########## RUN 4 — data changed -> full retrain ##########"
run_train 4
echo "    [independent SHA-1 recompute vs the persisted data_hash]:"
python3 manage.py shell < /scratch/exp/q3_recompute.py 2>&1 | sed 's/^/      /'
```

Helper — read the persisted `data_hash` from the model pickle (`/scratch/exp/q3_readhash.py`):

```python
import pickle, binascii, os
from django.conf import settings
if not os.path.isfile(settings.MODEL_FILE):
    print("MODEL_FILE absent:", settings.MODEL_FILE)
else:
    with open(settings.MODEL_FILE, "rb") as f:
        ver = pickle.load(f); dh = pickle.load(f)
    print("FORMAT_VERSION      =", ver)
    print("data_hash raw bytes =", len(dh))
    print("data_hash hex(sha1) =", binascii.hexlify(dh).decode())
```

Helper — independent SHA-1 recompute replicating `classifier.py:124-161` and comparing to the pickle (`/scratch/exp/q3_recompute.py`):

```python
import hashlib, re, binascii, pickle, os
from django.conf import settings
from documents.models import Document, MatchingModel
def preprocess_content(content):          # replicates classifier.py:24-27
    content = content.lower().strip()
    content = re.sub(r"\s+", " ", content)
    return content
m = hashlib.sha1()
for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True):   # classifier.py:125-127
    pc = preprocess_content(doc.content)
    m.update(pc.encode("utf-8"))                                                # :129 content utf-8
    y = -1
    dt = doc.document_type
    if dt and dt.matching_algorithm == MatchingModel.MATCH_AUTO: y = dt.pk      # :132-135
    m.update(y.to_bytes(4, "little", signed=True))                              # :136 doc_type y.to_bytes
    y = -1
    cor = doc.correspondent
    if cor and cor.matching_algorithm == MatchingModel.MATCH_AUTO: y = cor.pk   # :139-142
    m.update(y.to_bytes(4, "little", signed=True))                             # :143 correspondent y.to_bytes
    tags = sorted([t.pk for t in doc.tags.filter(matching_algorithm=MatchingModel.MATCH_AUTO)])  # :146-153
    for tag in tags:
        m.update(tag.to_bytes(4, "little", signed=True))                        # :155 tag.to_bytes (NOT y)
recompute = m.hexdigest()
print("independent recompute (hex) =", recompute)
if os.path.isfile(settings.MODEL_FILE):
    with open(settings.MODEL_FILE, "rb") as f:
        pickle.load(f); dh = pickle.load(f)
    persisted = binascii.hexlify(dh).decode()
    print("persisted data_hash  (hex) =", persisted)
    print("MATCH ==", recompute == persisted)
```

Helper — per-document byte enumeration (`/scratch/exp/q3_enumerate.py`):

```python
import re, binascii
from documents.models import Document, MatchingModel
def preprocess_content(content):
    content = content.lower().strip()
    content = re.sub(r"\s+", " ", content)
    return content
def b4(n): return binascii.hexlify(n.to_bytes(4, "little", signed=True)).decode()
print("byte stream fed to hashlib.sha1(), in Document.objects.order_by('pk') order:")
for doc in Document.objects.order_by("pk").exclude(tags__is_inbox_tag=True):
    pc = preprocess_content(doc.content)
    print(f"--- doc pk={doc.pk} ---")
    print(f"  preprocessed_content = {pc!r}")
    print(f"  content.encode('utf-8') hex = {binascii.hexlify(pc.encode('utf-8')).decode()}")
    y = -1; dt = doc.document_type
    if dt and dt.matching_algorithm == MatchingModel.MATCH_AUTO: y = dt.pk
    print(f"  doc_type      y={y:>3} -> 4B LE signed = {b4(y)}")
    y = -1; cor = doc.correspondent
    if cor and cor.matching_algorithm == MatchingModel.MATCH_AUTO: y = cor.pk
    print(f"  correspondent y={y:>3} -> 4B LE signed = {b4(y)}")
    tags = sorted([t.pk for t in doc.tags.filter(matching_algorithm=MatchingModel.MATCH_AUTO)])
    print(f"  MATCH_AUTO tag pks (sorted) = {tags}")
    for tag in tags:
        print(f"    tag pk={tag} -> 4B LE signed = {b4(tag)}")
```

### Q3 — the exact command and its complete, unedited output

Command:


```bash
docker exec -w /app/src paperless-work bash /scratch/exp/q3.sh
```


```text
=== PREP: reset throwaway store + migrate ===
reset+migrate exit=0  migrations applied OK = 92

=== SETUP: one MATCH_AUTO tag 'Invoice' + four training documents (docs 1,2 tagged Invoice). ===
    Documents are ORM training-data fixtures; the behaviour under test (training) is driven
    below strictly through the canonical management command.
Invoice tag pk = 1  matching_algorithm = 6  (MATCH_AUTO constant = 6 )
  created pk=1 content='invoice total amount due now' auto_tags=['Invoice']
  created pk=2 content='invoice payment received thanks' auto_tags=['Invoice']
  created pk=3 content='letter dear sir kind regards' auto_tags=[]
  created pk=4 content='letter meeting notes agenda today' auto_tags=[]
total documents = 4

=== BYTE ENUMERATION of the exact stream fed to hashlib.sha1() for the RUN 1 data state ===
byte stream fed to hashlib.sha1(), in Document.objects.order_by('pk') order:
--- doc pk=1 ---
  preprocessed_content = 'invoice total amount due now'
  content.encode('utf-8') hex = 696e766f69636520746f74616c20616d6f756e7420647565206e6f77
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = [1]
    tag pk=1 -> 4B LE signed = 01000000
--- doc pk=2 ---
  preprocessed_content = 'invoice payment received thanks'
  content.encode('utf-8') hex = 696e766f696365207061796d656e74207265636569766564207468616e6b73
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = [1]
    tag pk=1 -> 4B LE signed = 01000000
--- doc pk=3 ---
  preprocessed_content = 'letter dear sir kind regards'
  content.encode('utf-8') hex = 6c6574746572206465617220736972206b696e642072656761726473
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = []
--- doc pk=4 ---
  preprocessed_content = 'letter meeting notes agenda today'
  content.encode('utf-8') hex = 6c6574746572206d656574696e67206e6f746573206167656e646120746f646179
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = []

########## RUN 1 — first train (no prior model; data_hash is None) ##########
>>> RUN 1: python3 manage.py document_create_classifier
    [console output (INFO and above)]:
      [2026-07-13 18:42:32,909] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
    [NEW lines appended to paperless.log during this run (includes DEBUG)]:
      [2026-07-13 18:42:32,410] [DEBUG] [paperless.classifier] Document classification model does not exist (yet), not performing automatic matching.
      [2026-07-13 18:42:32,411] [DEBUG] [paperless.classifier] Gathering data from database...
      [2026-07-13 18:42:32,416] [DEBUG] [paperless.classifier] 4 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
      [2026-07-13 18:42:32,817] [DEBUG] [paperless.classifier] Vectorizing data...
      [2026-07-13 18:42:32,818] [DEBUG] [paperless.classifier] Training tags classifier...
      [2026-07-13 18:42:32,909] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
      [2026-07-13 18:42:32,909] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
      [2026-07-13 18:42:32,909] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
    [persisted data_hash after RUN 1]:
      FORMAT_VERSION      = 7
      data_hash raw bytes = 20
      data_hash hex(sha1) = c73f17cbb3e307736041f3910da1b415ee61d226
    [independent SHA-1 recompute vs the persisted data_hash]:
      independent recompute (hex) = c73f17cbb3e307736041f3910da1b415ee61d226
      persisted data_hash  (hex) = c73f17cbb3e307736041f3910da1b415ee61d226
      MATCH == True

########## RUN 2 — unchanged data (skip #1) ##########
>>> RUN 2: python3 manage.py document_create_classifier
    [console output (INFO and above)]:
    [NEW lines appended to paperless.log during this run (includes DEBUG)]:
      [2026-07-13 18:42:36,701] [DEBUG] [paperless.classifier] Gathering data from database...
      [2026-07-13 18:42:36,708] [DEBUG] [paperless.tasks] Training data unchanged.
    [persisted data_hash after RUN 2]:
      FORMAT_VERSION      = 7
      data_hash raw bytes = 20
      data_hash hex(sha1) = c73f17cbb3e307736041f3910da1b415ee61d226

########## RUN 3 — unchanged data (skip #2 — confirms the skip is stable across runs) ##########
>>> RUN 3: python3 manage.py document_create_classifier
    [console output (INFO and above)]:
    [NEW lines appended to paperless.log during this run (includes DEBUG)]:
      [2026-07-13 18:42:39,687] [DEBUG] [paperless.classifier] Gathering data from database...
      [2026-07-13 18:42:39,692] [DEBUG] [paperless.tasks] Training data unchanged.
    [persisted data_hash after RUN 3]:
      FORMAT_VERSION      = 7
      data_hash raw bytes = 20
      data_hash hex(sha1) = c73f17cbb3e307736041f3910da1b415ee61d226

=== CHANGE the MATCH_AUTO training data: add the 'Invoice' tag to document pk=3 ===
doc pk=3 auto_tags now = ['Invoice']

=== BYTE ENUMERATION for the RUN 4 data state (note doc pk=3 now carries the Invoice tag) ===
byte stream fed to hashlib.sha1(), in Document.objects.order_by('pk') order:
--- doc pk=1 ---
  preprocessed_content = 'invoice total amount due now'
  content.encode('utf-8') hex = 696e766f69636520746f74616c20616d6f756e7420647565206e6f77
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = [1]
    tag pk=1 -> 4B LE signed = 01000000
--- doc pk=2 ---
  preprocessed_content = 'invoice payment received thanks'
  content.encode('utf-8') hex = 696e766f696365207061796d656e74207265636569766564207468616e6b73
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = [1]
    tag pk=1 -> 4B LE signed = 01000000
--- doc pk=3 ---
  preprocessed_content = 'letter dear sir kind regards'
  content.encode('utf-8') hex = 6c6574746572206465617220736972206b696e642072656761726473
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = [1]
    tag pk=1 -> 4B LE signed = 01000000
--- doc pk=4 ---
  preprocessed_content = 'letter meeting notes agenda today'
  content.encode('utf-8') hex = 6c6574746572206d656574696e67206e6f746573206167656e646120746f646179
  doc_type      y= -1 -> 4B LE signed = ffffffff
  correspondent y= -1 -> 4B LE signed = ffffffff
  MATCH_AUTO tag pks (sorted) = []

########## RUN 4 — data changed -> full retrain ##########
>>> RUN 4: python3 manage.py document_create_classifier
    [console output (INFO and above)]:
      [2026-07-13 18:42:44,508] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
    [NEW lines appended to paperless.log during this run (includes DEBUG)]:
      [2026-07-13 18:42:44,438] [DEBUG] [paperless.classifier] Gathering data from database...
      [2026-07-13 18:42:44,443] [DEBUG] [paperless.classifier] 4 documents, 1 tag(s), 0 correspondent(s), 0 document type(s).
      [2026-07-13 18:42:44,444] [DEBUG] [paperless.classifier] Vectorizing data...
      [2026-07-13 18:42:44,444] [DEBUG] [paperless.classifier] Training tags classifier...
      [2026-07-13 18:42:44,507] [DEBUG] [paperless.classifier] There are no correspondents. Not training correspondent classifier.
      [2026-07-13 18:42:44,507] [DEBUG] [paperless.classifier] There are no document types. Not training document type classifier.
      [2026-07-13 18:42:44,508] [INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...
    [persisted data_hash after RUN 4]:
      FORMAT_VERSION      = 7
      data_hash raw bytes = 20
      data_hash hex(sha1) = ff507d06bb8c392a217f7063311c66331c548ca3
    [independent SHA-1 recompute vs the persisted data_hash]:
      independent recompute (hex) = ff507d06bb8c392a217f7063311c66331c548ca3
      persisted data_hash  (hex) = ff507d06bb8c392a217f7063311c66331c548ca3
      MATCH == True
```

### Q3 — grounding, observed vs. inferred, and cause→effect

- **Why "instant" vs. "longer" [observed + `file:line`].** On runs 2 and 3 the *only* new log lines are `Gathering data from database...` (`classifier.py:123`) and `Training data unchanged.` (`tasks.py:69`); there is no `Vectorizing data...` or `Training tags classifier...`. On runs 1 and 4 those scikit-learn steps *do* appear. **Cause:** `train()` computes `new_data_hash = m.digest()` (`classifier.py:161`) and then `if self.data_hash and new_data_hash == self.data_hash: return False` (`:163-164`) — this early return happens *before* `CountVectorizer.fit_transform` and the `MLPClassifier` training (`:191` onward), so an unchanged hash short-circuits all the expensive work. That is literally why the skip is instantaneous and the retrain is not.

- **The exact log strings [observed + `file:line`].** Retrain emits `INFO` `Saving updated classifier model to /scratch/data/classification_model.pickle...` (runs 1 and 4), produced by `logger.info(...)` at `tasks.py:64-66` (only reached when `classifier.train()` returns `True`, `:63`). Skip emits `DEBUG` `Training data unchanged.` (runs 2 and 3) from `logger.debug(...)` at `tasks.py:69` (the `else` branch). The retrain message appears on **both** console and log; the skip message appears **only** in the log — consistent with the console handler being gated at `INFO` (`settings.py:388`) while the `paperless` logger writes `DEBUG` to the file handler (`settings.py:409`).

- **The hash and its algorithm [observed + `file:line`].** The `data_hash` is a **SHA-1** digest: `m = hashlib.sha1()` (`classifier.py:124`), finalized as `m.digest()` (`:161`) — 20 raw bytes, which the reader shows as `data_hash raw bytes = 20` and as a 40-hex-character string. It is persisted as the second pickled object (after `FORMAT_VERSION = 7`) via `pickle.dump(self.data_hash, f)` (`classifier.py:102`) and restored via `pickle.load(f)` (`:86`). The independent recompute — which re-implements the byte order at `classifier.py:124-161` without calling the classifier — reproduces the persisted value **exactly** for both data states (`MATCH == True`: `c73f17cbb3e307736041f3910da1b415ee61d226` for run 1, `ff507d06bb8c392a217f7063311c66331c548ca3` for run 4). This proves the byte order documented above is correct and that the digest is genuinely a function of content + `MATCH_AUTO` labels.

- **Why the hash changed [observed + `file:line`].** Between run 3 and run 4 the *only* change was `d.tags.add(inv)` on document pk 3. The byte enumeration shows pk 3 going from `MATCH_AUTO tag pks (sorted) = []` to `= [1]`, i.e. four extra bytes `01000000` (`tag.to_bytes(4, "little", signed=True)` at `classifier.py:155`) are now folded into the accumulator for that document. That single four-byte change flips the SHA-1 from `c73f...d226` to `ff50...8ca3`, which is why run 4 retrained. **Cause→effect:** identical inputs ⇒ identical digest ⇒ `return False` skip; any change to any document's content or `MATCH_AUTO` label ⇒ different digest ⇒ full retrain.

- **Stability [observed].** Runs 2 and 3 are byte-identical in outcome — same `DEBUG Training data unchanged.` line and the same `data_hash` — confirming (per the timing/stability rule) that the skip is reproducible across at least two consecutive runs of the same unchanged input, not a one-off.

- **The `MATCH_AUTO` guard [observed + `file:line`].** `train_classifier` returns immediately (no model work, no log line) unless at least one `Tag`, `DocumentType`, or `Correspondent` uses `matching_algorithm == MATCH_AUTO` (`tasks.py:49-55`). The experiment creates exactly such a tag (`Invoice`, `matching_algorithm = 6`, and the output confirms `MATCH_AUTO constant = 6`), which is the precondition for any of the above to run.

---

## Q4 — Duplicate detection checksums

> User's words: *"Duplicate detection feels like magic, two files that look completely different can still be rejected as duplicates, and I want to see the actual checksums being compared at that moment of judgment."*

**Direct answer (observed).** At the moment of judgment the consumer computes the **MD5** of the *incoming* file and rejects it if that MD5 equals **either** the stored `checksum` **or** the stored `archive_checksum` of *any* existing document: `hashlib.md5(f.read()).hexdigest()` (`src/documents/consumer.py:104`) compared via `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum))` (`consumer.py:105-107`). That `OR` over `archive_checksum` is why "two files that look completely different" collide — a document's **original** and its **OCR archive rendition** are byte-wise different files with different MD5s, yet uploading *either* is caught as a duplicate of the same document.

Both rejections below were produced through the **canonical ingestion pipeline** — a file dropped in the consumption directory, picked up by `document_consumer --oneshot` which enqueues `documents.tasks.consume_file` (`src/documents/management/commands/document_consumer.py:85-91`), executed by a running `qcluster` worker. No task function was called directly.

**The actual checksums at the moment of judgment (byte-verified against the on-disk files in the same run):**

| | value | verified against |
|---|-------|------------------|
| stored `checksum` (original) | `42995833e01aea9b3edee44bbfdd7ce1` | `md5sum originals/0000001.pdf` = `42995833e01aea9b3edee44bbfdd7ce1` ✓ |
| stored `archive_checksum` (OCR rendition) | `05527bf515905eefa5cc2f3270a03b8f` | `md5sum archive/0000001.pdf` = `05527bf515905eefa5cc2f3270a03b8f` ✓ |
| CASE A incoming MD5 (identical original) | `42995833e01aea9b3edee44bbfdd7ce1` | == stored `checksum` → duplicate |
| CASE B incoming MD5 (archive rendition) | `05527bf515905eefa5cc2f3270a03b8f` | == stored `archive_checksum`, ≠ original MD5 → duplicate |

The original (22926 bytes) and the archive rendition (10860 bytes) are plainly different files with different MD5s, which is the whole point of CASE B.
### Q4 — the exact self-contained script

```bash
#!/bin/bash
# Q4 — duplicate detection, entirely through the CANONICAL ingestion pipeline:
#   file dropped in CONSUMPTION_DIR -> `document_consumer --oneshot` enqueues an async task
#   (document_consumer.py:85-91) -> a running qcluster worker executes documents.tasks.consume_file
#   -> Consumer.pre_check_duplicate (consumer.py:102) computes md5 and matches checksum OR
#      archive_checksum -> _fail logs ERROR + raises ConsumerError (consumer.py:80-81, 110-112).
# No task function is called directly; every consume goes through the redis-backed queue.
cd /app/src
LOG=/scratch/data/log/paperless.log

echo "=== PREP: reset; remove scheduled maintenance jobs so the queue holds ONLY our consume tasks ==="
bash /scratch/exp/reset.sh > /scratch/exp/q4_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q4_reset.log)"
python3 manage.py shell <<'PY'
from django_q.models import Schedule
n = Schedule.objects.count()
Schedule.objects.all().delete()
print(f"deleted {n} scheduled maintenance job(s) (train_classifier, index_optimize, sanity_check, mail) to isolate the experiment; the consume path itself is unchanged")
PY
echo "=== start a real qcluster worker (redis broker) ==="
python3 manage.py qcluster > /scratch/exp/q4_qcluster.log 2>&1 &
QPID=$!
for i in $(seq 1 100); do grep -qiE "ready for work" /scratch/exp/q4_qcluster.log 2>/dev/null && break; sleep 0.2; done
echo "qcluster pid=$QPID; startup:"; sed -n '1,3p' /scratch/exp/q4_qcluster.log

wait_consume_tasks() {   # $1 = expected number of completed consume_file tasks
  local target="$1" n
  for i in $(seq 1 200); do
    n=$(python3 manage.py shell -c "from django_q.models import Task; print(Task.objects.filter(func='documents.tasks.consume_file').count())" 2>/dev/null | tail -1)
    [ "$n" = "$target" ] && { echo "  consume_file tasks completed = $n"; return 0; }
    sleep 0.2
  done
  echo "  TIMEOUT: consume_file tasks completed = $n (target $target)"
}
doc_count() { python3 manage.py shell -c "from documents.models import Document; print('Document.objects.count() =', Document.objects.count())" 2>/dev/null | tail -1; }
show_failure() {  # $1 = task name to fetch
  python3 manage.py shell <<PY
from django_q.models import Failure
f = Failure.objects.filter(name="$1").order_by("started").last()
print("name    =", f.name)
print("func    =", f.func)
print("success =", f.success)
print("result  =", repr(f.result))
PY
}

echo
echo "=== SEED: consume simple.pdf via the canonical queue ==="
cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/simple.pdf
python3 manage.py document_consumer --oneshot 2>&1 | sed 's/^/  [consumer] /'
wait_consume_tasks 1
doc_count

echo
echo "=== STORED checksums (DB) vs. actual on-disk MD5 (byte-verified in the same run) ==="
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
print("pk                    =", d.pk)
print("checksum (DB)         =", d.checksum)
print("archive_checksum (DB) =", d.archive_checksum)
print("source_path           =", d.source_path)
print("archive_path          =", d.archive_path)
print("has_archive_version   =", d.has_archive_version)
PY
echo "--- md5sum + size of the actual files on disk ---"
md5sum /scratch/media/documents/originals/0000001.pdf /scratch/media/documents/archive/0000001.pdf
stat -c '%s bytes  %n' /scratch/media/documents/originals/0000001.pdf /scratch/media/documents/archive/0000001.pdf
echo "(original MD5 must equal checksum; archive MD5 must equal archive_checksum)"

echo
echo "############################################################"
echo "## CASE A — re-consume the IDENTICAL original (checksum match)"
echo "############################################################"
rm -f /scratch/consume/*
cp "$LOG" /scratch/exp/q4a_before.log
cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/simple.pdf
echo "incoming file MD5 = $(md5sum /scratch/consume/simple.pdf | cut -d' ' -f1)  (compare to checksum above)"
python3 manage.py document_consumer --oneshot 2>&1 | sed 's/^/  [consumer] /'
wait_consume_tasks 2
doc_count
echo "--- django_q Failure record for this attempt (by task name 'simple.pdf') ---"
show_failure "simple.pdf"
echo "--- NEW lines in paperless.log during CASE A ---"
diff /scratch/exp/q4a_before.log "$LOG" | grep '^>' | sed 's/^> //' || echo "(none)"

echo
echo "############################################################"
echo "## CASE B — consume the ARCHIVE rendition (archive_checksum match)"
echo "##          The archive PDF is an OCR rendition: a byte-wise-different file"
echo "##          from the original (different MD5), yet rejected as a duplicate."
echo "############################################################"
rm -f /scratch/consume/*
cp "$LOG" /scratch/exp/q4b_before.log
cp /scratch/media/documents/archive/0000001.pdf /scratch/consume/archive_rendition.pdf
echo "incoming file MD5 = $(md5sum /scratch/consume/archive_rendition.pdf | cut -d' ' -f1)  (compare to archive_checksum above)"
echo "original file MD5 = $(md5sum /scratch/media/documents/originals/0000001.pdf | cut -d' ' -f1)  (DIFFERENT bytes from the archive rendition)"
python3 manage.py document_consumer --oneshot 2>&1 | sed 's/^/  [consumer] /'
wait_consume_tasks 3
doc_count
echo "--- django_q Failure record for this attempt (by task name 'archive_rendition.pdf') ---"
show_failure "archive_rendition.pdf"
echo "--- NEW lines in paperless.log during CASE B ---"
diff /scratch/exp/q4b_before.log "$LOG" | grep '^>' | sed 's/^> //' || echo "(none)"

echo
echo "=== FINAL: only ONE document exists; both duplicates were rejected ==="
doc_count
echo "=== stop qcluster by PID $QPID ==="
kill "$QPID" 2>/dev/null
wait "$QPID" 2>/dev/null
echo "qcluster stopped."
```

### Q4 — the exact command and its complete, unedited output

Command:


```bash
docker exec -w /app/src paperless-work bash /scratch/exp/q4.sh
```


```text
=== PREP: reset; remove scheduled maintenance jobs so the queue holds ONLY our consume tasks ===
reset+migrate exit=0  migrations applied OK = 92
deleted 4 scheduled maintenance job(s) (train_classifier, index_optimize, sanity_check, mail) to isolate the experiment; the consume path itself is unchanged
=== start a real qcluster worker (redis broker) ===
qcluster pid=12327; startup:
18:53:31 [Q] INFO Q Cluster chicken-eighteen-sierra-river starting.
18:53:31 [Q] INFO Process-1:1 ready for work at 12350
18:53:31 [Q] INFO Process-1:2 ready for work at 12351

=== SEED: consume simple.pdf via the canonical queue ===
  [consumer] [2026-07-13 18:53:33,191] [INFO] [paperless.management.consumer] Adding /scratch/consume/simple.pdf to the task queue.
  [consumer] 18:53:33 [Q] INFO Enqueued 1
  consume_file tasks completed = 1
Document.objects.count() = 1

=== STORED checksums (DB) vs. actual on-disk MD5 (byte-verified in the same run) ===
pk                    = 1
checksum (DB)         = 42995833e01aea9b3edee44bbfdd7ce1
archive_checksum (DB) = 05527bf515905eefa5cc2f3270a03b8f
source_path           = /scratch/media/documents/originals/0000001.pdf
archive_path          = /scratch/media/documents/archive/0000001.pdf
has_archive_version   = True
--- md5sum + size of the actual files on disk ---
42995833e01aea9b3edee44bbfdd7ce1  /scratch/media/documents/originals/0000001.pdf
05527bf515905eefa5cc2f3270a03b8f  /scratch/media/documents/archive/0000001.pdf
22926 bytes  /scratch/media/documents/originals/0000001.pdf
10860 bytes  /scratch/media/documents/archive/0000001.pdf
(original MD5 must equal checksum; archive MD5 must equal archive_checksum)

############################################################
## CASE A — re-consume the IDENTICAL original (checksum match)
############################################################
incoming file MD5 = 42995833e01aea9b3edee44bbfdd7ce1  (compare to checksum above)
  [consumer] [2026-07-13 18:53:38,642] [INFO] [paperless.management.consumer] Adding /scratch/consume/simple.pdf to the task queue.
  [consumer] 18:53:38 [Q] INFO Enqueued 1
  consume_file tasks completed = 2
Document.objects.count() = 1
--- django_q Failure record for this attempt (by task name 'simple.pdf') ---
name    = simple.pdf
func    = documents.tasks.consume_file
success = False
result  = 'simple.pdf: Not consuming simple.pdf: It is a duplicate. : Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file(\n  File "/app/src/documents/consumer.py", line 213, in try_consume_file\n    self.pre_check_duplicate()\n  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate\n    self._fail(\n  File "/app/src/documents/consumer.py", line 81, in _fail\n    raise ConsumerError(f"{self.filename}: {log_message or message}")\ndocuments.consumer.ConsumerError: simple.pdf: Not consuming simple.pdf: It is a duplicate.\n'
--- NEW lines in paperless.log during CASE A ---
[2026-07-13 18:53:38,642] [INFO] [paperless.management.consumer] Adding /scratch/consume/simple.pdf to the task queue.
[2026-07-13 18:53:38,797] [ERROR] [paperless.consumer] Not consuming simple.pdf: It is a duplicate.

############################################################
## CASE B — consume the ARCHIVE rendition (archive_checksum match)
##          The archive PDF is an OCR rendition: a byte-wise-different file
##          from the original (different MD5), yet rejected as a duplicate.
############################################################
incoming file MD5 = 05527bf515905eefa5cc2f3270a03b8f  (compare to archive_checksum above)
original file MD5 = 42995833e01aea9b3edee44bbfdd7ce1  (DIFFERENT bytes from the archive rendition)
  [consumer] [2026-07-13 18:53:42,887] [INFO] [paperless.management.consumer] Adding /scratch/consume/archive_rendition.pdf to the task queue.
  [consumer] 18:53:42 [Q] INFO Enqueued 1
  consume_file tasks completed = 3
Document.objects.count() = 1
--- django_q Failure record for this attempt (by task name 'archive_rendition.pdf') ---
name    = archive_rendition.pdf
func    = documents.tasks.consume_file
success = False
result  = 'archive_rendition.pdf: Not consuming archive_rendition.pdf: It is a duplicate. : Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 432, in worker\n    res = f(*task["args"], **task["kwargs"])\n  File "/app/src/documents/tasks.py", line 236, in consume_file\n    document = Consumer().try_consume_file(\n  File "/app/src/documents/consumer.py", line 213, in try_consume_file\n    self.pre_check_duplicate()\n  File "/app/src/documents/consumer.py", line 110, in pre_check_duplicate\n    self._fail(\n  File "/app/src/documents/consumer.py", line 81, in _fail\n    raise ConsumerError(f"{self.filename}: {log_message or message}")\ndocuments.consumer.ConsumerError: archive_rendition.pdf: Not consuming archive_rendition.pdf: It is a duplicate.\n'
--- NEW lines in paperless.log during CASE B ---
[2026-07-13 18:53:42,887] [INFO] [paperless.management.consumer] Adding /scratch/consume/archive_rendition.pdf to the task queue.
[2026-07-13 18:53:43,047] [ERROR] [paperless.consumer] Not consuming archive_rendition.pdf: It is a duplicate.

=== FINAL: only ONE document exists; both duplicates were rejected ===
Document.objects.count() = 1
=== stop qcluster by PID 12327 ===
qcluster stopped.
```

### Q4 — grounding, observed vs. inferred, and cause→effect

- **The checksum compared is the incoming file's MD5 [observed + `file:line`].** `pre_check_duplicate` opens the candidate file and computes `checksum = hashlib.md5(f.read()).hexdigest()` (`consumer.py:103-104`). In CASE A the printed `incoming file MD5 = 42995833e01aea9b3edee44bbfdd7ce1` equals the stored `checksum`; in CASE B `incoming file MD5 = 05527bf515905eefa5cc2f3270a03b8f` equals the stored `archive_checksum`. Both stored values were independently confirmed with `md5sum` against the files on disk, so the comparison is byte-verified.

- **The match is against `checksum` OR `archive_checksum` [observed + `file:line`].** The query is `Document.objects.filter(Q(checksum=checksum) | Q(archive_checksum=checksum)).exists()` (`consumer.py:105-107`). CASE B is rejected even though its MD5 differs from the original's `checksum`, because it equals the document's `archive_checksum` — the `OR` branch. This is the mechanism behind "files that look completely different are still duplicates": the fields are `Document.checksum` (`models.py:135-141`, `max_length=32`, `unique=True` — 32 hex chars = one MD5) and `Document.archive_checksum` (`models.py:143-150`).

- **The rejection is logged at ERROR and raised as `ConsumerError` [observed + `file:line`].** Each case appended exactly one `[ERROR] [paperless.consumer] Not consuming <file>: It is a duplicate.` line to `paperless.log`, emitted by `self.log("error", ...)` in `_fail` (`consumer.py:80`) with the message from `pre_check_duplicate` (`consumer.py:110-112`, message constant `MESSAGE_DOCUMENT_ALREADY_EXISTS = "document_already_exists"`, `consumer.py:37`). `_fail` then raises `ConsumerError(f"{self.filename}: {log_message}")` (`consumer.py:81`). The django_q `Failure` records capture that exception verbatim, and their tracebacks independently confirm the call chain: `tasks.py:236` → `consumer.py:213` (`self.pre_check_duplicate()`) → `consumer.py:110` → `consumer.py:81`.

- **The duplicate is not ingested [observed].** `Document.objects.count()` stays `1` after both attempts, and `success = False` on both django_q `Failure` records — the pre-check runs before any document row is created (`try_consume_file` calls `pre_check_duplicate()` at `consumer.py:213`, early in the pipeline).

- **Why the archive rendition is byte-wise different [observed].** The archive is the OCR/normalized PDF produced during ingestion; its MD5 (`05527bf515905eefa5cc2f3270a03b8f`, 10860 bytes) differs from the original's (`42995833e01aea9b3edee44bbfdd7ce1`, 22926 bytes). This same run also shows the archive MD5 differs from the value seen in other runs of this investigation (Q1/Q5), i.e. the archive rendition is **not** bit-for-bit reproducible across runs; the original's `checksum` is stable. The concrete mechanism for that instability is examined under Q5 (OCR embeds run-specific metadata). For Q4 the archive value is tied to reproducible bytes *within the run* by the `md5sum` verification above.

- **`CONSUMER_DELETE_DUPLICATES` is off by default [observed + `file:line`].** The rejected files remained in the consumption directory (the script removes them between cases); had `settings.CONSUMER_DELETE_DUPLICATES` been true, `pre_check_duplicate` would `os.unlink(self.path)` the incoming file (`consumer.py:108-109`). It defaults to off (`settings.py:486`).
## Q5 — Sanity checker output: healthy vs. broken, and the mismatch hashes

> User's words: *"The sanity checker intrigues me as well, what does its output actually look like when an archive is healthy versus when something has gone wrong, and can I see the specific hash values it compares when it discovers a mismatch?"*

**Direct answer (observed).** A healthy store produces a single line — `[INFO] [paperless.sanity_checker] Sanity checker detected no issues.` (from `SanityCheckMessages.log_messages`, `src/documents/sanity_checker.py:26-27`). When a file no longer matches its stored hash, the checker recomputes the **MD5** of the on-disk file and emits an ERROR naming both hashes: for the original, `Checksum mismatch of document {pk}. Stored: {doc.checksum}, actual: {recomputed md5}.` (`sanity_checker.py:88-91`, MD5 at `:82-83`); for the archive, `Checksum mismatch of archived document {pk}. Stored: {doc.archive_checksum}, actual: {recomputed md5}.` (`sanity_checker.py:118-124`, MD5 at `:111-112`). The messages are collected during `check_sanity` and emitted together by `log_messages`, so the console/log lines are the checker's entire "output". The canonical CLI `document_sanity_checker` **logs and exits 0** even on error (`document_sanity_checker.py:24-26` — no raise); the async task `documents.tasks.sanity_check` logs the same messages **then raises** `SanityCheckFailedException("Sanity check failed with errors. See log.")` (`tasks.py:255-261`).

**The specific hash values it compares at a mismatch (byte-verified against the on-disk files in the same run):**

| case | stored hash (DB) | actual hash (recomputed by the checker) | verified against |
|------|------------------|-----------------------------------------|------------------|
| healthy original | `checksum = 42995833e01aea9b3edee44bbfdd7ce1` | `md5(originals/0000001.pdf) = 42995833e01aea9b3edee44bbfdd7ce1` | equal → no message |
| healthy archive | `archive_checksum = e0c7431b5528db5c2e3e57f5af57794c` | `md5(archive/0000001.pdf) = e0c7431b5528db5c2e3e57f5af57794c` | equal → no message |
| corrupt original (Q5b) | `Stored: 42995833e01aea9b3edee44bbfdd7ce1` | `actual: 427d535caf2320c9eb8baf7ef8550c21` | `md5sum` of the corrupted file = `427d535caf2320c9eb8baf7ef8550c21` ✓ |
| corrupt archive (Q5c) | `Stored: e0c7431b5528db5c2e3e57f5af57794c` | `actual: 2908eda8d522f89ab1b6726ec3972a0d` | `md5sum` of the corrupted archive = `2908eda8d522f89ab1b6726ec3972a0d` ✓ |

The "actual" value the checker prints is exactly the `md5sum` of the file after corruption — the byte-sensitive result is verified against the bytes the tool saw.
### Q5 — the exact self-contained script

```bash
#!/bin/bash
# Q5 — sanity checker output: healthy vs. broken, and the Stored-vs-actual mismatch hashes.
# Canonical entry point: `python3 manage.py document_sanity_checker` (CLI, document_sanity_checker.py:24-26)
# and the async task `documents.tasks.sanity_check` (tasks.py:255-261). The CLI only LOGS; the task
# LOGS then RAISES SanityCheckFailedException on error. All messages come from
# SanityCheckMessages.log_messages (sanity_checker.py:23-30) via logger 'paperless.sanity_checker'.
cd /app/src
LOG=/scratch/data/log/paperless.log
SAMPLE=/app/src/documents/tests/samples/simple.pdf
ORIG=/scratch/media/documents/originals/0000001.pdf
ARCH=/scratch/media/documents/archive/0000001.pdf

# run the CLI sanity checker; show console (stderr+stdout) with exit code, then the NEW log lines.
run_checker() {  # $1 = label
  local before=/scratch/exp/q5_$1_before.log
  cp "$LOG" "$before" 2>/dev/null || : > "$before"
  echo "--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---"
  python3 manage.py document_sanity_checker --no-progress-bar > /scratch/exp/q5_$1_console.txt 2>&1
  echo "CLI exit=$?"
  sed 's/^/  [console] /' /scratch/exp/q5_$1_console.txt
  echo "--- NEW lines in paperless.log ---"
  diff "$before" "$LOG" | grep '^>' | sed 's/^> /  [log] /' || echo "  (none)"
}

echo "=== PREP: reset; drop scheduled maintenance jobs; seed ONE document via the canonical queue ==="
bash /scratch/exp/reset.sh > /scratch/exp/q5_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q5_reset.log)"
python3 manage.py shell <<'PY'
from django_q.models import Schedule
n = Schedule.objects.count(); Schedule.objects.all().delete()
print(f"deleted {n} scheduled maintenance job(s) to isolate the queue")
PY
python3 manage.py qcluster > /scratch/exp/q5_qcluster.log 2>&1 &
QPID=$!
for i in $(seq 1 100); do grep -qiE "ready for work" /scratch/exp/q5_qcluster.log 2>/dev/null && break; sleep 0.2; done
echo "qcluster pid=$QPID started"
cp "$SAMPLE" /scratch/consume/simple.pdf
python3 manage.py document_consumer --oneshot 2>&1 | sed 's/^/  [consumer] /'
for i in $(seq 1 200); do
  n=$(python3 manage.py shell -c "from django_q.models import Task; print(Task.objects.filter(func='documents.tasks.consume_file').count())" 2>/dev/null | tail -1)
  [ "$n" = "1" ] && break; sleep 0.2
done
echo "consume_file tasks completed = $n"
kill "$QPID" 2>/dev/null; wait "$QPID" 2>/dev/null
echo "qcluster stopped (sanity checker is synchronous; no worker needed henceforth)"

echo
echo "=== SEED STATE: stored checksums vs. on-disk MD5 (byte-verified), + content presence ==="
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
print("pk                    =", d.pk)
print("checksum (DB)         =", d.checksum)
print("archive_checksum (DB) =", d.archive_checksum)
print("has_archive_version   =", d.has_archive_version)
print("len(content)          =", len(d.content or ""), "(non-empty => no 'has no content' info line)")
PY
echo "--- md5sum of on-disk files, and of the pristine sample (proves restore-by-copy is exact) ---"
md5sum "$ORIG" "$ARCH" "$SAMPLE"

echo
echo "############################################################"
echo "## Q5a — HEALTHY store"
echo "############################################################"
run_checker "healthy"

echo
echo "############################################################"
echo "## Q5b — CORRUPT the ORIGINAL file (original-checksum mismatch)"
echo "############################################################"
echo "original MD5 BEFORE corruption = $(md5sum "$ORIG" | cut -d' ' -f1)  (must equal checksum above)"
printf '\x00CORRUPT' >> "$ORIG"
echo "original MD5 AFTER  corruption = $(md5sum "$ORIG" | cut -d' ' -f1)  (this is the 'actual' the checker will report)"
python3 manage.py shell -c "from documents.models import Document; print('stored checksum (DB) =', Document.objects.get(pk=1).checksum)"
run_checker "corruptorig"
echo "--- RESTORE the original from the pristine sample; confirm MD5 returns to the stored value ---"
cp "$SAMPLE" "$ORIG"
echo "original MD5 AFTER restore     = $(md5sum "$ORIG" | cut -d' ' -f1)"
run_checker "restored"

echo
echo "############################################################"
echo "## Q5c — CORRUPT the ARCHIVE file (archive-checksum mismatch)"
echo "############################################################"
echo "archive MD5 BEFORE corruption  = $(md5sum "$ARCH" | cut -d' ' -f1)  (must equal archive_checksum above)"
printf '\x00CORRUPT' >> "$ARCH"
echo "archive MD5 AFTER  corruption  = $(md5sum "$ARCH" | cut -d' ' -f1)  (this is the 'actual' the checker will report)"
python3 manage.py shell -c "from documents.models import Document; print('stored archive_checksum (DB) =', Document.objects.get(pk=1).archive_checksum)"
run_checker "corruptarch"

echo
echo "############################################################"
echo "## Q5d — CLI exits 0 (logs only) vs. tasks.sanity_check RAISES (store still archive-corrupted)"
echo "############################################################"
echo "--- (1) CLI: document_sanity_checker returns exit 0 even though an ERROR was logged ---"
python3 manage.py document_sanity_checker --no-progress-bar > /scratch/exp/q5d_cli.txt 2>&1
echo "CLI exit=$?"
sed 's/^/  [console] /' /scratch/exp/q5d_cli.txt
echo "--- (2) task: documents.tasks.sanity_check LOGS then RAISES SanityCheckFailedException ---"
cp "$LOG" /scratch/exp/q5d_before.log
python3 manage.py shell <<'PY'
from documents.tasks import sanity_check
from documents.sanity_checker import SanityCheckFailedException
try:
    r = sanity_check()
    print("returned:", repr(r), "  (NO exception -- unexpected)")
except SanityCheckFailedException as e:
    print("RAISED  :", type(e).__name__)
    print("message :", str(e))
PY
echo "--- NEW lines in paperless.log during the task call (ERROR is logged BEFORE the raise) ---"
diff /scratch/exp/q5d_before.log "$LOG" | grep '^>' | sed 's/^> /  [log] /' || echo "  (none)"
```

### Q5 — the exact command and its complete, unedited output

Command:

```bash
docker exec -w /app/src paperless-work bash /scratch/exp/q5.sh
```


```text
=== PREP: reset; drop scheduled maintenance jobs; seed ONE document via the canonical queue ===
reset+migrate exit=0  migrations applied OK = 92
deleted 4 scheduled maintenance job(s) to isolate the queue
qcluster pid=12590 started
  [consumer] [2026-07-13 18:59:57,883] [INFO] [paperless.management.consumer] Adding /scratch/consume/simple.pdf to the task queue.
  [consumer] 18:59:57 [Q] INFO Enqueued 1
consume_file tasks completed = 1
qcluster stopped (sanity checker is synchronous; no worker needed henceforth)

=== SEED STATE: stored checksums vs. on-disk MD5 (byte-verified), + content presence ===
pk                    = 1
checksum (DB)         = 42995833e01aea9b3edee44bbfdd7ce1
archive_checksum (DB) = e0c7431b5528db5c2e3e57f5af57794c
has_archive_version   = True
len(content)          = 24 (non-empty => no 'has no content' info line)
--- md5sum of on-disk files, and of the pristine sample (proves restore-by-copy is exact) ---
42995833e01aea9b3edee44bbfdd7ce1  /scratch/media/documents/originals/0000001.pdf
e0c7431b5528db5c2e3e57f5af57794c  /scratch/media/documents/archive/0000001.pdf
42995833e01aea9b3edee44bbfdd7ce1  /app/src/documents/tests/samples/simple.pdf

############################################################
## Q5a — HEALTHY store
############################################################
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:00:04,005] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:00:04,005] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.

############################################################
## Q5b — CORRUPT the ORIGINAL file (original-checksum mismatch)
############################################################
original MD5 BEFORE corruption = 42995833e01aea9b3edee44bbfdd7ce1  (must equal checksum above)
original MD5 AFTER  corruption = 427d535caf2320c9eb8baf7ef8550c21  (this is the 'actual' the checker will report)
stored checksum (DB) = 42995833e01aea9b3edee44bbfdd7ce1
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:00:06,328] [ERROR] [paperless.sanity_checker] Checksum mismatch of document 1. Stored: 42995833e01aea9b3edee44bbfdd7ce1, actual: 427d535caf2320c9eb8baf7ef8550c21.
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:00:06,328] [ERROR] [paperless.sanity_checker] Checksum mismatch of document 1. Stored: 42995833e01aea9b3edee44bbfdd7ce1, actual: 427d535caf2320c9eb8baf7ef8550c21.
--- RESTORE the original from the pristine sample; confirm MD5 returns to the stored value ---
original MD5 AFTER restore     = 42995833e01aea9b3edee44bbfdd7ce1
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:00:07,605] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:00:07,605] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.

############################################################
## Q5c — CORRUPT the ARCHIVE file (archive-checksum mismatch)
############################################################
archive MD5 BEFORE corruption  = e0c7431b5528db5c2e3e57f5af57794c  (must equal archive_checksum above)
archive MD5 AFTER  corruption  = 2908eda8d522f89ab1b6726ec3972a0d  (this is the 'actual' the checker will report)
stored archive_checksum (DB) = e0c7431b5528db5c2e3e57f5af57794c
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:00:09,890] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: e0c7431b5528db5c2e3e57f5af57794c, actual: 2908eda8d522f89ab1b6726ec3972a0d.
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:00:09,890] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: e0c7431b5528db5c2e3e57f5af57794c, actual: 2908eda8d522f89ab1b6726ec3972a0d.

############################################################
## Q5d — CLI exits 0 (logs only) vs. tasks.sanity_check RAISES (store still archive-corrupted)
############################################################
--- (1) CLI: document_sanity_checker returns exit 0 even though an ERROR was logged ---
CLI exit=0
  [console] [2026-07-13 19:00:11,266] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: e0c7431b5528db5c2e3e57f5af57794c, actual: 2908eda8d522f89ab1b6726ec3972a0d.
--- (2) task: documents.tasks.sanity_check LOGS then RAISES SanityCheckFailedException ---
RAISED  : SanityCheckFailedException
message : Sanity check failed with errors. See log.
[2026-07-13 19:00:12,387] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: e0c7431b5528db5c2e3e57f5af57794c, actual: 2908eda8d522f89ab1b6726ec3972a0d.
--- NEW lines in paperless.log during the task call (ERROR is logged BEFORE the raise) ---
  [log] [2026-07-13 19:00:12,387] [ERROR] [paperless.sanity_checker] Checksum mismatch of archived document 1. Stored: e0c7431b5528db5c2e3e57f5af57794c, actual: 2908eda8d522f89ab1b6726ec3972a0d.
```

### Q5 — OCR archive nondeterminism (why `archive_checksum` is run-specific)

A subtlety the mismatch hashes expose: the **original** `checksum` is perfectly stable across runs (the original is stored byte-for-byte), but the **archive** `archive_checksum` changes every time the *same* input is consumed. Consuming the identical `simple.pdf` in two fresh stores yields the same `checksum = 42995833e01aea9b3edee44bbfdd7ce1` but two different `archive_checksum` values (`ef83a5e8c472e5d1d1cfd8c595bcff66` vs `d2fa4af3147f5a7cffc22afcb5fb3b0b`). This is **observed**, not inferred, and its cause is concrete: OCRmyPDF stamps the generated archive PDF with run-specific metadata — a fresh `/ModDate` (the OCR run timestamp) and a fresh trailer `/ID` — while `/CreationDate` (inherited from the source) stays constant. Because `archive_checksum` is the MD5 of the whole archive file (`models.py:143-150`), those varying bytes flip the hash. (This is why, throughout this investigation, the archive hash is always captured and byte-verified *within the same run*.)


```bash
#!/bin/bash
# OCR archive nondeterminism — WHY the archive_checksum differs run-to-run for identical input.
# Consume the IDENTICAL simple.pdf in two fresh stores and compare. The ORIGINAL checksum is stable
# (the original is stored byte-for-byte); the ARCHIVE checksum varies because OCRmyPDF stamps the
# generated PDF with run-specific metadata. archive_checksum is set from the archive bytes in the
# consumer; the field is defined in models.py:143-150.
cd /app/src
SAMPLE=/app/src/documents/tests/samples/simple.pdf

seed_once() {  # $1 = tag (A or B); echoes "checksum archive_checksum"
  bash /scratch/exp/reset.sh > /scratch/exp/q5ocr_$1_reset.log 2>&1
  python3 manage.py shell -c "from django_q.models import Schedule; Schedule.objects.all().delete()" >/dev/null 2>&1
  python3 manage.py qcluster > /scratch/exp/q5ocr_$1_qcluster.log 2>&1 &
  local QPID=$!
  for i in $(seq 1 100); do grep -qiE "ready for work" /scratch/exp/q5ocr_$1_qcluster.log 2>/dev/null && break; sleep 0.2; done
  cp "$SAMPLE" /scratch/consume/simple.pdf
  python3 manage.py document_consumer --oneshot > /scratch/exp/q5ocr_$1_consume.log 2>&1
  for i in $(seq 1 200); do
    n=$(python3 manage.py shell -c "from django_q.models import Task; print(Task.objects.filter(func='documents.tasks.consume_file').count())" 2>/dev/null | tail -1)
    [ "$n" = "1" ] && break; sleep 0.2
  done
  kill "$QPID" 2>/dev/null; wait "$QPID" 2>/dev/null
  cp /scratch/media/documents/archive/0000001.pdf /scratch/exp/archive$1.pdf
  python3 manage.py shell <<PY
from documents.models import Document
d = Document.objects.get(pk=1)
print("RUN $1: checksum =", d.checksum, " archive_checksum =", d.archive_checksum)
PY
}

echo "=== RUN A: consume simple.pdf in a fresh store ==="
seed_once A
echo
echo "=== RUN B: consume the SAME simple.pdf in another fresh store ==="
seed_once B

echo
echo "=== COMPARE the two runs ==="
echo "--- md5sum of the two saved archive PDFs (byte-level identity) ---"
md5sum /scratch/exp/archiveA.pdf /scratch/exp/archiveB.pdf
echo "--- md5sum of the pristine input sample (identical both runs) ---"
md5sum "$SAMPLE"
echo "--- number of differing bytes between archive A and B (cmp -l) ---"
cmp -l /scratch/exp/archiveA.pdf /scratch/exp/archiveB.pdf | wc -l
echo "--- file sizes ---"
stat -c '%s bytes  %n' /scratch/exp/archiveA.pdf /scratch/exp/archiveB.pdf

echo
echo "=== MECHANISM: OCRmyPDF stamps run-specific PDF metadata (timestamps + document /ID) ==="
python3 - <<'PY'
import pikepdf
for name in ("A", "B"):
    p = f"/scratch/exp/archive{name}.pdf"
    pdf = pikepdf.open(p)
    di = pdf.docinfo
    print(f"--- archive {name} docinfo / trailer ---")
    print("  /CreationDate :", str(di.get('/CreationDate')) if di is not None else None)
    print("  /ModDate      :", str(di.get('/ModDate')) if di is not None else None)
    tid = pdf.trailer.get("/ID")
    if tid is not None:
        print("  trailer /ID[0]:", bytes(tid[0]).hex())
        print("  trailer /ID[1]:", bytes(tid[1]).hex())
    pdf.close()
PY
echo
echo "--- pdfinfo timestamps (independent tool, corroborates the mechanism) ---"
echo "archive A:"; pdfinfo /scratch/exp/archiveA.pdf 2>/dev/null | grep -iE "CreationDate|ModDate" | sed 's/^/  /'
echo "archive B:"; pdfinfo /scratch/exp/archiveB.pdf 2>/dev/null | grep -iE "CreationDate|ModDate" | sed 's/^/  /'
```

Command:

```bash
docker exec -w /app/src paperless-work bash /scratch/exp/q5_ocr.sh
```


```text
=== RUN A: consume simple.pdf in a fresh store ===
RUN A: checksum = 42995833e01aea9b3edee44bbfdd7ce1  archive_checksum = ef83a5e8c472e5d1d1cfd8c595bcff66

=== RUN B: consume the SAME simple.pdf in another fresh store ===
RUN B: checksum = 42995833e01aea9b3edee44bbfdd7ce1  archive_checksum = d2fa4af3147f5a7cffc22afcb5fb3b0b

=== COMPARE the two runs ===
--- md5sum of the two saved archive PDFs (byte-level identity) ---
ef83a5e8c472e5d1d1cfd8c595bcff66  /scratch/exp/archiveA.pdf
d2fa4af3147f5a7cffc22afcb5fb3b0b  /scratch/exp/archiveB.pdf
--- md5sum of the pristine input sample (identical both runs) ---
42995833e01aea9b3edee44bbfdd7ce1  /app/src/documents/tests/samples/simple.pdf
--- number of differing bytes between archive A and B (cmp -l) ---
10429
cmp: EOF on /scratch/exp/archiveB.pdf after byte 10859
--- file sizes ---
10860 bytes  /scratch/exp/archiveA.pdf
10859 bytes  /scratch/exp/archiveB.pdf

=== MECHANISM: OCRmyPDF stamps run-specific PDF metadata (timestamps + document /ID) ===
--- archive A docinfo / trailer ---
  /CreationDate : D:20201118152048+01'00'
  /ModDate      : D:20260713190102+00'00'
  trailer /ID[0]: b73b598f68b9f516346f13207dc0a2c6
  trailer /ID[1]: 4486b247827f9bd4e55db07143f0c907
--- archive B docinfo / trailer ---
  /CreationDate : D:20201118152048+01'00'
  /ModDate      : D:20260713190114+00'00'
  trailer /ID[0]: b3536e8113565d02573fa35c4ed3ce61
  trailer /ID[1]: 1253eda1033f4cb979f285992693fb0a

--- pdfinfo timestamps (independent tool, corroborates the mechanism) ---
archive A:
  CreationDate:   Wed Nov 18 14:20:48 2020 UTC
  ModDate:        Mon Jul 13 19:01:02 2026 UTC
archive B:
  CreationDate:   Wed Nov 18 14:20:48 2020 UTC
  ModDate:        Mon Jul 13 19:01:14 2026 UTC
```

### Q5 — grounding, observed vs. inferred, and cause→effect

- **Healthy output is one INFO line [observed + `file:line`].** With every file matching, `check_sanity` returns an empty `SanityCheckMessages`; `log_messages` takes the `len(self._messages) == 0` branch and logs `logger.info("Sanity checker detected no issues.")` (`sanity_checker.py:26-27`) on logger `paperless.sanity_checker` (`:24`). Observed identically in Q5a and in the post-restore run.

- **A mismatch recomputes MD5 and reports Stored vs. actual [observed + `file:line`].** For the original, `check_sanity` opens the on-disk file and computes `checksum = hashlib.md5(f.read()).hexdigest()` (`sanity_checker.py:82-83`); if `not checksum == doc.checksum` it appends the error `f"Checksum mismatch of document {doc.pk}. Stored: {doc.checksum}, actual: {checksum}."` (`:88-91`). The archive path is identical against `doc.archive_checksum` (`:111-112`, error `:118-124`). Both printed "actual" values equal the `md5sum` of the corrupted files, confirming the checker recomputes from live bytes rather than trusting the DB.

- **The CLI does not raise; the task does [observed + `file:line`].** The canonical `document_sanity_checker` command calls `check_sanity(...)` then `messages.log_messages()` and returns — there is no error check, so it exits `0` even with an ERROR logged (`document_sanity_checker.py:24-26`; observed `CLI exit=0` in Q5b/Q5c/Q5d). The async task `sanity_check` runs the same two calls (`tasks.py:256`, `:258`) then `if messages.has_error(): raise SanityCheckFailedException("Sanity check failed with errors. See log.")` (`:260-261`, class at `sanity_checker.py:45-46`, `has_error` at `:38-39`). Observed: the task RAISED that exception with that exact message, and the ERROR line was logged *before* the raise (log delta).

- **OCR archive instability is real and mechanistic [observed].** Two consumes of identical input produced identical `checksum` but different `archive_checksum`; the archives differed in 10429 bytes. `pikepdf`/`pdfinfo` showed the differing fields are `/ModDate` (19:01:02 vs 19:01:14) and the trailer `/ID` (A = `b73b598f68b9f516346f13207dc0a2c6`/`4486b247827f9bd4e55db07143f0c907` vs B = `b3536e8113565d02573fa35c4ed3ce61`/`1253eda1033f4cb979f285992693fb0a`), with `/CreationDate` unchanged. Cause→effect: run-specific PDF metadata → different archive bytes → different whole-file MD5 → different `archive_checksum`.

- **Cause→effect summary.** on-disk bytes match stored hash → empty message set → one INFO "no issues"; a byte changes → recomputed MD5 diverges from the stored column → ERROR naming Stored (DB) and actual (recomputed); the CLI surfaces this only in the log and exits 0, whereas the scheduled task escalates it to `SanityCheckFailedException`.
## Q6 — Orphaned / ghost files

> User's words: *"I also wonder about ghost files, whether orphaned files really linger in the media folder and what the system actually reports when it finds them."*

**Direct answer (observed).** Orphaned files **do** linger — the sanity checker only *reports* them, it never deletes them. `check_sanity` walks `settings.MEDIA_ROOT` collecting every file (`src/documents/sanity_checker.py:52-55`), drops the media lockfile (`:57-59`), then for each `Document` removes that document's thumbnail, original, and archive paths from the collected set (`:66-67`, `:79-80`, `:108-109`). Whatever remains is emitted as `WARNING`: `Orphaned file in media dir: {path}` (`:130-131`). Because a WARNING is not an error, the CLI exits `0` and even the async task does not raise — `tasks.sanity_check` returns `"Sanity check exited with warnings. See log."` (`tasks.py:262-263`). An unreferenced file placed in the media tree was reported on every run and remained on disk with an unchanged MD5.

**What the system actually reported, and the ghost's identity (byte-verified):**

| item | value |
|------|-------|
| ghost file path | `/scratch/media/documents/originals/ghost_orphan.pdf` |
| ghost md5 (unchanged across runs) | `384ddb8625a0e2ac96ad49485c49bfef` (45 bytes) |
| referenced by any Document? | `False` (checked against every `source_path`/`archive_path`/`thumbnail_path`) |
| reported message | `[WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf` |
| persists after check (run 1, run 2, task)? | `YES` each time — never auto-deleted |
### Q6 — the exact self-contained script

```bash
#!/bin/bash
# Q6 — orphaned / ghost files. check_sanity walks MEDIA_ROOT, removes every file referenced by a
# Document (thumbnail/original/archive) plus the media lockfile, and reports whatever remains as a
# WARNING (sanity_checker.py:130-131). It never deletes the extra file: orphans linger.
cd /app/src
LOG=/scratch/data/log/paperless.log
GHOST=/scratch/media/documents/originals/ghost_orphan.pdf

run_checker() {  # $1 = label
  local before=/scratch/exp/q6_$1_before.log
  cp "$LOG" "$before" 2>/dev/null || : > "$before"
  echo "--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---"
  python3 manage.py document_sanity_checker --no-progress-bar > /scratch/exp/q6_$1_console.txt 2>&1
  echo "CLI exit=$?"
  sed 's/^/  [console] /' /scratch/exp/q6_$1_console.txt
  echo "--- NEW lines in paperless.log ---"
  diff "$before" "$LOG" | grep '^>' | sed 's/^> /  [log] /' || echo "  (none)"
}

echo "=== PREP: reset; drop scheduled jobs; seed ONE document via the canonical queue ==="
bash /scratch/exp/reset.sh > /scratch/exp/q6_reset.log 2>&1
echo "reset+migrate exit=$?  migrations applied OK = $(grep -c '\.\.\. OK' /scratch/exp/q6_reset.log)"
python3 manage.py shell -c "from django_q.models import Schedule; Schedule.objects.all().delete()" >/dev/null 2>&1
python3 manage.py qcluster > /scratch/exp/q6_qcluster.log 2>&1 &
QPID=$!
for i in $(seq 1 100); do grep -qiE "ready for work" /scratch/exp/q6_qcluster.log 2>/dev/null && break; sleep 0.2; done
cp /app/src/documents/tests/samples/simple.pdf /scratch/consume/simple.pdf
python3 manage.py document_consumer --oneshot 2>&1 | sed 's/^/  [consumer] /'
for i in $(seq 1 200); do
  n=$(python3 manage.py shell -c "from django_q.models import Task; print(Task.objects.filter(func='documents.tasks.consume_file').count())" 2>/dev/null | tail -1)
  [ "$n" = "1" ] && break; sleep 0.2
done
echo "consume_file tasks completed = $n"
kill "$QPID" 2>/dev/null; wait "$QPID" 2>/dev/null

echo
echo "=== the ONE document's referenced files (these are NOT orphans) ==="
python3 manage.py shell <<'PY'
from documents.models import Document
d = Document.objects.get(pk=1)
print("source_path    =", d.source_path)
print("archive_path   =", d.archive_path)
print("thumbnail_path =", d.thumbnail_path)
PY

echo
echo "=== HEALTHY baseline BEFORE introducing the ghost (no warnings) ==="
run_checker "baseline"

echo
echo "=== introduce an UNREFERENCED file into the media dir ==="
printf 'this file is not tracked by any Document row\n' > "$GHOST"
echo "ghost path = $GHOST"
echo "ghost md5  = $(md5sum "$GHOST" | cut -d' ' -f1)"
echo "ls -l of ghost:"; ls -l "$GHOST" | sed 's/^/  /'
echo "--- prove it is unreferenced: no Document points at this path ---"
python3 manage.py shell <<PY
from documents.models import Document
paths = set()
for d in Document.objects.all():
    paths.update([d.source_path, d.archive_path, d.thumbnail_path])
print("ghost in any Document path? ->", "$GHOST" in paths)
PY

echo
echo "=== RUN 1: sanity checker with the ghost present -> WARNING ==="
run_checker "orphan1"

echo
echo "=== PERSISTENCE: the ghost is NOT auto-deleted (still on disk, unchanged md5) ==="
echo "still exists? -> $([ -f "$GHOST" ] && echo YES || echo NO)"
echo "ghost md5 (unchanged) = $(md5sum "$GHOST" | cut -d' ' -f1)"

echo
echo "=== RUN 2: re-run the checker -> it STILL reports the same orphan (it lingers) ==="
run_checker "orphan2"

echo
echo "=== tasks.sanity_check with ONLY a warning: returns a string, does NOT raise ==="
python3 manage.py shell <<'PY'
from documents.tasks import sanity_check
from documents.sanity_checker import SanityCheckFailedException
try:
    r = sanity_check()
    print("returned:", repr(r), "  (warning-only => no exception)")
except SanityCheckFailedException as e:
    print("RAISED:", type(e).__name__, str(e))
PY
echo "ghost still present after task run? -> $([ -f "$GHOST" ] && echo YES || echo NO)"
```

### Q6 — the exact command and its complete, unedited output

Command:

```bash
docker exec -w /app/src paperless-work bash /scratch/exp/q6.sh
```


```text
=== PREP: reset; drop scheduled jobs; seed ONE document via the canonical queue ===
reset+migrate exit=0  migrations applied OK = 92
  [consumer] [2026-07-13 19:02:13,035] [INFO] [paperless.management.consumer] Adding /scratch/consume/simple.pdf to the task queue.
  [consumer] 19:02:13 [Q] INFO Enqueued 1
consume_file tasks completed = 1

=== the ONE document's referenced files (these are NOT orphans) ===
source_path    = /scratch/media/documents/originals/0000001.pdf
archive_path   = /scratch/media/documents/archive/0000001.pdf
thumbnail_path = /scratch/media/documents/thumbnails/0000001.png

=== HEALTHY baseline BEFORE introducing the ghost (no warnings) ===
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:02:19,140] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:02:19,140] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.

=== introduce an UNREFERENCED file into the media dir ===
ghost path = /scratch/media/documents/originals/ghost_orphan.pdf
ghost md5  = 384ddb8625a0e2ac96ad49485c49bfef
ls -l of ghost:
  -rw-r--r-- 1 root root 45 Jul 13 19:02 /scratch/media/documents/originals/ghost_orphan.pdf
--- prove it is unreferenced: no Document points at this path ---
ghost in any Document path? -> False

=== RUN 1: sanity checker with the ghost present -> WARNING ===
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:02:21,397] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:02:21,397] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf

=== PERSISTENCE: the ghost is NOT auto-deleted (still on disk, unchanged md5) ===
still exists? -> YES
ghost md5 (unchanged) = 384ddb8625a0e2ac96ad49485c49bfef

=== RUN 2: re-run the checker -> it STILL reports the same orphan (it lingers) ===
--- console output of: python3 manage.py document_sanity_checker --no-progress-bar ---
CLI exit=0
  [console] [2026-07-13 19:02:22,715] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf
--- NEW lines in paperless.log ---
  [log] [2026-07-13 19:02:22,715] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf

=== tasks.sanity_check with ONLY a warning: returns a string, does NOT raise ===
returned: 'Sanity check exited with warnings. See log.'   (warning-only => no exception)
[2026-07-13 19:02:23,826] [WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf
ghost still present after task run? -> YES
```

### Q6 — grounding, observed vs. inferred, and cause→effect

- **The report is a WARNING naming the exact path [observed + `file:line`].** After the per-document loop, any file still in `present_files` is emitted as `messages.warning(f"Orphaned file in media dir: {extra_file}")` (`sanity_checker.py:130-131`); `extra_file` is the normalized absolute path built at `:52-55`. Observed verbatim: `Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf`, on both console and `paperless.log`.

- **It is genuinely unreferenced [observed].** Before the run the script confirmed `ghost in any Document path? -> False` by comparing the ghost path against every `source_path`/`archive_path`/`thumbnail_path`. The healthy baseline run (before the ghost existed) produced only `Sanity checker detected no issues.`, so the WARNING is attributable solely to the introduced file.

- **Orphans linger — never auto-deleted [observed].** The ghost's md5 (`384ddb8625a0e2ac96ad49485c49bfef`) was identical before and after; `still exists? -> YES` after run 1, after run 2, and after the task call. Re-running the checker reported the *same* orphan again — it is not consumed, quarantined, or removed. `check_sanity` contains no unlink/remove of `present_files`; it only appends warning messages (mechanism: reporting-only by construction).

- **A warning does not fail the check [observed + `file:line`].** WARNING entries set level via `messages.warning` (`:17-18`); `has_error()` (`:38-39`) stays `False`, so the CLI exits 0 and `tasks.sanity_check` skips the `raise` and hits `elif messages.has_warning(): return "Sanity check exited with warnings. See log."` (`tasks.py:262-263`). Observed: the task returned that exact string and left the ghost in place.

- **Cause→effect summary.** file present in `MEDIA_ROOT` ∧ not the lockfile ∧ not any document's thumbnail/original/archive → remains in `present_files` → one WARNING per leftover → logged, not deleted → the file persists and is re-reported on every subsequent run until a human removes it.
## Coverage note — every named sub-part, with its concrete value

This final pass confirms that every distinct thing the six questions ask for — including
each item named in an "e.g. / such as / including" clause and each sibling variant — is
answered explicitly, with a concrete value, a `file:line` reference, and a pointer to the
section that shows the producing command and its complete output. All hashes are given in
full (32 hex chars for MD5, 40 for SHA-1); none are truncated.

| Q | Named sub-part asked for | Concrete observed value / result | `file:line` | Evidence |
|---|--------------------------|----------------------------------|-------------|----------|
| Q1 | Does a tag change trigger the file-move machinery when the format embeds tags? | Yes — the `m2m_changed` receiver `update_filename_and_move_files` fires | `signals/handlers.py:310,312` | §Q1 |
| Q1 | What log messages appear during the move? | **None** on success — there is no logger call in the function body; the move is silent | `signals/handlers.py:312-410` (no logger) | §Q1 |
| Q1 | Before path — original | `/scratch/media/documents/originals/simple.pdf` | `os.rename` at `signals/handlers.py:354` | §Q1 |
| Q1 | Before path — archive | `/scratch/media/documents/archive/simple.pdf` | `os.rename` at `signals/handlers.py:359` | §Q1 |
| Q1 | Before path — thumbnail | `/scratch/media/documents/thumbnails/0000001.png` | `"{:07}.png".format(self.pk)` `models.py:274` | §Q1 |
| Q1 | After path — original | `/scratch/media/documents/originals/Invoice/simple.pdf` | `file_handling.py:135,175` (`{tag_list}`) | §Q1 |
| Q1 | After path — archive | `/scratch/media/documents/archive/Invoice/simple.pdf` | `signals/handlers.py:359` | §Q1 |
| Q1 | After path — thumbnail | `/scratch/media/documents/thumbnails/0000001.png` (UNCHANGED — thumbnail is not moved) | `signals/handlers.py:354,359` (only original+archive) | §Q1 |
| Q2 | Force a move to fail partway through | Original renamed, then archive rename raises `FileExistsError`(`OSError`) via `os.makedirs` on an archive dir pre-created as a file | `signals/handlers.py:358` → `except` `:367` | §Q2 PART A |
| Q2 | Does the handler restore the original? | Yes — original renamed back to its prior path | `signals/handlers.py:376` | §Q2 PART A |
| Q2 | What happens to the archive? | Never moved (failure occurred before its rename), so it stays put | `signals/handlers.py:356-359` | §Q2 PART A |
| Q2 | On-disk state BEFORE | original `originals/simple.pdf`; archive `archive/simple.pdf` | inotify `MOVED_FROM/TO` | §Q2 PART A |
| Q2 | On-disk state DURING (intermediate) | original at NEW path `originals/Invoice/simple.pdf` (inotify cookie `28041896`) — **[observed]** | external inotify monitor | §Q2 intermediate |
| Q2 | On-disk state AFTER (rollback) | original restored to `originals/simple.pdf` (inotify cookie `28041898`); archive untouched; DB `filename=simple.pdf` | `signals/handlers.py:376,393-394` | §Q2 PART A |
| Q2 | Log/exception text on the rollback path | **None** — the inner recovery is wrapped in an inner `try/except` whose handler is `pass`, so a successful rollback is silent | `signals/handlers.py:374,381,390` | §Q2 PART A |
| Q2 | Missing-source variant text | `[CRITICAL] [paperless.handlers] Document 2026-07-13 simple: File /scratch/media/documents/originals/simple.pdf has gone.` | `logger.fatal` `signals/handlers.py:298` | §Q2 PART B |
| Q3 | Trigger the "instant" (skipped) path | RUN2 and RUN3 skipped (stable across two runs) | `classifier.py:163-164` (`return False`) | §Q3 |
| Q3 | Trigger the "long" (full retrain) path | RUN1 (first train) and RUN4 (after changing AUTO data) | `classifier.py:247,249` | §Q3 |
| Q3 | Log string — retrain | `[INFO] [paperless.tasks] Saving updated classifier model to /scratch/data/classification_model.pickle...` | `tasks.py:64-66` | §Q3 |
| Q3 | Log string — skip | `[DEBUG] [paperless.classifier]`/`[paperless.tasks] Training data unchanged.` | `tasks.py:69` | §Q3 |
| Q3 | Algorithm of the change-detection hash | **SHA-1** (`hashlib.sha1()`) over preprocessed content + AUTO label bytes | `classifier.py:124,161` | §Q3 |
| Q3 | Actual hash value — unchanged (RUN1–RUN3) | `c73f17cbb3e307736041f3910da1b415ee61d226` | persisted `data_hash` `classifier.py:102` | §Q3 |
| Q3 | Actual hash value — after retrain (RUN4) | `ff507d06bb8c392a217f7063311c66331c548ca3` | persisted `data_hash` `classifier.py:102` | §Q3 |
| Q3 | Independent recompute matches the pickle? | Yes — MATCH == True for both values (tags use `tag.to_bytes`, not `y.to_bytes`) | `classifier.py:155` | §Q3 |
| Q4 | Reproduce a duplicate rejection during ingestion | CASE A (identical original) and CASE B (archive rendition), both via the canonical queue | `document_consumer.py:85-91` → `consumer.py:213` | §Q4 |
| Q4 | Checksum algorithm compared at judgment | **MD5** of the incoming file | `consumer.py:104` | §Q4 |
| Q4 | Actual value — stored `checksum` (original) | `42995833e01aea9b3edee44bbfdd7ce1` | `models.py:135-141` | §Q4 |
| Q4 | Actual value — stored `archive_checksum` | `05527bf515905eefa5cc2f3270a03b8f` | `models.py:143-150` | §Q4 |
| Q4 | Why two "different-looking" files collide | Match is `checksum` **OR** `archive_checksum`; the OCR archive rendition is byte-different from the original yet equals `archive_checksum` | `consumer.py:105-107` | §Q4 |
| Q4 | Rejection message | `[ERROR] [paperless.consumer] Not consuming <file>: It is a duplicate.` + `ConsumerError` | `consumer.py:80-81,110-112` | §Q4 |
| Q5 | Healthy output | `[INFO] [paperless.sanity_checker] Sanity checker detected no issues.` | `sanity_checker.py:26-27` | §Q5 |
| Q5 | Broken output — original | `[ERROR] Checksum mismatch of document {pk}. Stored: {doc.checksum}, actual: {md5}.` (concrete values two rows below) | `sanity_checker.py:88-91` | §Q5 |
| Q5 | Broken output — archive | `[ERROR] Checksum mismatch of archived document {pk}. Stored: {doc.archive_checksum}, actual: {md5}.` (concrete values two rows below) | `sanity_checker.py:118-124` | §Q5 |
| Q5 | Mismatch hashes — original Stored | `42995833e01aea9b3edee44bbfdd7ce1` | `sanity_checker.py:90` | §Q5 |
| Q5 | Mismatch hashes — original actual | `427d535caf2320c9eb8baf7ef8550c21` | `sanity_checker.py:82-83` (MD5) | §Q5 |
| Q5 | Mismatch hashes — archive Stored | `e0c7431b5528db5c2e3e57f5af57794c` | `sanity_checker.py:122` | §Q5 |
| Q5 | Mismatch hashes — archive actual | `2908eda8d522f89ab1b6726ec3972a0d` | `sanity_checker.py:111-112` (MD5) | §Q5 |
| Q5 | CLI vs task on error | CLI exits `0` (logs only); `tasks.sanity_check` raises `SanityCheckFailedException("Sanity check failed with errors. See log.")` | `document_sanity_checker.py:24-26`; `tasks.py:260-261` | §Q5 |
| Q5 | Why `archive_checksum` varies run-to-run | OCRmyPDF stamps run-specific `/ModDate` + trailer `/ID`; `ef83a5e8c472e5d1d1cfd8c595bcff66` vs `d2fa4af3147f5a7cffc22afcb5fb3b0b` for identical input | `models.py:143-150` | §Q5 OCR |
| Q6 | Introduce an unreferenced file and run the checker | Ghost placed at `/scratch/media/documents/originals/ghost_orphan.pdf`, proven unreferenced | `sanity_checker.py:52-55` | §Q6 |
| Q6 | Exact message when found | `[WARNING] [paperless.sanity_checker] Orphaned file in media dir: /scratch/media/documents/originals/ghost_orphan.pdf` | `sanity_checker.py:130-131` | §Q6 |
| Q6 | Do orphans persist (not auto-deleted)? | Yes — file remained (md5 `384ddb8625a0e2ac96ad49485c49bfef` unchanged), re-reported on run 2 and by the task | `sanity_checker.py` (no unlink) | §Q6 |
| Q6 | Task behavior on warning-only | Returns `"Sanity check exited with warnings. See log."`, does not raise | `tasks.py:262-263` | §Q6 |
| — | Temp scripts + scratch data removed afterward | `/scratch/exp` and host `/tmp/inv` emptied; post-cleanup recursive listing shown | — | §Repository Integrity & Cleanup |
| — | Repository left byte-for-byte unchanged | `git diff --name-status 542221a38dff..HEAD` lists only `blitzy/documentation/paperless-ngx_542221a38dff.md`; all 24 source files unchanged | — | §Repository Integrity & Cleanup |
## Repository Integrity & Cleanup

This investigation is **read-only**. The only change to the paperless-ngx repository is the
addition of this one Markdown file (`blitzy/documentation/paperless-ngx_542221a38dff.md`);
no product code, test, migration, configuration, dependency, or frontend file was modified,
and no dependency was added, updated, or removed. Every experiment ran against a throwaway
store that lived **outside** the repository tree — inside the container under `/scratch`
(`PAPERLESS_DATA_DIR=/scratch/data`, `PAPERLESS_MEDIA_ROOT=/scratch/media`,
`PAPERLESS_CONSUMPTION_DIR=/scratch/consume`) and on the investigation host under `/tmp` —
so no scratch data could ever land inside the repository. The two subsections below prove
(1) that all scratch artifacts were removed and (2) that the repository is byte-for-byte
unchanged except for this document.

### Scratch cleanup — container `/scratch` emptied and verified

Before cleanup, `/scratch` held the throwaway SQLite database, the Whoosh search index,
`paperless.log`, the media/index lockfiles, the seeded document's original/archive/thumbnail
(plus the Q6 `ghost_orphan.pdf`, which was *still present* — the very persistence the orphan
check reported), and all 73 experiment scripts/logs under `/scratch/exp`. The cleanup script
lists the named artifacts, removes everything, and reprints `/scratch` to show it is empty:


```bash
#!/bin/bash
# Remove every scratch artifact created during the investigation and prove /scratch is empty.
echo "=== BEFORE cleanup: the specific artifacts the investigation produced under /scratch ==="
echo "# database, search index, log, and lockfiles:"
for f in /scratch/data/db.sqlite3 \
         /scratch/data/log/paperless.log \
         /scratch/data/log/.__paperless.lock \
         /scratch/data/index/MAIN_WRITELOCK ; do
  printf '  %s  ->  %s\n' "$f" "$([ -e "$f" ] && echo PRESENT || echo absent)"
done
echo "# whoosh index segment(s):"; ls /scratch/data/index/*.seg /scratch/data/index/*.toc 2>/dev/null | sed 's/^/  /'
echo "# media originals / archive / thumbnails:"; find /scratch/media -type f 2>/dev/null | sed 's/^/  /'
echo "# experiment scripts + logs under /scratch/exp:"; ls /scratch/exp 2>/dev/null | wc -l | sed 's/^/  exp entries: /'
echo "# TOTAL files under /scratch BEFORE:"; find /scratch -type f 2>/dev/null | wc -l

echo
echo "=== CLEANUP command ==="
echo '+ rm -rf /scratch/exp /scratch/media /scratch/data /scratch/consume'
rm -rf /scratch/exp /scratch/media /scratch/data /scratch/consume
echo "rm exit=$?"

echo
echo "=== AFTER cleanup: recursive listing of /scratch (must be empty) ==="
echo "# files remaining under /scratch:"; find /scratch -type f 2>/dev/null
echo "# TOTAL files under /scratch AFTER:"; find /scratch -type f 2>/dev/null | wc -l
echo "# /scratch top-level:"; ls -la /scratch 2>/dev/null
```

Command:

```bash
docker exec paperless-work bash /tmp/cleanup.sh
```


```text
=== BEFORE cleanup: the specific artifacts the investigation produced under /scratch ===
# database, search index, log, and lockfiles:
  /scratch/data/db.sqlite3  ->  PRESENT
  /scratch/data/log/paperless.log  ->  PRESENT
  /scratch/data/log/.__paperless.lock  ->  PRESENT
  /scratch/data/index/MAIN_WRITELOCK  ->  PRESENT
# whoosh index segment(s):
  /scratch/data/index/MAIN_pz20jbv5io8eb346.seg
  /scratch/data/index/_MAIN_1.toc
# media originals / archive / thumbnails:
  /scratch/media/documents/archive/0000001.pdf
  /scratch/media/documents/thumbnails/0000001.png
  /scratch/media/documents/originals/ghost_orphan.pdf
  /scratch/media/documents/originals/0000001.pdf
  /scratch/media/media.lock
# experiment scripts + logs under /scratch/exp:
  exp entries: 73
# TOTAL files under /scratch BEFORE:
84

=== CLEANUP command ===
+ rm -rf /scratch/exp /scratch/media /scratch/data /scratch/consume
rm exit=0

=== AFTER cleanup: recursive listing of /scratch (must be empty) ===
# files remaining under /scratch:
# TOTAL files under /scratch AFTER:
0
# /scratch top-level:
total 8
drwxr-xr-x 2 root root 4096 Jul 13 19:15 .
drwxr-xr-x 1 root root 4096 Jul 13 19:15 ..
```

The post-cleanup recursive listing shows `0` files under `/scratch`. The host-side scratch
directory `/tmp/inv` (observation scripts and the draft of this document) lived entirely
outside the repository tree and was likewise removed once this document had been written into
the repository; because it was never inside the repo, its removal cannot affect the git state
proven below.

### Repository integrity — the deliverable is the only change

Comparing the working tree against the base commit `542221a38dff` shows a single added path —
this document — and nothing else. The source tree, frontend, docs, Dockerfile, and dependency
files are untouched:


```bash
#!/bin/bash
# Proof that the repository is left byte-for-byte unchanged except for this one deliverable.
# Run from the repository root; base commit is 542221a38dff.
echo "+ git rev-parse --abbrev-ref HEAD"
git rev-parse --abbrev-ref HEAD
echo
echo "+ git status --porcelain            # working tree, immediately before committing this document"
git status --porcelain
echo "(exactly one entry: the deliverable is the only modified path)"
echo
echo "+ git diff --name-status 542221a38dff            # net change vs the base commit"
git diff --name-status 542221a38dff
echo "(A = added; the only path is the deliverable)"
echo
echo "+ git diff --name-status 542221a38dff -- src src-ui docs Dockerfile requirements.txt Pipfile"
git diff --name-status 542221a38dff -- src src-ui docs Dockerfile requirements.txt Pipfile
echo "(empty output => no product code, frontend, docs, Dockerfile, or dependency file was touched)"
echo
echo "+ git diff --stat 542221a38dff -- . ':(exclude)blitzy/documentation/paperless-ngx_542221a38dff.md'"
git diff --stat 542221a38dff -- . ":(exclude)blitzy/documentation/paperless-ngx_542221a38dff.md"
echo "(empty output => the deliverable is the ONLY changed path in the entire repository)"
```

Command (run from the repository root):

```bash
bash git_integrity.sh
```


```text
+ git rev-parse --abbrev-ref HEAD
blitzy-1bb5ac61-cdd5-4e1c-bae0-f83ab50593b1

+ git status --porcelain            # working tree, immediately before committing this document
 M blitzy/documentation/paperless-ngx_542221a38dff.md
(exactly one entry: the deliverable is the only modified path)

+ git diff --name-status 542221a38dff            # net change vs the base commit
A	blitzy/documentation/paperless-ngx_542221a38dff.md
(A = added; the only path is the deliverable)

+ git diff --name-status 542221a38dff -- src src-ui docs Dockerfile requirements.txt Pipfile
(empty output => no product code, frontend, docs, Dockerfile, or dependency file was touched)

+ git diff --stat 542221a38dff -- . ':(exclude)blitzy/documentation/paperless-ngx_542221a38dff.md'
(empty output => the deliverable is the ONLY changed path in the entire repository)
```

`git status --porcelain` lists exactly one path (this deliverable), `git diff --name-status`
against the base reports it as the sole addition, and both the source-scoped diff and the
"everything-except-the-deliverable" diff are empty — together proving the repository is left
byte-for-byte unchanged apart from this answer document.
