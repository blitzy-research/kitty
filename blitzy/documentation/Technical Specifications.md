# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to produce a comprehensive investigative markdown document (`kitty_815df1e210e0.md`) that answers detailed technical questions about the kitty terminal emulator's internal subsystems at startup time. Specifically, the investigation covers:

- **Text Shaping and Layout Engine Initialization** — How the kitty codebase configures support for complex Unicode (ligatures, bidirectional text, combining diacritics) and font fallback during initial startup
- **Font Family and Fallback Chain Diagnostics** — With verbose logging (`--debug-font-fallback`) enabled, what exact font families and fallback chains appear in startup diagnostics when handling mixed Arabic (RTL) and English (LTR) text, and what runtime configuration values confirm these selections before any text rendering begins
- **Cell Metrics and Decoration Alignment** — As the screen grid and HarfBuzz shaping subsystems initialize, what default cell metrics (cell_width, cell_height), baseline positioning, and decoration alignment values (underline position/thickness, strikethrough position/thickness) appear in debug output for complex grapheme clusters
- **GPU Texture Atlas Initialization** — For GPU texture atlas (sprite map) initialization at launch, what initial page layout, sizing, and capacity allocations are reported for shaped glyphs across primary and fallback fonts, and what startup logs verify the atlas is ready to receive shaped glyph data
- **Non-Destructive Investigation** — No source files may be modified; enabling debug output or runtime flags for observation is permitted, but all settings must be restored to their original state when complete

The implicit requirements surfaced from the codebase analysis include:

- The investigation must build the kitty project from source to produce the native C extensions required for the `fast_data_types` module, as the font/glyph/shaping subsystems are implemented in C and exposed to Python via CPython extension types
- The `--debug-font-fallback` CLI flag sets `global_state.debug_font_fallback`, which enables the `debug_fonts(...)` macro in `kitty/fonts.c` (aliased to `debug` via `#define debug debug_fonts`), producing fallback selection output on stderr
- HarfBuzz's `hb_buffer_guess_segment_properties()` is the mechanism that auto-detects script and direction for bidirectional text; the `force_ltr` configuration option overrides this to `HB_DIRECTION_LTR`
- The texture atlas is a `GL_TEXTURE_2D_ARRAY` managed by `kitty/shaders.c`, whose dimensions are derived from `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS` OpenGL queries, capped at 8192 and 512 respectively on Apple platforms

### 0.1.2 Special Instructions and Constraints

- **Read-Only Investigation**: The user explicitly states "Do not modify any source files." Per the project rule `SWE-AtlasQnA-Repo`, no existing files in the source repository may be modified, and no code may be added except the requested markdown document in `blitzy/documentation/`
- **Debug Flags Are Permitted**: Enabling debug output or runtime flags for observation is acceptable, but all settings must be restored to their original state when the investigation completes
- **Answer Grounded in Code**: Per the project rule, answers must be based on the code as truth — no assumptions. Thinking and rationale behind each answer must be provided
- **Output Document**: A new markdown document named `kitty_815df1e210e0.md` must be created in the `blitzy/documentation` directory, comprehensively answering all posed questions
- **Docker Environment**: The investigation runs inside the Docker container `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the text shaping / complex Unicode questions**, we will analyze the HarfBuzz integration in `kitty/fonts.c` (functions `load_hb_buffer`, `shape`, `shape_run`, `init_fonts`), the combining character handling in `kitty/unicode-data.h` (`is_combining_char`, `codepoint_for_mark`, `mark_for_codepoint`, `VS15`/`VS16`), and the `force_ltr` option that overrides `hb_buffer_guess_segment_properties()` auto-detection of bidirectional scripts
- To **answer the font family and fallback chain questions**, we will trace the startup path from `kitty/main.py` (`set_font_family` → `get_font_files` in `kitty/fonts/common.py` → platform backend `kitty/fonts/fontconfig.py` or `kitty/fonts/core_text.py`), the `create_fallback_face` function in `kitty/fontconfig.c`, the `dump_font_debug()` function in `kitty/fonts/render.py`, and the `debug_config()` reporting in `kitty/debug_config.py`
- To **answer the cell metrics and decoration alignment questions**, we will analyze the `calc_cell_metrics` function in `kitty/fonts.c` which calls `cell_metrics` in `kitty/freetype.c`, and the `prerender_function` in `kitty/fonts/render.py` which receives baseline, underline_position, underline_thickness, strikethrough_position, and strikethrough_thickness
- To **answer the GPU texture atlas questions**, we will analyze `alloc_sprite_map` in `kitty/shaders.c`, `sprite_tracker_set_layout` in `kitty/fonts.c`, the `SpriteMap` struct and `realloc_sprite_texture` in `kitty/shaders.c`, and `send_prerendered_sprites_for_window` which triggers initial atlas allocation
- To **produce the deliverable**, we will create `blitzy/documentation/kitty_815df1e210e0.md` containing the comprehensive answers, build from source as needed to verify behaviors, and ensure no source files are modified

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The following files and directories form the complete scope of the investigation, categorized by subsystem:

**Font Discovery and Resolution Pipeline**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty/fonts/common.py` | Platform-neutral font resolution workflow; turns `FontSpec` inputs into medium/bold/italic/bi face descriptors | Entry point for font family resolution via `get_font_files()` |
| `kitty/fonts/fontconfig.py` | Linux fontconfig backend; builds font maps, scores candidates, finds fallback fonts via `fc_match` | Primary backend for font discovery on Linux; `find_best_match()`, `find_last_resort_text_font()` |
| `kitty/fonts/core_text.py` | macOS CoreText backend; equivalent discovery/matching role | macOS-only alternative to fontconfig |
| `kitty/fonts/render.py` | Bridge from resolved fonts to runtime rendering state; `set_font_family()`, `dump_font_debug()`, `create_symbol_map()` | Font initialization, symbol map creation, and debug dump output |
| `kitty/fonts/__init__.py` | Shared typing and metadata (FontSpec, Descriptor, VariableData, family_name_to_key) | Data model contracts for all font operations |
| `kitty/fonts/list.py` | Font enumeration and JSON export; `choose-fonts` kitten launcher | Font listing used during discovery |
| `kitty/fontconfig.c` | Native fontconfig integration; `fc_list`, `fc_match`, `create_fallback_face` | C-level font matching and fallback face creation |
| `kitty/core_text.m` | Native CoreText integration (macOS) | macOS font discovery and descriptor construction |
| `kitty/font-names.c` | Font name table reading (SFNT NAME, fvar, STAT tables) | Variable font metadata extraction |

**Text Shaping and Complex Unicode**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty/fonts.c` | Core font engine: HarfBuzz shaping, ligature grouping, fallback dispatch, cell metrics, sprite tracking | `load_hb_buffer()`, `shape()`, `shape_run()`, `calc_cell_metrics()`, `fallback_font()`, `render_line()` |
| `kitty/fonts.h` | Font API header: declares `cell_metrics`, `render_glyphs_in_cells`, `create_fallback_face`, `harfbuzz_font_for_face` | API contract between font backends and shaping engine |
| `kitty/freetype.c` | FreeType face management; glyph rasterization; `cell_metrics()` function computing baseline, underline, strikethrough | Cell metric calculation from FreeType face ascender/descender/underline data |
| `kitty/unicode-data.c` / `kitty/unicode-data.h` | Unicode property tables; `is_combining_char()`, `codepoint_for_mark()`, `mark_for_codepoint()`, `VS15`/`VS16` | Combining character and variation selector handling |
| `kitty/emoji.h` | Emoji detection: `is_emoji()` function | Emoji presentation determination affecting font fallback |

**GPU Texture Atlas and Glyph Caching**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty/shaders.c` | OpenGL shader management; `SpriteMap` struct; `alloc_sprite_map()`, `realloc_sprite_texture()`, `send_sprite_to_gpu()`, `ensure_sprite_map()` | GPU texture atlas allocation, sizing from `GL_MAX_TEXTURE_SIZE`/`GL_MAX_ARRAY_TEXTURE_LAYERS` |
| `kitty/glyph-cache.c` | Sprite position hash table; `find_or_create_sprite_position()` | Glyph-to-sprite-coordinate mapping cache |
| `kitty/glyph-cache.h` | `SpritePosition` and `GlyphProperties` struct definitions | Data types for glyph cache entries |
| `kitty/gl.c` / `kitty/gl.h` | OpenGL wrapper; shader compilation, VAO/buffer helpers, debug/error checking | GL infrastructure underlying the texture atlas |

**Application Startup and Configuration**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty/main.py` | Main startup orchestration; `_run_app()`, `run_app()` (AppRunner), `init_glfw()`, `load_all_shaders()` | Startup sequence: `set_options()` → `set_font_family()` → `create_os_window()` → `dump_font_debug()` |
| `kitty/debug_config.py` | Diagnostic report generation; `debug_config()` outputs fonts, paths, OpenGL info, config diffs | Prints `current_fonts()` with `identify_for_debug()` for each face |
| `kitty/cli.py` | CLI definition; `--debug-font-fallback`, `--debug-rendering`, `--debug-keyboard` flags | Debug flag parsing |
| `kitty/state.h` | Global state struct; `debug_font_fallback`, `debug_rendering`, `force_ltr`, font metric adjustment fields | Runtime state holding debug flags and option values |
| `kitty/state.c` | `set_options()` function: sets `global_state.debug_font_fallback` from CLI args | Applies debug_font_fallback flag to global state |
| `kitty/data-types.h` | Core type definitions; `FONTS_DATA_HEAD`, `FONTS_DATA_HANDLE`, `CPUCell` with `cc_idx[3]` for combining chars | Fundamental data structures for cells and font groups |
| `kitty/options/definition.py` | Option definitions including `force_ltr`, `font_family`, `font_size`, `symbol_map` | Configuration option schema |

**Rendering and Shader Files**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty/cell_vertex.glsl` / `kitty/cell_fragment.glsl` | Cell rendering shaders consuming sprite atlas textures | Final consumers of glyph sprite data |
| `kitty/shaders.py` | Python-side shader loading and macro substitution | Shader source preparation |
| `kitty/fonts/box_drawing.py` | Box-drawing and special glyph rasterization | Pre-rendered sprites sent during `send_prerendered_sprites()` |

**Test Files**

| File Path | Purpose | Relevance |
|---|---|---|
| `kitty_tests/fonts.py` | Font selection, rendering, and shaping tests | Validates font resolution behavior, provides reference patterns |

### 0.2.2 Web Search Research Conducted

No external web search was required for this investigation. All answers are derived directly from the kitty source code as ground truth, per the project rules. The codebase contains sufficient documentation in option definitions, code comments, and function signatures to answer all posed questions.

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — Comprehensive markdown document answering all questions about text shaping, font fallback, cell metrics, and GPU texture atlas initialization in the kitty terminal emulator
- No other new files are required. No source files will be modified.

## 0.3 Dependency Inventory

### 0.3.1 Key Packages

The following packages are relevant to the investigation and must be present in the build/runtime environment:

| Registry | Package | Version Constraint | Purpose |
|---|---|---|---|
| System | Python | ≥ 3.8 (per `pyproject.toml` `requires-python`) | Runtime interpreter for kitty's Python layer |
| System | Go | ≥ 1.22 (per `go.mod`) | Go-based tooling and development launcher |
| System (pkg-config) | HarfBuzz | ≥ 1.5 (per `setup.py` `at_least_version('harfbuzz', 1, 5)`) | OpenType text shaping: ligatures, complex scripts, bidi |
| System (pkg-config) | FreeType | System version | Font rasterization and cell metric extraction |
| System (pkg-config) | fontconfig | System version | Linux font discovery and matching |
| System (shared lib) | libGL / OpenGL | ≥ 3.3 (GLSL version requirements) | GPU rendering, texture atlas via `GL_TEXTURE_2D_ARRAY` |
| System | GLFW | Vendored fork (3.4-based, in `glfw/` directory) | Platform windowing backend (X11/Wayland/Cocoa) |
| PyPI | mypy | Development only (per `pyproject.toml`) | Static type checking |
| PyPI | ruff | Development only (per `pyproject.toml`) | Linting and formatting |

### 0.3.2 Dependency Updates

No dependency updates are required for this task. The investigation is read-only and produces only a documentation artifact. All existing dependencies remain at their current versions as specified in the repository's dependency manifests (`pyproject.toml`, `go.mod`, `go.sum`, `setup.py`).

### 0.3.3 Import and Reference Notes

The investigation document will reference the following internal import chains to explain behavior:

- `kitty/main.py` imports `set_font_family` from `kitty/fonts/render.py` and `set_options` from `kitty/fast_data_types`
- `kitty/fonts/render.py` imports `get_font_files` from `kitty/fonts/common.py`
- `kitty/fonts/common.py` conditionally imports from `kitty/fonts/fontconfig.py` (Linux) or `kitty/fonts/core_text.py` (macOS)
- `kitty/fonts.c` includes `fonts.h`, `state.h`, `emoji.h`, `unicode-data.h`, `glyph-cache.h` and links against HarfBuzz (`hb.h`)
- `kitty/shaders.c` includes `fonts.h`, `gl.h`, and manages the `SpriteMap` and `GL_TEXTURE_2D_ARRAY` atlas

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation traces the following integration points through the startup and rendering initialization path. These are the code touchpoints that provide the answers to the user's questions:

**Font Resolution Chain (answers: font families and fallback chains)**

- `kitty/main.py` → `run_app.__call__()` invokes `set_font_family(opts)` at startup
- `kitty/fonts/render.py` → `set_font_family()` calls `get_font_files(opts)` to resolve medium/bold/italic/bi faces, then calls `create_symbol_map(opts)` and passes everything to native `set_font_data()`
- `kitty/fonts/common.py` → `get_font_files()` dispatches to `get_font_from_spec()` which queries the platform font backend
- `kitty/fonts/fontconfig.py` → `find_best_match()` uses `all_fonts_map()` (backed by `fc_list(spacing=FC_DUAL) + fc_list(spacing=FC_MONO)`) and `fc_match()` for alias resolution
- `kitty/fontconfig.c` → `create_fallback_face()` uses `FcPatternCreate` with FC_FAMILY set to "monospace" (or "emoji" for emoji presentation), adds the cell's charset, then calls `_fc_match()` to find a fallback font at render time

**Text Shaping Pipeline (answers: complex Unicode, bidi, ligatures)**

- `kitty/fonts.c` → `render_line()` iterates cells, calling `font_for_cell()` to select font (main, symbol map, or fallback)
- `kitty/fonts.c` → `shape_run()` calls `load_hb_buffer()` which populates the HarfBuzz buffer with codepoints including combining characters from `cc_idx[]`, then calls `hb_buffer_guess_segment_properties()` (auto-detects script/direction), optionally overridden by `force_ltr` to `HB_DIRECTION_LTR`
- `kitty/fonts.c` → `hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features)` performs the actual shaping with font-specific features (liga, dlig, calt)
- `kitty/fonts.c` → `init_fonts()` initializes the global HarfBuzz buffer with `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS` and creates `-liga`, `-dlig`, `-calt` feature toggles

**Cell Metrics Computation (answers: cell metrics, baseline, decorations)**

- `kitty/fonts.c` → `initialize_font_group()` calls `calc_cell_metrics(fg)`
- `kitty/fonts.c` → `calc_cell_metrics()` calls `cell_metrics()` from `kitty/freetype.c` to get raw values, then applies `modify_font` adjustments from options (`OPT(cell_width)`, `OPT(cell_height)`, `OPT(baseline)`, `OPT(underline_position)`, etc.)
- `kitty/freetype.c` → `cell_metrics()` computes: `baseline = font_units_to_pixels_y(ascender)`, `underline_position = font_units_to_pixels_y(ascender - underline_position)`, `underline_thickness = MAX(1, font_units_to_pixels_y(underline_thickness))`, strikethrough from OS/2 table or `baseline * 0.65` fallback
- `kitty/fonts.c` → After `calc_cell_metrics()`, `sprite_tracker_set_layout()` computes the initial sprite grid dimensions: `xnum = MIN(max_texture_size / cell_width, UINT16_MAX)`, `max_y = MIN(max_texture_size / cell_height, UINT16_MAX)`

**GPU Texture Atlas Initialization (answers: atlas sizing, capacity)**

- `kitty/main.py` → `_run_app()` calls `create_os_window()` which triggers window creation
- `kitty/fonts.c` → `send_prerendered_sprites_for_window()` is called on first window display, allocating the sprite map via `alloc_sprite_map(cell_width, cell_height)`
- `kitty/shaders.c` → `alloc_sprite_map()` queries `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS`, calls `sprite_tracker_set_limits()`, creates a `SpriteMap` struct with `xnum=1, ynum=1, last_num_of_layers=1`
- `kitty/shaders.c` → `realloc_sprite_texture()` creates the initial `GL_TEXTURE_2D_ARRAY` texture via `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)` using `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping
- `kitty/fonts.c` → `send_prerendered_sprites()` uploads blank cell, then underline/strikethrough/missing-glyph/cursor pre-rendered sprites as the first entries in the atlas

### 0.4.2 Debug Output Integration Points

- `kitty/fonts.c` line 18: `#define debug debug_fonts` — macro gated by `global_state.debug_font_fallback`
- `kitty/state.h` line 16: `#define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }`
- `kitty/fonts.c` → `output_cell_fallback_data()` prints `U+XXXX` codepoints, bold/italic/emoji_presentation flags, and the chosen fallback face
- `kitty/fonts.c` → fallback failure path prints "The font chosen by the OS for the text: U+XXXX ... is [face] but it does not actually contain glyphs for that text"
- `kitty/fonts/render.py` → `dump_font_debug()` logs Normal/Bold/Italic/Bold-Italic faces and symbol map fonts via `identify_for_debug()`
- `kitty/debug_config.py` → `debug_config()` outputs current fonts, OpenGL version, paths, and non-default config options

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this is a read-only investigation with a single documentation deliverable, the execution plan consists of analysis steps that produce content for the output document:

**Group 1 — Source Analysis (Read-Only)**

- **READ**: `kitty/fonts.c` — Analyze `init_fonts()`, `load_hb_buffer()`, `shape()`, `calc_cell_metrics()`, `initialize_font_group()`, `fallback_font()`, `send_prerendered_sprites()`, `sprite_tracker_set_layout()`
- **READ**: `kitty/freetype.c` — Analyze `cell_metrics()` function (lines 387–403) for baseline, underline_position, underline_thickness, strikethrough_position, strikethrough_thickness computation
- **READ**: `kitty/fonts/render.py` — Analyze `set_font_family()`, `dump_font_debug()`, `prerender_function()`, and `render_special()` for decoration rendering parameters
- **READ**: `kitty/fonts/common.py` — Analyze `get_font_files()` and `get_font_from_spec()` for the font resolution chain
- **READ**: `kitty/fonts/fontconfig.py` — Analyze `find_best_match()`, `all_fonts_map()`, `font_for_family()` for Linux font discovery
- **READ**: `kitty/fontconfig.c` — Analyze `create_fallback_face()` for runtime fallback font selection logic
- **READ**: `kitty/shaders.c` — Analyze `alloc_sprite_map()`, `realloc_sprite_texture()`, `send_sprite_to_gpu()`, and `SpriteMap` struct for atlas initialization
- **READ**: `kitty/glyph-cache.c` / `kitty/glyph-cache.h` — Analyze `find_or_create_sprite_position()` for glyph-to-sprite mapping
- **READ**: `kitty/main.py` — Analyze `run_app.__call__()`, `_run_app()`, `init_glfw()` for the startup orchestration sequence
- **READ**: `kitty/debug_config.py` — Analyze `debug_config()` for diagnostic output format
- **READ**: `kitty/state.h` — Analyze `GlobalState` struct for `debug_font_fallback`, `force_ltr`, font metric fields
- **READ**: `kitty/unicode-data.h` — Analyze combining character, variation selector, and emoji detection APIs
- **READ**: `kitty/options/definition.py` — Analyze `force_ltr`, `symbol_map`, `font_family`, `font_size`, `modify_font` option definitions

**Group 2 — Optional Build Verification (Non-Destructive)**

- **BUILD**: Attempt to build kitty from source within the Docker container to verify native extension behavior, using `python3 setup.py build` with appropriate system dependencies
- **RUN**: If build succeeds, optionally run `kitty --debug-font-fallback` in a headless or test configuration to capture actual diagnostic output format for Arabic/English mixed text
- **RESTORE**: Ensure all build artifacts are in the build directory and no source files are modified

**Group 3 — Documentation Creation**

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — Synthesize all findings into a comprehensive answer document organized by the four question areas

### 0.5.2 Implementation Approach

The implementation follows a source-code-first approach:

- **Establish the knowledge foundation** by reading and analyzing all C and Python source files in the font, shaping, glyph-cache, and startup subsystems
- **Trace each startup code path** from `kitty/main.py` through the font resolution, cell metric computation, HarfBuzz initialization, and sprite map allocation sequences
- **Document the exact values and formulas** used for cell metrics, sprite atlas sizing, and HarfBuzz buffer configuration by extracting them directly from the C source
- **Explain the bidirectional text handling** by documenting how `hb_buffer_guess_segment_properties()` interacts with `force_ltr` and how Arabic RTL/English LTR mixing is handled at the HarfBuzz level
- **Explain the font fallback chain** by documenting how `create_fallback_face()` constructs fontconfig patterns with charset constraints, and how the fallback map cache (`fallback_font_map_t`) avoids redundant lookups
- **Compile all findings** into a well-structured markdown document with code references, explaining the rationale behind each answer

### 0.5.3 User Interface Design

Not applicable — this task produces a documentation artifact only. No UI changes are involved.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Source files analyzed for the investigation (read-only):**

- `kitty/fonts.c` — Core font engine, HarfBuzz shaping, fallback dispatch, cell metrics, sprite tracking
- `kitty/fonts.h` — Font API header
- `kitty/freetype.c` — FreeType face management, `cell_metrics()` computation
- `kitty/fontconfig.c` — Native fontconfig integration, `create_fallback_face()`
- `kitty/core_text.m` — macOS CoreText integration (context only)
- `kitty/font-names.c` — Font name table reading
- `kitty/glyph-cache.c` — Sprite position hash table
- `kitty/glyph-cache.h` — Glyph cache data types
- `kitty/shaders.c` — SpriteMap, GL texture atlas management
- `kitty/shaders.py` — Python-side shader loading
- `kitty/unicode-data.c` — Unicode property tables
- `kitty/unicode-data.h` — Combining char, variation selector, emoji APIs
- `kitty/emoji.h` — Emoji detection
- `kitty/data-types.h` — Core types (CPUCell, FONTS_DATA_HEAD, sprite types)
- `kitty/state.h` — GlobalState (debug_font_fallback, force_ltr, font metric adjustments)
- `kitty/state.c` — `set_options()` applying debug flags
- `kitty/main.py` — Startup orchestration
- `kitty/debug_config.py` — Diagnostic report generation
- `kitty/cli.py` — CLI option definitions
- `kitty/fonts/render.py` — Font rendering setup, `dump_font_debug()`
- `kitty/fonts/common.py` — Platform-neutral font resolution
- `kitty/fonts/fontconfig.py` — Linux fontconfig backend
- `kitty/fonts/core_text.py` — macOS CoreText backend (context only)
- `kitty/fonts/__init__.py` — Font data model types
- `kitty/fonts/box_drawing.py` — Box-drawing glyph rasterization (pre-rendered sprites)
- `kitty/fonts/list.py` — Font enumeration
- `kitty/options/definition.py` — Option definitions (force_ltr, font_family, symbol_map, modify_font)
- `kitty/options/types.py` — Options type with default values
- `kitty_tests/fonts.py` — Font selection and rendering tests (reference)
- `kitty/gl.c` / `kitty/gl.h` — OpenGL wrapper infrastructure
- `kitty/cell_vertex.glsl` / `kitty/cell_fragment.glsl` — Cell rendering shaders

**Files to create:**

- `blitzy/documentation/kitty_815df1e210e0.md` — Investigation answer document

**Configuration and build files consulted:**

- `pyproject.toml` — Python version requirement (≥ 3.8)
- `go.mod` — Go module version (1.22)
- `setup.py` — Build system, native extension compilation, HarfBuzz ≥ 1.5 requirement
- `Makefile` — Build targets

### 0.6.2 Explicitly Out of Scope

- **Source file modifications** — No existing source files in the kitty repository will be modified under any circumstances
- **New feature code** — No runtime code, tests, or configurations will be added to the source tree
- **Kittens subsystem** — Built-in kitten applications (`kittens/`) are unrelated to font/shaping startup initialization
- **Shell integration** — `shell-integration/` scripts are not relevant to the font/rendering pipeline
- **Remote control protocol** — `kitty/remote_control.py`, `kitty/rc/` are unrelated
- **Graphics protocol** — `kitty/graphics.c`, image handling is not part of text shaping
- **File transmission** — `kitty/file_transmission.py` is unrelated
- **CI/CD and packaging** — `.github/`, `bypy/`, `publish.py` are not relevant
- **Performance optimization** — No profiling or optimization beyond documenting existing behavior
- **macOS-specific deep analysis** — CoreText path is referenced for context but detailed analysis focuses on the Linux/fontconfig path as the investigation runs in a Linux Docker container
- **Runtime rendering behavior** — The investigation focuses on initialization/startup values; ongoing rendering behavior after the first frame is out of scope

## 0.7 Rules for Feature Addition

### 0.7.1 Project-Specified Rules

The following rules are explicitly emphasized by the user and the project configuration:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt
- **Build and run the source code** to analyze the repository behavior as needed
- **Do not make assumptions** — base all answers on the code as the truth
- **Provide thinking / rationale** behind the answers
- **Do not modify any existing files** in the source repository
- **Do not add any other code** in the source repository besides the requested document
- **Place the generated document** in the `blitzy/documentation` directory in the destination repo

### 0.7.2 Investigation-Specific Constraints

- **Non-destructive observation**: Debug output or runtime flags for observation are permitted, but all settings must be restored to their original state when complete
- **Code as ground truth**: Every claim in the answer document must be traceable to a specific source file, function, and line number in the kitty codebase
- **Comprehensive coverage**: All four question areas (text shaping/unicode, font families/fallback, cell metrics/decorations, GPU atlas) must be answered completely
- **No speculative answers**: If a value cannot be determined statically from the source code and requires a running build that cannot be produced, this must be explicitly stated rather than guessed

### 0.7.3 Documentation Standards

- The output document must use clear markdown formatting with code blocks, tables, and section headings
- Source file references should include the file path and relevant function names
- Formulas extracted from C source should be presented in a readable pseudo-code or mathematical notation
- The document should be self-contained and understandable without requiring the reader to cross-reference the source code

## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were comprehensively searched across the codebase to derive conclusions for this Agent Action Plan:

**Root-level files consulted:**
- `pyproject.toml` — Python version constraints, tooling configuration
- `go.mod` — Go module version and dependency graph
- `setup.py` — Build system, native extension discovery, HarfBuzz/FreeType/fontconfig integration
- `Makefile` — Build command surface

**Core font and shaping subsystem (full content read):**
- `kitty/fonts.c` (1761 lines) — Complete font engine including HarfBuzz shaping, fallback dispatch, cell metrics, sprite tracking, ligature grouping
- `kitty/fonts.h` (64 lines) — Font API header declaring backend interfaces
- `kitty/freetype.c` (1037 lines) — FreeType face management, cell_metrics computation, glyph rasterization, identify_for_debug
- `kitty/fontconfig.c` — Native fontconfig integration, fc_list, fc_match, create_fallback_face (selected sections)
- `kitty/glyph-cache.c` (91 lines) — Complete sprite position and glyph properties hash tables
- `kitty/glyph-cache.h` (35 lines) — Complete data type definitions for glyph cache

**Python font subsystem (full content read):**
- `kitty/fonts/render.py` (531 lines) — Complete font rendering setup, debug dump, pre-rendering, symbol maps
- `kitty/fonts/common.py` (403 lines) — Complete platform-neutral font resolution workflow
- `kitty/fonts/fontconfig.py` (273 lines) — Complete Linux fontconfig backend

**GPU and shader subsystem (selected sections):**
- `kitty/shaders.c` — SpriteMap struct, alloc_sprite_map, realloc_sprite_texture, send_sprite_to_gpu, ensure_sprite_map (lines 1–165)
- `kitty/data-types.h` — FONTS_DATA_HEAD macro, FONTS_DATA_HANDLE type, CPUCell struct

**Application startup and configuration:**
- `kitty/main.py` (full content read) — Complete startup orchestration from CLI parsing through Boss creation
- `kitty/debug_config.py` (full content read) — Complete diagnostic report generation
- `kitty/cli.py` — Debug flag definitions (--debug-font-fallback, --debug-rendering)
- `kitty/state.h` — GlobalState struct definition, debug_font_fallback, debug_rendering, force_ltr, font metric fields
- `kitty/state.c` — set_options function applying debug flags to global state
- `kitty/options/definition.py` — force_ltr, symbol_map, font_family configuration definitions

**Unicode support:**
- `kitty/unicode-data.h` — Combining character, variation selector (VS15/VS16), emoji detection API declarations

**Test reference:**
- `kitty_tests/fonts.py` — Font selection and rendering test patterns (first 80 lines)

**Folder structure explored:**
- Root (`""`) — Complete repository structure
- `kitty/` — Full listing of all core source files and subdirectories
- `kitty/fonts/` — Complete Python font subsystem
- `kitty/options/` — Configuration option types and parsing

### 0.8.2 Attachments

No external attachments were provided for this project. No Figma URLs or design files are associated with this task.

### 0.8.3 Technical Specification Sections Consulted

The following sections from the existing technical specification were retrieved to provide additional architectural context:

- **4.2 APPLICATION STARTUP FLOW** — Python bootstrap and Boss initialization sequence; entry point dispatcher; startup orchestration steps 1–10
- **4.3 TERMINAL INPUT/OUTPUT PIPELINE** — GPU rendering pipeline architecture; font pipeline from discovery through rasterization to texture cache; threaded rendering model
- **7.2 GPU Rendering Pipeline** — Six-stage OpenGL shader pipeline; font rendering pipeline (FontConfig/CoreText → FreeType → HarfBuzz → Glyph Cache → Texture Atlas → Cell Shaders); OpenGL extension usage; rendering performance architecture

### 0.8.4 Environment

- **Docker Container**: `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
- **Source Branch**: `kitty_815df1e210e0`
- **Repository**: kitty terminal emulator by Kovid Goyal
- **Commit**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

