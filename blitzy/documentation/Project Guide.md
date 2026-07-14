# Blitzy Project Guide

## Paperless-ngx — Memory-Usage Investigation for Document Import / Metadata Handling

> **Task type:** Read-only investigation & documentation (Rules: `SWE-AtlasQnA-Repo`)
> **Sole deliverable:** `blitzy/documentation/paperless-ngx_542221a38dff.md`
> **Branch:** `blitzy-d8807159-272a-4a73-a584-5ff7a6e332ea` · **Base:** `542221a38dff` (Merge PR #792) · **HEAD:** `728cfbf05`

---

## 1. Executive Summary

### 1.1 Project Overview

This project investigates, reproduces, and explains an anomalous memory-usage pattern in the Paperless-ngx document-processing system, concentrated in the document import / metadata handling paths. The audience is the maintaining engineering team and the reporting stakeholder. The business impact is diagnostic clarity: it explains why memory spikes appear disproportionate to document size, why they seem inconsistent, and why memory is "not released promptly," classifying the behavior as normal allocator arena retention rather than a leak. The technical scope is a strictly read-only, runtime-first investigation of the Django ORM/serialization import-export paths, delivered as one evidence-grounded Markdown answer document with complete, unedited runtime measurements and `file:line` grounding for every claim.

### 1.2 Completion Status

The completion percentage is calculated using the AAP-scoped, hours-based (PA1) methodology: `Completed Hours / (Completed Hours + Remaining Hours) × 100 = 58 / 62 = 93.5%`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData title Completion Status — 93.5% Complete
    "Completed Work (Dark Blue)" : 58
    "Remaining Work (White)" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | 62 |
| **Completed Hours (AI + Manual)** | 58 |
| &nbsp;&nbsp;↳ Blitzy autonomous (AI) | 58 |
| &nbsp;&nbsp;↳ Manual (human) to date | 0 |
| **Remaining Hours** | 4 |
| **Percent Complete** | **93.5%** |

> **Color legend:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF`.

### 1.3 Key Accomplishments

- ✅ **Root cause identified and proven (O1):** Whole-corpus manifest materialization in `document_exporter.dump()` and `document_importer.handle()`; peak memory = document_count × `Document.content` size, not physical file size.
- ✅ **Unnecessary copies quantified (O2):** ~240.7 MB transient JSON-string + Python-list co-residency at 5000 text-heavy docs; importer retains `self.manifest` concurrently with `loaddata`.
- ✅ **Caching hypothesis resolved (O3):** Confirmed **no** `caching.py` module exists; accumulation is a bounded `dateparser` locale warm-up plus instance/train-time classifier vocabulary.
- ✅ **Differential + sensitivity established (O4, O5):** ~9.5× export memory at equal doc count (text-heavy vs metadata-only); ~5.01× linear growth batch 1000→5000; Unicode manifest ~2.2× ASCII.
- ✅ **"Not released" classified (O6):** Normal CPython/glibc arena retention (not a leak) — `malloc_trim(0)` reclaims ~485 MB; CLI exit reclaims all.
- ✅ **Runtime-first evidence:** All measurements from real `manage.py document_exporter` / `document_importer` invocations in the canonical Docker image (Python 3.9.23 + full pinned stack), each confirmed stable across ≥2 runs (6-run stdev 0.52 MB).
- ✅ **Read-only mandate honored:** `src/` diff = 0 lines; only the answer document added; working tree clean.
- ✅ **Scope exceeded:** The AAP's deferred end-to-end OCR consume limitation was actually addressed via canonical-container OCR reproduction (single-doc ~219 MB; 1-vs-4 concurrency).
- ✅ **Fully validated:** All 5 autonomous validation gates passed with zero defects and zero fixes required.

### 1.4 Critical Unresolved Issues

There are **no critical unresolved issues** that block the deliverable. The single item below is a normal path-to-production human step, not a defect.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Domain-expert sign-off on the analysis & claims not yet performed | Formal acceptance pending; deliverable is complete and independently validated | Maintainer / Reviewer | ~2h |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| `ghcr.io/scaleapi/swe-atlas` canonical Docker image | Registry pull | Needed **only** for an optional independent re-run of the reproduction harness; **not** required to read or accept the deliverable (evidence is self-contained with unedited output + `file:line` anchors) | Open (optional) | Reviewer |

> No service credentials, API keys, or repository permissions are required to consume the deliverable. Aside from the optional re-run image above, **no access issues identified**.

### 1.6 Recommended Next Steps

1. **[High]** Domain-expert technical review of the answer document — verify the root-cause reasoning (peak = corpus_count × `Document.content`), the O1–O6 answers, and spot-check `file:line` citations. (~2h)
2. **[Medium]** (Optional) Independently re-run 1–2 key probes in the canonical container to confirm the headline numbers. (~1h)
3. **[Medium]** Stakeholder acceptance and a decision on whether to open a **separate** engineering task to implement the documented optimizations for large-corpus installations. (~1h)
4. **[Low]** If pursued as a new task: implement streaming export/import (`QuerySet.iterator()` / `.values()` / streaming serialization) and memory guardrails — **out of scope for this read-only deliverable.**

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every row traces to an AAP requirement (O1–O6, methodology, deliverable) or an in-scope path-to-production activity. **Total = 58h (matches Completed Hours in Section 1.2).**

| Component | Hours | Description |
|-----------|-------|-------------|
| Root-cause code investigation & `file:line` anchoring (O1, O2) | 6 | Traced import/export paths; pinned exporter `:127-130`, importer `:73/:87`, `models.py:117-124`; distinguished transient copies from retained references |
| Runtime measurement harness authoring | 7 | Wrote `populate.py`, `export_probe.py`, loaddata/dateparser/consume-tree probes using stdlib `tracemalloc` / `/proc/self/statm` / `gc` |
| Canonical Docker provisioning & pinned-stack verification | 4 | Provisioned Python 3.9.23 + pinned stack (django 4.0.4, django-q 1.3.9, sklearn 1.0.2, dateparser 1.1.1, …), redis, native tools; `migrate` + `check` (0 issues) |
| Export/import measurements at scale, ≥2 runs (O4, O5, O6) | 6 | zero / meta5000 / text200 / text1000 / text5000 / uni5000 ×2 runs + importer round-trip + before/during/after capture |
| Caching evaluation (O3) | 3 | Confirmed no `caching.py`; measured bounded `dateparser` warm-up (2 cases × 2 runs); classifier instance/train-time vocabulary |
| GC / allocator classification (O6) | 2 | `del`+`gc.collect()` vs `malloc_trim(0)` vs process-exit; classified arena retention (not a leak) against explicit leak criteria |
| Canonical OCR consume reproduction (F-04, F-05; **beyond AAP scope**) | 4 | Real scanned-PDF + PNG consume (~219 MB single-doc) and 1-vs-4 concurrency; addressed the AAP's deferred limitation |
| Answer-document authoring (13 sections, 3,619 lines) | 12 | Direct answer, environment/method, complete unedited output, per-method attribution, differential, sensitivity, stability, classification, caching |
| Claim-status matrix, coverage pass & canonical labeling (§10–§12A) | 3 | 24 claims classified Source/Observed/Derived/Inferred; every O1–O6 sub-question mapped; canonical vs non-canonical labeling |
| Appendix reproduction harness (§13) | 3 | Full provisioning steps, probe sources, cleanup — executable-as-written |
| QA revision cycles (4 commits, 34 findings resolved) | 8 | Rewrite with canonical evidence (19 findings), executable harness (F1), Report-7 findings (15) |
| **Total** | **58** | |

### 2.2 Remaining Work Detail

Each row is a path-to-production human step. **Total = 4h (matches Remaining Hours in Section 1.2 and Section 7 pie).**

| Category | Hours | Priority |
|----------|-------|----------|
| Domain-expert technical review of analysis & claims | 2 | High |
| Independent reproduction-harness re-run (canonical container) | 1 | Medium |
| Stakeholder acceptance & remediation-follow-up decision | 1 | Medium |
| **Total** | **4** | |

> **Out-of-scope future work (NOT counted in the 4h above; would be a separate task):** implement streaming export/import (~16–24h) and memory guardrails/telemetry (~8–16h). These are the optimizations the read-only mandate explicitly excludes.

### 2.3 Hours Reconciliation

- **Rule 2 check:** Section 2.1 (58h) + Section 2.2 (4h) = **62h** = Total Project Hours in Section 1.2. ✅
- **Rule 1 check:** Remaining hours = **4h** identical in Section 1.2, Section 2.2, and the Section 7 pie chart. ✅
- **Completion:** 58 / 62 = **93.5%**, used consistently in Sections 1.2, 7, and 8. ✅

---

## 3. Test Results

> **Nature of "tests" for this task:** This is a read-only documentation deliverable, so **no code unit tests were authored** (the `src/` diff is 0 lines). The entries below are **Blitzy's autonomous deliverable-claim verification** — the checks the autonomous validation system executed against the deliverable and the canonical runtime. **All entries originate from Blitzy's autonomous validation logs for this project** (Integrity Rule 3).

| Test Category | Framework | Total Checks | Passed | Failed | Coverage % | Notes |
|---------------|-----------|--------------|--------|--------|------------|-------|
| Citation resolution — distinct `src` `file:line` | Blitzy Autonomous Validation | 17 | 17 | 0 | 100% | All resolve in-bounds and are semantically accurate vs the working tree |
| Citation resolution — bare-basename prose | Blitzy Autonomous Validation | 75 | 75 | 0 | 100% | Prose references cross-checked against source |
| Canonical runtime — export peak RSS | Blitzy Autonomous Validation (`/usr/bin/time -v`) | 3 | 3 | 0 | 100% | meta5000 / text1000 / text5000 reproduced within 0.2–3% (glibc arena granularity) |
| Canonical runtime — manifest byte-sizes | Blitzy Autonomous Validation | 3 | 3 | 0 | 100% | Byte-**exact** (4,665,956 / 50,729,956 / 253,665,956 B) |
| Canonical runtime — tracemalloc / gc | Blitzy Autonomous Validation | 3 | 3 | 0 | 100% | tm_peak 19.3 / 148.5 / 731.6 MB — byte-identical to doc |
| Deterministic runtime claims | Blitzy Autonomous Validation | 4 | 4 | 0 | 100% | `DATE_REGEX` tokenization; `FORMAT_VERSION=7`; per-instance vectorizer; `pickle.load` count = 7 |
| Stability / inconsistency (§7) | Blitzy Autonomous Validation | 6 runs | 6 | 0 | 100% | text1000 ×6 → tm_peaks mean 147.35, stdev 0.52 — deterministic per corpus |
| Round-trip integrity (export→import) | Real `manage.py` commands (Exit 0) | 2 | 2 | 0 | 100% | 5000 & 1000 `Document` rows restored in fresh migrated DBs |
| "Not released" reproduction (O6) | Blitzy Autonomous Validation | 1 | 1 | 0 | 100% | `malloc_trim(0)` reclaimed 484.8 MB; heap freed but RSS held until trim |
| Markdown structural integrity | Blitzy Autonomous Validation | — | Pass | 0 | 100% | 132 balanced code fences; 14 pipe-tables with consistent columns |
| Placeholder / stub scan | Blitzy Autonomous Validation | — | Pass | 0 | 100% | Zero TODO/FIXME/stub; scan hits are verbatim stdout / labeled excerpts |
| **Aggregate** | | **≈124 discrete checks** | **All Passed** | **0** | **100%** | Zero defects; zero fixes required |

---

## 4. Runtime Validation & UI Verification

There is **no UI component** in scope (frontend `src-ui/` is explicitly out of scope). Runtime validation covers the canonical management-command entry points that the deliverable exercises.

- ✅ **Operational** — `python manage.py document_exporter <target> --no-progress-bar` runs end-to-end (Exit 0) in the canonical Docker image (Python 3.9.23 + full pinned stack).
- ✅ **Operational** — `python manage.py document_importer <target> --no-progress-bar` runs end-to-end (Exit 0): `Installed 5001/1001 object(s) from 1 fixture(s)`.
- ✅ **Operational** — Export→import round-trip integrity: 5000 and 1000 `Document` rows restored into fresh migrated databases.
- ✅ **Operational** — `python manage.py migrate --no-input` (Exit 0) and `python manage.py check` (0 issues) in the provisioned container.
- ✅ **Operational** — Redis broker reachable (`PONG`) for the Django-Q context referenced in the analysis.
- ✅ **Operational** — "Not released promptly" (O6) reproduced: after `del`+`gc.collect()` the Python heap frees (tracemalloc current → ~1.4 MB) while RSS stays high until `malloc_trim(0)` returns ~485 MB — allocator arena retention, not a leak.
- ⚠ **Partial (host)** — The host interpreter (Python 3.13.7) **cannot** run the pinned Django 4.0.4 (the removed `cgi` module, PEP 594). This is expected and is why all canonical measurements were taken in the Docker runtime; the limitation is explicitly labeled in the deliverable.
- ❌ **Failing** — None.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and `SWE-AtlasQnA-Repo` rules to their verification status.

| Requirement / Rule | Benchmark | Status | Progress | Evidence / Fixes Applied |
|--------------------|-----------|--------|----------|--------------------------|
| Deliverable naming & location | `blitzy/documentation/<source_branch>.md` | ✅ Pass | 100% | `blitzy/documentation/paperless-ngx_542221a38dff.md` present (3,619 lines) |
| Read-only mandate | No source file modified; only answer doc added | ✅ Pass | 100% | `git diff 542221a38..HEAD --name-status` → single `A` line; `src/` diff = 0 |
| Run-first rule | Build & run before writing; capture real output | ✅ Pass | 100% | §3 contains complete unedited `/usr/bin/time -v` output from real `manage.py` runs |
| Magnitude / scale rule | Run at scale; state scale; ≥2 runs | ✅ Pass | 100% | Batches 200/1000/5000; each ≥2 runs; scale stated |
| Inconsistency rule | Repeat same input; report distribution | ✅ Pass | 100% | §7 — text1000 ×6, stdev 0.52 MB; deterministic per corpus |
| Canonical-observation rule | Real entry point; label non-canonical | ✅ Pass | 100% | Real `document_exporter`/`document_importer`; §10 canonical/non-canonical labeling |
| Complete-output rule | Full unedited output + exact command | ✅ Pass | 100% | §3 / §13 show commands and full output |
| Grounding rule | Every claim → `file:line` or observed output | ✅ Pass | 100% | 17 distinct `file:line` citations; §11 claim-status matrix (S/O/D/I) |
| Coverage rule | Address every named sub-question + final pass | ✅ Pass | 100% | §12 coverage pass maps O1–O6 and every sub-question |
| Cleanup obligation | Temp scripts removed; `git status` clean | ✅ Pass | 100% | Working tree clean; scratch confined to `/tmp` |
| O1 Root cause | Identify responsible components/paths | ✅ Pass | 100% | §1 — manifest materialization; exporter `:127-130`, importer `:73/:87` |
| O2 Copies / references | Determine unnecessary copies / retained refs | ✅ Pass | 100% | ~240.7 MB transient duplicate; retained `self.manifest` + concurrent `loaddata` |
| O3 Caching accumulation | Determine caching behavior | ✅ Pass | 100% | §9 — no `caching.py`; bounded `dateparser`; instance/train-time classifier |
| O4 Differential | Spike vs non-spike difference | ✅ Pass | 100% | §5 — aggregate `Document.content`; ~9.5× at equal doc count |
| O5 Sensitivity | Document type & batch size | ✅ Pass | 100% | §6 — ~5.01× linear; Unicode ~2.2×; consume concurrency |
| O6 Runtime evidence & classification | Measurements, named methods, GC classification | ✅ Pass | 100% | §3/§7/§8 — before/during/after; arena retention not leak |

**Fixes applied during autonomous validation:** None required — the deliverable was already accurate, reproducible, complete, and read-only compliant. Validation confirmed production-readiness through independent canonical reproduction rather than repair. **Outstanding compliance items:** None.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Run-to-run RSS variance 0.2–3% (glibc arena granularity) could complicate exact-number comparison | Technical | Low | Medium | Deliverable labels the variance; tracemalloc and manifest byte-sizes are byte-exact for comparison | Mitigated |
| Inferred (I) mechanism claims (arena retention, C-extension locale structures) not directly instrumented | Technical | Low | Low | §11 flags each inferred claim; supported by observed facts (tm_current→~0, trim-reclaim, exit-reclaim) | Mitigated |
| Application regression from the change | Technical | None | None | Read-only: `src/` diff = 0 lines; no code/config/dependency touched | Closed |
| New attack surface introduced | Security | None | None | No code or dependencies added; documentation only | Closed |
| Secrets / credentials leaked in the deliverable | Security | None | None | Grep-verified: no passwords/keys/tokens; matches are technical prose only | Closed |
| Underlying memory amplification (peak = corpus_count × `Document.content`) in CLI export/import unremediated — could OOM on very large corpora | Operational | Medium | Medium | **Out of scope by mandate (explain-only).** Deliverable provides a remediation roadmap (`iterator()`/`.values()`/streaming/worker-recycle). Nuance: the async consume path is already bounded by default `Q_CLUSTER` `recycle: 1` (worker recycled per task, `settings.py:452`) | Open by design (separate follow-up task) |
| Independent canonical reproduction requires the canonical Docker image + native tools | Integration | Medium | Medium | §13 documents full provisioning; findings are independently verifiable from `file:line` + shown unedited output **without** re-running | Open (optional, human) |

**Overall risk posture:** Dominated by Low / None. Because the change is read-only, all code, security, and regression risk to the application is eliminated. The only genuinely open items are a human acceptance decision (Op1) and an optional independent re-run (I1).

---

## 7. Visual Project Status

**Project Hours Breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 58
    "Remaining Work" : 4
```

> **Integrity check:** "Remaining Work" = **4** equals Remaining Hours in Section 1.2 and the sum of the Section 2.2 "Hours" column. "Completed Work" = **58** equals Completed Hours in Section 1.2 and the sum of the Section 2.1 "Hours" column.

**Remaining Work by Priority** (from Section 2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Remaining Hours by Priority
    "High" : 2
    "Medium" : 2
```

**Remaining Work by Category (hours):**

| Category | Hours | Bar |
|----------|-------|-----|
| Domain-expert technical review | 2 | ██████████ |
| Independent reproduction re-run | 1 | █████ |
| Stakeholder acceptance & decision | 1 | █████ |
| **Total** | **4** | |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, runtime-grounded answer to the reported memory anomaly. It proves that the spike originates from whole-corpus manifest materialization in the export/import path — where peak memory scales with document_count × `Document.content` size rather than physical file size — and it explains the "disproportionate," "inconsistent," and "not released promptly" symptoms with unedited measurements captured through the canonical `manage.py` entry points. The caching hypothesis is resolved (no cache module; bounded `dateparser` warm-up), the spike/non-spike differential and document-type/batch-size sensitivity are quantified, and the delayed-release behavior is classified as normal CPython/glibc allocator arena retention — not a leak.

**Remaining gaps.** No analytical gaps remain. The outstanding work is human path-to-production only: technical review, an optional independent re-run, and a stakeholder acceptance/decision — totaling **4 hours**.

**Critical path to production.** (1) Domain-expert review → (2) acceptance → (3) decision on whether to open a **separate** optimization task. The read-only mandate means nothing needs to be merged into `src/`; the deliverable is the product.

**Success metrics.** All 5 autonomous validation gates passed with zero defects; every citation resolves and is accurate; canonical measurements reproduced (RSS within 0.2–3%; tracemalloc/manifest byte-exact); 6-run stability stdev 0.52 MB; round-trip integrity exact.

**Production readiness.** The deliverable is **93.5% complete** (58 of 62 hours) — production-ready as a documentation artifact, pending a brief human acceptance step. The percentage reflects AAP-scoped work plus in-scope path-to-production only; the documented optimizations are deliberately excluded as a future, separate engineering task.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 93.5% (58 / 62 h) |
| Analytical substance (O1–O6) | 100% delivered & validated |
| Validation gates passed | 5 / 5 (zero defects) |
| Source-code regression risk | None (`src/` diff = 0) |
| Remaining (human path-to-production) | 4 h |

---

## 9. Development Guide

> This project's "application" is a documentation deliverable; the guide therefore covers (a) accessing the answer document and (b) reproducing the investigation through the canonical entry points. Every command below was executed and verified.

### 9.1 System Prerequisites

- **Git** (to read the repository and verify the read-only diff).
- **Docker 28.x** (verified: `Docker version 28.5.2`) — required for canonical reproduction.
- **Canonical image:** `ghcr.io/scaleapi/swe-atlas:paperless-ngx__paperless-ngx__542221a38dff...` (Python **3.9.23** + full pinned stack).
- **Do NOT use the host interpreter for reproduction.** The host is Python **3.13.7**, and pinned Django 4.0.4 imports the `cgi` module removed in Python 3.13 (PEP 594):

```bash
python3 --version           # Python 3.13.7
python3 -c "import cgi"      # ModuleNotFoundError: No module named 'cgi'  -> use canonical Docker
```

### 9.2 Access the Deliverable (primary artifact)

```bash
cd /tmp/blitzy/paperless-ngx/blitzy-d8807159-272a-4a73-a584-5ff7a6e332ea_c2b4eb

# Prove the change is read-only: only the answer document was added
git diff 542221a38..HEAD --name-status
# Expected: A   blitzy/documentation/paperless-ngx_542221a38dff.md

# Size and section map
wc -l blitzy/documentation/paperless-ngx_542221a38dff.md          # 3619
grep -nE "^#{1,2} " blitzy/documentation/paperless-ngx_542221a38dff.md | head
```

### 9.3 Verify the Root-Cause Citations (no build required)

```bash
# O1/O2 — whole-corpus manifest materialization
sed -n '127,130p' src/documents/management/commands/document_exporter.py
#   documents = Document.objects.order_by("id")
#   document_map = {d.pk: d for d in documents}
#   document_manifest = json.loads(serializers.serialize("json", documents))
#   manifest += document_manifest

sed -n '73p;87p' src/documents/management/commands/document_importer.py
#   self.manifest = json.load(f)
#   call_command("loaddata", manifest_path)

# The payload that makes "metadata" disproportionate to file size
sed -n '117,124p' src/documents/models.py         # content = models.TextField(...)

# O3 negative result
ls src/documents/caching.py                        # No such file or directory
```

### 9.4 Reproduction Environment Setup (canonical Docker)

```bash
# Launch the canonical image
docker run -d --name repro --entrypoint /bin/bash <CANONICAL_IMAGE> -c "sleep infinity"

# Provision native/service gaps
docker exec repro bash -lc "apt-get update && DEBIAN_FRONTEND=noninteractive \
  apt-get install -y redis-server libzbar0 poppler-utils pngquant time"
docker exec repro bash -lc "redis-server --daemonize yes --save '' --appendonly no"

# Initialize the app
docker exec repro bash -lc "cd /app/src && python manage.py migrate --no-input && python manage.py check"
# Expected: migrate exits 0; check -> 'System check identified no issues (0 silenced).'
```

### 9.5 Reproduce the Investigation

```bash
# 1) Populate a scratch DB (outside the repo, under /tmp)
python /tmp/scripts/populate.py --scratch /tmp/s --n 1000 --content-bytes 51200

# 2) Canonical export under memory measurement
cd /app/src
/usr/bin/time -v python manage.py document_exporter /tmp/tgt --no-progress-bar

# 3) Canonical import (round-trip)
/usr/bin/time -v python manage.py document_importer /tmp/tgt --no-progress-bar
# Expected: 'Installed 1001 object(s) from 1 fixture(s)'

# 4) tracemalloc/RSS probe (repeat for stability / inconsistency)
python /tmp/scripts/export_probe.py --scratch /tmp/s --n 1000 --label text1000 --repeat 6
```

### 9.6 Verification Steps & Expected Output

- **Read-only proof:** `git diff 542221a38..HEAD --name-status` returns exactly one `A` line.
- **Manifest bytes (byte-exact):** meta5000 = 4,665,956 B · text1000 = 50,729,956 B · text5000 = 253,665,956 B.
- **tracemalloc peak (byte-identical to doc):** meta5000 ≈ 19.3 MB · text1000 ≈ 148.5 MB · text5000 ≈ 731.6 MB.
- **Export peak RSS (within 0.2–3%):** meta5000 ≈ 84 MB · text1000 ≈ 217 MB · text5000 ≈ 825 MB.
- **Stability:** text1000 ×6 → tm_peaks mean ≈ 147.35 MB, stdev ≈ 0.52 MB (deterministic per corpus).
- **"Not released":** after `del`+`gc.collect()`, tracemalloc current → ~1.4 MB while RSS stays high; `malloc_trim(0)` reclaims ~485 MB.

### 9.7 Example Usage (interpreting a probe row)

```text
label=text5000  tm_peak=731.6MB  tm_current_live=490.9MB  rss_freed_by_trim=484.8MB  gc_objects=77929
# Transient duplicate (O2) = tm_peak - tm_current_live = 731.6 - 490.9 = 240.7 MB
```

### 9.8 Troubleshooting

- **`ModuleNotFoundError: No module named 'cgi'`** → You are on the host Python 3.13. Use the canonical Docker image (Python 3.9.23).
- **`redis.exceptions.ConnectionError`** → Start redis: `redis-server --daemonize yes`.
- **OCR / consume errors** → Install native deps: `apt-get install -y poppler-utils libzbar0 pngquant`.
- **RSS numbers differ by 0.2–3% between runs** → Expected (glibc arena granularity). Compare tracemalloc peak and manifest byte-sizes, which are deterministic.
- **`document_importer` reports duplicate rows** → Import into a **fresh** migrated database (re-run `migrate` in a clean scratch dir).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Read-only diff proof | `git diff 542221a38..HEAD --name-status` |
| Deliverable size | `wc -l blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Section map | `grep -nE "^#{1,2} " blitzy/documentation/paperless-ngx_542221a38dff.md` |
| Citation spot-check | `sed -n '127,130p' src/documents/management/commands/document_exporter.py` |
| Canonical export | `/usr/bin/time -v python manage.py document_exporter /tmp/tgt --no-progress-bar` |
| Canonical import | `/usr/bin/time -v python manage.py document_importer /tmp/tgt --no-progress-bar` |
| DB init | `python manage.py migrate --no-input && python manage.py check` |
| Memory probe | `python /tmp/scripts/export_probe.py --n 1000 --label text1000 --repeat 6` |

### B. Port Reference

| Service | Port | Notes |
|---------|------|-------|
| Redis (Django-Q broker / channels) | 6379 | Started via `redis-server --daemonize yes` for the reproduction context |

> No web server is started for this investigation; the exporter/importer are single-process CLI commands.

### C. Key File Locations

| Path | Role | Lines |
|------|------|-------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | **The deliverable** (answer document) | 3,619 |
| `src/documents/management/commands/document_exporter.py` | Manifest materialization (`:127-130`) | 278 |
| `src/documents/management/commands/document_importer.py` | Retained `self.manifest` (`:73`) + `loaddata` (`:87`) | 174 |
| `src/documents/models.py` | `Document.content` TextField (`:117-124`) | 466 |
| `src/documents/parsers.py` | `parse_date()` + `dateparser` (`:212-274`) | 350 |
| `src/documents/classifier.py` | `CountVectorizer` train/load | 292 |
| `src/paperless/settings.py` | `Q_CLUSTER` (`:449`), `recycle` default (`:452`) | 615 |
| `src/documents/caching.py` | **Does not exist** (O3 negative result) | — |

### D. Technology Versions (pinned stack — unchanged)

| Package | Version | Relevance |
|---------|---------|-----------|
| Python (canonical) | 3.9.23 | Runtime for all canonical measurements |
| Python (host) | 3.13.7 | Cannot run pinned Django 4.0.4 (`cgi`/PEP 594) |
| django | 4.0.4 | `serializers.serialize`, `loaddata` — the materialization mechanism |
| django-q | 1.3.9 | Long-lived worker context; default `recycle: 1` |
| scikit-learn | 1.0.2 | Classifier `CountVectorizer` / `MLPClassifier` |
| dateparser | 1.1.1 | Module-scoped locale cache in `parse_date()` |
| numpy / scipy | 1.22.3 / 1.8.0 | Backing arrays / sparse matrices |
| redis | 3.5.3 | Django-Q broker / channel layer |
| Docker | 28.5.2 | Canonical reproduction host |

### E. Environment Variable Reference

No project environment variables are required to read or reproduce the deliverable. Standard measurement toggles used by the probes are limited to shell scaffolding (scratch paths under `/tmp`); no `.env` or service credentials are needed.

### F. Developer Tools Guide

- **Memory instrumentation:** Python stdlib only — `tracemalloc` (Python-object peak/current), `/proc/self/statm` (process RSS), `gc` (object counts / `gc.collect()`), and `ctypes` `malloc_trim(0)` to demonstrate glibc arena release.
- **Timing / RSS at process level:** GNU `/usr/bin/time -v` (Maximum resident set size).
- **No profiling servers or browser tooling** are applicable — there is no UI in scope.

### G. Glossary

| Term | Meaning |
|------|---------|
| Manifest | The in-memory list of serialized model rows built by the exporter/importer |
| `Document.content` | `TextField` holding the full extracted OCR/plain text — the disproportionate payload |
| Arena retention | glibc/CPython holding freed memory in allocator arenas rather than returning it to the OS |
| `malloc_trim(0)` | glibc call that returns free arena memory to the OS (demonstrates "not a leak") |
| Canonical entry point | The real `manage.py` command path (vs. a synthetic stand-in) |
| S/O/D/I | Claim-status matrix labels: Source-grounded / Observed / Derived / Inferred |
| Round-trip | Export then re-import; integrity confirmed by restored row counts |

---

*Prepared following the Blitzy Project Guide Template. Brand colors applied: Completed / AI Work = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`; Headings / Accents = Violet-Black `#B23AF2`; Highlight = Mint `#A8FDD9`. All figures are consistent across Sections 1.2, 2.1, 2.2, 7, and 8 (58h completed / 4h remaining / 62h total / 93.5% complete).*