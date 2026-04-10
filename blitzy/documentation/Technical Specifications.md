# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive investigative Q&A document** that explains the end-to-end behavior of the `choose-fonts` kitten within the kovidgoyal/kitty terminal emulator repository. The user is onboarding to this codebase and has specific behavioral questions they need answered—grounded in source-code analysis—about how the font-selection workflow operates, how the chosen font is persisted, and how to verify that behavior at runtime.

- **Category:** Create new documentation
- **Documentation type:** Investigative Q&A / Onboarding Reference Guide

The user's documentation requirements, with enhanced clarity, are:

- **Q1 — Build and Launch:** How to build the kitty repository from source and start a single kitty instance with default (uncustomized) settings.
- **Q2 — Invocation:** How to invoke the `choose-fonts` kitten from inside a running kitty terminal instance.
- **Q3 — Subcommand Registration and Option Parsing:** How the `choose-fonts` subcommand is registered in kitty's CLI command tree, what options it accepts (e.g., `--reload-in`), and how those option values are parsed and stored.
- **Q4 — Option Value Flow:** How the parsed option values propagate through the program's layers—from the Go CLI handler through the TUI handler and down to the final confirmation pane.
- **Q5 — Finalization Behavior:** What kitty does when the user presses Enter at the final confirmation step—specifically, what file is modified, how the modification is structured, and whether kitty signals any process to reload configuration.
- **Q6 — Persistence Verification:** Whether the font selection persists across kitty restarts (i.e., is this a permanent change to `kitty.conf`, or a session-only change?), and how to verify this through a runtime example.

### 0.1.2 Special Instructions and Constraints

The user has established the following critical constraints:

- **No source file modifications:** "Don't modify any repository source files." The existing codebase must remain completely untouched.
- **Temporary artifacts allowed:** "Creating temporary configs, logging settings, or small test scripts/programs are fine."
- **Cleanup required:** "Delete any temporary test artifacts when you're done and leave the codebase unchanged."
- **Implementation rule — SWE-AtlasQnA-Repo:** The final deliverable must be a new markdown document named `<source_branch_name>.md` (resolved to `kitty_815df1e210e0.md`) placed in the `blitzy/documentation` directory in the destination repo. The document must comprehensively answer the questions posed, provide thinking/rationale behind the answers, base answers on the code as the source of truth, and not modify any existing files in the source repository.
- **No design system specified:** No component library or UI design system is relevant to this documentation task.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer Q1**, we will trace the build system entry point (`./dev.sh build` → `setup.py`) as documented in `docs/build.rst`, and explain the output binary path (`kitty/launcher/kitty`) and how to launch it with defaults.
- To **answer Q2**, we will document the two invocation routes: `kitten choose-fonts` (Go-native CLI via `tools/cmd/tool/main.go`) and `kitty +kitten choose_fonts` (Python runner via `kittens/runner.py` → `kitty/entry_points.py`).
- To **answer Q3**, we will trace the CLI registration flow in `kittens/choose_fonts/main.go:EntryPoint()` where the `choose-fonts` subcommand and `choose_fonts` alias are added to the root command tree, and document the `--reload-in` option spec (choices: `parent`, `all`, `none`; default: `parent`).
- To **answer Q4**, we will trace the `Options` struct through `main()` → `handler` → pane chain, showing how `opts.Reload_in` flows from the CLI parse to the `final_pane.on_key_event()` handler.
- To **answer Q5**, we will analyze `kittens/choose_fonts/final.go` lines 78–96, showing the `config.Patcher` call that writes `font_family`, `bold_font`, `italic_font`, and `bold_italic_font` settings into a `# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS` sentinel block in `~/.config/kitty/kitty.conf`, and the `config.ReloadConfigInKitty()` call in `tools/config/api.go` that sends `SIGUSR1` to the parent or all kitty processes.
- To **answer Q6**, we will provide evidence from the code that the change is **persistent** (disk write to `kitty.conf`), and outline a runtime verification procedure using a temporary `KITTY_CONFIG_DIRECTORY` to demonstrate the before/after state of the config file across restarts—then clean up all temporary artifacts.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation needs are inferred:

- **Backend protocol explanation:** The `choose-fonts` kitten uses a unique companion-process architecture (`backend.go` spawns `backend.py` via `kitty +runpy`). This two-process Go/Python IPC protocol is non-obvious and should be documented for onboarding clarity.
- **UI state machine:** The handler has three states (`SCANNING_FAMILIES`, `LISTING_FAMILIES`, `CHOOSING_FACES`) plus the final pane. Documenting this state flow is essential to understanding the end-to-end journey.
- **Backup file creation:** When the user presses Enter, the patcher creates a `.bak` backup of the original `kitty.conf` before modifying it. This is an important detail for someone trying to understand whether the change is recoverable.
- **Config directory resolution:** The config path is resolved via `utils.ConfigDir()` which checks `KITTY_CONFIG_DIRECTORY` → `XDG_CONFIG_HOME` → `~/.config/kitty`. This affects where the runtime test should look.
- **The `s` / `S` export alternative:** In addition to Enter (persist), the final pane supports pressing `s` to write settings to STDOUT instead of `kitty.conf`. This is a secondary path worth documenting for completeness.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals that kitty uses a **Sphinx/reStructuredText documentation system** with the following characteristics:

- **Documentation framework:** Sphinx, with the `furo` theme, configured in `docs/conf.py`
- **Documentation generator configuration:** `docs/conf.py` (Sphinx config), `docs/Makefile` (build driver)
- **Documentation dependencies:** Pinned in `docs/requirements.txt` (sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild)
- **Diagram tools detected:** Mermaid (used in tech spec context); the existing docs primarily use inline reStructuredText images and screenshots
- **Build commands:** `./dev.sh deps -for-docs && ./dev.sh docs` or `make -C docs html`
- **Existing kittens documentation:** 14 kitten-specific `.rst` pages exist in `docs/kittens/` (ssh, themes, broadcast, clipboard, hints, hyperlinked_grep, icat, query_terminal, remote_file, transfer, unicode_input, custom, diff, panel)
- **Critical finding:** There is **no existing documentation page** for the `choose-fonts` kitten in `docs/kittens/`. The kitten is completely undocumented in the Sphinx docs tree. References to `choose_fonts` appear only in the source code and the auto-generated feature catalog.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to locate all code relevant to the `choose-fonts` kitten:

- **Kitten implementation directory:** `kittens/choose_fonts/` — 14 files (Go + Python):
  - Entry points: `main.go` (CLI registration + session lifecycle), `main.py` (summary/docs metadata stub)
  - Backend: `backend.go` (Go process controller), `backend.py` (Python worker process)
  - UI layer: `ui.go` (handler/state machine), `list.go` (family browser), `family_list.go` (filtered list model)
  - Face editing: `faces.go` (face preview pane), `face.go` (single face editor)
  - Finalization: `final.go` (confirmation + config patching)
  - Supporting: `types.go` (data model), `styles.go` (style normalization), `graphics.go` (graphics protocol)
  - Package init: `__init__.py` (empty marker)

- **CLI registration site:** `tools/cmd/tool/main.go` line 82: `choose_fonts.EntryPoint(root)`

- **Config patching engine:** `tools/config/api.go` — `Patcher.Patch()` (sentinel-block replacement in config files), `ReloadConfigInKitty()` (SIGUSR1 dispatch)

- **Font infrastructure:** `kitty/fonts/` — `__init__.py` (type definitions), `fontconfig.py` (Linux backend), `core_text.py` (macOS backend), `common.py` (resolution pipeline), `list.py` (family grouping + JSON export + `choose-fonts` handoff)

- **Entry points:** `kitty/entry_points.py` — `list_fonts()` delegates to `kitty/fonts/list.py:main()` which `os.execlp`s into `kitten choose-fonts`; `run_kitten()` resolves via `kittens/runner.py`

- **Configuration system:** `kitty/config.py`, `kitty/options/definition.py` (font_family, bold_font, italic_font, bold_italic_font options defined at lines 35–57), `kitty/boss.py:load_config_file()` (reload handler)

- **Config directory resolution:** `kitty/constants.py` lines 87–133 (Python side), `tools/utils/paths.go` lines 88–134 (Go side)

### 0.2.3 Web Search Research Conducted

No external web searches were required for this task. All questions posed by the user can be answered exhaustively from the source code, which serves as the authoritative truth per the implementation rules. The documentation strategy relies solely on code-level evidence to ensure accuracy and avoid assumptions.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and files contain the source of truth for answering the user's questions. Each is mapped to the documentation content it contributes:

- **Module: `kittens/choose_fonts/main.go`**
  - Public APIs: `EntryPoint(root *cli.Command)`, `Options` struct, private `main(opts *Options)`
  - Current documentation: Missing — no kitten docs page exists
  - Documentation needed: Subcommand registration walkthrough, option definitions, session lifecycle narrative

- **Module: `kittens/choose_fonts/final.go`**
  - Public APIs: `final_pane` struct, `on_key_event()`, `faces_settings.serialized()`
  - Current documentation: Missing
  - Documentation needed: Explanation of the finalization step, config patching trigger, Enter/Esc/s keybindings, persistence proof

- **Module: `tools/config/api.go`**
  - Public APIs: `Patcher.Patch()`, `ReloadConfigInKitty()`
  - Current documentation: Missing (no standalone docs for config patching utilities)
  - Documentation needed: Detailed explanation of sentinel-block patching logic, backup creation, SIGUSR1 reload mechanism

- **Module: `kittens/choose_fonts/ui.go`**
  - Public APIs: `State` enum (`SCANNING_FAMILIES`, `LISTING_FAMILIES`, `CHOOSING_FACES`), `handler` struct, `TextStyle`
  - Current documentation: Missing
  - Documentation needed: State machine diagram, event routing explanation, terminal query protocol

- **Module: `kittens/choose_fonts/backend.go` + `backend.py`**
  - Public APIs: `kitty_font_backend_type.start()`, `.query()`, `.release()` (Go); `main()`, `list_monospaced_fonts`, `render_family_samples`, `read_variable_data` actions (Python)
  - Current documentation: Missing
  - Documentation needed: Go↔Python IPC protocol overview, command/response schema

- **Module: `kitty/entry_points.py`**
  - Relevant functions: `run_kitten()`, `list_fonts()`, `namespaced()`
  - Documentation needed: Explanation of how `kitty +kitten choose_fonts` invokes the Go-compiled kitten binary

- **Module: `docs/build.rst`**
  - Current documentation: Exists — covers building from source with `./dev.sh build` and running via `kitty/launcher/kitty`
  - Documentation needed: Summarized in the Q&A output document for the build-and-launch question

- **Module: `kitty/constants.py` (lines 87–133)**
  - Relevant logic: `_get_config_dir()` — config directory resolution
  - Documentation needed: Explanation of where `kitty.conf` is located and how it can be overridden for testing

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented kitten:** The `choose-fonts` kitten has zero user-facing documentation in the Sphinx tree. Of the 20+ built-in kittens, only 14 have documentation pages in `docs/kittens/`; `choose_fonts` is among the undocumented ones.
- **No choose-fonts onboarding guide:** There is no existing resource that walks a newcomer through the end-to-end behavior of this kitten.
- **No config patching documentation:** The `config.Patcher` mechanism in `tools/config/api.go` is entirely undocumented—it is used by both `choose_fonts` and `themes` kittens but has no standalone reference.
- **No architecture overview for mixed Go/Python kittens:** The `choose_fonts` kitten's companion-process architecture (Go TUI spawning a Python backend) is unique among kittens and has no architectural documentation.
- **Missing persistence explanation:** Nowhere in the existing documentation is it explicitly stated that `choose-fonts` modifies `kitty.conf` on disk. The themes kitten docs explain a similar persistence model (`current-theme.conf`), but no equivalent exists for font selection.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive Markdown document placed according to the implementation rule `SWE-AtlasQnA-Repo`. The document structure is:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction / Context
        ├── Q1: Building and Launching Kitty
        │   ├── Build procedure
        │   ├── Binary location
        │   └── Launching with default settings
        ├── Q2: Invoking choose-fonts
        │   ├── Primary invocation: kitten choose-fonts
        │   └── Alternative: kitty +kitten choose_fonts
        ├── Q3: Subcommand Registration and Option Parsing
        │   ├── EntryPoint registration in CLI tree
        │   ├── --reload-in option definition
        │   └── Options struct population
        ├── Q4: Option Value Flow Through the Program
        │   ├── main() receives Options
        │   ├── handler stores opts
        │   └── final_pane reads opts.Reload_in
        ├── Q5: What Happens When You Press Enter
        │   ├── faces_settings.serialized()
        │   ├── config.Patcher.Patch() sentinel block mechanism
        │   ├── kitty.conf modification details
        │   ├── Backup file creation
        │   └── SIGUSR1 reload signal dispatch
        ├── Q6: Persistence Verification
        │   ├── Code evidence for persistence
        │   ├── Runtime verification procedure
        │   └── Before/after kitty.conf inspection
        ├── End-to-End Flow Diagram (Mermaid)
        └── Summary and Key Takeaways
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- "Extract CLI registration details from `kittens/choose_fonts/main.go:EntryPoint()` (lines 74–99)"
- "Extract option flow by tracing the `Options` struct from `main.go:main()` through `ui.go:handler` to `final.go:on_key_event()`"
- "Extract persistence proof from `final.go:on_key_event()` (lines 78–96) cross-referenced with `tools/config/api.go:Patcher.Patch()` (lines 305–350)"
- "Extract reload mechanism from `tools/config/api.go:ReloadConfigInKitty()` (lines 352–371)"
- "Generate Mermaid diagrams by mapping the component relationships across `main.go` → `ui.go` → `list.go` → `faces.go` → `final.go`"
- "Build runtime verification script using `KITTY_CONFIG_DIRECTORY` override from `kitty/constants.py` lines 87–89"

**Documentation Standards:**

- Markdown formatting with hierarchical headers (`#`, `##`, `###`)
- Mermaid diagrams for architectural flows and state machines using fenced code blocks
- Code snippets using fenced blocks with language identifiers (go, python, bash) kept to 2–3 lines each for illustration
- Source citations in the form `Source: path/to/file.ext:LineNumber` for every technical claim
- Tables for parameter descriptions, option values, and component mappings

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the deliverable document:

- **End-to-End Flow Diagram:** A flowchart showing the journey from `kitten choose-fonts` invocation through CLI parsing, backend startup, UI state transitions, font selection, config patching, and reload signaling
- **State Machine Diagram:** A state diagram for the `handler`'s `State` enum: `SCANNING_FAMILIES` → `LISTING_FAMILIES` → `CHOOSING_FACES` → `final_pane`, with transitions labeled by user actions (search, select family, press Enter, press Esc)
- **Option Flow Diagram:** A sequence diagram showing how `--reload-in` flows from CLI parse to `cmd.GetOptionValues(&opts)` → `main(&opts)` → `handler{opts: opts}` → `final_pane.handler.opts.Reload_in` → `config.ReloadConfigInKitty()`
- **Config Patching Diagram:** A before/after illustration showing how `Patcher.Patch()` transforms `kitty.conf` by commenting out existing font directives and inserting the `# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS` sentinel block

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

The transformation mapping below captures every documentation file to be created, referenced, or used in this documentation exercise. Per the implementation rule `SWE-AtlasQnA-Repo`, the sole CREATE target is a single Markdown file placed in `blitzy/documentation/`.

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kittens/choose_fonts/main.go`, `kittens/choose_fonts/final.go`, `kittens/choose_fonts/ui.go`, `kittens/choose_fonts/backend.go`, `kittens/choose_fonts/backend.py`, `kittens/choose_fonts/faces.go`, `kittens/choose_fonts/face.go`, `kittens/choose_fonts/list.go`, `kittens/choose_fonts/family_list.go`, `kittens/choose_fonts/types.go`, `kittens/choose_fonts/styles.go`, `kittens/choose_fonts/graphics.go`, `tools/config/api.go`, `tools/cmd/tool/main.go`, `kitty/entry_points.py`, `kittens/runner.py`, `kitty/constants.py`, `kitty/fonts/list.py`, `kitty/boss.py`, `docs/build.rst` | Comprehensive Q&A document answering all six user questions about the choose-fonts kitten behavior, with code-based evidence, Mermaid diagrams, and a runtime verification procedure |
| `docs/build.rst` | REFERENCE | — | Used as source for build instructions (Q1); no modification |
| `docs/kittens/themes.rst` | REFERENCE | — | Used as a style reference for kitten documentation patterns; no modification |
| `kittens/choose_fonts/main.go` | REFERENCE | — | Primary source for subcommand registration, option definitions, and session lifecycle (Q2, Q3, Q4) |
| `kittens/choose_fonts/final.go` | REFERENCE | — | Primary source for finalization behavior, config patching trigger, and persistence proof (Q5, Q6) |
| `tools/config/api.go` | REFERENCE | — | Primary source for Patcher.Patch() mechanics, backup logic, and ReloadConfigInKitty() SIGUSR1 dispatch (Q5) |
| `kittens/choose_fonts/ui.go` | REFERENCE | — | Primary source for handler state machine, event routing, and terminal query initialization (Q4) |
| `kittens/choose_fonts/backend.go` | REFERENCE | — | Source for Go→Python IPC mechanism, backend process lifecycle (Q4) |
| `kittens/choose_fonts/backend.py` | REFERENCE | — | Source for Python backend command handlers: list_monospaced_fonts, render_family_samples, read_variable_data (Q4) |
| `kitty/constants.py` | REFERENCE | — | Source for config directory resolution logic used in runtime verification (Q6) |
| `kitty/entry_points.py` | REFERENCE | — | Source for entry point routing: how `kitty +kitten` reaches the kitten runner (Q2) |
| `kittens/runner.py` | REFERENCE | — | Source for kitten name resolution, module loading, and execution pipeline (Q2) |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Investigative Q&A / Onboarding Reference
Source Code: kittens/choose_fonts/*, tools/config/api.go, kitty/entry_points.py,
             kittens/runner.py, kitty/constants.py, docs/build.rst
Sections:
    - Introduction (context and purpose of the investigation)
    - Q1: Building and Launching Kitty (from docs/build.rst + setup.py)
    - Q2: Invoking choose-fonts (from main.go:EntryPoint + entry_points.py)
    - Q3: Subcommand Registration and Options (from main.go lines 74-99)
    - Q4: Option Value Flow (from main.go → ui.go → final.go trace)
    - Q5: Finalization Behavior (from final.go + tools/config/api.go)
    - Q6: Persistence Verification (from final.go + constants.py + runtime test)
    - End-to-End Flow Diagram (Mermaid)
    - Summary and Key Takeaways
Diagrams:
    - End-to-end flow diagram (Mermaid flowchart)
    - UI state machine diagram (Mermaid stateDiagram)
    - Option value flow diagram (Mermaid sequence diagram)
    - Config patching before/after (code block illustration)
Key Citations:
    kittens/choose_fonts/main.go, kittens/choose_fonts/final.go,
    kittens/choose_fonts/ui.go, kittens/choose_fonts/backend.go,
    kittens/choose_fonts/backend.py, tools/config/api.go,
    tools/cmd/tool/main.go, kitty/entry_points.py,
    kittens/runner.py, kitty/constants.py, kitty/fonts/list.py,
    docs/build.rst
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be modified for this task. The deliverable is a standalone Markdown file in `blitzy/documentation/` and does not integrate into kitty's Sphinx documentation pipeline. No changes to `docs/conf.py`, `docs/Makefile`, or any navigation/sidebar configuration are required.

### 0.5.4 Cross-Documentation Dependencies

- **Shared content:** None—the deliverable is self-contained.
- **Navigation links:** Not applicable—the file is placed in the destination repo's `blitzy/documentation/` directory, outside the Sphinx tree.
- **Table of contents:** Not applicable.
- **Internal cross-references:** The document will include relative references to source files by their repository paths (e.g., `kittens/choose_fonts/main.go`) for traceability, but no hyperlinks to other documentation pages are created.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task requires no external documentation tooling installation. The deliverable is a plain Markdown file that will be created directly. However, the following dependencies are relevant to the subject matter being documented and to the runtime verification procedure:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| system | python3 | >= 3.8 | Required runtime for kitty (per `pyproject.toml` `requires-python = ">=3.8"`) |
| system | go | >= 1.22 | Required build-time dependency (per `go.mod`) |
| system | gcc or clang | — | C compiler required for building kitty native extensions |
| pip | sphinx | pinned in `docs/requirements.txt` | Existing docs framework (reference only; not needed for this task) |
| pip | furo | pinned in `docs/requirements.txt` | Existing docs theme (reference only; not needed for this task) |
| go module | github.com/shirou/gopsutil/v3 | per `go.sum` | Used by `tools/config/api.go` for process enumeration in `ReloadConfigInKitty()` |
| go module | golang.org/x/sys | per `go.sum` | Used by `tools/config/api.go` for `unix.SIGUSR1` signal dispatch |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file is a standalone document that does not modify or depend on any existing documentation links. All internal references within the Q&A document use repository-relative source paths (e.g., `kittens/choose_fonts/main.go:74`) rather than hyperlinks to rendered documentation pages.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The coverage targets are measured against the six explicit questions the user posed:

| Question | Topic | Source Files Analyzed | Coverage Target |
|----------|-------|----------------------|-----------------|
| Q1 | Build and launch | `docs/build.rst`, `setup.py`, `Makefile`, `dev.sh` | 100% — complete build + launch instructions |
| Q2 | Invoking choose-fonts | `kittens/choose_fonts/main.go`, `kitty/entry_points.py`, `kittens/runner.py`, `kitty/fonts/list.py` | 100% — both invocation routes documented |
| Q3 | Subcommand registration & option parsing | `kittens/choose_fonts/main.go` (lines 74–99), `tools/cmd/tool/main.go` (line 82) | 100% — full registration trace with option spec |
| Q4 | Option value flow | `kittens/choose_fonts/main.go` (lines 16–68), `kittens/choose_fonts/ui.go` (lines 42–62), `kittens/choose_fonts/final.go` (lines 72–98) | 100% — end-to-end flow from CLI parse to finalization |
| Q5 | Finalization behavior | `kittens/choose_fonts/final.go` (lines 78–96), `tools/config/api.go` (lines 305–371) | 100% — Patcher mechanics, backup, SIGUSR1 dispatch |
| Q6 | Persistence verification | `kittens/choose_fonts/final.go`, `kitty/constants.py` (lines 87–133), `tools/utils/paths.go` (lines 132–134) | 100% — code evidence + runtime test procedure |

- **Overall target coverage:** 100% of the user's questions answered with code-level evidence
- **Diagrams coverage:** 4 Mermaid diagrams (end-to-end flow, state machine, option flow, config patching)

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer must cite specific source files and line numbers
- Every claim about behavior must be traceable to a code path
- The runtime verification procedure must be executable without modification (given a built kitty binary)
- All keybindings in the final pane (Enter, Esc, s/S, Ctrl+c) must be documented

**Accuracy validation:**

- All code snippets must be extracted directly from the repository (no fabrication)
- Function signatures and struct fields must match the current codebase exactly
- The config patching sentinel format (`# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS`) must be verified from `tools/config/api.go` line 330
- The `--reload-in` default value (`parent`) and choices (`parent, all, none`) must match `main.go` lines 89–90

**Clarity standards:**

- Technical concepts explained with progressive disclosure: high-level summary first, then detailed trace
- Each question answered in a self-contained section that can be read independently
- Consistent terminology: "kitten" (not "plugin"), "kitty.conf" (not "config file"), "sentinel block" (for the `BEGIN_/END_` markers)

**Maintainability:**

- Source citations provided for every technical claim to enable future verification
- Section structure mirrors the user's original questions for easy navigation
- Mermaid diagrams use standard syntax for portability across renderers

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable document answering all user questions

**Source files analyzed for documentation content (read-only, not modified):**

- `kittens/choose_fonts/**/*.go` — All Go source files in the choose_fonts kitten package
- `kittens/choose_fonts/**/*.py` — All Python source files in the choose_fonts kitten package
- `kittens/runner.py` — Kitten resolution and execution framework
- `tools/config/api.go` — Config patching engine and reload signaling
- `tools/cmd/tool/main.go` — CLI command tree assembly
- `tools/utils/paths.go` — Config and cache directory resolution (Go side)
- `kitty/entry_points.py` — Entry point routing for `kitty +kitten` and `kitty list-fonts`
- `kitty/constants.py` — Config directory resolution (Python side), version constants
- `kitty/fonts/list.py` — Font listing and choose-fonts handoff
- `kitty/boss.py` — Config reload handler (`load_config_file`)
- `kitty/options/definition.py` — Font configuration option definitions
- `docs/build.rst` — Build-from-source instructions

**Temporary artifacts (created and then deleted):**

- Temporary `KITTY_CONFIG_DIRECTORY` for runtime verification testing
- Small test/verification scripts for demonstrating persistence behavior

**Documentation content scope:**

- Build and launch instructions for kitty from source
- `choose-fonts` kitten invocation methods
- CLI subcommand registration and option parsing mechanics
- Option value propagation through the Go application layers
- Final confirmation step behavior (config patching, backup, reload)
- Font selection persistence evidence and runtime verification
- Mermaid diagrams for architectural flows and state machines

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing repository files will be modified, per user directive
- **Other kittens:** Documentation for kittens other than `choose-fonts` is not addressed
- **Sphinx documentation updates:** No `.rst` files in `docs/` or `docs/kittens/` will be created or modified
- **Feature additions or code refactoring:** No behavioral changes to any module
- **Test file modifications:** No changes to `kitty_tests/` or any test infrastructure
- **Deployment configuration:** No changes to CI/CD, packaging, or release scripts
- **macOS-specific CoreText analysis:** While the codebase supports macOS via `kitty/fonts/core_text.py`, the documentation focuses on the platform-independent kitten behavior (Linux/fontconfig paths are used as the primary reference)
- **Font rendering internals:** The FreeType/glyph-cache pipeline (`kitty/freetype.c`, `kitty/glyph-cache.c`) is out of scope; the documentation covers only the choose-fonts user-facing workflow
- **Remote control `set-font-size` command:** The `kitty/rc/set_font_size.py` remote command is a separate mechanism and is not covered

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not part of the Sphinx build
- **Documentation preview command:** Any Markdown renderer (e.g., `grip`, VS Code preview, GitHub rendering of `.md`) can preview the output
- **Diagram generation command:** Mermaid diagrams are embedded inline within fenced code blocks and rendered by any Mermaid-compatible viewer (GitHub, VS Code Mermaid extension, Mermaid Live Editor)
- **Documentation deployment command:** Not applicable — the file is committed directly to `blitzy/documentation/`
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with line numbers in the format `Source: path/to/file.ext:LineNumber`
- **Style guide:** Answers must provide thinking/rationale behind conclusions (per `SWE-AtlasQnA-Repo` rule), use the code as the truth, and avoid assumptions
- **Documentation validation:** Manual review for accuracy; code citations verifiable via `read_file` against the repository

### 0.9.2 Runtime Verification Procedure

The runtime verification procedure (for Q6) follows this protocol, designed to be non-destructive and to clean up after itself:

- **Step 1:** Create a temporary directory to serve as `KITTY_CONFIG_DIRECTORY`
- **Step 2:** Create a minimal `kitty.conf` in that directory (empty or with a single comment)
- **Step 3:** Launch kitty with `KITTY_CONFIG_DIRECTORY` pointing to the temporary directory
- **Step 4:** Inside that kitty instance, run `kitten choose-fonts`, select a font, and press Enter
- **Step 5:** Inspect the `kitty.conf` file in the temporary directory to observe the `# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS` block
- **Step 6:** Close and relaunch kitty with the same `KITTY_CONFIG_DIRECTORY` — verify the font is still applied
- **Step 7:** Delete the temporary directory to leave no artifacts

This procedure is documented in the deliverable as a step-by-step runtime example, with expected output at each stage.

## 0.10 Rules for Documentation

The following documentation-specific rules are explicitly mandated by the user and the implementation rule framework:

- **Do not modify any existing files in the source repository.** All source files are read-only references. The deliverable is a new file only.
- **Base all answers on the code as the truth.** Every behavioral claim must be traceable to a specific code path. No assumptions about undocumented behavior are permitted.
- **Provide thinking / rationale behind the answers.** Each answer must explain *why* the code behaves the way it does, not merely *what* it does. For example, when explaining persistence, the rationale should trace from the user pressing Enter → `on_key_event` → `Patcher.Patch()` → disk write → SIGUSR1 → reload.
- **Create the document as `kitty_815df1e210e0.md`.** The filename matches the current source branch name per the `SWE-AtlasQnA-Repo` rule.
- **Place the document in `blitzy/documentation/`.** This is the designated output directory per the implementation rule.
- **Temporary artifacts must be deleted.** Any temporary configs, scripts, or test directories created during the runtime verification procedure must be cleaned up, leaving the codebase unchanged.
- **Include Mermaid diagrams** for complex flows (end-to-end workflow, state machine, option value propagation, config patching).
- **Add source code citations** for all technical details in the format `Source: path/to/file:LineNumber`.
- **Do not assume behavior not evidenced in the code.** If a question cannot be answered from the code alone, state that explicitly rather than speculating.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Choose-Fonts Kitten Implementation (primary sources):**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `kittens/choose_fonts/main.go` | CLI subcommand registration (`EntryPoint`), `Options` struct, session lifecycle (`main()`) |
| `kittens/choose_fonts/final.go` | Final confirmation pane, `faces_settings.serialized()`, config patching trigger via `Patcher.Patch()`, SIGUSR1 reload dispatch, Enter/Esc/s key handling |
| `kittens/choose_fonts/ui.go` | Handler struct, `State` enum (SCANNING/LISTING/CHOOSING), event routing, terminal query initialization, pane lifecycle |
| `kittens/choose_fonts/backend.go` | Go-side backend process controller, pipe IPC, JSON request/response marshaling, kitty executable discovery |
| `kittens/choose_fonts/backend.py` | Python backend process, `list_monospaced_fonts`/`render_family_samples`/`read_variable_data` command handlers |
| `kittens/choose_fonts/list.go` | Family browser pane, search bar, preview cache, font listing display |
| `kittens/choose_fonts/family_list.go` | Filtered family list model, search matching, navigation |
| `kittens/choose_fonts/faces.go` | Face preview pane, `faces_settings` struct, face editing keyboard navigation |
| `kittens/choose_fonts/face.go` | Single face editor pane, variable-font axis tuning, style selection |
| `kittens/choose_fonts/types.go` | Shared data model (ListedFont, VariableData, ResolvedFaces, etc.), variable data cache |
| `kittens/choose_fonts/styles.go` | Style group normalization for variable and fixed fonts |
| `kittens/choose_fonts/graphics.go` | Graphics protocol management for font preview images (5 slots: main, bold, italic, bi, extra) |
| `kittens/choose_fonts/__init__.py` | Empty package marker |

**CLI and Entry Point Infrastructure:**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `tools/cmd/tool/main.go` | CLI command tree assembly; line 82 registers `choose_fonts.EntryPoint(root)` |
| `kitty/entry_points.py` | Entry point routing; `run_kitten()` delegates to `kittens/runner.py`; `list_fonts()` execs into `kitten choose-fonts` |
| `kittens/runner.py` | Kitten name resolution (`resolved_kitten`), module import, execution, result serialization |

**Configuration and Persistence:**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `tools/config/api.go` | `Patcher.Patch()` (lines 305–350): sentinel block replacement, backup creation, atomic file update; `ReloadConfigInKitty()` (lines 352–371): SIGUSR1 dispatch |
| `kitty/constants.py` | `_get_config_dir()` (lines 87–133): config directory resolution via `KITTY_CONFIG_DIRECTORY` / `XDG_CONFIG_HOME` / `~/.config` |
| `tools/utils/paths.go` | `ConfigDir()` (lines 132–134) and `ConfigDirForName()` (lines 88–130): Go-side config directory resolution |
| `kitty/boss.py` | `load_config_file()` (line 2691): config reload handler triggered by SIGUSR1 |
| `kitty/options/definition.py` | Font option definitions: `font_family`, `bold_font`, `italic_font`, `bold_italic_font` (lines 35–57) |

**Font Infrastructure:**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `kitty/fonts/list.py` | `create_family_groups()`, `as_json()`, `main()` → `os.execlp(kitten_exe(), 'kitten', 'choose-fonts')` |
| `kitty/fonts/__init__.py` | Font type definitions: `ListedFont`, `FontSpec`, `VariableData`, etc. |
| `kitty/fonts/common.py` | `get_font_files()`, `face_from_descriptor()`, `spec_for_descriptor()` |
| `kitty/fonts/fontconfig.py` | Linux fontconfig backend for font discovery and matching |

**Build and Documentation Infrastructure:**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `docs/build.rst` | Build-from-source instructions (`./dev.sh build`, binary at `kitty/launcher/kitty`) |
| `Makefile` | Build targets (`all`, `test`, `clean`, `debug`) delegating to `setup.py` |
| `setup.py` | Central build system: version extraction, extension compilation, Go compilation |
| `pyproject.toml` | Python version requirement (`requires-python = ">=3.8"`) |
| `go.mod` | Go module definition (`go 1.22`) and dependency declarations |
| `docs/kittens/themes.rst` | Template reference for kitten documentation style patterns |

**Folders explored:**

| Folder Path | Depth Achieved | Purpose |
|-------------|---------------|---------|
| (root) | 1 | Repository structure assessment |
| `kittens/` | 1 | Kitten package discovery |
| `kittens/choose_fonts/` | 1 (all files) | Complete kitten implementation analysis |
| `docs/` | 1 | Documentation infrastructure assessment |
| `docs/kittens/` | 1 | Existing kitten docs inventory (14 pages found; no choose-fonts page) |
| `kitty/` | targeted | Entry points, constants, boss, fonts subsystem |
| `tools/config/` | targeted | Config patching and reload utilities |
| `tools/cmd/tool/` | targeted | CLI command tree registration |
| `tools/utils/` | targeted | Path resolution utilities |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens, images, or external files were included.

### 0.11.3 External References

No external URLs or Figma links were provided. All analysis is based entirely on the repository source code.

