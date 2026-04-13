# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to produce a comprehensive investigative analysis document for the kitty terminal emulator codebase, answering detailed technical questions about the font rendering and GPU subsystem initialization paths. Specifically:

- **Text Shaping and Unicode Configuration at Startup**: Determine how kitty's text shaping engine (HarfBuzz) and font pipeline configure support for complex Unicode scripts, including OpenType ligatures, bidirectional (BiDi) text, and combining diacritical marks, during the initial startup sequence
- **Font Fallback Chain Diagnostics**: Identify exactly which font families and fallback chains kitty selects at launch (with verbose logging enabled via `--debug-font-fallback`) for handling mixed Arabic (RTL) and English (LTR) text, and what runtime configuration values confirm these selections before any text rendering begins
- **Cell Metrics and Decoration Alignment at Grid Initialization**: Document the default cell metrics (cell_width, cell_height), baseline positioning, and decoration alignment values (underline_position, underline_thickness, strikethrough_position, strikethrough_thickness) reported in debug output as the screen grid and shaping subsystems initialize for complex grapheme clusters
- **GPU Texture Atlas Initialization**: Detail the initial page layout, sizing (xnum, ynum), capacity allocations (max_texture_size, max_array_texture_layers), and startup logs for the sprite map / texture atlas used by the GPU rendering pipeline, verifying readiness to receive shaped glyph data from primary and fallback fonts
- **Observation-Only Constraint**: No source files may be modified; enabling debug output or runtime flags (e.g., `--debug-font-fallback`, `--debug-rendering`) for observation is permitted, but all settings must be restored to their original state when complete

### 0.1.2 Special Instructions and Constraints

- **No Source Modifications**: The user has explicitly stated: "Do not modify any source files." This is reinforced by the implementation rule `SWE-AtlasQnA-Repo` which mandates: "Do not modify any existing files in the source repository" and "Do not add any other code in the source repository (besides the above requested document)"
- **Documentation Output**: Per the `SWE-AtlasQnA-Repo` rule, the deliverable is a single markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory, comprehensively answering all posed questions with rationale grounded in the code
- **Debug Flags Are Permissible**: The user allows enabling debug output or runtime flags for observation purposes — the relevant CLI flags are `--debug-font-fallback` and `--debug-rendering` (also `--debug-gl`)
- **Restore State**: Any configuration changes made for observation must be reverted to the original state upon completion
- **Code-as-Truth Principle**: All answers must be derived from the actual source code, not assumptions or external documentation

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the text shaping configuration questions**, we will trace the HarfBuzz initialization and shaping pipeline in `kitty/fonts.c` (specifically `load_hb_buffer()` and the `hb_shape()` call path), `kitty/freetype.c` (for `init_ft_face()` and HarfBuzz font creation), and `kitty/unicode-data.c` (for combining character detection built from Unicode Standard 15.0.0)
- To **document font fallback chains**, we will analyze `kitty/fonts/common.py` (`get_font_files()`, `get_font_from_spec()`), `kitty/fonts/fontconfig.py` (the Linux backend's `all_fonts_map()`, `fc_match()`, `create_fallback_face()`), `kitty/fontconfig.c` (the C-level fallback face creation using FontConfig charset matching), and `kitty/fonts/render.py` (`dump_font_debug()`, `set_font_family()`)
- To **document cell metrics and decoration values**, we will trace `calc_cell_metrics()` in `kitty/fonts.c` which calls `cell_metrics()` in `kitty/freetype.c`, analyzing how baseline, underline_position/thickness, and strikethrough_position/thickness are computed from FreeType font units and subsequently adjusted by `modify_font` configuration directives
- To **document GPU texture atlas initialization**, we will analyze `alloc_sprite_map()` in `kitty/shaders.c`, `sprite_tracker_set_layout()` in `kitty/fonts.c`, and the `send_prerendered_sprites()` function which populates the initial sprite atlas with blank, underline, strikethrough, missing-glyph, and cursor sprites
- To **produce the deliverable**, we will create a single comprehensive markdown document at `blitzy/documentation/kitty_815df1e210e0.md` containing all findings with code-grounded rationale

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation spans the following major subsystems of the kitty terminal emulator (a C/Python/Go/GLSL codebase). All files below were examined to derive the answers for the documentation deliverable.

**Font Subsystem — Core Pipeline (C layer)**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/fonts.c` | Central font orchestration: font group management, HarfBuzz shaping (`load_hb_buffer`, `hb_shape`), sprite position tracking, cell rendering, fallback font loading (`load_fallback_font`, `fallback_font`), cell metric calculation (`calc_cell_metrics`), and symbol map handling | Primary: shaping, fallback, cell metrics |
| `kitty/fonts.h` | Font API header: declares `cell_metrics()`, `harfbuzz_font_for_face()`, `create_fallback_face()`, `render_glyphs_in_cells()`, `set_size_for_face()`, sprite tracker interfaces, and the `StringCanvas` type | Primary: API contracts for the font pipeline |
| `kitty/freetype.c` | FreeType face initialization (`init_ft_face`), HarfBuzz font creation (`hb_ft_font_create`), `cell_metrics()` implementation computing cell_width/height/baseline/underline/strikethrough from font units, glyph loading and rendering | Primary: cell metrics, font initialization |
| `kitty/fontconfig.c` | FontConfig integration: dynamically-loaded libfontconfig bindings, `fc_match()` for font matching, `create_fallback_face()` for Linux fallback using charset-based matching against the "monospace" or "emoji" family | Primary: Linux fallback chain |
| `kitty/core_text.m` | CoreText integration for macOS: `create_fallback_face()` macOS implementation for font fallback using CTFontCreateForString | Primary: macOS fallback chain |
| `kitty/glyph-cache.c` | Glyph cache hash table: `find_or_create_sprite_position()` using uthash for O(1) glyph-to-sprite-position lookup, `find_or_create_glyph_properties()` | Supporting: glyph caching layer |
| `kitty/glyph-cache.h` | Glyph cache types: `SpritePosition` (rendered, colored, x, y, z coordinates), `GlyphProperties` | Supporting: type definitions |
| `kitty/unicode-data.c` | Unicode Standard 15.0.0 tables: `is_combining_char()`, `is_emoji()`, `is_symbol()`, `codepoint_for_mark()`, `is_non_rendered_char()` | Primary: Unicode combining/emoji classification |
| `kitty/unicode-data.h` | Unicode data header: `VS15`/`VS16` variation selectors, function declarations, `is_private_use()` | Primary: variation selector constants |
| `kitty/font-names.c` | Font name table parsing from TrueType/OpenType `name` and `fvar`/`STAT` tables | Supporting: variable font metadata |

**Font Subsystem — Python Orchestration Layer**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/fonts/render.py` | Runtime font setup: `set_font_family()` resolves medium/bold/italic/bi faces plus symbol_map fonts, calls `set_font_data()` into C layer; `dump_font_debug()` prints loaded font diagnostics; `prerender_function()` pre-renders underline/strikethrough/cursor sprites | Primary: startup font init, debug output |
| `kitty/fonts/common.py` | Platform-neutral font resolution: `get_font_files()` resolves all four font faces, `get_font_from_spec()` / `get_fine_grained_font()` for fine-grained matching including variable font specialization, `find_best_match()`, `find_bold_italic_variant()` | Primary: font resolution logic |
| `kitty/fonts/fontconfig.py` | Linux fontconfig Python backend: `all_fonts_map()` builds family/ps/full/variable maps via `fc_list()`, `fc_match()` wrapper, `create_scorer()`, `find_best_match()`, `find_last_resort_text_font()` | Primary: Linux font discovery |
| `kitty/fonts/core_text.py` | macOS CoreText Python backend: equivalent discovery and matching functions for macOS | Supporting: macOS font discovery |
| `kitty/fonts/__init__.py` | Shared font type definitions: `FontSpec`, `Descriptor`, `VariableData`, `NamedStyle`, `DesignAxis`, `Scorer`, `family_name_to_key()` | Supporting: shared types |
| `kitty/fonts/list.py` | Font enumeration and JSON export for the `choose-fonts` kitten | Supporting: font listing |
| `kitty/fonts/box_drawing.py` | Box-drawing and braille character rasterization: `render_box_char()`, `render_missing_glyph()` | Supporting: box-drawing special rendering |

**GPU Rendering and Shader Pipeline**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/shaders.c` | Sprite map allocation (`alloc_sprite_map`), texture atlas reallocation (`realloc_sprite_texture`), `send_sprite_to_gpu()` using `GL_TEXTURE_2D_ARRAY` with `GL_SRGB8_ALPHA8`, OpenGL extension queries (`GL_MAX_TEXTURE_SIZE`, `GL_MAX_ARRAY_TEXTURE_LAYERS`) | Primary: GPU atlas initialization |
| `kitty/shaders.py` | Python-side shader orchestration: GLSL source loading, macro substitution | Supporting: shader management |
| `kitty/gl.c` / `kitty/gl.h` | GLAD-loaded OpenGL binding layer, shader compilation, VAO/buffer helpers | Supporting: OpenGL infrastructure |
| `kitty/cell_vertex.glsl` / `kitty/cell_fragment.glsl` | Cell rendering shaders: text glyph compositing, cursor drawing | Supporting: glyph rendering |
| `kitty/data-types.h` | Core type definitions: `pixel`, `glyph_index`, `sprite_index`, `char_type`, OpenGL version constants (`OPENGL_REQUIRED_VERSION_MAJOR/MINOR`) | Supporting: type system |

**Startup and Configuration**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/main.py` | Application entry: `_main()` → `set_locale()` → `init_glfw()` → `run_app()` → `set_font_family()` → `_run_app()` → `dump_font_debug()` (if `--debug-font-fallback`) | Primary: startup flow |
| `kitty/debug_config.py` | `debug_config()`: prints version, OpenGL info, loaded fonts via `current_fonts()` + `identify_for_debug()`, config diffs | Primary: runtime diagnostics |
| `kitty/state.h` | Global state struct, `Options` struct (containing `force_ltr`, `disable_ligatures`, font adjustment fields), debug macros (`debug_fonts`, `debug_rendering`) | Primary: runtime flags |
| `kitty/options/definition.py` | Configuration schema: `font_family`, `font_size`, `force_ltr`, `symbol_map`, `font_features`, `disable_ligatures`, `modify_font`, `text_composition_strategy`, `undercurl_style` | Primary: user-facing config |
| `kitty/options/types.py` | Generated `Options` class with typed fields for all settings | Supporting: typed config |
| `kitty/cli.py` | CLI argument definitions including `--debug-font-fallback`, `--debug-rendering`, `--debug-gl`, `--debug-input` | Primary: debug flag CLI |
| `kitty/entry_points.py` | Command dispatcher: routes to `kitty.main.main()` for GUI mode | Supporting: entry flow |
| `kitty/boss.py` | Boss controller: session creation, OS window provisioning | Supporting: startup orchestration |

**Test Infrastructure**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty_tests/fonts.py` | Font selection tests, shaping tests, render tests using `setup_for_testing()` context manager | Supporting: validation patterns |
| `kitty_tests/CascadiaCode-Regular.otf` | Test font with ligature support | Supporting: test assets |
| `kitty_tests/FiraCode-Medium.otf` | Test font with programming ligatures | Supporting: test assets |
| `kitty_tests/LiberationMono-Regular.ttf` | Monospace test font | Supporting: test assets |
| `kitty_tests/iosevka-regular.ttf` | Variable-weight test font (Iosevka) | Supporting: test assets |

### 0.2.2 Integration Point Discovery

- **HarfBuzz Integration Points**: `kitty/freetype.c:init_ft_face()` creates `hb_font_t` via `hb_ft_font_create()` and sets load flags with `hb_ft_font_set_load_flags()`; `kitty/fonts.c:load_hb_buffer()` fills the HarfBuzz buffer and calls `hb_buffer_guess_segment_properties()` for script/direction auto-detection; `kitty/fonts.c` line 813 calls `hb_shape()` with per-font feature arrays
- **FontConfig Fallback Path**: `kitty/fontconfig.c:create_fallback_face()` creates an FcPattern for "monospace" (or "emoji") family, adds the cell's charset, calls `fc_match()`, and returns either an existing face index or a new face from the descriptor
- **OpenGL Atlas Path**: `kitty/shaders.c:alloc_sprite_map()` → queries GPU limits → `kitty/fonts.c:sprite_tracker_set_layout()` computes grid dimensions → `kitty/fonts.c:send_prerendered_sprites()` uploads initial sprite data → `kitty/shaders.c:realloc_sprite_texture()` allocates `GL_TEXTURE_2D_ARRAY` storage
- **Debug Output Path**: `kitty/state.h` defines `debug_fonts(...)` macro gated by `global_state.debug_font_fallback`; `kitty/main.py:_run_app()` calls `dump_font_debug()` when `args.debug_font_fallback` is set; `kitty/debug_config.py:debug_config()` iterates `current_fonts()` and calls `identify_for_debug()` on each face

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive markdown document answering all user questions about text shaping, font fallback, cell metrics, and GPU atlas initialization, with code-grounded rationale
- No other new files are required per the implementation rules

## 0.3 Dependency Inventory

### 0.3.1 Key Packages Relevant to This Investigation

The following packages are directly relevant to the font shaping, rendering, and GPU subsystems under investigation. Versions are taken from the project's `setup.py`, `pyproject.toml`, and `go.mod` dependency manifests.

| Registry | Package | Version Constraint | Purpose |
|----------|---------|-------------------|---------|
| System (pkg-config) | HarfBuzz | ≥ 1.5 | OpenType text shaping for ligatures and complex scripts; `hb_shape()`, `hb_buffer_guess_segment_properties()` for script/direction auto-detection |
| System (pkg-config) | FreeType | ≥ 2.x (scalable font rasterization) | Glyph rasterization, cell metric computation (`cell_metrics()`), HarfBuzz font creation via `hb_ft_font_create()` |
| System (pkg-config) | FontConfig | System library (dynamically loaded) | Linux font discovery and matching via `fc_list()`, `fc_match()`, fallback face creation |
| System (pkg-config) | lcms2 | Required (no minimum specified) | ICC color management for sRGB gamma correction |
| System (pkg-config) | OpenSSL/libcrypto | Required (version auto-detected) | Cryptographic operations for remote control transport |
| System | libpng | Required | PNG decoding for graphics protocol and window icons |
| System | OpenGL | ≥ 3.1 (Linux) / ≥ 3.3 (macOS) | GPU rendering pipeline; `GL_TEXTURE_2D_ARRAY`, `GL_SRGB8_ALPHA8` |
| PyPI | Python | ≥ 3.8 | Runtime interpreter (per `pyproject.toml` `requires-python`) |
| Vendored | GLFW 3.4 (fork) | Vendored in `glfw/` | Platform windowing, input handling, OpenGL context management |
| Go modules | Go | 1.22 | Go-based tooling (`tools/` package tree) |
| Unicode | Unicode Standard | 15.0.0 | Character property tables generated into `kitty/unicode-data.c` |

### 0.3.2 Dependency Details for Investigation

- **HarfBuzz ≥ 1.5**: Enforced at build time by `setup.py` line 609: `at_least_version('harfbuzz', 1, 5)`. HarfBuzz is the text shaping engine responsible for ligature formation, combining mark attachment, script detection, and bidirectional text direction inference through `hb_buffer_guess_segment_properties()`
- **FreeType**: Loaded and linked via `setup.py` build system. The `Face` struct in `kitty/freetype.c` stores ascender, descender, height, underline_position, underline_thickness values from the font's OS/2 and post tables, used by `cell_metrics()` to compute pixel-level cell dimensions
- **FontConfig**: Dynamically loaded at runtime via `dlopen("libfontconfig.so")` in `kitty/fontconfig.c`. Provides `FcFontMatch()` for charset-based fallback font selection. Not directly linked at compile time
- **OpenGL Extensions**: The sprite map system queries `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS` at first use (`kitty/shaders.c:alloc_sprite_map()`). On macOS, values are clamped to 8192 and 512 respectively. Uses `GL_ARB_texture_storage` for immutable allocation and `GL_ARB_copy_image` for efficient texture copies (with a software fallback path)

### 0.3.3 Import and Reference Updates

Since this task produces only a documentation deliverable (a single markdown file), no import updates, build file modifications, or CI/CD changes are required. All dependency information is referenced in the output document for context.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation traces through the following critical integration points. No modifications are made; these are documented to show the data and control flow the markdown deliverable must explain.

**Startup Font Initialization Chain**

- `kitty/main.py:_main()` → `create_opts()` → `init_glfw()` → `run_app()` which calls `set_font_family(opts)` (from `kitty/fonts/render.py`) before `_run_app()`
- `kitty/fonts/render.py:set_font_family()` → calls `get_font_files(opts)` (from `kitty/fonts/common.py`) to resolve medium/bold/italic/bi descriptors → creates `current_faces` list → calls `create_symbol_map(opts)` for symbol_map fonts → calls native `set_font_data()` to push resolved font data to the C layer
- `kitty/fonts.c:set_font_data()` (Python C API) → stores box_drawing/prerender/descriptor callbacks, font feature settings, and symbol maps; calls `free_font_groups()` to clear prior state
- `kitty/fonts.c:font_group_for()` → `add_font_group()` → `initialize_font_group()` which allocates the `Font` array, initializes medium/bold/italic/bi fonts via `initialize_font()`, sets up symbol fonts, calls `calc_cell_metrics()`, and rescales symbol map faces to the computed cell height

**HarfBuzz Shaping Pipeline**

- `kitty/fonts.c:load_hb_buffer()` fills `hb_buffer_t` with UTF-32 codepoints from `CPUCell.ch` plus combining characters from `CPUCell.cc_idx[]` (resolved via `codepoint_for_mark()`)
- `hb_buffer_guess_segment_properties(harfbuzz_buffer)` auto-detects script (e.g., `HB_SCRIPT_ARABIC`), direction (e.g., `HB_DIRECTION_RTL`), and language
- If `OPT(force_ltr)` is true, direction is overridden: `hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR)`
- `hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features)` performs the actual shaping with per-font feature arrays (LIGA, DLIG, CALT, or user-specified `font_features`)

**Font Fallback Resolution**

- `kitty/fonts.c:font_for_cell()` determines which font handles each cell: checks for blank/box glyphs → consults symbol maps → selects bold/italic variant → checks if main font has the glyph (`has_cell_text()`) → falls back to `fallback_font()` if not
- `fallback_font()` checks a hash-map cache (`fallback_font_map`), and on cache miss calls `load_fallback_font()` which invokes platform-specific `create_fallback_face()`
- On Linux: `kitty/fontconfig.c:create_fallback_face()` creates an FcPattern for "monospace" family (or "emoji" for emoji presentation), adds the cell's unicode codepoints as a charset, calls `fc_match()`, and returns the matched face
- On macOS: `kitty/core_text.m:create_fallback_face()` uses CoreText's `CTFontCreateForString` for native fallback resolution
- When `global_state.debug_font_fallback` is true, `output_cell_fallback_data()` prints the codepoint(s), style flags, and selected face to stderr

**Cell Metric Computation Chain**

- `kitty/fonts.c:calc_cell_metrics()` → calls `cell_metrics()` on the medium font face
- `kitty/freetype.c:cell_metrics()` computes: `cell_width` = max advance width in pixels, `cell_height` from font height (with underscore-overflow correction), `baseline` = ascender in pixels, `underline_position` from ascender minus font underline_position, `underline_thickness`, `strikethrough_position` from OS/2 table `yStrikeoutPosition`, `strikethrough_thickness` from `yStrikeoutSize`
- Back in `calc_cell_metrics()`: `cell_width`/`cell_height` are adjusted by `modify_font cell_width`/`cell_height` config; baseline/underline/strikethrough positions are adjusted by their respective `modify_font` settings; underline position is clamped to `cell_height - 1`

**GPU Texture Atlas Initialization Chain**

- `kitty/fonts.c:send_prerendered_sprites_for_window()` → `alloc_sprite_map()` (from `kitty/shaders.c`) + `send_prerendered_sprites()`
- `alloc_sprite_map()` queries `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS`, calls `sprite_tracker_set_limits()` (which stores the values in `kitty/fonts.c` statics, capping array length at `0xfff`)
- `sprite_tracker_set_layout()` computes: `xnum = MIN(max_texture_size / cell_width, UINT16_MAX)`, `max_y = MIN(max_texture_size / cell_height, UINT16_MAX)`, initializes `ynum=1, x=0, y=0, z=0`
- `send_prerendered_sprites()` uploads: blank cell → underline sprites (1–5 styles) → strikethrough sprite → missing glyph sprite → cursor sprites (beam, underline, hollow) via `current_send_sprite_to_gpu()`
- `realloc_sprite_texture()` allocates `GL_TEXTURE_2D_ARRAY` with `GL_SRGB8_ALPHA8` format, dimensions `(xnum * cell_width, ynum * cell_height, z+1)`

### 0.4.2 Debug Output Integration Points

| Debug Flag | CLI Argument | C Macro / Python Check | Output Location |
|------------|-------------|----------------------|-----------------|
| Font fallback | `--debug-font-fallback` | `global_state.debug_font_fallback` / `debug_fonts(...)` in `kitty/state.h` | stderr via `timed_debug_print()` |
| Font debug dump | `--debug-font-fallback` | `args.debug_font_fallback` check in `kitty/main.py:_run_app()` | stderr via `dump_font_debug()` in `kitty/fonts/render.py` |
| Rendering debug | `--debug-rendering` / `--debug-gl` | `global_state.debug_rendering` / `debug_rendering(...)` | stderr via `timed_debug_print()` |
| Config debug | `kitty --debug-config` (or `Ctrl+Shift+F6`) | `debug_config()` in `kitty/debug_config.py` | stdout (formatted) |

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since no source files are modified, the execution plan consists of a single deliverable file. The markdown document must synthesize findings from all analyzed source files into clear, code-grounded answers.

**Group 1 — Deliverable Document**

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md`
  - Comprehensive Q&A document answering all four technical question areas
  - Must include thinking/rationale behind each answer
  - Must base answers solely on the codebase as ground truth
  - Must cover: text shaping configuration, font fallback chains, cell metrics/decorations, GPU atlas initialization

### 0.5.2 Implementation Approach — Document Content Structure

The document `kitty_815df1e210e0.md` must be organized around the user's four core question areas, with each section providing code-traced evidence.

**Section 1 — Text Shaping and Unicode Configuration at Startup**

- Trace the HarfBuzz initialization in `kitty/freetype.c:init_ft_face()`: `hb_ft_font_create(self->face, NULL)` creates a HarfBuzz font backed by the FreeType face, then `hb_ft_font_set_load_flags()` aligns the HarfBuzz load flags with kitty's hinting settings
- Document the global HarfBuzz buffer initialization in `kitty/fonts.c`: a static `hb_buffer_t *harfbuzz_buffer` is created once and reused for all shaping operations
- Document the three default HarfBuzz features (`hb_features[3]`): `LIGA_FEATURE`, `DLIG_FEATURE`, `CALT_FEATURE` — with CALT always enabled by default for all fonts, and LIGA/DLIG additionally enabled for NimbusMonoPS fonts
- Explain how per-font features are applied from the `font_features` config option through `font_feature_settings` dict in `init_font()`
- Explain BiDi handling: `hb_buffer_guess_segment_properties()` auto-detects script and direction from the buffer content; Arabic text → `HB_SCRIPT_ARABIC` + `HB_DIRECTION_RTL`; English text → `HB_SCRIPT_LATIN` + `HB_DIRECTION_LTR`; the `force_ltr` option overrides direction to `HB_DIRECTION_LTR` when enabled
- Document combining diacritics support: `CPUCell.cc_idx[]` stores combining marks as compressed indices, resolved to full codepoints via `codepoint_for_mark()`; these are appended to the HarfBuzz buffer alongside the base character; HarfBuzz's compose function (`hb_unicode_compose`) handles precomposed form detection in `has_cell_text()`
- Reference `kitty/unicode-data.c` as built from Unicode Standard 15.0.0 with `is_combining_char()` covering 6424 codepoints including Arabic diacritics (U+0610–U+061A, U+064B–U+065F, U+0670, etc.)

**Section 2 — Font Families and Fallback Chains in Startup Diagnostics**

- Document the `--debug-font-fallback` CLI flag and its effect: sets `global_state.debug_font_fallback = true`, which activates the `debug_fonts(...)` C macro for fallback tracing, and triggers `dump_font_debug()` in `_run_app()` after boss startup
- Document `dump_font_debug()` output format: iterates `current_fonts()` dict printing "Normal:", "Bold:", "Italic:", "Bold-Italic:" with each face's `identify_for_debug()` output (format: `PostScriptName: /path/to/font.ttf:face_index`), plus symbol map fonts
- Document the font resolution path for the default `monospace` family: `get_font_from_spec()` → `find_best_match('monospace', ...)` → FontConfig matches the system's default monospace font
- Explain fallback chain construction: no static fallback chain is pre-built at startup; instead, fallback fonts are loaded lazily in `load_fallback_font()` when `font_for_cell()` determines the main font lacks a glyph; the debug output prints each fallback lookup with codepoint, style flags, and selected font face
- For mixed Arabic/English text: HarfBuzz auto-detects per-run script/direction via `hb_buffer_guess_segment_properties()`; if the primary monospace font lacks Arabic glyphs, `create_fallback_face()` queries FontConfig for a font matching "monospace" family with the Arabic codepoint's charset → the OS selects an Arabic-capable font (e.g., Noto Sans Arabic Mono, DejaVu Sans Mono, or similar)
- Document `debug_config()` output: prints "Fonts:" section with face identifiers, OpenGL version, config diffs showing `force_ltr`, `font_family`, `font_size` values

**Section 3 — Cell Metrics, Baseline, and Decoration Alignment**

- Document `calc_cell_metrics()` computation from `cell_metrics()` results:
  - `cell_width` = `ceil(max_advance_width * x_scale / 64.0)` from the medium font's FreeType metrics
  - `cell_height` = `ceil(height * y_scale / 64.0)`, with an underscore overflow correction that increases height if the underscore glyph renders below the bounding box
  - `baseline` = `ceil(ascender * y_scale / 64.0)`
  - `underline_position` = `MIN(cell_height-1, ceil((ascender - underline_position) * y_scale / 64.0))`
  - `underline_thickness` = `MAX(1, ceil(underline_thickness * y_scale / 64.0))`
  - `strikethrough_position` from OS/2 table `yStrikeoutPosition` or `floor(baseline * 0.65)` as fallback
  - `strikethrough_thickness` from OS/2 `yStrikeoutSize` or same as underline_thickness as fallback
- Document adjustment via `modify_font` config options: each metric can be adjusted by point, percent, or pixel units through `adjust_metric()`
- Document the line_height_adjustment effect: if cell_height changes from modify_font, baseline and underline_position are shifted by half the adjustment
- Note that these values are stored in the `FontGroup` struct fields: `cell_width`, `cell_height`, `baseline`, `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness`
- The `prerender_function()` in `render.py` receives all these metrics to pre-render underline styles (single, double, curly, dotted, dashed), strikethrough, missing glyph, and cursor sprites
- There is no explicit "overline" metric in kitty's implementation; overline is not a built-in decoration style

**Section 4 — GPU Texture Atlas Initialization**

- Document `alloc_sprite_map()` behavior at first use:
  - Queries `GL_MAX_TEXTURE_SIZE` (e.g., 16384 on modern GPUs) and `GL_MAX_ARRAY_TEXTURE_LAYERS` (e.g., 2048)
  - On macOS, caps to `MIN(8192, GL_MAX_TEXTURE_SIZE)` and `MIN(512, GL_MAX_ARRAY_TEXTURE_LAYERS)`
  - Calls `sprite_tracker_set_limits(max_texture_size, MIN(0xfff, max_array_len))`
- Document `sprite_tracker_set_layout()` grid calculation:
  - `xnum = MIN(MAX(1, max_texture_size / cell_width), UINT16_MAX)` — number of sprite columns
  - `max_y = MIN(MAX(1, max_texture_size / cell_height), UINT16_MAX)` — maximum number of sprite rows per layer
  - Initial state: `ynum = 1, x = 0, y = 0, z = 0`
  - Example: for cell_width=8, cell_height=16, max_texture_size=16384 → xnum=2048, max_y=1024 → theoretical capacity of 2048 × 1024 × 4095 = ~8.5 billion glyphs across all layers
- Document `realloc_sprite_texture()`:
  - Creates a `GL_TEXTURE_2D_ARRAY` with `GL_SRGB8_ALPHA8` internal format
  - Dimensions: `width = xnum * cell_width`, `height = ynum * cell_height`, `depth = z + 1`
  - Uses `GL_NEAREST` filtering (to prevent inter-cell glyph bleeding) and `GL_CLAMP_TO_EDGE` wrapping
  - When growth is needed, copies existing layers via `glCopyImageSubData` (or a software fallback if `GL_ARB_copy_image` is unavailable)
- Document `send_prerendered_sprites()` initial population:
  - Sprite index 0: blank cell (all zeros)
  - Sprites 1–5: five underline styles (single line, double line, curly, dotted, dashed) — count defined by `NUM_UNDERLINE_STYLES`
  - Sprite 6: strikethrough
  - Sprite 7: missing glyph placeholder (rendered by `render_missing_glyph()`)
  - Sprites 8–10: cursor shapes (beam, underline, hollow)
  - All uploaded via `current_send_sprite_to_gpu()` which calls `send_sprite_to_gpu()` → `glTexSubImage3D()` with `GL_RGBA` / `GL_UNSIGNED_INT_8_8_8_8`

### 0.5.3 Key Observations for the Deliverable

- kitty does **not** implement full BiDi (UAX #9) — it states this explicitly in the `force_ltr` option documentation: "kitty does not support BIDI (bidirectional text)." Instead, HarfBuzz shapes individual runs with auto-detected direction, and words in RTL scripts display in reversed visual order (word-level RTL, not character-level BiDi reordering). For full BiDi, users are directed to use GNU FriBidi as an external filter in conjunction with `force_ltr=yes`
- The fallback font chain is **not pre-built** at startup — it is constructed lazily on first encounter of unmapped codepoints. The `--debug-font-fallback` output thus appears incrementally as new codepoints are encountered during rendering, not as a startup-time dump
- The GPU atlas starts with a single texture layer and grows dynamically via `realloc_sprite_texture()` as more glyphs are cached. There is no pre-allocation of all font glyph pages at startup
- The `identify_for_debug()` output format is `"PostScriptName: /path/to/font.ttf:face_index"`, providing precise identification of each loaded font face

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Deliverable Files**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole output artifact

**Source Files Analyzed for the Deliverable (read-only)**
- Font pipeline (C): `kitty/fonts.c`, `kitty/fonts.h`, `kitty/freetype.c`, `kitty/fontconfig.c`, `kitty/core_text.m`, `kitty/font-names.c`
- Font pipeline (Python): `kitty/fonts/render.py`, `kitty/fonts/common.py`, `kitty/fonts/fontconfig.py`, `kitty/fonts/core_text.py`, `kitty/fonts/__init__.py`, `kitty/fonts/list.py`, `kitty/fonts/box_drawing.py`
- Glyph cache: `kitty/glyph-cache.c`, `kitty/glyph-cache.h`
- GPU rendering: `kitty/shaders.c`, `kitty/shaders.py`, `kitty/gl.c`, `kitty/gl.h`
- Unicode data: `kitty/unicode-data.c`, `kitty/unicode-data.h`
- Startup flow: `kitty/main.py`, `kitty/entry_points.py`, `kitty/boss.py`
- Configuration: `kitty/options/definition.py`, `kitty/options/types.py`, `kitty/options/parse.py`
- Debug output: `kitty/debug_config.py`, `kitty/cli.py`
- State management: `kitty/state.h`, `kitty/state.c`, `kitty/data-types.h`
- Shader sources: `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl`
- Tests: `kitty_tests/fonts.py`
- Build config: `setup.py`, `pyproject.toml`, `go.mod`

**Observation-Only Runtime Interactions**
- CLI flag: `--debug-font-fallback` — to observe font fallback selection output
- CLI flag: `--debug-rendering` / `--debug-gl` — to observe OpenGL/rendering diagnostics
- Runtime introspection: `kitty --debug-config` — to observe loaded font and configuration state
- All debug flags are command-line arguments that do not modify any source or configuration files

**Topics In Scope for the Document**
- HarfBuzz text shaping initialization (buffer creation, segment property guessing, feature configuration)
- Combining diacritical mark handling (cc_idx storage, codepoint_for_mark resolution, compose support)
- BiDi/RTL behavior and limitations (hb_buffer_guess_segment_properties, force_ltr override)
- Ligature feature management (LIGA, DLIG, CALT feature arrays, font_features config)
- Font discovery and matching (FontConfig on Linux, CoreText on macOS)
- Lazy fallback font loading and caching (fallback_font_map hash table)
- Cell metric computation (cell_width, cell_height, baseline, underline/strikethrough positions and thicknesses)
- Metric adjustment via modify_font config directives
- GPU sprite map allocation and limits (GL_MAX_TEXTURE_SIZE, GL_MAX_ARRAY_TEXTURE_LAYERS)
- Texture atlas format (GL_TEXTURE_2D_ARRAY, GL_SRGB8_ALPHA8)
- Pre-rendered sprite population (blank, underlines, strikethrough, missing glyph, cursors)
- Debug output format and content for `--debug-font-fallback` and `debug_config()`

### 0.6.2 Explicitly Out of Scope

- **Source code modifications**: No C, Python, Go, GLSL, or configuration files in the kitty repository are to be modified
- **Feature additions**: No new functionality is being added to kitty
- **Performance optimizations**: Not investigating or recommending performance changes
- **Graphics protocol** (`kitty/graphics.c`): Image handling and inline graphics are not part of this investigation
- **Kittens framework** (`kittens/`): Built-in terminal programs are not analyzed except for `choose_fonts` references
- **Shell integration** (`shell-integration/`): Shell integration scripts are not relevant to font/GPU initialization
- **Remote control** (`kitty/rc/`): The remote control system is not relevant
- **Go tooling** (`tools/`): The Go tools package is not involved in font rendering
- **GLFW platform backends** (`glfw/`): Windowing internals beyond OpenGL context creation are not analyzed
- **CI/CD and release engineering** (`.github/`, `bypy/`, `publish.py`): Build and release automation is not relevant
- **Non-font configuration options**: Only font-related and rendering-related config options are in scope

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Constraints

- **Read-Only Analysis**: The user explicitly stated: *"Do not modify any source files."* No C, Python, Go, GLSL, configuration, or any other source file in the kitty repository may be altered
- **Debug Observation Only**: The user permits *"enabling debug output or runtime flags for observation"* — this means command-line flags such as `--debug-font-fallback` and `--debug-rendering` may be used when describing how to observe behavior, but these are invocation-time arguments that do not change any persisted file
- **Restore to Original**: The user states: *"restore all settings to original when complete"* — since no files are modified and debug flags are transient CLI arguments, this constraint is inherently satisfied; the documentation must note that all observation methods are non-destructive

### 0.7.2 Implementation Rule: SWE-AtlasQnA-Repo

Per the project-level implementation rule `SWE-AtlasQnA-Repo`:

- **Deliverable format**: A single markdown document named `kitty_815df1e210e0.md` (matching the source branch name)
- **Deliverable location**: Placed in the `blitzy/documentation/` directory in the destination repository
- **Content requirements**: Comprehensively answer the question(s) posed in the prompt, including rationale and thinking behind each answer
- **Evidence-based answers**: Do not make assumptions; base all answers on the actual code as the source of truth
- **No source modifications**: Do not modify any existing files in the source repository
- **No additional code**: Do not add any other code in the source repository beyond the requested document

### 0.7.3 Documentation Quality Rules

- Every claim about runtime behavior must reference a specific source file and function with clear attribution (e.g., "`calc_cell_metrics()` in `kitty/fonts.c`")
- Configuration defaults must be cited from `kitty/options/definition.py` with exact default values
- Debug output format descriptions must trace to the actual C macros or Python functions that produce the output (e.g., `debug_fonts(...)` macro in `kitty/state.h`, `identify_for_debug()` in `kitty/freetype.c`)
- Fallback chain descriptions must accurately reflect the lazy-loading architecture and not overstate what is available at startup
- GPU atlas sizing must reference the exact OpenGL queries and cap values as coded in `kitty/shaders.c:alloc_sprite_map()`

### 0.7.4 Technical Accuracy Rules

- **No invented log output**: Any example diagnostic output shown must be clearly labeled as representative and derived from the code's format strings and print patterns
- **Platform differentiation**: Where behavior differs between Linux (FontConfig) and macOS (CoreText), both paths must be documented separately
- **Version specificity**: Unicode Standard version (15.0.0), HarfBuzz minimum (≥1.5), OpenGL requirements (≥3.1 Linux / ≥3.3 macOS) must be cited precisely
- **Lazy vs. eager distinction**: The document must clearly explain that font fallback chains are populated lazily at glyph-rendering time, not eagerly at startup — this is a critical architectural fact that affects the answer to the user's question about "what font families appear in startup diagnostics"

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were directly retrieved or searched during the analysis:

| Category | Path | Purpose |
|----------|------|---------|
| Root | `/` (repository root) | Top-level structure discovery |
| Core directory | `kitty/` | Main source directory listing |
| Font subsystem (C) | `kitty/fonts.c` (lines 1–700, 1434–1570) | FontGroup struct, calc_cell_metrics, font_for_cell, fallback_font, load_fallback_font, initialize_font_group, sprite_tracker_set_layout, HarfBuzz buffer loading, feature arrays |
| Font subsystem (C) | `kitty/fonts.h` (full) | Font backend API contracts: cell_metrics, harfbuzz_font_for_face, create_fallback_face, render_glyphs_in_cells |
| FreeType backend | `kitty/freetype.c` (lines 1–475) | FreeType face init, HarfBuzz font creation, cell_metrics implementation, identify_for_debug |
| FontConfig backend | `kitty/fontconfig.c` (lines 1–520) | Dynamic libfontconfig loading, create_fallback_face pattern matching |
| Font pipeline (Python) | `kitty/fonts/render.py` (full, 532 lines) | set_font_family, dump_font_debug, shape_string, create_symbol_map |
| Font pipeline (Python) | `kitty/fonts/common.py` (full, 404 lines) | get_font_files, get_font_from_spec, get_fine_grained_font |
| Font pipeline (Python) | `kitty/fonts/fontconfig.py` (lines 1–100) | all_fonts_map, fc_list, fc_match wrappers |
| Glyph cache | `kitty/glyph-cache.c` (full, 92 lines) | uthash sprite position and glyph properties tables |
| Glyph cache | `kitty/glyph-cache.h` (full, 36 lines) | SpritePosition struct, GlyphProperties struct |
| GPU rendering | `kitty/shaders.c` (lines 1–200) | SpriteMap struct, alloc_sprite_map, realloc_sprite_texture, send_sprite_to_gpu |
| Unicode data | `kitty/unicode-data.c` (first 50 lines) | Unicode 15.0.0 source, is_combining_char covering 6424 codepoints |
| Unicode data | `kitty/unicode-data.h` (full) | VS15/VS16, is_combining_char, codepoint_for_mark |
| Startup flow | `kitty/main.py` (full, 532 lines) | _main entry, create_opts, set_locale, init_glfw, set_font_family, run_app |
| Debug output | `kitty/debug_config.py` (full, 293 lines) | debug_config function, current_fonts, identify_for_debug usage |
| CLI definitions | `kitty/cli.py` (lines 989–1015) | --debug-font-fallback, --debug-rendering, --debug-gl flags |
| State management | `kitty/state.h` (lines 1–200) | Options struct, debug_fonts macro, debug_rendering macro, force_ltr |
| Data types | `kitty/data-types.h` (lines 1–80) | pixel/glyph_index/sprite_index/char_type typedefs, OpenGL version constants |
| Configuration defs | `kitty/options/definition.py` (key sections) | font_family, font_size, force_ltr, symbol_map, font_features, disable_ligatures, modify_font, text_composition_strategy |
| Configuration types | `kitty/options/types.py` | Options TypedDict, font adjustment types |
| Python type stubs | `kitty/fast_data_types.pyi` | CurrentFonts TypedDict, Face.identify_for_debug, CTFace.identify_for_debug |
| Options directory | `kitty/options/` | Directory listing for all config-related files |
| Fonts directory | `kitty/fonts/` | Directory listing for all Python font modules |
| Tests | `kitty_tests/fonts.py` (lines 1–60) | Test infrastructure, setup_for_testing |
| Build config | `setup.py` | Dependency versions: HarfBuzz ≥1.5, lcms2, Python ≥3.8 |
| Build config | `pyproject.toml` | Build system configuration |
| Build config | `go.mod` | Go 1.22 version, Go module dependencies |

### 0.8.2 Bash Commands Executed

- `find /tmp/blitzy -name ".blitzyignore" -type f` — Checked for ignore patterns (none found)
- `ls /tmp/environments_files/` — Checked for user-provided environment files (none found)
- `grep -rn "cell_metrics" kitty/fonts.c kitty/freetype.c kitty/fonts.h` — Located cell metric computation functions
- `grep -rn "initialize_font_group\|FONTS_DATA_HEAD" kitty/fonts.c` — Traced font group initialization
- `grep -rn "hb_buffer_guess_segment_properties\|hb_buffer_set_direction" kitty/fonts.c` — Located HarfBuzz BiDi direction logic
- `grep -rn "force_ltr" kitty/state.h kitty/fonts.c kitty/options/definition.py` — Traced force_ltr config propagation
- `grep -rn "identify_for_debug" kitty/freetype.c kitty/debug_config.py kitty/fast_data_types.pyi` — Traced debug identification output
- `grep -rn "create_fallback_face" kitty/fontconfig.c kitty/core_text.m kitty/fonts.h` — Traced platform-specific fallback resolution
- `grep -rn "alloc_sprite_map\|sprite_tracker_set_layout\|GL_MAX_TEXTURE" kitty/shaders.c kitty/fonts.c` — Traced GPU atlas initialization
- `grep -rn "debug_fonts\|debug_rendering" kitty/state.h` — Located debug output macros

### 0.8.3 Technical Specification Sections Retrieved

| Section | Content Used For |
|---------|------------------|
| 4.2 Application Startup Flow | Startup sequence ordering: CLI parsing → config loading → locale → GLFW → font init → OpenGL → rendering loop |
| 4.3 Terminal Input/Output Pipeline | Text shaping pipeline context: input → HarfBuzz shaping → glyph cache → GPU rendering |
| 5.2 Component Details | Architecture of font subsystem, rendering engine, configuration system |
| 7.2 GPU Rendering Pipeline | Sprite atlas structure, shader pipeline, OpenGL texture management |

### 0.8.4 Attachments and External Resources

- **No attachments**: The user provided zero attachments to this project
- **No Figma URLs**: No design files were referenced
- **No environment files**: No environment-specific configuration was provided
- **Source branch**: `kitty_815df1e210e0` — the branch name used for the deliverable document filename per project rules

