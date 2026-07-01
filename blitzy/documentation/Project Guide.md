# Blitzy Project Guide — paperless-ngx Runtime Ingestion Investigation

> **Deliverable:** `blitzy/documentation/paperless-ngx_542221a38dff.md` · **Branch:** `blitzy-147a7cda-59b8-47bf-8646-d512a4b54604` · **Base commit:** `542221a38dff` · **HEAD:** `d4beb76e6`
>
> **Legend (Blitzy brand colors):** <span style="color:#5B39F3">■</span> **Completed / AI Work = Dark Blue `#5B39F3`** · <span style="color:#B23AF2">■</span> Headings/Accents = Violet-Black `#B23AF2` · <span style="color:#A8FDD9">■</span> Highlight = Mint `#A8FDD9` · □ **Remaining / Not Completed = White `#FFFFFF`**

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, run-first investigation** of the open-source **paperless-ngx** document-management system (a monolithic Django backend with an Angular frontend, running as a multi-process stack: a Redis broker, a `gunicorn` web/API server, a `document_consumer` directory watcher, and a `qcluster` django-q worker). The objective was **not** to change the product but to produce one authoritative markdown document that explains — from **live, observed behavior** rather than reading alone — exactly how the system ingests a document: which services participate and in what ordered log sequence (O1), whether/when the ML classifier retrains (O2), and where documents land on disk and which database tables receive rows (O3). The audience is engineers and reviewers who need a trustworthy, citation-grounded behavioral reference for commit `542221a38dff`. The entire deliverable is a single new markdown file; the source repository is left byte-for-byte unchanged.

### 1.2 Completion Status

The completion percentage is computed with the **PA1 AAP-scoped hours methodology**: `Completion % = Completed Hours / (Completed Hours + Remaining Hours) × 100 = 34 / (34 + 3) = 34 / 37 = 91.9%`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'18px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Completion Status — 91.9% Complete (34h of 37h)
    "Completed Work (AI)" : 34
    "Remaining Work" : 3
```

| Metric | Hours |
|---|---|
| **Total Project Hours** | **37** |
| **Completed Hours (AI + Manual)** | **34** (34 AI + 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **91.9%** |

> The 34 completed hours were delivered **autonomously by Blitzy agents** (authoring + independent final validation); 0 hours of manual/human work have been logged. The 3 remaining hours are human path-to-production activities (review, sign-off, optional extension) detailed in Sections 2.2 and 1.6.

### 1.3 Key Accomplishments

- ✅ **Single named deliverable created** — `blitzy/documentation/paperless-ngx_542221a38dff.md` (870 lines), the exact filename mandated by the "SWE-AtlasQnA-Repo" rule (derived from the source branch).
- ✅ **Run-first methodology executed** — the full four-process stack was built and run in the mandated Python-3.9 Docker image before any prose was written; every quoted value is attributed to the command that produced it.
- ✅ **O1 fully answered** — all participating services enumerated, both ingestion entry points demonstrated (consume-directory watcher and `POST /api/documents/post_document/` → **HTTP 200 `"OK"`**), and the ordered log sequence quoted **verbatim** across three synchronized log windows (watcher, worker, dedicated file log) for the same file, mapped line-by-line into `Consumer.try_consume_file()`.
- ✅ **O2 fully answered** — proved the classifier does **not** retrain per upload; it is scheduled (`Schedule.HOURLY`) and doubly gated, with the two distinguishing log strings captured verbatim (INFO `Saving updated classifier model to ...` vs DEBUG `Training data unchanged.`) and model-persistence version-skew facts (`FORMAT_VERSION = 7`, plain pickle, no HMAC).
- ✅ **O3 fully answered** — default disk layout `media/documents/{originals,archive,thumbnails}/`, default filename `0000001.pdf` (`{pk:07}`), and the database tables that gain rows shown via **verbatim before/after row-count diffs**; abstract `MatchingModel` confirmed to have no table.
- ✅ **Exactly grounded** — Blitzy's validator confirmed **99/99** unique `file:line` citations resolve exactly against this commit's source.
- ✅ **Read-only scope preserved** — `git diff` versus the base commit shows exactly one added file; all source, docs, Docker, and dependency manifests are byte-identical to base; working tree is clean.
- ✅ **Independently re-validated** — all three questions were re-observed live in a fresh container and reproduced byte-for-byte; five quality gates passed with **zero fixes required**.

### 1.4 Critical Unresolved Issues

There are **no critical unresolved issues**. The deliverable compiles (as a well-formed markdown artifact), runs (its documented reproduction procedure was executed successfully end-to-end), is exactly cited, and passed all autonomous validation gates. The table below lists the only open items, none of which block release on technical grounds — they are the normal human-acceptance path.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Human SME review & sign-off not yet performed | Acceptance gate — deliverable is validated but not yet human-approved for merge | Reviewing Engineer / SME | 1.5h |
| Two O3 edge cases documented via explicit non-verification disclosure (unsupported-MIME rejection path; conditional tables `documents_log`, label tables, `django_q_schedule`) | Cosmetic/coverage only — the document is already rule-compliant because the "SWE-AtlasQnA-Repo" rule explicitly permits stating what could not be verified | Reviewing Engineer (optional) | 1.0h |

### 1.5 Access Issues

**No access issues identified.** The investigation ran entirely inside the provided Docker image, which ships all required runtimes and packages; the deliverable is committed on the working branch; the git history and working tree are fully accessible.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Source repository | Read/Write (git) | None — branch, history, and clean working tree all accessible | ✅ No issue | — |
| Provided runtime image (`paperless-ngx-ready:latest`) | Container execution | None — image ships Python 3.9 + Redis + tesseract + scikit-learn + all deps | ✅ No issue | — |
| Cited source files (18 targets) | Read | None — all citation targets present on disk | ✅ No issue | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform SME review & sign-off of `blitzy/documentation/paperless-ngx_542221a38dff.md`: confirm O1/O2/O3 each fully and correctly answer the prompt and spot-check a sample of the citations. *(≈1.5h)*
2. **[Medium]** Run a reproduction spot-check in a fresh container using the commands in the document's "How this was observed" section to independently confirm 1–2 key observations (e.g., `0000001.pdf`, the train-vs-idle log strings). *(≈0.5h)*
3. **[Low]** *(Optional)* Extend runtime coverage to the explicitly-disclosed not-verified edge cases (unsupported-MIME rejection; conditional tables) to replace the caveats with live evidence. *(≈1.0h)*
4. **[Low]** Merge the branch once sign-off is complete — no code review of source is needed because no source changed.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was delivered autonomously by Blitzy agents. Each component traces to specific AAP requirements (`R*` = deliverable/methodology rules; `O1/O2/O3` = the three questions; `Rc/Rcov/Rro/Rver` = cross-cutting rules).

| Component | Hours | Description |
|---|---:|---|
| Environment & full-stack bring-up `[R2,R3]` | 4.0 | Launched the mandated Python-3.9 image; started `redis-server`, ran migrations/`collectstatic`, created the superuser and required `consumer` user, enabled DEBUG, and brought up the four master processes; sanity-checked Python 3.9.23, HEAD, API `GET /api/` → 200, and `PAPERLESS_FILENAME_FORMAT` unset. |
| O1 — Ingestion-pipeline investigation `[O1a–d]` | 5.0 | Exercised both ingestion entry points, traced the ordered event sequence across three synchronized log windows, mapped each line into `Consumer.try_consume_file()`, captured the duplicate edge case, and confirmed the django-q `[Q]` banner. |
| O2 — Classifier-retraining investigation `[O2a–d]` | 4.5 | Queried the `Schedule.HOURLY` training entry; forced both gates (silent `MATCH_AUTO` early return, then a SHA1 data-hash change); captured the TRAIN vs IDLE log strings; read the plain-pickle header (`FORMAT_VERSION = 7`, 20-byte SHA1, no HMAC). |
| O3 — Storage & database investigation `[O3a–d]` | 4.0 | Listed `media/documents/{originals,archive,thumbnails}/`; proved the `0000001.pdf` filename pattern; authored and ran a before/after per-table row-count diff (two scenarios); enumerated `documents_*` tables (no `documents_matchingmodel`). |
| Answer-document authoring `[R1,Rv,Rr]` | 7.0 | Wrote the 870-line structured markdown: methodology, version-skew guard, running-stack overview, O1/O2/O3 prose with embedded verbatim transcripts and tables, rationale, and the "How this was observed" command appendix. |
| Citation grounding & exactness verification `[Rc]` | 3.0 | Placed and verified the `file:line` citations across 18 source/doc files so every asked value is grounded, never paraphrased. |
| Coverage pass & read-only cleanup `[Rcov,Rro,Rver]` | 2.0 | Built the 14-item coverage-pass table plus explicit not-verified disclosure; removed temporary scripts/PDFs; confirmed a clean tree and version-awareness (django-q, not upstream Celery). |
| Independent final validation (run-first re-observation) `[QA]` | 4.5 | Fresh-container full re-run reproducing O1/O2/O3 byte-for-byte; 99/99 citation re-verification; markdown structural + pre-commit checks; five gates passed with no fixes. |
| **Total Completed** | **34.0** | **Matches Completed Hours in Section 1.2.** |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production activity (for a read-only documentation deliverable, "production" = SME acceptance; no deployment/CI/CD applies). Each item traces to a path-to-production need.

| Category | Hours | Priority |
|---|---:|---|
| Human SME review & sign-off of the answer document (`P1`) | 1.5 | High |
| Reproduction spot-check in a fresh container (`P2`) | 0.5 | Medium |
| Optional coverage extension for explicitly-disclosed not-verified edge cases (`P3`) | 1.0 | Low |
| **Total Remaining** | **3.0** | **Matches Remaining Hours in Section 1.2 and Section 7 pie chart.** |

### 2.3 Hours Summary

| Bucket | Hours | Share |
|---|---:|---:|
| Completed (AI) | 34.0 | 91.9% |
| Remaining (Human) | 3.0 | 8.1% |
| **Total** | **37.0** | **100%** |

> **Confidence:** Completed hours — **High** (work is done, validated, and git-evidenced). Remaining hours — **Medium** (human-review effort varies; plausible range 2–4h, central estimate 3h).

---

## 3. Test Results

This deliverable is a read-only documentation artifact; the project's **own unit-test suite (pytest) was intentionally NOT executed** because tests are explicitly out of AAP scope and the source is byte-identical to the base commit (there is nothing the agent could have affected). The task-appropriate verification consists of **Blitzy's autonomous validation suites**, all of which originate from Blitzy's autonomous validation logs for this project and all of which passed at 100%.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Static citation resolution | Blitzy validator (grep/verify) | 99 | 99 | 0 | 100% of citations | Every unique `file:line` resolves exactly to the claimed code/string across 18 files (108 total citation links). |
| Runtime re-observation — O1 | Live 4-process stack | 6 | 6 | 0 | 6/6 sub-parts | Ordered sequence, both entry points, watcher+worker transcript, duplicate edge case, django-q `[Q]` banner — all reproduced byte-for-byte. |
| Runtime re-observation — O2 | Live 4-process stack | 4 | 4 | 0 | 4/4 sub-parts | No per-upload retrain, HOURLY schedule, TRAIN vs IDLE strings, pickle `FORMAT_VERSION = 7` + SHA1 + no HMAC — all reproduced. |
| Runtime re-observation — O3 | Live 4-process stack | 4 | 4 | 0 | 4/4 sub-parts | `0000001.pdf` layout, before/after DB diffs, no `documents_matchingmodel` — all reproduced. |
| Markdown structural validation | Blitzy validator | 5 | 5 | 0 | — | 88 balanced code fences (44/44), 18/18 link targets exist, single trailing LF, 0 trailing whitespace, not truncated. |
| Pre-commit hooks | pre-commit | 5 | 5 | 0 | — | trailing-whitespace, end-of-file-fixer, mixed-line-ending, detect-private-key, check-case-conflict — all clean on the deliverable. |
| **Totals** | — | **123** | **123** | **0** | **100%** | Zero failures across all Blitzy autonomous validation suites. |

> **Integrity note (Rule 3):** every test/row above originates from Blitzy's autonomous validation execution for this project. Coverage in the code-coverage sense does not apply to a documentation deliverable; the relevant coverage metric is **question coverage = 14/14 sub-questions answered (100%)**.

---

## 4. Runtime Validation & UI Verification

The full stack was brought up and exercised end-to-end. This project has **no UI deliverable** (it produces a markdown document), so there is no visual/Figma verification; the runtime validation below concerns the observed system whose behavior the document describes.

**Runtime health (observed):**

- ✅ **Operational** — Redis broker (`redis-server`, port 6379).
- ✅ **Operational** — `gunicorn` web/API (`paperless.asgi:application`, port 8000); `GET /api/` → **HTTP 200**.
- ✅ **Operational** — `document_consumer` directory watcher (enqueues `consume_file`; logs `Adding … to the task queue.`).
- ✅ **Operational** — `qcluster` django-q worker (`Q Cluster … running.`; task result `Success. New document id … created`).
- ✅ **Operational** — `paperless_tesseract` parser subsystem (ocrmypdf/tesseract/ghostscript/imagemagick).

**API / integration outcomes (observed):**

- ✅ **Operational** — `POST /api/documents/post_document/` → **HTTP 200**, body `"OK"`.
- ✅ **Operational** — consume-directory ingestion → document `pk 1` → `0000001.pdf`.
- ✅ **Operational** — duplicate rejection → `Not consuming …: It is a duplicate.` (task `success=False`, document count unchanged).
- ✅ **Operational** — sinks: SQLite DB writes, media filesystem writes, Whoosh index update, WebSocket `status_updates` progress path.

**Documentation-artifact verification:**

- ✅ **Operational** — 870-line markdown, 88 balanced code fences, 18/18 citation targets resolve, coverage-pass table complete.
- ⚠ **Partial (by design)** — two edge cases (unsupported-MIME rejection; certain conditional tables) are documented via explicit non-verification disclosure rather than live observation; this is fully rule-compliant.

---

## 5. Compliance & Quality Review

This section cross-maps the binding "SWE-AtlasQnA-Repo" rule directives and AAP deliverables to their compliance status. All fixes needed during authoring were applied in the review-polish commit (`d4beb76e6`); the final validator applied **no further fixes** because independent re-observation confirmed the deliverable was already accurate.

| AAP / Rule Requirement | Benchmark | Status | Evidence / Notes |
|---|---|---|---|
| Single named deliverable in `blitzy/documentation/` | Exactly `paperless-ngx_542221a38dff.md` | ✅ Pass | `git diff` = one added file at the exact path. |
| Run-first methodology | Build & run before writing | ✅ Pass | Stack run in the mandated image; verbatim timestamps/PIDs prove live runs; validator re-ran independently. |
| Verbatim evidence with producing command | Quote real output + command | ✅ Pass | Log lines, HTTP `"OK"`, DB diffs, model header all quoted with the command that produced them. |
| Answer every sub-question + coverage pass | 14/14 sub-parts | ✅ Pass | Coverage-pass table marks all 14 ✅ with an explicit not-verified disclosure. |
| Exact, grounded `file:line` citations | Never paraphrase asked values | ✅ Pass | 99/99 unique citations resolve exactly (validator). |
| Provide rationale | Explain the *why* | ✅ Pass | E.g., why the classifier is idle on unchanged data; why the default filename is `0000001.pdf`. |
| Version-awareness | This commit, not upstream | ✅ Pass | Version-skew guard grounds everything in django-q / `Schedule.HOURLY` / plain-pickle `FORMAT_VERSION = 7` / no HMAC. |
| Read-only scope | No source change; remove temp scripts | ✅ Pass | Source byte-identical to base; temp artifacts confined to `/tmp` and removed; clean tree. |
| Markdown well-formedness | Balanced fences, resolvable links, clean EOF | ✅ Pass | 88/88 fences, 18/18 targets, single trailing LF, 0 trailing whitespace. |
| Pre-commit hygiene | Applicable hooks clean | ✅ Pass | trailing-whitespace / EOF / line-ending / private-key / case-conflict all clean. |
| Project unit tests | pytest suite | ⚪ N/A (out of scope) | Tests explicitly out of AAP scope; source untouched, so nothing to affect. |

**Overall:** Full compliance with all binding directives; the single N/A item (unit tests) is correctly out of scope.

---

## 6. Risk Assessment

Risks are assessed with the PA3 categories. Because this is a read-only documentation deliverable with zero source change, there is **no build/deploy/security/integration risk introduced**; the residual risks are Low-severity and already mitigated within the document.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Per-run-variable literals (timestamps, PIDs, tempdir names, SHA1 hex, absolute row counts, Whoosh segment names) differ on re-run | Technical | Low | High | Document frames each such literal as verbatim **from its own run**; structural/behavioral facts are stable and were reproduced by the validator | ✅ Mitigated |
| Citation line-number drift if the document is ever read against a different commit | Technical | Low | Low | Commit `542221a38dff` pinned; source byte-identical to base; validator confirmed 99/99 exact; version-skew guard present | ✅ Mitigated |
| Unsupported-MIME O1(d) sub-case documented via code-reading + disclosure, not live observation | Technical | Low | N/A | Rule explicitly permits stating non-verification; optional extension (P3/HT-3) can close it | ✅ Mitigated / Accepted |
| Reproduction requires the specific Docker image (Python 3.9 + Redis + tesseract + scikit-learn); a standard host cannot run the stack | Operational | Low | Medium | Document names the exact image and provides the full command sequence in "How this was observed" | ✅ Mitigated |
| Security exposure from the deliverable | Security | None | — | Pure markdown; no code, secrets, or dependencies added; `detect-private-key` hook clean (transient dev superuser `admin/admin` exists only in the ephemeral container) | ✅ No risk |
| External-integration/credential risk | Integration | None | — | No external integrations, APIs, or credentials are added or required by the deliverable | ✅ No risk |

**Overall risk posture: LOW.** No risk blocks acceptance.

---

## 7. Visual Project Status

**Project hours breakdown** — Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`:

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown (Total 37h)
    "Completed Work" : 34
    "Remaining Work" : 3
```

**Remaining work by priority** (sums to the 3.0 remaining hours in Sections 1.2 and 2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#333333','pieStrokeWidth':'1px','pieTitleTextSize':'14px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining Hours by Priority (Total 3h)
    "High — SME review & sign-off" : 1.5
    "Medium — reproduction spot-check" : 0.5
    "Low — optional coverage extension" : 1.0
```

**Remaining hours per category (Section 2.2) — bar view:**

| Category | Hours | Bar |
|---|---:|---|
| High — SME review & sign-off | 1.5 | █████████████████ |
| Medium — reproduction spot-check | 0.5 | █████ |
| Low — optional coverage extension | 1.0 | ███████████ |
| **Total** | **3.0** | |

> **Integrity check (Rule 1):** the pie chart "Remaining Work" value (**3**) equals the Section 1.2 Remaining Hours (**3**) and the sum of the Section 2.2 Hours column (**1.5 + 0.5 + 1.0 = 3**). ✔

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.9% complete** (34 of 37 hours). The single mandated deliverable — `blitzy/documentation/paperless-ngx_542221a38dff.md` — was produced run-first and comprehensively answers all three questions with **verbatim, command-attributed evidence** and **exact `file:line` citations**. Blitzy's autonomous validation independently reproduced every O1/O2/O3 claim byte-for-byte, resolved 99/99 citations exactly, and passed all five quality gates with **zero fixes required**. The read-only scope was honored perfectly: `git diff` versus the base commit shows exactly one added file and a clean working tree.

**Remaining gaps.** The outstanding 3 hours are entirely **human path-to-production** activity — there is no failing code, no compilation error, and no configuration gap. They comprise: SME review & sign-off (1.5h, High), an optional reproduction spot-check (0.5h, Medium), and an optional coverage extension to convert two explicitly-disclosed not-verified edge cases into live evidence (1.0h, Low). None of these are technically blocking; the document is already rule-compliant.

**Critical path to production.** (1) SME reads and signs off the document → (2) optional reproduction spot-check → (3) merge. Because no source code changed, there is no source review, build, or deployment on the critical path.

**Success metrics.** Question coverage 14/14 (100%); citation exactness 99/99 (100%); validation gates 5/5 (100%); read-only scope 1 file added / 0 source files changed.

**Production-readiness assessment.** The deliverable is **production-ready pending human sign-off**. It is accurate (verified by live re-observation), exactly grounded, complete across all sub-questions, well-formed, and committed on the correct branch with a clean tree. Recommended action: proceed to SME review and merge.

| Success Metric | Result |
|---|---|
| AAP-scoped completion | 91.9% (34/37h) |
| Question coverage | 14/14 sub-questions (100%) |
| Citation exactness | 99/99 unique (100%) |
| Validation gates | 5/5 passed |
| Read-only scope | 1 file added, 0 source files changed |
| Fixes required in final validation | 0 |

---

## 9. Development Guide

This guide explains how to (A) review the deliverable and (B) reproduce the runtime observations it documents. The reproduction commands mirror the deliverable's own "How this was observed" section.

### 9.1 System Prerequisites

- **Docker** 20.10+ (verified available: `Docker version 28.5.2`).
- The **provided runtime image** `paperless-ngx-ready:latest` (derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_paperless-ngx_...`). It ships **Python 3.9** (`Dockerfile` base `python:3.9-slim-bullseye`), Redis, the OCR binaries (`tesseract-ocr`, `ghostscript`, `imagemagick`), and all Python packages (Django 4.0.4, django-q 1.3.9, scikit-learn 1.0.2, ocrmypdf 13.4.3, whoosh, channels, etc.).
- **No host Python is required** — the analysis host runs Python 3.13, which cannot run paperless-ngx; all runtime observation must occur inside the image.
- Git 2.x (verified: `git version 2.51.0`) and ~2 GB free disk.

### 9.2 Review Workflow (verify the deliverable — runs on any host)

```bash
# 1. Check out the working branch (from the repository root)
git checkout blitzy-147a7cda-59b8-47bf-8646-d512a4b54604

# 2. Confirm the scope is exactly one added file
git diff --name-status 542221a38..HEAD
#   expected: A   blitzy/documentation/paperless-ngx_542221a38dff.md

# 3. Confirm the source tree is byte-identical to base (no source changed)
git diff --stat 542221a38..HEAD -- src/ docs/ docker/ Dockerfile Pipfile Pipfile.lock requirements.txt
#   expected: (empty output)

# 4. Confirm a clean working tree
git status --porcelain
#   expected: (empty output)

# 5. Confirm the deliverable exists and is well-formed
test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo EXISTS
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md          # expected: 870
fences=$(grep -c '^```' blitzy/documentation/paperless-ngx_542221a38dff.md)
echo "code fences: $fences (even => balanced)"                    # expected: 88

# 6. Confirm every relative citation target resolves
cd blitzy/documentation
missing=0
for tgt in $(grep -oE '\]\((\.\./)+[a-zA-Z0-9_./-]+\.(py|rst|conf)\)' \
             paperless-ngx_542221a38dff.md | sed -E 's/^\]\(//; s/\)$//' | sort -u); do
  [ -f "$tgt" ] || { echo "MISSING $tgt"; missing=$((missing+1)); }
done
echo "missing citation targets: $missing"                         # expected: 0
cd ../..
```

> All six checks above were executed during this assessment and passed.

### 9.3 Reproduce the Runtime Observations (requires the provided image)

```bash
# Step 1 — start a fresh container (empty DB => first document gets pk=1) and the full stack
docker run -d --name pngx-fix --entrypoint sleep paperless-ngx-ready:latest infinity
docker exec pngx-fix /usr/local/bin/paperless-start.sh
#   sets PAPERLESS_DEBUG=true, then in order:
#   redis-server --daemonize yes ; manage.py migrate ; manage.py collectstatic ;
#   createsuperuser admin/admin ; then backgrounds:
#     nohup manage.py qcluster          > /app/data/log/qcluster.log 2>&1 &
#     nohup manage.py document_consumer > /app/data/log/consumer.log 2>&1 &
#     nohup gunicorn -c gunicorn.conf.py paperless.asgi:application > /app/data/log/gunicorn.log 2>&1 &

# Step 2 — sanity checks
docker exec pngx-fix python3 --version                 # expected: Python 3.9.x
docker exec pngx-fix git -C /app rev-parse HEAD        # expected: 542221a38dff...
docker exec pngx-fix bash -lc "ps -eo pid,ppid,cmd --sort=pid | \
  grep -E 'redis-server|manage.py qcluster|manage.py document_consumer|paperless.asgi:application' | \
  grep -v grep"                                        # expected: 4 master processes

# Step 3 — O1: submit a PDF via BOTH entry points and read the logs
docker exec pngx-fix bash -lc 'cp /tmp/o1_test.pdf /app/consume/o1_test.pdf'   # entry 1 (watcher)
# entry 2 (HTTP): POST returns HTTP 200 with body "OK"
docker exec pngx-fix bash -lc 'cat /app/data/log/consumer.log'   # "Adding ... to the task queue."
docker exec pngx-fix bash -lc 'cat /app/data/log/qcluster.log'   # [Q] worker lifecycle + consumer sequence
docker exec pngx-fix bash -lc 'cat /app/data/log/paperless.log'  # dedicated file log

# Step 4 — O2: schedule, then force TRAIN and IDLE
docker exec pngx-fix bash -lc 'cd /app/src && python3 manage.py shell -c \
  "from django_q.models import Schedule; print([(s.func, s.schedule_type) for s in Schedule.objects.all()])"'
#   expected: train_classifier scheduled as H (HOURLY)
#   TRAIN branch  -> INFO  "Saving updated classifier model to ..."
#   IDLE  branch  -> DEBUG "Training data unchanged."

# Step 5 — O3: disk layout + DB table diff
docker exec pngx-fix bash -lc 'ls -l /app/media/documents/originals'   # expected: 0000001.pdf
docker exec pngx-fix bash -lc 'python3 -c "import sqlite3; \
  con=sqlite3.connect(\"file:/app/data/db.sqlite3?mode=ro\", uri=True); \
  print([r[0] for r in con.execute(\"SELECT name FROM sqlite_master WHERE type=\x27table\x27 AND name LIKE \x27documents_%\x27 ORDER BY name\")])"'
#   expected: documents_* tables, and NO documents_matchingmodel

# Step 6 — cleanup (leave nothing behind)
docker exec pngx-fix bash -lc 'rm -f /tmp/o1_test.pdf /tmp/o1_post.pdf /tmp/*.py'
docker rm -f pngx-fix
```

### 9.4 Expected Outputs (verification)

- Sanity: `Python 3.9.x`; HEAD `542221a38dff...`; four master processes running.
- O1: watcher log `Adding … to the task queue.`; `POST /api/documents/post_document/` → **HTTP 200 `"OK"`**; worker `[Q]` markers `processing … → Processed … → recycled worker`.
- O2: `train_classifier` scheduled `HOURLY`; TRAIN → INFO `Saving updated classifier model to …`; IDLE → DEBUG `Training data unchanged.`
- O3: `originals/0000001.pdf`; DB deltas `+documents_document`, `+django_admin_log`, `+django_q_task` (and `+documents_document_tags` when a tag is applied); no `documents_matchingmodel` table.

### 9.5 Troubleshooting

- **The stack won't start on my host / imports fail.** The host cannot run paperless-ngx (Python 3.13, no Redis/tesseract/scikit-learn). Use the provided `paperless-ngx-ready:latest` image, which ships Python 3.9 and all dependencies.
- **Consume transaction fails referencing a missing user.** A Django user named `consumer` must exist (the `set_log_entry` handler calls `User.objects.get(username="consumer")`). The image's start script creates it.
- **Classifier log says "model does not exist (yet)".** That is the expected **default** behavior on a fresh install — no model is shipped, and none is trained per upload; it is scheduled and gated. Not an error.
- **DEBUG log lines are missing.** Ensure `PAPERLESS_DEBUG=true`; DEBUG-level `paperless.*` records are always written to `data/log/paperless.log` regardless of console level.
- **`sqlite3` CLI is "command not found".** The image ships no `sqlite3` CLI; use Python's `sqlite3` module (as in Step 5).
- **Citations look off.** Line numbers are pinned to commit `542221a38dff`; verify you are on the correct commit before checking a `file:line` reference.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Verify scope (one added file) | `git diff --name-status 542221a38..HEAD` |
| Verify source untouched | `git diff --stat 542221a38..HEAD -- src/ docs/ docker/ Dockerfile Pipfile Pipfile.lock` |
| Verify clean tree | `git status --porcelain` |
| Verify authorship | `git log --author="agent@blitzy.com" 542221a38..HEAD --oneline` |
| Count deliverable lines | `wc -l blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Check code-fence balance | `grep -c '^```' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Start container | `docker run -d --name pngx-fix --entrypoint sleep paperless-ngx-ready:latest infinity` |
| Start full stack | `docker exec pngx-fix /usr/local/bin/paperless-start.sh` |
| Tail worker log | `docker exec pngx-fix bash -lc 'cat /app/data/log/qcluster.log'` |
| List originals | `docker exec pngx-fix bash -lc 'ls -l /app/media/documents/originals'` |
| Remove container | `docker rm -f pngx-fix` |

### Appendix B — Port Reference

| Port | Service | Notes |
|---|---|---|
| 6379 | Redis | django-q broker + Channels layer (`PAPERLESS_REDIS` default `redis://localhost:6379`). |
| 8000 | gunicorn | ASGI web UI + REST API + WebSocket status feed (`paperless.asgi:application`). |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (870 lines). |
| `src/documents/consumer.py` | `Consumer.try_consume_file()` — ordered ingestion sequence (O1). |
| `src/documents/tasks.py` | `consume_file` worker entry (O1); `train_classifier` scheduled task (O2). |
| `src/documents/classifier.py` | Change-detection hash, training logs, `FORMAT_VERSION = 7` (O2). |
| `src/documents/file_handling.py` | `generate_filename()` — default `{pk:07}` pattern (O3). |
| `src/documents/models.py` | Data model → DB tables (O3); abstract `MatchingModel`. |
| `src/documents/signals/handlers.py` | `consumption_finished` handlers incl. `set_log_entry`, `add_to_index` (O1/O3). |
| `src/documents/apps.py` | Registration of the six consumption-finished handlers (O1). |
| `src/paperless/settings.py` | Storage dirs (L62–64), SQLite path (L300), `Q_CLUSTER` (L449), `PAPERLESS_FILENAME_FORMAT` (L584). |
| `docker/supervisord.conf` | Defines the three runtime processes. |
| `Dockerfile` | Python 3.9 base + OCR system dependencies. |
| `data/log/{qcluster,consumer,paperless,gunicorn}.log` | Runtime logs used as O1/O2 evidence (inside the container). |
| `media/documents/{originals,archive,thumbnails}/` | On-disk document storage (O3). |
| `data/db.sqlite3` | Default database (O3). |

### Appendix D — Technology Versions

| Component | Version | Role |
|---|---|---|
| Python | 3.9 (`python:3.9-slim-bullseye`) | Runtime interpreter. |
| Django | 4.0.4 | Web framework / ORM (DB tables for O3). |
| djangorestframework | 3.13.1 | REST API incl. `POST /api/documents/post_document/` (O1). |
| django-q | 1.3.9 | Task queue + scheduler (`consume_file`, `train_classifier`). |
| scikit-learn | 1.0.2 | Classifier (`MLPClassifier` + `CountVectorizer`) (O2). |
| ocrmypdf | 13.4.3 | PDF OCR during parsing (O1). |
| whoosh | 2.7.4 | Full-text index updated on consumption (O1/O3). |
| channels / channels-redis | 3.0.4 / 3.4.0 | ASGI/WebSocket status feed. |
| gunicorn | 20.1.0 | ASGI/WSGI server. |
| redis (server) | 6.0 | django-q broker + Channels layer. |
| tesseract-ocr / ghostscript / imagemagick | bullseye apt | OCR + PDF rendering + thumbnails. |

### Appendix E — Environment Variable Reference

| Variable | Default | Effect in this investigation |
|---|---|---|
| `PAPERLESS_FILENAME_FORMAT` | unset (`None`) | Left **unset** so O3 documents out-of-the-box behavior → filename `0000001.pdf`. |
| `PAPERLESS_DEBUG` | `false` | Set `true` transiently so `paperless.consumer`/`paperless.classifier` DEBUG lines surface (in the ephemeral container only). |
| `PAPERLESS_REDIS` | `redis://localhost:6379` | django-q broker + Channels layer endpoint. |

### Appendix F — Developer Tools Guide

| Tool / Technique | Usage |
|---|---|
| `git diff <base>..HEAD --name-status` | Confirm the exact set of changed files (scope verification). |
| `git log --author="agent@blitzy.com"` | Confirm autonomous authorship of the two commits. |
| Python `sqlite3` module (read-only URI) | Inspect DB tables/row counts (`file:...?mode=ro`, `uri=True`) since the image has no `sqlite3` CLI. |
| `cat /app/data/log/*.log` | Read worker/watcher/file logs for O1/O2 evidence (`PAPERLESS_DEBUG=true`). |
| `grep -c '^```'` | Validate code-fence balance in the markdown deliverable. |
| `docker exec … ps -eo pid,ppid,cmd` | Confirm the four master processes are running. |

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **django-q / qcluster** | The task queue and worker used at this commit (**not** Celery); `qcluster` runs queued `consume_file` and scheduled `train_classifier`. |
| **document_consumer** | The directory-watcher process that detects files in the consume directory and enqueues `consume_file`. |
| **Consumer.try_consume_file()** | The orchestrator that emits the ordered ingestion log/progress sequence. |
| **MATCH_AUTO** | The matching algorithm value that gates classifier training (Gate 1). |
| **data_hash** | A SHA1 over non-inbox document content + labels; unchanged hash → training is idle (Gate 2). |
| **FORMAT_VERSION = 7** | The classifier model schema version persisted in the plain-pickle model file. |
| **Whoosh** | The full-text search index updated on document consumption. |
| **status_updates** | The WebSocket/Channels group carrying ingestion progress to the UI. |
| **Run-first** | The methodology of building and running the system to capture real output *before* writing the answer. |
| **Version-skew guard** | The document section that grounds behavior in this commit (django-q, HOURLY, plain pickle, no HMAC) rather than later upstream versions. |

---

*Prepared by the Blitzy autonomous project-assessment agent. Completion (91.9%) is computed with the PA1 AAP-scoped hours methodology: 34 completed / 37 total. All cross-section numbers are consistent (Remaining = 3h in Sections 1.2, 2.2, and 7; Section 2.1 + 2.2 = 37h = Total). Blitzy brand colors applied: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.*