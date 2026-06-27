# Blitzy Project Guide — Paperless-ngx OCR Pipeline Runtime Behavior Q&A

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **knowledge-extraction (Q&A) investigation**, not a code change. At paperless-ngx commit `542221a38dff`, a user reported "inconsistent OCR results" with multi-page PDFs and asked four precise questions about how a document flows from upload → asynchronous OCR → on-disk media → database persistence. The deliverable is a single, evidence-grounded Markdown report that answers each question with code-as-truth citations, connecting rationale, and live runtime output captured by genuinely building and running the stack. The audience is backend engineers triaging the OCR pipeline. Business impact: it converts an ambiguous bug report into a precise, verifiable mental model of the consume pipeline, accelerating future diagnosis — without modifying any source file.

### 1.2 Completion Status

The project is **91.4% complete** on an AAP-scoped basis. All autonomous work (the full investigation and the validated report) is finished; the only remaining work is human subject-matter-expert (SME) review and acceptance — an inherently human gate for a knowledge deliverable.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData
    title Completion Status — 91.4% Complete
    "Completed Work (AI)" : 32
    "Remaining Work" : 3
```

| Metric | Hours |
|---|---|
| **Total Hours** | **35** |
| Completed Hours (AI + Manual) | 32 (AI 32 + Manual 0) |
| Remaining Hours | 3 |
| **Percent Complete** | **91.4%** |

> Completion is computed by the PA1 hours method: `Completed / (Completed + Remaining) = 32 / (32 + 3) = 32/35 = 91.4%`. Per honest-assessment policy, completion is capped below 100% until human review closes the acceptance gate.

### 1.3 Key Accomplishments

- ✅ **All four user questions answered** with the structure Answer → Evidence (`file:line` citations) → Rationale → Observed live runtime output.
- ✅ **Q1 (HTTP response):** confirmed `200 OK` + JSON body `"OK"`, returned synchronously *before* processing (`Response("OK")`, `views.py:L535`).
- ✅ **Q2 (logs + OCR args):** captured the ordered `paperless.consumer` / `paperless.parsing.tesseract` stage lines and the full OCRmyPDF kwargs dict live, with the host-dependent `jobs` value expressed as a formula.
- ✅ **Q3 (filenames):** confirmed `originals/0000001.pdf`, `archive/0000001.pdf`, `thumbnails/0000001.png` (PNG in this commit) from live `ls` and `generate_filename`.
- ✅ **Q4 (DB columns):** enumerated the **15** stored columns (with live `pk=1` values) and separated them from the computed `@property` paths (`source_path`/`archive_path`/`thumbnail_path`/`file_type`).
- ✅ **Live stack validated end-to-end** (Redis + Django-Q `qcluster` + gunicorn ASGI :8000); a real 3-page image-only PDF processed successfully.
- ✅ **Full autonomous test suite green:** 481 passed / 0 failed / 2 intentional skips (483 total).
- ✅ **All AAP constraints honored:** zero source files modified (only the one report added), correct location/filename, full cleanup verified, lint-clean, committed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None — no blocking issues_ | The report is validation-complete; all four answers verified against source + live runtime with zero discrepancies | — | — |
| Awaiting human SME acceptance (non-blocking, by design) | Knowledge deliverable is "validator-verified" but not yet "human-accepted" | Backend SME | ~2.0h |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _No access issues identified_ | — | The investigation ran fully within the provided container image; repository read/write, Redis, and the ASGI endpoint were all reachable | Resolved / N/A | — |

No access issues identified. The repository, the provided runtime image (Python 3.9 + Tesseract + Ghostscript + qpdf + Redis), and the upload endpoint were all accessible throughout.

### 1.6 Recommended Next Steps

1. **[High]** Perform SME technical review & sign-off of `blitzy/documentation/paperless-ngx_542221a38dff.md`: read the four answers and spot-check 10–15 of the 80+ `file:line` citations against the pinned source (≈2.0h).
2. **[Medium]** Optionally reproduce the live observation at commit `542221a38dff` using the Section 9 bring-up commands to independently confirm Q1 (`200 "OK"`) and Q3 filenames (≈0.5h).
3. **[Low]** Distribute/link the report to the original requester and index it in the team knowledge base (≈0.5h).
4. **[Low]** If the underlying "inconsistent OCR" behavior must be *fixed* (explicitly out of scope here), open a separate engineering task using this report as the diagnostic baseline.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each component traces to an AAP objective (O0–O4) or a path-to-production activity. All hours are AI/autonomous.

| Component | Hours | Description |
|---|---|---|
| Environment bring-up & forced-OCR fixture [O0] | 5 | Stand up the Docker dev stack (Redis, `migrate`, `manage_superuser`, `qcluster`, gunicorn ASGI); generate a 3-page **image-only** PDF (verified zero embedded text) and upload it; troubleshoot infra deltas (jbig2enc absent, libzbar0 for the `pyzbar` import, ImageMagick PDF policy). |
| Q1 — HTTP upload-response analysis & write-up (§3) [O1] | 3 | Trace `urls.py` → `PostDocumentView.post` → `PostDocumentSerializer`; confirm `Response("OK")` = `200`; capture live `curl`; document `400`/auth edge cases + in-repo test corroboration. |
| Q2 — Stage logs + OCRmyPDF args analysis & write-up (§4) [O2] | 8 | Trace `consumer.py` stage logs + `loggers.py` routing + `parsers.construct_ocrmypdf_parameters`; capture live `qcluster` log stream; build the line-by-line arg table; derive the `THREADS_PER_WORKER`/`jobs` formula; document mode variations and the log-routing caveat. |
| Q3 — Media filename analysis & write-up (§5) [O3] | 3 | Trace `generate_filename` default branch + `thumbnail_path` property + mime→extension map; capture live `ls` of originals/archive/thumbnails. |
| Q4 — DB columns vs derived metadata analysis & write-up (§6) [O4] | 4 | Read `models.Document`; run live `PRAGMA table_info` + Django shell to capture all 15 columns with `pk=1` values; document `_store` population and post-consume signal handlers; separate computed `@property` non-columns. |
| Report scaffolding (§1/§2/§7/§9) | 4 | Methodology & environment section, process-topology table, critical version caveat (dependency cross-checks), summary table, overall Markdown structure/formatting. |
| Cleanup execution & non-persistence verification (§8) | 2 | Targeted row delete; sweep media/Whoosh/scratch; document the duplicate-upload temp-file nuance; post-cleanup verification commands; `git status` clean proof. |
| Evidence grounding + validation/QA refinements | 3 | Cross-check 80+ citations against source; three QA refinement commits (address code-review findings; fix the 8-core `jobs` value in §4.5; refine the OCRmyPDF log-routing caveat + normalize EOF). |
| **Total Completed** | **32** | |

### 2.2 Remaining Work Detail

Each category is a path-to-production activity for a knowledge deliverable (human-only work).

| Category | Hours | Priority |
|---|---|---|
| SME technical review & sign-off (validate Q1–Q4; spot-check citations vs pinned source; confirm cleanup & version caveat) | 2.0 | High |
| Independent reproduction spot-check (re-run upload at commit `542221a38`; confirm Q1 `200 "OK"` + Q3 filenames) | 0.5 | Medium |
| Distribution & archival (share with original requester; index in knowledge base) | 0.5 | Low |
| **Total Remaining** | **3.0** | |

### 2.3 Hours Reconciliation

- Section 2.1 total (Completed) = **32h**
- Section 2.2 total (Remaining) = **3h**
- **2.1 + 2.2 = 32 + 3 = 35h = Total Project Hours** (Section 1.2). ✓
- Completion = 32 / 35 = **91.4%** (matches Sections 1.2, 7, 8). ✓

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs (Final Validator GATE 1). The full backend suite was executed as a non-root user with `CI=true python3 -m pytest -p no:cacheprovider --no-cov -n 8`.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Backend suite (Unit + Integration + API) | pytest + pytest-django (xdist `-n 8`) | 483 | 481 | 0 | Not measured (`--no-cov` run) | 2 intentional source-level `@skip`; 0 errors; run as non-root `testuser`, `CI=true` |

**Doc-cited tests that specifically corroborate the four answers (all passing):**

| Test | Location | Corroborates |
|---|---|---|
| `test_upload` | `src/documents/tests/test_api.py:L762` | Q1 — `200` for a valid upload (with `async_task` mocked → proves the response does not wait for processing) |
| `test_upload_invalid_form` | `src/documents/tests/test_api.py:L808` | Q1 edge — `400` for a missing `document` field |
| `test_upload_invalid_file` | `src/documents/tests/test_api.py:L822` | Q1 edge — `400` for an unsupported type |
| `test_ocrmypdf_parameters` | `src/paperless_tesseract/tests/test_parser.py:L427` | Q2 — builds the OCRmyPDF args dict; asserts the `OCR_CLEAN`/`OCR_DESKEW` matrices |
| `TestParserFileTypes` | `src/paperless_tesseract/tests/test_parser.py:L474` | Q2/Q3 — image inputs produce an archive PDF + extracted OCR text |

> **Integrity note:** No tests were authored for this task (it is read-only). The figures above are the project's *existing* suite as executed by Blitzy's autonomous validation, used purely as runtime corroboration of the documented behavior.

---

## 4. Runtime Validation & UI Verification

**Runtime health (live stack at commit `542221a38dff`):**

- ✅ **Redis broker** — `redis-cli ping` → `PONG` (the hard prerequisite carrying tasks web → worker).
- ✅ **Django-Q `qcluster` worker** — reported **11 worker processes** ready; executed `documents.tasks.consume_file`; task result `Success. New document id 1 created`.
- ✅ **gunicorn ASGI web server** (`paperless.asgi:application`, :8000) — `GET /api/` → `200` with `admin:admin`.
- ✅ **End-to-end consume** — a real 3-page image-only PDF (verified 0 embedded text) processed through OCRmyPDF; archive PDF/A, thumbnail, and DB row all produced.
- ✅ **Database persistence** — `Document` row `pk=1` created with all 15 columns populated (`content` = OCR text proves OCR genuinely ran).

**API integration:**

- ✅ `POST /api/documents/post_document/` → `200` + body `"OK"` (synchronous accept).
- ✅ Duplicate re-upload → also `200`, but the worker rejected it as a duplicate — empirically proving `200` signals acceptance only, not processing success.

**UI verification:**

- ⚠ **Not applicable — by design.** This is a backend OCR/document-processing investigation with **no UI deliverable**. The AAP (§0.8) confirms there is no design surface, no Figma frames, and no UI design compliance sub-section. The Angular SPA (`src-ui/`) exists in the repository but is explicitly out of scope and was not modified. The "standard interface" under investigation is the REST upload endpoint, which was verified via `curl` (above).

---

## 5. Compliance & Quality Review

AAP deliverables and binding constraints cross-mapped to quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Requirement | Status | Evidence / Notes |
|---|---|---|
| O0 — Environment standup + admin + image-only PDF upload | ✅ Pass | Live stack; `manage_superuser` admin; forced-OCR fixture uploaded |
| O1/Q1 — HTTP upload response documented | ✅ Pass | §3: `200 "OK"`; `views.py:L535`; live `curl`; tests corroborate |
| O2/Q2 — Stage logs + OCRmyPDF args documented | ✅ Pass | §4: live log capture; line-by-line arg table; `jobs` formula |
| O3/Q3 — Archive PDF + thumbnail filenames documented | ✅ Pass | §5: live `ls`; `0000001.pdf` / `0000001.png` (PNG) |
| O4/Q4 — Persisted columns vs derived metadata documented | ✅ Pass | §6: 15 columns via live `PRAGMA`; `@property` non-columns separated |
| Code-as-truth evidence (no assumptions) | ✅ Pass | 80+ `file:line` citations; 17 cited files verified to exist |
| Rationale provided per answer | ✅ Pass | Every answer has an explicit Rationale subsection |
| No-modify source repository | ✅ Pass | `git diff 542221a38..HEAD -- src/` is **empty**; tree byte-unchanged |
| No-extra-code (only the report) | ✅ Pass | `git diff --name-status` = single `A` line (558 insertions / 0 deletions) |
| Correct filename & location | ✅ Pass | `blitzy/documentation/paperless-ngx_542221a38dff.md` (source-branch name) |
| Cleanup / non-persistence | ✅ Pass | DB documents=0, media=0/0/0, Whoosh=0, `/tmp/paperless` empty; `git status` clean |
| Lint / deterministic pre-commit hooks | ✅ Pass | 0 trailing-ws, 0 tabs, LF-only, balanced fences (19 blocks), 1 EOF LF, no private key |
| Autonomous test suite | ✅ Pass | 481 passed / 0 failed / 2 intentional skips (483 total) |
| Human SME acceptance | ⏳ In Progress | The only outstanding item (Section 2.2, 2.0h) |

**Fixes applied during autonomous validation (3 commits):**

1. `145bd7a38` — addressed code-review findings in the report.
2. `857e5e707` — corrected the 8-core `jobs` value in the OCRmyPDF args (§4.5), making the `jobs` formula worked-example accurate.
3. `07409611c` — refined the OCRmyPDF log-routing caveat (§4.4: `file_paperless` handler bound only to the `paperless` logger; `ocrmypdf.*` propagate to `root → console → qcluster.out`) and normalized the EOF to a single trailing LF.

**Documented decision:** `prettier` was intentionally **not** run — it is a bulk formatter that would reflow the entire 558-line file (tables/lists/spacing), producing a massive diff unrelated to the edits, violating the minimal-change mandate; it is also not auto-wired (binary absent). All actionable deterministic hooks are satisfied.

---

## 6. Risk Assessment

Risks are calibrated to a **read-only documentation deliverable**. Classic build risks (compilation, failing tests, vulnerable dependencies, runtime crashes) do not apply — no code, dependencies, auth surface, or deployed service was introduced.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Documentation accuracy not yet human-accepted | Technical / Quality | Medium | Low | 80+ citations cross-checked vs source + live runtime (zero discrepancies); Answer/Evidence/Rationale/live-output structure; 17 cited files verified | **Open** — closed by SME review (Section 2.2) |
| Findings mis-applied to newer paperless-ngx (Celery / WebP / plugin pipeline) | Integration | Medium | Low | §7 Critical Version Caveat pins findings to commit `542221a38` (Django-Q 1.3.9, Py 3.9, ocrmypdf 13.4.3, PNG) | Mitigated |
| Host-dependent `jobs` value (=11 on a 128-core host) read as universal | Technical | Low | Medium | §4.5 presents the `THREADS_PER_WORKER`/`jobs` **formula** with 4-core (=2) and 8-core (=4) worked examples | Mitigated |
| `pk`-dependent filenames (`0000001.*`) assumed for any DB | Technical | Low | Medium | §5.3 note: the `{pk:07}` rule generalizes to any starting count | Mitigated |
| Citation line-numbers drift if source is later refactored | Technical | Low | Low | Header pins exact commit/branch; citations valid at that HEAD; tree byte-unchanged | Mitigated |
| Stakeholder expects the "inconsistent OCR" root-cause/fix | Operational / Scope | Low | Low-Medium | AAP §0.3.2 + report scope frame this as understand-not-fix; the fix is explicitly out of scope | Accepted (by design) |
| Reproduction requires the (now cleaned-up) transient stack | Operational | Low | Low | §1/§9 give exact bring-up commands; in-repo tests provide static corroboration | Mitigated |
| Example dev credentials (`admin:admin`) appear in snippets | Security | Low | Low | Throwaway creds for a transient, destroyed dev stack; not real secrets; `detect-private-key` hook passed | Accepted (informational) |

**Overall risk posture: LOW.** One Open item (human acceptance) corresponds exactly to the 3h of remaining work; all others are mitigated by the report's own design or accepted by design.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData
    title Project Hours Breakdown (Total 35h)
    "Completed Work" : 32
    "Remaining Work" : 3
```

**Remaining hours by priority (Section 2.2):**

| Priority | Category | Hours |
|---|---|---|
| High | SME technical review & sign-off | 2.0 |
| Medium | Independent reproduction spot-check | 0.5 |
| Low | Distribution & archival | 0.5 |
| | **Total Remaining** | **3.0** |

> **Integrity check:** "Remaining Work" (3) in the pie equals Section 1.2 Remaining Hours (3) and the Section 2.2 total (3.0). "Completed Work" (32) equals Section 1.2 Completed Hours (32). Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a single, comprehensive, evidence-grounded Markdown report that answers all four user questions about the paperless-ngx OCR consume pipeline at commit `542221a38dff`. Every answer pairs **code-as-truth** citations with **live runtime output** captured from a genuinely running stack, plus explicit rationale. The investigation correctly framed the synchronous-accept vs asynchronous-process architecture (Django-Q, not Celery), captured the exact OCRmyPDF kwargs dict (with the host-dependent `jobs` value expressed as a formula), enumerated the 15 persisted database columns versus the computed `@property` paths, and proved cleanup with re-verified post-state.

**Remaining gaps & critical path.** The project is **91.4% complete**. The single remaining gap is **human SME review and acceptance** of the report (the path-to-production gate for a knowledge deliverable). The critical path is: SME reads the report and spot-checks citations (2.0h) → optional independent reproduction (0.5h) → distribution (0.5h). Total remaining: **3.0h**, all human-only.

**Success metrics (all met by autonomous work):** four questions answered ✅; code-as-truth with live corroboration ✅; zero source files modified ✅; full cleanup verified ✅; lint-clean & committed ✅; autonomous suite 481/483 green ✅.

**Production-readiness assessment.** For a documentation deliverable, "production-ready" means accurate, comprehensive, evidence-grounded, lint-clean, and committed — all satisfied. The report is **ready for human acceptance**; no engineering rework is required. Note that *fixing* the user's underlying "inconsistent OCR results" was explicitly out of scope; this report is the diagnostic baseline a follow-up engineering effort would build upon.

| Metric | Value |
|---|---|
| AAP-scoped completion | 91.4% |
| Total / Completed / Remaining hours | 35 / 32 / 3 |
| Source files modified | 0 |
| Deliverables produced | 1 (the report) |
| Autonomous tests passing | 481 / 483 (2 intentional skips) |
| Overall risk posture | Low |

---

## 9. Development Guide

This project produces documentation, so the guide has two tracks: **(A) verify the deliverable** (works on any checkout of this branch) and **(B) reproduce the live investigation** (requires the provided container image). All Track A commands were tested during this assessment.

### 9.1 System Prerequisites

- **To verify the deliverable (Track A):** `git` and any text/Markdown viewer. Optional: `grep`/`python3` (standard) for the lint/structure checks below. No application runtime required.
- **To reproduce the investigation (Track B):** the provided runtime image `andrewparkscaleai/coding-agent:paperless-ngx__…__542221a38dff…` (bundles **Python 3.9**, Tesseract, Ghostscript, qpdf, and Redis). The host used for this assessment runs Python 3.13 with Redis not on PATH, so reproduction **must** use the container image (or an equivalent paperless-ngx dev environment at this commit) to match the pinned dependencies.

### 9.2 Track A — Verify the Deliverable

```bash
# From the repository root of branch blitzy-9488df0b-4a72-499e-a30b-d0cbc564273f

# 1. The report exists (expect 558 lines / 46355 bytes)
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md
wc -c blitzy/documentation/paperless-ngx_542221a38dff.md

# 2. Scope discipline — only the report changed vs the base commit
git diff --name-status 542221a38..HEAD          # expect a single:  A  blitzy/documentation/paperless-ngx_542221a38dff.md
git diff --name-only  542221a38..HEAD -- src/   # expect EMPTY (source tree unchanged)
git status --porcelain                          # expect EMPTY (clean working tree)

# 3. Section index (expect 9 top-level sections in the report)
grep -nE '^## ' blitzy/documentation/paperless-ngx_542221a38dff.md

# 4. Lint cleanliness
F=blitzy/documentation/paperless-ngx_542221a38dff.md
printf 'trailing_ws=%s tabs=%s crlf=%s\n' "$(grep -cE ' +$' "$F")" "$(grep -cP '\t' "$F")" "$(grep -c $'\r' "$F")"
# Fence balance (backtick-free: chr(96)*3 builds the fence marker):
python3 -c "import sys;b=chr(96)*3;n=sum(1 for l in open(sys.argv[1]) if l.lstrip().startswith(b));print('fence_lines=%d -> %s'%(n,'BALANCED' if n%2==0 else 'UNBALANCED'))" "$F"

# 5. Jump straight to a key finding (the OCRmyPDF args)
grep -n "Calling OCRmyPDF with args" "$F"
```

Expected results (verified): single `A` line; empty `src/` diff; clean tree; 9 report sections; `trailing_ws=0 tabs=0 crlf=0`; `fence_lines=38 -> BALANCED`.

### 9.3 Track B — Reproduce the Live Investigation

Run **inside the provided container image** at commit `542221a38dff`:

```bash
# 1. Start the Redis broker (hard prerequisite for async processing)
redis-server --daemonize yes
redis-cli ping                       # expect: PONG

# 2. From src/: migrate and ensure the admin user
cd src
python3 manage.py migrate --no-input
python3 manage.py manage_superuser   # PAPERLESS_ADMIN_USER/PASSWORD -> admin/admin

# 3. Start the Django-Q worker (executes consume_file, emits the Q2 logs)
python3 manage.py qcluster &         # logs -> data/log/qcluster.out + data/log/paperless.log

# 4. Start the ASGI web server (accepts the upload, returns the Q1 response)
gunicorn -c gunicorn.conf.py paperless.asgi:application &   # :8000

# 5. Create a forced-OCR fixture: a multi-page IMAGE-ONLY PDF (no embedded text)
#    (generate with Pillow under /tmp; verify extracted-text length == 0 before upload)

# 6. Upload via the standard endpoint (Q1)
curl -sS -u admin:admin -F "document=@/tmp/img_only.pdf" \
     http://localhost:8000/api/documents/post_document/ -D -
#    expect:  HTTP/1.1 200 OK  ...  "OK"

# 7. Observe (Q2/Q3/Q4)
tail -n 40 data/log/paperless.log                       # ordered stage lines + OCRmyPDF args
ls -la media/documents/originals media/documents/archive media/documents/thumbnails
python3 manage.py shell -c "from documents.models import Document; d=Document.objects.get(pk=1); print(d.filename, d.archive_filename, d.checksum, d.mime_type)"

# 8. Tests (corroboration) — run as a non-root user
CI=true python3 -m pytest -p no:cacheprovider --no-cov -n 8     # expect 481 passed / 2 skipped
```

### 9.4 Cleanup (mandatory — non-persistence)

```bash
# Targeted delete (never .all().delete() on a non-pristine DB)
python3 manage.py shell -c "from documents.models import Document; Document.objects.filter(pk=1).delete()"
# Sweep any media residue, clear the Whoosh index, and empty /tmp/paperless
rm -f media/documents/originals/0000001.pdf media/documents/archive/0000001.pdf media/documents/thumbnails/0000001.png
rm -rf /tmp/paperless/*
# Verify
python3 manage.py shell -c "from documents.models import Document; print('documents=', Document.objects.count())"   # expect 0
git status --porcelain          # expect EMPTY (no source change)
```

### 9.5 Troubleshooting

- **No Q2 logs / document never processes** → Redis is not running. Start `redis-server`; confirm `redis-cli ping` → `PONG`. The web upload still returns `200`, but the async worker never receives the task.
- **`jobs` value differs from `11`** → expected and correct. `jobs` is host-CPU-dependent: `jobs = max(floor(cpu_count / TASK_WORKERS), 1)` (see §4.5 of the report). On 4-core → `2`; on 8-core → `4`.
- **OCR is skipped (no archive text)** → the input was a born-digital PDF. Under the default `OCR_MODE=skip` (`skip_text=True`), pages with an existing text layer are skipped. Use an **image-only** PDF to force OCR.
- **`pyzbar` / `libzbar0` ImportError at `tasks.py:25`** → install `libzbar0` in the runtime environment (required for the top-level `pyzbar` import even though barcode consumption is off by default).
- **ImageMagick `convert` fails on PDF** → the distro's ImageMagick `policy.xml` blocks PDF; relax the PDF policy to allow the thumbnail render.
- **Pinned-dependency mismatch** → the host Python (3.13 here) will not match this commit's pins. Use the provided **Python 3.9** container image.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Verify only the report changed | `git diff --name-status 542221a38..HEAD` |
| Verify source unchanged | `git diff --name-only 542221a38..HEAD -- src/` |
| Verify clean tree | `git status --porcelain` |
| Report section index | `grep -nE '^## ' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Doc commit history | `git log --oneline 542221a38..HEAD -- blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Start broker | `redis-server --daemonize yes` ; `redis-cli ping` |
| Migrate / admin | `python3 manage.py migrate --no-input` ; `python3 manage.py manage_superuser` |
| Start worker / web | `python3 manage.py qcluster &` ; `gunicorn -c gunicorn.conf.py paperless.asgi:application &` |
| Upload | `curl -sS -u admin:admin -F "document=@/tmp/img_only.pdf" http://localhost:8000/api/documents/post_document/ -D -` |
| Run tests | `CI=true python3 -m pytest -p no:cacheprovider --no-cov -n 8` |

### Appendix B — Port Reference

| Port | Service | Notes |
|---|---|---|
| `8000` | gunicorn ASGI (`paperless.asgi:application`) | Accepts the upload; returns the Q1 `200 "OK"` |
| `6379` | Redis | Broker/channel layer; carries the task web → worker |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (558 lines) |
| `src/paperless/urls.py` | Q1 — upload route registration |
| `src/documents/views.py` | Q1 — `PostDocumentView` returns `Response("OK")` |
| `src/documents/serialisers.py` | Q1 — MIME validation (`400` path) |
| `src/documents/tasks.py` | Q2 — `consume_file` Django-Q task |
| `src/documents/consumer.py` | Q2/Q3/Q4 — stage logs, file moves, DB write |
| `src/documents/loggers.py` | Q2 — logger naming/grouping |
| `src/paperless_tesseract/parsers.py` | Q2 — `construct_ocrmypdf_parameters` + the args log |
| `src/paperless_tesseract/signals.py` | Q3 — `application/pdf` → `.pdf` mapping |
| `src/documents/file_handling.py` | Q3 — `generate_filename` default `{pk:07}` |
| `src/documents/parsers.py` | Q2/Q3 — temp dir, PDF→PNG thumbnail, optipng |
| `src/documents/models.py` | Q3/Q4 — columns + computed path properties |
| `src/documents/signals/handlers.py` | Q4 — post-consume tag/correspondent/type/index |
| `src/documents/apps.py` | Q4 — connects post-consume signals |
| `src/paperless/settings.py` | Q2/Q3 — OCR defaults, media layout, log format |
| `media/documents/{originals,archive,thumbnails}/` | Q3 — generated artifacts (transient) |

### Appendix D — Technology Versions

| Component | Version | Relevance |
|---|---|---|
| Python | 3.9 | Runtime for this commit (`Dockerfile`) |
| Django | 4.0.4 | Web framework + ORM (`Document` model) |
| djangorestframework | 3.13.1 | `PostDocumentView` / `Response` (Q1) |
| django-q | 1.3.9 | Async `qcluster` worker (Q2) |
| ocrmypdf | 13.4.3 | OCR engine wrapper; the args dict (Q2) |
| pikepdf | 5.1.1 | PDF metadata handling |
| pdf2image | 1.16.0 | PDF → image rasterisation |
| Pillow | 9.1.x | Image/thumbnail handling (Q3) |
| redis (client) | 3.5.3 | Broker/channel client |
| channels / channels-redis | 3.0.4 / 3.4.0 | WebSocket progress broadcasts |
| python-magic | 0.4.25 | MIME detection (Q1/Q2) |
| whoosh | 2.7.4 | Full-text index (cleanup target) |
| scikit-learn | 1.0.2 | Post-consume classifier (Q4 context) |

> Environment note: the live host observed `qpdf 10.1.0` / `pngquant 2.12.2` (the AAP §0.4 listed `10.6.3` / `2.13.1`); the deliverable does not pin these tools, so no correction was needed.

### Appendix E — Environment Variable Reference

| Variable | Default | Effect on the answers |
|---|---|---|
| `PAPERLESS_ADMIN_USER` / `_PASSWORD` / `_MAIL` | (unset) → `admin/admin` here | Creates the superuser for the authenticated upload (O0) |
| `PAPERLESS_OCR_MODE` | `skip` | `skip_text=True` in the OCR args (Q2) |
| `PAPERLESS_OCR_LANGUAGE` | `eng` | `language='eng'` (Q2) |
| `PAPERLESS_OCR_OUTPUT_TYPE` | `pdfa` | `output_type='pdfa'` (Q2) |
| `PAPERLESS_OCR_CLEAN` | `clean` | `clean=True` (Q2) |
| `PAPERLESS_OCR_PAGES` | `0` | `sidecar` key present (vs `pages`) (Q2) |
| `PAPERLESS_FILENAME_FORMAT` | (unset) | Falls back to `{pk:07}` naming (Q3) |
| `PAPERLESS_MEDIA_ROOT` | `../media` | Root of originals/archive/thumbnails (Q3) |
| `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | `false` | Whether a duplicate's temp upload is unlinked (cleanup nuance, §8) |

### Appendix F — Developer Tools Guide

| Tool / One-liner | Use |
|---|---|
| `python3 -c "b=chr(96)*3;print(sum(1 for l in open('FILE') if l.lstrip().startswith(b)))"` | Confirm fenced-code-block balance (even = balanced) |
| `grep -cE ' +$' <file>` / `grep -cP '\t' <file>` | Detect trailing whitespace / tab characters |
| `git diff 542221a38..HEAD --stat` | One-line summary of all changes vs base |
| `python3 manage.py shell -c "…"` | Inspect the `Document` row / counts (Q4, cleanup) |
| `redis-cli ping` | Confirm the broker is up before expecting async logs |

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the primary directive defining this project's scope |
| **ASGI** | Async Server Gateway Interface; gunicorn serves `paperless.asgi:application` |
| **Django-Q / `qcluster`** | The asynchronous task queue/worker (this commit; *not* Celery) that runs `consume_file` |
| **Consume pipeline** | The end-to-end flow: validate → parse/OCR → thumbnail → store → index |
| **OCRmyPDF** | The OCR engine wrapper invoked via `ocrmypdf.ocr(**args)`; its kwargs dict is the focus of Q2 |
| **Sidecar** | The plain-text OCR output file OCRmyPDF writes alongside the archive PDF |
| **Archive PDF** | The OCR'd PDF/A produced from the original (Q3) |
| **`@property` (computed)** | A Python-derived value (e.g., `source_path`, `thumbnail_path`) that is **not** a DB column (Q4) |
| **Whoosh** | The on-disk full-text search index updated post-consume (a cleanup target) |
| **SME** | Subject-Matter Expert — the human reviewer who performs final acceptance |
