# Blitzy Project Guide

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive technical Q&A document (`blitzy/documentation/kitty_815df1e210e0.md`) that answers four deep question clusters about the kitty terminal emulator's internals: Unicode text shaping and font fallback at startup, cell metrics/baseline/decoration alignment computation, GPU texture atlas initialization, and observation-only debug methodology. The document is derived exclusively from source-code analysis of 15 kitty source files (C, Python, headers), with 60 verified source citations, 5 Mermaid diagrams, and 22 subsections spanning 1,260 lines. This is a **documentation-only** project — no source files, tests, or configuration were modified.

### 1.2 Completion Status

```mermaid
pie title Project Completion (90.5%)
    "Completed (38h)" : 38
    "Remaining (4h)" : 4
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 42 |
| **Completed Hours (AI)** | 38 |
| **Remaining Hours (Human)** | 4 |
| **Completion Percentage** | 90.5% (38 / 42) |

### 1.3 Key Accomplishments

- [x] Created 1,260-line technical Q&A document at `blitzy/documentation/kitty_815df1e210e0.md`
- [x] Q1 — Unicode Shaping & Font Fallback: 9 subsections covering HarfBuzz buffer/features, BiDi (RTL/LTR), combining diacritics, VS15/VS16, fontconfig resolution chain, fallback discovery, runtime confirmation
- [x] Q2 — Cell Metrics & Decorations: 3 subsections covering `cell_metrics()` algorithm, `calc_cell_metrics()` adjustment pipeline, pre-rendered sprite upload sequence (11 sprites)
- [x] Q3 — GPU Texture Atlas: 6 subsections covering SpriteMap allocation, GL limits, sprite tracker layout, texture format (GL_SRGB8_ALPHA8), sprite upload, atlas readiness
- [x] Q4 — Debug Methodology: 4 subsections covering CLI flags, startup call chain, expected output format, session-only restoration
- [x] Embedded 5 Mermaid diagrams (sequence diagram, 3 flowcharts, 1 data-flow diagram)
- [x] 60 source citations verified against 15 source files
- [x] 80 balanced code fences with language tags on all blocks
- [x] Resolved 8 code review findings and 11 bare code fence issues
- [x] Zero source files modified — confirmed via `git diff`

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| No critical issues | N/A | N/A | N/A |

No critical issues were identified. The deliverable is validated as production-ready. All four Q&A clusters are comprehensively answered with verified source citations. The document is committed and the working tree is clean.

### 1.5 Access Issues

No access issues identified. This is a documentation-only task requiring only read access to existing source files (all verified present) and write access to the `blitzy/documentation/` directory (successfully committed).

### 1.6 Recommended Next Steps

1. **[High]** Conduct human peer review of technical accuracy — verify source code citations against latest `kitty_815df1e210e0` branch state
2. **[Medium]** Validate Mermaid diagram rendering on the target documentation platform (GitHub/GitLab)
3. **[Medium]** Cross-reference document citations against any recent commits to the 15 referenced source files
4. **[Low]** Integrate document into project knowledge base or wiki navigation
5. **[Low]** Set up staleness detection process — re-verify citations when referenced source files change

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source Code Analysis & Tracing | 12 | Deep analysis of 15 source files (kitty/fonts.c, kitty/freetype.c, kitty/shaders.c, kitty/fonts/render.py, kitty/fonts/common.py, kitty/fonts/fontconfig.py, kitty/main.py, kitty/debug_config.py, kitty/state.h, kitty/data-types.h, kitty/unicode-data.h, kitty/glyph-cache.c/.h, kitty/cli.py, kitty/options/definition.py, kitty/fonts.h) to trace initialization call chains, algorithms, and data structures |
| Q1 — Unicode Shaping & Font Fallback Documentation | 8 | 9 subsections: HarfBuzz buffer creation (MONOTONE_CHARACTERS, 2048 pre-alloc), per-font features (-liga/-dlig/-calt), ligature control (disable_ligatures/font_features), BiDi handling (hb_buffer_guess_segment_properties + force_ltr), shaping execution, combining diacritics (CPUCell.cc_idx[3], VS15/VS16), font family resolution chain (fontconfig scoring), fallback font discovery and debug output, runtime configuration confirmation |
| Q2 — Cell Metrics & Decoration Documentation | 4 | 3 subsections: cell_metrics() algorithm with formulas for cell_width/cell_height/baseline/underline_position/thickness/strikethrough_position/thickness, calc_cell_metrics() adjustment pipeline (adjust_metric with POINT/PERCENT/PIXEL units, baseline shift propagation), pre-rendered sprite upload sequence (11 sprites) |
| Q3 — GPU Texture Atlas Documentation | 5 | 6 subsections: SpriteMap allocation and GL limit queries (GL_MAX_TEXTURE_SIZE, GL_MAX_ARRAY_TEXTURE_LAYERS), sprite tracker limits (0xfff cap), layout computation (xnum/max_y/ynum), GPUSpriteTracker increment logic and capacity model, texture format (GL_SRGB8_ALPHA8/GL_NEAREST/GL_CLAMP_TO_EDGE), sprite upload (glTexSubImage3D), atlas readiness verification |
| Q4 — Debug Methodology Documentation | 2 | 4 subsections: CLI debug flags (--debug-rendering, --debug-font-fallback), startup call chain and flag propagation, expected diagnostic output format (debug_config, dump_font_debug, output_cell_fallback_data), restoring settings (session-only flags) |
| Mermaid Diagram Creation | 3 | 5 diagrams: Font Initialization Call-Chain (sequence), HarfBuzz Shaping Pipeline (flowchart), Font Fallback Resolution (flowchart), GPU Atlas Layout (flowchart), Cell Metrics Derivation (data-flow) |
| Source Code References Compilation | 1 | Comprehensive reference table mapping 15 source files to all documented functions, structs, macros, and constants |
| Code Review Fixes & Quality Improvements | 1.5 | Resolved 8 code review findings, added language tags to 11 bare code fences, verified all code fence balance |
| Validation & Citation Cross-Checking | 1.5 | Verified 60 source citations against actual source files, confirmed function signatures, line numbers (±1), default values, struct definitions, and algorithm descriptions |
| **Total Completed** | **38** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Peer review of technical accuracy against source code | 2 | High |
| Mermaid diagram rendering validation on target platform | 0.5 | Medium |
| Cross-reference verification against latest source commits | 1 | Medium |
| Knowledge base / documentation system integration | 0.5 | Low |
| **Total Remaining** | **4** | |

### 2.3 Hours Calculation

```
Completed Hours:  38h (AAP deliverables implemented and validated)
Remaining Hours:   4h (path-to-production human review tasks)
Total Hours:      42h (38 + 4)
Completion:       38 / 42 = 90.5%
```

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Source Citation Verification | Manual (Blitzy Agent) | 60 | 60 | 0 | 100% | All source citations verified against actual files — function names, line numbers, default values, struct definitions confirmed |
| Document Structure Validation | Automated (grep/wc) | 6 | 6 | 0 | 100% | 38 headings, 137 table rows, 80 balanced code fences, 5 Mermaid diagrams, 60 citations, 6,069 words |
| Code Fence Balance Check | Automated (grep) | 1 | 1 | 0 | 100% | 80 code fences = 40 balanced pairs (even count confirmed) |
| Git Status Verification | git diff | 1 | 1 | 0 | 100% | Only `blitzy/documentation/kitty_815df1e210e0.md` modified (Added); no source files touched |
| AAP Requirement Coverage | Manual (Blitzy Agent) | 4 | 4 | 0 | 100% | All 4 Q&A clusters (Q1–Q4) comprehensively answered with evidence |

**Note:** This is a documentation-only project. No compiled code, unit tests, or application runtime validation is applicable. All tests listed above originate from Blitzy's autonomous validation process during the documentation creation and review cycle.

---

## 4. Runtime Validation & UI Verification

**Runtime Validation:**

This is a documentation-only project — no application runtime, API endpoints, or services were modified or deployed. Runtime validation is not applicable.

- ✅ **Git repository status**: Working tree clean, branch up-to-date with remote
- ✅ **File integrity**: `blitzy/documentation/kitty_815df1e210e0.md` — 1,260 lines, 53,305 bytes, UTF-8 text
- ✅ **No source modifications**: Confirmed via `git diff origin/kitty_815df1e210e0...HEAD --name-status` — only `A blitzy/documentation/kitty_815df1e210e0.md`
- ✅ **All 15 referenced source files exist**: Verified via `ls` — all files present in repository

**UI Verification:**

No UI changes were made. The deliverable is a markdown document. Visual verification covers:

- ✅ **Markdown structure**: Valid ATX headings (H1 → H2 → H3), proper nesting
- ✅ **Table formatting**: 137 pipe-delimited table rows with header separators
- ✅ **Code blocks**: 80 balanced fences with language tags (c, python, conf, text, mermaid)
- ⚠️ **Mermaid rendering**: Diagrams are syntactically valid but should be verified on the target rendering platform (GitHub/GitLab)

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|-----------------|--------|----------|
| Create `blitzy/documentation/kitty_815df1e210e0.md` | ✅ Pass | File exists: 1,260 lines, 53,305 bytes, committed in 3 commits |
| Q1 — Unicode Shaping & Font Fallback (9 subsections) | ✅ Pass | Sections 1.1–1.9 present with HarfBuzz buffer, BiDi, combining marks, font resolution, fallback, runtime confirmation |
| Q2 — Cell Metrics & Decoration (3 subsections) | ✅ Pass | Sections 2.1–2.3 present with cell_metrics() formulas, calc_cell_metrics() pipeline, sprite sequence |
| Q3 — GPU Texture Atlas (6 subsections) | ✅ Pass | Sections 3.1–3.6 present with SpriteMap, GL limits, tracker layout, texture format, upload, readiness |
| Q4 — Debug Methodology (4 subsections) | ✅ Pass | Sections 4.1–4.4 present with CLI flags, call chain, output format, restoration |
| 5 Mermaid diagrams | ✅ Pass | 5 mermaid blocks: sequenceDiagram, 3× flowchart TD, 1× flowchart LR |
| Source citations for every technical claim | ✅ Pass | 60 `Source:` citations verified against actual source files |
| No source files modified | ✅ Pass | `git diff` shows only `A blitzy/documentation/kitty_815df1e210e0.md` |
| Code-as-truth (no assumptions) | ✅ Pass | Every claim traces to specific file:function:line — verified 50+ citations |
| Thinking/rationale behind answers | ✅ Pass | Every subsection includes **Rationale:** paragraphs explaining *why* code works that way |
| Document in `blitzy/documentation/` directory | ✅ Pass | File at `blitzy/documentation/kitty_815df1e210e0.md` |
| Code review fixes applied | ✅ Pass | 8 findings resolved in commit `b2b5b09cd` |
| Language tags on code fences | ✅ Pass | 11 bare code fences fixed in commit `f8c1b2983` |

**Fixes Applied During Autonomous Validation:**
1. Resolved 8 code review findings (citation accuracy, formatting consistency, structural improvements)
2. Added language tags to 11 bare fenced code blocks (`c`, `python`, `conf`, `text`)

**Outstanding Compliance Items:** None — all AAP requirements met.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source citations become stale as kitty source evolves | Technical | Medium | Medium | Document header notes branch `kitty_815df1e210e0` for staleness detection; re-verify citations when referenced files change | Open |
| Mermaid diagrams may not render identically across all platforms | Technical | Low | Low | Diagrams use standard Mermaid syntax; validate on target platform (GitHub/GitLab) before merge | Open |
| Line number citations may drift ±5 lines after future commits | Technical | Low | Medium | Citations use `file:function()` format as primary reference with line numbers as supplementary; function names are more stable | Mitigated |
| Document may miss nuances of macOS CoreText font path | Technical | Low | Low | AAP explicitly scoped to fontconfig (Linux) path; CoreText mentioned only where shared interfaces exist | Accepted |
| No automated staleness detection for source citations | Operational | Low | Medium | Recommend CI check or manual re-verification process when referenced source files are modified | Open |
| Large document size (1,260 lines) may reduce readability | Operational | Low | Low | Document uses clear heading hierarchy, table of contents, and progressive disclosure (summary → detail → citation) | Mitigated |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 38
    "Remaining Work" : 4
```

**Remaining Hours by Category:**

| Category | Hours |
|----------|-------|
| Peer review of technical accuracy | 2 |
| Cross-reference verification | 1 |
| Mermaid rendering validation | 0.5 |
| Knowledge base integration | 0.5 |
| **Total** | **4** |

---

## 8. Summary & Recommendations

### Achievement Summary

The project is **90.5% complete** (38 hours completed out of 42 total hours). All four AAP question clusters have been comprehensively answered in a single, self-contained markdown document at `blitzy/documentation/kitty_815df1e210e0.md`. The document includes 22 subsections, 60 verified source citations, 5 Mermaid diagrams, and 137 structured table rows — all grounded exclusively in source-code analysis of 15 kitty source files with no assumptions.

### What Was Delivered

- A 1,260-line technical Q&A document answering deep questions about kitty's text shaping engine (HarfBuzz), font fallback system, cell metric computation, and GPU texture atlas initialization
- Complete code-path traces from Python entry points through C implementations
- Detailed algorithm documentation with formulas (cell_metrics, adjust_metric, sprite_tracker_set_layout)
- 5 architectural diagrams visualizing initialization sequences, shaping pipelines, fallback resolution, atlas layout, and metric derivation
- Every technical claim backed by source citations in `file:function:line` format

### Remaining Gaps

The 4 hours of remaining work are path-to-production human review tasks:
1. **Peer review** (2h): A human developer familiar with kitty's font subsystem should verify the technical accuracy of the source citations and algorithm descriptions
2. **Cross-reference verification** (1h): Verify citations remain accurate against any commits made to the 15 referenced source files since the analysis
3. **Mermaid rendering validation** (0.5h): Verify all 5 diagrams render correctly on the target documentation platform
4. **Knowledge base integration** (0.5h): Link the document into the project's documentation navigation or wiki

### Production Readiness Assessment

The document is **ready for merge** after human peer review. No blocking issues exist. The working tree is clean, all automated validations passed, and the deliverable meets every AAP requirement. The remaining tasks are standard documentation review activities that do not block the merge but are recommended before broad distribution.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Git | 2.x+ | Repository access and branch management |
| Python | 3.8+ | Repository tooling (pyproject.toml requires >=3.8) |
| Markdown viewer | Any | Viewing the deliverable document |

**Note:** This is a documentation-only project. No compilation, build tools, or runtime environments are required to view or validate the deliverable.

### Environment Setup

```bash
# Clone the repository and switch to the feature branch
git clone <repository-url>
cd kitty
git checkout blitzy-bac19829-f144-4570-955e-b786ee17e901
```

### Viewing the Document

```bash
# View the document in terminal
cat blitzy/documentation/kitty_815df1e210e0.md

# View with line numbers
cat -n blitzy/documentation/kitty_815df1e210e0.md

# Check document statistics
wc -l -w -c blitzy/documentation/kitty_815df1e210e0.md
# Expected output: 1260 lines, 6069 words, 53305 bytes
```

**For rich rendering (Mermaid diagrams, tables):**
- Open in GitHub/GitLab web UI (native Mermaid support)
- Open in VS Code with Markdown Preview Enhanced extension
- Use `grip` for local GitHub-style rendering:
  ```bash
  pip install grip
  python3 -m grip blitzy/documentation/kitty_815df1e210e0.md
  # Opens browser at http://localhost:6419
  ```

### Verification Steps

```bash
# Verify only the documentation file was changed
git diff origin/kitty_815df1e210e0...HEAD --name-status
# Expected: A	blitzy/documentation/kitty_815df1e210e0.md

# Verify no source files were modified
git diff origin/kitty_815df1e210e0...HEAD -- kitty/
# Expected: no output (no changes to kitty/ directory)

# Verify document structure
grep "^#" blitzy/documentation/kitty_815df1e210e0.md | head -40
# Expected: 38 headings with proper H1 → H2 → H3 hierarchy

# Verify code fence balance (should be even number)
grep -c '```' blitzy/documentation/kitty_815df1e210e0.md
# Expected: 80 (40 balanced pairs)

# Verify Mermaid diagram count
grep -c '```mermaid' blitzy/documentation/kitty_815df1e210e0.md
# Expected: 5

# Verify source citation count
grep -c 'Source:' blitzy/documentation/kitty_815df1e210e0.md
# Expected: 60

# Verify all referenced source files exist
ls kitty/fonts.c kitty/freetype.c kitty/shaders.c kitty/fonts/render.py \
   kitty/fonts/common.py kitty/fonts/fontconfig.py kitty/main.py \
   kitty/debug_config.py kitty/state.h kitty/data-types.h kitty/cli.py \
   kitty/options/definition.py kitty/fonts.h kitty/glyph-cache.c kitty/glyph-cache.h
# Expected: all 15 files listed without errors
```

### Validating Source Citations

To spot-check a source citation from the document:

```bash
# Example: Verify init_fonts() line reference
grep -n "init_fonts" kitty/fonts.c
# Expected: line 1746 — matches document citation

# Example: Verify cell_metrics() function
grep -n "cell_metrics" kitty/freetype.c
# Expected: line 387 — matches document citation

# Example: Verify VS15/VS16 constants
grep -n "VS15\|VS16" kitty/unicode-data.h
# Expected: line 5 with VS15=1364, VS16=1365
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Mermaid diagrams not rendering | Use a platform with native Mermaid support (GitHub, GitLab) or install VS Code extension "Markdown Preview Enhanced" |
| Long lines causing horizontal scroll | Document contains lines up to 546 chars; use a wide terminal or word-wrap-capable viewer |
| Source citation line numbers off by ±1 | Line numbers are accurate as of the analysis commit; re-verify with `grep -n` if source has been modified since |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `git diff origin/kitty_815df1e210e0...HEAD --name-status` | Verify only documentation file was changed |
| `git diff origin/kitty_815df1e210e0...HEAD --stat` | Summary of changes (1 file, 1260 insertions) |
| `wc -l -w -c blitzy/documentation/kitty_815df1e210e0.md` | Document statistics |
| `grep -c '```' blitzy/documentation/kitty_815df1e210e0.md` | Code fence balance check |
| `grep -c '```mermaid' blitzy/documentation/kitty_815df1e210e0.md` | Mermaid diagram count |
| `grep -c 'Source:' blitzy/documentation/kitty_815df1e210e0.md` | Source citation count |
| `grep "^#" blitzy/documentation/kitty_815df1e210e0.md` | Document heading structure |
| `python3 -m grip blitzy/documentation/kitty_815df1e210e0.md` | Local GitHub-style rendering |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **Deliverable** — Technical Q&A document |
| `kitty/fonts.c` | Primary source: font group init, HarfBuzz shaping, fallback, sprite tracker |
| `kitty/freetype.c` | Source: cell_metrics() algorithm, FreeType face init |
| `kitty/shaders.c` | Source: SpriteMap allocation, GL texture creation, sprite upload |
| `kitty/fonts/render.py` | Source: set_font_family(), prerender_function(), dump_font_debug() |
| `kitty/fonts/common.py` | Source: get_font_files(), font resolution chain |
| `kitty/fonts/fontconfig.py` | Source: fontconfig enumeration, FCScorer, find_best_match() |
| `kitty/main.py` | Source: startup entry point, debug flag propagation |
| `kitty/debug_config.py` | Source: debug_config() output format |
| `kitty/state.h` | Source: Options struct, GlobalState, debug macros |
| `kitty/data-types.h` | Source: FONTS_DATA_HEAD, CPUCell, VS15/VS16 |
| `kitty/options/definition.py` | Source: default values for font options |
| `kitty/cli.py` | Source: --debug-rendering, --debug-font-fallback flags |
| `kitty/fonts.h` | Source: font subsystem API contract |
| `kitty/glyph-cache.c` / `.h` | Source: GPU glyph cache management |
| `kitty/unicode-data.h` | Source: VS15 (1364), VS16 (1365), codepoint_for_mark() |

### D. Technology Versions

| Technology | Version | Role |
|------------|---------|------|
| Python | 3.12.3 (system) / >=3.8 (project requirement) | Repository tooling, font resolution (render.py, common.py, fontconfig.py) |
| Markdown | GitHub-Flavored Markdown (GFM) / CommonMark 0.31 | Document authoring format |
| Mermaid | 11.x (renderer-provided) | Embedded architectural diagrams |
| Git | 2.x+ | Version control, branch management |
| kitty source branch | `kitty_815df1e210e0` | Source code analyzed for documentation |

### G. Glossary

| Term | Definition |
|------|-----------|
| **Font Group** | The `FontGroup` struct in `kitty/fonts.c` — holds medium/bold/italic/bold-italic font faces, fallback fonts, symbol map fonts, cell metrics, and the sprite tracker for a single font configuration |
| **Sprite Tracker** | The `GPUSpriteTracker` struct — tracks the current (x, y, z) write position in the 3D texture atlas and manages grid dimensions (xnum, max_y, ynum) |
| **Sprite Map** | The `SpriteMap` struct in `kitty/shaders.c` — holds the OpenGL texture ID, cell dimensions, grid layout, and GL capability limits for one OS window's glyph texture atlas |
| **Cell Metrics** | The seven values computed by `cell_metrics()`: cell_width, cell_height, baseline, underline_position, underline_thickness, strikethrough_position, strikethrough_thickness |
| **HarfBuzz Buffer** | A `hb_buffer_t` object used for Unicode text shaping — receives codepoints, applies OpenType features, and produces positioned glyph output |
| **MONOTONE_CHARACTERS** | `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS` — cluster level ensuring monotonically increasing cluster indices for simplified cell-to-glyph mapping |
| **force_ltr** | Configuration option (default: false) that overrides HarfBuzz's auto-detected text direction to LTR, used with external BiDi tools like GNU FriBidi |
| **VS15 / VS16** | Variation Selectors 15 (text presentation, index 1364) and 16 (emoji presentation, index 1365) — stored as combining mark indices in CPUCell.cc_idx |
| **Pre-rendered Sprites** | The 11 sprites uploaded during initialization: 1 blank + 5 underline styles + 1 strikethrough + 1 missing glyph + 3 cursor styles |
| **FCScorer** | The `FCScorer` class in `kitty/fonts/fontconfig.py` — scores fontconfig font candidates using a composite tuple of (variable_score, weight+slant, monospace_match, width_score) |
| **modify_font** | Configuration option allowing adjustment of cell_width, cell_height, baseline, underline_position/thickness, strikethrough_position/thickness with POINT, PERCENT, or PIXEL units |
| **debug_fonts** | Macro in `kitty/state.h` — `#define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }` — gates all font fallback debug output behind the `--debug-font-fallback` CLI flag |