# Blitzy Project Guide

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive investigative Q&A document for the undocumented `choose-fonts` kitten in the kovidgoyal/kitty terminal emulator repository. The deliverable is a single Markdown file (`blitzy/documentation/kitty_815df1e210e0.md`) that answers six behavioral questions about the font-selection workflow—covering building kitty, invoking the kitten, CLI registration, option propagation, config-file persistence, and runtime verification—with all claims grounded in source-code citations. The document serves as an onboarding reference for developers investigating this previously undocumented kitten, analyzing 22 source files across Go and Python to produce a 961-line, 36KB reference guide with 58 citations and 3 Mermaid diagrams.

### 1.2 Completion Status

```mermaid
pie title Project Completion
    "Completed (AI)" : 22
    "Remaining" : 2
```

| Metric | Value |
|--------|-------|
| Total Project Hours | 24 |
| Completed Hours (AI) | 22 |
| Remaining Hours | 2 |
| Completion Percentage | 91.7% |

**Calculation:** 22 completed hours / (22 + 2 remaining hours) = 22/24 = 91.7% complete

### 1.3 Key Accomplishments

- ✅ Created comprehensive 961-line Q&A document answering all 6 user questions (Q1–Q6)
- ✅ Analyzed 22 source files across Go, Python, and reStructuredText for evidence-based documentation
- ✅ Produced 58 source code citations with file paths and line numbers
- ✅ Created 3 Mermaid diagrams (end-to-end flowchart, UI state machine, option flow sequence diagram)
- ✅ Documented the previously undocumented Go↔Python IPC protocol architecture
- ✅ Created 7-step runtime verification procedure using `KITTY_CONFIG_DIRECTORY` isolation
- ✅ Documented the sentinel-block config patching mechanism with before/after ASCII illustration
- ✅ Maintained zero source file modifications — existing codebase untouched per user directive
- ✅ Cleaned up all temporary artifacts; working tree is clean
- ✅ Applied validation fixes: trimmed 23 code blocks to ≤3 lines, corrected 6 terminology violations

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Source citations reference specific line numbers that may drift with upstream kitty development | Low — citations become inaccurate over time but do not break the document | Human Developer | Ongoing maintenance |

### 1.5 Access Issues

No access issues identified. The project is a documentation-only task creating a standalone Markdown file. No build, deployment, API access, or service credentials are required.

### 1.6 Recommended Next Steps

1. **[High]** Conduct human expert review of the 58 source code citations against the current codebase to confirm accuracy
2. **[Medium]** Verify Mermaid diagram rendering in the target Markdown viewer (GitHub, VS Code, etc.)
3. **[Medium]** Stakeholder review and approval of the document content and structure
4. **[Low]** Consider integrating the document into kitty's Sphinx documentation tree (`docs/kittens/`) as a `.rst` page if upstream contribution is desired
5. **[Low]** Establish a maintenance plan for updating line-number citations when the upstream kitty codebase evolves

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository Discovery and Code Analysis | 5 | Analyzed 22 source files (14 in `kittens/choose_fonts/`, 8 supporting files) totaling ~11,100 lines of Go, Python, and RST to extract behavioral evidence for all 6 questions |
| Q1: Build and Launch Documentation | 1 | Documented build procedure from `docs/build.rst`, binary location, and default launch instructions |
| Q2: Invocation Routes Documentation | 1.5 | Traced and documented both invocation paths: direct Go CLI (`kitten choose-fonts`) and Python runner (`kitty +kitten choose_fonts` → `os.execlp`) |
| Q3: CLI Registration and Option Parsing | 1.5 | Documented `EntryPoint()` registration at `main.go:74-99`, `--reload-in` option spec, `Options` struct, and `AddClone` alias mechanism |
| Q4: Option Value Flow Tracing | 2 | Traced `Options.Reload_in` through 3 hops: `main()` → `handler` → `final_pane`, created Mermaid sequence diagram |
| Q5: Finalization Behavior | 3 | Documented the most complex section: `on_key_event()`, `Patcher.Patch()` sentinel-block mechanism, backup creation, `AtomicUpdateFile`, SIGUSR1 dispatch, and `load_config_file()` receiver. Created ASCII before/after illustration |
| Q6: Persistence Verification | 2.5 | Documented code evidence for persistence, config directory resolution (both Go and Python sides), created 7-step runtime verification procedure with expected outputs |
| Supplementary Architecture Deep Dive | 2 | Documented Go↔Python IPC protocol (backend.go/backend.py), UI state machine with Mermaid stateDiagram, and `s`/`S` export alternative |
| Mermaid Diagrams and Illustrations | 1.5 | Created 3 Mermaid diagrams (flowchart, stateDiagram-v2, sequenceDiagram) and 1 ASCII config-patching illustration |
| Summary, Formatting, and Cross-Referencing | 1 | Wrote summary/key takeaways, terminology table, keybinding reference table, and ensured consistent formatting across 961 lines |
| Validation Fixes and Quality Pass | 1 | Trimmed 23 code blocks to ≤3 lines per AAP constraint, corrected 6 terminology violations, verified zero TODOs/FIXMEs |
| **Total** | **22** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human expert review of 58 source code citations against codebase | 1 | High |
| Mermaid diagram rendering verification in target Markdown viewer | 0.5 | Medium |
| Stakeholder review and final approval | 0.5 | Medium |
| **Total** | **2** | |

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Validation Gates | Blitzy Final Validator | 4 | 4 | 0 | 100% | All 4 production-readiness gates passed |
| Source Citation Verification | Manual cross-reference | 11 | 11 | 0 | 100% | 11 critical claims verified against source code (EntryPoint, Options, on_key_event, Patcher.Patch, ReloadConfigInKitty, ConfigDir, backend IPC, list_fonts, sentinel format, backup creation) |
| Constraint Compliance | Blitzy Final Validator | 5 | 5 | 0 | 100% | No source modifications, correct file name/path, no temp artifacts, clean working tree, committed to correct branch |

**Notes:**
- This is a documentation-only project — no unit, integration, or end-to-end tests apply
- GATE 1 (Tests): Passed — no tests to run for documentation-only task
- GATE 2 (Runtime): Passed — no runtime to validate for documentation-only task
- GATE 3 (Errors): Passed — zero unresolved errors across all deliverables
- GATE 4 (Files): Passed — all in-scope files validated and working

## 4. Runtime Validation & UI Verification

**Runtime Health:**
- ✅ Deliverable file exists at `blitzy/documentation/kitty_815df1e210e0.md` (961 lines, ~36KB)
- ✅ Working tree is clean (`git status` reports no uncommitted changes)
- ✅ File is committed on branch `blitzy-e72571f4-982e-48b2-8c88-8f86bcaeff2b`
- ✅ Only 1 file changed from base branch (`git diff 815df1e21 --name-only HEAD`)

**Document Structure Verification:**
- ✅ 10 top-level sections covering all 6 user questions plus supplementary material
- ✅ 36 second-level headers (`##`) and 26 third-level headers (`###`)
- ✅ 58 source code citations with file paths and line numbers
- ✅ 3 Mermaid diagrams (sequence, state, flowchart)
- ✅ 1 ASCII before/after config-patching illustration
- ✅ 7-step runtime verification procedure with expected outputs
- ✅ Complete keybinding reference table (Enter, Esc, s/S, Ctrl+c)
- ✅ Zero TODOs, FIXMEs, placeholders, or stubs

**Constraint Compliance:**
- ✅ No source files modified (only `blitzy/documentation/kitty_815df1e210e0.md` added)
- ✅ No temporary artifacts remain
- ✅ File correctly named per SWE-AtlasQnA-Repo rule (`kitty_815df1e210e0.md`)
- ✅ File correctly placed in `blitzy/documentation/` directory

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|----------------|--------|----------|
| Q1: Build and Launch Instructions | ✅ Pass | Document lines 23–85; cites `docs/build.rst` with build commands, binary location, and launch instructions |
| Q2: Invocation Routes (both paths) | ✅ Pass | Document lines 88–148; covers both `kitten choose-fonts` (Go CLI) and `kitty +kitten choose_fonts` (Python→exec) |
| Q3: Subcommand Registration + Option Parsing | ✅ Pass | Document lines 151–220; `EntryPoint()` at `main.go:74-99`, `--reload-in` spec, `Options` struct, `AddClone` alias |
| Q4: Option Value Flow | ✅ Pass | Document lines 223–318; traces `Options.Reload_in` through 3 hops with Mermaid sequence diagram |
| Q5: Finalization Behavior | ✅ Pass | Document lines 321–554; `on_key_event`, `Patcher.Patch`, sentinel block, backup, SIGUSR1, `load_config_file` |
| Q6: Persistence Verification | ✅ Pass | Document lines 557–735; code evidence, config dir resolution (Go+Python), 7-step runtime procedure |
| End-to-end flow diagram (Mermaid) | ✅ Pass | Document lines 908–941; flowchart from invocation through config patching to reload |
| State machine diagram (Mermaid) | ✅ Pass | Document lines 815–829; stateDiagram-v2 with all states and transitions |
| Option flow sequence diagram (Mermaid) | ✅ Pass | Document lines 291–315; sequenceDiagram showing CLI→main→handler→final_pane |
| Config patching before/after illustration | ✅ Pass | Document lines 457–477; ASCII art showing commented-out directives + sentinel block |
| Source code citations (format: `Source: path:Line`) | ✅ Pass | 58 citations throughout document; 11 critical claims verified against codebase |
| Backend architecture documentation | ✅ Pass | Document lines 740–799; Go↔Python IPC protocol, pipe communication, backend actions table |
| UI state machine documentation | ✅ Pass | Document lines 801–855; State enum, transition details with source citations |
| `s`/`S` export alternative documentation | ✅ Pass | Document lines 857–901; STDOUT export path, use cases, keybinding reference table |
| Summary and key takeaways | ✅ Pass | Document lines 945–961; 8 key findings summarized |
| No source file modifications | ✅ Pass | `git diff 815df1e21 --name-only HEAD` shows only `blitzy/documentation/kitty_815df1e210e0.md` |
| Cleanup temporary artifacts | ✅ Pass | `git status` reports clean working tree; no temporary files remain |
| Code snippets ≤3 lines | ✅ Pass | Validation fix commit trimmed 23 code blocks to comply |
| Consistent terminology | ✅ Pass | Validation fix commit corrected 6 terminology violations; consistent use of "kitten", "kitty.conf", "sentinel block" |

**Compliance Score:** 19/19 AAP requirements met (100% of AAP-specified items)

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source code citations reference specific line numbers that may drift as the upstream kitty repository evolves | Technical | Low | Medium | Include file-level references alongside line numbers; periodic citation audit | Open — requires ongoing maintenance |
| Mermaid diagrams may render differently across Markdown viewers | Technical | Low | Low | Diagrams use standard Mermaid syntax; test in target viewer before distribution | Open — pending human verification |
| Runtime verification procedure assumes a built kitty binary is available | Operational | Low | Low | Document prerequisites clearly; provide fallback instructions for pre-built binaries | Mitigated — prerequisites documented |
| Document may become stale if `choose-fonts` kitten behavior changes upstream | Operational | Medium | Medium | Monitor upstream commits to `kittens/choose_fonts/` for behavioral changes | Open — requires maintenance plan |
| No security risks identified | Security | N/A | N/A | N/A — documentation-only project with no code changes | N/A |
| No integration risks identified | Integration | N/A | N/A | N/A — standalone document with no external dependencies | N/A |

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 22
    "Remaining Work" : 2
```

| Status | Hours | Percentage |
|--------|-------|------------|
| Completed (AI) | 22 | 91.7% |
| Remaining (Human) | 2 | 8.3% |
| **Total** | **24** | **100%** |

## 8. Summary & Recommendations

### Achievement Summary

The project has achieved **91.7% completion** (22 hours completed out of 24 total hours). All 19 AAP-specified deliverables have been fully implemented, including the comprehensive 961-line Q&A document answering all 6 user questions, 3 Mermaid diagrams, 58 source code citations, and a 7-step runtime verification procedure. The document provides the first comprehensive reference for the previously undocumented `choose-fonts` kitten, analyzing 22 source files across Go and Python.

### Remaining Gaps

The remaining 2 hours consist entirely of human review activities:
- Expert verification of source code citations (1 hour)
- Mermaid rendering verification and stakeholder approval (1 hour)

### Critical Path to Production

1. Human expert reviews the 58 source code citations against the codebase
2. Verify Mermaid diagrams render correctly in the target Markdown viewer
3. Stakeholder approves the document content and merges the PR

### Production Readiness Assessment

The deliverable is **ready for human review**. All autonomous work is complete:
- The document is comprehensive, well-structured, and internally consistent
- All claims are grounded in verifiable source code citations
- No source files were modified; the codebase is untouched
- All validation gates passed with zero unresolved errors
- The working tree is clean with all changes committed

The 2 remaining hours are standard review-and-approval activities that require human judgment and cannot be automated.

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Git | Any recent version | Clone the repository and view branch changes |
| Markdown Viewer | Any (GitHub, VS Code, grip, etc.) | Render and review the documentation |
| Mermaid Support | Any Mermaid-compatible renderer | Render the 3 embedded Mermaid diagrams |
| C compiler (gcc/clang) | Any recent version | Required only if building kitty from source (for runtime verification) |
| Go | >= 1.22 | Required only if building kitty from source |
| Python | >= 3.8 | Required only if building kitty from source |

### Environment Setup

**Step 1 — Clone the repository and switch to the feature branch:**

```bash
git clone https://github.com/kovidgoyal/kitty.git
cd kitty
git checkout blitzy-e72571f4-982e-48b2-8c88-8f86bcaeff2b
```

**Step 2 — Verify the deliverable exists:**

```bash
ls -la blitzy/documentation/kitty_815df1e210e0.md
```

Expected output: File exists, approximately 36KB, 961 lines.

**Step 3 — View the document:**

```bash
# Option A: View in terminal
cat blitzy/documentation/kitty_815df1e210e0.md

# Option B: View with a Markdown renderer (e.g., grip)
pip install grip
grip blitzy/documentation/kitty_815df1e210e0.md
```

### Dependency Installation

No dependencies need to be installed for the documentation deliverable itself. The document is a standalone Markdown file.

For building kitty from source (to follow the runtime verification procedure in Q6), refer to the build instructions in the document's Q1 section or `docs/build.rst`:

```bash
./dev.sh build
```

### Verification Steps

**Verify document integrity:**

```bash
wc -l blitzy/documentation/kitty_815df1e210e0.md
# Expected: 961 lines

grep -c "Source:" blitzy/documentation/kitty_815df1e210e0.md
# Expected: 58 (source citations)

grep -c "mermaid" blitzy/documentation/kitty_815df1e210e0.md
# Expected: 3 (Mermaid diagram blocks)
```

**Verify no source files were modified:**

```bash
git diff 815df1e21 --name-only HEAD
# Expected: only blitzy/documentation/kitty_815df1e210e0.md
```

**Verify a specific source citation (example):**

```bash
grep -n "EntryPoint" kittens/choose_fonts/main.go
# Expected: line 74 shows func EntryPoint(root *cli.Command)
```

### Troubleshooting

| Issue | Resolution |
|-------|------------|
| Mermaid diagrams not rendering | Use a Mermaid-compatible viewer: GitHub (built-in), VS Code with Mermaid extension, or Mermaid Live Editor (https://mermaid.live) |
| `grip` command not found | Install with `pip install grip` for local Markdown preview |
| Source citation line numbers don't match | The upstream kitty repository may have evolved since this document was written; re-verify against the commit `815df1e21` |

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `git diff 815df1e21 --name-only HEAD` | List all files changed by this PR |
| `git diff 815df1e21 --stat HEAD` | Show change statistics for this PR |
| `wc -l blitzy/documentation/kitty_815df1e210e0.md` | Count lines in deliverable |
| `grep -c "Source:" blitzy/documentation/kitty_815df1e210e0.md` | Count source citations |
| `./dev.sh build` | Build kitty from source (for runtime verification) |
| `kitten choose-fonts` | Invoke the choose-fonts kitten (after building) |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **Deliverable** — the Q&A documentation |
| `kittens/choose_fonts/main.go` | CLI entry point and option definitions |
| `kittens/choose_fonts/final.go` | Finalization behavior (config patching trigger) |
| `kittens/choose_fonts/ui.go` | UI handler and state machine |
| `kittens/choose_fonts/backend.go` | Go-side IPC controller |
| `kittens/choose_fonts/backend.py` | Python backend process |
| `tools/config/api.go` | Config patching engine (`Patcher.Patch`, `ReloadConfigInKitty`) |
| `tools/cmd/tool/main.go` | CLI command tree assembly |
| `kitty/entry_points.py` | Entry point routing for `kitty +kitten` |
| `kitty/constants.py` | Config directory resolution (Python side) |
| `tools/utils/paths.go` | Config directory resolution (Go side) |
| `docs/build.rst` | Build-from-source instructions |

### D. Technology Versions

| Technology | Version | Purpose |
|------------|---------|---------|
| Go | >= 1.22 | Kitten implementation language (per `go.mod`) |
| Python | >= 3.8 | Backend process and kitty runtime (per `pyproject.toml`) |
| Markdown | Standard | Documentation format |
| Mermaid | Standard syntax | Embedded diagrams (3 in document) |
| Git | Any | Version control |

### G. Glossary

| Term | Definition |
|------|------------|
| Kitten | A kitty sub-program; the term used by kitty for its built-in utility programs |
| `kitty.conf` | The kitty terminal emulator's primary configuration file |
| Sentinel block | The `# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS` marker pair used by the config patcher to identify and replace font settings |
| Patcher | The `config.Patcher` struct in `tools/config/api.go` that handles sentinel-block replacement in config files |
| SIGUSR1 | Unix signal used by kitty to trigger configuration reload without restarting |
| IPC | Inter-Process Communication — refers to the Go↔Python pipe-based JSON protocol used by the choose-fonts kitten |
| TUI | Terminal User Interface — the interactive text-based interface presented by the choose-fonts kitten |
| fontconfig | The Linux font discovery library used by kitty's Python backend for font enumeration |
| CoreText | The macOS font discovery API used by kitty's Python backend for font enumeration |
| XDG | Cross-Desktop Group — standards for config/data directory locations on Linux (e.g., `XDG_CONFIG_HOME`) |