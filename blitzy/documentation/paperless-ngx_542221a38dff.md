# Paperless-NGX Runtime Idle-Behavior — Evidence-Grounded Q&A

**Commit:** `542221a38dff06361e07976452f9aea24d210542`
**Branch:** `paperless-ngx_542221a38dff`
**Scope:** Read-only runtime investigation. This document is the **only** artifact produced; **no source file was modified**.

This document answers five questions about how Paperless-NGX behaves at runtime once it is **up and idle** (no documents being processed):

- **Q1** — How do you bring the system up at this commit? What is the startup/readiness output?
- **Q2** — What background processes/tasks keep executing automatically once idle?
- **Q3** — Which log entries appear *periodically* to show the system is healthy/ready, at what frequency, and what do they mean?
- **Q4** — After briefly interrupting and restarting part of the system, which log messages confirm it has reconnected and is operational again?
- **Q5** — Which components/processes run continuously to keep the system in a ready state with zero document activity?

Every behavioral claim below is paired with **the exact command that produced it** and **actual, unedited log output**, and every factual claim carries a `file:line` citation. Where a value is **inferred** rather than **observed**, it is labeled as such; where a value depends on the environment it is labeled **environment-specific**; and where the observation environment departs from the canonical product image the departure is labeled a **non-canonical deviation**.

---

## 0. How this was investigated (methodology, provenance, environment)

### 0.1 Engine identity — Django-Q, **not** Celery (state this first)

At this commit the background-task engine is **Django-Q** (`python3 manage.py qcluster`), **not** Celery. Evidence: `requirements.txt` pins `django-q==1.3.9` [requirements.txt:37] and `django==4.0.4` [requirements.txt:38], and the file contains **no** `celery` dependency. (Paperless-NGX's "transition to Celery for background tasks" landed later, in the **1.10.0** release — the official release notes list *"Feature: Transition to celery for background tasks"* (PR #1648) at <https://github.com/paperless-ngx/paperless-ngx/releases/tag/v1.10.0>; this commit is from April 2022 and predates it.) Consequently, any `celery.py` / `celery worker` patterns in newer paperless-ngx documentation do **not** apply here. All periodic and reconnection behavior documented below is produced by the Django-Q cluster.

### 0.2 Provenance (image, containers, versions, git state)

All observation ran inside throwaway Docker containers spawned from the image supplied for this task, `paperless-ngx-ready:542221a38dff`. **This image is *derived from* the project setup, not the official product image built from the repository `Dockerfile`.** That distinction is made explicit here and its consequences are labeled as **non-canonical deviations** wherever they matter (see §0.3).

| Provenance item | Value (observed) |
|---|---|
| Observation image | `paperless-ngx-ready:542221a38dff` |
| Observation image ID | `sha256:aae32959b227dec71bfe12f555d5a02de2edc0ee6bcf1f321d35506fb0e71051` (locally built; empty `RepoDigests`) |
| Base image | `ghcr.io/scaleapi/swe-atlas:...qna_1.01` |
| Base image ID | `sha256:6e699f225ced49182033cf995daf2a07d3628fe29bb573f5aa4c4188c253969f` |
| Container — Run A (canonical supervisord) + Run B (isolated qcluster) | `87dc49c0978bfd50889028d88a50b9280fc6b1defbd59998658937079c1d4944` |
| Container — Run C (fresh canonical supervisord re-capture, Q4 + assertions) | `c61141c80470e3b7a27efd89cb956590344ef05da676de98cd9a5468853a4de2` |
| git `HEAD` of `/app` source | `542221a38dff06361e07976452f9aea24d210542` |

Version verification (run inside the container):

```text
$ python3 --version
Python 3.9.23
$ python3 -c "import django,django_q,redis,channels; \
print('django',django.get_version()); print('django_q',django_q.VERSION); \
print('redis-py',redis.__version__); print('channels',channels.__version__)"
django 4.0.4
django_q (1, 3, 9)
redis-py 3.5.3
channels 3.0.4
$ redis-server --version | head -1
Redis server v=6.0.16 sha=00000000:0 malloc=jemalloc-5.2.1 bits=64 build=d4b5be3f91fa055c
$ nproc
128
```

**Environment-specific values (labeled so throughout):** the humanized cluster id (e.g. `nevada-quebec-two-nuts`) is **random per start**, and the worker count is **CPU-derived**. Paperless computes `default_task_workers()` = `floor(sqrt(cores))` for hosts with ≥4 cores [src/paperless/settings.py:427-435], assigns it to `TASK_WORKERS` [src/paperless/settings.py:438], and uses it as `Q_CLUSTER["workers"]` [src/paperless/settings.py:455]. On this 128-core host that is `floor(sqrt(128)) = 11` workers. The documented `PAPERLESS_TASK_WORKERS=1` in `paperless.conf.example` is a **commented** example line [paperless.conf.example:57], so the *true* default is the CPU-derived value, not 1.

> **Note on `django_q/*` citations.** Line references of the form `django_q/<file>:<line>` point at the **installed dependency** `django-q==1.3.9` (at `/usr/local/lib/python3.9/site-packages/django_q/`), **not** repository source. They are cited to pin the exact log strings and timing constants that drive the observed behavior. All such citations were verified against the installed 1.3.9 code.

### 0.3 Non-canonical deviations (labeled honestly)

The following are the ways the observation environment departs from a fresh official Docker build. None affects the **scheduling, broker, or reconnection behavior** documented below (those are produced entirely by `django-q==1.3.9` + `redis-server` + the pinned settings, all identical here), but they are stated so the report is honest about what was and was not exercised:

- **Setup-derived image, not an official `Dockerfile` build.** The image lacks `/sbin/docker-entrypoint.sh`, `/etc/supervisord.conf`, the `/usr/src/paperless` path, the `paperless` user, and `gosu`. It *does* ship `supervisord` (at `/usr/bin/supervisord`, not the Dockerfile's `/usr/local/bin/supervisord` [Dockerfile:172]), `redis-server`, and `redis-cli`. The canonical entrypoint chain (`docker-entrypoint.sh` → `gosu paperless docker-prepare.sh` → `supervisord`) was therefore **reproduced by hand** (create the `paperless` user, symlink `/usr/src/paperless → /app`, run the real `docker/wait-for-redis.py` gate, then launch the repository's own `docker/supervisord.conf`).
- **Manually-created `paperless` user** with `uid=gid=1001` (uid 1000 was already taken in the image) and home `/home/paperless` (required by python-gnupg at import).
- **Co-located Redis on loopback** (`--bind 127.0.0.1 --protected-mode yes`) inside the same container, rather than an external broker service. This is an **observation-only** convenience; it is the exact broker transport Django-Q uses (`redis-py 3.5.3`), so broker/reconnection behavior is unaffected.
- **`supervisord` and `redis-server` run as root**, while the three supervised programs correctly run as `paperless` (matching `user=paperless` in `docker/supervisord.conf` [docker/supervisord.conf:12,21,30]).

### 0.4 Log formats you will see

Two distinct formats stream to stdout/stderr:

- **Django-Q engine lines**: `HH:MM:SS [Q] <LEVEL> <message>` — from the single `django-q` logger [django_q/conf.py:207] whose formatter is `fmt="%(asctime)s [Q] %(levelname)s %(message)s", datefmt="%H:%M:%S"` [django_q/conf.py:213-214].
- **Paperless application lines**: `[{asctime}] [{levelname}] [{name}] {message}` — e.g. `[2026-07-13 18:44:33,000] [INFO] [paperless.sanity_checker] Sanity checker detected no issues.`

**`PAPERLESS_DISABLE_DBHANDLER` was deliberately NOT set.** At this commit that variable has **no runtime effect**: a repository-wide search finds it **only** in `src/setup.cfg:12` (a `pytest`/test-harness environment key), and the runtime `LOGGING` dict in `src/paperless/settings.py` declares only `console`, `file_paperless`, and `file_mail` handlers — there is **no** database log handler and the variable is never read outside tests. Grep proof:

```text
$ grep -rn "PAPERLESS_DISABLE_DBHANDLER" --include="*.py" --include="*.cfg" src/
src/setup.cfg:12:  PAPERLESS_DISABLE_DBHANDLER=true
```

### 0.5 Reproduction procedure (fully executable; exact commands, PIDs, cleanup)

The following is a **complete, executable** procedure. It provisions the canonical `supervisord` orchestration by hand (see §0.3), captures every signal, performs the live Redis interrupt/restart, and cleans up by **explicit numeric PID** (never a broad `pkill`). Nothing touches the repository checkout.

```bash
set -eu
IMG=paperless-ngx-ready:542221a38dff
C=pngx-obs

# Idempotent teardown on ANY exit (success or failure) so this procedure never leaves a
# live runtime behind. `docker rm -f` also stops the container, so a single call suffices.
cleanup() { docker rm -f "$C" >/dev/null 2>&1 || true; }
trap cleanup EXIT

# Collision-safe: remove any stale container of the same name left by a prior run first.
docker rm -f "$C" >/dev/null 2>&1 || true

# Bounded poller: retry `<cmd...>` up to <tries> times, 0.5 s apart, until it succeeds. It is
# silent on success (so every observed output below is byte-for-byte unchanged) and, on
# timeout, returns non-zero -> `set -e` aborts the script -> the EXIT trap tears the container
# down. This is what makes each readiness gate below both deterministic and self-cleaning.
poll() {
  tries=$1; what=$2; shift 2; i=0
  while [ "$i" -lt "$tries" ]; do
    if "$@" >/dev/null 2>&1; then return 0; fi
    i=$((i + 1)); sleep 0.5
  done
  echo "TIMED OUT after $tries tries waiting for: $what" >&2; return 1
}

# 1) Spawn a throwaway container (sleeps; we drive it with `docker exec`)
docker run -d --name "$C" --entrypoint /bin/sleep -w /app "$IMG" infinity

# 2) Provision the canonical layout by hand (see §0.3 for why this is needed)
docker exec "$C" bash -c '
  id -u paperless >/dev/null 2>&1 || { groupadd -g 1001 paperless; \
    useradd -u 1001 -g paperless -M -s /usr/sbin/nologin paperless; }
  mkdir -p /home/paperless                 # python-gnupg needs $HOME to exist
  [ -e /usr/src/paperless ] || ln -s /app /usr/src/paperless
  mkdir -p /app/data /app/data/index \
    /app/media/documents/{originals,archive,thumbnails} \
    /app/consume /app/export /app/static /tmp/paperless_obs \
    /var/log/supervisord /var/run/supervisord
  chown -R paperless:paperless /app/data /app/media /app/consume /app/export \
    /app/static /home/paperless /tmp/paperless_obs /var/log/supervisord /var/run/supervisord
'

# 3) Start the Redis broker, loopback-bound (observation-only; see §0.3)
docker exec "$C" bash -c '
  redis-server --daemonize yes --bind 127.0.0.1 --protected-mode yes --port 6379 \
    --save "" --appendonly no --dir /tmp --pidfile /tmp/redis6379.pid
'
poll 20 "redis PONG" docker exec "$C" redis-cli -p 6379 ping   # wait for the broker to accept
docker exec "$C" bash -c '
  redis-cli -p 6379 ping                                # -> PONG
  redis-cli -p 6379 CONFIG GET bind                     # -> bind 127.0.0.1
  redis-cli -p 6379 CONFIG GET protected-mode           # -> protected-mode yes
'

# 4) Run the REAL startup gate + migrations, as the paperless user
docker exec -u paperless -e HOME=/home/paperless -e PAPERLESS_REDIS=redis://localhost:6379 \
  -w /app/src "$C" bash -c '
    python3 /app/docker/wait-for-redis.py          # emits Waiting/Connected lines
    python3 manage.py migrate --no-input           # creates SQLite DB + 4 Schedule rows
'

# 5) Launch the canonical supervisord orchestration (all 3 programs as paperless).
#    supervisord.conf has nodaemon=true; -d detaches the docker exec, log to a file.
docker exec -d -e HOME=/home/paperless -e PAPERLESS_REDIS=redis://localhost:6379 \
  -w /app/src "$C" bash -c 'exec supervisord -c /app/docker/supervisord.conf \
    > /tmp/paperless_obs/run.log 2>&1'

# supervisord writes its own pidfile asynchronously -> WAIT for it to exist and be non-empty
# before reading it (reading it too early is the race that made the original snippet fail),
# then capture the numeric PID for a clean numeric-PID stop later.
poll 40 "supervisord pidfile" docker exec "$C" bash -c 'test -s /var/run/supervisord/supervisord.pid'
SUP_PID=$(docker exec "$C" cat /var/run/supervisord/supervisord.pid)

# 5b) Write the three tiny active-assertion helpers (temporary; removed at cleanup).
#     Quoted heredoc delimiter (<<'PYEOF') prevents host-shell expansion; -i pipes stdin to tee.
docker exec "$C" mkdir -p /tmp/q4helpers
docker exec -i "$C" tee /tmp/q4helpers/assert_http.py >/dev/null <<'PYEOF'
import urllib.request, urllib.error
url = 'http://127.0.0.1:8000/api/'
try:
    resp = urllib.request.urlopen(url, timeout=10)
    body = resp.read(120)
    print(f"HTTP_STATUS={resp.status} url={url} body_prefix={body[:60]!r}")
except urllib.error.HTTPError as e:
    print(f"HTTP_STATUS={e.code} url={url} (HTTPError)")
except Exception as e:
    print(f"HTTP_ERROR={type(e).__name__}: {e}")
PYEOF
docker exec -i "$C" tee /tmp/q4helpers/assert_channels.py >/dev/null <<'PYEOF'
import os, sys, asyncio, django
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'paperless.settings')
django.setup()
from channels.layers import get_channel_layer
async def main():
    token = sys.argv[1]
    layer = get_channel_layer()
    await layer.send('q4-test-chan', {'type': 'probe', 'payload': token})
    msg = await layer.receive('q4-test-chan')
    print(f"CHANNELS_RECV={msg}")
asyncio.run(main())
PYEOF
docker exec -i "$C" tee /tmp/q4helpers/assert_broker.py >/dev/null <<'PYEOF'
import os, sys, django
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'paperless.settings')
django.setup()
from django_q.tasks import async_task, result
val = float(sys.argv[1])
tid = async_task('math.sqrt', val)
print(f"BROKER_ENQUEUED task_id={tid} func=math.sqrt arg={val}")
r = result(tid, wait=10000)  # block up to 10s for the worker to store the result
print(f"BROKER_RESULT={r}")
PYEOF

# 6) OBSERVE the idle steady state. Each command's actual output is shown in the noted Q-section.
E="-u paperless -e HOME=/home/paperless -e PYTHONPATH=/app/src -e PAPERLESS_REDIS=redis://localhost:6379 -w /app/src"
# Wait until all three supervised programs are ready before observing: Django-Q printed its
# one-time readiness banner, and gunicorn answers HTTP 200 on the REST API.
poll 60 "Django-Q readiness banner" docker exec "$C" grep -q 'running\.' /tmp/paperless_obs/run.log
poll 60 "gunicorn HTTP 200" docker exec $E "$C" bash -c 'python3 /tmp/q4helpers/assert_http.py | grep -q HTTP_STATUS=200'
docker exec "$C" sed -n '1,20p' /tmp/paperless_obs/run.log                            # startup banner        -> Q1.4
# The first scheduler pass fires ~30 s after the banner; wait for it to land before grepping.
poll 120 "first scheduler pass" docker exec "$C" grep -q 'created a task from schedule' /tmp/paperless_obs/run.log
docker exec "$C" grep -nE 'created a task from schedule' /tmp/paperless_obs/run.log   # ~30s pass + mail cadence -> Q3.3/Q3.4
docker exec "$C" bash -c 'timeout 4 redis-cli -p 6379 monitor | grep "cluster:"'      # 0.5s heartbeat SETs   -> Q3.2
# Idle process tree (ps/pgrep absent in this image -> walk /proc); -> Q5.1:
docker exec "$C" bash -c 'printf "%-6s %-6s %-9s %s\n" PID PPID USER CMD; \
  for p in /proc/[0-9]*; do pid=${p#/proc/}; [ -r "$p/stat" ] || continue; \
    ppid=$(cut -d" " -f4 "$p/stat"); user=$(stat -c %U "$p" 2>/dev/null); \
    cmd=$(tr "\0" " " < "$p/cmdline" 2>/dev/null); \
    case "$cmd" in *supervisord*|*qcluster*|*document_consumer*|*gunicorn*|*redis-server*) \
      printf "%-6s %-6s %-9s %s\n" "$pid" "$ppid" "$user" "$cmd";; esac; \
  done | sort -n -k1'
# BEFORE-outage active baseline via the REAL entry points (output -> Q4.1):
docker exec $E "$C" python3 /tmp/q4helpers/assert_http.py                      # -> HTTP_STATUS=200
docker exec $E "$C" python3 /tmp/q4helpers/assert_channels.py channels-ok-before  # -> CHANNELS_RECV
docker exec $E "$C" python3 /tmp/q4helpers/assert_broker.py 16                 # -> BROKER_RESULT=4.0

# 7) Q4: interrupt the LIVE broker, let the error burst accrue, then restart it
docker exec "$C" bash -c 'date +%s.%N; redis-cli -p 6379 shutdown nosave'   # record SHUTDOWN_TS
sleep 35
docker exec "$C" bash -c 'date +%s.%N; redis-server --daemonize yes \
  --bind 127.0.0.1 --protected-mode yes --port 6379 --save "" --appendonly no \
  --dir /tmp --pidfile /tmp/redis6379.pid; sleep 1; redis-cli -p 6379 ping'   # record RESTART_TS

# 8) AFTER-restart active assertions via the REAL entry points (output -> Q4.3).
# The guard must reincarnate the pusher and reconnect to the restarted broker first; wait for
# a real broker round-trip to succeed before capturing the shown assertion.
poll 24 "broker operational again" docker exec $E "$C" bash -c 'python3 /tmp/q4helpers/assert_broker.py 81 | grep -q "BROKER_RESULT=9.0"'
docker exec $E "$C" python3 /tmp/q4helpers/assert_broker.py 81                 # -> BROKER_RESULT=9.0
docker exec $E "$C" python3 /tmp/q4helpers/assert_channels.py channels-ok-after-recovery  # -> CHANNELS_RECV
docker exec $E "$C" python3 /tmp/q4helpers/assert_http.py                      # -> HTTP_STATUS=200
# Confirm the error burst ceased and the recovery signature is present in the log (-> Q4.3):
docker exec "$C" grep -nE 'stopped pushing tasks|reincarnated pusher|pushing tasks at' \
  /tmp/paperless_obs/run.log | tail -3
# Heartbeat resumed: the cluster Stat key exists again with a positive TTL (<= 3000 ms):
docker exec "$C" bash -c 'k=$(redis-cli -p 6379 --scan --pattern "django_q:paperless:cluster:*"); \
  echo "stat_key=$k"; redis-cli -p 6379 pttl "$k"'                             # -> PTTL_ms ~2656

# 9) Graceful stop by explicit numeric PID (never a broad pkill)
docker exec "$C" bash -c "kill -TERM $SUP_PID"    # supervisord stops all 3 programs
# Wait for supervisord (and thus all three programs) to actually exit before tearing down.
poll 60 "supervisord exit" docker exec "$C" bash -c "kill -0 $SUP_PID 2>/dev/null && exit 1 || exit 0"

# 10) Cleanup: stop Redis, remove the container, verify host clean
docker exec "$C" bash -c 'redis-cli -p 6379 shutdown nosave 2>/dev/null || true'
docker stop "$C" && docker rm "$C"
trap - EXIT                                        # success path: disarm the idempotent trap
docker ps -a --format '{{.Names}}' | grep -q "$C" && echo LEFTOVER || echo "container removed - clean"
```

**Run duration / scale.** The 10-minute mail cadence (Q3.4) was measured over **two fully independent, ~22-minute idle runs** — Run A (canonical `supervisord`) and Run B (an isolated `python3 manage.py qcluster` on a *separate* Redis instance and SQLite DB) — each capturing **three** consecutive mail-check firings. The ~30-second scheduler tick was confirmed on **three** independent cluster starts (Runs A, B, C). The Q4 interrupt/restart and all before/during/after active assertions were captured on Run C.

### 0.6 Evidence integrity — SHA-256 of every captured log

Every embedded log block below is a byte-faithful excerpt of one of these captured files. Hashes are provided so the evidence is auditable/regression-checkable. (The complete `--- Logging error ---` block hashed **identically** across two independent runs — `5c0ac25a…` — an independent byte-faithfulness cross-check.)

```text
d8ac158abeab461ea2c18fa952bef057cad20d4501fa5bdc07dbc440a31811b5  run_a.log            (Run A, canonical supervisord)
a32025f74ba3c024aed901e123acbbf020caa4e7a803eb768ee4e648cbc67cbb  run_b.log            (Run B, isolated qcluster)
197515786361a50bf8269d35647213b7157aa653126f0f178a662aab5a26a472  run_b_shutdown.txt   (Run B graceful stop)
078fd25c595c82ce05bf48bc802a1da8de2990271d40434b3f1dfd87869d52e8  q4_run.log           (Run C, canonical supervisord)
9140277c564cf68c916153c79f3739ef77b2110bef7b4c9ea42214e7a740d3cf  startup_banner_clean.txt
678ced9475dcc3b5d54d605101d21893e31aafb01cc6614586c56fdf3e6fc1cc  supervisord_orchestration.txt
0a3b9a0dfe142e70e50c3aa038968c763505e68325d99450cf8a2533f556aa78  migrate_schedules.log
4d12c68f7d448a4a5f508ac1a5f4afc571a145a1473583c9895a11f49430f985  first_scheduler_pass.txt
6260d7978d9acab15bae403c2e600e4413057e26ef8683547892d222c1e58eee  monitor_set.txt      (0.5s heartbeat)
528f4a40f016d9e92a40c03d9208e53953b741a82087e1246ad2e4a3974e5bae  assert_before.txt
43ad6ee921f3471a9e4bf2ad8d98fc58607750874b0165f816a17f9a44a170d2  assert_after.txt
e1637f88bca91179410299f7747db83d327013a808f5b9959e46fcb987ea5dad  outage_burst.txt
6f87b54c99af59d877bb43647e325919e4e92aeac27299ff0c278902cea816db  recovery_signature.txt
5c0ac25a6c3f0580cedc24b6de3c27a475e4eef9d2a6fff1844875cda938744a  logging_error_ONE.txt
db4c05531747a81b14cd07b3a9cd44ea45c215fe461c5d82f30cad3ef668a89a  shutdown_sequence.txt
6419469663df8d3de1577a3b7bb95b61985dc974647df74a2e3f39574bfaf6e5  proctree.txt
```

---

## Q1 — Bringing the system up (canonical `supervisord` path, actually executed)

### Q1.1 Canonical container startup path (what the official image does)

`ENTRYPOINT` [Dockerfile:168] runs `docker/docker-entrypoint.sh`, whose `initialize()` execs `gosu paperless /sbin/docker-prepare.sh` [docker/docker-entrypoint.sh:37]. `docker-prepare.sh`'s `do_work()` **body** [docker/docker-prepare.sh:66-79] then runs, in order:

1. `wait_for_postgres` — **only if `PAPERLESS_DBHOST` is set** [docker/docker-prepare.sh:67-69] (so with the default SQLite DB this step is skipped);
2. `wait_for_redis` [docker/docker-prepare.sh:71];
3. `migrations` [docker/docker-prepare.sh:73];
4. `search_index` [docker/docker-prepare.sh:75];
5. `superuser` [docker/docker-prepare.sh:77].

(`do_work` is *invoked* on the following line [docker/docker-prepare.sh:81].) The Redis gate is `docker/wait-for-redis.py`: it pings Redis up to `MAX_RETRY_COUNT = 5` times [docker/wait-for-redis.py:16] sleeping `RETRY_SLEEP_SECONDS = 5` between attempts [docker/wait-for-redis.py:17]; it prints `Waiting for Redis: <url>` [docker/wait-for-redis.py:21], then on success `Connected to Redis broker: <url>` [docker/wait-for-redis.py:41], or after 5 failures `Failed to connect to: <url>` and exits non-zero [docker/wait-for-redis.py:38].

The entrypoint then execs the `CMD` → `supervisord -c /etc/supervisord.conf` [Dockerfile:172], which runs **three** long-lived programs, each as `user=paperless` [docker/supervisord.conf:10-30]:

- `[program:gunicorn]` → `gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application` [docker/supervisord.conf:10-11], `user=paperless` [docker/supervisord.conf:12]
- `[program:consumer]` → `python3 manage.py document_consumer` [docker/supervisord.conf:19-20], `user=paperless` [docker/supervisord.conf:21]
- `[program:scheduler]` → `python3 manage.py qcluster` [docker/supervisord.conf:28-29], `user=paperless` [docker/supervisord.conf:30]

The real entry points these use are `src/manage.py` (`execute_from_command_line(sys.argv)` [src/manage.py:11], which launches `qcluster` and `document_consumer`) and `src/paperless/asgi.py` (`application = ProtocolTypeRouter({...})` [src/paperless/asgi.py:17], served by gunicorn).

### Q1.2 Observed startup gate (real `docker/wait-for-redis.py`)

This run executed the repository's **own** Redis gate (not a substitute). Command: `python3 /app/docker/wait-for-redis.py` (see §0.5, step 4). Actual, unedited output (`startup_gate.log`):

```text
Waiting for Redis: redis://localhost:6379
Connected to Redis broker: redis://localhost:6379
```

`migrate` then created the SQLite DB and the four Django-Q schedule rows (see Q2.2).

### Q1.3 Observed supervisord orchestration (all three programs, as `paperless`)

Unlike a debug/direct launch, the three programs were brought up by the repository's **own** `docker/supervisord.conf` via `supervisord` (see §0.5, step 5). supervisord's own log (`supervisord_orchestration.txt`) shows it spawn and supervise all three, and — on the graceful stop (Q5.1) — reap them with exit status 0:

```text
2026-07-13 19:20:23,513 INFO Set uid to user 0 succeeded
2026-07-13 19:20:23,515 INFO supervisord started with pid 444
2026-07-13 19:20:24,517 INFO spawned: 'consumer' with pid 457
2026-07-13 19:20:24,519 INFO spawned: 'gunicorn' with pid 458
2026-07-13 19:20:24,520 INFO spawned: 'scheduler' with pid 459
2026-07-13 19:20:25,629 INFO success: consumer entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-07-13 19:20:25,629 INFO success: gunicorn entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-07-13 19:20:25,629 INFO success: scheduler entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2026-07-13 19:24:25,841 WARN received SIGTERM indicating exit request
2026-07-13 19:24:25,842 INFO waiting for consumer, gunicorn, scheduler to die
2026-07-13 19:24:27,407 INFO stopped: scheduler (exit status 0)
2026-07-13 19:24:27,751 INFO stopped: gunicorn (exit status 0)
2026-07-13 19:24:28,758 INFO stopped: consumer (terminated by SIGTERM)
```

This proves the canonical `supervisord` → { `gunicorn`, `document_consumer`, `qcluster` } supervision was exercised for real, with `supervisord` as root (`Set uid to user 0 succeeded`) and the three children as `paperless` (uid 1001, verified in the process tree in Q5.1).

### Q1.4 Observed Django-Q readiness banner (the "engine is ready" signal)

**On the reliability of the banner under supervisord.** `supervisord.conf` streams each child's stdout to `/dev/stdout` and stderr to `/dev/stderr` [docker/supervisord.conf:14-17], and the `[Q]` logger writes to **stderr**. When both streams are merged into one log, the very first `[Q]` lines (`starting.` + the first workers) can be **reordered** against supervisord's own datestamped stdout lines due to independent buffering; the `running.` banner is always present and is the reliable readiness marker. To show the banner **cleanly and completely**, the block below is taken from Run B — an **isolated** `python3 manage.py qcluster` (the identical command supervisord runs at `docker/supervisord.conf:29`), whose single output stream is not interleaved. Actual, unedited output (`startup_banner_clean.txt`):

```text
18:48:26 [Q] INFO Q Cluster nevada-quebec-two-nuts starting.
18:48:26 [Q] INFO Process-1:1 ready for work at 934
18:48:26 [Q] INFO Process-1:2 ready for work at 935
18:48:26 [Q] INFO Process-1:3 ready for work at 936
18:48:26 [Q] INFO Process-1:4 ready for work at 937
18:48:26 [Q] INFO Process-1:5 ready for work at 938
18:48:26 [Q] INFO Process-1:6 ready for work at 939
18:48:26 [Q] INFO Process-1:7 ready for work at 940
18:48:26 [Q] INFO Process-1:8 ready for work at 941
18:48:26 [Q] INFO Process-1:9 ready for work at 942
18:48:26 [Q] INFO Process-1:10 ready for work at 943
18:48:26 [Q] INFO Process-1:11 ready for work at 944
18:48:26 [Q] INFO Process-1:12 monitoring at 945
18:48:26 [Q] INFO Process-1 guarding cluster nevada-quebec-two-nuts
18:48:26 [Q] INFO Process-1:13 pushing tasks at 946
18:48:26 [Q] INFO Q Cluster nevada-quebec-two-nuts running.
```

Meaning of each line and its source (dependency internals, `django-q==1.3.9`):

- `Q Cluster <id> starting.` — the cluster process begins bootstrapping [django_q/cluster.py:79].
- `Process-1:N ready for work at <pid>` — each **worker** in the pool comes up [django_q/cluster.py:410]. There are **11** here (`Process-1:1` … `Process-1:11`), the CPU-derived worker count (see §0.2). **Environment-specific.**
- `Process-1:12 monitoring at <pid>` — the **monitor** process (collects task results) [django_q/cluster.py:378].
- `Process-1 guarding cluster <id>` — the **guard/sentinel** [django_q/cluster.py:256].
- `Process-1:13 pushing tasks at <pid>` — the **pusher** (dequeues from the broker) [django_q/cluster.py:342].
- `Q Cluster <id> running.` — the **readiness banner**, emitted once after the guard, monitor, pusher and workers are all up [django_q/cluster.py:261]. **This is the "background engine is ready" signal.**

The humanized id `nevada-quebec-two-nuts` is **random per start** (Run A produced `emma-xray-aspen-early`; Run C produced `bulldog-skylark-magnesium-quebec`), and the worker count `11` is **CPU-derived** — both are environment-specific.

### Q1.5 Web-tier readiness (gunicorn) — served, verified by HTTP 200

`gunicorn`'s `when_ready()` hook logs `Server is ready. Spawning workers` [gunicorn.conf.py:17-18], binds `0.0.0.0:8000`, and uses `workers = 2` with `timeout = 120` [gunicorn.conf.py:3-6]. **Under supervisord, gunicorn's readiness lines do not reliably surface in the merged stream**; the authoritative proof that the web tier is *serving* is a live HTTP 200 (Q4 before/after assertions), plus supervisord's `success: gunicorn entered RUNNING state` (Q1.3). For completeness, the readiness hook was also captured directly from an isolated gunicorn invocation (labeled non-canonical port to avoid colliding with the supervised instance):

```text
[2026-07-13 18:47:55 +0000] [609] [INFO] Starting gunicorn 20.1.0
[2026-07-13 18:47:55 +0000] [609] [INFO] Listening at: http://0.0.0.0:8001 (609)
[2026-07-13 18:47:55 +0000] [609] [INFO] Using worker: paperless.workers.ConfigurableWorker
[2026-07-13 18:47:55 +0000] [609] [INFO] Server is ready. Spawning workers
```

### Q1.6 Consumer readiness (document_consumer)

The consumer logs through `paperless.management.consumer` [src/documents/management/commands/document_consumer.py:24] and prints `Using inotify to watch directory for changes: <dir>` [src/documents/management/commands/document_consumer.py:200] once at startup; it is otherwise **silent at idle** (event-driven inotify — see Q5.3).

---


## Q2 — Background processes and scheduled tasks that keep running when idle

There are **two layers**: (a) the **three continuously supervised programs**, and (b) the **four scheduled tasks** that the Django-Q cluster fires automatically.

### Q2.1 The three continuously supervised programs

Per `docker/supervisord.conf` [docker/supervisord.conf:10-30]:

| Program | Command | Role while idle |
|---|---|---|
| `gunicorn` | `gunicorn -c .../gunicorn.conf.py paperless.asgi:application` [docker/supervisord.conf:10-11] | ASGI web server (REST API + websockets); accepts requests but does no background work at idle |
| `document_consumer` | `python3 manage.py document_consumer` [docker/supervisord.conf:19-20] | inotify directory watcher; **silent at idle** (event-driven) |
| `qcluster` (`[program:scheduler]`) | `python3 manage.py qcluster` [docker/supervisord.conf:28-29] | the **Django-Q cluster** — the engine that runs the scheduled tasks below |

### Q2.2 The four scheduled tasks (each by name, with frequency + meaning)

The schedules are registered by database migrations that call `django_q.tasks.schedule(...)`. The **live** `django_q_schedule` rows created by `python3 manage.py migrate` were confirmed as follows (`migrate_schedules.log`).

Command:

```text
$ python3 manage.py shell -c "from django_q.models import Schedule
for s in Schedule.objects.all().order_by('func'): print(s.func, '|', s.schedule_type, '|', s.minutes)"
```

Actual, unedited output:

```text
documents.tasks.index_optimize | D | None
documents.tasks.sanity_check | W | None
documents.tasks.train_classifier | H | None
paperless_mail.tasks.process_mail_accounts | I | 10
```

(`schedule_type` codes: `I` = MINUTES, `H` = HOURLY, `D` = DAILY, `W` = WEEKLY.)

| Task (func) | Name | Frequency | Meaning | Registration citation |
|---|---|---|---|---|
| `paperless_mail.tasks.process_mail_accounts` | "Check all e-mail accounts" | **Every 10 minutes** (`Schedule.MINUTES`, `minutes=10`) | Poll configured IMAP mail accounts and consume attachments | [src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15] |
| `documents.tasks.train_classifier` | "Train the classifier" | **Hourly** (`Schedule.HOURLY`) | Retrain the auto-matching document classifier | [src/documents/migrations/1001_auto_20201109_1636.py:10-14] |
| `documents.tasks.index_optimize` | "Optimize the index" | **Daily** (`Schedule.DAILY`) | Optimize the Whoosh full-text search index | [src/documents/migrations/1001_auto_20201109_1636.py:15-19] |
| `documents.tasks.sanity_check` | "Perform sanity check" | **Weekly** (`Schedule.WEEKLY`) | Verify stored documents/checksums/thumbnails | [src/documents/migrations/1004_sanity_check_schedule.py:10-14] |

The task **bodies** live in `src/documents/tasks.py` (logger `paperless.tasks` [src/documents/tasks.py:29]; `index_optimize` [src/documents/tasks.py:32-35]; `train_classifier` [src/documents/tasks.py:48-72], which early-returns when no `MATCH_AUTO` Tag/DocumentType/Correspondent exists [src/documents/tasks.py:49-55] and logs `"Training data unchanged."` at DEBUG level when the model hash is unchanged [src/documents/tasks.py:69]) and in `src/paperless_mail/tasks.py` (logger `paperless.mail.tasks` [src/paperless_mail/tasks.py:8]; `process_mail_accounts` [src/paperless_mail/tasks.py:11]).

### Q2.3 Observed first scheduler pass (all four fire once at first evaluation)

Because `schedule()` defaults each task's `next_run` to "now", all four are already due at the first scheduler evaluation and fire once, then reschedule to their intervals. From Run C (canonical supervisord; cluster `running.` at `19:20:25`), the first scheduler pass fired at `19:20:55` (see Q3.3 for the ~30 s tick). Actual, unedited output (`first_scheduler_pass.txt`):

```text
19:20:55 [Q] INFO Process-1 created a task from schedule [Train the classifier]
19:20:55 [Q] INFO Process-1 created a task from schedule [Optimize the index]
19:20:55 [Q] INFO Process-1 created a task from schedule [Perform sanity check]
19:20:55 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

Line meanings and sources:

- `Enqueued <id>` — a due schedule was enqueued to the broker [django_q/tasks.py:74].
- `Process-1 created a task from schedule [<name>]` — the scheduler created the task instance from the named schedule [django_q/cluster.py:669].
- `Process-1:N processing [<task-id>]` — a worker picked the task up [django_q/cluster.py:420].
- `Processed [<task-id>]` — the monitor recorded successful completion [django_q/cluster.py:392].
- The bracketed `<task-id>` in the two lines above is a **humanized, randomly-generated identifier assigned per task instance**: Django-Q sets each task's `name` to `tag[0]` from `uuid()` [django_q/tasks.py:38-44] — the humanized form of a fresh random `uuid4()` [django_q/humanhash.py:358-359]. It therefore **varies for every task and every run** (e.g. `moon-uncle-cardinal-music` in Q3.3 below), while the surrounding log-line *structure* — `<name> processing [<task-id>]` [django_q/cluster.py:420] and `Processed [<task-id>]` [django_q/cluster.py:392] — is invariant.

**Canonical body execution.** Because this ran on the full canonical dependency set, the task **bodies executed for real** — task instances reach `Processed [...]` and, e.g., `sanity_check` emits its own `paperless.sanity_checker` line `Sanity checker detected no issues.` [src/documents/sanity_checker.py:27]. (In a *minimal* dependency environment lacking the heavy imports `pdf2image`/`pikepdf`/`pyzbar` [src/documents/tasks.py:23-25], the task modules are not importable and the bodies would fail while the **scheduling** lines — `Enqueued` / `created a task from schedule` / `processing` — remain identical; that failure mode is a **non-canonical environment artifact**, not present here.)

---


## Q3 — Periodic health/readiness log entries (frequency + meaning)

There are **three distinct things** here, and they are easy to conflate. Only one of them is a *periodic log line*; another is a *silent* health signal; the third is a *one-time* banner.

### Q3.1 One-time readiness banner (NOT periodic)

`Q Cluster <id> running.` [django_q/cluster.py:261] is emitted **exactly once**, at the end of startup (see Q1.4). The web-tier equivalent is gunicorn's `Server is ready. Spawning workers` [gunicorn.conf.py:17-18]. These are one-time signals that the system reached a ready state — not periodic heartbeats.

### Q3.2 Silent heartbeat every 0.5 s (a health signal with NO log line) — DIRECTLY measured

The guard loop calls `Stat(self).save()` on **every** cycle [django_q/cluster.py:288], and the cycle length is `GUARD_CYCLE = 0.5` seconds [django_q/conf.py:90] (the loop sleeps `sleep(cycle)` afterwards [django_q/cluster.py:289]). `Stat.save()` writes the cluster's live status into Redis with a **3-second TTL** [django_q/status.py:71-73] via `set_stat` → `redis SET key value ex=3` [django_q/brokers/redis_broker.py:47-48]. The key name is `django_q:paperless:cluster:<uuid>`, assembled by `Stat.get_key()` as `f"{Conf.Q_STAT}:{cluster_id}"` [django_q/status.py:69] from the prefix `Q_STAT = f"django_q:{PREFIX}:cluster"` [django_q/conf.py:174], where `PREFIX` is the cluster name `paperless` [django_q/conf.py:80].

**This is the true continuous "I am alive" signal — but at INFO level it produces NO log line.** It can only be observed in Redis. It was measured **directly** with `redis-cli monitor`, which timestamps every command the guard issues. Command + actual, **complete, unedited** output (`monitor_set.txt` — all eight SET operations captured in the ~4 s `monitor` window; the long opaque base64 value column is the signed cluster-status payload, shown verbatim, and the client port `52592`/cluster UUID are constant because it is the guard's single long-lived connection):

```text
$ timeout 4 redis-cli -p 6379 monitor | grep '"SET" "django_q:paperless:cluster:'
1783970661.367500 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3pnZdsfZqpRWrJ49rJelC0V7ryCl5GClRxmzM7ETdzaZJi_YLggKVkHerc-D4FX8AR49-geIV8G7R68iiD_QZGuhJjzyvny8vPd9uTfzZC5uTReeLm9zta3znb4FDs4Stq_7hA7o4n26Sz3s3NFmJIylG9RjPyL2M2K_IvY7Yn8i9iJmL2P2KmavY_Y5ZpuJwAT0kDAtOQiQY3E8PaC1rPOlc3Jh8Vx24Q1VKabhfqJV4Dw8EWApauAeb7TCfkSbtO_joQ88ZYRUBTeKg9TK0kaGSSNLYh8izIraWRAm9xjbzk2Pra31K0dKMJEK6PF899u1889uvn_7ce35d3H1wbuWHWJqhQKpRE3sa4SzR06cPfSm70DWtn_LqSL05TXhYl7zyV7eGL27lzvVyGJUBxE9XPqvBgxXtp6OS9jZFgBiN3iL7fXSF-yHudKhk7Xv7hsmFoxnIYjtlt6XfCe3ciK8GdiutAXCueLSYLAyKFaXVy4v-zfHWknQhlgrxi5wOzpWMts4WwnPZcmht17YuKEq-_cPS2dmnn6iaiGY64b9v9REzxQ:1wjMGL:k6nPXRwtDVSr4B77PcngCpAp2WC2tZfFaQ3AjIni_hg" "EX" "3"
1783970661.868777 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3plZu-Nsq0grVs8e1svShaLeeqggpeRgpUcZszOxEzebmSYvtF0QFKyCvFufN6_iD_Do0T9AvPoXePQqQvEHmmwt1IRH3pePl_e-Lw9PPZ-NW9OFZ8v7XG_V-XbfAgdnCdu3fUIHdOURPaAednZqMxLG0h3qsR8R-xmxXxH7HbE_EXsVs9cxexOztzH7ErONRGAC9ZAwLTkIkGNxMj2g1azztTM3v3Axu_yOqhTTcD-pdeA8PB1gKRRwj9dbYT-lDdr38cQHnjFC6oIbzUHW2tJ6hkkjS2KfIswK5SwIk3uMbeemx-bm2o1jJZhIDfTsXPf7rUsv7n58_3n15aG4-fhDyw4xtUKD1EIR-xbhzLETF4686TuQyvbvOV2EvlwRLuSKT_byxtS7e7nTjSxGKojo4eJ_NWC4tmo6LmFnSwCI3eAtttdKX7Af5kqHTirf3TdMLBjPQhDbLb0v-XZu5UR4M7Bd1RYIZ4urg8HyoLi-tHxtyb85rrWE2hBrxdgFbkcnSmYaZyvhuSw58tYLGzdUZf_-YfH83MohVfPBXDfs_wXR387j:1wjMGL:GBk81yBbVSaymLvkX6A8yvERgohUATGVgw0ZV66w0OQ" "EX" "3"
1783970662.370078 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3pnZdsfZqpQWrZ49rJelC0V7ryCl5GClRxmzM7ETdzaZJi_YLggKVkHerc-bNxF_gEeP_gEiePIv8OhVBPEHmmwt1IRH3pePl_e-L_dmnszFrenC0-VtrrZ1vtO3wMFZwvZ1n9ABXbxPd6mHnTvajISxdIN67EfEfkbsV8R-R-xPxF7E7GXMXsXsdcw-x2wzEZiAHhKmJQcBciyOpwe0lnW-dE4uLJ7LLryhKsU03E-0CpyHJwIsRQ3c441W2I9ok_Z9PPSBp4yQquBGcZBaWdrIMGlkSexDhFlROwvC5B5j27npsbW1fuVICSZSAT2e7367dv7pzfdvP609_y6uPnjXskNMrVAglaiJfY1w9siJs4fe9B3I2vZvOVWEvrwmXMxrPtnLG6N393KnGlmM6iCih0v_1YDhytbTcQk72wJA7AZvsb1e-oL9MFc6dLL23X3DxILxLASx3dL7ku_kVk6ENwPblbZAOFdcGgxWBsXq8srlZf_mWCsJ2hBrxdgFbkfHSmYbZyvhuSw59NYLGzdUZf_-YenMzLOPVC0Ec92w_xfUZ88W:1wjMGM:5tS_POtKFQHT_mjiLEVrZ_1vRN6Wy4u3ius53oUbyF4" "EX" "3"
1783970662.871374 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3plZu-Nsq5QWrT17WC9LF4p6ryC15GClRxmzM7ETdzaZJi-0XRAUrIK8W583r-IP8OjRP0C8-hd49CqC-ANNthZqwiPvy8fLe9-XB6eezcat6cKz5T2utnW-07fAwVnC9i2f0CFdekj3qYedXW1Gwli6TT32I2I_I_YrYr8j9idiL2P2KmavY_YmZp9jtpkITEAPCdOSgwA5FifTQ1rLOl86cwuLF7KLb6lKMQ33E60C5-HpAEtRA_d4oxX2E9qkAx-PfeAZI6QquFEcpFaWNjJMGlkS-xhhVtTOgjC5x9h2bnpsba1fO1aCiVRAT-e7324uP7_z4d2ntRffxfVH71t2iKkVCqQSNbGvEc4cO3H-yJu-A1nb_l2nitCX14SLec0n-3lj9N5-7lQji1EdRPRw6b8aMFzZejouYWdbAIi94C2210tfcBDmSodO1r67b5hYMJ6FILZbel_yndzKifBmYLvSFghni8uDweqguLqyemXFvznWSoI2xFoxdoHb0YmSmcbZSnguS4689cLGDVXZv39YOjd3Y5mqhWCuG_b_AsjnzhE:1wjMGM:wjYfJhmbhI5I8TqVP2lpsaLz3JEjiBmFfBqy9pXVKyo" "EX" "3"
1783970663.372667 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3pnZdsfZVZEWWj17WC9LF4r1XkFKycFKjzJmZ2InbjYzTV6wXRAUrIK8W583r-IP0JtH_wDx6l_g0asI4g802VqoCY-8Lx8v731f7s897cWt2cKz5R2ud-p8d2CBg7OE7Rs-oUO69IDuUR87d2szFsbSTeqzHxH7GbFfEfsdsT8RexGzlzF7FbPXMfscs61EYAL1iDAtOQiQE3EyPaT1rPOlc3ph8Xx28Q1VKabhflrrwHl4KsBSKOAeb7bCfkxbdODjkQ88Y4TUBTeag6y1pc0Mk0aWxD5GmBXKWRAm9xjbzs2O7e2Nq8dKMJEa6Mm57rfrF57d-vDu0_rz7-Law_ctO8LUCg1SC0Xsa4Tzx04sHXkzcCCVHdx2ugh9uSJczBWf7ueNqff2c6cbWYxVENHH5f9qwHBt1Wxcws6OABB7wVtsb5S-4CDMlY6cVL67b5hYMJ6FILZbel_y3dzKqfBmYLuqLRD2isvD4eqwuLKyurbi35zUWkJtiLVi7AK34xMl842zlfBclhx564VNGqqyf_-wvDT3tkfVQjDXjQZ_AcyjzmA:1wjMGN:VVzdtzZmqf2_EPr5Yzb1IbVeS1S-72nnGK3dwqpwlEA" "EX" "3"
1783970663.873927 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3plZu-Nsq5QWWgVvHtbL0oWi3itIKTlY7VHG7EzsxM0m08kLbRcEBasg79bnzav4Azx69A8Qr_4FHr2KIP5Ak62FmvDI-_Lx8t735cGpZ7Nxa7rwbHmP622T7_QtcHCWsH3TJ3RIlx7SfephZ9c0I9FYuk099iNiPyP2K2K_I_YnYi9j9ipmr2P2JmafY7aZCEzADAnTkoMAORYn00NayzpfOnMLi-eyi2-pSjEN9xOjA-fh6QBLoYB7vNEK-wlt0oGPxz7wTCOkLnijOUijLW1kmNSyJPYxwqxQzoJoco-x7dz02Npav3asBBOpgZ7Od7_dOP_8zod3n9ZefBfXH71v2SGmVmiQWihiXyOcOXZi6cibvgOpbP-u00XoyxXhYq74ZD-vG7O3nztdy2KkgogeLv9XAw3XVk3HJexsCwCxF7zF9nrpCw7CXOnQSeW7-4aJhcazEMR2S-9LvpNbORHeDGxXxgLhbHF5MFgdFFdXVq-s-DfHRkswDbFWjF3gdnSiZKZ2thKey5Ijb72wcU1V9u8flpfmbl2gaiGY64b9v8mOzh8:1wjMGN:SWQuY_EifTNQO0reMeV5Mob9I_aYn7U7rt1PGHm7a3Q" "EX" "3"
1783970664.375228 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3pnZdsfZqkgLrp49rJelC8V6ryCl5GClRxmzM7ETN5tMJy_YLggKVkHerc-bV_EHeBRP_gHi1b_Ak3gVQfyBJlsLNeGR9-Xj5b3vy725Jwtxa7bwdHmb622T7wwscHCWsH3dJ3RAF-_TXepj545pxqKxdIP67EfEfkbsV8R-R-xPxJ7H7EXMXsbsVcw-xWwzEZiAGRGmJQcBciKOpwe0lnW-dE4uLp3LLrymKsU03E-NDpyHJwIshQLu8UYr7Ee0Sfs-HvrAU42QuuCN5iCNtrSRYVLLktiHCLNCOQuiyT3GtnOzY2tr_cqREkykBnp8pvvt2vmnN9-_-bj27Lu4-uBdy44wtUKD1EIR-xrh_JETZw-9GTiQyg5uOV2EvlwRLuWKT_fyujG7e7nTtSzGKojoY--_Gmi4tmo2LmFnWwCI3eAtttdLX7Af5kpHTirf3TdMLDSehSC2W3pf8p3cyqnwZmC7MhYIF4pLw-HKsLi8vLK67N-cGC3BNMRaMXaB2_Gxkvna2Up4LksOvfXCJjVV2b9_6PXm3n6majGY60aDv9aRz0Y:1wjMGO:PurBXtynE7Qbxh5oJjNp6ulm7jQiv_TqTHFqA1FtVMg" "EX" "3"
1783970664.876521 [0 127.0.0.1:52592] "SET" "django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b" ".eJxVUc9rFDEU3plZu-Nsq0gLrp49rB6WLpTqvYKUkoOVHnXMzsRO3GwynbxguyAoWAV5tz5vXsUf4NGjf4B49S_w6FUE8QeabC3UhEfel4-X974vD048m49bs4Wny7tcb5t8Z2CBg7OE7Rs-oQO6-JDuUx8790wzFo2lm9RnPyL2M2K_IvY7Yn8i9jJmr2L2OmZvYvY5ZpuJwATMiDAtOQiQE3E8PaC1rPOls7C4dC678JaqFNNwPzU6cB6eDLAUCrjHG62wn9Am7ft47ANPNULqgjeagzTa0kaGSS1LYh8jzArlLIgm9xjbzs2Ora31q0dKMJEa6OmZ7rfr55_f_vDu09qL7-Lao_ctO8LUCg1SC0Xsa4RzR06cPfRm4EAqO7jjdBH6ckW4lCs-3cvrxuzu5U7XshirIKKPvf9qoOHaqtm4hJ1tASB2g7fYXi99wX6YKx05qXx33zCx0HgWgthu6X3Jd3Irp8Kbge3KWCCcL1aHw5VhcWV55fKyf3NitATTEGvF2AVux8dK5mpnK-G5LDn01gub1FRl__6h11u4dYmqxWCuGw3-Asqjzjc:1wjMGO:mM9BpU96lkffj1smPD_02eanMBYWkMza-mRT2FWzVVk" "EX" "3"
```

The seven consecutive inter-SET deltas are all `0.501 s` — a **direct** measurement of the `GUARD_CYCLE = 0.5 s` write cadence [django_q/conf.py:90], and each write carries the `EX 3` (3-second) TTL exactly as `Stat.save` sets it [django_q/status.py:73]. **Meaning:** the health signal exists and is continuous, but it is a Redis key write, not a periodic log message.

### Q3.3 Scheduler evaluation tick ~30 s (MEASURED, stable across 3 starts)

Within the same guard loop, the scheduler is evaluated only when a counter reaches 30: `counter += cycle` [django_q/cluster.py:283] and `if counter >= 30 and Conf.SCHEDULER:` → `scheduler(broker=self.broker)` [django_q/cluster.py:284-286]. Because `counter` increments by `GUARD_CYCLE = 0.5` each cycle, the scheduler pass fires after ~60 cycles ≈ **~30 seconds**.

**Measured, and stable across three independent starts:**

| Run | Entry point | Cluster `running.` banner | First scheduler pass (`created a task from schedule`) | Δ |
|---|---|---|---|---|
| A | canonical supervisord | `18:44:33` | `18:45:02` | **29 s** |
| B | isolated `manage.py qcluster` | `18:48:26` | `18:48:56` | **30 s** |
| C | canonical supervisord | `19:20:25` | `19:20:55` | **30 s** |

The cadence is **~30 s** (theoretical ~29.5–30 s given the 0.5 s granularity; the 29 s value on Run A is the same tick minus one 0.5 s cycle of rounding). This tick is normally **silent** — it only emits log lines when a schedule is actually due.

### Q3.4 The most-frequent VISIBLE periodic idle line = the 10-minute mail check (TWO independent runs)

Once the initial burst subsides, the most frequent *visible* periodic INFO line at idle is the **"Check all e-mail accounts"** firing. The underlying interval is **exactly 10 minutes** (`minutes=10` [src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15]). Mechanically, when the schedule fires the scheduler shifts its `next_run` by +10 min, repeating until the result is in the future because `catch_up=False` [src/paperless/settings.py:451] — `if s.schedule_type == s.MINUTES: next_run = next_run.shift(minutes=+(s.minutes or 1))` [django_q/cluster.py:615-616], looping while `Conf.CATCH_UP or next_run > arrow.utcnow()` [django_q/cluster.py:639], and only firing schedules whose `next_run < now()` [django_q/cluster.py:589].

**Measured over two fully independent ~22-minute idle runs — three consecutive firings each (unedited):**

Run A (canonical supervisord, cluster `emma-xray-aspen-early`), from `run_a.log`:

```text
18:45:02 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
18:54:34 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:04:35 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

Run B (isolated `manage.py qcluster`, cluster `nevada-quebec-two-nuts`, separate Redis + SQLite), from `run_b.log`:

```text
18:48:56 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
18:58:27 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
19:08:28 [Q] INFO Process-1 created a task from schedule [Check all e-mail accounts]
```

Observed gaps (identical distribution across the two independent runs):

| Run | 1st → 2nd gap | 2nd → 3rd gap |
|---|---|---|
| A | 9 m 32 s | **10 m 01 s** |
| B | 9 m 31 s | **10 m 01 s** |

- The **2nd → 3rd gap is 10 m 01 s in both runs** — a clean, reproduced demonstration of the exact 10-minute period (the extra ~1 s is the 0–30 s scheduler-tick quantization landing on the next tick after the due-time).
- The **1st → 2nd gap is shorter (~9 m 31–32 s)** in both runs for the same reason: the very first firing is **late** — the schedule's first due-time is anchored at the migration/registration instant, which is *before* the cluster start, so the first firing slips to the first ~30 s scheduler tick after start, compressing the first observed interval. This is **observed, reproduced across two runs**, not inferred.

**Meaning:** a `created a task from schedule [Check all e-mail accounts]` line roughly every 10 minutes is the normal "still alive and scheduling" heartbeat visible in the logs at idle; `train_classifier` (hourly), `index_optimize` (daily) and `sanity_check` (weekly) appear far less often.

### Q3.5 Worker recycling lines are NORMAL (not errors)

`Q_CLUSTER["recycle"] = 1` [src/paperless/settings.py:452] means **each worker is recycled after every single task**. So after any task you will periodically see this contiguous sequence (unedited, from Run C `q4_run.log` lines 46-48) — a task completes, the worker that ran it is recycled, and a fresh worker takes its place:

```text
19:20:55 [Q] INFO Processed [moon-uncle-cardinal-music]
19:20:55 [Q] INFO recycled worker Process-1:1
19:20:55 [Q] INFO Process-1:14 ready for work at 679
```

Sources: `Processed [<id>]` [django_q/cluster.py:392]; `recycled worker <name>` [django_q/cluster.py:232]; `ready for work at <pid>` [django_q/cluster.py:410]; the underlying `stopped doing work` line is [django_q/cluster.py:451]. The guard also reincarnates workers "after timeout" [django_q/cluster.py:230] or "after death" [django_q/cluster.py:234], so occasional worker respawn lines at idle are expected and are **not** error conditions.

---


## Q4 — Reconnection / "operational again" logs (interrupt + restart the live Redis broker)

Redis is the natural component to interrupt: it is the single shared dependency for idle operation (Django-Q broker **and** Channels backend — see Q5). The interruption and restart were performed against the **live** broker, not a mock. This section shows the state **before**, **during**, and **after** the outage, with active assertions on each side.

### Q4.1 BEFORE the outage — active baseline (system operational)

While the Run C cluster was idle and Redis was up, three active round-trips were exercised through the **real** entry points (`assert_before.txt`):

```text
== A. HTTP (real gunicorn ASGI, urllib GET /api/) ==
HTTP_STATUS=200 url=http://127.0.0.1:8000/api/ body_prefix=b'{"correspondents":"http://127.0.0.1:8000/api/correspondents/'

== B. Channels round-trip (channels_redis send/receive) ==
CHANNELS_RECV={'type': 'probe', 'payload': 'channels-ok-before'}

== C. Django-Q broker round-trip (async_task math.sqrt(16) -> result) ==
BROKER_ENQUEUED task_id=74d65dca4f324332b47b6c8ed19c27d7 func=math.sqrt arg=16.0
19:21:35 [Q] INFO Enqueued 1
BROKER_RESULT=4.0
```

All three succeed: HTTP **200** from gunicorn, a Channels send/receive round-trip via `channels_redis`, and a full Django-Q enqueue→dequeue→execute→result (`math.sqrt(16)` → `4.0`, with the `[Q] INFO Enqueued 1` line proving it transited Redis). This is the "operational" baseline.

### Q4.2 Interrupt — what "NOT operational" looks like

Command (executed while the Run C cluster was running), with the precise shutdown timestamp recorded (`outage_ts.txt`):

```text
$ date +%s.%N                     # SHUTDOWN_TS
1783970524.633453278              # = 19:22:04
$ redis-cli -p 6379 shutdown nosave
$ redis-cli -p 6379 ping
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

The moment Redis goes away, two things happen in the logs.

**(a) A steady ~2 lines/second `[Q] ERROR` burst — the dominant "NOT operational" signal.** Unedited sample (`q4_run.log` lines 149-154):

```text
19:22:04 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:05 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:05 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:06 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:07 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:07 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
```

Measured over the **exact** outage window (`outage_burst.txt`), with the producing command:

```text
$ grep -cE '\[Q\] ERROR Error [0-9]+ connecting to localhost:6379' q4_run.log
70
$ # first / last genuine error line
19:22:04 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.   (line 149)
19:22:39 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.   (line 521)
```

So **70 genuine connection-error lines over a 35.18 s outage** (shutdown `19:22:04.633`, restart `19:22:39.815` — see Q4.3) = **≈ 1.99 lines/second** (approximately 2/s; the count is exact, the rate is derived). This ≈2/s cadence matches `GUARD_CYCLE = 0.5 s` [django_q/conf.py:90], because the **source** is the guard's own heartbeat: every 0.5 s the guard calls `Stat(self).save()` [django_q/cluster.py:288], which tries to write to Redis and fails, and the failure is logged by `Stat.save`'s handler `except Exception as e: logger.error(e)` [django_q/status.py:74-75]. (**Observed source attribution**, corroborated by the ≈2/s rate == guard cycle; the function performing the work is `Stat.save`. Run A independently produced 234 such lines over a 117.35 s outage = ≈1.99/s — the same rate.)

- **The errno is host-specific.** Here it is **errno 111 ("Connection refused")** because the broker's port was closed by `shutdown`. On other hosts the same burst can appear as **errno 99 ("Cannot assign requested address")** or similar; only the errno/text differs, not the mechanism.

**(b) A multi-line `--- Logging error ---` traceback each time the pusher's `blpop` fails.** The **pusher** blocks on `blpop` [django_q/brokers/redis_broker.py:21] inside `broker.dequeue()` [django_q/cluster.py:345]; when Redis drops, that raises `redis.exceptions.ConnectionError: Connection closed by server.`, which the pusher logs with `logger.error(e, traceback.format_exc())` [django_q/cluster.py:347]. The extra positional argument (`traceback.format_exc()`) collides with Python's `logging` formatting (`msg % args`) and produces a `--- Logging error ---` report. The **complete, byte-faithful** block (all 89 lines, including the trailing `Call stack:`, `Message:`, and `Arguments:` sections that a truncated excerpt would omit) is (`logging_error_ONE.txt`):

```text
--- Logging error ---
Traceback (most recent call last):
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 345, in pusher
    task_set = broker.dequeue()
  File "/usr/local/lib/python3.9/site-packages/django_q/brokers/redis_broker.py", line 21, in dequeue
    task = self.connection.blpop(self.list_key, 1)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 1900, in blpop
    return self.execute_command('BLPOP', *keys)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 901, in execute_command
    return self.parse_response(conn, command_name, **options)
  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 915, in parse_response
    response = connection.read_response()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 739, in read_response
    response = self._parser.read_response()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 470, in read_response
    self.read_from_socket()
  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 429, in read_from_socket
    raise ConnectionError(SERVER_CLOSED_CONNECTION_ERROR)
redis.exceptions.ConnectionError: Connection closed by server.

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.9/logging/__init__.py", line 1083, in emit
    msg = self.format(record)
  File "/usr/local/lib/python3.9/logging/__init__.py", line 927, in format
    return fmt.format(record)
  File "/usr/local/lib/python3.9/logging/__init__.py", line 663, in format
    record.message = record.getMessage()
  File "/usr/local/lib/python3.9/logging/__init__.py", line 367, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  File "/app/src/manage.py", line 11, in <module>
    execute_from_command_line(sys.argv)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/__init__.py", line 446, in execute_from_command_line
    utility.execute()
  File "/usr/local/lib/python3.9/site-packages/django/core/management/__init__.py", line 440, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/base.py", line 414, in run_from_argv
    self.execute(*args, **cmd_options)
  File "/usr/local/lib/python3.9/site-packages/django/core/management/base.py", line 460, in execute
    output = self.handle(*args, **options)
  File "/usr/local/lib/python3.9/site-packages/django_q/management/commands/qcluster.py", line 22, in handle
    q.start()
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 78, in start
    self.sentinel.start()
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 121, in start
    self._popen = self._Popen(self)
  File "/usr/local/lib/python3.9/multiprocessing/context.py", line 224, in _Popen
    return _default_context.get_context().Process._Popen(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/context.py", line 277, in _Popen
    return Popen(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/popen_fork.py", line 19, in __init__
    self._launch(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/popen_fork.py", line 71, in _launch
    code = process_obj._bootstrap(parent_sentinel=child_r)
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 315, in _bootstrap
    self.run()
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 108, in run
    self._target(*self._args, **self._kwargs)
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 168, in __init__
    self.start()
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 172, in start
    self.spawn_cluster()
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 248, in spawn_cluster
    self.pusher = self.spawn_pusher()
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 201, in spawn_pusher
    return self.spawn_process(pusher, self.task_queue, self.event_out, self.broker)
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 197, in spawn_process
    p.start()
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 121, in start
    self._popen = self._Popen(self)
  File "/usr/local/lib/python3.9/multiprocessing/context.py", line 224, in _Popen
    return _default_context.get_context().Process._Popen(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/context.py", line 277, in _Popen
    return Popen(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/popen_fork.py", line 19, in __init__
    self._launch(process_obj)
  File "/usr/local/lib/python3.9/multiprocessing/popen_fork.py", line 71, in _launch
    code = process_obj._bootstrap(parent_sentinel=child_r)
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 315, in _bootstrap
    self.run()
  File "/usr/local/lib/python3.9/multiprocessing/process.py", line 108, in run
    self._target(*self._args, **self._kwargs)
  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 347, in pusher
    logger.error(e, traceback.format_exc())
Message: ConnectionError('Connection closed by server.')
Arguments: ('Traceback (most recent call last):\n  File "/usr/local/lib/python3.9/site-packages/django_q/cluster.py", line 345, in pusher\n    task_set = broker.dequeue()\n  File "/usr/local/lib/python3.9/site-packages/django_q/brokers/redis_broker.py", line 21, in dequeue\n    task = self.connection.blpop(self.list_key, 1)\n  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 1900, in blpop\n    return self.execute_command(\'BLPOP\', *keys)\n  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 901, in execute_command\n    return self.parse_response(conn, command_name, **options)\n  File "/usr/local/lib/python3.9/site-packages/redis/client.py", line 915, in parse_response\n    response = connection.read_response()\n  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 739, in read_response\n    response = self._parser.read_response()\n  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 470, in read_response\n    self.read_from_socket()\n  File "/usr/local/lib/python3.9/site-packages/redis/connection.py", line 429, in read_from_socket\n    raise ConnectionError(SERVER_CLOSED_CONNECTION_ERROR)\nredis.exceptions.ConnectionError: Connection closed by server.\n',)
```

This artifact is inherent to django-q 1.3.9's `logger.error(e, traceback.format_exc())` call [django_q/cluster.py:347] on the **canonical Python 3.9** runtime — it is not a Python-version quirk. The clean `[Q] ERROR Error N connecting to ...` burst (part a) is the reliable "NOT operational" signal.

### Q4.3 Restart — what "operational again" looks like

Command, with the precise restart timestamp recorded (`outage_ts.txt`):

```text
$ date +%s.%N                     # RESTART_TS
1783970559.815422746              # = 19:22:39
$ redis-server --daemonize yes --bind 127.0.0.1 --protected-mode yes --port 6379 \
    --save "" --appendonly no --dir /tmp --pidfile /tmp/redis6379.pid
$ redis-cli -p 6379 ping
PONG
```

**There is NO literal "reconnected" banner in Django-Q 1.3.9.** A reader waiting for a "reconnected" message would be misled. Moreover, **`pushing tasks at <pid>` by itself is NOT proof of reconnection**: during the outage the guard reincarnates the dead pusher roughly every 10 s, so `stopped pushing tasks` → `reincarnated pusher … after sudden death` → `pushing tasks at <pid>` appears **repeatedly while Redis is still down** (the freshly-spawned pusher immediately fails its next `blpop`). Recovery must therefore be confirmed by (a) the **cessation** of the connection-error burst, (b) the reincarnation that *sticks*, and — decisively — (c) **active round-trips** succeeding again.

**(a) The error burst STOPS (bounded zero-error window).** The last genuine connection error is at `19:22:39` (line 521), immediately after the restart timestamp; counting genuine errors after that line (`recovery_signature.txt` / producing command) yields **zero**:

```text
$ awk 'NR>521' q4_run.log | grep -cE '\[Q\] ERROR Error [0-9]+ connecting to localhost:6379'
0
```

**(b) The guard reincarnates the pusher, which then stays up.** Unedited (`recovery_signature.txt`):

```text
19:22:39 [Q] ERROR Error 111 connecting to localhost:6379. Connection refused.
19:22:45 [Q] INFO Process-1:21 stopped pushing tasks
19:22:45 [Q] ERROR reincarnated pusher Process-1:21 after sudden death
19:22:45 [Q] INFO Process-1:22 pushing tasks at 778
```

Sources: `stopped pushing tasks` [django_q/cluster.py:366]; `reincarnated pusher <name> after sudden death` [django_q/cluster.py:223], driven by the guard's pusher-liveness check `if not self.pusher.is_alive(): self.reincarnate(self.pusher)` [django_q/cluster.py:280-281]; `pushing tasks at <pid>` [django_q/cluster.py:342]. `reincarnated pusher … after sudden death` is logged at ERROR, but in the recovery context it is the **success** signal that the pusher was brought back and — this time, with Redis live — **stays** up.

**(c) Active round-trips succeed again (decisive proof).** After the restart, the same three real-entry-point assertions from Q4.1 were re-run, plus process-liveness and heartbeat checks (`assert_after.txt`):

```text
== A. Django-Q broker round-trip (async_task math.sqrt(81) -> result) ==
BROKER_ENQUEUED task_id=252d62cd58db45ba988420103556b166 func=math.sqrt arg=81.0
19:23:56 [Q] INFO Enqueued 1
BROKER_RESULT=9.0

== B. Channels round-trip (channels_redis send/receive) ==
CHANNELS_RECV={'type': 'probe', 'payload': 'channels-ok-after-recovery'}

== C. HTTP (real gunicorn ASGI, urllib GET /api/) ==
HTTP_STATUS=200 url=http://127.0.0.1:8000/api/ body_prefix=b'{"correspondents":"http://127.0.0.1:8000/api/correspondents/'

== D. Consumer + all 3 programs liveness (children of supervisord 444, as paperless) ==
pid=457 ALIVE cmd=python3 manage.py document_consumer  uid=1001
pid=458 ALIVE cmd=/usr/local/bin/python3.9 /usr/local/bin/gunic uid=1001
pid=459 ALIVE cmd=python3 manage.py qcluster  uid=1001

== E. Heartbeat resumed: cluster Stat key present with TTL (source-derived 3s TTL) ==
stat_key=django_q:paperless:cluster:bd844765-f8a5-43d4-b5c9-609c1a51f50b
PTTL_ms=2656
```

This is the decisive evidence of "operational again": a full Django-Q broker round-trip (`math.sqrt(81)` → `9.0`, with a fresh `[Q] INFO Enqueued 1`), a Channels send/receive, HTTP **200**, all three supervised programs still alive as `paperless` (uid 1001), and the silent 0.5 s heartbeat (Q3.2) resumed (the cluster Stat key is present with `PTTL_ms=2656`, i.e. ~2.66 s of its 3 s TTL remaining, so the guard is again writing every 0.5 s).

So the complete "operational again" signature is: **the `[Q] ERROR … connecting to …` burst ceases (bounded zero-error window)**, the sequence **`stopped pushing tasks` → `reincarnated pusher … after sudden death` → `pushing tasks at <pid>`** that *sticks* appears, **and active broker/Channels/HTTP round-trips succeed** with the heartbeat resumed. (For completeness: `docker/wait-for-redis.py`'s `Connected to Redis broker: <url>` [docker/wait-for-redis.py:41] is a *startup-gate* message only — it does **not** appear during live recovery.)

---


## Q5 — Components that run continuously to keep the system "ready" at idle

Even with **zero** document activity, the following run continuously.

### Q5.1 The idle process tree (observed, with producing command)

The tree below was produced by a small `/proc` walker (`ps` is absent in the image), saved as `proctree.sh` and cited here as the producing command:

```bash
# proctree.sh — print PID PPID USER CMD for the supervised stack (walks /proc)
for p in /proc/[0-9]*; do
  pid=$(basename "$p"); ppid=$(awk '/^PPid:/{print $2}' "$p/status")
  uid=$(awk '/^Uid:/{print $2}' "$p/status")
  case "$uid" in 0) user=root;; 1001) user=paperless;; *) user="uid$uid";; esac
  cmd=$(tr '\0' ' ' < "$p/cmdline")
  case "$cmd" in *supervisord*|*qcluster*|*document_consumer*|*asgi:application*|*redis-server*)
    printf "%-6s %-6s %-10s %s\n" "$pid" "$ppid" "$user" "$cmd";; esac
done | sort -n
```

Actual, unedited output (Run C, `proctree.txt`):

```text
PID    PPID   USER       CMD
40     1      root       redis-server 127.0.0.1:6379
444    0      root       /usr/bin/python3 /usr/bin/supervisord -c /app/docker/supervisord.conf
457    444    paperless  python3 manage.py document_consumer
458    444    paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
459    444    paperless  python3 manage.py qcluster
468    458    paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
469    458    paperless  /usr/local/bin/python3.9 /usr/local/bin/gunicorn -c /usr/src/paperless/gunicorn.conf.py paperless.asgi:application
499    459    paperless  python3 manage.py qcluster
500    499    paperless  python3 manage.py qcluster
501    499    paperless  python3 manage.py qcluster
502    499    paperless  python3 manage.py qcluster
503    499    paperless  python3 manage.py qcluster
504    499    paperless  python3 manage.py qcluster
505    499    paperless  python3 manage.py qcluster
506    499    paperless  python3 manage.py qcluster
507    499    paperless  python3 manage.py qcluster
508    499    paperless  python3 manage.py qcluster
509    499    paperless  python3 manage.py qcluster
511    499    paperless  python3 manage.py qcluster
512    499    paperless  python3 manage.py qcluster
513    499    paperless  python3 manage.py qcluster
```

Reading the tree: **supervisord** (pid 444, root) supervises the three programs (pids 457/458/459, all `paperless`); **gunicorn** master (458) has **2** workers (468, 469) matching `workers = 2` [gunicorn.conf.py:4]; the **qcluster** launcher (459) forks the **guard** "Process-1" (499), which in turn owns **11 workers** (500-509, 511), the **monitor** (512), and the **pusher** (513). The worker count (11) and PIDs are environment-specific; the **shape** (1 guard + N workers + 1 monitor + 1 pusher under a launcher) is invariant.

### Q5.2 The Django-Q cluster (guard loop) runs continuously

The guard runs the loop at [django_q/cluster.py:253-290] continuously (every 0.5 s), checking worker liveness, monitor liveness [django_q/cluster.py:277], and pusher liveness [django_q/cluster.py:280-281], ticking the scheduler ~every 30 s [django_q/cluster.py:283-286], and writing the heartbeat [django_q/cluster.py:288]. A graceful stop (SIGTERM to `supervisord`'s numeric PID, which stops `qcluster`) produces the **complete** sequence below — no ellipsis, every worker's `stopped doing work` line shown (Run C, `shutdown_sequence.txt`):

```text
19:24:25 [Q] INFO Q Cluster bulldog-skylark-magnesium-quebec stopping.
19:24:25 [Q] INFO Process-1 stopping cluster processes
19:24:26 [Q] INFO Process-1:22 stopped pushing tasks
19:24:26 [Q] INFO Process-1:7 stopped doing work
19:24:26 [Q] INFO Process-1:8 stopped doing work
19:24:26 [Q] INFO Process-1:9 stopped doing work
19:24:26 [Q] INFO Process-1:10 stopped doing work
19:24:26 [Q] INFO Process-1:11 stopped doing work
19:24:26 [Q] INFO Process-1:14 stopped doing work
19:24:26 [Q] INFO Process-1:15 stopped doing work
19:24:26 [Q] INFO Process-1:16 stopped doing work
19:24:26 [Q] INFO Process-1:17 stopped doing work
19:24:26 [Q] INFO Process-1:18 stopped doing work
19:24:26 [Q] INFO Process-1:23 stopped doing work
19:24:27 [Q] INFO Process-1 waiting for the monitor.
19:24:27 [Q] INFO Process-1:12 stopped monitoring results
19:24:27 [Q] INFO Q Cluster bulldog-skylark-magnesium-quebec has stopped.
```

Sources: `stopping.` [django_q/cluster.py:87]; `stopping cluster processes` [django_q/cluster.py:295]; `stopped pushing tasks` [django_q/cluster.py:366]; `stopped doing work` [django_q/cluster.py:451]; `waiting for the monitor.` [django_q/cluster.py:320]; `stopped monitoring results` [django_q/cluster.py:396]; `has stopped.` [django_q/cluster.py:90]. (The worker names shown — `:7`–`:11`, `:14`–`:18`, `:23` — are the *live* recycled worker generation at stop time, which is why they are not a contiguous `:1`–`:11`; 11 workers stop, consistent with the CPU-derived count.)

### Q5.3 gunicorn, document_consumer, Redis, supervisord, database

- **gunicorn** (web/ASGI) stays up continuously under supervisord [docker/supervisord.conf:10-11]: a master plus **2** workers (`workers = 2`), bind `0.0.0.0:8000`, `timeout = 120` [gunicorn.conf.py:3-6]. It serves the REST API and the Channels websocket used for live status but performs no background work at idle.
- **document_consumer** (inotify watcher) stays up continuously [docker/supervisord.conf:19-20]. It watches the consumption directory using **event-driven inotify** and is therefore **silent at idle** (its only line is the startup message in Q1.6). It switches to periodic polling **only** if `PAPERLESS_CONSUMER_POLLING > 0`: the command branches on `if settings.CONSUMER_POLLING == 0 and INotify:` → `handle_inotify(...)`, `else: handle_polling(...)` [src/documents/management/commands/document_consumer.py:178-181], and that setting's runtime default is **0** — `CONSUMER_POLLING = int(os.getenv("PAPERLESS_CONSUMER_POLLING", 0))` [src/paperless/settings.py:478] — matching the **commented** example (`#PAPERLESS_CONSUMER_POLLING=10`) [paperless.conf.example:60], so by default there is no polling activity.
- **Redis** must run continuously for idle operation because it is used simultaneously as the **Django-Q broker** — `Q_CLUSTER["redis"]` [src/paperless/settings.py:456] — and the **Channels websocket layer backend** — `CHANNEL_LAYERS` → `channels_redis.core.RedisChannelLayer` [src/paperless/settings.py:178-187]. This shared role is exactly why interrupting Redis (Q4) exercises the reconnection path.
- **supervisord** runs in the foreground (`nodaemon=true` [docker/supervisord.conf:2]) and supervises the three programs above, piping each program's stdout/stderr to the container's streams [docker/supervisord.conf:14-17]. It is the mechanism that keeps everything "ready".
- **Database:** the default is **SQLite**; PostgreSQL is used **only** when `PAPERLESS_DBHOST` is set [paperless.conf.example:11-16], which also gates the `wait_for_postgres` startup step [docker/docker-prepare.sh:67-69]. Idle behavior does not require PostgreSQL.

**Summary of what keeps running at idle:** `supervisord` → { `gunicorn` (master + 2 workers), `document_consumer` (silent inotify), `qcluster` (guard + monitor + pusher + 11 workers) } + `redis-server` (broker + Channels) + the SQLite DB file. The only components doing anything *periodically* are the Django-Q guard (0.5 s silent heartbeat, ~30 s scheduler tick) and, visibly, the 10-minute mail-check schedule.

---

## Citation quick-reference

**Repository source (at commit `542221a38dff`):**

- `Dockerfile` — 18 (`python:3.9-slim-bullseye`), 168 (`ENTRYPOINT`), 172 (`CMD supervisord`)
- `docker/supervisord.conf` — 2 (`nodaemon=true`), 8 (`user=root`), 10-11 gunicorn + 12 (`user=paperless`), 14-17 (stdout/stderr → /dev/stdout,/dev/stderr), 19-20 consumer + 21 (`user=paperless`), 28-29 qcluster + 30 (`user=paperless`)
- `docker/docker-entrypoint.sh` — 37 (`gosu paperless /sbin/docker-prepare.sh`)
- `docker/docker-prepare.sh` — **66-79 `do_work` body**; 67-69 postgres-if-`DBHOST`; 71 `wait_for_redis`; 73 migrations; 75 search_index; 77 superuser; **81 `do_work` invocation**
- `docker/wait-for-redis.py` — 16 `MAX_RETRY_COUNT=5`, 17 `RETRY_SLEEP_SECONDS=5`, 21 "Waiting for Redis", 38 "Failed to connect to", 41 "Connected to Redis broker"
- `gunicorn.conf.py` — 3-6 (bind `0.0.0.0:8000` / `workers=2` / `worker_class` / `timeout=120`), 17-18 "Server is ready. Spawning workers"
- `src/paperless/settings.py` — 178-187 `CHANNEL_LAYERS` (RedisChannelLayer 180, hosts 182), 427-435 `default_task_workers()`, 438 `TASK_WORKERS`, 440 `PAPERLESS_WORKER_TIMEOUT=1800`, 449-457 `Q_CLUSTER` (450 name, 451 `catch_up=False`, 452 `recycle=1`, 455 `workers`, 456 `redis`); LOGGING dict = console/file_paperless/file_mail only (no DB handler)
- `src/manage.py` — 11 `execute_from_command_line`; `src/paperless/asgi.py` — 17 `application = ProtocolTypeRouter`
- `paperless.conf.example` — 10 `PAPERLESS_REDIS` (commented), 11-16 DB vars (commented), 57 `PAPERLESS_TASK_WORKERS=1` (commented), 60 `PAPERLESS_CONSUMER_POLLING=10` (commented)
- `src/setup.cfg` — 12 `PAPERLESS_DISABLE_DBHANDLER=true` (pytest env only; no runtime effect)
- `requirements.txt` — 37 `django-q==1.3.9`, 38 `django==4.0.4` (no celery)
- `src/documents/tasks.py` — 23-25 heavy imports, 29 logger `paperless.tasks`, 32-35 `index_optimize`, 48-72 `train_classifier` (49-55 early-return, 69 "Training data unchanged.")
- `src/paperless_mail/tasks.py` — 8 logger `paperless.mail.tasks`, 11 `process_mail_accounts`
- `src/documents/management/commands/document_consumer.py` — 24 logger `paperless.management.consumer`, 200 "Using inotify to watch directory for changes"
- `src/documents/sanity_checker.py` — 27 "Sanity checker detected no issues."
- Schedule migrations — `src/paperless_mail/migrations/0002_auto_20201117_1334.py:10-15` (mail, 10 min); `src/documents/migrations/1001_auto_20201109_1636.py:10-14` (train, hourly) & `:15-19` (index, daily); `src/documents/migrations/1004_sanity_check_schedule.py:10-14` (sanity, weekly)

**Dependency internals (installed `django-q==1.3.9`; NOT repository source):**

- `django_q/conf.py` — 73 `POLL=0.2`, 80 `PREFIX = conf.get("name","default")`, 90 `GUARD_CYCLE=0.5`, 174 `Q_STAT = "django_q:{PREFIX}:cluster"`, 207 `logger = getLogger("django-q")`, 213-214 log format `%(asctime)s [Q] %(levelname)s %(message)s` / `%H:%M:%S`
- `django_q/cluster.py` — 79 "starting.", 87 "stopping.", 90 "has stopped.", 223 "reincarnated pusher … after sudden death", 230/232/234 worker reincarnate/recycle, 253-290 guard loop (277 monitor-liveness, 280-281 pusher-liveness check, 283 `counter += cycle`, 284-286 scheduler `counter >= 30`, 288 `Stat.save` heartbeat, 289 `sleep(cycle)`), 256 "guarding cluster", 261 "running.", 295 "stopping cluster processes", 320 "waiting for the monitor.", 342 "pushing tasks at", 345 `broker.dequeue()`, 347 `logger.error(e, traceback.format_exc())`, 366 "stopped pushing tasks", 378 "monitoring at", 392 "Processed", 396 "stopped monitoring results", 410 "ready for work at", 420 "processing", 451 "stopped doing work", 576 `scheduler()`, 589 `filter(next_run__lt=now())`, 615-616 MINUTES shift, 639 `CATCH_UP or next_run > now`, 669 "created a task from schedule"
- `django_q/status.py` — 69 `Stat.get_key()` builds `f"{Conf.Q_STAT}:{cluster_id}"`, 71-73 `Stat.save` (3 s TTL), 74-75 `except …: logger.error(e)` (the guard-side connection-error burst)
- `django_q/brokers/redis_broker.py` — 20-21 `dequeue` → `blpop(list_key, 1)`, 47-48 `set_stat` → `redis SET ex=timeout`
- `django_q/tasks.py` — 74 "Enqueued"

---

## Coverage summary (Q1–Q5)

- **Q1 — up & ready:** canonical path (entrypoint → `gosu paperless docker-prepare.sh` `do_work` body 66-79 → supervisord → 3 programs) **actually executed** via the repository's own `docker/wait-for-redis.py` gate and `docker/supervisord.conf` (supervisord spawns/reaps all three as `paperless`); the clean Django-Q readiness banner (`Q Cluster <id> running.`) shown from an isolated single-stream capture, with the supervisord stdout/stderr interleave caveat labeled; gunicorn readiness proven by live HTTP 200 (its `when_ready` line captured separately, labeled). Environment-specific values (random cluster id, CPU-derived 11 workers) labeled.
- **Q2 — idle background activity:** the three supervised programs, and the four scheduled tasks by name with frequency + meaning + citation, verified against the live `Schedule` rows and shown firing on the first pass; canonical body execution noted, with the minimal-env body-failure mode labeled non-canonical.
- **Q3 — periodic health logs:** three signals separated — one-time readiness banner; **silent 0.5 s heartbeat DIRECTLY measured** via `redis-cli monitor` (eight consecutive SET ops, seven 0.501 s deltas, each `EX 3`); **~30 s scheduler tick measured and stable across 3 starts** (29 s / 30 s / 30 s); most-frequent visible idle line = the 10-minute mail check, with **three firings measured in each of two independent runs** (both clean gaps `10 m 01 s`; first gap short and explained) and 0–30 s jitter attributed to the scheduler tick; worker-recycle lines labeled normal.
- **Q4 — operational again:** live Redis interrupted (`redis-cli shutdown nosave` at `19:22:04.633`) and restarted (`19:22:39.815`); **BEFORE** baseline active assertions (HTTP 200, Channels, broker `math.sqrt(16)`→`4.0`); **DURING** the `≈1.99 lines/s` errno-111 burst (**70 lines / 35.18 s**, exact count + derived rate, source = guard `Stat.save`) and the **complete 89-line** `--- Logging error ---` block (with `Call stack:`/`Message:`/`Arguments:`); **AFTER** recovery confirmed by burst cessation (**bounded 0 errors** after the last), the sticking `reincarnated pusher … after sudden death` → `pushing tasks at <pid>` sequence, and **decisive active round-trips** (broker `math.sqrt(81)`→`9.0`, Channels, HTTP 200, 3-program liveness, heartbeat resumed). Explicitly: **no literal "reconnected" banner**, and `pushing tasks at` alone is **not** reconnection proof (it recurs during the outage).
- **Q5 — continuously running:** observed `/proc` tree (with producing command) — supervisord + gunicorn (master + 2 workers) + document_consumer (silent inotify) + qcluster (guard + monitor + pusher + 11 workers) — plus Redis (broker + Channels); SQLite default noted; **complete** graceful-stop sequence shown (no ellipsis).

---

## Provenance, cleanup, and repository state

- **Derived from runtime observation** on the image `paperless-ngx-ready:542221a38dff` (`sha256:aae32959b227…`), across three independent cluster starts (canonical `supervisord` Runs A and C; isolated `manage.py qcluster` Run B) plus a live Redis interrupt/restart on Run C — combined with **read-only citation** of repository files under `docker/`, `src/`, and the root-level config, and of the installed dependency `django-q==1.3.9`. Every embedded log block is a byte-faithful excerpt of a captured file whose SHA-256 is listed in §0.6; observed-vs-inferred and canonical-vs-non-canonical distinctions are called out inline.
- **No source file was modified.** All temporary artifacts (throwaway containers `pngx-obs` and `pngx-q4`, the co-located Redis instances, and all data/log files under `/tmp` and `/app/data` inside those containers) were removed on completion; both containers were stopped and `docker rm`'d and their removal verified.
- **Repository state:** working tree clean (`git status --porcelain` empty); the source tree remains at commit `542221a38dff06361e07976452f9aea24d210542`; the only change introduced by this investigation is the addition of this document at `blitzy/documentation/paperless-ngx_542221a38dff.md`.
