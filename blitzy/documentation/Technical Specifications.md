# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to produce a comprehensive investigative document that answers a set of architectural questions about the kitty terminal emulator—not by static analysis alone, but by actively exercising the code and observing its runtime behavior. The repository must remain completely unmodified; only a single new Markdown document is created and placed in `blitzy/documentation/`.

The specific questions the document must answer are:

- **Language role at runtime**: kitty is marketed as "GPU accelerated," yet the codebase is a tight weave of Python (~62,874 lines), C (~61,806 lines of `.c`, ~37,939 lines of `.h`), and Go (~56,071 lines). Which language is really doing the heavy lifting once the terminal is running, and what does that say about where performance actually comes from?
- **GLSL shader purpose**: Thirteen `.glsl` shader files reside inside `kitty/`. Shader code inside a terminal emulator is unexpected—what role do they play, and how central are they to the system?
- **Main entry point failure**: Running `python3 __main__.py` directly fails almost immediately with a cryptic error. What exactly is missing at that moment, and what does the failure reveal about how Python is wired into the native C core?
- **The critical bridge piece**: There appears to be one critical piece that everything depends on—the `fast_data_types` C extension module—and the breakage makes it hard to ignore.
- **Kitten independence**: The `kittens/` directory contains what look like small, self-contained tools. But are they truly independent, or do they silently rely on the same native bridge to function? What actually happens if one is run on its own, and what does that reveal about how modular the system really is?

Implicit requirements detected:

- The analysis must be grounded in **empirical observation** (running the code, tracing import failures, testing kitten isolation) rather than purely reading source.
- Temporary scripts may be used for observation but must be cleaned up; the repository itself must remain unchanged.
- The output must articulate not just *what* happens but *why*, connecting observed behavior back to architectural design decisions.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability**: No existing files in the source repository may be modified. No new code may be added to the source repository beyond the requested Markdown document.
- **Document placement**: The output document must be named `kitty_815df1e210e0.md` (matching the source branch name) and placed in `blitzy/documentation/`.
- **Empirical method**: The user emphasizes observing "how the code behaves when exercised" rather than making assumptions about architecture. This mandates actually running Python import chains, attempting entry point execution, and testing kitten isolation.
- **Cleanup**: Any temporary scripts used for observation must be removed afterward.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **determine which language does the heavy lifting**, we will count lines of code by language, trace the runtime call chain from the native C launcher through CPython into the `fast_data_types` C extension, and map which subsystems (VT parsing, screen management, rendering, I/O) are implemented in C versus Python.
- To **explain the GLSL shader role**, we will catalog all 13 shader files, trace how Python's `shaders.py` loads them via `read_kitty_resource()`, how C's `shaders.c` compiles them into GPU programs via `glCreateProgram()` / `glCompileShader()`, and demonstrate that they form the sole rendering pipeline—there is no CPU fallback path.
- To **diagnose the main entry point failure**, we will execute `python3 __main__.py` and trace the exact import chain: `__main__.py` → `kitty.entry_points.main()` → `kitty.main` → `kitty.borders` → `kitty.fast_data_types` → `ModuleNotFoundError`. We will document what `fast_data_types` is (a compiled C extension produced from 40+ C source files) and why it cannot exist without a build step.
- To **test kitten independence**, we will attempt to import individual kitten modules and document which ones fail due to transitive `fast_data_types` dependencies, proving that even kittens that appear pure-Python are bound to the native bridge.
- To **produce the document**, we will create `blitzy/documentation/kitty_815df1e210e0.md` with structured answers, rationale, and supporting evidence from the experiments.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The kitty repository at commit `815df1e21` is a multi-language terminal emulator with the following high-level structure. Every file and folder listed below has been inspected during context gathering.

**Top-level directory structure:**

| Path | Type | Purpose |
|------|------|---------|
| `__main__.py` | File | Python entry point — delegates to `kitty.entry_points.main()` |
| `setup.py` | File | 2,173-line build orchestrator for C extensions, GLFW, Go binaries |
| `pyproject.toml` | File | Python project metadata; declares `requires-python = ">=3.8"` |
| `go.mod` / `go.sum` | Files | Go module definition (Go 1.22) with 15 direct dependencies |
| `Makefile` | File | Convenience wrapper around `setup.py` |
| `kitty/` | Dir | Core terminal engine — 102 Python, 51 C, 13 GLSL, 84 header files |
| `kittens/` | Dir | 20 kitten sub-packages (Python stubs + Go implementations) |
| `tools/` | Dir | 13 Go packages providing CLI tooling compiled into `kitten` binary |
| `glfw/` | Dir | Vendored GLFW 3.4 fork — ~65 files for cross-platform windowing |
| `kitty_tests/` | Dir | Python test suite (27 test modules) |
| `docs/` | Dir | Sphinx documentation source |
| `shell-integration/` | Dir | Bash/Zsh/Fish/SSH shell integration scripts |
| `gen/` | Dir | Build-time code generators for configs, Unicode tables, key constants |
| `3rdparty/` | Dir | Vendored ringbuf and base64 C libraries |
| `logo/` | Dir | Branding assets |
| `terminfo/` | Dir | Terminfo database for kitty terminal type |

**Core engine files in `kitty/` (categorized by subsystem):**

| Subsystem | C Files | Python Files | GLSL Files |
|-----------|---------|--------------|------------|
| VT Parsing | `vt-parser.c`, `vt-parser.h` | — | — |
| Screen Model | `screen.c`, `line.c`, `line-buf.c`, `cursor.c`, `history.c` | — | — |
| GPU Rendering | `gl.c`, `gl-wrapper.c`, `shaders.c` | `shaders.py` | `cell_vertex.glsl`, `cell_fragment.glsl`, `cell_defines.glsl`, `border_vertex.glsl`, `border_fragment.glsl`, `bgimage_vertex.glsl`, `bgimage_fragment.glsl`, `graphics_vertex.glsl`, `graphics_fragment.glsl`, `tint_vertex.glsl`, `tint_fragment.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl` |
| Font Pipeline | `fonts.c`, `freetype.c`, `fontconfig.c`, `glyph-cache.c`, `font-names.c` | `fonts/render.py`, `fonts/common.py`, `fonts/fontconfig.py`, `fonts/box_drawing.py` | — |
| Child Process Management | `child-monitor.c`, `child.c` | `child.py` | — |
| Input Handling | `keys.c`, `key_encoding.c`, `mouse.c` | `keys.py`, `key_encoding.py` | — |
| Graphics Protocol | `graphics.c`, `png-reader.c` | — | — |
| C Extension Bridge | `data-types.c` | `fast_data_types.pyi` (type stubs) | — |
| Application Logic | — | `boss.py` (3,094 lines), `main.py`, `entry_points.py`, `config.py`, `session.py`, `tabs.py`, `window.py`, `borders.py`, `clipboard.py` | — |
| Configuration | — | `options/definition.py`, `options/parse.py`, `options/types.py`, `options/utils.py`, `conf/utils.py`, `conf/types.py` | — |
| Remote Control | — | `rc/` (41 modules), `remote_control.py`, `client.py` | — |
| Layout Engine | — | `layout/base.py`, `layout/grid.py`, `layout/splits.py`, `layout/tall.py`, `layout/vertical.py`, `layout/stack.py` | — |
| Native Launcher | `launcher/main.c`, `launcher/single-instance.c` | — | — |

**Kittens directory — Python stubs and Go implementations:**

| Kitten | Python Stub | Go Files | Direct `fast_data_types` Imports | Transitive Dependency |
|--------|-------------|----------|----------------------------------|----------------------|
| `ask` | `main.py` | `choices.go`, `get_line.go`, `main.go` | 0 | Yes (via `kitty.typing` → `tui.handler`) |
| `broadcast` | `main.py` | — | 0 | Yes (7 `kitty.*` imports) |
| `choose_fonts` | `backend.py`, `main.py` | 10 Go files | 0 | Yes (10 `kitty.*` imports) |
| `clipboard` | `main.py` | `main.go`, `read.go`, `write.go` | 0 | Yes (via SystemExit stub) |
| `diff` | `main.py` | 8 Go files | 0 | Yes (via `kitty.cli`) |
| `hints` | `main.py` | `main.go`, `marks.go` | 1 | Yes |
| `hyperlinked_grep` | `main.py` | `main.go` | 0 | No (pure stub) |
| `icat` | `main.py` | 5 Go files | 0 | Yes (via `kitty.utils`) |
| `panel` | `main.py` | — | 1 | Yes |
| `query_terminal` | `main.py` | `main.go` | 10 | Yes |
| `show_key` | `main.py` | 3 Go files | 0 | No (pure stub) |
| `ssh` | `main.py` | 4 Go files | 0 | Yes (7 `kitty.*` imports) |
| `themes` | `main.py` | 3 Go files | 0 | Yes (via `kitty.cli`) |
| `transfer` | `main.py` | 6 Go files | 0 | Yes |
| `unicode_input` | `main.py` | 2 Go files | 0 | Yes |
| `tui/` | `handler.py`, `loop.py`, `images.py`, `operations.py`, others | — | 6 direct imports | Yes (foundation layer) |

### 0.2.2 Integration Point Discovery

The critical integration points for understanding the architecture are:

- **`kitty/data-types.c` → `PyInit_fast_data_types()`**: The single C extension module that bridges Python and C. It initializes 25+ subsystem modules (Screen, LineBuf, HistoryBuf, Cursor, Parser, ColorProfile, GLFW, fonts, shaders, graphics, keys, mouse, kittens, etc.) within a single `.so` shared object.
- **`kitty/launcher/main.c`**: The native binary entry point that embeds CPython, sets `sys.kitty_run_data`, and hands off to Python's `__main__.py`.
- **`kitty/shaders.py` ↔ `kitty/shaders.c`**: Python loads and preprocesses GLSL source; C compiles shader programs and manages OpenGL state.
- **`kitty/child-monitor.c` → `main_loop()`**: The C-implemented main event loop that runs on the main thread, with I/O and talk threads for PTY multiplexing and remote control.
- **`kittens/runner.py`**: The Python framework that dispatches kitten execution, importing kitten modules and managing their lifecycle.
- **`kitty/boss.py`**: The 3,094-line Python singleton that orchestrates windows, tabs, child processes, configuration, and user actions.

### 0.2.3 New File Requirements

A single new file will be created:

| File | Purpose |
|------|---------|
| `blitzy/documentation/kitty_815df1e210e0.md` | Comprehensive Markdown document answering the user's architectural questions with evidence from code execution and source analysis |

No source files in the existing repository will be modified.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

Since this task produces a documentation artifact rather than compiled software, no new packages are added. However, the following dependencies are relevant to understanding the architectural analysis:

**Python runtime and project configuration (from `pyproject.toml` and `setup.py`):**

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| System | Python | ≥ 3.8 (CI tests 3.8–3.11) | High-level application logic, kittens framework, build system |
| System | GCC / Clang | C11 (auto-detected) | Compiles `fast_data_types` C extension from 40+ C source files |
| System | pkg-config | — | Discovers system library paths for FreeType, HarfBuzz, FontConfig, etc. |

**Go module dependencies (from `go.mod`):**

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| Go modules | `go` | 1.22 | Compiles `kitten` static binary from `tools/` and `kittens/` Go sources |
| Go modules | `github.com/alecthomas/chroma/v2` | v2.14.0 | Syntax highlighting in diff kitten |
| Go modules | `github.com/zeebo/xxh3` | v1.0.2 | Fast hashing for file transfer |
| Go modules | `golang.org/x/sys` | v0.21.0 | Unix system calls |
| Go modules | `golang.org/x/image` | v0.17.0 | Image processing in icat kitten |

**Native library dependencies (discovered from `setup.py` `pkg_config()` calls and build flags):**

| Library | Purpose |
|---------|---------|
| FreeType | Glyph rasterization (`kitty/freetype.c`) |
| HarfBuzz (≥ 1.5) | OpenType text shaping and ligatures |
| FontConfig (Linux) | Font discovery (`kitty/fontconfig.c`) |
| libpng | PNG image decoding (`kitty/png-reader.c`) |
| zlib | Compression in graphics protocol (`kitty/graphics.c`) |
| lcms2 | ICC color profile management |
| OpenGL (3.3+) | GPU rendering via GLAD-loaded API (`kitty/gl.c`) |
| libxxhash | Hash computations for transfer kitten |
| OpenSSL/libcrypto | X25519 + AES-GCM encryption for remote control |

### 0.3.2 Dependency Updates

No dependency updates are required. The task is documentation-only; the existing dependency graph is analyzed and documented but not modified. The critical observation is that all dependencies converge through `setup.py`'s build process into a single artifact—the `fast_data_types.so` shared object—which is the mandatory prerequisite for any Python-level functionality.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

No existing code will be modified. The integration analysis below documents the architectural touchpoints that the investigation document must explain, based on empirical observations from running the code.

**The `fast_data_types` Bridge — The Single Critical Piece:**

The entire kitty architecture hinges on one compiled C extension module: `kitty.fast_data_types`. This module is built by `setup.py` from 40+ C source files (every `.c` file in `kitty/` except platform-specific exclusions) and produces a single shared object (`fast_data_types.cpython-*.so`). At module initialization time, `PyInit_fast_data_types()` in `kitty/data-types.c` (line 525) registers 25+ subsystem initializers:

- `init_Screen`, `init_LineBuf`, `init_HistoryBuf`, `init_Line`, `init_Cursor` — terminal state
- `init_glfw`, `init_shaders`, `init_graphics` — GPU rendering
- `init_fonts`, `init_freetype_library`, `init_fontconfig_library` — font pipeline
- `init_child_monitor`, `init_child` — process management
- `init_keys`, `init_mouse` — input handling
- `init_state`, `init_kittens`, `init_crypto_library` — global state and utilities

**Import dependency penetration:**

- 87 Python files in `kitty/` directly import from `fast_data_types`
- 24 Python files in `kittens/` directly import from `fast_data_types`
- Only 13 out of 43 Python modules in `kitty/` can be imported without it: `choose_entry`, `cli_stub`, `client`, `constants`, `entry_points`, `guess_mime_type`, `key_names`, `multiprocessing`, `search_query_parser`, `short_uuid`, `types`, `typing`, `window_list`
- The remaining 30 modules fail immediately with `ModuleNotFoundError: No module named 'kitty.fast_data_types'`

**Entry point call chain observed:**

```
__main__.py
  └─ kitty.entry_points.main()
       └─ kitty.main.main()           [kitty_main]
            └─ from .borders import    [line 11]
                 └─ from .fast_data_types import BORDERS_PROGRAM...
                      └─ ModuleNotFoundError ← FAILURE POINT
```

**Shader integration chain:**

```
kitty/shaders.py (Python)
  ├─ read_kitty_resource('cell_vertex.glsl')  → loads GLSL as bytes
  ├─ Preprocesses: injects #version, resolves #pragma kitty_include_shader
  └─ Calls fast_data_types.compile_program()  → C function in shaders.c
       ├─ glCreateProgram()
       ├─ glCompileShader() for vertex + fragment
       ├─ glLinkProgram()
       └─ init_uniforms()
```

**Kitten dependency chain observed:**

```
kittens/ask/main.py
  └─ from kitty.typing import BossType, TypedDict  [succeeds — stubs only]
  └─ from ..tui.handler import result_handler
       └─ from kitty.fast_data_types import monotonic
            └─ ModuleNotFoundError ← FAILURE POINT

kittens/diff/main.py
  └─ from kitty.cli import CONFIG_HELP
       └─ from .conf.utils import resolve_config
            └─ from ..fast_data_types import Color
                 └─ ModuleNotFoundError ← FAILURE POINT
```

### 0.4.2 Runtime Architecture Observations

**Three-thread C event loop**: The `child-monitor.c` `main_loop()` function (line 1259) drives the main thread, calling `run_main_loop()` which manages render scheduling, input processing, and state checks. A separate I/O thread (`io_loop`) polls PTY file descriptors, and a Talk thread handles remote control sockets. All three threads are created in C.

**Python as orchestrator, C as executor**: The `boss.py` (3,094 lines) orchestrates high-level application logic—window creation, tab management, configuration—but delegates all performance-sensitive work to C through `fast_data_types`. For example, `boss.child_monitor.main_loop()` transfers control to C's main event loop (line 234 of `main.py`).

**Go as independent CLI binary**: The Go code compiles into a standalone `kitten` binary. Kittens with Go implementations (diff, ssh, icat, clipboard, etc.) have Python stubs that simply raise `SystemExit('Must be run as kitten ...')`. The real execution happens in the Go binary, which communicates with the running kitty instance via terminal escape sequences or remote control sockets.

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

A single file will be created. No files will be modified.

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — Comprehensive investigation document answering the user's five architectural questions, structured with the following sections:
  - Language roles at runtime (Python vs C vs Go)
  - GLSL shader role and centrality
  - Main entry point failure diagnosis
  - The `fast_data_types` critical bridge
  - Kitten independence analysis

### 0.5.2 Implementation Approach

The document will be produced by synthesizing the results of the following experiments and analyses, all of which have been conducted:

**Experiment 1 — Language line-count census:**
- Counted lines across all `.c`, `.h`, `.py`, `.go`, and `.glsl` files
- Result: C+H = ~99,745 lines, Python = ~62,874 lines, Go = ~56,071 lines, GLSL = ~696 lines
- Conclusion: C is the largest codebase component and implements all hot-path subsystems

**Experiment 2 — Entry point execution:**
- Ran `python3 __main__.py` and captured the traceback
- Traced the import chain: `__main__` → `entry_points.main()` → `kitty.main` → `kitty.borders` → `kitty.fast_data_types` → `ModuleNotFoundError`
- Conclusion: The very first real import from `kitty.main` cascades into `fast_data_types`, proving it is the mandatory foundation

**Experiment 3 — Module isolation test:**
- Attempted to import all 43 Python modules in `kitty/` individually
- Result: Only 13 modules can be imported without `fast_data_types`; 30 fail immediately
- Conclusion: 70% of kitty's Python layer is inoperable without the compiled C bridge

**Experiment 4 — Kitten isolation tests:**
- Attempted to import kittens: `ask` (failed via `tui.handler` → `fast_data_types.monotonic`), `diff` (failed via `kitty.cli` → `conf.utils` → `fast_data_types.Color`), `themes` (failed similarly), `show_key` (succeeded — pure stub), `hyperlinked_grep` (import succeeded — pure stub with no `main` function)
- Scanned all 20 kittens for `fast_data_types` imports: 4 direct importers, but nearly all are caught by transitive dependencies through `kitty.cli`, `kitty.conf.utils`, or `kittens/tui/`
- Conclusion: Kittens are not independently runnable; they are structurally bound to the native bridge

**Experiment 5 — GLSL shader analysis:**
- Cataloged all 13 `.glsl` files and their roles
- Traced the loading pipeline: `shaders.py` reads GLSL as package resources → preprocesses with `#pragma kitty_include_shader` directives → passes to `shaders.c` `compile_program()` → OpenGL compilation
- Documented the six rendering stages: cell, border, background image, inline graphics, tint, utility
- Confirmed no CPU-based rendering fallback exists: shaders are the sole rendering path

**Experiment 6 — Build system analysis:**
- Examined `setup.py` `build()` function: compiles all C files into `kitty/fast_data_types`, then compiles GLFW, then compiles kitten C extensions, then builds Go binary
- Attempted build: failed due to missing C compiler in container environment, further proving the C compilation dependency
- Confirmed `data-types.c` `PyInit_fast_data_types()` initializes 25+ subsystem modules

### 0.5.3 Document Structure

The output document `kitty_815df1e210e0.md` will be organized as follows:

- **Introduction**: Context-setting overview of kitty's three-language architecture
- **Question 1 — Which language does the heavy lifting?**: Line-count analysis, subsystem mapping, runtime control flow tracing showing C handles VT parsing, screen management, rendering, font rasterization, and the main event loop; Python orchestrates configuration, layout, and application logic; Go provides the standalone `kitten` CLI binary
- **Question 2 — What role do GLSL shaders play?**: Catalog of all 13 shaders, the Python→C→GPU compilation pipeline, explanation of six rendering stages, confirmation that shaders are the sole rendering path with no CPU fallback
- **Question 3 — Why does the entry point fail?**: Exact traceback, import chain analysis, explanation of `fast_data_types` as a compiled C extension, what it contains, and why it cannot exist without building
- **Question 4 — The critical bridge piece**: Deep dive into `data-types.c`, the 25+ subsystem initializers, the 87-file import dependency penetration, and the 13-vs-30 module isolation test
- **Question 5 — Are kittens independent?**: Import test results, transitive dependency analysis, Go binary vs Python stub architecture, demonstration that kittens are structurally coupled to the native core
- **Conclusion**: Synthesis connecting all findings into a coherent architectural narrative

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Output artifact:**
- `blitzy/documentation/kitty_815df1e210e0.md` — the sole deliverable

**Source files analyzed for the investigation (read-only):**

- Core architecture files:
  - `__main__.py` — entry point
  - `kitty/entry_points.py` — CLI dispatch and `main()` function
  - `kitty/main.py` — application startup, GLFW init, shader loading, boss creation
  - `kitty/boss.py` — central application orchestrator (3,094 lines)
  - `kitty/constants.py` — version, paths, `kitty_exe()`, `kitten_exe()`, `read_kitty_resource()`
  - `kitty/data-types.c` — `PyInit_fast_data_types()` and all subsystem registration
  - `kitty/fast_data_types.pyi` — Python type stubs for the C extension
- GPU rendering pipeline:
  - `kitty/shaders.py` — Python shader loading and preprocessing
  - `kitty/shaders.c` — C shader compilation and OpenGL program management
  - `kitty/gl.c` — OpenGL initialization and error handling
  - `kitty/*.glsl` — all 13 GLSL shader files
- Child process and event loop:
  - `kitty/child-monitor.c` — main loop, I/O thread, three-thread architecture
  - `kitty/child.py` — Python-side child process management
- Font pipeline:
  - `kitty/fonts.c`, `kitty/freetype.c`, `kitty/fontconfig.c`, `kitty/glyph-cache.c`
- VT parser:
  - `kitty/vt-parser.c` — escape sequence parsing state machine
  - `kitty/screen.c` — terminal screen model (4,932 lines)
- Kittens framework:
  - `kittens/runner.py` — kitten dispatch and lifecycle
  - `kittens/*/main.py` — individual kitten Python stubs
  - `kittens/tui/loop.py`, `kittens/tui/handler.py` — TUI foundation layer
- Build system:
  - `setup.py` — 2,173-line build orchestrator
  - `pyproject.toml` — Python project metadata
  - `go.mod` — Go module definition
  - `Makefile` — build convenience wrapper
- Platform layer:
  - `kitty/launcher/main.c` — native C launcher embedding CPython
  - `glfw/` — vendored GLFW 3.4 fork

### 0.6.2 Explicitly Out of Scope

- **Code modifications**: No existing source file will be changed
- **New source code**: No code will be added to the repository beyond the Markdown document
- **Build execution**: The C extension will not be compiled (the container lacks a C compiler; the failure itself is documented as evidence)
- **Runtime testing with a built binary**: Full kitty execution (rendering, window creation) is out of scope since it requires GPU access and a display server
- **Performance benchmarking**: No quantitative performance measurements
- **Go binary compilation**: The `kitten` binary will not be built
- **Documentation for unrelated features**: Remote control protocol details, SSH integration internals, shell integration scripts, and other features not directly asked about
- **macOS-specific code paths**: Cocoa/CoreText code (`cocoa_window.m`, `core_text.m`) is noted but not deeply analyzed since the investigation environment is Linux-based

## 0.7 Rules for Feature Addition

The user has specified the following implementation rules that must be strictly followed:

- **SWE-AtlasQnA-Repo rule**: Create a new Markdown document named `kitty_815df1e210e0.md` (matching the `<source_branch_name>`) that comprehensively answers the questions posed in the prompt.
- **Build and run to analyze**: The source code must be built and run as needed to analyze repository behavior. Answers must be based on observed behavior, not assumptions.
- **Provide thinking and rationale**: Each answer must include the reasoning behind the conclusion, connecting observed behavior to architectural decisions.
- **Do not modify existing files**: No existing files in the source repository may be modified.
- **Do not add other code**: No code beyond the requested document may be added to the source repository.
- **Document placement**: The generated document must be placed in the `blitzy/documentation` directory in the destination repo.
- **Temporary scripts**: May be used for observation but must be cleaned up afterward. The repository must remain in its original state.
- **Empirical method**: The user emphasizes "observing how the code behaves when exercised, rather than assuming how the architecture is meant to work." All conclusions must be grounded in evidence from actual execution attempts.

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were directly retrieved and analyzed during context gathering:

**Top-level files read:**
- `__main__.py` — Python entry point (7 lines)
- `setup.py` — Build system orchestrator (examined lines 302–313, 469–530, 906–932, 981–1000, 1050–1120, 1084–1095, 1130–1195)
- `pyproject.toml` — Project metadata
- `go.mod` — Go module definition with dependencies
- `Makefile` — Build convenience targets
- `dev.sh` — Development shell script

**`kitty/` directory files read:**
- `kitty/entry_points.py` — All CLI entry points and main() dispatch (full file)
- `kitty/main.py` — Application startup and event loop entry (lines 1–96, 220–240)
- `kitty/constants.py` — Version constants, path resolution, `kitty_exe()`, `kitten_exe()`, `read_kitty_resource()` (lines 1–84, 241–260)
- `kitty/boss.py` — Central orchestrator (lines 1–50, import analysis)
- `kitty/data-types.c` — C extension module definition and `PyInit_fast_data_types()` (lines 450–620)
- `kitty/fast_data_types.pyi` — Python type stubs for C extension (lines 1–100)
- `kitty/shaders.py` — GLSL shader loading and compilation orchestration (full file, 187 lines)
- `kitty/shaders.c` — C-side shader compilation and OpenGL program management (lines 1–80, 1160–1240)
- `kitty/gl.c` — OpenGL initialization (lines 1–60)
- `kitty/child-monitor.c` — Main event loop and three-thread architecture (lines 1–80, 1259–1310, 1370–1430)
- `kitty/screen.c` — Terminal screen model (lines 1–30)
- `kitty/vt-parser.c` — VT escape sequence parser (lines 1–50)
- `kitty/graphics.c` — Graphics protocol implementation (lines 1–60)
- `kitty/launcher/main.c` — Native C launcher embedding CPython (lines 1–80)
- `kitty/borders.py` — Border rendering (import chain traced)
- `kitty/typing.py` — Type alias stubs (full file)
- All 13 GLSL files: headers examined for purpose identification
- `kitty/fonts.c`, `kitty/child.c`, `kitty/colors.c` — size analysis

**`kittens/` directory files read:**
- `kittens/runner.py` — Kitten dispatch framework (full file, 180 lines)
- `kittens/diff/main.py` — Diff kitten Python stub (lines 1–30)
- `kittens/icat/main.py` — icat kitten Python stub (lines 1–30)
- `kittens/ask/main.py` — Ask kitten Python stub (lines 1–15)
- `kittens/show_key/main.py` — Show key kitten Python stub (full file)
- `kittens/hyperlinked_grep/main.py` — Hyperlinked grep stub (full file)
- `kittens/tui/loop.py` — TUI event loop foundation (lines 1–30)
- `kittens/tui/handler.py` — TUI handler base class (lines 1–20)

**Other directories examined:**
- `glfw/` — Directory listing, line count analysis
- `tools/` — Directory listing, Go file enumeration
- `tools/cmd/at/main.go` — Go tool entry point (lines 1–30)
- `shell-integration/ssh/kitty` — Wrapped kittens list (line 27)
- `3rdparty/` — Directory listing

**Broad searches conducted:**
- `find . -name "*.c"` — C file enumeration (128 files)
- `find . -name "*.py"` — Python file enumeration (214 files)
- `find . -name "*.go"` — Go file enumeration (258 files)
- `find . -name "*.glsl"` — GLSL file enumeration (13 files)
- `grep -rn "fast_data_types"` — Import dependency analysis across entire codebase
- `grep -rn "from kitty."` — Transitive dependency analysis for all kittens
- Module isolation test — Systematic import of all 43 Python modules in `kitty/`

### 0.8.2 Technical Specification Sections Referenced

- Section 1.1 Executive Summary — Project overview, three-language architecture, value proposition
- Section 3.1 Programming Languages — Detailed language roles, compiler flags, key source files
- Section 5.1 High-Level Architecture — Layered architecture, core components, data flow
- Section 7.2 GPU Rendering Pipeline — Shader architecture, font rendering, performance architecture

### 0.8.3 Attachments and External Resources

- **Docker image**: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (container: `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)
- **Source branch**: `kitty_815df1e210e0`
- **Head commit**: `815df1e21` ("Wire up applying of font config")
- **No Figma attachments**: N/A
- **No additional user-uploaded files**: The `/tmp/environments_files/` directory is empty

