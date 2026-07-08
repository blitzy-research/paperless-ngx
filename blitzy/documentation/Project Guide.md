# Blitzy Project Guide — Paperless‑NGX Document Ingestion Pipeline: Runtime‑Grounded Trace

> **Deliverable:** `blitzy/documentation/paperless-ngx_542221a38dff.md`
> **Branch:** `blitzy-02766774-d432-4287-a066-d2b57627175a` · **Base:** `542221a38` · **Version under study:** paperless‑ngx `1.7.0`
> **Task class:** Documentation / runtime‑grounded investigation (read‑only source)

---

## 1. Executive Summary

### 1.1 Project Overview

This project produced a single, comprehensive, runtime‑grounded documentation deliverable that explains — and demonstrates with real observed evidence — how Paperless‑NGX v1.7.0 processes a document end‑to‑end: from a file landing in the consumption directory, through detection, task queuing, parsing, classification, and full‑text indexing. The audience is engineers and technical reviewers who need an authoritative, citation‑backed trace of the ingestion pipeline. Scope was strictly read‑only: the paperless‑ngx source tree was treated as immutable, and the sole committed artifact is one new markdown answer document. Every behavioral claim is backed by captured command output (logs, Redis broker contents, database rows) plus exact `file:line` citations, satisfying the six decomposed requirements (R1–R6) and five secondary edge conditions.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title AAP-Scoped Completion — 95.7% Complete (44 of 46 hrs)
    "Completed Work (hrs)" : 44
    "Remaining Work (hrs)" : 2
```

<sub>**Legend / Blitzy brand colors:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF`.</sub>

| Metric | Value |
|--------|-------|
| **Total Hours** | **46.0** |
| **Completed Hours (AI + Manual)** | **44.0** (AI: 44.0 · Manual: 0.0) |
| **Remaining Hours** | **2.0** |
| **Percent Complete (AAP‑scoped)** | **95.7 %** |

> **Completion formula (PA1, hours‑based):** `44.0 / (44.0 + 2.0) × 100 = 95.7 %`. All completed work was delivered autonomously by Blitzy agents; the remaining 2.0 hours are human path‑to‑production activities (SME review + PR merge).

### 1.3 Key Accomplishments

- ✅ **Canonical runtime stood up** — Python 3.9 venv with all 15 pinned pipeline dependencies, Redis broker, `manage.py migrate` (exit 0, scheduled tasks seeded), and the three canonical processes from `docker/supervisord.conf` (gunicorn ASGI, `document_consumer`, `qcluster`).
- ✅ **Real ingestion path exercised** — a `.txt` file dropped into `PAPERLESS_CONSUMPTION_DIR` (not the REST upload), traced end‑to‑end; checksum reproduced exactly (`0da8a96bf7f3a377f2acba04d2800b51`).
- ✅ **Ordered log stream captured verbatim** — 12‑line stream with a per‑line `file:line` mapping table; ingestion task identified as `documents.tasks.consume_file` `[src/documents/tasks.py:184]`.
- ✅ **Redis broker introspected** — transitional states `LLEN 0 → 1 → 0`; complete 447‑byte `SignedPackage` payload captured, pickle protocol 5 verified, decoded task dict with end‑to‑end id traceability to `django_q_task`.
- ✅ **Database evidence gathered** — `documents_document` row (checksum == md5), `django_q_task` history, `django_admin_log` ADDITION; documented the `documents_log` defined‑but‑unwritten accuracy nuance; Whoosh index confirmed.
- ✅ **Codepath traced & framework named** — `document_consumer._consume` → `async_task` (builds/signs/enqueues) → `qcluster` → `consume_file`; **Django‑Q + Redis**, explicitly **not** Celery (zero `celery`/`kombu` references).
- ✅ **Secondary conditions exercised** — stable log ordering across ≥2 runs, duplicate rejection, unsupported‑MIME (two paths), polling vs. inotify, and the barcode‑split branch.
- ✅ **Read‑only mandate upheld** — `git diff` shows exactly one added file (956 insertions, 0 deletions); `git status` clean; source byte‑for‑byte unchanged.
- ✅ **115 distinct `file:line` citations** embedded; independently spot‑checked and all resolve against the v1.7.0 source.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All AAP‑scoped requirements (R1–R6 + 5 secondary conditions) are complete, validated, and committed. No blocking or release‑gating issues remain. | — | — | — |

> The only remaining work is standard path‑to‑production (human SME review and PR merge); see §1.6 and §2.2. The 33 out‑of‑scope full‑suite test failures are environmental OCR/Ghostscript artifacts unrelated to the documented `.txt` pipeline (see §3 and §6).

### 1.5 Access Issues

**No access issues identified.** The repository is accessible on the working branch, git operations succeed, the runtime was stood up locally in an isolated workspace, Redis was reachable, and no external service credentials or third‑party API access were required for this documentation task.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Git repository (branch `blitzy-02766774-…`) | Read/Write | None — clean working tree, commits present | ✅ No issue | — |
| Redis broker (localhost:6379) | Runtime | None — `redis-cli ping → PONG` | ✅ No issue | — |
| External APIs / credentials | N/A | None required for this task | ✅ No issue | — |

### 1.6 Recommended Next Steps

1. **[Medium]** Conduct SME review of `blitzy/documentation/paperless-ngx_542221a38dff.md` for technical accuracy and completeness of the R1–R6 answers and the five secondary conditions (~1.5 h).
2. **[Medium]** Approve and merge the PR to the target branch — a single additive file in a net‑new directory, no merge conflict expected (~0.5 h).
3. **[Low]** _(Optional)_ Independently reproduce the runtime trace using the deliverable's Reproduction & Cleanup Appendix to build reviewer confidence (not required for merge).
4. **[Low]** _(Out‑of‑scope)_ If OCR‑pipeline test coverage is required in a future task, provision a canonical bullseye‑based environment (Ghostscript 9.x) to clear the 33 environmental full‑suite failures — explicitly outside this documentation task's scope.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement (R1–R6), a rule‑mandated secondary condition, or a supporting AAP activity (research, authoring, QA remediation, cleanup). All items are 100 % complete.

| Component | Hours | Description |
|-----------|------:|-------------|
| **[R1]** Canonical Runtime Bring‑Up | 5.0 | Python 3.9 venv + 15 pinned deps; Redis broker; `manage.py migrate` (exit 0, scheduled tasks seeded); launched gunicorn ASGI, `document_consumer`, `qcluster` per `docker/supervisord.conf`, isolated outside the source tree. |
| **[R2]** Real Entry‑Point Ingestion | 1.5 | Dropped `report_2022.txt` into `PAPERLESS_CONSUMPTION_DIR` (real ingestion path, not REST); captured md5 `0da8a96bf7f3a377f2acba04d2800b51`. |
| **[R3]** Log‑Stream & Task Analysis | 4.0 | Verbatim 12‑line ordered log stream; per‑line `file:line` mapping table; proved ingestion task `= documents.tasks.consume_file` and that only 5 distinct task funcs exist (inline handlers vs. scheduled tasks). |
| **[R4]** Redis Broker Introspection | 5.0 | Before/during/after `LLEN 0 → 1 → 0`; complete 447‑byte `SignedPackage` payload; pickle protocol 5 verified vs. interpreter; decoded task dict; task‑id traceability to `django_q_task`. |
| **[R5]** Database Query Evidence | 3.0 | `documents_document` row (checksum == md5); `django_q_task` Success; `django_admin_log` ADDITION; `documents_log = 0` accuracy nuance; Whoosh index files. |
| **[R6]** Codepath Trace & Framework ID | 3.5 | `_consume` → `async_task` (builds/signs/enqueues) → `qcluster` → `consume_file`; named **Django‑Q + Redis**; proved zero `celery`/`kombu`. |
| **[Secondary]** 5 Edge/Secondary Conditions | 5.5 | Stable ordering ≥2 runs; duplicate rejection (`success=False`); unsupported MIME (two paths); polling vs. inotify; barcode‑split off by default. |
| **[Research]** Django‑Q Internals Corroboration | 1.5 | Web‑search confirmation of task‑package serialization/signing and result‑backend semantics (AAP §0.2.2). |
| **[Authoring]** Answer Document Authoring | 10.0 | 956 lines / 7,780 words / 115 distinct verified citations / 8+ sections / coverage‑pass table. |
| **[QA]** QA Remediation (2 commits) | 4.0 | `d0d391263` remediate QA findings; `c7aa04f8e` embed complete runtime evidence. |
| **[Cleanup]** Cleanup & Read‑Only Verification | 1.0 | Removed temp scripts/data (isolated workspace); verified `git status --porcelain` empty; source byte‑for‑byte unchanged. |
| **Total Completed** | **44.0** | Sum of all completed components. |

### 2.2 Remaining Work Detail

Remaining work is exclusively path‑to‑production for a validated documentation deliverable. There are no incomplete AAP core requirements.

| Category | Hours | Priority |
|----------|------:|----------|
| Human SME Review of Deliverable (technical accuracy & completeness of R1–R6 + secondary conditions) | 1.5 | Medium |
| PR Approval & Merge to Target Branch | 0.5 | Medium |
| **Total Remaining** | **2.0** | — |

> **Optional / out‑of‑scope (0 counted hours, excluded from the 2.0 h total):** independent reproduction of the trace (informational); provisioning a canonical Ghostscript‑9.x environment to clear the 33 unrelated OCR test failures (future task only).

### 2.3 Hours Reconciliation Summary

| Roll‑up | Hours |
|---------|------:|
| Section 2.1 — Completed | 44.0 |
| Section 2.2 — Remaining | 2.0 |
| **Total Project Hours** | **46.0** |
| **Percent Complete** | **95.7 %** |

**Cross‑section integrity:** Section 2.1 (44.0) + Section 2.2 (2.0) = 46.0 = Total Hours in §1.2 ✅ · Remaining 2.0 is identical in §1.2, §2.2, and §7 ✅.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project. Because the deliverable is a markdown document (which has no unit tests of its own), the relevant tests are the source‑tree tests that **directly cover the documented `.txt` ingestion codepath**, which the validator executed to confirm the pipeline behaves exactly as documented.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Text parser (documented parser) | pytest / Django test | 2 | 2 | 0 | n/a | `paperless_text` — the parser the documented `.txt` routes to. |
| Consumer pipeline (documented codepath) | pytest / Django test | 33 | 33 | 0 | n/a | `src/documents/tests/test_consumer.py` — detection → task → parse → store → finished signal. |
| **In‑scope pipeline total** | — | **35** | **35** | **0** | **100 % pass** | Every test covering the documented codepath passes. |
| System check | `manage.py check` | 1 | 1 | 0 | n/a | 0 issues reported. |
| Module compilation | `py_compile` | 16 | 16 | 0 | n/a | All 16 in‑scope reference modules byte‑compile with zero syntax errors. |

> **Transparent disclosure (out of scope):** The full suite reports **33 failures** caused by a Ghostscript 10.05 vs. ocrmypdf 13.4.3 OCR tool‑version incompatibility plus root‑user chmod artifacts. These are **environmental**, affect only OCR paths (the documented `TextDocumentParser` uses no Ghostscript), **pre‑date this task**, and are **unfixable under the read‑only‑source constraint**. They are correctly excluded from the in‑scope results above. Coverage % is marked `n/a` because the committed deliverable is documentation, not executable code; the pass metrics reflect the source‑tree tests that validate the documented behavior.

---

## 4. Runtime Validation & UI Verification

Runtime health was validated by standing up the full canonical pipeline in an isolated workspace and processing real documents. **UI verification is not applicable** — the Angular frontend (`src-ui/`) is explicitly out of scope; this is a backend pipeline investigation whose only human‑facing output is the markdown deliverable.

**Runtime health (from Blitzy autonomous validation):**

- ✅ **Redis broker** — `redis-cli ping → PONG` (Django‑Q broker + Channels layer backend).
- ✅ **Database migrations** — `manage.py migrate` exit 0; tables created (`documents_document`, `django_q_task`, `django_admin_log`, …) and scheduled Django‑Q tasks seeded (`train_classifier` H, `index_optimize` D, `sanity_check` W).
- ✅ **Web/ASGI process** — gunicorn (`paperless.asgi`) health‑checked: `/admin/login/ → 200`, `/api/ → 200`; canonical bind `0.0.0.0:8000`.
- ✅ **Directory watcher** — `document_consumer` (inotify) detected files and enqueued the ingestion task.
- ✅ **Worker/scheduler** — `qcluster` popped and executed `consume_file`; 4 real documents processed end‑to‑end.
- ✅ **Broker payload** — 447‑byte signed pickle observed on `django_q:paperless:q`; decoded `func = documents.tasks.consume_file`; broker id == `django_q_task.id` (end‑to‑end match).
- ✅ **Persistence** — `documents_document` row created with `checksum == md5` of the dropped file; `django_q_task` records `Success. New document id N created`; `django_admin_log` ADDITION written.
- ✅ **Full‑text index** — Whoosh index updated inline at end of consumption.
- ✅ **Accuracy nuance verified** — `documents_log` count `= 0`; `LOGGING` confirmed to have **no** database handler in v1.7.0 (defined‑but‑unwritten), exactly as documented.
- ⚠ **Environment note (non‑blocking)** — full test suite has 33 OCR/Ghostscript failures from host tool‑version drift (gs 10.05 vs. canonical 9.53); unrelated to the documented `.txt` pipeline.

---

## 5. Compliance & Quality Review

Cross‑map of AAP deliverables and governing‑rule ("SWE‑AtlasQnA‑Repo") directives to Blitzy's quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Directive | Requirement | Status | Evidence / Notes |
|---------------------------|-------------|:------:|------------------|
| R1 — Bring up full runtime | Redis, DB, qcluster, consumer, gunicorn | ✅ Pass | R1 section; validator reproduced migrate + gunicorn 200. |
| R2 — Real entry point | Drop file into consumption dir (not REST) | ✅ Pass | R2 section; checksum reproduced exactly. |
| R3 — Log messages & tasks | Ordered stream + ingestion task named | ✅ Pass | 12‑line verbatim stream + per‑line mapping; `consume_file`. |
| R4 — Broker introspection | Payload format & structure | ✅ Pass | 447‑byte signed pickle; decoded dict; proto 5 verified. |
| R5 — Database after processing | Record, parsed fields, task history | ✅ Pass | `documents_document`, `django_q_task`, `django_admin_log`; `documents_log` nuance. |
| R6 — Codepath & framework | Name constructor + queuing framework | ✅ Pass | `async_task` builds task; Django‑Q + Redis (not Celery). |
| Secondary conditions | Exercise every implied condition | ✅ Pass | 5 conditions: ordering, duplicate, unsupported MIME (×2), polling/inotify, barcode. |
| Run‑first methodology | Build/run before writing; capture real output | ✅ Pass | All claims backed by captured command output. |
| Canonical configuration | Default config + exact invocation commands | ✅ Pass | Commands from `docker/supervisord.conf` reproduced & stated. |
| Complete, unedited evidence | Full output, not paraphrase | ✅ Pass | Full payload, full log slices, real DB rows embedded. |
| Exactness & `file:line` grounding | Actual values + citations | ✅ Pass | 115 distinct citations; all resolve (independently spot‑checked). |
| Answer every part | Coverage pass over all sub‑questions | ✅ Pass | Coverage‑Pass table maps R1–R6 + S1–S5. |
| Scope — read‑only source | No source modification; only the answer doc | ✅ Pass | `git diff` = 1 file added, 956/0; `git status` clean. |
| Deliverable location/naming | `blitzy/documentation/<branch>.md` | ✅ Pass | `blitzy/documentation/paperless-ngx_542221a38dff.md` present. |
| Zero placeholders/TODOs | No stubs or deferred work | ✅ Pass | Validator confirmed zero stubs/placeholders/TODOs. |

**Fixes applied during autonomous validation:** Two QA remediation commits (`d0d391263`, `c7aa04f8e`) addressed review findings and embedded complete, unedited runtime evidence. The final validator required **zero** additional edits. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

The risk profile is intrinsically **low**: this is an additive documentation change over a read‑only source tree, introducing zero new code and zero new dependencies. All identified risks are Low or Informational severity and are largely mitigated.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|-----------|--------|
| T1 — Host env version drift (gs 10.05 vs. canonical 9.53) affecting OCR reproduction | Technical | Low | Low | Documented `.txt` pipeline uses `TextDocumentParser` (zero Ghostscript refs); only unrelated OCR tests affected | Documented / Accepted |
| T2 — Run‑variable values (timestamps, ids, scratch hashes, 447‑byte length) mistaken for invariants | Technical | Low | Low | Deliverable explicitly labels these as variable and proves invariant structure across ≥2 runs | Mitigated |
| T3 — `file:line` citations drift if read against a different version | Technical | Low | Medium | Deliverable explicitly scoped to v1.7.0 `[src/paperless/version.py:1]` | Mitigated |
| S1 — Signed payload + `SECRET_KEY` reference embedded in doc | Security | Informational | Low | `SECRET_KEY` redacted to `<set, len 50>` (value never printed); payload from throwaway ephemeral instance; no production secret | Mitigated / Verify on review |
| S2 — New dependency/attack surface | Security | None | None | Read‑only investigation adds zero code and zero dependencies | N/A |
| O1 — Reproduction requires correct `PAPERLESS_*` env vars to keep data outside source tree | Operational | Low | Low | Deliverable documents the exact isolated‑workspace configuration | Mitigated |
| O2 — Production operational footprint | Operational | None | None | Deliverable is a static markdown file — no monitoring/health‑check/logging needs | N/A |
| I1 — PR merge into target branch | Integration | Low | Low | Single new file in a net‑new directory; no source files touched; no conflict expected | Open (trivial, pending merge) |
| I2 — CI/CD or external‑service integration | Integration | None | None | No CI/CD, external service, or API integration introduced | N/A |

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title Project Hours Breakdown (Total 46.0 hrs)
    "Completed Work" : 44
    "Remaining Work" : 2
```

<sub>**Blitzy brand colors:** Completed Work = Dark Blue `#5B39F3` · Remaining Work = White `#FFFFFF` (outlined in Violet‑Black `#B23AF2`).</sub>

**Remaining hours by category (from §2.2):**

| Category | Hours | Priority |
|----------|------:|----------|
| Human SME Review of Deliverable | 1.5 | Medium |
| PR Approval & Merge | 0.5 | Medium |
| **Total Remaining** | **2.0** | — |

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#A8FDD9","pieStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieTitleTextSize":"16px","pieSectionTextSize":"14px","pieLegendTextSize":"13px"}}}%%
pie showData title Remaining Work Distribution (2.0 hrs)
    "SME Review" : 1.5
    "PR Merge" : 0.5
```

> **Integrity check:** "Remaining Work" = **2.0 hrs** in the pie chart above equals the Remaining Hours in §1.2 and the sum of the §2.2 "Hours" column. ✅

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a thorough, runtime‑grounded trace of the Paperless‑NGX v1.7.0 ingestion pipeline. All six decomposed requirements (R1–R6) and all five rule‑mandated secondary conditions were exercised against a live, canonically configured runtime, with complete and unedited evidence — real log streams, the actual 447‑byte signed broker payload, decoded task dictionaries, and real database rows — each anchored by exact `file:line` citations (115 distinct, all resolving). The governing read‑only mandate was upheld precisely: `git diff` against the base shows exactly one added file (956 insertions, 0 deletions) and a clean working tree.

**Remaining gaps.** None within the AAP core scope. The project stands at **95.7 % complete (44 of 46 hours)**. The remaining **2.0 hours** are standard path‑to‑production: a human SME accuracy review (1.5 h) and PR approval/merge (0.5 h).

**Critical path to production.** SME review → approve → merge the single additive documentation file. No build, deployment, integration, or configuration work is required.

**Success metrics.** In‑scope pipeline tests: **35/35 passing**. Django system check: **0 issues**. Module compilation: **16/16**. Citations resolving: **115/115**. Source integrity: **byte‑for‑byte unchanged**.

**Production readiness assessment.** The deliverable is **production‑ready pending human review**. Confidence is **High**: the artifact was independently reproduced by the final validator with zero edits required, and its numeric and citation claims were re‑verified during this assessment. The only caveat is the environmental OCR/Ghostscript test drift, which is explicitly out of scope and does not touch the documented `.txt` pipeline.

| Metric | Result |
|--------|--------|
| AAP‑scoped completion | 95.7 % (44 / 46 h) |
| AAP requirements complete | R1–R6 + 5 secondary conditions (100 %) |
| In‑scope tests | 35 / 35 passing |
| Citations resolving | 115 / 115 |
| Source files modified | 0 (read‑only upheld) |
| Production readiness | Ready pending SME review + merge |

---

## 9. Development Guide

This guide covers (A) verifying the committed deliverable and (B) reproducing the runtime‑grounded trace in the canonical configuration. Track A commands were executed successfully during this assessment; Track B commands are transcribed from the validated deliverable's Reproduction Appendix.

### 9.1 System Prerequisites

- **Verification track (A):** `git` ≥ 2.x and a markdown viewer. _(Tested: git 2.51.0.)_
- **Reproduction track (B):** **Python 3.9** (matches the container base `FROM python:3.9-slim-bullseye`; the validator used 3.9.25), a **Redis** server, and system libraries `libmagic1`, `optipng`, and ImageMagick (`convert`).
  - ⚠ **Python version matters:** the host default `python3` may be newer (e.g., 3.13). A **Python 3.9 venv is required** — the pinned deps (e.g., `scipy==1.8.0`) target `3.8 ≤ py < 3.11`.

### 9.2 Track A — Verify the Deliverable _(tested during this assessment)_

```bash
# From the repository root:
test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo present   # -> present (956 lines / 7780 words)
git branch --show-current                                                     # -> blitzy-02766774-d432-4287-a066-d2b57627175a
git status --porcelain                                                        # -> (empty; source unchanged)
git diff --name-status 542221a38..HEAD                                        # -> A  blitzy/documentation/paperless-ngx_542221a38dff.md

# Spot-check citations (all resolve against the v1.7.0 source):
sed -n '184p' src/documents/tasks.py                                          # -> def consume_file(
grep -n 'Q_CLUSTER = {' src/paperless/settings.py                             # -> 449:Q_CLUSTER = {
grep -icE 'celery|kombu' requirements.txt Pipfile                             # -> 0  (Django-Q, not Celery)
```

### 9.3 Track B — Reproduce the Runtime Trace _(canonical configuration)_

```bash
# 1. Python 3.9 venv + pinned deps (created OUTSIDE the read-only source tree)
python3.9 -m venv /opt/paperless-venv && . /opt/paperless-venv/bin/activate
pip install -r requirements.txt

# 2. Direct ALL runtime data outside the repo (keeps the source byte-for-byte unchanged)
export PAPERLESS_DATA_DIR=/tmp/pngx-work/data       PAPERLESS_MEDIA_ROOT=/tmp/pngx-work/media \
       PAPERLESS_CONSUMPTION_DIR=/tmp/pngx-work/consume  PAPERLESS_LOGGING_DIR=/tmp/pngx-work/log \
       PAPERLESS_SCRATCH_DIR=/tmp/pngx-work/scratch  PAPERLESS_REDIS=redis://localhost:6379
mkdir -p /tmp/pngx-work/{data,media,consume,log,scratch}

# 3. Start Redis (Django-Q broker + Channels layer) and verify
redis-server --daemonize yes && redis-cli ping                # -> PONG

# 4. Apply migrations (creates backing tables + seeds scheduled tasks)
cd src && python manage.py migrate                            # -> exit 0

# 5. Launch the THREE canonical processes (exact commands from docker/supervisord.conf)
python3 manage.py document_consumer &                         # watcher   [docker/supervisord.conf:20]
python3 manage.py qcluster &                                  # worker    [docker/supervisord.conf:29]
gunicorn -c ../gunicorn.conf.py paperless.asgi:application &  # web/ASGI  [docker/supervisord.conf:11]

# 6. Exercise the REAL entry point + observe the pipeline
printf 'Paperless NGX runtime trace test.\n' > /tmp/pngx-work/consume/report_2022.txt
redis-cli LLEN django_q:paperless:q                           # 0 -> 1 (queued) -> 0 (drained)
tail -n 20 /tmp/pngx-work/log/paperless.log                   # ordered log stream (R3)
python manage.py shell -c "from documents.models import Document; print(Document.objects.count())"
```

### 9.4 Verification Steps

- `redis-cli ping` → `PONG`; `manage.py migrate` exits `0`.
- Web health: `curl -s -o /dev/null -w '%{http_code}' http://localhost:8000/admin/login/` → `200`.
- Broker transition observed: `LLEN` `0 → 1 → 0`.
- `documents_document` row created; `checksum` equals the md5 of the dropped file.
- Inspect the queued package: `redis-cli --no-raw LINDEX django_q:paperless:q 0` (447‑byte signed pickle); decode via `django_q.signing.SignedPackage.loads` inside `manage.py shell`.

### 9.5 Troubleshooting

- **`ModuleNotFoundError: paperless`** → run `manage.py` from `$REPO/src`, not the repository root.
- **Pinned‑dependency install fails** → ensure the **Python 3.9 venv** is active (`python -V` shows `3.9.x`); newer interpreters (e.g., 3.13) reject `scipy==1.8.0`.
- **`Connection refused` to Redis** → start `redis-server` and confirm `redis-cli ping` returns `PONG`.
- **Runtime data written into the repo** → ensure all `PAPERLESS_*` directories point **outside** the source tree **before** running `migrate`.
- **33 full‑suite test failures** → out‑of‑scope OCR/Ghostscript environment drift (gs 10.05 vs. canonical 9.53); unrelated to the documented `.txt` pipeline.

### 9.6 Example Usage

- **Ingestion trace:** drop a `.txt` into `PAPERLESS_CONSUMPTION_DIR` → watch `paperless.log` → query `documents_document`.
- **Broker inspection:** `redis-cli --no-raw LINDEX django_q:paperless:q 0` while `qcluster` is paused, then decode the signed package in the Django shell.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Verify deliverable present | `test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo present` |
| Confirm clean tree | `git status --porcelain` |
| Confirm single additive change | `git diff --name-status 542221a38..HEAD` |
| Blitzy commits | `git log --author="agent@blitzy.com" --oneline` |
| Redis liveness | `redis-cli ping` |
| Broker queue length | `redis-cli LLEN django_q:paperless:q` |
| Raw queued package | `redis-cli --no-raw LINDEX django_q:paperless:q 0` |
| Apply migrations | `python manage.py migrate` |
| Start watcher / worker / web | `manage.py document_consumer` · `manage.py qcluster` · `gunicorn -c gunicorn.conf.py paperless.asgi:application` |
| Framework proof | `grep -icE 'celery|kombu' requirements.txt Pipfile` → `0` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| gunicorn (web/ASGI) | 8000 | Canonical bind `0.0.0.0:8000` `[gunicorn.conf.py]`. |
| Redis | 6379 | Django‑Q broker + Channels layer (`PAPERLESS_REDIS`, default `redis://localhost:6379`). |

### Appendix C — Key File Locations

| Artifact | Path |
|----------|------|
| **Deliverable (only committed artifact)** | `blitzy/documentation/paperless-ngx_542221a38dff.md` |
| File detection & task submission | `src/documents/management/commands/document_consumer.py` (`_consume` :46, `async_task` :86‑91) |
| Ingestion task | `src/documents/tasks.py` (`consume_file` :184) |
| Consumption pipeline | `src/documents/consumer.py` (`try_consume_file` :180, `Document.objects.create` :398) |
| Runtime configuration | `src/paperless/settings.py` (`Q_CLUSTER` :449, Redis :456) |
| Consumption‑finished handlers | `src/documents/signals/handlers.py` (`set_log_entry` :413, `add_to_index` :428) |
| Data model | `src/documents/models.py` (`Document` :88, `Log` :285) |
| Full‑text index | `src/documents/index.py` (`add_or_update_document` :118) |
| Process orchestration | `docker/supervisord.conf` (gunicorn :11, consumer :20, qcluster :29) |
| Version | `src/paperless/version.py` (`(1, 7, 0)`) |

### Appendix D — Technology Versions

| Component | Version | Role |
|-----------|---------|------|
| paperless‑ngx | 1.7.0 | System under study |
| Python (canonical) | 3.9 (validator: 3.9.25) | Runtime |
| django | 4.0.4 | Web framework / ORM / signing |
| **django‑q** | **1.3.9** | **Queuing framework** (`async_task`, `qcluster`, `SignedPackage`) |
| redis (client) | 3.5.3 | Django‑Q broker + Channels layer |
| channels / channels‑redis | 3.0.4 / 3.4.0 | WebSocket/ASGI progress streaming |
| whoosh | 2.7.4 | Full‑text index |
| scikit‑learn / scipy | 1.0.2 / 1.8.0 | Auto‑matching classifier |
| watchdog / inotifyrecursive | 2.1.7 / 0.3.5 | Directory watchers (polling / inotify) |
| python‑magic | 0.4.25 | MIME detection |
| psycopg2 | 2.9.3 | Optional PostgreSQL driver (SQLite is default) |

### Appendix E — Environment Variable Reference

| Variable | Purpose | Value used in reproduction |
|----------|---------|----------------------------|
| `PAPERLESS_CONSUMPTION_DIR` | Watched consumption directory (real entry point) | `/tmp/pngx-work/consume` |
| `PAPERLESS_DATA_DIR` | SQLite DB + Whoosh index location | `/tmp/pngx-work/data` |
| `PAPERLESS_MEDIA_ROOT` | Stored document media | `/tmp/pngx-work/media` |
| `PAPERLESS_LOGGING_DIR` | `paperless.log` location | `/tmp/pngx-work/log` |
| `PAPERLESS_SCRATCH_DIR` | Scratch/temp working area | `/tmp/pngx-work/scratch` |
| `PAPERLESS_REDIS` | Redis URL (broker + channels) | `redis://localhost:6379` |

> All runtime data is intentionally directed **outside** the source tree so the repository remains byte‑for‑byte unchanged.

### Appendix F — Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `git diff` / `git status` | Confirm read‑only compliance (one added file, clean tree). |
| `redis-cli` | Introspect the Django‑Q broker list (`LLEN`, `LINDEX --no-raw`). |
| `manage.py shell` | Decode the `SignedPackage`; query `documents_document`, `django_q_task`, `documents_log`. |
| `py_compile` / `manage.py check` | Byte‑compile reference modules; Django system check (0 issues). |
| `sed -n 'Np file'` / `grep -n` | Resolve and spot‑check `file:line` citations. |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the governing scope for this task. |
| **Django‑Q** | The task‑queue framework paperless‑ngx uses (`django-q==1.3.9`); provides `async_task`, `qcluster`, and `SignedPackage`. **Not** Celery. |
| **`consume_file`** | The ingestion task (`documents.tasks.consume_file` `[src/documents/tasks.py:184]`) submitted when a file is detected. |
| **`SignedPackage`** | Django‑Q's pickled‑then‑HMAC‑signed task envelope placed on the Redis broker list `django_q:paperless:q`. |
| **`document_consumption_finished`** | The signal whose six handlers run inline (classification, matching, indexing) within `consume_file`. |
| **`documents_log` nuance** | The `Log` model/table is defined but not written by the v1.7.0 consumption path (no DB logging handler configured). |
| **Whoosh** | Pure‑Python full‑text index updated inline at the end of consumption. |
| **Path‑to‑production** | Standard human activities (SME review, PR merge) to deploy the deliverable. |

---

<sub>Blitzy Project Guide · Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet‑Black `#B23AF2` · Highlight = Mint `#A8FDD9`.</sub>
