# Blitzy Project Guide — paperless-ngx Document-Classification Test Flakiness Analysis

> **Project Type:** Documentation-only Q&A investigation (rule: `SWE-AtlasQnA-Repo`)
> **Branch:** `blitzy-3c3a1abf-36b6-494e-a456-954ef9e6f43f` · **Base:** `542221a38` · **HEAD:** `13cd19cf0`
> **Sole Deliverable:** `blitzy/documentation/paperless-ngx_542221a38dff.md` (923 lines)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, comprehensive, code-grounded markdown analysis that diagnoses **why the paperless-ngx document-classification tests fail non-deterministically ("flaky")**. The audience is the engineering team maintaining paperless-ngx and the reviewing stakeholder who posed five precise questions (Q1–Q5) about the ML classification and OCR pipeline. The investigation is strictly **read-only**: it answers each question with inline `[path:line]` citations, corroborates conclusions by building and running targeted tests in the pinned Docker environment (Python 3.9, scikit-learn 1.0.2), and synthesizes the root causes of non-determinism. The only persistent repository change is the analysis document itself; **no source code is modified**, honoring the verbatim user constraint.

### 1.2 Completion Status

The completion percentage is computed using the AAP-scoped, hours-based PA1 methodology: only work defined in the Agent Action Plan plus standard path-to-production activity (human acceptance review and PR merge) is counted.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Project Completion — 93.5% Complete (29h of 31h)
    "Completed Work (AI)" : 29
    "Remaining Work" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | 31.0 h |
| **Completed Hours (AI + Manual)** | 29.0 h (AI: 29.0 h · Manual: 0.0 h) |
| **Remaining Hours** | 2.0 h |
| **Percent Complete** | **93.5 %** |

> **Calculation:** Completion % = Completed ÷ Total × 100 = 29.0 ÷ 31.0 × 100 = **93.5 %**.

### 1.3 Key Accomplishments

- ✅ **Single deliverable authored & committed** — `blitzy/documentation/paperless-ngx_542221a38dff.md` (923 lines), the only repository change (`A` status, 0 source files touched).
- ✅ **All five questions answered** with an *Evidence → Observed behavior → Rationale* structure and explicit reasoning.
- ✅ **Consolidated non-determinism root-cause synthesis** identifying the prime suspect (`MLPClassifier` instantiated with **no `random_state`** at `classifier.py` L219/L227/L238) plus three contributing causes.
- ✅ **105/105 code citations verified** for content accuracy (not just line existence) by the validator and independently corroborated.
- ✅ **Behavior confirmed by running tests** — 9 distinct referenced tests across all five questions executed and passing; an MLP determinism probe reproduced the documented non-deterministic output.
- ✅ **Two factual corrections** to the prior tech spec captured (barcode logic lives in `tasks.py`, not `barcodes.py`; matching tests in `test_matchables.py`, not `test_matching.py`).
- ✅ **Source integrity guaranteed** — `git diff` against the base shows zero changes under `src/`; temporary probe removed; working tree clean.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| *None blocking.* The deliverable is complete, accurate, and committed. | No release blocker for this documentation task. | — | — |
| Underlying ML test non-determinism remains **unfixed by design** (out of AAP scope) | Ongoing CI flakiness in paperless-ngx classification tests; the document provides exact remediation but does **not** implement it (per the "do not modify source code" constraint) | paperless-ngx maintainers (follow-up) | Tracked as backlog item (H3) |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| paperless-ngx repository | Git read/write | Full access available; deliverable committed at `13cd19cf0` | ✅ Resolved | Blitzy Agent |
| Docker runtime image | Build/run | Pinned image (Python 3.9 + scikit-learn 1.0.2 etc.) available; tests executed successfully | ✅ Resolved | Blitzy Agent |

**No access issues identified** that would prevent build validation, integration, or deployment of this documentation deliverable.

### 1.6 Recommended Next Steps

1. **[High]** Technical reviewer performs a final accuracy read of the deliverable and signs off (spot-check 3–5 of the 105 citations against the source).
2. **[Medium]** Review and merge the single-file documentation PR into the target branch.
3. **[Low]** Triage the document's §7.6 remediation recommendations (add `random_state` to the three `MLPClassifier` instantiations; tighten per-test `MODEL_FILE` isolation) into a **separate** follow-up engineering ticket — governance only; do **not** bundle the code fix into this documentation PR.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to an AAP requirement (investigation, authoring, verification, or assembly of the single deliverable).

| Component | Hours | Description |
|-----------|-------|-------------|
| 1. Repository scope discovery & investigation setup | 2.5 | Mapped the in-scope files across `documents` and `paperless_tesseract` apps; corrected the standalone-barcode-module and `test_matching.py` assumptions against the real layout (AAP §0.2). |
| 2. Q1 — Model reuse vs. retrain analysis | 2.5 | Read `load_classifier`/`train`/`save` and the SHA-1 `data_hash` short-circuit; documented model reuse vs. retrain (`classifier.py` L30-57, L161-164). |
| 3. Q2 — Correspondent matching analysis | 3.0 | Established 3 documents created / 2 effective (inbox excluded), training-after-insert timing, and the **absence** of any probability/confidence threshold. |
| 4. Q3 — No-text OCR analysis | 2.5 | Enumerated the `ocrmypdf.ocr` primary call, the force-OCR fallback, the empty-text last resort, and MIME assignment via `magic.from_file`. |
| 5. Q4 — Barcode splitting analysis | 2.5 | Documented the K separators → K+1 documents rule, the `"PATCHT"` trigger, and the split-decision site in `scan_file_for_separating_barcodes`. |
| 6. Q5 — Split effect on training data | 2.0 | Traced the `consume_file` barcode branch returning before `Document` creation; explained the +N effective-training-set effect. |
| 7. Non-determinism synthesis & recommendations | 2.5 | Synthesized four root causes and authored documentation-only remediation guidance (§7). |
| 8. Behavioral verification & determinism probe | 3.5 | Built/ran the environment; authored and ran a temporary MLP probe reproducing 8 distinct weight signatures across 8 identical fits; cleaned it up afterward. |
| 9. Web research (corroboration) | 1.5 | Validated `MLPClassifier` `random_state` semantics and pytest-xdist isolation best practice against authoritative sources. |
| 10. Citation accuracy & content verification (105) | 3.0 | Verified all 105 `[path:line]` citations for content (not just line existence) across 16 source files. |
| 11. Introduction, methodology & references assembly | 1.5 | Authored the introduction/methodology and the references list (code + paraphrased external sources). |
| 12. Final-acceptance review, revision & source-integrity cleanup | 2.0 | Addressed final-acceptance review findings; confirmed zero source changes; removed temporary artifacts. |
| **Total Completed** | **29.0** | |

### 2.2 Remaining Work Detail

Each remaining category is a path-to-production human activity; the non-determinism *fix* is explicitly out of AAP scope and therefore **not** counted as remaining project work.

| Category | Hours | Priority |
|----------|-------|----------|
| Technical accuracy review & sign-off of the deliverable (H1) | 1.0 | High |
| PR review & merge of the single-file documentation deliverable (H2) | 0.5 | Medium |
| Triage §7.6 recommendations into a follow-up backlog ticket (governance only) (H3) | 0.5 | Low |
| **Total Remaining** | **2.0** | |

### 2.3 Hours Reconciliation

| Check | Computation | Result |
|-------|-------------|--------|
| Section 2.1 total | sum of 12 completed items | 29.0 h |
| Section 2.2 total | 1.0 + 0.5 + 0.5 | 2.0 h |
| Total Project Hours | 29.0 + 2.0 | 31.0 h |
| Completion % | 29.0 ÷ 31.0 × 100 | 93.5 % |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project and were **independently re-run** during this assessment. Framework: **pytest 7.1.1** with `pytest-django`. Invocation pattern (parallelism disabled for deterministic targeting): `cd src && ../.venv/bin/python -m pytest <target> -o addopts="" -n0 --no-cov -p no:sugar -W ignore`.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Q1 — Classifier lifecycle (Unit) | pytest-django | 3 | 3 | 0 | n/a | `testDatasetHashing`, `testSaveClassifier`, `testTrain` — confirm data-hash reuse/retrain. |
| Q2 — Correspondent prediction (Unit) | pytest-django | 1 | 1 | 0 | n/a | `testPredict`; validator ran it 20× for stability; stable on the tiny fixture. |
| Q3 — No-text OCR (Integration) | pytest-django | 2–3 | all | 0 | n/a | `test_with_form_error_notext`, `test_with_form_force` (and `test_skip_noarchive_notext`) invoke `ocrmypdf`. |
| Q4 — Barcode reader/split (Unit) | pytest-django | 4 | 4 | 0 | n/a | `test_separate_pages`, `test_barcode_reader`, `_qr`, `_128` in class `TestTasks`. |
| Q5 — Barcode consumption (Integration) | pytest-django | 2 | 2 | 0 | n/a | `test_consume_barcode_file`, `test_separate_pages` — split → re-consume behavior. |
| **Totals** | | **~13 runs / 9 distinct** | **all** | **0** | n/a | Zero failures across all five questions. |

**Independently corroborated by this assessment:** all 9 distinct tests above (re-run in batches: a 5-test batch, a 7-test batch, and a 2-test OCR batch — all green). Application sanity: `manage.py check` → "System check identified no issues (0 silenced)". Dependency sanity: `pip check` → "No broken requirements found." Coverage % is recorded as n/a because the deliverable is documentation; the test runs serve as behavioral corroboration of the analysis, not coverage of new code.

> **Note on non-determinism:** `testPredict` is *stable* on the small checked-in fixture, which is why the flakiness is intermittent in CI. The determinism probe (outside the repo) demonstrated the underlying variability — 8 distinct fitted-weight signatures across 8 identical fits — confirming the documented root cause without contradicting the green test runs.

---

## 4. Runtime Validation & UI Verification

This is a documentation-only deliverable with **no UI component**; "runtime validation" maps to verifying the analysis environment and the behaviors the document describes.

- ✅ **Operational** — Django app boots cleanly: `manage.py check` reports no issues (0 silenced).
- ✅ **Operational** — Pinned environment verified: Python 3.9.23; scikit-learn 1.0.2, ocrmypdf 13.4.3, pyzbar 0.1.9, pdf2image 1.16.0, numpy 1.22.3, scipy 1.8.0, joblib 1.1.0, Django 4.0.4, pytest 7.1.1, pytest-xdist 2.5.0 — all match AAP pins.
- ✅ **Operational** — Dependency graph consistent: `pip check` → "No broken requirements found."
- ✅ **Operational** — All 9 referenced tests execute and pass across the five questions.
- ✅ **Operational** — MLP determinism probe runs and reproduces documented variability (then removed; verified absent).
- ✅ **Operational** — Markdown deliverable renders correctly; headings, code blocks, and citations well-formed.
- ⚠ **Partial (by design)** — The underlying classification tests remain non-deterministic in parallel CI; this is the diagnosed condition, intentionally **not** remediated in this scope.
- ❌ **Failing** — None.

---

## 5. Compliance & Quality Review

The governing rule set is `SWE-AtlasQnA-Repo`. All eight directives are satisfied.

| Compliance Benchmark (SWE-AtlasQnA-Repo) | Status | Progress | Evidence / Notes |
|------------------------------------------|--------|----------|------------------|
| Create `<source_branch_name>.md` | ✅ Pass | 100% | `blitzy/documentation/paperless-ngx_542221a38dff.md` named for base commit `542221a38dff`. |
| Comprehensively answer the posed questions | ✅ Pass | 100% | Q1–Q5 + non-determinism synthesis all answered. |
| Build & run source to analyze behavior | ✅ Pass | 100% | 9 tests run; `manage.py check` clean; MLP probe executed. |
| Base answers on code as truth (no assumptions) | ✅ Pass | 100% | 105/105 `[path:line]` citations content-verified. |
| Provide thinking / rationale | ✅ Pass | 100% | Each answer uses Evidence → Observed behavior → Rationale. |
| Do not modify existing source files | ✅ Pass | 100% | `git diff --name-only 542221a38 HEAD -- src/` → empty. |
| Do not add other code besides the document | ✅ Pass | 100% | Only one `A` file; temp probe under `/tmp` removed. |
| Place document under `blitzy/documentation/` | ✅ Pass | 100% | Confirmed path; parent directory created. |

**Additional quality gates:** markdown passes applicable pre-commit hooks (LF line endings, no trailing whitespace, final newline present); external references in §8.2 are **paraphrased, not quoted** (copyright-safe); two prior-spec inaccuracies corrected; temporary helpers cleaned up.

**Outstanding compliance items:** None. The remediation recommendations are intentionally documentation-only and do not constitute a compliance gap.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Citation line numbers drift if source evolves | Technical | Low–Medium | Medium | Document anchors on stable symbol names; code is authoritative if lines shift; snapshot commit recorded | Mitigated by design |
| T2 — Analysis is snapshot-bound to base `542221a38` | Technical | Low | Low | Base commit and environment pins documented for reproducibility | Accepted |
| S1 — Security exposure from the deliverable | Security | None | Low | Documentation-only; no secrets, no code, no attack surface introduced | Not applicable |
| O1 — Underlying ML test non-determinism remains unfixed | Operational | Medium | High | Out of AAP scope by constraint; document provides exact remediation steps for a follow-up ticket | Open by design (codebase risk, not a deliverable defect) |
| O2 — Document discoverability | Operational | Low | Low | Placed under conventional `blitzy/documentation/`; referenced in PR | Mitigated |
| I1 — Reproducibility depends on pinned environment | Integration | Low | Low | §8.3 records exact run/reproduction commands and pinned versions | Mitigated |

**Overall risk profile: LOW.** No High-severity risks. The single Medium item (O1) is a pre-existing codebase condition that the deliverable diagnoses; remediating it is explicitly out of scope.

---

## 7. Visual Project Status

### 7.1 Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Project Hours — Completed 29h vs Remaining 2h (93.5% Complete)
    "Completed Work" : 29
    "Remaining Work" : 2
```

### 7.2 Remaining Work by Priority

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Remaining 2.0h by Priority
    "High — Accuracy review (H1)" : 1.0
    "Medium — PR merge (H2)" : 0.5
    "Low — Triage backlog (H3)" : 0.5
```

> **Integrity:** "Remaining Work" = **2.0 h**, matching the Section 1.2 metrics table and the Section 2.2 "Hours" total exactly. "Completed Work" = **29.0 h**, matching Section 1.2 and Section 2.1.

### 7.3 Remaining Hours per Category (Bar View)

| Category | Hours | Bar |
|----------|-------|-----|
| High — Accuracy review (H1) | 1.0 | ██████████ |
| Medium — PR merge (H2) | 0.5 | █████ |
| Low — Triage backlog (H3) | 0.5 | █████ |
| **Total** | **2.0** | |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a rigorous, fully code-cited analysis (923 lines) that answers all five questions and synthesizes the root causes of document-classification test flakiness in paperless-ngx. The prime suspect is precisely identified: the `MLPClassifier` is instantiated with only `tol=0.01` and **no `random_state`** (`classifier.py` L219/L227/L238), making stochastic weight initialization vary run-to-run. Three contributing causes are documented: persisted/shared on-disk model reuse via the SHA-1 `data_hash` short-circuit, uneven per-test `MODEL_FILE` isolation, and parallel `pytest-xdist` execution exposing shared-file coupling. Every conclusion is corroborated by running tests and, for the prime cause, by a determinism probe.

**Remaining gaps.** Only standard path-to-production activity remains: a human technical sign-off, PR merge of the single file, and triaging the remediation recommendations into a separate ticket. These total **2.0 hours**.

**Critical path to production.** Reviewer accuracy sign-off (H1, 1.0 h) → PR merge (H2, 0.5 h) → backlog triage of recommendations (H3, 0.5 h).

**Production readiness.** The deliverable is **93.5 % complete** on an AAP-scoped basis and is production-ready as a documentation artifact: it is committed, accurate, self-contained, and changes no source code. The remaining 6.5 % is human review/merge that cannot be performed autonomously.

| Success Metric | Target | Actual | Status |
|----------------|--------|--------|--------|
| Questions answered | 5 + synthesis | 5 + synthesis | ✅ |
| Citation accuracy | 100% | 105/105 | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Referenced tests passing | 100% | 9/9 distinct | ✅ |
| AAP-scoped completion | ≤ 99% pre-review | 93.5% | ✅ |

**Recommendation:** Approve and merge the documentation PR after a brief accuracy review, then open a separate engineering ticket to implement the `random_state` and test-isolation fixes the document recommends.

---

## 9. Development Guide

This guide reproduces the investigation environment, verifies the deliverable, and re-runs the referenced tests. All commands are copy-pasteable and were executed during validation. Run from the repository root unless noted.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu-based container) or macOS.
- **Python:** 3.9.x (pinned 3.9.23) — required for scikit-learn 1.0.2 model-load compatibility.
- **Git** ≥ 2.x.
- **System libraries** (for OCR/barcode tests): `tesseract-ocr`, `unpaper`, `pngquant`, `zbar` (`libzbar0`), `poppler-utils`. Provided by the pinned Docker image.
- A pre-built virtualenv at `.venv/` (provided) or the ability to create one.

### 9.2 Environment Setup

```bash
# From the repository root
cd /tmp/blitzy/paperless-ngx/blitzy-3c3a1abf-36b6-494e-a456-954ef9e6f43f_af690c

# Confirm the pinned interpreter
.venv/bin/python --version          # expected: Python 3.9.23

# Confirm dependency health
.venv/bin/pip check                 # expected: No broken requirements found.
```

If you must build the venv yourself:

```bash
python3.9 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 9.3 Dependency Verification

```bash
.venv/bin/python -c "import sklearn, ocrmypdf, pyzbar, pdf2image, numpy, scipy, joblib, django, pytest; \
print('sklearn', sklearn.__version__); print('django', django.__version__)"
# expected: sklearn 1.0.2 ; django 4.0.4
```

### 9.4 Application / Investigation Startup

There is no server to run for this task. Validate the Django project instead:

```bash
cd src
../.venv/bin/python manage.py check
# expected: System check identified no issues (0 silenced).
```

### 9.5 Verification Steps

**a) Confirm the deliverable exists and changes nothing else:**

```bash
# From repo root — only ONE added file, zero source changes
git diff --name-status 542221a38 HEAD
# expected: A    blitzy/documentation/paperless-ngx_542221a38dff.md

git diff --name-only 542221a38 HEAD -- src/
# expected: (empty)

wc -l blitzy/documentation/paperless-ngx_542221a38dff.md
# expected: 923
```

**b) Re-run the referenced tests (parallelism disabled for targeting):**

```bash
cd src
# Q1 + Q2 + Q4 lifecycle/prediction/barcode (7 tests)
../.venv/bin/python -m pytest \
  documents/tests/test_classifier.py::TestClassifier::testDatasetHashing \
  documents/tests/test_classifier.py::TestClassifier::testSaveClassifier \
  documents/tests/test_classifier.py::TestClassifier::testTrain \
  documents/tests/test_classifier.py::TestClassifier::testPredict \
  documents/tests/test_tasks.py::TestTasks::test_separate_pages \
  documents/tests/test_tasks.py::TestTasks::test_barcode_reader_qr \
  documents/tests/test_tasks.py::TestTasks::test_barcode_reader_128 \
  -o addopts="" -n0 --no-cov -p no:sugar -W ignore
# expected: 7 passed

# Q3 no-text OCR (2 tests; slower — invokes ocrmypdf)
../.venv/bin/python -m pytest \
  paperless_tesseract/tests/test_parser.py -k "notext or force" \
  -o addopts="" -n0 --no-cov -p no:sugar -W ignore
# expected: passed (≈15s)
```

**c) Read the deliverable:**

```bash
sed -n '1,60p' blitzy/documentation/paperless-ngx_542221a38dff.md
grep -n '^#' blitzy/documentation/paperless-ngx_542221a38dff.md   # section map
```

### 9.6 Example Usage

To reproduce the prime-cause finding (no `random_state`) directly from the source:

```bash
grep -n "MLPClassifier" src/documents/classifier.py
# expect three instantiations at ~L219/L227/L238, each with tol=0.01 and NO random_state

grep -rn "predict_proba" src/
# expect: (no matches) — confirms there is no probability/confidence threshold
```

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---------|-------|------------|
| `error: unrecognized arguments: -n0` | `--numprocesses auto` baked into `setup.cfg addopts` | Always pass `-o addopts=""` then add `-n0` explicitly. |
| Tests try to run the whole suite in parallel | Default `addopts` includes `--numprocesses auto --cov` | Use `-o addopts="" -n0 --no-cov` for targeted runs. |
| `class TestBarcode not found` | Barcode tests are not in a dedicated class/file | Tests live in `documents/tests/test_tasks.py` class **`TestTasks`** (`test_separate_pages` ≈ L305). |
| OCR test errors about missing binaries | Missing `tesseract`/`zbar` | Use the pinned Docker image, or install `tesseract-ocr` and `libzbar0`. |
| `externally-managed-environment` on pip | System Python PEP 668 marker | Use the project `.venv/` (preferred) or `pip install --break-system-packages`. |
| Pre-commit complains about whitespace/newline | Markdown formatting | Ensure LF endings, no trailing whitespace, single final newline. |

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Python version | `.venv/bin/python --version` |
| Dependency health | `.venv/bin/pip check` |
| Django system check | `cd src && ../.venv/bin/python manage.py check` |
| Targeted test run | `cd src && ../.venv/bin/python -m pytest <target> -o addopts="" -n0 --no-cov -p no:sugar -W ignore` |
| Deliverable diff (status) | `git diff --name-status 542221a38 HEAD` |
| Source-only diff (should be empty) | `git diff --name-only 542221a38 HEAD -- src/` |
| Section map of deliverable | `grep -n '^#' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Confirm no probability threshold | `grep -rn "predict_proba" src/` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| (none) | — | Documentation-only task; no server is started. |

### Appendix C — Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (923 lines). |
| `src/documents/classifier.py` | Model load/train/save; `MLPClassifier` (no `random_state`); `predict_correspondent`. |
| `src/documents/tasks.py` | `train_classifier`; barcode scan/split; `consume_file` barcode branch. |
| `src/documents/consumer.py` | MIME via `magic.from_file` (L219). |
| `src/paperless_tesseract/parsers.py` | OCR primary call, force-OCR fallback, empty-text last resort. |
| `src/paperless/settings.py` | `MODEL_FILE`, `CONSUMER_BARCODE_STRING="PATCHT"`, `OCR_OUTPUT_TYPE="pdfa"`. |
| `src/documents/tests/test_classifier.py` | Q1/Q2 lifecycle & prediction tests; isolation overrides. |
| `src/documents/tests/test_tasks.py` | Q4/Q5 barcode tests in class `TestTasks`. |
| `src/setup.cfg` | pytest config (`--numprocesses auto`, `PAPERLESS_DISABLE_DBHANDLER=true`). |

### Appendix D — Technology Versions

| Package | Version | Relevance |
|---------|---------|-----------|
| python | 3.9.23 | Pinned runtime. |
| scikit-learn | 1.0.2 | `MLPClassifier`; model-load compatibility. |
| joblib | 1.1.0 | Model serialization. |
| numpy | 1.22.3 | scikit-learn backend. |
| scipy | 1.8.0 | scikit-learn backend. |
| ocrmypdf | 13.4.3 | OCR subprocess (Q3). |
| pdf2image | 1.16.0 | PDF rendering for barcode scan (Q4). |
| pyzbar | 0.1.9 | Barcode decoding (Q4). |
| django | 4.0.4 | Web/ORM framework. |
| pytest | 7.1.1 | Test framework. |
| pytest-xdist | 2.5.0 | Parallel execution — central to flakiness analysis. |

### Appendix E — Environment Variable Reference

| Variable | Value | Source |
|----------|-------|--------|
| `PAPERLESS_DISABLE_DBHANDLER` | `true` | `src/setup.cfg` (test env). |
| `PAPERLESS_CONSUMER_ENABLE_BARCODES` | default `False` | `settings.py` L502 (`CONSUMER_ENABLE_BARCODES`). |
| `PAPERLESS_CONSUMER_BARCODE_STRING` | default `"PATCHT"` | `settings.py` L506. |
| `PAPERLESS_OCR_OUTPUT_TYPE` | default `"pdfa"` | `settings.py` L518. |
| `PAPERLESS_OCR_MODE` | default `"skip"` | `settings.py` L522. |

### Appendix F — Developer Tools Guide

| Tool | Use |
|------|-----|
| `pytest` (`-o addopts="" -n0 --no-cov`) | Run targeted tests deterministically, bypassing the parallel/coverage defaults. |
| `git diff --name-status <base> HEAD` | Prove the single-file change and zero source modifications. |
| `grep -rn "predict_proba" src/` | Demonstrate the absence of a probability threshold (Q2). |
| `manage.py check` | Validate Django app integrity before/after investigation. |
| MLP determinism probe (temporary, under `/tmp`) | Reproduce run-to-run weight variability; removed before finishing. |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the scoped requirements driving this project. |
| **`data_hash`** | SHA-1 over non-inbox documents (ordered by PK) used to short-circuit retraining. |
| **`MLPClassifier`** | scikit-learn multi-layer perceptron; here instantiated **without** `random_state`, the prime non-determinism cause. |
| **`random_state`** | scikit-learn seed for reproducible weight/bias initialization; absent in this codebase. |
| **`PATCHT`** | Default `CONSUMER_BARCODE_STRING` value that triggers a page split. |
| **K separators → K+1 docs** | Barcode split rule: K trigger pages yield K+1 output documents. |
| **pytest-xdist** | Parallel test runner (`--numprocesses auto`); exposes shared on-disk model coupling. |
| **Flaky test** | A test whose pass/fail outcome varies across runs without code changes. |

---

*Generated by the Blitzy Platform · AAP-scoped completion: 93.5% (29.0h of 31.0h) · Remaining: 2.0h human review/merge.*
