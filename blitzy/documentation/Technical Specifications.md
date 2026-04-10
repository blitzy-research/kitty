# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative documentation artifact** that comprehensively answers a series of interrelated questions about how the kitty terminal emulator maintains internal state consistency during rapid window lifecycle events — creation, resize, and destruction in quick succession.

- **Category**: Create new documentation
- **Documentation Type**: Technical deep-dive / architecture Q&A document
- **Target File**: `blitzy/documentation/kitty_815df1e210e0.md`

The user's documentation requirements translate to the following specific questions, each enhanced with technical precision:

- **Window Creation and Immediate Use**: When a new terminal window is created and a command is immediately spawned, how do child process launch, PTY allocation, layout computation, and screen buffer initialization interleave? What happens when SIGWINCH and resize events arrive before all initialization is complete?
- **Resize Event Propagation Under Rapid Succession**: When resize events and signals flow through the system during active window use, what mechanisms (debounce timers, the `LiveResizeInfo` struct, `input_delay` / `repaint_delay` parameters) throttle and coalesce these events to prevent inconsistency?
- **Window Destruction Before Completion of Pending State Changes**: If a window is closed (or its child process terminates) while resize or signal delivery is still in progress, how does kitty decide what state to keep and what to discard? Specifically, how do the `needs_removal` flag in `child-monitor.c`, the `remove_queue` / `remove_notify` arrays, and the `on_child_death` callback in `boss.py` coordinate cleanup?
- **Signal Delivery Timing**: How does the three-thread architecture (Main, I/O, Talk) in `child-monitor.c` affect the timing of `SIGCHLD` reaping, `SIGWINCH` delivery via PTY `ioctl(TIOCSWINSZ)`, and child death notification? What synchronization primitives (the `children_mutex`, the wakeup pipe, the `poll()` loop) gate these transitions?
- **Conflicting Views of Liveness**: Are there identifiable windows of time during which one thread considers a window alive while another has already marked it for removal? How does the system resolve such conflicting views — for example, when the I/O thread sets `needs_removal = true` on a child while the main thread is still parsing input for that child's screen?

### 0.1.2 Special Instructions and Constraints

- **Read-Only Repository Constraint**: The user explicitly states that the repository itself should remain unchanged. Temporary observation scripts may be created but must be cleaned up afterward. Per the implementation rules, no existing files in the source repository may be modified.
- **Implementation Rule — SWE-AtlasQnA-Repo**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the posed questions. Provide thinking and rationale behind answers. Base all answers on the code as the truth. Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **Evidence-Based Answers**: All conclusions must cite specific source files, line ranges, struct fields, and function names as evidence. No assumptions — the code is the truth.
- **Depth Expectation**: The user is interested in low-level timing, signal delivery mechanics, mutex-protected state transitions, and multi-thread coordination. The document should be thorough and aimed at an audience with systems programming expertise.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document **window creation and immediate use**, we will trace the path from `Boss.add_os_window()` → `Tab.new_window()` → `Window.__init__()` → `Child.fork()` → `ChildMonitor.add_child()` → `set_geometry()` → `resize_pty()` → `mark_terminal_ready()`, citing `kitty/boss.py`, `kitty/window.py`, `kitty/child.py`, and `kitty/child-monitor.c`.
- To document **resize event propagation**, we will analyze the `LiveResizeInfo` struct in `kitty/state.h`, the `framebuffer_size_callback()` / `live_resize_callback()` in `kitty/glfw.c`, the `process_pending_resizes()` function in `kitty/child-monitor.c`, the debounce logic governed by `resize_debounce_time.on_end` / `on_pause`, and the viewport dirty flag propagation.
- To document **window destruction under pending changes**, we will trace the `mark_for_close()` → `needs_removal` flag → `remove_children()` → `remove_queue` → `parse_input()` → `death_notify` → `Boss.on_child_death()` → `Window.destroy()` pipeline across `kitty/child-monitor.c`, `kitty/boss.py`, and `kitty/window.py`.
- To document **signal delivery timing**, we will detail the `SIGCHLD` → `handle_signal()` → `reap_children()` → `mark_child_for_removal()` path in the I/O thread, and how the main thread observes these changes through `children_mutex`-protected state transitions.
- To document **conflicting liveness views**, we will analyze the snapshot-then-release pattern in `parse_input()` (lines 456–483 of `child-monitor.c`) where the main thread copies `children[]` into `scratch[]` under the lock, increments reference counts, releases the lock, and then skips entries with `needs_removal == true`.

### 0.1.4 Inferred Documentation Needs

Based on repository analysis, the following implicit documentation needs have been identified:

- **Thread synchronization diagram**: The three-thread model (Main, I/O, Talk) and their mutex interactions (`children_mutex`, `talk_mutex`, screen buffer locks) require a visual diagram for clarity.
- **State transition diagram for window liveness**: Mapping the lifecycle of the `Child` struct's `needs_removal` flag from `false` through its various setting paths (POLLHUP, POLLNVAL, SIGCHLD reap, explicit `mark_for_close`) to the eventual `cleanup_child()` call.
- **Timing diagram for resize debounce**: Showing the relationship between `framebuffer_size_callback` events, the `live_resize.last_resize_event_at` timestamp, the debounce timer, and the eventual `update_os_window_viewport()` call.
- **Race-window analysis**: Documenting the precise windows of time during which inconsistent views can exist and how the codebase tolerates them through defensive checks (`if window is None: return` patterns throughout `boss.py`).

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx/reStructuredText documentation system** with comprehensive user-facing coverage but no existing deep-dive documentation on internal state consistency, thread synchronization, or window lifecycle race conditions.

- **Documentation framework**: Sphinx, configured in `docs/conf.py`
- **Documentation generator configuration**: `docs/conf.py` (Sphinx configuration), `docs/Makefile` (build driver)
- **Documentation dependencies**: `docs/requirements.txt` (pins `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`)
- **Diagram tools detected**: Mermaid (used in tech spec), reStructuredText directives (used in Sphinx docs)
- **API documentation tools in use**: None dedicated (no JSDoc, Sphinx autodoc, or Godoc configuration for the core C/Python sources)
- **Documentation hosting**: Published via `make docs` and `publish.py` to the kitty website

Existing documentation files examined for relevance to window lifecycle and state management:

| File | Content | Relevance |
|------|---------|-----------|
| `docs/overview.rst` | High-level project introduction | Low — no internals coverage |
| `docs/layouts.rst` | Layout algorithms (Tall, Fat, Grid, etc.) | Medium — covers layout computation but not lifecycle |
| `docs/conf.rst` | Configuration file reference | Low — documents `resize_debounce_time` option existence only |
| `docs/faq.rst` | Troubleshooting and FAQ | Low — no window state internals |
| `docs/performance.rst` | Performance architecture overview | Medium — mentions threading model at a high level |
| `docs/remote-control.rst` | Remote control protocol | Low — documents RC commands, not state management |

No existing documentation covers the specific topics requested: internal state consistency during rapid window lifecycle transitions, signal delivery timing, or multi-thread coordination during window creation/destruction.

The destination directory `blitzy/documentation/` does not yet exist and must be created.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were examined to extract evidence for the documentation:

**Primary sources (directly answering the user's questions):**

| File | Lines Examined | Key Content |
|------|---------------|-------------|
| `kitty/child-monitor.c` | 1–1600 | Three-thread architecture, `parse_input()`, `remove_children()`, `io_loop()`, `reap_children()`, `process_pending_resizes()`, `process_pending_closes()`, signal handling, `children_mutex` synchronization |
| `kitty/boss.py` | 340–1860 | `window_id_map` (WeakValueDictionary), `on_child_death()`, `mark_window_for_close()`, `on_window_resize()`, `on_os_window_closed()`, `_cleanup_tab_after_window_removal()` |
| `kitty/window.py` | 530–1600 | `Window.__init__()`, `set_geometry()`, `destroy()`, `close()`, resize watcher invocation, `child_is_launched` flag, `last_reported_pty_size` tracking |
| `kitty/state.h` | 1–402 | `GlobalState`, `OSWindow`, `Window`, `Tab`, `LiveResizeInfo`, `CloseRequest` enum, `Child` struct layout |
| `kitty/state.c` | 1–660 | `add_os_window()`, `add_window()`, `remove_window_inner()`, `remove_os_window()`, `destroy_window()`, `mark_os_window_for_close()`, viewport functions |
| `kitty/glfw.c` | 130–360 | `update_os_window_viewport()`, `live_resize_callback()`, `framebuffer_size_callback()`, `change_live_resize_state()`, `dpi_change_callback()` |
| `kitty/child.py` | 260–370 | `Child.fork()`, PTY allocation, `mark_terminal_ready()`, signal passing via `handled_signals` |
| `kitty/window_list.py` | 1–443 | `WindowList`, `WindowGroup`, `remove_window()`, `add_window()`, active group management |
| `kitty/tabs.py` | 560–970 | `Tab.remove_window()`, `TabManager.resize()`, `TabManager._remove_tab()`, `Tab.destroy()` |

**Supporting sources (context and type definitions):**

| File | Key Content |
|------|-------------|
| `kitty/cleanup.c` | Cleanup handler registration |
| `kitty/constants.py` | `handled_signals` set, signal registration |
| `kitty/fast_data_types.pyi` | Python type stubs for C extensions (`ChildMonitor`, `mark_for_close`, `resize_pty`) |

### 0.2.3 Web Search Research Conducted

No external web research was required for this task. All answers are derived directly from the source code, which the user explicitly mandated as the single source of truth ("Do not make assumptions, base your answers on the code as the truth"). The kitty codebase is self-contained and all relevant mechanisms are fully visible in the examined source files.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules contain the logic that must be documented to answer the user's questions comprehensively. Each module's relevant public interfaces, internal mechanisms, and documentation status are cataloged below.

**Module: `kitty/child-monitor.c` — Core Event Loop and Child Lifecycle**

- Public APIs: `ChildMonitor.add_child`, `ChildMonitor.mark_for_close`, `ChildMonitor.resize_pty`, `ChildMonitor.start`, `ChildMonitor.wakeup`, `ChildMonitor.shutdown_monitor`
- Internal mechanisms:
  - `io_loop()` — I/O thread main loop with `poll()` multiplexing
  - `parse_input()` — Main thread input parsing with snapshot-under-lock pattern
  - `remove_children()` — I/O thread child cleanup and queue transfer
  - `reap_children()` — SIGCHLD-driven child reaping via `waitpid()`
  - `process_pending_resizes()` — Resize debounce and viewport update
  - `process_pending_closes()` — Close request processing and OS window teardown
  - `handle_signal()` — Signal classification (SIGINT/SIGTERM/SIGHUP → kill, SIGCHLD → reap, SIGUSR1 → config reload)
- Current documentation: **None** — No existing deep-dive docs on thread coordination or state transitions
- Documentation needed: Complete analysis of the three-thread model, mutex-protected queues, signal delivery paths, and the timing relationships between I/O and main thread state views

**Module: `kitty/boss.py` — Boss Controller (Window Lifecycle Orchestration)**

- Public APIs: `on_child_death()`, `mark_window_for_close()`, `on_window_resize()`, `on_os_window_closed()`, `add_os_window()`, `close_window()`, `close_tab()`, `quit()`
- Internal mechanisms:
  - `window_id_map` — `WeakValueDictionary` that serves as the global window registry; entries vanish when windows are garbage-collected
  - `os_window_map` — Maps OS window IDs to `TabManager` instances
  - `_cleanup_tab_after_window_removal()` — Cascading cleanup: empty tab → remove from tab manager → empty tab manager → mark OS window for close
  - `suppress_focus_change_events()` — Context manager to batch focus changes during cleanup
- Current documentation: **None** for internal state consistency
- Documentation needed: Detailed walkthrough of the death notification callback, the cascading cleanup logic, and defensive null-check patterns

**Module: `kitty/window.py` — Window State and Geometry**

- Public APIs: `set_geometry()`, `destroy()`, `close()`, `refresh()`
- Internal mechanisms:
  - `set_geometry()` — Triggers screen resize, PTY resize via `resize_pty()`, SIGWINCH delivery, and watcher callbacks
  - `destroy()` — Calls `on_close` watchers, sets `destroyed = True`, cancels IME, breaks screen reference cycles
  - `child_is_launched` flag — Guards against premature PTY resize before child is ready
  - `last_reported_pty_size` — Deduplication tuple preventing redundant SIGWINCH delivery
- Current documentation: **None** for the resize/destroy interaction
- Documentation needed: The precise sequence of operations in `set_geometry()` and how it guards against operating on a destroyed window

**Module: `kitty/state.h` / `kitty/state.c` — Native Global State**

- Key data structures:
  - `GlobalState` — Singleton with `has_pending_resizes`, `has_pending_closes`, `quit_request`, `os_windows[]` array
  - `OSWindow` — Per-OS-window state including `live_resize` (`LiveResizeInfo`), `close_request`, `viewport_size_dirty`, `viewport_resized_at`
  - `LiveResizeInfo` — Debounce state: `last_resize_event_at`, `in_progress`, `from_os_notification`, `os_says_resize_complete`, `width`, `height`, `num_of_resize_events`
  - `CloseRequest` enum — `NO_CLOSE_REQUESTED`, `CONFIRMABLE_CLOSE_REQUESTED`, `CLOSE_BEING_CONFIRMED`, `IMPERATIVE_CLOSE_REQUESTED`
  - `Child` struct — Per-child tracking with `screen`, `needs_removal`, `fd`, `id`, `pid`
- Current documentation: **None** for state consistency semantics
- Documentation needed: Complete catalog of the state fields that participate in lifecycle transitions

**Module: `kitty/glfw.c` — Platform Resize Callbacks**

- Key functions: `update_os_window_viewport()`, `live_resize_callback()`, `framebuffer_size_callback()`, `dpi_change_callback()`, `change_live_resize_state()`
- Current documentation: **None** for the resize debounce interaction
- Documentation needed: The callback → debounce → viewport update → boss notification pipeline

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps must be addressed:

- **No existing documentation** covers the internal state consistency model during rapid window lifecycle transitions
- **No existing documentation** explains how the three-thread architecture coordinates signal delivery with state mutation
- **No existing documentation** describes the resize debounce pipeline from GLFW callback through to PTY resize and SIGWINCH delivery
- **No existing documentation** catalogs the defensive patterns (null checks, `needs_removal` guards, `WeakValueDictionary` auto-cleanup) that resolve conflicting liveness views
- **No existing documentation** provides timing diagrams or race-window analysis for concurrent state access

All of these gaps will be addressed by the new `blitzy/documentation/kitty_815df1e210e0.md` document.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The new document `blitzy/documentation/kitty_815df1e210e0.md` will be structured as a self-contained, comprehensive Q&A artifact that traces every relevant code path with citations.

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction (question restatement and scope)
        ├── Architectural Context
        │   ├── Three-Thread Model (Main, I/O, Talk)
        │   ├── Key Data Structures (GlobalState, OSWindow, Child, LiveResizeInfo)
        │   └── Synchronization Primitives (children_mutex, wakeup pipes, poll())
        ├── Window Creation and Immediate Use
        │   ├── Creation Path (Boss → Tab → Window → Child → ChildMonitor)
        │   ├── PTY Allocation and Signal Setup
        │   ├── First Geometry Assignment and SIGWINCH
        │   └── The child_is_launched Guard
        ├── Resize Event Propagation
        │   ├── GLFW Callback to LiveResizeInfo Recording
        │   ├── Debounce Pipeline (on_pause vs on_end timers)
        │   ├── Viewport Update and Boss Notification
        │   ├── Tab Relayout → Window set_geometry → Screen Resize → PTY ioctl
        │   └── SIGWINCH Deduplication via last_reported_pty_size
        ├── Window Destruction Under Pending Changes
        │   ├── Marking for Close (mark_for_close → needs_removal)
        │   ├── I/O Thread Cleanup (remove_children → close fd → hangup)
        │   ├── Queue Transfer (remove_queue → remove_notify)
        │   ├── Main Thread Death Processing (parse_input → death_notify → on_child_death)
        │   ├── Cascading Cleanup (tab → tab_manager → OS window)
        │   └── What State Is Kept vs Discarded
        ├── Signal Delivery and Timing
        │   ├── SIGCHLD Path (handle_signal → reap_children → mark_child_for_removal)
        │   ├── SIGWINCH Path (resize_pty → ioctl(TIOCSWINSZ))
        │   ├── Input Delay and Main Loop Wakeup Coalescing
        │   └── Race Between I/O Thread Reap and Main Thread Parse
        ├── Conflicting Views of Liveness
        │   ├── The Snapshot-Then-Release Pattern in parse_input()
        │   ├── needs_removal Skip Guard
        │   ├── WeakValueDictionary Auto-Eviction in boss.py
        │   ├── Defensive Null Checks Throughout boss.py
        │   └── Identified Race Windows and Their Resolution
        └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the thread architecture and synchronization model from `kitty/child-monitor.c` (lines 50–100 for data structures, 451–538 for `parse_input()`, 1306–1426 for child cleanup and signal handling, 1480–1578 for `io_loop()`)
- Extract the window lifecycle state machine from `kitty/boss.py` (`on_child_death()` at line 881, `on_os_window_closed()` at line 1775, `on_window_resize()` at line 1206)
- Extract the resize debounce pipeline from `kitty/glfw.c` (lines 130–360) and `kitty/child-monitor.c` (lines 1043–1080)
- Extract the state structures from `kitty/state.h` (lines 119–282) and `kitty/state.c` (lines 200–496)
- Extract defensive patterns from `kitty/window_list.py` (lines 373–395 for `remove_window()`) and `kitty/boss.py` (throughout — `if window is None: return` patterns)

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for thread architecture, state transitions, and timing sequences
- Code citations using format: `Source: kitty/child-monitor.c:451–538`
- Short inline code snippets (2–3 lines max) for critical logic points
- Tables for struct field catalogs and state transition summaries

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the document:

- **Thread Architecture Diagram**: Flowchart showing Main, I/O, and Talk threads with their respective responsibilities, mutex boundaries, and wakeup communication channels
- **Window Creation Sequence Diagram**: Sequence diagram tracing a new window from Boss through Child.fork() to ChildMonitor.add_child() and the first set_geometry() call
- **Resize Debounce State Machine**: State diagram showing the `LiveResizeInfo` transitions from idle → `in_progress` → debounce wait → `update_os_window_viewport()` → idle
- **Window Destruction Sequence Diagram**: Sequence diagram showing the I/O thread → `remove_queue` → main thread → `death_notify` → `Boss.on_child_death()` → cascading cleanup path
- **Liveness Conflict Timeline**: Timeline diagram showing the window of time between I/O thread marking `needs_removal` and main thread observing and acting on it

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window.py`, `kitty/state.h`, `kitty/state.c`, `kitty/glfw.c`, `kitty/child.py`, `kitty/window_list.py`, `kitty/tabs.py` | Comprehensive Q&A document answering how kitty maintains internal state consistency during rapid window creation, resize, and destruction. Includes thread architecture analysis, resize debounce pipeline, destruction cleanup cascade, signal delivery timing, and liveness conflict resolution. Contains Mermaid diagrams for thread model, creation sequence, resize state machine, destruction sequence, and liveness timeline. All claims cited to specific source files and line ranges. |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Deep-Dive / Architecture Q&A
Source Code:
  - kitty/child-monitor.c (primary: three-thread model, signal handling, parse_input, io_loop)
  - kitty/boss.py (primary: on_child_death, window_id_map, on_window_resize, on_os_window_closed)
  - kitty/window.py (primary: set_geometry, destroy, child_is_launched guard)
  - kitty/state.h (primary: GlobalState, OSWindow, LiveResizeInfo, Child, CloseRequest)
  - kitty/state.c (primary: add_window, remove_window_inner, remove_os_window, mark_os_window_for_close)
  - kitty/glfw.c (primary: update_os_window_viewport, live_resize_callback, framebuffer_size_callback)
  - kitty/child.py (supporting: fork(), mark_terminal_ready(), PTY allocation)
  - kitty/window_list.py (supporting: WindowList.remove_window, active group management)
  - kitty/tabs.py (supporting: Tab.remove_window, TabManager.resize, Tab.destroy)
Sections:
  - Introduction (question restatement and methodology)
  - Architectural Context (three-thread model, key data structures, synchronization primitives)
  - Window Creation and Immediate Use (creation path, PTY allocation, first geometry, child_is_launched)
  - Resize Event Propagation (GLFW callbacks, debounce pipeline, viewport update, SIGWINCH dedup)
  - Window Destruction Under Pending Changes (mark_for_close, I/O cleanup, queue transfer, cascading cleanup)
  - Signal Delivery and Timing (SIGCHLD path, SIGWINCH path, input delay, race analysis)
  - Conflicting Views of Liveness (snapshot pattern, needs_removal guard, WeakValueDictionary, null checks)
  - Summary (consolidated answers)
Diagrams:
  - Thread architecture flowchart (Main, I/O, Talk threads with mutex boundaries)
  - Window creation sequence diagram
  - Resize debounce state machine
  - Window destruction sequence diagram
  - Liveness conflict timeline
Key Citations:
  - kitty/child-monitor.c:50-100, 451-538, 540-564, 1043-1080, 1082-1095, 1306-1426, 1480-1578
  - kitty/boss.py:340-376, 854-919, 1181-1213, 1775-1787
  - kitty/window.py:544-580, 850-883, 1560-1572
  - kitty/state.h:65-72, 119-282
  - kitty/state.c:200-496, 578-582
  - kitty/glfw.c:130-170, 300-360
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration changes are needed. The `blitzy/documentation/` directory is a standalone output directory for Q&A artifacts per the `SWE-AtlasQnA-Repo` implementation rule. It does not integrate with kitty's Sphinx documentation system.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes**: The new document is self-contained
- **No navigation links**: The document does not integrate into kitty's existing Sphinx docs tree
- **No table of contents updates**: Not applicable for the `blitzy/documentation/` output directory
- **No index/glossary updates**: The document will define all terms inline

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tooling dependencies are required for this task. The output is a standalone Markdown file placed in the `blitzy/documentation/` directory. No documentation site generators, diagram renderers, or API documentation tools need to be installed or configured.

The project's own documentation toolchain (for reference only — not used in this task):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | pinned in `docs/requirements.txt` | Kitty's existing Sphinx documentation generator (not used for this task) |
| pip | furo | pinned in `docs/requirements.txt` | Sphinx theme for kitty docs site (not used for this task) |
| pip | sphinx-copybutton | pinned in `docs/requirements.txt` | Copy button for code blocks (not used for this task) |
| pip | sphinxext-opengraph | pinned in `docs/requirements.txt` | OpenGraph metadata for social sharing (not used for this task) |
| pip | sphinx-inline-tabs | pinned in `docs/requirements.txt` | Inline tab support in docs (not used for this task) |
| pip | sphinx-autobuild | pinned in `docs/requirements.txt` | Live preview during docs development (not used for this task) |

The project's core runtime dependencies (for context — defines the codebase being documented):

| Registry | Package Name | Version Constraint | Purpose |
|----------|--------------|-------------------|---------|
| pip | Python | ≥ 3.8 (from `pyproject.toml`) | Runtime interpreter for kitty's Python layer |
| go | Go | 1.22 (from `go.mod`) | Go tools and CLI binary compilation |
| system | GLFW | 3.4 (vendored fork in `glfw/`) | Platform windowing and OpenGL context |
| system | FreeType | system-provided | Font rasterization |
| system | HarfBuzz | ≥ 1.5 | Text shaping |
| system | OpenGL | ≥ 3.3 | GPU rendering |

### 0.6.2 Documentation Reference Updates

Not applicable. The new document is a standalone artifact and does not require updating any links in existing documentation files. Per the implementation rule, no existing files in the source repository may be modified.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The document must achieve complete coverage of the user's five interrelated questions:

| Question Domain | Source Modules | Coverage Target | Key Evidence Required |
|----------------|---------------|----------------|----------------------|
| Window creation + immediate use | `kitty/boss.py`, `kitty/child.py`, `kitty/window.py`, `kitty/child-monitor.c` | 100% — full creation path traced | `add_os_window()`, `Child.fork()`, `add_child()`, `set_geometry()`, `mark_terminal_ready()` |
| Resize event propagation | `kitty/glfw.c`, `kitty/child-monitor.c`, `kitty/state.h`, `kitty/window.py` | 100% — full debounce pipeline | `framebuffer_size_callback()`, `LiveResizeInfo`, `process_pending_resizes()`, `resize_pty()` |
| Window destruction under pending changes | `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window.py`, `kitty/tabs.py` | 100% — full destruction cascade | `mark_for_close()`, `remove_children()`, `on_child_death()`, `destroy()`, `_cleanup_tab_after_window_removal()` |
| Signal delivery timing | `kitty/child-monitor.c`, `kitty/child.py` | 100% — SIGCHLD and SIGWINCH paths | `handle_signal()`, `reap_children()`, `ioctl(TIOCSWINSZ)`, `input_delay` coalescing |
| Conflicting views of liveness | `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window_list.py` | 100% — race window identification | `parse_input()` snapshot pattern, `needs_removal` skip, `WeakValueDictionary`, null guards |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every answer must trace the relevant code path from entry to exit, citing file paths and line ranges
- Every state transition must be documented with the specific field or flag that triggers it
- Every synchronization point must identify the mutex or lock that protects it
- Every defensive check pattern must be cataloged with its purpose explained

**Accuracy validation:**

- All code citations must reference the actual codebase examined (branch `kitty_815df1e210e0`)
- All struct field names, function names, and variable names must be exact matches to the source code
- All claimed ordering constraints (e.g., "the I/O thread sets `needs_removal` before the main thread reads it") must be supported by the mutex acquisition patterns visible in the code
- No speculative claims — every assertion must point to a specific code artifact

**Clarity standards:**

- The document is aimed at systems programmers familiar with POSIX threading, signals, and PTYs
- Complex multi-thread interactions must be supplemented with Mermaid diagrams
- Each major section must begin with a summary answer before diving into the detailed trace
- The final Summary section must directly answer each of the user's original questions in concise form

**Maintainability:**

- All source citations include file path and line range for traceability
- The document is tied to branch `kitty_815df1e210e0` — future code changes may invalidate specific line numbers

### 0.7.3 Example and Diagram Requirements

- Minimum **5 Mermaid diagrams** as specified in Section 0.4.3
- Short inline code snippets (2–3 lines) at critical decision points — for example, showing the `needs_removal` check in `parse_input()` or the `ioctl(TIOCSWINSZ)` retry loop
- Tables summarizing struct fields, state transitions, and signal routing
- No screenshot requirements (this is a code-analysis document, not a UI document)

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive Q&A document answering the user's questions about internal state consistency during rapid window lifecycle transitions

**Code modules analyzed for documentation content (read-only — no modifications):**

- `kitty/child-monitor.c` — Three-thread architecture, signal handling, `parse_input()`, `io_loop()`, `remove_children()`, `reap_children()`, `process_pending_resizes()`, `process_pending_closes()`, `mark_for_close()`, `resize_pty()`, `cleanup_child()`
- `kitty/boss.py` — Boss controller: `on_child_death()`, `mark_window_for_close()`, `on_window_resize()`, `on_os_window_closed()`, `add_os_window()`, `_cleanup_tab_after_window_removal()`, `window_id_map`, `os_window_map`
- `kitty/window.py` — Window class: `__init__()`, `set_geometry()`, `destroy()`, `close()`, `child_is_launched`, `last_reported_pty_size`, watchers
- `kitty/state.h` — Native state structures: `GlobalState`, `OSWindow`, `Window`, `Tab`, `Child`, `LiveResizeInfo`, `CloseRequest`
- `kitty/state.c` — Native state operations: `add_os_window()`, `add_window()`, `remove_window_inner()`, `remove_os_window()`, `destroy_window()`, `destroy_tab()`, `mark_os_window_for_close()`
- `kitty/glfw.c` — Platform callbacks: `update_os_window_viewport()`, `live_resize_callback()`, `framebuffer_size_callback()`, `dpi_change_callback()`, `change_live_resize_state()`
- `kitty/child.py` — Child process: `fork()`, `mark_terminal_ready()`, PTY allocation, environment setup
- `kitty/window_list.py` — Window tracking: `WindowList`, `WindowGroup`, `add_window()`, `remove_window()`
- `kitty/tabs.py` — Tab management: `Tab.remove_window()`, `Tab.destroy()`, `TabManager.resize()`, `TabManager._remove_tab()`

**Topics covered in the document:**

- Thread architecture and synchronization model (mutexes, wakeup pipes, poll multiplexing)
- Window creation path from Boss through Child to ChildMonitor
- Resize event debounce pipeline from GLFW callback through viewport update to SIGWINCH
- Window destruction cascade from `mark_for_close` through I/O cleanup to Python-level death notification
- Signal delivery timing for SIGCHLD (child death) and SIGWINCH (terminal resize)
- Conflicting liveness views and the defensive patterns that resolve them

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No existing repository files will be modified (per user instruction and SWE-AtlasQnA-Repo rule)
- **Test file modifications**: No test files will be created or modified
- **Feature additions or code refactoring**: This is a documentation-only task
- **GPU rendering pipeline internals**: Shader stages, glyph caching, and OpenGL operations are not relevant to the state consistency question
- **Font subsystem**: Font discovery, rasterization, and shaping do not participate in the window lifecycle state machine
- **Remote control protocol**: The RC command system is tangential; it can trigger window operations but does not define the state consistency model
- **Kittens framework**: Kitten execution and error isolation are not relevant to the core window lifecycle question
- **Shell integration**: Shell startup environment modification does not affect internal state consistency
- **Configuration system**: While `resize_debounce_time`, `input_delay`, and `repaint_delay` are config options that parameterize the debounce pipeline, the configuration loading/parsing system itself is out of scope
- **Go tools layer**: The Go CLI tooling does not participate in window lifecycle management
- **macOS-specific Cocoa integration**: While briefly mentioned where relevant (e.g., `cocoa_out_of_sequence_render`), deep macOS-specific analysis is not the focus
- **Deployment configuration changes**: Not applicable
- **Kitty's Sphinx documentation site**: The new document lives in `blitzy/documentation/`, not `docs/`

## 0.9 Rules for Documentation

The following rules govern the creation of the documentation artifact, derived from the user's explicit instructions and the project's implementation rules:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` (matching the source branch name) and place it in the `blitzy/documentation` directory in the destination repo.
- **No Repository Modification**: Do not modify any existing files in the source repository. The repository must remain unchanged. If temporary scripts are created for observation, they must be cleaned up afterward.
- **Code as Truth**: Do not make assumptions. Base all answers on the code as the single source of truth. Every claim must be traceable to specific files and line ranges.
- **Provide Rationale**: Provide thinking and rationale behind the answers, not just conclusions. Explain *why* the code behaves as it does, not just *what* it does.
- **Comprehensive Answers**: The document must comprehensively answer all questions posed in the prompt — window creation timing, resize propagation, destruction under pending changes, signal delivery mechanics, and conflicting liveness views.
- **Cleanup After Temporary Work**: Any temporary scripts used for observation or analysis must be removed. The only persistent artifact is the final markdown document.
- **Evidence-Based Citations**: Every section must reference the source files examined, with file paths and line ranges for traceability.
- **Mermaid Diagrams**: Include Mermaid diagrams for complex multi-thread and multi-step interactions to aid reader comprehension.
- **Systems-Level Depth**: The user is asking about timing, signal delivery, internal bookkeeping, and conflicting views of liveness. The document must operate at the systems programming level, discussing mutexes, thread coordination, signal handlers, and file descriptor management.

## 0.10 References

### 0.10.1 Codebase Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Root-level exploration:**

- Repository root (`""`) — full folder contents retrieved; identified `kitty/`, `docs/`, `glfw/`, `kitty_tests/`, and all top-level files
- `pyproject.toml` — confirmed Python ≥ 3.8 requirement
- Git branch: `kitty_815df1e210e0` (confirmed via `git branch --show-current`)

**Core application sources (deep read):**

| File Path | Lines Read | Purpose |
|-----------|-----------|---------|
| `kitty/child-monitor.c` | 50–110, 451–600, 760–900, 1043–1110, 1140–1260, 1290–1600 | Three-thread architecture, parse_input, mark_for_close, resize_pty, process_pending_resizes, process_pending_closes, io_loop, reap_children, signal handling, child cleanup |
| `kitty/boss.py` | 1–120, 340–410, 854–970, 1181–1240, 1651–1720, 1724–1860 | Imports, initialization, child death handling, window closure, resize handling, OS window lifecycle, quit logic |
| `kitty/window.py` | 530–580, 840–930, 1550–1600 | Window constructor, set_geometry with resize logic, destroy method |
| `kitty/state.h` | 1–402 (full file) | All native data structures: GlobalState, OSWindow, Window, Tab, LiveResizeInfo, CloseRequest, Child |
| `kitty/state.c` | 1–80, 200–660 | Global state singleton, add/remove functions, window/tab/OS-window lifecycle, mark_os_window_for_close, viewport helpers |
| `kitty/glfw.c` | 130–170, 295–360 | update_os_window_viewport, live_resize_callback, framebuffer_size_callback, dpi_change_callback, change_live_resize_state |
| `kitty/child.py` | 1–60, 260–370 | Child class, fork/PTY allocation, mark_terminal_ready, environment setup |
| `kitty/window_list.py` | 1–443 (full file) | WindowList, WindowGroup, add_window, remove_window, active group tracking |
| `kitty/tabs.py` | 560–610, 840–970 | Tab.remove_window, Tab.destroy, TabManager.resize, TabManager._remove_tab |

**Grep-based searches conducted:**

| Search Pattern | Files Searched | Purpose |
|---------------|---------------|---------|
| `def.*close\|def.*remove\|def.*destroy\|def.*resize\|mark_for_close\|child_exited` | `kitty/boss.py` | Locate all window lifecycle methods |
| `def.*close\|def.*remove\|def.*destroy\|def.*resize\|destroy\|on_close\|on_resize` | `kitty/window.py` | Locate all window state transition methods |
| `def.*close\|def.*remove\|def.*destroy\|def.*resize\|mark_for_close` | `kitty/tabs.py` | Locate tab lifecycle methods |
| `live_resize\|has_pending_resizes\|pending_resizes\|resize_debounce\|viewport_size_dirty` | `kitty/*.c` | Locate all resize-related native code |
| `mark_for_close\|child_exited\|SIGCHLD\|resize_pty\|remove_window\|reap\|signal_received` | `kitty/child-monitor.c` | Locate all child lifecycle and signal handling code |
| `SIGWINCH` | `kitty/**/*.c`, `kitty/**/*.py`, `kitty/**/*.h` | Locate all SIGWINCH references |
| `mark_terminal_ready\|SIGWINCH\|resize\|pty` | `kitty/child.py` | Locate PTY and terminal-ready logic |

**Documentation infrastructure:**

| File/Folder | Purpose |
|-------------|---------|
| `docs/` folder | Examined folder contents for existing internal state documentation (none found) |
| `docs/conf.py` | Sphinx documentation configuration |
| `docs/requirements.txt` | Documentation build dependencies |

**Tech spec sections retrieved:**

| Section | Purpose |
|---------|---------|
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Application lifecycle context |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | I/O pipeline and VT parser context |
| 4.5 WINDOW AND TAB LIFECYCLE | Window state transitions, child launch flow |
| 4.10 ERROR HANDLING AND RECOVERY FLOWS | Error isolation patterns |
| 4.11 STATE TRANSITION DIAGRAMS | Launch type dispatch and session states |
| 5.1 HIGH-LEVEL ARCHITECTURE | Six-layer architecture overview |
| 5.2 COMPONENT DETAILS | Boss Controller, Child Monitor, VT Parser, Layout Engine, GLFW Platform Layer details |

### 0.10.2 Attachments

No attachments were provided by the user for this project.

### 0.10.3 Figma Screens

No Figma screens or URLs were provided for this project.

