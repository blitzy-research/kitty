# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive analytical document** that explains how the kitty terminal emulator maintains internal state consistency during rapid, overlapping window lifecycle events—specifically when terminal windows are created, resized, and destroyed in quick succession.

The investigation targets the following questions:

- **Creation-to-command race**: When a new window is created and immediately used to run a command, what ordering guarantees exist between window registration, PTY allocation, screen initialization, and the first byte of child-process I/O?
- **Resize event propagation**: How do resize events and `SIGWINCH` signals flow through kitty's multi-threaded architecture, and what debouncing or coalescing strategies prevent redundant reflows?
- **Premature destruction**: If a window is destroyed before all pending resize events, signal deliveries, or rendering operations have completed, how does kitty decide what state to keep and what to discard?
- **Conflicting liveness views**: Are there moments where different threads or subsystems hold contradictory beliefs about whether a window is alive, and how are those conflicts resolved?
- **Signal timing**: How does the timing of signal delivery (via `signalfd` or self-pipe) interact with the I/O thread's poll loop and the main thread's state processing?

- The repository itself must remain unchanged; any temporary scripts used for observation must be cleaned up afterward
- The deliverable is a Markdown document placed in `blitzy/documentation/` answering these questions with direct code-level evidence

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint**: The existing source repository must not be modified. Only a new Markdown document may be created in the destination path (`blitzy/documentation/`).
- **Evidence-based answers**: All conclusions must be grounded in the actual source code, not assumptions or general terminal-emulator knowledge.
- **Temporary observation scripts**: If any scripts are created for experimentation or code tracing, they must be cleaned up before completion.
- **Architectural requirement**: Follow the `SWE-AtlasQnA-Repo` rule — create a markdown document named `<source_branch_name>.md` that comprehensively answers the questions.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the creation-to-command race question**, we will trace the call chain from `Boss._new_os_window()` / `Tab.new_window()` through `Window.__init__()`, `add_window()` (C, `state.c`), `Child.fork()`, `ChildMonitor.add_child()`, and the I/O thread's `add_children()` function, documenting the ordering and synchronization primitives at each stage.
- To **answer the resize-event propagation question**, we will analyze the GLFW `framebuffer_size_callback()` → `LiveResizeInfo` → `process_pending_resizes()` → `update_os_window_viewport()` → `Boss.on_window_resize()` → `TabManager.resize()` → `Tab.relayout()` → `Window.set_geometry()` → `Screen.resize()` → `ChildMonitor.resize_pty()` (TIOCSWINSZ → SIGWINCH) pipeline, and the debounce timers (`resize_debounce_time.on_pause`, `resize_debounce_time.on_end`) that coalesce rapid events.
- To **answer the premature destruction question**, we will examine `mark_child_for_close()`, `remove_children()`, `close_os_window()`, `Boss.on_child_death()`, `Window.destroy()`, `destroy_window()` (C) and the `needs_removal` flag pattern, the `WITH_OS_WINDOW` / `WITH_WINDOW` guard macros, and the `Window.destroyed` sentinel check in `set_geometry()`.
- To **answer the conflicting liveness question**, we will analyze the mutex-protected `children[]` array in the I/O thread vs. the Python `window_id_map` in the main thread, the `remove_queue`/`remove_notify` handoff, and the `CloseRequest` state machine (`NO_CLOSE_REQUESTED` → `CONFIRMABLE_CLOSE_REQUESTED` → `CLOSE_BEING_CONFIRMED` → `IMPERATIVE_CLOSE_REQUESTED`).
- To **answer the signal timing question**, we will trace the `loop-utils.c` signal infrastructure (`signalfd` / self-pipe pattern), the `handle_signal()` callback in the I/O thread, the `kill_signal_received` / `reload_config_signal_received` flags, and the `reap_children()` → `mark_child_for_removal()` → `remove_queue` → main thread `parse_input()` handoff.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation spans the following critical source files. Each was retrieved and analyzed in full to derive the conclusions documented in this plan.

**Core State Management (C layer)**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/state.h` | Defines `GlobalState`, `OSWindow`, `Tab`, `Window`, `LiveResizeInfo`, `CloseRequest` enum, `WITH_OS_WINDOW`/`WITH_WINDOW` guard macros | Central data structures for the entire window hierarchy; contains the `has_pending_resizes`, `has_pending_closes` flags and the `LiveResizeInfo` struct that tracks debounce state |
| `kitty/state.c` | Implements `add_os_window()`, `add_tab()`, `add_window()`, `remove_window()`, `remove_os_window()`, `destroy_window()`, `destroy_tab()`, `detach_window()`, `attach_window()`, `resize_screen()`, `mark_os_window_for_close()` | All CRUD operations on the window/tab/os-window hierarchy; array-based storage with `REMOVER` macro for O(n) scans |
| `kitty/child-monitor.c` | Three-thread event loop: main thread (`process_global_state`, `parse_input`, `render`), I/O thread (`io_loop`), talk thread | Signal handling (`handle_signal`), child reaping (`reap_children`), resize-pty via `TIOCSWINSZ`, `process_pending_resizes()`, `process_pending_closes()`, `close_os_window()`, `mark_child_for_close()` |
| `kitty/screen.c` | `screen_resize()`, `screen_pause_rendering()`, `screen_check_pause_rendering()` | Screen buffer reflow logic during resize, pause-rendering snapshot mechanism |
| `kitty/glfw.c` | `update_os_window_viewport()`, `framebuffer_size_callback()`, `live_resize_callback()`, `dpi_change_callback()`, `window_close_callback()`, `change_live_resize_state()` | GLFW callback entry points for resize and close events, debounce trigger logic |
| `kitty/loop-utils.c` | Signal infrastructure: `signalfd`/self-pipe pattern, `init_signal_handlers()`, `read_signals()`, `wakeup_loop()` | Portable signal delivery mechanism used by both I/O and Python threads |

**Python Orchestration Layer**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/boss.py` | `Boss` singleton class: `on_window_resize()`, `on_child_death()`, `close_window()`, `_cleanup_tab_after_window_removal()`, `confirm_os_window_close()`, `on_os_window_closed()`, `destroy()`, `mark_window_for_close()` | High-level lifecycle orchestration; bridges C state operations to Python logic |
| `kitty/window.py` | `Window` class: `__init__()`, `set_geometry()`, `destroy()`, `destroyed` flag, `last_reported_pty_size`, `last_resized_at` | Per-window state tracking, geometry/resize application, destruction guard |
| `kitty/tabs.py` | `Tab.relayout()`, `Tab.remove_window()`, `Tab.destroy()`, `TabManager.resize()` | Layout recalculation chain after resize, window removal from tab hierarchy |
| `kitty/window_list.py` | `WindowList.add_window()`, `WindowList.remove_window()`, active group management | Group-level window tracking with notification on active-window changes |
| `kitty/child.py` | `Child.fork()`, `openpty()`, PTY creation, process spawning via `fast_data_types.spawn()` | Child-process lifecycle and PTY allocation |
| `kitty/main.py` | `_run_app()`, `_main()`, `mask_kitty_signals_process_wide()` | Application startup, signal masking before thread creation, main loop entry |

**Layout System**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/layout/base.py` | `Layout.__call__()` → `_set_dimensions()` → `update_visibility()` → `do_layout()` | Base layout framework invoked during every relayout after resize |
| `kitty/layout/tall.py`, `kitty/layout/grid.py`, `kitty/layout/splits.py`, `kitty/layout/vertical.py`, `kitty/layout/stack.py` | Seven tiling algorithms | Geometry computation for each window during relayout |

**Configuration and Options**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/options/definition.py` | `resize_debounce_time` option definition | Configurable debounce intervals for resize events |
| `kitty/options/types.py` | Typed `Options` class with `resize_debounce_time` fields | Runtime access to debounce configuration |

### 0.2.2 Web Search Research Conducted

No external web research was required for this task. All answers are derived exclusively from source code analysis of the kitty repository at commit `815df1e210e0`.

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/<source_branch_name>.md` — Comprehensive markdown document answering all questions about kitty's internal state consistency during rapid window lifecycle events
- No other files are created or modified


## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

This task is a documentation/analysis exercise that produces a Markdown file. No new package dependencies are introduced.

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| PyPI (built-in) | Python | >=3.8 (runtime: 3.12.3) | Kitty's Python runtime for boss, window, and tab orchestration |
| Go modules | Go | 1.22 | Kitty's Go tools layer (`tools/`) for the `kitten` binary |
| System | GCC | 13.x | C compiler for kitty's native C extensions |
| System | pkg-config | system | Build dependency resolution |
| System | libharfbuzz-dev | system | Text shaping for the font subsystem |
| System | libssl-dev | 3.0.13 | Cryptographic operations (remote control) |
| System | libxxhash-dev | 0.8.2 | Fast hashing for internal data structures |
| System | libsimde-dev | 0.7.2 | SIMD portability for string operations |

### 0.3.2 Dependency Updates

No dependency updates are required. This task produces a read-only analysis document and does not modify any source files, build configurations, or dependency manifests.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The analysis document must trace integration points across kitty's three-thread architecture. Below are the critical touchpoints discovered through source code inspection.

**Thread Boundary Crossings (State Consistency Points)**

- `kitty/child-monitor.c` — `children_mutex` protects the shared `children[]` array, `add_queue[]`, `remove_queue[]`, and `remove_notify[]` arrays. The I/O thread writes `needs_removal = true`; the main thread reads it in `parse_input()`.
- `kitty/child-monitor.c` — `talk_mutex` protects the `messages[]` array shared between the talk thread and main thread.
- `kitty/child-monitor.c` — `screen_mutex(lock, write)` / `screen_mutex(unlock, write)` protects the per-screen `write_buf` shared between the main thread (which schedules writes) and the I/O thread (which performs writes).

**Resize Event Flow (Direct Modifications)**

- `kitty/glfw.c:framebuffer_size_callback()` (line ~330): Sets `global_state.has_pending_resizes = true`, records `live_resize.last_resize_event_at`, and calls `request_tick_callback()`.
- `kitty/child-monitor.c:process_pending_resizes()` (line ~1063): Reads `LiveResizeInfo` fields, applies debounce logic, calls `update_os_window_viewport()` and `change_live_resize_state()`.
- `kitty/glfw.c:update_os_window_viewport()` (line ~130): Computes viewport dimensions, calls `Boss.on_window_resize()` via `call_boss`.
- `kitty/boss.py:on_window_resize()` (line ~1206): Delegates to `TabManager.resize()`.
- `kitty/tabs.py:TabManager.resize()` (line ~963): Calls `tab.relayout()` for every tab.
- `kitty/tabs.py:Tab.relayout()` (line ~298): Invokes the current layout's `__call__()` which calls `WindowGroup.set_geometry()`.
- `kitty/window.py:Window.set_geometry()` (line ~850): Calls `self.screen.resize()`, `boss.child_monitor.resize_pty()`, and `set_window_render_data()`.
- `kitty/child-monitor.c:resize_pty()` (line ~582): Issues `ioctl(fd, TIOCSWINSZ, &dim)` to deliver `SIGWINCH` to the child.

**Window Destruction Flow (Direct Modifications)**

- `kitty/child-monitor.c:mark_child_for_close()` (line ~465): Sets `children[i].needs_removal = true` under `children_mutex`.
- `kitty/child-monitor.c:remove_children()` (I/O thread, line ~1294): Calls `cleanup_child()` (closes fd, sends `SIGHUP` to process group), moves child to `remove_queue[]`.
- `kitty/child-monitor.c:parse_input()` (main thread, line ~380): Drains `remove_queue[]` into `remove_notify[]`, calls `death_notify` for each, then processes remaining children.
- `kitty/boss.py:on_child_death()` (line ~882): Pops window from `window_id_map`, calls `window.destroy()`, calls `tab.remove_window()`, invokes `_cleanup_tab_after_window_removal()`.
- `kitty/state.c:remove_window()` / `remove_os_window()`: Array-based removal with `REMOVER` macro, destroys GPU resources and Python references.

**Signal Delivery Integration**

- `kitty/loop-utils.c:init_signal_handlers()`: Creates `signalfd` (Linux) or self-pipe with `SA_SIGINFO` handlers for `SIGINT`, `SIGHUP`, `SIGTERM`, `SIGCHLD`, `SIGUSR1`, `SIGUSR2`.
- `kitty/child-monitor.c:io_loop()` (line ~1476): Polls `children_fds[1]` (signal fd) alongside PTY fds. On `POLLIN`, calls `read_signals()` with `handle_signal()` callback that sets `kill_signal`, `child_died`, `reload_config` flags.
- `kitty/child-monitor.c:reap_children()`: Non-blocking `waitpid(-1, &status, WNOHANG)` loop; marks children for removal and records monitored PID deaths.
- `kitty/child-monitor.c:parse_input()` (main thread): Reads `kill_signal_received` and `reload_config_signal_received` under mutex, converts to `IMPERATIVE_CLOSE_REQUESTED` or config reload.


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

The user's requirement is a **read-only analysis** of existing kitty source code. The repository itself must remain unchanged. All deliverables are confined to a single new markdown document placed in `blitzy/documentation/`. Temporary observation scripts may be created during analysis but must be cleaned up afterward.

**Group 1 — Primary Analysis Targets (Read Only)**

| Action | File | Purpose in Analysis |
|--------|------|-------------------|
| READ | `kitty/state.h` | Extract `GlobalState`, `OSWindow`, `LiveResizeInfo`, `CloseRequest` type definitions and flag semantics |
| READ | `kitty/state.c` | Trace `add_os_window()`, `remove_os_window()`, `destroy_window()`, `mark_os_window_for_close()`, `resize_screen()` logic |
| READ | `kitty/child-monitor.c` | Analyze `process_pending_resizes()`, `process_pending_closes()`, `parse_input()`, `close_os_window()`, `mark_child_for_close()`, `reap_children()`, `io_loop()`, `resize_pty()` |
| READ | `kitty/glfw.c` | Analyze `framebuffer_size_callback()`, `live_resize_callback()`, `update_os_window_viewport()`, `window_close_callback()`, `change_live_resize_state()` |
| READ | `kitty/screen.c` | Analyze `screen_resize()`, `screen_pause_rendering()`, buffer reallocation and reflow |
| READ | `kitty/loop-utils.c` | Analyze `signalfd`/self-pipe setup, `read_signals()`, `wakeup_loop()` |

**Group 2 — Python Layer Targets (Read Only)**

| Action | File | Purpose in Analysis |
|--------|------|-------------------|
| READ | `kitty/boss.py` | Trace `on_window_resize()`, `on_child_death()`, `_cleanup_tab_after_window_removal()`, `confirm_os_window_close()`, `on_os_window_closed()`, `destroy()` |
| READ | `kitty/window.py` | Trace `Window.__init__()`, `Window.set_geometry()` destroyed guard, `Window.destroy()` flag setting |
| READ | `kitty/tabs.py` | Trace `Tab.relayout()`, `Tab.remove_window()`, `TabManager.resize()` |
| READ | `kitty/window_list.py` | Trace `WindowList` group management, active-window tracking |
| READ | `kitty/child.py` | Trace `Child.fork()`, PTY creation, process-group setup |
| READ | `kitty/main.py` | Trace `mask_kitty_signals_process_wide()` ordering, `run_app()` lifecycle |

**Group 3 — Deliverable File (Create)**

| Action | File | Purpose |
|--------|------|---------|
| CREATE | `blitzy/documentation/<source_branch_name>.md` | Comprehensive analysis document answering all questions about state consistency, timing, signal delivery, and conflicting-view resolution |

### 0.5.2 Implementation Approach per File

The analysis document will be organized around five key investigation traces, each grounded in specific source files:

**Trace 1 — Rapid Window Creation and Immediate Command Execution**
- Source evidence from `kitty/state.c:add_os_window()`, `add_tab()`, `add_window()` and `kitty/child.py:Child.fork()`
- Document the sequence: `GlobalState.os_windows[]` allocation → Screen creation (24×80 default) → PTY open → `spawn()` → child added to `add_queue[]` under `children_mutex` → I/O thread picks up child on next poll cycle
- Key finding: the child process can begin producing output before the I/O thread has incorporated it into its poll set; output buffered by kernel PTY layer until polled

**Trace 2 — Resize During Live Resize with Debounce**
- Source evidence from `kitty/glfw.c:framebuffer_size_callback()`, `kitty/child-monitor.c:process_pending_resizes()`, `kitty/screen.c:screen_resize()`
- Document the two debounce strategies:
  - `from_os_notification` path: `live_resize_callback(true)` sets `in_progress`, subsequent `framebuffer_size_callback` events accumulate; `process_pending_resizes()` waits for `os_says_resize_complete` or `on_pause` timeout before applying
  - Non-notification path: main loop checks `last_resize_event_at` against `on_end` debounce; applies when sufficient time elapsed since last event
- Key finding: intermediate resize dimensions are discarded; only the final dimensions are applied to Screen buffers, preventing wasted reflow work

**Trace 3 — Window Destroyed Before Resize Completes**
- Source evidence from `kitty/window.py:Window.set_geometry()` (`if self.destroyed: return` guard), `kitty/child-monitor.c:process_pending_closes()` ordering (resizes processed before closes), `kitty/state.c:remove_os_window()`
- Document the race-prevention mechanisms:
  - Main-loop ordering: `process_pending_resizes()` runs before `process_pending_closes()`, ensuring pending resizes are either applied or abandoned before destruction
  - `WITH_OS_WINDOW`/`WITH_WINDOW` macros silently no-op if the target ID no longer exists in the `global_state` arrays
  - Python `destroyed` flag checked at entry to `set_geometry()` and other state-modifying methods

**Trace 4 — Signal Timing and Deferred Processing**
- Source evidence from `kitty/main.py:mask_kitty_signals_process_wide()`, `kitty/loop-utils.c`, `kitty/child-monitor.c:handle_signal()`, `parse_input()`
- Document the signal deferral architecture:
  - All `KITTY_HANDLED_SIGNALS` masked process-wide before GLFW starts display threads — ensures only the I/O thread receives signals via `signalfd`/self-pipe
  - I/O thread converts kernel signals into boolean flags (`kill_signal`, `child_died`, `reload_config`) protected by `children_mutex`
  - Main thread reads flags in `parse_input()`, converts to high-level actions (mark all windows for close, reload configuration, trigger child reaping)
  - No signal handler runs on the main thread — eliminates async-signal-safety concerns in Python and rendering code

**Trace 5 — Conflicting Views of Liveness**
- Source evidence from `kitty/child-monitor.c:close_os_window()`, `kitty/boss.py:on_child_death()`, `_cleanup_tab_after_window_removal()`
- Document the resolution strategies:
  - Child dies but window still in `global_state`: I/O thread sets `needs_removal`, moves to `remove_queue`; main thread drains to `remove_notify`, calls `death_notify` which triggers Python `on_child_death()` → `window.destroy()` → C layer `remove_window()`
  - OS window closes but children still alive: `close_os_window()` calls `destroy_os_window()` (GPU cleanup), then iterates all tabs/windows calling `mark_child_for_close()` for each child; I/O thread sends `SIGHUP` + `SIGKILL` (after 2s) during `cleanup_child()`
  - Both close and resize pending simultaneously: main loop ordering (`resizes` then `closes`) ensures resize is either completed or the OS window is found missing by `WITH_OS_WINDOW` macro and skipped

### 0.5.3 User Interface Design

Not applicable. This task produces a markdown analysis document and does not involve UI creation or modification. The kitty terminal's existing UI behavior (window rendering, resize overlays showing "NxM cells" text during live resize, tab bar relayout) is a subject of analysis only.


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**C-Layer Source Files (Read-Only Analysis)**

- `kitty/state.h` — Type definitions for `GlobalState`, `OSWindow`, `Tab`, `Window`, `LiveResizeInfo`, `CloseRequest` enum
- `kitty/state.c` — All CRUD operations on the global state hierarchy: `add_os_window()`, `add_tab()`, `add_window()`, `remove_os_window()`, `destroy_window()`, `destroy_tab()`, `mark_os_window_for_close()`, `resize_screen()`
- `kitty/child-monitor.c` — Three-thread main loop, `process_pending_resizes()`, `process_pending_closes()`, `parse_input()`, `close_os_window()`, `mark_child_for_close()`, `reap_children()`, `io_loop()`, `resize_pty()`, signal handling integration
- `kitty/glfw.c` — GLFW event callbacks: `framebuffer_size_callback()`, `live_resize_callback()`, `update_os_window_viewport()`, `window_close_callback()`, `change_live_resize_state()`
- `kitty/screen.c` — `screen_resize()`, `screen_pause_rendering()`, buffer reallocation and reflow logic
- `kitty/loop-utils.c` — `signalfd`/self-pipe signal infrastructure, `read_signals()`, `wakeup_loop()`
- `kitty/data-types.h` — Supporting type definitions and macros used by C modules

**Python-Layer Source Files (Read-Only Analysis)**

- `kitty/boss.py` — `on_window_resize()`, `on_child_death()`, `_cleanup_tab_after_window_removal()`, `confirm_os_window_close()`, `on_os_window_closed()`, `destroy()`
- `kitty/window.py` — `Window.__init__()`, `Window.set_geometry()` with `destroyed` guard, `Window.destroy()`
- `kitty/tabs.py` — `Tab.relayout()`, `Tab.remove_window()`, `TabManager.resize()`
- `kitty/window_list.py` — `WindowList` group management and active-window tracking
- `kitty/child.py` — `Child.fork()`, PTY creation via `os.openpty()`, spawn logic
- `kitty/main.py` — `mask_kitty_signals_process_wide()`, `run_app()`, application startup ordering
- `kitty/layout/base.py` — Layout `__call__()`, `_set_dimensions()`, `update_visibility()`, `set_window_group_geometry()`

**Configuration and Build Files (Read-Only Analysis)**

- `kitty/options/types.py` — `Options` class with `resize_debounce`, `confirm_os_window_close`, `repaint_delay` fields
- `kitty/options/definition.py` — Option declarations with default values and docs
- `pyproject.toml` — Python version and build system configuration
- `go.mod` — Go version requirements
- `setup.py` — Build system and C extension compilation

**Deliverable Files (Create)**

- `blitzy/documentation/<source_branch_name>.md` — The comprehensive analysis document

**Temporary Files (Create and Remove)**

- Any observation scripts created during analysis in `/tmp/` — must be cleaned up after use

### 0.6.2 Explicitly Out of Scope

- **Repository modifications**: No existing files in the kitty source tree will be modified, moved, or deleted
- **New feature code**: No new C, Python, or Go source code will be added to the kitty source tree
- **Dependency changes**: No packages will be added, removed, or upgraded in the project's dependency manifests
- **Build system changes**: No modifications to `setup.py`, `pyproject.toml`, `Makefile`, or CI/CD configurations
- **Performance optimization**: The analysis document describes existing behavior; it does not propose or implement optimizations
- **Refactoring**: No restructuring of existing code modules, even where the analysis may identify potential improvements
- **Unrelated subsystems**: The kitty kittens framework (`kitty/kittens/`), remote control protocol (`kitty/rc/`), shell integration (`shell-integration/`), Go tools (`tools/`), and font subsystem (`kitty/fonts/`) are not primary analysis targets unless they directly interact with window lifecycle or resize events
- **Wayland-specific paths**: The build was compiled with X11 backend only; Wayland-specific resize or close behavior in `kitty/glfw.c` conditional blocks is noted but not deeply traced
- **GPU shader internals**: The rendering pipeline (`kitty/shaders.c`, `kitty/gpu/`) is referenced only insofar as `set_window_render_data()` and `screen_pause_rendering()` interact with state consistency; shader implementation details are out of scope


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly stated by the user and in the project implementation rules:

- **Read-only repository**: The existing kitty source repository must remain completely unchanged. No files may be modified, added, or deleted within the source tree.
- **Markdown deliverable**: Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed in the prompt.
- **Placement**: The generated document must be placed in the `blitzy/documentation/` directory in the destination repo.
- **Evidence-based answers**: Do not make assumptions; base all answers on the code as the truth. Provide thinking and rationale behind the answers.
- **Temporary cleanup**: Temporary scripts may be used for observation during analysis, but must be cleaned up afterward. Nothing temporary should persist in the repository.

### 0.7.2 Analysis Quality Requirements

Derived from the user's questions, the analysis document must address each of these specific questions with code-level evidence:

- **State consistency during rapid lifecycle events**: How does kitty keep internal state consistent when windows appear, resize, and disappear in quick succession? The answer must trace through `GlobalState` → `OSWindow` → `Tab` → `Window` hierarchy maintenance.
- **Resize and signal flow on new windows**: What happens when a new window is created and immediately used to run a command while resize events and signals start flowing? The answer must trace the `framebuffer_size_callback` → `process_pending_resizes` → `screen_resize` → `resize_pty` → `SIGWINCH` pipeline.
- **Premature window destruction**: What happens if the window is gone before everything has finished reacting to resize or signal changes? The answer must document the `destroyed` guard in `set_geometry()`, the `WITH_OS_WINDOW`/`WITH_WINDOW` macro no-op behavior, and main-loop ordering guarantees.
- **State retention vs. discard decisions**: How does kitty decide what state to keep and what to discard? The answer must cover debounce strategies (intermediate resize dimensions discarded), the `remove_queue`/`remove_notify` handoff (deferred cleanup), and the `CloseRequest` state machine.
- **Signal timing**: How does timing affect signal delivery and internal bookkeeping? The answer must cover `mask_kitty_signals_process_wide()` ordering, `signalfd`/self-pipe deferral, mutex-protected flag propagation from I/O to main thread.
- **Conflicting liveness views**: Are there moments where the system has to resolve conflicting views of what is still alive? The answer must document the child-dead-but-window-alive and window-closed-but-children-alive scenarios, including the `SIGHUP`/`SIGKILL` escalation in `cleanup_child()`.

### 0.7.3 Documentation Standards

- All claims in the analysis document must cite specific source files and function names
- Code-level evidence must reference actual variable names, struct fields, and function signatures found in the repository
- The document must be organized to address each of the user's questions as distinct sections with clear headings
- Mermaid diagrams or ASCII flow diagrams should illustrate the most complex interaction sequences (resize pipeline, destruction cascade, signal deferral chain)


## 0.8 References

### 0.8.1 Repository Files and Folders Searched

**C-Layer Source Files**

| File Path | Lines Analyzed | Key Content Discovered |
|-----------|---------------|----------------------|
| `kitty/state.h` | 1–401 | `GlobalState`, `OSWindow`, `Tab`, `Window` structs; `LiveResizeInfo`, `CloseRequest` enum; `RENDER_STATE` enum; function declarations for state CRUD |
| `kitty/state.c` | 1–1492 | `add_os_window()`, `add_tab()`, `add_window()`, `destroy_window()`, `remove_window_inner()`, `destroy_tab()`, `remove_os_window()`, `mark_os_window_for_close()`, `resize_screen()`; `WITH_OS_WINDOW`/`WITH_TAB`/`WITH_WINDOW` macros; `REMOVER` macro; `DetachedWindows` struct |
| `kitty/child-monitor.c` | 1–2016 | `io_loop()`, `parse_input()`, `process_pending_resizes()`, `process_pending_closes()`, `close_os_window()`, `mark_child_for_close()`, `reap_children()`, `resize_pty()`, `render()`; `children_mutex`/`talk_mutex` synchronization; `SignalSet` flag struct; `remove_queue`/`remove_notify` handoff |
| `kitty/glfw.c` | 1–2525 | `framebuffer_size_callback()`, `live_resize_callback()`, `update_os_window_viewport()`, `window_close_callback()`, `change_live_resize_state()`, `set_callback_window()`; viewport ratio computation; swap interval adjustment during resize |
| `kitty/screen.c` | 1–4932 | `screen_resize()`, `screen_pause_rendering()`; line buffer reallocation with reflow via `realloc_lb()`; cursor tracking preservation; prompt preservation; graphics manager resize; tabstop reset |
| `kitty/loop-utils.c` | Full file | `init_signal_handlers()` with `signalfd` (Linux) and self-pipe (macOS/OpenBSD); `read_signals()`; `wakeup_loop()` via `eventfd` or pipe; `SFD_NONBLOCK\|SFD_CLOEXEC` flags |
| `kitty/data-types.h` | Partial | Supporting macro definitions and type aliases |

**Python-Layer Source Files**

| File Path | Lines Analyzed | Key Content Discovered |
|-----------|---------------|----------------------|
| `kitty/boss.py` | 1–3094 | `on_window_resize()`, `on_child_death()`, `_cleanup_tab_after_window_removal()`, `confirm_os_window_close()`, `on_os_window_closed()`, `destroy()`; `window_id_map` management; cascading tab/tab-manager cleanup |
| `kitty/window.py` | 1–1998 | `Window.__init__()` (Screen creation, C-layer registration via `add_window()`); `Window.set_geometry()` with `if self.destroyed: return` guard; `Window.destroy()` setting `destroyed = True`; watcher notification |
| `kitty/tabs.py` | 1–1268 | `Tab.relayout()`, `Tab.remove_window()`, `TabManager.resize()`, `TabManager.tab_bar_should_be_visible`; window-list management |
| `kitty/window_list.py` | 1–442 | `WindowList` class: group management, active-window tracking, history, notification callbacks |
| `kitty/child.py` | 1–500 | `Child.fork()`: `os.openpty()`, sync pipe creation, `fast_data_types.spawn()`, non-blocking master FD setup |
| `kitty/main.py` | 1–full | `_main()`: arg parsing, `mask_kitty_signals_process_wide()` before `init_glfw()`, `run_app()` → `child_monitor.main_loop()` → `boss.destroy()` |
| `kitty/layout/base.py` | 1–full | Layout `__call__()`: `_set_dimensions()` → `update_visibility()` → `do_layout()`; `set_window_group_geometry()` |

**Configuration and Build Files**

| File Path | Purpose |
|-----------|---------|
| `pyproject.toml` | Python version requirement (>=3.8), build system config |
| `go.mod` | Go version requirement (1.22) |
| `setup.py` | C extension compilation, dependency checks |
| `kitty/options/types.py` | `Options` class with `resize_debounce`, `confirm_os_window_close`, `repaint_delay` |
| `kitty/options/definition.py` | Option declarations, default values, documentation strings |

**Folders Explored**

| Folder Path | Depth | Content Summary |
|-------------|-------|----------------|
| Root (`""`) | 0 | Top-level repository structure: `kitty/`, `kittens/`, `tools/`, `docs/`, `shell-integration/`, build files |
| `kitty/` | 1 | Core source: C files, Python modules, options/, layout/, fonts/ |
| `kitty/layout/` | 2 | Layout engines: `base.py`, `tall.py`, `splits.py`, `grid.py`, `stack.py`, `fat.py`, `horizontal.py`, `vertical.py` |
| `kitty/options/` | 2 | Configuration system: `types.py`, `definition.py`, `parse.py`, `utils.py` |

### 0.8.2 Technical Specification Sections Retrieved

| Section Heading | Key Information Extracted |
|----------------|-------------------------|
| 4.5 WINDOW AND TAB LIFECYCLE | Session creation flow, child process launch, window lifecycle state transitions (Creating → Configuring → Spawning → Active → Focused/Resized/Scrollback/Overlay → ChildExited → Closing) |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | Input processing from keyboard/PTY through VT parser to GPU rendering pipeline; data flow between threads |
| 5.2 COMPONENT DETAILS | Descriptions of all 14 components: launcher, boss, child monitor, VT parser, GPU pipeline, font subsystem, GLFW, config, layout, RC, kittens, shell integration, Go tools, codegen |
| 4.10 ERROR HANDLING AND RECOVERY FLOWS | Startup error handling, runtime protocol error recovery, extension error isolation patterns |

### 0.8.3 Attachments

No file attachments were provided by the user. No Figma URLs or design assets were referenced.

### 0.8.4 External Research

No web searches were required for this analysis task. All findings are derived directly from the kitty source code repository at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.


