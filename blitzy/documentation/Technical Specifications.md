# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative research document** that answers four tightly coupled questions about the Kitty terminal emulator's startup lifecycle — from process launch through to a shell displaying its first output. The document must be grounded in concrete evidence drawn directly from the source code at commit `815df1e21` (branch `kitty_815df1e210e0`), supplemented by actual runtime observations obtained through a headless Xvfb execution environment.

- **Category:** Create new documentation
- **Documentation Type:** Technical investigation / Architecture walkthrough / Q&A research document
- **Target File:** `blitzy/documentation/kitty_815df1e210e0.md`

The documentation requirements decompose into four investigative questions:

- **Q1 — Startup Subsystem Sequence:** What happens when Kitty starts from this commit, before the terminal is ready for a shell? Which systems start up on the way to a working terminal, and what do you actually see on screen or in logs that shows them coming online? Includes setting up a virtual framebuffer (Xvfb) for headless execution.
- **Q2 — Initial Configuration Resolution:** How does Kitty decide on its initial configuration when it starts? Which configuration sources or default settings does it use for the first launch, how do they affect what you see when the window appears, and what output during a real launch shows that those settings were applied?
- **Q3 — Terminal-to-Shell Communication:** How does Kitty get the terminal ready to communicate with the shell that will run inside it? When the shell prints its first output, what concrete behavior shows the data was understood correctly and drawn in the terminal?
- **Q4 — Display System Evidence:** As those first characters appear, what visible evidence about fonts, layout, scrolling, or how the screen updates that tells you the display system is active and working properly? Include any log or console messages that help confirm this.

Implicit documentation needs surfaced from analysis:

- A walkthrough of the full startup call chain: `entry_points.main()` → `kitty.main._main()` → GLFW init → `AppRunner.__call__()` → `_run_app()` → `Boss.start()` → `Boss.startup_first_child()` → `Child.fork()`
- Documenting the configuration cascade: `KITTY_CONFIG_DIRECTORY` → `XDG_CONFIG_HOME` → `~/.config/kitty/kitty.conf` → built-in defaults from `kitty/options/definition.py`
- Explanation of PTY allocation, environment variable injection (`TERM`, `COLORTERM`, `TERMINFO`, `KITTY_SHELL_INTEGRATION`), and shell integration hook loading
- Evidence of the GPU rendering pipeline activation: GLSL shader compilation, font discovery via FontConfig, glyph rasterization via FreeType, glyph-cache upload

### 0.1.2 Special Instructions and Constraints

- **Read-Only Constraint (Critical):** "Do not modify any of the repository source files while investigating. Temporary scripts or logs may be created if needed but must be cleaned up when finished."
- **Implementation Rule:** Create a new markdown document named `kitty_815df1e210e0.md` and place it in the `blitzy/documentation` directory in the destination repo.
- **Implementation Rule:** Provide thinking and rationale behind all answers. Do not make assumptions — base answers on the code as the source of truth.
- **Implementation Rule:** Do not modify any existing files in the source repository.
- **Xvfb Requirement:** The user explicitly requests setting up a virtual framebuffer (Xvfb) to attempt headless execution of Kitty and observe startup behavior.
- **Evidence Emphasis:** Every question asks for observable evidence — log output, console messages, or screen behavior that can be captured and cited.
- **Style:** Investigative technical writing with reasoning grounded in specific source file paths and line references.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer Q1** (startup subsystem sequence), we will trace the call chain through `kitty/entry_points.py` → `kitty/main.py` → `kitty/boss.py` → `kitty/child.py`, documenting each subsystem initialization and the observable evidence of each step. We will create a headless Xvfb environment and attempt to launch Kitty, capturing any log or error output that reveals the startup sequence.
- To **answer Q2** (initial configuration resolution), we will analyze `kitty/constants.py` (config directory resolution via `_get_config_dir()`), `kitty/config.py` (`load_config()` pipeline), `kitty/options/definition.py` (default values), and `kitty/cli.py` (`create_opts()`) to document the complete configuration cascade and which defaults apply on first launch.
- To **answer Q3** (terminal-to-shell communication), we will document `kitty/child.py` (`Child.fork()`, `get_final_env()`, `mark_terminal_ready()`), the PTY allocation sequence, environment variable injection, the VT parser activation in `kitty/vt-parser.c`, and shell integration setup via `kitty/shell_integration.py`.
- To **answer Q4** (display system evidence), we will document the GPU rendering pipeline from `kitty/shaders.py` (`load_shader_programs`), font initialization via `kitty/fonts/render.py` (`set_font_family()`), glyph caching in `kitty/glyph-cache.c`, and the six-stage GLSL shader compilation evidenced by `kitty/shaders.py`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The `_main()` function in `kitty/main.py` contains the authoritative startup sequence but has no standalone documentation explaining the subsystem ordering and dependencies.
- `kitty/constants.py` contains `_get_config_dir()` which implements a multi-location configuration search strategy that is critical to Q2 but is only documented inline.
- The `Child.get_final_env()` method in `kitty/child.py` sets 9+ environment variables for the child process, directly relevant to Q3, but no consolidated documentation exists for this.
- The `AppRunner.__call__()` method in `kitty/main.py` orchestrates `set_options()` → `set_font_family()` → `_run_app()` — the bridge between configuration activation and display, relevant to both Q2 and Q4.
- The `load_all_shaders()` function in `kitty/main.py` compiles six shader programs during window creation — this is the point where the GPU rendering pipeline comes online, directly relevant to Q4.
- The GLFW null/headless backend (`glfw/null_init.c`, `null_window.c`) may be relevant for Xvfb testing, though Kitty's standard X11 backend (`glfw/x11_init.c`) is the one exercised under Xvfb.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation tree in the `docs/` directory with comprehensive user-facing documentation, but no existing document that consolidates the startup lifecycle or answers the user's investigative questions.

- **Documentation Framework:** Sphinx (version not pinned beyond latest in `docs/requirements.txt`)
- **Documentation Generator Configuration:** `docs/conf.py` — Sphinx configuration with custom lexers, generated document writers, and man-page hooks
- **Theme:** Furo (specified in `docs/requirements.txt`)
- **Extensions:** sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild
- **Diagram Tools Detected:** Mermaid (used extensively in the technical specification), no dedicated diagram tool in `docs/requirements.txt`
- **Documentation Build System:** `docs/Makefile` with standard Sphinx targets and `develop-docs` live-preview workflow via sphinx-autobuild
- **Documentation Hosting:** Not configured in-repo; published externally at `https://sw.kovidgoyal.net/kitty/`

Existing documentation structure:

| Document | Path | Relevance to Task |
|----------|------|--------------------|
| CLI invocation | `docs/invocation.rst` | Partially relevant — documents command-line options but not startup internals |
| Configuration guide | `docs/conf.rst` | Relevant to Q2 — documents `kitty.conf` format and include directives |
| Build instructions | `docs/build.rst` | Useful for Xvfb setup context |
| Shell integration | `docs/shell-integration.rst` | Relevant to Q3 — documents shell integration features |
| Performance | `docs/performance.rst` | Relevant to Q4 — discusses rendering design |
| FAQ | `docs/faq.rst` | Potentially relevant — may contain startup troubleshooting |
| Quick start | `docs/quickstart.rst` | General user onboarding, tangential |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code relevant to each question:

- **Startup sequence (Q1):** `kitty/entry_points.py`, `kitty/main.py` (`_main()`, `_run_app()`, `AppRunner`), `kitty/boss.py` (`Boss.__init__()`, `Boss.start()`, `Boss.startup_first_child()`), `kitty/session.py` (`create_sessions()`), `kitty/child.py` (`Child.fork()`)
- **Configuration resolution (Q2):** `kitty/constants.py` (`_get_config_dir()`, `defconf`, `config_dir`), `kitty/config.py` (`load_config()`, `finalize_keys()`, `finalize_mouse_mappings()`), `kitty/options/definition.py` (default values), `kitty/cli.py` (`create_opts()`, `parse_args()`)
- **Terminal-to-shell communication (Q3):** `kitty/child.py` (`get_final_env()`, `fork()`, `mark_terminal_ready()`), `kitty/shell_integration.py` (`modify_shell_environ()`), `kitty/vt-parser.c`, `kitty/screen.c`
- **Display system (Q4):** `kitty/shaders.py` (`LoadShaderPrograms`, `Program`), `kitty/fonts/render.py` (`set_font_family()`), `kitty/fonts/common.py`, `kitty/borders.py` (`load_borders_program()`), `kitty/main.py` (`load_all_shaders()`, `init_glfw()`)
- **GLFW platform layer:** `glfw/x11_init.c`, `glfw/x11_window.c`, `kitty/glfw.c`, `kitty/constants.py` (`glfw_path()`, `is_wayland()`)

Key directories examined:

| Directory | Contents | Relevance |
|-----------|----------|-----------|
| `kitty/` | Core application tree (Python + C extensions) | Primary source of all answers |
| `kitty/options/` | Configuration schema, parsing, types | Q2 configuration resolution |
| `kitty/fonts/` | Font discovery, rendering, box drawing | Q4 display evidence |
| `kitty/launcher/` | Native C launcher binary | Q1 bootstrap entry point |
| `kitty/layout/` | Window tiling algorithms | Q4 layout evidence |
| `glfw/` | Vendored GLFW windowing layer | Q1 platform initialization |
| `shell-integration/` | Shell-specific integration scripts | Q3 shell communication |
| `docs/` | Existing Sphinx documentation | Infrastructure assessment |
| `kitty_tests/` | Test suite with test fonts | Supporting evidence |

### 0.2.3 Web Search Research Conducted

No web search was required for this task. All four questions are answerable directly from the source code at commit `815df1e21`. The codebase is the authoritative source of truth per user instructions. The technical specification sections (4.2, 4.3, 4.4, 4.5, 4.7, 5.2, 7.2) provide comprehensive architectural context that supplements the code analysis.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigative document must trace the startup lifecycle through the following modules, mapping each to the user's four questions:

**Q1 — Startup Subsystem Sequence:**

- Module: `kitty/entry_points.py`
  - Public APIs: `main()`, `setup_openssl_environment()`, `entry_points` dict, `namespaced_entry_points` dict
  - Current documentation: No dedicated startup trace document
  - Documentation needed: Walkthrough of how `main()` dispatches to `kitty.main.main()`

- Module: `kitty/main.py`
  - Public APIs: `_main()`, `_run_app()`, `AppRunner.__call__()`, `init_glfw()`, `init_glfw_module()`, `load_all_shaders()`, `set_custom_ibeam_cursor()`, `setup_environment()`
  - Current documentation: Inline code comments only
  - Documentation needed: Complete startup sequence diagram with subsystem init ordering

- Module: `kitty/boss.py`
  - Public APIs: `Boss.__init__()`, `Boss.start()`, `Boss.startup_first_child()`, `Boss.add_os_window()`
  - Current documentation: Docstrings for action-decorated methods only
  - Documentation needed: Boss initialization steps, child monitor start, session provisioning

- Module: `kitty/session.py`
  - Public APIs: `create_sessions()`, `parse_session()`, `Session`, `Tab`, `WindowSpec`
  - Current documentation: No external documentation
  - Documentation needed: Session source selection priority cascade

**Q2 — Initial Configuration Resolution:**

- Module: `kitty/constants.py`
  - Public APIs: `_get_config_dir()`, `config_dir`, `defconf`, `cache_dir()`, `is_wayland()`
  - Current documentation: Inline comments
  - Documentation needed: Config directory resolution algorithm with all fallback locations

- Module: `kitty/config.py`
  - Public APIs: `load_config()`, `finalize_keys()`, `finalize_mouse_mappings()`, `parse_config()`, `cached_values_for()`
  - Current documentation: No external documentation
  - Documentation needed: Full configuration pipeline from file to Options object

- Module: `kitty/options/definition.py`
  - Key defaults: `font_family='monospace'`, `remember_window_size='yes'`, `initial_window_width=640`, `initial_window_height=400`, `startup_session='none'`
  - Documentation needed: Critical defaults that shape first-launch appearance

- Module: `kitty/cli.py`
  - Public APIs: `parse_args()`, `create_opts()`
  - Documentation needed: How CLI overrides merge with file config

**Q3 — Terminal-to-Shell Communication:**

- Module: `kitty/child.py`
  - Public APIs: `Child.__init__()`, `Child.fork()`, `Child.get_final_env()`, `Child.mark_terminal_ready()`
  - Current documentation: Inline comments
  - Documentation needed: PTY allocation, environment injection, ready-pipe synchronization

- Module: `kitty/shell_integration.py`
  - Public APIs: `modify_shell_environ()`, `setup_bash_env()`, `setup_zsh_env()`, `setup_fish_env()`
  - Current documentation: `docs/shell-integration.rst` covers user-facing features
  - Documentation needed: Internal mechanisms — how ZDOTDIR redirect, ENV rewrite, and XDG_DATA_DIRS prepend work

**Q4 — Display System Evidence:**

- Module: `kitty/shaders.py`
  - Public APIs: `LoadShaderPrograms.__call__()`, `Program.__init__()`, `Program.compile()`
  - Current documentation: No external documentation
  - Documentation needed: Six-stage shader compilation sequence and evidence markers

- Module: `kitty/fonts/render.py`
  - Public APIs: `set_font_family()`, `font_for_family()`
  - Documentation needed: Font discovery → rasterization → cache upload pipeline

- Module: `kitty/borders.py`
  - Public APIs: `load_borders_program()`
  - Documentation needed: Border shader compilation as startup evidence

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No consolidated startup lifecycle document** — The startup flow spans six Python modules and multiple C extensions. No single document traces the complete path from process entry to shell prompt.
- **No Xvfb headless testing guide** — No documentation describes how to run Kitty under a virtual framebuffer for headless testing or CI environments.
- **No configuration-to-visual mapping** — While `docs/conf.rst` documents config options, no document explains which defaults produce the specific visual appearance seen on first launch (640×400 cells, monospace font, default colors).
- **No PTY/shell handshake documentation** — The internal mechanics of PTY allocation, the ready-pipe synchronization, and environment variable injection are only documented in code comments within `kitty/child.py`.
- **No shader compilation evidence guide** — The six-stage GLSL shader pipeline compiles during `create_os_window()`, but no document captures what log output or debug flags reveal this process.
- **No first-output rendering trace** — No documentation traces what happens when the shell's first prompt bytes arrive at the VT parser, get parsed, update the screen model, and trigger a GPU render cycle.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The target document `blitzy/documentation/kitty_815df1e210e0.md` will follow this structure:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction (purpose, commit, constraints)
        ├── Xvfb Headless Environment Setup
        │   ├── Rationale for Xvfb
        │   ├── Setup procedure
        │   └── Observed behavior / limitations
        ├── Q1: Startup Subsystem Sequence
        │   ├── Entry point dispatch
        │   ├── Python bootstrap (_main)
        │   ├── GLFW platform initialization
        │   ├── Font initialization
        │   ├── OS window creation & shader compilation
        │   ├── Boss controller + child monitor
        │   ├── Session creation & child spawn
        │   └── Observable evidence (logs, debug output)
        ├── Q2: Initial Configuration Resolution
        │   ├── Config directory search algorithm
        │   ├── Config file loading pipeline
        │   ├── Default values that shape first launch
        │   ├── CLI override mechanics
        │   └── Observable evidence of config application
        ├── Q3: Terminal-to-Shell Communication
        │   ├── PTY allocation and child fork
        │   ├── Environment variable injection
        │   ├── Shell integration setup
        │   ├── Ready-pipe synchronization
        │   ├── VT parser activation and first bytes
        │   └── Observable evidence of correct data flow
        ├── Q4: Display System Evidence
        │   ├── Font discovery and rasterization
        │   ├── Glyph cache and texture upload
        │   ├── Shader pipeline activation
        │   ├── Cell rendering and cursor drawing
        │   ├── Layout and border system
        │   └── Observable evidence (debug flags, log markers)
        └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract the startup call chain from `kitty/entry_points.py` → `kitty/main.py` → `kitty/boss.py` using direct code reading"
- "Document the configuration cascade from `kitty/constants.py:_get_config_dir()` and `kitty/config.py:load_config()`"
- "Trace the PTY allocation from `kitty/child.py:Child.fork()` including ready-pipe and environment assembly"
- "Map the shader compilation sequence from `kitty/shaders.py:LoadShaderPrograms.__call__()` including all six stages"
- "Attempt headless launch under Xvfb and capture any stderr/stdout output that reveals startup steps"
- "Use `--debug-rendering` and `--debug-font-fallback` CLI flags (documented in `kitty/cli.py`) to extract additional diagnostic output"

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for startup flow and configuration cascade
- Code references using `Source: /path/to/file.py:LineNumber` format
- Tables for environment variables, configuration defaults, and shader stages
- Consistent terminology aligned with Kitty source: "OS window" (not "window"), "tab" (not "pane"), "child process" (not "subprocess")

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the target document:

- **Startup sequence flowchart** — From `entry_points.main()` through each subsystem initialization to `Boss.child_monitor.main_loop()`, showing the ordered steps and what each one produces
- **Configuration cascade diagram** — The priority order of config sources: `KITTY_CONFIG_DIRECTORY` → `XDG_CONFIG_HOME` → `~/.config/kitty/` → macOS Preferences → `XDG_CONFIG_DIRS` → built-in defaults
- **PTY/shell communication sequence** — `Child.fork()` → PTY open → environment assembly → `spawn()` → ready-pipe close → VT parser activation → first bytes rendered
- **GPU rendering pipeline diagram** — Font discovery → FreeType rasterization → glyph cache → six shader stages → GLFW swap buffers

These diagrams are derived from the source code's actual call sequences and will be validated against the code at commit `815df1e21`.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/entry_points.py`, `kitty/main.py`, `kitty/boss.py`, `kitty/child.py`, `kitty/config.py`, `kitty/constants.py`, `kitty/session.py`, `kitty/shaders.py`, `kitty/shell_integration.py`, `kitty/fonts/render.py`, `kitty/fonts/common.py`, `kitty/borders.py`, `kitty/os_window_size.py`, `kitty/debug_config.py`, `kitty/options/definition.py`, `kitty/cli.py`, `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/child-monitor.c`, `kitty/glyph-cache.c` | Complete investigative research document answering all four questions about Kitty startup lifecycle, configuration resolution, shell communication, and display system evidence, including Xvfb headless setup procedure, Mermaid diagrams, source code citations, and thinking/rationale for each answer |

No existing files are modified. No files are deleted. No other documentation files are created.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Q&A Research Document
Source Code: 20+ source files across kitty/, kitty/fonts/, kitty/options/, glfw/
Sections:
    - Introduction (purpose, commit SHA, branch, constraints)
    - Xvfb Headless Environment Setup
        - Procedure for setting up Xvfb
        - Attempt to launch Kitty headlessly
        - Observed behavior and any limitations
    - Q1: Startup Subsystem Sequence
        - Entry point dispatch (kitty/entry_points.py:main)
        - Python bootstrap (kitty/main.py:_main)
        - Signal masking and GLFW init (kitty/main.py:init_glfw)
        - Font system init (kitty/fonts/render.py:set_font_family)
        - OS window creation with shader compilation
        - Boss controller creation (kitty/boss.py:Boss.__init__)
        - Child monitor start and session provisioning
        - Observable startup evidence
    - Q2: Configuration Resolution
        - Config directory search (kitty/constants.py:_get_config_dir)
        - Config file loading pipeline (kitty/config.py:load_config)
        - Key defaults from definition.py
        - Cached window size and remember_window_size
        - CLI override mechanics
        - Observable config application evidence
    - Q3: Terminal-to-Shell Communication
        - PTY allocation and Child.fork()
        - Environment variable injection (get_final_env)
        - Shell integration env modification
        - Ready-pipe synchronization protocol
        - VT parser first bytes and screen update
        - Observable shell communication evidence
    - Q4: Display System Evidence
        - Font discovery (FontConfig/CoreText)
        - Glyph rasterization (FreeType + HarfBuzz)
        - Glyph cache GPU upload
        - Six-stage shader pipeline activation
        - Cell rendering, cursor, and border drawing
        - Debug flags for rendering evidence
    - Summary
Diagrams:
    - Startup sequence flowchart (Mermaid)
    - Configuration cascade diagram (Mermaid)
    - PTY/shell handshake sequence (Mermaid)
    - GPU rendering pipeline diagram (Mermaid)
Key Citations:
    kitty/entry_points.py, kitty/main.py, kitty/boss.py, kitty/child.py,
    kitty/config.py, kitty/constants.py, kitty/session.py, kitty/shaders.py,
    kitty/shell_integration.py, kitty/fonts/render.py, kitty/borders.py,
    kitty/os_window_size.py, kitty/options/definition.py, kitty/cli.py,
    kitty/debug_config.py, kitty/vt-parser.c, kitty/screen.c,
    kitty/child-monitor.c, kitty/glyph-cache.c, kitty/fonts/common.py
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration changes are required. The target document is a standalone Markdown file placed in `blitzy/documentation/`, which is outside the Sphinx documentation tree in `docs/`. No changes to `docs/conf.py`, `docs/Makefile`, or `docs/requirements.txt` are needed.

### 0.5.4 Cross-Documentation Dependencies

- The new document references source files but does not create inter-document links to the existing Sphinx documentation in `docs/`.
- No table-of-contents, navigation, or index updates are required in any existing documentation file.
- No shared content or include directives are involved.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No documentation generation tools are required for this task. The output is a single standalone Markdown file (`blitzy/documentation/kitty_815df1e210e0.md`) that does not require Sphinx, MkDocs, or any build system.

However, the following runtime dependencies are relevant to the **investigative procedure** (Xvfb headless execution attempt):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| apt | xvfb | System default | Virtual framebuffer for headless X11 display |
| apt | x11-utils | System default | `xdpyinfo` for verifying X11 display availability |
| pip | sphinx | Latest in docs/requirements.txt | Existing docs infrastructure (reference only, not modified) |
| pip | furo | Latest in docs/requirements.txt | Existing docs theme (reference only, not modified) |

The Kitty project itself requires the following for building (relevant context for Xvfb testing):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| system | Python | ≥ 3.8 (pyproject.toml) | Python runtime for Kitty |
| system | Go | 1.22 (go.mod) | Go runtime for kitten binary |
| system | libGL / Mesa | System default | OpenGL rendering backend |
| system | FreeType | System default | Font rasterization |
| system | HarfBuzz | ≥ 1.5 | Text shaping |
| system | FontConfig | System default | Font discovery on Linux |
| system | GLFW 3.4 | Vendored in glfw/ | Platform windowing (customized fork) |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new document is standalone and does not create or modify any cross-references in existing documentation files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

Coverage is measured against the user's four explicit questions and their sub-components:

| Question | Sub-Components | Target Coverage |
|----------|---------------|-----------------|
| Q1: Startup Subsystem Sequence | Entry point dispatch, Python bootstrap, GLFW init, font init, OS window creation, shader compilation, Boss init, child monitor start, session creation, child spawn, Xvfb setup | 100% — all 11 sub-components documented |
| Q2: Configuration Resolution | Config directory search, file loading pipeline, default values, CLI overrides, cached window size, observable evidence | 100% — all 6 sub-components documented |
| Q3: Terminal-to-Shell Communication | PTY allocation, environment injection, shell integration, ready-pipe sync, VT parser first bytes, observable evidence | 100% — all 6 sub-components documented |
| Q4: Display System Evidence | Font discovery, glyph rasterization, glyph cache, shader pipeline, cell rendering, layout/borders, debug output | 100% — all 7 sub-components documented |

Source file citation coverage:

- Primary source files to cite: 20 files (listed in Section 0.5.2)
- Each answer must reference specific file paths and function names
- Code-level rationale must be provided for every claim per user instructions

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**

- Every answer must trace the relevant code path from entry point to observable behavior
- Each subsystem mentioned must include the source file path and key function name
- Mermaid diagrams must accurately reflect the actual call sequences in the code
- The Xvfb section must include the setup procedure, the command attempted, and whatever output was observed (success or failure)

**Accuracy Validation:**

- All code references must be validated against the actual source at commit `815df1e21`
- Function signatures and call chains must match what the code actually does, not what documentation or comments claim
- Default configuration values must be verified against `kitty/options/definition.py`
- Environment variables must be verified against `kitty/child.py:get_final_env()`

**Clarity Standards:**

- Technical accuracy with accessible explanations of internal mechanisms
- Progressive disclosure: start with high-level flow, then drill into implementation details
- Thinking and rationale provided for each answer as required by user instructions
- Consistent use of Kitty's own terminology (OS window, tab, child process, screen model)

**Maintainability:**

- Source citations as `Source: path/to/file.py:FunctionName` for traceability
- Commit SHA documented in the introduction for version pinning
- Clear separation between code-derived facts and runtime observations

### 0.7.3 Example and Diagram Requirements

- Minimum diagrams: 4 (one per question area)
- Code snippet examples: Short inline references (2-3 lines max) showing key function calls or config defaults
- Debug flag documentation: `--debug-rendering`, `--debug-font-fallback`, `--debug-keyboard` flags and expected output
- Runtime evidence: Any stderr/stdout captured from Xvfb launch attempts

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — the single deliverable document

**Source files analyzed (read-only) to produce the document:**
- `kitty/entry_points.py` — entry point dispatch logic
- `kitty/main.py` — `_main()`, `_run_app()`, `AppRunner`, `init_glfw()`, `load_all_shaders()`, `setup_environment()`
- `kitty/boss.py` — `Boss.__init__()`, `Boss.start()`, `Boss.startup_first_child()`, `Boss.add_os_window()`
- `kitty/child.py` — `Child.__init__()`, `Child.fork()`, `Child.get_final_env()`, `Child.mark_terminal_ready()`
- `kitty/config.py` — `load_config()`, `finalize_keys()`, `finalize_mouse_mappings()`, `cached_values_for()`
- `kitty/constants.py` — `_get_config_dir()`, `config_dir`, `defconf`, `glfw_path()`, `is_wayland()`
- `kitty/session.py` — `create_sessions()`, `parse_session()`, `get_os_window_sizing_data()`
- `kitty/shaders.py` — `LoadShaderPrograms.__call__()`, `Program`, `program_for()`
- `kitty/shell_integration.py` — `modify_shell_environ()`, `setup_bash_env()`, `setup_zsh_env()`, `setup_fish_env()`
- `kitty/fonts/render.py` — `set_font_family()`, `font_for_family()`
- `kitty/fonts/common.py` — Platform-specific font abstraction
- `kitty/borders.py` — `load_borders_program()`
- `kitty/os_window_size.py` — `initial_window_size_func()`, `edge_spacing()`
- `kitty/debug_config.py` — Diagnostic output formatting
- `kitty/options/definition.py` — Configuration defaults (font_family, window size, colors)
- `kitty/cli.py` — `parse_args()`, `create_opts()`
- `kitty/window.py` — Window lifecycle and screen attachment
- `kitty/tabs.py` — `TabManager.__init__()`, `Tab.__init__()`
- `kitty/vt-parser.c` — VT parser state machine (analyzed via summary)
- `kitty/screen.c` — Screen model operations (analyzed via summary)
- `kitty/child-monitor.c` — Child monitor threading (analyzed via summary)
- `kitty/glyph-cache.c` — Glyph cache management (analyzed via summary)
- `kitty/freetype.c` — FreeType glyph rasterization (analyzed via summary)

**Investigative procedures in scope:**
- Setting up Xvfb virtual framebuffer
- Attempting headless Kitty launch and capturing output
- Creating temporary test scripts (to be cleaned up)
- Using debug CLI flags (`--debug-rendering`, `--debug-font-fallback`)

**Directory creation:**
- `blitzy/documentation/` — to house the deliverable

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — User explicitly states: "Do not modify any of the repository source files while investigating"
- **Existing documentation updates** — No changes to `docs/**/*.rst` or any other existing documentation
- **Test file modifications** — No changes to `kitty_tests/**`
- **Build system changes** — No changes to `setup.py`, `Makefile`, `pyproject.toml`
- **Feature additions or code refactoring** — Investigation only
- **macOS-specific paths** — Analysis focuses on Linux/X11 path since Xvfb is an X11 tool; macOS Cocoa and Wayland paths are mentioned for completeness but not the primary investigation target
- **Kittens documentation** — Not part of the startup lifecycle questions
- **Remote control system** — Not part of the startup lifecycle questions
- **Post-startup features** — Scrollback, clipboard, file transfer, graphics protocol, etc. are beyond the "before the terminal is ready for a shell" scope
- **Deployment configuration** — Not applicable to documentation investigation
- **Any files matching patterns that would appear in .blitzyignore** — None found in this repository

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not a Sphinx build
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and rendered by any Mermaid-compatible viewer
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files using `Source: path/to/file.py:FunctionName` notation
- **Style guide:** Investigative technical writing with reasoning/rationale for each answer. Code is the source of truth. No assumptions.

### 0.9.2 Investigative Procedure Commands

The following commands are relevant to the Xvfb headless investigation:

- **Xvfb setup:** `Xvfb :99 -screen 0 1280x1024x24 &` followed by `export DISPLAY=:99`
- **Kitty launch attempt (if buildable):** `python3 -c "from kitty.main import main; main()"` or `./kitty/launcher/kitty` under Xvfb
- **Debug rendering:** `kitty --debug-rendering` — enables GPU rendering debug output
- **Debug font fallback:** `kitty --debug-font-fallback` — enables font fallback diagnostic logging
- **Config debug:** `kitty --debug-config` — shows effective configuration after all sources are merged
- **Cleanup requirement:** All temporary scripts and log files must be removed after the investigation per user instructions

### 0.9.3 Validation Approach

- **Code tracing:** Every startup step documented is traced to a specific function and file in the source
- **Cross-referencing:** Claims are cross-referenced against the tech spec sections 4.2, 4.3, 4.4, 4.5, 4.7 for consistency
- **Default verification:** Configuration defaults are verified against `kitty/options/definition.py` source
- **Environment variable verification:** Environment variables are verified against `kitty/child.py:get_final_env()` source

## 0.10 Rules for Documentation

The following rules are derived from user-specified directives and implementation constraints:

- **Do not modify any repository source files.** The investigation is read-only. Only the new file `blitzy/documentation/kitty_815df1e210e0.md` is created.
- **Temporary scripts or logs may be created if needed but must be cleaned up when finished.** Any Xvfb processes, temporary scripts, or log files created during investigation must be terminated and removed.
- **Provide thinking and rationale behind the answers.** Every answer must include the reasoning that led to the conclusion, grounded in specific source code evidence.
- **Do not make assumptions — base answers on the code as the truth.** All claims must be traceable to actual source files at commit `815df1e21`.
- **Create the document as `kitty_815df1e210e0.md`.** The filename matches the source branch name per the implementation rule.
- **Place the document in `blitzy/documentation/`.** The directory must be created if it does not exist.
- **Do not modify any existing files in the source repository.** This includes `docs/`, `kitty/`, `kitty_tests/`, `setup.py`, `Makefile`, and all other repository files.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were systematically examined to derive the conclusions in this Agent Action Plan:

**Core startup chain (Q1):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `__main__.py` | Root entry point — delegates to `kitty.entry_points.main()` |
| `kitty/entry_points.py` | Dispatcher: legacy entry points, namespaced commands, default GUI path to `kitty.main.main()` |
| `kitty/main.py` | Primary startup orchestration: `_main()`, `_run_app()`, `AppRunner`, `init_glfw()`, `load_all_shaders()`, `setup_environment()`, `set_locale()` |
| `kitty/boss.py` | Boss controller: `__init__()`, `start()`, `startup_first_child()`, `add_os_window()`, `ChildMonitor` creation |
| `kitty/session.py` | Session creation: `create_sessions()`, `parse_session()`, `get_os_window_sizing_data()` |

**Configuration resolution (Q2):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `kitty/constants.py` | Config directory search: `_get_config_dir()`, `config_dir`, `defconf`, `cache_dir()`, `glfw_path()`, `is_wayland()` |
| `kitty/config.py` | Config loading pipeline: `load_config()`, `parse_config()`, `finalize_keys()`, `finalize_mouse_mappings()`, `cached_values_for()` |
| `kitty/options/definition.py` | Authoritative default values: `font_family`, `initial_window_width/height`, `remember_window_size`, `startup_session` |
| `kitty/cli.py` | CLI parsing: `parse_args()`, `create_opts()` |
| `kitty/conf/utils.py` | Config parsing base: `parse_config_base()`, `BadLine`, `resolve_config()` |
| `kitty/os_window_size.py` | Window sizing: `initial_window_size_func()`, `WindowSizeData`, `edge_spacing()` |
| `kitty/debug_config.py` | Config diagnostics: `compare_opts()`, font info, OpenGL version reporting |
| `pyproject.toml` | Python version constraint: `requires-python = ">=3.8"` |

**Terminal-to-shell communication (Q3):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `kitty/child.py` | Child process management: `Child.__init__()`, `fork()`, `get_final_env()`, `mark_terminal_ready()`, PTY allocation, environment injection |
| `kitty/shell_integration.py` | Shell environment modification: `modify_shell_environ()`, `setup_bash_env()`, `setup_zsh_env()`, `setup_fish_env()` |
| `kitty/tabs.py` | Tab/window lifecycle: `TabManager.__init__()`, `Tab.__init__()` |
| `kitty/window.py` | Window lifecycle: Screen attachment, focus tracking, render data setup |

**Display system (Q4):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `kitty/shaders.py` | Shader management: `LoadShaderPrograms.__call__()`, `Program.__init__()`, `Program.compile()`, six shader stages |
| `kitty/fonts/render.py` | Font initialization: `set_font_family()`, `font_for_family()` |
| `kitty/fonts/common.py` | Platform font abstraction: FontConfig vs CoreText branching |
| `kitty/borders.py` | Border shader: `load_borders_program()` |
| `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl` | Cell rendering shaders (identified, not read for content) |
| `kitty/border_vertex.glsl`, `kitty/border_fragment.glsl` | Border shaders (identified) |
| `kitty/bgimage_vertex.glsl`, `kitty/bgimage_fragment.glsl` | Background image shaders (identified) |
| `kitty/graphics_vertex.glsl`, `kitty/graphics_fragment.glsl` | Graphics protocol shaders (identified) |
| `kitty/tint_vertex.glsl`, `kitty/tint_fragment.glsl` | Tint overlay shaders (identified) |
| `kitty/alpha_blend.glsl`, `kitty/linear2srgb.glsl` | Utility shaders (identified) |

**Infrastructure and documentation:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `docs/` (folder) | Existing documentation infrastructure assessment |
| `docs/requirements.txt` | Sphinx dependencies: sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild |
| `docs/conf.py` | Sphinx configuration assessment |
| `docs/conf.rst` | Configuration documentation (existing) |
| `docs/invocation.rst` | CLI documentation (existing) |
| `go.mod` | Go module: Go 1.22 |
| `setup.py` | Build system: version extraction, platform detection |
| `kitty_tests/` (folder) | Test suite structure assessment |

**Technical specification sections consulted:**

| Section | Title | Relevance |
|---------|-------|-----------|
| 4.2 | APPLICATION STARTUP FLOW | Startup sequence, bootstrap, Boss initialization |
| 4.3 | TERMINAL INPUT/OUTPUT PIPELINE | VT parser, GPU rendering pipeline, keyboard/mouse input |
| 4.4 | CONFIGURATION MANAGEMENT FLOW | Config loading, parsing, live reload |
| 4.5 | WINDOW AND TAB LIFECYCLE | Session creation, child process launch, window states |
| 4.7 | SHELL INTEGRATION FLOW | Shell environment setup, Bash/Zsh/Fish integration |
| 5.2 | COMPONENT DETAILS | All major subsystems: launcher, Boss, child monitor, VT parser, GPU pipeline, fonts, GLFW, config, layout, kittens, shell integration |
| 7.2 | GPU Rendering Pipeline | Shader architecture, font rendering pipeline, performance architecture |

### 0.11.2 Attachments

No attachments were provided for this project. No Figma designs, external URLs, or supplementary files were supplied.

