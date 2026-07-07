# Blitzy Project Guide
## paperless-ngx — Runtime-Grounded OCR Behavior Investigation (Q&A)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, runtime-grounded documentation artifact that answers a user's practical question about **how OCR behaves at runtime in paperless-ngx**, focusing on the states and signals that are difficult to observe from the outside. The deliverable — `blitzy/documentation/paperless-ngx_542221a38dff.md` — was produced by *building, running, and observing* paperless-ngx in its canonical default configuration and capturing real output, per the SWE-AtlasQnA-Repo methodology. It answers four questions: (Q1) how OCR start and live processing state appear for an image with no text; (Q2) whether a text-bearing image/PDF skips or still touches the OCR pipeline; (Q3) which API fields reflect OCR-generated versus pre-existing text; and (Q4) what happens to a document's final state on weak/incomplete OCR. The scope is strictly read-only and additive: exactly one new file, with no source or dependency changes.

### 1.2 Completion Status

The project is **92.6% complete** on an AAP-scoped, hours-based basis. All autonomous investigation and authoring work is delivered and validated; the remaining 6 hours is acceptance-side path-to-production for a documentation artifact (human review, merge, optional enhancements).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData title Completion Status — 92.6% Complete
    "Completed Work (AI)" : 75
    "Remaining Work" : 6
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | 81 |
| **Completed Hours (AI + Manual)** | 75 |
| — of which AI (Blitzy autonomous) | 75 |
| — of which Manual (human) | 0 |
| **Remaining Hours** | 6 |
| **Percent Complete** | **92.6%** |

> **Calculation (PA1, AAP-scoped):** Completion % = Completed ÷ (Completed + Remaining) = 75 ÷ (75 + 6) = 75 ÷ 81 = **92.6%**.

### 1.3 Key Accomplishments

- ✅ **The single AAP deliverable is complete and committed** — `blitzy/documentation/paperless-ngx_542221a38dff.md` (1,179 lines, ~11,235 words, 54 captured-output code blocks).
- ✅ **All four questions (Q1–Q4) answered with runtime-observed evidence** — 56 `[observed-at-runtime]` claims, each with its exact command and complete, unedited output.
- ✅ **Canonical async stack stood up and driven end-to-end** — redis broker, django-q `qcluster`, and daphne ASGI server, with **11 documents + 1 duplicate** consumed live across all scenarios.
- ✅ **Fully grounded** — 37 unique `file:line` citations across 10 source files, each naming the implementing function/method; independently spot-checked and confirmed accurate.
- ✅ **Honest discrepancy disclosure** — the widely-assumed interpolated `WORKING 20→70` progress band is proven (via full-tree `grep` + a multi-page-PDF run) to be **never emitted** for OCR documents, and is surfaced up-front rather than glossed over.
- ✅ **Byte-sensitivity honored** — `checksum` / `archive_checksum` recomputed from on-disk bytes via two independent tools (`hashlib.md5` + OS `md5sum`), all `MATCH: True`.
- ✅ **Read-only constraint perfectly honored** — zero source changes, zero dependency changes, no temporary scripts left in the tree; net diff versus upstream base is exactly one added file.
- ✅ **OCRmyPDF semantics corroborated** against official documentation (skip/redo/force text handling + sidecar behavior), version-bracketed and honestly disclosed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None_ — the Final Validator reports zero unresolved errors and all five production-readiness gates passed. | No release blocker. The deliverable is accurate, internally consistent to the byte/millisecond, lint-safe, and committed. | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| OCRmyPDF official docs — exact `v13.4.3` page | Read (public web) | The exact-version documentation page could not be fetched during corroboration; semantics were validated via version-bracketing (v12.0.1 below / v15.x above, confirmed identical) and disclosed in the deliverable. | Mitigated / Disclosed (non-blocking) | Reviewer (optional) |

> All other systems (repository, canonical container, redis, django-q, ASGI) were fully accessible; **no access issue blocks build validation, integration, or delivery.**

### 1.6 Recommended Next Steps

1. **[High]** Have a paperless-ngx-knowledgeable engineer perform an SME technical review and sign-off of the deliverable (verify the four answers, spot-check citations and captured outputs, confirm the honest discrepancy disclosures). — *3.0h*
2. **[High]** Accept the PR and merge `blitzy/documentation/paperless-ngx_542221a38dff.md` to the target branch. — *0.5h*
3. **[Low]** Publish/link the answer from the internal knowledge base or docs portal for discoverability. — *1.0h*
4. **[Low]** *(Optional)* Exercise the `redo` OCR mode explicitly to round out the full mode matrix (skip / skip_noarchive / force are already demonstrated). — *1.5h*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work traces to an AAP requirement (the runtime investigation, the four answers, the methodological rules, and the read-only prerequisites). Total = **75 hours**.

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical environment bring-up & async stack orchestration | 6 | Standing up redis-server, running migrations, launching django-q `qcluster` and daphne ASGI; verifying the full OCR toolchain and confirming the default config (`OCR_MODE="skip"`, `OCR_OUTPUT_TYPE="pdfa"`, `OCR_LANGUAGE="eng"`). |
| Observation tooling authoring | 8 | Bespoke helpers written outside the tracked tree: authenticated WebSocket listener, session minter, upload driver, ORM reader, checksum verifier, and input generators (no-text image, text image, text-layer PDF, near-blank image, multi-page PDFs, boundary PDFs at 29/74 chars). |
| Q1 investigation & capture | 10 | 7 sub-sections: WebSocket before/during/after sequence; the no-DB-state-field proof; worker/OCR-invocation log capture; the true active-OCR signal (live `tesseract`/`unpaper`/`gs` subprocess tree in the 20→70 window); the 20→70-never-emitted discovery with full-tree `grep` proof + multi-page-PDF run; the FAILED duplicate path; run stability. |
| Q2 investigation & capture | 8 | Text-bearing image run; text-layer PDF run showing OCRmyPDF is still invoked (sidecar-discard → pdfminer); non-default `skip_noarchive` skip-entirely run; the 50-character boundary crafted and demonstrated at 29 and 74 chars. |
| Q3 investigation & capture | 6 | Side-by-side `GET /api/documents/{id}/` responses; field-by-field comparison of the 12 serializer fields; byte-exact `checksum`/`archive_checksum` recompute via two independent tools; the `skip_noarchive` `archived_file_name: null` REST tell. |
| Q4 investigation & capture | 6 | Near-blank image run; full fallback chain trace (`NoTextFoundException` → force-OCR retry → `self.text=""`); terminal `SUCCESS` with empty-but-otherwise-populated metadata; django-q `success=True`; run-to-run stability across two runs. |
| Web-search corroboration | 2 | Validating OCRmyPDF `--skip-text`/`--redo-ocr`/`--force-ocr` + sidecar semantics against official documentation; version-bracketing with honest disclosure. |
| Deliverable authoring | 14 | Writing the 1,179-line / ~11,235-word grounded markdown (TL;DR, Environment, Q1–Q4, Web corroboration, Coverage appendix, Read-only verification), embedding all captured output, applying `[observed]`/`[inferred]` provenance tags, and enforcing single-coherent-session internal consistency. |
| Citation grounding audit | 4 | Verifying all 37 unique `file:line` citations across 10 source files against source and naming the implementing functions/methods. |
| Internal-consistency verification | 3 | Confirming byte/millisecond agreement of every timestamp, `task_id`, `document_id`, checksum, and character count across all sections. |
| Code-review + QA + final-validation fix cycle | 6 | The major post-review revision (commit `76bf71fe7`, +859/-451), the pdfminer citation fix (`953ee1b50`), and the Final Validator's force-fallback log-quote fix (`785fec82c`). |
| Read-only cleanup & verification | 2 | Keeping all helpers outside the tracked tree, removing temporary artifacts, and documenting the git status/diff read-only proofs. |
| **Total** | **75** | **Matches Completed Hours in Section 1.2.** |

### 2.2 Remaining Work Detail

All remaining work is acceptance-side path-to-production for a documentation artifact. There are **no code defects and nothing to deploy**. Total = **6 hours**.

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review & sign-off of the deliverable | 3.0 | High |
| Stakeholder acceptance & PR merge | 0.5 | High |
| Publish/distribute to internal knowledge base / docs portal | 1.0 | Low |
| *(Optional)* Exercise `redo` OCR mode explicitly for extra variant completeness | 1.5 | Low |
| **Total** | **6.0** | **Matches Remaining Hours in Section 1.2 and Section 7.** |

### 2.3 Total Project Hours & Reconciliation

| Line | Hours |
|------|-------|
| Section 2.1 — Completed Work total | 75 |
| Section 2.2 — Remaining Work total | 6 |
| **Total Project Hours (2.1 + 2.2)** | **81** |
| Section 1.2 — Total Hours | 81 ✔ |
| Section 7 — Pie "Completed Work" / "Remaining Work" | 75 / 6 ✔ |

> **Integrity:** 75 (completed) + 6 (remaining) = 81 (total); 75 ÷ 81 = 92.6%. These figures are identical in Sections 1.2, 2.1, 2.2, 2.3, and 7.

---

## 3. Test Results

> **Integrity note.** All items below originate from **Blitzy's autonomous validation logs** for this project. No source-code unit-test suite was in scope — the AAP explicitly excludes test additions and all `src/` code is out of scope. For this read-only documentation deliverable, the **equivalent-of-tests** is the autonomous **runtime reproduction of every documented claim** plus the **citation / consistency / byte-sensitivity audits** enumerated here. "Coverage %" therefore refers to coverage of the question-items and documented claims, not code line coverage (which is N/A — no code was authored).

| Test / Validation Category | Framework / Harness | Total | Passed | Failed | Coverage % | Notes |
|----------------------------|---------------------|-------|--------|--------|-----------|-------|
| Runtime scenario reproduction (SUCCESS paths) | Blitzy autonomous runtime harness (live paperless-ngx stack) | 11 | 11 | 0 | 100% of Q1–Q4 scenarios | 11 documents consumed via the real `POST /api/documents/post_document/` entry point; every WebSocket sequence and terminal state reproduced exactly. |
| Runtime failure-path reproduction | Blitzy autonomous runtime harness | 1 | 1 | 0 | Duplicate-detection path | 1 duplicate correctly reached terminal `FAILED` — the **expected** outcome, so the failure-path test passes. |
| Citation grounding audit | Source cross-reference (`file:line`) | 37 | 37 | 0 | 100% unique citations | All 37 unique citations + all bare `:NNN` references verified against source; independently spot-checked (models.py:88, urls.py:137, parsers.py:234–244, settings.py:522). |
| Byte-sensitivity checksum verification | `hashlib.md5` (ORM) + OS `md5sum` | 2 | 2 | 0 | 100% MATCH | `checksum` / `archive_checksum` recomputed from on-disk bytes via two independent tools; all `MATCH: True`. |
| Internal-consistency audit | Byte/millisecond cross-section check | All sections | Pass | 0 | 100% | Every timestamp, `task_id`, `document_id`, checksum, and character count agrees across all sections (single coherent session). |
| Production-readiness gates | Blitzy Final Validator | 5 | 5 | 0 | 100% | All five gates passed; the one defect found (Finding #1) fixed and byte-verified. |
| **Totals** | — | **57** | **57** | **0** | **100%** | Every documented behavioral claim reproduced or verified. |

---

## 4. Runtime Validation & UI Verification

**Runtime health (canonical async stack, live in the canonical container):**

- ✅ **Operational** — redis broker (`redis-cli ping` → `PONG`); serves both the django-q broker and the Channels layer.
- ✅ **Operational** — django-q `qcluster` worker cluster (this is django-q, **not** Celery); executed `consume_file` and consumed 11 documents + processed 1 duplicate.
- ✅ **Operational** — daphne ASGI server; delivered the live `ws/status/` WebSocket payload sequence.
- ✅ **Operational** — OCR toolchain subprocesses (`ocrmypdf` → `tesseract` / `unpaper` / `ghostscript`) observed running live during the 20→70 window.

**API integration outcomes:**

- ✅ **Operational** — `POST /api/documents/post_document/` (DRF token auth) enqueues `consume_file`.
- ✅ **Operational** — `GET /api/documents/{id}/` returns the 12-field serialized response used for the Q3 comparison.
- ✅ **Operational** — WebSocket `ws/status/` (session-cookie auth) streams `STARTING → WORKING (20/70/90/95) → SUCCESS/FAILED` with `document_id` null until the terminal `SUCCESS`.

**UI verification:**

- ⚠ **Not in scope / not exercised** — no Angular front-end (`src-ui`) code was run or changed; this is a read-only backend runtime investigation conducted through the REST API and WebSocket surfaces. The `ws/status/` stream that backs the UI's live progress indicator **was** verified operational at the protocol level, so the data source the UI consumes is confirmed working.

---

## 5. Compliance & Quality Review

The deliverable is governed by the user-supplied **SWE-AtlasQnA-Repo** rule set. Each rule is cross-mapped to the quality benchmark and its verified status.

| Rule / Benchmark | Requirement | Status | Evidence |
|------------------|-------------|--------|----------|
| Deliverable rule | One markdown file named after the source branch in `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/paperless-ngx_542221a38dff.md` exists and is committed |
| Run-first rule | Build/run/observe before writing | ✅ Pass | 56 `[observed-at-runtime]` claims; investigation-first git history |
| Real-entry-point rule | Exercise the exact code path via its real entry point | ✅ Pass | `POST /api/documents/post_document/` (`views.py:491`) + `qcluster` |
| Canonical-configuration rule | Default config, exact commands stated | ✅ Pass | `OCR_MODE="skip"` read from live settings; full version + command capture |
| Every-condition rule | Enumerate/exercise every variant, incl. error/edge | ✅ Pass | 11-doc run matrix: image / text-layer PDF / near-blank / boundary (29 & 74) / `skip_noarchive` / FAILED duplicate |
| Before/during/after rule | Observe state transitions over time | ✅ Pass | §1a ordered WebSocket sequence; §1d during-OCR subprocess capture |
| Actual-output rule | Complete, unedited output + command per claim | ✅ Pass | 54 captured-output code blocks, each with its command |
| Byte-sensitivity rule | Verify checksums against exact emitted bytes | ✅ Pass | §3c byte-exact recompute via two independent tools, `MATCH: True` |
| Grounding rule | `file:line` + named function/method | ✅ Pass | 37 unique citations / 10 files; named methods |
| Web-search rule | Corroborate OCRmyPDF option semantics | ✅ Pass | Web corroboration section; version-bracketed + disclosed |
| Magnitude / run-to-run consistency | Confirm values stable across ≥2 runs | ✅ Pass | §1g and §4e stability across two runs |
| Fidelity reconciliation | Reconcile "image that already has text" vs paperless reality | ✅ Pass | Q2 reconciles raster image vs text-layer PDF explicitly |
| Report-what-you-observe rule | Lead with direct answer; disclose discrepancies honestly | ✅ Pass | 20→70 correction surfaced up-front in TL;DR + §1e with grep proof |
| Completeness rule | Coverage pass over every named item | ✅ Pass | Coverage & verification appendix with question-item table |
| Scope (read-only) rule | No source/dependency changes; temp scripts removed | ✅ Pass | Net diff = one added file; `src/` diff empty; clean tree |

**Fixes applied during autonomous validation:** the Final Validator identified one defect (Finding #1) — the Q4 §4a force-fallback log message was quoted without its real f-string prefix and cited an imprecise line range. It was corrected to the full line with precise citations (`parsers.py:266-267` raise, `:276` except, `:277-281` log) and re-verified byte-exact (commit `785fec82c`).

**Outstanding compliance items:** None.

---

## 6. Risk Assessment

Overall risk posture is **Low**, consistent with a read-only, additive, fully-validated documentation deliverable.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation drift — `file:line` values are pinned to HEAD `542221a38`; future source edits could shift line numbers | Technical | Low | Medium | Deliverable explicitly stamps branch, HEAD, and dependency versions; treat as a point-in-time snapshot | Mitigated |
| `20→70` progress-band nuance — naive reliance on interpolated OCR progress payloads would mislead | Technical | Low | Low | Corrected up-front in TL;DR + §1e with full-tree `grep` proof | Resolved / Documented |
| Redacted local DRF token shown in commands | Security | Low | Low | Token is a throwaway minted locally (`drf_create_token`), redacted as `<TOKEN>`, not a production secret; no source/deps changed → zero new attack surface | Mitigated |
| Reproduction requires the canonical container (Python 3.9 + OCR toolchain); env-derived values (`jobs:`, DPI) vary by host | Operational | Low | Medium | All versions and exact commands pinned; env-derived values disclosed as non-constant | Mitigated |
| Exact OCRmyPDF `v13.4.3` doc page unavailable for corroboration | Integration | Low | Low | Semantics validated via version-bracketing (v12.0.1 / v15.x, confirmed identical) and honestly disclosed | Mitigated / Disclosed |

> Not applicable to this deliverable: monitoring/health-check/backup gaps (nothing is deployed), external-service credentials, and network configuration (no integrations introduced).

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 75
    "Remaining Work" : 6
```

**Remaining hours by category (from Section 2.2):**

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review & sign-off | 3.0 | High |
| Stakeholder acceptance & PR merge | 0.5 | High |
| Publish/distribute to knowledge base | 1.0 | Low |
| *(Optional)* `redo`-mode variant | 1.5 | Low |
| **Total Remaining** | **6.0** | — |

> **Integrity:** the "Remaining Work" pie value (6) equals the Section 1.2 Remaining Hours (6) and the Section 2.2 Hours total (6). The "Completed Work" pie value (75) equals the Section 1.2 Completed Hours (75) and the Section 2.1 total (75). Colors: Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.6% complete**. The sole AAP deliverable — a runtime-grounded, question-and-answer investigation of OCR behavior in paperless-ngx — is complete, committed, and validated. Every one of the four questions is answered from *observed* runtime evidence (not code reading alone): 56 `[observed-at-runtime]` claims, each with its exact command and complete output, and 37 unique `file:line` citations across 10 source files. The Blitzy Final Validator reproduced every claim exactly in the canonical container across 11 consumed documents plus one expected-FAILED duplicate, verified all citations, and confirmed byte/millisecond internal consistency. Notably, the deliverable **corrects a common misconception** — that an interpolated `WORKING 20→70` progress band is emitted for OCR documents — proving with a source-tree `grep` and a multi-page-PDF run that it is never emitted.

**Remaining gaps and critical path to production.** Because this is a read-only, additive documentation artifact, there is **no application code to deploy** and **no code defect to fix**. The remaining 6 hours is purely acceptance-side: an SME technical review & sign-off (3.0h) is the one meaningful gate, followed by stakeholder acceptance & merge (0.5h). Two optional enhancements — publishing to a knowledge base (1.0h) and exercising the `redo` OCR mode explicitly for completeness (1.5h) — round out the remainder but are not required for delivery.

**Success metrics.** (1) All four questions answered with observed evidence — met. (2) Read-only constraint honored — met (net diff = exactly one added file; `src/` diff empty; clean tree). (3) Every claim reproducible in the canonical environment — met (Final Validator: 100% reproduction). (4) Grounded to `file:line` with named functions — met (37 unique citations, spot-checked).

**Production-readiness assessment.** The deliverable is **production-ready pending human sign-off**. It is accurate, internally consistent to the byte and millisecond, lint-safe, and committed with zero unresolved errors. The recommended path is: SME review → merge → (optional) publish. No blockers exist.

---

## 9. Development Guide

This guide covers two paths: **(A) Reviewer** — open, review, and verify the deliverable and confirm the read-only constraint (primary for the remaining acceptance work); and **(B) Reproducer** — stand up the canonical async stack and re-run the Q1–Q4 scenarios to independently confirm any documented claim.

### 9.1 System Prerequisites

- **Reviewer path:** `git` (≥ 2.x) and any text/markdown viewer. No build or runtime is required to read and review the document.
- **Reproducer path:** the canonical Docker image (Python **3.9-slim-bullseye** base) providing the full OCR toolchain — `ocrmypdf 13.4.3`, `tesseract 4.1.1` (+ `eng`/`deu`/`fra`/`ita`/`spa`), `ghostscript 9.53.3`, `unpaper 6.1`, `qpdf 10.1.0`, `pngquant`, `redis 6.0.16`. Recommended: ≥ 2 CPU / 4 GB RAM for OCR subprocesses.

### 9.2 Environment Setup (Reproducer path)

Runtime directories and the broker live **outside** the repository tree so no runtime artifact touches it. `OCR_MODE` is intentionally left **unset** so the canonical default (`skip`) applies.

```bash
# Point runtime dirs at /tmp so nothing touches the repo tree
export PAPERLESS_DATA_DIR=/tmp/pl/data
export PAPERLESS_MEDIA_ROOT=/tmp/pl/media
export PAPERLESS_CONSUMPTION_DIR=/tmp/pl/consume
export PAPERLESS_REDIS=redis://localhost:6379
mkdir -p "$PAPERLESS_DATA_DIR" "$PAPERLESS_MEDIA_ROOT" "$PAPERLESS_CONSUMPTION_DIR"
# OCR_MODE intentionally UNSET => default "skip"
```

### 9.3 Dependency Installation

Dependencies are pinned in `requirements.txt` and pre-installed in the canonical image (no changes were made). Confirm the key OCR-path pins:

```bash
grep -iE '^(ocrmypdf|Django|djangorestframework|django-q|channels|channels-redis|redis|pikepdf|img2pdf|pdf2image|python-magic|scikit-learn)==' requirements.txt
```

Expected (verified): `ocrmypdf==13.4.3`, `Django==4.0.4`, `djangorestframework==3.13.1`, `django-q==1.3.9`, `channels==3.0.4`, `channels-redis==3.4.0`, `redis==3.5.3`, `pikepdf==5.1.1`, `img2pdf==0.4.4`, `pdf2image==1.16.0`, `python-magic==0.4.25`, `scikit-learn==1.0.2`.

### 9.4 Application Startup Sequence (Reproducer path)

Start services in this order — the live "processing state" only exists when redis, the worker, and the ASGI server are all running:

```bash
redis-server --daemonize yes                                   # broker + channel layer; redis-cli ping => PONG
cd src && python manage.py migrate                             # sqlite at $PAPERLESS_DATA_DIR/db.sqlite3
python manage.py drf_create_token admin                        # prints a REST token (redact as <TOKEN>)
python manage.py qcluster > /tmp/pl/qcluster.log 2>&1 &        # django-q worker cluster (NOT Celery)
daphne -b 127.0.0.1 -p 8000 paperless.asgi:application &       # ASGI server for REST + WebSocket
```

Ports: **redis 6379**, **ASGI/daphne 8000**.

### 9.5 Verification Steps

```bash
redis-cli ping                                                 # => PONG
sed -n '1,16p' /tmp/pl/qcluster.log                            # => django-q cluster banner (worker started)
curl -sI http://127.0.0.1:8000/api/                            # => HTTP 200/401 (server responding)
```

### 9.6 Example Usage (Reproducer path)

```bash
# Upload an image with no embedded text (Q1) via the real entry point
curl -s -H "Authorization: Token <TOKEN>" \
     -F "document=@no_text.png" \
     http://127.0.0.1:8000/api/documents/post_document/

# Subscribe to the live status stream (session-cookie auth) to watch:
#   STARTING 0 -> WORKING 20 (parsing_document) -> 70 (generating_thumbnail) -> 95 -> SUCCESS 100
#   ws://127.0.0.1:8000/ws/status/

# After processing, inspect the durable result (Q3/Q4)
curl -s -H "Authorization: Token <TOKEN>" \
     http://127.0.0.1:8000/api/documents/1/ | python -m json.tool
```

### 9.7 Reviewer Path — Open & Verify the Deliverable

```bash
# Open the deliverable and jump to any answer section by heading
sed -n '1,40p' blitzy/documentation/paperless-ngx_542221a38dff.md
grep -nE '^## (Q1|Q2|Q3|Q4)' blitzy/documentation/paperless-ngx_542221a38dff.md

# Confirm the read-only constraint (net effect vs the upstream base commit)
git diff --name-status 542221a38dff HEAD          # => exactly: A blitzy/documentation/paperless-ngx_542221a38dff.md
git diff --name-status 542221a38dff HEAD -- src/  # => empty (zero source changes)
git status --porcelain                            # => empty (clean working tree)
```

### 9.8 Troubleshooting

- **`qcluster` isn't consuming uploads** → confirm `redis-cli ping` returns `PONG` and `PAPERLESS_REDIS` points at the running broker; check `/tmp/pl/qcluster.log`.
- **WebSocket `ws/status/` stream is empty** → ensure you launched **daphne (ASGI)**, not gunicorn (WSGI); the stream uses session-cookie auth (log in first), while REST uses token auth.
- **OCR appears "skipped" unexpectedly** → in the default `skip` mode OCRmyPDF is invoked for **every** document; a document is only skipped entirely under the non-default `skip_noarchive` mode with >50 embedded characters.
- **`jobs:` / DPI values differ from the document** → these are environment-derived (CPU count, image geometry) and are expected to vary by host; the deliverable discloses them as non-constant.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---------|---------|
| `redis-server --daemonize yes` | Start the broker + channel layer |
| `python manage.py migrate` | Initialize the SQLite database |
| `python manage.py drf_create_token admin` | Mint a DRF token for REST uploads |
| `python manage.py qcluster` | Start the django-q worker cluster |
| `daphne -b 127.0.0.1 -p 8000 paperless.asgi:application` | Start the ASGI server (REST + WebSocket) |
| `curl -F "document=@FILE" .../api/documents/post_document/` | Upload via the real entry point |
| `curl .../api/documents/{id}/` | Inspect the durable serialized result |
| `git diff --name-status 542221a38dff HEAD` | Verify the read-only net diff |

### Appendix B — Port Reference

| Service | Port |
|---------|------|
| redis (broker + channel layer) | 6379 |
| daphne / ASGI (REST + WebSocket) | 8000 |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (only added file) |
| `src/paperless_tesseract/parsers.py` | OCR engine wrapper — modes, sidecar, fallback, empty-text (Q2/Q3/Q4) |
| `src/documents/consumer.py` | Ingestion pipeline — WebSocket progress + atomic persistence (Q1/Q3/Q4) |
| `src/documents/tasks.py` | `consume_file` django-q background task (Q1) |
| `src/paperless/consumers.py` | `StatusConsumer` WebSocket relay (Q1) |
| `src/paperless/urls.py` (`:137`) | `ws/status/` route binding |
| `src/paperless/settings.py` (`:522`) | `OCR_MODE` default `"skip"` + `Q_CLUSTER` config |
| `src/documents/models.py` (`:88`) | `Document` model — no status field (Q1/Q4) |
| `src/documents/serialisers.py` (`:222-234`) | 12-field REST contract (Q3) |
| `src/documents/views.py` (`:491`) | `POST /api/documents/post_document/` entry point |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.9.23 (base `python:3.9-slim-bullseye`) | Dockerfile |
| ocrmypdf | 13.4.3 | requirements.txt |
| Django | 4.0.4 | requirements.txt |
| djangorestframework | 3.13.1 | requirements.txt |
| django-q | 1.3.9 (background worker — not Celery) | requirements.txt |
| channels / channels-redis | 3.0.4 / 3.4.0 | requirements.txt |
| redis (py) / redis (server) | 3.5.3 / 6.0.16 | requirements.txt / runtime |
| tesseract | 4.1.1 | Dockerfile (apt) |
| ghostscript | 9.53.3 | Dockerfile (apt) |
| unpaper | 6.1 | Dockerfile (apt) |
| qpdf | 10.1.0 | source build |

### Appendix E — Environment Variable Reference

| Variable | Default | Purpose |
|----------|---------|---------|
| `PAPERLESS_OCR_MODE` | `skip` | OCR mode; unset → canonical default `skip` (OCRmyPDF invoked with `skip_text=True`) |
| `PAPERLESS_OCR_OUTPUT_TYPE` | `pdfa` | Archival output type (PDF/A-2b) |
| `PAPERLESS_OCR_LANGUAGE` | `eng` | Tesseract language |
| `PAPERLESS_REDIS` | `redis://localhost:6379` | Broker + channel-layer backend |
| `PAPERLESS_DATA_DIR` / `PAPERLESS_MEDIA_ROOT` / `PAPERLESS_CONSUMPTION_DIR` | (runtime) | Runtime dirs — pointed at `/tmp` to keep the repo tree clean |

### Appendix F — Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `git diff --name-status <base> HEAD` | Confirm the read-only net diff (exactly one added file) |
| `grep -nE '^## (Q1\|Q2\|Q3\|Q4)' <deliverable>` | Jump to any answer section |
| `curl` | Drive the REST upload and inspect serialized responses |
| WebSocket client (`ws/status/`) | Observe the live `STARTING → WORKING → SUCCESS/FAILED` sequence |
| `hashlib.md5` + `md5sum` | Byte-exact checksum verification (Q3) |
| `python manage.py qcluster` | Run/observe the django-q worker executing `consume_file` |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **OCRmyPDF** | The OCR engine wrapper paperless invokes (v13.4.3); orchestrates tesseract/ghostscript/unpaper. |
| **skip / skip_noarchive / redo / force** | `OCR_MODE` values mapping to OCRmyPDF flags (`skip_text` / `skip_text`+early-return / `redo_ocr` / `force_ocr`). Default is `skip`. |
| **sidecar** | The text file OCRmyPDF emits containing only recognized text; a `"[OCR skipped on page"` marker triggers pdfminer fallback. |
| **django-q / qcluster** | The background task queue (not Celery) and its worker-cluster process that runs `consume_file`. |
| **ws/status/** | The Django Channels WebSocket route carrying live processing progress. |
| **`content`** | The `Document` field holding text regardless of provenance (OCR-generated or pre-existing). |
| **`archived_file_name` / `checksum` / `archive_checksum`** | The indirect provenance signals — the only way to distinguish OCR-generated vs pre-existing text. |
| **"Fully processed"** | Binary here: it simply means a `Document` row exists; even empty-content OCR yields a fully processed document. |
| **`[observed-at-runtime]` / `[inferred-from-reading]`** | Provenance tags distinguishing captured runtime evidence from source-derived statements. |

---

*This Blitzy Project Guide assesses an AAP-scoped, read-only documentation deliverable. Completion (92.6%) is measured strictly against AAP scope + path-to-production for a documentation artifact: 75 hours completed, 6 hours remaining, 81 hours total. All figures are consistent across Sections 1.2, 2.1, 2.2, 2.3, and 7.*