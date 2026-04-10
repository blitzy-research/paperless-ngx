# Blitzy Project Guide — Paperless-ngx v1.7.0 Multi-Page PDF OCR Pipeline Documentation

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive technical Q&A documentation file for the Paperless-ngx v1.7.0 multi-page PDF OCR processing pipeline. The sole deliverable is a standalone markdown document (`blitzy/documentation/paperless-ngx_542221a38dff.md`) that answers empirical questions about upload HTTP response behavior, processing log patterns, OCRmyPDF invocation parameters, generated filenames, and database record fields. All answers are derived from static source code analysis of 17 Python source files, following a strict "code-as-truth" directive with 31 inline source citations. No existing repository files were modified.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (17h)" : 17
    "Remaining (2h)" : 2
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 19h |
| **Completed Hours (AI)** | 17h |
| **Remaining Hours (Human)** | 2h |
| **Completion Percentage** | **89.5%** |

**Calculation:** 17h completed / (17h completed + 2h remaining) = 17/19 = **89.5% complete**

### 1.3 Key Accomplishments

- ✅ Delivered 858-line comprehensive Q&A documentation file (54,876 bytes) covering all 4 user questions with code-derived rationale
- ✅ Completed deep static analysis of 17 source files across `src/documents/`, `src/paperless_tesseract/`, and `src/paperless/` packages
- ✅ Documented exact HTTP 200 OK upload response with full 5-step code-path trace from URL routing through serializer to async task queuing
- ✅ Cataloged 11 processing stages with logger names, levels, message formats, and WebSocket progress status payloads
- ✅ Produced complete 15-parameter OCRmyPDF invocation table mapping every parameter to its source setting and default value
- ✅ Documented two-pass OCR fallback strategy critical for understanding inconsistent OCR results
- ✅ Created 16-field Document model database schema table with types, constraints, defaults, and example values
- ✅ Documented post-consumption signal handler chain (6 handlers) and their effects on the final database record
- ✅ Created Mermaid sequence diagram (12 participants) tracing the full upload→process→store lifecycle
- ✅ Created Mermaid flowchart documenting the OCR mode decision tree with two-pass fallback path
- ✅ Passed all validation gates: Prettier formatting, file integrity (UTF-8, LF, no trailing whitespace), zero TODO/FIXME markers
- ✅ Zero existing source files modified — clean `git diff` showing only 1 file added
- ✅ Working tree clean with committed deliverable

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| No critical issues identified | — | — | — |

All AAP requirements have been delivered. No blocking issues remain.

### 1.5 Access Issues

No access issues identified. The project is a documentation-only task that relies exclusively on source code analysis. No external services, API credentials, database connections, or runtime environments were required.

### 1.6 Recommended Next Steps

1. **[High]** Human peer review of all 31 source code citations to verify accuracy against actual source lines
2. **[Medium]** Address any feedback or corrections identified during review
3. **[Low]** Consider integrating the document into the existing Sphinx documentation build at `docs/` if ongoing maintenance is desired
4. **[Low]** Evaluate whether to add automated link-checking for source citations to detect code drift in future versions

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source Code Analysis | 3.5 | Deep analysis of 17 source files: views.py, consumer.py, parsers.py (×2), models.py, file_handling.py, settings.py, tasks.py, serialisers.py, loggers.py, signals/__init__.py, handlers.py, apps.py, urls.py, consumers.py, version.py, signals.py |
| Q1: HTTP Response Documentation | 1.5 | Full 5-step code-path trace from URL routing through PostDocumentSerializer validation to async_task queuing and Response("OK") return |
| Q2: Processing Log Patterns | 2.5 | 11-stage log catalog with logger names, levels, message formats; WebSocket progress status payload structure; OCR parser internal logs (10 entries) |
| Q2: OCRmyPDF Parameters | 1.5 | 15-parameter invocation table mapping each ocrmypdf.ocr() parameter to its source setting and default value; complete default args dictionary; OCR_USER_ARGS handling |
| Q2: Two-Pass Fallback Strategy | 1.0 | Detailed documentation of first-pass/fallback-pass OCR behavior with error handling matrix (EncryptedPdfError, NoTextFoundException, InputFileError) |
| Q3: Generated Filenames | 1.5 | Default naming patterns ({pk:07}.pdf/.png), storage directory layout, 6-step filename assignment during consumption, custom PAPERLESS_FILENAME_FORMAT placeholders, post-save renaming handler |
| Q4: Database Record Fields | 2.0 | Complete 16-field Document model table with Django field types, DB column types, constraints, defaults; _store() field assignment; apply_overrides(); 6-handler signal chain; example values |
| Mermaid Sequence Diagram | 1.0 | 12-participant upload→process→store lifecycle sequence diagram covering Client through Signal Handlers |
| Mermaid OCR Flowchart | 0.5 | Complex decision-tree flowchart documenting OCR mode routing with two-pass fallback path |
| Media Directory Tree | 0.5 | Text-based storage layout diagram with originals/archive/thumbnails directories and example filenames |
| Environment Setup Context | 0.5 | Environment settings reference table (8 entries) and OCR defaults configuration table (12 entries) |
| Document Structure & Formatting | 0.5 | Table of contents with anchor links, 39 markdown headers, cleanup notes section, consistent citation format |
| Validation & Compliance | 0.5 | Prettier formatting compliance, file integrity verification, zero-placeholder enforcement, SWE-AtlasQnA-Repo constraint validation |
| **Total Completed** | **17.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Review of Source Citation Accuracy | 1.5 | Medium |
| Address Review Feedback & Corrections | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

### 2.3 Hours Verification

- **Section 2.1 Total:** 17.0h
- **Section 2.2 Total:** 2.0h
- **Sum (2.1 + 2.2):** 17.0 + 2.0 = **19.0h** ✓
- **Section 1.2 Total Project Hours:** **19.0h** ✓
- **Completion:** 17.0 / 19.0 = **89.5%** ✓

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Markdown Formatting | Prettier 2.6.2 | 1 | 1 | 0 | 100% | `--check` mode: "All matched files use Prettier code style!" |
| File Integrity | Custom (bash) | 5 | 5 | 0 | 100% | UTF-8 encoding, LF line endings, no trailing whitespace, ends with newline, valid file size (54,876 bytes) |
| Placeholder Detection | Custom (grep) | 1 | 1 | 0 | 100% | Zero TODO/FIXME/PLACEHOLDER/HACK markers found |
| Content Completeness | Custom (section scan) | 5 | 5 | 0 | 100% | All required sections present: Q1, Q2, Q3, Q4, Cleanup Notes |
| Git Cleanliness | git status | 1 | 1 | 0 | 100% | `git status --porcelain` returns empty — working tree clean |
| **Totals** | | **13** | **13** | **0** | **100%** | |

All tests originate from Blitzy's autonomous validation pipeline. No external or pre-existing test suites were executed (this is a documentation-only project with no source code changes).

---

## 4. Runtime Validation & UI Verification

### File Existence and Integrity

- ✅ **File exists**: `blitzy/documentation/paperless-ngx_542221a38dff.md` present at expected path
- ✅ **File size**: 54,876 bytes (non-trivial, comprehensive document)
- ✅ **Line count**: 858 lines of content
- ✅ **Encoding**: Valid UTF-8 text
- ✅ **Line endings**: LF (Unix-style) throughout
- ✅ **Trailing whitespace**: None detected on any line
- ✅ **Terminal newline**: File ends with newline character

### Content Structure Verification

- ✅ **Headers**: 39 markdown headers providing clear document structure
- ✅ **Table data rows**: 164 table rows across parameter tables, field tables, and log catalogs
- ✅ **Code blocks**: 58 fenced code block markers (29 code blocks with syntax highlighting)
- ✅ **Source citations**: 31 inline source citations referencing specific file paths and line numbers
- ✅ **Mermaid diagrams**: 2 diagrams (sequence diagram + flowchart)
- ✅ **Table of contents**: Complete with anchor links to all major sections

### Git Status

- ✅ **Committed**: Commit `bb34bf24f` on branch `blitzy-bcaa849e-b391-4553-844b-102132a09919`
- ✅ **Working tree**: Clean (no uncommitted changes)
- ✅ **Diff from base**: 1 file added, 858 insertions, 0 deletions
- ✅ **No existing files modified**: `git diff --name-status` shows only `A` (added) status

### No Runtime Services Required

- ⚠ **N/A**: This is a documentation-only project. No application servers, databases, or external services need to be running for the deliverable to be valid.

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|-----------------|--------|----------|
| Create new markdown document in `blitzy/documentation/` | ✅ Pass | File exists at `blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Name file `<source_branch_name>.md` | ✅ Pass | Named `paperless-ngx_542221a38dff.md` matching source branch `paperless-ngx_542221a38dff` |
| Q1: Document HTTP response status and body | ✅ Pass | HTTP 200 OK, body "OK" with 5-step code-path trace (lines 42-183) |
| Q2: Document processing log patterns | ✅ Pass | 11-stage catalog, WebSocket progress, OCR parser logs (lines 187-317) |
| Q2 sub: Document OCRmyPDF invocation parameters | ✅ Pass | 15-parameter table with defaults and source settings (lines 319-382) |
| Q2 sub: Document two-pass fallback strategy | ✅ Pass | First pass, error handling matrix, fallback pass documentation (lines 384-416) |
| Q3: Document generated filenames | ✅ Pass | Default patterns, storage layout, 6-step assignment, custom format (lines 420-595) |
| Q4: Document database record fields | ✅ Pass | 16-field table, _store() values, signal chain, examples (lines 599-703) |
| Include Mermaid sequence diagram | ✅ Pass | 12-participant upload→process→store diagram (lines 707-768) |
| Include Mermaid OCR flowchart | ✅ Pass | Decision tree with fallback path (lines 772-815) |
| Include directory tree diagram | ✅ Pass | Text-based media storage layout (lines 481-492) |
| Provide source code citations | ✅ Pass | 31 inline citations with file paths and line numbers |
| Provide thinking/rationale for answers | ✅ Pass | Each Q section includes "Rationale" or "Code-Path Trace" subsections |
| Base answers on code (code-as-truth) | ✅ Pass | All claims traceable to specific source files; methodology stated in Overview |
| Do not modify existing files | ✅ Pass | `git diff --name-status` shows only `A` (add) for single new file |
| No permanent changes / clean up test files | ✅ Pass | No test files created; Cleanup Notes section confirms no artifacts |
| Prettier formatting compliance | ✅ Pass | `prettier --check` reports all files pass |
| Zero TODO/FIXME/placeholder markers | ✅ Pass | grep scan finds 0 occurrences |

**Compliance Score: 18/18 (100%)**

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source citation line numbers may drift as codebase evolves beyond v1.7.0 | Technical | Low | Medium | Document explicitly states version scope (v1.7.0); citations reference specific commit state | Mitigated |
| Source citations may reference incorrect line numbers within v1.7.0 | Technical | Medium | Low | Human peer review of all 31 citations recommended before merging | Open |
| Document is standalone, not integrated into Sphinx docs build | Operational | Low | Low | By design per AAP — `SWE-AtlasQnA-Repo` rule specifies `blitzy/documentation/` | Accepted |
| Mermaid diagrams may not render in all markdown viewers | Technical | Low | Low | Mermaid is widely supported (GitHub, GitLab, VS Code); raw code blocks remain readable | Accepted |
| No runtime verification of documented behavior (static analysis only) | Technical | Low | Low | AAP explicitly permits code-derived answers; methodology clearly stated in document overview | Accepted |

No security or integration risks apply to this documentation-only project. No credentials, API keys, external services, or runtime environments are involved.

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 17
    "Remaining Work" : 2
```

**Breakdown:**
- **Completed Work (17h)**: Source code analysis, Q1–Q4 documentation, diagrams, formatting, validation — all delivered by Blitzy autonomous agents
- **Remaining Work (2h)**: Human peer review of source citation accuracy (1.5h) + address review feedback (0.5h)

---

## 8. Summary & Recommendations

### Achievement Summary

The project has achieved **89.5% completion** (17 hours completed out of 19 total hours). All AAP-scoped autonomous work has been delivered successfully — a single comprehensive 858-line Q&A documentation file that answers all four user questions about the Paperless-ngx v1.7.0 multi-page PDF OCR processing pipeline.

The deliverable provides code-traced answers covering the complete document lifecycle: HTTP upload response (Q1), processing log patterns and OCRmyPDF parameters (Q2), generated filenames and storage structure (Q3), and database record fields with signal handler effects (Q4). Two Mermaid diagrams visualize the processing lifecycle and OCR decision tree. All 31 source citations reference specific file paths and line numbers.

### Remaining Gaps

The only remaining work (2 hours) is path-to-production human review:
1. **Citation Verification (1.5h)**: A subject matter expert should verify all 31 source citations against the actual v1.7.0 source code to confirm line number accuracy
2. **Review Feedback (0.5h)**: Address any corrections or improvements identified during peer review

### Critical Path to Production

1. Merge this PR after human review of source citation accuracy
2. No infrastructure, deployment, or configuration steps required — the deliverable is a standalone documentation file

### Production Readiness Assessment

| Criterion | Status |
|-----------|--------|
| All AAP requirements delivered | ✅ |
| All validation gates passed | ✅ |
| No blocking issues | ✅ |
| No existing files modified | ✅ |
| Working tree clean | ✅ |
| Ready for human review | ✅ |

The project is **ready for human review and merge** upon completion of the remaining 2 hours of peer review work.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Git | 2.x+ | Clone repository and inspect branch |
| Markdown viewer | Any | Read and render the documentation file |
| Node.js (optional) | 14+ | Run Prettier formatting checks |

### Environment Setup

No special environment setup is required. The deliverable is a standalone markdown file that can be viewed with any markdown-capable tool.

```bash
# Clone the repository and switch to the feature branch
git clone <repository-url>
cd paperless-ngx
git checkout blitzy-bcaa849e-b391-4553-844b-102132a09919
```

### Viewing the Documentation

```bash
# View the documentation file
cat blitzy/documentation/paperless-ngx_542221a38dff.md

# Or view with line numbers
nl -ba blitzy/documentation/paperless-ngx_542221a38dff.md | less

# View file metadata
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected output: 858 blitzy/documentation/paperless-ngx_542221a38dff.md

ls -la blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected output: 54876 bytes
```

### Verification Steps

```bash
# 1. Verify the file exists and has expected size
test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo "File exists" || echo "File missing"
# Expected: File exists

# 2. Verify line count
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected: 858

# 3. Verify no existing files were modified
git diff --name-status origin/paperless-ngx_542221a38dff...HEAD
# Expected: A    blitzy/documentation/paperless-ngx_542221a38dff.md

# 4. Verify Prettier formatting compliance
npx prettier@2.6.2 --check blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected: "All matched files use Prettier code style!"

# 5. Verify no TODO/FIXME markers
grep -c 'TODO\|FIXME\|PLACEHOLDER\|HACK' blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected: 0

# 6. Verify working tree is clean
git status --porcelain
# Expected: (empty output)
```

### Reviewing Source Citations

The document contains 31 source citations in the format `Source: <file_path>:<line_number>`. To verify a citation:

```bash
# Example: Verify the Response("OK") citation at views.py:535
sed -n '535p' src/documents/views.py
# Expected: line containing Response("OK")

# Example: Verify OCR settings at settings.py:514
sed -n '514p' src/paperless/settings.py
# Expected: line containing OCR_LANGUAGE
```

### Troubleshooting

| Issue | Resolution |
|-------|------------|
| Mermaid diagrams not rendering | Use a Mermaid-compatible viewer (GitHub, GitLab, VS Code with Mermaid extension) |
| Prettier check fails after edits | Run `npx prettier@2.6.2 --write blitzy/documentation/paperless-ngx_542221a38dff.md` to reformat |
| Source citation line number doesn't match | Line numbers reference v1.7.0 codebase; if code has changed since, verify against the original commit |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `cat blitzy/documentation/paperless-ngx_542221a38dff.md` | View the documentation file |
| `wc -l blitzy/documentation/paperless-ngx_542221a38dff.md` | Count lines (expected: 858) |
| `npx prettier@2.6.2 --check blitzy/documentation/paperless-ngx_542221a38dff.md` | Verify formatting compliance |
| `git diff --name-status origin/paperless-ngx_542221a38dff...HEAD` | Confirm only 1 file added |
| `git log --oneline HEAD --not origin/paperless-ngx_542221a38dff` | View branch commits |

### B. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **Deliverable** — comprehensive Q&A documentation |
| `src/documents/views.py` | Upload endpoint (PostDocumentView) — analyzed for Q1 |
| `src/documents/consumer.py` | Consumer pipeline — analyzed for Q2, Q3, Q4 |
| `src/paperless_tesseract/parsers.py` | OCR parser — analyzed for Q2 (OCRmyPDF parameters, fallback) |
| `src/documents/models.py` | Document model — analyzed for Q3 (thumbnail_path), Q4 (fields) |
| `src/documents/file_handling.py` | Filename generation — analyzed for Q3 |
| `src/paperless/settings.py` | Configuration — analyzed for Q2 (OCR defaults), Q3 (media paths) |
| `src/documents/signals/handlers.py` | Post-consumption handlers — analyzed for Q3, Q4 |
| `src/documents/apps.py` | Signal wiring — analyzed for Q4 (handler chain) |

### C. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Paperless-ngx | 1.7.0 | `src/paperless/version.py` |
| Django | 4.0.4 | `requirements.txt` |
| Django REST Framework | 3.13.1 | `requirements.txt` |
| OCRmyPDF | 13.4.3 | `requirements.txt` |
| Pillow | 9.1.0 | `requirements.txt` |
| pikepdf | 5.1.1 | `requirements.txt` |
| pdfminer.six | 20220319 | `requirements.txt` |
| django-q | 1.3.9 | `requirements.txt` |
| channels | 3.0.4 | `requirements.txt` |
| Redis | 3.5.3 | `requirements.txt` |
| Prettier (validation) | 2.6.2 | Blitzy validation toolchain |
| Python | 3.8+ | `.readthedocs.yml`, `Pipfile` |

### D. Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the primary directive defining all project requirements |
| Consumer | The `Consumer` class in `src/documents/consumer.py` that orchestrates the full document ingestion lifecycle |
| OCRmyPDF | Python library wrapping Tesseract OCR for PDF processing; invoked via `ocrmypdf.ocr()` |
| Sidecar file | A `.txt` file produced by OCRmyPDF alongside the archive PDF containing the extracted text |
| Safe fallback | The second-pass OCR strategy using `force_ocr=True` and ignoring `OCR_USER_ARGS` for robustness |
| SWE-AtlasQnA-Repo | Blitzy implementation rule requiring a new markdown file in `blitzy/documentation/` without modifying existing files |
| PDF/A | An ISO-standardized subset of PDF designed for long-term digital preservation; the default OCR output type |