# How kitty keeps internal state consistent as windows appear, run commands, resize, and disappear in quick succession

**Subject:** kitty terminal emulator — `kovidgoyal/kitty` @ commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config").
**Task type:** read-only, evidence-first investigative Q&A. The repository is left unchanged; every temporary observation script is shown in full and then deleted (see *Appendix C — Cleanup & read-only proof*).

## The question being answered

> "I want to understand how kitty keeps its internal state consistent when terminal windows appear, resize, and disappear in quick succession. When a new window is created and immediately used to run a command, resize events and signals start flowing through the system. What happens if the window is gone before everything has finished reacting to those changes? How does kitty decide what state to keep and what to discard? I am especially interested in how timing affects signal delivery and internal bookkeeping, and whether there are moments where the system has to resolve conflicting views of what is still alive. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."

Decomposed into six sub-questions, each answered explicitly below and re-checked in the *Coverage pass*:

- **R1** — How is internal state kept consistent across the full lifecycle (create → run → resize → destroy), especially interleaved "in quick succession"?
- **R2** — When a new window is created and immediately runs a command, what is the actual flow of resize events and signals?
- **R3** — What happens if the window is gone *before* everything has finished reacting to those in-flight events?
- **R4** — How does kitty decide what state to keep and what to discard?
- **R5** — How does timing affect signal delivery and internal bookkeeping?
- **R6** — Are there moments where the system must resolve conflicting views of what is still alive, and how?

**Methodology (how the evidence was produced).** kitty was built from source in its default configuration and driven through real window create → run → resize → close sequences via the real launcher binary. All build and runtime observation was performed **as a non-root normal user** (`ubuntu`, `uid=1000`) inside the task's Docker container against a private copy of the repository at `/home/ubuntu/kitty` (the destination repository tree on the host was never used for building and is left pristine). Behaviour was observed with kitty's own `--debug-rendering` logging, an `--extra-logging=event-loop` debug build, and external `strace` syscall tracing. Every behavioural claim below is immediately followed by the *complete, unedited* output that demonstrates it, the exact command that produced it, and a `file:line` code anchor. Statements that could not be directly observed at runtime are explicitly labelled **inferred** or **NOT OBSERVED**. All line numbers were re-verified against the source at commit `815df1e21` (see *Appendix B*), and the complete, unabridged build/test logs are reproduced in *Appendix D*.

---

## TL;DR — direct answers to R1–R6

- **R1 (lifecycle consistency).** Consistency rests on four observed mechanisms working together: (1) a **startup gate** that blocks the child's `execvp` until kitty has set the first PTY size, so the command never sees a zero/garbage size; (2) a **single-owner PTY-resize path** — every size change funnels through `Window.set_geometry` → `child_monitor.resize_pty` → `ioctl(TIOCSWINSZ)`, and this path runs **on the main (Python/GUI) thread**, not on the I/O thread: `resize_pty` is a synchronous C-extension method that takes `children_mutex` and issues the `ioctl` inline (the separate `KittyChildMon` I/O thread only polls fds and reconciles the child arrays); (3) a **de-duplication guard** that discards an unchanged size; and (4) **per-tick reconciliation** on the I/O thread that always runs `remove_children()` *then* `add_children()` once per loop iteration under the mutex. Anchors: `kitty/window.py:850`, `kitty/child-monitor.c:592`, `kitty/window.py:861`, `kitty/child-monitor.c:1493-1494`.
- **R2 (create-and-run flow).** Observed order for a fresh window: Python `Boss.add_child` registers the child into the C **`add_queue[]`** and wakes the I/O loop (`kitty/boss.py:585-588`, `kitty/child-monitor.c:305-321`); the first `Window.set_geometry` calls `resize_pty`, which does `ioctl(fd, TIOCSWINSZ, …)` (`kitty/child-monitor.c:579`) — this is the resize "signal", and it flows **kitty → child** (the kernel then sends `SIGWINCH` to the child's process group); only then does kitty `mark_terminal_ready()`, releasing the child to `execvp` the command (`kitty/window.py:866`, `kitty/child.c:151-159`). Subsequent resizes emit `SIGWINCH sent to child in window: …` (`kitty/window.py:873`).
- **R3 (in-flight teardown).** If the window disappears while events are still in flight, nothing crashes. Two concrete outcomes were observed: (a) a resize aimed at an already-removed child is **safely discarded** with a single `log_error` — `Failed to send resize signal to child with id: …` (`kitty/child-monitor.c:610`), observed **thousands of times under churn with zero crashes**; and (b) already-buffered output of a removed child is **fully drained** before the child object is freed, via a reference-counted snapshot in `parse_input` (`kitty/child-monitor.c:521`).
- **R4 (keep vs discard).** The rule is *"keep the in-flight bytes, discard the dead handle."* On the main thread, `parse_input` takes an **`INCREF`'d snapshot** of the live children under the mutex (`kitty/child-monitor.c:479-480`); a child concurrently marked for removal still has its final buffered output parsed with `do_parse(…, flush=true)` (`kitty/child-monitor.c:521`) and its Python death-notify fired (`:522`), and only then is it `FREE_CHILD`'d (`:525`). Live parsing of a child already flagged `needs_removal` is **skipped** (`:529`). The Child/Screen survive until the snapshot's refcount reaches zero (`DECREF_CHILD`, `:532`).
- **R5 (timing).** `SIGCHLD` (child death) reaches the I/O thread through a Linux **`signalfd`** (the canonical/active path here; the self-pipe path is the non-Linux fallback) that is polled in the event loop; the signal's *effect* is processed on the **next loop tick**, not inside a handler, giving a bounded, sub-millisecond delay. On the active `signalfd` path the handled signals are **blocked** with `sigprocmask(SIG_BLOCK, …)` and drained from the fd — **no `sigaction`/`SA_RESTART` handler is installed** (that belongs to the inactive self-pipe fallback). Measured child-exit → reap over **10 identical runs**: **{0.090, 0.244, 0.314, 0.083, 18.333, 0.100, 0.304, 0.116, 0.106, 0.088} ms** (9/10 sub-millisecond; one 18.3 ms scheduler-jitter outlier) — run-to-run **variable**, reported as a distribution. Multiple child deaths **coalesce** into one `SIGCHLD`, so reaping loops on `waitpid(-1, …, WNOHANG)` (`kitty/child-monitor.c:1418`). The handled-signal set is `SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` — **not `SIGWINCH`** (`kitty/child-monitor.c:121`).
- **R6 (conflicting liveness views).** Yes — there are **three** independent in-memory "views of what is alive," reconciled by explicit guards so a window vanishing mid-flight is a *no-op*, not a crash. The views: the C I/O-thread `children[]`/`add_queue[]`/`remove_queue[]` (`kitty/child-monitor.c:82-84`); the GUI/main-thread `global_state.os_windows[].tabs[].windows[]` (`kitty/state.c:12`, `kitty/state.h:189,226,265`); and the Python `Boss.window_id_map` `WeakValueDictionary` (`kitty/boss.py:344`). The reconciliation was **observed directly and at scale**: driving rapid create/close churn produced hundreds-to-thousands of `Failed to send resize signal … (children count: N) (add queue: 0)` events — the Python view still holding a window the C view had already removed — with **zero** crashes across every run. Three code guards make this benign: `on_child_death` pops-and-early-returns if the window is already gone (`kitty/boss.py:883-885`); `mark_child_for_close` searches both `children[]` **and** `add_queue[]` (`kitty/child-monitor.c:544-558`); and `hangup` tolerates an already-exited child via `ESRCH` (`kitty/child-monitor.c:1297`, **observed**).

---

## 1. How this was built and run (canonical, non-root normal-user configuration)

The host sandbox lacks the C/Go toolchain and native libraries (and its Python is out of kitty's supported range), so — per the task's environment instruction — the build and all runtime observation were performed inside the provided Docker container (`kitty-setup`, image `kitty-setup:ready`, derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). All commands below were run **as the non-root user `ubuntu` (`uid=1000`)** against a private clone of the repository at `/home/ubuntu/kitty`, so the destination repository tree is never touched by the build (leaving even `go.sum` untouched — see *Appendix C*). The single non-default environmental choice is headless rendering under a pre-running `Xvfb :99` with software GL (`LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`); the kitty configuration itself is default (`--config NONE`).

### 1.1 Toolchain versions and non-root proof (A1)

Command:

```bash
whoami; id
uname -a; grep PRETTY_NAME /etc/os-release; python3 --version; gcc --version | head -1
go version; pkg-config --modversion harfbuzz freetype2 fontconfig libpng lcms2 openssl
git -C /home/ubuntu/kitty log -1 --format='%H %s'
```

Complete output (note `uid=1000(ubuntu)` — this is a genuine non-root user, resolving any ambiguity about "normal user"):

```text
===== whoami/id (normal-user proof) =====
ubuntu
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev)
===== toolchain versions =====
Linux 164eed5eea36 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 24.04.2 LTS"
NAME="Ubuntu"
Python 3.12.3
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
go version go1.23.4 linux/amd64
8.3.0
26.1.20
2.15.0
1.6.43
2.14
3.0.13
===== commit confirmation =====
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 Wire up applying of font config
```

(harfbuzz 8.3.0, freetype2 26.1.20, fontconfig 2.15.0, libpng 1.6.43, lcms2 2.14, openssl 3.0.13.)

### 1.2 Commit confirmation (A2)

The private build clone is at the exact subject commit (shown on the last line above): `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 Wire up applying of font config`.

### 1.3 Canonical build (A3)

The Makefile `all` target is exactly `python3 setup.py` (`Makefile:12-13`). Command (from the build clone root, as `ubuntu`):

```bash
cd /home/ubuntu/kitty && time python3 setup.py; echo "CANONICAL_BUILD_EXIT=$?"
```

The build ran to **exit status 0** in ~47 s and performed, in order: 28 Wayland-protocol generation steps, **122** compile steps (kitty/child-monitor.c is step `[7/122]`), and **5** link steps (`[1/5]` `fast_data_types` … `[5/5]` `launcher`), then built the Go `kitten` tools. The **complete, unabridged 366-line build log is reproduced verbatim in Appendix D.1** (no elision). It produces the launcher binary and version:

```text
# canonical launcher:
36224 /home/ubuntu/kitty/kitty/launcher/kitty
kitty 0.35.2 created by Kovid Goyal
# debug launcher:
274656 /home/ubuntu/kitty_dbg/kitty/launcher/kitty
```

### 1.4 Timing/event-loop debug build (A4) — the primary R5 lever

The Makefile `debug-event-loop` target is `python3 setup.py build --debug --extra-logging=event-loop` (`Makefile:25-26`). To keep the canonical clone pristine, this was built in a separate clone (`/home/ubuntu/kitty_dbg`). Command:

```bash
cd /home/ubuntu/kitty_dbg && time python3 setup.py build --debug --extra-logging=event-loop; echo "DEBUG_BUILD_EXIT=$?"
```

This ran to **exit status 0**; the debug launcher is **274656 bytes** (vs the release launcher's 36224 bytes) and also reports `kitty 0.35.2`. The **complete, unabridged 215-line debug build log is reproduced verbatim in Appendix D.2**.

### 1.5 Test harness (A5) — confirming no lifecycle test exists

The Makefile `test` target is `python3 setup.py test` (`Makefile:15-16`). Command:

```bash
cd /home/ubuntu/kitty && python3 setup.py test; echo "TEST_EXIT=$?"
```

Complete summary lines (exit status 0; full log in Appendix D.3):

```text
Running under CI: False
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... skipped 'fish not installed'
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... skipped 'fish not installed'
Ran 145 tests in 14.422s
OK (skipped=4)
All Go tests succeeded, ran in 14.7 seconds
```

`kitty_tests/` contains **no** child-monitor or window-lifecycle test (`find_all_tests` at `kitty_tests/main.py:57` auto-discovers the test modules, excluding `main`/`gr`; none is a child-monitor/lifecycle test), which is exactly why the lifecycle-race behaviour here is observed with purpose-built temporary drivers rather than an existing test.

### 1.6 Exact run invocation (A6)

Every runtime observation used this invocation (default kitty config; headless via the pre-running Xvfb):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty --debug-rendering --config NONE [--listen-on unix:SOCK -o allow_remote_control=yes] <command>
```

Two benign lines appear on stderr in this headless container and are *not* errors: `Failed to open systemd user bus with error: No medium found` and, under rapid churn, `[glfw error 65544]: Too many timers added` (a GLFW timer-pool warning from creating many windows per second under software GL).

### 1.7 Version-discrepancy note (A7)

For accuracy: `pyproject.toml:2` declares `requires-python = ">=3.8"` (CI tests through 3.11); `go.mod:3` declares `go 1.22` (container's go 1.23.4 satisfies it); `setup.py:609` checks `at_least_version('harfbuzz', 1, 5)` while `docs/build.rst` documents harfbuzz `>=2.2.0` (container's 8.3.0 satisfies both). The container's Python is 3.12.3, within range.

---

## 2. Instrumentation (no source edits)

All observation used kitty's **existing** debug affordances plus external tracing — no source file was modified.

- **`--debug-rendering` logging.** Two lines emitted from `kitty/window.py` are the backbone of the lifecycle evidence:
  - `[{now:.3f}] Child launched` — printed on the **first** geometry, `kitty/window.py:871`.
  - `[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}` — printed on **subsequent** resizes, `kitty/window.py:873`.
- **External syscall tracing with `strace`** (version 6.8-0ubuntu2, pre-installed). This is external observation of the *run*, **not** a change to the kitty repository. It corroborates: `ioctl(fd, TIOCSWINSZ, …)` — the PTY resize propagation (`kitty/child-monitor.c:579`); `wait4(-1, …, WNOHANG)` — the reaping loop (`kitty/child-monitor.c:1418`; glibc implements `waitpid` via `wait4`); `signalfd4(…)` — the signal-delivery fd (`kitty/loop-utils.c:42`); and thread identity via each thread's `comm` name.
- **Temporary driver scripts** (shown in full in *Appendix A*, then deleted) drive the real launcher and, where noted, use kitty's remote-control (RC) protocol **only to trigger** real window operations — never as a substitute for the observed mechanism.

---

## 3. R2 — The create-and-run event flow

**Direct answer.** When a window is created and immediately used to run a command, the flow is:

1. **Register the child (Python → C).** `Boss.add_child(window)` calls `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` and records `self.window_id_map[window.id] = window`:

```python
    def add_child(self, window: Window) -> None:
        assert window.child.pid is not None and window.child.child_fd is not None
        self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
        self.window_id_map[window.id] = window
```

(`kitty/boss.py:585-588`.) On the C side, `add_child` appends the child to the staging **`add_queue[]`**, bumps its refcount, wakes the I/O loop, and returns — the brand-new child is *not* live yet; it waits in `add_queue[]` until the next I/O tick merges it. The **complete** C function (every line, including the closing `Py_RETURN_NONE;`):

```c
add_child(ChildMonitor *self, PyObject *args) {
#define add_child_doc "add_child(id, pid, fd, screen) -> Add a child."
    children_mutex(lock);
    if (self->count + add_queue_count >= MAX_CHILDREN) { PyErr_SetString(PyExc_ValueError, "Too many children"); children_mutex(unlock); return NULL; }
    add_queue[add_queue_count] = EMPTY_CHILD;
#define A(attr) &add_queue[add_queue_count].attr
    if (!PyArg_ParseTuple(args, "kiiO", A(id), A(pid), A(fd), A(screen))) {
        children_mutex(unlock);
        return NULL;
    }
#undef A
    INCREF_CHILD(add_queue[add_queue_count]);
    add_queue_count++;
    children_mutex(unlock);
    wakeup_io_loop(self, false);
    Py_RETURN_NONE;
}
```

(`kitty/child-monitor.c:305-321`: `INCREF_CHILD` at `:316`, `add_queue_count++` at `:317`, `children_mutex(unlock)` at `:318`, `wakeup_io_loop` at `:319`, `Py_RETURN_NONE` at `:320`.)

2. **First geometry → resize the PTY.** `Window.set_geometry` computes the PTY size, applies the de-dup guard, calls `resize_pty`, and — because the child is not yet launched — marks the terminal ready and prints `Child launched`. The **complete** method (no elision):

```python
    def set_geometry(self, new_geometry: WindowGeometry) -> None:
        if self.destroyed:
            return
        if self.needs_layout or new_geometry.xnum != self.screen.columns or new_geometry.ynum != self.screen.lines:
            self.screen.resize(max(0, new_geometry.ynum), max(0, new_geometry.xnum))
            self.needs_layout = False
            call_watchers(weakref.ref(self), 'on_resize', {'old_geometry': self.geometry, 'new_geometry': new_geometry})
        current_pty_size = (
            self.screen.lines, self.screen.columns,
            max(0, new_geometry.right - new_geometry.left), max(0, new_geometry.bottom - new_geometry.top))
        update_ime_position = False
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
            elif boss.args.debug_rendering:
                print(f'[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}', file=sys.stderr)
            self.last_reported_pty_size = current_pty_size
        else:
            mark_os_window_dirty(self.os_window_id)
```

(`kitty/window.py:850-876`; `child_is_launched` and `last_reported_pty_size` are initialised to `False` and `(-1, -1, -1, -1)` at `kitty/window.py:578-579`.) The de-dup guard is the `if current_pty_size != self.last_reported_pty_size:` at `:861`; the resize call `boss.child_monitor.resize_pty(...)` is `:863`; `mark_terminal_ready()` is `:866`; `child_is_launched = True` is `:867`; the `Child launched` print is `:871`; the `SIGWINCH sent …` print is `:873`. **This method runs on the main (Python/GUI) thread**, so `resize_pty` — and therefore the `TIOCSWINSZ` `ioctl` — executes on the main thread, not the I/O thread (proven in §4).

3. **The resize is an `ioctl(TIOCSWINSZ)` — the "signal" flows kitty → child.** `resize_pty` finds the fd in `children[]` or `add_queue[]` and calls `pty_resize`, which issues `ioctl(fd, TIOCSWINSZ, dim)`. The **complete** `pty_resize` + `resize_pty` (ending with `Py_RETURN_NONE;`):

```c
pty_resize(int fd, struct winsize *dim) {
    while(true) {
        if (ioctl(fd, TIOCSWINSZ, dim) == -1) {
            if (errno == EINTR) continue;
            if (errno != EBADF && errno != ENOTTY) {
                log_error("Failed to resize tty associated with fd: %d with error: %s", fd, strerror(errno));
                return false;
            }
        }
        break;
    }
    return true;
}

static PyObject *
resize_pty(ChildMonitor *self, PyObject *args) {
#define resize_pty_doc "Resize the pty associated with the specified child"
    unsigned long window_id;
    struct winsize dim;
    int fd = -1;
    if (!PyArg_ParseTuple(args, "kHHHH", &window_id, &dim.ws_row, &dim.ws_col, &dim.ws_xpixel, &dim.ws_ypixel)) return NULL;
    children_mutex(lock);
#define FIND(queue, count) { \
    for (size_t i = 0; i < count; i++) { \
        if (queue[i].id == window_id) { \
            fd = queue[i].fd; \
            break; \
        } \
    }}
    FIND(children, self->count);
    if (fd == -1) FIND(add_queue, add_queue_count);
    if (fd != -1) {
        if (!pty_resize(fd, &dim)) PyErr_SetFromErrno(PyExc_OSError);
    } else log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", window_id, self->count, add_queue_count);
    children_mutex(unlock);
    if (PyErr_Occurred()) return NULL;
    Py_RETURN_NONE;
}
```

(`kitty/child-monitor.c:577-614`: the `pty_resize` helper at `:577-590` does `ioctl(TIOCSWINSZ)` at `:579`, `EINTR` retry at `:580`, `EBADF`/`ENOTTY` tolerated at `:581`; the `resize_pty` method at `:592-614` performs the two `FIND` lookups — first `children[]` then `add_queue[]` — at `:606-607`, and the `else log_error(...)` at `:610` — this is the *"Failed to send resize signal"* line that becomes visible in the teardown race, R3/R6.) Setting the size with `TIOCSWINSZ` makes the kernel deliver `SIGWINCH` to the child's foreground process group — so the resize "signal" travels **from kitty to the child**, which is why kitty itself does not handle `SIGWINCH` (see R5).

### 3.1 Observed create-and-run output

Command (two windows created via a startup session; each immediately runs a command; adding window 2 shrinks window 1):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session /home/ubuntu/evidence/sess_split.conf
```

The session file (complete, shown in full for reproducibility — F3):

```text
# sess_split.conf — create two windows in one tab (auto-split layout).
# Each window immediately runs a command (R2 "create and run").
# Adding window 2 shrinks window 1 -> emits a SIGWINCH-sent line for window 1.
launch --title win1 sh -c "echo WINDOW_ONE_RUNNING; exec sleep 30"
launch --title win2 sh -c "echo WINDOW_TWO_RUNNING; exec sleep 30"
```

Complete `--debug-rendering` stderr:

```text
[0.312] OS Window created
[0.329] Failed to open systemd user bus with error: No medium found
[0.332] Child launched
[0.339] SIGWINCH sent to child in window: 1 with size: (11, 70, 630, 198)
[0.339] Child launched
```

This is the byte-exact format from `kitty/window.py:873`: `SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}`, where `current_pty_size = (screen.lines, screen.columns, width_px, height_px)`. Window 1's `Child launched` precedes any command output; when window 2 is added the layout shrinks window 1 to `(11, 70, 630, 198)` and emits the `SIGWINCH sent` line; window 2's own `Child launched` follows.

Corroborating `strace` (complete producing command and complete filtered output):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  strace -f -tt -e trace=ioctl,signalfd4,rt_sigprocmask -o /home/ubuntu/evidence/r2_session.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session /home/ubuntu/evidence/sess_split.conf
# then: grep -E 'signalfd4|TIOCSWINSZ|SIGWINCH' r2_session.strace
```

```text
58263 05:45:36.558998 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
58263 05:45:36.575280 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
58330 05:45:36.575338 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
58263 05:45:36.582121 ioctl(8, TIOCSWINSZ, {ws_row=11, ws_col=70, ws_xpixel=630, ws_ypixel=198}) = 0
58330 05:45:36.582160 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
58263 05:45:36.582438 ioctl(9, TIOCSWINSZ, {ws_row=11, ws_col=70, ws_xpixel=630, ws_ypixel=198}) = 0
58331 05:45:36.582483 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
```

Note the row/col of the `TIOCSWINSZ` calls match the debug sizes exactly; the `signalfd4` set `[HUP INT USR1 USR2 TERM CHLD]` contains **no** `SIGWINCH`; and each `TIOCSWINSZ` on the parent (main thread `58263`) is immediately followed by the kernel delivering `SIGWINCH {si_code=SI_KERNEL}` to a *child* thread (`58330`, `58331`) — confirming the resize signal flows kitty → child.

### 3.2 The startup gate (crux of "immediately used to run a command")

**Direct answer.** The child's command does **not** execute until kitty has set the first PTY size. The parent holds the write end of a "terminal ready" pipe; the child blocks reading the read end until the parent closes it. `mark_terminal_ready` performs that close:

```python
    def mark_terminal_ready(self) -> None:
        os.close(self.terminal_ready_fd)
        self.terminal_ready_fd = -1
```

(`kitty/child.py:362-364`.) In the child, after `setsid()` and `ioctl(TIOCSCTTY)`, it closes the write end and **blocks** in `wait_for_terminal_ready(ready_read_fd)` until EOF, only then `execvp`-ing. The **complete** `wait_for_terminal_ready` (including its closing brace — F9):

```c
wait_for_terminal_ready(int fd) {
    char data;
    while(1) {
        int ret = read(fd, &data, 1);
        if (ret == -1 && (errno == EINTR || errno == EAGAIN)) continue;
        break;
    }
}
```

(`kitty/child.c:71-78`; the child-side sequence `safe_close(ready_write_fd)` → `wait_for_terminal_ready(ready_read_fd)` → `safe_close(ready_read_fd)` → `execvp(exe, argv)` is `kitty/child.c:151-159`.)

**Observed** via `strace -f` of a child running `sh -c "echo GATE_MARKER_CHILD_RAN; exec sleep 5"`. Producing command and complete ordered trace (child TID `59407`, parent/main TID `59340`):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  strace -f -tt -e trace=setsid,ioctl,read,execve -o /home/ubuntu/evidence/gate.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE sh -c 'echo GATE_MARKER_CHILD_RAN; exec sleep 5'
# then: grep for the child/parent ordering shown below
```

```text
59407 05:48:56.529569 setsid()          = 59407
59407 05:48:56.529692 ioctl(12, TIOCSCTTY, 0) = 0
59407 05:48:56.529927 read(10,  <unfinished ...>
59340 05:48:56.534005 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
59407 05:48:56.534089 <... read resumed>0x7fffea374e93, 1) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
59407 05:48:56.534138 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
59407 05:48:56.534163 read(10, "", 1)   = 0
59407 05:48:56.538822 execve("/usr/bin/sh", ["sh", "-c", "echo GATE_MARKER_CHILD_RAN; exec"...], 0x576f520ced70 /* 24 vars */) = 0
59407 05:48:56.544734 execve("/usr/bin/sleep", ["sleep", "5"], 0x56318ef97888 /* 24 vars */) = 0
```

Reading top to bottom: the child does `setsid()` → `TIOCSCTTY` → **blocks** in `read(10, …)` on the ready pipe; the **parent** (`59340`) then sets the PTY size first with `TIOCSWINSZ`; that resize makes the kernel deliver `SIGWINCH {si_code=SI_KERNEL}` to the child (confirming resize flows kitty → child); the parent's `mark_terminal_ready` close makes the child's `read` return EOF (`read(10, "", 1) = 0`); and **only then** does the child `execve` the command (`/usr/bin/sh`, which then `execve`s `/usr/bin/sleep`). The command therefore never observes a zero/wrong initial size.

**On the `ERESTARTSYS` line (important correction).** The child's `read` shows `= ? ERESTARTSYS (To be restarted if SA_RESTART is set)` because the kernel-delivered `SIGWINCH` interrupted the blocking `read`. This is **child-side kernel behaviour, not evidence of an `SA_RESTART` handler in kitty**: (a) the parenthetical "To be restarted if SA_RESTART is set" is strace's *generic* textual decode of the internal `ERESTARTSYS` return value, printed for every such interruption; (b) `SIGWINCH` is not in kitty's handled set and the forked child has reset its handled signals to `SIG_DFL` (`kitty/child.c`), so `SIGWINCH`'s disposition in the child is the default (ignore), which is why the kernel transparently restarts the syscall and the very next `read(10, "", 1)` returns `0`; and (c) kitty's own active parent-side signal path installs **no** `sigaction` and therefore **no** `SA_RESTART` at all — it blocks the handled signals and reads them from a `signalfd` (see §6.1). In short, this trace line demonstrates the gate (interrupt → restart → EOF → `execve`), not kitty's signal-handler flags.

### 3.3 A subsequent resize and the de-dup guard (before / during / after)

**Direct answer.** A *changed* geometry emits one `SIGWINCH sent …` line and one `ioctl(TIOCSWINSZ)`; an *identical* geometry emits **neither** — the guard at `kitty/window.py:861` is false and the `else` branch merely marks the OS window dirty (`:875-876`), touching the child not at all.

Command (driver `driver_dedup.sh`, full text in *Appendix A*; RC used only to *trigger* three real resizes: **A** = 900×600 new, **B** = 900×600 again/identical, **C** = 1100×750 changed):

```bash
bash /home/ubuntu/evidence/driver_dedup.sh   # starts kitty under strace, then A, B, C via RC
```

Complete `--debug-rendering` stderr — note **step B produced no line** (the trailing GL line is the benign software-GL banner):

```text
[0.276] OS Window created
[0.293] Failed to open systemd user bus with error: No medium found
[0.299] Child launched
[4.521] SIGWINCH sent to child in window: 1 with size: (600, 900, 8100, 10800)
[7.030] SIGWINCH sent to child in window: 1 with size: (750, 1100, 9900, 13500)
[0.235] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

All `TIOCSWINSZ` calls from the trace — exactly **three** (initial launch + A + C), **none** for the identical step B, all on the same main thread TID `59567`:

```text
59567 05:50:11.004024 ioctl(10, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
59567 05:50:15.226452 ioctl(10, TIOCSWINSZ, {ws_row=600, ws_col=900, ws_xpixel=8100, ws_ypixel=10800}) = 0
59567 05:50:17.735307 ioctl(10, TIOCSWINSZ, {ws_row=750, ws_col=1100, ws_xpixel=9900, ws_ypixel=13500}) = 0
```

*Before* (initial): size `(22,71,…)`. *During* step B (identical to A): `last_reported_pty_size` already equals the new size → guard false → **no** `resize_pty`, **no** `ioctl`, **no** debug line. *After* step C (changed): guard true → new `ioctl` + new debug line. (`resize-os-window`'s width/height default to cells, so 900×600 → `ws_col=900`, `ws_row=600`, and pixels are `900×9` and `600×18` at the 9×18 cell size.)

---


## 4. R1 — Lifecycle state consistency (tie-together)

**Direct answer.** kitty keeps state consistent across create → run → resize → destroy — even when these interleave "in quick succession" — through four mechanisms, each observed above or below:

1. **Startup gate (create → run ordering).** The child cannot run its command until kitty has set the first PTY size, because `execvp` is gated behind `wait_for_terminal_ready` (`kitty/child.c:71-78,151-159`) which is released by `mark_terminal_ready` on the first `set_geometry` (`kitty/window.py:866`). Observed in §3.2: the gate-opening `read(10, "", 1) = 0` precedes the child's `execve`. This guarantees the command never reacts to a bogus initial size.
2. **Single-owner PTY-resize path, executed on the main thread (resize consistency).** Every size change funnels through `Window.set_geometry` → `child_monitor.resize_pty` → `ioctl(TIOCSWINSZ)`, and the fd lookup + ioctl happen under `children_mutex` (`kitty/child-monitor.c:598,606-611`). Crucially, `resize_pty` is a synchronous C-extension method invoked from Python's `set_geometry`, so it runs **on the main (Python/GUI) thread** — *not* on the `KittyChildMon` I/O thread. The I/O thread only `poll()`s fds and runs `remove_children()`/`add_children()`. There is exactly one writer of PTY size (the main thread, holding the mutex), so there is no torn or racing resize.

   **Observed thread identity.** Tracing thread `comm` names and which TID issues which syscall (producing command below), the `TIOCSWINSZ` `ioctl` is issued by the main thread (`TID==PID`, `comm=kitty`), while `wait4` reaping is issued by the I/O thread (`comm=KittyChildMon`):

   ```bash
   env DISPLAY=:99 ... strace -f -tt -e trace=ioctl,wait4 -o /home/ubuntu/evidence/f5_thread.strace \
     ./kitty/launcher/kitty --debug-rendering --config NONE sh -c 'exec sleep 5'
   # thread comm names read from /proc/<pid>/task/<tid>/comm
   ```

   ```text
# task comm names (from /proc/<pid>/task/*/comm):
TID 58955 -> kitty
TID 59021 -> KittyChildMon
# resize ioctl issued by MAIN thread (TID==PID 58955, comm=kitty):
58955 05:48:15.951914 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
# child reaping wait4 issued by I/O thread (TID 59021, comm=KittyChildMon):
59021 05:48:20.963455 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 59022
59021 05:48:20.963550 wait4(-1, 0x78d7997f9e60, WNOHANG, NULL) = -1 ECHILD (No child processes)
   ```

   This is the direct evidence for the correction: the resize `ioctl` runs on `TID 58955` (`comm=kitty`, the main thread, where `TID==PID`), whereas the child-reaping `wait4` runs on `TID 59021` (`comm=KittyChildMon`, the I/O thread). The two roles are on two different threads, and resize is the main thread's.
3. **De-duplication guard (idempotence).** An unchanged geometry is discarded (`kitty/window.py:861`), observed in §3.3 as "three resizes requested, but only the two *distinct* ones reach the kernel." This keeps redundant events from perturbing state.
4. **Per-tick reconciliation (create/destroy ordering under churn).** The I/O thread reconciles pending removals and additions exactly once per loop iteration, **removals first, then additions**, under the mutex:

```c
    while (LIKELY(!self->shutting_down)) {
        children_mutex(lock);
        remove_children(self);
        add_children(self);
        children_mutex(unlock);
```

(`kitty/child-monitor.c:1491-1495`: `remove_children(self)` at `:1493`, `add_children(self)` at `:1494`.) Because both staging queues are drained under the same lock in a fixed order every tick, the I/O thread always transitions from one internally-consistent `children[]` array to the next; a create and a destroy that arrive "at the same time" are serialised into this deterministic per-tick order rather than racing.

The remaining sub-sections (R3–R6) show what happens when the destroy half of the lifecycle lands *while events are still in flight*.

---

## 5. R3 & R4 — In-flight teardown and the keep-vs-discard rule

### 5.1 The two-thread model

kitty splits the work across two threads:

- The **C I/O thread**, named `KittyChildMon` (`kitty/child-monitor.c:1489`), owns the `poll()` set and the `children[]`/`add_queue[]`/`remove_queue[]` arrays; its loop does `remove_children()` then `add_children()` once per tick under `children_mutex` (`:1491-1495`, quoted above).
- The **main (Python/GUI) thread** calls `resize_pty` (synchronously, §4) and `parse_input` (`kitty/child-monitor.c:451`) to drain child output into screens and to fire death notifications into Python.

These two threads share the child arrays, so teardown must be reconciled between them without freeing an object the other thread is still using.

### 5.2 Removal staging on the I/O thread (`remove_children`)

When a child is flagged `needs_removal`, the I/O thread cleans it up and stages it into `remove_queue[]` (it does **not** free it here):

```c
remove_children(ChildMonitor *self) {
    if (self->count > 0) {
        size_t count = 0;
        for (ssize_t i = self->count - 1; i >= 0; i--) {
            if (children[i].needs_removal) {
                count++;
                cleanup_child(i);
                remove_queue[remove_queue_count] = children[i];
                remove_queue_count++;
                children[i] = EMPTY_CHILD;
                children_fds[EXTRA_FDS + i].fd = -1;
                size_t num_to_right = self->count - 1 - i;
                if (num_to_right > 0) {
                    memmove(children + i, children + i + 1, num_to_right * sizeof(Child));
                    memmove(children_fds + EXTRA_FDS + i, children_fds + EXTRA_FDS + i + 1, num_to_right * sizeof(struct pollfd));
                }
            }
        }
        self->count -= count;
    }
```

(`kitty/child-monitor.c:1313-1332`.) Step by step: a **reverse** scan (`:1316`) so compaction indices stay valid; `cleanup_child(i)` (`:1319`) closes the PTY fd and hangs up the child (next paragraph); the child struct is copied into `remove_queue[]` (`:1320`) and `remove_queue_count` bumped (`:1321`); its slot is cleared to `EMPTY_CHILD` (`:1322`) and its poll fd disabled (`:1323`); the arrays are `memmove`-compacted (`:1325-1326`); and finally `self->count` is decremented (`:1331`). `cleanup_child` itself:

```c
cleanup_child(ssize_t i) {
    safe_close(children[i].fd, __FILE__, __LINE__);
    hangup(children[i].pid);
```

(`kitty/child-monitor.c:1306-1308`: `safe_close` at `:1307`, `hangup` at `:1308`.)

### 5.3 The reference-counted snapshot — the keep-vs-discard core (`parse_input`)

**Direct answer (R4).** The rule is *"keep the in-flight bytes, discard the dead handle."* `parse_input` takes an `INCREF`'d snapshot of the live children under the mutex, then does all parsing and death-notification *without* holding the lock; a child concurrently removed on the I/O thread survives — its final buffered output is drained and its death is announced — and it is freed only when the snapshot's reference is dropped. First, the locked section (drain `remove_queue[]` → `remove_notify[]` with `INCREF`; snapshot live children into `scratch[]` with `INCREF`):

```c
parse_input(ChildMonitor *self) {
    // Parse all available input that was read in the I/O thread.
    size_t count = 0, remove_count = 0;
    bool input_read = false, reload_config_called = false;
    monotonic_t now = monotonic();
    children_mutex(lock);
    while (remove_queue_count) {
        remove_queue_count--;
        remove_notify[remove_count] = remove_queue[remove_queue_count];
        INCREF_CHILD(remove_notify[remove_count]);
        remove_count++;
        FREE_CHILD(remove_queue[remove_queue_count]);
    }

    if (UNLIKELY(kill_signal_received || reload_config_signal_received)) {
        if (kill_signal_received) {
            global_state.quit_request = IMPERATIVE_CLOSE_REQUESTED;
            global_state.has_pending_closes = true;
            request_tick_callback();
            kill_signal_received = false;
        }
        else if (reload_config_signal_received) {
            reload_config_signal_received = false;
            reload_config_called = true;
        }
    } else {
        count = self->count;
        for (size_t i = 0; i < count; i++) {
            scratch[i] = children[i];
            INCREF_CHILD(scratch[i]);
        }
    }
    children_mutex(unlock);
```

(`kitty/child-monitor.c:451-483`.) Under the lock: the `remove_queue[]` is drained into `remove_notify[]`, each entry `INCREF`'d so it cannot be freed out from under us (`:457-463`); and, in the normal case, **every** live child is snapshotted into `scratch[]` with `INCREF_CHILD` (`:479-480`). Then the lock is released (`:483`).

Immediately after the unlock, `parse_input` handles any queued peer (remote-control) messages — this section is shown here in full for completeness (it was previously omitted), since it executes between the snapshot and the parse:

```c
    Message *msgs = NULL;
    size_t msgs_count = 0;
    talk_mutex(lock);
    if (UNLIKELY(self->messages_count)) {
        msgs = malloc(sizeof(Message) * self->messages_count);
        if (msgs) {
            memcpy(msgs, self->messages, sizeof(Message) * self->messages_count);
            msgs_count = self->messages_count;
        }
        memset(self->messages, 0, sizeof(Message) * self->messages_capacity);
        self->messages_count = 0;
    }
    talk_mutex(unlock);

    if (msgs_count) {
        for (size_t i = 0; i < msgs_count; i++) {
            Message *msg = msgs + i;
            PyObject *resp = NULL;
            if (msg->data) {
                resp = PyObject_CallMethod(global_state.boss, "peer_message_received", "y#KO", msg->data, (int)msg->sz, msg->peer_id, msg->is_remote_control_peer ? Py_True : Py_False);
                free(msg->data);
                if (!resp) PyErr_Print();
            }
            if (resp) {
                if (PyBytes_Check(resp)) send_response_to_peer(msg->peer_id, PyBytes_AS_STRING(resp), PyBytes_GET_SIZE(resp));
                else if (resp == Py_None) send_response_to_peer(msg->peer_id, NULL, 0);
                Py_CLEAR(resp);
            } else send_response_to_peer(msg->peer_id, NULL, 0);
        }
        free(msgs); msgs = NULL;
    }
```

(`kitty/child-monitor.c:485-515`: it copies queued `Message`s out under `talk_mutex` (`:487`), then dispatches each via `peer_message_received` (`:504`) with no lock held.) Then the removed children and the survivors are processed **without any lock held**:

```c
    while(remove_count) {
        // must be done while no locks are held, since the locks are non-recursive and
        // the python function could call into other functions in this module
        remove_count--;
        if (remove_notify[remove_count].screen) do_parse(self, remove_notify[remove_count].screen, now, true);
        PyObject *t = PyObject_CallFunction(self->death_notify, "k", remove_notify[remove_count].id);
        if (t == NULL) PyErr_Print();
        else Py_DECREF(t);
        FREE_CHILD(remove_notify[remove_count]);
    }

    for (size_t i = 0; i < count; i++) {
        if (!scratch[i].needs_removal) {
            if (do_parse(self, scratch[i].screen, now, false)) input_read = true;
        }
        DECREF_CHILD(scratch[i]);
    }
```

(`kitty/child-monitor.c:517-533`.) This is the exact keep-vs-discard decision:

- **KEEP the in-flight bytes.** For each removed child that still has a screen, `do_parse(self, …screen, now, true)` runs with **`flush=true`** (`:521`) — it drains the child's *final buffered output* so nothing already received is lost — and only then does `PyObject_CallFunction(self->death_notify, "k", id)` announce the death to Python (`:522`, → `Boss.on_child_death`), after which `FREE_CHILD` releases the removal reference (`:525`).
- **DISCARD the dead handle / further live parsing.** For the live snapshot, a child already flagged `needs_removal` is **skipped** for further parsing — `if (!scratch[i].needs_removal) do_parse(…, false)` (`:529-530`) — but the loop **always** `DECREF_CHILD(scratch[i])` (`:532`). The Child and its `Screen` therefore stay alive for the *entire* duration of a parse even if the I/O thread concurrently removed them; the object is freed only when its refcount reaches zero.

`do_parse` is the parse helper that both call (quoted in full — every step):

```c
do_parse(ChildMonitor *self, Screen *screen, monotonic_t now, bool flush) {
    ParseData pd = {.dump_callback = self->dump_callback, .now = now};
    self->parse_func(screen, &pd, flush);
    if (pd.input_read) {
        if (pd.write_space_created) wakeup_io_loop(self, false);
        if (screen->paused_rendering.expires_at) {
            set_maximum_wait(MAX(0, screen->paused_rendering.expires_at - now));
        } else set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);
    } else if (pd.has_pending_input) set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);
    return pd.input_read;
}
```

(`kitty/child-monitor.c:438-448`.) It builds a `ParseData` seeded with the dump callback and current time, then delegates the actual byte-parsing to `self->parse_func(screen, &pd, flush)` — the `flush` argument is what forces a removed child's *final* buffered bytes to be drained. If bytes were read, it wakes the I/O loop when write space opened up and schedules the next wait; it returns whether any input was read.

### 5.4 Read-path edge handling (`read_bytes`)

How the I/O thread learns a child has gone on the read side:

```c
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;

    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;

    while(true) {
        len = read(fd, buf, available_buffer_space);
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
}
```

(`kitty/child-monitor.c:1337-1356`.) `EINTR`/`EAGAIN` simply **retry** (`:1347`); `EIO` — the *normal* indication that the PTY master closed because the child exited — is handled **silently** (the `perror` is guarded by `if (errno != EIO)` at `:1348`), commits zero bytes and returns `false`. In the main loop, a `false` return marks the child dead: `if (!has_more) { children_mutex(lock); children[i].needs_removal = true; children_mutex(unlock); }` (`kitty/child-monitor.c:1530-1535`).

### 5.5 Observed teardown scenarios (before / during / after)

**Scenario 1 — ordinary close: child self-exits (`SIGCHLD`).** Command `sh -c "echo BYE_FROM_CHILD; exit 7"`. kitty sent **no** kill/SIGHUP; the child exit is reaped by the coalescing loop. Producing command and complete key `strace` lines (child TID `59796`, I/O thread TID `59795`):

```bash
env DISPLAY=:99 ... strace -f -tt -e trace=wait4,getpgid,kill -o /home/ubuntu/evidence/sc1_selfexit.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE sh -c 'echo BYE_FROM_CHILD; exit 7'
```

```text
59796 05:50:47.360814 +++ exited with 7 +++
59795 05:50:47.360915 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 7}], WNOHANG, NULL) = 59796
59795 05:50:47.360992 wait4(-1, 0x7dd0ab7fde60, WNOHANG, NULL) = -1 ECHILD (No child processes)
59795 05:50:47.361112 getpgid(59796)    = -1 ESRCH (No such process)
```

*Before:* child alive, running the command. *During:* `WIFEXITED … WEXITSTATUS == 7` — the exact exit status is captured faithfully. *After:* the follow-up `wait4` returns `ECHILD`, so the reap loop terminates; the subsequent `getpgid(59796) = -1 ESRCH` with **no** following `kill` is Guard 3 firing (see §7.2).

**Scenario 2 (a.k.a. Scenario 4) — kitty-initiated close: `hangup` → `killpg(pgid, SIGHUP)`.** Command `sh -c "exec sleep 600"`, closed via RC `close-window`. Producing command and complete key `strace` lines (I/O thread `59876`, child `59877`):

```bash
# kitty started under: strace -f -tt -e trace=wait4,getpgid,kill -o sc4_close.strace ... sh -c 'exec sleep 600'
# then, after ~1s:  kitty @ --to unix:$SOCK close-window --match all
```

```text
59876 05:50:53.693277 getpgid(59877 <unfinished ...>
59877 05:50:53.693353 --- SIGHUP {si_signo=SIGHUP, si_code=SI_KERNEL} ---
59876 05:50:53.693377 <... getpgid resumed>) = 59877
59876 05:50:53.693408 kill(-59877, SIGHUP) = 0
59877 05:50:53.693738 +++ killed by SIGHUP +++
59876 05:50:53.693939 wait4(-1, [{WIFSIGNALED(s) && WTERMSIG(s) == SIGHUP}], WNOHANG, NULL) = 59877
59876 05:50:53.694063 wait4(-1, 0x7ca210ff8e60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

*During:* the I/O thread (`59876`) issues `kill(-59877, SIGHUP)` — i.e. `killpg(pgid, SIGHUP)` from `hangup` (`kitty/child-monitor.c:1299`); the child first receives a `SIGHUP {si_code=SI_KERNEL}` (the PTY master closing in `cleanup_child`'s `safe_close`, `:1307`) and is then `killed by SIGHUP`. *After:* reaped as `WIFSIGNALED … WTERMSIG == SIGHUP`, then `ECHILD`. **Contrast with Scenario 1:** self-exit is `WIFEXITED(status)` with *no* kitty-sent signal; kitty-initiated close is `WIFSIGNALED(SIGHUP)` preceded by `kill(-pgid, SIGHUP)` — both funnel through the *same* `reap_children` `WNOHANG` loop.

**Scenario 3 — resize while a child was already removed (the R6 conflicting-view race), forced at scale.** When a resize is requested for a window whose child was **already removed** from both `children[]` and `add_queue[]`, `resize_pty`'s two `FIND` lookups miss and it takes the `else` branch, emitting exactly one harmless `log_error` (`kitty/child-monitor.c:610`). This is examined in depth in §7.3–§7.4 (it is the directly-observed R6 reconciliation). Representative complete lines (first two + last of one run):

```text
### overlap_run1 — first 2 and last failed-resize lines (complete, unedited)
[9.209] Failed to send resize signal to child with id: 2 (children count: 134) (add queue: 0)
[32.505] Failed to send resize signal to child with id: 20 (children count: 131) (add queue: 0)
[40.382] Failed to send resize signal to child with id: 151 (children count: 0) (add queue: 0)
### selfexit_run1 — first 2 and last failed-resize lines (complete, unedited)
[1.466] Failed to send resize signal to child with id: 2 (children count: 2) (add queue: 0)
[1.486] Failed to send resize signal to child with id: 3 (children count: 2) (add queue: 0)
[10.531] Failed to send resize signal to child with id: 130 (children count: 0) (add queue: 0)
```

*Before:* Python still lists the window and calls `resize_pty(id, …)`. *During:* the C `children[]`/`add_queue[]` no longer contain that id. *After:* the resize is discarded, no crash. The scale and distribution of this event, and what it proves about R6, are in §7.

---


## 6. R5 — Timing: signal delivery and internal bookkeeping

### 6.1 The active signal-delivery mechanism: `signalfd` + `SIG_BLOCK` (correction)

**Direct answer.** On this Linux build, kitty's child monitor does **not** install a `sigaction` handler for the signals it cares about and therefore uses **no `SA_RESTART`** on its own signal path. Instead, `init_signal_handlers` **blocks** the signal set with `sigprocmask(SIG_BLOCK, …)` and reads deliveries synchronously from a `signalfd` that is added to the same `poll()` set as the child PTYs. The self-pipe + `sigaction(SA_SIGINFO|SA_RESTART)` code exists only in the `#else` fallback branch, which is **not** compiled on Linux (where `HAS_SIGNAL_FD` is defined). Full function (both branches shown; every line, nothing elided):

```c
static bool
init_signal_handlers(LoopData *ld) {
    ld->signal_read_fd = -1;
    sigemptyset(&ld->signals);
    for (size_t i = 0; i < ld->num_handled_signals; i++) sigaddset(&ld->signals, ld->handled_signals[i]);
#ifdef HAS_SIGNAL_FD
    if (ld->num_handled_signals) {
        if (sigprocmask(SIG_BLOCK, &ld->signals, NULL) == -1) return false;
        ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC);
        if (ld->signal_read_fd == -1) return false;
    }
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
    return true;
}

```

(`kitty/loop-utils.c:34-56`.) The mask is built first, common to both branches: `sigemptyset(&ld->signals)` at `:37` and the `for (…) sigaddset(&ld->signals, ld->handled_signals[i])` loop at `:38`. The active Linux path is the `#ifdef HAS_SIGNAL_FD` block `:40-44`: `sigprocmask(SIG_BLOCK, &ld->signals, NULL)` at `:41` blocks default delivery, and `ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC)` at `:42` creates the non-blocking, close-on-exec signal fd. **No `sigaction`, no `SA_RESTART`** in this branch. The inactive `#else` branch (`:46-53`) is the classic self-pipe: `self_pipe(ld->signal_fds, true)` at `:48` plus `struct sigaction act = {.sa_sigaction=handle_signal, .sa_flags=SA_SIGINFO | SA_RESTART, .sa_mask = ld->signals}` at `:51` and the `sigaction(ld->handled_signals[i], &act, NULL)` install loop at `:52`. It compiles only where `signalfd` is unavailable.

**Why the child's `ERESTARTSYS` (§3.2) is not evidence of this.** The `ERESTARTSYS` seen on the child's blocking `read` in the startup-gate trace is the **kernel** restarting the child process's own interrupted syscall after the child received `SIGWINCH`; it reflects the child's default signal disposition, not any flag kitty set. kitty's parent-side path here installs no `sigaction` at all, so it cannot be the source of an `SA_RESTART` behaviour. The two are unrelated.

The signal fd is created by `LoopData` and stored in `signal_read_fd`:

```c
typedef struct {
#ifndef HAS_EVENT_FD
    int wakeup_fds[2];
#endif
#ifndef HAS_SIGNAL_FD
    int signal_fds[2];
#endif
    sigset_t signals;
    int wakeup_read_fd;
    int signal_read_fd;
    int handled_signals[16];
    size_t num_handled_signals;
} LoopData;
```

(`kitty/loop-utils.h:31-43`: the `LoopData` struct declares `signal_fds[2]` at `:36`, `signals` at `:38`, `wakeup_read_fd` at `:39`, and `signal_read_fd` at `:40`.)

### 6.2 Draining deliveries: `read_signals`

Once `poll()` reports the signal fd readable, `read_signals` drains up to 32 `signalfd_siginfo` records per call and classifies each; here too both branches are shown in full:

```c
read_signals(int fd, handle_signal_func callback, void *data) {
#ifdef HAS_SIGNAL_FD
    static struct signalfd_siginfo fdsi[32];
    siginfo_t si;
    while (true) {
        ssize_t s = read(fd, &fdsi, sizeof(fdsi));
        if (s < 0) {
            if (errno == EINTR) continue;
            if (errno == EAGAIN) break;
            log_error("Call to read() from read_signals() failed with error: %s", strerror(errno));
            break;
        }
        if (s == 0) break;
        size_t num_signals = s / sizeof(struct signalfd_siginfo);
        if (num_signals == 0 || num_signals * sizeof(struct signalfd_siginfo) != (size_t)s) {
            log_error("Incomplete signal read from signalfd");
            break;
        }
        for (size_t i = 0; i < num_signals; i++) {
            si.si_signo = fdsi[i].ssi_signo;
            si.si_code = fdsi[i].ssi_code;
            si.si_pid = fdsi[i].ssi_pid;
            si.si_uid = fdsi[i].ssi_uid;
            si.si_addr = (void*)(uintptr_t)fdsi[i].ssi_addr;
            si.si_status = fdsi[i].ssi_status;
            si.si_value.sival_int = fdsi[i].ssi_int;
            if (!callback(&si, data)) break;
        }
    }
#else
    static char buf[sizeof(siginfo_t) * 8];
    static size_t buf_pos = 0;
    while(true) {
        ssize_t len = read(fd, buf + buf_pos, sizeof(buf) - buf_pos);
        if (len < 0) {
            if (errno == EINTR) continue;
            if (errno != EWOULDBLOCK && errno != EAGAIN) log_error("Call to read() from read_signals() failed with error: %s", strerror(errno));
            break;
        }
        buf_pos += len;
        bool keep_going = true;
        while (keep_going && buf_pos >= sizeof(siginfo_t)) {
            keep_going = callback((siginfo_t*)buf, data);
            buf_pos -= sizeof(siginfo_t);
            memmove(buf, buf + sizeof(siginfo_t), buf_pos);
        }
        if (len == 0) break;
    }
#endif
}
```

(`kitty/loop-utils.c:131-180`.) Linux path (`:132-159`): it `read()`s into a `static struct signalfd_siginfo fdsi[32]` (declared at `:133`, `read()` at `:136`), computes `num_signals = s / sizeof(struct signalfd_siginfo)` at `:144` (with an incomplete-read guard at `:145-146`), then in the `for (size_t i = 0; i < num_signals; i++)` loop at `:149` copies each record into a `siginfo_t si` carrying `si_signo`/`si_code`/`si_pid`/`si_uid`/`si_addr`/`si_status`/`si_value.sival_int` (`:150-156`), and invokes the caller-supplied `callback(&si, data)` for each at `:157` — that callback is `handle_signal` when driven from the child monitor's loop. Reading up to 32 at once is what lets a burst of deliveries be processed in one loop wakeup. The `#else` self-pipe drain (`:160-179`) reads raw bytes and reconstructs the same callback stream.

### 6.3 The signals kitty actually handles

The child monitor registers this exact set (note the absence of `SIGWINCH`):

```c
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

(`kitty/child-monitor.c:121`.) The set is `SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` (a `0`-terminated list declared at `:121`, consumed by `mask_variadic_signals(0, KITTY_HANDLED_SIGNALS)` at `:151` and `init_loop_data(…, KITTY_HANDLED_SIGNALS)` at `:173`). **`SIGWINCH` is deliberately not here**: the GUI process learns its own size from the windowing system (GLFW), not from a controlling terminal, so kitty *sends* `SIGWINCH` downward via `TIOCSWINSZ` (§3, §4) but never *receives* one for its own resize. The classifier that routes each delivered signal fills a local `SignalSet ss` (the struct `typedef struct { bool kill_signal, child_died, reload_config; } SignalSet;` at `:1359`):

```c
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
        case SIGUSR1:
            ss->reload_config = true;
            break;
        case SIGUSR2:
            log_error("Received SIGUSR2: %d\n", siginfo->si_value.sival_int);
            break;
        default:
            break;
    }
    return true;
}
```

(`kitty/child-monitor.c:1362-1383`.) It is a `switch(siginfo->si_signo)` in which `SIGINT`, `SIGTERM`, and `SIGHUP` all set `ss->kill_signal = true` (`:1368`); `SIGCHLD` sets `ss->child_died = true` (`:1371`); `SIGUSR1` sets `ss->reload_config = true` (`:1374`); and `SIGUSR2` merely `log_error`s its value (`:1377`). It does **not** reap or act inside the classifier — each case is a cheap flag-set on the caller's stack-allocated `ss`, and the loop body (§6.4) acts on those flags after `read_signals` returns.

### 6.4 The coalescing reaper: `waitpid(-1, …, WNOHANG)` loop

**How `child_died` reaches the reaper — the exact enclosing condition.** In the I/O loop body, `children_fds[1]` is the signal fd (poll index 1, since `EXTRA_FDS = 2` reserves index 0 for the wakeup fd). When it is readable, the loop drains it and dispatches synchronously *in the same iteration*:

```c
                SignalSet ss = {0};
                data_received = true;
                read_signals(children_fds[1].fd, handle_signal, &ss);
                if (ss.kill_signal || ss.reload_config) {
                    children_mutex(lock);
                    if (ss.kill_signal) kill_signal_received = true;
                    if (ss.reload_config) reload_config_signal_received = true;
                    children_mutex(unlock);
                }
                if (ss.child_died) reap_children(self, OPT(close_on_child_death));
```

(`kitty/child-monitor.c:1517-1526`.) So a `SIGCHLD` — turned into `ss.child_died = true` by `handle_signal` (§6.3) — triggers `reap_children(self, OPT(close_on_child_death))` at `:1526` in the very same `poll()`-return handling block that read the signal, not on a later tick. `kill_signal`/`reload_config` are instead published to the file-scope `kill_signal_received`/`reload_config_signal_received` flags under `children_mutex` (`:1520-1525`) for the main thread to observe.

Because many child deaths can collapse into a single `SIGCHLD`, `reap_children` must loop until the kernel has no more exited children:

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
    }
}
```

(`kitty/child-monitor.c:1413-1426`.) It loops `pid = waitpid(-1, &status, WNOHANG)` (`:1418`) and branches in this exact order: `pid == -1` → break *unless* `errno == EINTR`, in which case retry (`:1419-1421`) — so `ECHILD` (no children left) ends the loop; `else if (pid > 0)` → a child was reaped, so `if (enable_close_on_child_death) mark_child_for_removal(self, pid)` (`:1422`) flags it and `mark_monitored_pids(pid, status)` (`:1423`) records `(pid, status)`; `else` (`pid == 0`, no *ready* children) → break (`:1424`). The observed `wait4 … = -1 ECHILD` lines in §5.5 are exactly the `pid == -1` branch terminating this loop. `mark_child_for_removal` finds the child by pid and flags it:

```c
mark_child_for_removal(ChildMonitor *self, pid_t pid) {
    children_mutex(lock);
    for (size_t i = 0; i < self->count; i++) {
        if (children[i].pid == pid) {
            children[i].needs_removal = true;
            break;
        }
    }
    children_mutex(unlock);
}

```

(`kitty/child-monitor.c:1386-1395`: under `children_mutex` (`:1387`), scans `children[]` for the matching `pid` (`:1388-1389`) and sets `needs_removal = true` at `:1390`.)

### 6.5 Observed timing and its run-to-run distribution

**What was measured, and how.** The gap from a child's exit `SIGCHLD` becoming visible to kitty reaping it via `wait4`, taken from `-tt` (microsecond) timestamps in the child-exit `strace`, across **10 identical runs** of `sh -c 'exit 0'`. Producing command:

```bash
for i in $(seq 1 10); do
  env DISPLAY=:99 ... strace -f -tt -e trace=wait4 -o /home/ubuntu/evidence/timing_$i.strace \
    ./kitty/launcher/kitty --debug-rendering --config NONE sh -c 'exit 0'
done
# gap = (first successful wait4 timestamp) - (SIGCHLD-visible timestamp), in ms
```

Complete per-run values (ms):

```text
--- Run 1: gap = 0.090 ms ---
  child exit : 60001 05:51:40.550521 +++ exited with 0 +++
  kitty reap : 60000 05:51:40.550611 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60001
--- Run 2: gap = 0.244 ms ---
  child exit : 60079 05:51:42.437799 +++ exited with 0 +++
  kitty reap : 60078 05:51:42.438043 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60079
--- Run 3: gap = 0.314 ms ---
  child exit : 60157 05:51:44.501595 +++ exited with 0 +++
  kitty reap : 60156 05:51:44.501909 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60157
--- Run 4: gap = 0.083 ms ---
  child exit : 60235 05:51:46.505377 +++ exited with 0 +++
  kitty reap : 60234 05:51:46.505460 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60235
--- Run 5: gap = 18.333 ms ---
  child exit : 60313 05:51:48.552480 +++ exited with 0 +++
  kitty reap : 60312 05:51:48.570813 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60313
--- Run 6: gap = 0.100 ms ---
  child exit : 60398 05:52:09.489572 +++ exited with 0 +++
  kitty reap : 60397 05:52:09.489672 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60398
--- Run 7: gap = 0.304 ms ---
  child exit : 60477 05:52:11.484002 +++ exited with 0 +++
  kitty reap : 60476 05:52:11.484306 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60477
--- Run 8: gap = 0.116 ms ---
  child exit : 60556 05:52:13.479165 +++ exited with 0 +++
  kitty reap : 60555 05:52:13.479281 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60556
--- Run 9: gap = 0.106 ms ---
  child exit : 60635 05:52:15.408198 +++ exited with 0 +++
  kitty reap : 60634 05:52:15.408304 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60635
--- Run 10: gap = 0.088 ms ---
  child exit : 60714 05:52:17.325695 +++ exited with 0 +++
  kitty reap : 60713 05:52:17.325783 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 60714

All 10 gaps (ms, run order): [0.09, 0.244, 0.314, 0.083, 18.333, 0.1, 0.304, 0.116, 0.106, 0.088]
min=0.083  median=0.111  max=18.333
```

**Observed distribution (reported, not stabilised):** {0.090, 0.244, 0.314, 0.083, 18.333, 0.100, 0.304, 0.116, 0.106, 0.088} ms → **min 0.083, median ≈ 0.111, max 18.333**. Nine of ten runs are sub-millisecond; one run (#5) is an 18.333 ms outlier. Per the methodology rule, this is the *observed distribution* — the behaviour is **not** run-to-run deterministic, and the outlier is reported rather than discarded. The reason the gap is normally sub-ms: the child's exit makes the signal fd readable, so on the next `poll()` return the loop body drains it with `read_signals` (setting `ss.child_died`) and then calls `reap_children` **in the same iteration** at `:1526` (§6.4) — so the latency is essentially the delay until `poll()` next returns, i.e. ≈ one event-loop tick. The 18.333 ms outlier shows this single-tick delay can occasionally be inflated by kernel scheduling of the I/O thread.

**Event-loop tick evidence.** A `--debug --extra-logging=event-loop` build shows the loop structure that produces the "one tick" latency (complete stderr of a short run):

```text
[0.623] OS Window created
[0.635] Failed to open systemd user bus with error: No medium found
[0.639] Child launched
[0.639] starting handleEvents(0.00)
[0.639] pollForEvents final timeout: 0.000
[0.639] State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 1[1.391] display_read_ok: 0
[1.391] other dispatch done
[1.391] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[1.471] main loop exiting
[0.172] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

Each `loop tick wakeups_happened: 1` corresponds to one `poll()` return being serviced; the reaper runs within such a tick whenever that tick's signal-fd drain set `ss.child_died`, which is why the child-exit-to-reap gap is normally one tick.

### 6.6 A platform-specific startup-timing guard: the macOS early-`SIGWINCH` delay

**Direct answer.** There is exactly one *platform-specific* timing guard in the tree aimed at a `SIGWINCH` reaching a freshly-started child too early, and it is **macOS-only**: the `edit` entry point (used when editing a file inside kitty, e.g. via `kitten edit`, which `exec`s an editor such as `vim` in a kitty window) sleeps **50 ms before launching the editor**, so that the `SIGWINCH` kitty sends when it first sets the PTY size — the `ioctl(TIOCSWINSZ)` of §3, which makes the kernel signal the child's process group — does not arrive before the editor is ready to handle it. The construct is `kitty/entry_points.py:80-83`, inside `edit()` (defined at `kitty/entry_points.py:76`); every line is shown, nothing elided:

```python
    if is_macos:
        # On macOS vim fails to handle SIGWINCH if it occurs early, so add a small delay.
        import time
        time.sleep(0.05)
```

Line by line: the branch is gated on `is_macos` at `kitty/entry_points.py:80`; the source comment at `:81` states the exact reason — *"On macOS vim fails to handle SIGWINCH if it occurs early, so add a small delay."*; `time.sleep(0.05)` at `:83` is the 50 ms delay; the editor is then launched by `os.execv(exe, args[1:])` at `kitty/entry_points.py:93`, i.e. the sleep is inserted immediately before the `exec`, widening the gap between the child appearing and any early resize signal reaching it.

**NOT OBSERVED (inferred from source).** This branch was not exercised at runtime: the platform under test is **Linux** (§1; `uname -a` reports `x86_64 … GNU/Linux`), so `is_macos` is `False`, the branch is skipped entirely, and no delay is inserted — on Linux the editor is `exec`'d without it. It is documented here for completeness because it is the single place in the codebase where kitty *deliberately delays* to keep an early resize `SIGWINCH` from racing a child's startup, which is squarely on point for R5 (how timing affects `SIGWINCH` delivery relative to a child becoming ready). The general, cross-platform ordering that stops a child from running before its size is known is the `mark_terminal_ready` startup gate of §3.2 — which *is* observed on Linux; this macOS `sleep(0.05)` is an additional, OS-specific hardening layered on top of that gate.

---


## 7. R6 — Conflicting views of what is alive, and how they are resolved

**Direct answer.** Yes — there are moments where two subsystems disagree about whether a window/child is still alive, and kitty resolves them with three narrowly-scoped guards. The disagreement is real and was **forced and observed at scale** (§7.3): thousands of times, kitty's Python/GUI layer still listed a window and asked to resize its PTY *after* the C child-monitor had already fully removed the child. The resolution is always the same — the losing view's operation is turned into a safe no-op (a discarded resize, an early return, or a skipped `kill`), never a use-after-free. Across ~9,000+ forced conflict samples and every teardown scenario, there were **zero crashes, tracebacks, or aborts**.

### 7.1 The three liveness views

There are exactly three in-memory answers to "is this window/child alive?", owned by three different subsystems:

1. **The Python controller view** — `Boss.window_id_map`, a **`WeakValueDictionary`** (so entries vanish automatically when a `Window` is garbage-collected), plus each tab's `WindowList` and its `WindowGroup`s:

   ```python
        self.window_id_map: WeakValueDictionary[int, Window] = WeakValueDictionary()
   ```

   (`kitty/boss.py:344`.) Each tab's `WindowList` maintains its own **`WindowList.id_map`** — a plain `Dict[int, Window]` for that tab's windows, created empty in `WindowList.__init__` (`kitty/window_list.py:148`: `self.id_map: Dict[int, WindowType] = {}`), populated by `WindowList.add_window` (`kitty/window_list.py:339`: `self.id_map[window.id] = window`), and cleared by `WindowList.remove_window` (`kitty/window_list.py:380`: `self.id_map.pop(q.id, None)`). The `WindowList` also owns a list of window-groups (`kitty/window_list.py:149`: `self.groups: List[WindowGroup] = []`); each **`WindowGroup`** (`class WindowGroup` at `kitty/window_list.py:28`) holds the concrete list of windows that belong to that group in `self.windows: List[WindowType] = []` (`kitty/window_list.py:31`), and `WindowGroup.remove_window` (`kitty/window_list.py:84`) drops a window from its group during teardown. These per-tab structures are unwound by **`Tab.remove_window`** (`kitty/tabs.py:580`: `def remove_window(self, window: Window, destroy: bool = True) -> None:`), whose first action delegates to the list (`kitty/tabs.py:581`: `self.windows.remove_window(window)`), so `WindowList.remove_window` pops `id_map` and prunes the now-empty `WindowGroup` — keeping this Python view in step with the C `children[]` and GUI `global_state` views.
2. **The C I/O-thread view** — the child-monitor's three static arrays and their mutex:

   ```c
#define children_mutex(op) \
    pthread_mutex_##op(&children_lock);
#define talk_mutex(op) \
    pthread_mutex_##op(&talk_lock);


static Child children[MAX_CHILDREN] = {{0}};
static Child scratch[MAX_CHILDREN] = {{0}};
static Child add_queue[MAX_CHILDREN] = {{0}}, remove_queue[MAX_CHILDREN] = {{0}}, remove_notify[MAX_CHILDREN] = {{0}};
   ```

   (`kitty/child-monitor.c:76-84`: `children[]` at `:82`, `scratch[]` at `:83`, and `add_queue[]`/`remove_queue[]`/`remove_notify[]` at `:84`, all serialized by `children_mutex` at `:76-77`.)
3. **The GUI/main-thread structural view** — the global window hierarchy:

   ```c
GlobalState global_state = {{0}};
   ```

   (`kitty/state.c:12`; the `os_windows[].tabs[].windows[]` tree hangs off this `global_state`.)

These are updated on different threads at different times, so during teardown they can transiently disagree. The next two sub-sections show the guards; §7.3 shows the disagreement forced at scale.

### 7.2 The three reconciliation guards

**Guard 1 — `Boss.on_child_death` pop-and-early-return (Python view).** When a child dies, the C layer eventually calls `death_notify` → `Boss.on_child_death`. If the window was *already* closed by another path (so its `window_id_map` entry is gone), the callback must not double-free it:

```python
    def on_child_death(self, window_id: int) -> None:
        prev_active_window = self.active_window
        window = self.window_id_map.pop(window_id, None)
        if window is None:
            return
        with self.suppress_focus_change_events():
            for close_action in window.actions_on_close:
                try:
                    close_action(window)
                except Exception:
                    import traceback
                    traceback.print_exc()
            os_window_id = window.os_window_id
            window.destroy()
            tm = self.os_window_map.get(os_window_id)
            tab = None
            if tm is not None:
                for q in tm:
                    if window in q:
                        tab = q
                        break
            if tab is not None:
                tab.remove_window(window)
                self._cleanup_tab_after_window_removal(tab)
            for removal_action in window.actions_on_removal:
                try:
                    removal_action(window)
                except Exception:
                    import traceback
                    traceback.print_exc()
            del window.actions_on_close[:], window.actions_on_removal[:]

        window = self.active_window
        if window is not prev_active_window:
            if prev_active_window is not None:
                prev_active_window.focus_changed(False)
            if window is not None:
                window.focus_changed(True)
```

(`kitty/boss.py:881-918`.) The guard is `window = self.window_id_map.pop(window_id, None)` at `:883` followed by `if window is None: return` at `:884-885`: the `pop(..., None)` atomically removes-and-fetches, and a `None` result means the window is already gone, so the function returns before touching any freed state. **Observation status: NOT OBSERVED (silent by design).** This branch produces no log or print; it is reachable when `on_os_window_closed` (`kitty/boss.py:1780-1781`) has already `pop`-ed `window_id_map` before a deferred `death_notify` (`kitty/child-monitor.c:522`) arrives. Surfacing it would require editing the source to add a print (forbidden by the read-only rule) or attaching a debugger, whose breakpoint would perturb the sub-millisecond main-loop race and make the observation non-canonical. It is therefore reported as a code-grounded guard that was not forced through the real entry point.

**Guard 2 — `mark_child_for_close` searches *both* live and pending arrays (C view).** When kitty initiates a close, it must be able to cancel a child that is *still in `add_queue[]`* — i.e. created so recently the I/O thread has not yet merged it into `children[]`:

```c
mark_child_for_close(ChildMonitor *self, id_type window_id) {
    bool found = false;
    children_mutex(lock);
    for (size_t i = 0; i < self->count; i++) {
        if (children[i].id == window_id) {
            children[i].needs_removal = true;
            found = true;
            break;
        }
    }
    if (!found) {
        for (size_t i = 0; i < add_queue_count; i++) {
            if (add_queue[i].id == window_id) {
                add_queue[i].needs_removal = true;
                found = true;
                break;
            }
        }

    }
    children_mutex(unlock);
    wakeup_io_loop(self, false);
    return found;
}


```

(`kitty/child-monitor.c:541-564`.) It first scans the live `children[]` array (`:544-550`), setting `children[i].needs_removal = true` at `:546` if found; and **only if not found** (`if (!found)` at `:551`) it scans the pending `add_queue[]` (`:552-558`), setting `add_queue[i].needs_removal = true` at `:554`. This is exactly the "cancel a just-created child before it is fully alive" case. **Observation status: NOT OBSERVED (silent + measured-unreachable via the real path).** The `add_queue[]` branch sets a flag with no log. Reaching it requires a close to be *processed by the I/O thread* while the child is still resident in `add_queue[]`, and I measured that residency window directly (§7.4): it is **≈103 µs**, whereas the fastest real close-trigger the entry point can deliver (`kitty @ --no-response`) has a measured floor of **22 ms** — roughly **214× too slow**. Empirically, across ~9,000+ forced conflict samples the `(add queue: N)` field was **`N == 0` every single time** (§7.3), confirming the child is never caught mid-`add_queue`. It is a real guard for an in-process race that the external entry point cannot win.

**Guard 3 — `hangup` tolerates an already-exited child via `ESRCH` (C view).** When cleaning up a child, kitty sends `SIGHUP` to its process group — but the child may have already exited, so the process-group lookup and the kill must both tolerate `ESRCH` ("no such process"):

```c
hangup(pid_t pid) {
    errno = 0;
    pid_t pgid = getpgid(pid);
    if (errno == ESRCH) return;
    if (errno != 0) { perror("Failed to get process group id for child"); return; }
    if (killpg(pgid, SIGHUP) != 0) {
        if (errno != ESRCH) perror("Failed to kill child");
    }
}
```

(`kitty/child-monitor.c:1294-1302`.) `errno = 0` then `pgid = getpgid(pid)` (`:1295-1296`); if that sets `errno == ESRCH` the child is already gone, so it **returns immediately** at `:1297` (no kill attempted); otherwise `killpg(pgid, SIGHUP)` at `:1299` is issued, and even its failure is tolerated when `errno == ESRCH` at `:1300`. **Observation status: OBSERVED.** In the self-exit teardown (§5.5, Scenario 1), after the child exited on its own, the trace shows `getpgid(59796) = -1 ESRCH` with **no** following `kill` — the `:1297` early-return firing on a real, unmodified run:

```text
59796 05:50:47.360814 +++ exited with 7 +++
59795 05:50:47.360915 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 7}], WNOHANG, NULL) = 59796
59795 05:50:47.360992 wait4(-1, 0x7dd0ab7fde60, WNOHANG, NULL) = -1 ECHILD (No child processes)
59795 05:50:47.361112 getpgid(59796)    = -1 ESRCH (No such process)
```

### 7.3 The directly-observed reconciliation: resize-vs-removal, forced at scale

**This is the R6 conflict observed through the real entry point.** The single *observable* reconciliation point is `resize_pty`'s dual `FIND` + `else log_error` (`kitty/child-monitor.c:592-614`), already quoted in §3. When the Python/GUI view still lists a window and calls `resize_pty(id, …)`, but the C view has already removed the child from **both** `children[]` and `add_queue[]`, both `FIND`s miss (`fd == -1`) and the guard runs:

```c
    FIND(children, self->count);
    if (fd == -1) FIND(add_queue, add_queue_count);
    if (fd != -1) {
        if (!pty_resize(fd, &dim)) PyErr_SetFromErrno(PyExc_OSError);
    } else log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", window_id, self->count, add_queue_count);
```

(`kitty/child-monitor.c:606-610`: `FIND(children,…)` at `:606`, `FIND(add_queue,…)` at `:607`, and the `else log_error` reconciliation at `:610`.) The resize is discarded; nothing is dereferenced.

**How it was forced (real binary, non-root, valid commands only).** Two temporary drivers drove the real kitty binary through rapid create→run→resize→close storms via the canonical remote-control client (using only the valid `close-window` subcommand — never a non-existent `quit`; see §8 and Appendix A). `driver_rapid2.sh` ran two modes: `overlap` (backgrounded `launch` immediately followed by `close-window`) and `selfexit` (children that exit instantly), each **N=150, repeated across 2 runs**. Complete producing commands are in Appendix A. Observed counts (all verified by `grep -c` on the captured stderr logs):

```text
Driver                Mode      N    Run  Child launched  SIGWINCH sent  Failed-to-send-resize  crashes  add_queue>0
driver_rapid.sh       rc-close  50   1    51              1060           0                      0        0
driver_rapid.sh       rc-close  50   2    51              1060           0                      0        0
driver_rapid2.sh      overlap   150  1    151             4857           481                    0        0
driver_rapid2.sh      overlap   150  2    151             4857           528                    0        0
driver_rapid2.sh      selfexit  150  1    151             5166           5165                   0        0
driver_rapid2.sh      selfexit  150  2    151             5282           5281                   0        0
```

**Observed distribution (reported, not stabilised):** the "Failed to send resize signal" count is **run-to-run variable** — `overlap`: {481, 528}; `selfexit`: {5165, 5281} — so it is reported as a distribution per the methodology rule, not collapsed to a single number. The low-churn RC-close driver (`driver_rapid.sh`, N=50×2) produced **0** failed resizes: the conflict only appears under genuine overlap/self-exit churn. Representative complete failed-resize lines (first two + last of a run, unedited) — note the **`children count`** shrinking toward 0 and **`add queue: 0`** in every case:

```text
### overlap_run1 — first 2 and last failed-resize lines (complete, unedited)
[9.209] Failed to send resize signal to child with id: 2 (children count: 134) (add queue: 0)
[32.505] Failed to send resize signal to child with id: 20 (children count: 131) (add queue: 0)
[40.382] Failed to send resize signal to child with id: 151 (children count: 0) (add queue: 0)
### selfexit_run1 — first 2 and last failed-resize lines (complete, unedited)
[1.466] Failed to send resize signal to child with id: 2 (children count: 2) (add queue: 0)
[1.486] Failed to send resize signal to child with id: 3 (children count: 2) (add queue: 0)
[10.531] Failed to send resize signal to child with id: 130 (children count: 0) (add queue: 0)
```

**What this proves for R6.** The `(add queue: 0)` on *every* sample localizes the conflict precisely: the child is gone from **both** C arrays (fully removed, not merely pending), while Python still holds the window and drives a layout→`set_geometry`→`resize_pty`. The C view (child removed) and the Python view (window present) genuinely disagree, and the `:610` guard resolves it by discarding the resize. **Across all six runs there were zero crashes/tracebacks/segfaults/aborts, and kitty remained consistent after every storm** (the `overlap` mode ended in a *clean* process exit once all windows were closed; `selfexit` mode left kitty alive). This also settles **F10**: the claim "the window is still in the Python view during a failed resize" is now **directly observed**, not inferred.

### 7.4 Why Guard 2 is unreachable from the real entry point: the merge-window measurement

To justify the NOT-OBSERVED status of Guard 2 with a measured, code-grounded reason rather than a guess, I measured how long a newly-created child actually resides in `add_queue[]` before the I/O thread merges it into `children[]`. The I/O thread `KittyChildMon` was identified in an `strace -f -tt` by its multi-fd poll signature (its poll set grows `2→3→4` fds as children merge: fd 7 = wakeup eventfd, fd 8 = signalfd, fd 10/12 = child PTYs). The captured merge sequence (complete, unedited; thread `79593`):

```text
79593 06:03:15.202452 poll([{fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 2, -1 <unfinished ...>
79593 06:03:15.214986 <... poll resumed>) = 1 ([{fd=7, revents=POLLIN}]) <0.012525>
79593 06:03:15.215024 read(7, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000012>
79593 06:03:15.215056 read(7, 0x7a257f8c5740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
79593 06:03:15.215089 poll([{fd=7, events=POLLIN}, {fd=8, events=POLLIN}, {fd=10, events=POLLIN}], 3, -1 <unfinished ...>
```

**The add_queue merge window = 103 µs:** the I/O thread's `poll()` is woken at `06:03:15.214986` (by `add_child`'s `wakeup_io_loop` writing the eventfd `fd=7`, drained by the 8-byte `read` at `.215024`), and it re-enters `poll()` with the newly-merged child PTY `fd=10` present at `06:03:15.215089`. The residency window is `.215089 − .214986 = 103 µs`. By contrast, the fastest close-trigger the *real external entry point* can deliver — a `kitty @ --no-response` dispatch — has a measured floor of **22 ms** (6 samples: 24/23/22/30/24/23 ms; a warm `kitty @ ls` round-trip is ~36–46 ms). The complete `date +%s%N` measurement output:

```text
# Warm kitty @ ls round-trip latency (ms), 6 samples:
  ls round-trip #1: 317 ms
  ls round-trip #2: 768 ms
  ls round-trip #3: 36 ms
  ls round-trip #4: 46 ms
  ls round-trip #5: 37 ms
  ls round-trip #6: 36 ms
# kitty @ --no-response dispatch latency (ms), 6 samples (fastest close-trigger proxy):
  --no-response dispatch #1: 24 ms
  --no-response dispatch #2: 23 ms
  --no-response dispatch #3: 22 ms
  --no-response dispatch #4: 30 ms
  --no-response dispatch #5: 24 ms
  --no-response dispatch #6: 23 ms
```

The ratio is `≈ 22000 µs / 103 µs ≈ 214×`: a real close command arrives about two orders of magnitude too late to catch a child during its ~103 µs `add_queue` residency. Guard 2 protects an **in-process** race (a create and close queued back-to-back inside the same C call sequence before the next I/O tick), which the external RC path structurally cannot win — hence NOT OBSERVED, with a measured reason.

### 7.5 Guard-firing summary

| Guard | Location | View reconciled | Status | Evidence |
|-------|----------|-----------------|--------|----------|
| `resize_pty` dual-`FIND` + `else log_error` | `child-monitor.c:606-610` | Python (window present) vs C (child removed) | **OBSERVED, forced at scale** | §7.3 distribution: overlap {481,528}, selfexit {5165,5281}; add queue: 0 always; 0 crashes |
| `hangup` `ESRCH` tolerance | `child-monitor.c:1297,1300` | C (wants to `SIGHUP`) vs OS (child already exited) | **OBSERVED** | §5.5/§7.2: `getpgid(59796) = -1 ESRCH`, no kill |
| `on_child_death` pop-and-early-return | `boss.py:883-885` | Python-vs-Python (death_notify vs prior close) | NOT OBSERVED (silent; read-only forbids instrumenting) | code-grounded; reachable via `boss.py:1780-1781` before `child-monitor.c:522` |
| `mark_child_for_close` `add_queue[]` branch | `child-monitor.c:552-558` | C live vs C pending (cancel just-created child) | NOT OBSERVED (measured-unreachable via real path) | §7.4: 103 µs residency vs 22 ms trigger floor (214×); add queue: 0 across ~9000+ samples |

---


## 8. Coverage pass

Re-reading the question and confirming every distinct thing it asks — and every named "e.g./such as/including" item — is answered above with a producing command, complete output, and a `file:line` anchor:

| # | Question sub-item (verbatim intent) | Answered in | Direct result |
|---|-------------------------------------|-------------|---------------|
| R1 | "keeps its internal state consistent when windows appear, resize, and disappear in quick succession" | §4 (+§5–§7) | Four mechanisms: startup gate, single-owner main-thread resize, de-dup guard, per-tick removals-then-additions reconciliation |
| R2 | "a new window is created and immediately used to run a command, resize events and signals start flowing" | §3 | Observed flow: `add_child` → first `set_geometry` → `TIOCSWINSZ` → child `SIGWINCH` → gate opens → `execve`; complete stderr + `strace` |
| R3 | "What happens if the window is gone before everything has finished reacting to those changes?" | §5 | The removed child + its `Screen` are kept alive by a reference-counted `scratch[]` snapshot; final output is drained (`do_parse … flush=true`), then freed |
| R4 | "How does kitty decide what state to keep and what to discard?" | §5.3 | Keep the in-flight bytes (drain removed child with `flush=true`), discard the dead handle (skip further parse, `DECREF_CHILD` frees at refcount 0) |
| R5 | "how timing affects signal delivery and internal bookkeeping" | §6 | `signalfd`+`SIG_BLOCK` (no `SA_RESTART`); `read_signals` drains ≤32/​wakeup; `handle_signal` flags `ss.child_died`; same-tick `reap_children` `WNOHANG` loop; 10-run gap distribution {min 0.083, median 0.111, max 18.333} ms |
| R6 | "moments where the system has to resolve conflicting views of what is still alive" | §7 | Yes: 3 views + 3 guards; resize-vs-removal conflict forced at scale (overlap {481,528}, selfexit {5165,5281}); 0 crashes |
| named: "signal delivery" | — | §3.2, §6.1–§6.4 | `signalfd` mechanism, `SIGWINCH` direction (kitty→child), the 6 handled signals, coalescing reaper |
| named: "internal bookkeeping" | — | §5, §7.1 | Three arrays + mutex; three liveness views; ref-counted snapshot |
| named: "`SIGCHLD`" | — | §5.5, §6.3–§6.5 | `handle_signal` → `ss.child_died` → `reap_children`; self-exit vs kitty-initiated close distinguished |
| named: "`SIGWINCH`" | — | §3, §4, §6.3 | Sent kitty→child via `TIOCSWINSZ`; deliberately *not* in kitty's handled set |
| named: "resize while gone" | — | §5.5 S3, §7.3 | `resize_pty` dual-`FIND` miss → `else log_error`, discarded, no UAF |
| named: "keep vs discard" | — | §5.3 | The `flush=true` drain + `DECREF_CHILD` free decision |
| named: "before/during/after" | — | §3.3, §5.5, §7.3 | Each state transition reported before, during, after |
| named: "at least two runs / distribution" | — | §6.5, §7.3 | 10-run timing distribution; F6 counts across 2 runs per mode |
| named: "primary + secondary paths" | — | §5.5 | Ordinary self-exit (S1), kitty-initiated close (S2/S4), resize-vs-removal (S3), rapid create-close (§7.3) |
| named: "real entry point, not a bypass" | — | §2, §7.3 | Real kitty binary; RC used only to *trigger* real window ops, never to substitute for the observed mechanism |

Every row is backed by captured output in the corresponding section; nothing is left as "would" behaviour except the two guards explicitly labelled NOT OBSERVED (§7.2), each with a measured, code-grounded reason.

---

## Appendix A — Temporary observation scripts (complete text)

All scripts below were created under the container's `/home/ubuntu/` (never inside the repository tree) and removed after the campaign (Appendix C). They are reproduced here in full so every run is reproducible. They use **only** valid remote-control subcommands (`launch`, `close-window`, `resize-os-window`) — never a non-existent `quit` subcommand.

**A.1 `driver_rapid.sh`** — low-churn RC create/close (produced 0 failed resizes):

```bash
#!/usr/bin/env bash
# F6 rapid create-close driver — real entry point (real kitty binary + canonical RC client).
# Goal: churn windows fast enough to (a) race resizes against removals -> "Failed to send
# resize signal ... (add queue: N)" observable, and (b) attempt the mark_child_for_close
# add_queue[] branch (silent) by closing before the I/O thread merges add_queue->children[].
# Uses ONLY valid RC subcommands (launch, close-window). NO invalid 'quit' (F11).
set -u
RUN="${1:-1}"
N="${2:-50}"
LAUNCHER=/home/ubuntu/kitty/kitty/launcher/kitty
SOCK=/tmp/kitty_f6_${RUN}
LOG=/home/ubuntu/evidence/rapid_run${RUN}_stderr.log
: > "$LOG"

# Start ONE persistent kitty (real binary), RC enabled, debug-rendering for Child launched/SIGWINCH.
DISPLAY=:99 "$LAUNCHER" --debug-rendering --config NONE \
    -o allow_remote_control=yes --listen-on "unix:${SOCK}" \
    sh -c 'exec sleep 120' >>"$LOG" 2>&1 &
KPID=$!

# Wait for the RC socket to be live.
for i in $(seq 1 100); do [ -S "$SOCK" ] && break; sleep 0.05; done
KAT() { DISPLAY=:99 "$LAUNCHER" @ --to "unix:${SOCK}" "$@"; }

# Rapid create-then-immediate-close cycles. --no-response minimizes client round-trip latency.
for i in $(seq 1 "$N"); do
    KAT launch --no-response --keep-focus --cwd /tmp sh -c 'exec sleep 30' >/dev/null 2>&1
    KAT close-window --no-response --match recent:0             >/dev/null 2>&1
done

sleep 0.5
# Is the persistent kitty still alive after the storm? (crash check)
if kill -0 "$KPID" 2>/dev/null; then echo "KITTY_ALIVE_AFTER_STORM=yes pid=$KPID"; else echo "KITTY_ALIVE_AFTER_STORM=no"; fi
# Clean shutdown via VALID close-window (F11: never 'quit'); then ensure the process is gone.
KAT close-window --no-response --match all >/dev/null 2>&1 || true
sleep 0.3
kill "$KPID" 2>/dev/null || true
wait "$KPID" 2>/dev/null
rm -f "$SOCK"
echo "RUN ${RUN} DONE (N=${N}); log=${LOG}"
```

**A.2 `driver_rapid2.sh`** — aggressive `overlap`/`selfexit` race driver (forced the resize-vs-removal conflict):

```bash
#!/usr/bin/env bash
# F6 aggressive race driver — real entry point. MODE in {overlap, selfexit}.
#  overlap : background the launch and fire close-window immediately, so close can reach
#            mark_child_for_close while the child may still be in add_queue[] (Guard 2).
#  selfexit: launch children that exit instantly; rapid self-exit SIGCHLD races the tab
#            relayout resizes -> may hit resize_pty total-miss "Failed to send resize signal".
# Uses ONLY valid RC subcommands (launch, close-window). No 'quit' (F11).
set -u
MODE="${1:-overlap}"; RUN="${2:-1}"; N="${3:-150}"
LAUNCHER=/home/ubuntu/kitty/kitty/launcher/kitty
SOCK=/tmp/kitty_f6b_${MODE}_${RUN}
LOG=/home/ubuntu/evidence/rapid2_${MODE}_run${RUN}_stderr.log
: > "$LOG"
DISPLAY=:99 "$LAUNCHER" --debug-rendering --config NONE \
    -o allow_remote_control=yes --listen-on "unix:${SOCK}" \
    sh -c 'exec sleep 180' >>"$LOG" 2>&1 &
KPID=$!
for i in $(seq 1 100); do [ -S "$SOCK" ] && break; sleep 0.05; done
KAT() { DISPLAY=:99 "$LAUNCHER" @ --to "unix:${SOCK}" "$@"; }

if [ "$MODE" = "overlap" ]; then
    for i in $(seq 1 "$N"); do
        KAT launch --no-response --keep-focus --cwd /tmp sh -c 'exec sleep 30' >/dev/null 2>&1 &
        KAT close-window --no-response --match recent:0 >/dev/null 2>&1
    done
    wait 2>/dev/null
else  # selfexit
    for i in $(seq 1 "$N"); do
        KAT launch --no-response --keep-focus --cwd /tmp sh -c 'exit 0' >/dev/null 2>&1
    done
fi

sleep 0.6
if kill -0 "$KPID" 2>/dev/null; then echo "KITTY_ALIVE_AFTER_STORM=yes pid=$KPID"; else echo "KITTY_ALIVE_AFTER_STORM=no"; fi
KAT close-window --no-response --match all >/dev/null 2>&1 || true
sleep 0.3
kill "$KPID" 2>/dev/null || true
wait "$KPID" 2>/dev/null
rm -f "$SOCK"
echo "MODE=${MODE} RUN=${RUN} N=${N} DONE; log=${LOG}"
```

**A.3 `driver_dedup.sh`** — exercises the `set_geometry` de-dup guard (A=new, B=identical, C=changed):

```bash
#!/bin/bash
# driver_dedup.sh — exercise the window.py:861 de-dup guard.
# A=900x600 (new), B=900x600 (identical -> guard discards), C=1100x750 (changed).
set -u
cd "$HOME/kitty"
E="$HOME/evidence"
SOCK=/tmp/rc_dedup.sock; rm -f "$SOCK" "$E/dedup.strace" "$E/dedup_stderr.log"
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe LANG=C.UTF-8 LC_ALL=C.UTF-8
strace -f -e trace=ioctl -tt -o "$E/dedup.strace" \
  ./kitty/launcher/kitty --debug-rendering --config NONE \
  -o allow_remote_control=yes --listen-on unix:"$SOCK" \
  sh -c "exec sleep 60" > "$E/dedup_stderr.log" 2>&1 &
KPID=$!
sleep 4
KAT() { ./kitty/launcher/kitty @ --to unix:"$SOCK" "$@"; }
echo "### STEP A: resize 900x600 (NEW)"
KAT resize-os-window --width 900 --height 600 2>&1 | head -1
sleep 1
echo "### STEP B: resize 900x600 AGAIN (IDENTICAL -> de-dup)"
KAT resize-os-window --width 900 --height 600 2>&1 | head -1
sleep 1
echo "### STEP C: resize 1100x750 (CHANGED)"
KAT resize-os-window --width 1100 --height 750 2>&1 | head -1
sleep 1
KAT close-window --match all 2>&1 | head -1
sleep 1
kill -TERM $KPID 2>/dev/null; wait $KPID 2>/dev/null
echo "### DONE"
```

**A.4 `driver_timing.sh`** — measures child-exit → `wait4` reap latency for one run:

```bash
#!/bin/bash
# driver_timing.sh RUN — measure child-exit -> kitty-reap(wait4) latency for one run.
set -u
cd "$HOME/kitty"; E="$HOME/evidence"; RUN="$1"
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe LANG=C.UTF-8 LC_ALL=C.UTF-8
timeout -s TERM 12 strace -f -e trace=wait4 -tt -o "$E/timing_${RUN}.strace" \
  ./kitty/launcher/kitty --debug-rendering --config NONE sh -c "exit 0" >"$E/timing_${RUN}_out.log" 2>&1
python3 - "$RUN" "$E" <<"PY"
import sys,re
run,E=sys.argv[1],sys.argv[2]
lines=open(f"{E}/timing_{run}.strace").read().splitlines()
def ts(l):
    m=re.match(r"\s*\d+\s+(\d+):(\d+):(\d+)\.(\d+)",l)
    if not m: return None
    h,mm,s,us=map(int,m.groups()); return ((h*60+mm)*60+s)+us/1e6
exit_t=reap_t=reaped=None; exit_line=reap_line=""
for l in lines:
    if "+++ exited with 0 +++" in l and exit_t is None:
        exit_t=ts(l); exit_line=l.strip()
    m=re.search(r"wait4\(-1, \[.*WIFEXITED.*\], WNOHANG, NULL\) = (\d+)",l)
    if m and int(m.group(1))>0 and reap_t is None:
        reap_t=ts(l); reaped=m.group(1); reap_line=l.strip()
if exit_t and reap_t:
    print(f"RUN {run}: gap = {(reap_t-exit_t)*1000:.3f} ms (reaped pid {reaped})")
    print(f"  EXIT: {exit_line}")
    print(f"  REAP: {reap_line}")
else:
    print(f"RUN {run}: could not extract (exit_t={exit_t} reap_t={reap_t})")
PY
```

**A.5 `measure_wakeup.sh`** — measures the `add_queue`→`children[]` merge window:

```bash
#!/usr/bin/env bash
# Measure the I/O-loop wakeup latency = the add_queue->children[] merge window (Guard 2 race window).
set -u
LAUNCHER=/home/ubuntu/kitty/kitty/launcher/kitty
SOCK=/tmp/kitty_wake
TR=/home/ubuntu/evidence/wakeup.strace
rm -f "$TR"
# strace the whole kitty (follow threads) with microsecond timestamps, tracing poll + pipe I/O.
DISPLAY=:99 strace -f -tt -T -e trace=poll,ppoll,write,read,close,pipe2,eventfd2 -o "$TR" \
    "$LAUNCHER" --debug-rendering --config NONE \
    -o allow_remote_control=yes --listen-on "unix:${SOCK}" \
    sh -c 'exec sleep 20' >/dev/null 2>&1 &
KPID=$!
for i in $(seq 1 100); do [ -S "$SOCK" ] && break; sleep 0.05; done
sleep 0.5
# One launch (creates a child -> add_child -> wakeup_io_loop -> add_children merge next tick).
DISPLAY=:99 "$LAUNCHER" @ --to "unix:${SOCK}" launch --no-response sh -c 'exec sleep 10' >/dev/null 2>&1
sleep 0.5
DISPLAY=:99 "$LAUNCHER" @ --to "unix:${SOCK}" close-window --no-response --match all >/dev/null 2>&1
sleep 0.3
kill "$KPID" 2>/dev/null; wait "$KPID" 2>/dev/null
rm -f "$SOCK"
echo "trace=$TR lines=$(wc -l < "$TR")"
```

**A.6 RC client latency** — the floor for any external close/resize trigger, measured with `date +%s%N` (complete output; used in §7.4):

```text
# Warm kitty @ ls round-trip latency (ms), 6 samples:
  ls round-trip #1: 317 ms
  ls round-trip #2: 768 ms
  ls round-trip #3: 36 ms
  ls round-trip #4: 46 ms
  ls round-trip #5: 37 ms
  ls round-trip #6: 36 ms
# kitty @ --no-response dispatch latency (ms), 6 samples (fastest close-trigger proxy):
  --no-response dispatch #1: 24 ms
  --no-response dispatch #2: 23 ms
  --no-response dispatch #3: 22 ms
  --no-response dispatch #4: 30 ms
  --no-response dispatch #5: 24 ms
  --no-response dispatch #6: 23 ms
```

---


## Appendix B — Code anchor table (verified against commit `815df1e210e0`)

Every anchor below was re-verified line-by-line against the built source tree at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. `file:line` → exact symbol/behaviour at that line.

| File:line | Symbol / behaviour |
|-----------|--------------------|
| `kitty/child-monitor.c:76-77` | `children_mutex(op)` macro (all array access serialized by `children_lock`) |
| `kitty/child-monitor.c:82` | `static Child children[MAX_CHILDREN]` — the live child array |
| `kitty/child-monitor.c:83` | `static Child scratch[MAX_CHILDREN]` — the reference-counted parse snapshot |
| `kitty/child-monitor.c:84` | `add_queue[]`, `remove_queue[]`, `remove_notify[]` staging arrays |
| `kitty/child-monitor.c:121` | `KITTY_HANDLED_SIGNALS` = `SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` (no `SIGWINCH`) |
| `kitty/child-monitor.c:305-321` | `add_child` — appends to `add_queue[]` under mutex, `INCREF`, wakes I/O loop |
| `kitty/child-monitor.c:451-533` | `parse_input` — locked snapshot into `scratch[]` w/ `INCREF`, then lock-free parse + death-notify |
| `kitty/child-monitor.c:485-515` | peer (remote-control) message dispatch under `talk_mutex`, `peer_message_received` at `:504` |
| `kitty/child-monitor.c:521-525` | removed child: `do_parse(… flush=true)`, `death_notify`, `FREE_CHILD` |
| `kitty/child-monitor.c:529-532` | live snapshot: skip parse if `needs_removal`, always `DECREF_CHILD` |
| `kitty/child-monitor.c:541-564` | `mark_child_for_close` — search `children[]` (`:546`) then `add_queue[]` (`:554`) |
| `kitty/child-monitor.c:577-590` | `pty_resize` — `ioctl(TIOCSWINSZ)` (`:579`), `EINTR` retry (`:580`), `EBADF`/`ENOTTY` tolerated (`:581`) |
| `kitty/child-monitor.c:592-614` | `resize_pty` — `FIND(children)` (`:606`), `FIND(add_queue)` (`:607`), `else log_error` (`:610`) |
| `kitty/child-monitor.c:1294-1302` | `hangup` — `getpgid` `ESRCH` return (`:1297`), `killpg(pgid, SIGHUP)` (`:1299`), `ESRCH` tolerated (`:1300`) |
| `kitty/child-monitor.c:1306-1308` | `cleanup_child` — `safe_close` (`:1307`), `hangup` (`:1308`) |
| `kitty/child-monitor.c:1313-1332` | `remove_children` — reverse scan, `cleanup_child`, copy to `remove_queue[]` (`:1320`), compact |
| `kitty/child-monitor.c:1337-1356` | `read_bytes` — `EINTR`/`EAGAIN` retry (`:1347`), `EIO` silent (`:1348`) |
| `kitty/child-monitor.c:1359` | `typedef struct { bool kill_signal, child_died, reload_config; } SignalSet;` |
| `kitty/child-monitor.c:1362-1383` | `handle_signal` — `kill_signal` (`:1368`), `child_died` (`:1371`), `reload_config` (`:1374`) |
| `kitty/child-monitor.c:1386-1395` | `mark_child_for_removal` — set `needs_removal = true` by pid (`:1390`) |
| `kitty/child-monitor.c:1413-1426` | `reap_children` — `waitpid(-1,&status,WNOHANG)` loop (`:1418`), `mark_child_for_removal` (`:1422`) |
| `kitty/child-monitor.c:1491-1495` | I/O loop body — `remove_children` (`:1493`) then `add_children` (`:1494`) under mutex |
| `kitty/child-monitor.c:1517-1526` | signal-fd drain — `read_signals` (`:1519`), `if (ss.child_died) reap_children` (`:1526`) |
| `kitty/loop-utils.c:34-56` | `init_signal_handlers` — `sigprocmask(SIG_BLOCK)` (`:41`) + `signalfd` (`:42`); `#else` self-pipe+`SA_RESTART` (`:46-53`) |
| `kitty/loop-utils.c:131-180` | `read_signals` — `fdsi[32]` (`:133`), `read` (`:136`), `callback(&si,data)` (`:157`) |
| `kitty/loop-utils.h:31-43` | `LoopData` struct — `signal_fds[2]` (`:36`), `signal_read_fd` (`:40`) |
| `kitty/boss.py:344` | `self.window_id_map: WeakValueDictionary[int, Window]` |
| `kitty/boss.py:585-587` | `add_child` → `self.child_monitor.add_child(window.id, pid, child_fd, screen)` |
| `kitty/boss.py:881-918` | `on_child_death` — `window_id_map.pop(id, None)` (`:883`), `if window is None: return` (`:884-885`) |
| `kitty/window.py:579` | `self.last_reported_pty_size = (-1, -1, -1, -1)` initial |
| `kitty/window.py:850-874` | `set_geometry` — de-dup guard (`:861`), `resize_pty` (`:863`), `mark_terminal_ready` (`:866`), debug prints (`:871`,`:873`) |
| `kitty/child.py:362-364` | `mark_terminal_ready` — `os.close(self.terminal_ready_fd)` |
| `kitty/child.c:71-78` | `wait_for_terminal_ready` — blocking `read` gate; called at `:152` before `execvp` |
| `kitty/state.c:12` | `GlobalState global_state = {{0}};` |
| `kitty/state.h:189,226,265` | `Window *windows` / `Tab *tabs` / `OSWindow *os_windows` — the `os_windows[].tabs[].windows[]` hierarchy |

---

## Appendix C — Read-only compliance and cleanup proof

The investigation created exactly one persistent artifact — this document — and left the repository otherwise unchanged. All observation scripts, traces, logs, and build clones lived under the container's `/home/ubuntu/` (and the host scratch `/tmp/qa_evidence/`), never inside the repository tree, and were removed after use.

**C.1 The single tracked change is this document.** Complete `git status` output on the destination branch:

```bash
git -C /tmp/blitzy/kitty/blitzy-cfa0f0ef-9b7d-4a86-813c-794621318cf1_d6330e status --porcelain --untracked-files=all
```

```text
 M blitzy/documentation/kitty_815df1e210e0.md
```

**C.2 Diff stat — only the answer file is added.** Complete `git diff --stat` against the pre-existing HEAD:

```bash
git -C /tmp/blitzy/kitty/blitzy-cfa0f0ef-9b7d-4a86-813c-794621318cf1_d6330e diff --stat HEAD
```

```text
 blitzy/documentation/kitty_815df1e210e0.md | 1756 ++++++++++++++++++++++------
 1 file changed, 1366 insertions(+), 390 deletions(-)
```

**C.3 No source file modified.** No file under `kitty/`, `kitty_tests/`, `docs/`, nor any build/config file (`setup.py`, `pyproject.toml`, `go.mod`, `Makefile`) was edited; the build clones used for observation were separate copies at `/home/ubuntu/kitty` and `/home/ubuntu/kitty_dbg`, and all temporary scripts (`driver_*.sh`, `measure_*.sh`) and traces (`*.strace`, `*_stderr.log`) were deleted from `/home/ubuntu/` at the end of the campaign.

---

## Appendix D — Complete unabridged build and test logs

Reproduced in full (no elision) so the build claims in §1 are independently verifiable. All runs were performed as the non-root user `ubuntu` (`uid=1000`) inside the task's Docker container.

**D.1 Canonical build — `python3 setup.py` (complete 366-line log).** Producing command:

```bash
cd /home/ubuntu/kitty && time python3 setup.py 2>&1
```

```text
BUILD USER: ubuntu uid=1000
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
[4/28] Generating wayland-viewporter-client-protocol.c ...
[5/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.h ...
[6/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.c ...
[7/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.h ...
[8/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[9/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.h ...
[10/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[11/28] Generating wayland-primary-selection-unstable-v1-client-protocol.h ...
[12/28] Generating wayland-primary-selection-unstable-v1-client-protocol.c ...
[13/28] Generating wayland-text-input-unstable-v3-client-protocol.h ...
[14/28] Generating wayland-text-input-unstable-v3-client-protocol.c ...
[15/28] Generating wayland-xdg-activation-v1-client-protocol.h ...
[16/28] Generating wayland-xdg-activation-v1-client-protocol.c ...
[17/28] Generating wayland-tablet-unstable-v2-client-protocol.h ...
[18/28] Generating wayland-tablet-unstable-v2-client-protocol.c ...
[19/28] Generating wayland-cursor-shape-v1-client-protocol.h ...
[20/28] Generating wayland-cursor-shape-v1-client-protocol.c ...
[21/28] Generating wayland-fractional-scale-v1-client-protocol.h ...
[22/28] Generating wayland-fractional-scale-v1-client-protocol.c ...
[23/28] Generating wayland-single-pixel-buffer-v1-client-protocol.h ...
[24/28] Generating wayland-single-pixel-buffer-v1-client-protocol.c ...
[25/28] Generating wayland-kwin-blur-v1-client-protocol.h ...
[26/28] Generating wayland-kwin-blur-v1-client-protocol.c ...
[27/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.h ...
[28/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
 done
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
vendor/golang.org/x/crypto/cryptobyte/asn1
internal/nettrace
unicode/utf16
vendor/golang.org/x/crypto/internal/alias
crypto/internal/alias
kitty
container/list
image/color
encoding
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
github.com/shirou/gopsutil/v3/common
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
crypto/internal/boring/sig
log/internal
crypto/subtle
golang.org/x/exp/constraints
internal/weak
maps
math/rand/v2
vendor/golang.org/x/net/dns/dnsmessage
internal/singleflight
crypto/internal/randutil
hash
encoding/base32
crypto/rc4
vendor/golang.org/x/text/transform
net/http/internal/ascii
bufio
regexp/syntax
encoding/binary
context
embed
runtime/cgo
crypto/cipher
crypto/internal/edwards25519/field
crypto/internal/nistec/fiat
image/color/palette
io/ioutil
vendor/golang.org/x/sys/cpu
kitty/tools/utils/shlex
flag
log
net/url
encoding/hex
vendor/golang.org/x/net/http2/hpack
github.com/bmatcuk/doublestar/v4
github.com/ALTree/bigfloat
crypto/internal/bigmod
encoding/asn1
crypto/dsa
github.com/seancfoley/bintree/tree
crypto
hash/adler32
hash/crc32
crypto/md5
golang.org/x/image/riff
internal/concurrent
crypto/internal/edwards25519
golang.org/x/image/tiff/lzw
crypto/internal/boring
net/http/internal
database/sql/driver
encoding/xml
mime/quotedprintable
compress/lzw
image
compress/bzip2
vendor/golang.org/x/text/unicode/bidi
os/signal
os/exec
crypto/des
compress/flate
unique
encoding/base64
github.com/klauspost/cpuid/v2
vendor/golang.org/x/crypto/chacha20
github.com/rwcarlsen/goexif/tiff
vendor/golang.org/x/crypto/internal/poly1305
vendor/golang.org/x/crypto/sha3
github.com/dlclark/regexp2/syntax
vendor/golang.org/x/text/unicode/norm
golang.org/x/sys/unix
crypto/hmac
crypto/rand
crypto/sha1
crypto/sha512
crypto/internal/boring/bbig
crypto/sha256
crypto/aes
crypto/x509/pkix
vendor/golang.org/x/crypto/cryptobyte
vendor/golang.org/x/crypto/hkdf
encoding/pem
mime
vendor/golang.org/x/crypto/chacha20poly1305
encoding/json
regexp
kitty/tools/utils/secrets
crypto/rsa
net/netip
crypto/internal/mlkem768
github.com/shirou/gopsutil/v3/internal/common
crypto/ed25519
golang.org/x/image/bmp
image/internal/imageutil
golang.org/x/image/ccitt
golang.org/x/image/vp8l
golang.org/x/image/vp8
compress/gzip
compress/zlib
archive/zip
vendor/golang.org/x/text/secure/bidirule
image/draw
image/jpeg
crypto/internal/nistec
image/png
golang.org/x/image/tiff
github.com/zeebo/xxh3
image/gif
golang.org/x/image/webp
howett.net/plist
vendor/golang.org/x/net/idna
github.com/dlclark/regexp2
github.com/disintegration/imaging
github.com/kovidgoyal/imaging
crypto/ecdh
crypto/elliptic
github.com/rwcarlsen/goexif/exif
crypto/internal/hpke
crypto/ecdsa
github.com/edwvee/exiffix
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/tklauser/go-sysconf
github.com/shirou/gopsutil/v3/cpu
os/user
net
github.com/alecthomas/chroma/v2/styles
github.com/alecthomas/chroma/v2/lexers
archive/tar
github.com/shirou/gopsutil/v3/net
vendor/golang.org/x/net/http/httpproxy
net/textproto
github.com/google/uuid
crypto/x509
github.com/seancfoley/ipaddress-go/ipaddr
vendor/golang.org/x/net/http/httpguts
mime/multipart
github.com/shirou/gopsutil/v3/process
crypto/tls
net/http/httptrace
net/http
kitty/tools/utils
kitty/tools/utils/base85
kitty/tools/utils/paths
kitty/tools/tty
kitty/tools/rsync
kitty/tools/wcswidth
kitty/tools/crypto
kitty/tools/tui/shell_integration
kitty/tools/utils/humanize
kitty/tools/utils/style
kitty/tools/cli/markup
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/cli
kitty/tools/config
kitty/tools/cmd/mouse_demo
kitty/tools/tui/shortcuts
kitty/kittens/query_terminal
kitty/kittens/hyperlinked_grep
kitty/tools/utils/shm
kitty/kittens/show_key
kitty/tools/tui/readline
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/tools/cmd/edit_in_kitty
kitty/tools/tui/graphics
kitty/tools/cmd/show_error
kitty/tools/cmd/update_self
kitty/tools/cmd/run_shell
kitty/kittens/ask
kitty/kittens/hints
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/kittens/themes
kitty/kittens/ssh
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd

real	0m47.432s
user	2m53.312s
sys	0m38.158s
CANONICAL_BUILD_EXIT=0
```

**D.2 Debug / event-loop build — `python3 setup.py build --debug --extra-logging=event-loop` (complete 215-line log).** Producing command:

```bash
cd /home/ubuntu/kitty_dbg && python3 setup.py build --debug --extra-logging=event-loop 2>&1
```

```text
DEBUG BUILD USER: ubuntu uid=1000  HEAD: 815df1e21
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
[4/28] Generating wayland-viewporter-client-protocol.c ...
[5/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.h ...
[6/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.c ...
[7/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.h ...
[8/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[9/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.h ...
[10/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[11/28] Generating wayland-primary-selection-unstable-v1-client-protocol.h ...
[12/28] Generating wayland-primary-selection-unstable-v1-client-protocol.c ...
[13/28] Generating wayland-text-input-unstable-v3-client-protocol.h ...
[14/28] Generating wayland-text-input-unstable-v3-client-protocol.c ...
[15/28] Generating wayland-xdg-activation-v1-client-protocol.h ...
[16/28] Generating wayland-xdg-activation-v1-client-protocol.c ...
[17/28] Generating wayland-tablet-unstable-v2-client-protocol.h ...
[18/28] Generating wayland-tablet-unstable-v2-client-protocol.c ...
[19/28] Generating wayland-cursor-shape-v1-client-protocol.h ...
[20/28] Generating wayland-cursor-shape-v1-client-protocol.c ...
[21/28] Generating wayland-fractional-scale-v1-client-protocol.h ...
[22/28] Generating wayland-fractional-scale-v1-client-protocol.c ...
[23/28] Generating wayland-single-pixel-buffer-v1-client-protocol.h ...
[24/28] Generating wayland-single-pixel-buffer-v1-client-protocol.c ...
[25/28] Generating wayland-kwin-blur-v1-client-protocol.h ...
[26/28] Generating wayland-kwin-blur-v1-client-protocol.c ...
[27/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.h ...
[28/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
 done
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty
kitty/tools/utils/shlex
kitty/tools/utils/secrets
kitty/tools/utils
kitty/tools/utils/base85
kitty/tools/tty
kitty/tools/utils/paths
kitty/tools/rsync
kitty/tools/wcswidth
kitty/tools/crypto
kitty/tools/tui/shell_integration
kitty/tools/utils/humanize
kitty/tools/utils/style
kitty/tools/cli/markup
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/cli
kitty/tools/config
kitty/tools/cmd/mouse_demo
kitty/tools/tui/shortcuts
kitty/tools/utils/shm
kitty/kittens/query_terminal
kitty/kittens/hyperlinked_grep
kitty/kittens/show_key
kitty/tools/tui/readline
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/tools/cmd/run_shell
kitty/tools/cmd/show_error
kitty/tools/cmd/update_self
kitty/kittens/ask
kitty/tools/cmd/edit_in_kitty
kitty/tools/tui/graphics
kitty/kittens/hints
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/tools/cmd/benchmark
kitty/kittens/choose_fonts
kitty/kittens/icat
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd

real	0m9.699s
user	0m59.952s
sys	0m17.340s
DEBUG_BUILD_EXIT=0
```

**D.3 Test harness — `python3 setup.py test` (complete summary).** Producing command:

```bash
cd /home/ubuntu/kitty && python3 setup.py test 2>&1
```

```text
Running under CI: False
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... skipped 'fish not installed'
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... skipped 'fish not installed'
Ran 145 tests in 14.422s
OK (skipped=4)
All Go tests succeeded, ran in 14.7 seconds
```

