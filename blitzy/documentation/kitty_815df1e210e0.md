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

**COMMAND (exact build wrapper run, capturing the exit status):**
```bash
python3 setup.py build --debug --extra-logging=event-loop > /tmp/blitzy_obs/build.log 2>&1
echo "BUILD_EXIT=$?"
```

**OUTPUT (the complete `/tmp/blitzy_obs/build.log`, 11 lines, followed by the captured exit status):**
```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
BUILD_EXIT=0
```

The trailing `[1/1] Compiling … done` / `[1/1] Linking kitty/fast_data_types … done` lines plus `BUILD_EXIT=0` are the proof the build completed successfully. This invocation was *incremental* (the extension had been built once already, so only `kitty/data-types.c` was recompiled); the two independent checks below prove the event-loop/signal logging is compiled into the extension that is actually loaded at run time.

That `kitty/child-monitor.c` is compiled with the event-loop/debug defines was read directly from the build database:

**COMMAND:**
```bash
python3 - <<'PY'
import json
d = json.load(open('build/compile_commands.json'))
for e in d:
    if e['file'].endswith('child-monitor.c'):
        cmd = e.get('command') or ' '.join(e.get('arguments', []))
        print(' '.join(sorted({t for t in cmd.split() if 'DEBUG' in t})))
        break
PY
```

**OUTPUT:**
```text
-DDEBUG -DDEBUG_EVENT_LOOP -DKITTY_DEBUG_BUILD
```

The build produces the C extension `kitty/fast_data_types.so` (plus `kitty/glfw-x11.so`, `kitty/launcher/kitty`, and `kitty/launcher/kitten`). The exact log literals are present in the compiled `.so`:

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

**COMMAND (start the headless X server first):**
```bash
Xvfb :99 -screen 0 1280x800x24 -nolisten tcp > /tmp/blitzy_obs/xvfb.log 2>&1 &
```

**COMMAND (exact run command as executed — `timeout 90` was used as a safety net so the run cannot hang the session):**
```bash
export DISPLAY=:99 TERM=xterm-kitty LANG=C.UTF-8 LC_ALL=C.UTF-8
timeout 90 ./kitty/launcher/kitty --config NONE --debug-rendering -o close_on_child_death=yes \
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
LOG=/tmp/blitzy_obs/run.log
echo "SIGWINCH sent to child : $(grep -c 'SIGWINCH sent to child' "$LOG")"
echo "Failed to send resize  : $(grep -c 'Failed to send resize signal' "$LOG")"
echo "Child launched         : $(grep -c 'Child launched' "$LOG")"
echo "OS Window created      : $(grep -c 'OS Window created' "$LOG")"
echo "fd unexpectedly closed : $(grep -c 'had its fd unexpectedly closed' "$LOG")"
echo "loop tick              : $(grep -c 'loop tick, wakeups_happened' "$LOG")"
echo "main loop exiting      : $(grep -c 'main loop exiting' "$LOG")"
```

**OUTPUT:**
```text
SIGWINCH sent to child : 48
Failed to send resize  : 21
Child launched         : 20
OS Window created      : 1
fd unexpectedly closed : 0
loop tick              : 4
main loop exiting      : 1
```

Interpreting these counts (this interpretation is grounded in `kitty/window.py:861-873`, quoted in O1): `Child launched` is printed **once per window, on that window's first resize only** — the observed **20** exactly matches the 20 launched windows. `SIGWINCH sent to child` is printed on **subsequent** resizes (the two prints are mutually exclusive per resize), so the observed **48** are later resizes triggered by relayout as siblings appear and disappear. `Failed to send resize signal` (**21**) is the race being provoked: a `resize_pty` for a window id that the C child registry no longer holds. `fd unexpectedly closed` was **0** — that `POLLNVAL` edge path (which exists in the code) did not trigger in this run; this is reported honestly and explained in O5.

### Environment caveats (required disclosures)

- **No display → headless `Xvfb`.** OpenGL is provided by Mesa software rasterization, recorded in the run log:

  **COMMAND:**
  ```bash
  grep -n 'GL version string' /tmp/blitzy_obs/run.log
  ```
  **OUTPUT:**
  ```text
  113:[0.136] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
  ```
- **Wayland backend disabled.** The build prints `Disabling building of wayland backend` (quoted above) because `wayland-protocols >= 1.17` is absent; the run is therefore X11-only.
- **Remote control not used.** Churn is driven by a startup `--session` file rather than by `kitty @` remote control. (Note: contrary to some environment notes, the Go `kitten` binary *was* built here at `kitty/launcher/kitten`; remote control was nonetheless deliberately avoided in favor of the deterministic session file.)
- **Python version.** The build and run succeed on this environment's **Python 3.13.7**. For reference, `pyproject.toml:2` declares `requires-python = ">=3.8"`, and the CI matrix in `.github/workflows/ci.yml` builds/tests on `"3.8"` (`:26`), `"3.9"` (`:34`), and `"3.10"` (`:30`), with docs/lint on `"3.11"` (`:85`).
- **One harmless startup line.** The run prints a single systemd-bus warning:

  **COMMAND:**
  ```bash
  grep -n 'Failed to open systemd user bus' /tmp/blitzy_obs/run.log
  ```
  **OUTPUT:**
  ```text
  2:[0.169] Failed to open systemd user bus with error: Connection refused
  ```
  This is expected under `Xvfb` (no systemd session bus) and is unrelated to the window/signal machinery.

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

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` lines 3,5,7 — `Child launched` printed once per window on first resize; 20 total == 20 windows):**
```text
[0.171] Child launched
[0.176] Child launched
[0.183] Child launched
```

The producing command (line numbers + content, so the reference is self-contained):
```bash
grep -n 'Child launched' /tmp/blitzy_obs/run.log | head -3
```
```text
3:[0.171] Child launched
5:[0.176] Child launched
7:[0.183] Child launched
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

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` lines 4,6,8 — subsequent resizes; 48 total):**
```bash
grep -n 'SIGWINCH sent to child' /tmp/blitzy_obs/run.log | head -3
```
```text
4:[0.176] SIGWINCH sent to child in window: 1 with size: (22, 34, 306, 396)
6:[0.182] SIGWINCH sent to child in window: 2 with size: (11, 35, 315, 198)
8:[0.191] SIGWINCH sent to child in window: 1 with size: (11, 34, 306, 198)
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

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` lines 21,38,45 — a departed-window resize; 21 total in this run):**
```bash
grep -n 'Failed to send resize signal' /tmp/blitzy_obs/run.log | head -3
```
```text
21:[0.231] Failed to send resize signal to child with id: 3 (children count: 6) (add queue: 0)
38:[0.301] Failed to send resize signal to child with id: 14 (children count: 5) (add queue: 0)
45:[0.328] Failed to send resize signal to child with id: 18 (children count: 7) (add queue: 0)
```

Note the reported `(children count: N)` and `(add queue: 0)`: the id is in neither structure — the child left the registry, yet a resize for it was still in flight. The `children count` field is the size of the live `children[]` array *at the instant each failed resize was logged*. The full sequence of those counts across the run's 21 failed-resize lines, in log order, is produced by:

**COMMAND:**
```bash
grep -oE 'children count: [0-9]+' /tmp/blitzy_obs/run.log | grep -oE '[0-9]+' | tr '\n' ' '; echo
```

**OUTPUT:**
```text
6 5 7 6 5 5 5 5 5 5 5 5 4 4 4 4 4 4 4 1 1
```

The distinct values observed are **7, 6, 5, 4, 1**. Note the sequence is *not* monotonic (it rises `6 → 5 → 7` early on): because layout dispatches resizes while children are still being **added** (later tabs filling) *and* being **removed** (early windows already dying) concurrently, the live count both grows and shrinks during the burst. By the end it collapses to **1** as the registry sheds the last dying children while relayout keeps dispatching resizes to them.

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

**Is this `pop → None` branch actually taken in the churn run?** This is a *source-verified conflict path*, and to report honestly I instrumented it directly rather than inferring it from exit status. Using a temporary `usercustomize.py` (injected via `PYTHONPATH`, outside the repository) I wrapped `Boss.on_child_death` to record, for every death, whether the window id was still present in `window_id_map` *before* the `pop` (present) or already gone (would make `pop` return `None`, i.e. absent):

**COMMAND:**
```bash
# usercustomize.py wraps Boss.on_child_death: present = (window_id in self.window_id_map) before pop
timeout 90 env PYTHONPATH=/tmp/blitzy_obs/inject ./kitty/launcher/kitty --config NONE \
    --debug-rendering -o close_on_child_death=yes -o confirm_os_window_close=0 \
    --session /tmp/blitzy_obs/session.conf > /tmp/blitzy_obs/run_instrumented.log 2>&1
echo "on_child_death calls    : $(grep -cF '[OBS] on_child_death' /tmp/blitzy_obs/run_instrumented.log)"
echo "PRESENT                 : $(grep -F '[OBS] on_child_death' /tmp/blitzy_obs/run_instrumented.log | grep -c PRESENT)"
echo "ABSENT (pop returns None): $(grep -F '[OBS] on_child_death' /tmp/blitzy_obs/run_instrumented.log | grep -c ABSENT)"
```

**OUTPUT:**
```text
on_child_death calls    : 20
PRESENT                 : 20
ABSENT (pop returns None): 0
```

Representative per-death records (verbatim `[OBS]` lines; `grep -oE '\[OBS\] on_child_death.*'` strips interleaved stderr):

**OUTPUT:**
```text
[OBS-INJECT] patched Boss.on_child_death OK
[OBS] on_child_death(window_id=3) PRESENT calls=1 present=1 absent=0
[OBS] on_child_death(window_id=8) PRESENT calls=10 present=10 absent=0
[OBS] on_child_death(window_id=20) PRESENT calls=20 present=20 absent=0
```

So in **this** run all **20** deaths found the window still present, and `pop` returned the window every time — the `pop → None` branch was **not directly exercised** by this churn (`ABSENT = 0`). That is the honest, observed result: the discard branch is a **source-verified conflict path** that guards against a death arriving for an already-removed window (a double-notify or a manual close racing the reap), but the specific ordering that yields `None` did not occur in the captured run. What the run *does* confirm is that the surrounding teardown is benign: the process reached `main loop exiting` and exited with status `0` (quoted in O5), with no traceback from any death/removal interleaving.

**Summary of O2:** whichever side is "late," the operation is a benign skip: a resize for a vanished child logs `Failed to send resize signal to child with id: N` and returns (Case A, directly observed **21×**); a death for a vanished window would hit `pop(window_id, None) → None` and return (Case B, a source-verified guard; in this run `on_child_death` fired **20×**, all with the window still **present**, so the `None` branch was not taken).

---

## O3 — Keep-vs-discard: what state kitty retains and what it throws away

During churn kitty makes three distinct keep-vs-discard decisions.

### Decision 1 — The Python liveness registry auto-drops dead windows (`WeakValueDictionary`)

The Python window registry holds only **weak** references, so once no strong reference to a `Window` remains, its entry disappears on its own — kitty does not have to hunt down and delete it:

**SOURCE — `kitty/boss.py:344`:**
```python
        self.window_id_map: WeakValueDictionary[int, Window] = WeakValueDictionary()
```

This is why a stale `on_child_death` (O2 Case B) *would* find `None`: if the window object was already collected, `pop` returns nothing. The weak map is the "discard" side for the Python layer. (As the O2 Case B instrumentation shows, this `None` branch is a source-verified guard that was not itself triggered in the captured run — every one of the 20 deaths found the window still present.)

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

**OBSERVED EVIDENCE for O3:** windows are dropped cleanly and the whole session tears down without hanging or crashing — the run ends with `main loop exiting` and exit status `0` (both quoted in O4/O5). Tying this to `close_on_child_death=yes`: each child death removes its window; when the last window of the OS window is gone, the loop exits. The `children count` values in the O2 evidence (produced above by the `children count: N` extraction command: `6 5 7 6 5 5 5 5 5 5 5 5 4 4 4 4 4 4 4 1 1`, distinct **7, 6, 5, 4, 1**) are the C registry shedding dead children in real time.


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

### Asynchronous signal arrival, flag-only handling — via `signalfd` on Linux (self-pipe is the non-Linux fallback)

Signals arrive **asynchronously** (a child can exit at any instant), but kitty never services them inside an async signal handler that touches shared state. *How* the asynchronous arrival is funneled onto the I/O thread is decided at **compile time** by whether `<sys/signalfd.h>` exists:

**SOURCE — `kitty/loop-utils.h:14-18` (the platform switch that defines `HAS_SIGNAL_FD`):**
```c
#ifdef __has_include
#if __has_include(<sys/signalfd.h>)
#define HAS_SIGNAL_FD
#include <sys/signalfd.h>
#endif
```

On **Linux — which is this run's platform — `HAS_SIGNAL_FD` is defined**, so kitty does *not* install an async `sigaction` handler at all. Instead it **blocks** the handled signals process-wide with `sigprocmask(SIG_BLOCK, …)` and reads them synchronously from a pollable `signalfd` file descriptor:

**SOURCE — `kitty/loop-utils.c:39-42` (the `signalfd` branch — the path taken on Linux):**
```c
#ifdef HAS_SIGNAL_FD
    if (ld->num_handled_signals) {
        if (sigprocmask(SIG_BLOCK, &ld->signals, NULL) == -1) return false;
        ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC);
```

Only on platforms **without** `signalfd` (the `#else` branch — e.g. macOS/BSD) does kitty fall back to the classic **self-pipe trick**: an async `sigaction` handler writes the signal to a pipe whose read end the loop drains. This branch is *not* compiled on Linux:

**SOURCE — `kitty/loop-utils.c:45-54` (the `#else` self-pipe fallback):**
```c
#else
    ld->signal_fds[0] = -1; ld->signal_fds[1] = -1;
    if (ld->num_handled_signals) {
        if (!self_pipe(ld->signal_fds, true)) return false;
        signal_write_fd = ld->signal_fds[1];
        ld->signal_read_fd = ld->signal_fds[0];
        struct sigaction act = {.sa_sigaction=handle_signal, .sa_flags=SA_SIGINFO | SA_RESTART, .sa_mask = ld->signals};
        for (size_t i = 0; i < ld->num_handled_signals; i++) { if (sigaction(ld->handled_signals[i], &act, NULL) != 0) return false; }
    }
#endif
```

Either way the loop drains the read fd with `read_signals`; on Linux the `HAS_SIGNAL_FD` branch decodes `struct signalfd_siginfo` records:

**SOURCE — `kitty/loop-utils.c:130-133`:**
```c
void
read_signals(int fd, handle_signal_func callback, void *data) {
#ifdef HAS_SIGNAL_FD
    static struct signalfd_siginfo fdsi[32];
```

I confirmed the `signalfd` branch is the one actually compiled and running here with **two independent checks**.

**(1)** The preprocessor decision, using the same guard as `loop-utils.h`:

**COMMAND:**
```bash
cat > /tmp/blitzy_obs/probe_signalfd.c <<'C'
#include <stdio.h>
#ifdef __has_include
#  if __has_include(<sys/signalfd.h>)
#    define HAS_SIGNAL_FD
#  endif
#else
#  define HAS_SIGNAL_FD
#endif
int main(void){
#ifdef HAS_SIGNAL_FD
    printf("HAS_SIGNAL_FD=1\n");
#else
    printf("HAS_SIGNAL_FD=0\n");
#endif
    return 0; }
C
cc /tmp/blitzy_obs/probe_signalfd.c -o /tmp/blitzy_obs/probe_signalfd && /tmp/blitzy_obs/probe_signalfd
```

**OUTPUT:**
```text
HAS_SIGNAL_FD=1
```

**(2)** The *running* emulator actually holds a `signalfd`. With one long-lived window keeping kitty alive, its `/proc/$PID/fd` (the command below resolves `$PID` via `pgrep`) shows a `signalfd` descriptor and the descriptor's `fdinfo` reveals the exact watched-signal mask:

**COMMAND:**
```bash
PID=$(pgrep -f 'launcher/kitty --config NONE' | head -1)
ls -l /proc/$PID/fd | grep -i signalfd
for f in /proc/$PID/fdinfo/*; do grep -q '^sigmask:' "$f" && { echo "fd $(basename $f) -> $(readlink /proc/$PID/fd/$(basename $f))"; grep '^sigmask:' "$f"; }; done
```

**OUTPUT:**
```text
lrwx------ 1 root root 64 Jul  1 22:27 7 -> anon_inode:[signalfd]
fd 7 -> anon_inode:[signalfd]
sigmask:	0000000000014a03
```

That `sigmask` decodes to exactly kitty's handled-signal set — including `SIGCHLD`:

**COMMAND:**
```bash
python3 -c "import signal; m=0x14a03; print(', '.join(signal.Signals(b+1).name for b in range(64) if m&(1<<b)))"
```

**OUTPUT:**
```text
SIGHUP, SIGINT, SIGUSR1, SIGUSR2, SIGTERM, SIGCHLD
```

which is precisely `KITTY_HANDLED_SIGNALS`:

**SOURCE — `kitty/child-monitor.c:121`:**
```c
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

Crucially, whichever transport a platform uses, the per-signal callback does **no real work** — it only sets boolean flags in a `SignalSet`, so signal handling can never race with child-registry mutation:

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

The callback is invoked from the I/O-thread loop, which reads the signal fd and then acts on the flags:

**SOURCE — `kitty/child-monitor.c:1519`:**
```c
                read_signals(children_fds[1].fd, handle_signal, &ss);
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

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` lines 111-112 — the last loop tick immediately followed by clean shutdown; the `Processing global state…` prefix on line 112 is a harmless stderr-interleaving artifact):**
```bash
grep -nE 'loop tick, wakeups_happened|main loop exiting' /tmp/blitzy_obs/run.log | tail -2
```
```text
111:[0.441] --------- loop tick, wakeups_happened: 1 ----------
112:Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.448] main loop exiting
```

### The timing race, captured in a single millisecond

The clearest timing evidence is a **same-millisecond pair**: within one relayout pass, the C `resize_pty` cannot find the id (the child already left `children[]`, and its removal has been applied) so it logs `Failed to send resize signal`, and in the *same* Python dispatch the `elif debug_rendering` branch still prints `SIGWINCH sent to child` for that same id. The C view (gone) and the Python view (alive) disagree in the same millisecond:

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` lines 21-22 — identical `[0.231]` timestamp, identical id 3):**
```bash
sed -n '21p;22p' /tmp/blitzy_obs/run.log
```
```text
[0.231] Failed to send resize signal to child with id: 3 (children count: 6) (add queue: 0)
[0.231] SIGWINCH sent to child in window: 3 with size: (10, 35, 315, 180)
```

To measure how often this exact adjacency occurs (a `Failed…` line immediately followed by a same-timestamp `SIGWINCH…` for the same id) and the timestamp delta within each pair, the run log was analyzed with a temporary script:

**COMMAND:**
```bash
python3 - /tmp/blitzy_obs/run.log <<'PY'
import re, sys
lines = open(sys.argv[1]).read().splitlines()
ts    = re.compile(r'\[(\d+\.\d+)\]')
fail  = re.compile(r'Failed to send resize signal to child with id: (\d+)')
winch = re.compile(r'SIGWINCH sent to child in window: (\d+)')
pairs = []
for i in range(len(lines) - 1):
    f, tf = fail.search(lines[i]), ts.search(lines[i])
    w, tw = winch.search(lines[i+1]), ts.search(lines[i+1])
    if f and tf and w and tw and f.group(1) == w.group(1) and tf.group(1) == tw.group(1):
        pairs.append((tf.group(1), f.group(1), round((float(tw.group(1)) - float(tf.group(1))) * 1000, 3)))
print("same_ms_same_id_adjacent_pairs =", len(pairs))
print("distinct_delta_ms              =", sorted({p[2] for p in pairs}))
print("first_pairs (ts,id,delta_ms)   =", pairs[:4])
PY
```

**OUTPUT:**
```text
same_ms_same_id_adjacent_pairs = 20
distinct_delta_ms              = [0.0]
first_pairs (ts,id,delta_ms)   = [('0.231', '3', 0.0), ('0.301', '14', 0.0), ('0.328', '18', 0.0), ('0.340', '18', 0.0)]
```

So this same-millisecond disagreement occurred **20 times** in this run, and the measured timestamp delta within every pair is **0.0 ms** (`distinct_delta_ms = [0.0]`) — both prints land in the same synchronous relayout pass, straddling the C boundary where the two registries momentarily disagree.

**Summary of O4:** signals arrive asynchronously and coalesce, but on Linux they are blocked (`sigprocmask`) and drained synchronously from a `signalfd` on the I/O thread (self-pipe only on non-`signalfd` platforms), where the callback is flag-only (`handle_signal` → `child_died`; then the `waitpid(-1,…,WNOHANG)` loop); all registry mutation is synchronous and serialized at the tick top (`remove_children` then `add_children` under `children_lock`); the `[s.mmm]` prefixes expose the disagreement in time.

---

## O5 — Conflicting-liveness resolution: the moments kitty must decide what is still alive

There are exactly **two graceful resolution points** where kitty must reconcile conflicting views of liveness, plus a set of edge cases that all funnel into the same `needs_removal` discard.

### Resolution point 1 — `resize_pty` id-not-found skip (C thinks the child is gone)

When a resize targets an id absent from both `children[]` and `add_queue[]`, `resize_pty` logs and returns without error (no dereference, no crash):

**SOURCE — `kitty/child-monitor.c:610`:**
```c
    } else log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", window_id, self->count, add_queue_count);
```

**OBSERVED EVIDENCE (`/tmp/blitzy_obs/run.log` line 21):**
```bash
sed -n '21p' /tmp/blitzy_obs/run.log
```
```text
[0.231] Failed to send resize signal to child with id: 3 (children count: 6) (add queue: 0)
```

### Resolution point 2 — `on_child_death` pop→`None` discard (Python thinks the window is gone)

When a death arrives for a window no longer in the map, `pop(window_id, None)` yields `None` and the callback returns early:

**SOURCE — `kitty/boss.py:883-885`:**
```python
        window = self.window_id_map.pop(window_id, None)
        if window is None:
            return
```

As reported under O2 Case B, this resolution point is a **source-verified conflict path**: direct instrumentation of `on_child_death` in the churn run recorded 20 deaths, **all** with the window still present (`ABSENT (pop returns None): 0`), so the `None`-discard branch guards against a race (double-notify / manual-close-vs-reap) that this particular run did not provoke. Resolution point 1 (the `resize_pty` id-not-found skip), by contrast, *was* directly observed **21×**.

### Both resolve benignly — proven by exit status 0

The decisive evidence that these conflicts resolve without crashing is that the process completed normally despite **21** `Failed to send resize signal` conflicts:

**OBSERVED EVIDENCE (exit status + clean shutdown marker):**
```bash
cat /tmp/blitzy_obs/kitty_exit.txt
grep -o '\[[0-9.]*\] main loop exiting' /tmp/blitzy_obs/run.log
```
```text
KITTY_EXIT=0
[0.448] main loop exiting
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

  `read_bytes` (`kitty/child-monitor.c:1337`) returns `len != 0` (`kitty/child-monitor.c:1355`), i.e. false at EOF — a **source-described** removal path that sets `needs_removal`. Whether this branch fired for every child cannot be asserted here: this run was launched with `close_on_child_death=yes`, which enables a **second** removal trigger — when a child exits, `reap_children` reaps it by pid and calls `mark_child_for_removal(self, pid)`:

  **SOURCE — `kitty/child-monitor.c:1418,1422` (the `SIGCHLD`-reap removal path enabled by `close_on_child_death`):**
  ```c
          pid = waitpid(-1, &status, WNOHANG);
  ```
  ```c
              if (enable_close_on_child_death) mark_child_for_removal(self, pid);
  ```

  Both the PTY-EOF branch and the `SIGCHLD`-reap branch converge on the same `needs_removal` flag, so from the outside the removal is observed but *which* branch fired first for a given child cannot be distinguished without instrumenting the C loop (which the read-only scope forbids). What **is** directly observable is that all 20 children were removed and their deaths delivered — `on_child_death` fired **20×** (instrumented under O2 Case B) — and that the invalid-fd `POLLNVAL` branch did **not** fire (`0`, shown below). This document therefore does **not** claim all 20 children exited specifically via the EOF branch; it reports EOF as a source-verified path and the `SIGCHLD`-reap path as the trigger that `close_on_child_death=yes` explicitly enables.

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

  **OBSERVED EVIDENCE (honest null result):** this line did **not** appear in this run.

  **COMMAND:**
  ```bash
  grep -c 'had its fd unexpectedly closed' /tmp/blitzy_obs/run.log
  ```

  **OUTPUT:**
  ```text
  0
  ```

  Reported exactly as observed (rule: report even absence): the `POLLNVAL` path exists and its literal is compiled into `fast_data_types.so` (proven by the `strings` check in "How this was observed"), but it was not exercised in this run — the children's fds were never found invalid. How the teardown *was* driven is analyzed just below.

- **Shutdown (mark all, then remove).** At loop shutdown every remaining child is flagged and removed in one pass:

  **SOURCE — `kitty/child-monitor.c:1574-1575`:**
  ```c
    for (i = 0; i < self->count; i++) children[i].needs_removal = true;
    remove_children(self);
  ```

  The corresponding observed end-of-run marker is `[0.448] main loop exiting` (quoted above).

- **Supporting removal helpers.** By pid on reap: `mark_child_for_removal` (`kitty/child-monitor.c:1386`); on a UI close request: `mark_child_for_close` (`kitty/child-monitor.c:541`), which scans `children[]` then `add_queue[]` and sets `needs_removal`; and `hangup` (`kitty/child-monitor.c:1294`) delivers `SIGHUP` to the child's process group during removal. All roads set the same `needs_removal` flag consumed by `remove_children` (`kitty/child-monitor.c:1313`) at the next tick.

**Summary of O5:** the two liveness conflicts (a resize for a child C no longer holds; a death for a window Python no longer holds) are each resolved by a benign skip; every teardown trigger (EOF, `POLLNVAL`, close request, reap-by-pid, shutdown) converges on the single `needs_removal` flag applied synchronously at the tick top — and the run's exit status `0` confirms the resolution never crashes.


---

## Design vs. established best practice (framing only)

kitty's design aligns with well-established systems-programming practice; its own code and the observed behavior above remain the source of truth. Three alignments are worth naming:

- **`SIGCHLD` reaping with a `waitpid(-1, …, WNOHANG)` loop.** Because Unix does not queue `SIGCHLD`, the canonical remedy for a burst of exits is to loop `waitpid` with `WNOHANG` until it stops returning positive pids. kitty does exactly this at `kitty/child-monitor.c:1418`, so a coalesced signal still reaps every child.
- **Async-signal-safe, flag-only handling — via `signalfd` on Linux.** Best practice is that signal work be deferred off the async-handler context — only set a flag or drain a descriptor on the event loop. On Linux kitty goes further than the self-pipe trick: it blocks the handled signals (`sigprocmask(SIG_BLOCK, …)`) and reads them from a pollable `signalfd` (`kitty/loop-utils.c:39-42`; `HAS_SIGNAL_FD` guard at `kitty/loop-utils.h:14-18`), reserving the classic self-pipe + `sigaction` handler for non-`signalfd` platforms (`kitty/loop-utils.c:45-54`, the `#else` branch). Either transport funnels into the flag-only callback `handle_signal`, which sets only `SignalSet` booleans (`kitty/child-monitor.c:1359-1372`); the real work runs later on the I/O thread after `read_signals` (`kitty/loop-utils.c:130-133`) drains the fd. The `signalfd` path is the one observed here (`HAS_SIGNAL_FD=1`; `/proc/$PID/fd/7 -> anon_inode:[signalfd]`).
- **`TIOCSWINSZ`-driven `SIGWINCH`.** The kernel auto-delivers `SIGWINCH` to a PTY's foreground process group only when the size actually changes via `TIOCSWINSZ`; kitty issues exactly that ioctl in `pty_resize` (`kitty/child-monitor.c:579`), and guards the dispatch behind a size-changed check (`kitty/window.py:861`) so redundant resizes are suppressed.

The single most important structural choice — the **queue-and-apply** split (async signals set flags; the I/O thread applies queued adds/removes synchronously under `children_lock` at the top of each tick, `kitty/child-monitor.c:1491-1494`) — is what keeps the two liveness views from corrupting shared state even while they momentarily disagree (O4/O5).

---

## Coverage pass

Every distinct thing the question names is addressed below, each with its section and its grounding.

**Question clauses:**

- **"terminal windows appear, resize, and disappear in quick succession"** — the whole document; provoked by the 5-tab / 20-short-lived-window `--session` run (see "How this was observed"). Appear = `Child launched` ×20; resize = `SIGWINCH sent to child` ×48; disappear = `close_on_child_death=yes` teardown ending in `main loop exiting`.
- **"a new window is created and immediately used to run a command"** — **O1** (each `launch` window runs `sh -c "…; true"`; `Child launched` at run.log:3,5,7).
- **"resize events AND signals start flowing"** — **O1** (resize dispatch `kitty/window.py:861-873`) + **O4** (`SIGWINCH` via `TIOCSWINSZ`, `SIGCHLD` delivered through a `signalfd` on Linux — self-pipe on non-`signalfd` platforms). Both resize events and signals are covered explicitly.
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
| `WeakValueDictionary` | O3 | `kitty/boss.py:344` | auto-drop backs the `pop → None` guard (O2 Case B; source-verified — instrumented `ABSENT=0` this run) |
| `needs_removal` | O3, O5 | `kitty/child-monitor.c:67,1535,1545,1574` | children count values `6 5 7 6 5 … 4 … 1` (distinct 7,6,5,4,1; run.log:21,38,45,…) |
| `add_queue` / `remove_queue` | O4 | `kitty/child-monitor.c:1491-1494` (apply), `:305` (queue) | `(add queue: 0)` in the `Failed…` lines |
| `signalfd` (Linux) / self-pipe (fallback) / `read_signals` | O4 | `kitty/loop-utils.h:14-18`, `kitty/loop-utils.c:39-42,45-54,130-133`; call at `kitty/child-monitor.c:1519` | `HAS_SIGNAL_FD=1`; `/proc/$PID/fd/7 -> anon_inode:[signalfd]`; `sigmask 0x14a03` = handled set |
| `waitpid(-1, …, WNOHANG)` | O4, best-practice | `kitty/child-monitor.c:1418` | coalesced reap; exit 0 |
| `resize_pty` | O1, O2, O5 | `kitty/child-monitor.c:592,610` | `Failed to send resize signal…` ×21 |
| `on_child_death` | O2, O5 | `kitty/boss.py:881-885` | instrumented: fired 20×, all `PRESENT`, `pop → None` not taken; `None` branch source-verified; exit 0 |

**Honest null result:** the `POLLNVAL` line `The child %lu had its fd unexpectedly closed` (`kitty/child-monitor.c:1547`) did **not** occur in this run (`grep -c … → 0`); the invalid-fd path exists but was not exercised. Under `close_on_child_death=yes` each child's exit is removed via the `SIGCHLD`/`reap_children` → `mark_child_for_removal` path (`kitty/child-monitor.c:1418,1422`) and/or the PTY-EOF branch (`POLLIN|POLLHUP`) — both converging on `needs_removal` — neither of which is the invalid-fd `POLLNVAL` condition.

**Evidence discipline:** every count (`48`, `21`, `20`, `1`, `0`, `4`), every derived value (same-ms pairs `20`, delta `0.0 ms`, children-count sequence), every timestamp (`[0.231]`, `[0.448]`, …), and the exit status (`0`) is traceable to a pasted `COMMAND` block above and to the captured `/tmp/blitzy_obs/run.log`. Every `file:line` citation was re-grepped against the live tree at branch snapshot `815df1e210e0` before being pasted.

---

## Reproducibility note

To reproduce, run the three commands below exactly as used to gather this document's evidence — build with event-loop logging, start a headless X server, then run kitty against the churn session file:

**COMMAND:**
```bash
# 1. Build with debug + event-loop logging
python3 setup.py build --debug --extra-logging=event-loop
# 2. Start a headless X server
Xvfb :99 -screen 0 1280x800x24 -nolisten tcp > /tmp/blitzy_obs/xvfb.log 2>&1 &
export DISPLAY=:99 TERM=xterm-kitty LANG=C.UTF-8 LC_ALL=C.UTF-8
# 3. Run kitty under the churn session (5 tabs × 4 short-lived windows = 20 windows, layout grid)
timeout 90 ./kitty/launcher/kitty --config NONE --debug-rendering \
    -o close_on_child_death=yes -o confirm_os_window_close=0 \
    --session /tmp/blitzy_obs/session.conf > /tmp/blitzy_obs/run.log 2>&1
```

The session file `/tmp/blitzy_obs/session.conf` opens 20 short-lived windows across five `layout grid` tabs (one initial tab plus four `new_tab`s, four `launch sh -c "…; true"` windows each) — the exact 5-tab / 20-window driver used throughout this document. Grep the captured stderr for `Child launched`, `SIGWINCH sent to child`, and `Failed to send resize signal`. Exact counts vary run-to-run with scheduling, but the same **classes** of lines — including same-millisecond `Failed…`/`SIGWINCH…` pairs for one id and a final `main loop exiting` with exit status `0` — reproduce reliably.

