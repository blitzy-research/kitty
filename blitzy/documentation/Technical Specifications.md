# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a deep, code-grounded explanation of how the kitty terminal emulator handles complex Unicode at runtime under extreme spatial constraints.

- **Documentation Type:** Technical deep-dive / Q&A analysis document
- **Category:** Create new documentation
- **Target Audience:** Developers and terminal application authors who need to understand kitty's internal Unicode processing pipeline

The user's questions decompose into the following discrete documentation requirements:

- **Requirement 1 — ZWJ Emoji Screen Buffer Behavior:** When the terminal receives a stream of zero-width joiners (U+200D) forming a multi-codepoint emoji (e.g., a family emoji like 👨‍👩‍👧‍👦), document how the internal screen buffer decides what to keep when the terminal has almost no available space (such as a single 1×1 cell). Trace the exact C-level code path from `draw_text_loop()` through `draw_combining_char()` and `line_add_combining_char()` to explain the final cell state.
- **Requirement 2 — Settled Cell State Under Constraint:** After the emoji sequence has been fully processed, document what the terminal believes is actually present in the final cell. Describe the `CPUCell.ch`, `CPUCell.cc_idx[3]`, and `GPUCell.attrs.width` values that result from the processing pipeline, explaining how codepoints beyond the three combining-character slots are handled.
- **Requirement 3 — State Reporting via Control Sequences:** When the terminal is asked to report part of its current state through a control sequence query (e.g., DSR for cursor position, DECRQSS for settings, or `as_text` for screen content extraction), document the exact response generated and how that response reflects the grapheme handling decisions made during the earlier write phase.
- **Requirement 4 — Interaction of Normalization, Grapheme Breaking, and State Reporting:** Provide an integrated explanation of how kitty's character classification (`is_combining_char`), grapheme segmentation (the cell-based model with `cc_idx` slots), and state extraction (`cell_as_unicode`, `unicode_in_range`, `line_as_ansi`) interact when the terminal is under extreme size constraints.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL Directive — Repository Immutability:** The user explicitly requires that the repository itself remain unchanged. Temporary scripts may be used for observation purposes, but must be cleaned up afterward. The implementation rule further states: "Do not modify any existing files in the source repository."
- **CRITICAL Directive — Output Location:** Per the implementation rule, the deliverable is a single markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory.
- **Content Mandate:** The document must provide thinking and rationale behind all answers, and must not make assumptions — all answers must be grounded in the code as the source of truth.
- **Style Requirements:** The documentation should be a self-contained markdown Q&A analysis with code citations, diagrams, and detailed technical explanations that trace behavior through the actual source files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document ZWJ emoji buffer behavior, we will create a new analysis document that traces the code path through `kitty/screen.c:draw_text_loop()` → `is_combining_char()` (in `kitty/unicode-data.c`) → `draw_combining_char()` → `line_add_combining_char()` (in `kitty/line.c`), referencing the `CPUCell`/`GPUCell` structures defined in `kitty/data-types.h`.
- To document the settled cell state, we will analyze the `cc_idx[3]` combining character storage mechanism, the `mark_for_codepoint()`/`codepoint_for_mark()` index mapping in `kitty/unicode-data.c`, and the overflow behavior when more than three combining characters are attached to a single cell.
- To document state reporting, we will trace the `cell_as_unicode()`, `cell_as_utf8()`, `unicode_in_range()`, `line_as_ansi()` extraction paths in `kitty/line.c`, and the DSR/DECRQSS reporting paths in `kitty/screen.c`.
- To document the integrated interaction under constraints, we will synthesize findings from the width-calculation logic in `kitty/wcswidth.c`, the emoji presentation handling in `kitty/screen.c:draw_combining_char()` (VS15/VS16 logic), and the text extraction pipeline.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Cell Structure Primer:** The document must explain the `CPUCell` (12 bytes: `ch` + `hyperlink_id` + `cc_idx[3]`) and `GPUCell` (20 bytes: colors + sprite indices + `CellAttrs` with 2-bit `width`) dual-structure per cell, as this is foundational to understanding all grapheme behavior.
- **Mark Index System:** The combining character storage uses an indirection table (`mark_for_codepoint()` / `codepoint_for_mark()`) with 6,425 entries, mapping full codepoints to compact `uint16_t` indices. This must be explained to understand what `cc_idx` values actually mean.
- **Wide Character Padding Cells:** When a wide (width=2) character occupies a cell, the next cell has `ch=0` and `width=0`. The `draw_combining_char` function redirects combining characters aimed at these padding cells back to the base cell. This implicit behavior must be documented.
- **Edge Case: Extreme Terminal Sizes:** The behavior when `columns=1` or `columns=2` and a width-2 emoji is written involves wrap/no-wrap logic that produces different outcomes depending on the DECAWM mode setting.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx-based documentation infrastructure with comprehensive coverage of protocols and user-facing features, but limited internal architecture documentation for Unicode processing internals.

- **Documentation Framework:** Sphinx (configured in `docs/conf.py`), using the Furo theme
- **Documentation Generator Configuration:** `docs/conf.py` (Python-based Sphinx configuration importing `kitty.constants` and `kitty.conf.types`)
- **Documentation Dependencies:** Listed in `docs/requirements.txt`:
  - `sphinx`
  - `furo`
  - `sphinx-copybutton`
  - `sphinxext-opengraph`
  - `sphinx-inline-tabs`
  - `sphinx-autobuild`
- **Source Format:** reStructuredText (`.rst`) with custom roles and directives defined in `docs/conf.py`
- **Diagram Tools:** No Mermaid or PlantUML integration detected in the existing documentation pipeline; diagrams would be a new addition via the markdown document
- **Documentation Hosting:** Published to the kitty website via `publish.py` which handles docs building and upload

**Key documentation files examined:**

| File | Content |
|------|---------|
| `docs/index.rst` | Root index linking all documentation sections |
| `docs/protocol-extensions.rst` | Terminal protocol extensions documentation |
| `docs/keyboard-protocol.rst` | Kitty keyboard protocol specification |
| `docs/graphics-protocol.rst` | Kitty graphics protocol specification |
| `docs/faq.rst` | Frequently asked questions |
| `docs/conf.py` | Sphinx configuration with custom lexers and roles |
| `docs/glossary.rst` | Project glossary |
| `docs/performance.rst` | Performance documentation |
| `CONTRIBUTING.md` | Contributor guidelines |
| `README.asciidoc` | Project introduction (AsciiDoc format) |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for identifying code relevant to this documentation task:

- **Character Processing Pipeline:** `kitty/screen.c` (4,932 lines) — `draw_text_loop()`, `draw_combining_char()`, `draw_second_flag_codepoint()`, `move_widened_char()`
- **Cell Data Structures:** `kitty/data-types.h` (438 lines) — `CPUCell`, `GPUCell`, `CellAttrs`, `Line`, `LineBuf`
- **Line Operations:** `kitty/line.c` (1,003 lines) — `line_add_combining_char()`, `cell_as_unicode()`, `cell_as_utf8()`, `unicode_in_range()`, `line_as_ansi()`, `cell_text()`
- **Line Operation Headers:** `kitty/lineops.h` (136 lines) — Function declarations for all line operations
- **Unicode Data Tables:** `kitty/unicode-data.c` (3,088 lines) — `is_combining_char()`, `is_ignored_char()`, `mark_for_codepoint()`, `codepoint_for_mark()`
- **Unicode Data Declarations:** `kitty/unicode-data.h` (84 lines) — `VS15`, `VS16` constants, combining type declarations
- **Width Calculation:** `kitty/wcswidth.c` (149 lines) — `wcswidth_step()`, `wcswidth_string()` with emoji presentation awareness
- **Width Standard Tables:** `kitty/wcwidth-std.h` — `wcwidth_std()`, `is_emoji_presentation_base()`
- **Emoji Classification:** `kitty/emoji.h` (790 lines) — `is_emoji()` classification function
- **Screen Header:** `kitty/screen.h` — Screen structure and function declarations
- **VT Parser:** `kitty/vt-parser.c` — Escape sequence dispatch including `screen_request_capabilities`
- **Window Python Layer:** `kitty/window.py` — `as_text()`, `request_capabilities()`, `text_for_selection()`
- **Test Files:**
  - `kitty_tests/screen.py` — `test_zwj()`, `test_emoji_skin_tone_modifiers()`, `test_variation_selectors()`, `test_regional_indicators()`
  - `kitty_tests/datatypes.py` — `add_combining_char` tests for combining character overflow
- **Code Generation:** `gen/wcwidth.py` — Generates `wcwidth-std.h`, `unicode-data.c`, emoji tables from Unicode Standard 15.0.0 data files

### 0.2.3 Web Search Research Conducted

No external web searches were required for this task. All necessary information is directly available in the source code, which the user explicitly identified as the ground truth. The Unicode Standard version (15.0.0) is documented in the generated code header of `kitty/emoji.h`.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage for this task:

- **Module: `kitty/screen.c` — Character Drawing Engine**
  - Public APIs: `screen_draw_text()`, `draw_text_loop()`, `draw_combining_char()`, `draw_second_flag_codepoint()`, `move_widened_char()`, `continue_to_next_line()`
  - Current documentation: No internal documentation exists for the character-to-cell pipeline; only the test suite (`kitty_tests/screen.py`) provides behavioral assertions
  - Documentation needed: Detailed code-path trace for ZWJ sequences, width overflow handling, wrap behavior under space constraints

- **Module: `kitty/data-types.h` — Cell Data Structures**
  - Public APIs: `CPUCell`, `GPUCell`, `CellAttrs`, `Line`, `LineBuf`, `BLANK_CHAR`, `COPY_CELL`
  - Current documentation: In-code struct definitions with static assertions but no prose documentation
  - Documentation needed: Structural explanation of the dual-cell model, bit-field layout of `CellAttrs.width`, combining character index array semantics

- **Module: `kitty/line.c` — Line Operations and Text Extraction**
  - Public APIs: `line_add_combining_char()`, `cell_as_unicode()`, `cell_as_utf8()`, `unicode_in_range()`, `line_as_ansi()`, `cell_text()`, `line_as_unicode()`
  - Current documentation: Docstrings for Python-exposed methods only
  - Documentation needed: Combining character overflow behavior, text reconstruction pipeline from cells, padding-cell redirect logic

- **Module: `kitty/unicode-data.c` — Unicode Classification and Mark Tables**
  - Public APIs: `is_combining_char()`, `is_ignored_char()`, `mark_for_codepoint()`, `codepoint_for_mark()`
  - Current documentation: Generated code with minimal comments
  - Documentation needed: How ZWJ (U+200D) is classified, the mark indirection table mechanics, the 6,425-entry mapping array

- **Module: `kitty/wcswidth.c` — Width Calculation**
  - Public APIs: `wcswidth_step()`, `wcswidth_string()`
  - Current documentation: None beyond function signatures
  - Documentation needed: How width interacts with emoji presentation selectors

- **Module: `kitty/screen.c` — State Reporting**
  - Public APIs: `report_device_status()`, `screen_request_capabilities()`, `screen_report_size()`, `as_text()`, `as_text_non_visual()`
  - Current documentation: Inline comments referencing VT specs
  - Documentation needed: What responses are generated for DSR/DECRQSS when cells contain complex graphemes

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internal behavior:** No existing documentation explains how ZWJ sequences are stored in the cell model, how the 3-slot combining character limit affects grapheme fidelity, or how text extraction reconstructs multi-codepoint sequences from the compact cell representation
- **Missing width-constraint analysis:** No documentation covers the interaction between wide character width requirements (2 cells) and terminals with fewer than 2 columns
- **Absent state-reporting documentation for Unicode edge cases:** The existing protocol documentation (`docs/protocol-extensions.rst`) covers the wire protocol but not how internal cell state maps to query responses when cells contain complex graphemes
- **No cross-cutting Unicode processing documentation:** The character classification → storage → extraction pipeline spans 6+ source files with no unified narrative documentation

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive markdown document. The planned hierarchy is:

```
blitzy/documentation/
└── kitty_815df1e210e0.md
    ├── Introduction / Context
    ├── Question 1: ZWJ Emoji in the Screen Buffer
    │   ├── Character Classification Pipeline
    │   ├── The Cell Storage Model (CPUCell + GPUCell)
    │   ├── The Combining Character Index System
    │   ├── Code-Path Trace: Family Emoji Sequence
    │   └── Behavior Under Extreme Constraints (1×1, 2×1)
    ├── Question 2: What the Terminal Thinks Is in the Cell
    │   ├── Settled CPUCell and GPUCell State
    │   ├── The Three-Slot Combining Character Limit
    │   └── Overflow Semantics
    ├── Question 3: State Reporting via Control Sequences
    │   ├── Text Extraction: cell_as_unicode / unicode_in_range
    │   ├── ANSI Serialization: line_as_ansi
    │   ├── Cursor Position Reporting: DSR (CSI 6 n)
    │   ├── Setting Queries: DECRQSS (DCS $ q)
    │   └── Remote Control: as_text via Python Layer
    ├── Question 4: Normalization, Grapheme Breaking, and Reporting Interaction
    │   ├── Classification vs. Segmentation vs. Extraction
    │   ├── Lossy Fidelity Under Constraint
    │   └── Integrated Pipeline Diagram
    └── Summary and Key Findings
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract cell structure definitions from `kitty/data-types.h:196-228` (CPUCell, GPUCell, CellAttrs)
- Trace the character drawing pipeline from `kitty/screen.c:763-845` (draw_text_loop)
- Analyze combining character handling from `kitty/line.c:457-467` (line_add_combining_char)
- Map the mark indirection system from `kitty/unicode-data.c:2749-2800` (codepoint_for_mark, mark_for_codepoint)
- Extract text reconstruction logic from `kitty/line.c:199-207` (cell_as_unicode) and `kitty/line.c:252-279` (unicode_in_range)
- Document state reporting from `kitty/screen.c:2178-2200` (report_device_status) and `kitty/screen.c:2446-2480` (screen_request_capabilities)
- Validate behavior against existing tests in `kitty_tests/screen.py:123-135` (test_zwj)

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for pipeline visualization
- Code citations using format: `Source: kitty/screen.c:LINE_NUMBER`
- Short inline code snippets for critical data structure fields
- Tables for parameter descriptions and state mappings

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the document:

- **Character Processing Pipeline:** Flowchart showing the path from byte input through VT parser → `draw_text_loop` → character classification → combining char storage or direct cell write
- **Cell Memory Layout:** Diagram showing `CPUCell` (ch + cc_idx[3]) and `GPUCell` (attrs.width + colors) paired per column
- **ZWJ Emoji Trace Diagram:** Step-by-step cell state evolution as each codepoint of a family emoji is processed
- **Text Extraction Pipeline:** Flowchart from `cell_as_unicode` through `unicode_in_range` to `line_as_ansi` showing how stored cell data is reconstructed into Unicode text

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/screen.c`, `kitty/line.c`, `kitty/data-types.h`, `kitty/unicode-data.c`, `kitty/unicode-data.h`, `kitty/wcswidth.c`, `kitty/wcwidth-std.h`, `kitty/emoji.h`, `kitty/screen.h`, `kitty/lineops.h`, `kitty/window.py`, `kitty/vt-parser.c`, `kitty_tests/screen.py`, `kitty_tests/datatypes.py` | Complete Q&A analysis document answering all four user questions about ZWJ emoji handling, cell state, state reporting, and the interaction of normalization/grapheme-breaking/reporting under extreme constraints |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Deep-Dive / Q&A Analysis
Source Code Files:
    - kitty/data-types.h (CPUCell, GPUCell, CellAttrs definitions)
    - kitty/screen.c (draw_text_loop, draw_combining_char, report_device_status, screen_request_capabilities)
    - kitty/line.c (line_add_combining_char, cell_as_unicode, unicode_in_range, line_as_ansi)
    - kitty/unicode-data.c (is_combining_char, mark_for_codepoint, codepoint_for_mark)
    - kitty/unicode-data.h (VS15/VS16 mark constants)
    - kitty/wcswidth.c (wcswidth_step, emoji presentation width adjustment)
    - kitty/wcwidth-std.h (wcwidth_std, is_emoji_presentation_base)
    - kitty/emoji.h (is_emoji classification)
    - kitty/lineops.h (line operation declarations, xlimit_for_line)
    - kitty/screen.h (Screen structure declaration)
    - kitty/window.py (as_text, request_capabilities Python layer)
    - kitty/vt-parser.c (escape sequence dispatch to screen_request_capabilities)
    - kitty_tests/screen.py (test_zwj, test_variation_selectors, test_emoji_skin_tone_modifiers)
    - kitty_tests/datatypes.py (combining character overflow tests)
    - gen/wcwidth.py (Unicode data generation from Unicode 15.0.0)
Sections:
    - Introduction and Context
    - Cell Storage Model (CPUCell/GPUCell dual structure, CellAttrs.width 2-bit field, cc_idx[3] slots)
    - Character Classification Pipeline (is_combining_char including ZWJ at U+200D, is_ignored_char, wcwidth_std)
    - Mark Indirection Table (mark_for_codepoint/codepoint_for_mark with 6,425-entry lookup)
    - ZWJ Emoji Processing Trace (step-by-step trace through draw_text_loop for family emoji)
    - Extreme Constraint Behavior (1×1 and 2×1 terminal scenarios with DECAWM on/off)
    - Settled Cell State Analysis (final ch, cc_idx, width values after processing)
    - Three-Slot Combining Character Overflow (last-slot overwrite behavior)
    - Text Extraction Pipeline (cell_as_unicode → unicode_in_range → line_as_ansi)
    - Control Sequence State Reporting (DSR cursor position, DECRQSS settings, as_text content)
    - Integrated Pipeline: Classification → Storage → Extraction
    - Summary and Key Findings
Diagrams:
    - Character processing pipeline flowchart
    - Cell memory layout diagram
    - Step-by-step ZWJ emoji cell evolution
    - Text extraction pipeline flowchart
Key Citations:
    - kitty/data-types.h:196-228
    - kitty/screen.c:663-700, 763-845
    - kitty/line.c:457-467, 199-207, 252-279
    - kitty/unicode-data.c:11-40, 2749-2800
    - kitty/unicode-data.h:5
    - kitty/wcswidth.c:23-119
    - kitty/screen.c:2178-2200, 2446-2480
    - kitty_tests/screen.py:123-135, 592-605
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The deliverable is a standalone markdown file that does not integrate into the existing Sphinx documentation pipeline. It is placed in `blitzy/documentation/` per the implementation rules, which is a separate output directory from the repository's `docs/` tree.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tooling packages are required for this task. The deliverable is a plain Markdown file that requires no build step, generator, or rendering pipeline.

For reference, the existing project documentation infrastructure uses the following packages (from `docs/requirements.txt`), which are **not** dependencies of this task:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | (project-managed) | Existing project documentation site generator |
| pip | furo | (project-managed) | Sphinx theme for existing docs |
| pip | sphinx-copybutton | (project-managed) | Copy button for code blocks in existing docs |
| pip | sphinxext-opengraph | (project-managed) | OpenGraph metadata for existing docs |
| pip | sphinx-inline-tabs | (project-managed) | Tabbed content in existing docs |
| pip | sphinx-autobuild | (project-managed) | Auto-rebuild for existing docs development |

### 0.6.2 Project Runtime Context

The following project metadata is relevant for understanding the codebase context documented in the deliverable:

| Item | Value | Source |
|------|-------|--------|
| Python requirement | >= 3.8 | `pyproject.toml` |
| Unicode Standard version | 15.0.0 | `kitty/emoji.h` header comment |
| Go module | `kitty` | `go.mod` |
| License | GPLv3 | `LICENSE` |
| Source branch | `kitty_815df1e210e0` | Git branch |

### 0.6.3 Documentation Reference Updates

No link updates are required. The new document is a standalone analysis file in `blitzy/documentation/` that does not modify or cross-reference any existing documentation files in the repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the Unicode processing pipeline (pre-task):**

| Area | Documented | Total | Coverage |
|------|-----------|-------|----------|
| Cell data structure internals (`CPUCell`, `GPUCell`, `CellAttrs`) | 0 | 3 | 0% |
| Character classification functions (`is_combining_char`, `is_ignored_char`, `is_emoji_presentation_base`) | 0 | 3 | 0% |
| Combining character storage mechanics (`line_add_combining_char`, `cc_idx`, mark tables) | 0 | 3 | 0% |
| Drawing pipeline functions (`draw_text_loop`, `draw_combining_char`, `move_widened_char`) | 0 | 3 | 0% |
| Text extraction functions (`cell_as_unicode`, `unicode_in_range`, `line_as_ansi`) | 0 | 3 | 0% |
| State reporting functions (`report_device_status`, `screen_request_capabilities`, `as_text`) | 0 | 3 | 0% |
| Width handling under constraints (DECAWM wrap, 1-column edge cases) | 0 | 1 | 0% |
| **Total** | **0** | **19** | **0%** |

**Target coverage:** 100% of the areas listed above will be documented in the new file, achieving full coverage for the user's four questions.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every claim in the document must reference a specific source file and line number
- All four user questions must be answered with detailed code-path traces
- The combining character overflow behavior (3-slot limit with last-slot overwrite) must be explained with concrete examples
- Edge cases for 1-column and 2-column terminals must be traced through the exact conditional branches in `draw_text_loop()`

**Accuracy validation:**

- All code references must cite actual function names, variable names, and line numbers from the inspected source files
- Behavioral assertions must be cross-validated against the existing test suite in `kitty_tests/screen.py` (e.g., `test_zwj` asserting `cursor.x == 8` for the family emoji on a 20-column screen)
- Data structure sizes must match the `static_assert` declarations in `kitty/data-types.h` (`CPUCell` = 12 bytes, `GPUCell` = 20 bytes)

**Clarity standards:**

- Technical accuracy with code-level precision
- Progressive disclosure: start with the cell model, then classification, then drawing, then extraction
- Each question answered in a self-contained section with cross-references to shared concepts
- Mermaid diagrams for all pipeline flows

### 0.7.3 Example and Diagram Requirements

- Minimum 1 worked example per question: trace the family emoji 👨‍👩‍👧‍👦 (`U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466`) through the pipeline
- Required diagram types: flowchart (character processing pipeline), structure diagram (cell layout), sequence trace (codepoint-by-codepoint cell evolution), flowchart (text extraction)
- All examples grounded in the actual code with citations

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive Q&A analysis document

**Source code files analyzed for documentation content (read-only):**
- `kitty/data-types.h` — Cell structure definitions (CPUCell, GPUCell, CellAttrs, Line, LineBuf)
- `kitty/screen.c` — Character drawing pipeline, state reporting, screen operations
- `kitty/screen.h` — Screen structure and function declarations
- `kitty/line.c` — Line operations, combining char handling, text extraction
- `kitty/lineops.h` — Line operation function declarations
- `kitty/unicode-data.c` — Character classification, mark indirection tables
- `kitty/unicode-data.h` — VS15/VS16 constants, combining type declarations
- `kitty/wcswidth.c` — Width calculation with emoji presentation awareness
- `kitty/wcwidth-std.h` — Standard width tables, is_emoji_presentation_base
- `kitty/emoji.h` — Emoji classification tables
- `kitty/window.py` — Python-layer text extraction (as_text, request_capabilities)
- `kitty/vt-parser.c` — Escape sequence dispatch
- `kitty_tests/screen.py` — ZWJ, emoji, and variation selector tests
- `kitty_tests/datatypes.py` — Combining character tests
- `gen/wcwidth.py` — Unicode data generation script

**Topics covered:**
- ZWJ emoji processing in the screen buffer
- Cell state after complex grapheme processing
- Control sequence state reporting reflecting grapheme handling
- Interaction of normalization, grapheme breaking, and state reporting under extreme constraints
- Cell memory layout and combining character storage model
- Mark indirection table mechanics
- Wide character wrapping and padding cell behavior
- Text extraction and ANSI serialization from cell data

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No changes to any file in the kitty repository (per user instruction: "the repository itself should remain unchanged")
- **Test file modifications:** No changes to test files
- **Existing documentation updates:** No changes to any file in the `docs/` directory
- **Feature additions or code refactoring:** Not applicable to this documentation task
- **Deployment configuration changes:** Not applicable
- **Font rendering pipeline:** While `kitty/fonts.c` and `kitty/freetype.c` handle glyph rendering for emoji, the user's questions focus on the screen buffer and state reporting layers, not the GPU rendering of glyphs
- **Graphics protocol:** The kitty graphics protocol (`kitty/graphics.c`) is unrelated to Unicode text cell handling
- **Shell integration:** Shell integration scripts in `shell-integration/` are unrelated
- **Keyboard protocol:** The keyboard protocol (`kitty/key_encoding.c`) is orthogonal to character drawing and state reporting for graphemes
- **Build system and packaging:** `setup.py`, `Makefile`, `bypy/` are unrelated to documentation content
- **Go tooling:** The `tools/` directory contains Go-based utilities unrelated to the C-level Unicode processing pipeline

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file requiring no build step
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and rendered by any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Documentation deployment command:** Not applicable — the file is placed directly into the output directory
- **Default format:** Markdown (`.md`) with embedded Mermaid diagrams in fenced code blocks
- **Citation requirement:** Every technical claim must reference a specific source file and line range using the format `Source: path/to/file.c:LINE_START-LINE_END`
- **Style guide:** Self-contained technical analysis document with:
  - Clear section headers for each question
  - Code-grounded rationale for all answers
  - Progressive disclosure (fundamentals → specifics → edge cases)
  - Mermaid diagrams for all pipelines and state transitions
- **Documentation validation:** Manual review to ensure all four user questions are fully answered with code citations

### 0.9.2 File Naming and Placement

Per the implementation rule `SWE-AtlasQnA-Repo`:
- **File name:** `kitty_815df1e210e0.md` (derived from the source branch name `kitty_815df1e210e0`)
- **Directory:** `blitzy/documentation/`
- **Full path:** `blitzy/documentation/kitty_815df1e210e0.md`

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The analysis document must be created as a new file only. No existing `.c`, `.h`, `.py`, `.rst`, or any other file may be changed.
- **Provide thinking and rationale behind the answers.** Every answer must explain the "why" behind the behavior by tracing through the actual code, not by stating conclusions without evidence.
- **Do not make assumptions — base all answers on the code as the truth.** All behavioral claims must be grounded in specific source file references. Do not speculate about behavior that cannot be verified from the code.
- **Temporary scripts may be used for observation but must be cleaned up afterward.** If any temporary exploration scripts are created during analysis, they must not persist in the final state.
- **Place the generated document in the `blitzy/documentation` directory.** The file must be at `blitzy/documentation/kitty_815df1e210e0.md` — not in the repository's existing `docs/` folder.
- **Name the document `kitty_815df1e210e0.md`** using the source branch name as specified by the implementation rule.
- **Include source code citations for all technical details.** Every claim about internal behavior must cite the source file path and relevant line numbers.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were examined during analysis to derive the conclusions in this Agent Action Plan:

**Core Character Processing Files:**

| File Path | Purpose | Key Functions/Structures Examined |
|-----------|---------|-----------------------------------|
| `kitty/data-types.h` | Cell data structure definitions | `CPUCell` (L223-228), `GPUCell` (L216-221), `CellAttrs` (L196-209), `Line` (L241-249), `LineBuf` (L252-260), `BLANK_CHAR` (L115), `char_type` (L57), `combining_type` (L62) |
| `kitty/screen.c` | Screen operations and drawing pipeline | `draw_text_loop` (L763-845), `draw_combining_char` (L663-700), `draw_text` (L849-863), `text_loop_state` (L517-521), `zero_cells` (L657-660), `move_widened_char` (L575-596), `continue_to_next_line` (L524-528), `report_device_status` (L2178-2200), `screen_request_capabilities` (L2446-2480), `screen_report_size` (L2142-2167), `draw_second_flag_codepoint` (L638-654), `as_text` (L3485-3486) |
| `kitty/screen.h` | Screen structure and function declarations | Screen struct definition, function prototypes |
| `kitty/line.c` | Line operations and text extraction | `line_add_combining_char` (L457-467), `cell_as_unicode` (L199-207), `cell_as_utf8` (L222-231), `unicode_in_range` (L252-279), `line_as_ansi` (L338-413), `cell_text` (L41-49), `line_set_char` (L668-685), `line_as_unicode` (L282-284), `line_get_char` (L661-665) |
| `kitty/lineops.h` | Line operation function declarations | `line_add_combining_char` (L90), `cell_as_unicode` (L96), `cell_as_utf8` (L98), `xlimit_for_line` (L39-47) |
| `kitty/unicode-data.c` | Unicode character classification and mark tables | `is_combining_char` (L11-670 — covers U+200B-200F range including ZWJ at L323), `is_ignored_char` (L671+), `codepoint_for_mark` (L2749-2753, 6425-entry static array), `mark_for_codepoint` (L2755+) |
| `kitty/unicode-data.h` | Combining type declarations | `VS15 = 1364`, `VS16 = 1365` (L5), function declarations for `is_combining_char`, `codepoint_for_mark`, `mark_for_codepoint` |
| `kitty/wcswidth.c` | Width calculation with state machine | `wcswidth_step` (L23-119, emoji presentation selector handling at L46-58), `wcswidth_string` (L121-128) |
| `kitty/wcwidth-std.h` | Standard character width tables | `wcwidth_std` function, `is_emoji_presentation_base` (L2942+) |
| `kitty/emoji.h` | Emoji classification tables | `is_emoji` function (auto-generated from Unicode 15.0.0) |

**State Reporting and Python Layer Files:**

| File Path | Purpose | Key Functions/Structures Examined |
|-----------|---------|-----------------------------------|
| `kitty/window.py` | Python-layer window operations | `as_text` (L363-394), `request_capabilities` (L1275-1277), `text_for_selection` (L1542+) |
| `kitty/vt-parser.c` | VT escape sequence parser and dispatcher | `screen_request_capabilities` dispatch (L628-631) |

**Test Files:**

| File Path | Purpose | Key Tests Examined |
|-----------|---------|---------------------|
| `kitty_tests/screen.py` | Screen operation tests | `test_zwj` (L123-135: family emoji → cursor at x=8 on 20-col screen), `test_emoji_skin_tone_modifiers` (L105-110), `test_variation_selectors` (L592-605: VS15/VS16 width toggling), `test_regional_indicators` (L112-121), `test_writing_with_cursor_on_trailer_of_wide_character` (L607-620) |
| `kitty_tests/datatypes.py` | Data type unit tests | Combining character `add_combining_char` tests (L178, L202-208) |

**Code Generation Files:**

| File Path | Purpose |
|-----------|---------|
| `gen/wcwidth.py` | Generates `wcwidth-std.h` and `unicode-data.c` from Unicode 15.0.0 data files |

**Project Metadata Files:**

| File Path | Purpose |
|-----------|---------|
| `pyproject.toml` | Project configuration (Python >= 3.8 requirement) |
| `docs/conf.py` | Sphinx documentation configuration |
| `docs/requirements.txt` | Documentation build dependencies |

### 0.11.2 Attachments

No attachments were provided by the user.

### 0.11.3 Figma Screens

No Figma screens were provided for this task.

### 0.11.4 Technical Specification Sections Referenced

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project context: kitty is a GPU-accelerated terminal emulator with C/Python/Go architecture and full Unicode support |
| 4.3 Terminal Input/Output Pipeline | VT parser dispatch architecture, screen model operations, text insertion to line buffer flow |

