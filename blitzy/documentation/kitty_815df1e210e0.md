# Kitty Internal State Consistency During Rapid Window Lifecycle Events

## A Comprehensive Technical Analysis

This document answers the question: *How does kitty keep its internal state consistent when terminal windows appear, resize, and disappear in quick succession?* Every claim is grounded in the actual source code with specific file names, function names, and approximate line numbers. No external documentation or assumptions are used — the code is the truth.

---

## Table of Contents

1. [Architectural Foundations](#1-architectural-foundations)
2. [Window Lifecycle](#2-window-lifecycle)
3. [Signal Delivery and Timing](#3-signal-delivery-and-timing)
4. [State Consistency Mechanisms](#4-state-consistency-mechanisms)
5. [Window Disappears Before Reactions Complete](#5-window-disappears-before-reactions-complete)
6. [Resolving Conflicting Views](#6-resolving-conflicting-views)
7. [Cascade Cleanup Pattern](#7-cascade-cleanup-pattern)
8. [Timing-Sensitive Edge Cases](#8-timing-sensitive-edge-cases)
9. [Summary](#9-summary)

---

## 1. Architectural Foundations

### 1.1 Three-Thread Model

Kitty operates on exactly three threads, each with a distinct responsibility:

1. **Main thread** — Runs the GLFW event loop and calls `process_global_state()` on each tick (`kitty/child-monitor.c` ~line 1224). This thread owns all GUI state, all Python code execution, and all OpenGL rendering. It is the *only* thread that mutates the `GlobalState` hierarchy directly.

2. **I/O thread** — Runs `io_loop()` (`kitty/child-monitor.c` ~line 1480). This thread `poll()`s on child PTY file descriptors and the signal FD. It reads bytes from children, writes buffered output to children, detects child death (via `POLLHUP` or read-returns-zero), and reaps zombie processes via `waitpid()`. It communicates with the main thread exclusively through mutex-protected queues.

3. **Talk thread** — Runs `talk_loop()` (declared at `kitty/child-monitor.c` ~line 230). This thread handles peer communication for kitty's remote control protocol. It is started on-demand when the first peer connects (`inject_peer()` ~line 248-278). It communicates with the main thread through the `talk_lock` mutex and the `messages[]` array.

> **Rationale**: By confining all state mutations and Python callbacks to the main thread, kitty eliminates an entire class of concurrency bugs. The I/O thread and talk thread only set flags and enqueue data — they never directly modify window or tab state. This design means the main thread can process state changes in a well-defined order without races.

### 1.2 GlobalState Singleton

The `GlobalState` struct is defined in `kitty/state.h` ~line 259-280. A single instance is declared at `kitty/state.c` line 12:

```c
GlobalState global_state = {{0}};
```

Key fields include:
- `os_windows` — A dynamically sized contiguous C array of `OSWindow` structs
- `num_os_windows`, `capacity` — Count and allocation capacity for the array
- `boss` — A `PyObject*` pointer to the Python `Boss` singleton
- `has_pending_resizes` — Flag checked each tick to trigger resize processing
- `has_pending_closes` — Flag checked each tick to trigger close processing
- `os_window_id_counter`, `tab_id_counter`, `window_id_counter` — Monotonically increasing ID generators

### 1.3 Three-Level Hierarchy

All window state is organized in a strict three-level hierarchy of contiguous C arrays (not linked lists):

```
GlobalState
  └── OSWindow[]          (os_windows, indexed 0..num_os_windows-1)
        └── Tab[]          (tabs, indexed 0..num_tabs-1)
              └── Window[]  (windows, indexed 0..num_windows-1)
```

Each level is defined in `kitty/state.h`:
- `OSWindow` (~line 216-256): Contains `handle` (GLFW window pointer), `tabs[]`, `live_resize` state, `close_request` enum, viewport dimensions, and rendering state.
- `Tab` (~line 186-191): Contains `windows[]`, `border_rects`, `active_window` index.
- `Window` (~line 156-172): Contains `id`, `visible` flag, `render_data` (screen, VAO), `geometry`, `mouse_pos`.

> **Key insight**: Using contiguous arrays rather than linked lists means that array compaction after element removal is an O(n) memmove, but the arrays are small (typically single-digit windows per tab) and the compaction is atomic from the perspective of the main thread — no other thread observes intermediate states.

### 1.4 Cross-Thread Primitives

Three classes of primitives coordinate the threads:

**1. `children_mutex`** (`pthread_mutex_t`, declared at `kitty/child-monitor.c` ~line 87):
Protects `children[]`, `add_queue[]`, `remove_queue[]`, and `monitored_pids[]`. The convenience macro `children_mutex(lock)` / `children_mutex(unlock)` expands to `pthread_mutex_lock(&children_lock)` / `pthread_mutex_unlock(&children_lock)`.

**2. Wakeup FDs** (initialized in `kitty/loop-utils.c` `init_loop_data()` ~line 58-77):
- On Linux: Uses `eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK)` — a single FD that can be written to wake the I/O thread from `poll()`.
- On macOS/others: Uses a self-pipe via `self_pipe()` (`kitty/loop-utils.h` ~line 52-73).
- The `wakeup_loop()` function (`kitty/loop-utils.c` ~line 112-127) writes to this FD, handling `EINTR` retries.

**3. Signal FD** (initialized in `kitty/loop-utils.c` `init_signal_handlers()` ~line 34-56):
- On Linux (`HAS_SIGNAL_FD`): `sigprocmask(SIG_BLOCK)` blocks signals process-wide, then `signalfd(-1, &signals, SFD_NONBLOCK | SFD_CLOEXEC)` creates a readable FD for synchronous signal consumption.
- On macOS: `sigaction(SA_SIGINFO)` installs `handle_signal()` (~line 14-31) which writes `siginfo_t` structs atomically to a self-pipe (guaranteed atomic for writes < PIPE_BUF).

> **Rationale**: Converting asynchronous signals into pollable FD events allows the I/O thread to handle signals in the same `poll()` loop as child I/O, without needing signal-safe code paths in the main thread. This is a textbook pattern for safe signal handling in event-driven systems.

### 1.5 Main Loop Tick Ordering

The function `process_global_state()` at `kitty/child-monitor.c` ~line 1224-1256 defines the strict execution order within each main loop tick:

1. **`process_pending_resizes(now)`** (~line 1232-1235) — Commits debounced resize events to viewports
2. **`parse_input(self)`** (~line 1236) — Drains `remove_queue`, processes death notifications, parses screen input from children
3. **`render(now, input_read)`** (~line 1237) — Renders all OS windows
4. **(macOS) `process_cocoa_pending_actions()`** (~line 1239-1242) — Handles deferred menu actions
5. **`report_reaped_pids()`** (~line 1244) — Reports monitored PID deaths to Python
6. **`process_pending_closes(self)`** (~line 1246) — Handles close requests and may trigger shutdown

> **Rationale**: This strict ordering is the backbone of kitty's state consistency. Resizes are committed *before* input is parsed, so newly resized screens receive data at the correct dimensions. Deaths are processed *before* rendering, so dead windows are cleaned up before the renderer tries to draw them. Closes happen *last* to avoid use-after-free — by the time a window is destroyed, all pending I/O and rendering for that tick is already complete.

---

## 2. Window Lifecycle

### 2.1 Creation (5-Step Flow)

Window creation follows a well-defined pipeline that crosses the Python/C boundary:

1. **Python orchestration**: `Boss` calls `Tab.new_window()` (`kitty/tabs.py` ~line 504-543), which creates a `Child` object and a `Window` object. The `Window.__init__()` (`kitty/window.py` ~line 570-610) initializes `self.destroyed = False`, creates the `Screen`, and stores a `weakref` to its parent tab (`self.tabref = weakref.ref(tab)` at line 596).

2. **PTY creation**: `Child.fork()` (`kitty/child.py` ~line 276-330) calls `openpty()` (~line 170-175) to create a PTY master/slave pair via `os.openpty()`, then `os.fork()` to spawn the child process.

3. **Registration with ChildMonitor**: `add_child()` (`kitty/child-monitor.c` ~line 304-321) is called from Python. It acquires `children_mutex`, validates capacity, places the new `Child` struct in `add_queue[add_queue_count]`, increments `add_queue_count`, releases the mutex, and calls `wakeup_io_loop()`.

4. **I/O thread pickup**: In the I/O thread's `io_loop()`, at the top of each iteration (~line 1492-1495), `add_children()` (~line 1281-1289) is called under `children_mutex`. It drains `add_queue` into `children[]` by decrementing `add_queue_count` and copying entries, setting up `children_fds[EXTRA_FDS + i]` for `poll()`.

5. **Layout**: Back in Python, `Tab._add_window()` places the window into the `WindowList` and calls `Tab.relayout()` (~line 298-301), which triggers `Window.set_geometry()` for every window in the tab.

> **Key insight**: There is a brief race window between step 3 (add to `add_queue`) and step 4 (I/O thread moves to `children[]`). During this window, the child exists in `add_queue` but not in `children[]`. The `mark_child_for_close()` function (~line 540-563) explicitly handles this by checking BOTH `children[]` AND `add_queue[]`. Similarly, `resize_pty()` (~line 591-614) uses the same `FIND` macro to search both arrays.

### 2.2 Resize (5-Stage Pipeline)

When a user resizes a kitty window, the event traverses five distinct stages before the child process sees the new terminal size:

**Stage 1 — GLFW Callback**: `framebuffer_size_callback()` in `kitty/glfw.c` ~line 330-346 fires on the main thread. It checks `ignore_resize_events`, validates minimum dimensions, then:
- Sets `global_state.has_pending_resizes = true`
- Calls `change_live_resize_state(window, true)` to mark resize in-progress
- Records `live_resize.last_resize_event_at = monotonic()`
- Stores `live_resize.width` and `live_resize.height`
- Increments `live_resize.num_of_resize_events`
- Calls `request_tick_callback()` to wake the main loop

**Stage 2 — LiveResizeInfo Debounce**: `process_pending_resizes()` in `child-monitor.c` ~line 1042-1080 runs at the start of each main loop tick. It implements two debounce paths:
- **`from_os_notification` path** (e.g., macOS sends explicit resize-start/end): Waits for `os_says_resize_complete` to be set, or falls back to `on_pause` timeout if the OS never sends the completion event.
- **Non-notification path** (e.g., X11 continuous resize): Waits for `on_end` debounce time to elapse since `last_resize_event_at`. If not enough time has passed, sets `has_pending_resizes = true` again and schedules a wakeup.

**Stage 3 — Viewport Commit**: When debounce completes, `update_os_window_viewport()` in `kitty/glfw.c` ~line 129-171 is called. This function contains the critical **no-op viewport guard** at line 138-139:
```c
if (fw == window->viewport_width && fh == window->viewport_height &&
    w == window->window_width && h == window->window_height &&
    xdpi == new_xdpi && ydpi == new_ydpi) {
    return; // no change, ignore
}
```
If dimensions have actually changed, it updates all viewport fields and calls `call_boss(on_window_resize, ...)`.

**Stage 4 — Python Relayout**: `Boss.on_window_resize()` (`kitty/boss.py` ~line 1206-1212) delegates to `TabManager.resize()` (`kitty/tabs.py` ~line 963-969), which iterates all tabs calling `Tab.relayout()` (~line 298-301). Each `relayout()` invokes the current layout engine, which calls `Window.set_geometry()` (~line 850-870) for each window.

**Stage 5 — PTY Resize**: Inside `Window.set_geometry()`, if the PTY size has changed, it calls `boss.child_monitor.resize_pty(self.id, ...)`. This enters C code at `resize_pty()` (`kitty/child-monitor.c` ~line 591-614), which acquires `children_mutex`, finds the child by ID (searching both `children[]` and `add_queue[]`), and calls `pty_resize()` (~line 576-589). The `pty_resize()` function calls `ioctl(fd, TIOCSWINSZ, dim)` with EINTR retry. The kernel then delivers `SIGWINCH` to the child process group.

> **Rationale**: The 5-stage pipeline ensures that SIGWINCH is only delivered to the child process *after* the debounce is complete, the viewport is committed, and the Python layout is updated. This means the child process sees a terminal size that is consistent with kitty's internal state — there is no window of time where the child has a different size than what kitty thinks.

### 2.3 Destruction (3 Entry Points + Cascade Cleanup)

Window destruction can be initiated through three distinct entry points:

**Entry Point 1 — Child Death (I/O thread detection)**:
The I/O thread detects child death in `io_loop()` (~line 1528-1537) when `read_bytes()` returns false (indicating `POLLHUP` or read-returns-zero):
```c
has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);
if (!has_more) {
    children_mutex(lock);
    children[i].needs_removal = true;
    children_mutex(unlock);
}
```
Alternatively, `SIGCHLD` arrives on the signal FD, triggering `reap_children()` (~line 1412-1426), which calls `waitpid(-1, &status, WNOHANG)` in a loop. If `close_on_child_death` is enabled, `mark_child_for_removal()` (~line 1386-1395) sets `needs_removal = true`.

**Entry Point 2 — User/Programmatic Close**:
`Boss.mark_window_for_close()` (`kitty/boss.py` ~line 920-928) calls `self.child_monitor.mark_for_close(window.id)`, which enters `mark_child_for_close()` (`kitty/child-monitor.c` ~line 540-563). This function acquires `children_mutex` and searches both `children[]` and `add_queue[]` to set `needs_removal = true`.

**Entry Point 3 — OS Window Close**:
`window_close_callback()` in `kitty/glfw.c` ~line 248-256 sets `close_request = CONFIRMABLE_CLOSE_REQUESTED` on the `OSWindow` and sets `global_state.has_pending_closes = true`, then calls `request_tick_callback()`.

**The Cleanup Cascade**:

Once `needs_removal` is set, the cleanup proceeds through these stages:

1. **I/O thread**: `remove_children()` (`kitty/child-monitor.c` ~line 1312-1333) iterates `children[]` in reverse order. For each child with `needs_removal = true`, it calls `cleanup_child()` (~line 1306-1309) which closes the PTY FD and sends `SIGHUP` to the child's process group. The child entry is moved to `remove_queue[]`, and `children[]` is compacted.

2. **Main thread (under lock)**: `parse_input()` (~line 450-538) acquires `children_mutex` and drains `remove_queue` into `remove_notify[]`, incrementing refcounts. The mutex is then released.

3. **Main thread (outside lock)**: For each entry in `remove_notify[]`, `parse_input()` calls `do_parse()` to flush any remaining screen data, then calls `self->death_notify` — which is the Python `Boss.on_child_death()` (~line 881-918).

4. **Python cascade**: `Boss.on_child_death()` executes within `suppress_focus_change_events()` context:
   - Pops the window from `window_id_map`
   - Calls `window.destroy()` (sets `self.destroyed = True`, resets screen callbacks)
   - Finds the containing tab and calls `tab.remove_window(window)`
   - Calls `_cleanup_tab_after_window_removal()` which checks if the tab is empty, removes it if so, then checks if the tab manager has zero tabs and closes the OS window if so

> **Key insight**: The death notification processing happens OUTSIDE the `children_mutex` lock. This is critical because Python callbacks might re-enter C code that needs the same lock (e.g., `mark_child_for_close()`). Since `pthread_mutex_t` is non-recursive by default, holding the lock during callbacks would deadlock. The snapshot-then-process pattern (copy to `remove_notify[]` under lock, process outside lock) avoids this.

---

## 3. Signal Delivery and Timing

### 3.1 signalfd Mechanism (Linux)

On Linux systems with `signalfd()` support (detected via `__has_include(<sys/signalfd.h>)` in `kitty/loop-utils.h` ~line 14-18):

1. `init_signal_handlers()` (`kitty/loop-utils.c` ~line 34-56) calls `sigprocmask(SIG_BLOCK, &signals, NULL)` to block all handled signals process-wide.
2. `signalfd(-1, &signals, SFD_NONBLOCK | SFD_CLOEXEC)` creates a file descriptor that becomes readable when a blocked signal arrives.
3. `read_signals()` (~line 130-179) reads `signalfd_siginfo` structs from this FD and converts them to `siginfo_t` for the callback.

### 3.2 Self-Pipe Mechanism (macOS)

On macOS (no `signalfd` support), `kitty/loop-utils.c` ~line 46-53:

1. A self-pipe is created via `self_pipe(signal_fds, true)`.
2. `sigaction(SA_SIGINFO)` installs `handle_signal()` (~line 14-31) for each signal.
3. When a signal fires, the handler writes the raw `siginfo_t` bytes to the pipe. Since `sizeof(siginfo_t) < PIPE_BUF`, writes are guaranteed atomic by POSIX.
4. `read_signals()` (~line 160-179) reads from the pipe, reassembling `siginfo_t` structs and invoking the callback.

### 3.3 Signal Classification

The I/O thread classifies signals in `handle_signal()` at `kitty/child-monitor.c` ~line 1361-1383:

| Signal | Flag Set | Action |
|--------|----------|--------|
| `SIGINT`, `SIGTERM`, `SIGHUP` | `kill_signal = true` | Triggers application shutdown |
| `SIGCHLD` | `child_died = true` | Triggers `reap_children()` |
| `SIGUSR1` | `reload_config = true` | Triggers config reload |
| `SIGUSR2` | (none) | Logged only |

When `kill_signal` or `reload_config` is set, the I/O thread stores these under `children_mutex` as global flags (`kill_signal_received`, `reload_config_signal_received` at ~line 1520-1524). The main thread reads these flags in `parse_input()` (~line 465-475).

### 3.4 SIGWINCH Delivery Timing

SIGWINCH is *not* handled by kitty's signal infrastructure — it is sent *by* kitty to child processes. The delivery happens at the very end of the resize pipeline:

1. Debounce completes in `process_pending_resizes()`
2. Viewport is committed in `update_os_window_viewport()`
3. Python relayout runs, calling `Window.set_geometry()`
4. `set_geometry()` calls `child_monitor.resize_pty()` if the PTY size changed
5. `resize_pty()` calls `ioctl(fd, TIOCSWINSZ, &dim)` via `pty_resize()` (~line 576-589)
6. The kernel delivers SIGWINCH to the child's process group

> **Rationale**: Because SIGWINCH delivery is the *last* step after all internal state is updated, the child process can immediately query its terminal size (via `ioctl(TIOCGWINSZ)`) and get the correct dimensions. There is no race between kitty updating its internal layout and the child seeing the new size.

### 3.5 SIGCHLD Timing and Coalescing

SIGCHLD is delivered asynchronously by the kernel. Multiple child deaths can coalesce into a single signal delivery. The `reap_children()` function (~line 1412-1426) handles this with a loop:

```c
while(true) {
    pid = waitpid(-1, &status, WNOHANG);
    if (pid == -1) {
        if (errno != EINTR) break;
    } else if (pid > 0) {
        if (enable_close_on_child_death) mark_child_for_removal(self, pid);
        mark_monitored_pids(pid, status);
    } else break;
}
```

The `WNOHANG` flag ensures non-blocking reaping, and the loop continues until all dead children are collected. This means even if only one SIGCHLD is delivered for multiple child deaths, all dead children will be reaped.

> **Key insight**: The combination of `waitpid(-1, ..., WNOHANG)` looping and the signal FD mechanism means kitty is robust against SIGCHLD coalescing. Whether one signal or five are delivered, the same code path collects all dead children.

---

## 4. State Consistency Mechanisms

Kitty employs eight distinct mechanisms to maintain state consistency during rapid lifecycle events:

### 4.1 LiveResizeInfo Debouncing State Machine

Defined in `kitty/state.h` ~line 196-202:

```c
typedef struct {
    monotonic_t last_resize_event_at;
    bool in_progress;
    bool from_os_notification;
    bool os_says_resize_complete;
    unsigned int width, height, num_of_resize_events;
} LiveResizeInfo;
```

The `process_pending_resizes()` function at `kitty/child-monitor.c` ~line 1042-1080 implements the state machine:
- When `from_os_notification` is true (macOS), it waits for the OS to signal completion OR for the `on_pause` timeout to prevent hangs.
- When `from_os_notification` is false (X11), it waits for `on_end` debounce time since the last event.
- On commit, the `LiveResizeInfo` is zeroed via `zero_at_ptr(&w->live_resize)` (~line 1075).

> **Rationale**: Without debouncing, a window drag on X11 could generate dozens of resize events per second, each triggering a full relayout cascade and PTY resize. The debounce collapses these into a single state transition, dramatically reducing CPU load and preventing screen flickering.

### 4.2 No-Op Viewport Guard

In `update_os_window_viewport()` at `kitty/glfw.c` ~line 138-139:

```c
if (fw == window->viewport_width && fh == window->viewport_height &&
    w == window->window_width && h == window->window_height &&
    xdpi == new_xdpi && ydpi == new_ydpi) {
    return; // no change, ignore
}
```

This guard checks six independent dimensions. If the debounced resize event ultimately results in the same dimensions (e.g., a drag that ends where it started, or redundant DPI notifications), the entire downstream cascade (Python relayout, PTY resize, SIGWINCH) is suppressed.

> **Rationale**: This is a critical performance optimization that also prevents unnecessary SIGWINCH signals to child processes. Applications like vim that respond to SIGWINCH by redrawing their entire screen would otherwise redraw unnecessarily.

### 4.3 `destroyed` Flag on Python Window Objects

The `Window` class (`kitty/window.py`) initializes `self.destroyed = False` at ~line 597. The flag is set to `True` in `Window.destroy()` at ~line 1562.

Two critical guard checks use this flag:

- `Window.set_geometry()` (~line 850-852): `if self.destroyed: return` — prevents geometry updates on dead windows
- `Window.focus_changed()` (~line 1123-1124): `if self.destroyed or self.ignore_focus_changes or self.is_focused == focused: return` — prevents focus events on dead windows

> **Rationale**: Between the time a window is logically destroyed and the time all references to it are cleared, code paths may still attempt to call methods on it (e.g., during relayout triggered by a sibling window's removal). The `destroyed` flag turns these into safe no-ops.

### 4.4 `ignore_focus_changes` Suppression

`Boss.suppress_focus_change_events()` (`kitty/boss.py` ~line 869-879) is a context manager:

```python
@contextmanager
def suppress_focus_change_events(self) -> Generator[None, None, None]:
    changes = {}
    for w in self.window_id_map.values():
        changes[w] = w.ignore_focus_changes
        w.ignore_focus_changes = True
    try:
        yield
    finally:
        for w, val in changes.items():
            w.ignore_focus_changes = val
```

This is used in `Boss.on_child_death()` (~line 886) and during window movement operations (~line 2815). It prevents the focus-change cascade that would normally occur when a window is removed from a tab — the layout engine would select a new active window, which would trigger focus events, which could trigger watchers and actions that might interfere with the ongoing removal.

> **Rationale**: Window removal is a multi-step operation (destroy window, remove from tab, potentially remove tab, potentially close OS window). Focus changes during this cascade could trigger re-entrant state modifications. Suppressing focus events ensures the entire removal completes atomically from the perspective of focus-tracking code.

### 4.5 Mutex-Protected Queue Transfers

The child lifecycle pipeline uses four arrays, all protected by `children_mutex`:

```
add_queue[] → children[] → remove_queue[] → remove_notify[]
```

- `add_queue[]` — Filled by main thread in `add_child()` (~line 304-321)
- `children[]` — Drained from `add_queue` by I/O thread in `add_children()` (~line 1281-1289)
- `remove_queue[]` — Filled by I/O thread in `remove_children()` (~line 1312-1333)
- `remove_notify[]` — Drained from `remove_queue` by main thread in `parse_input()` (~line 457-463)

> **Key insight**: `remove_notify[]` processing happens OUTSIDE the lock (`kitty/child-monitor.c` ~line 517-526). This is because processing involves calling Python's `death_notify` callback, which could re-enter C code requiring `children_mutex`. Since the mutex is non-recursive, processing under the lock would deadlock. The snapshot-then-process pattern (copy under lock, process without lock) avoids this while maintaining consistency.

### 4.6 Weak References for Tab/Boss Back-References

Kitty uses Python `weakref` to prevent circular reference chains:

- `Window.tabref` = `weakref.ref(tab)` (`kitty/window.py` ~line 596)
- `Tab.tab_manager_ref` = `weakref.ref(tab_manager)` (`kitty/tabs.py` ~line 141)
- `WindowList.tabref` = `weakref.ref(tab)` (`kitty/window_list.py` ~line 152)

Before accessing these back-references, code always checks for `None`:
```python
tm = self.tab_manager_ref()
if tm is not None:
    tm.title_changed(self)
```
(Example from `kitty/tabs.py` ~line 291-293)

> **Rationale**: If a `Tab` is destroyed (garbage collected) but a `Window` still holds a reference to it, the `weakref` returns `None` rather than a dangling pointer. This is a safety net against the cascade cleanup not perfectly clearing all references — the weakref degrades gracefully to a no-op.

### 4.7 ID-Lookup-Failure-as-Normal-Flow

The C-level lookup macros in `kitty/state.c` (~line 24-49) are designed so that a failed lookup is not an error:

```c
#define WITH_OS_WINDOW(os_window_id) \
    for (size_t o = 0; o < global_state.num_os_windows; o++) { \
        OSWindow *os_window = global_state.os_windows + o; \
        if (os_window->id == os_window_id) {
#define END_WITH_OS_WINDOW break; }}
```

If the ID is not found in the array, the loop body is never entered, and execution continues after `END_WITH_OS_WINDOW`. The same pattern applies to `WITH_TAB` (~line 30-37) and `WITH_WINDOW` (~line 39-49).

This means functions like `remove_window()` (`kitty/state.c` ~line 367-372) or `resize_screen()` can be called with stale IDs and will simply do nothing:

```c
static void remove_window(id_type os_window_id, id_type tab_id, id_type id) {
    WITH_TAB(os_window_id, tab_id);
        make_os_window_context_current(osw);
        remove_window_inner(tab, id);
    END_WITH_TAB;
}
```

> **Rationale**: Because window IDs are monotonically increasing and never reused (generated by `++global_state.window_id_counter`), a stale ID cannot accidentally match a different window. This makes the ID-lookup pattern a safe, zero-cost way to handle the "window might already be gone" scenario that pervades rapid lifecycle operations.

### 4.8 `REMOVER` Macro for Atomic Array Element Removal

Defined in `kitty/state.c` ~line 14-22:

```c
#define REMOVER(array, qid, count, destroy, capacity) { \
    for (size_t i = 0; i < count; i++) { \
        if (array[i].id == qid) { \
            destroy(array + i); \
            zero_at_i(array, i); \
            remove_i_from_array(array, i, count); \
            break; \
        } \
    }}
```

This macro performs find-destroy-zero-compact in a single pass:
1. Finds the element by ID
2. Calls the `destroy` callback (e.g., `destroy_window()`, `destroy_tab()`)
3. Zeros the memory at the slot
4. Compacts the array by shifting subsequent elements left

Because this happens in a single main-thread function call with no yielding, no intermediate state is visible to any other code path.

---

## 5. Window Disappears Before Reactions Complete

### 5.1 Create-and-Immediate-Close

**Scenario**: A window is created, its child enters `add_queue`, and a close request arrives before the I/O thread picks it up.

**Step-by-step analysis**:

1. `Tab.new_window()` calls `add_child()`, placing the child in `add_queue[]` under `children_mutex`.
2. Before the I/O thread's next iteration, `Boss.mark_window_for_close()` is called.
3. `mark_child_for_close()` (~line 540-563) acquires `children_mutex`, searches `children[]` first (not found), then searches `add_queue[]` (~line 551-558) and sets `needs_removal = true`.
4. When the I/O thread calls `add_children()` (~line 1281-1289), it moves the child from `add_queue` to `children[]`. The `needs_removal` flag is preserved because it's part of the `Child` struct that's copied.
5. On the very next iteration, `remove_children()` (~line 1312-1333) finds `needs_removal = true`, closes the FD, sends SIGHUP, and moves the child to `remove_queue`.
6. The main thread processes the death notification normally.

**Net effect**: The child briefly exists in `children[]` then is immediately cleaned up. No I/O is processed for the child because `needs_removal` is already set when it enters `children[]`. The PTY FD was only open long enough for the child process to start.

> **Key insight**: The dual-search in `mark_child_for_close()` is the critical safety net. Without it, a close request during the `add_queue` window would be silently lost, leaving an orphaned child process.

### 5.2 Resize Arrives After Window Death

**Scenario**: A relayout triggers `Window.set_geometry()` for a window that has already been destroyed (e.g., during a tab relayout triggered by a sibling window's removal).

**Step-by-step analysis**:

1. Window A dies. `boss.on_child_death()` calls `window_A.destroy()`, setting `window_A.destroyed = True`.
2. `tab.remove_window(window_A)` removes it from the `WindowList` and calls `tab.relayout()`.
3. `Tab.relayout()` calls `Window.set_geometry()` for all remaining windows.
4. If any code path still holds a reference to window A and attempts `window_A.set_geometry()`, the guard at `kitty/window.py` ~line 851 catches it: `if self.destroyed: return`.
5. At the C level, `remove_window()` has already removed window A from `tab->windows[]`. The `WITH_WINDOW` macro will fail to find window A's ID, and the body will not execute.

**Net effect**: Both Python and C layers independently guard against operations on dead windows. The resize is harmlessly ignored at both levels.

### 5.3 Death During Live Resize

**Scenario**: A window is being resized (drag in progress), and its child process dies mid-resize.

**Step-by-step analysis**:

1. Multiple `framebuffer_size_callback()` events have set `has_pending_resizes = true` and populated `live_resize`.
2. The child process exits. The I/O thread detects this via `POLLHUP`/read-returns-zero and sets `needs_removal = true`.
3. On the next main loop tick, `process_global_state()` runs:
   - **First**: `process_pending_resizes()` commits the viewport (if debounce has elapsed). This calls `update_os_window_viewport()` and triggers the Python relayout cascade. The relayout calls `set_geometry()` for all windows including the dying one — but the child is not yet in `remove_notify`, so the window is not yet `destroyed`. The relayout succeeds and `resize_pty()` may even be called (which will find the child in `children[]` since removal hasn't happened yet).
   - **Second**: `parse_input()` drains `remove_queue` to `remove_notify`, then calls `boss.on_child_death()`.
   - `window.destroy()` sets `destroyed = True`.
   - `tab.remove_window()` removes the window and triggers another relayout of the remaining windows.

4. If the OS window has no more tabs after the cascade, it too is closed.

**Net effect**: The viewport commit for the dead window's resize was "wasted" — the PTY was resized and SIGWINCH was sent to a process that was dying — but this is completely harmless. The strict tick ordering (resize → parse → render → close) ensures the death is processed cleanly after the resize completes.

### 5.4 OS Window Close During Active Operations

**Scenario**: The user clicks the OS window's close button while operations (input parsing, rendering) are ongoing.

**Step-by-step analysis**:

1. `window_close_callback()` (`kitty/glfw.c` ~line 248-256) fires:
   - Sets `close_request = CONFIRMABLE_CLOSE_REQUESTED`
   - Sets `global_state.has_pending_closes = true`
   - Calls `request_tick_callback()`
   - Calls `glfwSetWindowShouldClose(window, false)` to prevent GLFW from destroying the window immediately

2. On the next tick, `process_pending_closes()` (~line 1098-1134) runs **last** (after resize, parse, and render):
   - For `CONFIRMABLE_CLOSE_REQUESTED`: Sets `close_request = CLOSE_BEING_CONFIRMED`, calls `confirm_os_window_close()` in Python.
   - If confirmed (Python returns `IMPERATIVE_CLOSE_REQUESTED`): Calls `close_os_window()` (~line 1082-1095).

3. `close_os_window()` executes:
   - `destroy_os_window(os_window)` — Destroys the GLFW window handle
   - `call_boss(on_os_window_closed, ...)` — Python cleanup: pops `TabManager` from `os_window_map`, calls `tm.destroy()`, removes all window IDs from `window_id_map`
   - Loops through ALL tabs and windows, calling `mark_child_for_close()` for each
   - `remove_os_window(os_window->id)` — Removes from `global_state.os_windows[]` using `REMOVER`

**Net effect**: Because close processing happens last in the tick, all pending resizes and input are processed before the window is destroyed. The `CloseRequest` state machine (`NO_CLOSE_REQUESTED → CONFIRMABLE_CLOSE_REQUESTED → CLOSE_BEING_CONFIRMED → IMPERATIVE_CLOSE_REQUESTED`, defined in `kitty/state.h` ~line 194) ensures the close goes through proper confirmation flow and cannot be accidentally triggered twice.

---

## 6. Resolving Conflicting Views

When rapid lifecycle events occur, different parts of the system may temporarily hold conflicting views of what is alive. Kitty resolves these conflicts through seven patterns:

### 6.1 Authoritative Source of Truth

The C-level `GlobalState` hierarchy (`global_state.os_windows[].tabs[].windows[]`) is the single source of truth. Python objects (`Boss`, `Tab`, `Window`) hold derived state. When conflicts arise, the C state wins — this is enforced by the `WITH_*` lookup macros that only operate on what actually exists in the arrays.

### 6.2 Idempotent `needs_removal`

The `needs_removal` flag on `Child` structs is a boolean. Setting it to `true` multiple times has no additional effect. The I/O thread checks it once during `remove_children()` and acts on it. This means multiple code paths can independently decide a child should be removed (e.g., POLLHUP detection AND SIGCHLD reaping AND explicit close) without coordination — the first one to set the flag wins, and the rest are harmless no-ops.

### 6.3 ID Lookup Failure as Normal Flow

As described in Section 4.7, the `WITH_OS_WINDOW`/`WITH_TAB`/`WITH_WINDOW` macros silently skip their body if the ID is not found. This means stale references from earlier ticks naturally become harmless. No explicit "is this window still alive?" check is needed — the lookup itself serves as the check.

### 6.4 Snapshot-Then-Process

`parse_input()` at `kitty/child-monitor.c` ~line 477-482 copies the `children[]` array to `scratch[]` under the lock:

```c
count = self->count;
for (size_t i = 0; i < count; i++) {
    scratch[i] = children[i];
    INCREF_CHILD(scratch[i]);
}
```

After releasing the lock, it processes `scratch[]` entries. This means changes to `children[]` by the I/O thread during processing do not affect the current tick's parsing — each tick works with a consistent snapshot.

### 6.5 Post-Removal Cleanup Cascade

Python's `Boss._cleanup_tab_after_window_removal()` (`kitty/boss.py` ~line 859-867) implements a bottom-up cleanup check:

```python
def _cleanup_tab_after_window_removal(self, src_tab: Tab) -> None:
    if len(src_tab) < 1:
        tm = src_tab.tab_manager_ref()
        if tm is not None:
            tm.remove(src_tab)
            src_tab.destroy()
            if len(tm) == 0:
                if not self.shutting_down:
                    self.mark_os_window_for_close(src_tab.os_window_id)
```

This cascade check runs after every window removal, ensuring that empty containers are cleaned up immediately rather than lingering.

### 6.6 Ordered Destruction via Tick Sequencing

The `process_global_state()` ordering (resize → input → render → close) ensures that conflicting operations are resolved in a deterministic order. A resize and a close for the same window are not processed simultaneously — the resize completes first, then the close runs.

### 6.7 `CloseRequest` State Machine

The `CloseRequest` enum (`kitty/state.h` ~line 194) defines four states:

```
NO_CLOSE_REQUESTED → CONFIRMABLE_CLOSE_REQUESTED → CLOSE_BEING_CONFIRMED → IMPERATIVE_CLOSE_REQUESTED
```

`process_pending_closes()` (`kitty/child-monitor.c` ~line 1098-1134) uses a switch statement to handle each state:
- `NO_CLOSE_REQUESTED` → window stays open
- `CONFIRMABLE_CLOSE_REQUESTED` → transitions to `CLOSE_BEING_CONFIRMED`, asks Python for confirmation
- `CLOSE_BEING_CONFIRMED` → window stays open (waiting for user response)
- `IMPERATIVE_CLOSE_REQUESTED` → window is closed immediately

This state machine prevents double-close issues: once a window is in `CLOSE_BEING_CONFIRMED` state, a second close request is absorbed — the confirmation dialog is already showing.

---

## 7. Cascade Cleanup Pattern

### 7.1 Window → Tab Emptiness Cascade

When `Tab.remove_window()` (`kitty/tabs.py` ~line 580-591) removes a window:

1. `self.windows.remove_window(window)` removes it from the `WindowList`
2. `remove_window(self.os_window_id, self.id, window.id)` removes it from the C-level `Tab.windows[]` array via `REMOVER`
3. `self.relayout()` recomputes layout for remaining windows

After this, `Boss._cleanup_tab_after_window_removal()` checks `len(src_tab) < 1`. If the tab is empty, the cascade continues upward.

### 7.2 Tab → OS Window Emptiness Cascade

When the last window in a tab is removed:

1. `tm.remove(src_tab)` calls `TabManager._remove_tab()` (~line 932-937), which calls `remove_tab()` (C level) and removes the tab from `self.tabs`
2. `src_tab.destroy()` calls `Tab.destroy()` (~line 850), which destroys all remaining windows and their resources
3. `len(tm) == 0` checks if the tab manager has no tabs remaining
4. If empty and not shutting down: `self.mark_os_window_for_close(src_tab.os_window_id)` schedules the OS window for closure

### 7.3 Multi-Window Cascade During OS Window Close

When an OS window is closed via `close_os_window()` (`kitty/child-monitor.c` ~line 1082-1095):

```c
for (size_t t=0; t < os_window->num_tabs; t++) {
    Tab *tab = os_window->tabs + t;
    for (size_t w = 0; w < tab->num_windows; w++)
        mark_child_for_close(self, tab->windows[w].id);
}
```

This iterates ALL tabs and ALL windows within the closing OS window and marks every child for removal. The actual cleanup of these children follows the normal `needs_removal → remove_children → remove_queue → remove_notify → death_notify` pipeline over subsequent I/O thread iterations and main loop ticks.

### 7.4 Stale Reference Guards During Cascade

During the cascade, multiple guards prevent use-after-free:

- **Weak references**: `window.tabref()` returns `None` if the tab has been garbage collected
- **ID lookups**: C-level `WITH_*` macros skip bodies for removed entities
- **`destroyed` flag**: Python `Window.destroy()` sets the flag before any removal operations
- **`window_id_map.pop()`**: `Boss.on_child_death()` immediately removes the window from the lookup map (~line 883), preventing any subsequent code from finding it by ID

---

## 8. Timing-Sensitive Edge Cases

### 8.1 Create-and-Immediate-Close Race

**Window**: Between `add_child()` and the I/O thread's `add_children()`.

**Resolution**: `mark_child_for_close()` (~line 540-563) searches both `children[]` AND `add_queue[]`. The `needs_removal` flag is set in whichever array the child currently resides. When `add_children()` moves it to `children[]`, the flag persists. `remove_children()` immediately removes it on the next I/O iteration.

### 8.2 Resize-Then-Die

**Window**: Debounce timer is running, child dies during the debounce period.

**Resolution**: Per tick ordering, `process_pending_resizes()` runs first and may commit a viewport update for the dying window's OS window. This triggers a relayout that calls `set_geometry()` on all windows — including the dying one if it hasn't been removed yet. The resize and SIGWINCH are sent to a process that may already be a zombie. This is harmless because:
- `pty_resize()` retries on `EINTR` and gracefully handles `EBADF`/`ENOTTY` (~line 576-589)
- SIGWINCH to a dead process is silently ignored by the kernel
- The death notification is processed immediately after in `parse_input()`

### 8.3 OS-Close-During-Live-Resize

**Window**: User clicks close button while a live resize drag is in progress.

**Resolution**: `window_close_callback()` sets `close_request = CONFIRMABLE_CLOSE_REQUESTED` and `has_pending_closes = true`. In `process_global_state()`, `process_pending_resizes()` runs first (~line 1232-1235), committing the resize. Then `parse_input()` and `render()` run. Finally, `process_pending_closes()` runs last (~line 1246), handling the close. The resize completes fully before the close begins — no torn state.

### 8.4 add_queue Race Window

**Window**: Between `add_child()` returning and the I/O thread calling `add_children()`.

**Resolution**: During this window, the child exists only in `add_queue` and is not being polled by the I/O thread. The child process is already running (it was `fork()`ed before `add_child()` was called), so it may be producing output that buffers in the kernel's PTY buffer. This is safe — the kernel buffers PTY output until the master FD is read. Once `add_children()` moves the child to `children[]`, the I/O thread begins polling and drains any accumulated output.

### 8.5 EINTR Handling in pty_resize

**Window**: A signal arrives during the `ioctl(TIOCSWINSZ)` call in `pty_resize()`.

**Resolution**: The `pty_resize()` function (~line 576-589) has explicit EINTR handling:

```c
static bool pty_resize(int fd, struct winsize *dim) {
    while(true) {
        if (ioctl(fd, TIOCSWINSZ, dim) == -1) {
            if (errno == EINTR) continue;
            if (errno != EBADF && errno != ENOTTY) {
                log_error("Failed to resize tty...");
                return false;
            }
        }
        break;
    }
    return true;
}
```

`EINTR` triggers a retry. `EBADF` and `ENOTTY` (indicating the FD was closed or is no longer a terminal — which happens if the child died between the mutex-protected FD lookup and the `ioctl` call) are silently accepted. All other errors are logged but do not crash the application.

---

## 9. Summary

Kitty maintains internal state consistency during rapid window lifecycle events through eight fundamental design principles:

1. **Single-threaded main loop guarantees ordered processing**: All state mutations and Python callbacks execute on the main thread in a strict order (resize → parse → render → close). This eliminates races between concurrent state changes and ensures each tick sees a consistent world view.

2. **Mutex-protected queue transfers bridge threads safely**: The `children_mutex` protects all transitions in the `add_queue → children → remove_queue → remove_notify` pipeline. The I/O thread and main thread never simultaneously mutate the same data structures.

3. **Debouncing collapses rapid events into single state transitions**: The `LiveResizeInfo` state machine absorbs multiple resize events during window dragging and commits a single viewport update after the user stops. This prevents O(n) relayout cascades per drag pixel.

4. **Guard flags prevent stale operations**: The Python-level `destroyed` flag on `Window` objects and the C-level `needs_removal` flag on `Child` structs ensure that operations on dead or dying entities are safely converted to no-ops. The `ignore_focus_changes` flag prevents re-entrant state modifications during removal cascades.

5. **ID-based lookups make stale references harmless**: The `WITH_OS_WINDOW`/`WITH_TAB`/`WITH_WINDOW` macros silently skip their body if the target ID is not found. Since IDs are monotonically increasing and never reused, a stale ID cannot accidentally match a different entity.

6. **Weak references prevent Python-level dangling pointers**: `Window.tabref` and `Tab.tab_manager_ref` use `weakref.ref()`, which returns `None` if the referent has been garbage collected. All call sites check for `None` before accessing the referenced object.

7. **Snapshot-then-process pattern avoids holding locks during callbacks**: `parse_input()` copies `children[]` to `scratch[]` under the mutex, then releases the lock before processing. This prevents deadlocks from Python callbacks that might re-enter C code requiring the same lock.

8. **Cascade cleanup ensures no orphaned resources**: The `_cleanup_tab_after_window_removal()` function checks for empty containers bottom-up (empty tab → remove tab → empty tab manager → close OS window). Combined with the multi-window mark-for-close loop in `close_os_window()`, this ensures every child process, PTY FD, and GUI resource is properly cleaned up regardless of the destruction entry point.

Together, these mechanisms form a robust defense-in-depth strategy where any single failure point is covered by at least one other mechanism. The result is a system that gracefully handles the inherent chaos of rapid window creation, resizing, and destruction without losing consistency.
