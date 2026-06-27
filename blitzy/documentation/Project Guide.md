# Blitzy Project Guide — Paperless-ngx API Token Authentication Investigation

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, authoritative, code-grounded Markdown document explaining exactly how Paperless-ngx REST API **token authentication** works, enabling the user to integrate external tooling against the API with confidence. The deliverable answers seven precise questions — how to run an instance and mint a token, the exact auth header (`Authorization: Token <key>`), the complete documents endpoint (`/api/documents/`), the paginated JSON response shape, the verbatim unauthenticated `401` outcome, and the Django/DRF code path — each pairing a conclusion with rationale and `file:line` citations, confirmed empirically against a live **Paperless-ngx v1.7.0** runtime. It is an additive, **documentation-only** task: exactly one new file, **zero** modifications to the Paperless-ngx source tree.

### 1.2 Completion Status

The project is **91.9% complete** on an AAP-scoped, hours-based basis. All autonomous investigation, verification, authoring, and cleanup work is finished and committed; the remaining **3 hours** are the inherent human-side path-to-production (technical review, optional reproduction, and PR merge).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'14px'}}}%%
pie showData title 91.9% Complete (AAP-scoped hours)
    "Completed Work (AI)" : 34
    "Remaining Work" : 3
```

| Metric | Hours |
|---|---|
| **Total Hours** | **37** |
| Completed Hours (AI + Manual) | 34 |
| &nbsp;&nbsp;• Completed by AI (autonomous) | 34 |
| &nbsp;&nbsp;• Completed by Manual (human) | 0 |
| **Remaining Hours** | **3** |
| **Percent Complete** | **91.9%** |

> Completion formula (PA1, AAP-scoped): `34 / (34 + 3) = 34/37 = 91.9%`.

### 1.3 Key Accomplishments

- ✅ Authored a 664-line, citation-dense investigation & integration guide at `blitzy/documentation/paperless-ngx_542221a38dff.md`, answering **all 7** required questions with conclusion + rationale per answer.
- ✅ Established ground truth by tracing the full request path through the Paperless-ngx source **and** the DRF 3.13.1 dependency internals (82 lines carry `file:line` citations).
- ✅ Confirmed every conclusion empirically against the **real Paperless-ngx v1.7.0** application (run from the user-provided Docker image with `DEBUG=False`, Redis, SQLite) via 8 verbatim `curl` probes plus a decisive 401-vs-403 "LINCHPIN" proof.
- ✅ Corroborated the DRF token-auth contract (header keyword `Token`, 401 + `WWW-Authenticate`) against the official DRF documentation.
- ✅ Preserved **byte-for-byte source-tree integrity** — git shows exactly one file added, zero modifications/deletions across `src/**`.
- ✅ Cleaned up all ephemeral artifacts (containers, network, scratch DB, test user/token, host scratch files); working tree is clean.
- ✅ Passed all five autonomous production-readiness gates (citation verification, runtime, zero unresolved errors, in-scope validation, source integrity).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| None identified | No issue blocks release or validation. All 7 questions are answered, all citations verified, all probes pass, and the deliverable is committed. | — | — |

### 1.5 Access Issues

**No access issues identified.** The autonomous workflow had everything required: full repository access, the pinned runtime (`django==4.0.4` + `djangorestframework==3.13.1`), and the user-provided Docker image used for the end-to-end runtime verification. No repository permissions, service credentials, or third-party API access are outstanding.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Repository (`paperless-ngx`) | Read/Write | None — deliverable committed | ✅ Resolved | — |
| User Docker image (runtime) | Pull/Run | None — image pulled & run end-to-end | ✅ Resolved | — |

### 1.6 Recommended Next Steps

1. **[High]** Technical review & sign-off — read `blitzy/documentation/paperless-ngx_542221a38dff.md` and confirm it answers the seven integration questions accurately (≈1.5h).
2. **[Medium]** Approve & merge the documentation PR to the target branch (≈0.5h).
3. **[Low]** Optionally reproduce the runtime probes (Docker image or the pinned-deps SQLite fallback) to confirm the contract first-hand (≈1.0h).
4. **[Low]** *(Beyond this project's scope)* Build the external-tool integration using the document's copy-paste recipe — explicitly excluded from this task per AAP §0.5.2.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| Source-code tracing & ground-truth establishment | 8 | Traced the full request path: `settings.py` authenticator ordering (Basic→Session→Token), `urls.py` routing (`DefaultRouter`, documents register, `^api/`, token route), `documents/views.py` ViewSet + permission + pagination, `paperless/views.py` `StandardPagination`, plus DRF 3.13.1 internals (`authentication.py`, `authtoken/models.py`, `views.py`, `exceptions.py`, `authtoken/views.py`, `authtoken/admin.py`, `drf_create_token`). Established the decisive 401-vs-403 linchpin. |
| Runtime environment provisioning (real Docker stack) | 6 | Stood up **Paperless-ngx v1.7.0** from the user image: Redis sidecar + network, applied migrations (creating `authtoken_token`), created user `apitester`, generated a token, seeded 30 documents, and published `runserver` on port 8000. |
| Empirical verification (8 `curl` probes + LINCHPIN proof) | 4 | Captured verbatim status lines, headers, and bodies for: no-creds→401, valid→200, invalid→401, `POST /api/token/`→200, `?page_size`, bad-password→400, `Bearer`→401, no-trailing-slash→302; isolated the 401-vs-403 flip via the real DRF classes; byte-count cross-checks. |
| Web-search corroboration (DRF 3.13.1 contract) | 1 | Cross-checked the `Authorization: Token <key>` header keyword and the 401 + `WWW-Authenticate` behavior against the official DRF documentation (corroborative only). |
| Document authoring — 7 Q&A answers + supporting sections | 10 | Authored Q1–Q7 (each conclusion + rationale + `[file:line]` citations), plus TL;DR, "How this was verified", a copy-paste integration recipe, caveats/operational notes, a Mermaid decision flowchart, the verbatim appendix, and References (664 lines total). |
| Validation, citation correction & refinement (3 commits) | 4 | Line-by-line citation verification (zero discrepancies); corrected the token-provisioning source trace; fixed `exceptions.py` citation precision (status_code lines); upgraded the empirical evidence to the real v1.7.0 runtime; hygiene checks. |
| Ephemeral cleanup & source-tree integrity | 1 | Removed containers, network, scratch DB, seeded data, test user/token, and host scratch files; verified the 5 reference files are byte-for-byte unchanged (md5) and the working tree is clean. |
| **Total Completed** | **34** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Documentation technical review & sign-off (read the doc; confirm the 7 answers; spot-check `[file:line]` citations against current source) | 1.5 | High |
| Optional independent runtime reproduction (re-run key `curl` probes from the Docker image or the pinned-deps SQLite fallback to confirm the contract) | 1.0 | Low |
| PR approval & merge of the documentation deliverable to the target branch | 0.5 | Medium |
| **Total Remaining** | **3.0** | |

> The actual external-tool integration is **out of scope** (AAP §0.5.2) and is therefore **not** counted in remaining hours; the document enables it via a copy-paste recipe.

### 2.3 Hours Reconciliation

| Quantity | Hours | Source |
|---|---|---|
| Completed (Section 2.1 total) | 34 | Sum of completed components |
| Remaining (Section 2.2 total) | 3 | Sum of remaining categories |
| **Total Project Hours** | **37** | 34 + 3 |
| **Percent Complete** | **91.9%** | 34 / 37 |

---

## 3. Test Results

> **Context for this documentation-only task.** The AAP explicitly excludes any source or test changes, so there are **no in-scope application unit/integration tests** to run or modify. Per the autonomous validation logs, the verification activities that stand in for tests on this QnA task are: (a) line-by-line citation verification and (b) the empirical API probes + LINCHPIN proof. **All test rows below originate from Blitzy's autonomous validation logs for this project.**

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Source-tree citation verification (`src/**`) | Manual line-by-line vs. source | 30+ | 30+ | 0 | 100% | Every `[src/...:L#]` citation verified; zero discrepancies. Spot-check independently re-confirmed 9 key citations. |
| DRF 3.13.1 dependency citation verification | Manual vs. installed dependency | 20+ | 20+ | 0 | 100% | All substantively correct; 1 precision fix applied (`exceptions.py` status_code lines). |
| Empirical API probes (live runtime) | `curl` vs. real Paperless-ngx v1.7.0 | 8 | 8 | 0 | 100% | no-creds→401; valid→200; invalid→401; `POST /api/token/`→200; `?page_size=5`→5; bad-pass→400; `Bearer`→401; no-slash→302. Byte-count cross-checks passed. |
| 401-vs-403 LINCHPIN proof | Real DRF authenticator classes | 1 | 1 | 0 | 100% | Basic-first→`401` (+`WWW-Authenticate`); Session-first→`403` (none). Proven without modifying source. |
| Document hygiene / lint | Structural checks | 1 | 1 | 0 | 100% | LF-only, balanced code fences (34), Mermaid intact, 0 placeholders/TODO, single EOF newline. |
| **Aggregate** | — | **60+** | **60+** | **0** | **100%** | 100% pass rate across all autonomous verification activities. |

---

## 4. Runtime Validation & UI Verification

**Runtime validation** was performed end-to-end against the real Paperless-ngx v1.7.0 application (Python 3.9.23, `django==4.0.4`, `djangorestframework==3.13.1`, Redis sidecar, SQLite, `DEBUG=False`):

- ✅ **Operational** — Application boots; migrations applied (`authtoken_token` table created); server serves on port 8000.
- ✅ **Operational** — Token provisioning: `drf_create_token apitester` → `Generated token <40-hex> for user apitester`.
- ✅ **Operational** — Authenticated read: `GET /api/documents/` + `Authorization: Token <key>` → **200**, body `{count, next, previous, results}`, 25 items/page, `count=30`, `X-Version: 1.7.0`.
- ✅ **Operational** — Unauthenticated read: `GET /api/documents/` (no creds) → **401**, `WWW-Authenticate: Basic realm="api"`, body `{"detail":"Authentication credentials were not provided."}`.
- ✅ **Operational** — Invalid token → **401** `{"detail":"Invalid token."}`; `Bearer` scheme → **401**; missing trailing slash → **302**.
- ✅ **Operational** — Token exchange: `POST /api/token/` (valid creds) → **200** `{"token":"<40-hex>"}` (same key as `drf_create_token`); bad creds → **400**.
- ✅ **Operational** — Page-size control: `?page_size=5` → 5 items.

**API integration outcomes:** All probed behaviors match the code reading exactly; the decisive 401-vs-403 outcome was proven against the real DRF classes.

**UI verification:** Not applicable — there is **no UI in scope**. The deliverable is a Markdown document; its rendering was validated structurally (22 headings, 1 Mermaid flowchart, 34 balanced code fences, no placeholders).

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and governing-rule (SWE-AtlasQnA-Repo) constraints to their verification status:

| AAP / Rule Requirement | Benchmark | Status | Evidence |
|---|---|---|---|
| Q1 — Run instance + create user + token | Answered w/ rationale + citations + runtime | ✅ Pass | Doc Q1 + appendix (token `a1b6fe70…`) |
| Q2 — Exercise authenticated read | Answered + empirical 200 | ✅ Pass | Doc Q2 + PROBE 2 |
| Q3 — EXACT token header | `Authorization: Token <40-hex>` | ✅ Pass | Doc Q3 (`keyword='Token'`) + web corroboration |
| Q4 — COMPLETE endpoint path | `/api/documents/` (trailing slash) | ✅ Pass | Doc Q4 + PROBE 8 (302 w/o slash) |
| Q5 — JSON response shape | Paginated `{count,next,previous,results}`, 25/page | ✅ Pass | Doc Q5 + PROBE 2/5 |
| Q6 — Unauthenticated outcome (verbatim) | 401 + body + `WWW-Authenticate` | ✅ Pass | Doc Q6 + PROBE 1 + LINCHPIN |
| Q7 — Trace Django code path | `TokenAuthentication` + `Token` model | ✅ Pass | Doc Q7 (`authtoken_token`) |
| Code as the source of truth | `[file:line]` for every claim | ✅ Pass | 82 citation lines; zero discrepancies |
| Rationale / "thinking" per answer | Conclusion + reasoning | ✅ Pass | Every Q has an Answer + Rationale |
| Empirical verification (build & run) | Live probes captured verbatim | ✅ Pass | Appendix: 8 probes + LINCHPIN |
| Web-search corroboration | DRF 3.13.1 contract cross-checked | ✅ Pass | References; independently re-confirmed |
| Deliverable placement & filename | `blitzy/documentation/<branch>.md` | ✅ Pass | File committed at HEAD |
| Do NOT modify source | Zero `src/**` edits | ✅ Pass | `git diff` = 0 changes; md5 match |
| No other code added to source repo | Only the one Markdown file | ✅ Pass | `blitzy/` holds only the deliverable |
| Ephemeral cleanup | No leftover artifacts | ✅ Pass | Containers/DB/files removed; tree clean |

**Fixes applied during autonomous validation:** (1) corrected the token-provisioning source trace and citations; (2) fixed `exceptions.py` citation precision (status_code lines); (3) upgraded empirical evidence from a minimal reproduction to the real v1.7.0 Docker runtime; (4) added probes for the `Bearer` scheme and the missing trailing slash. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Findings are pinned to `django==4.0.4` + `djangorestframework==3.13.1` and the current authenticator ordering; a future upgrade/settings change could alter behavior | Technical | Low | Low | Document states version-specificity prominently; re-verify after any dependency/settings bump | Mitigated |
| Citation line numbers could drift if the source changes later | Technical | Low | Medium (long-term) | Citations are file+line and version-anchored; re-validate on source upgrades | Open (informational) |
| DRF tokens are non-expiring, stored unhashed, and incur a DB lookup per request; an example token appears in the doc | Security | Low | Low | Caveats cover non-expiry / DB-lookup / treat-as-password; the example token was ephemeral and its container was destroyed | Mitigated |
| New attack surface introduced | Security | None | N/A | Additive documentation only — zero code, dependencies, endpoints, or config added to the source | N/A |
| End-to-end reproduction depends on the user-provided Docker image | Operational | Low | Low | Contract is faithfully reproducible with just the pinned `django`+`DRF` on SQLite (documented fallback) | Mitigated |
| Runtime footprint (monitoring/health checks) | Operational | None | N/A | Documentation deliverable has no runtime footprint | N/A |
| The actual external-tool integration is not built | Integration | Medium | N/A | Explicitly out of scope (AAP §0.5.2); the doc provides a copy-paste recipe + caveats to enable it | By design |
| Under Apache/`mod_wsgi`, the `Authorization` header may be stripped unless `WSGIPassAuthorization On` | Integration | Low | Low | Noted as a deployment caveat; Paperless ships gunicorn, so probability is low | Informational |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'14px'}}}%%
pie showData title Project Hours Breakdown (Total 37h)
    "Completed Work" : 34
    "Remaining Work" : 3
```

**Remaining work by category** (hours, from Section 2.2 — sums to 3h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#5B39F3','pieOuterStrokeColor':'#5B39F3','pieSectionTextColor':'#1A1A1A','pieTitleTextSize':'15px'}}}%%
pie showData title Remaining Work by Category (3h)
    "Review & sign-off [High]" : 1.5
    "Optional reproduction [Low]" : 1.0
    "PR merge [Medium]" : 0.5
```

> **Integrity check:** "Remaining Work" = **3h**, identical to Section 1.2 (Remaining Hours), the Section 2.2 total, and the human-task total in Section 8.

---

## 8. Summary & Recommendations

**Achievements.** The project fulfilled its single objective: a comprehensive, code-grounded, empirically-verified Markdown document that answers all seven questions about Paperless-ngx API token authentication. Every conclusion is backed by `file:line` citations against the pinned source **and** verbatim runtime evidence from the real Paperless-ngx v1.7.0 application. The headline contract is unambiguous: send `Authorization: Token <40-hex-key>` to `GET /api/documents/` (trailing slash required); a valid token yields a **200** with a paginated `{count, next, previous, results}` envelope (25 items/page); no credentials yield **401** with `WWW-Authenticate: Basic realm="api"`; and the flow is handled by `TokenAuthentication` backed by the `Token` model in the `authtoken_token` table.

**Remaining gaps & critical path to production.** No technical gaps remain in the autonomous work. The critical path is purely human acceptance: **review → (optional) reproduce → merge** (3 hours total). The actual external-tool integration is intentionally out of scope.

**Production-readiness assessment.** The deliverable is **production-ready**: it is accurate (zero citation discrepancies), complete (all 7 questions + integration recipe + caveats), self-contained, and committed, with byte-for-byte source-tree integrity preserved. At **91.9% complete**, the only work outstanding is the inherent human review/acceptance that no autonomous agent can perform.

| Success Metric | Target | Actual |
|---|---|---|
| AAP questions answered | 7 / 7 | ✅ 7 / 7 |
| Citation accuracy | 100% | ✅ 100% (zero discrepancies) |
| Empirical probes passing | All | ✅ 8 / 8 + LINCHPIN |
| Source files modified | 0 | ✅ 0 |
| Ephemeral artifacts left behind | 0 | ✅ 0 |
| AAP-scoped completion | ≥ 90% | ✅ 91.9% |

---

## 9. Development Guide

> All commands below were tested in the project environment (git 2.51.0, Python 3.13.7, Docker 28.5.2, curl 8.14.1). Inspection/integrity commands run as-is; the runtime-reproduction recipe requires the user-provided Docker image (or the pinned-deps fallback).

### 9.1 System Prerequisites

- **git** ≥ 2.x (repository access)
- **A Markdown viewer** (to read the deliverable) — VS Code, GitHub, `glow`, or any browser preview
- *For optional runtime reproduction:* **Docker** ≥ 28.x (preferred) **or** **Python 3.9** + `venv` (pinned-deps fallback), plus **curl**
- ~2 GB free disk for the Docker image

### 9.2 Accessing the Deliverable (primary workflow)

```bash
# From the repository root, on the project branch:
git checkout blitzy-683f9c5b-20d9-4bab-9ff4-7cac00fb772c

# Confirm the single deliverable exists:
test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo "OK: deliverable present"

# Navigate its structure (expect 22 headings):
grep -nE '^#{1,3} ' blitzy/documentation/paperless-ngx_542221a38dff.md

# Open it in your preferred Markdown viewer, e.g.:
less blitzy/documentation/paperless-ngx_542221a38dff.md
```

### 9.3 Verifying Source-Tree Integrity

```bash
# Expect 0 — the task modified no source files:
git diff --name-only 542221a38..HEAD -- src/ | wc -l

# Confirm exactly one file was added by the agent:
git diff --name-status 542221a38..HEAD
# -> A    blitzy/documentation/paperless-ngx_542221a38dff.md

# Confirm provenance (3 agent commits):
git log --author="agent@blitzy.com" --oneline 542221a38..HEAD
```

### 9.4 (Optional) Reproduce the Runtime Verification — Preferred (Docker)

```bash
# 1) Network + Redis sidecar
docker network create pl-net
docker run -d --name pl-redis --network pl-net redis:7-alpine

# 2) App container from the user-provided image (sleep entrypoint so we drive it manually)
docker run -d --name pl-app --network pl-net \
  -e PAPERLESS_REDIS=redis://pl-redis:6379 -p 8000:8000 \
  --entrypoint sleep <USER_IMAGE> infinity

# 3) Prepare dirs, apply migrations (creates authtoken_token), create user + token
docker exec pl-app bash -c "mkdir -p /app/consume /app/media /app/data"
docker exec pl-app bash -c "cd /app/src && python3 manage.py migrate --noinput"
docker exec pl-app bash -c "cd /app/src && python3 manage.py drf_create_token apitester"

# 4) Start the server (insecure flag serves without collectstatic)
docker exec -d pl-app bash -c "cd /app/src && python3 manage.py runserver 0.0.0.0:8000 --noreload --insecure"
```

### 9.5 (Optional) Reproduce the Runtime Verification — Fallback (pinned deps)

```bash
python3.9 -m venv .venv && source .venv/bin/activate
pip install "django==4.0.4" "djangorestframework==3.13.1"
# Mirror the Paperless REST_FRAMEWORK authenticator order (Basic -> Session -> Token),
# an IsAuthenticated documents ViewSet, and StandardPagination(page_size=25).
```

### 9.6 Verification Steps (the key probes)

```bash
# No credentials -> 401 + WWW-Authenticate
curl -i http://localhost:8000/api/documents/

# Valid token -> 200 + paginated envelope (25 items)
curl -i -H "Authorization: Token <40-hex-key>" http://localhost:8000/api/documents/

# Exchange credentials for a token -> 200 {"token":"<40-hex-key>"}
curl -i -X POST -d 'username=apitester&password=<password>' http://localhost:8000/api/token/
```

Expected: `401` with `{"detail":"Authentication credentials were not provided."}` for the first; `200` with `{"count":…,"next":…,"previous":…,"results":[…]}` for the second; `200` with `{"token":…}` for the third.

### 9.7 Cleanup (after optional reproduction)

```bash
docker rm -f pl-app pl-redis
docker network rm pl-net
# deactivate && rm -rf .venv   # if you used the fallback
```

### 9.8 Troubleshooting

- **Trailing slash required** — `GET /api/documents` (no slash) → `302` to the login page; always use `/api/documents/`.
- **Use `Token`, not `Bearer`** — only the `Token` keyword is recognized; `Bearer` → `401` (treated as no credentials).
- **Missing `authtoken_token` table** — run `manage.py migrate` before generating/using tokens.
- **`401 {"detail":"Invalid token."}`** — the token is malformed/unknown; re-mint via `drf_create_token` or `POST /api/token/`.
- **Header stripped behind Apache/`mod_wsgi`** — set `WSGIPassAuthorization On`.
- **403 instead of 401** — would only occur if `SessionAuthentication` were listed first; Paperless lists `BasicAuthentication` first, so the result is `401`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Confirm deliverable present | `test -f blitzy/documentation/paperless-ngx_542221a38dff.md && echo OK` |
| List document headings | `grep -nE '^#{1,3} ' blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Source-integrity check (expect 0) | `git diff --name-only 542221a38..HEAD -- src/ \| wc -l` |
| Agent provenance | `git log --author="agent@blitzy.com" --oneline 542221a38..HEAD` |
| Generate token (server) | `python3 manage.py drf_create_token apitester` |
| Authenticated read | `curl -i -H "Authorization: Token <key>" http://localhost:8000/api/documents/` |
| Token exchange | `curl -i -X POST -d 'username=U&password=P' http://localhost:8000/api/token/` |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 8000 | Paperless-ngx web/API | Published by `runserver 0.0.0.0:8000` (reproduction only) |
| 6379 | Redis | `django-q` task queue / Channels (reproduction sidecar) |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (664 lines) |
| `src/paperless/settings.py` | REST_FRAMEWORK auth ordering [L116-L121]; `authtoken` in INSTALLED_APPS [L108] |
| `src/paperless/urls.py` | `DefaultRouter` [L29]; documents register [L32]; `^api/` [L40]; token route [L81] |
| `src/documents/views.py` | `DocumentViewSet` [L172]; pagination [L182]; `IsAuthenticated` [L183] |
| `src/paperless/views.py` | `StandardPagination` page_size 25 / cap 100000 [L8-L11] |
| `requirements.txt` | Pinned `django==4.0.4` [L38], `djangorestframework==3.13.1` [L39] |

### D. Technology Versions

| Component | Version |
|---|---|
| Paperless-ngx | v1.7.0 |
| Python (runtime) | 3.9.x (Docker image) |
| Django | 4.0.4 |
| Django REST Framework | 3.13.1 |
| Datastore (reproduction) | SQLite (default); Redis sidecar |
| Tooling (this environment) | git 2.51.0, Python 3.13.7, Docker 28.5.2, curl 8.14.1 |

### E. Environment Variable Reference

| Variable | Purpose | Used In |
|---|---|---|
| `PAPERLESS_REDIS` | Redis connection URL for the app container | Runtime reproduction |
| `PAPERLESS_ENABLE_HTTP_REMOTE_USER` | Enables optional remote-user auth (off by default; does **not** alter the documented contract) | Source config (`settings.py` L201, L207-L215) |
| `DEBUG` | When `False` (production), authenticators are exactly Basic→Session→Token | Source config |

### F. Developer Tools Guide

- **Markdown preview:** VS Code (`Ctrl+Shift+V`), GitHub rendering, or `glow`/`mdcat`.
- **Mermaid rendering:** GitHub and VS Code (with a Mermaid extension) render the Q6 decision flowchart and the status pies in this guide.
- **API exploration:** `curl -i` (shown above) or `httpie` (`http GET :8000/api/documents/ "Authorization: Token <key>"`).
- **Docker:** `docker logs pl-app` to inspect server output during reproduction.

### G. Glossary

| Term | Meaning |
|---|---|
| **Token (DRF)** | A 40-character hex key stored in `authtoken_token`, one per user (OneToOne), non-expiring by default. |
| **`Authorization: Token <key>`** | The exact header an integrator sends; keyword is the literal `Token`. |
| **Paginated envelope** | The DRF response `{count, next, previous, results}`; 25 items/page by default, `?page_size=` adjustable (cap 100000). |
| **401 vs 403 linchpin** | The unauthenticated status is `401` because `BasicAuthentication` is listed first and returns a truthy `WWW-Authenticate` value; a `None` (e.g., Session first) would coerce to `403`. |
| **LINCHPIN proof** | Exercising the real DRF authenticator classes to demonstrate the 401-vs-403 flip without modifying source. |
| **AAP** | Agent Action Plan — the governing specification for this task. |