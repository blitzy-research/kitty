# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to produce a comprehensive technical analysis document that answers deep-dive questions about kitty's internal state consistency mechanisms during rapid window lifecycle events. Specifically:

- **Primary objective**: Create a new markdown document named `kitty_815df1e210e0.md` placed in `blitzy/documentation/` that comprehensively answers how kitty keeps its internal state consistent when terminal windows appear, resize, and disappear in quick succession.
- **Question scope**: The analysis must cover what happens when a new window is created and immediately used to run a command while resize events and signals start flowing through the system, and what occurs if the window is gone before everything has finished reacting to those changes.
- **State management focus**: How kitty decides what state to keep and what to discard, how timing affects signal delivery and internal bookkeeping, and whether there are moments where the system has to resolve conflicting views of what is still alive.
- **Implicit requirement — Threading model documentation**: The questions inherently require understanding kitty's three-thread architecture (main thread, I/O thread, talk thread) and how they coordinate through mutexes, wakeup FDs, and signal FDs.
- **Implicit requirement — Debounce and guard mechanism documentation**: Answering "what happens during rapid succession" requires documenting the `LiveResizeInfo` state machine, `process_pending_resizes()` debouncing, the `destroyed` flag on Python Window objects, and the no-op viewport guard.
- **Implicit requirement — Queue transfer pipeline documentation**: Understanding what happens when a window disappears mid-operation requires tracing the `add_queue → children[] → remove_queue → remove_notify[]` pipeline and the mutex-protected transitions between them.

### 0.1.2 Special Instructions and Constraints

- **No repository modification**: The implementation rules explicitly state "Do not modify any existing files in the source repository" and "Do not add any other code in the source repository (besides the above requested document)."
- **Document-only output**: The sole deliverable is a markdown document at `blitzy/documentation/kitty_815df1e210e0.md`.
- **Temporary scripts**: The user mentions that "temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward." No temporary scripts were needed as all analysis was conducted through code reading.
- **Rationale requirement**: Per the implementation rules, the document must "provide thinking / rationale behind the answers."
- **Code-grounded answers**: Per the implementation rules, "Do not make assumptions, base your answers on the code as the truth."

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To produce the analysis document, we will read and analyze the core source files governing window lifecycle, state management, signal handling, and thread coordination in the kitty codebase.
- To answer the state consistency questions, we will trace the complete execution path through `kitty/state.h` (data structures), `kitty/state.c` (state mutations), `kitty/child-monitor.c` (event loop and cross-thread coordination), `kitty/glfw.c` (GLFW callbacks and viewport management), `kitty/boss.py` (Python orchestration), `kitty/window.py` (window lifecycle), `kitty/tabs.py` (tab/layout management), and `kitty/window_list.py` (window group management).
- To document signal delivery timing, we will analyze `kitty/loop-utils.c` (signal FD mechanism) and the signal handling path in `kitty/child-monitor.c`.
- To create the document, we will write a comprehensive markdown file organized by topic (architecture, lifecycle, signals, consistency mechanisms, conflict resolution, cascade cleanup, edge cases) with detailed rationale sections grounded in specific code references.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The following files were identified as relevant to answering the user's questions about state consistency during rapid window lifecycle events. All files were read and analyzed in their entirety.

**Core C State Management:**

| File | Lines | Purpose | Relevance |
|------|-------|---------|-----------|
| `kitty/state.h` | ~350 | Defines `GlobalState`, `OSWindow`, `Tab`, `Window`, `LiveResizeInfo`, `CloseRequest` structs | Central data structures for all window state; defines the `LiveResizeInfo` debounce state machine and `CloseRequest` enum |
| `kitty/state.c` | ~1492 | Implements `add_os_window()`, `add_window()`, `remove_window()`, `remove_os_window()`, `destroy_window()`, `mark_os_window_for_close()`, `resize_screen()`, `detach_window()`/`attach_window()`, `REMOVER` macro | All state mutation operations; the `REMOVER` macro is the atomic array-removal primitive; `WITH_OS_WINDOW`/`WITH_TAB`/`WITH_WINDOW` macros provide safe ID-based lookups |
| `kitty/child-monitor.c` | ~2016 | Three-thread event loop: `io_loop()`, `parse_input()`, `process_global_state()`, `process_pending_resizes()`, `process_pending_closes()`, `close_os_window()`, `mark_child_for_close()`, `resize_pty()`, `remove_children()`, `reap_children()` | The central coordination point; `process_global_state()` defines the strict execution order (resizes → input → render → closes) that ensures state consistency |
| `kitty/loop-utils.c` | ~120 | Signal FD setup (`signalfd()` on Linux, self-pipe on macOS), `wakeup_loop()`, `read_signals()` | Signal delivery mechanism; converts async signals into synchronous pollable events |
| `kitty/glfw.c` | ~500+ | GLFW callbacks: `framebuffer_size_callback()`, `live_resize_callback()`, `dpi_change_callback()`, `window_close_callback()`, `update_os_window_viewport()` | Entry points for resize and close events from the OS; the no-op viewport guard lives here |
| `kitty/child.py` | ~501 | `Child` class: `fork()` (PTY creation), `openpty()`, FD management | How child processes are spawned with PTY master/slave pairs |

**Python Orchestration:**

| File | Lines | Purpose | Relevance |
|------|-------|---------|-----------|
| `kitty/boss.py` | ~3094 | `Boss` class: `on_child_death()`, `on_window_resize()`, `on_os_window_closed()`, `mark_window_for_close()`, `suppress_focus_change_events()`, `_cleanup_tab_after_window_removal()`, `destroy()` | Top-level orchestration of all lifecycle events; the cascade cleanup pattern (window → tab → OS window) is implemented here |
| `kitty/window.py` | ~1998 | `Window` class: `__init__()`, `set_geometry()`, `focus_changed()`, `destroy()` | Window-level lifecycle; the `destroyed` guard in `set_geometry()` and `ignore_focus_changes` guard in `focus_changed()` are critical consistency mechanisms |
| `kitty/tabs.py` | ~1268 | `Tab` class: `new_window()`, `remove_window()`, `relayout()`, `destroy()`; `TabManager` class: `resize()`, `_remove_tab()` | Tab-level layout management; `relayout()` triggers `set_geometry()` on all windows; `TabManager.resize()` is the entry point for resize propagation |
| `kitty/window_list.py` | ~350 | `WindowList` and `WindowGroup` classes: `add_window()`, `remove_window()`, active group management | Window group management within tabs; handles overlay windows and focus tracking after removal |

**Test Files Identified:**

| File | Relevance |
|------|-----------|
| `kitty_tests/layout.py` | Tests for `WindowList.remove_window()` and layout computations after window removal |
| `kitty_tests/screen.py` | Tests for screen resize behavior |
| `kitty_tests/__init__.py` | Test infrastructure with resize/close references |

### 0.2.2 Web Search Research Conducted

No external web searches were required. All answers are derived directly from reading the source code, per the implementation rule: "Do not make assumptions, base your answers on the code as the truth."

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — The comprehensive analysis document answering the user's questions about kitty's state consistency mechanisms during rapid window lifecycle events. This is the sole deliverable.

No other files are created or modified.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

This task is a read-only analysis producing a markdown document. No new dependencies are introduced, and no existing dependencies are modified. The kitty project's existing dependency stack was reviewed only for context:

| Package Registry | Name | Version | Purpose in Analysis |
|-----------------|------|---------|-------------------|
| System | Python | 3.x (as per pyproject.toml) | Runtime for kitty's Python orchestration layer (boss.py, window.py, tabs.py) |
| System | GCC/Clang | N/A | Compiled the C extension modules (state.c, child-monitor.c, glfw.c) |
| System | GLFW | Vendored (in `glfw/` directory) | Provides window management callbacks analyzed in glfw.c |
| PyPI | N/A | N/A | No Python packages are added or modified |

### 0.3.2 Dependency Updates

No dependency updates are required. This task:
- Does not add any new imports to any existing file
- Does not modify any configuration files
- Does not alter any build files
- Does not change any CI/CD pipelines

The only output is the creation of a new markdown file in a new directory (`blitzy/documentation/`).

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

This is a documentation-only task. No existing code is modified. However, the analysis required deep understanding of the following integration points within kitty's architecture, which are documented in the output markdown:

**Cross-Thread Integration Points Analyzed:**

- `kitty/child-monitor.c` ↔ `kitty/state.h`: The I/O thread reads and writes `children[]`, `add_queue[]`, `remove_queue[]` under the `children_mutex`. The main thread's `parse_input()` drains `remove_queue` into `remove_notify[]` under the same mutex, then processes deaths outside the lock.
- `kitty/child-monitor.c` ↔ `kitty/boss.py`: The C function `parse_input()` calls `self->death_notify` (Python's `boss.on_child_death()`) and `call_boss()` for close confirmations. These cross the C/Python boundary via CPython API calls.
- `kitty/glfw.c` ↔ `kitty/child-monitor.c`: GLFW callbacks set flags on `OSWindow` structs (`has_pending_resizes`, `close_request`), which `process_global_state()` reads on each tick.
- `kitty/glfw.c` ↔ `kitty/boss.py`: `update_os_window_viewport()` calls `boss.on_window_resize()` to trigger the Python-level relayout cascade.
- `kitty/boss.py` ↔ `kitty/window.py` ↔ `kitty/tabs.py`: The Boss delegates resize to `TabManager.resize()` → `Tab.relayout()` → `Window.set_geometry()`. Death cleanup flows `boss.on_child_death()` → `window.destroy()` → `tab.remove_window()`.
- `kitty/window.py` ↔ `kitty/child-monitor.c`: `Window.set_geometry()` calls `child_monitor.resize_pty()` which performs `ioctl(TIOCSWINSZ)` under the mutex.

**State Consistency Boundaries Analyzed:**

- The `destroyed` flag on Python `Window` objects serves as the boundary between "structurally valid" and "logically dead" — code paths check this flag before accessing window state.
- The `CloseRequest` enum on `OSWindow` structs serves as the boundary between "alive", "pending confirmation", "being confirmed", and "imperatively closing".
- The `needs_removal` flag on `Child` structs in `children[]` serves as the boundary between "active" and "scheduled for removal" in the I/O thread.

### 0.4.2 Database/Schema Updates

No database or schema updates are required. Kitty does not use a database.

### 0.4.3 Dependency Injections

No dependency injections or service registrations are required. The task creates only a documentation file.

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

**Group 1 — Documentation Output (sole deliverable):**

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md`
  - Comprehensive markdown document (~490 lines, ~43KB)
  - Organized into 9 major sections and 35 subsections
  - Covers: threading architecture, window creation/resize/destruction lifecycle, signal delivery mechanisms, state consistency guards, conflict resolution patterns, cascade cleanup, and timing-sensitive edge cases

**No other files are created, modified, or deleted.**

### 0.5.2 Implementation Approach

The implementation follows a read-analyze-document approach:

- **Establish foundational understanding** by reading `kitty/state.h` to understand the data structures (`GlobalState`, `OSWindow`, `Tab`, `Window`, `LiveResizeInfo`, `CloseRequest`) that hold all window state.
- **Trace the main event loop** by reading `kitty/child-monitor.c` to understand `process_global_state()` and its strict ordering: resizes → input parsing → rendering → close processing.
- **Map the resize pipeline** from GLFW callbacks (`kitty/glfw.c`) through debouncing (`process_pending_resizes()`) to viewport commit (`update_os_window_viewport()`) to Python relayout (`boss.on_window_resize()` → `TabManager.resize()` → `Tab.relayout()` → `Window.set_geometry()`) to PTY resize (`resize_pty()` → `ioctl(TIOCSWINSZ)` → SIGWINCH).
- **Map the destruction pipeline** from child death detection (I/O thread `POLLHUP`/read-returns-zero or `SIGCHLD` → `reap_children()`) through queue transfer (`remove_children()` → `remove_queue`) to main thread processing (`parse_input()` → `boss.on_child_death()` → `window.destroy()` → `tab.remove_window()` → cascade cleanup).
- **Identify all consistency guards** including the `destroyed` flag, `ignore_focus_changes` suppression, no-op viewport guard, idempotent `needs_removal`, `REMOVER` macro atomicity, weak references, and ID-lookup-failure-as-normal-flow.
- **Document conflict resolution scenarios** with concrete timelines showing what happens when creation-and-immediate-close, resize-during-death, and OS-window-close-during-live-resize occur.

### 0.5.3 Document Structure

The output document is organized as follows:

| Section | Title | Content Focus |
|---------|-------|--------------|
| 1 | Architectural Foundations | Threading model, GlobalState singleton, cross-thread primitives, main loop tick ordering |
| 2 | Window Lifecycle | Creation (5-step flow), resize (5-stage pipeline), destruction (3 entry points + cleanup) |
| 3 | Signal Delivery and Timing | signalfd/self-pipe mechanism, signal classification, SIGCHLD timing, SIGWINCH delivery |
| 4 | State Consistency Mechanisms | 8 specific mechanisms: debouncing, no-op guard, destroyed flag, focus suppression, mutex queues, weak refs, ID lookups, REMOVER macro |
| 5 | Window Disappears Before Reactions Complete | 4 concrete scenarios with step-by-step analysis |
| 6 | Resolving Conflicting Views | 7 resolution patterns with code-grounded explanations |
| 7 | Cascade Cleanup Pattern | Window→Tab→OS window cascade, multi-window cascade, stale reference guards |
| 8 | Timing-Sensitive Edge Cases | 5 specific edge cases with timing analysis |
| 9 | Summary | 8-point synthesis of all consistency principles |

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Output artifact:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable

**Source files analyzed to produce the document (read-only, not modified):**
- `kitty/state.h` — GlobalState, OSWindow, Tab, Window, LiveResizeInfo, CloseRequest definitions
- `kitty/state.c` — All state mutation functions: add/remove/destroy for windows, tabs, OS windows; REMOVER macro; detach/attach; WITH_* lookup macros
- `kitty/child-monitor.c` — Three-thread event loop, process_global_state(), parse_input(), process_pending_resizes(), process_pending_closes(), close_os_window(), mark_child_for_close(), resize_pty(), remove_children(), reap_children(), io_loop()
- `kitty/loop-utils.c` — Signal FD setup (signalfd/self-pipe), wakeup_loop(), read_signals()
- `kitty/glfw.c` — GLFW callbacks (framebuffer_size_callback, live_resize_callback, dpi_change_callback, window_close_callback), update_os_window_viewport()
- `kitty/boss.py` — Boss lifecycle methods: on_child_death(), on_window_resize(), on_os_window_closed(), mark_window_for_close(), suppress_focus_change_events(), _cleanup_tab_after_window_removal(), destroy()
- `kitty/window.py` — Window lifecycle: __init__(), set_geometry(), focus_changed(), destroy()
- `kitty/tabs.py` — Tab/TabManager lifecycle: new_window(), remove_window(), relayout(), resize(), _remove_tab(), destroy()
- `kitty/window_list.py` — WindowList/WindowGroup: add_window(), remove_window(), active group management
- `kitty/child.py` — Child process spawning: fork(), openpty(), PTY FD management
- `kitty_tests/layout.py` — Test references for WindowList.remove_window()
- `kitty_tests/screen.py` — Test references for screen resize

**Technical topics covered in the document:**
- Three-thread architecture (main, I/O, talk) and their coordination
- GlobalState singleton and the three-level hierarchy (OSWindow → Tab → Window)
- Mutex-protected queue transfers (add_queue, children, remove_queue, remove_notify)
- Signal delivery via signalfd (Linux) and self-pipe (macOS)
- Resize debouncing via LiveResizeInfo state machine
- Window creation, resize (5-stage pipeline), and destruction (3 entry points)
- SIGWINCH delivery timing and its dependence on debounce completion
- Guard flags: destroyed, ignore_focus_changes, no-op viewport guard
- Conflict resolution: authoritative sources, idempotent transitions, ID lookup failure as normal flow, snapshot-then-process, post-removal cleanup, ordered destruction
- Cascade cleanup: window → tab → OS window emptiness cascade
- Edge cases: create-and-immediate-close, resize-then-die, OS-close-during-resize, add_queue race window, EINTR handling in pty_resize

### 0.6.2 Explicitly Out of Scope

- **Modifying any existing repository files** — Explicitly prohibited by implementation rules
- **Adding code to the repository** — Only the documentation file is added
- **GPU rendering pipeline details** — The rendering subsystem (shaders, VAOs, textures) is not analyzed beyond its interface with window lifecycle
- **Remote control protocol** — The talk thread's peer messaging is mentioned only as part of the three-thread model; its protocol details are not analyzed
- **Kittens framework** — Not relevant to window lifecycle state consistency
- **Configuration parsing** — `kitty.conf` parsing is not analyzed
- **Font rendering and text shaping** — Not relevant to the state consistency question
- **Shell integration** — Not relevant to the core window lifecycle question
- **Performance optimization** — No performance analysis or benchmarking is included
- **macOS-specific Cocoa actions** — Mentioned only in the context of `process_global_state()` ordering; not analyzed in detail
- **Detach/attach window mechanics** — Referenced but not analyzed in depth as it is a secondary path not central to the rapid-lifecycle question

## 0.7 Rules for Feature Addition

The following rules govern this task, derived from the user's implementation rules (`SWE-AtlasQnA-Repo`):

- **Create a new markdown document** named `kitty_815df1e210e0.md` (derived from the source branch name `kitty_815df1e210e0`) that comprehensively answers the questions posed in the prompt.
- **Provide thinking / rationale** behind the answers. Every major claim in the document is accompanied by a "Rationale" paragraph or a "Key insight" callout explaining *why* kitty's design works the way it does, grounded in the code.
- **Do not make assumptions** — base answers on the code as the truth. All statements in the document reference specific source files, function names, and approximate line numbers. No external documentation or blog posts were used as sources; all conclusions are derived from reading the actual C and Python source code.
- **Do not modify any existing files** in the source repository. Verified via `git status` showing only the new `blitzy/` directory as untracked.
- **Do not add any other code** in the source repository besides the requested document. No scripts, no test files, no configuration changes.
- **Place the generated document** in the `blitzy/documentation` directory in the destination repo. The document is at `blitzy/documentation/kitty_815df1e210e0.md`.
- **Temporary cleanup**: The user mentioned that temporary scripts may be used but should be cleaned up. No temporary scripts were created; all analysis was conducted through direct source code reading using repository inspection tools.

## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were searched and analyzed to derive the conclusions documented in the output markdown:

**C Source Files (core state and event handling):**
- `kitty/state.h` — Data structure definitions (GlobalState, OSWindow, Tab, Window, LiveResizeInfo, CloseRequest)
- `kitty/state.c` — State mutation implementations (add/remove/destroy at all hierarchy levels, REMOVER macro, WITH_* lookup macros, detach/attach, update_os_window_references)
- `kitty/child-monitor.c` — Three-thread event loop (io_loop, parse_input, process_global_state, process_pending_resizes, process_pending_closes, close_os_window, mark_child_for_close, resize_pty, remove_children, add_children, reap_children, handle_signal, read_bytes, write_to_child)
- `kitty/loop-utils.c` — Signal delivery infrastructure (signalfd setup on Linux, self-pipe on macOS, wakeup_loop, read_signals, sig_handler)
- `kitty/glfw.c` — GLFW callback implementations (framebuffer_size_callback, live_resize_callback, dpi_change_callback, window_close_callback, update_os_window_viewport)
- `kitty/child.c` — Child process low-level operations (referenced for completeness; main logic is in child.py)

**Python Source Files (orchestration and high-level lifecycle):**
- `kitty/boss.py` — Boss class (on_child_death, on_window_resize, on_os_window_closed, mark_window_for_close, suppress_focus_change_events, _cleanup_tab_after_window_removal, close_os_window, destroy)
- `kitty/window.py` — Window class (__init__, set_geometry, focus_changed, destroy, destroyed flag, child_is_launched)
- `kitty/tabs.py` — Tab class (new_window, remove_window, relayout, destroy) and TabManager class (resize, _remove_tab)
- `kitty/window_list.py` — WindowList class (add_window, remove_window, active group management) and WindowGroup class
- `kitty/child.py` — Child class (fork, openpty, PTY master/slave creation, FD management, process group inspection)

**Test Files:**
- `kitty_tests/layout.py` — WindowList removal tests
- `kitty_tests/screen.py` — Screen resize tests
- `kitty_tests/__init__.py` — Test infrastructure

**Configuration Files:**
- `pyproject.toml` — Project configuration (version, build system)

**Folders Explored:**
- Repository root (`""`) — Top-level structure
- `kitty/` — Core application source
- `kitty_tests/` — Test suite

### 0.8.2 Tech Spec Sections Referenced

- Section 1.1 — Executive Summary (project overview and architecture context)
- Section 4.5 — Window and Tab Lifecycle (lifecycle flow documentation)
- Section 4.10 — Error Handling and Recovery Flows (error handling patterns)
- Section 4.11 — State Transition Diagrams (state machine documentation)
- Section 5.2 — Component Details (component architecture)

### 0.8.3 Attachments

No attachments were provided for this project. No Figma URLs or design assets are referenced.

