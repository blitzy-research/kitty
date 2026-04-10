# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a comprehensive technical deep-dive into kitty's terminal reflow (rewrap) system — the internal C-level mechanism responsible for redistributing text across new terminal dimensions when the window is resized.

- **Category:** Create new documentation
- **Documentation Type:** Technical internals / Architecture documentation with code-level trace
- **Target Audience:** Developers working on or investigating kitty's terminal buffer system, contributors diagnosing reflow edge cases, and terminal emulator implementers studying kitty's approach

The user's requirements, restated with enhanced clarity:

- **R1 — Rewrap Algorithm Trace:** Trace through the `rewrap_inner()` implementation in `kitty/rewrap.h`, explaining each step of the algorithm: source line iteration, continuation detection, trailing blank trimming, chunk-based cell copying, destination line advancement, and cursor position remapping.
- **R2 — Screen/Scrollback Interaction:** Explain how the visible screen buffer (`LineBuf`) and the scrollback history buffer (`HistoryBuf`) interact during a resize event. Specifically, how `screen_resize()` in `kitty/screen.c` orchestrates the reflow of both buffers through `realloc_hb()` and `realloc_lb()`, and how lines overflow from the screen into history.
- **R3 — Line Continuation State Propagation:** Identify how line continuation state (the `next_char_was_wrapped` bit in `GPUCell.attrs` and the `is_continued` bit in `LineAttrs`) is propagated between the screen buffer and the history buffer during rewrap, and flag any potential issues in that propagation.
- **R4 — Complete Data Flow:** Document the complete data flow from the resize entry point (`screen_resize()`) through to the final buffer state, including prompt-protection logic, cursor tracking, the alternate buffer, and the scrollback-fill-on-enlarge feature.
- **R5 — Edge Case Identification:** Identify potential issues where logical line boundaries may not be correctly preserved, including the macro-override mechanism where `HistoryBuf` redefines `next_dest_line`, `init_src_line`, and `first_dest_line` before including `rewrap.h`.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — No file creation or modification in the source repository:** The user explicitly states "Don't create or modify any files." However, the project's implementation rules require creation of a markdown document named `kitty_815df1e210e0.md` in the `blitzy/documentation/` directory. This single documentation output file is the only file to be created.
- **Analysis-only scope:** The user wants a code trace and architectural explanation, not code changes, bug fixes, or refactors.
- **Provide thinking and rationale:** Per implementation rules, answers must provide the reasoning behind conclusions.
- **Base answers on code as truth:** Do not make assumptions — all conclusions must be grounded in the actual source code.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **trace the rewrap implementation**, we will create a detailed walkthrough of `rewrap_inner()` in `kitty/rewrap.h` (lines 56–96), explaining the algorithm's outer source-line loop, inner chunk-copy loop, continuation detection via `is_src_line_continued()`, trailing blank trimming, and cursor coordinate remapping through the `TrackCursor` struct.
- To **explain screen/scrollback interaction**, we will document the `screen_resize()` function in `kitty/screen.c` (lines 346–463), covering the ordered sequence: history buffer reallocation via `realloc_hb()`, prompt-protection via `prevent_current_prompt_from_rewrapping()`, main line buffer reallocation via `realloc_lb()`, alternate line buffer reallocation, cursor position resolution, and scrollback fill-on-enlarge.
- To **document continuation state propagation**, we will trace the `next_char_was_wrapped` flag through `linebuf_set_last_char_as_continuation()` and `history_buf_set_last_char_as_continuation()`, explain how the two different macro expansions of `next_dest_line` in `line-buf.c` vs `history.c` handle this flag differently, and analyze the `HistoryBuf.init_line()` function that derives `is_continued` from the previous line's GPU cell and the pager history ring buffer.
- To **identify edge cases**, we will analyze: the cursor tracking formula `t->x = dest_x + (t->x - src_x + (t->x > 0))` for off-by-one behavior, the prompt-blanking strategy that removes content before reflow, the `HistoryBuf` circular buffer indexing during rewrap, and the pager history's deferred rewrap flag.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Data structure primer:** The reflow documentation requires explaining the `LineBuf`, `HistoryBuf`, `Line`, `CPUCell`, `GPUCell`, `CellAttrs`, and `LineAttrs` types from `kitty/data-types.h` since these are fundamental to understanding the rewrap logic.
- **Macro-parametric design explanation:** The `rewrap.h` header uses `#ifndef`-guarded macros (`BufType`, `init_src_line`, `first_dest_line`, `next_dest_line`, `is_src_line_continued`) that are overridden by `history.c` before inclusion, creating two distinct rewrap behaviors from one algorithm. This design pattern requires explicit documentation.
- **Diagram requirement:** The data flow through `screen_resize()` → `realloc_hb()` → `realloc_lb()` → `rewrap_inner()` involves multiple buffer allocations and cursor tracking state transfers that benefit from a visual sequence/flow diagram.
- **Test coverage context:** The test suite in `kitty_tests/screen.py` (`test_resize`, `test_cursor_after_resize`, `test_scrollback_fill_after_resize`) documents expected behavior for key scenarios and serves as a reference for correctness criteria.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based reStructuredText documentation system** with comprehensive user-facing documentation but **no internal developer documentation covering the terminal reflow/rewrap subsystem**.

- **Documentation framework:** Sphinx with the `furo` theme
- **Documentation generator configuration:** `docs/conf.py`
- **Documentation dependencies:** `docs/requirements.txt` (sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild)
- **Build driver:** `docs/Makefile` (provides `help`, Sphinx build targets, `develop-docs` live preview)
- **Diagram tools detected:** Mermaid diagrams used in the technical specification; no Mermaid or PlantUML configured in the Sphinx build
- **Documentation hosting:** Online changelog and website referenced from `CHANGELOG.rst` and `README.asciidoc`

**Existing documentation files relevant to the terminal internals:**

| File | Content | Relevance to Reflow |
|------|---------|---------------------|
| `docs/changelog.rst` | Release notes mentioning resize bug fixes (issues #7493, #7324, #5635, #4768) | Historical context for reflow edge cases |
| `docs/conf.rst` | Configuration option documentation | Documents `scrollback_fill_enlarged_window` and `resize_draw_strategy` options |
| `docs/performance.rst` | Performance tuning documentation | Mentions rendering pipeline but not buffer management |
| `docs/overview.rst` | High-level project introduction | No internal architecture coverage |

**Key finding:** There is no existing developer documentation that explains how the rewrap algorithm works, how the screen buffer interacts with the history buffer during resize, or how continuation state is propagated. The changelog entries reference bug fixes (#5635: "Fix cursor position at x=0 changing to x=1 on resize", #4768: improvements to "the resized buffer") that hint at edge cases in the reflow system but provide no architectural explanation.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code requiring documentation:

- **Rewrap algorithm core:** `kitty/rewrap.h` — The `rewrap_inner()` function (lines 56–96) and supporting macros/structures
- **Screen buffer rewrap integration:** `kitty/line-buf.c` — `linebuf_rewrap()` (lines 586–622), fast-path memcpy, content-line counting, `TrackCursor` array setup
- **History buffer rewrap integration:** `kitty/history.c` — `historybuf_rewrap()` (lines 595–614), macro overrides (`BufType`, `init_src_line`, `next_dest_line`, `first_dest_line`), circular buffer indexing
- **Resize entry point:** `kitty/screen.c` — `screen_resize()` (lines 346–463), `realloc_hb()` (lines 216–223), `realloc_lb()` (lines 234–242), `prevent_current_prompt_from_rewrapping()` (lines 302–343)
- **Data structures:** `kitty/data-types.h` — `CellAttrs` (lines 196–209), `GPUCell` (lines 216–220), `CPUCell` (lines 223–228), `LineAttrs` (lines 231–239), `Line` (lines 241–249), `LineBuf` (lines 252–260), `HistoryBuf` (lines 282–290), `PagerHistoryBuf` (lines 268–272)
- **Line operations:** `kitty/lineops.h` — Function declarations for `linebuf_set_last_char_as_continuation()`, `linebuf_line_ends_with_continuation()`, `history_buf_endswith_wrap()`, `copy_line()`
- **Continuation state management:** `kitty/line-buf.c` (lines 188–198), `kitty/history.c` (lines 302–307)
- **Tests:** `kitty_tests/screen.py` — `test_resize` (line 280), `test_cursor_after_resize` (line 308), `test_scrollback_fill_after_resize` (line 343)

Key directories examined:

- `kitty/` — Core C source and Python source for the terminal engine
- `kitty_tests/` — Test suite for screen and buffer behavior
- `docs/` — Existing Sphinx documentation tree
- `blitzy/documentation/` — Target output directory (does not yet exist)

### 0.2.3 Web Search Research Conducted

No external web search was conducted for this task. The analysis is entirely grounded in the source code as the authoritative truth, per the implementation rules. The reflow system is kitty-specific internal architecture with no external standard to reference beyond general terminal emulator concepts already understood from the codebase.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `kitty/rewrap.h` — Core Rewrap Algorithm**
- Public APIs / Key Constructs:
  - `rewrap_inner()` — Static function performing the actual line reflow
  - `copy_range()` — Static inline helper for cell range duplication
  - `TrackCursor` struct — Cursor position tracking during reflow
  - Configurable macros: `BufType`, `init_src_line`, `set_dest_line_attrs`, `first_dest_line`, `next_dest_line`, `is_src_line_continued`
- Current documentation: **Missing** — No documentation exists for this algorithm
- Documentation needed: Full algorithm walkthrough with annotated control flow, macro-override pattern explanation, and cursor remapping logic

**Module: `kitty/screen.c` — Resize Orchestration**
- Public APIs / Key Constructs:
  - `screen_resize()` (line 346) — Top-level resize entry point
  - `realloc_hb()` (line 216) — History buffer reallocation with rewrap
  - `realloc_lb()` (line 234) — Line buffer reallocation with rewrap
  - `prevent_current_prompt_from_rewrapping()` (line 302) — Prompt protection logic
  - `CursorTrack` struct (line 226) — Extended cursor tracking for resize
- Current documentation: **Missing** — No internal documentation for the resize flow
- Documentation needed: Complete data flow from resize entry through all buffer operations, cursor tracking, prompt protection, and scrollback fill

**Module: `kitty/line-buf.c` — LineBuf Rewrap Wrapper**
- Public APIs / Key Constructs:
  - `linebuf_rewrap()` (line 586) — Wrapper coordinating rewrap for screen buffers
  - `linebuf_set_last_char_as_continuation()` (line 194) — Continuation flag setter
  - `linebuf_line_ends_with_continuation()` (line 188) — Continuation flag reader
  - `linebuf_init_line()` (line 141) — Line initialization with continuation derivation
  - `linebuf_index()` (line 317) — Line map scroll operation used in `next_dest_line`
- Current documentation: **Missing** — Only Python-facing docstrings exist
- Documentation needed: How `linebuf_rewrap()` integrates with `rewrap_inner()`, fast-path optimization, content-line counting, and history overflow during LineBuf rewrap

**Module: `kitty/history.c` — HistoryBuf Rewrap and Macro Overrides**
- Public APIs / Key Constructs:
  - `historybuf_rewrap()` (line 595) — Wrapper coordinating rewrap for history buffers
  - `historybuf_push()` (line 276) — Push new line into circular buffer (used in overridden `next_dest_line`)
  - `history_buf_set_last_char_as_continuation()` (line 302) — Continuation flag on history lines
  - `init_line()` (line 162) — History-specific line init with continuation derivation from pager history
  - Macro overrides at lines 582–591: `BufType`, `map_src_index`, `init_src_line`, `next_dest_line`, `first_dest_line`
- Current documentation: **Missing**
- Documentation needed: How `HistoryBuf` overrides rewrap macros to use circular buffer semantics and `historybuf_push()` instead of `linebuf_index()`; pager history continuation state derivation

**Module: `kitty/data-types.h` — Core Data Structures**
- Key Constructs:
  - `CellAttrs` union with `next_char_was_wrapped` bit (line 206)
  - `LineAttrs` union with `is_continued` bit (line 233)
  - `LineBuf` struct with `line_map`, `line_attrs`, cell buffers (lines 252–260)
  - `HistoryBuf` struct with `segments`, `start_of_data`, `count` (lines 282–290)
  - `PagerHistoryBuf` struct with `ringbuf` and `rewrap_needed` flag (lines 268–272)
- Current documentation: **Missing** — Type definitions with static_asserts but no prose documentation
- Documentation needed: Structure primer showing how these types support the reflow system

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented algorithm:** The `rewrap_inner()` function in `kitty/rewrap.h` is the central reflow algorithm for the entire terminal emulator, yet has no documentation beyond a copyright header.
- **Undocumented macro-parametric design:** The header-only inclusion pattern where `history.c` redefines macros before `#include "rewrap.h"` is a critical architectural decision that is entirely undocumented.
- **Undocumented buffer interaction:** The ordered sequence in `screen_resize()` — history first, then main, then alt — and the rationale for that order (history must exist before LineBuf rewrap can overflow into it) is not documented.
- **Undocumented continuation state duality:** Two separate mechanisms track line continuation (`next_char_was_wrapped` on GPU cells and `is_continued` on LineAttrs), and the derivation logic differs between `LineBuf` and `HistoryBuf`. This is a known source of confusion and potential bugs.
- **Undocumented prompt protection:** The `prevent_current_prompt_from_rewrapping()` function implements a non-obvious strategy of blanking prompt lines before reflow and restoring them after, which directly affects reflow behavior.
- **Undocumented edge cases:** The cursor tracking formula, the pager history continuation detection, and the scrollback-fill-on-enlarge feature interact with the reflow system in ways that have caused past bugs (per changelog references).

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document placed at:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
```

The internal structure of `kitty_815df1e210e0.md` follows a top-down approach, progressing from high-level architecture to low-level algorithm details:

```
kitty_815df1e210e0.md
├── 1. Overview and Motivation
│   ├── What reflow/rewrap does
│   └── When it is triggered
├── 2. Core Data Structures
│   ├── Cell-level: CPUCell, GPUCell, CellAttrs
│   ├── Line-level: Line, LineAttrs
│   ├── Buffer-level: LineBuf, HistoryBuf, PagerHistoryBuf
│   └── Continuation state: next_char_was_wrapped vs is_continued
├── 3. Resize Entry Point: screen_resize()
│   ├── Pre-resize preparation (dummy output, prompt protection)
│   ├── History buffer reallocation (realloc_hb)
│   ├── Main line buffer reallocation (realloc_lb)
│   ├── Alternate buffer reallocation
│   ├── Post-resize cursor resolution
│   └── Scrollback fill-on-enlarge
├── 4. The Rewrap Algorithm: rewrap_inner()
│   ├── Macro-parametric design pattern
│   ├── LineBuf macro defaults
│   ├── HistoryBuf macro overrides
│   ├── Algorithm walkthrough (step-by-step)
│   └── Cursor tracking and remapping
├── 5. LineBuf Rewrap Integration
│   ├── linebuf_rewrap() wrapper
│   ├── Fast-path optimization
│   ├── Content-line counting
│   └── History overflow via next_dest_line
├── 6. HistoryBuf Rewrap Integration
│   ├── historybuf_rewrap() wrapper
│   ├── Circular buffer indexing (map_src_index)
│   ├── Pager history rewrap (deferred)
│   └── Continuation state at buffer boundary
├── 7. Continuation State Propagation Analysis
│   ├── Two-mechanism design
│   ├── LineBuf derivation path
│   ├── HistoryBuf derivation path (including pager history)
│   └── Potential issues and edge cases
├── 8. Prompt Protection During Resize
│   ├── prevent_current_prompt_from_rewrapping()
│   ├── Blanking strategy and rationale
│   └── Post-resize prompt restoration
├── 9. Edge Cases and Potential Issues
│   ├── Cursor tracking formula analysis
│   ├── Continuation state at HistoryBuf index 0
│   ├── Pager history ring buffer boundary
│   └── Wide characters and trailing blanks
└── 10. Test Coverage Summary
    ├── test_resize scenarios
    ├── test_cursor_after_resize scenarios
    └── test_scrollback_fill_after_resize scenarios
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract algorithm logic from `kitty/rewrap.h` (lines 56–96) using line-by-line code analysis
- Extract resize orchestration from `kitty/screen.c` (lines 195–463) following the execution path
- Extract macro overrides from `kitty/history.c` (lines 582–614) by comparing with default macros
- Extract data structures from `kitty/data-types.h` (lines 194–290)
- Extract continuation state logic from `kitty/line-buf.c` (lines 141–198) and `kitty/history.c` (lines 153–177)
- Generate edge case analysis by studying test expectations in `kitty_tests/screen.py` (lines 280–400)

**Template Application:**

Per the implementation rules, the document must:
- Provide thinking/rationale behind all answers
- Base all conclusions on the code as truth
- Not make assumptions

**Documentation Standards:**

- Markdown formatting with proper heading hierarchy (# through ####)
- Mermaid diagrams for the resize data flow and buffer interaction
- Code snippet citations in the format `Source: kitty/rewrap.h:56-96`
- Tables for data structure field inventories and macro comparison
- Consistent terminology: "rewrap" for the algorithm, "reflow" for the user-visible behavior, "resize" for the triggering event

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the document:

- **Flow diagram:** Complete `screen_resize()` execution path showing the ordered buffer reallocations, cursor tracking, and conditional branches
- **Sequence diagram:** Interaction between `rewrap_inner()`, `next_dest_line`, and `historybuf_push()`/`linebuf_index()` showing how lines flow from source to destination with history overflow
- **State diagram:** Continuation state propagation showing how `next_char_was_wrapped` on GPU cells maps to `is_continued` on LineAttrs, and how this differs between LineBuf and HistoryBuf
- **Block diagram:** Memory layout of LineBuf (flat arrays with line_map indirection) vs HistoryBuf (segmented circular buffer)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/rewrap.h`, `kitty/screen.c`, `kitty/line-buf.c`, `kitty/history.c`, `kitty/data-types.h`, `kitty/lineops.h`, `kitty_tests/screen.py` | Comprehensive technical deep-dive answering the user's questions about the reflow system: algorithm trace, buffer interaction, continuation state propagation, complete data flow, and edge case analysis |

This is the **only** file to be created. No existing files are to be modified or deleted, per the user's explicit instruction.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Internals / Architecture Analysis
Source Code:
  - kitty/rewrap.h (entire file, 96 lines)
  - kitty/screen.c (lines 195–463: resize flow)
  - kitty/line-buf.c (lines 135–198, 580–637: rewrap wrapper and continuation)
  - kitty/history.c (lines 150–210, 258–307, 580–625: rewrap wrapper, macro overrides, continuation)
  - kitty/data-types.h (lines 194–290: core data structures)
  - kitty/lineops.h (lines 1–130: helper declarations)
  - kitty_tests/screen.py (lines 280–400: resize tests)

Sections:
  - Overview and Motivation (purpose and trigger conditions)
  - Core Data Structures (CPUCell, GPUCell, CellAttrs, Line, LineAttrs, LineBuf, HistoryBuf)
  - Resize Entry Point: screen_resize() (from kitty/screen.c:346)
  - The Rewrap Algorithm: rewrap_inner() (from kitty/rewrap.h:56)
  - LineBuf Rewrap Integration (from kitty/line-buf.c:586)
  - HistoryBuf Rewrap Integration (from kitty/history.c:595)
  - Continuation State Propagation Analysis
  - Prompt Protection During Resize (from kitty/screen.c:302)
  - Edge Cases and Potential Issues
  - Test Coverage Summary (from kitty_tests/screen.py:280)

Diagrams:
  - Mermaid flowchart: screen_resize() execution path
  - Mermaid sequence diagram: rewrap_inner() buffer interaction
  - Mermaid diagram: continuation state propagation paths

Key Citations:
  - kitty/rewrap.h (complete algorithm)
  - kitty/screen.c (resize orchestration)
  - kitty/line-buf.c (LineBuf rewrap wrapper, continuation state)
  - kitty/history.c (HistoryBuf rewrap wrapper, circular buffer, pager history)
  - kitty/data-types.h (structure definitions)
  - kitty/lineops.h (function declarations and inline helpers)
  - kitty_tests/screen.py (behavioral expectations)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The output file is a standalone markdown document placed in `blitzy/documentation/`, which is independent of the project's Sphinx documentation build system. The existing `docs/conf.py`, `docs/Makefile`, and `docs/requirements.txt` are not modified.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content dependencies:** The new document is self-contained and does not include or reference other documentation files via transclusion.
- **Internal source citations:** All technical claims in the document are cited with source file paths and line numbers pointing into the kitty source tree.
- **No navigation/TOC updates required:** The document lives outside the Sphinx documentation tree and does not need to be registered in any navigation structure.
- **No index/glossary updates needed:** The document is standalone analysis output.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tools or packages are required for this task. The output is a plain Markdown file (`kitty_815df1e210e0.md`) that uses standard Markdown syntax with embedded Mermaid diagram blocks. No documentation site generator, API doc extractor, or diagram rendering tool needs to be installed or executed.

For reference, the existing project documentation infrastructure uses:

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| pip | sphinx | (unpinned) | Project's existing documentation site generator (not used for this task) |
| pip | furo | (unpinned) | Sphinx theme for the project's docs site (not used for this task) |
| pip | sphinx-copybutton | (unpinned) | Code block copy buttons (not used for this task) |
| pip | sphinxext-opengraph | (unpinned) | OpenGraph metadata generation (not used for this task) |
| pip | sphinx-inline-tabs | (unpinned) | Tab UI in docs (not used for this task) |
| pip | sphinx-autobuild | (unpinned) | Live preview for docs development (not used for this task) |

None of these are required for producing the output document.

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates. The new document does not replace or supersede any existing documentation, and no cross-references need to be established from the existing Sphinx documentation tree to the new analysis document.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the reflow subsystem:**

| Area | Components | Currently Documented | Coverage |
|------|-----------|---------------------|----------|
| Rewrap algorithm (`rewrap_inner`) | 1 core function, 6 macros, 2 helper types | 0/9 | 0% |
| Screen resize flow (`screen_resize`) | 1 entry point, 2 realloc helpers, 1 prompt protector, 1 cursor track struct | 0/5 | 0% |
| LineBuf rewrap wrapper | 1 wrapper function, 2 continuation helpers | 0/3 | 0% |
| HistoryBuf rewrap wrapper | 1 wrapper function, 5 macro overrides, 1 push function | 0/7 | 0% |
| Data structures (reflow-relevant) | 7 type definitions (CellAttrs, GPUCell, CPUCell, LineAttrs, Line, LineBuf, HistoryBuf) | 0/7 | 0% |
| Continuation state propagation | 4 distinct derivation paths | 0/4 | 0% |
| Edge cases and known issues | At least 4 identified scenarios | 0/4 | 0% |

**Target coverage:** 100% of the reflow-relevant components listed above will be documented in the output file, fulfilling all five user requirements (R1–R5).

**Coverage gaps to address:**
- `kitty/rewrap.h`: Currently 0% documented — target 100% (full algorithm trace)
- `kitty/screen.c` resize path: Currently 0% documented — target 100% (complete data flow)
- Continuation state propagation: Currently 0% documented — target 100% (all derivation paths and potential issues)

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every function in the resize/rewrap call chain must be explained with its purpose, inputs, outputs, and side effects
- Every data structure field relevant to reflow must be described with its type, meaning, and usage context
- Every macro in `rewrap.h` must be documented with its default behavior and its `HistoryBuf` override variant
- The cursor tracking formula must be analyzed with a concrete worked example
- At least one illustrative example per major code path (narrowing resize, widening resize, history overflow)

**Accuracy validation:**
- All code references must cite exact file paths and line numbers from the repository
- All behavioral claims must be verifiable against the code or the test suite
- Data structure sizes and layouts must match the `static_assert` checks in `kitty/data-types.h`
- No speculation — conclusions must be explicitly derived from source code

**Clarity standards:**
- Technical accuracy with accessible explanations for developers not already familiar with kitty internals
- Progressive disclosure: start with high-level overview, drill into implementation details
- Consistent terminology: "rewrap" = algorithm, "reflow" = user-visible behavior, "resize" = trigger event, "continuation" = a line that was visually wrapped from the previous line

**Maintainability:**
- Every section references its source file and line range for traceability
- The document structure mirrors the code organization for easy cross-referencing

### 0.7.3 Example and Diagram Requirements

- **Minimum code citation examples:** At least one relevant code snippet per major function analyzed
- **Diagram types required:** Mermaid flowchart (resize data flow), Mermaid sequence diagram (rewrap buffer interaction), Mermaid state/block diagram (continuation state)
- **Worked examples:** At least two concrete reflow scenarios traced through the algorithm: (1) narrowing a 5-column buffer to 3 columns with continuation, (2) widening with history line retrieval
- **Code example testing:** All examples derived directly from observed test cases in `kitty_tests/screen.py`

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole output artifact

**Source code analyzed for documentation (read-only, no modifications):**
- `kitty/rewrap.h` — Complete file: rewrap algorithm, TrackCursor struct, copy_range helper, configurable macros
- `kitty/screen.c` — Lines 195–463: `screen_resize()`, `realloc_hb()`, `realloc_lb()`, `prevent_current_prompt_from_rewrapping()`, `CursorTrack`, scrollback fill, cursor clamping
- `kitty/line-buf.c` — Lines 135–198, 580–637: `linebuf_init_line()`, `linebuf_line_ends_with_continuation()`, `linebuf_set_last_char_as_continuation()`, `linebuf_rewrap()`, fast-path optimization
- `kitty/history.c` — Lines 1–105, 150–210, 258–307, 380–432, 580–625: segmented storage, circular buffer indexing, `init_line()`, `historybuf_push()`, `pagerhist_push()`, `pagerhist_rewrap_to()`, macro overrides, `historybuf_rewrap()`
- `kitty/data-types.h` — Lines 194–290: `CellAttrs`, `GPUCell`, `CPUCell`, `LineAttrs`, `Line`, `LineBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `HistoryBuf`
- `kitty/lineops.h` — Lines 1–130: inline helpers (`copy_line`, `xlimit_for_line`, `line_is_empty`), function declarations
- `kitty_tests/screen.py` — Lines 280–400: `test_resize`, `test_cursor_after_resize`, `test_scrollback_fill_after_resize`

**Documentation topics in scope:**
- Complete `screen_resize()` data flow with all conditional branches
- `rewrap_inner()` algorithm walkthrough with macro parametrization
- LineBuf and HistoryBuf rewrap integration mechanics
- Continuation state propagation through both buffer types
- Cursor tracking and remapping during reflow
- Prompt protection strategy
- Scrollback fill-on-enlarge feature interaction with reflow
- Edge case identification and analysis
- Test coverage summary for the reflow subsystem

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be created, modified, or deleted beyond the single output document in `blitzy/documentation/`
- **Test file modifications:** No changes to `kitty_tests/screen.py` or any other test file
- **Bug fixes or refactoring:** This is a documentation/analysis exercise, not an implementation task
- **Existing documentation updates:** No changes to the Sphinx docs in `docs/**`
- **Non-reflow screen.c functionality:** The screen.c file contains ~4800 lines covering many unrelated features (cursor movement, ANSI processing, graphics, selections, etc.) — only the resize/reflow path is in scope
- **GPU rendering pipeline:** How the reflowed buffer is rendered to the screen is not in scope
- **VT parser and input handling:** How text enters the buffer before a resize is not in scope
- **Remote control resize commands:** The `resize_window` and `resize-os-window` remote control commands are not in scope (they are entry points that ultimately call `screen_resize`, but the RC layer itself is not analyzed)
- **Platform-specific GLFW resize handling:** How the GLFW backend detects and reports window size changes is not in scope
- **Configuration system:** The `resize_draw_strategy` option and its interaction with rendering during resize is not in scope
- **Alternate screen rewrap details:** While the alt buffer reallocation is noted in the data flow, the alternate screen does not use history and is a simpler case that does not require deep analysis

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not part of a generated docs site
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command:** Mermaid blocks are embedded inline in the Markdown; rendering is handled by the viewer (GitHub, VS Code, etc.)
- **Documentation deployment command:** Not applicable
- **Default format:** Markdown with embedded Mermaid diagram blocks
- **Citation requirement:** Every technical claim must reference the source file path and line number(s) from which it was derived
- **Style guide:** Follow the project's existing documentation conventions where applicable (clear, technical prose with code references); structure mirrors the code architecture
- **Documentation validation:** Verify all cited line numbers match the current codebase; ensure all Mermaid blocks use valid syntax; confirm all file paths reference existing files in the repository

### 0.9.2 Output File Naming

Per the project's implementation rules, the output document is named after the source branch:

- **Branch name:** `kitty_815df1e210e0`
- **Output path:** `blitzy/documentation/kitty_815df1e210e0.md`

### 0.9.3 Content Constraints

- The document must comprehensively answer all questions posed in the user's prompt
- The document must provide thinking and rationale behind all answers
- All conclusions must be grounded in the actual source code, with no assumptions
- No existing files in the source repository may be modified

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and project implementation directives:

- **"Don't create or modify any files"** in the source repository. The only permissible file creation is the output document at `blitzy/documentation/kitty_815df1e210e0.md`, as required by the project's implementation rules.
- **"Do not modify any existing files in the source repository."** — Per the SWE-AtlasQnA-Repo implementation rule. This rule applies to all files in the kitty source tree including docs, source code, tests, and configuration.
- **"Provide thinking / rationale behind the answers."** — Every conclusion in the document must include the reasoning process that led to it, not just the conclusion itself.
- **"Do not make assumptions, base your answers on the code as the truth."** — All claims must be directly traceable to specific lines of source code. Speculative statements must be clearly labeled as potential issues rather than definitive bugs.
- **"Create a new markdown document named `<source_branch_name>.md`"** — The output file must be named `kitty_815df1e210e0.md`.
- **"Place the generated document in the `blitzy/documentation` directory"** — The output directory must be created if it does not exist.
- **Source code citations required:** All technical details must include file path and line number references to enable verification against the codebase.
- **Mermaid diagrams for complex flows:** The resize data flow and buffer interaction patterns must be visualized with Mermaid diagrams embedded in the Markdown output.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were examined to derive all conclusions in this Agent Action Plan:

**Core reflow/rewrap source files:**

| File | Lines Examined | Key Content |
|------|---------------|-------------|
| `kitty/rewrap.h` | 1–96 (complete) | `rewrap_inner()` algorithm, `TrackCursor` struct, `copy_range()` helper, configurable macros (`BufType`, `init_src_line`, `set_dest_line_attrs`, `first_dest_line`, `next_dest_line`, `is_src_line_continued`) |
| `kitty/screen.c` | 55–470 | `screen_resize()` (line 346), `realloc_hb()` (line 216), `realloc_lb()` (line 234), `prevent_current_prompt_from_rewrapping()` (line 302), `CursorTrack` struct (line 226), Screen constructor (line 102), scrollback fill (line 428) |
| `kitty/line-buf.c` | 135–198, 300–340, 439–465, 580–641 | `linebuf_init_line()` (line 141), `linebuf_line_ends_with_continuation()` (line 188), `linebuf_set_last_char_as_continuation()` (line 194), `linebuf_rewrap()` (line 586), `linebuf_index()` (line 317), `linebuf_clear_line()` (line 300), `linebuf_copy_line_to()` (line 439) |
| `kitty/history.c` | 1–105, 150–210, 245–310, 380–432, 580–625 | Segment allocation (line 18), `index_of()` (line 153), `init_line()` (line 162), `historybuf_push()` (line 276), `historybuf_add_line()` (line 287), `historybuf_pop_line()` (line 294), `history_buf_set_last_char_as_continuation()` (line 302), `pagerhist_push()` (line 258), `pagerhist_rewrap_to()` (line 392), macro overrides (lines 582–592), `historybuf_rewrap()` (line 595) |
| `kitty/data-types.h` | 194–310 | `CellAttrs` (line 196), `GPUCell` (line 216), `CPUCell` (line 223), `LineAttrs` (line 231), `Line` (line 241), `LineBuf` (line 252), `HistoryBufSegment` (line 262), `PagerHistoryBuf` (line 268), `HistoryBuf` (line 282), `Cursor` (line 292) |
| `kitty/lineops.h` | 1–130 | `copy_line()` (line 25), `xlimit_for_line()` (line 39), `line_is_empty()` (line 76), function declarations for all buffer operations |

**Test files:**

| File | Lines Examined | Key Content |
|------|---------------|-------------|
| `kitty_tests/screen.py` | 280–400 | `test_resize` (line 280), `test_cursor_after_resize` (line 308), `test_scrollback_fill_after_resize` (line 343) |

**Documentation infrastructure files:**

| File | Purpose |
|------|---------|
| `docs/` (folder) | Existing Sphinx documentation tree — examined for reflow-related content (none found) |
| `docs/conf.py` | Sphinx configuration — examined for documentation generation setup |
| `docs/requirements.txt` | Documentation dependencies: sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild |
| `docs/changelog.rst` | Release notes — identified historical reflow bug references (issues #7493, #7324, #5635, #4768) |
| `docs/Makefile` | Documentation build system |

**Project metadata files:**

| File | Purpose |
|------|---------|
| `README.asciidoc` | Project overview |
| `CONTRIBUTING.md` | Contribution guidelines |
| `setup.py` | Build configuration |

**Folders explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| `` (root) | Level 0 | Project structure and entry points |
| `kitty/` | Level 1 | Core C/Python source for terminal engine |
| `kitty_tests/` | Level 1 | Test suite |
| `docs/` | Level 1 | Sphinx documentation tree |
| `docs/kittens/` | Level 2 | Kitten-specific documentation |
| `docs/_static/` | Level 2 | Documentation CSS/JS assets |
| `docs/_templates/` | Level 2 | Documentation Jinja templates |

### 0.11.2 Attachments

No attachments were provided by the user for this task.

### 0.11.3 Figma Screens

No Figma screens were provided for this task.

### 0.11.4 External References

No external URLs or web resources were referenced. All analysis is derived entirely from the repository source code.

### 0.11.5 Tech Spec Sections Consulted

The following technical specification sections were retrieved for background context:

- **1.1 Executive Summary** — Project overview confirming kitty is a GPU-accelerated terminal emulator with C/Python/Go architecture
- **4.3 Terminal Input/Output Pipeline** — VT parser and screen model architecture confirming the role of `screen.c`, `line-buf.c`, and `history.c`
- **5.2 Component Details** — Component architecture confirming the VT Parser, Screen Model, and History Buffer roles within the system

