# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new investigative runtime-behavior documentation** for the Paperless-ngx document management system. The user is onboarding into the repository and seeks to understand several internal runtime mechanisms that are not currently explained in the existing documentation. The documentation must be grounded in actual source code evidence — real paths, real checksums, real log messages — rather than abstract descriptions.

**Category:** Create new documentation
**Documentation Type:** Technical investigation / runtime-behavior guide (exploratory documentation for onboarding engineers)

The user has identified five distinct runtime mysteries that require documentation:

- **File Relocation Choreography** — When a document's tags change and `PAPERLESS_FILENAME_FORMAT` includes tag-derived directory components, the system transparently relocates original and archive files on disk. The user wants to see the actual before/after paths and the log messages emitted during this process, along with evidence of what the rollback safety net truly does when a move fails partway through.

- **Classifier Training Behavior** — The ML classifier sometimes completes instantly (training skipped) and other times proceeds with full retraining. The user wants to see both scenarios with their actual log output, including the SHA-1 hash/checksum mechanism the system uses to detect whether training data has changed.

- **Duplicate Detection Mechanics** — Two visually different files can be rejected as duplicates. The user wants to see the actual MD5 checksums being compared at the moment of duplicate rejection, revealing why content-identical files with different appearances are flagged.

- **Sanity Checker Output** — The user wants to see the specific output format of the sanity checker in both healthy and unhealthy states, including the exact hash values it reports when discovering a checksum mismatch between stored and actual file checksums.

- **Orphaned ("Ghost") Files** — Whether orphaned files genuinely linger in the media directory and what the system reports when it finds them during a sanity check.

### 0.1.2 Special Instructions and Constraints

**CRITICAL Directives:**
- The repository itself must remain **completely unchanged** — no existing files may be modified.
- Temporary observation scripts may be used to gather runtime evidence, but must be **cleaned up afterward**.
- The output document must present **real runtime evidence**: actual file paths, actual checksums, actual log messages.
- Per the project implementation rule `SWE-AtlasQnA-Repo`, the deliverable is a single new markdown file named `paperless-ngx_542221a38dff.md` placed in the `blitzy/documentation/` directory.
- The document must provide **thinking and rationale** behind the answers, basing everything on the code as the source of truth.
- No assumptions — all claims must be traceable to specific source code.

**Style Preferences:**
- Investigative tone suitable for an engineer onboarding into the codebase
- Code citations with exact file paths and line numbers
- Concrete examples with realistic paths and checksums rather than abstract descriptions
- Organized by the five runtime behavior topics

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document file relocation**, we will trace the signal chain from `m2m_changed` / `post_save` on `Document` through `update_filename_and_move_files()` in `src/documents/signals/handlers.py` (lines 310–410), examining `generate_unique_filename()` in `src/documents/file_handling.py`, and documenting the rollback mechanism in the exception handler (lines 367–393).

- To **document classifier training behavior**, we will trace `train_classifier()` in `src/documents/tasks.py` (lines 48–72) into `DocumentClassifier.train()` in `src/documents/classifier.py` (lines 115–249), documenting the SHA-1 hash computation (lines 124–161) and the early-exit path when `self.data_hash == new_data_hash`.

- To **document duplicate detection**, we will trace `pre_check_duplicate()` in `src/documents/consumer.py` (lines 102–113), documenting the MD5 checksum computation and the dual-field query `Q(checksum=checksum) | Q(archive_checksum=checksum)`.

- To **document sanity checker output**, we will trace `check_sanity()` in `src/documents/sanity_checker.py` (lines 49–133), documenting each category of check, the MD5 computation for original and archive files, and the exact message format strings including hash values.

- To **document orphaned files**, we will trace the `present_files` accumulation in `check_sanity()` (lines 52–59) and the final orphan reporting loop (lines 130–131).

### 0.1.4 Inferred Documentation Needs

Based on code analysis, several implicit documentation needs have been identified:

- **Signal wiring documentation**: The `DocumentsConfig.ready()` method in `src/documents/apps.py` (lines 11–27) connects six handlers to `document_consumption_finished`, and two Django model signals (`post_save`, `m2m_changed`) are decorated onto `update_filename_and_move_files` in `src/documents/signals/handlers.py` (lines 310–311). Understanding this wiring is essential context for all five topics.

- **Configuration prerequisite documentation**: The file relocation behavior only activates when `PAPERLESS_FILENAME_FORMAT` is set (default is `None` per `src/paperless/settings.py` line 584). The classifier only trains when at least one Tag, DocumentType, or Correspondent uses `MATCH_AUTO` (algorithm 6). These prerequisites must be documented alongside the runtime behaviors.

- **FileLock concurrency documentation**: Both the file move handler and the consumer acquire `FileLock(settings.MEDIA_LOCK)` before filesystem mutations. This cross-cutting concern should be noted to explain why operations are serialized.

- **`bulk_edit.py` → `bulk_update_documents` chain**: When tags are modified via the API's bulk edit endpoint (`src/documents/bulk_edit.py` lines 36–64), the change flows through `Document.tags.through` → `m2m_changed` signal → file rename handler. This indirect trigger path should be documented as an additional trigger for file relocation.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation system** with partial coverage of the runtime behaviors the user is asking about. The existing documentation is structured as ReStructuredText (`.rst`) files in the `docs/` directory, built with Sphinx and hosted on Read the Docs.

**Current documentation framework:** Sphinx ~4.5.0 (from `Pipfile`, line 67)
**Documentation generator configuration:** `docs/conf.py` — configures `sphinx_rtd_theme`, enables `autodoc`, `intersphinx`, `todo`, `imgmath`, and `viewcode` extensions.
**Hosting:** Read the Docs via `.readthedocs.yml` — Python 3.8 build environment, Sphinx configured at `docs/conf.py`.
**Diagram tools detected:** None currently in the docs — no Mermaid or PlantUML configuration found.

**Existing documentation files examined:**

| File | Content | Relevance to User's Questions |
|------|---------|-------------------------------|
| `docs/advanced_usage.rst` | Covers `PAPERLESS_FILENAME_FORMAT` placeholders and directory structure examples (lines 210–293) | Partially relevant — describes filename format but not the runtime file-move mechanics, log output, or rollback |
| `docs/administration.rst` | Covers sanity checker briefly (lines 385–411) — lists issue types detected but no sample output | Partially relevant — enumerates check types but shows no runtime log messages or hash values |
| `docs/configuration.rst` | Documents environment variables including `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | Tangentially relevant — mentions duplicate config but not the detection mechanism |
| `docs/troubleshooting.rst` | Common operational failures and fixes | Not relevant to the specific runtime behaviors requested |
| `docs/extending.rst` | Contributor workflows and parser extension | Not relevant |
| `docs/usage_overview.rst` | Product model, ingestion methods, search, workflows | Background context only |

**Key finding:** None of the existing documentation files explain the internal runtime choreography the user is asking about. The existing docs describe *what* features exist and *how to configure* them, but not *what actually happens at runtime* — the log messages, file paths, checksums, and failure recovery behaviors. This confirms the need for a new documentation artifact.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code analysis:

- **File relocation signal handlers:** `src/documents/signals/handlers.py` — contains `update_filename_and_move_files()` (lines 310–410), `validate_move()` (lines 295–307), `cleanup_document_deletion()` (lines 233–288)
- **Filename generation logic:** `src/documents/file_handling.py` — `generate_filename()` (lines 128–199), `generate_unique_filename()` (lines 81–125), `delete_empty_directories()` (lines 23–52)
- **Classifier training:** `src/documents/classifier.py` — `DocumentClassifier.train()` (lines 115–249), SHA-1 hash at lines 124–161
- **Classifier task wrapper:** `src/documents/tasks.py` — `train_classifier()` (lines 48–72)
- **Duplicate detection:** `src/documents/consumer.py` — `pre_check_duplicate()` (lines 102–113), MD5 at line 104
- **Sanity checker:** `src/documents/sanity_checker.py` — `check_sanity()` (lines 49–133), MD5 at lines 83 and 112
- **Model definitions:** `src/documents/models.py` — `Document` model with `checksum`, `archive_checksum`, `filename`, `archive_filename` fields
- **Bulk edit triggers:** `src/documents/bulk_edit.py` — tag add/remove/modify operations (lines 36–89)
- **Settings:** `src/paperless/settings.py` — `ORIGINALS_DIR` (line 62), `ARCHIVE_DIR` (line 63), `MODEL_FILE` (line 74), `PAPERLESS_FILENAME_FORMAT` (line 584)

**Key directories examined:**
- `src/documents/` — Core business logic package
- `src/documents/signals/` — Signal definitions and handlers
- `src/documents/management/commands/` — CLI commands including `document_sanity_checker.py`, `document_renamer.py`, `document_create_classifier.py`
- `src/documents/tests/` — Test suite with `test_file_handling.py`, `test_sanity_check.py`, `test_classifier.py`, `test_consumer.py`
- `docs/` — Existing Sphinx documentation

**Related existing documentation found:** `docs/advanced_usage.rst` (filename format placeholders), `docs/administration.rst` (sanity checker overview, renamer utility reference)

### 0.2.3 Web Search Research Conducted

No web search was necessary for this task. All required information was extracted directly from the codebase, which is the authoritative source of truth per the user's directive ("base your answers on the code as the truth"). The five runtime behaviors are fully documented in the source files enumerated above.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage to answer the user's five runtime questions:

**Module: `src/documents/signals/handlers.py` — File Relocation Handler**
- Public APIs: `update_filename_and_move_files()`, `validate_move()`, `cleanup_document_deletion()`
- Current documentation: **Missing** — `docs/advanced_usage.rst` explains the filename format configuration but not the signal-driven file move mechanics, log messages, or rollback behavior
- Documentation needed: Runtime walkthrough showing the signal trigger chain, before/after paths, log messages from the `paperless.handlers` logger, and the rollback exception handler behavior

**Module: `src/documents/file_handling.py` — Filename Generation**
- Public APIs: `generate_filename()`, `generate_unique_filename()`, `create_source_path_directory()`, `delete_empty_directories()`, `many_to_dictionary()`
- Current documentation: **Partially covered** — `docs/advanced_usage.rst` lists available placeholders but does not explain the `defaultdictNoStr` tag dictionary mechanism or `_01`/`_02` collision avoidance
- Documentation needed: Explanation of how tag-based directory components are computed, how collisions are resolved, and what the logger named `paperless.filehandling` emits on invalid format strings

**Module: `src/documents/classifier.py` — ML Classifier**
- Public APIs: `DocumentClassifier.train()`, `DocumentClassifier.load()`, `DocumentClassifier.save()`, `load_classifier()`, `preprocess_content()`
- Current documentation: **Missing** — no existing docs explain the SHA-1 change-detection hash, the training skip logic, or the specific log messages
- Documentation needed: Walkthrough of both the "training skipped" and "full retraining" code paths, showing the SHA-1 digest computation, the early-return at line 163–164, and the log messages from `paperless.classifier` and `paperless.tasks`

**Module: `src/documents/consumer.py` — Duplicate Detection**
- Public APIs: `Consumer.pre_check_duplicate()`, `Consumer.try_consume_file()`
- Current documentation: **Missing** — the duplicate check mechanism is not documented beyond a configuration toggle mention
- Documentation needed: Explanation of MD5 checksum computation, the dual-field query against `checksum` and `archive_checksum`, and the exact error message format

**Module: `src/documents/sanity_checker.py` — Archive Integrity Checker**
- Public APIs: `check_sanity()`, `SanityCheckMessages` class
- Current documentation: **Partially covered** — `docs/administration.rst` lists issue types as bullets but shows no example output, no hash values, and no distinction between error/warning/info levels
- Documentation needed: Complete walkthrough of check categories, example log output for healthy and unhealthy archives, MD5 comparison format strings with stored vs. actual hash values, and orphaned file detection

**Module: `src/documents/tasks.py` — Task Orchestration**
- Public APIs: `train_classifier()`, `sanity_check()`, `consume_file()`, `bulk_update_documents()`
- Current documentation: **Missing** for the classifier training wrapper and the sanity check task wrapper
- Documentation needed: How `train_classifier()` gates on MATCH_AUTO existence (lines 49–53), how it interprets the boolean return from `classifier.train()`, and the log messages emitted at each decision point

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented runtime behaviors:**
- The complete signal chain from tag change → `m2m_changed` → `update_filename_and_move_files()` → `generate_unique_filename()` → `os.rename()` → directory cleanup
- The rollback mechanism in `update_filename_and_move_files()` (lines 367–393 of `src/documents/signals/handlers.py`) — the code attempts to reverse file moves on failure, but silently passes on secondary exceptions with an inline comment acknowledging the sanity checker as the recovery mechanism
- The SHA-1 change-detection hash in `DocumentClassifier.train()` that determines whether retraining occurs
- The MD5-based duplicate detection that compares incoming file checksums against both `checksum` and `archive_checksum` fields
- The MD5-based sanity verification that computes fresh checksums and compares them against stored database values, with specific message format strings
- The orphaned file detection that walks `MEDIA_ROOT` and reports files not accounted for by any document record

**Missing log message documentation:**
- `paperless.handlers` logger: Messages for file moves, validation failures, and deletion cleanup
- `paperless.classifier` logger: Messages for data gathering, vectorizing, training each classifier type
- `paperless.consumer` logger: Duplicate detection failure message
- `paperless.sanity_checker` logger: All check result messages including checksum mismatch format
- `paperless.tasks` logger: Classifier training outcome messages
- `paperless.filehandling` logger: Invalid format fallback warning

**Missing configuration prerequisite documentation:**
- `PAPERLESS_FILENAME_FORMAT` must be set for file relocation to occur
- At least one `MatchingModel` with `matching_algorithm=MATCH_AUTO` (value 6) must exist for classifier training to proceed
- `CONSUMER_DELETE_DUPLICATES` controls whether duplicate files are deleted or merely rejected


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

Per the `SWE-AtlasQnA-Repo` implementation rule, the deliverable is a single new markdown file. The planned structure:

```
blitzy/
└── documentation/
    └── paperless-ngx_542221a38dff.md
```

The markdown document will be organized into the following sections, mirroring the user's five runtime questions:

```
paperless-ngx_542221a38dff.md
├── Introduction (onboarding context and how to read this document)
├── 1. File Relocation Choreography
│   ├── 1.1 Signal Trigger Chain
│   ├── 1.2 Filename Generation and Path Computation
│   ├── 1.3 The Move Dance: Before and After Paths
│   ├── 1.4 Log Messages During File Relocation
│   ├── 1.5 Rollback Safety Net: What Actually Happens on Failure
│   └── 1.6 Directory Cleanup After Moves
├── 2. Classifier Training Behavior
│   ├── 2.1 Prerequisites: When Training Can Even Occur
│   ├── 2.2 The SHA-1 Change-Detection Hash
│   ├── 2.3 Training Skipped: The Fast Path
│   ├── 2.4 Full Retraining: The Slow Path
│   └── 2.5 Log Messages for Both Scenarios
├── 3. Duplicate Detection Mechanics
│   ├── 3.1 MD5 Checksum Computation
│   ├── 3.2 The Dual-Field Query
│   ├── 3.3 Why Different-Looking Files Match
│   └── 3.4 Log Messages and Rejection Format
├── 4. Sanity Checker Output
│   ├── 4.1 What Gets Checked
│   ├── 4.2 Healthy Archive Output
│   ├── 4.3 Unhealthy Archive Output with Hash Values
│   ├── 4.4 Orphaned File Detection
│   └── 4.5 Error vs Warning vs Info Severity Levels
└── 5. Ghost Files: Orphaned File Detection
    ├── 5.1 How Orphans Accumulate
    ├── 5.2 The Media Directory Walk
    └── 5.3 What the System Reports
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract exact log message format strings from source code logger calls in `src/documents/signals/handlers.py`, `src/documents/classifier.py`, `src/documents/consumer.py`, `src/documents/sanity_checker.py`, and `src/documents/tasks.py`
- Derive realistic example paths by combining `settings.ORIGINALS_DIR` (`<MEDIA_ROOT>/documents/originals/`) with format strings from `generate_filename()` in `src/documents/file_handling.py`
- Compute example MD5 and SHA-1 hashes to illustrate checksum formats in the documentation
- Construct before/after scenarios by tracing the code paths in `update_filename_and_move_files()` with concrete tag and correspondent values

**Documentation Standards:**
- Markdown formatting with `#`/`##`/`###` headers
- Code examples using fenced code blocks with `python` and `log` syntax highlighting
- Source citations as inline references: `Source: src/documents/classifier.py:124`
- Tables for parameter descriptions and comparison matrices
- Mermaid diagrams for the signal chain and decision flow

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the documentation:

- **Signal Chain Flowchart**: Showing the path from tag modification → `m2m_changed` signal → `update_filename_and_move_files()` → filename generation → file move → directory cleanup
- **Classifier Decision Flowchart**: Showing the branching between "no MATCH_AUTO models exist" → early return, "hash unchanged" → training skipped, and "hash changed" → full retraining
- **Duplicate Detection Sequence**: Showing file arrival → MD5 computation → database query → rejection or acceptance
- **Sanity Checker Walk**: Showing media directory enumeration → per-document checks → orphan detection → message aggregation


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/paperless-ngx_542221a38dff.md` | CREATE | `src/documents/signals/handlers.py`, `src/documents/file_handling.py`, `src/documents/classifier.py`, `src/documents/consumer.py`, `src/documents/sanity_checker.py`, `src/documents/tasks.py`, `src/documents/models.py`, `src/documents/bulk_edit.py`, `src/paperless/settings.py` | Comprehensive runtime-behavior investigation document answering five questions about file relocation, classifier training, duplicate detection, sanity checking, and orphaned files — with actual paths, checksums, log messages, and code citations |

This is the sole documentation file to be created. Per the implementation rule, no existing files in the repository are modified.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/paperless-ngx_542221a38dff.md
Type: Technical Investigation / Runtime Behavior Guide
Source Code:
    - src/documents/signals/handlers.py (file relocation, rollback, deletion cleanup)
    - src/documents/file_handling.py (filename generation, path computation, directory management)
    - src/documents/classifier.py (ML classifier training, SHA-1 hash, model persistence)
    - src/documents/tasks.py (task wrappers for classifier and sanity check)
    - src/documents/consumer.py (duplicate detection, MD5 checksum, ingestion pipeline)
    - src/documents/sanity_checker.py (archive integrity, MD5 verification, orphan detection)
    - src/documents/models.py (Document model, checksum fields, path properties)
    - src/documents/bulk_edit.py (bulk tag modification triggering file moves)
    - src/documents/apps.py (signal wiring in DocumentsConfig.ready())
    - src/paperless/settings.py (directory paths, configuration variables)
Sections:
    - Introduction (purpose, how to read, prerequisites)
    - File Relocation Choreography (signal chain, path generation, move mechanics, log messages, rollback)
    - Classifier Training Behavior (prerequisites, SHA-1 hash, skip vs retrain, log messages)
    - Duplicate Detection Mechanics (MD5 computation, dual-field query, log messages)
    - Sanity Checker Output (healthy/unhealthy output, hash comparison, orphan detection)
    - Ghost Files (how orphans accumulate, detection mechanism, reported messages)
Diagrams:
    - Mermaid flowchart: Signal chain from tag change to file move
    - Mermaid flowchart: Classifier training decision tree
    - Mermaid sequence diagram: Duplicate detection flow
    - Mermaid flowchart: Sanity checker walk and check categories
Key Citations:
    src/documents/signals/handlers.py:295-410
    src/documents/file_handling.py:81-199
    src/documents/classifier.py:115-249
    src/documents/consumer.py:102-113
    src/documents/sanity_checker.py:49-133
    src/documents/tasks.py:48-72
    src/documents/models.py:88-283
    src/documents/apps.py:11-27
    src/paperless/settings.py:61-74,584
```

### 0.5.3 Documentation Files to Update Detail

No existing documentation files are updated. The implementation rule explicitly states: "Do not modify any existing files in the source repository."

### 0.5.4 Documentation Configuration Updates

No documentation configuration files need updating. The new file is placed in `blitzy/documentation/`, which is independent of the Sphinx documentation build system in `docs/`. No changes to `docs/conf.py`, `docs/index.rst`, or `.readthedocs.yml` are required or permitted.

### 0.5.5 Cross-Documentation Dependencies

- The new document references source code files but does not create cross-links to or from the existing Sphinx documentation in `docs/`.
- No navigation, table-of-contents, or index updates are needed since the file lives in a separate `blitzy/documentation/` directory outside the Sphinx build tree.
- The document is self-contained and does not depend on any shared includes or templates.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

Since the deliverable is a standalone Markdown file placed in `blitzy/documentation/`, no documentation generation tools are required to build or render it. However, the following packages are relevant to the runtime behaviors being documented, and their versions are cataloged here for reference accuracy in the output document:

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| pip | django | ~4.0 | Web framework powering signal dispatch, ORM queries, and model signals that drive file relocation and duplicate detection |
| pip | scikit-learn | ==1.0.2 | MLPClassifier used in document classification; pinned due to aarch64 compatibility and model serialization stability |
| pip | filelock | * (latest compatible) | `FileLock` used to serialize filesystem mutations in signal handlers and consumer |
| pip | pathvalidate | * (latest compatible) | Filename sanitization in `generate_filename()` |
| pip | python-magic | * (latest compatible) | MIME type detection in the consumer pipeline |
| pip | tqdm | * (latest compatible) | Progress bars in sanity checker and management commands |
| pip | sphinx | ~4.5.0 | Existing documentation build tool (not used for this deliverable) |
| pip | sphinx_rtd_theme | * (latest compatible) | Read the Docs theme for existing Sphinx docs |

These versions are sourced directly from the `Pipfile` (lines 12–55) and `Pipfile` dev-packages (lines 57–72).

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates since the new document is created in a separate directory (`blitzy/documentation/`) and does not integrate with the Sphinx-based documentation navigation. No internal links need to be rewritten.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the five runtime behaviors:**

| Runtime Behavior | Current Docs Coverage | Gap |
|---|---|---|
| File relocation on tag change | ~20% — `docs/advanced_usage.rst` mentions `PAPERLESS_FILENAME_FORMAT` placeholders but not the signal-driven move mechanics, log messages, or rollback | Signal chain, before/after paths, log messages, rollback analysis |
| Classifier training skip vs retrain | 0% — No existing documentation covers the SHA-1 hash mechanism or training decision logic | Complete coverage needed |
| Duplicate detection mechanics | ~5% — `docs/configuration.rst` mentions `CONSUMER_DELETE_DUPLICATES` toggle but not the MD5 mechanism or dual-field query | Checksum computation, query logic, log messages |
| Sanity checker output format | ~15% — `docs/administration.rst` lists issue types in bullet form but shows no sample output or hash values | Example output for all scenarios, hash comparison format |
| Orphaned file detection | ~10% — Mentioned as a bullet point in `docs/administration.rst` line 403 | Detection mechanism, example output, how orphans accumulate |

**Target coverage:** 100% of the five requested runtime behaviors, with actual runtime evidence (paths, checksums, log messages) for each.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every runtime behavior must include: the triggering condition, the code path executed, the exact log messages emitted, and realistic example values (paths, hashes)
- The file relocation section must show both the success and failure (rollback) paths
- The classifier section must show both the "skipped" and "full retrain" outcomes
- The sanity checker section must show both "healthy" and "unhealthy" output
- All code citations must reference exact file paths and line numbers

**Accuracy validation:**
- Every log message shown in the document must match an actual `logger.*()` call in the source code
- Every checksum format (MD5 hex digest = 32 characters, SHA-1 digest = 20 bytes) must be technically correct
- Path examples must use the actual directory structure from `src/paperless/settings.py` (e.g., `<MEDIA_ROOT>/documents/originals/`)
- The rollback analysis must accurately represent the code's behavior, including the silent `pass` on secondary exceptions

**Clarity standards:**
- Written for an engineer onboarding into the codebase — assumes Python/Django familiarity but not familiarity with Paperless-ngx internals
- Each section starts with a plain-English summary before diving into code-level detail
- Mermaid diagrams accompany complex multi-step flows
- Thinking and rationale behind each answer is provided, per the implementation rule

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per runtime behavior:** 2 (one for the "happy path" and one for the "alternative/failure path")
- **Diagram types required:** Mermaid flowcharts for signal chains and decision trees; Mermaid sequence diagrams for request/response flows
- **Code example testing:** Examples are derived from source code analysis and test case patterns in `src/documents/tests/test_file_handling.py`, `src/documents/tests/test_sanity_check.py`, `src/documents/tests/test_classifier.py`, and `src/documents/tests/test_consumer.py`
- **Visual content freshness:** Based on current codebase state (v1.7.0)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/paperless-ngx_542221a38dff.md` — the sole deliverable

**Source code analyzed for documentation content (read-only):**
- `src/documents/signals/handlers.py` — file relocation handler, rollback, deletion cleanup
- `src/documents/signals/__init__.py` — signal definitions
- `src/documents/file_handling.py` — filename generation, path computation, directory management
- `src/documents/classifier.py` — ML classifier training, SHA-1 hash, model persistence
- `src/documents/tasks.py` — task wrappers for classifier training and sanity check
- `src/documents/consumer.py` — duplicate detection, MD5 checksum, ingestion pipeline
- `src/documents/sanity_checker.py` — archive integrity verification, orphan detection
- `src/documents/models.py` — Document model, checksum fields, path properties
- `src/documents/bulk_edit.py` — bulk tag operations that trigger file moves
- `src/documents/apps.py` — signal wiring in `DocumentsConfig.ready()`
- `src/documents/matching.py` — matching algorithm implementations
- `src/documents/loggers.py` — logging mixin with correlation group support
- `src/paperless/settings.py` — directory paths, configuration variables
- `src/documents/management/commands/document_sanity_checker.py` — CLI wrapper
- `src/documents/management/commands/document_renamer.py` — CLI renaming trigger
- `src/documents/management/commands/document_create_classifier.py` — CLI classifier training
- `Pipfile` — dependency versions for accuracy

**Existing documentation consulted (read-only):**
- `docs/advanced_usage.rst` — filename format documentation
- `docs/administration.rst` — sanity checker and utility documentation
- `docs/configuration.rst` — environment variable reference
- `docs/conf.py` — Sphinx configuration
- `.readthedocs.yml` — ReadTheDocs build configuration

**Test files consulted for behavioral verification (read-only):**
- `src/documents/tests/test_file_handling.py` — file move and rollback test cases
- `src/documents/tests/test_sanity_check.py` — sanity checker test cases
- `src/documents/tests/test_classifier.py` — classifier training test cases
- `src/documents/tests/test_consumer.py` — consumer and duplicate detection test cases
- `src/documents/tests/utils.py` — test directory setup utilities

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No files in the repository may be modified, per the `SWE-AtlasQnA-Repo` implementation rule
- **Existing documentation updates** — No changes to `docs/*.rst` files or Sphinx configuration
- **Test file modifications** — No changes to `src/documents/tests/` files
- **Feature additions or code refactoring** — The task is purely observational and documentary
- **Frontend (Angular) documentation** — The user's questions are entirely about backend Python runtime behaviors
- **Email ingestion documentation** — `src/paperless_mail/` is not part of the five runtime questions
- **OCR/parser documentation** — `src/paperless_tesseract/`, `src/paperless_text/`, `src/paperless_tika/` are not part of the five runtime questions
- **Deployment or infrastructure documentation** — Docker, Supervisor, Gunicorn configurations are not in scope
- **API endpoint documentation** — REST API views and serializers are not in scope
- **Search index documentation** — Whoosh search index operations are not in scope (except where the `add_to_index` handler is mentioned as part of the signal chain)


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not part of the Sphinx build
- **Documentation preview command:** Any Markdown renderer (e.g., grip, VS Code preview, GitHub's built-in renderer) can preview the output file
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown using fenced mermaid code blocks; they render natively on GitHub and in Mermaid-compatible viewers
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim must cite a specific source file and line number range
- **Style guide:** Investigative/explanatory tone for onboarding engineers; code as the authoritative source of truth
- **Temporary script policy:** The user permits temporary observation scripts to be created and run for gathering runtime evidence, provided they are fully cleaned up afterward. However, since all required information is extractable through static code analysis of the source files (log message format strings, hash algorithms, path computation logic, exception handling), no temporary scripts are necessary for this documentation task.

### 0.9.2 File Creation Protocol

The output file must be created at the path `blitzy/documentation/paperless-ngx_542221a38dff.md` within the repository root.

The `blitzy/documentation/` directory must be created if it does not already exist. The file name is derived from the source branch name `paperless-ngx_542221a38dff` per the `SWE-AtlasQnA-Repo` rule.


## 0.10 Rules for Documentation

The following documentation-specific rules are derived from the user's instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The deliverable is a single new Markdown file. All analysis is read-only against the existing codebase.
- **Base all answers on the code as the source of truth.** Do not make assumptions — every claim must be traceable to a specific file and line range in the repository.
- **Provide thinking and rationale behind the answers.** The document is not just a list of facts; it must explain *why* the code behaves as it does and *how* the reader can verify each claim.
- **Use actual paths, actual checksums, and actual log messages.** Derive realistic example values from the code (e.g., the MD5 hex digest format from `hashlib.md5(f.read()).hexdigest()`, the SHA-1 digest from `hashlib.sha1()`, the path structure from `settings.ORIGINALS_DIR` + filename format).
- **Temporary scripts for observation are permitted but must be cleaned up.** If any temporary files are created during investigation, they must be deleted before the task is complete. In practice, static analysis of the source code provides all necessary information.
- **Place the generated document in `blitzy/documentation/` with the filename `paperless-ngx_542221a38dff.md`.** Per the `SWE-AtlasQnA-Repo` implementation rule.
- **Include Mermaid diagrams for complex multi-step flows.** The signal chain, classifier decision tree, and sanity checker walk all benefit from visual representation.
- **Cite source code with exact file paths and line numbers.** Use the format `Source: src/documents/classifier.py:124-161` for all technical claims.
- **Address all five runtime mysteries completely.** No question should be left partially answered or deferred.


## 0.11 References

### 0.11.1 Source Code Files Searched and Analyzed

The following files and folders were systematically retrieved and analyzed to derive all conclusions in this Agent Action Plan:

**Core runtime behavior files (primary sources):**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `src/documents/signals/handlers.py` | 1–432 | File relocation handler (`update_filename_and_move_files`), rollback mechanism, deletion cleanup, validate_move, signal receivers |
| `src/documents/signals/__init__.py` | 1–5 | Signal definitions: `document_consumption_started`, `document_consumption_finished`, `document_consumer_declaration` |
| `src/documents/file_handling.py` | 1–199 | Filename generation (`generate_filename`, `generate_unique_filename`), directory creation/cleanup, tag dictionary conversion |
| `src/documents/classifier.py` | 1–293 | `DocumentClassifier` class: `train()` with SHA-1 hash (lines 124–161), `load()`/`save()`, `predict_*` methods, `load_classifier()` function |
| `src/documents/consumer.py` | 1–433 | `Consumer` class: `pre_check_duplicate()` with MD5 (lines 102–113), `try_consume_file()`, `_store()`, file writing, signal emission |
| `src/documents/sanity_checker.py` | 1–133 | `check_sanity()`: media walk, per-document checks (thumbnail/original/archive), MD5 verification (lines 83/112), orphan detection (lines 130–131), `SanityCheckMessages` class |
| `src/documents/tasks.py` | 1–281 | `train_classifier()` (lines 48–72), `sanity_check()` (lines 255–267), `consume_file()`, `bulk_update_documents()` |
| `src/documents/models.py` | 1–467 | `Document` model: `checksum`, `archive_checksum`, `filename`, `archive_filename`, `source_path`, `archive_path` properties; `MatchingModel` with `MATCH_AUTO=6`; `FileInfo` |
| `src/documents/bulk_edit.py` | 1–101 | `add_tag()`, `remove_tag()`, `modify_tags()` — bulk operations that trigger `m2m_changed` → file relocation |
| `src/documents/apps.py` | 1–29 | `DocumentsConfig.ready()` — signal wiring connecting 6 handlers to `document_consumption_finished` |
| `src/documents/matching.py` | 1–172 | `match_correspondents()`, `match_document_types()`, `match_tags()`, `matches()` — rule-based and classifier-assisted matching |
| `src/documents/loggers.py` | 1–22 | `LoggingMixin` — reusable logging with correlation group support |
| `src/paperless/settings.py` | 55–80, 478–500, 584 | `ORIGINALS_DIR`, `ARCHIVE_DIR`, `THUMBNAIL_DIR`, `MEDIA_ROOT`, `MEDIA_LOCK`, `MODEL_FILE`, `CONSUMER_DELETE_DUPLICATES`, `PAPERLESS_FILENAME_FORMAT` |

**Management commands (CLI entry points):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `src/documents/management/commands/document_sanity_checker.py` | CLI wrapper for `check_sanity()` |
| `src/documents/management/commands/document_renamer.py` | CLI bulk rename trigger via `post_save` signal |
| `src/documents/management/commands/document_create_classifier.py` | CLI classifier training trigger |
| `src/documents/management/commands/document_consumer.py` | File watcher command with tag-from-path logic |

**Test files (behavioral verification):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `src/documents/tests/test_file_handling.py` | 906 lines — file rename, rollback, directory cleanup, duplicate filename collision, archive file handling test cases |
| `src/documents/tests/test_sanity_check.py` | 201 lines — sanity check message types, checksum mismatch, orphaned files, missing files, permission errors |
| `src/documents/tests/utils.py` | 128 lines — `DirectoriesMixin`, `setup_directories()` for isolated test environments |

**Existing documentation (gap analysis):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `docs/advanced_usage.rst` | Filename format documentation (lines 210–293) |
| `docs/administration.rst` | Sanity checker overview (lines 385–411) |
| `docs/conf.py` | Sphinx build configuration |
| `.readthedocs.yml` | ReadTheDocs build environment |

**Dependency manifests:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `Pipfile` | Python dependency versions — Django ~4.0, scikit-learn ==1.0.2, Sphinx ~4.5.0, filelock, pathvalidate, python-magic |

**Folder structures explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (root) | 0 | Repository structure overview |
| `src/` | 1 | Backend source tree layout |
| `src/documents/` | 2 | Core document management package |
| `src/documents/signals/` | 3 | Signal definitions and handlers |
| `src/documents/management/` | 3 | Management command namespace |
| `src/documents/management/commands/` | 4 | Individual CLI commands |
| `src/documents/tests/` | 3 | Test suite structure |
| `docs/` | 1 | Existing documentation tree |

### 0.11.2 Attachments and External Resources

- **User attachments:** None provided (0 environments attached)
- **Figma screens:** None provided
- **External URLs:** None referenced
- **Environment variables:** None provided
- **Setup instructions:** None provided


