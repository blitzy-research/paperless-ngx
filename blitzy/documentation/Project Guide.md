# Blitzy Project Guide — paperless-ngx OCR/Ingestion Runtime Investigation

> Brand palette applied throughout — **Completed / AI Work:** Dark Blue `#5B39F3` · **Remaining / Not Completed:** White `#FFFFFF` · **Headings / Accents:** Violet-Black `#B23AF2` · **Highlight:** Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, documentation-only runtime investigation** of the paperless-ngx document-ingestion / OCR pipeline, motivated by a user report of "inconsistent OCR results" on multi-page PDFs. The objective was to run the software in its default configuration, upload an OCR-requiring PDF through the canonical API, observe real behavior at every stage, and answer four questions (immediate HTTP response, processing-stage logs + exhaustive OCRmyPDF parameters, generated media filenames, database-stored fields) in a single evidence-backed deliverable — `blitzy/documentation/paperless-ngx_542221a38dff.md`. Target users are the maintainers and the reporting user. No production code, configuration, or dependencies were changed; the source tree remains byte-for-byte unchanged.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title AAP-Scoped Completion — 91.3% Complete
    "Completed Work (42.0h)" : 42.0
    "Remaining Work (4.0h)" : 4.0
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **46.0 h** |
| **Completed Hours (AI + Manual)** | **42.0 h** (42.0 h AI · 0.0 h Manual) |
| **Remaining Hours** | **4.0 h** |
| **Percent Complete** | **91.3 %** |

Calculation (PA1, AAP-scoped): `Completion % = Completed ÷ (Completed + Remaining) = 42.0 ÷ 46.0 = 91.3 %`.

### 1.3 Key Accomplishments

- ✅ Full canonical stack brought up in the provided container (Django web server + **mandatory** `django-q` qcluster worker + Redis + default SQLite) with default configuration verified unset.
- ✅ **Q1** answered from observed output: `POST /api/documents/post_document/` returns **HTTP 200**, body **`"OK"`** (`application/json`, Content-Length 4, ~0.103 s), proven to precede processing (log byte-offset delta 0 at the response instant).
- ✅ **Q2** answered exhaustively: full 15-line ordered stage log captured, and **all 13 OCRmyPDF parameters enumerated** with observed values and per-key origin citations; absent keys and the `force_ocr` fallback path explained (primary = 1, fallback = 0).
- ✅ **Q3** answered: default `{pk:07}` media filenames — `originals/0000001.pdf`, `archive/0000001.pdf`, `thumbnails/0000001.png`.
- ✅ **Q4** answered: 15-column `documents_document` schema dumped and mapped to model fields; on-disk path attributes proven to be Python `@property` (not DB columns).
- ✅ Synthesis confirmed the "inconsistent OCR" mechanism = default `OCR_MODE='skip'` → `skip_text=True` (text-bearing pages copied through un-OCR'd on mixed PDFs).
- ✅ 8 edge/error paths (E1–E8) reproduced with before/after side-effect snapshots; final coverage pass completed.
- ✅ **203 `[file:line]` citations across 17 source files verified accurate** at HEAD `542221a38` (zero mismatches).
- ✅ All test artifacts cleaned up; `git status` clean; source tree byte-for-byte unchanged (single new file added).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All AAP-scoped work is complete and independently reproduced; no blocking issues remain. | None | — | — |

> Two pre-existing **product** behaviors are documented honestly and are **out of scope** to fix under the read-only rule (see §8 Informational Findings): a duplicate-upload scratch-file leak and a concurrent-upload TOCTOU checksum race. Neither blocks this deliverable.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| _None identified._ | — | The full stack (web, worker, Redis, SQLite) ran locally in the provided container with no external credentials; reproduction succeeded end-to-end. | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Perform a human technical review of `blitzy/documentation/paperless-ngx_542221a38dff.md` — confirm Q1–Q4 completeness, the 13-parameter OCRmyPDF enumeration, and a sample of citations against source at HEAD `542221a38`. (2.0 h)
2. **[Medium]** Approve and merge the pull request (single new file; source tree unchanged). (0.5 h)
3. **[Low]** Optionally re-run the reproduction in the provided container to independently confirm key observations (Q1 status/body, Q2 args dict, Q3 filenames). (1.5 h)
4. **[Low]** File the two informational product findings (scratch-file leak; TOCTOU checksum race) into the maintainers' backlog for future, separately-scoped work.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime env & full-stack orchestration | 3.5 | Container bring-up; Redis; `migrate` (92 migrations); web server + **mandatory** `qcluster` worker (11 processes); printed runtime settings; concurrency-formula derivation. |
| Admin provisioning & OCR-forcing fixture | 2.0 | Non-interactive superuser; generator for a multi-page **image-only** PDF (reproducible SHA-256) that forces the OCR path under default `skip_text`. |
| Q1 — synchronous HTTP response investigation | 3.0 | Canonical upload driver; captured status/body/headers/timing; log byte-offset delta proof; precise async-ordering measurement. |
| Q2 — stage logs & exhaustive OCRmyPDF parameter enumeration | 6.0 | 15-line ordered stage log mapped to call sites; **all 13** parameters with per-key origin; absent-key rationale; primary/fallback counting; 6 Channels progress events; worker result; duplicate same-input behavior + archive-metadata variance. |
| Q3 — media-filename investigation | 1.5 | Media-directory inspection; `{pk:07}` filename-origin tracing; post-save no-op confirmation. |
| Q4 — DB-fields vs computed-properties investigation | 2.5 | 15-column `PRAGMA` schema; ORM stored values; proof that `source_path`/`archive_path`/`thumbnail_path`/`file_type` are properties, not columns. |
| Synthesis — "inconsistent OCR" mechanism | 2.0 | `skip_text` behavior on mixed PDFs; controlled image-only demonstration; archive-checksum variance shown to be a red herring. |
| Edge/error coverage (E1–E8) + coverage pass | 5.0 | 401×2, 400×2, 405, CORS allowed/foreign, ws/status 302, TOCTOU race; before/after side-effect snapshots; final named-item coverage pass. |
| Deliverable authoring (1119-line markdown) | 5.0 | Structured document: 1 H1 / 10 H2 / 31 H3, 40 reproducible code blocks, per-question sections and tables. |
| Citation discipline & verification (203 tokens / 17 files) | 2.5 | Every factual claim carries a `[file:line]` locator; citations verified for existence and semantics at HEAD `542221a38`. |
| Iterative remediation & reproducibility hardening | 4.0 | Three follow-up commits: addressed 17 review findings, rewrote for true-default reproducibility, completed Q1 response-header evidence (~3,197-line cumulative churn). |
| Final independent validation (5-gate reproduction) | 3.5 | Full-stack rerun reproducing every documented observation line-for-line; citations re-verified; cleanup + git-clean re-confirmed. |
| Cleanup & git-clean verification | 1.5 | ORM delete (cascade), media/scratch clear, pk-sequence reset, Whoosh reindex, PID-exact process stop; git tree confirmed clean. |
| **Total Completed** | **42.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & acceptance of the deliverable | 2.0 | High |
| Independent re-run / spot-reproduction of key observations | 1.5 | Low |
| PR acceptance & merge to target branch | 0.5 | Medium |
| **Total Remaining** | **4.0** | |

> **Consistency:** §2.1 (42.0 h) + §2.2 (4.0 h) = **46.0 h** total (matches §1.2). §2.2 total (4.0 h) equals §1.2 Remaining Hours and the §7 pie "Remaining Work" value.

---

## 3. Test Results

**Nature of testing for this task.** This is a documentation deliverable with **zero production-code changes**, so no application unit-test framework applies. Blitzy's autonomous validation "test suite" is a **full runtime reproduction**: every documented observation was independently reproduced through the canonical entry point, and every citation was verified against source. The results below originate exclusively from **Blitzy's autonomous validation logs** for this project.

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|-------|--------|--------|------------|-------|
| Citation accuracy ("compilation") | grep / manual vs source @ HEAD 542221a38 | 153 | 153 | 0 | 100% | All `[file:line]` citations verified for existence **and** semantics; 0 mismatches (203 total tokens). |
| Q1 — synchronous response | Canonical HTTP (`requests`/`curl`) reproduction | 1 | 1 | 0 | 100% | HTTP 200, body `"OK"`, `application/json`, Content-Length 4, ~0.103 s, log delta 0. |
| Q2 — stage logs & OCR parameters | Runtime log capture at DEBUG | 5 | 5 | 0 | 100% | 15-line stage log; 13-key args dict; primary=1/fallback=0; 6 progress events; worker `success=True`. |
| Q3 — media filenames | Filesystem inspection | 1 | 1 | 0 | 100% | `originals/0000001.pdf`, `archive/0000001.pdf`, `thumbnails/0000001.png`. |
| Q4 — DB-stored fields | SQLite `PRAGMA` + Django ORM | 1 | 1 | 0 | 100% | 15-column schema; stored values; computed-property proof. |
| Synthesis — "inconsistent OCR" | Controlled mixed vs image-only reproduction | 1 | 1 | 0 | 100% | `skip_text` mechanism confirmed; checksum-variance red herring shown. |
| Edge / error paths (E1–E8) | Canonical HTTP reproduction + side-effect snapshots | 8 | 8 | 0 | 100% | 401×2, 400×2, 405, CORS allowed/foreign, ws/status 302, TOCTOU race — each with zero-side-effect proof. |
| Full-stack runtime ("run") | web + qcluster + Redis + SQLite | 1 | 1 | 0 | 100% | End-to-end ingestion succeeded (pk=1); worker task `success=True`. |
| **Total** | | **171** | **171** | **0** | **100%** | All checks sourced from Blitzy autonomous validation logs. |

---

## 4. Runtime Validation & UI Verification

**Runtime health & API integration (default/canonical configuration):**

- ✅ **Operational** — Full stack up: Django web server, `django-q` qcluster worker (11 processes), Redis (`redis-cli ping` → `PONG`), default SQLite (92 migrations applied).
- ✅ **Operational** — Canonical API `POST /api/documents/post_document/` → **HTTP 200 / `"OK"`** synchronously (enqueue acknowledgement).
- ✅ **Operational** — Asynchronous worker ingestion → document created (pk=1), worker result `success=True`, `"Success. New document id 1 created"`.
- ✅ **Operational** — OCR pipeline → archive PDF (`output_type='pdfa'`), thumbnail PNG, and sidecar text produced; all 3 fixture pages OCR'd on the image-only control.
- ✅ **Operational** — Database persistence → 15-column `documents_document` row with expected stored values.
- ✅ **Operational** — Channels/Redis progress broadcast → 6 STARTING/WORKING/SUCCESS events captured at the channel-layer boundary.
- ✅ **Operational** — Cleanup → test document/media/scratch removed; processes stopped by exact PID; `git status` clean.
- ⚠ **Partial (informational, out of scope)** — Duplicate-upload path leaves a staged file in `SCRATCH_DIR`; concurrent identical uploads can hit a UNIQUE-constraint (TOCTOU) race. Documented, not fixed (read-only rule).

**UI Verification:** ❌ **N/A / Not in scope.** This is a documentation-only investigation; no Angular frontend (`src-ui/`) work was in scope and no UI changes were made. The canonical interface exercised is the documented HTTP API, not the web UI.

---

## 5. Compliance & Quality Review

Cross-mapping of the binding "SWE-AtlasQnA-Repo" rules and Blitzy quality benchmarks to observed evidence.

| Benchmark / Rule | Status | Progress | Evidence |
|------------------|--------|----------|----------|
| Single branch-named deliverable | ✅ Pass | 100% | `blitzy/documentation/paperless-ngx_542221a38dff.md` (name = source branch). |
| Run first, then write (observed, not code-read) | ✅ Pass | 100% | Every claim shows the command + complete unedited output. |
| Canonical entry point only (no mocks/debug hooks) | ✅ Pass | 100% | `POST /api/documents/post_document/` via `requests`/`curl` + real worker. |
| Default / canonical configuration | ✅ Pass | 100% | `OCR_MODE` unset→`skip`, `FILENAME_FORMAT` unset, SQLite; purity printed. |
| Exhaustive OCRmyPDF parameter enumeration | ✅ Pass | 100% | All 13 keys tabulated with observed values + per-key origin; absent keys explained. |
| Primary **and** fallback/edge/error paths covered | ✅ Pass | 100% | primary=1/fallback=0; E1–E8 error paths; before/after snapshots. |
| Reproduce inconsistency faithfully | ✅ Pass | 100% | Same unchanged input repeated; skip_text mechanism isolated with image-only control. |
| Exact & grounded (`[file:line]`, named functions) | ✅ Pass | 100% | 203 citations across 17 files; functions named (`PostDocumentView.post`, `construct_ocrmypdf_parameters`, etc.). |
| Read-only scope (no source edits) | ✅ Pass | 100% | `git diff` vs base = only the one new file; source byte-for-byte unchanged. |
| Complete cleanup / no permanent changes | ✅ Pass | 100% | Document/media/scratch/processes removed; `git status` clean. |
| Zero placeholders (TODO/FIXME/TBD) | ✅ Pass | 100% | 0 placeholder markers in the deliverable. |
| Citation accuracy | ✅ Pass | 100% | 153 distinct citations verified for existence + semantics; 0 mismatches. |
| Markdown hygiene (pre-commit) | ✅ Pass | 100% | 0 trailing-whitespace, 0 tabs, LF, ends-with-newline; 40 balanced code fences. |

**Fixes applied during autonomous validation:** none required — the deliverable reproduced 100% and every citation was accurate, so validation was a NO-OP (editing would have broken internal consistency and violated the "show actual observed output" rule). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

Overall posture: **LOW** — a read-only documentation task that changes zero production code, adds zero dependencies, and introduces zero new attack surface. No High/Critical risks.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Run-specific values (checksums, timestamps, task UUIDs, scratch suffixes, byte offsets) are non-reproducible | Technical | Low | Medium | Document labels these as run/fixture-specific and never claims them fixed | Mitigated |
| Citation line-number drift if source is later modified | Technical | Low | Low | All 203 citations anchored to HEAD `542221a38` (stated in appendix) | Mitigated |
| `jobs=11` is environment-derived (cpu_count 128); differs on other hardware | Technical | Low | Medium | Derivation cited `[settings.py:L469-L471]`; explained in dev guide | Mitigated |
| Reproduction requires both web server **and** qcluster worker | Technical | Low | Medium | Dev guide mandates starting `qcluster`; otherwise Q2 logs never appear | Mitigated |
| Ephemeral test credential `admin:admin` shown in commands | Security | Low | Low | Throwaway superuser in ephemeral container; deleted at cleanup; never committed | Mitigated |
| Zero new attack surface (no code/config/dep change) | Security | Low | Low | Read-only task; single markdown file added | No risk introduced |
| Duplicate-upload staged-file leak (pre-existing product behavior) | Security/Operational | Low | Low | Documented honestly; out of scope to fix (read-only) — routed to product backlog | Documented (out of scope) |
| Concurrent-upload TOCTOU checksum race (pre-existing product behavior) | Security/Operational | Low | Low | Documented honestly; out of scope to fix — routed to product backlog | Documented (out of scope) |
| Leftover runtime artifacts after reproduction | Operational | Low | Low | Cleanup section + validator-confirmed 0 leaks; PID-exact process stop; git clean | Mitigated |
| Redis unreachable → pipeline won't run | Integration | Low | Low | Dev guide starts + verifies `redis-server` (`redis-cli ping`) | Mitigated |
| Native OCR toolchain absent outside the provided container | Integration | Low | Medium | Dev guide mandates the provided container (bundles tesseract/gs/qpdf/unpaper/pngquant) | Mitigated |
| Nothing deployed (no monitoring/rollback surface) | Operational | Low | Low | Documentation artifact only; no deployment | N/A |

---

## 7. Visual Project Status

**Hours breakdown (Completed vs Remaining):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 42.0
    "Remaining Work" : 4.0
```

**Remaining work by priority (hours from §2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining Hours by Category (Total 4.0h)
    "Human technical review [High]" : 2.0
    "Independent re-run [Low]" : 1.5
    "PR acceptance & merge [Medium]" : 0.5
```

> **Integrity:** the "Remaining Work" value (4.0 h) equals §1.2 Remaining Hours and the sum of the §2.2 Hours column. "Completed Work" (42.0 h) equals the §2.1 total. Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **91.3 % complete** on an AAP-scoped basis (42.0 of 46.0 hours). All four questions are answered from observed runtime output with complete, unedited evidence and precise `[file:line]` citations: Q1 (immediate `HTTP 200`/`"OK"` enqueue acknowledgement), Q2 (full stage log + **all 13** OCRmyPDF parameters, plus the fallback path), Q3 (default `{pk:07}` media filenames), and Q4 (15 stored DB columns vs. computed path properties). The synthesis empirically confirms the user's reported "inconsistent OCR results" arises from the default `OCR_MODE='skip'` (`skip_text=True`), which copies text-bearing pages through un-OCR'd on mixed PDFs. Eight edge/error paths were reproduced with zero-side-effect proofs, and the entire deliverable was independently reproduced 100% by the final validator.

**Remaining gaps & critical path to production.** For a documentation artifact, "production" means acceptance and merge. The critical path is short: **(1)** human technical review (2.0 h) → **(2)** PR acceptance & merge (0.5 h), with an **optional** independent re-run (1.5 h) for extra assurance — **4.0 h** total. There is no build, deployment, CI/CD, or environment configuration to complete because the source repository is unchanged.

**Production-readiness assessment.** ✅ **Ready for review.** The single in-scope deliverable is complete, accurate, internally self-consistent, and fully cited; the source tree is byte-for-byte unchanged; `git status` is clean; and every claim has been independently reproduced. No blocking issues remain.

**Informational findings (out of scope — for product backlog, not part of the 4.0 h).** The document honestly records two pre-existing product behaviors that the read-only rule forbids fixing here: (a) a duplicate-upload staged-file leak in `SCRATCH_DIR`, and (b) a concurrent-upload TOCTOU checksum race. These are recommended as separately-scoped future work for the maintainers.

| Success Metric | Target | Actual |
|----------------|--------|--------|
| Questions answered from observed output | 4 / 4 | ✅ 4 / 4 |
| OCRmyPDF parameters enumerated | All | ✅ 13 / 13 |
| Citation accuracy | 100% | ✅ 100% (0 mismatches) |
| Documented observations reproduced | 100% | ✅ 100% |
| Source files modified | 0 | ✅ 0 (git clean) |

---

## 9. Development Guide

This guide reproduces the investigation. Read-only verification commands run anywhere; the **OCR reproduction must run inside the provided container**, which bundles the native toolchain (this orchestration sandbox has Python 3.13 + Docker but not the paperless Python 3.9 runtime).

### 9.1 System Prerequisites

- The **provided container** (image bundles: tesseract 4.1.1, ocrmypdf 13.4.3, ghostscript 9.53.3, qpdf 10.1.0, unpaper 6.1, pngquant 2.12.2, pdftoppm; Debian 11.11; Python 3.9.23). Note: `jbig2`/`jbig2enc` are **absent** in the image and documented as an edge condition.
- Redis (used as the django-q broker **and** the Channels layer).
- Default SQLite database (no external DB server needed).

### 9.2 Environment Setup — Default-Configuration Purity

Keep the three answer-affecting variables **unset** so the code takes its default branches:

```bash
# All three MUST be unset for canonical/default behavior
for v in PAPERLESS_OCR_MODE PAPERLESS_FILENAME_FORMAT PAPERLESS_DBHOST; do
  printf "%s=[%s]\n" "$v" "${!v-<UNSET>}"
done
# Expected: each prints <UNSET>  → OCR_MODE='skip', FILENAME_FORMAT=None, DB=SQLite
```

Verify the pinned Python dependencies match the manifest:

```bash
grep -iE '^(ocrmypdf|Django|django-q|djangorestframework|channels|channels-redis|redis|pikepdf)==' requirements.txt
# ocrmypdf==13.4.3, Django==4.0.4, django-q==1.3.9, djangorestframework==3.13.1,
# channels==3.0.4, channels-redis==3.4.0, redis==3.5.3, pikepdf==5.1.1
```

### 9.3 Bring Up the Stack (Redis → migrate → superuser → web → worker)

Run inside the provided container (project lives at `/app/src`):

```bash
# 1) Redis (leave as-found if already running)
redis-server --daemonize yes
redis-cli ping            # → PONG

# 2) Apply migrations (idempotent on the pre-migrated image)
cd /app/src && python3 manage.py migrate

# 3) Create the admin superuser (non-interactive)
DJANGO_SUPERUSER_PASSWORD=admin python3 manage.py createsuperuser \
  --noinput --username admin --email admin@example.com

# 4) Start the web server (detached; capture PID)
nohup python3 manage.py runserver 0.0.0.0:8000 --noreload > /tmp/qa_run/runserver.log 2>&1 &
echo "runserver pid=$!"

# 5) Start the MANDATORY django-q worker (detached; capture PID)
#    Without qcluster, ingestion never runs and Q2 stage logs never appear.
nohup python3 manage.py qcluster > /tmp/qa_run/qcluster.log 2>&1 &
echo "qcluster pid=$!"
```

### 9.4 Drive the Canonical Entry Point & Verify

```bash
# Upload a multi-page, IMAGE-ONLY PDF (forces OCR under default skip_text)
curl -u admin:admin -F 'document=@/path/to/fixture.pdf' \
  http://localhost:8000/api/documents/post_document/
# → HTTP 200, body: "OK"   (immediate enqueue acknowledgement — Q1)

# Q2: watch the processing-stage logs (look for the OCRmyPDF invocation)
tail -f /app/data/log/paperless.log | grep -m1 "Calling OCRmyPDF with args:"

# Q3: generated media filenames (default {pk:07})
ls -l /app/media/documents/originals /app/media/documents/archive /app/media/documents/thumbnails
# → originals/0000001.pdf, archive/0000001.pdf, thumbnails/0000001.png

# Q4: stored DB columns (15) via the Django DB cursor
cd /app/src && python3 -c "import os,django; \
os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); \
from django.db import connection; c=connection.cursor(); \
c.execute('PRAGMA table_info(documents_document);'); \
print('\n'.join('|'.join('' if v is None else str(v) for v in r) for r in c.fetchall()))"
```

### 9.5 Cleanup / Reset for Re-run

```bash
cd /app/src && python3 -c "import os,django; \
os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings'); django.setup(); \
from documents.models import Document; Document.objects.all().delete(); \
print('documents_count =', Document.objects.count())"
rm -f /app/media/documents/{originals,archive,thumbnails}/* 2>/dev/null
rm -rf /tmp/paperless/* 2>/dev/null
python3 manage.py document_index reindex --no-progress-bar
kill <runserver_pid> <qcluster_pid>     # stop by EXACT PID — never use pkill
```

### 9.6 Read-Only Verification (works in any checkout)

```bash
git status --porcelain                                   # empty = clean tree
git diff origin/paperless-ngx_542221a38dff --name-status # A blitzy/documentation/paperless-ngx_542221a38dff.md
git log --author="agent@blitzy.com" --oneline            # 4 documentation commits
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md # 1119
```

### 9.7 Troubleshooting

- **No Q2 stage logs / document stuck** → the `qcluster` worker is not running. Start it (§9.3 step 5); it is mandatory.
- **HTTP 401** → missing/invalid Basic credentials — use `-u admin:admin`.
- **HTTP 400** → the `document` field is missing, or the file type is unsupported (rejected before enqueue).
- **HTTP 405** → wrong method; the endpoint accepts `POST` (and `OPTIONS`).
- **`jobs` ≠ 11 in the args dict** → expected; the value is `THREADS_PER_WORKER`, derived from CPU count `[src/paperless/settings.py:L469-L471]`, not a fixed constant.
- **A page shows no OCR text / "OCR skipped on page(s) N"** → that page already had a text layer; under default `skip_text=True` only text-less pages are OCR'd (the "inconsistent OCR" mechanism).
- **Redis errors on startup** → ensure `redis-server` is running (`redis-cli ping` → `PONG`).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Ping Redis | `redis-cli ping` |
| Apply migrations | `python3 manage.py migrate` |
| Create superuser | `DJANGO_SUPERUSER_PASSWORD=admin python3 manage.py createsuperuser --noinput --username admin --email admin@example.com` |
| Start web server | `python3 manage.py runserver 0.0.0.0:8000 --noreload` |
| Start worker (mandatory) | `python3 manage.py qcluster` |
| Canonical upload | `curl -u admin:admin -F 'document=@file.pdf' http://localhost:8000/api/documents/post_document/` |
| Tail pipeline log | `tail -f /app/data/log/paperless.log` |
| List media | `ls -l /app/media/documents/{originals,archive,thumbnails}` |
| Dump DB schema | `PRAGMA table_info(documents_document);` (via Django DB cursor) |
| Rebuild search index | `python3 manage.py document_index reindex --no-progress-bar` |

### B. Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 8000 | Django web server (`runserver`) | Canonical upload endpoint under `/api/` |
| 6379 | Redis | django-q broker + Channels layer (`redis://localhost:6379`) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (1119 lines) |
| `src/paperless/urls.py` | Q1 — upload route under `/api/` |
| `src/documents/views.py` | Q1 — `PostDocumentView.post()` → `Response("OK")` |
| `src/documents/tasks.py` | Q2 — async `consume_file` worker entry |
| `src/documents/consumer.py` | Q2/Q3 — stage logs + filename-assigning persistence |
| `src/paperless_tesseract/parsers.py` | Q2 — OCRmyPDF args + invocation |
| `src/documents/models.py` | Q3/Q4 — DB columns vs. computed path properties |
| `src/documents/file_handling.py` | Q3 — `{pk:07}` filename generation |
| `src/paperless/settings.py` | Defaults: OCR, logging, media layout, DB, concurrency |
| `/app/data/log/paperless.log` | Q2 — DEBUG pipeline log |
| `/app/media/documents/{originals,archive,thumbnails}` | Q3 — generated media |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| Python (runtime) | 3.9.23 (`python:3.9-slim-bullseye`) |
| Django | 4.0.4 |
| djangorestframework | 3.13.1 |
| django-q | 1.3.9 |
| channels / channels-redis | 3.0.4 / 3.4.0 |
| redis (py) | 3.5.3 |
| ocrmypdf | 13.4.3 |
| pikepdf | 5.1.1 |
| tesseract | 4.1.1 |
| ghostscript | 9.53.3 |
| qpdf | 10.1.0 |
| unpaper / pngquant | 6.1 / 2.12.2 |

### E. Environment Variable Reference

| Variable | Default (unset) | Effect if set |
|----------|-----------------|---------------|
| `PAPERLESS_OCR_MODE` | `skip` → `skip_text=True` | `force`/`redo` change OCR coverage behavior |
| `PAPERLESS_FILENAME_FORMAT` | `None` → `{pk:07}` filenames | Custom archive/thumbnail naming |
| `PAPERLESS_DBHOST` | unset → SQLite | Switches to PostgreSQL |
| `PAPERLESS_REDIS` | `redis://localhost:6379` | Alternate broker/Channels backend |
| `PAPERLESS_OCR_LANGUAGE` | `eng` | OCR language(s) |
| `PAPERLESS_OCR_OUTPUT_TYPE` | `pdfa` | Archive output format |
| `PAPERLESS_TASK_WORKERS` | `floor(sqrt(cpu_count))` | qcluster worker process count |
| `PAPERLESS_THREADS_PER_WORKER` | `floor(cpu_count / TASK_WORKERS)` | OCRmyPDF `jobs` value |

### F. Developer Tools Guide

- **git** — verify scope: `git status --porcelain` (clean), `git diff <base> --name-status` (single new file), `git log --author="agent@blitzy.com" --oneline` (4 commits).
- **Django management commands** — `migrate`, `showmigrations`, `createsuperuser`, `qcluster`, `document_index reindex`.
- **Redis CLI** — `redis-cli ping` for liveness.
- **curl / requests** — drive the canonical upload; inspect status/headers with `curl -i`.
- **Chrome DevTools MCP** — not applicable (no UI work in scope for this deliverable).

### G. Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the governing requirements for this task. |
| **Canonical entry point** | The documented `POST /api/documents/post_document/` upload path (no mocks/debug hooks). |
| **`skip_text`** | OCRmyPDF mode (from default `OCR_MODE='skip'`) that copies text-bearing pages through un-OCR'd. |
| **Sidecar** | The plain-text file OCRmyPDF writes with recognized text; paperless reads it into `content`. |
| **qcluster** | The django-q worker process that executes `consume_file` asynchronously. |
| **Archive PDF** | The PDF/A output (`output_type='pdfa'`) stored under `media/documents/archive`. |
| **Computed property** | A model `@property` (e.g. `source_path`) composing a filesystem path at runtime — **not** a DB column. |
| **TOCTOU** | Time-of-check-to-time-of-use race (the documented concurrent-upload checksum race). |

---

*Prepared by the Blitzy autonomous platform. Completion percentage (91.3%) reflects AAP-scoped work only: 42.0 completed of 46.0 total hours, 4.0 hours remaining (human review & acceptance). Colors — Completed `#5B39F3`, Remaining `#FFFFFF`.*