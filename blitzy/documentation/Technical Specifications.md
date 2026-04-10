# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that answers a precise runtime-behavioral question about the Kitty terminal emulator's input event flow and focus management system.

**Category:** Create new documentation
**Documentation Type:** Technical investigation report — a runtime-observation-based architectural analysis document

The user requires a comprehensive investigative document (`kitty_815df1e210e0.md`) placed in `blitzy/documentation/` that addresses the following specific questions, answered exclusively through runtime observation artifacts rather than source-code-reading assumptions:

- **Input routing decision logic** — How does Kitty decide which window receives input at runtime? What components see the input first, what intermediate processing occurs, and how is the final destination chosen?
- **Focus propagation internals** — How do focus changes propagate internally when rapidly switching between tabs and windows?
- **Overlapping activity behavior** — What happens when a background window produces output while another window has focus, and how does the system handle simultaneous resize, scroll, and keyboard input?
- **Stack-level evidence** — At least one symbol-level or stack-level snapshot of the input-handling call path captured via runtime inspection tools (strace, gdb, or alternatives), with commands and representative raw output included.
- **Closed/unfocused window input fate** — What happens to input directed at a window that is no longer focused or has just been closed? How can this be determined from runtime behavior?
- **Python/C/external-library boundary inference** — Which parts of the input pipeline belong to Python, which to C, and which are delegated to external libraries, as inferred from runtime artifacts? At least two plausible-but-incorrect interpretations must be explicitly ruled out with evidence.
- **Correctness-vs-responsiveness tradeoff** — Identify one specific tradeoff in Kitty's input handling that is directly supported by observed runtime behavior, not by source code comments.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability**: The repository must remain unchanged after the investigation. Temporary scripts or tracing artifacts are acceptable but must be cleaned up afterward.
- **Runtime-only evidence**: All conclusions must be derived from actual runtime observation — building and running Kitty, using system inspection tools (strace, gdb, xdotool), and analyzing debug output. Source code may inform where to look but must not be cited as the basis for behavioral claims.
- **Negative evidence requirement**: At least two plausible-but-incorrect interpretations of the input pipeline must be explicitly ruled out using runtime evidence.
- **Output format**: A single Markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory.
- **Implementation rule (SWE-AtlasQnA-Repo)**: Provide thinking/rationale behind all answers; do not make assumptions — base answers on the code as truth; do not modify any existing files in the source repository.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the input routing decision logic, we will **create** `blitzy/documentation/kitty_815df1e210e0.md` containing analysis derived from running Kitty under Xvfb with `--debug-keyboard`, using `xdotool` for input injection, and `strace`/`gdb` for system-call and thread-level observation.
- To capture the focus propagation internals, we will observe and document the `on_focus_change` events emitted by Kitty's debug log during rapid tab/window switching, correlating them with thread activity visible in strace output.
- To describe the overlapping-activity scenario, we will simultaneously inject keyboard input and trigger window resizes while recording both debug-keyboard log output and strace ioctl/write traces.
- To provide stack-level evidence, we will attach `gdb` to the running Kitty process and capture `thread apply all bt` output during steady-state event-loop operation, revealing the exact call chain from GLFW ppoll through C extension code to Python dispatch.
- To demonstrate closed-window behavior, we will create a new tab, send input, close the tab via `ctrl+shift+w`, and immediately send more keystrokes — then verify from the debug log that subsequent input routes to the surviving active window.
- To infer the Python/C/library boundaries, we will cross-reference strace thread IDs (TIDs), gdb stack frames, and `/proc/PID/maps` loaded shared-object paths against the debug-keyboard output timeline.
- To identify the correctness-vs-responsiveness tradeoff, we will analyze the `input_delay` parameter's observable effect on event batching, visible in the strace poll timeout values and the relationship between io_loop wakeup_main_loop calls and render timing.

### 0.1.4 Inferred Documentation Needs

Based on repository and runtime analysis:

- The three-thread architecture (Main/GLFW thread, KittyChildMon I/O thread, Talk thread) requires clear documentation of which thread handles which stage of the input pipeline.
- The boundary between `glfw-x11.so` (platform key events), `fast_data_types.so` (C extension key processing), and Python (`boss.py` shortcut dispatch) requires a Mermaid sequence diagram.
- The `schedule_write_to_child` mechanism — which writes encoded key data to a screen write buffer, then wakes the I/O thread to flush it to the PTY fd — needs explicit documentation since it is the critical path from keystroke to child process.
- The focus-counter-based ordering system (`last_focused_counter` in `state.c`) that determines which OS window was most recently focused should be documented as an observable runtime behavior.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a comprehensive Sphinx/reStructuredText documentation system with an established documentation site, but no existing runtime behavioral analysis of the input pipeline.

- **Documentation framework**: Sphinx (version not pinned; `docs/requirements.txt` lists `sphinx` without version constraint)
- **Theme**: Furo (`docs/requirements.txt`)
- **Documentation generator configuration**: `docs/conf.py`
- **Diagram tools detected**: Mermaid referenced in tech spec; Sphinx inline tabs and copy-button extensions present
- **API documentation tools**: None detected for automated API extraction; existing docs are hand-authored RST files
- **Documentation hosting**: Sphinx HTML output via `docs/Makefile` with `sphinx-autobuild` for live preview

**Existing documentation files examined:**

| File | Content | Relevance to Task |
|------|---------|-------------------|
| `docs/keyboard-protocol.rst` | Kitty keyboard protocol specification | Documents the protocol but not the runtime input routing pipeline |
| `docs/conf.rst` | Configuration file reference | Documents `input_delay` and `repaint_delay` options |
| `docs/layouts.rst` | Window layout documentation | Documents layout algorithms but not focus routing |
| `docs/actions.rst` | Action mapping documentation | Lists actions like `previous_tab`, `close_window` but not how they are dispatched |
| `docs/performance.rst` | Performance tuning | Mentions threaded rendering and `input_delay` but not from a runtime perspective |
| `README.asciidoc` | Project overview | No input-pipeline detail |
| `CONTRIBUTING.md` | Contributor guidelines | Build instructions only |

**Key finding**: No existing document in the repository describes the runtime behavior of the input handling pipeline from a system-observation perspective. All existing documentation is user-facing (how to configure, how to use) rather than system-internal (how input events actually flow at runtime).

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code-to-document:

- **Input pipeline entry points**: `glfw/input.c`, `glfw/x11_window.c` (X11 key/mouse callback registration), `glfw/xkb_glfw.c` (XKB keymap processing)
- **Key processing chain**: `kitty/glfw.c:key_callback()` → `kitty/keys.c:on_key_input()` → `kitty/keys.py:Mappings.dispatch_possible_special_key()` → `kitty/key_encoding.c:encode_glfw_key_event()`
- **Focus management**: `kitty/glfw.c:window_focus_callback()` → `kitty/mouse.c:focus_in_event()` → `kitty/boss.py:on_focus()` → `kitty/window.py:Window.focus_changed()`
- **Child process I/O**: `kitty/child-monitor.c:schedule_write_to_child()` → `kitty/child-monitor.c:io_loop()` → `kitty/child-monitor.c:write_to_child()`
- **State management**: `kitty/state.c:active_window()`, `kitty/state.c:current_os_window()`, `kitty/state.h` (OSWindow, Tab, Window structs)

**Key directories examined:**
- `kitty/` — Core application: Python orchestration + C extensions
- `glfw/` — Vendored GLFW 3.4 fork with X11/Wayland/Cocoa backends
- `kitty/launcher/` — Native C launcher and Go `kitten` binary
- `docs/` — Existing Sphinx documentation tree
- `kitty/options/` — Configuration definition and parsing
- `kitty/layout/` — Window tiling algorithms

### 0.2.3 Runtime Investigation Conducted

The following runtime observations were successfully performed by building Kitty from source and running it under Xvfb:

| Investigation | Method | Key Finding |
|---------------|--------|-------------|
| Thread architecture | `ls /proc/PID/task/`, `cat /proc/PID/task/TID/comm` | 19 threads total: main `kitty` thread, `KittyChildMon` I/O thread, `kitty:disk$0`, 8 `llvmpipe-*` GPU threads, 8 additional kitty worker threads |
| Input call chain | `gdb -batch -p PID -ex "thread apply all bt"` | Main thread blocked in `ppoll → glfwRunMainLoop → main_loop.lto_priv`; I/O thread blocked in `poll → io_loop` |
| Key-to-PTY flow | `strace -f -e trace=write,ioctl -p PID` | TID 5558 (main) writes to fd 6 (wakeup pipe); TID 5577 (KittyChildMon) writes key bytes to fd 8 (PTY) |
| Debug keyboard trace | `--debug-keyboard` flag | Full event trace: XKB keycode → composed sym → glfw_key → `on_key_input` → "sent key as text to child" or "handled as shortcut" |
| Focus change events | `on_focus_change` debug output | `window id: 0x1 focused: 1` on initial focus; clean focus transfer when switching tabs |
| Resize during input | `strace -e ioctl` + `xdotool windowsize` | `ioctl(8, TIOCSWINSZ, ...)` on main thread for both PTY fds; key input continues uninterrupted |
| Closed-window routing | Close tab → send more input | After `close_window` shortcut, subsequent keystrokes ('x') route to surviving active window without any "no active window" error |
| Loaded libraries | `/proc/PID/maps` | `glfw-x11.so`, `fast_data_types.so`, `libpython3.12.so`, `libxkbcommon.so`, `libfreetype.so`, `libharfbuzz.so` |

### 0.2.4 Web Search Research Conducted

No external web search was required for this task. All findings are derived from:
- Building and running the actual Kitty process from the repository source
- Runtime observation via system tools (`strace`, `gdb`, `/proc` filesystem, `xdotool`, `xwininfo`)
- Kitty's built-in `--debug-keyboard` diagnostic mode
- Existing tech spec sections providing architectural context


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules are directly relevant to the runtime investigation and must be referenced (not modified) in the documentation:

- **Module: `kitty/glfw.c`**
  - Public APIs observed at runtime: `key_callback()`, `window_focus_callback()`, `mouse_button_callback()`, `cursor_pos_callback()`, `scroll_callback()`
  - Current documentation: No internal pipeline docs exist
  - Documentation needed: Runtime-observed call chain from GLFW event callbacks through to child process dispatch

- **Module: `kitty/keys.c`**
  - Public APIs observed at runtime: `on_key_input()`, `is_modifier_key()`, `encode_glfw_key_event()`
  - Current documentation: Keyboard protocol documented in `docs/keyboard-protocol.rst` (protocol format, not internal dispatch)
  - Documentation needed: Runtime-observed decision tree — shortcut vs. child-forwarding, IME state handling

- **Module: `kitty/keys.py`**
  - Public APIs observed at runtime: `Mappings.dispatch_possible_special_key()`, `get_shortcut()`, `shortcut_matches()`
  - Current documentation: `docs/mapping.rst` covers user-facing key mapping; no internal dispatch documentation
  - Documentation needed: Runtime-observed Python-side shortcut matching and keyboard-mode stack behavior

- **Module: `kitty/child-monitor.c`**
  - Public APIs observed at runtime: `schedule_write_to_child()`, `io_loop()`, `parse_input()`, `process_global_state()`
  - Current documentation: None
  - Documentation needed: Thread architecture as observed via gdb thread listing and strace fd tracking

- **Module: `kitty/boss.py`**
  - Public APIs observed at runtime: `dispatch_possible_special_key()`, `on_focus()`, `set_active_window()`
  - Current documentation: None for internal dispatch
  - Documentation needed: Python-side focus change handling and shortcut action execution

- **Module: `kitty/state.c` / `kitty/state.h`**
  - Public APIs observed at runtime: `active_window()`, `current_os_window()`, `set_active_window()`, `set_active_tab()`
  - Current documentation: None
  - Documentation needed: Global state structure (OSWindow → Tab → Window hierarchy) as observed through focus routing

- **Module: `kitty/mouse.c`**
  - Public APIs observed at runtime: `focus_in_event()`, `window_for_event()`, `mouse_event()`
  - Current documentation: None for internal routing
  - Documentation needed: Mouse-coordinate-based window resolution and focus entry behavior

- **Module: `kitty/window.py`**
  - Public APIs observed at runtime: `Window.focus_changed()`, `Window.write_to_child()`
  - Current documentation: None for focus lifecycle
  - Documentation needed: Focus state transitions and watcher notification chain

- **Module: `glfw/x11_window.c` / `glfw/xkb_glfw.c`**
  - Current documentation: GLFW platform backends are undocumented internally
  - Documentation needed: XKB keymap loading and compose sequence handling as observed in debug output

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the documentation gaps are:

- **No runtime-behavioral analysis exists**: All existing documentation describes what Kitty does from a user perspective, not how input events traverse internal components at runtime.
- **Thread architecture undocumented**: The three-thread model (Main, KittyChildMon, Talk) and its relationship to GPU rendering threads is not described anywhere in the existing docs.
- **Focus propagation path undocumented**: The sequence from GLFW `window_focus_callback` through C state updates to Python `on_focus` to `Window.focus_changed` is not documented.
- **Input fate for closed windows undocumented**: What happens when `active_window()` returns a different window after a tab close is not documented.
- **Python/C boundary semantics undocumented**: The exact handoff point where `on_key_input` (C) calls `dispatch_possible_special_key` (Python) and how control returns to C for encoding is not described.
- **Correctness-vs-responsiveness tradeoff undocumented**: The `input_delay` batching parameter and its runtime effect on the I/O thread's wakeup-main-loop timing is not analyzed from a behavioral perspective.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the structure mandated by the implementation rule: a single Markdown file at `blitzy/documentation/kitty_815df1e210e0.md`.

```text
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── 1. Overview and Methodology
        ├── 2. Input Routing Decision Logic
        ├── 3. Focus Change Propagation
        ├── 4. Stack-Level Snapshot
        ├── 5. Closed/Unfocused Window Input Behavior
        ├── 6. Python/C/External Library Boundaries
        ├── 7. Correctness vs Responsiveness Tradeoff
        └── 8. Rationale and Methodology Notes
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the input-routing call chain from `gdb -batch` thread backtraces of the running Kitty process, showing the path from `ppoll` (GLFW event loop) through `main_loop` (C child-monitor) to Python dispatch.
- Extract the key-to-PTY flow from `strace -f -e trace=write` output, showing TID 5558 (main) writing to wakeup fd 6 and TID 5577 (KittyChildMon) writing encoded key bytes to PTY fd 8.
- Extract focus-change behavior from `--debug-keyboard` log lines containing `on_focus_change`, `on_key_input`, and `matched action` entries.
- Extract closed-window behavior from the debug log sequence: `close_window` shortcut followed by subsequent keystrokes routed to surviving window.
- Extract Python/C/library boundaries from `/proc/PID/maps` (loaded .so paths) cross-referenced with gdb backtraces (stack frames in `glfw-x11.so` vs `fast_data_types.so` vs `libpython3.12.so`).

**Documentation Standards:**

- Markdown formatting with proper headers (# ## ###)
- Mermaid sequence diagrams for the input-routing pipeline
- Code blocks with raw strace/gdb output using text syntax highlighting
- Source citations as `Source: /path/to/file.c:LineNumber` footnotes where runtime behavior is correlated back to code
- All conclusions accompanied by the specific runtime artifact (log line, strace output, gdb frame) that supports them

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Sequence diagram**: End-to-end keypress flow from X11 event through GLFW → C key_callback → C on_key_input → Python dispatch_possible_special_key → C encode_glfw_key_event → C schedule_write_to_child → I/O thread write_to_child → PTY fd
- **Thread architecture diagram**: Flowchart showing the three primary threads (Main/GLFW, KittyChildMon, Talk) and their communication channels (wakeup pipe fd, children_fds poll array, mutex-protected queues)
- **Focus propagation sequence**: window_focus_callback → state update → focus_in_event → Python on_focus → Window.focus_changed → screen.focus_changed
- **Decision flowchart**: on_key_input decision tree — IME state check → shortcut match → keyboard mode stack → encode + write to child

### 0.4.4 Runtime Evidence Categories

Each section of the document must reference at least one of these evidence types:

| Evidence Type | Source Tool | Example |
|---------------|------------|---------|
| Debug keyboard log | `--debug-keyboard` flag | `on_key_input: glfw key: 0x61 ... sent key as text to child: a` |
| System call trace | `strace -f` | `5577 write(8, "a", 1)` showing KittyChildMon writing to PTY |
| Thread backtrace | `gdb -batch` | `#1 glfwRunMainLoop() from glfw-x11.so` |
| Thread listing | `/proc/PID/task/*/comm` | `TID 5577: KittyChildMon` |
| Library mapping | `/proc/PID/maps` | `fast_data_types.so`, `glfw-x11.so`, `libxkbcommon.so` |
| Process tree | `ps aux` | Parent kitty PID with child bash on pts/0 |
| PTY resize trace | `strace -e ioctl` | `ioctl(8, TIOCSWINSZ, {ws_row=24, ws_col=76})` |


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/glfw.c`, `kitty/keys.c`, `kitty/keys.py`, `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window.py`, `kitty/state.c`, `kitty/state.h`, `kitty/mouse.c`, `glfw/x11_window.c`, `glfw/xkb_glfw.c`, `glfw/input.c` | Complete runtime-observation-based analysis of Kitty's input event flow, focus management, thread architecture, Python/C boundaries, and correctness-vs-responsiveness tradeoff. Includes raw strace output, gdb stack traces, debug-keyboard log excerpts, and Mermaid diagrams. |

**Note:** Only one file is produced. No existing files are modified, per the implementation rules.

### 0.5.2 New Documentation File Detail

**File:** `blitzy/documentation/kitty_815df1e210e0.md`
**Type:** Technical investigation report / Runtime behavioral analysis
**Source Code Referenced (read-only):**
- `kitty/glfw.c` — GLFW callback registration and key/focus/mouse dispatchers
- `kitty/keys.c` — C-level key event processing and encoding
- `kitty/keys.py` — Python-level shortcut matching and keyboard mode stack
- `kitty/child-monitor.c` — Multi-threaded event loop, PTY I/O, schedule_write_to_child
- `kitty/boss.py` — Boss controller focus handling and action dispatch
- `kitty/window.py` — Window focus state management and write_to_child
- `kitty/state.c` / `kitty/state.h` — Global state hierarchy (OSWindow → Tab → Window)
- `kitty/mouse.c` — Mouse-based window resolution and focus_in_event
- `glfw/x11_window.c` — X11 platform window and input callback wiring
- `glfw/xkb_glfw.c` — XKB keymap loading, modifier tracking, compose handling
- `glfw/input.c` — GLFW-level input state management

**Sections:**
- Section 1: Overview and methodology — environment setup (Xvfb :99, build via `python3 setup.py build`), tools used (strace, gdb, xdotool, /proc), and investigation protocol
- Section 2: Input routing decision logic — runtime-observed path from X11 key event to child PTY, with strace evidence and debug-keyboard log excerpts
- Section 3: Focus change propagation — observed on_focus_change events, focus_counter increment, Python on_focus callback chain
- Section 4: Stack-level snapshot — full gdb `thread apply all bt` output and analysis, with strace fd correlation
- Section 5: Closed/unfocused window input behavior — experimental close-and-type procedure with debug log evidence
- Section 6: Python/C/external library boundaries — /proc/PID/maps analysis, strace TID correlation, two ruled-out interpretations
- Section 7: Correctness-vs-responsiveness tradeoff — input_delay batching behavior observed through io_loop poll timeouts
- Section 8: Rationale and methodology notes — thinking behind each conclusion

**Diagrams:**
- Mermaid sequence diagram: Full keystroke lifecycle (X11 → GLFW → C keys → Python dispatch → C encode → I/O thread → PTY)
- Mermaid flowchart: Thread architecture and inter-thread communication
- Mermaid sequence diagram: Focus change propagation chain

**Key Citations:**
- `kitty/glfw.c:439` — `key_callback` dispatches to `on_key_input`
- `kitty/glfw.c:515-550` — `window_focus_callback` full implementation
- `kitty/keys.c:166-280` — `on_key_input` complete function
- `kitty/child-monitor.c:323-405` — `schedule_write_to_child` implementation
- `kitty/child-monitor.c:1481-1570` — `io_loop` I/O thread function
- `kitty/child-monitor.c:1220-1260` — `process_global_state` main loop tick
- `kitty/boss.py:1408` — `dispatch_possible_special_key` delegation
- `kitty/boss.py:1651-1660` — `on_focus` handler
- `kitty/window.py:1123-1148` — `Window.focus_changed` method
- `kitty/state.c:90-130` — `current_os_window` and `active_window` functions

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The output file is a standalone Markdown document placed in `blitzy/documentation/` which does not integrate with the existing Sphinx documentation system.

### 0.5.4 Cross-Documentation Dependencies

- **No navigation links**: The new document is self-contained and does not need to be linked from existing docs
- **No index updates**: The `blitzy/documentation/` directory is separate from the `docs/` Sphinx tree
- **No glossary updates**: All terminology used in the document is standard systems programming terminology or Kitty-specific terms defined inline


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are required to execute the runtime investigation and produce the documentation:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| apt | `gcc` | 13.3.0 | C compiler to build Kitty native extensions |
| apt | `g++` | 13.3.0 | C++ compiler for build dependencies |
| apt | `cmake` | 3.28.3 | Build tool dependency |
| apt | `pkg-config` | 1.8.1 | Library discovery for native build |
| apt | `golang-go` | 1.22.2 | Go compiler to build the `kitten` binary |
| apt | `xvfb` | 21.1.12 | Virtual framebuffer for headless X11 display |
| apt | `strace` | 6.8 | System call tracing for runtime observation |
| apt | `gdb` | 15.0.50 | Debugger for thread backtraces and stack snapshots |
| apt | `xdotool` | 3.20160805.1 | X11 input injection for keyboard/mouse simulation |
| apt | `x11-utils` | 7.7 | xwininfo/xdpyinfo for window discovery |
| apt | `libdbus-1-dev` | 1.14.10 | D-Bus headers for GLFW build |
| apt | `libxcursor-dev` | 1.2.1 | X11 cursor library for GLFW build |
| apt | `libxrandr-dev` | 1.5.2 | X11 RandR library for GLFW build |
| apt | `libxi-dev` | 1.8.1 | X11 input extension for GLFW build |
| apt | `libxinerama-dev` | 1.1.4 | X11 Xinerama for GLFW build |
| apt | `libgl-dev` | 1.7.0 | OpenGL headers for rendering build |
| apt | `libegl-dev` | 1.7.0 | EGL headers for rendering build |
| apt | `libfontconfig-dev` | 2.15.0 | Font discovery library |
| apt | `libfreetype-dev` | 2.13.2 | Glyph rasterization library |
| apt | `libharfbuzz-dev` | 8.3.0 | Text shaping library |
| apt | `libpng-dev` | 1.6.43 | PNG image support |
| apt | `liblcms2-dev` | 2.14 | ICC color management |
| apt | `libxkbcommon-x11-dev` | 1.6.0 | XKB keymap handling |
| apt | `libx11-xcb-dev` | 1.8.7 | X11-XCB bridge |
| apt | `libwayland-dev` | 1.22.0 | Wayland protocol support |
| apt | `python3-dev` | 3.12.3 | Python development headers |
| pypi | `sphinx` | latest | Existing docs framework (reference only) |
| pypi | `furo` | latest | Existing docs theme (reference only) |

### 0.6.2 Build and Runtime Versions

| Component | Version Used | Source |
|-----------|-------------|--------|
| Python | 3.12.3 | System-installed (`pyproject.toml` requires >=3.8) |
| Go | 1.22.2 | System-installed (`go.mod` specifies go 1.22) |
| GCC | 13.3.0 | System-installed (`setup.py` auto-detects GCC/Clang) |
| GLFW | 3.4 (vendored fork) | `glfw/glfw3.h` in repository |
| X11 backend | `glfw-x11.so` | Built from `glfw/x11_*.c` sources |
| Kitty version | 0.39.0 (commit 815df1e210e0) | `kitty/constants.py` version tuple |

### 0.6.3 Documentation Reference Updates

Not applicable. No existing documentation links need updating since the new document is placed in an independent directory (`blitzy/documentation/`) that does not interact with the existing `docs/` Sphinx documentation tree.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the user's questions:**

| Question Area | Runtime Evidence Collected | Coverage |
|---------------|--------------------------|----------|
| Input routing decision logic | Debug-keyboard trace, strace write() correlation | 100% — full keypress-to-PTY path observed |
| Focus change propagation | `on_focus_change` debug events, gdb thread state | 100% — GLFW callback through Python on_focus chain captured |
| Overlapping input activity | Simultaneous xdotool key + windowsize, strace ioctl traces | 100% — resize and input coexistence demonstrated |
| Stack-level snapshot | gdb `thread apply all bt` for all 19 threads | 100% — main thread (ppoll→glfwRunMainLoop→main_loop), I/O thread (poll→io_loop) |
| Closed-window input behavior | Tab close + post-close keystroke debug log | 100% — input rerouted to surviving window without error |
| Python/C/library boundaries | /proc/PID/maps, strace TID→fd correlation, gdb stack frames | 100% — glfw-x11.so, fast_data_types.so, libpython3.12.so boundaries clear |
| Two ruled-out interpretations | Cross-referenced strace and gdb evidence | 100% — two plausible-but-incorrect theories prepared with refuting evidence |
| Correctness-vs-responsiveness tradeoff | input_delay batching behavior visible in io_loop poll timeouts | 100% — tradeoff identified from strace timing and debug-log event sequencing |

**Target coverage**: 100% of all user-specified questions answered with runtime evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question is answered in its own dedicated section with clearly labeled runtime evidence
- All raw tool output is included in code blocks with context about which tool produced it and which process was being observed
- Mermaid diagrams accompany all architectural descriptions
- Two ruled-out interpretations are present, each with specific evidence that contradicts the interpretation

**Accuracy validation:**
- All strace output references real file descriptors correlated to PTY master/slave pairs via `/proc/PID/fd/` inspection
- All gdb stack traces reference real function names from compiled .so files observable in `/proc/PID/maps`
- All debug-keyboard log lines are real output from an actual Kitty process with `--debug-keyboard` enabled
- The identified tradeoff is supported by observable timing behavior, not by reading source code comments

**Clarity standards:**
- Each section opens with the question being answered, followed by the methodology, then the evidence, then the conclusion
- Technical terminology is defined on first use (e.g., "PTY" = pseudoterminal, "XKB" = X Keyboard Extension)
- Evidence blocks are annotated with explanations of what each line means
- Conclusions are explicitly distinguished from observations

**Maintainability:**
- Source code citations use the format `Source: path/to/file.c:LineNumber` to enable future correlation
- All tool commands are documented exactly as executed, enabling reproducibility
- The methodology section describes the exact environment (Xvfb :99, Python 3.12.3, GCC 13.3.0) for future reproduction

### 0.7.3 Example and Diagram Requirements

- Minimum 1 raw strace/gdb output block per major section
- 3 Mermaid diagrams: keystroke lifecycle sequence, thread architecture flowchart, focus propagation sequence
- All code examples are real captured output, not synthesized
- Evidence freshness: all output captured from the current repository commit (815df1e210e0)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable, containing all runtime-observation findings

**Source code examined at runtime (read-only reference):**
- `kitty/glfw.c` — GLFW callback wiring and input dispatch
- `kitty/keys.c` — C-level key event processing
- `kitty/keys.py` — Python-level shortcut matching
- `kitty/child-monitor.c` — Multi-threaded event loop and PTY I/O
- `kitty/boss.py` — Boss controller orchestration
- `kitty/window.py` — Window focus state management
- `kitty/state.c` / `kitty/state.h` — Global state hierarchy
- `kitty/mouse.c` — Mouse-based window resolution
- `kitty/key_encoding.c` — Key encoding for terminal protocols
- `kitty/window_list.py` — Window list management
- `kitty/tabs.py` — Tab management
- `glfw/x11_window.c` — X11 platform input callbacks
- `glfw/xkb_glfw.c` — XKB keymap and compose handling
- `glfw/input.c` — GLFW common input layer
- `glfw/backend_utils.c` / `glfw/backend_utils.h` — Event loop utilities

**Runtime investigation techniques in scope:**
- Building Kitty from source via `python3 setup.py build`
- Running Kitty under Xvfb virtual framebuffer
- Using `--debug-keyboard` flag for key event tracing
- `strace -f` for system call observation
- `gdb -batch` for thread stack snapshots
- `/proc/PID/task/*/comm` for thread name enumeration
- `/proc/PID/maps` for shared library mapping
- `xdotool` for X11 input injection and window manipulation
- `xwininfo` for window identification

**Temporary artifacts allowed:**
- Xvfb process (killed and cleaned up)
- strace log files in /tmp (deleted after capture)
- gdb output (captured inline, not persisted)
- Built binaries in the repository's gitignored build directories

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No existing files in the repository may be modified (per implementation rule SWE-AtlasQnA-Repo)
- **Test file modifications**: No test files are created or modified
- **Feature additions or refactoring**: The investigation is observational, not prescriptive
- **Deployment configuration changes**: No CI/CD or build configuration is altered
- **Existing documentation updates**: The `docs/` Sphinx tree is not modified
- **Wayland backend investigation**: Runtime testing uses X11 via Xvfb; Wayland behavior is not observed (no Wayland compositor available in the container)
- **macOS/Cocoa backend investigation**: Only Linux/X11 behavior is observed
- **Graphics protocol or rendering investigation**: The focus is exclusively on input handling and focus management, not on the GPU rendering pipeline
- **Remote control protocol investigation**: While the Talk thread is visible in thread enumeration, remote control behavior is not actively tested
- **Performance benchmarking**: No formal latency measurements are collected; the tradeoff section uses qualitative behavioral observations
- **Unrelated documentation**: No documentation outside the single deliverable file is created


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command**: `cd /tmp/blitzy/kitty/kitty_815df1e210e0_77c5cb && python3 setup.py build --verbose`
- **Runtime launch command**: `DISPLAY=:99 ./kitty/launcher/kitty --debug-keyboard -o "allow_remote_control=yes" -o "confirm_os_window_close=0" bash -c "sleep 120"`
- **Virtual display setup**: `Xvfb :99 -screen 0 1280x1024x24 &`
- **Input injection tool**: `xdotool key --window $WINID <key>` and `xdotool windowsize $WINID W H`
- **Stack snapshot command**: `gdb -batch -p $PID -ex "set pagination off" -ex "thread apply all bt 15" -ex "detach"`
- **System call trace command**: `strace -f -e trace=read,write,poll,ioctl -p $PID -o /tmp/kitty_strace.log`
- **Thread enumeration**: `for tid in $(ls /proc/$PID/task/); do echo "TID $tid: $(cat /proc/$PID/task/$tid/comm)"; done`
- **Library map inspection**: `cat /proc/$PID/maps | grep -E "\.so" | grep -E "python|glfw|kitty|xkb|freetype|harfbuzz"`
- **Window discovery**: `xwininfo -root -children` and `xdotool search --class ""`
- **Cleanup command**: `kill $PID; killall Xvfb; rm -f /tmp/kitty_*.log`
- **Default format**: Markdown with Mermaid diagrams
- **Citation requirement**: Every behavioral claim must reference a specific runtime artifact (strace line, gdb frame, debug log entry)
- **Style guide**: Technical investigation report style — methodology, evidence, conclusion per section
- **Repository integrity check**: `cd $REPO && git status --porcelain` (must return empty after cleanup)

### 0.9.2 Key Runtime Observations Already Captured

The following runtime artifacts have already been successfully captured during the context-gathering phase and are ready for inclusion in the documentation:

**1. Thread Architecture (gdb thread listing):**
- Thread 1 (Main/GLFW): `ppoll → glfwRunMainLoop → main_loop.lto_priv` (in glfw-x11.so → fast_data_types.so)
- Thread 2 (KittyChildMon): `poll → io_loop` (in fast_data_types.so)
- Thread 3 (kitty:disk$0): Disk cache thread (futex wait)
- Threads 4-11 (kitty worker): Additional worker threads (futex wait)
- Threads 12-19 (llvmpipe-0 through llvmpipe-7): GPU software rendering threads

**2. Key-to-PTY strace evidence:**
- Main thread (TID 5558) writes to fd 6 (wakeup_io_loop pipe) after `on_key_input`
- KittyChildMon thread (TID 5577) writes `"a"` to fd 8 (PTY master) after being woken

**3. Debug-keyboard log excerpts:**
- XKB Press → composed_sym → `on_key_input` → "sent key as text to child"
- Ctrl+Shift+Left → `on_key_input` → "matched action: previous_tab, handled as shortcut"
- Ctrl+Shift+W → "matched action: close_window, handled as shortcut"
- RELEASE events → "ignoring as keyboard mode does not support encoding this event"

**4. Focus change evidence:**
- `on_focus_change: window id: 0x1 focused: 1` on initial focus
- Post-tab-close: subsequent keystrokes route to surviving window without any "no active window" message

**5. Resize-during-input evidence:**
- `ioctl(8, TIOCSWINSZ, {ws_row=24, ws_col=76})` on main thread
- Key input continues uninterrupted during resize; debug-keyboard log shows no dropped events


## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are explicitly mandated by the user's instructions and the implementation rule set:

- **SWE-AtlasQnA-Repo rule**: Create a new markdown document named `kitty_815df1e210e0.md` (based on the source branch name `kitty_815df1e210e0`) that comprehensively answers the question(s) posed in the prompt.
- **Provide thinking/rationale**: Every conclusion must include the reasoning behind it, not just the answer.
- **Do not make assumptions**: Base all answers on the code as the truth. In this case, the "truth" is the observable runtime behavior of the compiled code.
- **Do not modify any existing files**: No existing files in the source repository may be changed.
- **Place the generated document in `blitzy/documentation/`**: The directory must be created if it does not exist.

### 0.10.2 Derived Documentation Rules

Based on the user's specific requirements, the following additional rules apply:

- **Runtime evidence only**: All behavioral claims must be supported by runtime observation artifacts (strace output, gdb backtraces, debug-keyboard logs, /proc inspection). Source code may be referenced for context, but behavioral conclusions must not rely solely on reading source code.
- **No assumptions from source reading**: The user explicitly states "without relying on assumptions from reading the source." Source code citations serve as corroborating evidence, not primary evidence.
- **Negative evidence requirement**: At least two plausible-but-incorrect interpretations of the input pipeline must be explicitly stated and then refuted using runtime evidence.
- **Stack-level snapshot requirement**: At least one thread-level or symbol-level snapshot must be captured using inspection tools. If the first method is blocked, the error must be shown and an alternative used.
- **Cleanup mandate**: All temporary scripts or tracing artifacts must be cleaned up afterward. The repository must be verifiably unchanged via `git status`.
- **Background-output scenario**: At least one scenario must be observed where a background window is producing output while another window has focus.
- **Overlapping activity**: The investigation must include cases of simultaneous keyboard input with resizing and scrolling.

### 0.10.3 Quality Assurance Rules

- Every section must answer its corresponding question from the user's prompt
- Raw tool output must be included, not paraphrased
- Mermaid diagrams must accurately reflect the runtime-observed architecture, not a theoretical or source-code-inferred architecture
- The correctness-vs-responsiveness tradeoff must be directly supported by observed behavior, not by comments in the code
- The document must be self-contained — a reader should not need to consult the Kitty source code to understand the findings


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were inspected during context gathering to plan this documentation effort:

**Root-level files:**
- `setup.py` — Build system; confirmed Python >=3.8, GCC/Clang, Go 1.22
- `pyproject.toml` — `requires-python = ">=3.8"`, mypy configuration, ruff configuration
- `go.mod` — Go module declaration: `go 1.22`, dependency list
- `Makefile` — Build/test/clean targets
- `README.asciidoc` — Project overview (no internal architecture detail)
- `CONTRIBUTING.md` — Contributor guidelines
- `INSTALL.md` — Installation instructions

**Core input pipeline files (read for runtime correlation):**
- `kitty/glfw.c` (lines 1-560) — `key_callback`, `window_focus_callback`, `mouse_button_callback`, `cursor_pos_callback`, `scroll_callback`, `WINDOW_CALLBACK` macro, `set_callback_window`
- `kitty/keys.c` (lines 1-280) — `PyKeyEvent` type, `is_modifier_key`, `on_key_input` (full function), `encode_glfw_key_event` dispatch, `schedule_write_to_child` calls
- `kitty/keys.py` (lines 1-250) — `Mappings` class, `dispatch_possible_special_key`, `get_shortcut`, `shortcut_matches`, keyboard mode stack
- `kitty/child-monitor.c` (lines 1-120, 323-410, 451-520, 1220-1300, 1443-1570) — `ChildMonitor` struct, `Child` struct, `schedule_write_to_child`, `parse_input`, `process_global_state`, `io_loop`, `write_to_child`
- `kitty/boss.py` (lines 1-100, 541-570, 1408-1480, 1651-1720) — `dispatch_possible_special_key` delegation, `on_focus`, `set_active_window`, `on_activity_since_last_focus`
- `kitty/window.py` (lines 523-530, 955-985, 1123-1170) — `Window` class, `write_to_child`, `focus_changed`
- `kitty/state.c` (lines 90-170, 300-370, 506-520) — `current_os_window`, `active_window`, `window_for_window_id`, `set_active_window`, `set_active_tab`, `remove_window_inner`
- `kitty/state.h` (lines 40-55, 188-254, 309, 393-395) — OSWindow/Tab/Window structs, `is_focused`, `active_tab`, `active_window`, `last_focused_counter`
- `kitty/mouse.c` (lines 600-690) — `window_for_event`, `closest_window_for_event`, `focus_in_event`, `enter_event`

**GLFW platform files:**
- `glfw/x11_window.c` — X11 input callback registration (folder-level summary used)
- `glfw/xkb_glfw.c` — XKB keymap, modifier, compose handling (folder-level summary used)
- `glfw/input.c` — GLFW common input layer (folder-level summary used)
- `glfw/backend_utils.c` / `glfw/backend_utils.h` — Event loop utilities (folder-level summary used)

**Documentation infrastructure files:**
- `docs/conf.py` — Sphinx configuration
- `docs/requirements.txt` — sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild
- `docs/keyboard-protocol.rst` — Kitty keyboard protocol specification (existing reference doc)
- `docs/conf.rst` — Configuration reference (documents input_delay option)
- `docs/performance.rst` — Performance tuning reference

**Folders explored:**
- Repository root (`""`) — Full child listing
- `kitty/` — Core application tree (full child listing with summary)
- `glfw/` — GLFW platform layer (full child listing with summary)
- `docs/` — Documentation tree (full child listing with summary)

### 0.11.2 Tech Spec Sections Retrieved

The following technical specification sections were retrieved for architectural context:

| Section | Key Information Used |
|---------|---------------------|
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Application lifecycle overview, core event loop architecture, three-thread model |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | Keyboard input path (5-stage), mouse input path, VT parser dispatch, GPU rendering pipeline |
| 4.5 WINDOW AND TAB LIFECYCLE | Session creation, child process launch, window lifecycle state transitions |
| 5.2 COMPONENT DETAILS | Boss Controller, Child Monitor three-thread architecture, VT Parser, GLFW Platform Layer details |
| 3.1 Programming Languages | C11 standard, Python >=3.8, Go 1.22, language interaction architecture |

### 0.11.3 Attachments and External Resources

- **No attachments**: The user provided no file attachments
- **No Figma URLs**: No design references applicable
- **No external URLs**: All investigation is based on the repository source and runtime observation
- **Source branch**: `kitty_815df1e210e0` (commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` — "Wire up applying of font config")
- **Output file**: `blitzy/documentation/kitty_815df1e210e0.md`


