# Blitzy Project Guide — Paperless-ngx REST API Authentication Investigation

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive, evidence-based technical investigation document (`blitzy/documentation/paperless-ngx_542221a38dff.md`) answering concrete, testable questions about the Paperless-ngx v1.7.0 REST API authentication system. The document targets integration developers who need to programmatically interact with the API from external tools, scripts, and automation platforms. It covers local setup, token provisioning, authenticated and unauthenticated request behavior, JSON response structure, Django code tracing of the authentication class chain and token model, custom authentication middleware, security advisories for pinned dependency vulnerabilities, and production hardening recommendations. All answers are verified against the actual codebase with 25 source code citations and 5 live API test scenarios.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (AI)" : 26
    "Remaining" : 3.5
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 29.5 |
| **Completed Hours (AI)** | 26 |
| **Remaining Hours** | 3.5 |
| **Completion Percentage** | 88.1% |

**Calculation:** 26 completed hours / 29.5 total hours = 88.1% complete

### 1.3 Key Accomplishments

- ✅ Created comprehensive 898-line investigation document with 12 complete sections
- ✅ Verified all 25 source code citations against actual repository files and line numbers
- ✅ Executed and documented 5 live API test scenarios (authenticated, unauthenticated, invalid token, token acquisition, wrong keyword)
- ✅ Traced Django authentication class chain from `settings.py` configuration through `TokenAuthentication` to `Token` model
- ✅ Documented pagination behavior (`StandardPagination`: page_size=25, max_page_size=100000)
- ✅ Included Mermaid authentication flow diagram
- ✅ Added security advisory for Django 4.0.4 (45+ CVEs) and DRF 3.13.1 (CVE-2024-21520)
- ✅ Added 4 security hardening recommendations for production deployments
- ✅ Passed Prettier v2.6.2 formatting compliance and all pre-commit checks
- ✅ Maintained read-only codebase policy — zero source code files modified
- ✅ Included Thinking/Rationale subsection in every section per SWE-AtlasQnA-Repo rule
- ✅ Clean git working tree with all changes committed

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Human review of documentation accuracy required | Content may need minor corrections based on stakeholder domain expertise | Human Developer | 2 hours |
| Security advisory dependency upgrades not applied | Django 4.0.4 and DRF 3.13.1 CVEs remain in production codebase | Project Maintainers | Separate effort |

### 1.5 Access Issues

No access issues identified. The project is a documentation-only deliverable that does not require external service credentials, API keys, or special repository permissions beyond standard read access. All source code was accessible for analysis.

### 1.6 Recommended Next Steps

1. **[High]** Review documentation content for domain accuracy — verify that all 12 sections correctly represent the intended API behavior as understood by the Paperless-ngx maintainers
2. **[High]** Merge the PR after review — the document is complete, formatted, and committed on the feature branch
3. **[Medium]** Act on security advisory — evaluate upgrading Django to >=4.2.28 LTS and DRF to >=3.15.2 to address documented CVEs
4. **[Low]** Consider integrating the document into the Sphinx documentation tree — the standalone markdown could be converted to RST and linked from `docs/api.rst`
5. **[Low]** Establish a review schedule — source code citations (25 references) should be re-verified when the Paperless-ngx codebase is updated

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository analysis and code investigation | 5 | Analyzed 13 source files across `src/paperless/`, `src/documents/`, and DRF library code to map authentication system architecture |
| Live API testing environment setup | 2 | Configured temporary environment variables, ran database migrations, started development server for API testing |
| Live API test execution and capture | 1 | Executed 5 test scenarios (authenticated, unauthenticated, invalid token, token acquisition, wrong keyword) and captured exact HTTP responses |
| Documentation authoring — Sections 1–6 (Core Q&A) | 6 | Authored core investigation sections: local setup, user/token creation, authenticated requests, JSON response structure, unauthenticated errors, Django code trace |
| Documentation authoring — Sections 7–10 (Supplementary) | 4 | Authored supplementary sections: custom authentication classes, alternative auth methods, cleanup instructions, quick reference summary |
| Documentation authoring — Sections 11–12 (Security) | 2.5 | Researched and authored security advisory for Django/DRF CVEs and production hardening recommendations |
| Mermaid authentication flow diagram | 0.5 | Designed and created flowchart diagram showing DRF authentication class evaluation chain |
| Source citation verification | 2 | Verified all 25 source code references against actual file contents and line numbers |
| Code review fix iteration | 1 | Addressed 6 code review findings across documentation content |
| QA fix iteration | 0.5 | Resolved 2 QA findings in documentation |
| Prettier formatting compliance | 0.5 | Applied and verified Prettier v2.6.2 formatting for pre-commit compliance |
| Final validation and commit | 0.5 | Performed final validation pass, confirmed clean working tree, committed all changes |
| **Total Completed** | **26** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human review of documentation accuracy and completeness | 2 | High |
| Stakeholder review and domain-specific feedback incorporation | 1 | Medium |
| PR merge and post-merge verification | 0.5 | Medium |
| **Total Remaining** | **3.5** | |

**Verification:** 26 (completed) + 3.5 (remaining) = 29.5 (total) ✓

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Pre-commit formatting | Prettier v2.6.2 | 1 | 1 | 0 | 100% | All matched files use Prettier code style |
| Pre-commit hygiene | pre-commit-hooks v4.2.0 | 4 | 4 | 0 | 100% | end-of-file-fixer, trailing-whitespace, mixed-line-ending, check-case-conflict |
| Source citation verification | Manual (25 references) | 25 | 25 | 0 | 100% | All file paths and line numbers verified against repository |
| Live API test scenarios | curl / HTTP | 5 | 5 | 0 | 100% | Authenticated, unauthenticated, invalid token, token acquisition, wrong keyword |
| Scope compliance | git diff --name-status | 1 | 1 | 0 | 100% | Only `blitzy/documentation/paperless-ngx_542221a38dff.md` modified; zero source code changes |

All tests originate from Blitzy's autonomous validation pipeline for this project. No external test suites were executed as this is a documentation-only deliverable with no source code changes.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Paperless-ngx development server**: Successfully started with temporary environment configuration for API testing
- ✅ **Database migrations**: Completed successfully with `--skip-checks` flag
- ✅ **Token endpoint (`POST /api/token/`)**: HTTP 200 with valid credentials, returns `{"token":"<40-char-hex>"}`
- ✅ **Document listing endpoint (`GET /api/documents/`)**: HTTP 200 with valid token, returns paginated JSON envelope
- ✅ **Unauthenticated access**: HTTP 401 with `{"detail":"Authentication credentials were not provided."}`
- ✅ **Invalid token access**: HTTP 401 with `{"detail":"Invalid token."}`
- ✅ **Wrong keyword (`Bearer`)**: HTTP 401 with `{"detail":"Authentication credentials were not provided."}`

### UI Verification

Not applicable — this project is a pure API/code investigation document with no UI components. No screenshots required.

### API Integration Outcomes

- ✅ All 5 documented API test scenarios produce the exact responses described in the investigation document
- ✅ Response headers include `X-Api-Version: 2` and `X-Version: 1.7.0` for authenticated requests
- ✅ Pagination envelope (`count`, `next`, `previous`, `results`) present even with zero documents

---

## 5. Compliance & Quality Review

| AAP Deliverable | Status | Quality Check | Notes |
|-----------------|--------|---------------|-------|
| Create `blitzy/documentation/paperless-ngx_542221a38dff.md` | ✅ Pass | File exists, 898 lines, 55,579 bytes | Correct path per SWE-AtlasQnA-Repo rule |
| Answer all 9 explicit questions from AAP §0.7.1 | ✅ Pass | All questions have dedicated sections with direct answers | Coverage: 100% (was 17% before) |
| Address implicit documentation needs (token flow, versioning, auth chain, pagination, error formats) | ✅ Pass | All 5 implicit needs documented | Sections 2, 3, 4, 6, 5 respectively |
| Evidence-based with source code citations | ✅ Pass | 25 source references, all verified | File:line format throughout |
| Thinking/Rationale in every section | ✅ Pass | 12 sections, 12 Thinking/Rationale subsections | Per SWE-AtlasQnA-Repo rule |
| Mermaid diagrams | ✅ Pass | 1 authentication flow diagram at line 528 | Flowchart showing DRF class chain |
| Read-only codebase policy | ✅ Pass | `git diff --name-status` shows only 1 new .md file | Zero source code modifications |
| Cleanup of temporary artifacts | ✅ Pass | Documented in Section 9 | `/tmp/paperless_*` directories removed |
| Prettier v2.6.2 formatting | ✅ Pass | `npx prettier --check` returns clean | All matched files use Prettier code style |
| Pre-commit hook compliance | ✅ Pass | end-of-file-fixer, trailing-whitespace, mixed-line-ending | LF line endings, trailing newline present |
| Live API test verification | ✅ Pass | 5 test scenarios executed and captured | Results documented in Sections 3, 5, 10 |
| Security advisory for dependency CVEs | ✅ Pass | Django 4.0.4 and DRF 3.13.1 CVEs documented | Section 11 |
| Security hardening recommendations | ✅ Pass | 4 production hardening recommendations | Section 12 |

### Fixes Applied During Autonomous Validation

| Fix | Commit | Details |
|-----|--------|---------|
| 6 code review findings | `2e2d1ff` | Addressed content accuracy and completeness issues |
| 2 QA findings | `d16568e` | Resolved documentation quality issues |
| Prettier formatting | `5bc1c13` | Applied prettier --write for table alignment, JSON formatting, italic syntax |
| Security sections added | `d4eaca9` | Added Sections 11–12 for security advisory and hardening |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source code line numbers may drift if Paperless-ngx codebase is updated | Technical | Medium | High | 25-entry source reference table (Section 10) enables targeted re-verification | Documented |
| Django 4.0.4 has 45+ known CVEs including critical SQL injection | Security | Critical | High | Security advisory documented in Section 11; upgrade to >=4.2.28 recommended | Advisory only — read-only policy |
| DRF 3.13.1 has CVE-2024-21520 (XSS in browsable API) | Security | Medium | Medium | Documented in Section 11; upgrade to >=3.15.2 recommended | Advisory only — read-only policy |
| Documentation accuracy depends on human domain expertise review | Operational | Low | Medium | Structured review checklist in recommended next steps; all claims cite source | Pending human review |
| Document not integrated into Sphinx documentation tree | Integration | Low | Low | Standalone markdown works in any viewer; conversion to RST is optional | By design per AAP |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 26
    "Remaining Work" : 3.5
```

**Completed Work: 26 hours | Remaining Work: 3.5 hours | Total: 29.5 hours | 88.1% Complete**

---

## 8. Summary & Recommendations

### Achievements

The project has successfully delivered a comprehensive 898-line investigation document that fully answers all questions defined in the Agent Action Plan. The document covers local setup, token provisioning, authenticated and unauthenticated API behavior, JSON response structure, Django authentication code tracing, custom middleware analysis, security advisories, and production hardening — all with verified source code citations and live test results.

The project is **88.1% complete** (26 completed hours out of 29.5 total hours). All autonomous deliverables defined in the AAP are fully implemented, validated, and committed. The remaining 3.5 hours consist exclusively of human review activities: accuracy verification by domain experts, stakeholder feedback incorporation, and PR merge.

### Remaining Gaps

1. **Human accuracy review (2h)**: While all 25 source citations are verified against the codebase, a domain expert should confirm that the documented behavior matches the intended API contract
2. **Stakeholder review (1h)**: Security recommendations in Sections 11–12 require maintainer acknowledgment
3. **PR merge (0.5h)**: Final merge and post-merge verification

### Critical Path to Production

The document is ready for merge. The critical path is:
1. Human review → Approve or request changes
2. Address any feedback → Update document
3. Merge PR → Document available in repository

### Production Readiness Assessment

The deliverable is production-ready for its intended purpose (standalone reference document). It requires no runtime infrastructure, no deployment pipeline, and no external service dependencies. The only prerequisite for "production" is human review and PR merge.

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.8+ | Runtime for Paperless-ngx Django application |
| pip | Latest | Python package manager |
| Git | 2.x+ | Version control |
| Node.js | 14+ (optional) | For prettier formatting checks |

### Environment Setup

```bash
# 1. Clone the repository and switch to the feature branch
git clone <repository-url>
cd paperless-ngx
git checkout blitzy-c5ef6f57-48b5-44f2-8dfd-b4104e759057

# 2. View the deliverable document
cat blitzy/documentation/paperless-ngx_542221a38dff.md
```

### Viewing the Document

The deliverable is a standalone Markdown file. It can be viewed with:

```bash
# Terminal viewing
cat blitzy/documentation/paperless-ngx_542221a38dff.md

# Or with a pager
less blitzy/documentation/paperless-ngx_542221a38dff.md

# Mermaid diagrams render in GitHub, VS Code, or any Mermaid-compatible viewer
```

### Running the Local Paperless-ngx Server (for API Testing Verification)

To verify the document's API examples against a live server:

```bash
# 1. Install Python dependencies
pip install -r requirements.txt

# 2. Navigate to source directory
cd src

# 3. Set environment variables
export PAPERLESS_DATA_DIR=/tmp/paperless_data
export PAPERLESS_MEDIA_ROOT=/tmp/paperless_media
export PAPERLESS_CONSUMPTION_DIR=/tmp/paperless_consume
export PAPERLESS_LOGGING_DIR=/tmp/paperless_log
export PAPERLESS_SECRET_KEY=testkey123

# 4. Create required directories
mkdir -p $PAPERLESS_DATA_DIR $PAPERLESS_MEDIA_ROOT $PAPERLESS_CONSUMPTION_DIR $PAPERLESS_LOGGING_DIR

# 5. Run database migrations
python3 manage.py migrate --skip-checks

# 6. Create test user and token
python3 manage.py shell -c "
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
user = User.objects.create_superuser('testuser', 'test@example.com', 'testpassword')
token = Token.objects.create(user=user)
print(f'Token: {token.key}')
"

# 7. Start the development server (background)
python3 manage.py runserver 0.0.0.0:8000 &

# 8. Test authenticated request (replace <TOKEN> with the output from step 6)
curl -s http://localhost:8000/api/documents/ \
  -H "Authorization: Token <TOKEN>"

# Expected: {"count":0,"next":null,"previous":null,"results":[]}

# 9. Test unauthenticated request
curl -s http://localhost:8000/api/documents/

# Expected: {"detail":"Authentication credentials were not provided."}
```

### Formatting Verification

```bash
# Verify prettier formatting compliance
npx prettier@2.6.2 --check blitzy/documentation/paperless-ngx_542221a38dff.md
# Expected: "All matched files use Prettier code style!"
```

### Cleanup

```bash
# Stop the development server
kill %1

# Remove temporary directories
rm -rf /tmp/paperless_data /tmp/paperless_media /tmp/paperless_consume /tmp/paperless_log
```

### Troubleshooting

| Issue | Resolution |
|-------|------------|
| `migrate` fails with system check errors | Use `--skip-checks` flag to bypass OCR binary validation |
| `prettier` not found | Install with `npm install -g prettier@2.6.2` |
| Port 8000 already in use | Kill existing process: `lsof -ti:8000 \| xargs kill` |
| `ModuleNotFoundError` on import | Ensure `pip install -r requirements.txt` completed successfully |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose | Working Directory |
|---------|---------|-------------------|
| `cat blitzy/documentation/paperless-ngx_542221a38dff.md` | View the deliverable document | Repository root |
| `npx prettier@2.6.2 --check blitzy/documentation/paperless-ngx_542221a38dff.md` | Verify formatting compliance | Repository root |
| `git diff origin/paperless-ngx_542221a38dff...HEAD --stat` | View all changes in this branch | Repository root |
| `git diff origin/paperless-ngx_542221a38dff...HEAD --name-status` | Verify only expected files changed | Repository root |
| `python3 manage.py migrate --skip-checks` | Run database migrations | `src/` |
| `python3 manage.py runserver 0.0.0.0:8000` | Start development server | `src/` |

### B. Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 8000 | Paperless-ngx Django development server | Used for API testing verification only |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **Deliverable** — Investigation document (898 lines) |
| `src/paperless/settings.py` | DRF authentication configuration (lines 116–121) |
| `src/paperless/urls.py` | API routing — documents endpoint (line 32), token endpoint (line 81) |
| `src/paperless/views.py` | StandardPagination class (lines 8–11) |
| `src/paperless/middleware.py` | ApiVersionMiddleware (lines 5–16) |
| `src/paperless/auth.py` | Custom authentication classes (lines 9–41) |
| `src/documents/views.py` | DocumentViewSet with IsAuthenticated permission (lines 172–200) |
| `src/documents/serialisers.py` | DocumentSerializer field definitions (lines 201–235) |
| `src/paperless/version.py` | Version tuple `(1, 7, 0)` |
| `requirements.txt` | Pinned dependencies (Django 4.0.4 at line 38, DRF 3.13.1 at line 39) |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Paperless-ngx | 1.7.0 | `src/paperless/version.py` |
| Django | 4.0.4 | `requirements.txt` line 38 |
| Django REST Framework | 3.13.1 | `requirements.txt` line 39 |
| Python | 3.8+ | `.readthedocs.yml` |
| Prettier | 2.6.2 | `.pre-commit-config.yaml` |
| Sphinx | ~4.5.0 | `Pipfile` (dev dependency) |

### E. Environment Variable Reference

| Variable | Purpose | Default |
|----------|---------|---------|
| `PAPERLESS_DATA_DIR` | Database and search index storage | `<BASE_DIR>/../data` |
| `PAPERLESS_MEDIA_ROOT` | Document file storage | `<BASE_DIR>/../media` |
| `PAPERLESS_CONSUMPTION_DIR` | Consumption intake directory | `<BASE_DIR>/../consume` |
| `PAPERLESS_LOGGING_DIR` | Application log directory | `<DATA_DIR>/log` |
| `PAPERLESS_SECRET_KEY` | Django secret key | Built-in fallback (insecure) |
| `PAPERLESS_DEBUG` | Enable debug mode | `NO` |
| `PAPERLESS_AUTO_LOGIN_USERNAME` | Auto-login user (single-user mode) | Not set |
| `PAPERLESS_ENABLE_HTTP_REMOTE_USER` | Enable SSO via reverse proxy | Not set |

### G. Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the primary directive document defining project scope and requirements |
| DRF | Django REST Framework — the Python library providing REST API capabilities for Django |
| TokenAuthentication | DRF's built-in authentication class that validates `Authorization: Token <key>` headers |
| Token model | `rest_framework.authtoken.models.Token` — ORM model storing API tokens in `authtoken_token` table |
| StandardPagination | Paperless-ngx's pagination class extending DRF's `PageNumberPagination` with page_size=25 |
| SWE-AtlasQnA-Repo | Implementation rule requiring a new markdown document with thinking/rationale behind answers |
| CVE | Common Vulnerabilities and Exposures — standardized identifier for security vulnerabilities |
| HSTS | HTTP Strict Transport Security — header preventing protocol downgrade attacks |