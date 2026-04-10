# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new explanatory document** that traces the observable runtime path of keyboard input through Kitty's core components, grounded in behavior that can be verified by running the application with its built-in diagnostic options rather than by reading source code alone.

- **Category:** Create new documentation
- **Documentation Type:** Technical architecture walkthrough / Runtime behavior analysis
- **Target Audience:** A developer seeking to build intuition about Kitty's internal input-to-display pipeline by observing the running program

The user's request decomposes into the following concrete documentation requirements:

- **R-1 — Identify available tracing and debugging facilities.** Catalog every built-in command-line flag (e.g., `--debug-keyboard`, `--dump-commands`, `--debug-rendering`) and build-time option (e.g., `make debug`, `make debug-event-loop`) that can surface input-handling behavior at runtime without modifying source files.
- **R-2 — Describe the observable journey of a simple keypress.** Starting from the moment a key is physically pressed, trace what each diagnostic channel reveals about how the event enters GLFW, passes through key processing, gets encoded, crosses the PTY boundary, is echoed by the shell, returns through the VT parser, updates the screen model, and triggers a GPU render frame.
- **R-3 — Name the major subsystems that participate and their roles.** Identify — at a high level — which components appear to receive input first (platform/GLFW layer), which handle intermediate processing (key encoding → PTY → child monitor → VT parser → screen model), and which produce the updated display (GPU shader pipeline).
- **R-4 — Ground every claim in consistently repeatable runtime signals.** The document must distinguish observations that appear reliably during every keypress from behaviors that are inferred or incidental.
- **R-5 — Permit temporary helper scripts for observation only.** The user authorizes the creation of transient tracing wrappers or log-capture scripts to aid observation, provided they do not modify Kitty's source and are cleaned up afterward.

### 0.1.2 Special Instructions and Constraints

- **No source modifications:** The user explicitly requires "do not modify Kitty's source files." Documentation must be derived from external observation using existing debug flags and temporary helper scripts.
- **Cleanup requirement:** Any temporary artifacts created during exploration must be removed afterward.
- **Behavioral grounding:** "Keep the explanation grounded in behaviors that consistently appear while Kitty is running, rather than assumptions from reading the code alone." This means every architectural claim must cite a specific observable signal (debug log line, strace output, dump-command entry) visible at runtime.
- **High-level scope:** The user requests a high-level description of which parts receive input first, which handle intermediate processing, and how the display update is produced — not an exhaustive line-by-line code walkthrough.
- **Output format:** Per the project implementation rules, the document must be a Markdown file named `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory. The document must include thinking and rationale behind the answers and must not modify any existing repository files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **catalog debugging facilities** (R-1), we will create a new section in `blitzy/documentation/kitty_815df1e210e0.md` that enumerates every runtime and build-time diagnostic flag found in `kitty/cli.py`, `kitty/main.py`, `kitty/state.h`, and the `Makefile`, along with what each flag surfaces.
- To **trace a keypress** (R-2), we will describe the observable sequence of debug log messages emitted under `--debug-keyboard` and `--dump-commands`, mapping each message to the subsystem that produces it: the GLFW layer (`glfw/xkb_glfw.c` → `glfw/input.c`), the key processing layer (`kitty/keys.c`), the encoding layer (`kitty/key_encoding.c`), the PTY write path (`kitty/child-monitor.c`), the VT parser (`kitty/vt-parser.c`), the screen model (`kitty/screen.c`), and the GPU rendering pipeline (`kitty/shaders.c`).
- To **name subsystems and roles** (R-3), we will provide a layered architecture summary showing the three-phase pipeline (Input Reception → Intermediate Processing → Display Production) with the specific C and Python modules at each phase.
- To **ensure behavioral grounding** (R-4), every claim will be annotated with the diagnostic channel and specific log pattern that confirms it.
- To **handle helper scripts** (R-5), we will document the exact temporary scripts or commands used during exploration, their purpose, and their cleanup.

### 0.1.4 Inferred Documentation Needs

Based on repository analysis, the following implicit documentation needs are surfaced:

- **Thread architecture explanation:** The child-monitor's three-thread model (Main thread, I/O thread `KittyChildMon`, Talk thread) is central to understanding why input and output processing overlap. The debug output from `--debug-keyboard` appears on the main thread, while PTY reads occur on the I/O thread — this concurrency is observable through timing annotations in the debug log and must be explained.
- **Encoding mode divergence:** The `--debug-keyboard` output explicitly logs when keys are "sent encoded key to child" versus "sent key as text to child," revealing the protocol selection decision (legacy mode vs. Kitty Keyboard Protocol). This behavior change is observable and should be documented.
- **Render timing:** The `repaint_delay` and `input_delay` configuration options govern how quickly the screen visually updates after input. These timing parameters are visible through `--debug-rendering` output and explain the latency characteristics a user observes.
- **Mermaid diagram:** A high-level data-flow diagram showing the keypress journey from GLFW through the PTY to the screen is needed to consolidate the textual explanation visually.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation tree under `docs/` that serves as the official Kitty user manual, supplemented by root-level Markdown and AsciiDoc governance files. The documentation infrastructure is well-established but contains no existing document that specifically traces the observable runtime input-to-display pipeline from a debugging/tracing perspective.

- **Documentation framework:** Sphinx (version unpinned in `docs/requirements.txt`, using the `furo` theme)
- **Documentation generator configuration:** `docs/conf.py` — sets project metadata, custom lexers, man-page generation hooks, and the `furo` theme
- **Documentation build driver:** `docs/Makefile` — wires `help`, Sphinx targets, warning-as-error behavior, and `sphinx-autobuild` live preview via `make develop-docs`
- **Documentation dependencies:** `docs/requirements.txt` pins: `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **Diagram tools:** Mermaid-compatible diagrams are used in the tech spec; the existing Sphinx docs do not use Mermaid but use plain-text diagrams and screenshots
- **API documentation tools:** No dedicated API-doc generator (no JSDoc, Sphinx autodoc, or Godoc equivalent). In-code documentation relies on Python docstrings and C comments.

**Root-level documentation files:**

| File | Format | Purpose |
|------|--------|---------|
| `README.asciidoc` | AsciiDoc | Public-facing project introduction with links to website, FAQ, CI, packaging |
| `CONTRIBUTING.md` | Markdown | Contributor workflows for bug reporting and code submissions |
| `INSTALL.md` | Markdown | Pointers to source-build and binary-install docs |
| `SECURITY.md` | Markdown | Security reporting policy and release expectations |
| `CHANGELOG.rst` | reStructuredText | Pointer to the authoritative online changelog |
| `LICENSE` | Plain text | GPLv3 license text |

**Docs tree overview (first-level):**

| Path | Content |
|------|---------|
| `docs/index.rst` | Primary navigation and landing page |
| `docs/overview.rst` | Project introduction |
| `docs/quickstart.rst` | Quick start guide |
| `docs/invocation.rst` | Command-line reference |
| `docs/conf.rst` | Configuration file format |
| `docs/keyboard-protocol.rst` | Kitty Keyboard Protocol specification |
| `docs/performance.rst` | Performance tuning guide |
| `docs/basic.rst` | Basic usage guide |
| `docs/mapping.rst` | Keyboard shortcut mapping |
| `docs/actions.rst` | Actions reference |
| `docs/faq.rst` | Frequently asked questions |
| `docs/build.rst` | Building from source |
| `docs/kittens/` | Built-in kittens documentation (icat, diff, ssh, themes, etc.) |

### 0.2.2 Repository Code Analysis for Documentation

The following source files and directories were examined to understand the input-handling pipeline that the new document must describe. The search focused on components that produce observable diagnostic output at runtime.

**Input reception layer (Platform/GLFW):**

| Path | Relevance |
|------|-----------|
| `glfw/input.c` | Core GLFW input translation; `_glfwInputKeyboard()` normalizes key events |
| `glfw/xkb_glfw.c` | XKB keymap translation, compose sequences, IME handling on Linux |
| `glfw/x11_window.c` | X11 backend; calls `glfw_xkb_handle_key_event()` on key press/release |
| `glfw/wl_window.c` | Wayland backend; analogous key event path |
| `glfw/glfw3.h` | Defines `GLFW_DEBUG_KEYBOARD` hint constant |

**Key processing and encoding layer (Kitty core):**

| Path | Relevance |
|------|-----------|
| `kitty/glfw.c` | `key_callback()` receives GLFW events and calls `on_key_input()` |
| `kitty/keys.c` | `on_key_input()` — the central key dispatch function; emits `--debug-keyboard` output |
| `kitty/keys.py` | Python-side shortcut resolution via `Mappings.dispatch_possible_special_key()` |
| `kitty/key_encoding.c` | `encode_glfw_key_event()` — serializes key events to escape sequences |
| `kitty/key_encoding.py` | Python-side key encoding helpers |
| `kitty/state.h` | Defines `debug_input(...)` macro gated on `OPT(debug_keyboard)` |
| `kitty/boss.py` | `dispatch_possible_special_key()` routes through `Mappings`; `DumpCommands` class for `--dump-commands` |
| `kitty/mouse.c` | Mouse event processing; also uses `debug_input(...)` macro |

**PTY I/O and child monitoring layer:**

| Path | Relevance |
|------|-----------|
| `kitty/child-monitor.c` | `io_loop()` — I/O thread polling PTY fds; `parse_input()` — main thread consuming parsed data; `render()` — render scheduling |
| `kitty/child.c` / `kitty/child.py` | PTY/process spawning |

**VT parsing and screen update layer:**

| Path | Relevance |
|------|-----------|
| `kitty/vt-parser.c` | VT parser state machine; DUMP_COMMANDS build variant enables tracing |
| `kitty/screen.c` | Screen model; accepts parsed VT commands, updates cell buffers |
| `kitty/line.c`, `kitty/line-buf.c` | Line and cell buffer management |
| `kitty/history.c` | Scrollback buffer |
| `kitty/cursor.c` | Cursor position tracking |

**GPU rendering layer:**

| Path | Relevance |
|------|-----------|
| `kitty/shaders.c` | Central OpenGL rendering; `draw_cells()` produces frames |
| `kitty/shaders.py` | Shader source loading and compilation orchestration |
| `kitty/gl.c`, `kitty/gl-wrapper.c` | OpenGL infrastructure |
| `kitty/glyph-cache.c` | Glyph texture atlas management |
| 12 GLSL files (`kitty/cell_*.glsl`, `kitty/border_*.glsl`, etc.) | GPU shader programs |

**Debugging and diagnostic infrastructure:**

| Path | Relevance |
|------|-----------|
| `kitty/cli.py` (lines 972–1010) | CLI flag definitions: `--dump-commands`, `--debug-keyboard`, `--debug-rendering`, `--debug-font-fallback`, `--dump-bytes` |
| `kitty/main.py` (line 514) | Passes `debug_keyboard` and `debug_rendering` flags to GLFW init |
| `kitty/debug_config.py` | Diagnostic report generator (`kitty --debug-config`) |
| `kitty/client.py` | `--replay-commands` replay engine |
| `Makefile` (lines 25–30) | `make debug`, `make debug-event-loop`, `make asan` build targets |
| `setup.py` (line 722) | `vt-parser-dump.c` compiled with `DUMP_COMMANDS` define |
| `setup.py` (lines 1927–1933) | `--extra-logging=event-loop` build option |

### 0.2.3 Web Search Research Conducted

No external web searches are required for this task. All information needed to document the observable runtime input pipeline is directly available through:

- The repository's built-in debug flags and their implementation in source code
- The tech spec sections (4.3 Terminal Input/Output Pipeline, 5.1 High-Level Architecture, 4.1 High-Level System Workflow) that map the architectural layers
- The `--debug-keyboard` log format strings visible in `kitty/keys.c` (line 176) and `kitty/mouse.c` (lines 167, 767)

The documentation conventions for this project (Markdown in `blitzy/documentation/`) are established by the implementation rules, not by the existing Sphinx documentation framework.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The new document must describe the observable behavior of the following modules, grouped by their role in the input-to-display pipeline. Each module's traceability through Kitty's built-in diagnostics is noted.

**Phase 1 — Input Reception (Platform Layer)**

- Module: `glfw/input.c`
  - Public APIs: `_glfwInputKeyboard()`, `_glfwInputScroll()`, `_glfwInputMouseClick()`
  - Current documentation: General GLFW docs exist externally; no Kitty-specific input-reception trace documentation exists
  - Observable via: `--debug-keyboard` triggers `GLFW_DEBUG_KEYBOARD` hint (set in `kitty/glfw.c:1444`), which enables platform-level XKB/IME debug output
  - Documentation needed: Description of what the debug output reveals about event arrival from the OS

- Module: `glfw/xkb_glfw.c`
  - Public APIs: `glfw_xkb_handle_key_event()` — translates XKB keysyms to GLFW key events
  - Current documentation: No user-facing documentation for XKB translation behavior
  - Observable via: XKB debug messages emitted when `GLFW_DEBUG_KEYBOARD` is active
  - Documentation needed: Explanation of keymap resolution and compose processing visible in debug output

- Module: `kitty/glfw.c`
  - Public APIs: `key_callback()` (line 430) — GLFW-to-Kitty bridge
  - Current documentation: None
  - Observable via: This is the bridge that calls `on_key_input()`; the debug output from `keys.c` immediately follows
  - Documentation needed: Mention as the handoff point from GLFW to Kitty core

**Phase 2 — Key Processing and Encoding (Core Engine)**

- Module: `kitty/keys.c`
  - Public APIs: `on_key_input()` (line 166) — central key dispatch
  - Current documentation: No runtime trace documentation
  - Observable via: `--debug-keyboard` produces formatted log lines starting with `on_key_input:` showing GLFW key code, native code, action (PRESS/RELEASE/REPEAT), modifiers, text, and IME state
  - Documentation needed: Full description of the debug log format, shortcut dispatch decision, and encoding outcome messages ("sent encoded key to child", "handled as shortcut", "sent key as text to child")

- Module: `kitty/keys.py`
  - Public APIs: `Mappings.dispatch_possible_special_key()` (line 154)
  - Current documentation: `docs/mapping.rst` covers user-facing shortcut configuration but not the internal dispatch mechanism
  - Observable via: `--debug-keyboard` logs "matched action:" with function name when `boss.py:1580` fires
  - Documentation needed: Explanation of the shortcut lookup step visible in debug output

- Module: `kitty/key_encoding.c`
  - Public APIs: `encode_glfw_key_event()` — key-to-escape-sequence serializer
  - Current documentation: `docs/keyboard-protocol.rst` documents the protocol specification but not the runtime encoding behavior
  - Observable via: `--debug-keyboard` logs the encoded bytes after "sent encoded key to child:" with hex/character breakdown
  - Documentation needed: Explanation of the encoded output visible in debug logs (CSI u sequences, legacy escapes)

- Module: `kitty/boss.py`
  - Public APIs: `DumpCommands` class (line 232) — diagnostic callback sink for `--dump-commands` and `--dump-bytes`
  - Current documentation: CLI help text for `--dump-commands` in `kitty/cli.py`
  - Observable via: stdout output when running with `--dump-commands`
  - Documentation needed: Description of DumpCommands output format

**Phase 3 — PTY Transit and Child I/O (Child Monitor)**

- Module: `kitty/child-monitor.c`
  - Public APIs: `io_loop()` (line 1480) — I/O thread; `parse_input()` (line 451) — main thread parsing; `schedule_write_to_child()` (line 372) — write scheduling; `render()` (line 871) — render trigger
  - Current documentation: No user-facing documentation on the three-thread model
  - Observable via: `strace -e trace=write,read -p <pid>` can capture PTY read/write syscalls; `--debug-keyboard` shows the write side; `--dump-commands` shows the read side after VT parsing
  - Documentation needed: Explanation of the observable I/O thread cycle and how `input_delay` governs wakeup timing

**Phase 4 — VT Parsing and Screen Update (Output Pipeline)**

- Module: `kitty/vt-parser.c`
  - Public APIs: VT parser state machine; `parse_worker()` / `parse_worker_dump()` (DUMP_COMMANDS variant)
  - Current documentation: No runtime trace documentation
  - Observable via: `--dump-commands` activates the DUMP_COMMANDS build path, outputting every parsed VT command to stdout
  - Documentation needed: Description of the command dump format and what it reveals about how shell echo is processed

- Module: `kitty/screen.c`
  - Public APIs: Screen model state machine (cursor movement, cell writes, scrolling, mode toggling)
  - Current documentation: No runtime trace documentation
  - Observable via: `--dump-commands` output shows `draw`, `cursor_position`, `set_mode`, and other screen operations
  - Documentation needed: Explanation of screen operations visible in the dump output for a simple keypress echo

**Phase 5 — GPU Rendering (Display Production)**

- Module: `kitty/shaders.c`
  - Public APIs: `draw_cells()`, `render_os_window()`
  - Current documentation: No runtime trace documentation
  - Observable via: `--debug-rendering` enables OpenGL error checking and miscellaneous debug output
  - Documentation needed: High-level description of how the screen model's dirty flag triggers a render cycle

- Module: `kitty/glyph-cache.c`
  - Public APIs: Glyph atlas management
  - Current documentation: None
  - Observable via: `--debug-font-fallback` shows font selection; glyph caching is implicit
  - Documentation needed: Mention of glyph caching role in the pipeline

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist:

- **No existing runtime-trace walkthrough:** The docs tree contains protocol specifications (`docs/keyboard-protocol.rst`), performance guidance (`docs/performance.rst`), and configuration references (`docs/conf.rst`), but no document that walks a developer through what they would see if they ran Kitty with debug flags and typed a key.
- **No debugging guide:** While `kitty/cli.py` defines `--debug-keyboard`, `--dump-commands`, `--debug-rendering`, `--debug-font-fallback`, and `--dump-bytes`, there is no consolidated guide explaining when to use each flag and what output to expect.
- **No architecture-from-observation document:** The tech spec (Sections 4.3, 5.1) provides a thorough code-level architecture, but no document grounds that architecture in observable runtime signals.
- **No thread-model explanation for developers:** The three-thread architecture (Main, I/O/`KittyChildMon`, Talk) in `kitty/child-monitor.c` is well-documented in code comments but has no user-facing documentation.
- **No pipeline diagram tied to debug output:** Existing Mermaid diagrams in the tech spec show the data flow, but none map debug-log entries to pipeline stages.

All of these gaps will be addressed by the single new document `blitzy/documentation/kitty_815df1e210e0.md`.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single Markdown file whose internal structure mirrors the keypress journey. It is placed outside the Sphinx `docs/` tree per the implementation rules.

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction
        │   ├── Purpose and scope
        │   └── How to read this document
        ├── Available Debugging and Tracing Facilities
        │   ├── Runtime CLI flags (--debug-keyboard, --dump-commands, etc.)
        │   ├── Build-time options (make debug, --extra-logging)
        │   └── External tracing tools (strace, ltrace)
        ├── Observable Input-to-Display Pipeline
        │   ├── Phase 1: Platform Input Reception (GLFW layer)
        │   ├── Phase 2: Key Processing and Encoding (kitty core)
        │   ├── Phase 3: PTY Transit (child-monitor I/O thread)
        │   ├── Phase 4: VT Parsing and Screen Model Update
        │   └── Phase 5: GPU Rendering and Display Production
        ├── High-Level Architecture Diagram
        │   └── Mermaid diagram mapping subsystems to pipeline phases
        ├── Putting It Together: A Single Keypress Traced End-to-End
        │   └── Annotated walkthrough of a concrete "press 'a'" example
        ├── Thread Model and Timing Observations
        │   ├── Main thread vs. I/O thread vs. Talk thread
        │   └── input_delay and repaint_delay observable effects
        ├── Rationale and Methodology
        │   ├── Why each conclusion is grounded in runtime behavior
        │   └── Temporary helper scripts used (if any) and cleanup
        └── References
            └── Source files, debug flags, and tech spec sections cited
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach**

- Extract debug log format strings from `kitty/keys.c` (lines 172–262) and `kitty/mouse.c` (lines 167, 767) to document the exact output a developer would see under `--debug-keyboard`.
- Extract DumpCommands callback behavior from `kitty/boss.py` (lines 232–244) and the `DUMP_COMMANDS` preprocessor guards in `kitty/vt-parser.c` to document `--dump-commands` output.
- Extract CLI flag definitions from `kitty/cli.py` (lines 968–1010) to produce the debugging facilities catalog.
- Map the I/O thread cycle from `kitty/child-monitor.c` (`io_loop()` at line 1480, `parse_input()` at line 451, `render()` at line 871) to describe the thread-timing behavior.
- Derive the render trigger chain from `kitty/child-monitor.c` (lines 860–900) showing how `repaint_delay` and `input_delay` gate frame production.
- Source the encoding decision tree from `kitty/key_encoding.c` (`encode_glfw_key_event()`) and the protocol selection logic in `kitty/keys.c` (lines 253–270).

**Template Application**

The document follows a narrative Markdown template structured around the keypress journey, with each section grounding its claims in a specific debug output pattern. The style is explanatory technical prose with embedded code-formatted log excerpts and Mermaid diagrams.

**Documentation Standards**

- Markdown formatting with hierarchical headers (`#`, `##`, `###`)
- Mermaid diagrams for the pipeline overview and thread model
- Code blocks with syntax highlighting for debug log examples and CLI invocations
- Source citations as inline references in the format `Source: kitty/keys.c:176`
- Tables for structured comparisons (debug flags, subsystem roles)
- Consistent terminology aligned with Kitty's own naming (e.g., "child monitor" not "process manager"; "VT parser" not "terminal decoder")

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams:

- **Input-to-Display Pipeline Diagram:** A top-to-bottom flowchart showing the five phases from platform key event through GLFW, key processing, PTY, VT parser, screen model, to GPU render — with each node annotated by the debug flag that makes it observable.
- **Thread Model Diagram:** A swimlane diagram showing the Main thread (GLFW callbacks, shortcut dispatch, rendering), the I/O thread (PTY poll/read/write, buffer management), and the Talk thread (peer sockets), with arrows showing cross-thread communication via wakeup file descriptors and mutexes.
- **Encoding Decision Flowchart:** A compact decision tree showing Legacy Mode vs. Kitty Keyboard Protocol selection, grounded in the observable difference in `--debug-keyboard` output between "sent encoded key to child: ^[" (legacy) and "sent encoded key to child: ^[ [ ... u" (CSI u).

No screenshots are required. All visual elements are Mermaid-renderable.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

The following table comprehensively maps every documentation file to be created. Since the implementation rules prohibit modifying existing repository files, only CREATE mode is used for the deliverable, and REFERENCE mode is used for files that inform the document's content and style.

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/keys.c`, `kitty/key_encoding.c`, `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/shaders.c`, `kitty/boss.py`, `kitty/keys.py`, `kitty/glfw.c`, `kitty/cli.py`, `kitty/main.py`, `kitty/state.h`, `glfw/input.c`, `glfw/xkb_glfw.c`, `Makefile`, `setup.py` | Complete runtime-behavior document tracing keyboard input through Kitty's core components to display update, grounded in observable debug output from `--debug-keyboard`, `--dump-commands`, `--debug-rendering`, and related diagnostics |
| `docs/keyboard-protocol.rst` | REFERENCE | — | Used as reference for Kitty Keyboard Protocol encoding format and CSI u sequence specification |
| `docs/performance.rst` | REFERENCE | — | Used as reference for `repaint_delay` and `input_delay` configuration option documentation |
| `docs/conf.rst` | REFERENCE | — | Used as reference for configuration option naming conventions and documentation style |
| `docs/invocation.rst` | REFERENCE | — | Used as reference for CLI flag documentation patterns |
| `docs/mapping.rst` | REFERENCE | — | Used as reference for shortcut/key mapping documentation style |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Architecture Walkthrough / Runtime Behavior Analysis
Source Code: kitty/keys.c, kitty/key_encoding.c, kitty/child-monitor.c,
             kitty/vt-parser.c, kitty/screen.c, kitty/shaders.c,
             kitty/boss.py, kitty/keys.py, kitty/glfw.c, kitty/cli.py,
             kitty/main.py, kitty/state.h, glfw/input.c, glfw/xkb_glfw.c,
             Makefile, setup.py
Sections:
    - Introduction (purpose, scope, target audience)
    - Available Debugging and Tracing Facilities
        - Runtime CLI flags catalog (from kitty/cli.py:968-1010)
        - Build-time options catalog (from Makefile:25-30, setup.py:1925-1933)
        - External tracing tools guidance
    - Observable Input-to-Display Pipeline
        - Phase 1: Platform Input Reception
            - GLFW key callback (from kitty/glfw.c:430-440)
            - XKB/IME processing (from glfw/xkb_glfw.c)
        - Phase 2: Key Processing and Encoding
            - on_key_input dispatch (from kitty/keys.c:166-280)
            - Shortcut resolution (from kitty/keys.py:154, kitty/boss.py:1408)
            - Key encoding (from kitty/key_encoding.c)
        - Phase 3: PTY Transit
            - schedule_write_to_child (from kitty/child-monitor.c:372)
            - I/O thread write_to_child (from kitty/child-monitor.c:1443)
        - Phase 4: VT Parsing and Screen Update
            - I/O thread read_bytes and VT parsing (from kitty/vt-parser.c)
            - Screen model update (from kitty/screen.c)
        - Phase 5: GPU Rendering
            - Render scheduling (from kitty/child-monitor.c:871)
            - Shader pipeline execution (from kitty/shaders.c)
    - High-Level Architecture Diagram (Mermaid flowchart)
    - End-to-End Keypress Trace Example
    - Thread Model and Timing Observations
    - Rationale and Methodology
    - References
Diagrams:
    - Input-to-Display pipeline flowchart (Mermaid)
    - Thread model swimlane diagram (Mermaid)
    - Encoding decision flowchart (Mermaid)
Key Citations: kitty/keys.c, kitty/key_encoding.c, kitty/child-monitor.c,
               kitty/vt-parser.c, kitty/screen.c, kitty/shaders.c,
               kitty/boss.py, kitty/glfw.c, kitty/cli.py, kitty/state.h,
               glfw/input.c, glfw/xkb_glfw.c
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration files need to be modified. The new file is placed in `blitzy/documentation/`, which is independent of the Sphinx documentation tree under `docs/`. This is consistent with the implementation rule: "Do not modify any existing files in the source repository."

### 0.5.4 Cross-Documentation Dependencies

- **No navigation updates required:** The `blitzy/documentation/` directory is standalone and is not referenced in `docs/index.rst` or any Sphinx navigation.
- **No shared includes:** The new document does not depend on any Sphinx `.. include::` directives or shared content blocks.
- **Internal cross-references:** The document will internally cross-reference its own sections (e.g., "see Phase 2 above") but does not need to link to external Sphinx-built pages.
- **No index or glossary updates:** The document is self-contained with its own terminology definitions where needed.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The deliverable is a standalone Markdown file that does not require any documentation build tools for creation. However, the following tools are relevant to the exploration and verification workflow described in the document, and to the repository's existing documentation infrastructure.

**Tools relevant to this documentation exercise:**

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | (unpinned) | Repository's documentation site generator (`docs/requirements.txt`) — reference only |
| pip | furo | (unpinned) | Sphinx theme used by the repository — reference only |
| pip | sphinx-copybutton | (unpinned) | Sphinx extension — reference only |
| pip | sphinxext-opengraph | (unpinned) | Sphinx extension — reference only |
| pip | sphinx-inline-tabs | (unpinned) | Sphinx extension — reference only |
| pip | sphinx-autobuild | (unpinned) | Live documentation preview — reference only |
| system | strace | (system) | Linux syscall tracer — potentially used for PTY observation |
| system | python3 | ≥ 3.8 | Required runtime for Kitty (enforced in `pyproject.toml`: `requires-python = ">=3.8"`) |
| system | gcc / clang | (system) | C11 compiler required if building Kitty's debug variant |
| system | go | 1.22 | Go toolchain for static kitten binary (specified in `go.mod`) |

**Build-time tools needed to observe runtime behavior (if building from source):**

| Tool | Version Constraint | Purpose |
|------|-------------------|---------|
| Python | ≥ 3.8 (from `pyproject.toml`) | Build system and Python layer |
| GCC or Clang | C11 capable (enforced by `-std=c11` in `setup.py`) | Compile C extensions including DUMP_COMMANDS variant |
| Go | 1.22 (from `go.mod`) | Build the `kitten` static binary |
| FreeType | (system) | Font rasterization library |
| HarfBuzz | ≥ 1.5 | Text shaping library |
| OpenGL | 3.3+ | GPU rendering API |
| pkg-config | (system) | Dependency resolution for C libraries |

### 0.6.2 Documentation Reference Updates

No existing documentation links require updates. The new document `blitzy/documentation/kitty_815df1e210e0.md` is self-contained and does not require modifications to any existing link structures.

Since no existing files are modified, there are no link transformation rules to apply.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the input-to-display pipeline:**

| Pipeline Phase | Modules Involved | Currently Documented | Target | Gap |
|----------------|-----------------|---------------------|--------|-----|
| Platform Input Reception | `glfw/input.c`, `glfw/xkb_glfw.c`, `kitty/glfw.c` | 0% — no runtime-trace documentation exists | 100% | Full phase description needed |
| Key Processing and Encoding | `kitty/keys.c`, `kitty/keys.py`, `kitty/key_encoding.c`, `kitty/boss.py` | ~20% — `docs/keyboard-protocol.rst` covers protocol spec, `docs/mapping.rst` covers user shortcuts, but no runtime-trace walkthrough | 100% | Debug output explanation, encoding decision trace |
| PTY Transit (Child Monitor) | `kitty/child-monitor.c`, `kitty/child.c` | 0% — no user-facing documentation on I/O thread or PTY write/read cycle | 100% | Thread model, timing parameters, observable syscalls |
| VT Parsing and Screen Update | `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/line.c` | ~10% — tech spec documents the architecture but nothing traces observable output | 100% | DUMP_COMMANDS output format, screen operation trace |
| GPU Rendering | `kitty/shaders.c`, `kitty/gl.c`, GLSL files | ~15% — `docs/performance.rst` mentions repaint_delay and GPU rendering, but no debug-trace documentation | 100% | Render trigger chain, debug-rendering output description |
| Debugging Facilities (cross-cutting) | `kitty/cli.py`, `kitty/state.h`, `Makefile`, `setup.py` | ~25% — CLI help text exists for each flag, but no consolidated guide | 100% | Unified catalog with expected output descriptions |

**Overall coverage:** Currently approximately 12% of the observable runtime input pipeline is documented from a developer-tracing perspective. Target: 100% of the five pipeline phases and the debugging facility catalog.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every pipeline phase (five phases) must have a dedicated section describing what is observable through Kitty's diagnostic tools
- Every built-in debug flag (`--debug-keyboard`, `--dump-commands`, `--dump-bytes`, `--debug-rendering`, `--debug-font-fallback`) and build-time option (`make debug`, `make debug-event-loop`, `--extra-logging=event-loop`) must be cataloged with its purpose, invocation, and expected output format
- The end-to-end keypress trace must cover a concrete example (pressing the `a` key in a default shell) with annotated debug log output at each stage
- All three Mermaid diagrams (pipeline, thread model, encoding decision) must be present and syntactically valid

**Accuracy validation:**

- Debug log format strings must match the actual format specifiers in source code (e.g., the `on_key_input:` format in `kitty/keys.c:176`)
- Subsystem names must use Kitty's actual module names (e.g., `ChildMonitor`, not a made-up name)
- Thread names must match the actual `set_thread_name()` calls (e.g., `KittyChildMon` from `kitty/child-monitor.c:1490`)
- CLI flag names must match the definitions in `kitty/cli.py`

**Clarity standards:**

- Technical accuracy with accessible language — the document targets a developer who is new to Kitty's internals
- Progressive disclosure — begin with the high-level pipeline overview before diving into phase-by-phase detail
- Consistent terminology throughout, aligned with Kitty's source code naming conventions
- Every architectural claim must cite the specific debug channel and log pattern that supports it

**Maintainability:**

- Source citations reference file paths and line numbers for traceability
- The document is versioned alongside the branch (`kitty_815df1e210e0`) for clear provenance
- Standalone Markdown format requires no build tooling to read or update

### 0.7.3 Example and Diagram Requirements

| Requirement | Count | Verification Method |
|-------------|-------|-------------------|
| Mermaid diagrams | 3 (pipeline, threads, encoding decision) | Visual inspection of rendered Mermaid syntax |
| Annotated debug log examples | ≥ 5 (one per pipeline phase) | Cross-reference with format strings in source code |
| CLI invocation examples | ≥ 3 (`--debug-keyboard`, `--dump-commands`, `--debug-rendering`) | Validation against `kitty/cli.py` flag definitions |
| End-to-end trace walkthrough | 1 (pressing `a` key) | Consistency with all five pipeline phase descriptions |
| Subsystem role table | 1 (mapping modules to pipeline phases) | Cross-reference with tech spec Section 5.1 |


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/kitty_815df1e210e0.md` — the sole deliverable document

**Source files analyzed for documentation content (read-only, not modified):**

- `kitty/keys.c` — Key input dispatch and debug logging
- `kitty/keys.py` — Python-side shortcut resolution
- `kitty/key_encoding.c` — Key-to-escape-sequence encoding
- `kitty/key_encoding.py` — Python key encoding helpers
- `kitty/child-monitor.c` — I/O thread, PTY multiplexing, render scheduling
- `kitty/vt-parser.c` — VT parser state machine, DUMP_COMMANDS variant
- `kitty/screen.c` — Screen model state machine
- `kitty/line.c`, `kitty/line-buf.c` — Line/cell buffer management
- `kitty/cursor.c` — Cursor position tracking
- `kitty/history.c` — Scrollback buffer
- `kitty/shaders.c` — OpenGL rendering pipeline
- `kitty/shaders.py` — Shader orchestration
- `kitty/gl.c`, `kitty/gl-wrapper.c` — OpenGL infrastructure
- `kitty/glyph-cache.c` — Glyph texture atlas
- `kitty/boss.py` — Central controller, DumpCommands class
- `kitty/window.py` — Per-window behavior and input forwarding
- `kitty/main.py` — Application startup, debug flag propagation
- `kitty/glfw.c` — GLFW-to-Kitty bridge, key_callback
- `kitty/cli.py` — CLI flag definitions
- `kitty/state.h` — Debug macros, global state structure
- `kitty/mouse.c` — Mouse input processing
- `kitty/debug_config.py` — Diagnostic report generator
- `kitty/client.py` — Replay commands engine
- `kitty/constants.py` — Version and path constants
- `glfw/input.c` — GLFW input layer
- `glfw/xkb_glfw.c` — XKB keyboard integration
- `glfw/x11_window.c` — X11 backend key event path
- `glfw/wl_window.c` — Wayland backend key event path
- `glfw/glfw3.h` — GLFW API header, GLFW_DEBUG_KEYBOARD constant
- `Makefile` — Build targets (debug, debug-event-loop, asan)
- `setup.py` — Build system, DUMP_COMMANDS compilation, extra-logging option
- `pyproject.toml` — Python version constraints
- `go.mod` — Go version constraint
- `docs/requirements.txt` — Documentation tool dependencies
- `docs/conf.py` — Sphinx configuration
- `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl` — Cell shader sources (referenced for rendering description)

**Documentation reference files (read-only):**

- `docs/keyboard-protocol.rst` — Keyboard protocol spec reference
- `docs/performance.rst` — Performance tuning reference
- `docs/conf.rst` — Configuration format reference
- `docs/invocation.rst` — CLI reference
- `docs/mapping.rst` — Shortcut mapping reference

**Temporary artifacts (permitted, must be cleaned up):**

- Any helper scripts created for tracing observation (e.g., shell wrappers that invoke `strace` or capture `--debug-keyboard` output)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No changes to any `.c`, `.py`, `.h`, `.glsl`, `.m`, `.go`, or any other source file in the repository. The user explicitly stated: "do not modify Kitty's source files."
- **Existing documentation modifications:** No changes to any file under `docs/`, `README.asciidoc`, `CONTRIBUTING.md`, `INSTALL.md`, `SECURITY.md`, `CHANGELOG.rst`, or `LICENSE`.
- **Test file modifications:** No changes to files under `kitty_tests/`.
- **Build system modifications:** No changes to `Makefile`, `setup.py`, `pyproject.toml`, `go.mod`, `go.sum`, or any build configuration.
- **Feature additions or code refactoring:** The document describes existing behavior; it does not propose changes.
- **Deployment configuration changes:** No CI/CD, packaging, or release changes.
- **macOS-specific Cocoa input path:** While mentioned for completeness, the document focuses on the Linux (X11/Wayland) input path where `--debug-keyboard` output is richest due to XKB tracing. macOS specifics (CoreText, `cocoa_window.m`) are noted but not deeply traced.
- **Graphics Protocol, Clipboard Protocol, and other protocol handlers:** These are downstream of the VT parser but are not exercised by simple keyboard input. They are outside the scope of a "press a few simple keys" trace.
- **Kittens framework, remote control, and shell integration internals:** These are extension-layer features not involved in the basic input-to-display pipeline for simple key presses.
- **Go tools (`tools/`):** Not involved in the runtime keyboard input pipeline.


## 0.9 Rules for Documentation

### 0.9.1 User-Specified Rules

The following rules are derived directly from the user's requirements and the project's implementation rules:

- **"Do not modify Kitty's source files."** — The new document must be created without any changes to any existing file in the repository. All analysis is read-only.
- **"Clean up any temporary artifacts afterward."** — Any temporary helper scripts, log capture files, or tracing wrappers created during the exploration phase must be removed before the task is considered complete.
- **"Keep the explanation grounded in behaviors that consistently appear while Kitty is running, rather than assumptions from reading the code alone."** — Every claim about the input pipeline must cite a specific observable signal: a `--debug-keyboard` log line, a `--dump-commands` entry, a `strace` syscall, or a `--debug-rendering` message. Speculative statements based solely on code reading must be clearly flagged as inferred.
- **"Describe at a high level which parts of the system appear to receive the input first, which parts seem to handle the intermediate processing, and how the updated display is ultimately produced."** — The document must be organized around this three-part structure (reception → processing → display) rather than following source-file order or module hierarchy.
- **"You may use temporary helper scripts or tools to capture signals during this exploration."** — Temporary tooling for observation is permitted but must not alter Kitty's source code and must be cleaned up.
- **"Create a new markdown document named `<source_branch_name>.md`."** — The output file must be named `kitty_815df1e210e0.md` and placed in the `blitzy/documentation/` directory.
- **"Provide thinking / rationale behind the answers."** — The document must include a section explaining the methodology and rationale for each conclusion, making the reasoning transparent.
- **"Do not make assumptions, base your answers on the code as the truth."** — Claims must be grounded in verifiable source code artifacts (debug format strings, function call chains, preprocessor guards), not in external documentation or third-party descriptions of similar systems.
- **"Do not modify any existing files in the source repository."** — Reinforces the read-only constraint across all file types.

### 0.9.2 Documentation-Specific Conventions

The following conventions apply to the content and formatting of the deliverable document:

- **Format:** Markdown with GitHub-compatible Mermaid diagram fencing
- **Citation style:** Inline source references using `Source: <path>:<line>` format
- **Debug output formatting:** Debug log examples presented in fenced code blocks with no syntax highlighting (plain text) to accurately represent terminal output
- **Terminology consistency:** Use Kitty's own names for components — "VT parser" (not "terminal parser"), "child monitor" (not "process manager"), "Boss controller" (not "main controller"), "glyph cache" (not "font cache")
- **Diagram style:** Mermaid `flowchart TD` (top-down) for pipeline diagrams; `flowchart LR` (left-right) for decision trees
- **Section depth:** Maximum three heading levels (`#`, `##`, `###`) in the output document
- **No hardcoded log output:** Debug log examples should be presented as representative patterns (e.g., `on_key_input: glfw key: 0x61 native_code: 0x26 action: PRESS ...`) with placeholders for variable values, rather than copy-pasted from a specific run, since exact values vary by platform and locale


## 0.10 References

### 0.10.1 Source Files and Folders Searched

The following files and folders were systematically examined to derive the conclusions in this Agent Action Plan. They are grouped by the pipeline phase they inform.

**Platform Input Reception Layer:**

| Path | Purpose of Examination |
|------|----------------------|
| `glfw/input.c` | GLFW input translation; `_glfwInputKeyboard()` entry point for normalized key events |
| `glfw/xkb_glfw.c` | XKB keymap translation, compose sequences, IME handling; `glfw_xkb_handle_key_event()` |
| `glfw/x11_window.c` | X11 backend; confirmed calls to `glfw_xkb_handle_key_event()` on key press/release (lines 1254, 1293) |
| `glfw/wl_window.c` | Wayland backend; analogous key event path |
| `glfw/glfw3.h` | Confirmed `GLFW_DEBUG_KEYBOARD` constant definition (line 1159) |
| `glfw/` (folder) | Full GLFW integration layer structure and backend inventory |

**Key Processing and Encoding Layer:**

| Path | Purpose of Examination |
|------|----------------------|
| `kitty/glfw.c` | `key_callback()` bridge from GLFW to `on_key_input()` (lines 430-440); GLFW_DEBUG_KEYBOARD hint init (line 1444) |
| `kitty/keys.c` | `on_key_input()` central dispatch (lines 166-280); debug format strings (lines 172-176); encoding and write-to-child logic (lines 253-270) |
| `kitty/keys.py` | `Mappings.dispatch_possible_special_key()` shortcut resolution (line 154) |
| `kitty/key_encoding.c` | `encode_glfw_key_event()` serialization (full file); legacy vs. Kitty protocol decision tree |
| `kitty/key_encoding.py` | Python key encoding helpers |
| `kitty/boss.py` | `dispatch_possible_special_key()` (line 1408); `DumpCommands` class (lines 232-244); debug_keyboard logging (line 1580) |
| `kitty/state.h` | `debug_input()` macro definition (line 15); `debug_keyboard` option flag (line 80) |
| `kitty/mouse.c` | Mouse debug logging (lines 167, 767) |

**PTY I/O and Child Monitoring Layer:**

| Path | Purpose of Examination |
|------|----------------------|
| `kitty/child-monitor.c` | `io_loop()` I/O thread (lines 1480-1575); `parse_input()` main-thread parsing (lines 451-510); `schedule_write_to_child()` (line 372); `write_to_child()` (line 1443); `render()` trigger (lines 860-900); `input_delay`/`repaint_delay` timing (lines 445-446, 874-876, 1508, 1563-1569) |
| `kitty/child.c` | PTY/process spawning |
| `kitty/child.py` | Python-side child process management |

**VT Parsing and Screen Model Layer:**

| Path | Purpose of Examination |
|------|----------------------|
| `kitty/vt-parser.c` | VT parser state machine; DUMP_COMMANDS preprocessor guards (lines 44, 500, 537, 953, 1370, 1396, 1448, 1491, 1499) |
| `kitty/screen.c` | Screen model state machine (full file summary) |
| `kitty/line.c` | Line buffer management |
| `kitty/line-buf.c` | Line buffer container |
| `kitty/cursor.c` | Cursor position tracking |
| `kitty/history.c` | Scrollback buffer |

**GPU Rendering Layer:**

| Path | Purpose of Examination |
|------|----------------------|
| `kitty/shaders.c` | OpenGL rendering pipeline; `draw_cells()` frame production |
| `kitty/shaders.py` | Shader source loading and compilation orchestration |
| `kitty/gl.c` | OpenGL infrastructure |
| `kitty/gl-wrapper.c` | GLAD/GLFW/OpenGL binding |
| `kitty/glyph-cache.c` | Glyph texture atlas management |
| `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl` | Cell rendering shaders |

**Debugging and Diagnostic Infrastructure:**

| Path | Purpose of Examination |
|------|----------------------|
| `kitty/cli.py` | CLI flag definitions: `--dump-commands` (line 972), `--debug-keyboard` (lines 996-997), `--debug-rendering` (lines 986-990), `--debug-font-fallback` (lines 1003-1005), `--dump-bytes` (line 985) |
| `kitty/main.py` | Debug flag propagation to GLFW init (line 514); AppRunner startup (line 249) |
| `kitty/debug_config.py` | Diagnostic report generator |
| `kitty/client.py` | `--replay-commands` replay engine |
| `Makefile` | Build targets: `debug` (line 25), `debug-event-loop` (line 28), `asan` (line 31) |
| `setup.py` | `vt-parser-dump.c` with DUMP_COMMANDS (line 722); `--extra-logging` (lines 1925-1933) |

**Configuration and Metadata:**

| Path | Purpose of Examination |
|------|----------------------|
| `pyproject.toml` | Python version constraint: `requires-python = ">=3.8"` |
| `go.mod` | Go version: `go 1.22` |
| `kitty/constants.py` | Kitty version: `0.35.2` (line 25) |
| `docs/requirements.txt` | Sphinx documentation dependencies |
| `docs/conf.py` | Sphinx configuration and project metadata |
| `kitty/window.py` | Per-window input forwarding and behavior |
| `key_encoding.json` | Static key-name-to-code lookup table |

**Documentation Reference Files:**

| Path | Purpose of Examination |
|------|----------------------|
| `docs/keyboard-protocol.rst` | Kitty Keyboard Protocol specification — used for encoding format reference |
| `docs/performance.rst` | Performance tuning — used for `repaint_delay`/`input_delay` documentation |
| `docs/conf.rst` | Configuration format reference |
| `docs/invocation.rst` | CLI flag documentation patterns |
| `docs/mapping.rst` | Shortcut mapping documentation style |
| `README.asciidoc` | Project overview and external links |

**Folder-Level Exploration:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| `` (root) | Level 0 | Repository structure, top-level files, and major subtree identification |
| `kitty/` | Level 1 | Core application tree; all C extensions, Python modules, GLSL shaders |
| `docs/` | Level 1 | Sphinx documentation tree; existing manual structure assessment |
| `glfw/` | Level 1 | GLFW integration layer; platform backends, input handling |

### 0.10.2 Tech Spec Sections Referenced

| Section | Content Used |
|---------|-------------|
| 1.1 Executive Summary | Project overview, three-language architecture, version identification (0.35.2) |
| 3.1 Programming Languages | C11 standard, Python ≥ 3.8, Go 1.22, GLSL shader inventory, compiler flags |
| 4.1 High-Level System Workflow | Application lifecycle, core event loop architecture, timing considerations |
| 4.3 Terminal Input/Output Pipeline | Keyboard input path, VT parser dispatch, GPU rendering pipeline, shader stage details |
| 5.1 High-Level Architecture | Six-layer architecture, core components table, data flow description, external integration points |

### 0.10.3 Attachments

No attachments were provided for this project. No Figma URLs or external design assets are referenced.

### 0.10.4 External URLs

No external URLs were provided by the user. The document is derived entirely from repository analysis and tech spec content.


