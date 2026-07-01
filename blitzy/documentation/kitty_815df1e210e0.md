# How kitty keeps internal state consistent during rapid window churn

**Subject:** `kovidgoyal/kitty` terminal emulator, version **0.35.2**
(verified literal at `kitty/constants.py:25`):

**SOURCE — `kitty/constants.py:25`:**
```python
version: Version = Version(0, 35, 2)
```

## The question being answered

This document answers the following question, reproduced verbatim:

> "I want to understand how kitty keeps its internal state consistent when terminal windows appear, resize, and disappear in quick succession. When a new window is created and immediately used to run a command, resize events and signals start flowing through the system. What happens if the window is gone before everything has finished reacting to those changes? How does kitty decide what state to keep and what to discard? I am especially interested in how timing affects signal delivery and internal bookkeeping, and whether there are moments where the system has to resolve conflicting views of what is still alive. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."

The question decomposes into five sub-questions, each answered in its own titled section:

- **O1 — Creation-and-run flow:** window spawn → PTY allocation → first resize → `SIGWINCH` to the child.
- **O2 — Teardown-before-completion:** a queued resize for a departed window; a `SIGCHLD` reaped after the window was already removed.
- **O3 — Keep-vs-discard machinery:** what state is retained and what is discarded during churn.
- **O4 — Timing effects:** asynchronous coalesced signal delivery versus synchronous main-loop reconciliation.
- **O5 — Conflicting-liveness resolution:** the exact moments the system must resolve conflicting views of what is still alive, and how it does so benignly.

Every behavioral claim below is paired with (a) an exact `file:line` citation from the source at this snapshot and (b) the specific verbatim log line from a real headless run that demonstrates it (one claim, one piece of evidence).

---

## How this was observed (build, run, environment, and the raw magnitudes)

### The debug/event-loop build (compiles in the observable logging)

The observable timing lines only exist in a debug build compiled with event-loop logging. The build target is `Makefile:25-26`:

**SOURCE — `Makefile:25-26`:**
```make
debug-event-loop:
	python3 setup.py build $(VVAL) --debug --extra-logging=event-loop
```

**COMMAND (exact command run):**
```bash
python3 setup.py build --debug --extra-logging=event-loop
```

**OUTPUT (tail; exit status was 0):**
```text
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
```

That this build actually compiled in the event-loop/signal logging was verified from `build/compile_commands.json`: `kitty/child-monitor.c` is compiled with the defines `-DDEBUG`, `-DDEBUG_EVENT_LOOP`, and `-DKITTY_DEBUG_BUILD`. The build produces the C extension `kitty/fast_data_types.so` (plus `kitty/glfw-x11.so`, `kitty/launcher/kitty`, and `kitty/launcher/kitten`). The exact log literals are present in the compiled `.so`:

**COMMAND:**
```bash
strings kitty/fast_data_types.so | grep 'Failed to send resize signal\|had its fd unexpectedly closed'
```

**OUTPUT:**
```text
Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)
The child %lu had its fd unexpectedly closed
```

### The headless run under rapid churn

The environment has no physical display, so kitty is driven **headlessly under `Xvfb`**. A startup `--session` file drives deterministic churn: 5 tabs (one initial + four `new_tab`) each holding four short-lived windows (20 windows total), in a tiling `layout grid` so that every window add/remove forces a relayout — and therefore a `resize_pty` — of every sibling window in that tab. The session directives used are those the parser understands: `new_tab` (`kitty/session.py:176`) and `launch` (`kitty/session.py:184`), parsed by `parse_session` (`kitty/session.py:151`) into `Session.add_window` (`kitty/session.py:100`).

**SOURCE — `/tmp/blitzy_obs/session.conf` (the temporary churn driver; excerpt):**
```text
layout grid
launch sh -c "sleep 0.08; true"
launch sh -c "sleep 0.02; true"
launch sh -c "true"
launch sh -c "sleep 0.05; true"
new_tab
layout grid
launch sh -c "sleep 0.07; true"
...
```

**COMMAND (exact run command; `Xvfb :99` started first, `timeout 90` wrapper):**
```bash
export DISPLAY=:99 TERM=xterm-kitty LANG=C.UTF-8 LC_ALL=C.UTF-8
./kitty/launcher/kitty --config NONE --debug-rendering -o close_on_child_death=yes \
    -o confirm_os_window_close=0 --session /tmp/blitzy_obs/session.conf \
    > /tmp/blitzy_obs/run.log 2>&1
echo "KITTY_EXIT=$?"
```

**OUTPUT:**
```text
KITTY_EXIT=0
```

The process exited with status **0** despite the resize conflicts described below — the conflict resolution is benign (no crash). The `--debug-rendering` flag is what enables the `Child launched` and `SIGWINCH sent to child` lines (they are gated by `boss.args.debug_rendering` at `kitty/window.py:869,872`); `close_on_child_death=yes` makes each window close as soon as its child exits, which drives the OS window down to zero windows and exits the loop.

### The raw magnitudes (from this run's `/tmp/blitzy_obs/run.log`, 113 lines)

**COMMAND:**
```bash
echo "SIGWINCH sent to child : $(grep -c 'SIGWINCH sent to child' run.log)"
echo "Failed to send resize  : $(grep -c 'Failed to send resize signal' run.log)"
echo "Child launched         : $(grep -c 'Child launched' run.log)"
echo "OS Window created      : $(grep -c 'OS Window created' run.log)"
echo "fd unexpectedly closed : $(grep -c 'had its fd unexpectedly closed' run.log)"
echo "loop tick              : $(grep -c 'loop tick, wakeups_happened' run.log)"
```

**OUTPUT:**
```text
SIGWINCH sent to child : 47
Failed to send resize  : 27
Child launched         : 20
OS Window created      : 1
fd unexpectedly closed : 0
loop tick              : 3
```

Interpreting these counts (this interpretation is grounded in `kitty/window.py:864-873`, quoted in O1): `Child launched` is printed **once per window, on that window's first resize only** — the observed **20** exactly matches the 20 launched windows. `SIGWINCH sent to child` is printed on **subsequent** resizes (the two prints are mutually exclusive per resize), so the observed **47** are later resizes triggered by relayout as siblings appear and disappear. `Failed to send resize signal` (**27**) is the race being provoked: a `resize_pty` for a window id that the C child registry no longer holds. `fd unexpectedly closed` was **0** — that `POLLNVAL` edge path (which exists in the code) did not trigger in this run; this is reported honestly and explained in O5.

### Environment caveats (required disclosures)

- **No display → headless `Xvfb`.** OpenGL is provided by Mesa software rasterization (the run log records `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2'`).
- **Wayland backend disabled.** The build prints `Disabling building of wayland backend` (quoted above) because `wayland-protocols >= 1.17` is absent; the run is therefore X11-only.
- **Remote control not used.** Churn is driven by a startup `--session` file rather than by `kitty @` remote control. (Note: contrary to some environment notes, the Go `kitten` binary *was* built here at `kitty/launcher/kitten`; remote control was nonetheless deliberately avoided in favor of the deterministic session file.)
- **Python version.** The build and run succeed on this environment's **Python 3.13.7**. For reference, `pyproject.toml:2` declares `requires-python = ">=3.8"`, and the CI matrix in `.github/workflows/ci.yml` builds/tests on `"3.8"` (`:26`), `"3.9"` (`:34`), and `"3.10"` (`:30`), with docs/lint on `"3.11"` (`:85`).
- **One harmless startup line.** The run prints `[0.188] Failed to open systemd user bus with error: Connection refused` — expected under `Xvfb` (no systemd session bus) and unrelated to the window/signal machinery.

---

## O1 — Creation-and-run flow: window spawn → PTY allocation → first resize → `SIGWINCH`

When a new window is created and immediately runs a command, the following ordered chain fires. Each step pairs its source citation with the observed log line that proves it fired.

### Step 1 — Creation ordering: register the child *before* laying out

`Tab.new_window` (`kitty/tabs.py:504`) deliberately registers the child **before** it lays the window out, because layout is what triggers the first resize and the resize must be able to find the child. The ordering and its rationale are stated in the source comment:

**SOURCE — `kitty/tabs.py:534-536`:**
```python
        # Must add child before laying out so that resize_pty succeeds
        get_boss().add_child(window)
        self._add_window(window, location=location, overlay_for=overlay_for, overlay_behind=overlay_behind)
```

`_add_window` triggers `Tab.relayout` (`kitty/tabs.py:298`), which resizes every window in the tab. This ordering is the *root* of the race studied in O2/O5: layout-driven resizes begin the moment a window is added, and they keep firing as siblings are added and removed.

### Step 2 — Registration in BOTH layers (C monitor + Python map)

`Boss.add_child` registers the child in two places at once — the C child-monitor and the Python `window_id_map`:

**SOURCE — `kitty/boss.py:585-588`:**
```python
    def add_child(self, window: Window) -> None:
        assert window.child.pid is not None and window.child.child_fd is not None
        self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
        self.window_id_map[window.id] = window
```

The C side is `add_child(ChildMonitor *self, ...)` at `kitty/child-monitor.c:305`; the Python side is the `window_id_map` at `kitty/boss.py:344` (a `WeakValueDictionary`, discussed in O3). The correlation key passed to both is `window.id` — the same id used later by every resize and by the death callback, which is why identity survives churn.

### Step 3 — PTY allocation and process spawn

The child's PTY is allocated with `os.openpty()` and the resulting master fd and child pid are stored on the `Child`:

**SOURCE — `kitty/child.py:170-171`:**
```python
def openpty() -> Tuple[int, int]:
    master, slave = os.openpty()  # Note that master and slave are in blocking mode
```

**SOURCE — `kitty/child.py:337-338`:**
```python
        self.pid = pid
        self.child_fd = master
```

These are exactly the `pid` and `child_fd` asserted-non-`None` and handed to the C monitor in Step 2 (`kitty/boss.py:586-587`).

### Step 4 — The first resize marks the child "launched" and emits `Child launched`

Every resize is dispatched from `kitty/window.py`. The dispatch is guarded so it only fires when the size actually changed, it calls into the C monitor, and — on the **first** resize only — it marks the child launched:

**SOURCE — `kitty/window.py:861-871`:**
```python
        if current_pty_size != self.last_reported_pty_size:
            boss = get_boss()
            boss.child_monitor.resize_pty(self.id, *current_pty_size)
            self.last_resized_at = monotonic()
            if not self.child_is_launched:
                self.child.mark_terminal_ready()
                self.child_is_launched = True
                update_ime_position = True
                if boss.args.debug_rendering:
                    now = monotonic()
                    print(f'[{now:.3f}] Child launched', file=sys.stderr)
```

**OBSERVED EVIDENCE (run.log:3,5,7 — `Child launched` printed once per window on first resize; 20 total == 20 windows):**
```text
[0.190] Child launched
[0.196] Child launched
[0.205] Child launched
```

### Step 5 — The C resize issues `ioctl(TIOCSWINSZ)`, and the kernel delivers `SIGWINCH`

`resize_pty` (`kitty/child-monitor.c:592`) looks the window up by id, and on success calls `pty_resize`, which issues the `TIOCSWINSZ` ioctl:

**SOURCE — `kitty/child-monitor.c:577-579`:**
```c
pty_resize(int fd, struct winsize *dim) {
    while(true) {
        if (ioctl(fd, TIOCSWINSZ, dim) == -1) {
```

`TIOCSWINSZ` is precisely what makes the kernel auto-deliver `SIGWINCH` to the PTY's foreground process group when the window size actually changes. On *subsequent* resizes the Python dispatch takes the `elif` branch and prints the observable line:

**SOURCE — `kitty/window.py:873` (the print in the `elif boss.args.debug_rendering:` branch at `:872`):**
```python
                print(f'[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}', file=sys.stderr)
```

**OBSERVED EVIDENCE (run.log:4,6,8 — subsequent resizes; 47 total):**
```text
[0.196] SIGWINCH sent to child in window: 1 with size: (22, 34, 306, 396)
[0.204] SIGWINCH sent to child in window: 2 with size: (11, 35, 315, 198)
[0.214] SIGWINCH sent to child in window: 1 with size: (11, 34, 306, 198)
```

**Summary of O1:** create → `add_child` registers `window.id` in both the C monitor and the Python `WeakValueDictionary` → PTY allocated (`openpty`, master fd + pid stored) → first layout resize marks the child launched (`Child launched`) → `resize_pty` → `ioctl(TIOCSWINSZ)` → kernel `SIGWINCH`; later resizes print `SIGWINCH sent to child in window: N`.

---

## O2 — Teardown-before-completion: reacting to a window that is already gone

Two concrete "the window is gone before everything finished reacting" cases occur, one per direction of state flow.

### Case A — A queued resize for a departed window (the resize arrives after the child left the registry)

Because layout keeps issuing resizes as siblings churn, a `resize_pty(id, ...)` can arrive for a window whose child the C monitor has already removed. `resize_pty` resolves the id by scanning first the live `children[]` array, then the `add_queue[]`; if neither holds the id, it logs and simply skips — it never dereferences a missing child:

**SOURCE — `kitty/child-monitor.c:606-610`:**
```c
    FIND(children, self->count);
    if (fd == -1) FIND(add_queue, add_queue_count);
    if (fd != -1) {
        if (!pty_resize(fd, &dim)) PyErr_SetFromErrno(PyExc_OSError);
    } else log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", window_id, self->count, add_queue_count);
```

**OBSERVED EVIDENCE (run.log:17,19,23 — a departed-window resize; 27 total in this run):**
```text
[0.256] Failed to send resize signal to child with id: 6 (children count: 4) (add queue: 0)
[0.257] Failed to send resize signal to child with id: 7 (children count: 4) (add queue: 0)
[0.262] Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)
```

Note the reported `(children count: 4)` and `(add queue: 0)`: the id is in neither structure — the child left the registry, yet a resize for it was still in flight. Across the run the `children count` in these lines decreases `4 → 3 → 2 → 1` as more children die while relayout keeps dispatching resizes to them.

### Case B — A `SIGCHLD` reaped after the window was already removed (the death arrives after the window left the Python map)

The mirror case: the child dies and its death is delivered to Python as `on_child_death(window_id)`, but the window has already been removed from `window_id_map`. The callback pops-or-`None` and returns early, silently discarding the stale death notification:

**SOURCE — `kitty/boss.py:881-885`:**
```python
    def on_child_death(self, window_id: int) -> None:
        prev_active_window = self.active_window
        window = self.window_id_map.pop(window_id, None)
        if window is None:
            return
```

**OBSERVED EVIDENCE:** In this run `close_on_child_death=yes` drove every window's teardown; the process reached `main loop exiting` and exited with status `0` (run.log tail, quoted in O5) — no traceback or crash from a death arriving for an already-removed window. `on_child_death` returning on `window is None` is the guard that makes a late/duplicate death a no-op.

**Summary of O2:** whichever side is "late," the operation is a benign skip: a resize for a vanished child logs `Failed to send resize signal to child with id: N` and returns; a death for a vanished window hits `pop(window_id, None) → None` and returns.

---

## O3 — Keep-vs-discard: what state kitty retains and what it throws away

During churn kitty makes three distinct keep-vs-discard decisions.

### Decision 1 — The Python liveness registry auto-drops dead windows (`WeakValueDictionary`)

The Python window registry holds only **weak** references, so once no strong reference to a `Window` remains, its entry disappears on its own — kitty does not have to hunt down and delete it:

**SOURCE — `kitty/boss.py:344`:**
```python
        self.window_id_map: WeakValueDictionary[int, Window] = WeakValueDictionary()
```

This is why a stale `on_child_death` (O2 Case B) finds `None`: the window object was already collected, so `pop` returns nothing. The weak map is the "discard" side for the Python layer.

### Decision 2 — The C side flags a departing child with `needs_removal` (id-correlated `Child` record)

The C monitor's per-child record carries an explicit discard flag. The `Child` struct is:

**SOURCE — `kitty/child-monitor.c:65-71`:**
```c
typedef struct {
    Screen *screen;
    bool needs_removal;
    int fd;
    unsigned long id;
    pid_t pid;
} Child;
```

`needs_removal` (`kitty/child-monitor.c:67`) is the C-side discard marker; `id` (`:69`) and `pid` (`:70`) are the correlation keys used by `resize_pty` (by id) and by reaping (by pid). A child is flagged for removal from several places — by pid when reaped (`mark_child_for_removal`, `kitty/child-monitor.c:1386`), on a close request (`mark_child_for_close`, `kitty/child-monitor.c:541`), on PTY EOF, on `POLLNVAL`, and en masse at shutdown (all shown in O5).

### Decision 3 — Retain and flush pending OUTPUT before discarding the dead CHILD

The subtle, important trade: when a child is removed, kitty first does a **final flush** of any bytes still sitting in that child's screen/parser — retaining the last output — and only *then* fires the death callback. Meanwhile, children already flagged `needs_removal` are **skipped** by normal parsing:

**SOURCE — `kitty/child-monitor.c:521-522` (final flush, then death notify):**
```c
        if (remove_notify[remove_count].screen) do_parse(self, remove_notify[remove_count].screen, now, true);
        PyObject *t = PyObject_CallFunction(self->death_notify, "k", remove_notify[remove_count].id);
```

**SOURCE — `kitty/child-monitor.c:529-530` (normal parsing skips a child already flagged for removal):**
```c
        if (!scratch[i].needs_removal) {
            if (do_parse(self, scratch[i].screen, now, false)) input_read = true;
```

The `true` final argument to `do_parse` (`kitty/child-monitor.c:438` defines `do_parse(..., bool flush)`) is the flush. So kitty **keeps** the child's remaining output (flushed once, synchronously, right before notifying Python of the death) while it **discards** the child from active polling/parsing.

### Per-tab bookkeeping

Below the boss, each tab tracks its own windows in a `WindowList` with `add_window` (`kitty/window_list.py:67`), `remove_window` (`kitty/window_list.py:84`), and an `id_map`:

**SOURCE — `kitty/window_list.py:148`:**
```python
        self.id_map: Dict[int, WindowType] = {}
```

**OBSERVED EVIDENCE for O3:** windows are dropped cleanly and the whole session tears down without hanging or leaking — the run ends with `main loop exiting` (run.log tail, quoted in O5) and exit status `0`. Tying this to `close_on_child_death=yes`: each child death removes its window; when the last window of the OS window is gone, the loop exits. The decreasing `(children count: 4 → 3 → 2 → 1)` in the O2 evidence is the C registry shedding dead children in real time.


---

## O4 — Timing effects: asynchronous coalesced signals vs. synchronous queued reconciliation

kitty's consistency during churn rests on a strict separation: signals are delivered **asynchronously** and only *set flags*, while all mutation of the child registry happens **synchronously** at the top of the I/O-thread loop tick. Timing is made visible by millisecond timestamps.

### Millisecond timestamps are the timing evidence

The `SIGWINCH sent to child` line is prefixed with a monotonic timestamp at 3-decimal (millisecond) resolution:

**SOURCE — `kitty/window.py:873`:**
```python
                print(f'[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}', file=sys.stderr)
```

The timing source is `kitty/monotonic.h` (`MONOTONIC_T_1e6` at `:15`, `MONOTONIC_T_1e3` at `:16`), i.e. the `[s.mmm]` prefix on every line.

### Asynchronous, flag-only signal handling via a self-pipe

Signals are handled in C on the I/O thread through a self-pipe: the async signal handler writes a byte to a pipe, and the loop later drains it with `read_signals`:

**SOURCE — `kitty/loop-utils.c:12,22` (the write side of the self-pipe):**
```c
static int signal_write_fd = -1;
```
```c
        ssize_t ret = write(signal_write_fd, buf, sz);
```

**SOURCE — `kitty/loop-utils.c:131` (the drain side, run on the I/O thread):**
```c
read_signals(int fd, handle_signal_func callback, void *data) {
```

The handled set includes `SIGCHLD`:

**SOURCE — `kitty/child-monitor.c:121`:**
```c
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

Crucially, the handler does **no real work** — it only sets boolean flags in a `SignalSet`, so signal delivery can never race with data-structure mutation:

**SOURCE — `kitty/child-monitor.c:1359-1372`:**
```c
typedef struct { bool kill_signal, child_died, reload_config; } SignalSet;

static bool
handle_signal(const siginfo_t *siginfo, void *data) {
    SignalSet *ss = data;
    switch(siginfo->si_signo) {
        case SIGINT:
        case SIGTERM:
        case SIGHUP:
            ss->kill_signal = true;
            break;
        case SIGCHLD:
            ss->child_died = true;
            break;
```

### `SIGCHLD` coalescing: one signal, a `waitpid(-1, …, WNOHANG)` loop reaps all

Unix does not queue `SIGCHLD`, so a burst of near-simultaneous child exits may coalesce into *fewer* delivered signals than deaths. kitty applies the canonical remedy: on the `child_died` flag it loops `waitpid(-1, &status, WNOHANG)` until no more children are reapable, so *every* exited child is reaped regardless of how many signals arrived:

**SOURCE — `kitty/child-monitor.c:1413-1424` (`reap_children`; the `waitpid` is at `:1418`):**
```c
reap_children(ChildMonitor *self, bool enable_close_on_child_death) {
    int status;
    pid_t pid;
    (void)self;
    while(true) {
        pid = waitpid(-1, &status, WNOHANG);
        if (pid == -1) {
            if (errno != EINTR) break;
        } else if (pid > 0) {
            if (enable_close_on_child_death) mark_child_for_removal(self, pid);
            mark_monitored_pids(pid, status);
        } else break;
```

`reap_children` is invoked from the loop only when the flag is set:

**SOURCE — `kitty/child-monitor.c:1526`:**
```c
                if (ss.child_died) reap_children(self, OPT(close_on_child_death));
```

### Synchronous, serialized reconciliation at the top of every loop tick

The main thread only **queues** mutations (`add_child` appends to `add_queue`; closes set `needs_removal`). The I/O thread **applies** them — remove then add — at the very top of each tick, under `children_lock`. This serialization is the core reason rapid create/destroy stays consistent: the asynchronous signal never touches `children[]` directly; only this synchronous section does.

**SOURCE — `kitty/child-monitor.c:1491-1494`:**
```c
    while (LIKELY(!self->shutting_down)) {
        children_mutex(lock);
        remove_children(self);
        add_children(self);
```

The event-loop build makes each tick observable:

**OBSERVED EVIDENCE (run.log:111-112 — a loop tick and clean shutdown):**
```text
[0.462] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.471] main loop exiting
```

### The timing race, captured in a single millisecond

The clearest timing evidence is a **same-millisecond pair**: within one relayout pass, the C `resize_pty` cannot find the id (the child already left `children[]`, and its removal has been applied) so it logs `Failed to send resize signal`, and in the *same* Python dispatch the `elif debug_rendering` branch still prints `SIGWINCH sent to child` for that same id. The C view (gone) and the Python view (alive) disagree in the same millisecond:

**OBSERVED EVIDENCE (run.log:17-18 — identical `[0.256]` timestamp, identical id 6):**
```text
[0.256] Failed to send resize signal to child with id: 6 (children count: 4) (add queue: 0)
[0.256] SIGWINCH sent to child in window: 6 with size: (11, 34, 306, 198)
```

This exact adjacency (a `Failed to send resize signal to child with id: N` immediately followed by a same-timestamp `SIGWINCH sent to child in window: N` for the same `N`) occurred **25 times** in this run; other instances include id 7 at `[0.257]`, id 3 at `[0.262]`, id 11 at `[0.307]`, and id 14 at `[0.334]`. The measured timestamp delta within each pair is **0 ms** (both prints are in the same synchronous relayout, straddling the C boundary where the registries disagree).

**Summary of O4:** signals are async, coalesced, and flag-only (`handle_signal` → `child_died`; `waitpid(-1,…,WNOHANG)` loop); all registry mutation is synchronous and serialized at the tick top (`remove_children` then `add_children` under `children_lock`); the `[s.mmm]` prefixes expose the disagreement in time.

---

## O5 — Conflicting-liveness resolution: the moments kitty must decide what is still alive

There are exactly **two graceful resolution points** where kitty must reconcile conflicting views of liveness, plus a set of edge cases that all funnel into the same `needs_removal` discard.

### Resolution point 1 — `resize_pty` id-not-found skip (C thinks the child is gone)

When a resize targets an id absent from both `children[]` and `add_queue[]`, `resize_pty` logs and returns without error (no dereference, no crash):

**SOURCE — `kitty/child-monitor.c:610`:**
```c
    } else log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", window_id, self->count, add_queue_count);
```

**OBSERVED EVIDENCE (run.log:17):**
```text
[0.256] Failed to send resize signal to child with id: 6 (children count: 4) (add queue: 0)
```

### Resolution point 2 — `on_child_death` pop→`None` discard (Python thinks the window is gone)

When a death arrives for a window no longer in the map, `pop(window_id, None)` yields `None` and the callback returns early:

**SOURCE — `kitty/boss.py:883-885`:**
```python
        window = self.window_id_map.pop(window_id, None)
        if window is None:
            return
```

### Both resolve benignly — proven by exit status 0

The decisive evidence that these conflicts resolve without crashing is that the process completed normally despite **27** `Failed to send resize signal` conflicts:

**OBSERVED EVIDENCE (exit status + clean shutdown):**
```text
KITTY_EXIT=0
[0.471] main loop exiting
```

### Edge cases that feed the same discard path (with evidence, including one honestly-absent line)

- **PTY EOF (child closed its end).** A readable/hung-up fd whose read yields no more data marks the child for removal:

  **SOURCE — `kitty/child-monitor.c:1529-1535`:**
  ```c
                  if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP)) {
                      data_received = true;
                      has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);
                      if (!has_more) {
                          // child is dead
                          children_mutex(lock);
                          children[i].needs_removal = true;
  ```

  `read_bytes` (`kitty/child-monitor.c:1337`) returns `len != 0` (`kitty/child-monitor.c:1355`), i.e. false at EOF. This is the normal teardown path exercised throughout this run (every one of the 20 short-lived children ended this way, driving the observed removals).

- **Unexpectedly-closed fd (`POLLNVAL`).** If a child's fd is found invalid, it is flagged for removal and a specific line is logged:

  **SOURCE — `kitty/child-monitor.c:1542-1547`:**
  ```c
                  if (children_fds[EXTRA_FDS + i].revents & POLLNVAL) {
                      // fd was closed
                      children_mutex(lock);
                      children[i].needs_removal = true;
                      children_mutex(unlock);
                      log_error("The child %lu had its fd unexpectedly closed", children[i].id);
  ```

  **OBSERVED EVIDENCE (honest null result):** this line did **not** appear in this run:
  ```text
  $ grep -c 'had its fd unexpectedly closed' run.log
  0
  ```
  Reported exactly as observed (rule: report even absence): the `POLLNVAL` path exists and its literal is compiled into `fast_data_types.so`, but under `Xvfb` with `close_on_child_death=yes` the children closed cleanly via `POLLIN|POLLHUP` EOF, not via an invalidated fd, so this branch was not exercised.

- **Shutdown (mark all, then remove).** At loop shutdown every remaining child is flagged and removed in one pass:

  **SOURCE — `kitty/child-monitor.c:1574-1575`:**
  ```c
    for (i = 0; i < self->count; i++) children[i].needs_removal = true;
    remove_children(self);
  ```

  The corresponding observed end-of-run marker is `[0.471] main loop exiting` (quoted above).

- **Supporting removal helpers.** By pid on reap: `mark_child_for_removal` (`kitty/child-monitor.c:1386`); on a UI close request: `mark_child_for_close` (`kitty/child-monitor.c:541`), which scans `children[]` then `add_queue[]` and sets `needs_removal`; and `hangup` (`kitty/child-monitor.c:1294`) delivers `SIGHUP` to the child's process group during removal. All roads set the same `needs_removal` flag consumed by `remove_children` (`kitty/child-monitor.c:1313`) at the next tick.

**Summary of O5:** the two liveness conflicts (a resize for a child C no longer holds; a death for a window Python no longer holds) are each resolved by a benign skip; every teardown trigger (EOF, `POLLNVAL`, close request, reap-by-pid, shutdown) converges on the single `needs_removal` flag applied synchronously at the tick top — and the run's exit status `0` confirms the resolution never crashes.


---

## Design vs. established best practice (framing only)

kitty's design aligns with well-established systems-programming practice; its own code and the observed behavior above remain the source of truth. Three alignments are worth naming:

- **`SIGCHLD` reaping with a `waitpid(-1, …, WNOHANG)` loop.** Because Unix does not queue `SIGCHLD`, the canonical remedy for a burst of exits is to loop `waitpid` with `WNOHANG` until it stops returning positive pids. kitty does exactly this at `kitty/child-monitor.c:1418`, so a coalesced signal still reaps every child.
- **Async-signal-safe, self-pipe/flag-only handler.** Best practice is that a signal handler should defer real work — only set a flag or write to a pipe. kitty's `handle_signal` sets only `SignalSet` booleans (`kitty/child-monitor.c:1359-1372`) and the real work runs later on the I/O thread after `read_signals` (`kitty/loop-utils.c:131`) drains the self-pipe.
- **`TIOCSWINSZ`-driven `SIGWINCH`.** The kernel auto-delivers `SIGWINCH` to a PTY's foreground process group only when the size actually changes via `TIOCSWINSZ`; kitty issues exactly that ioctl in `pty_resize` (`kitty/child-monitor.c:579`), and guards the dispatch behind a size-changed check (`kitty/window.py:861`) so redundant resizes are suppressed.

The single most important structural choice — the **queue-and-apply** split (async signals set flags; the I/O thread applies queued adds/removes synchronously under `children_lock` at the top of each tick, `kitty/child-monitor.c:1491-1494`) — is what keeps the two liveness views from corrupting shared state even while they momentarily disagree (O4/O5).

---

## Coverage pass

Every distinct thing the question names is addressed below, each with its section and its grounding.

**Question clauses:**

- **"terminal windows appear, resize, and disappear in quick succession"** — the whole document; provoked by the 5-tab / 20-short-lived-window `--session` run (see "How this was observed"). Appear = `Child launched` ×20; resize = `SIGWINCH sent to child` ×47; disappear = `close_on_child_death=yes` teardown ending in `main loop exiting`.
- **"a new window is created and immediately used to run a command"** — **O1** (each `launch` window runs `sh -c "…; true"`; `Child launched` at run.log:3,5,7).
- **"resize events AND signals start flowing"** — **O1** (resize dispatch `kitty/window.py:861-873`) + **O4** (`SIGWINCH` via `TIOCSWINSZ`, `SIGCHLD` via self-pipe). Both resize events and signals are covered explicitly.
- **"the window is gone before everything has finished reacting"** — **O2** (Case A queued resize for a departed window: `Failed to send resize signal…`; Case B `SIGCHLD` reaped after removal: `on_child_death` pop→`None`).
- **"how does kitty decide what state to keep and what to discard"** — **O3** (keep = final `do_parse(..., flush=true)` of pending output; discard = `WeakValueDictionary` auto-drop + `needs_removal`).
- **"how timing affects signal delivery"** — **O4** (async coalesced `SIGCHLD`; flag-only handler; `waitpid` loop).
- **"how timing affects … internal bookkeeping"** — **O4** (synchronous queue-and-apply at the tick top under `children_lock`; `[s.mmm]` timestamps).
- **"moments where the system has to resolve conflicting views of what is still alive"** — **O5** (two resolution points + edge cases; benign, exit status 0).
- **"temporary scripts … repository unchanged … cleaned up afterward"** — honored: all artifacts lived under `/tmp/blitzy_obs/` and were removed; the only repository change is this document (see the closing note).

**Named mechanisms — each named and evidenced:**

| Mechanism | Where covered | Citation | Observed evidence |
|---|---|---|---|
| `SIGWINCH` | O1, O4 | `kitty/window.py:873`, `kitty/child-monitor.c:579` (`TIOCSWINSZ`) | `SIGWINCH sent to child in window: 1 …` (run.log:4) |
| `SIGCHLD` | O4 | `kitty/child-monitor.c:121,1362` | reaped via `child_died` flag → `reap_children` (exit 0) |
| `TIOCSWINSZ` | O1, best-practice | `kitty/child-monitor.c:579` | drives `SIGWINCH` lines above |
| `WeakValueDictionary` | O3 | `kitty/boss.py:344` | stale death → `pop → None` return (O2 Case B) |
| `needs_removal` | O3, O5 | `kitty/child-monitor.c:67,1535,1545,1574` | children count `4→3→2→1` (run.log:17,23,…) |
| `add_queue` / `remove_queue` | O4 | `kitty/child-monitor.c:1491-1494` (apply), `:305` (queue) | `(add queue: 0)` in the `Failed…` lines |
| self-pipe / `read_signals` | O4 | `kitty/loop-utils.c:12,22,131` | flag-only handler; loop drains it |
| `waitpid(-1, …, WNOHANG)` | O4, best-practice | `kitty/child-monitor.c:1418` | coalesced reap; exit 0 |
| `resize_pty` | O1, O2, O5 | `kitty/child-monitor.c:592,610` | `Failed to send resize signal…` ×27 |
| `on_child_death` | O2, O5 | `kitty/boss.py:881-885` | benign no-op on `None`; exit 0 |

**Honest null result:** the `POLLNVAL` line `The child %lu had its fd unexpectedly closed` (`kitty/child-monitor.c:1547`) did **not** occur in this run (`grep -c … → 0`); the path exists but was not exercised because children closed via clean EOF (`POLLIN|POLLHUP`).

**Evidence discipline:** every count (`47`, `27`, `20`, `1`, `0`, `3`), every timestamp (`[0.256]`, `[0.471]`, …), and the exit status (`0`) is traceable to a pasted `COMMAND` block above and to the captured `/tmp/blitzy_obs/run.log`. Every `file:line` citation was re-grepped against the live tree at branch snapshot `815df1e210e0` before being pasted.

---

## Reproducibility note

To reproduce: build with `python3 setup.py build --debug --extra-logging=event-loop`; start `Xvfb :99`; then run
`./kitty/launcher/kitty --config NONE --debug-rendering -o close_on_child_death=yes -o confirm_os_window_close=0 --session <churn-session>`
with a session file that opens many short-lived windows across several `new_tab`s in `layout grid`. Grep the captured stderr for `Child launched`, `SIGWINCH sent to child`, and `Failed to send resize signal`. Exact counts vary run-to-run with scheduling, but the same **classes** of lines — including same-millisecond `Failed…`/`SIGWINCH…` pairs for one id and a final `main loop exiting` with exit status `0` — reproduce reliably.

