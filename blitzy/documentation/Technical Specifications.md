# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that traces and explains the initialization (startup) flow of the kitty terminal emulator, with a specific focus on GPU context creation, font system setup, rendering backend selection, display configuration detection, and text cell calculations that occur before any terminal content is rendered.

- **Category:** Create new documentation
- **Documentation Type:** Technical investigation / Architecture analysis document
- **Target Output:** A comprehensive markdown document named `kitty_815df1e210e0.md` in the `blitzy/documentation/` directory

The documentation requirements, restated with enhanced clarity:

- **Build and Runtime Tracing:** The user seeks to understand kitty's bootstrapping sequence by examining the source code — specifically what happens from process launch through to the first rendered frame. This is a code-based investigation, not a runtime instrumentation exercise.
- **GPU Context Creation:** Document how the OpenGL context is established, which GLFW backend is selected (Cocoa / X11 / Wayland), what OpenGL version and extensions are required, and how the SRGB framebuffer is configured.
- **Font System Setup:** Document how fonts are loaded and resolved, how FreeType/HarfBuzz compute cell metrics (cell_width, cell_height, baseline, underline_position), and how the font group is initialized before rendering.
- **Rendering Backend Selection:** Document how kitty determines which windowing backend to use (`cocoa`, `wayland`, or `x11`) and how the `is_wayland()` logic works including DPI detection and content scaling.
- **Text Cell Calculations:** Document how `calc_cell_width()` and `calc_cell_height()` work, how `font_units_to_pixels_y/x()` convert FreeType font units, and how sprite tracker layouts are derived from cell dimensions.
- **Subsystem Initialization Order:** Provide a clear ordered mapping of which subsystems initialize in what sequence and what inter-dependencies exist between them.
- **Read-Only Analysis:** The user has explicitly stated: "Do not make any changes to the repository and leave the actual codebase unchanged."

### 0.1.2 Special Instructions and Constraints

- **CRITICAL:** No modifications to the existing source repository are permitted. The user directive is explicit: "Do not make any changes to the repository and leave the actual codebase unchanged."
- **Implementation Rule — SWE-AtlasQnA-Repo:** Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the questions posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.
- **No runtime output required:** The user asks to "run it and tell me what you observe," but the actual deliverable is a code-derived analysis document, as the container environment lacks a display server, GPU, and the native C toolchain dependencies (FreeType, HarfBuzz, GLFW) needed for a real build.
- **Style:** Answers must cite specific source files, line numbers, and code constructs as evidence. Reasoning and rationale must accompany each conclusion.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the startup sequence**, we will create a new comprehensive markdown file (`blitzy/documentation/kitty_815df1e210e0.md`) that traces the call chain from `kitty/entry_points.py:main()` → `kitty/main.py:_main()` → GLFW init → font setup → `create_os_window()` → GL init → shader loading → Boss creation → main loop entry.
- To **explain GPU context creation**, we will analyze `kitty/glfw.c:create_os_window()` (lines 1107–1323), `kitty/gl.c:gl_init()` (lines 52–77), and the OpenGL version requirements in `kitty/data-types.h` (lines 20–26).
- To **explain font system setup**, we will analyze `kitty/fonts/render.py:set_font_family()`, `kitty/fonts.c:initialize_font_group()` (lines 1495–1517), `kitty/fonts.c:calc_cell_metrics()` (lines 373–422), and `kitty/freetype.c:cell_metrics()` (lines 387–405).
- To **explain rendering backend selection**, we will analyze `kitty/main.py:init_glfw()` (lines 95–98), `kitty/constants.py:is_wayland()` (lines 207–217), and `kitty/glfw.c:dpi_from_scale()` (lines 812–820).
- To **explain text cell calculations**, we will analyze `kitty/freetype.c:calc_cell_width()` (lines 374–383), `calc_cell_height()` (lines 141–152), and `font_units_to_pixels_y()` (lines 92–93).
- To **map initialization order**, we will produce a sequenced diagram and narrative tracing the full call chain with specific source citations.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation needs are inferred:

- **DPI and Content Scale Detection:** The `get_window_content_scale()` function in `kitty/glfw.c` (line 823) computes DPI from scale factors using a platform-specific base (72.0 on macOS, 96.0 on Linux). This is a critical bridge between the window system and font sizing that must be documented.
- **Sprite Tracker Layout Initialization:** The `sprite_tracker_set_layout()` function in `kitty/fonts.c` (line 276) computes the texture atlas grid dimensions from cell_width and cell_height against `max_texture_size`. This determines how many glyphs can be cached in GPU memory.
- **Temporary Window for DPI Probing:** On non-Wayland systems, `create_os_window()` creates a temporary invisible 640×480 window to probe the DPI before creating the actual window. This is a non-obvious but critical architectural detail.
- **SRGB Color Pipeline:** The first-window initialization enables `GL_FRAMEBUFFER_SRGB` and validates the framebuffer supports sRGB encoding (line 1248), which affects all color rendering.
- **Pre-rendered Sprites:** Before the first frame, kitty pre-renders underlines, strikethroughs, cursors, and missing-glyph placeholders into the glyph cache via `send_prerendered_sprites_for_window()`.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx/reStructuredText-based documentation system** with comprehensive user-facing documentation but no existing internal architecture analysis of the initialization flow.

- **Current documentation framework:** Sphinx with `furo` theme
- **Documentation generator configuration:** `docs/conf.py` (Sphinx configuration) and `docs/Makefile` (build driver)
- **API documentation tools in use:** No formal API doc generator (no JSDoc, Sphinx autodoc for C); inline code comments serve as the primary reference
- **Diagram tools detected:** Mermaid (used in the existing tech spec sections)
- **Documentation hosting:** Static HTML generated by Sphinx, published via `make docs` and `publish.py`

**Existing documentation structure:**
- `docs/` — 40+ `.rst` files covering user guides, protocol specs, configuration, FAQ
- `docs/build.rst` — Build-from-source instructions (uses `./dev.sh build`)
- `docs/conf.rst` — Configuration reference
- `docs/performance.rst` — Performance documentation
- `docs/kittens/` — Kitten-specific documentation
- `README.asciidoc` — Project introduction
- `CONTRIBUTING.md`, `INSTALL.md`, `SECURITY.md` — Contributor/policy docs

**Key finding:** There is no existing document that traces the initialization flow, maps subsystem boot order, or explains the GPU/font/window-system relationship during startup. The tech spec sections (4.2, 5.1, 7.2) provide high-level architecture context but do not answer the user's specific code-level questions about computed values and initialization timing.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for initialization code to document:

- **Entry points:** `kitty/entry_points.py` — Dispatches `sys.argv` to appropriate handler; default path calls `kitty.main.main()`
- **Main startup orchestration:** `kitty/main.py` — Contains `_main()` (lines 441–521), `init_glfw()` (lines 95–98), `AppRunner.__call__()` (lines 247–260), and `_run_app()` (lines 202–236)
- **GLFW initialization (C):** `kitty/glfw.c` — Contains `glfw_init()` (lines 1431–1468), `create_os_window()` (lines 1107–1323), `get_window_content_scale()` (lines 823–835), `dpi_from_scale()` (lines 812–820)
- **OpenGL initialization:** `kitty/gl.c` — Contains `gl_init()` (lines 52–77) with GLAD loading and version validation
- **Font system initialization:** `kitty/fonts.c` — Contains `initialize_font_group()` (lines 1495–1517), `calc_cell_metrics()` (lines 373–422), `font_group_for()` (lines 203–218), `load_fonts_data()` (lines 1529–1533)
- **FreeType cell calculations:** `kitty/freetype.c` — Contains `cell_metrics()` (lines 387–405), `calc_cell_width()` (lines 374–383), `calc_cell_height()` (lines 141–152), `set_size_for_face()` (lines 190–197), `font_units_to_pixels_y()` (lines 92–93)
- **Font rendering (Python):** `kitty/fonts/render.py` — Contains `set_font_family()` (lines 173–193), `prerender_function`, and special glyph rendering
- **Shader loading:** `kitty/shaders.py` — Contains `LoadShaderPrograms.__call__()` (lines 147–204), `Program` class for GLSL loading
- **Global state:** `kitty/state.h` — Contains `GlobalState` (lines 259–280) with `gl_version`, `default_dpi`, `is_wayland`, `debug_rendering`
- **OS window state:** `kitty/state.h` — Contains `OSWindow` struct (lines 216–256) with viewport dimensions, fonts_data, window chrome
- **Constants/config:** `kitty/data-types.h` — Contains `OPENGL_REQUIRED_VERSION_MAJOR` (3), `OPENGL_REQUIRED_VERSION_MINOR` (3 on macOS, 1 on Linux), `GLSL_VERSION` (140)
- **Window sizing:** `kitty/os_window_size.py` — Contains `initial_window_size_func()` which computes window dimensions from cell metrics, DPI, and scale
- **Boss controller:** `kitty/boss.py` — Central lifecycle manager instantiated after window creation
- **Debug config output:** `kitty/debug_config.py` — Contains `debug_config()` which outputs OpenGL version, compositor, fonts, and paths

### 0.2.3 Web Search Research Conducted

No web searches were required for this analysis. The source code and existing tech spec sections provided sufficient evidence to fully document the initialization flow. All conclusions are derived directly from source code examination.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation in the initialization analysis:**

- **Module: `kitty/entry_points.py`**
  - Public APIs: `main()`, `namespaced()`, `setup_openssl_environment()`
  - Current documentation: Minimal — no initialization flow documentation exists
  - Documentation needed: Entry point dispatch logic, how default GUI path is selected

- **Module: `kitty/main.py`**
  - Public APIs: `_main()`, `init_glfw()`, `init_glfw_module()`, `AppRunner.__call__()`, `_run_app()`, `load_all_shaders()`, `set_locale()`, `setup_environment()`
  - Current documentation: No architecture-level documentation
  - Documentation needed: Full ordered startup sequence with code citations, GLFW backend selection logic, font initialization trigger, shader loading, Boss creation

- **Module: `kitty/glfw.c`**
  - Key Functions: `glfw_init()` (line 1431), `create_os_window()` (line 1107), `get_window_content_scale()` (line 823), `dpi_from_scale()` (line 812), `update_os_window_viewport()` (line 130)
  - Current documentation: C inline comments only
  - Documentation needed: GPU context creation flow, DPI detection, temp window creation, SRGB configuration, callback registration, window state initialization

- **Module: `kitty/gl.c`**
  - Key Functions: `gl_init()` (line 52), `gl_version_string()` (line 42), `update_surface_size()` (line 80)
  - Current documentation: None beyond inline comments
  - Documentation needed: GLAD loading, OpenGL version validation, ARB extension checks, debug rendering output

- **Module: `kitty/fonts.c`**
  - Key Functions: `initialize_font_group()` (line 1495), `calc_cell_metrics()` (line 373), `load_fonts_data()` (line 1529), `font_group_for()` (line 203), `sprite_tracker_set_layout()` (line 276), `send_prerendered_sprites()` (line 1449)
  - Current documentation: None
  - Documentation needed: Font group creation, cell metric computation, sprite atlas sizing, pre-rendered sprite generation

- **Module: `kitty/freetype.c`**
  - Key Functions: `cell_metrics()` (line 387), `calc_cell_width()` (line 374), `calc_cell_height()` (line 141), `set_size_for_face()` (line 190), `font_units_to_pixels_y()` (line 92), `init_ft_face()` (line 212)
  - Current documentation: None
  - Documentation needed: How font units convert to pixels, how cell width is computed from ASCII glyph advances, baseline/underline/strikethrough positioning

- **Module: `kitty/fonts/render.py`**
  - Key Functions: `set_font_family()` (line 173), `create_symbol_map()` (line 139), `render_special()` (line 284)
  - Current documentation: None
  - Documentation needed: How font descriptors are resolved, face loading, symbol map assembly, prerender callback interface

- **Module: `kitty/shaders.py`**
  - Key Functions: `LoadShaderPrograms.__call__()` (line 147), `Program._load_sources()` (line 61)
  - Current documentation: None
  - Documentation needed: Shader compilation pipeline, macro substitution, multi-pass cell program variants

- **Module: `kitty/data-types.h`**
  - Key Constants: `OPENGL_REQUIRED_VERSION_MAJOR` (3), `OPENGL_REQUIRED_VERSION_MINOR` (3/1), `GLSL_VERSION` (140), `FONTS_DATA_HEAD`
  - Documentation needed: Platform-conditional OpenGL version requirements, GLSL version contract

- **Module: `kitty/state.h`**
  - Key Structures: `GlobalState` (line 259), `OSWindow` (line 216), `Options` (line 36), `FONTS_DATA_HEAD` macro (from `data-types.h` line 347)
  - Documentation needed: State layout during initialization, key fields populated during startup

- **Module: `kitty/os_window_size.py`**
  - Key Functions: `initial_window_size_func()` (line 54), `edge_spacing()` (line 40)
  - Documentation needed: How window dimensions are derived from cell metrics, DPI, and scale

- **Module: `kitty/constants.py`**
  - Key Functions: `is_wayland()` (line 207), `glfw_path()` (line 191), `detect_if_wayland_ok()` (line 196)
  - Documentation needed: Backend detection logic, GLFW module path resolution

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the documentation gaps include:

- **Undocumented initialization sequence:** No existing document traces the full boot path from process entry to main loop
- **Undocumented GPU initialization:** The relationship between GLFW backend selection, temp window creation, GL context activation, and GLAD loading is not documented anywhere
- **Undocumented font/cell metric pipeline:** The conversion chain from font_size_in_pts → DPI → FreeType char size → font units → pixel metrics → cell dimensions → window size is entirely undocumented
- **Undocumented DPI/scale logic:** The `dpi_from_scale()` platform-conditional base factor (72 on macOS, 96 on Linux) and its impact on font rendering is not explained
- **Undocumented sprite atlas initialization:** How the sprite tracker layout is computed from cell dimensions and GPU texture limits is not documented
- **Undocumented shader compilation:** The multi-variant cell program compilation (BOTH, BACKGROUND, SPECIAL, FOREGROUND) with macro injection is not explained outside the code itself


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/kitty_815df1e210e0.md` will follow this structure:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction & Rationale
        ├── Startup Sequence Overview (ordered phases)
        ├── Phase 1: Entry Point Dispatch
        ├── Phase 2: CLI Parsing & Environment Setup
        ├── Phase 3: GLFW Backend Selection & Initialization
        ├── Phase 4: Font System Setup & Cell Metric Computation
        ├── Phase 5: OS Window Creation & GPU Context
        ├── Phase 6: OpenGL Initialization & Shader Compilation
        ├── Phase 7: Pre-rendered Sprites & Window Finalization
        ├── Phase 8: Boss Controller & Main Loop Entry
        ├── Subsystem Relationship Map (Mermaid)
        ├── Key Values Computed During Startup (Table)
        ├── Rendering Backend Selection Logic
        ├── Display Configuration Detection
        ├── Text Rendering Capabilities
        └── Source File Reference Index
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract startup sequence by tracing the call graph starting from `kitty/entry_points.py:main()` through `kitty/main.py:_main()`, `init_glfw()`, `AppRunner.__call__()`, `_run_app()`, and `create_os_window()`
- Extract cell metric calculations from `kitty/freetype.c:cell_metrics()` and `kitty/fonts.c:calc_cell_metrics()` using code parsing
- Extract GPU initialization details from `kitty/gl.c:gl_init()` and `kitty/glfw.c:create_os_window()`
- Extract DPI computation from `kitty/glfw.c:dpi_from_scale()` and `get_window_content_scale()`
- Generate relationship diagrams by mapping the initialization call chain across Python and C layers
- Create value computation tables by identifying all key constants and computed values from source

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration using triple-backtick mermaid blocks for initialization sequence, subsystem relationships, and data flow
- Code references using the format: `Source: /path/to/file.py:LineNumber`
- Tables for computed values, key constants, and subsystem initialization order
- Consistent terminology matching kitty's own codebase naming conventions

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create in the output document:

- **Initialization Sequence Diagram** — A flowchart showing the ordered phases from process entry through main loop entry, with decision points for backend selection
- **Subsystem Relationship Diagram** — A graph showing how GLFW, OpenGL, FreeType, Font Groups, Sprite Tracker, Shaders, and the Boss controller interconnect
- **Font Metric Computation Flow** — A sequence diagram tracing font_size_in_pts through DPI conversion, FreeType sizing, cell metric extraction, to sprite layout
- **DPI/Scale Detection Flow** — A flowchart showing how platform-specific DPI is derived from content scale factors

These diagrams will be embedded directly in the output markdown document using `mermaid` code blocks.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/entry_points.py`, `kitty/main.py`, `kitty/glfw.c`, `kitty/gl.c`, `kitty/fonts.c`, `kitty/freetype.c`, `kitty/fonts/render.py`, `kitty/shaders.py`, `kitty/state.h`, `kitty/data-types.h`, `kitty/os_window_size.py`, `kitty/constants.py`, `kitty/boss.py`, `kitty/debug_config.py`, `kitty/child-monitor.c`, `kitty/shaders.c`, `kitty/fonts.h`, `kitty/gl.h`, `kitty/session.py`, `glfw/init.c` | Comprehensive technical analysis document answering all user questions about kitty's initialization flow, GPU context creation, font system setup, rendering backend selection, display configuration detection, text cell calculations, and subsystem initialization order |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Architecture Analysis
Source Code:
    - kitty/entry_points.py (entry dispatch)
    - kitty/main.py (main startup orchestration)
    - kitty/glfw.c (GLFW init, OS window creation, DPI)
    - kitty/gl.c (OpenGL initialization)
    - kitty/fonts.c (font group init, cell metrics, sprites)
    - kitty/freetype.c (FreeType cell calculations)
    - kitty/fonts/render.py (font family setup)
    - kitty/shaders.py (shader program compilation)
    - kitty/state.h (global state, OSWindow structs)
    - kitty/data-types.h (OpenGL/GLSL version constants)
    - kitty/os_window_size.py (window sizing from cells)
    - kitty/constants.py (Wayland detection, paths)
    - kitty/boss.py (Boss controller creation)
    - kitty/debug_config.py (debug output routines)
    - kitty/child-monitor.c (main loop)
    - kitty/shaders.c (shader C-level compilation)
    - kitty/session.py (window sizing data)
    - glfw/init.c (GLFW library initialization)
Sections:
    - Introduction answering the user's core questions
    - Ordered initialization sequence (8 phases)
    - GPU context creation deep-dive
    - Font system and cell metric computation
    - Rendering backend selection logic
    - Display configuration and DPI detection
    - Text rendering capabilities analysis
    - Subsystem relationship map with Mermaid diagrams
    - Key computed values reference table
    - Source file index with line citations
Diagrams:
    - Initialization sequence flowchart
    - Subsystem interconnection graph
    - Font metric computation pipeline
    - DPI/scale detection decision tree
Key Citations:
    - kitty/main.py:441-521 (_main startup sequence)
    - kitty/glfw.c:1107-1323 (create_os_window)
    - kitty/gl.c:52-77 (gl_init)
    - kitty/fonts.c:1495-1517 (initialize_font_group)
    - kitty/freetype.c:387-405 (cell_metrics)
    - kitty/data-types.h:20-26 (OpenGL/GLSL versions)
    - kitty/glfw.c:812-835 (DPI from scale)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files require updates. The output is a standalone markdown document placed in a new `blitzy/documentation/` directory. No changes to:
- `docs/conf.py` — Not affected (existing Sphinx config for user docs)
- `docs/Makefile` — Not affected
- No navigation, sidebar, or index updates needed

### 0.5.4 Cross-Documentation Dependencies

- The output document references tech spec sections 4.2 (Application Startup Flow), 5.1 (High-Level Architecture), and 7.2 (GPU Rendering Pipeline) for architectural context
- No shared content/includes required
- No existing documentation files need link updates
- The document is self-contained with all necessary code citations embedded inline


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tooling is required for this task. The deliverable is a standalone Markdown file that does not require any build tool, generator, or formatter.

The following runtime and build dependencies of kitty are **referenced in the analysis** (not installed for doc generation):

| Registry | Package Name | Version | Purpose in Documentation |
|----------|--------------|---------|--------------------------|
| System | Python | ≥ 3.8 | Runtime documented in `pyproject.toml` `requires-python = ">=3.8"` |
| System | OpenGL | ≥ 3.3 (macOS) / ≥ 3.1 (Linux) | Minimum version from `kitty/data-types.h:20-24` |
| System | GLSL | 140 | Shader version from `kitty/data-types.h:26` |
| Vendored | GLFW | 3.4 (fork) | Vendored in `glfw/` directory, referenced in `glfw/glfw3.h` |
| System | FreeType | (system) | Font rasterization, referenced in `kitty/freetype.c` |
| System | HarfBuzz | ≥ 1.5 | Text shaping, referenced in `kitty/fonts.h` |
| System | FontConfig | (system/Linux) | Font discovery on Linux, referenced in `kitty/fontconfig.c` |
| System | CoreText | (framework/macOS) | Font discovery on macOS, referenced in `kitty/core_text.m` |
| pip | sphinx | (latest) | Existing docs framework per `docs/requirements.txt` — not used for this task |
| pip | furo | (latest) | Sphinx theme per `docs/requirements.txt` — not used for this task |

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The new document is placed in a separate `blitzy/documentation/` directory and does not integrate with the existing `docs/` Sphinx tree. No links in existing documentation need modification.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **Current coverage analysis:**
  - Initialization flow documented: 0/8 phases (0%) — no existing initialization analysis exists
  - GPU context creation documented: 0% — not covered in user-facing docs
  - Font cell metric computation documented: 0% — not covered outside inline C comments
  - Backend selection logic documented: Partially covered in tech spec section 4.2, but not at code-trace level
  - Subsystem initialization order documented: Partially covered in tech spec section 4.2.3, but lacking computed-value detail

- **Target coverage:** 100% of all 8 initialization phases fully traced with source citations
- **Coverage gaps to address:**
  - `kitty/glfw.c:create_os_window()` — 0% documented at code level, target 100%
  - `kitty/gl.c:gl_init()` — 0% documented at code level, target 100%
  - `kitty/fonts.c:initialize_font_group()` / `calc_cell_metrics()` — 0% documented, target 100%
  - `kitty/freetype.c:cell_metrics()` / `calc_cell_width()` / `calc_cell_height()` — 0% documented, target 100%
  - `kitty/glfw.c:dpi_from_scale()` — 0% documented, target 100%

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every user question must be directly answered with code-backed evidence
  - All initialization phases must be covered with specific source file citations and line numbers
  - All key computed values (cell_width, cell_height, baseline, DPI, sprite layout) must be explained with their derivation formulas

- **Accuracy validation:**
  - All claims must reference specific source files and line numbers
  - No assumptions — only conclusions supported by code as ground truth
  - Platform-conditional behavior (macOS vs Linux) must be clearly distinguished

- **Clarity standards:**
  - Technical accuracy with accessible language
  - Progressive disclosure: overview first, then deep-dive per subsystem
  - Consistent terminology matching kitty's own codebase naming conventions
  - Mermaid diagrams for complex relationships and sequences

- **Maintainability:**
  - Source citations in format `Source: filepath:line` for traceability
  - Clear section structure for easy navigation
  - Self-contained document requiring no external dependencies to read

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 (initialization sequence, subsystem relationships, font metric pipeline, DPI detection flow)
- **Diagram types:** Mermaid flowcharts and sequence diagrams
- **Code example approach:** Short inline code snippets (2-3 lines max) from source files to illustrate key logic
- **Value computation tables:** At least one comprehensive table mapping all key values computed during startup with their source locations and derivation logic


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation files:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable document

- **Source code analysis targets (read-only — no modifications):**
  - `kitty/entry_points.py` — Entry point dispatch analysis
  - `kitty/main.py` — Full startup orchestration trace
  - `kitty/glfw.c` — GLFW init, `create_os_window()`, DPI/scale detection, callback registration
  - `kitty/gl.c` — `gl_init()`, GLAD loading, OpenGL version checks
  - `kitty/fonts.c` — Font group initialization, cell metrics, sprite tracker, prerendered sprites
  - `kitty/freetype.c` — Cell width/height calculation, font unit to pixel conversion, FreeType face initialization
  - `kitty/fonts/render.py` — `set_font_family()`, symbol map creation, prerender callback
  - `kitty/fonts/common.py` — `get_font_files()` font resolution
  - `kitty/shaders.py` — GLSL source loading, macro substitution, multi-variant compilation
  - `kitty/shaders.c` — C-level shader program management
  - `kitty/state.h` — `GlobalState`, `OSWindow`, `Options` structure definitions
  - `kitty/data-types.h` — OpenGL/GLSL version constants, `FONTS_DATA_HEAD` macro
  - `kitty/os_window_size.py` — Window dimension calculation from cell metrics
  - `kitty/constants.py` — `is_wayland()`, `glfw_path()`, `detect_if_wayland_ok()`
  - `kitty/boss.py` — Boss controller creation
  - `kitty/debug_config.py` — Debug output: OpenGL version, compositor, fonts
  - `kitty/child-monitor.c` — Main loop entry, render loop
  - `kitty/session.py` — `get_os_window_sizing_data()`
  - `kitty/fonts.h` — Font API contracts, `cell_metrics` declaration
  - `kitty/gl.h` — GL function declarations
  - `kitty/gl-wrapper.h` — GLAD wrapper declarations
  - `glfw/init.c` — GLFW library-level initialization
  - `glfw/context.c` — Context creation backend
  - `kitty/borders.py` — `load_borders_program()` shader
  - `kitty/config.py` — Configuration loading used during startup

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be modified, added, or deleted (per explicit user instruction)
- **Runtime execution:** Building and actually running kitty is out of scope — the environment lacks the required display server, GPU, and native library dependencies
- **Test file modifications:** No test files will be modified
- **Feature additions or code refactoring:** Explicitly excluded
- **Unrelated documentation:** Documentation not pertaining to the initialization flow is excluded
- **Existing docs modifications:** No changes to files in `docs/`, `README.asciidoc`, `CONTRIBUTING.md`, `INSTALL.md`, etc.
- **Post-initialization runtime behavior:** Event loop internals, key handling, VT parsing, graphics protocol, and remote control — these are beyond the "early startup phase" scope
- **Kittens framework:** Not part of the initialization flow
- **Shell integration:** Not part of the early startup phase
- **Go tools (`tools/`):** Not involved in the kitty GUI initialization path


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file placed at `blitzy/documentation/kitty_815df1e210e0.md` and does not require a build step
- **Documentation preview command:** Any Markdown renderer (e.g., VS Code preview, GitHub web UI, or a local Markdown-to-HTML tool)
- **Diagram generation command:** Diagrams are authored as Mermaid fenced code blocks inside the Markdown file; they render natively in GitHub, GitLab, and any Mermaid-aware viewer — no separate generation step is required
- **Documentation deployment command:** Not applicable — no documentation hosting infrastructure is involved
- **Default format:** Markdown (.md) with embedded Mermaid diagrams in fenced code blocks
- **Citation requirement:** Every technical claim must reference the source file and, where meaningful, the line number or function name (e.g., Source: kitty/glfw.c — create_os_window())
- **Style guide to follow:** The implementation rule SWE-AtlasQnA-Repo mandates:
  - Document name: kitty_815df1e210e0.md
  - Location: blitzy/documentation/ directory
  - Content: comprehensively answers all questions posed in the prompt
  - Includes thinking and rationale behind the answers
  - Base all answers on code as the truth — no assumptions
  - Do not modify any existing files in the source repository
- **Documentation validation:**
  - Confirm every Mermaid diagram parses correctly (no nested triple-backtick violations)
  - Confirm all referenced file paths exist in the repository
  - Confirm the document answers every question the user posed


## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

- **Read-only constraint (explicit):** "Do not make any changes to the repository and leave the actual codebase unchanged." — No existing file in the repository may be modified, added, or deleted under any circumstances.
- **Code-as-truth rule (implementation rule SWE-AtlasQnA-Repo):** "Do not make assumptions, base your answers on the code as the truth." — Every claim in the output document must be traceable to a specific source file, function, or data structure in the kitty repository.
- **Rationale inclusion (implementation rule SWE-AtlasQnA-Repo):** "Provide thinking / rationale behind the answers." — The document must explain the reasoning and evidence chain behind each architectural conclusion, not merely list facts.
- **Comprehensive answering (implementation rule SWE-AtlasQnA-Repo):** "Create a new markdown document that comprehensively answers the question(s) posed in the prompt." — Every question the user asked must receive a thorough, evidence-backed answer.

### 0.10.2 Derived Documentation Rules

- **Source citations required:** Every technical statement about initialization order, value computation, or subsystem interaction must cite the originating source file and the relevant function or line
- **Mermaid diagrams for all workflows:** The startup sequence, subsystem initialization order, and component connections must be illustrated with Mermaid diagrams (sequence, flowchart, or graph as appropriate)
- **Platform-conditional documentation:** Where behavior diverges between macOS (Cocoa), Linux/X11, and Linux/Wayland, all three paths must be documented explicitly with the conditional logic that selects each path
- **Numerical precision:** Key computed values (DPI factors, OpenGL version requirements, GLSL version, cell dimension formulas) must include the exact formulas found in source, not approximations
- **No speculative content:** If the code does not definitively answer a question (e.g., "what does it actually select at runtime"), the document must explain what the code would select given specific conditions rather than fabricating runtime output
- **Document placement:** The output must be placed at exactly blitzy/documentation/kitty_815df1e210e0.md — no other location is acceptable


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were retrieved and analyzed to derive all conclusions in this Agent Action Plan:

| File Path | Purpose in Analysis |
|---|---|
| kitty/entry_points.py | Top-level entry dispatch — identifies how kitty main is invoked |
| kitty/main.py | Full startup orchestrator — _main(), init_glfw(), AppRunner, _run_app(), load_all_shaders() |
| kitty/glfw.c | GLFW initialization (glfw_init), OS window creation (create_os_window), DPI detection, GL context setup, callback registration |
| kitty/gl.c | OpenGL initialization (gl_init) — GLAD loading, version validation, extension checks |
| kitty/gl.h | GL API declarations, Program/Uniform/VAO structures |
| kitty/fonts.c | Font group initialization, cell metrics computation, sprite tracker layout, prerendered sprites, load_fonts_data |
| kitty/fonts.h | Font backend API contracts — cell_metrics, set_size_for_face, render_glyphs_in_cells |
| kitty/freetype.c | FreeType cell metric calculations — calc_cell_width, calc_cell_height, font_units_to_pixels_y, set_size_for_face |
| kitty/fonts/render.py | Python font orchestration — set_font_family, prerender_function, symbol maps |
| kitty/shaders.py | Shader program loading — GLSL compilation, macro substitution, multi-variant cell/graphics programs |
| kitty/shaders.c | C-level shader program enum and compilation entry points |
| kitty/state.h | GlobalState and OSWindow structure definitions — viewport, fonts_data, gl_version, is_wayland |
| kitty/data-types.h | OPENGL_REQUIRED_VERSION constants, GLSL_VERSION, FONTS_DATA_HEAD macro |
| kitty/os_window_size.py | Window dimension computation from cell metrics, DPI, and user-configured cell count |
| kitty/constants.py | is_wayland(), glfw_path(), detect_if_wayland_ok(), platform detection logic |
| kitty/boss.py | Boss controller class structure and imports |
| kitty/session.py | get_os_window_sizing_data() for session-based window sizing |
| kitty/debug_config.py | Debug output functions — OpenGL version, compositor, fonts, environment |
| kitty/child-monitor.c | Main loop (main_loop), render loop (render_os_window), prepare_to_render_os_window |
| glfw/init.c | GLFW library-level initialization, global state, init hints |
| kitty/borders.py | load_borders_program() shader loading (referenced by load_all_shaders) |

The following folders were explored to discover relevant files:

| Folder Path | Purpose in Analysis |
|---|---|
| (root) | Repository structure discovery — identified kitty/, glfw/, docs/, tools/, gen/ |
| kitty/ | Core application source — C extensions, Python modules, shaders |
| kitty/fonts/ | Font subsystem Python package — render.py, common.py |
| glfw/ | GLFW integration layer — platform backends (Wayland, X11, Cocoa) |
| docs/ | Existing documentation infrastructure discovery |

### 0.11.2 Tech Spec Sections Referenced

| Section Heading | Information Extracted |
|---|---|
| 4.2 APPLICATION STARTUP FLOW | Bootstrap sequence, single-instance detection, Python bootstrap, Boss initialization lifecycle |
| 5.1 HIGH-LEVEL ARCHITECTURE | Six architectural layers, core components, data flow, external integrations |
| 7.2 GPU Rendering Pipeline | 6-stage shader pipeline, 12 GLSL shader files, font rendering chain (FreeType → HarfBuzz → Glyph Cache → Texture Atlas → Cell Shader), threaded rendering |

### 0.11.3 Attachments and External Resources

- **Attachments provided:** None (0 attachments)
- **Figma URLs provided:** None
- **Environment files provided:** None (no files in /tmp/environments_files/)
- **User-provided setup instructions:** None

### 0.11.4 Implementation Rules Applied

| Rule Name | Key Directive |
|---|---|
| SWE-AtlasQnA-Repo | Create blitzy/documentation/kitty_815df1e210e0.md answering all questions; include rationale; base answers on code; do not modify existing files |


