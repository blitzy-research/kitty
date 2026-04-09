# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new exploratory documentation** that answers a set of interconnected architectural questions about the kitty terminal emulator's internal design — specifically the interplay between Python, C, and GLSL, the centrality of the native bridge module, and the true modularity of the kittens subsystem. The user is onboarding into the codebase and seeks evidence-based answers grounded in direct observation of code behavior, not in assumptions about intended architecture.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Architecture exploration / codebase Q&A document
- **Target Audience:** A developer onboarding into the kitty codebase who needs to understand architectural relationships at a fundamental level

The user's requirements decompose into five distinct investigative questions:

- **Q1 — Language Role Distribution:** Which language (Python vs. C) does the runtime heavy lifting, and what does that reveal about where performance actually comes from? The codebase has ~35,155 lines of C in `kitty/*.c`, ~20,647 lines of Python in `kitty/*.py`, ~193 Go files in `tools/`, and 13 GLSL shader files — the documentation must analyze what each layer contributes at runtime.
- **Q2 — Shader Centrality:** What role do the 13 GLSL files in `kitty/` play, and how central are they to the terminal emulator? The user explicitly calls this unexpected for a terminal application.
- **Q3 — Entry Point Failure and the Native Bridge:** When running `python3 __main__.py` directly, it fails with `ModuleNotFoundError: No module named 'kitty.fast_data_types'`. What is missing, and what does the failure reveal about how Python is wired into the native C core?
- **Q4 — Kittens Independence:** Are kittens truly self-contained tools, or do they depend on the same native bridge? What happens when a kitten is run standalone?
- **Q5 — Observational Method:** The user wants answers derived from exercising the code and observing actual behavior, not from reading documentation about intended architecture.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — Repository Immutability:** "The repository itself should remain unchanged." No files in the source repository may be modified. Any temporary scripts created for observation must be cleaned up afterward.
- **CRITICAL — Implementation Rule:** The user's project rule (`SWE-AtlasQnA-Repo`) specifies: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt. Place it in the `blitzy/documentation` directory.
- **Evidence-Based Answers:** "I want to explore these questions by observing how the code behaves when exercised, rather than assuming how the architecture is meant to work."
- **Rationale Required:** Provide thinking and rationale behind each answer.
- **No Assumptions:** Base all answers on the code as the source of truth.
- **Temporary Scripts:** May be used for observation but must be cleaned up afterward.
- **Document Style:** Markdown, comprehensive, with rationale behind each answer.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer the language role distribution question**, we will analyze the call chain from `__main__.py` → `kitty/entry_points.py` → `kitty/main.py`, enumerate the C extension modules compiled into `fast_data_types` (defined in `kitty/data-types.c`), and trace which performance-critical subsystems (VT parsing, screen model, rendering, font rasterization) are in C versus which orchestration tasks (configuration, layout, kittens framework, remote control) are in Python.
- To **answer the shader centrality question**, we will examine the 13 GLSL files, their loading mechanism in `kitty/shaders.py`, their compilation through `kitty/shaders.c`, and demonstrate that there is no CPU-based rendering fallback — all visual output passes through the GPU shader pipeline.
- To **document the entry point failure**, we will execute `python3 __main__.py` and trace the import chain that terminates at the missing `fast_data_types` C extension, explaining that this module is the single critical bridge compiled by `setup.py` that bundles all native subsystems (screen, fonts, GLFW, OpenGL, child monitor, etc.) into a single `.so` file.
- To **assess kittens independence**, we will attempt to invoke a kitten standalone and trace how it fails at the same `fast_data_types` boundary, revealing that kittens are not independent — they are deeply coupled to the native core through the TUI framework (`kittens/tui/loop.py`) and through their direct imports of `kitty.fast_data_types`.
- To **create the output document**, we will generate `blitzy/documentation/kitty_815df1e210e0.md` containing all answers with code evidence and rationale.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs emerge:

- **Architecture Diagram:** A visual representation of how the three language layers (C, Python, GLSL) relate at runtime, showing the `fast_data_types` bridge as the central integration point.
- **Build Dependency Explanation:** The user's entry point failure implies a need to document that kitty requires a compilation step (`python3 setup.py`) before any Python code can function, because the C extension is not a development-time convenience — it is a runtime necessity.
- **Kittens Dependency Chain:** Each kitten's import graph terminates at `fast_data_types`, and this should be documented with concrete traces showing the import chain from a kitten through `kittens/tui/loop.py` → `kitty/fast_data_types`.
- **Shader Pipeline Documentation:** A description of the six shader stages (cell, border, background image, graphics, tint, utility) and how `kitty/shaders.py` loads GLSL source from disk, applies compile-time macro substitutions, and compiles them via `kitty/shaders.c` into OpenGL programs.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx-based documentation infrastructure with comprehensive user-facing manuals, but no existing architecture exploration document of the type the user requires.

- **Documentation Framework:** Sphinx (reStructuredText), with the Furo theme
- **Documentation Generator Configuration:** `docs/conf.py` (central Sphinx config); `docs/Makefile` (build driver with `html`, `man`, `dirhtml`, `develop-docs` targets)
- **Documentation Dependencies:** Defined in `docs/requirements.txt`:
  - `sphinx`
  - `furo`
  - `sphinx-copybutton`
  - `sphinxext-opengraph`
  - `sphinx-inline-tabs`
  - `sphinx-autobuild`
- **Diagram Tools Detected:** Mermaid (used in the tech spec, not natively in the Sphinx docs); GLSL-based diagrams are code, not visual docs
- **Documentation Hosting:** Published to `https://sw.kovidgoyal.net/kitty/` via `publish.py`
- **API Documentation Tools:** None detected for auto-generating Python or C API docs (no Sphinx autodoc, JSDoc, etc.). Documentation is hand-authored in `.rst` files.

**Existing Documentation Files Examined:**

| File/Directory | Type | Content | Relevance |
|---|---|---|---|
| `README.asciidoc` | AsciiDoc | Project introduction with links to website, FAQ, CI | Low — external pointer only |
| `CONTRIBUTING.md` | Markdown | Contributor workflows for bug reporting and submissions | Low — process, not architecture |
| `INSTALL.md` | Markdown | Points to source-build and binary-install docs | Medium — references build process |
| `docs/overview.rst` | RST | Project overview in Sphinx docs | Medium — describes project goals |
| `docs/build.rst` | RST | Building from source documentation | High — explains build process |
| `docs/performance.rst` | RST | Performance tuning documentation | Medium — discusses rendering pipeline |
| `docs/conf.py` | Python | Sphinx configuration module | Low — build infra only |
| `docs/kittens/` | Directory | Individual kitten manuals (icat, diff, ssh, themes, etc.) | Medium — documents kitten usage, not internals |
| `docs/kittens_intro.rst` | RST | Introduction to the kittens framework | Medium — overview of kittens concept |

**Key Finding:** No existing document in the repository addresses the user's specific architectural questions about language role distribution, shader centrality, the `fast_data_types` bridge, or kittens' dependency on the native core. The existing documentation is user-facing (how to use kitty) rather than developer-facing (how kitty works internally).

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code requiring documentation:

- **Native Bridge Module:** `kitty/data-types.c` — defines the `fast_data_types` CPython extension module with `PyInit_fast_data_types()` at line 525. This single module aggregates 25+ C subsystem initializers including `init_Screen`, `init_glfw`, `init_shaders`, `init_fonts`, `init_child_monitor`, `init_graphics`, and more.
- **Entry Points:** `__main__.py` → `kitty/entry_points.py` → `kitty/main.py` — the startup chain that immediately depends on `fast_data_types`.
- **Shader Pipeline:** 13 GLSL files in `kitty/`, loaded by `kitty/shaders.py` (Python orchestration) and compiled via `kitty/shaders.c` (C OpenGL calls).
- **Build System:** `setup.py` (2,173 lines) — compiles all C files into `kitty/fast_data_types.so` at line 1091.
- **Kittens Framework:** `kittens/runner.py` (discovery/launch), `kittens/tui/loop.py` (TUI foundation importing `fast_data_types`), and 20 kitten subpackages.
- **Launcher:** `kitty/launcher/main.c` — C-based process entry that embeds CPython.
- **Type Stubs:** `kitty/fast_data_types.pyi` — the typing facade documenting the full surface area of the C extension.

**Key Directories Examined:**
- `kitty/` — Core application tree (C, Python, GLSL, headers)
- `kittens/` — 20 kitten subpackages plus `runner.py` and `tui/`
- `kitty/launcher/` — Native C launcher (3 files)
- `docs/` — Sphinx documentation tree
- `setup.py` — Central build orchestration
- `tools/` — Go CLI tools (193 Go files)

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. The user's questions are entirely answerable from direct codebase observation and behavioral experiments. The existing tech spec sections (1.1, 3.1, 5.1) provide sufficient architectural context validated against the source code.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules, files, and subsystems must be analyzed and documented to answer the user's five questions comprehensively:

**Q1 — Language Role Distribution: Modules Requiring Analysis**

- Module: `kitty/data-types.c` (612 lines)
  - Public APIs: `PyInit_fast_data_types()`, 25+ module methods (`wcwidth`, `open_tty`, `raw_tty`, `monotonic`, `base64_encode/decode`, etc.)
  - Current documentation: No internal architecture documentation exists
  - Documentation needed: Explanation of what this module bundles, line counts by language (C: 35,155 lines, Python: 20,647 lines, Go: 193 files, GLSL: 13 files), and analysis of which language owns which runtime subsystem

- Module: `kitty/screen.c` (4,932 lines — largest C file)
  - Public APIs: Screen manipulation functions exposed to Python
  - Current documentation: None for internals
  - Documentation needed: Evidence that the terminal's screen model is entirely C-native

- Module: `kitty/boss.py` (3,094 lines — largest Python file)
  - Public APIs: Central lifecycle coordinator
  - Current documentation: None for internals
  - Documentation needed: Evidence that Python orchestrates rather than computes

- Module: `kitty/child-monitor.c` (2,016 lines)
  - Public APIs: PTY I/O multiplexing, child process management
  - Current documentation: None for internals
  - Documentation needed: Evidence that the I/O hot path is in C

**Q2 — Shader Centrality: Modules Requiring Analysis**

- Module: `kitty/*.glsl` (13 files)
  - Shader programs: cell (vertex/fragment), border (vertex/fragment), bgimage (vertex/fragment), graphics (vertex/fragment), tint (vertex/fragment), alpha_blend, linear2srgb, cell_defines
  - Current documentation: Inline comments only
  - Documentation needed: Role of each shader, the compile-time macro substitution system, and evidence that shaders are the sole rendering path

- Module: `kitty/shaders.py` (205 lines)
  - Public APIs: `LoadShaderPrograms`, `Program` class, `compile_program`
  - Current documentation: None
  - Documentation needed: How GLSL source is loaded, preprocessed (with `{PLACEHOLDER}` macro substitution), and compiled via OpenGL

- Module: `kitty/shaders.c` (1,285 lines)
  - Public APIs: `compile_program()`, `init_cell_program()`, shader uniform management
  - Current documentation: None
  - Documentation needed: C-side OpenGL shader compilation mechanics

**Q3 — Entry Point Failure: Modules Requiring Analysis**

- Module: `__main__.py` (7 lines)
  - Call chain: `main()` → `kitty.entry_points.main()`
  - Documentation needed: Trace the exact import failure

- Module: `kitty/entry_points.py` (198 lines)
  - Call chain: Falls through to `kitty.main.main()` for default invocation
  - Documentation needed: How the dispatch works and where it hits the native wall

- Module: `kitty/main.py` (531 lines)
  - Import chain: Line 11 imports from `.borders` → which imports from `.fast_data_types`
  - Documentation needed: The exact traceback showing the failure at the `fast_data_types` boundary

- Module: `setup.py` (line 1091)
  - Build command: `compile_c_extension(kitty_env(args), 'kitty/fast_data_types', ...)`
  - Documentation needed: How and why the C extension must be built before any Python code can run

**Q4 — Kittens Independence: Modules Requiring Analysis**

- Module: `kittens/runner.py` (203 lines)
  - Import chain: Line 14 imports `kitty.utils` → which imports `kitty.fast_data_types`
  - Documentation needed: Evidence that even the kitten launcher itself depends on the native bridge

- Module: `kittens/tui/loop.py`
  - Import: Line 19 imports directly from `kitty.fast_data_types` (`open_tty`, `raw_tty`, `close_tty`, `parse_input_from_terminal`)
  - Documentation needed: The TUI foundation's hard dependency on C functions for terminal I/O

- Module: Multiple kitten `main.py` files
  - Pattern: `kittens/diff/main.py:14` → `raise SystemExit('Must be run as kitten diff')`
  - Pattern: `kittens/icat/main.py:172` → `raise SystemExit('This should be run as kitten icat')`
  - Documentation needed: Evidence that kittens enforce they must be run inside the kitty framework, not standalone

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented Architecture Internals:** No document in the repository explains the runtime relationship between Python, C, and GLSL at an architectural level. The existing docs are user-focused (how to configure, how to use kittens) rather than contributor-focused (how the internals work).
- **Missing `fast_data_types` Documentation:** Despite being the single most critical module in the entire codebase (the bridge that makes everything work), `fast_data_types` has zero internal documentation beyond the `.pyi` type stub.
- **Missing Shader Pipeline Walkthrough:** The 13 GLSL files are undocumented beyond inline comments. No document explains the six shader stages or the compile-time macro substitution system.
- **Missing Kittens Dependency Analysis:** No existing document addresses whether kittens are truly modular or how deeply they depend on the native core.
- **Missing Build Prerequisite Documentation:** While `docs/build.rst` and `INSTALL.md` cover building from source, no document explains *why* the build is necessary from an architectural perspective — i.e., that the Python code is structurally incapable of running without the compiled C extension.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the user-specified rule: a single markdown file `kitty_815df1e210e0.md` in `blitzy/documentation/`. The document will be structured to address each question as a self-contained section, with evidence, rationale, and diagrams.

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction and Approach
        ├── Q1: Language Role Distribution at Runtime
        │   ├── Quantitative Code Breakdown
        │   ├── What C Owns (Performance-Critical Hot Paths)
        │   ├── What Python Owns (Orchestration Layer)
        │   ├── What Go Owns (CLI Tooling)
        │   └── Architectural Diagram
        ├── Q2: The Role of GLSL Shaders in a Terminal
        │   ├── Inventory of Shader Files
        │   ├── How Shaders Are Loaded and Compiled
        │   ├── The Six Rendering Stages
        │   └── Why There Is No CPU Fallback
        ├── Q3: The Entry Point Failure and the Native Bridge
        │   ├── Reproducing the Failure
        │   ├── Tracing the Import Chain
        │   ├── What fast_data_types Actually Contains
        │   ├── Why the Build Step Is a Runtime Necessity
        │   └── The Launcher Bypass
        ├── Q4: Kittens — Modularity vs. Native Dependency
        │   ├── Attempting to Run a Kitten Standalone
        │   ├── The Import Dependency Chain
        │   ├── Kittens That Explicitly Reject Standalone Execution
        │   └── The TUI Foundation's Hard Dependency
        └── Conclusions and Architectural Insights
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract the full `PyInit_fast_data_types()` initialization sequence from `kitty/data-types.c` (lines 525–612) to catalog every C subsystem bundled into the native bridge."
- "Execute `python3 __main__.py` and `python3 -c 'from kittens.runner import run_kitten; run_kitten("diff")'` to capture actual tracebacks that demonstrate the native bridge dependency."
- "Enumerate all 13 GLSL files with their line counts and roles from direct file examination."
- "Trace the import chain from `kittens/tui/loop.py` line 19 through `kitty.fast_data_types` to demonstrate the hard native dependency."
- "Count lines of code by language using `wc -l` on all `.c`, `.py`, `.go`, `.glsl` files to provide quantitative evidence."
- "Grep all kittens for `raise SystemExit` patterns to catalog which kittens explicitly reject standalone execution."

**Documentation Standards:**

- Markdown formatting with `#`, `##`, `###` heading hierarchy
- Mermaid diagrams for architecture visualization
- Code blocks with syntax highlighting (`python`, `c`, `glsl`, `bash`)
- Source citations as inline references: `Source: kitty/data-types.c:525`
- Tables for structured data (shader inventory, module catalogs, line counts)
- Actual traceback output as evidence for behavioral observations

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **Runtime Language Architecture Diagram:** A layered diagram showing how C (engine), Python (orchestration), GLSL (GPU rendering), and Go (CLI tools) interact through the `fast_data_types` bridge.
- **Shader Pipeline Diagram:** A flowchart showing how GLSL source flows from disk through `shaders.py` preprocessing to `shaders.c` OpenGL compilation and finally to GPU execution across six rendering stages.
- **Entry Point Failure Trace:** A sequence diagram showing the import chain from `__main__.py` to the `fast_data_types` failure.
- **Kittens Dependency Graph:** A diagram showing how kittens route through `runner.py` → `tui/loop.py` → `fast_data_types`.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/data-types.c`, `kitty/main.py`, `kitty/entry_points.py`, `__main__.py`, `kitty/shaders.py`, `kitty/shaders.c`, `kitty/*.glsl`, `kittens/runner.py`, `kittens/tui/loop.py`, `kittens/*/main.py`, `setup.py`, `kitty/constants.py`, `kitty/launcher/main.c`, `kitty/fast_data_types.pyi`, `kitty/boss.py`, `kitty/screen.c`, `kitty/child-monitor.c` | Comprehensive Q&A document answering five architectural questions about kitty's language architecture, shader role, native bridge, kittens modularity, with evidence from code execution and source analysis |

**No other files are created, updated, or deleted.** The user's rule explicitly states: "Do not modify any existing files in the source repository." The output is a single new file in the `blitzy/documentation` directory.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Architecture Q&A / Codebase Exploration Document
Source Code References:
  - kitty/data-types.c (lines 430–612: PyInit_fast_data_types, module method table, subsystem initializers)
  - kitty/main.py (lines 11–53: imports from fast_data_types for GLFW, fonts, shaders, windows)
  - kitty/entry_points.py (lines 183–197: main() dispatch falling through to kitty.main)
  - __main__.py (lines 5–7: entry point invoking kitty.entry_points.main)
  - kitty/shaders.py (lines 1–205: Program class, LoadShaderPrograms, GLSL loading/preprocessing)
  - kitty/shaders.c (lines 1–1285: compile_program, OpenGL shader compilation)
  - kitty/cell_vertex.glsl (233 lines: cell rendering vertex shader)
  - kitty/cell_fragment.glsl (~170 lines: cell rendering fragment shader with multi-pass logic)
  - kitty/cell_defines.glsl (32 lines: compile-time macro definitions for shader phases)
  - kitty/border_vertex.glsl (~50 lines: border drawing vertex shader)
  - kitty/border_fragment.glsl (5 lines: border pass-through fragment shader)
  - kitty/bgimage_*.glsl, kitty/graphics_*.glsl, kitty/tint_*.glsl (background image, inline graphics, tint shaders)
  - kitty/alpha_blend.glsl, kitty/linear2srgb.glsl (utility shaders for blending and color space)
  - kittens/runner.py (lines 14, 110–133: kitten discovery and execution)
  - kittens/tui/loop.py (line 19: imports from fast_data_types for TTY operations)
  - kittens/diff/main.py (line 14: "Must be run as kitten diff")
  - kittens/icat/main.py (line 172: "This should be run as kitten icat")
  - kittens/hints/main.py (line 11: imports fast_data_types.get_options)
  - setup.py (line 1091: compile_c_extension for fast_data_types)
  - kitty/constants.py (lines 23–26: version, appname)
  - kitty/launcher/main.c (lines 1–60: native launcher embedding CPython)
  - kitty/fast_data_types.pyi (type stubs documenting the C extension surface area)
  - kitty/screen.c (4,932 lines: largest C module — terminal screen model)
  - kitty/boss.py (3,094 lines: largest Python module — lifecycle coordinator)
  - kitty/child-monitor.c (2,016 lines: PTY I/O multiplexing)
Sections:
  - Introduction and Methodology (approach, constraints, observational method)
  - Q1: Language Roles at Runtime (quantitative breakdown, per-language analysis, diagram)
  - Q2: GLSL Shaders in a Terminal (13-file inventory, loading pipeline, rendering stages)
  - Q3: The Entry Point Failure (traceback, import chain, fast_data_types contents, build necessity)
  - Q4: Kittens Modularity (standalone attempt, dependency chain, SystemExit guards, TUI dependency)
  - Conclusions (architectural insights, what the evidence reveals)
Diagrams:
  - Mermaid: Runtime language architecture layered diagram
  - Mermaid: Shader compilation and rendering pipeline
  - Mermaid: Entry point import failure trace
  - Mermaid: Kittens dependency graph
Key Citations: kitty/data-types.c, kitty/main.py, kitty/shaders.py, kittens/runner.py, kittens/tui/loop.py, setup.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration changes are required. The new file resides in `blitzy/documentation/`, which is a standalone output directory not governed by Sphinx, mkdocs, or any documentation generator configuration in the repository.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The output document is self-contained and does not reference or include content from the existing Sphinx docs tree.
- **No navigation links:** The document is not part of the `docs/` Sphinx build.
- **No table of contents updates:** No existing docs index needs modification.
- **No glossary updates:** The document uses standard kitty terminology already established in the codebase.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task does not require any documentation generation tools, frameworks, or build pipelines. The output is a hand-authored Markdown file. However, the following project-level dependencies are relevant to understanding and referencing the architecture being documented:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| PyPI | sphinx | (per docs/requirements.txt) | Existing docs site generator (not used for this task) |
| PyPI | furo | (per docs/requirements.txt) | Sphinx theme for existing docs (not used for this task) |
| System | Python | >=3.8 (per pyproject.toml line 2) | Runtime for kitty's Python layer |
| System | GCC / Clang | C11-capable (per setup.py line 492) | Compiles kitty/fast_data_types.so |
| System | Go | 1.22 (per go.mod line 3) | Compiles the kitten static binary |
| System | OpenGL | 3.3+ (per tech spec Section 5.1) | GPU rendering API consumed by GLSL shaders |
| System | FreeType | (pkg-config detected) | Font rasterization in kitty/freetype.c |
| System | HarfBuzz | >=1.5 (per tech spec) | Text shaping |
| System | FontConfig | (pkg-config detected, Linux only) | Font discovery |

**Note:** No documentation-specific packages need to be installed. The task produces a single `.md` file using only information gathered from the existing codebase and behavioral observations.

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file `blitzy/documentation/kitty_815df1e210e0.md` is not referenced by any existing documentation and does not replace or supersede any existing docs.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt contains five distinct questions. Coverage is measured by complete, evidence-based answers to each:

| Question | Topic | Coverage Target | Evidence Sources |
|---|---|---|---|
| Q1 | Language Role Distribution (Python vs. C at runtime) | 100% — quantitative line counts, per-subsystem language assignment, runtime call chain analysis | `kitty/*.c` (35,155 LOC), `kitty/*.py` (20,647 LOC), `tools/*.go` (193 files), `setup.py` line 1091, `kitty/data-types.c` lines 525–612 |
| Q2 | GLSL Shader Role and Centrality | 100% — inventory of all 13 shaders, loading/compilation pipeline, rendering stage descriptions, evidence of no CPU fallback | `kitty/*.glsl` (13 files), `kitty/shaders.py`, `kitty/shaders.c`, `kitty/cell_defines.glsl` |
| Q3 | Entry Point Failure and the Native Bridge | 100% — actual traceback from `python3 __main__.py`, complete import chain trace, contents of `fast_data_types`, build system explanation | `__main__.py`, `kitty/entry_points.py`, `kitty/main.py`, `kitty/borders.py`, `kitty/data-types.c`, `setup.py` |
| Q4 | Kittens Independence Assessment | 100% — standalone execution attempt, import dependency chain, SystemExit guards catalog, TUI framework dependency | `kittens/runner.py`, `kittens/tui/loop.py`, `kittens/diff/main.py`, `kittens/icat/main.py`, all kitten `main.py` files |
| Q5 | Observational Method | 100% — all answers derived from code execution and source analysis, not assumptions | Behavioral experiments run in the repository environment |

- **Current Coverage:** 0% (document does not exist yet)
- **Target Coverage:** 100% — all five questions fully answered with evidence and rationale

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**

- Every question must be answered with specific file paths and line numbers as evidence
- Every architectural claim must be supported by either source code inspection or behavioral observation (command execution with output)
- Diagrams must accompany complex relationships (language layers, shader pipeline, import chains)
- No question may be answered with "we assume" or "typically" — only evidence-based assertions

**Accuracy Validation:**

- Traceback outputs must be actual outputs from running commands in the repository (already captured during context gathering)
- Line counts must come from actual `wc -l` measurements (already obtained: C=35,155, Python=20,647)
- Import chain traces must follow actual source code import statements with file:line references
- GLSL file inventory must match the actual files found via `find kitty/ -name "*.glsl"` (13 files confirmed)

**Clarity Standards:**

- Technical accuracy with accessible language for a developer onboarding into the codebase
- Progressive disclosure: start with the question, present the evidence, then derive the conclusion
- Consistent terminology: use "native bridge" for `fast_data_types`, "orchestration layer" for Python, "engine layer" for C throughout

**Maintainability:**

- Source citations included for every claim (file path and line number)
- Document structured as a standalone reference that does not depend on external documents

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 (architecture layers, shader pipeline, import failure trace, kittens dependency graph)
- **Minimum code evidence blocks:** 5 (actual traceback from `__main__.py`, kitten standalone failure, `PyInit_fast_data_types` init sequence, shader loading code, kitten SystemExit guards)
- **Code example testing:** All tracebacks and command outputs captured from live execution in the repository environment
- **Visual content freshness:** Based on current repository state at commit `815df1e21`

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New Documentation Files:**

- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive Q&A document answering the user's five architectural questions

**Source Code Analyzed (Read-Only — No Modifications):**

- `kitty/data-types.c` — Native bridge module definition and initialization
- `kitty/data-types.h` — Header for the native bridge
- `kitty/fast_data_types.pyi` — Type stubs documenting the C extension's Python surface area
- `kitty/main.py` — Application startup sequence
- `kitty/entry_points.py` — CLI dispatch and entry point routing
- `__main__.py` — Top-level Python entry point
- `kitty/shaders.py` — GLSL shader loading, preprocessing, and compilation orchestration
- `kitty/shaders.c` — C-side OpenGL shader compilation
- `kitty/*.glsl` — All 13 GLSL shader source files (cell, border, bgimage, graphics, tint, alpha_blend, linear2srgb, cell_defines)
- `kitty/screen.c` — Terminal screen model (largest C file, 4,932 lines)
- `kitty/child-monitor.c` — PTY I/O multiplexing
- `kitty/boss.py` — Central Python lifecycle coordinator
- `kitty/constants.py` — Runtime constants and path resolution
- `kitty/borders.py` — Border rendering (first import chain failure point)
- `kitty/launcher/main.c` — Native C launcher embedding CPython
- `kittens/runner.py` — Kitten discovery and execution framework
- `kittens/tui/loop.py` — TUI event loop with native bridge dependency
- `kittens/tui/handler.py` — TUI handler base class
- `kittens/diff/main.py` — Diff kitten (example of standalone rejection)
- `kittens/icat/main.py` — icat kitten (example of standalone rejection)
- `kittens/hints/main.py` — Hints kitten (direct fast_data_types import)
- `kittens/ask/main.py`, `kittens/clipboard/main.py`, `kittens/pager/main.py`, `kittens/show_key/main.py`, `kittens/hyperlinked_grep/main.py`, `kittens/query_terminal/main.py` — Additional kittens demonstrating the SystemExit pattern
- `setup.py` — Build system (specifically line 1091 for C extension compilation)
- `pyproject.toml` — Python version constraint
- `go.mod` — Go module version
- `Makefile` — Build targets
- `docs/requirements.txt` — Documentation dependencies

**Behavioral Experiments (Temporary — Cleaned Up):**

- `python3 __main__.py` — Capturing entry point failure traceback
- `python3 -c "import kitty.fast_data_types"` — Verifying native module absence
- `python3 -c "from kittens.runner import run_kitten; run_kitten('diff')"` — Verifying kitten standalone failure
- `wc -l` on C, Python, Go, and GLSL files — Quantitative language breakdown
- `grep` and `find` commands — Import chain discovery

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository may be modified per user instruction ("the repository itself should remain unchanged")
- **Test file modifications:** No test files are created or modified
- **Feature additions or refactoring:** This is a documentation-only task
- **Build system execution:** The project is not built (`setup.py` is not run); the documentation explains why it would need to be, but does not perform the build
- **Existing documentation updates:** No changes to `docs/**/*.rst`, `README.asciidoc`, `CONTRIBUTING.md`, `INSTALL.md`, or any existing files
- **Deployment configuration changes:** No CI/CD, packaging, or deployment changes
- **Go tooling analysis depth:** While Go is mentioned in the language distribution section, deep analysis of the `tools/` directory is out of scope — the focus is on the Python/C/GLSL interaction per user questions
- **macOS-specific paths:** The Objective-C files (`core_text.m`, `cocoa_window.m`) are acknowledged but not deeply analyzed since the user's questions focus on the Python/C/GLSL relationship
- **Performance benchmarking:** No runtime performance testing is performed; performance claims reference architectural design, not measurements

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a single hand-authored Markdown file, not generated by a documentation framework
- **Documentation preview command:** The file can be previewed with any Markdown renderer (e.g., `cat blitzy/documentation/kitty_815df1e210e0.md` or viewed in a Markdown-capable editor/viewer)
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown file using fenced code blocks (```` ```mermaid ... ``` ````); no external generation step is needed
- **Documentation deployment command:** Not applicable — the file is committed to the repository, not deployed to a docs hosting service
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every architectural claim must reference specific source files with file paths and line numbers
- **Style guide:** Follow the user's implementation rule:
  - Provide thinking and rationale behind answers
  - Base all answers on code as the source of truth
  - Do not make assumptions
  - Answers should be comprehensive
- **Documentation validation:** Verify that:
  - All five user questions are addressed
  - All tracebacks and command outputs match actual repository behavior
  - All file paths and line numbers reference real locations in the codebase
  - Mermaid diagrams render correctly with standard Mermaid syntax

### 0.9.2 Repository Preservation Protocol

Per the user's constraint, all observational experiments are conducted using read-only commands:

- `python3 __main__.py` — Observes failure; does not modify anything
- `python3 -c "..."` — Executes in-memory; no file side effects
- `grep`, `find`, `wc -l`, `cat`, `head` — Read-only filesystem inspection
- Any temporary scripts (if created) must be deleted before task completion
- `git status` can be used to verify no repository modifications occurred

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project-level implementation rule:

- **Do not modify any existing files in the source repository.** The repository must remain unchanged. Only the new file `blitzy/documentation/kitty_815df1e210e0.md` may be created.
- **Create a new markdown document named `kitty_815df1e210e0.md`** (matching the source branch name `kitty_815df1e210e0`). Place it in the `blitzy/documentation` directory.
- **Provide thinking and rationale behind the answers.** Each answer must explain the reasoning process, not just state conclusions.
- **Do not make assumptions — base answers on the code as the source of truth.** Every claim must be traceable to a specific file, line number, or behavioral observation.
- **Explore by observing how the code behaves when exercised.** Use actual execution outputs (tracebacks, command results) as primary evidence, rather than relying solely on reading documentation about intended architecture.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.** If any helper scripts are created during the exploration process, they must be removed before the task is complete.
- **Answer comprehensively.** The document should leave no question partially answered. Each of the five questions must be addressed completely with evidence and rationale.

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

The following files and folders were directly examined to derive the conclusions in this Agent Action Plan:

**Root Level Files:**

| File | Purpose in Analysis |
|---|---|
| `__main__.py` | Entry point — traced import chain to `fast_data_types` failure |
| `setup.py` | Build system — confirmed C extension compilation target at line 1091 |
| `pyproject.toml` | Python version constraint (`>=3.8`), mypy strict config |
| `go.mod` | Go module version (1.22), dependency graph |
| `Makefile` | Build and documentation targets |
| `README.asciidoc` | Existing project introduction — assessed for architecture content |
| `CONTRIBUTING.md` | Contributor workflows — assessed for architecture content |
| `INSTALL.md` | Install/build docs — assessed for architecture content |

**kitty/ Directory (Core Application):**

| File | Purpose in Analysis |
|---|---|
| `kitty/data-types.c` (lines 430–612) | `PyInit_fast_data_types()` — the central native bridge module definition |
| `kitty/fast_data_types.pyi` (lines 1–80) | Type stubs documenting the C extension API surface |
| `kitty/main.py` (full) | Application startup — traced import dependencies on `fast_data_types` |
| `kitty/entry_points.py` (full) | CLI dispatch — traced fallthrough to `kitty.main.main()` |
| `kitty/constants.py` (full) | Version info, path resolution, `kitty_exe()`, `kitten_exe()` |
| `kitty/shaders.py` (full) | GLSL loading, preprocessing (macro substitution), compilation orchestration |
| `kitty/shaders.c` | C-side OpenGL shader compilation (1,285 lines) |
| `kitty/cell_vertex.glsl` (full, 233 lines) | Cell rendering vertex shader — most complex GLSL file |
| `kitty/cell_fragment.glsl` (full, ~170 lines) | Cell rendering fragment shader with multi-pass logic |
| `kitty/cell_defines.glsl` (full, 32 lines) | Compile-time macro definitions for shader phases |
| `kitty/border_vertex.glsl` (full) | Border drawing vertex shader |
| `kitty/border_fragment.glsl` (full, 5 lines) | Border pass-through fragment shader |
| `kitty/alpha_blend.glsl` | Alpha blending utility shader |
| `kitty/linear2srgb.glsl` | Color space conversion utility shader |
| `kitty/screen.c` | Terminal screen model (4,932 lines — largest C file) |
| `kitty/boss.py` | Lifecycle coordinator (3,094 lines — largest Python file) |
| `kitty/child-monitor.c` | PTY I/O multiplexing (2,016 lines) |
| `kitty/launcher/main.c` (lines 1–60) | Native C launcher embedding CPython |
| `kitty/launcher/` folder | Launcher architecture (3 files: main.c, single-instance.c, launcher.h) |

**kittens/ Directory:**

| File | Purpose in Analysis |
|---|---|
| `kittens/runner.py` (full, 203 lines) | Kitten discovery, resolution, and execution — confirmed `fast_data_types` dependency |
| `kittens/tui/loop.py` (line 19) | TUI event loop — confirmed direct import of `fast_data_types` for TTY operations |
| `kittens/tui/handler.py` (line 10) | TUI handler — confirmed import of `fast_data_types.monotonic` |
| `kittens/tui/` folder (full) | TUI foundation architecture (12 files) |
| `kittens/diff/main.py` (lines 1–14) | Diff kitten — `SystemExit('Must be run as kitten diff')` |
| `kittens/icat/main.py` (lines 1–30, 172) | icat kitten — `SystemExit('This should be run as kitten icat')` |
| `kittens/hints/main.py` (lines 1–15, 259) | Hints kitten — direct `fast_data_types` import + standalone rejection |
| `kittens/ask/main.py` (line 76) | Ask kitten — standalone rejection |
| `kittens/clipboard/main.py` (line 83) | Clipboard kitten — standalone rejection |
| `kittens/pager/main.py` (line 31) | Pager kitten — standalone rejection |
| `kittens/show_key/main.py` (line 22) | Show_key kitten — standalone rejection |
| `kittens/hyperlinked_grep/main.py` (line 7) | Hyperlinked_grep kitten — standalone rejection |
| `kittens/query_terminal/main.py` (line 265) | Query_terminal kitten — standalone rejection |

**docs/ Directory:**

| File/Folder | Purpose in Analysis |
|---|---|
| `docs/` folder (full contents) | Assessed existing documentation infrastructure |
| `docs/requirements.txt` (full) | Confirmed Sphinx + Furo + plugins as doc toolchain |
| `docs/kittens/` folder | Assessed existing kitten documentation |
| `docs/conf.py` | Sphinx configuration — assessed for documentation patterns |

**Quantitative Measurements:**

| Measurement | Command | Result |
|---|---|---|
| C files (non-vendored) | `find . -name "*.c" -not -path "./3rdparty/*" -not -path "./glfw/*" -not -path "./glad/*" \| wc -l` | 52 files |
| Python files | `find . -name "*.py" -not -path "./.git/*" \| wc -l` | 214 files |
| Go files | `find . -name "*.go" -not -path "./.git/*" -not -path "./3rdparty/*" \| wc -l` | 258 files |
| GLSL files | `find . -name "*.glsl" \| wc -l` | 13 files |
| C LOC in kitty/ | `wc -l kitty/*.c` | 35,155 lines |
| Python LOC in kitty/ | `wc -l kitty/*.py` | 20,647 lines |

**Tech Spec Sections Retrieved:**

| Section | Relevance |
|---|---|
| 1.1 Executive Summary | Project overview, three-language architecture description |
| 3.1 Programming Languages | Detailed language roles, C11 standard, GLSL shader inventory, Go version |
| 5.1 High-Level Architecture | System layers, component inventory, data flow descriptions |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

