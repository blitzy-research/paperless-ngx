# Blitzy Project Guide

**Project:** paperless-ngx — Root-cause investigation of "haunted" documents-list pagination
**Deliverable branch:** `blitzy-05bc5bf4-24a0-4b86-b38a-86e4556a2ec2`
**Source HEAD (pinned):** `542221a38dff06361e07976452f9aea24d210542`
**Project rule:** `SWE-AtlasQnA-Repo` (documentation-only; source repository immutable)

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers a posed diagnostic question — strictly from the paperless-ngx source code as the source of truth — explaining why the documents list "can feel haunted during normal browsing": paging a filtered documents view can make the same document appear on two neighboring pages (a *duplicate*) or vanish for a page and then reappear (a *skip*), even though nobody edits anything and the sort order looks unchanged. The audience is the maintainer/SME who reported the behavior. The technical scope spans the Django/DRF backend query-filter-paginate pipeline and the Angular frontend pagination model. The deliverable is exactly **one** new Markdown document; no source code is changed. Business impact: a precise, evidence-backed root cause that de-risks any future fix.

### 1.2 Completion Status

The completion percentage is computed using the AAP-scoped, hours-based methodology (completed hours ÷ total hours). All work in scope is the production of one rigorous, code-anchored, empirically-validated investigation document plus the read-only analysis behind it.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333'}}}%%
pie showData
    title Completion — 92% (23.0h of 25.0h)
    "Completed Work (AI)" : 23
    "Remaining Work" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **25.0 h** |
| Completed Hours (AI + Manual) | 23.0 h (23.0 AI autonomous + 0.0 manual) |
| Remaining Hours | 2.0 h |
| **Percent Complete** | **92.0%** |

> Color legend: **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`** (applied consistently across Sections 1.2 and 7).

### 1.3 Key Accomplishments

- ✅ Authored the sole deliverable `blitzy/documentation/paperless-ngx_542221a38dff.md` (239 lines), a complete root-cause Q&A with request-lifecycle trace, SQL operation-order analysis, hypothesis verdicts, permission-premise reconciliation, and a conceptual remedy.
- ✅ Adjudicated all three user hypotheses with code evidence: **H1 partially true** (not the cross-page cause), **H2 false**, **H3 true — the root cause** (non-unique `-created` ordering with no unique tiebreaker).
- ✅ Reconciled the "sharing rules / what the user can see" premise against this code version, proving there is **no object-level permission layer** (`permission_classes = (IsAuthenticated,)`; zero matches for object-permission primitives in `src/`).
- ✅ Code-anchored every claim: **51/51 citations verified** against the pinned HEAD (0 mismatches); corrected two AAP locator drifts.
- ✅ Empirically reproduced the exact duplicate/skip symptom on production-matching **PostgreSQL 13.23** (no row edits), and demonstrated that appending `id` to the sort eliminates it.
- ✅ Preserved source immutability: `git diff` confirms a **single-file add**; `src/`, `src-ui/`, and manifests are byte-for-byte unchanged; working tree clean; all ephemeral artifacts removed.
- ✅ Confirmed no regression: backend `pytest` baseline **481 passed / 2 skipped**; frontend `jest` **5/5**.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| None blocking | The deliverable is complete, accurate, and committed; no defects, no compilation/test failures attributable to the work | — | — |
| (Clarification, non-blocking) Live-environment paperless-ngx version unconfirmed | Determines whether Section 5's permission-layer caveat applies; root cause is version-invariant | Requester / SME | 0.5 h |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository (`paperless-ngx`) | Read/write (branch) | None — repository checked out at pinned HEAD; commits applied successfully | ✅ Resolved | Blitzy |
| Docker / PostgreSQL 13 | Container runtime | None — canonical Docker image available; empirical reproduction completed | ✅ Resolved | Blitzy |
| Requester's **live** environment | Operational context | Not accessible to Blitzy; needed only to confirm which paperless-ngx version is deployed (the #1 follow-up) | ⚠ Pending requester input | Requester |

No access issues prevented build validation, analysis, or delivery. The only outstanding item is an informational question for the requester, not an access blocker.

### 1.6 Recommended Next Steps

1. **[High]** Confirm with the requester which paperless-ngx version their live environment runs and whether it includes an object-level permission/sharing layer (validates the Section 5 caveat). — 0.5 h
2. **[Medium]** SME review: read the 239-line document and spot-check a sample of the 51 `file:line` citations and the H1/H2/H3 verdicts. — 1.0 h
3. **[Medium]** Approve and merge the single-file documentation PR. — 0.5 h
4. **[Low]** *(Out of scope for this task)* If the team elects to act on the diagnosis, open a **separate** code-change task to append a unique tiebreaker (`ORDER BY created, id`) or adopt keyset/seek pagination.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to specific AAP requirements (R1–R20). Total matches Completed Hours in Section 1.2.

| Component | Hours | Description |
|-----------|------:|-------------|
| Request-lifecycle investigation (read-only) | 6.0 | Cross-stack tracing across the 10 AAP evidence files: Django/DRF routing → `UnifiedSearchViewSet` ORM-vs-Whoosh branching → `DjangoFilterBackend`/`OrderingFilter` → `Document.objects.distinct()` → `StandardPagination` `LIMIT`/`OFFSET`; plus the Angular list/REST services. [R1, R3, R16] |
| SQL semantics analysis + web-search corroboration | 2.5 | Reasoned the `DISTINCT → ORDER BY → LIMIT/OFFSET` operation order; corroborated with PostgreSQL §7.6 "LIMIT and OFFSET" and Markus Winand's keyset/tiebreaker guidance. [R2, R4] |
| Document authoring | 5.0 | Wrote the 239-line root-cause write-up: verbatim Q&A, lifecycle (+mermaid), SQL shape, H1/H2/H3 verdicts (+verdict table), permission reconciliation, conceptual remedy, summary/references, and two appendices. [R5–R15] |
| Citation anchoring & verification | 3.0 | Anchored every claim to `file:line` at the pinned HEAD; verified 51/51 citations and corrected two AAP locator drifts (settings file path; `models.py` line numbers). [R16] |
| Empirical corroboration (Docker / PostgreSQL 13) | 4.0 | Reproduced the duplicate/skip symptom on PostgreSQL 13.23 after a heap reorganization with no row edits; demonstrated the `ORDER BY created DESC, id` remedy yields zero duplicates/skips; proved the ORM SQL shape. [R17] |
| Immutability, cleanup & regression baseline | 2.5 | Verified source byte-for-byte unchanged; removed all ephemeral artifacts; ran the regression baseline (backend 481 passed / 2 skipped; frontend jest 5/5 preserved). [R18, R19, R20] |
| **Total Completed** | **23.0** | |

### 2.2 Remaining Work Detail

All remaining work is path-to-production human activity; there is **no remaining AAP code work**. Total matches Remaining Hours in Section 1.2 and the Section 7 pie "Remaining Work" value.

| Category | Hours | Priority |
|----------|------:|----------|
| Confirm version/permission assumption with requester (the #1 follow-up) | 0.5 | High |
| SME review of the deliverable (read doc; spot-check citations & verdicts) | 1.0 | Medium |
| Merge the documentation PR to the target branch | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

> *Excluded from project hours (out of AAP scope per §0.5.2):* implementing the conceptual remedy (tiebreaker / keyset pagination) is a separate future code-change task and is intentionally **not** counted in the 25.0 h total.

### 2.3 Total Project Hours & Methodology

| Roll-up | Hours |
|---------|------:|
| Section 2.1 — Completed | 23.0 |
| Section 2.2 — Remaining | 2.0 |
| **Total Project Hours** | **25.0** |

**Completion formula (PA1, AAP-scoped):** `Completed ÷ Total = 23.0 ÷ 25.0 = 92.0%`. Hours reflect senior-engineer diagnostic effort (cross-stack code comprehension, SQL semantics reasoning, technical writing, citation verification, and Docker-based empirical reproduction). Scope is defined exclusively by the AAP deliverable plus path-to-production acceptance; remedy implementation is excluded as out of scope.

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project. Because the deliverable is a Markdown document with **no runtime surface**, the software tests function as a **regression baseline** proving the unchanged source still behaves identically; the documentation-specific validation (citation verification + empirical reproduction) confirms the deliverable's correctness.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Backend unit/integration (regression baseline) | pytest (Django 4.0.4) | 483 | 481 | 0 | N/A* | 2 skipped. One transient `paperless_tesseract` failure diagnosed as an **environment file-permission artifact** (non-root user cannot overwrite a root-owned sample image) — not a code defect, not a regression; resolved environment-only → 481/481. |
| Frontend unit (regression baseline) | jest (`ng test`) | 5 | 5 | 0 | N/A* | Preserved by construction — zero `src-ui/` changes. |
| Documentation citation verification | Manual/scripted vs source HEAD | 51 | 51 | 0 | 100% of cited claims | 0 mismatches; corrected 2 AAP locator drifts. |
| Empirical symptom reproduction | Django ORM + PostgreSQL 13.23 | 2 scenarios | 2 | 0 | — | (a) SQL-shape proof: `distinct()/all()/?ordering=-created` all compile to `ORDER BY created DESC` with no `id` tiebreaker; (b) duplicate/skip reproduced after heap reorg with no edits; remedy `ORDER BY created DESC, id` → 0 dup/skip. |

\* Coverage percentages are not asserted for the regression baselines because the source is unchanged and no coverage delta is attributable to this documentation-only deliverable; reporting a fabricated number would violate integrity rules.

**Aggregate:** 541 autonomous checks executed (483 backend + 5 frontend + 51 citations + 2 empirical scenarios), 0 genuine failures. The single transient backend failure was an environment artifact, not a code or deliverable defect.

---

## 4. Runtime Validation & UI Verification

The deliverable is a static Markdown document and introduces **no runtime or UI surface of its own**. Runtime validation therefore confirms (a) the unchanged application still runs/builds, and (b) the documented behavior was reproduced empirically.

- ✅ **Operational** — Backend Django app + ORM ran successfully inside the canonical Docker container during empirical proofs (Django 4.0.4, DRF 3.13.1).
- ✅ **Operational** — Production-matching **PostgreSQL 13.23** reproduced the exact duplicate/skip symptom across neighboring `LIMIT`/`OFFSET` pages after a heap reorganization with no row edits; the `id`-tiebreaker remedy eliminated it.
- ✅ **Operational** — Frontend production build path unaffected (zero `src-ui/` changes); `jest` 5/5 preserved.
- ✅ **Operational** — Document renders as well-formed Markdown: balanced code fences (1 mermaid flowchart + 2 SQL blocks), a valid verdict table, and all internal cross-references resolve.
- ✅ **Operational** — API ↔ UI correlation documented and verified against source: the API returns `Results<T>{count, results}` while the UI computes `getLastPage() = Math.ceil(count / pageSize)`, assuming a stable partition the backend does not guarantee on ties.
- ⚠ **Partial (informational)** — Permission-correlation framing depends on the requester's live paperless-ngx version (pinned analysis has no permission layer). Non-blocking; tracked as the #1 follow-up.
- ❌ **Failing** — None.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverables and project-rule directives to Blitzy quality/compliance benchmarks.

| Benchmark / AAP Directive | Requirement | Status | Evidence / Notes |
|---------------------------|-------------|--------|------------------|
| Single deliverable (rule `SWE-AtlasQnA-Repo`) | Exactly one new Markdown file, correctly named & located | ✅ Pass | `blitzy/documentation/paperless-ngx_542221a38dff.md` present (239 lines) |
| Source immutability | No existing file created/modified/deleted | ✅ Pass | `git diff` vs HEAD = single add; `src/`,`src-ui/`,manifests unchanged; tree clean |
| Code is the source of truth | Every claim anchored to `file:line`; no assumptions | ✅ Pass | 51/51 citations verified; 2 AAP locator drifts corrected |
| Verbatim fidelity | Preserve the user's symptom statement + 3 hypotheses + premise + method | ✅ Pass | 4 verbatim quote blocks preserved exactly |
| Hypothesis adjudication | Answer H1/H2/H3 with evidence & rationale | ✅ Pass | H1 partially true; H2 false; H3 root cause — all code-backed |
| Permission-premise reconciliation | Test "sharing rules" premise against this version | ✅ Pass | No object-level layer; `IsAuthenticated` only; reframed to filters + tie-break |
| Web-search corroboration | External validation of `LIMIT`/`OFFSET`-on-ties semantics + remedy | ✅ Pass | PostgreSQL §7.6; Winand keyset/tiebreaker; cited as corroboration only |
| Conceptual remedy, not implemented | Describe fix without changing code | ✅ Pass | Section 6 explicitly states no code is written |
| Ephemeral-script cleanup | Temporary observation scripts never committed | ✅ Pass | All artifacts removed; tree clean |
| Zero-placeholder policy | No TODO/FIXME/stub content | ✅ Pass | Only "not implemented" phrasing is the intentional remedy framing |
| Markdown quality | Well-formed structure, diagrams, tables | ✅ Pass | Balanced fences; mermaid + 2 SQL blocks; verdict table |

**Fixes applied during autonomous validation:** corrected the PostgreSQL external reference to match the production image (`postgres:13`); corrected two AAP locator drifts (settings file path; `models.py` ordering line numbers 207–208). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Version/permission-layer assumption mismatch (live env may be a newer release with sharing) | Integration | Medium | Medium | Confirm requester's version before generalizing the permission framing; root cause (unstable tie ordering) is version-invariant | ⚠ Open (tracked as HT-1; already flagged in doc §5) |
| Reproduction-backend divergence (SQLite dev vs PostgreSQL prod) | Technical | Low | Medium | Doc directs reproduction to PostgreSQL (`postgres:13`) where OFFSET reorder-on-ties is most visible; empirical proof on PG 13.23 | ✅ Mitigated |
| Citation line-number drift if read against a different commit | Technical/Operational | Low | Low | Every citation pinned to HEAD `542221a38dff`, stated explicitly in the doc header | ✅ Mitigated |
| Root-cause correctness (could H3 be wrong?) | Technical | High (if wrong) | Very Low | Empirically reproduced on production-matching PostgreSQL 13.23; 51 verified citations; H1/H2 disproved via operation-order reasoning | ✅ Closed |
| Source-repository immutability violation | Operational/Process | High (if violated) | Very Low | `git diff` confirms single-file add; regression baseline 481 passed/2 skipped confirms no behavioral change | ✅ Closed |
| Reader treats conceptual remedy as a delivered fix | Operational | Low | Low | Doc repeatedly states the remedy is conceptual and **not** implemented; implementing it is a separate out-of-scope task | ✅ Mitigated |

**Security risks:** None identified — documentation-only change; no code, no dependency additions/updates, no runtime or attack surface introduced.
**Operational/Integration risks beyond the above:** None — static Markdown artifact; no external services, credentials, monitoring, or deployment surface.
**Net posture:** Very low. Five of six tracked risks are mitigated/closed; the single open item is a non-blocking clarifying question already surfaced in the deliverable.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`). The "Remaining Work" value (2) equals Section 1.2 Remaining Hours and the Section 2.2 total.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333'}}}%%
pie showData
    title Project Hours Breakdown (Total 25.0h)
    "Completed Work" : 23
    "Remaining Work" : 2
```

**Remaining work by category** (hours from Section 2.2; sums to 2.0 h):

| Category | Hours | Priority | Bar |
|----------|------:|----------|-----|
| Confirm version/permission assumption | 0.5 | High | ██████ |
| SME review of deliverable | 1.0 | Medium | ████████████ |
| Merge documentation PR | 0.5 | Medium | ██████ |
| **Total** | **2.0** | | |

**Completed work by component** (hours from Section 2.1; sums to 23.0 h):

| Component | Hours | Bar |
|-----------|------:|-----|
| Request-lifecycle investigation | 6.0 | ████████████████████████ |
| Document authoring | 5.0 | ████████████████████ |
| Empirical corroboration (PostgreSQL 13) | 4.0 | ████████████████ |
| Citation anchoring & verification | 3.0 | ████████████ |
| SQL semantics + web research | 2.5 | ██████████ |
| Immutability, cleanup & regression baseline | 2.5 | ██████████ |
| **Total** | **23.0** | |

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.0% complete** (23.0 h of 25.0 h AAP-scoped). The sole deliverable is finished, validated, and committed: a 239-line, fully code-anchored root-cause document that adjudicates all three hypotheses (H1 partially true; H2 false; **H3 — unstable ordering on a non-unique sort key — is the root cause**), reconciles the "sharing rules" premise against a code version that has no object-level permission layer, and describes (without implementing) the deterministic-ordering remedy. The conclusion is corroborated externally (PostgreSQL docs; keyset-pagination practice) and **empirically reproduced** on production-matching PostgreSQL 13.23.

**Remaining gaps & critical path.** The remaining 2.0 h is entirely path-to-production human work: (1) confirm the requester's live paperless-ngx version, (2) SME review, (3) merge. There is no remaining AAP code work, and the source repository is byte-for-byte unchanged.

**Success metrics.** Single-file deliverable ✓ · immutability preserved ✓ · 51/51 citations verified ✓ · all 4 verbatim quotes preserved ✓ · empirical reproduction + remedy demonstrated ✓ · regression baseline green (481 passed/2 skipped backend; jest 5/5 frontend) ✓.

**Production-readiness assessment.** The deliverable is **production-ready** for its purpose (a diagnostic document). It is accurate, complete, and self-contained, with no defects and no compilation/test failures attributable to the work. The only caveat is the version assumption — a clarifying question, not a defect. Recommended path: confirm version → SME review → merge. Should the team choose to remediate the underlying behavior, open a separate, out-of-scope code-change task to append a unique `ORDER BY` tiebreaker or adopt keyset pagination.

---

## 9. Development Guide

This guide explains how to view, verify, and (optionally) reproduce the findings. Every command was tested in the project environment. All commands assume the repository root:
`/tmp/blitzy/paperless-ngx/blitzy-05bc5bf4-24a0-4b86-b38a-86e4556a2ec2_6cd322`.

### 9.1 System Prerequisites

| Tool | Observed in env | Notes |
|------|-----------------|-------|
| git | 2.51.0 | Required to view the deliverable and verify immutability |
| Python | 3.13.7 (host) | Project-canonical runtime for **building** the pinned source is **3.9** (CI 3.8/3.10) — see 9.5 |
| Node.js / npm | 20.20.2 / 11.1.0 (host) | Project-canonical for building the frontend is **Node 16** — see 9.5 |
| Docker | 28.5.2 | Optional — needed only to reproduce the symptom on PostgreSQL 13 |

> Viewing/verifying the deliverable requires only **git + a text viewer**. Python/Node/Docker are needed only for the optional build-and-reproduce path.

### 9.2 Get the Code (checkout the pinned HEAD)

```bash
# Already on the deliverable branch in this environment:
git rev-parse --abbrev-ref HEAD          # -> blitzy-05bc5bf4-24a0-4b86-b38a-86e4556a2ec2
git log --oneline -2                     # -> 94d162bbe, ecfbe9d9e (both agent@blitzy.com)

# The analysis is pinned to this source commit:
git rev-parse 542221a38dff06361e07976452f9aea24d210542
```

### 9.3 View the Deliverable (primary path — no build required)

```bash
# Size and length
ls -l   blitzy/documentation/paperless-ngx_542221a38dff.md     # 28,517 bytes
wc -l   blitzy/documentation/paperless-ngx_542221a38dff.md     # 239 lines

# Read it (use your pager of choice)
sed -n '1,60p' blitzy/documentation/paperless-ngx_542221a38dff.md

# Table of contents
grep -nE '^#{1,3} ' blitzy/documentation/paperless-ngx_542221a38dff.md
```

### 9.4 Verify Immutability (the core project constraint)

```bash
# 1) Only one file added vs the source HEAD (expect a single 'A' line)
git diff --name-status 542221a38dff06361e07976452f9aea24d210542..HEAD
#   A    blitzy/documentation/paperless-ngx_542221a38dff.md

# 2) Source tree untouched (expect EMPTY output)
git diff --stat 542221a38dff06361e07976452f9aea24d210542..HEAD -- src/ src-ui/ requirements.txt Pipfile Pipfile.lock

# 3) Working tree clean (expect EMPTY output)
git status --porcelain

# 4) Markdown sanity: balanced fences (expect 6) and 4 verbatim quotes
grep -c '```' blitzy/documentation/paperless-ngx_542221a38dff.md   # 6 (3 pairs)
grep -c '^> "' blitzy/documentation/paperless-ngx_542221a38dff.md  # 4

# 5) Spot-check the root-cause citation
sed -n '207,208p' src/documents/models.py                          # ordering = ("-created",)
```

### 9.5 (Optional) Build & Run to Reproduce on PostgreSQL

The AAP permits building/running the source to corroborate behavior. Use the **canonical Docker image** or Python 3.9 / Node 16 to match the pinned dependencies (Django 4.0.4, etc.); building on the host Python 3.13 may fail against the pinned requirements.

```bash
# Bring up a production-like stack (PostgreSQL 13 is the production DB image)
docker compose -f docker/compose/docker-compose.postgres.yml up -d

# Confirm the DB image pin
grep -n 'image: postgres' docker/compose/docker-compose.postgres.yml   # image: postgres:13
```

### 9.6 (Optional) Reproduce the Symptom (Appendix A procedure)

```bash
# Issue successive authenticated page requests and diff the id sets across boundaries.
# (Concentrate same-`created` rows by enabling a tag/correspondent filter.)
for N in 1 2 3; do
  curl -s "http://localhost:8000/api/documents/?page=$N&page_size=25&ordering=-created" \
    -H "Authorization: Token <YOUR_TOKEN>" \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print('page', $N, 'count', d['count'], 'ids', [r['id'] for r in d['results']])"
done
# An id appearing on BOTH page N and N+1 is a DUPLICATE; an id on NEITHER (yet within count) is a SKIP.
```

### 9.7 (Optional) Regression Baseline

```bash
# Backend (from src/): expect 481 passed / 2 skipped
cd src && python -m pytest -q --tb=short ; cd ..

# Frontend (from src-ui/): expect 5/5
cd src-ui && CI=true npx ng test --watch=false ; cd ..
```

### 9.8 Troubleshooting

- **Symptom won't reproduce on SQLite.** SQLite often returns rows in a more stable physical order for simple queries. Use **PostgreSQL (`postgres:13`)** where the planner may legitimately reorder tied rows per `OFFSET`; concentrate same-`created` rows with a tag/correspondent filter.
- **Citation line numbers don't match.** Ensure your checkout is exactly HEAD `542221a38dff`; the document pins all line numbers to that commit.
- **`paperless_tesseract` test fails (`test_image_simple_alpha`).** Environment file-permission artifact (a non-root user cannot overwrite a root-owned sample image). Resolve environment-only (e.g., fix sample-file permissions); it is **not** a code defect, **not** a regression, and **not** in scope for this deliverable.
- **Dependency install errors on host Python 3.13 / Node 20.** Build with Python 3.9 / Node 16 or the provided Docker image to match the pinned `requirements.txt`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Show branch / recent commits | `git log --oneline -2` |
| List changed files vs source HEAD | `git diff --name-status 542221a38dff06361e07976452f9aea24d210542..HEAD` |
| Prove source unchanged | `git diff --stat 542221a38dff..HEAD -- src/ src-ui/ requirements.txt Pipfile` |
| Confirm clean tree | `git status --porcelain` |
| Verify authorship | `git log --author="agent@blitzy.com" 542221a38dff..HEAD --oneline` |
| View deliverable TOC | `grep -nE '^#{1,3} ' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Check fence balance | `grep -c '```' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Backend tests | `cd src && python -m pytest -q` |
| Frontend tests | `cd src-ui && CI=true npx ng test --watch=false` |
| Bring up PostgreSQL stack | `docker compose -f docker/compose/docker-compose.postgres.yml up -d` |

### B. Port Reference

| Service | Port | Notes |
|---------|------|-------|
| paperless-ngx web/API (dev) | 8000 | Default Django/DRF dev server; API at `/api/documents/` |
| PostgreSQL | 5432 | From `docker-compose.postgres.yml` (`postgres:13`) |

*The deliverable itself exposes no ports; the above apply only to the optional reproduce-the-symptom path.*

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The sole deliverable** (root-cause analysis) |
| `src/paperless/urls.py` (L32) | Registers `documents` → `UnifiedSearchViewSet` |
| `src/documents/views.py` (L172, 183–199, 377–426) | `DocumentViewSet`/`UnifiedSearchViewSet`: queryset/`distinct()`, `IsAuthenticated`, filter backends, `ordering_fields`, ORM-vs-Whoosh branching |
| `src/paperless/views.py` (L8–11) | `StandardPagination` (`PageNumberPagination`, `LIMIT`/`OFFSET`) |
| `src/documents/models.py` (L207–208) | `Document.Meta.ordering = ("-created",)` — non-unique, no tiebreaker |
| `src/documents/filters.py` (L42–58) | `TagsFilter` M2M JOIN + `.distinct()` |
| `src-ui/.../document-list-view.service.ts` (L93–94, 276–278) | UI sort defaults; `getLastPage = ceil(count/pageSize)` |
| `src-ui/.../rest/abstract-paperless-service.ts` (L24–30) | Single `ordering` param, no tiebreaker |
| `src-ui/.../data/results.ts` (L1–5) | `Results<T>{count, results}` contract |
| `src-ui/.../services/settings.service.ts` (L39, 68) | `DOCUMENT_LIST_SIZE` default `50` |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Django | 4.0.4 | `requirements.txt` |
| djangorestframework | 3.13.1 | `requirements.txt` |
| django-filter | 21.1 | `requirements.txt` |
| psycopg2 | 2.9.3 | `requirements.txt` |
| whoosh | 2.7.4 | `requirements.txt` |
| @angular/core | ~13.3.4 | `src-ui/package.json` |
| PostgreSQL (production) | 13 | `docker/compose/docker-compose.postgres.yml` |
| Python (canonical build) | 3.9 (CI 3.8/3.10) | AAP §0.6.1 |
| Node (canonical build) | 16 | AAP §0.6.1 |

### E. Environment Variable Reference

The deliverable requires **no** environment variables. For the optional reproduce-the-symptom path, the only ad-hoc value is an authentication token for API calls:

| Variable | Used by | Notes |
|----------|---------|-------|
| `<YOUR_TOKEN>` | `curl` Authorization header | A valid paperless-ngx API token for an authenticated user (optional path only) |

Standard paperless-ngx runtime configuration (e.g., `PAPERLESS_DBHOST`, database credentials) is documented in `docker/compose/docker-compose.env` and `paperless.conf.example`; none is needed to view or verify the deliverable.

### F. Developer Tools Guide

| Tool | Use in this project |
|------|--------------------|
| `git diff` / `git status` | Prove source immutability (the core constraint) |
| `pytest` | Backend regression baseline (481 passed / 2 skipped) |
| `ng test` (jest) | Frontend regression baseline (5/5) |
| `docker compose` | Optional production-like PostgreSQL 13 stack for reproduction |
| `curl` + `python3 -m json.tool` | Observe `/api/documents/` page payloads (Appendix A) |
| Mermaid renderer | View the lifecycle flowchart in the deliverable |

### G. Glossary

| Term | Definition |
|------|------------|
| **Duplicate symptom** | The same document appearing on two neighboring pages |
| **Skip symptom** | A document missing from a page and reappearing later |
| **Tiebreaker** | A unique column (e.g., `id`/`pk`) appended to `ORDER BY` to make the sort a total order |
| **Keyset / seek pagination** | Pagination using `WHERE (sort_key, id) > (...)` instead of `LIMIT`/`OFFSET`; stable and efficient for deep paging |
| **`DISTINCT` before `LIMIT`/`OFFSET`** | Within one SQL statement, de-duplication is applied before the page window is sliced |
| **ORM metadata path** | The non-full-text `/api/documents/` branch (no `query`/`more_like_id`), governed by SQL `ORDER BY` |
| **Whoosh path** | The full-text branch (`query`/`more_like_id` present), ordered by relevance `score`, out of scope for this symptom |
| **Pinned HEAD** | Source commit `542221a38dff06361e07976452f9aea24d210542` to which all citations refer |

---

*Prepared by the Blitzy autonomous platform. Completion: 92.0% (23.0 h of 25.0 h). Source repository byte-for-byte unchanged; the sole deliverable is `blitzy/documentation/paperless-ngx_542221a38dff.md`.*