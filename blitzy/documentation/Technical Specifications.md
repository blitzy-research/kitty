# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive explanatory document** that answers a series of deeply interrelated questions about Kitty's live terminal interaction pipeline — specifically how raw, concurrent input streams (keystrokes, paste bursts, resize signals, shell integration escape sequences) are received, ordered, disambiguated, and reconciled into a coherent terminal state. The user is onboarding into the Kitty codebase and needs an authoritative narrative that traces the full lifecycle of mixed input from the moment it arrives through to when the interface settles after processing.

**Category:** Create new documentation
**Documentation type:** Deep-dive technical explainer / Architecture walkthrough

The requirements, restated with enhanced technical clarity:

- **Input entry point identification** — Where does raw input first enter the Kitty process? The answer involves the GLFW platform backend delivering events through `glfw/` to `kitty/keys.c` and `kitty/mouse.c`, and child PTY output entering through the I/O thread in `kitty/child-monitor.c`.
- **Concurrent stream arbitration** — When keystrokes, paste bursts, and resize signals arrive simultaneously, what decides handling order? The answer lies in the three-thread architecture (Main, I/O, Talk) of the Child Monitor, the `poll()` loop in `io_loop()`, the `process_global_state()` main-thread callback, and the configurable `input_delay` / `repaint_delay` timing.
- **Shell integration interleaving** — How are OSC 133 prompt markers, OSC 7 CWD notifications, and other shell integration escapes handled when mixed with ordinary output? The answer is in the VT parser's dispatch architecture (`kitty/vt-parser.c`) where `shell_prompt_marking()` in `kitty/screen.c` processes markers inline within the same byte-stream classification pass.
- **Backpressure and degraded conditions** — How does the system behave when the VT parser buffer fills, when a remote connection is unstable, or when a session is paused and resumed? The answer involves the 1 MB ring buffer in `vt-parser.c`, the `vt_parser_has_space_for_input()` flow-control gate, and the `screen_pause_rendering()` / synchronized update (DCS `=1s` / `=2s`) mechanisms.
- **End-to-end coherence** — How do all these moving parts avoid drifting out of sync, and what keeps the rhythm? The answer is the deterministic `process_global_state()` cycle: pending resizes → parse input → render → process closes, with timing governed by `input_delay`, `repaint_delay`, and `resize_debounce_time`.

### 0.1.2 Special Instructions and Constraints

- **Repository must remain unchanged:** The user explicitly states: "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." No existing files may be modified.
- **Implementation rule — SWE-AtlasQnA-Repo:** Create a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory. The document must comprehensively answer the posed questions, provide thinking and rationale, base all answers on code as ground truth, and must not modify any existing repository files.
- **Temporary scripts allowed for observation:** The user permits temporary scripts for observation purposes, but they must be cleaned up. Since this is a documentation-only task, no observation scripts are needed.
- **Answer grounded in code:** All claims must cite specific source files and, where possible, line ranges as evidence.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **explain the input entry point**, we will create a section in `blitzy/documentation/kitty_815df1e210e0.md` tracing the GLFW → `keys.c` → `key_encoding.c` → PTY write path for keyboard input, the `mouse.c` path for mouse events, and the `io_loop()` → `read_bytes()` → `vt_parser_create_write_buffer()` → `vt_parser_commit_write()` path for child output.
- To **explain concurrent stream arbitration**, we will document the `poll()` multiplexing in the I/O thread, the `input_delay`-gated wakeup of the main loop, and the deterministic `process_global_state()` ordering: resizes first, then input parsing, then rendering, then close processing.
- To **explain shell integration interleaving**, we will trace how OSC 133 A/C/D markers flow through the VT parser's `consume_input()` → `shell_prompt_marking()` path in `screen.c`, showing that they are processed inline within the same classification pass as ordinary text, CSI sequences, and other escapes.
- To **explain backpressure behavior**, we will document the 1 MB (`BUF_SZ`) parser buffer, the `vt_parser_has_space_for_input()` backpressure gate that removes `POLLIN` from the I/O thread's poll set when the buffer is full, and the synchronized update / paused rendering mechanisms.
- To **explain end-to-end coherence**, we will provide a Mermaid diagram of the full cycle and explain the timing parameters (`input_delay`, `repaint_delay`, `resize_debounce_time`, `sync_to_monitor`) that govern the rhythm.

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The three-thread architecture (Main, I/O, Talk) in `kitty/child-monitor.c` is a critical but non-obvious coordination mechanism that needs a clear diagram and narrative.
- The `ParseData` struct in `kitty/vt-parser.h` carries the `input_read`, `write_space_created`, `has_pending_input`, and `time_since_new_input` fields that form the feedback loop between the parser and the I/O thread — this needs explicit documentation.
- The paste pathway (`Window.paste_text()` in `kitty/window.py` → `screen.paste()` / `screen.paste_bytes()`) involves bracketed paste mode detection and sanitization — relevant because paste bursts were explicitly mentioned.
- The resize path through `process_pending_resizes()` in `child-monitor.c`, which involves `resize_debounce_time` (with distinct `on_end` and `on_pause` thresholds) and `pty_resize()` for SIGWINCH delivery, needs documentation because resize signals were explicitly mentioned.
- The `screen_pause_rendering()` function in `kitty/screen.c` snapshots the entire visual state (cursor, colors, line buffer, selections, graphics) into a `paused_rendering` struct so the display remains frozen while a synchronized update is in flight — this is directly relevant to the "unseen conductor" metaphor the user invoked.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation framework with extensive user-facing manuals, but no existing deep-dive document that traces the live runtime behavior of the terminal interaction pipeline end to end.

- **Documentation framework:** Sphinx with reStructuredText (`.rst`) files, configured in `docs/conf.py`
- **Theme:** Furo theme with sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, and sphinx-autobuild extensions
- **Documentation build driver:** `docs/Makefile` providing standard Sphinx targets plus `develop-docs` live-preview
- **Documentation dependencies:** Pinned in `docs/requirements.txt`: `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **Diagram tools detected:** Mermaid diagrams are used in the existing tech spec but are not natively part of the Sphinx documentation; the codebase uses Sphinx roles and custom lexers in `docs/conf.py`
- **API documentation tools:** No standalone API documentation generator (e.g., Sphinx autodoc, JSDoc) is used; the `docs/conf.py` defines custom Sphinx roles for configuration options, keyboard shortcuts, and kitten references

Existing documentation relevant to the terminal interaction pipeline:

| Document | Path | Coverage of Pipeline |
|----------|------|---------------------|
| Performance guide | `docs/performance.rst` | Discusses `repaint_delay`, `input_delay`, `sync_to_monitor` at a user-facing level; mentions threaded rendering and SIMD parsing |
| Shell integration guide | `docs/shell-integration.rst` | Documents user-facing shell integration features (prompt jumping, output viewing, cursor shape) but not the internal OSC 133 dispatch flow |
| Keyboard protocol spec | `docs/keyboard-protocol.rst` | Specifies the Kitty keyboard protocol from a client/application perspective, not the internal key processing path |
| Configuration reference | `docs/conf.rst` | Documents all configuration options including `input_delay`, `repaint_delay`, `resize_debounce_time` |
| Remote control protocol | `docs/rc_protocol.rst` | Documents the JSON remote control protocol and transport |

**Key finding:** No existing document traces the internal runtime behavior of concurrent input processing, VT parser dispatch, shell integration interleaving, or backpressure management. The new document fills a genuine gap.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify source code relevant to the pipeline questions:

- **Input entry points:** `kitty/keys.c` (keyboard event processing), `kitty/mouse.c` (mouse event processing), `kitty/child-monitor.c` (I/O thread, PTY read/write multiplexing)
- **VT parser and dispatch:** `kitty/vt-parser.c` (state machine, `consume_input()`, `run_worker()`), `kitty/vt-parser.h` (`ParseData` struct, API declarations)
- **Screen model:** `kitty/screen.c` (`shell_prompt_marking()`, `process_cwd_notification()`, `screen_pause_rendering()`), `kitty/screen.h` (`paused_rendering` struct)
- **Global state and timing:** `kitty/state.c`, `kitty/state.h` (`global_state`, `OPT()` macro for accessing `input_delay`, `repaint_delay`, `resize_debounce_time`)
- **Event loop infrastructure:** `kitty/loop-utils.c` and `kitty/loop-utils.h` (`LoopData`, signal handling, wakeup pipes/eventfds)
- **Shell integration setup:** `kitty/shell_integration.py` (`modify_shell_environ()`), `shell-integration/bash/kitty.bash`, `shell-integration/zsh/kitty.zsh`, `shell-integration/fish/`
- **Window and paste handling:** `kitty/window.py` (`Window.paste_text()`, `cmd_output_marking()`, `write_to_child()`)
- **Boss controller:** `kitty/boss.py` (`on_window_resize()`, `on_child_death()`, `dispatch_action()`, `peer_message_received()`)
- **Configuration definitions:** `kitty/options/definition.py` (option declarations for `repaint_delay`, `input_delay`, `sync_to_monitor`, `resize_debounce_time`)
- **Control codes and modes:** `kitty/control-codes.h`, `kitty/modes.h` (`PENDING_UPDATE` mode `2026`)
- **Key encoding:** `kitty/key_encoding.c`, `kitty/key_encoding.py` (Kitty keyboard protocol encoding)

Key directories examined:
- `kitty/` — Core application tree (~100+ C and Python files)
- `shell-integration/` — Shell-specific integration scripts (bash, zsh, fish, ssh)
- `docs/` — Existing Sphinx documentation tree
- `glfw/` — Vendored GLFW 3.4 fork with custom input handling extensions

### 0.2.3 Web Search Research Conducted

No external web research was required for this task. All answers are grounded exclusively in the codebase, which is the authoritative source of truth per the user's implementation rules. The existing tech spec sections (4.1, 4.3, 4.5, 4.7, 4.10, 5.1, 5.2) provided additional architectural context that aligns with the source code analysis.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user's questions map to the following modules and their specific concerns:

- **Module: `kitty/child-monitor.c`**
  - Public APIs / key functions: `io_loop()`, `read_bytes()`, `write_to_child()`, `parse_input()`, `process_global_state()`, `process_pending_resizes()`, `do_parse()`, `wakeup_io_loop()`, `pty_resize()`
  - Current documentation: User-facing docs mention threaded rendering in `docs/performance.rst` but never explain the three-thread architecture, the poll loop, or the `input_delay`-gated wakeup mechanism
  - Documentation needed: Full narrative of the I/O thread loop, the main thread's `process_global_state()` cycle, and the feedback between them

- **Module: `kitty/vt-parser.c` + `kitty/vt-parser.h`**
  - Key functions: `run_worker()`, `consume_input()`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `vt_parser_has_space_for_input()`
  - `ParseData` struct fields: `input_read`, `write_space_created`, `has_pending_input`, `time_since_new_input`
  - Current documentation: The VT parser is referenced in the tech spec (sections 4.3, 5.2.4) but no existing document explains the 1 MB buffer management, the backpressure gate, or the `input_delay` batch-and-flush decision logic
  - Documentation needed: Buffer lifecycle, backpressure mechanics, timing-based flush decisions

- **Module: `kitty/screen.c`**
  - Key functions: `shell_prompt_marking()`, `process_cwd_notification()`, `screen_pause_rendering()`, `screen_check_pause_rendering()`
  - Current documentation: `docs/shell-integration.rst` documents user-visible features but not the internal `shell_prompt_marking()` dispatch or the `paused_rendering` snapshot mechanism
  - Documentation needed: How OSC 133 markers are classified and processed inline, how synchronized updates freeze and restore visual state

- **Module: `kitty/keys.c` + `kitty/keys.py`**
  - Key functions: `is_modifier_key()`, `convert_glfw_key_event_to_python()`, `dispatch_possible_special_key()`
  - Current documentation: `docs/keyboard-protocol.rst` covers the protocol from a client perspective; `docs/mapping.rst` covers user key bindings
  - Documentation needed: Internal path from GLFW key event → `keys.c` processing → Python dispatch → PTY write

- **Module: `kitty/window.py`**
  - Key functions: `paste_text()`, `write_to_child()`, `cmd_output_marking()`, `send_key_sequence()`
  - Current documentation: Documented as part of remote control and kitten APIs but not as part of the input pipeline
  - Documentation needed: How paste bursts flow through bracketed paste detection and sanitization

- **Module: `kitty/boss.py`**
  - Key functions: `on_window_resize()`, `dispatch_action()`, `on_child_death()`
  - Current documentation: Referenced in tech spec section 5.2.2 but the resize → layout → PTY resize path is not traced end to end
  - Documentation needed: How resize events propagate from GLFW through the Boss to `pty_resize()`

- **Module: `kitty/loop-utils.c` + `kitty/loop-utils.h`**
  - Key structures: `LoopData`, signal handling via `signalfd()` or self-pipe, `wakeup_loop()`
  - Current documentation: None
  - Documentation needed: How the signal infrastructure connects SIGINT/SIGTERM/SIGCHLD/SIGUSR1 to the I/O thread

- **Module: `kitty/shell_integration.py` + `shell-integration/bash/kitty.bash`**
  - Key functions: `modify_shell_environ()`, `setup_bash_env()`, `setup_zsh_env()`, `setup_fish_env()`
  - Current documentation: `docs/shell-integration.rst` covers user-facing setup but not the internal environment mutation pipeline
  - Documentation needed: How integration scripts inject OSC 133 markers into the child's output stream

- **Configuration options requiring documentation:**
  - `repaint_delay` (default: 10ms) — Controls minimum time between render cycles
  - `input_delay` (default: 3ms) — Controls batching of input before processing
  - `sync_to_monitor` (default: yes) — Controls vsync alignment
  - `resize_debounce_time` (default: 0.1s on_end, 0.5s on_pause) — Controls resize event debouncing

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing document traces the runtime pipeline end to end** — From raw GLFW event delivery through I/O thread multiplexing, VT parser classification, screen model update, to GPU render cycle
- **The three-thread architecture (Main, I/O, Talk) has no narrative documentation** — Only the tech spec provides a table summary
- **Backpressure and flow control are undocumented** — The `vt_parser_has_space_for_input()` gate, the `BUF_SZ` (1 MB) buffer limit, and the `write_space_created` feedback path have no existing documentation
- **Synchronized update / paused rendering is undocumented externally** — The DCS `=1s`/`=2s` pending mode and `screen_pause_rendering()` snapshot mechanism appear only in code comments
- **The `input_delay` batch-and-flush decision is never explained** — The logic in `run_worker()` that decides whether to process input immediately or defer based on `input_delay` and buffer fullness has no documentation
- **Shell integration's internal pathway from OSC emission to screen marking is undocumented** — How `shell_prompt_marking()` receives OSC 133 markers and annotates line attributes is not covered
- **Resize debounce logic is undocumented** — The dual-threshold (`on_end` / `on_pause`) mechanism in `process_pending_resizes()` and the Wayland-specific flicker avoidance logic have no documentation


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The single output document `blitzy/documentation/kitty_815df1e210e0.md` will be a self-contained Markdown narrative structured as follows:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction & Orientation
        │   └── What this document covers, who it is for
        ├── Where Input First Enters the System
        │   ├── GLFW platform events → keys.c / mouse.c
        │   ├── Child PTY output → io_loop() → vt_parser
        │   └── Entry point diagram
        ├── The Three-Thread Architecture
        │   ├── Main thread responsibilities
        │   ├── I/O thread (KittyChildMon) responsibilities
        │   ├── Talk thread responsibilities
        │   └── Thread coordination mechanisms
        ├── The I/O Thread's Poll Loop
        │   ├── File descriptor setup and poll() mechanics
        │   ├── Signal handling (SIGCHLD, SIGINT, etc.)
        │   ├── read_bytes() and write_to_child() paths
        │   └── input_delay-gated main loop wakeup
        ├── The Main Thread's Processing Cycle
        │   ├── process_global_state() ordering
        │   ├── Resize processing (process_pending_resizes)
        │   ├── Input parsing (parse_input → do_parse)
        │   ├── Render scheduling
        │   └── Close processing
        ├── The VT Parser: Classifying the Byte Stream
        │   ├── Buffer architecture (1 MB ring)
        │   ├── run_worker() batch-and-flush decision
        │   ├── Sequence classification and dispatch
        │   └── Backpressure: vt_parser_has_space_for_input()
        ├── Shell Integration In the Stream
        │   ├── How OSC 133 markers reach the parser
        │   ├── shell_prompt_marking() dispatch (A/C/D)
        │   ├── OSC 7 CWD notifications
        │   └── Alignment with ordinary text and CSI sequences
        ├── Paste Bursts and Bracketed Paste
        │   ├── Window.paste_text() → screen.paste()
        │   ├── Bracketed paste mode detection
        │   └── Sanitization and PTY write
        ├── Resize Signals: From GLFW to PTY
        │   ├── GLFW resize event → live_resize tracking
        │   ├── Debounce logic (on_end / on_pause thresholds)
        │   ├── pty_resize() and SIGWINCH delivery
        │   └── Wayland-specific considerations
        ├── Synchronized Updates and Paused Rendering
        │   ├── DCS =1s / =2s pending mode
        │   ├── screen_pause_rendering() state snapshot
        │   ├── Timeout-based safety net
        │   └── Visual coherence during bulk updates
        ├── Timing Parameters and Their Interplay
        │   ├── input_delay — batching threshold
        │   ├── repaint_delay — render scheduling floor
        │   ├── resize_debounce_time — resize settling
        │   ├── sync_to_monitor — vsync alignment
        │   └── How they interact under load
        ├── Degraded Conditions and Recovery
        │   ├── Buffer full / backpressure scenario
        │   ├── Unstable remote connection behavior
        │   ├── Session pause and resume
        │   └── Error isolation and graceful degradation
        └── Summary: The Full Journey
            └── End-to-end Mermaid diagram with narrative
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract threading architecture from `kitty/child-monitor.c` by tracing `io_loop()`, `parse_input()`, and `process_global_state()`
- Extract VT parser mechanics from `kitty/vt-parser.c` by tracing `run_worker()`, `consume_input()`, and the buffer management API
- Extract shell integration dispatch from `kitty/screen.c` `shell_prompt_marking()` and `process_cwd_notification()`
- Extract paste handling from `kitty/window.py` `paste_text()`
- Extract resize flow from `kitty/child-monitor.c` `process_pending_resizes()` and `pty_resize()`
- Extract timing parameters from `kitty/options/definition.py` and `kitty/state.h`

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration using triple-backtick `mermaid` blocks for architecture, sequence, and flowchart diagrams
- Code reference citations as inline `Source: path/to/file.c:function_name` annotations
- Thinking/rationale blocks explaining *why* the architecture works the way it does
- All claims grounded in specific source files per the user's implementation rules

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Entry point diagram** — Flowchart showing all input entry points (GLFW keyboard, GLFW mouse, PTY child output, remote control socket) converging on the three threads
- **Three-thread architecture diagram** — Component diagram showing Main, I/O, and Talk threads with shared data structures and synchronization primitives
- **I/O thread poll loop** — Flowchart of the `io_loop()` function showing the poll → read → write → wakeup cycle
- **Main thread processing cycle** — Flowchart of `process_global_state()` showing the deterministic ordering
- **VT parser dispatch** — Flowchart showing byte classification into text, CSI, OSC, DCS paths with shell integration callout
- **Backpressure sequence** — Sequence diagram showing the feedback loop between `vt_parser_has_space_for_input()`, I/O thread `POLLIN` gating, and `write_space_created` notification
- **Resize flow** — Sequence diagram from GLFW resize event through debounce to `pty_resize()` and child SIGWINCH
- **End-to-end journey** — A comprehensive flowchart tracing a mixed input burst from arrival to settled interface


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/vt-parser.h`, `kitty/screen.c`, `kitty/screen.h`, `kitty/keys.c`, `kitty/keys.py`, `kitty/key_encoding.c`, `kitty/mouse.c`, `kitty/window.py`, `kitty/boss.py`, `kitty/shell_integration.py`, `kitty/state.h`, `kitty/state.c`, `kitty/loop-utils.c`, `kitty/loop-utils.h`, `kitty/modes.h`, `kitty/options/definition.py`, `shell-integration/bash/kitty.bash` | Comprehensive Q&A document answering all pipeline questions with architecture diagrams, code-grounded rationale, and end-to-end narrative |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Deep-dive technical explainer / Architecture Q&A
Source Code:
    - kitty/child-monitor.c (three-thread architecture, io_loop, poll multiplexing, parse_input, process_global_state, process_pending_resizes, pty_resize, read_bytes, write_to_child)
    - kitty/vt-parser.c (run_worker, consume_input, buffer management, backpressure)
    - kitty/vt-parser.h (ParseData struct, thread-safe API)
    - kitty/screen.c (shell_prompt_marking, process_cwd_notification, screen_pause_rendering, screen_check_pause_rendering)
    - kitty/screen.h (paused_rendering struct, pending_scroll fields)
    - kitty/keys.c (GLFW key event conversion, is_modifier_key)
    - kitty/keys.py (dispatch_possible_special_key, keyboard mode stack)
    - kitty/key_encoding.c (Kitty keyboard protocol encoding)
    - kitty/mouse.c (mouse event resolution against mouse_map)
    - kitty/window.py (paste_text, write_to_child, cmd_output_marking, send_key_sequence)
    - kitty/boss.py (on_window_resize, dispatch_action, on_child_death)
    - kitty/shell_integration.py (modify_shell_environ, setup_bash_env, setup_zsh_env, setup_fish_env)
    - kitty/state.h (global_state struct, OPT macro, timing fields)
    - kitty/loop-utils.c/h (LoopData, signal handling, wakeup mechanisms)
    - kitty/modes.h (PENDING_UPDATE mode 2026)
    - kitty/options/definition.py (repaint_delay, input_delay, sync_to_monitor, resize_debounce_time)
    - shell-integration/bash/kitty.bash (PS0/PS1/PS2 modification, OSC 133 emission)
Sections:
    - Introduction & Orientation
    - Where Input First Enters the System
    - The Three-Thread Architecture (Main, I/O, Talk)
    - The I/O Thread's Poll Loop (poll, read_bytes, write_to_child, wakeup)
    - The Main Thread's Processing Cycle (process_global_state ordering)
    - The VT Parser: Classifying the Byte Stream (buffer, run_worker, dispatch)
    - Shell Integration In the Stream (OSC 133, OSC 7, inline processing)
    - Paste Bursts and Bracketed Paste (paste_text, sanitization)
    - Resize Signals: From GLFW to PTY (debounce, pty_resize, SIGWINCH)
    - Synchronized Updates and Paused Rendering (DCS =1s/=2s, snapshot)
    - Timing Parameters and Their Interplay (input_delay, repaint_delay, etc.)
    - Degraded Conditions and Recovery (backpressure, unstable connection)
    - Summary: The Full Journey (end-to-end diagram)
Diagrams:
    - Entry point flowchart
    - Three-thread architecture diagram
    - I/O thread poll loop flowchart
    - Main thread processing cycle flowchart
    - VT parser dispatch flowchart with shell integration callout
    - Backpressure feedback sequence diagram
    - Resize flow sequence diagram
    - End-to-end journey flowchart
Key Citations:
    - kitty/child-monitor.c: io_loop(), process_global_state(), parse_input(), read_bytes(), write_to_child(), process_pending_resizes(), pty_resize()
    - kitty/vt-parser.c: run_worker(), consume_input(), vt_parser_create_write_buffer(), vt_parser_commit_write(), vt_parser_has_space_for_input()
    - kitty/screen.c: shell_prompt_marking(), screen_pause_rendering(), process_cwd_notification()
    - kitty/window.py: paste_text(), write_to_child(), cmd_output_marking()
    - kitty/options/definition.py: repaint_delay, input_delay, resize_debounce_time, sync_to_monitor
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone Markdown document placed in `blitzy/documentation/`, which is outside the existing Sphinx documentation tree and does not require changes to `docs/conf.py`, `docs/Makefile`, or any navigation configuration.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The new document is self-contained and does not reference or include content from the existing Sphinx documentation tree.
- **No navigation updates required:** The `blitzy/documentation/` directory is independent of the `docs/` Sphinx tree.
- **No index or glossary updates needed:** The document stands alone as a Q&A response.
- **Source code references are read-only:** All citations point to existing source files and functions; no source code modifications are involved.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task involves creating a single Markdown file. No documentation build tools are required to produce the output. The document uses standard GitHub-Flavored Markdown with Mermaid diagram syntax, which is natively rendered by GitHub and many Markdown viewers.

For reference, the existing project documentation ecosystem uses the following tools (not required for this task but relevant context):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | (from docs/requirements.txt) | Existing project documentation site generator |
| pip | furo | (from docs/requirements.txt) | Material-style Sphinx theme |
| pip | sphinx-copybutton | (from docs/requirements.txt) | Code block copy buttons |
| pip | sphinxext-opengraph | (from docs/requirements.txt) | OpenGraph meta tags for social sharing |
| pip | sphinx-inline-tabs | (from docs/requirements.txt) | Inline tab content blocks |
| pip | sphinx-autobuild | (from docs/requirements.txt) | Live-reload documentation development |

The project itself requires:
| Registry | Runtime | Version Constraint | Purpose |
|----------|---------|-------------------|---------|
| python.org | Python | >= 3.8 (from pyproject.toml) | Core application runtime |
| go.dev | Go | 1.22 (from go.mod) | CLI tooling and static binary |
| gcc/clang | C11 | (enforced via -std=c11 in setup.py) | Performance-critical native extensions |

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are required. The new file `blitzy/documentation/kitty_815df1e210e0.md` is a standalone document that:
- Does not modify any existing documentation files
- Does not require links from existing documents
- Does not introduce new cross-references that would need updating elsewhere
- Contains only internal anchors and source code citations


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's questions define five distinct knowledge areas. Coverage targets for each:

| Knowledge Area | User Question | Source Modules | Target Coverage |
|----------------|--------------|----------------|-----------------|
| Input entry points | "where does it first enter the system?" | `kitty/keys.c`, `kitty/mouse.c`, `kitty/child-monitor.c` (io_loop, read_bytes) | 100% — All entry paths documented |
| Concurrent stream arbitration | "what decides which event gets handled first?" | `kitty/child-monitor.c` (process_global_state, poll, input_delay gating) | 100% — Ordering logic and timing thresholds documented |
| Shell integration interleaving | "how does the system keep screen state, command context, and input meaning aligned?" | `kitty/vt-parser.c` (OSC dispatch), `kitty/screen.c` (shell_prompt_marking, process_cwd_notification) | 100% — OSC 133 A/C/D handling documented with line attribute annotation |
| Backpressure and degraded conditions | "does it behave differently when there is heavy backpressure or an unstable remote connection?" | `kitty/vt-parser.c` (BUF_SZ, vt_parser_has_space_for_input), `kitty/screen.c` (screen_pause_rendering) | 100% — Buffer limits, flow control gates, and synchronized update mechanisms documented |
| End-to-end coherence | "how all those moving parts manage to keep their rhythm" | All modules above plus `kitty/options/definition.py` timing parameters | 100% — Full journey narrative with timing parameter interplay |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must have a clearly identified answer section
- All claims must cite specific source files with function names
- Thinking/rationale must be provided for each answer (per implementation rule)
- Architecture diagrams must be present for all major subsystems discussed

**Accuracy validation:**
- All code references must match the actual codebase (verified through `read_file` and `bash` tool during context gathering)
- Function signatures and struct field names must be accurate
- Timing parameter defaults must match `kitty/options/definition.py`
- Thread names and roles must match `kitty/child-monitor.c` code

**Clarity standards:**
- Technical accuracy with accessible language suitable for an onboarding engineer
- Progressive disclosure: start with high-level overview, then drill into each subsystem
- Consistent terminology: "I/O thread" (not "child monitor thread"), "Main thread" (not "render thread"), "Talk thread" (not "socket thread")
- Each section should be self-contained but reference related sections

**Maintainability:**
- Source citations for every technical claim enable future verification
- File path references enable readers to jump directly to the code
- Mermaid diagrams can be updated independently of the narrative

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 8 Mermaid diagrams (entry points, three-thread architecture, I/O thread loop, main thread cycle, VT parser dispatch, backpressure feedback, resize flow, end-to-end journey)
- **Diagram types:** Flowcharts (6), sequence diagrams (2)
- **No code examples to test:** This is a pure explanation document; code snippets are used for citation, not for execution
- **Visual content freshness:** Diagrams are derived from source code analysis and will remain accurate as long as the referenced functions exist


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable, a comprehensive Markdown Q&A document

- **Source modules to analyze and cite (read-only):**
  - `kitty/child-monitor.c` — Three-thread architecture, I/O loop, input parsing, resize processing
  - `kitty/vt-parser.c` — VT parser state machine, buffer management, backpressure, `run_worker()`
  - `kitty/vt-parser.h` — `ParseData` struct definition, thread-safe API declarations
  - `kitty/screen.c` — Shell prompt marking, CWD notifications, paused rendering, mode handling
  - `kitty/screen.h` — `paused_rendering` struct, screen data structures
  - `kitty/keys.c` — Keyboard event processing, modifier detection
  - `kitty/keys.py` — Shortcut dispatch, keyboard mode stack management
  - `kitty/key_encoding.c` — Kitty keyboard protocol encoding
  - `kitty/mouse.c` — Mouse event resolution
  - `kitty/window.py` — Paste handling, write_to_child, command output marking
  - `kitty/boss.py` — Window resize handling, action dispatch, child death handling
  - `kitty/shell_integration.py` — Shell environment modification
  - `kitty/state.h` — Global state structure, OPT macro, timing fields
  - `kitty/state.c` — Global state implementation
  - `kitty/loop-utils.c` / `kitty/loop-utils.h` — Event loop infrastructure, signal handling
  - `kitty/modes.h` — Terminal mode definitions including PENDING_UPDATE (2026)
  - `kitty/control-codes.h` — Escape code constants
  - `kitty/options/definition.py` — Configuration option definitions
  - `shell-integration/bash/kitty.bash` — Bash-side OSC 133 emission
  - `shell-integration/zsh/kitty.zsh` — Zsh integration launcher
  - `shell-integration/fish/` — Fish integration scripts

- **Topics to cover:**
  - Input entry points (keyboard, mouse, PTY output)
  - Three-thread architecture and coordination
  - I/O thread poll loop mechanics
  - Main thread processing cycle and deterministic ordering
  - VT parser classification and dispatch
  - Shell integration interleaving (OSC 133, OSC 7)
  - Paste burst handling with bracketed paste
  - Resize signal propagation and debouncing
  - Synchronized updates and paused rendering
  - Timing parameter interplay (input_delay, repaint_delay, resize_debounce_time, sync_to_monitor)
  - Backpressure and degraded condition behavior
  - End-to-end coherence narrative

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing repository files will be modified (per user instruction and implementation rule)
- **Test file modifications:** No test files will be created or modified
- **GPU rendering pipeline internals:** While mentioned for context (as the endpoint of the pipeline), the GLSL shader architecture, glyph caching, and OpenGL infrastructure are not the focus of the user's questions
- **Font subsystem:** Font discovery, rasterization, and HarfBuzz shaping are not relevant to the input pipeline questions
- **Remote control protocol details:** The Talk thread is mentioned architecturally, but the full 7-layer security chain and RC command execution are out of scope
- **Kittens framework execution:** Kitten lifecycle and error isolation are not relevant to the pipeline questions
- **Configuration file parsing:** The configuration loading pipeline (section 4.4) is out of scope; only the runtime effect of timing parameters is relevant
- **SSH bootstrap sequence:** The `shell-integration/ssh/` bootstrap is not directly relevant to the live-running pipeline questions
- **Build system and packaging:** `setup.py`, `Makefile`, `bypy/`, and release engineering are unrelated
- **Existing Sphinx documentation updates:** The `docs/` tree will not be modified
- **Feature additions or code refactoring:** This is a pure documentation task


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Output format:** Standard Markdown (`.md`) with GitHub-Flavored Markdown extensions and Mermaid diagram blocks
- **Output location:** `blitzy/documentation/kitty_815df1e210e0.md`
- **Output filename convention:** `<source_branch_name>.md` → `kitty_815df1e210e0.md` (derived from branch `kitty_815df1e210e0`)
- **Directory creation:** The `blitzy/documentation/` directory must be created as it does not currently exist in the repository
- **Documentation build command:** None required — the output is a standalone Markdown file
- **Documentation preview:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram rendering:** Mermaid diagrams are embedded inline using triple-backtick `mermaid` blocks; they render natively on GitHub and in VS Code with the Mermaid extension
- **Citation format:** Inline source references as `Source: path/to/file.c:function_name()` or `(see kitty/child-monitor.c, lines N-M)`
- **Style guide:** Technical explainer with onboarding-engineer audience; progressive disclosure from overview to detail; thinking/rationale blocks for each major answer
- **Validation:** All claims verified against source code through repository inspection during context gathering phase; no automated link checking or linting required for standalone Markdown

### 0.9.2 Repository Preservation

Per the user's explicit instruction and the SWE-AtlasQnA-Repo implementation rule:

- **No existing files will be modified** — All existing `.c`, `.py`, `.h`, `.rst`, and other source files remain untouched
- **Only new files created:** `blitzy/documentation/kitty_815df1e210e0.md` (and the `blitzy/documentation/` directory)
- **No temporary scripts:** Although the user permitted temporary observation scripts, this documentation task requires none; all analysis was performed through repository inspection tools
- **Cleanup:** No cleanup required since no temporary artifacts are created


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the SWE-AtlasQnA-Repo implementation rule:

- **Create a new markdown document named `kitty_815df1e210e0.md`** that comprehensively answers the questions posed in the prompt. The filename is derived from the source branch name `kitty_815df1e210e0`.
- **Provide thinking / rationale behind the answers.** Every major answer section must include an explanation of *why* the architecture works the way it does, not just *what* it does. The user wants to understand the design reasoning, not just the mechanics.
- **Do not make assumptions, base your answers on the code as the truth.** All claims must cite specific source files. Assertions about timing, ordering, and behavior must be traceable to actual code paths in `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/screen.c`, and related files.
- **Do not modify any existing files in the source repository.** The repository must remain exactly as it was before this task. The only filesystem change is the creation of `blitzy/documentation/kitty_815df1e210e0.md` and its parent directory.
- **Place the generated document in the `blitzy/documentation` directory** in the destination repo. This directory must be created if it does not exist.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.** Since this is a pure documentation task based on code reading, no temporary scripts are needed.


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were systematically examined during context gathering to derive all conclusions in this Agent Action Plan:

**Core pipeline source files (read and analyzed):**

| File Path | Key Elements Extracted |
|-----------|----------------------|
| `kitty/child-monitor.c` | `ChildMonitor` struct, `io_loop()`, `read_bytes()`, `write_to_child()`, `parse_input()`, `do_parse()`, `process_global_state()`, `process_pending_resizes()`, `pty_resize()`, `wakeup_io_loop()`, `main_loop()`, `schedule_write_to_child()`, signal handling, three-thread architecture |
| `kitty/vt-parser.c` | `run_worker()`, `consume_input()`, `BUF_SZ` (1 MB), `MAX_ESCAPE_CODE_LENGTH`, VTE state machine, DCS pending mode handling (`=1s`/`=2s`), OSC dispatch, `parse_worker()`, `parse_worker_dump()` |
| `kitty/vt-parser.h` | `Parser` struct, `ParseData` struct (`input_read`, `write_space_created`, `has_pending_input`, `time_since_new_input`), `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `vt_parser_has_space_for_input()` |
| `kitty/screen.c` | `shell_prompt_marking()` (OSC 133 A/C/D dispatch), `process_cwd_notification()` (OSC 7), `screen_pause_rendering()`, `screen_check_pause_rendering()`, `BRACKETED_PASTE` mode, `screen_cursor_at_a_shell_prompt()` |
| `kitty/screen.h` | `paused_rendering` struct fields (expires_at, cursor_visible, inverted, linebuf, grman, selections, url_ranges, color_profile), `pending_scroll_pixels_x/y` |
| `kitty/keys.c` | `PyKeyEvent` struct, `convert_glfw_key_event_to_python()`, `is_modifier_key()`, `is_no_action_key()`, `SingleKey` type |
| `kitty/keys.py` | `dispatch_possible_special_key()`, keyboard mode stack, sequence matching, `matching_key_actions()` |
| `kitty/key_encoding.c` | Kitty keyboard protocol CSI u encoding |
| `kitty/window.py` | `Window` class, `paste_text()`, `write_to_child()`, `cmd_output_marking()`, `send_key_sequence()`, `Watchers` class (on_resize, on_focus_change, on_cmd_startstop), `load_paste_filter()` |
| `kitty/boss.py` | `on_window_resize()`, `dispatch_action()`, `on_child_death()`, `peer_message_received()`, `mark_window_for_close()` |
| `kitty/shell_integration.py` | `modify_shell_environ()`, `setup_bash_env()`, `setup_zsh_env()`, `setup_fish_env()`, `get_effective_ksi_env_var()`, `ENV_MODIFIERS` dict, `ENV_SERIALIZERS` dict |
| `kitty/state.h` | `global_state` struct fields (`opts.repaint_delay`, `opts.input_delay`, `opts.resize_debounce_time`, `opts.sync_to_monitor`, `has_pending_resizes`, `has_pending_closes`), `OPT()` macro |
| `kitty/loop-utils.c` | `init_loop_data()`, signal handling via `signalfd()` or self-pipe fallback, `wakeup_loop()` |
| `kitty/loop-utils.h` | `LoopData` struct, `drain_fd()`, `self_pipe()`, `read_signals()` |
| `kitty/modes.h` | `PENDING_UPDATE` (mode 2026), `BRACKETED_PASTE` mode definition |
| `kitty/control-codes.h` | Escape code constants (ESC_DCS, ESC_OSC, etc.) |
| `kitty/options/definition.py` | `repaint_delay` (default 10ms), `input_delay` (default 3ms), `sync_to_monitor` (default yes), `resize_debounce_time` (default 0.1s on_end, 0.5s on_pause) |

**Shell integration scripts (read and analyzed):**

| File/Folder Path | Key Elements Extracted |
|-----------------|----------------------|
| `shell-integration/bash/kitty.bash` | PS0/PS1/PS2 modification, OSC 133 marker emission, clone-session packaging |
| `shell-integration/zsh/kitty.zsh` | Zsh compatibility launcher, kitty-integration autoload |
| `shell-integration/fish/` | Fish vendor completions and startup integration |
| `shell-integration/ssh/` | Remote bootstrap scripts (architecture context only) |

**Existing documentation (read for gap analysis):**

| File Path | Key Elements Extracted |
|-----------|----------------------|
| `docs/performance.rst` | User-facing performance tuning guidance, `input_delay`/`repaint_delay`/`sync_to_monitor` descriptions |
| `docs/shell-integration.rst` | User-facing shell integration features and configuration |
| `docs/conf.py` | Sphinx configuration, custom roles, extensions |
| `docs/requirements.txt` | Documentation build dependencies |

**Build and project metadata (read for context):**

| File Path | Key Elements Extracted |
|-----------|----------------------|
| `pyproject.toml` | Python version constraint (>= 3.8), tool configurations |
| `setup.py` | Build system, C11 enforcement, multi-language compilation |
| `go.mod` | Go module identity, Go 1.22 requirement |

**Tech spec sections retrieved:**

| Section | Key Context Provided |
|---------|---------------------|
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Application lifecycle, core event loop architecture, timing considerations |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | Input processing flow, VT parser dispatch, GPU rendering pipeline |
| 4.4 CONFIGURATION MANAGEMENT FLOW | Configuration loading pipeline, live reload |
| 4.5 WINDOW AND TAB LIFECYCLE | Session creation, child process launch, window state transitions |
| 4.7 SHELL INTEGRATION FLOW | Shell environment setup, SSH bootstrap, active integration features |
| 4.10 ERROR HANDLING AND RECOVERY FLOWS | Startup errors, shell integration error isolation, extension error containment |
| 5.1 HIGH-LEVEL ARCHITECTURE | System overview, six-layer architecture, data flow description |
| 5.2 COMPONENT DETAILS | Child Monitor, VT Parser, Boss Controller, Shell Integration component details |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens or external design assets are associated with this task.

### 0.11.3 External URLs

No external URLs were referenced by the user. All analysis is based exclusively on the repository source code and the existing tech spec document.


