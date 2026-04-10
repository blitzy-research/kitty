# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers an investigative, source-code-grounded question about kitty's build and test architecture — specifically mapping the relationship between compiled C extension modules and the test execution flow.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical investigation / Architecture documentation (build-test dependency analysis)
- **Target Artifact:** A markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory, per the `SWE-AtlasQnA-Repo` implementation rule

The user's requirements decompose into the following precise objectives:

- **Build Architecture Tracing:** Document how kitty's `setup.py`-based build system compiles three categories of C extension shared objects — `kitty/fast_data_types.so` (the monolithic core), `kitty/glfw-{x11,wayland,cocoa}.so` (platform GLFW backends), and `kittens/transfer/rsync.so` (the rsync kitten) — and how these artifacts feed into test execution
- **Test Execution Flow Mapping:** Trace how `test.py` → `kitty_tests/main.py` → `find_all_tests()` discovers and orchestrates the full test suite, and how each test module's import chain resolves to compiled C extension symbols exposed through `kitty.fast_data_types`
- **Extension-to-Test Dependency Analysis:** For each of the 22 test modules, classify whether they directly import from `kitty.fast_data_types`, transitively depend on it through intermediate Python modules (e.g., `kitty.config` → `kitty.conf.utils` → `fast_data_types.Color`), or have no C extension dependency
- **Failure Cascade Documentation:** Analyze and document the import failure pattern that emerges when `kitty/fast_data_types.so` is absent — specifically that the `kitty_tests/__init__.py` base test infrastructure itself fails to import (due to `kitty.conf.utils` → `fast_data_types.Color`), which cascades to prevent every single test module from loading
- **Critical vs. Optional Classification:** Map which C extension sub-modules within `fast_data_types` are critical (blocking test execution) versus those that are test-specifically consumed, and identify the few test modules that could theoretically run without compiled extensions if the base harness import chain were satisfied

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: Read-only codebase requirement.** The user explicitly states: "do not make any changes to the repository and leave the actual codebase unchanged by removing any temporary files when you're done." Temporary test scripts and sample data files may be created during investigation but must be removed afterward.
- **Source-grounded answers only.** Per the `SWE-AtlasQnA-Repo` rule: "Do not make assumptions, base your answers on the code as the truth."
- **Thinking/rationale required.** The rule states: "Provide thinking / rationale behind the answers."
- **No existing file modifications.** "Do not modify any existing files in the source repository."
- **Build attempt required.** The user requests actual building and test execution, with observations about what occurs. In the current environment, a full native build is not possible (no C compiler or dev packages available), so the documentation must clearly document this constraint and derive its analysis from exhaustive source code inspection rather than runtime observation.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the build architecture**, we will **create** a new markdown file `blitzy/documentation/kitty_815df1e210e0.md` that traces the compilation pipeline in `setup.py`, specifically the `build()` function (line 1084), `compile_c_extension()` (line 856), `compile_glfw()` (line 932), and `compile_kittens()` (line 967), documenting how each produces `.so` files
- To **document the test execution flow**, we will **analyze** `test.py`, `kitty_tests/main.py` (specifically `run_tests()`, `find_all_tests()`, `env_for_python_tests()`), and `kitty_tests/__init__.py` (`BaseTest`, `PTY`, `Callbacks`), mapping the complete orchestration chain
- To **trace extension-to-test dependencies**, we will **analyze** every import statement across all 22 test modules in `kitty_tests/` and the intermediate kitty modules they reference, producing a comprehensive dependency matrix
- To **document failure cascades**, we will **analyze** the import chain starting from `kitty_tests/__init__.py` line 22 (`from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, ...`) and the transitive chain `kitty.config` → `kitty.conf.utils` → `kitty.fast_data_types.Color`, documenting why every test module fails when the `.so` is absent

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- **The `kitty/data-types.c` module is a monolithic extension hub.** It registers 30+ sub-initializers (`init_LineBuf`, `init_Screen`, `init_graphics`, `init_crypto_library`, etc.) into a single `fast_data_types` Python module via `PyInit_fast_data_types()` (line 525). This architectural choice means the extension is all-or-nothing — there is no way to load a subset of functionality.
- **The `BaseTest` class has an inescapable transitive dependency on `fast_data_types`.** The import chain `kitty_tests/__init__.py` → `kitty.config` → `kitty.conf.utils` → `kitty.fast_data_types.Color` means that even test modules that never directly use C extensions (like `layout.py`, `search_query_parser.py`, `open_actions.py`) cannot load their base class without the compiled extension.
- **Platform-conditional initialization exists.** On Linux, `init_freetype_library`, `init_fontconfig_library`, and `init_desktop` are called, while on macOS, `init_CoreText`, `init_cocoa`, and `init_macos_process_info` are called instead. If any single platform initializer fails, the entire module fails to load.
- **The Go test suite runs independently** in parallel via `GoProc(Thread)`, meaning Go tests do not depend on C extension availability and can execute even when Python extension loading fails.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature, Sphinx-based documentation infrastructure with comprehensive user-facing documentation but no dedicated build-test architecture documentation of the kind requested.

- **Documentation framework:** Sphinx (reStructuredText-based), using the `furo` theme
- **Documentation generator configuration:** `docs/conf.py` (Sphinx configuration), `docs/Makefile` (build targets)
- **Documentation dependencies:** `docs/requirements.txt` — sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild
- **Existing build documentation:** `docs/build.rst` covers user-facing build instructions (dependencies, `./dev.sh build`, debug modes), but does not document internal build-test architecture or extension-test dependency relationships
- **Existing test documentation:** No dedicated testing documentation exists in `docs/`. Test architecture knowledge is encoded entirely within the source files (`kitty_tests/main.py`, `setup.py`)
- **Documentation hosting:** Generated HTML at `docs/_build/html/`, man pages at `docs/_build/man/`
- **API documentation tools:** No JSDoc, Sphinx autodoc, or similar API documentation generators are configured. The Python typing stub `kitty/fast_data_types.pyi` serves as the closest thing to API reference documentation for C extensions.
- **Diagram tools:** The existing docs use external badge images but no Mermaid or PlantUML diagrams. The tech spec already uses Mermaid extensively.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code relevant to the documentation objective:

- **Build system entry points:** `setup.py` (2,173 lines) — the central build orchestrator; `Makefile` (72 lines) — convenience wrapper; `test.py` (13 lines) — test bootstrapper
- **C extension compilation pipeline:** `setup.py::build()` (line 1084), `setup.py::compile_c_extension()` (line 856), `setup.py::compile_glfw()` (line 932), `setup.py::compile_kittens()` (line 967), `setup.py::find_c_files()` (line 906)
- **C extension module initialization:** `kitty/data-types.c` lines 525–620 — `PyInit_fast_data_types()` registering 30+ sub-modules
- **Test infrastructure:** `kitty_tests/__init__.py` (415 lines) — `BaseTest`, `PTY`, `Callbacks`, helper functions; `kitty_tests/main.py` (339 lines) — test runner/orchestrator
- **All 22 test modules** in `kitty_tests/`: `check_build.py`, `clipboard.py`, `completion.py`, `crypto.py`, `datatypes.py`, `file_transmission.py`, `fonts.py`, `glfw.py`, `graphics.py`, `keys.py`, `layout.py`, `main.py`, `mouse.py`, `open_actions.py`, `options.py`, `parser.py`, `screen.py`, `search_query_parser.py`, `shell_integration.py`, `shm.py`, `ssh.py`, `tui.py`, `utmp.py`
- **GLFW backend modules:** `glfw/` directory — platform-specific GLFW builds producing `kitty/glfw-{x11,wayland,cocoa}.so`
- **Kittens extension:** `kittens/transfer/algorithm.c` — compiled into `kittens/transfer/rsync.so`
- **Python version requirement:** `pyproject.toml` specifies `requires-python = ">=3.8"`
- **Go version requirement:** `go.mod` specifies `go 1.22`

Key directories examined:
- `kitty/` — Core application source (C and Python)
- `kitty_tests/` — All test modules
- `glfw/` — GLFW backend compilation support
- `kittens/transfer/` — Rsync extension C source
- `3rdparty/` — Vendored dependencies (ringbuf, base64)
- `docs/` — Existing Sphinx documentation tree

### 0.2.3 Build Environment Assessment

The current sandboxed environment lacks the prerequisites to build kitty from source:

- **Missing C compiler:** No `gcc`, `cc`, or `clang` executable available
- **Missing development packages:** No `-dev` headers for harfbuzz, libpng, lcms2, freetype, fontconfig, libgl, libssl, xxhash, or X11/Wayland development libraries
- **No apt repository access:** The `apt-get install` command cannot resolve any packages
- **Available runtime libraries:** `libharfbuzz.so.0`, `libpng16.so.16`, `liblcms2.so.2`, `libfreetype.so.6`, `libfontconfig.so.1` are present as runtime shared libraries
- **Available Python:** Python 3.12.3 with `python3.12-dev` headers installed
- **No Go compiler:** The `go` binary is not available in the PATH

This means: No `.so` extension files can be compiled, and consequently `import kitty.fast_data_types` fails with `ModuleNotFoundError`. All analysis in the resulting documentation is derived from exhaustive source code inspection, AST-level import tracing, and static analysis of the build pipeline.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The documentation must trace relationships across three codebases: the C build pipeline, the Python test harness, and the extension import graph.

**Module: Build System (`setup.py`)**
- Public APIs: `build()`, `compile_c_extension()`, `compile_glfw()`, `compile_kittens()`, `find_c_files()`, `build_launcher()`
- Current documentation: Partially covered in `docs/build.rst` (user-facing only)
- Documentation needed: Internal architecture trace of the compilation pipeline, produced artifacts, and their role in test execution

**Module: C Extension Hub (`kitty/data-types.c`)**
- Public APIs: `PyInit_fast_data_types()` registering 30+ sub-initializers
- Current documentation: `kitty/fast_data_types.pyi` typing stub
- Documentation needed: Inventory of all initialized sub-modules and their downstream test consumers

**Module: Test Runner (`kitty_tests/main.py`)**
- Public APIs: `run_tests()`, `find_all_tests()`, `run_python_tests()`, `run_go()`, `env_for_python_tests()`
- Current documentation: None
- Documentation needed: Orchestration flow, discovery mechanism, concurrent Go/Python execution model

**Module: Test Infrastructure (`kitty_tests/__init__.py`)**
- Public APIs: `BaseTest`, `PTY`, `Callbacks`, `parse_bytes()`, `filled_line_buf()`, etc.
- Current documentation: None
- Documentation needed: Import dependency chain from `BaseTest` to `fast_data_types`, cascade analysis

**Module: All 22 Test Modules**
- Current documentation: None
- Documentation needed: Per-module classification of direct, transitive, and zero C extension dependencies

### 0.3.2 Extension Module Dependency Matrix

Each test module's dependency on compiled C extensions was traced via AST import analysis:

| Test Module | Direct `fast_data_types` Imports | Transitive via Base Harness | Key Symbols Used |
|---|---|---|---|
| `__init__.py` (BaseTest) | **Yes** — Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options | N/A (is the harness) | Core terminal model types |
| `datatypes.py` | **Yes** — Color, ColorProfile, HistoryBuf, LineBuf, Cursor, wcswidth, wcwidth, expand_ansi_c_escapes, parse_input_from_terminal, SingleKey | Yes | Low-level data type validation |
| `parser.py` | **Yes** — CURSOR_BLOCK, VT_PARSER_BUFFER_SIZE, base64_decode, base64_encode, has_avx2, has_sse4_2, test_find_either_of_two_bytes | Yes | VT parser + SIMD parity |
| `screen.py` | **Yes** — DECAWM, DECCOLM, DECOM, IRM, VT_PARSER_BUFFER_SIZE, Cursor | Yes | Screen model constants |
| `graphics.py` | **Yes** — base64_decode, base64_encode, has_avx2, has_sse4_2, load_png_data, shm_unlink, shm_write, test_xor64 | Yes | Graphics protocol + SHM |
| `fonts.py` | **Yes** — DECAWM, get_fallback_font, sprite_map_set_layout, sprite_map_set_limits, test_render_line, wcwidth | Yes | Font pipeline native APIs |
| `keys.py` | **No** (uses `kitty.fast_data_types` as `defines` module) | Yes | Key encoding constants |
| `mouse.py` | **Yes** — GLFW_MOD_ALT, GLFW_MOD_CONTROL, GLFW_MOUSE_BUTTON_LEFT, create_mock_window, send_mock_mouse_event_to_window | Yes | Mouse event simulation |
| `crypto.py` | **Yes** (deferred) — AES256GCMDecrypt, AES256GCMEncrypt, CryptoError, EllipticCurveKey | Yes | Crypto primitives |
| `shell_integration.py` | **Yes** — CURSOR_BEAM, CURSOR_BLOCK, CURSOR_UNDERLINE | Yes | Cursor shape constants |
| `ssh.py` | **Yes** — CURSOR_BEAM, shm_unlink | Yes | Cursor + SHM |
| `options.py` | **Yes** — Color | Yes | Color type for config parsing |
| `shm.py` | **Yes** — shm_unlink | Yes | Shared memory cleanup |
| `utmp.py` | **Yes** — num_users | Yes | User counting native function |
| `main.py` (runner) | **Yes** — has_avx2, has_sse4_2 | Yes | Intrinsics reporting |
| `check_build.py` | **No** direct (deferred in test methods) | Yes | Build verification |
| `glfw.py` | **No** direct | Yes | GLFW utility functions |
| `clipboard.py` | **No** direct | Yes | Clipboard protocol |
| `completion.py` | **No** direct | Yes | Completion engine |
| `layout.py` | **No** direct | Yes | Layout algorithms |
| `file_transmission.py` | **No** direct | Yes | File transfer protocol |
| `open_actions.py` | **No** direct | Yes | URL/MIME matching |
| `search_query_parser.py` | **No** direct | Yes | Query parsing |
| `tui.py` | **No** direct | Yes | TUI components |

### 0.3.3 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing documentation** traces the relationship between compiled C extensions and the test suite
- **No existing documentation** maps the import dependency graph from test modules to `fast_data_types`
- **No existing documentation** classifies test modules by their C extension dependency depth (direct/transitive/none)
- **No existing documentation** explains the monolithic nature of `PyInit_fast_data_types()` and its all-or-nothing initialization pattern
- **No existing documentation** describes the failure cascade when extensions are unavailable
- **The `docs/build.rst` file** covers user-facing build instructions but not internal architecture
- **The tech spec Section 6.6** covers the testing strategy comprehensively but does not trace extension-to-test import chains at the granularity requested


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The documentation will be placed as a single comprehensive markdown file following the `SWE-AtlasQnA-Repo` rule:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
```

The document's internal structure will address each question posed by the user:

```
kitty_815df1e210e0.md
├── Introduction and Methodology
├── Build Architecture
│   ├── The Compilation Pipeline (setup.py → .so artifacts)
│   ├── C Extension Modules Inventory (3 extension families)
│   └── The Monolithic fast_data_types Module (30+ sub-initializers)
├── Test Architecture
│   ├── Test Runner Orchestration (test.py → main.py → unittest)
│   ├── Test Infrastructure (BaseTest, PTY, Callbacks)
│   └── Go/Python Concurrent Execution Model
├── Extension-to-Test Dependency Map
│   ├── Direct Importers (15 modules)
│   ├── Transitive-Only Importers (8 modules)
│   ├── The Critical Base Harness Chain
│   └── Complete Import Chain Diagram
├── Failure Cascade Analysis
│   ├── What Happens Without fast_data_types.so
│   ├── The Inescapable BaseTest Dependency
│   ├── Platform-Conditional Initialization Failures
│   └── Go Tests: The Independent Survivor
├── Critical vs. Optional Module Classification
│   ├── Critical Extension Sub-modules
│   ├── Test-Category-Specific Sub-modules
│   └── Theoretical Isolation Analysis
└── Conclusions
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the compilation pipeline from `setup.py::build()` (line 1084), `compile_c_extension()` (line 856), and `find_c_files()` (line 906)
- Extract all 30+ sub-initializer registrations from `kitty/data-types.c` lines 540–575
- Map import chains using AST-level analysis of all 22 test modules and their transitive kitty module dependencies
- Trace the critical failure path: `kitty_tests/__init__.py:22` → `kitty.config:10` → `kitty.conf.utils:27` → `kitty.fast_data_types.Color`
- Document the Go test independence by analyzing `kitty_tests/main.py::GoProc` and `run_go()`

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` heading hierarchy
- Mermaid diagrams for the build pipeline, import dependency graph, and failure cascade flow
- Tables for the per-module dependency matrix, sub-initializer inventory, and classification
- Source citations as `Source: /path/to/file.py:LineNumber` inline references
- Code examples using fenced code blocks with syntax highlighting

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the documentation:

- **Build Pipeline Flowchart:** `setup.py` → `build()` → `compile_c_extension()` / `compile_glfw()` / `compile_kittens()` → `.so` outputs
- **Test Runner Orchestration:** `test.py` → `kitty_tests/main.py::run_tests()` → `find_all_tests()` + `run_go()` → concurrent execution
- **Import Dependency Graph:** All 22 test modules → intermediate kitty modules → `fast_data_types` with edge annotations
- **Failure Cascade Chain:** `fast_data_types.so missing` → `__init__.py import fails` → `BaseTest unavailable` → all test modules fail
- **C Extension Initialization Sequence:** `PyInit_fast_data_types()` → sequential `init_*()` calls with platform conditionals

### 0.4.4 Template Application

Per the `SWE-AtlasQnA-Repo` rule, the document must:
- Comprehensively answer the questions posed in the prompt
- Provide thinking/rationale behind the answers
- Not make assumptions — base answers on the code as the truth
- Not modify any existing files in the source repository


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `setup.py`, `kitty/data-types.c`, `kitty_tests/main.py`, `kitty_tests/__init__.py`, all 22 test modules, `kitty/constants.py`, `kitty/conf/utils.py`, `kitty/config.py`, `kitty/options/types.py`, `kitty/utils.py`, `kitty/window.py`, `kitty/child.py`, `kitty/keys.py`, `kitty/shm.py`, `kitty/fonts/common.py`, `kitty/fonts/render.py`, `kitty/shaders.py`, `kitty/file_transmission.py`, `kitty/clipboard.py`, `kitty/notify.py`, `kitty/key_encoding.py`, `Makefile`, `test.py`, `pyproject.toml`, `go.mod`, `docs/build.rst` | Comprehensive Q&A document tracing the relationship between compiled C extensions and the test execution flow, covering build pipeline, extension-test dependency mapping, failure cascades, and critical vs. optional module classification |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Architecture Q&A
Source Code:
  Primary: setup.py, kitty/data-types.c, kitty_tests/main.py, kitty_tests/__init__.py
  Secondary: All 22 kitty_tests/*.py modules, kitty/conf/utils.py, kitty/config.py
  Tertiary: kitty/constants.py, kitty/fonts/common.py, kitty/shaders.py, Makefile, go.mod
Sections:
  - Introduction and Methodology
  - Build Architecture (setup.py compilation pipeline, 3 extension families)
  - The Monolithic fast_data_types Module (30+ sub-initializers from data-types.c)
  - Test Runner Orchestration (test.py → main.py → unittest + GoProc)
  - Test Infrastructure (BaseTest, PTY, Callbacks from __init__.py)
  - Extension-to-Test Dependency Matrix (all 22 modules classified)
  - Import Chain Analysis (direct, transitive, and zero-dependency modules)
  - Failure Cascade Analysis (missing .so → BaseTest import failure → total cascade)
  - Platform-Conditional Initialization (Linux vs macOS sub-initializers)
  - Critical vs. Optional Sub-module Classification
  - Go Test Independence (GoProc thread, separate process model)
  - Conclusions
Diagrams:
  - Build pipeline flowchart (setup.py → .so outputs)
  - Test runner orchestration sequence
  - Import dependency graph (test modules → fast_data_types)
  - Failure cascade chain diagram
  - C extension initialization sequence with platform conditionals
Key Citations:
  setup.py:525-620, setup.py:856-904, setup.py:906-929, setup.py:932-954,
  setup.py:967-992, setup.py:1084-1095, setup.py:2098-2103,
  kitty/data-types.c:525-620, kitty_tests/__init__.py:21-27,
  kitty_tests/__init__.py:208-257, kitty_tests/main.py:57-66,
  kitty_tests/main.py:210-243, kitty_tests/main.py:246-279,
  kitty_tests/main.py:297-331, kitty/conf/utils.py:27,
  kitty/config.py:10, test.py:1-13, Makefile:15-16
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The new file is placed in `blitzy/documentation/` which is an output directory for the Blitzy platform's Q&A documentation, not part of the kitty project's Sphinx documentation build.

### 0.5.4 Cross-Documentation Dependencies

- The new document references the existing `docs/build.rst` for user-facing build instructions context
- The document cross-references the tech spec Section 6.6 (Testing Strategy) and Section 8.2 (Build System Architecture) for broader architectural context
- No navigation, ToC, or index updates are needed in the existing documentation tree since this document is placed in the separate `blitzy/documentation/` directory


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

No external documentation generation tools are required for this task. The output is a standalone Markdown file with embedded Mermaid diagrams.

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| N/A | Markdown | N/A | Document format — rendered by any Markdown viewer or GitHub |
| N/A | Mermaid | N/A | Diagram notation — rendered by GitHub, GitLab, and Mermaid-compatible viewers |

### 0.6.2 Build System Dependencies (Documented in Output)

The following dependencies are documented within the output file as part of the build architecture analysis. These are kitty's build-time and runtime dependencies, not documentation tool dependencies:

| Registry | Package Name | Version | Purpose in Kitty |
|---|---|---|---|
| system | Python | >= 3.8 | Build host interpreter and runtime (from `pyproject.toml`) |
| system | Go | >= 1.22 | Static `kitten` binary compilation (from `go.mod`) |
| system | gcc or clang | Any C11-capable | C extension compilation |
| system | pkg-config | Any | Library discovery for harfbuzz, libpng, lcms2, etc. |
| system | harfbuzz | >= 2.2.0 | Text shaping library (from `docs/build.rst`) |
| system | libpng | Any | PNG image support |
| system | liblcms2 | Any | Color management |
| system | libxxhash | Any | Fast hashing for rsync kitten |
| system | openssl/libcrypto | Any | Cryptographic operations (AES-256-GCM, X25519) |
| system | freetype | Any (Linux) | Font rendering |
| system | fontconfig | Any (Linux) | Font discovery |
| pip | sphinx | latest | Existing documentation build (from `docs/requirements.txt`) |
| pip | furo | latest | Sphinx theme (from `docs/requirements.txt`) |
| go module | github.com/google/go-cmp | v0.6.0 | Go test assertion library (from `go.mod`) |

### 0.6.3 Documentation Reference Updates

Not applicable. The new document is self-contained in `blitzy/documentation/` and does not require link updates to any existing documentation files.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

Current coverage analysis for the specific documentation objective:

- **Build pipeline documented:** 0% → Target: 100% (all 3 extension compilation paths traced)
- **Test module dependency classification:** 0% → Target: 100% (all 22 test modules + main.py + __init__.py classified)
- **C extension sub-initializer inventory:** 0% → Target: 100% (all 30+ sub-modules in `data-types.c` cataloged)
- **Failure cascade paths documented:** 0% → Target: 100% (complete import chain from missing `.so` to test failure)
- **Critical vs. optional classification:** 0% → Target: 100% (every sub-initializer categorized)

Coverage gaps to address:

| Area | Current | Target | Focus |
|---|---|---|---|
| Build pipeline trace | 0% | 100% | `setup.py::build()` → `compile_c_extension()` → `.so` outputs |
| Extension sub-module inventory | 0% | 100% | All `init_*()` calls in `PyInit_fast_data_types()` |
| Test module import chains | 0% | 100% | Direct + transitive `fast_data_types` dependencies for all 22 modules |
| Failure cascade documentation | 0% | 100% | `__init__.py` → `kitty.config` → `kitty.conf.utils` → `fast_data_types` |
| Platform-conditional paths | 0% | 100% | Linux (freetype/fontconfig/desktop) vs macOS (CoreText/cocoa) |
| Go test independence | 0% | 100% | `GoProc` thread model, separate `go test` process |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question in the user's prompt must be directly and explicitly answered
- All 22 test modules must appear in the dependency classification matrix
- All 30+ sub-initializers in `data-types.c` must be inventoried
- The failure cascade must be traced from root cause to observable effect
- Both Python and Go test execution paths must be documented

**Accuracy validation:**
- Every claim must cite a specific source file and line number
- Import chains must be verified against actual `import` statements in the source
- The sub-initializer list must match the actual `init_*()` calls in `data-types.c`
- Platform-conditional compilation paths must match the `#ifdef __APPLE__` guards

**Clarity standards:**
- Mermaid diagrams for all complex relationships (build pipeline, import graph, failure cascade)
- Tables for structured data (dependency matrix, sub-initializer inventory)
- Progressive disclosure: overview first, then detailed per-module analysis
- Explicit rationale for each classification decision, per the `SWE-AtlasQnA-Repo` rule

**Maintainability:**
- Source citations link every claim to specific file paths and line numbers
- Structured tables enable easy updating if test modules are added or removed
- Mermaid diagrams can be regenerated from the textual descriptions

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 (build pipeline, test orchestration, import dependency graph, failure cascade)
- **Diagram types required:** Mermaid flowchart and sequence diagrams
- **Tables required:** Extension sub-initializer inventory, test module dependency matrix, critical vs. optional classification
- **Code examples:** Short inline references to specific import lines (e.g., `from kitty.fast_data_types import Cursor, Screen`)


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — the complete Q&A document

- **Source files analyzed for documentation content:**
  - `setup.py` — build pipeline, compilation functions, test invocation
  - `Makefile` — build target convenience wrappers
  - `test.py` — test bootstrapper entry point
  - `kitty/data-types.c` — `PyInit_fast_data_types()` and all `init_*()` registrations
  - `kitty_tests/__init__.py` — `BaseTest`, `PTY`, `Callbacks`, import dependencies
  - `kitty_tests/main.py` — test runner, discovery, filtering, Go test orchestration
  - `kitty_tests/check_build.py` — build verification tests
  - `kitty_tests/clipboard.py` — clipboard test imports
  - `kitty_tests/completion.py` — completion test imports
  - `kitty_tests/crypto.py` — cryptographic test imports
  - `kitty_tests/datatypes.py` — data types test imports
  - `kitty_tests/file_transmission.py` — file transfer test imports
  - `kitty_tests/fonts.py` — font test imports
  - `kitty_tests/glfw.py` — GLFW test imports
  - `kitty_tests/graphics.py` — graphics test imports
  - `kitty_tests/keys.py` — keyboard test imports
  - `kitty_tests/layout.py` — layout test imports
  - `kitty_tests/mouse.py` — mouse test imports
  - `kitty_tests/open_actions.py` — open actions test imports
  - `kitty_tests/options.py` — options test imports
  - `kitty_tests/parser.py` — parser test imports
  - `kitty_tests/screen.py` — screen test imports
  - `kitty_tests/search_query_parser.py` — search parser test imports
  - `kitty_tests/shell_integration.py` — shell integration test imports
  - `kitty_tests/shm.py` — shared memory test imports
  - `kitty_tests/ssh.py` — SSH test imports
  - `kitty_tests/tui.py` — TUI test imports
  - `kitty_tests/utmp.py` — utmp test imports
  - `kitty/conf/utils.py` — transitive import chain analysis
  - `kitty/config.py` — transitive import chain analysis
  - `kitty/options/types.py` — transitive import chain analysis
  - `kitty/constants.py` — `glfw_path()`, `kitty_exe()`, `extensions_dir`
  - `kitty/utils.py` — transitive `fast_data_types` imports
  - `kitty/window.py` — transitive `fast_data_types` imports
  - `kitty/child.py` — transitive `fast_data_types` imports
  - `kitty/keys.py` — transitive `fast_data_types` imports
  - `kitty/shm.py` — transitive `fast_data_types` imports
  - `kitty/fonts/common.py` — transitive `fast_data_types` imports
  - `kitty/fonts/render.py` — transitive `fast_data_types` imports
  - `kitty/shaders.py` — transitive `fast_data_types` imports
  - `kitty/file_transmission.py` — transitive `fast_data_types` imports
  - `kitty/clipboard.py` — transitive `fast_data_types` imports
  - `kitty/notify.py` — transitive `fast_data_types` imports
  - `kitty/key_encoding.py` — transitive `fast_data_types` imports
  - `pyproject.toml` — Python version requirement
  - `go.mod` — Go version and module dependencies
  - `docs/build.rst` — existing build documentation reference
  - `docs/requirements.txt` — documentation tool dependencies
  - `glfw/` — GLFW backend compilation context
  - `kittens/transfer/` — rsync kitten extension context
  - `3rdparty/` — vendored C library sources (ringbuf, base64)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No changes to any repository file. The `SWE-AtlasQnA-Repo` rule and user instructions both prohibit modifications.
- **Test file modifications** — No changes to any test file
- **Feature additions or code refactoring** — Not applicable
- **Deployment configuration changes** — Not applicable
- **Documentation in the `docs/` directory** — The new document goes in `blitzy/documentation/`, not in kitty's Sphinx tree
- **Actual compilation of C extensions** — The environment lacks a C compiler and development libraries; analysis is source-code-based
- **Actual test execution with runtime output** — Without compiled extensions, the test suite cannot run; failure behavior is documented from static analysis
- **Go test execution** — No Go compiler available in the environment
- **CI/CD pipeline documentation** — Covered comprehensively in tech spec Section 6.6.11, not duplicated here
- **Performance benchmarking** — Not part of the user's question
- **Runtime analysis with instrumentation** — No compiled artifacts to instrument


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file (`blitzy/documentation/kitty_815df1e210e0.md`), not a Sphinx/MkDocs site
- **Documentation preview command:** Any Markdown renderer (e.g., `grip`, VS Code preview, GitHub rendering)
- **Diagram generation:** Mermaid diagrams are embedded inline within the Markdown using fenced mermaid code blocks; no separate generation step required
- **Default format:** Markdown with Mermaid diagrams for architecture flows, tables for dependency matrices, and fenced code blocks for file excerpts
- **Citation requirement:** Every technical claim must reference the source file and line number where the evidence was found (e.g., `Source: setup.py:1084`, `Source: kitty/data-types.c:540-575`)

### 0.9.2 Build Commands Reference (for Documentation Content)

The following commands are documented within the output file as part of the build-and-test architecture analysis — they are not executed by the documentation generation process itself:

| Command | Purpose | Source |
|---------|---------|--------|
| `python3 setup.py` | Full build from source | `Makefile:5-6` |
| `python3 setup.py build --debug` | Debug build with symbols | `setup.py:2089-2093` |
| `python3 setup.py test` | Run full test suite | `setup.py:2101-2103` |
| `make test` | Convenience wrapper for test | `Makefile:16` |
| `python3 setup.py linux-package` | Linux packaging build | `setup.py:2095-2100` |
| `make clean` | Clean build artifacts | `Makefile:10-13` |

### 0.9.3 Analysis Methodology

Because the sandbox environment lacks a C compiler and development libraries, the analysis methodology for this documentation task is:

- **Static source-code tracing** rather than runtime observation
- **AST-level import parsing** of all 22 test modules to extract dependency graphs
- **C source inspection** of `PyInit_fast_data_types()` to inventory registered sub-initializers
- **Transitive dependency chain tracing** through intermediate Python modules (`kitty.config` → `kitty.conf.utils` → `fast_data_types`)
- **Cross-referencing** of `setup.py` compilation functions with the C source files they process

### 0.9.4 Style Guide

- **Tone:** Technical, analytical, and evidence-based; appropriate for an engineering audience investigating build internals
- **Structure:** Question-driven sections that directly address the user's stated areas of interest
- **Code references:** Inline `Source: file:line` citations for all factual claims
- **Diagram style:** Mermaid with clear node labels, directional edges, and legend annotations where needed
- **Table formatting:** Standard Markdown tables with left-aligned text columns
- **No speculation:** Where runtime behavior cannot be verified statically, the document explicitly states the limitation and provides the strongest available static evidence


## 0.10 Rules for Documentation


### 0.10.1 User-Specified Directives

The following rules are derived directly from the user's instructions and the project-level `SWE-AtlasQnA-Repo` implementation rule:

- **SWE-AtlasQnA-Repo Rule:** Create a new Markdown document named `kitty_815df1e210e0.md` (matching the source branch name) that comprehensively answers the questions posed in the prompt. Place the generated document in the `blitzy/documentation` directory.
- **Provide thinking and rationale:** Do not simply list facts; explain the reasoning behind each answer and the evidence chain that supports it.
- **Base answers on code as the truth:** Do not make assumptions. Every claim must be traceable to specific source files and line numbers in the repository.
- **Do not modify any existing files** in the source repository. The codebase must remain entirely unchanged.
- **Temporary files are permitted** for analysis purposes (e.g., test scripts, sample data files) but must be removed when done. The user explicitly stated: "leave the actual codebase unchanged by removing any temporary files when you're done."

### 0.10.2 Documentation Content Rules

- **Answer all three question threads comprehensively:**
  - Build architecture and compiled extension structure
  - Extension module loading during test execution and failure cascade behavior
  - Import chains, critical vs. optional modules, and test category dependency mapping
- **Include Mermaid diagrams** for all architectural relationships (build pipeline, dependency graph, failure cascade, import chain)
- **Include the complete extension module dependency matrix** covering all 22 test modules with their direct and transitive `fast_data_types` dependencies
- **Classify every test module** by dependency criticality: critical (direct imports of core extension symbols), high (direct imports of specialized extension symbols), or transitive-only (depends solely through `BaseTest` inheritance)
- **Document the monolithic nature** of `fast_data_types.so` — explain that all 30+ sub-initializers are registered sequentially in a single `PyInit_fast_data_types()` call, so any initialization failure prevents the entire module from loading
- **Distinguish Python tests from Go tests** — Go tests run independently via `go test` in a separate process and have no dependency on compiled C extensions
- **Acknowledge analysis limitations** — State clearly where conclusions are based on static source analysis versus runtime observation, and where the absence of a build environment constrains the analysis

### 0.10.3 Formatting and Structure Rules

- **Use standard Markdown** with GitHub-compatible syntax
- **Mermaid diagrams** for all architectural flows and dependency relationships
- **Tables** for structured data (dependency matrices, sub-initializer inventories, import chains)
- **Fenced code blocks** with language annotations for file excerpts (e.g., `python`, `c`, `bash`)
- **Source citations** inline using the format `Source: filename:line` or `Source: filename:line-range`
- **Section headers** following a logical progression from build overview → extension structure → test architecture → dependency analysis → failure patterns → conclusions


## 0.11 References


### 0.11.1 Repository Files and Folders Searched

The following files and folders were inspected during context gathering to derive the conclusions documented in this Agent Action Plan:

**Build System Files:**

| File | Lines Read | Key Information Extracted |
|------|-----------|--------------------------|
| `setup.py` | 1-2173 (full) | Build orchestration, `compile_c_extension()`, `compile_glfw()`, `compile_kittens()`, `find_c_files()`, platform detection, test invocation |
| `Makefile` | 1-72 (full) | Build targets: `all`, `test`, `clean`, `docs`, delegation to `setup.py` |
| `test.py` | 1-13 (full) | Test bootstrapper, imports `kitty_tests.main`, calls `main()` |
| `pyproject.toml` | 1-33 (full) | Python >= 3.8 requirement, mypy strict configuration |
| `go.mod` | 1-5 (header) | Go 1.22, module path `kitty` |

**C Extension Source Files:**

| File | Lines Read | Key Information Extracted |
|------|-----------|--------------------------|
| `kitty/data-types.c` | 525-580 | `PyInit_fast_data_types()` — 30+ `init_*()` sub-initializer registrations |

**Test Infrastructure Files:**

| File | Lines Read | Key Information Extracted |
|------|-----------|--------------------------|
| `kitty_tests/__init__.py` | 1-415 (full) | `BaseTest(TestCase)`, `PTY`, `Callbacks`, `parse_bytes()`, `filled_line_buf()`, critical import of `fast_data_types` at line 22 |
| `kitty_tests/main.py` | 1-339 (full) | `run_tests()`, `find_all_tests()`, `run_python_tests()`, `run_go()`, `GoProc(Thread)`, `env_for_python_tests()` |
| `kitty_tests/check_build.py` | 1-128 (full) | Build verification test class, `has_*` capability flags |

**Test Module Import Analysis (first ~30 lines each for import extraction):**

| File | Lines Read | Direct fast_data_types Imports |
|------|-----------|-------------------------------|
| `kitty_tests/datatypes.py` | 1-30 | `LineBuf`, `Cursor`, `HistoryBuf` |
| `kitty_tests/parser.py` | 1-30 | `parse_bytes` (via BaseTest) |
| `kitty_tests/crypto.py` | 1-30 | Deferred imports within test methods |
| `kitty_tests/graphics.py` | 1-30 | `set_send_to_gpu`, `parse_bytes` |
| `kitty_tests/fonts.py` | 1-30 | `coretext_all_fonts`, `FontConfigPattern`, etc. |
| `kitty_tests/screen.py` | 1-30 | `Cursor`, `LineBuf`, `HistoryBuf` |
| `kitty_tests/keys.py` | 1-30 | `kitty.fast_data_types as defines` |
| `kitty_tests/glfw.py` | 1-54 (full) | `GLFW_*` constants |
| `kitty_tests/layout.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/options.py` | 1-30 | `Color` |
| `kitty_tests/clipboard.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/completion.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/file_transmission.py` | 1-30 | `shm_unlink` |
| `kitty_tests/mouse.py` | 1-30 | `GLFW_*` mouse constants |
| `kitty_tests/open_actions.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/search_query_parser.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/shell_integration.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/shm.py` | 1-30 | `SharedMemory` (via `kitty.shm`) |
| `kitty_tests/ssh.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/tui.py` | 1-30 | None direct — transitive through BaseTest |
| `kitty_tests/utmp.py` | 1-30 | None direct — transitive through BaseTest |

**Transitive Dependency Chain Files:**

| File | Lines Read | Key Information Extracted |
|------|-----------|--------------------------|
| `kitty/config.py` | 1-30 | Imports `kitty.conf.utils` — entry point of transitive chain |
| `kitty/conf/utils.py` | 25-30 | Line 27: `from kitty.fast_data_types import Color` — the critical link |
| `kitty/options/types.py` | 1-30 | Imports from `fast_data_types` |
| `kitty/utils.py` | 1-30 | Imports from `fast_data_types` |
| `kitty/window.py` | 1-30 | Imports from `fast_data_types` |
| `kitty/constants.py` | 1-30 | `glfw_path()` returns GLFW `.so` path |

**Documentation and Configuration Files:**

| File | Lines Read | Key Information Extracted |
|------|-----------|--------------------------|
| `docs/build.rst` | 1-full | User-facing build-from-source documentation |
| `docs/requirements.txt` | 1-full | Documentation tool dependencies (Sphinx, theme) |

**Folders Explored:**

| Folder | Purpose |
|--------|---------|
| `/` (root) | Repository structure overview |
| `kitty/` | Core source directory — C extensions and Python modules |
| `kitty_tests/` | Full test suite directory — all 22 test modules |
| `docs/` | Existing Sphinx documentation |
| `glfw/` | GLFW backend source files |
| `kittens/transfer/` | Rsync kitten extension |
| `3rdparty/` | Vendored C libraries (ringbuf, base64) |

### 0.11.2 Tech Spec Sections Retrieved

| Section | Key Information Used |
|---------|---------------------|
| 1.1 Executive Summary | Project overview — kitty is a GPU-accelerated terminal emulator written in C and Python with Go-based kittens |
| 6.6 Testing Strategy | ~144 Python tests, ~64 Go tests, test categories, coverage framework |
| 8.2 Build System Architecture | Compilation pipeline, extension targets, platform-conditional builds |

### 0.11.3 User Attachments

No attachments were provided for this project.

### 0.11.4 Figma Screens

No Figma screens were provided or referenced for this project.

### 0.11.5 External URLs

No external URLs were provided by the user. All analysis is based on repository source code inspection.


