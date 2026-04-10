# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of deep technical questions about kitty's text shaping and layout engine, font fallback system, cell metric initialization, and GPU texture atlas startup behavior—all derived exclusively from source-code analysis with no guesswork.

- **Documentation Type**: Technical Q&A deep-dive document (placed at `blitzy/documentation/kitty_815df1e210e0.md`)
- **Category**: Create new documentation

The requirements break down into four interconnected question clusters:

- **Q1 — Unicode Shaping & Font Fallback at Startup**: How does the text shaping engine configure support for ligatures, BiDi (RTL/LTR), and combining diacritics, and what font families and fallback chains appear in verbose startup diagnostics when processing mixed Arabic (RTL) and English (LTR) text?
- **Q2 — Runtime Configuration Confirmation**: What runtime configuration values confirm these font and shaping selections before any text rendering begins?
- **Q3 — Cell Metrics, Baseline, and Decoration Alignment**: What default cell metrics, baseline positioning, and decoration alignment (overline/underline) values are reported in debug output as the screen grid and shaping subsystems initialize for complex grapheme clusters?
- **Q4 — GPU Texture Atlas Initialization**: What initial page layout, sizing, and capacity allocations are reported at launch for the glyph texture atlas, and what startup logs verify the atlas is ready to receive shaped glyph data?

### 0.1.2 Special Instructions and Constraints

- **CRITICAL**: "Do not modify any source files" — all analysis is observational, derived from reading the code.
- **Observation permission**: "enabling debug output or runtime flags for observation is fine, but restore all settings to original when complete" — the document may describe *how* to enable `--debug-font-fallback` and `--debug-rendering`, but no actual source modifications are made.
- **Implementation rule (SWE-AtlasQnA-Repo)**: Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed. Provide thinking/rationale behind answers. Do not make assumptions—base answers on the code as truth. Do not modify any existing files. Place the document in the `blitzy/documentation` directory.
- **No existing files are modified** in the source repository.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document Unicode shaping and font fallback**, we will **create** `blitzy/documentation/kitty_815df1e210e0.md` by tracing the startup call chain: `_main()` → `run_app()` → `set_font_family()` → `set_font_data()` → `font_group_for()` → `initialize_font_group()` → `calc_cell_metrics()` → `send_prerendered_sprites()`, referencing `kitty/main.py`, `kitty/fonts/render.py`, `kitty/fonts/common.py`, `kitty/fonts.c`, `kitty/freetype.c`, and `kitty/shaders.c`.
- To **document font fallback and RTL/BiDi behavior**, we will analyze `load_hb_buffer()` in `kitty/fonts.c` (which calls `hb_buffer_guess_segment_properties()` and conditionally `hb_buffer_set_direction(HB_DIRECTION_LTR)`) and `load_fallback_font()` which uses `create_fallback_face()` with `debug_font_fallback` logging.
- To **document cell metrics and decoration alignment**, we will trace `cell_metrics()` in `kitty/freetype.c` and `calc_cell_metrics()` in `kitty/fonts.c`, documenting the computed `cell_width`, `cell_height`, `baseline`, `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness`.
- To **document GPU texture atlas initialization**, we will trace `alloc_sprite_map()` in `kitty/shaders.c`, `sprite_tracker_set_layout()` in `kitty/fonts.c`, and the `SpriteMap`/`GPUSpriteTracker` structures, documenting the initial page layout computed from `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs arise:

- **HarfBuzz shaping pipeline internals**: The shaping call chain (buffer loading → segment guessing → feature application → cluster processing) must be documented to explain how Arabic reshaping and combining characters are handled.
- **`force_ltr` option interaction**: The `force_ltr` configuration option overrides HarfBuzz's auto-detected script direction, which directly affects RTL text display. This must be explained.
- **`font_features` and ligature control**: The per-font HarfBuzz feature system (`-liga`, `-dlig`, `-calt`, and user-specified features via `font_features`) must be documented as it controls ligature behavior.
- **`modify_font` adjustments**: The `modify_font` options for `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness`, `cell_width`, `cell_height`, and `baseline` directly alter the values computed during initialization and must be documented.
- **Sprite tracker capacity model**: The relationship between `max_texture_size`, `max_array_texture_layers`, `xnum`, `ynum`, and `z` layering in the `GPUSpriteTracker` must be explained to answer capacity questions.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature documentation structure with Sphinx-based docs in `docs/`, AsciiDoc/Markdown at root level, and a `blitzy/documentation/` directory as the target for generated Q&A documents.

- **Documentation framework**: Sphinx (via `docs/conf.py`)
- **Documentation build system**: `Makefile` targets for documentation generation
- **User-facing docs**: `docs/` folder containing `.rst` files covering configuration (`conf.rst`), invocation (`invocation.rst`), FAQ (`faq.rst`), keyboard protocol, graphics protocol, etc.
- **Root-level docs**: `README.asciidoc`, `CONTRIBUTING.md`, `INSTALL.md`, `SECURITY.md`, `CHANGELOG.rst`
- **Existing font documentation**: No dedicated font subsystem deep-dive exists; font configuration is documented declaratively in `kitty/options/definition.py` (the `Fonts` option group) and in `docs/conf.rst`
- **Debug documentation**: The `--debug-font-fallback` and `--debug-rendering` CLI flags are documented in `kitty/cli.py` (lines 989–1005) but their output format is not formally documented
- **Diagram tools**: Mermaid diagrams are used in the tech spec; the codebase uses no built-in diagram generation tool

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code to document:

- **Font subsystem core**: `kitty/fonts.c`, `kitty/fonts.h`, `kitty/freetype.c` — cell metrics calculation, font group initialization, HarfBuzz shaping pipeline, sprite tracker setup
- **Font discovery (Linux)**: `kitty/fontconfig.c`, `kitty/fonts/fontconfig.py` — fontconfig-based family matching, weight scoring, fallback resolution
- **Font discovery (macOS)**: `kitty/core_text.m`, `kitty/fonts/core_text.py` — CoreText-based matching
- **Font resolution pipeline**: `kitty/fonts/common.py` — `get_font_files()`, `get_font_from_spec()`, variable font handling
- **Font rendering bridge**: `kitty/fonts/render.py` — `set_font_family()`, `prerender_function()`, symbol map creation, debug dump
- **GPU texture atlas**: `kitty/shaders.c` — `alloc_sprite_map()`, `realloc_sprite_texture()`, `send_sprite_to_gpu()`
- **Glyph cache**: `kitty/glyph-cache.c`, `kitty/glyph-cache.h` — sprite position hash tables, glyph properties cache
- **Startup entry point**: `kitty/main.py` — `_main()`, `run_app()`, `AppRunner.__call__()`
- **Debug output**: `kitty/debug_config.py` — `debug_config()`, `current_fonts()`, `identify_for_debug()`
- **Configuration options**: `kitty/options/definition.py` — `font_family`, `force_ltr`, `font_features`, `disable_ligatures`, `symbol_map`, `modify_font`
- **State and type definitions**: `kitty/state.h`, `kitty/data-types.h` — `FONTS_DATA_HEAD`, `debug_font_fallback`, `debug_rendering`, `GPUSpriteTracker`, `FontGroup`, `SpriteMap`
- **Unicode data**: `kitty/unicode-data.h` — `VS15`, `VS16`, `codepoint_for_mark()`, `is_combining_char()`

### 0.2.3 Web Search Research Conducted

No web searches were necessary for this task. All answers are derived exclusively from the source code, per the user's explicit instruction: "Do not make assumptions, base your answers on the code as the truth."


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

- **Module: `kitty/fonts.c`**
  - Public APIs: `initialize_font_group()`, `calc_cell_metrics()`, `load_hb_buffer()`, `shape()`, `font_for_cell()`, `fallback_font()`, `render_group()`, `sprite_position_for()`, `sprite_tracker_set_layout()`, `send_prerendered_sprites()`, `set_font_data()`, `init_fonts()`
  - Current documentation: Undocumented internals; only user-facing option docs exist
  - Documentation needed: Complete Q&A on initialization sequence, cell metric computation, HarfBuzz buffer loading with BiDi handling, fallback font resolution with debug output format

- **Module: `kitty/freetype.c`**
  - Public APIs: `cell_metrics()`, `init_ft_face()`, `set_size_for_face()`, `face_from_descriptor()`, `identify_for_debug()`, `render_glyphs_in_cells()`
  - Current documentation: No public deep-dive documentation
  - Documentation needed: How cell width (max of ASCII 32–127 advances), cell height (ascender-to-descender + underscore correction), baseline (ascender in pixels), underline/strikethrough positions are derived from FreeType face metrics

- **Module: `kitty/shaders.c`**
  - Public APIs: `alloc_sprite_map()`, `realloc_sprite_texture()`, `send_sprite_to_gpu()`, `init_cell_program()`
  - Current documentation: No public documentation on atlas sizing logic
  - Documentation needed: How `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS` are queried, clamped (macOS: 8192/512), and used to set sprite tracker limits; how the initial `GL_TEXTURE_2D_ARRAY` is allocated with `glTexStorage3D(GL_SRGB8_ALPHA8)`

- **Module: `kitty/fonts/render.py`**
  - Public APIs: `set_font_family()`, `dump_font_debug()`, `prerender_function()`, `create_symbol_map()`
  - Current documentation: No public documentation
  - Documentation needed: How the Python-side orchestrates font file resolution, sets font data into the C layer, and triggers the pre-rendering of underline/strikethrough/cursor sprites

- **Module: `kitty/fonts/common.py`**
  - Public APIs: `get_font_files()`, `get_font_from_spec()`, `get_fine_grained_font()`
  - Current documentation: No public documentation
  - Documentation needed: The resolution chain from `FontSpec` → fontconfig/CoreText matching → variable font specialization → bold/italic variant discovery

- **Module: `kitty/fonts/fontconfig.py`**
  - Public APIs: `find_best_match()`, `font_for_family()`, `all_fonts_map()`, `find_last_resort_text_font()`
  - Current documentation: No public documentation
  - Documentation needed: How fc_list and fc_match are used to build family/ps/full/variable font maps and resolve fallback chains

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No public documentation** exists for the HarfBuzz shaping pipeline as implemented in `kitty/fonts.c` (`load_hb_buffer()`, `shape()`, `group_normal()`, `group_iosevka()`)
- **No public documentation** exists for the `--debug-font-fallback` output format and what exact font families/paths it prints
- **No public documentation** exists for the cell metric computation algorithm in `kitty/freetype.c:cell_metrics()` and its adjustment pipeline in `kitty/fonts.c:calc_cell_metrics()`
- **No public documentation** exists for the `GPUSpriteTracker` layout computation and `SpriteMap` allocation in `kitty/shaders.c`
- **No public documentation** exists for the pre-rendered sprite upload sequence (`send_prerendered_sprites()`) and its interaction with the sprite tracker
- **Undocumented configuration interactions**: The `modify_font` option's effect on baseline, underline_position, and strikethrough_position adjustments is not documented beyond the brief option help text


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive markdown Q&A document placed at `blitzy/documentation/kitty_815df1e210e0.md`. The document's internal structure is organized around the four question clusters identified during intent clarification, with rationale and source-code evidence for every answer.

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction & Scope
        ├── Q1 – Unicode Support & Font Fallback at Startup
        │   ├── HarfBuzz Buffer & Feature Configuration
        │   ├── BiDi Handling (RTL Arabic + LTR English)
        │   ├── Combining Diacritics & Variation Selectors
        │   ├── Font Family Resolution Chain
        │   ├── Fallback Font Discovery & Debug Output
        │   └── Runtime Configuration Confirmation
        ├── Q2 – Cell Metrics, Baseline & Decoration Alignment
        │   ├── cell_metrics() Algorithm
        │   ├── calc_cell_metrics() Adjustment Pipeline
        │   ├── Baseline Positioning
        │   ├── Underline & Strikethrough Derivation
        │   ├── modify_font Overrides
        │   └── Pre-rendered Sprite Upload Sequence
        ├── Q3 – GPU Texture Atlas Initialization
        │   ├── SpriteMap Allocation & GL Limits
        │   ├── sprite_tracker_set_layout() Grid Computation
        │   ├── Initial Page Layout & Capacity
        │   ├── Texture Format & Filtering
        │   └── Atlas Readiness Verification
        ├── Q4 – Observation-Only Debug Methodology
        │   ├── Enabling --debug-rendering and --debug-font-fallback
        │   ├── Expected Startup Diagnostic Output
        │   └── Restoring Original Settings
        └── Source Code References
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach**
  - Extract the font initialization call chain from `kitty/main.py` → `kitty/fonts/render.py` → `kitty/fonts/common.py` → `kitty/fonts.c` → `kitty/freetype.c` using direct source-code tracing
  - Extract cell metric formulas from `kitty/freetype.c:cell_metrics()` and adjustment logic from `kitty/fonts.c:calc_cell_metrics()`
  - Extract GPU atlas sizing from `kitty/shaders.c:alloc_sprite_map()` and sprite tracker layout from `kitty/fonts.c:sprite_tracker_set_layout()`
  - Extract BiDi/RTL handling from `kitty/fonts.c:load_hb_buffer()` — specifically `hb_buffer_guess_segment_properties()` and the `force_ltr` override
  - Extract font fallback debug output format from `kitty/fonts.c:output_cell_fallback_data()` and `debug_fonts()` macro in `kitty/state.h`
  - Extract `debug_config()` output from `kitty/debug_config.py` for startup font reporting

- **Template Application**
  - Each question cluster becomes a top-level section with sub-sections for each sub-question
  - Every technical claim is followed by its rationale and a source citation in the format `Source: kitty/<file>:<function>` or `Source: kitty/<file>:L<number>`
  - Code-path traces use inline fenced code references (function names, struct fields) to ground each answer

- **Documentation Standards**
  - Markdown with ATX-style headers (`#`, `##`, `###`)
  - Mermaid diagrams for call-chain and data-flow visualization
  - Fenced code blocks with `c` or `python` language tags for illustrative snippets
  - Tables for metric derivations, option mappings, and atlas capacity parameters
  - Every section cites the specific source file and function backing its claims

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be embedded in the deliverable document:

- **Font Initialization Call-Chain** — a sequence diagram tracing:
  `_main()` → `run_app()` → `set_options()` → `set_font_family()` → `get_font_files()` → `set_font_data()` → `initialize_font_group()` → `calc_cell_metrics()` → `sprite_tracker_set_layout()` → `send_prerendered_sprites()`

- **HarfBuzz Shaping Pipeline** — a flowchart showing:
  `load_hb_buffer()` → `hb_buffer_add_utf32()` → `hb_buffer_guess_segment_properties()` → optional `force_ltr` override → `hb_shape()` with font-specific features → cluster reordering for RTL

- **Font Fallback Resolution** — a flowchart showing:
  `font_for_cell()` routing: BLANK_FONT / BOX_FONT / symbol_map / main font / `fallback_font()` → hash lookup → `load_fallback_font()` → `create_fallback_face()` → `debug_fonts()` logging

- **GPU Atlas Layout** — a diagram showing:
  `alloc_sprite_map()` → GL limit queries → `sprite_tracker_set_limits()` → `sprite_tracker_set_layout()` → `realloc_sprite_texture()` → `send_prerendered_sprites()` → `send_sprite_to_gpu()`

- **Cell Metrics Derivation** — a data-flow diagram showing:
  FreeType face fields (`ascender`, `descender`, `height`, `underline_position`, `underline_thickness`, OS/2 `yStrikeoutPosition`, `yStrikeoutSize`) → `cell_metrics()` computation → `calc_cell_metrics()` adjustments via `modify_font` → final `baseline`, `cell_width`, `cell_height`, `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness`


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| blitzy/documentation/kitty_815df1e210e0.md | CREATE | kitty/fonts.c, kitty/freetype.c, kitty/shaders.c, kitty/fonts/render.py, kitty/fonts/common.py, kitty/fonts/fontconfig.py, kitty/main.py, kitty/debug_config.py, kitty/state.h, kitty/data-types.h, kitty/glyph-cache.c, kitty/glyph-cache.h, kitty/cli.py, kitty/options/definition.py | Comprehensive Q&A document answering all four question clusters: Unicode/font-fallback startup diagnostics, cell metrics/baseline/decoration alignment, GPU texture atlas initialization, and observation-only debug methodology. Every answer includes rationale and source citations. |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Q&A (Codebase Deep-Dive)
Source Code:
  - kitty/fonts.c (font group init, HarfBuzz shaping, fallback, sprite tracker)
  - kitty/freetype.c (cell_metrics, face init, identify_for_debug)
  - kitty/shaders.c (SpriteMap alloc, GL texture creation, sprite upload)
  - kitty/fonts/render.py (set_font_family, prerender_function, dump_font_debug)
  - kitty/fonts/common.py (get_font_files, get_font_from_spec, font resolution)
  - kitty/fonts/fontconfig.py (all_fonts_map, find_best_match, fc scoring)
  - kitty/main.py (startup chain, debug flag propagation)
  - kitty/debug_config.py (debug_config output: fonts, version, options)
  - kitty/state.h (Options struct, debug_fonts macro, global_state flags)
  - kitty/data-types.h (FONTS_DATA_HEAD, CPUCell.cc_idx, VS15/VS16)
  - kitty/glyph-cache.c / .h (GPU-side glyph cache management)
  - kitty/cli.py (--debug-rendering, --debug-font-fallback flags)
  - kitty/options/definition.py (font_family, modify_font, force_ltr defaults)
Sections:
  - Introduction & Scope
  - Q1: Unicode Support & Font Fallback at Startup
    - HarfBuzz buffer creation (MONOTONE_CHARACTERS, 2048 pre-alloc)
    - Default features (-liga, -dlig, -calt) and per-font features
    - BiDi: hb_buffer_guess_segment_properties() + force_ltr override
    - Combining marks: CPUCell.cc_idx[3], VS15/VS16 handling
    - Font resolution: get_font_files() → attr_map → fontconfig scoring
    - Fallback chain: font_for_cell() routing, fallback_font() hash+load
    - Debug output: --debug-font-fallback → output_cell_fallback_data()
    - Runtime confirmation: debug_config() → current_fonts() → identify_for_debug()
  - Q2: Cell Metrics, Baseline & Decoration Alignment
    - cell_metrics() algorithm (cell_width, cell_height, baseline derivation)
    - Underscore correction and ascender/descender calculations
    - underline_position/thickness from FreeType face fields
    - strikethrough from OS/2 table or baseline * 0.65 fallback
    - calc_cell_metrics() adjustment pipeline via modify_font
    - Pre-rendered sprite sequence: blank, underlines (1–NUM_UNDERLINE_STYLES), strikethrough, missing glyph, cursor styles
  - Q3: GPU Texture Atlas Initialization
    - alloc_sprite_map(): GL_MAX_TEXTURE_SIZE, GL_MAX_ARRAY_TEXTURE_LAYERS queries
    - macOS cap: 8192 texture size, 512 array layers
    - sprite_tracker_set_layout(): xnum, max_y, ynum=1 grid computation
    - Texture format: GL_TEXTURE_2D_ARRAY, GL_SRGB8_ALPHA8, GL_NEAREST filtering
    - Capacity: xnum * max_y * z_limit slots per font group
    - Upload: glTexSubImage3D with GL_UNSIGNED_INT_8_8_8_8
    - Reallocation: glCopyImageSubData (or readback fallback)
  - Q4: Observation-Only Debug Methodology
    - CLI flags: --debug-rendering, --debug-font-fallback
    - kitty +runonce to invoke single-shot with debug flags
    - Expected output format and parsing
    - Restoring settings: no persistent changes (flags are session-only)
  - Source Code References (file:function index)
Diagrams:
  - Font Initialization Call-Chain (Mermaid sequence diagram)
  - HarfBuzz Shaping Pipeline (Mermaid flowchart)
  - Font Fallback Resolution (Mermaid flowchart)
  - GPU Atlas Layout (Mermaid flowchart)
  - Cell Metrics Derivation (Mermaid data-flow diagram)
Key Citations:
  kitty/fonts.c, kitty/freetype.c, kitty/shaders.c, kitty/fonts/render.py,
  kitty/fonts/common.py, kitty/fonts/fontconfig.py, kitty/main.py,
  kitty/debug_config.py, kitty/state.h, kitty/data-types.h, kitty/cli.py,
  kitty/options/definition.py, kitty/glyph-cache.c, kitty/glyph-cache.h
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration files need to be modified. The repository uses Sphinx-based documentation under `docs/` with a `conf.py`, but the deliverable is a standalone markdown file in `blitzy/documentation/` outside the existing documentation tree. No `mkdocs.yml`, `docusaurus.config.js`, or `.readthedocs.yml` changes are required.

### 0.5.4 Cross-Documentation Dependencies

- **No shared includes** are consumed or produced; the deliverable is self-contained
- **Navigation links**: Not applicable — the document lives in `blitzy/documentation/`, not in the project's Sphinx tree
- **Source-code cross-references**: The document contains citations to 14 source files; these are read-only references and require no changes to those files
- **No table-of-contents or index updates** are required in the existing `docs/` hierarchy


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation-generation tools are required for this task. The deliverable is a hand-authored Markdown file (`blitzy/documentation/kitty_815df1e210e0.md`) that uses only standard GitHub-Flavored Markdown and Mermaid diagram syntax (rendered natively by GitHub, GitLab, and most Markdown viewers).

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| N/A | Markdown (GFM) | CommonMark 0.31 | Document authoring format |
| N/A | Mermaid | 11.x (renderer-provided) | Embedded diagrams rendered by hosting platform |

No `pip`, `npm`, or other package installations are needed to produce or consume this document. The Mermaid diagrams are authored inline using fenced code blocks (`mermaid`) and rely on the rendering platform's built-in Mermaid support.

### 0.6.2 Source-Code Dependencies (Read-Only)

The following source files are read as reference material. They are not modified and carry no runtime dependency for the documentation itself, but are catalogued here because the accuracy of the document depends on their current content.

| Source File | Role in Document |
|-------------|-----------------|
| kitty/fonts.c | Font group initialization, HarfBuzz shaping pipeline, fallback font mechanism, sprite tracker layout, pre-rendered sprite upload |
| kitty/freetype.c | cell_metrics() algorithm, FreeType face initialization, identify_for_debug() output format |
| kitty/shaders.c | SpriteMap allocation, GL texture creation, sprite upload, GL limit queries |
| kitty/fonts/render.py | set_font_family() orchestration, prerender_function() sprite generation, dump_font_debug() |
| kitty/fonts/common.py | get_font_files() resolution, get_font_from_spec(), variable font specialization |
| kitty/fonts/fontconfig.py | fontconfig-based font enumeration, FCScorer, find_best_match() |
| kitty/main.py | Startup entry point, debug flag propagation, run_app() call chain |
| kitty/debug_config.py | debug_config() output format (fonts, options, version) |
| kitty/state.h | Options struct (force_ltr, modify_font fields), debug_fonts() macro, global_state |
| kitty/data-types.h | FONTS_DATA_HEAD macro, CPUCell combining marks (cc_idx), VS15/VS16 constants |
| kitty/glyph-cache.c | GPU-side glyph cache management |
| kitty/glyph-cache.h | Glyph cache API declarations |
| kitty/cli.py | --debug-rendering and --debug-font-fallback CLI flag definitions |
| kitty/options/definition.py | Default values for font_family, font_size, force_ltr, disable_ligatures, modify_font, symbol_map |
| kitty/fonts.h | Font subsystem API contract, cell_metrics() signature, sprite tracker API |

### 0.6.3 Documentation Reference Updates

No link updates are required. The deliverable is a new standalone file with no inbound links from the existing documentation tree. All internal cross-references within the document use relative anchor links (`#section-heading`) which are self-contained.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **Current coverage analysis** (prior to this task):
  - Text shaping / HarfBuzz internals documented: 0/6 key functions (0%)
  - Cell metric derivation algorithm documented: 0/2 key functions (0%)
  - GPU atlas initialization documented: 0/3 key functions (0%)
  - Font fallback debug output format documented: 0/2 key functions (0%)
  - BiDi / RTL handling documented: 0/1 key code path (0%)
  - Combining diacritics handling documented: 0/1 data structure (0%)

- **Target coverage after this task**: 100% of the user's four question clusters, mapped to the following specific functions and structures:

| Question Cluster | Functions / Structures Covered | Target |
|-----------------|-------------------------------|--------|
| Q1 – Unicode & Font Fallback | `init_fonts()`, `load_hb_buffer()`, `shape()`, `font_for_cell()`, `fallback_font()`, `get_font_files()`, `find_best_match()`, `debug_config()`, `dump_font_debug()` | 9/9 (100%) |
| Q2 – Cell Metrics & Decorations | `cell_metrics()`, `calc_cell_metrics()`, `send_prerendered_sprites()`, `prerender_function()` | 4/4 (100%) |
| Q3 – GPU Texture Atlas | `alloc_sprite_map()`, `sprite_tracker_set_layout()`, `realloc_sprite_texture()`, `send_sprite_to_gpu()` | 4/4 (100%) |
| Q4 – Debug Methodology | `--debug-rendering`, `--debug-font-fallback`, `debug_fonts()` macro, `output_cell_fallback_data()` | 4/4 (100%) |

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements**:
  - Every answer traces the full code path from Python entry point through C implementation
  - Every numeric value (cell_width, baseline, underline_position, atlas xnum/max_y) is derived with the exact formula from source
  - Every configuration option (force_ltr, disable_ligatures, modify_font, font_features) is documented with its default value and effect
  - Every debug flag (--debug-rendering, --debug-font-fallback) is documented with its exact CLI syntax and the output it produces

- **Accuracy validation**:
  - All function signatures and struct field names are verified against source files at the current commit
  - All default values are verified against `kitty/options/definition.py` (e.g., `font_family='monospace'`, `font_size=11.0`, `force_ltr=False`, `disable_ligatures='never'`)
  - All metric formulas are verified against `kitty/freetype.c:cell_metrics()` and `kitty/fonts.c:calc_cell_metrics()`
  - All GL constants are verified against `kitty/shaders.c` (GL_SRGB8_ALPHA8, GL_NEAREST, GL_CLAMP_TO_EDGE, GL_UNSIGNED_INT_8_8_8_8)

- **Clarity standards**:
  - Each answer opens with a direct, unambiguous statement before presenting supporting evidence
  - Technical rationale follows each claim, explaining *why* the code works this way
  - Progressive disclosure: summary → detailed formula → source citation
  - Consistent terminology: "cell metrics" (not "character metrics"), "sprite tracker" (not "glyph tracker"), "font group" (not "font set")

- **Maintainability**:
  - Every section includes `Source: kitty/<file>:<function_or_line>` citations for traceability
  - The document header notes the commit/branch (`kitty_815df1e210e0`) for future staleness detection
  - Sections are self-contained and can be updated independently as the codebase evolves

### 0.7.3 Example and Diagram Requirements

| Diagram | Type | Purpose |
|---------|------|---------|
| Font Initialization Call-Chain | Mermaid sequence diagram | Traces startup from `_main()` through `send_prerendered_sprites()` |
| HarfBuzz Shaping Pipeline | Mermaid flowchart | Shows buffer loading, property guessing, feature application, cluster handling |
| Font Fallback Resolution | Mermaid flowchart | Illustrates `font_for_cell()` routing and `fallback_font()` cache-or-load logic |
| GPU Atlas Layout | Mermaid flowchart | Shows GL limit query → tracker layout → texture allocation → sprite upload |
| Cell Metrics Derivation | Mermaid data-flow diagram | Maps FreeType fields through `cell_metrics()` → `calc_cell_metrics()` → final values |

- Minimum diagrams: 5 (one per major topic)
- All diagrams are Mermaid-based (no external image dependencies)
- Code examples limited to 2–3 line illustrative snippets referencing actual source constructs


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/kitty_815df1e210e0.md` — the sole deliverable

- **Source files analyzed for documentation content** (read-only):
  - `kitty/fonts.c` — font group init, HarfBuzz shaping, fallback, sprite tracker, ligature strategy detection
  - `kitty/freetype.c` — FreeType face init, cell_metrics(), identify_for_debug()
  - `kitty/shaders.c` — SpriteMap alloc, GL texture creation/upload, GL limit queries
  - `kitty/fonts/render.py` — set_font_family(), prerender_function(), dump_font_debug()
  - `kitty/fonts/common.py` — get_font_files(), get_font_from_spec(), variable font specialization
  - `kitty/fonts/fontconfig.py` — fontconfig enumeration, scoring, find_best_match()
  - `kitty/main.py` — startup chain, debug flag propagation
  - `kitty/debug_config.py` — debug_config() output format
  - `kitty/state.h` — Options struct, debug_fonts() macro, global_state debug flags
  - `kitty/data-types.h` — FONTS_DATA_HEAD, CPUCell.cc_idx, VS15/VS16 constants
  - `kitty/glyph-cache.c` — GPU glyph cache management
  - `kitty/glyph-cache.h` — glyph cache API
  - `kitty/cli.py` — --debug-rendering, --debug-font-fallback definitions
  - `kitty/options/definition.py` — font_family, font_size, force_ltr, disable_ligatures, modify_font, symbol_map, font_features defaults
  - `kitty/fonts.h` — font subsystem API contract

- **Documentation topics covered**:
  - HarfBuzz buffer configuration and shaping features for complex Unicode
  - BiDi (RTL Arabic + LTR English) handling via `hb_buffer_guess_segment_properties()` and `force_ltr`
  - Combining diacritics via `CPUCell.cc_idx[3]` and variation selectors VS15/VS16
  - Ligature support and strategy detection (SPACERS_BEFORE, SPACERS_AFTER, SPACERS_IOSEVKA)
  - Font family resolution chain from config options through fontconfig scoring to face instantiation
  - Fallback font discovery, caching, and `--debug-font-fallback` diagnostic output
  - Cell metric computation: cell_width, cell_height, baseline, underline_position/thickness, strikethrough_position/thickness
  - `modify_font` adjustment pipeline for metric overrides
  - Pre-rendered sprite upload sequence (blank, underlines, strikethrough, missing glyph, cursors)
  - GPU texture atlas: SpriteMap allocation, GL limit queries, sprite_tracker_set_layout() grid computation, texture format and filtering
  - Observation-only debug flags (--debug-rendering, --debug-font-fallback) and their restoration

- **Observation-only runtime actions** (permitted per user instructions):
  - Passing `--debug-rendering` and `--debug-font-fallback` CLI flags to kitty
  - Reading resulting stdout/stderr diagnostic output
  - Restoring all settings to original state upon completion (these flags are session-only and leave no persistent changes)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No `.c`, `.py`, `.h`, `.glsl`, or any other source file is created, modified, or deleted
- **Persistent configuration changes**: No `kitty.conf`, environment variable, or build flag is permanently altered
- **Test file modifications**: No test files are created or edited
- **Feature additions or code refactoring**: No functional changes to kitty
- **Existing documentation tree changes**: No files under `docs/` are modified; the deliverable lives in `blitzy/documentation/`
- **Build system or deployment changes**: No `setup.py`, `Makefile`, CI config, or packaging changes
- **Font file installation**: No new fonts are installed on the system
- **Non-font subsystems**: Keyboard input handling, network protocols, kitten plugins, shell integration, window management, and other kitty subsystems outside the font/shaping/atlas pipeline are not documented
- **Performance benchmarking**: No profiling or timing measurements of the shaping/atlas pipeline
- **Cross-platform comparisons**: macOS CoreText backend (`kitty/fonts/core_text.py`) is mentioned only where it shares interfaces with the fontconfig path; no dedicated CoreText deep-dive is in scope


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the deliverable is a standalone Markdown file that requires no build step. Any Markdown renderer (GitHub, VS Code, `grip`, `pandoc`) can display it.

- **Documentation preview command**:
  ```
  python3 -m grip blitzy/documentation/kitty_815df1e210e0.md
  ```
  Alternatively, open the file in any Markdown-capable editor or viewer.

- **Diagram generation command**: No offline generation required. Mermaid diagrams are authored inline and rendered by the hosting platform. For local rendering:
  ```
  npx @mermaid-js/mermaid-cli mmdc -i input.mmd -o output.svg
  ```

- **Documentation deployment command**: Not applicable — the file is committed directly to the repository at `blitzy/documentation/kitty_815df1e210e0.md`.

- **Default format**: GitHub-Flavored Markdown (GFM) with embedded Mermaid diagram blocks.

- **Citation requirement**: Every technical claim must include a source citation in the format `Source: kitty/<file>:<function_or_line>`.

- **Style guide**: The document follows the implementation rule `SWE-AtlasQnA-Repo`:
  - Provide thinking and rationale behind every answer
  - Base all answers on the code as the single source of truth — no assumptions
  - Do not modify any existing files in the source repository
  - Place the generated document in `blitzy/documentation/`

- **Documentation validation**: Manual review to confirm:
  - All four question clusters are fully answered
  - Every source citation references an existing file and function
  - All Mermaid diagrams render without syntax errors
  - No source files were modified (verify with `git status`)

### 0.9.2 Debug Observation Protocol

The user explicitly permits enabling debug output for observation purposes, provided all settings are restored. The following observation-only actions are sanctioned:

- **Enable diagnostic flags** by launching kitty with CLI arguments:
  ```
  kitty --debug-rendering --debug-font-fallback
  ```
  These flags set `global_state.debug_rendering = true` and `global_state.debug_font_fallback = true` in `kitty/state.h`. They are session-scoped and leave no persistent changes.

- **Capture startup output** from:
  - `debug_config()` (in `kitty/debug_config.py`): prints kitty version, OS, OpenGL info, current font faces via `identify_for_debug()`, loaded config paths, and all non-default options
  - `debug_fonts()` macro (in `kitty/state.h`): emits `timed_debug_print()` messages during font fallback resolution, showing codepoints, style flags, and chosen face
  - `dump_font_debug()` (in `kitty/fonts/render.py`): logs current fonts when `--debug-font-fallback` is active

- **Restore settings**: No restoration action is needed — all debug flags are CLI arguments that do not persist to `kitty.conf` or any configuration file. Closing the kitty session returns to default behavior.


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the `SWE-AtlasQnA-Repo` implementation rule:

- **Do not modify any existing files in the source repository.** The deliverable is a new file at `blitzy/documentation/kitty_815df1e210e0.md` — no source, configuration, test, or documentation file in the kitty repository is created, altered, or deleted.

- **Base all answers on the code as the single source of truth.** Every technical claim must be traceable to a specific function, struct, macro, or constant in the kitty source. No assumptions, no inferred behavior that cannot be confirmed in code.

- **Provide thinking and rationale behind every answer.** Each answer must explain *why* the code behaves the way it does — not just *what* it does. Rationale sections follow every technical claim.

- **Place the generated document in `blitzy/documentation/`.** The exact file path is `blitzy/documentation/kitty_815df1e210e0.md`, matching the branch name `kitty_815df1e210e0`.

- **Enabling debug output or runtime flags for observation is permitted, but restore all settings to original when complete.** The debug flags `--debug-rendering` and `--debug-font-fallback` are session-only CLI arguments. No persistent configuration changes are made. Closing the kitty process fully restores default behavior.

- **Include source citations for all technical details.** Use the format `Source: kitty/<file>:<function_or_line>` for every claim backed by source-code evidence.

- **Embed Mermaid diagrams for all major workflows.** The font initialization call-chain, HarfBuzz shaping pipeline, font fallback resolution, GPU atlas layout, and cell metrics derivation each require a dedicated diagram.

- **Use consistent terminology from the kitty codebase.** Prefer source-code names: "font group" (not "font set"), "sprite tracker" (not "glyph tracker"), "cell metrics" (not "character metrics"), "fallback font map" (not "fallback cache"), "sprite map" (not "texture atlas" when referring to the `SpriteMap` struct).

- **Keep code snippets to 2–3 lines maximum.** Snippets are illustrative references to actual constructs, not full reproductions of source functions.

- **No cross-platform speculation.** Document the fontconfig (Linux) path as the primary reference. Mention the CoreText (macOS) path only where shared interfaces are relevant. Do not speculate about behavior differences without code evidence.


## 0.11 References

### 0.11.1 Source Files Searched and Analyzed

The following files and folders were systematically searched and retrieved to derive the conclusions in this Agent Action Plan:

| File Path | Content Examined |
|-----------|-----------------|
| `kitty/fonts.c` | FontGroup struct, GPUSpriteTracker, init_fonts(), set_font_data(), initialize_font_group(), calc_cell_metrics(), sprite_tracker_set_layout(), send_prerendered_sprites(), load_hb_buffer(), shape(), font_for_cell(), fallback_font(), output_cell_fallback_data(), detect_spacer_strategy(), group_normal(), group_iosevka(), HarfBuzz feature constants (LIGA, DLIG, CALT) |
| `kitty/fonts.h` | Font API contract: cell_metrics() signature (7 output params), render_glyphs_in_cells(), create_fallback_face(), sprite_tracker_set_limits(), sprite_tracker_current_layout(), StringCanvas struct |
| `kitty/freetype.c` | Face struct (ascender, descender, height, underline_position/thickness, strikethrough_position/thickness, hinting, hintstyle, is_variable, has_color, harfbuzz_font), cell_metrics() algorithm, init_ft_face(), identify_for_debug() output format, load flags logic |
| `kitty/shaders.c` | SpriteMap struct (cell_width, cell_height, xnum, ynum, x, y, z, max_texture_size, max_array_texture_layers, texture_id), alloc_sprite_map() GL queries, realloc_sprite_texture() GL_TEXTURE_2D_ARRAY creation, send_sprite_to_gpu() upload logic |
| `kitty/fonts/render.py` | set_font_family() orchestration, prerender_function() sprite generation (underlines, strikethrough, missing glyph, cursors), dump_font_debug(), current_fonts(), coalesce_symbol_maps() |
| `kitty/fonts/common.py` | get_font_files(), get_font_from_spec(), get_fine_grained_font(), all_fonts_map(), attr_map (bold/italic → config option mapping), variable font specialization (find_medium_variant, find_bold_italic_variant) |
| `kitty/fonts/fontconfig.py` | all_fonts_map() with fc_list(FC_DUAL) + fc_list(FC_MONO), FCScorer.score() (variable_score, bold_score, italic_score, monospace_match, width_score), find_best_match() resolution chain (ps_map → full_map → family_map → fc_match fallback) |
| `kitty/main.py` | _main() → parse_args() → create_opts() → setup_environment() → run_app() → set_options(debug_rendering, debug_font_fallback) → set_font_family() → create_os_window(load_all_shaders) → Boss.start() → dump_font_debug() |
| `kitty/debug_config.py` | debug_config() function: version, OS, OpenGL version, current_fonts() with identify_for_debug(), loaded config paths, non-default options (symbol_map, modify_font), environment variables |
| `kitty/state.h` | Options struct fields (force_ltr, disable_ligatures, debug_keyboard, underline_position/thickness, strikethrough_position/thickness, cell_width, cell_height, baseline with val+AdjustmentUnit), GlobalState (debug_rendering, debug_font_fallback), debug_fonts() macro → timed_debug_print() |
| `kitty/data-types.h` | FONTS_DATA_HEAD macro (sprite_map, logical_dpi_x/y, font_sz_in_pts, cell_width, cell_height), CPUCell.cc_idx[3] (combining_type = uint16_t), VS15 (1364), VS16 (1365), codepoint_for_mark(), has_cell_text() |
| `kitty/glyph-cache.c` | GPU-side glyph cache management (cache initialization, eviction strategies) |
| `kitty/glyph-cache.h` | Glyph cache API declarations |
| `kitty/cli.py` | --debug-rendering (bool-set), --debug-font-fallback (bool-set), --debug-gl flag definitions |
| `kitty/options/definition.py` | font_family (default 'monospace'), bold/italic/bi_font (default 'auto'), font_size (default 11.0), force_ltr (default 'no'), symbol_map, narrow_symbols, disable_ligatures (never/cursor/always), font_features, modify_font |
| `kitty/freetype_render_ui_text.c` | FreeType/HarfBuzz UI text rendering (discovered via search) |

| Folder Path | Content Examined |
|-------------|-----------------|
| `kitty/` | Complete directory listing: all C, Python, GLSL, and header files; subfolders conf/, fonts/, launcher/, layout/, options/, rc/ |
| `kitty/fonts/` | All 7 files: __init__.py, box_drawing.py, common.py, core_text.py, fontconfig.py, list.py, render.py |
| Repository root (`""`) | Full structure: gen/, docs/, kittens/, kitty/, tools/, shell-integration/, .github/, 3rdparty/, glad/, glfw/, bypy/, kitty_tests/, logo/ |

### 0.11.2 Tech Spec Sections Retrieved

| Section | Purpose |
|---------|---------|
| 5.2 Component Details | Background on kitty's architectural components and module responsibilities |
| 7.2 GPU Rendering Pipeline | Background on the GPU rendering architecture, texture management, and sprite upload flow |

### 0.11.3 Attachments

No attachments were provided by the user. No Figma URLs or external design assets are referenced.

### 0.11.4 External References

No web searches were conducted. All conclusions are derived exclusively from source-code analysis of the kitty repository at branch `kitty_815df1e210e0`.


